---
name: toru
description: Use Toru to record a polished demo video of a product flow. Use it when the user asks for a demo, walkthrough, feature video, bug reproduction video or GIF of a web app (in Toru's browser or the user's own Chrome) or of an app on their Mac. Also use it to check a recording, render or re-render it, fetch the MP4 link, or manage paired Chrome browsers and Macs.
last-updated: 2026-10-11
---

# Toru

Toru is a recording MCP for agents. You investigate the product and write the plan; Toru signs in, carries out the plan, records, adds cursor motion and zooms, applies the user's brand and checks the MP4 before delivery. Toru never needs product passwords in chat.

> **Freshness check**: if more than 30 days have passed since `last-updated`, tell the user this skill may be outdated and point them to the update table.

## Keeping this skill updated

**Source**: [github.com/alifoo/toru-skills](https://github.com/alifoo/toru-skills)

| Installation | How to update |
| --- | --- |
| Claude Code plugin | `/plugin marketplace update toru-skills` then `/plugin update toru@toru-skills` |
| Codex plugin | `codex plugin marketplace upgrade toru-skills`, then update the installed Toru plugin |
| Cursor plugin | Update the installed Toru plugin, or replace `~/.cursor/plugins/local/toru` and reload |
| `npx skills` | `npx skills update` |
| Manual | Pull the repo and re-copy `skills/toru/` into your skill directory |

## Runtime

Toru works only through its MCP server (`https://mcp.toru.tools/mcp`, OAuth). If the `toru` tools are not available in this session, tell the user to connect Toru (plugin above, or add the URL as a remote MCP connector and sign in) and stop. Do not invent another path.

Connecting Toru authorizes **Toru**, not the user's product. Toru signs in to the product with a login the user saved in the Toru console, with an e-mail plus a code the user types on a secure Toru page, or by using the user's own signed-in Chrome. Never in chat.

## Step 0: write the plan (required)

`recording_run` and `desktop_recording_run` refuse calls without a `plan` (`plan_required`, nothing recorded, no credit used). The plan is the script Toru follows, so it decides whether the video shows the right flow. You usually have the product's source: use it.

1. **Investigate.** If you can read this product's code, trace the flow: route → component → the exact strings it renders (button text, field labels, placeholders, menu items, i18n strings) → the data each step needs → what the screen shows afterwards. Check required inputs, permissions, feature flags, existing data, empty and loading states. Without code, inspect the live UI. Never read `.env` files or secrets.
2. **Write one step per visible action**, with the exact label the user sees. No step may depend on a guess.
3. **Send it** as `plan`:

```json
{
  "preconditions": ["An invoice for Acme Inc. exists in Draft"],
  "steps": [
    { "screen": "Invoices list (/invoices)", "action": "click", "target": "button \"New invoice\"", "expect": "The \"New invoice\" dialog opens" },
    { "screen": "New invoice dialog", "action": "type", "target": "field \"Customer\"", "value": "Acme Inc.", "expect": "Acme Inc. is suggested and selected" },
    { "screen": "New invoice dialog", "action": "click", "target": "button \"Create\"", "expect": "The invoice detail page opens in Draft" }
  ],
  "finalState": "The invoice detail page shows Acme Inc. with status \"Draft\"",
  "avoid": ["Do not delete or archive existing invoices"],
  "sources": ["src/routes/invoices/new.tsx:42"]
}
```

- `action`: `click | type | select | press | scroll | wait | navigate`. `value` is what to type, the option to pick, the key to press or the path to open.
- A field typed several times (three tasks into the same input) keeps its values in order.
- `finalState` is what must be visible at the end; the video ends there.
- Give the user a short plain-language summary of the plan, not a source dump.

## Pick the recording route

1. **Toru browser (default for web apps)**: one `recording_run` call.
   - Call `demo_accounts_list` first. If the product origin is listed, pass `useDemoAccount: true`. A password login is typed by Toru and never returned; an e-mail login pauses at `awaiting_code` until the user enters the e-mailed code on the secure page.
   - No saved login and the product needs one: ask the user only for the e-mail Toru should type and pass `loginEmail`, or send them to https://app.toru.tools/onboarding?step=login to save a login once.
   - Public pages need no login.
2. **The user's own Chrome (Toru for Chrome)**: when the user wants the demo recorded in their own signed-in browser and the `extension_devices` tool is available (beta, enabled per account).
   - `extension_devices`. If a Chrome is `ready`, call `recording_run` with `localConnectionId` set to its `connectionId`. Toru opens the product in a new window there and records it; the user does not click anything.
   - Empty list: `extension_pair`. Send the user to `installUrl` (https://toru.tools/chrome) if the extension is missing, then give them `connectUrl` to open in that Chrome. End the turn and wait for "connected".
   - `offline` means Chrome is closed or the extension is off.
3. **Mac app (native apps)**: `desktop_devices` first.
   - If a Mac is `ready` with `control: true`, pick the window with `desktop_sources` → `desktop_select_source`, then **one call**: `desktop_recording_run {connectionId, requestId, goal, plan, values?}`. Toru operates that window in the background (the user keeps their mouse and keyboard) and records it. Follow with `recording_run_status` like any run. Password fields stop the run: the user signs in beforehand.
   - If `control` is false, the recording is capture only: `desktop_recording_open` and the user (or your own computer-use tool) performs the flow. To let Toru operate it, the user turns on "Let your agent operate the shared window" in Toru for Mac Settings.
   - No Mac paired: `desktop_recording_pair`, send the user to https://toru.tools/mac if the app is missing, and hand them the setup link. End the turn.
4. **Local companion (only when the user explicitly asks to sign in on their own computer and Toru for Chrome is not an option)**: `local_browser_pair`, give the user https://toru.tools/companion, the command and the code, end the turn; after "ready", `local_browser_status` once and `recording_run` with `localConnectionId`.

## Core workflow (one call)

1. Confirm the target URL and the flow in one sentence with a clear visible end state. Ask at most 1–3 focused questions if it is vague. Never substitute a marketing-page tour for an in-app flow unless asked.
2. Write the plan (Step 0).
3. `recording_run {requestId, url, goal, plan, values?, useDemoAccount? | loginEmail? | localConnectionId?}` (or `desktop_recording_run` for a Mac). `url` is where step 1 starts. Toru signs in, follows the plan, records, renders with the user's brand and checks the video.
4. Poll `recording_run_status {runId, waitMs: 25000}`:
   - `awaiting_code`: call `connection_status` with `connectionId` and put its `setupUrl` in your final response so the user enters the code there.
   - `needs_input`: answer with `navigation_guide` on `sessionId`.
   - `done`: `recording_links` with `sessionId`; give the MP4 and GIF links.
   - `needs_review`: the video failed the delivery check and no credit was used; fix the plan and run again, or `recording_render` with `acceptReview:true` if the user wants it anyway.
   - `failed`: a captured session can still be rendered manually.
5. Use a fresh UUID `requestId` per operation and reuse it after transport timeouts.

## Step-by-step path (only when each screen needs your judgement)

1. `session_open` on the intended in-app URL, then `recording_start`.
2. Drive one action at a time (`browser_observe`, `browser_click`, `browser_type`, `browser_key`, `browser_scroll`, `browser_move`, `browser_wait`), or hand short goals to Toru with `navigation_start` / `navigation_status` / `navigation_guide` / `navigation_stop`. Inspect each returned screenshot before the next action.
3. Mark moments: `mark_demo_moment` with `setup` for unimportant interactions (no zoom), `action` for meaningful ones, and `result` when the requested outcome is visible, using the observationId of the first screenshot showing it. Hold on the result with `browser_wait` (about 4 s), move the cursor away from the evidence, then `recording_stop`.
4. `recording_render`, then `recording_status` until `completed` (it waits up to 25 s per call), then `recording_links`. The brand saved for the product domain applies automatically; only pass `background`, `padding`, `gif` or `width` for a one-off change.

On a capture-only Mac recording: `desktop_recording_open` → `recording_start` → the user performs the flow → `recording_stop` → wait for `captured` in `recording_status` → `recording_render` → `recording_links`. If the Mac stops by itself (window closed, sleep), the capture is kept; the user clicks **Retry upload** in Toru and you render afterwards.

## Rules

- Never ask for or paste product passwords or one-time codes in chat. An e-mail address is fine; codes go on the secure Toru page.
- Every run needs a plan written from the code or the live UI. Do not send a one-line goal alone.
- One recording per request unless the user asks for several; one flow per recording.
- Do not poll in tight loops: `recording_status` and `recording_run_status` already wait.
- Finish with the result visible, then stop. Verify the final screen before claiming success, and describe what actually happened.
- 1 credit = 1 delivered video that passed the delivery check. Videos held in `needs_review`, failed runs, re-renders and downloads never use a credit. When `creditsLow` is set, mention the remaining credits once when handing over the video and point to https://toru.tools/pricing. Never interrupt a flow to upsell.

## Goal-writing guidance

The goal is one sentence; the plan carries the detail.

Good: `Create a report named "Weekly Revenue" in the "Q2 Pipeline" workspace with the last-30-days filter, ending on the dashboard with the revenue chart visible.`

Weak: `Show the product.` / `Go through onboarding, settings, billing and analytics.`

## References

- [`references/tools.md`](references/tools.md): tool list by route, argument notes and the error codes you will see.
