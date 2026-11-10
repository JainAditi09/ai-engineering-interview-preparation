# Project 1 — From-Scratch Agent

**Source roadmap:** Week 1, Days 1–6.

## Objective
Build a working tool-calling ReAct agent in plain Python before adopting a framework.

## Required capabilities
- CLI chat interface, streaming, conversation history, token/cost tracking.
- Structured tool schemas and argument validation.
- Calculator, web search, file read/write, and sandboxed Python execution.
- Observe–reason–act–observe loop with max steps, token budget, and loop detection.
- Planner and critic experiment across 10 tasks.
- Failure injection for exceptions, junk output, and timeouts.

## Interview walkthrough
1. State the problem and why an agent is justified.
2. Draw the request → model → tool → observation loop.
3. Explain validation, permission boundaries, stopping conditions, and error handling.
4. Show a failed trace and how the system recovers.
5. Share measured quality, latency, and cost.

## Evidence to publish
- Architecture diagram, tests, sample traces, README, and `What broke and why`.
