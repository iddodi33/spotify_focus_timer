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

### Timer Logic
- All session times stored as absolute timestamps (Date.now()) not countdowns
- startFocusSession() → stores focus_session_start, calls startPlayback('focus')
- endFocusSession() → increments session count, calls startBreakSession()
- startBreakSession() → stores break_session_start, calls startPlayback('break')
- stopSession() — MUST clear ALL of these or they leak:
  sessionTimer, progressInterval, commCheckInterval,
  transitionPlaybackTimeout, transitionNextSessionTimeout
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
- CSS !important is REQUIRED on all section show/hide rules — check this 
  first whenever a section is not visible
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
