# pingone-sample-apps — a Claude Code skill

A skill for building, running, and troubleshooting the PingOne devdocs sample apps (`ping-rocks/devdocs-sample-apps`) — 4 use cases × 5 stacks.

Install it and Claude Code will know how to pick the right sample, run the one-time PingOne bootstrap, wire the correct worker-app roles, and diagnose the errors these samples actually produce.

## Install

Copy the skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/curtismu7/pingone-sample-apps-skill.git
cp -r pingone-sample-apps-skill/pingone-sample-apps ~/.claude/skills/
```

Or, if you already have this repo checked out:

```bash
cp -r pingone-sample-apps ~/.claude/skills/
```

That's it — no restart needed. Claude picks it up on the next session. To confirm, start Claude Code and ask something like *"how do I set up the m2m-credentials sample?"*

**Project-scoped instead of global:** put it in `<your-project>/.claude/skills/` to make it available only inside that project.

## What it covers

- **Which sample to pick**, including which two cannot be completed without a real email inbox
- **The one-time PingOne bootstrap** — admin environment vs sandbox environment, worker app creation, role assignment, the root `.env`
- **The correct worker-app roles** — the repo's `AGENTS.md` gets two of these wrong, one of them in a way that over-privileges your tenant
- **Run commands per stack**, and the inconsistencies between them
- **The `response_mode=pi.flow` architecture**, and why the security parameters that are safely omitted there must *not* be omitted in a normal redirect flow
- **A failure-mode table** — 401 / 403 / 406 / 415, expired bootstrap tokens, missing OTPs, and the rest

## Why it disagrees with AGENTS.md

The skill was written from a run of all 20 sample apps, and two role entries in the repo's own `AGENTS.md` are wrong:

| Use case | Correct | `AGENTS.md` |
|---|---|---|
| `custom-admin-role` | `Environment Admin` | `Organization Admin` |
| `m2m-credentials` | `Identity Data Read Only` + `Environment Admin` | `Identity Data Read Only` + `PingOne Protect` |

The first matters most: `Organization Admin` is a real role, so granting it **succeeds** — your tenant is just over-privileged org-wide, with nothing to signal it. The sample itself was deliberately written to avoid needing it, hardcoding the Environment Admin role ID rather than doing a lookup that only an Organization Admin could perform.

The skill carries the corrected values. If `AGENTS.md` is fixed upstream, this note can go.

## Requirements

- Claude Code
- Access to `ping-rocks/devdocs-sample-apps` — it is Ping Identity EMU-gated, so a personal GitHub account cannot see it
- A PingOne tenant with an administrator environment and a sandbox environment
- `jq` and `curl`, plus the toolchain for whichever stack you run

## License

Provided as-is, with no warranty or support, matching the disclaimer on the sample apps themselves.
