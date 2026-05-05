# PLF vs Cursor Agent Trace (and adjacent formats)

> A short comparison written so first-time readers — and the inevitable Show HN
> commenter — can see at a glance how PLF relates to the other things in this
> space. Short version: **different layers of the same problem; complementary,
> not competitive.**

## TL;DR

| | **PLF** | **Cursor Agent Trace** |
|---|---|---|
| **What it stores** | The prompts you sent the agent | The agent's actions, mapped to code ranges |
| **Direction** | Forward: prompt → outcome summary | Backward: code range → originating conversation |
| **Where the data lives** | `.prompts/` in your git repo, append-only JSONL | Cursor's local DB / cloud, queried via the editor |
| **Granularity** | One record per prompt | One trace per code edit |
| **Tool scope** | Vendor-neutral, intended for any agent to write | Cursor-specific |
| **License / openness** | Open spec (MIT), JSON Schema published | Proprietary, queried through Cursor's UI |
| **Primary use case** | Audit log, provenance, dataset, portability | "Why is this code here? Show me the chat that produced it." |

## What's actually different

### 1. Direction of lookup

Agent Trace answers a **code-first** question: *"I'm looking at line 42 — what
conversation produced it?"* It's optimized for in-editor navigation from a
piece of source back to its originating chat turn.

PLF answers a **prompt-first** question: *"What prompts did this team send
last week, on which branches, with what outcomes?"* It's a flat, queryable
log over time, not a graph rooted in code.

These are inverse views of the same underlying relationship. A team could
reasonably want both.

### 2. Where the data lives

Agent Trace lives in Cursor's local index (and, depending on settings, in
their cloud). It's queryable through the editor; it's not a file you check
into your repo.

PLF lives in `.prompts/` in your git repo. It's plain JSONL. `jq` works.
GitHub renders it. Your CI can grep it. When you switch tools, the data
comes with you because it was always yours.

That difference matters less for an individual developer and more for a
team that wants prompts to be a first-class artifact alongside code.

### 3. Tool scope

Agent Trace is a feature of one IDE. By design.

PLF is a spec any agent can implement — Claude Code does today via
[Promptcellar](https://github.com/dominiek/promptcellar-for-claude-code);
the format is structured so Cursor, Aider, Codex, OpenCode, Cline, or a
homegrown agent could emit the same records into the same `.prompts/`
directory. The point of writing it as a spec rather than a library is
exactly this: cross-tool comparability and portability.

### 4. Granularity and what counts as "one thing"

Agent Trace's atomic unit is an **edit-with-context**: a code range plus the
conversation that produced it. Multiple edits in one chat turn become
multiple traces.

PLF's atomic unit is **one prompt submission**: one record per prompt, with
the outcome (files touched, commits, status, tokens, cost) folded into the
same record at agent-stop time. Multiple edits inside one prompt become
fields *inside* that record, not separate records.

Both choices are reasonable. They reflect the different lookup directions
above.

## Why PLF and Agent Trace are complementary

A team using both would get:

- **Agent Trace inside the editor** for "explain this line" navigation while
  coding.
- **PLF in the repo** for audits ("which prompts touched the auth module last
  quarter?"), dataset construction (training a custom completion on your own
  prompt history), cost reporting, and tool-portability ("we're trying Aider
  for a week — keep the prompt log going").

Neither replaces the other. The interesting integration question is whether
PLF records can carry an `agent_trace_id` enrichment so the two can be
joined when both are present in the same workflow. The schema is permissive
about additional fields under `enrichments`, so this is a one-line change
when someone wants to wire it up.

## Adjacent formats, briefly

- **Aider's `.aider.chat.history.md`** — closest existing thing to PLF.
  Markdown, repo-checked, per-session. Different shape (transcript-style,
  not record-style) and Aider-specific. PLF is, in part, an attempt to
  generalize this idea across agents.
- **Continue / Cline session exports** — JSON dumps of one session at a
  time, written on demand rather than continuously. Useful as snapshots,
  not as a running log.
- **OpenAI / Anthropic API request logs** — exist at the API layer, capture
  the *model call* not the *developer prompt*, and are typically held by
  the platform vendor.
- **Langfuse / Helicone / PromptLayer** — runtime observability platforms.
  They sit at the API layer too; great for "what did the model see," not
  designed for "what did the developer ask." Complementary to PLF, not
  redundant with it.
- **`git-ai-project/git-ai`** — a sibling effort exploring AI provenance in
  git. Conversation about interop is open at their issue tracker.

## What PLF does **not** try to be

- Not a transcript format. Outcome fields are summaries; if you want full
  agent traces, store them elsewhere and link via record `id`.
- Not an analytics product. Promptcellar Nerve Center will be one such
  product over `.prompts/`, but the spec is consciously underneath it.
- Not a wire format for live API calls. PLF is on-disk, append-only, and
  written after the fact by tools that observe their own activity.

## Pull requests welcome

If you maintain an adjacent format and the comparison above misrepresents
it, please open an issue or PR. The goal is to make the boundaries clear,
not to play down what other people have built.
