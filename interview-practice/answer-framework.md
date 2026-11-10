# Answer framework for AI engineering interviews

Use this structure for most technical questions:

1. **Direct answer:** define the concept in 1–2 sentences.
2. **Request/data flow:** explain what happens step by step.
3. **Engineering choice:** explain why you chose this approach.
4. **Trade-offs:** mention cost, latency, quality, complexity, and security where relevant.
5. **Failure handling:** describe timeouts, malformed output, retries, fallbacks, or human approval.
6. **Measurement:** name the metric and how you would evaluate it.
7. **Project evidence:** connect to code or a measured result you actually produced.

For system design, cover:
- Requirements and assumptions.
- High-level architecture and data flow.
- APIs, state, storage, queues, and tenancy.
- Tool permissions and human approval.
- Memory and retrieval.
- Evals and observability.
- Reliability, security, cost, and latency.
- Bottlenecks and rollout strategy.

Do not invent metrics. If you have not measured a result, say how you would measure it.
