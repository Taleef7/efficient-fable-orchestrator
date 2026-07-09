# efficient-orchestrator

A [Claude Code](https://claude.com/claude-code) skill that turns a session into a
quota-conscious orchestrator: it plans at a high level and delegates actual
implementation work to other AI CLIs (Grok CLI, Codex CLI) you already pay for,
instead of burning your Claude usage on the grind.

## The actual goal — read this before you install it

This exists to protect **one specific resource**: my own Claude usage, specifically
the **weekly quota tied to the model I use for orchestration**. That's the primary,
confirmed benefit. Protecting the rolling 5-hour session window is a secondary,
best-effort byproduct — not a guarantee, because session windows on these plans are
typically blended across every model used within that session, so a delegated
subagent's usage plausibly still counts against it even though it's a cheaper model.

The mechanism: an orchestrating Claude session almost never writes code itself. It
spawns short-lived, low-effort Sonnet "driver" subagents whose only job is to shell
out to an external CLI (Grok, Codex) that does the actual reasoning-heavy work. That
work happens entirely on infrastructure outside Claude's own meter. I'm deliberately
leaning on Grok (SuperGrok) and Codex (ChatGPT Plus) because those plans currently have
more headroom for me than my Claude plan — this is a trade I'm making on purpose, not
a workaround I'm hiding.

**This is not a universal efficiency tool.** It is not free, and it doesn't make total
token cost across every provider disappear — it moves cost off the resource that's
tightest for me onto resources that aren't. If your bottleneck is different from mine
(e.g. you're not quota-constrained on Claude, or you don't have a Grok/ChatGPT
subscription), this skill probably isn't useful to you as-is.

## What it actually does

- Classifies work into 5 tiers by difficulty (menial → ultra/flagged) and routes each
  tier to the cheapest sufficient delegate — Grok 4.5 or the GPT-5.6 family (Luna for
  fast/cheap work, Terra for deep reasoning), each with an explicit effort ceiling.
- **Tier 0 (genuinely trivial work) is done directly, with no subagent spawned at all.**
  This isn't an oversight — spawning any subagent has a large, measured fixed cost in
  a Claude Code environment (tool/skill catalog + session bootstrap overhead), and for
  a one-line fix that overhead dwarfs the task. See "What's actually proven" below for
  where this number comes from and why it isn't universal.
- Distills project rules (from `CLAUDE.md` or equivalent) into a single ground-rules
  file once per repo, so delegates get the constraints that matter without every driver
  prompt re-composing them from scratch.
- Every finished lane opens a PR. Nothing merges without explicit permission.
- Full protocol details, tier table, and CLI invocation specifics live in
  [`efficient-orchestrator/SKILL.md`](efficient-orchestrator/SKILL.md) — that's the
  actual skill file Claude Code loads.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- [Grok CLI](https://x.ai) with an active SuperGrok (or equivalent) subscription, **and/or**
- [Codex CLI](https://github.com/openai/codex) with an active ChatGPT Plus (or equivalent)
  subscription
- `gh` CLI authenticated, if you want the issue/PR steps to actually run
- You don't need both delegates — the skill degrades gracefully to whichever is
  available, and reports plainly (not silently) when it has to.

## Install

```bash
mkdir -p ~/.claude/skills
cp -r efficient-orchestrator ~/.claude/skills/
```

Then in a Claude Code session: *"work in efficient orchestrator mode"* (or "use the
efficient orchestrator skill" / "activate the orchestrator"). It's opt-in by design —
it won't change how normal sessions behave otherwise.

## What's actually proven vs. what isn't

I built and iterated on this against direct, repeated testing, not just vibes — but I
want to be specific about what that testing did and didn't cover, since I'm publishing
this rather than just using it privately.

**Verified by direct, repeated testing:**
- The core delegation mechanism (Sonnet driver → Bash → Grok/Codex CLI → independent
  verification) — run multiple times, output inspected directly, not just trusted from
  the delegate's or driver's self-report.
- The Tier 0 direct-edit carve-out.
- Ground-rules-file distillation actually pulling the right project constraints and the
  delegate actually following them (verified by reading the diff, not just running tests).
- Terse structured reporting back to the orchestrator.
- A real, reproducible Windows-specific Codex CLI bug (`CreateProcessAsUserW` /
  Windows error 5 — see [openai/codex#26803](https://github.com/openai/codex/issues/26803),
  [#22428](https://github.com/openai/codex/issues/22428), both open upstream as of this
  writing) and its confirmed workaround, both documented in the skill file.

**Not yet exercised at the time of first publishing** (see the skill's own commit
history / issue tracker for current status — I'm actively closing these gaps):
- Tier 3/4 (high-effort) paths under real task pressure, not just trivial probes.
- Multi-round re-invocation (a lane that fails tests and needs a second pass with
  re-supplied context) — designed, not yet proven under a real failure.
- Real GitHub issue/PR creation against a live repo.
- The "second delegate reviews the first delegate's diff" review step.
- More than a couple of parallel lanes at once.
- Behavior when *both* delegates are degraded (rate-limited, not just absent).

**A number worth being skeptical of:** the "~150,000-180,000 tokens of fixed overhead
per subagent spawn" figure cited in the skill file is real, but it was measured on my
own machine, which has an unusually large number of MCP servers and skills installed.
Your own overhead may be meaningfully lower with a leaner setup — recalibrate rather
than trusting that number as a constant.

## Adapting this for yourself

The tier table hardcodes specific model names and effort ceilings (Grok 4.5,
`gpt-5.6-luna`, `gpt-5.6-terra`) that reflect what I currently have access to and what
those models were called as of mid-2026. Model lineups and names change. If you fork
this, expect to update the routing table and the confirmed CLI invocation flags — the
skill file's own "discover before you invoke" step is there specifically because CLI
flags and model names drift, and you should treat that principle as the durable part,
not the specific strings.

## License

MIT — see [LICENSE](LICENSE).
