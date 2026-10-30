# Herdr orchestration and review protocol

Read and follow this reference whenever the GitHub Issue PR Manager delegates work.

## Mandatory proclaude and procodex launchers

Every delegated agent must be launched by executing `proclaude` or `procodex` according to the required profiles. This overrides Herdr's normal `herdr agent start` recommendation.

- Never use `herdr agent start ... --kind codex` or any direct unmanaged launcher.
- Never invoke `claude`, `claude.exe`, `codex`, `codex.exe`, or an alias resolving directly to Claude Code or Codex.
- Create and label the pane first, then run proclaude or procodex with `herdr pane run`.
- Fail closed if proclaude or procodex is unavailable; never substitute direct Claude Code or Codex.

Default profiles (verify aliases with `proclaude --list-models` before launch):

| Agent | Launcher | Default/baseline |
|---|---|---|
| `imp-planner` | `proclaude --model "$IMP_PLANNER_MODEL"` | `pro-adaai` |
| `plan-reviewer` | `proclaude --model advanced-adaai` | `advanced-adaai` |
| `code-reviewer` | `proclaude --model "$CODE_REVIEWER_MODEL"` | `pro-adaai` |
| `imp-coder-1` through `imp-coder-3` | `proclaude --model lite-adaai` | `lite-adaai` |
| `pr-reviewer` | `proclaude --model advanced-adaai` | `advanced-adaai` |
| `e2e-tester` | `proclaude --model lite-adaai` | `lite-adaai` |

### Complexity-based model selection for the planner and code reviewer

`pro-adaai` is the requested baseline for both `imp-planner` and `code-reviewer`. Classify the work from the issue, repository evidence, and (for `code-reviewer`) the approved plan before launching the agent. Use `advanced-adaai` only for a clearly low-complexity task: a localized, well-understood change with straightforward acceptance criteria and no security, authorization, data-integrity, migration, concurrency, financial, API-compatibility, or cross-service implications. Use the `pro-adaai` baseline for medium/high complexity, any high-impact work, and whenever complexity is uncertain. If scope or risk increases before `code-reviewer` starts, retain Pro. Do not switch a persistent agent's model mid-session; choose before launch. Keep all other agent profiles unchanged.

Benchmark snapshot (2026-10-31, this machine): `proclaude --list-models` exposed `copilot-gpt-4.1`, `copilot-gpt-5.4-nano`, `copilot-search-a`, `lite-adaai`, `plus-adaai`, `advanced-adaai`, and `pro-adaai`. One identical no-tool prompt covered small, medium, and complex plans plus a role-authorization diff review. `advanced-adaai` completed in 9s (reported `$0.0314`) and produced a sound plan and caught the self-role escalation; `copilot-gpt-5.4-nano` completed in 10s (`$0.0466`) and caught the same issue, though its complex plan was less thorough. `pro-adaai` and `plus-adaai` returned no usable answer after reporting budget exhaustion (reported `$0.1793` and `$0.1058`, respectively, despite a `$0.10` CLI budget); `lite-adaai`, `copilot-gpt-4.1`, and `copilot-search-a` timed out at 45s without a usable result. Treat these as single-run observations, not a model leaderboard or proof of quality/cost; the budget setting did not act as a hard spend cap. Do not make further benchmark calls without an explicit spend limit. The incomplete runs are inconclusive, not evidence of poor model quality.

`pro-adaai` is AdaAI Pro (Complex), `advanced-adaai` is AdaAI Advanced (Challenging), and `lite-adaai` is AdaAI Lite. These profiles use always-on thinking; do not pass `--effort`. Do not invent or substitute aliases: re-verify them in `proclaude --list-models` (or the catalog at `~/.adaai/state/catalog-cache.json` / `~/.proxy-claude/state/catalog-cache.json`) before launch. If the selected `advanced-adaai` alias is unavailable, fall back to the `pro-adaai` baseline; if the baseline itself is unavailable, stop and report the blocker rather than switching launchers.

For every agent: create its pane with the correct cwd and `--no-focus`; parse the pane ID from JSON; label it with `herdr pane rename <pane-id> <agent-name>`; wait for the interactive shell prompt (sleep 2) to prevent login banner input drops; run the exact proclaude or procodex command with `herdr pane run`; wait for an agent to occupy the pane (at most 15 checks, 2s apart, reading `herdr agent list` for that pane); assign the matching Herdr agent name with `herdr agent rename <pane-id> <agent-name>`; inspect `herdr pane process-info <pane-id>` to verify the ProClaude/procodex wrapper started before sending tasks. A bare Claude Code or Codex process is acceptable only when proclaude/procodex startup is evidenced.

If Herdr cannot recognize proclaude or procodex as an agent, use pane send/read/wait commands; never fall back to direct Claude Code or Codex. Parse all IDs from Herdr JSON.

Delegated agents inherit their harness settings (`~/.claude/settings.json` or Codex configuration) and run within their assigned scope. Only `code-reviewer` is authorized to commit and push implementation code and CI fixes to the feature branch. All other delegated agents are strictly prohibited from performing commit, push, merge, rebase, reset, or force operations. If an agent still hits an unexpected permission prompt, apply the permission-prompt detection and escalation rules in Passive supervision below.

Every task prompt must include repository path, branch, issue context, acceptance criteria, exact scope, write boundaries, prohibitions, repository instructions, expected handoff, and known assumptions or risks. It must also include the notification restriction below, plus this mandatory simplicity and anti-overcomplication directive:

> STRICT ANTI-OVERCOMPLICATION DIRECTIVE: Deliver the minimal sufficient solution for the assigned scope: the smallest correct change consistent with repository conventions. Do not overcomplicate, over-abstract, or over-engineer. Do not introduce new layers, abstractions, helpers, wrappers, configuration, or dependencies unless strictly necessary to satisfy the acceptance criteria. Do not refactor, clean up, reformat, or "improve" code outside the assigned requirement. Defer all optional ideas; mention them in your handoff instead of implementing them. Finish your assigned deliverable cleanly rather than broadening it.

Reject plans and diffs that overcomplicate the task: treat unnecessary new abstractions, layers, modules, configuration, or dependencies as concrete findings and send them back for simplification. Similarly, ensure reviewers do not demand theoretical refactoring, over-abstraction, or unnecessary architectural complexity in their review comments.


## Notification ownership

The main issue manager is the only agent authorized to send a user-facing workflow-completion notification. No delegated agent (`imp-planner`, `plan-reviewer`, `code-reviewer`, `imp-coder-1`, `imp-coder-2`, `imp-coder-3`, `pr-reviewer`, `e2e-tester`) may send desktop, system, Herdr, or other out-of-band completion notifications.

Include this instruction in every delegated-agent prompt:

> Do not notify the user when you finish. Do not invoke a Herdr notification command or any desktop/system notification. Leave your concise completion handoff in this agent session, enter a settled or idle state, and wait for your supervising agent to inspect it. Report blockers and necessary escalations through this session only.

Completion handoffs follow the supervision hierarchy:

- `imp-planner` reports only to `plan-reviewer`.
- `imp-coder-*` reports only to `code-reviewer`.
- `plan-reviewer`, `code-reviewer`, `pr-reviewer`, and `e2e-tester` report only to the main issue manager.
- `pr-reviewer` may still publish its separately authorized formal GitHub review and inline comments; those PR artifacts are not completion notifications.

The supervising agent detects completion through Herdr state and recent history. Do not ask a subordinate to send a notification and do not rely on user-facing notifications to coordinate the workflow.

If a subordinate needs user input, it reports the blocker to its supervisor. Only the main issue manager may ask the user for input or issue a user-facing blocked-status message. At final completion, the main issue manager sends one completion notification after verifying the completion invariant; it must not relay separate done notifications for each subordinate.

## Supervision hierarchy

```
main issue manager
├── plan-reviewer (Stage 1)
│   └── imp-planner
├── code-reviewer (Stage 2, persists into Stage 3)
│   ├── imp-coder-1
│   ├── imp-coder-2 (optional)
│   └── imp-coder-3 (optional)
├── pr-reviewer (Stage 3, main tab right pane)
└── e2e-tester (Stage 3, dedicated e2e-<issue#> tab)
```

The main issue manager supervises `plan-reviewer`, `code-reviewer`, `pr-reviewer`, and `e2e-tester`. `plan-reviewer` supervises `imp-planner`. `code-reviewer` supervises `imp-coder-*` agents. Delegation does not transfer ownership.

## Passive supervision

Do not require scheduled or midpoint progress reports. Subordinates work uninterrupted and provide only a concise completion handoff or a necessary escalation when blocked, materially uncertain, outside their write set, overlapping another assignment, or facing a required shared-interface or unsafe-operation change.

While waiting, inspect non-intrusively with `herdr agent get`, `herdr agent read --source recent-unwrapped`, `herdr pane read --source recent-unwrapped`, and `herdr pane process-info`. Inspect at meaningful boundaries: after assignment, when editing begins, before broad formatting/generation/dependency operations, after unexpected files appear, when work lasts unexpectedly long, when assumptions change, when blocked, when a pending permission or confirmation prompt is visible, and before accepting completion. Do not send status prompts or poll continuously.

### Permission-prompt detection and escalation

A delegated agent waiting at an interactive permission prompt (for example a `Bash(git ...)` approval dialog) is blocked: it produces no completion handoff, no escalation, and no notification. Supervisors must detect this state actively instead of waiting for the subordinate to report it.

On every inspection pass, check each subordinate pane for a visible pending approval prompt — an unanswered tool-permission or confirmation dialog in recent `herdr pane read --source recent-unwrapped` output, or a session whose last output is a permission question with no subsequent activity.

When a pending permission prompt is found:

1. Classify the requested command against the agent's assigned scope, write set, and the delegated-agent prohibitions.
2. If the command is prohibited or outside the agent's write set, deny it through the pane (send the decline choice), then redirect the agent. This is a violation to correct, not a prompt to approve.
3. If the command is legitimately in scope but requires user permission, the supervisor escalates up the hierarchy until it reaches the main issue manager, who is the only agent that may ask the user for approval. Present the agent name, the exact command, its purpose, and the in-scope justification.
4. After the user decides, the main issue manager returns the decision down the hierarchy to the subordinate pane. Do not approve a command on the user's behalf without asking.
5. Treat an unhandled permission prompt as an active blocker: include it in every subsequent inspection pass until resolved, and do not count the subordinate as making progress while it waits.

Redirect or interrupt an agent when history shows scope drift, duplicate work, unrelated refactoring, unapproved shared-file changes, unnecessary dependency or lockfile changes, broad formatting/generation, excessive testing, repeated failed approaches, overcomplication (new abstractions, layers, wrappers, or tooling the task does not need), speculative "while I'm here" improvements, review pedantry/overengineering demands, or prohibited Git/GitHub operations. State what must stop, the allowed scope, and the next concrete action. Preserve valid work and inspect status before correcting anything.

### Non-intrusively updating "done" agents to "idle" without stealing focus

In Herdr, an agent that finishes background work in an unfocused pane or tab enters the `done` state. In Herdr's UI, sidebar, and priority listings, agents in `done` state are considered unread/unseen background work and are pinned to the top of the priority list.

**Do NOT use `herdr agent focus` or `herdr tab focus` to clear `done` status.** Focusing steals the user's active UI focus and switches their active tab/terminal while they may be interactively typing or prompting another agent.

Instead, update the agent's lifecycle state non-intrusively using `herdr pane report-agent`:

```bash
herdr pane report-agent <pane-id> --source herdr:cli --agent <agent-label> --state idle
```

To resolve the pane ID and agent label from the agent name:
```bash
PANE_INFO=$(herdr agent get <agent-name> 2>/dev/null)
PANE_ID=$(echo "$PANE_INFO" | jq -r '.result.agent.pane_id')
AGENT_LABEL=$(echo "$PANE_INFO" | jq -r '.result.agent.agent // "claude"')

herdr pane report-agent "$PANE_ID" --source herdr:cli --agent "$AGENT_LABEL" --state idle
```

This immediately transitions the agent and its tab from `done` to `idle` in Herdr, demoting it down the priority list without switching tabs, moving focus, or interrupting the user.

Supervisors must apply this non-intrusive state update whenever a subordinate finishes its deliverable or turn:
- **`plan-reviewer`**: Reports `imp-planner` as `idle` when it finishes plan drafts.
- **Main issue manager**: Reports `plan-reviewer` and `imp-planner` as `idle` once the plan is approved.
- **`code-reviewer`**: Reports each `imp-coder-*` as `idle` when it finishes its code deliverable.
- **Main issue manager**: Reports `code-reviewer` as `idle` after implementation commits and pushes are complete.
- **Main issue manager**: Reports `pr-reviewer` as `idle` after it completes draft findings or formal reviews.
- **Main issue manager**: Reports `e2e-tester` as `idle` after E2E test runs settle in `e2e-<issue#>`.

### Preservation of spawned tabs and panes across stages

Do NOT close, kill, or destroy spawned Herdr tabs or agent panes upon stage transitions.
- The `planning-<issue#>` tab and its panes (`plan-reviewer`, `imp-planner`) remain open and idle after Stage 1 finishes so their plans, reviews, and logs are preserved for inspection.
- The `implementation-<issue#>` tab and its panes (`code-reviewer`, `imp-coder-*`) remain open throughout Stage 2, Stage 3, and subsequent stages.
- The main tab's `pr-reviewer` pane remains open after review completion.
- The `e2e-<issue#>` tab and its `e2e-tester` pane remain open and idle after test execution for audit and log inspection.
- Retaining tabs and panes ensures full auditability, debugging history, and provenance throughout the entire issue lifecycle.

### Strict Workspace Confinement Rule
Every tab spawned by this workflow (`planning-<issue#>`, `implementation-<issue#>`, `e2e-<issue#>`) MUST be explicitly bound to the current workspace by passing `--workspace "$HERDR_WORKSPACE_ID"` to `herdr tab create`. Never call `herdr tab create` without `--workspace "$HERDR_WORKSPACE_ID"`. Omitting `--workspace` causes Herdr to default to whichever workspace is currently active/focused on the server, leaking tabs into parent epic manager workspaces or unrelated workspaces.

## Stage 1: Planning tab protocol

### Tab topology

Record parent pane ID for Stage 3 splitting:
```bash
PARENT_PANE_ID="${HERDR_PANE_ID}"
```

Spawn Herdr tab `planning-<issue#>` inside the current workspace:

```bash
PLAN_TAB_JSON=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label "planning-${ISSUE_NUMBER}" --no-focus)
PLAN_TAB_ID=$(echo "$PLAN_TAB_JSON" | jq -r '.result.tab.tab_id // .result.tab_id')
PLAN_REVIEWER_PANE_ID=$(echo "$PLAN_TAB_JSON" | jq -r '.result.root_pane.pane_id // .result.pane_id')
PLAN_PLANNER_PANE_ID=$(herdr pane split "$PLAN_REVIEWER_PANE_ID" --direction right --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')

herdr pane rename "$PLAN_REVIEWER_PANE_ID" "plan-reviewer"
herdr pane rename "$PLAN_PLANNER_PANE_ID" "imp-planner"

# Wait for interactive shell prompt before running launcher to prevent banner race
sleep 2

herdr pane run "$PLAN_REVIEWER_PANE_ID" "proclaude --model advanced-adaai"
herdr pane run "$PLAN_PLANNER_PANE_ID" "proclaude --model ${IMP_PLANNER_MODEL}"

# Wait for agent to occupy each pane (bounded: at most 15 checks, 2s apart)
for p in "$PLAN_REVIEWER_PANE_ID" "$PLAN_PLANNER_PANE_ID"; do
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

herdr agent rename "$PLAN_REVIEWER_PANE_ID" "plan-reviewer"
herdr agent rename "$PLAN_PLANNER_PANE_ID" "imp-planner"

# Confirm wrappers started
herdr pane process-info "$PLAN_REVIEWER_PANE_ID"
herdr pane process-info "$PLAN_PLANNER_PANE_ID"
```

```text
+--------------------------+--------------------------+
| plan-reviewer            | imp-planner              |
|                          |                          |
|                          |                          |
+--------------------------+--------------------------+
```

### Autonomous plan review loop

The manager spawns both agents and sends the initial task to `imp-planner` (issue context, acceptance criteria, repo instructions, scope). `plan-reviewer` receives a prompt instructing it to review `imp-planner`'s output iteratively.

`plan-reviewer` autonomously iterates with `imp-planner`:
1. `imp-planner` produces a minimal implementation plan covering intended behavior, acceptance criteria, relevant files, ordered changes, risks, validation, assumptions, and open questions.
2. `plan-reviewer` reviews the plan against repository evidence, codebase structure, and acceptance criteria.
3. If the plan has findings (missing coverage, incorrect assumptions, overcomplicated approach, unnecessary scope, wrong files), `plan-reviewer` sends concrete findings back to `imp-planner` requesting revisions.
4. `imp-planner` revises and resubmits.
5. Repeat until `plan-reviewer` determines the plan is sound, minimal, and implementation-ready.
6. `plan-reviewer` signals approval in its handoff to the manager.

The manager monitors the planning tab passively. It does NOT inject itself into the plan review loop. Once `plan-reviewer` signals approval, the manager:
1. Reads the approved plan from `imp-planner`'s output.
2. Posts the approved plan to the GitHub issue.
3. Preserves the planning tab and its agent panes (`plan-reviewer`, `imp-planner`) open and idle; do not close or kill them upon entering Stage 2.

Do not begin implementation before the approved plan is posted.

## Stage 2: Implementation tab protocol

### Tab topology

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

```text
+--------------------------+--------------------------+
|                          | imp-coder-1              |
|                          +--------------------------+
| code-reviewer            | imp-coder-2 (optional)   |
|                          +--------------------------+
|                          | imp-coder-3 (optional)   |
+--------------------------+--------------------------+
```

For single-coder work (default), only `imp-coder-1` appears on the right.

### code-reviewer responsibilities

`code-reviewer` is the lead agent in the implementation tab. It:
1. Reads the approved plan and decomposes it into assignments.
2. Decides whether to use one coder (default) or up to three coders for substantial independent workstreams with disjoint write sets, stable interfaces, and lower integration cost than serial work.
3. Sends each assignment to its `imp-coder-*` sub-agents with exact scope, write boundaries, and prohibitions.
4. Reviews each integrated diff for correctness, acceptance-criteria coverage, regressions, compatibility, security, data integrity, API changes, test quality, scope creep, artifacts, secrets, and unrelated edits.
5. Requests changes until review results contain at most acceptable (low/nit) findings.
6. After all changes are approved, creates scoped commits and pushes the branch.

### imp-coder-* agents

At most three `imp-coder-*` agents may be active concurrently.

- **Recreate coders for new unrelated tasks:** Whenever a task is unrelated to a previous task (such as a distinct workstream, a new CI failure batch, or rebase conflict reconciliation), recreate the coder by launching a fresh `imp-coder-*` agent in a new pane with clean context. Reusing coder sessions across unrelated tasks pollutes agent context with stale code, prior discussions, and obsolete assumptions, leading to scope drift and hallucinations.
- **Same-task iteration:** A coder session should only be continued when actively iterating on direct review revisions, feedback, or fixups for its *own* ongoing assignment.
- **Preserve prior coder panes:** Leave completed or idle coder panes open and undisturbed for audit, debugging, and provenance; do not close or kill coder panes when spawning a replacement. Ensure at most three coders are actively executing concurrently.
- **Decomposition required:** Every new code-change cycle requires a fresh decomposition from `code-reviewer`.
- **Worktree isolation and write boundaries:** Prefer isolated temporary Git worktrees for concurrent writers. Coders must stay within explicit write sets and must not modify shared exports, configuration, migrations, generated files, dependencies, or lockfiles unless assigned. They must not commit, push, publish, merge, or rewrite history. Before removing temporary worktrees, verify paths and confirm accepted changes are integrated.

### Persistent code-reviewer standby

`code-reviewer` is a persistent lead agent, not a disposable per-stage worker.

- Launch it once in the implementation tab after the approved plan is posted.
- Keep its proclaude process and context alive through Stage 2, Stage 3, and any rebase cycles.
- When no code change is currently assigned, leave it at its interactive prompt/on standby.
- Send new confirmed findings (from `pr-reviewer` or CI failures) to the existing `code-reviewer` session.
- Do not close and respawn `code-reviewer` between implementation, PR review, CI, and rebase cycles.
- `imp-coder-*` agents remain disposable and must be recreated for new unrelated tasks with clean context, subject to the maximum of three active coders and a fresh approved decomposition.
- Close `code-reviewer` only after the PR is review-ready, required CI passes on the latest commit, and no further code work is reasonably expected.

If the persistent proclaude session exits unexpectedly, record the failure, verify the pane and working directory, and restore the named `code-reviewer` role only after confirming that the original session cannot be recovered. Do not replace it with direct Claude Code or Codex.

### Manager role during Stage 2

The manager monitors the implementation tab passively. It does NOT review diffs or approve code changes — that is `code-reviewer`'s responsibility. Once `code-reviewer` signals that all changes are reviewed, committed, and pushed, the manager:
1. Reviews the final commit history for scope (not code correctness — that was `code-reviewer`'s job).
2. Opens a PR in **draft** status linked to the issue (`gh pr create --draft ...`). Use `Fixes #<number>` when appropriate.
3. Proceeds to Stage 3.

## Stage 3: PR review, E2E testing, and CI

### Tab topology

**Main tab (`main`):**
```text
+--------------------------+--------------------------+
|                          | pr-reviewer              |
| main issue manager       |                          |
|                          |                          |
+--------------------------+--------------------------+
```

**Dedicated E2E tab (`e2e-<issue#>`):**
```text
+-----------------------------------------------------+
| e2e-tester                                          |
|                                                     |
|                                                     |
+-----------------------------------------------------+
```

The implementation tab with `code-reviewer` remains open and on standby.

### Adaptive review, E2E testing, and CI prioritization

Start CI, E2E testing, and PR review concurrently against the draft PR. Do not force review-first, test-first, or CI-first ordering. The main issue manager chooses the next safe action using severity, blocking impact, expected fix size, likely shared root cause, chance of invalidating in-progress validation, CI duration, and failure classification.

Prefer an immediate CI or E2E correction when it reveals a fundamental compile, typecheck, startup, security, data-integrity, migration, or compatibility problem, or when it makes meaningful review impossible. Prefer a review correction when it is high impact, small and safe, likely to resolve CI/test failures, or likely to make a long-running CI/test result stale. Continue investigation instead of changing code when evidence suggests flakiness, infrastructure failure, ambiguity, or imminent decisive output.

After every pushed code change, re-evaluate all sources against the new head SHA. Results from older commits are historical only. Use only checks, E2E results, and review outcomes associated with the same latest commit for completion.

### E2E testing protocol (`e2e-tester` in tab `e2e-<issue#>`)

In Stage 3, the manager spawns `pr-reviewer` in the main tab (right pane) and `e2e-tester` in a dedicated Herdr tab `e2e-<issue#>` inside the current workspace:

```bash
# 1. Main tab: split right for pr-reviewer
PR_REV_PANE=$(herdr pane split "$PARENT_PANE_ID" --direction right --no-focus | jq -r '.result.pane_id // .result.pane.pane_id')
herdr pane rename "$PR_REV_PANE" "pr-reviewer"

# 2. Dedicated tab: e2e-<issue#>
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

1. **Manifest discovery:**
   - `e2e-tester` inspects the repository for End-to-End or system integration tests, browser test suites, or test compose files (e.g., `playwright.config.*`, `cypress.config.*`, `cypress.json`, `puppeteer`, `selenium`, `docker-compose.e2e.yml`, `compose.e2e.yml`, `npm run test:e2e`, `pytest tests/e2e`, `make e2e`).
   - If no E2E manifests or test commands exist in the repository, `e2e-tester` records that E2E testing is absent/skipped, enters an idle state, and reports this to the manager. For repos with no E2E suite, `skipped` is the expected result and does not block PR readiness.

2. **Test execution:**
   - If an E2E suite is found, `e2e-tester` spins up necessary dependencies/containers, executes the tests against the draft PR head commit, and records output. Label containers with `e2e-${ISSUE_NUMBER}` and ensure `$TMPDIR` defaults to `/tmp`.
   - Run tests in headless/non-interactive mode suitable for CI/automation.

3. **Failure reporting & cleanup:**
   - If any test fails, `e2e-tester` captures complete untruncated failure traces, console errors, stack traces, and pointers to screenshots/videos/reports, and sends the report to the manager.
   - The manager incorporates E2E failures into the batch CI diagnosis protocol and routes them to `code-reviewer` for fixing.
   - `e2e-tester` tears down test containers and volumes (`docker compose -f <manifest> down -v`).
   - The `e2e-<issue#>` tab and pane remain open and idle for audit trail and log inspection.

### PR CI discovery and validation policy

At the PR stage, discover and establish the authoritative CI validation method for the repository:

1. **Check for Woodpecker PR CI manifest:**
   - Inspect `.woodpecker/pr.yaml`, `.woodpecker/pr.yml`, or root `.woodpecker.yml`/`.woodpecker.yaml` for pull request event triggers (`event: pull_request`).
   - If a Woodpecker PR manifest exists and Woodpecker CI is configured for the repo, monitor Woodpecker CI for the PR head SHA (e.g. via `wpci` or status checks).
   - **Woodpecker CI pass is optional:** Some projects do not have a Woodpecker CI manifest for PRs or do not have Woodpecker enabled on the repository. Do not stall or fail the PR stage waiting for a nonexistent Woodpecker pipeline. If no Woodpecker PR manifest is found or Woodpecker CI is not active for PRs, Woodpecker CI pass is optional and must not block completion.
   - **Handling Woodpecker CI 401 / 403 (User not authorized / expired token):** If `wpci` returns `HTTP 401` or `HTTP 403: User not authorized` and pipeline logs cannot be fetched via CLI:
     - Use `gh pr checks` for aggregate PR check status.
     - Reproduce locally in an ephemeral container matching the CI image if Docker/Podman is available, OR
     - Escalate to the user with the direct Woodpecker pipeline URL and request the failed step logs or refreshed credentials. Never guess failures blindly.

2. **CI / Test Compose containers alternative:**
   - If the repository has a CI or test Docker Compose manifest (e.g. `docker-compose.ci.yml`, `docker-compose.test.yml`, `compose.ci.yml`, `compose.test.yml`, `docker-compose.check.yml`, `compose.check.yml`, or compose files defining CI/test services), run the CI compose containers locally to validate the PR head:
     - Run `docker compose -f <manifest> up --build --abort-on-container-exit` (or the repository's specified compose test command).
     - Capture exit codes and full container logs.
     - Always clean up compose containers and ephemeral networks/volumes when finished (`docker compose -f <manifest> down -v`).
     - If any CI compose container exits with an error, treat it as a CI failure: pass the full untruncated logs into the batch CI diagnosis protocol.

3. **External status checks & native test fallback:**
   - If GitHub Actions or other required status checks are active on the PR, monitor them via `gh pr checks`.
   - If neither a Woodpecker PR manifest nor a CI compose manifest exists, execute the repository's native test/lint commands (e.g. `npm test`, `pytest`, `cargo test`, `make test`) before declaring validation complete.

### Draft PR status and readiness transition

The PR is initially created in **draft** status (`gh pr create --draft`). This prevents premature notification or review by external contributors until internal validation completes.

- `pr-reviewer` conducts its review, `e2e-tester` executes E2E test suites in `e2e-<issue#>`, and CI validation (Woodpecker CI if manifest exists, or CI compose container runs / fallback suites) proceeds while the PR is in draft status.
- Once the latest PR head commit passes formal PR review (no unresolved blocking, high, or medium actionable findings), satisfies required CI validation criteria (passing Woodpecker CI if present, or passing CI compose containers, or passing repository test suite), AND passes E2E testing (if E2E tests exist in the repository; skipped if none), the manager converts the PR from draft to open / ready for review via `gh pr ready <pr-number>`.
- The PR must not be converted to ready for review until formal review, E2E tests (if present), and required CI / validation criteria have all passed on the latest commit.

### PR review flow

`pr-reviewer` is read-only for repository files and Git history. It may write only review artifacts on the managed PR: formal reviews, inline comments, review summaries, and necessary follow-up replies. It must not edit code, commit, push, merge, close, change PR metadata, or manage labels.

Before publishing, it reads the issue, approved plan, repository instructions, PR description, complete diff, surrounding code, existing conversations, CI state, and exact latest head SHA. It returns proposed findings privately to the main issue manager with severity (`blocking`, `high`, `medium`, `low`, or `nit`), path/location, impact, evidence, correction, and whether each belongs inline or in the summary.

The manager validates the findings and removes false positives, stale or duplicate findings, unsupported preferences, unrelated redesign, and already-resolved issues. Immediately before submission, `pr-reviewer` rechecks the head SHA and refreshes findings and inline locations if it changed.

After validation, submit a formal GitHub review. Use inline comments for a specific changed line or compact range when that location helps correction. Use the summary for cross-cutting concerns, missing behavior without a suitable changed line, architectural or acceptance gaps, CI concerns, and limitations. Include the reviewed SHA, disposition, finding counts by severity, acceptance assessment, and observed CI state.

Use `REQUEST_CHANGES` for validated blocking, high, or medium actionable findings; `COMMENT` for low, nit, informational findings, or when approval is not permitted; and `APPROVE` only when no actionable findings remain and repository policy permits it. Never request changes for nits alone. If the identity cannot approve its own PR (GitHub disallows author self-approval), use `COMMENT` to state that no actionable findings remain. The formal review gate is "no unresolved blocking, high, or medium actionable findings", not strictly "approved".

### Routing findings to code-reviewer

After `pr-reviewer` submits a formal review, the manager sends confirmed code-change findings to `code-reviewer` in the implementation tab. `code-reviewer` analyzes the findings, assigns necessary changes to its `imp-coder-*` sub-agents, reviews the changes, and pushes to the PR.

After fixes are pushed, `pr-reviewer` reviews the latest SHA and submits a follow-up formal review. The manager decides whether findings are resolved, accepted, false positives, or out of scope.

### Batch CI diagnosis and untruncated handoff

When CI or E2E tests fail for a PR head (Woodpecker CI, CI compose container run, E2E tests, or other required checks):

1. Identify the exact latest PR head SHA.
2. Allow all relevant jobs for that SHA to finish when practical; do not fix the first visible failure and immediately restart the cycle.
3. Collect every failed, cancelled, or dependent job (or failing CI compose container, or failing E2E test suite) that may be caused by the patch, including complete job names, container names, commands, exit statuses, failure messages, relevant log sections, links, and artifact paths.
4. Classify each result as patch-related, flaky, external, infrastructure, or blocked by another failure.
5. Group failures with a common root cause and identify independent failures.
6. Prepare one consolidated CI/test correction task and send it to `code-reviewer` covering all patch-related failures discovered for that head SHA.
7. Do not send one prompt per failing job when the failures can be diagnosed together.
8. `code-reviewer` assigns the full confirmed CI failure set to its `imp-coder-*` sub-agents in one implementation cycle. It recreates coders for new unrelated tasks (providing clean context while preserving earlier panes open and idle). It may use up to three coders for disjoint fixes, but their results must be integrated before any CI-fix commit or push.
9. Do not create intermediate CI-fix commits and do not push partial fixes while `code-reviewer` or its coders are still addressing the batch.
10. After all known patch-related CI failures for the head SHA are fixed, `code-reviewer` reviews the complete combined diff.
11. `code-reviewer` creates exactly one cohesive CI-fix commit containing all approved fixes from that batch.
12. `code-reviewer` pushes exactly once after that commit. Do not use one commit or push per failed job.

The CI handoff must not truncate failure messages or logs. Prefer writing a complete Markdown report to a temporary file outside the repository, such as `%TEMP%\\github-issue-pr-manager\\ci-report-<head-sha>.md`, and pass the absolute path to `code-reviewer` with instructions to read the entire report. The report is authoritative and must include the head SHA, all job results, full failure messages, complete relevant log output, links, classification, suspected root causes, and the requested combined correction. Do not rely on a short prompt excerpt, limited terminal viewport, or fixed line cap as the sole handoff.

If the complete report fits in an interactive prompt, include it in full. Otherwise send the file path plus a concise index and verify that `code-reviewer` acknowledged reading the complete report before it starts assigning fixes.

If one failure prevents other jobs from starting, record that dependency, collect every available result, and send the currently complete report in one batch. After the combined fix is pushed, rerun and reassess the full CI set for the new head SHA. Do not treat the first visible failure as the only CI issue.
A CI batch is scoped to one source head SHA and the failures observable from its run. If a corrected run exposes failures that were previously hidden by an earlier blocking failure, treat those newly visible failures as a new CI batch: collect the complete new result set, produce one consolidated task, create one new CI-fix commit, and push once. Infrastructure-only or flaky failures that require no repository change produce no commit.

To conserve time and tokens, the manager must avoid repeated summaries of the same logs, duplicate prompts, per-job fix turns, and intermediate pushes. Use the single authoritative CI report, let `code-reviewer` and its coders complete the whole batch, then one review, one commit, and one push.

## Base-branch isolation and post-validation rebase

### Do not integrate origin/main early

During planning, implementation, code review, and the initial PR review/CI cycle:

- Do not run `git pull origin main` or any equivalent pull.
- Do not merge `origin/main` into the feature branch.
- Do not rebase the feature branch onto `origin/main`.
- Do not cherry-pick base-branch commits merely to update the branch.
- Do not rewrite the feature branch to follow new `origin/main` commits before the PR has first passed both formal review and required CI / validation on one head SHA.

A fetch that only updates remote-tracking metadata is allowed, but it must not modify the working branch. The required synchronization decision occurs only after the current PR head has completed a passing review and CI cycle.

### Check for divergence after PR validation passes

After both review and CI / validation pass for the same latest PR head and the PR is marked ready for review:

1. Confirm the working tree and index are clean.
2. Record the current branch name and pre-rebase commit SHA.
3. Run `git fetch origin main` or an equivalent fetch that updates `origin/main` without merging.
4. Determine whether `origin/main` is already an ancestor of the feature branch, for example with `git merge-base --is-ancestor origin/main HEAD` and `git rev-list --left-right --count origin/main...HEAD`.
5. If the feature branch already contains the latest `origin/main`, the rebase gate passes.
6. If `origin/main` contains commits not in the feature branch, create a recoverable local backup reference and rebase the feature branch onto `origin/main`.

Do not use `git pull` for this gate. Fetch and inspect first, then explicitly rebase only when needed.

### Safe rebase and conflict resolution

Before rebasing:

- Require a clean working tree and index.
- Record the exact pre-rebase SHA.
- Create a clearly named local backup branch or other recoverable Git reference at that SHA.
- Confirm the target is the freshly fetched `origin/main`.

The main issue manager owns the rebase operation. `code-reviewer` and its `imp-coder-*` agents remain available in the implementation tab for semantic code reconciliation (recreating coders for new unrelated conflict reconciliation tasks with clean context). Up to three coders may be used only for genuinely disjoint conflict groups after a fresh approved decomposition from `code-reviewer`.

For every conflict:

1. Inspect the base, feature-branch intent, new `origin/main` behavior, issue acceptance criteria, and relevant surrounding history.
2. Reconcile the code so the result preserves the intended behavior from both branches whenever those behaviors are compatible.
3. Do not blindly choose the feature side or `origin/main` side for an entire file.
4. Do not use blanket ours/theirs resolution, discard commits, or remove unfamiliar changes merely to make the rebase continue.
5. Preserve unrelated upstream changes and preserve all valid issue implementation behavior.
6. Update tests or integration code when both branches changed an interface or assumption.
7. Stage only files whose conflicts were deliberately resolved, then continue the rebase.

If requirements are genuinely mutually exclusive or preserving both behaviors is impossible, stop before discarding either change and escalate the exact semantic conflict to the main issue manager. Abort the rebase if necessary to return to the recorded pre-rebase state while the conflict is analyzed.

After the rebase completes:

- Confirm no commits were unintentionally dropped or duplicated.
- Compare the rebased branch against both `origin/main` and the pre-rebase backup reference.
- Review the complete diff, not only conflict markers.
- Verify that upstream behavior and issue acceptance criteria are both preserved.
- Run targeted validation where useful.
- Push the rewritten branch with `git push --force-with-lease`; never use unrestricted `--force`.

### Restart the PR stage after every rebase

A rebase creates a new PR head SHA. All earlier CI and formal-review results become historical and do not count toward completion.

After pushing the rebased branch:

1. Restart CI / validation, E2E testing, and PR review concurrently for the new head.
2. Require `pr-reviewer` to perform a fresh full review of the complete rebased PR, not only the conflict resolutions or new upstream commits.
3. Require a new formal GitHub review for the new head.
4. Run all required CI checks (or CI compose container validation if Woodpecker PR CI is absent/optional) and E2E tests (if present in repo) for the new head.
5. Apply the normal adaptive review/CI prioritization and batched CI-fix rules.
6. Keep the persistent `code-reviewer` and its coders available for corrections.
7. Complete review, E2E testing, and CI / validation on the same new head SHA.
8. Fetch `origin/main` and perform the divergence check again.
9. Repeat the rebase and full PR-stage restart if `origin/main` advanced again.

The workflow may finish only after a full review, E2E, and CI / validation cycle passes and the subsequent fetched `origin/main` divergence check confirms that another rebase is not required.

## Completion invariant

The workflow is complete only when the PR is marked open / ready for review (no longer draft), the latest PR head has both a satisfactory formal review, passing E2E tests (if present in repository), and all required CI checks passing (with Woodpecker CI optional when no PR manifest exists, verified instead via CI compose containers if available or native test suite), followed by a fresh `origin/main` divergence check that confirms no rebase is required. If a rebase was required, the rebased head must complete the entire PR review, E2E, and CI stage again and be marked ready for review. A green CI result or review for an older or pre-rebase commit never satisfies completion. After verifying this invariant and closing or settling remaining delegated work, the main issue manager sends the only user-facing completion notification and final report.
