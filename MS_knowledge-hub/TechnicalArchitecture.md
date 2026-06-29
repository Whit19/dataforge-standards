# TechnicalArchitecture — knowledge-hub
Last updated: 2026-06-10

---

## What This Is

A unified platform with two modules running on one Express server:

| Module | Interaction Model | Content Type |
|--------|------------------|--------------|
| Ask the Docs | AI query — upload a PDF, ask questions | White papers, supplier docs, any PDF |
| Tech Team Wiki | Browse, create, and edit — full CMS | Human-written pages maintained by Dale |

These are intentionally different tools. Ask the Docs answers "What does the Novozymes white paper say about silage?" The wiki answers "How do we blend B. subtilis?"

---

## Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Runtime | Node.js v22+ | |
| Framework | Express | REST API — React handles all wiki rendering |
| AI | Anthropic API — `claude-sonnet-4-20250514` | |
| File handling | multer (upload), mammoth (DOCX extraction) | |
| Ask the Docs frontend | Plain HTML/CSS/JS (single file) | No build step |
| Wiki frontend | React SPA | Required for dynamic block editor |
| Database | SQLite via `better-sqlite3` | Persistent Railway volume at `/data/wiki.db` |
| Auth | JWT (`jsonwebtoken`) + bcrypt | Phase 2 option: Google SSO @microsynergies.com |
| Hosting | Railway (~$5–10/month) | |
| Session store | In-memory (Phase 1); Redis planned for Phase 2 | |

**Why not Firebase/Firestore:** This is a document tool with an AI layer, not a real-time user-facing app. No live sync, no mobile PWA, no push notifications needed. Node/Express on Railway is simpler to explain to the client and fits the actual requirements.

**Why SQLite (not PostgreSQL):** For a team of 11–50, SQLite on a Railway persistent volume handles concurrent reads/writes comfortably. Simpler deployment, no separate DB service, trivial to migrate to Postgres later if needed.

---

## File Structure

```
knowledge-hub/
├── docs/                        ← Project MD files (never published)
│   ├── SessionStarter.md
│   ├── TechnicalArchitecture.md
│   ├── ProjectRoadmap.md
│   ├── DecisionLog.md
│   ├── IssuesTracker.md
│   ├── BestMethods.md
│   └── TimeLog.md
├── public/
│   └── microsynergies_demo_v4.html   ← Default root route (demo)
├── wiki-client/                 ← React SPA (wiki frontend)
│   ├── src/
│   └── dist/                    ← Built output, served by Express
├── uploads/                     ← Temp dir — files deleted after upload
├── data/
│   ├── wiki.db                  ← SQLite database (Railway persistent volume)
│   └── document-manifest.json  ← Tracks file_id, summary, topics per PDF
├── server.js                    ← All routes: Ask the Docs + Wiki API + Auth
├── db/
│   └── init.js                  ← Schema creation + initial Admin user seed
├── .env                         ← [never commit] ANTHROPIC_API_KEY, JWT_SECRET, PORT
├── .gitignore                   ← .env, node_modules, uploads/*, data/wiki.db
└── package.json
```

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Anthropic API key — MicroSynergies owns this in production |
| `JWT_SECRET` | Random secret for signing JWTs — generate once, store in Railway dashboard |
| `PORT` | Server port — default `3000` |
| `NODE_ENV` | `production` on Railway |

---

## Dependencies

```json
{
  "@anthropic-ai/sdk": "latest",
  "express": "^4.x",
  "multer": "^1.x",
  "mammoth": "^1.x",
  "better-sqlite3": "^9.x",
  "jsonwebtoken": "^9.x",
  "bcrypt": "^5.x",
  "cors": "^2.x",
  "dotenv": "^16.x"
}
```

---

## Module 1 — Ask the Docs

### Architecture Flow

```
User uploads PDF/DOCX
        │
        ▼
   multer (temp file)
        │
        ├── DOCX → mammoth.extractRawText() → store text in session
        │
        └── PDF  → anthropic.beta.files.upload() → store file_id in session
                            │
                            ▼
                   Anthropic Files API
                   (retained ≤7 days, never trained on)

User asks a question
        │
        ▼
   POST /api/ask { sessionId, question }
        │
        ├── PDF session  → beta.messages.create with document block + citations
        └── Text session → messages.create with full text in prompt
                            │
                            ▼
                   Return { answer, inputTokens, outputTokens }
```

### API Routes

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/health` | Health check |
| POST | `/api/upload` | Upload PDF or DOCX → returns `sessionId`, `type`, `fileId` (PDF) or `charCount` (DOCX) |
| POST | `/api/ask` | `{ sessionId, question }` → returns `{ answer, inputTokens, outputTokens }` |
| DELETE | `/api/session/:id` | Deletes session + removes file from Anthropic storage |

### Session Store (in-memory)

```javascript
// Keyed by sessionId — lives in process memory
{
  "session_1234_abc": {
    type: "pdf",
    fileId: "file_011...",   // Anthropic Files API ID
    name: "whitepaper.pdf"
  },
  "session_5678_def": {
    type: "text",
    content: "full extracted text...",
    name: "document.docx"
  }
}
```

> Resets on server restart. Acceptable for Phase 1. Redis planned for Phase 2.

### Critical SDK Pattern — Do Not Revert

```javascript
// ✅ CORRECT — betas inside the call params
import Anthropic, { toFile } from "@anthropic-ai/sdk";

await anthropic.beta.files.upload({
  file: await toFile(buffer, filename, { type: "application/pdf" }),
  betas: ["files-api-2025-04-14"],
});

await anthropic.beta.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1500,
  messages: [...],
  betas: ["files-api-2025-04-14"],
});

await anthropic.beta.files.delete(fileId, {
  betas: ["files-api-2025-04-14"],
});

// ❌ WRONG — causes 404 not_found_error every time
await anthropic.beta.files.upload(
  { file: ... },
  { headers: { "anthropic-beta": "files-api-2025-04-14" } }
);
```

### Cost Reference

| Item | Rate |
|------|------|
| Input tokens | ~$3 / million tokens |
| Output tokens | ~$15 / million tokens |
| Typical session (10 questions, 50-page doc) | ~$0.10–$0.30 |

MicroSynergies owns their API key — costs billed directly to them.

---

## Module 2 — Tech Team Wiki (CMS)

### What Changed From Original Scope

The original Phase 1 was scoped as a static HTML wiki with in-browser markdown editing committed back to GitHub. Brad and Dale's kickoff feedback changed the requirements substantially:

| Original Scope | Revised Scope |
|----------------|--------------|
| Static HTML pages, markdown files | Database-backed CMS |
| Single shared password | Role-based access (Viewer / Editor / Admin) |
| Textarea editor, GitHub commit on save | Block-based in-app editor — no code access required |
| No approval workflow | Three-state approval flags (Green / Yellow / Red) |
| GitHub history as versioning | Full DB version snapshots with editor + timestamp |
| Basic search | Tag-filtered search across all content |

### System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Browser (any device)                          │
│              React SPA — View, Edit, Admin UI                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS / REST
┌──────────────────────────▼──────────────────────────────────────┐
│                  Node.js / Express Server                        │
│  ┌─────────────┐  ┌────────────────┐  ┌───────────────────────┐ │
│  │  Auth API   │  │  Pages API     │  │  Admin API            │ │
│  │  /auth/*    │  │  /pages/*      │  │  /admin/*             │ │
│  │             │  │  /sections/*   │  │  /users/*             │ │
│  │  JWT issuer │  │  /versions/*   │  │  /categories/*        │ │
│  └─────────────┘  └───────┬────────┘  └───────────────────────┘ │
└─────────────────────────────┬─────────────────────────────────────┘
                              │
               ┌──────────────▼──────────────┐
               │       SQLite Database         │
               │     (Railway /data volume)    │
               │                               │
               │  pages  ·  sections           │
               │  page_versions  ·  users      │
               │  categories  ·  sessions      │
               └───────────────────────────────┘
```

### Database Schema

```sql
CREATE TABLE users (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  name        TEXT NOT NULL,
  email       TEXT UNIQUE NOT NULL,
  password    TEXT NOT NULL,          -- bcrypt hash
  role        TEXT NOT NULL DEFAULT 'viewer',  -- 'viewer' | 'editor' | 'admin'
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_login  DATETIME
);

CREATE TABLE categories (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  title       TEXT NOT NULL,
  slug        TEXT UNIQUE NOT NULL,
  sort_order  INTEGER DEFAULT 0
);

CREATE TABLE pages (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  category_id   INTEGER REFERENCES categories(id),
  title         TEXT NOT NULL,
  slug          TEXT UNIQUE NOT NULL,
  status        TEXT NOT NULL DEFAULT 'draft',  -- 'draft' | 'needs_review' | 'approved'
  created_by    INTEGER REFERENCES users(id),
  updated_by    INTEGER REFERENCES users(id),
  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
  tags          TEXT                              -- JSON array stored as string
);

CREATE TABLE sections (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  page_id     INTEGER NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  type        TEXT NOT NULL,   -- see Section Types below
  content     TEXT NOT NULL,   -- JSON blob
  sort_order  INTEGER DEFAULT 0
);

CREATE TABLE page_versions (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  page_id          INTEGER NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  title            TEXT NOT NULL,
  sections_snapshot TEXT NOT NULL,  -- full JSON snapshot of sections at save time
  saved_by         INTEGER REFERENCES users(id),
  saved_at         DATETIME DEFAULT CURRENT_TIMESTAMP,
  change_note      TEXT
);

CREATE TABLE sessions (
  id          TEXT PRIMARY KEY,     -- UUID
  user_id     INTEGER REFERENCES users(id),
  token       TEXT NOT NULL,
  expires_at  DATETIME NOT NULL
);
```

### Section / Block Types

Each section row has a `type` string and a `content` JSON blob. The editor renders the appropriate block UI per type.

| Type | Content JSON Shape | Rendered As |
|------|--------------------|-------------|
| `richtext` | `{ html: "..." }` | Formatted paragraphs, bold, italic, headings |
| `accordion` | `{ title: "...", body: "..." }` | Collapsible dropdown |
| `table` | `{ headers: [...], rows: [[...]] }` | Sortable, filterable table |
| `bullets` | `{ items: ["...", "..."] }` | Bulleted list |
| `image` | `{ url: "...", caption: "...", alt: "..." }` | Inline image with optional caption |
| `link` | `{ label: "...", url: "...", link_type: "external\|internal\|onedrive", target_page_id: null\|int }` | Styled hyperlink block |
| `warning` | `{ level: "info\|warning\|danger", message: "..." }` | Color-coded callout box |
| `tags` | `{ tags: ["...", "..."] }` | Clickable tag badges, searchable |

**Hyperlink types:**
- `external` — standard URL, opens in new tab
- `internal` — links to another wiki page by `target_page_id`; rendered as `/wiki/[slug]`; page picker dropdown in editor
- `onedrive` — paste any OneDrive share URL; renders as a styled file link

> **OneDrive note:** Phase 1 treats OneDrive links as plain HTTPS URLs. Works for any file shared via "Anyone with the link." Files requiring MS account login will prompt users to sign in — no special integration needed. Confirm sharing model with Brad/Dale before launch.

### Approval Workflow

```
[draft]  ──Editor submits──►  [needs_review]  ──Admin approves──►  [approved]
   ▲                                │                                    │
   └────────Admin rejects───────────┘           Any edit by Editor ──────┘
                                                auto-flags back to [needs_review]
```

**Visual status badges:**

| Status | Badge | Color |
|--------|-------|-------|
| `approved` | ✅ Approved | Green `#2E7D32` |
| `needs_review` | 🔄 Needs Review | Yellow `#F9A825` |
| `draft` | 🔴 Draft | Red `#C62828` |

### Role Permissions

| Action | Viewer | Editor | Admin |
|--------|:------:|:------:|:-----:|
| Read any page | ✓ | ✓ | ✓ |
| Search & filter by tag | ✓ | ✓ | ✓ |
| Edit page content | — | ✓ | ✓ |
| Add / rename pages | — | ✓ | ✓ |
| Add / reorder sections | — | ✓ | ✓ |
| Submit page for review | — | ✓ | ✓ |
| View version history | — | ✓ | ✓ |
| Approve / reject page | — | — | ✓ |
| Restore prior version | — | — | ✓ |
| Manage categories | — | — | ✓ |
| Manage users | — | — | ✓ |
| Delete pages | — | — | ✓ |

### Version History

Every save by an Editor or Admin writes a snapshot to `page_versions`:
- Full `sections` array captured as JSON
- `saved_by` user ID and `saved_at` timestamp
- Optional `change_note`

Version list UI shows: date/time, saved by, change note, View button (read-only), and Restore button (Admin only). Restore overwrites current sections, creates a new version entry noting the restore, and flags the page as `needs_review`.

### Authentication

**Phase 1 — Username + Password (JWT)**
- `POST /auth/login` → validates email/password, returns signed JWT (24-hour expiry)
- JWT stored in `localStorage`, sent as `Authorization: Bearer <token>` on all API calls
- Admin creates user accounts via admin panel — no self-registration
- `POST /auth/logout` → invalidates session record in DB

**Phase 2 option — Google SSO**
- Restrict to `@microsynergies.com` via Google OAuth 2.0
- Estimate: ~4–6 hrs additional work, quote separately

### Wiki API Routes

```
POST   /auth/login
POST   /auth/logout

GET    /pages                          list all pages (status, category, tags)
GET    /pages/:slug                    single page with sections
POST   /pages                          create page (Editor/Admin)
PUT    /pages/:id                      update page metadata (Editor/Admin)
DELETE /pages/:id                      delete page (Admin)

GET    /pages/:id/versions             version list
GET    /pages/:id/versions/:vid        single version snapshot
POST   /pages/:id/versions/:vid/restore  restore (Admin)

POST   /pages/:id/submit               submit for review (Editor/Admin)
POST   /pages/:id/approve              approve (Admin)
POST   /pages/:id/reject               reject to draft (Admin)

GET    /sections?page_id=X             sections for a page
POST   /sections                       create section
PUT    /sections/:id                   update section content
DELETE /sections/:id                   delete section
POST   /sections/reorder               update sort_order array

GET    /categories                     all categories with sort order
POST   /categories                     create (Admin)
PUT    /categories/:id                 rename / reorder (Admin)
DELETE /categories/:id                 delete (Admin)

GET    /admin/users                    user list (Admin)
POST   /admin/users                    create user (Admin)
PUT    /admin/users/:id                update role (Admin)
DELETE /admin/users/:id                delete user (Admin)

GET    /search?q=X&tags=Y,Z            full-text search across pages + sections
```

### Frontend (React SPA)

Three distinct views served from `wiki-client/dist/index.html`:

**Viewer Mode (all roles):** Category sidebar nav, page content rendered from section blocks, approval status badge, search bar with tag filter, clickable tag badges.

**Editor Mode (Editor/Admin):** All of Viewer plus Edit Page button, block editor (add/reorder/edit/delete sections by type), Submit for Review button, version history panel (read-only for Editors).

**Admin Panel (Admin only):** User management, category management, approval queue with approve/reject, version restore controls.

### Migration From Demo (v4)

The current demo has 12 pages across 5 categories. Migration plan:
1. Confirm final category structure with Brad and Dale (they want to review this)
2. Write a `seed.js` script to convert existing page content to DB rows
3. Seeded pages start as `draft` — Dale reviews and approves each one
4. Demo HTML is archived, not deleted

### Dale's Role

Dale is the designated Admin and primary content owner. His 2026 incentive goal maps directly onto this system:
- Tom builds the platform and seeds it with FAQ-derived content as a starting point
- Dale populates the ~90% of content that hasn't been documented yet
- Dale manages the approval workflow and owns category structure decisions
- A 30-minute category review session with Brad and Dale should happen before build starts

---

## Phase 2 — Document Library (Ask the Docs)

### Storage Strategy

PDFs live in an OneDrive shared folder maintained by the MicroSynergies team. The server connects via the OneDrive/SharePoint API and syncs on startup and on demand. This fits their existing setup and means Brad/Dale can add new documents without involving Tom.

Bridge option for Phase 1.5 (before OneDrive integration): a `documents/` folder committed directly to the repo. Suitable for a small, stable set of PDFs (under ~20).

```
knowledge-hub/
├── documents/               ← Phase 1.5 bridge: PDFs committed to repo
│   ├── novozymes-silage.pdf
│   └── ...
├── data/
│   └── document-manifest.json  ← Tracks file_id, summary, topics per document
```

### Document Manifest (document-manifest.json)

The manifest stores the Anthropic `file_id` for each PDF so documents are uploaded once and reused across all sessions — no re-uploading, no re-processing.

```json
{
  "documents": [
    {
      "filename": "novozymes-silage.pdf",
      "displayName": "Novozymes — Silage Inoculants White Paper",
      "fileId": "file_011abc...",
      "fileIdExpiresAt": "2026-06-16T00:00:00Z",
      "summary": "Covers inoculant strain selection, application rates, and efficacy data for corn and grass silage.",
      "topics": ["silage", "inoculants", "Novozymes", "corn", "grass"],
      "uploadedAt": "2026-06-09T12:00:00Z",
      "sizeKb": 840
    }
  ]
}
```

`file_id` values expire after 7 days — the server checks `fileIdExpiresAt` on startup and re-uploads any expired files automatically.

### Phase 2 Server Startup Flow

```
Server starts
        │
        ▼
Load document-manifest.json
        │
        ▼
For each document:
    ├── file_id present and not expired → reuse as-is
    └── file_id missing or expired → re-upload to Files API → update manifest
        │
        ▼
Server ready — all file_ids live in memory, manifest written to disk
```

### Phase 2 Ask the Docs Flow (Document Library Mode)

```
User opens Ask the Docs
        │
        ▼
GET /api/documents
→ Returns list: { filename, displayName, summary, topics }
        │
        ▼
User browses/filters library, selects a document
        │
        ▼
POST /api/ask { documentId, question }
→ file_id looked up from in-memory manifest (no upload needed)
→ beta.messages.create with existing file_id
        │
        ▼
Return { answer, inputTokens, outputTokens, sourceDocument }
```

### Token Efficiency Strategy

| Technique | Token Impact | When |
|-----------|-------------|------|
| Reuse `file_id` across sessions | High — eliminates redundant uploads | Phase 2 |
| Pre-generate document summaries | Medium — user picks the right doc before asking | Phase 2 |
| Store summaries in manifest (not re-generated) | Medium — one-time cost per document | Phase 2 |
| Filter by topic before querying | Medium — avoids querying wrong documents | Phase 2 |
| Chunk very large PDFs by section | High for 100+ page docs | Phase 3 if needed |

### Phase 2 New API Routes

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/documents` | List all library documents with display names, summaries, topics |
| POST | `/api/documents/sync` | Re-sync from OneDrive (or re-scan `documents/` folder) |
| POST | `/api/ask` | Updated: accepts `{ documentId, question }` in addition to `{ sessionId, question }` |

### Phase 3 — OneDrive Sync Flow

```
Admin triggers sync (or scheduled job runs)
        │
        ▼
OneDrive/SharePoint API — list files in shared folder
        │
        ▼
Compare against manifest
        │
        ├── New file → download → upload to Files API → generate summary → add to manifest
        ├── Removed file → delete from Files API → remove from manifest
        └── Unchanged file → check file_id expiry only
        │
        ▼
Manifest written to disk, server memory updated
```

New dependency: Microsoft Graph API (`@microsoft/microsoft-graph-client`). Service account credentials stored in `.env` — never committed.

---

## Ownership Model

| Responsibility | Owner |
|----------------|-------|
| Platform build and structure | Tom (DataForge) |
| Ask the Docs — initial setup and deployment | Tom (DataForge) |
| Wiki — initial content population (from FAQ doc) | Tom (DataForge) |
| Wiki — ongoing content updates | Dale (Admin) |
| Category structure decisions | Dale + Brad |
| Technical maintenance | Tom (retainer) |
| Anthropic API key (production) | MicroSynergies — console.anthropic.com |

---

## Hosting & Deployment

| Resource | Detail |
|----------|--------|
| Platform | Railway |
| Service | Single Node.js service |
| Persistent volume | `/data/` — SQLite DB + uploaded images + document manifest |
| Estimated cost | ~$5–10/month |
| Domain | `hub.microsynergies.com` (CNAME pending from MicroSynergies IT) |
| Deploy trigger | Push to GitHub `main` → Railway auto-deploys |
| First-run | `db/init.js` creates schema + seeds initial Admin user |

---

## Open Questions (Resolve Before Build)

- [ ] Confirm final category list and top-level navigation structure with Brad/Dale
- [ ] Confirm number of initial user accounts needed and role assignments
- [ ] OneDrive link behavior: publicly shared (anyone with link) or requires MS login?
- [ ] Dale's starting role: Editor or Admin? (Admin recommended given incentive goal ownership)
- [ ] Railway: keep wiki on same service as Ask the Docs, or separate project?

---

## Phase Roadmap

| Phase | Module | Scope | Est. Effort |
|-------|--------|-------|-------------|
| Phase 1 | Wiki | CMS platform: DB, auth, all block types, approval workflow, version history, search | 40–59 hrs |
| Phase 1 | Ask the Docs | Core build (already scoped separately) | 12–18 hrs |
| Phase 2 | Wiki | Google SSO, email notifications on approval | 7–11 hrs |
| Phase 2 | Ask the Docs | Document library + OneDrive sync | 10–15 hrs |
| Phase 3 | Both | AI search layer embedded in wiki + Ask the Docs integration | 8–12 hrs |
| Retainer | Both | Hosting oversight, bug fixes, minor updates | $200–350/month |
