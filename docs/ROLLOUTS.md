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
   concretely-named open question rather than proposing something new. — Status: ✅ [`docs/rollouts/cli_gateway_hook_parity_audit.md`](rollouts/cli_gateway_hook_parity_audit.md) — found a real CLI/gateway difference (interrupted-turn check) and refined the "4 deferral paths" framing (nudge collision is a subsystem race, not a hook branch)
3. **Selected: `vram_tag_comparison_protocol`.** Reason: the leading unresolved hypothesis in
   `known_unknowns.md`; independent of the two remaining design-spec items. — Status: ✅ [`docs/rollouts/vram_tag_comparison_protocol.md`](rollouts/vram_tag_comparison_protocol.md)
4. **Selected: `goal_aware_nudge_scheduler_design`.** Reason: the more load-bearing of the two
   remaining design specs — it's the actual resolution `catch22.md` names, whereas the counter
   mockup is a visibility nicety on top of whatever telemetry format rollout 1 defines. — Status: ✅ [`docs/rollouts/goal_aware_nudge_scheduler_design.md`](rollouts/goal_aware_nudge_scheduler_design.md)
5. **Selected: `goal_status_deferral_counter_mockup`.** Reason: last by elimination, and
   naturally builds on rollout 1's telemetry vocabulary, which is now available. — Status: ✅ [`docs/rollouts/goal_status_deferral_counter_mockup.md`](rollouts/goal_status_deferral_counter_mockup.md)

**Round 1 complete: 5/5.** Two organic findings surfaced during execution that weren't
anticipated when the candidates were proposed: (a) the "4 deferral paths" framing in the core
docs is a simplification — the nudge-collision path is actually a race with a separate
background-review subsystem, not a fourth symmetric branch in either hook (found while auditing
rollout 2); (b) the CLI's interrupted-turn auto-pause isn't confirmed to exist on the gateway
surface (also rollout 2) — a genuine, previously-unflagged gap. Round 2 below picks up both.

## Round 2

Candidates informed by what round 1 actually turned up — particularly the two findings from
`cli_gateway_hook_parity_audit` above, which round 1 didn't have when it was proposed:

| # | Slug | Extends | One-line pitch |
|---|---|---|---|
| 1 | `background_review_subsystem_deep_dive` | Round 1 finding (a) | What the background-review subsystem actually is, and why it races goal continuations at all |
| 2 | `gateway_interrupted_turn_gap` | Round 1 finding (b) | Follow the gateway's turn-completion path further to confirm/deny the missing interrupted-turn check |
| 3 | `completion_contract_effectiveness_review` | `../technical_rationale.md` / `../threat_model.md` | Does a completion contract measurably reduce the false-done risk named in `threat_model.md`? A review of the mechanism, not a live test |
| 4 | `kanban_goal_mode_worker_session_audit` | `../meta_loop_integrations.md` open item | Whether a Kanban `--goal` card's worker session hits the same 4(ish) deferral paths as an interactive CLI session |
| 5 | `deferral_rate_baseline_methodology` | `vram_tag_comparison_protocol` (round 1) | A general measurement methodology (deferral rate, not raw count) reusable across any future comparison, not just the VRAM-tag one |

### Picks

1. **Selected: `background_review_subsystem_deep_dive`.** Reason: the most foundational of the
   five — several other round 2 candidates and round 1's own refinement both lean on claims about
   this subsystem that were inferred, not directly read from its own source. Settling this first
   makes the others more precise. — Status: ✅ [`docs/rollouts/background_review_subsystem_deep_dive.md`](rollouts/background_review_subsystem_deep_dive.md) — **found and fixed a real error**: `API_socket_connectors.md` (round 0) had the cancellation roles backwards; the background review's own turn gets cancelled by the next live turn, never the reverse. Correction note added to that doc.
2. **Selected: `gateway_interrupted_turn_gap`.** Reason: directly closes a named, real gap from
   round 1 rather than opening a new one — highest-priority unfinished thread. — Status: ✅ [`docs/rollouts/gateway_interrupted_turn_gap.md`](rollouts/gateway_interrupted_turn_gap.md) — **confirmed**: the gateway hook judges partial output from interrupted turns; the CLI never does.
3. **Selected: `deferral_rate_baseline_methodology`.** Reason: independent of the other four,
   and generalizes round 1's protocol work into something reusable rather than one-off. — Status: ✅ [`docs/rollouts/deferral_rate_baseline_methodology.md`](rollouts/deferral_rate_baseline_methodology.md)
4. **Selected: `kanban_goal_mode_worker_session_audit`.** Reason: same shape as round 1's CLI/
   gateway audit, applied to the third surface `../meta_loop_integrations.md` named but didn't
   check — natural continuation once two of three surfaces are covered. — Status: ✅ [`docs/rollouts/kanban_goal_mode_worker_session_audit.md`](rollouts/kanban_goal_mode_worker_session_audit.md) — **found and fixed another error**: Kanban goal-mode does NOT share `GoalManager`'s per-turn hook; it judges once at handoff via a separate function. Correction added to `meta_loop_integrations.md`.
5. **Selected: `completion_contract_effectiveness_review`.** Reason: last by elimination; most
   speculative of the five since it reviews a mechanism's design rather than auditing code or
   proposing a new one. — Status: ✅ [`docs/rollouts/completion_contract_effectiveness_review.md`](rollouts/completion_contract_effectiveness_review.md)

**Round 2 complete: 5/5 (10/15 overall).** This round ran heavier on source-grounded audits than
round 1, and found and fixed two real errors in earlier docs: the background-review cancellation
mechanism ([`API_socket_connectors.md`](API_socket_connectors.md) had the roles backwards) and the
Kanban goal-mode judging path ([`meta_loop_integrations.md`](meta_loop_integrations.md) assumed
too much shared machinery with `/goal`). It also fully confirmed the gateway's interrupted-turn
gap that round 1 could only flag as open.

## Round 3

Candidates informed by round 2 — two threads deliberately left open there, plus natural
next steps once two of three surfaces (CLI, gateway) and Kanban are now characterized:

| # | Slug | Extends | One-line pitch |
|---|---|---|---|
| 1 | `gateway_queued_message_check` | `cli_gateway_hook_parity_audit` (round 1), last open parity row | Read the gateway's queued-user-message handling to close the one remaining parity-table row |
| 2 | `nudge_interval_source_reading` | `background_review_subsystem_deep_dive` (round 2) | What actually decides a nudge is "due" — the interval logic itself, not yet read |
| 3 | `heartbeat_collision_check` | `../threat_model.md` / `../meta_loop_integrations.md` | Does heartbeat's own idle-wake mechanism collide with `/goal` the same way background-review nudges do? |
| 4 | `rollouts_process_retrospective` | This ledger itself | A retrospective on the MCTS-style process after 2 rounds: what worked, what the picks got right/wrong, sourced from this ledger's own record |
| 5 | `contract_drafting_prompt_review` | `completion_contract_effectiveness_review` (round 2) | What the `goal_judge` auxiliary model is actually asked, when drafting a contract via `/goal draft` — read the prompt, not just the resulting fields |

### Picks

1. **Selected: `gateway_queued_message_check`.** Reason: the single most concrete unfinished
   thread across both prior rounds — a named, specific gap with a clear yes/no answer available
   directly from source, same shape as the round 2 rollout that closed the interrupted-turn row.
   — Status: ✅ [`docs/rollouts/gateway_queued_message_check.md`](rollouts/gateway_queued_message_check.md) — closes the CLI/gateway parity table's last row: both surfaces achieve the same guarantee via genuinely different mechanisms.
2. **Selected: `nudge_interval_source_reading`.** Reason: the deep dive in round 2 explicitly
   flagged this as unread and load-bearing for
   `goal_aware_nudge_scheduler_design` (round 1) actually being implementable — closes a
   dependency the design spec was written without. — Status: ⬜
3. **Selected: `heartbeat_collision_check`.** Reason: independent of the other four, and the
   most natural remaining "does this other mechanism collide too" question left after nudges and
   Kanban have both been checked. — Status: ⬜
4. **Selected: `rollouts_process_retrospective`.** Reason: two full rounds is enough material for
   an honest retrospective on the process itself, and it's a good anchor before round 3's other,
   more technical items. — Status: ⬜
5. **Selected: `contract_drafting_prompt_review`.** Reason: last by elimination; narrowest scope
   of the five and most directly a follow-up to a single round 2 rollout rather than a broader
   thread. — Status: ⬜
