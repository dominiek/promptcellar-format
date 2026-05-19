# Changelog

The PLF version (`plf-1`, `plf-2`, …) is recorded inside every record's
`version` field. Additive clarifications under the same version are listed
here under their dated entry; breaking changes bump the version.

## `plf-1` — 2026-04-29 (latest revision: 2026-05-19)

Initial public draft.

- The `.prompts/YYYY/MM/DD/<session-id>.jsonl` directory layout, bucketed by session start date in UTC.
- Required record fields: `version`, `id`, `session_id`, `timestamp`, `author`, `tool`, `model`.
- Optional record fields: `prompt`, `git`, `parent`, `outcome` (summary, files touched, commits, status), `enrichments` (tokens, cost, duration).
- Excluded-stub records emitted when a prompt matches `.promptcellarignore`.
- The `.promptcellarignore` file format (regex patterns with optional `id:` labels).
- Versioning rules for additive vs. breaking changes.

### 2026-04-30 — clarification

- Path layout simplified: dropped the hour bucket. Records land at
  `.prompts/YYYY/MM/DD/<session-id>.jsonl`, not
  `.prompts/YYYY/MM/DD/HH/<session-id>.jsonl`. Pre-launch clarification
  ahead of the first stable adopters; not a `plf-2` bump.

### 2026-05-19 — additive: `cwd` field

- New optional top-level `cwd` field (§3.7) gives the working directory the
  prompt was issued from, expressed as a path relative to the PLF store
  root. Disambiguates `outcome.files_touched` when one store collects
  prompts from multiple sibling repos (e.g. a workspace destination that
  pools several projects into one `.prompts/`). Omitted when the working
  directory equals the store root, MUST use forward slashes, and MUST NOT
  be absolute. Excluded stubs (§3.10) MUST NOT carry `cwd`. Additive — no
  `plf-2` bump.
