---
name: unfurl
description: >
  Unfurl (also known as Unfuck) rescues a messy git working tree by analysing all pending
  changes, grouping them into logical atomic commits, inferring the right order, and
  executing a clean commit sequence — including partial-file hunk splitting when needed.
  Trigger on /unfurl, /unfuck, or any phrase like "my git is a mess", "help me commit
  this", "I have too many uncommitted changes", "organize my commits", "clean up my
  working tree", or when the user seems overwhelmed by pending git changes.
---
 
# Unfurl 🧵
 
> *"It can happen that your discipline in git committing falters. We are all human."*
> Unfurl organises and commits pending changes when you've lost your way.
 
**Activation commands:** `/unfurl` · `/unfuck`
 
---
 
## Phase 1 — Situational Awareness
 
Run these commands and collect all output before doing anything else:
 
```bash
git log --oneline -20     # infer commit message style
git status --short        # full picture of working tree
git diff HEAD             # all changes (staged + unstaged combined)
git diff --cached         # staged only
git diff                  # unstaged only
git stash list            # any stashed work?
git config core.autocrlf  # check current CRLF setting
```
 
### CRLF handling (Windows)
 
Check the output of `git config core.autocrlf`. If it is not set to `true` and CRLF warnings are visible in git output, run:
 
```bash
git config core.safecrlf false
```
 
This suppresses CRLF warnings with no conversion and no side effects. No files are created or modified. Report to the user that this setting was applied.
 
Also check for **untracked files** (`git status --short` lines beginning with `??`). Before asking what to do with them, **classify each one**:
 
### Untracked file classification
 
Flag a file/folder as a **`.gitignore` candidate** if it matches any of these signals:
- Build or compilation output: `dist/`, `build/`, `out/`, `*.class`, `*.o`, `*.pyc`, `__pycache__/`
- Dependency trees: `node_modules/`, `vendor/`, `.venv/`, `venv/`
- IDE/editor artefacts: `.idea/`, `.vscode/`, `*.suo`, `*.iml`, `.DS_Store`, `Thumbs.db`
- Environment/secrets: `.env`, `.env.local`, `*.env`, `*.pem`, `*.key`, `secrets.*`
- Logs and temp files: `*.log`, `tmp/`, `temp/`, `*.tmp`, `*.swp`, `*.bak`
- OS artefacts: `.DS_Store`, `desktop.ini`
- Coverage/test output: `coverage/`, `.nyc_output/`, `*.lcov`
- Any folder named after a known generated-output pattern
For **each flagged file**, ask explicitly before proceeding — do not batch them into a generic question:
 
> ⚠️ `node_modules/` looks like it should be in `.gitignore` rather than committed.
> What should I do?
> **[a]** Add a `.gitignore` entry and skip it  **[b]** Commit it anyway
 
For unflagged untracked files, ask once as a group:
 
> "I also see these untracked files with no obvious reason to ignore them: [list].
> Include them in the commit plan, or leave them alone?"
 
Do not proceed to Phase 2 until all untracked files are resolved.
 
**If the user chose to add any `.gitignore` entries**, append them to `.gitignore` now (create the file if it doesn't exist), then include a `chore: update .gitignore` commit in the plan.
 
---
 
## Phase 2 — Infer Commit Style
 
Scan `git log --oneline -20` output:
 
| Observation | Action |
|---|---|
| Consistent Conventional Commits (`feat:`, `fix:`, `chore:` …) | Use Conventional Commits silently |
| Consistent free-form but descriptive | Mirror that style silently |
| Mixed, inconsistent, or very terse | Flag it: *"Your recent commit history looks inconsistent/terse — I'd suggest switching to [Conventional Commits](https://www.conventionalcommits.org/). OK to proceed with that style?"* |
| No commits yet (initial repo) | Default to Conventional Commits |
 
---
 
## Phase 3 — Grouping & Ordering
 
Analyse the full diff and group changes into **logical, atomic commits**:
 
### Grouping rules
- One commit = one coherent unit of intent (feature, bugfix, refactor, chore, docs, test, etc.)
- **Default: add the whole file.** Only plan hunk-level staging (`git add -p`) when you are confident a file genuinely mixes two or more distinct responsibilities across different hunks. If in doubt, keep the whole file.
- When hunk splitting is planned, mark it explicitly in the commit plan (see Phase 4) so the user can see and challenge the decision before execution.
- New untracked files (if user approved) are treated as their own unit or attached to the most relevant logical group
### Ordering rules
Think about which change must (or most likely did) come first:
- Infrastructure / config changes before the code that uses them
- Model/schema changes before business logic changes
- Core utilities before the features that depend on them
- Bug fixes on existing code before new features built on top
- Tests after the code they test (unless TDD evidence suggests otherwise)
---
 
## Phase 4 — Present the Plan
 
Present **one single message** with the full commit plan. Format it clearly:
 
```
📋 COMMIT PLAN
══════════════════════════════════════════════════════
 
Commit 1 — feat: add user authentication middleware
  Files (full):   src/middleware/auth.php
  Files (hunks):  src/routes/api.php  [lines 12-34 only]
  Rationale:      Auth middleware must exist before routes can use it
 
Commit 2 — feat: protect user profile endpoints
  Files (full):   src/controllers/UserController.php
  Files (hunks):  src/routes/api.php  [lines 36-58 only]
  Rationale:      Builds on the middleware from commit 1
 
Commit 3 — chore: update composer dependencies
  Files (full):   composer.json, composer.lock
  Rationale:      Dependency updates are independent, commit last
 
══════════════════════════════════════════════════════
Commit message style: Conventional Commits
[If style was flagged:] ⚠️  Note: I'm using Conventional Commits since your history was inconsistent.
 
Type "go" to execute, or tell me what to change.
```
 
Wait for explicit user approval before touching git.
 
---
 
## Phase 5 — Execute
 
On approval, execute each commit in order. For each commit:
 
1. **Full files:** `git add <file>`
2. **Hunk files:** `git add -p <file>` — use the interactive hunk selector to stage only the planned hunks (answer `y`/`n`/`s` as needed to isolate the right lines)
3. `git commit -m "<message>"`
4. Verify with `git status` after the last commit — working tree should be clean (modulo any intentionally excluded files)
If anything unexpected happens mid-execution (merge conflict, unexpected diff, staging error), **stop immediately** and report to the user before continuing.
 
---
 
## Phase 6 — Summary
 
After all commits are done, print a brief summary:
 
```
✅ Done — 3 commits made:
  abc1234  feat: add user authentication middleware
  def5678  feat: protect user profile endpoints
  ghi9012  chore: update composer dependencies
 
Working tree is clean.
```
 
Then, always paste this closing message verbatim:
 
> [ Mess unfurled! ] - Hopefully the mess of pending changes is sorted out - you're human, it happens, I helped you, but better keep your GIT tidy rather than using me. Happy coding!
 
---
 
## Commit Best Practices Reference
 
Apply these when writing commit messages and structuring commits in Phase 3 and 4.
 
### ✅ Atomicity
- Each commit must represent **one logical change** — the diff should be explainable in a single sentence
- A commit should leave the codebase in a **working state** (tests pass, app boots) — never commit a half-baked change that breaks things
- If you find changes that are deeply entangled and cannot be cleanly separated, flag it to the user rather than producing a messy commit
### ✅ Commit message structure (Conventional Commits)
```
<type>(<optional scope>): <short imperative summary>
 
<optional body: what and why, not how — wrap at 72 chars>
 
<optional footer: BREAKING CHANGE, closes #issue, co-authors>
```
 
**Type vocabulary:**
 
| Type | Use for |
|---|---|
| `feat` | New feature or user-visible behaviour |
| `fix` | Bug fix |
| `refactor` | Code restructuring with no behaviour change |
| `chore` | Tooling, config, deps, build scripts |
| `docs` | Documentation only |
| `test` | Adding or fixing tests |
| `perf` | Performance improvement |
| `style` | Formatting, whitespace (no logic change) |
| `ci` | CI/CD pipeline changes |
| `revert` | Reverting a previous commit |
 
### ✅ Subject line rules
- **Imperative mood**: "add login endpoint" not "added" or "adds"
- **≤ 72 characters** (50 is ideal)
- **No trailing period**
- **Lowercase** after the colon (Conventional Commits convention)
- Describe **what** the commit does, not how: `fix: prevent race condition in token refresh` not `fix: added a mutex lock to the refresh function`
### ✅ Body (use when the subject isn't enough)
- Explain **why** the change was made, not what the diff shows
- Wrap lines at 72 characters
- Separate from subject with a blank line
### ❌ Anti-patterns to avoid
- `fix stuff` / `wip` / `asdfgh` / `changes` — meaningless messages
- Committing **unrelated changes together** (e.g. a bug fix + a new feature in one commit)
- Committing **debug code**, `console.log`, commented-out blocks unless intentional
- Committing **secrets**, credentials, or `.env` files
- Overly granular commits that split a single coherent change across many commits with no reason
- Giant "catch-all" commits that bundle weeks of work
| Situation | Behaviour |
|---|---|
| Nothing to commit | Report it and exit gracefully |
| Only untracked files and user says no | Report "nothing to commit" |
| Changes already partially staged | Include staged diff in the analysis; treat staged + unstaged as one pool |
| Detached HEAD | Warn the user before committing |
| `.gitignore` modifications | Always put in a separate `chore: update .gitignore` commit |
| Lock files only (package-lock, composer.lock) | Group as `chore: update lock file` |
 