# Promptcellar Logging Format (PLF)

PLF is an open, file-based standard for logging the prompts a developer sends to agentic coding tools. It captures the human signal that built a piece of software — the questions, instructions, and corrections that shaped it — so that signal can be audited, traced, and carried across tools.

This repository is the **specification and reference assets** for `plf-1`. It is not an implementation. The first implementation is [Promptcellar for Claude Code](https://github.com/dominiek/promptcellar-for-claude-code); the format is intentionally separate so any other agent (Cursor, Aider, Codex, your own) can write or read the same files without depending on Promptcellar.

## What it is

A PLF record is a single JSON object describing one prompt. Records are stored one per line (JSONL), one file per session:

```
.prompts/YYYY/MM/DD/<session-id>.jsonl
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

Optional fields cover git state at prompt time, outcome (files touched, commits, status), and enrichments (tokens, cost, duration). The full schema is in [`SPEC.md`](./SPEC.md); JSON Schema 2020-12 is at [`schemas/plf-1.schema.json`](./schemas/plf-1.schema.json); working samples in [`examples/`](./examples/).

## Design choices

- **JSONL, one file per session.** Sessions are single-writer, so two branches can never write to the same file — merge conflicts in `.prompts/` are avoided by construction.
- **Tool- and model-agnostic.** `tool` and `model` are separate fields so any agent can emit the same format.
- **Outcomes are summaries, not transcripts.** PLF is an audit log, not a replay log. Tools that want full traces store them elsewhere and link via record `id`.
- **`.promptcellarignore`** lets a team declare patterns whose match excludes a prompt from capture; a stub record is written in place so the timeline isn't gappy.

See [`SPEC_VS_AGENT_TRACE.md`](./SPEC_VS_AGENT_TRACE.md) for how PLF compares to Cursor's Agent Trace and other adjacent formats.

## Validating a record

Any JSON Schema 2020-12 validator works:

```sh
pipx run check-jsonschema --schemafile schemas/plf-1.schema.json examples/full.json
npx ajv-cli validate -s schemas/plf-1.schema.json -d examples/full.json
```

## Status

`plf-1` is the first published version. Additive changes (new optional fields) won't bump it. Breaking changes will release as `plf-2`; both versions can coexist in the same repo because every record carries its own `version`.

## License

MIT — see [`LICENSE`](./LICENSE).
