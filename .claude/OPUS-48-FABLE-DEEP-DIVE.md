# Making Opus 4.8 Act Like Fable 5 — Deep Dive

Round 2 of the research in `FABLE-MODE-SETUP.md`, geared specifically to what
**Claude Opus 4.8** actually supports and how it differs from Fable 5. The
first pass installed the generic fable-mode v3 skill set; this pass tunes it
using Anthropic's own model documentation and migration guidance (which
includes, in both directions, exactly the deltas between Fable 5 and Opus 4.8).

## 1. Opus 4.8 capability sheet (what we're working with)

| Capability | Opus 4.8 | Notes for this setup |
|---|---|---|
| Model ID | `claude-opus-4-8` | In Claude Code: `/model opus` |
| Context / max output | 1M tokens / 128K | No long-context premium |
| Pricing | $5 / $25 per MTok | Fable 5 is $10 / $50 |
| Thinking | Adaptive only (`{type: "adaptive"}`); **off when omitted** | Fable 5: always on, can't be disabled |
| Effort | `low` / `medium` / `high` / `xhigh` / `max` (default `high`; Claude Code defaults to `xhigh`) | The main intelligence↔cost lever; matters more on 4.8 than any prior Opus |
| Fast mode | Yes (research preview) — `/fast` in Claude Code, `speed: "fast"` + beta `fast-mode-2026-02-01` on the API | ~2.5× output speed, premium price. 4.8 is the durable fast tier (4.7 fast is deprecated) |
| Task budgets | Yes (beta) — token ceiling the model paces itself against | Useful cap for long agentic loops |
| Mid-conversation system messages | **Opus 4.8 only** — `{"role": "system"}` in `messages[]` | Inject operator context mid-session without cache invalidation |
| High-res vision | 2576px long edge, pixel-accurate coordinates | |
| Sampling params | `temperature`/`top_p`/`top_k` removed (400) | Steer with prompting |

## 2. What Fable 5 does that Opus 4.8 doesn't do by default

From Anthropic's migration guides (Fable 5 section and Opus 4.8 section) — the
gap is mostly **default behavior**, not raw request surface:

1. **Delegation reliability.** On Fable 5, parallel sub-agents are dependable
   and Anthropic recommends *encouraging* delegation. Opus 4.8 is the
   opposite: it **under-reaches for subagents, file-based memory, and custom
   tools** unless told exactly when to use them. Prescriptive "call this
   when…" trigger conditions in descriptions give measurable lift on 4.8.
2. **Search-first research depth.** 4.8 is high-precision / low-recall on tool
   triggering with a system prompt present — it answers from context when
   Fable would go gather evidence. Needs an explicit search-first instruction.
3. **Grounded long-horizon execution.** Fable's edge is long-run coherence and
   self-verification. On 4.8, Anthropic recommends: full task spec up front in
   one well-specified turn, run at `high`/`xhigh`, and require progress claims
   to be audited against tool results (nearly eliminates fabricated status
   reports).
4. **Deliberateness overshoot.** 4.8 asks more often on minor decisions than
   Fable-style autonomous work wants; explicit small-decisions-don't-ask
   guidance cuts ask-rate ~12 points with no over-reach increase.
5. **Literal severity filters.** 4.8 (like Fable) follows "only report
   high-severity" instructions literally — review/verify harnesses must ask
   for coverage and filter downstream, or measured recall drops.
6. **Always-on thinking.** Fable thinks on every request; 4.8 runs *without*
   thinking when the field is omitted. In Claude Code this is handled for you;
   on raw API calls set `thinking: {type: "adaptive"}` explicitly.

What prompting **cannot** close: Fable's higher reasoning ceiling, its
long-horizon coherence living in the weights, and its 1M-context always-on
reasoning economics. The skill files are honest about this — procedure, not a
capability transplant.

## 3. What changed in this repo (the 4.8 gearing)

### `CLAUDE.md` (new, root)
The always-loaded prompt layer. Carries the five Fable-behavior corrections
Anthropic recommends prompting into 4.8: act-don't-idle, autonomy on minor
decisions, explicit capability triggers (subagents / search / verification),
grounded reporting, and narration discipline — plus the effort & model routing
table for this repo.

### Agent definitions (`.claude/agents/*.md`)
Claude Code agent frontmatter supports `model` and `effort` — now set
explicitly per role:

| Agent | Model | Effort | Why |
|---|---|---|---|
| `fable-orchestrator` | opus (→ 4.8) | `high` | Planning/coordination; Anthropic's 4.8 default recommendation. Raise to `xhigh` per-run only for the hardest synthesis. |
| `fable-worker-sonnet` | sonnet | `high` | Reasoning-stage work; `high` is the intelligence-sensitive minimum. |
| `fable-worker-haiku` | haiku | `low` | Bulk mechanical work — "low for subagents or simple tasks"; cheapest correct tier. |
| `fable-verifier` | haiku | `high` | Verification is the load-bearing stage; spend effort here, not on narration. |

Body changes:
- **Orchestrator** gained an "Opus 4.8 tuning" section: delegate eagerly
  (counteracting 4.8's under-delegation — the Write-less design now has
  explicit behavioral backing), grounded progress claims, autonomy on minor
  decisions, silence-default narration.
- **Verifier** gained a coverage rule: report every confirmed failure
  regardless of severity, with confidence + severity labels, filter
  downstream — countering 4.8's literal reading of conservative-reporting
  instructions, while keeping verify-before-flag intact.

### Unchanged
The five skills (`fable-mode`, `fable-opus/sonnet/haiku`,
`execution-guardrails`) remain verbatim from upstream `mrtooher/fable-mode` —
their five-gate loop (scope → evidence → attack → verify → report) is already
the right countermeasure set; the 4.8 gearing lives in the layers around them.

## 4. How to run it, 4.8-style

- **Session model:** `/model opus` → Opus 4.8. Claude Code already runs it at
  `xhigh` effort by default — the sweet spot for coding/agentic work.
- **Big tasks:** give the *full* task specification in one well-specified
  first turn (4.8's long-horizon strength is unlocked by a clear up-front
  goal), then say "deep work mode" or invoke `fable-mode` — it routes through
  the orchestrator with enforced delegation.
- **Fast interactive loops:** `/fast` toggles fast mode (same 4.8 model,
  ~2.5× output speed, premium pricing) — good for tight edit-run-fix cycles,
  wasteful for long autonomous runs.
- **Cost control on long runs:** effort is the first lever (drop workers to
  `medium`/`low`), task budgets (API beta `task-budgets-2026-03-13`) the
  second.

## 5. Sources

- Anthropic model & migration reference via the `claude-api` skill (authoritative,
  cached 2026-06): Opus 4.8 capability sheet, "Migrating to Opus 4.8"
  behavioral-shift guidance, "Migrating to Claude Fable 5" (read in reverse as
  the Fable↔Opus delta), effort-level table, fast mode, task budgets.
- Claude Code subagent frontmatter (`model`, `effort` fields):
  [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents),
  [feature discussion](https://github.com/anthropics/claude-code/issues/43083)
- Original method: Nate Herk, ["How I Make Opus Think Like Fable (5 easy steps)"](https://www.youtube.com/watch?v=XTBWVVcF3Pk) / [article](https://x.com/nateherk/article/2074324638159581404)
- Skill implementation: [mrtooher/fable-mode](https://github.com/mrtooher/fable-mode)
- Related community work: [dilitS/op-fable](https://github.com/dilitS/op-fable),
  [Ken Huang — Claude Fable 5: What Changed](https://kenhuangus.substack.com/p/claude-fable-5-what-changed-and-how),
  [XDA — recreating Fable 5 with Opus and agent loops](https://www.xda-developers.com/recreated-fable-5-opus-agent-loops-close-stopped-missing-banned-model/)
