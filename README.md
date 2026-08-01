# CodeStats *(rename me)*

A self-hosted, WakaTime-style coding activity tracker. It tracks the languages and projects you work on in VS Code, aggregates the time into stats, and displays them on a personal dashboard **and** on your GitHub profile/README via a live badge.

---

## ✨ Features

- 🔑 **GitHub OAuth login** — sign in with GitHub, no separate password system
- 🧩 **VS Code extension** — silently tracks coding activity in the background via editor events
- ⏱️ **Idle-aware time tracking** — uses a gap-based algorithm (like WakaTime) so idle/AFK time isn't counted
- 📊 **One-page dashboard** — language breakdown, project breakdown, daily/weekly activity
- 🔐 **API key management** — generate/revoke keys used by the extension to authenticate
- 🛡️ **Privacy controls** — exclude specific projects or file paths from tracking
- 🏷️ **GitHub badge** — a dynamically generated SVG (`/api/badge/username.svg`) you can embed in any README
- 🤖 **GitHub Action (optional)** — auto-updates a section of your README on a schedule, no manual badge refresh needed

---

## 🏗️ Architecture

```
┌─────────────────┐      heartbeats       ┌──────────────────┐
│  VS Code         │  ───────────────────▶ │  Ingestion API    │
│  Extension        │   (batched, via API   │  (Vercel function) │
│                   │    key auth)          │                   │
└─────────────────┘                        └─────────┬─────────┘
                                                       │ writes
                                                       ▼
                                             ┌───────────────────┐
                                             │  Postgres           │
                                             │  Heartbeats (raw)   │
                                             │  DailySummary (agg) │
                                             └─────────┬───────────┘
                                                       │ read
                       ┌───────────────────────────────┼───────────────────────────────┐
                       ▼                               ▼                               ▼
             ┌───────────────────┐         ┌──────────────────────┐        ┌──────────────────────┐
             │  Web Dashboard      │         │  Badge API (SVG)       │        │  GitHub Action          │
             │  (Next.js, auth'd)  │         │  /api/badge/[user].svg │        │  rewrites README stats  │
             └───────────────────┘         └──────────────────────┘        └──────────────────────┘
                                                       │
                                                       ▼
                                             embedded in any
                                             GitHub README via
                                             ![stats](badge-url)
```

---

## 🔄 End-to-End Flow

### 1. Setup
1. User signs up on the web dashboard via **GitHub OAuth**.
2. User generates an **API key** from the dashboard.
3. User installs the **VS Code extension** and pastes the API key (stored securely via `SecretStorage`).

### 2. Tracking (while coding)
1. Extension listens to editor events: file open/save/edit, window focus change.
2. On activity, it builds a **heartbeat**: `{ timestamp, file, language, project, is_write }`.
3. Heartbeats are throttled client-side and queued locally.
4. Every 30–60s (or on window blur), the queue is **flushed as a batch** to the Ingestion API.
5. If offline, heartbeats stay queued and retry later.

### 3. Ingestion & Aggregation
1. API validates the API key and the heartbeat payload (timestamp sanity, batch size limits).
2. Raw heartbeats are stored in the database.
3. A background process (on-write or via **Vercel Cron**) applies a **gap-based duration algorithm**:
   - Heartbeats within a threshold (e.g. 15 min) of each other = continuous coding time.
   - A larger gap = session boundary / idle time, not counted.
4. Results are rolled up into `DailySummary` rows: `(user, date, project, language, seconds)`.

### 4. Visualization
1. **Dashboard** queries `DailySummary` and renders charts: languages, projects, daily/weekly trends.
2. **Badge API** queries the same aggregated data and renders an on-the-fly SVG.
3. **GitHub README** embeds the badge as an image, or a **GitHub Action** periodically fetches stats and rewrites a marked section of the README directly.

---

## 🧱 Tech Stack

| Layer            | Choice                                   |
|-------------------|-------------------------------------------|
| Frontend/API      | Next.js (App Router) on Vercel            |
| Database          | Postgres (Neon / Vercel Postgres)         |
| ORM               | Prisma                                    |
| Auth              | NextAuth.js (GitHub provider)             |
| Extension         | VS Code Extension API (TypeScript)        |
| Scheduling        | Vercel Cron Jobs / GitHub Actions         |
| Charts            | Recharts                                  |

---

## 📂 Suggested Repo Structure

```
.
├── apps/
│   ├── web/                 # Next.js dashboard + API routes
│   │   ├── app/
│   │   │   ├── dashboard/
│   │   │   ├── api/
│   │   │   │   ├── heartbeats/route.ts
│   │   │   │   ├── badge/[user]/route.ts
│   │   │   │   └── cron/aggregate/route.ts
│   │   │   └── auth/
│   │   └── prisma/schema.prisma
│   └── vscode-extension/    # Extension source
│       ├── src/extension.ts
│       └── package.json
└── README.md
```

---

## 🚧 Roadmap / Known Edge Cases

- [ ] Handle overlapping heartbeats from multiple open VS Code windows (dedup, not double-count)
- [ ] Map VS Code `languageId` to GitHub linguist-style categories
- [ ] Strip absolute file paths before storing (privacy)
- [ ] Rate-limit the ingestion endpoint
- [ ] Handle clock skew between client and server timestamps
- [ ] Cache-control tuning so GitHub's image proxy refreshes badges properly

---

## 📜 License

MIT