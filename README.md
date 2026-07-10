# efficient-fable-orchestrator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](CONTRIBUTING.md)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-5A45FF)](https://docs.claude.com/en/docs/claude-code/skills)

A [Claude Code](https://claude.com/claude-code) skill that turns a session into a
quota-conscious orchestrator. It plans at a high level and hands the actual
implementation work to other AI CLIs (Grok CLI, Codex CLI) you're already paying for,
instead of burning your Claude usage on the grind.

A quick note on the name: "Fable" is what I call the orchestrating Claude model in this
setup, the one doing the planning and delegating rather than the coding. If you've seen
that name elsewhere and wondered, that's it, and it's also the name of Anthropic's newest
model line as of mid-2026. This skill exists specifically to minimize *its* usage.

## Where this came from

I got the idea watching [Theo (t3.gg)'s "A proper guide to Fable 5"](https://youtu.be/8GRmLR__OGQ?si=g_xN55h9h6hSCsui).
He runs Fable almost entirely as an orchestrator: it scouts, plans, kicks off workflows,
reviews PRs, and rarely touches the actual diff itself. Watching that, the obvious next
question for me was: if the orchestrator barely codes, what's actually stopping me from
routing the coding part somewhere that isn't metered against my tightest plan? That
question became this skill.

## The actual goal, read this before you install it

This exists to protect one specific resource: my own Claude usage, specifically the
weekly quota tied to Fable, the model I use for orchestration. That's the primary,
confirmed benefit. Protecting the rolling 5-hour session window is a secondary,
best-effort byproduct, not a guarantee, because session windows on these plans are
typically blended across every model used within that session. A delegated subagent's
usage plausibly still counts against it even though it's a cheaper model.

The mechanism: Fable almost never writes code itself. It spawns short-lived, low-effort
Sonnet "driver" subagents whose only job is to shell out to an external CLI (Grok, Codex)
that does the actual reasoning-heavy work. That work happens entirely on infrastructure
outside Claude's own meter. I'm deliberately leaning on Grok (SuperGrok) and Codex
(ChatGPT Plus) because those plans currently have more headroom for me than my Claude
plan does. It's a trade I'm making on purpose, not a workaround I'm hiding.

This is not a universal efficiency tool, and I want to be upfront about that. It doesn't
make total token cost across every provider disappear, it moves cost off the resource
that's tightest for me onto resources that aren't. If your bottleneck looks different
from mine (you're not quota-constrained on Claude, or you don't have a Grok/ChatGPT
subscription sitting mostly idle), this skill probably isn't useful to you as-is.

## What it actually does

- Classifies work into 5 tiers by difficulty, from menial to flagged/ultra, and routes
  each tier to the cheapest sufficient delegate: Grok 4.5 or the GPT-5.6 family (Luna for
  fast/cheap work, Terra for deep reasoning), each with an explicit effort ceiling.
- Tier 0, genuinely trivial work, gets done directly, with no subagent spawned at all.
  That's not an oversight. Spawning any subagent has a large, measured fixed cost in a
  Claude Code environment (tool/skill catalog and session bootstrap overhead), and for a
  one-line fix that overhead dwarfs the task. See "What's actually proven" below for
  where that number comes from and why it isn't universal.
- Distills project rules (from `CLAUDE.md` or equivalent) into a single ground-rules file
  once per repo, so delegates get the constraints that matter without every driver prompt
  re-composing them from scratch.
- Every finished lane opens a PR. Nothing merges without explicit permission.
- Full protocol details, tier table, and CLI invocation specifics live in
  [`efficient-fable-orchestrator/SKILL.md`](efficient-fable-orchestrator/SKILL.md), the
  actual skill file Claude Code loads.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- [Grok CLI](https://x.ai) with an active SuperGrok (or equivalent) subscription, and/or
- [Codex CLI](https://github.com/openai/codex) with an active ChatGPT Plus (or equivalent)
  subscription
- `gh` CLI authenticated, if you want the issue/PR steps to actually run
- You don't need both delegates. The skill degrades gracefully to whichever is available,
  and reports plainly, not silently, when it has to.

**My own setup, for reference, roughly $150/month total:**

| Plan | Cost | What it backs |
|---|---|---|
| Claude Max 5x | ~$100/mo | Fable itself, the resource this skill exists to protect |
| ChatGPT Plus | ~$20/mo | Codex CLI (GPT-5.6 Luna / Terra delegates) |
| SuperGrok | ~$30/mo (often a free first week) | Grok CLI (Grok 4.5 delegate) |

That's the actual trade the skill's framing assumes: two ~$20-30/mo plans with more
headroom absorbing work that would otherwise eat into a ~$100/mo plan's tighter quota. If
your own subscription mix looks different, say you only have ChatGPT Plus, or your
bottleneck isn't Claude at all, the math changes. Re-derive which resource is actually
scarce for you before assuming this skill helps. See "Adapting the model routing to your
own setup" below.

## Install

```bash
mkdir -p ~/.claude/skills
cp -r efficient-fable-orchestrator ~/.claude/skills/
```

Then in a Claude Code session: *"work in efficient orchestrator mode"* (or "use the
efficient fable orchestrator skill" / "activate the orchestrator"). It's opt-in by design.
It won't change how normal sessions behave otherwise.

## Example

**User**: *"work in efficient orchestrator mode. There's a pricing bug in
`pricing.py`, bulk discounts aren't applying at the tier boundaries. Fix it."*

**Fable**: classifies this as Tier 3, a real bug, non-trivial logic, worth a second
opinion, rather than implementing it directly. Spawns one Sonnet driver subagent in an
isolated worktree, instructing it to shell out to `grok --reasoning-effort high` to fix
the bug, then a second driver to independently review the resulting diff via a separate
`grok --reasoning-effort max` call before anything gets proposed as done.

**What actually happened when this was run** (a real result from testing, not a
hypothetical): the implementing delegate fixed the stated boundary bug correctly. The
separate review-pass delegate, looking only at the diff with no memory of the first call,
caught two things the implementer missed: booleans were being silently accepted as
numeric discount input, and `NaN` wasn't being rejected. Both were real gaps in an
already-passing, already-tested diff.

**Driver's report back to Fable** (deliberately terse, this is the whole point of the
protocol):

```
- Status: complete, PR opened
- Delegate: Grok 4.5 (implement: high, review: max)
- PR: <link>, Closes #<issue>
- Review flagged 2 additional gaps beyond the stated bug (bool/NaN handling as discount
  input), both fixed in the same PR before opening it
- Nothing needs your judgment call; tests pass, diff is small
```

Fable never opened `pricing.py`. Its own context stayed almost entirely on "is this the
right scope, is this fix good enough to open a PR," not on writing or re-reading the
implementation.

## What's actually proven vs. what isn't

I built and iterated on this against direct, repeated testing, not just vibes. But I want
to be specific about what that testing did and didn't cover, since I'm publishing this
rather than just using it privately.

Verified by direct, repeated testing, including a real (throwaway, private) GitHub repo:

- The core delegation mechanism, Sonnet driver → Bash → Grok/Codex CLI → independent
  verification, run many times across Tier 0 through Tier 4, output inspected directly,
  not just trusted from the delegate's or driver's self-report.
- The Tier 0 direct-edit carve-out.
- Ground-rules-file distillation actually pulling the right project constraints, and the
  delegate actually following them (verified by reading the diff, not just running
  tests). This surfaced a real failure mode, task-specific notes leaking into the shared
  file, that got caught and fixed.
- Terse structured reporting back to the orchestrator.
- Tier 3/4 paths under real tasks: Grok 4.5 at `high` found and fixed a genuine
  operator-precedence bug; GPT-5.6 Terra at `xhigh` implemented a tiered-discount
  function correctly against spec.
- The second-opinion review step earns its cost, concretely. A separate Grok `max`
  review call caught two real gaps (booleans silently accepted as numeric input, NaN not
  rejected) in an already-passing, already-tested diff that the primary implementer
  missed.
- 4+ parallel lanes running concurrently, which surfaced a real gap: without
  `isolation: "worktree"` on every lane, concurrent drivers collide on shared filenames.
  That's now documented as effectively mandatory in the skill file, not optional.
- A real end-to-end GitHub flow: issue created, PR opened referencing it, correctly not
  auto-merged, verified independently against the actual repo, not just the driver's
  report.
- A real, reproducible Windows-specific Codex CLI bug (`CreateProcessAsUserW`, Windows
  error 5, see [openai/codex#26803](https://github.com/openai/codex/issues/26803) and
  [#22428](https://github.com/openai/codex/issues/22428), both open upstream as of this
  writing) and its confirmed workaround, reconfirmed across 4 independent invocations.

Exercised, but with an honest caveat: multi-round re-invocation (a lane that fails tests
and needs a second pass with re-supplied context) is implemented and was manually
verified to work. But three separate honest attempts to trigger it organically, including
one with a real, easy-to-miss Python gotcha deliberately left unmentioned in the task,
all resulted in the delegate getting it right on the first pass anyway, because it read
the existing tests itself. Read that as "capable delegates rarely need the retry path on
well-scoped work with tests to read against," not as "the mechanism doesn't work."

Not yet exercised: behavior when both delegates are degraded at once (rate-limited, not
just one absent), and very large fan-out (10+ concurrent lanes).

A number worth being skeptical of: the "roughly 150,000 to 180,000 tokens of fixed
overhead per subagent spawn" figure cited in the skill file is real, but it was measured
on my own machine, which has an unusually large number of MCP servers and skills
installed. Your own overhead may be meaningfully lower with a leaner setup. Recalibrate
rather than trusting that number as a constant.

## Adapting the model routing to your own setup

The tier table hardcodes specific model names and effort ceilings (Grok 4.5,
`gpt-5.6-luna`, `gpt-5.6-terra`) that reflect what I currently have access to and pay
for. This is the part of the skill that's specific to me, not a universal
recommendation, so swap it for whatever you actually subscribe to. The durable part is
the *shape* of the protocol (tiers, driver subagents, discover-before-invoke,
ground-rules file, terse reporting), not these exact strings.

To adapt it:

1. **Figure out which resource is actually scarce for you.** The whole skill is built
   around protecting one tight resource by spending on looser ones. If you're not
   quota-constrained on Claude, or you don't have a second or third AI subscription with
   spare headroom, this skill's core trade doesn't apply to you as-is. See "The actual
   goal" above before you invest time adapting it.
2. **Edit the "Tiers and routing" table** in
   [`efficient-fable-orchestrator/SKILL.md`](efficient-fable-orchestrator/SKILL.md).
   Swap in whatever delegate CLIs and models you actually have (Gemini CLI, a different
   OpenAI model tier, a local model via Ollama). Keep the shape: cheapest sufficient
   delegate per tier, an explicit effort ceiling per delegate so nothing accidentally
   burns your most expensive tier on menial work.
3. **Update "Known-good invocation shapes"** with your own CLI's actual flags. Don't
   trust mine. Run the "discover before you invoke" probe (`<cli> --help`) yourself
   first. The skill leans on this step specifically because flags and model names go
   stale fast. If you confirm something new, consider sending a PR (see Contributing).
4. **Update "My current setup"** at the bottom of `SKILL.md` with your own subscription
   mix and confirmed CLI versions. That section is deliberately marked "reference, not
   load-bearing," so getting it right doesn't require touching the protocol logic above
   it.
5. **Re-run a dry run** on a throwaway repo before trusting it on real work. The "What's
   actually proven" section above is a good checklist of what to personally re-verify
   (the Tier 0 carve-out, worktree isolation on parallel lanes, stateless-CLI context
   resupply), since none of that is guaranteed to transfer cleanly to a different CLI's
   actual behavior.

## Contributing

Model names and CLI flags drift, that's the most useful thing to send a PR for. See
[CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT, see [LICENSE](LICENSE).
