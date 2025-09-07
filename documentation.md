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
- `README.md` — main repository readme (quick reference)

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
  - `OPENAI_API_KEY`, `OPENAI_MODEL` (use an accessible model; e.g., `gpt-4o-mini`). Note: always set `OPENAI_MODEL` in `.env`; do not rely on code defaults.
- News & refresh
  - `NEWS_ENABLE`, `NEWS_REFRESH_SEC`, `NEWS_MAX_ITEMS`
- Telemetry
  - Latency recorded internally and shown in System Health

## Performance Tuning
- Use `float32` for indicator frames to cut memory ~50% (`DF_FLOAT_DTYPE=float32`).
- Limit indicator window length via `IND_MAX_LEN`.
- Control chart density via `PLOT_BARS`.
- Aggressive caches pruning and periodic GC are enabled in the app.

## AI Orchestrator (Compact Spec)
- Goals
  - Patuh limit RPM/TPM per provider+model, tetap terasa real‑time, output konsisten, dan compliant ToS.
  - Adaptif timeframe (1m/5m/15m+), rotasi otomatis lintas provider (OpenAI, Gemini, Grok), dan degradasi elegan.

- Providers & Models
  - OpenAI: fast `gpt-4o-mini` (default real‑time), deep `gpt-4o` (laporan terjadwal/eskalasi).
  - Gemini: fast `gemini-1.5-flash` (fallback/rotasi; free‑tier friendly), deep `gemini-2.5-pro`/`1.5-pro` (terjadwal/eskalasi).
  - Grok: fast/tertiary `grok-4-latest` (rotasi/fallback), opsional untuk deep eskalasi terbatas.

- Routing & Rotation
  - Urutan provider dibaca dari `AI_PROVIDER_ORDER` (default: `openai,gemini,grok`).
  - Fast lane (real‑time): pilih model cepat/hemat; Deep lane: hanya saat ambiguitas tinggi/event besar/terjadwal.
  - Headroom 10–20%: hentikan sebelum menyentuh limit keras; hindari oscillation dengan cool‑down per provider.

- Rate Limits, Concurrency, Backoff
  - Token‑bucket ganda per provider+model: satu untuk RPM, satu untuk TPM; concurrency guard per model.
  - Estimasi token: `prompt_tokens + max_output_tokens` dipakai untuk pacing TPM.
  - Retry 429/5xx: exponential backoff + jitter, hormati `Retry-After`; circuit breaker (open → half‑open → closed).

- Timeframe‑Aware Scheduling
  - 1m: trigger saat candle baru/perubahan signifikan; debounce 300–500 ms; batas 1 panggilan/menit per simbol; Deep lane: nonaktif.
  - 5m/15m+: trigger pada open/close candle; Deep lane diizinkan saat event besar (news/ATR spike) atau ambiguitas tinggi.
  - Cancel stale: batalkan request lama saat datang candle/input baru.

- Output Contract (JSON, lintas provider)
  - Fields: `decision` (long|short|flat|none), `target_price` (opsional), `rationale_bullets` (≤5), `risk_flags` (array), `confidence_explained` (singkat), `version`.
  - Batas output: fast lane ≤ 120 tokens; deep lane lebih panjang namun jarang/terjadwal.

- Caching & Deduplication
  - Cache key: `(symbol, timeframe, last_candle_ts, model_class, prompt_hash)`; TTL ≈ panjang timeframe.
  - Dedup: gabungkan permintaan identik yang sedang berjalan; Cancel: hentikan job yang menjadi stale.

- Degradation & Fallback Order
  - Kecilkan `max_output_tokens` → gunakan cache → rotasi provider (sesuai `AI_PROVIDER_ORDER`) → heuristik lokal.

- Telemetry & Ops
  - Metrik per provider+model: requests/min, tokens/min, 429 rate, latency, status circuit, ETA reset.
  - Panel System Health menampilkan provider aktif, alasan fallback/eskalasi, dan sisa kapasitas perkiraan.

- Security
  - Simpan kunci di `.env`; jangan tempel di log/chat. Rotasi kunci yang terekspos.

- Configuration (.env)
  - Routing: `AI_PROVIDER_ORDER=openai,gemini,grok`
  - OpenAI: `OPENAI_API_KEY`, `OPENAI_MODEL=gpt-4o-mini`, `OPENAI_MAX_RPM`, `OPENAI_MAX_TPM`, `OPENAI_MAX_CONCURRENCY`
  - Gemini: `GEMINI_API_KEY`, `GEMINI_MODEL=gemini-1.5-flash`, `GEMINI_MAX_RPM`, `GEMINI_MAX_TPM`, `GEMINI_MAX_CONCURRENCY`
  - Grok: `GROK_API_KEY`, `GROK_MODEL=grok-4-latest`, `GROK_MAX_RPM`, `GROK_MAX_TPM`, `GROK_MAX_CONCURRENCY`
  - Timeframe & tokens: `AI_MIN_INTERVAL_1M=60`, `AI_MIN_INTERVAL_5M=300`, `AI_MIN_INTERVAL_15M=900`, `AI_DEBOUNCE_MS=400`, `AI_MAX_TOKENS_FAST=256`, `AI_MAX_TOKENS_DEEP=768`
  - Catatan: selalu set `OPENAI_MODEL` via `.env`; isi nilai RPM/TPM sesuai kuota nyata, mulai konservatif dan kalibrasi dari telemetry/error header.
  - Fine‑Tuning (signals):
    - Base thresholds: `ENTRY_CONF`, `ADX_MIN`, `SQUEEZE_MIN_BBWIDTH`, `P_UP_MIN_BUY`, `ATR_PCT_MIN`, `TP_ATR_MULT`, `SL_ATR_MULT`, `RISK_AVERSION`, `OB_PAD_ATR`, `OB_BONUS`.
    - Gating dekat level: `FT_BLOCK_NEAR_RES_ATR_BUY`, `FT_BLOCK_NEAR_SUP_ATR_SELL` (ATR‑based distance gates).
    - News gates: `FT_NEWS_LONG_MIN`, `FT_NEWS_SHORT_MAX`, `FT_RATE_HIKE_BLOCKS_LONG`, `FT_RATE_CUT_BLOCKS_SHORT`.
    - Confidence calibration: `FT_CONF_W0`, `FT_CONF_W1`, `FT_CONF_W2`, `FT_CONF_W3`.
    - Per‑TF gates: `ENTRY_CONF_1M/5M/15M/1H/4H/1D`, `RR_MIN_DEFAULT`, `RR_MIN_1M/5M/15M/1H/4H/1D`.
  - AI Policy (Hybrid):
    - `CONF_HARD_MIN` — batas bawah confidence (default 55).
    - `RR_HARD_MIN` — batas bawah R:R (default 1.6).
    - AI akan menyarankan `conf_min` dan `rr_min` dinamis per konteks; sistem menerapkan max(hard_min, saran_AI).

- Implementation Plan (Non‑coding)
  - Rancang “AI Orchestrator” (limiter RPM/TPM, queue, concurrency guard, circuit breaker, routing).
  - Scheduler adaptif timeframe + debounce + cancel stale + cache; samakan skema output JSON lintas provider.
  - QA: simulasi 429/Retry‑After, uji 1m/5m/15m, failover OpenAI↔Gemini↔Grok, dan verifikasi metrik di System Health.

## Project Structure
- Root files
  - `app_dash.py` — single‑file Dash app (UI, data, signals, AI orchestration)
  - `requirements.txt` — Python dependencies
  - `Makefile` — setup/run/check convenience targets
  - `README.md` — quick reference and usage
  - `documentation.md` — detailed docs (specs, architecture, ops)
  - `.env.example` — environment template (no secrets)
  - `.env` — local environment (ignored by Git)
- Runtime artifacts
  - `logs/` — rotating application logs (trading, errors)
  - `state/` — persisted lightweight state/cache
  - `server.log` — recent server output (optional)
  - `venv/` — local virtual environment

## Project Logic Structure
- Config & Env
  - Loads `.env` via `python-dotenv`; all runtime knobs controlled by env (AI, thresholds, risk, performance).
  - Constants for thresholds (confidence/RR per TF), ATR gates, news gates, and orchestrator settings.

- Logging & Telemetry
  - `setup_quantum_logging()` configures console/file loggers and error logs.
  - `health_record()` tracks last/EMA latency per subsystem; exposed in System Health.

- Global State & Caches
  - `STATE`: app‑level dictionary for UI state (positions, ai_cache, system_health, etc.).
  - Caches: `CACHE` (OHLCV), `TICKER_CACHE`, `POS_CACHE`, `fig_cache`, `ob_cache`, `ai_cache`.

- Exchange Layer
  - `EX` from `ccxt` (Bitget) for ticker, OHLCV, balance, positions, orders.
  - `SafeExchange` fallback for offline operation (UI remains functional).

- Data Pipeline
  - `fetch_ohlcv_bitget_raw()` → low‑level fetch; `fetch_initial_bars()` and `fetch_incremental_bars()` for backfill/incremental.
  - `fetch_ohlcv_df()` merges, dedups, downcasts, and caches; `ensure_clean_tf_df()` enforces TF freshness and min bars.

- Indicators & Features
  - `compute_indicators()` builds EMA/RSI/MACD/Stoch/BB/VWAP/OBV, etc.
  - `swing_levels()`, `atr_*()` and pattern helpers; OB zones via `get_ob_zones_cached()`.

- AI Layer
  - Orchestrator `AIOrchestrator`: provider rotation (OpenAI→Gemini→Grok), basic RPM/TPM gating, concurrency guard, cooldown on 429, JSON/text unification.
  - AI calls unified through `AI.call_chat()`; functions: `analyze_news_pulse_with_ai()`, `ai_explain()`, `ai_predict_target()`, `ai_decide_scale_plan()`.
  - Cache per candle for AI text/targets; news analyzed on interval (`ensure_news_fetcher()`).

- Signal Engine (Fine‑Tuned)
  - Direction heuristics: `ai_predict_direction()` (vol‑normalized scores).
  - Core decision: `simple_signal()` with HTF bias, ATR S/R proximity, breakout/fakeout, oscillator/trend votes.
  - Fine‑tuning gates: ATR proximity blocks (avoid buy into resistance / sell into support), news‑aware gating (crypto sentiment, rate bias).
  - Confidence: blended heuristic + logistic‑calibrated confidence; HTF bonus; dynamic per‑TF minimum confidence.
  - Targets: `derive_tp_sl_mtf()` and `derive_tp_sl()` (ATR/structure), minimum RR per TF.

- Execution & Risk
  - Entry path in main callback: sizing (`choose_leverage()`, `compute_notional_usdt_enhanced()`), `place_market_order()`.
  - Post‑entry: ensure TP/SL, trailing stop, partial TP (ROE steps), and audit (`perf_*()` utils, `compute_roe()`).
  - Circuit breakers in practice: gates + thresholds avoid low‑quality entries; AI layer cooldown when rate‑limited.

- UI & Callbacks
  - Components: `create_logo_header()`, sidebar, metric cards, chart builder.
  - Clientside alert shows BUY/SELL beside ONLINE • LIVE/PAPER.
  - Server callbacks: `refresh_dashboard()` (core loop), `update_connection_status()`, `update_health()`, CSV export.

- Background Workers
  - `ensure_bg_fetcher()` for market data refresh; `ensure_news_fetcher()` for news pulse and AI analysis.

- Caching Strategy
  - Disk caches for OHLCV; in‑memory caches for figures/OB/AI. AI caches keyed by `(symbol, tf, last_candle_ts, kind)`.

- Tick Sequence (High‑Level)
  - Timer tick → heartbeat + live toggle → position detection.
  - Load/refresh OHLCV → compute indicators/OB → compute S/R + ATR proximity.
  - AI target (cached per candle) → `simple_signal()` (with TF/news gates) → derive TP/SL & RR plan.
  - If allowed and gates pass: size order → place order → ensure TP/SL/trailing.
  - Build chart/tiles → update AI explanation (cached) → update System Health/metrics.

## Flow Diagrams

Dashboard Tick Flow
```
[Timer/Inputs]
      |
      v
[Heartbeat + Live toggle + Position detect]
      |
      v
[Fetch/Refresh OHLCV + Cache] --(stale/backfill)--> [Merged OHLCV]
      |
      v
[Compute Indicators + OB zones]
      |
      v
[S/R (LTF+HTF) + ATR proximity]
      |
      +--> [AI Target (per‑candle cache)]
      |
      v
[simple_signal(): votes + breakout/fakeout + HTF bias]
      |
      v
[Calibrated Confidence (logistic blend) + Gates]
  (ATR‑level blocks, news sentiment/rate bias,
   AI Policy (conf_min/rr_min) with hard guards)
      |
      v
[Decision]  -> HOLD ───────────────┐
   | BUY/SELL (no position)        |
   v                               |
[Size & Leverage]                  |
   |                               |
   v                               |
[Place Order]                      |
   |                               |
   v                               |
[Ensure TP/SL + Trailing + Partial TP]
      |
      v
[Build Chart & Tiles + AI Explain (cache)]
      |
      v
[Update System Health + Caches]
```

AI Orchestrator Flow (OpenAI → Gemini → Grok)
```
[Messages + mode(json/text) + max_tokens]
      |
      v
[Estimate tokens]
      |
      v
[For provider in AI_PROVIDER_ORDER]
   |  Check: key present, cooldown elapsed
   |  Reset minute window; check RPM/TPM headroom
   |  Try acquire concurrency semaphore
   |   |
   |   +--> Call provider API
   |          - OpenAI client
   |          - Grok via OpenAI client (base_url x.ai)
   |          - Gemini REST (generateContent)
   |        On success: bump rpm/tpm, update System Health, return text
   |        On error/429: parse Retry‑After → set cooldown → release semaphore → try next
   |
   └── If none eligible/succeed → raise (fallback to heuristics where used)
```

## Fine‑Tuning & Execution (Compact Plan)
- Objectives
  - Tingkatkan akurasi sinyal entry, kurangi false positive, dan minimalkan slippage saat eksekusi.
  - Kalibrasi “confidence” yang dapat dipakai untuk sizing dan gating keputusan.

- Labeling & Data Hygiene
  - Target definisi jelas (mis. arah + threshold ret/ATR dalam N bar) atau triple‑barrier labeling (TP/SL/time‑out).
  - Hindari look‑ahead; sinkron waktu; gunakan multi‑TF (LTF/HTF) yang diselaraskan.
  - Sample weighting oleh volatilitas/regime agar tidak bias ke kondisi tenang.

- Features (ringkas, kuat, real‑time)
  - OHLCV multi‑resolusi, indikator inti (EMA cross, MACD hist, RSI, Stoch, BB, ATR).
  - Context: HTF bias, order block proximity, distance ke S/R berbasis ATR.
  - Microstructure (opsional): spread, imbalance, burst/velocity (jika data tersedia).
  - Exogenous ringan: news pulse (crypto/politics/rate) → fitur terbatas, terkalibrasi.

- Models & Training
  - Baseline kuat: Gradient‑Boosted Trees (LightGBM/XGBoost) untuk klasifikasi long/short/flat.
  - Alternatif DL: 1D‑CNN/TCN/LSTM untuk pola sekuens (latensi rendah, batch kecil).
  - Validasi: walk‑forward, time series split, early‑stopping; metrik: hit‑rate, MCC/F1, expectancy.
  - Kalibrasi probabilitas: Platt/Isotonic untuk confidence yang reliabel.

- Thresholding & Sizing
  - Optimasi threshold by cost‑aware objective (biaya, slip, funding) → maksimumkan expectancy/Sharpe.
  - Dynamic threshold oleh regime/volatilitas; gunakan ATR untuk normalisasi.
  - Position sizing: Kelly fraksi/vol targeting; cap leverage; guardrail risiko.

- Entry & Execution Quality
  - Gate entry: konfirmasi HTF bias + OB proximity + jarak ke S/R (hindari masuk tepat di level).
  - Hindari spread melebar/vol spike; no‑trade windows saat news berdampak.
  - Eksekusi: limit/iceberg/TWAP kecil pada likuiditas memadai; market hanya saat perlu dan dengan slip guard.
  - Cancel/replace bila tidak terisi dalam X detik atau kondisi berubah.

- Risk Controls & Circuit Breakers
  - Hard stop, max daily loss, pause saat performa buruk/429 API beruntun/latensi abnormal.
  - Audit trail keputusan (fitur, proba, threshold, alasan gating) untuk evaluasi.

- Evaluation & Monitoring
  - Backtest biaya realistis (maker/taker, funding, slip); OOS panjang; stress regime.
  - Track: hit‑rate, expectancy, MDD, PSR, turnover; alarm outlier.
  - A/B di paper mode sebelum live, kemudian progressive rollout.

- Implementation Roadmap (non‑coding outline)
  1) Siapkan dataset label (triple‑barrier/ATR target) + features multi‑TF.
  2) Latih baseline GBDT, validasi time‑split, kalibrasi probabilitas.
  3) Optimasi threshold cost‑aware + sizing rules; definisikan gate eksekusi.
  4) Integrasikan ke app: scoring ringan + gating entry + audit logging.
  5) Uji walk‑forward dan paper trade; iterasi parameter; rollout bertahap.

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
- 2025-09-08
  - UI: Signal alert moved to header, displayed inline beside `ONLINE • LIVE/PAPER` to keep bottom area clean.
  - Docs: Updated README and documentation.md for root-level layout, added note to always set `OPENAI_MODEL` in `.env`, and included direct run command.
  - No changes to trading logic or APIs.

---

Made by HarizDharma • Quantum Capital
This repository contains the minimal files to run the Dash app. See README for usage.
