# 01 — Every Topic You Need to Learn (with detailed, easy explanations)

This file does two things:

1. **PART A** — a complete checklist of every topic/technology/concept used in
   this backend (and the connected frontend). Use it to track your learning.
2. **PART B** — each topic explained **one by one, in detail, in easy
   language**, always pointing at the **real place in this codebase** where it
   is used, so theory and practice stay connected.

---

# PART A — The Complete Topic Checklist

## Group 1 — JavaScript language (used everywhere)
- [ ] 1.1 CommonJS modules (`require` / `module.exports`)
- [ ] 1.2 `const` / `let`, arrow functions, template literals
- [ ] 1.3 Objects & arrays: destructuring, spread (`...`), `map`/`filter`/`reduce`
- [ ] 1.4 Optional chaining (`?.`) and nullish coalescing (`??`)
- [ ] 1.5 Promises and `async` / `await`
- [ ] 1.6 `try` / `catch` / `finally` and throwing errors
- [ ] 1.7 Classes and inheritance (`class X extends Error`)
- [ ] 1.8 Closures and the singleton pattern
- [ ] 1.9 `JSON.parse` / `JSON.stringify`
- [ ] 1.10 `Map` and `Set`
- [ ] 1.11 Number safety: `Number()`, `NaN`, `Number.isFinite`, `parseInt`
- [ ] 1.12 Timers: `setTimeout`, `setInterval`
- [ ] 1.13 Truthy/falsy and why `||` vs `??` matters for the number `0`

## Group 2 — Node.js runtime
- [ ] 2.1 What Node.js is (event loop, single thread, non-blocking I/O)
- [ ] 2.2 The `http` module and why Express sits on top of it
- [ ] 2.3 `process`: env vars, `process.exit`, PID
- [ ] 2.4 Process signals: SIGTERM / SIGINT and graceful shutdown
- [ ] 2.5 `unhandledRejection` and `uncaughtException`
- [ ] 2.6 npm, `package.json`, dependencies vs devDependencies, nodemon

## Group 3 — Express.js (the web framework)
- [ ] 3.1 What Express is; `app`, routes, handlers
- [ ] 3.2 Middleware and the `next()` chain
- [ ] 3.3 `express.Router()` and route mounting
- [ ] 3.4 `req` (params, query, body, headers, cookies) and `res` (status, json)
- [ ] 3.5 Error-handling middleware (the special 4-argument function)
- [ ] 3.6 Body parsing (`express.json`) and `cookie-parser`
- [ ] 3.7 REST API design: GET/POST/PUT/PATCH/DELETE, status codes
- [ ] 3.8 Pagination (page / limit / total / totalPages)

## Group 4 — Authentication & security
- [ ] 4.1 JWT (JSON Web Token): what it is, its 3 parts, signing & verifying
- [ ] 4.2 Bearer tokens vs cookie tokens
- [ ] 4.3 Sharing one JWT secret between two backends (single sign-on style)
- [ ] 4.4 CORS — what it is, why `credentials: true` + origin allowlist
- [ ] 4.5 Why tokens are NOT allowed in the URL query string for HTTP (but ARE for WebSocket)
- [ ] 4.6 SQL injection and parameterized queries (`$1, $2`)
- [ ] 4.7 Ownership checks (`WHERE user_id = $X` on every query)

## Group 5 — Validation with Joi
- [ ] 5.1 What schema validation is and why we never trust client input
- [ ] 5.2 Joi basics: `Joi.object`, `.string()`, `.number()`, `.required()`, `.valid()`
- [ ] 5.3 `abortEarly: false` and `stripUnknown: true`
- [ ] 5.4 Conditional rules: `Joi.when()` and `Joi.alternatives().try()`
- [ ] 5.5 Validating body vs query string

## Group 6 — PostgreSQL & the `pg` library
- [ ] 6.1 Relational databases, tables, rows, primary keys
- [ ] 6.2 Connection pooling (`pg.Pool`) and why we never open one connection per query
- [ ] 6.3 Parameterized queries (`pool.query(sql, [params])`)
- [ ] 6.4 Transactions: `BEGIN` / `COMMIT` / `ROLLBACK` and why
- [ ] 6.5 `RETURNING *` — getting the row back from INSERT/UPDATE
- [ ] 6.6 UPSERT: `INSERT ... ON CONFLICT ... DO UPDATE`
- [ ] 6.7 `JSONB` columns — storing JSON inside Postgres
- [ ] 6.8 Joins: `LEFT JOIN` vs `INNER JOIN`
- [ ] 6.9 `GROUP BY` + aggregates: `COUNT`, `MAX`, `array_agg`, `json_agg`, `FILTER (WHERE …)`
- [ ] 6.10 `COALESCE`, `GREATEST`, `LEAST`
- [ ] 6.11 Indexes (and partial indexes like `WHERE is_active = true`)
- [ ] 6.12 `SERIAL`, `TIMESTAMPTZ`, `DOUBLE PRECISION`, `NUMERIC`
- [ ] 6.13 `LIMIT` / `OFFSET` pagination
- [ ] 6.14 Quoted identifiers — why `"OHLC_1MIN"` and `"timestamp"` have quotes
- [ ] 6.15 Atomic conditional UPDATE (compare-and-swap) — the race-safety trick
- [ ] 6.16 One service, two databases (`victory_db` vs `victory_market_db`)

## Group 7 — Redis (pub/sub messaging)
- [ ] 7.1 What Redis is; what publish/subscribe means
- [ ] 7.2 Channels and messages (`ohlc:candle_close`)
- [ ] 7.3 The `ioredis` client; why subscriber and publisher are separate connections
- [ ] 7.4 Event-driven architecture — "don't poll, get notified"

## Group 8 — WebSockets (real-time push)
- [ ] 8.1 HTTP vs WebSocket — request/response vs always-open pipe
- [ ] 8.2 The WS handshake ("HTTP upgrade") and why WS shares the HTTP server
- [ ] 8.3 The `ws` library: `WebSocket.Server`, `connection`, `message`, `close`
- [ ] 8.4 Authenticating a WebSocket (token in query string / cookie)
- [ ] 8.5 Per-user connection registry (a `Map` of userId → Set of sockets)
- [ ] 8.6 Heartbeat: ping/pong and killing dead connections
- [ ] 8.7 Custom close codes (e.g. `4401` = auth required)
- [ ] 8.8 Client-side reconnect logic (frontend `useSignals`)

## Group 9 — Logging with Winston
- [ ] 9.1 Log levels (debug / info / warn / error)
- [ ] 9.2 Formats: timestamps, colorized dev output vs JSON production output
- [ ] 9.3 Logging metadata objects and stack traces

## Group 10 — Trading domain knowledge
- [ ] 10.1 OHLC candles (Open, High, Low, Close, Volume) and timeframes
- [ ] 10.2 Candle close — why the engine only acts when a candle completes
- [ ] 10.3 Technical indicators; SMA (Simple Moving Average) specifically
- [ ] 10.4 TA-Lib — the native indicator-calculation library
- [ ] 10.5 Trading strategies & conditions (variable / operator / value)
- [ ] 10.6 Operators: greater/less than, **Crosses Above / Crosses Below**
- [ ] 10.7 Multi-timeframe analysis (one condition on 5m, another on 1h)
- [ ] 10.8 Signals: ENTRY vs EXIT; BUY vs SELL; long vs short
- [ ] 10.9 Edge-triggered state machine: IDLE ↔ TRIGGERED (fire once, not every candle)
- [ ] 10.10 Stop loss: fixed, trailing percent, and stepped trailing
- [ ] 10.11 Target (take profit): percent vs absolute price
- [ ] 10.12 Position sizing: fixed quantity vs fixed capital
- [ ] 10.13 Brokerage & costs; gross P&L vs net P&L
- [ ] 10.14 Backtesting; win rate, profit factor, largest win/loss
- [ ] 10.15 Market time / IST (Indian Standard Time) conditions

## Group 11 — Architecture & design patterns
- [ ] 11.1 Layered architecture: routes → controllers → services → models
- [ ] 11.2 Microservices: why the Strategy Engine is separate from the main backend
- [ ] 11.3 Single source of truth (constants.js)
- [ ] 11.4 Dependency injection (`SignalService.setBroadcast`)
- [ ] 11.5 Singletons (DB pools, Redis clients, the market WS on the frontend)
- [ ] 11.6 Race conditions and atomic compare-and-swap
- [ ] 11.7 Defensive parsing (never trust DB rows blindly)
- [ ] 11.8 Operational vs programmer errors (`isOperational`)
- [ ] 11.9 Graceful shutdown
- [ ] 11.10 "Policy B" — synthetic EXIT signals when a live strategy is edited

## Group 12 — Frontend (React Admin Panel) topics
- [ ] 12.1 React components, props, JSX
- [ ] 12.2 Hooks: `useState`, `useEffect`, `useCallback`, `useMemo`, `useRef`
- [ ] 12.3 Custom hooks (`useApi`, `useStrategies`, `useSignals`)
- [ ] 12.4 axios: instances, `baseURL`, interceptors, `withCredentials`
- [ ] 12.5 React Router: routes, `useParams`, `useNavigate`, `location.state`
- [ ] 12.6 Redux Toolkit (permissions slice) + `ProtectedRoute`
- [ ] 12.7 Browser `WebSocket` API + reconnect pattern
- [ ] 12.8 CSS Modules
- [ ] 12.9 Controlled forms & multi-step wizards (StrategyForm)
- [ ] 12.10 Mirroring backend validation limits on the frontend

---

# PART B — Each Topic Explained in Detail

---

## Group 1 — JavaScript language

### 1.1 CommonJS modules (`require` / `module.exports`)

Node.js splits code into **files (modules)**. A file *exports* things, another
file *imports* them.

```js
// utils/helpers.js  — exports
const last = (arr) => arr[arr.length - 1];
module.exports = { last };

// services/IndicatorService.js  — imports
const { last } = require('../utils/helpers');
```

Key facts:
- `module.exports = X` decides what other files receive when they `require` this file.
- Node **caches** modules: the second `require` of the same file returns the
  **same object**, not a fresh copy. The backend relies on this for singletons
  (one DB pool shared by everyone — see topic 1.8).
- The frontend uses the newer **ES Modules** style instead (`import` /
  `export`) because it is compiled by Create React App. Same idea, different syntax.

**In this codebase:** every backend file uses CommonJS; every frontend file uses ES modules.

### 1.2 `const` / `let`, arrow functions, template literals

- `const x = 5` — variable that cannot be re-assigned. Used by default everywhere.
- `let y` — variable that can be re-assigned (e.g. `let idx = 2;` in
  `strategy.model.js` while building SQL).
- Arrow function: `const f = (a, b) => a + b;` — short function syntax. Inline
  versions appear constantly: `candles.map(r => Number(r.Close))`.
- Template literal: `` `Candle close received: ${symbol} ${timeframe}` `` —
  a string with `${…}` placeholders. Used in all log messages and to build
  dynamic SQL strings.

### 1.3 Destructuring, spread, array methods

**Destructuring** pulls fields out of an object/array in one line:

```js
const { symbol, timeframe, ts } = data;          // object destructuring
const { rows } = await pool.query(sql, params);  // grab .rows from result
```

**Spread (`...`)** copies the contents of an object/array into a new one:

```js
const payload = { ...req.body, user_id: req.user.id }; // copy body + add field
const mainParams = [...params, limit, offset];          // copy array + append
```

**Array methods** transform lists without manual loops:
- `.map(fn)` — converts each item: `candles.map(r => Number(r.Close))` turns DB rows into a number array for TA-Lib.
- `.filter(fn)` — keeps matching items: `tradesPl.filter(p => p > 0)` = only winning trades.
- `.reduce(fn, start)` — folds a list into one value: `arr.reduce((s, v) => s + v, 0)` = sum.
- `.find(fn)` — first match: `TF_ORDER.find(tf => timeframes.includes(tf))` picks the smallest timeframe.
- `.forEach(fn)` — just loop.

### 1.4 Optional chaining (`?.`) and nullish coalescing (`??`)

- `a?.b` — "give me `a.b`, but if `a` is `null`/`undefined`, give me
  `undefined` **instead of crashing**".
  Example from `auth.js`: `req.cookies?.token` — works even if cookies were never parsed.
- `x ?? y` — "use `x` unless it is exactly `null` or `undefined`, then use `y`".

**The classic bug this prevents** (real comment in `signal.model.js`):

```js
data.trigger_price ?? null   // CORRECT: price 0 stays 0
data.trigger_price || null   // WRONG: price 0 is falsy → becomes null!
```

`||` falls back on **any** falsy value (`0`, `''`, `false`), `??` only on
`null`/`undefined`. For prices, `0` is a legitimate value, so `??` is used.

### 1.5 Promises and `async` / `await`

Node does I/O (database queries, network) **asynchronously** — it starts the
work and continues without waiting. A **Promise** is the "receipt" for work
that will finish later.

`async`/`await` is the easy syntax for it:

```js
async function compute(symbol, timeframe) {
    const candles = await indicatorModel.getOhlcCandles(symbol, timeframe);
    // ↑ pause HERE (without blocking the whole server) until rows arrive
    ...
}
```

Rules to remember:
- `await` only works inside an `async` function.
- An `async` function **always returns a Promise**.
- If the awaited thing fails, `await` **throws** — which is why it is wrapped
  in `try/catch`.
- Sequential awaits in a loop run one-by-one (the engine does this on purpose
  to keep DB load predictable).

**What `await` actually does (the event loop):**

```
  CODE                                EVENT LOOP (single thread)
  ────────────────────────            ──────────────────────────────────
  const c = await getCandles()  ──►   start DB query, REMEMBER where we were,
                                      then GO DO OTHER WORK (other requests,
                                      WS messages, the next candle event...)
                                              │
                                      ...DB answers (50 ms later)...
                                              │
  const close = c.map(...)      ◄──   RESUME this function right here,
                                      with `c` now filled in
```

The thread is **never blocked** waiting — that's why one Node process can
serve the REST API, the WebSockets, and the candle processing all at once.

### 1.6 `try` / `catch` / `finally`

```js
try {
    await client.query('BEGIN');
    ...                              // happy path
    await client.query('COMMIT');
} catch (err) {
    await client.query('ROLLBACK');  // undo on any failure
    throw err;                       // re-throw so caller knows
} finally {
    client.release();                // ALWAYS runs — success or failure
}
```

This exact pattern is in `strategy.model.js` `create()` and `update()`.
`finally` is crucial: if we forgot `client.release()`, the connection would
leak and the pool would eventually run dry.

Also note the pattern in `ConditionEngine.evaluate()`: each strategy is
evaluated inside its own `try/catch`, so **one broken strategy cannot stop the
other strategies** from being evaluated.

### 1.7 Classes and inheritance — the error family

`utils/errors.js` defines a small **class hierarchy**:

```js
class AppError extends Error {           // base: knows statusCode, code, isOperational
    constructor(message, statusCode = 500, code = 'INTERNAL_ERROR') {
        super(message);                  // call Error's constructor
        this.statusCode = statusCode;
        this.isOperational = true;       // "expected" error, safe to show user
    }
}
class ValidationError extends AppError { /* 400 VALIDATION_ERROR + details */ }
class NotFoundError   extends AppError { /* 404 NOT_FOUND */ }
class AuthError       extends AppError { /* 401 AUTH_ERROR */ }
class ForbiddenError  extends AppError { /* 403 FORBIDDEN */ }
```

Why classes? So a controller can simply `throw new NotFoundError('Strategy')`
and the central `errorHandler` middleware knows the right HTTP status without
any extra code. `extends` means "is a special kind of" — a `NotFoundError`
**is** an `AppError` **is** an `Error`.

### 1.8 Closures and the singleton pattern

A **closure** = a function that remembers variables from where it was created.
The DB files use it to make a **singleton** (a thing created only once):

```js
let marketPool;                       // lives in module scope (private)
const getMarketPool = () => {
    if (!marketPool) {                // first call → create
        marketPool = new Pool({...});
    }
    return marketPool;                // every later call → same pool
};
```

Combined with module caching (topic 1.1), **every file in the app shares one
pool**. Same trick in `redis.js` (one subscriber, one publisher) and on the
frontend in `marketWs.js` (one WebSocket for the whole app).

```
   strategy.model.js ─┐
   signal.model.js   ─┼──► getMarketPool() ──► [ same single Pool object ]
   indicator.model.js ┘         │                        │
                                first call creates it;    │ 20 reusable
                                later calls return it     ▼ connections
                                                  ┌──────────────────┐
                                                  │ victory_market_db│
                                                  └──────────────────┘
```

### 1.9 `JSON.parse` / `JSON.stringify`

- `JSON.stringify(obj)` → string. Used before saving conditions into the
  `JSONB` column and before sending WebSocket messages.
- `JSON.parse(str)` → object. Used on Redis messages and WS messages.
- `JSON.parse` **throws** on bad input, so it is always wrapped:
  `candleCloseHandler` catches and logs "Invalid candle_close message format"
  instead of crashing.
- `parseConditions()` in ConditionEngine handles **both** cases: the `pg`
  driver usually auto-parses JSONB to objects, but if a string arrives it
  parses manually. This "accept both" defensiveness appears in many places
  (`parseTrailingStopLoss`, `parseTarget`, frontend form prefill).

### 1.10 `Map` and `Set`

- **`Set`** — a bag of unique values. `allTimeframes.add('5m')` twice still
  stores one `'5m'`. Used to deduplicate timeframes, symbols, indicator columns.
  `set.has(x)` is a fast "is it in there?".
- **`Map`** — key → value pairs, better than plain objects when keys are
  dynamic. The WebSocket server keeps
  `userConnections: Map<userId, Set<WebSocket>>` — each user maps to the set of
  their open sockets (a user may have several browser tabs!).

### 1.11 Number safety

Data coming from DB / JSON / user input may be strings or garbage. The
codebase converts and checks **before doing math**:

- `Number(x)` — convert to number ( `Number("12.5")` → `12.5`, `Number("abc")` → `NaN`).
- `NaN` — "Not a Number", the result of failed math. Caution: `NaN === NaN` is **false**; you must use `isNaN()` or `Number.isFinite()`.
- `Number.isFinite(x)` — true only for real usable numbers (not `NaN`, not `Infinity`, not strings). The engine's favorite guard:
  ```js
  if (!Number.isFinite(currentClose) || currentClose <= 0) return; // skip safely
  ```
- `parseInt(str, 10)` — string → integer, always with base 10.
- Rounding money to 2 decimals: `Math.round(n * 100) / 100` (the `round2` helper).

### 1.12 Timers

- `setInterval(fn, 30000)` — run `fn` every 30s. Used for the WebSocket
  heartbeat (server pings clients) and the frontend's 400ms price flush.
- `setTimeout(fn, 10000)` — run once after a delay. Used in graceful shutdown
  ("force exit after 10s") and frontend WS reconnects ("retry in 3s").
- `clearInterval` / `clearTimeout` — cancel them. The frontend carefully
  cancels the reconnect timer to avoid "zombie" reconnections.

### 1.13 Truthy / falsy

Falsy values: `false, 0, '', null, undefined, NaN`. Everything else is truthy.
Used for quick existence checks (`if (!token)`), but **dangerous for numbers**
— see topic 1.4 for why `0` needs `??` instead of `||`.

---

## Group 2 — Node.js runtime

### 2.1 What Node.js is

Node.js runs JavaScript outside the browser. Its superpower is the **event
loop**: a single thread that never waits. When the code does I/O (DB query,
network call), Node hands the work to the operating system, continues running
other code, and comes back when the result is ready (that's what `await`
pauses for).

Consequence: this single process can handle the REST API, the WebSocket
connections, **and** the Redis-driven candle processing all at once — as long
as no code blocks the loop with heavy synchronous work. (The TA-Lib SMA call
is synchronous but tiny, so it's fine.)

### 2.2 The `http` module

Express is just a request handler. To attach **WebSockets** to the same port,
we need the raw Node `http.Server`:

```js
// server.js
const server = http.createServer(app);  // wrap Express in raw HTTP server
const wss = signalWs.init(server);      // WS hooks into the same server
server.listen(env.port, ...);           // one port (3003) serves both
```

WebSocket connections begin life as an HTTP "Upgrade" request, so the `ws`
library needs the actual server instance to intercept those.

### 2.3 `process`

The global `process` object = the running program.
- `process.env.PG_HOST` — environment variables (config from `.env`).
- `process.exit(1)` — quit with an error code (non-zero = failure). `env.js`
  does this when a required variable is missing: **fail fast at startup**, not
  randomly later.
- `process.pid` — process ID, logged at startup.
- `process.uptime()` — seconds since start, returned by `/health`.

### 2.4 Graceful shutdown (SIGTERM / SIGINT)

When you press Ctrl+C (SIGINT) or a process manager wants to stop the app
(SIGTERM), we should not die mid-request. `server.js`:

1. `server.close(...)` — stop accepting **new** connections, let in-flight
   requests finish, then exit cleanly with code 0.
2. A 10-second `setTimeout` safety net — if some connection refuses to close,
   force-exit with code 1 so the orchestrator (the tool that runs and
   auto-restarts our app — e.g. PM2, Docker, or Kubernetes) restarts us.

### 2.5 Last-resort error handlers

- `process.on('unhandledRejection', ...)` — a Promise failed and nobody
  caught it. We **log and keep running** (it's usually recoverable).
- `process.on('uncaughtException', ...)` — a synchronous error escaped
  everything. The process state is now unknown → **log and exit(1)** so it
  restarts fresh.

### 2.6 npm & package.json

- `dependencies` — needed in production: express, pg, ioredis, ws, joi,
  jsonwebtoken, winston, talib, cors, cookie-parser, dotenv.
- `devDependencies` — development only: nodemon (auto-restarts the server when
  a file changes; `npm run dev`).
- `npm start` → `node server.js` (production).

---

## Group 3 — Express.js

### 3.1 What Express is

A tiny framework over Node's http module: you declare **routes** (URL +
method → handler function) and **middleware** (functions that run on every
request before the handler).

### 3.2 Middleware and `next()`

A middleware is `(req, res, next) => { ... }`. It may:
- modify `req` (e.g. `auth.js` attaches `req.user`),
- end the response (`res.json(...)`),
- or pass control on: `next()` — or pass an error: `next(err)`.

The request flows through a **pipeline** in registration order. In `app.js`:

```
  incoming request
        │
        ▼
  ┌──────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────┐  ┌───────────┐
  │   cors   │─►│ express.json │─►│ urlencoded │─►│ cookieParser │─►│  request  │
  └──────────┘  └──────────────┘  └────────────┘  └──────────────┘  │  logger   │
                                                                     └─────┬─────┘
        ┌────────────────────────────────────────────────────────────────┘
        ▼
  ┌──────────┐ matched? ┌──────────────────────────────────────────────┐
  │ /health  │─────────►│ res.json(...)  ── DONE                        │
  └────┬─────┘          └──────────────────────────────────────────────┘
       │ no
       ▼
  ┌──────────────────────────────────────────┐
  │ /api router:  auth → validate → controller│──► res.json(...)  ── DONE
  └────┬─────────────────────────────────────┘
       │ nothing matched
       ▼
  ┌──────────────┐         every next(err) anywhere above jumps here ──┐
  │ 404 handler  │                                                     │
  └──────────────┘                                                     ▼
                                                          ┌─────────────────────┐
                                                          │    errorHandler     │
                                                          │ (4-arg, formats JSON)│
                                                          └─────────────────────┘
```

Each box is a middleware. It can **end** the response, **pass control**
forward with `next()`, or **jump to the error handler** with `next(err)`.

### 3.3 Routers

`express.Router()` is a mini-app for a group of routes. `routes/index.js`
mounts the feature routers:

```js
router.use('/strategies', strategiesRoutes);  // /api/strategies/...
router.use('/signals', signalsRoutes);        // /api/signals/...
router.use('/config', configRoutes);          // /api/config/...
```

And each feature router declares its endpoints with **the middleware chain
inline**:

```js
router.post('/', auth, validate(createStrategySchema), ctrl.create);
//             ↑ 1st: JWT  ↑ 2nd: Joi check            ↑ 3rd: real work
```

### 3.4 `req` and `res`

- `req.params.id` — from the URL path (`/strategies/:id`).
- `req.query.page` — from the query string (`?page=2`).
- `req.body` — the parsed JSON body (thanks to `express.json()`).
- `req.headers.authorization` — HTTP headers.
- `req.cookies.token` — cookies (thanks to `cookie-parser`).
- `res.status(201).json({...})` — set status code and send a JSON response.

### 3.5 Error-handling middleware

A middleware with **four** arguments `(err, req, res, next)` is special:
Express only calls it when some earlier code did `next(err)` or threw inside a
route. `middleware/errorHandler.js` is the **only** place that converts errors
to HTTP responses — controllers just `throw` / `next(err)` and stay clean.

It uses the `isOperational` flag (topic 1.7): operational errors show their
real message to the client; unexpected errors hide details behind a generic
"Internal server error" (never leak stack traces to users).

### 3.6 Body parsing & cookies

- `express.json({ limit: '1mb' })` — parses `Content-Type: application/json`
  bodies into `req.body`, rejecting bodies above 1 MB.
- `cookie-parser` — parses the `Cookie` header into `req.cookies`. This is how
  the login cookie issued by the main backend (:3002) is read here (:3003).

### 3.7 REST conventions used here

| Method | URL | Meaning | Code |
|--------|-----|---------|------|
| GET | `/api/strategies` | list (paginated, filterable) | 200 |
| GET | `/api/strategies/:id` | one strategy | 200 / 404 |
| POST | `/api/strategies` | create | **201** |
| PUT | `/api/strategies/:id` | update | 200 |
| PATCH | `/api/strategies/:id/toggle` | partial action (flip active) | 200 |
| DELETE | `/api/strategies/:id` | delete | 200 |
| POST | `/api/strategies/:id/backtest` | run an action | 200 |
| GET | `/api/signals` | list signals | 200 |
| GET | `/api/signals/strategy/:id` | signals of one strategy | 200 |
| GET | `/api/config/strategy-options` | indicator dropdown options | 200 |
| GET | `/api/indicators/:timeframe/:symbol` | latest indicator row | 200 |
| GET | `/health` | liveness check (no auth) | 200 |

Every response uses one **consistent envelope** (from `utils/response.js`):

```json
{ "success": true,  "data": ...,  "meta": { "page": 1, "limit": 10, "total": 42, "totalPages": 5 } }
{ "success": false, "error": { "message": "...", "code": "NOT_FOUND" } }
```

The frontend depends on this shape (it reads `response.data` and
`response.meta.total`).

### 3.8 Pagination

Page numbers from the client are turned into SQL:

```js
const offset = (page - 1) * limit;   // page 3, limit 10 → skip 20 rows
... ORDER BY created_at DESC LIMIT $n OFFSET $n+1
```

Plus a second `COUNT(*)` query for the total, so the frontend can render page
buttons. The Joi schemas cap `limit` at 100 and force `page ≥ 1` (page 0 would
make a negative OFFSET → Postgres error).

---

## Group 4 — Authentication & security

### 4.1 JWT — JSON Web Token

A JWT is a **signed** string with three parts: `header.payload.signature`.
(Each part is written in **base64** — a plain way of packing text/data into
safe characters for a URL or header. It is NOT encryption; it just rearranges
the characters, and anyone can un-pack it.)

- **payload** — JSON data, e.g. `{ "userId": 7, "permission": [...], "exp": 1730000000 }`.
- **signature** — a "tamper seal" made from header+payload plus a **secret
  key** (the technical name for the sealing method is **HMAC** — think of it
  as a special fingerprint that can only be produced if you know the secret).
  If anyone edits the payload, the fingerprint no longer matches and the token
  is rejected.

So the server doesn't need to store sessions: it just verifies the signature.

```
   A JWT is three base64 chunks joined by dots:

   eyJhbGciOi...  .  eyJ1c2VySWQiOjcs...  .  3xV9_kQ2...
   └─── HEADER ──┘    └──── PAYLOAD ─────┘    └─ SIGNATURE ┘
   {alg:"HS256"}      {userId:7,             HMAC(header+payload,
                       permission:[...],      SECRET)
                       exp:1730000000}        │
                                              │
   Anyone can READ header+payload.           │
   But only someone with the SECRET can ─────┘ produce a matching
   signature. Change one byte of payload → signature no longer matches
   → jwt.verify throws.
```

```js
const decoded = jwt.verify(token, env.jwt.secret); // throws if forged/expired
req.user = { id: decoded.userId, ... };
```

Important: the payload is only **encoded, not encrypted** — anyone can read
it; they just can't forge it.

### 4.2 Bearer header vs cookie

`auth.js` accepts the token from two places, in priority order:
1. `Authorization: Bearer <token>` header (typical for API clients),
2. the `token` **cookie** (what the browser actually uses here — the main
   backend sets it at login and the browser sends it automatically).

### 4.3 One secret, two backends

The main backend (:3002) **issues** tokens at login. The Strategy Engine
(:3003) **verifies** them with the **same `JWT_SECRET`**. That's how a user
logged into the admin panel is automatically authenticated to the Strategy
Engine — no second login. (This is why `.env` says "same secret as main
backend".)

### 4.4 CORS

Browsers block JavaScript on `http://localhost:3000` from calling
`http://localhost:3003` **unless** the API explicitly allows that origin —
that is CORS (Cross-Origin Resource Sharing).

`app.js` configures it strictly:

```js
cors({
  origin: env.corsOrigins,   // explicit allowlist from .env (CORS_ORIGINS)
  credentials: true,         // allow cookies to be included
  ...
})
```

**Security rule encoded in the comments:** never use `origin: true` together
with `credentials: true` — that would reflect ANY website's origin and let any
malicious page make cookie-authenticated calls to your API on behalf of the
victim's browser (CSRF-style data theft).

### 4.5 Why no `?token=` in HTTP URLs (but yes for WebSocket)

URLs get written into server logs, proxy logs, and browser history. A token in
the URL = credentials leaked into all those places. So `auth.js` deliberately
does NOT read tokens from the query string.

Exception: the **WebSocket handshake** cannot carry custom headers from the
browser, so `signalWs.js` accepts `ws://...?token=...` (and also cookies,
which usually do the job anyway).

### 4.6 SQL injection & parameterized queries

NEVER build SQL by gluing user input into the string:

```js
// ☠ DANGEROUS: pool.query(`SELECT * FROM users WHERE name = '${name}'`)
// name = "x'; DROP TABLE users; --"  → game over
```

Instead, **placeholders**:

```js
pool.query('SELECT * FROM strategies WHERE id = $1 AND user_id = $2', [id, userId]);
```

The driver sends SQL and values separately; values can never become SQL. Every
query in this codebase does this. (Table names like `${config.indicatorTable}`
are interpolated, but they come from the hard-coded `TIMEFRAME_MAP`, never
from user input — that's the rule that keeps it safe.)

### 4.7 Ownership checks

Almost every query includes `AND user_id = $X` with the id taken from the
**verified JWT** (`req.user.id`) — never from the request body. So user 7 can
never read, edit, delete, or backtest user 9's strategies, even by guessing
IDs. This is "row-level authorization done by hand".

---

## Group 5 — Validation with Joi

### 5.1 Why validate

Anything in `req.body` / `req.query` was typed by a client and can be
anything. Validation gives three protections: (1) clear 400 errors instead of
mysterious crashes, (2) defaults filled in, (3) junk fields removed.

### 5.2 Joi basics

A **schema** describes the allowed shape:

```js
const createStrategySchema = Joi.object({
    name: Joi.string().trim().min(1).max(100).required(),
    action: Joi.string().valid('BUY', 'SELL').default('BUY'),
    conditions: Joi.array().items(conditionSchema).min(1).required(),
    ...
}).or('symbols', 'symbol');  // at least one of these two must exist
```

`schema.validate(data)` returns `{ error, value }` — `value` is the cleaned
data (trimmed, defaulted, uppercased…).

### 5.3 `abortEarly: false` and `stripUnknown: true`

In `middleware/validate.js`:
- `abortEarly: false` — collect **all** problems, not just the first, so the
  user can fix the whole form in one go. They are mapped to
  `[{ field, message }]` and returned in `error.details`.
- `stripUnknown: true` — silently delete fields the schema doesn't know.
  **Gotcha documented in the code:** this is why `signal_type` had to be added
  to `getSignalsSchema` — otherwise the middleware would strip the filter
  before the controller saw it!
- The middleware then **replaces** `req.body`/`req.query` with the cleaned
  `value`, so controllers always work with safe data.

### 5.4 Conditional and alternative schemas

- `Joi.when()` — a field's rules depend on another field. In
  `conditionSchema`: if `valueType === 'IndicatorValue'`, then `value` must be
  one of the supported indicator names; otherwise it may be a string or number.
- `Joi.alternatives().try(a, b, c)` — value must match one of several shapes.
  Used for `trailing_stop_loss` (fixed | percent | stepped) and `target`
  (percent | price).

A great design detail: the schema only allows variables in
`SUPPORTED_VARIABLES` (built from `VARIABLE_TO_COLUMN`). Anything else would
save fine but **never fire** (the engine returns false for unknown variables)
— so the validator **fails loudly at save time** instead of letting the user
create a strategy that silently does nothing.

### 5.5 Body vs query validation

`validate(schema, source)` — the same middleware validates `req.body` (POST/PUT)
or `req.query` (GET lists) depending on the `source` argument:
`validate(listStrategiesSchema, 'query')`.

---

## Group 6 — PostgreSQL & `pg`

### 6.1 Relational basics

Data lives in **tables** with typed **columns**; each row has a **primary
key** (`id SERIAL PRIMARY KEY` = auto-incrementing integer). Tables reference
each other through id columns (`signals.strategy_id` → `strategies.id`).

### 6.2 Connection pooling

Opening a Postgres connection is slow (~tens of ms + handshake). A **pool**
keeps up to `max: 20` connections open and lends them out per query:

```js
new Pool({ ...env.marketDb, max: 20, idleTimeoutMillis: 30000, connectionTimeoutMillis: 10000 });
```

- `pool.query(...)` — borrow a connection, run, return it automatically.
- `pool.connect()` — borrow a connection **manually** (needed for
  transactions, because all statements must run on the SAME connection). You
  MUST `client.release()` in `finally`.

### 6.3 Parameterized queries — see topic 4.6.

### 6.4 Transactions

A transaction makes several statements **all-or-nothing**:

```js
await client.query('BEGIN');
// 1) INSERT INTO strategies ... RETURNING *
// 2) INSERT INTO strategy_symbols (one row per symbol)
await client.query('COMMIT');     // both become visible together
// on any error: ROLLBACK → as if nothing happened
```

Used in `strategy.model.create()` / `update()` because a strategy without its
symbol rows (or half its symbol rows) would corrupt the engine's view of the
world.

### 6.5 `RETURNING *`

Postgres can return the affected row from a write:

```sql
INSERT INTO signals (...) VALUES (...) RETURNING *;   -- get the new row + id
UPDATE ... RETURNING strategy_id;                      -- did anything match?
```

The second form doubles as a **success check**: `rows.length > 0` means the
UPDATE actually matched a row — the heart of the compare-and-swap trick (6.15).

### 6.6 UPSERT

"Insert, but if the row already exists, update it instead":

```sql
INSERT INTO feature_table_5min (symbol, "timestamp", sma_10, sma_20)
VALUES ($1, $2, $3, $4)
ON CONFLICT (symbol, "timestamp") DO UPDATE SET sma_10=$3, sma_20=$4;
```

Used by `indicator.model.upsert()` so re-processing the same candle never
creates duplicates. Also `ON CONFLICT ... DO NOTHING` when inserting symbol
rows (a duplicate symbol is simply ignored).

### 6.7 JSONB

Postgres can store JSON natively. Columns `conditions`, `exit_conditions`,
`position_sizing`, `brokerage_config`, `trailing_stop_loss`, `target`,
`condition_snapshot` are all `JSONB`.

Why JSONB instead of separate tables? The condition structure is flexible and
the engine always reads conditions **as a whole**, never queries "find all
strategies where condition 2 uses SMA(10)". JSONB = schema flexibility where
relational power is not needed.

The `pg` driver auto-converts JSONB to JS objects when reading; we
`JSON.stringify` when writing.

### 6.8 Joins

- `LEFT JOIN` — keep every left row even if there is no match on the right.
  `strategies LEFT JOIN strategy_symbols` — a strategy with zero symbol rows
  still appears (symbols become `'{}'`).
- `INNER JOIN` — only rows that match on both sides. `findActiveBySymbol` uses
  INNER JOIN because a strategy is only relevant if it actually has that symbol.

### 6.9 GROUP BY + aggregates

`findAll` builds one row per strategy with its symbols **aggregated into an
array**:

```sql
SELECT s.*,
       COALESCE(array_agg(ss.symbol ORDER BY ss.symbol)
                FILTER (WHERE ss.symbol IS NOT NULL), '{}') AS symbols,
       MAX(ss.last_triggered_at) AS last_activity_at
FROM strategies s
LEFT JOIN strategy_symbols ss ON ss.strategy_id = s.id
GROUP BY s.id
```

- `array_agg(x)` — collect values into a Postgres array (becomes a JS array).
- `json_agg(json_build_object(...))` — collect rows into a JSON array of
  objects (used in `findById` to return per-symbol trigger states).
- `FILTER (WHERE …)` — aggregate only matching rows. Without it, a LEFT JOIN
  with no symbols would produce `[null]` instead of `[]`. Also used for
  conditional counts: `COUNT(*) FILTER (WHERE is_active = true)`.

### 6.10 `COALESCE`, `GREATEST`, `LEAST`

- `COALESCE(a, b)` — first non-null. Used for "empty array instead of null".
- `GREATEST(a, b)` / `LEAST(a, b)` — max/min of the arguments. The TSL peak
  tracker uses them **inside the UPDATE** so the comparison happens atomically
  in the database:
  ```sql
  SET tsl_extreme_price = GREATEST(COALESCE(tsl_extreme_price, $3), $3)  -- long: keep the higher
  ```

### 6.11 Indexes

An index is a lookup structure that makes `WHERE`/`ORDER BY` fast.
Migrations create:
- `idx_signals_user (user_id, created_at DESC)` — perfect for the signal feed
  query (filter by user, newest first).
- **Partial index** `idx_strategies_active ... WHERE is_active = true` —
  indexes only active strategies, smaller and faster for the engine's hot path.

### 6.12 Column types seen here

- `SERIAL` — auto-incrementing integer id.
- `TIMESTAMPTZ` — timestamp **with timezone** (always store time with tz!).
- `DOUBLE PRECISION` — float (used for `trigger_price`).
- `NUMERIC(18,4)` — exact decimal (used for TSL prices — no float rounding errors).
- `BOOLEAN`, `TEXT`, `INTEGER`, `JSONB`.

### 6.13 LIMIT/OFFSET — see topic 3.8.

### 6.14 Quoted identifiers

Postgres lowercases unquoted names. The co-developer's tables were created
with capitals, so they MUST be quoted forever: `"OHLC_1MIN"`, `"Close"`,
`"Symbol"`, `"Time"`. Likewise `"timestamp"` is quoted because `timestamp` is
also a type name in SQL. Our own tables use lowercase names → no quotes needed.

### 6.15 Atomic compare-and-swap (CAS) — the most important trick in this codebase

Problem: a strategy must fire ENTRY **once**, not once per worker / per
duplicate event. Two evaluations could both see `trigger_state = 'IDLE'` and
both create signals.

Solution: don't check-then-update (two steps = race window). Do **one** atomic
UPDATE whose WHERE clause contains the expected old state:

```sql
UPDATE strategy_symbols
SET trigger_state = 'TRIGGERED', ...
WHERE strategy_id = $1 AND symbol = $2 AND trigger_state = 'IDLE'   -- ← the guard
RETURNING strategy_id;
```

Postgres row-locks during UPDATE, so only ONE caller can win the
IDLE→TRIGGERED flip; the loser's UPDATE matches 0 rows → `rows.length === 0` →
it silently skips signal creation. The same guard protects TRIGGERED→IDLE
(exits) — that's why exactly one EXIT signal can ever exist per position, "by
construction". Used by `transitionToTriggered`, `resetTriggerState`,
`updateTslStepCount` (guard: `tsl_step_count < $3` — steps only move forward).

```
   Two workers process near-duplicate events for the SAME (strategy, symbol):

   Worker A                          Worker B
   ────────                          ────────
   UPDATE ... WHERE state='IDLE' ──┐
                                   ├─► Postgres locks the row, lets ONE in
   UPDATE ... WHERE state='IDLE' ──┘
        │                                  │
        ▼                                  ▼
   row matched (1) ──► state now      row matched (0)  ← guard already failed
   = TRIGGERED                        because state is no longer 'IDLE'
        │                                  │
        ▼                                  ▼
   rows.length>0 → CREATE ENTRY       rows.length===0 → DO NOTHING (skip)

   Result: exactly ONE ENTRY signal, no matter how many workers race.
```

### 6.16 Two databases

- `victory_db` (the "app DB") — main backend's database. A pool for it exists
  (`appDb.js`) but **the models don't currently use it**.
- `victory_market_db` (the "market DB") — where EVERYTHING this service touches
  lives: our `strategies`, `strategy_symbols`, `signals` **and** the
  co-developer's `OHLC_*` / `feature_table_*` tables. Keeping strategies next
  to the candle data lets the engine join indicator rows with candle closes in
  one query (`getHistoricalRange`).

---

## Group 7 — Redis pub/sub

### 7.1 What Redis is

An in-memory data store. Here we use only one feature: **publish/subscribe** —
a built-in megaphone. A publisher sends a message to a named **channel**; every
subscriber of that channel receives it instantly. Messages are not stored: if
nobody is listening, the message is gone (fire-and-forget).

### 7.2 The channel that drives everything

The co-developer's pipeline publishes to **`ohlc:candle_close`** whenever a
candle finishes:

```json
{ "symbol": "RELIANCE", "timeframe": "5m", "ts": "2026-06-11T10:30:00Z" }
```

```
   PUBLISHER                    REDIS                    SUBSCRIBER
   (data pipeline)         channel:                 (candleCloseHandler)
        │              "ohlc:candle_close"                  │
        │  PUBLISH  ──────────►  📢  ──────────► message ──►│
        │  {symbol,tf,ts}                                   │
                                                            ▼
                          IndicatorService.compute(symbol, tf)
                                                            │
                                                            ▼
                          ConditionEngine.evaluate(symbol)
```

`jobs/candleCloseHandler.js` subscribes and, for each message:
`IndicatorService.compute(symbol, timeframe)` → `ConditionEngine.evaluate(symbol)`.
If nobody is subscribed when a message is published, it's simply gone
(fire-and-forget — Redis pub/sub does not store messages).

This makes the whole engine **event-driven**: zero work when the market is
quiet, instant work when a candle closes — no polling loop.

### 7.3 Why two Redis connections

A Redis connection in **subscribe mode** can ONLY subscribe — it cannot run
normal commands. So `redis.js` keeps two singletons: `getSubscriber()` and
`getPublisher()`. (The publisher exists for future use.)

### 7.4 Event-driven thinking

Compare:
- **Polling**: "every second, check the DB for new candles" — wasteful, late.
- **Events**: "tell me the moment a candle closes" — efficient, instant.

The same philosophy repeats at the next hop: the backend doesn't make the
browser poll for signals; it **pushes** them over WebSocket.

---

## Group 8 — WebSockets

### 8.1 HTTP vs WebSocket

- HTTP: client asks → server answers → connection conceptually done. The server
  can never speak first.
- WebSocket: one handshake, then a **permanently open two-way pipe**. The
  server can push a signal to the browser the millisecond it is created.

### 8.2 The upgrade handshake

A WS connection starts as a normal HTTP GET with `Upgrade: websocket` headers.
That's why `signalWs.init(server)` needs the raw `http.Server` — the `ws`
library intercepts upgrade requests on path `/ws` while Express keeps handling
normal requests on the same port.

### 8.3 Server-side lifecycle (`ws` library)

```js
const wss = new WebSocket.Server({ server, path: '/ws' });
wss.on('connection', (ws, req) => { ... });   // each new client
ws.on('close', ...); ws.on('error', ...);     // per-client events
ws.send(JSON.stringify(payload));             // push a message
```

### 8.4 Authenticating the socket

On connection, `extractToken(req)` looks for the JWT in (1) `?token=` query
param, (2) cookie, (3) Authorization header — then `jwt.verify`. **No valid
token → `ws.close(4401, 'Authentication required')`** immediately. The code
comment explains why this is strict: signals are per-user trading data; an
older version bucketed anonymous sockets and leaked every user's signals to
anyone who connected.

### 8.5 The connection registry

```js
const userConnections = new Map();   // userId → Set of sockets
```

```
   userConnections (Map)
   ┌──────────┬───────────────────────────────────┐
   │ userId 7 │ Set { ws(tab A), ws(tab B) }       │  ← user 7 has 2 tabs open
   │ userId 9 │ Set { ws(phone) }                  │
   └──────────┴───────────────────────────────────┘

   broadcastSignal(7, payload)
        │  looks up userId 7's Set
        ▼
   sends ONLY to tab A and tab B.   User 9 NEVER sees user 7's signals.
```

`broadcastSignal(userId, payload)` sends ONLY to that user's sockets (all
their open tabs), never to anyone else. When a socket closes it is removed;
when the set empties, the user's entry is deleted.

### 8.6 Heartbeat (ping/pong)

Networks silently drop idle connections. Every 30s the server pings each
client; the browser automatically pongs. A client that missed a pong by the
next sweep is `terminate()`d. This keeps the registry honest (no ghost
sockets) and keeps the connection alive through the network equipment in the
middle (routers, proxies) that likes to cut idle connections.

### 8.7 Close codes

WebSocket close frames carry a numeric code. 1000 = normal. Apps may use
4000–4999 for custom meanings — `4401` here mirrors HTTP 401 "unauthorized".

### 8.8 Client-side reconnect (frontend)

`useSignals.js` in the admin panel:
- `new WebSocket(url)` → `onopen` / `onmessage` / `onclose` / `onerror`.
- On message `type === 'NEW_SIGNAL'`: prepend `signal.data` to the list (the
  new signal appears at the top of the feed instantly).
- On close: schedule `connectWebSocket` again in 3 s (auto-reconnect).
- On intentional disconnect: **clear the timer and null out `onclose`** first,
  so the reconnect doesn't fire after unmount ("zombie connection" prevention).

---

## Group 9 — Logging with Winston

### 9.1 Levels

`logger.debug` < `info` < `warn` < `error`. The configured level filters out
everything below it: development shows `debug`+, production shows `info`+.
Usage pattern in the codebase:
- `debug` — chatty internals (each HTTP request, broadcast counts).
- `info` — business events (server started, ENTRY/EXIT fired, indicators computed).
- `warn` — recoverable oddities (insufficient candles, unknown variable, invalid WS token).
- `error` — failures (DB pool errors, evaluation errors).

### 9.2 Formats

`logger.js` builds the format pipeline: timestamps + error-stack capture, then
- **dev**: colorized, human-readable single lines (`2026-06-11 10:30:01 [info]: ENTRY ...`),
- **prod**: pure JSON lines (machines/log collectors parse these).

### 9.3 Metadata

Always pass context as a second object — it lands as structured fields:

```js
logger.error('candleCloseHandler failed', { error: err.message, stack: err.stack });
```

---

## Group 10 — Trading domain knowledge

### 10.1 OHLC candles & timeframes

A **candle** summarizes price movement during a fixed interval:
**O**pen (first price), **H**igh, **L**ow, **C**lose (last price), plus
**Volume**. A **timeframe** is the interval length. This system supports
`1m, 5m, 10m, 15m, 1h`, each in its own table (`"OHLC_5MIN"` etc.), written by
the co-developer's pipeline.

### 10.2 Candle close

A candle is only trustworthy once its interval **ends** — mid-candle values
keep changing. That's why all evaluation is driven by the `ohlc:candle_close`
event and uses closed-candle data only. (One consequence: live exits happen at
candle granularity, e.g. evaluated on the close of each driver-timeframe candle.)

### 10.3 Indicators & SMA

An **indicator** is a formula over price history.
**SMA(N)** = Simple Moving Average = average of the last N closes. SMA(10) on
5m = average of the last 10 five-minute closing prices. SMA smooths noise:
price above a rising SMA suggests an uptrend. The classic "golden cross"
pattern is exactly expressible here: *SMA(10) Crosses Above SMA(20)*.

Currently only `sma_10` and `sma_20` are live; the constants/comments show
RSI, EMA, MACD, Bollinger, ATR planned (they need DB columns from the
co-developer first).

### 10.4 TA-Lib

The industry-standard **T**echnical **A**nalysis library (a native C library
with Node bindings). One call computes a whole series:

```js
talib.execute({ name: 'SMA', inReal: close, optInTimePeriod: 10 }).result.outReal
```

`inReal` = array of closes, `outReal` = array of SMA values (shorter than the
input, because the first 9 candles can't have an SMA(10)). The engine takes
`last(...)` — only the newest value matters live. SMA(20) needs ≥ 20 candles,
hence the `candles.length < 20` guard.

### 10.5 Strategies & conditions

A strategy here = **symbols** + **action** (BUY/SELL) + **entry conditions** +
optional **exit conditions / stop loss / target** + extras (sizing, brokerage).
A condition is a small JSON object:

```json
{
  "category": "Technical Indicator",
  "variable": "SMA (10)",          ← left side
  "variableTimeframe": "5m",       ← which chart the left side reads
  "operator": "Crosses Above",
  "value": "SMA (20)",             ← right side
  "valueType": "IndicatorValue",   ← right side is another indicator (vs StaticValue = a typed number)
  "valueTimeframe": "5m",
  "logicalOperator": "AND"         ← how to combine with the NEXT condition
}
```

Conditions chain left-to-right: `((c1 OP c2) OP c3)` where each OP is the
`logicalOperator` of the **previous** condition (AND/OR).

### 10.6 Crosses Above / Crosses Below

"Is greater than" is true on **every** candle while above. "**Crosses** above"
is true only at the **moment of crossing**:

```
crossed above ⇔ previous: var ≤ compare  AND  now: var > compare
```

```
   price
     │              SMA(10) ____________╱‾‾‾‾  ← now ABOVE
     │                      ╱‾          ╱
     │   SMA(20) ─────────────────────────────
     │              ____╱‾‾  ← was BELOW
     │             ╱
     └──────────────┬──────────┬───────────────► time
                 candle N-1   candle N
                 (10≤20)      (10>20)
                              ▲
                              └── THE CROSS — fires here, ONCE only.
                                  On candle N+1 "10>20" is still true,
                                  but it has NOT just crossed → no fire.
```

That's why the engine fetches the **last 2 rows** (`getRecent(symbol, tf, 2)`)
— it needs previous + current values to detect the change.

### 10.7 Multi-timeframe

Each condition carries its own timeframes, so one strategy can require
*SMA(10)[5m] > SMA(20)[5m]* AND *SMA(10)[1h] > SMA(20)[1h]* (short-term entry
confirmed by long-term trend). The engine collects all timeframes used, loads
indicator data per timeframe into `indicatorMap`, and each condition reads its
own slice. The **driver timeframe** = the smallest one used (order
`1m→5m→10m→15m→1h`); it decides which candle's close becomes the trigger price
and sets the rhythm of the backtest loop.

### 10.8 Signals, BUY/SELL, long/short

A **signal** is a recorded recommendation: "BUY RELIANCE @ 2500".
- `signal_type`: **ENTRY** (open a position) or **EXIT** (close it).
- `action_type`: BUY or SELL — note the EXIT's action is always the **flip**
  of the strategy action (a BUY strategy exits by SELLing).
- A **long** position (action BUY) profits when price rises; a **short**
  (action SELL) profits when price falls. All stop/target math is mirrored for
  short positions (peak→trough, above→below).
- `status`: 'pending' → (future) 'executed'. `alert_or_trade`: Alert = just
  notify; Trade = (future) auto-execute.

### 10.9 The edge-triggered state machine (IDLE ↔ TRIGGERED)

Without state, "SMA10 > SMA20" would fire a signal **every candle** while
true. So each (strategy, symbol) pair keeps `trigger_state`:

```
                    entry conditions become true
                    (atomic IDLE→TRIGGERED + ENTRY signal)
            ┌───────────────────────────────────────────┐
            │                                            ▼
     ┌────────────┐                              ┌──────────────┐
     │    IDLE    │                              │  TRIGGERED   │
     │ (watching, │                              │ (in a trade, │
     │  no trade) │                              │  watching    │
     │            │                              │  for exit)   │
     └────────────┘                              └──────────────┘
            ▲                                            │
            └────────────────────────────────────────────┘
            TSL hit  OR  target hit  OR  exit conditions true
            (atomic TRIGGERED→IDLE + EXIT signal)
```

While TRIGGERED, entry conditions are ignored (you're already in the trade).
State lives **per symbol** in `strategy_symbols`, so one strategy watching 20
symbols runs 20 independent little state machines. Transitions are atomic
(topic 6.15). Toggling or rule-editing a strategy force-resets to IDLE.

### 10.10 Stop loss (three flavours)

A stop loss limits damage by exiting when price moves against you. Implemented
in `utils/tslMath.js` — **one shared function** used by live engine AND
backtest so they can never disagree.

1. **fixed** — stop frozen at entry ± value%. Entry 100, 2% → stop 98 forever.

   ```
   price ╱╲    ╱╲╱╲
        ╱  ╲  ╱      ← price wanders
   100 ●────────────  entry
    98 ┄┄┄┄┄┄┄┄┄┄┄┄┄  stop (NEVER moves) ── exit if price closes here
   ```

2. **percent (trailing)** — stop follows the best price seen ("extreme") at a
   fixed distance: stop = peak × (1 − v/100) for long. Price 100→110 lifts the
   stop 98→107.8; the stop never moves back down. Locks in profit.

   ```
   price                    peak 110
                          ╱‾‾‾╲          ← price falls back
   110              ____╱‾     ╲___ 108  ← closes below stop → EXIT
   100 ●──────────╱
        stop ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄
              98 → 99 → ... → 107.8 (trails 2% under the rising peak,
                                      ratchets UP only, never down)
   ```

3. **stepped** — three numbers: `initial_stop_pct` (starting stop),
   `hurdle_pct` (profit needed to switch trailing on — a ONE-TIME switch), and
   `step_pct` (stop climbs one step per step of further profit).

   ```
   entry 100, initial 2%, hurdle 5%, step 1%

   price:   100 ──► 104 ──► 105 ──► 106 ──► 107 ──► 108
                          (hurdle!)
   stop:     98     98      99      100     101     102
             └─ frozen ─┘    └──── one step per +1% profit ────┘
                            trailing switches ON at the hurdle (once),
                            then climbs one step per step_pct of profit.
                            Steps only move FORWARD (DB guard).
   ```

   Worked example (entry 100, initial 2, hurdle 5, step 1): below 105 → stop
   98; at 105 → first step → 99; 106 → 100; 107 → 101…

Supporting state in `strategy_symbols`: `tsl_entry_price` (seeded at entry),
`tsl_extreme_price` (peak/trough, ratcheted with GREATEST/LEAST),
`tsl_step_count`, `tsl_current_stop_price`.

### 10.11 Target (take profit)

The mirror of a stop loss: book profit at a level **above** entry (long) /
**below** entry (short). Two types (in `utils/targetMath.js`):
- `percent` — entry × (1 ± v/100); needs the stored entry price.
- `price` — an absolute level; entry not needed.

**Priority order each candle (both live & backtest): stop loss → target →
exit conditions.** Risk management first; one exit per candle wins.

```
   On every candle while TRIGGERED, check in this order — first hit wins:

   ┌─ 1. STOP LOSS ──┐  hit?  yes ─► EXIT (triggered_by: TSL)
   └────────┬────────┘
            │ no
   ┌─ 2. TARGET ─────┐  hit?  yes ─► EXIT (triggered_by: TARGET)
   └────────┬────────┘
            │ no
   ┌─ 3. EXIT CONDS ─┐  true? yes ─► EXIT (triggered_by: EXIT_CONDITIONS)
   └────────┬────────┘
            │ no
            ▼
        stay in the trade, wait for the next candle
```

### 10.12 Position sizing

How many shares to "trade" in the backtest:
- `fixed_quantity` — always N shares.
- `fixed_capital` — `floor(capital / entryPrice)` shares. If even 1 share
  doesn't fit the capital, the trade is **skipped** and recorded in
  `skipped_trades` with reason `insufficient_capital`.
Without sizing, the backtest still finds entries/exits but reports no P&L
("Not configured" — no fake numbers).

### 10.13 Brokerage & net P&L

`brokerage_config` = per-side percentages (buy_pct / sell_pct, 0–5%).
- Buying: total cost = gross + brokerage (you pay more).
- Selling: total received = gross − brokerage (you receive less).
- **Gross P&L** = (exit − entry) × qty × direction. **Net P&L** = sell-side
  total − buy-side total (i.e. after costs). The backtest reports both.

### 10.14 Backtesting & statistics

Backtest = replay history through the SAME condition logic and exit rules and
record the trades that would have happened. Key stats (in `computePnLSummary`):
- **win rate** = winners / total trades × 100,
- **profit factor** = total profit / |total loss| (>1 = profitable system;
  `null`→∞ when there are no losses),
- avg profit / avg loss, largest win / largest loss, total brokerage paid.
Range is capped at 30 days per run. Important honesty detail: the backtest
exits **at the stop/target level** (the theoretical price), while live exits
record the actual candle close — tiny differences are expected.

### 10.15 Market time & IST

Indian markets run 09:15–15:30 **IST**. A "Time" condition ("Market Time Is
After 09:30") lets strategies avoid the volatile open or stop entries before
close. Implementation detail: timestamps are UTC; IST = UTC + 5:30, so the
engine converts via "minutes since midnight": `utcH*60 + utcM + 330 (mod
1440)` and compares pure minute numbers — no timezone library needed.

---

## Group 11 — Architecture & design patterns

### 11.1 Layered architecture

```
   ┌──────────────────────────────────────────────────────────────┐
   │ ROUTE        routes/        "URL + which middlewares run"      │
   │              e.g. POST /strategies → auth → validate → create  │
   ├──────────────────────────────────────────────────────────────┤
   │ CONTROLLER   controllers/   "parse req, call below, send res"  │
   │              reads req.body, req.user; calls models/services   │
   ├──────────────────────────────────────────────────────────────┤
   │ SERVICE      services/      "the business brain"  (NO req/res) │
   │              ConditionEngine, BacktestService, SignalService   │
   ├──────────────────────────────────────────────────────────────┤
   │ MODEL        models/        "ALL the SQL"  (nothing else)      │
   │              strategy.model, signal.model, indicator.model     │
   └────────────────────────────┬─────────────────────────────────┘
                                ▼
                          PostgreSQL
```

Each layer only talks to the one directly below it. Why: every layer is
testable and replaceable alone; you always know where to look (SQL bug →
models; wrong status code → controllers; wrong trading logic → services).

### 11.2 Microservice split

The Strategy Engine is its own process/repo/port (3003), separate from the
main backend (3002). Benefits: independent deploys/restarts, a market-data
crash can't take down login, and the heavy candle-processing load is isolated.
They cooperate via the shared JWT secret (4.3) — no inter-service calls needed.

### 11.3 Single source of truth

`config/constants.js` defines timeframes/indicators/operators **once**;
`VARIABLE_TO_COLUMN` and `INDICATOR_COLUMNS` are **derived** from
`INDICATOR_CONFIG`, never written by hand. Adding an indicator = 1 line there
+ 1 TA-Lib call. The validator, engine, upsert, and the `/config/strategy-options`
endpoint (which feeds the frontend dropdowns) all read from this file — so
they can't drift apart.

### 11.4 Dependency injection

`SignalService` must broadcast over WS, but importing `ws/signalWs.js` would
couple the service to the transport. Instead the service exposes
`setBroadcast(fn)` and `server.js` wires it at boot:

```js
SignalService.setBroadcast(signalWs.broadcastSignal);
```

The service just calls "the function it was given". Tests could inject a fake;
no WS server needed.

```
   At boot (server.js):
       SignalService.setBroadcast( signalWs.broadcastSignal )
                          │                    │
                          ▼                    │
       SignalService remembers it as  broadcastFn
                          │                    │
   Later, when a signal fires:                 │
       SignalService.create(...)               │
            │ INSERT row                        │
            └─► broadcastFn(userId, signal) ────┘──► WS push to browser

   SignalService never imports ws/signalWs.js — it just calls the
   function it was handed. (Swap in a fake for tests.)
```

### 11.5 Singletons — see 1.8.

### 11.6 Race conditions & CAS — see 6.15 and 10.9.

### 11.7 Defensive parsing

`parseTrailingStopLoss`, `parseTarget`, `parseSizing`, `parseBrokerage` all
re-validate values **read from the DB** with the same ranges the Joi schema
enforces on the way in. Why double-check? A row could have been written by an
older version, a manual SQL edit, or a bug. Example consequence prevented: a
stepped TSL with `step_pct = 0` would divide by zero → `NaN` → silently
disabled stop. Rule of thumb baked into this codebase: **validate at every
trust boundary** (client→API via Joi, DB→engine via parsers, API→UI via the
frontend's own mirror parsers).

### 11.8 Operational vs programmer errors

- Operational (`isOperational: true`, the `AppError` family): expected
  situations — invalid input, not found, no auth. Logged as `warn`, real
  message sent to the client.
- Programmer errors (everything else): a bug. Logged as `error` **with stack
  trace**, client sees only "Internal server error".

### 11.9 Graceful shutdown — see 2.4.

### 11.10 "Policy B" — synthetic EXIT on rule edits

Scenario: a strategy is TRIGGERED (position open) and the user **edits its
rules** (action/conditions/exit_conditions/TSL/target). The old exit logic is
gone — the position would be orphaned (maybe never exit!). Policy B (in
`strategies.controller.update`):

1. Before updating, fetch symbols currently TRIGGERED.
2. Apply the update.
3. For each previously-TRIGGERED symbol, atomically reset TRIGGERED→IDLE
   (CAS — if the live engine *just* exited it, we do nothing → still exactly
   one EXIT), and emit a **synthetic EXIT signal** at the latest 1m close with
   `condition_snapshot.triggered_by = 'STRATEGY_EDITED'`.

Toggling (pause/activate) instead uses the simpler `forceResetTriggerState`
(all symbols → IDLE, no synthetic exits).

---

## Group 12 — Frontend topics (short versions; full detail in 04_FRONTEND_CONNECTION.md)

### 12.1–12.2 React & hooks
- A component is a function returning JSX (HTML-like syntax). Props = inputs.
- `useState` — component memory that re-renders on change.
- `useEffect(fn, deps)` — run side effects (fetching, subscriptions) after
  render, when `deps` change; return value = cleanup function.
- `useCallback` / `useMemo` — keep functions/values stable between renders so
  effects don't loop and children don't re-render needlessly.
- `useRef` — a mutable box that does **not** trigger re-render (the WS object,
  reconnect timers, "pending nav symbol" all live in refs).

### 12.3 Custom hooks

The app's data pattern: page → `useStrategies()` / `useSignals()` →
`strategyEngineService` → axios instance. `useApi()` centralizes
loading/error state for any request.

### 12.4 axios

`strategyEngineApi.js` creates a dedicated instance: `baseURL =
REACT_APP_STRATEGY_ENGINE_URL + 'api'`, `withCredentials: true` (send the auth
cookie cross-origin), and a **response interceptor that unwraps to
`response.data`** — so services return the backend envelope directly
(`{ success, data, meta }`), not the axios wrapper. The error interceptor
normalizes every failure to `{ message, status, data }`.

### 12.5 React Router

Routes in `App.js`: `/strategies`, `/strategies/:id`,
`/strategies/:id/backtest`, `/signals`. `useParams()` reads `:id`;
`useNavigate()` changes pages in code; `navigate(url, { state })` passes data
(StrategyDetail → BacktestPage sends symbol/dates/autoRun via `location.state`).

### 12.6 Redux + ProtectedRoute

Login presence = `localStorage["userSession"]`; permissions are fetched once
from the main backend (`/me/permissions`) into a Redux slice. Every authed
route is wrapped in `<ProtectedRoute permission="...">` which redirects to `/`
(no session) or `/access-denied` (missing permission).

### 12.7 Browser WebSocket — see 8.8.

### 12.8 CSS Modules

`Component.module.css` imported as `styles` → class names are scoped per
component (no global clashes).

### 12.9 Controlled forms & the wizard

`StrategyForm` keeps the whole strategy draft in one `formData` state object,
edits it immutably (`setFormData(prev => ({ ...prev, ... }))`), validates per
step (3-step wizard: Basic Info → Conditions → Review), and coerces
string inputs to numbers only at submit time.

### 12.10 Mirrored limits

The frontend repeats backend bounds as constants (TSL 0.1–50, target percent
0.1–500, brokerage 0–5, symbol cap 20…) for instant feedback — but the backend
Joi schemas remain the real enforcement. The comments explicitly say: keep
them in sync.

---

**Next:** read [02_ARCHITECTURE.md](02_ARCHITECTURE.md) to see how all these
pieces snap together.
