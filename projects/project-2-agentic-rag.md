# Project 2 — Agentic RAG with Metrics

**Source roadmap:** Week 2, Days 8–14.

## Objective
Build a measured RAG system and expose retrieval through MCP.

## Required capabilities
- Ingest a real dataset into pgvector.
- Baseline chunking and retrieval pipeline.
- 40-question test set with known answers.
- Compare hybrid BM25 + vector search, reranking, query rewriting, multi-query, and HyDE one at a time.
- Report Recall@K and MRR.
- Agent decides when/what to retrieve and checks evidence sufficiency.
- Citations in answers.
- Persistent memory in Postgres.
- MCP server exposing RAG search and a database tool.

## Interview walkthrough
Explain chunking decisions, retrieval metrics, access-control filters, RAG vs long context/SQL/fine-tuning, and when RAG is the wrong choice.
