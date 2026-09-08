---
name: mirror-memory-to-repo
description: "Always keep a visible copy of the memory store inside the project repo, in sync with the real memory files"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: ""
  modified: 2026-09-08T00:00:00.000Z
---

Whenever a memory file is created, edited, or deleted in the Claude memory store for this project, mirror the same change into `MTI_agenticflow/.claude-memory/` (same filenames, same content, including `MEMORY.md`).

**Why:** The real memory store lives under `~/.claude/projects/...`, keyed by workspace path, which is outside the project folder and doesn't show up in the user's IDE file tree. A copy inside the repo lets anyone with the repo see what context Claude carries, kept updated always — not a one-time snapshot.

**How to apply:** Treat `.claude-memory/` in the repo as a read-mirror, not the source of truth — always write/edit the real memory file first, then copy it over (or re-copy after edits) before ending the turn. Do this for every memory write in this project going forward, not just on explicit request.
