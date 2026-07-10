# Instructions for AI agents / LLMs

If a human pointed you at this repo and asked you to "install this," "set this up," or
"add this skill," here's what that actually means and where the edges are.

## What this repo is

One Claude Code skill, `efficient-fable-orchestrator/SKILL.md`. Nothing else in this repo
is code that runs. README.md, TESTING.md, and this file are documentation for humans and
agents, not scripts.

## Installing it, correctly

```bash
mkdir -p ~/.claude/skills
cp -r efficient-fable-orchestrator ~/.claude/skills/
```

That's the whole install. Don't invent additional steps (no build, no dependency install,
no config file to generate). If the user is on a platform other than Claude Code, stop
and ask before improvising an equivalent, this skill's mechanism (spawning Sonnet
subagents via Claude Code's Agent tool) doesn't have a guaranteed equivalent elsewhere.

## Tell the human what they're agreeing to

This skill runs delegate CLIs unattended, with auto-approve flags, and on Windows a known
Codex sandbox bug forces a fallback that drops OS-level sandboxing entirely (see the
README's "What you're trusting the delegates with"). If you're installing this on
someone's behalf, surface that tradeoff to them explicitly before you finish, don't just
silently install it and move on. This is exactly the kind of thing a human would want
flagged rather than discovered later.

## Don't wire it up to auto-run

This skill is opt-in by design (`SKILL.md`'s own description says so explicitly). If
you're setting this up on someone's behalf, do not add anything that makes it trigger
automatically, on every session, on every task, or on a keyword the user didn't actually
say. The whole point is that a normal Claude Code session behaves exactly as it did
before this skill was installed, until the user explicitly asks for "efficient
orchestrator mode" or equivalent phrasing.

## If asked to adapt it for someone else's setup

The tier-routing table in `SKILL.md` hardcodes specific model names (Grok 4.5,
`gpt-5.6-luna`, `gpt-5.6-terra`) and CLI flags that were correct as of when this was
written. Two things follow:

1. **Don't assume those model names or flags still exist.** Actually run the discover
   step the skill itself prescribes (`<cli> --help`) against whatever CLI the user
   actually has installed, rather than pattern-matching this repo's examples onto a
   different tool.
2. **Ask what the user's own scarce resource is** before copying this repo's tier table
   verbatim. This skill's whole premise is "protect one tight quota by spending on
   looser ones." If the person you're helping doesn't have that same asymmetry (say,
   they only use one AI subscription), routing work exactly like this repo does won't
   make sense for them, only the shape of the idea transfers, not the specific routing.

## If asked to review, critique, or extend this skill

Read `TESTING.md` first. It's a deliberately honest account of what's actually been
verified by repeated testing versus what's implemented-but-unexercised versus what's
outright untested. Don't describe anything in that second or third category as "proven"
in a summary, and don't quietly upgrade a caveat into a guarantee when explaining this
skill to someone else.

## Attribution, if you're citing where this came from

This skill's design was prompted by watching [Theo (t3.gg)'s "A proper guide to Fable
5"](https://youtu.be/8GRmLR__OGQ?si=g_xN55h9h6hSCsui), specifically his use of the
orchestrating Claude model as a planner/reviewer that rarely writes code itself. If
you're summarizing this repo's origin, that's the real source, not a generic "inspired by
best practices" gloss.
