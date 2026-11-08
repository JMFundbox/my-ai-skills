# My AI Skills

A personal collection and working space for agentic coding skills. Each package is kept self-contained so it can be installed into a supported agent's skills directory or edited here and synced to other machines.

## Skills

- `github-issue-pr-manager` — orchestrates a GitHub issue through implementation, review, PR publication, and CI.
- `github-epic-issue-manager` — coordinates implementation across an epic and its sub-issues.
- `github-flock-issue-pr-manager` — Flock-coordinated issue-to-PR workflow.
- `github-flock-epic-issue-manager` — Flock-coordinated epic workflow.

Skill packages live under [`skills/`](skills/). Keep package support files such as `references/` and `agents/` alongside each `SKILL.md`.

## Install on another machine

Both Claude Code (`proclaude`) and Codex (`procodex`) discover user-level skills in a `skills/` directory under their config folder: `~/.claude/skills/` and `~/.codex/skills/` respectively. Each package is self-contained, so installing a skill means placing its directory under the target harness's skills directory.

### Claude Code (proclaude)

The following installs the skills for your Claude Code user account on macOS or Linux. It uses symlinks, so pulling updates in the clone updates the installed skills too.

```sh
mkdir -p "$HOME/src"
git clone https://github.com/JMFundbox/my-ai-skills.git "$HOME/src/my-ai-skills"
mkdir -p "$HOME/.claude/skills"
repo="$HOME/src/my-ai-skills"
for skill in \
  github-issue-pr-manager \
  github-epic-issue-manager \
  github-flock-issue-pr-manager \
  github-flock-epic-issue-manager
do
  ln -s "$repo/skills/$skill" "$HOME/.claude/skills/$skill"
done
```

If you already have a skill with one of these names in `~/.claude/skills`, back it up or move it out of the way before linking; the commands intentionally do not overwrite existing entries.

### Codex (procodex)

The equivalent install for Codex user-level skills targets `~/.codex/skills/`:

```sh
mkdir -p "$HOME/src"
git clone https://github.com/JMFundbox/my-ai-skills.git "$HOME/src/my-ai-skills"
mkdir -p "$HOME/.codex/skills"
repo="$HOME/src/my-ai-skills"
for skill in \
  github-issue-pr-manager \
  github-epic-issue-manager \
  github-flock-issue-pr-manager \
  github-flock-epic-issue-manager
do
  ln -s "$repo/skills/$skill" "$HOME/.codex/skills/$skill"
done
```

Move any existing entry with the same name out of the way first; these commands also do not overwrite existing entries. To install both harnesses at once, repeat the loop with each skills directory.

### Windows

Symlinks require Developer Mode or an elevated shell. On Windows, copy the four directories from `skills/` into `%USERPROFILE%\.claude\skills\` and `%USERPROFILE%\.codex\skills\` instead:

```sh
repo="/path/to/my-ai-skills"
for skill in \
  github-issue-pr-manager \
  github-epic-issue-manager \
  github-flock-issue-pr-manager \
  github-flock-epic-issue-manager
do
  for target in "$HOME/.claude/skills" "$HOME/.codex/skills"; do
    mkdir -p "$target"
    cp -R "$repo/skills/$skill" "$target/$skill"
  done
done
```

Copy-based installs are snapshots: re-copy the directories after pulling to pick up updates.

To update a symlink-based install later:

```sh
git -C "$HOME/src/my-ai-skills" pull --ff-only
```

Restart or reload Claude Code after installing so it discovers the skills. These packages orchestrate other tools (including Herdr and, for the Flock variants, Flock); installing the skill files does not install or authenticate those dependencies. See each skill's `SKILL.md` and `references/` for its requirements.

## Develop and improve skills

- Edit the canonical package under `skills/<skill-name>/`; avoid maintaining separate copies in this repository.
- Keep each package self-contained and preserve relative references, agent metadata, and supporting files when changing it.
- When adding a skill, add its package under `skills/` and update the list and install commands in this README.
- Before sharing a change, verify that the `name` in `SKILL.md` matches its directory and that every referenced local file exists.
- Do not commit credentials, machine-specific configuration, or unrelated workspace state.

See [`AGENTS.md`](AGENTS.md) for instructions for coding agents working in this repository.
