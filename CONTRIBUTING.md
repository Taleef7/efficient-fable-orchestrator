# Contributing

This is a small, single-purpose skill, so contribution scope is intentionally narrow.

## The most useful kind of PR

Model names and CLI flags drift. `SKILL.md` hardcodes what worked as of mid-2026 for
Grok CLI and Codex CLI (model IDs, effort-level enums, sandbox flags). If you've verified
that any of these have changed — a renamed model, a new flag, a different default sandbox
behavior — a PR updating the "Known-good invocation shapes" section with what you actually
confirmed (not what you assume) is the single most valuable contribution this repo can get.

Please include *how* you verified it (the actual command + output), not just the new value.
The skill's own "discover before you invoke" step exists because these details go stale —
help keep the documented defaults honest rather than just current-as-of-when-you-read-this.

## Other contributions

- **New delegate CLIs** (e.g. Gemini CLI, another provider's headless mode): welcome, as a
  new subsection under "Known-good invocation shapes," following the same format — the
  actual invocation, the actual flags, and what broke before you found the right ones.
- **Bug reports**: if you hit a failure mode this skill doesn't document (a hang, a silent
  wrong answer, a sandbox error), open an issue with the exact command and output. That's
  exactly how the documented Windows Codex sandbox workaround was found.
- **Platform notes**: this was built and tested on Windows. If something behaves
  differently on macOS/Linux, a short note is genuinely useful — I can't test what I don't
  have.

## What's probably out of scope

- Expanding this into a general-purpose multi-agent framework. The whole point is that it
  stays a thin, legible protocol a human can read end-to-end in a few minutes. If your
  change needs its own README section to explain, it's probably a fork, not a PR.
- Adding delegates or tiers "just in case" without a real, tested use case behind them —
  see the README's "what's actually proven vs. what isn't" section for why that matters
  here specifically.

## Process

1. Fork, branch, make the change.
2. If you changed anything under "Known-good invocation shapes," show your verification
   (command + output) in the PR description.
3. Open a PR. No CI, no formal review checklist — just be specific about what you tested.
