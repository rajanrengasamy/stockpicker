# Project Journal: Stock Analysis Engine System

> **Purpose**: This file provides Claude with session continuity across conversations. 
> Read this file at the start of every new session to understand current state and context.

---

## Active Context (Updated: 2025-12-29 Late Evening AEST)

### Current Focus
**MAJOR ARCHITECTURE CHANGE: RAG + Vector Database integration planned.** User decided to replace static playbook concatenation with dynamic RAG-based knowledge retrieval. Comprehensive PRD created (`PRDChange-todo.md`) covering all aspects of the change. This upgrades the architecture from v4.2 to v5.0 (pending implementation). Source materials confirmed: 15 unique PDFs in `/sources/` folder. Awaiting user approval to begin implementation.

### Recent Decisions Still in Effect

**NEW (v5.0 — RAG Architecture):**
- **RAG for knowledge retrieval**: Replace playbook concatenation with semantic retrieval from vector database
- **Vector DB**: Qdrant (local embedded mode) — no Docker required
- **Embedding model**: OpenAI text-embedding-3-large (3072 dimensions) — user confirmed
- **RAG framework**: LlamaIndex — RAG-first design, built-in Qdrant integration
- **Chunking strategy**: Semantic + section-aware, 400-512 tokens, 10-15% overlap
- **Reranker**: None for V1; add cross-encoder in V1.1 if precision insufficient
- **Source materials**: 15 unique PDFs confirmed in `/sources/` (~121MB, ~2M tokens, ~4,500 chunks)
- **One-time embedding cost**: ~$0.26 (trivial)
- **8 knowledge collections**: core_principles, valuation, behavioral, governance, supply_chain, geopolitics, catalysts, fundamentals
- **Engine-specific retrieval**: Each engine queries relevant collections with weighted query templates
- **Knowledge provenance**: `_meta.knowledge_retrieval` tracks chunk_ids, queries, relevance_scores

**Carried Forward (v4.2):**
- **FMP Ultimate plan**: Purchased for 5-6 months — primary quantitative data source
- **Data Compiler**: Deterministic Python script with sector adapters — pulls FMP data, computes metrics + valuation bands
- **"Engines interpret, not calculate"**: Core architectural principle — LLMs never compute financial metrics
- **"No LLM-generated prices"**: Decision Engine outputs valuation zones (UNDERVALUED/FAIR/OVERVALUED), NOT target prices
- **Valuation bands**: Data Compiler computes peer_median_pe_implied, historical_5y_range, conservative_growth_implied
- **Provenance metadata**: ALL engine outputs include `_meta` object with model, prompt_hash, input_hashes, timestamp
- **Evidence registry**: Scout produces structured evidence objects; engines reference by ID (e.g., "E01")
- **Sector adapters**: Data Compiler routes through banks/reits/miners/insurers adapters; gaps flagged not approximated
- **Quality thresholds**: Explicit numeric conditions for PASS/WARN/FAIL defined
- **Decision on WARN**: Max verdict is WATCH; must include confidence_constraints and upgrade_conditions
- **Corporate actions**: Data Compiler logs price_adjustment method and corporate_actions_12m
- **API resilience**: Rate limiting with exponential backoff, 24-hour caching, checkpointing for resume
- **Scout-only web search**: Scout Engine is the ONLY component that searches the web (Perplexity API)
- **Schema validation gates**: JSON schema validation after each engine; fail-fast on validation errors
- **Quality Engine as GATE**: Can BLOCK Decision Engine if audit fails
- **Prompt injection guardrails**: Hard delimiters around all external content
- **Source hierarchy**: FMP (primary) → ASX → Company IR → Perplexity → Reuters/Bloomberg (demoted)
- **Enhanced peer scope**: Local (primary benchmarking) + Global (reference only) + Pure-play
- **Engine naming**: All agents called "Engines" with functional names
- **Investment horizon**: Long-only, 1-3 year holding period
- **Short selling**: Descoped from V1

**SUPERSEDED:**
- ~~No MCP, no RAG~~: **NOW USING RAG** — playbook concatenation replaced with vector retrieval

### Immediate Next Steps
1. **User reviews PRDChange-todo.md** → Confirm architecture is complete and accurate
2. **User approves technology choices** → Qdrant, LlamaIndex, OpenAI embeddings
3. **Remove duplicate Porter PDF** → `Competitive advantage...Porter...(1).pdf`
4. **Begin Phase -1 implementation** → Knowledge ingestion pipeline
5. Create `scripts/knowledge_pipeline/` with parsers, chunkers, embedders, storage
6. Ingest all 15 source PDFs into Qdrant
7. Implement retrieval layer for engine integration
8. Update orchestrator to use retrieval instead of concatenation

### Open Questions Awaiting Input
- Does user approve the PRD as complete and accurate?
- Any changes to collection structure or source-to-collection mapping?
- Confirm Python 3.10+ environment available

---

## Session History

### Session: 2025-12-29 Late Evening (Current) — RAG Architecture v5.0 Planning

#### Summary
User reviewed existing docs and decided to implement RAG (Retrieval Augmented Generation) with Vector Database for knowledge retrieval, replacing the previous "playbook concatenation" approach. Claude conducted extensive web research on 2025 best practices for RAG architectures, vector databases, embedding models, chunking strategies, and frameworks. Created comprehensive PRD change document (`PRDChange-todo.md`, 18 sections, ~1100 lines) covering all aspects of the migration. Analyzed user's source folder — found 15 unique investment theory PDFs totaling ~121MB. User confirmed OpenAI embeddings and that project is still in planning stage (no existing playbooks to migrate).

#### Research Conducted

| Topic | Key Findings | Sources |
|:------|:-------------|:--------|
| **RAG Architectures 2025** | Start simple (Basic RAG), layer in complexity; GraphRAG for knowledge graphs; Self-RAG reduces hallucinations 52% | Eden AI, arXiv 2501.07391, TDS |
| **Vector Databases** | Qdrant best for local deployment (Rust, embedded mode); Pinecone cloud-only (overkill); Chroma good for prototyping | LiquidMetal AI, Latenode |
| **Embedding Models** | OpenAI text-embedding-3-large highest quality; Nomic good open-source alternative; ~$0.00013/1k tokens | Elephas, TigerData |
| **Chunking Strategies** | Semantic chunking + section-aware best for documents; 400-512 tokens optimal; 10-15% overlap | Firecrawl, Weaviate, Snowflake |
| **Frameworks** | LlamaIndex RAG-first, simpler learning curve; LangChain more flexible for agents; Can use both together | IBM, Latenode |
| **Reranking** | Cross-encoder improves precision 20-35%; MiniLM good balance; Cohere higher quality but adds latency | ZeroEntropy, Pinecone |
| **Financial RAG** | Element-based chunking best for reports; metadata enrichment critical; Fitch Group production patterns | arXiv 2402.05131, CFA Institute |

#### Topics & Outcomes

| Topic | Decision | Rationale |
|:------|:---------|:----------|
| Replace playbook concatenation | **YES — Use RAG** | Reduces token usage 60-80%, enables semantic retrieval, scales to unlimited books |
| Vector Database | **Qdrant (local)** | Embedded mode (no Docker), excellent filtering, hybrid search, active development |
| Embedding Model | **OpenAI text-embedding-3-large** | Best quality, minimal cost (~$0.26 one-time), consistent with existing OpenAI usage |
| Framework | **LlamaIndex** | RAG-first design, built-in PDF parsing and Qdrant integration, simpler than LangChain |
| Chunking | **Semantic + section-aware** | Preserves context, respects document structure, 400-512 tokens with 10-15% overlap |
| Reranker | **None for V1** | Start simple, add cross-encoder/ms-marco-MiniLM in V1.1 if precision insufficient |
| Collections | **8 domains** | core_principles, valuation, behavioral, governance, supply_chain, geopolitics, catalysts, fundamentals |

#### Source Material Analysis

| Category | Books | Key Authors |
|:---------|:------|:------------|
| **Value Investing/Fundamentals** | Intelligent Investor, Security Analysis, Buffett Financial Statements | Graham, Buffett/Clark |
| **Behavioral Finance** | Misbehaving, Irrational Exuberance | Thaler, Shiller |
| **Competitive Strategy** | Competitive Advantage, Competitive Strategy | Porter |
| **Geopolitics** | Geopolitical Alpha, Prisoners of Geography | Papic, Marshall |
| **Investment Process** | Investment Checklist, Principles, Common Stocks | Shearn, Dalio, Fisher |
| **Economic/Markets** | Guide to Economic Indicators, Mapping the Markets | The Economist, Owen |

**Total: 15 unique books** (~121MB, ~2M tokens, ~4,500 chunks estimated)

**Issue Found:** Duplicate Porter book — `Competitive advantage...Porter...(1).pdf` should be removed before ingestion.

#### Deliverables Produced

| Artifact | Type | Status | Notes |
|:---------|:-----|:-------|:------|
| `PRDChange-todo.md` | PRD Document | ✅ Complete | 18 sections, ~1100 lines, comprehensive architecture change plan |
| Source inventory | Analysis | ✅ Complete | 15 PDFs catalogued with domains and collection mapping |
| Cost estimates | Analysis | ✅ Complete | ~$0.26 one-time, ~$0.06/month ongoing |
| Technology decisions | Decision | ✅ Complete | Qdrant + LlamaIndex + OpenAI embeddings |
| Implementation checklist | Planning | ✅ Complete | Prerequisites, phases, validation criteria |
| Python dependencies | Planning | ✅ Complete | requirements.txt with core and optional deps |

#### Key Architectural Changes (v4.2 → v5.0)

| Component | v4.2 | v5.0 (Proposed) |
|:----------|:-----|:----------------|
| **Phase -1** | Knowledge Curator (GPT → markdown) | **Knowledge Ingestion Pipeline** (PDF → Vector DB) |
| **Knowledge Storage** | Flat markdown files (`knowledge/*.md`) | **Qdrant vector database** (`knowledge_db/`) |
| **Knowledge Injection** | Full playbook concatenation | **Semantic retrieval** (top-k relevant chunks) |
| **Engine Prompts** | Static knowledge section | **Dynamic retrieved context** with source attribution |
| **_meta Provenance** | Input file hashes | **+ knowledge_retrieval** (chunk_ids, queries, scores) |
| **Token Usage** | High (full playbooks) | **~60-80% reduction** |
| **Scalability** | Limited by context window | **Unlimited knowledge base** |
| **New Dependencies** | None | LlamaIndex, Qdrant, sentence-transformers |

#### Outstanding Items

| Item | Priority | Owner | Next Action |
|:-----|:---------|:------|:------------|
| Review PRDChange-todo.md | 🔴 High | User | Confirm complete and accurate |
| Approve technology choices | 🔴 High | User | Qdrant, LlamaIndex, OpenAI embeddings |
| Remove duplicate Porter PDF | 🔴 High | User | Delete `...Porter...(1).pdf` |
| Confirm Python 3.10+ available | 🔴 High | User | Check environment |
| Begin implementation | 🔴 High | Claude | After PRD approved |
| Confirm FMP subscription active | 🟡 Med | User | Still needed for Data Compiler |
| Provide FMP API key | 🟡 Med | User | Add to .env |

---

### Session: 2025-12-29 Evening — Architecture v4.2

#### Summary
Received external architect review of v4.1 architecture. Claude performed critical analysis of the 6 recommendations, agreeing with 5 fully and deferring 1 (full structured evidence objects) to V2. Key outcomes: removed target_price from Decision Engine (replaced with valuation zones), added valuation_bands to Data Compiler, added provenance metadata standard, added evidence_registry to Scout, added sector adapters with explicit gap flagging, defined explicit Quality thresholds, defined Decision Engine behavior on WARN, added corporate actions logging, added API resilience features.

#### Topics & Outcomes

| Topic | Consultant Recommendation | Claude Assessment | Outcome |
|:------|:--------------------------|:------------------|:--------|
| Target price violates "no LLM math" | Remove target_price; Decision chooses among bands | ✅ **AGREE** — but consultant fix incomplete | Valuation zones + reference_bands; no target_price |
| Provenance and run metadata | Add model, prompt_hash, input_hashes, timestamps to all outputs | ✅ **STRONGLY AGREE** | `_meta` object on ALL outputs |
| Structured evidence objects | Full metadata per citation | ⚠️ **DEFER TO V2** — over-engineered for V1 | V1: evidence_registry with id, url, retrieved_at, category; V2: full metadata |
| Sector-specific metric gaps | FMP won't have all sector metrics; add adapters | ✅ **STRONGLY AGREE** | Modular sector adapters + gap flagging |
| Quality thresholds WARN vs FAIL | Define explicit conditions and Decision behavior | ✅ **AGREE — This was a gap** | Numeric thresholds defined; Decision max WATCH on WARN |
| Corporate actions | Track splits, dividends, adjustment methods | ✅ **AGREE** | price_adjustment + corporate_actions_12m in data_quality |

#### Claude's Additional Identified Gaps (Not in Consultant Feedback)

| Gap | Risk | Recommendation Adopted |
|:----|:-----|:-----------------------|
| Rate limiting / API quotas | FMP Ultimate has call limits; burst runs could fail | Retry with exponential backoff |
| Caching strategy | Re-running analysis for same stock wastes API calls | 24-hour TTL cache |
| Error recovery | Phase fails mid-run, lose all progress | Checkpointing with resume capability |

#### Key Architectural Changes (v4.1 → v4.2)

| Component | v4.1 | v4.2 |
|:----------|:-----|:-----|
| Decision Engine output | target_price ($115.00) | **valuation_zone + reference_bands** |
| Data Compiler valuation | Not computed | **valuation_bands (peer_median, historical, conservative)** |
| Output provenance | Not standardized | **_meta object on ALL outputs** |
| Scout citations | Free-form | **evidence_registry with IDs** |
| Data Compiler structure | Generic | **Sector adapters (banks, REITs, miners, insurers)** |
| Quality thresholds | Implicit | **Explicit numeric conditions** |
| Decision on WARN | Not specified | **Max verdict WATCH; must include upgrade_conditions** |
| Corporate actions | Not tracked | **price_adjustment + corporate_actions_12m** |
| API handling | Not specified | **Rate limiting, 24h cache, checkpointing** |

#### Deliverables Produced

| Artifact | Type | Status |
|:---------|:-----|:-------|
| `stock-analysis-architecture-v4_2.html` | Reference/Visual | ✅ Complete |
| `PROJECT_SUMMARY_2025-12-29_v5.md` | Documentation | ✅ Complete |
| `journal_2025-12-29_v5.md` | Documentation | ✅ Complete (this file) |
| Consultant feedback analysis | Analysis | ✅ Complete |
| v4.1 → v4.2 change mapping | Analysis | ✅ Complete |

#### Key Decisions & Rationale

| Decision | Rationale | Implications |
|:---------|:----------|:-------------|
| **Remove target_price from Decision Engine** | LLMs should not generate prices; violates "interpret, not calculate" | Decision outputs valuation_zone (UNDERVALUED/FAIR/OVERVALUED) + entry_thesis |
| **Add valuation_bands to Data Compiler** | Deterministic reference points replace LLM-generated prices | peer_median_pe_implied, historical_5y_range, conservative_growth_implied |
| **Add _meta provenance to all outputs** | Reproducibility + audit trail for investment decisions | model, prompt_hash, input_hashes, generated_at on every JSON |
| **Defer full evidence metadata to V2** | Extracting publisher, published_date requires additional web fetches | V1: id, url, retrieved_at, category_hint; V2: full metadata |
| **Sector adapters with gap flagging** | FMP doesn't have all sector metrics (CET1, AISC, etc.) | Adapters for banks/REITs/miners/insurers; gaps flagged in sector_gaps |
| **Explicit Quality thresholds** | Was a gap — implicit PASS/WARN/FAIL not operational | Numeric conditions defined (e.g., >1% metric mismatch = FAIL) |
| **Decision constrained on WARN** | Prevents overconfident outputs with thin evidence | Max verdict WATCH; must explain upgrade_conditions |
| **Corporate actions logging** | Multi-year return calculations affected by splits/dividends | price_adjustment method + corporate_actions_12m |
| **API resilience features** | Production robustness for re-runs and failures | Rate limiting, 24h cache, checkpointing |

#### Outstanding Items

| Item | Priority | Owner | Next Action |
|:-----|:---------|:------|:------------|
| Confirm FMP subscription active | 🔴 High | User | Verify subscription |
| Provide FMP API key | 🔴 High | User | Share key or add to .env |
| Provide theory PDFs | 🔴 High | User | Upload/share PDF files |
| Create Data Compiler with sector adapters | 🔴 High | Claude | After FMP confirmed |
| Create JSON schemas with _meta | 🔴 High | Claude | Can start now |
| Create Scout Engine with evidence_registry | 🔴 High | Claude | After schemas ready |
| Create Knowledge Curator prompts | 🔴 High | Claude | After PDFs received |
| Set up project folder | 🔴 High | User | After components ready |
| Define investment philosophy | 🟡 Med | User | For Decision Engine |
| Define decision rules/weights | 🟡 Med | User | For Decision Engine |

---

### Session: 2025-12-29 Morning — Architecture v4.1

#### Summary
Incorporated OpenAI ChatGPT architect P0/P1 recommendations into architecture. User decided to purchase FMP Ultimate plan for 5-6 months. This fundamentally changed the data strategy — FMP becomes primary quantitative source, enabling "engines interpret, not calculate" principle. Added Data Compiler as deterministic script. Consolidated all web search to Scout Engine only. Added JSON schema validation gates with fail-fast behavior. Added prompt injection guardrails. Enhanced peer scope to include global context. Added Knowledge Curator versioning and regression tests.

#### Topics & Outcomes

| Topic | Context | Conclusion |
|:------|:--------|:-----------|
| OpenAI P0-1: Deterministic data layer | LLMs should not calculate from narrative | Added Data Compiler (Python script, FMP API) |
| FMP Ultimate decision | User decided to pay for FMP | Primary quantitative data source; enables ASX coverage |
| OpenAI P0-2: Web search consistency | Mixed search (Scout + Catalyst + Geopolitics) is fragile | Scout Engine is now the ONLY search component |
| OpenAI P0-3: Source hardening | Paywalled outlets reduce reproducibility | FMP primary; Reuters/Bloomberg demoted to "nice-to-have" |
| OpenAI P0-4: Schema gating | No failure behavior defined | JSON schema validation gates; Quality can BLOCK Decision |
| OpenAI P0-5: Prompt injection | Concatenating web text is dangerous | Hard delimiters; "UNTRUSTED REFERENCE" wrapper |
| OpenAI P1-1: Peer scope fallback | "Local only" breaks for some sectors | Local (primary) + Global (reference) + Pure-play |
| OpenAI P1-2: Knowledge versioning | No regression checks | Added version metadata + regression test suite |

---

### Session: 2025-12-23

#### Summary
Incorporated recommendations from OpenAI ChatGPT architect review (initial). Major changes: renamed all agents to engines with functional names, added Knowledge Curator as new Phase -1, defined knowledge architecture with playbook structure and assignment matrix, added orchestrator knowledge selection, explicitly defined investment horizon (1-3 years, long-only), descoped short selling, added explicit news sourcing, upgraded Catalyst Engine to structured output, upgraded Quality Engine with audit methodology.

---

### Session: 2025-12-22 ~14:00-19:30 AEST

#### Summary
Comprehensive design session establishing the stock analysis agent architecture. Started with concept validation, evolved through V2 framework (17 sections), defined 8-agent architecture with GPT Decision Engine, then added Agent 0 (Discovery) using Perplexity API. Produced project summary v1/v2, architecture documentation v2/v3, and project journal v1/v2.

#### Key Decisions Made
- Perplexity API over Gemini CLI for Agent 0 (mandatory citations)
- `sonar-reasoning-pro` model for deep research
- Dual output (JSON + MD) for all agents
- Local peers only (no global) — *later updated in v4.1*
- All exchanges in country included
- No MCP, no RAG for V1
- GPT as Decision Engine (separate from analysis)

---

## Reference: Component Registry (v5.0 Proposed)

| Component | Type | Role | Platform | Output |
|:----------|:-----|:-----|:---------|:-------|
| **Knowledge Ingestion Pipeline** | **Script + LLM** | **PDF → Chunk → Embed → Index** | **Python + Qdrant + OpenAI** | **`knowledge_db/`** |
| **Knowledge Retrieval Layer** | **Script** | **Semantic search per engine query** | **Python + Qdrant** | **Retrieved context** |
| Scout Engine | LLM + Search | Discovery, peers, news, policy, evidence_registry (ONLY search) | Perplexity | `00_scout.json` |
| **Data Compiler** | **Script** | FMP data, sector adapters, valuation bands, corporate actions (NO LLM) | Python + FMP | `00_data_pack.json` |
| Fundamentals Engine | LLM + **Retrieval** | Interpret financials (§0-7) | Claude | `01_fundamentals.json` |
| Supply Chain Engine | LLM + **Retrieval** | Value chain (§11) | Claude | `02_supply_chain.json` |
| Contrarian Engine | LLM + **Retrieval** | Challenge consensus (§10,13,15) | Claude | `03_contrarian.json` |
| Geopolitics Engine | LLM + **Retrieval** | Policy & scenarios (§12) | Claude | `04_geopolitics.json` |
| Peer Benchmark Engine | LLM + **Retrieval** | Peer comparison | Claude | `05_peer_benchmark.json` |
| Governance Engine | LLM + **Retrieval** | Management (§8-9) | Claude | `06_governance.json` |
| Catalyst Engine | LLM + **Retrieval** | Event tracking (§14) | Claude | `07_catalysts.json` |
| Quality Engine | LLM | Audit with explicit thresholds + GATE function (NO retrieval) | Claude | `08_quality.json` |
| Decision Engine | LLM + **Retrieval** | Valuation zone verdict (no target prices) | GPT | `09_decision.json` |

**Total: 13 components** = 1 Knowledge Pipeline + 1 Retrieval Layer + 1 Scout + 1 Data Compiler (script) + 8 Claude engines + 1 Decision Engine.

**NEW:** All LLM engines (except Quality) now include retrieval step. All outputs include `_meta` provenance with `knowledge_retrieval` section.

---

## Reference: Phase Structure (v5.0 Proposed)

```
Phase -1: Knowledge Ingestion (one-time + incremental updates)
└── Knowledge Ingestion Pipeline (Python + LLM assist)
    └── Input: Theory PDFs (15 books in /sources/)
    └── Process: Parse → Chunk → Enrich → Embed → Index
    └── Output: knowledge_db/ (Qdrant vector database)
    └── Collections: core_principles, valuation, behavioral, governance,
                     supply_chain, geopolitics, catalysts, fundamentals

Phase 0: Discovery + Data (per-stock)
├── Step 0A: Scout Engine (Perplexity) — ONLY web search component
│   └── Input: Ticker → Output: 00_scout.json
│   └── Includes: peers, sector_lens, knowledge_packs, news, policy_context
│   └── evidence_registry (structured citations)
│
└── Step 0B: Data Compiler (Deterministic Python Script)
    └── Input: 00_scout.json → Output: 00_data_pack.json
    └── Source: FMP API with sector adapters
    └── Computes: All financial metrics, peer comparisons
    └── valuation_bands, corporate_actions_12m

Phase 1: Foundation (with RAG retrieval)
└── Fundamentals + Peer Benchmark (parallel)
    └── RETRIEVE: Query relevant knowledge from vector DB
    └── Consumes: 00_scout.json + 00_data_pack.json + retrieved_context
    └── Interprets data; NEVER calculates
    └── ALL outputs include _meta.knowledge_retrieval provenance

Phase 2: Context (with RAG retrieval)
└── Governance + Supply Chain + Catalyst (parallel)
    └── Each engine retrieves from its mapped collections
    └── Catalyst uses Scout's evidence_registry (no web search)

Phase 3: Analysis (with RAG retrieval)
└── Geopolitics → Contrarian (sequential)
    └── Geopolitics uses Scout's policy context (no web search)
    └── Contrarian retrieves behavioral/core_principles knowledge

Phase 4: Synthesis (GATED)
└── Quality → Decision (sequential)
    └── Quality: Explicit thresholds for PASS/WARN/FAIL (NO retrieval)
    └── Quality can BLOCK Decision if audit fails
    └── Decision: Retrieves core_principles + valuation for final synthesis
    └── Decision on WARN: Max verdict WATCH; must include upgrade_conditions
    └── Decision: Valuation zone verdict (no target_price)
```

---

## Reference: Quality Thresholds (v4.2)

| Condition | Result | Rationale |
|:----------|:-------|:----------|
| Any metric differs from data_pack by >1% | **FAIL** | Prevents hallucinated numbers |
| Required section missing | **FAIL** | Incomplete analysis |
| Evidence coverage <50% | **FAIL** | Insufficient support |
| Core financial data >90 days stale | **FAIL** | Outdated analysis |
| Evidence coverage 50-75% | **WARN** | Thin but acceptable |
| Data staleness 30-90 days | **WARN** | Usable but flag |
| Peer set <3 local comparables | **WARN** | Low confidence |
| Sector adapter flagged gaps | **WARN** | Missing context |

**Decision on WARN:** Max verdict WATCH; must include confidence_constraints and upgrade_conditions.

---

## Reference: Source Hierarchy (v4.2)

| Priority | Source | Type | Usage |
|:---------|:-------|:-----|:------|
| 1 | FMP API | Quantitative | Financial statements, ratios, metrics — **PRIMARY** |
| 2 | ASX Announcements | Official | Material disclosures, filings |
| 3 | Company IR | Official | Annual reports, presentations, transcripts |
| 4 | Regulators/Filings | Official | ASIC, RBA, regulatory calendars |
| 5 | Perplexity (Scout) | Qualitative | News, sector context, policy developments |
| 6 | Reuters/Bloomberg | Qualitative | Validation only — **NOT dependency** |

---

## Reference: Key v4.2 Changes Summary

| Change | Before (v4.1) | After (v4.2) |
|:-------|:--------------|:-------------|
| Decision output | target_price | valuation_zone + reference_bands |
| Valuation bands | Not computed | Data Compiler produces |
| Provenance | Not standardized | _meta on ALL outputs |
| Citations | Free-form | evidence_registry with IDs |
| Sector handling | Generic | Modular adapters + gap flagging |
| Quality thresholds | Implicit | Explicit numeric |
| Decision on WARN | Not specified | Max WATCH + upgrade_conditions |
| Corporate actions | Not tracked | price_adjustment + 12m actions |
| API resilience | Not specified | Rate limit, cache, checkpoint |

---

## Reference: Key Documents

| Document | Version | Location | Status |
|:---------|:--------|:---------|:-------|
| V2 Framework | — | `stock_analysisv2.md` | Foundational |
| Architecture Doc | v4.2 | `stock-analysis-architecture-v4_2.html` | Current (pre-RAG) |
| Project Summary | v5 | `PROJECT_SUMMARY_2025-12-29_v5.md` | Current (pre-RAG) |
| **RAG PRD Change** | **v1.1** | **`PRDChange-todo.md`** | **NEW — Pending Approval** |
| This Journal | v5 | `journal_2025-12-29_v5.md` | Current |

### Source Materials (for RAG ingestion)

| Location | Contents | Status |
|:---------|:---------|:-------|
| `/sources/` | 15 unique investment theory PDFs (~121MB) | Ready for ingestion |
| Duplicate to remove | `Competitive advantage...Porter...(1).pdf` | Pending deletion |

---

## Behavior Rules

- **Active Context**: ALWAYS update this section to reflect the latest state
- **Append, don't replace**: Add new session entries; never delete historical entries
- **Session order**: Most recent session appears directly under Active Context
- **Be specific**: Include file names, exact decisions, specific blockers
- **Assume fresh start**: Write as if Claude has zero memory beyond this document
- **Include "why"**: Decisions without rationale force re-discussion

---

## Version History

| Version | Date | Changes |
|:--------|:-----|:--------|
| v5 | 2025-12-29 | Architecture v4.2 with consultant feedback integration |
| v5.1 | 2025-12-29 | Added RAG architecture planning session (v5.0 proposed) |

---

*End of Journal v5.1*
