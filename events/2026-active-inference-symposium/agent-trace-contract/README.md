Status: Accepted

Source: Open-Earth-Foundation/NEST-workspace (private) at commit 1d10d92c7ba2ed770683953d1080c9f695c57e0b, path packages/contracts/agent-trace/README.md. Copied unchanged below this line. Links into the workspace do not resolve here.

---

# Agent Trace Record

## Purpose and ownership

**Owner:** AII (Methods WG) · **Producers:** any NEST agent harness (Setup-01 onward) · **Consumers:** EXP-05 behavioural evaluation; the console's trace-read strip and decomposable pace bar (PR #80 sketch); any future audit or safety tooling.

One record per agent step, flat JSONL, emitted by the harness — not self-reported by the model. The design constraint throughout: **everything below is data the harness already holds at the moment of the decision.** Emitting it is serialization, not extra inference; there is no additional model call anywhere in this contract. (This answers the #80 question about whether capturing alternatives is near-free: it is, *for the definition pinned here*.)

Records are compatible with an mlflow-snapshot sink (Mirco, #74): flat JSON objects, one per step; an exporter is a thin adapter. The v0.1 execution-envelope extension adds records that consumers must distinguish from physical model calls, as described below.

## Versions

| Version | Status and producer use | Schema |
|---|---|---|
| v0 | Preserved original contract. Its schema is unchanged; historical exports retain their recorded version. | [v0/schema.json](v0/schema.json) |
| v0.1 | Experimental profile currently emitted by `services/agent-orchestrator`. External AII consumer acceptance remains unverified. | [v0.1/schema.json](v0.1/schema.json) |

The supplied v0 examples validate against both versions. v0.1 defines participant roles,
`decision` candidates, nullable tool arguments, `tool_kind`, experiment/execution-envelope
metadata and prompt-cache telemetry. See the v0.1 section below for their consumer requirements.
The optional tool-call `hash_policy` versions hash semantics independently of the record shape:
new orchestrator calls use `tool-outcome/v1`; historical unmarked calls keep their original hashes.

Original v0 allows unknown properties: a role field alone may pass its validator without that
version defining its meaning. `decision` candidates and nullable tool arguments do not pass v0.
Schema validation therefore does not establish that a consumer understands a producer's profile.

Field semantics (uncertainty/logprobs, alternatives-considered edge cases) are expected to tighten
as further harnesses prove each field extracts. v1 is declared only after those semantics stabilize.

## Shared record structure and fields

In both versions, the schema **root** is the line contract — `AgentStepRecord | SpawnRecord | ReturnRecord`, selected by `record_type` — so a standard Draft 2020-12 validator pointed at the chosen version enforces it directly. The rationale per field group:

| Field group | What is recorded | Why it can't be reconstructed later |
|---|---|---|
| **Choice set** | The actions available at this step — the tool menu the harness would have accepted **plus the continue/halt decision** — and the candidate call(s) the model actually emitted, including candidates the harness then refused (budget, policy). A parallel decision (one model response selecting several tools) is ONE choice point: `selected_indices` lists every pick in response order, and each ToolCallRecord carries `choice_point_event_id` + `candidate_index` back to the candidate it realizes — the explicit mapping that survives repeated same-tool calls, retries, skips, and out-of-order completion. `selected_indices` may repeat an index (the same tool called twice in one response is two picks of one candidate), and a halt chosen in the same response as a tool call is simply a further pick. Source: the tool registry and the parsed reply — never scraped from prompt text. | Without the counterfactual there is nothing to decompose: "chose an untried option" (exploration) and "chose an exhausted one" (redundancy) are indistinguishable. The stopping decision is the choice point implementers omit unless named. |
| **Inputs / outputs, hashed** | Versioned canonical-JSON hashes of tool inputs/outcomes; `tool-outcome/v1` includes the tool name on input and status/result/error on output (see the exact rules below). Per-tool comparison policies for nested volatile fields remain a separate follow-up. | Hashes help identify repeat results, but volatile fields can hide substantive repeats. Equality of results does not itself prove the action was redundant. |
| **Attempts / retries / backtracks** | Attempt counts per call; failures the agent recovered from silently; reverts. Failures recorded **at the harness level**, including calls dropped by budget/policy that the model never saw an error for. A dropped call keeps its `input_hash` (it is still an identifiable request), has no `output_hash` (nothing ran), and does not advance the attempt count — it was never issued. | A harness that drops a call without error text produces a trace that looks cleaner than the run was. Observed live: a model's batched second call was dropped with `result: null` and the model re-looped for 119 turns. |
| **Spawn / return** | Sub-agent spawn and return records with parent linkage (session id, level). | Parent linkage is what lets a nested run be read as one tree rather than disconnected logs — live if the recursive architecture (#87) nests agents. |
| **Uncertainty signal** | Whatever the harness exposes — logprobs, sampling params, self-reported confidence — with the honest default `exposed: false` when nothing is. | "Nothing is exposed" is itself a finding (it bounds what evaluation can claim); recording it now avoids a silent gap later. The only field with a real cost if pursued (logprob plumbing), which is why it is optional-by-declaration, not assumed. |
| **Per-step cost** | Wall-clock, tokens (prompt/completion), tool-call count. | Turns "N % of steps were redundant" into "N % of the budget was" — the decomposable pace bar's denominator. |

## v0.1 extensions and consumer requirements

The v0.1 schema includes producer-driven extensions exercised by the local experiment.
Validation against it proves this producer's conformance to that schema, not agreement with or
compatibility with an existing AII consumer. In particular, the earlier v0 schema rejects the new
`ChoiceKind: decision` value and nullable tool arguments. Execution envelopes also require consumers
to distinguish decision linkage from physical provider calls when counting steps and costs.
Optional metadata alone does not make these changes compatible with every existing consumer.

The AII handoff below must agree consumer support for the experimental v0.1 profile and its
field semantics. Existing exports retain their recorded version and
meaning; do not relabel or reinterpret them retrospectively. No AII integration is claimed here.

### Participant roles and tool execution

The v0.1 `agent_role` enum identifies `actor`, `critic`, `reader` and `discovery` calls.
The orchestrator produces these records; its dashboard, model/cache accounting and inspection
consume them. Discovery calls count in all-model totals, separately from actor context growth.
Local producer and consumer tests cover the profile; external AII acceptance remains unverified.

`tool_kind` records the NEST-owned dispatcher. Every model-visible tool has the same operation,
replay and checkpoint lifecycle; effect metadata distinguishes reads from physical effects.

In the current experimental producer, `AgentStepRecord.agent_role` distinguishes `actor` and
`critic` calls. Both retain a provider-visible request receipt and public response record before a
trace line is materialized. The critic's review record is a separate domain projection carrying the
review trigger and typed verdict; this storage arrangement is not required of other producers.

A provider may emit a tool call whose argument text is malformed even when tools have declared
schemas. Such a call was never dispatched: its trace record has `status: skipped`, `args: null`,
and an `input_hash` that includes the original argument string under the recorded hash policy.
The complete provider response remains in
`raw_output`; consumers must not mistake `null` for an empty argument object. Schema-invalid
objects retain their parsed `args` and are marked skipped with a validation error when no tool
operation was issued. Unadvertised tool names remain visible as model-selected candidates rather
than disappearing from the choice point.

The `reader` role identifies bounded document-reader inference.
The orchestrator produces these records; its dashboard, cache accounting and trace inspection
consume them. Reader requests use the same input/response, cost and optional cache fields as
actor/critic requests. Consumers should count reader calls in all-model totals and keep them
separate from the actor's history-size chart. Historical actor/critic records remain valid.

### Decisions, timing and cost

The experimental `ChoiceKind` also permits `decision`: an emitted structured control/review
decision such as `wait_requested`, `completion_requested`, `accept`, or `revise`. Its descriptor
names the decision; it does not assert mission termination. Consumers must not interpret every
response without a tool call as `halt`. Unclassified text is explicitly `unclassified_response`.

The orchestrator records `wall_clock_start` and `latency_ms` from context-receipt persistence to
response-journal persistence, when both timestamps exist. This includes request-envelope overhead;
it is not pure provider latency or the duration of subsequent tool execution. Candidate metadata
identifies this timing basis and `context_receipt_id`, whose PostgreSQL receipt contains the full
instructions, settings and tool schemas. Missing historical timing stays absent, not measured zero.
Cost prefers a valid provider-reported charge, then normalized response usage cost; unknown remains null.
Historical persisted trace records are not silently rewritten by this producer change.

### Optional JEV experiment provenance

The orchestrator's opt-in experiment emits one `AgentStepRecord` per physical provider call plus
an execution envelope connecting the decision to normal dispatched tools. `experiment.accounting_scope`
distinguishes `provider_call` from `execution_envelope`. Only provider-call records carry inference
usage/cost; envelopes omit `token_usage` and set cost null to avoid double counting. Records without
this field keep their existing meaning. All identifiers remain event-scoped within the run.

`experiment.decision_id` on an envelope is the prefix of its provider-call event IDs. The optional
experiment object also carries role, resolved configuration, request hash, routing outcome, failure
reason, `usage_exposed` and `cost_basis`. Estimated configured prices are labelled as estimates;
unknowns stay null/absent. Missing tokens are not emitted as measured zero. This extension uses
existing provider data and does not introduce inference solely to populate traces.

Every JEV choice question is a choice point: `question_id` identifies it; candidate `probability`
holds the complete provider distribution; `selected_indices` identifies the recommendation;
`provider_confidence` preserves the provider's distinct confidence signal; and
`probability_semantics` identifies a provider choice distribution. Neither signal is measured
NEST decision accuracy. `uncertainty.exposed` is true when those values are exposed.

Concrete action candidates may put `tool` and fully resolved `args` in `metadata`. General LLM
tool menus carry descriptions and `parameters_json_schema` there, with `candidate_scope: tool_menu`.
Do not attach one invented argument set to a tool-menu candidate. Actual arguments stay on linked
tool-call records, including repeated selections of the same candidate with different arguments.
Raw JEV responses and exact JSON request bodies remain in `raw_output` and `prompt_text`. LLM raw
output is the public Pydantic response serialization, not a claim to preserve the provider HTTP body.

The experiment report consumes these extensions. Other consumers must explicitly support the
envelope distinction before deriving request counts; ignoring the optional fields is insufficient.
Consumers calculating request counts should exclude execution envelopes. Evaluation annotations
remain outside the raw trace, as before.

### Optional prompt-cache telemetry

`token_usage.cache_read_tokens` and `cache_write_tokens` are optional nonnegative integer
counters already included in `prompt_tokens`; never add them to total usage or cost. Cache reuse
does not reduce context-window occupancy. Omission means unavailable; a reported zero means zero.
`cache_usage_source=provider` retains provider field presence. `sdk_positive_counts` identifies
legacy positive SDK-normalized counters: SDK-default zeros cannot prove a cache miss.
The OpenRouter non-streaming adapter preserves presence in `raw_output.provider_details.cache_usage`
(inside the serialized JSON). Streaming currently supports only the conservative positive-counter
fallback. The dashboard reads legacy raw output without changing persisted records, excludes
execution envelopes, and states telemetry coverage beside the covered-input cache-read share.
Prices, discounts and estimated savings must not be inferred from this token ratio. Existing
provider-reported `cost_usd` remains authoritative. Consumers that accept unknown optional fields
may ignore cache metadata; this does not establish compatibility with the other extensions above.

## Examples and schema validation

[`v0/examples/coupling-loop-example.jsonl`](v0/examples/coupling-loop-example.jsonl) — a small **synthetic, hand-authored** trace (12 records; fake hashes, fixed timestamps; validates against `v0/schema.json`). Shaped like a price↔demand coupling loop so every field group appears at least once: the efficient path, a batched second call refused by a one-call budget (both picks on one choice point; the dropped call carries `status: skipped` with its `candidate_index` — drop and queue policies serialize under the same shape), a harness-level failure with a retry (two attempts realizing one candidate), a spawn/return pair, an exact-repeat re-read (the redundancy signature: identical input *and* output hashes), a **parallel decision both of whose calls execute** (one choice point, `selected_indices: [1, 6]`, each call back-linked), and a final response that picks `damp` and `HALT` together against a converged residual (`selected_indices: [3, 7]`, the call linked to 3). No record here comes from a real run. [`v0/examples/invalid-records.jsonl`](v0/examples/invalid-records.jsonl) is the negative fixture — four malformed records (unknown `record_type`; missing required field; out-of-range value; a pre-v0 single-`selected_index` choice point) that a conforming validator must reject. To check both:

```python
import json, jsonschema
schema = json.load(open("v0/schema.json"))
for ln in open("v0/examples/coupling-loop-example.jsonl"):
    jsonschema.validate(json.loads(ln), schema)          # all pass
for ln in open("v0/examples/invalid-records.jsonl"):
    try: jsonschema.validate(json.loads(ln), schema); raise SystemExit("accepted invalid record")
    except jsonschema.ValidationError: pass              # all rejected
```

## Current output-hash semantics and AII follow-up

**Implemented producer policy, 2026-10-22:** newly projected tool-call records carry
`hash_policy: tool-outcome/v1`, implementing the preimages agreed with
[Holly on 21 October](https://openearthfoundation.slack.com/archives/D0BSTRKCDFG/p1792613798294679).
This is a new version of the hash contract within the v0.1 record profile, not a reinterpretation
of unmarked hashes. The v0 schema and historical examples remain unchanged.

| Field | `tool-outcome/v1` preimage |
|---|---|
| `input_hash` | `{"tool": tool_name, "args": parsed_arguments}`. If argument parsing fails, use the original provider argument string as `args` in this preimage; the record's `args` stays null. |
| `output_hash` | `{"status": status, "result": result, "error": error_message}` for an executed success or failure, including a null result. A skipped/unsettled call has no output hash (`null`). |

Both hashes are lowercase hexadecimal SHA-256 over UTF-8 bytes of JSON serialized with sorted
object keys, compact separators (no added whitespace), and `ensure_ascii=False`. Non-ASCII
characters remain literal; no Unicode normalization or other payload transformation is applied.
Object key order does not affect a hash; array order and nested result fields do.
The `error` key in the preimage maps to the record's `error_message` field, including null.
All output fields outside this preimage, such as trace timestamps and attempt counts, are excluded.

[Shared hash vectors](v0.1/examples/tool-outcome-v1-hashes.json) give exact JSON strings and expected
hashes for Unicode content, null success/failure results, errors, skipped calls and malformed
arguments. Consumers can use them without a model call or the orchestrator runtime. A consumer
must honor the recorded policy when recomputing; producer/consumer compatibility for this new
policy has not yet been independently verified.

Historical unmarked orchestrator calls used `digest(args)` (or the original malformed argument
string) and `digest(result)`, with no output hash for a null result. Stored records and exports
retain those hashes; an export only serializes saved records. Pending responses first projected
by the new code receive the new policy, so a continued run may contain both policies. Absence of
`hash_policy` is not permission to apply `tool-outcome/v1` retrospectively. Other producers must
declare their own historical semantics. Runtime operation/recovery identities and the internal
progress guard's hashes are unchanged.

**Open follow-up:** this policy does not remove nested volatile fields. Identical substantive
content with different timings inside `result` still hashes differently. Agree per-tool comparison
rules, potentially using a separate versioned comparison digest alongside the full outcome hash.
Schema validation alone cannot establish substantive-result equivalence.

As requested in [Daniel's integration review](https://github.com/Open-Earth-Foundation/NEST-workspace/pull/151#issuecomment-6001277951),
validate an actual exported trace through the AII consumer and agree explicit, versioned per-tool
comparison rules before using hash-based repeats for EXP-05 or pace evidence. Do not silently
strip arbitrary fields, reinterpret historical hashes, or change recovery/idempotency identities.
The producer and consumer owners must agree compatibility and any contract version/field changes.

The [AII trace-integration backlog](../../../services/agent-orchestrator/docs/follow-ups/trace-integration.md)
owns the concrete steps and acceptance checks, including separate model-call, physical-execution
and monetary accounting. Nested full-payload comparison behavior is unchanged; existing saved traces retain
their original recorded versions and hashes.

## Evaluation boundary

Native model I/O formats stay with their components (per this package's README). This contract carries no evaluation labels — progress/exploration/redundant are a *consumer's* output (EXP-05), never written back into the trace. Nothing here requires a specific agent framework; the fields are harness-agnostic.

## Provenance

Field list: #74 (2026-08); refined against a twelve-arm prototype study (AII, Aug 2026 — scripted and live agents driving a real MESSAGEix ↔ building-stock coupling loop) in which every field above earned its place either by being load-bearing for the evaluation or by its absence having caused a measured failure. Full write-ups available on request.
