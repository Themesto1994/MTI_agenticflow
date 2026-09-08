---
name: agentic-test-pipeline-workflow
description: "The reusable 2-job GitHub Actions workflow (planner / authoring-execution) in this repo, and how to invoke it for a new requirements doc"
metadata: 
  node_type: memory
  type: project
  originSessionId: ""
  modified: 2026-09-08T00:00:00.000Z
---

`.github/workflows/agentic-test-pipeline.yml` on this repo (`MTI_agenticflow`, branch `master`) implements a planner → authoring → execution flow as a generic, reusable workflow, currently **2 jobs**: `planner`, then `authoring-execution` (authoring and execution were originally separate jobs but were merged to drop duplicated checkout/setup-node/install-kane-cli/login steps and an artifact hop that only existed to carry authoring's local meta forward — execution's `testrun run` now runs directly after authoring in the same workspace, so that meta is already on disk).

- **Trigger:** `workflow_dispatch` (manual, Actions tab) and `workflow_call` (so other workflows/repos can call it with `uses: <owner>/MTI_agenticflow/.github/workflows/agentic-test-pipeline.yml@master` + `secrets: inherit`).
- **Inputs:** `requirements_doc` (blank = auto-detects first `*.pdf`/`*.docx` in repo root; must be a repo-relative path, not a github.com URL — auto-detect will find `Login_Flow_PRD.pdf`), `generation_objective` (defaults to the standard positive/negative/edge prompt with login-specific URL/credential guidance), `test_language` (`javascript`/`python`, controls the kane-cli export step), `max_tests_to_author` (number, default `0` = no cap — caps how many of the planner's generated tests get authored+run), `kane_project_id`/`kane_folder_id` (both optional strings, blank by default — passed to `kane-cli login --project-id`/`--folder-id` when set; blank leaves kane-cli to auto-select the account default).
- **Required secrets:** `KANE_USERNAME`, `KANE_ACCESS_KEY`, `LT_USERNAME`, `LT_ACCESS_KEY` — add these under repo Settings > Secrets and variables > Actions before running.

**planner job** — `kane-cli generate <objective> --files <doc> --agent` (objective and `--save` cannot be combined in one call — CLI errors with "--save takes no objective; use --refine to add to a request"), then a separate `kane-cli generate --save --req <id> --agent` call, with the `request_id` parsed out of the first call's `generate_done` JSON line. Uploads `generated-tests` artifact (`*_test.md` files) and publishes a job summary. No Test Manager link is possible at this stage — a test case only gets a TMS testcase ID once it's authored/run in the next job.

**authoring-execution job** (single job, runs after planner):
1. Downloads `generated-tests`, selects which files to author (respects `max_tests_to_author` cap, writes `selected_tests.txt`).
2. **Authoring**: `kane-cli testmd run <file> --headless --agent --code-export --code-language <lang>` per selected test — one call both syncs the test to TMS *and* exports its automation code. `--headless` required: GitHub-hosted runners have no display server. Each call's output is `tee`'d to a temp file so one failing test doesn't abort the batch — failures logged as `::warning::`. Uploads `automation-code` artifact. Its `test_md_summary`/`test_md_done` JSON carries a real public `share_url` and `testcase_id` per test — captured into `authoring_links.tsv` and published as a job-summary table with both public and private links.
3. **Execution**: `kane-cli testrun run <file1> <file2> ... --headless --on-failure continue --name "..."` once over all selected tests. Requires each member to already have local meta (written by step 2's `testmd run`) — satisfied automatically since authoring ran earlier in the same job/workspace. Output captured to `testrun_output.ndjson`.
4. **Evidence + summary**: evidence pack at `.testmuai/evidence/<execution_id>.evidence`, uploaded as `test-run-evidence` artifact. `jq`-parsed markdown written to `$GITHUB_STEP_SUMMARY`.

**Does NOT execute the exported Playwright code directly** (`node test.js`) for the execution stage — LambdaTest rejects replays with `400 Bad Request: "Test case validation failed: bad request"` because the exported file's `testmu.configure({ tcId, build })` values are fixed at export time. kane-cli's own `testmd run`/`testrun run` work because they create a fresh session each call.

**PRD in use:** `Login_Flow_PRD.pdf` — the generation objective is extended with login-specific guidance ("make sure the URL, and rest details like username, password etc are correctly given to the test cases generated") so generated tests include concrete credentials/URLs from the PRD rather than placeholders.

**How to apply:** Trigger workflow from Actions tab (leave `requirements_doc` blank — auto-detect picks up `Login_Flow_PRD.pdf`). For a different PRD, drop the new PDF into the repo root and pass its name as `requirements_doc`. After the first run, update [[login-flow-suite]] with the real request ID from the planner job summary. See [[testcase-generation-flow]] for the equivalent local (non-CI) process.
