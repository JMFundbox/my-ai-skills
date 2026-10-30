---
name: github-flock-epic-issue-manager
description: Manage a GitHub epic issue in coordination with Herdr and github-flock-issue-pr-manager. Executes in an isolated epic worktree, keeps main strictly unaltered, drafts an epic PR targeting main, spawns Herdr sub-issue worktrees targeting the epic branch driven by /github-flock-issue-pr-manager, merges completed PRs into the epic branch, syncs only the epic worktree, and runs the final Epic PR Stage via a specialized Flock validation team.
---

# GitHub Epic Issue Manager (Flock-Powered)

The **GitHub Epic Issue Manager** orchestrates high-level epic/umbrella issues across Herdr workspaces. It coordinates isolated worktrees, delegates individual code-change sub-issues to subordinate agents driven by `/github-flock-issue-pr-manager`, merges approved PRs into the epic issue's branch, synchronizes **only** the epic issue's worktree, and runs a Flock-powered validation team for the parent Epic PR following exact Herdr tab and pane conventions.

---

## 1. Core Principles & Safety Guardrails

1. **`main` Branch Remains Strictly Unaltered**: The `main` (default) branch is **never** checked out, modified, committed to, or merged into during this workflow. All work happens on the epic branch and sub-issue branches. The transition to `main` happens solely via the GitHub PR review interface.
2. **Worktree & Branch Hygiene**: The epic runs on `epic-<number>-<short-description>`, tracking `origin`.
3. **Parent Epic PR Targets Main**: A draft parent PR is opened for the epic branch targeting `main`.
4. **Sub-Issue PRs Target Epic Branch**: Every sub-issue worktree branches off `$EPIC_BRANCH`, and its PR targets and merges into `$EPIC_BRANCH`.
5. **Sync Epic Worktree Only**: When a sub-issue PR merges, synchronize **only the epic worktree** (`git pull --ff-only origin "$EPIC_BRANCH"`).
6. **Task Classification**: This skill **only** manages sub-issues that involve **code changes** destined for PRs. It skips **live execution** sub-issues (deployments, running migrations in production, manual QA, ops tasks).
7. **Strict Herdr Tab & Pane Conventions**:
   - Each sub-issue gets a Herdr worktree workspace branching from `$EPIC_BRANCH`.
   - Initial tab is renamed to `#<issue-number>`.
   - Root pane is renamed to `sub-issue-<issue-number>`.
   - Child agent runs `/github-flock-issue-pr-manager` (`proclaude --model advanced-adaai`).
8. **Epic PR Stage via Flock**: Once all code changes are merged (code-change stop condition met), it executes the final PR stage on the parent Epic PR using the exact layout:
   - Tab 1: Must be named `epic-${EPIC_NUMBER}` (never `m1` or `epic-lead`). Left pane: `epic-lead`, right pane: `pr-reviewer` (`proclaude --model advanced-adaai`).
   - Tab 2: Dedicated tab named `e2e-${EPIC_NUMBER}` (never `m2`) with `e2e-tester` (`proclaude --model lite-adaai`).
   - Panes adopted into Flock for non-blocking coordination.
   - **MANDATORY POST-JOIN TAB RENAME**: When panes run `pc-tool flock adopt`, `flock join` asynchronously renames tabs to member names (`m1`, `m2`). The manager MUST re-assert and rename Tab 1 to `epic-${EPIC_NUMBER}` and Tab 2 to `e2e-${EPIC_NUMBER}` after agents join and before entering `pc-tool flock wait`.
9. **Teardown Finished Worktrees**: Once a sub-issue PR merges into the epic branch, close and remove the child Herdr worktree (`herdr worktree remove`).
10. **Sub-Issue PR Validation Ownership**: Sub-issue CI/Woodpecker (`wpci`) monitoring, test checks, and fix cycles belong strictly to the subordinate `/github-flock-issue-pr-manager` agent inside its own worktree. The Epic Manager MUST NOT poll, tail, inspect logs for, or diagnose Woodpecker CI (`wpci`) on sub-issue PRs. The Epic Manager only performs high-level passive monitoring (Herdr agent status `idle`/`done` and `isDraft: false` on the PR). CI and Woodpecker inspection only enters the Epic Manager's scope during the parent Epic PR Stage for the parent Epic PR targeting main.
11. **Strict Prohibition on Manual Sub-Issue Worktree Implementation by Epic Manager**: The Epic Manager is an orchestrator, NOT an implementer. The Epic Manager MUST NEVER `cd` into a sub-issue worktree directory, edit code files, write test patches, run test suites (`pytest`, `uv run`), execute `git commit`, or push commits for a sub-issue. All sub-issue implementation and review MUST occur strictly through the subordinate agent (`sub-issue-<N>`) spawned inside its Herdr worktree running `/github-flock-issue-pr-manager`. The Epic Manager interacts with sub-issues SOLELY via Herdr commands (`herdr agent prompt`, `herdr pane read`, `herdr worktree remove`) and GitHub CLI (`gh pr merge`).
12. **Strict Prohibition on e2e-tester Modifying Code**: `e2e-tester` has ZERO write authority on repository code or tests. `e2e-tester` MUST NEVER edit production files, patch test files, write new tests, or execute git commits. It is strictly a test runner and reporter. If tests fail or formal review requests changes during the Epic PR Stage, the Epic Manager MUST spawn/respawn dedicated implementation agents (`code-reviewer` and `imp-coder-1`) in an implementation tab (`implementation-epic-${EPIC_NUMBER}`) to implement, review, and commit the fixes.

---

## 2. Prerequisites & Workspace Setup

Verify Herdr and Git environment:
```bash
test "${HERDR_ENV:-}" = 1 || { echo "ERROR: Not running inside Herdr (HERDR_ENV != 1)."; exit 1; }
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "ERROR: Not inside Git repository."; exit 1; }
gh auth status >/dev/null 2>&1 || { echo "ERROR: GitHub CLI not authenticated."; exit 1; }
```

### Initialize Epic Branch & Draft Parent PR
```bash
EPIC_WORKTREE_ROOT="$(git rev-parse --show-toplevel)"
DEFAULT_BRANCH="$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || echo "main")"
CURRENT_BRANCH="$(git branch --show-current)"

# Verify clean tree
test -z "$(git status --porcelain)" || { echo "ERROR: Working tree dirty."; exit 1; }

# Format branch name: epic-<number>-<slug>
EPIC_TITLE="$(gh issue view "$EPIC_NUMBER" --json title -q .title)"
SHORT_SLUG="$(echo "$EPIC_TITLE" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-' | sed 's/^-//;s/-$//' | cut -c1-30)"
DESIRED_BRANCH="epic-${EPIC_NUMBER}-${SHORT_SLUG}"

if [ "$CURRENT_BRANCH" = "$DEFAULT_BRANCH" ]; then
  git checkout -b "$DESIRED_BRANCH"
elif [ "$CURRENT_BRANCH" != "$DESIRED_BRANCH" ]; then
  git branch -m "$DESIRED_BRANCH"
fi
EPIC_BRANCH="$(git branch --show-current)"

# Push and track
if ! git rev-parse --abbrev-ref @{u} >/dev/null 2>&1; then
  git push -u origin "$EPIC_BRANCH"
fi

# Draft Epic PR targeting default branch (main remains untouched)
EPIC_PR_NUMBER=$(gh pr list --head "$EPIC_BRANCH" --base "$DEFAULT_BRANCH" --json number -q '.[0].number')
if [ -z "$EPIC_PR_NUMBER" ] || [ "$EPIC_PR_NUMBER" = "null" ]; then
  EPIC_PR_NUMBER=$(gh pr create --draft \
    --base "$DEFAULT_BRANCH" \
    --head "$EPIC_BRANCH" \
    --title "Epic #${EPIC_NUMBER}: ${EPIC_TITLE}" \
    --body "Tracking epic issue #${EPIC_NUMBER}.\n\nAll code-change sub-issue PRs will target this branch and merge here before opening for final review.")
fi
```

---

## 3. Sub-Issue Extraction & Execution Loop

### 3.1 Classify Sub-Issues
Parse the epic issue checklist (`- [ ]`). Classify each unchecked item:
- **Code Change (IN SCOPE)**: Produces repository commits/PR.
- **Live Execution (OUT OF SCOPE)**: Marked with `[live]`, `[manual]`, `[deploy]`, or labels `deploy`, `ops`, `live-execution`. Skip automated dispatch for these tasks.

### 3.2 Dispatch Available Code-Change Sub-Issues
For each available code-change sub-issue `#<ISSUE_NUM>` (concurrency limit: 2–3):

1. **Create Herdr Worktree**:
   ```bash
   BRANCH_NAME="issue-${ISSUE_NUM}"
   WORKTREE_JSON=$(herdr worktree create \
     --workspace "$PARENT_WORKSPACE_ID" \
     --base "$EPIC_BRANCH" \
     --branch "$BRANCH_NAME" \
     --label "#${ISSUE_NUM}" \
     --no-focus \
     --json)
   TARGET_WORKSPACE_ID=$(echo "$WORKTREE_JSON" | jq -r '.result.workspace.workspace_id // .result.workspace_id')
   TARGET_TAB_ID=$(echo "$WORKTREE_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
   TARGET_PANE_ID=$(echo "$WORKTREE_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')

   # Rename initial tab to #<issue-number>
   herdr tab rename "$TARGET_TAB_ID" "#${ISSUE_NUM}"
   ```
2. **Start Sub-Manager Agent**:
   ```bash
   herdr pane rename "$TARGET_PANE_ID" "sub-issue-${ISSUE_NUM}"
   herdr pane run "$TARGET_PANE_ID" "proclaude --model advanced-adaai"
   herdr agent wait "sub-issue-${ISSUE_NUM}" --until idle --timeout 60000 2>/dev/null || sleep 8
   ```
3. **Dispatch Goal**:
   ```bash
   herdr agent prompt "$TARGET_PANE_ID" "/goal use /github-flock-issue-pr-manager to complete mergeable PR for issue #${ISSUE_NUM} targeting branch ${EPIC_BRANCH}. CRITICAL WORKSPACE ISOLATION: Ensure all spawned tabs (planning, implementation, e2e) are created strictly inside this sub-issue workspace by passing --workspace \"\$HERDR_WORKSPACE_ID\" to herdr tab create."
   ```

### 3.3 Monitor & Merge Completed PRs

**Strict Prohibition on Sub-Issue wpci Monitoring**: Sub-issue CI/Woodpecker (`wpci`) monitoring, pipeline inspection, and fix cycles belong strictly to the subordinate `/github-flock-issue-pr-manager` agent inside its own worktree. The Epic Manager MUST NOT poll `wpci` or tail Woodpecker pipeline logs for child sub-issue PRs.

The Epic Manager periodically inspects child worktrees passively:
1. **Herdr Agent Status**: `herdr agent get "sub-issue-${ISSUE_NUM}"` (looking for `idle`/`done`).
2. **High-Level PR State**: `gh pr list --head "issue-${ISSUE_NUM}" --base "$EPIC_BRANCH" --json number,title,state,isDraft,mergeable,reviewDecision`.

A sub-issue PR is ready for merge when:
- `isDraft` is `false` (marked ready by the subordinate agent after its reviews and CI pass).
- `mergeable` is `MERGEABLE`.
- `reviewDecision` is `APPROVED` or clean.
- The child agent is settled (`idle` or `done`).

When ready:
1. Merge the sub-issue PR:
   ```bash
   PR_NUMBER=$(gh pr list --head "issue-${ISSUE_NUM}" --base "$EPIC_BRANCH" --json number -q '.[0].number')
   gh pr merge "$PR_NUMBER" --squash --delete-branch --subject "Merge PR #${PR_NUMBER} for issue #${ISSUE_NUM} into ${EPIC_BRANCH}"
   ```
2. Fast-forward pull **only the epic worktree**:
   ```bash
   cd "$EPIC_WORKTREE_ROOT"
   git fetch origin "$EPIC_BRANCH"
   git checkout "$EPIC_BRANCH"
   git pull --ff-only origin "$EPIC_BRANCH"
   ```
3. Update checklist in the epic issue body (`- [x] #<ISSUE_NUM>`).
4. Teardown completed worktree:
   ```bash
   herdr worktree remove --workspace "$TARGET_WORKSPACE_ID" --force --json
   ```

---

## 4. Epic PR Stage (Executed via Flock After Stop Condition)

When **all code-change sub-issues are merged**, execute the Epic PR Stage against the parent PR targeting `$DEFAULT_BRANCH` using a dedicated Flock validation team adhering to exact Herdr conventions:

1. **Start Validation Flock in Epic Worktree**:
   ```bash
   cd "$EPIC_WORKTREE_ROOT"
   pc-tool flock start --name epic-lead --epic "#${EPIC_NUMBER}"
   PARENT_TAB_ID="${HERDR_TAB_ID:-}"
   test -n "$PARENT_TAB_ID" && herdr tab rename "$PARENT_TAB_ID" "epic-${EPIC_NUMBER}"
   ```
2. **Spawn Panes According to Exact Layout**:
   ```bash
   # 1. Main tab: split right for pr-reviewer
   PR_REV_PANE=$(herdr pane split "$PARENT_PANE_ID" --direction right --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')
   herdr pane rename "$PR_REV_PANE" "pr-reviewer"
   herdr pane run "$PR_REV_PANE" "proclaude --model advanced-adaai"
   pc-tool flock adopt --pane "$PR_REV_PANE" --name m1

   # 2. Dedicated tab e2e-<epic#> for e2e-tester
   E2E_TAB_JSON=$(herdr tab create --workspace "$PARENT_WORKSPACE_ID" --cwd "$PWD" --label "e2e-${EPIC_NUMBER}" --no-focus --json)
   E2E_TAB_ID=$(echo "$E2E_TAB_JSON" | jq -r '.result.tab_id // .result.tab.tab_id')
   E2E_PANE=$(echo "$E2E_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
   herdr pane rename "$E2E_PANE" "e2e-tester"
   herdr pane run "$E2E_PANE" "proclaude --model lite-adaai"
   pc-tool flock adopt --pane "$E2E_PANE" --name m2

   # CRITICAL POST-JOIN TIMING:
   # flock join (run by m1 and m2 on startup) automatically overwrites Herdr tab labels with 'm1' and 'm2'.
   # You MUST wait for both m1 and m2 to complete startup join before re-asserting canonical tab names!
   herdr agent wait "pr-reviewer" --until idle --timeout 60000 2>/dev/null || sleep 6
   herdr agent wait "e2e-tester" --until idle --timeout 60000 2>/dev/null || sleep 6

   # Re-assert canonical tab names:
   test -n "$PARENT_TAB_ID" && herdr tab rename "$PARENT_TAB_ID" "epic-${EPIC_NUMBER}"
   herdr tab rename "$E2E_TAB_ID" "e2e-${EPIC_NUMBER}"

   # Now start background flock wait
   pc-tool flock wait &
   ```
3. **Concurrent Validation via Flock**:
   - `m1` (`pr-reviewer`, Advanced AdaAI) reviews the complete diff against `$DEFAULT_BRANCH` and submits formal GitHub review with inline comments. Subordinates MUST always send completion reports with `--to epic-lead`.
   - `m2` (`e2e-tester`, Lite AdaAI) executes containerized E2E test suites (with containers labeled `$FLOCK_DOCKER_LABEL`) and tears them down.
     - **STRICT READ-ONLY RESTRICTION**: `e2e-tester` has ZERO write authority on repository code or tests. `e2e-tester` MUST NEVER edit production code, modify test files, or write new test scripts. If tests fail, `e2e-tester` captures complete untruncated failure traces and logs, and reports them to `epic-lead` via `pc-tool flock send --to epic-lead --kind done --ref "<ref>"`.
   - Lead monitors required CI / status checks. If Woodpecker CI fails and `wpci` returns 401, reproduce locally in an ephemeral container or escalate to user with the direct build URL.
   - Update agents to `idle` non-intrusively via `herdr pane report-agent`. Keep tabs open.

4. **Code Fixes & Remediation on the Epic Branch (Implementation Agents Required)**:
   If formal PR review findings (`pr-reviewer`), CI failures, or E2E tests (`e2e-tester`) reveal code defects, missing behavior, or test failures:
   - **DO NOT allow `e2e-tester` or `pr-reviewer` to edit code or write fixes.**
   - The Epic Manager MUST spawn (or re-activate) a dedicated implementation tab in the epic workspace:
     ```bash
     IMPL_TAB_JSON=$(herdr tab create --workspace "$PARENT_WORKSPACE_ID" --cwd "$PWD" --label "implementation-epic-${EPIC_NUMBER}" --no-focus --json)
     IMPL_TAB_ID=$(echo "$IMPL_TAB_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
     CODE_REV_PANE=$(echo "$IMPL_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
     CODER_1_PANE=$(herdr pane split "$CODE_REV_PANE" --direction right --no-focus --json | jq -r '.result.pane_id // .result.pane.pane_id')

     herdr pane rename "$CODE_REV_PANE" "code-reviewer"
     herdr pane run "$CODE_REV_PANE" "proclaude --model pro-adaai"

     herdr pane rename "$CODER_1_PANE" "imp-coder-1"
     herdr pane run "$CODER_1_PANE" "proclaude --model lite-adaai"

     pc-tool flock adopt --pane "$CODE_REV_PANE" --name m3
     pc-tool flock adopt --pane "$CODER_1_PANE" --name m4

     herdr agent wait "code-reviewer" --until idle --timeout 60000 2>/dev/null || sleep 6
     herdr agent wait "imp-coder-1" --until idle --timeout 60000 2>/dev/null || sleep 6
     herdr tab rename "$IMPL_TAB_ID" "implementation-epic-${EPIC_NUMBER}"
     ```
   - The Epic Manager compiles a consolidated failure report into `/tmp/ci-report-epic-${EPIC_NUMBER}.md` and dispatches it to `m3` (`code-reviewer`).
   - `m3` assigns the fix tasks to `m4` (`imp-coder-1`) with explicit write boundaries and the strict anti-overcomplication directive.
   - `m4` implements the fix, reviews its own diff, and reports to `m3`.
   - `m3` reviews the diff, creates **exactly one cohesive commit**, and pushes to `$EPIC_BRANCH`.
   - `m3` reports `--kind done` to `epic-lead`.
   - The Epic Manager then re-dispatches `pr-reviewer` and `e2e-tester` to re-validate the new head SHA.

5. **Mark Ready for Review**:
   Once review, E2E tests, and CI checks pass:
   ```bash
   gh pr ready "$EPIC_PR_NUMBER"
   ```
   (Use `gh pr ready`, not `pc-tool flock ready`, for external/non-hardcoded repositories).
6. **Post-Validation `origin/main` Rebase Gate**:
   Fetch `origin/$DEFAULT_BRANCH`. If diverged, rebase `$EPIC_BRANCH` onto `origin/$DEFAULT_BRANCH`, push with `--force-with-lease`, and re-verify.
7. **Teardown Validation Flock**:
   ```bash
   pc-tool flock cleanup --converge
   pc-tool flock close
   ```

---

## 5. Stopping Points

### Stopping Point A: Only Live Execution Sub-Issues Remain
If all code changes are merged and the Epic PR is open, but live execution tasks (`[live]`) remain unchecked:
- Do **not** close the epic issue.
- Present a clear handoff report listing the open Epic PR URL, verified commit SHA, and the list of remaining live execution tasks for human operator execution.

### Stopping Point B: All Sub-Issues Complete
When **all** sub-issues (code changes and live tasks) are resolved:
- Close the epic issue:
  ```bash
  gh issue close "$EPIC_NUMBER" --comment "All sub-issues completed. Epic PR #${EPIC_PR_NUMBER} verified and open for merge into ${DEFAULT_BRANCH}."
  ```
- Present a structured summary of completed PRs, test results, and final commit SHA.
