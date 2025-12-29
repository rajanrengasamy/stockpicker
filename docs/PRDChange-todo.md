# PRD Change: RAG & Vector Database Integration

**Document:** `PRDChange-todo.md`
**Created:** 2025-12-29
**Last Updated:** 2025-12-29
**Status:** Draft — Pending Review
**Architecture Impact:** Major — Affects Phase -1 and all engine execution
**Source Location:** `/Users/rajan/Documents/Projects/stockpicker/sources/`

---

## 1. Executive Summary

### 1.1 Change Request

Migrate from static prompt concatenation of knowledge playbooks to a **Retrieval-Augmented Generation (RAG)** architecture with **vector database** storage for dynamic, context-aware knowledge retrieval.

### 1.2 Current State (v4.2)

```
Phase -1: Knowledge Curator (GPT)
├── Input: Theory PDFs (Intelligent Investor, etc.)
├── Process: Extract theory into markdown playbooks
├── Output: knowledge/*.md files (versioned)
└── Injection: Full playbooks concatenated into engine prompts
```

**Limitations of Current Approach:**
- Full playbook concatenation consumes significant context window
- No semantic retrieval — all content injected regardless of relevance
- Scaling issues — more books = larger prompts = higher costs
- No ability to query specific concepts dynamically
- Version management via file hashing only

### 1.3 Proposed State (v5.0)

```
Phase -1: Knowledge Ingestion Pipeline
├── Input: Theory PDFs + structured documents
├── Process: Parse → Chunk → Embed → Index
├── Storage: Local vector database (persistent)
└── Retrieval: Semantic search per engine query

Engine Execution:
├── Step 1: Formulate retrieval query based on task
├── Step 2: Retrieve top-k relevant knowledge chunks
├── Step 3: Inject retrieved context into prompt
└── Step 4: Execute LLM analysis
```

### 1.4 Key Benefits

| Benefit | Impact |
|:--------|:-------|
| **Reduced token usage** | Only inject relevant knowledge (est. 60-80% reduction) |
| **Better precision** | Semantic matching surfaces most relevant theory |
| **Scalability** | Add unlimited books without prompt bloat |
| **Dynamic retrieval** | Different queries retrieve different knowledge |
| **Auditability** | Track which knowledge chunks influenced each analysis |
| **Hybrid search** | Combine semantic + keyword for financial terminology |

---

## 2. Source Material Inventory

### 2.1 Available PDFs (16 files, ~152MB)

Located in `/Users/rajan/Documents/Projects/stockpicker/sources/`

| # | Book Title | Author(s) | Size | Primary Domain | Notes |
|:--|:-----------|:----------|:-----|:---------------|:------|
| 1 | **The Intelligent Investor** | Benjamin Graham | 5.5MB | Core Principles, Valuation | Foundation text |
| 2 | **Security Analysis** | Benjamin Graham | 3.0MB | Valuation, Fundamentals | 6th Edition |
| 3 | **Common Stocks and Uncommon Profits** | Philip Fisher | 2.6MB | Growth Investing, Governance | Scuttlebutt method |
| 4 | **The Education of a Value Investor** | Guy Spier | 1.6MB | Core Principles, Behavioral | Investment philosophy |
| 5 | **Warren Buffett and the Interpretation of Financial Statements** | Mary Buffett, David Clark | 8.9MB | Fundamentals, Valuation | Financial statement analysis |
| 6 | **Misbehaving: The Making of Behavioral Economics** | Richard Thaler | 21.9MB | Behavioral | Nobel laureate, behavioral economics |
| 7 | **Irrational Exuberance** | Robert Shiller | 4.6MB | Behavioral, Valuation | Market psychology, bubbles |
| 8 | **Competitive Advantage** | Michael Porter | 29.3MB | Supply Chain, Strategy | Value chain analysis |
| 9 | **Competitive Strategy** | Michael Porter | 2.2MB | Supply Chain, Strategy | Industry analysis, five forces |
| 10 | **Geopolitical Alpha** | Marko Papic | 6.7MB | Geopolitics | Investment framework for geopolitics |
| 11 | **Prisoners of Geography** | Tim Marshall | 0.2MB | Geopolitics | Geographic constraints on nations |
| 12 | **Principles: Life and Work** | Ray Dalio | 6.4MB | Core Principles, Governance | Decision-making framework |
| 13 | **The Investment Checklist** | Michael Shearn | 21.5MB | Fundamentals, Governance | Due diligence methodology |
| 14 | **Guide to Economic Indicators** | The Economist | 4.0MB | Macro, Catalysts | Economic indicator reference |
| 15 | **Mapping the Markets** | Deborah Owen, Robin Griffiths | 2.8MB | Technical, Macro | Market analysis guide |

**Total: 15 unique books** (~121MB after removing duplicate)

> **Note:** `Competitive Advantage - Michael Porter` appears twice (29.3MB and 31.0MB). The duplicate should be removed before ingestion to avoid redundant embeddings.

### 2.2 Source-to-Collection Mapping

| Collection | Primary Sources | Relevance |
|:-----------|:----------------|:----------|
| **core_principles** | Intelligent Investor, Education of Value Investor, Principles, Common Stocks | Foundational investment philosophy |
| **valuation** | Security Analysis, Intelligent Investor, Buffett Financial Statements, Irrational Exuberance | Valuation methodologies |
| **behavioral** | Misbehaving, Irrational Exuberance, Education of Value Investor | Cognitive biases, market psychology |
| **governance** | Investment Checklist, Common Stocks, Principles | Management assessment, capital allocation |
| **supply_chain** | Competitive Advantage, Competitive Strategy | Value chain, competitive dynamics |
| **geopolitics** | Geopolitical Alpha, Prisoners of Geography | Policy analysis, geographic constraints |
| **catalysts** | Guide to Economic Indicators, Mapping the Markets | Event-driven, macro indicators |
| **fundamentals** | Security Analysis, Buffett Financial Statements, Investment Checklist | Financial statement analysis |

### 2.3 Updated Cost Estimates

Based on actual source materials:

| Item | Calculation | Estimate |
|:-----|:------------|:---------|
| **Total PDF size** | 121MB (after dedup) | — |
| **Estimated text extraction** | ~40-50% of PDF size = ~55MB text | — |
| **Estimated tokens** | 55MB ÷ 4 bytes/token ≈ 2.0M tokens | 2,000,000 tokens |
| **Embedding cost (one-time)** | 2M × $0.00013/1k | **~$0.26** |
| **Chunk count (est.)** | 2M tokens ÷ 450 tokens/chunk | ~4,500 chunks |
| **Vector storage** | 4,500 × 3072 dims × 4 bytes | ~55MB |

**One-time embedding cost: ~$0.26** (previously estimated $0.10 with fewer books)

### 2.4 Pre-Ingestion Cleanup Tasks

- [ ] Remove duplicate Porter book (`Competitive advantage...Porter...(1).pdf`)
- [ ] Verify all PDFs are readable (not password-protected or corrupted)
- [ ] Create source manifest with SHA-256 hashes for versioning

---

## 3. Architecture Deep Dive

### 3.1 New Component: Knowledge Ingestion Pipeline

Replaces the current Knowledge Curator's output format.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PHASE -1: KNOWLEDGE INGESTION                        │
│                                                                         │
│   Step 1: Document Processing                                           │
│   ├── PDF Extraction (PyMuPDF / Unstructured.io)                       │
│   ├── Structure Detection (headers, sections, tables)                   │
│   └── Markdown Conversion (preserves hierarchy)                         │
│                              │                                          │
│                              ▼                                          │
│   Step 2: Chunking Strategy                                             │
│   ├── Semantic Chunking (sentence-transformer boundaries)              │
│   ├── Section-Aware (respect chapter/section breaks)                   │
│   ├── Overlap: 10-20% for context continuity                           │
│   └── Target: 400-512 tokens per chunk (embedding model optimized)     │
│                              │                                          │
│                              ▼                                          │
│   Step 3: Metadata Enrichment                                           │
│   ├── Source: {book_title, author, chapter, section}                   │
│   ├── Category: {valuation, behavioral, governance, etc.}              │
│   ├── Concepts: LLM-extracted key concepts per chunk                   │
│   └── Chunk ID: Deterministic hash for deduplication                   │
│                              │                                          │
│                              ▼                                          │
│   Step 4: Embedding Generation                                          │
│   ├── Model: text-embedding-3-large (OpenAI) OR nomic-embed-text-v1   │
│   ├── Dimensions: 3072 (OpenAI) or 768 (Nomic)                         │
│   └── Batch processing with rate limiting                              │
│                              │                                          │
│                              ▼                                          │
│   Step 5: Vector Storage                                                │
│   ├── Database: Qdrant (local) or Chroma (prototyping)                 │
│   ├── Collections: Organized by knowledge domain                       │
│   ├── Indexes: HNSW for fast approximate nearest neighbor              │
│   └── Persistence: Local disk storage                                  │
│                                                                         │
│   Output: Indexed knowledge base ready for retrieval                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 New Component: Knowledge Retrieval Layer

Sits between the orchestrator and each engine.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RETRIEVAL LAYER (Per Engine Call)                    │
│                                                                         │
│   Input: Engine context (ticker, sector, analysis type)                 │
│                              │                                          │
│                              ▼                                          │
│   Step 1: Query Formulation                                             │
│   ├── Engine-specific query templates                                   │
│   ├── Example (Fundamentals): "valuation methods for {sector} stocks"  │
│   ├── Example (Contrarian): "behavioral biases market psychology"      │
│   └── Multiple queries per engine for comprehensive retrieval          │
│                              │                                          │
│                              ▼                                          │
│   Step 2: Hybrid Search                                                 │
│   ├── Semantic: Vector similarity (cosine distance)                    │
│   ├── Keyword: BM25 for exact financial terms (P/E, DCF, ROIC)        │
│   ├── Fusion: Reciprocal Rank Fusion (RRF) to combine results          │
│   └── Retrieve top-k candidates (k=20-50)                              │
│                              │                                          │
│                              ▼                                          │
│   Step 3: Reranking (Optional but Recommended)                          │
│   ├── Model: cross-encoder/ms-marco-MiniLM-L-6-v2 (local)              │
│   ├── OR: Cohere Rerank API (higher quality, adds latency)             │
│   ├── Rerank top-50 → select top-10                                    │
│   └── Adds ~50-200ms latency, improves precision 20-35%                │
│                              │                                          │
│                              ▼                                          │
│   Step 4: Context Assembly                                              │
│   ├── Deduplicate overlapping chunks                                   │
│   ├── Order by relevance score                                         │
│   ├── Truncate to fit context budget (e.g., 4000 tokens)               │
│   └── Format with source attribution                                   │
│                              │                                          │
│                              ▼                                          │
│   Output: Retrieved knowledge context + source metadata                 │
│   Injected into engine prompt with delimiters                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Updated Engine Execution Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ENGINE EXECUTION (Updated)                           │
│                                                                         │
│   Example: Fundamentals Engine                                          │
│                                                                         │
│   Inputs:                                                               │
│   ├── 00_scout.json (discovery context)                                │
│   ├── 00_data_pack.json (FMP financial data)                           │
│   └── [RETRIEVED] Relevant knowledge chunks                   ← NEW    │
│                              │                                          │
│                              ▼                                          │
│   Prompt Assembly:                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │ ## System Instructions                                          │  │
│   │ You are the Fundamentals Engine...                              │  │
│   │                                                                 │  │
│   │ ## Retrieved Knowledge Context                          ← NEW   │  │
│   │ ═══════════ BEGIN KNOWLEDGE CONTEXT ═══════════                │  │
│   │ [Chunk 1] Source: The Intelligent Investor, Ch. 8              │  │
│   │ "The margin of safety concept..."                               │  │
│   │                                                                 │  │
│   │ [Chunk 2] Source: Valuation (McKinsey), Ch. 12                 │  │
│   │ "ROIC decomposition reveals..."                                 │  │
│   │ ═══════════ END KNOWLEDGE CONTEXT ═════════════                │  │
│   │                                                                 │  │
│   │ ## Financial Data (from data_pack)                              │  │
│   │ ═══════════ BEGIN EXTERNAL CONTENT ═══════════                 │  │
│   │ {...FMP data...}                                                │  │
│   │ ═══════════ END EXTERNAL CONTENT ═════════════                 │  │
│   │                                                                 │  │
│   │ ## Your Analysis Task                                           │  │
│   │ Analyze the fundamentals of {ticker}...                         │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│                              ▼                                          │
│   Output: 01_fundamentals.json                                          │
│   ├── Standard analysis output                                         │
│   ├── _meta.knowledge_chunks: ["chunk_id_1", "chunk_id_2", ...]← NEW  │
│   └── Full provenance for audit                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Technology Recommendations

### 4.1 Vector Database Selection

**Context:** Application runs locally, LLM APIs are external. Need local-first vector DB.

| Option | Recommendation | Rationale |
|:-------|:---------------|:----------|
| **Qdrant** | **RECOMMENDED for Production** | Rust-based, excellent performance, runs locally via Docker or embedded mode, strong filtering, hybrid search support, active development |
| **Chroma** | Good for Prototyping | Python-native, easiest setup, but less performant at scale, limited filtering |
| **Weaviate** | Alternative | More complex setup, better for GraphRAG patterns |
| **Pinecone** | Not Recommended | Cloud-only, overkill for single-user local deployment |

**Decision: Qdrant (local mode)**

```python
# Qdrant can run in-process (no Docker needed)
from qdrant_client import QdrantClient

# Option 1: Embedded (simplest, data in local folder)
client = QdrantClient(path="./knowledge_db")

# Option 2: Local server (Docker)
# docker run -p 6333:6333 qdrant/qdrant
client = QdrantClient(host="localhost", port=6333)
```

### 4.2 Embedding Model Selection

| Option | Dimensions | Quality | Cost | Latency | Recommendation |
|:-------|:-----------|:--------|:-----|:--------|:---------------|
| **OpenAI text-embedding-3-large** | 3072 | Excellent | $0.00013/1k tokens | ~100ms | **RECOMMENDED** — Best quality, reasonable cost for knowledge base size |
| **OpenAI text-embedding-3-small** | 1536 | Good | $0.00002/1k tokens | ~80ms | Budget alternative |
| **Nomic nomic-embed-text-v1** | 768 | Very Good | Free (local) | ~50ms (GPU) | Privacy-focused, requires local GPU |
| **Cohere embed-v4** | 1536 | Excellent | $0.12/1M tokens | ~100ms | Good multilingual, but overkill |

**Decision: OpenAI text-embedding-3-large**

Rationale:
- Knowledge base is relatively small (a few books = ~50k-200k tokens)
- One-time embedding cost is minimal (~$0.03-0.10 total)
- Best quality for financial/investment domain
- Consistent with using OpenAI for Decision Engine

**Cost Estimate:**
- 10 investment books ≈ 500,000 tokens
- Embedding cost: $0.065 (one-time)
- Query embeddings: ~100 queries/day × 500 tokens = negligible

### 4.3 Framework Selection

| Option | Strengths | Weaknesses | Recommendation |
|:-------|:----------|:-----------|:---------------|
| **LlamaIndex** | RAG-first design, excellent ingestion pipeline, good observability, simpler learning curve | Less flexible for complex agent workflows | **RECOMMENDED** — Perfect fit for knowledge retrieval use case |
| **LangChain** | Flexible orchestration, many integrations, LangGraph for agents | Steeper learning curve, RAG is not primary focus | Alternative for future agent features |
| **Custom** | Full control, no dependencies | More development time, reinvent the wheel | Not recommended for V1 |

**Decision: LlamaIndex**

Rationale:
- RAG is the primary use case
- Built-in support for: PDF parsing, chunking strategies, Qdrant integration, hybrid search
- Lower complexity than LangChain for this specific task
- Can integrate with LangChain later if agentic features needed

### 4.4 Chunking Strategy

Based on research on financial documents:

| Strategy | Use Case | Recommendation |
|:---------|:---------|:---------------|
| **Semantic Chunking** | General text | Use as primary strategy |
| **Section-Aware** | Preserve chapter/section boundaries | Enable for structured books |
| **Fixed Size + Overlap** | Fallback | 400-512 tokens, 10-15% overlap |

**Decision: Hybrid approach**

```python
# LlamaIndex chunking configuration
from llama_index.core.node_parser import (
    SemanticSplitterNodeParser,
    SentenceWindowNodeParser,
)

# Primary: Semantic chunking with section awareness
node_parser = SemanticSplitterNodeParser(
    buffer_size=1,  # sentences before/after for context
    breakpoint_percentile_threshold=95,  # semantic boundary sensitivity
    embed_model=embed_model,
)

# Fallback: Fixed size with overlap
fallback_parser = SentenceSplitter(
    chunk_size=512,
    chunk_overlap=64,  # ~12% overlap
)
```

### 4.5 Reranking Strategy

| Option | Latency | Quality Improvement | Cost | Recommendation |
|:-------|:--------|:--------------------|:-----|:---------------|
| **No reranker** | 0ms | Baseline | Free | Start here |
| **cross-encoder/ms-marco-MiniLM-L-6-v2** | 50-100ms | +15-25% | Free (local) | **RECOMMENDED** — Good balance |
| **Cohere Rerank** | 150-300ms | +25-35% | $2/1k queries | Future upgrade if needed |
| **BGE-reranker-large** | 100-200ms | +20-30% | Free (local) | Alternative to MiniLM |

**Decision: Two-phase approach**

1. **V1:** No reranker — validate base retrieval quality first
2. **V1.1:** Add cross-encoder/ms-marco-MiniLM-L-6-v2 if precision insufficient

---

## 5. Knowledge Organization Schema

### 5.1 Collection Structure (Qdrant)

```
knowledge_db/
├── collection: "core_principles"
│   ├── Margin of safety
│   ├── Circle of competence
│   └── Long-term thinking
│
├── collection: "valuation"
│   ├── DCF methodology
│   ├── Relative valuation
│   ├── Sector-specific (banks, REITs, etc.)
│   └── Valuation pitfalls
│
├── collection: "behavioral"
│   ├── Cognitive biases
│   ├── Market psychology
│   └── Contrarian thinking
│
├── collection: "governance"
│   ├── Management quality assessment
│   ├── Capital allocation
│   └── Incentive alignment
│
├── collection: "supply_chain"
│   ├── Value chain analysis
│   ├── Competitive dynamics
│   └── Picks and shovels
│
├── collection: "geopolitics"
│   ├── Regulatory analysis
│   ├── Trade policy
│   └── Scenario planning
│
└── collection: "catalysts"
    ├── Event-driven analysis
    ├── Earnings patterns
    └── Corporate actions
```

### 5.2 Chunk Metadata Schema

```json
{
  "chunk_id": "sha256:a1b2c3...",
  "content": "The margin of safety concept suggests...",
  "metadata": {
    "source": {
      "title": "The Intelligent Investor",
      "author": "Benjamin Graham",
      "edition": "Revised Edition, 2003",
      "chapter": "Chapter 20: Margin of Safety",
      "page_range": [512, 515]
    },
    "category": "core_principles",
    "subcategory": "margin_of_safety",
    "concepts": ["margin of safety", "risk management", "valuation buffer"],
    "ingestion": {
      "timestamp": "2025-12-29T10:00:00Z",
      "pipeline_version": "1.0.0",
      "chunk_strategy": "semantic",
      "token_count": 487
    }
  },
  "embedding": [0.023, -0.041, ...],  // 3072 dimensions
  "embedding_model": "text-embedding-3-large"
}
```

### 5.3 Engine-to-Collection Mapping

| Engine | Primary Collections | Query Templates |
|:-------|:--------------------|:----------------|
| **Fundamentals** | valuation, core_principles, fundamentals | "valuation methods for {sector}", "financial ratio analysis", "margin of safety" |
| **Peer Benchmark** | valuation, supply_chain | "peer comparison methodology", "relative valuation", "competitive positioning" |
| **Supply Chain** | supply_chain, core_principles | "value chain analysis", "competitive advantage", "industry structure" |
| **Governance** | governance, core_principles | "management quality indicators", "capital allocation", "incentive alignment" |
| **Contrarian** | behavioral, core_principles | "market psychology", "contrarian indicators", "crowd behavior", "second-level thinking" |
| **Geopolitics** | geopolitics | "regulatory analysis", "policy impact", "geographic constraints", "political risk" |
| **Catalyst** | catalysts, fundamentals | "event-driven investing", "earnings analysis", "corporate actions" |
| **Quality** | (no retrieval) | Uses validation rules only — no knowledge needed |
| **Decision** | core_principles, valuation, behavioral | "investment decision framework", "risk assessment", "position sizing" |

### 5.4 Query Template Configuration (config/engine_queries.yaml)

```yaml
# Example configuration for engine query templates
fundamentals_engine:
  collections: [valuation, core_principles, fundamentals]
  queries:
    - template: "valuation methods for {sector} companies"
      weight: 1.0
    - template: "financial statement analysis {sector}"
      weight: 0.8
    - template: "margin of safety calculation"
      weight: 0.7
  max_chunks: 10
  context_budget_tokens: 4000

contrarian_engine:
  collections: [behavioral, core_principles]
  queries:
    - template: "market psychology and crowd behavior"
      weight: 1.0
    - template: "contrarian investing indicators"
      weight: 0.9
    - template: "second-level thinking investment"
      weight: 0.8
  max_chunks: 8
  context_budget_tokens: 3000
```

---

## 6. Updated Phase Structure

### 6.1 Phase -1: Knowledge Ingestion (Replaces Knowledge Curator)

| Attribute | Current (v4.2) | Proposed (v5.0) |
|:----------|:---------------|:----------------|
| **Name** | Knowledge Curator | Knowledge Ingestion Pipeline |
| **Type** | LLM (GPT) | **Deterministic Pipeline + LLM assist** |
| **Input** | Theory PDFs | Theory PDFs, structured documents |
| **Process** | Extract to markdown playbooks | Parse → Chunk → Enrich → Embed → Index |
| **Output** | `knowledge/*.md` files | Qdrant vector database |
| **Storage** | Flat files | Persistent vector DB on disk |
| **Versioning** | File content hash | Collection versioning + chunk IDs |
| **Runs** | One-time + periodic updates | One-time + incremental updates |

**New Pipeline Components:**

```
scripts/knowledge_pipeline/
├── __init__.py
├── ingest.py              # Main entry point
├── parsers/
│   ├── pdf_parser.py      # PyMuPDF + Unstructured.io
│   ├── markdown_parser.py # For existing playbooks
│   └── structured_parser.py # For tables, lists
├── chunkers/
│   ├── semantic_chunker.py
│   ├── section_chunker.py
│   └── hybrid_chunker.py
├── enrichers/
│   ├── metadata_extractor.py  # LLM-assisted concept extraction
│   └── category_classifier.py
├── embedders/
│   ├── openai_embedder.py
│   └── local_embedder.py      # Nomic fallback
├── storage/
│   ├── qdrant_store.py
│   └── chroma_store.py        # Fallback for testing
└── cli.py                     # Command-line interface
```

**CLI Usage:**

```bash
# Initial ingestion (all 15 books)
python -m knowledge_pipeline.cli ingest \
  --source ./sources/ \
  --output ./knowledge_db/ \
  --embedding-model openai \
  --chunking-strategy semantic

# Incremental update (new book)
python -m knowledge_pipeline.cli add \
  --file "./sources/new_book.pdf" \
  --collection valuation

# Verify index
python -m knowledge_pipeline.cli verify

# Query test
python -m knowledge_pipeline.cli query \
  --text "margin of safety valuation" \
  --top-k 5

# List collections and stats
python -m knowledge_pipeline.cli stats
```

### 6.2 Phase 0-4: Updated with Retrieval Layer

```
Phase 0: Discovery + Data (UNCHANGED)
├── Step 0A: Scout Engine (Perplexity) — web search
└── Step 0B: Data Compiler (FMP) — financial data

Phase 1: Foundation (UPDATED)
├── Fundamentals Engine
│   ├── RETRIEVE: Query "valuation {sector}" from knowledge_db
│   ├── EXECUTE: Analyze with retrieved context
│   └── OUTPUT: Include knowledge_chunk_ids in _meta
│
└── Peer Benchmark Engine
    ├── RETRIEVE: Query "peer comparison methodology"
    ├── EXECUTE: Analyze with retrieved context
    └── OUTPUT: Include knowledge_chunk_ids in _meta

Phase 2: Context (UPDATED)
├── Governance Engine [+ retrieval]
├── Supply Chain Engine [+ retrieval]
└── Catalyst Engine [+ retrieval]

Phase 3: Analysis (UPDATED)
├── Geopolitics Engine [+ retrieval]
└── Contrarian Engine [+ retrieval]

Phase 4: Synthesis (UPDATED)
├── Quality Engine (NO retrieval — uses validation rules)
└── Decision Engine [+ retrieval for final synthesis]
```

### 6.3 Retrieval Integration in Orchestrator

```python
# scripts/run_analysis.py (updated)

from knowledge_pipeline.retrieval import KnowledgeRetriever

class AnalysisOrchestrator:
    def __init__(self, config):
        self.retriever = KnowledgeRetriever(
            db_path="./knowledge_db",
            embedding_model="text-embedding-3-large",
            top_k=10,
            reranker=None,  # Add in v1.1 if needed
        )

    async def run_engine(self, engine_name: str, inputs: dict) -> dict:
        # Step 1: Formulate retrieval queries
        queries = self.get_retrieval_queries(engine_name, inputs)

        # Step 2: Retrieve relevant knowledge
        knowledge_context = await self.retriever.retrieve(
            queries=queries,
            collections=ENGINE_COLLECTION_MAP[engine_name],
            max_tokens=4000,
        )

        # Step 3: Assemble prompt with knowledge
        prompt = self.assemble_prompt(
            engine_name=engine_name,
            inputs=inputs,
            knowledge_context=knowledge_context,
        )

        # Step 4: Execute engine
        result = await self.execute_llm(engine_name, prompt)

        # Step 5: Add provenance
        result["_meta"]["knowledge_chunks"] = [
            chunk.id for chunk in knowledge_context.chunks
        ]

        return result
```

---

## 7. Updated _meta Schema

### 7.1 Knowledge Provenance Extension

```json
{
  "_meta": {
    "engine_name": "fundamentals_engine",
    "engine_version": "1.0.0",
    "model": "claude-3-opus-20240229",
    "temperature": 0.0,
    "prompt_hash": "sha256:a1b2c3...",
    "input_hashes": {
      "00_scout.json": "sha256:d4e5f6...",
      "00_data_pack.json": "sha256:g7h8i9..."
    },
    "generated_at": "2025-12-29T14:30:00Z",

    "knowledge_retrieval": {
      "queries": [
        "valuation methods for banking sector",
        "ROIC analysis financial institutions"
      ],
      "chunks_retrieved": [
        {
          "chunk_id": "sha256:abc123...",
          "source": "The Intelligent Investor, Ch. 20",
          "relevance_score": 0.89
        },
        {
          "chunk_id": "sha256:def456...",
          "source": "Valuation (McKinsey), Ch. 12",
          "relevance_score": 0.84
        }
      ],
      "retrieval_model": "text-embedding-3-large",
      "reranker_model": null,
      "total_tokens_retrieved": 3847
    }
  }
}
```

---

## 8. Migration Path

### 8.1 Phase 1: Parallel Operation (Recommended)

Run both systems in parallel during transition:

```
Week 1-2: Build Knowledge Ingestion Pipeline
├── Implement PDF parsing
├── Implement chunking strategies
├── Set up Qdrant locally
├── Embed existing PDFs
└── Validate retrieval quality

Week 3: Build Retrieval Layer
├── Implement retrieval module
├── Integrate with one engine (Fundamentals)
├── Compare output quality vs concatenation
└── Tune retrieval parameters

Week 4: Full Integration
├── Integrate with all engines
├── Update orchestrator
├── Update _meta schema
├── Regression testing
└── Documentation update
```

### 8.2 Rollback Strategy

Keep playbook concatenation as fallback:

```python
class AnalysisOrchestrator:
    def __init__(self, config):
        self.use_rag = config.get("use_rag", True)
        if self.use_rag:
            self.retriever = KnowledgeRetriever(...)
        else:
            self.playbook_loader = PlaybookLoader(...)  # Legacy
```

### 8.3 Validation Criteria

Before full migration:

| Metric | Threshold | Measurement |
|:-------|:----------|:------------|
| Retrieval Recall | >80% | Manual review: Are relevant concepts retrieved? |
| Engine Output Quality | Equivalent or better | Side-by-side comparison with concatenation |
| Token Reduction | >50% | Compare prompt sizes |
| Latency | <500ms added | Measure retrieval + rerank time |
| Cost | Acceptable | Embedding + query costs within budget |

---

## 9. Cost Analysis

### 9.1 One-Time Costs (Updated with Actual Sources)

| Item | Estimate | Calculation |
|:-----|:---------|:------------|
| **PDF embedding (15 books)** | **~$0.26** | 2M tokens × $0.00013/1k |
| Vector DB storage | ~55MB disk | 4,500 chunks × 3072 dims × 4 bytes |
| Development time | 20-30 hours | Pipeline + integration |

### 9.2 Ongoing Costs

| Item | Per Analysis | Monthly (30 analyses) |
|:-----|:-------------|:----------------------|
| Query embeddings | ~$0.002 | ~$0.06 |
| Reranker (if added) | $0.00 (local) | $0.00 |
| Vector DB | $0.00 (local) | $0.00 |

**Total additional cost: Negligible (~$0.06/month)**

> **Note:** One-time embedding cost of ~$0.26 is trivial. Re-embedding only needed if changing embedding model or adding new books.

---

## 10. Risk Assessment

### 10.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|:-----|:-----------|:-------|:-----------|
| Retrieval quality insufficient | Medium | High | Start simple, add reranker if needed |
| Chunking loses context | Medium | Medium | Use semantic + section-aware chunking |
| Vector DB corruption | Low | High | Regular backups, rebuild capability |
| Embedding model changes | Low | Medium | Store model version, rebuild if needed |

### 10.2 Operational Risks

| Risk | Likelihood | Impact | Mitigation |
|:-----|:-----------|:-------|:-----------|
| Increased complexity | High | Medium | Clear documentation, modular design |
| Debugging harder | Medium | Medium | Comprehensive logging, chunk tracing |
| New dependencies | High | Low | Pin versions, minimal dependencies |

### 10.3 PDF Parsing Risks

| Risk | Likelihood | Impact | Mitigation |
|:-----|:-----------|:-------|:-----------|
| Scanned PDFs (image-only) | Low | High | Use OCR fallback (pytesseract) if needed |
| Complex tables/charts | Medium | Medium | Extract as structured data or skip with flag |
| Non-English content | Low | Low | Most books are English; flag if detected |
| Corrupted PDFs | Low | Medium | Validate before ingestion, skip with error log |

---

## 11. Python Dependencies

### 11.1 Core Dependencies (requirements.txt)

```txt
# RAG Framework
llama-index>=0.10.0
llama-index-vector-stores-qdrant>=0.2.0
llama-index-embeddings-openai>=0.1.0

# Vector Database
qdrant-client>=1.7.0

# PDF Processing
pymupdf>=1.23.0           # Fast PDF parsing
unstructured>=0.12.0      # Structure detection (optional)

# Embeddings
openai>=1.0.0             # OpenAI API client
tiktoken>=0.5.0           # Token counting

# Utilities
python-dotenv>=1.0.0      # Environment variables
pyyaml>=6.0.0             # Configuration files
tqdm>=4.66.0              # Progress bars
```

### 11.2 Optional Dependencies

```txt
# Reranking (add in v1.1 if needed)
sentence-transformers>=2.2.0  # For local cross-encoder

# OCR fallback (if scanned PDFs detected)
pytesseract>=0.3.10
pdf2image>=1.16.0
```

### 11.3 Version Pinning Strategy

- Pin major.minor versions for stability
- Update quarterly or when security patches needed
- Test in isolation before upgrading

---

## 12. Implementation Checklist

### 12.1 Prerequisites

- [x] Confirm list of source books (PDFs) available — **15 PDFs confirmed in `/sources/`**
- [ ] Remove duplicate Porter book before ingestion
- [ ] Verify local Python environment (3.10+)
- [ ] Disk space for vector DB (~100MB estimated)
- [ ] OpenAI API key configured (for embeddings) — **same key used for Decision Engine**

### 12.2 Phase -1 Implementation

- [ ] Install LlamaIndex and Qdrant
- [ ] Implement PDF parser
- [ ] Implement semantic chunker
- [ ] Implement metadata enricher
- [ ] Implement embedding pipeline
- [ ] Implement Qdrant storage
- [ ] Create CLI tool
- [ ] Ingest all source books
- [ ] Validate index quality

### 12.3 Retrieval Layer Implementation

- [ ] Implement retrieval module
- [ ] Define engine-collection mapping
- [ ] Define query templates per engine
- [ ] Implement hybrid search (semantic + BM25)
- [ ] Implement context assembly
- [ ] (Optional) Implement reranker

### 12.4 Integration

- [ ] Update orchestrator
- [ ] Update engine prompts
- [ ] Update _meta schema
- [ ] Update JSON schemas
- [ ] Integration testing
- [ ] Performance benchmarking

### 12.5 Documentation

- [ ] Update PROJECT_SUMMARY
- [ ] Update journal
- [ ] Update architecture HTML
- [ ] Create knowledge pipeline README

---

## 13. Updated Directory Structure

```
stock-analysis/
├── CLAUDE.md
├── .env                           # Add: OPENAI_API_KEY (for embeddings)
│
├── knowledge_db/                  # NEW: Vector database storage
│   ├── qdrant/                    # Qdrant data files (embedded mode)
│   ├── manifests/                 # Source file hashes for versioning
│   └── snapshots/                 # For backups
│
├── sources/                       # Theory PDFs (EXISTING - 15 books)
│   ├── The Intelligent Investor - BENJAMIN GRAHAM.pdf
│   ├── security-analysis-benjamin-graham-6th-edition.pdf
│   ├── Competitive advantage - Michael E. Porter.pdf
│   ├── Geopolitical alpha - Marko Papic.pdf
│   ├── Misbehaving - Richard Thaler.pdf
│   └── ... (15 total, see Section 2)
│
├── scripts/
│   ├── knowledge_pipeline/        # NEW: Ingestion + retrieval pipeline
│   │   ├── __init__.py
│   │   ├── ingest.py              # Main ingestion entry point
│   │   ├── retrieval.py           # Retrieval layer for engines
│   │   ├── parsers/
│   │   │   ├── pdf_parser.py      # PyMuPDF + Unstructured.io
│   │   │   └── structured_parser.py
│   │   ├── chunkers/
│   │   │   ├── semantic_chunker.py
│   │   │   └── hybrid_chunker.py
│   │   ├── enrichers/
│   │   │   ├── metadata_extractor.py
│   │   │   └── category_classifier.py
│   │   ├── embedders/
│   │   │   └── openai_embedder.py
│   │   ├── storage/
│   │   │   └── qdrant_store.py
│   │   └── cli.py                 # Command-line interface
│   │
│   ├── data_compiler/             # Unchanged (FMP API)
│   ├── validate_schema.py
│   └── run_analysis.sh
│
├── schemas/
│   ├── _meta.schema.json          # UPDATE: Add knowledge_retrieval
│   ├── knowledge_chunk.schema.json # NEW: Chunk metadata schema
│   └── ...
│
├── config/
│   ├── collections.yaml           # NEW: Collection definitions
│   ├── engine_queries.yaml        # NEW: Query templates per engine
│   └── decision_rules.md
│
└── analyses/
    └── {TICKER}_{DATE}/
        └── ... (unchanged)
```

> **Note:** `sources/` folder already exists at `/Users/rajan/Documents/Projects/stockpicker/sources/` with 15 PDFs.

---

## 14. Decision Points for User

### 14.1 Required Decisions

| Decision | Options | Recommendation | Impact |
|:---------|:--------|:---------------|:-------|
| **Vector DB** | Qdrant vs Chroma | Qdrant (local) | Affects performance, filtering |
| **Embedding Model** | OpenAI vs Nomic | OpenAI text-embedding-3-large | Affects quality, cost |
| **Reranker** | None vs MiniLM | Start with None | Affects precision, latency |
| **Framework** | LlamaIndex vs custom | LlamaIndex | Affects dev time |

### 14.2 User Decisions (Resolved)

| Question | Decision | Notes |
|:---------|:---------|:------|
| **Source books** | 15 unique PDFs in `/sources/` | See Section 2 for full inventory |
| **Embedding model** | OpenAI text-embedding-3-large | User confirmed |
| **Existing playbooks** | None — starting fresh | Project is in planning stage |
| **Reranker** | Start without, add if needed | Default recommendation accepted |
| **Vector DB** | Qdrant (local) | Default recommendation accepted |
| **Framework** | LlamaIndex | Default recommendation accepted |

---

## 15. Summary of Architecture Changes

| Component | v4.2 | v5.0 (Proposed) |
|:----------|:-----|:----------------|
| **Phase -1** | Knowledge Curator (GPT → markdown) | Knowledge Ingestion Pipeline (PDF → Vector DB) |
| **Knowledge Storage** | Flat markdown files | Qdrant vector database |
| **Knowledge Injection** | Full playbook concatenation | Semantic retrieval (top-k chunks) |
| **Engine Prompts** | Static knowledge section | Dynamic retrieved context |
| **_meta Provenance** | Input file hashes | + Knowledge chunk IDs + retrieval queries |
| **New Dependencies** | None | LlamaIndex, Qdrant, sentence-transformers |
| **Token Usage** | High (full playbooks) | ~60-80% reduction |
| **Scalability** | Limited by context window | Unlimited knowledge base |

---

## 16. References

### Research & Best Practices
- [2025 Guide to RAG - Eden AI](https://www.edenai.co/post/the-2025-guide-to-retrieval-augmented-generation-rag)
- [Enhancing RAG: Best Practices Study (arXiv)](https://arxiv.org/abs/2501.07391)
- [Six Lessons Building RAG in Production - TDS](https://towardsdatascience.com/six-lessons-learned-building-rag-systems-in-production/)
- [Financial Report Chunking for RAG (arXiv)](https://arxiv.org/abs/2402.05131)
- [Optimizing RAG in Financial Services - Fitch Group](https://odsc.medium.com/optimizing-rag-pipelines-in-financial-services-advanced-strategies-from-fitch-group-cc3e81684817)

### Vector Databases
- [Vector DB Comparison 2025 - LiquidMetal AI](https://liquidmetal.ai/casesAndBlogs/vector-comparison/)
- [Best Vector Databases for RAG 2025 - Latenode](https://latenode.com/blog/ai-frameworks-technical-infrastructure/vector-databases-embeddings/best-vector-databases-for-rag-complete-2025-comparison-guide)

### Embedding Models
- [Best Embedding Models 2025 - Elephas](https://elephas.app/blog/best-embedding-models)
- [OpenAI vs Open-Source Embeddings - TigerData](https://www.tigerdata.com/blog/open-source-vs-openai-embeddings-for-rag)

### Frameworks
- [LlamaIndex vs LangChain - IBM](https://www.ibm.com/think/topics/llamaindex-vs-langchain)
- [LangChain vs LlamaIndex 2025 - Latenode](https://latenode.com/blog/platform-comparisons-alternatives/automation-platform-comparisons/langchain-vs-llamaindex-2025-complete-rag-framework-comparison)

### Chunking & Reranking
- [Best Chunking Strategies 2025 - Firecrawl](https://www.firecrawl.dev/blog/best-chunking-strategies-rag-2025)
- [Chunking Strategies - Weaviate](https://weaviate.io/blog/chunking-strategies-for-rag)
- [Guide to Reranking Models 2025 - ZeroEntropy](https://www.zeroentropy.dev/articles/ultimate-guide-to-choosing-the-best-reranking-model-in-2025)
- [Rerankers for RAG - Pinecone](https://www.pinecone.io/learn/series/rag/rerankers/)

---

## 17. Next Steps

### 17.1 Immediate Actions (Pre-Implementation)

1. **User Review:** Review this PRD and confirm architecture decisions
2. **Remove Duplicate:** Delete `Competitive advantage...Porter...(1).pdf` from sources
3. **Verify Environment:** Confirm Python 3.10+ and disk space available
4. **Confirm OpenAI Key:** Verify API key is ready for embedding calls

### 17.2 Implementation Sequence

Once approved, implementation proceeds in this order:

| Phase | Task | Dependency | Output |
|:------|:-----|:-----------|:-------|
| **1** | Set up project structure | PRD approval | `scripts/knowledge_pipeline/` scaffold |
| **2** | Implement PDF parser | Phase 1 | Working text extraction |
| **3** | Implement chunking | Phase 2 | Chunked documents |
| **4** | Implement Qdrant storage | Phase 3 | Indexed vectors |
| **5** | Implement retrieval layer | Phase 4 | Working retrieval |
| **6** | Integrate with one engine (Fundamentals) | Phase 5 | End-to-end validation |
| **7** | Roll out to all engines | Phase 6 | Full integration |
| **8** | Update documentation | Phase 7 | Updated PROJECT_SUMMARY, journal |

### 17.3 Approval Required

Before implementation begins:

- [ ] User confirms PRD is complete and accurate
- [ ] User approves technology choices (Qdrant, LlamaIndex, OpenAI embeddings)
- [ ] User approves collection structure and source-to-collection mapping

---

## 18. Document History

| Version | Date | Author | Changes |
|:--------|:-----|:-------|:--------|
| 1.0 | 2025-12-29 | Claude | Initial PRD created |
| 1.1 | 2025-12-29 | Claude | Added source inventory (15 PDFs), updated cost estimates, resolved user decisions, corrected directory structure |

---

*End of PRDChange-todo.md*
