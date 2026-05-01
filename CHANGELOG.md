# Changelog

## `plf-1` — 2026-04-29

Initial public draft.

- The `.prompts/YYYY/MM/DD/<session-id>.jsonl` directory layout, bucketed by session start date in UTC.
- Required record fields: `version`, `id`, `session_id`, `timestamp`, `author`, `tool`, `model`.
- Optional record fields: `prompt`, `git`, `parent`, `outcome` (summary, files touched, commits, status), `enrichments` (tokens, cost, duration).
- Excluded-stub records emitted when a prompt matches `.promptcellarignore`.
- The `.promptcellarignore` file format (regex patterns with optional `id:` labels).
- Versioning rules for additive vs. breaking changes.
