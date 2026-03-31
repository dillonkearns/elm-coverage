## Plan: Mutation Testing PoC in elm-build

### Why elm-build is ideal for this

elm-build's `PureTestRunner` already has everything mutation testing needs:

- **Tests run as pure Elm functions** via `Test.Runner` — no subprocess per test run, no elm-test CLI
- **Dependency graph** (`DepGraph`) knows which tests transitively depend on which source files — so you only re-run affected tests per mutation
- **Content-addressed caching** (`Cache`) means if you mutate file A but test T only depends on file B, T's cached result is reused automatically
- **`elm-syntax` is already a dependency** (`stil4m/elm-syntax: 7.3.9`) — you can parse Elm source into a full AST, generate mutations, and pretty-print back, all in pure Elm. No external binary needed.
- **`Cache.compute`** is the perfect primitive for caching mutation test results — each (mutation, test) pair gets a deterministic hash

### Architecture

```
Source files (in memory from setupSourceFiles)
        |
        v
  Parse with elm-syntax → AST
        |
        v
  Generate mutations (AST → AST transforms)
  Each mutation = { file, location, operator, mutatedSource }
        |
        v
  For each mutation:
    1. Compute new source hash (changed file has different content)
    2. Look up affected tests via DepGraph.transitiveDeps (reverse lookup)
    3. Re-run only affected tests via Cache.compute (with mutated hash as input)
    4. Record result: killed (test failed = good) or survived (all passed = weak test)
        |
        v
  Report: list of surviving mutants with location + description
```

### The recompilation question

There's one fundamental constraint: `PureTestRunner` compiles tests into the elm-pages script itself. Mutating a source string at runtime doesn't change the compiled code.

Two approaches to handle this:

**Approach A: Recompile per mutation (accurate, slower)**

For each mutation:
1. Write mutated source file to a temp copy of the project
2. Use `Cache.commandInWritableDirectory` to run `lamdera make` on the mutated project
3. Run the compiled output and collect test results
4. The cache means identical mutations (same hash) skip recompilation

This is the traditional mutation testing approach. elm-build's cache makes it faster than naive mutation testing because:
- Only the mutated module recompiles (Lamdera incremental compilation)
- Unaffected test results come from cache
- You can batch mutations to the same file and reuse the compilation workspace

**Approach B: Expression-level mutation via test wrappers (fast, limited scope)**

Instead of mutating source and recompiling, design test helpers that test multiple variants:

```elm
-- A module that provides "mutation-aware" assertions
MutationTest.withMutations
    { original = myFunction
    , mutations =
        [ { name = "negate comparison", fn = myFunction_mutated1 }
        , { name = "swap branches", fn = myFunction_mutated2 }
        ]
    }
    (\fn -> fn input |> Expect.equal expected)
```

This is more like property-based testing with explicit variants. It's fast (no recompilation) but requires the user to define mutation targets.

**Approach C (recommended for PoC): Source-level mutation with batch recompilation**

1. Generate all mutations upfront (pure Elm, using elm-syntax)
2. Group mutations by file
3. For each mutated file, write it + recompile + run affected tests
4. Use `Cache.commandInWritableDirectory` for the compile step so elm-stuff is reused across mutations to the same file

### Step-by-step implementation

#### Step 1: Add an Elm mutation generator module

Create `src/Mutator.elm` that uses `elm-syntax` to parse source and generate mutations.

```elm
module Mutator exposing (Mutation, generateMutations)

import Elm.Parser
import Elm.Syntax.Expression exposing (Expression(..))
import Elm.Syntax.Node exposing (Node(..))

type alias Mutation =
    { filePath : String
    , line : Int
    , description : String
    , mutatedSource : String
    }

generateMutations : String -> String -> List Mutation
generateMutations filePath source =
    case Elm.Parser.parse source of
        Ok rawFile ->
            rawFile
                |> findMutableExpressions
                |> List.concatMap (applyMutationOperators source)

        Err _ ->
            []
```

Mutation operators to implement (start with a few, expand later):

| Operator | Transform | What it catches |
|---|---|---|
| `negateCondition` | `if x` → `if not x` | Tests that only exercise one branch |
| `swapComparison` | `>` → `>=`, `==` → `/=` | Off-by-one, boundary conditions |
| `replaceWithIdentity` | `f x` → `x` | Tests that don't verify the transformation |
| `swapBooleanLiteral` | `True` → `False` | Hardcoded boolean checks |
| `removeListElement` | `[a, b, c]` → `[a, c]` | Tests that don't check list contents |
| `replaceArithmetic` | `+` → `-`, `*` → `//` | Arithmetic correctness |

#### Step 2: Add a mutation test runner script

Create `src/MutationTestRunner.elm` as a new elm-pages script:

```elm
module MutationTestRunner exposing (run)

run : Script
run =
    Script.withCliOptions programConfig task

task : Config -> BackendTask FatalError ()
task config =
    -- 1. Read all source files (reuse setupSourceFiles from PureTestRunner)
    Do.do (setupSourceFiles (Path.path ".")) <| \{ inputsByPath, depGraph } ->

    -- 2. Generate mutations for each source file
    Do.do (generateAllMutations inputsByPath) <| \mutations ->

    -- 3. For each mutation, run affected tests
    Do.do (runMutations config mutations depGraph) <| \results ->

    -- 4. Display report
    displayMutationReport results
```

#### Step 3: Implement the mutation loop

For each mutation:

```elm
runSingleMutation :
    Config
    -> DepGraph.Graph
    -> Mutation
    -> Cache.Monad MutationResult
runSingleMutation config depGraph mutation =
    let
        -- Which tests depend on the mutated file?
        affectedTests : Set String
        affectedTests =
            DepGraph.reverseDeps depGraph mutation.filePath

        -- Write the mutated project to a temp directory
        -- and compile + run affected tests
    in
    Cache.commandInWritableDirectory "lamdera"
        [ "make", "TestRunner.elm", "--output", "elm.js" ]
        projectHash
    <| \compiledHash ->
        -- Run tests in the compiled output, check for failures
        ...
```

#### Step 4: Add reverse dependency lookup to DepGraph

Currently `DepGraph.transitiveDeps` goes forward (file → what it depends on). Mutation testing needs the reverse: file → what depends on it (i.e., which tests are affected by changing this file).

```elm
reverseDeps : Graph -> String -> Set String
reverseDeps (Graph { deps }) targetFile =
    deps
        |> Dict.toList
        |> List.filter (\( _, fileDeps ) -> Set.member targetFile fileDeps)
        |> List.map Tuple.first
        |> Set.fromList
```

Or precompute a reverse index for efficiency.

#### Step 5: LLM-optimized output format

The report should be structured for LLM consumption:

```json
{
  "summary": { "killed": 47, "survived": 3, "score": "94%" },
  "survivingMutants": [
    {
      "file": "src/Route/Login.elm",
      "line": 34,
      "function": "validate",
      "operator": "swapComparison",
      "description": "Changed `String.length password > 8` to `String.length password >= 8`",
      "affectedTests": ["Login > validation > short password rejected"],
      "suggestion": "Add a test with a password of exactly length 8"
    }
  ],
  "killedMutants": [
    {
      "file": "src/Route/Login.elm",
      "line": 12,
      "operator": "negateCondition",
      "killedBy": "Login > validation > empty email rejected"
    }
  ]
}
```

### What makes this faster than traditional mutation testing

1. **No elm-test subprocess overhead** — tests run as pure functions in-process
2. **Dependency-guided test selection** — DepGraph already knows the graph, only run affected tests
3. **Content-addressed caching** — identical source hashes skip recompilation and test execution entirely
4. **Incremental compilation** — Lamdera only recompiles the mutated module + dependents
5. **Batch mutations per file** — reuse the same compilation workspace for multiple mutations to one file

### Files to create/modify in elm-build

| File | Description |
|---|---|
| `src/Mutator.elm` (new) | Elm-syntax based mutation generator |
| `src/MutationTestRunner.elm` (new) | elm-pages script that orchestrates mutation testing |
| `src/DepGraph.elm` (modify) | Add `reverseDeps` for affected-test lookup |
| `src/SampleTests.elm` (no change) | Existing tests serve as the test suite |

### PoC scope

For an initial PoC, keep it minimal:

1. **2-3 mutation operators** — `negateCondition` and `swapComparison` are the highest value
2. **Single file target** — mutate one source file, run all tests (skip DepGraph optimization initially)
3. **Recompile per mutation** — use `Cache.commandInWritableDirectory` with `lamdera make`
4. **Text output** — print surviving mutants to console, skip JSON report initially

This should be achievable in ~200-300 lines of Elm across `Mutator.elm` and `MutationTestRunner.elm`, building entirely on elm-build's existing Cache and DepGraph infrastructure.

### Future extensions

- **Coverage-guided mutation**: only mutate covered expressions (skip what you know is untested — you already know that from elm-pages `--coverage`)
- **Equivalent mutant detection**: some mutations produce semantically identical code (e.g., negating a condition that's always true). Use heuristics or type info to skip these.
- **Mutation testing in CI**: fail the build if mutation score drops below a threshold
- **LLM loop**: feed surviving mutants to an LLM, have it generate new tests, re-run mutation testing, repeat until mutation score target is hit
