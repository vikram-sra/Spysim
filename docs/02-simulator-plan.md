# Spysim — Simulator Plan for the Volume + Retest Strategy

Goal: an **honest** event-driven simulator that can (1) replay SPY intraday
sessions bar by bar, (2) run the strategy variants from
[`01-volume-retest-strategy.md`](01-volume-retest-strategy.md) with realistic
fills and costs, and (3) tell us with statistical discipline whether
volume adds edge. If it doesn't, the simulator has still done its job.

## 1. Design principles

1. **No look-ahead by construction.** The engine only hands strategies data up
   to and including the just-closed bar. Indicators are incremental (streaming)
   objects, not vectorized over the whole day.
2. **One code path for backtest, replay and paper trading.** The same `Strategy`
   object consumes a bar stream from a file, a replay clock, or a live feed.
3. **Pessimistic by default.** Stops fill at the worse of stop price and bar
   open (gaps). Limit orders fill only on a trade *through* the price, not a
   touch. Spread and slippage are always applied.
4. **Every decision is logged.** Every state transition (ARMED → BROKEN → …)
   is recorded with the bar, the reason and the feature values, so failed
   setups can be studied as well as trades.
5. **Parameters are data.** YAML/JSON configs are hashed into run IDs so every
   result can be reproduced.

## 2. Tech stack

| Concern | Choice | Why |
|---|---|---|
| Language | Python 3.12 | Ecosystem, research speed |
| Data frames | **Polars** (pandas interop) | Fast with 1-min bars × 15+ years |
| Storage | Parquet, partitioned `symbol/year/` | Columnar, cheap, reproducible |
| Hot loop | Plain Python first; Numba on indicators if profiling shows it's needed | ~100k bars/yr/symbol is small |
| Config | Pydantic models + YAML | Validated params, hashable |
| Stats | NumPy, SciPy, statsmodels | Bootstrap, deflated Sharpe |
| Viz | Plotly (interactive candles + markers), matplotlib for reports | Inspect individual setups |
| UI (phase 4) | Streamlit or a static HTML report | Replay and parameter exploration |
| Tests | pytest + hypothesis | Synthetic-bar property tests |
| CLI | Typer | `spysim backtest`, `spysim sweep`, `spysim replay` |

## 3. Data

**Needed:** SPY 1-min OHLCV bars with consolidated (SIP) volume, including
premarket (04:00–09:30) and RTH, 2010→present. Phase 3 adds 1-sec bars or
trades/quotes for fill realism. ES/MES 1-min bars are optional.

| Source | Notes |
|---|---|
| **Databento** | Consolidated/exchange-level data, trades and quotes (MBP-1), historical back to about 2018 for equities; pay-per-use with starter credit. Best for fill-realism work. |
| **Polygon.io (Massive)** | 1-min aggregates with long history; minute bars are the floor below the top tiers. |
| **Alpaca** | Free data is **IEX-only volume**, a fraction of consolidated volume. Fine for prototyping, **wrong for RVOL research**. |
| **FirstRate Data** | One-time purchase, SPY 1-min from 2000. Good for long-history research. |

Data layer tasks:
- `DataSource` interface → `load_bars(symbol, start, end, tf) -> pl.DataFrame`
- Normalize to ET, tag sessions (pre / RTH / post), handle half-days and holidays (`pandas_market_calendars`).
- Validate: missing minutes, zero-volume bars, outlier prints, split/dividend policy (raw prices for intraday; levels never cross sessions without adjustment).
- Cache to Parquet; record the data version hash in every run.

## 4. Architecture

```
spysim/
  data/        sources (databento, polygon, csv), calendar, validation, cache
  features/    streaming indicators: ATR, VWAP(+bands), RVOL_bar, RVOL_day,
               volume profile (POC/VAH/VAL/HVN/LVN), swing pivots
  levels/      LevelProvider: PDH/PDL/PDC, PMH/PML, ORH/ORL(5/15/30),
               VWAP, prior POC/VAH/VAL, round numbers, swing pivots
               → Level(id, price, kind, strength, created_at, expires_at)
  strategy/    base.Strategy, volume_retest.SetupStateMachine (per level,
               per side), variants V0–V4, baselines (ORB, noise-band, random)
  execution/   order types (market, stop, limit), fill models,
               cost models (SPY shares, MES/ES), latency
  portfolio/   positions, sizing (fixed-R, fixed-fractional, Kelly-capped),
               risk limits (daily stop, max trades, kill switch)
  engine/      event loop: for bar in stream → features.update → levels.update
               → strategy.on_bar → execution.match → portfolio.mark
  analytics/   trade log, metrics, R-distribution, breakdowns, walk-forward,
               Monte Carlo, deflated Sharpe, reports
  viz/         setup charts, equity curves, heatmaps, replay
  cli.py
configs/       strategy & sweep YAMLs
tests/
notebooks/     exploratory research only (never source of truth)
```

### 4.1 Event loop (per bar, strict order)

```
1. bar closes                     (bar t is now known)
2. execution.match(open orders, bar t)   # orders placed at t-1 fill inside bar t
3. features.update(bar t)                 # ATR, VWAP, RVOL, profile
4. levels.update(bar t)                   # new/expired levels (e.g. ORH at 09:35)
5. strategy.on_bar(bar t, features, levels, portfolio) -> new orders
6. portfolio.mark_to_market(bar t); risk checks (daily stop, EOD flatten)
```

Orders created in step 5 can only fill at bar t+1 or later. This is the core
no-look-ahead guarantee, and a unit test enforces it.

### 4.2 Intrabar ambiguity

When a 1-min bar touches both the stop and the target, the **default assumes
the stop was hit first** (pessimistic). Phase 3 can resolve the order from
1-sec bars or trades when that data is available. The simulator reports how
often this ambiguity occurred, since a high rate means the bar timeframe is too
coarse for the strategy.

### 4.3 Fill and cost models (pluggable)

| Model | Market | Stop | Limit |
|---|---|---|---|
| `Naive` (debug only) | next open | stop price | touch |
| **`Realistic` (default)** | next open ± half-spread + k·ticks slippage | max(stop, open) if gapped, + slippage scaled by bar range / ATR | fills only if price trades ≥1 tick through |
| `Stress` | 2× slippage, +1 tick on every fill | | |

Costs: SPY at ~$0.0035/sh commission (configurable to $0) plus SEC/TAF fees on
sells. MES at about $0.62 per side per contract all-in (configurable).

## 5. Strategy module — the state machine

- One `SetupStateMachine` instance per (level, side). States: `ARMED, BROKEN,
  AWAIT_RETEST, RETESTING, TRIGGERED, IN_TRADE, DONE, INVALID` (see §5.2 of
  the strategy doc).
- Each transition emits a `SetupEvent(ts, level_id, from, to, reason, features)`.
- A `SetupArbiter` resolves conflicts: one position at a time, a priority
  order between levels, and a cooldown after a loss.
- All thresholds come from `VolumeRetestParams` (Pydantic), so variants are
  just configs: V0 = volume filters disabled, and so on.

## 6. Analytics — what "it works" must mean

**Per run:** number of trades, trades per day, win rate, avg win/loss in R,
**expectancy (R/trade)**, profit factor, net P&L, CAGR, Sharpe and Sortino
(daily), max drawdown in R and %, longest drawdown, worst day, exposure %, and
the share of P&L lost to costs.

**Breakdowns:** by level kind, side, hour, break-bar RVOL decile, retest/break
volume-ratio decile, day RVOL tercile, year, and exit variant.

**Funnel report:** setups ARMED → BROKEN → RETESTED → TRIGGERED → WON. This
shows *where* volume filters remove bad setups.

**Robustness protocol (the gate before any live money):**
1. **Split:** 2010–2019 for development, 2020–2023 for validation, and
   2024–present as a **holdout touched once**.
2. **Walk-forward:** a rolling 3-year train / 1-year test window, parameters
   re-chosen each window from a *coarse* grid.
3. **Parameter plateau:** heatmaps over pairs of key parameters. Require a broad
   profitable region, not a single sharp peak.
4. **Ablation:** V0 vs V1 vs V2 vs V3 vs the random-entry baseline at equal
   frequency. Volume must earn its place.
5. **Cost stress:** still positive at 2× costs.
6. **Bootstrap / Monte Carlo:** resample trades and days to get a 95% CI on
   expectancy and the max-drawdown distribution, plus risk of ruin at the
   chosen risk %.
7. **Deflated Sharpe ratio** (Bailey & López de Prado), accounting for the
   number of configurations tried. Every sweep logs its trial count.

**Go/no-go criteria for paper trading:** out-of-sample expectancy ≥ +0.10R
with the CI excluding 0, at least 300 OOS trades, PF ≥ 1.2 under `Stress`
costs, no single year below −15R, and V3 beats V0 by a statistically
meaningful margin.

## 7. Visualization and replay

- **Setup gallery:** for any trade or failed setup, a Plotly candle chart with
  levels, VWAP, volume bars coloured by RVOL, state-transition markers, and
  entry/stop/target lines. This is essential for checking that the code matches
  the intended pattern.
- **Replay mode:** step through a session bar by bar and watch the state machine
  update. This doubles as a manual "would I take this?" training tool.
- **Report:** a static HTML report per run (metrics, equity curve, R histogram,
  breakdown tables, heatmaps).

## 8. Testing strategy

- **Synthetic sessions:** generators produce bars with a scripted break → retest →
  continuation (and break → failed retest). Assert the exact state sequence and
  entry and stop prices.
- **Look-ahead guard:** run the strategy on data truncated at random times *t*
  and assert that decisions up to *t* are identical to the full run.
- **Fill-model unit tests:** gap through stop, limit touch vs trade-through,
  both-hit bars.
- **Indicator parity:** streaming ATR, VWAP and RVOL vs a vectorized reference
  implementation on real data.
- **Property tests (hypothesis):** P&L accounting identity (cash + positions =
  equity), no position after 15:55, and risk per trade ≤ configured R.
- **Golden run:** a small fixed dataset with a pinned config and a snapshot
  trade log, so regressions show up in CI.

## 9. Milestones

| # | Milestone | Deliverable | Exit criterion |
|---|---|---|---|
| **M0** | Skeleton | Repo layout, pyproject, CI (ruff, mypy, pytest), config models | CI green |
| **M1** | Data | Loader + Parquet cache + calendar + validation for SPY 1-min (start with one source) | 1 year loads, validation report clean |
| **M2** | Features & levels | Streaming ATR/VWAP/RVOL/profile; LevelProvider for PDH/PDL/PMH/PML/ORH/ORL/VWAP/round numbers | Parity tests pass; chart of one day shows correct levels |
| **M3** | Engine + execution | Event loop, orders, Realistic fill model, cost model, portfolio, risk limits | Look-ahead and fill tests pass |
| **M4** | Strategy | Volume-retest state machine, variants V0–V4, baselines (immediate breakout, ORB, random) | Synthetic-session tests pass; setup gallery checked by eye on 20 real days |
| **M5** | Analytics | Metrics, funnel, breakdowns, HTML report | One full-history run produces the report |
| **M6** | Research | Sweeps, walk-forward, ablation, bootstrap, deflated Sharpe → **research findings doc** | Answers the 8 hypotheses in strategy doc §6 |
| **M7** | Replay UI | Streamlit/HTML replay and parameter explorer | Can step through any day |
| **M8** | Paper trading *(only if M6 passes go/no-go)* | Live feed adapter (Alpaca/IBKR/Databento live) running the same Strategy object; daily sim-vs-paper reconciliation | 2–3 months of paper trading within the backtest CI |
| **M9** | Extensions | QQQ/IWM, ES/MES execution model, 1-sec fill resolution, 0DTE options payoff model | — |

## 10. Open decisions (defaults chosen; revisit if needed)

- **Data vendor:** default to Databento (consolidated volume, plus quotes for
  later fill work). Use FirstRate Data if pre-2018 history is needed cheaply.
- **Bar size:** 1-min signals, with a 5-min variant as a config option.
- **Primary instrument for P&L:** SPY shares first, with MES as a second cost model.
- **Holdout start:** 2024-01-01.
