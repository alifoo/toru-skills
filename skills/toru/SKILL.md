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

Connecting Toru authorizes **Toru**, not the user's product. Product sign-in happens on the user's computer.

## Pick the recording route

1. **Local browser (default for web apps)**: `local_browser_pair` with the product URL. Give the user the companion link (`https://toru.tools/companion`), the companion command and the one-time code, then END THE TURN. The user signs in inside the companion's Chrome window and replies "ready". Then `local_browser_status` once, and `session_open` with `localConnectionId`.
2. **Mac app (native apps, or the user's real browser session)**: `desktop_devices` first. If a Mac is paired and ready, `desktop_sources` → `desktop_select_source` to choose the window, then `desktop_recording_open`. If no Mac is paired, `desktop_recording_pair` and hand the user the setup link; opening it pairs and connects the Mac with no further clicks. Browser tools and Jev are not available on the Mac route: the user (or your own computer-use tool) performs the actions while Toru records.
3. **Hosted browser (fallback only when the user asks)**: `session_connect` with the sign-in URL, give the user the `setupUrl`, wait for "ready", then `connection_status` and `session_open` with the `importId`. Two ways to keep passwords out of the chat: pass `loginEmail` (Toru submits only the e-mail; when the product asks for a code or magic link the connection pauses at `awaiting_code` and the user enters it on the same `setupUrl`, never in chat), or `useDemoAccount: true` when `demo_accounts_list` shows the product origin (the user saved a demo account in the console; the password is typed by Toru and never returned). Both also work inside `recording_run`.

## Core workflow (web routes)

1. Confirm the target URL and the flow in one sentence with a clear visible end state. Ask at most 1–3 focused questions if it is vague. Never substitute a marketing-page tour for an in-app flow unless asked.
2. Pair (route 1 or 3). Then prefer **one call for the whole demo**: `recording_run {requestId, url, goal, guidance?, values?, localConnectionId? | importId?}`. Toru opens the session, starts recording, lets Jev drive the flow, stops, renders and checks the video. Poll `recording_run_status {runId, waitMs: 25000}` until `done` (then `recording_links` with `sessionId`), `needs_input` (answer with `navigation_guide` on `sessionId`), `needs_review` (the video failed the check: no credit was used; fix the flow and run again, or `recording_render` with `acceptReview:true` if the user wants it anyway) or `failed` (a captured session can still be rendered manually). Use the manual steps below only when the flow needs your own judgement at each screen.
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

- Never ask for or paste product credentials or one-time codes in chat. Sign-in is the user's, on their computer or on the secure Toru page.
- One recording per request unless the user asks for several; one flow per recording.
- Do not poll in tight loops: `recording_status` already waits; `local_browser_status` once after the user replies.
- Finish with the result visible, then stop. Verify the final screen before claiming success.
- 1 credit = 1 delivered video that passed the delivery check. Videos held in `needs_review`, failed runs, re-renders and downloads never use a credit. `recording_status` returns `credits` with the final video; when `creditsLow` is set, mention the remaining credits once when handing over the video and point to https://toru.tools/pricing (plans or credit packs). Never interrupt a flow to upsell.

## Goal-writing guidance

Good: `Sign in to https://app.example.com, open the "Q2 Pipeline" workspace, create a report named "Weekly Revenue", apply the last-30-days filter, and end on the dashboard with the revenue chart visible.`

Weak: `Show the product.` / `Go through onboarding, settings, billing and analytics.`

## References

- [`references/tools.md`](references/tools.md): tool list by route, argument notes and the error codes you will see.
