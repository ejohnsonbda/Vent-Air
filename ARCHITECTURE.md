# VENT — Architecture & Technical Reference

## Overview

VENT is an anonymous emotional venting web app delivered as a Progressive Web App (PWA). Users pick an emotion, write freely, and submit — no account, no login, no trace. A separate password-protected admin portal lets authorized users browse and analyze every submission in real time.

The entire product is two HTML files deployed to GitHub Pages, backed by a Supabase PostgreSQL database. There is no build step, no server to run, and no package manager to maintain.

---

## Repository Structure

```
ejohnsonbda/Vent-Air (GitHub Pages)
│
├── index.html          # The VENT app (user-facing)
├── portal.html         # Admin portal (password-protected)
├── manifest.json       # PWA manifest
├── icons/
│   ├── icon-192.png
│   └── icon-512.png
└── ARCHITECTURE.md     # This document
```

---

## Front End — The VENT App (`index.html`)

### Technology Stack

| Layer | Choice | Why |
|---|---|---|
| UI framework | React 18 (UMD CDN) | No build step; JSX compiled in-browser |
| JSX compiler | Babel Standalone | Transforms JSX at runtime |
| Database client | Supabase JS v2 (UMD CDN) | Single CDN script, no npm |
| Styling | Inline CSS-in-JS + `<style>` block | Self-contained, no external sheets |
| Hosting | GitHub Pages | Free, zero-config static hosting |

All scripts are loaded from CDN — the file is fully self-contained and works offline after first load via the browser cache.

### App Shell Layout

The app uses a single `#root` div that fills the entire viewport:

```
#root  (position: fixed; inset: 0)
  └── .app-shell
        └── <CurrentScreen />
```

`.app-shell` is a CSS-only responsive container:

- **Mobile (< 640px):** fills the full screen — `width: 100%; height: 100%;` — respects safe-area insets for notches and home indicators
- **Tablet (≥ 640px):** centers a 420 × 840px column with rounded corners and a box shadow, like a phone on a desk
- **Laptop (≥ 1024px):** same column, slightly taller (900px max)

### State Machine

The app's entire state lives in a single React `useState` object managed at the top-level `App` component. Navigation is a string `route` field — there is no React Router.

```js
{
  route:       'welcome',   // current screen
  lastRoute:   null,        // previous screen (for back navigation)
  emotion:     null,        // { label, category, emoji } chosen by user
  vent:        '',          // the text the user typed
  submitError: null         // error message from Supabase, if any
}
```

All state mutations happen through a `setState` function passed down via a `AppCtx` React context, so every screen can read and write state without prop drilling.

### Screen Flow

```
welcome
  └── emotion         ← pick from 84 feelings (grid or list, filterable by category)
        └── vent      ← free-text entry (min 1 char to proceed)
              └── review     ← read back what you wrote + chosen emotion
                    ├── submitting  ← spinner while Supabase insert runs
                    │     ├── thanks     ← success (auto-advances after 2.5s)
                    │     └── [error]    ← inline error + "Try again" / "Go back"
                    └── breathe    ← optional 4-7-8 breathing exercise
                          └── support    ← mental health resource links
                                └── privacy  ← privacy statement
```

Each screen is a standalone function component. The `key={state.route}` on the screen wrapper forces a re-mount on every route change, giving each screen a clean slate.

### Feelings Data

84 emotions are hard-coded in a `FEELINGS` array. Each entry has:

```js
{ label: 'Anxiety', category: 'Fear', emoji: '😰' }
```

Categories: **Fear · Sadness · Anger · Joy · Love · Shame · Confusion · Other**

The emotion screen renders these in a filterable grid. Selecting one sets `state.emotion` and advances to the `vent` screen.

### Supabase Integration (Submit)

The Supabase client is initialized once at the top of the Babel script:

```js
const db = window.supabase.createClient(
  'https://pkgvgkleujecyvqvjojb.supabase.co',
  '<anon-key>'
);
```

When the user confirms their vent on the review screen, `actions.submit()` runs:

```js
submit: async () => {
  setState(s => ({ ...s, route: 'submitting', submitError: null }));
  try {
    const { error } = await db.from('vents').insert({
      'I am felling': state.emotion?.label ?? null,
      'vb_Anonymous': state.vent,
      'Owner':        null,
    });
    if (error) throw error;
    setState(s => ({ ...s, route: 'thanks' }));
  } catch (err) {
    setState(s => ({ ...s, submitError: err.message || 'Something went wrong.' }));
  }
}
```

The `ID`, `Created Date`, and `Updated Date` fields are auto-populated by Supabase (via `DEFAULT gen_random_uuid()` and `DEFAULT now()`).

### PWA Configuration

`manifest.json` registers the app as an installable PWA:

```json
{
  "name": "VENT",
  "short_name": "VENT",
  "display": "standalone",
  "background_color": "#0B0B0B",
  "theme_color": "#0B0B0B",
  "start_url": "/",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192" },
    { "src": "icons/icon-512.png", "sizes": "512x512" }
  ]
}
```

`index.html` includes Apple-specific meta tags for iOS home screen install:

```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="viewport" content="viewport-fit=cover, ...">
```

---

## Front End — The Admin Portal (`portal.html`)

### Technology Stack

| Layer | Choice |
|---|---|
| UI framework | Vanilla JS (no framework) |
| Database client | Supabase JS v2 (UMD CDN) |
| Styling | Inline `<style>` block, CSS custom properties |
| Hosting | GitHub Pages (same repo) |

### Authentication

The portal has a client-side login screen. Credentials are loaded from `./passcore.json` at startup. If that file is missing or unreachable, the fallback credential is `admin / 8804114`.

```js
async function loadCredentials() {
  const res = await fetch('./passcore.json?' + Date.now());
  VALID_CREDENTIALS = parseCredentials(await res.json());
  // fallback is always included:
  VALID_CREDENTIALS.push({ username: 'admin', password: '8804114' });
}
```

On successful login, `sessionStorage` is written so refreshing the page skips the login form. Logout clears `sessionStorage` and shows the form again.

> **Note:** This is a client-side credential check only. It is not a replacement for true server-side authentication — it prevents casual access but is not cryptographically secure. The anon key in the source can be seen by anyone who views page source.

### Data Loading

After login, `loadData()` fetches all rows from Supabase ordered newest-first:

```js
async function loadData() {
  const { data, error } = await db
    .from('vents')
    .select('*')
    .order('Created Date', { ascending: false });

  DATA = (error || !data?.length) ? getSampleData() : data;
  initializeApp();
  subscribeRealtime();
}
```

If Supabase is unreachable (no SELECT policy, network error, etc.), it falls back to six built-in sample records so the UI is never blank.

### Real-Time Updates

After the initial load, a Supabase Realtime channel listens for `INSERT` events on the `vents` table:

```js
function subscribeRealtime() {
  db.channel('vents-live')
    .on('postgres_changes',
      { event: 'INSERT', schema: 'public', table: 'vents' },
      payload => {
        DATA.unshift(payload.new);   // prepend new record
        buildFeelingList();
        buildYearList();
        applyFilters();              // re-render grid
      }
    )
    .subscribe();
}
```

When a user submits via the VENT app, the new card appears in the portal within ~1 second — no page refresh needed.

### UI Features

| Feature | How it works |
|---|---|
| **Cards grid** | Responsive CSS grid (3 cols → 2 → 1 at breakpoints) |
| **Search** | `JSON.stringify(record).toLowerCase().includes(query)` — searches all fields |
| **Feeling filter** | Dropdown built dynamically from unique values in `DATA` |
| **Year filter** | Dropdown + quick-year pill buttons |
| **Date range** | From / To date pickers filter on `Created Date` |
| **Sort** | Newest first / Oldest first toggle |
| **Detail modal** | HTML `<dialog>` element; shows full vent text + metadata |
| **Stats modal** | Top 10 feelings by count with proportional bar chart |
| **Count chips** | Live "Showing X of Y · Z feelings" strip updates on every filter change |

---

## Back End — Supabase

### Database

**Project:** `pkgvgkleujecyvqvjojb.supabase.co`  
**Table:** `public.vents`

| Column | Type | Notes |
|---|---|---|
| `ID` | `text` | Primary key; `DEFAULT gen_random_uuid()::text` |
| `Created Date` | `timestamptz` | `DEFAULT now()` |
| `Updated Date` | `timestamptz` | `DEFAULT now()` |
| `I am felling` | `text` | Intentional typo — preserved from original dataset schema |
| `vb_Anonymous` | `text` | The vent body text |
| `Owner` | `text` | UUID of submitter (currently `null` — app is anonymous) |

Column names contain spaces and mixed case — they must always be quoted in SQL (`"I am felling"`) and passed as exact string keys in JavaScript.

### Row Level Security (RLS)

RLS is enabled on the table. Two policies are required:

```sql
-- The VENT app: anyone can insert
CREATE POLICY "anon insert"
  ON public.vents FOR INSERT TO anon WITH CHECK (true);

-- The portal: anyone (with anon key) can read
CREATE POLICY "anon select"
  ON public.vents FOR SELECT TO anon USING (true);
```

Without the SELECT policy, the portal cannot load data and falls back to sample records.

### Real-Time

Supabase Real-Time uses PostgreSQL's `LISTEN / NOTIFY` under the hood. The portal subscribes to the `postgres_changes` feed on `public.vents`. Supabase must have **Replication** enabled on the `vents` table for this to fire (enabled in the Supabase dashboard under Database → Replication).

### API Access

Both the app and portal use the **anon (public) key**. This key is safe to ship in client-side code because RLS policies restrict what it can actually do. The key cannot bypass RLS; only the service-role key (never in client code) can do that.

| Key type | Used in | Can do |
|---|---|---|
| `anon` | `index.html`, `portal.html` | INSERT + SELECT (per RLS policies) |
| `service_role` | Never client-side | Bypasses RLS — admin/import use only |

---

## Data Flow

### User Submitting a Vent

```
User fills out app
        │
        ▼
actions.submit() called in React
        │
        ▼
db.from('vents').insert({ ... })  ← Supabase JS v2 client
        │
        ▼
HTTPS POST → Supabase REST API
        │
        ▼
PostgreSQL INSERT into public.vents
        │
        ├──▶ HTTP 201 → app shows "thanks" screen
        │
        └──▶ postgres_changes event fires
                    │
                    ▼
             Portal WebSocket receives INSERT payload
                    │
                    ▼
             DATA.unshift(payload.new) → grid re-renders
```

### Portal Loading Data

```
Admin logs in
        │
        ▼
loadData() → db.from('vents').select('*').order('Created Date', desc)
        │
        ▼
HTTPS GET → Supabase REST API
        │
        ▼
All rows returned → DATA array populated
        │
        ▼
initializeApp() → builds filters, renders grid
        │
        ▼
subscribeRealtime() → WebSocket channel open
```

---

## Deployment

The repo is deployed via **GitHub Pages** from the `main` branch root.

| URL | File |
|---|---|
| `https://ejohnsonbda.github.io/Vent-Air/` | `index.html` — the VENT app |
| `https://ejohnsonbda.github.io/Vent-Air/portal.html` | Admin portal |

Every `git push origin main` automatically deploys within ~30 seconds. There is no CI pipeline or build step — GitHub Pages serves the files as-is.

---

## Environment Variables / Secrets

There are no server-side secrets. The Supabase anon key is embedded in both HTML files. This is the standard pattern for Supabase client apps — the key's permissions are constrained by RLS policies, not by keeping the key secret.

If the portal password needs to be changed, update `passcore.json` in the repo root:

```json
[
  { "username": "admin", "password": "yournewpassword" }
]
```

The hardcoded fallback `admin / 8804114` is always active regardless of `passcore.json`.

---

## Quick Reference — Adding a Feature

| Task | Where to edit |
|---|---|
| Add a new feeling | `FEELINGS` array in `index.html` |
| Add a new screen | Add a new component + entry in the `screens` object in `index.html` |
| Change the accent color | `THEME.accent` constant in `index.html`; `--accent` in `portal.html` |
| Change portal password | `passcore.json` in repo root |
| Modify the database schema | Supabase SQL Editor + update insert/select queries in both HTML files |
| Add a new column | Supabase SQL Editor `ALTER TABLE`, then update the `insert()` call in `index.html` and `renderGrid()` / `openDetail()` in `portal.html` |
