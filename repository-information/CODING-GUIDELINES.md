# Coding Guidelines

Domain-specific coding knowledge and architectural reference for this project. Claude reads this file on demand when working on the relevant feature area.

CLAUDE.md sections reference this file with: *See `repository-information/CODING-GUIDELINES.md` — section "X"*

---

## GAS Code Constraints
- **All GAS `.gs` code must be valid Google Apps Script syntax** — test mentally that strings, escapes, and quotes parse correctly before committing
- Avoid deeply nested quote escaping in HTML strings built inside `.gs` files. Instead, store values in global JS variables and reference them in `onclick` handlers (e.g. `_signInUrl` pattern)
- **`readPushedVersionFromCache()` must NOT delete the cache entry** — it must return the value without calling `cache.remove()`. Deleting it causes only the first polling client to see the update; all others miss the "Code Ready" blue splash reload. The cache has a 1-hour TTL and expires naturally.
- The GAS auto-update "Code Ready" splash flow works as follows:
  1. GitHub Actions workflow calls `doPost(?action=deploy)` on the **old** deployed GAS
  2. `pullAndDeployFromGitHub()` fetches new code from GitHub, updates the script, creates a new version, updates the deployment
  3. It writes the new version string to `CacheService.getScriptCache()` with key `"pushed_version"`
  4. Client-side JS polls `readPushedVersionFromCache()` every 15 seconds
  5. If the returned version differs from the version displayed in `#gv`, it sends a `gas-reload` postMessage to the parent embedding page
  6. The embedding page receives the message, sets session storage flags, reloads, and shows the blue "Code Ready" splash

---

## Race Conditions — Config vs. Data Fetch
- **Never fire `saveConfig` and a dependent data-fetch (`getFormData`) in parallel** — the data-fetch may read stale config values from the sheet
- When the client switches a config value (e.g. year) and needs fresh data for that value, **pass the value directly as a parameter** to the server function (e.g. `getFormData(_token, year)`) rather than relying on `saveConfig` completing first
- Server functions that read config should accept optional override parameters (e.g. `opt_yearOverride`) so the client can bypass the saved config when needed
- This pattern avoids race conditions without needing to chain callbacks (which adds latency)

---

## API Call Optimization (Scaling Goal)
- **Minimize Google API calls** in every GAS function — the app is designed to scale to many users
- **Cache `getUserInfo` results** in `CacheService` (keyed by token suffix) for 5 minutes to avoid hitting the OAuth userinfo endpoint on every `google.script.run` call
- **Cache `checkSpreadsheetAccess` results** in `CacheService` (keyed by email) for 10 minutes to avoid listing editors/viewers on every call
- **Open `SpreadsheetApp.openById()` once per function** — pass the `ss` object to `checkSpreadsheetAccess(email, opt_ss)` instead of opening the spreadsheet twice
- When adding new server-side functions, always consider: can this result be cached? Can I reuse an already-opened spreadsheet object? Avoid redundant `UrlFetchApp` or `SpreadsheetApp` calls
- Cache TTLs are intentionally short (5–10 min) so permission changes and token revocations take effect quickly

---

## UI Dialogs — No Browser Defaults
- **Never use `alert()`, `confirm()`, or `prompt()`** — all confirmation dialogs, alerts, and input prompts must use custom styled HTML/CSS modals
- This applies to both GAS `.gs` code and parent embedding pages (`.html`)
- Use overlay + modal patterns consistent with the existing sheet/modal styles in the codebase

---

## AudioContext & Browser Autoplay Policy
- **AudioContext starts as `'suspended'`** on every page load — browsers require a user gesture (click/touch) before allowing audio playback
- **`resume()` without a gesture** generally stays pending or silently fails. It does NOT reject — the promise just never resolves, which causes dangling `.then()` callbacks that fire unexpectedly when the user eventually clicks
- **Never schedule `decodeAudioData` + `source.start(0)` while context is suspended** — the audio gets queued and plays the moment the context resumes (on user click), causing surprise delayed sound. Instead, gate playback behind `if (ctx.state !== 'running') return`
- **JS-triggered `window.location.reload()` vs manual F5 refresh** behave differently: JS reload carries forward the gesture allowance (AudioContext can auto-resume), F5 does NOT. So auto-refresh reloads can play sound, but manual refreshes cannot
- **`onstatechange` fires before DOM is ready**: when `resume()` is called early in the script, `onstatechange` may fire before the `#audio-status` element exists — the `updateAudioStatus()` call bails silently and never retries. Fix: save the resume promise and chain `updateAudioStatus` onto it after the element is created
- **Use `sessionStorage` (not `localStorage`) for audio state flags** — `sessionStorage` is per-tab, so a flag set in one tab doesn't leak into new tabs that have no gesture context. `localStorage` would cause false-positive "sound ready" icons in fresh tabs
- **The `audio-unlocked` sessionStorage flag** remembers that audio was successfully unlocked in this tab. On F5 refresh, the AudioContext is suspended but the flag tells the icon to show "ready" instead of "muted" — because a click will instantly restore it. Without this flag, the icon flashes "muted" on every refresh even though audio works fine
- **Chrome "Duplicate Tab" copies `sessionStorage`** — including the `audio-unlocked` flag — into the new tab, but the new tab has no gesture context, so the AudioContext is suspended. The stale flag causes the icon to falsely show "sound ready." Fix: use `performance.getEntriesByType('navigation')` to detect the navigation type; if it's anything other than `'reload'` (e.g. `'navigate'` for duplicate tab, back/forward, or new navigation), clear the `audio-unlocked` flag before creating the AudioContext. This must run **before** `new AudioContext()` so that `updateAudioStatus()` sees the correct flag state from the start

---

## Google Sign-In (GIS) for GAS Embedded Apps
When a GAS app embedded in a GitHub Pages iframe needs Google sign-in (e.g. to restrict access to authorized users), the sign-in **must run from the parent embedding page**, not from inside the GAS iframe.

### Why
- GAS iframes are served from dynamic `*.googleusercontent.com` subdomains with long hash-based hostnames that change when the deployment changes — they can't be reliably registered as OAuth origins
- Google OAuth requires the JavaScript origin to be registered in Cloud Console
- The parent GitHub Pages domain (e.g. `<your-org>.github.io`) is a stable origin that can be registered once

### Architecture
1. **GAS iframe** detects auth is needed → sends a `gas-needs-auth` postMessage to the parent (with `authStatus` and `email` fields)
2. **Parent embedding page** receives the message → shows an auth wall overlay → loads GIS and triggers sign-in popup
3. After successful sign-in → parent hides the auth wall → reloads just the iframe (`iframe.src = iframe.src`)
4. GIS code (Google Identity Services library) lives **only** in the parent HTML, never in the `.gs` file

### OAuth Setup (Google Cloud Console)
- **OAuth Client ID**: Create or use an existing OAuth 2.0 Client ID from your Google Cloud project (format: `<client-id>.apps.googleusercontent.com`)
- **Authorized JavaScript origins** must include your GitHub Pages domain (e.g. `https://<your-org>.github.io`) and any custom domains pointing to it
- To configure: Google Cloud Console → APIs & Services → Credentials → OAuth 2.0 Client IDs → edit the client → add the origin
- If you add new embedding domains (e.g. a custom domain), add those origins too

### Key postMessage Types for Auth
| Message Type | Direction | Purpose |
|---|---|---|
| `gas-needs-auth` | GAS iframe → parent | Tells parent to show sign-in wall (includes `authStatus`, `email`) |
| `gas-auth-complete` | GAS iframe → parent | Tells parent auth succeeded (hides wall, reloads iframe) |

Developed by: ShadowAISolutions
