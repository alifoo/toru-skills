# Toru tools by route

## Account
- `account_usage`: credits available/used/reserved and admission state.

## Whole demo in one call
- `recording_run {requestId, url, goal, plan, guidance?, values?, maximumActions?, useDemoAccount? | loginEmail? | importId? | localConnectionId?, allowedOrigins?, render?}` → `runId`, `status: planning`. `plan` is required (see below); `localConnectionId` is a paired Chrome from `extension_devices` or a local companion.
- `desktop_recording_run {connectionId, requestId, goal, plan, guidance?, values?, maximumActions?, render?}`: the same for the window a Mac shares; needs `control: true` in `desktop_devices`.
- `recording_run_status {runId, waitMs?}` → `planning | connecting | awaiting_code | navigating | needs_input | recording | rendering | done | needs_review | failed | canceled | interrupted`, with `sessionId`, `actions`, `stages`, `qualityReasons` and a `nextCall`.
- `recording_run_cancel {runId}`.

### plan
- `preconditions?: string[]`: data or state that must exist first.
- `steps: {screen, action, target, value?, expect}[]` (1–30): `screen` is where it happens (route, page title or dialog); `action` is `click | type | select | press | scroll | wait | navigate`; `target` is the exact visible label (`button "Create invoice"`, `field "Customer"`); `value` is the text, option, key or path; `expect` is what visibly changes.
- `finalState: string`: what must be on screen at the end.
- `avoid?: string[]`: controls or changes the demo must not touch.
- `sources?: string[]`: where the plan came from (`file:line`), for the record.

## Toru browser sign-in
- `demo_accounts_list {}` → logins the user saved in the console: `origin`, `username`, `kind` (`password` or `email`). Never secrets. Call it first for any web demo.
- `session_connect {url, loginEmail?, useDemoAccount?}` → `setupUrl` for the secure sign-in page. `connection_status {id}` → `ready` + `importId` means signed in; `awaiting_code` means the user must enter the e-mailed code or link on `setupUrl`. `session_open {url, importId, requestId}`.

## Toru for Chrome (the user's own Chrome; beta, enabled per account)
- `extension_devices` → paired Chrome browsers with `status` (`ready` | `in_use` | `offline`) and `connectionId`, plus `installUrl`; empty list includes `nextAction: extension_pair`.
- `extension_pair` → `connectUrl` (one-time, 10 minutes) and `installUrl`. The user installs the extension, opens `connectUrl` in that Chrome, and says connected.
- `extension_disconnect {connectionId}` unpairs a Chrome.

## Mac
- `desktop_devices` → paired Macs with `status` (`ready` | `in_use` | `awaiting_approval` | `offline` | `revoked`), `control` and `connectionId`; empty list includes `installCommand` (`curl -fsSL https://toru.tools/install.sh | bash`), `brewCommand`, `downloadUrl` and `nextAction`.
- `desktop_recording_pair` → `setupUrl` (https page that opens the Toru app), `pairingCode` (5 minutes), `installCommand`, `brewCommand`, `downloadUrl`.
- `desktop_sources {connectionId}` → windows and displays with ids; `desktop_select_source {connectionId, sourceId}`.
- `desktop_recording_open {connectionId, requestId}` → capture-only session id for the recording tools.
- `desktop_device_disconnect {connectionId}` revokes a Mac.

## Local companion (only when the user asks to sign in on their own computer)
- `local_browser_pair {url}` → `pairingCode`, `setupUrl` (companion page), `companionCommand`, `downloadUrl`. Give these to the user; end the turn.
- `local_browser_status {connectionId}` → `waiting_for_companion` | `waiting_for_local_sign_in` | `ready` | `in_use` | `disconnected`.

## Step-by-step browser actions
- `session_open {url, importId? | localConnectionId?, requestId}` → recording session id.
- `browser_observe`, `browser_navigate`, `browser_click`, `browser_type`, `browser_key`, `browser_scroll`, `browser_move`, `browser_wait`, `browser_select`.
- Toru navigation: `navigation_start {id, goal}`, `navigation_status`, `navigation_guide`, `navigation_stop`.
- `mark_demo_moment {id, kind: setup|action|result, note, observationId?}`.

## Recording (all routes)
- `recording_start {id}`, `recording_stop {id}`, `recording_status {id, waitMs?}`, `recording_cancel {id}`, `recording_render {id, requestId, background?, padding?, cursorScale?, compressPauses?, gif?, width?, acceptReview?}` (the brand saved in the console for the product domain applies automatically), `recording_links {id}` → `videoUrl`, optional `previewUrl`, `gifUrl`, `rawUrl`, `planUrl`, `eventsUrl`, `expiresAt`.

## Error codes worth knowing
- `plan_required` / `plan_invalid`: write or fix the plan and call again; the error includes a template. Nothing was recorded and no credit was used.
- `desktop_control_not_allowed`: the Mac is capture only; ask the user to turn on "Let your agent operate the shared window" in Toru for Mac Settings, or use `desktop_recording_open`.
- `extension_offline`: the paired Chrome is closed or the extension is off; ask the user to open Chrome.
- `local_browser_disconnected`: pair again; partial captures are kept.
- `desktop_upload_pending`: the user is recording manually on the Mac; wait for `captured`.
- `capture_interrupted`: the Mac stopped by itself; the user can Retry upload, then render.
- `artifacts_not_ready`: render not finished; call `recording_status` again.
- `quality_review` (`needsReview: true`): the rendered video failed the delivery check (`blank_frames`, `static_video`, `too_short`); not delivered, no credit used. Re-record, or `recording_render` with `acceptReview:true` on explicit user request.
- `run_in_progress`: one run at a time per account; wait for `recording_run_status` or cancel.
- `invalid_credentials`: the saved login failed; ask the user to fix it at https://app.toru.tools/#accounts. Do not retry.
- `rerender_limit`, `daily_service_limit`, `beta_admission_required`: report plainly; do not retry blindly.
