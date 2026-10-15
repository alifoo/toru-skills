# Toru tools by route

Toru records in two places only: the user's own Chrome (Toru for Chrome) and their Mac (Toru for Mac).

## Account
- `account_usage`: credits available/used/reserved and admission state.

## Whole demo in one call
- `recording_run {requestId, url, goal, plan, localConnectionId, viewport?, guidance?, values?, maximumActions?, render?}` → `runId`, `status: planning`. `plan` is required (see below); `localConnectionId` is a ready Chrome from `extension_devices`. `viewport` is the page size the recording uses: `1280x720`, `1440x900` or `1920x1080` (default).
- `desktop_recording_run {connectionId, requestId, goal, plan, windowSize?, guidance?, values?, maximumActions?, render?}`: the same for the window a Mac shares; needs `control: true` in `desktop_devices`. `windowSize` (`1280x720`, `1440x900`, `1920x1080` default) resizes the shared window for the recording, fitted to the screen, and puts it back afterwards; a window that refuses to resize is recorded as is and reported.
- `recording_run_status {runId, waitMs?}` → `planning | navigating | needs_input | recording | rendering | done | needs_review | failed | canceled | interrupted`, with `sessionId`, `actions`, `stages`, `qualityReasons` and a `nextCall`.
- `recording_run_cancel {runId}`.

### plan
- `preconditions?: string[]`: data or state that must exist first.
- `steps: {screen, action, target, value?, expect}[]` (1–30): `screen` is where it happens (route, page title or dialog); `action` is `click | type | select | press | scroll | wait | navigate`; `target` is the exact visible label (`button "Create invoice"`, `field "Customer"`); `value` is the text, option, key or path; `expect` is what visibly changes.
- `finalState: string`: what must be on screen at the end.
- `avoid?: string[]`: controls or changes the demo must not touch.
- `sources?: string[]`: where the plan came from (`file:line`), for the record.

## Toru for Chrome (web apps, in the user's own Chrome)
- `extension_devices` → paired Chrome browsers with `status` (`ready` | `in_use` | `offline`) and `connectionId`, plus `installUrl`; empty list includes `nextAction: extension_pair`.
- `extension_pair` → `connectUrl` (one-time, 10 minutes) and `installUrl`. The user installs the extension, opens `connectUrl` in that Chrome, and says connected.
- `extension_disconnect {connectionId}` unpairs a Chrome.

## Toru for Mac (native apps)
- `desktop_devices` → paired Macs with `status` (`ready` | `in_use` | `awaiting_approval` | `offline` | `revoked`), `control` and `connectionId`; empty list includes `installCommand` (`curl -fsSL https://toru.tools/install.sh | bash`), `brewCommand`, `downloadUrl` and `nextAction`.
- `desktop_recording_pair` → `setupUrl` (https page that opens the Toru app), `pairingCode` (5 minutes), `installCommand`, `brewCommand`, `downloadUrl`.
- `desktop_sources {connectionId}` → windows and displays with ids; `desktop_select_source {connectionId, sourceId}`.
- `desktop_recording_open {connectionId, requestId, windowSize?}` → capture-only session id for the recording tools. `windowSize` only applies when control is on; otherwise the window is recorded as is.
- `desktop_device_disconnect {connectionId}` revokes a Mac.

## Step-by-step browser actions
- `session_open {url, localConnectionId, requestId, viewport?}` → recording session id in the user's Chrome, at the chosen `viewport` (default `1920x1080`).
- `browser_observe` returns the screenshot and element refs. When the page is wider than the screenshot it also returns `viewport` and `screenshotScale` (0.6667 at 1920): clicks take page pixels, so divide a position read off the screenshot by `screenshotScale`, or click by element ref.
- `browser_navigate`, `browser_click`, `browser_type`, `browser_key`, `browser_scroll`, `browser_move`, `browser_wait`, `browser_select`.
- Toru navigation: `navigation_start {id, goal}`, `navigation_status`, `navigation_guide`, `navigation_stop`.
- `mark_demo_moment {id, kind: setup|action|result, note, observationId?}`.

## Recording (all routes)
- `recording_start {id}`, `recording_stop {id}`, `recording_status {id, waitMs?}`, `recording_cancel {id}`, `recording_render {id, requestId, background?, padding?, cursorScale?, compressPauses?, gif?, width?, acceptReview?}` (the brand saved in the console for the product domain applies automatically; output is 1920 wide unless the brand or `width` says otherwise), `recording_links {id}` → `videoUrl`, optional `previewUrl`, `gifUrl`, `rawUrl`, `planUrl`, `eventsUrl`, `expiresAt`.

## Error codes worth knowing
- `plan_required` / `plan_invalid`: write or fix the plan and call again; the error includes a template. Nothing was recorded and no credit was used.
- `desktop_control_not_allowed`: the Mac is capture only; ask the user to turn on "Let your agent operate the shared window" in Toru for Mac Settings, or use `desktop_recording_open`.
- `extension_offline`: the paired Chrome is closed or the extension is off; ask the user to open Chrome.
- `desktop_upload_pending`: the user is recording manually on the Mac; wait for `captured`.
- `capture_interrupted`: the Mac stopped by itself; the user can Retry upload, then render.
- `artifacts_not_ready`: render not finished; call `recording_status` again.
- `quality_review` (`needsReview: true`): the rendered video failed the delivery check (`blank_frames`, `static_video`, `too_short`); not delivered, no credit used. Re-record, or `recording_render` with `acceptReview:true` on explicit user request.
- `run_in_progress`: one run at a time per account; wait for `recording_run_status` or cancel.
- `rerender_limit`, `daily_service_limit`, `beta_admission_required`: report plainly; do not retry blindly.
