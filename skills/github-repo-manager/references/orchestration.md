# GitHub Repo Manager — Orchestration Reference

Command patterns for `github-repo-manager`. They follow the conventions of `github-issue-pr-manager` and `github-epic-issue-manager` (jq fallbacks for Herdr JSON shapes, bounded agent-startup wait). Treat them as patterns to adapt, not a script to run blindly; the installed `herdr` binary is the authority for syntax (`herdr worktree`, `herdr tab`, `herdr agent`).

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

Examples: `work issue #20` with an epic label or native sub-issue → `epic`; `work #21` with no epic evidence → `issue`; `use /github-issue-pr-manager for #20` → `issue` even if it is an epic.

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
  if [ "$kind" = epic ]; then skill=github-epic-issue-manager; else skill=github-issue-pr-manager; fi
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

```bash
# Dispatch = branch matches issue-N/epic-N AND workspace has an `admin-N` tab.
list_dispatches() {
  herdr worktree list | jq -c '.result.worktrees[]?
      | select(.branch | test("^(issue|epic)-[0-9]+(-|$)"))
      | select(.open_workspace_id != null)
      | {branch, ws: .open_workspace_id, path}' |
  while read -r wt; do
    ws="$(echo "$wt" | jq -r .ws)"
    n="$(echo "$wt" | jq -r '.branch | capture("^(issue|epic)-(?<n>[0-9]+)").n')"
    if herdr tab list --workspace "$ws" --json | jq -e --arg t "admin-$n" '.result.tabs[]? | select((.label // .name) == $t)' >/dev/null 2>&1; then
      echo "$wt" | jq -c --arg n "$n" '. + {number: ($n | tonumber), kind: (.branch | split("-")[0])}'
    fi
  done
}
```

One repo-wide PR snapshot per monitoring cycle:

```bash
gh pr list --state open --json number,headRefName,baseRefName,headRefOid,isDraft,mergeable,mergeStateStatus,reviewDecision,url
```

PRs for dispatches that are no longer open: `gh pr list --state merged --head "<branch>" --json number,mergedAt,mergeCommit`.

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

## 7. Divergence notice template

Send with `herdr agent prompt "manager-<N>" "<text>"`:

```
Base branch diverged: PR #<PR> targets <BASE>, which advanced to <BASE_SHA7> (<reason: PR #Q merged | external push>). Your head <HEAD_SHA7> does not contain it.

Run your post-validation origin/<BASE> rebase gate now: fetch origin/<BASE>, rebase your branch onto it (resolve conflicts semantically, preserve both sides' intended behavior), push with --force-with-lease, then redo a fresh full formal review, CI, and E2E on the new head. Keep the PR ready for review and do NOT merge it; the repo manager merges once it is mergeable again. Report when the new head has passed.
```

Reminder (once, if the agent went idle on the same head): same text prefixed with `Reminder:` and a request to state what is blocking.

## 8. Post-merge sync and cleanup

Run after every merged PR (also when someone else merged it): sync the base branch first, then remove the merged dispatch's worktree. A standing "keep merged worktrees" instruction skips the removal; a dirty worktree is never removed without the user.

```bash
# Fast-forward the manager's checkout to the merged base. Never rebase/merge/reset.
sync_base() {  # <base>
  git fetch origin "$1" --quiet || return 1
  if [ "$(git branch --show-current)" = "$1" ]; then
    git pull --ff-only origin "$1"
  else
    git fetch origin "$1:$1" 2>/dev/null \
      || echo "NOTE: local $1 left alone (cannot fast-forward, or checked out in another worktree)."
  fi
}

cleanup_dispatch() {  # <workspace-id> <worktree-path> <pr-number>
  [ "$(gh pr view "$3" --json state -q .state)" = MERGED ] || { echo "PR not merged; not removing."; return 1; }
  [ -z "$(git -C "$2" status --porcelain)" ] || { echo "Worktree dirty; not removing."; return 1; }
  herdr worktree remove --workspace "$1" --force --json
}
```

Removal discards the retained `planning-<N>`, `implementation-<N>`, and `e2e-<N>` tabs of the worktree and ends `manager-<N>`; say so in the merge report.

## 9. Topology produced

```text
Primary checkout (repo manager, any branch; only changed by the post-merge base sync)
 └─ Herdr worktree workspace per dispatch (base: origin/<base>)
     ├─ branch:  issue-<N>-<slug>  |  epic-<N>-<slug>
     ├─ tab:     admin-<N>          (initial tab)
     └─ pane:    manager-<N>        (proclaude --model advanced-adaai)
                   └─ /goal use /github-issue-pr-manager | /github-epic-issue-manager …
                      (spawns planning-<N>, implementation-<N>, e2e-<N>, pr-reviewer in this workspace;
                       an epic manager also spawns `#<M>` sub-issue worktrees)
```
