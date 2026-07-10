# What's actually proven vs. what isn't

I built and iterated on this against direct, repeated testing, not just vibes. This page
is the honest accounting of what that testing did and didn't cover, since I'm publishing
this rather than just using it privately. The short version lives in the main
[README](README.md#whats-actually-proven); this is the full version.

## Verified by direct, repeated testing

Including a real (throwaway, private, since deleted) GitHub repo, not just local dry runs.

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

## Exercised, but with an honest caveat

Multi-round re-invocation (a lane that fails tests and needs a second pass with
re-supplied context) is implemented and was manually verified to work. But three separate
honest attempts to trigger it organically, including one with a real, easy-to-miss Python
gotcha deliberately left unmentioned in the task, all resulted in the delegate getting it
right on the first pass anyway, because it read the existing tests itself. Read that as
"capable delegates rarely need the retry path on well-scoped work with tests to read
against," not as "the mechanism doesn't work."

## Not yet exercised

- Behavior when both delegates are degraded at once (rate-limited, not just one absent).
- Very large fan-out (10+ concurrent lanes).

## A number worth being skeptical of

The "roughly 150,000 to 180,000 tokens of fixed overhead per subagent spawn" figure cited
in the skill file is real, but it was measured on my own machine, which has an unusually
large number of MCP servers and skills installed. Your own overhead may be meaningfully
lower with a leaner setup. Recalibrate rather than trusting that number as a constant.
