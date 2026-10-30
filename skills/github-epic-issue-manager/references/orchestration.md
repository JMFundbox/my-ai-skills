# GitHub Epic Issue Manager Orchestration Reference

This reference documents technical protocols, command templates, and error-handling routines for orchestrating GitHub epic issues with Herdr.

## 1. Sub-Issue Parsing Protocols

GitHub issues use various conventions for epic tracking. The manager must parse:

1. **Standard Markdown Checklists**:
   ```markdown
   - [ ] #101 Implement database schema
   - [ ] #102 Add API controller
   - [x] #100 Initial scaffolding
   ```
   Regex for unchecked items: `- \[ \] (?:https?:\/\/[^\/]+\/[^\/]+\/[^\/]+\/issues\/|#)([0-9]+)`

2. **Hierarchical / Sub-Issues API**:
   If the repository uses GitHub's native issue hierarchy:
   ```bash
   gh issue view <EPIC_NUM> --json subIssues,items
   ```

3. **Explicit Dependency Annotations**:
   Check issue bodies for notes like:
   - `Depends on #101`
   - `Prerequisite: #101`
   - `Blocked by #101`
   Block child tasks until their prerequisites are merged.

## 2. Classification of Sub-Issues (Code Change vs Live Execution)

The Epic Manager strictly manages **code changes**. Sub-issues must be filtered before any worktree is spawned:

- **Code Change Tasks (Processed)**: Features, bugfixes, refactoring, unit/integration test additions, CI config tweaks, documentation in the repo.
- **Live Execution Tasks (Ignored/Skipped)**: Deployments to environments, executing database migrations against live databases, running manual smoke tests, DNS changes, secret rotation, ops procedures.

### Detection Criteria
An issue is classified as Live Execution if:
- It has labels: `live-execution`, `live`, `deploy`, `deployment`, `ops`, `release`, `manual`, `manual-qa`, `infra-ops`, `maintenance`.
- It is located under an epic section header matching `Deployment`, `Live Execution`, `Manual`, or has inline markers `[live]`, `[manual]`, `[deploy]`, `[ops]`.
- Its scope centers on live environments rather than a repository PR.

### Stopping Condition on Live Execution
If the only remaining unchecked sub-issues are classified as live execution tasks (all code-change sub-issues are merged into `$EPIC_BRANCH`):
1. Execute the Epic PR Stage (spawn `pr-reviewer` and `e2e-tester`, validate diff, run CI and E2E suites, mark `gh pr ready "$EPIC_PR_NUMBER"`).
2. Verify post-validation `origin/main` divergence check / rebase gate.
3. Do not spawn any more worktrees for live execution sub-issues.
4. Do not close the epic.
5. Pause and present a handoff report listing the open Epic PR and remaining live tasks for manual/human execution.

## 3. Epic Worktree Initialization & Branch Hygiene

When the skill starts (whether in a dedicated worktree or the main space):

```bash
init_epic_worktree_and_pr() {
  # 1. Ensure working tree is clean
  if [ -n "$(git status --porcelain)" ]; then
    echo "ERROR: Working tree is dirty. Stash or commit before starting."
    return 1
  fi

  # 2. Get current branch and repo defaults
  local current_branch
  current_branch="$(git branch --show-current)"
  DEFAULT_BRANCH="$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || echo "main")"

  # 3. Rename epic issue branch appropriately before tracking origin
  local epic_title short_slug desired_branch
  epic_title="$(gh issue view "$EPIC_NUM" --json title -q .title)"
  short_slug="$(echo "$epic_title" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-' | sed 's/^-//;s/-$//' | cut -c1-30)"
  desired_branch="epic-${EPIC_NUM}-${short_slug}"

  if [ "$current_branch" = "$DEFAULT_BRANCH" ]; then
    git checkout -b "$desired_branch"
    current_branch="$desired_branch"
  elif [ "$current_branch" != "$desired_branch" ]; then
    git branch -m "$desired_branch"
    current_branch="$desired_branch"
  fi
  EPIC_BRANCH="$current_branch"

  # 4. Ensure renamed branch tracks origin
  if ! git rev-parse --abbrev-ref @{u} >/dev/null 2>&1; then
    git push -u origin "$EPIC_BRANCH"
  fi

  # 5. Draft parent Epic PR targeting main (main branch remains unaltered)
  EPIC_PR_NUMBER=$(gh pr list --head "$EPIC_BRANCH" --base "$DEFAULT_BRANCH" --json number -q '.[0].number')
  if [ -z "$EPIC_PR_NUMBER" ] || [ "$EPIC_PR_NUMBER" = "null" ]; then
    EPIC_PR_NUMBER=$(gh pr create --draft \
      --base "$DEFAULT_BRANCH" \
      --head "$EPIC_BRANCH" \
      --title "Epic #${EPIC_NUM}: ${epic_title}" \
      --body "Tracking epic issue #${EPIC_NUM}.\n\nAll sub-issue PRs will target this branch and merge here before opening for final review.")
  fi
}
```

## 4. Herdr Worktree & Tab Conventions

- **Branch Naming**: `issue-<issue-number>`
- **Base Branch**: `$EPIC_BRANCH`
- **Tab Label**: `#<issue-number>`
- **Pane Renaming**: `sub-issue-<issue-number>`

### Strict Prohibition on Manual Sub-Issue Implementation
The Epic Manager MUST NEVER `cd` into sub-issue worktrees (e.g. `.herdr/worktrees/.../issue-<N>`), edit source files, write patch scripts, run tests, or commit/push code for sub-issues directly. All implementation and validation MUST be executed by the subordinate `/github-issue-pr-manager` agent inside its Herdr worktree. The Epic Manager coordinates purely through Herdr CLI commands and GitHub PR merges.

### Worktree Creation Script Pattern

```bash
create_subissue_worktree() {
  local issue_num="$1"
  local branch_name="issue-${issue_num}"

  # 1. Create worktree branching from EPIC_BRANCH
  local res
  res=$(herdr worktree create \
    --workspace "$PARENT_WORKSPACE_ID" \
    --base "$EPIC_BRANCH" \
    --branch "$branch_name" \
    --label "#${issue_num}" \
    --no-focus \
    --json)

  local ws_id tab_id pane_id
  ws_id=$(echo "$res" | jq -r '.result.workspace.workspace_id // .result.workspace_id')
  tab_id=$(echo "$res" | jq -r '.result.tab.tab_id // .result.tab_id')
  pane_id=$(echo "$res" | jq -r '.result.root_pane.pane_id // .result.pane_id')

  if [ -z "$pane_id" ] || [ "$pane_id" = "null" ]; then
    pane_id=$(herdr pane list --workspace "$ws_id" --json | jq -r '.result.panes[0].pane_id')
  fi
  if [ -z "$tab_id" ] || [ "$tab_id" = "null" ]; then
    tab_id=$(herdr tab list --workspace "$ws_id" --json | jq -r '.result.tabs[0].tab_id')
  fi

  # 2. Rename tab into "#<issue number>"
  herdr tab rename "$tab_id" "#${issue_num}"
  herdr pane rename "$pane_id" "sub-issue-${issue_num}"

  # 3. Start proclaude with Advanced AdaAI (always-on thinking, do not pass --effort)
  herdr pane run "$pane_id" "proclaude --model advanced-adaai"

  # 4. Wait for proclaude ready
  herdr agent wait "sub-issue-${issue_num}" --until idle --timeout 60000 2>/dev/null || sleep 8

  # 5. Send goal prompt targeting the epic branch with workspace isolation instruction
  herdr agent prompt "$pane_id" "/goal use /github-issue-pr-manager to complete mergeable PR for issue #${issue_num} targeting branch ${EPIC_BRANCH}. CRITICAL WORKSPACE ISOLATION: Ensure all spawned tabs (planning, implementation, e2e) are created strictly inside this sub-issue workspace by passing --workspace \"\$HERDR_WORKSPACE_ID\" to herdr tab create."
}
```

## 5. PR Merge & Epic Worktree Sync Pattern

### 5.1 Sub-Issue Monitoring Boundary (No Sub-Issue wpci)
The Epic Manager monitors child worktrees purely through passive Herdr agent status (`idle`/`done`) and high-level PR status (`gh pr list --head "issue-${issue_num}" --base "$EPIC_BRANCH" --json number,state,isDraft,mergeable,reviewDecision`).
**Strict Boundary**: Sub-issue CI/Woodpecker (`wpci`) pipeline monitoring, log inspection, and build-fail fixes are exclusively handled by `/github-issue-pr-manager` inside the child worktree. The Epic Manager MUST NOT run `wpci` or poll Woodpecker pipelines for sub-issue PRs. When the child agent successfully validates the PR, it will mark the PR ready for review (`isDraft: false`).

### 5.2 Completion and Merge Routine

When a worktree agent completes:

```bash
complete_and_sync_subissue() {
  local issue_num="$1"
  local ws_id="$2"

  # Find PR targeting the epic branch
  local pr_num
  pr_num=$(gh pr list --head "issue-${issue_num}" --base "$EPIC_BRANCH" --json number -q '.[0].number')

  # Merge PR into the epic branch
  gh pr merge "$pr_num" --squash --delete-branch --subject "Merge PR #${pr_num} for issue #${issue_num} into ${EPIC_BRANCH}"

  # Sync ONLY the epic issue's worktree (main branch is strictly unaltered)
  cd "$EPIC_WORKTREE_ROOT"
  git fetch origin "$EPIC_BRANCH"
  git checkout "$EPIC_BRANCH"
  git pull --ff-only origin "$EPIC_BRANCH"

  # Update epic checklist
  local body new_body
  body=$(gh issue view "$EPIC_NUM" --json body -q .body)
  new_body=$(echo "$body" | sed -E "s/- \[ \] (.*(#|issues\/)${issue_num}[^0-9]?)/- [x] \1/g")
  gh issue edit "$EPIC_NUM" --body "$new_body"

  # Post comment
  gh issue comment "$EPIC_NUM" --body "Sub-issue #${issue_num} merged into ${EPIC_BRANCH} via PR #${pr_num}. Epic worktree updated."

  # Remove/close finished sub-issue worktree
  herdr worktree remove --workspace "$ws_id" --force --json
}
```

## 6. Epic PR Stage Protocol (After Stop Condition is Met)

Once all code-change sub-issues are merged into `$EPIC_BRANCH`, the code-change stop condition is met. The Epic Manager executes the PR stage on the parent Epic PR targeting `$DEFAULT_BRANCH` (main), spawning review and testing agents **the exact same as `/github-issue-pr-manager` PR stage**.

### 6.1 Agent Topology and Launchers

| Agent | Launcher | Location |
|---|---|---|
| `sub-issue-<N>` | `proclaude --model advanced-adaai` | Sub-issue worktree tab `#<N>` root pane |
| `pr-reviewer` | `proclaude --model advanced-adaai` | Main epic manager tab, right pane |
| `e2e-tester` | `proclaude --model lite-adaai` | Dedicated tab `e2e-${EPIC_NUM}` |
| `code-reviewer` (if fixes needed) | `proclaude --model pro-adaai` | Implementation tab `implementation-epic-${EPIC_NUM}` |
| `imp-coder-1` (if fixes needed) | `proclaude --model lite-adaai` | Implementation tab `implementation-epic-${EPIC_NUM}` |

### 6.2 Spawning PR Stage Agents

```bash
spawn_epic_pr_stage_agents() {
  # 1. Main tab: keep manager on full-height left, spawn pr-reviewer on the right
  local reviewer_pane
  reviewer_pane=$(herdr pane split "$PARENT_PANE_ID" --direction right --no-focus --json | jq -r '.result.pane_id // .result.pane.pane_id')
  herdr pane rename "$reviewer_pane" "pr-reviewer"
  herdr pane run "$reviewer_pane" "proclaude --model advanced-adaai"

  # 2. Dedicated tab: spawn e2e-tester in tab e2e-<epic#>
  local e2e_tab e2e_pane
  e2e_tab=$(herdr tab create --workspace "$PARENT_WORKSPACE_ID" --label "e2e-${EPIC_NUM}" --no-focus --json)
  e2e_pane=$(echo "$e2e_tab" | jq -r '.result.root_pane.pane_id // .result.pane_id')
  if [ -z "$e2e_pane" ] || [ "$e2e_pane" = "null" ]; then
    e2e_pane=$(herdr pane list --tab "$(echo "$e2e_tab" | jq -r '.result.tab_id // .result.tab.tab_id')" --json | jq -r '.result.panes[0].pane_id')
  fi
  herdr pane rename "$e2e_pane" "e2e-tester"
  herdr pane run "$e2e_pane" "proclaude --model lite-adaai"

  # 3. Wait for agents to become ready
  herdr agent wait "pr-reviewer" --until idle --timeout 60000 2>/dev/null || sleep 8
  herdr agent wait "e2e-tester" --until idle --timeout 60000 2>/dev/null || sleep 8
}
```

### 6.3 Prompting, Directives, and Restrictions

Every delegated agent prompt must include:
1. **Mandatory Simplicity & Anti-Overcomplication Directive**:
   > Deliver the minimal sufficient solution/review for the assigned scope: the smallest correct change consistent with repository conventions. Do not overcomplicate, over-abstract, or over-engineer. Do not introduce new layers, abstractions, helpers, wrappers, configuration, or dependencies.

2. **Notification Ownership Restriction**:
   > Do not notify the user when you finish. Do not invoke a Herdr notification command or any desktop/system notification. Leave your concise completion handoff in this agent session, enter a settled or idle state, and wait for your supervising agent to inspect it. Report blockers through this session only.

### 6.4 E2E Testing Protocol (`e2e-tester` in tab `e2e-${EPIC_NUM}`)

1. **Discovery**: Inspect repository for E2E suites or test compose files (`playwright.config.*`, `cypress.config.*`, `docker-compose.e2e.yml`, `compose.e2e.yml`, `npm run test:e2e`, `pytest tests/e2e`, `make e2e`).
2. **STRICT READ-ONLY RULE**: `e2e-tester` has ZERO write authority on repository code or tests. It MUST NEVER edit production files, patch test files, write new tests, or execute git commits. It is strictly a test runner and reporter.
3. **If Absent**: Report that E2E tests are absent/skipped, enter `idle`, and do not block PR readiness.
4. **If Present**: Build/start dependencies, execute test suites headless against Epic PR head SHA, capture exit codes and untruncated logs.
5. **Failures**: If tests fail, capture untruncated failure traces and report them to the Epic Manager. Code fixes MUST be routed to dedicated implementation agents (`code-reviewer` and `imp-coder-1`).
6. **Cleanup**: Tear down temporary test containers and volumes (`docker compose -f <manifest> down -v`).
7. **Preservation**: Keep `e2e-${EPIC_NUM}` tab and pane open and idle for audit history.

### 6.5 PR CI Discovery & Validation Policy

1. **Woodpecker CI Optional**: Check for `.woodpecker/pr.yaml`, `.woodpecker/pr.yml`, or root `.woodpecker.yml`/`.woodpecker.yaml`. If absent, Woodpecker CI is optional; do not block progress.
2. **CI Compose Containers Option**: If `docker-compose.ci.yml`, `docker-compose.test.yml`, `compose.ci.yml`, or `compose.test.yml` exists, run local CI compose containers (`docker compose -f <manifest> up --build --abort-on-container-exit`) and clean up (`docker compose -f <manifest> down -v`).
3. **Status Checks & Native Fallback**: Monitor GitHub Actions via `gh pr checks`. If neither Woodpecker nor compose CI exists, run repository test suites (`npm test`, `pytest`, `cargo test`, `make test`).

### 6.6 Formal PR Review Flow (`pr-reviewer`)

1. `pr-reviewer` inspects the full diff of `$EPIC_BRANCH` targeting `$DEFAULT_BRANCH` on GitHub.
2. Returns proposed draft findings privately to Epic Manager.
3. Epic Manager validates findings (filters false positives, duplicate findings, or out-of-scope feedback).
4. `pr-reviewer` submits formal review via GitHub CLI / API with inline comments on changed lines and summary for cross-cutting items.

### 6.7 Viewing "Done" Agents to Transition to "Idle"

When `pr-reviewer` or `e2e-tester` completes a turn and enters `done` in Herdr:
```bash
# View pr-reviewer and restore focus
herdr agent focus "pr-reviewer"
herdr agent focus "$PARENT_PANE_ID"

# View e2e-tester and restore focus
herdr agent focus "e2e-tester"
herdr tab focus "$PARENT_TAB_ID"
```
Retain all panes and tabs open across stages for full audit trail.

### 6.8 Code Fixes & Remediation on the Epic Branch (Implementation Agents Required)

If formal PR review findings (`pr-reviewer`), CI failures, or E2E tests (`e2e-tester`) reveal code defects, missing behavior, or test failures:
- **DO NOT allow `e2e-tester` or `pr-reviewer` to edit code or write fixes.**
- The Epic Manager MUST spawn (or re-activate) an implementation tab `implementation-epic-${EPIC_NUMBER}` with `code-reviewer` (`proclaude --model pro-adaai`) and `imp-coder-1` (`proclaude --model lite-adaai`).
- The Epic Manager compiles a consolidated failure report and dispatches it to `code-reviewer`.
- `code-reviewer` assigns the fix tasks to `imp-coder-1` with explicit write boundaries and the strict anti-overcomplication directive.
- `imp-coder-1` implements the fix, reviews its own diff, and reports to `code-reviewer`.
- `code-reviewer` reviews the diff, creates **exactly one cohesive commit**, and pushes to `$EPIC_BRANCH`.
- The Epic Manager then re-dispatches `pr-reviewer` and `e2e-tester` to re-validate the new head SHA.

### 6.9 Epic PR Readiness Transition (Draft to Open)

The parent Epic PR remains in **draft** status until:
1. Formal PR review has no unresolved blocking, high, or medium actionable findings.
2. Required CI / validation passes (passing Woodpecker CI if present, or CI compose containers, or repository test suite).
3. E2E tests pass (if present; skipped if none).

Once all three criteria pass:
```bash
gh pr ready "$EPIC_PR_NUMBER"
```

### 6.10 Post-Validation `origin/main` Rebase Gate

After the Epic PR is marked ready for review:
1. Confirm working tree is clean.
2. `git fetch origin "$DEFAULT_BRANCH"` without merging.
3. Check divergence: `git merge-base --is-ancestor origin/$DEFAULT_BRANCH HEAD || DIVERGED=1`.
4. If diverged:
   - Record pre-rebase reference.
   - Rebase `$EPIC_BRANCH` onto `origin/$DEFAULT_BRANCH`.
   - Resolve conflicts semantically.
   - Push with `git push --force-with-lease origin "$EPIC_BRANCH"`.
   - Restart the PR stage: rerun review, E2E tests, and CI for the rebased head commit.

---

## 7. Conflict Resolution & Branch Divergence

When multiple sub-issues are being worked on:
- One sub-issue merging updates `origin/$EPIC_BRANCH`.
- The remaining open sub-issue PRs may fall behind `origin/$EPIC_BRANCH`.
- `/github-issue-pr-manager` Stage 4 automatically fetches the target base branch (`$EPIC_BRANCH`) before marking PR ready, rebasing if needed.
- If conflicts arise in a running worktree, the subordinate `code-reviewer` resolves them or the Epic Manager detects `mergeable == CONFLICTING` and triggers rebase in the respective worktree pane.
