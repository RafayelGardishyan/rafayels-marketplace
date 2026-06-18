# Issue Tracker Resolution

The workflows in this plugin (`brainstorm`, `plan`, `work`, `review`, `compound`,
`re:feature`) refer to a runtime-agnostic **`issue_tracker`** with a small set of
verbs. This file binds those abstract verbs to a concrete backend.

**Resolve `issue_tracker` in this order:**

1. **A registered `issue_tracker` tool** (the pi-runtime extension in
   `extensions/issue-tracker.ts`, stored under `.pi/issues/`). If your runtime
   exposes it, call it directly.
2. **Beads (`bd` CLI)** — if a `.beads/` directory exists at the repo root (i.e.
   `bd init` has been run). This is the binding for Claude Code and any runtime
   without the pi tool. **Prefer this when both are available** — it is git-backed
   and survives context compaction.
3. **No tracker** — fall back to ephemeral session todos (TodoWrite / file-todos).
   Skip the persistent steps; nothing else changes.

Detect the binding once at the start of a workflow:

```bash
test -d .beads && echo "tracker=beads" || echo "tracker=none-or-pi"
```

## Verb → command map

| Abstract verb        | Beads (`bd`)                                              | pi tool (`issue_tracker`)        |
|----------------------|----------------------------------------------------------|----------------------------------|
| `list` (open work)   | `bd list --status open` / `bd ready` (unblocked only)    | `action: list`                   |
| `get`                | `bd show <id>`                                            | `action: get`                    |
| `create`             | `bd create "<title>" -t <type> -p <0-4> -d "<desc>" -l <labels>` | `action: create`        |
| `update`             | `bd update <id> -s <status> -p <0-4> --add-label <tag>`  | `action: update`                 |
| `append_note`        | `bd update <id> --append-notes "<text>"`                 | `action: append_note`            |
| `close`              | `bd close <id> --reason "<why>"`                          | `action: close`                  |
| `reopen`             | `bd reopen <id>`                                          | `action: reopen`                 |
| link child → parent  | `bd create ... --parent <epic-id>` (or `bd update <id> --parent <epic-id>`) | tags + convention |
| blocking dependency  | `bd dep <blocker-id> --blocks <blocked-id>`              | tags + convention                |
| persist to git       | `bd sync` (also runs automatically via git hooks)        | files are committed with the repo |

Notes on beads specifics:

- **IDs are content-hashed**, e.g. `parai-sam-a3f2dd` — never assume sequential
  numbers. Capture the printed ID (or `--json`) when you create an issue.
- **Priority is `0`–`4`, `0` = highest.** Map review severities: P1 → `-p 1`,
  P2 → `-p 2`, P3 → `-p 3`.
- **Types** (`-t`): `task` (default), `bug`, `feature`, `epic`, `chore`.
- `bd ready` returns only issues with no open blockers — use it to pick the next
  task. `bd list --status open` returns everything open.
- `bd sync` runs at session end; bd's git hooks also auto-commit changes.

## Conventions

- **Epic per feature.** `re:feature` opens one `-t epic` issue for the feature and
  files every phase/task issue under it with `--parent <epic-id>`. Close the epic
  (or run `bd epic close-eligible`) when the feature merges.
- **Labels carry phase/source.** Use `-l brainstorm`, `-l plan`, `-l work`,
  `-l review`, `-l open-question`, `-l follow-up`, plus a feature slug
  (e.g. `-l feat-survey-export`) so any workflow can `bd list` its slice.
- **Findings → issues.** `review` files each surviving P1/P2/P3 finding as a `-t bug`
  (or `task`) issue labelled `review`, parented to the feature epic, with the PR ref
  in `--external-ref gh-<pr#>`.
- **Close with a reason.** Always `--reason` on close so the audit trail explains
  why (PR merged, finding fixed, won't-fix, duplicate).

## When NOT to use the tracker

Single-session, linear work whose context lives entirely in the current
conversation belongs in ephemeral todos, not the tracker. Use the tracker for work
that spans sessions, has dependencies/blockers, or must survive compaction. (See
the beads skill's bd-vs-TodoWrite guidance.)
