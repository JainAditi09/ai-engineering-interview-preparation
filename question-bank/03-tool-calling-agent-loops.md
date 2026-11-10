# Tool calling and agent loops — detailed answers

## 1. Walk through a tool call from model request to application execution and tool result.

**Answer.** The application sends the conversation and available tool schemas to the model. The model may return a normal message or a structured tool-call request. The application—not the model—validates the tool name and arguments, checks the caller's identity and permissions, executes the tool, and captures its result. It appends the tool result to the conversation and calls the model again so it can interpret the observation or request another tool. The backend owns authorization and side effects.

**Follow-ups**
- *Does the model execute the tool?* No. It proposes a call; application code executes it.
- *What if the model requests an unknown tool?* Reject it, log a safe error, and do not execute arbitrary code.
- *Why send the tool result back to the model?* The result is new evidence needed to decide the next step or formulate the response.

## 2. What belongs in a tool schema, and how should required fields be represented?

**Answer.** A schema should describe the tool's purpose, parameter names, types, meanings, constraints, enums, and required fields. Keep tools narrow and unambiguous. Avoid exposing raw infrastructure capabilities when a domain-specific operation is safer. Include units and timezone semantics for dates, money, and quantities.

**Follow-ups**
- *Why are narrow tools safer?* They limit the actions the model can request and make permission checks easier.
- *Should a tool schema include authorization rules?* It can describe intended use, but enforcement must be implemented in backend code.
- *What makes a tool description poor?* Ambiguous names, overlapping responsibilities, missing units, and unclear side effects.

## 3. How does a model decide which tool to call?

**Answer.** The model uses the conversation, instructions, and tool descriptions/schema to predict whether a tool is useful and which arguments to provide. This is probabilistic behavior, not deterministic planning. The application should constrain the available tools to the current task and validate every proposed call.

**Follow-ups**
- *Can you force a particular tool?* Some APIs support tool-choice controls; still validate the request.
- *How do you improve tool selection?* Clear descriptions, fewer overlapping tools, examples, and evaluation of tool-choice accuracy.
- *What if two tools seem suitable?* Make their responsibilities distinct or use a deterministic router for the decision.

## 4. Why are tool descriptions important to agent quality?

**Answer.** Tool descriptions are part of the model's decision context. They explain what a tool does, when to use it, required inputs, side effects, and limitations. Poor descriptions lead to incorrect tool selection or arguments. Descriptions should be concise, specific, and aligned with actual backend behavior.

**Follow-ups**
- *Can descriptions replace code validation?* No.
- *How do you test descriptions?* Create tasks with expected tool choices and measure selection accuracy, argument validity, and unnecessary calls.
- *Should descriptions expose implementation details?* Only details that help safe selection; avoid confusing internal details.

## 5. How do you validate tool arguments before execution?

**Answer.** Parse arguments against a strict schema, reject unknown or missing fields, normalize safe formats, and apply domain validation. Then check identity, tenant scope, permissions, resource ownership, rate limits, and business rules. For writes, use idempotency/deduplication and transactional checks. Never trust the model-supplied user ID or authorization claims.

**Follow-ups**
- *Why validate ownership separately from schema?* A well-typed order ID can still belong to another user.
- *Where should tenant ID come from?* From authenticated server-side context, not model output.
- *What should happen on validation failure?* Return a typed error and avoid side effects.

## 6. How should an application handle unknown tools, malformed arguments, timeouts, and exceptions?

**Answer.** Treat each failure as a typed tool result: unknown tool, invalid arguments, permission denied, timeout, rate limit, or internal error. Do not expose secrets or stack traces to the model. Retry only transient failures under a bounded policy. A timeout may leave an operation's outcome unknown, so query the operation status or use an idempotency key before retrying a write.

**Follow-ups**
- *Why is retrying a timed-out refund dangerous?* The refund may have succeeded while the response was lost, causing a duplicate.
- *How should errors be presented to the model?* Concise, actionable, and sanitized.
- *What should be logged?* Tool name/version, correlation ID, duration, result class, retry count, and redacted metadata.

## 7. When can tool calls run in parallel, and when must they be sequential?

**Answer.** Parallelize independent, read-only calls when results do not depend on one another—for example, fetching weather and calendar availability. Keep dependent operations sequential: create a booking only after availability is checked; issue a refund only after order and policy validation. For concurrent writes, use locking, optimistic concurrency, or transactions.

**Follow-ups**
- *What is the main parallelism risk?* Race conditions, rate limits, nondeterministic ordering, and conflicting side effects.
- *How do you bound parallel calls?* Concurrency limits, per-tool rate limits, deadlines, and cancellation.
- *Can the model request parallel calls?* It may; the application decides whether and how to execute them.

## 8. Explain the observe–reason–act–observe loop.

**Answer.** The agent receives a task and current state (observe), selects a next step (reason/decide), performs a tool or action (act), and receives the result (observe). It repeats until a completion condition or safety limit is reached. Each cycle should update a traceable state and have explicit budgets for steps, tokens, time, and tool calls.

**Follow-ups**
- *Why can the loop fail?* Wrong assumptions, bad tools, poor observations, repeated actions, or missing stopping conditions.
- *How do you make it debuggable?* Persist step-level inputs/outputs, tool calls, timestamps, and terminal reason with sensitive data redacted.
- *When should the loop stop?* When task success is verified, the budget is exhausted, the user cancels, or a safety/permission boundary is reached.

## 9. What is ReAct, and how does it differ from a fixed workflow?

**Answer.** ReAct is a pattern in which a model alternates reasoning/decision steps with actions and observations. A fixed workflow follows predefined transitions and is more predictable. ReAct is useful when the next step depends on information discovered during execution; fixed workflows are better when the process is known and compliance requires deterministic paths.

**Follow-ups**
- *Is ReAct always better?* No; it adds latency, cost, and unpredictability.
- *Can a ReAct agent be constrained?* Yes—limit tools, steps, permissions, and allowed transitions.
- *What is a hybrid design?* Use deterministic workflow stages with agent decisions only at bounded points.

## 10. How do you define stopping conditions, maximum steps, and token budgets?

**Answer.** Define success as a verifiable application condition, not merely the model saying “done.” Add maximum iterations, wall-clock deadline, token/cost budget, per-tool timeout, and cancellation. Stop on successful completion, user cancellation, policy denial, repeated-state detection, or exhausted budget. Return a meaningful terminal status.

**Follow-ups**
- *What if the task is incomplete at the budget limit?* Return partial progress and state clearly what remains.
- *How do you detect loops?* Compare normalized recent states/actions, detect repeated tool arguments, and track lack of progress.
- *Should the model choose its own budget?* It may suggest complexity, but the server enforces limits.

## 11. How would you detect repeated tool calls or an infinite loop?

**Answer.** Track tool name plus normalized arguments, state hashes, recent observations, and progress signals. Detect exact repeated calls, cycles across multiple tools, and repeated failures. Apply per-tool call limits and an overall step budget. Some legitimate tasks repeat similar calls, so use context-aware thresholds and allow explicit exceptions.

**Follow-ups**
- *Can repeated calls be legitimate?* Yes, polling or pagination may repeat a tool with a changing cursor or state.
- *How do you avoid false positives?* Include relevant cursor/state/version fields in the call fingerprint.
- *What happens when a loop is detected?* Stop safely, preserve trace, and explain the incomplete state.

## 12. What should be logged for each tool invocation?

**Answer.** Log correlation/job ID, authenticated actor and tenant identifiers (appropriately protected), tool name/version, validated argument summary, permission decision, start/end time, duration, result category, retry count, and whether a side effect was committed. Redact secrets, personal data, and unnecessary payloads. Keep audit logs for sensitive actions.

**Follow-ups**
- *Should you log full prompts and tool results?* Only when justified by privacy/security policy; prefer redacted or sampled traces.
- *How do logs differ from traces?* Logs are discrete events; traces connect spans across a request and its downstream calls.
- *How do you debug a missing tool result?* Follow correlation IDs through the model call, tool execution, queue, and persistence layers.
