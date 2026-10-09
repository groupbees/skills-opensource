# 🐝 skills-opensource

> Conventions for open-source projects — commits, release tags and registry publishing — the `skills-opensource` module of the GroupBees skills catalog.

## 🎯 Purpose

This repo is the source of truth for its skills — **one module** alongside
`skills-core` (transverse) and the others. Each skill is a standard **Agent
Skill** (`SKILL.md`, the [agentskills.io](https://agentskills.io) open standard),
read natively by **Claude Code**, **GitHub Copilot** and **Cursor**.

## 🚀 Usage

Install with [pollen](https://github.com/groupbees/pollen): add this module to
your project's `pollen.yaml` (or to `~/.config/pollen/pollen.yaml` for every
project, `pollen update -g`), then run `pollen update`.

```yaml
  - repo: https://github.com/groupbees/skills-opensource
    revision: vX.Y.Z   # then `pollen autoupdate --freeze` pins its commit
    paths:
      - path: skills
        recurse: true
```

Everything else about pollen — pinning, updates, errors — is in its docs and in
the `pollen` skill it ships (add `groupbees/pollen` with `path: skills`).

## 📋 Skills Catalog

<!-- BEGIN skills-table — auto-generated from skills/*/*/SKILL.md by scripts/skills-catalog.sh --readme; do not edit by hand -->
| Domain | Skill | Description |
|--------|-------|-------------|
| Git | [commit-open-source](skills/git/commit-open-source/) | Commit message conventions for open-source and side-project repos outside the groupbees org — libraries, demos, blog-article repos hosted on a personal or community GitHub account. Trigger when creating a git commit in such a repo. Covers descriptive titles, HEREDOC for multi-line bodies, and the rule that NO Co-Authored-By trailer is added even on AI-assisted commits. Do NOT trigger for any repo under github.com/groupbees, public open-source ones included (e.g. pollen) — they use `commit-groupbees`. |
| Git | [tag-opensource](skills/git/tag-opensource/) | Conventions for creating release tags in open-source projects (PyPI packages, Terraform Registry modules, Go modules, npm packages, public GitHub releases). Trigger when creating/pushing a git tag in a public open-source repository — on a personal account or under an org, including public projects under github.com/groupbees such as pollen — or when the user mentions publishing to PyPI, the Terraform Registry, npm, or any public package registry. Covers semver + `v` prefix, annotated tags (signing optional), pre-tag release-prep (version-file bump), GitHub Releases as the changelog (no CHANGELOG.md; tag message body = release intro), per-stack publish flows (Python+uv+PyPI Trusted Publishing, Terraform Registry, Go modules), and registry-aware rollback/yank semantics. Do NOT trigger in GroupBees internal repos (private apps, IaC, catalogs whose tags deploy rather than publish) — use `tag-groupbees` there. |
<!-- END skills-table -->

> Generated from the skills' `description` fields: `scripts/skills-catalog.sh --readme`.

## ✅ CI

[`.github/workflows/ci.yml`](.github/workflows/ci.yml) calls the shared
`skills-module` workflow of `groupbees/.github`: `skill-validator check
--strict`, `pollen validate`, the generated README table, and
`shellcheck` on every `*.sh`.

## 🏷️ Versioning

Repo-wide git tags (`vX.Y.Z`) cut with `tag-opensource`; the GitHub Releases page
is the changelog.

## 🤝 Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
