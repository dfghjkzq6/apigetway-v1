# AGENTS.md — TokenGate Project Guide
# This file tells any AI assistant how this project works.
# Read this before writing or suggesting any code.

---

## 🧠 Who is the author?
- Total beginner — no developer background
- Mobile-first: every UI must work perfectly on a phone screen first
- Needs complete, copy-paste ready code — never partial snippets
- Needs numbered steps when there is a sequence to follow
- Never assume prior knowledge — explain what and why, briefly

---

## 📦 What is TokenGate?
A personal API token vault and proxy panel.
- Store real API tokens once, in the database
- Tools hit a proxy URL instead of the real API
- Admin panel to manage gateways, tokens, and read logs
- One switch flips to cut off any service instantly

---

## 🗂️ Project Stack — never suggest alternatives
| Layer | Tool |
|-------|------|
| Framework | Next.js 15 (App Router) |
| Language | JavaScript only — NO TypeScript in new files |
| Styling | Tailwind CSS — mobile first |
| Database | PostgreSQL on Neon (serverless) |
| ORM | Drizzle ORM |
| Hosting | Vercel |
| Package manager | npm |

---

## 🗄️ Database — 3 tables only

### gateways
```js
id          UUID  PK
name        TEXT  unique         // used as proxy URL slug — no spaces, URL-safe
base_url    TEXT                 // real upstream API base URL
status      TEXT  ON | OFF
gateway_key TEXT  nullable       // null = open mode, set = protected mode
created_at  TIMESTAMP
```

### tokens
```js
id           UUID  PK
gateway_id   UUID  FK → gateways.id  (1-to-1, cascade delete)
token_value  TEXT                    // the real secret token
created_at   TIMESTAMP
```

### logs
```js
id          UUID  PK
gateway_id  UUID  FK → gateways.id  (SET NULL on delete)
result      TEXT  ALLOWED | BLOCKED | OFF | NOT_FOUND
called_by   TEXT
endpoint    TEXT
created_at  TIMESTAMP
```

> Logs are INSERT ONLY — never update or delete log rows.
> Always use LEFT JOIN when joining logs to gateways in queries.

---

## 📁 Project Structure — never add files outside this tree

```
tokengate/
├── .env.local                        ← secrets live here only
├── drizzle.config.js                 ← Drizzle config
├── db/
│   ├── schema.js                     ← single source of truth for all tables
│   ├── index.js                      ← exports drizzle db instance
│   └── migrations/                   ← auto-generated, never edit manually
├── lib/
│   ├── db.js                         ← Neon connection
│   ├── config.js                     ← reads from .env.local
│   └── utils.js                      ← small helpers
├── app/
│   ├── layout.tsx                    ← root layout (keep as .tsx, dont change)
│   ├── page.tsx                      ← dashboard (keep as .tsx, dont change)
│   ├── gateways/
│   │   ├── page.jsx
│   │   └── [name]/page.jsx
│   ├── logs/page.jsx
│   ├── analytics/page.jsx
│   ├── settings/page.jsx
│   └── api/
│       ├── gateway/[name]/route.js   ← THE PROXY — most critical file
│       └── gateways/
│           ├── route.js              ← GET list + POST create
│           └── [name]/route.js       ← PATCH edit + DELETE
└── components/
    ├── layout/
    │   ├── BottomNav.jsx             ← mobile bottom navigation
    │   └── PageShell.jsx             ← wraps every page
    ├── ui/
    │   ├── Badge.jsx                 ← status pills
    │   ├── Button.jsx
    │   ├── Card.jsx
    │   └── Modal.jsx
    └── gateway/
        ├── GatewayCard.jsx
        ├── GatewayForm.jsx
        └── LogRow.jsx
```

---

## 🔑 Environment Variables — all in .env.local

```bash
DATABASE_URL          # Neon PostgreSQL connection string
NEXT_PUBLIC_APP_NAME  # Display name of the app
NEXT_PUBLIC_APP_URL   # Base URL (localhost in dev, real URL in prod)
```

> Never hardcode any value that is in .env.local.
> Always read it through lib/config.js or process.env.

---

## 🔌 API Routes — how they work

### The Proxy (most important)
```
POST /api/gateway/[name]
1. Look up gateway by name in gateways table
2. If not found → log NOT_FOUND → return 404
3. If status = OFF → log OFF → return 403
4. If gateway_key set and caller key wrong → log BLOCKED → return 403
5. Fetch token_value from tokens table
6. Inject token into request headers
7. Forward request to base_url
8. Log ALLOWED
9. Return upstream response to caller
```

### Gateway CRUD
```
GET    /api/gateways          → SELECT * FROM gateways
POST   /api/gateways          → INSERT INTO gateways + tokens
PATCH  /api/gateways/[name]   → UPDATE gateways SET ...
DELETE /api/gateways/[name]   → DELETE FROM gateways (cascades to tokens)
```

---

## 🎨 UI & Styling Rules

- **Mobile first always** — design for 390px width, then scale up
- Use Tailwind utility classes only — no custom CSS files
- Bottom navigation bar for mobile (BottomNav.jsx)
- Every page is wrapped in PageShell.jsx
- Status colours:
  - ON / ALLOWED → green
  - OFF / BLOCKED → red / gray
  - NOT_FOUND → amber
  - Protected mode → yellow

---

## ✅ Code Rules — always follow these

1. **JavaScript only** — never write TypeScript in new .js or .jsx files
2. **Complete files only** — never give partial code or "add this section" snippets
3. **One file at a time** — finish one file completely before moving to the next
4. **No inline secrets** — all config comes from .env.local via lib/config.js
5. **Drizzle for all DB queries** — never write raw SQL strings in route files
6. **db/schema.js is the single source of truth** — never define table structure anywhere else
7. **logs are insert-only** — never write UPDATE or DELETE on the logs table
8. **API routes are the only files that touch the database** — pages fetch from API routes
9. **Always use LEFT JOIN** when joining logs to gateways
10. **gateway name must be URL-safe** — validate: lowercase, no spaces, no special chars

---

## 🚫 Never Do These

- Never use TypeScript in new files (existing .tsx files are fine)
- Never write raw SQL — use Drizzle ORM methods
- Never put secrets in any file other than .env.local
- Never edit files inside db/migrations/ manually
- Never write UPDATE or DELETE queries against the logs table
- Never add new dependencies without explaining what they do and why
- Never create files outside the defined project tree
- Never give partial code snippets — always give the full file

---

## 🔁 Build Order (follow this sequence)

```
Phase 1 — Foundation
  1. drizzle.config.js
  2. db/schema.js
  3. db/index.js
  4. lib/config.js
  5. lib/db.js
  → Run: npx drizzle-kit push

Phase 2 — API Routes
  6. app/api/gateways/route.js          (GET + POST)
  7. app/api/gateways/[name]/route.js   (PATCH + DELETE)
  8. app/api/gateway/[name]/route.js    (THE PROXY)

Phase 3 — Pages
  9.  app/layout.tsx                    (root layout + BottomNav)
  10. app/page.tsx                      (Dashboard)
  11. app/gateways/page.jsx             (Gateway list)
  12. app/gateways/[name]/page.jsx      (Gateway detail)
  13. app/logs/page.jsx
  14. app/analytics/page.jsx
  15. app/settings/page.jsx

Phase 4 — Components (build as needed)
  16. components/layout/PageShell.jsx
  17. components/layout/BottomNav.jsx
  18. components/ui/Badge.jsx
  19. components/ui/Button.jsx
  20. components/ui/Card.jsx
  21. components/ui/Modal.jsx
  22. components/gateway/GatewayCard.jsx
  23. components/gateway/GatewayForm.jsx
  24. components/gateway/LogRow.jsx
```

---

## 🧪 How to test locally
```bash
npm run dev          # start dev server at localhost:3000
npx drizzle-kit push # push schema changes to Neon DB
npx drizzle-kit studio # visual DB browser (optional)
```

---

## 🚀 How to deploy
```bash
# Push to GitHub, then connect repo to Vercel
# Add all .env.local variables to Vercel Environment Variables
# Vercel auto-deploys on every git push to main
```

---

## 📋 Current Status
- [x] Project scaffolded
- [x] Folder structure created
- [x] Dependencies installed (drizzle-orm, @neondatabase/serverless, drizzle-kit)
- [ ] drizzle.config.js written
- [ ] db/schema.js written
- [ ] db/index.js written
- [ ] lib/config.js written
- [ ] lib/db.js written
- [ ] Migration run (npx drizzle-kit push)
- [ ] API routes built
- [ ] Pages built
- [ ] Components built
- [ ] Deployed to Vercel