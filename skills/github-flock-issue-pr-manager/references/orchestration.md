# GitHub Issue PR Manager (Flock-Powered) Orchestration Reference

This reference documents the technical protocols, command templates, layout structures, and error-handling routines for orchestrating a GitHub issue using **Herdr** tabs/panes and **Flock** (`pc-tool flock`).

---

## 1. Architectural Model & Supervision Hierarchy

The **Main Issue Manager** acts as the **Flock Lead**. It coordinates planning, implementation, diff reviews, formal PR reviews, E2E testing, and CI validation.

### Strict Herdr Tab & Agent Pane Conventions

Every stage creates dedicated tabs and split-pane topologies matching `/github-issue-pr-manager`. Never create generic member tabs (`m1`, `m2`, etc.) via `flock start --members` or `flock add --members`. Panes are spawned in their canonical Herdr layouts and then adopted into Flock via `pc-tool flock adopt --pane <pane-id> --name m<N>`.

```text
HERDR TAB TOPOLOGY:

1. Stage 1 Tab: 'planning-<issue#>'
   +----------------------------------+----------------------------------+
   | plan-reviewer (left pane)        | imp-planner (right pane)         |
   | proclaude --model advanced-adaai | proclaude --model pro-adaai      |
   | (Flock: m1)                      | (Flock: m2)                      |
   +----------------------------------+----------------------------------+

2. Stage 2 Tab: 'implementation-<issue#>'
   +---------------------------------------+----------------------------------+
   |                                       | imp-coder-1                      |
   | code-reviewer (left column)           | proclaude --model lite-adaai   |
   | proclaude --model pro-adaai           | (Flock: m4)                      |
   | (Flock: m3)                           +----------------------------------+
   |                                       | imp-coder-2 (optional stacked)   |
   |                                       | proclaude --model lite-adaai   |
   +---------------------------------------+----------------------------------+

3. Stage 3 Main Tab:
   +---------------------------------------+----------------------------------+
   | main issue manager (Flock Lead)       | pr-reviewer (right pane)         |
   |                                       | proclaude --model advanced-adaai |
   |                                       | (Flock: m5)                      |
   +---------------------------------------+----------------------------------+

4. Stage 3 Dedicated Tab (only when E2E exists): 'e2e-<issue#>'
   +---------------------------------------------------------------------+
   | e2e-tester                                                          |
   | proclaude --model lite-adaai (Flock: m6)                            |
   +---------------------------------------------------------------------+
```

### Strict Workspace Confinement Rule
Every Herdr tab spawned by this workflow (`planning-<issue#>`, `implementation-<issue#>`, `e2e-<issue#>`) MUST be explicitly bound to the current workspace by passing `--workspace "$HERDR_WORKSPACE_ID"` to `herdr tab create`. Never invoke `herdr tab create` without `--workspace "$HERDR_WORKSPACE_ID"`. Omitting `--workspace` causes Herdr to create the tab in whichever workspace is currently active/focused on the server, which leaks sub-issue tabs into parent epic manager workspaces or unrelated workspaces. Always pass `--workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD"`.

---

## 2. Model Routing & Launcher Profiles

Launch every delegated agent with `proclaude` using verified catalog aliases:

| Role | Herdr Pane Label | Flock Name | Model Launcher | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `plan-reviewer` | `plan-reviewer` | `m1` | `proclaude --model advanced-adaai` | Adversarial plan review & acceptance criteria validation |
| `imp-planner` | `imp-planner` | `m2` | `proclaude --model pro-adaai` | Minimal sufficient implementation plan generation |
| `code-reviewer` | `code-reviewer` | `m3` | `proclaude --model pro-adaai` | Work decomposition, diff gating, commits, and pushes |
| `imp-coder-1` | `imp-coder-1` | `m4` | `proclaude --model lite-adaai` | Fast, minimal code implementation within strict write set |
| `imp-coder-2` | `imp-coder-2` | *(free mN)* | `proclaude --model lite-adaai` | Optional concurrent coder for disjoint write sets |
| `pr-reviewer` | `pr-reviewer` | `m5` | `proclaude --model advanced-adaai` | Formal GitHub PR review with inline line comments |
| `e2e-tester` | `e2e-tester` | `m6` | `proclaude --model lite-adaai` | End-to-end and integration test discovery & execution |

`lite-adaai` uses always-on thinking; do not pass `--effort`. It is the implementation-worker choice because it was the fastest available route in the benchmark while meeting the coding task's acceptance criteria.

---

## 3. Communication Contract & Zero-Polling Discipline

All coordination occurs exclusively through `pc-tool flock send`. Agents never type into other panes or use raw terminal injection.

### Zero-Polling Invariant
- **NEVER POLL TERMINAL PANES:** Do not run `sleep 15; herdr pane read` loops. Run `pc-tool flock wait` under the background monitor as a one-shot task (tracked background Bash task). The Lead is woken by Flock only on turn-ending reports (`done`, `review`, `blocked`, `question`). Catch-up is via `pc-tool flock next`. Rearm `pc-tool flock wait` after every wake whenever `pc-tool flock status` shows `lead listener: NONE`.
- **Lead Token Conservation:** The Lead should read only the orchestration reference sections needed for the current stage rather than reading the entire reference at once.
- **MANDATORY `--to lead` ON COMPLETION REPORTS:** Every terminal report (`done`, `review`, `blocked`, `question`) from a worker must explicitly specify `--to lead`:
  ```bash
  pc-tool flock send --to lead --kind done --ref "<ref>" --summary "<summary>" --body-file /tmp/report.md
  ```
  If `--to lead` is omitted, the report defaults to addressing the sender, meaning the Lead's wait listener is never woken and the workflow stalls.
- **Heartbeat & Tick Maintenance:** If `pc-tool flock status` displays `WARNING: the flock tick has not run for Nm`, run `pc-tool flock tick` to process internal tick actions, leases, and pending alerts.

### Message Kinds & Routing Rules

| Kind | Purpose | Wakes? | Ends Turn? |
| :--- | :--- | :--- | :--- |
| `assign` | Assigns an issue, slice, or task. May use `--take`, `--reassign`, `--review`, `--queue`, `--after`. | Yes | No |
| `progress` | Non-blocking status breadcrumb. | **No** | **No** |
| `done` | Task completed cleanly. | Yes | **Yes** |
| `review` | Task deliverable ready for review (surfaced as `ready-for-review`). | Yes | **Yes** |
| `blocked` | Execution blocked. Requires `--blocker <prose>`. | Yes | **Yes** |
| `question` | Clarification needed from lead. | Yes | **Yes** |
| `waiting` | Waiting on external process (CI, build). Use `--on ci:<sha> \| pr:<repo>#<n> \| review`. | No | **Yes** |
| `stop` | Urgent cancellation. Automatically urgent. Releases ref. | Yes | No |

### Multi-Line Body Discipline (`--body-file`)
Never type multi-line strings, code diffs, logs, or plans into `--body`. Shell quoting strips backticks and executes subshells. Always use `--body-file`:
```bash
pc-tool flock send --to m1 --kind assign --ref "#${ISSUE_NUMBER}" --summary "Plan specification" --body-file /tmp/plan-request.md
```

### Strict Anti-Overcomplication Directive
Every task assignment must include the following mandatory directive:
> **STRICT ANTI-OVERCOMPLICATION DIRECTIVE:** Deliver the minimal sufficient solution for the assigned scope: the smallest correct change consistent with repository conventions. Do not overcomplicate, over-abstract, or over-engineer. Do not introduce new layers, abstractions, helpers, wrappers, configuration, or dependencies unless strictly necessary to satisfy the acceptance criteria. Do not refactor, clean up, reformat, or "improve" code outside the assigned requirement. Defer all optional ideas; note them in the handoff instead of implementing them. Finish your assigned deliverable cleanly rather than broadening it.

### Notification Ownership
Only the **Main Issue Manager (Flock Lead)** is authorized to send user-facing completion notifications. Subordinates must never notify the user or use system/desktop alerts; they conclude their turns strictly with `pc-tool flock send`.

### Non-Intrusive State Transitions ("Done" to "Idle")
In Herdr, agents that finish background work enter the `done` state and are pinned to the top of the priority list.
**Do NOT use `herdr agent focus` or `herdr tab focus` to clear `done` status.** Focusing steals the user's active UI focus.
Instead, update lifecycle state non-intrusively via `herdr pane report-agent`:
```bash
herdr pane report-agent <pane-id> --source herdr:cli --agent <agent-label> --state idle
```

---

## 4. Stage 1: Planning Protocol

### 1. Tab & Pane Setup
Verify clean working tree. Rename branch to `issue-<number>-<short-description>`.
Set environment and record parent pane ID:
```bash
PARENT_PANE_ID="${HERDR_PANE_ID}"
FLOCK_DOCKER_LABEL="flock.session=${FLOCK_SESSION_ID:-issue-${ISSUE_NUMBER}}"
FLOCK_TMPDIR="/tmp"
```

Start Flock (omit `--epic` for plain issues):
```bash
pc-tool flock start --name lead
PARENT_TAB_ID="${HERDR_TAB_ID:-}"
test -n "$PARENT_TAB_ID" && herdr tab rename "$PARENT_TAB_ID" "#${ISSUE_NUMBER}"
# Start one-shot background wait task (rearmed after each wake when lead listener: NONE)
pc-tool flock wait
```

Spawn Herdr tab `planning-<issue#>` inside the current workspace:
```bash
PLAN_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "planning-${ISSUE_NUMBER}" --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus)
PLAN_TAB_ID=$(echo "$PLAN_TAB_JSON" | jq -r '.result.tab_id // .result.tab.tab_id')
PLAN_REV_PANE=$(echo "$PLAN_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
IMP_PLAN_PANE=$(herdr pane split "$PLAN_REV_PANE" --direction right --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')

herdr pane rename "$PLAN_REV_PANE" "plan-reviewer"
herdr pane rename "$IMP_PLAN_PANE" "imp-planner"

# Wait for interactive shell prompt before running launcher to prevent banner race
sleep 2

herdr pane run "$PLAN_REV_PANE" "proclaude --model advanced-adaai"
herdr pane run "$IMP_PLAN_PANE" "proclaude --model pro-adaai"

# Wait for agent to occupy each pane (bounded: at most 15 checks, 2s apart)
for p in "$PLAN_REV_PANE" "$IMP_PLAN_PANE"; do
  found=0
  for i in $(seq 1 15); do
    if herdr agent list | jq -e --arg p "$p" '.result.agents[]? | select(.pane_id == $p)' >/dev/null 2>&1; then
      found=1
      break
    fi
    sleep 2
  done
  if [ "$found" -ne 1 ]; then
    echo "ERROR: Agent failed to start in pane $p"
    herdr pane process-info "$p"
    exit 1
  fi
done

herdr agent rename "$PLAN_REV_PANE" "plan-reviewer"
herdr agent rename "$IMP_PLAN_PANE" "imp-planner"

# Confirm wrappers started before adopting into Flock
herdr pane process-info "$PLAN_REV_PANE"
herdr pane process-info "$IMP_PLAN_PANE"
```

### 2. Adoption into Flock & Tab Label Restoration (POST-JOIN)
```bash
pc-tool flock adopt --pane "$PLAN_REV_PANE" --name m1
pc-tool flock adopt --pane "$IMP_PLAN_PANE" --name m2

# Single read of flock status; stop and report if member missing (no unbounded polling loop)
FLOCK_STATUS=$(pc-tool flock status)
if ! echo "$FLOCK_STATUS" | grep -q "m1" || ! echo "$FLOCK_STATUS" | grep -q "m2"; then
  echo "ERROR: Member m1 or m2 missing from flock status:"
  echo "$FLOCK_STATUS"
  herdr pane process-info "$PLAN_REV_PANE"
  herdr pane process-info "$IMP_PLAN_PANE"
  exit 1
fi
herdr agent wait "$PLAN_REV_PANE" --until idle --timeout 60000
herdr agent wait "$IMP_PLAN_PANE" --until idle --timeout 60000
herdr tab rename "$PLAN_TAB_ID" "planning-${ISSUE_NUMBER}"
```

### 3. Iterative Plan Review Loop
The Lead sends the task to `m2` (`imp-planner`) and binds `m1` (`plan-reviewer`):
```bash
cat << 'EOF' > /tmp/plan-request-${ISSUE_NUMBER}.md
Analyze issue requirements, acceptance criteria, relevant files, risks, and test coverage.
Produce a minimal sufficient implementation plan. Adhere to repository conventions.
Include ordered file modifications and explicit validation steps.
EOF

pc-tool flock send --to m2 --kind assign --ref "#${ISSUE_NUMBER}" \
  --summary "Produce minimal implementation plan" \
  --body-file /tmp/plan-request-${ISSUE_NUMBER}.md

pc-tool flock send --to m1 --kind assign --review --ref "#${ISSUE_NUMBER}" \
  --summary "Adversarially review implementation plan from imp-planner"
```

`m1` and `m2` iterate autonomously over Flock:
1. `m2` submits `--kind review --body-file /tmp/plan-draft.md`.
2. `m1` reviews against repository code. Rejects overcomplication, unnecessary abstractions, or missing acceptance criteria.
3. Once approved, `m1` sends `--to lead --kind done --ref "#${ISSUE_NUMBER}"`.
4. Lead receives `done`, posts the approved plan to GitHub (`gh issue comment`), and updates both agents to `idle` via `herdr pane report-agent`.
5. Keep `planning-<issue#>` tab and panes open for inspection; do not kill them.

---

## 5. Stage 2: Implementation Protocol

### 1. Tab & Pane Setup
Spawn Herdr tab `implementation-<issue#>`:
```bash
IMPL_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "implementation-${ISSUE_NUMBER}" --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus)
IMPL_TAB_ID=$(echo "$IMPL_TAB_JSON" | jq -r '.result.tab_id // .result.tab.tab_id')
CODE_REV_PANE=$(echo "$IMPL_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
# Split right from code-reviewer to establish the right column for coders
CODER_1_PANE=$(herdr pane split "$CODE_REV_PANE" --direction right --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')

herdr pane rename "$CODE_REV_PANE" "code-reviewer"
herdr pane rename "$CODER_1_PANE" "imp-coder-1"

# Wait for interactive shell prompt before running launcher to prevent banner race
sleep 2

herdr pane run "$CODE_REV_PANE" "proclaude --model pro-adaai"
herdr pane run "$CODER_1_PANE" "proclaude --model lite-adaai"

# Wait for agent to occupy each pane (bounded: at most 15 checks, 2s apart)
for p in "$CODE_REV_PANE" "$CODER_1_PANE"; do
  found=0
  for i in $(seq 1 15); do
    if herdr agent list | jq -e --arg p "$p" '.result.agents[]? | select(.pane_id == $p)' >/dev/null 2>&1; then
      found=1
      break
    fi
    sleep 2
  done
  if [ "$found" -ne 1 ]; then
    echo "ERROR: Agent failed to start in pane $p"
    herdr pane process-info "$p"
    exit 1
  fi
done

herdr agent rename "$CODE_REV_PANE" "code-reviewer"
herdr agent rename "$CODER_1_PANE" "imp-coder-1"

# Confirm wrappers started before adopting into Flock
herdr pane process-info "$CODE_REV_PANE"
herdr pane process-info "$CODER_1_PANE"

# If concurrent coders are needed: ALWAYS split downward from CODER_1_PANE (right column).
# NEVER split downward from CODE_REV_PANE — code-reviewer must remain full-height on the left.
# CODER_2_PANE=$(herdr pane split "$CODER_1_PANE" --direction down --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')
# herdr pane rename "$CODER_2_PANE" "imp-coder-2"
# sleep 2
# herdr pane run "$CODER_2_PANE" "proclaude --model lite-adaai"
# found=0; for i in $(seq 1 15); do if herdr agent list | jq -e --arg p "$CODER_2_PANE" '.result.agents[]? | select(.pane_id == $p)' >/dev/null 2>&1; then found=1; break; fi; sleep 2; done
# herdr agent rename "$CODER_2_PANE" "imp-coder-2"
# herdr pane process-info "$CODER_2_PANE"

pc-tool flock adopt --pane "$CODE_REV_PANE" --name m3
pc-tool flock adopt --pane "$CODER_1_PANE" --name m4

# Single read of flock status; stop and report if member missing (no unbounded polling loop)
FLOCK_STATUS=$(pc-tool flock status)
if ! echo "$FLOCK_STATUS" | grep -q "m3" || ! echo "$FLOCK_STATUS" | grep -q "m4"; then
  echo "ERROR: Member m3 or m4 missing from flock status:"
  echo "$FLOCK_STATUS"
  herdr pane process-info "$CODE_REV_PANE"
  herdr pane process-info "$CODER_1_PANE"
  exit 1
fi
herdr agent wait "$CODE_REV_PANE" --until idle --timeout 60000
herdr agent wait "$CODER_1_PANE" --until idle --timeout 60000
herdr tab rename "$IMPL_TAB_ID" "implementation-${ISSUE_NUMBER}"
```

### 2. Decomposition & Implementation
1. The Lead assigns the implementer `m4` (`imp-coder-1`) as owner of `#${ISSUE_NUMBER}` using `--reassign`, then binds `m3` (`code-reviewer`) as the review gate using `--review`. Owner assign must come first:
   ```bash
   cat << 'EOF' > /tmp/imp-request-${ISSUE_NUMBER}.md
   Implement minimal sufficient changes for the approved plan.
   Stay within assigned write set. Do not refactor unrelated code.
   EOF

   pc-tool flock send --to m4 --kind assign --ref "#${ISSUE_NUMBER}" --reassign \
     --summary "Implement minimal sufficient changes for #${ISSUE_NUMBER}" \
     --body-file /tmp/imp-request-${ISSUE_NUMBER}.md

   pc-tool flock send --to m3 --kind assign --review --ref "#${ISSUE_NUMBER}" \
     --summary "Review diffs from imp-coder-1 and gate commits"
   ```
2. If concurrent disjoint workstreams exist, split `CODER_1_PANE` downward to spawn `imp-coder-2`, adopt as `m5`, and assign with a distinct ref/slice.
3. Coders implement assigned changes, review their own diff, generate a self-review, and send `--kind review --body-file /tmp/self-review.md`.
4. `m3` reviews each diff. Once approved, `m3` creates scoped git commits, pushes the branch, and reports `--to lead --kind done --ref "#${ISSUE_NUMBER}"`.
5. Lead updates `code-reviewer` and coders to `idle` via `herdr pane report-agent`. Keep `implementation-<issue#>` open.
6. Lead publishes the PR in **draft** status:
   ```bash
   gh pr create --draft --title "Fix #${ISSUE_NUMBER}: <title>" --body "Fixes #${ISSUE_NUMBER}"
   ```

---

## 6. Stage 3: Concurrent PR Review, E2E Testing, & CI

Once the draft PR exists, review, required CI, and E2E (only when present) proceed concurrently.

### E2E discovery gate

Before creating an E2E tab or agent, the Lead itself inspects the current PR head across the repository, including relevant monorepo packages: repository instructions and test documentation, CI workflows, package scripts and other build/test targets, test directories, framework configuration, and E2E or system-integration compose manifests. Examples include Playwright, Cypress, Puppeteer, Selenium, `docker-compose.e2e.yml`, `compose.e2e.yml`, `test:e2e`, `pytest tests/e2e`, and `make e2e`. Inspect candidates to distinguish actual suites or runnable commands from mere dependencies or incidental mentions; do not rely on one filename or tool.

Record the head SHA and the evidence/paths and commands found. If no suite exists, record `E2E: skipped (no suite)` with the evidence and proceed without creating, launching, adopting, assigning, waiting for, or supervising an E2E tab or agent (`m6`). If discovery is inconclusive (for example, inaccessible files or ambiguous configuration), investigate or escalate rather than claim a skip. An existing but broken or unexecutable suite is not absent; treat it as a validation failure or blocker. A skip never substitutes for required CI or PR review.

Repeat discovery for each new PR head, including after fixes and rebases. If a suite appears, create the agent and run it; if one remains, rerun it against the new head; if it disappears, record the new head's skip and do not wait on or start E2E work. Retain any previously spawned tab idle for audit.

### 1. Formal PR Review (`pr-reviewer` in Main Tab)
```bash
PR_REV_PANE=$(herdr pane split "$PARENT_PANE_ID" --direction right --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')
herdr pane rename "$PR_REV_PANE" "pr-reviewer"

sleep 2
herdr pane run "$PR_REV_PANE" "proclaude --model advanced-adaai"

found=0
for i in $(seq 1 15); do
  if herdr agent list | jq -e --arg p "$PR_REV_PANE" '.result.agents[]? | select(.pane_id == $p)' >/dev/null 2>&1; then
    found=1
    break
  fi
  sleep 2
done
if [ "$found" -ne 1 ]; then
  echo "ERROR: Agent failed to start in pane $PR_REV_PANE"
  herdr pane process-info "$PR_REV_PANE"
  exit 1
fi

herdr agent rename "$PR_REV_PANE" "pr-reviewer"
herdr pane process-info "$PR_REV_PANE"

pc-tool flock adopt --pane "$PR_REV_PANE" --name m5

# One owner per ref: assign pr#${PR_NUMBER} to m5
pc-tool flock send --to m5 --kind assign --ref "pr#${PR_NUMBER}" \
  --summary "Conduct formal PR review with inline comments"
```
`m5` inspects the PR diff via `gh pr diff` and submits a formal GitHub review with inline comments on changed lines.

**Review Self-Approval & Gate Rule:** If the reviewer identity is the author of the PR, GitHub disallows `APPROVE`; submit a `COMMENT` review stating that no actionable findings remain. The formal review gate is "no unresolved blocking, high, or medium actionable findings", not strictly "approved".

### 2. E2E Testing Protocol (`e2e-tester` in Tab `e2e-<issue#>`)

Run this entire step only if the Lead's discovery gate found a suite; otherwise skip it. Write the identified paths, commands, and required task instructions to `/tmp/e2e-request-${ISSUE_NUMBER}.md` for the tester.
```bash
E2E_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "e2e-${ISSUE_NUMBER}" --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus)
E2E_TAB_ID=$(echo "$E2E_TAB_JSON" | jq -r '.result.tab_id // .result.tab.tab_id')
E2E_PANE=$(echo "$E2E_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
herdr pane rename "$E2E_PANE" "e2e-tester"

sleep 2
herdr pane run "$E2E_PANE" "proclaude --model lite-adaai"

found=0
for i in $(seq 1 15); do
  if herdr agent list | jq -e --arg p "$E2E_PANE" '.result.agents[]? | select(.pane_id == $p)' >/dev/null 2>&1; then
    found=1
    break
  fi
  sleep 2
done
if [ "$found" -ne 1 ]; then
  echo "ERROR: Agent failed to start in pane $E2E_PANE"
  herdr pane process-info "$E2E_PANE"
  exit 1
fi

herdr agent rename "$E2E_PANE" "e2e-tester"
herdr pane process-info "$E2E_PANE"

pc-tool flock adopt --pane "$E2E_PANE" --name m6

# Single read of flock status; stop and report if member missing (no unbounded polling loop)
FLOCK_STATUS=$(pc-tool flock status)
if ! echo "$FLOCK_STATUS" | grep -q "m6"; then
  echo "ERROR: Member m6 missing from flock status:"
  echo "$FLOCK_STATUS"
  herdr pane process-info "$E2E_PANE"
  exit 1
fi
herdr agent wait "$E2E_PANE" --until idle --timeout 60000
herdr tab rename "$E2E_TAB_ID" "e2e-${ISSUE_NUMBER}"

# One owner per ref: assign e2e:#${PR_NUMBER} to m6 (distinct ref from m5)
pc-tool flock send --to m6 --kind assign --ref "e2e:#${PR_NUMBER}" \
  --summary "Run discovered E2E / integration test suites" \
  --body-file "/tmp/e2e-request-${ISSUE_NUMBER}.md"
```
- `m6` runs the identified headless suite with containers labeled `$FLOCK_DOCKER_LABEL` (defaults to `e2e-${ISSUE_NUMBER}`) and `$TMPDIR` default `/tmp`. Cleans up afterwards with `docker compose down -v`.
- If no suite exists, the Lead records the skip directly; there is no `m6` report to await.

### 3. PR CI Discovery & Batch Diagnosis Protocol
- If `.woodpecker/pr.yaml` exists, monitor pipeline. (Woodpecker CI is optional if no manifest exists).
- If CI compose files exist (`docker-compose.ci.yml`), execute local container checks.
- **Handling Woodpecker CI 401 / 403 (User not authorized / expired token):** If `wpci` returns `HTTP 401` or `HTTP 403: User not authorized` and pipeline logs cannot be fetched via CLI:
  1. Check aggregate check status with `gh pr checks "$PR_NUMBER"`.
  2. Check if an ephemeral container runner (e.g. `podman run --rm -v $(pwd):/repo:Z ...`) can execute the failing test step locally using the exact CI image.
  3. If container execution is unavailable or prohibited, immediately escalate to the user with the direct Woodpecker pipeline URL and request the failed step logs or refreshed credentials. Do not guess failures blindly.
- If checks fail: **Do not fix failures one-by-one.**
  1. Lead collects all failing logs for the current SHA into `/tmp/ci-report-<sha>.md`.
  2. Lead sends one consolidated fix task to `m3` (`code-reviewer`):
     ```bash
     pc-tool flock send --to m3 --kind assign --ref "#${ISSUE_NUMBER}" \
       --summary "Fix CI batch failures for SHA ${SHA}" \
       --body-file /tmp/ci-report-<sha>.md
     ```
  3. `m3` assigns fixes to coders; coders fix all issues in the batch; `m3` reviews and creates **exactly one** CI-fix commit and pushes **once**.

### 4. Readiness Transition
- Once formal PR review and required CI pass, and E2E passes if present (or is recorded as skipped if absent), convert the PR from draft to open:
  ```bash
  gh pr ready "$PR_NUMBER"
  ```
- If `pc-tool flock review-quality` is run and reports that the repository has no wiring row in `fleet/flock/quality-verb.ts`, state "quality record unavailable for this repo" in the report. Do not write a `none:unwired` record to fake the gate.
- Do not rely on `pc-tool flock ready` for external repositories that are not registered in Flock's internal `MERGE_METHOD` registry. `gh pr ready` is the authoritative GitHub transition.

---

## 7. Stage 4: Post-Validation `origin/main` Rebase Gate

After PR review and CI checks pass, and E2E passes if present (or is recorded as skipped if absent), on the latest head commit:
1. Mark PR ready for review: `gh pr ready "$PR_NUMBER"`.
2. Fetch default branch: `git fetch origin main`.
3. Check divergence:
   ```bash
   git merge-base --is-ancestor origin/main HEAD || DIVERGED=1
   ```
4. If diverged:
   - Record pre-rebase commit reference.
   - Rebase feature branch onto `origin/main`.
   - Resolve conflicts semantically.
   - Force-push with lease: `git push --force-with-lease origin "$CURRENT_BRANCH"`.
   - **Restart Stage 3:** Rediscover E2E for the rebased head; rerun review, CI, and E2E if present, or record E2E as skipped if absent.

---

## 8. Stage 5: Teardown, Cleanup, & Finish

Finish only when the PR is open (ready for review), CI is passing, review findings are resolved, and the post-validation divergence check passes:
1. Clean up temporary containers and resources:
   ```bash
   pc-tool flock cleanup --dry-run
   # Report cleanup findings. Run --converge only with owner's approval or durable authorization:
   pc-tool flock cleanup --converge
   ```
2. Close the flock session:
   ```bash
   pc-tool flock close
   ```
3. Output the single completion report to the user. All Herdr tabs remain open for inspection.
   ```
2. Close the flock session:
   ```bash
   pc-tool flock close
   ```
3. Output the single completion report to the user. All Herdr tabs remain open for inspection.
