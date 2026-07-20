# agentforce-feedback (Claude Code skill)

Turns detailed FDE/SE product feedback into a triage-ready **Linear in-app Document + summary comment** on a target issue (the Agentforce Feedback intake, `AFP-*`). Feature-agnostic — works for Testing Center, Agent Script, Voice, Builder, etc. No downloadable attachments: reviewers stay in Linear.

## What an SE gets

For any issue they point it at, the skill:
1. verifies the Linear MCP is on the right workspace (`eventsmobileapp` / "Agentforce Feedback");
2. reads the issue's intake fields so the write-up answers the customer's stated need;
3. drafts **one self-contained evidence document** (verdict → evidence → expected-vs-actual → root cause → why it matters → ask → appendix, with raw data tucked into collapsible blocks);
4. links it under the issue and posts a **concise summary comment** with the link.

The SE supplies the evidence (transcripts, data, screenshots, root cause, the ask); the skill structures and publishes it.

## Install

**Team-shared (recommended).** Commit the `agentforce-feedback/` folder into your team's skills location — either:
- a repo folder that syncs to `~/.claude/skills/` (or `.claude/skills/` in a shared project), or
- your team's Claude Code **plugin** skills directory.

**Personal / try it out.** Copy `agentforce-feedback/` into `~/.claude/skills/`.

Each user needs the **Linear MCP** connected. Feedback issues live in the `eventsmobileapp` workspace — if `get_issue AFP-###` fails, reconnect via `/mcp` → linear → pick `eventsmobileapp`.

## Use

In Claude Code:

```
/agentforce-feedback
```

or just ask: *"attach this feedback to AFP-123 as an in-app doc + summary comment."* Provide the target issue, the feature area, and your evidence. Claude will ask for anything missing, then draft, post, and return the links.

## Requirements

- Claude Code with skills enabled.
- Linear MCP connected (with access to the target workspace).
- `curl` available (only needed if attaching binary evidence like screenshots).

## Notes / known constraints

- Linear renders markdown (tables, code, `<details>`) but **not** HTML attachments inline — the deliverable is always a markdown doc, never an HTML file.
- The Linear MCP has **no delete/archive-document** capability; to retire a doc, the skill renames it to a redirect stub and asks you to archive it in-app.
- Always sanitize org IDs / customer PII / tokens before posting.
