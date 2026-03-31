## Plan: Mutation Testing PoC in elm-build

### Why elm-build is ideal for this

elm-build's `PureTestRunner` already has everything mutation testing needs:

- **Tests run via elm-interpreter** — code under test is evaluated from source at runtime, not compiled by elm/lamdera. The script itself is compiled once, but test execution interprets Elm source directly. This means **mutating source = mutating behavior with zero recompilation**.
- **Source files are already in memory** — `setupSourceFiles` reads all `.elm` files via `BackendTask.File.rawFile`
- **`elm-syntax` is already a dependency** (`stil4m/elm-syntax: 7.3.9`) — parse Elm source into a full AST, apply mutation transforms, and feed the mutated AST/source back to the interpreter. All in pure Elm, all in memory.
- **Dependency graph** (`DepGraph`) knows which tests transitively depend on which source files — so you only re-run affected tests per mutation
- **Content-addressed caching** (`Cache`) means identical mutation+test combinations are never re-executed

### Architecture

The mutation loop is **entirely in-memory** — no disk I/O, no recompilation, no temp directories:

```
Source files (already in memory from setupSourceFiles)
        |
        v
  Parse with elm-syntax → AST
        |
        v
  Generate mutations (AST → AST transforms)
  Each mutation = { file, location, operator, mutatedAST }
        |
        v
  For each mutation:
    1. Feed mutated source/AST to the interpreter
    2. Run affected tests (DepGraph reverse lookup tells you which ones)
    3. Did any test fail?
       - Yes → mutant killed (test suite is strong here)
       - No  → mutant survived (test suite is weak here)
        |
        v
  Report: list of surviving mutants with location + description
```

**Key insight**: since the interpreter evaluates source at runtime, the mutation loop is just:

```
parse → transform AST → interpret → check test results → next mutation
```

No `lamdera make`. No writing files to disk. No subprocesses. This is as fast as mutation testing can get — it's bounded by interpreter speed, not compilation speed.

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
    , column : Int
    , operator : String
    , description : String
    , apply : RawFile -> RawFile  -- AST → AST transform
    }

generateMutations : String -> String -> List Mutation
generateMutations filePath source =
    case Elm.Parser.parse source of
        Ok rawFile ->
            rawFile
                |> findMutableExpressions
                |> List.concatMap (applyMutationOperators filePath)

        Err _ ->
            []
```

Mutation operators (start with highest-value, expand later):

| Operator | Transform | What it catches |
|---|---|---|
| `negateCondition` | `if x` → `if not x` | Tests that only exercise one branch |
| `swapComparison` | `>` → `>=`, `==` → `/=` | Off-by-one, boundary conditions |
| `replaceWithIdentity` | `f x` → `x` | Tests that don't verify the transformation |
| `swapBooleanLiteral` | `True` → `False` | Hardcoded boolean checks |
| `replaceArithmetic` | `+` → `-`, `*` → `//` | Arithmetic correctness |
| `removeListElement` | `[a, b, c]` → `[a, c]` | Tests that don't check list contents |

For the PoC, just `negateCondition` and `swapComparison` are enough to prove the concept.

#### Step 2: Add a mutation test runner script

Create `src/MutationTestRunner.elm` as a new elm-pages script:

```elm
module MutationTestRunner exposing (run)

run : Script
run =
    Script.withCliOptions programConfig task

task : Config -> BackendTask FatalError ()
task config =
    -- 1. Read all source files into memory
    Do.do (readSourceFiles (Path.path ".")) <| \sources ->

    -- 2. Parse each source file and generate mutations
    let
        mutations =
            sources
                |> List.concatMap (\( path, content ) ->
                    Mutator.generateMutations path content
                )
    in

    -- 3. For each mutation, interpret the mutated source and run tests
    Do.do (runAllMutations sources mutations) <| \results ->

    -- 4. Display report
    displayMutationReport results
```

#### Step 3: Implement the mutation loop

The core loop — for each mutation, swap the AST, interpret, and run affected tests:

```elm
runSingleMutation :
    Dict String String          -- all source files (path → content)
    -> DepGraph.Graph
    -> List Test.Runner.Runner  -- the test suite
    -> Mutation
    -> MutationResult
runSingleMutation sources depGraph runners mutation =
    let
        -- Apply the mutation: replace the original source with mutated version
        mutatedSources =
            Dict.update mutation.filePath
                (\_ -> Just (mutation.apply originalAST |> prettyPrint))
                sources

        -- Which tests are affected by this file change?
        affectedRunners =
            runners
                |> List.filter (\runner ->
                    let
                        testFile = resolveTestFile runner.labels
                    in
                    Set.member mutation.filePath
                        (DepGraph.transitiveDeps depGraph testFile)
                )

        -- Run affected tests against the mutated source via interpreter
        results =
            affectedRunners
                |> List.map (\runner -> interpretAndRun mutatedSources runner)

        anyFailed =
            List.any .failed results
    in
    if anyFailed then
        Killed { mutation = mutation, killedBy = List.filter .failed results }
    else
        Survived { mutation = mutation, testsRun = List.length results }
```

The critical line is `interpretAndRun mutatedSources runner` — this feeds the mutated source to the interpreter instead of using compiled code. The exact API depends on how the elm-interpreter is integrated, but the concept is: give it a modified source environment, ask it to evaluate the test.

#### Step 4: Add reverse dependency lookup to DepGraph

Currently `DepGraph.transitiveDeps` goes forward (file → what it depends on). Mutation testing needs the reverse: file → what depends on it (i.e., which tests are affected by changing this file).

```elm
reverseDeps : Graph -> String -> Set String
reverseDeps (Graph { deps }) targetFile =
    -- For every file in the graph, check if targetFile is in its transitive deps
    deps
        |> Dict.keys
        |> List.filter (\file ->
            Set.member targetFile (transitiveDeps (Graph { deps }) file)
        )
        |> Set.fromList
```

Or precompute a reverse adjacency list from the forward graph for efficiency.

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

### Why this is so fast

No recompilation. The interpreter evaluates source directly, so mutations are instant:

1. **AST transform** — microseconds (swap a node in the syntax tree)
2. **Interpret mutated source** — milliseconds (interpreter evaluates the changed function)
3. **Run affected tests** — milliseconds (pure function calls, filtered by DepGraph)
4. **No disk I/O** — sources stay in memory, no temp files, no file copies
5. **No subprocess overhead** — everything runs in-process

Compare to traditional mutation testing (e.g., Stryker, PIT):
- Traditional: mutate → write file → recompile → spawn test process → collect results → repeat
- elm-build: mutate AST in memory → interpret → check results → repeat

The bottleneck shifts from compilation to interpretation speed, which is orders of magnitude faster for the "mutate one expression, run a few tests" loop.

### Files to create/modify in elm-build

| File | Description |
|---|---|
| `src/Mutator.elm` (new) | elm-syntax based mutation generator — parse, transform, generate variants |
| `src/MutationTestRunner.elm` (new) | elm-pages script that orchestrates the mutation loop |
| `src/DepGraph.elm` (modify) | Add `reverseDeps` for affected-test lookup |
| `src/SampleTests.elm` (no change) | Existing tests serve as the test suite to check mutations against |

### PoC scope

Keep it minimal to prove the concept:

1. **2 mutation operators** — `negateCondition` and `swapComparison`
2. **Single file target** — mutate one source file (e.g., a module that `SampleTests` imports)
3. **All tests run per mutation** — skip the DepGraph filtering initially for simplicity
4. **Text output** — print surviving mutants to console with file/line/description
5. **No caching** — add Cache integration after the core loop works

This should be ~200-300 lines of Elm across `Mutator.elm` and `MutationTestRunner.elm`.

### Future extensions

- **Coverage-guided mutation**: only mutate covered expressions (skip what you know is untested — use the lcov data from elm-pages `--coverage`)
- **Equivalent mutant detection**: some mutations produce semantically identical code (e.g., negating a condition that's always true). Use heuristics or type info to skip these.
- **Cache integration**: hash (mutated source + test labels) → cache mutation test results across runs. Only re-test mutations when relevant source changes.
- **Mutation testing in CI**: fail the build if mutation score drops below a threshold
- **LLM loop**: feed surviving mutants to an LLM, have it generate new tests, re-run mutation testing, repeat until mutation score target is hit
- **Parallel mutation evaluation**: since each mutation is independent, evaluate multiple mutations concurrently using elm-build's `combineBy` parallelism
