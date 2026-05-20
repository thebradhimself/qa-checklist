# qa-checklist

A [Claude Code](https://claude.com/claude-code) plugin that adds the
`/qa-checklist` slash command. It generates a **branch-aware, interactive
QA checklist** as a single self-contained HTML file, derived entirely from
the actual `git diff` of the current branch against its base — plus optional
GitHub PR metadata.

No generic filler. Every checklist item traces back to real changes on the
branch.

## Features

- Branch-aware: diffs `HEAD` against the detected base branch (default branch,
  `main`, or `master`) and tailors the checklist to those changes.
- Optional `gh` integration: pulls PR title, description, labels, and review
  comments when `--pr` is passed and the GitHub CLI is available.
- Single-file HTML output: ready to share with QA, attach to a PR, or open in
  a browser.
- Optional JSON and Markdown summary artifacts.
- Focus areas: bias the checklist toward `ui`, `backend`, `regression`,
  `security`, `perf`, or `a11y`.
- Read-only against your repo. The command never stages, commits, pushes,
  resets, checks out, rebases, or merges.

## Installation

### From this marketplace

In Claude Code:

```text
/plugin marketplace add thebradhimself/qa-checklist
/plugin install qa-checklist@qa-checklist
```

The first command registers this repo as a plugin marketplace. The second
installs the `qa-checklist` plugin from that marketplace.

### From a local clone

```bash
git clone https://github.com/thebradhimself/qa-checklist.git
```

Then in Claude Code:

```text
/plugin marketplace add /absolute/path/to/qa-checklist
/plugin install qa-checklist@qa-checklist
```

## Usage

From within a git repository, in a Claude Code session:

```text
/qa-checklist
```

Common variations:

```text
/qa-checklist --base main
/qa-checklist --pr
/qa-checklist --focus ui --focus a11y
/qa-checklist --output tmp/qa-checklist.html --json --summary
/qa-checklist --include-low-priority
```

### Arguments

| Flag | Description |
|---|---|
| `--base <branch>` | Base branch to diff against. Default: detected default branch → `main` → `master`. |
| `--pr` | Also fetch PR metadata via `gh pr view` / `gh pr diff` if available. |
| `--output <path>` | Output HTML path. Default: `tmp/qa-checklist.html`. |
| `--focus <area>` | Bias toward an area. Repeatable. Values: `ui`, `backend`, `regression`, `security`, `perf`, `a11y`. |
| `--include-low-priority` | Include `low`-priority items (off by default). |
| `--json` | Also write `tmp/qa-checklist-data.json`. |
| `--summary` | Also write `tmp/qa-checklist-summary.md`. |

### Output

By default, the command writes a single HTML file to `tmp/qa-checklist.html`.
Open it in any browser. The page is self-contained — no network requests,
no external assets — so it can be safely attached to a PR or emailed.

With `--json`, a machine-readable companion file is written alongside.
With `--summary`, a short Markdown summary is written for quick PR comments.

## Requirements

- Claude Code
- `git` on `PATH`
- Optional: `gh` (GitHub CLI), only when using `--pr`

## Project layout

```text
qa-checklist/
├── .claude-plugin/
│   ├── plugin.json         Plugin manifest
│   └── marketplace.json    Single-plugin marketplace catalog
├── commands/
│   └── qa-checklist.md     The /qa-checklist slash command
├── CHANGELOG.md
├── LICENSE
└── README.md
```

## Publishing to the Claude Code marketplace

This repo is structured so it can be consumed two ways:

1. As a standalone marketplace (`/plugin marketplace add thebradhimself/qa-checklist`),
   thanks to `.claude-plugin/marketplace.json` at the repo root.
2. As a plugin source referenced from another marketplace catalog.

To submit to the official Anthropic marketplace once it exists publicly, follow
the submission process documented at
<https://code.claude.com/docs/en/plugin-marketplaces>. Keep `version` in
`plugin.json` and the `CHANGELOG.md` in sync on every release.

## License

[MIT](./LICENSE)
