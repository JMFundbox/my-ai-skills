---
name: github-flock-repo-manager
description: Flock variant of github-repo-manager. Run a long-lived repo-level manager that waits for the user to ask for GitHub issues or epics to be worked, then creates a Herdr worktree per issue/epic, launches a ProClaude manager agent in it with /goal (/github-flock-issue-pr-manager for issues, /github-flock-epic-issue-manager for epics), monitors those agents and their PRs, and consolidates merging — merging each mergeable PR serially, or telling the agent that its PR's target branch has diverged. Use when the user wants Flock-coordinated issue/epic managers dispatched, tracked, and merged from one place; not for working a single issue yourself.
---

# GitHub Flock Repo Manager

A repo-level dispatcher and merge coordinator for the Flock-coordinated managers. It does not implement anything. It turns "work issue #12 and epic #20" into isolated Herdr worktrees with a Flock-powered manager agent in each, watches them, and merges their PRs one at a time when they are genuinely mergeable.

This workflow requires Herdr. Read the user-level `herdr` skill, then read [references/orchestration.md](references/orchestration.md) for the command patterns before creating worktrees or prompting agents. If `HERDR_ENV` is not `1`, say so and stop. Read only the reference sections needed for the current step.

This skill dispatches [`github-flock-issue-pr-manager`](../github-flock-issue-pr-manager/SKILL.md) and [`github-flock-epic-issue-manager`](../github-flock-epic-issue-manager/SKILL.md). Those skills, and Flock (`pc-tool flock`), must be available to the harness the dispatched agents run in; this package installs neither. The non-Flock equivalent of this skill is `github-repo-manager`.

## What differs from the non-Flock repo manager

Each dispatched manager becomes a Flock Lead inside its own worktree and runs its own flock (planning, implementation, review, and E2E members adopted as `m1`…`m6`). Three consequences shape this skill:

1. **The `admin-<N>` tab name does not survive.** `flock start` renames the lead's tab to `lead`, and the Flock issue manager then renames it to `#<N>` (the Flock epic manager later renames it to `epic-<N>`). Dispatch discovery therefore never depends on the tab name, and the manager re-applies `admin-<N>` on each monitoring cycle as a cosmetic step.
2. **"Agent idle" is not "agent finished".** A Flock lead ends its turn and waits for a background `flock wait` wake, so Herdr shows it `idle` while members are still reviewing or testing. The merge gate adds a completion handshake (below).
3. **Each worktree has its own flock.** The repo manager never starts, joins, sends to, cleans up, or closes any flock. It talks to the lead only with `herdr agent prompt`.

## Ownership and hard limits

The repo manager owns: intake, worktree/tab/pane/agent setup, dispatch prompts, monitoring, merge decisions, divergence notices, and user reporting.

It must not:

- Edit, commit, push, rebase, or run tests in any dispatched worktree, or `cd` into one to do work. Interact with dispatches only through `herdr` commands and `gh`.
- Check out, commit to, pull into, or otherwise alter the checkout it runs in. `git fetch` is allowed. The base branch is only ever changed by merging a PR on GitHub.
- Merge with `--admin`, bypass branch protection or a merge queue, or force a merge past a failing required check or unresolved review.
- Answer a `blocked` agent's permission or trust prompt on the user's behalf. Surface the prompt and ask.
- Close issues, delete branches, or remove worktrees except as stated in "After a merge".
- Add scope. Dispatch exactly what was asked, with the exact goal text below.
- Run any `pc-tool flock` verb that changes state (`start`, `adopt`, `send`, `cleanup`, `close`, `tick`) in or for a dispatched worktree, or inject into a lead's members. The lead owns its flock.

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
2. **Choose the manager independently:** inspect the issue's type, labels, body, and native sub-issues before dispatch. Treat it as an epic if it has an Epic issue type, an `epic`/`umbrella` label, at least one native sub-issue, or a task list of two or more issue references; also recognize an issue explicitly described by the user or in GitHub as an epic or umbrella for other issues. Route it to `github-flock-epic-issue-manager` without asking the user to choose. A request such as “work issue #20” does not override epic evidence: GitHub epics are issues too. If inspection succeeds and finds no epic evidence, route it to `github-flock-issue-pr-manager` by default without asking. Honor a specific instruction to use a named manager or to treat the item as a non-epic even if it conflicts with the evidence; merely calling it an issue is not such an override. If GitHub data needed for classification cannot be retrieved, report the access error and skip that number rather than guess. See the [classification protocol](references/orchestration.md#issue-kind-classification).
3. Not already dispatched: no worktree branch matching `^(issue|epic)-<N>(-|$)` and no open PR from such a branch. This also stops it from duplicating a sub-issue that an epic manager is already working (those use branch `issue-<N>`).
4. If the base branch is an epic branch being run by an epic manager, skip and say so; that epic manager merges its own sub-issue PRs.
5. Base is valid: `git fetch origin <base>` succeeds and local `<base>`, if it exists, has no commits missing from `origin/<base>`. If it does, stop and ask the user to push or confirm using the origin tip.
6. Soft cap: each Flock dispatch runs a lead plus up to six members, so if more than 3 dispatches are already active, ask before adding more.

**Steps**

1. **Create the worktree** from `origin/<base>` with an appropriate name:
   - Branch: `issue-<N>-<short-kebab-title>` or `epic-<N>-<short-kebab-title>` (slug ≤ 30 chars). Epics use the exact name `github-flock-epic-issue-manager` would pick, so it won't rename. If the title yields an empty slug (for example non-ASCII only), use `work`.
   - Worktree label: `issue-<N>` / `epic-<N>`.
   - Do not steal focus. Afterwards unset any upstream inherited from `origin/<base>` (so the child's first `git push -u origin <branch>` works) and verify the worktree's `HEAD` equals the fetched `origin/<base>`.
2. **Name the topology:** initial tab → `admin-<N>`; initial pane and its agent → `manager-<N>`. The lead will rename the tab (see above); that is expected.
3. **Start the agent:** run `proclaude --model advanced-adaai` in that pane; wait (bounded) for the agent to appear, name it `manager-<N>`, and wait for it to settle. If it is `blocked` (e.g. a folder-trust prompt), surface it to the user and wait; do not answer it.
4. **Dispatch** with `herdr agent prompt manager-<N>`, using exactly:
   - Issue: `/goal use /github-flock-issue-pr-manager to complete mergeable PR for issue #<N>`
   - Epic: `/goal use /github-flock-epic-issue-manager to complete mergeable PR for issue #<N>`
   - If `<base>` is not the repo's default branch, append ` targeting branch <base>`. For an epic with a non-default base, also verify the epic PR's base once it exists, since the epic manager defaults to the default branch; correct it by messaging the agent.
   - The lead creates its own tabs with `--workspace "$HERDR_WORKSPACE_ID"` from inside the new worktree, so no isolation clause is added to the goal text.
5. Record the dispatch and report the branch, workspace, base, and agent name to the user in one line.

If any step fails after the worktree exists, report the failure and the state left behind; do not silently remove the worktree.

## Tracking

State lives in Herdr and GitHub; do not invent a ledger file.

A **dispatch** is a worktree (`herdr worktree list`) whose workspace **label** is `issue-<N>` or `epic-<N>` (set at creation) or whose branch matches `^(issue|epic)-<N>-.+` (slug required). Epic-manager sub-issue worktrees use label `#<N>` and the slugless branch `issue-<N>`, so they are not dispatches. Do not use the tab name: the lead renames it.

Per dispatch, derive: kind, number, branch (the lead may rename it, but keeps the `<kind>-<N>-` prefix), workspace, manager pane (the pane in that workspace labelled `manager-<N>`; prompt by pane ID), base (PR `baseRefName`; before a PR exists, the base recorded at dispatch, else ask), PR (open PRs by head branch), and agent state (`herdr agent get` on that pane).

The only non-derivable state is *what the manager already told an agent* — the `(headSha, baseSha)` of the last divergence notice per PR. Keep it in the conversation; if lost, the worst outcome is one repeated notice.

## Monitor

Run a cycle after each user message, and on a timer when the harness offers one (for example `/loop`, a scheduled wakeup, or a monitor). Use roughly 3–5 minute spacing and skip the first check for a few minutes after a dispatch; these agents run for a long time. Between cycles, wait for the user. Do not busy-poll.

Each cycle is cheap-first: one `herdr agent list`, one repo-wide `gh pr list --state open --json …`, then per-PR calls only for PRs that are ready. Re-apply the `admin-<N>` tab name on each dispatch's first tab if the lead renamed it (cosmetic; skip if the tab was closed). Classify each dispatch:

| Observation | Action |
|---|---|
| Agent `blocked` | Show the last ~30 lines of its pane and ask the user. Never answer it. |
| Agent missing/exited and no open PR | Report; offer to redispatch. Do not auto-restart. |
| No PR yet, or PR is a draft | Keep waiting. The lead and its flock handle CI, review, and the rebase gate. Do not treat an idle lead as stuck. |
| PR closed without merge | Report; leave the worktree. |
| PR merged by someone else | Run "After a merge". |
| PR ready (non-draft) and agent settled | Run the merge gate. |

Tell the user only about changes that need attention or that finished: merged, diverged-notified, blocked, closed, died, or a decision needed.

## Merge gate

A PR is **merge-ready** only if all hold, for the same head SHA:

1. `state` is `OPEN`, `isDraft` is false.
2. **Lead finished (completion handshake).** All of:
   - the lead is `idle`/`done` (not `working` or `blocked`);
   - the head SHA was identical on two consecutive cycles;
   - the tail of the lead's pane (`herdr agent read`, last ~60 lines) contains its final completion report naming the PR and the **current head SHA** (a 7+ character prefix match). The Flock issue manager's Stage 5 and the Flock epic manager's stopping-point reports both state the final commit SHA.

   An idle lead alone is not enough: it waits on a background `flock wait` while members review or test, and after a rebase the PR stays ready while review and CI rerun. A report naming an older SHA means the PR stage was restarted; wait. If everything else holds for two further cycles but no matching report appears, send the lead one status ping (template in the reference) per head SHA, then re-evaluate. Optionally, a read-only `pc-tool flock status` run for that worktree can corroborate (session closed or all members idle), but it never replaces the handshake.
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

If a ready PR fails check 3 or reports `BEHIND`/`DIRTY`, do not merge. Send the lead one notice with `herdr agent prompt` on its manager pane, saying that base `<base>` advanced to `<baseSha7>`, that PR `#<n>` head `<headSha7>` no longer contains it, and to run its post-validation rebase gate (fetch `origin/<base>`, rebase with semantic conflict resolution, `--force-with-lease` push, then restart its PR stage with its flock reviewers and E2E members for a fresh full review/CI/E2E on the new head), keep the PR ready, not merge, and restate its final report with the new head SHA when done. Template in the reference.

Record `(headSha, baseSha)`. Do not resend until the head changes. If the lead goes idle with the same head and no restated report after a notice, remind it once, then escalate to the user. If the lead is `blocked`, surface that instead of prompting. A lead that is `working` already has the PR stage in hand; wait before prompting.

## After a merge

1. Verify with `gh pr view <n> --json state,mergedAt,mergeCommit`.
2. Report the PR, merge commit, and whether the issue closed. Closing keywords only auto-close for the default branch, so an issue merged into a non-default base legitimately stays open; report it, don't close it.
3. Immediately re-run the merge gate for the other ready PRs that share the merged base. Each one now fails the divergence check and gets a notice. This is the main consolidation loop.
4. Cleanup is opt-in. Ask the user once whether to remove merged worktrees (or follow a standing "clean up after merge" instruction). Before removing, require the PR merged, the lead `idle`/`done`, the worktree clean (`git -C <path> status --porcelain` empty), and warn that removal kills the lead's flock members and discards the retained `planning-`/`implementation-`/`e2e-` tabs kept for provenance. Prefer waiting until the lead has run its own `flock cleanup`/`close` (its final report was delivered). Then `herdr worktree remove --workspace <id> --force --json`.

## Reporting

For a merged PR, report: kind and number, PR URL, merge method, merge commit, base, and whether the issue closed. For a status request, print one table: `#`, kind, base, agent state, PR (number + draft/ready), merge status, last action.

If the user says to stop, stop monitoring and leave all worktrees and agents running; say which are still active.

## Guardrails

- Never merge a PR the manager did not gate in the current cycle for its current head.
- Never merge on the strength of an agent's chat message alone; GitHub state is the source of truth.
- If GitHub and Herdr disagree (PR ready but agent `working`), wait; do not reconcile by acting.
- Prefer doing nothing and asking over a guess that could merge or discard work.
