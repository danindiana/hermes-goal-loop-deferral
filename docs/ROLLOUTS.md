# Rollouts ledger

An MCTS-style exploration log for follow-up work on this repo: each round proposes 5 candidate
follow-ups, one is selected and completed, then the process returns to the remaining candidates
and picks again until all 5 are done — then a new round of 5 is generated. Three rounds total (15
completed rollouts). Each completed rollout is a doc in [`docs/rollouts/`](rollouts/) with its own
diagram in [`diagrams/`](../diagrams/).

This ledger is the audit trail: every candidate proposed, the one-line reason each pick was made,
and a link once it's done.

## Round 1

Candidates proposed, each extending an open thread from the core docs (`future_directions.md`,
`known_unknowns.md`, `catch22.md`, `meta_loop_integrations.md`):

| # | Slug | Extends | One-line pitch |
|---|---|---|---|
| 1 | `deferral_telemetry_spec` | `future_directions.md` mid-term item | Concrete log-line format for each of the 4 deferral paths, currently silent |
| 2 | `cli_gateway_hook_parity_audit` | `known_unknowns.md` open question | Side-by-side audit: does the gateway hook share the CLI hook's 4 gaps? |
| 3 | `vram_tag_comparison_protocol` | `known_unknowns.md` leading hypothesis | Runnable experiment design: VRAM-tuned vs. plain model tag, same workload |
| 4 | `goal_aware_nudge_scheduler_design` | `catch22.md` / `future_directions.md` mid-term item | Design spec for the nudge scheduler resolution catch22.md points toward |
| 5 | `goal_status_deferral_counter_mockup` | `future_directions.md` longer-term item | Before/after mockup of a deferral counter in `/goal status` output |

### Picks

1. **Selected: `deferral_telemetry_spec`.** Reason: highest standalone value — every other
   rollout in this round either depends on being able to *observe* deferrals more precisely
   (the audit, the experiment) or is easier to specify well once the telemetry format exists as a
   reference vocabulary. No dependency on anything undone. — Status: ✅ [`docs/rollouts/deferral_telemetry_spec.md`](rollouts/deferral_telemetry_spec.md)
2. **Selected: `cli_gateway_hook_parity_audit`.** Reason: fully source-grounded (reads existing
   code, no new mechanism to design), independent of the other three remaining, and closes a
   concretely-named open question rather than proposing something new. — Status: ⬜
3. **Selected: `vram_tag_comparison_protocol`.** Reason: the leading unresolved hypothesis in
   `known_unknowns.md`; independent of the two remaining design-spec items. — Status: ⬜
4. **Selected: `goal_aware_nudge_scheduler_design`.** Reason: the more load-bearing of the two
   remaining design specs — it's the actual resolution `catch22.md` names, whereas the counter
   mockup is a visibility nicety on top of whatever telemetry format rollout 1 defines. — Status: ⬜
5. **Selected: `goal_status_deferral_counter_mockup`.** Reason: last by elimination, and
   naturally builds on rollout 1's telemetry vocabulary, which is now available. — Status: ⬜

## Round 2

_Generated after Round 1 completes._

## Round 3

_Generated after Round 2 completes._
