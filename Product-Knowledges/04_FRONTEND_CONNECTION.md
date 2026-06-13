# 04 — Frontend Connection (the Admin Panel side)

This file documents how the **React admin panel** at
`D:\Admin-panel\Victory_Admin_Pannel` talks to the Strategy Engine. We look
at: the URL configuration, the axios layer, the service layer, the hooks,
the actual page components, and the WebSocket for live signals. By the end
you should be able to trace any user action in the browser all the way
across the wire.

If you haven't read 02 (architecture) and 03 (backend walkthrough), do that
first — this file assumes you know what `useStrategies` is supposed to call.

---

## 1. The frontend's home for Strategy-Engine code

```
src/
├── apiurl.js                            # base URL for the MAIN backend (3002)
│                                        # — Strategy Engine URL is in .env
├── services/
│   ├── api.js                           # axios instance for the MAIN backend
│   ├── strategyEngineApi.js             # axios instance for STRATEGY ENGINE
│   └── strategyEngineService.js         # the function map (REST endpoints)
├── hooks/
│   ├── useApi.js                        # shared loading/error wrapper
│   ├── useStrategies.js                 # strategy CRUD + backtest
│   └── useSignals.js                    # signal list + WebSocket
├── pages/
│   ├── Strategies/
│   │   ├── Strategies.jsx               # list + create/edit/delete modals
│   │   ├── StrategyDetail.jsx           # one strategy's detail + signal history
│   │   └── BacktestPage.jsx             # full-screen backtest page
│   └── Signals/Signals.jsx              # live signal feed
├── components/
│   └── strategies/
│       ├── StrategyForm/StrategyForm.jsx     # 3-step wizard (Basic / Conditions / Review)
│       ├── ConditionBuilder/ConditionBuilder.jsx
│       ├── ConditionRow/ConditionRow.jsx
│       ├── StrategyCard/StrategyCard.jsx
│       ├── StrategyStats/StrategyStats.jsx
│       ├── SignalCard/SignalCard.jsx
│       └── BacktestModal/BacktestModal.jsx   # old modal, kept unused
├── constants/strategy.js                # fallback dropdown options
├── ws/marketWs.js                       # MAIN backend price WS (not engine!)
└── store/marketPrices.js                # live LTP cache (also main backend)
```

The **Strategy Engine has its own dedicated axios instance** so it cannot
accidentally hit the main backend, and the two URLs can be configured
independently.

### The data flow (every strategy/signal screen follows this)

```
   ┌─────────────────────────────────────────────────────────────┐
   │ PAGE         pages/Strategies/Strategies.jsx                 │
   │              (renders cards, modals, toasts, pagination)     │
   └──────────────────────────┬──────────────────────────────────┘
                              │ calls
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ HOOK         hooks/useStrategies.js                          │
   │              (holds list/total state, wraps loading/error    │
   │               via useApi)                                    │
   └──────────────────────────┬──────────────────────────────────┘
                              │ calls
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ SERVICE      services/strategyEngineService.js               │
   │              (one function per REST endpoint — no React)     │
   └──────────────────────────┬──────────────────────────────────┘
                              │ calls
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ AXIOS        services/strategyEngineApi.js                   │
   │              (baseURL, withCredentials cookie, interceptors  │
   │               that unwrap to response.data)                  │
   └──────────────────────────┬──────────────────────────────────┘
                              │ HTTPS + cookie
                              ▼
                  Strategy Engine  :3003 /api/...
```

---

## 2. URL configuration

### Main backend URL — hard-coded in `src/apiurl.js`

```js
export const REACT_APP_API_BASE_URL = 'http://192.168.1.3:3002/'
export const wsUrl = "192.168.1.3:3002";
```

This is **not** read from `.env` — it's hand-edited (one line is commented
out for prod). That's the documented gotcha: before debugging "can't reach
backend", check this file first.

### Strategy Engine URL — read from `.env`

```js
// strategyEngineApi.js
export const STRATEGY_ENGINE_URL = process.env.REACT_APP_STRATEGY_ENGINE_URL
                                || 'http://localhost:3003/';
```

Create React App injects `REACT_APP_*` env vars at build time. Set
`REACT_APP_STRATEGY_ENGINE_URL=http://192.168.1.3:3003/` in `.env` to point
at a remote engine.

---

## 3. The axios instance for the Strategy Engine

`src/services/strategyEngineApi.js`:

```js
const strategyApi = axios.create({
    baseURL: `${STRATEGY_ENGINE_URL}api`,   // → http://.../api
    timeout: 30000,
    withCredentials: true,                  // send cookies cross-origin
});
```

(An **interceptor** is just a function axios runs **automatically** on every
request or every response — like a checkpoint the data passes through on its
way out or back in.)

**Request interceptor** — only one tweak: if the body is `FormData` (the
format used for file uploads), delete the `Content-Type` header so the
browser can fill it in correctly itself. (No file uploads in this engine
today, but defensive.)

**Response interceptor** — two important behaviours:

1. **Unwrap to `response.data`** — services receive the backend envelope
   (`{ success, data, meta }`) directly, NOT the axios `{ data: ... }`
   wrapper. So `useStrategies` does `response.data` to get the list and
   `response.meta?.total` for the count — those are backend-level fields,
   not axios fields.
2. **Normalize errors** — every failure becomes:
   ```js
   { message, status, data }
   ```
   pulling the real message from `error.response.data.error.message` or
   falling back. ECONNABORTED → "Request timeout. Please try again."

### Auth pathway

The browser was logged in via `main-backend:3002/api/login` and received an
HttpOnly cookie. Because `withCredentials: true` is set on both axios
instances and CORS is allowed on the engine, the cookie flows
automatically on every Strategy Engine request. Nothing extra to wire.

---

## 4. The service layer — `strategyEngineService.js`

A thin object with one function per REST endpoint:

```js
export const strategyEngineService = {
    // Config
    getStrategyOptions:     ()      => strategyApi.get('/config/strategy-options'),

    // Strategies
    getStrategies:          (page=1, limit=10, filters={}) =>
                            strategyApi.get('/strategies', { params: { page, limit, ...filters }}),
    getStrategyById:        (id)    => strategyApi.get(`/strategies/${id}`),
    createStrategy:         (data)  => strategyApi.post('/strategies', data),
    updateStrategy:         (id, d) => strategyApi.put(`/strategies/${id}`, d),
    deleteStrategy:         (id)    => strategyApi.delete(`/strategies/${id}`),
    toggleStrategy:         (id)    => strategyApi.patch(`/strategies/${id}/toggle`),
    backtestStrategy:       (id, startDate, endDate, symbol) => {
                            const body = { start_date: startDate, end_date: endDate };
                            if (symbol) body.symbol = symbol;
                            return strategyApi.post(`/strategies/${id}/backtest`, body);
                            },

    // Signals
    getSignals:             (page=1, limit=20, filters={}) =>
                            strategyApi.get('/signals', { params: { page, limit, ...filters }}),
    getSignalsByStrategy:   (id, page=1, limit=20) =>
                            strategyApi.get(`/signals/strategy/${id}`, { params: { page, limit }}),

    // Indicators
    getLatestIndicators:    (tf, sym) => strategyApi.get(`/indicators/${tf}/${sym}`),
};
```

**Why a service object?** Decoupling. The hooks know nothing about URLs;
the service knows nothing about React. Renaming an endpoint = edit this
one file.

---

## 5. The `useApi` hook — shared loading/error

`hooks/useApi.js`:

```js
const useApi = () => {
    const [loading, setLoading] = useState(false);
    const [error, setError]     = useState(null);

    const executeRequest = useCallback(async (apiFunction, ...args) => {
        setLoading(true); setError(null);
        try { return await apiFunction(...args); }
        catch (err) { setError(err); throw err; }
        finally { setLoading(false); }
    }, []);

    return { loading, error, executeRequest, clearError };
};
```

Anyone calling `executeRequest(someService, arg1, arg2)` gets:
- automatic `loading` ON during the call, OFF after,
- automatic `error` capture (still re-thrown so the caller can `await` it),
- a `clearError()` to dismiss banners.

---

## 6. `useStrategies` — the strategy CRUD hook

```js
export const useStrategies = () => {
    const { loading, error, executeRequest, clearError } = useApi();
    const [strategies, setStrategies] = useState([]);
    const [total, setTotal] = useState(0);
    const [selectedStrategy, setSelectedStrategy] = useState(null);

    const fetchStrategies = useCallback(async (page=1, limit=10, filters={}) => {
        const response = await executeRequest(strategyEngineService.getStrategies, page, limit, filters);
        setStrategies(response.data || []);
        setTotal(response.meta?.total || 0);
        return response;
    }, [executeRequest]);
    // ... fetchStrategyById, createStrategy, updateStrategy, deleteStrategy,
    //     toggleStrategy, backtestStrategy, clearSelectedStrategy
};
```

### How a page uses it

`pages/Strategies/Strategies.jsx`:

```js
const { strategies, total, loading, error,
        fetchStrategies, createStrategy, updateStrategy,
        deleteStrategy, toggleStrategy, clearError } = useStrategies();

useEffect(() => {
    fetchStrategies(page, limit).then(...).catch(...);
}, [page, limit, fetchStrategies]);

const handleCreate = async (data) => {
    try { await createStrategy(data); /* close modal, toast, re-fetch */ }
    catch (err) { toast.error(err.message || 'Failed'); }
};
```

The `useStrategies` hook hides loading/error completely from the page;
the page just deals with **toasts, modals, and pagination**.

---

## 7. `useSignals` — list **and** WebSocket

`hooks/useSignals.js` does two jobs in one hook:

### 7a. The list (REST)
- `fetchSignals(page, limit, filters)` → GET `/signals`.
- `fetchSignalsByStrategy(strategyId, page, limit)` → GET `/signals/strategy/:id`.

### 7b. The live feed (WebSocket)

```js
const connectWebSocket = useCallback(() => {
    if (reconnectTimerRef.current) { clearTimeout(...); }   // cancel pending reconnect
    const wsUrl = STRATEGY_ENGINE_URL.replace('http', 'ws') + 'ws';  // http→ws / https→wss
    const ws = new WebSocket(wsUrl);                        // browser sends cookie automatically

    ws.onopen    = () => setWsConnected(true);
    ws.onmessage = (event) => {
        const signal = JSON.parse(event.data);
        if (signal.type === 'NEW_SIGNAL') {
            setSignals(prev => [signal.data, ...prev]);     // PREPEND so newest is on top
            setTotal(prev => prev + 1);
        }
    };
    ws.onclose = () => {
        setWsConnected(false);
        reconnectTimerRef.current = setTimeout(connectWebSocket, 3000);  // auto-reconnect
    };
    ws.onerror = (err) => ws.close();
    wsRef.current = ws;
}, []);

const disconnectWebSocket = useCallback(() => {
    clearTimeout(reconnectTimerRef.current); reconnectTimerRef.current = null;
    if (wsRef.current) {
        wsRef.current.onclose = null;       // ← prevent the auto-reconnect on intentional close
        wsRef.current.close();
        wsRef.current = null;
    }
    setWsConnected(false);
}, []);
```

### How the page uses it

`pages/Signals/Signals.jsx`:

```js
useEffect(() => { connectWebSocket(); return () => disconnectWebSocket(); }, ...);
```

That's it — connect on mount, clean up on unmount. The hook handles every
network detail.

```
   Signals page MOUNTS
        │
        ▼
   connectWebSocket() ──► ws open ──► onmessage: prepend NEW_SIGNAL to list
        ▲                                 │
        │                                 │ server/network drops the socket
        │                                 ▼
        └──── setTimeout 3s ◄──────── onclose (auto-reconnect)

   Signals page UNMOUNTS
        │
        ▼
   disconnectWebSocket():
        clearTimeout(reconnect)        ← stop the 3s retry
        ws.onclose = null              ← so close() does NOT trigger a reconnect
        ws.close()                     ← clean, no zombie socket left behind
```

### Gotchas (very important)
- **String replace `http` → `ws`** also turns `https` → `wss` (because the
  `s` is the next character). Tested: works.
- The reconnect timer is held in a ref, NOT state, because changing it must
  not trigger a re-render.
- Setting `ws.onclose = null` before `.close()` is **the only way** to
  cancel the automatic reconnect (otherwise unmounting the page would
  trigger one final reconnect that never gets cleaned up — a "zombie"
  socket).

---

## 8. The pages

### 8a. `pages/Strategies/Strategies.jsx`

Just a list page:
1. On mount → `fetchStrategies(page, limit)`.
2. `StrategyStats` at the top (total / active / signals today — partly
   placeholder).
3. Grid of `StrategyCard`s, each with toggle / edit / delete buttons.
4. `FullscreenModal` for create/edit (`StrategyForm` inside).
5. `Modal` for delete confirmation.
6. Pagination (1..5 page buttons + prev/next).

### 8b. `pages/Strategies/StrategyDetail.jsx`

Shows one strategy and its history:
1. Reads `:id` via `useParams()`.
2. `fetchStrategyById(id)` → `selectedStrategy`.
3. `fetchSignalsByStrategy(id, page, limit)` for the signal history list.
4. **Defensive parsers** for `trailing_stop_loss` and `target` (mirroring
   the backend ranges). The pattern matches the engine's `parseX` functions
   exactly — a malformed row never crashes the page.
5. Renders per-symbol chips (`TRIGGERED` or `IDLE` dot) from
   `strategy.symbol_states` (the rich array returned by `findById`).
6. Hosts the **Backtest config card** (date pickers + symbol picker). Click
   "Run Backtest" → `navigate('/strategies/:id/backtest', { state })`. The
   target page reads `location.state` and auto-runs.

### 8c. `pages/Strategies/BacktestPage.jsx`

Full-page replacement of an older modal. Logic highlights:
1. Initial dates: from `navState` if present, else last 7 days → today.
2. Symbol selection: `availableSymbols` = strategy's `symbols` array (or
   wrap legacy `symbol`). One must be selected.
3. On Run → `backtestStrategy(id, startDate, endDate, selectedSymbol)`.
4. Renders the result: summary, P&L card, matches table with sticky
   header / sticky first column (so wide indicator-column sets scroll
   horizontally without losing the row label).
5. Re-runs: same dates kept; pencil button re-opens the date card.

### 8d. `pages/Signals/Signals.jsx`

The live feed:
1. On mount → `fetchSignals(page, limit, filters)`.
2. On mount → `connectWebSocket()`.
3. WS-connected dot in the header for at-a-glance status.
4. Filter bar: symbol, action_type, status, signal_type.
5. List of `SignalCard`s. Pagination at the bottom.

### 8e. Components used

- **`StrategyForm`** — 3-step wizard:

  ```
   ┌── Step 1 ──┐     ┌── Step 2 ─────────────┐     ┌── Step 3 ──┐
   │ Basic Info │ ──► │ Conditions            │ ──► │  Review    │
   │ • name     │     │ • entry conditions    │     │ • summary  │
   │ • symbols  │     │ • exit conditions     │     │ • Save     │
   │   (chips,  │     │ • position sizing     │     │   Draft    │
   │   cap 20)  │     │ • trailing stop loss  │     │ • Save &   │
   │            │     │ • target / brokerage  │     │   Activate │
   └────────────┘     └───────────────────────┘     └────────────┘
        canGoNext() gates each step; values stay STRINGS while typing,
        coerced to numbers (malformed → null) only at submit.
  ```

  Basic Info (name + symbols), Conditions (entry+exit conditions + sizing +
  TSL + target + brokerage), Review (summary + Save Draft / Save & Activate
  buttons). All values stay strings while typing; final submission coerces to
  numbers and **nulls malformed configs** so the backend never sees a
  half-filled object.
  Symbol cap (20), TSL ranges, target ranges, brokerage ranges — all the
  backend Joi limits are mirrored as constants here (commented as such).
- **`ConditionBuilder`** — list of `ConditionRow`s with an Add button.
- **`ConditionRow`** — the actual per-condition UI: Category → Variable →
  TF → Operator → Compare To. Switches between Time/HH:MM input,
  Static/Indicator value, AND/OR logic chip.
- **`StrategyCard`** — visible badge (Active / Inactive), action (BUY/SELL),
  symbol list, last activity, action buttons. Used in the grid.
- **`SignalCard`** — colour-coded BUY (green) / SELL (red), ENTRY/EXIT
  badge, symbol, price, strategy name, relative time, status.

---

## 9. The independent "market prices" pipeline (not used by Strategy Engine)

The frontend ALSO opens a different WebSocket to the **main backend**
(`/ws/prices`) for live LTP (last-traded price) data — used by other pages
in the panel.

- `src/ws/marketWs.js` — singleton WS, opens once on App startup.
- `src/store/marketPrices.js` — plain object cache `{ SYMBOL: { price, ts } }`.
- `src/hooks/useRealtimePrices.js` — subscribes, buffers, flushes every
  400 ms, returns `{ prices }`.

**This is unrelated to the Strategy Engine** — it lives in the same browser
process but talks to port 3002, not 3003. Strategy Engine signals come over
their own `useSignals` WS connection (described above) to port 3003.

This separation is intentional: market data is high-frequency / shared
across many pages; signals are low-frequency / per-user. They have
different "fan-out" (how many places the data is sent to), different auth
scopes, and different latency (delay) needs.

```
   The browser holds TWO separate WebSocket connections:

   ┌──────────────────────────────────────────────────────────────┐
   │ BROWSER                                                       │
   │   marketWs (singleton, opened once in App.js)                 │
   │        └────────────────► MAIN backend :3002 /ws/prices       │
   │           live LTP for many pages (high frequency, shared)    │
   │                                                               │
   │   useSignals WS (opened per Signals page mount)               │
   │        └────────────────► STRATEGY engine :3003 /ws           │
   │           NEW_SIGNAL pushes (low frequency, per-user)         │
   └──────────────────────────────────────────────────────────────┘
```

---

## 10. How a click flows — final example

User clicks **Pause** on `StrategyCard`:

```
StrategyCard onClick={() => onToggle()}
   ↓
Strategies.jsx handleToggle(strategy)
   ↓
useStrategies.toggleStrategy(strategy.id)
   ↓
strategyEngineService.toggleStrategy(id)
   ↓
strategyApi.patch(`/strategies/${id}/toggle`)
   ↓ HTTPS PATCH http://localhost:3003/api/strategies/123/toggle
                  (cookie attached automatically)
   ↓
[Strategy Engine] auth → ctrl.toggle → strategyModel.toggleActive → forceResetTriggerState → findById
   ↓ 200 OK { success:true, data: { ...strategy, symbol_states: [...] }}
   ↓
response unwrapped → executeRequest resolves → toast.success → fetchStrategies(page, limit)
```

That single click thus does **three HTTP calls** total: 1 PATCH + 1 GET
(re-fetch). The user only ever sees a toast and an updated card.

---

**Next:** [05_DATABASE.md](05_DATABASE.md)
