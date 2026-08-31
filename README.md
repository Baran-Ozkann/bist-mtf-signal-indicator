# BIST Multi-Timeframe Signal System

A Pine Script v6 indicator for BIST-listed equities. It scores every bar
against thirteen independent layers, locks direction to the daily trend, times
entry on lower timeframes, manages the resulting position to an exit, and
reports what all of that actually earned.

Written from scratch. Conceptually inspired by SCALPTOOL R1.1 (JustUncleL), no
code in common. It works on any BIST ticker with no symbol hardcoded anywhere —
the two external references (index, FX) are `input.symbol` fields.

**It is an indicator, not a strategy.** It draws signals, fires alerts and
measures outcomes; it does not place orders, size positions, or model
commission and slippage. Every outcome it reports is **gross R**.

---

## Quick start

1. Add `bist_mtf_signal.pine` to a **4h chart** of a BIST stock.
2. Open *Settings → Performance Log* and read the **SCORE REACH** section
   before anything else. If the `peak` net score is below your threshold, the
   threshold is unreachable on this symbol and no amount of waiting will
   produce a signal. The cell turns red when that is the case.
3. Leave everything else at defaults until the panel holds resolved signals.

The indicator runs on any timeframe. 4h is the design point; the trade-offs of
running it elsewhere are in [Known limitations](#known-limitations).

---

## Timeframe architecture

| Timeframe | Role | Repaint-safe? |
|---|---|---|
| **1D** | Directional bias lock. `ADX > 25` **and** `close` vs `SMA(50)` **and** EMA ordering must all agree, or the bias is neutral. Acts as a **hard veto**: a candidate disagreeing with the daily bias has its entire score zeroed before display. | **Yes** — reads the last *closed* daily bar by default |
| **4h** | Primary signal timeframe. All thirteen scoring layers are computed here. | Yes |
| **1h** | Entry timing. Once the score crosses the threshold, requires a PAC touch **and** a directional confirmation candle. | Moves intrabar — see limitations |
| **15m** | Exhaustion gate. A boolean, not a score. Opens on **any one** of: RSI flattening or turning, a pin bar or engulfing, or N bars without a new extreme. | Moves intrabar — see limitations |

Five `request.security()` calls in total: bias, entry timing, exhaustion,
index, FX. All use `lookahead_off` with an explicit `gaps` setting, and each is
documented with its reasoning in the file header. Stages beyond the fifth call
add none, so this is the script's permanent footprint.

Heikin Ashi is derived arithmetically from the chart's own real OHLC rather
than fetched with a sixth call — which saves the call, is inherently
repaint-safe, and keeps real OHLC available on the same bar for the layers that
require it.

---

## How a signal is produced

Five gates in sequence. Each can be disabled independently, in which case it
passes through.

```
score >= threshold          scoreArmed
  and direction == 1D bias  (hard veto)
  and 1h PAC touch + candle entryReady
  and 15m exhaustion open   entrySignal
  and rising edge           entryFreshRaw
  and cooldown clear        entrySignalFresh
  and no position open      entryAccepted   -> marker, log, alert
```

The live score label on the last bar names the gate a setup is currently stuck
at, which is the difference between "this setup is worthless" and "this setup
is one exhaustion bar away".

---

## The thirteen layers

Each returns a confidence of 0.0–1.0, multiplied by its weight and summed.
Design weights total exactly 100. With layer 12 disabled by default, 95 points
are live; the *Renormalise* switch (on by default) rescales the score across
enabled layers so it always spans 0–100.

| # | Layer | Weight | Candles | Notes |
|---|---|---|---|---|
| 1 | Regime / trend filter | 20 | Real + HA | ADX, price vs SMA50, EMA ordering — confidence is met-conditions ÷ 3 |
| 2 | PAC crossover | 10 | HA | Fresh break of the channel, plus a pullback below PAC-mid within N bars |
| 3 | Momentum | 15 | Real | RSI vs its smoothing line (½) + MACD histogram turn (½) |
| 4 | Volume confirmation | 10 | Real | `volume / SMA(volume)` mapped through two anchors. Non-directional |
| 5 | **1D directional lock** | **veto** | Real + HA | Carries no weight. Zeros the whole score on disagreement |
| 6 | Structural level proximity | 10 | Real | Distance to nearest confirmed pivot in ATR units, with an optional headroom penalty |
| 7 | Volatility regime | 5 | Real | ATR level vs its baseline, blended with ATR slope. Non-directional |
| 8 | Market / index correlation | 5 | Real | Index agrees 1.0, inside the neutral band 0.5, disagrees 0.0 |
| 9 | Liquidity sweep | 5 | **Real only** | Wick pierces a confirmed level, close returns inside. Decays with age |
| 10 | RSI / price divergence | 5 | **Real only** | Between the last two *confirmed* pivots. Decays with age |
| 11 | Candle pattern | 5 | **Real only** | Pin bar or engulfing, tuned separately from the 15m gate |
| 12 | USDTRY correlation | 5 | Real | **Off by default** — requires declaring the company's FX linkage |
| 13 | Session / time filter | 5 | — | Opening, lunch and closing windows score a penalty. Timezone-aware |

**Mixed-candle policy.** PAC and the regime EMAs may use Heikin Ashi (default
on). ADX, SMA, RSI, MACD, volume, ATR and every pivot always use real OHLC.
Layers 9, 10 and 11 use real OHLC *mandatorily* — Heikin Ashi opens are
synthetic midpoints, so nearly every HA bar reads as an engulfing and the
patterns lose all meaning.

**Layers 9, 10 and 11 return a true zero on most bars by design.** An absent
pattern is a known negative, not an unknown. That means roughly 15 of the 95
live points are structurally unavailable on an ordinary bar — which is why the
threshold default is 60 rather than 70, and why you should read SCORE REACH on
your own symbol before trusting any threshold.

---

## Exits

Eight rules, each independently toggleable, resolving in reason-code order so
the lowest code wins when several fire on one bar.

| ID | Rule | Track | Default |
|---|---|---|---|
| E1 | Hard stop — **its distance defines 1R** | Both | On, 1.0 ATR |
| E2 | Fixed target | Fixed-R | On, 2.0 ATR |
| E3 | Trailing stop | Both | **Off** — see below |
| E4 | Score decay | Structural | On, floor 40 for 2 bars |
| E5 | 1D bias flip (opposite only, never neutral) | Structural | On |
| E6 | PAC cross against | Structural | On |
| E7 | Opposite signal | Structural | On |
| E8 | Time stop | Both | On, 250 bars |

Two models are simulated on **every** signal regardless of which is live, so
the panel compares them without you reconfiguring anything. One is nominated
live and alone drives the chart markers and the exit alert; **Structural** is
the default.

**E3 ships disabled on measured evidence.** Not because trailing is a poor rule
— in isolation it scored respectably — but because it *displaces* a better one.
Price levels resolve before state rules, so on any bar where both the trail and
E4 would fire, the trail wins; and because a trail sits a fixed distance under
the high-water mark, the trades it intercepts are disproportionately the
biggest winners. On TOASO it cut E4 from a 16% share at +8.08 average R to 4%
at +1.95. Full write-up in `stage_8_5_exit_logic.md` §12.

---

## The panel

Each section answers a different question. All are independently toggleable and
all are built only on the last bar.

| Section | Question |
|---|---|
| **SCORE REACH** | *Can the threshold even be hit on this symbol?* Counts bars, not signals, so it works on a chart that has never signalled. Raw and net columns separate a weak-layer problem from a veto problem. Also carries the suggested stop/target multiples. |
| **BY BAND** | *Of the signals that fired, which score bands paid, and what did they cost on the way?* Win rate, average return, MFE and MAE per band. |
| **BY EXIT MODEL** | *Which exit model is better?* Horizon control vs fixed-R vs structural, all on the same signals and all in the same R. |
| **BENCHMARK** | *Did any of this beat simply holding?* Total system R against buy-and-hold R over the same window. |
| **BY EXIT REASON** | *Which rule is doing the work?* A structural model where most exits come from the time stop is not a structural model. |
| **ABLATION** | *Which layers earn their weight?* Fourteen parallel streams — BASE plus each layer removed — in one pass. |
| **LAST SIGNALS** | The individual records behind the averages. |

`logMaxSignals` (default 100) is the **only** cap on the signal statistics.
Every figure except SCORE REACH is computed by scanning that log, so it sets
the *n* behind all of them. Read *n* before reading any rate.

### Reading the ablation section

Fourteen streams run in parallel on one chart: BASE, plus the same system with
each layer removed in turn. `dR` is the contribution — **positive means the
system scored better without that layer.**

Two things about it are deliberate and matter:

- **Every row is renormalised** over its own enabled weights, whatever the
  *Renormalise* setting says. Without that, removing a layer lowers the
  attainable maximum, fewer bars clear a fixed threshold, and the test measures
  *threshold sensitivity* instead of layer contribution. This is the trap in
  running the test by hand with the toggles.
- **The streams omit the position filter**, so `BASE` will not equal the
  `HORIZON` row above it — BASE normally sees more signals, because the live
  log drops signals arriving while a position is already open. That is a fact
  about timing, not signal quality. Compare ablation rows *with each other*,
  never against the other sections.

A `dR` of 0.2 over 30 signals on one symbol is noise. Check that the sign holds
across several tickers before acting on it.

---

## Alerts

Two conditions, and only two. Layer-level events (a PAC touch alone, a
divergence alone) deliberately never reach an alert.

- **Entry Signal** — fires on the fully-gated accepted entry. Message carries
  direction and score.
- **Exit Signal** — fires when the live position closes. Message carries
  direction, R multiple and the numeric E1–E8 reason code.

`alertcondition()` messages are compile-time strings and cannot concatenate a
series value, so dynamic figures travel through TradingView's
`{{plot("Title")}}` placeholders, resolved at fire time. Direction arrives as
`+1` / `-1` rather than the words LONG/SHORT, because a placeholder can only
resolve to a number; the mapping is spelled out in the message text.

---

## Calibration workflow

In this order. Each step depends on the one before it.

1. **Threshold** — read SCORE REACH. If `peak` net is below your threshold, no
   signal can ever fire; lower it until the 60–79 band holds a few percent of
   bars. This must come first: the other sections are empty until signals fire.
2. **Stop and target** — once the log holds resolved signals, read the
   `suggest` row: the 75th percentile of observed MAE and the median observed
   MFE, in ATR. Set `exitStopATR` and `exitTargetATR` from those. They ship at
   1.0 and 2.0, which are round numbers rather than measured ones. The
   suggestions come from the *horizon-bounded* excursion window, which is the
   only window independent of the stop being calibrated — a stop fitted to
   excursions measured inside stopped trades would only ever confirm itself.
3. **Weights** — turn on ABLATION and read `dR`. Repeat across several tickers
   before changing a weight.
4. **Exit model** — read BY EXIT MODEL and BY EXIT REASON together. Average R
   decides; the reason breakdown tells you whether the model is behaving the
   way its name claims.
5. **Reality check** — read BENCHMARK. If the edge over buy-and-hold is
   negative across several symbols, the calibration above is polishing
   something that is not working.

Do not skip to step 3. Ablation numbers computed under an unreachable threshold
describe nothing.

---

## Known limitations

**The 1h and 15m gates move within a forming bar.** On a 4h chart these are
*lower* timeframes, and Pine hands back only the last lower-TF value inside
each chart bar. The gate is also, by design, a realtime intrabar construct — a
gate that only resolved at 4h close would be useless for the job it has. The
confirmed-bar toggles reduce the movement but cannot remove it, because the
underlying limit is what `request.security` exposes, not the offset. **The
score and the 1D veto are fully repaint-safe and may be read at any time; judge
gated signals on chart-bar close.** Running the indicator on a 15m or 1h chart
removes the issue entirely, at the cost of a noisier signal timeframe.

**Layer 13 degrades on 4h.** The session test flags a bar when its *open* falls
inside a window. On a 4h chart only the opening window can ever match — no 4h
bar opens at 12:30 or 17:30 — so the layer effectively penalises the day's
first bar and nothing else. This is deliberate: a bar-span *overlap* test would
flag every bar on a 4h chart, since a 10:00–14:00 bar overlaps both the opening
and lunch windows. All three windows behave as intended on 15m and 1h.

**The cooldown is measured in bars, not wall-clock time.** Its real duration
therefore follows whatever timeframe you run on: four bars is 16 hours on a 4h
chart and one hour on a 15m chart. Re-check it when you change timeframe.

**Outcomes are gross R.** No commission, slippage, position sizing or
compounding. The panel measures signal and exit quality, not account
performance — and the difference matters most exactly where the numbers look
best.

**Intrabar exit resolution.** Stop and target hits are tested against a bar's
high and low, which move while the bar forms. Same class of behaviour as the
gates: judge on close. Where one bar covers both stop and target, the stop is
assumed to have come first — the only assumption that cannot flatter the
result, and it is exposed as an input so the bias is testable.

**Sample size.** The log holds the last 100 signals, split across two models
and four score bands. A win rate over a handful of resolved trades is
decoration. The window must also stay comfortably larger than the number of
signals that can fire inside one position's maximum lifetime, since a record
dropped while its exit tracks are still open leaves the statistics silently —
with E8 at 250 bars, the earlier cap of 25 was uncomfortably close to that
boundary.

**Drawing budget.** `max_labels_count` is 500. On a long history the earliest
exits lose their labels; their markers and panel records remain.

**Layer 12 needs a human.** FX linkage is a per-company fact the script cannot
derive from bar data — an exporter and an importer react to the same lira move
in opposite directions — so it ships disabled and stays neutral until you
declare the direction.

---

## Files

| File | Contents |
|---|---|
| `bist_mtf_signal.pine` | The entire indicator. Single file by design |
| `stage_8_5_exit_logic.md` | Exit logic specification, decisions, and the E3 evidence |
| `calibration_findings.md` | What was measured, and what it changed |

---

## Licence

Mozilla Public License 2.0.
