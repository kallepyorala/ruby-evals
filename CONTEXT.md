# ruby_evals

The ubiquitous language for a Ruby library that evaluates AI-powered applications and agents — "test AI behaviour in Ruby like you test everything else." This glossary is binding on the `SPEC.md` and every design decision; it is a glossary only, not a spec.

## Language

**Case**:
One unit under test — a single `input` with optional `expected` output and `metadata`. The data-level unit; when run through the RSpec adapter, a Case *becomes* an RSpec example (the runtime layer), keeping the two layers distinct.
_Avoid_: Example (reserved for RSpec's runtime unit), Scenario (Cucumber/BDD baggage), Sample, Golden, Test

**Dataset**:
A collection of Cases — the *data* under evaluation. Distinct from a runtime "suite" of tests.
_Avoid_: Suite (implies a runtime grouping of tests, not data), Collection, Set

**Task**:
The arbitrary Ruby callable under evaluation — takes a Case's input and produces an output (typically by calling an LLM or agent). Often just a block. The core evaluates any callable; it is not coupled to any framework.
_Avoid_: Subject (reserved — collides with RSpec's `subject` DSL keyword), Target (promptfoo uses it for the model/provider), System-under-test

**Scorer**:
The core check contract — any callable taking the case context (input, output, expected) and returning a Score. Deterministic checks and LLM Judges are both Scorers; a pure pass/fail check is a Scorer whose Score sets `pass`.
_Avoid_: Evaluator (too broad — the whole library evaluates; also the Python-framework word), Metric (reserved for aggregates), Check

**Judge**:
An LLM-based Scorer (rubric + choices + choice→score). Every Judge is a Scorer; "Judge" is reserved for the LLM-backed kind.
_Avoid_: Grader (use for the pluggable judge *model* config, not the scorer), LLM-scorer

**Matcher** / **Assertion**:
The framework-integration *views* onto a Scorer — `Matcher` is the RSpec adapter, `Assertion` the Minitest adapter. Thin adapters over the one core Scorer contract, not core nouns themselves.

**Metric**:
An aggregate measure across results (accuracy, mean score, stderr) — a run-level number, not a per-case check.
_Avoid_: using "Metric" for the checker itself (DeepEval/Ragas usage — we use Scorer)

**Eval**:
The named definition bundling a Dataset, a Task, and Scorers — the thing you author ("write an eval"). The library's namesake noun.
_Avoid_: Suite (broader here than a test suite), Experiment (implies hosted experiment-tracking, which is out of scope), Benchmark

**Run**:
One execution of an Eval, producing a Report. Local-first and humble.
_Avoid_: Experiment (hosted-tracking baggage), Execution

**Trial**:
One repeated execution of a single Case within a Run (for nondeterminism). How repeats aggregate (epochs/reducers) is decided in #13; "Trial" names the unit.
_Avoid_: Epoch (Inspect's word for the *repeat count* mechanism, not one iteration), Repeat, Attempt

**Score**:
One Scorer's output on one Case/Trial — a small value object carrying `value`, `pass`, and `reason`.
_Avoid_: Verdict (LLM-judge jargon for the `pass` boolean we already carry)

**Result**:
The outcome for one Case within a Run — the Task's actual output plus the collection of Scores plus the case's overall pass/fail. One row of a Report.
_Avoid_: Outcome, Record

**Report**:
The output of a whole Run — all Results plus aggregate Metrics plus the overall pass/fail gate (`Report#passed?`). The CI artifact.
_Avoid_: Experiment, Summary
