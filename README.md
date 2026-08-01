# Agentic document intelligence over real SEC filings

An implementation of [MongoDB's published FSI reference architecture](https://www.mongodb.com/company/blog/innovation/unlocking-financial-services-document-intelligence-with-agentic-ai-mongodb) — supervisor multi-agent system, agentic RAG, MongoDB Atlas Vector Search — applied to real financial filings, not a synthetic demo corpus.

An analyst asks a question in natural language:

> *"How has this company's debt position changed over the last three years, and what did management say about it?"*

The system decides what to retrieve, pulls from multiple filings across multiple years, reconciles the numbers against the narrative, and answers **with citations to the exact filing, section, and page**. When the filings don't support an answer, it says so — and that refusal rate is measured, not hidden.

> **Status: in development.** The measurement table is the deliverable.

---

## 1. Why this project

**The architecture is validated, not invented.** MongoDB published this blueprint for exactly this use case — company credit rating, KYC onboarding, loan origination, investment research. Implementing a vendor reference architecture is a claim a client can verify, unlike "I designed an agent system."

**The data is unlimited, free, and public domain.** [SEC EDGAR](https://www.sec.gov/edgar) — every 10-K, 10-Q, and 8-K ever filed, no licensing risk, no PII, no synthesis credibility problem.

**Financial tables are genuinely hard.** Naive parsers break on them routinely. Extraction quality here is measured against XBRL ground truth, not asserted.

**Numbers come from XBRL, not from an LLM reading a table.** SEC filings ship machine-readable structured financial facts alongside the prose. Using XBRL for figures and reserving the LLM for narrative interpretation is both more accurate and more defensible — and it makes a **hallucinated-figure rate** a real, checkable number instead of a hope.

---

## 2. System overview

```
                          ┌───────────────────────┐
   Analyst question ─────►│   SUPERVISOR AGENT    │
   (chat or voice)        │   goal → delegation   │
                          └───────────┬───────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐         ┌───────────────────┐         ┌────────────────────┐
│  INGESTION    │         │  DOCUMENT         │         │   ANALYSIS         │
│  PIPELINE     │         │  ASSISTANT        │         │   AGENTS           │
│  (offline)    │         │  (agentic RAG)    │         │                    │
├───────────────┤         ├───────────────────┤         ├────────────────────┤
│ Scanner       │         │ decides whether   │         │ Financial extractor│
│  find filings │         │ to retrieve       │         │  XBRL facts        │
│      ↓        │         │ reformulates      │         │ Narrative analyst  │
│ Evaluator     │         │  queries          │         │  MD&A, risk factors│
│  score        │         │ multi-hop         │         │ Reconciler         │
│  relevance    │         │  retrieval        │         │  numbers vs. story │
│      ↓        │         │ judges retrieved  │         │ Citation verifier  │
│ Extractor     │         │  quality before   │         │  every claim →     │
│  → markdown   │         │  answering        │         │  source span       │
│      ↓        │         │ refuses when      │         │                    │
│ Processor     │         │  unsupported      │         │                    │
│  chunk+embed  │         │                   │         │                    │
└───────┬───────┘         └─────────┬─────────┘         └──────────┬─────────┘
        │                           │                              │
        └───────────────┬───────────┴──────────────────────────────┘
                        ▼
        ┌───────────────────────────────────┐
        │   MongoDB Atlas (M0 free tier)    │
        │   • documents + extracted markdown│
        │   • Atlas Vector Search index     │
        │   • LangGraph checkpoints (memory)│
        │   • audit ledger (cost + latency) │
        └───────────────────────────────────┘
```

### Agent roles

**Ingestion (offline pipeline):** Scanner (finds relevant filings for a company + date range) → Evaluator (scores relevance, avoids ingesting everything) → Extractor (filing → structured markdown, preserving table integrity — the hardest stage) → Processor (section-aware chunking, metadata tagging, embedding).

**Query time:** Document Assistant runs agentic RAG — decides *whether* retrieval is needed, reformulates queries, retrieves multiple times across documents, judges result quality before answering. Financial extractor pulls figures from XBRL. Narrative analyst reads MD&A and risk factors. Reconciler cross-checks the two — where they diverge is the interesting finding. Citation verifier strips any claim that doesn't resolve to a retrieved span.

### Design rules

- Numbers from XBRL, narrative from the LLM. Never the reverse.
- Refusal is a first-class, measured outcome — not a failure mode to hide.
- Citations are verified against retrieved spans before an answer ships, not generated and trusted.
- Cost is written at trace time, per agent, per retrieval.

---

## 3. Data sources

**Primary: SEC EDGAR — public domain, free, no API key.**

| Endpoint | Use |
|---|---|
| `data.sec.gov` Submissions API | Filing history per company (CIK) |
| `data.sec.gov` CompanyFacts / CompanyConcept (XBRL) | Authoritative structured financials |
| `efts.sec.gov` Full-text search | Search inside filing text |
| EDGAR archives | The filing documents themselves |

**Access rules, enforced:**
- **10 requests/second** hard limit across all EDGAR APIs — exceeding it earns a ~10-minute IP block
- **`User-Agent` header with name and email is mandatory** — requests without it are rejected outright
- No API key required

Ingestion runs well under the rate limit, caches every raw filing on disk so the pipeline replays without re-fetching, and commits a manifest of exactly which filings the corpus contains.

**Corpus scope, deliberately narrow:** 5–10 companies in one sector, 3 fiscal years of 10-K/10-Q filings each — roughly 100–200 filings. Narrow enough to iterate on, comparable enough for real multi-hop questions.

**Honesty note:** EDGAR filings are digital text, not scans — this is complex-layout and table extraction, not OCR. Stated plainly. An optional later phase adds a small synthetic degraded-document tier ([Augraphy](https://github.com/sparkfish/augraphy), MIT) for a measured extraction-accuracy-vs-quality curve, built with ground truth by construction.

---

## 4. Infrastructure

| Component | Service | Tier |
|---|---|---|
| Documents, vectors, checkpoints, audit ledger | **MongoDB Atlas M0** — Vector Search included | Free (512 MB) |
| LLM + embeddings | OpenAI via LiteLLM | pay-per-token |
| Complex layout/table extraction | Azure Document Intelligence (layout model) | F0 free |
| Voice interface (phase 3) | Azure Speech | F0 free |
| API | Azure Container Apps, scale-to-zero | free grant |
| Ingestion runs | GitHub Actions | free |
| Traces | Application Insights | 5 GB free |

Atlas replaces both a separate document store and a separate vector index — one service for documents, vectors, and agent memory, matching MongoDB's reference architecture directly. Azure region: `centralindia` (the only region in this subscription's allowlist carrying every needed service).

Target cost: **under $5/month.**

---

## 5. Measurement (the actual deliverable)

| Metric | Definition |
|---|---|
| Table extraction fidelity | % of financial tables preserved with correct row/column structure, checked against XBRL |
| Retrieval recall@k | Against a frozen question set with labeled source sections |
| Citation validity rate | % of claims resolving to a real supporting span |
| Correct refusal rate | Unanswerable questions correctly declined |
| Hallucinated-figure rate | Answer figures that don't match XBRL — target zero, reported honestly either way |
| Cost + P95 latency | Per agent and end-to-end |

**Frozen eval set:** 50–80 analyst questions with labeled source sections, written before tuning, deliberately including **unanswerable questions**. A system that never refuses isn't trustworthy — it's just confident.

_Table populated at ship. At least one weak question class will be published, not hidden._

---

## 6. Phases

| Phase | Deliverable |
|---|---|
| 1 — Core | EDGAR ingestion (rate-limited, cached) → extraction → Atlas Vector Search → Document Assistant with verified citations → frozen eval baseline |
| 2 — Multi-agent + MCP | Financial extractor, narrative analyst, reconciler, citation verifier. EDGAR access refactored behind MCP tools |
| 3 — Voice | [Praxis](https://github.com/KiRzEa/praxis) reused as the spoken query interface — an analyst asks aloud, hears the answer with sources |
| 4 — Optional | Synthetic scanned-document tier; extraction-accuracy-vs-quality curve |

Each phase ships independently. Phase 1 alone is a complete artifact.

---

## 7. Stack

Python · FastAPI · LangGraph · LiteLLM (OpenAI) · MongoDB Atlas (Vector Search + checkpointing) · Azure Document Intelligence · Azure Speech (phase 3) · Azure Container Apps · Application Insights

Built on [AgentBase](https://github.com/KiRzEa/AgentBase) — supervisor routing, hybrid retrieval, LiteLLM gateway, and observability emitter are reused, not rebuilt.

## 8. Honest limitations

- Digital-text extraction, not OCR — stated plainly.
- Narrow corpus (5–10 companies, one sector); results are not claimed to generalize.
- XBRL coverage varies by filer and period; where missing, figures fall back to LLM extraction and are flagged lower-confidence.
- Citation verification confirms a claim is *supported by a retrieved span* — not that the span itself is correct.
- Not investment advice.

## 9. Roadmap

- [ ] EDGAR ingestion pipeline (rate-limited fetcher, disk cache, corpus manifest)
- [ ] Filing extraction → structured markdown, table-preserving
- [ ] Chunking + Atlas Vector Search index
- [ ] Document Assistant (agentic RAG) with citation verification
- [ ] Frozen eval set + baseline measurement
- [ ] Multi-agent analysis layer (financial extractor, narrative analyst, reconciler)
- [ ] EDGAR access behind MCP tools
- [ ] Voice interface via Praxis
