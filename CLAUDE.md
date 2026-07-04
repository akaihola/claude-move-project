# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`clamp` (**CL**aude **A**I **M**ove **P**roject) is a bash utility that moves, fixes, lists, verifies, prunes, and manages Claude Code projects while preserving all session history and settings. It handles three interconnected data stores:

1. **Project folder** - The actual project directory with code and `.claude/` settings
2. **History folder** - `~/.claude/projects/[encoded-path]/` containing session JSONL files
3. **History index** - `~/.claude/history.jsonl` with project path references

The encoded path format replaces **every non-alphanumeric character** (`/`, spaces, `_`, `.`, etc.) with `-`, so `/path/to/my_dir` becomes `-path-to-my-dir`. See "Path Encoding" below for the full rules.

## Testing

Test locally by running with `--dry-run` flag:
```bash
./clamp ./test-project ~/new-location --dry-run
```

## Key Implementation Details

- Uses `set -euo pipefail` for strict error handling
- Implements atomic rollback via EXIT trap if any step fails
- Handles macOS vs Linux `sed -i` differences
- Path resolution works for both existing and non-existing destination paths
- Must be compatible with bash 3.2 (macOS default) — no associative arrays
- Encoded path format is lossy (can't decode back) — use history.jsonl as source of truth
- `_list_has` uses `grep -qFx --` (note `--` to handle values starting with `-`)

## Path Encoding

`encode_path` must exactly reproduce Claude Code's own encoder (functions `$E`/`fNu`/`jCe` in the Claude binary), or `clamp` will look for the wrong session folder and fail to find/move history. The rules:

1. Replace every character matching `[^a-zA-Z0-9]` with `-` (mirrors Claude's `replace(/[^a-zA-Z0-9]/g, "-")`). This covers `/`, spaces, `_`, `.`, and all Unicode.
2. If the encoded name exceeds **200 characters**, truncate to the first 200 chars and append `-` plus a base-36 hash of the *original* path. The hash is a Java-style 31-multiplier `hashCode` over UTF-16 code units, masked to 32 bits, interpreted as signed, then `Math.abs(...).toString(36)`.

Helpers: `_claude_path_hash` (reads UTF-16BE code units via `iconv`, with an ASCII byte-value fallback) and `_to_base36`. Known limitation: astral (non-BMP, e.g. emoji) characters are not reproduced byte-for-byte, since Claude counts them as two UTF-16 code units — vanishingly rare in project paths. Validate any change by diffing `encode_path` output against a verbatim JS copy of `$E`/`fNu`/`jCe` run under `node`.

## Migration Sequence

The script performs operations in this order (critical for rollback):
1. Backup `history.jsonl`
2. Move project folder
3. Rename history folder in `~/.claude/projects/`
4. Update path references in `history.jsonl`

Rollback reverses these steps if any operation fails.
