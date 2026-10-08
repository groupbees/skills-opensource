---
name: tag-opensource
description: Conventions for creating release tags in open-source projects (PyPI packages, Terraform Registry modules, Go modules, npm packages, public GitHub releases). Trigger when creating/pushing a git tag in a public open-source repository — on a personal account or under an org, including public projects under github.com/groupbees such as pollen — or when the user mentions publishing to PyPI, the Terraform Registry, npm, or any public package registry. Covers semver + `v` prefix, annotated tags (signing optional), pre-tag release-prep (version-file bump), GitHub Releases as the changelog (no CHANGELOG.md; tag message body = release intro), per-stack publish flows (Python+uv+PyPI Trusted Publishing, Terraform Registry, Go modules), and registry-aware rollback/yank semantics. Do NOT trigger in GroupBees internal repos (private apps, IaC, catalogs whose tags deploy rather than publish) — use `tag-groupbees` there.
---

# Open-source release tag conventions

**Scope**: decided by what the project *is*, not by who owns it. A public open-source project — a personal library or a public repo under an org such as `groupbees/pollen` — follows this skill. A GroupBees internal repo, whose tags trigger deployments, follows `tag-groupbees`. Commits and PRs follow the owning org's conventions either way (`commit-groupbees` / `pr-groupbees` inside groupbees).

OSS tags are **publication triggers + public history markers**. Once a tag is pushed and the publish job runs, the version is on a public registry where consumers' lockfiles will pin to it. Treat the tag push as immutable — yanking is possible but disruptive.

## Tag format

- **Format**: `vX.Y.Z` (semver, with `v` prefix). Always **annotated** (`git tag -a`). **Signing is optional and off by default**: artifact provenance is already covered by the registry (PyPI attestations via Trusted Publishing, see below), and a GPG passphrase prompt blocks non-interactive tagging. Sign (`git tag -s`) only if it is decided globally for all projects, with `pinentry-mac` + Keychain so it stays non-interactive.
- **Pre-release suffixes**:
  - General: `vX.Y.Z-rc1`, `vX.Y.Z-alpha1`, `vX.Y.Z-beta1`, `vX.Y.Z-dev0`
  - **Python / PEP 440 nuance**: PyPI normalizes versions per [PEP 440](https://peps.python.org/pep-0440/). `1.2.0rc1`, `1.2.0a1`, `1.2.0b1`, `1.2.0.dev1` are the canonical forms PyPI displays. SemVer with a hyphen (`1.2.0-rc1`) is accepted but normalized to `1.2.0rc1`. Use the PEP 440 form directly in `pyproject.toml` to avoid surprise diffs.
- **Bump rules**:
  - **Patch** (`v1.2.0 → v1.2.1`): bugfix-only, fully backwards compatible.
  - **Minor** (`v1.2.0 → v1.3.0`): additive non-breaking (new public API, new optional param with default).
  - **Major** (`v1.2.0 → v2.0.0`): any breaking change in the public API — function rename, parameter removal, behavior change a caller could observe, dropped Python/runtime version support.

For OSS, be **stricter than internal** on what counts as breaking. A removed kwarg with a default that nobody documented is still a breaking change to someone's lockfile.

## Source-of-truth version per stack

Where the version lives differs by ecosystem. The tag and the source-of-truth must agree.

| Stack | Version source-of-truth | Auto-derive from tag? |
|---|---|---|
| **Python (uv / Hatch / PDM)** | `pyproject.toml` `[project] version` | Yes, via `hatch-vcs` or `setuptools-scm` — version derives from `git describe`. Highly recommended (one less thing to bump). |
| **Python (Poetry pre-2.0)** | `pyproject.toml` `[tool.poetry] version` | Less common; `poetry-dynamic-versioning` exists. |
| **Terraform module** | None — **tag IS the version**. Terraform Registry reads tags directly. | N/A (no manual file to bump). |
| **Go module** | None — tag IS the version. Module path in `go.mod` only changes at major bumps (`/v2`, `/v3` for v2+). | N/A. |
| **npm** | `package.json` `version` | Less common to auto-derive; `npm version <patch\|minor\|major>` is the standard bump command. |

**Recommendation for Python projects**: use `hatch-vcs` so the tag is the only place version lives. Eliminates "did I bump the file?" mistakes.

## Pre-tag checklist

Run through these BEFORE `git tag`:

1. **Local `main` is clean and up to date**:
   ```bash
   git fetch && git checkout main && git pull --ff-only
   git status   # working tree clean
   ```
2. **CI is green on the commit being tagged** — same rule as foundation. A red tag means a failed publish job + a half-published state.
3. **Version-file bump merged** (if not auto-deriving from tag): the release-prep PR with the new `pyproject.toml` / `package.json` version is merged to `main`. Tag the merge commit.
4. **Release notes = the GitHub Release, no `CHANGELOG.md`**: one source of truth instead of two that drift apart. The release is built from (a) the **body of the annotated tag message** (intro, highlights, breaking changes) and (b) the **generated list of PRs** merged since the previous tag (`generate_release_notes: true`). This only works if PR titles are descriptive and PRs are squash-merged, which is the convention anyway. Point package metadata to the Releases (`[project.urls] Changelog = "https://github.com/<owner>/<repo>/releases"` for Python).
5. **Breaking changes are loud**: in the tag message body (so in the release intro), breaking changes need a dedicated section with a migration snippet. OSS consumers don't read your commit history.
6. **License + README sanity**: README install command shows the new version (or `pip install <pkg>` if unpinned — preferred). License year is current.
7. **For libraries with type stubs**: `py.typed` marker present in the package, mypy/ruff/pyright green.

### Release-prep PR template (Python example)

A typical OSS release-prep PR:

```
Title: Release v1.3.0

Diff:
- pyproject.toml: version = "1.2.0" → "1.3.0"  (skip if hatch-vcs)
- README.md: bump install pin example if pinned
- (release notes are written in the tag message, not in a file)
- (no code changes — those merged in their own PRs)
```

Merge → tag the merge commit on `main`.

## Creating the tag

**Annotated** (tag the merge commit of the release-prep PR):

```bash
git tag -a v1.3.0 <merge-commit> -m "$(cat <<'EOF'
v1.3.0

<One-paragraph intro: what this release is about.>

Highlights:
- <headline change 1>
- <headline change 2>

Breaking changes:            # omit when there are none
- <change> — migrate with: <snippet>
EOF
)"
```

The first line is the tag subject; **everything after it becomes the intro of the GitHub Release**
(see the publish workflow below), followed by the generated list of merged PRs. Write it for users.

If signing is adopted later (globally, not per project):

```bash
git config --global user.signingkey <KEYID>
git config --global tag.gpgSign true          # `git tag -a` then signs too
brew install pinentry-mac && echo "pinentry-program $(brew --prefix)/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf
# public key on GitHub: account Settings → SSH and GPG keys (not the repo settings);
# the key's email must be a verified address of the account, else tags show "Unverified"
```

Keep a passphrase on the key: an unprotected private key lets anyone with disk access publish "Verified" tags in your name.

**Push** — this is the publish trigger:

```bash
git push origin v1.3.0
```

## Per-stack publish flow

### Python → PyPI (Trusted Publishing, the 2026 standard)

No API tokens. PyPI verifies the GitHub workflow via OIDC. The publish action also uploads **PEP 740 attestations** (Sigstore) by default: the PyPI page shows which repo, workflow and commit built each file. That is the package-level signature (PyPI no longer accepts GPG signatures); nothing to configure.

One-time PyPI setup:
1. PyPI project → **Publishing** → add a Trusted Publisher: workflow = `release.yml`, repo = `<owner>/<repo>`, environment = `pypi` (recommended for the approval gate).

Workflow (`.github/workflows/release.yml`):

```yaml
on:
  push:
    tags: ["v*"]

jobs:
  publish:
    runs-on: ubuntu-latest
    environment: pypi   # manual approval gate
    permissions:
      id-token: write   # for OIDC
      contents: write   # for GitHub Release creation
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv build                       # produces dist/*.whl + dist/*.tar.gz
      - uses: pypa/gh-action-pypi-publish@release/v1   # no with: password — OIDC

  github-release:
    needs: publish
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - name: Release intro = body of the annotated tag message
        run: |
          # A shallow checkout can see an annotated tag as lightweight: fetch the tag object itself.
          git fetch --force origin "refs/tags/${GITHUB_REF_NAME}:refs/tags/${GITHUB_REF_NAME}"
          # contents:body excludes the subject line and the GPG signature
          git for-each-ref "refs/tags/${GITHUB_REF_NAME}" --format='%(contents:body)' > release-intro.md
      - uses: softprops/action-gh-release@v2
        with:
          body_path: release-intro.md
          generate_release_notes: true   # appended after the intro
```

### Terraform module → Terraform Registry

1. Repo named `terraform-<provider>-<name>` (e.g. `terraform-google-vpc`).
2. Public GitHub repo, connected to your Terraform Registry namespace.
3. Tag `vX.Y.Z` → Registry auto-detects and publishes within a few minutes.
4. No publish workflow needed in the repo — Registry polls GitHub.
5. **README.md drives the Registry page** — keep it documented per Registry conventions (Usage / Inputs / Outputs sections).

For Major v2+: the module path stays the same on Terraform Registry (unlike Go). Just tag `v2.0.0` and consumers update their `version = "~> 2.0"`.

### Go module → proxy.golang.org

1. Tag `vX.Y.Z` → `proxy.golang.org` indexes it automatically within minutes.
2. **Major v2+ exception**: the import path must include the major version suffix. For v2 of `github.com/you/foo`, the module path in `go.mod` becomes `github.com/you/foo/v2` and consumers import `github.com/you/foo/v2/...`. This is a hard Go convention, enforced by the toolchain.
3. No publish workflow needed — module proxy reads tags directly.

### npm → npmjs.com

1. Bump `package.json` (or `npm version patch|minor|major` which bumps + tags in one step).
2. Workflow on tag push: `npm publish --provenance --access public`. `--provenance` (npm 9.5+) attests the publish to a public build, similar to PyPI Trusted Publishing.
3. Requires `NPM_TOKEN` secret OR npm Trusted Publishers (in beta as of 2026 — prefer once GA).

## Post-tag verification

After pushing:

1. **Publish job** in Actions → green.
2. **Registry shows the new version**:
   - PyPI: `https://pypi.org/project/<pkg>/<version>/`
   - Terraform Registry: `https://registry.terraform.io/modules/<ns>/<name>/<provider>/<version>`
   - Go: `https://pkg.go.dev/<module>@<version>` (may take 10–30 min to index)
   - npm: `https://www.npmjs.com/package/<pkg>/v/<version>`
3. **GitHub Release** created at `https://github.com/<owner>/<repo>/releases/tag/v1.3.0`: the tag message body as intro, then the generated PR list. Fix wording there if needed (editing a Release is safe).
4. **Smoke test install** from a fresh venv:
   ```bash
   uv venv /tmp/smoke && cd /tmp/smoke && uv pip install <pkg>==1.3.0
   python -c "import <pkg>; print(<pkg>.__version__)"
   ```

## Rollback / yank semantics per registry

| Registry | Can delete? | Can yank/deprecate? | Notes |
|---|---|---|---|
| **PyPI** | No (deletion is permanent; ask PyPI admins) | Yes — yank a release: `pypi.org/manage/project/<pkg>/release/<v>/` → Yank. Yanked versions stay installable when pinned exactly but excluded from resolvers. | Yanking is the right answer for "this was broken" — preserves consumer reproducibility while warning new installs off. |
| **Terraform Registry** | No directly; can hide via Registry support | Effectively no yank — Registry mirrors GitHub tags | Best fix: tag `vX.Y.Z+1` immediately with the revert/fix. Update README to recommend the new version. |
| **Go proxy** | No (proxy mirror is immutable) | Yes — `retract` directive in `go.mod`: `retract v1.3.0 // contains breaking bug` | Add the retract, tag a new patch, push. Consumers see deprecation warnings on `go get`. |
| **npm** | Yes within 72hr of publish (`npm unpublish`); after, only `npm deprecate` | Yes — `npm deprecate <pkg>@1.3.0 "broken, use 1.3.1"` | Unpublish breaks downstream lockfiles — avoid in favor of deprecate + new patch. |

**Universal rule**: never `git push --force` a tag that's already been published. The registry has the old artifact; moving the tag desyncs Git history from registry state.

If a tag pushed an unintended version:

1. **Tag a corrective patch** (`v1.3.1` reverting the change) — always works, always safe.
2. **Yank/deprecate the bad version** on the registry per the table above.
3. **GitHub Release**: edit the release notes to add a "⚠️ Use vX.Y.Z+1 instead" banner. Don't delete the Release — consumers may have it bookmarked.

## What to avoid

- ❌ **Lightweight tags** (`git tag v1.3.0` without `-a`). Lose the message, hence the release intro.
- ❌ **Missing `v` prefix** (most CI workflows + Terraform Registry filter on `v*`).
- ❌ **Tagging without bumping the version file** (unless using `hatch-vcs` / `setuptools-scm`). Causes `pip install <pkg>==1.3.0` to install something that reports `__version__ == "1.2.0"`.
- ❌ **Force-pushing a tag that's been published**. Registry retains the old artifact; you've created a Git-vs-registry split brain.
- ❌ **Publishing a major version without a migration guide** in the README + the release intro (tag message). OSS consumers will open issues.
- ❌ **Maintaining a `CHANGELOG.md` next to GitHub Releases**. Two sources drift apart; the Release is the changelog. Exception: a package consumed offline or under a compliance rule that requires a changelog in the sdist: then generate it from the Releases, don't hand-write both.
- ❌ **Manual `twine upload` with long-lived API tokens** in 2026 — Trusted Publishing (PyPI) / Provenance (npm) is the standard, no tokens.
- ❌ **Skipping the release-prep PR** and tagging directly off a feature branch or unreviewed commit. Always tag the merge commit on `main`.
- ❌ **Tagging from a maintenance branch (e.g. `v1.x`) without a corresponding `1.x` tag scheme**. Backport patches to `v1.x` branch → tag `v1.5.4` from that branch is fine; consumers expect the older line to keep receiving patches as long as it's supported.
- ❌ **`Co-Authored-By: Claude ...` trailer on release-prep commits or tag messages**. Do NOT append it, even on AI-assisted work — same rule as internal repos.
