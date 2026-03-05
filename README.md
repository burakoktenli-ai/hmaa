# HMAA — Hierarchical Mission Authority Architecture

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-2.4.2-green.svg)](https://github.com/burakoktenli-ai/hmaa/releases/tag/v2.4.2)
[![Node](https://img.shields.io/badge/node-%3E%3D16-brightgreen.svg)](https://nodejs.org/)
[![ORCID](https://img.shields.io/badge/ORCID-0009--0001--8573--1667-a6ce39.svg)](https://orcid.org/0009-0001-8573-1667)

**Author:** Burak Oktenli · Georgetown University, School of Continuing Studies — MPS Applied Intelligence  
**Live Demo:** https://burakoktenli-ai.github.io/hmaa  
**Technical Report (v2.4.2):** https://doi.org/10.5281/zenodo.18861653
---
## Architecture Overview

![HMAA Architecture](docs/hmaa_architecture)

## What is HMAA?

HMAA is a **deterministic authority-gating engine** for autonomous systems in contested or safety-critical environments. It computes a scalar authority value `A ∈ [0, 1]` from four real-time inputs and maps it to a **five-tier operational classification**.

**The core problem:** naive authority models grant full autonomy if the operator is sufficiently credentialed or the mission context is sufficiently legitimate. HMAA rejects this: even a perfectly credentialed operator cannot override degraded sensor trust (τ) or a hostile electronic environment (E). The engine enforces this through a multiplicative gate-and-damp structure where *every* factor must be simultaneously sufficient.

**Contributions:**
1. A closed-form, fully deterministic authority formula with a τ-dependent gate exponent (γ) that sharpens the credential requirement as sensor trust degrades
2. An Electronic Warfare (EW) hysteresis gate with a configurable dead-band that prevents authority oscillation at the threat boundary
3. A five-tier authority classifier with an independent RTB fail-safe sub-state (below the authority layer — does not require A > 0)
4. A reproducible headless CLI engine (`simulate.js`) for safety-case evidence generation with seeded PRNG and SHA-256-stable outputs
5. A companion technical assurance report (v2.4.2) with boundary test suite, GSN safety argument, FMEA, and evidence register

---

## Inputs

| Symbol | Name | Meaning |
|--------|------|---------|
| Q | Operator Quality | Credential / qualification score |
| C | Context Confidence | Legitimacy of the current mission context |
| E | Environmental Threat | Electronic Warfare (EW) / jamming level |
| τ | Sensor Trust | Reliability of the sensor chain (fusion health monitor) |

All inputs ∈ [0, 1]. They are **external to the Trusted Computing Base (TCB)** and treated as potentially adversarial — the engine does not verify their provenance.

---

## Model Specification (Normative)

This is the authoritative spec. `simulate.js` is the reference implementation.

### Step 1 — Electronic Warfare (EW) Hysteresis Gate

**EW** = Electronic Warfare. The EW boolean state is computed first and passed forward:

```
First step (no prior):  EW = (E ≥ 0.60)
EW was ON  (prior):     EW = (E ≥ 0.55)   // hold zone: stays ON until E drops below 0.55
EW was OFF (prior):     EW = (E ≥ 0.60)
```

The dead-band `[0.55, 0.60)` prevents authority chattering at the engagement boundary.

### Step 2 — Dynamic Weight Adjustment (active under EW)

Under EW, operator quality (Q) is weighted more heavily — the engine becomes more conservative toward remote context and more reliant on verified credentials:

```
sig  = 1 / (1 + exp(−20 · (E − 0.60)))   // soft sigmoid centred on Eon
fac  = sig · (EW ? 1 : 0)                 // fac = 0 when EW is off
wq   = min(1.0,  0.55 + 0.15 · fac)       // wq ∈ [0.55, 0.70]
wc   = 1 − wq                             // wc ∈ [0.30, 0.45]
```

### Step 3 — Authority Computation

```
// 1. Gate exponent: sharpens as sensor trust degrades
gam  = 0.50 + 1.50 · (1 − τ)             // gam ∈ [0.50, 2.00]

// 2. Multiplicative gate — zero if Q=0 OR C=0 (I3 enforcement)
gate = 0                                   if Q · C ≤ 0
gate = (Q · C)^gam                         otherwise
gate = 0                                   if non-finite

// 3. Exponential environmental damping
damp = exp(−2.5 · E)

// 4. Weighted linear base
base = wq · Q + wc · C

// 5. Raw authority
A_raw = base · gate · damp · τ

// 6. NaN guard + clamp (defence-in-depth)
A = 0                                      if A_raw is non-finite or NaN
A = max(0, min(1, A_raw))                  otherwise
```

### Full Parameter Table

| Param | Default | Role | Rationale |
|-------|---------|------|-----------|
| wq | 0.55 | Quality weight | Q weighted slightly above C; shifts toward 0.70 under EW |
| wc | 0.45 | Context weight | Complement of wq (wq + wc = 1.0 always) |
| kd | 2.5 | Damping constant | At E=0.60: damp≈0.22; ensures threat-level environment severely reduces A |
| gb | 0.50 | Gate base exponent | gam=0.50 at τ=1.0 (gentle, square-root-like gate) |
| gs | 1.50 | Gate sensor scale | gam=2.00 at τ=0.0 (harsh, quadratic gate) |
| Eon | 0.60 | EW engagement | EW engages at 60% threat; heuristic default |
| Eoff | 0.55 | EW disengagement | 5-point dead-band prevents chattering |
| es | 20 | Sigmoid sharpness | Fast EW transition at threshold |
| dw | 0.15 | Weight shift under EW | wq increases by up to 0.15 under EW |
| tF | 0.80 | T4 threshold | Full autonomy requires A ≥ 0.80 |
| tS | 0.55 | T3 threshold | Supervised: A ≥ 0.55 |
| tR | 0.30 | T2 threshold | Restricted: A ≥ 0.30 |
| tL | 0.10 | T1 threshold | Restricted+: A ≥ 0.10; below is lockout |

**Parameter rationale:** all values are **heuristic defaults** designed to satisfy three behavioral requirements: (1) a certified operator in benign conditions lands in T2–T3, not T0; (2) adversarial inputs (high Q/C with degraded τ and high E) produce 100% T0 lockout; (3) the EW dead-band eliminates chattering at typical sensor noise levels. They have not been fit to external operational data and should be calibrated per deployment.

---

## Authority Tiers

| Tier | Name | Range | Allowed Actions |
|------|------|-------|-----------------|
| T0 | LOCKOUT | A < 0.10 | None. Pre-programmed RTB reflex engages (independent of A — see note) |
| T1 | RESTRICTED+ | 0.10 ≤ A < 0.30 | Emergency commands only; multi-party token required |
| T2 | RESTRICTED | 0.30 ≤ A < 0.55 | Limited waypoint navigation; single-party token |
| T3 | SUPERVISED | 0.55 ≤ A < 0.80 | Full mission execution; human oversight notification required |
| T4 | FULL AUTONOMY | A ≥ 0.80 | Unrestricted autonomous operation |

**RTB note:** Return-to-Base is a hardcoded fail-safe reflex executed by a sub-state machine (makeT0Model) that operates *below* the authority layer. It does not require A > 0 and is not gated by the authority engine. This resolves the apparent contradiction: RTB is not autonomous action — it is a mandatory safety reflex.

**Design intent:** T3/T4 are intentionally hard to reach. A certified operator in a nominal environment (Q=0.90, C=0.85, E=0.15, τ=0.98) lands at T2 (A=0.5128), just below T3. This is conservative by policy: authority escalation requires improving sensor conditions (τ→1, E→0), not inflating credentials.

---

## Safety Properties (Tested by Randomized CI — Not Formally Proved)

> **Important:** the invariants below are *specified* and *tested* via randomized simulation (property-based testing). This is **not** formal verification — no theorem prover, model checker, or formal spec (TLA+, Coq, etc.) has been applied. A CI gate with 10,000 random trials provides statistical confidence, not mathematical proof.

| ID | Statement | Enforcement Mechanism | Test Result |
|----|-----------|----------------------|-------------|
| I1 | A ∈ [0, 1] | NaN guard (A=0 if non-finite) + clamp | 0 violations / 10,000 trials |
| I2 | τ = 0 ⟹ A = 0 | τ appears as final multiplicative factor in A_raw | Algebraically guaranteed; τ=0.10 → A=0.0882 |
| I3 | Q = 0 **OR** C = 0 ⟹ A = 0 | gate = 0 when Q·C ≤ 0; gate is multiplicative | Tested: Q=0,C=0.85→A=0 · Q=0.9,C=0→A=0 |

`node simulate.js --builtin ci_gate --seed 42` → exit 0 (all pass).

---

## Experimental Methodology

### CI Gate (Invariant Randomized Testing)
- **Distribution:** Q, C, E, τ each sampled independently from U[0, 1]
- **Trials:** 10,000 · **Seed:** 42 (xorshift32 PRNG) · **Independence:** inputs uncorrelated
- **Test:** for each trial: check I1 (A ∈ [0,1]) and no-NaN

### Adversarial Monte Carlo
- **Distribution:** Q, C ~ U[0.6, 1.0] (high credentials); E ~ U[0.6, 0.9] (EW-active threat); τ ~ U[0.05, 0.25] (degraded sensors)
- **Trials:** 5,000 · **Seed:** 42
- **Intent:** verify that no combination of high credentials can overcome degraded τ and high E

### Threshold independence
Thresholds (0.15 for H1 toggle rate, T0/T3/T4 tier boundaries) were set prior to simulation design and not tuned against these specific results.

---

## Ablation Study (adversarial inputs: Q,C ~ U[0.6,1.0], E ~ U[0.6,0.9], τ ~ U[0.05,0.25])

| Variant | Mean A | T0 Rate | What this shows |
|---------|--------|---------|-----------------|
| **Full HMAA** | **0.0092** | **100.0%** | All components active |
| No damping (kd=0) | 0.0579 | 88.1% | Damping contributes ~12% additional lockout under high E |
| No τ factor (τ=1 override) | 0.1041 | 49.2% | τ is the dominant lockout driver — removing it nearly halves lockout rate |
| No gate (gate=1) | 0.0186 | 100.0% | Gate alone doesn't change lockout rate; τ + damp suffice |
| No EW weight shift (dw=0) | 0.0090 | 100.0% | Weight shift has minimal effect under adversarial E (damp dominates) |

**Key finding:** τ is the dominant security mechanism under adversarial conditions. Damping provides a secondary layer. The gate sharpens the credential requirement as τ degrades (gam increases), but τ as a direct multiplier is the primary driver.

---

## Sensitivity Analysis (perturbations at nominal: Q=0.90, C=0.85, E=0.15, τ=0.98)

| Perturbation | A | ΔA | Dominant? |
|---|---|---|---|
| Baseline | 0.5128 | — | — |
| E − 0.05 | 0.5811 | +0.068 | E is the most sensitive input at nominal |
| E + 0.05 | 0.4525 | −0.060 | |
| Q + 0.05 | 0.5443 | +0.031 | Q, C comparable and symmetric |
| Q − 0.05 | 0.4819 | −0.031 | |
| C + 0.05 | 0.5421 | +0.029 | |
| C − 0.05 | 0.4839 | −0.029 | |
| τ − 0.05 | 0.4770 | −0.036 | τ drop more impactful than τ rise |
| τ → 1.00 | 0.5275 | +0.015 | τ already near ceiling at 0.98 |

E is the most sensitive input at the nominal operating point, followed by τ.

---

## Key Simulation Results (seed=42, Node.js v22.22.0, Linux x64)

| Scenario | n | Mean A | T0 Lockout | Assertion |
|----------|---|--------|-----------|-----------|
| Nominal (Q=0.90, C=0.85, E=0.15, τ=0.98) | 1,000 | 0.5128 | 0.00% | ✓ T2, no false lockout |
| Adversarial (Q/C high, τ≈0.15, E≈0.72) | 5,000 | 0.0107 | 100.00% | ✓ Full lockout |
| Sensor degradation (τ: 0.95→0.05) | 200 steps | 0.0265 | 97.50% | ✓ Graceful degradation |
| Trust collapse (C drops at step 50) | 200 steps | 0.1637 | 35.00% | ✓ Partial recovery |
| EW hysteresis stress (400-step oscillation) | 400 steps | 0.1420 | 0.00% | ✓ toggle_rate=0.1100 < 0.15 |
| CI gate (uniform random) | 10,000 | 0.0324 | 90.87% | ✓ 0 invariant violations |

**Reproducibility:** SHA-256 of results content (excluding timestamp/run-id) is stable across repeated runs on the same platform. Cross-platform numeric agreement is expected for IEEE-754 Node.js environments; bitwise identity across OS has not been verified. Confirm: `node simulate.js --builtin monte_carlo_standard --seed 42` → `mean=0.051052`.

---

## Failure Modes and Limitations

| Failure Mode | Condition | Engine Response | Limitation |
|---|---|---|---|
| Sensor spoofing | Attacker reports inflated τ | Engine trusts τ — cannot detect spoofing | τ provenance must be enforced externally |
| Q mis-scoring | Credential system assigns wrong Q | Engine grants authority at wrong level | Trustworthiness depends on input pipeline |
| E noise near 0.60 | E oscillates around EW threshold | Dead-band [0.55, 0.60) absorbs ≤5-point swings | Wider noise requires Eoff/Eon recalibration |
| Threshold mis-calibration | Defaults not matched to mission | Engine correct to its parameters, not the mission | Parameters require per-deployment calibration |
| No formal proof | I1–I3 tested, not proved | CI gate catches violations statistically | No mathematical guarantee for untested inputs |

---

## Hazard Checks (Evidence-Based, Not a Certified Safety Case)

> "Closed" here means: a scenario was designed, run, and the assertion passed. This is simulation evidence, not a certified safety argument. A full safety case requires a structured argument (GSN), explicit assumptions, and operational design domain specification.

| Check | Hazard | Evidence | Result |
|---|---|---|---|
| H1 | EW chattering destabilises control | ew_hysteresis_stress: toggle_rate=0.1100 < 0.15 | ✅ Passed |
| H2 | High-E environment grants elevated authority | adversarial_mc: T0=100%, mean A=0.0107 | ✅ Passed |
| H3 | False lockout of certified operator | nominal: T0=0.00%, A=0.5128 | ✅ Passed |
| H4 | NaN/Infinity propagation | ci_gate: 0 non-finite values / 10,000 trials | ✅ Passed |

Full GSN safety argument, FMEA, and evidence register (with SHA-256 content hashes) are in the [Technical Assurance Report v2.4.2].

---

## Related Work and Positioning

HMAA relates to the following lines of work:

- **Mixed-initiative control / adaptive autonomy:** Scerri et al. (2002), Goodrich & Schultz (2007) — authority sharing between human and autonomous agents based on task and context. HMAA adds an explicit sensor-trust gate (τ) absent from most mixed-initiative models.
- **Trust in autonomy / human-robot teaming:** Hancock et al. (2011), Mercado et al. (2016) — trust calibration in human-robot interaction. HMAA operationalises trust as a numerical gate rather than a latent variable, making it directly computable and auditable.
- **Safety-critical authority control:** Litt et al. (NASA/TM-2008-215419) — hardware authority enforcement in distributed engine control / FADEC. HMAA generalises this pattern to software-defined authority for general autonomous systems.
- **Hysteresis in control:** standard in bang-bang controllers and Schmitt triggers to prevent chattering. HMAA applies this to authority-state transitions via the EW dead-band.
- **Safety cases / GSN:** Kelly (1998), IEC 61508 — structured goal-strategy-evidence safety arguments. The hazard checks above are a lightweight analogue; the Technical Report contains a fuller GSN structure.

**What HMAA does not claim:** this is not formally verified, not a certified safety component, and not deployment-ready without mission-specific parameter calibration and integration-level validation.

---

## Repository Contents

```
hmaa/
├── index.html        ← Full interactive dashboard (any browser, no server, no install)
├── simulate.js       ← Headless CLI engine (Node.js ≥ 16, zero dependencies)
├── README.md         ← This file
├── CITATION.cff      ← Machine-readable citation metadata
└── LICENSE           ← MIT
```

---

## Live Dashboard

**Open `index.html` in any browser.**

| Tab | What it shows |
|-----|--------------|
| LIVE CALCULATOR | Q/C/E/τ sliders → A + all intermediates (gate, damp, wq) in real time |
| EXPERIMENT ENGINE | Monte Carlo (n=5000, seed=42) → distribution, percentiles, tier occupancy |
| SCENARIO LIBRARY | 14 pre-built scenarios — one click to run |
| MODEL COMPARISON | HMAA vs Threshold / Weighted / Logistic baselines head-to-head |
| SAFETY CASE | Live hazard checks — click RE-VERIFY to re-run assertions |
| REPRO | Seed audit — confirm deterministic output |

---

## CLI Quick Start

```bash
# Node.js ≥ 16 required — no npm install
node simulate.js --list
node simulate.js --builtin nominal              --seed 42
node simulate.js --builtin ci_gate              --seed 42   # exit 3 on invariant failure
node simulate.js --builtin adversarial_monte_carlo --seed 42
node simulate.js scenarios/custom.json          --seed 42 --out results.json
```

**Exit codes:** `0` pass · `1` input error · `2` simulation error · `3` assertion failure

---

## TL;DR (30 seconds)

HMAA is a deterministic authority-gating engine for autonomous systems.

It computes a value **A ∈ [0,1]** from four inputs (Q, C, E, τ) and maps the result to five authority tiers (T0–T4).

The goal is to prevent unsafe authority escalation when sensor trust is degraded or electronic warfare threat is high.

Quick test:

node simulate.js --builtin nominal --seed 42

---

## Citation

DOI assigned on Zenodo upload:

```bibtex
@software{oktenli2026hmaa,
  author    = {Oktenli, Burak},
  title     = {HMAA — Hierarchical Mission Authority Architecture (v2.4.2)},
  year      = {2026},
  version   = {2.4.2},
  url       = {https://github.com/burakoktenli-ai/hmaa},
  note      = {Georgetown University, MPS Applied Intelligence. DOI assigned on Zenodo upload.}
}
```

---

## References

- Goodrich, M. A., & Schultz, A. C. (2007). Human-robot interaction: a survey. *Foundations and Trends in HRI*, 1(3), 203–275.
- Hancock, P. A., et al. (2011). A meta-analysis of factors affecting trust in human-robot interaction. *Human Factors*, 53(5), 517–527.
- IEC 61508. (2010). *Functional Safety of E/E/PE Safety-Related Systems*. Geneva: IEC.
- Kelly, T. P. (1998). *Arguing Safety: A Systematic Approach to Managing Safety Cases*. PhD thesis, University of York.
- Litt, J. S., et al. (2008). Distributed engine control: the next step in propulsion control technology. NASA/TM-2008-215419. NASA Glenn Research Center.
- Mercado, J. E., et al. (2016). Intelligent agent transparency in human-robot teaming. *Human Factors*, 58(3), 401–415.
- NIST SP 800-218. (2022). *Secure Software Development Framework (SSDF)*. Gaithersburg: NIST.
- RTCA DO-178C. (2011). *Software Considerations in Airborne Systems and Equipment Certification*. RTCA/EUROCAE.
- Scerri, P., et al. (2002). Adjustable autonomy for the real world. *Journal of Experimental & Theoretical AI*, 14(2–3), 171–206.

---

## License

MIT — see [LICENSE](LICENSE).

---

*Georgetown University · School of Continuing Studies · MPS Applied Intelligence*
