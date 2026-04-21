# molecule-ai-plugin-molecule-audit-trail

Appends a one-line JSONL audit record for every Edit and Write tool call to
`.claude/audit.jsonl`. Provides an append-only, chronologically ordered history
of all file modifications across a workspace session.

**Version:** 1.0.0
**Runtime:** `claude_code`
**Hook:** `post-edit-audit` (PostToolUse)
**Output:** `.claude/audit.jsonl`

---

## What It Does

Every time an Edit or Write tool succeeds, the `post-edit-audit` hook appends a
single JSON line to `.claude/audit.jsonl` in the workspace root:

```jsonl
{"ts": "2026-04-21T14:00:00Z", "tool": "Edit", "file": "src/main.go", "ok": true}
{"ts": "2026-04-21T14:00:01Z", "tool": "Write", "file": "README.md", "ok": true}
```

- Timestamps are UTC (`YYYY-MM-DDTHH:MM:SSZ`).
- `file` is relative to the repo root.
- Only Edit and Write are audited — Read/List/Bash/agent calls are not logged.
- Failures are silently swallowed; the tool always completes successfully.

---

## Repository Layout

```
molecule-audit-trail/
├── hooks/
│   ├── post-edit-audit.py   # Python hook implementation
│   ├── post-edit-audit.sh   # Thin shim invoking the Python module
│   └── _lib.py              # Shared helpers: input reading, stderr warnings
├── adapters/
│   ├── __init__.py
│   └── claude_code.py       # Claude Code harness adapter
├── plugin.yaml              # Plugin manifest
├── settings-fragment.json   # Workspace config snippet
└── README.md
```

---

## Hook Details

### `post-edit-audit` (PostToolUse)

Triggered **after** any Edit or Write tool call returns. The Python hook:

1. Reads the tool call payload from stdin (`tool_name`, `tool_input`, `tool_response`).
2. Extracts `file_path` from `tool_input` (handles both `file_path` and `notebook_path`).
3. Records `{ts, tool, file, ok}` and appends to `.claude/audit.jsonl`.
4. Exits `0` — never blocks tool execution.

The `.claude/` directory must exist; if it does not, the hook silently succeeds
with no output written.

---

## Development

```bash
# Simulate a tool call and run the hook locally
echo '{"tool_name":"Edit","tool_input":{"file_path":"src/foo.go"},"tool_response":{"success":true}}' \
  | python hooks/post-edit-audit.py

# Check the audit log
cat .claude/audit.jsonl | jq

# Lint Python
python -m py_compile hooks/post-edit-audit.py hooks/_lib.py
```

---

## Key Conventions

| Topic | Convention |
|---|---|
| **Output format** | JSONL, one record per line, append-only |
| **Timestamp** | UTC, ISO 8601 (`YYYY-MM-DDTHH:MM:SSZ`) |
| **File path** | Relative to repo root (never absolute) |
| **Failure behavior** | Silent — never blocks tool execution |
| **No auth required** | Hook has no external dependencies |
| **Python version** | Compatible with Python 3.x (no external deps) |

---

## Relationship to Other Plugins

- Pairs with `molecule-session-context` to replay audit context at session start.
- Complements `molecule-workflow-retro` which records retrospective notes rather
  than raw file edits.
