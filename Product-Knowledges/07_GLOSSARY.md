# 07 — Plain-Language Glossary (every tough word, explained simply)

Whenever you hit a word in the other files that you don't know, look it up
here. Each entry is written in **everyday language**, often with a simple
real-life comparison. Terms are grouped by topic, and there's an A–Z quick
index at the bottom.

> Tip: most of these are also explained where they first appear in
> [01_TOPICS_TO_LEARN.md](01_TOPICS_TO_LEARN.md). This file is the "I just
> need a one-line reminder" version.

---

## General programming words

**API** — a set of web addresses (URLs) a program offers so other programs
can ask it to do things. Like a restaurant menu: you order from the menu,
the kitchen does the work.

**Endpoint** — one specific item on that menu. E.g. `GET /api/strategies` is
the "give me the list of strategies" endpoint.

**Request / Response** — a request is the message your browser sends ("please
give me strategy 5"). The response is what the server sends back.

**Payload** — the actual data inside a message. When you save a strategy, the
payload is the strategy's name, symbols, conditions, etc.

**Parse** — to read a piece of text and turn it into something the program can
work with. E.g. "parse JSON" = turn a text string into a usable object.

**Coerce** — to force a value into a particular type. E.g. coerce the text
`"100"` into the number `100`.

**String** — text. **Integer** — a whole number. **Float / floating-point** —
a number with decimals (like `2500.50`).

**Boolean** — a true/false value.

**Null** — "nothing here / no value". Different from `0` or empty text.

**Array** — an ordered list of things: `["RELIANCE", "TCS"]`.

**Object** — a bag of named values: `{ name: "...", price: 2500 }`.

**Default** — the value used when you don't provide one. E.g. action
defaults to "BUY".

**Defensive (defensive coding)** — writing code that double-checks things and
copes calmly with bad/unexpected input instead of crashing. "Trust nothing."

**Edge case** — an unusual situation at the "edge" of normal use (e.g. a price
of exactly 0, an empty list, the very last candle of the day).

**Idempotent** — doing the same thing twice has the same effect as doing it
once. (E.g. the upsert: re-processing the same candle does not create a
duplicate.)

**Big-O / O(N+M)** — shorthand for "how much slower does this get as the data
grows?" `O(N+M)` means the work grows gently (touch each item once). `O(N×M)`
means it grows much faster (re-scan everything for every item). Smaller is
better.

**Algorithm** — a step-by-step recipe for doing something.

---

## How the code is organised

**Module** — one code file. It can "export" things for other files to use.

**Import / require** — pulling another file's exported things into the current
file so you can use them.

**Singleton** — a thing that is created **only once** and shared everywhere
(like one shared printer for a whole office). The database pool is a
singleton.

**Closure** — a function that "remembers" a variable from where it was
created. Used to build singletons.

**Class** — a blueprint for making objects of a certain kind. `AppError` is a
class; each thrown error is an object made from it.

**Inherit / extends** — "is a special kind of". `NotFoundError` extends
`AppError`, meaning it's a more specific type of `AppError`.

**Layered architecture** — splitting code into stacked layers (route →
controller → service → model), each with one job, each only talking to the
layer below.

**Dependency injection (DI)** — instead of a piece of code grabbing what it
needs itself, you **hand it** what it needs from outside. Makes testing
easier. (We hand the broadcast function to `SignalService`.)

**Microservice** — a separate, self-contained mini-application. The Strategy
Engine is a microservice, separate from the main backend.

**Scaffolding** — half-built, placeholder code left in place for a feature
that isn't finished yet (e.g. the commented-out RSI/MACD indicators).

**Relic / legacy** — old code or columns kept around from a previous design,
no longer the main way things work (e.g. the single `trigger_state` column on
the `strategies` table).

---

## Web / server words

**Backend** — the server-side program (runs on a server, not in the browser).

**Frontend** — the part that runs in the browser (the React admin panel).

**Express** — the framework that makes it easy to build a backend web server
in Node.js.

**Middleware** — a function that runs on a request **before** the final
handler, in a chain. Each one can check, change, stop, or pass on the request.
(Like airport checkpoints: security → passport → boarding.)

**Route** — the rule that says "this URL + this method runs this code".

**Controller** — the code that handles a request: reads it, calls the
business logic, sends a reply.

**Service** — the "brain" code that does the real work (no web stuff inside).

**Model** — the code that holds all the database queries.

**Handler** — the function that actually deals with a request.

**REST** — a common style for designing web APIs using standard verbs:
GET (read), POST (create), PUT (update), PATCH (small update), DELETE.

**Status code** — a number the server sends to say how it went: 200 = OK,
201 = created, 400 = bad input, 401 = not logged in, 404 = not found,
500 = server error.

**Envelope** — the consistent wrapper shape every reply uses here:
`{ success: true, data: ... }` or `{ success: false, error: ... }`.

**Pagination** — sending a big list in small pages (page 1, page 2, …) instead
of all at once. `limit` = page size; `offset` = how many to skip.

**CORS** — the browser's rule about which websites are allowed to call which
servers. We keep a list ("allowlist") of permitted sites.

**Allowlist** — a list of things that are explicitly permitted (the opposite
of a blocklist).

**Cross-origin** — when the page (e.g. `localhost:3000`) talks to a server on
a different address/port (e.g. `localhost:3003`). The browser is extra careful
about this.

**Interceptor** — a function that runs automatically on every request or every
response (a checkpoint the data passes through). axios uses these to attach
settings and unwrap replies.

**Liveness check** — a simple "are you alive?" endpoint (`/health`) that
monitoring tools ping to confirm the app is running.

**Orchestrator** — the tool that runs and auto-restarts the app if it crashes
(e.g. PM2, Docker, Kubernetes / "k8s").

**Load balancer** — a traffic director that spreads requests across several
copies of an app.

**Graceful shutdown** — stopping the app politely: stop taking new work, let
current work finish, then exit (instead of dying instantly mid-request).

---

## Security & login words

**Authentication (auth)** — proving who you are (logging in).

**Authorization** — checking what you're allowed to do once you're logged in.

**JWT (JSON Web Token)** — a signed login ticket the server gives you after
login; you show it on every later request. The server can check it's genuine
without storing sessions.

**Token** — the login ticket string itself.

**Base64** — a way of packing data into safe plain characters (for URLs,
headers, tokens). It is **not** secret/encryption — anyone can un-pack it.

**HMAC** — the "tamper seal" maths used to sign a JWT: a special fingerprint
that can only be produced by someone who knows the secret key.

**Encryption vs encoding** — encoding (like base64) just rearranges data and
is reversible by anyone. Encryption hides data so only someone with the key
can read it. A JWT payload is **encoded**, not encrypted.

**Secret / secret key** — a private password-like string the server uses to
sign and verify tokens. Both backends share the same one.

**Cookie** — a small piece of data the browser stores and automatically sends
back to the server on every request (here it carries the JWT).

**Bearer token** — a token sent in the `Authorization: Bearer <token>` header.

**CSRF (Cross-Site Request Forgery)** — an attack where a malicious website
secretly makes requests to your API using your logged-in cookie. Strict CORS
+ an origin allowlist prevents it.

**SQL injection** — an attack where a user types SQL code into an input box
hoping the server runs it. Prevented by **parameterized queries** (never glue
user text into the query).

**Parameterized query** — a query with placeholders (`$1, $2`) where the
values are sent separately, so user input can never become commands.

**Ownership check** — adding `WHERE user_id = <you>` to every query so you can
only ever see/touch your own data.

---

## Database words

**PostgreSQL / Postgres** — the database system used here.

**Table** — a grid of data (rows and columns), like a spreadsheet.

**Row / record** — one entry in a table (one strategy, one signal).

**Column / field** — one property in a table (name, price, status).

**Primary key (PK)** — the unique id of a row (`id`).

**Foreign key (FK)** — a column that points to a row in another table
(`signals.strategy_id` points to a `strategies.id`).

**Junction table** — a table that links two things and stores info about the
link. `strategy_symbols` links a strategy to each of its symbols and stores
that symbol's live state.

**Query** — a request to the database (read or write).

**Connection pool** — a small set of ready-to-use database connections, shared
and reused (because opening a new one each time is slow). "Pool" = a shared
puddle of connections.

**Transaction** — a group of database changes that must **all** succeed or
**all** be undone, never half. `BEGIN` … `COMMIT` (keep) or `ROLLBACK` (undo).

**Atomic** — happens as one indivisible step; no one can see a half-done
state. (Like flipping a switch — it's either off or on, never in-between.)

**Compare-and-swap (CAS)** — a safe one-step update: "change this **only if**
it's still in the state I expect." Used so two workers can't both fire the
same signal. (See the race diagram in 01, topic 6.15.)

**Race condition** — a bug where two things happen at the same time and step on
each other. CAS + transactions prevent it.

**Row lock** — the database briefly "reserves" a row during an update so only
one writer touches it at a time.

**Upsert** — "update if it exists, otherwise insert". One command that does
both (`INSERT … ON CONFLICT … DO UPDATE`).

**JSONB** — a column type that stores a whole JSON object/array inside one
cell (used for `conditions`, `target`, etc.).

**Join** — combining rows from two tables that match on a column.
**LEFT JOIN** keeps all left-side rows even with no match; **INNER JOIN** keeps
only matched rows.

**Aggregate** — squashing many rows into one value: `COUNT` (how many),
`MAX` (biggest), `array_agg` (collect into a list), `json_agg` (collect into a
list of objects).

**Index** — a lookup shortcut that makes searches/sorts fast (like a book's
index). A **partial index** only covers some rows (e.g. only active
strategies).

**SERIAL** — an auto-counting id column (1, 2, 3, …).

**TIMESTAMPTZ** — a date-and-time value that remembers its timezone.

**COALESCE / GREATEST / LEAST** — "first non-empty value" / "the biggest" /
"the smallest" of the given values.

**Schema** — the shape/structure of data (which tables/columns exist, or in
Joi's case, what a valid request looks like).

**Migration** — a script that sets up or changes the database structure.

---

## Real-time words (Redis & WebSockets)

**Redis** — a fast in-memory data store. Here it's used only as a "megaphone"
(pub/sub).

**Publish / Subscribe (pub/sub)** — one program shouts a message on a named
channel; everyone listening on that channel hears it. If no one is listening,
the message is gone (not stored).

**Channel** — the named "radio frequency" for pub/sub (`ohlc:candle_close`).

**Event-driven** — the program reacts to events as they happen ("tell me when
a candle closes") instead of constantly asking ("any new candles yet? yet?").

**Polling** — repeatedly asking for updates on a timer. The opposite of
event-driven. (We avoid it.)

**WebSocket (WS)** — a connection that stays open both ways, so the server can
push data to the browser the instant it happens (vs normal HTTP, where the
browser must ask first).

**Handshake** — the initial setup step that opens a WebSocket.

**Heartbeat (ping/pong)** — the server periodically "pings" each connection;
a live one auto-"pongs". No pong = dead connection, drop it. Keeps the list
clean and stops the network from cutting an idle line.

**Broadcast** — sending a message to many connections at once (here, to all of
one user's open tabs).

**Fan-out** — how many places a message gets sent to.

**Reconnect / backoff** — automatically re-opening a dropped connection.
"Backoff" = wait a bit longer before each retry instead of hammering nonstop.

**Latency** — delay; how long data takes to arrive.

**Zombie connection** — a leftover connection that should have been closed but
keeps lingering. The frontend carefully avoids these.

---

## React / frontend words

**React** — the library used to build the browser UI out of reusable pieces.

**Component** — one reusable UI piece (a button, a card, a whole page).

**Props** — the inputs you pass into a component.

**State** — a component's memory; changing it re-draws the screen.

**Hook** — a special React function (name starts with `use`) that adds a
capability to a component: `useState` (memory), `useEffect` (run code after
render), `useRef` (a memory box that doesn't redraw), `useCallback`/`useMemo`
(remember a function/value so it isn't rebuilt every time).

**Custom hook** — your own reusable hook built from the built-in ones
(`useStrategies`, `useSignals`).

**Render** — React drawing the component on screen.

**Mount / Unmount** — when a component appears on screen / is removed from it.

**axios** — the tool the frontend uses to make HTTP requests to the backend.

**baseURL** — the common start of every address an axios instance calls.

**withCredentials** — an axios setting that says "include the login cookie on
cross-origin requests".

**Redux** — a central store for app-wide state (here, the user's permissions).

**Router / route** — the frontend's map of which URL shows which page.

**CSS Module** — a styling file scoped to one component so its class names
can't clash with others.

**Controlled form** — a form whose inputs are driven by React state (React is
the single source of truth for what's typed).

**Toast** — a small pop-up notification ("Strategy created!").

---

## Trading words

**OHLC candle** — a summary of price over a time period: **O**pen, **H**igh,
**L**ow, **C**lose, plus **V**olume (how much traded).

**Timeframe** — the length of each candle: 1m, 5m, 10m, 15m, 1h.

**Candle close** — the moment a candle's time period ends and its values are
final. The engine only acts on closed candles.

**Indicator** — a number calculated from price history to spot patterns.

**SMA (Simple Moving Average)** — the average of the last N closing prices.
SMA(10) = average of the last 10 closes. Smooths out the noise.

**TA-Lib** — the well-known library that calculates indicators for us.

**Strategy** — your saved trading rule: which symbols, BUY or SELL, and the
conditions that trigger it.

**Symbol** — a stock's short code (e.g. RELIANCE, TCS).

**Condition** — one rule, like "SMA(10) is greater than SMA(20)".

**Operator** — the comparison in a condition: greater than, less than,
crosses above, etc.

**Crosses Above / Below** — true only at the **moment** one line moves past
another (not every candle while it stays past).

**Signal** — a recorded recommendation the engine produced (BUY/SELL,
ENTRY/EXIT).

**ENTRY / EXIT** — opening a position / closing it.

**Long / Short** — a long (BUY) trade profits when price rises; a short (SELL)
trade profits when price falls.

**Position** — an open trade you currently hold.

**Edge-triggered** — fire once on the **change** into "true", not repeatedly
while it stays true. (Like a doorbell: it rings when pressed, not the whole
time it's held.)

**State machine** — something that is always in one of a few defined states
and moves between them on events. Here: IDLE ↔ TRIGGERED.

**Stop loss** — an automatic exit that limits your loss if price moves against
you.

**Trailing stop** — a stop that follows the price in your favour (moves up as
price rises for a long trade) but never backwards, locking in profit.

**Ratchet** — moves one direction only, never back (like a spanner that turns
one way). The trailing stop and the peak price "ratchet".

**Extreme price (peak/trough)** — the best price reached since entry — the
highest for a long trade, the lowest for a short.

**Hurdle** — in a stepped stop: the amount of profit needed before the stop
starts trailing (a one-time switch).

**Target / take-profit** — an automatic exit that books profit at a chosen
level (the mirror image of a stop loss).

**Position sizing** — how big a trade is: a fixed number of shares, or a fixed
amount of money.

**Brokerage** — the fee charged on each buy/sell. **Gross P&L** = profit
before fees; **Net P&L** = profit after fees. ("P&L" = profit and loss.)

**Backtest** — replaying a strategy over past data to see how it would have
performed.

**Win rate** — the percentage of trades that made money.

**Profit factor** — total profit divided by total loss. Above 1 = profitable.

**Driver timeframe** — the smallest timeframe a strategy uses; it sets the
pace of evaluation and provides the trigger price.

**IST** — Indian Standard Time (UTC + 5 hours 30 minutes). Used for "Market
Time" conditions.

**LTP** — Last Traded Price; the most recent price of a stock.

**Snapshot (condition_snapshot)** — the saved record of *why* a signal fired
(which indicator values were true at that moment).

**Policy B** — this project's rule: if you edit a live strategy's logic while a
position is open, the engine emits a "synthetic" (system-made) EXIT signal so
the open position is never left stranded.

---

## A–Z quick index

Aggregate · Algorithm · Allowlist · API · Array · Atomic · Authentication ·
Authorization · axios · Backend · Backoff · Backtest · Base64 · baseURL ·
Bearer token · Big-O · Boolean · Broadcast · Brokerage · Candle · CAS · Channel ·
Class · Closure · Coerce · COALESCE · Column · Component · Condition ·
Connection pool · Controller · Cookie · CORS · Controlled form · Cross-origin ·
Crosses Above/Below · CSRF · CSS Module · Custom hook · Default · Defensive ·
Dependency injection · Driver timeframe · Edge case · Edge-triggered ·
Encryption · Endpoint · Envelope · ENTRY/EXIT · Event-driven · Express ·
Extreme price · Fan-out · Float · Foreign key · Frontend · Graceful shutdown ·
Gross/Net P&L · Handler · Handshake · Heartbeat · HMAC · Hook · Hurdle ·
Idempotent · Index · Indicator · Inherit · Integer · Interceptor · IST · JOIN ·
JSONB · Junction table · JWT · Latency · Layered architecture · LEFT/INNER JOIN ·
Liveness · Load balancer · Long/Short · LTP · Microservice · Middleware ·
Migration · Model · Module · Mount/Unmount · Null · Object · OHLC · Operator ·
Orchestrator · Ownership check · Pagination · Parameterized query · Parse ·
Partial index · Payload · Policy B · Polling · Position · Position sizing ·
PostgreSQL · Primary key · Profit factor · Props · Publish/Subscribe · Query ·
Race condition · Ratchet · React · Reconnect · Redis · Redux · Relic · Render ·
Request/Response · REST · Route · Router · Row · Row lock · Scaffolding ·
Schema · Secret · SERIAL · Service · Signal · Singleton · SMA · Snapshot ·
SQL injection · State · State machine · Status code · Stop loss · Strategy ·
String · Symbol · TA-Lib · Table · Target · Timeframe · TIMESTAMPTZ · Toast ·
Token · Trailing stop · Transaction · Upsert · WebSocket · Win rate ·
withCredentials · Zombie connection
