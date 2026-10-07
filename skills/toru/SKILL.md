---
name: toru
description: Use Toru to record a polished demo video of a product flow. Use it when the user asks for a demo, walkthrough, feature video, bug reproduction video or GIF of a web app (recorded in a browser they sign in to themselves) or of a window on their Mac. Also use it to check a recording, render or re-render it, fetch the MP4 link, or manage paired Macs.
last-updated: 2026-10-07
---

# Toru

Toru is a recording MCP for agents. You direct the demo; Toru provides the browser (or the Mac capture), the cursor motion capture and the rendered MP4. The user signs in to their product themselves: Toru never stores product passwords.

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

Connecting Toru authorizes **Toru**, not the user's product. Toru signs in to the product with a login the user saved in the Toru console, or with an e-mail plus a code the user types on a secure Toru page. Never in chat.

## Pick the recording route

1. **Toru browser (default for web apps)**: one `recording_run` call.
   - Call `demo_accounts_list` first. If the product origin is listed, pass `useDemoAccount: true`. A password login is typed by Toru and never returned; an e-mail login pauses at `awaiting_code` until the user enters the e-mailed code on the secure page.
   - No saved login and the product needs one: ask the user only for the e-mail Toru should type and pass `loginEmail`, or send them to https://app.toru.tools/onboarding?step=login to save a login once.
   - Public pages need no login.
2. **Mac app (native apps)**: `desktop_devices` first. If a Mac is paired and ready, `desktop_sources` → `desktop_select_source` to choose the window, then `desktop_recording_open`. If no Mac is paired, `desktop_recording_pair` and hand the user the setup link; opening it pairs and connects the Mac. Browser tools and Jev are not available on the Mac route: the user (or your own computer-use tool) performs the actions while Toru records.
3. **Local companion (only when the user explicitly asks to sign in on their own computer)**: `local_browser_pair`, give the user https://toru.tools/companion, the command and the code, end the turn; after "ready", `local_browser_status` once and `recording_run` with `localConnectionId`.

## Core workflow (web routes)

1. Confirm the target URL and the flow in one sentence with a clear visible end state. Ask at most 1–3 focused questions if it is vague. Never substitute a marketing-page tour for an in-app flow unless asked.
2. **One call for the whole demo**: `recording_run {requestId, url, goal, guidance?, values?, useDemoAccount? | loginEmail? | localConnectionId?}`. Toru opens the session, signs in, records, lets Jev drive the flow, renders with the user's brand and checks the video. Poll `recording_run_status {runId, waitMs: 25000}`:
   - `awaiting_code`: call `connection_status` with `connectionId` and put its `setupUrl` in your final response so the user enters the code there.
   - `needs_input`: answer with `navigation_guide` on `sessionId`.
   - `done`: `recording_links` with `sessionId`; give the MP4 and GIF links.
   - `needs_review`: the video failed the check and no credit was used; fix the flow and run again, or `recording_render` with `acceptReview:true` if the user wants it anyway.
   - `failed`: a captured session can still be rendered manually.
   Use the manual steps below only when the flow needs your own judgement at each screen.
2b. Manual path: open the session on the intended in-app URL with `session_open`.
3. `recording_start`. Drive the flow with the browser tools one action at a time (`browser_observe`, `browser_click`, `browser_type`, `browser_key`, `browser_scroll`, `browser_move`, `browser_wait`), or hand short goals to Jev with `navigation_start` / `navigation_status` / `navigation_guide` / `navigation_stop`. Inspect each returned screenshot before the next action.
4. Mark moments: `mark_demo_moment` with `setup` for unimportant interactions (suppresses zoom), `action` for meaningful ones, and `result` when the requested outcome is visible, using the observationId of the first screenshot showing it. Hold on the result with `browser_wait` (about 4 s), move the cursor away from the evidence, then `recording_stop`.
5. `recording_render`, then `recording_status` until `completed` (it waits up to 25 s per call), then `recording_links` for the MP4 (and preview, GIF, raw capture, plan and event log when available). The user's brand for the product domain (colors, logo, intro/outro, GIF, size) is applied automatically; do not ask for brand details unless the user wants a one-off change (`background`, `padding`, `gif`, `width` on `recording_render`). If `recording_status` returns `needsReview`, the video failed the delivery check (blank, static or too short): it was not delivered and no credit was used; fix the flow and re-record, or call `recording_render` with `acceptReview:true` only if the user explicitly wants it. Export useful partial attempts too, and describe what actually happened; a success score is not required to export.
6. Use a fresh UUID `requestId` per operation and reuse it after transport timeouts.

## Core workflow (Mac route)

1. `desktop_devices`; if empty, `desktop_recording_pair` and give the user the setup link (and the install link if Toru for Mac is missing: https://toru.tools/mac). End the turn.
2. `desktop_sources` and `desktop_select_source` for the window the user named (default is the main display).
3. `desktop_recording_open` → `recording_start`. Tell the user (or your computer-use tool) to perform the flow. `recording_stop` → wait for `captured` in `recording_status` → `recording_render` → `recording_links`.
4. If the Mac stops by itself (window closed, sleep), the capture is kept locally; the user clicks **Retry upload** in Toru and you can render afterwards.

## Rules

- Never ask for or paste product passwords or one-time codes in chat. An e-mail address is fine; codes go on the secure Toru page.
- One recording per request unless the user asks for several; one flow per recording.
- Do not poll in tight loops: `recording_status` and `recording_run_status` already wait.
- Finish with the result visible, then stop. Verify the final screen before claiming success.
- 1 credit = 1 delivered video that passed the delivery check. Videos held in `needs_review`, failed runs, re-renders and downloads never use a credit. `recording_status` returns `credits` with the final video; when `creditsLow` is set, mention the remaining credits once when handing over the video and point to https://toru.tools/pricing (plans or credit packs). Never interrupt a flow to upsell.

## Goal-writing guidance

Good: `Sign in to https://app.example.com, open the "Q2 Pipeline" workspace, create a report named "Weekly Revenue", apply the last-30-days filter, and end on the dashboard with the revenue chart visible.`

Weak: `Show the product.` / `Go through onboarding, settings, billing and analytics.`

## References

- [`references/tools.md`](references/tools.md): tool list by route, argument notes and the error codes you will see.
