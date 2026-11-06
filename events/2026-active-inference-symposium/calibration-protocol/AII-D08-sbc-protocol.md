Status: Experiment

Source: Open-Earth-Foundation/NEST-workspace (private) at commit e6923581d6f484ee69c720830ceca17bf0e0283f, path docs/programme/partners/aii/deliverables/2026-09/AII-D08-sbc-protocol.md. Copied unchanged below this line. Links into the workspace do not resolve here.

---

# AII-D08 — SBC protocol + first synthetic run

## Status

`done` (2026-09-21: SBC protocol pre-registered before any SBC code ran, amended before the first run with a measured statistic-calibration finding; the first synthetic run — 1000 sims of the D07 controlled 4-state model in one Julia process — is logged with per-state KS statistics and coverage in `runs/d08/`; all three acceptance criteria met, evidence below)

## Owner

AII

## Depends on

AII-D07

## Acceptance criteria

- Written SBC protocol
- First run with KS statistics
- Gates aligned with Year1 plan

## Evidence / path

**Review note (2026-09-14, from the AII-D04/AII-D03 model runs):** RxInfer's Bethe free energy is environment-dependent — byte-identical posteriors carry different FE values on the host Julia 1.12.7 stack vs the pinned image — and is not the log evidence (the exact Kalman innovation log-likelihood over the AII-D04 fixture is +118.0224). See the rxinfer-bridge README note before any FE-based gate lands in this protocol.

### Protocol — simulation-based calibration of the D07 4-state controlled LGSSM (pre-registered 2026-09-21, before any SBC code ran)

**What is calibrated.** Talts-style simulation-based calibration (Talts et al. 2018, *"Validating Bayesian inference algorithms via simulation-based calibration"*) adapted to the AII-D07 model: a 4-state **controlled** linear-Gaussian LGSSM (`x_1 ~ N(prior_mean, prior_cov)`; `x_t = F x_{t-1} + B u_t + N(0, Q)`; `y_t = H x_t + N(0, R)`) whose parameters are **declared constants** of the committed fixture's generating process — the RxInfer graph *is* the generating process. So this is **posterior calibration SBC for the latent states**: per simulation the "prior draw" is a full state trajectory `x[1:T,4]` and observations `y[T,4]` under the fixed parameters; the rank statistic is the true state's rank within its posterior marginal. Any parameter-posterior calibration is out of scope while the parameters stay declared constants (they are observed, not inferred — the same regime the D07 posterior check ran under). What the coupling loop consumes is exactly these per-iteration latent beliefs, so state calibration is the quantity Stage 2 leans on.

**Draw per simulation (the prior, under fixed parameters).** `N = 1000` simulations, horizon `T = 24` (the committed fixture's), control `u` = the fixture's declared schedule (`d07_fixture.control(24)`: zero to `t = 11`, then the −0.0030/iteration tightening ramp — declared exogenous input, **not** drawn). Sim `i` (0-based) seeds numpy `default_rng(seed_base + i)` with **seed base 20260923** and draws, through the committed, A1–A8-tested generator `d07_fixture.simulate` (D04's draw order: one standard-normal vector for `x_1`, then the `T−1` process noises, then the `T` observation noises, covariances through Cholesky factors): `x_1 ~ N(prior_mean, prior_cov)`, `x_t = F x_{t-1} + B u_t + N(0, Q)`, `y_t = H x_t + N(0, R)`. Reusing the tested generator is deliberate: the SBC draws come from the exact code path that produced the committed fixture.

**Inference.** The D07 RxInfer graph — `d07_lgssm` from `julia/d07_model.jl`, reused by inclusion, not copied — runs full-data smoothing on each sim's `(y, u)`. All `N` sims run in **one Julia process** (`julia/d08_sbc.jl`, driven by `src/nest_rxinfer_bridge/d08_sbc.py`): standalone-CLI startup (~10 s) makes one process per sim infeasible at this N, and the per-sim inference is stateless, so the loop is inside the process. No free-energy computation per sim (the 2026-09-14 review note: the Bethe FE is environment-dependent and is not evidence; the exact innovation log-likelihood is — see the diagnostic below).

**Rank statistic (stated exactly).** The **normalized rank (probability integral transform)**: for sim `i`, iteration `t`, state component `k`, with the graph's smoothed marginal `N(m_{i,t,k}, v_{i,t,k})` (mean; variance = the diagonal of the graph's posterior covariance) and the drawn truth `x*_{i,t,k}`:

    u_{i,t,k} = Φ((x* − m)/√v)     (Φ = standard-normal CDF)

computed from the **exact** Gaussian marginal, not from samples. Why this form and not the 999-draw integer rank: the smoothed marginals on this tree are *exactly* Gaussian (the D07 run's posterior matched the Kalman-with-control reference to 3.2e-16), and for an exactly Gaussian posterior the PIT is the continuous limit of Talts' normalized rank as the number of posterior draws grows; taking it from the exact CDF removes the Monte Carlo noise a 999-draw rank would inject into every statistic, so the KS test below is exact rather than itself a Monte Carlo estimate. As a recorded cross-check, the Talts-style discrete ranks (999 posterior draws + the truth, rank of the truth among the 1000) are computed per `(i, t, k)` from a dedicated seeded numpy stream and pooled into per-state rank histograms — agreement of the discrete-rank histogram with the PIT result is part of the evidence record.

**Uniformity test.** Per state component `k`: Kolmogorov–Smirnov of the `N·T` pooled PIT values against `Uniform(0,1)`. No scipy in the component environment (pinned image installs numpy only), so the KS statistic and the Stephens-corrected asymptotic Kolmogorov p-value are implemented in numpy and tested against the published critical values (`tests/test_d08_sbc.py`). **Gate (recorded here as the forward-looking AII-D19 gate): KS p > 0.05 per state** — the threshold belongs to the AII-D19 gate record; D08 writes the protocol and this first run's numbers.

**Amendment (2026-09-21, before the first run — the pooled KS is anti-conservative under within-simulation dependence).** Dry-run calibration of the test statistic (numpy exact posteriors, 100 replicates of `T = 12`, `N = 100` — a correctly-specified model, so every rejection is a false positive) showed the paragraph above's pooled `N·T` KS **cannot be the per-state gate**: the PIT values within one simulation share the data draw and are strongly correlated across `t` — most of all for the slowest, least-observed state (`policy_latency`, `F₄₄ = 0.94`) — so the pooled ECDF fluctuates far more than `n = N·T` i.i.d. values would. Measured false-rejection rates at the 0.05 level: **21 % / 3 % / 17 % / 66 %** (decarb / demand / margin / latency) against a nominal 5 %. The rank statistic itself is untouched — each `u_{i,t,k}` is marginally exactly `Uniform(0,1)` (per-`t` KS tests all calibrated) — only the pooling is wrong. **Amended per-state uniformity test:** a KS test per state **per iteration** (pooling the `N` independent simulations at each `t`, where the PIT values *are* i.i.d.), with the per-state gate statistic `min_t p_t` and threshold `0.05/T` (Bonferroni — exact under arbitrary within-simulation dependence). Measured on the same 100 replicates: `min_t p_t < 0.05/12` in **7 % / 5 % / 3 % / 4 %** of replicates — calibrated at the nominal 5 %. The plan-worded pooled KS stays in the run record as a **descriptive** statistic with this caveat, and the KS implementation (Stephens-corrected p-value) is unchanged. Coverage reporting gains an honest standard error from the per-simulation coverage indicators (i.i.d. across sims). Everything else in this protocol is unchanged; no SBC run had been executed when this amendment was recorded.

**Coverage check.** Central credible intervals from the exact Gaussian marginals: 90% CI = `m ± z_{0.95}·σ`, 95% CI = `m ± z_{0.975}·σ`. Empirical coverage per state, pooled over all sims and iterations, recorded for both levels — the 90% level against the **Year-1 band 85–95%** (the AII-D19 gate row "Coverage 85–95% empirical"); the 95% level recorded alongside for context.

**Diagnostic (log evidence, not FE).** Per-sim exact innovation log-likelihood `log p(y_{1:T} | u_{1:T})` (pure-Julia `innovation_loglik`, reused from `d07_model.jl`): distribution summary (mean, sd, min, max) over the `N` sims. Sanity anchor: the committed fixture's draw (seed 20260922) scored **+120.2121**; the SBC draws come from the same generating process, so the distribution should centre near it. Per-sim Bethe free energy is deliberately **not** recorded (environment-dependent, per the review note).

**Expectation and honesty clause.** Exact inference on a correctly specified linear-Gaussian model ⇒ PIT uniform (KS p > 0.05) and coverage at the nominal levels. If the run disagrees, that is **a finding to record, not something to tune away**: the evidence section reports whatever the run produced.

**What this SBC can and cannot detect (review ask, 2026-09-26).** With the parameters fixed constants and the smoothed posteriors exactly Gaussian (RxInfer matching the exact Kalman posterior to ~1e-15), the PIT ranks are uniform **by construction** — the run can only fail through a mismatch between the generator and the model, i.e. a bug. That makes it a regression guard, **not** evidence of calibration under uncertainty: parameter-posterior calibration is scoped out of D08 explicitly, and **D13/D19 must add parameter uncertainty (priors on `F`, `B`, `Q`, `R`) before the D19 gate means what its name says** — a reader should not over-read the 0.897–0.903 coverage.

**Artifacts.** `runs/d08/d08_sbc_record.json` (protocol parameters + per-state KS table + coverage table + log-likelihood summary + versions/timing), `runs/d08/d08_ranks.npz` (per-sim truths, posterior means/variances, PIT values, discrete ranks — full reconstructability). Driver: `python -m nest_rxinfer_bridge.d08_sbc` (standalone CLI + one JSON record, the documented pattern; Julia ≥ 1.12 host path like D04/D07).

**Commit order:** this protocol → SBC code (Julia + Python driver + tests) → run artifacts → evidence record below (status decision last).

### First synthetic run — evidence (registered run 2026-09-21, macOS/arm64)

**Delivered:** SBC loop [`julia/d08_sbc.jl`](../../../../../../models/active_inference/rxinfer-bridge/julia/d08_sbc.jl) (includes `d07_model.jl` — the D07 graph reused by inclusion, not copied; N sims in ONE process), driver [`src/nest_rxinfer_bridge/d08_sbc.py`](../../../../../../models/active_inference/rxinfer-bridge/src/nest_rxinfer_bridge/d08_sbc.py) (standalone CLI, one JSON record, the documented pattern), tests [`tests/test_d08_sbc.py`](../../../../../../models/active_inference/rxinfer-bridge/tests/test_d08_sbc.py) (16 numpy tests: the normal CDF/quantile and the Stephens-corrected KS p-value against published critical values, the PIT rank statistic, the Talts-rank cross-check, and an end-to-end mini-SBC on the model's exact A6-validated posterior), run artifacts [`runs/d08/`](../../../../../../models/active_inference/rxinfer-bridge/runs/d08/) (`d08_sbc_record.json`, `d08_ranks.npz` — truths, posteriors, PIT values, integer ranks — and the raw Julia output `d08_sbc_posteriors.json`).

**Run order kept:** protocol pre-registration committed (`5b71553f`) → SBC code committed (`7eb06446`) → run artifacts committed (`10f3f011`) → protocol amendment (`dbeb03ca`, drafted before the run executed — its text records that) → this evidence record. Not pushed. **The ledger order is checkable by a reader:** the gate change itself (per-iteration KS with the Bonferroni min-p gate) is implemented in `7eb06446` (committed 2026-09-21T09:13) — before the first run's artifacts (`10f3f011`) exist; `git show 7eb06446:models/active_inference/rxinfer-bridge/src/nest_rxinfer_bridge/d08_sbc.py` already contains the per-iteration/Bonferroni gate. Process note from review: next time the amendment commits *before* the run, so the ledger shows the order on its own rather than via drafted-before evidence.

**As run (2026-09-21, macOS/arm64, julia 1.12.7 + RxInfer 5.5.0, python 3.14.6 + numpy 2.5.3 — the D07 host stack):** `PYTHONPATH=src python3 -m nest_rxinfer_bridge.d08_sbc` — **N = 1000** sims, `T = 24`, seed base `20260923` (seeds `20260923 + i`, i = 0…999), control = the declared cap schedule, draws through the committed `d07_fixture.simulate`. All 1000 sims inferred in **one Julia process**: 23.6 s inside Julia (**0.0236 s/sim** steady state — one process per sim would have cost ~1000 × the ~10 s standalone-CLI startup), 41 s wall.

**Posterior proof, per sim (the ranks mean something only if the graph's posterior is the exact one):** the D07 graph's smoothed posterior vs the exact joint-Gaussian conditional (`d07_fixture.exact_joint_posterior`, the A6-validated reference) over **every one of the 1000 sims**: max |Δmean| **2.8e-15**, max |Δvar| **2.6e-17** — round-off against the 1e-6 bar. Innovation log-likelihood numpy↔Julia: max diff **9.9e-14**. Fixture-constant provenance: the generator's matrices equal the committed fixture's to **0.0**.

**Uniformity (the amended per-state gate) — all four states pass.** Per-iteration KS (N = 1000 i.i.d. sims per iteration), per-state minimum p over the 24 iterations against the Bonferroni threshold 0.05/24 = 0.00208:

| State | min per-iteration KS p | argmin t | pooled-KS p (descriptive) | Gate (p > 0.00208) |
|---|---|---|---|---|
| `decarb_rate_dev` | 0.1108 | t = 5 | 0.583 | pass |
| `demand_pressure` | 0.0676 | t = 17 | 0.396 | pass |
| `capacity_margin_pressure` | 0.0349 | t = 1 | 0.239 | pass |
| `policy_latency` | 0.0192 | t = 12 | **8.06e-06** | pass |

**The amendment's finding, demonstrated live by this run:** under the pre-amended pooling, `policy_latency` would have been read as miscalibrated (pooled p ≈ 8e-06) while its 24 per-iteration tests are all calibrated (min p 0.019) — the anti-conservatism the dry-run measured (66 % false-rejection for this state at `T = 12`, `N = 100`) is exactly what the pooled statistic produces on a correctly calibrated model. The per-iteration KS with the Bonferroni gate is the recorded test; the pooled number stays in the run record (`runs/d08/d08_sbc_record.json` → `ks_table`) flagged descriptive.

**Coverage — all four states in the Year-1 band, at nominal:** 90 % CI coverage **0.8974 / 0.8976 / 0.8996 / 0.9032** (decarb / demand / margin / latency; SE 0.0026 / 0.0022 / 0.0019 / 0.0049 from the i.i.d.-across-sims indicators) — inside **85–95 %** and within 0.004 of the nominal 0.90; 95 % CI coverage **0.9483 / 0.9469 / 0.9499 / 0.9498**. PIT mean per state 0.4917–0.5008 (sd 0.286–0.290 against the uniform 0.2887).

**Log evidence (not the Bethe FE):** per-sim exact innovation log-likelihood over the 1000 draws — mean **120.099**, sd 6.624, range **[98.067, 135.722]** — centred on the committed fixture's **+120.2121** anchor exactly as the protocol expected (the fixture's draw is one point of this distribution). Per-sim Bethe free energy not computed (environment-dependent; review note).

**Talts rank cross-check (999-draw integer ranks):** per-state decile counts within ±1.3 % of flat (2400 per bin), mean ranks **500.3 / 498.3 / 497.5 / 491.3** against the uniform expectation 499.5 — the sampled ranks agree with the exact-PIT result.

**Tests as run:** `PYTHONPATH=src python3 -m unittest tests.test_d08_sbc` — **16 tests, 16 pass**; the D07 and D04 fixture suites re-run alongside (17 pass — the shared generator untouched). The Julia loop is exercised by the run itself (the D04/D07 discipline: model runs are host-run records, not CI unit tests).

**In-image replication (2026-09-26, hollygrimm — closes the open replication item):** the full SBC re-run in the pinned image (`ghcr.io/open-earth-foundation/nest-rxinfer-bridge:sha-0c6762220ba546b70feed4e592a73d1fd15ff49a`, Julia 1.11.7, RxInfer 5.5.0, read-only, UID 10001) with this branch's `src/ tests/ fixtures/ julia/` mounted: **every statistic identical to the committed record** (per-state min-p, coverage, ranks, log-likelihood summary; only round-off fields differ) and the unit suite 46 tests OK — PR #144 review comment 5850085456.

**Status decision:** all three acceptance criteria are met — the written SBC protocol (pre-registered, then amended *before* the first run with a measured, documented reason); the first run with KS statistics (per-state, `runs/d08/`, posterior provenance at round-off on every sim); and the gates aligned with the Year-1 plan thresholds, recorded as the forward-looking AII-D19 gate (the plan's "KS p > 0.05" and "coverage 85–95 % empirical" rows; the dependence-valid operationalization — per-iteration KS, Bonferroni min-p — is documented in the protocol amendment for D19 to adopt or revise). Hence `done`.

## Links

- PLAN: [../PLAN.md](../../PLAN.md)
- Year 1: [Year1 plan](../../../../../archive/programme/2026-05-year1-6month-plan.md) — WS1 Stage 2
