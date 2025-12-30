# AlgoTrading Capstone (Bitcoin RL) – Current Project Context

## Why this file exists
This Markdown is a **high-level, up-to-date context snapshot** for AI tools working on the project.  
It focuses on **what is implemented today**, the **key contracts** between components, and the **planned integration path**.

---

## Current reality: two working codebases

### A. Bitcoin RL Training Engine (offline, CLI)
**Goal:** produce trained RL policies + backtests from historical data.  
**Runs on:** developer machines (local CPU/GPU), not as a server.  
**Outputs:** model artifacts (actor checkpoint), run metadata, evaluation metrics, and backtest artifacts.

### B. Bitcoin Trading Strategy Execution System (runtime, server-side)
**Goal:** operate as a long-lived service (target: AWS 24/7) that maintains a rolling OHLCV window in TimescaleDB and periodically runs strategies to emit signals.  
**Current behavior:** signal generation + scheduling; **no live trade execution yet**.

---

## A. RL Training Engine – what exists now

### Responsibilities
- **Data pipeline** that downloads and prepares historical BTC data (CCXT-based), including feature engineering and strategy signal generation.
- **Custom RL environment** (`BitcoinTradingEnv`) that models portfolio evolution, position constraints, fees, stop-loss logic, and reward shaping.
- **Training** using a vendored ElegantRL stack (e.g., PPO/SAC), driven by a single CLI orchestrator (`main.py`).
- **Backtesting & reporting**: produces `steps.csv`, `trades.csv`, `metrics.json`, `summary.json`, and plots per run/backtest.

### Key contracts (must stay stable for deployment compatibility)
**State vector composition (high level):**
- Portfolio state (balance/holdings)
- Market features (OHLCV + technical indicators)
- Risk features (turbulence / optional external series like VIX)
- **Strategy signals** encoded consistently (one-hot per strategy per signal type)
- Normalization rules are part of the contract.

**Action space (current):** continuous `[-1, 1]` with two components:
- `a_pos`: desired exposure (direction + magnitude)
- `a_sl`: stop-loss tightness control

**Trade constraints (examples):** leverage limit, max BTC position cap, transaction fee/slippage, exposure deadzone, stop-loss bounds.

### Outputs and run structure
Training runs are organized under a run directory that contains:
- `metadata.json` (the “model contract”: indicator list, strategy list/order, env/training parameters, etc.)
- `elegantrl/act.pth` (actor checkpoint) and related training artifacts
- `backtests/` subfolders with backtest outputs and plots

---

## B. Strategy Execution System – what exists now

### Responsibilities
- **Database-backed market data service**:
  - Stores candles in PostgreSQL + TimescaleDB hypertable
  - Maintains a rolling window of `MAX_LOOKBACK_HOURS` for `MIN_TIMEFRAME` candles
  - Syncs from Binance via CCXT (incremental, idempotent; prunes old candles)
- **Scheduler + orchestration**:
  - APScheduler-driven global tick (`GLOBAL_TICK`, e.g., every minute)
  - Per-strategy timeframe alignment (execute only on timeframe boundaries)
- **Strategy execution engine**:
  - Loads enabled strategies from `strategies_registry.json`
  - Loads base OHLCV once per tick, resamples per strategy timeframe, trims to each strategy’s `lookback_hours`
  - Executes strategies in parallel (ThreadPoolExecutor) with per-strategy timeout isolation

### Current outputs
- Each strategy returns `StrategyRecommendation(signal, timestamp)` where signal is:
  - `LONG`, `SHORT`, `FLAT`, or `HOLD`
- Results are currently surfaced via console/log output (no persistent signal store yet).

---

## Shared strategy framework

Current implemented strategies:
- VolatilitySystem (ATR breakout)
- SupertrendStrategy
- OTTStrategy
- BbandRsi
- AwesomeMacd

### Strategy Candidates

1. **Technical strategies**  
   - Trend / Moving Averages
   - Volatility / Breakout
   - Simple mean-reversion around VWAP/range

2. **Flows / On-exchange data**  
   - Exchange inflows
   - Exchange outflows

3. **Derivatives-based**  
   - Funding rate extremes
   - Open interest spikes / perp–spot imbalance

4. **News / Event-driven**  
   - Detect breaking crypto/BTC news from major sources  
   - Mark news-related windows as “high attention”

5. **Social / Sentiment**  
   - Spikes in BTC mentions  
   - Basic positive/negative sentiment

---

## What is explicitly NOT implemented yet
- Live trading / order execution
- Position reconciliation against an exchange
- Centralized persistence of decisions/trades in the runtime system
- Full RL feature/state construction in the runtime system
- Policy inference in the runtime system (actor loading + forward pass)
- Risk management policy beyond what exists in the training environment
- The server-side part of the project will run on AWS, most likely on a Kubernetes cluster
- The client-side part is a separate desktop application (Java/JavaFX) that connects to the server-side services for monitoring and control.

---

## Near-term roadmap (the “bridge” between codebases)
1. **State parity:** build the exact RL state in the runtime system (same feature order + normalization + strategy encoding as training).
2. **Model inference:** load the selected actor checkpoint (`act.pth`) and compute actions on each decision step.
3. **Execution layer:** convert actions to real trades (planned: Kraken Futures via CCXT), including position/risk controls and order lifecycle.
4. **Observability:** persist and monitor signals/actions/trades, health checks, and latency.

---

## Repository & Project Management (GitHub)
- **GitHub Repositories** — multiple repos
- **GitHub Projects / Issues** — for task management, backlog
- **GitHub Actions (CI)** — automated builds, tests
- **ArgoCD (CD)** — pulls from the GitHub repos and syncs to the Kubernetes cluster (GitOps flow)

---

## Development Environment
- **IDE:** PyCharm (Professional), IntelliJ IDEA (Professional)
- **Python Version:** 3.11.7  

---

## Database Infrastructure

- **PostgreSQL 16.11** with **TimescaleDB 2.23.1** extension
- Time-series optimizations: hypertables, resampling, 90% compression
- ACID-compliant for orders/metadata + time-series performance for OHLCV candles
- Python: **psycopg2-binary 2.9.9**, **SQLAlchemy 2.0.23**

---

## Purpose of this Document
This document is meant to give LLMs full context about:
- what the project is trying to achieve
- what components already exist conceptually
- which tools/libraries have been chosen
- and which parts are intentionally *not* finalized yet

It should be used as a high-level description of the system’s architecture and intent, not as a final implementation spec.
