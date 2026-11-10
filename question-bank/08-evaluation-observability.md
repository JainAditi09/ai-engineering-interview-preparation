# Evaluation and observability — detailed answers

## 1. What is a golden dataset, and how do you maintain one?

**Answer.** A golden dataset is a curated set of representative inputs with expected outcomes, labels, or evaluation rubrics. It acts as a stable regression suite for prompts, models, retrieval, and agent changes. Include normal, boundary, ambiguous, adversarial, and no-answer cases. Version it, document label provenance, review disagreements, and add production failures after removing sensitive data.

**Follow-ups**
- *Can expected answers be multiple?* Yes. Use acceptable answer sets or rubrics when tasks allow multiple valid responses.
- *How do you avoid overfitting?* Maintain a held-out set and rotate/add cases.
- *Who should label it?* Domain experts or trained reviewers; measure inter-annotator agreement for subjective tasks.

## 2. How do you test a non-deterministic agent?

**Answer.** Combine deterministic unit tests for application logic with repeated model-level evaluations. Mock tool/model boundaries to test state transitions, retries, permissions, and error paths. For end-to-end behavior, run a fixed dataset multiple times under controlled model/prompt versions and evaluate distributions rather than one result. Use human review for nuanced quality and safety.

**Follow-ups**
- *Why are unit tests still valuable?* They catch deterministic bugs without relying on model consistency.
- *How many repeated runs are enough?* It depends on variance and risk; use statistical reasoning and confidence intervals rather than a universal count.
- *How do you keep CI stable?* Keep deterministic tests in required CI and run costly/stochastic evals in scheduled or gated pipelines.

## 3. What is LLM-as-judge? What biases can affect it?

**Answer.** LLM-as-judge uses a model to score or compare generated answers against a rubric. It scales evaluation but can be biased by wording, answer order, verbosity, familiar styles, and the judge's own model preferences. A judge may reward confident language rather than factual support. Calibrate it against human labels and use it as one signal, not absolute truth.

**Follow-ups**
- *How do you reduce position bias?* Randomize or swap answer order and aggregate pairwise judgments.
- *How do you reduce verbosity bias?* Use a rubric that explicitly rewards correctness and concision, not length.
- *Can the same model judge itself?* It can, but self-preference and correlated errors require independent checks.

## 4. How can position, verbosity, and self-preference bias a judge model?

**Answer.** Position bias occurs when the judge favors the first or second candidate regardless of quality. Verbosity bias rewards longer answers even when they add no correct information. Self-preference bias occurs when a judge favors answers that resemble its own style or model family. Detect these with controlled experiments: swap candidate order, compare concise/verbose variants with equal facts, and compare judges from different models/humans.

**Follow-ups**
- *What is a pairwise swap test?* Evaluate A vs B and then B vs A; large changes suggest order sensitivity.
- *Why use a rubric?* It makes the criteria explicit and reduces reliance on stylistic preference.
- *What if judges disagree?* Escalate to human review or report uncertainty rather than force a false consensus.

## 5. How do you reduce evaluator bias with pairwise tests, rubrics, or human review?

**Answer.** Define criteria before scoring, use anchored score descriptions, randomize answer order, run pairwise comparisons in both directions, and sample cases for human review. Track agreement and disagreement by task type. Keep judge prompts and model versions fixed during comparisons. For critical decisions, use deterministic tests or domain experts.

**Follow-ups**
- *What is inter-rater agreement?* A measure of how consistently evaluators apply labels or scores.
- *Why blind reviewers?* It reduces influence from model identity, experiment condition, or author expectations.
- *Should humans review every response?* Usually not scalable; sample based on risk and uncertainty.

## 6. What is trajectory evaluation, and why judge the path as well as the final answer?

**Answer.** Trajectory evaluation assesses the sequence of decisions, tool calls, observations, and state transitions—not only the final response. An agent may reach a correct answer by leaking data, calling unauthorized tools, or making unnecessary expensive calls. Evaluate whether the path respected permissions, used evidence, avoided duplicate side effects, and stayed within budgets.

**Follow-ups**
- *Can a correct final answer still be a failure?* Yes, if the process violated policy or performed unsafe actions.
- *What should a trajectory rubric include?* Tool choice, argument correctness, authorization, evidence use, recovery, and termination.
- *How do you evaluate hidden reasoning?* Evaluate observable actions and evidence, not private chain-of-thought.

## 7. How do you detect regressions after a prompt or model change?

**Answer.** Run the same versioned evaluation set before and after the change. Compare task success, groundedness, schema validity, tool selection, safety violations, latency, and cost. Segment results by task category. Use thresholds and a canary rollout; if metrics regress, roll back to a known-good prompt/model configuration.

**Follow-ups**
- *Why can aggregate scores hide regressions?* Gains in common cases can mask severe failures in rare/high-risk cases.
- *What is a canary?* A small rollout to a subset of traffic before wider deployment.
- *How do you handle noisy metrics?* Repeat runs, confidence intervals, and targeted review of changed cases.

## 8. What should an eval suite measure: correctness, groundedness, tool use, safety, or something else?

**Answer.** It should reflect the product's failure costs. Common dimensions include task success, factual correctness, evidence support, citation accuracy, schema validity, tool selection, argument validity, policy compliance, latency, and cost. For side-effecting agents, include duplicate actions, authorization, and recovery. Avoid one composite score that hides critical safety failures.

**Follow-ups**
- *Can safety be averaged with quality?* High-severity safety violations should often be hard gates, not offset by better average quality.
- *How do you measure user satisfaction?* Combine feedback with behavioral metrics while accounting for selection bias.
- *How do you choose thresholds?* Set them from product risk and baseline distributions, then review false positives/negatives.

## 9. What are the limitations of benchmarks such as SWE-bench, GAIA, and tau-bench?

**Answer.** Benchmarks provide standardized tasks but represent only a slice of real-world usage. Scores depend on benchmark version, environment, agent scaffolding, tools, model, and evaluation protocol. They may suffer from contamination, narrow task coverage, or mismatch with a product's users and constraints. Use them for comparison and research, not as a substitute for product-specific evals.

**Follow-ups**
- *Why report benchmark configuration?* Without it, scores are not reproducible or fairly comparable.
- *What is benchmark contamination?* Test data may have appeared in model training or tuning.
- *What should complement public benchmarks?* A private representative evaluation set and production monitoring.

## 10. What should you trace in Langfuse or LangSmith?

**Answer.** Trace the end-to-end request and child spans for retrieval, model calls, tool execution, retries, graph nodes, and external services. Capture timestamps, model/prompt versions, token usage, cost, status, tool name, redacted arguments, and correlation IDs. Store enough information to debug without logging secrets or unnecessary personal content.

**Follow-ups**
- *What is a span?* A timed unit of work within a trace.
- *How do you trace async work?* Propagate trace context and correlation IDs through queues and workers.
- *Why redact?* Traces can contain private data, credentials, and business-sensitive information.

## 11. How do cost, latency, token usage, tool errors, and success rate fit into observability?

**Answer.** They form a view of operational and product health. Track cost per request and successful task, p50/p95 latency, time-to-first-token, input/output/cached tokens, retrieval duration, tool error and retry rates, and completion/safety rates. Segment by feature, model, tenant, and version. Alerts should identify actionable deviations rather than every normal fluctuation.

**Follow-ups**
- *Why p95 instead of only average latency?* Tail latency captures slow experiences that averages hide.
- *What is a useful SLO?* A measurable target such as a percentage of eligible tasks completing within a latency/error threshold.
- *How do you avoid cardinality explosions?* Do not use raw user IDs or unbounded values as metric labels.

## 12. Where can OpenTelemetry fit into an AI application?

**Answer.** OpenTelemetry provides vendor-neutral APIs and data formats for traces, metrics, and logs. Instrument frontend/backend requests, queues, retrieval, model-provider calls, and tools, then export to an observability backend. AI-specific details such as prompts and token counts may need custom attributes or conventions; handle sensitive data carefully.

**Follow-ups**
- *Why use vendor-neutral instrumentation?* It reduces lock-in and supports multiple backends.
- *What is distributed tracing?* Following one request across services using trace and span context.
- *Should prompt text be an attribute?* Only if policy allows; often store redacted or sampled payloads separately.

## 13. How would you run 50+ eval cases in CI without making builds unreliable or prohibitively slow?

**Answer.** Split tests into layers. Run fast deterministic unit/schema/permission tests on every commit. Run a small stable model eval on pull requests with fixed fixtures and budgets. Run the full 50+ stochastic suite nightly or before release, with controlled model versions, rate limits, retries, and result storage. Gate releases on critical failures and statistically meaningful regressions, not a single noisy score.

**Follow-ups**
- *What should never be flaky?* Deterministic permission, schema, state-transition, and idempotency tests.
- *How do you manage provider rate limits?* Concurrency caps, caching where valid, retry-after handling, and test budgets.
- *How do you compare eval runs?* Store dataset version, prompt/model version, environment, seeds where supported, metrics, and case-level results.
