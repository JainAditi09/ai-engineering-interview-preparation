# Prompting and structured outputs — detailed answers

## 1. What makes a prompt clear, testable, and robust?

**Answer.** A robust prompt states the task, relevant context, constraints, output contract, and what to do when information is missing. Separate instructions from untrusted input with clear delimiters. Define ambiguous terms, give examples for edge cases, and avoid conflicting requirements. A prompt is testable when expected behavior can be checked against a representative set of inputs.

**Example.** For an invoice extractor, specify fields, types, currency/date rules, null behavior, and prohibit guessing missing values. Then validate the output against a schema.

**Follow-ups**
- *Should prompts contain every business rule?* No. Put deterministic rules, authorization, and calculations in code; use prompts for interpretation and language tasks.
- *How do you version prompts?* Store templates with version IDs, commit changes, and run regression evals before rollout.
- *How do you handle conflicting instructions?* Establish a clear instruction hierarchy and reject or clarify ambiguous task requirements.

## 2. When would you use few-shot examples instead of a longer instruction?

**Answer.** Few-shot examples are useful when the desired pattern is easier to demonstrate than to describe—for example, classification boundaries, tone, or a complex output format. A long instruction is better for explicit rules and constraints. Examples should be representative, diverse, and include difficult boundary cases; otherwise the model may overfit to superficial patterns.

**Follow-ups**
- *How many examples should you include?* There is no universal number. Add only examples that improve held-out evaluation enough to justify context cost.
- *What is a bad few-shot example?* One that is ambiguous, inconsistent with the schema, or demonstrates an accidental behavior.
- *How do you select examples?* Cover common cases, edge cases, and important classes; evaluate on examples not used in the prompt.

## 3. When should you use JSON, JSON Schema, or XML-style delimiters?

**Answer.** Use JSON when downstream software needs structured data. JSON Schema describes allowed fields, types, required properties, and constraints. XML-style tags or delimiters can make sections of a prompt easier for the model to distinguish, such as instructions, context, and user-provided documents. Delimiters do not create a security boundary; content inside them remains untrusted.

**Follow-ups**
- *Does asking for JSON guarantee valid JSON?* No. Use provider-supported constrained output when available and validate server-side.
- *Why use a schema?* It makes the contract explicit and allows deterministic validation.
- *Are XML tags safer than plain text?* They improve structure but do not prevent prompt injection.

## 4. How do you validate model output with Pydantic?

**Answer.** Define a Pydantic model that represents the application contract, including required fields, types, enums, bounds, and custom validators. Parse the raw model output into that model. If parsing fails, capture structured validation errors and either retry with a bounded repair prompt or return a controlled failure. Treat validation as a gate before downstream side effects.

**Example.** A support intent model might require `intent: Literal["refund","delivery","other"]`, an optional order ID, and a confidence value constrained to [0,1]. The backend must still verify the order and permissions.

**Follow-ups**
- *What is the difference between parsing and validation?* Parsing converts data into a representation; validation checks whether it satisfies the declared contract and constraints.
- *Should you coerce invalid values automatically?* Only when coercion is explicitly safe and documented; otherwise reject.
- *Can schema validation prove the answer is true?* No. It proves shape/type constraints, not factual correctness.

## 5. What should happen when generated JSON is malformed?

**Answer.** First capture the failure category: truncated response, syntax error, missing required field, invalid enum, or semantic inconsistency. If the output may be incomplete due to a token limit or interrupted stream, do not attempt to repair it as if it were complete. For recoverable cases, perform a small, bounded repair attempt using the schema and validation errors. If it still fails, return a typed error or ask the user for clarification.

**Follow-ups**
- *Why not retry indefinitely?* It creates cost and latency loops and can repeat the same failure.
- *Should the original response be retained?* Retain a redacted trace for debugging where privacy policy permits.
- *What if a field is missing but optional?* Apply the schema's documented default/null behavior rather than inventing a value.

## 6. How would you design a bounded retry/repair loop for invalid output?

**Answer.** Use a finite retry budget (often one or two repairs), a timeout, and a maximum token/cost budget. Send the original task, the expected schema, and concise validation errors—not an unbounded transcript. Validate each repaired result again. Distinguish transient provider failures from content/schema failures and use different retry policies. After the budget is exhausted, fail safely.

**Follow-ups**
- *Which errors are retryable?* Transient rate limits, network errors, and some malformed outputs; not authorization failures or deterministic business-rule violations.
- *How do you avoid retry storms?* Exponential backoff with jitter, per-request budgets, concurrency limits, and circuit breakers.
- *How do you measure repair quality?* Track first-pass validity, repair success, semantic accuracy, added latency, and cost.

## 7. How do you prevent a repair prompt from silently changing the user's intent?

**Answer.** Keep the original task immutable and ask the model only to repair formatting or satisfy explicit schema constraints. Do not allow the repair step to add facts. Re-run semantic checks against the original task and evidence. If required information is absent, represent it as unknown/null or request clarification rather than fabricating it.

**Follow-ups**
- *Can a model self-check its own output?* It can help detect issues but is not an independent correctness guarantee.
- *What should happen when the schema conflicts with the task?* Treat it as an application contract defect; revise the schema or clarify the task.
- *How do you audit changes?* Log the original and repaired versions with redaction and compare changed fields.

## 8. How would you version prompts and test prompt changes?

**Answer.** Store prompts as versioned artifacts with owner, purpose, model assumptions, schema version, and change notes. Build a golden dataset of representative and adversarial inputs. Run regression evaluations for correctness, formatting, safety, tool selection, latency, and cost. Deploy gradually with a rollback path and compare outcomes by version.

**Follow-ups**
- *Why can a prompt change break unrelated cases?* It can alter attention to constraints, examples, or output patterns; evaluate the whole behavior surface.
- *What is an offline eval vs online experiment?* Offline eval uses a fixed dataset; online tests measure real user outcomes under controlled rollout.
- *Should prompt and schema versions be coupled?* Track them separately but record compatibility explicitly.

## 9. How do you handle missing, extra, or incorrectly typed fields?

**Answer.** Define required and optional fields clearly. Reject unknown fields when strictness is important, or explicitly allow them for forward compatibility. Apply defaults only when they have a sound semantic meaning. Type coercion should be conservative; a string that looks like a number is not always safe to convert. Return validation errors in a form the caller can act on.

**Follow-ups**
- *When should unknown fields be rejected?* When extra output could cause security or data-integrity problems, especially in commands or side effects.
- *How do you evolve a schema?* Version it, add backward-compatible optional fields where possible, and test old/new clients.
- *What if a required field cannot be inferred?* Return a validation failure or ask a clarifying question.

## 10. Why is syntactically valid JSON not necessarily semantically correct?

**Answer.** JSON syntax only ensures that the response is parseable. A field may contain a valid but incorrect value, inconsistent fields, unsupported claims, or an action the user did not authorize. Semantic validation checks relationships and business invariants; factual validation checks claims against trusted evidence; authorization checks whether the requested action is allowed.

**Follow-ups**
- *Give an example.* `{"refund_amount": 100000}` can be valid JSON but exceed the order's refundable balance.
- *Where should business invariants be enforced?* In deterministic backend services and database transactions.
- *How do you test semantic correctness?* Use domain-specific assertions, golden cases, adversarial inputs, and human review for ambiguous cases.
