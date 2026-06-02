# ABX — ApexBalanceEngine

![Python](https://img.shields.io/badge/python-3.10%2B-blue?logo=python)
![License](https://img.shields.io/badge/license-MIT-green)
![Status](https://img.shields.io/badge/status-Active-brightgreen)
![Dependencies](https://img.shields.io/badge/dependencies-minimal%20%28pydantic%29-lightgrey)

---

## 🧠 Overview

ABX (ApexBalanceEngine) is a production-grade, multi-layered AI agent memory
system purpose-built for algorithmic trading. It solves a fundamental problem
in autonomous trading: agents that execute without memory degrade under regime
change, repeat costly mistakes, and cannot adapt to evolving market structure.
ABX gives every agent in the stack a structured, persistent, and auditable
cognitive substrate — so decisions are grounded in accumulated evidence, not
stateless heuristics.

The system implements a full cognitive loop across six stages:
**PERCEIVE → RECALL → DECIDE → EXECUTE → REFLECT → ADAPT**. Market data and
signals enter through the active layer, are enriched by semantic and episodic
recall, routed through a rule engine with adaptive weights, executed via a
typed toolbox, reflected into episodic summaries, and fed back as weight
updates — closing the loop on every session.

For production deployment, ABX offers the properties quant teams require:
phase-gated learning that prevents premature weight adaptation on thin trade
history, R:multiple-based reinforcement grounded in realized edge, a
zero-external-dependency vector store for episodic similarity search, a
SQLite-backed audit trail with full weight snapshot history, thread-safe
concurrency throughout, and a single public API adapter that isolates all
agents from internal memory topology changes.

---

## ✨ Key Features

### Memory Architecture
- **Four-layer memory model** — Active (live state), Knowledge (facts/specs),
  Episodic (session reflections), and a Vector Store (similarity search) operate
  as independent, composable layers orchestrated by `UnifiedTradingMemory`
- **Per-ticker behavioral profiles** — `EntityProfile` tracks win rate by
  regime and session, avg R:multiple, MAE/MFE distributions, TP1 reach rate,
  and best context per instrument
- **Time-decayed knowledge** — `KnowledgeEntry` confidence scores decay over
  time; stale facts lose influence without manual pruning
- **18-dimension deterministic episode encoding** — `EpisodeEncoder` produces
  reproducible feature vectors (scales score, ML probability, direction, regime,
  session, timeframe, ATR ratio, spread ratio) enabling cosine similarity recall
  with no external ML dependencies

### Learning System
- **Phase-gated adaptation** — Phase 1 (≤50 trades), Phase 2 (51–150),
  Phase 3 (150+) gate weight update aggressiveness; the system cannot
  over-fit on sparse data
- **R:multiple reinforcement** — `adapt_weights()` adjusts rule weights
  bounded between `WEIGHT_MIN=0.10` and `WEIGHT_MAX=2.00` based on realized
  R:multiple, not raw P&L, making learning currency-agnostic
- **Lesson extraction and decay** — `_extract_lessons()` generates structured
  lessons from session summaries; `get_decayed_lessons()` applies confidence
  decay so old lessons fade gracefully

### Risk & Execution
- **Phase-aware guardrails** — `get_phase_guardrails()` and
  `apply_guardrails()` enforce risk controls calibrated to the agent's current
  learning phase, tightening exposure during low-confidence periods
- **Live MAE/MFE tracking** — Per-tick adverse and favorable excursion
  monitoring with stale position detection and unrealized P&L snapshots
- **Phase-aware cache TTL** — `SemanticCache` applies TTLs of 15s / 30s / 60s
  for Phases 1–3 respectively, preventing stale signal reuse during
  fast-moving conditions
- **Signal expiry enforcement** — `Signal.is_expired()` prevents execution of
  time-expired directional signals regardless of downstream state

### Infrastructure & Auditability
- **SQLite weight audit trail** — `database_weight_history.py` persists every
  weight snapshot and baseline per rule; full evolution history is queryable
  without external infrastructure — critical for post-incident review
- **Thread-safe concurrency** — `RLock` + WAL journal mode in the database
  layer; `SemanticCache` is independently thread-safe; safe for multi-threaded
  broker integrations
- **Zero external vector dependencies** — `VectorStore` runs entirely
  in-process with JSON serialization and namespace isolation; no Pinecone,
  Weaviate, or Chroma required
- **Structured async logging** — Loguru with three sinks (stdout DEBUG+,
  `trading_agent.log` INFO+, `errors.log` ERROR+ JSON) and `enqueue=True`
  for non-blocking writes; JSON error sink enables log aggregation pipelines

---

## 🏗️ Architecture

### Cognitive Loop

      flowchart LR
<img width="516" height="98" alt="image" src="https://github.com/user-attachments/assets/b745a16b-7af4-4355-bff5-2c276a7fb165" />

  

Layer Map
Layer	Module	Responsibility
Active	active_layer.py	Live positions, signals, real-time P&L, MAE/MFE
Execution	execution_layer.py	Rule engine, adaptive weights, guardrails, toolbox
Knowledge	knowledge_layer.py	Market rules, macro events, instrument specs, entity profiles
Episodic	episodic_layer.py	Session reflections, lesson extraction, performance feed
Orchestration	unified_memory.py	Phase detection, confidence assembly, layer wiring
Public API	abx_memory_adapter.py	Agent-facing interface; order ID mapping
Vector Store	vector_store.py	In-process episodic similarity search
Audit DB	database_weight_history.py	SQLite weight history, snapshots, baselines

📁 Repository Structure

<img width="681" height="299" alt="image" src="https://github.com/user-attachments/assets/2f14d9d7-514e-4d74-9a27-ffb00d4c3663" />


⚙️ Installation
Requirements: Python 3.10+, no broker SDK required for core memory system.

cd abx
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
requirements.txt declares a single confirmed external dependency:

pydantic>=2.0.0
All other components — vector store, SQLite audit layer, logging — use the
Python standard library.

🚀 Quick Start
1. Initialize the Memory System
from unified_memory import UnifiedTradingMemory

memory = UnifiedTradingMemory()
2. Connect an Agent via the Public Adapter

from abx_memory_adapter import ABXMemoryAdapter

adapter = ABXMemoryAdapter(memory)

# Horizon Scanner deposits a signal
adapter.horizon_scanner_store_signal(
    ticker="NQ",
    direction="LONG",
    regime="trending",
    session="london",
    setup_context="breakout_retest",
    scales_score=0.82,
)
3. Pre-Trade Confidence Assembly

# Oracle retrieves enriched context before sizing a position
context = adapter.oracle_get_trade_context(
    ticker="NQ",
    regime="trending",
    session="london",
)
# Returns: confidence score, decayed lessons, entity profile,
#          similar historical episodes, active guardrails
4. Record a Completed Trade

from trade import Trade

trade = Trade(
    ticker="NQ",
    direction="LONG",
    regime="trending",
    session="london",
    setup_context="breakout_retest",
    scales_score=0.82,
    reached_tp1=True,
    mae=-4.5,
    mfe=18.2,
    duration_hours=1.3,
    r_multiple=2.1,
)
adapter.chronicle_record_trade(trade)
5. Post-Session Reflection and Weight Adaptation

# Guardian triggers end-of-session reflection
adapter.guardian_close_session(session_id="session_20260601_london")

# WorkflowEngine pulls performance data and adapts rule weights
memory.execution_layer.adapt_weights(
    memory.episodic_layer.get_performance_data()
)


🔄 Phase System
ABX gates learning aggressiveness by trade count to prevent overfitting on
sparse data. Phase boundaries are enforced in UnifiedTradingMemory and
propagate to cache TTLs, guardrail thresholds, and weight update magnitude.

Phase	Trade Count	Cache TTL	Learning Posture
Phase 1	≤ 50 trades	15 seconds	Conservative — wide guardrails, slow weight drift
Phase 2	51–150 trades	30 seconds	Moderate — balanced adaptation
Phase 3	150+ trades	60 seconds	Confident — tighter guardrails, full weight range
Weight bounds apply across all phases: WEIGHT_MIN = 0.10, WEIGHT_MAX = 2.00.


🔍 Vector Similarity Search
VectorStore provides in-process episodic recall without any external vector
database. Each episode is encoded as an 18-dimensional deterministic feature
vector by EpisodeEncoder:


[scales_score, ml_probability, direction,
 regime×4, session×5, timeframe×4,
 atr_ratio, spread_ratio]
Similarity is computed via cosine distance. Namespaces isolate episodes by
ticker or strategy. The store serializes to JSON for persistence and reloads
deterministically — no embedding model, no network call, no non-determinism.


# Recall the 5 most similar historical episodes to a current signal
similar = memory.recall(signal=current_signal, top_k=5)


🛡️ Risk Controls Summary
ABX enforces risk at multiple independent layers — a single misconfigured
rule cannot bypass all controls.

Control	Layer	Mechanism
Phase guardrails	Execution	get_phase_guardrails() / apply_guardrails()
Weight bounds	Execution	WEIGHT_MIN=0.10, WEIGHT_MAX=2.00 hard clamps
Signal expiry	Active	Signal.is_expired() blocks stale execution
Stale position detection	Active	WorkingMemory flags positions past TTL
Cache TTL (phase-aware)	Active	SemanticCache.set_with_phase()
Confidence gating	Orchestration	Pre-trade score threshold in UnifiedTradingMemory
Audit trail	Database	Immutable SQLite weight snapshots, WAL mode


🤖 Agent Interface Reference
ABXMemoryAdapter exposes a dedicated method surface for each agent role,
keeping agent code decoupled from internal memory topology.

Agent	Role	Key Adapter Methods
Horizon Scanner	Signal generation	store_signal()
Oracle	Pre-trade analysis	get_trade_context(), get_confidence_score()
Guardian	Risk oversight	close_session(), get_guardrails()
Fulcrum	Position sizing	get_entity_profile(), get_phase()
SCALES	Entry scoring	get_scales_context()
Compass	Regime detection	get_regime_knowledge()
Herald	News/macro events	store_news_impact(), get_news_profile()
Chronicle	Trade logging	record_trade(), get_session_summary()
All agents share a single order_id_map (abx_trade_id → memory_trade_id)
maintained by the adapter, ensuring consistent trade identity across the stack.


📊 Data Models
Signal

Signal(
    ticker="NQ",
    direction="LONG",       # LONG/BUY or SHORT/SELL accepted
    regime="trending",
    session="london",
    setup_context="breakout_retest",
    scales_score=0.82,
    timestamp=datetime.utcnow(),
)
# signal.is_expired()        → bool
# signal.to_oracle_context() → dict
Trade

Trade(
    ticker="NQ",
    direction="LONG",
    regime="trending",
    session="london",
    setup_context="breakout_retest",
    scales_score=0.82,
    reached_tp1=True,
    mae=-4.5,               # Max Adverse Excursion (points)
    mfe=18.2,               # Max Favorable Excursion (points)
    r_multiple=2.1,
    duration_hours=1.3,
    setup_id="setup_NQ_breakout_v2",
    abx_trade_id="abx_20260601_001",
)


🪵 Logging
Three concurrent sinks are configured at startup via logger.py:

Sink	Level	Format	Purpose
stdout	DEBUG+	Human-readable	Development / live monitoring
trading_agent.log	INFO+	Structured text	Operational audit log
errors.log	ERROR+	JSON	Alerting pipelines / log aggregators
All sinks use enqueue=True — log writes are non-blocking and will not
introduce latency into the execution path.


🗄️ Audit Trail
database_weight_history.py maintains a SQLite database with two tables:

weight_snapshots — full timestamped history of every rule weight change, queryable for post-incident reconstruction
weight_baselines — per-rule baseline weights for drift analysis
The database uses WAL (Write-Ahead Logging) journal mode and an RLock for
thread safety. Zero external dependencies — the audit trail is always
available regardless of network or cloud service status.


from database_weight_history import WeightHistoryDB

db = WeightHistoryDB("abx_weights.db")
db.snapshot_weights(rule_id="breakout_retest", weight=1.45, phase=2)
history = db.get_weight_history(rule_id="breakout_retest")


🧪 Testing Guidance
ABX's deterministic design makes unit testing straightforward:

EpisodeEncoder — assert identical feature vectors for identical inputs (no randomness, no model calls)
SemanticCache — test TTL expiry per phase boundary and LRU eviction under concurrent access
adapt_weights() — verify weight clamps at WEIGHT_MIN / WEIGHT_MAX and correct directional update for positive vs. negative R:multiples
WeightHistoryDB — confirm snapshot persistence and retrieval across process restarts using a temp SQLite file
Phase detection — assert correct phase assignment at trade counts 0, 50, 51, 150, 151
Integration tests should instantiate UnifiedTradingMemory end-to-end and
drive a synthetic session through all six cognitive loop stages.


🗺️ Roadmap
 Async-native execution layer for high-frequency signal throughput
 Pluggable vector backend interface (Chroma / Qdrant drop-in)
 REST API wrapper for multi-process agent deployments
 Prometheus metrics exporter for weight drift and phase transitions
 Backtesting harness with deterministic session replay

 
📄 License
MIT License — see LICENSE for full terms.

ABX is a memory and decision infrastructure layer. It does not constitute
financial advice and carries no warranty of trading performance.
All deployment decisions remain the responsibility of the operator.
