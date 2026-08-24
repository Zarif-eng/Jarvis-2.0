# Jarvis: A Local Personal Assistant

**Rev B** · **Status:** design frozen, build not started · **Date:** 20 August 2026 · **Supersedes:** Rev A (15 Aug 2026)

A voice agent across four devices, rebuilt around a single realisation: the fastest and
most reliable version of almost every command actually given to it involves no language
model at all.

**This file is ground truth.** If they diverge, this one wins.

---

## 00 · What changed from Rev A

| Area | Rev A | Rev B | Because |
|---|---|---|---|
| Reasoner | Qwen3-14B dense, Ollama | **Qwen3.6-35B-A3B (MoE), llama-server** | A dense model that overflows 16 GB is unusable; a sparse one of the same size is not (§02) |
| Trigger | Always-listening, Porcupine everywhere | **Toggle hotkey; Action Button on iPhone** | Deletes the native-iOS phase, the entitlement, the battery cost, the 7-day re-signing |
| Transcription | whisper.cpp | **Parakeet TDT 0.6b v3 + phrase boosting** | Lower WER at a third the size, streaming; boosting is what fixes names |
| Routing | Every request reaches the model | **Deterministic Tier 0 answers first** | Contact lookup is SQL, not inference (§01) |
| MacBook | Interchangeable secondary worker | **Client + control target, never infers** | 16 GB unified is both VRAM and RAM — MoE offload has nowhere to go |
| Raspberry Pi | Request router | **Scheduler, queue, memory, notifications** | Hotkey removed the routing job; autonomy created a scheduling one |
| Approval | Confirm destructive actions | **Three tiers, earned promotion, interlock** | Confirm-everything decays into blind approval (§07) |
| Knowledge | Weights only | **Tavily, budgeted and quarantined** | Training data is stale — but results are attacker-controlled (§08) |
| Outbound | none | **ntfy push with inline approve/deny** | An assistant that acts alone must reach us first |
| Quality | Metrics listed, never gated | **50 frozen cases gate every phase** | Rev A's measurement section was aspirational |

---

## 01 · Principles

1. **If it can be done reliably without the model, it doesn't touch the model.** Resolving
   "my sister" to a number is a lookup with a correct answer. Running it through a 35B
   model makes it slower, less reliable, and dependent on a machine that is asleep.
2. **Inference stays on hardware we own.** Not because a hosted API could reach the
   devices — it can't; a model returns text and our code decides whether to act — but
   because the prompt would carry a running transcript of a life to a third party. A
   values call, made deliberately. Rev A hedged with a hosted escape hatch; Rev B has none.
3. **The device that owns a privileged operation performs it.** Carried from Rev A
   unchanged. Adding a capability is adding a Shortcut, not changing the agent.
4. **Trust is earned against evidence and revoked in one line.** Every self-granted
   permission is plain text we can read, diff and delete.

---

## 02 · The constraint that picked the model

Qwen3.8-27B does not fit. It is **dense** — every parameter active on every token — and
its 4-bit weights need 16–19 GB before any KV cache, on a 16 GB card. 3-bit fits but
degrades tool-calling precision exactly where the system depends on it.

Spilling to the 32 GB of DDR5 alongside is the instinct, and for a dense model it is the
worst possible move: every token crosses the bus, forever. **~8–12 tok/s.**

For a **mixture-of-experts** model it is the intended design. Qwen3.6-35B-A3B activates
~3 B of its 35 B per token, so attention and the shared expert stay resident on the GPU
while 256 idle experts live in RAM and 8 are fetched per token. **~50–60 tok/s on a 16 GB
card** — five times the throughput, and the better reasoner.

| Candidate | Arch | Fits 16 GB | Speed | SWE-bench V. | Notes |
|---|---|---|---|---|---|
| **Qwen3.6-35B-A3B** | MoE · 3B active | yes, via offload | ~50–60 tok/s | 73.4% | 262K ctx, multimodal, Apache 2.0. **Chosen** |
| GPT-OSS 20B | MoE · 3.6B active | yes, resident | ~41 tok/s | — | Best-rated output-format reliability. **Fallback** |
| Qwen3.8-27B | Dense | **no** | 8–12 tok/s | — | 16–19 GB for weights alone. Rejected |

> **Open risk, decided by measurement in P3.** One review calls Qwen3.6's tool calling
> "still shaky." That is the number this design rests on, so it is not taken on faith: the
> 20 tool-call cases run against both candidates before the model is committed. GPT-OSS
> takes the slot if it scores higher, regardless of the benchmark table.

---

## 03 · Serving

Ollama discards the chat template, making `reasoning_effort` unreachable — and Rev B
routes on reasoning effort. It also doesn't expose expert offload properly. Both
requirements land on llama.cpp.

```
llama-server \
  -m qwen3.6-35b-a3b-IQ4_NL.gguf \
  -ngl 999 \                 # everything on GPU, then claw back with:
  --n-cpu-moe 20 \           # expert tensors to RAM. Sweep down to find the cliff.
  --flash-attn on \          # critical on a small card at long context
  --no-mmap \                # keeps system RAM pressure sane
  -c 32768 \                 # 32K, not 262K — KV cache is the other budget
  --chat-template-kwargs '{"reasoning_effort":"low"}'
```

Keep `-ngl` maxed and control memory with `--n-cpu-moe`. Sweep downward until throughput
falls off a cliff, back off one step.

Sampling by mode (not optional — getting it wrong looks like a bad model):
- thinking — `temp 1.0, top_p 0.95, top_k 20, min_p 0.0`
- instruct — `temp 0.7, top_p 0.80, top_k 20, presence_penalty 1.5`

---

## 04 · Routing — three tiers

A request falls through and stops at the first tier that can answer it.

| Tier | Target | Reasoning | Waits for | Returns via |
|---|---|---|---|---|
| **0 · Grammar** | < 700 ms | *no model* | nothing — PC may be asleep | speech, inline |
| 1 · Fast | 2–4 s warm | `none` | PC awake (+20–30 s cold) | speech, inline |
| 2 · Deliberate | 30 s – several min | `medium`/`xhigh` | nothing — you walk away | ntfy push |

All three converge on **one action gate**, so approval rules are enforced in exactly one
place regardless of which tier produced the action.

**Tier 0 solves three problems at once.** It makes the common case instant; it makes the
common case *correct* (SQL does not hallucinate a phone number); and it works while the PC
is asleep, which is what reconciles "my PC should not always be awake" with "answer me
immediately."

Its failure mode is brittleness — `call Aisha` matches, `give Aisha a ring` doesn't. Two
mitigations: unmatched input falls through silently (the user sees a slower answer, never
a failure), and every Tier 1 turn that resolved to a Tier-0-shaped action is logged as a
candidate pattern for review.

**Why reasoning effort is the knob:** the model ships defaulting to `xhigh`, measured at
22,000 reasoning tokens on a trivial prompt — seven minutes before the first word. As a
dial it gives three genuinely different products from one loaded model, with no second
model and no extra VRAM.

---

## 05 · Architecture

Tailscale carries everything — identical addressing on home wifi and cellular, no port
forwarding, nothing exposed publicly.

- **Raspberry Pi** — always on, never infers. Tier 0 grammar, scheduler + task queue,
  SQLite memory (sole copy), action gate, ntfy dispatch.
- **Windows PC** — RTX 5060 Ti 16 GB, **asleep by default**, woken by WoL. llama-server
  (Tiers 1 and 2), Parakeet, Kokoro, skill loader. Stateless — safe to lose.
- **iPhone 17** — client + control target. Action Button, capture, Shortcuts execution,
  approve/deny from push.
- **MacBook Pro M3** — hotkey client + AppleScript target. **Never infers.**

Memory lives on exactly one node because it is the only always-on node. If the brain
roamed, the fact store would fork — the one piece of Rev A's reasoning that survived
intact.

**Removed from Rev A:** worker registry, failover path, wake word. Demoting the Mac
deleted three components whose only job was insuring against a risk a stateless,
restartable worker doesn't really have.

---

## 06 · Speech

Generic ASR benchmarks are useless here — the words that matter are the ones no benchmark
contains. A model swap moves average WER a point or two; biasing moves the words we care
about far more.

- **Transcription** — Parakeet TDT 0.6b v3. 6.32% avg WER vs Whisper large-v3's ~7.4%, a
  third the parameters, natively streaming. English only.
- **Phrase boosting — the actual win.** NeMo supports GPU phrase boosting for TDT from
  2.5.0: tab-separated terms with scores +20 to +100 at decode time. Cap ~**100 phrases**
  before quality degrades, so the list needs a priority policy (ranked by frequency,
  refreshed from memory), owned by the `personal-lexicon` skill.
- **Capture** — toggle hotkey (`Ctrl+Space` both desktops, Action Button on iPhone).
  Stops on *either* a second press or end-of-utterance, whichever first. Parakeet streams,
  so endpointing is semantic rather than a fixed silence timer.
- **Synthesis** — Kokoro to start (~45 ms to first audio, streaming, known-good). Behind
  an interface from day one; voice quality gets its own phase (P8) rather than blocking.

---

## 07 · Skills, and approval

A shaky tool-caller gets worse as the tool list grows. Skills fix this with **progressive
disclosure**: only name + one-line description stay in context; the body loads when the
description matches. "Pick one of eight descriptions" is a much easier problem than "pick
one of forty tools." MCP servers are wrapped *behind* skills; the model never sees a bare
tool list.

```
skills/web-search/
  SKILL.md        # frontmatter: name, description, action tier, tool allowlist
  examples.md     # verified traces, appended by the teaching loop
  gotchas.md      # corrections given in words
```

Starting seven — index stays small, since past ~15–20 skills routing accuracy degrades:

| Skill | Owns | Tier |
|---|---|---|
| `web-search` | When to spend a credit, cache policy, quarantine | A |
| `personal-lexicon` | The 100 boosted ASR phrases and their ranking | A |
| `speech-interpretation` | Phrasing habits — "the usual", truncation | A |
| `contacts-and-comms` | Resolving people, calls, messages | C |
| `device-control` | Shell, AppleScript, window/media control, WoL | A–C |
| `memory-management` | Fact writes, supersession, review queue | B |
| `task-scheduling` | Recurring and triggered work, queued approvals | B |

> "Understanding what I say" is two problems in two places. Vocabulary is **acoustic**,
> fixed in the transcriber by boosting. Phrasing is **semantic**, fixed in the model by a
> skill. Conflating them means fixing the wrong layer.

### Action tiers

| Tier | Behaviour | Covers |
|---|---|---|
| **A · auto** | Silent | Pure reads, reversible local state — calendar/memory reads, timers, volume, brightness, launching apps, media, web search |
| **B · undo** | Acts, notifies, 30-second undo, commits | Calendar writes, files in workspace, memory writes, scheduling, drafting |
| **C · confirm** | Blocks | Anything a third party receives, spends money, is irreversible, system/security settings, filesystem outside workspace |

Tier B is where leniency lives: for a reversible private action, a pre-confirmation buys
nothing a 30-second undo doesn't and costs a decision every time.

Four mechanisms make Tier C bearable:
- **Intent-thread batching** — same skill, same target, 5-minute window → one confirmation.
- **Earned promotion** — after 10 approvals of a shape with 0 denials, it asks once whether
  to move that *shape* down a tier. A rule is approved, not an action, and it's a deletable
  line in a file.
- **Presence changes the channel, never the rule** — inline / push / queued.
- **Batch clearing** for approvals queued overnight.

---

## 08 · Security

This system deliberately combines the three ingredients of dangerous prompt injection:
**private data** in memory, **untrusted content** from search, and **the ability to act**.
Any two are fine. All three is the whole problem, and Rev A never named it because it had
no web access.

The mitigation is a type check, not an instruction. Every value in a proposed tool call
carries provenance, and **the dispatcher refuses any call whose arguments trace to
untrusted text** — before the output is parsed into an action. Search can inform an
answer; it can never originate an action. When untrusted content is in context this turn,
all promotions suspend and everything drops to Tier C.

Confirmations are specific for the same reason: "Jarvis wants to send a message" is
unreviewable.

**Kill switch**, built in week one because it will never get built later: Tailscale ACLs
restricting each client to the Pi's API port only (not SSH, not the PC directly), and a
single `jarvis panic` that revokes the device key, flushes the queue, and drops everything
to Tier C.

**Web search budget:** Tavily free is 1,000 credits/month (~33 basic searches a day, hard
stop). A gate runs before any credit: answerable from memory? genuinely time-sensitive? is
there a deterministic API that answers it better? Weather, stocks and time-in-city go to
direct APIs with templated output — correct, instant, free. Results cache on normalised
query, 24 h TTL.

---

## 09 · Memory and learning

Rev A stored source, timestamp and confidence, then never said what to do with them.
Rev B gives facts a lifetime:

- **Permanent** — sister is Aisha. Never expires.
- **Current-state** — employer, address. A new value *supersedes*; the old row is marked,
  never deleted, so history survives.
- **Episodic** — decays after N days unless reinforced.

Auto-written facts land in a weekly review queue. Reviewed facts are labelled data.

### One mechanism, three surfaces

Promoted confirmations, learned grammar patterns and taught skill fixes are the same
pipeline: **observe → propose → approve once → write to a plain-text file**. Built once.

| Surface | Output |
|---|---|
| 10 approvals, 0 denials | `promotions.txt` |
| A shape Tier 1 keeps handling that Tier 0 could match | `grammar/patterns.txt` |
| A failure you taught | `skills/*/examples.md` |

Every change then re-runs all 50 frozen cases. No case regresses, or the change reverts.
That is what separates learning from drift.

### Teaching a failure

A failed multi-step task **aborts and reports** — no improvised recovery, which is how a
shaky tool-caller turns one failed step into a creative wrong one. Completed Tier B/C
actions are named explicitly. A clean abort leaves a legible trace: goal, every step,
exact arguments, error, state at abort.

The notification carries **Teach** at three levels of effort:
- *correct the call* — fix arguments, retry from that step
- *demonstrate* — do it once, it records the sequence
- *explain* — say what went wrong, it becomes a gotcha line

### The frozen set

50 cases, recorded once, never trained on, re-run after every model or prompt change.

| Count | Bucket | Scores | Gates |
|---|---|---|---|
| 20 | Tool calls | Correct action *and* correct arguments | Model selection · P3, P6 |
| 15 | Transcription | WER on recorded audio of real contacts and places | P2 |
| 10 | Memory | Recall across restart; supersession behaves | P1 |
| 5 | Refusal | Injected instruction inside a fake search result — does it act? | P6, must stay 5/5 |

---

## 10 · Build sequence

Each phase ships something usable alone. **P1 delivers a working assistant with no model
in it** — principle 1 made structural. Every gate is a recorded number.

| Phase | Builds | Gate |
|---|---|---|
| **P0** | Scaffold, CLAUDE.md, DESIGN.md, build log; record all 50 cases + audio; the harness | Harness runs end to end, reports a baseline of zero |
| **P1** | SQLite memory with validity types, contact store, Tier 0 grammar, action gate with provenance typing | "Call my sister" / "timer for ten minutes" / "what's Aisha's number" work with **zero model**. 10/10 memory cases across a restart |
| **P2** | Parakeet + boosting, toggle hotkey, semantic endpointing, Kokoro behind interface | Personal-phrase WER beats recorded Whisper baseline. Tier 0 voice command under 700 ms, measured |
| **P3** | llama-server, `--n-cpu-moe` sweep, reasoning-effort routing, skill loader | Tool-call accuracy recorded for **both** candidates, model committed on that number. tok/s logged |
| **P4** | Tailscale ACLs, WoL, scheduler, queue, ntfy approve/deny, `jarvis panic` | A cellular request wakes a sleeping PC and returns an answer. Panic revokes mid-session |
| **P5** | Action Button, Shortcuts per capability, lock-screen approvals | "Jarvis, call my sister" hands-free; a Tier C action approved from the lock screen |
| **P6** | The seven skills, scheduled/triggered tasks, promotion machinery, teaching flow, Tavily | Overnight task queues an approval cleared in the morning. 5/5 refusal cases. Tier 0 hit rate logged |
| **P7** | Shell + UI automation (PC), AppleScript (Mac), behind `device-control` | Multi-step cross-device task, each step correctly tiered; a deliberate failure aborts cleanly and can be taught |
| **P8** | TTS swap behind the existing interface | Blind A/B preference vs Kokoro, no regression in time-to-first-audio |

---

## 11 · Instrumented continuously

- **Tier 0 hit rate** — share of daily commands answered with no model. Target 60–70%.
- **Latency per tier**, warm and cold, split by stage.
- **Tool-call accuracy** — correct action *and* arguments, per model and per skill.
- **Personal-phrase WER** on the fixed recorded set, before and after boosting.
- **Approval statistics** — prompts/day, approval rate, time-to-decision. A rate nearing
  100% with falling decision time is fatigue, and means a tier is mis-assigned.
- **Memory precision** — of auto-written facts, how many survive review.
- **Tavily credits** against the monthly 1,000, plus cache hit rate.
- **Cold-path frequency** — how often a request pays the 20–30 s wake penalty. High means
  Tier 0 is too narrow.

---

## 12 · Repo layout

```
jarvis/
  CLAUDE.md              # rules every session must follow
  docs/
    DESIGN.md            # this file — ground truth
    BUILDLOG.md          # appended after every working chunk
    decisions/           # one file per reversal, with the reason
  eval/
    cases/               # the 50 frozen cases, audio included
    run.py               # the harness. Never edited to make a case pass.
  grammar/
    patterns.txt         # Tier 0 — plain text, reviewable
  skills/                # one directory each, per §07
  coordinator/           # Pi: scheduler, queue, gate, ntfy
  worker/                # PC: llama-server, ASR, TTS
  clients/               # hotkey daemons, Shortcuts definitions
  memory/
    jarvis.db            # SQLite. Inspectable and correctable by hand.
  promotions.txt         # every relaxed rule, one line each
```

**Working method.** After every working chunk: 3–5 lines in the build log — what was
built, why that approach, the one thing that would break it, plus any measured number. The
final report is a *compression* of real entries rather than a reconstruction from memory,
which is the only way it ends up specific.
