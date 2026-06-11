---
name: end-day
description: >
  End-of-day handoff for Lucas's dev environment. Commits and pushes all
  in-progress work across every project, then updates each active project's
  JOURNAL.md with a dated session summary so the next session (on any machine)
  can pick up without context loss. Trigger when Lucas says "end my day",
  "wrap up", "I'm done for today", "end session", or similar.
allowed-tools: Bash, PowerShell, Read, Edit, Write, Glob
---

# End-of-Day Handoff

When invoked, execute these steps in order. Be terse — report findings, not narration.

---

## Step 1: Discover projects and check status

Discover all git repos directly under `E:\Code\`:

```powershell
Get-ChildItem E:\Code -Directory | Where-Object { Test-Path "$($_.FullName)\.git" }
```

For each repo found, run:
```powershell
cd E:\Code\<project>
git status --short
git log --oneline --since=midnight
```

Collect: (a) which projects have uncommitted changes, (b) which projects had commits today.

---

## Step 2: Pull remotes

For every project that has a remote, pull the **current branch** before touching anything:
```powershell
git pull --rebase origin <current-branch>
```

Never pull or touch `main` — that branch is only updated via explicit merge when work is complete.

If a pull fails (conflict, diverged history), **stop for that project**, report it, and ask Lucas what to do. Do not commit or push until resolved.

---

## Step 3: Commit uncommitted work

For every project with uncommitted changes:

1. `git add -A`
2. Ask Lucas for a commit message OR use `"wip: end of day"` if he says to just do it.
3. `git commit -m "<message>"`

---

## Step 4: Push all branches

Push the current working branch only — never push `main` as part of this flow:
```powershell
git push origin <current-branch>
```

Report any failures clearly (auth issues, no remote, diverged branch). If a project has no remote, note it and skip.

---

## Step 5: Update JOURNAL.md for active projects

For each project that had **any commits today**:

### 4a. Read recent context
- Read the last 30 lines of the project's existing JOURNAL.md (if it exists)
- Run `git log --oneline --since=midnight` to get today's commits
- Run `git diff HEAD~<n>..HEAD --stat` where n = number of today's commits, to see what files changed

### 4b. Ask Lucas one question per active project
"For [project]: anything to note for next session beyond what's in the commits? (or skip)"

Keep it brief — this is a prompt for things git can't capture: half-finished ideas, gotchas discovered, what to tackle first next time.

### 4c. Write the journal entry

Append to `E:\Code\<project>\JOURNAL.md` (create if missing):

```markdown
## Session [YYYY-MM-DD]

### Done
- [bullet per meaningful commit or change, written as outcomes not file names]

### State
[one sentence on where things stand — what's working, what's pending]

### Next session
- [bullet per thing to pick up, ordered by priority]

[Lucas's freeform notes if he gave any]
```

Rules for the entry:
- "Done" bullets are outcomes ("fixed wrong workout day bug") not diffs ("edited TodayWorkout.tsx")
- "Next session" bullets are actionable starting points, not vague intentions
- If nothing interesting happened (only a checkpoint commit with no real work), write a one-liner: `## Session [date] — no significant changes` and skip the rest
- Keep each entry under 15 lines

---

## Step 6: Report summary

Print a single table:

```
Project            Branch               Committed   Pushed   Journal
WorkoutWebHelper   wip/workout-tracker  ✓           ✓        updated
...
```

Then: "Done. Safe to close."

If anything failed (push rejected, no remote, etc.) list it clearly after the table so Lucas can act on it.

---

## Notes

- On Windows, use PowerShell for git commands.
- Never push to a remote that doesn't exist yet — just note it.
- Don't update JOURNAL.md if the only commit was the end-of-day checkpoint with no real work behind it.
- If Lucas is in a hurry and says "just do it", skip the freeform notes question and proceed.
