# Synchronized AI Engineering Interview Plan
## Company-wise interview taxonomy × 4-Week Agentic AI Engineering Intensive

This file maps the uploaded four-week learning syllabus to the topic organization used by the public reference repository: https://github.com/pallavi-shekhar/ai-engineering-interview-questions-company-wise.

**Source distinction:** PDF topics, builds, weekly goals and milestones are the learning sequence. Common/company-tagged interview questions belong to the reference repository. Any prompt explicitly marked **PDF-derived practice** is created from the syllabus and is not a claim that a company asked it. Company tags should be treated as repository-reported, not independently verified interview transcripts.

## Topic mapping

| Interview category | Roadmap days | Priority |
|---|---:|---|
| LLM Internals and Architecture | 1–2, 8, 12 | Core |
| Inference, Serving and GPU Performance | 1, 20, 24–26 | Core engineering extension |
| RAG and Retrieval | 8–11 | Core |
| Agents and Tool Use | 3–6, 11, 13, 15–18 | Core |
| Fine-Tuning, Post-Training and Alignment | No dedicated day | Role-dependent extension |
| Evaluation and Observability | 19–20, 25–28 | Core |
| Safety, Security and Responsible AI | 22, 25–26 | Core |
| AI System Design | 25–27 | Core |
| Coding and Data Structures | 2–6, 8–10, 15–16, 23–27 | Applied coding |
| Multimodal, Speech and Voice AI | No dedicated day | Role-dependent extension |

## 28-day synchronized schedule

Each day: learn the PDF concept, answer mapped questions, implement the PDF build, and commit a short explanation plus test evidence.

### Week 1 — LLM foundations and agent loop

| Day | Learn from the roadmap | Interview mapping / practice | Build |
|---|---|---|---|
| 1 | Tokens, context window, attention, KV cache, temperature/top-p, stop sequences, prompts, hallucinations, cost/latency | `01-llm-fundamentals`; LLM internals and inference/serving. Practice: context growth and memory implications of KV cache. | CLI chatbot with streaming, history, token and cost counter |
| 2 | Prompt patterns, few-shot, JSON/XML, JSON Schema, Pydantic and repair | `02-prompting-structured-output`; coding and tool-use overlap. Practice: bounded validation/repair and semantic checks. | Validated messy-text extractor with retry and tests |
| 3 | Tool schemas, model tool selection, descriptions, parallel calls, arguments, timeouts and exceptions | `03-tool-calling-agent-loops`; Agents and Tool Use. Practice: safe parallelism and argument validation. | Calculator, search, file and sandboxed Python tools |
| 4 | Observe–reason–act–observe, ReAct, stopping conditions, budgets, loop detection; workflow vs agent | `03-tool-calling-agent-loops`, `04-agent-patterns-reliability`. Practice: define termination and when not to use an agent. | Plain-Python ReAct agent (~150 lines) |
| 5 | Plan-and-execute, reflection, router, chaining, parallelism, orchestrator-worker | `04-agent-patterns-reliability`; Agents and Tool Use / AI System Design. Practice: compare pattern trade-offs. | Add planner and critic; compare across 10 tasks |
| 6 | Infinite loops, hallucinated arguments, context bloat and compounding errors | `04-agent-patterns-reliability`, `09-security-production`, `08-evaluation-observability`. Practice: recovery from malformed or timed-out tools. | Failure injection; finish Project 1 — From-Scratch Agent |
| 7 | Review Week 1; no new topic | Explain ReAct, workflow vs agent, tool calls, limits and recovery aloud. | Re-run tests, review gaps and clean README |

### Week 2 — RAG, memory, context engineering and MCP

| Day | Learn from the roadmap | Interview mapping / practice | Build |
|---|---|---|---|
| 8 | Embeddings, cosine similarity, ANN/HNSW, pgvector vs Qdrant vs Pinecone | `05-rag-retrieval`; RAG and coding. Practice: embedding batch constraints. | Ingest real docs/PDF/code into pgvector |
| 9 | Fixed/recursive/semantic/structure-aware chunking, overlap, metadata and access filters | `05-rag-retrieval`. Practice: chunk boundaries, retrieval quality and per-user access control. | Baseline RAG + 40-question labelled set |
| 10 | Hybrid BM25/vector search, cross-encoder reranking, query rewriting, multi-query, HyDE, Recall@K and MRR | `05-rag-retrieval`; inference trade-offs. Practice: choose 10 passages from 50. | Add retrieval techniques individually and measure Recall@K/MRR |
| 11 | Agentic RAG, iterative retrieval, evidence sufficiency, RAG vs long context/SQL/fine-tuning, citations | `05-rag-retrieval`, `03-tool-calling-agent-loops`. Practice: enterprise RAG with permission checks. | RAG as an agent tool with citations |
| 12 | Short/long-term and episodic/semantic memory, summarization, trimming, prompt caching and context ordering | `06-memory-context-mcp`; LLM internals and inference. Practice: stale/contradictory memory. | Persistent preference and session memory in Postgres |
| 13 | MCP host/client/server, tools/resources/prompts/transports; A2A overview | `06-memory-context-mcp`, tool integration. Practice: native, OpenAPI and MCP tool layer. | MCP server for RAG search and database tool |
| 14 | Review retrieval, metrics, access control, memory and MCP | `05-rag-retrieval`, `06-memory-context-mcp`. | Project 2 — Agentic RAG with Metrics; publish baseline comparison |

### Week 3 — Frameworks, multi-agent systems and evaluations

| Day | Learn from the roadmap | Interview mapping / practice | Build |
|---|---|---|---|
| 15 | LangGraph state, nodes, edges, conditional routing and reducers | `07-langgraph-multi-agent`; Agents and AI System Design. Practice: map ReAct to graph nodes. | Port Week 1 agent to LangGraph |
| 16 | Checkpointing, persistence, interrupts, streaming and subgraphs | `07-langgraph-multi-agent`, `09-security-production`. Practice: approve, pause and resume risky actions. | Approval gate and crash/resume test |
| 17 | OpenAI Agents SDK, Claude Agent SDK, Google ADK, CrewAI, AutoGen; framework vs plain code | `07-langgraph-multi-agent`; Agents and System Design. Verify current APIs before implementation. | Small agent in a second framework; comparison notes |
| 18 | Supervisor, hierarchical, handoff/swarm, debate, shared state vs message passing | `07-langgraph-multi-agent`, `11-system-design`, evaluation. Practice: justify multi-agent overhead. | Planner + parallel researchers + writer + critic |
| 19 | Golden datasets, unit-style evals, LLM-as-judge bias, trajectory evaluation | `08-evaluation-observability`. Practice: build eval set without perfect labels. | 50+ eval cases with pytest runner |
| 20 | Regression testing, benchmark limits, Langfuse/LangSmith, OpenTelemetry, cost/latency dashboard | `08-evaluation-observability`; inference/serving. Practice: investigate p95 time-to-first-token regression. | Traces and dashboard; compare single- vs multi-agent |
| 21 | Review Week 3 and begin applications as source roadmap recommends | Explain non-deterministic evaluation, judge bias and multi-agent trade-offs. | Consolidate gaps and benchmark report |

### Week 4 — Security, production, capstone and interview preparation

| Day | Learn from the roadmap | Interview mapping / practice | Build |
|---|---|---|---|
| 22 | Direct/indirect prompt injection, exfiltration, least privilege, lethal trifecta, sandboxing and approvals | `09-security-production`; Safety, Security and Responsible AI. Practice: least-privilege refund agent. | Attack agent with injected documents; add regression tests |
| 23 | Retries/backoff, timeouts, circuit breakers, fallbacks, idempotency and durable execution | `09-security-production`, `11-system-design`. Practice: safe retries and deduplication. | Resume agent after process kill |
| 24 | Prompt caching, model routing, batching, parallel tools and streaming | `10-cost-latency-architecture`; inference/serving and LLM internals. Practice: optimize 10-step agent. | Before/after cost and latency report; claim only measured results |
| 25–26 | Production capstone: customer support/refunds, SQL analyst or research agent | `11-system-design`, security, evaluation, agents and serving. Use the five system designs from the PDF. | FastAPI, async workers, streaming, permissioned tools, approval, memory, RAG, tracing, 50+ CI evals, injection tests, Docker |
| 27 | Question bank, system designs, live coding and 2-minute project walkthrough | All common categories; prioritize role-relevant coding. | Tool loop, retriever, retry/validation wrapper and project demos |
| 28 | Two mock interviews, resume/LinkedIn polish and repository pinning | Company-specific sections plus common question base. | Two scorecards, gap list and evidence-backed resume bullets |

## Five system-design prompts from the roadmap

1. Customer-support agent that can issue refunds safely.
2. Coding agent working on a large repository.
3. Enterprise document Q&A with per-user permissions.
4. Long-running research agent that survives crashes.
5. Multi-tenant agent platform where customers bring their own tools.

For each design, cover architecture, tools, memory, retrieval, evaluations, failure handling, security, cost and monitoring.

## Priority order

**Must-master overlap:** workflow vs agent, ReAct and tool loops; structured output and validation; RAG chunking/hybrid search/reranking/metrics/access control; memory and MCP; LangGraph checkpoints and human approval; multi-agent trade-offs; evaluation and regression; prompt injection, least privilege, idempotency, crash recovery; AI system design, cost and observability.

**Role-dependent extensions:** deep transformer internals (including MQA/GQA/MLA, MoE, normalization and scaling laws), GPU serving/quantization/KV-cache compression, fine-tuning/SFT/RLHF/RLAIF/LoRA, multimodal/speech/voice and advanced ML research. These topics appear in the interview taxonomy but do not have dedicated teaching modules in the four-week PDF. Prioritize them only when the target role requires them.

## Company-wise workflow

1. Start with common questions, then choose a target company and role.
2. Read its company section and reported interview-loop notes.
3. Tag each company question with topic-bank areas and PDF day numbers.
4. Separate verified, reported and practice-only questions. Record source/confidence; never invent company attribution.
5. Track each item as `Not started`, `Practiced`, `Can explain` or `Can implement`.

## Daily completion checklist

- [ ] Read the day's PDF concept.
- [ ] Attempt mapped interview questions without notes.
- [ ] Write answers with trade-offs and failure modes.
- [ ] Complete the practical build.
- [ ] Add tests and commit working code.
- [ ] Record one failure/lesson in `notes/what-broke-and-why.md`.
- [ ] Explain the topic aloud in two minutes.
