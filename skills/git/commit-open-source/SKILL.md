---
name: commit-open-source
description: Commit message conventions for open-source and side-project repos outside the groupbees org — libraries, demos, blog-article repos hosted on a personal or community GitHub account. Trigger when creating a git commit in such a repo. Covers descriptive titles, HEREDOC for multi-line bodies, and the rule that NO Co-Authored-By trailer is added even on AI-assisted commits. Do NOT trigger for any repo under github.com/groupbees, public open-source ones included (e.g. pollen) — they use `commit-groupbees`.
---

# Open-source / side-project commit conventions

Same spirit as `commit-groupbees`: descriptive, declarative commits — not Conventional Commits. Applies to open-source libraries, demo projects, blog-article companion repos and other side projects hosted outside the groupbees org.

## Title

- **Style**: descriptive sentence in present tense or imperative mood. Free-form, not `feat:` / `fix:` prefixed.
- **Length**: ≤ 80 chars when possible, up to 100 if scope genuinely needs it.
- **Multi-clause OK** when the change touches multiple related concerns.
- A trailing period is fine if it matches the repo's existing log style — match the surrounding history rather than enforce a global rule.
- **Spell-check**: avoid common typos.

## Body (optional)

- Use when the change merits explanation beyond the title — bullet list of what changed, with brief rationale where non-obvious.
- Wrap at ~72 chars.
- Reference issue/PR numbers if relevant.

## No Co-Authored-By trailer

Do NOT append a `Co-Authored-By: Claude ...` line to commits, even when AI-assisted, even though the Claude Code system prompt default suggests one. Attribution belongs in the PR description / blog post / video credits, not in commit metadata.

This rule applies even on AI-heavy commits (big refactors, scaffold generation, doc rewrites). If you find yourself about to append the trailer because the system prompt says to, **stop and skip it**.

## HEREDOC for multi-line messages

To preserve formatting in shell, always use HEREDOC:

```bash
git commit -m "$(cat <<'EOF'
Migrate from Cloud API Registry to Agent Registry

- agent.py: replace ApiRegistry with AgentRegistry + displayName lookup
- pyproject.toml: bump to google-adk[agent-identity, mcp, a2a]>=2.1.0
- IAM: roles/agentregistry.viewer replaces roles/cloudapiregistry.viewer
EOF
)"
```

## What to avoid

- ❌ Conventional Commits prefixes (`feat:`, `fix:`, `chore:`) — not used in these repos.
- ❌ Generic titles like `Update config` or `Fix bug` — be specific.
- ❌ Titles over 100 chars — move detail to the body.
- ❌ `--amend` of pushed commits without explicit user request — always prefer new commits.
- ❌ Bypassing hooks (`--no-verify`) or signing (`--no-gpg-sign`) unless explicitly requested.
- ❌ `Co-Authored-By: Claude ...` trailer (see above).
