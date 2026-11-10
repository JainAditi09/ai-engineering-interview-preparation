# AI system design — detailed interview answers

For every design, begin with requirements and assumptions. Then cover APIs, architecture/data flow, storage, authorization, async execution, observability, failure modes, cost/latency, and trade-offs. A model call is only one component of a production AI system.

## 1. Design a customer-support agent that can issue refunds safely.

**Requirements.** Understand the customer's question, fetch the correct order, explain policy, determine eligibility, and issue a refund only when authorized. Support auditability, duplicate prevention, human approval for exceptional/high-value cases, and clear user-visible status.

**Architecture.** Web/mobile client → authenticated API → conversation/job service → orchestrator/workflow → model provider. The workflow exposes narrow tools: search orders for the authenticated user, fetch policy, calculate refundable amount, request approval, and execute refund. Order and payment services remain the source of truth. Persist conversation/job state, tool audit events, and idempotency records in durable storage.

**Flow.** Authenticate → classify intent → resolve order from server-side user scope → fetch authoritative order/policy → deterministic eligibility calculation → show proposed amount/reason → approval if required → execute with idempotency key → verify payment-provider status → respond and persist result.

**Safety/reliability.** Never trust a model-generated user ID or amount. Recalculate amount in backend, verify ownership, bind approval to exact order/amount, and reconcile ambiguous timeouts before retrying. Limit model tools to the minimum. Trace every decision and side effect.

**Trade-offs.** A deterministic workflow is safer than an unconstrained agent for financial actions. The model can help interpret natural language and explain policy, but it should not own eligibility or authorization.

**Follow-ups**
- *What if payment provider times out after refund submission?* Query by idempotency/operation ID and reconcile status; do not issue a new refund blindly.
- *How do you prevent prompt injection from a customer message?* Treat message text as untrusted, expose only narrow tools, and enforce authorization and refund policy in code.
- *How do you scale during traffic spikes?* Queue non-urgent work, apply per-tenant limits, cache read-only policy data, and protect payment APIs with concurrency controls.
- *What should the UI show?* Proposed refund, approval pending, processing, completed, failed/unknown, and a support reference ID.

## 2. Design a coding agent that works on a large repository.

**Requirements.** Understand a task, inspect relevant code, plan edits, modify files, run tests, explain changes, and avoid destructive actions without approval. It needs repository-level access controls, reproducibility, and isolation for executing code.

**Architecture.** Client → task API/job queue → agent orchestrator → repository index/search service → file tools → isolated execution sandbox → test runner → diff/review service. Persist task state, patch/diff, tool traces, and evaluation results. Use version control worktrees or isolated clones per task.

**Flow.** Parse request → inspect repository metadata → search symbols/files → build a scoped plan → read relevant files → propose patch → apply in isolated workspace → run targeted tests/lint/type checks → inspect diff → request approval if needed → publish branch/patch and summary.

**Safety.** Tools should be constrained to a workspace; sandbox network and secrets; validate paths against traversal; do not execute arbitrary commands on the host. Require approval before pushing, merging, deleting, or deploying. Treat repository files and issue text as untrusted instructions.

**Trade-offs.** Full-repository indexing speeds discovery but adds index freshness and storage complexity. Broad context may help architecture understanding but is costly; retrieve relevant symbols and dependencies. Autonomous edits are convenient but require robust diff review and tests.

**Follow-ups**
- *How do you keep code context fresh?* Re-index changed files or use commit-aware retrieval and verify file versions before patching.
- *What if tests fail because of environment issues?* Separate code failure from infrastructure failure and report reproducible logs.
- *How do you prevent destructive tool calls?* Workspace isolation, command allowlists, read/write separation, approval, and audit.
- *How do you measure success?* Patch acceptance, test pass rate, regression rate, time to resolution, and human review effort.

## 3. Design enterprise document Q&A with per-user permissions.

**Requirements.** Ingest documents from multiple sources, answer questions with citations, enforce user/tenant ACLs, handle document updates/deletions, and support audit and retention policies.

**Architecture.** Connectors → ingestion queue → parser/OCR → metadata/ACL extraction → chunker → embedding service → vector/lexical indexes. Query API authenticates user → authorization-aware retriever → reranker → grounded LLM response → citation validator. Store document metadata, ACLs, versions, chunks, and audit records in durable databases.

**Flow.** Sync source → parse and normalize → chunk with document structure → embed/index → user query → derive tenant/user from auth → apply mandatory ACL filters → retrieve and rerank → generate answer with source IDs → validate citations and permissions → return answer. Re-check authorization when citation links are opened.

**Security.** Never trust client-supplied tenant filters or model-generated identity. Cache keys must include tenant/ACL version. Deletion must propagate to all indexes and caches. Treat document text as untrusted content and defend against indirect prompt injection.

**Evaluation.** Retrieval Recall@K/MRR, answer correctness, groundedness, citation validity, permission leakage tests, freshness lag, latency, and cost.

**Follow-ups**
- *How do you handle document permission changes?* Version ACL metadata, invalidate caches, and update/delete index records promptly; re-check access at read time.
- *How do you prevent cross-tenant leakage?* Server-derived scope, mandatory filters, defense-in-depth authorization, tenant-aware caches, and adversarial isolation tests.
- *What if the answer is not in the corpus?* Abstain or say evidence was not found; do not invent a response.
- *How do you support citations?* Preserve document ID/version/chunk offsets and resolve links through an authorization-checked endpoint.

## 4. Design a long-running research agent that survives crashes.

**Requirements.** Decompose research, search approved sources, collect citations, synthesize findings, pause for approval if needed, stream progress, and recover after worker/process failure.

**Architecture.** API → durable job table → queue → worker/orchestrator → search/retrieval tools → source store → synthesis/evaluation → result store. Checkpoint task state and completed subtask outputs. Frontend polls or subscribes to SSE/WebSocket events using a stable job ID.

**Flow.** Create idempotent job → enqueue → plan research subtasks → execute bounded searches → persist each source/result → checkpoint → synthesize → verify citations → save final output. On worker crash, lease expires and another worker resumes from last checkpoint. Each external side effect is idempotent or reconciled.

**Reliability.** Use heartbeats, leases, retry budgets, backoff, dead-letter handling, cancellation, and explicit states. A timeout after a search/write may be ambiguous; use stable operation IDs. Do not claim exactly-once execution across distributed components.

**Observability.** Track job age, stage durations, search failures, model calls, token/cost usage, citation validation, retry count, and terminal status.

**Follow-ups**
- *How do you avoid duplicate research work after restart?* Persist completed subtask IDs/results and make each task idempotent.
- *How does the UI reconnect?* Fetch current job status and replay events after the last known event ID.
- *What if a source disappears?* Preserve retrieval timestamp and permitted source snapshot/metadata; mark unavailable citations honestly.
- *How do you cap spend?* Per-job token/search budgets, deadlines, maximum subtasks, and tenant quotas.

## 5. Design a multi-tenant agent platform where customers bring their own tools.

**Requirements.** Multiple organizations configure agents, credentials, tools, policies, and usage budgets. Ensure strict tenant isolation, safe tool execution, versioning, audit, and fair resource allocation.

**Architecture.** Control plane manages tenant configuration, tool registry, credentials/secrets, policy, and model settings. Data plane handles authenticated requests, tenant-scoped orchestration, isolated tool adapters/workers, memory/RAG stores, and usage metering. Use separate tenant scopes and, for high-risk customers, stronger physical isolation.

**Flow.** Authenticate user → resolve tenant → load policy and allowed tools → create scoped job → model proposes tool call → validate schema → authorize against tenant policy → execute with tenant-specific credentials in isolated worker → meter usage → persist redacted trace/result.

**Security.** Store credentials in a secrets manager; never place raw secrets in prompts or logs. Enforce tenant scope in every query and tool adapter. Apply egress controls, sandboxing, rate limits, and human approval for risky actions. Treat customer-provided tool descriptions and outputs as untrusted.

**Operations.** Version tools and policies, provide revocation/rotation, monitor per-tenant budgets, isolate noisy neighbors, and support incident audit. Define data retention and deletion across memory, vectors, traces, and backups.

**Follow-ups**
- *How do you prevent one tenant's tool from being called by another?* Tenant-scoped registry and credentials, server-side authorization, isolated execution context, and negative isolation tests.
- *How do you support BYOK (bring your own key)?* Encrypt keys in a secrets manager, scope access to the tenant, rotate/revoke, and never expose keys to model context.
- *How do you limit noisy neighbors?* Per-tenant concurrency/token budgets, fair queues, quotas, and workload isolation.
- *How do you handle a malicious tool server?* Capability allowlists, network egress restrictions, response validation, timeouts, and sandbox boundaries.

## 6. For each design, explain architecture, tool permissions, memory, retrieval, evals, failure handling, security, cost, and monitoring.

**Answer framework.** Start by clarifying user types, scale, latency, availability, data sensitivity, and acceptable failure modes. Draw the synchronous path and asynchronous path separately. Identify the source of truth for each piece of data. Define which decisions are deterministic and which are model-assisted. Then cover permissions at the tool boundary, tenant-scoped memory/retrieval, offline and online evaluations, retry/idempotency semantics, prompt-injection defenses, budget limits, and trace/metric coverage.

**Follow-ups**
- *What if requirements conflict?* Make trade-offs explicit and ask which constraint is highest priority.
- *What is the first bottleneck to estimate?* Request volume, model tokens, retrieval throughput, tool dependency limits, and concurrency.
- *What should be designed first?* The critical user journey and trust boundaries, not a specific framework.

## 7. How would you handle streaming updates from async workers to a web client?

**Answer.** The API creates a durable job and returns a job ID. Workers publish status/progress events to a broker or event store. The client subscribes through SSE for one-way updates or WebSockets for bidirectional interaction. Persist events or current state so reconnecting clients can recover after a network drop. Include event IDs and monotonic sequence numbers to deduplicate/reorder safely.

**Follow-ups**
- *Why not keep the worker request open?* Long-lived HTTP requests tie up resources and do not provide durable recovery.
- *SSE or WebSocket?* SSE is simpler for server-to-client progress; WebSockets fit interactive two-way control.
- *What if events arrive twice?* Use event IDs and idempotent client reducers.
- *What if a client disconnects?* Work can continue according to policy; client reconnects and fetches status/events since its last cursor.

## 8. How would you design approval, cancellation, and resume flows for long-running agent jobs?

**Answer.** Persist job state and explicit states such as queued, running, waiting_for_approval, cancelling, interrupted, failed, and completed. An approval request contains the exact action payload, expiry, and approver policy. On approval, revalidate the payload and underlying preconditions before execution. Cancellation is a durable request checked at safe boundaries; resume loads the last checkpoint and uses idempotent side effects.

**Follow-ups**
- *Can a user cancel during an external API call?* Cancellation may be cooperative; reconcile the external operation before reporting final cancellation.
- *Can approval expire?* Yes; expired approvals should require a new decision.
- *What does resume mean after partial side effects?* Reconcile completed actions, then continue from a safe point—not blindly replay all steps.

## 9. How would you isolate tenant data across vector search, memory, traces, and tools?

**Answer.** Resolve tenant identity from authenticated server-side context. Apply tenant scope to every database/vector query, memory read/write, trace access, cache key, and tool credential. Use row-level security or separate schemas/instances as appropriate. Validate tenant ownership at service boundaries and test adversarial cross-tenant IDs. Ensure logs and dashboards enforce the same access policy.

**Follow-ups**
- *Is a tenant_id filter sent by the frontend sufficient?* No; the backend must derive and enforce it.
- *Can vector metadata filters be bypassed?* Bugs or alternate query paths can bypass them; enforce authorization at the service layer too.
- *What about cached embeddings/results?* Scope cache keys and invalidate on ACL changes.
- *How do you prove isolation?* Automated negative tests, access audits, and penetration testing.

## 10. How would you roll out a new model or prompt safely and roll it back?

**Answer.** Version the model, prompt, schema, tool definitions, and evaluation set. Run offline regression/safety tests, then deploy to a small canary cohort or shadow traffic where appropriate. Compare task success, groundedness, tool errors, latency, cost, and safety incidents by version. Increase traffic gradually if thresholds hold. Keep the previous configuration available and roll back routing/configuration quickly if regressions occur.

**Follow-ups**
- *What is shadow traffic?* Requests are evaluated by a candidate system without using its response as the user-visible result.
- *What should trigger rollback?* Critical safety violations, schema failures, significant task-success regression, or SLA/cost breaches.
- *Why version schemas too?* Model changes can alter output shape; compatibility must be tested independently.
- *How do you avoid comparing different user populations?* Use controlled assignment and segment by task type/tenant while respecting privacy.
