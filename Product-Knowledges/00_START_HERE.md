# 📚 Strategy Engine — Complete Learning Documentation

Welcome! This documentation explains **every part** of the Trade Backend (the
"Strategy Engine") and how it connects to the Admin Panel frontend and the
databases. It is written in **easy language** so you can build a 100% grip on
how everything works — from a button click in the browser, through the backend
code, down to the database, and back again.

---

## How to read this documentation

Read the files **in order**. Each file builds on the previous one.

| # | File | What it teaches you |
|---|------|---------------------|
| 1 | [01_TOPICS_TO_LEARN.md](01_TOPICS_TO_LEARN.md) | **Every topic / concept / technology** used in this codebase, explained one by one in detail. Read this first — once you know these topics, the code will feel easy. |
| 2 | [02_ARCHITECTURE.md](02_ARCHITECTURE.md) | The **big picture**: all the moving parts (frontend, two backends, two databases, Redis, WebSockets) and how data flows between them. |
| 3 | [03_BACKEND_CODE_WALKTHROUGH.md](03_BACKEND_CODE_WALKTHROUGH.md) | A **file-by-file, flow-by-flow walkthrough of the entire backend code**. Every file in `src/` is explained. |
| 4 | [04_FRONTEND_CONNECTION.md](04_FRONTEND_CONNECTION.md) | How the **Admin Panel (React)** talks to this backend — pages, hooks, services, axios, WebSocket. |
| 5 | [05_DATABASE.md](05_DATABASE.md) | Every **table and column**, who writes it, who reads it, and why. |
| 6 | [06_LIFECYCLES.md](06_LIFECYCLES.md) | **End-to-end stories**: "user creates a strategy", "a candle closes and a signal fires", "trailing stop loss exits a trade", "a backtest runs" — step by step through the real code. |
| 7 | [07_GLOSSARY.md](07_GLOSSARY.md) | **Plain-language dictionary of every tough/technical word** used anywhere in these docs. Whenever a word is unclear, look it up here — each one is explained in everyday language, often with a simple real-life comparison. |

> **Everything here is written in easy, beginner-friendly language.** Wherever
> a technical word appears, it is explained right there in plain English — and
> if you ever want a quick one-line reminder of any term, the
> [glossary (07)](07_GLOSSARY.md) defines all of them in simple words.

---

## What is this project, in one paragraph?

The **Strategy Engine** is a Node.js + Express service (port **3003**) that lets
an admin user define **trading strategies** ("BUY RELIANCE when SMA(10) crosses
above SMA(20) on the 5-minute chart"). A separate data pipeline (built by a
co-developer) writes live market candles into a PostgreSQL database and
announces every closed candle on **Redis**. The Strategy Engine listens for
those announcements, computes indicators (SMA) with **TA-Lib**, evaluates every
active strategy's conditions, and when a strategy's conditions become true it
creates a **signal** (ENTRY or EXIT) in the database and **pushes it instantly
to the browser over WebSocket**. It can also **backtest** a strategy against
historical data, including position sizing, brokerage costs, trailing stop
loss, and profit targets. The **Admin Panel** (React app, port 3000) is the UI
for creating strategies, watching the live signal feed, and running backtests.

---

## The three codebases involved

| Codebase | Where | Port | Role |
|----------|-------|------|------|
| **Strategy Engine** (this repo) | `D:\Trade-Backend\Backend` | 3003 | Strategy CRUD, signal generation, backtesting, signal WebSocket |
| **Main backend** (separate repo, not on this machine) | — | 3002 | Login/auth (issues the JWT cookie), users, permissions, market price WebSocket (`/ws/prices`), everything else in the admin panel |
| **Admin Panel** (React frontend) | `D:\Admin-panel\Victory_Admin_Pannel` | 3000 | The browser UI for everything, including Strategies / Signals / Backtest pages |

Plus one **invisible partner**: the co-developer's **data pipeline**, which owns
the candle tables (`OHLC_*`) and indicator tables (`feature_table_*`) and
publishes the `ohlc:candle_close` Redis event that drives the whole engine.

### How the pieces sit together (10-second view)

```
        ┌──────────────────────────────────────────────────────────┐
        │            BROWSER — Admin Panel (React, :3000)          │
        └───────────────┬──────────────────────────┬───────────────┘
                        │ login, users, prices     │ strategies, signals
                        ▼                          ▼
            ┌────────────────────┐      ┌──────────────────────────┐
            │  MAIN BACKEND :3002 │      │  STRATEGY ENGINE :3003   │  ◄── THIS REPO
            │  (issues JWT cookie)│      │  (verifies same JWT)     │
            └─────────┬──────────┘      └────┬─────────────┬───────┘
                      │                       │             │ listens
                      ▼                       ▼             ▼
              ┌──────────────┐      ┌──────────────────┐  ┌───────┐
              │  victory_db  │      │ victory_market_db│  │ REDIS │
              │ users/perms  │      │ strategies,      │  │ pub/  │
              └──────────────┘      │ signals, candles │  │ sub   │
                                    └────────▲─────────┘  └───▲───┘
                                             │  writes candles │ publishes
                                             │  + indicators   │ "candle_close"
                                    ┌────────┴─────────────────┴────┐
                                    │  Co-developer's DATA PIPELINE │
                                    │  (broker quotes → candles)    │
                                    └───────────────────────────────┘
```

You will see this same picture, in more detail, in
[02_ARCHITECTURE.md](02_ARCHITECTURE.md).

---

## Quick map of the backend folder

```
Backend/
├── server.js                  ← entry point: boots HTTP + WS + background jobs
├── migrateTsl.js              ← one-off script that added TSL columns
├── .env / .env.example        ← configuration (DB credentials, JWT secret…)
├── migrations/                ← reference SQL for the tables this service owns
└── src/
    ├── app.js                 ← the Express app (middleware + routes wiring)
    ├── config/
    │   ├── env.js             ← reads + validates .env, exports `env` object
    │   ├── constants.js       ← single source of truth: timeframes, indicators, operators
    │   └── logger.js          ← winston logger setup
    ├── db/
    │   ├── appDb.js           ← Postgres pool for victory_db (currently unused)
    │   ├── marketDb.js        ← Postgres pool for victory_market_db (used everywhere)
    │   └── redis.js           ← Redis connections (subscriber + publisher)
    ├── middleware/
    │   ├── auth.js            ← JWT check for every API route
    │   ├── validate.js        ← runs Joi schemas on request body/query
    │   └── errorHandler.js    ← converts thrown errors into JSON responses
    ├── routes/                ← URL → controller mapping
    ├── controllers/           ← parse request, call models/services, send response
    ├── models/                ← all SQL lives here
    ├── services/              ← business logic (the brain)
    │   ├── IndicatorService.js  ← computes SMA via TA-Lib
    │   ├── ConditionEngine.js   ← evaluates strategies, fires ENTRY/EXIT
    │   ├── SignalService.js     ← saves signal + broadcasts over WS
    │   └── BacktestService.js   ← replays history, simulates trades + P&L
    ├── validators/            ← Joi schemas (what a valid request looks like)
    ├── jobs/
    │   └── candleCloseHandler.js ← Redis listener that triggers everything
    ├── ws/
    │   └── signalWs.js        ← WebSocket server that pushes signals to users
    └── utils/                 ← small helpers (errors, responses, TSL math, target math)
```

Happy learning — start with [01_TOPICS_TO_LEARN.md](01_TOPICS_TO_LEARN.md).
