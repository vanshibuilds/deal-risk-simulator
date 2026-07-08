# Methodology

Technical documentation for the **Deal Architecture & Risk Simulator** — the financial model, the risk simulation, the numerical methods, and the design decisions behind them.

For a quick overview and the live demo, see the [README](README.md).

---

## Contents

1. [Overview](#1-overview)
2. [The financial model](#2-the-financial-model)
3. [Unit economics and benchmarks](#3-unit-economics-and-benchmarks)
4. [Scenarios, sensitivity and break-evens](#4-scenarios-sensitivity-and-break-evens)
5. [Monte Carlo risk simulation](#5-monte-carlo-risk-simulation)
6. [Deal comparison](#6-deal-comparison)
7. [Architecture](#7-architecture)
8. [Assumptions and limitations](#8-assumptions-and-limitations)
9. [Roadmap](#9-roadmap)

---

## 1. Overview

The simulator is a general-purpose appraisal engine for recurring-revenue deals. Any deal that fits the shape *"acquire a cohort of paying units against a fixed commitment plus a revenue share"* — a streaming content licence, a B2B SaaS subscription line, a consumer app partnership — can be defined, priced, stressed and risk-simulated.

It is built to answer the four questions an investment committee asks about a deal:

| Question | Output |
|---|---|
| Is it worth it at these terms? | NPV, IRR, payback on incremental discounted cash flows |
| What is it most exposed to? | Tornado chart ranking every driver by NPV swing |
| When does it stop working? | Break-even volume, churn, and **maximum supportable fee** |
| What's the chance we lose money? | Correlated Monte Carlo distribution, percentiles, tail-loss probability |
| Is the acquisition efficient? | CAC, LTV, LTV:CAC and CAC payback vs sector benchmarks |
| Which deal should get the capital? | Head-to-head comparison of any two configurations |

All figures in the tool are illustrative.

---

## 2. The financial model

### 2.1 Cohort revenue build

The deal's benefit is modelled as an acquired cohort of units — subscribers, enterprise customers, or app users depending on the template — paying a monthly price and decaying at a monthly churn rate.

Gross adds phase in over a configurable launch ramp (1–6 months) with front-weighted phasing: for a ramp of *n* months, month *i* receives weight proportional to (*n − i*), so a three-month ramp phases roughly 50/33/17. This reflects launch behaviour — the acquisition spike is largest at launch and tails off as the marketing wave passes.

Each month the cohort evolves as:

```
cohort(m) = cohort(m-1) × (1 − churn) + adds(m)
revenue(m) = cohort(m) × monthly price
```

This is a deliberately simple single-geometric-decay cohort. Additional layers — reactivation, seasonality, in-term price changes — would each need evidence to calibrate, and an appraisal model should not carry precision it cannot justify.

**Only incremental economics are scored.** The base business is excluded entirely; the deal must stand on the cash flows it alone creates. Deals that look attractive only because they sit on top of a healthy base are exactly the ones that disappoint.

### 2.2 Contribution margin, not gross revenue

The cohort is scored at **contribution margin** (default 70%), not gross revenue. Payment processing, delivery and hosting, and the cohort's fair share of the wider content or platform cost base all consume revenue before it becomes deal value.

This is the single most consequential modelling decision. Scored on gross revenue, a representative content deal returns an IRR well above 100% with a sub-12-month payback — figures no real content deal produces. The model is not mechanically wrong there; it is economically wrong, because it treats every unit of revenue as pure margin. On a contribution basis the same deal is genuinely marginal, which is both more realistic and more useful for decision-making.

### 2.3 Cost structure

Three cost blocks sit against cohort contribution:

- **Fixed commitment** (minimum guarantee / upfront build cost) — contractually fixed, paid on a configurable schedule: a chosen percentage at signing, remainder split equally across annual milestones within the term.
- **Revenue share** — accrues monthly as a percentage of cohort revenue, paid on top of the fixed commitment (a hybrid structure common in content licensing).
- **Marketing and ops** — launch marketing phases over the same ramp as adds; running ops (delivery, localisation, support) is a flat monthly cost.

The delivery-cost-variance lever stresses only the *controllable* cost base (marketing and ops), because the fixed commitment cannot overrun — it is a contract. This bounds contractual cost risk and leaves execution cost risk open-ended, which is the correct way to frame the cost conversation.

### 2.4 NPV, IRR, payback

**NPV** discounts monthly net cash flow at the monthly equivalent of the annual hurdle rate:

```
monthly rate = (1 + annual rate)^(1/12) − 1
```

A geometric conversion, not annual ÷ 12, so compounding is exact.

**IRR** is solved by bisection — searching for the monthly rate at which NPV = 0, halving the bracketing interval 80 times, then annualising. Bisection is chosen over Newton–Raphson because it cannot diverge; in a tool that recalculates on every input change, unconditional stability beats speed. If cash flows never change sign, no IRR exists and the tool reports `n/a` rather than inventing one.

**Payback** is the first month cumulative *undiscounted* cash flow turns positive — reported undiscounted deliberately, since payback is a liquidity/exposure measure, not a value measure, and discounting it would double-count the time value already in NPV.

### 2.5 NPV bridge

The waterfall decomposes NPV into present-value components: cohort revenue → less the variable-cost haircut (revenue minus contribution) → less revenue share → less fixed commitment → less marketing and ops → deal NPV. Every component is discounted on the same monthly basis, so the bridge reconciles to NPV exactly.

---

## 3. Unit economics and benchmarks

Alongside the aggregate appraisal, the tool computes the per-unit view:

```
expected paying months (within term T) = (1 − (1 − churn)^T) / churn
unit monthly contribution              = price × (margin − revenue share)
LTV                                    = unit monthly contribution × expected paying months
CAC                                    = acquisition marketing / gross adds
```

LTV:CAC and CAC payback are traffic-lighted against illustrative sector ranges per template (streaming, B2B SaaS, consumer apps), as is the churn assumption itself — flagging when an input sits outside the plausible range for its sector.

A deliberate design point: **the fixed commitment is excluded from CAC**, because it is the cost of the asset, not of acquisition. This is why the panel can show a healthy LTV:CAC on a deal with marginal NPV — the acquisition works, but the fixed fee captures the surplus. The tool surfaces this tension explicitly rather than letting one metric masquerade as the other: strong unit economics say the *marketing* works; only NPV says the *deal* works.

Benchmark ranges are illustrative heuristics for demonstration; a production version would source them from market data.

## 4. Scenarios, sensitivity and break-evens

### 4.1 Relative scenarios

Downside / base / upside are defined *relative* to the currently loaded deal, not as absolute numbers:

| Scenario | Adds | Churn | Cost |
|---|---|---|---|
| Downside | ×0.65 | +1.2pp | +20% |
| Base | ×1.00 | — | — |
| Upside | ×1.30 | −0.7pp | −5% |

This keeps the stress logic portable across templates. The asymmetry (downside larger than upside) reflects the empirical shape of deal outcomes.

### 4.2 Tornado

The tornado flexes one driver at a time between an adverse and a favourable bound — adds ±25%, churn ±1pp, contribution margin ±5pp, price ±10%, fixed commitment ±15%, revenue share ±3pp, delivery cost +20/−10%, discount rate ±2pp — holding all else constant, then ranks drivers by NPV swing. Bounds are scaled to plausible forecast error *per variable* rather than a uniform percentage, because a 25% miss on volume is common while a 25% miss on the discount rate is not.

### 4.3 Break-evens

Three break-evens are solved by bisection on the full model:

- **Minimum volume** for NPV ≥ 0
- **Maximum churn** the deal can absorb
- **Maximum supportable fee** — the largest fixed commitment before NPV turns negative

Search bounds adapt to the loaded deal, so the solver works for both a small B2B deal and a mass-market app deal. The maximum supportable fee is the most commercially useful output: it converts an appraisal into a negotiation ceiling — reframing "should we do this deal?" as "what is the most we should ever agree to pay?"

---

## 5. Monte Carlo risk simulation

### 5.1 Why simulation

Three scenarios give three points with no probabilities attached. A committee approving a deal with conditions needs the probability of loss and the size of the tail. The simulation runs 4,000 iterations of the full model, each with randomly drawn values for the three most uncertain inputs, producing a distribution of NPV outcomes and from it: probability of positive NPV, the P5/P50/P95 percentiles, expected NPV, and the probability of a loss beyond a materiality threshold.

### 5.2 Distributions

| Variable | Distribution | Shape |
|---|---|---|
| Gross adds | Triangular, 55%–125% of plan, mode at plan | Skewed to under-delivery |
| Churn | Normal, σ ≈ 0.6pp, clamped | Roughly symmetric |
| Cost variance | Triangular, −10% to +35% | Skewed to overrun |

The skews on adds and cost encode the empirical asymmetry of launches — they miss low and run over more often than the reverse, so symmetric distributions would understate downside. Triangular distributions are used for the skewed variables because they express asymmetry with three intuitive parameters (worst / most likely / best) that a stakeholder can sanity-check.

### 5.3 Correlation

By default, draws are **correlated through a shared launch-quality factor**: a single standard normal draw represents execution quality, and adds shortfall, cost overrun (ρ = 0.6) and churn (ρ = 0.3) all load on it. Correlated uniforms are produced via the normal CDF and pushed through each variable's inverse-transform. The rationale is empirical: a weak launch tends to miss on volume, overrun on cost and churn faster *together*, and independent draws understate exactly the joint-failure outcomes a committee cares about.

The effect is measurable and behaves as theory predicts: correlation leaves the central estimate roughly unchanged but fattens the tails — in the representative content deal, P5 worsens by ~£1M and the probability of a >£5M loss rises by ~3 percentage points versus independent sampling. A toggle allows direct comparison against independent draws.

The correlation coefficients are judgement-based, not estimated from data — stated plainly in the limitations.

### 5.4 Implementation

Normal draws use the Box–Muller transform; triangular draws use inverse-transform sampling (accepting an externally supplied uniform, which is what makes the correlation structure possible); the normal CDF uses the Abramowitz–Stegun approximation. Each iteration re-runs the full deal model, so 4,000 iterations execute ~150k–250k monthly steps — well under 50ms in-browser, which is why the simulation reruns live on every input change with no "run" button. 4,000 iterations is the point at which reported percentiles stabilise to within a decimal place between runs; more adds computation without adding information at this precision.

---

## 6. Deal comparison

Any configuration can be pinned as "Deal A"; everything — template, terms, assumptions — can then be changed and compared head-to-head on NPV, IRR, payback, P(NPV > 0) and LTV:CAC, with signed deltas. This turns the appraiser into a simple capital-allocation tool: not just *"is this deal good?"* but *"which of these two deals should get the money?"* — which is the decision committees actually make.

## 7. Architecture

One self-contained HTML file: HTML for structure, CSS for the board-pack visual style, vanilla JavaScript for logic, and [Chart.js](https://www.chartjs.org/) (via CDN) as the only dependency. No build step, no framework, no server.

**One pure function.** The deal model is a single side-effect-free `runDeal(params)` returning cash flows, NPV components and payback. Every other feature — scenario table, tornado, break-even solvers, Monte Carlo — is just that function called with modified parameters. This is why a single hardcoded deal generalised into a template-driven engine in one refactor: the model never knew *what* it was pricing, only the parameters it was handed.

**Templates** are plain configuration objects (slider ranges, defaults, labels, metadata); switching template re-ranges the interface without touching the model. Recalculation is debounced at 60ms so dragging a slider doesn't fire hundreds of full simulations.

**Save/load** exports the complete deal state as a downloadable JSON file and re-imports via a file picker — no browser storage, so a saved deal is a portable file the user owns.

**Committee memo** has two paths: an optional LLM API call that receives *only the already-computed figures*, with a prompt that forbids inventing numbers, and a deterministic fallback that builds the same memo offline. The model computes; the language model only narrates — so the memo can never disagree with the numbers on screen. (The public demo uses the deterministic path, since a client-side page cannot safely hold an API key.)

---

## 8. Assumptions and limitations

Stated plainly, because a model's honesty about its own limits is part of its usefulness:

- **Judgement-based correlations.** Monte Carlo draws are correlated through a launch-quality factor, but the coefficients (ρ = 0.6, ρ = 0.3) are analyst judgement, not estimates from historical deal data.
- **Single cohort, geometric decay.** No reactivation, no seasonality, no multiple cohorts or product tiers.
- **Flat pricing over the term.** No escalators or discounting schedules.
- **No attribution haircut.** All "incremental" adds are treated as truly incremental; a rigorous version would discount for units that would have been acquired anyway.
- **Illustrative benchmarks.** ARPU, churn and margin defaults — and the sector benchmark ranges in the unit-economics panel — are plausible heuristics, not sourced from a specific market dataset.

---

## 9. Roadmap

In rough priority order:

1. Correlation coefficients estimated from historical deal outcomes rather than judgement
2. Attribution haircut on gross adds (true incrementality)
3. Multi-cohort support (a second content drop / product tier stacking)
4. Price escalation over the term
5. Two-way sensitivity grid (adds × churn heat map)
6. Portfolio mode — ranking many saved deal files under a capital constraint

---

*Built as a commercial-finance portfolio project. Model design, calibration and interpretation are the author's; implementation was AI-assisted.*
