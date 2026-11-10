# LangGraph and multi-agent systems — detailed answers

## 1. Explain LangGraph state, nodes, edges, conditional routing, and reducers.

**Answer.** A graph-based agent models execution as state transitions. State is the shared data structure passed between steps. Nodes are functions that read state and return updates. Edges define which node runs next; conditional edges choose a route based on state. Reducers specify how updates are combined, especially when multiple nodes update the same state field. This makes execution explicit and inspectable.

**Example.** A support graph may contain classify → fetch order → check policy → approval → execute refund → respond. A conditional edge routes ineligible refunds to an explanation node.

**Follow-ups**
- *Why use reducers?* To define deterministic merge behavior when nodes contribute updates, particularly in parallel branches.
- *What belongs in graph state?* Task ID, validated inputs, relevant evidence, decisions, and status—not secrets or unnecessary raw data.
- *How do you prevent state growth?* Keep compact structured state and store large artifacts externally with references.

## 2. How would you represent a ReAct agent as a graph?

**Answer.** Create an agent node that receives the current messages/state and returns either a final answer or a tool request. A conditional edge routes tool requests to a tool-execution node and final responses to an end node. The tool node validates and executes the requested tool, appends the observation, then routes back to the agent node. Add maximum-step and budget checks to the routing logic.

**Follow-ups**
- *Where do you validate tool arguments?* In the tool execution/backend layer, before side effects.
- *How do you stop loops?* Use a step counter, deadline, repeated-call detection, and terminal conditions in graph state.
- *What is the benefit over a while loop?* Explicit state, conditional transitions, persistence/checkpoint integration, and easier visualization.

## 3. What does checkpointing persist, and how does it support recovery?

**Answer.** A checkpoint stores graph state at a defined execution boundary so a workflow can continue after interruption. Depending on the framework and checkpointer, it may persist state snapshots and execution metadata keyed by thread/session. On restart, the application loads the last checkpoint and resumes from a safe point. External side effects still need idempotency because a crash can occur after a side effect but before its completion is recorded.

**Follow-ups**
- *Does checkpointing guarantee exactly-once tool execution?* No. Use idempotency keys, deduplication, and reconciliation.
- *What should be checkpointed?* Minimal state required to resume, with sensitive values handled securely.
- *How do you test it?* Kill the process at different boundaries, restart, and verify no duplicate side effects or lost state.

## 4. How do human-in-the-loop interrupts work in a risky workflow?

**Answer.** The graph pauses at a defined approval node before an irreversible or sensitive action. It persists the pending state and presents a review request to an authorized human. The application records approve/reject, identity, timestamp, and any changes; then it resumes or follows a rejection path. Approval must be bound to the exact action payload and expire or be invalidated if the payload changes.

**Follow-ups**
- *Why bind approval to a payload hash?* So approval for one amount/order cannot be reused for a modified action.
- *Can the model approve its own action?* No; approval must come from an authorized human or policy engine.
- *What if approval arrives after the underlying data changes?* Revalidate preconditions before execution.

## 5. What are subgraphs, and when are they useful?

**Answer.** A subgraph is a graph used as a component inside a larger graph. It can encapsulate a reusable workflow such as retrieval, document analysis, or approval. Subgraphs improve modularity and can separate state and responsibilities, but interfaces and state mappings must be designed carefully.

**Follow-ups**
- *When not to use a subgraph?* For a trivial one-step operation where abstraction adds more complexity than reuse.
- *How do you pass state?* Define explicit input/output fields and validate the boundary.
- *How do you debug nested graphs?* Trace parent and child execution with shared correlation IDs.

## 6. When would you choose LangGraph versus plain Python?

**Answer.** Plain Python is often best for a simple bounded loop or fixed workflow. A graph framework becomes useful when the process has branching, persistent state, human interrupts, retries, subgraphs, or complex observability requirements. Choose the least complex tool that meets requirements and avoid adopting a framework only because it is popular.

**Follow-ups**
- *What is the framework trade-off?* Faster orchestration and built-in abstractions versus learning curve, framework coupling, and debugging complexity.
- *Can you migrate later?* Yes, if tool interfaces and domain logic are kept independent of orchestration.
- *What should remain outside the graph?* Authorization, transactional business logic, and domain invariants.

## 7. Compare supervisor, hierarchical, handoff/swarm, and debate patterns.

**Answer.** A supervisor delegates tasks to workers and integrates their results. A hierarchical design has multiple levels of coordinators and specialists. Handoff/swarm patterns let an agent transfer control to another specialist, often based on task ownership. Debate runs multiple agents to critique or compare answers. These patterns trade specialization and parallelism for more calls, coordination, and failure surfaces.

**Follow-ups**
- *Which pattern suits a research task?* A supervisor with scoped parallel researchers may work if sources/tasks are independent.
- *Which suits a customer service workflow?* A deterministic workflow with bounded specialist handoffs is often safer than open-ended debate.
- *How do you prevent agent ping-pong?* Explicit ownership, allowed transitions, handoff limits, and terminal conditions.

## 8. Compare shared state with message passing between agents.

**Answer.** Shared state gives agents access to a common data structure, which can simplify coordination but creates coupling and concurrent-write conflicts. Message passing gives each agent explicit inputs/outputs and can improve isolation, auditability, and distributed execution, but requires schemas and message orchestration. Choose based on concurrency, isolation, scale, and traceability needs.

**Follow-ups**
- *How do you avoid race conditions in shared state?* Use reducers, version checks, locks, or single-writer ownership.
- *What makes a good agent message?* Typed task, scope, context references, constraints, expected output, and correlation ID.
- *How do you prevent oversized messages?* Pass references to stored artifacts and only the necessary summary.

## 9. When does multi-agent parallelism justify its additional cost?

**Answer.** It can help when subtasks are independent, require genuinely different expertise or tools, and can be run concurrently. It is not justified merely because a problem is complex. Measure whether parallelism improves quality or wall-clock time enough to compensate for extra tokens, coordination, rate limits, and aggregation errors.

**Follow-ups**
- *How do you establish a baseline?* Run a single-agent or deterministic implementation on the same evaluation set.
- *What if agents agree but are all wrong?* Agreement is not evidence; validate against sources/tests.
- *What metrics matter?* Task success, quality, p95 latency, cost per successful task, and failure/retry rate.

## 10. What coordination errors and debugging problems occur in multi-agent systems?

**Answer.** Common issues include duplicated work, conflicting conclusions, lost context during handoff, stale shared state, deadlocks, loops, partial failures, and inconsistent schemas. Debugging is harder because one user task spans many model and tool calls. Use structured task IDs, explicit ownership, typed messages, per-agent traces, bounded handoffs, and deterministic aggregation.

**Follow-ups**
- *How do you handle one worker failing?* Retry only safe transient failures, use partial results if policy permits, or fail the parent task explicitly.
- *How do you detect duplicated work?* Track normalized task scopes and outputs.
- *How do you make a run reproducible?* Persist model/prompt versions, inputs, tool outputs, and execution configuration.

## 11. How would you build a planner, parallel researcher, writer, and critic?

**Answer.** The planner decomposes the request into independent research questions and assigns scopes. Researchers retrieve evidence from approved sources and return structured findings with citations. The writer synthesizes only supported findings. The critic checks coverage, contradictions, and citation support against explicit criteria. A final validator checks citations and policy; the graph ends when quality criteria pass or the bounded revision budget is exhausted.

**Follow-ups**
- *How do you prevent researchers from duplicating each other?* Give each a distinct question/source scope and inspect overlap.
- *Should the critic rewrite everything?* No; it should return specific defects so revisions are targeted.
- *How do you handle conflicting sources?* Preserve provenance and surface conflicts rather than flattening them.

## 12. How would you compare a multi-agent implementation against a single-agent baseline?

**Answer.** Use the same task dataset, source corpus, tool permissions, and success rubric. Compare task completion and quality, citation correctness, tool errors, p50/p95 latency, token usage, cost per successful task, and human review time. Run enough trials to account for model variability and report confidence/variance, not just one impressive example.

**Follow-ups**
- *Why compare cost per successful task?* A more expensive agent may be worthwhile if it materially improves success, but cost per request hides failures.
- *How do you avoid unfair comparison?* Match budgets, tools, and available evidence.
- *What if quality improves but latency worsens?* Decide based on product SLA and user value; consider parallelism or selective escalation.
