# Ruby AI Ecosystem Scan

*Primary-source landscape scan to inform a new Ruby-first library for evaluating AI-powered applications and agents (promptfoo/Braintrust-style, Ruby-native).*

**Date:** 2026-07-17

---

## Executive summary

- **The gap is real and largely open.** There is no mature, widely-adopted, framework-agnostic Ruby eval library. The closest existing gems (`leva`, `ruby_llm-evals`, `ruby_llm-tribunal`) are all young (2025–2026), low-adoption, and mostly Rails/RubyLLM-coupled. No Ruby equivalent of promptfoo or DeepEval in maturity exists. ([leva](https://github.com/kieranklaassen/leva), [ruby_llm-evals](https://github.com/sinaptia/ruby_llm-evals))
- **Biggest integration target: RubyLLM.** It is by far the most-used modern Ruby LLM library (~10.6M downloads, ~4.3k stars, ~monthly releases, last commit 2026-07-15) and — critically — it already exposes a first-class instrumentation/trace surface (`ActiveSupport::Notifications` events, structured `Message` objects, token/cost data, additive callbacks). ([ruby_llm on rubygems](https://rubygems.org/gems/ruby_llm), [instrumentation docs](https://rubyllm.com/instrumentation/))
- **Biggest collision risk: the gem name `evals` is already taken** by a real (if abandoned-looking) "LLM evals" gem, a direct name-and-domain collision. `ruby-evals`, `ruby_evals`, `evalz`, `evalable`, and `assess` are free. ([evals gem](https://rubygems.org/gems/evals))
- **Platform Ruby SDKs are near-absent.** Only Braintrust ships a first-party Ruby SDK, and it is explicitly beta. Langfuse, LangSmith, and Arize/Phoenix (OpenInference) are officially Python/TS-only (some add Java/Go, never Ruby). Helicone needs no SDK (proxy). promptfoo is a Node CLI that can *call* Ruby scripts but has no Ruby library. ([Braintrust Ruby SDK](https://github.com/braintrustdata/braintrust-sdk-ruby), [Langfuse SDK overview](https://langfuse.com/docs/sdk/overview))
- **Caching recommendation: build a purpose-built, LLM-aware cache — do not make VCR the core replay path for streaming.** WebMock (VCR's default backend) collapses multi-chunk streamed responses into a single body at record time (open issue since 2018), and VCR stores bodies as a single string, so token-by-token SSE replay is not faithful out of the box. Keep VCR/WebMock only for incidental non-streaming HTTP. ([webmock#742](https://github.com/bblimke/webmock/issues/742), [webmock#629](https://github.com/bblimke/webmock/issues/629))
- **Only one Ruby library ships built-in scored evals today: langchainrb** (RAGAS metrics + LLM-graded QA correctness) — but it has shipped no gem release since May 2025 and is slow-maintenance. ([langchainrb](https://github.com/patterns-ai-core/langchainrb))
- **Best existing trace/run-output surfaces to consume** come from RubyLLM (instrumentation events + `Message`/token data), ActiveAgent (span waterfalls, token metrics, dev-console telemetry), and langchainrb (`messages` + callbacks). The official OpenAI/Anthropic SDKs expose only raw provider response objects (usage, tool_calls) with no first-class trace layer.
- **Surprising discovery:** Braintrust's beta Ruby SDK already auto-instruments RubyLLM specifically (alongside OpenAI/Anthropic) — a commercial platform has already bet on RubyLLM as the Ruby integration point, validating it as the ecosystem's center of gravity. ([Braintrust Ruby SDK](https://github.com/braintrustdata/braintrust-sdk-ruby))

---

## Overlap / gap table

| Project / tool | Category | Built-in eval story? | Emits traces / run output? | Maintained? | Relevance to us |
|---|---|---|---|---|---|
| **RubyLLM** | LLM client/framework | No | **Yes** — instrumentation events, `Message` objects, token/cost, callbacks | Active (1.16.0, 2026-06; commit 2026-07-15) | **Primary integration target** |
| langchainrb | LLM app framework | **Yes** (RAGAS + LLM-graded QA) | Yes (`messages`, callbacks, token usage) | Slow (no release since 2025-05) | Prior art + integration; competitor on eval sub-feature |
| ruby-openai (community) | OpenAI client (Faraday) | No | Raw provider responses only | Active-ish (8.3.0, 2025-08) | Provider under test; VCR prior art |
| openai (official SDK) | OpenAI client (net/http) | No | Typed response objects only | Very active (0.71.0, 2026-07-17) | Provider under test |
| anthropic (official SDK) | Claude client (net/http) | No | Structured `Message` + usage | Very active (1.56.0, 2026-07-16) | Provider under test |
| Raix | AI mixins | No | Transcript + auto-tracked function calls | Active (2.0.5, 2026-06) | Possible integration surface |
| ActiveAgent | Rails agent framework | No (observability, not scored) | **Yes** — span traces, token metrics, dev console | Active (1.0.3, 2026-07-14) | Strong trace source; integration |
| sublayer | Agent framework | No | None documented | **Dormant** (0.2.9, 2025-06) | Low relevance |
| informers | Local transformer inference | No | Structured pipeline results (not agent traces) | Active (1.3.0, 2026-04) | Possible local-judge/embeddings backend |
| aia | CLI prompt tool | No (token/cost + multi-model compare) | History log, token/cost tables | Active-small (1.1.1, 2026-05) | Low relevance |
| leva | **Ruby eval framework** | **Yes** (Rails/ActiveRecord datasets) | Dataset-driven | Active (0.3.4, 2025-12; 142★) | **Closest competitor** |
| ruby_llm-evals | **Ruby eval framework** | **Yes** (exact/contains/regex/human) | Built on RubyLLM | New (0.1.0, 2026-02) | Direct competitor |
| ruby_llm-tribunal | **Ruby eval framework** | **Yes** (LLM-judge, assertions, red-team) | Built on RubyLLM | Very new (0.1.1, 2026-01; 57★) | Direct competitor |
| Braintrust Ruby SDK | Platform client (evals+tracing) | Yes (scorers) | Yes (OTel, auto-instrument) | Beta (0.4.0, 2026-06) | Competitor + integration; validates RubyLLM |
| Langfuse (community gems) | Observability client | No (tracing) | Yes (via API/OTel) | Community-only | Export target |
| promptfoo | Node CLI eval | Yes (but not Ruby) | N/A | Active (Node) | Conceptual model; Ruby via script-provider only |
| VCR / WebMock | HTTP record/stub | N/A | N/A | Active (VCR 6.4.0 2025-12; WebMock 3.26.2 2026-03) | Caching prior art; streaming-limited |

---

## 1. RubyLLM

Primary sources: [github.com/crmne/ruby_llm](https://github.com/crmne/ruby_llm), [rubyllm.com](https://rubyllm.com/), [rubygems.org/gems/ruby_llm](https://rubygems.org/gems/ruby_llm).

**API surface.** Top-level module methods: `RubyLLM.chat` (returns `RubyLLM::Chat`), `RubyLLM.embed`, `RubyLLM.paint` (image gen/edit), `RubyLLM.transcribe`, `RubyLLM.speak`, `RubyLLM.moderate`, `RubyLLM.batch`, `RubyLLM.models`. Note `ask` is a chat-instance method (`chat.ask`), not a module method. Chat instances are chainable: `with_model`, `with_temperature`, `with_instructions`, `with_schema`, `with_params`, `with_headers`, `with_tool`/`with_tools`, `add_message`, and `messages` (full history). Config via `RubyLLM.configure`, including `config.instrumenter` and per-provider `api_base`. Providers: OpenAI, Anthropic, Gemini, Bedrock, Mistral, Perplexity, VertexAI, xAI, Ollama, and any OpenAI-compatible endpoint ("800+ models"). ([chat docs](https://rubyllm.com/chat/), [1.16.0 release](https://github.com/crmne/ruby_llm/releases/tag/1.16.0))

**Extension points.**
- **Tools/function calling:** subclass `RubyLLM::Tool` with `description` and `execute(**kwargs)`; params via signature inference (1.15+), `param` helper, or the `params` DSL (1.9+, nested objects/arrays or raw JSON Schema). Attach with `with_tool`/`with_tools` (`replace:`, `choice:`, `calls:`); tools can run concurrently (`concurrency: true`, 1.16). Tool results enter history as `:tool`-role messages. ([tools docs](https://rubyllm.com/tools/))
- **Custom providers/models:** no direct "register a model" API; use `assume_model_exists: true` with explicit `provider:` to bypass registry validation, plus per-provider `api_base` for custom endpoints. ([models docs](https://rubyllm.com/models/))
- **Callbacks/hooks (key eval surface):** Rails-style *additive* callbacks (1.15+): `before_message`, `after_message`, `before_tool_call`, `after_tool_result` (all registrations run; work for streaming and non-streaming). Legacy *replacing* callbacks `on_new_message`/`on_end_message`/`on_tool_call`/`on_tool_result` still exist but emit deprecation warnings. ([chat docs](https://rubyllm.com/chat/))
- **Model registry:** `RubyLLM.models` with `find`, `refresh!`, `chat_models`, `embedding_models`, `by_provider`, `by_family`, Enumerable chaining. `Model::Info` exposes context window, max tokens, per-million input/output pricing, `supports_vision`/`supports_functions`, cache pricing. ([models docs](https://rubyllm.com/models/))

**Agent-trace usability (strong).** Several primitives a downstream eval gem can target:
- `chat.messages` → `RubyLLM::Message` objects with `role`, `content`, `model_id`, `tool_calls`, `tokens`.
- `message.tokens` with `.input`, `.output`, `.cache_read`, `.cache_write`, `.thinking`; cost helpers `response.cost.total` (1.15+).
- Streaming blocks receive `RubyLLM::Chunk` (a `Message` subclass) with cumulative tokens; `ask` returns the final assembled `Message`. ([streaming docs](https://rubyllm.com/streaming/))
- **Instrumentation (strongest trace surface):** `ActiveSupport::Notifications` in Rails (`subscribe('chat.ruby_llm')`) or a custom `config.instrumenter` with `instrument(name, payload)` outside Rails. Documented events: `request.ruby_llm`, `chat.ruby_llm`, `tool_call.ruby_llm`, `embedding.ruby_llm`, `image.ruby_llm`, `moderation.ruby_llm`, `transcription.ruby_llm`, `models.refresh.ruby_llm`. Payloads carry provider, model, messages, response, `input_tokens`/`output_tokens`, exceptions, with automatic timing. Docs explicitly frame this for distributed tracing/observability. ([instrumentation docs](https://rubyllm.com/instrumentation/))
- `RubyLLM::Agent` exists (declarative `model`/`instructions`/`tools`/`schema`), but multi-step traces are **not** dedicated trace objects — orchestration is plain Ruby `.ask` chaining. ([agentic workflows](https://rubyllm.com/agentic-workflows/))

**API stability.** No formal semver/stability guarantee. Upgrade docs state the legacy `acts_as` API "is deprecated and will be removed in RubyLLM 2.0.0," breaking changes are reserved for 2.0, and the 1.x line stays backwards-compatible with opt-in features; deprecations can be silenced or raised. Targeting the 1.x public API (Chat/Message/Tool/callbacks/instrumentation) is reasonably safe short-term, but there is no explicit long-term commitment. ([upgrading docs](https://rubyllm.com/upgrading/))

**Release cadence & activity.** Latest **1.16.0, 2026-06-09**; roughly monthly minors (1.15.0 2026-05-07, 1.14.1 2026-04-02, 1.14.0 2026-03-16). ~10.6M total downloads; Ruby ≥ 3.1.3; minimal deps (Faraday, Zeitwerk, Marcel). Last commit 2026-07-15; ~4.3k stars, 479 forks, 44 open issues; MIT. ([rubygems](https://rubygems.org/gems/ruby_llm), [releases](https://github.com/crmne/ruby_llm/releases))

**Ecosystem stance.** A dedicated [Ecosystem page](https://rubyllm.com/ecosystem/) lists official/community extensions: `RubyLLM::Schema`, `RubyLLM::MCP`, `RubyLLM::Instrumentation`, `RubyLLM::Monitoring`, `RubyLLM::RedCandle`, `opentelemetry-instrumentation-ruby_llm`, **`RubyLLM::Tribunal` (eval/testing)**, `RubyLLM::TopSecret`, and **`RubyLLM::Test` (response stubbing)**. Maintainer stance: projects are third-party-maintained and listed "for discoverability," with an open invitation to add extensions. First-class Rails integration (`acts_as_chat`, `bin/rails generate ruby_llm:install`/`chat_ui`) exists. ([Rails docs](https://rubyllm.com/rails/))

*Could not verify:* a personal maintainer (Carmine Paolino) quote endorsing downstream gems beyond the ecosystem-page language; exact verbatim event/callback strings should be spot-checked against `lib/ruby_llm` before being pasted into shipped code.

---

## 2. Other Ruby AI libraries

All facts as of 2026-07-17; versions/dates from rubygems and GitHub.

- **langchainrb** ([repo](https://github.com/patterns-ai-core/langchainrb), [gem](https://rubygems.org/gems/langchainrb)) — Multi-provider framework with RAG, vector stores, an `Assistant` abstraction. **Only library here with built-in scored evals:** `Langchain::Evals` ships RAGAS metrics (faithfulness, context relevance, answer relevance) and an LLM-graded QA correctness evaluator. Traces: `Assistant#messages`, `add_message_callback`/`tool_execution_callback`, and `prompt_tokens`/`completion_tokens`/`total_tokens`. **Maintenance flag:** latest gem 0.19.5 (2025-05-01) — occasional commits (last ~2026-05-01) but **no release in ~14 months**; ~1,990 stars.
- **ruby-openai (community)** ([repo](https://github.com/alexrudall/ruby-openai), [gem](https://rubygems.org/gems/ruby-openai)) — Faraday-based OpenAI wrapper. No user-facing eval/test story (uses VCR in its *own* suite). Exposes only raw provider responses (`usage`, `tool_calls`, Assistants run/step objects). Latest 8.3.0 (2025-08-29), ~43.9M downloads, last commit ~2026-05-01; active but slowed.
- **openai (official SDK)** ([repo](https://github.com/openai/openai-ruby), [gem](https://rubygems.org/gems/openai)) — Official SDK, Ruby 3.2+, full Sorbet/RBS types, **net/http** transport, SSE streaming. No eval helpers. Typed parsed responses + `usage`. **Very active:** 0.71.0 released 2026-07-17 (today), 94 releases, Apache-2.0.
- **anthropic (official SDK)** ([repo](https://github.com/anthropics/anthropic-sdk-ruby), [gem](https://rubygems.org/gems/anthropic)) — Official Claude SDK, Ruby 3.2+, `anthropic.messages.create`. No eval story. Structured `Message` + `usage`. **Very active:** 1.56.0 (2026-07-16), 79 releases.
- **Raix** ([repo](https://github.com/OlympiaAI/raix)) — "Ruby AI eXtensions": `ChatCompletion`, `FunctionDispatch`, `PromptDeclarations` mixins. Maintains an OpenAI-format transcript; function calls auto-appended. No eval helpers; token-usage API not documented (unverified). Latest 2.0.5 (2026-06-04), ~330 stars, active. Author Obie Fernandez.
- **ActiveAgent** ([repo](https://github.com/activeagents/activeagent), [activeagents.ai](https://activeagents.ai)) — "Agents are Controllers" Rails framework; multi-provider (incl. RubyLLM), ERB prompts, tool calling, ActionCable streaming. **Strong observability:** local dev console with request traces, span waterfalls, token-usage metrics, per-agent performance, plus optional remote telemetry. Story is traces/observability, not scored evals (no eval-metric/RSpec-matcher API found). Latest 1.0.3 (2026-07-14), ~961 stars, active.
- **sublayer** ([repo](https://github.com/sublayerapp/sublayer)) — Model-agnostic agent framework (Generators/Actions/Tasks/Agents/Triggers). No eval features; no documented traces/token tracking. **Dormant:** latest 0.2.9 (2025-06-13), last commit ~2025-06-25, ~22.4k downloads.
- **informers** ([repo](https://github.com/ankane/informers)) — Local transformer inference (ONNX; ports Transformers.js): embeddings, reranking, sentiment, NER, QA. No LLM/agent layer, no traces. Returns structured pipeline results (vectors, scored pairs). Latest 1.3.0 (2026-04-15), ~2.5M downloads, active. Relevant as a possible **local judge/embeddings backend**.
- **aia** ([repo](https://github.com/MadBomber/aia)) — CLI prompt tool ("The Prompt is the Code"): interactive/batch prompts, roles/skills, MCP, 20+ providers, multi-model compare & consensus. Eval-adjacent: per-model token/cost tracking, comparison tables. History log to `~/.prompts/_prompts.log`. CLI-oriented, not a programmatic run-object API. Latest 1.1.1 (2026-05-01), ~42k downloads, small/active.

---

## 3. Existing Ruby eval gems & name collisions

### Existing eval gems (sparse, young, mostly RubyLLM/Rails-coupled)

- **leva** — most mature. "Rails framework for evaluating LLMs using ActiveRecord datasets on production models." Web UI at `/leva`, dataset management, DSPy prompt optimization, Together AI fine-tuning. 142 stars, ~13.2k downloads, latest 0.3.4 (2025-12-17), 221 commits. ([gem](https://rubygems.org/gems/leva), [repo](https://github.com/kieranklaassen/leva))
- **ruby_llm-evals** — "LLM evaluation engine for Rails," built on RubyLLM (owner sinaptia). ~8k downloads, 0.1.0 (2026-02-18). Eval types: exact match, contains, regex, human review. ([gem](https://rubygems.org/gems/ruby_llm-evals), [repo](https://github.com/sinaptia/ruby_llm-evals))
- **ruby_llm-tribunal** (`RubyLLM::Tribunal`) — "LLM evaluation framework powered by RubyLLM." Hallucination detection, LLM-as-Judge (faithfulness), deterministic assertions (substring/regex/JSON), safety + red-team attack generation. 57 stars, 0.1.1 (2026-01-16), ~10 commits — very new. Listed on RubyLLM's ecosystem page. ([repo](https://github.com/Alqemist-labs/ruby_llm-tribunal))
- **braintrust** (`braintrust-sdk-ruby`) — official Ruby SDK for the hosted Braintrust platform (tracing + evals, OTel, auto-instrumentation for OpenAI/Anthropic/RubyLLM, scorers, dev server). A commercial-product client, not a standalone OSS framework. 10 stars, 0.4.0 (2026-06-15), Apache-2.0. ([repo](https://github.com/braintrustdata/braintrust-sdk-ruby))
- **evals** (andyw8) — tiny gem literally named "A library for LLM evals," based on Anthropic's prompt-evaluation example. Requires an Anthropic API key. 3 stars, 5 commits, ~824 downloads, 0.1.1 (2025-07-17), no updates since — effectively a stub. **This occupies the `evals` name.** ([gem](https://rubygems.org/gems/evals), [repo](https://github.com/andyw8/evals))
- **langchainrb** — includes the `Langchain::Evals` module (see §2); eval is a sub-feature, not the gem's purpose.

**What does not exist:** no widely-adopted, standalone, framework-agnostic Ruby eval library with significant downloads/stars — no Ruby promptfoo/DeepEval equivalent. The market is genuinely early and has no dominant player.

### Gem name availability

RubyGems treats hyphen and underscore as **distinct** (`ruby-evals` ≠ `ruby_evals`), so both were checked. "Free" reflects a live 404 today; RubyGems can additionally block names as too-similar at `gem push` time, so confirm with a push dry-run before committing.

| Candidate | Status | URL | Notes |
|---|---|---|---|
| `evals` | **TAKEN** | [link](https://rubygems.org/gems/evals) | Owner andyw8; ~824 downloads; 0.1.1 (2025-07-17). Literal "LLM evals" gem — direct name+domain collision. Looks abandoned but occupied. |
| `ruby-evals` | **FREE** | [link](https://rubygems.org/gems/ruby-evals) | 404. |
| `ruby_evals` | **FREE** | [link](https://rubygems.org/gems/ruby_evals) | 404. |
| `evalz` | **FREE** | [link](https://rubygems.org/gems/evalz) | 404. |
| `evalable` | **FREE** | [link](https://rubygems.org/gems/evalable) | 404. |
| `assess` | **FREE** | [link](https://rubygems.org/gems/assess) | 404. |
| `grade` | **TAKEN** | [link](https://rubygems.org/gems/grade) | Owner aaronliu; ~1.9k downloads; 0.1.0 (2021-01-03). Abandoned, unrelated. |
| `rubric` | **TAKEN** | [link](https://rubygems.org/gems/rubric) | ~3.7k downloads; 0.0.1 (2015-06-02). Abandoned ~decade. |
| `judge` | **TAKEN** | [link](https://rubygems.org/gems/judge) | ~293.9k downloads; 3.1.0 (2023-02-27). Established Rails client-side validation gem — not available. |
| `verdict` | **TAKEN** | [link](https://rubygems.org/gems/verdict) | Shopify; ~691.2k downloads; 0.16.1 (2020-12-14). "Shopify Experiments" — stale but high-download, Shopify-owned. |

**Free:** `ruby-evals`, `ruby_evals`, `evalz`, `evalable`, `assess`. **Taken:** `evals`, `grade`, `rubric`, `judge`, `verdict`.

---

## 4. Ruby SDKs from eval/observability platforms

- **Braintrust — official Ruby SDK (beta).** Two official packages under [braintrustdata](https://github.com/braintrustdata): [`braintrust-sdk-ruby`](https://github.com/braintrustdata/braintrust-sdk-ruby) (tracing & evals, explicitly BETA, auto-instruments OpenAI/Anthropic/**RubyLLM**, latest 0.4.0 2026-06-15) and [`braintrust-ruby`](https://github.com/braintrustdata/braintrust-ruby) (Stainless-generated REST client). Also ships Go and .NET SDKs. **The only platform with a first-party Ruby SDK.**
- **Langfuse — no official Ruby; community gems only.** The [SDK overview](https://langfuse.com/docs/sdk/overview) lists only Python (v4) and JS/TS (v5) as official; other languages are directed to the OTel endpoint + public API. Community gems: [`langfuse-rb`](https://rubygems.org/gems/langfuse-rb) (SimplePractice), [`langfuse-ruby`](https://rubygems.org/gems/langfuse-ruby) (ai-firstly), [`langfuse`](https://rubygems.org/gems/langfuse). Maintainer confirms Ruby is community-maintained, not official ([discussion #9501](https://github.com/orgs/langfuse/discussions/9501)).
- **LangSmith — Python/TS only, no Ruby.** Official [`langsmith-sdk`](https://github.com/langchain-ai/langsmith-sdk) ships Python + JS/TS packages only ([reference](https://docs.langchain.com/langsmith/reference)). No Ruby. (A secondary snippet claiming Go/Java could not be confirmed from the repo README; irrelevant to Ruby.)
- **Arize Phoenix / Arize AX — no Ruby.** Instrumentation is via [OpenInference](https://github.com/Arize-ai/openinference) (OTel conventions) for Python, JS/TS, Java, Go only — **no Ruby package** (repo language breakdown confirms). A Ruby app could hand-emit OTel/OpenInference spans, but there is no Ruby instrumentation library. ([Phoenix docs](https://arize.com/docs/phoenix))
- **Helicone — no SDK by design (proxy).** Integration is "change the base URL and API key" ([gateway docs](https://docs.helicone.ai/getting-started/integration-method/gateway)); Ruby works by pointing an existing OpenAI Ruby client at the proxy. Helicone even documents a Ruby-specific quirk — `helicone-stream-force-format` header needed "for libraries that do not inherently support [stream formatting], such as Ruby" ([header directory](https://docs.helicone.ai/helicone-headers/header-directory)).
- **promptfoo — Node CLI, Ruby via script-provider only.** `npx promptfoo@latest eval` ([getting started](https://www.promptfoo.dev/docs/getting-started/)). It shells out to Ruby via a [Ruby provider](https://www.promptfoo.dev/docs/providers/ruby/) (`file://my_provider.rb`, implements `call_api`) or [exec provider](https://www.promptfoo.dev/docs/providers/custom-script/). This is script invocation from Node, **not a Ruby SDK** — you cannot drive promptfoo from Ruby.

**Positioning implication.** Ruby is a near-total gap. Only Braintrust ships a first-party Ruby SDK (beta); Langfuse/LangSmith/Arize are officially Python/TS-only (some add Java/Go, never Ruby); Helicone sidesteps SDKs; promptfoo can only call Ruby scripts. A genuinely idiomatic, first-class, non-beta Ruby eval/observability library would have essentially no direct native competitor, and can differentiate against OTel/OpenInference by offering native Ruby instrumentation instead of hand-rolled span emission.

---

## 5. Recording / caching prior art (VCR & WebMock)

- **VCR** ([repo](https://github.com/vcr/vcr), [gem](https://rubygems.org/gems/vcr)) records HTTP interactions into inspectable YAML/JSON "cassettes" for fast, deterministic replay. It doesn't intercept HTTP itself; it plugs into a backend via `hook_into :webmock | :typhoeus | :faraday | :excon`. Through WebMock it covers Net::HTTP and everything built on it (HTTParty, RestClient, Mechanize). Request matching is configurable on method, URI, host, path, query, body (incl. `:body_as_json`), headers, or a custom matcher; documented default is `[:method, :uri]`. Latest **6.4.0 (2025-12-22)**, ~160M downloads, Ruby ≥ 2.7. Actively released but explicitly asking for more maintainers.
- **WebMock** ([repo](https://github.com/bblimke/webmock), [gem](https://rubygems.org/gems/webmock)) stubs/matches requests but stores no responses itself — VCR layers persistence on top and it is VCR's default backend. Latest **3.26.2 (2026-03-18)**, ~409M downloads, actively maintained.

**Composition with LLM clients (non-streaming): clean.** The two dominant client transports are exactly what VCR/WebMock cover — ruby-openai is Faraday-based (and uses VCR in its own suite), the official openai SDK uses net/http. So recorded non-streaming request/response works. ([ruby-openai README](https://raw.githubusercontent.com/alexrudall/ruby-openai/main/README.md))

**Streaming / SSE friction: the core problem.**
- **Record side (hard blocker):** [webmock#742](https://github.com/bblimke/webmock/issues/742) — "WebMock … mangles and unbuffers streamed responses," **open since 2018-01-28**. A response that natively arrives as 5 chunks arrives, with WebMock in the path, as a single concatenated body. Because VCR records *through* WebMock, cassettes capture the collapsed body — SSE chunk boundaries and timing are lost at record time.
- **Replay side (partial, awkward):** WebMock can yield multiple chunks only if the stub body is an explicit array of strings ([webmock#629](https://github.com/bblimke/webmock/issues/629), open since 2016). VCR cassettes serialize the body as a single string, so a normal cassette replays streamed output as one chunk — a `stream:`/`on_data` callback fires once with the whole payload, not token-by-token.
- **Upstream fix stalled:** [webmock#1051](https://github.com/bblimke/webmock/pull/1051) ("implement body streaming for Net::HTTP") was **closed unmerged** over API-design disagreement; native chunked-body streaming for Net::HTTP is not a settled feature.
- Related streaming plumbing bugs (Typhoeus FrozenError #1078, HTTP.rb "body already consumed" #1017/#361, etc.) reinforce that streaming is a rough edge across this stack.

*Could not verify:* a first-party VCR/ruby-openai/ruby_llm doc explicitly stating "VCR cannot faithfully replay SSE chunks" — the conclusion is assembled from the WebMock issues above plus how VCR serializes bodies. (Note: R-ecosystem issue `ropensci/vcr#1046` is easy to mis-cite — it is *not* the Ruby gem.)

**Prior art of VCR + OpenAI:** ruby-openai commits VCR cassettes and runs specs offline against them (PRs [#165](https://github.com/alexrudall/ruby-openai/pull/165), [#252](https://github.com/alexrudall/ruby-openai/pull/252)) — strong prior art, but predominantly non-streaming.

**Assessment: do not make VCR the core replay path for streaming; build a purpose-built, LLM-aware cache.** VCR/WebMock structurally lose streaming fidelity at record time, VCR's matching value (body/`body_as_json`) is cheap to replicate with a request-hash→response cache, and betting the core replay path on an unsolved streaming gap in a thinly-staffed dependency is added risk. VCR/WebMock remain fine for incidental non-streaming HTTP.

---

## Implications for the library

### Positioning
- **First-mover in a near-empty native niche.** No mature, framework-agnostic Ruby eval library exists; platform Ruby SDKs are absent or beta. Aim to be *the* idiomatic, non-beta, framework-agnostic Ruby evals library.
- **Differentiate from the young RubyLLM-coupled gems** (`leva`, `ruby_llm-evals`, `ruby_llm-tribunal`) by being **not** hard-wired to RubyLLM or Rails — support RubyLLM as the first-class adapter but keep a provider-neutral core so it also works with the official OpenAI/Anthropic SDKs and plain HTTP.
- **Avoid the `evals` name** (taken + direct topical collision). `ruby-evals`/`ruby_evals` are free and on-brand; verify with a `gem push` dry-run for similarity blocks.
- **Interop, don't compete, with observability platforms:** offer export to Langfuse/Braintrust/OTel rather than reinventing dashboards. Braintrust already auto-instruments RubyLLM — align with that data shape.

### Provider abstraction
- **Target RubyLLM's 1.x public API as the primary adapter:** consume `chat.messages` (`role`/`content`/`tool_calls`/`tokens`), `message.tokens` + cost helpers, and — most importantly — the `ActiveSupport::Notifications` / `config.instrumenter` event stream (`chat.ruby_llm`, `tool_call.ruby_llm`, etc.) as the canonical trace source. This gives traces, tool calls, token usage, and timing "for free."
- **Define a small internal trace/run model** (messages, tool calls, token usage, timing, cost) and map each provider into it. For the official OpenAI/Anthropic SDKs and ruby-openai, derive it from raw response objects (`usage`, `tool_calls`); for ActiveAgent/langchainrb, from their message/callback APIs. Don't let RubyLLM's object shapes leak into the public eval API — no long-term stability guarantee exists there.
- **Reuse `informers`** as an optional local backend for embedding-similarity and classification-based scorers (no API cost).

### Caching
- **Build a thin, LLM-aware deterministic cache as the core replay path.** Canonicalize the request (model + messages/params) into a stable key; store the response — for streaming, the **ordered raw SSE event list** with framing intact — and on replay re-emit events through the client's own `stream:`/SSE handler so streaming code paths are exercised faithfully.
- **Keep VCR/WebMock optional, for incidental non-streaming HTTP only** (proven by ruby-openai's own specs). Do not rely on it for token-by-token streaming determinism.
- Mirror VCR's ergonomics that developers already know (cassette-style files, request matching, secret filtering) so the custom cache feels familiar — but own the storage format for streaming.

---

## Sources

**RubyLLM:** [repo](https://github.com/crmne/ruby_llm) · [rubygems](https://rubygems.org/gems/ruby_llm) · [chat](https://rubyllm.com/chat/) · [tools](https://rubyllm.com/tools/) · [models](https://rubyllm.com/models/) · [streaming](https://rubyllm.com/streaming/) · [instrumentation](https://rubyllm.com/instrumentation/) · [agentic workflows](https://rubyllm.com/agentic-workflows/) · [upgrading](https://rubyllm.com/upgrading/) · [ecosystem](https://rubyllm.com/ecosystem/) · [Rails](https://rubyllm.com/rails/) · [1.16.0 release](https://github.com/crmne/ruby_llm/releases/tag/1.16.0)

**Other Ruby AI libraries:** [langchainrb](https://github.com/patterns-ai-core/langchainrb) · [ruby-openai](https://github.com/alexrudall/ruby-openai) · [openai-ruby](https://github.com/openai/openai-ruby) · [anthropic-sdk-ruby](https://github.com/anthropics/anthropic-sdk-ruby) · [Raix](https://github.com/OlympiaAI/raix) · [ActiveAgent](https://github.com/activeagents/activeagent) · [sublayer](https://github.com/sublayerapp/sublayer) · [informers](https://github.com/ankane/informers) · [aia](https://github.com/MadBomber/aia)

**Ruby eval gems & names:** [leva](https://github.com/kieranklaassen/leva) · [ruby_llm-evals](https://github.com/sinaptia/ruby_llm-evals) · [ruby_llm-tribunal](https://github.com/Alqemist-labs/ruby_llm-tribunal) · [evals gem](https://rubygems.org/gems/evals) · [grade](https://rubygems.org/gems/grade) · [rubric](https://rubygems.org/gems/rubric) · [judge](https://rubygems.org/gems/judge) · [verdict](https://rubygems.org/gems/verdict)

**Platform SDKs:** [Braintrust Ruby SDK](https://github.com/braintrustdata/braintrust-sdk-ruby) · [braintrustdata org](https://github.com/braintrustdata) · [Langfuse SDK overview](https://langfuse.com/docs/sdk/overview) · [Langfuse discussion #9501](https://github.com/orgs/langfuse/discussions/9501) · [LangSmith SDK](https://github.com/langchain-ai/langsmith-sdk) · [OpenInference](https://github.com/Arize-ai/openinference) · [Helicone gateway](https://docs.helicone.ai/getting-started/integration-method/gateway) · [Helicone headers](https://docs.helicone.ai/helicone-headers/header-directory) · [promptfoo Ruby provider](https://www.promptfoo.dev/docs/providers/ruby/)

**VCR / WebMock:** [VCR repo](https://github.com/vcr/vcr) · [VCR gem](https://rubygems.org/gems/vcr) · [WebMock repo](https://github.com/bblimke/webmock) · [WebMock gem](https://rubygems.org/gems/webmock) · [webmock#742](https://github.com/bblimke/webmock/issues/742) · [webmock#629](https://github.com/bblimke/webmock/issues/629) · [webmock#1051](https://github.com/bblimke/webmock/pull/1051)
