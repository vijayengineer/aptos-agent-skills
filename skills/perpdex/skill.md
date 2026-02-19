# Skill: perp-architecture

## Purpose
This skill defines the canonical architecture for Apollo Perps: an experimental, hackathon-grade orderbook perpetual exchange built on Aptos.

It is used to constrain Codex/agent behavior so implementations across Move, backend, and frontend remain consistent.

This skill MUST activate for work involving:
- matching engine
- margin / collateral model
- liquidation logic
- REST API routes
- WebSocket message formats
- market naming and price feed integration
- on-chain escrow and position sync

---

## Scope and Non-Goals

### In scope (hackathon MVP)
- Isolated margin only
- Max leverage <= 5x
- Simple orderbook: limit orders only (market orders optional)
- Off-chain matching engine with REST + WS
- On-chain USDC escrow + operator-controlled position sync
- Real-time mark price streaming (Hyperliquid WS reference)
- Basic liquidation monitoring loop

### Out of scope (do NOT implement)
- cross-margin
- funding payments distribution
- insurance fund
- multi-collateral
- partial liquidation tiers
- on-chain matching
- MEV protection beyond basic sanity
- hedging execution to external venues

---

## High-level Architecture

### On-chain (Aptos Move)
Responsibilities:
1) Escrow vault for `Coin<USDC>` (generic phantom type)
2) Deposit and withdraw (user-signed)
3) Operator/admin sync for position state:
   - upsert_position
   - close_position
4) Liquidation (optional) or close_position used by operator after liquidation is detected off-chain

No matching logic on-chain.

### Off-chain (Node/TypeScript)
Responsibilities:
1) In-memory CLOB matching engine
2) Account state tracking:
   - collateral (total)
   - locked (used margin)
   - free collateral
3) Trade matching and order book maintenance
4) Liquidation monitoring based on mark price
5) REST API surface for:
   - auth
   - faucet (testnet)
   - deposit verification
   - withdraw verification
   - order placement
   - querying orderbook/positions
6) WebSocket broadcasting:
   - mark ticks
   - orderbook updates
   - trade updates
   - optional position updates

### Market Data (Hyperliquid WS)
HL is reference only:
- mark price / mid / best bid/ask
- used for UI display, PnL and liquidation checks
- NEVER route user trades to HL
- If HL disconnected, pause trading OR fallback to last known mark

---

## Canonical Market IDs
All systems MUST use the same market IDs.

Primary markets:
- GOLD-PERP
- SILVER-PERP

Optional additional markets:
- SPX-PERP
- OIL-PERP
- BTC-PERP

Market ID values MUST match:
- frontend market selector
- backend engine keys
- /orderbook?market=...
- /marks map keys
- /markHistory keys
- websocket `market` fields

---

## Margin Model (Isolated)

Constants:
- MAX_LEVERAGE = 5
- INITIAL_MARGIN = 0.20  (20%)
- MAINT_MARGIN = 0.10    (10%)
- LIQ_PENALTY = 0.02     (2% of notional)

Definitions:
- notional = abs(size) * mark_price
- unrealized_pnl:
  - long: size * (mark_price - entry_price)
  - short: size * (entry_price - mark_price)
- equity = collateral_allocated + unrealized_pnl
- maintenance_requirement = notional * MAINT_MARGIN

Liquidation condition:
- equity <= maintenance_requirement

On liquidation:
- close at mark price
- apply penalty
- release remaining collateral back to free collateral

---

## Orderbook / Matching Engine

Order types:
- Limit only for MVP

Order fields:
- id: string (uuid)
- user: address string
- market: MarketId
- side: buy | sell
- price: number
- size: number
- remaining: number
- ts: unix_ms

Sorting:
- bids: price desc, then ts asc (FIFO)
- asks: price asc, then ts asc

Crossing condition:
- match while bestBid.price >= bestAsk.price

Fill:
- fillSize = min(bid.remaining, ask.remaining)
- tradePrice = bestAsk.price (simple maker price rule)

After fill:
- decrement remaining
- remove orders with remaining == 0
- emit trade event

---

## On-chain Sync Rules

Deposits/Withdrawals:
- User signs deposit<USDC>(vault_addr, amount) / withdraw<USDC>(vault_addr, amount)
- Backend verifies txHash:
  - success true
  - sender matches user
  - function id matches
  - type args match APOLLO_USDC_TYPE
  - vault addr matches configured vault
- Backend updates local collateral state AFTER verification

Position Sync:
- Operator key signs:
  - upsert_position<USDC>(vault_addr, trader, size, entry_price, margin, ts)
  - close_position<USDC>(vault_addr, trader)

Idempotency requirement:
- close_position MUST be safe to call even if position does not exist (no abort)

Rollback requirement:
- When matching triggers on-chain updates:
  - take engine snapshot
  - apply local match
  - perform on-chain updates
  - if any on-chain update fails: restore snapshot

---

## REST API Contract

Auth:
- POST /auth/challenge -> { nonce }
- POST /auth/verify -> { token }

Faucet (testnet/devnet only):
- POST /faucet { user } -> { minted?: true, alreadyFunded?: true }

Trading:
- POST /order
  body: { user, market, side, price, size }
  returns: { ok, order, matches, onchainUpdates }

Reads:
- GET /orderbook?market=...
- GET /positions?user=...
  returns:
  { user, account: { collateral, locked }, positions: [...] }
- GET /marks
  returns:
  { marks: { [market]: { price, t } } }
- GET /markHistory?market=...&limit=...
- GET /trades
- GET /liquidations

Health:
- GET /health/hl -> { connected, lastTickMsAgo, markets }

---

## WebSocket Message Contract

WS URL:
- wss://<host>/ws (or ws:// for local dev)

Messages sent from server:
- { type: "markPrice", market, price, t }
- { type: "orderbook", market, bids, asks, mid, t }
- { type: "trade", market, price, size, maker, taker, t }
- { type: "positionUpdate", user, account, positions, t } (optional)

Client subscription (optional):
- { type: "subscribe", market }

---

## Frontend Integration Rules

- UI components under src/ui are presentational only.
- Networking lives in src/services and src/hooks.
- Default base URLs if env vars unset:
  - API = `${window.location.origin}/api`
  - WS  = `${window.location.origin.replace("http","ws")}/ws`

Charts:
- Prefill history from:
  1) localStorage cache `apollo:markHistory:<market>`
  2) GET /markHistory
- Append live ticks from WS markPrice events
- Keep last N=300 points
- Dedupe by timestamp bucket (1s)

---

## Acceptance Criteria (Definition of Done)
- Deposit shows collateral in UI
- Withdraw works and updates collateral
- Orders rest and appear in orderbook
- Crossing orders fill and create positions
- On-chain sync does not abort on close_position when position missing
- Mark ticks stream and chart updates live with prefilled history
- Build passes and demo works from a single origin URL
