# Toru Agent Skills

Agent skill repository for [Toru](https://toru.tools): your agent records a polished product demo video, in a browser you sign in to yourself or in a window on your Mac, and Toru renders the MP4 with cursor motion and zooms. Toru never stores your product passwords.

## Install

Claude Code:

```
/plugin marketplace add alifoo/toru-skills
/plugin install toru@toru-skills
/reload-plugins
```

Then authorize the Toru MCP server when prompted (`/plugin` → Installed → Toru).

Cursor, Codex CLI and others: `npx skills add alifoo/toru-skills`, or add `https://mcp.toru.tools/mcp` as a remote MCP connector and copy `skills/toru/` into your skills directory.

## First recording

Paste into your agent:

> Record a demo video of https://app.example.com with Toru: sign in, create a new project, and end on the project dashboard. Use the local companion so I sign in myself, and give me the MP4.

Your video appears in your library at https://toru.tools.

## What's in this repo

- `skills/toru/SKILL.md`: the workflow, routes (Toru browser with a saved login, Mac app, local companion on request), rules and goal-writing guidance.
- `skills/toru/references/tools.md`: the tool list by route and error codes.
- `skills/toru/.mcp.json`: the hosted MCP server.
- Plugin manifests for Claude Code (`.claude-plugin`), Cursor (`.cursor-plugin`) and Codex (`plugins/toru`, `.agents`).

MIT license.
