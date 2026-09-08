# Agentic Test Pipeline — MTI Login Flow

An AI-driven test pipeline: drop a requirements document (PDF/DOCX) into the repo, and a GitHub Actions workflow reads it, generates test cases, authors runnable automation code for them, and executes that automation against [LambdaTest](https://www.lambdatest.com/), publishing results as job summaries and downloadable artifacts.

Built on [`kane-cli`](https://www.testmuai.com/support/docs/kane-cli-introduction) (LambdaTest's AI test-authoring CLI).

This repo uses `Login_Flow_PRD.pdf` as the requirements document — covering positive, negative, and edge cases for the application login flow (valid credentials, invalid credentials, empty fields, locked accounts, session handling, etc.).

## How it works

The workflow (`.github/workflows/agentic-test-pipeline.yml`) has 2 jobs:

1. **`planner`** — reads the requirements document and generates test cases from it (`kane-cli generate`), covering positive, negative, and edge cases. Saves them as `*_test.md` files.
2. **`authoring-execution`** — one job, two stages back to back:
   - **Authoring**: runs each generated test once (`kane-cli testmd run --code-export`), which syncs it to LambdaTest Test Manager and exports standalone Playwright automation code (JavaScript or Python).
   - **Execution**: bundles all the authored tests into a single LambdaTest test run (`kane-cli testrun run`) and reports pass/fail results.

Every job publishes a **GitHub Actions job summary** (visible on the run page) and uploads artifacts you can download from the run.

```mermaid
flowchart TD
    doc["📄 Requirements doc<br/>(Login_Flow_PRD.pdf)"]

    subgraph job1["Job 1 · planner"]
        gen["kane-cli generate<br/>(positive / negative / edge cases)"]
        save["kane-cli generate --save"]
        gen --> save
    end

    subgraph job2["Job 2 · authoring-execution (same runner)"]
        direction TB
        select["Select tests<br/>(respects max_tests_to_author)"]
        subgraph authoring["Authoring — per test"]
            run1["kane-cli testmd run<br/>--code-export"]
        end
        subgraph execution["Execution — one bundled run"]
            run2["kane-cli testrun run<br/>(all selected tests)"]
        end
        select --> authoring --> execution
    end

    artTests[("🗂️ generated-tests<br/>artifact")]
    artCode[("🗂️ automation-code<br/>artifact")]
    artEvidence[("🗂️ test-run-evidence<br/>artifact")]
    sumPlanner["📋 Planner summary"]
    sumAuth["📋 Authoring summary<br/>public + private TM links"]
    sumExec["📋 Execution summary<br/>pass/fail per test"]
    lt["☁️ LambdaTest<br/>Test Manager"]

    doc --> gen
    save --> artTests
    save --> sumPlanner
    artTests --> select
    run1 --> artCode
    run1 --> sumAuth
    run1 -. syncs .-> lt
    run2 --> artEvidence
    run2 --> sumExec
    run2 -. runs on .-> lt
```

## One-time setup

### 1. Add repository secrets

Go to **Settings → Secrets and variables → Actions** on this repo and add:

| Secret | What it is |
|---|---|
| `KANE_USERNAME` | Your kane-cli / LambdaTest account username (same as `kane-cli login --username`) |
| `KANE_ACCESS_KEY` | Your kane-cli / LambdaTest access key (same as `kane-cli login --access-key`) |
| `LT_USERNAME` | LambdaTest username (used during execution) |
| `LT_ACCESS_KEY` | LambdaTest access key (used during execution) |

Without these, every run fails immediately at the `kane-cli login` step.

### 2. Requirements document

`Login_Flow_PRD.pdf` is already in the repo root. The pipeline auto-detects it — no extra setup needed. To use a different PRD, drop it in the repo root and pass its name as the `requirements_doc` input when triggering the workflow.

## Running the pipeline

Go to the **Actions** tab → **Agentic Test Pipeline** → **Run workflow**, and fill in the inputs (all optional):

| Input | Default | What it does |
|---|---|---|
| `requirements_doc` | *(blank → auto-detects `Login_Flow_PRD.pdf`)* | Path to the requirements document, **relative to the repo root** (e.g. `Login_Flow_PRD.pdf`) — not a github.com URL. |
| `generation_objective` | positive/negative/edge-case prompt with login URL/credential guidance | The instruction passed to `kane-cli generate`. Override this to steer what kind of coverage gets generated. |
| `test_language` | `javascript` | `javascript` or `python` — the language for the exported automation code. |
| `max_tests_to_author` | `0` (no cap) | Caps how many of the planner's generated tests actually get authored and run. Useful for a cheap smoke-test trigger before authoring the full suite. |
| `kane_project_id` | *(blank → account default)* | LambdaTest/kane-cli project ID. Leave blank to use your account's default project. |
| `kane_folder_id` | *(blank → account default)* | LambdaTest/kane-cli folder ID within the project. Leave blank to use the default folder. |

Click **Run workflow**. There's no automatic trigger on push/PR by default — planner and authoring each call a paid AI generation/execution API, so runs are manual-only to control cost. (You can add a `push`/`pull_request` trigger yourself once you're comfortable with the cost profile.)

## Reading the results

Each job's **Summary** (visible on the run page, or via **Actions → \[run\] → Summary**) shows:

- **Planner summary** — which document was used, the kane-cli request ID, and a note that `generated-tests` was uploaded as an artifact.
- **Authoring summary** — a table of every authored test with its real LambdaTest Test Manager links: a **public** link (no login required) and a **private** link (requires being logged into your LambdaTest account), plus the `automation-code` artifact.
- **Execution summary** — a pass/fail/duration table for every test in the bundled test run, overall totals, and the evidence pack (see below).

### Artifacts (download from the run page)

| Artifact | Contents |
|---|---|
| `generated-tests` | The raw `*_test.md` test case files |
| `automation-code` | Exported standalone Playwright automation code (`test.js`/`test.py` + deps) |
| `test-run-evidence` | The sealed evidence pack (`.evidence` file) from the bundled test run — view it locally with `kane-cli evidence serve <file>.evidence` |

**Note:** the bundled test run (`kane-cli testrun run`) doesn't produce a hosted LambdaTest dashboard link of its own — only the authoring stage's individual test runs do (that's where the public/private links in the authoring summary come from).

## Running locally

You can reproduce any stage on your own machine with `kane-cli` installed (`npm install -g @testmuai/kane-cli`) and logged in (`kane-cli login`):

```bash
# Generate test cases from the Login Flow PRD
kane-cli generate "generate test cases for the requirements in the attached document, covering positive, negative, and edge cases for each functional requirement, also make sure the URL, and rest details like username, password etc are correctly given to the test cases generated" \
  --files ./Login_Flow_PRD.pdf --agent
kane-cli generate --save --req <request_id> --agent

# Author + export one generated test
kane-cli testmd run .testmuai/tests/<suite>/<scenario>/<name>_test.md \
  --headless --agent --code-export --code-language javascript

# Bundle multiple authored tests into one LambdaTest test run
kane-cli testrun run <file1>_test.md <file2>_test.md --headless --on-failure continue

# View a downloaded evidence pack
kane-cli evidence serve <file>.evidence
```

## Reusing this pipeline in another repo

The workflow supports `workflow_call`, so another repo can invoke it directly instead of copy-pasting the YAML:

```yaml
jobs:
  tests:
    uses: <owner>/MTI_agenticflow/.github/workflows/agentic-test-pipeline.yml@master
    with:
      requirements_doc: Login_Flow_PRD.pdf
    secrets: inherit
```
