# Applied AI Engineer Roadmap

Reach for this when planning what to learn, what to skip, and what to build. Six stages, 24 weeks, one capstone.

The title is not earned by using Spring AI. It is earned by being able to answer, with numbers from your own system: **"how do you know that change made the AI feature better and not worse?"**

## What the job is (and isn't)

| | |
|---|---|
| **Applied AI engineer** | Ships LLM-backed features into production and owns their behaviour, cost, and failure modes. Orchestration, retrieval, evaluation, operations. |
| **ML engineer** | Trains and fine-tunes models. Backprop, CUDA, distributed training. Different role, different hiring pipeline. |

You do not need a GPU. You need to know what an embedding **is**, not how to train one. Fine-tuning is a last resort after retrieval and prompting are exhausted.

## The seven competencies

| Competency | What "good" looks like | Stage |
|---|---|---|
| Retrieval | Hybrid search, reranking, chunking chosen from measurement not a blog post | 02 |
| Evaluation | Golden dataset, LLM-as-judge, regression gate in CI that can fail a build | 03 |
| Structured output | Schema-constrained responses, validation, repair loop, no regex over prose | 01 |
| Tools & agents | Step budgets, approval gates, audit trail, graceful failure on tool error | 04 |
| Context engineering | Deliberate token budget per request; you can say where every token goes | 01-04 |
| Safety | Prompt-injection posture, PII handling, output validation before side effects | 05 |
| Operations | Cost per request, p99 latency, provider fallback, caching, per-tenant limits | 05 |

---

## Stage 00 - Mental model & Python beachhead

**Weeks 1-2.** Buying vocabulary and a Python environment you're not afraid of.

- Tokenization, context windows, why token count != word count. Compute a request's cost by hand before a library does it for you.
- Embeddings as geometry: cosine similarity, dimensionality, why two paraphrases land close together.
- Sampling parameters (temperature, top-p) and when determinism matters.
- Pricing mechanics: input vs output tokens, cached input, why output tokens dominate.
- Python to a working standard: `uv`, virtualenvs, notebooks, pandas. Not Django. Not FastAPI yet.

| Kind | Resource |
|---|---|
| Video | [Karpathy - Deep Dive into LLMs like ChatGPT](https://www.youtube.com/watch?v=7xTGNNLPyMI) - best 3 hours for intuition |
| Book | [Chip Huyen - AI Engineering](https://www.oreilly.com/library/view/ai-engineering/9781098166298/) - ch. 1-3 now, rest as you hit each stage |
| Docs | [Anthropic API essentials](https://docs.claude.com/en/docs/build-with-claude/overview) |
| Tool | [uv](https://docs.astral.sh/uv/) - the only Python packaging tool you need |
| Course | [Anthropic courses repo](https://github.com/anthropics/courses) - free notebooks |

!!! success "Gate"
    You can predict, within ~20%, the token count and dollar cost of a request before sending it - and explain why the same question costs 8x more with 30 retrieved chunks attached.

---

## Stage 01 - LLM mechanics inside Spring Boot

**Weeks 3-5.** Home turf. Make the model a well-behaved dependency like any other remote call.

- Spring AI `ChatClient` fluent API, prompt templates, advisors, chat memory.
- Structured output: `BeanOutputConverter` into records, JSON-schema constrained responses, repair loop on parse failure.
- Streaming with SSE end to end - and why streaming changes error handling.
- Provider as unreliable network dependency: timeouts, `@Retryable`, `@ConcurrencyLimit`, circuit-breaking on 429s.
- Prompt versioning - prompts are code, they belong in git, not in a string constant someone edits in prod.

| Kind | Resource |
|---|---|
| Docs | [Spring AI reference](https://docs.spring.io/spring-ai/reference/) |
| Code | [spring-projects/spring-ai](https://github.com/spring-projects/spring-ai) - read the tests, not the READMEs |
| Essay | [Eugene Yan - Patterns for building LLM systems](https://eugeneyan.com/writing/llm-patterns/) |
| Reference | [Instructor docs](https://python.useinstructor.com) - Python, but the patterns transfer |

!!! note "Ship - M1"
    A service that answers as a validated record, streams to the client, survives a provider timeout, and logs tokens in/out per request.

---

## Stage 02 - Retrieval, the part everyone does badly

**Weeks 6-9.** Naive cosine similarity over 512-token chunks is what everyone ships and nobody measures.

- Chunking: fixed, recursive, semantic, parent-document / small-to-big, and the metadata that rides along.
- pgvector: HNSW vs IVFFlat, `m` / `ef_construction` / `ef_search`, and the recall-vs-latency tradeoff you're actually choosing.
- Hybrid search: Postgres full-text (`tsvector`) fused with vector results via Reciprocal Rank Fusion.
- Reranking with a cross-encoder - usually the single highest-leverage quality fix in the pipeline.
- Query transformation: rewriting, decomposition, HyDE. Citation-grounded answers so every claim points at a chunk.
- Corpus versioning - re-embedding is a migration, treat it like [Flyway](../data/flyway.md).

| Kind | Resource |
|---|---|
| Docs | [pgvector](https://github.com/pgvector/pgvector) - read the indexing section twice |
| Docs | [Postgres full-text search](https://www.postgresql.org/docs/current/textsearch.html) - the BM25 half |
| Essay | [Jason Liu - systematic RAG improvement](https://jxnl.co/writing/) |
| Docs | [Sentence-Transformers](https://sbert.net) - bi-encoders vs cross-encoders |
| Model | [BGE reranker v2](https://huggingface.co/BAAI/bge-reranker-v2-m3) - open, self-hostable |

!!! note "Ship - M2/M3"
    Ingestion pipeline plus hybrid retrieval with reranking over a corpus you know well enough to spot a wrong answer instantly.

---

## Stage 03 - Evaluation (the hinge)

**Weeks 10-13.** If you only do one stage properly, do this one. Almost nobody in the Java ecosystem has it.

- Golden dataset by hand - 50-100 real questions with known-good answers and known-relevant sources. No shortcut; you write these yourself.
- Error analysis: read 100 real outputs, label failure modes, count them. Highest-value activity in applied AI, and it's unglamorous manual work.
- Retrieval metrics: recall@k, precision@k, MRR, NDCG - and which matches your product's actual failure cost.
- Generation metrics: faithfulness/groundedness, answer relevance, citation accuracy.
- LLM-as-judge: rubric design, pairwise vs pointwise, position bias, validating the judge against your own labels.
- Regression gating in CI: a threshold, a diff between runs, a build that fails when quality drops.

| Kind | Resource |
|---|---|
| Essay | [Hamel Husain - Your AI product needs evals](https://hamel.dev/blog/posts/evals/) - canonical, read it three times |
| Essay | [Hamel Husain - LLM-as-a-Judge](https://hamel.dev/blog/posts/llm-judge/) |
| Tool | [Ragas](https://docs.ragas.io) - RAG-specific metrics |
| Tool | [promptfoo](https://www.promptfoo.dev) - runs cleanly in CI |
| Tool | [DeepEval](https://github.com/confident-ai/deepeval) - pytest-shaped |

!!! success "Gate - the one that earns the title"
    You change the chunk size, run one command, and get a scorecard: retrieval recall@5 moved 0.71 -> 0.78, faithfulness held at 0.89 - and CI would have blocked the change if it hadn't.

---

## Stage 04 - Tools, agents, MCP

**Weeks 14-17.** Where the model stops answering and starts doing. Almost entirely about constraint.

- Spring AI `@Tool` methods, schema generation from records, tool result handling.
- Model Context Protocol - server and client, the integration boundary worth standardising on.
- Workflow (you control the path) vs agent (the model does). **Default to workflows.**
- Control surfaces: max-step budget, max-token budget, tool allow-lists, human approval before writes.
- Idempotency and compensation - the model *will* call the same tool twice.
- Audit trail: every tool call, arguments, result, cost - queryable after the fact.

| Kind | Resource |
|---|---|
| Essay | [Anthropic - Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) |
| Essay | [Anthropic - Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) |
| Spec | [Model Context Protocol](https://modelcontextprotocol.io) - read it, then build one server |
| Docs | [Spring AI tool calling](https://docs.spring.io/spring-ai/reference/api/tools.html) |

!!! note "Ship - M5"
    An agent that reads and writes your domain DB through tools, refuses to exceed its step budget, and cannot write without an approval record.

---

## Stage 05 - Production

**Weeks 18-24.** What separates a portfolio project from something a company lets near customers.

- Prompt injection, direct and indirect. Retrieved documents are untrusted input. No prompt-level instruction is a real defence - **the boundary is what the tools are allowed to do**.
- PII detection and redaction before text leaves your network.
- Output validation before side effects: schema, business rules, refusal path.
- Semantic caching - and the correctness trap of serving a cached answer to a near-miss question.
- Provider abstraction: fallback, routing by cost/capability, degradation when everything is down.
- Token accounting and budget enforcement per tenant, per user, per feature.
- OTel GenAI semantic conventions - spans for retrieval, rerank, generation, tool calls, with token and cost attributes.
- Multi-tenant isolation in a vector store that a crafted query cannot bypass.

| Kind | Resource |
|---|---|
| Standard | [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) |
| Blog | [Simon Willison on prompt injection](https://simonwillison.net/tags/prompt-injection/) |
| Spec | [OTel GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) |
| Tool | [Langfuse](https://langfuse.com/docs) - tracing, cost, eval storage; self-hostable |
| Tool | [Arize Phoenix](https://arize.com/docs/phoenix) - OTel-native tracing and eval UI |

!!! success "Gate"
    You can open a dashboard and state this month's cost per answered question, the p99 latency split by retrieval / rerank / generation, and the injection attempt your guardrail blocked on the 14th.

---

## What to learn besides Java

| Skill | Priority | Scope |
|---|---|---|
| **Python** | Required | Evals, data prep, embedding experiments. Every AI library ships here first; the eval tooling has no Java equivalent worth using. `uv`, pandas, Jupyter, pytest, one eval framework. Not web frameworks. |
| **SQL & pgvector** | Required | Retrieval is a database problem in an AI costume. Index types, recall tuning, query plans on hybrid queries, tenant-safe filtering. |
| **ML literacy** | Required | Conceptual only: tokenization, embeddings, attention at intuition depth, why models hallucinate, bi-encoder vs cross-encoder. No maths homework. |
| **TypeScript** | High | Without a frontend your demos are curl commands. One framework, the Vercel AI SDK, streaming and citation rendering. |
| **K8s / infra** | High | Enough to run a self-hosted reranker or embedding model next to your app and schedule re-embedding jobs. |
| **Go** | Later | Revisit after the capstone, for infrastructure reasons rather than AI ones. Node as a backend language: skip. |

---

## The capstone - Atlas

A tenant-aware, evaluated, observable AI knowledge and action platform. Built in Spring Boot 4, grown one stage at a time. **One system, not six demos.**

Pick a domain you genuinely know - your own docs, a public regulatory corpus, a football statistics archive. Domain familiarity is what makes your golden dataset trustworthy, and the golden dataset is what makes the whole thing credible.

| Slice | Stack |
|---|---|
| Ingestion & corpus - versioned, configurable chunking, re-embedding as migration | Spring Boot 4, Flyway, virtual threads, Postgres 18 |
| Hybrid retrieval - HNSW + full-text, RRF fusion, cross-encoder rerank | pgvector, tsvector, BGE reranker |
| Answer service - streaming, citation-grounded, refusal path on low confidence | Spring AI ChatClient, SSE, BeanOutputConverter |
| Action layer - tools over domain DB, step budgets, approval gates, audit | Spring AI `@Tool`, MCP |
| Eval harness - scorecard, run diff, PR gate | Python, uv, Ragas/promptfoo, GitHub Actions |
| Gateway & guardrails - fallback, semantic cache, budgets, PII, injection | RestClient, `@Retryable`, `@ConcurrencyLimit` |
| Observability - GenAI spans with token/cost attributes | OTLP, Micrometer, Langfuse, Grafana |
| Client - streaming chat, citations, approval prompts | TypeScript, Vercel AI SDK |

### Milestones

| # | Milestone | Acceptance |
|---|---|---|
| M1 | Answer service skeleton | Structured parse success >= 99% over 200 calls; survives a 3s provider timeout without a 500 |
| M2 | Ingestion pipeline | Re-running produces zero duplicate chunks; full rebuild is one command |
| M3 | Hybrid retrieval + rerank | Beats pure-vector recall@5 by a measured margin on the golden set |
| M4 | **Eval harness in CI** | A PR degrading faithfulness by >3 points fails CI, demonstrably, on a commit you can show |
| M5 | Action layer | Agent can't exceed step budget or write without an approval row; both proven by tests |
| M6 | Guardrails & multi-tenancy | Red-team suite of 30 injection and cross-tenant attempts passes with zero leaks |
| M7 | Cost & observability | Cost per answered question is a live number; cache brings it down measurably |
| M8 | Client, deploy, write-up | A stranger can read the README and reproduce your headline eval numbers locally |

!!! tip "Why this reads as senior"
    Almost every AI portfolio project is a chat wrapper with a vector store. Three things are rare enough to stand out: **a golden dataset you wrote by hand**, **a CI gate that can block a merge on quality**, and **cost per request as a tracked metric**.

---

## Claiming the title

When you can hold a 30-minute conversation about a system you built where every quality claim is backed by a number from your own eval harness, every cost claim by your own dashboard, and every safety claim by a test you can run on the call.

Not when you finish a course. The Spring AI API surface is a weekend; the judgement about retrieval, evaluation, and cost is the job.

### Weekly rhythm

| When | What |
|---|---|
| Mon-Tue | Learn the stage's concept - one resource properly, not five superficially |
| Wed-Thu | Build the capstone slice, with tests |
| Fri | Run the evals, record what moved and why. From stage 03 on, non-negotiable |
| Weekend | One content artefact, or rest. Not both, and not neither for three weeks running |
