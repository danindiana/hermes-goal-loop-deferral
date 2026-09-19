<p align="center">
  <img src="assets/logo.svg" alt="hermes-goal-loop-deferral logo" width="480">
</p>

<p align="center">
  <img alt="license" src="https://img.shields.io/badge/license-MIT-blue.svg">
  <img alt="platform" src="https://img.shields.io/badge/platform-Linux-informational">
  <img alt="made-with-hermes" src="https://img.shields.io/badge/made%20with-Hermes%20Agent-8b5cf6">
  <img alt="made-with-ollama" src="https://img.shields.io/badge/made%20with-Ollama-000000">
  <img alt="diagrams" src="https://img.shields.io/badge/diagrams-30%20%C3%97%202%20formats-orange">
  <img alt="rendered-with" src="https://img.shields.io/badge/rendered%20with-Graphviz-2e8b57">
  <a href="https://github.com/danindiana/hermes-goal-loop-deferral/actions/workflows/verify-diagrams.yml"><img alt="CI" src="https://github.com/danindiana/hermes-goal-loop-deferral/actions/workflows/verify-diagrams.yml/badge.svg"></a>
  <img alt="last-commit" src="https://img.shields.io/github/last-commit/danindiana/hermes-goal-loop-deferral">
  <img alt="repo-size" src="https://img.shields.io/github/repo-size/danindiana/hermes-goal-loop-deferral">
</p>

# hermes-goal-loop-deferral

[Hermes Agent](https://hermes-agent.nousresearch.com/)'s `/goal` command gives it a standing
objective and a judge model that decides, after every turn, whether to keep going. Watching it
run against a locally-loaded [Ollama](https://ollama.com/) model, the loop looked flaky — long
stretches where a turn would finish and the loop just... wouldn't continue, with no obvious
error. This repo is the actual diagnosis, sourced entirely from real agent logs: the judge was
never broken. A separate post-turn hook was silently skipping the judge call under four specific
conditions, and the locally-loaded model was hitting those conditions far more often than its
peers.

Nothing here is hypothetical. Every claim is sourced from real `agent.log` output — verdicts,
turn boundaries, interruption events, and stream-drop messages — not a synthetic reproduction.

## Contents

- [The symptom](#the-symptom)
- [The judge is not the problem](#the-judge-is-not-the-problem)
- [Root cause: four silent deferral paths](#root-cause-four-silent-deferral-paths)
- [Why one model hit it hardest](#why-one-model-hit-it-hardest)
- [Mitigation proposals](#mitigation-proposals)
- [Diagrams](#diagrams)
- [Documentation](#documentation)
- [Rollouts](#rollouts)
- [Repo structure](#repo-structure)
- [License](#license)

## The symptom

`/goal` sets a standing objective and, after each turn, is supposed to call a judge model that
returns one of `done` / `continue` / `blocked` / `wait`. On `continue`, a continuation prompt is
automatically re-queued into the same session, and the loop keeps going without the user typing
"keep going" again.

In practice, long-running goal sessions against a locally-loaded model showed turns completing
with substantial, real responses — and then *nothing*. No judge verdict logged, no continuation
prompt queued, no pause message. The loop would eventually resume a turn or two later, as if
nothing had happened, which made the behavior read as random rather than deterministic.

## The judge is not the problem

The judge itself — a separate, fast auxiliary model configured independently from the main loaded
model — resolved correctly and consistently across every session checked, days apart, with
substantive one-line rationales:

```
INFO agent.auxiliary_client: Auxiliary goal_judge: using <auxiliary provider/model>
INFO hermes_cli.goals: goal judge: verdict=continue reason=The agent is executing code to compute
  metrics from benchmark results, but the execution is incomplete (truncated output)...
```

```
INFO hermes_cli.goals: goal judge: verdict=blocked reason=The goal statement is too vague to be
  actionable; the agent correctly identified that clarification is needed...
```

One hard failure did turn up in an older log window — the judge's provider was unresolvable and no
fallback chain was configured — but that predates an explicit provider pin added to config since,
and has not recurred in any log window checked after that change.

## Root cause: four silent deferral paths

The actual gate is a post-turn hook that runs after every turn and decides *whether to call the
judge at all*. Four conditions each cause it to return early — silently, with no judge call and
(in most cases) no visible message beyond an easily-missed status line:

1. **Interrupted / stale-stream turns.** If the turn was interrupted (a cancelled generation, a
   stream the client had to force-close), the hook doesn't judge — it auto-pauses the goal
   instead. A pause is recoverable, but it looks identical to "the loop just stopped."

2. **Empty or whitespace-only responses.** Skipped outright, by design — judging an empty response
   would almost always come back `continue` and just requeue a no-op. (Note: a *reasoning-only*
   response — the model produced only chain-of-thought and no distinct final answer — is still
   treated as valid text and *does* get judged; only a truly empty/dropped stream is skipped here.)

3. **A queued real user message.** If the user has already typed something new, judging defers
   until after their turn runs — deliberate, since user input should always preempt the loop.

4. **Turns superseded by a periodic background-review nudge.** Hermes periodically injects its own
   maintenance turns — e.g. a skill-library review nudge — into the same session. That injected
   turn is not a goal-continuation turn, so it isn't judged; the goal continuation only resumes on
   the turn *after* the nudge completes or is itself superseded. Reconstructed from a real log
   window:

   ```
   turn_context: ...history=163 msg='proceed with next forty steps...'
   Turn ended: reason=text_response ... response_len=1128
   turn_context: ...history=177 msg='Review the conversation above and update the skill library...'
   auxiliary_client: Auxiliary auto-detect: using main provider (local model)
   Shut down the stale stream's socket to unblock the reader (attempt superseded...)
   Turn ended: reason=interrupted_during_api_call(background_review_superseded)
   turn_context: ...history=177 msg='[Continuing toward your standing goal] ...'
   ```

   No judge call happened for the turn that completed just before the nudge fired — the
   continuation only resumed once the nudge itself got superseded.

Only when none of these four apply does the hook actually call the judge.

## Why one model hit it hardest

Counting these events across a real multi-day log window makes the pattern obvious: one locally
loaded model accounted for the overwhelming majority of both interrupted turns and
reasoning-only stalls, far above every other model in the same environment.

| Event | Loaded model (heaviest hit) | Next 4 models combined |
|---|---|---|
| Interrupted / stale-stream turns | **55** | 15 |
| Reasoning-only clean stops | **87** | 13 |

Recurring signature right before an interruption:

```
agent.chat_completion_helpers: Shut down the stale stream's socket to unblock the reader
  (attempt superseded; model=<loaded model>).
```

Sessions inspected show this model running deep into its context window (well over half of a
large configured context length, with 94–99% prompt-cache hit rates) during long goal loops —
consistent with the model running under context/VRAM pressure at length, and/or a
concurrent background-review call on the same model contending for the same stream and
superseding an in-flight goal-continuation response.

## Mitigation proposals

Not applied here — documented as concrete next steps:

- **Space out background-review nudges relative to active goals.** Raising the skill-library and
  memory nudge intervals reduces how often a nudge turn collides with a goal-continuation turn.
  An open question for a deeper fix: whether the nudge scheduler could skip firing entirely while
  a goal is active, rather than just spacing the interval out.
- **Investigate the loaded model's stream-drop rate specifically.** A model tag tuned to fit
  entirely in VRAM may leave less headroom under sustained long-context goal loops than an
  equivalent tag without that tuning. Worth comparing interruption rate on the same workload
  across both variants, watching live VRAM/context usage during a long run.
- **No action needed, but worth re-checking:** the one auxiliary-provider failure found in an
  older log window appears already resolved by an explicit provider pin in config — flag it if it
  ever recurs, rather than treating it as still-open.

## Diagrams

| # | Diagram | What it shows |
|---|---|---|
| 01 | [`goal_loop_architecture`](diagrams/01_goal_loop_architecture.svg) | The `/goal` loop end to end, with the main loaded model and the auxiliary judge model as two architecturally separate boxes |
| 02 | [`deferral_decision_flow`](diagrams/02_deferral_decision_flow.svg) | The post-turn hook's decision tree — four skip paths before the one path that calls the judge |
| 03 | [`nudge_collision_sequence`](diagrams/03_nudge_collision_sequence.svg) | The reconstructed timeline where a skill-library nudge swallows a judge call |
| 04 | [`model_interruption_comparison`](diagrams/04_model_interruption_comparison.svg) | Interrupted-turn and reasoning-stall counts by model, with the heaviest-hit model flagged |
| 05 | [`catch22`](diagrams/05_catch22.svg) | The tension between reducing nudge collisions and preserving what the nudges protect |
| 06 | [`howto`](diagrams/06_howto.svg) | Step-by-step flowchart for diagnosing a stalled goal loop |
| 07 | [`technical_rationale`](diagrams/07_technical_rationale.svg) | Design goal → mechanism → tradeoff for each deferral path |
| 08 | [`why_this_repo_exists`](diagrams/08_why_this_repo_exists.svg) | This repo's scope relative to Hermes Agent core and sibling writeup repos |
| 09 | [`known_unknowns`](diagrams/09_known_unknowns.svg) | Confirmed vs. inferred vs. genuinely open questions |
| 10 | [`future_directions`](diagrams/10_future_directions.svg) | Near- to longer-term follow-up roadmap |
| 11 | [`meta_loop_integrations`](diagrams/11_meta_loop_integrations.svg) | How `/goal` relates to `/loop`, Kanban goal-mode cards, and heartbeat |
| 12 | [`api_socket_connectors`](diagrams/12_api_socket_connectors.svg) | The client/socket sequence behind a stream supersession |
| 13 | [`glossary`](diagrams/13_glossary.svg) | Core terms and how they relate |
| 14 | [`threat_model`](diagrams/14_threat_model.svg) | Likelihood × impact matrix for the known failure modes |
| 15 | [`faq`](diagrams/15_faq.svg) | Wayfinding map from common questions to the doc that answers them |
| 16 | [`deferral_telemetry_spec`](diagrams/16_deferral_telemetry_spec.svg) | Proposed log line at each of the four deferral points |
| 17 | [`cli_gateway_hook_parity_audit`](diagrams/17_cli_gateway_hook_parity_audit.svg) | CLI vs. gateway hook: shared, differs, and unconfirmed |
| 18 | [`vram_tag_comparison_protocol`](diagrams/18_vram_tag_comparison_protocol.svg) | Experiment protocol: VRAM-tuned tag vs. plain tag |
| 19 | [`goal_aware_nudge_scheduler_design`](diagrams/19_goal_aware_nudge_scheduler_design.svg) | Proposed nudge scheduler state machine |
| 20 | [`goal_status_deferral_counter_mockup`](diagrams/20_goal_status_deferral_counter_mockup.svg) | Before/after mockup of a `/goal status` deferral counter |
| 21 | [`background_review_subsystem_deep_dive`](diagrams/21_background_review_subsystem_deep_dive.svg) | Corrected mechanism: the nudge gets cancelled, not the goal turn |
| 22 | [`gateway_interrupted_turn_gap`](diagrams/22_gateway_interrupted_turn_gap.svg) | Confirmed: gateway judges partial output from interrupted turns |
| 23 | [`deferral_rate_baseline_methodology`](diagrams/23_deferral_rate_baseline_methodology.svg) | A reusable rate-based comparison methodology |
| 24 | [`kanban_goal_mode_worker_session_audit`](diagrams/24_kanban_goal_mode_worker_session_audit.svg) | Corrected: Kanban goal-mode judges at handoff, not per turn |
| 25 | [`completion_contract_effectiveness_review`](diagrams/25_completion_contract_effectiveness_review.svg) | Which contract fields address which false-done risk factor |
| 26 | [`gateway_queued_message_check`](diagrams/26_gateway_queued_message_check.svg) | Same guarantee, different mechanism: queued real user message |
| 27 | [`nudge_interval_source_reading`](diagrams/27_nudge_interval_source_reading.svg) | Skill nudge counts iterations; memory nudge counts turns |
| 28 | [`heartbeat_collision_check`](diagrams/28_heartbeat_collision_check.svg) | Heartbeat's explicit busy-check, and one plausible narrow gap |
| 29 | [`rollouts_process_retrospective`](diagrams/29_rollouts_process_retrospective.svg) | Tally of 15 rollouts by kind: audits, corrections, design specs |
| 30 | [`contract_drafting_prompt_review`](diagrams/30_contract_drafting_prompt_review.svg) | What the contract-drafting model is asked, and what it can't see |

Each diagram ships as `.dot` (source), `.png`, and `.svg`. Re-render any of them with:

```bash
dot -Tpng -Gdpi=160 diagrams/01_goal_loop_architecture.dot -o diagrams/01_goal_loop_architecture.png
dot -Tsvg diagrams/01_goal_loop_architecture.dot -o diagrams/01_goal_loop_architecture.svg
```

CI (`.github/workflows/verify-diagrams.yml`) re-renders every `.dot` on push/PR and fails if the
committed SVG has drifted from its source.

## Documentation

Each doc below covers one angle in more depth than the README does, and carries its own dedicated
diagram (05–15 in the table above continue from these, in the same order):

| Doc | What it covers |
|---|---|
| [`docs/catch22.md`](docs/catch22.md) | Why the obvious fix (space out the nudges) trades off against the reason the nudges exist |
| [`docs/howto.md`](docs/howto.md) | Step-by-step: telling a silently-deferred goal from a genuinely finished one |
| [`docs/technical_rationale.md`](docs/technical_rationale.md) | Why each deferral path is a deliberate design choice, not a bug |
| [`docs/why_this_repo_exists.md`](docs/why_this_repo_exists.md) | Why this is documentation (not a patch) and its own repo (not a subfolder) |
| [`docs/known_unknowns.md`](docs/known_unknowns.md) | What's confirmed in logs vs. inferred vs. genuinely still open |
| [`docs/future_directions.md`](docs/future_directions.md) | Concrete near-, mid-, and longer-term follow-ups |
| [`docs/meta_loop_integrations.md`](docs/meta_loop_integrations.md) | How `/goal` relates to `/loop`, Kanban goal-mode cards, and heartbeat |
| [`docs/API_socket_connectors.md`](docs/API_socket_connectors.md) | The client/socket mechanics behind the "stale stream, superseded" log line |
| [`docs/glossary.md`](docs/glossary.md) | Core terms and how they relate to each other |
| [`docs/threat_model.md`](docs/threat_model.md) | Operational/reliability risk matrix (explicitly not a security threat model) |
| [`docs/faq.md`](docs/faq.md) | Short Q&A, each answer linking to the doc with the full version |

## Rollouts

Follow-up work beyond the core diagnosis was tracked as an explicit, auditable, MCTS-style
exploration process in [`docs/ROLLOUTS.md`](docs/ROLLOUTS.md): 3 rounds of 5 candidates each,
one picked and completed at a time, informed by what the prior round actually found. **Complete:
15/15 rollouts**, including 2 real corrections to earlier docs in this repo, found by rollouts
that audited source code the original diagnosis hadn't read. See that ledger for every candidate,
every pick's reasoning, and links to all fifteen finished pieces in
[`docs/rollouts/`](docs/rollouts/).

## Repo structure

```
.
├── assets/
│   └── logo.svg
├── diagrams/
│   ├── 01_goal_loop_architecture.{dot,png,svg}
│   ├── 02_deferral_decision_flow.{dot,png,svg}
│   ├── 03_nudge_collision_sequence.{dot,png,svg}
│   ├── 04_model_interruption_comparison.{dot,png,svg}
│   ├── 05_catch22.{dot,png,svg}
│   ├── 06_howto.{dot,png,svg}
│   ├── 07_technical_rationale.{dot,png,svg}
│   ├── 08_why_this_repo_exists.{dot,png,svg}
│   ├── 09_known_unknowns.{dot,png,svg}
│   ├── 10_future_directions.{dot,png,svg}
│   ├── 11_meta_loop_integrations.{dot,png,svg}
│   ├── 12_api_socket_connectors.{dot,png,svg}
│   ├── 13_glossary.{dot,png,svg}
│   ├── 14_threat_model.{dot,png,svg}
│   ├── 15_faq.{dot,png,svg}
│   ├── 16_deferral_telemetry_spec.{dot,png,svg}
│   ├── 17_cli_gateway_hook_parity_audit.{dot,png,svg}
│   ├── 18_vram_tag_comparison_protocol.{dot,png,svg}
│   ├── 19_goal_aware_nudge_scheduler_design.{dot,png,svg}
│   ├── 20_goal_status_deferral_counter_mockup.{dot,png,svg}
│   ├── 21_background_review_subsystem_deep_dive.{dot,png,svg}
│   ├── 22_gateway_interrupted_turn_gap.{dot,png,svg}
│   ├── 23_deferral_rate_baseline_methodology.{dot,png,svg}
│   ├── 24_kanban_goal_mode_worker_session_audit.{dot,png,svg}
│   ├── 25_completion_contract_effectiveness_review.{dot,png,svg}
│   ├── 26_gateway_queued_message_check.{dot,png,svg}
│   ├── 27_nudge_interval_source_reading.{dot,png,svg}
│   ├── 28_heartbeat_collision_check.{dot,png,svg}
│   ├── 29_rollouts_process_retrospective.{dot,png,svg}
│   └── 30_contract_drafting_prompt_review.{dot,png,svg}
├── docs/
│   ├── ROLLOUTS.md
│   ├── catch22.md
│   ├── howto.md
│   ├── technical_rationale.md
│   ├── why_this_repo_exists.md
│   ├── known_unknowns.md
│   ├── future_directions.md
│   ├── meta_loop_integrations.md
│   ├── API_socket_connectors.md
│   ├── glossary.md
│   ├── threat_model.md
│   ├── faq.md
│   └── rollouts/
│       ├── deferral_telemetry_spec.md
│       ├── cli_gateway_hook_parity_audit.md
│       ├── vram_tag_comparison_protocol.md
│       ├── goal_aware_nudge_scheduler_design.md
│       ├── goal_status_deferral_counter_mockup.md
│       ├── background_review_subsystem_deep_dive.md
│       ├── gateway_interrupted_turn_gap.md
│       ├── deferral_rate_baseline_methodology.md
│       ├── kanban_goal_mode_worker_session_audit.md
│       ├── completion_contract_effectiveness_review.md
│       ├── gateway_queued_message_check.md
│       ├── nudge_interval_source_reading.md
│       ├── heartbeat_collision_check.md
│       ├── rollouts_process_retrospective.md
│       └── contract_drafting_prompt_review.md
├── .github/workflows/verify-diagrams.yml
├── LICENSE
└── README.md
```

## License

[MIT](LICENSE)
