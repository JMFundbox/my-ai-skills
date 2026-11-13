---
name: github-flock-issue-pr-manager
description: Manage a GitHub issue through planning, implementation, independent review, PR publication, and passing CI using supervised Herdr agents coordinated via Flock. Matches /github-issue-pr-manager's tab and pane topologies, creating an E2E tab only when the repository has E2E tests.
---

# GitHub Issue PR Manager (Flock-Powered)

Own the requested issue through a review-ready, CI-green pull request. Do not merge unless the user separately requests it.

This workflow requires Herdr and Flock (`pc-tool flock`). Read [references/orchestration.md](references/orchestration.md) before creating tabs, panes, or delegated agents. If `HERDR_ENV` is not `1`, explain that orchestration cannot start from the current pane and stop before delegation.

---

## 1. Manager Ownership & Herdr Topology Rules

The main issue manager runs in the main worktree/tab as the **Flock Lead**. It coordinates planning, implementation, diff reviews, formal PR reviews, E2E testing, and CI validation.

### Strict Herdr Tab & Agent Pane Conventions (Must Match `/github-issue-pr-manager`)
Do **not** create generic member tabs (`m1`, `m2`, etc.) via `flock start --members` or `flock add --members`. Instead, spawn the exact Herdr tabs and split-pane layouts required by each stage, label them with canonical role names, and adopt them into Flock using `pc-tool flock adopt --pane <pane-id> --name m<N>`:

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

### Critical: Post-Join Tab Rename Lifecycle Rule
When a pane is adopted with `pc-tool flock adopt`, `proclaude` starts up and asynchronously executes `pc-tool flock join` 3–6 seconds later. Flock's internal `join` routine (`membership-verb.ts:690 -> labelCallerTab`) **automatically renames the Herdr tab to the member's Flock name (`m1`, `m2`, `m3`, `m4`, `m6`)**.
- Calling `herdr tab rename` immediately after `flock adopt` creates a race condition where the tab is renamed before the child process runs `join`, causing `flock join` to overwrite the tab name seconds later.
- Note that `herdr pane rename` sets a pane label, not an agent name. Always also call `herdr agent rename <pane-id> <name>` so Herdr can resolve the agent.
- **MANDATORY SEQUENCE**:
  1. Spawn tab and panes, rename panes with `herdr pane rename`.
  2. Wait 2 seconds for the interactive shell prompt to appear before launching proclaude (prevents shell login banners from dropping the input).
  3. Run `proclaude` with `herdr pane run`.
  4. Wait for an agent to occupy the pane (at most 15 checks, 2s apart, reading `herdr agent list`). If it never appears, stop and report with `herdr pane process-info <pane-id>`.
  5. Set the agent name with `herdr agent rename <pane-id> <name>` and inspect `herdr pane process-info <pane-id>`.
  6. Run `pc-tool flock adopt --pane <pane-id> --name m<N>`.
  7. Read `pc-tool flock status` once; if the member row (`m<N>`) is missing, stop and report it with `herdr pane process-info`.
  8. Wait on the pane ID directly (`herdr agent wait <pane-id> --until idle --timeout 60000`).
  9. Call `herdr tab rename <tab-id> <canonical-name>` **AFTER** agents have joined.
  10. As a safety net, re-verify and enforce canonical tab names before entering `pc-tool flock wait`.

### Strict Workspace Confinement Rule
Every Herdr tab spawned by this workflow (`planning-<issue#>`, `implementation-<issue#>`, `e2e-<issue#>`) MUST be explicitly bound to the current workspace by passing `--workspace "$HERDR_WORKSPACE_ID"` to `herdr tab create`. Never invoke `herdr tab create` without `--workspace "$HERDR_WORKSPACE_ID"`. Omitting `--workspace` causes Herdr to create the tab in whichever workspace is currently active/focused on the server, which leaks sub-issue tabs into parent epic manager workspaces or unrelated workspaces. Always pass `--workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD"`.

### Tab & Pane Preservation Invariant
Do **not** close, kill, or destroy spawned Herdr tabs or agent panes upon stage transitions.
- The `planning-<issue#>` tab and its panes (`plan-reviewer`, `imp-planner`) remain open and idle after Stage 1 finishes.
- The `implementation-<issue#>` tab and its panes (`code-reviewer`, `imp-coder-*`) remain open and idle throughout Stages 2, 3, and subsequent stages.
- The main tab's `pr-reviewer` pane remains open after review completion.
- The `e2e-<issue#>` tab and its `e2e-tester` pane remain open and idle after test execution for audit and log inspection.

### Non-Intrusively Updating Agents from "Done" to "Idle"
In Herdr, agents that finish background work enter the `done` state and are pinned to the top of the priority list.
**Do NOT use `herdr agent focus` or `herdr tab focus` to clear `done` status** (focusing steals the user's active UI focus).
Instead, update lifecycle state non-intrusively via `herdr pane report-agent`:
```bash
herdr pane report-agent <pane-id> --source herdr:cli --agent <agent-label> --state idle
```

---

## 2. Model Profiles (Catalog Aliases)

All delegated agents are launched with `proclaude` using verified catalog aliases:

| Role | Herdr Pane Label | Flock Name | Model Launcher | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **`plan-reviewer`** | `plan-reviewer` | `m1` | `proclaude --model advanced-adaai` | Adversarial plan review & acceptance criteria validation |
| **`imp-planner`** | `imp-planner` | `m2` | `proclaude --model pro-adaai` | Minimal sufficient implementation plan generation |
| **`code-reviewer`** | `code-reviewer` | `m3` | `proclaude --model pro-adaai` | Work decomposition, diff gating, commits, and pushes |
| **`imp-coder-1`** | `imp-coder-1` | `m4` | `proclaude --model lite-adaai` | Fast, minimal code implementation within strict write set |
| **`imp-coder-2`** | `imp-coder-2` | *(free mN)* | `proclaude --model lite-adaai` | Optional concurrent coder for disjoint write sets |
| **`pr-reviewer`** | `pr-reviewer` | `m5` | `proclaude --model advanced-adaai` | Formal GitHub PR review with inline line comments |
| **`e2e-tester`** | `e2e-tester` | `m6` | `proclaude --model lite-adaai` | End-to-end and integration test discovery & execution |

`lite-adaai` uses always-on thinking; do not pass `--effort`. It is the implementation-worker choice because it was the fastest available route in the benchmark while meeting the coding task's acceptance criteria.

---

## 3. Communication Contract & Zero-Polling Discipline

All inter-agent communication flows through Flock's SQLite turn bus (`pc-tool flock send`).
- **NEVER POLL TERMINAL PANES:** Do not run `sleep 15; herdr pane read` loops. Run `pc-tool flock wait` under the background monitor. The Lead is woken by Flock only on turn-ending reports (`done`, `review`, `blocked`, `question`). Catch-up is via `pc-tool flock next`.
- **MANDATORY `--to lead` ON COMPLETION REPORTS:**
  Subordinate agents reporting turn conclusions (`done`, `review`, `blocked`, `question`) must ALWAYS explicitly specify `--to lead`:
  ```bash
  pc-tool flock send --to lead --kind done --ref "<ref>" --summary "<summary>" --body-file <file>
  ```
  Never omit `--to lead`, otherwise the report addresses the sender and the Lead is never woken.
- **Always use `--body-file`:** Never pass multi-line plans, diffs, or logs via `--body`. Write to a file under `/tmp/` and pass `--body-file`.
- **Mandatory Anti-Overcomplication Directive:** Inject the directive into every task prompt sent via Flock.
- **Single Notification Ownership:** Delegated agents report only through `pc-tool flock send`. Only the main issue manager sends the final completion notification to the user.
- **Heartbeat Maintenance:** If `pc-tool flock status` reports that the flock tick has not run, execute `pc-tool flock tick` to advance timers, latches, and pending alerts.
- **Lead Token Conservation:** The Lead should read only the orchestration reference sections needed for the current stage rather than reading the entire reference at once.
- **Rearm Lead Listener:** Run one-shot `pc-tool flock wait` as a tracked background Bash task (using `run_in_background: true`). Rearm it after every wake, whenever `pc-tool flock status` shows `lead listener: NONE`.

---

## 4. Required 5-Stage Workflow

### Stage 1: Planning (`planning-<issue#>`)

1. **Setup & Start Flock**:
   - Verify clean working tree. Rename branch to `issue-<number>-<short-description>`.
   - Set environment and record parent pane ID:
     ```bash
     PARENT_PANE_ID="${HERDR_PANE_ID}"
     FLOCK_DOCKER_LABEL="flock.session=${FLOCK_SESSION_ID:-issue-${ISSUE_NUMBER}}"
     FLOCK_TMPDIR="/tmp"
     ```
   - Start Flock (omit `--epic` for plain issues):
     ```bash
     pc-tool flock start --name lead
     # CRITICAL: flock start renames the current tab to "lead". Restore the canonical tab name:
     PARENT_TAB_ID="${HERDR_TAB_ID:-}"
     test -n "$PARENT_TAB_ID" && herdr tab rename "$PARENT_TAB_ID" "#${ISSUE_NUMBER}"
     # One-shot wait in background (rearm after every wake if lead listener: NONE)
     pc-tool flock wait
     ```
2. **Spawn Herdr Tab `planning-<issue#>`**:
   ```bash
   PLAN_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "planning-${ISSUE_NUMBER}" --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus)
   PLAN_TAB_ID=$(echo "$PLAN_TAB_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
   PLAN_REV_PANE=$(echo "$PLAN_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
   IMP_PLAN_PANE=$(herdr pane split "$PLAN_REV_PANE" --direction right --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')

   herdr pane rename "$PLAN_REV_PANE" "plan-reviewer"
   herdr pane rename "$IMP_PLAN_PANE" "imp-planner"

   # Wait for shell prompt before running launcher to prevent banner race
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
3. **Adopt into Flock & Restore Canonical Tab Label (POST-JOIN)**:
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
4. **Dispatch Plan Task**:
   - Write issue requirements and acceptance criteria to `/tmp/plan-request-${ISSUE_NUMBER}.md`.
   - Send to `m2` (`imp-planner`):
     ```bash
     pc-tool flock send --to m2 --kind assign --ref "#${ISSUE_NUMBER}" \
       --summary "Produce minimal implementation plan" \
       --body-file /tmp/plan-request-${ISSUE_NUMBER}.md
     ```
   - Bind `m1` (`plan-reviewer`) as the reviewer:
     ```bash
     pc-tool flock send --to m1 --kind assign --review --ref "#${ISSUE_NUMBER}" \
       --summary "Review implementation plan from imp-planner"
     ```
5. **Plan Iteration & Approval**:
   - `m2` submits `--kind review --body-file /tmp/plan-draft.md`.
   - `m1` reviews adversarially against the codebase. Once satisfied, `m1` reports `--to lead --kind done --ref "#${ISSUE_NUMBER}"`.
   - Lead posts the approved plan to the GitHub issue (`gh issue comment`).
   - Lead marks `m1` and `m2` as `idle` via `herdr pane report-agent`. Keep `planning-<issue#>` open.

### Stage 2: Implementation (`implementation-<issue#>`)

1. **Spawn Herdr Tab `implementation-<issue#>`**:
   ```bash
   IMPL_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "implementation-${ISSUE_NUMBER}" --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus)
   IMPL_TAB_ID=$(echo "$IMPL_TAB_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
   CODE_REV_PANE=$(echo "$IMPL_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
   # Split right from code-reviewer to create the right column (imp-coder-1)
   CODER_1_PANE=$(herdr pane split "$CODE_REV_PANE" --direction right --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')

   herdr pane rename "$CODE_REV_PANE" "code-reviewer"
   herdr pane rename "$CODER_1_PANE" "imp-coder-1"

   # Wait for shell prompt before running launcher to prevent banner race
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

   # If concurrent coders are needed: ALWAYS split downward from the right-column coder (CODER_1_PANE).
   # NEVER split downward from CODE_REV_PANE — code-reviewer must remain full-height in the left column.
   # CODER_2_PANE=$(herdr pane split "$CODER_1_PANE" --direction down --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')
   # herdr pane rename "$CODER_2_PANE" "imp-coder-2"
   # sleep 2
   # herdr pane run "$CODER_2_PANE" "proclaude --model lite-adaai"
   # found=0; for i in $(seq 1 15); do if herdr agent list | jq -e --arg p "$CODER_2_PANE" '.result.agents[]? | select(.pane_id == $p)' >/dev/null 2>&1; then found=1; break; fi; sleep 2; done
   # herdr agent rename "$CODER_2_PANE" "imp-coder-2"
   # herdr pane process-info "$CODER_2_PANE"
   ```
2. **Adopt into Flock & Restore Canonical Tab Label (POST-JOIN)**:
   ```bash
   pc-tool flock adopt --pane "$CODE_REV_PANE" --name m3
   pc-tool flock adopt --pane "$CODER_1_PANE" --name m4
   # If imp-coder-2 was spawned:
   # pc-tool flock adopt --pane "$CODER_2_PANE" --name m5

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
3. **Decomposition & Implementation**:
   - The Lead assigns the implementer `m4` (`imp-coder-1`) as owner of `#${ISSUE_NUMBER}` using `--reassign`, then binds `m3` (`code-reviewer`) as the review gate using `--review`. Owner assign must come first:
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
   - `m4` writes code within its write set, generates a self-review, and reports `--kind review --body-file /tmp/self-review.md`.
   - `m3` reviews the diff. Once approved, `m3` creates scoped commits, pushes the branch, and reports `--to lead --kind done --ref "#${ISSUE_NUMBER}"`.
   - Lead marks `m3` and `m4` as `idle` via `herdr pane report-agent`. Keep `implementation-<issue#>` open.
4. **Draft PR Publication**:
   - Lead opens the PR in draft status:
     ```bash
     gh pr create --draft --title "Fix #${ISSUE_NUMBER}: <title>" --body "Fixes #${ISSUE_NUMBER}"
     ```

### Stage 3: PR Review, E2E Testing, & CI

Before any E2E tab, agent, or Flock member is created, the Lead performs the [E2E discovery gate](references/orchestration.md#e2e-discovery-gate) on the current PR head. If no suite exists, record E2E as skipped with evidence and do not create, launch, adopt, assign, wait for, or supervise `e2e-tester` / `m6`. Continue PR review and required CI independently. Repeat discovery on each new head, including after fixes and rebases.

1. **Spawn `pr-reviewer` in Main Tab**:
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
   ```
2. **Only if the discovery gate found a suite:** spawn dedicated tab `e2e-<issue#>` and restore its canonical label (POST-JOIN). Otherwise skip this entire step:
   ```bash
   E2E_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "e2e-${ISSUE_NUMBER}" --env "FLOCK_DOCKER_LABEL=$FLOCK_DOCKER_LABEL" --env "TMPDIR=$FLOCK_TMPDIR" --no-focus)
   E2E_TAB_ID=$(echo "$E2E_TAB_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
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
   ```
3. **Concurrent Validation & CI Policy**:
   - **One owner per ref:** Assign distinct refs to concurrent validation agents:
     - Lead assigns formal PR review to `m5` (`pr-reviewer`) with ref `pr#${PR_NUMBER}`:
       ```bash
       pc-tool flock send --to m5 --kind assign --ref "pr#${PR_NUMBER}" \
         --summary "Conduct formal PR review with inline comments"
       ```
     - Only when a suite exists, Lead writes the discovered paths, commands, and required task instructions to `/tmp/e2e-request-${ISSUE_NUMBER}.md` and assigns E2E tests to `m6` (`e2e-tester`) with ref `e2e:#${PR_NUMBER}`:
       ```bash
       pc-tool flock send --to m6 --kind assign --ref "e2e:#${PR_NUMBER}" \
         --summary "Run discovered E2E / integration test suites" \
         --body-file "/tmp/e2e-request-${ISSUE_NUMBER}.md"
       ```
   - **Review Self-Approval & Gate Rule:** `m5` inspects diff and submits formal review on GitHub with inline comments. If the reviewer identity is the author of the PR, GitHub disallows `APPROVE`; submit a `COMMENT` review stating that no actionable findings remain. The formal review gate is "no unresolved blocking, high, or medium actionable findings", not strictly "approved".
   - **E2E Testing:** If a suite exists, `m6` runs it and cleans up containers labeled `$FLOCK_DOCKER_LABEL` (defaults to `e2e-${ISSUE_NUMBER}`). Otherwise the Lead records `skipped` without spawning `m6`; this does not block PR readiness.
   - **CI Monitoring & Woodpecker 401 / 403:**
     - Lead monitors required CI checks via `gh pr checks "$PR_NUMBER"` and Woodpecker CI.
     - If `wpci` returns `HTTP 401` or `HTTP 403: User not authorized` and pipeline logs cannot be retrieved via CLI:
       - Check aggregate check status with `gh pr checks`.
       - Check if Docker/Podman is available to reproduce the CI test step in an ephemeral container matching the CI image (e.g. `python:3.12-slim`), OR
       - If container execution is unavailable, immediately alert the user with the direct Woodpecker pipeline URL and request the failed step traceback or refreshed credentials. Never guess test failures blindly.
4. **Batch CI Fix Protocol**:
   - If CI, E2E, or reviews reveal failures: collect all failures for the latest head SHA into `/tmp/ci-report-<sha>.md`.
   - Send one consolidated fix task to `m3` (`code-reviewer`).
   - `code-reviewer` assigns fixes to coders, integrates changes, and produces **exactly one** CI-fix commit and pushes **once**.
   - Lead reports agents as `idle` via `herdr pane report-agent`.
5. **Marking PR Ready for Review**:
   - Once all criteria pass, mark the PR ready for review via `gh pr ready "$PR_NUMBER"`.
   - If `pc-tool flock review-quality` is run and reports that the repository has no wiring row in `fleet/flock/quality-verb.ts`, state "quality record unavailable for this repo" in the report. Do not write a `none:unwired` record to fake the gate.
   - Do not rely on `pc-tool flock ready` for external repos that lack a hardcoded entry in Flock's internal `MERGE_METHOD` map.

### Stage 4: Post-Validation `origin/main` Rebase Gate

Once formal PR review, E2E tests (if present; otherwise recorded as skipped), and required CI all pass on the latest head commit and the PR is marked ready for review:
1. Fetch default branch: `git fetch origin main` (or `staging` as applicable).
2. Check divergence:
   ```bash
   git merge-base --is-ancestor origin/main HEAD || DIVERGED=1
   ```
3. If diverged:
   - Rebase feature branch onto the target base branch.
   - Resolve conflicts semantically.
   - Force-push with lease: `git push --force-with-lease origin "$CURRENT_BRANCH"`.
   - **Restart Stage 3:** Rediscover E2E for the rebased head; rerun review, CI, and E2E if present, or record E2E as skipped if absent.

### Stage 5: Teardown, Cleanup, & Finish

Finish only when PR is open (ready for review), CI is green, reviews are clean, and divergence check passes:
1. Clean up containers and temporary resources:
   ```bash
   pc-tool flock cleanup --dry-run
   # Report cleanup findings. Run --converge only with owner's approval or durable authorization:
   pc-tool flock cleanup --converge
   ```
2. Close the flock session:
   ```bash
   pc-tool flock close
   ```
3. Output the single completion report to the user (branch, PR URL, head SHA, review outcome, test/CI results, and E2E result or skip with evidence). All spawned Herdr tabs remain open for inspection.
