---
name: github-issue-pr-manager
description: Manage a GitHub issue through planning, implementation, independent review, PR publication, and passing CI using supervised Herdr agents launched through proclaude and procodex. Supports adaptive parallel implementation coders and formal GitHub PR reviews with inline comments. Use for an explicitly delegated issue-to-PR workflow, not a simple local patch or review-only request.
---

# GitHub Issue PR Manager

Own the requested issue through a review-ready, CI-green pull request. Do not merge unless the user separately requests it.

This workflow requires Herdr. Read the user-level `herdr` skill and then read [references/orchestration.md](references/orchestration.md) before creating panes or delegated agents. If `HERDR_ENV` is not `1`, explain that orchestration cannot start from the current pane and stop before delegation.

**Manager token conservation:** Read only the orchestration reference sections needed for the current stage rather than reading the entire reference at once.

## Manager ownership

The main issue manager is an orchestrator and coordinator. It owns issue reading, branch setup, issue updates, stage transitions, PR publication, CI monitoring, rebase operations, user communication, and final readiness verification.

The manager does NOT review implementation plans (delegated to `plan-reviewer`) and does NOT review code changes or diffs (delegated to `code-reviewer`). The manager does NOT commit or push implementation code (delegated to `code-reviewer`).

Delegated agents must not publish or merge a PR, or rewrite history. The following external-write exceptions apply:
- `code-reviewer` may commit and push implementation code and CI fixes to the feature branch.
- `pr-reviewer` may submit formal reviews and inline review comments to the managed PR.

Delegated agents must not send user-facing completion notifications; they leave internal handoffs in their panes. Only the main issue manager may notify the user that the workflow is complete.

Before implementation:

1. Read the issue, discussion, acceptance criteria, repository instructions, relevant code, tests, configuration, and recent history.
2. Inspect the current branch and working tree; preserve unrelated user changes.
3. Rename the current branch using repository convention. If none is discoverable, use `issue-<number>-<short-kebab-description>`.
4. Avoid destructive cleanup, history rewriting, and force-pushes unless clearly necessary and safe.
Do not pull, merge, cherry-pick, or rebase commits from `origin/main` into the feature branch during planning, implementation, or the initial PR validation cycle. First obtain a passing formal PR review and passing required CI / validation for the same PR head. Only then fetch and check `origin/main` for divergence as described below.

## Simplicity and completion discipline

The goal is a finished, review-ready, CI-green PR — not an exhaustive or overengineered one. This principle is strictly binding on the main issue manager AND all delegated agents (`imp-planner`, `plan-reviewer`, `code-reviewer`, `imp-coder-*`, `pr-reviewer`, `e2e-tester`). Every supervisor must explicitly enforce anti-overcomplication in every sub-agent prompt and during active/passive supervision:

- **Minimal sufficient solution across all roles.** Implement and review strictly what the issue's acceptance criteria require, in the smallest correct change consistent with repository conventions. Prefer extending existing patterns and functions over introducing new abstractions, helpers, layers, wrappers, or dependencies.
- **Strict prohibition on speculative work.** No refactoring, cleanup, extra features, unrequested defensive options, or "while I'm here" improvements that the issue does not mandate. Defer everything optional; note it in the handoff or follow-up issue instead of building it.
- **Mandatory anti-overcomplication in sub-agent prompts.** Every supervisor must explicitly inject anti-overcomplication instructions into every task prompt sent to its subordinates.
- **Enforcement during monitoring.** When inspecting sub-agent history (planner output, coder turns, reviewer drafts), actively watch for overengineering, unnecessary files, bloated plans, or review pedantry. If any agent begins introducing new modules, abstractions, or scope creep, interrupt or redirect immediately.
- **Reject overcomplicated plans, diffs, and review feedback.** During plan review, diff review, and PR review, treat unnecessary complexity — new files, new config, indirection, framework scaffolding, or tooling for a small task — as a concrete finding and send it back for simplification. Reviewers must not demand over-architected designs or speculative abstractions.
- **Fix forward, don't gold-plate fixes.** Correction cycles address only confirmed findings and patch-related CI failures, with the smallest safe change. Do not restructure working code to absorb a fix.
- **Prefer fewer coders.** Use a single `imp-coder-1` by default; spawn additional coders only when the savings are clearly larger than the coordination cost.
- **Finish.** When the completion gate passes, stop — do not run extra validation, re-review, or polish beyond the gate. A simpler completed PR beats a more elaborate one.

## Required workflow

### Strict Workspace Isolation
Every Herdr tab spawned by this workflow (`planning-<issue#>`, `implementation-<issue#>`, `e2e-<issue#>`) MUST be explicitly bound to the current workspace by passing `--workspace "$HERDR_WORKSPACE_ID"` to `herdr tab create`. Never invoke `herdr tab create` without `--workspace "$HERDR_WORKSPACE_ID"`. Omitting `--workspace` causes Herdr to create the tab in whichever workspace is currently active/focused on the server, which leaks sub-issue tabs into parent epic manager workspaces or unrelated workspaces.

### Stage 1. Planning

Before spawning agents, select `IMP_PLANNER_MODEL` using the complexity policy in [references/orchestration.md](references/orchestration.md). Set `IMP_PLANNER_MODEL=pro-adaai` as the baseline; set it to `advanced-adaai` only for a clearly low-complexity task. Verify the selected alias with `proclaude --list-models` and fall back to the Pro baseline if Advanced is unavailable.

Record parent pane ID for Stage 3 splitting:
```bash
PARENT_PANE_ID="${HERDR_PANE_ID}"
```

Spawn Herdr tab `planning-<issue#>` inside the current workspace:
```bash
PLAN_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "planning-${ISSUE_NUMBER}" --no-focus)
PLAN_TAB_ID=$(echo "$PLAN_TAB_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
PLAN_REV_PANE=$(echo "$PLAN_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
IMP_PLAN_PANE=$(herdr pane split "$PLAN_REV_PANE" --direction right --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')

herdr pane rename "$PLAN_REV_PANE" "plan-reviewer"
herdr pane rename "$IMP_PLAN_PANE" "imp-planner"

# Wait for interactive shell prompt before running launcher to prevent banner race
sleep 2

herdr pane run "$PLAN_REV_PANE" "proclaude --model advanced-adaai"
herdr pane run "$IMP_PLAN_PANE" "proclaude --model ${IMP_PLANNER_MODEL}"

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

# Confirm wrappers started before dispatching tasks
herdr pane process-info "$PLAN_REV_PANE"
herdr pane process-info "$IMP_PLAN_PANE"
```
The tab contains two agent panes:
- Left: `plan-reviewer` — reviews the implementation plan, requests changes, and approves when the plan is sound and implementation-ready.
- Right: `imp-planner` — produces a minimal implementation plan covering intended behavior, acceptance criteria, relevant files, ordered changes, risks, validation, assumptions, and open questions.

`plan-reviewer` autonomously iterates with `imp-planner`: reviewing the plan against repository evidence, returning concrete findings, and requesting revisions until the plan meets acceptance criteria and is implementation-ready. The manager does not review the plan itself.

Once `plan-reviewer` signals approval, the manager updates `plan-reviewer` and `imp-planner` to `idle` non-intrusively using `herdr pane report-agent` (demoting them down the priority list without stealing focus) and posts the approved plan to the GitHub issue. Keep the planning tab and its panes open for inspection and provenance; do not close or kill them after the stage transition.

Do not begin implementation before the approved plan is posted.

### Stage 2. Implementation

Before spawning `code-reviewer`, choose `CODE_REVIEWER_MODEL` from the approved plan and repository evidence using the same complexity policy. Set `CODE_REVIEWER_MODEL=pro-adaai` by default; use `advanced-adaai` only when both the issue and approved plan are clearly low-complexity. Verify the selected alias before launch. Do not change a persistent agent's model mid-session.

Spawn Herdr tab `implementation-<issue#>` inside the current workspace:
```bash
IMPL_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "implementation-${ISSUE_NUMBER}" --no-focus)
IMPL_TAB_ID=$(echo "$IMPL_TAB_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
CODE_REVIEWER_PANE_ID=$(echo "$IMPL_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
CODER_1_PANE_ID=$(herdr pane split "$CODE_REVIEWER_PANE_ID" --direction right --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')

herdr pane rename "$CODE_REVIEWER_PANE_ID" "code-reviewer"
herdr pane rename "$CODER_1_PANE_ID" "imp-coder-1"

# Wait for interactive shell prompt before running launcher
sleep 2

herdr pane run "$CODE_REVIEWER_PANE_ID" "proclaude --model ${CODE_REVIEWER_MODEL}"
herdr pane run "$CODER_1_PANE_ID" "proclaude --model lite-adaai"

# Wait for agent to occupy each pane (bounded: at most 15 checks, 2s apart)
for p in "$CODE_REVIEWER_PANE_ID" "$CODER_1_PANE_ID"; do
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

herdr agent rename "$CODE_REVIEWER_PANE_ID" "code-reviewer"
herdr agent rename "$CODER_1_PANE_ID" "imp-coder-1"

# Confirm wrappers started
herdr pane process-info "$CODE_REVIEWER_PANE_ID"
herdr pane process-info "$CODER_1_PANE_ID"
```
The tab has two columns:
- Left column: `code-reviewer` — owns work decomposition, code review, commits, and pushes.
- Right column: `imp-coder-1` through `imp-coder-3` (1–3 coders depending on workload), stacked vertically.

`code-reviewer` reads the approved plan and decomposes it into assignments for its sub-agents. It decides whether to use one coder (default) or up to three coders for substantial independent workstreams with disjoint write sets, stable interfaces, and lower integration cost than serial implementation.

Each `imp-coder-*` implements its assigned changes. Coders must be recreated for new tasks that are unrelated to previous assignments to ensure clean context and avoid cross-task context pollution (previous coder panes remain open and idle for audit/record keeping; same-task revisions continue in the active coder's session). `code-reviewer` reviews each integrated diff for correctness, acceptance-criteria coverage, regressions, compatibility, security, data integrity, API changes, test quality, scope creep, artifacts, secrets, and unrelated edits. Whenever an `imp-coder-*` finishes its deliverable, `code-reviewer` reports it as `idle` via `herdr pane report-agent` to put it down the priority list without switching tabs or stealing UI focus. It requests changes until review results contain at most acceptable (low/nit) findings.

After all changes are reviewed and approved, `code-reviewer` creates scoped commits and pushes the branch. The manager updates `code-reviewer` to `idle` via `herdr pane report-agent` (demoting it down the priority list without stealing focus), then publishes the PR linked to the issue in **draft** status (`gh pr create --draft`). Use `Fixes #<number>` when appropriate.

Do not force a full local test suite; CI is authoritative for full validation. However, Woodpecker CI is optional at the PR stage if the repository lacks a Woodpecker PR manifest. If a CI/test Docker Compose manifest exists, run CI compose containers for authoritative validation. Targeted inexpensive checks are allowed when useful.

### Stage 3. PR review, E2E testing, and CI

In the main issue-manager tab, keep the manager on the full-height left and spawn `pr-reviewer` in a pane to the right. Concurrently, spawn a dedicated Herdr tab `e2e-<issue#>` inside the current workspace:
```bash
# Main tab: split right for pr-reviewer
PR_REV_PANE=$(herdr pane split "$PARENT_PANE_ID" --direction right --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')
herdr pane rename "$PR_REV_PANE" "pr-reviewer"

# Dedicated tab: e2e-<issue#>
E2E_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "e2e-${ISSUE_NUMBER}" --no-focus)
E2E_TAB_ID=$(echo "$E2E_TAB_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
E2E_PANE_ID=$(echo "$E2E_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
herdr pane rename "$E2E_PANE_ID" "e2e-tester"

# Wait for interactive shell prompt before running launcher
sleep 2

herdr pane run "$PR_REV_PANE" "proclaude --model advanced-adaai"
herdr pane run "$E2E_PANE_ID" "proclaude --model lite-adaai"

# Wait for agent to occupy each pane (bounded: at most 15 checks, 2s apart)
for p in "$PR_REV_PANE" "$E2E_PANE_ID"; do
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

herdr agent rename "$PR_REV_PANE" "pr-reviewer"
herdr agent rename "$E2E_PANE_ID" "e2e-tester"

# Confirm wrappers started
herdr pane process-info "$PR_REV_PANE"
herdr pane process-info "$E2E_PANE_ID"
```

Start CI / validation, E2E testing, and PR review as early as possible and let them proceed concurrently against the draft PR. Do not impose a fixed review-first, test-first, or CI-first order. The manager dynamically chooses the next safe action based on severity, blocking impact, expected fix size, likely overlap with other findings, validation invalidation risk, CI duration, and whether the failure is patch-related or infrastructure-related.

**E2E Testing Protocol (`e2e-tester` in tab `e2e-<issue#>`):**
- `e2e-tester` inspects the repository for End-to-End or system integration tests (e.g. Playwright, Cypress, Puppeteer, Selenium, `docker-compose.e2e.yml`, `compose.e2e.yml`, `npm run test:e2e`, `pytest tests/e2e`, etc.).
- If the repository has NO E2E test manifests or scripts, `e2e-tester` reports that E2E tests are absent/skipped, settles into an idle state, and does not block PR readiness. For repos with no E2E suite, `skipped` is the expected result.
- If E2E test manifests or scripts exist, `e2e-tester` builds/starts necessary services, executes the E2E suite against the draft PR head, and captures exit codes and full logs. Label test containers with `e2e-${ISSUE_NUMBER}` and ensure `$TMPDIR` defaults to `/tmp`.
- If E2E tests fail: `e2e-tester` collects complete failure traces, logs, and screenshots/reports, and provides the failure report to the main issue manager. The manager treats E2E failures as CI failures and routes them to `code-reviewer` via the batch CI diagnosis protocol.
- `e2e-tester` cleans up any temporary containers, networks, or volumes created for testing. The `e2e-<issue#>` tab and pane remain open and idle for audit and debugging history.

**PR CI Policy & Manifest Discovery:**
- **Woodpecker CI is optional:** Inspect the repository for a Woodpecker PR manifest (e.g., `.woodpecker/pr.yaml`, `.woodpecker/pr.yml`, or a root `.woodpecker.yml`/`.woodpecker.yaml` triggering on pull requests) and check if Woodpecker CI is configured for the repo. If the project does NOT have a Woodpecker CI manifest for PRs (or Woodpecker CI is not active), Woodpecker CI pass is optional; do not block PR progress or wait for nonexistent Woodpecker pipelines.
- **Handling Woodpecker CI 401 / 403 (User not authorized / expired token):** If `wpci` returns `HTTP 401` or `HTTP 403: User not authorized` and pipeline logs cannot be fetched via CLI:
  1. Use `gh pr checks` for aggregate PR check status.
  2. Reproduce the test step in an ephemeral container matching the CI image if Docker/Podman is available, OR
  3. Escalate to the user with the direct Woodpecker pipeline URL and request the failed step traceback or refreshed credentials. Never guess test failures blindly.
- **CI Compose Containers Option:** If the project has a CI/test Docker Compose manifest (e.g., `docker-compose.ci.yml`, `docker-compose.test.yml`, `compose.ci.yml`, `compose.test.yml`, or similar test compose files), run the CI compose containers (e.g., `docker compose -f <manifest> up --build --abort-on-container-exit`) to validate the PR head locally. Failures in CI compose containers are treated as CI failures and diagnosed via the batch CI protocol. Ensure containers and temporary resources are cleaned up after runs.
- **Other Checks & Native Fallback:** If GitHub Actions or other required status checks are active on the PR, monitor them via `gh pr checks`. If neither Woodpecker PR CI nor a CI compose manifest exists, rely on repository test/lint suites (e.g. `npm test`, `pytest`, `cargo test`, `make test`).

`pr-reviewer` reviews the PR diff and returns draft findings privately to the manager. The manager validates the findings (removes false positives, stale/duplicate findings, unsupported preferences, unrelated redesign, and already-resolved issues). After validation, `pr-reviewer` submits a formal GitHub review. Whenever `pr-reviewer` or `e2e-tester` completes a turn, the manager reports the agent as `idle` via `herdr pane report-agent` (demoting it down the priority list without stealing UI focus or switching tabs).

**Self-approval & Review Gate Rule:** If the reviewer identity is the author of the PR, GitHub disallows `APPROVE`. In that case, submit a `COMMENT` review stating that no actionable findings remain. The formal review gate is "no unresolved blocking, high, or medium actionable findings", not strictly "approved".

Confirmed code-change findings and CI fixes are sent to `code-reviewer` in the implementation tab. `code-reviewer` analyzes the findings, assigns necessary changes to its `imp-coder-*` sub-agents (recreating coders for new unrelated tasks or CI fix streams with clean context while leaving prior panes open), reviews the changes, and pushes to the PR. `pr-reviewer` then re-reviews the latest commit and submits a follow-up formal review.

Confirmed code-change findings and CI fixes are sent to `code-reviewer` in the implementation tab. `code-reviewer` analyzes the findings, assigns necessary changes to its `imp-coder-*` sub-agents (recreating coders for new unrelated tasks or CI fix streams with clean context while leaving prior panes open), reviews the changes, and pushes to the PR. `pr-reviewer` then re-reviews the latest commit and submits a follow-up formal review.

When CI or E2E tests fail (whether Woodpecker CI, CI compose containers, E2E tests, or other required checks), the manager collects all relevant failed jobs/logs for the same latest head SHA before sending fixes. It prepares one consolidated CI correction task and sends it to `code-reviewer`. `code-reviewer` assigns fixes to its sub-agents, reviews the combined result, creates exactly one CI-fix commit, and pushes exactly once. See orchestration reference for full batch CI diagnosis protocol.

**PR Readiness Transition (Draft to Open):**
The PR remains in **draft** status throughout the review, E2E, and CI iterations. Once the latest PR commit satisfies all criteria:
1. Formal PR review has no unresolved blocking, high, or medium actionable findings.
2. Required CI / validation passes (passing Woodpecker CI if manifest exists, or passing CI compose containers, or passing repository test suite).
3. E2E tests pass (if E2E manifests/scripts exist in the repository; skipped if none).
The manager sets the PR as open / ready for review (`gh pr ready <pr-number>`). Do not mark the PR as ready for review before these criteria are met.

### Stage 4. Post-validation origin/main rebase gate

After formal PR review, E2E testing (if present), and required CI / validation (with Woodpecker CI optional when no PR manifest exists, verified via CI compose containers if available or native test suite) all pass for the same latest commit and the PR is marked ready for review, fetch `origin/main` without pulling or merging it. Check whether the feature branch contains the latest `origin/main`.

If `origin/main` has not advanced beyond the feature branch base, continue to final completion. If it has diverged, the main issue manager rebases the feature branch onto `origin/main`. Before rebasing, require a clean working tree and record a recoverable pre-rebase branch or commit reference.

Resolve every conflict semantically. Preserve both the current `origin/main` behavior and the issue branch's intended behavior and acceptance criteria. Do not resolve conflicts by blindly choosing one side, discarding commits, or overwriting unrelated changes. Use `code-reviewer` and its `imp-coder-*` agents for code-level reconciliation when needed.

After a successful rebase, review the complete result, run targeted validation where useful, and push with `--force-with-lease`, never unrestricted `--force`. The rebase creates a new PR head and invalidates all prior review and CI completion evidence. Restart the entire PR stage: perform a fresh full formal review and run all required CI, E2E, and validation for the rebased head. After they pass, fetch and check `origin/main` again; repeat the rebase-and-revalidate loop if it advanced again.

### Stage 5. Finish

Use only review, E2E, and CI results for the same latest PR head. Finish only when the branch was renamed, the plan was posted and approved by `plan-reviewer`, the implementation changes were reviewed and approved by `code-reviewer`, all spawned agents across retained tabs and panes have been reported as `idle` via `herdr pane report-agent` (ensuring no lingering `done` agents remain at the top of the priority list, achieved non-intrusively without stealing UI focus), the formal PR review has no unresolved blocking/high/medium actionable findings, accepted low-risk findings are documented, all required CI checks (or CI compose container validation if Woodpecker PR CI is absent/optional) and E2E tests (if present) pass on the latest commit, the PR is marked as open (ready for review, no longer draft), the post-validation `origin/main` divergence check passes, any required rebase has been followed by a complete fresh PR review, E2E, and CI cycle, and the PR remains open and unmerged. After verifying this completion gate, the main issue manager sends the single user-facing completion notification and final report; no delegated agent sends a separate done notification.

Report the branch name, PR number and URL, latest commit SHA, implementation summary, validation, E2E, and CI results (noting whether Woodpecker CI, CI compose containers, or repo test suite was used), review outcome, and accepted risks or follow-ups.
