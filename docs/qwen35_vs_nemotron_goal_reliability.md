# Why `qwen3.5:9b-vram-fit` seems to "miss" the goal-continuation nudge while `nemotron-3.5-lightning:1m` catches it reliably

> **Update (live-confirmed, same evening): `/reasoning none` eliminates the failure mode
> entirely.** See [the section below](#reasoning-none-eliminates-the-failure-mode-live-confirmed)
> — 0 reasoning-only stalls across 40+ tool turns with reasoning off, vs. 8 stalls in ~40 minutes
> of the same live session with it on. This is the strongest, most actionable finding on this
> page.

**Question:** operating both models side by side, `qwen3.5:9b-vram-fit` unreliably picks up
`/goal`'s continuation nudge, while `nemotron-3.5-lightning:1m` catches it reliably. Why?

**Scope note:** this is empirical log analysis of the model/sampling layer, distinct from — and
complementary to — the interrupt-misattribution fix this repo's [Fix](../README.md#fix) section
documents. That fix addresses plumbing that mishandled a genuine interrupt or a `-q`-mode no-op.
This page is about something upstream of that plumbing: what the *model itself* does under load,
independent of whether the CLI/loop code around it is correct.

## Bottom line

**The raw interruption rate does not explain the perceived reliability gap.** In the fairest
comparison window (from the day `qwen3.5:9b-vram-fit` was created, and after an unrelated
`OLLAMA_MAX_LOADED_MODELS` thrashing bug that had been corrupting nemotron's earlier numbers was
fixed), `nemotron-3.5-lightning:1m` actually has a *higher* raw turn-interruption rate than
`qwen3.5:9b-vram-fit` (19.2% vs 13.8%). If "does the turn get interrupted at all" were the whole
story, nemotron should look *less* reliable, not more.

**The real differentiator is "reasoning-only clean stops"** — turns where the model produces only
chain-of-thought and never emits a proper final answer, and Hermes's own fallback synthesizes a
response from the raw reasoning text. `qwen3.5:9b-vram-fit` hits this at **23.7%** of its turns in
the same window; `nemotron-3.5-lightning:1m` hits it at **2.6–11.5%** (sample-size caveat below).
That's the actual mechanism an operator would perceive as "the goal nudge doesn't stick" — the
loop keeps technically cycling (the judge still evaluates the synthesized text and still returns a
verdict), but the content is frequently just narrated intent rather than completed work, so from
the outside it looks exactly like the loop isn't really responding to the nudge.

## Method

All numbers below are from real `agent.log` output, grepped directly — not sampled or estimated.
Two log signatures were used:

- `Turn ended: reason=interrupted...` — a turn that didn't complete normally (Ctrl+C, a
  system-issued abort, or a superseded stream).
- `Reasoning-only clean stop (N chars) — returning the reasoning as the final response` — a
  *clean* stop (not an interruption) where the model's stream ended without a distinct final
  answer segment; Hermes's `agent/turn_final_response.py` catches this and manufactures a "final
  response" out of the reasoning text itself so the turn doesn't come back empty. Exact criterion,
  read straight from source: `done_reason == "stop"` AND no `tool_calls` AND `content` is
  empty/whitespace AND the extracted reasoning is non-empty.

The comparison window starts on the day `qwen3.5:9b-vram-fit` (the derived, VRAM-tuned tag) was
created — a same-day investigation fixed a real model-loading thrashing bug that had been
corrupting earlier `nemotron-3.5-lightning:1m` numbers; a naive full-history comparison would have
wrongly attributed that thrashing-era noise to nemotron's ongoing reliability.

## The numbers

| Metric | `qwen3.5:9b-vram-fit` | `nemotron-3.5-lightning:1m` |
|---|---|---|
| Total logged turns (comparison window) | 630 | 52 |
| Interrupted (any reason) | 87 (13.8%) | 10 (19.2%) |
| — of which superseded by a background review | 42 (6.7% of total) | ~0 (negligible in-window) |
| Reasoning-only clean stop | 149 (**23.7%**) | 6 (**11.5%**) |
| Combined "turn didn't produce clean, actionable output" | 236 (37.5%) | 16 (30.8%) |

**Sample-size caveat:** nemotron has only 52 logged turns in this window vs. qwen3.5's 630 —
qwen3.5 has been the default chat model since the window opened, so it has vastly more
opportunities to exhibit any given failure mode. Nemotron's 52-turn sample means its rates carry
wide error bars; treat the *reasoning-only* gap (11.5% vs 23.7%, ~2x) as directionally real but
the exact ratio as soft. The *interruption* rate comparison (19.2% vs 13.8%, nemotron higher) is
the more surprising and arguably more solid finding precisely because it cuts against the
"nemotron is just cleaner" intuition, on the same small sample.

## What's actually different mechanically

Read directly from `ollama show <tag>` and each Modelfile:

| | `qwen3.5:9b-vram-fit` | `nemotron-3.5-lightning:1m` |
|---|---|---|
| Architecture | dense, 9.7B | hybrid MoE, 32.9B |
| Context | 262144 (VRAM-tuned, full native) | 1048576 |
| GPU fit (idle, single model) | 100% GPU (the whole point of the `-vram-fit` tag) | partial CPU spillover even alone |
| `num_batch` | explicit override, added for the vram-fit derivation | default |
| `presence_penalty` | **1.5** | none (Ollama default 0) |
| `thinking` capability | yes | yes |

Two things stand out:

1. **`presence_penalty=1.5` is baked into the base tag itself** (confirmed via `ollama show
   --modelfile`) — a vendor default shipped with the model, not something introduced during the
   VRAM-fit derivation. A presence penalty this high actively discourages the model from repeating
   tokens it's already used — exactly the kind of pressure that can push a thinking-capable model
   to cut a long reasoning chain short, or fail to cleanly transition out of its thinking block
   into a structured final answer, rather than let the penalty degrade output quality further. A
   plausible, testable mechanism — see the A/B test below.
2. **Nemotron is the more resource-constrained model at idle** (partial CPU spillover on a
   32.9B MoE) yet shows *fewer* reasoning-only stalls. This cuts against a simple "VRAM/compute
   pressure causes reasoning-only stalls" story — if raw resource headroom were the driver,
   nemotron should be worse, not better, on this specific metric. Interruptions do correlate with
   resource/context pressure (see the [Fix](../README.md#fix) section); reasoning-only stalls
   specifically don't track the same resource-pressure story.

## Reasoning-only stalls correlate with depth, not breadth

For `qwen3.5:9b-vram-fit`'s 149 reasoning-only stalls specifically:

- **Average input context at time of stall: ~103,300 tokens**, vs. ~89,600 tokens average across
  *all* its API calls in the same window — stalls skew toward deeper context, not shallow/early
  turns.
- **Average tool-call count at time of stall: 42.4** — these are turns deep into long, sustained
  agentic sessions (many tool calls already accumulated), not one-shot or early-session failures.

## Why this looks like "not catching the nudge" even though the judge still fires

A reasoning-only clean stop is still treated as non-empty text by Hermes and **does** get judged —
it's only a truly empty/dropped stream that's skipped. So mechanically, the goal loop's post-turn
hook does not silently skip judging on a reasoning-only stall — the judge sees whatever text got
synthesized from the reasoning and returns a real verdict.

The perceived unreliability is therefore not a plumbing gap — it's a **content quality problem one
layer up**. A synthesized "final response" built out of narrated chain-of-thought (e.g. "I should
now write the file and then verify it...") is exactly the kind of text a judge model will often
read as genuine progress and return `continue` on, even though no actual tool call happened that
turn. The loop keeps technically cycling, but real work isn't advancing at the rate the turn count
would suggest — from an operator watching the session, this reads as "the goal nudge fired but
nothing happened," which is functionally indistinguishable from "the nudge didn't fire" unless
you're reading `agent.log` line by line.

## The `presence_penalty` A/B test

### Pass 1: synthetic single-turn — inconclusive

Built `qwen3.5:9b-vram-fit-pp0` (identical Modelfile, `presence_penalty=0` instead of the
inherited `1.5`). Constructed a real ~102K-token context (matching the production stall-time
average almost exactly) by concatenating three real source files as the prompt. Sent 15 varied
single-turn tasks (short-answer questions and tool-call-eliciting instructions, with a tool schema
available) against this same context to each tag, applying the exact production classification
criterion.

**Result: 0/15 reasoning-only clean stops for either tag.** Both variants behaved near-identically
— similar tool-call rates, similar content-vs-empty splits. This is a null result, not a negative
one: a single isolated turn with a large text-blob context and zero prior tool-call history is
materially different from the production cases analyzed above, which cluster at ~42 average
accumulated tool calls deep into a long *multi-turn agentic loop*. Raw context token count alone
(which this test matched) was evidently not the operative variable.

### Pass 2: real multi-turn `/goal` loops — a clear, if small-n, signal

Ran 3 real `/goal`-loop runs per model variant (real sandbox, real tool execution, real judge
calls) — a genuinely open-ended, multi-step task: read three real source files and write a
function report per file, via `hermes chat -q "/goal <task>" -m <tag> --format stream-json`.
Report files deleted between runs to force fresh work each time.

| Run | Model | Goal-loop turns needed | Reasoning-only stall events |
|---|---|---|---|
| baseline 1 | pp=1.5 | 1/20 | 1 (at final turn) |
| baseline 2 | pp=1.5 | **5**/20 | **3** (2 mid-task + 1 final) |
| baseline 3 | pp=1.5 | 1/20 | 1 (at final turn) |
| pp0 1 | pp=0 | 1/20 | 1 (at final turn) |
| pp0 2 | pp=0 | 1/20 | 1 (at final turn) |
| pp0 3 | pp=0 | 1/20 | 1 (at final turn) |

**Finding 1 — a "final-turn reasoning-only stall" is universal, independent of
`presence_penalty`.** All 6 runs (both arms) produced a reasoning-only clean stop at the final
wrap-up turn, right after real work was already complete — the model narrates "I've finished, let
me confirm..." in its reasoning and never emits distinct final-answer content. The goal judge
handled this correctly every time (it checks actual conversation/artifacts, not just the
synthesized text, and returned `done` in all 6 cases). This looks like an architecture/parser
characteristic of this model family's "wrap-up" transition, not something `presence_penalty`
touches at all.

**Finding 2 — the real difference is MID-task reasoning-only stalls, and `presence_penalty=0` had
zero of them across 3 runs vs. baseline's 1-of-3-runs incident.** Baseline run 2 hit two
reasoning-only stalls *during* the work, deep into sustained tool use — matching the depth
correlation above almost exactly — each one costing a wasted goal-turn where the judge had to say
`continue` and re-prompt before real progress resumed: exactly the "loop looks stuck" pattern this
whole investigation started from. None of the 3 `pp0` runs exhibited this: every `pp0` run
completed in exactly 1 goal-loop turn. Baseline needed 7 total goal-turns across 3 runs; `pp0`
needed 3.

**Honest caveats:** n=3 per arm is small — one bad baseline run drives the entire turns-needed
gap; a formal significance test isn't meaningful at this sample size, and this is a directional
signal, not a proven effect. It's also a single task shape; it doesn't establish the effect
generalizes to other task types. But the mechanism lines up cleanly with the hypothesis (a real
mid-task reasoning-only stall, at real depth, only in the baseline arm).

## `/reasoning none` eliminates the failure mode — live-confirmed

Found in real use, not a planned test — running a live interactive session on
`qwen3.5:9b-vram-fit-pp0` doing real work, the operator hit **8 reasoning-only stalls in the first
~40 minutes**, requiring manual re-prompts each time (frequency escalating to one every ~20-30
seconds by the end). Turning reasoning off entirely (`/reasoning none`) and then invoking `/goal`
for the first time in that session: **287 consecutive judge calls, 0 reasoning-only stalls,
reaching tool_turns=40+** — well past the depth where every prior stall in this investigation
occurred.

**Mechanism, confirmed in source:** `/reasoning none` is not a no-op for local Ollama, despite an
earlier finding that graded effort levels (low/medium/high) don't forward there.
`plugins/model-providers/custom/__init__.py`'s `CustomProfile.build_api_kwargs_extras`
special-cases `effort=="none"` — on an Ollama endpoint it sends `extra_body["think"]=False`, which
Ollama's native API honors directly, fully disabling the model's thinking channel. With no
thinking channel, "reasoning-only clean stop" (non-empty reasoning + empty content) becomes
structurally unreachable — there's nowhere for the model to strand an answer.

**Why this is stronger than the `presence_penalty` result above:** that A/B test found a
*directional* reduction at n=3 per arm — real, but small-sample. This is a complete elimination
(0/287 vs. 8 incidents), confirmed at real depth, in a real unplanned session — not inferred from
a correlation, but a direct demonstration of the mechanism (no thinking channel → no
reasoning-only state possible).

**Trade-off, not evaluated:** disabling reasoning entirely removes whatever quality benefit
thinking mode provides on genuinely hard problems — no data here on whether task *correctness*
(not just goal-loop liveness) suffers. `presence_penalty=0` is a softer intervention that keeps
thinking intact; the two aren't mutually exclusive.

**Practical recommendation:** run `/reasoning none` before starting a `/goal` loop on
`qwen3.5:9b-vram-fit-pp0` (or any local-Ollama-backed model) when loop liveness matters more than
reasoning-assisted quality for that task. It's a per-session runtime toggle, not a Modelfile
change.

## What this page does not claim

- It does not claim `/reasoning none` is free — no data on whether task correctness suffers
  without a thinking channel on genuinely hard problems. What's confirmed is goal-loop
  *liveness*, not output quality.
- It does not claim `presence_penalty=1.5` is *proven* to cause mid-task reasoning-only stalls —
  the real multi-turn A/B test found a clean directional signal at n=3 per arm, not a
  statistically robust result. Treat this as "worth adopting `pp0` as a trial default and
  watching," not "conclusively fixed."
- It does not claim nemotron is unconditionally "more reliable" — its higher raw interruption rate
  in the same window is real and unexplained, and its small sample size means the reasoning-only
  gap, while directionally clear, isn't precisely quantified.
- It does not revise this repo's original diagnosis or fix — it adds a new, separate failure mode
  (reasoning-only stalls) that sits alongside, not in place of, the interrupt-misattribution
  mechanism already diagnosed and fixed there.

## Candidate next steps

1. **Instrument reasoning-only stalls with judge-agreement tracking.** Log whether the judge's
   verdict on a synthesized reasoning-only response was later contradicted by the next turn's real
   output — would directly confirm or refute the "looks like progress, isn't" mechanism above
   rather than leaving it inferred.
2. **Re-run this comparison after the interrupt fixes accumulate a few days of data**, to isolate
   whether reasoning-only stalls are really the dominant remaining gap once interruption
   misattribution is out of the picture.
3. **Control for usage-pattern confound.** Nemotron's sample doesn't include much deep-context,
   long-session usage (it hasn't been the default model). A same-model, same-depth controlled
   comparison (forcing nemotron into an equally long goal loop) would help separate model-intrinsic
   behavior from how hard each model has actually been run.
