# Making Opus 4.8 Act Like Fable — Research & Setup Notes

> **Round 2:** see `OPUS-48-FABLE-DEEP-DIVE.md` for the follow-up research
> geared specifically to Opus 4.8's capabilities (effort levels, fast mode,
> behavioral deltas vs Fable 5) and the config changes it drove: per-agent
> `effort` frontmatter, 4.8 tuning sections in the agents, and the root
> `CLAUDE.md` operating layer.

Research into the approach from Nate Herk's video **"How I Make Opus Think Like
Fable (5 easy steps)"** (https://youtu.be/XTBWVVcF3Pk), and the configuration
installed in this repo's `.claude/` directory as a result.

## The core idea

Fable 5's edge over cheaper models is less about raw intelligence and more
about *process discipline*: how it reads what a request is actually asking
for, breaks hard problems into checkable pieces, verifies its own work instead
of trusting what "sounds right", and refuses to guess when it doesn't know.
That process can be extracted into plain-language instructions (a Claude Code
**skill**) and enforced on Opus 4.8, Sonnet, and Haiku. It shapes *procedure*,
not *capability* — it will not raise a cheaper model's reasoning ceiling, but
it reliably stops the classic cheap-model failures (skipped verification,
plausible fabrication, silent scope drift).

## The video's method, summarized

1. **Extract Fable's operating manual.** Ask Fable (or reconstruct from its
   behavior) a description of its own working process, and turn it into a
   reusable skill file rather than a per-conversation prompt.
2. **Five gates for every large task** — the "Fable mode" loop:
   - **Scope** — restate the problem in precise terms before acting.
   - **Evidence** — gather the actual inputs (open the files, fetch the
     sources) before producing anything.
   - **Attack** — play devil's advocate against your own plan/assumptions.
   - **Verify** — check work against a test that can *fail*, not a feeling.
   - **Report** — deliver with confirmed facts separated from inferences.
3. **Plan first, and critique the plan** — a written stage map with one
   verifiable artifact per stage, revised at most twice before going back to
   the user.
4. **Use effort levels deliberately** — higher effort is not always better;
   tune effort to the task instead of maxing it (in Claude Code: `/model` and
   the effort setting).
5. **Model routing table** — send bulk mechanical work to Haiku, standard
   reasoning to Sonnet, and reserve Opus for orchestration/peak synthesis, so
   cost stays down without quality loss on the parts that matter.

## What's installed in this repo

This is the open-source **fable-mode v3** skill set
(https://github.com/mrtooher/fable-mode), which implements the video's method
for Claude Code. Files are copied verbatim.

Skills (`.claude/skills/`):

| Skill | Purpose |
|---|---|
| `fable-mode` | The core staged-execution loop; triggers on large multi-file/multi-source tasks or on request ("deep work mode", "be systematic"). |
| `fable-opus` | Routes a task through the Opus orchestrator agent — the strongest staged run. |
| `fable-sonnet` | Same loop pinned to Sonnet — the balanced cost/quality choice. |
| `fable-haiku` | Same loop pinned to Haiku — bulk mechanical work, tightened verification. |
| `execution-guardrails` | Always-on rules for every task and model: verify-before-flag, warning batching (threshold 3), word-boundary find-and-replace safety. |

Agents (`.claude/agents/`) — the v3 "load-bearing change": delegation is
enforced *structurally* instead of suggested in prose:

| Agent | Model | Key constraint |
|---|---|---|
| `fable-orchestrator` | Opus | **No Write/Edit tools** — it physically cannot do the work itself, so every artifact must come from a named worker. |
| `fable-worker-sonnet` | Sonnet | One bounded assignment, must run the named pass-condition check and show its output. |
| `fable-worker-haiku` | Haiku | Same, tightened: a bare "unverified" is itself a failure; escalates to Sonnet instead of improvising. |
| `fable-verifier` | Haiku | Read-only cold verifier; briefed with only the spec + artifact path so it can't inherit the producer's blind spots. |

## How to use it

- Ordinary tasks: do nothing — the skills deliberately do NOT trigger on
  single-pass work; staging a trivial task buries the answer in ceremony.
- Big tasks (multi-file refactor, research sweep, migration): say
  "deep work mode" / "be systematic", or invoke `fable-mode` directly. With
  the agents installed, it routes through `@fable-orchestrator`.
- To pin the run to a model: "fable on opus" / "stage this on sonnet" /
  "deep work mode but cheap" (haiku).
- Set the session model to Opus 4.8 with `/model`. Effort levels are tunable
  there too — the video's point is that max effort everywhere wastes money;
  the routing table (Haiku for mechanical, Sonnet for reasoning, Opus for
  orchestration) is encoded in the agent definitions.

## Caveats (from the skill author's own benchmarks, 2026-07)

- On short graded tasks, Opus/Sonnet score the same with or without the skill.
  The measured value shows on open-ended research (real sources vs. plausible
  fabrication) and on forcing verification at lower tiers.
- On Haiku the effect swung both directions at n=1 (+25 / −17) — route
  quality-critical work to Sonnet, not Haiku.
- The orchestrator's Bash access could technically create files; its prompt
  forbids it, but that's the known side door.
- `execution-guardrails` references a "source-of-truth" skill that is not part
  of this set; the reference is inert.

## Sources

- Video: [How I Make Opus Think Like Fable (5 easy steps)](https://www.youtube.com/watch?v=XTBWVVcF3Pk) — Nate Herk
- Article version: [x.com/nateherk](https://x.com/nateherk/article/2074324638159581404)
- Skill implementation installed here: [mrtooher/fable-mode](https://github.com/mrtooher/fable-mode)
- Five-gates breakdown: [Geeky Gadgets](https://www.geeky-gadgets.com/steps-opus-fable-mode/), [MindStudio](https://www.mindstudio.ai/blog/fable-mode-skill-cheaper-models-think-like-frontier)
- Related: [Fable Skills (getmasset)](https://www.getmasset.com/resources/blog/fable-skills), [Ken Huang — Claude Fable 5: What Changed](https://kenhuangus.substack.com/p/claude-fable-5-what-changed-and-how)
