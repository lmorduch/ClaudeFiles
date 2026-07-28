# CLAUDE.md — Lucas's Dev Environment

## Who you're working with

Lucas (lmorduchowicz / lmorduch@gmail.com). Solo developer building personal tools and a game.
Direct communicator. No sycophancy. Push back when something is a bad idea.
**Just do it** — don't ask for confirmation on obvious follow-ups.

Physical constraints: can't do hanging leg raises or barbell bicep curls (wrist issues).
Home gym: barbell, cable machine, dumbbells up to 55 lbs.

---

## Workspace layout

All work lives inside this repo (`ClaudeFiles/`). Projects are checked out under `projects/` which is gitignored — only skills, CLAUDE.md, and config are tracked here.

```
ClaudeFiles/
  CLAUDE.md
  skills/
  projects/          ← gitignored; clone each project repo here
```

On a new machine: clone ClaudeFiles to a consistent path, then clone each project into `projects/`.

## Projects

| Dir | What it is | Stack |
|-----|------------|-------|
| `projects/WorkoutWebHelper/` | Personal workout tracker (Walrus Workout Buddy) | FastAPI + React/TS + SQLite → Railway |
| `projects/Todo/` | Today & Onward todo app | Node/Express + React/Vite + Postgres → Railway |
| `projects/Budget/` | Household budget tracker (shared with Wayo) | Node/Express + React/TS/Vite + Tailwind + Postgres + Plaid → Railway. Default branch is `master`, not `main`. |
| `projects/Groceries/` | Grocery list app | FastAPI + React → Railway |
| `projects/Plants/` | Plant tracker | FastAPI + React → Railway |
| `projects/wizard_soccer/` | Witnessed League — Godot 4 roguelike soccer game | GDScript, no runtime in this environment |
| `projects/Boys/` | — | — |
| `projects/Meds/` | — | — |
| `projects/comic-bookkeeper/` | — | — |
| `projects/tattoo/` | Tattoo artist booking tracker — daily Instagram scraping + email alerts | FastAPI + React/TS + Postgres → Railway |

---

## Tech stack defaults

**Backend**: FastAPI (Python) or Node/Express — depends on project. Check before assuming.
**Frontend**: React + Vite. TypeScript for WorkoutWebHelper, JS for Todo. Single `App.css`, dark theme, mobile-first, bottom nav.
**Auth**: Google OAuth everywhere. JWT cookie (Python projects) or Passport sessions (Node projects).
**DB**: SQLite for workout/plants/groceries. Postgres for Todo. Schema migrations run at startup — `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` pattern for SQLite.
**Deploy**: Railway, always. Dockerfile for Python projects. `railway.json` for Node. Single service — frontend built and served by backend.

No test framework in frontend (WorkoutWebHelper confirmed). No Redux.

---

## Railway deployment

**Check first whether the project is GitHub-integrated** (Budget, WorkoutWebHelper) — for those, pushing the deploy branch is the whole deploy and `railway up` is wrong. See Branching + deploy below.

For the rest:
```bash
railway up          # CORRECT — rebuilds Docker image and deploys
railway service restart  # WRONG — does not rebuild, use only if image is already current
```

Ctrl+C after "Uploaded" is fine — build continues on Railway.

### Checklist for a new Node project on Railway

**railway.toml** — always include this:
```toml
[build]
builder = "nixpacks"
buildCommand = "npm install && npm run build"

[deploy]
startCommand = "node server/index.js"
healthcheckPath = "/health"
healthcheckTimeout = 60
restartPolicyType = "on_failure"

[environments.production.variables]
NODE_ENV = "production"
```

**package.json** — pin Node version or Railway defaults to Node 18:
```json
"engines": { "node": ">=22" }
```

**Healthcheck endpoint** — always add a dedicated `/health` route that returns 200. Never use `/api/me` or any auth-gated route — Railway requires 2xx and auth routes return 401, which Railway treats as unhealthy.
```js
app.get('/health', (_req, res) => res.json({ ok: true }))
```

**Database** — SQLite doesn't survive Railway deploys (ephemeral filesystem). Use Postgres for anything that needs to persist. Railway provisions it automatically when you add a Postgres service to the project.

**DATABASE_URL** — Railway does NOT auto-inject Postgres vars into other services. In the Budget service → Variables, add:
```
DATABASE_URL=${{Postgres.DATABASE_URL}}
```

**Trust proxy** — Railway terminates SSL at the load balancer, so Express sees HTTP internally. Without this, `express-session` with `secure: true` never sets the cookie and sessions don't persist after OAuth. Add before any middleware:
```js
app.set('trust proxy', 1)
```

**Google OAuth env vars** needed at deploy time:
```
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_CALLBACK_URL=https://<railway-domain>/auth/google/callback
ALLOWED_EMAILS=email1@gmail.com,email2@gmail.com
SESSION_SECRET=<openssl rand -hex 32>
```
OAuth redirect URIs must be registered in Google Cloud Console for the Railway domain. Forgetting this is a common auth failure on first deploy.

### npm on HubSpot machines (affects all personal Node projects)

The global `~/.npmrc` sets `registry = https://npm.hubteam.com/` which bleeds into
`package-lock.json` via `resolved` URLs. Railway can't reach `npm.hubteam.com`.

**Fix for every new Node project:**
1. Add a project-level `.npmrc`: `registry=https://registry.npmjs.org/`
2. Use `npm install` in `buildCommand`, not `npm ci` — `npm ci` has an "exit handler never called" bug on Node 22 in Docker
3. After generating the lockfile locally: `sed -i '' 's|https://npm.hubteam.com/|https://registry.npmjs.org/|g' package-lock.json`

### Express 5 breaking changes

- Wildcard routes: `'*'` is invalid — use `'/{*path}'` instead
- Async route handlers propagate thrown errors automatically (no need for try/catch + next(err))

### Scrubbing sensitive data from git history

```bash
pip3 install git-filter-repo --break-system-packages
git filter-repo --path <FILE> --invert-paths --force
# filter-repo removes the remote; re-add it after:
git remote add origin git@github.com:lmorduch/<REPO>.git
```

Always gitignore JOURNAL.md and any file with real financial/personal data before pushing to a remote.

---

### WorkoutWebHelper
- **One DB rule**: only ever use `backend/workout.db`. Never set `DB_PATH=workout_railway.db` — that created a second copy and caused session confusion.
- **ExerciseDB API**: `gifUrl` field was removed. Use `/image?exerciseId=X` endpoint instead.
- **Seeding**: seed scripts skip if the program name exists. To force re-seed: `DELETE FROM programs WHERE name='...';` first.
- **Wrong workout day bug (fixed)**: was using a hardcoded `DAY_TO_WORKOUT` map. Always pass `workout.workout_day` explicitly — don't derive from day-of-week.
- **Feedback button**: no GitHub repo exists publicly. Use in-app feedback modal → `POST /api/feedback` → `feedback` table.
- Active program: Zing Coach (program_run_id=2, start_date=2026-06-01). Lucas uses Cable Curls not Barbell Bicep Curls, and Cable Tricep Pushdowns not Dips.

### Godot (wizard_soccer)
- No local Godot binary. Code is syntax-reviewed, not runtime-verified. Say so explicitly.
- GDScript `match` is top-to-bottom — duplicate arms silently dead-code. Check for this.
- GDScript enum aliases (e.g. `O = Wizard.Origin`) make invalid members silently resolve to 0. Always validate against the actual enum definition.
- Editor-only wiring (Autoloads, node references in .tscn) cannot be done from code alone — flag these for Lucas to do in the editor.

---

## GitHub accounts

Lucas has two GitHub accounts: `lmorduch` (personal) and `lmorduchowicz_hubspot` (work). The active `gh` account can drift to the HubSpot account mid-session.

For all personal project git/gh operations, use:
```bash
gh-personal <args>   # gh CLI forced to lmorduch account
```

For git push from personal repos, set the remote URL with the username to avoid credential confusion:
```bash
git remote set-url origin https://lmorduch@github.com/lmorduch/<REPO>.git
```

`gh-personal` is defined in `~/.zshrc` and sets `GH_TOKEN` from the stored `lmorduch` token for the duration of each call.

---

## Branching + deploy

- All work happens on a branch. Never commit directly to `main`.
- `main` is the deploy branch for every project. Only merge into it when a feature/fix is complete and ready to ship.
- Branch naming: `wip/<short-description>` for ongoing work, `fix/<short-description>` for bugfixes.
- Deploy is `railway up` from local filesystem (not triggered by git push) — for most projects.
- **Exceptions — GitHub-integrated, the push IS the deploy (no `railway up`):**
  - **Budget** tracks `master` — `git push origin master` builds and releases to production.
  - **WorkoutWebHelper** tracks `main` — `git push origin main` builds and releases to production.

Commit frequently on the working branch.

---

## Context handoff docs

Each project maintains its own handoff/journal files. Check these before starting work:
- `projects/WorkoutWebHelper/CONTINUATION.md` — architecture, data model, open tasks
- `projects/WorkoutWebHelper/JOURNAL.md` — session-by-session history and pending TODOs
- `projects/WorkoutWebHelper/HANDOFF.md` — Lucas's preferences, program state, common operations
- `projects/wizard_soccer/JOURNAL.md` — architecture decisions, known bugs, session log
- `projects/wizard_soccer/DESIGN.md` — full game design document
- `projects/wizard_soccer/SESSION_DECISIONS.md` — rationale for non-obvious choices

Read the relevant doc before proposing anything — many "obvious" approaches have already been tried and rejected with documented reasons.
