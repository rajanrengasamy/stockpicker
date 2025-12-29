# Stock Analysis Engine Project Summary

**Filename:** `PROJECT_SUMMARY_2025-12-29_v5.md`  
**Last Updated:** 2025-12-29 AEST  
**Version:** 5.0 (Architecture v4.2)  

---

## Quick Context (For Claude/Claude Code)

This document summarizes a stock analysis engine system being built by a retail investor. Read this first to understand the project scope, decisions made, and current status before proceeding with any tasks.

**What's New in v5 (Architecture v4.2):**
- **No LLM-generated prices** — Decision Engine outputs valuation zones, NOT target prices
- **Valuation bands** — Data Compiler computes deterministic reference bands (peer median implied, historical range, conservative implied)
- **Provenance metadata (_meta)** — ALL engine outputs include standardized provenance for reproducibility
- **Evidence registry** — Scout produces structured evidence objects with IDs; engines reference by ID
- **Sector adapters** — Data Compiler routes through modular adapters (banks, REITs, miners) with gap flagging
- **Explicit Quality thresholds** — Numeric conditions for PASS/WARN/FAIL defined
- **Decision behavior on WARN** — Max verdict is WATCH; must include confidence_constraints and upgrade_conditions
- **Corporate actions logging** — price_adjustment method and corporate_actions_12m tracked in data_pack
- **API resilience** — Rate limiting, 24-hour caching, checkpointing for resume

---

## 1. Project Overview

### 1.1 Goal
Build a multi-component system for **single-stock fundamental analysis** with peer comparison. The system should help identify investment opportunities using:
- Traditional fundamental analysis (valuation, growth, health, etc.)
- Supply chain investing / "Picks and Shovels" plays
- Second-level thinking (contrarian analysis)
- Geopolitics impact assessment

### 1.2 Investment Philosophy

| Attribute | Value |
|:----------|:------|
| **Horizon** | Long-only positions, 1–3 year holding period |
| **Style** | Fundamental analysis with qualitative overlays |
| **Edge Sources** | Longer time horizon, concentration, neglected stocks, second-order beneficiaries |
| **Out of Scope** | Short selling, options, day trading, portfolio optimization |

> "I know I can't beat the traders and people with insider information. But I can use information and techniques that are tested and available to get an edge."

The edge comes from:
- Longer time horizon than institutional traders (2–5 years vs quarters)
- Concentration on best ideas rather than diversification
- Finding neglected stocks and second-order beneficiaries
- Supply chain investing / "Picks and Shovels" plays
- Second-level thinking and contrarian analysis
- Patience to wait for opportunities

### 1.3 Scope

| Dimension | Current Scope | Future Scope |
|:----------|:--------------|:-------------|
| Analysis unit | Single stock | Portfolio |
| Comparison | Peer set (3-8 stocks) | Cross-portfolio |
| Decision | Stock-level conviction | Position sizing |
| Direction | Long-only | Long + Short |

---

## 2. Technical Architecture

### 2.1 System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PHASE -1: KNOWLEDGE CURATION                         │
│   Knowledge Curator (GPT executes, Claude creates prompts)              │
│   Input: Theory PDFs → Output: Versioned playbooks in knowledge/        │
│   Runs: One-time setup + periodic updates                               │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    PHASE 0: DISCOVERY + DATA                            │
│                                                                         │
│   Step 0A: Scout Engine (Perplexity API) — ONLY WEB SEARCH              │
│   Input: Ticker → Output: 00_scout.json                                 │
│   Produces: Peers, sector_lens, knowledge_packs, news, policy,          │
│             evidence_registry (structured citations)           ← NEW    │
│                              │                                          │
│                              ▼                                          │
│   Step 0B: Data Compiler (Deterministic Python Script)                  │
│   Input: 00_scout.json → Output: 00_data_pack.json                      │
│   Source: FMP API (Ultimate plan) with sector adapters         ← NEW    │
│   Computes: All metrics, peer comparisons, valuation_bands     ← NEW    │
│   Logs: corporate_actions_12m, price_adjustment method         ← NEW    │
│                                                                         │
│   ⚠️ SCHEMA VALIDATION GATE — Fail-fast on invalid output               │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      PHASE 1: FOUNDATION                                │
│   Fundamentals Engine + Peer Benchmark Engine (Claude)                  │
│   Parallel execution, consumes Scout + Data Compiler outputs            │
│   INTERPRETS data; NEVER calculates metrics                             │
│   ALL outputs include _meta provenance                         ← NEW    │
│                                                                         │
│   ⚠️ SCHEMA VALIDATION GATE                                             │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       PHASE 2: CONTEXT                                  │
│   Governance Engine + Supply Chain Engine + Catalyst Engine             │
│   Parallel execution, uses Phase 0+1 context                            │
│   Catalyst uses Scout's evidence_registry (no web search)               │
│                                                                         │
│   ⚠️ SCHEMA VALIDATION GATE                                             │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      PHASE 3: ANALYSIS                                  │
│   Geopolitics Engine → Contrarian Engine (Claude)                       │
│   Sequential: Geopolitics uses Scout's policy context                   │
│   Contrarian reads all prior outputs                                    │
│                                                                         │
│   ⚠️ SCHEMA VALIDATION GATE                                             │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      PHASE 4: SYNTHESIS                                 │
│   Quality Engine → Decision Engine (GPT)                                │
│                                                                         │
│   Quality Engine: Audit with EXPLICIT NUMERIC THRESHOLDS       ← NEW    │
│   Can output: PASS / WARN / FAIL                                        │
│                              │                                          │
│                    ┌─────────┴─────────┐                                │
│                    ▼                   ▼                                │
│               [PASS]              [WARN]              [FAIL]            │
│                 │                   │                   │               │
│                 ▼                   ▼                   ▼               │
│          Decision Engine    Decision Engine      Decision BLOCKED       │
│          (normal)           (constrained)        (returns error)        │
│          Any verdict        Max: WATCH ← NEW                            │
│                             Must include:                               │
│                             - confidence_constraints                    │
│                             - upgrade_conditions                        │
│                                                                         │
│   Decision Engine outputs: valuation_zone (NOT target_price)   ← NEW    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Core Architectural Principles

> **"Engines interpret, they do not calculate."**

All financial metrics, ratios, valuation bands, and peer comparisons are computed deterministically by the Data Compiler using FMP data. LLM engines receive pre-computed metrics and focus on:
- Interpretation and pattern recognition
- Qualitative analysis
- Narrative construction
- Risk identification
- Contrarian thinking

> **"No LLM-generated prices."**

Decision Engine outputs valuation zones (UNDERVALUED / FAIR / OVERVALUED) and entry thesis, referencing Data Compiler's valuation bands. It does NOT output target prices.

### 2.3 Technical Constraints (User Decisions)

| Constraint | Decision | Rationale |
|:-----------|:---------|:----------|
| **Primary Data Source** | FMP API (Ultimate plan) with sector adapters | Structured, reliable, ASX coverage, eliminates LLM calculation |
| **Knowledge Curator** | GPT (execution), Claude (prompt creation) | GPT's thinking models excel at structured extraction |
| **Web Search** | Scout Engine ONLY (Perplexity API) | Consistent citations, no environment-dependent failures |
| **Analysis engines** | Claude (via Claude Code) | Strong reasoning for interpretation tasks |
| **Decision Engine** | GPT with valuation zones (no target prices) | Separation of analysis vs decision; LLMs don't generate prices |
| **Orchestration** | CLI invocation with validation gates + checkpointing | Simplicity + fail-fast + resumability |
| **Knowledge base** | No RAG/Vector DB | Versioned playbooks injected via prompt concatenation |
| **Output format** | JSON (validated with _meta) + MD files | JSON for engine handoffs with provenance; MD for human review |
| **Scope** | Single stock + peers | Portfolio features deferred |
| **Peer scope** | Local (primary) + Global (reference) | Local for metrics; global flagged for context only |
| **Investment horizon** | Long-only, 1-3 years | Aligns with retail investor edge |
| **Security** | Hard delimiters for external content | Prompt injection guardrails |
| **Provenance** | _meta object on ALL outputs | Reproducibility + audit trail |

### 2.4 Tools & Services

| Service | Plan | Purpose |
|:--------|:-----|:--------|
| FMP API | Ultimate (~$149/mo) | Primary quantitative data source |
| Perplexity API | Pro | Scout Engine web search |
| Claude | Max (via Claude Code) | Analysis engines |
| OpenAI GPT | Pro | Knowledge Curator + Decision Engine |

---

## 3. Provenance Metadata Standard (NEW in v4.2)

### 3.1 Purpose
Every engine output includes a standardized `_meta` object for reproducibility and audit trails. When a verdict changes, you can determine exactly why—did the data change? The prompt? The model?

### 3.2 Required _meta Fields

| Field | Description |
|:------|:------------|
| **engine_name** | Which engine produced this output |
| **engine_version** | Semantic version (e.g., 1.0.0) |
| **model** | LLM model identifier (or "deterministic" for Data Compiler) |
| **temperature** | Temperature setting (0.0 for deterministic) |
| **prompt_hash** | SHA-256 of the prompt template used |
| **input_hashes** | SHA-256 of each input file |
| **generated_at** | ISO 8601 timestamp |

### 3.3 _meta Schema Example

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
    "generated_at": "2025-12-29T14:30:00Z"
  }
}
```

---

## 4. Data Compiler: Deterministic Metrics (UPDATED in v4.2)

### 4.1 Purpose
The Data Compiler is a **deterministic Python script** (NOT an LLM) that pulls structured financial data from FMP API, routes through sector adapters, computes standardized metrics, produces valuation reference bands, and logs corporate actions.

### 4.2 Configuration

| Attribute | Value |
|:----------|:------|
| **Type** | Deterministic Python script |
| **Data Source** | FMP API (Ultimate plan) |
| **Input** | `00_scout.json` (ticker + peers) |
| **Output** | `00_data_pack.json` |
| **Runs** | Phase 0, after Scout, before all engines |
| **Caching** | 24-hour TTL for re-runs |
| **Rate Limiting** | Retry with exponential backoff |

### 4.3 What Data Compiler Produces (v4.2)

```json
{
  "_meta": {
    "engine_name": "data_compiler",
    "engine_version": "1.0.0",
    "model": "deterministic",
    "generated_at": "2025-12-29T10:30:00Z"
  },
  "target_company": {
    "profile": {...},
    "financials": {
      "income_statement": [...],
      "balance_sheet": [...],
      "cash_flow": [...]
    },
    "ratios": {...},
    "key_metrics": {...}
  },
  "peer_companies": [...],
  "computed_metrics": {
    "valuation_vs_peers": {...},
    "growth_comparison": {...},
    "profitability_ranking": {...}
  },
  "valuation_bands": {
    "peer_median_pe_implied": 112.50,
    "historical_5y_range": [95.00, 125.00],
    "conservative_growth_implied": 105.00
  },
  "data_quality": {
    "completeness_score": 0.95,
    "sector_gaps": [
      {
        "metric": "CET1_ratio",
        "reason": "Regulatory capital not available via FMP",
        "alternative": "Check company investor presentation"
      }
    ],
    "price_adjustment": {
      "method": "split_and_dividend_adjusted",
      "source": "FMP historical-price-full",
      "last_adjustment_date": "2024-06-15"
    },
    "corporate_actions_12m": [
      {
        "date": "2024-06-15",
        "type": "dividend",
        "detail": "Final dividend $2.15 per share"
      }
    ]
  }
}
```

### 4.4 Valuation Bands (NEW in v4.2)

Data Compiler computes deterministic valuation reference points that Decision Engine can reference—replacing LLM-generated target prices.

| Band | Calculation | Usage |
|:-----|:------------|:------|
| **peer_median_pe_implied** | Current EPS × Peer median P/E | Relative valuation reference |
| **historical_5y_range** | 5-year P/E range × Current EPS | Historical context |
| **conservative_growth_implied** | DCF with conservative assumptions | Downside reference |

### 4.5 Sector Adapters (NEW in v4.2)

Modular adapters handle sector-specific metrics. When gaps exist, they're flagged rather than approximated.

```
data_compiler/
├── core.py                 # Base FMP data pull + standard ratios
├── adapters/
│   ├── banks.py            # NIM proxy, cost-to-income, loan growth
│   ├── reits.py            # FFO approximation, NTA
│   ├── miners.py           # Stub — flags missing AISC, reserve life
│   ├── insurers.py         # Stub — flags combined ratio gaps
│   └── general.py          # Default fallback
└── sector_detector.py      # Routes to appropriate adapter via GICS
```

### 4.6 Sector Gap Reality Check

| Metric Category | Can Derive from FMP? | Notes |
|:----------------|:---------------------|:------|
| **Banks: NIM** | ⚠️ Approximate only | Interest income / average interest-earning assets |
| **Banks: CET1** | ❌ No | Regulatory capital disclosure; not in standard financials |
| **Banks: Cost-to-Income** | ✅ Yes | Operating expenses / operating income |
| **REITs: FFO** | ⚠️ Approximate only | Net income + depreciation - gains on sale |
| **Miners: AISC** | ❌ No | All-in sustaining cost is company-reported |
| **Miners: Reserve Life** | ❌ No | Requires reserves data from filings |
| **Insurers: Combined Ratio** | ❌ No | Insurance-specific disclosure |

---

## 5. Scout Engine: Discovery & Evidence Registry

### 5.1 Purpose
Scout Engine is the **ONLY component that performs web search**. It now produces a structured evidence registry that downstream engines reference by ID.

### 5.2 Configuration

| Attribute | Value |
|:----------|:------|
| **API** | Perplexity |
| **Model** | `sonar-reasoning-pro` |
| **Input** | Stock ticker (e.g., `CBA.AX`) |
| **Output** | `00_scout.json` + `00_scout.md` |
| **Citations** | Mandatory (structured registry) |
| **Web Search** | EXCLUSIVE to Scout (no other engine searches) |

### 5.3 Evidence Registry (NEW in v4.2)

```json
"evidence_registry": [
  {
    "id": "E01",
    "url": "https://asx.com.au/...",
    "retrieved_at": "2025-12-29T10:00:00Z",
    "category": "filing",
    "snippet": "CBA reported FY24..."
  },
  {
    "id": "E02",
    "url": "https://commbank.com.au/...",
    "retrieved_at": "2025-12-29T10:00:00Z",
    "category": "IR",
    "snippet": "Investor presentation..."
  }
]
```

### 5.4 Category Classification (V1)

| Category | Domain Pattern | Examples |
|:---------|:---------------|:---------|
| **filing** | asx.com.au, sec.gov | ASX announcements, SEC filings |
| **IR** | */investor-relations/*, */investors/* | Company presentations, reports |
| **regulator** | rba.gov.au, asic.gov.au | Policy announcements |
| **news** | reuters.com, afr.com | News articles |
| **other** | Default fallback | Analyst reports, forums |

### 5.5 Enhanced Peer Identification

| Category | Criteria | Count | Usage |
|:---------|:---------|:------|:------|
| **Local Comparables** | Same primary business, same country, similar market cap (0.5x-2x), all exchanges | 3-8 | Primary benchmarking |
| **Adjacent Peers** | Same broad industry, different segment or model, same country | 3-5 | Context, moat analysis |
| **Global Context** | Same business model, different country (flagged) | 2-3 | Reference only, NOT for metrics |
| **Pure-Play** | Closest business model match regardless of size | 1-2 | Business model analysis |

---

## 6. Schema Validation & Quality Thresholds (UPDATED in v4.2)

### 6.1 Quality Engine Thresholds

| Condition | Result | Rationale |
|:----------|:-------|:----------|
| Any metric in engine output differs from `data_pack` by >1% | **FAIL** | Prevents hallucinated numbers |
| Required section missing (per engine schema) | **FAIL** | Incomplete analysis |
| Evidence coverage <50% of major claims | **FAIL** | Insufficient support |
| Core financial data >90 days stale | **FAIL** | Outdated analysis |
| Evidence coverage 50-75% | **WARN** | Thin but acceptable |
| Data staleness 30-90 days | **WARN** | Usable but flag |
| Peer set <3 local comparables | **WARN** | Low confidence benchmarking |
| Sector adapter flagged gaps | **WARN** | Missing sector-specific context |

### 6.2 Decision Engine Behavior by Audit Result

| Audit Result | Decision Engine Behavior | Max Verdict | Required Fields |
|:-------------|:-------------------------|:------------|:----------------|
| **PASS** | Normal execution | BUY / WATCH / AVOID | Standard output |
| **WARN** | Constrained execution | **WATCH** (max) | Must include `confidence_constraints` and `upgrade_conditions` |
| **FAIL** | BLOCKED — refuses to run | BLOCKED | Returns error with blocking issues |

---

## 7. Decision Engine Output Schema (UPDATED in v4.2)

Decision Engine synthesizes all inputs and outputs a valuation zone verdict. It does NOT output target prices.

```json
{
  "_meta": {...},
  "ticker": "CBA.AX",
  "analysis_date": "2025-12-29",
  "verdict": "BUY" | "WATCH" | "AVOID" | "BLOCKED",
  "conviction_score": 7.5,
  "investment_horizon": "1-3 years",

  "valuation_assessment": {
    "zone": "FAIR",
    "current_price": 108.50,
    "reference_bands": {
      "peer_median_implied": 112.50,
      "historical_range": [95.00, 125.00],
      "conservative_implied": 105.00
    },
    "entry_thesis": "Trading near peer median; attractive entry below $100"
  },

  "score_breakdown": {
    "fundamentals": { "score": 8, "weight": 0.25 },
    "valuation": { "score": 5, "weight": 0.20 },
    "moat_quality": { "score": 9, "weight": 0.20 },
    "supply_chain_edge": { "score": 8, "weight": 0.15 },
    "geopolitics_risk": { "score": 6, "weight": 0.10 },
    "governance": { "score": 8, "weight": 0.10 }
  },

  "red_flags": [...],
  "falsifiers": [...],
  "monitoring_triggers": {...},
  "evidence_ids": ["E01", "E05", "E12"],

  "confidence_constraints": {
    "reasons": ["Limited peer comparables", "Data 60 days stale"],
    "upgrade_conditions": ["Wait for half-year results", "Add MQG.AX to peer set"]
  }
}
```

**Key Change:** `scenarios.target_price` removed. Replaced with `valuation_assessment.zone` and `reference_bands` that point to Data Compiler outputs. LLMs do not generate prices.

---

## 8. API Resilience & Operational Robustness (NEW in v4.2)

### 8.1 Rate Limiting

| Service | Limit | Handling |
|:--------|:------|:---------|
| FMP Ultimate | 750 calls/min | Retry with exponential backoff |
| Perplexity Pro | Rate limits vary | Respect 429 responses |
| Claude API | Token limits | Batch if needed |
| OpenAI API | Standard limits | Standard handling |

### 8.2 Caching Strategy

- FMP data cached with 24-hour TTL
- Scout output cached per ticker per day
- Cache keyed by ticker + date
- Force refresh with `--no-cache` flag

### 8.3 Checkpointing

```json
{
  "last_completed_phase": 2,
  "completed_engines": ["scout", "data_compiler", "fundamentals", "peer_benchmark", "governance"],
  "failed_at": "supply_chain",
  "resume_from": "supply_chain"
}
```

- If Phase 2 fails mid-run, resume from failed engine
- Completed outputs preserved
- `run_analysis.sh --resume` flag

---

## 9. Component Registry

### 9.1 Complete Component List

| Component | Type | Role | Platform | Output |
|:----------|:-----|:-----|:---------|:-------|
| **Knowledge Curator** | LLM | Extract theory → versioned playbooks | GPT | `knowledge/*.md` |
| **Scout Engine** | LLM + Search | Discovery, peers, news, policy, evidence registry (ONLY search) | Perplexity API | `00_scout.json` |
| **Data Compiler** | **Script** | FMP data, sector adapters, valuation bands, corporate actions (NO LLM) | Python + FMP | `00_data_pack.json` |
| **Fundamentals Engine** | LLM | Interpret financials (§0-7) | Claude | `01_fundamentals.json` |
| **Supply Chain Engine** | LLM | Value chain (§11) | Claude | `02_supply_chain.json` |
| **Contrarian Engine** | LLM | Challenge consensus (§10,13,15) | Claude | `03_contrarian.json` |
| **Geopolitics Engine** | LLM | Policy & scenarios (§12) | Claude | `04_geopolitics.json` |
| **Peer Benchmark Engine** | LLM | Peer comparison | Claude | `05_peer_benchmark.json` |
| **Governance Engine** | LLM | Management (§8-9) | Claude | `06_governance.json` |
| **Catalyst Engine** | LLM | Event tracking (§14) | Claude | `07_catalysts.json` |
| **Quality Engine** | LLM | Audit with explicit thresholds + GATE function | Claude | `08_quality.json` |
| **Decision Engine** | LLM | Valuation zone verdict (no target prices) | GPT | `09_decision.json` |

**Total: 12 components** = 1 Knowledge Curator + 1 Scout + 1 Data Compiler (script) + 8 Claude engines + 1 Decision Engine. All outputs include `_meta` provenance.

---

## 10. Project Directory Structure

```
stock-analysis/
├── CLAUDE.md                      ← Auto-read by Claude Code
├── .env                           ← API keys (FMP, Perplexity, OpenAI)
│
├── knowledge/                      ← Versioned theory playbooks
│   ├── core/
│   │   └── core_principles.md
│   ├── valuation/
│   │   ├── common.md
│   │   └── sector_*.md
│   ├── industry/
│   ├── behavioral/
│   ├── governance/
│   ├── geopolitics/
│   ├── catalysts/
│   └── methodology/
│
├── schemas/                        ← JSON validation schemas
│   ├── _meta.schema.json          ← NEW: Provenance metadata
│   ├── scout_output.schema.json
│   ├── data_pack.schema.json
│   ├── fundamentals.schema.json
│   └── *.schema.json
│
├── prompts/
│   ├── knowledge_curator_prompt.md
│   ├── scout_engine.md
│   ├── fundamentals_engine.md
│   ├── supply_chain_engine.md
│   ├── contrarian_engine.md
│   ├── geopolitics_engine.md
│   ├── peer_benchmark_engine.md
│   ├── governance_engine.md
│   ├── catalyst_engine.md
│   ├── quality_engine.md
│   └── decision_engine.md
│
├── scripts/
│   ├── scout_engine.sh
│   ├── data_compiler/               ← NEW: Modular structure
│   │   ├── core.py
│   │   ├── adapters/
│   │   │   ├── banks.py
│   │   │   ├── reits.py
│   │   │   ├── miners.py
│   │   │   └── general.py
│   │   └── sector_detector.py
│   ├── validate_schema.py
│   ├── wrap_external.sh
│   └── run_analysis.sh               ← Orchestrator with gates + checkpointing
│
├── config/
│   └── decision_rules.md              ← Your investment philosophy
│
├── cache/                          ← NEW: API response cache
│   └── {TICKER}_{DATE}.json
│
├── analyses/
│   └── {TICKER}_{DATE}/
│       ├── .checkpoint                  ← NEW: Resume state
│       ├── 00_scout.json
│       ├── 00_scout.md
│       ├── 00_data_pack.json
│       ├── 01_fundamentals.json
│       ├── 02_supply_chain.json
│       ├── 03_contrarian.json
│       ├── 04_geopolitics.json
│       ├── 05_peer_benchmark.json
│       ├── 06_governance.json
│       ├── 07_catalysts.json
│       ├── 08_quality.json
│       ├── 09_decision.json
│       └── 10_full_report.md
│
├── source_books/                   ← Your theory PDFs
│   └── *.pdf
│
├── tests/
│   └── regression_suite.py
│
└── reference/
    ├── stock_analysisv2.md
    ├── journal_*.md
    └── PROJECT_SUMMARY_*.md
```

---

## 11. Implementation Sequence

Build incrementally. Validate each step before proceeding. Data Compiler with sector adapters is on the critical path.

| Step | Task | Dependencies | Priority | Status |
|:-----|:-----|:-------------|:---------|:-------|
| 1 | Project Setup — folder structure, CLAUDE.md, .env | — | 🔴 | ⬜ |
| 2 | FMP API Validation — test connection, verify ASX coverage | Step 1 | 🔴 | ⬜ |
| 3 | JSON Schemas with _meta — validation schemas including provenance | Step 1 | 🔴 | ⬜ |
| 4 | Data Compiler with Sector Adapters + Valuation Bands | Step 2 | 🔴 | ⬜ |
| 5 | Knowledge Curator — prompts + GPT extraction | Theory PDFs | 🔴 | ⬜ |
| 6 | Scout Engine with Evidence Registry | Step 1 | 🔴 | ⬜ |
| 7 | Orchestrator with Gates + Checkpointing | Steps 3-6 | 🔴 | ⬜ |
| 8 | Fundamentals Engine | Step 7 | 🔴 | ⬜ |
| 9 | Peer Benchmark Engine | Step 7 | 🔴 | ⬜ |
| 10 | Supply Chain Engine | Steps 8-9 | 🟡 | ⬜ |
| 11 | Geopolitics Engine | Step 10 | 🟡 | ⬜ |
| 12 | Governance Engine | Steps 8-9 | 🟡 | ⬜ |
| 13 | Catalyst Engine | Steps 8-9 | 🟡 | ⬜ |
| 14 | Contrarian Engine | Steps 10-13 | 🟡 | ⬜ |
| 15 | Quality Engine with Explicit Thresholds | Steps 8-14 | 🔴 | ⬜ |
| 16 | Decision Engine (Valuation Zones) | Step 15 | 🔴 | ⬜ |
| 17 | Regression Test Suite | Steps 5, 16 | 🟡 | ⬜ |

**First Milestone:** Data Compiler produces valid `00_data_pack.json` for CBA.AX with complete financials, peer metrics, valuation bands, and corporate actions from FMP.

**Second Milestone:** Scout Engine produces valid `00_scout.json` with exhaustive peer list, news, policy context, and structured evidence registry.

---

## 12. Open Items & Decisions Needed

| Item | Status | Priority | Notes |
|:-----|:-------|:---------|:------|
| Confirm FMP subscription active | ⬜ Pending | 🔴 High | Required for Data Compiler |
| Provide FMP API key | ⬜ Pending | 🔴 High | Add to .env |
| Provide theory PDFs | ⬜ Pending | 🔴 High | For Knowledge Curator |
| Create Data Compiler script with adapters | ⬜ Not started | 🔴 High | After FMP confirmed |
| Create JSON schemas with _meta | ⬜ Not started | 🔴 High | All engine outputs |
| Create Knowledge Curator prompts | ⬜ Not started | 🔴 High | After PDFs received |
| Investment philosophy | ⬜ Needed | 🟡 Medium | User must define |
| Decision rules / weights | ⬜ Needed | 🟡 Medium | For Decision Engine |

---

## 13. Change Log

| Date | Version | Changes |
|:-----|:--------|:--------|
| 2025-12-22 | v1.0 | Initial project summary. Established: architecture (4 phases), agent mapping, execution phases, directory structure. |
| 2025-12-22 | v2.0 | Added Agent 0 (Discovery) specification. Expanded to 5 phases. Added Perplexity API as discovery tool. Defined peer identification rules. Added sector-aware guidance matrix. |
| 2025-12-23 | v3.0 | Renamed all agents to engines. Added Knowledge Curator (Phase -1). Added knowledge architecture. Defined investment horizon (1-3 years, long-only). Descoped short selling. Expanded to 6 phases. |
| 2025-12-29 | v4.0 | Major update incorporating OpenAI P0/P1 feedback + FMP decision: Added Data Compiler (deterministic script, FMP API). Consolidated web search to Scout only. Added JSON schema validation gates. Added Quality Engine as GATE. Added prompt injection guardrails. Hardened source hierarchy. Enhanced peer scope. Added Knowledge versioning + regression tests. Component count: 11 → 12. |
| 2025-12-29 | **v5.0** | **Incorporated external architecture review feedback:** Removed target_price from Decision Engine (now valuation zones). Added valuation_bands to Data Compiler. Added _meta provenance to ALL outputs. Added evidence_registry to Scout. Added sector adapters with gap flagging. Added explicit Quality thresholds (PASS/WARN/FAIL). Defined Decision behavior on WARN. Added corporate actions logging. Added API resilience (rate limiting, caching, checkpointing). |

---

## 14. Reference Documents

| Document | Description | Location |
|:---------|:------------|:---------|
| V2 Framework | 17-section analysis template | `stock_analysisv2.md` |
| Architecture Doc v4.2 | Visual HTML reference | `stock-analysis-architecture-v4_2.html` |
| Project Journal v5 | Session continuity | `journal_2025-12-29_v5.md` |
| This Summary | Project context | `PROJECT_SUMMARY_2025-12-29_v5.md` |

---

## 15. Next Session Prompt

When starting a new conversation, upload this file and use:

> "I'm continuing work on a stock analysis engine project. I've attached the project summary v5 (architecture v4.2). Please review it and confirm you understand the architecture including the Data Compiler with sector adapters and valuation bands, provenance metadata, evidence registry, explicit Quality thresholds, and Decision Engine valuation zones. Then let's proceed with [SPECIFIC TASK]."

**Immediate next tasks:**
1. User confirms FMP subscription active
2. User provides FMP API key
3. User provides theory PDFs
4. Claude creates Data Compiler specification with sector adapters
5. Claude creates JSON schemas with _meta provenance
6. Claude creates Knowledge Curator prompts
7. Project folder setup

---

*End of Project Summary v5*
