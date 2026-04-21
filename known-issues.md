# Known Issues

Active and recently resolved issues for `molecule-audit-trail`.

---

## Active Issues

*(None currently open. File an issue if you encounter a problem.)*

---

## Known Gotchas

These are not bugs but behaviors that may surprise contributors or operators.

### `.claude/` directory must exist before first audit write

**Severity:** Low
**Workaround:** Ensure `.claude/` is created (e.g., by another plugin or a session-start hook) before the first Edit/Write call. The hook silently succeeds with no output if the directory is missing.

**Prevention:** The hook could `os.makedirs(AUDIT, exist_ok=True)` but intentionally does not, to avoid creating directories that were not expected to exist.

---

### Audit failures are silent — no observability signal

**Severity:** Low
**Impact:** If the `.claude/` directory is missing or not writable, the hook silently completes with no error. The workspace operator has no way to detect that auditing is not working.

**Workaround:** Manually verify `.claude/audit.jsonl` exists and is growing after the first Edit/Write call. For critical audit requirements, add a pre-session check that verifies writability:

```bash
touch .claude/audit.jsonl && echo "audit write OK" || echo "audit write FAILED"
```

---

### `ok` field reflects `tool_response.success`, not HTTP/exit-code status

**Severity:** Informational
**Detail:** The `ok` field in each JSONL record is `tool_response.success`, which reports whether the tool declared success — not whether the underlying write was durable. A tool that reports success but fails at the filesystem level (e.g., ENOSPC) may still record `"ok": true`.

---

### No size management — audit log grows indefinitely

**Severity:** Low (operational)
**Impact:** In long-running sessions with many file edits, `.claude/audit.jsonl` can grow large.

**Workaround:** Periodically rotate the log or rely on the session-bound lifecycle: `.claude/` is typically ephemeral per workspace and is cleaned up on workspace teardown.

---

## Recently Resolved

*(None yet.)*
