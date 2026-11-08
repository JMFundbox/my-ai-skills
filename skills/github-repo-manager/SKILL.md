---
name: github-repo-manager
description: Run a long-lived repo-level manager that waits for the user to ask for GitHub issues or epics to be worked, then creates a Herdr worktree per issue/epic, launches a ProClaude manager agent in it with /goal (/github-issue-pr-manager for issues, /github-epic-issue-manager for epics), monitors those agents and their PRs, and consolidates merging — merging each mergeable PR serially, or telling the agent that its PR's target branch has diverged. Use when the user wants to dispatch, track, or merge several issue/epic PRs from one place; not for working a single issue yourself.
---

# GitHub Repo Manager

A repo-level dispatcher and merge coordinator. It does not implement anything. It turns "work issue #12 and epic #20" into isolated Herdr worktrees with a manager agent in each, watches them, and merges their PRs one at a time when they are genuinely mergeable.

This workflow requires Herdr. Read the user-level `herdr` skill, then read [references/orchestration.md](references/orchestration.md) for the command patterns before creating worktrees or prompting agents. If `HERDR_ENV` is not `1`, say so and stop. Read only the reference sections needed for the current step.

This skill dispatches [`github-issue-pr-manager`](../github-issue-pr-manager/SKILL.md) and [`github-epic-issue-manager`](../github-epic-issue-manager/SKILL.md). Those skills must be installed for the harness the dispatched agents run in; this package does not install them.

## Ownership and hard limits

The repo manager owns: intake, worktree/tab/pane/agent setup, dispatch prompts, monitoring, merge decisions, divergence notices, and user reporting.

It must not:

- Edit, commit, push, rebase, or run tests in any dispatched worktree, or `cd` into one to do work. Interact with dispatches only through `herdr` commands and `gh`.
- Check out, commit to, pull into, or otherwise alter the checkout it runs in. `git fetch` is allowed. The base branch is only ever changed by merging a PR on GitHub.
- Merge with `--admin`, bypass branch protection or a merge queue, or force a merge past a failing required check or unresolved review.
- Answer a `blocked` agent's permission or trust prompt on the user's behalf. Surface the prompt and ask.
- Close issues, delete branches, or remove worktrees except as stated in "After a merge".
- Add scope. Dispatch exactly what was asked, with the exact goal text below.

Dispatched agents own implementation, review, CI, and their own PR readiness. The repo manager trusts their readiness signal (non-draft PR plus settled agent) but re-verifies GitHub state before merging.

## Settings

Resolve these once per session; the user can override any of them in a request.

- **Manager model:** `advanced-adaai` (`proclaude --model advanced-adaai`). Keep it in a single `MANAGER_MODEL` variable.
- **Base branch:** user-specified, else the current branch of the checkout the manager runs in. It must exist on `origin`.
- **Merge method:** user-specified, else the only method the repo allows, else ask the user once and remember the answer for the session. `gh pr merge` has no usable non-interactive default, so never guess.
- **Merge mode:** `auto` (default — merge as soon as gates pass) or `confirm` (report merge-ready PRs and wait for the user). Switch when the user says "ask before merging" / "hold merges".

## Start

1. Run preflight (Herdr env, `gh auth status`, repo slug, default branch).
2. Adopt existing dispatches (see "Tracking") so a restarted manager picks up in-flight work.
3. Show the current dispatch table, if any, and a short list of open issues (`gh issue list --state open --limit 30`) so the user can pick.
4. Wait for a request. Accept forms like "dispatch #12", "issues #12 #15 and epic #20 from `release/1.2`", or "merge with rebase". If a request selects by query ("all open bugs") rather than by number, show the resolved list and get confirmation before dispatching.

## Dispatch (per issue or epic)

For each requested number, in order. Skip a number that fails a check, tell the user why, and continue with the rest.

**Checks**

1. Issue exists and is `OPEN`.
2. Kind: the user's word (issue/epic) wins. Otherwise treat it as an epic only if it has an `epic`/`umbrella` label, native sub-issues, or a task list of two or more issue references. If unclear, ask.
3. Not already dispatched: no worktree branch matching `^(issue|epic)-<N>(-|$)` and no open PR from such a branch. This also stops it from duplicating a sub-issue that an epic manager is already working (those use branch `issue-<N>`).
4. If the base branch is an epic branch being run by an epic manager, skip and say so; that epic manager merges its own sub-issue PRs.
5. Base is valid: `git fetch origin <base>` succeeds and local `<base>`, if it exists, has no commits missing from `origin/<base>`. If it does, stop and ask the user to push or confirm using the origin tip.
6. Soft cap: if more than 4 dispatches are already active, ask before adding more.

**Steps**

1. **Create the worktree** from `origin/<base>` with an appropriate name:
   - Branch: `issue-<N>-<short-kebab-title>` or `epic-<N>-<short-kebab-title>` (slug ≤ 30 chars). Epics use the exact name `github-epic-issue-manager` would pick, so it won't rename. If the title yields an empty slug (for example non-ASCII only), use `work`.
   - Worktree label: `issue-<N>` / `epic-<N>`.
   - Do not steal focus. Afterwards unset any upstream inherited from `origin/<base>` (so the child's first `git push -u origin <branch>` works) and verify the worktree's `HEAD` equals the fetched `origin/<base>`.
2. **Name the topology:** initial tab → `admin-<N>`; initial pane and its agent → `manager-<N>`.
3. **Start the agent:** run `proclaude --model advanced-adaai` in that pane; wait (bounded) for the agent to appear, name it `manager-<N>`, and wait for it to settle. If it is `blocked` (e.g. a folder-trust prompt), surface it to the user and wait; do not answer it.
4. **Dispatch** with `herdr agent prompt manager-<N>`, using exactly:
   - Issue: `/goal use /github-issue-pr-manager to complete mergeable PR for issue #<N>`
   - Epic: `/goal use /github-epic-issue-manager to complete mergeable PR for issue #<N>`
   - If `<base>` is not the repo's default branch, append ` targeting branch <base>`. For an epic with a non-default base, also verify the epic PR's base once it exists, since the epic manager defaults to the default branch; correct it by messaging the agent.
5. Record the dispatch and report the branch, workspace, base, and agent name to the user in one line.

If any step fails after the worktree exists, report the failure and the state left behind; do not silently remove the worktree.

## Tracking

State lives in Herdr and GitHub; do not invent a ledger file.

A **dispatch** is a worktree (`herdr worktree list`) whose branch matches `^(issue|epic)-<N>(-|$)` and whose workspace has a tab named `admin-<N>`. Epic-manager sub-issue worktrees use a `#<N>` tab and are therefore not dispatches.

Per dispatch, derive: kind, number, branch, workspace/pane, base (PR `baseRefName`; before a PR exists, the base recorded at dispatch, else ask), PR (open PRs by head branch), and agent state (`herdr agent get manager-<N>`).

The only non-derivable state is *what the manager already told an agent* — the `(headSha, baseSha)` of the last divergence notice per PR. Keep it in the conversation; if lost, the worst outcome is one repeated notice.

## Monitor

Run a cycle after each user message, and on a timer when the harness offers one (for example `/loop`, a scheduled wakeup, or a monitor). Use roughly 3–5 minute spacing and skip the first check for a few minutes after a dispatch; these agents run for a long time. Between cycles, wait for the user. Do not busy-poll.

Each cycle is cheap-first: one `herdr agent list`, one repo-wide `gh pr list --state open --json …`, then per-PR calls only for PRs that are ready. Classify each dispatch:

| Observation | Action |
|---|---|
| Agent `blocked` | Show the last ~30 lines of its pane and ask the user. Never answer it. |
| Agent missing/exited and no open PR | Report; offer to redispatch. Do not auto-restart. |
| No PR yet, or PR is a draft | Keep waiting. The agent handles its own CI, review, and rebase gate. |
| PR closed without merge | Report; leave the worktree. |
| PR merged by someone else | Run "After a merge". |
| PR ready (non-draft) and agent settled | Run the merge gate. |

Tell the user only about changes that need attention or that finished: merged, diverged-notified, blocked, closed, died, or a decision needed.

## Merge gate

A PR is **merge-ready** only if all hold, for the same head SHA:

1. `state` is `OPEN`, `isDraft` is false.
2. The manager agent is `idle`/`done` (not `working`), and the head SHA was identical on two consecutive cycles. This avoids merging mid-rebase.
3. **Not diverged:** the base tip is an ancestor of the PR head (`gh api …/compare/<base>...<head>` → `behind_by == 0`).
4. `mergeable` is `MERGEABLE` and `mergeStateStatus` is `CLEAN` (or `HAS_HOOKS`).
5. `reviewDecision` is not `CHANGES_REQUESTED`.
6. Required checks are green (`gh pr checks <n> --required`; if none are configured, all checks must be green or absent). Pending checks mean wait.
7. For an epic PR: the epic issue has no remaining unchecked (`- [ ]`) sub-issues. Unchecked items usually mean live-execution tasks remain (the epic manager's "Stopping Point A"); do not auto-merge, tell the user, and let them decide.

Other `mergeStateStatus` values:

- `BEHIND` / `DIRTY` → divergence path.
- `BLOCKED` → something required is missing (commonly a human approval, since authors cannot self-approve). Report once and wait; never `--admin`.
- `UNSTABLE` or `UNKNOWN` → recheck next cycle; if it persists, report.

### Merge (serial)

Merge one PR at a time, in the order they became ready. In `confirm` mode, report merge-ready PRs and wait.

```
gh pr merge <n> --<squash|merge|rebase> --match-head-commit <headSha>
```

Pass `--match-head-commit` so a head that moved after the gate fails the merge instead of merging unreviewed code. Do not pass `--delete-branch` (it fails for branches checked out in a worktree); remote branch deletion is left to the repo's own setting. If the merge is rejected, re-run the gate rather than retrying blindly.

### Diverged: tell the agent

If a ready PR fails check 3 or reports `BEHIND`/`DIRTY`, do not merge. Send the agent one notice with `herdr agent prompt manager-<N>`, saying that base `<base>` advanced to `<baseSha7>`, that PR `#<n>` head `<headSha7>` no longer contains it, and to run its post-validation rebase gate (fetch `origin/<base>`, rebase with semantic conflict resolution, `--force-with-lease` push, then a fresh full review/CI/E2E on the new head), keep the PR ready, and not merge. Template in the reference.

Record `(headSha, baseSha)`. Do not resend until the head changes. If the agent goes idle with the same head after a notice, remind it once, then escalate to the user. If the agent is `blocked`, surface that instead of prompting.

## After a merge

1. Verify with `gh pr view <n> --json state,mergedAt,mergeCommit`.
2. Report the PR, merge commit, and whether the issue closed. Closing keywords only auto-close for the default branch, so an issue merged into a non-default base legitimately stays open; report it, don't close it.
3. Immediately re-run the merge gate for the other ready PRs that share the merged base. Each one now fails the divergence check and gets a notice. This is the main consolidation loop.
4. Cleanup is opt-in. Ask the user once whether to remove merged worktrees (or follow a standing "clean up after merge" instruction). Before removing, require the PR merged, the worktree clean (`git -C <path> status --porcelain` empty), and warn that removal discards the retained `planning-`/`implementation-`/`e2e-` tabs kept for provenance. Then `herdr worktree remove --workspace <id> --force --json`.

## Reporting

For a merged PR, report: kind and number, PR URL, merge method, merge commit, base, and whether the issue closed. For a status request, print one table: `#`, kind, base, agent state, PR (number + draft/ready), merge status, last action.

If the user says to stop, stop monitoring and leave all worktrees and agents running; say which are still active.

## Guardrails

- Never merge a PR the manager did not gate in the current cycle for its current head.
- Never merge on the strength of an agent's chat message alone; GitHub state is the source of truth.
- If GitHub and Herdr disagree (PR ready but agent `working`), wait; do not reconcile by acting.
- Prefer doing nothing and asking over a guess that could merge or discard work.
