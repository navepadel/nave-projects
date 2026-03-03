# NAVE Project — Claude Reference

## Repo & Deployment

- **GitHub repo:** `navepadel/nave-app` (remote alias: `github`)
- **Live URL:** `https://app.navepadel.com/` (custom domain via `CNAME` file)
- **Hosting:** GitHub Pages, deployed from `main` branch
- **Local proxy remote:** `origin` → `http://local_proxy@127.0.0.1:27270/git/paulotrigo-eng/NAVE`
- **PAT:** stored in `github` remote URL; use for GitHub API calls with `Authorization: token <PAT>`
- **Push pattern:** `git push github main` to deploy; `git push -u origin <branch>` for working branches

## Project Structure

```
/home/user/NAVE/
├── time.html                  # Staff check-in/out app (main deliverable)
├── gate-monitor.js            # Shelly script: real-time gate open/close → Telegram
├── heartbeat.js               # Shelly script: daily 9AM health check → Telegram
├── shelly-heating-monitor-telegram.js  # Shelly heating monitor
├── CNAME                      # app.navepadel.com
└── README.md                  # Documents gate-monitor + heartbeat scripts
```

The **nave-app repo also contains** (not mirrored locally): `index.html`, `og-tools.png`, `og-time.png`.

## time.html — Staff Check-in/out App

### Purpose
A single-page, no-framework web app used by NAVE Padel staff to clock in and out. Staff enter their phone number; the app looks up their identity via webhook, then allows them to register entry/exit.

### Tech Stack
- Vanilla HTML/CSS/JS — zero dependencies, no build step
- Font: **Barlow** (Google Fonts)
- Language: Portuguese (pt-PT locale throughout)
- Backend: **Make.com** webhooks → **Fibery** database

### Three Screens
1. **Screen 1 (`screen-phone`)** — Phone number entry
2. **Screen 2 (`screen-action`)** — Check-in / check-out action + time picker + recent entries list
3. **Screen 3 (`screen-confirm`)** — Confirmation with animated countdown, then auto-reset

### State Object
```js
let state = {
  phone:         '',    // E.164 format, e.g. "+351912345678"
  staffId:       '',    // Fibery UUID
  staffName:     '',
  nif:           '',
  email:         '',
  openEntry:     null,  // { id, checkInTime } or null — active check-in
  recentEntries: [],    // last 10 entries from lookup webhook
  selectedTime:  null,  // Date chosen in time picker (null = use Date.now())
};
```

### Webhook Contract

**LOOKUP_WEBHOOK** `POST`
- Request: `{ phone: "+351..." }`
- Response (found):
  ```json
  {
    "found": true,
    "staffId": "fibery-uuid",
    "name": "...",
    "email": "...",
    "nif": "...",
    "openEntry": { "id": "uuid", "checkInTime": "ISO8601" } | null,
    "recentEntries": [
      { "checkIn": "ISO8601", "checkOut": "ISO8601|null", "status": "...", "statusDate": "ISO8601|null" }
    ]
  }
  ```
- Response (not found): `{ "found": false }`

**CHECKIN_WEBHOOK** `POST`
- Request: `{ staffId, staffName, nif, email, phone, timestamp: ISO8601 }`
- Response: `{ success: true, entryId: "..." }`

**CHECKOUT_WEBHOOK** `POST`
- Request: `{ entryId, staffId, staffName, nif, email, phone, timestamp: ISO8601 }`
- Response: `{ success: true, duration: "8h 30m" }` (duration optional)

### Key Conventions
- **Phone normalisation:** strips non-digits, handles `00351`/`351` prefixes, always sends `+351XXXXXXXXX`
- **Time picker:** defaults to `now`; clamps future values back to `now`; shows retroactive hint (e.g. "15min atrás")
- **Check-in guard:** if any `recentEntries` entry has `checkOut: null`, check-in button is hidden and an admin-contact error is shown
- **`openEntry` detection:** checked via `data.openEntry && data.openEntry.exists` — webhook must set `exists: true` on the object; `null` or absent means no open entry
- **Timestamps sent to webhooks:** always `state.selectedTime || new Date()` — respects retroactive time edits
- **Auto-reset:** after confirmation, countdown bar animates for `CONFIG.resetDelay` seconds (default 6), then calls `reset()`
- **Duration live update:** when on Screen 2 with an open entry, duration refreshes every 60 s via `window._durationTimer`

### CSS Colour Tokens
```css
--ocre:       #BC5828   /* orange/rust — primary accent */
--verde:      #006E54   /* green — checkout button, duration badges */
--black:      #0F0D0A
--cream:      #F7F4EF   /* page background */
--cream-dark: #EDE9E1   /* borders, dividers */
--grey:       #7F7870
--grey-light: #C8C4BC
--white:      #FFFFFF
--error:      #C0392B
```

### Version
Footer displays current version. Bump `v1.1.x` in the footer `<div>` when shipping changes.
Current: **v1.1.4**

## Shelly Scripts

- **`gate-monitor.js`** and **`heartbeat.js`** run on a Shelly IoT device
- Telegram Bot Token: `7456309482:AAHOfsj_WZQ5JV5aSi1VsM193j1pdoWsNbQ`
- Telegram Chat ID: `@pratesmatrix`
- Gate input: `Input ID 0`
- Alert if gate open > 2 min (`120000 ms`)
- Heartbeat fires daily at 09:00 via `Schedule.Create`

## Make.com Architecture

Three webhook scenarios (all on `hook.eu2.make.com`) connect `time.html` to **Fibery** (database).

### Scenario 1 — Lookup (`lookupWebhook`)

**Module flow:**
1. **Webhook trigger** — receives `{ phone }` POST
2. **Fibery → Find Records** — search `People / Staff` where `Phone = phone`; returns staff record (id, Name, Email, NIF)
3. **Module A — Find Open Time Entry** — `People / Time Entries` where `Staff = [staff.id]` AND `Check Out` is empty; Sort: `Check In` desc; Limit: 1
4. **Module B — Find Recent Time Entries** — `People / Time Entries` where `Staff = [staff.id]`; Sort: `Check In` desc; Limit: 10
5. **Webhook Response** — returns JSON body (see below)

**Response body template:**
```json
{
  "found": true,
  "staffId": "{{staff.id}}",
  "name": "{{staff.Name}}",
  "email": "{{staff.Email}}",
  "nif": "{{staff.NIF}}",
  "openEntry": {{if(moduleA.id; formatJSON({"id": moduleA.id, "checkInTime": moduleA.`Check In`}); "null")}},
  "recentEntries": {{toArray(moduleB; {
    "checkIn":    formatDate(item.`Check In`; "YYYY-MM-DDTHH:mm:ss"),
    "checkOut":   if(item.`Check Out`; formatDate(item.`Check Out`; "YYYY-MM-DDTHH:mm:ss"); null),
    "status":     item.State.name,
    "statusDate": item.`Status Date`
  })}}
}
```

> Note on `openEntry`: the frontend checks `data.openEntry && data.openEntry.exists`. The Fibery module must set `exists: true` on the object, or the response must include `"exists": true` inline. Alternatively use a Set Variable module to build the JSON conditionally.

**Duration formatting** (no built-in formatter in Make.com):
```
{{floor(dateDifference(item.`Check In`; item.`Check Out`; "minutes") / 60)}}h {{mod(dateDifference(item.`Check In`; item.`Check Out`; "minutes"); 60)}}min
```

**Not-found path:**
```json
{ "found": false }
```

---

### Scenario 2 — Check-in (`checkinWebhook`)

Receives: `{ staffId, staffName, nif, email, phone, timestamp }`

Creates a new `Time Entries` record in `People` space:
- `Staff` → linked by `staffId`
- `Check In` → `timestamp` (ISO 8601 from frontend — may be retroactive)

Returns: `{ "success": true, "entryId": "fibery-uuid" }`

---

### Scenario 3 — Check-out (`checkoutWebhook`)

Receives: `{ entryId, staffId, staffName, nif, email, phone, timestamp }`

Updates the `Time Entries` record identified by `entryId`:
- `Check Out` → `timestamp`

Returns: `{ "success": true, "duration": "8h 30min" }` (duration is optional; frontend computes it locally if absent)

---

### Fibery Data Model

Space: `People`
- **Staff** entity: fields include `Name`, `Email`, `NIF`, `Phone`
- **Time Entries** entity: fields include `Staff` (link), `Check In` (datetime), `Check Out` (datetime, nullable), `State` (status with `.name`), `Status Date`

---

### recentEntries — role in frontend logic

`recentEntries` from the lookup response drives two things:
1. **History list** on Screen 2 — shows last 10 entries with date, times, duration badge, and optional status
2. **Check-in guard** — if any entry has `checkOut: null`, the check-in button is hidden and staff must contact admin to resolve

## Important Decisions

- **Single-file app:** `time.html` is entirely self-contained (HTML + CSS + JS in one file). No bundler, no framework.
- **No future times:** time picker `max` attribute is kept at current time and enforced in JS too.
- **Backward-compatible webhook:** `recentEntries` defaults to `[]` if absent — old Make.com scenarios still work.
- **`openEntry.exists` check:** webhook returns `{ exists: true, id, checkInTime }` or `null`; the frontend checks `data.openEntry && data.openEntry.exists` to distinguish the two cases.
- **og: URLs:** All Open Graph and Twitter card meta tags use `https://app.navepadel.com/` (not the `.github.io` URL).
