# Quantum Capital — AI Futures Trading Dashboard

Institutional‑grade Dash application for AI‑assisted crypto futures trading with a professional hedge‑fund UI, robust caching, and safe offline fallback.

## Overview
- Professional dark UI with metric tiles, analysis panels, and a high‑fidelity chart.
- Live and paper modes with defensive callbacks and SafeExchange offline fallback.
- AI market analysis and target price using OpenAI (with deterministic heuristics fallback).
- System Health panel: connection status, cache sizes, and latency telemetry (OHLCV/Ticker/Balance/Positions/Orders).
- Export tools: CSV (OHLC + indicators) and PNG (with branding/watermark toggle).

## Repository Layout
- `app_dash.py` — main Dash app (single‑file application)
- `requirements.txt` — Python dependencies
- `Makefile` — helper targets for setup, run, checks
- `.env.example` — environment template (no secrets)
- `.env` — your local environment (ignored by Git; create from example)
- `README.md` — this file

## Requirements
- Python 3.11
- macOS/Linux/WSL recommended
- Network access for live market data (Bitget via `ccxt`) and optional OpenAI analysis

## Quickstart
1) Create virtualenv and install dependencies
   - `python3.11 -m venv venv`
   - `source venv/bin/activate`
   - `pip install --upgrade pip`
   - `pip install -r requirements.txt`

2) Configure environment
   - Copy `.env.example` to `.env` and fill the following (do not commit `.env`):
     - `BITGET_KEY`, `BITGET_SECRET`, `BITGET_PASSWORD` (optional for live trading/balances)
     - `OPENAI_API_KEY` (optional for AI analysis)
     - `OPENAI_MODEL` — always set via `.env` to a model you have access to (e.g., `gpt-4o-mini`); if not set the app uses heuristics
     - `LIVE_TRADING=false` (recommended while testing)

3) Run the app
   - `make run`
   - Or directly: `source venv/bin/activate && python app_dash.py`
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
  - Trading Mode select: None (IDLE), Paper, Backtesting, Live
  - Policy Mode select: Strict (ENV), Hybrid (AI + guards), AI Only

## Fine‑Tuning & Execution
- Goal: tingkatkan akurasi entry dan minimalkan slippage eksekusi.
- Data & labels: triple‑barrier/ATR‑based; hindari look‑ahead; multi‑TF sinkron.
- Model: GBDT (LightGBM/XGBoost) baseline; DL sekuens (1D‑CNN/TCN/LSTM) opsional.
- Kalibrasi: Platt/Isotonic untuk confidence yang reliabel; threshold cost‑aware.
- Eksekusi: gate entry (HTF bias, OB proximity, jarak S/R), slip guard, limit/TWAP kecil, cancel/replace.
- Detail lengkap: lihat `documentation.md` bagian Fine‑Tuning & Execution.

## Architecture & Data Flow
- UI selects `symbol` and `timeframe` → background fetcher updates OHLCV cache.
- OHLCV data: cached on disk, fetched/merged incrementally; indicators computed in batch with memory‑optimized dtypes.
- AI: direction probability, textual analysis, and target price. If OpenAI is unavailable, deterministic heuristics keep UI informative.
- Chart: candlestick + overlays (Elliott Wave, Order Blocks, Fibonacci, VWAP, PSAR), order levels (Entry/TP/SL), watermark toggle.
- State & Caches: managed by a state manager with persistence for light state, in‑memory caches for heavy objects.
- System Health: latency telemetry (EMA) per subsystem, connection status, cache sizes, background thread states.

Flow (High‑Level)
```
Timer → Position detect → OHLCV refresh → Indicators/OB → S/R + ATR
   → AI Target (cached per candle)
   → simple_signal() + calibrated confidence + gates (ATR level, news, per‑TF)
   → Decision (BUY/SELL/HOLD)
   → If BUY/SELL (no pos): Size&Leverage → Order → TP/SL/Trailing
   → Build chart/tiles + AI Explain (cache) → System Health update
```

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
  - `OPENAI_API_KEY`, `OPENAI_MODEL` (use an accessible model; e.g., `gpt-4o-mini`). Note: always set `OPENAI_MODEL` in `.env`; do not rely on code defaults.
- News & refresh
  - `NEWS_ENABLE`, `NEWS_REFRESH_SEC`, `NEWS_MAX_ITEMS`
- Telemetry
  - Latency recorded internally and shown in System Health

- AI Orchestrator
  - Routing: `AI_PROVIDER_ORDER=openai,gemini,grok`
  - OpenAI: `OPENAI_MAX_RPM`, `OPENAI_MAX_TPM`, `OPENAI_MAX_CONCURRENCY`
  - Gemini: `GEMINI_API_KEY`, `GEMINI_MODEL`, `GEMINI_MAX_RPM`, `GEMINI_MAX_TPM`, `GEMINI_MAX_CONCURRENCY`
  - Grok: `GROK_API_KEY`, `GROK_MODEL`, `GROK_MAX_RPM`, `GROK_MAX_TPM`, `GROK_MAX_CONCURRENCY`
  - Timeframe & tokens: `AI_MIN_INTERVAL_*`, `AI_DEBOUNCE_MS`, `AI_MAX_TOKENS_FAST`, `AI_MAX_TOKENS_DEEP`

- Fine‑Tuning (signals)
  - Base thresholds: `ENTRY_CONF`, `ADX_MIN`, `SQUEEZE_MIN_BBWIDTH`, `P_UP_MIN_BUY`, `ATR_PCT_MIN`, `TP_ATR_MULT`, `SL_ATR_MULT`, `RISK_AVERSION`, `OB_PAD_ATR`, `OB_BONUS`
  - Level gates: `FT_BLOCK_NEAR_RES_ATR_BUY`, `FT_BLOCK_NEAR_SUP_ATR_SELL`
  - News gates: `FT_NEWS_LONG_MIN`, `FT_NEWS_SHORT_MAX`, `FT_RATE_HIKE_BLOCKS_LONG`, `FT_RATE_CUT_BLOCKS_SHORT`
  - Confidence calibration: `FT_CONF_W0`, `FT_CONF_W1`, `FT_CONF_W2`, `FT_CONF_W3`
  - Per‑TF gates: `ENTRY_CONF_1M/5M/15M/1H/4H/1D`, `RR_MIN_DEFAULT`, `RR_MIN_1M/5M/15M/1H/4H/1D`
  - AI Policy (Hybrid): `CONF_HARD_MIN`, `RR_HARD_MIN` — AI menyarankan ambang dinamis; sistem menerapkan batas bawah minimal untuk menjaga expectancy.
  - Policy Mode: `POLICY_MODE` (strict|hybrid|ai_only), `AI_ONLY_FALLBACK` (no_trade|hybrid)

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

## Changelog
- 2025-09-08 (later)
  - Docs: Tambah rencana Fine‑Tuning & Execution (akurasi entry + kualitas eksekusi) dan Project Structure.
- 2025-09-08
  - UI: Signal alert moved to header, displayed inline beside `ONLINE • LIVE/PAPER` to keep bottom area clean.
  - Docs: Updated README and documentation.md to reflect root-level layout, added note to always set `OPENAI_MODEL` in `.env`, and included direct run command.
  - No changes to trading logic or APIs.

---

Made by HarizDharma • Quantum Capital
This repository contains the minimal files to run the Dash app.
