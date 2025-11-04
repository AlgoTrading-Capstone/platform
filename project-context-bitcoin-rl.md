# Meta-Level Strategy for Managing Multiple Bitcoin Trading Strategies

## Concept
This project defines a **meta-level (top) strategy** that manages and coordinates several independent Bitcoin trading strategies. Each strategy runs separately (as a microservice or module), produces its own trading recommendation, and the meta-strategy decides how to combine these recommendations into a single trading action.

The target market is **Bitcoin futures** so that the system can:
- go long,
- go short,
- stay flat/hold,
and, of course, exit existing positions.

---

## Main Components

### 1. Strategy Services (Microservices)
- Each trading idea/logic is implemented as an independent strategy service (Python-based).
- Strategies will be sourced from existing, proven open-source algo-trading codebases (after manual review).
- Each strategy is called at runtime and returns a structured **recommendation** for the current time step.
- We have **not** finalized the exact output schema of a strategy yet — only that it will be unified so the meta-engine can consume it.
- The purpose is to combine *already working* or *already defined* strategies, not to reinvent every signal from scratch.

#### Strategy Candidates (to be wrapped as microservices)

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

### 2. RL-Based Decision Engine
- Above the strategy services sits a **Reinforcement Learning (RL)** engine that chooses the final action.
- Current preferred RL engine: **FinRL** (because it is finance-oriented and supports standard algorithms like PPO/A2C/SAC).
- The RL environment will be customized so that its **observation** is essentially “what all strategies said right now” plus a few global fields (e.g. current position, volatility).
- The RL **action space** will be a **limited, predefined set** of actions (e.g. increase exposure, decrease exposure, stay flat, possibly reverse). We are **not** locking to “always buy/sell full position”.
- Whenever a **new strategy** is added to the system, the RL model will be **retrained** so it can learn all permutations/combinations of the available strategies.
- There will also be **periodic retraining** (daily/weekly) to re-align the meta-strategy with current market conditions.

### 3. Exchange Integration
- Actual order placement will be done via **CCXT**.
- Primary exchange for now: **Kraken**.
- Using CCXT keeps the system exchange-agnostic and lets us switch or add exchanges with minimal changes.
- Trading is intended for **Bitcoin futures** to support long/short/flat flows.

### 4. Runtime Environment
- The **server-side part** of the project will run on **AWS**, most likely on a **Kubernetes cluster**.
- Each trading strategy will run as its **own pod** (separate deployable unit), so strategies stay isolated and can be updated independently.
- Strategy pods will be invoked **concurrently** to minimize latency in each decision cycle.
- The **client-side part** is a separate desktop application (Java/JavaFX) that connects to the server-side services for monitoring and control.

### 5. Client Application
- A **desktop GUI** is planned, built in **Java / JavaFX**.
- Purpose: monitor system state and executed actions; chart embedding is planned but not finalized yet (we may later feed Kraken data into the chart and overlay trades).

---

## Performance Considerations
- We will use **multi-threading / concurrency** specifically for **invoking multiple strategy services in parallel**, so the meta-engine can get all recommendations in one time window.
- The actual **RL decision** and **order execution** will remain controlled/sequential to keep consistency and avoid race conditions with real funds.

---

## Not Yet Finalized
- **Strategy output schema**: we know it must be unified, but exact fields (e.g. confidence, horizon) are not fixed yet.
- **Exact action set** for the RL: we will use a small, fixed action space, but the precise actions are still to be defined.
- **Risk controls**: max position, daily loss caps, etc. have **not** been defined yet — this will be added once the basic loop runs.

---

## Technologies (current choice)
- **Python** — strategy services and server-side logic
- **FinRL** — main RL engine for training/decision learning
- **CCXT** — exchange/execution layer (Kraken first)
- **Java / JavaFX** — desktop client/GUI
- **AWS** — hosting/running strategy services in parallel
- **Kubernetes + Helm** — orchestration and packaging (one pod per strategy)
- **GitHub Actions** — CI for building, testing, and publishing service images
- **ArgoCD** — GitOps-based CD for deploying updated services to the Kubernetes cluster
- **1Password** — secure storage and sharing of secrets, especially exchange API keys

---

## Repository & Project Management (GitHub)
- **GitHub Repositories** — Repository structure is still open (single monorepo vs. multiple repos).
- **GitHub Projects / Issues** — for task management, backlog, and tracking new strategies to be onboarded.
- **GitHub Actions (CI)** — automated builds, tests, linting, and container image creation on each push/PR.
- **ArgoCD (CD)** — pulls from the GitHub repos and syncs to the Kubernetes cluster (GitOps flow).

---

## Purpose of this Document
This document is meant to give LLMs full context about:
- what the project is trying to achieve,
- what components already exist conceptually,
- which tools/libraries have been chosen,
- and which parts are intentionally *not* finalized yet.

It should be used as a high-level description of the system’s architecture and intent, not as a final implementation spec.