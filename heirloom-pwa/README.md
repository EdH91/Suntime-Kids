# Heirloom (PWA, Phase 1)

Heirloom is a parent-facing companion to the **Wide Eyes** journaling app. Parents, grandparents and other contributors record video or audio messages for a child, seal them with a time-lock (a date or a life milestone), and gift them later as a `.heirloom` file the child opens in Wide Eyes.

Vanilla HTML/CSS/JS. No framework, no build step, no npm dependencies. Everything is stored on the device in IndexedDB; nothing is uploaded.

```
heirloom-pwa/
  index.html      app shell (static markup for every screen)
  app.js          all logic, prompt library, stock backgrounds
  styles.css      design system + screens
  manifest.json   PWA manifest (relative paths)
  sw.js           cache-first service worker
  icons/          icon-192.png, icon-512.png
```

## Run locally

Any static file server works. Camera and microphone need a secure context, and `localhost` counts:

```bash
cd heirloom-pwa
python3 -m http.server 8080
# open http://localhost:8080/
```

It also works from a subfolder (for example `http://localhost:8080/heirloom-pwa/` when serving the repo root), because every path in `index.html`, `manifest.json` and `sw.js` is relative.

To test on a phone, serve over HTTPS (for example a Cloudflare Pages preview). `getUserMedia` is blocked on plain `http://` other than localhost.

### Checks before committing

```bash
node --check heirloom-pwa/app.js
node --check heirloom-pwa/sw.js
node -e "JSON.parse(require('fs').readFileSync('heirloom-pwa/manifest.json','utf8'))"
```

After changing any shell file, bump `CACHE` in `sw.js` (`heirloom-v1` → `heirloom-v2`) so installed copies pick up the new version. Old caches are deleted on activate.

## Coding rules (same as the rest of this repo)

- No inline event handlers. Every static element is wired exactly once in `wireStatic()`; dynamic lists use event delegation (`data-action`, `data-id`, ...). Buttons created at runtime inside modals and sheets are wired where they are created.
- No `innerHTML` with dynamic content. Build DOM with `el(tag, props, children)` and `clearEl(node)`. Names, titles and prompts are user or library text and always go through `textContent`.
- Strings containing apostrophes use double quotes or template literals.
- All IndexedDB and localStorage access is wrapped so failures show a friendly on-screen message.

## Screens

1. **PIN gate**: shown on every open. A PIN is matched against all contributors. "Add contributor" leads to contributor setup. With no contributors, onboarding is shown instead.
2. **Onboarding** (first launch): add a child (name, date of birth, optional photo), then create your account (name and 4-digit PIN). Sets `localStorage.onboardingComplete = "true"`.
3. **Home**: photo carousel (Zone A) with a "Ready to gift" banner and a featured sealed entry, then the parchment panel (Zone B) with filters and the entry list. Tap the child name to switch child when there is more than one.
4. **New recording**: prompt → record (video or audio, 1/3/5 min limit) → photos → seal.
5. **Entry detail**: metadata, sealed panel with "Unlock now" (and "set the date" for milestones), player once unlocked, photo strip, Edit/Delete menu, share.
6. **Share**: exports a `.heirloom` file and sets `sharedAt`.
7. **Contributors**: add, rename, change PIN, link to children, view their entries, remove.
8. **Settings**: children, contributors, back up all recordings (zip), switch contributor, clear all data.

## Data model

IndexedDB database `heirloom-db`, version 1.

### `heirloom-entries` (keyPath `id`, autoIncrement; indexes `date`, `childId`, `contributorId`)

| Field | Type | Notes |
|---|---|---|
| `id` | number | |
| `date` | ISO string | when it was recorded |
| `childId` | number or `'all'` | |
| `contributorId` | number | |
| `category` | `'family-stories'` `'milestones'` `'advice-letters'` `'us-together'` `'just-because'` | from the prompt, editable |
| `prompt` | string or null | null when the prompt was skipped |
| `title` | string | |
| `durationLimit` | 1, 3 or 5 | minutes |
| `durationSec` | number | actual length (Phase 1 addition, used for display) |
| `mediaB64` | DataURL | the recording |
| `mediaType` | string | the MediaRecorder MIME actually used, e.g. `video/mp4` or `audio/webm;codecs=opus` |
| `photos` | DataURL[] | JPEG, longest edge 1280px, quality 0.82 |
| `locked` | bool | |
| `revealDate` | ISO or null | local midnight of the chosen day |
| `revealMilestone` | string or null | label, e.g. `Turns 18` |
| `revealedAt` | ISO or null | set when a time-lock opens (automatically or via "Unlock now") |
| `sharedAt` | ISO or null | set when a `.heirloom` file is exported for gifting |
| `deleted` | bool | soft delete |

An entry left unsealed has `locked: false` and `revealedAt: null`. The "Ready to gift" banner shows entries with `locked: false`, a `revealedAt` and no `sharedAt`.

### `heirloom-profiles` (keyPath `id`, autoIncrement)

`{ id, type: 'contributor' | 'child', name, dob, photo, pin, linkedChildren }`

- Children: `dob` (ISO), `photo` (DataURL, 512px JPEG, or null).
- Contributors: `pin` (4 digits, unique across contributors), `linkedChildren` (child ids; empty means every child).

### `heirloom-bg` (keyPath `key`)

`{ key: 'bgPhotos', value: DataURL[] }`, at most 6. When an entry is saved with photos, its first photo is appended (oldest dropped). The home carousel uses user photos first and fills up to 3 slides with the bundled stock gradients; with 3 or more user photos the stock images are retired.

### localStorage

`onboardingComplete`, `heirloom-currentChild` (the last selected child), `heirloom-lastBackup` and `heirloom-backupSnooze` (ISO times, see Backups). Nothing else.

## Time-lock

`checkUnlocks()` runs after every successful PIN entry (and when the app returns to the foreground on the home screen). Every entry with `locked: true`, `revealedAt: null`, `deleted: false` and a `revealDate` on or before now is updated to `locked: false, revealedAt: now`.

Milestones are labels only. The parent confirms one by setting its date from the entry detail screen ("It has happened — set the date"); a date in the past unlocks immediately. There is no automatic milestone date calculation in Phase 1. "You decide" simply opens the date picker.

## `.heirloom` file format

A UTF-8 JSON file, MIME `application/x-heirloom`, named `<child-name>-<YYYY-MM-DD>.heirloom` (the recording date; `all-children-...` for entries addressed to every child).

```json
{
  "version": 1,
  "entry": { "id": 1, "date": "...", "title": "...", "mediaB64": "data:video/mp4;base64,...", "mediaType": "video/mp4", "photos": ["data:image/jpeg;base64,..."], "locked": true, "revealDate": "...", "...": "every field of the stored entry" },
  "contributorName": "Dad",
  "childName": "Maya",
  "exportedAt": "2026-10-22T18:00:00.000Z"
}
```

`version` and `entry` are the contract. `contributorName`, `childName` and `exportedAt` are convenience fields for the receiving app (ids are local to this device). A sealed entry is exported with its lock fields so Wide Eyes can keep it sealed.

## Backups

Everything lives only on the device, so Heirloom reminds the parent to save a copy elsewhere. A web app cannot write files off the phone on its own (iOS gives websites no background tasks and no file system access), so the backup is a reminder plus one tap.

- **Weekly prompt.** After PIN entry, at most once per app open, a "Time for a backup" prompt appears when:
  - there has never been a backup and the oldest recording is at least a day old, or
  - the last backup is 7 or more days old and at least one recording was made since.

  "Remind me tomorrow" snoozes it for 24 hours.
- **One file.** "Back up now" (or Settings → "Back up all recordings") zips every non-deleted entry as `.heirloom` files into `heirloom-backup-YYYY-MM-DD.zip`, using JSZip from cdnjs. Entries are stored uncompressed in the zip, since media and photos are already compressed.
- **Saving it.** A second tap, "Save backup", opens the share sheet where the browser supports sharing files (iOS: Save to Files / iCloud Drive, AirDrop, Mail). Elsewhere it downloads. iOS only allows the share sheet straight after a tap, which is why there are two steps. Cancelling the share sheet does not count as a backup.
- **Offline.** JSZip is prefetched after login while online and cached by the service worker, so backups work offline afterwards. If it has never been fetched, the prompt asks to try again online; Settings falls back to downloading the `.heirloom` files one at a time.
- Heirloom also asks the browser for persistent storage (`navigator.storage.persist()`), which makes eviction less likely where supported.
- Backups do not set `sharedAt`, and do not include profiles or PINs.

**There is no restore yet.** The `.heirloom` files in a backup open in Wide Eyes, but Heirloom itself cannot import a backup in Phase 1.

## Known limitations (Phase 1)

- **PINs are not security.** They are stored in plain text in IndexedDB, and anyone holding the unlocked device can add a contributor from the PIN screen. The PIN separates family members; it does not protect data.
- **Storage size.** Media is stored as base64 DataURLs (about 33% larger than the raw file). Video is recorded at roughly 0.8 Mbps, so a 5-minute video is around 40 MB in storage and in the exported file. Browsers, especially iOS Safari, may evict or cap storage; the weekly backup prompt exists for this reason.
- **iOS.** Safari records `video/mp4` / `audio/mp4`; Chromium records MP4 or WebM depending on version. A WebM recording made on Android or desktop may not play on older iOS. Installed home-screen apps on iOS keep a separate storage area from Safari tabs.
- **Sharing.** Export uses an `<a download>` blob link, as specified. On iOS this saves to Files and the parent shares from there; the Web Share API (direct AirDrop sheet) is not used yet.
- **Recording only while visible.** Locking the phone or switching apps mid-recording can stop the browser's recorder.
- **No sync or restore.** `.heirloom` files and backup zips are the only way data leaves the device. Heirloom cannot import them back yet, and there is no cloud or peer sync.
- **Removing a child** keeps their recordings on the device but hides them from the home screen.
- **Fonts** come from Google Fonts; they are cached by the service worker after the first online visit and fall back to system serif/sans offline.
