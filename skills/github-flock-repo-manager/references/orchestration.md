# GitHub Flock Repo Manager — Orchestration Reference

Command patterns for `github-flock-repo-manager`. They follow the conventions of `github-flock-issue-pr-manager` and `github-flock-epic-issue-manager` (jq fallbacks for Herdr JSON shapes, bounded agent-startup wait). Treat them as patterns to adapt, not a script to run blindly; the installed `herdr` binary is the authority for syntax (`herdr worktree`, `herdr tab`, `herdr agent`). No `pc-tool flock` state-changing verb appears here: the repo manager never touches a dispatched flock.

## 1. Preflight

```bash
test "${HERDR_ENV:-}" = 1 || { echo "ERROR: Not running inside Herdr (HERDR_ENV != 1)."; exit 1; }
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "ERROR: Not inside a Git repository."; exit 1; }
gh auth status >/dev/null 2>&1 || { echo "ERROR: gh not authenticated."; exit 1; }

REPO_SLUG="$(gh repo view --json nameWithOwner -q .nameWithOwner)"
DEFAULT_BRANCH="$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)"
MANAGER_MODEL="${MANAGER_MODEL:-advanced-adaai}"
MANAGER_WORKSPACE_ID="$HERDR_WORKSPACE_ID"
```

## 2. Naming and duplicate check

```bash
slugify() {
  local s
  s="$(echo "$1" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-' | sed 's/^-//;s/-$//' | cut -c1-30 | sed 's/-$//')"
  echo "${s:-work}"
}

# Returns 0 if issue/epic <n> already has a worktree branch or an open PR branch.
already_dispatched() {
  local n="$1" re="^(issue|epic)-${1}(-|\$)"
  herdr worktree list | jq -e --arg re "$re" '.result.worktrees[]? | select(.branch | test($re))' >/dev/null 2>&1 && return 0
  gh pr list --state open --json headRefName | jq -e --arg re "$re" '.[] | select(.headRefName | test($re))' >/dev/null 2>&1
}
```

## 3. Base resolution

```bash
# Prints the validated base branch, or returns non-zero with a reason on stderr.
resolve_base() {
  local base="${1:-$(git branch --show-current)}"
  [ -n "$base" ] || { echo "Detached HEAD: ask the user for a base branch." >&2; return 1; }
  git fetch origin "$base" --quiet 2>/dev/null || { echo "origin/$base does not exist; a PR cannot target it." >&2; return 1; }
  if git rev-parse --verify -q "refs/heads/$base" >/dev/null &&
     [ "$(git rev-list --count "origin/$base..$base")" -gt 0 ]; then
    echo "Local $base has unpushed commits; push, or confirm using origin/$base." >&2
    return 1
  fi
  echo "$base"
}
```

## 4. Dispatch

### Issue kind classification

Before calling `dispatch_one`, independently inspect the requested GitHub issue's type, labels, body, and native sub-issues (including all pages when needed). Use the GitHub API or `gh` fields available on the host; distinguish an empty sub-issue list from an API/authentication error. Where a particular metadata feature is unsupported, use the available issue data and repository conventions rather than treating unsupported as a positive or a retrieval failure. If required available data cannot be read, report the error and skip the dispatch; never silently treat a failed lookup as an empty list.

Classify as `epic` when the user explicitly describes it as an epic, or the issue has an Epic type, `epic`/`umbrella` label, at least one native sub-issue, a task list with two or more issue references, or an explicit epic/umbrella description covering other issues. Otherwise, after successful inspection, default to `issue`. “Issue #N” alone does not override epic evidence. A specific request to use a named manager or to treat the item as a non-epic overrides automatic classification. Do not ask the user to choose a manager because the kind is ambiguous. Announce the chosen kind in the normal dispatch report.

Examples: `work issue #20` with an epic label or native sub-issue → `epic`; `work #21` with no epic evidence → `issue`; `use /github-flock-issue-pr-manager for #20` → `issue` even if it is an epic.

### Dispatch command

```bash
# usage: dispatch_one <issue|epic> <number> <base>
dispatch_one() {
  local kind="$1" n="$2" base="$3"
  case "$kind" in issue|epic) ;; *) echo "ERROR: invalid dispatch kind: $kind" >&2; return 1 ;; esac
  local title slug branch label skill goal
  title="$(gh issue view "$n" --json title -q .title)"
  slug="$(slugify "$title")"
  branch="${kind}-${n}-${slug}"
  label="${kind}-${n}"

  # 1. Worktree from the fetched base tip, no focus steal.
  local res ws tab pane
  res="$(herdr worktree create \
    --workspace "$MANAGER_WORKSPACE_ID" \
    --base "origin/${base}" \
    --branch "$branch" \
    --label "$label" \
    --no-focus --json)" || { echo "ERROR: worktree create failed"; return 1; }

  ws="$(echo "$res"   | jq -r '.result.workspace.workspace_id // .result.workspace_id')"
  tab="$(echo "$res"  | jq -r '.result.tab.tab_id // .result.tab_id')"
  pane="$(echo "$res" | jq -r '.result.root_pane.pane_id // .result.pane_id')"
  [ -z "$pane" ] || [ "$pane" = null ] && pane="$(herdr pane list --workspace "$ws" --json | jq -r '.result.panes[0].pane_id')"
  [ -z "$tab"  ] || [ "$tab"  = null ] && tab="$(herdr tab list --workspace "$ws" --json | jq -r '.result.tabs[0].tab_id')"

  # 2. Drop the upstream inherited from origin/<base> and verify the starting point.
  local wt_path
  wt_path="$(herdr worktree list | jq -r --arg b "$branch" '.result.worktrees[] | select(.branch == $b) | .path')"
  git -C "$wt_path" branch --unset-upstream "$branch" 2>/dev/null || true
  [ "$(git -C "$wt_path" rev-parse HEAD)" = "$(git rev-parse "origin/${base}")" ] || {
    echo "ERROR: worktree HEAD is not origin/${base}; leaving $wt_path for inspection."; return 1; }

  # 3. Name tab and pane.
  herdr tab rename  "$tab"  "admin-${n}"
  herdr pane rename "$pane" "manager-${n}"

  # 4. Start the manager agent (bounded wait for it to occupy the pane).
  sleep 2
  herdr pane run "$pane" "proclaude --model ${MANAGER_MODEL}"
  local found=0 i
  for i in $(seq 1 15); do
    herdr agent list | jq -e --arg p "$pane" '.result.agents[]? | select(.pane_id == $p)' >/dev/null 2>&1 && { found=1; break; }
    sleep 2
  done
  [ "$found" -eq 1 ] || { echo "ERROR: agent failed to start in $pane"; herdr pane process-info "$pane"; return 1; }
  herdr agent rename "$pane" "manager-${n}"

  # Wait until it settles; a blocked agent (e.g. folder-trust prompt) is the user's call.
  herdr agent wait "manager-${n}" --timeout 90000 >/dev/null 2>&1
  local state
  state="$(herdr agent get "manager-${n}" | jq -r '.result.agent.status // .result.agent.state // "unknown"')"
  if [ "$state" = blocked ]; then
    echo "Agent manager-${n} is blocked; surface the prompt to the user:"
    herdr agent read "manager-${n}" | tail -n 30
    return 2
  fi

  # 5. Dispatch with the exact goal text.
  if [ "$kind" = epic ]; then skill=github-flock-epic-issue-manager; else skill=github-flock-issue-pr-manager; fi
  goal="/goal use /${skill} to complete mergeable PR for issue #${n}"
  [ "$base" != "$DEFAULT_BRANCH" ] && goal="${goal} targeting branch ${base}"
  herdr agent prompt "manager-${n}" "$goal" || return 1

  echo "Dispatched ${kind} #${n}: branch=${branch} workspace=${ws} base=${base} agent=manager-${n}"
}
```

Notes:

- The `agent status` field name is read defensively (`.status // .state`); confirm against `herdr agent get` output on the installed version.
- `herdr agent prompt` rejects a blocked agent with `agent_blocked` and a non-reactive one with `agent_prompt_stalled`. Treat either as "report to the user", not "retry".

## 5. Discover dispatches

The lead renames its tab (`lead` → `#<N>`, later `epic-<N>`), so the tab name is not an identifier here. Use the workspace label set at creation, or a slugged branch. Epic-manager sub-issue worktrees have label `#<N>` and the slugless branch `issue-<N>`, so neither test matches them.

```bash
list_dispatches() {
  herdr worktree list | jq -c '.result.worktrees[]?
      | select(.open_workspace_id != null)
      | select(((.label // "") | test("^(issue|epic)-[0-9]+$"))
               or ((.branch // "") | test("^(issue|epic)-[0-9]+-.+")))
      | . as $w
      | (($w.label // "") | capture("^(?<k>issue|epic)-(?<n>[0-9]+)$")?
         // ($w.branch | capture("^(?<k>issue|epic)-(?<n>[0-9]+)-"))) as $id
      | {kind: $id.k, number: ($id.n | tonumber), branch: $w.branch, ws: $w.open_workspace_id, path: $w.path}'
}

# Manager pane of a dispatch workspace (prompt by pane ID, not by name).
manager_pane() {  # <workspace-id> <number>
  herdr pane list --workspace "$1" --json |
    jq -r --arg l "manager-$2" '.result.panes[]? | select((.label // .name) == $l) | .pane_id' | head -n1
}

# Cosmetic: restore the requested tab name after the lead renames it.
reassert_admin_tab() {  # <workspace-id> <number>
  local tab
  tab="$(herdr tab list --workspace "$1" --json | jq -r '.result.tabs[0].tab_id // empty')"
  [ -n "$tab" ] && herdr tab rename "$tab" "admin-$2" >/dev/null 2>&1 || true
}
```

Verify the pane label field (`.label` vs `.name`) against `herdr pane list` on the installed version.

One repo-wide PR snapshot per monitoring cycle:

```bash
gh pr list --state open --json number,headRefName,baseRefName,headRefOid,isDraft,mergeable,mergeStateStatus,reviewDecision,url
```

PRs for dispatches that are no longer open: `gh pr list --state merged --head "<branch>" --json number,mergedAt,mergeCommit`.

### Background monitor shell (mandatory when idle with no timer)

When live dispatches exist and no timer (`/loop`, scheduled wakeup, monitor) drives cycles, the manager must keep one ordinary Herdr pane labelled `monitor` running a read-only watch loop. It observes only: `herdr worktree list`, `herdr agent list`, and repo-wide `gh` PR state. It must never merge, prompt leads, edit/commit/push code, run tests in a dispatch worktree, close worktrees, answer permission prompts, or run any `pc-tool flock` state-changing verb. A read-only `flock status` look is allowed only as corroboration, never as a gate. All decisions stay in the manager turn, which re-reads Herdr/GitHub itself before acting.

```bash
# Prints the monitor pane id in the manager workspace, or nothing.
monitor_pane() {
  herdr pane list --workspace "$MANAGER_WORKSPACE_ID" --json |
    jq -r '.result.panes[]? | select((.label // .name) == "monitor") | .pane_id' | head -n1
}

# Starts the read-only watch loop if it is missing or dead. Prints the pane id.
# usage: ensure_monitor_shell <log-file>
ensure_monitor_shell() {
  local log="$1" pane tab root
  pane="$(monitor_pane)"
  if [ -n "$pane" ] && herdr pane process-info "$pane" >/dev/null 2>&1; then
    echo "$pane"
    return 0
  fi
  tab="$(herdr tab create --workspace "$MANAGER_WORKSPACE_ID" --cwd "$PWD" --label monitor --no-focus --json | jq -r '.result.tab.tab_id // .result.tab_id')"
  root="$(herdr tab list --workspace "$MANAGER_WORKSPACE_ID" --json | jq -r --arg t "$tab" '.result.tabs[]? | select(.tab_id == $t) | .root_pane.pane_id // empty')"
  pane="${root:-$(herdr pane list --workspace "$MANAGER_WORKSPACE_ID" --json | jq -r '.result.panes[0].pane_id')}"
  herdr pane rename "$pane" "monitor"
  sleep 2
  herdr pane run "$pane" "LOG='$log'; while true; do echo \"=== \$(date -u +%FT%TZ) ===\" >>\"\$LOG\" 2>&1; herdr worktree list --json 2>&1 | jq -c '.result.worktrees[]? | {branch, path, open_workspace_id}' >>\"\$LOG\" 2>&1; herdr agent list --json 2>&1 | jq -c '.result.agents[]? | {name, pane_id}' >>\"\$LOG\" 2>&1; gh pr list --state open --json number,headRefName,headRefOid,isDraft,mergeable,mergeStateStatus >>\"\$LOG\" 2>&1; sleep 240; done"
  echo "$pane"
}

# Liveness check each manager cycle; recreate when dead.
# pane="$(monitor_pane)"; herdr pane process-info "$pane" >/dev/null 2>&1 || pane="$(ensure_monitor_shell "$MONITOR_LOG")"

# Stop only the monitor you created when the user says to stop monitoring:
# herdr pane close "$(monitor_pane)"
```

Set `MONITOR_LOG` once per session (for example `/tmp/repo-manager-monitor-<slug>.log`), reuse it across cycles, and read its tail instead of re-polling panes in a tight loop. Closing the monitor never closes dispatch worktrees, tabs, leads, or flock members.

## 6. Merge gate helpers

```bash
# Number of base commits missing from the PR head. >0 means the target branch diverged.
behind_by() { gh api "repos/${REPO_SLUG}/compare/${1}...${2}" --jq .behind_by; }   # base, head-sha

# Required checks only; fall back to all checks when none are required.
checks_state() {  # prints: pass | pending | fail
  local out
  out="$(gh pr checks "$1" --required --json bucket 2>/dev/null)"
  [ -z "$out" ] || [ "$out" = "[]" ] && out="$(gh pr checks "$1" --json bucket 2>/dev/null || echo '[]')"
  echo "$out" | jq -r 'if any(.[]; .bucket=="fail" or .bucket=="cancel") then "fail"
                        elif any(.[]; .bucket=="pending") then "pending" else "pass" end'
}

# Completion handshake: the lead's recent output must name BOTH the PR and the CURRENT
# head SHA (7-char prefix). Idle alone is not "finished" for a Flock lead. This is a
# heuristic gate, always combined with idle/done + a head SHA stable across cycles.
lead_reported_head() {  # <manager-pane-id> <pr-number> <head-sha>
  local out short="${3:0:7}"
  out="$(herdr agent read "$1" 2>/dev/null | tail -n 60)"
  echo "$out" | grep -q "$short" && echo "$out" | grep -qE "#${2}([^0-9]|\$)|/pull/${2}([^0-9]|\$)"
}

# Epic completeness: any remaining unchecked task-list item.
epic_has_open_items() { gh issue view "$1" --json body -q .body | grep -qE '^\s*- \[ \]'; }
```

Verify `gh pr checks --json bucket` is supported by the installed `gh`; if not, parse `gh pr checks` text output instead.

### Merge method

```bash
repo_merge_methods() {
  gh repo view --json squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed |
  jq -r '[ (if .squashMergeAllowed then "squash" else empty end),
           (if .mergeCommitAllowed then "merge"  else empty end),
           (if .rebaseMergeAllowed then "rebase" else empty end) ] | .[]'
}
# user-specified > exactly one allowed > ask once and cache for the session.
# Reject a chosen method that repo_merge_methods does not list.
```

### Merge

```bash
merge_pr() {  # <pr> <method> <head-sha>
  gh pr merge "$1" "--$2" --match-head-commit "$3"
}
```

No `--delete-branch` (fails when the branch is checked out in a worktree), no `--admin`, no `--auto`.

## 7. Prompt templates

Send each with `herdr agent prompt "$(manager_pane <ws> <N>)" "<text>"` (pane ID, not agent name). If it fails with `agent_blocked`, surface the blocked UI to the user instead of retrying.

**Divergence notice:**

```
Base branch diverged: PR #<PR> targets <BASE>, which advanced to <BASE_SHA7> (<reason: PR #Q merged | external push>). Your head <HEAD_SHA7> does not contain it.

Run your post-validation origin/<BASE> rebase gate now: fetch origin/<BASE>, rebase your branch onto it (resolve conflicts semantically, preserve both sides' intended behavior), push with --force-with-lease, then restart your PR stage with your flock reviewer and E2E members for a fresh full formal review, CI, and E2E on the new head. Keep the PR ready for review and do NOT merge it; the repo manager merges once it is mergeable again. When the new head has passed, restate your final report naming PR #<PR> and the new head SHA.
```

**Reminder** (once, if the lead went idle on the same head with no restated report): the same text prefixed with `Reminder:` and a request to state what is blocking.

**Status ping** (once per head SHA, only when the lead is idle, the head is stable for two further cycles, and no final report names the head):

```
Status check from the repo manager for PR #<PR> (head <HEAD_SHA7>). If your PR stage is complete and every gate in your completion criteria has passed for this head, restate your final report now, naming PR #<PR> and head <HEAD_SHA7>. Otherwise reply in one line with what remains. Do not merge.
```

## 8. Cleanup (opt-in)

```bash
cleanup_dispatch() {  # <workspace-id> <worktree-path> <pr-number>
  [ "$(gh pr view "$3" --json state -q .state)" = MERGED ] || { echo "PR not merged; not removing."; return 1; }
  [ -z "$(git -C "$2" status --porcelain)" ] || { echo "Worktree dirty; not removing."; return 1; }
  herdr worktree remove --workspace "$1" --force --json
}
```

Removal kills the lead's flock members and discards the retained `planning-<N>`, `implementation-<N>`, and `e2e-<N>` tabs of the worktree. Also require the lead `idle`/`done` and, preferably, its final report delivered (it runs `flock cleanup` and `flock close` as its last stage). Warn the user before the first removal in a session.

## 9. Topology produced

```text
Primary checkout (repo manager, any branch, never modified)
 └─ Herdr worktree workspace per dispatch (base: origin/<base>)
     ├─ branch:  issue-<N>-<slug>  |  epic-<N>-<slug>
     ├─ tab:     admin-<N>          (initial tab; the lead renames it to lead → #<N> / epic-<N>,
     │                               the repo manager cosmetically re-applies admin-<N>)
     └─ pane:    manager-<N>        (proclaude --model advanced-adaai; becomes the flock Lead)
                   └─ /goal use /github-flock-issue-pr-manager | /github-flock-epic-issue-manager …
                      (own flock; spawns planning-<N>, implementation-<N>, e2e-<N>, pr-reviewer
                       in this workspace and adopts them as m1…m6; an epic manager also spawns
                       `#<M>` sub-issue worktrees, each with its own flock)
```
