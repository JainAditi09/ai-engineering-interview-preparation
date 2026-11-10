# Security and production reliability — detailed answers

## 1. What is direct prompt injection? What is indirect prompt injection?

**Answer.** Direct prompt injection is malicious instruction text supplied by the user to manipulate model behavior. Indirect prompt injection is malicious instruction text embedded in content the model reads, such as a webpage, email, PDF, code comment, or retrieved document. Both attempt to make untrusted content override intended application behavior.

**Mitigations.** Treat user/retrieved content as data, isolate it from privileged instructions, restrict tools, validate outputs, and enforce authorization outside the model. Delimiters and prompt wording help but are not security boundaries.

**Follow-ups**
- *Can a prompt completely prevent injection?* No. Defense must be layered and enforced in application/tool boundaries.
- *Why is indirect injection dangerous for agents?* An agent can read hostile content and then use tools that have access to private data or external communication.
- *How do you test it?* Add adversarial documents/pages and assert that unauthorized tool calls and data disclosures do not occur.

## 2. Why must retrieved documents and tool outputs be treated as untrusted input?

**Answer.** They may contain incorrect data, malicious instructions, poisoned content, or values controlled by an external party. A tool result is evidence/data, not a new privileged instruction. Parse it according to an expected schema, retain provenance, and do not let its text change tool permissions or system policy.

**Follow-ups**
- *What if the document says “ignore previous instructions”?* Treat that sentence as document content; do not follow it as an instruction.
- *Can a trusted internal database contain unsafe text?* Yes; trust in the data source does not turn arbitrary stored text into privileged instructions.
- *How do you limit impact?* Least privilege, content validation, safe rendering, and constrained tool capabilities.

## 3. Explain the lethal trifecta: private data, untrusted content, and external communication.

**Answer.** The risk becomes severe when an agent simultaneously has access to sensitive private data, consumes untrusted content that can influence it, and has a channel to send data externally. An attacker may place instructions in a document that persuade the agent to retrieve private data and transmit it through email, a browser, or another tool.

**Mitigations.** Break the chain: minimize data access, isolate untrusted content, restrict outbound tools/domains, require approval for sensitive transmission, and audit actions.

**Follow-ups**
- *Do all three need to be present for every attack?* This is a useful risk model, not a universal necessary-and-sufficient formula.
- *How do you break the chain?* Remove unnecessary permissions or constrain the communication destination and data scope.
- *Why is a read-only tool still risky?* Reading private data can enable exfiltration through another capability.

## 4. How do you design least-privilege tool permissions?

**Answer.** Give each tool only the actions and data it needs. Scope credentials by user, tenant, resource, and operation. Separate read from write tools, use short-lived credentials, enforce server-side checks, and require approval for high-impact actions. Do not expose a general-purpose shell or unrestricted database connection to an agent unless isolated and tightly controlled.

**Follow-ups**
- *Why not trust the model's decision to use a tool safely?* Model output is probabilistic and can be influenced by malicious content.
- *How do you revoke access?* Use centralized credential/permission management and check authorization at execution time.
- *What is defense in depth?* Multiple independent safeguards so one failure does not expose the whole system.

## 5. How can an agent exfiltrate data through an otherwise legitimate tool?

**Answer.** A tool may be legitimate in isolation—such as sending email, creating a ticket, or fetching a URL—but dangerous when combined with access to private data and untrusted instructions. The agent could place sensitive content into a message or request to an attacker-controlled endpoint. Restrict destination allowlists, redact sensitive fields, separate data access from outbound communication, and require approval for risky combinations.

**Follow-ups**
- *Is domain allowlisting sufficient?* No; allowed domains can still be abused, and data sensitivity must be considered.
- *How do you detect exfiltration?* Audit outbound payload metadata, DLP checks, anomaly detection, and policy evaluation.
- *Should the model choose recipients?* The application should validate recipient authorization and user intent.

## 6. How do you sandbox code execution?

**Answer.** Run untrusted code in an isolated environment with no host privileges, restricted filesystem, network disabled or allowlisted, CPU/memory/time limits, process limits, and ephemeral storage. Use separate identities and containers/VMs or stronger isolation appropriate to the threat model. Never mount production secrets or host sockets into the sandbox. Destroy the environment after execution.

**Follow-ups**
- *Is a container always a complete security boundary?* No; kernel sharing and configuration matter. Higher-risk workloads may need microVMs or dedicated isolation.
- *Why disable network access?* It limits data exfiltration and remote code execution chains.
- *How do you handle dependencies?* Use prebuilt images or controlled package mirrors rather than arbitrary unrestricted downloads.

## 7. Where should human approval gates be placed?

**Answer.** Place approval before irreversible, high-impact, costly, privacy-sensitive, or externally visible actions—such as issuing refunds, deleting data, sending messages, or changing account permissions. Show the human the exact action, target, amount/data, and consequences. Bind approval to the payload and revalidate preconditions before execution.

**Follow-ups**
- *Can approval be skipped for low-risk actions?* Yes, according to a documented risk policy and user authorization.
- *What if the action changes after approval?* Invalidate approval and request a new one.
- *How do you audit approval?* Record approver identity, timestamp, payload hash, decision, and execution result.

## 8. Why validate model output before performing a side effect?

**Answer.** Model output may be malformed, incomplete, hallucinated, or manipulated. Before a write, parse it into a strict schema, validate business invariants, verify user authorization and resource ownership, and use a transaction/idempotency mechanism. A valid schema alone does not prove the action is allowed.

**Follow-ups**
- *What is the order of checks?* Authenticate, validate schema, authorize resource/action, validate business rules, then execute safely.
- *What if the provider times out after a write?* Query operation status or use an idempotency key before retrying.
- *Why keep side effects out of the model?* The model should propose; deterministic application code enforces and executes.

## 9. How do retries with exponential backoff and jitter work?

**Answer.** Exponential backoff increases the wait between retries, commonly multiplying the delay after each attempt. Jitter adds randomness so many clients do not retry simultaneously. Respect provider Retry-After headers and enforce an overall deadline and retry budget.

**Follow-ups**
- *Which errors are retryable?* Temporary network errors, some 5xx responses, and rate limits; not validation or authorization failures.
- *Why use jitter?* It prevents synchronized retry storms.
- *What is a retry budget?* A limit on retries per request/service/time window to prevent overload amplification.

## 10. When should a request not be retried?

**Answer.** Do not blindly retry deterministic client errors, invalid schema, permission denial, unsupported operations, or business-rule failures. Be especially careful with writes where the original request may have succeeded but the response was lost. Use idempotency keys, operation-status lookup, or reconciliation before retrying uncertain side effects.

**Follow-ups**
- *What is an ambiguous timeout?* The caller does not know whether the operation committed.
- *How do you resolve it?* Query by operation/idempotency ID or reconcile against the source of truth.
- *Can GET requests always be retried?* Usually safer, but rate limits, cost, and semantics still matter.

## 11. What are timeouts and circuit breakers protecting against?

**Answer.** Timeouts bound how long a caller waits for a dependency. Circuit breakers stop sending requests to a dependency that is failing, allowing it time to recover and preventing resource exhaustion. Typical states are closed (normal), open (fail fast), and half-open (allow limited probes).

**Follow-ups**
- *What is the danger of no timeout?* Threads/connections/worker capacity can remain occupied indefinitely.
- *How do you choose a timeout?* Based on latency distributions, product deadlines, and downstream budgets.
- *Can a circuit breaker hide an outage?* It should expose a clear degraded state and emit metrics/alerts.

## 12. How do model fallbacks affect correctness, latency, and cost?

**Answer.** A fallback model can improve availability when the primary provider is unavailable, but models may differ in capabilities, output format, tool calling, safety behavior, and quality. Validate the fallback against the same schema and task evals. Avoid silent fallback for high-impact workflows if behavior is not equivalent.

**Follow-ups**
- *Should every error trigger fallback?* No; invalid inputs or authorization failures are not provider availability issues.
- *How do you keep output consistent?* Use versioned schemas, adapters, validation, and fallback-specific tests.
- *What is the cost trade-off?* A fallback may be cheaper or more expensive; monitor cost and success per provider.

## 13. What makes a tool operation idempotent?

**Answer.** An operation is idempotent if repeating the same request has the same intended effect as performing it once. For example, setting a status to “closed” is naturally idempotent; “increment balance by 10” is not. For non-idempotent operations, use an idempotency key stored with the result and reject or replay duplicate requests consistently.

**Follow-ups**
- *Who generates the idempotency key?* The application/client creates a stable key for one logical operation; the server binds it to the request payload and actor.
- *What if the same key is reused with a different payload?* Reject it.
- *How long should keys be retained?* Long enough to cover retry/replay windows and business requirements.

## 14. How can a long-running agent resume after a process kill?

**Answer.** Persist job state, completed steps, tool outputs, and checkpoint references in durable storage. Run the job through a queue/worker system and use leases/heartbeats to detect abandoned workers. On restart, a new worker claims the job and resumes from a safe checkpoint. External actions need idempotency and reconciliation because a crash can occur between the side effect and state persistence.

**Follow-ups**
- *What is the outbox pattern?* A way to atomically record a database change and an event to publish, reducing lost-event problems.
- *What is a lease?* A time-limited ownership claim that can expire if a worker dies.
- *Can you promise exactly-once execution?* Generally not across distributed systems; design for at-least-once delivery plus idempotent effects.

## 15. How would you test crash recovery and duplicate delivery?

**Answer.** Inject process termination before and after each critical boundary: job claim, model response, tool call, side effect, checkpoint write, and event publication. Redeliver the same queue message and verify the idempotency layer prevents duplicate effects. Assert that state converges to a correct terminal status and that the UI can reconnect to the job.

**Follow-ups**
- *Why test after side effect but before checkpoint?* It is a common ambiguous failure window.
- *How do you simulate duplicate queue messages?* Publish the same job/event ID multiple times.
- *What metrics matter?* Duplicate suppression, recovery time, abandoned jobs, and inconsistent terminal states.

## 16. How do queues and durable execution help long-running agent workflows?

**Answer.** A queue decouples the user request from worker execution, absorbs bursts, supports retries, and lets the API return a job ID quickly. Durable execution persists workflow progress and schedules continuation after waits, approvals, or failures. Design job states, visibility timeouts/leases, idempotent handlers, cancellation, deadlines, and dead-letter handling.

**Follow-ups**
- *What should the API return?* A stable job ID and accepted/queued status, plus a way to query or stream progress.
- *How do you prevent a poison message from retrying forever?* Bound attempts and route persistent failures to a dead-letter queue for inspection.
- *How do you support cancellation?* Persist a cancellation request and check it at safe execution boundaries.
