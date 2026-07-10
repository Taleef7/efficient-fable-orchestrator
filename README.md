# efficient-fable-orchestrator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](CONTRIBUTING.md)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-5A45FF)](https://docs.claude.com/en/docs/claude-code/skills)

A [Claude Code](https://claude.com/claude-code) skill that turns a session into a
quota-conscious orchestrator. It plans at a high level and hands the actual
implementation work to other AI CLIs (Grok CLI, Codex CLI) you're already paying for,
instead of burning your Claude usage on the grind.

"Fable" is what I call the orchestrating Claude model here, the one planning and
delegating instead of coding. It's also Anthropic's current model-line name. This skill
exists to minimize *its* usage specifically.

## Where this came from

I got the idea watching [Theo (t3.gg)'s "A proper guide to Fable
5"](https://youtu.be/8GRmLR__OGQ?si=g_xN55h9h6hSCsui). He runs Fable almost entirely as an
orchestrator, scouting, planning, kicking off workflows, reviewing PRs, rarely touching
the diff itself. Watching that, the obvious question was: if the orchestrator barely
codes, what's stopping me from routing the coding part somewhere that isn't metered
against my tightest plan? This skill is that question, answered.

## The goal

Protect one specific resource: my weekly Claude quota, tied to Fable. That's the
confirmed benefit. Protecting the rolling 5-hour session window is a secondary,
best-effort byproduct, not a guarantee, since session windows tend to blend usage across
every model in a session.

The mechanism: Fable spawns short-lived, low-effort Sonnet "driver" subagents whose only
job is shelling out to Grok CLI or Codex CLI, which do the actual reasoning-heavy work on
infrastructure outside Claude's meter entirely. I'm leaning on Grok and Codex on purpose,
because those plans have more headroom for me than Claude does right now.

**This isn't a universal efficiency tool.** It moves cost off my tightest resource onto
looser ones, it doesn't make total cost vanish. If you're not quota-constrained on
Claude, or don't have a Grok/ChatGPT subscription sitting mostly idle, this probably
isn't useful to you as-is.

## What it does

- Classifies work into 5 tiers by difficulty and routes each to the cheapest sufficient
  delegate, with an explicit effort ceiling per tier.
- Tier 0, genuinely trivial work, gets done directly. No subagent spawn. Spawning has a
  large, measured fixed cost (session bootstrap overhead) that dwarfs a one-line fix.
- Distills project rules (`CLAUDE.md` or equivalent) into one ground-rules file per repo,
  so delegates get the constraints without every prompt re-composing them.
- Every finished lane opens a PR. Nothing merges without explicit permission.

Full protocol, tier table, and CLI invocation details live in
[`efficient-fable-orchestrator/SKILL.md`](efficient-fable-orchestrator/SKILL.md), the
actual file Claude Code loads.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- [Grok CLI](https://x.ai) (SuperGrok) and/or [Codex CLI](https://github.com/openai/codex)
  (ChatGPT Plus). You don't need both; it degrades gracefully to whichever is available.
- `gh` CLI authenticated, if you want the issue/PR steps to run.

**My own setup, roughly $150/month total:**

| Plan | Cost | Backs |
|---|---|---|
| Claude Max 5x | ~$100/mo | Fable, the resource this skill protects |
| ChatGPT Plus | ~$20/mo | Codex CLI (GPT-5.6 Luna / Terra) |
| SuperGrok | ~$30/mo (often free week 1) | Grok CLI (Grok 4.5) |

Two cheaper, looser plans absorbing work that would otherwise eat the tight one. If your
mix looks different, re-derive which resource is actually scarce for you (see Adapting,
below) before assuming this helps.

## Install

```bash
mkdir -p ~/.claude/skills
cp -r efficient-fable-orchestrator ~/.claude/skills/
```

Then, in a Claude Code session: *"work in efficient orchestrator mode."* It's opt-in;
normal sessions behave exactly as before until you ask for it by name.

## Example

**User**: *"work in efficient orchestrator mode. There's a pricing bug in `pricing.py`,
bulk discounts aren't applying at the tier boundaries. Fix it."*

**Fable**: classifies this as Tier 3 (a real bug, worth a second opinion) and delegates
rather than implementing it directly. One driver shells out to `grok --reasoning-effort
high` to fix it; a second, independent driver reviews the diff via a separate `grok
--reasoning-effort max` call before anything is proposed as done.

**What actually happened** (a real test result, not a hypothetical): the fix was correct.
The separate review call, seeing only the diff, caught two things the implementer missed:
booleans silently accepted as numeric discount input, and `NaN` not rejected. Both got
fixed in the same PR.

**Driver's report back** (deliberately terse, that's the whole point):

```
- Status: complete, PR opened
- Delegate: Grok 4.5 (implement: high, review: max)
- PR: <link>, Closes #<issue>
- Review flagged 2 additional gaps beyond the stated bug (bool/NaN handling), fixed
  before opening
- Nothing needs your judgment call; tests pass, diff is small
```

Fable never opened `pricing.py`.

## What's actually proven

I tested this repeatedly, not just vibed it, and I want to be specific about what that
did and didn't cover. Short version: the core delegation loop, the Tier 0 carve-out, the
ground-rules distillation, second-opinion review, 4+ parallel lanes, a full GitHub
issue/PR flow, and a real Windows Codex sandbox bug and its fix are all verified by
repeated direct testing, not self-report. Multi-round re-invocation is implemented and
manually verified but never organically triggered across three honest attempts. Behavior
under double-delegate failure and 10+ concurrent lanes is untested.

Full accounting, including the exact failure modes found and fixed along the way, in
[TESTING.md](TESTING.md).

## Adapting to your own setup

The tier table hardcodes my models and plans. That part is specific to me, not a
recommendation, swap it for whatever you actually pay for. The durable part is the
*shape*: tiers, driver subagents, discover-before-invoke, a ground-rules file, terse
reporting.

1. Figure out which resource is actually scarce for *you* first (see The goal, above).
   If nothing is, this skill's core trade doesn't apply.
2. Edit the tier table and invocation flags in
   [`SKILL.md`](efficient-fable-orchestrator/SKILL.md) for your own CLIs, verified via
   each CLI's own `--help`, not copied from mine.
3. Dry-run it on a throwaway repo before trusting it on real work.

## For AI agents

If you're an LLM being asked to install, adapt, or extend this on someone's behalf, read
[AGENTS.md](AGENTS.md) first. Short version: don't wire this to auto-trigger, and don't
assume my model names or flags still exist, rerun the discovery step yourself.

## Contributing

Model names and CLI flags drift, that's the most useful thing to send a PR for. See
[CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT, see [LICENSE](LICENSE).
