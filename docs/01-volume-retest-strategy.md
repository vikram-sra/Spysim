# Volume + Retest Strategy — Research & Specification

> Status: research / planning. Nothing here is investment advice. Every number
> below is a hypothesis to be validated by the simulator (see
> [`02-simulator-plan.md`](02-simulator-plan.md)).

## 1. What the strategy is

A **breakout-and-retest** trade with **volume as the gatekeeper** at each stage:

1. **Level** — a price level that many participants watch (prior-day high/low,
   premarket high/low, opening-range high/low, VWAP, volume-profile POC/HVN,
   round numbers).
2. **Break** — price closes *through* the level on **expanding** volume
   (participation: someone with size wants through).
3. **Retest** — price pulls back to the level on **contracting** volume
   (no one with size is pushing it back; old resistance absorbs as support).
4. **Trigger** — price rejects the level and resumes in the breakout direction
   with volume picking up again. Enter here, with the stop just past the retest.

The retest is what makes the risk small: the stop sits a few cents beyond a
level that *should* hold if the breakout was real. The volume filters are what
are supposed to separate real breaks from traps.

```
          break bar (high vol)
               │   ┌─ trigger (vol picks up) → ENTRY
   ────────────┼───┼──────────────  level (old resistance → support)
               │ ╲╱  retest (low vol)
  ...range...  │     STOP just below retest low
```

The mirror image (break of support → retest from below → short) is the same
strategy with signs flipped. A **failed** retest (price closes back through the
level on high volume) is itself a signal for the opposite trade (“trap”/fade) —
treated as a separate variant (§5.4).

## 2. Why it might work — the evidence, honestly graded

| Claim | Evidence quality | What it means for us |
|---|---|---|
| S/R levels exist because orders cluster there | **Good.** Osler (NY Fed SR125; J. Finance 2003): take-profit orders cluster *at* round numbers, stop-loss orders cluster *just beyond* them → reversals at levels, acceleration once crossed. | The "break" leg is partly stop-runs. Retest entries sit where the stop cascade has already fired. Levels that many people watch (round numbers, PDH/PDL, ORH/ORL) should matter most. |
| Unusual volume carries information | **Good, but at daily/weekly horizon.** Gervais, Kaniel & Mingelgrin (J. Finance 2001): abnormally high volume → higher returns over the next month (visibility effect). | Supports volume as a signal, but not directly at the 1–5 min horizon. Must be tested intraday. |
| Intraday momentum in SPY | **Good.** Gao, Han, Li & Zhou (JFE 2018): SPY's first half-hour return predicts the last half-hour; stronger on high-volume and high-volatility days. | Breakouts on high-volume days are more likely to continue. Use day-level RVOL as a regime filter. |
| Momentum breakouts out of a "noise band" in SPY are tradable | **Medium.** Zarattini, Aziz & Barbon, *Beat the Market* (SFI 24-97): SPY noise-area breakout with VWAP trailing stop, 2007–2024, Sharpe ≈1.33 net of $0.0035/sh commission + $0.001/sh slippage. Informal reimplementations got weaker results. | The best published SPY intraday breakout uses **VWAP** as the stop/trail, and its edge is modest. It's a good **baseline** for our simulator. |
| Relative volume is what drives ORB edge | **Medium–good for stocks.** Zarattini, Barbon & Aziz (2024), ORB on "stocks in play": plain ORB Sharpe ≈0.48, filtered to top-20 by opening RVOL → Sharpe ≈2.81 (2016–2023). A QuantConnect recreation got ≈2.4. Commentators note **the RVOL selection did almost all the work.** | **The single most important research finding for this project:** volume used as a *selection* filter is where the documented edge is. On a single ticker (SPY) we can't pick the top-20 movers, so RVOL becomes a "trade today / skip today" filter. |
| Plain ORB on index futures after costs | **Negative.** An independent replication of the 5-min ORB paper on five indices reproduced the gross result (~+0.13R/trade on NQ) but found **net ≈ zero** after spread and slippage. | Costs are the main risk. Any SPY edge has to survive realistic fills. |
| Break-and-retest win rates | **Weak.** These are vendor or blog claims: 45–65% win rates, PF ~1.3–1.6, often on tiny samples (one SPY daily test had 8 trades). One strict forex ruleset lost money (18% win rate, PF 0.43, 98 trades). | No credible published statistics exist. That's a reason to build the simulator, not to trust the claims. |
| Retail day traders as a group | **Strongly negative.** Barber, Lee, Liu & Odean (Taiwan): under 1% of day traders are predictably profitable after fees. Chague, De-Losso & Giovannetti (Brazil): about 97% of those who persisted 300+ days lost money. | The base rate is terrible. The simulator's main job is to **kill bad variants cheaply**, not to confirm a story. |

**Bottom line:** the mechanism (order clustering at levels, plus volume as
evidence of participation) is plausible and partly documented. The specific
"volume + retest" rule set has **no rigorous public backtest**. The edges that
are documented come from (a) **selecting high-relative-volume days or
instruments**, (b) **asymmetric payoff** (small stop, let winners run or trail
on VWAP) and (c) **low costs**. The plan is built around those three.

## 3. How one makes money with it — the arithmetic

### 3.1 Expectancy in R

Let *R* be the dollar risk per trade (entry − stop) × shares.

```
E[R] = p·W̄ − (1−p)·L̄ − C
  p  = win rate, W̄ = avg win in R, L̄ = avg loss in R (≈1 plus slippage), C = cost per trade in R
```

| p | W̄ | L̄ | C | E[R] / trade |
|---|---|---|---|---|
| 0.40 | 2.0 | 1.05 | 0.05 | **+0.12** |
| 0.45 | 2.0 | 1.05 | 0.05 | +0.27 |
| 0.35 | 2.0 | 1.05 | 0.05 | −0.03 (breakeven zone) |
| 0.25 | 4.0 | 1.05 | 0.05 | +0.16 (trend-trail style, like Zarattini ORB) |
| 0.45 | 2.0 | 1.05 | **0.25** | +0.07 (tight stops make costs dominate) |

### 3.2 Why costs, measured in R, are the hidden killer on SPY

SPY's spread is usually $0.01, so round-trip spread plus slippage plus fees comes
to roughly $0.01–0.03 per share. The cost in R is that figure **divided by the
stop distance**:

- stop $0.50 away → C ≈ 0.02–0.06R ✅
- stop $0.10 away → C ≈ 0.10–0.30R ❌ (wipes out most edges)

**Design consequence:** stops must be ATR-based with a minimum distance, never
"1 cent below the wick". The simulator must model this explicitly.

### 3.3 Turning expectancy into money

```
Annual P&L ≈ E[R] × trades/year × (account × risk%)
```

Example: E = 0.15R, 1 trade/day × 250 days, $50k account at 0.5% risk ($250/R)
→ 0.15 × 250 × $250 ≈ **$9.4k/yr (~19%)**, *before* drawdowns, taxes and
trading days with no setup. A plausible SPY-only setup frequency is
**0.3–1.5 qualified setups per day**. The simulator will tell us the real number.

Levers, in order of importance:

1. **Selectivity (higher p, higher W̄).** Trade only on high-RVOL days, only
   A-grade levels, only with the trend or VWAP side.
2. **Payoff shape (W̄).** Partial at 1R, then trail on VWAP or the 1-min swing,
   or hold to the next level. Avoid fixed small targets.
3. **Costs (C).** Minimum stop distance, limit-order entries on the retest
   where possible, low-commission venue. **MES/ES futures** cost less per unit
   of exposure, have no PDT rule and get 60/40 tax treatment in the US (§1256).
4. **Frequency.** Add more levels or a second instrument (QQQ, IWM) only
   *after* SPY alone shows an edge out of sample.
5. **Sizing.** Fixed-fractional 0.25–1% risk per trade. Use **quarter-Kelly at
   most**, because estimated edges are noisy. Daily loss limit of −3R. Turn the
   strategy off if rolling-100-trade expectancy goes below 0.

### 3.4 Instrument choice

| Instrument | Pros | Cons |
|---|---|---|
| SPY shares | Penny spread, deep book, simple fills | PDT rule (check current FINRA status), short-term capital-gains tax, capital-intensive |
| MES / ES futures | Tight, cheap, nearly 24h, 60/40 tax, no PDT | Tick size $1.25 (MES) or $12.50 (ES) per 0.25 pt; sizing is coarse for small accounts |
| SPY 0DTE options | Leverage, defined risk | Wide spreads in R, theta and gamma path-dependence; a different payoff that needs its own model (phase 2+) |

The simulator signals on SPY/ES prices and has **pluggable execution models**
per instrument.

## 4. Market-structure context for SPY

- Regular trading hours (RTH) are 09:30–16:00 ET. Volume has a U-shape (heavy
  at the open and close, light at midday), so **RVOL must be time-of-day
  normalized**. Comparing 10:00 volume with an all-day average is wrong.
- High-information times: 09:30–10:30, 14:00–16:00, and FOMC, CPI and NFP
  releases (8:30 ET). Midday breakouts fail more often (hypothesis to test).
- Watched levels, roughly in order of how many people watch them: prior-day
  high/low/close (PDH/PDL/PDC), premarket high/low (PMH/PML), opening range
  high/low (ORH/ORL; 5/15/30 min), session VWAP and anchored VWAPs, prior-day
  POC/VAH/VAL, round numbers ($1 and $5 SPY handles), and intraday swing
  pivots.

## 5. Strategy specification v0 (parameterized; defaults are starting guesses)

### 5.1 Definitions

- `ATR1` = 14-period ATR on 1-min bars; `ATRd` = 14-day daily ATR.
- `RVOL_bar(t)` = bar volume ÷ mean volume of the **same minute-of-day** over
  the last 20 sessions.
- `RVOL_day(t)` = cumulative session volume up to *t* ÷ mean cumulative volume
  up to the same time over the last 20 sessions.
- `tol` = level tolerance band = `max(0.10·ATR1, $0.03)`.

### 5.2 Long setup state machine (short is symmetric)

| State | Condition to advance | Invalidation (→ IDLE) |
|---|---|---|
| **ARMED** | Price within 1·ATRd below an active level, with RVOL_day ≥ `day_rvol_min` (1.0) | Level expires or session ends |
| **BROKEN** | A bar **closes** ≥ `break_buf` (0.10·ATR1) above the level, with RVOL_bar ≥ `break_rvol` (2.0) and body ≥ 50% of range | — |
| **AWAIT_RETEST** | Within `retest_window` (3–30 bars), a bar's low comes within `tol` of the level | Price runs > `max_ext` (0.5·ATRd) without retesting → missed, skip. Window expires. |
| **RETESTING** | Retest bars have average volume ≤ `retest_vol_ratio` (0.6) × break-bar volume, and no close below `level − 0.25·ATR1` | A close below `level − 0.25·ATR1`, or a retest bar with RVOL_bar ≥ `break_rvol` going the wrong way → **failed breakout** (variant 5.4) |
| **TRIGGER** | A bar closes above the prior bar's high (or is a rejection bar with lower wick ≥ 50%), with volume > average retest-bar volume | Retest lasts more than `max_retest_bars` (10) |
| **ENTRY** | Buy at next bar's open (market) **or** buy-stop at trigger high + $0.01 | Gap past entry by more than 0.5R → skip |

### 5.3 Trade management

- **Stop:** `min(retest_low, level − 0.25·ATR1) − $0.02`, with stop distance ≥ `min_stop` (`max(0.5·ATR1, $0.15)`). Skip the trade if the required stop is too wide (> `max_stop`, 0.35·ATRd).
- **Targets (variants to compare):** (a) fixed 2R; (b) 50% off at 1R, move stop to break-even, trail the rest on VWAP; (c) next level; (d) hold to 15:55.
- **Time rules:** no entries before 09:35 or after 15:30; flatten at 15:55; skip 2 minutes either side of scheduled macro releases (optional filter).
- **Session limits:** at most 3 trades per day, stop for the day at −2R, and one trade per level per direction.

### 5.4 Variants (for ablation, all in one engine)

1. **V0 Pure retest** — §5.2 with all volume filters **off**. This is the control.
2. **V1 Volume on break** only.
3. **V2 Volume on break + drying-up on retest**.
4. **V3 = V2 + day RVOL filter + VWAP-side filter** (long only above VWAP).
5. **V4 Failed-breakout fade** — enter against the break when a retest fails on high volume.
6. **Baselines:** immediate breakout entry with no retest; Zarattini 5-min ORB; Zarattini noise-band/VWAP; random entries with the same stop/target and frequency.

If V2 or V3 doesn't beat V0 and the random baseline **out of sample**, then
**volume adds nothing** and the thesis is rejected.

## 6. Hypotheses the simulator must answer

1. Does break-bar RVOL ≥ X raise the retest-hold rate? Plot hold rate against
   the RVOL bucket.
2. Does lower retest volume relative to break volume raise the win rate?
3. Which level types work best (PDH/PDL vs ORH vs VWAP vs round numbers)?
4. Time of day: morning vs midday vs afternoon.
5. Day regime: high vs low RVOL_day, trend vs range days, VIX level.
6. Exit style: which of 2R, trail VWAP, next level or EOD gives the best
   expectancy and drawdown?
7. Is the edge stable year by year, and does it survive 2× the modeled costs?
8. Setup frequency: what are realistic trades per day and per year?

## 7. Risks and failure modes

- **Overfitting:** many parameters × many variants. Mitigate with walk-forward
  testing, a held-out final period (touch it once), a deflated Sharpe ratio and
  parameter-plateau checks.
- **Look-ahead:** levels and RVOL must use only data available at bar close.
  Session volume profiles in particular must not leak.
- **Fill optimism:** stops fill past their price in fast markets, and limit
  orders on the retest do not always fill (touch ≠ fill).
- **Regime change:** 0DTE growth since 2022 has changed SPY intraday dynamics.
  Test 2023+ separately.
- **Behavioral:** live execution drifts from the rules. Paper trade, then trade
  small live, comparing fills against the sim.

## Sources

- Osler, C. — *Support for Resistance: Technical Analysis and Intraday Exchange Rates*, NY Fed SR125 — https://www.newyorkfed.org/medialibrary/media/research/staff_reports/sr125.pdf
- Gervais, Kaniel & Mingelgrin — *The High-Volume Return Premium*, J. Finance 2001 — https://rodneywhitecenter.wharton.upenn.edu/wp-content/uploads/2014/04/9901.pdf
- Lo, Mamaysky & Wang — *Foundations of Technical Analysis*, J. Finance 2000 — https://www.nber.org/system/files/working_papers/w7613/w7613.pdf
- Gao, Han, Li & Zhou — *Market Intraday Momentum*, JFE 2018 — https://alphaarchitect.com/2014/08/attention-prop-traders-the-first-half-hour-of-trading-predicts-the-last-half-hour/
- Zarattini, Aziz & Barbon — *Beat the Market: An Effective Intraday Momentum Strategy for SPY* — https://www.sfi.ch/de/publications/n-24-97-beat-the-market-an-effective-intraday-momentum-strategy-for-s-p500-etf-spy
- Zarattini, Barbon & Aziz — *A Profitable Day Trading Strategy for the U.S. Equity Market* (ORB + RVOL) — https://ideas.repec.org/p/chf/rpseri/rp2498.html ; QuantConnect recreation — https://www.quantconnect.com/research/18444/opening-range-breakout-for-stocks-in-play/
- ORB research summary — https://danfin.net/opening-range-breakout-research
- Independent ORB replication (gross reproduced, net zero) — https://www.mql5.com/en/blogs/post/776235
- Practitioner break-and-retest guides — https://atas.net/blog/break-and-retest-strategy/ , https://www.benzinga.com/money/break-and-retest-strategy , https://botsfolio.com/learn/break-and-retest
- Volume profile HVN/LVN guide — https://audacity.capital/fr/trading-guides/volume-profile-trading-strategy/
- Retail day-trader outcomes: Barber, Lee, Liu & Odean, *The Cross-Section of Speculator Skill* (Taiwan); Chague, De-Losso & Giovannetti, *Day Trading for a Living?* (Brazil, 2019)
