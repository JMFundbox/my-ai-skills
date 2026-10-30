# Agent instructions

This repository is a workspace for maintaining reusable agentic coding skills.

- Treat `skills/<skill-name>/` as the canonical package. Do not edit a machine-local installed copy as the source of truth.
- Read the target `SKILL.md` and any referenced documentation before changing a workflow. Preserve intended safety gates, user confirmations, tool requirements, and stop conditions.
- Keep each skill self-contained: preserve its `SKILL.md`, `references/`, `agents/`, and any other package assets. Keep internal links relative to the skill directory and verify they still resolve after edits.
- Keep the directory name and the `name` field in `SKILL.md` aligned. Use concise, actionable skill descriptions that explain when the skill should be invoked.
- When adding or renaming a skill, update the inventory and installation instructions in `README.md`.
- Never add secrets, credentials, personal machine configuration, or unrelated files. Do not claim that tool dependencies are installed merely because a skill package is present.
- Review the final diff for accidental deletions or unrelated changes. Validate frontmatter, local references, and the package structure before considering a skill change complete.
