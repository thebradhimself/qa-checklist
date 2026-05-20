---
description: "Generate a branch-aware interactive QA checklist (HTML) from the current git diff and optional PR metadata."
argument-hint: "[--base main] [--pr] [--output tmp/qa-checklist.html] [--focus ui|backend|regression|security|perf|a11y] [--include-low-priority] [--json] [--summary]"
allowed-tools: "Bash(git:*), Bash(gh:*), Bash(mkdir:*), Bash(ls:*), Bash(test:*), Bash(date:*), Read, Write, Glob, Grep"
---

# /qa-checklist — Branch-aware QA checklist

You are generating a **branch-specific** interactive QA checklist as a single self-contained HTML file. Every item must trace back to actual changes on this branch. Do not produce generic filler.

Raw arguments: `$ARGUMENTS`

## Argument parsing

Parse from `$ARGUMENTS`. Defaults:

- `--base <branch>` — base branch (default: remote default → `main` → `master`)
- `--pr` — also pull PR metadata via `gh` if available; otherwise silently skip
- `--output <path>` — output HTML path (default: `tmp/qa-checklist.html`)
- `--focus <area>` — bias toward an area; repeatable. Values: `ui`, `backend`, `regression`, `security`, `perf`, `a11y`
- `--include-low-priority` — include `low`-priority items (off by default)
- `--json` — also write `tmp/qa-checklist-data.json`
- `--summary` — also write `tmp/qa-checklist-summary.md`

## Rules of engagement

- **Read-only against the repo.** Run only safe git/gh commands: `rev-parse`, `branch`, `log`, `diff`, `show`, `remote show`, `fetch --quiet`, `gh pr view`, `gh pr diff`. Never stage, commit, push, reset, checkout, rebase, or merge.
- **Do not modify application code.** Only write the QA artifact files.
- Use the repo's conventions where obvious.
- If any tool warns the GitNexus index is stale and the repo declares GitNexus in `CLAUDE.md`, mention it in the final report but do not run `analyze` automatically.

## Step 1 — Detect branch state

```bash
git rev-parse --abbrev-ref HEAD
git rev-parse --short HEAD
git remote show origin 2>/dev/null | sed -n 's/.*HEAD branch: //p'
git fetch origin --quiet || true
git log --oneline "<base>..HEAD"
git diff --stat "<base>...HEAD"
git diff --name-status "<base>...HEAD"
```

Resolve `<base>`:
1. `--base` flag if given.
2. Else remote default branch.
3. Else first of `main`, `master` that exists locally or on origin.

For each meaningfully changed file (cap ~40 files; prioritize routes, controllers, models, views, migrations, jobs, mailers, policies, components, configs, package manifests), inspect the actual diff hunks with `git diff "<base>...HEAD" -- <file>`.

If `--pr` and `gh` exists and the branch has a PR:
```bash
gh pr view --json number,title,body,headRefName,baseRefName,files
gh pr diff
```
Fold PR title/body into context. Do not include private PR comments unless they are clearly QA-relevant.

## Step 2 — Detect framework and conventions

Detect from these signals:

| Signal | Framework |
|---|---|
| `Gemfile`, `config/routes.rb`, `app/controllers/` | Rails |
| `package.json` + `next.config.*` | Next.js |
| `package.json` + `react` dep | React |
| `vue.config.*` or `vite.config.*` + `vue` | Vue |
| `composer.json` + `artisan` | Laravel |
| `manage.py` + `settings.py` | Django |
| `package.json` + `express` | Node/Express |
| `ios/`, `android/`, `App.tsx` | Mobile (React Native / native) |

For **Rails** branches, also detect and weight checks on: `app/controllers/`, `app/models/`, `app/views/**/*.{slim,erb}`, `app/helpers/`, `app/policies/`, `db/migrate/`, `app/jobs/`, `app/mailers/`, `app/javascript/controllers/` (Stimulus), Turbo Frame/Stream targets in views, `test/` or `spec/`. Look for strong params, validations, callbacks, scope changes, removed/added `includes`/`preload` (N+1 risk), and migrations needing backfill or non-reversible changes.

## Step 3 — Build the item data model

Build a JSON array. Each item:

```json
{
  "id": "qa-001",
  "title": "Concrete user-observable action",
  "description": "Why this matters for THIS branch",
  "category": "ui | forms | auth | data | api | jobs | notifications | edge | regression | responsive | a11y | perf | security | migrations | errors",
  "priority": "critical | high | medium | low",
  "status": "untested",
  "steps_to_test": ["Numbered, role-aware, action-by-action"],
  "expected_result": "Observable outcome",
  "related_files": ["app/controllers/x_controller.rb:42", "db/migrate/2026..._add_y.rb"],
  "notes": "",
  "failure_details": "",
  "suggested_fix_prompt_fragment": "One sentence telling Claude what to do if this fails"
}
```

Quality bar — items MUST look like:

- "Log in as an admin, navigate to Settings → Billing, change plan name to 'Pro Annual', save; verify the new name appears on the billing overview without a page refresh."
- "Attempt the same plan-name update as a non-admin and verify a 403 (or redirect) is returned and no `Plan` record is modified."
- "Submit the form with `name` blank and verify inline validation error appears and no `Plan` record is created."

Forbidden vague items: "test the page works", "make sure nothing breaks", "verify it looks good".

Rules:
- If a category has no real change on this branch, omit it entirely.
- If `--include-low-priority` not set, drop `low` items from output.
- If `--focus` set, weight selected areas (include more depth there) but keep all `critical`/`high` items regardless of focus.
- Cap at ~40 items unless the diff is genuinely large. Prefer fewer, sharper items.
- For Rails migrations: always include a `migrations` item covering forward run, rollback (`db:rollback`), data integrity check, and any required backfill.
- For controller changes: include both happy-path AND authorization-denied paths.
- For Turbo Stream broadcasts: include "verify the DOM updates without a full page reload."

## Step 4 — Render the HTML from the template

The HTML/CSS/JS lives in a template asset shipped with this plugin. Do not inline or rewrite it.

1. **Read** `${CLAUDE_PLUGIN_ROOT}/assets/qa-checklist.html.tmpl` verbatim with the Read tool.
2. **Substitute** the following tokens (string replacement only; no other edits to the template):

   | Token | Replacement |
   |---|---|
   | `__BRANCH__` | current branch name |
   | `__BASE__` | resolved base branch |
   | `__COMMIT__` | short HEAD sha |
   | `__GENERATED_AT__` | ISO 8601 timestamp (UTC, e.g. `2026-05-20T14:32:11Z`) |
   | `__ITEMS_JSON__` | the JSON array from Step 3, with every `</` rewritten as `<\/` so the inline `<script>` block stays valid |
   | `__FOCUS__` | JSON array of focus strings, e.g. `["ui","regression"]` or `[]` |

3. **Write** the substituted result to `--output` (default `tmp/qa-checklist.html`). Run `mkdir -p` on the parent directory first.

Do not modify the template's HTML structure, styles, or JavaScript. If the template is missing, abort and report — do not improvise a replacement.

## Step 5 — Optional artifacts

- `--json` → write the same `{meta, items}` payload to `tmp/qa-checklist-data.json`.
- `--summary` → write a short markdown digest to `tmp/qa-checklist-summary.md`:
  - title with branch and commit
  - top 5 critical/high items
  - file groups (controllers / models / migrations / views / jobs / other)
  - one-paragraph "what this branch appears to do"

## Step 6 — Report back to the user

Print:
1. Output path(s).
2. Counts: total items, by category, by priority.
3. One-paragraph branch summary ("Rails branch; 4 controllers, 2 migrations, 1 mailer changed; generated 22 QA items").
4. How to open: `open tmp/qa-checklist.html` (or platform-appropriate command).
5. Note that progress is auto-saved to `localStorage` keyed by branch+commit; resetting requires the Reset button.
