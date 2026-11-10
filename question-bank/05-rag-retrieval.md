# RAG and retrieval — detailed answers

## 1. What problem does RAG solve, and what does it not solve?

**Answer.** Retrieval-augmented generation retrieves relevant external information and supplies it to a model as context before generation. It is useful for private, changing, or domain-specific knowledge that may not be in model weights. A typical pipeline is ingest → normalize → chunk → embed/index → retrieve → optionally rerank → construct grounded prompt → generate with citations. RAG does not guarantee truth: retrieval may miss the answer, return stale or unauthorized data, or the model may misread the evidence.

**Follow-ups**
- *Why not fine-tune?* Fine-tuning changes model behavior/weights; RAG supplies current facts and source passages at request time.
- *Does RAG make the model current?* Only for the information indexed and retrieved at request time.
- *When is RAG unnecessary?* Small static knowledge sets, deterministic database queries, or tasks where a tool/API is the source of truth.

## 2. What is an embedding? How is cosine similarity used in retrieval?

**Answer.** An embedding is a numeric vector representing aspects of text or another input in a learned semantic space. Similar meanings often map to nearby vectors, though embeddings are imperfect and task-dependent. Cosine similarity compares the angle between two vectors: values closer to 1 indicate more aligned directions. A query embedding is compared with document-chunk embeddings to find candidate matches.

**Follow-ups**
- *Does similarity mean factual relevance?* No. It is a ranking signal, not proof that a passage answers the question.
- *Why normalize vectors?* With normalized vectors, dot product and cosine similarity are equivalent and can be efficient.
- *What happens when embedding models change?* Re-embed the corpus or maintain versioned indexes; vectors from incompatible models should not be mixed blindly.

## 3. What is an approximate nearest-neighbor index? Explain HNSW intuitively.

**Answer.** Exact nearest-neighbor search compares a query against every vector, which becomes expensive at scale. Approximate nearest-neighbor (ANN) indexes trade a small amount of recall for faster search. HNSW builds a multi-layer graph: upper layers provide long-range navigation, and lower layers refine the search among nearby vectors. Search traverses the graph rather than scanning every vector.

**Follow-ups**
- *What is the trade-off?* Search speed and memory versus recall and index-build/update cost.
- *How do you tune HNSW?* Evaluate recall/latency while adjusting graph construction and search parameters for the workload.
- *Can ANN miss the best result?* Yes; it is approximate. Measure recall against exact search on a sample.

## 4. Compare pgvector, Qdrant, and Pinecone. What factors guide the choice?

**Answer.** pgvector adds vector similarity search to PostgreSQL, which can simplify operations when relational data and vectors should share transactions and access-control logic. Qdrant is a dedicated vector search engine with filtering and vector-specific features. Pinecone is a managed vector database that reduces operational burden but introduces managed-service cost, network dependency, and vendor considerations. Choose based on scale, filtering needs, latency, operational expertise, tenancy, consistency, cost, and existing infrastructure.

**Follow-ups**
- *When is pgvector attractive?* Moderate scale, relational joins, existing Postgres operations, and a desire to keep metadata/permissions close to application data.
- *When is a managed service attractive?* When operational simplicity and elastic managed infrastructure outweigh cost/control concerns.
- *Should a vector DB own authorization?* Do not rely solely on a client-supplied filter; enforce tenant scope server-side and test isolation.

## 5. Compare fixed, recursive, semantic, and structure-aware chunking.

**Answer.** Fixed chunking splits text at a target size, often with overlap; it is simple but may break sentences or sections. Recursive chunking tries preferred separators (for example paragraphs, then sentences) until chunks fit. Semantic chunking groups text based on meaning or topic shifts, which can preserve coherence but costs more and may be harder to reproduce. Structure-aware chunking respects document boundaries such as headings, code functions, tables, or pages.

**Follow-ups**
- *Which should you start with?* A simple baseline such as recursive or structure-aware chunking, then evaluate alternatives on real queries.
- *Why is code chunking different?* Function/class boundaries and symbol relationships matter more than arbitrary character lengths.
- *Can chunks be too small?* Yes; they may lose context and create many weak matches.

## 6. How do chunk size and overlap affect retrieval quality and cost?

**Answer.** Large chunks preserve more context but may contain multiple topics and dilute similarity; they also consume more prompt tokens. Small chunks are precise but can omit the context needed to interpret a fact. Overlap helps preserve information near boundaries but increases index size and duplicate retrieval. Tune size and overlap with a labelled query set and measure retrieval and answer quality, not intuition alone.

**Follow-ups**
- *What is a good chunk size?* There is no universal value; it depends on document structure, embedding model, query type, and context budget.
- *How do you avoid duplicate context?* Deduplicate overlapping results, merge adjacent chunks, or diversify retrieval.
- *What metric helps choose?* Recall@K/MRR for retrieval plus grounded answer quality and token cost.

## 7. How do metadata filters improve retrieval?

**Answer.** Metadata can represent document type, tenant, owner, language, timestamp, product, or access level. Filters narrow candidate retrieval to records relevant to the task and can enforce access scope. Filters should be applied using trusted server-side identity and policy, not arbitrary model-generated tenant IDs.

**Follow-ups**
- *Can metadata filters reduce recall?* Yes, incorrect or overly restrictive filters can exclude the correct document.
- *What if metadata is missing?* Define ingestion validation and explicit handling; do not assume missing metadata means public access.
- *Where should authorization happen?* In backend services before results are exposed to the model or user.

## 8. How do you enforce per-user or per-tenant access control in retrieval?

**Answer.** Derive the authenticated user and tenant from the server's trusted session/token. Apply mandatory tenant and ACL filters in the retrieval service, then re-check document permissions before returning content. Avoid accepting tenant IDs from model arguments or client filters. Test cross-tenant attacks, direct document IDs, cached results, and citation URLs.

**Follow-ups**
- *Is filtering after retrieval enough?* It can waste resources and risks exposing unauthorized text to intermediate components; enforce scope as early as possible and validate again before response.
- *How can caches leak data?* A cache key that omits tenant/user/ACL version may return another user's result.
- *What should citations do?* Resolve through an authorization-checked endpoint, not a public raw storage URL.

## 9. What is hybrid search? Why combine BM25 and vector search?

**Answer.** BM25 is lexical retrieval based on term frequency and inverse document frequency. Vector search retrieves semantically similar content even when wording differs. Hybrid search combines the rankings or scores, improving coverage for both exact terms (IDs, product names, error codes) and paraphrases. Fusion methods include reciprocal rank fusion or normalized score combination.

**Follow-ups**
- *Why not simply add raw scores?* BM25 and vector scores may have different scales; rank fusion or calibrated normalization is safer.
- *When does lexical search win?* Exact identifiers, rare terms, version numbers, and quoted phrases.
- *How do you know hybrid helps?* Compare on a fixed labelled dataset and report Recall@K, MRR, latency, and cost.

## 10. What does a cross-encoder reranker add after initial retrieval?

**Answer.** A bi-encoder embeds query and documents independently, enabling fast candidate search. A cross-encoder jointly processes the query and each candidate passage, often producing a more precise relevance score but at higher compute cost. A common design retrieves perhaps dozens of candidates cheaply, reranks a smaller set, then sends the top few to the LLM.

**Follow-ups**
- *Why not rerank the entire corpus?* Cross-encoder inference is more expensive and does not scale as a first-stage search.
- *Can reranking hurt?* Yes, domain mismatch or poor training can reorder relevant passages incorrectly.
- *How do you evaluate it?* Measure ranking metrics before/after and include added latency.

## 11. What are query rewriting, multi-query retrieval, and HyDE?

**Answer.** Query rewriting transforms a user's question into a search-friendly form, often resolving references or adding terminology. Multi-query generates several alternative queries and merges their results to improve recall. HyDE generates a hypothetical answer/document and embeds it to locate semantically similar real passages. These methods can increase recall but also introduce query drift, extra model cost, and irrelevant candidates. Retrieved real evidence must remain the source for the final answer.

**Follow-ups**
- *Can HyDE's hypothetical answer be cited?* No. It is a retrieval aid, not evidence.
- *How do you prevent query drift?* Preserve the original intent and evaluate whether expanded queries retrieve relevant documents.
- *When should you avoid multi-query?* Simple queries with strong baseline retrieval or strict latency budgets.

## 12. Define Recall@K and Mean Reciprocal Rank (MRR). When would you use each?

**Answer.** Recall@K measures the fraction of relevant items retrieved within the top K results. If a query has four relevant documents and two appear in the top five, recall@5 is 0.5. MRR averages the reciprocal rank of the first relevant result across queries; a relevant result at rank 1 contributes 1, rank 2 contributes 0.5, and so on. Recall measures coverage; MRR emphasizes how early the first useful result appears.

**Follow-ups**
- *What if multiple relevant passages exist?* Recall@K accounts for all labelled relevant passages; MRR only uses the first relevant rank.
- *What is nDCG?* A ranking metric that can account for graded relevance and position discounts.
- *Do retrieval metrics guarantee answer quality?* No; generation, grounding, and citation correctness need separate evaluation.

## 13. How would you create a 40-question retrieval evaluation set?

**Answer.** Sample real user intents across common, rare, ambiguous, multi-hop, exact-identifier, and no-answer cases. For each query, label relevant source chunks (or documents), expected facts, access scope, and whether the corpus contains an answer. Have labels reviewed for consistency. Keep a held-out set so tuning chunking and retrieval does not overfit the evaluation.

**Follow-ups**
- *Why include no-answer queries?* To test whether the system abstains instead of fabricating an answer.
- *How do you avoid leakage?* Keep held-out queries and documents or time periods separate from tuning.
- *How often should you refresh the set?* Add representative production failures while preserving a stable regression core.

## 14. What is agentic RAG? How does the agent decide whether to retrieve again?

**Answer.** Agentic RAG allows a model-driven controller to decide whether retrieval is needed, which query/tool to use, and whether evidence is sufficient. A safe design has a bounded number of retrieval rounds, explicit evidence sufficiency criteria, source restrictions, and a final grounded response. The model's confidence alone should not authorize high-impact decisions.

**Follow-ups**
- *How do you prevent endless retrieval?* Set round/time/token limits and detect repeated queries.
- *When should it stop?* When evidence meets task criteria, no useful new sources remain, or the budget is exhausted.
- *How is it different from basic RAG?* Basic RAG often retrieves once using a fixed pipeline; agentic RAG can adapt retrieval based on intermediate observations.

## 15. Compare RAG, long context, SQL querying, and fine-tuning. When is each appropriate?

**Answer.** RAG is useful for searching a large or changing corpus and attaching evidence. Long context is useful when a manageable amount of material must be considered together, but can be costly and still requires selection. SQL is appropriate for structured, exact, aggregatable data and should generally be executed with parameterized queries and permissions. Fine-tuning is for changing model behavior, style, or task patterns—not the preferred mechanism for frequently changing factual records.

**Follow-ups**
- *Can these approaches be combined?* Yes; an agent can route to SQL, retrieval, or a long-context summarizer based on task type.
- *Why not fine-tune on every new document?* It is slow to update, hard to cite, and may not reliably store exact facts.
- *How do you choose?* Match data shape, freshness, exactness, corpus size, latency, governance, and evaluation requirements.

## 16. How do you attach citations and verify that an answer is supported by retrieved evidence?

**Answer.** Preserve source identifiers, chunk boundaries, document versions, and metadata through retrieval. Ask the model to associate factual claims with source IDs, then validate that each ID belongs to the retrieved set and is authorized. For high-quality grounding, evaluate whether the cited passage actually entails the claim; a citation existing is not proof of support.

**Follow-ups**
- *Can the model invent a citation ID?* Yes; validate IDs against the retrieval result set.
- *What if evidence conflicts?* Present the conflict with source dates/authority or apply a documented source-of-truth rule.
- *How do you prevent stale citations?* Store document version and retrieval timestamp, and re-check access when the citation is opened.

## 17. What are common causes of a correct document not being retrieved?

**Answer.** Causes include poor extraction/OCR, bad chunk boundaries, missing metadata, embedding mismatch, query vocabulary mismatch, incorrect filters, stale index, low top-K, lexical-only or vector-only limitations, and access-control bugs. Debug each stage independently: ingestion coverage, chunk inspection, embedding/version, filter scope, candidate ranking, reranking, and prompt assembly.

**Follow-ups**
- *How do you distinguish retrieval failure from generation failure?* Inspect retrieved candidates first; if evidence is absent, it is retrieval/ingestion. If present but ignored or misrepresented, it is prompt/generation/grounding.
- *What does a retrieval trace contain?* Query, filters, index/version, candidate IDs/scores, reranker results, and timing.
- *Can increasing top-K fix everything?* No; it may add noise and cost without correcting ingestion or filter defects.
