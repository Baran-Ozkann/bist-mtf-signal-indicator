# Calibration Findings

What was measured, what it changed, and what remains unsettled.

The stages 1–10 specification defines the entry machinery; `stage_8_5_exit_logic.md`
covers the exit stage. This document is the evidence trail: the readings that
moved a default,
and the readings that refused to settle a question.

**Read every finding with its *n*.** Most of what follows rests on one or two
symbols. That is enough to change a default away from an arbitrary placeholder;
it is not enough to call anything settled.

---

## The headline finding — the layers do not earn their place, and the system loses to buy-and-hold

Measured across six BIST tickers with the ablation harness, after the harness
was validated against the manual method (§1c). Three results, all negative, all
pointing the same way.

### 1. Eleven of thirteen layers show no measurable contribution

`dR` is the change in average R per signal when the layer is removed, with
*Renormalise* on. **Negative means the layer helps; positive means it hurts.**
L1 was disabled in all six runs, after the BIMAS finding below.

| Layer | TUPRS | TOASO | ASELS | GARAN | THYAO | EREGL | helps |
|---|---|---|---|---|---|---|---|
| L5 veto | −1.53 | +0.03 | −0.37 | −0.16 | −0.62 | −0.07 | **5/6** |
| L3 momentum | −0.14 | −0.01 | −0.04 | +0.13 | −0.52 | −0.12 | **5/6** |
| L10 divergence | −0.27 | +0.18 | −0.29 | −0.18 | −0.20 | +0.05 | 4/6 |
| L4 volume | −0.34 | +0.31 | +0.06 | −0.02 | −0.32 | −0.05 | 4/6 |
| L11 candle | −0.13 | +0.25 | −0.29 | −0.15 | −0.28 | +0.20 | 4/6 |
| L8 index | −0.10 | +0.39 | −0.07 | −0.22 | +0.31 | −0.10 | 4/6 |
| L6 S/R | −0.08 | +0.24 | −0.05 | −0.24 | +0.37 | +0.31 | 3/6 |
| L7 volatility | −0.09 | +0.04 | −0.18 | −0.08 | +0.09 | +0.04 | 3/6 |
| L9 sweep | −0.06 | +0.47 | −0.31 | −0.37 | +0.40 | +0.25 | 3/6 |
| L2 PAC | −0.36 | +0.33 | −0.06 | +0.06 | +0.07 | +0.02 | 2/6 |
| L13 session | −0.10 | +0.22 | +0.05 | −0.26 | +0.40 | +0.25 | 2/6 |

Only two layers hold a consistent sign: **L5, the veto, and L3, momentum**,
both 5/6. The other nine flip sign from ticker to ticker. At n≈30–60 signals
per ticker, that is what a coin flip looks like. Nine layers of code, weight
and tuning surface produce nothing that shows up in the results.

### 2. The heaviest layer is actively harmful on at least one ticker

L1, the regime layer, carries **weight 20** — the largest single allocation in
the system, and the one the whole scoring scheme leans on hardest. On BIMAS,
removing it raised *both* average R and win rate.

This is the one result confirmed by hand rather than by the harness alone:
disabling L1 manually moved `HORIZON` from **+0.19 to +0.71**, against the
harness's predicted +0.26 — same sign, larger magnitude, which is the expected
direction of disagreement (§1c). L1 was left disabled for every subsequent
ablation run.

### 3. The system loses to buy-and-hold on every ticker tested

| Ticker | Buy-and-hold | System |
|---|---|---|
| ASELS | **+24468 R** | +23.60 |
| THYAO | **+3127 R** | +14.25 |
| TUPRS | **+12420 R** | +27.85 |
| EREGL | **+2267 R** | +9.47 |
| GARAN | **+1698 R** | +0.90 |
| TOASO | **+2995 R** | **−5.94** |

Six of six, by two to three orders of magnitude. The narrowest benchmark,
GARAN at +1698 R, still beats the system's +0.90 by a factor of roughly 1900.
On TOASO the system is outright negative while holding the stock returned
+2995 R.

### What this means

- **Eleven of thirteen layers show no measurable contribution.** The system's
  complexity is not buying anything that appears in the results.
- **The heaviest-weighted layer is harmful** on the one ticker where it was
  isolated cleanly, and was disabled for the rest of the work.
- **The whole apparatus underperforms the null strategy** — buy the stock and
  hold it — on every ticker tested.

There is no reading of these numbers in which the thirteen-layer design is
validated. What survives measurement is two layers, one of which is a veto.

### The caveats, stated in full

These bound the finding. They do not rescue it.

- **n is small.** 30–60 signals per ticker on the ablation runs. Nine layers
  reading as coin flips is a statement about the *absence of a detectable
  effect at this n*, not proof that each is worthless — a real but small edge
  would be invisible here.
- **The benchmark is not like-for-like.** Buy-and-hold compounds; the system
  risks a fixed unit per trade and never compounds. Part of the gap is that
  difference rather than skill. The *sign* of the comparison is not in doubt on
  any of the six; the multiple is overstated.
- **Six tickers is not the market.** All BIST, all large and liquid, over one
  window. A different universe or regime could read differently.
- **L1 was off in all six ablation runs.** Every other layer's `dR` is
  therefore measured against a system already missing its heaviest layer, and
  would not necessarily reproduce with L1 restored.

---

## 1. Stage 10 verification

### 1a. Static audit — complete

Verified by reading the source against the specification's checklist. No
defects found; no fixes were required.

| Check | Result |
|---|---|
| No hardcoded symbols | **Pass.** Only two exchange-prefixed literals exist in the file — the `idxSymbol` and `fxSymbol` `input.symbol` defaults. Every other reference is `syminfo.tickerid`. The remaining occurrences are comments, the indicator title and alert-message prefixes, none of which fetch data |
| Every layer independently toggleable | **Pass.** All twelve weighted layers have a `useLn` and a `wLn` reaching `f_aggregate`; layer 5 is the unweighted veto under `useVeto` |
| Weights total 100 | **Pass.** 20 + 10 + 15 + 10 + 10 + 5 + 5 + 5 + 5 + 5 + 5 + 5 = 100. With layer 12 off by default, 95 are live |
| Real OHLC vs Heikin Ashi | **Pass.** Layers 9, 10 and 11 take real `open`/`high`/`low`/`close` only, as do ADX, SMA, RSI, MACD, volume, ATR and every pivot. Heikin Ashi reaches only the PAC channel and the regime EMAs, via `srcOpen`/`srcClose`/`srcHigh`/`srcLow` |
| `request.security()` hygiene | **Pass.** Five calls, each with `lookahead = barmerge.lookahead_off` and an explicit `gaps = barmerge.gaps_off`. Count unchanged by stages 6–10 |
| Division and `na` guards | **Pass.** Every ratio helper guards a zero or `na` denominator and returns neutral 0.5 rather than `na`. Pin-bar wick ratios floor the divisor at 1% of range so a doji cannot divide by zero |
| Warm-up behaviour | **Pass.** Every layer returns neutral 0.5 while its inputs are `na`. Layers 9 and 10 distinguish a *known* zero (no pattern present) from an unknown (no levels yet), which is the correct distinction |
| `max_bars_back` | **Pass.** 1000, against a deepest default lookback of 200 (slow EMA) and a user ceiling well inside it |
| Session filter on half-days | **Pass.** Time-of-day tests only; nothing assumes a 10:00–18:00 session. See the layer 13 caveat in the README |
| Drawing budget | **Pass.** `max_labels_count` 500, documented behaviour on overflow |

### 1b. Bar-replay repaint protocol — to be run

Static reading cannot settle repaint behaviour; it has to be watched. Run on at
least two tickers, one liquid and one thin.

| # | What to do | Expected |
|---|---|---|
| 1 | Bar replay across ~200 bars with `biasConfirmedOnly` **on**. Compare the score line, raw score and 1D bias trace against the same bars viewed historically | **Identical.** Any movement is a bug |
| 2 | Watch the veto: find a bar where the daily bias is neutral | Score zeroed on that bar, both live and historically |
| 3 | Watch the entry and exhaustion gate traces (debug: gate states) within a forming 4h bar | **Will move.** Documented, not a defect — judge on bar close |
| 4 | Watch a pivot marker and the active S/R traces on the realtime bar | Levels **never** shift. A pivot appears only `srRight` bars after it printed and never moves afterwards |
| 5 | Let a position open and watch a stop/target level within the forming bar | Level static; the *hit* resolves intrabar and settles at close |
| 6 | Set `biasConfirmedOnly` **off** and repeat step 1 | Bias now mutates intrabar. Confirms the toggle does what it claims — debugging only |

Results: *not yet recorded.*

### 1c. Ablation harness cross-check — passed

The harness computes what the manual method would, without the reloads, so it
must be validated against the manual method once before its numbers are
trusted.

1. Read one layer's `dR` from the ABLATION section.
2. Disable that same layer by hand, with *Renormalise* **on**, and read the
   `HORIZON` row.
3. Compare.

**Expected: the same sign and rough magnitude, not the same number.** The
harness omits the position filter by design, so its BASE sees more signals than
the log does. A disagreement in *sign* means the harness is wrong and nothing
should be concluded from it until that is resolved.

**Result: passed.** On BIMAS, layer 1. The harness reported `dR` +0.26.
Disabling L1 by hand with *Renormalise* on moved the `HORIZON` row from **+0.19
to +0.71** — a manual delta of +0.52. Same sign, larger magnitude, which is the
disagreement the design predicts: the harness's BASE sees signals the position
filter keeps out of the log.

The harness's numbers are used from here on. Note what the check does and does
not establish — one symbol, one layer, and it shows the harness is not
sign-inverted, not that its magnitudes are accurate.

---

## 2. The threshold was unreachable — 70 → 60

The finding that started the calibration work.

Net score distribution, measured over all warmed-up bars:

| Ticker | 70–79 band | 80+ band |
|---|---|---|
| GARAN | 0.8% of bars | 0.0% |
| AKBNK | 0.8% of bars | 0.0% |

The 60–69 band sits near 3.5–4%.

**Mechanism.** Layers 9, 10 and 11 return a true zero on most bars *by design* —
an absent sweep, divergence or candle pattern is a known negative, not an
unknown. That makes roughly 15 of the 95 live weight points structurally
unavailable on an ordinary bar. The spec's default of 70 was therefore not
merely strict; it was above what the score could reach.

**Changed:** default threshold 70 → 60. **Also built:** the SCORE REACH panel
section, so the question is answerable on any symbol without guessing — it
counts bars rather than signals, which is why it still works on a chart that
has produced no signals at all. Its `peak` row turns red when the configured
threshold sits above anything the score has ever reached.

This is the finding that makes the panel's empty state legible. Before it, a
well-tuned system that had not yet triggered and a badly-tuned one that never
could looked identical.

---

## 3. ADX and EMA loosening tests

BIMAS, 4h, threshold 60, stop 1.0 ATR, `logMaxSignals` 25. One parameter set
varied per run, everything else held.

| Run | HORIZON | FIXED-R | STRUCT | net bars in 60–69 band | win% |
|---|---|---|---|---|---|
| **Baseline** — 1D EMA 21/50/200, ADX > 25 | **+0.35** | **+1.04** | **+0.54** | 326 (3.3%) | **48%** |
| EMA shortened to 13/34/89, ADX > 25 | +0.07 | +0.77 | +0.51 | 367 (3.7%) | 44% |
| EMA back to 21/50/200, ADX > 25 → 20 | −0.45 | +0.56 | +0.29 | 489 (4.9%) | 36% |

**Every loosening step raised the signal count and degraded quality
monotonically.** Both tracks move together and win% falls with them, so this is
not an artefact of one exit model. Loosening the ADX gate is the more damaging
of the two: it produced the largest reach gain (3.3% → 4.9% of bars) and the
only negative `HORIZON` reading in the set.

**Changed: nothing.** The baseline settings were kept — 1D EMA 21/50/200 and
ADX > 25. Nothing here argued for loosening either.

**Caveat, and it is a large one.** Single ticker, n=25 per run. The later n=100
retest showed that n=25 results do not hold. Read this as **direction, not
magnitude** — the ordering of the three runs is the finding; the numbers
themselves are not reliable at this sample size.

---

## 4. The trailing stop displaces a better rule — E3 ships disabled

Recorded in full in `stage_8_5_exit_logic.md` §12, including why it is not
fixable by reordering the rules. Summarised here because E3 is the kind of rule
that sounds obviously right and will be proposed again.

Clean retest — stop held at 1.0 ATR in both runs, 4h charts, n=25 each, only
`useE3` varied. Average R per signal, structural model:

| Ticker | E3 off | E3 on | delta |
|---|---|---|---|
| BIMAS | +0.74 | +0.78 | +0.04 |
| TOASO | +0.99 | +0.26 | **−0.73** |

The `HORIZON` control row stayed flat across both runs (BIMAS +0.52/+0.48,
TOASO +0.71/+0.58). That track applies no exit rule at all, so its stability
rules out the R-denominator artefact that made an earlier, confounded run
unreadable.

**The mechanism is displacement, not destruction.** The trail is not a bad rule
in isolation — on TOASO it took a 28% share of exits at +3.00 average R. The
damage is to what it pre-empts: on the same ticker E4 fell from a 16% share at
**+8.08** average R to a 4% share at **+1.95**. Both numbers moved, and the
second is the finding. The trail did not take a random sample of E4's trades;
it took the best ones. A trail sits a fixed distance below the high-water mark,
so the longer and further a position runs, the more certain it becomes that
some pullback eventually reaches it — the biggest winners are therefore the
trades most likely to be intercepted before their thesis actually decays.

**Unsettled:** BIMAS shows the same displacement but nets out to +0.04, because
E4 and E3 perform similarly on that name. The cost of E3 is a function of how
much better E4 is on a given symbol, which makes it symbol-dependent — and two
tickers at n=25 cannot size it. A trail wide enough not to compete with E4, as
a pure catastrophe floor rather than an exit rule, remains an untested
hypothesis.

**This finding is the direct reason the ablation harness exists.** A
symbol-dependent effect needs many symbols, and a method costing fourteen chart
reloads per symbol does not get repeated often enough to find one.

---

## 5. The sample was too small to survive its own exit rules — 25 → 100

Not a measurement so much as a structural fault found while reading the
measurements.

A log record is dropped when the log overflows, **regardless of whether its
exit tracks have resolved**. A record dropped while pending leaves the
statistics silently — it is not counted as a win, a loss, or a pending trade.
It simply is not there.

That is harmless when positions are short-lived. It stopped being harmless when
E8, the time stop, was raised from 40 to 250 bars: a position may now stay open
for 250 bars, and every signal firing inside that span competes for the same 25
slots. The window has to be comfortably larger than the number of signals that
can fire within one position's maximum lifetime, and 25 was not.

**Changed:** `logMaxSignals` default 25 → 100, with the constraint documented in
the input's own tooltip so the next person to raise E8 sees it.

**Related, and worth stating plainly:** `logMaxSignals` is the *only* cap on the
signal statistics. The band breakdown, the exit-model comparison, the
exit-reason shares and the suggested multiples are all computed by scanning
that log. SCORE REACH is the sole exception — it counts bars, never signals,
and is uncapped.

**Cost:** the per-bar update loop walks the whole log, so very large values add
execution time on long histories. 100 is a compromise, not a maximum.

---

## 6. Open questions

| # | Question | Status |
|---|---|---|
| Q1 | E1 and E2 defaults from observed MFE/MAE | **Open.** They ship at 1.0 and 2.0 ATR, round numbers rather than measured ones. The `suggest` panel row now produces the readings; they have not been applied |
| Q2 | Does any layer fail to earn its weight? | **Answered — eleven of thirteen do.** See the headline finding. Only L5 and L3 hold a consistent sign across six tickers; L1 is harmful on BIMAS |
| Q3 | Does the system beat buy-and-hold? | **Answered — no.** It loses on all six tickers tested, by two to three orders of magnitude. See the headline finding |
| Q4 | Is E3 viable as a wide catastrophe floor? | **Open.** Hypothesis only, never cleanly measured |
| Q5 | ADX / EMA loosening | **Recorded — see §3.** Every loosening step degraded quality; no default changed |
| Q6 | Does the L1 harm generalise beyond BIMAS? | **Open.** L1 was measured on one ticker and then disabled for every subsequent run, so the other five say nothing about it. It needs the same manual isolation on at least three more symbols |
| Q7 | What is left if the system is rebuilt on L5 and L3 alone? | **Open, and now the first question.** Nine layers show no detectable edge and the full system loses to holding the stock. A two-layer version has not been measured against either the current system or the benchmark |
