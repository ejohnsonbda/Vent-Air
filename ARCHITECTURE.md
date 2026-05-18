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

## Visual Design & Layout System

### Design Tokens

Every color, shadow, radius, and font is defined as a CSS custom property in `:root` and shared across both files. Changing `--accent` in one place recolors every button, badge, glow, and highlight.

```css
:root {
  /* Brand */
  --accent:       #F47B20;          /* VENT orange */
  --accent-soft:  rgba(244,123,32,.14);
  --accent-line:  rgba(244,123,32,.45);
  --accent-glow:  rgba(244,123,32,.22);

  /* Backgrounds — layered dark surfaces */
  --bg:           #0B0B0B;          /* page background */
  --surface-1:    #141414;          /* lowest card level */
  --surface-2:    #1a1a1a;          /* main card */
  --surface-3:    #1f1f1f;          /* hover state */
  --surface-4:    #262626;          /* deepest element */

  /* Text */
  --text:          rgba(255,255,255,0.94);
  --text-secondary:rgba(255,255,255,0.6);
  --text-muted:    rgba(255,255,255,0.42);
  --text-faint:    rgba(255,255,255,0.28);

  /* Borders */
  --line:          rgba(255,255,255,0.08);
  --line-strong:   rgba(255,255,255,0.16);
  --line-hover:    rgba(244,123,32,0.4);

  /* Shape */
  --radius-xs: 2px;   --radius-sm: 4px;
  --radius-md: 6px;   --radius-lg: 8px;

  /* Elevation */
  --shadow-sm:  0 1px 3px rgba(0,0,0,.4);
  --shadow-md:  0 8px 24px rgba(0,0,0,.5);
  --shadow-lg:  0 24px 60px rgba(0,0,0,.55);

  /* Motion */
  --t: 180ms cubic-bezier(.2,.7,.3,1);

  /* Typography */
  --font-display: "Anton", "Oswald", system-ui, sans-serif;
  --font-body:    "Manrope", system-ui, sans-serif;
}
```

### Typography

| Role | Font | Weight | Usage |
|---|---|---|---|
| Display / Wordmark | Anton (Google Fonts) | 400 | Screen titles, the VENT logo, stat numbers |
| Body | Manrope (Google Fonts) | 400–800 | All prose, labels, buttons, filters |
| Eyebrow labels | Manrope | 700 | Uppercase spaced-out section labels |

Anton is a condensed display face that gives VENT its editorial, intense personality. Manrope provides legibility across a wide weight range for UI chrome.

### VENT App Layout (index.html)

```
┌─────────────────────────────────────┐
│              viewport               │  position: fixed; inset: 0
│                                     │
│   ┌─────────────────────────────┐   │
│   │                             │   │
│   │         .app-shell          │   │  Mobile: width 100%, height 100%
│   │                             │   │  Tablet: 420px × 840px centered
│   │    ┌─────────────────┐      │   │  Laptop: 420px × 900px centered
│   │    │  <CurrentScreen>│      │   │
│   │    └─────────────────┘      │   │
│   │                             │   │
│   └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

Each screen occupies the full `.app-shell` height and handles its own internal scroll. Most screens have three zones:

```
┌────────────────────────┐
│   Top bar / wordmark   │  ~64–80px — logo, back button
├────────────────────────┤
│                        │
│      Scroll body       │  flex: 1; overflow-y: auto
│                        │
├────────────────────────┤
│    Action / button     │  fixed at bottom of shell, above home indicator
└────────────────────────┘
```

Safe-area insets are applied so content is never hidden behind the iOS notch or Android status bar:

```css
padding-top:    env(safe-area-inset-top);
padding-bottom: env(safe-area-inset-bottom);
```

### Screen-by-Screen Layout Details

| Screen | Layout pattern | Key elements |
|---|---|---|
| **Welcome** | Full-bleed centered hero | Large "VENT" wordmark, tagline, single CTA button |
| **Emotion** | Sticky category tabs + scrollable grid | 84 feeling chips in a 3-column grid; list/grid toggle |
| **Vent** | Flex column | Feeling recap chip, growing textarea (min 120px), char count |
| **Review** | Card with two sections | Emotion badge on top, vent text body, two action buttons |
| **Submitting** | Centered full-screen | Spinner animation + "Sending…" label; error state shows message + retry |
| **Thanks** | Centered celebration | Large checkmark, gratitude copy, auto-advance timer |
| **Breathe** | Animated circle | Expanding/contracting ring with 4-7-8 inhale/hold/exhale phases |
| **Support** | Scrollable link list | Resource cards with name, description, and phone/web link |
| **Privacy** | Scrollable prose | Policy text with section headings |

### Admin Portal Layout (portal.html)

The portal is a traditional top-to-bottom document layout (no fixed-height shell):

```
┌──────────────────────────────────────────────────────┐
│  HEADER (sticky, blur backdrop)                      │
│  Logo · "Console v2.4" tag · Stats · Logout          │
├──────────────────────────────────────────────────────┤
│  HERO SECTION                                        │
│  Live badge · Title · Description · Warning banner   │
│  ┌──────────────────────────┐                        │
│  │  Search input            │                        │
│  └──────────────────────────┘                        │
├──────────────────────────────────────────────────────┤
│  FILTERS BAR (surface-1 background)                  │
│  Feeling ▾  Year ▾  From [date]  To [date]  Sort ▾  Reset │
│  Quick year pills: 2023 · 2024 · 2025 · 2026 · All  │
│  Chips: Showing 42 · Total 4,656 · Feelings 84      │
├──────────────────────────────────────────────────────┤
│  CARDS GRID  (max-width 1280px, 32px padding)        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  Card    │  │  Card    │  │  Card    │  3 cols    │
│  │ Feeling  │  │ Feeling  │  │ Feeling  │  desktop   │
│  │ Created  │  │ Created  │  │ Created  │            │
│  │ Snippet  │  │ Snippet  │  │ Snippet  │            │
│  └──────────┘  └──────────┘  └──────────┘           │
│   (→ 2 cols at 1024px, → 1 col at 640px)            │
├──────────────────────────────────────────────────────┤
│  FOOTER BAR (fixed, blur backdrop)                   │
│  "Click any card to open details"  ·  Dashboard →   │
└──────────────────────────────────────────────────────┘
```

**Card anatomy:**

```
┌─────────────────────────────────────┐
│ [● ANXIETY]            [ID: abc123] │  card-hdr: feeling badge + ID tag
├─────────────────────────────────────┤
│ Created  May 17, 2025  Owner  76f…  │  card-meta: dark tinted row
├─────────────────────────────────────┤
│ Heart pounding for no clear reason. │
│ Again. Trying to remember that it   │  card-body: 4-line clamped snippet
│ always passes but right now it      │
│ feels permanent.                    │
└─────────────────────────────────────┘
```

### Responsive Breakpoints

| Breakpoint | App behavior | Portal behavior |
|---|---|---|
| < 400px | Full-screen shell | Filters stack to 1 column |
| 400–600px | Full-screen shell | Filters stack to 2 columns |
| 600–640px | Full-screen shell | Filters at 3 columns |
| 640–1024px | 420×840px centered shell | Cards at 2 columns |
| ≥ 1024px | 420×900px centered shell | Cards at 3 columns |

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

> **Note:** This is a client-side credential check only. It prevents casual access but is not cryptographically secure. The anon key in the source can be seen by anyone who views page source.

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

If Supabase is unreachable, it falls back to six built-in sample records so the UI is never blank.

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

Both the app and portal use the **anon (public) key**. This key is safe to ship in client-side code because RLS policies restrict what it can actually do.

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

## Launching Both Sites from GitHub

### How GitHub Pages Works

GitHub Pages reads the `main` branch of the repository and serves every file as a static website. No server configuration, no deployment pipeline — just push a file and it's live within ~30 seconds.

The VENT repo (`ejohnsonbda/Vent-Air`) serves two URLs from the same codebase:

| Site | URL | File |
|---|---|---|
| VENT App (user-facing) | `https://ejohnsonbda.github.io/Vent-Air/` | `index.html` |
| Admin Portal | `https://ejohnsonbda.github.io/Vent-Air/portal.html` | `portal.html` |

### Enabling GitHub Pages (First Time)

If GitHub Pages is not yet enabled on the repo:

1. Go to `github.com/ejohnsonbda/Vent-Air`
2. Click **Settings** (top tab)
3. Scroll to **Pages** in the left sidebar
4. Under **Source**, select **Deploy from a branch**
5. Branch: **main** · Folder: **/ (root)**
6. Click **Save**

GitHub will show a green banner with the live URL within a minute.

### Publishing an Update

Every change to the codebase is deployed by pushing to `main`:

```bash
# Edit a file, then:
git add index.html          # or portal.html, manifest.json, etc.
git commit -m "Describe the change"
git push origin main
```

GitHub Pages detects the push automatically and redeploys. No manual deploy step is needed.

### Sharing Each Site

- **VENT app** — share `https://ejohnsonbda.github.io/Vent-Air/` with users. On mobile, they can tap "Add to Home Screen" to install it as a PWA icon on their phone.
- **Admin portal** — share `https://ejohnsonbda.github.io/Vent-Air/portal.html` only with authorized staff. The login screen protects casual access.

### Custom Domain (Optional)

To use a custom domain (e.g., `ventapp.com`) instead of the `github.io` URL:

1. Buy the domain from any registrar (Namecheap, Cloudflare, etc.)
2. In the registrar's DNS settings, add a CNAME record: `www → ejohnsonbda.github.io`
3. In GitHub → Settings → Pages → Custom domain, enter `www.ventapp.com`
4. Check **Enforce HTTPS**
5. Create a file named `CNAME` in the repo root containing just `www.ventapp.com`

The PWA manifest, Supabase API calls, and all links will continue to work because GitHub Pages rewrites all traffic through the custom domain.

---

## Publishing to App Stores

VENT is built as a PWA, which means it can be submitted to both the Google Play Store and the Apple App Store without rewriting the app in Swift or Kotlin. The approach packages the existing web app in a thin native wrapper.

### Prerequisites (Both Stores)

Before submitting to either store, the following must be in place:

| Requirement | Status | Notes |
|---|---|---|
| HTTPS hosting | ✅ Done | GitHub Pages provides free HTTPS |
| `manifest.json` | ✅ Done | Already configured |
| App icons (192px, 512px) | ✅ Done | In `/icons/` folder |
| Service Worker | ⚠️ Not yet | Required for Play Store TWA; strongly recommended for App Store |
| Privacy Policy URL | ⚠️ Needed | Both stores require a public privacy policy page |

**Adding a service worker** (needed before store submission):

Create `sw.js` in the repo root:

```js
const CACHE = 'vent-v1';
const ASSETS = ['/', '/index.html', '/manifest.json'];

self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(ASSETS)));
});
self.addEventListener('fetch', e => {
  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));
});
```

Register it in `index.html` just before `</body>`:

```html
<script>
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/Vent-Air/sw.js');
  }
</script>
```

---

### Google Play Store — Trusted Web Activity (TWA)

A **Trusted Web Activity** lets a PWA run in Chrome with no browser UI, indistinguishable from a native app. It is the official Google-supported method for publishing PWAs on Play.

#### Option A — PWABuilder (Easiest, No Code)

[PWABuilder](https://www.pwabuilder.com) is a free Microsoft tool that generates the Android project for you.

1. Go to **pwabuilder.com** and enter `https://ejohnsonbda.github.io/Vent-Air/`
2. PWABuilder scans your manifest and service worker and shows a score
3. Click **Package for Stores** → **Google Play**
4. Fill in:
   - Package name: `com.ventapp.vent` (or similar)
   - App version: `1`
   - Signing key: generate a new one and **save the keystore file** — you need it for every future update
5. Download the generated `.aab` (Android App Bundle) file
6. Go to **play.google.com/console**, create a new app
7. Upload the `.aab` under **Production → Create new release**
8. Fill in store listing (description, screenshots, content rating)
9. Submit for review (~3 business days for new apps)

#### Option B — Bubblewrap CLI (More Control)

```bash
npm install -g @bubblewrap/cli
bubblewrap init --manifest https://ejohnsonbda.github.io/Vent-Air/manifest.json
bubblewrap build
```

This generates a signed `.aab` ready for Play Console.

#### Digital Asset Links (Required for TWA)

The Play Store TWA verifies ownership of the domain. You must host a file at:

```
https://ejohnsonbda.github.io/Vent-Air/.well-known/assetlinks.json
```

PWABuilder generates this file for you. Add it to the repo at `.well-known/assetlinks.json`:

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.ventapp.vent",
    "sha256_cert_fingerprints": ["YOUR_SIGNING_KEY_FINGERPRINT"]
  }
}]
```

---

### Apple App Store

Apple does not support TWA. The options for getting VENT into the App Store are:

#### Option A — PWABuilder (Easiest)

PWABuilder also generates an iOS/macOS package using a `WKWebView` wrapper:

1. Go to **pwabuilder.com** → enter the VENT URL → **Package for Stores** → **iOS**
2. Fill in App Name, Bundle ID (`com.ventapp.vent`), version
3. Download the generated Xcode project (`.zip`)
4. Open in **Xcode** (requires a Mac)
5. Set your **Apple Developer Team** in the project settings
6. Archive the app: **Product → Archive**
7. Upload to App Store Connect via **Organizer → Distribute App**
8. In App Store Connect, fill in:
   - App description, keywords, screenshots (required sizes: 6.5", 5.5", iPad 12.9")
   - Privacy policy URL (mandatory)
   - Age rating
9. Submit for review (~1–3 business days)

#### Option B — Capacitor (Recommended for Future Native Features)

[Capacitor](https://capacitorjs.com) wraps the web app but also gives access to native iOS APIs (camera, push notifications, haptics, etc.) if needed later.

```bash
npm install @capacitor/core @capacitor/cli @capacitor/ios
npx cap init VENT com.ventapp.vent
npx cap add ios
npx cap copy
npx cap open ios   # Opens Xcode
```

Then archive and submit from Xcode as above.

#### App Store Requirements Checklist

| Requirement | Notes |
|---|---|
| Apple Developer Account | $99/year at developer.apple.com |
| Mac with Xcode | Required for building and uploading iOS apps |
| App icons (all sizes) | Xcode accepts a 1024×1024 PNG and generates all sizes |
| Privacy policy URL | Must be a live public web page |
| Screenshots | At minimum: one 6.5-inch (iPhone 14 Pro Max) and one 5.5-inch |
| Content rating | VENT should rate 12+ (emotional content) |
| App description | Explain the anonymous venting concept clearly |

---

### Store Listing Copy (Suggested)

**Name:** VENT

**Short description (Play Store, 80 chars):**
`A safe, anonymous space to set it down. No account. No trace.`

**Full description:**
```
Sometimes you just need to get it out.

VENT is a private, anonymous space to express whatever you're feeling — 
without judgment, without an account, and without a trace.

Pick how you're feeling from 84 emotions, write freely, and let it go. 
Your words are stored anonymously with no connection to your identity.

After you vent, VENT guides you through an optional breathing exercise 
and connects you with mental health resources if you need them.

• 100% anonymous — no account, no email, no name
• 84 emotions across 8 categories  
• 4-7-8 guided breathing after each vent
• Mental health resource links
• Works offline after first load
```

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
| Update the live site | `git add` → `git commit` → `git push origin main` |
| Add service worker | Create `sw.js` in repo root, register in `index.html` |
