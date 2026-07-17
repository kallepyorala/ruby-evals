# Core API & DSL prototype — 3 competing sketches

Throwaway API sketches for [ticket #9](https://github.com/kallepyorala/ruby-evals/issues/9) — the central design decision: **what does the primary public API look like, and how do the runner layer and the test-framework layer relate?**

All three write the *same* canonical scenario — the **autonomous-todo agent, Rung 1 (`todo → plan`)** — and each shows: defining Cases, invoking the Task, a deterministic Scorer, an LLM Judge, repeated Trials, reading the Report, and the *simplest possible eval*. Vocabulary follows [`CONTEXT.md`](../../../CONTEXT.md).

They are not runnable — the library doesn't exist yet. They exist to be reacted to.

| # | Style | Center of gravity | You run it with | Best when |
|---|-------|-------------------|-----------------|-----------|
| [01](01_standalone_runner.rb) | **Standalone runner** | `Eval`/`Dataset`/`Report` objects; RSpec absent | `ruby eval.rb` / rake task | eval scripts, non-RSpec shops, batch runs |
| [02](02_rspec_native.rb) | **RSpec-native** | RSpec; Cases *are* examples, Scorers *are* matchers | `rspec` | teams who already gate CI on `rspec` |
| [03](03_layered.rb) | **Layered** | Core objects are the spine; RSpec DSL is a thin adapter over them | either | want both front doors sharing one result model |

## The fork

- **Standalone** is the cleanest *runner spine* but leaves RSpec users to bolt it on.
- **RSpec-native** is the friendliest *on-ramp* for the persona (Rails dev who writes RSpec) but risks trapping the evaluation logic inside RSpec, hostile to CI-script / non-RSpec use.
- **Layered** keeps one Scorer contract + one result model and offers *both* doors — at the cost of more surface area and the discipline of keeping the adapter genuinely thin.

Positioning ([#6](https://github.com/kallepyorala/ruby-evals/issues/6)) already leans **layered** ("runner spine + RSpec on-ramp, one result model / one scorer contract"). The open questions this prototype exists to settle: does the layering *read well* in code, what exactly is the core object surface, and **which style leads the README**?
