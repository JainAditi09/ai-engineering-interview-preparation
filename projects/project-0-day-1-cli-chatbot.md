# Project 0 — Day 1: Gemini Python CLI Chatbot

**Roadmap mapping:** Week 1, Day 1 — How LLMs work.

## Implementation

[**ai-cli-chatbot — Python + Google Gemini**](https://github.com/JainAditi09/ai-cli-chatbot)

A hands-on terminal chatbot built with Python and Google's official GenAI SDK. It demonstrates the application-side mechanics of making multi-turn LLM requests and streaming responses.

## Features implemented

- Gemini API integration with API key and model configuration loaded from environment variables.
- Streaming generation with `send_message_stream`, rendering response chunks as they arrive.
- Multi-turn conversation history managed through a Gemini chat session.
- Token usage extraction from response usage metadata.
- Per-request and cumulative session cost estimation using configurable input/output token rates.
- CLI commands: `/history`, `/reset`, `/usage`, and `/exit`.
- Environment-variable handling and `.gitignore` protection for credentials.

## Concepts demonstrated

| Day 1 concept | What the project demonstrates |
|---|---|
| System prompt | Configures the assistant's behaviour for a chat session. |
| User / model turns | Sends user messages and maintains multi-turn conversation context. |
| Streaming | Receives and displays generated text incrementally. |
| Token usage | Reads prompt and candidate token counts from Gemini usage metadata. |
| Token cost | Estimates request and session cost from configured per-million-token rates. |
| Conversation context | Reuses chat history across turns and resets it with `/reset`. |
| Latency / UX | Shows output as chunks arrive instead of waiting for the complete response. |

**Important scope note:** This project demonstrates how an application calls and manages an LLM. It does not implement tokenizer internals, transformer attention, prefill/decode, KV-cache internals, or model training. Those concepts are covered in the Day 1 learning notes and interview drill.

## Run locally

See the project README for setup instructions. In brief:

```bash
git clone https://github.com/JainAditi09/ai-cli-chatbot.git
cd ai-cli-chatbot
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# Add your Gemini API key and model pricing to .env
python main.py
```

Never commit the real `.env` file or API key.

## Interview walkthrough

Be ready to explain:
1. How a user prompt and prior chat history are sent to Gemini.
2. How streaming chunks are consumed and displayed.
3. How token usage metadata is used to estimate cost.
4. Why longer histories can increase input token usage and latency.
5. What the app does when a stream or API request fails.

## Next improvements

- Persist chat sessions in SQLite so history survives process restarts.
- Add token-aware context budgeting and summarization.
- Add tests for interrupted streams, missing usage metadata, and API failures.
- Record measured latency and cost across short and long conversations.
