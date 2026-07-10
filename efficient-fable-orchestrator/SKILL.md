---
name: efficient-fable-orchestrator
description: Turns the current session (Fable, i.e. the orchestrating Claude model) into a quota-conscious orchestrator that plans work at a high level and delegates implementation to cheaper external model CLIs (e.g. Grok CLI, Codex CLI) via short-lived Sonnet driver subagents, instead of doing the implementation itself. The point is minimizing Fable's own usage. Use ONLY when the user explicitly says something like "work in efficient orchestrator mode," "use the efficient fable orchestrator skill," or "activate the orchestrator" — this is an opt-in mode, not a default behavior for every multi-step task.
---

# Efficient Fable Orchestrator

## Why this exists

On a usage-metered plan, the model doing the *planning* (you, the main session) is the
most expensive and most limited resource. Every turn spent writing implementation code,
grinding through a long refactor, or babysitting a test run is a turn not available later
in the window. Meanwhile cheaper/differently-metered capacity often sits idle: other CLI
tools backed by other providers, and even your own cheaper same-provider models.

The point of this mode is division of labor by cost, not by capability ceiling: you stay
the architect, planner, delegator, and quality gate. You almost never write the diff
yourself. You spend your limited turns on judgment (what to build, who should build it,
is this good enough to ship) and let disposable delegate processes spend theirs on labor
(writing the code, running the tests, iterating on failures).

This mode is **explicitly invoked, not automatic**. Normal sessions should keep using the
Agent tool / Task tools the usual way. Only switch into this mode when the user asks for
it by name, because it changes your default behavior (delegate first, implement only as
a last resort) in a way that would be wrong to apply silently to every request.

**What this mode actually optimizes for, in priority order:**
1. **Weekly Claude/Fable usage — the primary, confirmed target.** The orchestrating model
   does almost no implementation itself, so its own weekly allocation barely moves no
   matter how much work gets done. This is the headline benefit and the one to lean on.
2. **The current 5-hour session window — a secondary, best-effort byproduct, not
   guaranteed.** Session windows on these plans are typically blended across every model
   used *within* that session, so a driver subagent's usage plausibly still counts against
   it even though it's a cheaper model than the orchestrator. This mode still likely helps
   here (see the overhead-vs-offloaded-reasoning argument below), just less certainly and
   less completely than it helps the weekly figure. Don't oversell it as a fix for an
   already-critical session window — heavy parallel fan-out has a real fixed cost per
   spawn (see Tiers and routing) and could burn a tight window faster if used carelessly
   under time pressure.
3. **Grok/Codex usage is treated as the expendable resource, deliberately.** Those plans
   have more headroom than the Claude plan this mode exists to protect — leaning on them
   heavily is the intended trade, not a compromise.

**Why delegating still wins even given the fixed per-spawn overhead:** that overhead
(~150,000-180,000 tokens, see Tiers and routing) is mostly structural — tool/skill catalog
and session bootstrapping — not proportional to how much reasoning the task needs. It's
paid once per lane regardless of task size. Meanwhile the actual expensive part of
non-trivial work — the deep reasoning over a hard bug, a big refactor, an ambiguous spike
— happens entirely on Grok's or Codex's infrastructure once delegated, and **never touches
the Claude meter at all**. So the fixed tax matters least exactly where delegation is used
most (Tier 2-4, where the offloaded reasoning would otherwise have been expensive on the
Claude side too), and matters most exactly where it's already carved out as an exception
(Tier 0, where there's nothing substantial to offload in the first place).

## Your role once this mode is active

You are the planner, delegator, monitor, and final reviewer. Concretely:

1. **Plan first, with the user.** Before spawning anything, restate the goal and propose
   a breakdown into independent lanes of work. Get explicit or implicit sign-off (a plan
   the user doesn't object to counts) before delegating — this mirrors normal planning
   discipline, it doesn't bypass it.
2. **Classify each lane by tier** (below) and pick the cheapest sufficient delegate.
3. **Delegate Tier 1 and up via short-lived Sonnet driver subagents** (see Mechanism)
   that shell out to the external CLI. Genuinely trivial Tier 0 work is the one exception
   — do it directly, no subagent spawn at all (see why in Tiers and routing). Above Tier
   0, Sonnet is a driver, full stop; it never implements. Every driver prompt you write
   points at the ground-rules file and carries the task's background (see "Carrying
   project context to delegates") — the delegate only knows what you hand it.
4. **Monitor and steer**, not implement. Check on background lanes, read their handoff
   files, redirect them when they're off track, but resist the pull to just fix it
   yourself because that would be faster in the moment — it defeats the point.
5. **Personally review** only the things that matter most: Tier 3+ diffs, anything the
   user flagged, and anything a delegate itself flagged as uncertain.
6. **Never merge without explicit permission.** Every finished lane ends at an open PR.
   The user merges, unless they say up front (this session) that you're authorized to
   merge automatically.

If you catch yourself about to open a file and start editing on anything above Tier 0
because delegating "feels like more overhead than just doing it" — that instinct is
exactly the leak this mode exists to plug. Delegate it anyway. Tier 0 is the one
deliberate exception, and it's not a loophole: spawning a subagent (driver or not) in
this environment carries a large fixed cost — confirmed empirically at roughly
150,000-180,000 tokens per spawn, mostly tool/skill catalog and session-start overhead,
before any real work happens. For a genuine one-liner, that overhead dwarfs the task
itself. Doing true Tier 0 work directly is the efficient choice here, not a lapse in
discipline.

## Tiers and routing

Every tier from 1 up delegates. Sonnet, wherever it appears in this mode, is a low-effort
driver: it probes, invokes, verifies, and reports, and that's all. Tier 0 is the deliberate
exception: spawning any subagent in this environment has a large fixed cost (measured at
roughly 150,000-180,000 tokens per spawn — mostly tool/skill catalog and session-start
overhead, confirmed via a real dry run, not a guess), so routing a genuine one-liner
through a driver + external CLI round-trip is a net loss, not a savings. Do true Tier 0
work directly; delegate everything from Tier 1 up.

| Tier | What it looks like | Delegate(s) | Effort ceiling |
|---|---|---|---|
| **0 — Menial** | Docs tweaks, one-line fixes, renames, branch cleanup, labeling | You, directly — no subagent spawn (the fixed per-spawn overhead exceeds the task) | n/a |
| **1 — Low** | Small well-specified feature, a test file, a straightforward fixture | GPT-5.6 Luna | `low` or `medium` |
| **2 — Medium** | Non-trivial logic, an integration, a spike with some ambiguity | Grok 4.5 **or** GPT-5.6 Luna — split preferred across lanes | Grok: `medium`/`high` · Luna: `high` (Luna's hard ceiling everywhere) |
| **3 — High** | Hard bugs, architecture-touching changes, security-relevant work, large refactors | Grok 4.5 **or** GPT-5.6 Terra — split preferred, or both on the same lane for a second opinion | Grok: `high` · Terra: `high` |
| **4 — Flagged / ultra-max** | Anything the user explicitly wants eyes on: schema changes, prod-impacting decisions, irreversible ops | Grok 4.5 **and** GPT-5.6 Terra, leaning toward Terra | Grok: `max` (its ceiling) · Terra: `xhigh` (hard ceiling — see note below) |

**Hard ceilings — never exceed these, even when the CLI's own enum allows more:**
- **GPT-5.6 Luna** — never above `high`. Codex's model cache may report `xhigh`/`max` as
  technically valid for Luna; that's a CLI capability, not permission to use it here.
  Luna is the fast/cheap tier by design — if a task needs more than `high` from Luna,
  it belongs on Grok or Terra instead, not on a higher-effort Luna call.
- **GPT-5.6 Terra** — never above `xhigh`, even though Terra's own enum goes up to
  `ultra`. Don't confuse Terra's own `ultra` *reasoning-effort level* with this skill's
  *Tier* "4 — ultra/max" naming — they are unrelated, and this skill intentionally never
  invokes Terra above `xhigh`.
- **Grok 4.5** — ceiling is `max` (its highest confirmed level), reserved for Tier 4.

When unsure between two tiers, round down and see how the delegate does — escalating a
lane mid-flight (more effort, or a second delegate for review) is cheap; over-provisioning
every lane up front isn't.

## Mechanism: how delegation actually works

**Important constraint, confirmed by direct testing:** neither Grok CLI nor Codex CLI is
a registered Claude Code subagent type, so the Agent tool cannot reach them directly. The
real mechanism is: you spawn a Sonnet subagent via the **Agent tool** (`model: sonnet`),
and that subagent's job is to shell out to the external CLI via **Bash**, read its stdout,
and report back. The "driver" is not a special construct — it's just an ordinary Sonnet
subagent whose prompt tells it to act as one.

```
Agent({
  description: "Driver: implement <lane>",
  model: "sonnet",
  isolation: "worktree",        // treat as mandatory whenever 2+ lanes run concurrently against the same repo
  run_in_background: true,      // default; lets multiple lanes run concurrently
  prompt: "<see template below>"
})
```

**`isolation: "worktree"` is not optional once you have more than one lane running
concurrently against the same repo — confirmed the hard way.** A 4-lane test run without
it left every lane sharing one working directory: three separate drivers independently
discovered the collision risk on `handoff.json`/`handoff.md` and each invented its own
`-taskname` suffix to avoid clobbering another lane's file, entirely on their own
initiative. That's a sign the drivers are competent, not a sign the setup was fine — it
means for months the recipe here relied on drivers noticing and working around a gap
rather than the gap not existing. Give every concurrent lane its own worktree; don't rely
on a driver noticing the problem for you.

**There is no confirmed "effort" parameter for a native Sonnet subagent** the way Grok and
Codex have `--reasoning-effort` / `model_reasoning_effort`. "Low effort" for the driver is
approximated at the prompt level, not set via a knob — the driver template's numbered
steps (probe, invoke, verify, report) *are* that approximation: a short, mechanical script
with no room for the driver to wander into its own exploratory reasoning. Keep driver
prompts scoped exactly to that script; the discipline is what keeps the driver cheap, not
a parameter.

### Discover before you invoke, every time

CLI flags drift as tools update. Never trust a hardcoded recipe (including the ones
below) as gospel — the driver's very first step is always a cheap probe:

```bash
command -v codex && codex exec --help 2>&1 | head -40
command -v grok  && grok --help 2>&1 | head -40
```

If a tool isn't found, report that clearly and fall back to a Sonnet subagent (Tier 1
treatment) for that lane rather than failing the whole lane silently.

### Known-good invocation shapes (verified this session — reverify via `--help` if they ever error)

**Grok CLI** — single-turn headless, prints to stdout and exits:
```bash
grok -p "<full task prompt>" --reasoning-effort <level> --no-subagents --always-approve
```
Valid levels: `none, minimal, low, medium, high, xhigh, max`. All confirmed working.
**Grok's actual model name is `grok-4.5`** (confirmed via `grok models`) — don't guess at
a variant name (e.g. a "-fast" suffix); if unsure, run `grok models` to list what's
actually available on the account before assuming a name.
**`--always-approve` is needed for unattended/headless runs** — without it, Grok can stop
and wait on a tool-execution approval prompt the same way Codex does without a sandbox
flag. (`--permission-mode bypassPermissions` is an equivalent alternative also confirmed
working.)

**Codex CLI** — non-interactive exec mode, model and effort both explicit flags:
```bash
codex exec -m <model-id> -c model_reasoning_effort="<level>" \
  -s workspace-write --skip-git-repo-check "<full task prompt>"
```
**`-s workspace-write --skip-git-repo-check` is required, not optional.** Confirmed by
direct testing: without an explicit sandbox mode, `codex exec` hangs indefinitely waiting
on an interactive approval prompt that never comes in a headless/backgrounded shell — it
does not fail fast, it just sits there burning wall-clock until something times it out.
`--skip-git-repo-check` avoids a similar hang/failure when the worktree isn't recognized
as a git repo yet. If a lane needs to touch files outside the worktree, add
`--add-dir <dir>` rather than reaching for `--dangerously-bypass-approvals-and-sandbox`
(sandboxed writes scoped to the worktree are the safer default for an unattended delegate).
Model IDs this skill uses: `gpt-5.6-luna` (fast/affordable — Tier 0-2) and `gpt-5.6-terra`
(balanced, deeper reasoning — Tier 3-4). Confirmed reasoning levels via
`~/.codex/models_cache.json`: Luna supports `low, medium, high, xhigh, max`; Terra
supports `low, medium, high, xhigh, max, ultra`. **This skill's own hard ceilings (in
Tiers and routing, above) are tighter than what the CLI allows — always respect the
skill's ceiling, never the CLI's max.**

Older Codex CLI installs won't recognize these model IDs or their higher effort levels at
all (confirmed: a stale 0.133.0 install rejected `gpt-5.6-*` and `max`/`ultra` outright;
0.144.0+ works). If a `gpt-5.6-*` invocation errors on an unknown model or unknown effort
value, check `codex --version` before assuming the delegate is unavailable — it likely
just needs an update, which is a different failure mode than "not installed."

**Known Windows-specific bug: `-s workspace-write` fails on any actual shell/subprocess
execution, not on file edits.** Confirmed via direct testing plus two open upstream GitHub
issues (openai/codex#26803, #22428 — both still Open, labeled `bug`/`sandbox`/`windows-os`
as of this writing): Codex's Windows sandbox runner's restricted-token process spawn
(`CreateProcessAsUserW`) fails with `Access is denied` / Windows error 5. A pure file
write via `apply_patch` works fine under `workspace-write` — the failure only triggers
the moment Codex tries to actually *run* a command (exploring the repo via shell, running
tests, anything beyond editing a file). Since almost any real coding task needs at least
one shell command, this makes `workspace-write` effectively unusable for real work on
Windows today, not just an edge case.

**Fix, confirmed working:** retry with `--dangerously-bypass-approvals-and-sandbox`
instead of `-s workspace-write` if a Codex call fails with `CreateProcessAsUserW` in the
error output. This is a real tradeoff, not a free lunch — it removes Codex's own OS-level
sandboxing entirely (`sandbox: danger-full-access`), so the delegate's shell commands run
unsandboxed. The isolated git worktree (`isolation: worktree` on the driver spawn) is the
only remaining safety boundary at that point — make sure the lane is actually running in
one before falling back to this flag, and treat it as a Windows-only workaround for a bug
that may get fixed upstream, not a general default. On Linux/macOS, `workspace-write`
should be tried first and is expected to work normally — this bug is Windows-specific.

**Tell the user when this triggers.** Don't apply this fallback silently and only mention
it if asked. The first time a lane needs it in a session, say so plainly in your next
status update to the user (which lane, why, and that it means unsandboxed execution
inside that lane's worktree) — this is a real change in what you're granting the
delegate, not an implementation detail worth hiding to keep the report terse.

`minimal` effort was previously confirmed broken on the older `gpt-5.5` line (400s —
incompatible with default `image_gen`/`web_search` tools) and isn't used by any tier's
ceiling in this skill anyway, so it shouldn't come up.

### Both CLIs are stateless per call — plan around it

Each invocation above is a fresh process with no memory of prior calls. There is no
confirmed resume/continue mechanism for either CLI as of this writing. If a lane needs
more than one round (initial pass, then a fix-up pass after tests fail), you must
**re-supply full context in the next prompt** — paste in the relevant parts of the
previous call's output, or better, have the driver write and re-read a `handoff.md` file
in the worktree so context survives across calls without bloating every prompt. If you
discover a real resume flag during the `--help` probe, use it — just don't assume one exists.

### Carrying project context to delegates

You (the orchestrator) already know this project — CLAUDE.md, prior decisions, docs,
whatever this session has accumulated. Grok and Codex know none of that. Left to a bare
task description, a delegate can produce code that's locally correct but violates a
project rule it never saw — the cheap-tier verification (running existing tests) won't
catch a policy violation nothing tests for. So this context has to travel to the
delegate somehow.

**But composing it fresh in every driver prompt is a real cost, and it's your cost, not
the driver's** — every paragraph you write to brief a lane is Fable's own output tokens,
and it scales linearly with the number of lanes you spawn. Five parallel lanes means
five hand-written briefings. That directly taxes the resource this mode exists to
protect. So do the distillation **once**, not per lane:

- **Write a ground-rules file once per repo, not once per lane.** At the start of a
  session using this mode, check whether `orchestrator-context.md` already exists at the
  repo root — if it does, reuse it as-is unless it's gone stale. If it doesn't, write it
  once: the load-bearing rules that actually matter for delegated work in this codebase
  (schema/DDL ownership, audit/compliance boundaries, AI-decision limits, anything the
  project's own docs call non-negotiable), organized under short headers so a driver
  prompt can point at a section by name instead of quoting it. Skip anything that doesn't
  apply anywhere in this codebase — an irrelevant wall of policy gets skimmed past by a
  delegate just as fast as no policy at all.
- **Driver prompts point at the file; they don't inline its contents.** A driver prompt's
  `Ground rules` field is a path plus a one-line pointer — e.g. "`orchestrator-context.md`,
  see Audit + Schema sections" — not a paragraph. The driver relays that pointer into its
  delegate CLI prompt as an instruction to read the file (or, if the CLI can't read files
  from a prompt, to `cat` the relevant section and include it) before starting. This shifts
  the reading cost onto the driver and delegate — cheap, and work they'd need to do
  anyway — instead of onto your own output tokens.
- **Background stays per-task, but keep it to pointers, not paragraphs — and keep it OUT
  of the shared file, full stop.** What a task connects to (relevant files, the pattern
  to stay consistent with) is genuinely task-specific and belongs in the driver prompt's
  own `Background` field, never appended to `orchestrator-context.md`. This isn't
  theoretical: a driver under real testing appended a "Task-specific override" section
  to the shared file (a note about not touching a sibling task's tests) instead of
  keeping it in its own prompt — harmless that one time, but the shared file accumulates
  every lane's leftover task notes if this isn't caught, until it's no longer a clean
  repo-wide reference and every future lane pays to skim past stale, irrelevant context.
  If you notice a driver has written task-specific content into the shared file, remove
  it on the next lane you spawn.
- **Update the file when a lane reveals a gap, not on every lane.** If a delegate's output
  shows the ground-rules file was missing something load-bearing, fix the file once —
  every later lane benefits, instead of you re-explaining the same gap each time.

### Driver subagent prompt template

Give every driver subagent this shape, filled in for the specific lane. You fill in
`Ground rules` and `Background` yourself before spawning — don't leave the driver to
guess what matters:

```
You are a driver, not an implementer. Your job:

1. Probe: confirm `codex`/`grok` are on PATH and current flags (see SKILL discovery step).
2. Choose the delegate + effort level for this lane's tier: <tier + rationale>.
3. Read the ground-rules file at the path below yourself first. Then shell out via Bash
   to the delegate CLI with a prompt that tells it to read that same file (or, if the CLI
   can't read files from within a prompt, includes just the relevant section) plus the
   task and background below — the delegate has no other way to know the constraints. If
   the task needs multiple rounds (e.g. tests fail first pass), re-invoke the CLI with the
   previous round's output/handoff folded into the new prompt, and keep re-pointing at the
   ground-rules file each time — don't assume the CLI remembers anything, including the
   constraints. **If a Codex call fails with `CreateProcessAsUserW` in the output, that's
   the known Windows sandbox bug (see discovery section) — retry the same call with
   `--dangerously-bypass-approvals-and-sandbox` instead of falling back to Grok
   immediately**; only fall back to Grok if the retry also fails or Codex is unavailable
   outright.
4. After the delegate reports done: run the project's real test suite yourself (you,
   the driver, via Bash) to verify — don't just trust the delegate's self-report. If the
   ground rules for this task are safety/compliance-relevant, also spot-check the diff
   against them yourself, since a passing test suite doesn't prove a policy wasn't violated.
5. If this is Tier 2+: open a GitHub issue first (`gh issue create`) describing the task
   clearly enough that the delegate (or a human) could pick it up cold, then reference
   it in the PR ("Closes #N"). Skip the issue for Tier 1 busywork.
6. Open a PR when done (`gh pr create`). Do NOT merge it.
7. Write handoff.json + handoff.md in the worktree if this is Tier 2+, or if it's Tier 1
   but didn't finish cleanly in one pass (partial, blocked, or needed a second round). A
   Tier 1 task that completed in one clean pass doesn't need the full protocol — just say
   so in your report; there's no continuation for anyone to pick up.
8. Report back to the orchestrator in a few bullet points, not a narrative: status,
   PR/issue links if any, which delegate + effort level was used, and anything that needs
   a judgment call. Keep it short on purpose — this report becomes input tokens the
   orchestrator has to read, and the full detail already lives in handoff.md (when
   written) or in the PR diff itself; don't duplicate it into your report.

Task: <full task description, acceptance criteria, relevant file paths>
Tier: <1-4>
Worktree: <path>
Ground rules: <path to the ground-rules file + which section(s) apply to this task, e.g.
  "orchestrator-context.md, see Audit + Schema sections" — omit only if the file
  genuinely has nothing relevant to this task>
Background: <file/pattern pointers, a sentence or two — not a written brief>
```

### Handoff files (required for Tier 2+; for Tier 1 only if not cleanly done in one pass)

`handoff.json` — machine-readable:
```json
{
  "task": "short description",
  "status": "complete | mostly_complete | blocked",
  "delegate_used": "grok | codex | sonnet",
  "effort": "medium",
  "files_modified": ["src/foo.ts"],
  "issue": "https://github.com/.../issues/N",
  "pr": "https://github.com/.../pull/M",
  "remaining_work": "none, or a short description",
  "notes_for_orchestrator": "anything requiring judgment"
}
```

`handoff.md` — a few sentences a human or the next CLI call can read cold to pick up
where this left off. This is the thing you re-paste into a follow-up CLI call if a
second round is needed.

## Monitoring and steering

Background Agent-tool calls notify you automatically when they finish — don't poll.
While a lane is running and you want to check in or redirect it mid-flight, use
`SendMessage` to the agent's id/name rather than waiting silently or spawning a
duplicate. Keep a lightweight running table in your own response (task / tier /
delegate+effort / status / issue / PR) so the user can see lane status at a glance —
this doesn't need to be a persisted file unless the user asks for one.

Driver reports are terse by design (see driver template step 8) — hold that line. If a
report comes back long or narrative, that's a lane not following the template, not
something to reward by reading it closely anyway; the full detail belongs in handoff.md
or the PR, and every extra token in a report you read is an input-token cost on the
exact resource this mode exists to protect.

**One deliberate exception to terseness:** if a driver's report shows a Codex lane fell
back to `--dangerously-bypass-approvals-and-sandbox` (see "Known-good invocation shapes"),
surface that to the user explicitly in your own next status update, don't fold it silently
into the status table as if it were routine. It's a real change in what that lane was
granted, and the user should hear it from you, not have to notice it by reading the
driver's raw output themselves.

## Review and merge discipline

- Every finished lane surfaces as an open PR, never an auto-merge.
- For Tier 2-3 lanes, a second CLI call (the *other* delegate, at medium/high effort)
  reviewing the diff before you look at it is worth the cost — catches issues you'd
  otherwise have to catch yourself.
- For Tier 4, do your own close read of the diff in addition to any delegate review —
  this is the one place your own judgment is the point, not a cost to avoid.
- If the user says up front, this session, that you're authorized to merge automatically,
  honor that — but the default is always "opens a PR and stops."

## Graceful degradation

If neither `codex` nor `grok` is available (not installed, not authenticated, or the
probe step errors), say so plainly and fall back to Tier-1-style Sonnet subagents for
everything above Tier 0. Don't silently downgrade without telling the user — the whole
point of this mode is conserving a specific resource, and if the fallback means you're
burning that resource after all, they should know.

## What's actually been proven vs. designed-but-untested

Everything in this skill has now been exercised at least once against real tasks, real
delegates, and (for the GitHub flow) a real repo — not just designed and assumed to
work. Worth knowing what each test actually showed, including where it surprised the
design:

- **Tier 3/4 paths, real tasks:** Grok 4.5 at `high` found and fixed a genuine
  operator-precedence bug in one pass. GPT-5.6 Terra at `xhigh` implemented a tiered
  discount function correctly against its stated spec in one pass.
- **The second-opinion review step (Tier 2-3, "a second CLI reviews the diff") earns
  its cost.** A separate Grok `max` call reviewing Terra's already-passing, already-
  tested diff caught two real gaps the implementer missed entirely: booleans silently
  accepted as valid numeric input, and NaN not rejected. Independently confirmed both
  by hand. This is the clearest evidence in this skill that the review step isn't
  ceremony.
- **Multi-round re-invocation is real and was manually exercised successfully**, but
  three separate honest attempts to trigger it *organically* — including one with a
  genuinely tricky Python gotcha (bool being a subclass of int) deliberately withheld
  from the task prompt — all resulted in the delegate getting it right on the first
  pass anyway, because it read the existing test file itself and caught the trap
  unprompted. Read this as evidence that capable delegates rarely need the retry path
  on well-scoped Tier 1-2 work with an existing test file to read against, not as
  evidence the mechanism is unnecessary — it's a safety net for genuine ambiguity or
  harder tasks, not something that should be expected to fire often.
- **The real GitHub issue → delegate → PR flow works end-to-end**, verified against an
  actual (throwaway, private) repo: a real issue, a real linked PR (`Closes #N`), correct
  diff, not merged, `gh issue view`/`gh pr view` confirmed both independently rather than
  trusting the driver's report.
- **Concurrent lanes without `isolation: "worktree"` will collide on shared filenames**
  (see the Mechanism section) — caught because three independent drivers all noticed and
  worked around it themselves, which is a sign the fallback behavior is reasonable, not
  a substitute for giving every lane its own worktree.
- **The ground-rules file can get polluted with task-specific notes** if a driver isn't
  explicitly told to keep them out of it (see "Carrying project context to delegates") —
  caught once, fixed in the instructions, confirmed clean on the next real-repo run.
- **The Windows Codex sandbox bug (`CreateProcessAsUserW`) and its
  `--dangerously-bypass-approvals-and-sandbox` fix reconfirmed across four independent
  real invocations**, not just the first discovery — including one case where Codex's
  own self-verification hit the bug but its file edits had already landed successfully
  first, so the task still completed without needing the retry. The fix generalizes.

## My current setup (reference, not load-bearing for the skill logic)

- **Subscriptions, roughly $150/month total:** Claude Max 5x (~$100/mo) is the orchestrator
  plan this mode exists to protect; ChatGPT Plus (~$20/mo) backs Codex CLI; SuperGrok
  (~$30/mo, often a free first week on signup) backs Grok CLI. This is the actual
  cost basis the "lean on Grok/Codex" trade in this skill's framing assumes — if your own
  subscription mix is different, the "which resource is scarce for me" calculus changes
  and you should re-derive it rather than copy mine (see "Adapting the model routing"
  in the README).
- Delegates on hand: Grok CLI (`grok`, SuperGrok-backed, model Grok 4.5) and Codex CLI
  (`codex`, ChatGPT Plus-backed, models `gpt-5.6-luna` and `gpt-5.6-terra`).
- Orchestrator: this session (Claude, Max plan) — the resource this mode exists to
  conserve. Sonnet is spawned only as a low-effort driver, never as an implementer, at
  any tier including Tier 0.
- Codex CLI needs to be reasonably current (confirmed working on 0.144.0; a stale 0.133.0
  install didn't know the `gpt-5.6` family or its higher effort tiers at all) — the
  discover-first probe should catch a stale install via an unknown-model/unknown-effort
  error and report it as "needs an update," not silently misroute to a different model.
- Both delegate CLIs confirmed callable via plain Bash, with working `--reasoning-effort`
  (Grok) and `-m <model> -c model_reasoning_effort=` (Codex) flags as documented above.
  Reverify with `--help` / `codex --version` if a future run errors — CLI interfaces drift.
