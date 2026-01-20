# Product Requirements Document (PRD)  
**Polymarket Arbitrage Connector for Hummingbot**  
**Version: MVP 1.0**  
**Date: January 19, 2026**  
**Author: Luis (with Grok assistance)**  
**Project Status: Pre-implementation – Private MVP**

## Project Overview

**Goal**  
Build a high-performance, low-latency connector for Polymarket (CLOB-style prediction market on Polygon) integrated into Hummingbot, focused on cross-market arbitrage between Polymarket short-duration crypto markets and Binance spot prices.

**Primary Strategy (MVP)**  
Latency/momentum arbitrage on high-volume, short-duration Polymarket crypto prediction markets (mainly 15-minute up/down, some hourly and daily threshold markets) for BTC, ETH, SOL, XRP, etc.  
The bot detects when Binance spot price momentum creates a temporary mispricing in Polymarket probabilities → executes trades before the market reprices.

**Key Value Proposition**  
- Exploit frequent, small (3–5%+ EV) but very high-probability edges (target win rate 95%+) in rolling 15-minute windows  
- Leverage Basel location for <30–50 ms round-trip latency to Polymarket CLOB and Binance endpoints  
- Maximize early alpha with private deployment → open-source contribution after edge compression

**Deployment Philosophy**  
Private first (milk the edge for weeks/months) → later open-source contribution to official Hummingbot repository

**Target Outcome (MVP)**  
Stable, profitable bot capable of farming dozens to hundreds of high-confidence trades per day across multiple crypto assets with controlled drawdown.

## Core Requirements

- Real-time price & momentum feed from Binance WebSocket (primary truth source)  
- Full read/write access to Polymarket CLOB (market data, order book, place/cancel orders, trade fills)  
- Ultra-low-latency order execution path (fast ECDSA signing + WebSocket order submission)  
- Conservative expected-value (EV) based entry filter (minimum 3–5% EV after fees & slippage)  
- Dynamic position sizing based on edge strength (fractional Kelly)  
- Adverse momentum protection exit (safety override)  
- Hard daily loss protection (10% bankroll)  
- Liquidity & slippage pre-check before every order

## Core Features

| Feature                              | Description                                                                                   | Priority |
|--------------------------------------|-----------------------------------------------------------------------------------------------|----------|
| Polymarket CLOB Connector            | REST + WebSocket for markets, order book, orders, fills; wallet-based signing (py-clob-client) | Must-have |
| Binance WebSocket Integration        | Real-time ticker/trade/depth for BTCUSDT, ETHUSDT, SOLUSDT, etc.                             | Must-have |
| Momentum-based True Probability      | 2-min % delta → sigmoid(sensitivity=3) → true prob of up/down or threshold breach            | Must-have |
| EV Entry Logic                       | Calculate EV post-fees/slippage; trigger ≥ 3–5%                                               | Must-have |
| Dynamic Position Sizing              | Quarter-Kelly approximation, high caps ($20k/trade, $50k/market)                             | Must-have |
| Adverse Momentum Exit                | Sell full position if true_prob reverses below ~0.35–0.4 mid-window                         | Must-have |
| Hold-to-Resolution (default)         | No partial profit taking; rely on quick resolution                                            | Must-have |
| Liquidity/Slippage Filter            | Pre-check book depth; max 1.5% projected slippage at target size                             | Must-have |
| Daily Loss Circuit Breaker           | Pause trading for UTC day after -10% bankroll drawdown                                        | Must-have |
| No concurrency limit                 | Farm every qualifying market simultaneously                                                   | Must-have |
| Configurable Parameters              | Sensitivity, lookback, EV threshold, Kelly fraction, caps, etc.                              | Should-have |
| Logging & Monitoring                 | Detailed trade logs, EV calc traces, latency metrics                                          | Should-have |

## Core Components

| Component                            | Technology / Library                                      | Responsibility |
|--------------------------------------|------------------------------------------------------------|----------------|
| Polymarket Connector                 | Custom CLOB-style (inherits Hummingbot base) + py-clob-client | API communication, signing, order lifecycle |
| Binance Data Feed                    | python-binance or websocket-client                         | Real-time price & trade stream |
| Strategy Controller                  | Hummingbot V2 Controller / custom strategy                | EV calc, entry/exit decisions, sizing |
| Momentum Engine                      | Pure Python (numpy optional for speed)                     | 2-min delta → sigmoid true prob |
| Risk Engine                          | Internal logic                                             | Daily loss tracking, slippage simulation, adverse exit trigger |
| Wallet & Signing                     | web3.py + private key                                      | Polygon transaction signing (for settlement if needed) |
| Configuration                        | YAML (Hummingbot standard)                                 | All tunable params |
| Logging                              | Hummingbot logger + structured JSON logs                   | Debugging & performance analysis |

## App / User Flow

1. **Setup**  
   - User configures private key (Polymarket wallet), USDC funding amount  
   - Selects target assets (BTC, ETH, SOL, XRP…)  
   - Adjusts strategy params (EV min, sensitivity, lookback, Kelly fraction, caps)

2. **Startup**  
   - Connects to Polymarket CLOB WS + REST  
   - Subscribes to Binance WS streams for selected pairs  
   - Discovers active short-duration crypto markets via Gamma API

3. **Main Loop (per second / on tick)**  
   - Update Binance price window (rolling 2 min)  
   - Calculate current delta → true_prob for each monitored market  
   - Fetch Polymarket mid-price / book for corresponding YES/NO  
   - Compute EV (post dynamic fees + projected slippage)  
   - If EV ≥ threshold & slippage ≤ 1.5%:  
     → Calculate dynamic size (quarter-Kelly)  
     → Place limit/market order (aggressive to front-run)  
   - Monitor open positions:  
     → Check adverse momentum every few seconds  
     → Sell full position if true_prob reverses significantly  
     → Otherwise hold until resolution

4. **Resolution & Settlement**  
   - Auto-redeem winning shares post-resolution (if needed)  
   - Update bankroll & daily P&L  
   - Pause if -10% daily limit hit

5. **Shutdown / Pause**  
   - Cancel all open orders  
   - Log final performance stats

## Tech Stack

| Layer                     | Technology / Library                                 |
|---------------------------|------------------------------------------------------|
| Base Framework            | Hummingbot (latest dev branch, Python 3.10+)         |
| Polymarket API Client     | py-clob-client (official Python client)              |
| Binance Real-time Data    | python-binance or pure websocket-client + asyncio   |
| Signing & Web3            | web3.py (Polygon RPC)                                |
| Async / Performance       | asyncio, uvloop (recommended)                        |
| Math & Speed              | numpy (optional), pure Python for core logic         |
| Deployment                | Docker (Hummingbot standard) or source install       |
| Monitoring                | Prometheus + Grafana (optional extension)            |
| Location Optimization     | Basel, Switzerland VPS / dedicated server            |

## Implementation Plan (High-Level Phases)

**Phase 0 – Preparation (1–3 days)**  
- Fork Hummingbot repo locally  
- Set up dev environment (Docker/source)  
- Study existing CLOB connectors (dYdX v4, Injective)  

**Phase 1 – Connector Skeleton (5–10 days)**  
- Implement PolymarketExchange class (CLOB base)  
- REST/WS market data, order book, balances  
- Order placement/cancellation with py-clob-client signing  
- Trade/fill websocket handling  

**Phase 2 – Binance Feed & Momentum Engine (3–7 days)**  
- Real-time Binance WS subscription  
- Rolling 2-min price window  
- Delta → sigmoid true_prob function  

**Phase 3 – Strategy Logic (7–14 days)**  
- Market discovery (Gamma API filter for short crypto)  
- EV calculation engine  
- Entry filter + slippage simulation  
- Dynamic quarter-Kelly sizing  
- Adverse momentum exit trigger  
- Daily loss circuit breaker  

**Phase 4 – Testing & Tuning (ongoing)**  
- Backtesting harness (historical Binance ticks + Polymarket snapshots)  
- Paper trading mode  
- Small real capital test ($1k–$5k)  
- Monitor latency, win rate, drawdown  

**Phase 5 – Private Live & Optimization (weeks–months)**  
- Scale up capital  
- Tune parameters (sensitivity, EV threshold, Kelly fraction)  
- Add logging/alerts  

**Phase 6 – Open-source Contribution (after edge compression)**  
- Clean code, add unit/integration tests  
- Write documentation & usage guide  
- Submit PR / Hummingbot Governance Proposal  