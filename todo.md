# Implementation Plan – Polymarket CLOB Arbitrage Connector & Strategy (MVP)
Level of granularity: small-to-medium tasks (~0.5–4 hours each for experienced Python/Hummingbot developer)

## Phase 0 – Preparation & Environment (1–3 days)

- [ ] **0.1** – Set up development environment
  - Dependencies: 0.0 (starting point)
  - Create dedicated python 3.10+ virtualenv or conda env
  - Clone latest hummingbot dev branch (`git clone https://github.com/hummingbot/hummingbot.git -b development`)
  - Install hummingbot dev dependencies (`bin/install && bin/compile`)
  - Install additional libs: `pip install py-clob-client python-binance web3 numpy`
  - Create new branch: `git checkout -b feat/polymarket-connector-arbitrage-mvp`
  - Verify you can run `bin/hummingbot.py` in paper mode

- [ ] **0.2** – Create project documentation folder & initial notes
  - Dependencies: none
  - In root create folder `/docs/polymarket-arbitrage-mvp/`
  - Copy current PRD into `PRD.md`
  - Create `implementation-notes.md` for decisions, gotchas, measurements
  - Create `latency-measurements.md` (will be filled later)

## Phase 1 – Polymarket Connector Skeleton (core read-only first)

- [ ] **1.1** – Create folder structure for new connector
  - Dependencies: 0.1
  - Create folders:
    ```
    hummingbot/connector/exchange/polymarket/
    ├── __init__.py
    ├── polymarket_exchange.py
    ├── polymarket_constants.py
    ├── polymarket_utils.py
    ├── polymarket_web_utils.py
    └── data_sources/
        ├── polymarket_data_source.py
        └── polymarket_active_markets.py
    ```

- [ ] **1.2** – Implement basic constants & configuration skeleton
  - Dependencies: 1.1
  - In `polymarket_constants.py`:
    - Define base URLs (clob.polymarket.com, gamma-api.polymarket.com)
    - Chain ID (Polygon = 137)
    - Default tokens (USDC address)
    - Fee tiers mapping (from current docs)
  - Create basic ConfigMap dataclass in `polymarket_exchange.py`

- [ ] **1.3** – Implement read-only market data endpoints (REST + WS)
  - Dependencies: 1.2
  - Implement `get_markets()`, `get_order_book()`, `get_recent_trades()`
  - Use official `py-clob-client` for unauthenticated read operations
  - Cache active markets list (refresh every 5–15 min)
  - Add basic market filtering helper (crypto tags + short duration)

- [ ] **1.4** – Implement WebSocket public channel subscriptions
  - Dependencies: 1.3
  - Subscribe to orderbook, ticker, trades for selected token_ids
  - Parse incoming messages into Hummingbot standard format
  - Implement reconnection logic with exponential backoff

- [ ] **1.5** – Add private endpoints skeleton (balance, orders) – read only
  - Dependencies: 1.4
  - Add `private_key` to config
  - Implement `get_balance()` using py-clob-client authenticated client
  - Implement `get_open_orders()`

## Phase 2 – Order Execution & Full Trading Loop

- [ ] **2.1** – Implement order placement & cancellation (signed)
  - Dependencies: 1.5
  - Add `create_order()`, `cancel_order()`, `batch_cancel()`
  - Use py-clob-client `create_and_post_order()` with proper signature
  - Support LIMIT and MARKET orders
  - Handle post-only / IOC / FOK if supported

- [ ] **2.2** – Implement order & fill tracking via WebSocket
  - Dependencies: 2.1 + 1.4
  - Subscribe to user private channel (fills, order updates)
  - Map Polymarket events → Hummingbot OrderFilledEvent / OrderCancelledEvent
  - Maintain in-flight orders dictionary

- [ ] **2.3** – Implement full balance snapshot & real-time updates
  - Dependencies: 2.2
  - Track USDC available + locked
  - Listen to funding events if any

- [ ] **2.4** – Add basic paper trading simulation mode
  - Dependencies: 2.3
  - When paper trading → simulate fills based on book depth
  - Very useful for early strategy testing

## Phase 3 – Binance Real-time Data Feed

- [ ] **3.1** – Create Binance data feed module
  - Dependencies: 0.1
  - New file: `hummingbot/market_data_feed/binance_ws_feed.py`
  - Subscribe to `@ticker`, `@trade` or `@bookTicker` for selected pairs
  - Keep rolling window of last 3 minutes of trades/ticks

- [ ] **3.2** – Implement 2-minute delta calculator
  - Dependencies: 3.1
  - Function: `calculate_momentum_delta(pair: str, window_sec: int = 120) -> float`
  - Return % change over exact window

- [ ] **3.3** – Implement sigmoid probability mapper
  - Dependencies: 3.2
  - Function: `delta_to_true_prob(delta_pct: float, sensitivity: float = 3.0) -> float`
  - Implement centered logistic function (0.5 base)

## Phase 4 – Core Arbitrage Strategy Logic

- [ ] **4.1** – Create strategy folder & skeleton
  - Dependencies: 2.4 + 3.3
  - Create `hummingbot/strategy_v2/controllers/polymarket_momentum_arb/`
  - Basic Controller class inheriting from `ExecutorControllerBase`

- [ ] **4.2** – Implement market discovery & selection logic
  - Dependencies: 4.1 + 1.3
  - Filter active 15-min / hourly crypto markets
  - Map Binance symbol → Polymarket condition token_id(s)

- [ ] **4.3** – Implement core EV calculation
  - Dependencies: 4.2
  - Function that takes:
    - true_prob
    - polymarket mid price
    - current fees (dynamic)
    - projected slippage
  - Returns EV percentage

- [ ] **4.4** – Implement dynamic sizing (quarter-Kelly)
  - Dependencies: 4.3
  - Function: `calculate_position_size(ev: float, price: float, bankroll: float)`
  - Implement fractional Kelly with safety caps

- [ ] **4.5** – Implement entry decision logic
  - Dependencies: 4.3 + 4.4
  - Check EV ≥ threshold
  - Check slippage ≤ 1.5%
  - Calculate size
  - Decide buy YES / NO

- [ ] **4.6** – Implement adverse momentum exit logic
  - Dependencies: 4.5
  - Monitor open positions
  - If true_prob reverses below threshold → market sell full position

- [ ] **4.7** – Implement daily loss circuit breaker
  - Dependencies: 4.6
  - Track running P&L since UTC midnight
  - Pause all new entries when -10%

## Phase 5 – Integration, Testing & Hardening

- [ ] **5.1** – Integrate strategy with connector
  - Dependencies: 4.7 + 2.4
  - Wire controller to PolymarketExchange instance

- [ ] **5.2** – Add comprehensive logging & metrics
  - Dependencies: 5.1
  - Log every EV calculation, entry decision, exit reason
  - Track latency per stage (data → decision → order sent)

- [ ] **5.3** – Create configuration YAML schema & examples
  - Dependencies: 5.2

- [ ] **5.4** – Write basic unit tests (data parsing, EV calc, sizing)
  - Dependencies: 5.3

- [ ] **5.5** – Paper trading validation (at least 3–7 days)
  - Dependencies: 5.4

- [ ] **5.6** – Small real-money test (very small size)
  - Dependencies: 5.5

## Phase 6 – Optimization & Private Live Running

- [ ] **6.1** – Latency measurement & optimization round
  - Dependencies: 5.6
- [ ] **6.2** – Parameter sensitivity analysis & tuning
  - Dependencies: 6.1
- [ ] **6.3** – Scale-up monitoring & alerting setup
  - Dependencies: 6.2

```

### Quick Summary of Dependencies Structure

```
0.1 → 0.2
      ↓
    1.1 → 1.2 → 1.3 → 1.4 → 1.5 → 2.1 → 2.2 → 2.3 → 2.4
                        ↓                             ↓
                      3.1 → 3.2 → 3.3                 │
                              ↓                       │
                            4.1 ←─────────────────────┘
                              ↓
                  4.2 → 4.3 → 4.4 → 4.5 → 4.6 → 4.7
                                        ↓
                                      5.1 → 5.2 → 5.3 → 5.4 → 5.5 → 5.6
                                                            ↓
                                                          6.1 → 6.2 → 6.3