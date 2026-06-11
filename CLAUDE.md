# CLAUDE.md — Lucas's Dev Environment

## Who you're working with

Lucas (lmorduchowicz / lmorduch@gmail.com). Solo developer building personal tools and a game.
Direct communicator. No sycophancy. Push back when something is a bad idea.
**Just do it** — don't ask for confirmation on obvious follow-ups.

Physical constraints: can't do hanging leg raises or barbell bicep curls (wrist issues).
Home gym: barbell, cable machine, dumbbells up to 55 lbs.

---

## Projects

| Dir | What it is | Stack |
|-----|------------|-------|
| `WorkoutWebHelper/` | Personal workout tracker (Walrus Workout Buddy) | FastAPI + React/TS + SQLite → Railway |
| `Todo/` | Today & Onward todo app | Node/Express + React/Vite + Postgres → Railway |
| `Groceries/` | Grocery list app | FastAPI + React → Railway |
| `Plants/` | Plant tracker | FastAPI + React → Railway |
| `wizard_soccer/` | Witnessed League — Godot 4 roguelike soccer game | GDScript, no runtime in this environment |

---

## Tech stack defaults

**Backend**: FastAPI (Python) or Node/Express — depends on project. Check before assuming.
**Frontend**: React + Vite. TypeScript for WorkoutWebHelper, JS for Todo. Single `App.css`, dark theme, mobile-first, bottom nav.
**Auth**: Google OAuth everywhere. JWT cookie (Python projects) or Passport sessions (Node projects).
**DB**: SQLite for workout/plants/groceries. Postgres for Todo. Schema migrations run at startup — `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` pattern for SQLite.
**Deploy**: Railway, always. Dockerfile for Python projects. `railway.json` for Node. Single service — frontend built and served by backend.

No test framework in frontend (WorkoutWebHelper confirmed). No Redux.

---

## Deployment

```bash
railway up          # CORRECT — rebuilds Docker image and deploys
railway service restart  # WRONG — does not rebuild, use only if image is already current
```

Ctrl+C after "Uploaded" is fine — build continues on Railway.

**WorkoutWebHelper DB sync:**
```bash
./db_pull.sh   # Railway → local (backs up first)
./db_push.sh   # local → Railway + redeploy (prompts)
```

**OAuth redirect URIs** must be registered for all three: `localhost`, `192.168.50.81.nip.io` (local network), and the Railway domain. Forgetting the Railway domain is a common auth failure after first deploy.

---

## Known pitfalls

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

## Branching + deploy

- All work happens on a branch. Never commit directly to `main`.
- `main` is the deploy branch for every project. Only merge into it when a feature/fix is complete and ready to ship.
- Branch naming: `wip/<short-description>` for ongoing work, `fix/<short-description>` for bugfixes.
- Deploy is `railway up` from local filesystem (not triggered by git push).

Commit frequently on the working branch.

---

## Context handoff docs

Each project maintains its own handoff/journal files. Check these before starting work:
- `WorkoutWebHelper/CONTINUATION.md` — architecture, data model, open tasks
- `WorkoutWebHelper/JOURNAL.md` — session-by-session history and pending TODOs
- `WorkoutWebHelper/HANDOFF.md` — Lucas's preferences, program state, common operations
- `wizard_soccer/JOURNAL.md` — architecture decisions, known bugs, session log
- `wizard_soccer/DESIGN.md` — full game design document
- `wizard_soccer/SESSION_DECISIONS.md` — rationale for non-obvious choices

Read the relevant doc before proposing anything — many "obvious" approaches have already been tried and rejected with documented reasons.
