# Promptcellar Logging Format (PLF)

PLF is an open, file-based standard for capturing the prompts a developer sends to agentic coding tools. It exists to reliably capture the human signal that built a piece of software — the questions, instructions, and corrections that shaped it. This information can be used to more easily rewrite, port and abstract living software.

This repository is the specification and reference assets for `plf-1`.

## What it is

A PLF record is a single JSON object describing one prompt: who wrote it, when, with which tool and model, the prompt itself, and (optionally) what the agent did in response. Records are stored one per line (JSONL) under `.prompts/`:

```
.prompts/
  YYYY/MM/DD/
    <session-id>.jsonl
```

A minimal record:

```json
{
  "version": "plf-1",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "8f14e45f-ceea-467a-9575-d68a64236d57",
  "timestamp": "2026-04-29T09:14:22.001Z",
  "author": { "email": "jane@example.com", "name": "Jane Doe" },
  "tool": { "name": "claude-code", "version": "2.4.0" },
  "model": { "provider": "anthropic", "name": "claude-opus-4-7" },
  "prompt": "add a health check endpoint at /healthz that returns 200 OK"
}
```

The full schema is in [`SPEC.md`](./SPEC.md). Working samples live in [`examples/`](./examples/).

## Why

Agentic coding shifts where meaningful design decisions live. Reasoning that used to show up in code review now happens in chat with an agent. PLF captures that conversation in the repo itself, so it can be:

- **Audited** — who asked the agent to do what, when.
- **Traced** — link a commit, file, or line back to the prompt that produced it.
- **Compared** — across teams, projects, and tools, because the format is shared.
- **Owned by the team** — the data lives in the repo, not in a vendor's database.

## Design choices, briefly

- **JSONL, one file per session.** Sessions are single-writer, so two branches can never write to the same file. Merge conflicts are avoided by construction.
- **Tool- and model-agnostic.** `tool` and `model` are separate fields. Claude Code, Cursor, Aider, Codex, and any other agent can write the same format.
- **Outcomes are summaries, not transcripts.** PLF is an audit log, not a replay log. Tools that want full traces can store them elsewhere and link via record `id`.
- **`.promptcellarignore`** lets a team declare patterns whose match excludes a prompt from capture (e.g. anything mentioning `AWS_SECRET_ACCESS_KEY`). A stub record is written in place so the timeline is not gappy.

## Project layout

```
.
├── README.md          ← this file
├── SPEC.md            ← the formal specification
├── schemas/
│   └── plf-1.schema.json
├── examples/
│   ├── minimal.json
│   ├── full.json
│   ├── excluded.json
│   ├── session.jsonl
│   └── .promptcellarignore
├── CHANGELOG.md
└── LICENSE
```

## Validating a record

`schemas/plf-1.schema.json` is JSON Schema 2020-12. Any conforming validator works:

```sh
# With check-jsonschema
pipx run check-jsonschema --schemafile schemas/plf-1.schema.json examples/full.json

# With ajv
npx ajv-cli validate -s schemas/plf-1.schema.json -d examples/full.json
```

## Reading PLF data

Every session is a JSONL file, so anything that reads JSONL works:

```sh
# Most recent prompts across the repo
find .prompts -name '*.jsonl' -exec cat {} + \
  | jq -s 'sort_by(.timestamp) | .[-10:] | .[] | {timestamp, author: .author.email, prompt}'

# Just the prompts a given author has written
find .prompts -name '*.jsonl' -exec cat {} + \
  | jq -r 'select(.author.email == "jane@example.com") | .prompt'

# Total cost spent on a branch
find .prompts -name '*.jsonl' -exec cat {} + \
  | jq -s '[.[] | select(.git.branch == "feat/auth-rewrite") | .enrichments.cost_usd // 0] | add'
```

## Status

`plf-1` is the first published version. Additive changes (new optional fields) will not bump the version. Breaking changes will release as `plf-2`; both versions can coexist in the same repo because every record carries its own `version`.

## License

MIT — see [`LICENSE`](./LICENSE).
