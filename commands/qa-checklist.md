---
description: Generate a branch-aware interactive QA checklist (HTML) from the current git diff and optional PR metadata.
argument-hint: [--base main] [--pr] [--output tmp/qa-checklist.html] [--focus ui|backend|regression|security|perf|a11y] [--include-low-priority] [--json] [--summary]
allowed-tools: Bash(git:*), Bash(gh:*), Bash(mkdir:*), Bash(ls:*), Bash(test:*), Bash(date:*), Bash(rg:*), Read, Write, Glob, Grep
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

## Step 4 — Write the HTML

Create parent dir with `mkdir -p` then write to `--output` (default `tmp/qa-checklist.html`). Use the template in `## HTML template` below. Substitute:

- `__BRANCH__` — current branch
- `__BASE__` — resolved base branch
- `__COMMIT__` — short HEAD sha
- `__GENERATED_AT__` — ISO 8601 timestamp
- `__ITEMS_JSON__` — the JSON array. Escape any `</` inside as `<\/` to keep the inline `<script>` valid.
- `__FOCUS__` — JSON array of focus strings, e.g. `["ui","regression"]` or `[]`

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

---

## HTML template

Write the file below verbatim with the substitutions from Step 4.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>QA Checklist — __BRANCH__</title>
<style>
  :root { --bg:#0f1115; --panel:#171a21; --border:#262b36; --text:#e6e8ee; --muted:#9aa3b2;
    --accent:#5b8def; --pass:#34c759; --fail:#ff453a; --skip:#8e8e93; --review:#ffd60a;
    --crit:#ff453a; --high:#ff9f0a; --med:#ffd60a; --low:#8e8e93; }
  @media (prefers-color-scheme: light) {
    :root { --bg:#f7f8fa; --panel:#fff; --border:#e3e6ec; --text:#1a1d24; --muted:#5b6273; }
  }
  *{box-sizing:border-box}
  body{margin:0;font:14px/1.5 -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif;background:var(--bg);color:var(--text)}
  header{padding:18px 24px;border-bottom:1px solid var(--border);background:var(--panel);position:sticky;top:0;z-index:10}
  header h1{margin:0 0 4px;font-size:18px}
  header .meta{color:var(--muted);font-size:12px}
  header .meta code{background:var(--bg);padding:1px 5px;border-radius:3px;font-family:ui-monospace,SFMono-Regular,Menlo,monospace}
  .controls{display:flex;flex-wrap:wrap;gap:8px;padding:12px 24px;border-bottom:1px solid var(--border);background:var(--panel)}
  .controls select,.controls button,.controls input{background:var(--bg);color:var(--text);border:1px solid var(--border);border-radius:6px;padding:6px 10px;font:inherit}
  .controls button{cursor:pointer}
  .controls button.primary{background:var(--accent);color:#fff;border-color:var(--accent)}
  .progress{display:flex;gap:16px;padding:12px 24px;border-bottom:1px solid var(--border);font-size:12px;color:var(--muted);flex-wrap:wrap;background:var(--panel)}
  .progress span b{color:var(--text)}
  main{padding:16px 24px;max-width:1100px;margin:0 auto}
  .category{margin:24px 0 12px;font-size:12px;text-transform:uppercase;letter-spacing:0.1em;color:var(--muted);font-weight:600}
  .item{background:var(--panel);border:1px solid var(--border);border-radius:8px;padding:14px 16px;margin-bottom:10px;border-left:3px solid var(--border)}
  .item.status-pass{border-left-color:var(--pass)}
  .item.status-fail{border-left-color:var(--fail)}
  .item.status-skip{border-left-color:var(--skip)}
  .item.status-needs_review{border-left-color:var(--review)}
  .item-head{display:flex;align-items:flex-start;gap:10px;justify-content:space-between}
  .item-title{font-weight:600;font-size:15px}
  .badges{display:flex;gap:6px;flex-shrink:0}
  .badge{font-size:10px;text-transform:uppercase;letter-spacing:0.06em;padding:2px 6px;border-radius:4px;font-weight:600}
  .badge.prio-critical{background:var(--crit);color:#fff}
  .badge.prio-high{background:var(--high);color:#000}
  .badge.prio-medium{background:var(--med);color:#000}
  .badge.prio-low{background:var(--low);color:#fff}
  .badge.cat{background:var(--border);color:var(--muted)}
  .desc{color:var(--muted);margin:6px 0 10px;font-size:13px}
  .steps{margin:6px 0;padding-left:20px}
  .steps li{margin:2px 0}
  .expected{font-size:13px;margin:6px 0;padding:8px 10px;background:var(--bg);border-radius:6px;border:1px solid var(--border)}
  .expected b{color:var(--muted);font-weight:500;font-size:10px;text-transform:uppercase;letter-spacing:0.08em;display:block;margin-bottom:2px}
  .files{font-size:11px;color:var(--muted);font-family:ui-monospace,SFMono-Regular,Menlo,monospace;margin:6px 0;display:flex;flex-wrap:wrap;gap:4px}
  .files code{background:var(--bg);padding:1px 5px;border-radius:3px}
  .statuses{display:flex;gap:4px;margin:10px 0 8px;flex-wrap:wrap}
  .statuses button{background:var(--bg);color:var(--text);border:1px solid var(--border);border-radius:6px;padding:5px 10px;cursor:pointer;font-size:12px}
  .statuses button.active[data-s="pass"]{background:var(--pass);border-color:var(--pass);color:#fff}
  .statuses button.active[data-s="fail"]{background:var(--fail);border-color:var(--fail);color:#fff}
  .statuses button.active[data-s="skip"]{background:var(--skip);border-color:var(--skip);color:#fff}
  .statuses button.active[data-s="needs_review"]{background:var(--review);border-color:var(--review);color:#000}
  .statuses button.active[data-s="untested"]{background:var(--muted);border-color:var(--muted);color:#fff}
  textarea{width:100%;background:var(--bg);color:var(--text);border:1px solid var(--border);border-radius:6px;padding:8px;font:inherit;resize:vertical;min-height:46px}
  textarea.fail-only{display:none}
  .item.status-fail textarea.fail-only,.item.status-needs_review textarea.fail-only{display:block;border-color:var(--fail)}
  label{font-size:10px;color:var(--muted);text-transform:uppercase;letter-spacing:0.08em;display:block;margin:8px 0 4px;font-weight:600}
  .modal{position:fixed;inset:0;background:rgba(0,0,0,0.6);display:none;align-items:center;justify-content:center;z-index:100}
  .modal.open{display:flex}
  .modal-body{background:var(--panel);border:1px solid var(--border);border-radius:10px;width:min(900px,92vw);max-height:85vh;display:flex;flex-direction:column;overflow:hidden}
  .modal-head{padding:14px 18px;border-bottom:1px solid var(--border);display:flex;justify-content:space-between;align-items:center}
  .modal-head h2{margin:0;font-size:15px}
  .modal-head button{background:transparent;color:var(--muted);border:0;cursor:pointer;font-size:16px}
  .modal-body textarea{flex:1;min-height:320px;border:0;border-radius:0;font-family:ui-monospace,SFMono-Regular,Menlo,monospace;font-size:12px}
  .modal-foot{padding:10px 18px;border-top:1px solid var(--border);display:flex;gap:8px;justify-content:flex-end;background:var(--bg)}
  .modal-foot button{background:var(--panel);color:var(--text);border:1px solid var(--border);border-radius:6px;padding:6px 12px;cursor:pointer}
  .modal-foot button.primary{background:var(--accent);border-color:var(--accent);color:#fff}
</style>
</head>
<body>
<header>
  <h1>QA Checklist</h1>
  <div class="meta">
    Branch <b>__BRANCH__</b> vs <b>__BASE__</b> · commit <code>__COMMIT__</code> · generated __GENERATED_AT__
  </div>
</header>

<div class="controls">
  <select id="f-status">
    <option value="">All statuses</option>
    <option value="untested">Untested only</option>
    <option value="pass">Pass</option>
    <option value="fail">Failed only</option>
    <option value="skip">Skip</option>
    <option value="needs_review">Needs review</option>
  </select>
  <select id="f-priority">
    <option value="">All priorities</option>
    <option value="critical">Critical</option>
    <option value="high">High only</option>
    <option value="medium">Medium</option>
    <option value="low">Low</option>
  </select>
  <select id="f-category"><option value="">All categories</option></select>
  <input id="f-search" type="search" placeholder="Search title/files…" />
  <span style="flex:1"></span>
  <button class="primary" data-act="export-fix">Export Fix Prompt</button>
  <button data-act="export-summary">Export QA Summary</button>
  <button data-act="download-json">Download JSON</button>
  <button data-act="reset">Reset</button>
</div>

<div class="progress" id="progress"></div>

<main id="list"></main>

<div class="modal" id="modal">
  <div class="modal-body">
    <div class="modal-head">
      <h2 id="modal-title">Export</h2>
      <button data-act="close-modal" aria-label="Close">✕</button>
    </div>
    <textarea id="modal-text" readonly></textarea>
    <div class="modal-foot">
      <button class="primary" data-act="copy">Copy to clipboard</button>
      <button data-act="close-modal">Close</button>
    </div>
  </div>
</div>

<script id="qa-data" type="application/json">__ITEMS_JSON__</script>
<script>
const META = { branch:"__BRANCH__", base:"__BASE__", commit:"__COMMIT__", generatedAt:"__GENERATED_AT__", focus:__FOCUS__ };
const STORAGE_KEY = `qa:${META.branch}:${META.commit}`;
const ITEMS = JSON.parse(document.getElementById('qa-data').textContent);
const STATUSES = ['untested','pass','fail','skip','needs_review'];
const STATUS_LABELS = {untested:'Untested',pass:'Pass',fail:'Fail',skip:'Skip',needs_review:'Needs Review'};

function loadState(){ try { return JSON.parse(localStorage.getItem(STORAGE_KEY)) || {}; } catch { return {}; } }
function saveState(s){ localStorage.setItem(STORAGE_KEY, JSON.stringify(s)); }
let state = loadState();

function getItem(i){ return { ...i, ...(state[i.id] || {}) }; }
function setField(id, field, val){
  state[id] = { ...(state[id]||{}), [field]: val };
  saveState(state);
  if (field === 'status') render(); else renderProgress();
}

function render(){
  const catSel = document.getElementById('f-category');
  if (catSel.options.length <= 1) {
    [...new Set(ITEMS.map(i=>i.category))].sort().forEach(c=>{
      const o=document.createElement('option'); o.value=c; o.textContent=c; catSel.appendChild(o);
    });
  }
  const fStatus = document.getElementById('f-status').value;
  const fPrio = document.getElementById('f-priority').value;
  const fCat = document.getElementById('f-category').value;
  const fQ = document.getElementById('f-search').value.trim().toLowerCase();

  const list = document.getElementById('list');
  list.innerHTML = '';

  const grouped = {};
  ITEMS.forEach(raw => {
    const i = getItem(raw);
    const status = i.status || 'untested';
    if (fStatus && status !== fStatus) return;
    if (fPrio && i.priority !== fPrio) return;
    if (fCat && i.category !== fCat) return;
    if (fQ && !(i.title+' '+i.description+' '+(i.related_files||[]).join(' ')).toLowerCase().includes(fQ)) return;
    (grouped[i.category] ||= []).push(i);
  });

  Object.keys(grouped).sort().forEach(cat=>{
    const h = document.createElement('div'); h.className='category'; h.textContent=cat;
    list.appendChild(h);
    grouped[cat].forEach(i=>list.appendChild(renderItem(i)));
  });

  if (!list.children.length) {
    const empty = document.createElement('div');
    empty.style.cssText = 'padding:40px;text-align:center;color:var(--muted)';
    empty.textContent = 'No items match current filters.';
    list.appendChild(empty);
  }

  renderProgress();
}

function renderItem(i){
  const status = i.status || 'untested';
  const el = document.createElement('div');
  el.className = `item status-${status}`;
  el.innerHTML = `
    <div class="item-head">
      <div class="item-title">${escapeHtml(i.title)}</div>
      <div class="badges">
        <span class="badge prio-${i.priority}">${i.priority}</span>
        <span class="badge cat">${i.category}</span>
      </div>
    </div>
    ${i.description ? `<div class="desc">${escapeHtml(i.description)}</div>` : ''}
    ${i.steps_to_test?.length ? `<label>Steps</label><ol class="steps">${i.steps_to_test.map(s=>`<li>${escapeHtml(s)}</li>`).join('')}</ol>` : ''}
    ${i.expected_result ? `<div class="expected"><b>Expected</b>${escapeHtml(i.expected_result)}</div>` : ''}
    ${i.related_files?.length ? `<div class="files">${i.related_files.map(f=>`<code>${escapeHtml(f)}</code>`).join('')}</div>` : ''}
    <div class="statuses">
      ${STATUSES.map(s=>`<button data-s="${s}" class="${s===status?'active':''}">${STATUS_LABELS[s]}</button>`).join('')}
    </div>
    <label>Notes</label>
    <textarea data-field="notes" placeholder="Optional notes…">${escapeHtml(i.notes||'')}</textarea>
    <textarea class="fail-only" data-field="failure_details" placeholder="Failure details: what happened, screenshots, console errors, repro env…">${escapeHtml(i.failure_details||'')}</textarea>
  `;
  el.querySelectorAll('.statuses button').forEach(b=>b.addEventListener('click', ()=>setField(i.id,'status',b.dataset.s)));
  el.querySelectorAll('textarea[data-field]').forEach(t=>t.addEventListener('input', e=>setField(i.id,e.target.dataset.field,e.target.value)));
  return el;
}

function renderProgress(){
  const counts = {total:ITEMS.length, pass:0, fail:0, skip:0, needs_review:0, untested:0};
  ITEMS.forEach(raw=>{ const s = getItem(raw).status || 'untested'; counts[s]++; });
  const done = counts.pass+counts.fail+counts.skip+counts.needs_review;
  const pct = counts.total ? Math.round((done/counts.total)*100) : 0;
  document.getElementById('progress').innerHTML = `
    <span>Progress <b>${pct}%</b></span>
    <span>Total <b>${counts.total}</b></span>
    <span>Pass <b style="color:var(--pass)">${counts.pass}</b></span>
    <span>Fail <b style="color:var(--fail)">${counts.fail}</b></span>
    <span>Needs review <b style="color:var(--review)">${counts.needs_review}</b></span>
    <span>Skip <b>${counts.skip}</b></span>
    <span>Untested <b>${counts.untested}</b></span>
  `;
}

function escapeHtml(s){ return String(s??'').replace(/[&<>"']/g, c=>({ '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;' }[c])); }

function buildFixPrompt(){
  const data = ITEMS.map(getItem);
  const failed = data.filter(i=>i.status==='fail');
  const review = data.filter(i=>i.status==='needs_review');
  const skipped = data.filter(i=>i.status==='skip' && (i.notes||'').trim());
  const noted = data.filter(i=>['pass','untested'].includes(i.status||'untested') && (i.notes||'').trim());

  const section = (title, arr) => arr.length ? `\n## ${title}\n\n` + arr.map(i=>[
    `### [${i.id}] ${i.title} (${i.priority}, ${i.category})`,
    i.failure_details ? `**Observed:** ${i.failure_details}` : '',
    `**Expected:** ${i.expected_result || '—'}`,
    i.notes ? `**Notes:** ${i.notes}` : '',
    i.related_files?.length ? `**Related files:** ${i.related_files.join(', ')}` : '',
    i.suggested_fix_prompt_fragment ? `**Suggested:** ${i.suggested_fix_prompt_fragment}` : '',
  ].filter(Boolean).join('\n')).join('\n\n') : '';

  return [
    `You are working in this repo on branch \`${META.branch}\` (base \`${META.base}\`, commit ${META.commit}).`,
    `I ran QA using the generated checklist on ${META.generatedAt}.`,
    ``,
    `Please inspect the failed and needs-review items below, identify the root cause, implement fixes, and update or add tests where appropriate. Do not make unrelated changes. Follow the repo's existing conventions and any impact-analysis rules in CLAUDE.md.`,
    ``,
    `## Requested output format`,
    `For each issue: (1) root-cause analysis, (2) proposed fix with diff, (3) test added/updated, (4) regression check against the related files listed.`,
    section('Failed items', failed),
    section('Needs review', review),
    section('Skipped (with notes)', skipped),
    section('Other notes', noted),
  ].filter(Boolean).join('\n');
}

function buildSummary(){
  const data = ITEMS.map(getItem);
  const c = {pass:0,fail:0,skip:0,needs_review:0,untested:0};
  data.forEach(i=>c[i.status||'untested']++);
  const critFail = data.filter(i=>i.status==='fail' && (i.priority==='critical'||i.priority==='high'));
  const recommend = critFail.length ? 'NOT READY — critical/high failures present'
                  : c.fail ? 'NOT READY — failures present'
                  : c.needs_review ? 'NEEDS REVIEW'
                  : c.untested ? 'INCOMPLETE — items still untested'
                  : 'READY';
  const notes = data.filter(i=>(i.notes||'').trim()).map(i=>`- _${i.title}_: ${i.notes}`).join('\n');
  return [
    `# QA Summary — ${META.branch}`,
    `Generated: ${META.generatedAt} · commit ${META.commit} · base ${META.base}`,
    ``,
    `**Recommendation: ${recommend}**`,
    ``,
    `- Total: ${data.length}`,
    `- Pass: ${c.pass} · Fail: ${c.fail} · Needs review: ${c.needs_review} · Skip: ${c.skip} · Untested: ${c.untested}`,
    ``,
    critFail.length ? `## Critical / high failures\n` + critFail.map(i=>`- **${i.title}** (${i.priority}) — ${i.failure_details||'no details provided'}`).join('\n') : '',
    ``,
    `## Notes from testers`,
    notes || '_none_',
  ].filter(Boolean).join('\n');
}

function openModal(title, text){
  document.getElementById('modal-title').textContent = title;
  document.getElementById('modal-text').value = text;
  document.getElementById('modal').classList.add('open');
  navigator.clipboard?.writeText(text).catch(()=>{});
}

document.addEventListener('click', e=>{
  const a = e.target.closest('[data-act]'); if (!a) return;
  const act = a.dataset.act;
  if (act === 'export-fix') openModal('Fix Prompt (copied to clipboard)', buildFixPrompt());
  else if (act === 'export-summary') openModal('QA Summary (copied to clipboard)', buildSummary());
  else if (act === 'download-json') {
    const data = ITEMS.map(getItem);
    const blob = new Blob([JSON.stringify({meta:META, items:data}, null, 2)], {type:'application/json'});
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url; link.download = `qa-checklist-${META.branch}-${META.commit}.json`; link.click();
    URL.revokeObjectURL(url);
  }
  else if (act === 'close-modal') document.getElementById('modal').classList.remove('open');
  else if (act === 'copy') navigator.clipboard?.writeText(document.getElementById('modal-text').value);
  else if (act === 'reset') {
    if (confirm('Reset all QA progress for this branch/commit? This cannot be undone.')) {
      localStorage.removeItem(STORAGE_KEY); state = {}; render();
    }
  }
});
['f-status','f-priority','f-category','f-search'].forEach(id=>document.getElementById(id).addEventListener('input', render));
render();
</script>
</body>
</html>
```
