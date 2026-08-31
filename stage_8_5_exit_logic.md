# Stage 8.5 — Exit Logic

**Status:** approved and implemented.
**Position in the build order:** inserted between Stage 8 (performance log
panel) and Stage 9 (unified alert).
**Depends on:** the Stage 8 signal log and panel.

The stages 1–10 specification covers entry machinery and does not include
exits. This document covers only Stage 8.5, which sits outside it.

---

## 0. Why this stage exists

The stages 1–10 specification is entry machinery end to end. The system opened
positions and never closed them.

The stage was also prompted by a calibration finding. On two large-cap names
the net score distribution was near-identical:

| Ticker | 70–79 band | 80+ band |
|---|---|---|
| GARAN | 0.8% of bars | 0.0% |
| AKBNK | 0.8% of bars | 0.0% |

The 60–69 band sits near 3.5–4%. The 70 threshold was structurally
unreachable rather than merely strict: layers 9, 10 and 11 return a true zero
on most bars by design, so roughly 15 of the 95 live weight points are
unavailable on an ordinary bar. The threshold default change ships in 8.5e.

---

## 1. Scope

### In scope

- A single-position state machine driven by the existing accepted signal.
- Eight exit rules, each independently toggleable, in the pattern the scoring
  layers already use.
- Parallel measurement of a fixed-R exit and a structural exit against every
  logged signal.
- Exit markers, labels and risk levels on the price chart.
- Two new performance panel sections reporting the comparison.

### Not in scope

- Position sizing, commission, slippage, compounding. Outcomes are **gross R**.
- Pyramiding or scaling out. One position at a time, one exit.
- Conversion to `strategy()`. This remains an indicator, and the panel remains
  a measurement instrument rather than a backtest report.

---

## 2. Position model

- **States:** `FLAT` → `LONG`/`SHORT` → `FLAT`. Held in `var` state that
  resets through the existing symbol-change path alongside the pivot,
  divergence, cooldown and log state.
- **Entry price** is the close of the signal bar, the same anchor the Stage 8
  log uses. A real fill would differ; that difference is documented, not
  modelled.
- **ATR is frozen at entry.** Stop and target distances derive from the ATR
  recorded on the signal bar, so levels never drift under an open position.
- **Signals arriving while in a position are ignored,** except that rule E7
  treats an opposite-direction signal as an exit. No pyramiding, and no
  reversal in a single step.
- **No re-entry on the exit bar itself.** A position closes and the next
  opportunity is the following bar at the earliest, which keeps the cooldown
  from being sidestepped by an exit and a re-entry landing on one bar.

---

## 3. The risk unit

| Quantity | Definition |
|---|---|
| 1R | `exitStopATR × atrAt` — price units, fixed at entry |
| Outcome | `(exit − entry) × dir / 1R` |

All three outcome tracks use the same R denominator, including the structural
track, which may not have used the stop that defines it. Without a shared
denominator a wide-stop model and a tight-stop model produce numbers that
cannot be laid side by side, and the comparison becomes one of leverage rather
than of exit quality.

---

## 4. Exit rules

| ID | Rule | Track | Default | Parameters |
|---|---|---|---|---|
| E1 | Hard stop | Both | On | `exitStopATR` 1.0 |
| E2 | Fixed target | Fixed-R | On | `exitTargetATR` 2.0 |
| E3 | Trailing stop | Both | **Off** | `exitTrailATR` 3.0 |
| E4 | Score decay | Structural | On | `exitScoreFloor` 40, `exitScoreBars` 2 |
| E5 | 1D bias flip | Structural | On | — |
| E6 | PAC cross against | Structural | On | `exitPacLine` mid |
| E7 | Opposite signal | Structural | On | — |
| E8 | Time stop | Both | On | `exitMaxBars` 250 |

Notes on individual rules:

- **E1** is the risk anchor. Its distance defines 1R, so it is present in both
  tracks whether or not it is what closes the trade.
- **E3** trails the favourable extreme achieved through the **previous** bar.
  Ratcheting on the current bar's own high and then testing that same bar's
  low against it would be intrabar lookahead.

  **Amended after implementation.** E3 was originally scoped to the fixed-R
  track. That left the structural model with no mechanism for holding a
  position: E4-E7 are all exit rules, none of them can keep a trade open, so
  the structural floor stayed at the fixed E1 stop for the life of the trade.
  On a strong trend that means the position is stopped out early or timed out
  by E8 and the structural rules never get to speak. Observed on BIMAS: 72%
  E1, 28% E8, zero structural exits, across a move from 80 to 425 - the
  structural rules were right to stay silent, and the floor closed the trade.
  E3 is now available to both tracks under its own toggle. On the structural
  track it is the trailing floor, with E4-E7 live as early-exit overrides.

  **Trail geometry.** For a long the trail sits at
  `entry + tMfe - trailATR x atr` against a hard stop at
  `entry - stopATR x atr`, so the trail only becomes the binding floor once
  `tMfe > (trailATR - stopATR) x atr`. It reaches breakeven at
  `tMfe = trailATR x atr` and locks in X R at `tMfe = (trailATR + X) x atr`.

  **Tested and rejected — see §12. E3 ships disabled.** Not because the trail
  is a poor rule, but because it pre-empts a better one.
- **E4** reads the **pre-veto directional** score. The post-veto score is
  zeroed whenever the daily bias goes neutral, so using it would fire E4 on
  every neutral daily bar — duplicating E5, far more often, and for a reason
  unrelated to the position's own case decaying.
- **E5** fires only when the daily bias turns **opposite**, never on neutral.
  Neutral is whatever ADX below the threshold produces; exiting on it would
  close most positions within a few bars and make E5 a disguised time stop.
- **E6** compares the same candle source the PAC is built from, so the test
  does not mix real and Heikin Ashi closes.
- **E8** backstops both models, guaranteeing every position resolves so the
  panel cannot fill with trades that never end. It is a backstop, not a
  decision: at the original 40 bars — roughly a calendar month on a 4h BIST
  chart — it was truncating trends rather than catching stuck positions, and
  a quarter of structural exits were hitting it. Raised to 250, which costs
  the fixed-R model nothing since a 1 ATR stop and 2 ATR target resolve long
  before it.

`exitStopATR` and `exitTargetATR` ship as **placeholders**. See §12.

---

## 5. Resolution order

Reason codes double as the order — the lowest code wins when several rules
fire on one bar.

1. **E1 hard stop**, then **E3 trailing stop**.
2. **E2 fixed target** — only reached if no stop fired on this bar.
3. **E4 → E7 structural**, in ID order; the first match is recorded.
4. **E8 time stop**, last, so a genuine reason always beats the backstop.

Price-level rules resolve against the bar's own range. State rules fill at the
close, since there is no level that was touched.

**Same-bar stop and target.** Nothing in the data says which came first — Pine
does not expose the intrabar sequence at this timeframe. The stop is assumed,
as the only assumption that cannot flatter the result, and it is exposed as an
input so the bias is testable rather than buried.

**Gaps.** A bar that opened beyond the stop never offered the stop price, so
the fill is the worse of the stop and the open. Applied to the **stop only**:
crediting the same price improvement on a target would flatter the result.

---

## 6. Three outcome tracks

Every logged signal carries three independent outcomes.

| Track | Role | Rules |
|---|---|---|
| Horizon | Control — what the entry was worth before any exit rule | none; fixed N-bar return |
| Fixed-R | Model A — mechanical, knows only distance from entry | E1 + E2 (+E3) + E8 |
| Structural | Model B — holds while the reasoning still stands | E1 + E4…E7 + E8 |

Both models are simulated on every signal regardless of which is live, so the
comparison never requires reconfiguring the script and re-reading the chart.
One model is nominated **live** by an input and alone drives the chart markers
and the Stage 9 exit alert.

A single combiner function resolves a model from its enable flags, with three
callers — the live position and both simulated tracks. A model is therefore
defined entirely by the flags handed to it, and the live path cannot behave
differently from the path that measures it.

### Two excursion windows

`mfe` and `mae` stop at the **horizon**. They exist to calibrate the stop and
target, and a stop calibrated from excursions measured inside a stopped trade
is circular: a trade cut at 1.0 ATR can never report an MAE above 1.0 ATR, so
the data would only ever confirm whatever stop was already set. The horizon
window is independent of every exit rule, and that independence is what makes
it usable as evidence.

A separate `tMfe` runs for the whole trade, because the trailing stop needs the
real high-water mark and has no interest in where the horizon fell.

---

## 7. Visuals

Direction is already carried by which side of the bar the exit cross sits on,
so colour carries what the entry marker cannot: whether the trade worked.

- **Exit marker** — cross on the exit bar, opposite the entry triangle. Teal
  for a non-negative R, maroon for a loss, gray when R was uncomputable.
- **Exit label** — reason and R, so a losing exit says immediately whether it
  was stopped, decayed or timed out.
- **Risk levels** — stop and target traced with a break-on-gap plot rather
  than line objects. No per-position drawing budget, levels kept on every
  historical position, and the line breaks between trades instead of joining
  two unrelated ones. The exit bar falls back to the levels captured before
  the position was cleared.
- **Score label** reports unrealised R while a position is open.

All sit under the *Show entry signals* master rather than the debug master: an
exit is something you act on, not something that explains a score.

---

## 8. Panel additions

### By exit model

Rows: `HORIZON`, `FIXED-R`, `STRUCT` (live model starred).
Columns: `n`, `win%`, `avg R`, `avg win`, `avg loss`, `bars`.

Average R is the column that decides between models — it already folds win
rate and payoff together, so a model with a worse win rate but better
expectancy is visibly the better rule.

> **Departure from the approved spec.** The spec listed `avg R` and
> `expectancy` as separate columns. They are arithmetically the same number,
> so the second was replaced by the average win and average loss split, which
> adds information rather than restating it.

### By exit reason

One row per rule E1–E8 for the live model, with count, share of exits and
average R. A structural model where most exits come from the time stop is not
a structural model.

### Suggested multiples

One row in the existing *Score reach* section: the 75th percentile of observed
MAE and the median observed MFE, both in ATR, beside the currently configured
stop and target. **Display only** — never auto-applied, since levels that
shifted as the log filled would make historical exits inconsistent with the
ones the user watched happen.

---

## 9. Changes to existing behaviour

- **Signal threshold default 70 → 60**, on the distribution evidence in §0. An
  existing user override is untouched. The tooltip now points at the panel's
  own reach section rather than asserting a number for every symbol.
- **Cooldown anchor moves from entry to exit.** With positions modelled, an
  entry-anchored cooldown is largely inert — a new same-direction signal is
  already blocked by the position being open. With exits disabled the Stage 6
  entry-anchored behaviour is preserved unchanged.
- **Stage 9 gains a second alert.** The single-unified-alert rule targets
  layer-level noise, not exits. An entry alert that cannot be paired with an
  exit alert is half a system.
- **Signals arriving while in a position** no longer print entry markers or
  reach the log.

---

## 10. Honesty notes

- **Intrabar resolution.** Stop and target hits are tested against the bar's
  high and low, which move while a bar forms and settle at its close. Same
  class of behaviour as the entry markers: judge on bar close.
- **Gross R only.** No commission, slippage, sizing or compounding. The panel
  measures exit quality, not account performance, and the difference matters
  most exactly where the numbers look best.
- **Sample size.** The log holds the last 25 signals by default, split across
  two models and four score bands. A win rate over a handful of resolved
  trades is decoration rather than evidence. The panel prints `n` beside every
  rate; read it first.
- **Label budget.** `max_labels_count` is 500. On a long history the earliest
  exits lose their labels while their markers and panel records remain.

---

## 11. Build order

| Commit | Contents |
|---|---|
| 8.5a | Position state machine, E1, E2, R accounting |
| 8.5b | Structural rules E4–E7, E8 time stop, E3 trailing stop |
| 8.5c | Dual-track simulation inside the signal log |
| 8.5d | Exit markers, labels and risk levels |
| 8.5e | Panel sections, suggested multiples, threshold default |

A correctness fix landed between 8.5d and 8.5e, restoring the horizon-bounded
excursion window described in §6.

---

## 12. The trailing stop was tested and rejected

Recorded here because E3 is the kind of rule that sounds obviously right and
will be proposed again. It is not enabled by default, and the reasoning below
is why.

### The argument for it

Every other structural rule is an exit rule. None of E4–E7 can keep a trade
open — they can only end one — so a structural model without a trail has no
mechanism for riding a trend, and its floor stays the fixed E1 stop for the
life of the trade. That argument is sound, and it is what motivated making E3
available to both tracks.

### What was measured

Clean retest: stop held at 1.0 ATR in both runs, 4h charts, n=25 each, only
`useE3` varied. Average R per signal, structural model:

| Ticker | E3 off | E3 on | delta |
|---|---|---|---|
| BIMAS | +0.74 | +0.78 | +0.04 |
| TOASO | +0.99 | +0.26 | **−0.73** |

The `HORIZON` control row stayed flat across both runs — BIMAS +0.52/+0.48,
TOASO +0.71/+0.58. That track applies no exit rule at all, so its stability
rules out the R-denominator artefact that made the first, confounded run
unreadable. The effect is real.

*(An earlier run varied stop width alongside `useE3` and is not reproduced
here; it measured a much larger apparent cost, most of which was the
denominator. It should not be cited.)*

### The mechanism: displacement, not destruction

**The trail is not a bad rule in isolation.** On TOASO it took a 28% share of
exits at +3.00 average R. That is a perfectly respectable rule.

The damage is to what it pre-empts. On the same ticker, E4 (score decay) went
from a 16% share at **+8.08** average R down to a 4% share at **+1.95**.

Two things happened there, and the second is the important one:

1. E4's *share* collapsed — the trail fired first on bars where both would
   have triggered.
2. E4's *average R* collapsed with it, from +8.08 to +1.95. The trail did not
   take a random sample of E4's trades. It took the best ones.

That second point is the whole finding. A trail sits a fixed distance below
the high-water mark, so the longer and further a position runs, the more
certain it becomes that some pullback eventually touches it. The biggest
winners are therefore the trades most likely to be intercepted before their
thesis actually decays. E3 systematically harvests exactly the trades E4 was
holding for.

BIMAS shows the same displacement, but E4 and E3 perform similarly on that
symbol, so it nets out to +0.04. **The cost of E3 is therefore a function of
how much better E4 is on a given name** — which makes it symbol-dependent,
and two tickers at n=25 cannot settle its size.

### Why this is not fixable by reordering the rules

The obvious response is to let the structural rules resolve before the trail.
That would be wrong. The trail is tested against the bar's low, so it is hit
*intrabar*; E4 depends on the score, which is only final at the close. On a
bar where both trigger, the trail genuinely was touched first, and a position
cannot un-hit a stop that has already filled. The current order reflects real
execution rather than an arbitrary preference.

### What remains untested

A trail wide enough not to compete with E4 — well beyond 3.0 ATR — might add
value as a pure catastrophe floor rather than as an exit rule, catching only
the collapses that E4 is too slow for. 5.0 ATR was tried once, in the
confounded run, so it has not really been measured. This is a hypothesis, not
a recommendation.

---

## 13. Decisions

| # | Question | Resolution |
|---|---|---|
| D1 | Structural track keeps the hard stop? | Yes |
| D2 | Stop before target on a same-bar tie? | Yes, with an input to flip it |
| D3 | Gap through the stop fills where? | Worse of stop level and bar open |
| D4 | E5 on neutral bias, or opposite only? | Opposite only |
| D5 | E6 line — mid or opposite band? | Mid, configurable |
| D6 | Live model by default? | ~~Fixed-R~~ → **Structural** |
| D7 | Cooldown anchored at entry or exit? | Exit |
| D8 | Separate exit alert in Stage 9? | Yes |
| D9 | Observed MFE/MAE for E1/E2 defaults? | **Deferred** |

**D3 note.** E3 was later made available to both tracks (§4), then disabled
by default on the evidence in §12.

**D6 was revised after testing.** The live model default is now Structural.
Fixed-R was chosen as the simpler baseline to judge against; once E3 became
shared, the structural model was the one worth watching on the chart.

**D9 is open.** At the 70 threshold the panel held essentially no resolved
signals, so there was no distribution to read — the readings needed to
calibrate the stop and target could not exist until the threshold change
shipped. E1 and E2 therefore ship at 1.0 and 2.0 ATR, which are round numbers
rather than measured ones.

**Recalibration loop.** Once the panel holds resolved signals across a few
tickers, read the *suggested multiples* row and set `exitStopATR` and
`exitTargetATR` from it. Nothing else in the stage depends on the outcome.
