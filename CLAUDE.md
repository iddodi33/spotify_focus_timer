# Your Music Timer — Project Context for AI Assistants

## What This Project Is
A browser-based focus/break productivity timer that controls music playback 
across streaming services. Built as a PWA (Progressive Web App), hosted on 
GitHub Pages. No backend, no build step — single HTML file with vanilla JS.

## Live URL
https://iddodi33.github.io/spotify_focus_timer/spotify-focus-timer-pkce.html

## Repo
https://github.com/iddodi33/spotify_focus_timer

## Active File
spotify-focus-timer-pkce.html — this is the ONLY file to edit.
spotify-focus-timer.html — OLD VERSION, ignore completely.

## Tech Stack
- Vanilla HTML/CSS/JS, single file
- Spotify Web API (PKCE OAuth, no backend needed)
- YouTube IFrame Player API
- Web Speech API (voice announcements)
- Web Audio API (notification sounds)
- localStorage for persistence
- GitHub Pages for hosting
- PWA with manifest.json and sw.js

## Spotify Configuration
- Client ID: 8e34dc8a40374855a3c8e3064e271405
- Redirect URI: https://iddodi33.github.io/spotify_focus_timer/spotify-focus-timer-pkce.html
- Scopes: user-read-playback-state user-modify-playback-state 
  playlist-read-private playlist-read-collaborative 
  user-library-read playlist-modify-private

## Architecture

### Section Visibility — CRITICAL
The app has three main sections controlled by CSS classes:
- #service-selector — initial screen, choose Spotify/YouTube/Apple Music
- #login-section — Spotify connect screen (shown after choosing Spotify)
- #app-section — main timer UI

ALWAYS use !important on ALL show/hide rules — this has broken multiple 
times and is the most common bug in this project:
#service-selector { display: none !important; }
#service-selector.show { display: block !important; }
#login-section { display: none !important; }
#login-section.show { display: block !important; }
#app-section { display: none !important; }
#app-section.show { display: block !important; }

### Music Service Flow
1. User lands on #service-selector
2. Chooses Spotify → PKCE OAuth → #login-section → auth → checkPremium()
   - Premium → normal Spotify flow → #app-section
   - Free → show modal with "Back to Service Selector" and "Continue with Free YouTube"
3. Chooses YouTube → activateYouTubeMode() directly → #app-section
4. Chooses Apple Music → disabled, coming soon

### Return Visit Behavior
- spotify_token in localStorage → skip selector, go straight to Spotify app
- yt_free_accepted = "true" in localStorage → skip selector, go straight to YouTube mode
- Neither → show service selector

### Change Music Source Button
- Appears in top-right of #app-section
- Clears all localStorage keys and resets all state
- Returns to #service-selector cleanly

### Premium Detection — checkPremium()
- Always calls ensureValidToken() first
- Calls GET /v1/me and reads data.product
- "premium" → isPremium = true, proceed normally
- "free" or "open" → isPremium = false, show free tier modal
- undefined (API error, bad token) → default to isPremium = true
  NEVER show free modal on undefined — always default to Premium

### Device Resolution — resolveActiveDevice()
- Spotify PUT /v1/me/player/play returns 404 NO_ACTIVE_DEVICE if no device
  is currently "active", even when the desktop app is open
- resolveActiveDevice(showModal = true) is called at the top of
  startPlayback() before every play attempt
- Flow: ensureValidToken() → GET /v1/me/player/devices →
  if empty, show #no-device-modal and return null →
  if active device exists, return its id →
  otherwise pick Computer > Smartphone > first available,
  transfer via PUT /v1/me/player with play: false, wait 500ms, return id
- Always append ?device_id=X to the play URL — never call play without it
- Break transitions in endFocusSession pass silentOnNoDevice = true
  so a missing device mid-session does not pop the modal or block the timer
- noDeviceRetryFn is set to startFocusSession() or startCommSlot(slot)
  before each play attempt so the "Try Again" button re-enters the
  full flow cleanly

### localStorage Keys
- spotify_token — Spotify access token
- refresh_token — Spotify refresh token
- token_expiry — timestamp for token expiry check
- code_verifier — PKCE code verifier (cleared after auth)
- yt_free_accepted — "true" if user chose free YouTube mode
- comm_slots — JSON array of communication slot times and toggle state
- session_count_[date] — daily session counter
- focus_session_start — absolute timestamp of current focus session start
- break_session_start — absolute timestamp of current break session start
- plan_payload — the validated plan pushed in from Cockpit (see Plan Mode)
- plan_comm_slots — the plan's own comms windows, as
  [{hour, minute, duration}]. SEPARATE from comm_slots on purpose — see below

## Plan Mode (payload from Cockpit)

Cockpit (the task/day-planner app) opens this page with a day's blocks in the
URL hash. One way only: Cockpit → timer. This page has no backend and
Cockpit's database is locked to two owner emails, so nothing can be sent back.
Do not add a return path, a callback or a shared token.

### The hash-before-OAuth rule — READ THIS FIRST
`capturePlanFromHash()` is called at SCRIPT PARSE TIME, above `init()` and
before any OAuth handling. This ordering is load-bearing and is the thing a
future change is most likely to break:

- The Spotify PKCE redirect leaves this page and returns to REDIRECT_URI, which
  is `origin + pathname` — **no fragment**. A payload still sitting in
  `window.location.hash` when the user connects Spotify is gone for good.
- So the hash is decoded and banked into `plan_payload` immediately, then
  stripped from the URL with `history.replaceState`.
- **Everything after that reads localStorage. Nothing else may read
  `window.location.hash`.** If plans start vanishing when the user reconnects
  Spotify, check this ordering first.

### Payload
`#plan=<base64url of UTF-8 JSON>`:
```
{ v: 1, date: "YYYY-MM-DD", tz: "Europe/Dublin",
  blocks: [{ title, kind, start, end, cycles: [[work, break], ...] }],
  comms:  [{ start, end }, ...] }
```
`start`/`end` are absolute ISO instants; cycles are whole minutes.

Decoding is **not** plain `atob()`: swap `-`→`+` and `_`→`/`, re-pad to a
multiple of 4 with `=`, `atob()` to bytes, then `new TextDecoder().decode()`.
Block titles are real sentences containing em dashes and fadas — skipping the
TextDecoder step corrupts them, and skipping the character swap makes `atob`
throw.

### Rules
- **Never recompute the cycles.** The 85/15 split is Cockpit's job and is unit
  tested there. This page runs what arrives.
- **A block with an empty `cycles` array is an idle span, not an error.**
  Domestic blocks arrive that way deliberately: show the title and the time
  range, run no cycles, play no music, move on when it ends.
- **Fail visibly, never half-apply.** A payload that does not parse, is the
  wrong `v`, or is not today's local date is refused whole, with a one-line
  reason in `#plan-note`, and the timer runs normally. A stored plan is
  re-validated on every load, so yesterday's plan can never run today.
- **Never write `comm_slots`.** The plan's windows go to `plan_comm_slots`;
  `getEffectiveCommSlots()` prefers them while plan mode is on. The user's own
  slots must come back untouched on leaving plan mode. Note this is why plan
  slots are NOT routed through `getCommSlots()` — `saveCommSlotsConfig()`
  writes back from that function and would overwrite the user's list.
- **No meeting handling.** Meetings never stop the music; the manual In Call
  Mode button is the only interruption.

### How it runs
- `planStateAt(ts, plan, slots)` answers "what should be happening at this
  instant" from ABSOLUTE timestamps: focus / break / quiet / comm / idle / done.
  Nothing accumulates elapsed time, which is what makes screen-off recovery
  correct.
- `planSync()` runs that answer: sets the music, the progress bar, the status,
  one notification and ONE timer (`planTimeout`) for the next boundary. It is
  safe to call at any moment — start, boundary, visibilitychange.
- A **segment key** (`kind:block:cycle:start`) guards the transitions. A
  re-sync of the same segment must not restart the playlist or re-announce the
  block; only a changed key triggers playback and speech.
- `planTimeout` MUST be cleared in `stopSession()` like every other timer, and
  the notification is rescheduled per cycle — otherwise two fire across one
  cycle boundary.
- Stop ≠ Leave. "Stop Session" ends the run and keeps the plan loaded; "Leave
  plan mode" clears both plan keys and restores the normal timer in place, with
  no reload.
- Plan mode skips `saveSessionState()`/session recovery entirely: the plan plus
  absolute timestamps already restore the day exactly, so a stale session
  snapshot is cleared rather than left to resurface.

### Testing plan mode
The pure logic sits between `// === PLAN-PURE-START` and `// === PLAN-PURE-END`
markers so it can be extracted and run under Node without a browser (decode,
validation, segment splitting, and the state machine across a whole day). Keep
that region free of DOM, network and `Date.now()` — every function takes the
instant it needs. Build a payload with the same encoder Cockpit uses, append
`#plan=…` to the file URL, and load it.

### Timer Logic
- All session times stored as absolute timestamps (Date.now()) not countdowns
- startFocusSession() → stores focus_session_start, calls startPlayback('focus')
- endFocusSession() → increments session count, calls startBreakSession()
- startBreakSession() → stores break_session_start, calls startPlayback('break')
- stopSession() — MUST clear ALL of these or they leak:
  sessionTimer, progressInterval, commCheckInterval,
  transitionPlaybackTimeout, transitionNextSessionTimeout, planTimeout
  (and cancelNotification(), or notifications double-fire)
- startPlayback(type) returns true/false
  NEVER start timer if startPlayback() returns false

### Mobile Screen-Off Handling
- Page Visibility API used to detect screen on/off
- On visibilitychange → visible: calculate real elapsed time using timestamps
- If session should have ended while screen was off → trigger end immediately
- If session still ongoing → snap timer display to correct remaining time
- Push notifications scheduled at session start for when session ends
- Notifications cancelled in stopSession() to prevent double-firing
- Permission requested on app load with clear explanation to user

### Communication Slots
- Configurable time slots where focus pauses for emails/calls/meetings
- Stored in localStorage as JSON array
- Editable and toggleable in UI
- Default slots: 9:00, 12:30, 14:00, 15:30 (disabled by default)
- Recalculated on Page Visibility change same as session timer

### YouTube Mode
- activateYouTubeMode() sets isPremium = false, shows YouTube UI
- YT.Player instance controls playback
- Visible iframe in UI: 100% width, 200px tall, border-radius 12px, controls: 1
- rel: 0 to prevent related videos
- Default focus playlist: https://www.youtube.com/watch?v=u2ah9tWTkmk&list=PL4QNnZJr8sRPmuz_d87ygGR6YAYEF-fmw
- Default break playlist: https://www.youtube.com/watch?v=r0c9Q21AfLA&list=PLvZ8_HnTraXnQZ70T2ZEet6i-bSo381zV
- Free tier shows persistent ad warning banner
- YouTube Premium reduces ads but is not required

### Auto-Playlist Creation (Spotify Premium only)
- If no "Focus" or "Break" playlist found in user's library
- Fetches up to 100 liked songs via GET /v1/me/tracks
- Sorts by audio features: energy > 0.6 AND tempo > 110 BPM → Focus
- Remaining tracks → Break
- Creates "Focus - Auto" and "Break - Auto" private playlists
- Note: audio-features endpoint may be deprecated for new Spotify apps (403)
  Fallback: random 50/50 split of liked songs

## Known Issues / Watch Out For
- NEVER call PUT /v1/me/player/play without first calling
  resolveActiveDevice() and appending ?device_id=X. Spotify returns
  404 NO_ACTIVE_DEVICE if you skip this, even when the desktop app
  is open. This has broken once already.
- CSS !important is REQUIRED on all section show/hide rules — check this 
  first whenever a section is not visible. This now covers #plan-section and
  #plan-note as well as the three original sections
- Plan mode: the hash MUST be read before OAuth handling (see Plan Mode). A
  plan that vanishes when the user reconnects Spotify is this rule broken
- startPlayback() must return true before starting any timer
- All intervals and timeouts must be cleared in stopSession()
- Token refresh must happen before every Spotify API call
- YouTube autoplay requires a user gesture — Start button counts as one
- Mobile browsers freeze JS timers when screen is off — handled via 
  Page Visibility API + absolute timestamps + push notifications
- Spotify playback API requires Premium — free accounts get 403
- Spotify dev mode limited to 25 users — apply for Extended Quota for public release

## Roadmap

### Phase 2 — Browser Extension (next)
- Chrome extension (Manifest V3)
- Firefox extension port
- background.js for timer persistence when popup is closed
- Tab detection for open Spotify/YouTube tabs
- New OAuth redirect URIs for extension context
- Estimated time: 4-6 hours

### Phase 3 — Apple Music
- MusicKit JS integration
- Requires Apple Developer account ($99/year)
- Works best in Safari
- Estimated time: 3-4 hours

### Phase 4 — Android App
- Decision pending: PWA install vs React Native
- PWA already installable on Android via Chrome
- Estimated time: 2-8 hours depending on route

## Deployment
- Push to main branch → GitHub Pages auto-deploys in ~1 minute
- No build step needed
- Always test live URL after pushing

## Style Guide
- Fonts: DM Mono + Space Mono (Google Fonts)
- Primary color: #1db954 (Spotify green)
- Background: #0a0e27
- Card background: #151b3d
- Break/warning color: #f59e0b (amber)
- Error color: #ef4444 (red)
- All buttons: Space Mono, uppercase, letter-spacing: 1px
- Border radius: 12px inputs/buttons, 24px cards
