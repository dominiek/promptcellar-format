# Promptcellar Logging Format — `plf-1` Specification

**Status:** Draft 1 (2026-04-29)
**Conformance language:** the words MUST, SHOULD, and MAY are used per RFC 2119.

## 1. Overview

The Promptcellar Logging Format (PLF) is a file-based, append-only format for capturing prompts sent to agentic coding tools. Version `plf-1` is specified by this document.

A PLF data store is a directory tree rooted at `.prompts/` inside a host project (typically a git repository). The tree contains JSON Lines files; each line is one PLF record describing one prompt.

The format is designed to be:

- Writable by any agentic coding tool.
- Readable with general-purpose JSON tooling.
- Mergeable across git branches without conflicts in normal operation.
- Stable: records written today are readable by future versions of conforming tools.

## 2. Directory layout

A PLF data store has the following layout, relative to its root (typically `.prompts/`):

```
<root>/
  YYYY/
    MM/
      DD/
        <session-id>.jsonl
```

- `YYYY` is a four-digit year (UTC).
- `MM` is a two-digit month (`01`–`12`).
- `DD` is a two-digit day (`01`–`31`).
- A file's path bucket MUST be derived from the **session start date** in UTC. Records added later in the session, even when they cross day boundaries, MUST be appended to the same file.
- `<session-id>` is the same string used as the `session_id` field of every record in the file. It MUST be globally unique. UUIDv4 is RECOMMENDED.

### 2.1 File framing

Each file is JSON Lines (`application/x-ndjson`):

- One record per line.
- Each line is a single JSON object with no embedded newlines.
- Lines MUST be terminated by `\n` (LF). The final line MUST be terminated.
- Files SHOULD be UTF-8 without BOM.
- Files MUST be append-only. Tools MUST NOT rewrite or reorder existing lines.

### 2.2 Avoiding merge conflicts

Each session file is written by exactly one process. Different sessions, including those created on different branches or by different developers, MUST use distinct `session_id`s and therefore live in distinct files. Git therefore sees only file additions and never overlapping edits.

## 3. Record schema

Each line of a session file is one PLF record: a JSON object whose top-level fields are listed below.

A record is in one of two shapes:

- **Captured**: contains a `prompt` field with the user's text.
- **Excluded**: contains an `excluded` field describing why the prompt was not captured (see §3.9). An excluded record MUST NOT contain `prompt`, `outcome`, `enrichments`, `git`, or `parent`.

### 3.1 Required fields

| Field | Type | Description |
|---|---|---|
| `version` | string | MUST be the literal `"plf-1"`. |
| `id` | string | UUID for this record. UUIDv4 RECOMMENDED. |
| `session_id` | string | Stable across all records in the session. MUST equal the basename (without `.jsonl`) of the file. |
| `timestamp` | string | RFC 3339 date-time in UTC. Millisecond precision RECOMMENDED. |
| `author` | object | See §3.2. |
| `tool` | object | See §3.3. |
| `model` | object | See §3.4. |

### 3.2 `author`

The human who issued the prompt.

| Field | Type | Notes |
|---|---|---|
| `email` | string | Required. Typically `git config user.email`. |
| `name` | string | Required. Typically `git config user.name`. |
| `id` | string \| null | Optional stable identifier (e.g. signing-key fingerprint, SSO subject). |

### 3.3 `tool`

The agentic coding tool that captured the prompt.

| Field | Type | Notes |
|---|---|---|
| `name` | string | Required. Lower-kebab-case, e.g. `"claude-code"`, `"cursor"`, `"aider"`. |
| `version` | string | Required. The tool's own version string. |

### 3.4 `model`

The model the tool was using when the prompt was issued.

| Field | Type | Notes |
|---|---|---|
| `provider` | string | Required. e.g. `"anthropic"`, `"openai"`, `"google"`, `"local"`. |
| `name` | string | Required. e.g. `"claude-opus-4-7"`. |
| `version` | string \| null | Optional version pin if known. |

### 3.5 `prompt`

A string containing the verbatim prompt the user typed. Required for captured records, forbidden for excluded stubs.

### 3.6 `git` (optional)

Snapshot of the working repository at prompt time.

| Field | Type | Notes |
|---|---|---|
| `branch` | string | The branch checked out when the prompt was issued. |
| `head_commit` | string | SHA (7–40 hex chars) of `HEAD` at prompt time. |
| `dirty` | boolean | `true` if the working tree had uncommitted changes. |

### 3.7 `parent` (optional)

Links this prompt to a prior prompt in the same session. Useful for representing retries or branched conversations.

| Field | Type | Notes |
|---|---|---|
| `prompt_id` | string | The `id` of the prior record. MUST belong to the same `session_id`. |

### 3.8 `outcome` (optional)

A summary of what the agent did in response. PLF deliberately captures summaries, not full transcripts.

| Field | Type | Notes |
|---|---|---|
| `summary` | string | Short natural-language summary. SHOULD be ≤ ~500 characters. |
| `files_touched` | array of strings | Repository-relative paths the agent edited. |
| `commits` | array of strings | SHAs created during the session that this prompt contributed to. |
| `status` | string | One of `"completed"`, `"errored"`, `"interrupted"`, `"unknown"`. |

### 3.9 `excluded` (optional)

Present only on stub records emitted when a prompt matched a `.promptcellarignore` pattern (see §4). When `excluded` is present:

- `prompt`, `outcome`, `enrichments`, `git`, and `parent` MUST be absent.
- All required fields (§3.1) MUST be present.

| Field | Type | Notes |
|---|---|---|
| `reason` | string | Required. Short human-readable phrase, e.g. `"matched .promptcellarignore"`. |
| `pattern_id` | string | Optional. The `id:` label of the matching pattern, if it declared one. |

### 3.10 `enrichments` (optional)

Cost and time attribution. Tools SHOULD record what they have; all sub-fields are optional.

| Field | Type | Notes |
|---|---|---|
| `tokens.input` | integer ≥ 0 | |
| `tokens.output` | integer ≥ 0 | |
| `tokens.cache_read` | integer ≥ 0 | |
| `tokens.cache_write` | integer ≥ 0 | |
| `cost_usd` | number ≥ 0 | Best-effort estimate at capture time. |
| `duration_ms` | integer ≥ 0 | Wall-clock from prompt submission to agent response complete. |

### 3.11 Unknown fields

Conforming readers MUST preserve unknown top-level fields when re-serialising a record (forward compatibility). Tool-specific extensions SHOULD be namespaced under a single `x_<tool>` object so they do not collide with future reserved names.

## 4. `.promptcellarignore`

A repo-root file listing patterns that, when matched against a prompt's text, cause the prompt to be excluded from capture.

### 4.1 Syntax

- One pattern per line.
- Lines beginning with `#` are comments. Blank lines are ignored.
- A line of the form `id: <name>` immediately preceding a pattern names that pattern. The name MUST match `[A-Za-z0-9_-]+`. Names appear in `excluded.pattern_id`.
- Patterns are POSIX extended regular expressions, applied case-insensitively by default.

### 4.2 Behaviour

When any pattern matches the prompt text:

- The captured record is NOT written.
- An excluded stub (§3.9) is written in its place. The stub preserves the timeline so consumers can see that capture was skipped.
- Matching MUST be performed locally before any prompt is written to disk.

### 4.3 Example

```
id: secrets
(AWS_SECRET_ACCESS_KEY|GITHUB_TOKEN|OPENAI_API_KEY)

id: security-paths
\bsecurity/(runbooks|incident)\b

id: credential-shapes
(ghp_[A-Za-z0-9]{36}|sk-[A-Za-z0-9]{32,})
```

## 5. Versioning

- Every record carries its own `version` field. Records of different versions MAY coexist in the same data store.
- Additive changes (new optional fields, new enum members on existing open vocabularies) do not bump the version.
- Breaking changes (removed or renamed fields, changes to required fields) bump to a new version (`plf-2`, etc.).
- Reading tools MUST handle records with unknown `version` strings by ignoring or warning, never by erroring out the whole file.

## 6. Out of scope

- **Full transcripts and tool-call traces.** The audit unit is `outcome.summary`. Tools that want richer detail can store it externally and link by record `id`.
- **Indexing, search, and dashboards.** These are tooling concerns above the format.
- **Encryption.** PLF's premise is shared visibility into the human signal that built the code; teams that cannot commit prompts in plaintext should not adopt the format.

## 7. References

- [JSON Lines](https://jsonlines.org)
- [RFC 3339 — Date and Time on the Internet](https://www.rfc-editor.org/rfc/rfc3339)
- [JSON Schema 2020-12](https://json-schema.org/specification.html)
- [RFC 2119 — Key words for use in RFCs](https://www.rfc-editor.org/rfc/rfc2119)
