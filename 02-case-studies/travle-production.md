# Production-Grade RAG System for a Travel Agent

## 1. Problem clarification
We want a production-grade Retrieval-Augmented Generation (RAG) system for a travel agent.

The system should answer user questions using trusted travel knowledge such as:
- destination guides
- airline and hotel policies
- visa/travel rules
- internal support documents
- FAQs
- pricing or product information

The goal is to improve answer quality while reducing hallucinations and keeping latency and cost under control.

---

## 2. Requirements

### Functional requirements
- Ingest travel content from multiple sources
- Chunk and index documents
- Generate embeddings
- Store embeddings in a vector database
- Retrieve relevant context for user queries
- Rerank retrieved chunks
- Build prompts for the LLM
- Evaluate retrieval and generation quality

### Non-functional requirements
- Low latency for user queries
- High answer accuracy
- Strong hallucination control
- Secure handling of internal and user data
- Cost-efficient at scale
- Observable and debuggable in production

---

## 3. High-level architecture

### Main components
1. **Ingestion pipeline**
2. **Chunking layer**
3. **Embedding service**
4. **Vector DB**
5. **Retrieval service**
6. **Reranking service**
7. **Prompt construction layer**
8. **LLM inference layer**
9. **Evaluation and monitoring stack**
10. **Security and access control layer**

### Data flow
1. Raw travel content is ingested from source systems.
2. The content is cleaned, normalized, and chunked.
3. Each chunk is embedded.
4. Embeddings are stored in the vector DB with metadata.
5. At query time, the user question is embedded.
6. The retrieval layer fetches top-k candidate chunks.
7. A reranker scores the candidates again.
8. The best chunks are inserted into a prompt.
9. The LLM generates the final answer.
10. Logs, metrics, traces, and evaluation results are collected.

---

## 4. Ingestion pipeline

### Sources
- Static docs: policies, FAQs, help-center articles
- Internal documents: travel rules, partner terms, support SOPs
- External sources: destination guides, public travel advisories
- Structured sources: product catalogs, hotel metadata, route data

### Pipeline stages
1. **Fetch**
   - Pull documents from APIs, file storage, CMS, or databases.
2. **Validate**
   - Check file format, schema, language, and completeness.
3. **Clean**
   - Remove HTML noise, boilerplate, duplicates, and malformed text.
4. **Normalize**
   - Standardize encoding, timestamps, country names, and formatting.
5. **Version**
   - Attach source version, ingestion time, and document hash.
6. **Enrich**
   - Add metadata such as topic, destination, language, and access scope.
7. **Route**
   - Send documents to the chunking and embedding pipeline.

### Best practices
- Use idempotent ingestion so the same document can be safely reprocessed.
- Track document hashes to detect updates.
- Separate public and internal content early.
- Keep provenance metadata for every chunk.

---

## 5. Chunking

Chunking is one of the most important parts of RAG quality.

### Goals
- Keep chunks small enough for precise retrieval
- Keep chunks large enough to preserve meaning
- Avoid splitting important semantic boundaries

### Recommended strategy
- Chunk by structure first: headings, sections, bullet lists, tables
- Then apply token-based size limits
- Use overlap for continuity
- Keep chunk metadata attached

### Travel-specific considerations
- A visa policy section should remain intact
- Hotel cancellation rules should not be split across unrelated chunks
- Destination FAQs can be chunked by question-answer pair
- Flight rules and baggage policies should be preserved as atomic units when possible

### Suggested chunking parameters
- Chunk size: 300–800 tokens
- Overlap: 50–120 tokens
- Larger chunks for policy or legal text
- Smaller chunks for FAQ-style content

### Metadata to keep
- source document ID
- title
- section heading
- destination
- content type
- version
- access level
- language
- last updated time

---

## 6. Embedding

### Purpose
Embeddings convert chunks and queries into vectors so semantic similarity search can work.

### Design choices
- Use one stable embedding model per index version
- Store embedding model name and version in metadata
- Re-embed when the model changes or the content changes

### Batch strategy
- Embed documents in batches for throughput
- Use retry logic for transient failures
- Store failed records for reprocessing

### Best practices
- Use the same embedding space for documents and queries
- Normalize text before embedding
- Avoid embedding noisy boilerplate
- Track embedding cost per document

### Operational notes
- Keep a reindexing strategy for embedding model upgrades
- Evaluate whether multilingual support is needed for travel content

---

## 7. Vector DB

### What it stores
- Chunk embeddings
- Chunk text or chunk references
- Metadata for filtering and access control

### Requirements
- Fast approximate nearest neighbor search
- Metadata filtering
- Namespace or tenant isolation
- Scalability for large corpora
- Backup and restore support

### Common design pattern
Store each chunk with:
- vector
- chunk text
- document ID
- metadata
- ACL or tenant ID
- version ID

### Indexing strategy
- Use ANN indexes for speed
- Partition by tenant or content domain if needed
- Use hybrid search if supported: vector + keyword

### Travel use case
Hybrid retrieval is useful because travel queries often contain exact terms like:
- airline names
- airport codes
- hotel names
- country names
- policy numbers

Vector-only search may miss exact-match needs, so hybrid search is usually better.

---

## 8. Retrieval

### Retrieval flow
1. Receive user query
2. Normalize and classify query
3. Build query embedding
4. Retrieve top-k from vector DB
5. Apply metadata filters
6. Optionally combine with keyword search
7. Send candidates to reranker

### Retrieval filters
- language
- destination
- product type
- tenant/user permissions
- freshness/version
- content source type

### Query understanding
Before retrieval, classify the query into categories such as:
- destination info
- policy question
- itinerary planning
- booking support
- troubleshooting

This can improve filtering and prompt selection.

### Important practices
- Retrieve enough candidates for reranking
- Use time-based freshness rules for changing content
- Do not return content the user is not authorized to see

---

## 9. Reranking

### Why reranking is needed
Vector search gets good candidates, but not always the best ones.
Reranking improves final relevance.

### Reranking options
- Cross-encoder reranker
- Lightweight LLM-based scoring
- Learning-to-rank model

### Recommended approach
- Retrieve top 20–50 chunks
- Rerank to top 3–8 chunks
- Use metadata-aware reranking when possible

### Travel-specific reranking signals
- exact destination match
- freshness
- authority of the source
- policy relevance
- semantic match to the user’s intent

### Trade-off
Reranking improves quality but adds latency and cost.
Use it when answer quality matters more than ultra-low latency.

---

## 10. Prompt construction

### Prompt goals
- Ground the LLM in retrieved evidence
- Keep responses factual and concise
- Make uncertainty explicit
- Reduce hallucination risk

### Recommended prompt structure
1. System instructions
2. User question
3. Retrieved context
4. Constraints and policies
5. Answer format requirements

### Prompt rules
- Tell the model to answer only from provided context when possible
- Ask it to cite retrieved sources if your product supports citations
- Instruct it to say “I don’t know” when evidence is insufficient
- Separate factual context from instructions clearly

### Example prompt design principles
- Keep top evidence first
- Deduplicate overlapping chunks
- Include source IDs or URLs
- Highlight freshness if relevant
- Keep context within token budget

### Travel-agent behavior
The prompt should instruct the model to:
- avoid inventing prices or availability
- avoid making unsupported booking promises
- ask follow-up questions when user intent is unclear
- distinguish advisory content from live inventory data

---

## 11. Evaluation

### What to evaluate
1. **Retrieval quality**
   - Did we fetch the right chunks?
2. **Reranking quality**
   - Did the best evidence move to the top?
3. **Generation quality**
   - Is the final answer correct and helpful?
4. **Grounding quality**
   - Is the answer supported by evidence?
5. **Latency and cost**
   - Is the system fast enough and affordable?

### Offline evaluation metrics
- Recall@k
- Precision@k
- MRR
- nDCG
- Faithfulness / groundedness
- Answer correctness
- Citation accuracy

### Test set design
Create evaluation questions for:
- visa rules
- cancellation policies
- baggage allowance
- destination recommendations
- hotel check-in policies
- multi-hop travel planning
- ambiguous user questions

### Human review
Use human evaluators to score:
- correctness
- completeness
- hallucination rate
- helpfulness
- citation quality

### Continuous evaluation
- Run regression tests on every content or prompt update
- Compare retrieval drift across index versions
- Track quality by query category

---

## 12. Hallucination control

### Main controls
- Strong retrieval filters
- Better reranking
- Prompt grounding
- Citation requirements
- Confidence thresholds
- Refusal behavior when evidence is weak

### Specific techniques
- If retrieval confidence is low, ask clarifying questions instead of answering
- If context is missing, return “I could not find reliable information”
- Use structured output for critical decisions
- Verify facts like dates, prices, and policies against source data
- Prefer direct quotations for policy text when appropriate

### Travel-specific safety rules
- Never invent live flight availability
- Never fabricate hotel inventory or prices
- Mark policy data as source-backed only
- For time-sensitive info, require freshness checks

### Optional guardrails
- Second-pass answer verifier
- Citation validator
- Rule-based safety checks for high-risk topics

---

## 13. Security

### Security requirements
- Authenticate users
- Authorize access to tenant-specific or internal documents
- Protect secrets and API keys
- Encrypt data in transit and at rest
- Log safely without exposing sensitive data

### Access control
- Use RBAC or ABAC for document access
- Filter retrieval by permission before returning chunks
- Never let the model see unauthorized content

### Data protection
- Redact sensitive information in logs
- Minimize retention of user prompts and personal data
- Protect PII in conversations and support records

### External API safety
- Use allowlists for external sources
- Validate fetched content before ingestion
- Protect against prompt injection from external documents
- Treat external content as untrusted input

### Prompt injection defense
- Separate instructions from retrieved content
- Never allow retrieved text to override system rules
- Detect malicious patterns in documents
- Use content sanitization for web-sourced material

---

## 14. Cost

### Main cost drivers
- Embedding generation
- Vector DB storage and indexing
- Retrieval and reranking latency
- LLM token usage
- Ingestion and reindexing jobs
- Monitoring and logging volume

### Cost optimization strategies
- Deduplicate documents before embedding
- Chunk carefully to avoid too many small vectors
- Use cheaper embedding models if quality is acceptable
- Cache frequent query results
- Limit reranker usage to important queries
- Keep prompts short and focused
- Use hybrid retrieval efficiently instead of over-retrieving

### Cost trade-offs
- Higher retrieval depth usually improves quality but increases cost
- Reranking improves precision but adds latency
- Larger models improve answer quality but cost more per request

### Practical recommendation
Start with a high-quality but cost-aware setup:
- moderate chunk sizes
- top-k retrieval with reranking
- cached embeddings
- small prompt windows
- strong evaluation before scaling model size

---

## 15. Monitoring

### What to monitor
- Query volume
- Retrieval latency
- Reranking latency
- LLM latency
- Token usage
- Cache hit rate
- Vector DB latency
- Ingestion failures
- Hallucination rate
- Citation coverage
- User feedback scores

### Logging
Log:
- query ID
- user intent category
- retrieved document IDs
- reranker scores
- prompt version
- model version
- response length
- latency breakdown
- error codes

### Tracing
Use distributed tracing to follow a request across:
- frontend
- API layer
- retrieval
- reranking
- LLM generation
- external services

### Alerting
Create alerts for:
- retrieval failure spikes
- vector DB latency increases
- LLM error spikes
- ingestion backlog growth
- low answer quality signals
- unusual prompt injection patterns

### Business monitoring
Also track product metrics:
- answer acceptance rate
- follow-up question rate
- user satisfaction
- conversion to booking or next action

---

## 16. Latency optimization for voice-enabled travel agent

### The latency problem
Traditional RAG systems for voice interaction suffer from high latency:
- STT (Speech-to-Text) waits for full audio before processing
- LLM generates response after STT completes
- TTS (Text-to-Speech) starts only after LLM finishes
- Total latency: 3–6 seconds or more

This creates a poor conversational experience.

### Solution: streaming end-to-end pipeline

Use a fully streaming architecture where each stage processes and forwards data incrementally:

**STT → LLM → TTS** all run concurrently, not sequentially.

### Architecture comparison

| Approach | Latency | User experience | Complexity |
|----------|---------|-----------------|------------|
| **Sequential (baseline)** | 3–6s | Poor, noticeable delay | Low |
| **Partial streaming** | 1.5–3s | Better, but still laggy | Medium |
| **Full streaming + VAD + same-region** | 500ms–1s | Near real-time, natural conversation | High |

### Implementation strategy

#### 1. Azure Speech streaming with partial results
- Use Azure Speech Service real-time STT
- Enable partial recognition results
- Forward partial transcripts to LLM immediately
- Do not wait for final transcript

Benefits:
- LLM starts processing while user is still speaking
- Perceived latency drops significantly

#### 2. Azure nhà cung cấp dịch vụ AI streaming response
- Use streaming completion API
- Forward tokens as they are generated
- Do not wait for full completion

Benefits:
- TTS can start speaking before the full answer is ready
- User hears the first words within 500ms

#### 3. WebSocket persistent connection
- Maintain long-lived WebSocket connection between client and server
- Avoid TCP/TLS handshake overhead per request
- Enable bidirectional streaming

Benefits:
- Lower connection latency
- Supports real-time audio streaming
- Better for conversational back-and-forth

#### 4. Voice Activity Detection (VAD)
- Detect end of speech without waiting for silence timeout
- Use cloud VAD or client-side VAD
- Trigger STT finalization and LLM processing immediately

Benefits:
- Eliminates 500ms–1s silence detection delay
- Faster turn-taking in conversation

#### 5. Same-region deployment
- Deploy all services in the same Azure region
- STT, LLM, TTS, Vector DB, and backend in one region
- Use private endpoints when possible

Benefits:
- Reduces inter-region network latency (50–200ms per hop)
- More consistent latency

### Streaming pipeline flow

```
User speaks
  ↓
[Azure Speech STT] ← WebSocket
  ↓ (partial results)
[LangGraph Agent + RAG]
  ↓ (retrieve context in parallel)
[Azure nhà cung cấp dịch vụ AI] ← streaming prompt
  ↓ (stream tokens)
[Azure Speech TTS] ← streaming text
  ↓ (stream audio chunks)
User hears response
```

Everything flows concurrently.

### Latency breakdown (optimized)

| Stage | Baseline | Optimized | Improvement |
|-------|----------|-----------|-------------|
| STT (wait for silence) | 1–2s | 200–400ms (VAD) | 60–80% faster |
| Retrieval + LLM first token | 800ms–1.5s | 300–600ms (streaming + region) | 50% faster |
| TTS first audio | 500ms–1s | 100–200ms (streaming) | 70–80% faster |
| **Total perceived latency** | 3–6s | **500ms–1s** | **5–10x improvement** |

### Key optimization principles

1. **Start early**: Begin processing before the full input is ready
2. **Stream everything**: Never wait for a full response before forwarding
3. **Parallel where possible**: Run retrieval and embedding in parallel
4. **Minimize round trips**: Use WebSocket, same-region deployment
5. **Cut silence delays**: Use VAD instead of timeout-based end-of-speech

### Trade-offs

| Optimization | Benefit | Cost |
|--------------|---------|------|
| Streaming STT | Lower latency | Partial results may be inaccurate, may need correction |
| Streaming LLM | Faster first response | Cannot rerank or filter full response before sending |
| VAD | Faster turn-taking | May cut off user if sensitivity is too high |
| Same-region deployment | Lower network latency | Higher infrastructure cost if multi-region is needed later |
| WebSocket | Persistent connection, low overhead | More complex connection management, scaling, and debugging |

### Recommended stack for voice-enabled travel agent

- **STT**: Azure Speech Service (streaming mode, partial results enabled)
- **LLM**: Azure nhà cung cấp dịch vụ AI (streaming API)
- **TTS**: Azure Speech Service (streaming synthesis)
- **Transport**: WebSocket for bidirectional audio streaming
- **VAD**: Azure Speech VAD or client-side Silero VAD
- **Region**: Single Azure region (e.g., East US, West Europe) for all services
- **Vector DB**: Deploy in same region (e.g., Pinecone, Azure AI Search)

### Production considerations

- **Retry and fallback**: If streaming fails, fall back to batch mode
- **Monitoring**: Track end-to-end latency percentiles (p50, p95, p99)
- **Error recovery**: Handle partial transcript corrections gracefully
- **Interruption support**: Allow user to interrupt agent mid-response
- **Cost**: Streaming may use more API calls; monitor usage and optimize

---

## 17. Recommended production blueprint

### MVP-to-production path
1. Start with a clean ingestion pipeline
2. Build quality chunking rules
3. Add stable embeddings and a vector DB
4. Use hybrid retrieval if possible
5. Add reranking for quality-sensitive queries
6. Keep prompts short and evidence-based
7. Build an offline evaluation set early
8. Add hallucination guardrails and citations
9. Lock down access control and logging
10. Monitor quality continuously and reindex when content changes
11. **For voice interaction**: Add streaming STT/LLM/TTS pipeline with VAD and same-region deployment

### Final recommendation
For a travel agent, production RAG should prioritize:
- trustworthy answers
- freshness
- access control
- low hallucination rate
- measurable quality
- **low latency for voice conversations** (if voice-enabled)

The best system is not the most complex one — it is the one that is reliably grounded in the right travel information and delivers answers quickly enough for the use case.
