# Quantum Capital — AI Futures Trading Dashboard
# Quantum Capital Dash App

Institutional‑grade Dash application for AI‑assisted crypto futures trading with a professional hedge‑fund UI, robust caching, and safe offline fallback.

## Overview
- Professional dark UI with metric tiles, analysis panels, and a high‑fidelity chart.
- Live and paper modes with defensive callbacks and SafeExchange offline fallback.
- AI market analysis and target price using OpenAI (with deterministic heuristics fallback).
- System Health panel: connection status, cache sizes, and latency telemetry (OHLCV/Ticker/Balance/Positions/Orders).
- Export tools: CSV (OHLC + indicators) and PNG (with branding/watermark toggle).

## Repository Layout
- `dash_app/`
  - `app_dash.py` — main Dash app (single‑file application)
  - `requirements.txt` — Python dependencies
  - `Makefile` — helper targets for setup, run, checks
  - `.env.example` — environment template (no secrets)
  - `README.md` — minimal note pointing to this document

## Requirements
- Python 3.11
- macOS/Linux/WSL recommended
- Network access for live market data (Bitget via `ccxt`) and optional OpenAI analysis

## Quickstart
1) Create virtualenv and install dependencies
   - `cd dash_app`
   - `python3.11 -m venv venv`
   - `source venv/bin/activate`
   - `pip install --upgrade pip`
   - `pip install -r requirements.txt`

2) Configure environment
   - Copy `.env.example` to `.env` and fill the following (do not commit `.env`):
     - `BITGET_KEY`, `BITGET_SECRET`, `BITGET_PASSWORD` (optional for live trading/balances)
     - `OPENAI_API_KEY` (optional for AI analysis)
     - `OPENAI_MODEL` (e.g., `gpt-4o-mini`; if unavailable the app uses heuristics)
     - `LIVE_TRADING=false` (recommended while testing)

3) Run the app
   - `make run`
   - Open `http://127.0.0.1:8050`

Convenience targets
- `make setup`         — Create venv and install dependencies
- `make syntax-check`  — Compile with `py_compile`
- `make import-check`  — Import smoke test (checks basic startup)
- `make run`           — Run Dash server

## Features
- Market Overview: 8 premium metric cards
  - Current Price, USDT Balance, Strategy Signal, AI Confidence, Position, Risk:Reward, Progress, AI Target
- Analysis Panel
  - Left: overlays (VWAP, PSAR, Fibonacci, Elliott Wave, Breakout/Fakeout, Watermark)
  - Right: AI Market Analysis (OpenAI if available; otherwise heuristics)
- Technical Section
  - Rich indicator grid and professional plotly chart with dark theme
- Export
  - PNG (with branding) and CSV (OHLC+indicator dataset)
- Alerts & Status
  - Signal alert banner (BUY/SELL) and header connection indicator (ONLINE/OFFLINE • LIVE/PAPER)

## Architecture & Data Flow
- UI selects `symbol` and `timeframe` → background fetcher updates OHLCV cache.
- OHLCV data: cached on disk, fetched/merged incrementally; indicators computed in batch with memory‑optimized dtypes.
- AI: direction probability, textual analysis, and target price. If OpenAI is unavailable, deterministic heuristics keep UI informative.
- Chart: candlestick + overlays (Elliott Wave, Order Blocks, Fibonacci, VWAP, PSAR), order levels (Entry/TP/SL), watermark toggle.
- State & Caches: managed by a state manager with persistence for light state, in‑memory caches for heavy objects.
- System Health: latency telemetry (EMA) per subsystem, connection status, cache sizes, background thread states.

## Offline‑Safe Operation
- Without credentials or network, the app automatically uses a SafeExchange stub:
  - UI and callbacks still render
  - Chart builds from cache if available
  - Trading actions are disabled (paper mode)

## Environment Variables (selected)
- Trading & runtime
  - `LIVE_TRADING` — enable real orders only when true and LIVE toggle is on
  - `PLOT_BARS`, `IND_MAX_LEN`, `DF_FLOAT_DTYPE` — performance/memory knobs
- Bitget (optional)
  - `BITGET_KEY`, `BITGET_SECRET`, `BITGET_PASSWORD`
- OpenAI (optional)
  - `OPENAI_API_KEY`, `OPENAI_MODEL` (use an accessible model; e.g., `gpt-4o-mini`)
- News & refresh
  - `NEWS_ENABLE`, `NEWS_REFRESH_SEC`, `NEWS_MAX_ITEMS`
- Telemetry
  - Latency recorded internally and shown in System Health

## Performance Tuning
- Use `float32` for indicator frames to cut memory ~50% (`DF_FLOAT_DTYPE=float32`).
- Limit indicator window length via `IND_MAX_LEN`.
- Control chart density via `PLOT_BARS`.
- Aggressive caches pruning and periodic GC are enabled in the app.

## Troubleshooting
- Port in use: “Port 8050 is in use” → stop the previous server (`pkill -f app_dash.py`) or run on a different port.
- OpenAI model error: 404 “model not found” → set `OPENAI_MODEL` you have access to (e.g., `gpt-4o-mini`) or leave blank to use heuristics.
- Rate limit (429) from exchange: the app throttles and backs off automatically; latency/err counters visible in System Health.
- White/empty chart: ensure data cache populated by selecting a valid `symbol` and `timeframe`, wait a few seconds for background fetcher.

## Security & Hygiene
- Never commit `.env` or secrets. Use `.env.example` as a template.
- Keep `LIVE_TRADING=false` unless you intend to trade.
- The app does not place live orders unless LIVE toggle is on and credentials are set.

## Roadmap / Next
- Deep E2E live testing (Bitget + OpenAI): balance/positions/order path, news analysis, latency telemetry.
- Toast notifications for order events (success/failure, partial TP, trailing SL, rate‑limit info).
- Export Excel multi‑sheet (dataset, indicator summary, performance snapshot).
- Internal rate‑limiting for order actions and extended audit trail.
- Unit/integration tests for indicators/Elliott/signal modules.

---

Made by HarizDharma • Quantum Capital
This folder contains the minimal files to run the Dash app. See README at repository root for usage.
