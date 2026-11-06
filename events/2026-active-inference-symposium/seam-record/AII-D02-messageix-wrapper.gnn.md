Status: Draft

Source: Open-Earth-Foundation/NEST-workspace (private) at commit e56ee11479189cf2077a8357bdd1b35a79275bd0, path docs/programme/partners/aii/deliverables/2026-08/AII-D02-messageix-wrapper.gnn.md. Copied unchanged below this line. Links into the workspace do not resolve here.

---

# GNN: MESSAGEix National Energy Model — Markov Blanket Wrapper (NEST AII-D02 draft)
# GNN Version: 1.0
# Machine-readable draft of the MESSAGEix wrapper blanket for NEST's national↔urban coupling. Dimensions shown (1 region, 2 demand commodities, 13 periods, 4 technologies) are ILLUSTRATIVE except the period count: 13 = the model years 2020–2060 by 5 and 2070–2100 by 10, one annual slice each, observed 2026-10-09 on the R11 baseline and STURM's WEU runner (the earlier "12 time slices" was a placeholder with no basis). Temporal support changes the answer class (docs/research/boundary-convention-benchmarks.md, finding 2), so the reference run's own year set is still to be pinned, not declared here. The blanket structure is scenario-invariant; only the dims instantiate per scenario. Since 2026-10-04 this spec owns structure only (names, dims, types, roles, connections); units, conventions, supports, status and provenance live in AII-D02.boundary.yaml.

## GNNSection
MESSAGEixWrapper

## GNNVersionAndFlags
GNN v1

## ModelName
MESSAGEix National Wrapper Blanket

## ModelAnnotation
Markov blanket for MESSAGEix as a coupled component in NEST Stage 1.

Boundary scope: this blanket is drawn around MODEL ↔ NEST PLATFORM. Of the
active states, p and ef cross onward to the coupled model (STURM); e, lam and J
are active toward the platform (gates, monitoring, convergence accounting)
and never reach STURM. cap arrives as an imposed constraint — the HIERARCHY
regime of PR #87 Q-13; the AII-D03 (STURM) blanket has no counterpart input.

Scope vs Q1 (Technical_Responses_Draft_v1): this file specifies the wrapper's
model-level I/O blanket — Q1 functions #1–#3. Q1 function #4 (the Active
Inference latent-summary layer, via RxInfer) is deliberately NOT modeled here;
those latent states are the factor-graph deliverables (AII-D04, AII-D07), which will
sit behind this blanket's internal side.

- Sensory states: demand per commodity (exchanged, from STURM) and the emissions cap (imposed, platform-facing).
- Active states: commodity prices (PRICE_COMMODITY duals) and emission factors (both to STURM); total emissions, the carbon shadow price and the objective value (platform-facing).
- Internal states: capacities, dispatch and the LP solution machinery — never exchanged.
- Where the rest lives: per-variable units, conventions, supports, validity bounds, status and provenance (including what AII observed on 2026-07-14) are in AII-D02.boundary.yaml. The translation to STURM — temporal alignment, spatial exchange, allocation basis, the mapping rule, the shared scenario id and the convergence contract — is written once on the seam, AII-D02-D03-seam-messageix-sturm.yaml.
- This is the AII-D02 pattern file; AII-D03 (STURM wrapper) mirrors it with sensory/active reversed on the price/demand pair.
- Renderable form: this wrapper is a *structural* blanket (it declares boundary structure, not a generative parameterization). The renderable instantiation of this blanket is the continuous-exemplar fixture block (`models/active_inference/rxinfer-bridge/fixtures/d04/d04_fixture.gnn.md`), which renders to RxInfer and executes; render the instantiation, not the wrapper (upstream: GeneralizedNotationNotation#111; resolution recorded in AII-D14).

## StateSpaceBlock
# This spec owns names, dims, types, roles and connections.
# Units, conventions, supports, status and provenance for every blanket variable live in
# AII-D02.boundary.yaml (keyed by variable name); translation to STURM lives on the seam.

# Sensory — exchanged (inputs crossing the blanket from STURM)
d[2,13,type=float]     # demand per commodity per period

# Sensory — imposed (platform-facing policy lever; never exchanged with STURM)
cap[1,type=float]      # emissions cap

# Active — exchanged (outputs crossing the blanket to STURM)
p[2,13,type=float]     # commodity prices (PRICE_COMMODITY duals)
ef[2,13,type=float]    # emission factors per commodity

# Active — platform-facing (never exchanged with STURM)
e[1,type=float]        # total emissions
lam[1,type=float]      # carbon shadow price (dual of the emissions bound)
J[1,type=float]        # objective value

# Internal (never exchanged; no boundary-file entry, so their units stay here)
K[4,type=float]        # installed capacity per technology [MW]
x[4,13,type=float]     # dispatch per technology per period [MWh]
s[8,1,type=float]      # internal solver state summary (basis, duals machinery)

# Time
t[1,type=int]          # coupling iteration index

## Connections
d>s
cap>s
s>K
s>x
K>p
x>p
x>e
x>ef
s>lam
s>J
p>d

## InitialParameterization
# Instantiated per scenario from the versioned ixmp platform; the wrapper
# blanket declares structure, not values. Which future is being run is named by
# the seam's scenario.id, shared with the STURM wrapper (AII-D03), so both models
# provably run the same future. The p>d edge documents loop closure through the
# external building-stock model (STURM side, AII-D03) — it is not a claim that
# this wrapper models STURM's response.
d={(0.0,0.0)}
cap={(200000.0)}

## Equations
# min_{K,x} J = annualised capital(K) + fixed(K) + variable(x)
# s.t. demand balance per commodity/slice (dual: p), capacity limits,
#      emissions(x) <= cap (dual: lam, the carbon shadow price).
# Duals are exact with respect to the engine's own objective convention —
# the declared `convention` field exists because nothing about the number
# announces which convention produced it (docs/research/boundary-convention-benchmarks.md, finding 1).

## Time
Dynamic
DiscreteTime=t
ModelTimeHorizon=20   # illustrative horizon for the t index (coupling passes), for parsing and fixtures only — NOT the iteration contract, which is the seam's convergence.max_iterations (null: a configuration value; once-through in practice). Was Unbounded

## ActInfOntologyAnnotation
d=Observation
cap=Observation        # imposed, not inferred from STURM; a lever for scenario work. Whether a prior-preference term fits better is open.
p=Action
ef=Action
e=Action
lam=Action
J=Action
K=HiddenState
x=HiddenState
s=HiddenState
t=Time

## ModelParameters
num_commodities: 2
num_periods: 13       # model years 2020–2060 by 5, 2070–2100 by 10, one annual slice each (observed 2026-10-09) — see AII-D02.boundary.yaml#variables.d.temporal_support
num_technologies: 4
num_regions: 1

## Footer
NEST AII-D02 draft — AII (Holly Grimm), 2026-08; structure/declaration split 2026-10-04. Pattern for AII-D03 (STURM). Boundary-field semantics: packages/contracts/ + docs/research/boundary-convention-benchmarks.md.

## Signature
NEST AII Methods WG
