# 28-Day AI Engineering Interview Preparation Plan

This plan follows the four-week structure of the uploaded roadmap. Each day includes the learning target, interview drill, practical output, and a self-check. Week 1 intentionally uses no agent framework.

## Week 1 — LLM foundations and the agent loop

**Goal:** Explain and build a working agent with no framework.

### Day 1 — How LLMs work
- Learn: tokens, context window, attention intuition, KV cache, temperature, top-p, stop sequences, system/user prompts, hallucinations, token cost, latency, reasoning vs standard models.
- Interview drill: Explain the generation lifecycle; compare temperature and top-p; explain why context length affects cost and latency.
- Build: CLI chatbot with streaming, history, token and cost counter ([project notes in this repository](../projects/project-0-day-1-cli-chatbot.md) · [standalone Gemini Python CLI chatbot repo](https://github.com/JainAditi09/ai-cli-chatbot)).
- Deliverable: [Day 1 CLI chatbot implementation](../projects/project-0-day-1-cli-chatbot.md) + notes.

### Day 2 — Prompting and structured outputs
- Learn: prompt patterns, few-shot examples, XML/JSON structure, JSON schema, Pydantic validation, retry-on-invalid-output.
- Interview drill: What makes output reliably machine-readable? How do you recover from malformed JSON?
- Build: messy-text extractor returning validated Pydantic objects with repair/retry.
- Deliverable: schema, validators, invalid-output tests.

### Day 3 — Tool calling
- Learn: tool schemas, model tool selection, parallel calls, tool results, tool descriptions, argument errors, timeouts, exceptions.
- Interview drill: Walk through a tool call from model request to result; distinguish model decision from application execution.
- Build: calculator, web search, file read/write, and Python sandbox tools wired to a model.
- Deliverable: tool registry and tests for bad arguments/timeouts.

### Day 4 — The agent loop
- Learn: observe–reason–act–observe, stopping conditions, max steps, token budgets, loop detection; workflow vs agent.
- Interview drill: Explain ReAct; when would you avoid an agent; how do you prevent infinite loops?
- Build: approximately 150-line plain-Python ReAct agent.
- Deliverable: framework-free agent and [ReAct sequence diagram + walkthrough](../projects/project-1-from-scratch-agent.md#react-agent-sequence-diagram).

### Day 5 — Agent patterns
- Learn: plan-and-execute, reflection/self-critique, router, prompt chaining, parallelization, orchestrator-worker.
- Interview drill: Select a pattern for a concrete task and justify the complexity/cost trade-off.
- Build: add planner and critic steps to Day 4 agent; compare on 10 tasks.
- Deliverable: [Day 5 agent patterns guide, planner/critic implementation, and 10-task evaluation](../projects/day-5-agent-patterns.md).

### Day 6 — Failure modes and consolidation
- Learn: infinite loops, hallucinated tool arguments, context bloat, error compounding.
- Interview drill: Diagnose a failing agent trace and propose safeguards.
- Build: failure-injection tools that fail, return junk, or time out; make the agent recover safely.
- Deliverable: [Day 6 — Agent Failure Modes, Recovery, and Consolidation](../projects/day-6-agent-failure-modes.md); Project 1 README and failure log.

### Day 7 — Review and rest
- Explain aloud: What is an agent? How does the loop work? When would you avoid agents? How do you stop loops?
- Revisit weak answers and tidy commits. Avoid adding new topics.

## Week 2 — RAG, memory, context engineering, and MCP

**Goal:** Build a measured agentic RAG system and expose tools through MCP.

### Day 8 — Embeddings and vector search
- Learn: embeddings, cosine similarity, ANN indexes (HNSW), pgvector vs Qdrant vs Pinecone.
- Interview drill: What does an embedding represent? What does cosine similarity measure? Why use ANN?
- Build: ingest a real document/PDF/codebase dataset into pgvector.
- Deliverable: ingestion script and retrieval demo.

### Day 9 — Chunking and retrieval quality
- Learn: fixed, recursive, semantic, structure-aware chunking; overlap trade-offs; metadata and access-control filters.
- Interview drill: How do you choose chunk size? How can chunk boundaries damage retrieval?
- Build: baseline RAG pipeline and 40-question test set with known answers.
- Deliverable: baseline retrieval report.

### Day 10 — Advanced retrieval
- Learn: hybrid BM25 + vector search, cross-encoder reranking, query rewriting, multi-query, HyDE.
- Interview drill: Why combine lexical and vector retrieval? What does reranking add? What do Recall@K and MRR tell you?
- Build: add techniques one at a time and record Recall@K and MRR.
- Deliverable: before/after retrieval metrics.

### Day 11 — Agentic RAG
- Learn: agent decides when/what to retrieve, query iteration, sufficiency checks; RAG vs long context vs SQL vs fine-tuning.
- Interview drill: When is RAG the wrong tool? How do citations and evidence grounding work?
- Build: turn RAG into a tool-using agent with citations.
- Deliverable: agentic RAG demo and trace.

### Day 12 — Memory and context engineering
- Learn: short/long-term memory, episodic/semantic memory, summarization, trimming, retrieval into context, prompt caching, context ordering.
- Interview drill: What should be stored as memory? How do you prevent stale or irrelevant memory from polluting context?
- Build: persistent user preferences and session memory in Postgres.
- Deliverable: memory schema and retrieval policy.

### Day 13 — MCP
- Learn: host, client, server, tools, resources, prompts, transports; read about A2A interoperability.
- Interview drill: Explain MCP components and request flow; distinguish MCP from an agent framework.
- Build: MCP server exposing RAG search and a database tool; connect it to an agent.
- Deliverable: working MCP demo and architecture diagram.

### Day 14 — Review and Project 2
- Finish Project 2: Agentic RAG with Metrics.
- Interview drill: How do you evaluate retrieval? Why hybrid search? When is RAG the wrong tool?
- Deliverable: baseline-vs-improvements results table and project walkthrough.

## Week 3 — Frameworks, multi-agent systems, and evals

**Goal:** Use LangGraph confidently, know when multi-agent is justified, and build a real eval suite.

### Day 15 — LangGraph fundamentals
- Learn: state, nodes, edges, conditional routing, reducers.
- Interview drill: Map a plain-Python agent loop into a graph; explain state transitions.
- Build: port Week 1 agent to LangGraph.
- Deliverable: graph diagram and runnable agent.

### Day 16 — LangGraph advanced
- Learn: checkpointing, persistence, human-in-the-loop interrupts, streaming, subgraphs.
- Interview drill: How do you pause for approval and resume after a crash?
- Build: approval gate before a risky tool call; resume from checkpoint after crash.
- Deliverable: interrupted/resumed execution test.

### Day 17 — Other frameworks
- Learn/breadth: OpenAI Agents SDK, Claude Agent SDK, Google ADK, CrewAI, AutoGen. Verify current documentation and APIs.
- Interview drill: Framework vs plain code; what abstraction does a framework provide and what does it obscure?
- Build: one small agent in a second framework.
- Deliverable: concise framework comparison.

### Day 18 — Multi-agent systems
- Learn: supervisor, hierarchical, handoff/swarm, debate; shared state vs message passing; benefits and coordination costs.
- Interview drill: When does specialization/parallelism justify multi-agent overhead?
- Build: planner + parallel researchers + writer + critic research system.
- Deliverable: trace and single-agent comparison.

### Day 19 — Evals part 1
- Learn: golden datasets, unit-style evals, LLM-as-judge biases (position, verbosity, self-preference), trajectory evaluation.
- Interview drill: Why judge the trajectory as well as the final answer?
- Build: 50+ evaluation cases with pytest runner.
- Deliverable: reproducible evaluation report.

### Day 20 — Evals part 2 and observability
- Learn: regression testing, SWE-bench/GAIA/tau-bench and limits, Langfuse/LangSmith tracing, OpenTelemetry basics.
- Interview drill: How do you detect regression after a prompt/model change? What should a trace contain?
- Build: tracing on every project and dashboard for cost, latency, success rate; compare multi-agent vs single-agent with numbers.
- Deliverable: trace screenshots/export and benchmark report.

### Day 21 — Review
- Explain aloud: When is multi-agent worse than single-agent? How do you evaluate a non-deterministic system? What is wrong with LLM-as-judge?
- Start/continue applications, as the source roadmap recommends applying in Week 3.

## Week 4 — Security, production, capstone, and interview prep

**Goal:** Ship one production-grade agent and become interview-fluent.

### Day 22 — Security
- Learn: direct/indirect prompt injection, data exfiltration, tool permissions, least privilege, lethal trifecta, sandboxing, output validation, approval gates.
- Interview drill: Explain the lethal trifecta and design controls for each risk.
- Build: attack your agent with injected documents and fix what breaks.
- Deliverable: threat model and attack/regression tests.

### Day 23 — Reliability and durable execution
- Learn: retries with backoff, timeouts, circuit breakers, model fallbacks, idempotent tools, durable execution (Temporal, queues, LangGraph persistence).
- Interview drill: How can a long-running agent survive process termination? What makes a retry safe?
- Build: make the agent resume correctly after a process kill.
- Deliverable: crash/recovery test and idempotency notes.

### Day 24 — Cost and latency
- Learn: prompt caching, model routing, batching, parallel tool calls, streaming.
- Interview drill: Optimize a 10-step agent without harming answer quality.
- Build: reduce cost per task by at least 40% and record before/after numbers (only claim what your measurements show).
- Deliverable: cost/latency report.

### Days 25–26 — Capstone: production-grade agent
Choose one: customer-support agent with safe refunds; data-analyst agent over SQL; or research agent.
- Requirements: FastAPI, async job workers, streaming, permissioned tools, approval gate, memory, RAG, tracing, 50+ evals in CI, prompt-injection tests, guardrails, Docker deployment/public URL if possible.
- Deliverable: architecture diagram, cost per task, failure rate, eval results, and “what failed and how I fixed it” section.

### Day 27 — Interview preparation
- Drill the question bank and system designs.
- Live-code without a framework: tool-calling loop, retriever, retry and validation wrapper.
- Prepare a two-minute walkthrough of each project.
- Deliverable: concise answer notes and project pitches.

### Day 28 — Mock interviews and polish
- Do two mock interviews (friend or self-recording).
- Update resume and LinkedIn; pin the three project repositories.
- Optional: publish a blog post.
- Deliverable: mock scorecards and final gap list.

## Completion criteria from the source roadmap

- Week 1: build/explain a tool-calling agent from scratch and show failure recovery.
- Week 2: show RAG results table and MCP server working with an agent.
- Week 3: run eval suite on LangGraph agent and defend multi-agent vs single-agent with data.
- Week 4: demo deployed agent with tracing, security tests, cost numbers, and answer the question bank fluently.
