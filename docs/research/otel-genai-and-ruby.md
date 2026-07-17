# OpenTelemetry GenAI Semantic Conventions & the Ruby OTel SDK

*Primary-source investigation to inform OPTIONAL OpenTelemetry instrumentation of an eval run in a new Ruby-first evals library.*

**Date:** 2026-07-17

---

## Executive summary

- **The GenAI semantic conventions have moved to a dedicated repository.** They are no longer maintained in `open-telemetry/semantic-conventions`; the `docs/gen-ai/README.md` there now carries a "moved" notice pointing at [`open-telemetry/semantic-conventions-genai`](https://github.com/open-telemetry/semantic-conventions-genai). The mirror registry page on opentelemetry.io marks every `gen_ai.*` attribute as moved/deprecated-at-that-location. ([old README](https://raw.githubusercontent.com/open-telemetry/semantic-conventions/main/docs/gen-ai/README.md), [new repo](https://github.com/open-telemetry/semantic-conventions-genai))
- **Maturity: still experimental ("Development").** Every GenAI signal — spans, agent spans, events, metrics — carries a **Development** stability badge, and the new repo has **no tagged release yet** (as of 2026-07-17). Attributes reusing core conventions (`server.address`, `error.type`) are Stable, but all `gen_ai.*` are Development. Treat the whole surface as unstable/opt-in. ([spans](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-spans.md), [releases](https://github.com/open-telemetry/semantic-conventions-genai/releases))
- **Eval-result conventions DO now exist** (this is the headline change vs. a year ago). There is a **`gen_ai.evaluation.result`** log-based event plus a `gen_ai.evaluation.*` attribute family: `gen_ai.evaluation.name`, `gen_ai.evaluation.score.value` (double), `gen_ai.evaluation.score.label` (string), `gen_ai.evaluation.explanation`. It is emitted **parented to the GenAI span being evaluated**, falling back to `gen_ai.response.id` when there is no span. There is **no evaluation metric** and **no eval-run/eval-suite span** — only the per-result event. ([events](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-events.md))
- **The Ruby OTel SDK is production-ready and stable (1.x).** `opentelemetry-api` 1.10.1 and `opentelemetry-sdk` 1.12.1 (both released 2026-07-08, ~72M / ~57M downloads, Ruby >= 3.3, Apache-2.0). The API↔SDK split means a library can depend only on the tiny stable **API** and stay a no-op until an app installs the SDK. ([api gem](https://rubygems.org/gems/opentelemetry-api), [sdk gem](https://rubygems.org/gems/opentelemetry-sdk))
- **There is a well-established optional-dependency pattern in Ruby OTel.** `opentelemetry-ruby-contrib` instrumentation gems never take a runtime dependency on the library they instrument — they gate on `present { defined?(::Faraday) }` / `compatible { ... }` blocks and only load patches inside `install`. We should mirror this: no hard gemspec dep on OTel, conditional require, noop when absent. ([base README](https://raw.githubusercontent.com/open-telemetry/opentelemetry-ruby-contrib/main/instrumentation/base/README.md), [faraday gemspec](https://raw.githubusercontent.com/open-telemetry/opentelemetry-ruby-contrib/main/instrumentation/faraday/opentelemetry-instrumentation-faraday.gemspec))
- **Everyone else is ahead of the official spec, in incompatible ways.** OpenInference (Arize) has an `EVALUATOR` span *kind* but no eval attributes; Braintrust ingests OTLP and reads a `braintrust.scores` JSON attribute; Pydantic Evals leans on Logfire and *span-querying* evaluators rather than a documented emitted schema. None of them uses the new `gen_ai.evaluation.result` event. ([OpenInference](https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md), [Braintrust](https://www.braintrust.dev/docs/integrations/sdk-integrations/opentelemetry))
- **Recommendation:** wire OTel as a soft/optional dependency on `opentelemetry-api` only, emit a small custom span tree for run→case→trial (no standard exists for those), reuse `gen_ai.*` on the model-call spans, and represent every score with the standard **`gen_ai.evaluation.result`** event so downstream backends that understand it get scores for free.

---

## 1. GenAI semantic conventions: status & content

### 1.1 Where they live now, and how stable they are

The conventions were relocated out of the monorepo. `open-telemetry/semantic-conventions/docs/gen-ai/README.md` now reads: *"GenAI semantic conventions have moved to the OpenTelemetry GenAI semantic conventions repository. This page has moved and is no longer maintained in this repository."* ([raw README](https://raw.githubusercontent.com/open-telemetry/semantic-conventions/main/docs/gen-ai/README.md)). The opentelemetry.io registry page for `gen_ai.*` similarly flags the attributes as moved to the new repo ([registry](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/)).

The new repo, [`open-telemetry/semantic-conventions-genai`](https://github.com/open-telemetry/semantic-conventions-genai), is structured as `docs/` (human-readable prose + signal definitions), `model/` (YAML source of truth), `reference/` (Python compliance matrix), built with Weaver on top of core conventions. `docs/gen-ai/` contains: `README.md`, `gen-ai-spans.md`, `gen-ai-agent-spans.md`, `gen-ai-events.md`, `gen-ai-metrics.md`, `gen-ai-exceptions.md`, `mcp.md`, and provider files `openai.md`, `anthropic.md`, `aws-bedrock.md`, `azure-ai-inference.md`, plus a `non-normative/` folder.

**Stability: Development (experimental) across the board**, and **no versioned release exists yet** (the Releases page shows "There aren't any releases here" as of 2026-07-17). So there is not even a pinned version to cite — only `main`. ([releases](https://github.com/open-telemetry/semantic-conventions-genai/releases))

### 1.2 Spans

Span name format is generally `{gen_ai.operation.name} {gen_ai.request.model}` (or a model-less form where no model applies). ([spans](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-spans.md), [agent spans](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-agent-spans.md))

| Span | `gen_ai.operation.name` | Span name | Kind |
|---|---|---|---|
| Inference / chat | `chat`, `text_completion`, `generate_content` | `{op} {request.model}` | CLIENT |
| Embeddings | `embeddings` | `{op} {request.model}` | CLIENT |
| Execute tool | `execute_tool` | `execute_tool {gen_ai.tool.name}` | INTERNAL |
| Create agent | `create_agent` | `create_agent {gen_ai.agent.name}` | CLIENT / INTERNAL |
| Invoke agent | `invoke_agent` | `invoke_agent {gen_ai.agent.name}` | CLIENT (remote) / INTERNAL (in-process frameworks) |
| Retrieval | `retrieval` | `{op} {gen_ai.data_source.id}` | — |
| Memory ops | `create_memory`, `search_memory`, `delete_memory`, … | `{op}` | — |

Nesting is hierarchical: inference/tool spans nest **under** an `invoke_agent` span; token/latency roll up naturally. ([agent spans](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-agent-spans.md))

### 1.3 Core `gen_ai.*` attributes

**Inference/chat span** (all `gen_ai.*` are Development; `server.*`/`error.type` are Stable):

| Attribute | Requirement | Notes |
|---|---|---|
| `gen_ai.operation.name` | Required | one of `chat`/`text_completion`/`embeddings`/`generate_content`/`create_agent`/`invoke_agent`/`execute_tool`/`invoke_workflow`/`retrieval` |
| `gen_ai.provider.name` | Required | `openai`, `anthropic`, `gcp.vertex_ai`, `gcp.gemini`, `aws.bedrock`, `azure.ai.openai`, `azure.ai.inference`, `cohere`, `mistral_ai`, `groq`, `deepseek`, `perplexity`, `x_ai`, `ibm.watsonx.ai`, … (this replaces the older `gen_ai.system`) |
| `gen_ai.request.model` | Conditionally Required | if available |
| `gen_ai.response.model` | Recommended | model that actually served |
| `gen_ai.response.id` | Recommended | provider completion id — also the eval-event join key |
| `gen_ai.response.finish_reasons` | Recommended | string[] |
| `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` | Recommended | int |
| `gen_ai.request.temperature` / `top_p` / `top_k` / `max_tokens` / `frequency_penalty` / `presence_penalty` / `seed` / `stop_sequences` / `choice_count` | Recommended | sampling params |
| `gen_ai.output.type` | — | `text`/`json`/`image`/`speech` |
| `gen_ai.conversation.id` | Conditionally Required | session/thread grouping |
| `server.address` / `server.port` | Recommended / Cond. Req. | **Stable** |
| `error.type` | Conditionally Required (on error) | **Stable** |

Extra usage attributes exist for modern billing: `gen_ai.usage.cache_creation.input_tokens`, `gen_ai.usage.cache_read.input_tokens`, `gen_ai.usage.reasoning.output_tokens`. ([registry](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/))

**Execute-tool span:** `gen_ai.tool.name` (Required), `gen_ai.tool.type` (`function`/`extension`/`datastore`), `gen_ai.tool.call.id`, and **Opt-In** `gen_ai.tool.call.arguments` / `gen_ai.tool.call.result`. **Agent spans:** `gen_ai.agent.id`, `gen_ai.agent.name`, `gen_ai.agent.description`, `gen_ai.agent.version`, `gen_ai.conversation.id`, `gen_ai.request.model`.

### 1.4 How prompts/completions are captured

Content is captured as **structured attributes** on the span (or as event fields), NOT free-text: `gen_ai.input.messages`, `gen_ai.output.messages`, `gen_ai.system_instructions` (each a structured JSON value following a defined message schema; MAY be a JSON string if structured values aren't supported). **Because this is sensitive/large, instrumentations SHOULD NOT capture it by default and SHOULD provide an opt-in**, gated by `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`. The spec also blesses an in-process hook pattern to upload content externally and store only a reference on the span. There is also a dedicated log event **`gen_ai.client.inference.operation.details`** carrying request params + chat history independent of the trace. ([spans](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-spans.md), [events](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-events.md))

### 1.5 Evaluation results — YES, a convention now exists

Searching the repo for "evaluation" is fruitful: `docs/gen-ai/gen-ai-events.md` defines a log-based event that **MUST be named `gen_ai.evaluation.result`**. ([events](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-events.md))

| Attribute | Requirement | Type / meaning |
|---|---|---|
| `gen_ai.evaluation.name` | Required | metric name, e.g. `relevance`, `correctness` |
| `gen_ai.evaluation.score.value` | Conditionally Required | double numeric score |
| `gen_ai.evaluation.score.label` | Conditionally Required | human-readable interpretation, e.g. `"relevant"` (spec's own example: value `1` may mean "relevant" in one system, "not relevant" in another — so the label disambiguates) |
| `gen_ai.evaluation.explanation` | Recommended | free-form rationale (judge reasoning) |
| `gen_ai.response.id` | Recommended | join key to the evaluated response |
| `error.type` | Conditionally Required | **Stable**; when the evaluation itself failed |

Emission model: *"This event SHOULD be parented to the GenAI operation span being evaluated when possible, or set `gen_ai.response.id` when the span id is not available."* So a score attaches to the model-call span it grades (in-band), or is correlated after the fact via `gen_ai.response.id` (out-of-band, e.g. offline eval of production traces). The event is a **log record**, and the spec flags events as "in-development and not yet available in some languages."

**Important scope limits:** the convention covers a **single evaluation result** on a single response. It defines **no** eval-run, eval-suite, dataset, or case span; **no** eval-run identifier; and **no** evaluation metric. Anything above the per-result granularity is unspecified and must be custom.

### 1.6 Metrics (for completeness)

`gen_ai.client.token.usage` (Histogram `{token}`), `gen_ai.client.operation.duration` (Histogram `s`), streaming `time_to_first_chunk` / `time_per_output_chunk`; server-side `gen_ai.server.request.duration`, `time_to_first_token`, `time_per_output_token`; agent `gen_ai.invoke_agent.duration` / `.inference_calls` / `.tool_calls`; `gen_ai.execute_tool.duration`; `gen_ai.workflow.duration`. All Development. **No evaluation metric.** ([metrics](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-metrics.md))

---

## 2. How other eval frameworks emit telemetry

Comparison against the official spec:

| Framework | Transport | Eval representation | Uses `gen_ai.*`? | Uses `gen_ai.evaluation.result`? |
|---|---|---|---|---|
| **OTel GenAI (official)** | spans + log events | `gen_ai.evaluation.result` event, parented to graded span | — | yes (it *is* the spec) |
| **OpenInference (Arize)** | OTel spans | `EVALUATOR` span *kind*, but no eval score attributes | no | no |
| **Braintrust** | OTLP → hosted | `braintrust.scores` JSON attribute on span | reads them | no |
| **Pydantic Evals** | OTel via Logfire | span-*querying* evaluators; emitted schema undocumented | via underlying agent instrumentation | no |

### 2.1 Pydantic Evals → Logfire/OTel

`pydantic-evals` takes an **optional dependency on `logfire`** "if you'd like to use OpenTelemetry traces in your evals." Its distinctive feature is **span-based evaluation**: when Logfire is configured, "Pydantic Evals captures all OpenTelemetry spans generated during task execution," and evaluators (`HasMatchingSpan`, custom `SpanTree`-walking evaluators) *query* that span tree (by `name_contains` / `name_equals`, attributes, duration, status) to judge internal agent behavior. Notably, the docs describe *how to consume* spans your task emits and how results surface in the Logfire UI, **but do not document the span names/attribute schema Pydantic Evals itself emits** for runs/cases/evaluators — so I cannot verify from primary sources that it uses (or doesn't use) a stable convention here. It clearly does **not** adopt the new `gen_ai.evaluation.result` event. ([evals overview](https://pydantic.dev/docs/ai/evals/evals/), [span-based](https://pydantic.dev/docs/ai/evals/evaluators/span-based/))

### 2.2 Braintrust OTel

Braintrust exposes an **OTLP endpoint** (`https://api.braintrust.dev/otel/v1/traces`, EU variant available) configured via `OTEL_EXPORTER_OTLP_ENDPOINT`, so any OTel SDK can export to it. It ingests **both** `gen_ai.*` (`gen_ai.prompt`, `gen_ai.completion`, `gen_ai.request.model`, `gen_ai.usage.*`, `gen_ai.operation.name`) **and** a `braintrust.*` family: `braintrust.input`, `braintrust.output`, `braintrust.expected`, `braintrust.metadata`, `braintrust.metrics`, `braintrust.tags`, `braintrust.span_attributes`, and crucially **`braintrust.scores`** — a JSON-serialized dict of `{score_name: value}` on the span (also settable flattened as `braintrust.scores.<name>`). So Braintrust puts scores **as span attributes**, a different model from the official per-result **event**. Note it reads the older `gen_ai.prompt`/`gen_ai.completion` names rather than the newer `gen_ai.input.messages`/`output.messages`. ([Braintrust OTel](https://www.braintrust.dev/docs/integrations/sdk-integrations/opentelemetry))

### 2.3 OpenInference (Arize)

OpenInference defines `openinference.span.kind` (required on every span) with ten kinds — `LLM`, `EMBEDDING`, `CHAIN`, `RETRIEVER`, `RERANKER`, `TOOL`, `AGENT`, `GUARDRAIL`, `PROMPT`, and **`EVALUATOR`** ("a call to a function or process performing an evaluation of the language model's outputs"). But the spec provides **no eval-specific attribute namespace** (no `openinference.evaluator.*` scores/labels) — the EVALUATOR kind just classifies the span; scores/labels are handled out-of-band by Arize Phoenix (span annotations), not in this OTel attribute spec. Attribute prefixes are `openinference.*`, `llm.*`, `message.*`, `tool.*`, `document.*`, etc. — a **separate namespace** from `gen_ai.*`, with no reference to the OTel GenAI conventions. ([OpenInference spec](https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md))

**Takeaway:** the ecosystem has three incompatible eval representations (Arize span-kind, Braintrust score-attribute, OTel score-event). Only the official one models a score as a first-class, joinable artifact. Emitting the official `gen_ai.evaluation.result` costs little and is the most future-proof; a Braintrust-style attribute is trivial to add as an adapter if we want that backend.

---

## 3. Ruby OTel SDK maturity

### 3.1 Versions & stability

| Gem | Latest | Released | Downloads | Ruby | Stable? |
|---|---|---|---|---|---|
| `opentelemetry-api` | 1.10.1 | 2026-07-08 | ~72.0M | >= 3.3 | **Yes, 1.x** |
| `opentelemetry-sdk` | 1.12.1 | 2026-07-08 | ~57.5M | >= 3.3 | **Yes, 1.x** |

Both Apache-2.0. The **API/SDK split is the key architectural fact**: `opentelemetry-api` is a tiny gem whose tracer is a **no-op by default**; the heavyweight `opentelemetry-sdk` (exporters, sampling, batch processor) is only wired in when the *application* configures it via `OpenTelemetry::SDK.configure`. A library should depend on the API (if anything) and never on the SDK. ([api gem](https://rubygems.org/gems/opentelemetry-api), [sdk gem](https://rubygems.org/gems/opentelemetry-sdk))

Note: **no `gen_ai` metrics/semconv gem exists for Ruby.** The `gen_ai.*` names are just strings — Ruby has the tracing plumbing but no packaged GenAI convention constants, so we'd define our own attribute-key constants.

### 3.2 The contrib instrumentation-gem pattern

`opentelemetry-ruby-contrib` ships one gem per instrumented library (`opentelemetry-instrumentation-faraday`, `-rack`, `-rails`, `-net_http`, `-sidekiq`, …). Each defines an `OpenTelemetry::Instrumentation::<Lib>::Instrumentation < OpenTelemetry::Instrumentation::Base`. Subclassing auto-registers it in a global registry; `OpenTelemetry::SDK.configure { |c| c.use 'OpenTelemetry::Instrumentation::Faraday' }` (or `c.use_all`) later calls `registry.install_all`. ([base README](https://raw.githubusercontent.com/open-telemetry/opentelemetry-ruby-contrib/main/instrumentation/base/README.md))

The `Base` DSL blocks:

- `present { defined?(::Faraday) }` — gate: if the target library isn't loaded, installation is skipped entirely.
- `compatible { gem_version >= MINIMUM_VERSION }` — version guard.
- `install { |config| require_dependencies; patch }` — **all** patching/`require`s of instrumentation code happen only here, after the gates pass.

### 3.3 The idiomatic optional-dependency mechanism

The instrumented library is **never a runtime gemspec dependency**. `opentelemetry-instrumentation-faraday.gemspec` lists exactly **one runtime dependency — `opentelemetry-instrumentation-base` (~> 0.25)** (which pulls `opentelemetry-api`); `faraday` itself appears only as a development/test dependency (matched at runtime via `defined?(::Faraday)`, not required). ([faraday gemspec](https://raw.githubusercontent.com/open-telemetry/opentelemetry-ruby-contrib/main/instrumentation/faraday/opentelemetry-instrumentation-faraday.gemspec), [faraday instrumentation.rb](https://raw.githubusercontent.com/open-telemetry/opentelemetry-ruby-contrib/main/instrumentation/faraday/lib/opentelemetry/instrumentation/faraday/instrumentation.rb))

For **our** gem (which *emits* rather than *auto-instruments*), the symmetric move is:

- **No hard dep on OTel in the gemspec.** Either (a) zero OTel deps and detect `defined?(OpenTelemetry)`, or (b) a soft dependency on the small, stable `opentelemetry-api` only — never the SDK.
- **Conditional require + noop fallback.** If OTel is absent (or the app never configures an SDK), `OpenTelemetry.tracer_provider.tracer(...)` already returns a no-op tracer, so emission code can run unconditionally once guarded by a `defined?` check — no branching in hot paths.
- **A config flag** (`config.otel = true/false`, default off or auto-detect) so users opt in explicitly.

---

## 4. Recommendation sketch

### 4.1 Span tree an eval run should emit

No standard exists above a single evaluation result, so the run/case/trial spans are **our own custom spans**; the model-call layer reuses the official `gen_ai.*` conventions; scores use the official `gen_ai.evaluation.result` event.

```
eval.run                     (INTERNAL, custom)  ── one per `evals` invocation
└─ eval.case                 (INTERNAL, custom)  ── one per dataset row/case
   └─ eval.trial             (INTERNAL, custom)  ── one per repetition (n>1 sampling)
      ├─ invoke_agent {name}       (gen_ai, if task is an agent)   ← official
      │  ├─ chat {request.model}   (gen_ai)                        ← official
      │  └─ execute_tool {name}    (gen_ai)                        ← official
      └─ eval.scorer {name}  (INTERNAL, custom)  ── the grader/judge
         └── emits event: gen_ai.evaluation.result                ← official
```

The scorer span, when it grades a specific model response, **parents** the `gen_ai.evaluation.result` event on the graded `chat` span (per §1.5); for offline scoring of pre-existing traces it instead sets `gen_ai.response.id`.

### 4.2 Attributes per span

| Span | Standard attrs | Custom attrs (no convention exists — flagged) |
|---|---|---|
| `eval.run` | `service.name` | `eval.run.id`, `eval.suite.name`, `eval.dataset.name`, `eval.dataset.size`, `eval.trials_per_case` |
| `eval.case` | — | `eval.case.id`, `eval.case.name`, `eval.case.tags` (and inputs/expected as **opt-in** custom, sensitivity-gated) |
| `eval.trial` | — | `eval.trial.index` |
| `invoke_agent` / `chat` / `execute_tool` | full `gen_ai.*` from §1.3 (`gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.response.id`, `gen_ai.usage.*`, request params); message content **opt-in** via our own `CAPTURE_MESSAGE_CONTENT`-style flag mirroring the spec | — |
| `eval.scorer` | emits `gen_ai.evaluation.result` event | `eval.scorer.kind` (`exact`/`llm_judge`/…) |

**Where scores attach:** the standard `gen_ai.evaluation.result` event with `gen_ai.evaluation.name` (= scorer name), `gen_ai.evaluation.score.value`, `gen_ai.evaluation.score.label`, `gen_ai.evaluation.explanation` (judge reasoning), joined via parent span or `gen_ai.response.id`. This is the one part of the eval that has an official home — use it verbatim. Optionally also mirror the aggregate onto the `eval.run`/`eval.case` span as custom `eval.score.<name>` attributes for easy dashboarding, and note it's non-standard. If a Braintrust backend is a target, add an adapter that additionally writes `braintrust.scores` (§2.2).

> **Namespace caveat:** `eval.*` is a name we are minting; it is NOT an OpenTelemetry convention. Document it as ours and keep it isolated so a future standard (should one appear above the per-result level) can supersede it without churn.

### 4.3 Wiring OTel as an optional dependency (concrete)

1. **Gemspec:** do **not** add `opentelemetry-sdk`. Prefer zero OTel runtime deps and detect at runtime; or at most a soft dep on `opentelemetry-api` (stable 1.10.x). Keep `opentelemetry-sdk` + an exporter as **development** deps for our own tests only.
2. **Single seam:** a `RubyEvals::Telemetry` module that lazily resolves a tracer: `defined?(::OpenTelemetry) ? ::OpenTelemetry.tracer_provider.tracer('ruby-evals', VERSION) : NoopTracer`. Because the API's default provider is already a no-op, apps that don't configure an SDK pay ~nothing.
3. **Opt-in flag:** `RubyEvals.configure { |c| c.otel = true }` (default off, or auto-on when `defined?(OpenTelemetry)`), plus a content-capture flag defaulting to off (mirroring `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`) so prompts/completions are never exported unless asked.
4. **Contrib-style gating** if we later ship a separate `opentelemetry-instrumentation-ruby_evals` gem: subclass `OpenTelemetry::Instrumentation::Base`, `present { defined?(::RubyEvals) }`, patch in `install`. Its only runtime dep would be `opentelemetry-instrumentation-base`.

---

## Sources

**OpenTelemetry GenAI conventions (primary):**
- Moved notice, old repo: <https://raw.githubusercontent.com/open-telemetry/semantic-conventions/main/docs/gen-ai/README.md>
- New repo root: <https://github.com/open-telemetry/semantic-conventions-genai>
- Releases (none yet): <https://github.com/open-telemetry/semantic-conventions-genai/releases>
- Spans: <https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-spans.md>
- Agent spans: <https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-agent-spans.md>
- Events (incl. `gen_ai.evaluation.result`): <https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-events.md>
- Metrics: <https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-metrics.md>
- Attribute registry (mirror, marked moved): <https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/>

**Other eval frameworks (primary):**
- OpenInference spec: <https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md>
- Braintrust OTel integration: <https://www.braintrust.dev/docs/integrations/sdk-integrations/opentelemetry>
- Pydantic Evals overview: <https://pydantic.dev/docs/ai/evals/evals/>
- Pydantic Evals span-based evaluators: <https://pydantic.dev/docs/ai/evals/evaluators/span-based/>

**Ruby OTel SDK (primary):**
- `opentelemetry-api` gem: <https://rubygems.org/gems/opentelemetry-api>
- `opentelemetry-sdk` gem: <https://rubygems.org/gems/opentelemetry-sdk>
- Contrib repo: <https://github.com/open-telemetry/opentelemetry-ruby-contrib>
- Instrumentation base README: <https://raw.githubusercontent.com/open-telemetry/opentelemetry-ruby-contrib/main/instrumentation/base/README.md>
- Faraday instrumentation.rb: <https://raw.githubusercontent.com/open-telemetry/opentelemetry-ruby-contrib/main/instrumentation/faraday/lib/opentelemetry/instrumentation/faraday/instrumentation.rb>
- Faraday gemspec (optional-dep pattern): <https://raw.githubusercontent.com/open-telemetry/opentelemetry-ruby-contrib/main/instrumentation/faraday/opentelemetry-instrumentation-faraday.gemspec>

*Not used as sources (secondary blogs seen in search, excluded per primary-source rule): Greptime and Fiddler AI blog posts on OTel GenAI.*
