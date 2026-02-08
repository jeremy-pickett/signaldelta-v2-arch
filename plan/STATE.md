# SignalDelta v2 — Development State & Decision Log

## Purpose
This file is the running context for atomic plan development. It tracks:
- Decisions made during planning conversations
- Corrections to the original meta-plan
- Cross-plan dependencies and interface contracts
- Open questions that need resolution in specific plans

Reference this file at the start of each planning session to avoid re-deriving context.

---

## Corrections to Meta-Plan (0.0.1)

### C-001: Timestamp Model Clarification
**Original assumption**: Four timestamps (event_time, observed_time, ingest_time, available_time) are all knowable.
**Correction**: For most collectors, the only timestamp we reliably have at ingest is `ingest_time` (when we fetched it). `event_time` is available from some authoritative sources (SEC filing timestamps, block timestamps) but is **nullable** for sources that don't provide it. `available_time` is **never stored at ingest** — it is derived at scoring/backtest time by the latency model. Market discovery time is not a timestamp we possess; it is the *output* of cascade analysis, not an input.

**Impact on plans**: P01 schema must make `event_time` nullable. P06 collectors must document which sources provide authoritative `event_time` vs. which only have `ingest_time`. P08/P09 latency model must handle missing `event_time` gracefully.

### C-002: P09a Is Engineering, Not Manual Curation
**Original assumption**: P09a is "primarily a data curation effort, not engineering" requiring manual labeling.
**Correction**: P09a is a **collector + classifier pipeline**. Labeled events are sourced programmatically from structured sources (EDGAR, Polygon, on-chain). Classification logic (e.g., "this 8-K is an M&A announcement") is NLP/regex against structured filings, coded once. Confidence scoring is algorithmic (source authoritativeness mapped in code, not assigned by hand).

**Impact on plans**: P09a fits architecturally alongside P06 (collector framework) as specialized collectors writing to `labeled_events`. It is engineering work with the same schedule/registry/health patterns as any collector. Dependency: P09a needs P06's base collector interface, so it should be planned alongside P06/P07 in Phase 3, not as a detached parallel workstream.

---

## Key Design Constraints (Carry Forward)

### DC-001: Single Schema, Dual Instance
One Postgres schema, two locations (AWS + offsite). Single-writer-per-table-group. Replication is per-table-group, unidirectional. Write routing enforced in application code. IDs: ULIDs for replicated tables.

### DC-002: Timestamp Contract
Every signal record carries: `event_time` (nullable), `observed_time` (when collector saw it), `ingest_time` (when written to DB). `available_time` is computed, not stored. All UTC TIMESTAMPTZ.

### DC-003: Scorer Is a Pure Function
`score()` has zero I/O, zero side effects. Same code runs production and backtests. Input: signal events + scoring model config. Output: ScoreSample. Extracted from research.py's 11K-line monolith.

### DC-004: LLM Weight Classes
- Light (cloud, any location): explain, cohort_match, report_chat, signal_classify (simple)
- Heavy (offsite only): deep_research, report_generate, html_render
- AWS API server must never call heavy models synchronously. Heavy use-cases dispatched via ARQ.

### DC-005: Determinism Invariant
Identical inputs → byte-identical outputs. Enforced via: sorted iteration, canonical JSON serialization (sorted keys, explicit nulls, 10-sig-digit floats), sha256 hashing, no RNG without explicit seed.

### DC-006: v2.0 Signal Manifest (TBD)
P06/P07 planning must produce a concrete list of S### signals that ship in v2.0. Estimated 25-30 signals. Selection criteria: collector exists and is functional, event_time is available or derivable, signal is wired to scorer or can be wired with minimal effort.

---

## Plan Status Tracker

| Plan | Status | Dependencies Satisfied? | Notes |
|------|--------|------------------------|-------|
| P01: Database Schema | **NEXT** | ✅ (Layer 0) | No deps |
| P02: Environment | Not started | ✅ (Layer 0) | No deps |
| P03: Auth | Not started | Needs P01 | |
| P04: LLM Gateway | Not started | Needs P01, P02 | |
| P05: Background Tasks | Not started | Needs P02 | |
| P06: Collectors | Not started | Needs P01, P02, P05 | |
| P07: Signal Norm | Not started | Needs P06 | |
| P08: Scoring Engine | Not started | Needs P07 | Plan jointly with P07 |
| P09a: Labeled Events | Not started | Needs P06 | Engineering, not curation (C-002) |
| P09: Backtesting | Not started | Needs P08, P09a | |
| P10: Tempo | Not started | Needs P08 | |
| P11: Cascade | Not started | Needs P08 | |
| P12: Conductor | Not started | Needs P08 | |
| P13: API Routes | Not started | Needs P01–P12 | |
| P14: Frontend | Not started | Needs P13 | |
| P15a: Min Observability | Not started | Needs P01, P02 | Gating for P05, P06, P09 |
| P15b: Full Observability | Not started | Needs all | |
| P16: Reports | Not started | Needs P08, P04, P05 | |

---

## Interface Contracts (Defined Across Plans)

### IC-001: RawRecord (P06 → P07)
The hard contract between collectors and signal normalization.
```
RawRecord:
  raw_record_id: ULID
  entity_id: str           # {ticker}:{exchange} or {token}:{chain}
  source_type: str          # collector identifier
  event_time: TIMESTAMPTZ?  # nullable per C-001
  observed_time: TIMESTAMPTZ
  ingest_time: TIMESTAMPTZ
  collector_version: str
  collector_run_id: ULID
  schema_version: str
  payload: JSONB
```

### IC-002: SignalEvent (P07 → P08/P09)
The hard contract between signal normalization and scoring/backtesting.
Defined in Backtest Spec §4.1. Carries: signal_id, family_id, entity_id, event_time, available_time, strength_norm [0,1], direction {-1,0,+1}, reliability [0,1], latency_confidence [0,1], chain_affinity vector.

### IC-003: ScoreSample (P08 → P09/P12/P13/P16)
Output of the scoring engine. Carries: magnitude [0,1], lean [-1,+1], confidence [0,1], chain_probs map, top_contributors list, counterfactual deltas. Plus legacy 0-100 score reconstruction.

### IC-004: LabeledEvent (P09a → P09)
```
LabeledEvent:
  event_id: ULID
  entity_id: str
  event_type: enum         # EARNINGS_SURPRISE, MNA_ANNOUNCEMENT, SEC_ENFORCEMENT, RUG_PULL, etc.
  event_time: TIMESTAMPTZ  # NOT nullable — this is ground truth
  event_description: str
  source_provenance: str   # where the label came from (EDGAR, Polygon, on-chain)
  label_confidence: float  # algorithmic, based on source authoritativeness
  metadata: JSONB
  dataset_version: str     # immutable snapshot versioning
```

---

## Open Questions (Resolve During Planning)

### OQ-001: Replication Lag SLA
What is the acceptable staleness for the AWS instance serving API reads? 5 seconds? 60 seconds? This affects user experience (stale scores) and operational decisions. Resolve in P01.

### OQ-002: Collector Liveness Check
How do we verify that 140+ collectors still work against current API endpoints before triaging? Automated health sweep? Resolve in P06.

### OQ-003: v2.0 Signal Manifest
Which specific S### signals ship in v2.0? The gap analysis suggests ~25-30 are feasible. Selection must happen during P06/P07 planning.

### OQ-004: Schema Migration Across Two Instances
How are DDL changes applied atomically to both AWS and offsite? Alembic doesn't natively handle multi-instance. Resolve in P01.

### OQ-005: Conflict Recovery Procedure
If a write hits the wrong instance (bug), what's the recovery path? Just logging isn't enough. Resolve in P01.
