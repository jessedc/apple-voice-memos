---
name: apple-voice-memos
description: Extract and process transcripts from Apple Voice Memos synced via iCloud. Use when the user wants to access, read, or summarize voice memos.
allowed-tools: "Bash(python3:*), Read, Write, Task"
compatibility: macOS with Voice Memos iCloud sync enabled, Python 3
license: 0BSD
---

# Apple Voice Memos

Extract and process transcripts from Apple Voice Memos synced via iCloud.

## Prerequisites

Voice Memos must be synced with iCloud on macOS.

## Tools

This skill includes two scripts in its `scripts/` directory. Script paths in this document are relative to this skill's directory — your working directory is likely elsewhere, so invoke them with absolute paths (e.g. `python3 <skill-dir>/scripts/extract-apple-voice-memos-metadata`).

- **`extract-apple-voice-memos-metadata`** — Queries the CloudRecordings.db SQLite database (read-only) and outputs CSV with columns: `title`, `date`, `duration`, `path`, `has_transcript`. Supports optional flags: `--limit N` (default 10), `--offset N`, `--search TERM`, `--after YYYY-MM-DD`, `--before YYYY-MM-DD`.
- **`extract-apple-voice-memos-transcript`** — Extracts the embedded transcript from a `.m4a` file's `tsrp` atom. Outputs timestamped text with filler words removed, intelligent line breaks, and paragraph breaks at natural pauses.

## Step 1: Select a voice memo

Run the metadata script to find the right recording. Choose flags based on what the user asked for:

- **No specific request** → run with no flags (returns 10 most recent)
- **User mentions a topic or keyword** → use `--search TERM`
- **User mentions a time period** → use `--after YYYY-MM-DD` and/or `--before YYYY-MM-DD`
- **User wants to see more results** → use `--offset N` to paginate, or `--limit N` to increase the batch size

Flags can be combined, e.g. `--search work --after 2026-01-01 --limit 5`.

```bash
python3 scripts/extract-apple-voice-memos-metadata [flags]
```

If the user's request unambiguously identifies a single memo (e.g., "my latest voice memo", or a search that returns exactly one match), proceed directly with that memo without asking. Otherwise, present the results as a numbered list showing title, date, and duration — mark any memo with `has_transcript` = `no` as "(no transcript)" — and ask the user which memo they'd like to work with.

**Error handling:**
- "Database not found" → Voice Memos iCloud sync is not enabled on this Mac.

## Step 2: Extract and process the transcript

Spawn a subagent with fresh context to extract and process the transcript. Do not run the transcript script yourself first — letting the subagent extract it keeps the full transcript out of the main conversation context. The subagent prompt must contain:

1. The `title` and `path` values of the selected recording (from the metadata output)
2. The absolute paths to this skill's `scripts/extract-apple-voice-memos-transcript` script and `PROMPT.md`
3. These instructions: run the transcript script with the recording path, read `PROMPT.md`, append the memo title after the `## Memo Title` heading and the transcript after the `## Transcript` heading, then follow the complete prompt and return the resulting markdown document.

If your environment has no subagent or task tool, do the same work yourself: run the transcript script, read `PROMPT.md`, and follow its instructions directly with the extracted transcript.

Only run the transcript script directly if the user explicitly asks to see the raw timestamped transcript:

```bash
python3 scripts/extract-apple-voice-memos-transcript "<FILENAME>.m4a"
```

**Error handling (reported by the subagent):**
- "tsrp atom not found" → This recording does not have an embedded transcript. Apple generates transcripts on-device and not all recordings will have one.
- File not found → The recording file may not have synced to this Mac yet.

## Step 3: Present the output

The subagent will produce a markdown document starting with a `# <Memo Title>` heading, followed by narrative summary, detailed notes, asides, and action items. Insert a metadata block directly beneath the title heading, then present the output to the user:

```markdown
# <Memo Title>

- **Date:** <date from the metadata output>
- **Duration:** <duration from the metadata output>
- **Source:** <FILENAME>.m4a
```

After presenting the output, ask the user if they'd like to save it as a markdown file. Suggest a filename in the format `YYYY-MM-DD-slugified-title.md` derived from the memo's title and date (e.g., `2026-02-04-the-soul-of-a-new-machine.md`). Save to the current working directory by default. The user may accept, provide a different name or path, request adjustments to the content first, or skip saving.
