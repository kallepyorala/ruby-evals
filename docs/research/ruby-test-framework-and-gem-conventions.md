# Ruby Test-Framework Extension & Gem Conventions

*Primary-source research on how well-regarded gems extend RSpec and Minitest, what conventions modern Ruby gems follow, and how an I/O-bound eval runner should handle concurrency — to produce concrete recommendations for a new Ruby-first AI-evals library (framework-agnostic core; optional RSpec/Minitest/Rails adapters).*

**Date:** 2026-07-17

---

## Executive summary

- **Matcher approach: model assertions as plain Ruby objects, then wrap them per framework.** Keep the eval assertions (score thresholds, judge verdicts, etc.) as framework-agnostic objects/predicates in the core. Expose them to RSpec via `RSpec::Matchers.define` for the simple/value cases and the **class-based matcher protocol** (`matches?`, `failure_message`, `failure_message_when_negated`, `description`, `supports_block_expectations?`) for stateful/block cases; expose the same objects to Minitest as a `Minitest::Assertions` module. This mirrors how shoulda-matchers serves both frameworks from one implementation. ([define-matcher](https://rspec.info/features/3-13/rspec-expectations/custom-matchers/define-matcher/), [rspec-expectations Matchers](https://rubydoc.info/gems/rspec-expectations/RSpec/Matchers), [shoulda-matchers](https://github.com/thoughtbot/shoulda-matchers))
- **Formatter/JUnit: own a neutral result model in the core; render JSON and JUnit XML from it.** Don't couple reporting to a test framework. Emit a GitLab/GitHub-compatible `testsuite/testcase` JUnit schema directly, and for users already inside RSpec/Minitest, document `rspec_junit_formatter` and `minitest-reporters` rather than reimplementing them. ([RSpec formatters](https://rspec.info/features/3-13/rspec-core/formatters/), [GitLab unit test reports](https://docs.gitlab.com/ci/testing/unit_test_reports/))
- **Gem structure: single gem with optional `require`s / gated adapters — not a monorepo (yet).** Ship one gem whose core loads with zero heavy deps; put RSpec/Minitest/Rails glue behind separate requirable files that no-op unless the host framework is present. RSpec's split into `rspec-core`/`rspec-expectations`/`rspec-mocks`/`rspec-support` is the heavyweight precedent to graduate to only if adapters grow large. ([rspec-expectations repo note](https://github.com/rspec/rspec-expectations))
- **Lint: `standard`.** Zero-config, bikeshed-proof, RuboCop-based, and `bundle gem --linter=standard` scaffolds it; used by AWS, Datadog, thoughtbot, Evil Martians. RuboCop remains the choice only if you need a custom, curated cop set. ([standard](https://github.com/standardrb/standard))
- **CLI: Thor.** Ubiquitous (~1.1B downloads), self-documenting, minimal ceremony. `dry-cli` (class-per-command, used by Hanami) is the cleaner-architecture alternative; plain `OptionParser` if you want zero CLI deps. ([Thor](https://github.com/rails/thor), [dry-cli](https://github.com/dry-rb/dry-cli))
- **Typing: ship a light `sig/` RBS, skip Sorbet.** `bundle gem` now scaffolds `sig/NAME.rbs` by default (verified below), but most popular gems still ship *no* inline signatures — the community maintains them out-of-tree in `gem_rbs_collection`. A curated public-API RBS is a good DX signal without the maintenance burden of full coverage or Sorbet sigils. ([gem_rbs_collection](https://github.com/ruby/gem_rbs_collection))
- **Loading: Zeitwerk, gem convention.** `Zeitwerk::Loader.for_gem` + snake_case→CamelCase, one constant per file. ([zeitwerk](https://github.com/fxn/zeitwerk))
- **Concurrency: threads + `concurrent-ruby` (a bounded pool), NOT async/fibers.** The runner executes *arbitrary user callables that may hit Rails/ActiveRecord*. ActiveRecord's connection pool checks out connections per execution context, which is **per-thread by default**; the fiber path (`isolation_level = :fiber`, Falcon/async) is opt-in and has documented memory/perf regressions. A `Concurrent::FixedThreadPool` is the compatible, boring choice for I/O-bound eval work. ([concurrent-ruby](https://github.com/ruby-concurrency/concurrent-ruby), [Rails #52617](https://github.com/rails/rails/issues/52617), [async](https://github.com/socketry/async))

---

## 1. RSpec extension patterns

RSpec ships as a **monorepo of cooperating gems** — `rspec-core`, `rspec-expectations`, `rspec-mocks`, `rspec-support` (the old per-repo repositories now redirect to `rspec/rspec`). Current `rspec-core` is **3.13.6**. ([rspec-expectations repo](https://github.com/rspec/rspec-expectations), [rubygems](https://rubygems.org/gems/rspec-core))

### 1.1 Custom matchers

**The `RSpec::Matchers.define` DSL** (RSpec 3.13) is the ergonomic path. Blocks/helpers available: `match`, `match_when_negated`, `failure_message`, `failure_message_when_negated`, `description`, `chain` (fluent interface), `supports_block_expectations`, plus `RSpec::Matchers.alias_matcher`. Helper methods can be defined inline or mixed in with `include`. ([define-matcher](https://rspec.info/features/3-13/rspec-expectations/custom-matchers/define-matcher/))

```ruby
RSpec::Matchers.define :be_a_multiple_of do |expected|
  match do |actual|
    actual % expected == 0
  end
  failure_message do |actual|
    "expected that #{actual} would be a multiple of #{expected}"
  end
end
```

**Block expectations** (`expect { ... }.to my_matcher`) require opting in — either the `supports_block_expectations` shortcut in the DSL, or a `supports_block_expectations?` method returning true: *"When you wish to support block expectations ... with your custom matchers you must specify this."* ([define-block-matcher](https://rspec.info/features/3-13/rspec-expectations/custom-matchers/define-block-matcher/))

**The class-based matcher protocol** is the escape hatch for stateful matchers (a judge that runs an LLM call, accumulates per-field errors, etc.). A matcher is any object responding to:

- `matches?(actual)` → truthy/falsy (required)
- `failure_message` → string for `expect(..).to` failures (required in practice)
- `failure_message_when_negated` → string for `expect(..).not_to` failures
- `description` → string used in generated docs/output
- `supports_block_expectations?` → declare block support
- optionally `include RSpec::Matchers::Composable` and compare nested values with `values_match?(expected, actual)` so the matcher composes inside `and`/`or`/collection matchers. ([rspec-expectations Matchers](https://rubydoc.info/gems/rspec-expectations/RSpec/Matchers))

`rspec-json_expectations` (its `include_json` matcher) is a good exemplar of the class-based route: it builds a matcher object via its own `MatcherFactory#define_matcher`, parses/validates input, delegates comparison to a `JsonTraverser`, accumulates path-scoped errors, and renders them through a dedicated failure presenter — i.e. a hand-rolled `matches?`/`failure_message` object rather than the DSL. ([source](https://github.com/waterlink/rspec-json_expectations))

### 1.2 Example-group DSL extension

Configured through `RSpec.configure` (`RSpec::Core::Configuration`):

- **`config.include(mod, *filters)`** — methods become available on **examples** (instances / the `it` block body). Use for helpers you call inside tests.
- **`config.extend(mod, *filters)`** — methods become available on **example groups** (the `describe`/`context` class body). *"Similar to include, but behavior is added to example groups, which are classes, rather than the examples."* Use for a DSL like a top-level `eval_case "..."` macro.
- Both accept **metadata filters** so extension is scoped, e.g. `config.include AuthHelpers, type: :request` or `config.extend EvalDSL, type: :eval`. `config.define_derived_metadata(file_path: %r{/spec/evals/}) { |m| m[:type] = :eval }` auto-tags files so users don't repeat metadata. ([Configuration](https://rubydoc.info/gems/rspec-core/RSpec/Core/Configuration))
- **Shared contexts/examples**: define with `RSpec.shared_context`/`shared_examples`, pull in with `include_context`/`include_examples`, or auto-include via `config.include_context "name", :tag`. ([Configuration](https://rubydoc.info/gems/rspec-core/RSpec/Core/Configuration))

shoulda-matchers (**8.0.1**, 2026-06-12) is the canonical example: one `Shoulda::Matchers.configure { |c| c.integrate { |with| with.test_framework :rspec; with.library :rails } }` block, matchers exposed only in metadata-scoped groups (model matchers in `type: :model`, controller matchers in `type: :controller`), and the *same* matcher library is offered to Minitest by flipping `with.test_framework :minitest`. ([shoulda-matchers](https://github.com/thoughtbot/shoulda-matchers))

### 1.3 Formatters

Custom output goes through the formatter API: a class registers with `RSpec::Core::Formatters.register(formatter_class, *notifications)` and implements handler methods for the notifications it subscribes to (`start`, `example_passed`, `example_failed`, `example_pending`, `dump_summary`, `stop`, `seed`, `close`, …). Selected at runtime with `--format`/`-f` and directed to a file with `--out`/`-o` (multiple formatters can run at once). ([RSpec formatters](https://rspec.info/features/3-13/rspec-core/formatters/))

The built-in **`JsonFormatter`** is the reference implementation and a ready-made machine-readable output:

```ruby
Formatters.register self, :message, :dump_summary, :dump_profile, :stop, :seed, :close
```

It emits a hash keyed by `:version`, `:examples` (each with `id`, `description`, `status`, `file_path`, `line_number`, `run_time`, exception info), `:summary` (`duration`, `example_count`, `failure_count`, `pending_count`, `errors_outside_of_examples_count`), `:summary_line`, `:seed`, `:profile`. Built-in text formatters are `progress` (default) and `documentation`. ([json_formatter.rb](https://github.com/rspec/rspec-core/blob/main/lib/rspec/core/formatters/json_formatter.rb)) RSpec has **no built-in JUnit formatter** — that's the `rspec_junit_formatter` gem (§5).

### 1.4 aggregate_failures & tag filtering

- **`aggregate_failures`** wraps several expectations so *"expectation failures will not immediately abort ... instead, the failures will be aggregated into a single exception"* — a `RSpec::Expectations::MultipleExpectationsNotMetError` — raised at block end. Available as a block helper (`aggregate_failures("label") do ... end`) or as example/config metadata (`it "...", :aggregate_failures`). Caveat relevant to a concurrent runner: it uses **thread-local** state, so *"expectation failures occurring in separate threads will abort normally rather than aggregate."* This is a natural fit for eval assertions where you want every metric's result, not just the first failure. ([aggregating-failures](https://rspec.info/features/3-13/rspec-expectations/aggregating-failures/))
- **Tag filtering**: `--tag NAME` / `--tag NAME:value` on the CLI, and `config.filter_run_when_matching :focus` (applies the filter *only if some example matches*, unlike `filter_run` which always applies). ([Configuration](https://rubydoc.info/gems/rspec-core/RSpec/Core/Configuration))

---

## 2. Minitest extension

Current Minitest is **6.0.6** (README targets Ruby 3.2+). ([rubygems](https://rubygems.org/gems/minitest), [minitest](https://github.com/minitest/minitest))

**Plugin system (load-path discovery).** *"To define a plugin, add a file named `minitest/XXX_plugin.rb` to your project/gem. That file must be discoverable via ruby's LOAD_PATH ... Minitest will find and require that file using `Gem.find_files`."* It then calls `plugin_XXX_init(options)` at startup and `plugin_XXX_options(opts, options)` (passed the `OptionParser` and options hash) during option parsing:

```ruby
# minitest/bogus_plugin.rb
module Minitest
  def self.plugin_bogus_options(opts, options)
    opts.on "--myci", "Report results to my CI" do
      options[:myci] = true
    end
  end

  def self.plugin_bogus_init(options)
    self.reporter << MyCI.new(options) if options[:myci]
  end
end
```

This is how a gem hooks the runner without the user editing a config file — the ideal place to register an eval reporter/CI export. ([minitest README](https://github.com/minitest/minitest))

**Custom assertions.** Assertions live in `Minitest::Assertions`; extend by putting helpers in a module that's mixed into test classes (the same module can be shared with the RSpec adapter's logic). Spec-style expectations are thin wrappers: a `_(obj).must_equal x` / `value(obj).must_equal x` form built on the assertions. ([minitest README](https://github.com/minitest/minitest))

**Two test styles, one runner.** `Minitest::Test` with `test_*` methods and `assert_*`/`refute_*`; or `Minitest::Spec` with `describe`/`it` and `must_*`/`wont_*` expectations (which *"must be wrapped in a value call (eg `_`)"*). An adapter should support both — register assertions on `Minitest::Assertions` (reaches both) and, if offering a spec DSL macro, hang it off `Minitest::Spec::DSL`.

**Reporters.** Minitest uses a **composite reporter**; add reporters to it during the plugin `init` phase. Custom reporters subclass `Minitest::AbstractReporter` and override `start`, `record`, `report`, `passed?`. The `minitest-reporters` gem is the ecosystem's pluggable-reporter layer (§5). ([minitest README](https://github.com/minitest/minitest))

---

## 3. Modern gem conventions

### 3.1 Zeitwerk loading & gem skeleton

`Zeitwerk` (**2.8.2**; README's 2.7 line requires Ruby 3.2+) is the de-facto autoloader. Convention: snake_case file → CamelCase constant (`lib/my_gem/bar_baz.rb` → `MyGem::BarBaz`), one constant per file, acronyms via `loader.inflector.inflect(...)`. Canonical gem entrypoint:

```ruby
# lib/my_gem.rb
require "zeitwerk"
loader = Zeitwerk::Loader.for_gem
loader.setup
module MyGem; end
# loader.eager_load  # optional; recommended before threading (see §4)
```

Thread-safety: *autoloading itself is thread-safe*, but *"In order to reload safely, no other thread can be autoloading or reloading concurrently"* — reloading is a dev-only concern; a library that fans work across threads should prefer `eager_load` up front. ([zeitwerk](https://github.com/fxn/zeitwerk))

**`bundle gem` skeleton (verified).** Running `bundle gem demo_gem --test=rspec --linter=standard --ci=github` on **Bundler 4.0.16** produces: `lib/demo_gem.rb`, `lib/demo_gem/version.rb`, `demo_gem.gemspec` (`required_ruby_version >= 3.2.0`), `Gemfile`, `Rakefile`, `README.md`, `bin/console`, `bin/setup`, `.rspec` + `spec/`, `.standard.yml`, `.github/workflows/main.yml`, **and `sig/demo_gem.rbs` by default**:

```rbs
module DemoGem
  VERSION: String
end
```

Flags: `--test` (`minitest`/`rspec`/`test-unit`), `--linter` (`rubocop`/`standard`), `--ci` (`github`/`gitlab`/`circle`), plus `--exe`, `--mit`, `--coc`, `--changelog`. ([bundle-gem man page](https://bundler.io/man/bundle-gem.1.html))

### 3.2 Lint: standard vs rubocop

- **`standard`** — an opinionated, unconfigurable RuboCop wrapper ("bikeshed-proof linter and formatter"), maintained by Test Double, extensible only through `lint_roller` plugins (`standard-rails`, `standard-sorbet`). Adopters listed in its README include AWS, Datadog, Brave, thoughtbot, Evil Martians. Run via `standardrb` / `rake standard`. ([standard](https://github.com/standardrb/standard))
- **`rubocop`** (**1.88.2**, ~718M downloads) — the underlying engine; the choice when you want a bespoke curated cop set (Shopify/`rubocop-shopify`, Rails core, most large apps). ([rubygems](https://rubygems.org/gems/rubocop))

For a small, contributor-friendly library, `standard` removes style debate entirely and matches `bundle gem --linter=standard`.

### 3.3 CLI: Thor vs dry-cli vs OptionParser

- **Thor** (**1.5.0**, ~1.1B downloads) — *"a simple and efficient tool for building self-documenting command line utilities."* Subclass `Thor`, `desc` + a method per command, `method_option`/`class_option`, subcommands. Lowest-friction, enormous install base (the standard choice for gem CLIs and Rails/Bundler-style tooling). ([Thor](https://github.com/rails/thor), [rubygems](https://rubygems.org/gems/thor))
- **dry-cli** (**1.4.1**) — class-per-command (`Dry::CLI::Command` subclasses, `register`, arguments/options DSL); used by **Hanami**. Cleaner separation and testable command objects, smaller ecosystem. ([dry-cli](https://github.com/dry-rb/dry-cli))
- **OptionParser** — stdlib, zero deps; fine for a one-verb CLI but you hand-roll subcommands/help.

Recommendation: **Thor** for breadth of subcommands (`run`, `list`, `report`) with self-documenting help at near-zero cost; drop to OptionParser only if minimizing dependencies is a hard goal.

### 3.4 Typing: YARD, RBS, Sorbet

- **RBS** is the standard signature syntax and ships with Ruby, but shipping inline `sig/` is still **the exception, not the rule** among popular gems — `gem_rbs_collection` exists precisely as *"a community-managed collection of RBS files for gems that ship without RBS,"* consumed via `rbs collection install` / `rbs_collection.yaml`. So the realistic bar is: a small, curated `sig/` covering your public API (which `bundle gem` already stubs) is a nice DX signal; full coverage is not expected. ([gem_rbs_collection](https://github.com/ruby/gem_rbs_collection))
- **Sorbet** (`sig`/`T::Sig`, inline `# typed:` sigils) is used at scale (Shopify, Stripe) but is heavier and more invasive; adoption in general-purpose OSS gems is comparatively rare. Not recommended for this library.
- **YARD** docstrings remain the common documentation convention and pair well with a hand-written RBS.

### 3.5 Monorepo vs single gem with optional requires

Two established patterns:

- **Monorepo of multiple gems** — RSpec (`rspec-core`, `rspec-expectations`, `rspec-mocks`, `rspec-support`, meta-gem `rspec`). Buys independent versioning and lets users install only what they need; costs release/CI coordination. ([rspec-expectations](https://github.com/rspec/rspec-expectations))
- **Single gem gating optional integration code** — shoulda-matchers ships one gem and *activates* the RSpec vs Minitest / Rails vs plain-ActiveModel paths at runtime via its `configure`/`integrate` block, loading framework glue only when asked. ([shoulda-matchers](https://github.com/thoughtbot/shoulda-matchers))

For a young library, start as **one gem** with the core dependency-light and adapters behind `require "ruby_evals/rspec"` / `require "ruby_evals/minitest"` files that detect and hook the host framework (Minitest's `minitest/*_plugin.rb` auto-discovery, §2, makes the Minitest adapter especially clean). Split into a monorepo only if an adapter grows heavy dependencies (e.g. a Rails adapter pulling ActiveRecord) that you don't want in the core's dependency graph.

---

## 4. Runner concurrency for an I/O-bound eval runner

The runner executes **arbitrary user-supplied callables** (a prompt call, a judge, an assertion) that are I/O-bound (LLM/network) and *may* touch Rails/ActiveRecord. Two models:

**Threads + `concurrent-ruby` (1.3.7).** Provides `FixedThreadPool`/`CachedThreadPool`, `Concurrent::Promises`, and thread-safe `Concurrent::Map`. It is battle-tested (Rails depends on it) and, crucially, **matches how Rails is built to run concurrently**:

- Rails' **Executor** wraps any framework-invoked app code; among its default callbacks it *"return[s] acquired Active Record connections to the pool"* per unit of work, and the **load interlock** ensures autoloading/reloading is coordinated so a constant isn't swapped mid-execution. A library invoking app code *"should wrap it with a call to the executor."* ([Rails threading guide](https://guides.rubyonrails.org/threading_and_code_execution.html))
- ActiveRecord's connection pool checks out connections **per execution context, thread by default** — the ordinary, well-supported path for Puma/Sidekiq-style thread concurrency.

**Async / fibers (socketry `async` 2.42.0).** A fiber reactor on `io-event`; great scalability for non-blocking I/O, but it demands the whole stack be **fiber-aware**: blocking C extensions and non-fiber-aware drivers stall the reactor, and shared per-thread state is *shared across all fibers on that thread* unless isolated. For ActiveRecord specifically, correct pooling under fibers requires `config.active_support.isolation_level = :fiber` (the Falcon path) so multiple fibers of one thread each check out their own connection — and that path has **documented memory-leak/performance regressions** (`rails/rails#52617`). Default Rails (`:thread`) will hand many fibers the *same* connection. ([async](https://github.com/socketry/async), [Rails #52617](https://github.com/rails/rails/issues/52617), [connection_pool.rb](https://github.com/rails/rails/blob/main/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb))

**Recommendation: threads + a bounded `Concurrent::FixedThreadPool`.** For running user callables that may hit Rails/ActiveRecord, threads are the compatible default: they align with ActiveRecord's per-thread connection checkout, work with arbitrary blocking user code, and let the runner (optionally) wrap each callable in `Rails.application.executor.wrap { ... }` when Rails is present. `eager_load` (or Zeitwerk `loader.eager_load`) before spawning workers avoids concurrent-autoload hazards. Reserve fibers/async as an *opt-in* backend for advanced users on a fully fiber-safe stack — do not make it the default for a library that can't control what the user's callable does.

---

## 5. JUnit XML for GitHub Actions & GitLab CI

**The schema CI consumers read.** GitLab (`artifacts:reports:junit`) parses a `testsuites` → `testsuite` (`name`, `time`) → `testcase` tree, where each `testcase` carries `classname` ("Displayed as the suite name in UI"), `name`, `file` ("File path where the test is defined"), and `time`, with `failure` / `error` / `skipped` children and `system-out` ("Only parsed from `testcase` elements"). ([GitLab unit test reports](https://docs.gitlab.com/ci/testing/unit_test_reports/)) GitHub Actions has **no built-in JUnit ingestion** — the same file format is consumed by community actions/annotators, so producing standards-compliant `testsuite/testcase` XML covers both.

```yaml
# GitLab: the documented Ruby/RSpec example
ruby:
  script: bundle exec rspec --format RspecJunitFormatter --out rspec.xml
  artifacts:
    when: always
    reports:
      junit: rspec.xml
```

**RSpec → `rspec_junit_formatter`** (0.6.0). A formatter plugin selected with `--format RspecJunitFormatter --out rspec.xml` (or in `.rspec`), combinable with other formatters, with `$TEST_ENV_NUMBER` support for parallel runs and stdout/stderr capture. Explicitly targets *"Jenkins, Buildkite, CircleCI, Gitlab, and probably more."* ([rspec_junit_formatter](https://github.com/sj26/rspec_junit_formatter))

**Minitest → `minitest-reporters`.** Its `JUnitReporter` is wired via the composite reporter: `Minitest::Reporters.use! [Minitest::Reporters::SpecReporter.new, Minitest::Reporters::JUnitReporter.new]` (alongside `DefaultReporter`, `SpecReporter`, `ProgressReporter`, `HtmlReporter`, `MeanTimeReporter`, etc.). ([minitest-reporters](https://github.com/minitest-reporters/minitest-reporters))

For this library: since the core owns a neutral result model (§1.3 reasoning), render JUnit XML directly to the `testsuite/testcase` schema above (one `testsuite` per eval suite, one `testcase` per case, `failure` with the assertion/judge message, `system-out` for the model transcript). Only fall back to `rspec_junit_formatter`/`minitest-reporters` for users who want their eval run reported *through* an existing test-framework invocation.

---

## Recommendations for THIS library

Judged against the design principles — Ruby-first, DX over breadth, framework-agnostic core with optional adapters, conventional Ruby objects/blocks/modules/matchers, strong-where-useful docs/typing, pleasant to contribute to:

| Area | Decision |
|---|---|
| **Assertions/matchers** | Core = plain Ruby predicate/result objects (no framework). RSpec adapter: `RSpec::Matchers.define` for simple value matchers, class-based protocol (`matches?`/`failure_message`/`supports_block_expectations?`, `include Composable`) for stateful/judge matchers. Minitest adapter: a `Minitest::Assertions` module wrapping the same objects. One implementation, two thin adapters (shoulda-matchers model). |
| **Group DSL** | Offer an opt-in `eval …` DSL via `config.extend Mod, type: :eval` + `define_derived_metadata` to auto-tag `spec/evals/`. Keep it small; don't invent a parallel test runner. |
| **Reporting** | Neutral result model in core → JSON reporter (mirror RSpec's `JsonFormatter` keys for familiarity) + a direct JUnit XML reporter (GitLab/GitHub `testsuite/testcase` schema). Use `aggregate_failures` semantics so all metrics report, not just the first failure. |
| **Gem structure** | **Single gem**, dependency-light core, adapters behind `require "ruby_evals/rspec"` / `ruby_evals/minitest` (+ Minitest `minitest/ruby_evals_plugin.rb` auto-discovery). Graduate to an RSpec-style monorepo only if a Rails/AR adapter would pollute the core dep graph. |
| **Loading** | Zeitwerk via `Zeitwerk::Loader.for_gem`; expose `eager_load` for pre-threading warmup. |
| **Lint** | `standard` (matches `bundle gem --linter=standard`; zero bikeshedding for contributors). |
| **CLI** | **Thor** for `run`/`list`/`report` subcommands with free self-documenting help. |
| **Typing** | Ship a curated public-API `sig/*.rbs` (already scaffolded by `bundle gem`); add to `gem_rbs_collection` later if wanted. No Sorbet. YARD on the public API. |
| **Concurrency** | **Threads + `Concurrent::FixedThreadPool`**, bounded. When Rails is present, wrap each user callable in `Rails.application.executor.wrap`; `eager_load` before spawning. Fibers/async = opt-in advanced backend only. |
| **Ruby/tooling baseline** | Ruby ≥ 3.2 (Zeitwerk 2.8 / minitest 6 / `bundle gem` default). |

---

## Sources

- RSpec custom matcher DSL — https://rspec.info/features/3-13/rspec-expectations/custom-matchers/define-matcher/
- RSpec block-expectation matchers — https://rspec.info/features/3-13/rspec-expectations/custom-matchers/define-block-matcher/
- RSpec class-based matcher protocol (`RSpec::Matchers`) — https://rubydoc.info/gems/rspec-expectations/RSpec/Matchers
- RSpec `RSpec::Core::Configuration` (extend/include/filter_run_when_matching/define_derived_metadata/include_context) — https://rubydoc.info/gems/rspec-core/RSpec/Core/Configuration
- RSpec formatters (register API, --format/--out) — https://rspec.info/features/3-13/rspec-core/formatters/
- RSpec built-in `JsonFormatter` source — https://github.com/rspec/rspec-core/blob/main/lib/rspec/core/formatters/json_formatter.rb
- RSpec `aggregate_failures` — https://rspec.info/features/3-13/rspec-expectations/aggregating-failures/
- rspec-expectations repo (monorepo layout note) — https://github.com/rspec/rspec-expectations
- shoulda-matchers (dual framework config, metadata-scoped include) — https://github.com/thoughtbot/shoulda-matchers
- rspec-json_expectations (class-based matcher exemplar) — https://github.com/waterlink/rspec-json_expectations
- Minitest README (plugin discovery, assertions, spec DSL, reporters) — https://github.com/minitest/minitest
- Minitest gem (v6.0.6) — https://rubygems.org/gems/minitest
- Zeitwerk (conventions, thread-safety) — https://github.com/fxn/zeitwerk
- `bundle gem` man page — https://bundler.io/man/bundle-gem.1.html
- standard (linter) — https://github.com/standardrb/standard
- rubocop gem — https://rubygems.org/gems/rubocop
- Thor — https://github.com/rails/thor / https://rubygems.org/gems/thor
- dry-cli — https://github.com/dry-rb/dry-cli
- gem_rbs_collection (RBS reality) — https://github.com/ruby/gem_rbs_collection
- concurrent-ruby — https://github.com/ruby-concurrency/concurrent-ruby
- socketry/async — https://github.com/socketry/async
- Rails threading & code execution guide (Executor, load interlock, connection return) — https://guides.rubyonrails.org/threading_and_code_execution.html
- Rails fiber isolation memory/perf regression (#52617) — https://github.com/rails/rails/issues/52617
- ActiveRecord ConnectionPool source — https://github.com/rails/rails/blob/main/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb
- GitLab unit test reports (JUnit schema) — https://docs.gitlab.com/ci/testing/unit_test_reports/
- rspec_junit_formatter — https://github.com/sj26/rspec_junit_formatter
- minitest-reporters — https://github.com/minitest-reporters/minitest-reporters
