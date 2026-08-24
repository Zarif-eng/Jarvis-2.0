# Jarvis 2.0 — design

A ground-up redesign of my self-hosted voice assistant
([v1.0 is here, built and measured](https://github.com/Zarif-eng/Jarvis-1.0)), rebuilt around one finding from
actually living with the first version:

> **Most of the commands I actually gave it didn't need a language model at all.**
> Resolving "my sister" to a phone number is a database lookup with a correct answer.
> Running it through a 35-billion-parameter model makes it slower, less reliable, and
> dependent on a machine that might be asleep.

**Status: design frozen, build not started.** This repository is currently a design
document and a set of build rules. There is no code yet. It's here because the design
is the part I want reviewed — and because v1.0 taught me that writing the gate down
before building is what stops a target from quietly moving.

---

## The idea: answer without the model where you can

Requests fall through three tiers and stop at the first one that can answer:

| Tier | Latency | Uses the model? | Needs the GPU awake? |
|---|---|---|---|
| **0 — Grammar** | < 700 ms | **no** | **no** |
| 1 — Fast | 2–4s warm | yes, reasoning off | yes |
| 2 — Deliberate | 30s – minutes | yes, full reasoning | no — it pushes to your phone when done |

Tier 0 solves three problems at once: the common case becomes instant, it becomes
*correct* (SQL doesn't hallucinate a phone number), and it works while the desktop is
asleep — which is what reconciles "my GPU shouldn't run 24/7" with "answer me now."

Target: **60–70% of daily commands answered with no model involved.**

All three tiers converge on **one action gate**, so permission rules are enforced in
exactly one place no matter which tier produced the action.

---

## Three decisions worth reading

**A sparse model beats a dense one that doesn't fit.** A dense 27B needs 16–19 GB for
weights alone on a 16 GB card, and spilling to system RAM means every token crosses the
bus forever — about 8–12 tok/s. A mixture-of-experts model of *larger* nominal size
activates only ~3B parameters per token, so attention stays resident on the GPU while
idle experts live in RAM: **~50–60 tok/s on the same card**, and the better reasoner.
Same memory budget, five times the throughput.

The benchmark isn't taken on faith — one review calls the chosen model's tool calling
"still shaky," and tool calling is what the whole design rests on. So 20 recorded
tool-call cases run against both candidates and **the model is committed on that
number**, not on the benchmark table.

**The security problem is named, and fixed in code rather than in a prompt.** This
system deliberately combines the three ingredients of dangerous prompt injection:
private data in memory, untrusted content from web search, and the ability to act. Any
two are fine; all three is the whole problem.

So every value in a proposed tool call carries **provenance**, and the dispatcher
refuses any call whose arguments trace back to untrusted text — checked before the
model's output is ever parsed into an action. Search can inform an answer; it can never
originate one. A system-prompt instruction is not a security boundary.

**Confirm-everything decays into blind approval.** Three action tiers instead: silent
for reversible reads, act-then-30-second-undo for reversible writes, and blocking
confirmation only for things that reach another person, spend money, or can't be
undone. After 10 approvals of the same shape with zero denials, it asks once whether to
promote that *shape* — and the resulting rule is a line in a plain-text file you can
read, diff and delete.

---

## Measurement gates the build

Nine phases (P0–P8), each shipping something usable on its own, each with a gate that is a
**recorded number rather than a judgement call**. 50 frozen test cases — 20 tool calls,
15 transcription, 10 memory, 5 prompt-injection refusals — are recorded once, never
trained on, and re-run after every model change, prompt change or learned rule. Any
case that regresses reverts the change.

Phase 1's gate is the one that makes the whole thesis concrete: **"call my sister",
"timer for ten minutes" and "what's her number" all work with zero model loaded.**

v1.0's design listed metrics and never gated on them. That's the specific mistake this
version is built not to repeat.

---

## Read the design

- **[`docs/DESIGN.md`](docs/DESIGN.md)** — the full engineering memo, and ground truth.
  Model selection and the memory-bandwidth argument, the three-tier router, speech
  (transcription, phrase boosting, synthesis), skills and progressive disclosure, the
  action gate, the injection interlock, memory with fact lifetimes, the nine build
  phases and their gates, and what's instrumented.
- **[`CLAUDE.md`](CLAUDE.md)** — the operating rules for every working session, including
  the ones that exist to stop me cheating: never edit a test case to make it pass, don't
  build ahead of the current phase, don't move past a gate you haven't measured.

Rev B supersedes Rev A; §00 of the design is a table of what changed and why, which is
the fastest way to see the reasoning.
