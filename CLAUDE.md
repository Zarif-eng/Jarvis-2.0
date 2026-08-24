# Jarvis — operating rules for every session

Read `docs/DESIGN.md` before proposing architecture. It is ground truth. If reality
contradicts it, edit it in the same commit — do not leave the design stale.

## How to work with me

I am building this to learn, not just to have it. After every working chunk, give me
**3–5 lines**: what you built, why that approach, and the one thing that would break it.
No lectures, no walls of theory, no restating what the code already says. If a decision
had a real alternative, name it in one clause.

Append the same content to `docs/BUILDLOG.md` with any number you measured. The final
report is compressed from those entries, so an entry that says "wired up the router" is
a wasted entry.

## The four principles

1. **If it can be done reliably and accurately without the model, it does not touch the
   model.** Contact lookup is SQL. Timers are a scheduler. Unit conversion is a function.
   Before adding an LLM call, state why a deterministic path can't do it.
2. **Inference stays on hardware we own.** No hosted API for reasoning. This is decided;
   do not re-propose it.
3. **The device that owns a privileged operation performs it.** The agent resolves intent
   and emits a structured action; the phone opens `tel://`, the Mac runs the AppleScript.
4. **Every self-granted permission is a line in a plain-text file** that can be read,
   diffed and deleted. Nothing is learned into a place we can't look.

## Action tiers — enforced at the gate, never self-declared at runtime

- **A · auto** — pure reads, reversible local state. Silent.
- **B · undo** — acts, notifies, 30-second undo, then commits.
- **C · confirm** — blocks. Anything a third party receives, anything that spends money,
  anything irreversible, system/security settings, filesystem outside the workspace.

Confirmations must be **specific**: `Send to Aisha (+1 555…): "running late"`, never
"Jarvis wants to send a message".

## The injection interlock — code, not prompt

Every value in a proposed tool call carries provenance. The dispatcher **refuses any call
whose arguments trace to untrusted text** (search results, email bodies, web pages, files
the user didn't author), before the model's output is parsed into an action. When
untrusted content is in context this turn, all promotions suspend and everything drops to
Tier C. A system-prompt instruction is not a security boundary — do not implement this as
one.

## Evaluation

`eval/cases/` holds 50 frozen cases. They are never training data.
**Never edit `eval/run.py` or a case to make something pass.** If a case is genuinely
wrong, say so and wait for me.

Every model change, prompt change, promotion or taught fix re-runs the full set. A change
that regresses any case reverts.

## Serving

`llama-server` from llama.cpp — **not Ollama** (it discards the chat template, which makes
`reasoning_effort` unreachable). Reasoning effort is the tier routing knob:
`none` for Tier 1, `medium`/`xhigh` for Tier 2. The model ships defaulting to `xhigh`;
never leave it there.

Sampling differs by mode and is not optional:
- thinking — `temp 1.0, top_p 0.95, top_k 20, min_p 0.0`
- instruct — `temp 0.7, top_p 0.80, top_k 20, presence_penalty 1.5`

## Scope

Do not build ahead of the current phase. Phases are in `docs/DESIGN.md` §12 and each has a
gate that is a recorded number, not a judgement call. Don't move past a gate you haven't
measured.
