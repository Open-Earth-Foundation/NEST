Status: Draft

Source: Open-Earth-Foundation/NEST-workspace (private) at commit e56ee11479189cf2077a8357bdd1b35a79275bd0, path docs/programme/partners/aii/deliverables/2026-08/AII-D03-sturm-wrapper.gnn.md. Copied unchanged below this line. Links into the workspace do not resolve here.

---

# GNN: STURM Building-Stock Model — Markov Blanket Wrapper (NEST AII-D03 draft)
# GNN Version: 1.0
# Machine-readable draft of the STURM wrapper blanket for NEST's national↔urban coupling (drafted 2026-09-11). STURM = MESSAGEix-Buildings (IIASA, `iiasa/message-ix-buildings`: R core, Python wrapper) — confirmed from the code 2026-07-14; the packaged version is pending IIASA. EXP-01 scope (IIASA, 2026-09-22): buildings — residential + non-residential (commercial less detailed), no appliances. Dimensions shown (1 region, 2 fuel types, 13 periods, 4 archetypes) are ILLUSTRATIVE except the period count: 13 = the model years 2020–2060 by 5 and 2070–2100 by 10 that STURM's WEU runner emits, one value each (observed 2026-10-09 on iiasa/message-ix-buildings@e560643; the earlier "12 time slices" was a placeholder with no basis — STURM's native step is 5 years). Temporal support changes the answer class (docs/research/boundary-convention-benchmarks.md, finding 2), so the packaged run's year set is still to be pinned, not declared here. The blanket structure is scenario-invariant; only the dims instantiate per scenario.

## GNNSection
SturmWrapper

## GNNVersionAndFlags
GNN v1

## ModelName
STURM Building-Stock Wrapper Blanket

## ModelAnnotation
Markov blanket for STURM as a coupled component in NEST Stage 1.

Boundary scope: this blanket is drawn around MODEL ↔ NEST PLATFORM. STURM
receives energy prices by fuel type (from the national model) and
returns final energy demand by fuel; renovation and stock-turnover machinery
(archetype shares, vintage survival, fuel switching) stays internal and is
never exchanged. This is the AII-D02 mirror with sensory/active reversed on the
price/demand pair — AII-D02's sensory d ↔ AII-D03's active d, AII-D02's active p ↔ AII-D03's
sensory p. There is NO counterpart to AII-D02's cap: AII-D02's cap is the HIERARCHY
regime imposed constraint of PR #87 Q-13; STURM has no imposed constraint
input, so this blanket has no such sensory variable.

Scope vs Q1 (Technical_Responses_Draft_v1): this file specifies the wrapper's
model-level I/O blanket — Q1 functions #1–#3. Q1 function #4 (the Active
Inference latent-summary layer, via RxInfer) is deliberately NOT modeled here;
those latent states are the factor-graph deliverables (AII-D04, AII-D07), which will
sit behind this blanket's internal side.

- Sensory states: energy prices by fuel type and emission factors per fuel, both from the national model. Emission factors are used for CO2 reporting and do not enter the stock-turnover dynamics.
- Active states: final energy demand by fuel. Renovation and turnover rates are produced and consumed inside STURM; they are internal, not exchanged.
- Internal states: archetype stock shares, renovation/turnover flows and the turnover state machinery — never exchanged.
- Where the rest lives: STURM's provenance (MESSAGEix-Buildings, `iiasa/message-ix-buildings`), EXP-01 scope, per-variable units, conventions, supports, status, the catalog units conflict (#152) and what AII observed on 2026-07-14 are in AII-D03.boundary.yaml. The translation to MESSAGEix — temporal alignment, spatial exchange, allocation basis, the mapping rule, the shared scenario id and the convergence contract — is written once on the seam, AII-D02-D03-seam-messageix-sturm.yaml.
- This mirrors the AII-D02 pattern file (MESSAGEix wrapper); AII-D03 is its price/demand mirror.
- Renderable form: structural blanket. The renderable instantiation is the continuous-exemplar fixture block (`models/active_inference/rxinfer-bridge/fixtures/sturm/sturm_fixture.gnn.md`); render the instantiation, not the wrapper (upstream: GeneralizedNotationNotation#111; resolution recorded in AII-D14).

## StateSpaceBlock
# This spec owns names, dims, types, roles and connections.
# Units, conventions, supports, status and provenance for every blanket variable live in
# AII-D03.boundary.yaml (keyed by variable name); translation to MESSAGEix lives on the seam.

# Sensory — exchanged (inputs crossing the blanket from MESSAGEix)
p[2,13,type=float]     # energy prices by fuel type
ef[2,13,type=float]    # emission factors per fuel (CO2 reporting only — no edge into the turnover dynamics, by design)

# Active — exchanged (outputs crossing the blanket to MESSAGEix)
d[2,13,type=float]     # final energy demand by fuel, buildings

# Internal (never exchanged; no boundary-file entry, so their units stay here)
q[4,type=float]        # archetype stock shares [share; sums to 1]
r[4,13,type=float]     # renovation / turnover flows per archetype per period [share/yr]
u[8,1,type=float]      # internal turnover state summary (vintage survival, fuel-switching machinery)

# Time
t[1,type=int]          # coupling iteration index

## Connections
p>u
u>q
u>r
q>d
r>d
d>p

## InitialParameterization
# Instantiated per scenario from the versioned STURM baseline (pending IIASA
# packaging); the wrapper blanket declares structure, not values. The d>p
# edge documents loop closure through the external national model (MESSAGEix
# side, AII-D02) — it is not a claim that this wrapper models MESSAGEix's
# response; the coupling closes only because AII-D02's p>d and AII-D03's d>p are two
# halves of the same external loop.
p={(0.0,0.0)}

## Equations
# Stock-turnover simulation, not an LP: given prices p, evolve the archetype
# stock q_t = survival(q_{t-1}; vintage decay) + renovation + new construction,
# switch fuels at the renovation margin (price-elastic), and report final
# energy demand d_t = sum over archetypes of stock x fuel intensity.
# No duals: there is no optimisation objective on this side of the loop —
# contrast with AII-D02, where p is a PRICE_COMMODITY dual of the LP and carries
# the engine's annualisation convention (docs/research/boundary-convention-benchmarks.md, finding 1). AII-D03's p is a
# *price input*; its convention is the downscaling convention of the
# national→STURM handoff, not a dual convention.

## Time
Dynamic
DiscreteTime=t
ModelTimeHorizon=20   # illustrative horizon for the t index (coupling passes), for parsing and fixtures only — NOT the iteration contract, which is the seam's convergence.max_iterations (null: a configuration value; once-through in practice). Was Unbounded

## ActInfOntologyAnnotation
p=Observation
ef=Observation
d=Action
q=HiddenState
r=HiddenState
u=HiddenState
t=Time

## ModelParameters
num_fuel_types: 2
num_periods: 13       # model years 2020–2060 by 5, 2070–2100 by 10, as emitted by the WEU runners (observed 2026-10-09); STURM's native step is 5 years — see AII-D03.boundary.yaml#variables.d.temporal_support
num_archetypes: 4
num_regions: 1

## Footer
NEST AII-D03 draft — AII (Holly Grimm), 2026-09; structure/declaration split 2026-10-04. Mirror of AII-D02 (MESSAGEix wrapper); STURM assumption pending IIASA confirmation. Boundary-field semantics: packages/contracts/ + docs/research/boundary-convention-benchmarks.md.

## Signature
NEST AII Methods WG
