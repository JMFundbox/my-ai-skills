---
name: github-epic-issue-manager
description: Manage a GitHub epic issue in coordination with Herdr and github-issue-pr-manager. Supports execution in an isolated epic worktree or main worktree, ensures a clean worktree, renames the epic branch appropriately before tracking origin, drafts an epic PR targeting main, spawns Herdr sub-issue worktrees targeting the epic branch, supervises ProClaude agents running /github-issue-pr-manager, merges completed PRs into the epic branch, syncs only the epic issue's worktree (keeping main strictly unaltered), closes/removes sub-issue worktrees once done or PR merged, and only after the code-change stop condition is met, executes the PR stage by spawning pr-reviewer (Advanced AdaAI) and e2e-tester (Lite AdaAI) to validate and open the epic PR for review (stopping when only live execution tasks remain or when the epic is complete).
---

# GitHub Epic Issue Manager

The **GitHub Epic Issue Manager** orchestrates high-level epic/umbrella issues across a Herdr space. It coordinates multiple isolated worktrees, delegates individual code-change sub-issues to subordinate agents driven by `/github-issue-pr-manager`, monitors PR readiness, merges approved PRs into the epic issue's branch, synchronizes **only** the epic issue's worktree, and continuously drives the epic.

**Crucial Scope & Worktree Rules**:
1. **Worktree & Branch Hygiene**: The skill may be started in a dedicated worktree or the main workspace. It verifies the worktree is completely clean.
2. **Branch Renaming & Tracking**: The skill renames the current branch appropriately (e.g., `epic-<number>-<short-kebab-description>`) before pushing and tracking to `origin`.
3. **Main Branch Remains Strictly Unaltered**: The `main` (default) branch is **never** checked out, modified, committed to, or merged into during this workflow. All work happens on the epic branch and sub-issue branches.
4. **Epic PR Targeting Main**: It drafts a parent PR for the epic branch targeting `main` (or default branch).
5. **Sub-Issue PRs Target Epic Branch**: Every sub-issue worktree branches off the epic branch, and its PR targets and merges into the epic branch.
6. **Sync Epic Worktree Only**: Every time a sub-issue PR is merged, the manager syncs **only the epic issue's worktree** (never the main worktree).
7. **Code Changes Only**: This skill **only** manages sub-issues that involve **code changes** destined for pull requests. It does **not** manage sub-issues that are for **live execution** (deployments, running migrations in production/staging, manual QA, ops tasks).
8. **Epic PR Stage After Stop Condition**: Once all code-change sub-issues are merged into the epic branch (the code-change stop condition is met), the skill does not immediately mark the Epic PR as ready. Instead, it executes the PR stage by spawning `pr-reviewer` (`proclaude --model advanced-adaai`) in the main tab and `e2e-tester` (`proclaude --model lite-adaai`) in a dedicated `e2e-<epic#>` tab—the exact same as `/github-issue-pr-manager` PR stage. Only after formal review passes, E2E tests pass, and required CI / validation passes on the latest head, it opens the Epic PR for review (`gh pr ready`), performs the post-validation `origin/main` divergence check, and halts at the appropriate stopping point (Stopping Point A if live execution tasks remain, or Stopping Point B when the entire epic is complete).
9. **Close Finished Sub-Issue Worktrees**: Once a sub-issue is done and its PR is merged into the epic branch, the manager closes and removes the sub-issue Herdr worktree (`herdr worktree remove --workspace "$TARGET_WORKSPACE_ID" --force --json`) to keep the workspace clean and free up disk/process resources.
10. **Sub-Issue PR Validation Ownership**: Sub-issue CI/Woodpecker (`wpci`) monitoring, test checks, and fix cycles belong strictly to the subordinate `/github-issue-pr-manager` agent inside its own worktree. The Epic Manager MUST NOT poll, tail, inspect logs for, or diagnose Woodpecker CI (`wpci`) on sub-issue PRs. The Epic Manager only performs high-level passive monitoring (Herdr agent status `idle`/`done` and `isDraft: false` on the PR). CI and Woodpecker inspection only enters the Epic Manager's scope during the parent Epic PR Stage for the parent Epic PR targeting main.
11. **Strict Prohibition on Manual Sub-Issue Worktree Implementation by Epic Manager**: The Epic Manager is an orchestrator, NOT an implementer. The Epic Manager MUST NEVER `cd` into a sub-issue worktree directory, edit code files, write test patches, run test suites (`pytest`, `uv run`), execute `git commit`, or push commits for a sub-issue. All sub-issue implementation and review MUST occur strictly through the subordinate agent (`sub-issue-<N>`) spawned inside its Herdr worktree running `/github-issue-pr-manager`. The Epic Manager interacts with sub-issues SOLELY via Herdr commands (`herdr agent prompt`, `herdr pane read`, `herdr worktree remove`) and GitHub CLI (`gh pr merge`).
12. **Strict Prohibition on e2e-tester Modifying Code**: `e2e-tester` has ZERO write authority on repository code or tests. `e2e-tester` MUST NEVER edit production files, patch test files, write new tests, or execute git commits. It is strictly a test runner and reporter. If tests fail or formal review requests changes during the Epic PR Stage, the Epic Manager MUST spawn/respawn dedicated implementation agents (`code-reviewer` and `imp-coder-1`) in an implementation tab (`implementation-epic-${EPIC_NUMBER}`) to implement, review, and commit the fixes.

This skill coordinates directly with the `herdr` skill and `github-issue-pr-manager`.

---

## 1. Prerequisites and Environment Verification

Before executing any commands, verify that this session is running in a Herdr-managed pane within an authenticated Git repository:

```bash
# 1. Verify Herdr environment
test "${HERDR_ENV:-}" = 1 || { echo "ERROR: Not running inside Herdr (HERDR_ENV != 1)."; exit 1; }

# 2. Verify git repository and gh CLI authentication
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "ERROR: Not inside a Git repository."; exit 1; }
gh auth status >/dev/null 2>&1 || { echo "ERROR: GitHub CLI (gh) not authenticated."; exit 1; }

# 3. Discover workspace and worktree context
PARENT_WORKSPACE_ID="${HERDR_WORKSPACE_ID:-}"
PARENT_TAB_ID="${HERDR_TAB_ID:-}"
PARENT_PANE_ID="${HERDR_PANE_ID:-}"
EPIC_WORKTREE_ROOT="$(git rev-parse --show-toplevel)"
DEFAULT_BRANCH="$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || echo "main")"
CURRENT_BRANCH="$(git branch --show-current)"
```

### 1.1 Worktree Cleanliness, Branch Renaming, and Tracking

The skill may be started in a dedicated worktree created for the epic issue, or in the primary repository workspace. In all cases, `main` must remain completely unaltered.

1. **Verify Clean Worktree**:
   Ensure there are no uncommitted or dirty changes:
   ```bash
   if [ -n "$(git status --porcelain)" ]; then
     echo "ERROR: Working tree is dirty. Stash, commit, or clean working tree before starting."
     exit 1
   fi
   ```

2. **Rename Epic Issue Branch Appropriately**:
   Fetch the epic issue title and generate a clean kebab-cased branch name:
   ```bash
   EPIC_TITLE="$(gh issue view "$EPIC_NUMBER" --json title -q .title)"
   SHORT_SLUG="$(echo "$EPIC_TITLE" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-' | sed 's/^-//;s/-$//' | cut -c1-30)"
   DESIRED_BRANCH="epic-${EPIC_NUMBER}-${SHORT_SLUG}"

   # If currently on the default branch (e.g. main), switch to a new epic branch
   if [ "$CURRENT_BRANCH" = "$DEFAULT_BRANCH" ]; then
     git checkout -b "$DESIRED_BRANCH"
     CURRENT_BRANCH="$DESIRED_BRANCH"
   elif [ "$CURRENT_BRANCH" != "$DESIRED_BRANCH" ]; then
     # Rename current worktree branch to the standard convention before tracking
     git branch -m "$DESIRED_BRANCH"
     CURRENT_BRANCH="$DESIRED_BRANCH"
   fi
   EPIC_BRANCH="$CURRENT_BRANCH"
   ```

3. **Track Origin**:
   Push the appropriately renamed branch and set upstream tracking to `origin`:
   ```bash
   if ! git rev-parse --abbrev-ref @{u} >/dev/null 2>&1; then
     git push -u origin "$EPIC_BRANCH" || { echo "ERROR: Failed to push and track $EPIC_BRANCH on origin."; exit 1; }
   fi
   ```

4. **Draft Epic PR Targeting Main**:
   Check if a PR already exists for `$EPIC_BRANCH` targeting `$DEFAULT_BRANCH`. If not, create a draft PR:
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

*(Note: The `main` branch remains untouched throughout; only the draft PR references it as the eventual target base).*

---

## 2. Epic Architecture & Lifecycle

```text
+-----------------------------------------------------------------------------------------+
|                                    EPIC MANAGER                                         |
|  (Worktree for Epic Branch: '$EPIC_BRANCH', tracking origin)                            |
|  (Epic PR: Draft PR targeting '$DEFAULT_BRANCH' / main)                                 |
|                                                                                         |
|  1. Verify clean worktree & branch tracking origin -> Draft Epic PR targeting main      |
|  2. Read Epic Issue (#EPIC) -> Filter for Code-Change Sub-Issues (Skip Live Tasks)      |
|  3. For each available code-change sub-issue:                                           |
|       - Spawn Herdr worktree branching from '$EPIC_BRANCH'                              |
|       - Rename initial Herdr tab to "#<issue-number>"                                   |
|       - Launch ProClaude agent (advanced-adaai) in tab's root pane                     |
|       - Instruct: "/goal use /github-issue-pr-manager to complete mergeable PR for #N   |
|                   targeting branch $EPIC_BRANCH"                                        |
|  4. Monitor worktrees passively via Herdr state & PR isDraft status (no sub-issue wpci)  |
|  5. On Sub-Issue PR Done:                                                               |
|       - Merge PR into '$EPIC_BRANCH'                                                    |
|       - Synchronize Epic worktree (git fetch & git pull --ff-only origin $EPIC_BRANCH)  |
|       - Mark sub-issue complete in Epic checklist (- [x] #N)                            |
|       - Close and remove completed sub-issue worktree                                   |
|  6. Repeat until all code-change sub-issues are merged (Stop Condition Met)             |
|  7. Epic PR Stage (spawn agents the same as /github-issue-pr-manager PR stage):         |
|       - Main tab right pane: 'pr-reviewer' (proclaude --model advanced-adaai)            |
|       - Dedicated tab 'e2e-<epic#>': 'e2e-tester' (proclaude --model lite-adaai)         |
|       - Concurrent PR review, E2E testing, CI validation against draft Epic PR          |
|       - Manually view 'done' agents to update to 'idle'                                 |
|       - Mark Epic PR ready for review (gh pr ready $EPIC_PR_NUMBER) once all pass       |
|       - Post-validation origin/main rebase gate                                         |
|  8. Stop if only live execution sub-issues remain, OR close Epic when fully complete    |
+-----------------------------------------------------------------------------------------+
        |                                       |
        v                                       v
+-------------------------------+       +-------------------------------+
| Herdr Worktree: #101          |       | Herdr Worktree: #102          |
| Base: $EPIC_BRANCH            |       | Base: $EPIC_BRANCH            |
| Tab: "#101"                   |       | Tab: "#102"                   |
| PR targets: $EPIC_BRANCH      |       | PR targets: $EPIC_BRANCH      |
| Merges into: $EPIC_BRANCH     |       | Merges into: $EPIC_BRANCH     |
+-------------------------------+       +-------------------------------+
```

---

## 3. Step 1: Read Epic Issue & Extract Sub-Issues

Identify the target epic issue number from the invocation argument or context (e.g., `#50` or `50`).

### 3.1 Fetch Issue Details

```bash
gh issue view <EPIC_NUMBER> --json number,title,body,state,comments,labels
```

### 3.2 Extract and Classify Sub-Issues

Sub-issues may be declared in the epic issue via:
1. **GitHub Task Lists**: Markdown checkboxes such as `- [ ] #101` or `- [ ] https://github.com/<owner>/<repo>/issues/101`.
2. **Native GitHub Sub-Issues / Hierarchy**: Issues linked under GitHub issue hierarchy (if enabled on the repository).
3. **Ordered List with Dependencies**: Sub-issues listed with sequential ordering or explicit dependency notes (e.g., `Issue #102 depends on #101`).

Every sub-issue must be classified into one of two categories:

- **Code Change Sub-Issues (IN SCOPE)**: Tasks that modify repository code, tests, documentation, configuration, or CI pipelines, producing a pull request for review and merge via `/github-issue-pr-manager`.
- **Live Execution Sub-Issues (OUT OF SCOPE)**: Tasks requiring operational, manual, or live-environment action (e.g., deploying services to staging/production, running database migrations on live instances, manual smoke-testing, DNS/domain updates, secret/credential provisioning, server-side data backfills).

#### Identification of Live Execution Sub-Issues

Classify a sub-issue as **Live Execution** if any of the following apply:
1. **Labels**: Carries labels such as `live-execution`, `live`, `deploy`, `deployment`, `ops`, `release`, `manual`, `manual-qa`, `infra-ops`, or `maintenance`.
2. **Epic Sectioning & Tags**: Positioned under headings such as `### Deployment`, `### Live Execution`, `### Manual Operations`, or marked with inline tags like `[live]`, `[manual]`, `[deploy]`, `[ops]`.
3. **Issue Title & Body**: The issue title or acceptance criteria focus on environment actions (e.g., "Deploy to prod", "Run schema migration against DB", "Verify endpoint in live environment", "Rotate API keys") rather than code modifications producing a pull request.

**Rule**: The skill **must not** create worktrees or launch `/github-issue-pr-manager` agents for live execution sub-issues.

### 3.3 Determine "Available" Sub-Issues

A sub-issue is **Available** for automated execution if:
1. **Code Change Only**: It is classified as a code change sub-issue (not a live execution sub-issue).
2. **State is Open**: `gh issue view <SUB_ISSUE_NUM> --json state -q .state` returns `OPEN`.
3. **Unchecked in Epic Checklist**: Present as `- [ ] #<SUB_ISSUE_NUM>` (not already `- [x]`).
4. **No Active Worktree/Agent**: It does not currently have an active Herdr worktree or running agent.
5. **Dependencies Met**: Any prerequisite sub-issue mentioned as a blocker has already been merged and marked complete.

*Concurrency Note*: Unless directed otherwise, spawn available sub-issues concurrently up to a concurrency limit (default: 2–3 concurrent worktrees) to balance system resources and reduce merge conflicts. If tasks modify overlapping modules, run them sequentially in dependency order.

---

## 4. Step 2: Spawn Herdr Worktree & Launch Sub-Issue Agent

For every available sub-issue `#<ISSUE_NUMBER>`:

### 4.1 Spawn Herdr Worktree

Create a Git worktree-backed workspace branching from `$EPIC_BRANCH` using Herdr CLI:

```bash
# Branch name convention: issue-<issue-number>
BRANCH_NAME="issue-${ISSUE_NUM}"

# Create worktree workspace branching from EPIC_BRANCH without shifting user focus
WORKTREE_JSON=$(herdr worktree create \
  --workspace "$PARENT_WORKSPACE_ID" \
  --base "$EPIC_BRANCH" \
  --branch "$BRANCH_NAME" \
  --label "#${ISSUE_NUM}" \
  --no-focus \
  --json)

# Parse created workspace, tab, and pane IDs from JSON
TARGET_WORKSPACE_ID=$(echo "$WORKTREE_JSON" | jq -r '.result.workspace.workspace_id // .result.workspace_id')
TARGET_TAB_ID=$(echo "$WORKTREE_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
TARGET_PANE_ID=$(echo "$WORKTREE_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')

# Fallback lookup if nested in list
if [ -z "$TARGET_PANE_ID" ] || [ "$TARGET_PANE_ID" = "null" ]; then
  TARGET_PANE_ID=$(herdr pane list --workspace "$TARGET_WORKSPACE_ID" --json | jq -r '.result.panes[0].pane_id')
fi
if [ -z "$TARGET_TAB_ID" ] || [ "$TARGET_TAB_ID" = "null" ]; then
  TARGET_TAB_ID=$(herdr tab list --workspace "$TARGET_WORKSPACE_ID" --json | jq -r '.result.tabs[0].tab_id')
fi
```

### 4.2 Rename Initial Herdr Tab

Rename the initial Herdr tab of the worktree into `#<issue number>`:

```bash
herdr tab rename "$TARGET_TAB_ID" "#${ISSUE_NUM}"
```

### 4.3 Label the Root Pane

Label the pane for clear tracking:

```bash
herdr pane rename "$TARGET_PANE_ID" "sub-issue-${ISSUE_NUM}"
```

### 4.4 Start the Agent with ProClaude (Advanced AdaAI)

Launch `proclaude` in the pane using the exact model profile:

```bash
# Launch proclaude with Advanced AdaAI (always-on thinking, do not pass --effort)
herdr pane run "$TARGET_PANE_ID" "proclaude --model advanced-adaai"
```

Wait until ProClaude completes startup and reaches its interactive prompt:

```bash
# Wait for the agent to become ready
herdr agent wait "sub-issue-${ISSUE_NUM}" --until idle --timeout 60000 2>/dev/null || \
  sleep 10
```

### 4.5 Instruct the Agent with Goal Command

Submit the required goal prompt to the newly running agent, explicitly instructing it that the PR must target `$EPIC_BRANCH` as base, and strictly enforcing workspace isolation (ensuring all spawned tabs are bound to `$HERDR_WORKSPACE_ID`):

```bash
herdr agent prompt "$TARGET_PANE_ID" "/goal use /github-issue-pr-manager to complete mergeable PR for issue #${ISSUE_NUM} targeting branch ${EPIC_BRANCH}. CRITICAL WORKSPACE ISOLATION: Ensure all spawned tabs (planning, implementation, e2e) are created strictly inside this sub-issue workspace by passing --workspace \"\$HERDR_WORKSPACE_ID\" to herdr tab create."
```

Or via pane send-text if Herdr agent wrapper is initializing:

```bash
herdr pane send-text "$TARGET_PANE_ID" "/goal use /github-issue-pr-manager to complete mergeable PR for issue #${ISSUE_NUM} targeting branch ${EPIC_BRANCH}. CRITICAL WORKSPACE ISOLATION: Ensure all spawned tabs (planning, implementation, e2e) are created strictly inside this sub-issue workspace by passing --workspace \"\$HERDR_WORKSPACE_ID\" to herdr tab create."
herdr pane send-keys "$TARGET_PANE_ID" Enter
```

---

## 5. Step 3: Monitor Active Worktrees

The Epic Manager monitors all active worktrees passively without interrupting subordinate workflows.

**Strict Prohibition on Sub-Issue CI/wpci Polling**: Sub-issue CI/Woodpecker (`wpci`) monitoring, log extraction, container debugging, and test runs belong strictly to the subordinate agent running `/github-issue-pr-manager` inside its own worktree. The Epic Manager MUST NOT poll `wpci`, tail Woodpecker pipeline logs, or manage CI failures for child sub-issue PRs. CI monitoring by the Epic Manager is strictly reserved for the parent Epic PR Stage (Step 5).

### 5.1 Inspection Loop

Periodically inspect each active sub-issue passively:

1. **Check Herdr Pane / Agent Lifecycle State**:
   ```bash
   herdr agent get "sub-issue-${ISSUE_NUM}" 2>/dev/null
   herdr pane read "$TARGET_PANE_ID" --source recent-unwrapped --lines 60
   ```
   Check for:
   - `idle` / `done` states.
   - Interactive permission prompts or blocks (refer to permission escalation rules in `github-issue-pr-manager`).

2. **Check Associated GitHub PR Status Targeting Epic Branch (High-Level Only)**:
   ```bash
   gh pr list --head "issue-${ISSUE_NUM}" --base "$EPIC_BRANCH" --json number,title,state,isDraft,mergeable,reviewDecision
   ```
   *(Note: Do not poll or query status checks / Woodpecker logs for sub-issues; the subordinate `/github-issue-pr-manager` handles CI passing and will transition the PR from draft to ready (`isDraft: false`) once CI and reviews pass).*

### 5.2 Completion Invariant

A sub-issue worktree is considered **Done** when all of the following conditions are met:
1. **Pull Request Published & Open**: PR exists for `issue-${ISSUE_NUM}` targeting `$EPIC_BRANCH` and is not closed/merged yet.
2. **PR is Ready for Review**: `isDraft` is `false` (marked ready by the child `/github-issue-pr-manager` after its review and CI validation have passed).
3. **Mergeable**: `mergeable` is `MERGEABLE` (no conflicts with `$EPIC_BRANCH`).
4. **No Blocking Review Feedback**: `reviewDecision` is `APPROVED` or clean with low/nit findings resolved.
5. **Subordinate Agent Settled**: The child agent has finished its execution and settled into an idle/done state.

---

## 6. Step 4: PR Merging & Epic Worktree Synchronization

Once a sub-issue worktree satisfies the completion invariant:

### 6.1 Merge the Pull Request into Epic Branch

Merge the sub-issue PR into the epic branch using repository standards (default: `--squash`):

```bash
PR_NUMBER=$(gh pr list --head "issue-${ISSUE_NUM}" --base "$EPIC_BRANCH" --json number -q '.[0].number')

# Merge PR into the epic branch and delete remote branch
gh pr merge "$PR_NUMBER" --squash --delete-branch --subject "Merge PR #${PR_NUMBER} for issue #${ISSUE_NUM} into ${EPIC_BRANCH}"
```

### 6.2 Synchronize the Epic Worktree

Switch to or execute in the epic worktree root to pull the merged commit:

```bash
cd "$EPIC_WORKTREE_ROOT"

# Fetch latest epic branch from origin
git fetch origin "$EPIC_BRANCH"

# Ensure clean working tree and fast-forward pull
git checkout "$EPIC_BRANCH"
git pull --ff-only origin "$EPIC_BRANCH"
```

Verify that `git status` reports a clean tree matching `origin/$EPIC_BRANCH`.

### 6.3 Update the Epic Issue Checklist

Update the task list in the epic issue description so progress is recorded durably:

```bash
# Fetch current epic body
EPIC_BODY=$(gh issue view "$EPIC_NUMBER" --json body -q .body)

# Replace "- [ ] #<ISSUE_NUM>" or "- [ ] <URL>/issues/<ISSUE_NUM>" with "- [x] ..."
NEW_BODY=$(echo "$EPIC_BODY" | sed -E "s/- \[ \] (.*(#|issues\/)${ISSUE_NUM}[^0-9]?)/- [x] \1/g")

# Update epic issue
gh issue edit "$EPIC_NUMBER" --body "$NEW_BODY"

# Post progress comment
gh issue comment "$EPIC_NUMBER" --body "Completed sub-issue #${ISSUE_NUM} via merged PR #${PR_NUMBER} into ${EPIC_BRANCH}. Epic worktree synchronized."
```

### 6.4 Clean Up and Remove Completed Sub-Issue Worktree

Once the sub-issue is complete and its PR is merged into `$EPIC_BRANCH`, close and remove the finished Herdr worktree to prevent clutter, process overhead, and disk accumulation:

```bash
herdr worktree remove --workspace "$TARGET_WORKSPACE_ID" --force --json
```

*(Note: If the user explicitly asks to preserve tabs/worktrees for a specific run or audit, skip removal and rename the tab to `[DONE] #${ISSUE_NUM}` instead).*

---

## 7. Step 5: Iterative Loop, Epic PR Stage (After Stop Condition), and Stopping Conditions

After completing a sub-issue and synchronizing the epic worktree:

### 7.1 Re-evaluate the Epic Issue & Discover Remaining Sub-Issues

1. **Fetch Latest Epic Status**:
   ```bash
   gh issue view "$EPIC_NUMBER" --json body,state
   ```

2. **Discover Remaining Sub-Issues**:
   - Parse all sub-issues that remain unchecked (`- [ ]`).
   - Classify them as **Code Change** or **Live Execution**.
   - Check if any code change sub-issues are unblocked by recent merges.

3. **Continue Sub-Issue Loop (If Code Changes Remain)**:
   If available code-change sub-issues still exist:
   - Handle branch divergence on active worktrees if needed (subordinate `/github-issue-pr-manager` rebase cycle).
   - Spawn new worktrees as described in Step 2, and continue the monitoring loop.

---

### 7.2 Code-Change Stop Condition Met: Execute Epic PR Stage

When **all code-change sub-issues are merged** into `$EPIC_BRANCH`, the **code-change stop condition is met**.

Do **not** immediately mark the Epic PR as ready for review or close the epic. Instead, execute the **Epic PR Stage** against the draft parent PR targeting `$DEFAULT_BRANCH` (main), spawning review and testing agents **the exact same as `/github-issue-pr-manager` PR stage**:

#### 1. PR Stage Topology & Agent Spawning

In the main epic manager tab, keep the manager on the full-height left and spawn `pr-reviewer` in a pane to the right. Concurrently, spawn a dedicated Herdr tab `e2e-${EPIC_NUMBER}` containing `e2e-tester`:

```bash
# 1. Spawn pr-reviewer in main tab (right pane) with Advanced AdaAI (always-on thinking)
PR_REVIEWER_PANE_ID=$(herdr pane split "$PARENT_PANE_ID" --direction right --no-focus --json | jq -r '.result.pane_id // .result.pane.pane_id')
herdr pane rename "$PR_REVIEWER_PANE_ID" "pr-reviewer"
herdr pane run "$PR_REVIEWER_PANE_ID" "proclaude --model advanced-adaai"
herdr agent wait "pr-reviewer" --until idle --timeout 60000 2>/dev/null || sleep 8

# 2. Spawn e2e-tester in dedicated tab with Lite AdaAI (always-on thinking)
E2E_TAB_JSON=$(herdr tab create --workspace "$PARENT_WORKSPACE_ID" --label "e2e-${EPIC_NUMBER}" --no-focus --json)
E2E_TAB_ID=$(echo "$E2E_TAB_JSON" | jq -r '.result.tab_id // .result.tab.tab_id')
E2E_PANE_ID=$(echo "$E2E_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
herdr pane rename "$E2E_PANE_ID" "e2e-tester"
herdr pane run "$E2E_PANE_ID" "proclaude --model lite-adaai"
herdr agent wait "e2e-tester" --until idle --timeout 60000 2>/dev/null || sleep 8
```

#### 2. Agent Prompts & Anti-Overcomplication Directives

Every delegated agent must receive:
- The exact repo path, epic branch `$EPIC_BRANCH`, base branch `$DEFAULT_BRANCH`, Epic PR number `$EPIC_PR_NUMBER`, and latest head SHA.
- The **Mandatory Anti-Overcomplication Directive**: Deliver the minimal sufficient solution/review. Do not overcomplicate or introduce new layers, abstractions, or scope creep.
- The **Notification Ownership Restriction**: Do not notify the user. Leave completion handoffs in the session and wait for the Epic Manager to inspect them.

#### 3. Concurrent PR Review, E2E Testing, and CI Validation

Start CI / validation, E2E testing, and PR review concurrently against the draft Epic PR head commit:

- **E2E Testing Protocol (`e2e-tester` in tab `e2e-${EPIC_NUMBER}`)**:
  - `e2e-tester` inspects the repository for End-to-End or system integration tests (Playwright, Cypress, `docker-compose.e2e.yml`, `compose.e2e.yml`, `npm run test:e2e`, `pytest tests/e2e`, etc.).
  - **STRICT READ-ONLY RESTRICTION**: `e2e-tester` has ZERO write authority on repository code or tests. `e2e-tester` MUST NEVER edit production code, modify test files, or write new test scripts. In the prompt to `e2e-tester`, explicitly include:
    > "You are strictly a test executor and validator. You have ZERO write authority on repository code or tests. Do NOT modify any files, do NOT write patches, and do NOT create new test files. If tests fail, capture untruncated failure traces and report them back. Code fixes are strictly performed by dedicated implementation agents."
  - If NO E2E test manifests exist, `e2e-tester` reports absent/skipped, settles into `idle`, and does not block PR readiness.
  - If E2E test manifests exist, `e2e-tester` spins up required services, executes the suite headless against the Epic PR head, captures complete logs/traces, and cleans up test containers and volumes (`docker compose -f <manifest> down -v`).
  - Failures are reported to the Epic Manager to route to dedicated implementation agents (`code-reviewer` and `imp-coder-1`).

- **PR CI Policy & Manifest Discovery**:
  - **Woodpecker CI is optional**: If the repo lacks a Woodpecker PR manifest, Woodpecker CI pass is optional; do not block.
  - **CI Compose Containers Option**: If a CI/test Docker Compose manifest exists (`docker-compose.ci.yml`, `docker-compose.test.yml`, etc.), run CI compose containers locally (`docker compose -f <manifest> up --build --abort-on-container-exit`) and tear down afterward (`docker compose -f <manifest> down -v`).
  - **External Status Checks & Native Fallback**: Monitor GitHub Actions via `gh pr checks`. If neither Woodpecker nor compose CI exists, run repository test/lint suites (`npm test`, `pytest`, `cargo test`, `make test`).

- **Formal PR Review Flow (`pr-reviewer`)**:
  - `pr-reviewer` reviews the full epic diff against `$DEFAULT_BRANCH` and returns draft findings privately to the Epic Manager.
  - The Epic Manager validates findings (filters false positives, duplicate findings, or out-of-scope feedback).
  - After validation, `pr-reviewer` submits a formal GitHub review with inline comments on changed lines and a summary for cross-cutting issues.

#### 4. Viewing "Done" Agents to Transition to "Idle"

Whenever `pr-reviewer` or `e2e-tester` finishes a turn and enters the `done` state in Herdr:
```bash
# Transition pr-reviewer to idle
herdr agent focus "pr-reviewer"
herdr agent focus "$PARENT_PANE_ID"

# Transition e2e-tester to idle
herdr agent focus "e2e-tester"
herdr tab focus "$PARENT_TAB_ID"
```
Retain the `e2e-${EPIC_NUMBER}` tab and `pr-reviewer` pane open and idle for audit and debugging history; do not close or kill them.

#### 5. Code Fixes & Remediation on the Epic Branch

If formal PR review findings or CI/E2E test failures require code fixes:
- The Epic Manager prepares a consolidated correction batch.
- If needed, spawn an implementation tab `implementation-epic-${EPIC_NUMBER}` with `code-reviewer` (`proclaude --model pro-adaai`) and `imp-coder-1` (`proclaude --model lite-adaai`).
- `code-reviewer` assigns fixes to `imp-coder-1`, reviews the combined diff, creates exactly one cohesive commit, and pushes to `$EPIC_BRANCH`.
- Re-evaluate review, E2E tests, and CI for the new head SHA.

#### 6. Epic PR Readiness Transition (Draft to Open)

The Epic PR remains in **draft** status throughout review, E2E, and CI cycles. Once the latest Epic PR commit satisfies all criteria:
1. Formal PR review has no unresolved blocking, high, or medium actionable findings.
2. Required CI / validation passes (passing Woodpecker CI if manifest exists, or passing CI compose containers, or passing repository test suite).
3. E2E tests pass (if E2E manifests exist; skipped if none).

Mark the Epic PR as ready for review:
```bash
gh pr ready "$EPIC_PR_NUMBER"
```

Update the Epic PR description:
```bash
gh pr edit "$EPIC_PR_NUMBER" --body "## Epic #${EPIC_NUMBER} Implementation Complete

All code-change sub-issues have been implemented, reviewed, and merged into \`${EPIC_BRANCH}\`.
Formal PR review, E2E testing, and CI validation have passed on head \`$(git rev-parse --short HEAD)\`.

### Merged Sub-Issues
$(gh issue view "$EPIC_NUMBER" --json body -q .body | grep -E '^- \[x\]')

This PR is now open and ready for final review and merge into \`${DEFAULT_BRANCH}\`."
```

#### 7. Post-Validation `origin/main` Rebase Gate

After the Epic PR is marked ready for review:
1. Ensure the working tree is clean on `$EPIC_WORKTREE_ROOT`.
2. Fetch origin default branch without merging:
   ```bash
   git fetch origin "$DEFAULT_BRANCH"
   ```
3. Check whether `origin/$DEFAULT_BRANCH` has advanced beyond the epic branch base:
   ```bash
   git merge-base --is-ancestor "origin/$DEFAULT_BRANCH" HEAD || DIVERGED=1
   ```
4. If diverged:
   - Record pre-rebase commit reference.
   - Rebase `$EPIC_BRANCH` onto `origin/$DEFAULT_BRANCH`.
   - Resolve conflicts semantically (preserving both upstream and epic behaviors).
   - Push rewritten branch with `git push --force-with-lease origin "$EPIC_BRANCH"`.
   - **Restart the PR stage**: A rebase creates a new head SHA. Rerun formal review, E2E testing, and CI validation on the rebased head before proceeding.

---

### 7.3 Evaluate Stopping Points

Now that the Epic PR has completed the PR stage and is open for review:

#### Stopping Point A: Only Live Execution Sub-Issues Remain Available
If all code-change sub-issues are merged into `$EPIC_BRANCH`, Epic PR `#${EPIC_PR_NUMBER}` has passed the PR stage and is marked ready for review, but **live execution sub-issues remain incomplete**:
- **Pause and Stop Automated Workflow**: Do **not** spawn worktrees or start `/github-issue-pr-manager` for live execution sub-issues.
- **Do NOT close the Epic Issue**: Live execution steps remain incomplete and require operator/human execution.
- **Report Hand-Off to User**: Provide a clear, actionable stopping report:
  - State that all code changes are complete, merged into `$EPIC_BRANCH`, verified via PR review, E2E tests, and CI, and Epic PR `#${EPIC_PR_NUMBER}` is now open for review targeting `${DEFAULT_BRANCH}`.
  - Explicitly list the pending live execution sub-issues (e.g., `#105 Run database migration on production`, `#106 Deploy v2 to staging`).
  - Hand over control to the user to review the Epic PR and execute the live tasks.
  - Note that the user can re-invoke `/github-epic-issue-manager` once the live steps are executed and checked off if subsequent code changes depend on them.

#### Stopping Point B: All Sub-Issues Complete (Epic Finished)
When **all** sub-issues declared in the epic (both code changes and live execution) are resolved and checked off (`- [x]` in task lists and all linked issues are closed):
- Note that Epic PR `#${EPIC_PR_NUMBER}` is open for review targeting `${DEFAULT_BRANCH}`. Merging into `${DEFAULT_BRANCH}` is performed via the PR review process, keeping `${DEFAULT_BRANCH}` unaltered by this automated workflow.
- Close the epic issue:
  ```bash
  gh issue close "$EPIC_NUMBER" --comment "All sub-issues (code changes and live execution) completed. Epic PR #${EPIC_PR_NUMBER} has passed formal review, E2E testing, and CI, and is ready for review/merge. Epic issue closed."
  ```
- Present a concise, structured final report to the user summarizing:
  - Epic issue number and title.
  - Epic PR number, URL, and ready-for-review status.
  - Completed code change sub-issues and their merged PR numbers into `$EPIC_BRANCH`.
  - Validation outcomes (formal PR review, E2E tests, CI checks).
  - Final synchronized commit SHA on the epic branch `$EPIC_BRANCH`.

---

## 8. Anti-Overcomplication and Safety Guardrails

- **Main Branch Strictly Unaltered**: Never checkout, commit to, push to, pull into, or merge directly into `main` (or `$DEFAULT_BRANCH`). All automated execution, worktrees, commits, and intermediate PR merges take place strictly on `$EPIC_BRANCH` and sub-issue branches. The transition to `main` happens solely via the GitHub PR review interface.
- **Sync Epic Worktree Only**: When a sub-issue PR is merged, synchronize **only** the epic issue's worktree (`$EPIC_WORKTREE_ROOT`). Do not sync or touch the main worktree.
- **Branch Renaming Before Tracking**: Always rename the epic branch to the standard convention (`epic-<number>-<short-kebab-description>`) before pushing to origin or creating the draft PR.
- **Zero Speculative Code & Sub-Issue Isolation**: The Epic Manager does not write code, alter diffs, run local tests, or commit/push code for sub-issues directly. Implementation and review belong entirely to `/github-issue-pr-manager` running inside the subordinate Herdr worktree. The Epic Manager MUST NEVER `cd` into child worktree paths (`.herdr/worktrees/.../issue-<N>`) to implement or fix sub-issues manually.
- **Strict Prohibition on Sub-Issue wpci Monitoring**: Sub-issue CI status, pipeline logs, and fix cycles are owned by the child `/github-issue-pr-manager`. Never poll or monitor `wpci` for sub-issue PRs from the Epic Manager.
- **Fail Safe on Merge Conflicts**: If a PR cannot be cleanly merged (`mergeable: CONFLICTING`), do not force-merge. Dispatch a rebase instruction to the subordinate worktree agent or prompt the user.
- **Auditable Worktrees**: Use consistent naming conventions:
  - Epic Branch: `epic-<number>-<short-kebab-description>`
  - Sub-Issue Branch: `issue-<number>`
  - Herdr Tab: `#<number>`
  - Herdr Pane: `sub-issue-<number>`
- **Single Source of Truth**: GitHub issues and Git history remain the sole sources of truth. No ephemeral state is stored outside of the Git repository and GitHub issue tracker.
