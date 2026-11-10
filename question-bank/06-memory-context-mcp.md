# Memory, context engineering, and MCP — detailed answers

## 1. Differentiate short-term and long-term memory.

**Answer.** Short-term memory is information needed within the current interaction or task, such as recent turns, current plan, and temporary tool observations. Long-term memory persists across sessions, such as stable preferences or durable project facts. Both are application-managed: the model only sees information the application selects and places in context.

**Follow-ups**
- *Should every conversation be stored forever?* No. Apply retention, user control, privacy, and relevance policies.
- *How do you expire short-term memory?* Use session/task boundaries and TTLs where appropriate.
- *How do you prevent long-term memory from becoming stale?* Store provenance and timestamps, allow correction, and revalidate facts before important actions.

## 2. Differentiate episodic and semantic memory.

**Answer.** Episodic memory records specific events or experiences—for example, a task was attempted and failed on a date. Semantic memory stores generalized facts or knowledge, such as a user's preferred output format or a product's documented behavior. A system may derive semantic summaries from episodes, but should preserve source and confidence rather than treating every inferred pattern as confirmed truth.

**Follow-ups**
- *Which memory is useful for debugging?* Episodic traces preserve the sequence of actions and observations.
- *Which memory is useful for personalization?* Relevant, user-confirmed semantic preferences.
- *Can semantic memory lose nuance?* Yes; summaries can overgeneralize, so preserve provenance and allow correction.

## 3. What information should an agent remember across sessions, and what should it not store?

**Answer.** Store only information that is useful for future tasks, permitted by the product's privacy policy, and appropriate for the user's expectations. Examples may include explicit project decisions or stable formatting preferences. Avoid unnecessary sensitive data, credentials, raw secrets, and speculative inferences. Provide retention and deletion mechanisms and respect user corrections.

**Follow-ups**
- *Should the agent store API keys?* No; use a secrets manager and scoped credentials.
- *How do you decide relevance?* Consider expected future utility, confidence, freshness, sensitivity, and user consent.
- *What if the user corrects a memory?* Update the canonical fact, preserve audit history only where policy requires, and stop using the incorrect value.

## 4. How do you summarize or trim conversation history without losing important constraints?

**Answer.** Use a structured summary with fields such as goal, confirmed facts, decisions, explicit constraints, unresolved questions, and references to artifacts. Keep the most recent turns verbatim when they contain active details. Token-budget the summary and retrieve supporting history when needed. Validate summaries by asking whether a downstream task can still be completed correctly.

**Follow-ups**
- *What is dangerous to summarize away?* Authorization constraints, user corrections, exact identifiers, deadlines, and unresolved commitments.
- *Should summaries be model-generated?* They can be, but validate critical fields and preserve original source references.
- *How do you test summary quality?* Run continuation tasks against full history and compressed history and compare success.

## 5. What is context engineering? How do you decide what enters the context and in what order?

**Answer.** Context engineering is the deliberate selection, organization, and budgeting of information supplied to a model for a particular task. It includes system instructions, user input, relevant history, retrieved evidence, tool schemas/results, and output requirements. Prioritize trusted instructions and constraints, then task-specific facts/evidence; remove irrelevant and duplicate content. Exact ordering can be model-dependent, so evaluate with representative tasks.

**Follow-ups**
- *Is more context always better?* No; noise, cost, and attention dilution can harm results.
- *How do you defend against prompt injection in context?* Treat retrieved/user-provided text as untrusted data, isolate it from instructions, and enforce tools/permissions in code.
- *How do you measure context quality?* Task success, groundedness, token use, and failure rate under context ablations.

## 6. What is prompt caching, and what workload patterns benefit from it?

**Answer.** Prompt caching reuses computation for repeated prompt prefixes or provider-supported cached content. It can reduce time and/or cost when requests share a stable prefix, such as system instructions, tool definitions, or a large static reference. It is less useful when every request has a unique prefix or cache eligibility/TTL is low.

**Follow-ups**
- *What should be placed in a stable prefix?* Repeated instructions and schemas, not user-specific secrets or volatile data.
- *Can cache behavior be assumed?* No; provider rules, minimum token sizes, TTLs, and pricing vary. Measure actual cache hits.
- *What is a cache invalidation concern?* Prompt or policy updates must not continue using stale cached content beyond the intended validity.

## 7. How would you store and retrieve user memory in Postgres?

**Answer.** A simple design uses a memory table with memory ID, tenant/user ID, type, content or structured payload, source/provenance, confidence, created/updated timestamps, expiry, and status. Index by user/tenant and relevant metadata. Read only memories authorized for the authenticated user, rank by relevance and freshness, and insert selected memories into context under a token budget. For semantic retrieval, add vector embeddings while retaining relational access controls.

**Follow-ups**
- *How do you prevent cross-user leakage?* Enforce tenant/user predicates in server-side queries and test isolation.
- *Should memory updates overwrite old records?* Depends on audit requirements; maintain version/history where correction traceability matters.
- *How do you handle TTL?* Use explicit expiry and scheduled cleanup/partitioning according to retention policy.

## 8. How do you handle stale, contradictory, or user-corrected memories?

**Answer.** Store provenance, timestamps, confidence, and validity. Prefer recent explicit corrections over older inferred values. When two facts conflict and neither is clearly authoritative, ask the user rather than silently choosing. Mark superseded memories and ensure retrieval does not return both as current facts.

**Follow-ups**
- *Should recency always win?* No; a newer weak inference should not override an older verified source.
- *How do you represent uncertainty?* Store source type, confidence, and status such as confirmed, inferred, or superseded.
- *What should happen after correction?* Update the canonical memory and test that future retrieval no longer surfaces the old value as current.

## 9. What are the host, client, and server in MCP?

**Answer.** In the Model Context Protocol, the host is the AI application coordinating the interaction; an MCP client inside the host maintains a connection to an MCP server; the server exposes capabilities such as tools, resources, or prompts. This provides a standardized way for AI applications to discover and invoke external capabilities. The host remains responsible for user experience, policy, and permission decisions.

**Follow-ups**
- *Can a host connect to multiple servers?* Yes, subject to configuration, permissions, and resource limits.
- *Does an MCP server automatically get access to all user data?* No; access should be scoped to configured credentials and explicit capabilities.
- *Is MCP an agent framework?* No; it standardizes context/capability exchange, not the full planning loop.

## 10. Explain MCP tools, resources, prompts, and transports.

**Answer.** Tools are callable operations; resources expose data/context that a client can read; prompts provide reusable prompt templates. Transports define how messages move between client and server, such as local process communication or network-based transport. Exact protocol versions and transport details evolve, so check the relevant specification and SDK version when implementing.

**Follow-ups**
- *When is a resource better than a tool?* When the client needs to read/reference data rather than invoke an operation.
- *Can tools have side effects?* Yes; therefore permission checks, confirmation, and audit logging are important.
- *Does transport provide authorization by itself?* No; authentication and authorization must be designed for the deployment.

## 11. How would you expose a RAG search tool and database tool through an MCP server?

**Answer.** Define narrow tools with typed schemas, such as `search_knowledge(query, limit)` and `get_order_status(order_id)`. The server authenticates the client, derives tenant scope from trusted credentials, validates inputs, calls internal services, and returns structured results with provenance. Do not expose arbitrary SQL or unbounded database access. Log calls and enforce timeouts/rate limits.

**Follow-ups**
- *Should the model supply tenant_id?* No; derive it from authenticated context.
- *How do you prevent SQL injection?* Use parameterized queries and fixed query templates, not model-generated SQL concatenation.
- *How do you protect sensitive results?* Filter by permissions before returning content and minimize returned fields.

## 12. How is MCP different from a tool-calling API or an agent framework?

**Answer.** A model tool-calling API defines how a model can request tool use within a provider's interaction format. MCP standardizes how applications connect to servers that expose tools/resources/prompts. An agent framework provides orchestration abstractions such as state, loops, routing, retries, or graph execution. These can be combined: an agent framework can call tools exposed through MCP using a model's tool-calling capability.

**Follow-ups**
- *Can you use MCP without an LLM?* Yes; it is a protocol for capability/context exchange.
- *Can you build an agent without MCP?* Yes; tools can be integrated directly.
- *Why use MCP?* Reuse and interoperability across compatible clients and servers, balanced against protocol and operational overhead.

## 13. What is A2A intended to enable, at a high level?

**Answer.** Agent-to-agent (A2A) protocols aim to let independently built agents communicate and delegate work across organizational or technical boundaries. The key idea is interoperability: discover another agent's capabilities, exchange task requests and status, and receive results without requiring both agents to share the same internal framework. Security, identity, trust, and task authorization remain essential.

**Follow-ups**
- *How is A2A different from MCP?* MCP focuses on an application connecting to tools/context servers; A2A focuses on agent-to-agent collaboration.
- *Does interoperability mean an agent should trust another agent's output?* No; validate identity, provenance, permissions, and returned data.
- *When is A2A unnecessary?* When one application can call a well-defined internal service directly.
