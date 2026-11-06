# Summaries Inbox — Build Spec for Claude Code

> Version: 1.1 · Updated: 2026-11-06 · Owner: <השם-שלך>
> Phase 1 (features) and phase 2 (design) are **done and approved**. This document is the handoff for phase 3: **build**.
> The document is in English. **The app UI is Hebrew, right-to-left (RTL).** the owner writes to you in Hebrew; answer him in Hebrew.

---

## 0. READ THIS FIRST — how you (Claude Code) must work

### 0.1 Before you touch anything
1. Read this whole document, top to bottom.
2. Read the n8n build rules: the owner's Obsidian note on n8n build rules (`make-an-n8n`), or `make-an-n8n.md` in the repo `itoosh-45/Make-an-automation-`, branch `claude/beautiful-brown-pccd7w`. Summary in §9.
3. Read the owner's Obsidian note about the n8n server (access, management, security).
4. Look at the approved mockups (§6) before writing any UI.
5. **Ask the owner your questions before executing anything.** Start with the list in §11 ("Questions to confirm before starting"), plus anything in this document that is unclear to you. Ask them all in one message, in Hebrew, short and numbered. Do not start milestone M0 until he answers.

### 0.2 How to build
- Build **one milestone at a time** (§10), in order. Never start the next milestone before the owner approves the current one.
- Each milestone ends with **two kinds of tests**:
  1. **Your tests:** run them yourself (queries, curl, n8n test executions, browser checks). Report what you ran and the result.
  2. **The owner's tests:** a short, numbered checklist in Hebrew of what he should do on his phone or computer, and what he should see. Then **stop and wait** for his "עובד" / "לא עובד".
- If a test fails: fix, re-test, report. Don't move on with a known failure.
- Before every change that touches something **live** (an active workflow, the server, the database, DNS), say in one line what you're about to change and how to undo it. For anything destructive (deleting data, dropping tables, removing workflows, changing DNS), ask first and wait.
- After each milestone: tick the checkboxes in §10, add a line to the log (§14), and update decisions (§12) if anything changed. Keep this file as the single source of truth.

### 0.3 Hard rules
- **Everything stays free.** No paid services, no paid API credits. If something would cost money, stop and ask.
- **No personal data in this repo** (it may be public): no emails, names, IPs, server/instance names, real domains, n8n IDs, vault names/paths, personal subscriptions. Use placeholders: `<השם-שלך>` `<המייל-שלך>` `<כתובת-השרת>` `<שם-השרת>` `<n8n-domain>` `<app-domain>` `<שם-ה-vault>` `<newsletter-1>` `<wf:…>` (workflow, look up by name in n8n) `<cred:…>` (credential, look up by name). Real values live in the cloud environment variables, in n8n itself, and in the owner's private notes. Before every push: confirm the repo is private and scan the diff for these values.
- **Never write secrets** (passwords, tokens, API keys, `N8N_ENCRYPTION_KEY`, SSH keys) into this file, the repo, Obsidian, or chat. Write only *where* a secret lives. Secrets go in n8n Credentials, the server `.env`, or the owner's password manager.
- **Gemini free tier ≈ 20 calls/day on the primary model.** Don't burn it on tests. Use pinned data (`prepare_workflow_pin_data`) or a single short episode. Never run the full podcast workflow manually more than once a day.
- **Emails keep being sent in parallel** until the owner explicitly switches them off (M7).
- **n8n 2.x: saving is not publishing.** After editing an active workflow, publish it, or the old version keeps running.
- Talk to the owner in plain, simple Hebrew. He has some HTML/CSS/C#/Python background and wants to learn, so explain *why*, briefly, when it helps.

---

## 1. Goal
One place on the iPhone where all summaries, news digests and newsletters arrive like an inbox, instead of being scattered across Gmail. Read, search, star, annotate, manage sources, delete.

**Guiding principle:** completely free, running on the owner's own server and n8n.

---

## 2. Current state (as of 2026-11-06)

### 2.1 Server
| What | Value |
|---|---|
| Server | Oracle Cloud Free Tier · Ubuntu 24.04 ARM64 · 2 OCPU / 12GB · instance `<שם-השרת>` |
| Public IP | **`<כתובת-השרת>`** (current, per the owner). Obsidian still lists the old `<כתובת-ישנה>` → must update DuckDNS, `C:\Users\<user>\.ssh\config` (`HostName`) and the Obsidian server note (M0) |
| SSH | `ssh n8n` from the owner's Windows PC (PowerShell). User `ubuntu`. Key at `C:\Users\<user>\.ssh\oracle-n8n.key` |
| n8n | https://<n8n-domain> · v2.39.x pinned in `~/docker/n8n/.env` · folder `~/docker/n8n/` |
| Containers | `n8n` · `n8n-postgres` · `npm` (Nginx Proxy Manager, admin on :81 via SSH tunnel → `localhost:8181`) · `portainer` (:9443 via tunnel) · `obsidian-couchdb` |
| Open ports | 22, 80, 443 only |
| Config changes | Add env vars to `~/docker/n8n/docker-compose.override.yml`, then `docker compose up -d` (not `restart`). Never edit the main compose file |
| AI | Gemini free tier, credential **`Gemini API`** (`<cred:gemini>`, httpHeaderAuth) |
| Email | credential **`SMTP account`** (`<cred:smtp>`), sends to `<המייל-שלך>` |
| Error workflow | `<wf:error>` ("התראת כשל") |
| Execute Command | Allowed and **required** (ffmpeg, base64, cleanup). Never add `NODES_EXCLUDE` for it |
| Claude Code (cloud) | Environment **`summaries-inbox`**: HTTPS only, **no SSH to the server** → the owner runs server commands. Allowed hosts: `<n8n-domain>`, `<app-domain>`, `itunes.apple.com`, the 4 podcast feed hosts (remove after M1). Local Postgres 16 for testing `sql/` before it touches the server. Vars: `TZ`, `INBOX_API_URL`, `INBOX_APP_ORIGIN`. No secrets saved yet (API token only after the M2 no-token tests) |

### 2.2 Workflows
| ID | Name | State | In scope | What it does today |
|---|---|---|---|---|
| `<wf:podcasts>` | סיכום פודקאסטים שבועי | 🟢 | ✅ | Fri 04:00 (Asia/Jerusalem). 5 RSS feeds **hardcoded** in node "רשימת פידים" → episodes from the last 8 days → dedupe (2000 GUIDs) → max 4 per run → ffmpeg 30-min chunks → Gemini transcription (+fallback) → Gemini analysis with `responseSchema` (+fallback) → "בניית HTML" → email with HTML attachment → "ניקוי" → next episode |
| `<wf:on-demand-page>` | פודקאסט לפי קישור — עמוד ובקשות | 🟢 | ✅ | HTML page + POST/GET webhooks; Spotify episode link → queue on server via `node /home/node/.n8n-files/podcast-on-demand/runtime.cjs` |
| `<wf:on-demand-worker>` | פודקאסט לפי קישור — תמלול וסיכום | 🟢 | ✅ | Worker for the queue. Models configured in `podcast-on-demand/config.json` on the server. **Inspect before changing.** |
| `<wf:ai-news>` | ai news | 🟢 | ✅ | Weekly. 9 feeds + <newsletter-1> → Gemini picks 8–12 items, Hebrew summary → email. **Inspect before changing** |
| `<wf:world-news>` | סיכום שבועי — ישראל ויחסי חוץ | 🟢 | ✅ | Daily RSS collection; Sun 02:00 weekly Hebrew digest (two-stage Gemini + fallback) → RTL email. **Inspect before changing** |
| `<wf:tts>` | פודקאסט באנגלית — תרגום והקראה בעברית | 🟢 | ❌ | Not part of this project. Don't touch |
| `<wf:tldr>` | Tdlr newsletter עברית | ⚪ | ❌ | Off. Don't touch |
| `<wf:error>` | התראת כשל | 🟢 | — | Shared error workflow |
| — | <newsletter-1> | external | ✅ | Has an RSS feed (RSS icon in the email footer; the same feed is already one of the inputs of `ai news`) |

Old experiments (My workflow 1–4, `סיכום פודקאסטים שבועי copy 2`, `TEMP — בדיקת קבצי פודקאסט`, `YouTube download - probe`, `Ai Instagram Infulancer`): leave alone unless the owner asks to delete.

### 2.3 Data the podcast workflow already produces
Before it builds the email, node "איחוי תמלול" + "ניתוח Gemini" produce:
- Episode meta (from "הכנת ffmpeg"): `podcast`, `title`, `guid`, `audioUrl`, `link`, `duration`, `pubIso`, `pubHe`
- `transcript`: full text, segments formatted `[MM:SS] text` (or `[H:MM:SS]`), separated by blank lines, timestamps already shifted to absolute time
- Analysis JSON: `summary` (string), `highlights[{time, point}]`, `quotes[{time, speaker, text}]`, `glossary[{term, definition}]`

So the app never needs to read email. Each workflow writes straight into the database.

---

## 3. Architecture (approved)

```
 Podcast RSS ──┐
 Spotify link ─┤                                   ┌─► Email (temporary, switchable off in Settings)
 News RSS ─────┼─► n8n workflows (transcribe/sum.) ┤
 Newsletter ───┘   (newsletters: no AI)            └─► Postgres DB "inbox" (on the server)
   RSS / KTN                                                  ▲
                                                              │ SQL
                                     n8n workflow "אינבוקס — API" (one webhook, token-protected)
                                                              ▲
                                                              │ HTTPS + header token
                                     PWA "אינבוקס" at https://<app-domain> (static files, nginx container)
```

| Part | Decision |
|---|---|
| Database | **Existing `n8n-postgres` container, new separate database `inbox` with its own user `inbox_app`.** n8n's own DB stays untouched |
| API | One n8n workflow, one POST webhook `inbox-api`, action router (Switch). Auth: Header Auth with a long random token. CORS: only the app's origin |
| App | **PWA** (installed to the iPhone home screen). Plain HTML + CSS + vanilla JS modules, **no build step**, so the owner can read and edit it |
| Hosting | **On the owner's server**: a tiny `nginx:alpine` container serving the PWA folder, exposed through Nginx Proxy Manager at a **new DuckDNS subdomain `<app-subdomain>`** with Let's Encrypt SSL |
| Code | New GitHub repo **`itoosh-45/summaries-inbox`** (confirm name with the owner): `app/` (PWA), `sql/` (schema + seeds), `n8n/` (exported workflow JSON for backup), `docs/SPEC.md` (this file), `CLAUDE.md` pointing here |
| Newsletters | Own RSS when available (<newsletter-1>: yes). Otherwise a free [Kill the Newsletter](https://kill-the-newsletter.com) address per newsletter → Atom feed. No Gmail, no Google |
| Podcast search | iTunes Search API (free, no key), called **from the app** |
| Obsidian export | **No server work.** The app builds Markdown and opens `obsidian://new?vault=<שם-ה-vault>&file=00-Inbox/<name>&content=<md>` on the iPhone, with a "copy Markdown" fallback |
| Timestamps | `open.spotify.com/episode/<ID>?t=<seconds>` when the Spotify episode ID is known; otherwise an in-app audio player on `audio_url`, seeking to that second |

**Why PWA, not native:** native iOS needs a Mac + Xcode; with a free Apple account the app expires every 7 days, and the fix costs $99/year. A PWA is free, updates without reinstalling, supports Home Screen install and (iOS 16.4+) push. It can be wrapped later with Capacitor if the owner ever wants the App Store.

---

## 4. Features (MVP — all approved)

### 4.1 Inbox (main screen)
- Gmail-on-iPhone structure: pill search bar at top (menu button on the right, avatar on the left → Settings), filter chips, list of rows, floating "הוסף מקור" button, bottom nav (אינבוקס · מחברת · מקורות).
- Filter chips: הכל · פודקאסטים · חדשות · ניוזלטרים · מסומנים.
- Row: round logo of the source · source name (bold if unread) · time · title · one-line snippet · a type pill (e.g. "פודקאסט · 52 דק׳") · "N הערות שלי" pill if notes exist · unread dot · star (tap toggles).
- Sorted newest first. Pull-to-refresh. Infinite scroll (cursor pagination, 30 per page).
- Swipe left on a row → delete (with undo toast for 5 s). Opening an item marks it read.

### 4.2 Search (top bar)
- Searches titles, summaries, transcripts and my notes. Scope chips: הכל · כותרות וסיכומים · תמלולים · ההערות שלי.
- Results grouped by scope, match highlighted, transcript hits show the timestamp and open the transcript at that point.
- Server-side, Postgres `ILIKE` with `pg_trgm` indexes (works for Hebrew; no Hebrew full-text dictionary needed). Debounce 300 ms, minimum 2 characters.

### 4.3 Side menu = one folder per source
- Sections: כל הסיכומים (unread count) · מסומנים בכוכב · מחברת · **פודקאסטים** (one folder each) · **חדשות** (AI news, Israel & world) · **ניוזלטרים** (one folder each) · ניהול מקורות · הגדרות.
- Source folder page: big logo, name, "פודקאסט · N פרקים · נבדק כל שישי", buttons "עריכה ולוגו" and "השהיה / המשך מעקב", list of that source's items.
- Empty folder of a newly added podcast shows: "הפרקים הראשונים יגיעו ביום שישי".

### 4.4 Podcast episode screen
- Top bar: back · star · export to Obsidian · delete.
- Header: source logo + name (tap → folder) · date · length · title · buttons "פתח ב-Spotify" and "נגן כאן".
- Tabs: סיכום · נקודות · ציטוטים · מונחים · תמלול.
  - סיכום: summary paragraph + my general notes (yellow card).
  - נקודות: list, each with a timestamp pill (tap → Spotify / player).
  - ציטוטים: quote, speaker, timestamp pill.
  - מונחים: term + definition.
- Bottom composer: "הוסף הערה על הפרק…" → saves a general note.

### 4.5 Transcript (approved format, important)
- **One continuous block of running text**, verbatim, in the original order. **No paragraph breaks, no speaker cards or labels, no reordering, no rewording.**
- Timestamps sit in a **narrow gutter on the right side** (RTL start), aligned with the line where each segment begins. Implementation hint (used in the mockup): the text container is `position: relative` with right padding; each segment's timestamp is an absolutely-positioned element with only `right` set and `top` left `auto`, so it stays on the line where its segment starts.
- Parsing: split the stored `transcript` on `[MM:SS]` / `[H:MM:SS]` markers into segments `{t_seconds, label, text}`. Strip speaker prefixes like "מנחה:" / "אורח:" / "דובר 2:" for display only (keep the raw transcript in the DB). `[לא ברור]` stays as is.
- Tap a segment → it highlights inline (`#FDE68A`) → bottom sheet: textarea for a note · הדגשה · ב-Spotify (at that time) · העתקה · שמירה.
- Segments with a note: light yellow background (`#FEF3C7`) + small dot next to their timestamp.
- Search inside the transcript (icon in the header).

### 4.6 Notes, highlights, Notebook
- Note types: `note` (general, on the item), `highlight` (a transcript segment, optional text), `quote` (saved quote from the quotes tab).
- Notebook screen: all of them across all items, grouped by item, filter chips הכל · הערות · הדגשות · ציטוטים, timestamp pills tappable, button "ייצוא לאובסידיאן".
- Starred items and items that have notes are **never auto-deleted**.

### 4.7 Export to Obsidian
- Per item (episode screen) and per item group (Notebook).
- Markdown: frontmatter (`type: summary`, `source`, `date`, `link`, `tags: [summary, <type>]`), title, summary, highlights with timestamps, quotes, my notes. **Not** the full transcript (URL length).
- Open `obsidian://new?vault=<שם-ה-vault>&file=00-Inbox/<safe-title>&content=<urlencoded md>`. If the URL is over ~6,000 characters, or if opening fails, copy the Markdown to the clipboard and show "הועתק, הדבק באובסידיאן".
- Folder `00-Inbox/` shown and editable in Settings.

### 4.8 Timestamps → Spotify, with in-app player fallback
- If `spotify_episode_id` is known: `https://open.spotify.com/episode/<ID>?t=<seconds>` (opens the Spotify app on iOS).
- Otherwise: in-app mini player (HTML `<audio>` on `audio_url`), set `currentTime = seconds`, play. Player bar sticks to the bottom while playing.
- Episodes submitted by Spotify link already have the ID. For RSS episodes, Spotify matching is an **open item** (§11): only try the Spotify Web API if it works for free; never block on it.

### 4.9 Sources management
- Tabs: פודקאסטים · ניוזלטרים · חדשות.
- **Add podcast flow:**
  1. Search field → app calls `https://itunes.apple.com/search?media=podcast&entity=podcast&country=IL&limit=10&term=<q>` directly (CORS is open on that API; if not, proxy it through the API action `itunes_search`).
  2. Results: artwork, name, author, "הוספה" button.
  3. Tap → API `source_create` with `type=podcast`, `name`, `feed_url`, `artwork_url`, `itunes_id`, logo default `{kind:'artwork'}`.
  4. Folder appears immediately. **No immediate processing**: it waits for the next Friday run.
- Paste RSS manually as an alternative ("הדבקת RSS").
- Pause/resume toggle per source (`active`). Remove source (asks to confirm; deletes the source, its items and notes).
- Dedupe history in the workflow prevents re-processing if a podcast is removed and re-added.
- Spotify-only podcasts (no RSS) won't appear in search; single episodes still work through "פודקאסט לפי קישור".
- **Add newsletter:** "הדבקת RSS" (preferred) or "כתובת מייל חדשה" → instructions screen: open kill-the-newsletter.com, create a feed with the newsletter's name, paste the feed URL back, subscribe to the newsletter with the address it shows. Confirmation email arrives as a normal item; open its link from the app.

### 4.10 Source edit (logo, name)
- Live preview (big logo + how the row looks in the inbox).
- Display name.
- Logo type: אותיות וצבע (up to 2 letters + 8-color palette) · תמונת הפודקאסט (`artwork_url`) · תמונה מהטלפון (file picker, **resized in the browser to 256×256 and stored as a data URL in the DB**, no file hosting).
- Shape: עיגול · ריבוע מעוגל.
- Spotify show link (optional, helps timestamp matching).
- RSS feed (read-only), and "הסרת המקור והתיקייה" (red, confirm).

### 4.11 Settings
- Auto-delete per type: פודקאסטים · חדשות AI · ישראל ויחסי חוץ · ניוזלטרים. Options: שבוע · שבועיים · חודש · 3 חודשים · לעולם לא. Defaults: 30 / 14 / 14 / 30 days. Note: starred or annotated items are never deleted.
- Obsidian folder (default `00-Inbox/`).
- "לשלוח גם במייל" toggle (default on). Workflows read it before sending email.
- API token field (entered once, saved in `localStorage`), and server status ("מחובר · עודכן לפני X").

### 4.12 Not in MVP
- Later: submit episode by link from inside the app · push notifications · offline reading.
- Rejected: on-demand processing of older episodes · weekly unread reminder · reading-time estimate · the Hebrew TTS workflow.

---

## 5. Data model (Postgres, database `inbox`)

Create with `sql/001_schema.sql`. Run as the DB superuser from the server.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE sources (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  type             text NOT NULL CHECK (type IN ('podcast','ai_news','world_news','newsletter','on_demand')),
  name             text NOT NULL,
  feed_url         text UNIQUE,              -- null for digests and on_demand
  artwork_url      text,
  itunes_id        text,
  spotify_show_url text,
  logo             jsonb NOT NULL DEFAULT '{"kind":"letters","letters":"","color":"#0C6F68","shape":"circle"}',
                   -- {kind: letters|artwork|upload, letters, color, shape: circle|square, image (data URL, upload only)}
  active           boolean NOT NULL DEFAULT true,
  sort_order       int NOT NULL DEFAULT 0,
  created_at       timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE items (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  source_id          uuid NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
  guid               text NOT NULL,           -- RSS guid / Spotify episode id / digest week key
  type               text NOT NULL,           -- copied from source.type for fast filtering
  title              text NOT NULL,
  published_at       timestamptz,
  received_at        timestamptz NOT NULL DEFAULT now(),
  summary            text,                    -- one paragraph, also used as inbox snippet
  content            jsonb NOT NULL DEFAULT '{}',
  original_url       text,
  audio_url          text,
  spotify_episode_id text,
  duration_sec       int,
  is_read            boolean NOT NULL DEFAULT false,
  is_starred         boolean NOT NULL DEFAULT false,
  UNIQUE (source_id, guid)
);
-- content by type:
--  podcast/on_demand: {highlights:[{time,point}], quotes:[{time,speaker,text}], glossary:[{term,definition}], transcript:"[00:00] ...", podcast_name}
--  ai_news/world_news: {intro, items:[{title, summary, source, url}]}   (match the workflow's existing JSON; inspect first)
--  newsletter: {html (sanitized), text}

CREATE TABLE notes (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  item_id     uuid NOT NULL REFERENCES items(id) ON DELETE CASCADE,
  kind        text NOT NULL CHECK (kind IN ('note','highlight','quote')),
  anchor_sec  int,          -- transcript/quote time in seconds, null = general note
  anchor_text text,         -- the segment or quote text it refers to
  text        text,         -- my words (may be empty for a plain highlight)
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE settings (
  key   text PRIMARY KEY,
  value jsonb NOT NULL
);

CREATE INDEX items_inbox_idx   ON items (received_at DESC);
CREATE INDEX items_source_idx  ON items (source_id, received_at DESC);
CREATE INDEX items_title_trgm  ON items USING gin (title gin_trgm_ops);
CREATE INDEX items_sum_trgm    ON items USING gin (summary gin_trgm_ops);
CREATE INDEX items_tr_trgm     ON items USING gin ((content->>'transcript') gin_trgm_ops);
CREATE INDEX notes_item_idx    ON notes (item_id);
CREATE INDEX notes_text_trgm   ON notes USING gin (text gin_trgm_ops);
```

Seed with `sql/002_seed.sql`:
- `settings`: `retention_days` = `{"podcast":30,"ai_news":14,"world_news":14,"newsletter":30}` (null = never) · `send_email` = `true` · `obsidian_folder` = `"00-Inbox/"`.
- `sources`:
  - The 5 podcast feeds currently hardcoded in "רשימת פידים" of `<wf:podcasts>`:
    *(The 5 feed URLs are not written here, for privacy. Read them from node "רשימת פידים" of `<wf:podcasts>` at build time.)*
    Fetch each feed once to fill `name` and `artwork_url` (`<itunes:image href>`), logo `{kind:'artwork'}`.
  - `ai_news` "חדשות AI" (letters `AI`, `#6D28D9`) · `world_news` "ישראל ויחסי חוץ" (letters `יח`, `#475569`) · `on_demand` "פרקים לפי קישור" (letters `ק`, `#0369A1`).
  - `newsletter` "<newsletter-1>" (2 letters from its name, `#1E3A8A`), `feed_url` = its RSS URL. **Copy it from the feed list inside `ai news` (`<wf:ai-news>`)**, or ask the owner to tap the RSS icon in the newsletter email.
- <newsletter-1> **stays** inside the `ai news` digest **and** is its own newsletter source (the owner's decision).

DB access:
- User `inbox_app` with rights only on database `inbox`. Password generated on the server, stored in the owner's password manager, and entered by the owner into a new n8n **Postgres** credential named **`Inbox DB`** (host `n8n-postgres`, port 5432, db `inbox`). This is one of the two new credentials this project needs; everything else reuses existing ones.
- Add the `inbox` database to the server backup script (`~/backup-n8n.sh`, `pg_dump`) — personal notes make backups mandatory.

---

## 6. Design (approved)

- **Clickable mockups v1 (approved):** a claude.ai artifact (link kept privately) — 10 phone screens, 390×844. Match them closely. An exported copy is kept **outside** this repo (privacy); ask the owner for it.
- **Direction:** light, Gmail-on-iPhone structure, own teal accent, Hebrew RTL, font **Heebo** (Google Fonts, weights 400/500/600/700; self-host the woff2 files for offline).
- **Tokens:**

| Token | Value |
|---|---|
| background | `#F6F8FC` |
| surface / cards | `#FFFFFF` |
| search pill, inputs | `#E9EEF6` / `#F0F4F9` |
| text | `#1F1F1F` |
| secondary text | `#5E5E5E` · tertiary icons `#444746` |
| borders | `#C4C7C5` · dividers `#E3E3E3` / `#EEF0F2` |
| accent | `#0C6F68` (buttons, unread dot, toggles) · hover/dark `#0A4D48` |
| selected chip / nav pill / FAB | `#CDE8E4` with text `#0A3D39` |
| timestamp pill | bg `#E6F2F0`, text `#0A4D48`, monospace, LTR |
| my notes | `#FEF9E7` card · `#FEF3C7` pill/segment · text `#6B4E00` |
| highlight (selected) | `#FDE68A` |
| quote badge | `#E0E7FF` / `#312E81` |
| star | fill `#F2B705`, stroke `#B88600`; empty stroke `#6B6B6B` |
| destructive | `#B3261E` |
| success dot | `#1E8E3E` |
| radius | rows/cards 20px · chips 8px · buttons & pills fully round · sheets 24px top |
| source palette | `#0C6F68` `#B45309` `#0369A1` `#1E3A8A` `#6D28D9` `#9D174D` `#4D7C0F` `#475569` |

- **Screens:** Inbox · Drawer · Search · Source folder · Episode · Transcript · Notebook · Sources · Source edit · Settings. Behaviour per §4.
- **Accessibility:** real `<button>`/`<a>`/`<input>` + `<label>`, `aria-label` on icon buttons, touch targets ≥ 44px, contrast ≥ 4.5:1, icons as inline stroke SVG (no emoji).
- iPhone specifics: `viewport-fit=cover` and `env(safe-area-inset-*)` padding, `apple-mobile-web-app-capable`, status bar style default (light), 180×180 apple-touch-icon, `manifest.webmanifest` (`dir: rtl`, `lang: he`, `display: standalone`, theme `#F6F8FC`).

---

## 7. API (n8n workflow "אינבוקס — API")

- One Webhook node: `POST https://<n8n-domain>/webhook/inbox-api`, response via "Respond to Webhook".
- Auth: n8n **Header Auth** credential **`Inbox API token`** (header `X-Inbox-Token`, long random value; the owner creates it in n8n and types the same value once into the app's Settings). Second new credential of the project.
- CORS: `allowedOrigins` = `https://<app-domain>` only. Handle `OPTIONS` preflight.
- Body: `{ "action": "<name>", ...params }`. A Switch node routes by `action` to a Postgres node per action. **Always use parameterized queries** (`$1, $2`, "Query Parameters"), never string-concatenate user input into SQL.
- Response: `{ ok: true, data }` or `{ ok: false, error: "<Hebrew message>" }` with status 400/401/404/500.

| action | params | does |
|---|---|---|
| `items_list` | `type?`, `source_id?`, `starred?`, `cursor?` (received_at,id), `limit` (≤50) | inbox/folder list: id, source (name, logo, type), title, summary snippet (≤140 chars), received_at, duration, is_read, is_starred, notes_count |
| `item_get` | `id` | full item + its notes |
| `item_update` | `id`, `is_read?`, `is_starred?` | |
| `item_delete` | `id` | |
| `search` | `q` (≥2 chars), `scope` (all/titles/transcripts/notes), `limit` | grouped hits with `before/match/after` snippets (±60 chars) and `time` for transcript hits |
| `sources_list` | | all sources with item counts and unread counts (used by drawer + Sources) |
| `source_create` | `type`, `name`, `feed_url`, `artwork_url?`, `itunes_id?`, `logo?` | validate URL (http/https, ≤500 chars), reject duplicates with a clear message |
| `source_update` | `id`, `name?`, `logo?`, `active?`, `spotify_show_url?` | logo image data URL ≤ 200 KB |
| `source_delete` | `id` | cascades |
| `note_create` / `note_update` / `note_delete` | `item_id`, `kind`, `anchor_sec?`, `anchor_text?`, `text?` / `id`, … | |
| `notebook` | `kind?` | notes joined with item + source, newest first |
| `settings_get` / `settings_update` | `key`, `value` | |
| `status` | | `{ok, last_item_at, counts}` for the Settings screen |
| `itunes_search` | `term` | only if the app can't call iTunes directly (CORS) |

Workflow settings: `errorWorkflow: <wf:error>`, `saveDataSuccessExecution: none` (don't store personal notes in execution logs), timezone `Asia/Jerusalem`.

---

## 8. Changes to existing workflows + new workflows

General: inspect each workflow with `get_workflow_details` first; edit with targeted `update_workflow` operations; keep the visual order and sticky notes (§9) updated; publish; test with pinned data.

### 8.1 `סיכום פודקאסטים שבועי` (`<wf:podcasts>`)
1. Replace Code node "רשימת פידים" with a Postgres node **"רשימת פידים מהמסד"**: `SELECT id AS source_id, feed_url AS "feedUrl" FROM sources WHERE type='podcast' AND active ORDER BY sort_order`. Downstream "משיכת פיד" keeps using `$json.feedUrl`. Carry `source_id` through "חילוץ פרקים" (it reads `$input.all()`; add `source_id` to each output item via paired items or by looking up `feedUrl`).
2. Change "בניית HTML" so it also outputs the parsed analysis (`analysis: a`) next to `html`, `fileName`, `subject`.
3. New node **"שמירה במסד"** (Postgres, upsert) between "שליחת מיייל" and "ניקוי" (inline, **not** a parallel branch: the loop closes through "ניקוי"). `INSERT ... ON CONFLICT (source_id, guid) DO UPDATE SET ...`. Fields: title, published_at (`pubIso`), summary, content `{highlights, quotes, glossary, transcript, podcast_name}`, original_url (`link`), audio_url, duration_sec (parse `itunes:duration`). Settings: `onError: continueRegularOutput`, retry 2× — a DB problem must never break the email.
4. Email switch: new If node **"האם לשלוח מייל"** before "קובץ מצורף" reading `settings.send_email` (one Postgres read at the start of the run). If false, skip straight to "שמירה במסד".
5. Update the sticky notes: the green "how to add a podcast" note now says "מוסיפים מהאפליקציה, מסך מקורות" (the code list is gone). Note the ⚠️ that "שמירה במסד" must stay inline before "ניקוי".

### 8.2 `ai news` (`<wf:ai-news>`) and `סיכום שבועי — ישראל ויחסי חוץ` (`<wf:world-news>`)
- Find the node with the final structured Gemini output. Add **"שמירה במסד"** (upsert, `guid` = ISO week key like `2026-W45`) with `summary` = intro and `content.items[]` (keep the workflow's own fields). Same email-switch If as 8.1. `onError: continueRegularOutput`.
- These workflows weren't fully mapped yet: inspect first and adapt the `content` shape in §5 to what they really produce; update this doc.

### 8.3 `פודקאסט לפי קישור — תמלול וסיכום` (`<wf:on-demand-worker>`)
- Logic partly lives in `/home/node/.n8n-files/podcast-on-demand/runtime.cjs` + `config.json` on the server. Inspect both before changing.
- Add "שמירה במסד" under source `on_demand`, with `spotify_episode_id` (from the submitted link) and `guid` = Spotify episode ID.

### 8.4 New: **"ניוזלטרים — איסוף"**
- Schedule every 3 hours. Postgres: active `newsletter` sources → for each: HTTP GET feed (RSS/Atom, timeout 60 s, retry) → parse XML → entries from the last 14 days → dedupe by `guid` (DB `ON CONFLICT DO NOTHING` is enough) → Code **"ניקוי HTML"**: strip `<script>`, `<style>` blocks that break layout, tracking pixels (1×1 imgs), `on*` attributes; keep links (add `target="_blank" rel="noopener"`) → insert item `{html, text}`, `summary` = first ~200 chars of text. **No AI calls.**
- The app renders newsletter HTML inside a sandboxed `<iframe sandbox="allow-popups allow-popups-to-escape-sandbox">` with `srcdoc`.

### 8.5 New: **"אינבוקס — מחיקה אוטומטית"**
- Daily 03:30. Read `retention_days`; for each type with a number: `DELETE FROM items i USING sources s WHERE i.source_id=s.id AND s.type=$1 AND i.received_at < now() - ($2 || ' days')::interval AND NOT i.is_starred AND NOT EXISTS (SELECT 1 FROM notes n WHERE n.item_id=i.id)`. Log counts.

### 8.6 New: **"אינבוקס — API"** (§7)

### 8.7 Optional backfill
- A one-time workflow or script that parses the last few weeks of summary emails is **not** required. Ask the owner whether he wants history; default: start empty.

---

## 9. n8n build rules (from `make-an-n8n`, mandatory)
1. **Visual order:** numbered stages left → right, one column per stage, nothing flows backwards. Y lanes: main `0`, outer loop body `260`, inner loop body `620`, each fallback node `+180` below its primary. Horizontal spacing 220–260.
2. **A sticky note per stage:** `## N️⃣ שם השלב`, then one line per node: `**שם הצומת** – מה הוא עושה`.
3. **Overview note** (color 7) on top: what it does + the route with arrows + a key to the notes. **Green note** (color 4): how to add an input/source, with a code block. **Blue note** (color 5): how to swap the AI model, listing every AI node and its model. Notes don't overlap: 20 px between columns; a note starts 240–320 px above its first node.
4. **⚠️ on every trap:** nodes referenced by name in `$('...')`, dedupe nodes, loop-closing connections.
5. **Node names in Hebrew, descriptive.** Never `Code`, `Postgres1`, `Schedule Trigger1`.
6. **Every AI call:** primary + fallback on a **different** model (`onError: continueErrorOutput` → fallback → same target), with `retryOnFail` and `timeout`; structured output via `responseSchema`, prompt and schema built in a Code node before it; the model returns `id`s, not links.
7. **Settings:** `errorWorkflow: <wf:error>`, timezone `Asia/Jerusalem`. Reuse existing credentials; the only new ones are `Inbox DB` and `Inbox API token`.
8. **MCP workflow:** `get_workflow_sdk_reference` → `search_nodes` → `get_node_types` → `validate_node_config` → `validate_workflow` → create/update → test → **publish** → update docs. Don't guess parameter names.
9. After each new/changed workflow: update the workflows table in the owner's Obsidian n8n workflows note (write access via the Obsidian MCP may be limited to `00-Inbox/`; if so, put an update note there and tell the owner) and §2.2 here.

---

## 10. Build plan — milestones (one at a time, tests at the end of each)

### M0 — Prep and safety
- [x] Ask the §11 questions; wait for answers. (answers in §11)
- [ ] Guide the owner to update DuckDNS `<n8n-subdomain>` to `<כתובת-השרת>`, the SSH `HostName`, and the Obsidian server note. Verify: `(Resolve-DnsName <n8n-domain> -Type A).IPAddress` returns the new IP; `ssh n8n` works; n8n opens.
- [ ] Check backups exist (`ls -lh ~/backups | tail -5`). If none, set up `~/backup-n8n.sh` + daily cron (README §14) **before** creating personal data.
- [ ] **Capacity check** (expected to be fine, see §13.1): `df -h /` · `free -h` · `docker stats --no-stream` · `docker system df` · `du -sh ~/backups`. Go ahead if disk has ≥ 10 GB free and RAM ≥ 4 GB available. If disk is tight: `docker image prune` (ask first), rotate old backups, check n8n execution-data pruning (`EXECUTIONS_DATA_PRUNE=true`, `EXECUTIONS_DATA_MAX_AGE=168` in the override file).
- [x] Create the GitHub repo and a `CLAUDE.md` that points to `docs/SPEC.md`.
- **Your tests:** DNS resolves, n8n reachable over HTTPS, a fresh backup file exists, capacity numbers reported to the owner.
- **The owner's tests:** open https://<n8n-domain> on the phone; run `ssh n8n` and see the `ubuntu@<שם-השרת>` prompt.

### M1 — Database
- [ ] Create database `inbox` + user `inbox_app` inside `n8n-postgres` (give the owner the exact commands; he runs them over SSH, or you run them if you have shell access to the server).
- [ ] Run `sql/001_schema.sql`, `sql/002_seed.sql`.
- [ ] the owner creates the n8n credential `Inbox DB`.
- [ ] Add `inbox` to the backup script.
- **Your tests:** a throwaway n8n test workflow with the `Inbox DB` credential runs `SELECT count(*) FROM sources` → 9 rows (5 podcasts, 2 digests, 1 on-demand, 1 newsletter); insert + delete a test item; `pg_dump inbox` produces a file.
- **The owner's tests:** in n8n, open the credential and click "Test" → green.

### M2 — API
- [ ] Build "אינבוקס — API" (§7) with the `Inbox API token` credential (The owner creates it).
- **Your tests:** curl every action: with no token → 401; with token → expected JSON; SQL-injection string in `q` → treated as text; `OPTIONS` from the app origin → CORS OK; another origin → blocked.
- **The owner's tests:** none technical; confirm he saved the token in his password manager.

### M3 — PWA shell + hosting
- [ ] Repo `app/`: `index.html`, `styles.css`, `js/` (router, api client, screens), `manifest.webmanifest`, `sw.js` (cache app shell; network-first for API), icons, self-hosted Heebo.
- [ ] Server: `~/docker/inbox/` with `nginx:alpine` serving `app/` (read-only volume), on the same Docker network as `npm`. DuckDNS subdomain `<app-subdomain>` → server IP. NPM proxy host + Let's Encrypt. Deploy = `git pull` in `~/docker/inbox/app` (document the command).
- [ ] Screens with real data: Inbox (list, filters, star, read state), Drawer, Settings (token entry, status).
- **Your tests:** Lighthouse PWA/installability check (or equivalent), RTL layout at 390 px, no console errors, API calls succeed with the token.
- **The owner's tests:** on the iPhone, open https://<app-domain> in Safari → Share → "הוסף למסך הבית" → open from the icon (full screen, no Safari bars) → paste the token in Settings → see the seeded sources in the drawer.

### M4 — Ingest: podcasts end to end
- [ ] §8.1 changes, publish.
- [ ] Test with pinned data from a past execution (no new Gemini calls), then let the real Friday run happen, or do **one** real run on a single short episode if the owner agrees.
- [ ] Episode screen (all tabs), Transcript screen (continuous text + side timestamps), notes/highlights, in-app player, Spotify links where an ID exists.
- **Your tests:** the pinned run writes one item with full content; email still arrives (send_email=true); re-running doesn't duplicate (upsert); transcript parser handles `[MM:SS]`, `[H:MM:SS]`, speaker prefixes and `[לא ברור]`.
- **The owner's tests:** open the episode on the phone; check each tab; tap a timestamp → player jumps to the right second; tap a transcript sentence → add a note → it shows yellow with a dot; the note appears in the Notebook; **test a Spotify `?t=` link from an on-demand episode opens the Spotify app at the right time.**

### M5 — Ingest: news digests, on-demand, newsletters, retention
- [ ] §8.2, §8.3, §8.4, §8.5. Publish each.
- [ ] Digest screen (intro + list of items with links), newsletter screen (sandboxed HTML).
- **Your tests:** each workflow writes items correctly with pinned data; newsletter run on the <newsletter-1> feed creates items with clean HTML; retention dry-run (`SELECT` version of the delete) returns the expected rows and never starred/annotated ones.
- **The owner's tests:** a news digest and the latest <newsletter-1> issue look right on the phone; links open.

### M6 — Search, sources management, logos, Obsidian export
- [ ] Search screen + API. Sources screen (iTunes search, add, pause, remove, paste RSS, KTN instructions). Source edit (logo types, color, shape, upload resize, Spotify show link). Folder pages. Notebook + Obsidian export.
- **Your tests:** search returns transcript hits with correct timestamps; adding a podcast via iTunes creates a source with artwork; a paused source is skipped by the weekly query; uploaded logo is ≤ 200 KB; the export Markdown is valid and under the URL limit.
- **The owner's tests:** search "שיחה" and open a transcript hit; add a new podcast by name and see its empty folder ("הפרקים הראשונים יגיעו ביום שישי"); change a logo to letters + color, then to a phone photo; export an episode → the note opens in Obsidian `00-Inbox`; on the following Friday, the new podcast's episodes arrive.

### M7 — Polish and cutover
- [ ] Swipe-to-delete + undo, pull-to-refresh, empty/error/loading states in Hebrew, offline app shell.
- [ ] After a week of the owner using the app happily: he switches off "לשלוח גם במייל" in Settings.
- [ ] Final docs: Obsidian project note for the app, workflow table updated, this file's log closed.
- **The owner's tests:** a full week of normal use; one week with emails off; nothing missing.

---

## 11. Questions to confirm with the owner BEFORE starting
Ask these (in Hebrew, numbered, one message), plus anything you find unclear:
1. Does Claude Code have shell access to the server (SSH from this machine), or should every server command be given to the owner to run?
2. Repo name `itoosh-45/summaries-inbox` OK? Public or private (recommended: private)?
3. Subdomain `<app-domain>` OK?
4. Has DuckDNS already been updated to the new IP `<כתובת-השרת>`?
5. Backups: is `~/backup-n8n.sh` + cron set up today?
6. Start empty, or import a few weeks of existing summary emails as history?
7. For testing M4: OK to spend one real Gemini run on a single short episode, or only pinned data until the Friday run?
8. Spotify: does the owner have Spotify Premium? (Affects whether the Spotify Web API route is worth checking for matching RSS episodes.)

### Answers (The owner, 2026-11-06)
1. Shell access: **no**. The cloud session reaches the internet over HTTPS only. Every server command is given to the owner to run (`ssh n8n` from PowerShell).
2. Repo: `itoosh-45/summaries-inbox`, **private**.
3. Subdomain `<app-domain>`: **OK**.
4. DuckDNS updated to the new IP: **yes** (verify in M0).
5. Backups: **unknown** → check in M0.
6. History: **start empty** (no backfill, §8.7 skipped).
7. M4 testing: **pinned data from old executions only**, no extra Gemini runs; real data arrives with the Friday run.
8. Spotify Premium: **yes** → worth checking the Spotify Web API for matching RSS episodes (M4).

## 11.1 Still-open technical items (decide during the build, report the decision)
- Spotify Web API on a free account for matching RSS episodes → if not possible, in-app player only for RSS episodes.
- iTunes Search API CORS from the PWA → if blocked, use the `itunes_search` API action.
- Exact JSON shape of the two news digests (§8.2).
- Kill the Newsletter blocked by a publisher → fall back to forwarding from a regular address.

---

## 12. Decisions (all dated 2026-11-06)
| Decision | Why |
|---|---|
| Stay on n8n + free Gemini + Oracle server; don't move processing to Claude | Zero cost. Claude API is pay-per-use; a Claude subscription doesn't include API calls |
| PWA, not native iOS | Free, no Mac, no 7-day expiry |
| Postgres in the existing container, separate `inbox` DB | Handles long transcripts and search; already running |
| App hosted on the server (nginx + NPM + DuckDNS `<app-subdomain>`) | Everything stays on the owner's server |
| Emails keep running in parallel until switched off in Settings | Safety net during migration |
| Newsletters: own RSS first, else Kill the Newsletter; no Google | Free, same mechanism as RSS |
| <newsletter-1> = own RSS; stays in `ai news` **and** is its own newsletter source | The owner's choice |
| News: one combined weekly digest per item | Same as today |
| Podcasts: few, swappable from the app via iTunes search | Flexibility; protects Gemini quota |
| New podcasts wait for the Friday run (no immediate processing) | The owner's choice; protects quota |
| MVP: notebook, Obsidian export, timestamps → Spotify with in-app player fallback, search, folder per source, editable logos | The owner's choices |
| Retention per type, set in the app | The owner's choice |
| Transcript = continuous running text, timestamps in a side gutter, no paragraphs, no reordering | The owner's choice, design approved |
| Design v1 approved: light, Gmail-like structure, teal accent, Heebo | The owner approved |
| No unread reminder, no reading time, no old-episode processing, no Hebrew TTS workflow | Not needed |
| Spec in English, app UI in Hebrew | The owner's choice |
| Repo `itoosh-45/summaries-inbox`, private; no personal data in it (placeholders only), mockups kept outside it | The owner's choice; the repo may become public by mistake |
| Inbox starts empty, no email backfill | The owner's choice |
| Tests use pinned data only, no extra Gemini runs | Protects the free quota |
| Claude Code works from a cloud environment without SSH; the owner runs server commands | HTTPS-only sandbox; SSH keys never leave the owner's PC |

## 13. Risks & tech debt

### 13.1 Server capacity (estimate, verify in M0)
- Server: 2 OCPU / 12 GB RAM; Oracle free-tier boot volume is usually ~47 GB (free tier allows up to 200 GB of block storage).
- **Database growth is tiny:** everything stored is text. A 1-hour transcript ≈ 50–100 KB; summary + highlights + quotes ≈ 5 KB. ~4 episodes/week + 3 digests/week + newsletters ≈ **20–40 MB per year**, less with auto-delete. Uploaded logos ≤ 200 KB each.
- **No audio is stored:** ffmpeg chunks live in `/tmp/podcasts/<id>` only during processing (~15 MB per hour of audio) and node "ניקוי" deletes them.
- **New services:** `nginx:alpine` ≈ 10 MB RAM, ~25 MB disk. The `inbox` DB adds almost nothing to the existing Postgres.
- **Things that do grow and must be watched:** n8n execution data (enable pruning), Docker images after upgrades, local backups (keep the last 14).

### 13.2 Other risks
- **Free Gemini:** Google may use free-tier content to improve models. Fine for public podcasts. **Never send personal notes to Gemini.**
- **Oracle may reclaim idle free instances.** The IP already changed once: DuckDNS + SSH config must follow (consider the DuckDNS auto-update cron, README §15).
- **Security:** the existing "פודקאסט לפי קישור" webhooks are public without auth (pre-existing; out of scope unless the owner asks). The new API **must** use the header token and a strict CORS origin. Disable saving successful API executions so notes don't sit in n8n logs.
- **Backups:** mandatory once notes exist (M0/M1). Weekly copy off the server: `scp -r n8n:~/backups C:\Users\<user>\n8n-backups`.
- Still open from the server note: 2FA on the n8n account; `N8N_ENCRYPTION_KEY` saved in a password manager.

## 14. Log
- v0.1–0.7 (2026-11-06): features defined with the owner, all workflows and the Obsidian vault mapped, architecture chosen, mockups designed and approved, transcript format finalized.
- v1.0 (2026-11-06): full build spec for Claude Code: DB schema, API, workflow changes, milestones with tests, questions to ask first.
- v1.1 (2026-11-06): §11 answered; repo created; cloud environment `summaries-inbox` set up (tested: §5 schema runs on local Postgres 16); M0 started.
