# Toru tools by route

## Account
- `account_usage`: credits available/used/reserved and admission state.

## Local companion (only when the user asks to sign in on their own computer)
- `local_browser_pair {url}` → `pairingCode`, `setupUrl` (companion page), `companionCommand`, `downloadUrl`. Give these to the user; end the turn.
- `local_browser_status {connectionId}` → `waiting_for_companion` | `waiting_for_local_sign_in` | `ready` | `in_use` | `disconnected`.
- `session_open {url, localConnectionId, requestId}` → recording session id.

## Toru browser sign-in (step-by-step sessions)
- `session_connect {url, loginEmail?, useDemoAccount?}` → `setupUrl` for the secure sign-in page. `connection_status {id}` → `ready` + `importId` means signed in; `awaiting_code` means the user must enter the e-mailed code or link on `setupUrl`. `session_open {url, importId, requestId}`.
- `demo_accounts_list {}` → logins the user saved in the console: `origin`, `username`, `kind` (`password` or `email`). Never secrets. Call it first for any web demo.

## Whole demo in one call (both web routes)
- `recording_run {requestId, url, goal, guidance?, values?, maximumActions?, localConnectionId? | importId? | useDemoAccount? | loginEmail?, allowedOrigins?, render?}` → `runId`, `status: planning`.
- `recording_run_status {runId, waitMs?}` → `planning | connecting | awaiting_code | navigating | needs_input | recording | rendering | done | needs_review | failed | canceled | interrupted`, with `sessionId`, `actions`, `decisions`, `costUsd`, `stages`, `qualityReasons` and a `nextCall`.
- `recording_run_cancel {runId}`.

## Browser actions (both web routes)
- `browser_observe`, `browser_navigate`, `browser_click`, `browser_type`, `browser_key`, `browser_scroll`, `browser_move`, `browser_wait`, `browser_select`.
- Jev: `navigation_start {id, goal}`, `navigation_status`, `navigation_guide`, `navigation_stop`.
- `mark_demo_moment {id, kind: setup|action|result, note, observationId?}`.

## Recording (all routes)
- `recording_start {id}`, `recording_stop {id}`, `recording_status {id, waitMs?}`, `recording_cancel {id}`, `recording_render {id, requestId, background?, padding?, cursorScale?, compressPauses?, gif?, width?, acceptReview?}` (the brand saved in the console for the product domain applies automatically), `recording_links {id}` → `videoUrl`, optional `previewUrl`, `gifUrl`, `rawUrl`, `planUrl`, `eventsUrl`, `expiresAt`.

## Mac
- `desktop_devices` → paired Macs with `status` (`ready` | `in_use` | `awaiting_approval` | `offline` | `revoked`) and `connectionId`; empty list includes `downloadUrl` and `nextAction`.
- `desktop_recording_pair` → `setupUrl` (https page that opens the Toru app), `pairingCode` (5 minutes), `downloadUrl`.
- `desktop_sources {connectionId}` → windows and displays with ids; `desktop_select_source {connectionId, sourceId}`.
- `desktop_recording_open {connectionId, requestId}` → session id for the recording tools.
- `desktop_device_disconnect {connectionId}` revokes a Mac.

## Error codes worth knowing
- `local_browser_disconnected`: pair again; partial captures are kept.
- `desktop_upload_pending`: the user is recording manually on the Mac; wait for `captured`.
- `capture_interrupted`: the Mac stopped by itself; the user can Retry upload, then render.
- `artifacts_not_ready`: render not finished; call `recording_status` again.
- `quality_review` (`needsReview: true`): the rendered video failed the delivery check (`blank_frames`, `static_video`, `too_short`); not delivered, no credit used. Re-record, or `recording_render` with `acceptReview:true` on explicit user request.
- `run_in_progress`: one run at a time per account; wait for `recording_run_status` or cancel.
- `rerender_limit`, `daily_service_limit`, `beta_admission_required`: report plainly; do not retry blindly.
