# GitHub Epic Issue Manager (Flock-Powered) Orchestration Reference

This reference documents protocols and operational routines for orchestrating GitHub epic issues with Herdr and Flock-powered child managers.

---

## 1. Sub-Issue Parsing & Classification Protocols

### 1.1 Parsing Checklists & Hierarchies
- **Standard Markdown Checklists**:
  Regex for unchecked items: `- \[ \] (?:https?:\/\/[^\/]+\/[^\/]+\/[^\/]+\/issues\/|#)([0-9]+)`
- **Native GitHub Sub-Issues API**:
  `gh issue view <EPIC_NUM> --json subIssues,items`
- **Dependencies**: Check issue bodies for `Depends on #<num>`, `Prerequisite: #<num>`, `Blocked by #<num>`.

### 1.2 Classification (Code Change vs Live Execution)
Every sub-issue must be classified before any worktree is spawned:
- **Code Change Tasks (IN SCOPE)**: Produce code, tests, docs, or config in the repo, destined for a PR.
- **Live Execution Tasks (OUT OF SCOPE)**: Deployments, live database schema migrations, manual QA, DNS updates, secret rotations.

**Detection Rules for Live Execution:**
- Labels: `live-execution`, `live`, `deploy`, `deployment`, `ops`, `release`, `manual`, `manual-qa`, `infra-ops`, `maintenance`.
- Epic section headers matching `### Deployment`, `### Live Execution`, `### Manual Operations`, or tags `[live]`, `[manual]`, `[deploy]`, `[ops]`.
- Issues focusing on environment mutations rather than repository PRs.

---

## 2. Epic Worktree Initialization & Branch Hygiene

When the manager starts:
1. Verify working tree is clean: `git status --porcelain`.
2. Determine default branch: `DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || echo "main")`.
3. Format epic branch: `epic-${EPIC_NUMBER}-${SHORT_SLUG}`.
4. If on default branch, switch to new epic branch: `git checkout -b "$EPIC_BRANCH"`.
5. If already on another branch, rename: `git branch -m "$EPIC_BRANCH"`.
6. Push and track: `git push -u origin "$EPIC_BRANCH"`.
7. Draft parent Epic PR targeting default branch:
   ```bash
   EPIC_PR_NUMBER=$(gh pr list --head "$EPIC_BRANCH" --base "$DEFAULT_BRANCH" --json number -q '.[0].number')
   if [ -z "$EPIC_PR_NUMBER" ] || [ "$EPIC_PR_NUMBER" = "null" ]; then
     EPIC_PR_NUMBER=$(gh pr create --draft \
       --base "$DEFAULT_BRANCH" \
       --head "$EPIC_BRANCH" \
       --title "Epic #${EPIC_NUMBER}: ${EPIC_TITLE}" \
       --body "Tracking epic issue #${EPIC_NUMBER}.\n\nAll code-change sub-issue PRs will target this branch and merge here before opening for final review.")
   fi
   ```
*(Note: `main` / `$DEFAULT_BRANCH` remains completely untouched).*

---

## 3. Sub-Issue Worktree & Sub-Manager Lifecycle

For each available code-change sub-issue `#<ISSUE_NUM>`:

### Strict Prohibition on Manual Sub-Issue Implementation
The Epic Manager MUST NEVER `cd` into sub-issue worktrees (e.g. `.herdr/worktrees/.../issue-<N>`), edit source files, write patch scripts, run tests, or commit/push code for sub-issues directly. All implementation and validation MUST be executed by the subordinate `/github-flock-issue-pr-manager` agent inside its Herdr worktree. The Epic Manager coordinates purely through Herdr CLI commands and GitHub PR merges.

### 3.1 Spawn Isolated Herdr Worktree
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

# Rename initial Herdr tab to "#<issue-number>"
herdr tab rename "$TARGET_TAB_ID" "#${ISSUE_NUM}"
```

### 3.2 Launch Child Flock Manager
Launch the agent in the new worktree pane with Advanced AdaAI:
```bash
herdr pane rename "$TARGET_PANE_ID" "sub-issue-${ISSUE_NUM}"
herdr pane run "$TARGET_PANE_ID" "proclaude --model advanced-adaai"
herdr agent wait "sub-issue-${ISSUE_NUM}" --until idle --timeout 60000 2>/dev/null || sleep 8
```

Submit the goal command explicitly instructing the sub-manager to use `/github-flock-issue-pr-manager` and target `$EPIC_BRANCH`, with workspace isolation enforced:
```bash
herdr agent prompt "$TARGET_PANE_ID" "/goal use /github-flock-issue-pr-manager to complete mergeable PR for issue #${ISSUE_NUM} targeting branch ${EPIC_BRANCH}. CRITICAL WORKSPACE ISOLATION: Ensure all spawned tabs (planning, implementation, e2e) are created strictly inside this sub-issue workspace by passing --workspace \"\$HERDR_WORKSPACE_ID\" to herdr tab create."
```

### 3.3 PR Merge & Worktree Teardown

**Sub-Issue Monitoring Boundary (No Sub-Issue wpci)**:
The Epic Manager monitors child worktrees purely through passive Herdr agent status (`idle`/`done`) and high-level PR status (`gh pr list --head "issue-${ISSUE_NUM}" --base "$EPIC_BRANCH" --json number,title,state,isDraft,mergeable,reviewDecision`).
**Strict Boundary**: Sub-issue CI/Woodpecker (`wpci`) pipeline monitoring, log inspection, and build-fail fixes are exclusively handled by `/github-flock-issue-pr-manager` inside the child worktree. The Epic Manager MUST NOT run `wpci` or poll Woodpecker pipelines for sub-issue PRs. When the child agent successfully validates the PR, it will mark the PR ready for review (`isDraft: false`).

When the sub-issue PR targeting `$EPIC_BRANCH` is approved, green, and open:
1. Merge the PR:
   ```bash
   PR_NUMBER=$(gh pr list --head "issue-${ISSUE_NUM}" --base "$EPIC_BRANCH" --json number -q '.[0].number')
   gh pr merge "$PR_NUMBER" --squash --delete-branch --subject "Merge PR #${PR_NUMBER} for issue #${ISSUE_NUM} into ${EPIC_BRANCH}"
   ```
2. Fast-forward pull **only** the epic worktree:
   ```bash
   cd "$EPIC_WORKTREE_ROOT"
   git fetch origin "$EPIC_BRANCH"
   git checkout "$EPIC_BRANCH"
   git pull --ff-only origin "$EPIC_BRANCH"
   ```
3. Update epic task list in issue body (`- [x] #<ISSUE_NUM>`).
4. Remove the finished sub-issue Herdr worktree:
   ```bash
   herdr worktree remove --workspace "$TARGET_WORKSPACE_ID" --force --json
   ```

---

## 4. Epic PR Stage (Executed via Flock After Stop Condition)

When **all code-change sub-issues are merged**, execute the Epic PR Stage against the parent PR targeting `$DEFAULT_BRANCH` using a dedicated Flock validation team conforming to Herdr tab conventions:

### 4.1 Launch Validation Flock & Panes
```bash
# 1. Start Flock session in the Epic worktree
cd "$EPIC_WORKTREE_ROOT"
pc-tool flock start --name epic-lead --epic "#${EPIC_NUMBER}"
PARENT_TAB_ID="${HERDR_TAB_ID:-}"
test -n "$PARENT_TAB_ID" && herdr tab rename "$PARENT_TAB_ID" "epic-${EPIC_NUMBER}"

# 2. Main tab: split right for pr-reviewer (Advanced AdaAI, always-on thinking)
PR_REV_PANE=$(herdr pane split "$PARENT_PANE_ID" --direction right --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')
herdr pane rename "$PR_REV_PANE" "pr-reviewer"
herdr pane run "$PR_REV_PANE" "proclaude --model advanced-adaai"
pc-tool flock adopt --pane "$PR_REV_PANE" --name m1

# 3. Dedicated tab e2e-<epic#> for e2e-tester (Lite AdaAI, always-on thinking)
E2E_TAB_JSON=$(herdr tab create --workspace "$PARENT_WORKSPACE_ID" --cwd "$PWD" --label "e2e-${EPIC_NUMBER}" --no-focus --json)
E2E_TAB_ID=$(echo "$E2E_TAB_JSON" | jq -r '.result.tab_id // .result.tab.tab_id')
E2E_PANE=$(echo "$E2E_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
herdr pane rename "$E2E_PANE" "e2e-tester"
herdr pane run "$E2E_PANE" "proclaude --model lite-adaai"
pc-tool flock adopt --pane "$E2E_PANE" --name m2

# CRITICAL POST-JOIN TIMING:
# flock join automatically overwrites Herdr tab labels with 'm1' and 'm2'.
# Wait for both m1 and m2 to complete startup join before re-asserting canonical tab names!
herdr agent wait "pr-reviewer" --until idle --timeout 60000 2>/dev/null || sleep 6
herdr agent wait "e2e-tester" --until idle --timeout 60000 2>/dev/null || sleep 6

# Re-assert canonical tab names:
test -n "$PARENT_TAB_ID" && herdr tab rename "$PARENT_TAB_ID" "epic-${EPIC_NUMBER}"
herdr tab rename "$E2E_TAB_ID" "e2e-${EPIC_NUMBER}"

# Now start background flock wait
pc-tool flock wait &
```

### 4.2 Concurrent Validation
- **Formal Review**: Lead assigns PR review to `m1`. `m1` reviews the full epic diff against `$DEFAULT_BRANCH` and submits formal review with inline comments. Subordinates must conclude turns using `--to epic-lead`.
- **E2E Testing**: Lead assigns E2E tests to `m2`. `m2` executes containerized E2E suites and cleans up containers.
  - **STRICT PROHIBITION ON E2E-TESTER MODIFYING CODE**: `e2e-tester` has ZERO write authority on repository code or tests. `e2e-tester` MUST NEVER edit production code, modify test files, or write new test scripts. In the prompt to `e2e-tester`, explicitly include:
    > "You are strictly a test executor and validator. You have ZERO write authority on repository code or tests. Do NOT modify any files, do NOT write patches, and do NOT create new test files. If tests fail, capture untruncated failure traces and report them back. Code fixes are strictly performed by dedicated implementation agents."
- **CI Validation**: Lead monitors GitHub status checks and Woodpecker CI (if manifest exists). If Woodpecker CI fails and `wpci` returns 401, reproduce locally in an ephemeral container matching the CI image or request logs from the user.
- **Report Idle**: Lead marks agents as `idle` via `herdr pane report-agent` non-intrusively. All tabs remain open for inspection.

### 4.3 Code Fixes & Remediation on the Epic Branch (Implementation Agents Required)
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

### 4.4 Epic PR Ready Transition
Once review, E2E tests, and CI pass:
```bash
gh pr ready "$EPIC_PR_NUMBER"
```
(Use `gh pr ready`, not `pc-tool flock ready`, for external/non-hardcoded repositories).

### 4.5 Post-Validation `origin/main` Rebase Gate
1. Fetch default branch: `git fetch origin "$DEFAULT_BRANCH"`.
2. Check divergence: `git merge-base --is-ancestor origin/$DEFAULT_BRANCH HEAD || DIVERGED=1`.
3. If diverged, rebase `$EPIC_BRANCH` onto `origin/$DEFAULT_BRANCH`, push with `--force-with-lease`, and restart validation.

### 4.6 Teardown Validation Flock
```bash
pc-tool flock cleanup --converge
pc-tool flock close
```

---

## 5. Stopping Points

### Stopping Point A: Live Execution Tasks Remain
If all code-change sub-issues are merged and Epic PR is open, but live execution tasks remain unchecked:
- Do **not** close the epic issue.
- Output a clear handoff report listing the open Epic PR URL and the remaining live execution steps for human action.

### Stopping Point B: Epic Fully Complete
If all tasks (both code changes and live execution) are complete:
- Close the epic issue:
  ```bash
  gh issue close "$EPIC_NUMBER" --comment "All sub-issues completed. Epic PR #${EPIC_PR_NUMBER} has passed formal review, E2E testing, and CI, and is ready for review/merge. Epic issue closed."
  ```
- Present concise final report to the user.
