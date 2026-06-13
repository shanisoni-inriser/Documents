# 05 — The Database (every table, every column, who writes what)

All Strategy Engine data lives in **`victory_market_db`** (PostgreSQL).
This file documents the five table families the engine touches:

| Table | Owner | Engine's access |
|-------|-------|-----------------|
| `strategies` | **Strategy Engine** | full read/write |
| `strategy_symbols` | **Strategy Engine** | full read/write |
| `signals` | **Strategy Engine** | insert + read |
| `"OHLC_1MIN"` … `"OHLC_1HOUR"` | co-developer's pipeline | **read-only** |
| `feature_table_1min` … `feature_table_1hour` | co-developer's pipeline (historical) | read + **upsert live rows only** |

(The `victory_db` database — users, permissions, billing — belongs to the
main backend. The engine has a pool for it (`appDb.js`) but currently
never queries it.)

---

## 1. `strategies` — one row per strategy

Created by `migrations/002_create_strategies.sql` and extended over time.

| Column | Type | Meaning |
|--------|------|---------|
| `id` | SERIAL PK | strategy id (used in URLs: `/strategies/:id`) |
| `user_id` | INTEGER | owner — **every query filters on this** (from JWT, never from client body) |
| `name` | TEXT | display name, e.g. "SMA Crossover RELIANCE" |
| `symbol` | TEXT | **legacy** primary symbol. Kept in sync = `symbols[0]`. The real list lives in `strategy_symbols` |
| `is_active` | BOOLEAN, default false | OFF = engine ignores it. New strategies start inactive |
| `action` | TEXT, default 'BUY' | BUY = long strategy, SELL = short strategy |
| `timeframe` | TEXT, default '5m' | **fallback** timeframe — only used when a condition doesn't carry its own `variableTimeframe` |
| `conditions` | JSONB, default '[]' | entry conditions array (the shape in topic 10.5) |
| `exit_conditions` | JSONB, default '[]' | exit conditions array (optional) |
| `alert_or_trade` | TEXT, default 'Alert' | Alert = notify only; Trade = (future) auto-execute |
| `last_trigger` | TEXT, default 'Never' | legacy display string |
| `trigger_state` | TEXT, default 'IDLE' | **legacy** — superseded by per-symbol state in `strategy_symbols` |
| `trigger_count` | INTEGER, default 0 | legacy counter (same note) |
| `last_triggered_at` | TIMESTAMPTZ | legacy timestamp (same note) |
| `position_sizing` | JSONB nullable | `{type:'fixed_quantity'\|'fixed_capital', value}` or NULL = not configured |
| `brokerage_config` | JSONB nullable | `{type:'percent', buy_pct, sell_pct}` or NULL |
| `trailing_stop_loss` | JSONB nullable | `{type:'fixed'\|'percent', value}` or `{type:'stepped', initial_stop_pct, hurdle_pct, step_pct}` or NULL |
| `target` | JSONB nullable | `{type:'percent'\|'price', value}` or NULL |
| `created_at` / `updated_at` | TIMESTAMPTZ | bookkeeping; `updated_at = NOW()` set by the update model |

**Indexes:** `user_id`; partial indexes on `is_active = true` (plain and by
symbol) so the engine's hot lookups stay fast as inactive strategies pile up.

**NULL semantics for the four JSONB configs** (this rule repeats everywhere):
- `undefined` in an update payload → leave the column alone,
- `null` → clear it ("not configured"),
- object → set it.

---

## 2. `strategy_symbols` — the junction table + per-symbol state machine

One row per (strategy, symbol) pair. This is where the **live trading state**
actually lives. There is no migration file for the base table (it was added
after 002/003); `migrateTsl.js` added the four TSL columns.

| Column | Type | Meaning |
|--------|------|---------|
| `strategy_id` | INTEGER | FK → strategies.id |
| `symbol` | TEXT | e.g. 'RELIANCE'. Unique together with strategy_id (`ON CONFLICT (strategy_id, symbol)`) |
| `trigger_state` | TEXT | **'IDLE'** (waiting for entry) or **'TRIGGERED'** (position open). The edge-trigger state machine |
| `trigger_count` | INTEGER | how many times this symbol has fired ENTRY (incremented atomically on transition) |
| `last_triggered_at` | TIMESTAMPTZ | when the last ENTRY fired |
| `tsl_extreme_price` | NUMERIC(18,4) | best price since entry — highest for long, lowest for short. Ratcheted with GREATEST/LEAST inside the UPDATE |
| `tsl_entry_price` | NUMERIC(18,4) | the entry close, seeded on IDLE→TRIGGERED. Needed by stepped TSL and percent targets |
| `tsl_step_count` | INT default 0 | how many steps a stepped TSL has climbed. Guard `tsl_step_count < $3` means it only moves forward |
| `tsl_current_stop_price` | NUMERIC(18,4) | the latest computed stop level (for display/debug) |
| `last_tsl_updated_at` | TIMESTAMPTZ | when the stop last stepped |

**State transitions and who performs them:**

| Transition | SQL guard | Performed by |
|------------|----------|--------------|
| IDLE → TRIGGERED | `WHERE trigger_state='IDLE'` | `ConditionEngine` on entry match |
| TRIGGERED → IDLE | `WHERE trigger_state='TRIGGERED'` | `ConditionEngine` (TSL/target/exit), or controller's Policy B |
| any → IDLE (all symbols) | no guard | `forceResetTriggerState` on toggle |

All TSL columns are cleared on every reset (NULL/0) so the next entry starts
fresh.

```
   One strategy row + N strategy_symbols rows = N independent state machines:

   strategies(id=42, action=BUY)
        │
        ├── strategy_symbols(42, 'RELIANCE')  state: TRIGGERED  ← position open
        │        tsl_entry_price=2500.50  tsl_extreme_price=2600.00  step=3
        │
        ├── strategy_symbols(42, 'TCS')       state: IDLE       ← waiting
        │        tsl_* = NULL/0
        │
        └── strategy_symbols(42, 'INFY')      state: IDLE       ← waiting

   RELIANCE can be in a trade while TCS and INFY are not — the state MUST
   live on the junction row, not on the strategy row.
```

**Why per-symbol state?** One strategy can watch up to 20 symbols. RELIANCE
may be TRIGGERED while TCS is IDLE — they are independent positions, so the
state must live on the junction row, not the strategy row. (The
`trigger_state` column still on `strategies` is the old single-symbol relic.)

---

## 3. `signals` — every signal ever fired (append-only history)

Created by `migrations/003_create_signals.sql`.

| Column | Type | Meaning |
|--------|------|---------|
| `id` | SERIAL PK | |
| `strategy_id` | INTEGER | which strategy fired it (LEFT JOINed for `strategy_name` in the feed) |
| `user_id` | INTEGER | owner — the WS broadcast and feed queries key on this |
| `symbol` | TEXT | which symbol fired |
| `action_type` | TEXT | 'BUY' or 'SELL'. For EXIT signals this is the **flip** of the strategy action |
| `signal_type` | TEXT, default 'ENTRY' | 'ENTRY' or 'EXIT' |
| `signal_time` | TIMESTAMPTZ | when the engine fired it (`new Date()` at creation) |
| `trigger_price` | DOUBLE PRECISION | the driver-timeframe candle close at fire time (`?? null` so 0 survives) |
| `condition_snapshot` | JSONB, default '{}' | **the audit trail** — see below |
| `status` | TEXT, default 'pending' | 'pending' → (future execution module) 'executed' |
| `created_at` | TIMESTAMPTZ | insert time |

**Indexes:** `(user_id, created_at DESC)` — the feed query;
`(strategy_id, created_at DESC)` — the per-strategy history; `signal_type`.

### `condition_snapshot` shapes (one per fire reason)

```jsonc
// Normal ENTRY / condition-driven EXIT:
{ "conditions": [...], "indicator_values": { "5m": { "current": {...}, "recent": [...] } } }

// TSL exit:
{ "triggered_by": "TSL", "tsl_type": "stepped", "trail_pct": "stepped",
  "peak_price": 110.25, "stop_level": 104.5, "exit_price": 104.1 }

// Target exit:
{ "triggered_by": "TARGET", "target_type": "percent", "target_value": 5,
  "target_level": 105.0, "exit_price": 105.2 }

// Policy B (strategy edited while a position was open):
{ "triggered_by": "STRATEGY_EDITED", "reason": "...", "previous_action": "BUY" }
```

So you can always answer "**why** did this signal fire?" from the row itself,
even months later. Signals are deliberately preserved when a strategy is
deleted (the UI says so: "Signal history will be preserved").

---

## 4. `"OHLC_1MIN"` … `"OHLC_1HOUR"` — candles (pipeline-owned)

One table per timeframe, **created and written by the co-developer's
pipeline**. The engine never writes them. Identifiers are capitalised and
must be double-quoted in SQL.

| Column (as used by the engine) | Meaning |
|--------------------------------|---------|
| `"Symbol"` | e.g. 'RELIANCE' |
| `"Time"` | candle close timestamp |
| `"Open"`, `"High"`, `"Low"`, `"Close"` | prices |
| `"Volume"` | traded volume |

Engine reads:
- `getOhlcCandles` — last N candles (for TA-Lib input and live trigger prices),
- the `LEFT JOIN` in `getHistoricalRange` — to attach a close price to each
  indicator row for backtests.

Mapping timeframe → table lives ONLY in `constants.js` → `TIMEFRAME_MAP`.

---

## 5. `feature_table_1min` … `feature_table_1hour` — indicator values

One table per timeframe. **Shared ownership:**
- Historical rows: written by the co-developer's pipeline (read-only for us).
- The latest live row: **upserted by `IndicatorService`** after each candle
  close (`ON CONFLICT (symbol, "timestamp") DO UPDATE`).

| Column (as used) | Meaning |
|------------------|---------|
| `symbol` | lowercase identifier this time (our convention) |
| `"timestamp"` | candle close time — quoted because `timestamp` is a SQL type name. **It is NOT called `ts`** (a documented gotcha) |
| `sma_10`, `sma_20` | the two live indicator columns |
| (future: `rsi_14`, `ema_20`, …) | will appear as the pipeline adds columns |

Engine reads:
- `getLatest` / `getRecent(limit 2)` — live evaluation (current + previous
  row for cross-over operators),
- `getHistoricalRange` — backtest data.

---

## 6. Entity relationship picture

```
            ┌──────────────────────┐
            │      strategies      │
            │ id (PK)              │
            │ user_id              │──── ties to users in victory_db (logically,
            │ conditions JSONB     │     no FK across databases)
            │ trailing_stop_loss   │
            │ target / sizing /    │
            │ brokerage  JSONB     │
            └─────────┬────────────┘
                      │ 1
                      │
                      │ N
        ┌─────────────▼──────────────┐         ┌────────────────────┐
        │      strategy_symbols      │         │      signals       │
        │ strategy_id (FK)           │         │ id (PK)            │
        │ symbol                     │         │ strategy_id (FK)   │◄─── N:1 to strategies
        │ trigger_state IDLE/TRIG    │         │ user_id            │
        │ trigger_count              │         │ symbol             │
        │ tsl_extreme_price          │         │ action_type        │
        │ tsl_entry_price            │         │ signal_type        │
        │ tsl_step_count             │         │ trigger_price      │
        │ tsl_current_stop_price     │         │ condition_snapshot │
        └────────────┬───────────────┘         └────────────────────┘
                     │  (symbol matches by VALUE, no FK)
                     ▼
   ┌──────────────────────────────┐   ┌──────────────────────────────┐
   │ "OHLC_5MIN" (per timeframe)  │   │ feature_table_5min (per tf)  │
   │ "Symbol" "Time"              │◄──┤ symbol "timestamp"           │
   │ "Open" "High" "Low" "Close"  │   │ sma_10 sma_20 ...            │
   │ "Volume"                     │   │ (joined on symbol+timestamp) │
   └──────────────────────────────┘   └──────────────────────────────┘
```

---

## 7. Which query runs when (cheat sheet)

| Moment | Queries (in order) |
|--------|--------------------|
| Candle closes | `getOhlcCandles` (TA-Lib input) → `upsert` (write SMA) → `findActiveBySymbol` → `getLatest`+`getRecent` per timeframe → state-machine UPDATEs → `INSERT signals` |
| GET /strategies | `COUNT(*)` + paged SELECT with array_agg |
| GET /strategies/:id | single SELECT with array_agg + json_agg |
| POST /strategies | BEGIN → INSERT strategies → INSERT strategy_symbols×N → COMMIT |
| PUT /strategies/:id | (maybe `findTriggeredSymbols` + `findById`) → BEGIN → dynamic UPDATE → symbol diff DELETE/INSERT → final SELECT → COMMIT → (maybe Policy B: `resetTriggerState` + `getOhlcCandles` + `INSERT signals` per symbol) |
| PATCH toggle | UPDATE is_active → UPDATE all symbols to IDLE → `findById` |
| DELETE | DELETE strategies row |
| POST backtest | `findById` → `getHistoricalRange` per timeframe → (all in memory) |
| GET /signals | `COUNT(*)` + paged SELECT joined to strategies |

---

## 8. Setting up a fresh database

```bash
psql -d victory_market_db -f migrations/002_create_strategies.sql
psql -d victory_market_db -f migrations/003_create_signals.sql
# strategy_symbols: create manually (no migration file); then:
node migrateTsl.js        # adds the TSL columns
# OHLC_* / feature_table_*: created by the co-developer's pipeline — never create by hand
```

**Next:** [06_LIFECYCLES.md](06_LIFECYCLES.md) — full end-to-end stories.
