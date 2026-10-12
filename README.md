# Toru Agent Skills

Agent skill repository for [Toru](https://toru.tools): your agent investigates your product, writes a step-by-step plan and Toru records a polished demo video of it, in Toru's browser, in your own Chrome ([Toru for Chrome](https://toru.tools/chrome)) or in an app on your Mac ([Toru for Mac](https://toru.tools/mac)). Toru renders the MP4 with cursor motion, zooms and your brand, and checks it before delivery. Your passwords never reach the chat.

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

> Record a demo video of https://app.example.com with Toru: create a new project and end on the project dashboard. Give me the MP4.

Your agent reads the code (or the live UI) to write the plan Toru follows: the screens, the exact button and field labels, the values and the final screen. To record in your own signed-in Chrome, add "in my Chrome"; for a Mac app, name the app.

Your video appears in your library at https://toru.tools.

## Toru for Mac

To record native apps, install the Mac app with one line in Terminal (macOS 15+, Apple silicon):

```
curl -fsSL https://toru.tools/install.sh | bash
```

Or `brew install --cask alifoo/toru/toru`. Your agent pairs it with a link when it first records your Mac.

## What's in this repo

- `skills/toru/SKILL.md`: the plan Toru requires, the routes (Toru browser with a saved login, your own Chrome, your Mac, local companion on request), the workflow and rules.
- `skills/toru/references/tools.md`: the tool list by route and error codes.
- `skills/toru/.mcp.json`: the hosted MCP server.
- Plugin manifests for Claude Code (`.claude-plugin`), Cursor (`.cursor-plugin`) and Codex (`plugins/toru`, `.agents`).

MIT license.
