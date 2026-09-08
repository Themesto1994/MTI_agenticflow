---
name: login-flow-suite
description: Location and kane-cli request ID for the Login Flow test suite generated from the Login_Flow_PRD.pdf
metadata: 
  node_type: memory
  type: project
  originSessionId: ""
  modified: 2026-09-08T00:00:00.000Z
---

Source doc: `Login_Flow_PRD.pdf` (project root). Suite generated via `kane-cli generate` against this PRD; request ID and suite path will be populated after the first pipeline run (check `planner` job summary for the request ID, and `.testmuai/tests/` for the suite directory name).

Expected scenarios (from Login_Flow_PRD.pdf — to be confirmed by first generation pass):
- **Login-Positive** — valid credentials, successful login to application
- **Login-Negative** — invalid username, invalid password, empty fields, locked account
- **Login-Edge** — case sensitivity, special characters in credentials, session expiry, concurrent logins

**Why:** Keeps the request id and suite path discoverable so further work (more `--refine --req <id>` passes, re-saving, or `kane-cli testmd run`) doesn't require re-generating from scratch. Update this file with the real request ID and suite path after the first successful pipeline run.

**How to apply:** If the user references "the login tests," "the login suite," or wants to extend/refine/run this suite, use the request id and suite path recorded here directly instead of asking or re-reading the PDF from scratch. See [[testcase-generation-flow]] for the general process this suite follows.
