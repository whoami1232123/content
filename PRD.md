# Harmonic Pattern & PRZ Matrix Dashboard — Product Requirements Document

| | |
|---|---|
| **Status** | Draft v1.0 |
| **Date** | 2026-08-22 |
| **Product type** | Decision-support trading dashboard (non-executing) |
| **Markets** | Crypto, Forex, Metals/Commodities, Equities & Indices |
| **Source** | Derived from the "Harmonic Pattern Execution & PRZ Matrix" design brief, with the harmonic specification corrected — see [Appendix A](#appendix-a--audit-of-the-source-brief) |

> **Disclaimer.** This product is an analytical instrument. It does not provide financial advice,
> does not place orders, and does not guarantee outcomes. Harmonic pattern trading carries
> substantial risk of loss. See [§13](#13-disclaimer--regulatory-posture).

---

## 1. Overview

### 1.1 Problem

Harmonic pattern trading is arithmetic-heavy and unforgiving. A trader plotting a Gartley or Crab
by hand must compute a retracement of XA, a retracement of AB, a projection of BC, an AB=CD
completion, and then find the price band where several of those independently converge. Each of
those is a place to make a mistake, and the mistakes are not visible — an invalid pattern looks
exactly like a valid one on the chart.

The consequences are concrete. The design brief that seeded this product contained two worked
case studies; **both fail their own stated ratios**, one of them in four independent ways
including a completely inverted trade direction (Appendix A). These were not careless typos in an
otherwise sound method — they are the normal failure mode of doing this arithmetic manually, and
they are precisely what software should eliminate.

Existing tools sit at two unhelpful extremes. Broker-platform harmonic indicators auto-detect
patterns but repaint — a pattern present on this bar is gone on the next — and expose no
underlying ratios to audit. Manual Fibonacci tooling is honest but slow, and offers no validation
at all: it will happily let a trader mark a "Bat" whose B point is at 62.5%.

### 1.2 Product

A dashboard that treats the harmonic specification as **executable, auditable rules** rather than
chart decoration. It:

1. **Scans** live and historical data for the five core patterns, using confirmed pivots only.
2. **Validates** every candidate against the full ratio set — including BC projection and AB=CD,
   which most tools omit — and rejects anything that fails a structural rule.
3. **Computes the PRZ as a convergence interval**, showing which ratios contribute to it and how
   tightly they cluster, not as a single price.
4. **Derives a complete risk plan** — invalidation, stop with a real buffer, two targets on the AD
   leg, R:R, and position size — and refuses setups below the R:R floor.
5. **Lets the trader plot manually** and re-runs the same validation live, so the engine's
   judgement is always inspectable.

### 1.3 Users

| Persona | Need | Primary surface |
|---|---|---|
| **Discretionary swing trader** | Wants a shortlist of D-completions maturing this week across a watchlist, with the arithmetic already done | Scanner + PRZ detail |
| **Intraday trader** | Wants an alert when price enters a PRZ on H1/H4, plus a confirmation gate before committing | Alerts + confirmation panel |
| **Analyst / educator** | Wants to plot a historical structure and have the tool show exactly why it is or isn't a valid Bat | Manual plotting mode |
| **Systematic researcher** | Wants per-pattern historical hit-rates on their own data, not vendor marketing claims | Backtest module (P3) |

### 1.4 Goals

- **G1** — Make an invalid pattern impossible to act on unnoticed. Every rejected rule is shown.
- **G2** — Every number on screen is reproducible from the raw OHLC by a documented formula.
- **G3** — No repainting. A pattern, once surfaced, never silently disappears.
- **G4** — No setup reaches "actionable" without a stop that lies outside its own PRZ and R:R ≥ 2:1.
- **G5** — One engine, four asset classes, via an adapter boundary.

### 1.5 Non-goals

- **Does not place, modify, or cancel orders.** No broker credentials are ever collected or stored.
  The order ticket is a formatted summary the user copies or re-keys into their own platform.
- **Not a signal service.** No shared or sold signal feed; no social/copy-trading.
- **Not financial advice**, and no claim of predictive accuracy.
- **No pattern discovery.** The pattern set is a fixed, documented specification (§3), not a
  machine-learned or user-defined one, in v1.
- **Not a charting platform.** It renders enough chart to show the pattern; it does not compete
  with general-purpose charting.

### 1.6 Platform decision: TradingView / Pine Script v6

Phase 1 (§11) ships as a **TradingView Pine Script v6 `strategy()`**, run on TradingView's own
charts and data. This was chosen over a standalone web build because it reuses an existing feed,
chart, and — critically for §12 — an existing historical **Strategy Tester** to validate the
engine against real bars, rather than requiring a bespoke backtest harness before anything can be
proven. Chosen over a plain `indicator()` because a `strategy()` is what makes §12's win-rate /
profit-factor / drawdown numbers real, not asserted: TradingView computes them from the script's
own `strategy.entry` / `strategy.exit` calls replayed across history.

This is a real constraint, not a free upgrade, and it reshapes parts of the spec:

| Spec section | On TradingView | Disposition |
|---|---|---|
| §3 normative ratios, §6 PRZ, §7.1–7.6 risk/stop/target rules | Implement exactly as specified — pure arithmetic, no platform dependency | **In scope, v1** |
| §5.2 anti-repainting (D1–D5) | `ta.pivothigh(depth, depth)` / `ta.pivotlow(depth, depth)` are non-repainting by construction — a value is only ever returned `depth` bars after the pivot bar, once confirmed | **In scope, v1** — this is the mechanism, not a reimplementation of it |
| §12 historical validation | Native: apply the script to a symbol/timeframe, open the **Strategy Tester** tab | **In scope, v1** — see §11.5 below |
| §5.3 candidate enumeration | One chart = one instrument = one timeframe. A watchlist scan across 50 instruments (§12's engineering target) is not something one Pine script does | **Deferred.** Phase 1 ships as a per-chart script; a cross-symbol scanner is a separate TradingView Screener build or an external service, out of scope here |
| §7.7 portfolio-level limits (correlation clusters, daily loss cap, max concurrent risk) | Pine has no state that persists *across* symbols or *across* sessions independent of the chart it's running on | **Deferred to a companion service.** The ratio/PRZ/risk engine (§3–§7.6) is portable logic; portfolio aggregation needs a backend that Pine cannot provide. Noted as a gap, not silently dropped |
| §8 manual plotting with live recompute | TradingView's manual Fib/XABCD drawing tools do not talk to a script's validation logic | **Deferred.** v1's validation is scanner-only; a manual mode would need a second, browser-hosted build reusing the same ratio module |
| §9 UI (status states, portfolio heat panel, order ticket) | Pine draws on-chart (lines, boxes, labels) and posts `alertcondition()`/`alert()` text; it does not render a side panel | **Simplified.** Status is conveyed via plot color, box label text, and alert payload — not the full §9.1 layout |

**Tolerance implementation note.** §2.4 defines tolerance as a price width (`tolerance_pct × |A−X|`)
applied to each contributor price. The Pine v1 script applies `tolerance_pct` directly in **ratio
space** to every constraint window (B, C, BC-projection, AB=CD, D) for implementation simplicity.
This is the convention essentially all public Pine harmonic scripts use, and is *not* identical to
§2.4's leg-relative price tolerance — it is documented here as a known v1 simplification, not a
silent deviation. The §6.2 PRZ envelope itself, however, **is** computed the spec-faithful way: the
three contributor prices (D-of-XA, BC projection, AB=CD) are converted to prices first, and a single
`tolerance_pct × |A−X|` price band is applied around their min/max.

---

## 2. Measurement conventions

Ambiguity here is the single largest source of error in harmonic tooling, so the conventions are
normative and stated before any ratio.

### 2.1 The projection formula

For a leg from point **A** to point **X**, a ratio **r** projects to:

```
P(A, X, r) = A + r · (X − A)
```

The Fibonacci anchor sits at **A**, with **100% at X**. Values of `r > 1` extend *beyond X*.

This is the convention under which the standard statement "the Butterfly and Crab terminate beyond
point X" is true, and under which "the Gartley and Bat never breach X" is true. It must be used
consistently. Anchoring the other way inverts every extension pattern and produces a trade in the
wrong direction — this is exactly the failure in Case Study 2 of the source brief.

**Worked check.** Bullish Butterfly, X = 100 (low), A = 200 (high). D at 1.270:
`P(200, 100, 1.270) = 200 + 1.270 × (−100) = 73`. D = 73 lies below X = 100. Correct: the Butterfly's
D makes a new extreme beyond X.

### 2.2 Leg and ratio definitions

| Term | Definition |
|---|---|
| **XA leg length** | `\|A − X\|` in price units. The reference scale for all tolerances. |
| **B retracement** | `\|A − B\| / \|A − X\|` — B as a fraction of XA |
| **C retracement** | `\|C − B\| / \|A − B\|` — C as a fraction of AB |
| **BC projection** | `\|C − D\| / \|C − B\|` — how far CD extends the BC leg |
| **AB=CD ratio** | `\|C − D\| / \|A − B\|` — CD leg measured against AB |
| **AD leg** | `\|A − D\|` — the reference leg for take-profit targets |

### 2.3 Orientation

A pattern is **bullish** when D is a low and the trade is long; **bearish** when D is a high and the
trade is short. Orientation is determined solely by the sign of `(A − X)`:

- `A > X` (XA rises) → D projects downward → **bullish**, long at D.
- `A < X` (XA falls) → D projects upward → **bearish**, short at D.

The engine derives orientation; it is never a user-supplied label. **Requirement:** if a manually
plotted pattern's derived orientation contradicts a user-selected direction, the UI blocks the
setup and shows the derivation.

### 2.4 Tolerance

Tolerance is expressed as **a fraction of the XA leg length, in price units**:

```
tolerance_price = tolerance_pct × |A − X|
```

A ratio matches if the actual price falls within `± tolerance_price` of the ideal ratio's price.

This must not be confused with a percentage of the instrument's price. On an instrument at 68,000
with an XA leg of 4,720, "±1%" means **±47**, not ±680 — a 14× difference that would turn a strict
filter into a meaningless one. **Requirement:** every tolerance shown in the UI displays both the
percentage and its resolved price width.

---

## 3. Normative harmonic specification

This section is the engine's source of truth. Implementations must derive constants from here.

### 3.1 Universal structural rules

These are hard gates. A candidate failing any rule is rejected regardless of how well its ratios fit.

| ID | Rule | Rationale |
|---|---|---|
| **S1** | Points must alternate direction: X→A→B→C→D strictly zig-zags | A non-alternating sequence is not a harmonic structure |
| **S2** | **C must not exceed A** | If C passes A the AB leg is fully retraced; the structure has broken |
| **S3** | **B must not exceed X** | Same reasoning applied to the XA leg |
| **S4** | For Gartley and Bat, **D must not exceed X** | These are retracement patterns; breaching X is their invalidation |
| **S5** | For Butterfly and Crab, **D must exceed X** | These are extension patterns; D failing to pass X means the pattern hasn't completed |
| **S6** | All five (or six) points come from **confirmed** pivots (§5.2) | Anti-repainting |
| **S7** | Each leg must span ≥ `min_leg_bars` (default 3) | Rejects noise micro-structures |

### 3.2 The four-point patterns

All ratios are of the leg named in the column header. **Bold** = the pattern's defining ratio.

| Pattern | B (of XA) | C (of AB) | BC projection | AB=CD | D (of XA) | Invalidation | Stop level |
|---|---|---|---|---|---|---|---|
| **Gartley** | **0.618** | 0.382 – 0.886 | 1.130 – 1.618 | 1.000 (equal) | **0.786** | D > 1.000 XA (breaches X) | 1.050 XA |
| **Bat** | 0.382 – 0.500 | 0.382 – 0.886 | 1.618 – 2.618 | 1.270 – 1.618 | **0.886** | D > 1.000 XA (breaches X) | 1.050 XA |
| **Butterfly** | **0.786** | 0.382 – 0.886 | 1.618 – 2.240 | 1.270 (ext.) | **1.270 – 1.618** | D > 1.618 XA | 1.750 XA |
| **Crab** | 0.382 – 0.618 | 0.382 – 0.886 | **2.240 – 3.618** | 1.618 (ext.) | **1.618** | D > 1.618 XA (toward 2.000) | 1.902 XA |

**Notes.**

- **BC projection is not optional.** It is the second constraint whose intersection with the XA
  ratio *forms* the PRZ. A tool computing only the XA ratio is computing a price, not a PRZ.
- **The Crab is identified by its BC projection**, 2.240–3.618 — the widest in the set. Two
  candidates can both terminate near 1.618 XA and only the BC projection distinguishes a Crab
  from a Butterfly that overshot.
- **Stop levels leave a real buffer.** Setting the Crab's stop at 1.618 — its own D — produces a
  zero-width stop, which is unfillable. Verified gaps: Gartley +0.264 XA, Bat +0.164, Butterfly
  +0.132 (from the 1.618 far edge), Crab +0.284.

### 3.3 The five-point pattern: Shark

The Shark uses **O-X-A-B-C** labelling, where **C is the terminal (entry) point** — it is not a
four-point XABD pattern with a renamed vertex, and it will not fit a four-point schema. This drives
the polymorphic data model in §4.

| Leg | Ratio | Of |
|---|---|---|
| **AB** | 1.130 – 1.618 | extension of XA |
| **BC** | 1.618 – 2.240 | extension of AB |
| **C (terminal)** | **0.886 – 1.130** | retracement of **OX** |
| Invalidation | C > 1.130 OX | |
| Stop level | 1.250 OX | (gap +0.120) |
| Target | 0.500 retracement of BC | (Shark uses its own target rule — see §7.3) |

### 3.4 Deferred patterns

Out of scope for v1, but the data model must accommodate them without migration: **Cypher**
(B 0.382–0.618 XA, C 1.272–1.414 XA, D 0.786 of XC), **Deep Crab** (B 0.886, D 1.618),
**Alt Bat** (B 0.382, D 1.130), **5-0** (five-point, D at 0.500 BC), **Three Drives**, and bare
**AB=CD**.

---

## 4. Data model

### 4.1 Core entities

```
Instrument
  symbol, asset_class, tick_size, price_precision,
  contract_size, quote_currency, session_profile

Bar
  instrument_id, timeframe, open_time, o, h, l, c, v, is_closed

Pivot
  instrument_id, timeframe, bar_time, price,
  kind          : HIGH | LOW
  confirmed_at  : bar_time of the bar that confirmed it   -- never null when surfaced
  depth         : bars of confirmation on each side

PatternDefinition                 -- data, not code; loaded from §3
  code          : GARTLEY | BAT | BUTTERFLY | CRAB | SHARK
  vertex_count  : 4 | 5
  vertex_labels : ["X","A","B","C","D"] | ["O","X","A","B","C"]
  constraints   : [RatioConstraint]
  structural_rules : [rule_id]
  invalidation_ratio, stop_ratio, stop_reference_leg

RatioConstraint
  name          : "B_of_XA" | "BC_projection" | ...
  numerator_leg, denominator_leg
  min_ratio, max_ratio, ideal_ratio
  is_defining   : bool           -- drives ranking weight
  is_prz_contributor : bool      -- participates in §6.3 convergence

PatternCandidate
  definition_code, instrument_id, timeframe
  vertices      : ordered [ {label, pivot_id, price, bar_time} ]   -- 4 or 5
  orientation   : BULLISH | BEARISH        -- derived, §2.3
  measurements  : { constraint_name -> {actual_ratio, deviation_pct, passed} }
  structural_results : { rule_id -> passed }
  status        : FORMING | PRZ_ACTIVE | CONFIRMED | INVALIDATED | EXPIRED
  quality_score : 0..100
  detected_at, confirmed_at, source : SCANNER | MANUAL

PRZ                                -- an interval, never a single price
  candidate_id
  low, high                        -- convergence envelope
  contributions : [ {constraint_name, projected_price, weight} ]
  convergence_score : 0..100       -- tightness, §6.3
  entered_at, first_touch_price

RiskPlan
  candidate_id
  entry_price, entry_mode : LIMIT_AT_PRZ | ON_CONFIRMATION
  invalidation_price, stop_price, stop_buffer_price
  tp1_price, tp2_price               -- retracements of AD, §7.3
  rr_to_tp1, rr_to_tp2
  position_size, risk_amount, risk_pct_of_equity
  passes_rr_floor : bool

ConfirmationCheck
  candidate_id, kind : RSI_DIVERGENCE | ENGULFING | ...
  satisfied, evaluated_at, detail
```

### 4.2 Polymorphic vertices

`PatternCandidate.vertices` is an **ordered list with labels**, not fixed `x/a/b/c/d` columns.
Ratio constraints reference legs *by label pair* (`("X","A")`), so the same evaluator handles the
four-point Gartley and the five-point Shark with no branching. Adding Cypher or 5-0 is a data
change, not a code change.

### 4.3 Asset-class adapter

One engine, four markets, behind a single interface:

```
MarketAdapter
  fetch_bars(instrument, timeframe, range) -> [Bar]
  subscribe(instrument, timeframe)         -> stream[Bar]
  session_state(timestamp)                 -> OPEN | CLOSED | HALTED | PRE | POST
  is_gap(prev_bar, bar)                    -> bool
  value_per_price_unit(instrument, size)   -> quote_currency   -- pip/tick/contract math
  normalize_history(bars)                  -> [Bar]            -- splits, dividends
```

| Adapter | Session model | Specific concerns |
|---|---|---|
| `CryptoAdapter` | 24/7, no gaps | Fractional sizing; venue-specific tick sizes; funding rates ignored in v1 |
| `ForexAdapter` | 24/5, weekend gap | Pip value per pair and lot; rollover; Sunday-open gaps that skip a PRZ entirely |
| `MetalsAdapter` | Like FX operationally | Contract size differs (XAG 5,000oz); otherwise reuses `ForexAdapter` logic |
| `EquityAdapter` | Market hours, halts | **Split/dividend adjustment must be applied before pivot detection** — an unadjusted split creates a phantom 50% "leg" and a fictitious pattern |

**Requirement (gap handling).** When a gap spans a PRZ, the setup must be marked `GAPPED_THROUGH`
and excluded from actionable status. Price never traded in the zone; a limit order would not have
filled and backtests that assume a fill are inflated.

---

## 5. Detection engine

### 5.1 Pipeline

```
Bars ──▶ Pivot detection ──▶ Candidate enumeration ──▶ Ratio evaluation
                                                            │
        Ranked setups ◀── Risk plan ◀── PRZ convergence ◀────┘
```

### 5.2 Pivot detection and the repainting requirement

**This is where harmonic scanners fail.** A ZigZag using an unconfirmed pivot will surface a
pattern that vanishes when the next bar prints. Users lose trust immediately, and any backtest
built on it is fiction — it "knew" the pivot before it existed.

**Requirements:**

- **D1** — A pivot is emitted only after `depth` bars have closed on *both* sides. A high at bar
  `t` with `depth = 5` is not available before bar `t+5` closes.
- **D2** — `Pivot.confirmed_at` is stored and is always ≥ `bar_time + depth`. Backtests must use
  `confirmed_at`, never `bar_time`, when deciding what was knowable.
- **D3** — A surfaced candidate never silently disappears. It transitions to `INVALIDATED` or
  `EXPIRED`, with a reason, and stays in history.
- **D4** — Pivot depth is configurable per timeframe and forms part of a scan's saved
  configuration, since results are not comparable across depths.
- **D5** — The current forming bar is excluded from pivot detection entirely.

**Acceptance test.** Replay a dataset bar by bar. For every candidate surfaced at bar `t`, assert it
still exists (in some status) at every bar after `t`. Any disappearance fails the build.

### 5.3 Candidate enumeration

For each instrument × timeframe, take the last `N` confirmed pivots (default 40) and enumerate
alternating 4- and 5-vertex sequences. Prune early and cheaply: apply structural rules S1–S3 and
`min_leg_bars` before computing any ratio.

Complexity is bounded by alternation — pivots strictly alternate HIGH/LOW, so sequences are drawn
from two interleaved chains, not all C(N,5) combinations. **Requirement:** enumeration for one
instrument × timeframe completes in < 50 ms at N = 40.

### 5.4 Ratio evaluation

For each candidate × pattern definition, evaluate every `RatioConstraint` and record the actual
ratio, deviation, and pass/fail — **including for failures**. The UI must be able to show "this is
not a Bat because B is at 62.5%, outside 38.2–50.0%". Silent rejection is a specified defect: the
teaching value is in the rejection.

A candidate is `valid` when all structural rules and all constraints pass within tolerance.

### 5.5 Quality score

```
quality = 40 × defining_ratio_fit      -- deviation of is_defining constraints from ideal
        + 25 × convergence_score       -- §6.3
        + 15 × non_defining_fit
        + 10 × leg_symmetry            -- time symmetry of AB vs CD
        + 10 × htf_alignment           -- agreement with the higher timeframe
```

The weights are configuration, not constants, and the UI shows the component breakdown. **A quality
score is never a substitute for the pass/fail gate** — a 95-scoring candidate that fails a
structural rule is still rejected.

---

## 6. PRZ computation

### 6.1 A PRZ is an interval

The Potential Reversal Zone is the price band where **several independent projections converge**.
It is not the D ratio alone. Contributors:

1. The **XA-leg ratio** for D (e.g. Bat 0.886).
2. The **BC projection** (e.g. Bat 1.618–2.618).
3. The **AB=CD completion** (e.g. Bat 1.270–1.618 extended).

Each yields a price or price range. Because 2 and 3 are ranges, each is evaluated at both bounds
and the ideal.

### 6.2 Envelope

```
prz_low  = min(all contributor prices) − tolerance_price
prz_high = max(all contributor prices) + tolerance_price
```

**Requirement:** the UI draws each contributor as a distinct line inside the shaded band, labelled
with its ratio, so the trader sees *why* the zone is where it is.

### 6.3 Convergence score

```
spread = (max_contributor − min_contributor) / |A − X|
convergence_score = clamp(100 × (1 − spread / max_acceptable_spread), 0, 100)
```

with `max_acceptable_spread` defaulting to 0.10 (10% of the XA leg).

A tight cluster (all contributors within 2% of XA) scores near 100 and is the classic high-quality
PRZ. A spread beyond `max_acceptable_spread` scores 0 and the candidate is flagged
**`PRZ_DISPERSED`** — the ratios do not agree on a zone, so there is no coherent PRZ to trade.

### 6.4 Status transitions

| From | To | Trigger |
|---|---|---|
| `FORMING` | `PRZ_ACTIVE` | Price trades within `[prz_low, prz_high]` |
| `PRZ_ACTIVE` | `CONFIRMED` | A confirmation check (§7.4) is satisfied while in the zone |
| any | `INVALIDATED` | Price closes beyond the invalidation ratio, or a structural rule breaks |
| `FORMING` | `EXPIRED` | `max_bars_to_complete` elapse without reaching the PRZ (default 3× the CD leg's expected duration) |
| `PRZ_ACTIVE` | `GAPPED_THROUGH` | A gap spanned the zone without trading in it (§4.3) |

**Requirement:** invalidation is evaluated on **bar close**, not on wick touch, and the choice is
per-style configurable (§8.2). Wick-based invalidation on low timeframes stops out on noise; the
default is close-based.

---

## 7. Risk and execution rules

### 7.1 Stop placement — per pattern, never per style

The source brief placed the day-trading stop at "1.130 XA extension" for all patterns. **This is
unfillable for the Butterfly and the Crab**, whose D points sit at 1.270 and 1.618: price crosses
1.130 *on the way to* D, so the stop triggers before the entry fills. The rule is coherent only
for the Gartley (0.786) and Bat (0.886).

**Requirement:** the stop ratio is a property of the **pattern** (§3.2/§3.3). Trading style adjusts
only the buffer.

```
stop_price = P(A, X, stop_ratio) + direction × style_buffer × |A − X|
```

**Hard invariant (must be enforced in code):** `stop_price` lies strictly outside
`[prz_low, prz_high]`, on the far side from entry. A setup violating this is never actionable —
it is a specification error, not a configuration choice.

### 7.2 Trading-style tiers

| Style | Timeframes | Ratio tolerance | Style buffer | Invalidation basis |
|---|---|---|---|---|
| **Scalping** | M15 – M30 | ±1.0% – ±2.0% of XA | 0.02 XA | Close |
| **Day trading** | H1 – H4 | ±2.0% – ±4.0% of XA | 0.05 XA | Close |
| **Swing** | D1 – W1 | ±3.0% – ±5.0% of XA | 0.08 XA | Close, plus structural S/R override |

**On the M15 floor.** The source brief specified scalping from M1. On M1–M5 the spread plus
realistic slippage frequently exceeds the entire PRZ width, so the measured edge is consumed by
transaction costs before any pattern logic applies. The scanner therefore does not offer M1–M5 in
v1. **Requirement:** the setup panel displays estimated spread + slippage as a percentage of the
stop distance, and warns above 10%.

### 7.3 Targets

Targets are **retracements of the AD leg**:

```
TP1 = P(D, A, 0.382)      -- move stop to break-even on fill
TP2 = P(D, A, 0.618)
```

**Not the CD leg.** On a Crab the CD leg is enormous, and 38.2% of it can overshoot point C
entirely, producing a target further away than the structure that generated it. The AD leg is the
canonical reference and is well-behaved across all four patterns.

The **Shark is the exception** and uses its own rule: **TP = 0.500 retracement of BC** (§3.3).

### 7.4 The confirmation gate

**Rule:** never rest a blind limit order at D. The PRZ is a zone of *potential* reversal; price
reaches it and continues often enough that unconfirmed entry is a materially different — and worse
— strategy.

Entry requires at least one confirmation while price is inside the PRZ:

| Check | Definition |
|---|---|
| **RSI divergence** | Price makes an extreme beyond the prior swing; RSI(14) does not confirm |
| **Engulfing candle** | A close-confirmed engulfing bar in the trade direction, formed inside the zone |
| **Structure shift** | A confirmed lower-high (bearish) or higher-low (bullish) on the timeframe below |
| **Volume climax** | Volume ≥ 2× the 20-period average on the touch bar — *crypto and equities only; FX spot volume is venue-local and not comparable* |

`confirmations_required` defaults to 1 and is configurable to 2 for a stricter profile.

### 7.5 Position sizing

```
risk_amount   = account_equity × risk_pct           -- default 1.0%
stop_distance = |entry_price − stop_price|
position_size = risk_amount / (stop_distance × value_per_price_unit)
```

`value_per_price_unit` comes from the asset-class adapter (§4.3): pip value for FX, contract size
for metals, tick value for futures-style instruments, unit price for crypto and equities.

### 7.6 Gates — a setup is actionable only if all pass

| Gate | Threshold |
|---|---|
| All structural rules pass | mandatory |
| All ratio constraints within tolerance | mandatory |
| `convergence_score` > 0 (not `PRZ_DISPERSED`) | mandatory |
| Stop lies outside the PRZ | mandatory invariant (§7.1) |
| **R:R to TP2 ≥ 2.0** | mandatory |
| Confirmation satisfied | mandatory before entry |
| Portfolio gates (§7.7) | mandatory |

**R:R is measured to TP2.** TP1 frequently lands below 2:1 — the corrected Case Study 1 gives 1.39
to TP1 and 2.26 to TP2 — so TP1 functions as a partial-exit and break-even trigger, not as the
trade's justification.

### 7.7 Portfolio-level limits

Absent from the source brief, and the most common way a per-trade-disciplined trader still
suffers an outsized loss.

| Limit | Default | Rationale |
|---|---|---|
| Max concurrent open risk | 4% of equity | Caps aggregate exposure |
| Max risk per correlated cluster | 2% of equity | **BTC, XAG and FX majors all load on the dollar.** Three "independent" 1% shorts against USD is one 3% position |
| Max concurrent setups per instrument | 1 | Prevents stacking Gartley + Bat on the same D |
| Daily loss limit | 3% of equity | Halts new setups until the next session |
| Max open setups | 6 | Attention limit |

**Requirement:** correlation clusters are configurable, with sensible defaults (USD-denominated
metals, USD FX majors, large-cap crypto, index-correlated equities). The dashboard shows current
portfolio heat against each limit, and blocks a setup that would breach one, stating which.

---

## 8. Manual plotting mode

### 8.1 Behaviour

The trader places 4 or 5 points by clicking or dragging on the chart. On every drag frame the
engine re-runs the **identical** evaluation path used by the scanner — not a parallel
implementation — and updates live:

- Each ratio, with actual value, target window, and pass/fail.
- The PRZ envelope with its individual contributor lines.
- Stop, TP1, TP2, R:R, position size.
- Which pattern definitions the current geometry satisfies — a structure can qualify as more than
  one, and the tool shows all matches ranked.

### 8.2 Requirements

- **M1** — Manual mode uses the same code path as the scanner. Divergence between the two is a
  defect. This is also the scanner's test harness: plot a known-good pattern manually, confirm the
  scanner finds the same one with the same numbers.
- **M2** — Points snap to nearby confirmed pivots by default, with a modifier to place freely.
  Freely-placed points are flagged, since they may not correspond to confirmable structure.
- **M3** — Rejections are explained in plain language: *"Not a Bat: B is at 62.5% of XA, outside
  the 38.2–50.0% window. At 50.0%, B would be 64,910."* — including the corrective price.
- **M4** — Manual setups are saved with `source = MANUAL` and are separable in analytics, since
  their hit-rate is not comparable to scanner output.
- **M5** — A manual setup can be plotted on historical data and evaluated with a "what was knowable
  at bar *t*" toggle, which enforces `Pivot.confirmed_at` (§5.2 D2).

---

## 9. UI specification

### 9.1 Layout

```
┌────────────────────────────────────────────────────────────────────┐
│  HEADER: instrument · timeframe · system status · tolerance profile │
├──────────────────────────────────────────┬─────────────────────────┤
│                                          │  RISK & EXECUTION       │
│  CHART + PATTERN OVERLAY                 │  entry / stop / TP1/TP2 │
│  (XABCD polygon, PRZ band,               │  R:R · size · heat      │
│   contributor lines, stop & target)      │  confirmation checklist │
│                                          ├─────────────────────────┤
├──────────────────────────────────────────┤  ACTIVE SETUPS          │
│  PRZ MATRIX / RATIO PANEL                │  ranked, with status    │
│  per-constraint actual vs window         │                         │
├──────────────────────────────────────────┴─────────────────────────┤
│  PATTERN REFERENCE TABLE (§3.2 / §3.3, read-only)                   │
└────────────────────────────────────────────────────────────────────┘
```

### 9.2 System status states

The brief specified only "Active PRZ Detected". The full set, each with a distinct visual treatment:

| State | Meaning |
|---|---|
| `SCANNING` | Engine running, no valid candidates |
| `PATTERN_FORMING` | Valid candidate, price not yet in the PRZ |
| `PRZ_ACTIVE` | Price inside a PRZ, awaiting confirmation |
| `CONFIRMED` | Confirmation satisfied, all gates passed — actionable |
| `BLOCKED` | Valid and confirmed, but a portfolio gate blocks it (states which) |
| `INVALIDATED` | Invalidation breached |
| `PRZ_DISPERSED` | Ratios do not converge (§6.3) |
| `GAPPED_THROUGH` | Price gapped past the zone |
| `DATA_STALE` | Feed disconnected or lagging beyond threshold |

**`DATA_STALE` is a safety requirement,** not a nicety: a dashboard that silently shows a frozen
price is worse than one that shows nothing.

### 9.3 Visual design

Base palette from the source brief:

| Token | Value | Use |
|---|---|---|
| `--bg` | `#0D1117` | Background |
| `--vector` | `#00D2FF` | Pattern legs |
| `--long` | `#00FF66` | Long PRZ |
| `--short` | `#FF3366` | Short PRZ, invalidation |

**Accessibility requirements — these modify the brief:**

- **V1** — Direction must never be encoded by hue alone. Deuteranopia affects roughly 6% of men
  and would render `--long` and `--short` near-identical. Every long/short indication carries a
  **text label and a distinct shape** (▲/▼) in addition to colour.
- **V2** — Ship a **colorblind-safe alternate palette** (blue `#3B82F6` / orange `#F59E0B`)
  toggleable in settings.
- **V3** — Saturated neon on near-black haloes badly at small sizes. Body text and table numerals
  use a desaturated foreground (`#C9D1D9`); full-saturation neon is reserved for chart vectors and
  large-format figures.
- **V4** — All text meets **WCAG AA (4.5:1)** against its background. `#00FF66` on `#0D1117` passes
  at large sizes but must not be used for small numerals.
- **V5** — Every numeric cell shows full precision on hover; the display is rounded to the
  instrument's `price_precision`.

### 9.4 Alerts

Configurable per setup and globally: PRZ entry, confirmation satisfied, invalidation breached,
target reached, portfolio-limit breach. Delivery via in-app, browser push, and webhook. **Every
alert names the instrument, pattern, timeframe and price**, so it is actionable without opening
the dashboard.

---

## 10. Worked case studies

Both case studies in the source brief are arithmetically invalid. Full audit in Appendix A. The
corrected and replacement examples below are the ones the product ships as reference content; every
figure has been recomputed and verified.

### 10.1 Corrected — XAG/USD Bearish Butterfly

The source structure and direction were **correct**; the arithmetic was not. Corrected values:

| Point | Price | Ratio | Check |
|---|---|---|---|
| X | 63.80 | — | Swing high |
| A | 55.27 | XA leg = 8.53 | Swing low |
| B | **61.97** | 78.6% of XA | Was stated 61.10 = 68.3% — **corrected** |
| C | 56.68 | 75.8% of AB | Within 38.2–88.6% ✓ |
| D | **66.10** | 1.270 XA | Was stated 64.88–65.21 = 1.127–1.165 XA — the **1.13 band, not 1.270** — **corrected** |

**Risk plan.** Short 66.10 · invalidation above 69.07 (1.618 XA) · stop 69.07 + style buffer.
TP1 61.96 (38.2% of AD) · TP2 59.41 (61.8% of AD).

**The geometry is correct — and the dashboard still blocks it.** R:R depends on the buffer:

| Stop basis | Stop | Risk | R:R to TP1 | R:R to TP2 | Gate |
|---|---|---|---|---|---|
| Bare invalidation (no buffer) | 69.07 | 2.97 | 1.39 | **2.26** | passes |
| Day-trading buffer (0.05 XA) | 69.50 | 3.39 | 1.22 | **1.97** | **blocked** |
| Swing buffer (0.08 XA) | 69.75 | 3.65 | 1.13 | **1.83** | **blocked** |

At any realistic buffer the setup falls below the 2:1 floor (§7.6) and is marked `BLOCKED`. This is
the intended behaviour and the reason it ships as reference content: a pattern can be *perfectly
valid and still not worth trading*. A Butterfly entered at the near edge of a wide 1.270–1.618 PRZ
carries a stop that is structurally large relative to its AD-leg targets. The tool's job is to
surface that before capital is committed, not after.

Note also that the source brief's stop at 66.70 sits at 1.340 XA — *inside* the 1.270–1.618 PRZ —
and would have been hit during entry regardless of where price went next.

### 10.2 Replacement — BTC/USDT Bullish Bat

The source's Case Study 2 failed in four independent ways (Appendix A.7) and is replaced rather
than patched.

| Point | Price | Ratio | Check |
|---|---|---|---|
| X | 53,500 | — | Swing low |
| A | 71,980 | XA leg = 18,480 | Swing high |
| B | 62,740 | 50.0% of XA | Bat window 38.2–50.0 ✓ |
| C | 68,450 | 61.8% of AB | Window 38.2–88.6 ✓ · C < A ✓ |
| D | 55,607 | 88.6% of XA | D > X, never breaches ✓ |

**Convergence.** BC projection 2.249 (Bat window 1.618–2.618 ✓) · AB=CD 1.390 (window 1.270–1.618 ✓).
All three contributors agree → tight PRZ.

**Risk plan.** Long 55,607 · invalidation below 53,500 (point X) · stop 53,200 (risk 2,407).
TP1 61,861 (38.2% of AD) · TP2 65,725 (61.8% of AD).

**R:R 2.60 to TP1, 4.20 to TP2.** Passes all gates.

---

## 11. Phasing

The confirmed scope — four asset classes plus both detection modes — is wide for a v1. **This is
the primary delivery risk.** The phasing below sequences it so the engine is proven on the
cheapest data before market-specific complexity is added.

### Phase 1 — Engine + Crypto, on TradingView (foundation)

Crypto first because the feed is free and open, it runs 24/7 with no session or gap logic, and
sizing is fractional — the fewest confounds while the core engine is stabilised. Built as the
Pine Script v6 `strategy()` described in §1.6, run against TradingView's own crypto data.

- Pattern definitions §3 as data (a Pine `PatternDef` table); ratio evaluator; structural rules
- Pivot detection via confirmed `ta.pivothigh`/`ta.pivotlow` — the anti-repainting guarantee is
  structural, not bolted on (§5.2, §1.6)
- PRZ convergence (§6), risk plan (§7.1–7.6), all gates — implemented exactly as specified
- Gartley, Bat, Butterfly, Crab, on one symbol/timeframe per chart
- On-chart polygon, PRZ box, stop/TP lines; `alertcondition()` for PRZ entry and confirmed entry
- **Scanner-only.** Manual plotting (§8) and the cross-symbol watchlist scanner (§5.3) are the
  parts of this phase's original ambition that Pine cannot provide (§1.6) — deferred, not silently
  dropped, to a browser-hosted build in a later phase

**Exit criteria:** the §11.5 historical validation protocol passes on Bar Replay (no repainting)
and reproduces both §10 case studies within one tick; the Strategy Tester's trade list shows zero
trades with a stop inside their own PRZ and zero trades below the configured R:R floor.

### 11.5 Historical validation protocol (Pine Script v1)

Concrete, repeatable steps for validating `harmonic_prz_scanner.pine` (Appendix B) against real
history on TradingView, rather than trusting the script on faith.

1. **Non-repainting check.** Apply the script to a liquid chart (e.g. BTCUSDT, H1). Open **Bar
   Replay**, step forward through a period where the script draws a pattern polygon and PRZ box.
   **Pass condition:** once a polygon is drawn, it never disappears or moves on a later bar — it
   only ever transitions to `INVALIDATED`/`EXPIRED` styling (§6.4). This directly exercises D3.
2. **Case-study reproduction.** Load XAG/USD (or the closest available silver CFD/futures symbol)
   on a daily chart spanning the corrected Case Study 1 dates (§10.1). Confirm the script's plotted
   X/A/B/C prices and PRZ band match §10.1's corrected figures (B ≈ 61.97, D-zone around 66.10)
   within one tick — not the source brief's incorrect 61.10 / 64.88–65.21. Repeat for the BTC/USDT
   replacement (§10.2).
3. **Strategy Tester read.** With the script running as a `strategy()`, open TradingView's
   **Strategy Tester** panel → **Performance Summary** for net profit, win rate, and profit factor;
   **List of Trades** for entry/exit prices per trade, cross-checked against the on-chart PRZ/stop/
   TP levels for a handful of trades by hand.
   **This is decision-support-only historical validation, not a claim of a tradeable edge:** default
   inputs are unoptimized, no commission/slippage model is configured out of the box (add both under
   Strategy Properties before drawing any conclusion — §7.2's spread/slippage warning applies here
   too), and sample sizes on a single symbol/timeframe will usually fall short of §12's 100-trade
   research bar. Its purpose is to prove the **arithmetic and gating logic behave as specified**
   (G1–G4), not to certify profitability.
4. **Gate verification.** Confirm in the trade list that no trade shows a stop price inside the PRZ
   band it entered from (§7.1 hard invariant), and that every trade's realized TP2 distance divided
   by its risk is ≥ `rrMin` (default 2.0, §7.6) — a trade violating either is a script defect, not a
   market outcome.

### Phase 2 — Forex + Metals

Operationally similar to each other, so they ship together.

- `ForexAdapter` and `MetalsAdapter`: session state, weekend gaps, pip/contract value
- Gap-through detection and the `GAPPED_THROUGH` status
- **Shark** — the five-point path, exercising the polymorphic vertex model
- Portfolio correlation clusters (§7.7), which first become meaningful with USD-denominated pairs
- Session/killzone context in the setup panel

### Phase 3 — Equities & Indices + research

- `EquityAdapter`: market hours, halts, pre/post
- **Split and dividend adjustment before pivot detection** — the sharpest correctness risk in this
  phase; an unadjusted 2:1 split fabricates a 50% leg and a phantom pattern
- Backtest module: per-pattern, per-timeframe, per-asset-class hit rates on the user's own data,
  using `Pivot.confirmed_at` for knowability
- Deferred patterns from §3.4

---

## 12. Success metrics

**Correctness (release-blocking).**

| Metric | Target |
|---|---|
| Repainting incidents in the replay test | **0** |
| Scanner vs manual numerical agreement | **100%** |
| Setups reaching actionable with a stop inside their PRZ | **0** (invariant §7.1) |
| Reference case studies reproducing exactly | **100%** |

**Engineering.**

| Metric | Target |
|---|---|
| Enumeration, one instrument × timeframe, N=40 | < 50 ms |
| Full watchlist scan (50 instruments × 4 timeframes) | < 5 s |
| Manual-drag recompute | < 16 ms (60 fps) |
| Alert latency from bar close | < 2 s |

**Product.**

| Metric | Target |
|---|---|
| Setups where the user inspected the ratio panel before acting | > 60% |
| Manual-mode sessions per active user per week | > 3 (indicates the validation is trusted) |
| Setups blocked by portfolio gates that the user overrode | < 10% (higher suggests mis-tuned limits) |

**Explicitly not a metric:** pattern win rate as a product KPI. The tool's job is correct
arithmetic and enforced discipline; market outcomes are not within its control, and optimising a
displayed win-rate would create pressure to loosen the gates.

---

## 13. Disclaimer & regulatory posture

- The product is **decision-support only**. It holds no broker credentials, has no market access,
  and cannot place, modify, or cancel an order. The order ticket is a formatted summary for the
  user to act on in their own platform.
- **Not financial advice, and not a personal recommendation.** All output is deterministic
  arithmetic on user-selected inputs.
- A **risk disclosure** appears at onboarding and remains accessible; the trading surface carries a
  persistent non-advice notice.
- **No performance claims.** Historical statistics, when shown (P3), are labelled with their
  sample size, date range, instrument set, and the knowability constraint applied.
- Market data is redistributed strictly per each vendor's licence terms. **Equity data licensing is
  a P3 gating dependency** and must be resolved before that phase begins.
- Users are not permitted to resell or redistribute generated signals; this is an end-user tool.

---

## 14. Open questions

| # | Question | Owner | Blocks |
|---|---|---|---|
| 1 | Which crypto venue is the P1 reference feed — does it need to match the user's execution venue for prices to be actionable? | Product | P1 |
| 2 | Should tolerance be **adaptive to volatility** (ATR-scaled) rather than a fixed XA fraction? Defensible, but harder to reason about and to reproduce | Research | P2 |
| 3 | FX/metals data vendor, given redistribution terms | Product/Legal | P2 |
| 4 | Is `confirmations_required = 1` the right default, or should extension patterns (Butterfly, Crab) require 2 given their wider stops? | Research | P1 |
| 5 | Should HTF alignment be a scoring input (as specified in §5.5) or a hard gate? | Product | P1 |
| 6 | Local-only persistence vs. account sync — affects whether any user positional data is stored server-side | Eng/Legal | P1 |
| 7 | Equity data licensing cost and terms | Legal | P3 |

---

## Appendix A — Audit of the source brief

Full arithmetic audit of the design brief that seeded this PRD. Every figure below was recomputed
from the stated points. Conducted under the senior-trading-auditor standard.

### A.1 What was correct

The following were verified and carried through unchanged: Gartley B = 61.8% / D = 78.6%;
Bat B = 38.2–50% / D = 88.6%; Butterfly B = 78.6% / D = 1.270–1.618; Crab B = 38.2–61.8% / D = 1.618;
the 38.2–88.6% AB retracement for C across all four; Gartley and Bat invalidation beyond X; the
confirmation-before-entry rule; and the trading-style tolerance *concept*. The core of the brief is
sound — the defects are specific and fixable.

### A.2 Omission — BC projections absent entirely

The brief's table has no BC projection column. Since the PRZ is by definition the convergence of the
XA ratio with the BC projection, a table without it does not specify a PRZ. Added in §3.2:
Gartley 1.130–1.618, Bat 1.618–2.618, Butterfly 1.618–2.240, Crab 2.240–3.618. The Crab's is its
defining constraint and its absence makes Crab and Butterfly indistinguishable near 1.618.

### A.3 Omission — AB=CD completion

Not mentioned; it is a required leg for the Gartley and the Bat. Added in §3.2.

### A.4 Omission — structural validity rules

Unstated, notably **C must not exceed A**. Case Study 2 violates exactly this. Added as S1–S7 in §3.1.

### A.5 Logic bug — zero-width stops

The brief sets Butterfly D at 1.270–1.618 with invalidation "> 1.618", and Crab D at 1.618 with
invalidation "> 1.618" — then contradicts itself parenthetically with "(Breaches 2.00 XA)". Entering
at the far edge of a PRZ whose stop sits at that same edge gives a stop distance of zero, which
cannot be filled. Corrected in §3.2 to Butterfly 1.750 and Crab 1.902, verified to leave gaps of
+0.132 and +0.284 XA respectively.

### A.6 Logic bug — the 1.130 XA day-trading stop

The brief places the day-trading stop at "1.130 XA extension" for all patterns. For the Butterfly
(D = 1.270) and Crab (D = 1.618), price crosses 1.130 *before reaching D*, so the stop fires before
the entry fills. Coherent only for the Gartley and Bat. Corrected in §7.1: the stop ratio belongs to
the pattern; style adjusts only the buffer.

### A.7 The Shark row is invalid

Stated as "B = 1.130–1.618 AB | C = 1.130–1.618 XC | D = 88.6%–1.130 XA". `C = 1.130–1.618 XC`
defines C in terms of itself — circular and unimplementable. The Shark is also a **five-point
O-X-A-B-C** pattern and does not fit the brief's four-point column schema, and its terminal point is
a retracement of **OX**, not XA. Corrected in §3.3; this drove the polymorphic vertex model in §4.2.

### A.8 Tolerance unit undefined

"±1.0%" never states percent of what. On the brief's own Case Study 2 numbers, percent-of-price gives
±$680 while percent-of-XA-leg gives ±$47 — a 14× difference that determines whether the filter means
anything. Fixed in §2.4 as a fraction of the XA leg length. Separately, the M1 scalping tier is not
viable: spread plus slippage routinely exceeds the whole PRZ width. Floor raised to M15 (§7.2).

### A.9 Targets measured on the wrong leg

The brief sets TP1/TP2 at 38.2%/61.8% of **CD**. The canonical reference is the **AD** leg. On a Crab,
where CD is very large, 38.2% of CD can overshoot point C entirely and yield a target beyond the
structure that produced it. Corrected in §7.3.

### A.10 Case Study 1 — XAG/USD Bearish Butterfly: 3 of 4 values wrong

Structure and direction correct. XA leg = 8.53.

| Field | Stated | Recomputed | Verdict |
|---|---|---|---|
| B | 61.10 "= 78.6%" | **68.3%** of XA; 78.6% = **61.97** | ✗ |
| C | 56.68 | 75.8% of AB — inside 38.2–88.6% | ✓ |
| D (PRZ) | 64.88–65.21 "= 1.270" | **1.127–1.165 XA** — the 1.13 band. True 1.270 = **66.10**; 1.618 = 69.07 | ✗ |
| SL | 66.70 | **1.340 XA — inside the 1.270–1.618 PRZ** | ✗ fatal |

The stop error is the fatal one: it sits between the near and far edges of the entry zone, so the
trade is stopped out during entry regardless of what price subsequently does. Corrected setup in §10.1.

### A.11 Case Study 2 — BTC/USDT "Bullish Bat": four independent errors

XA leg = 4.72k. Unsalvageable; replaced in §10.2.

1. **Direction inverted.** X = 67.27k is the high and A = 62.55k the low, so D at 88.6% = **66.73k, a
   lower high → SHORT**. The brief labels it "Long Execution". This is the §2.1 anchoring error
   producing a trade in the wrong direction.
2. **Stated D breaches its own invalidation.** D = 68.33k = **1.225 XA** — past X, which the brief's
   own table defines as a Bat hard invalidation. The entry is placed at the stop-out level.
3. **B outside the Bat window.** 65.50k = **62.5%** of XA; 50% would be 64.91k. At 62.5% the structure
   is not a Bat at all.
4. **C is degenerate.** C = 62.55k = exactly A → a **100% AB retracement**, violating both the
   38.2–88.6% window and structural rule S2.

### A.12 Gaps beyond the brief

No position sizing, no portfolio exposure or correlation limits (BTC, XAG and FX majors all load on
the dollar, so three "independent" 1% positions can be one 3% bet), no daily loss limit, no
gap-through handling, no data-staleness state, no disclaimer or regulatory posture, and no
accessibility treatment for a red/green-only direction encoding. All addressed in §7.7, §9.2, §9.3
and §13.

### A.13 Verdict

```
═══════════════════════════════════════════════════
SENIOR QUANT AUDIT — Harmonic PRZ Dashboard Spec
═══════════════════════════════════════════════════
VERDICT: CONDITIONAL APPROVE — spec is salvageable, worked examples are not
INSTITUTIONAL GRADE: C  (B+ after the §3/§7 corrections)

CONFIRMED STRENGTHS
1. Core XA and AB ratios correct for all four 4-point patterns
2. Confirmation-before-entry rule is correct and non-negotiable — kept as §7.4
3. Trading-style tolerance tiering is the right concept
4. Case Study 1's structure and direction were correctly identified

CRITICAL ISSUES (showstoppers)
1. Both worked case studies fail their own arithmetic (A.10, A.11)
2. Case Study 2 is directionally inverted — a real trade from it loses on entry
3. BC projection absent — no PRZ is actually being computed (A.2)
4. Butterfly/Crab stops are zero-width; the 1.130 XA rule is unfillable (A.5, A.6)
5. Shark definition is circular and unimplementable (A.7)
6. Tolerance unit undefined — 14x ambiguity (A.8)

REQUIRED CHANGES — all incorporated into this PRD
1. Adopt the §2.1 anchoring convention explicitly and test it
2. Add BC projection, AB=CD, and structural rules S1-S7
3. Move stop ratios to per-pattern with verified non-zero gaps
4. Re-specify Shark as 5-point O-X-A-B-C off the OX leg
5. Define tolerance as a fraction of the XA leg; raise the floor to M15
6. Move targets to the AD leg; enforce R:R >= 2:1 on TP2
7. Replace Case Study 2; correct Case Study 1

HARSH TRUTH
The method is fine. The execution of the arithmetic is not, and that is the whole
game in harmonics — a Bat with B at 62.5% is not a Bat, and no amount of dashboard
styling makes it one. Two of two worked examples failed. If the person writing the
reference spec gets it wrong twice out of twice, a trader working by hand at speed
will do worse. That is the actual case for building this product, and it is a good one.

— Marcus Chen, CFA FRM
   "The market doesn't care about your opinions, only your math."
═══════════════════════════════════════════════════
```
