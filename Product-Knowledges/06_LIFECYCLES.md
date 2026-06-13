# 06 — End-to-End Lifecycles (full stories through the real code)

This final file walks through complete scenarios **naming the exact
functions** at every hop. If you can narrate these six stories from memory,
you have the 100% grip you wanted.

---

## Story 1 — Creating and activating a strategy

**Goal:** an admin wants "BUY RELIANCE & TCS when SMA(10) crosses above
SMA(20) on 5m, exit when SMA(10) crosses below SMA(20), with a 2% trailing
stop and a 5% target, 100 shares per trade."

### Frontend
1. `/strategies` page → "New Strategy" → `FullscreenModal` opens
   `StrategyForm`.
2. `StrategyForm` mounts and immediately calls **two APIs**:
   - `api.get('/internaltoken/list')` (main backend!) — the symbol
     autocomplete list,
   - `strategyEngineService.getStrategyOptions()` → engine
     `GET /api/config/strategy-options` — the indicator dropdown options
     (overrides the hard-coded fallback in `constants/strategy.js`).
3. Step 1: name typed, RELIANCE and TCS added as chips (cap 20).
4. Step 2: two conditions built with `ConditionBuilder`/`ConditionRow`,
   `trailing_stop_loss` = `{type:'percent', value:2}`, `target` =
   `{type:'percent', value:5}`, `position_sizing` =
   `{type:'fixed_quantity', value:100}`. Step gate `canGoNext()` checks all
   blocks are valid-or-absent.
5. Step 3 review → "Save & Activate" → `handleSubmit(true)` coerces
   string inputs to numbers, collapses anything malformed to `null`, sets
   `is_active: true`, calls `onSubmit(payload)`.
6. `Strategies.handleCreate` → `useStrategies.createStrategy` →
   `strategyEngineService.createStrategy` → `POST /api/strategies`.

### Backend
7. `app.js` middleware → `auth` (JWT from cookie → `req.user`) →
   `validate(createStrategySchema)` (Joi: every condition variable must be a
   supported indicator; TSL/target ranges checked; `symbols` deduped,
   uppercased).
8. `strategies.controller.create` — attaches `user_id: req.user.id`.
9. `strategy.model.create` — transaction:
   - `INSERT INTO strategies (...) RETURNING *` (symbol column = symbols[0],
     configs stringified to JSONB),
   - two `INSERT INTO strategy_symbols` rows (RELIANCE, TCS) — both start
     `trigger_state='IDLE'`,
   - `COMMIT`.
10. `success(res, strategy, 201)`.

### Frontend again
11. Interceptor unwraps, hook resolves, page toasts "Strategy created
    successfully!", modal closes, `fetchStrategies` re-pulls the list — the
    new card shows **Active**.

State now: 1 row in `strategies` (is_active=true), 2 rows in
`strategy_symbols` (both IDLE). The engine will see it on the next candle.

---

## Story 2 — A candle closes and an ENTRY signal fires

**Setup:** the strategy from Story 1 is live. It is 10:30 IST; the
RELIANCE 5-minute candle just completed and SMA(10) crossed above SMA(20).

```
 Pipeline   Redis    candleClose   Indicator    Condition    Signal    signalWs   Browser
    │         │        Handler      Service      Engine      Service     │          │
    │ PUBLISH │          │            │            │           │         │          │
    │ close ─►│─ msg ───►│            │            │           │         │          │
    │         │          │─ compute ─►│            │           │         │          │
    │         │          │     (SMA → upsert)      │           │         │          │
    │         │          │─ evaluate ────────────►│           │         │          │
    │         │          │            │   findActiveBySymbol   │         │          │
    │         │          │            │   conditions match!    │         │          │
    │         │          │            │   transitionToTriggered│         │          │
    │         │          │            │   (CAS IDLE→TRIGGERED) │         │          │
    │         │          │            │            │─ create ─►│         │          │
    │         │          │            │            │      INSERT signal  │          │
    │         │          │            │            │           │─ broadcast ─►│      │
    │         │          │            │            │           │         │─ send ──►│
    │         │          │            │            │           │         │   NEW_SIGNAL
    │         │          │            │            │           │         │   (green card
    │         │          │            │            │           │         │    pops in)
```


1. **Data pipeline** inserts the candle into `"OHLC_5MIN"` and publishes
   Redis `ohlc:candle_close` →
   `{"symbol":"RELIANCE","timeframe":"5m","ts":"..."}`.
2. **`candleCloseHandler`** receives it, parses JSON, checks
   symbol+timeframe present.
3. **`IndicatorService.compute('RELIANCE','5m')`**:
   - `indicator.model.getOhlcCandles` → last 100 candles (reversed to
     ascending),
   - guard: ≥ 20 candles,
   - `talib.execute(SMA, 10)` and `(SMA, 20)` → take `last(...)` of each,
   - `indicator.model.upsert('5m','RELIANCE', ts, {sma_10, sma_20})` →
     `feature_table_5min` now has a fresh row for this candle.
4. **`ConditionEngine.evaluate('RELIANCE')`**:
   - `strategy.model.findActiveBySymbol('RELIANCE')` → our strategy row
     (joined with the RELIANCE junction row: `trigger_state='IDLE'`).
   - `collectTimeframes` → `['5m']` →
     `buildLiveIndicatorMap` → `{ '5m': { current: <new row>, recent: [<prev row>, <new row>] } }`.
   - State is IDLE → `evaluateConditions(entry)`:
     - condition 1: `Crosses Above` → previous sma_10 ≤ previous sma_20 AND
       current sma_10 > current sma_20 → **true**.
   - **Entry path**:
     - `pickDriverTimeframe(['5m'])` → '5m'; latest `"OHLC_5MIN"` close →
       say `2500.50` (the entry/trigger price),
     - `strategy.model.transitionToTriggered(id,'RELIANCE', 2500.50)` —
       atomic CAS `IDLE→TRIGGERED`, seeds `tsl_extreme_price =
       tsl_entry_price = 2500.50`, `trigger_count + 1`. Returns true (we won
       the race).
     - `SignalService.create({ signal_type:'ENTRY', action_type:'BUY',
       trigger_price:2500.50, condition_snapshot:{conditions,
       indicator_values} })`:
       - `signal.model.create` → INSERT, status 'pending',
       - `broadcastFn(user_id, {type:'NEW_SIGNAL', data:signal})` →
         `signalWs.broadcastSignal` → every open socket of that user gets
         the frame.
5. **Browser** (Signals page open): `useSignals` `ws.onmessage` →
   `setSignals(prev => [signal, ...prev])` — the green BUY ENTRY card pops
   in at the top within milliseconds.

Note what did **not** happen: TCS was untouched (its own junction row is
still IDLE and it had no candle close event), and the entry conditions will
now be **ignored** for RELIANCE until the position exits.

---

## Story 3 — The trailing stop loss steps up and finally exits

**Setup:** position open from Story 2. TSL = `{type:'percent', value:2}` →
the stop trails 2% below the running peak.

Each subsequent RELIANCE 5m candle close repeats steps 1–4 above, but now
`trigger_state === 'TRIGGERED'`, so the engine goes down the **exit branch**:

1. `parseTrailingStopLoss` / `parseTarget` re-validate the stored JSONB
   (defense in depth).
2. The candle close is fetched **once** (shared by TSL + target checks):
   say `2550.00`.
3. **TSL block**:
   - `strategy.model.updateTslExtreme(id,'RELIANCE',2550.00,isLong=true)` —
     `SET tsl_extreme_price = GREATEST(COALESCE(extreme, 2550), 2550)` →
     peak "ratchets" up to 2550.00 (ratchet = it can only go up, never back
     down — like a spanner that turns one way only).
   - `calculateTslStopLevel({percent,2}, entry=2500.50, extreme=2550.00,
     long)` → stop = 2550 × 0.98 = **2499.00**.
   - Hit check: `2550.00 ≤ 2499.00`? No → position stays open.
4. **Target block** (target 5% → level = 2500.50 × 1.05 = 2625.53): `2550 ≥
   2625.53`? No.
5. **Exit conditions**: SMA(10) still above SMA(20) → no cross below → no.

…candles keep coming; suppose the peak eventually reaches **2600.00**
(stop = 2548.00), then a red candle closes at **2540.00**:

6. `updateTslExtreme` → GREATEST keeps the peak at 2600.00 (it never goes
   down).
7. Stop = 2600 × 0.98 = 2548.00. Hit check: `2540.00 ≤ 2548.00` → **HIT**.
8. `strategy.model.resetTriggerState(id,'RELIANCE')` — atomic CAS
   `TRIGGERED→IDLE`, clears all TSL columns. Returns true → we own the exit.
9. `SignalService.create({ signal_type:'EXIT', action_type:'SELL',
   trigger_price:2540.00, condition_snapshot:{ triggered_by:'TSL',
   tsl_type:'percent', trail_pct:2, peak_price:2600, stop_level:2548,
   exit_price:2540 }})` → INSERT + WS push.
10. Browser: a red SELL EXIT card appears. The strategy detail page's
    RELIANCE chip flips back to IDLE on next refresh.

The cycle can now repeat — next entry cross will fire a fresh ENTRY.

**Stepped TSL variant:** with `{stepped, initial 2, hurdle 5, step 1}` the
stop starts at entry −2% and doesn't move until profit ≥ 5%; then
`stepCount = floor((profit − hurdle)/step) + 1` and each step lifts the stop
by `step_pct` of entry. `updateTslStepCount` persists each new step with the
forward-only guard, and the `TSL STEP` log line records it.

---

## Story 4 — Target (take-profit) exit

Same setup, but suppose price ran straight up and a candle closes at
**2630.00** (target level 2625.53):

1. TSL: peak 2630, stop 2577.40 — not hit (price is above).
2. Target block (runs because `tslExited` is false):
   - entry price from `strategy.tsl_entry_price` (projected by
     `findActiveBySymbol`) = 2500.50,
   - `calculateTargetLevel({percent,5}, 2500.50, long)` → 2625.53,
   - `2630.00 >= 2625.53` → **HIT** → CAS reset → EXIT signal with
     `triggered_by:'TARGET'`, `trigger_price = 2630.00` (the actual close —
     "Option B" in the comments).
3. Exit-conditions block is skipped (`targetExited = true`).

**Priority recap (every candle, both live and backtest):**
stop loss → target → exit conditions. Risk first, profit second,
discretionary rules last; only one exit can win per candle.

---

## Story 5 — Editing a live strategy (Policy B)

**Setup:** RELIANCE is TRIGGERED (open position). The user edits the
strategy and changes the entry conditions.

```
   PUT /strategies/:id  (rule logic changed)
        │
        ▼
   ruleLogicChanged?  ── no ──► just update, done
        │ yes
        ▼
   findTriggeredSymbols()  →  [RELIANCE]   (snapshot BEFORE the update)
        │
        ▼
   update the strategy (transaction)
        │
        ▼
   for each previously-triggered symbol:
        resetTriggerState (CAS TRIGGERED→IDLE)
             │
             ├─ false ─► live engine ALREADY exited it → emit nothing
             │           (still exactly one EXIT, by construction)
             │
             └─ true ──► emit synthetic EXIT @ latest 1m close
                         condition_snapshot.triggered_by = 'STRATEGY_EDITED'
```


1. Frontend `StrategyForm` (edit mode) pre-fills from the fetched strategy
   (defensively JSON-parsing every JSONB field), user edits, submits →
   `PUT /api/strategies/:id`.
2. `strategies.controller.update`:
   - `ruleLogicChanged = true` (conditions present in payload),
   - `findTriggeredSymbols(id)` → `[ {symbol:'RELIANCE', ...} ]`,
   - `findById` → `oldStrategy` (to know the OLD action),
   - `strategy.model.update` — transaction: dynamic UPDATE + symbol diff.
3. **Policy B loop** over `['RELIANCE']`:
   - `resetTriggerState(id,'RELIANCE')` — CAS. If the live engine exited
     this very second, this returns false and we emit **nothing** (exactly
     one EXIT per position, by construction). Assume true.
   - `getOhlcCandles('RELIANCE','1m',1)` → latest 1m close, say 2575.25.
   - `SignalService.create({ signal_type:'EXIT', action_type:'SELL'
     (flip of old 'BUY'), trigger_price:2575.25, condition_snapshot:
     { triggered_by:'STRATEGY_EDITED', reason:'...', previous_action:'BUY' }})`.
   - Logged as `POLICY_B EXIT: ...`.
4. Response returns the updated strategy. The user's feed shows the
   synthetic EXIT — their position bookkeeping stays consistent even though
   the rules changed underneath it.

**Toggle instead of edit?** `PATCH /:id/toggle` flips `is_active` and calls
`forceResetTriggerState` (all symbols → IDLE, **no** synthetic exits) — a
pause is not an exit recommendation.

---

## Story 6 — Running a backtest

**Setup:** the user opens the strategy detail, picks RELIANCE,
2026-05-10 → 2026-06-09, clicks "Run Backtest".

### Frontend
1. `StrategyDetail.handleRunBacktest` → `navigate('/strategies/:id/backtest',
   { state: { symbol, startDate, endDate, autoRun: true }})`.
2. `BacktestPage` mounts, reads `location.state`, auto-runs:
   `useStrategies.backtestStrategy(id, start, end, symbol)` →
   `POST /api/strategies/:id/backtest` with
   `{ start_date, end_date, symbol }`.

### Backend — `BacktestService.run`

```
   The candle loop is a tiny simulated trader walking the driver candles:

   flat? ──yes──► entry conditions true? ──yes──► open: record ENTRY row
     ▲                                              (compute qty/brokerage),
     │                                              pendingEntry = {...}
     │                                                     │
     │ exit recorded                                       ▼
     │                                            in a trade now
   record EXIT row ◄── first hit wins: ── TSL? ── TARGET? ── EXIT conds? ──┐
   (gross_pl, net_pl)                                                      │
     ▲                                                                     │
     └─────────────────────── none hit ── stay in trade, next candle ◄────┘

   At the end → computePnLSummary(matches): win rate, profit factor, etc.
```

3. `findById(strategyId, userId)` — ownership; symbol must belong to the
   strategy (forged symbols rejected).
4. Date checks: valid, start ≤ end, ≤ 30 days (`MAX_DATE_RANGE_DAYS`).
5. `collectTimeframes(entry + exit conditions)` → e.g. `['5m']`;
   `getHistoricalRange('RELIANCE','5m', '2026-05-10', '2026-06-09 23:59:59.999')`
   → all indicator rows joined with closes, ascending.
6. `driverTf` = smallest timeframe; non-driver timeframes get **pointers**
   that advance while `their ts ≤ driver ts` (the "walk two sorted lists
   together" trick: each driver candle sees the latest completed row of every
   other timeframe — never a future one).
7. Parse `sizing` / `brokerage` / `trailingStopLoss` / `target` once.
8. **The candle loop** (a tiny simulated trader):
   - flat + entry conditions true → compute `quantity`
     (`fixed_quantity` → 100; `fixed_capital` → floor(capital/price); zero →
     record in `skipped_trades`, stay flat) → record ENTRY row with
     `indicator_snapshot`, per-condition `condition_results` trace, and the
     money columns (brokerage, adjusted_price, total_amount) →
     `pendingEntry = { price, quantity, total_amount, extremePrice }`.
   - in trade → update `extremePrice`, then TSL → target → exit conditions
     (same priority as live). Exit records `triggered_by` and the financial
     block from `computeExitFinancials` (gross_pl, net_pl after both legs'
     brokerage).
   - Backtest exits at the **stop/target level** (theoretical price);
     live exits record the actual close — expect small differences.
9. `computePnLSummary(matches, sizing)` — totals, win rate, profit factor
   (null = no losses → frontend shows ∞), largest win/loss, total brokerage.
   Without sizing: `{ configured: false }` — no fake numbers.
10. Response: `{ matches, driverTimeframe, indicatorColumns, summary,
    skipped_trades, pnl_summary }`.

### Frontend again
11. `BacktestPage` renders: summary header pill (date range), the P&L card,
    and the matches table — ENTRY/EXIT rows, per-indicator columns
    (`indicatorColumns` drives the headers), hover tooltips that spell out
    the P&L formula (`buildPnLTooltip`), and a condition-by-condition
    breakdown per row (`condition_results`).

**Key insight:** the backtest and the live engine share
`evaluateConditions` / `evaluateSingle` / `tslMath` / `targetMath` — the
same brain runs in both, so a backtest is a faithful preview of live
behaviour (modulo the exit-price note above).

---

## Mini-glossary (quick reference)

| Term | Meaning here |
|------|--------------|
| Driver timeframe | The smallest timeframe a strategy uses; sets the evaluation rhythm and the trigger price |
| Edge-triggered | Fire on the *change* into true, not on every candle while true |
| CAS | Compare-and-swap — atomic UPDATE with the expected old value in WHERE |
| Junction row | A `strategy_symbols` row: one symbol's state within one strategy |
| Extreme price | Best price since entry (peak for long, trough for short) |
| Hurdle | Profit % that switches a stepped TSL from static to trailing (one-time) |
| Policy B | Synthetic EXIT signals emitted when rule logic is edited on an open position |
| Snapshot | The `condition_snapshot` JSONB recording *why* a signal fired |
| Driver close | The close of the latest driver-timeframe candle, used as trigger price |
| Envelope | The `{ success, data, meta }` response shape every endpoint uses |

---

You've now seen every layer: the topics (01), the map (02), the backend
code (03), the frontend code (04), the database (05), and the live stories
(06). When you read the actual source files now, nothing should surprise
you. Happy trading-engine hacking! 🚀
