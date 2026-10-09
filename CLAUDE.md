# SunTimeKids — CLAUDE.md

Developer guide for AI-assisted work on this codebase. Read this before making any changes.

---

## What this app is

SunTimeKids is a mobile-first PWA (single HTML file) that tracks children's UV exposure and vitamin D synthesis in real time. It uses live GPS, the Open-Meteo UV Index API, each child's Fitzpatrick skin type, clothing coverage level, and SPF to calculate vitamin D progress toward a daily goal, and to warn when burn risk is approaching.

**Live URL:** https://suntime-kids.pages.dev  
**Repo:** https://github.com/EdH91/Suntime-Kids  
**Owner:** Ed Hancox / Discendia Labs

---

## Architecture

- **Single-file PWA** — all HTML, CSS, and JS in `index.html`. No build step, no framework, no bundler.
- **No backend** — all data lives on-device (COPPA compliant).
- **Storage:** IndexedDB (primary) → localStorage (fallback) → cookie (last resort). Handles iOS PWA quirks.
- **UV data:** Open-Meteo API (`api.open-meteo.com`) — free, no API key needed, global coverage.
- **Geocoding:** Open-Meteo geocoding API for city name lookup.
- **Fonts:** Nunito (body) and Baloo 2 (display headings) loaded from Google Fonts.

### Screen routing

The app uses `goTo(id)` to toggle the `.active` class between named `<div id="screen-*">` elements. `goTo` also re-renders the target screen and shows the bottom nav on home/dash/settings. Screens:

| ID | Purpose |
|----|---------|
| `screen-welcome` | First launch / no profiles |
| `screen-onboard` | Add/edit the two profiles (also reached via Settings → Edit Profiles) |
| `screen-home` | Per-child daily progress cards, weekly stats |
| `screen-whosout` | Bottom sheet: pick which children are going out |
| `screen-setup` | Per-child clothing & SPF setup |
| `screen-session` | Active tracking session (swipeable per-child cards) |
| `screen-celebrate` | Goal reached prompt for one child |
| `screen-dash` | Week chart, trends, per-child summary, history |
| `screen-settings` | Goal-reached behaviour, disclaimer, privacy |

---

## Key calculations

### Vitamin D synthesis

Base minutes to synthesise 400 IU at UV Index 5, by Fitzpatrick skin type (I–VI):

```
[8, 12, 16, 22, 32, 45]
```

Adjusted for current UV Index: `baseMinutes * (5 / uvIndex)`.

Further multiplied by exposure factor (clothing coverage × SPF reduction).

### Burn risk

Base minutes to burn at UV Index 5, by Fitzpatrick type:

```
[15, 20, 30, 45, 60, 90]
```

Same UV adjustment formula applies.

### Live UV during a session

UV is refreshed every 5 minutes (`UV_REFRESH_MS`), and also on session start or return from background if the last reading is stale. When a reading **changes** during a session, `applyUVToSession()` banks each child's vitamin D and burn totals at the old rate (`vitdBanked`, `burnBanked`, `rateSince`), then recomputes `vitdPerSec`/`burnPerSec` for the new UV. Always read totals via `vitdAt(cs, elapsed)` / `burnAt(cs, elapsed)`, never `elapsed * cs.vitdPerSec`.

- A failed refresh keeps the last good reading (no re-rate).
- Children marked `done` are not re-rated, so their totals stay frozen.
- Time spent backgrounded is banked at the UV in effect before the app was suspended (no historical UV is fetched).
- The saved session `uvIndex` is the time-weighted average (`sessionAverageUV`).

### Daily goals

- Child (0–12): 400 IU  
- Teen (13–18): 600 IU  
- Adult: 600 IU

---

## Critical coding rules

### 1. Button wiring — addEventListener only

**Never use inline `onclick` attributes.** Cloudflare Pages enforces a Content Security Policy that blocks inline event handlers. All buttons must be wired via `addEventListener` in JavaScript.

Correct pattern:
```js
document.getElementById('btn-start').addEventListener('click', function() { ... });
```

Wrong (will silently break on Cloudflare):
```html
<button onclick="startSession()">Start</button>
```

The function `wireButtonsSync()` runs as an IIFE at script load (before `appInit()`) to ensure buttons are wired before any async work completes. **It is the only place static buttons are wired.** Wiring the same button again elsewhere (as `appInit()` used to) makes every tap fire twice, which broke Pause (pause + instant resume) and made End/Done throw. Elements created at runtime (e.g. the notification banner) are wired where they are created, and dynamic grids use event delegation.

### 2. No string concatenation in DOM rendering

Use `document.createElement` for all dynamic DOM construction. **Never use `innerHTML` with JS variables or template literals that include untrusted/dynamic content.** This avoids both CSP violations and XSS surface. (Child names are user input; location names come from the geocoding API.)

Use the `el(tag, props, children)` helper for concise construction (`text` sets `textContent`, `style` sets `cssText`), plus `clearEl(node)` to empty a container. `renderSessionCards()` shows the long-form `createElement` style.

### 3. No apostrophes or special characters in JS strings

JavaScript string literals in this file must not contain apostrophes (`'`) unless the string is delimited by double quotes or backticks. The previous pattern of using single-quoted strings with contractions (`'It's'`, `'We're'`) caused `SyntaxError` in some environments. Prefer plain words without contractions, or use template literals.

### 4. Syntax-check before deploying

Before committing and pushing (which deploys), extract the `<script>` block and syntax-check it:

```bash
awk '/<script>/{f=1;next}/<\/script>/{f=0}f' index.html > /tmp/stk-check.js && node --check /tmp/stk-check.js && echo OK
```

(`node --check index.html` does **not** work: Node rejects the `.html` extension with `ERR_UNKNOWN_FILE_EXTENSION` without parsing anything.)

A syntax check does not catch logic bugs. For behaviour, serve the folder locally (`python3 -m http.server 8765`, also configured in `.claude/launch.json`) and click through: onboarding → head out → setup → start → pause/resume → goal → Done/Keep Playing → dashboard → reload.

---

## extendedMode — how it works

When a child reaches their vitamin D goal and the parent taps "Keep Playing Outside", the child enters `extendedMode`. Rules:

- `vitdPercent` is pinned at 100% — the ring stays full.
- Burn risk continues to climb from its current level (not reset).
- The session timer keeps running (no stop at goal).
- `extendedMode` is tracked **per child**, not globally — so one child's extension doesn't affect another's active tracking.

The flag lives in the per-child session object in `APP.childSessions[i]` (`cs.extendedMode = true`).

Related per-child flow:

- Celebrations are shown **one child at a time** (`APP.currentCelebrationSliderIdx`). Another child who hits the goal meanwhile is celebrated on the next tick after the parent answers.
- "Done for <name>" sets `cs.done = true` and returns to the session for the other children. Only when nobody else is active does it end the whole session.
- Settings → "Show 60-second prompt" (`sessionEndMode: 'auto'`): if nobody answers within 60 s, the child is moved to extendedMode automatically. "Keep tracking automatically" (`'keep'`) skips the celebration screen.

---

## Timer implementation

The timer uses **timestamp-based tracking**, not `setInterval` tick counting. This means:

- On session start, record `session.startTime = Date.now()`.
- `getElapsedSeconds()` = `pausedElapsed + (Date.now() - startTime)`, or just `pausedElapsed` while paused.
- This survives app backgrounding on iOS and Android without drift. The visibility handler only stops/starts the interval; it does not touch the timestamps.
- UV 0 (night) is a real reading. Use `currentUV()`, never `APP.uvIndex || 4`.

`totalMins` is computed as a plain integer — do not use `padStart` with a fixed width that caps at two digits, as this broke the timer at 60+ minutes in v2.4.

---

## Storage pattern

`storeSet(key, val, cookieVal?)` writes to **all three**: IndexedDB, localStorage, and a base64 cookie. `storeLoad(key)` reads IndexedDB → localStorage → cookie → in-memory. Keys: `stk_profiles`, `stk_sessions`, `stk_session_end_mode`.

- Cookies are skipped when the encoded value exceeds ~3.8 KB (browsers silently drop larger ones). Profiles pass a photo-less `cookieVal`.
- Photos are downscaled to 256 px JPEG before being stored.

All storage reads and writes are wrapped in `try/catch`. The app must degrade gracefully when storage is unavailable (private browsing on iOS).

### Session record shape

```js
{ id, date /* start time, ISO */, childIndices, elapsedSeconds, uvIndex, location,
  perChild: [{ childIndex, vitd /* capped at goal */, goal, goalReached, elapsedSeconds }],
  vitdAccumulated, goalReached /* session-level summary, legacy */ }
```

Records saved before v2.5 have no `perChild`. Always read through `sessionChildResults(s)`, which handles both shapes. Daily goals can be met across several sessions (`childGoalHitOnDay`).

---

## Deployment process

Cloudflare Pages is connected to the GitHub repo and **deploys automatically on every push to `main`**. There is no manual upload step.

1. Syntax-check the script (see rule 4 above) and test locally.
2. Commit and push to `main`, either with `git push origin main` or the **Push origin** button in GitHub Desktop.
3. Within a minute or two the new version is live at https://suntime-kids.pages.dev. Check Cloudflare → Workers & Pages → `suntime-kids` → Deployments if needed.

Do **not** edit `index.html` on GitHub's website or use Cloudflare Direct Upload. Web edits make the local copy fall behind, and a direct upload is overwritten by the next push.

Installed home-screen copies of the PWA may keep the old version until the app is fully closed and reopened.

---

## What to check when something breaks

| Symptom | Likely cause |
|---------|-------------|
| Buttons don't respond | `onclick` attribute used instead of `addEventListener` |
| Button acts twice / Pause does nothing | Same button wired in two places. Wire only in `wireButtonsSync()` |
| Blank screen / JS error in console | Syntax error. Run the extract + `node --check` command above |
| App works on desktop but not iPhone | iOS PWA storage issue — check IndexedDB/localStorage fallback |
| Timer stops at 1 hour | `padStart` or string coercion capping `totalMins` to 2 digits |
| "Keep Playing Outside" resets vitd to 0% | `extendedMode` flag not set or not checked in render loop |
| UV data not loading | Open-Meteo API rate limit or GPS permission denied |
| Child celebration ends wrong child's session | `extendedMode` set globally instead of per-child |

---

## What not to change without careful thought

- **The Fitzpatrick calculation arrays** — these are clinically derived values. Any change needs a source.
- **The triple-fallback storage chain** — required for iOS PWA compatibility. Don't simplify it without thorough iOS testing.
- **The `wireButtonsSync()` IIFE** — must run synchronously at script load, before `appInit()`. Moving it inside an async function will break button wiring on slow loads.
- **CSP-incompatible patterns** — see rules 1 and 2 above. Cloudflare Pages is strict; GitHub Pages is stricter still.

---

## Planned next features (v2.6+)

- Freemium gate: enforce 2-profile limit and show upgrade prompt
- Cloud backup/sync (Pro tier)
- Sunscreen reapplication reminder improvements
- React Native build for App Store submission

---

## Legal & IP notes

- Copyright: Ed Hancox / Discendia Labs (automatic, all rights reserved)
- Trademark "SunTimeKids": USPTO TESS search pending before wider release
- Provisional patent: UV calculation algorithm is potentially patentable — consult IP attorney before open-sourcing any core calculation logic

---

*Last updated: October 2026 — v2.5*
