# app-template

The standard setup for app repos: the repos that hold an application's
source and build the container image a GitOps repo (tnwks-ops) deploys.
One place defines it, and every app repo consumes it:

- **Reusable workflows** in [`.github/workflows/`](.github/workflows) hold
  all CI logic (`on: workflow_call`). An app repo keeps only thin callers,
  pinned to a template release by commit SHA.
- **A [Copier](https://copier.readthedocs.io) template** in
  [`template/`](template) renders those callers and the rest of the
  per-repo files (hooks, labels, Renovate config, Taskfile, `AGENTS.md`,
  `bin/privacy-check`). Each repo records the release it is on in
  `.copier-answers.yml`.
- **Renovate presets** in [`renovate/`](renovate), which app repos extend.

A template release (`vX.Y.Z` tag) reaches every app repo as a
`copier update` PR, opened by Renovate, that the repo's own CI tests.

Contents: [What an app repo gets](#what-an-app-repo-gets) ·
[Reusable workflows](#reusable-workflows) ·
[Create a new app repo](#create-a-new-app-repo) ·
[Move an existing repo onto the template](#move-an-existing-repo-onto-the-template) ·
[How updates flow](#how-updates-flow) ·
[CI](#ci-on-pull-requests) · [Releases and image tags](#releases-and-image-tags) ·
[Renovate](#renovate) · [Hooks](#hooks) · [Privacy](#privacy) ·
[Merging without branch protection](#merging-without-branch-protection) ·
[Releasing the template](#releasing-the-template) ·
[Working on the template](#working-on-the-template)

## What an app repo gets

Managed by Copier (a template update rewrites them; a local edit survives
the 3-way merge, but keep those rare):

| File | What it does |
| --- | --- |
| `.github/workflows/build.yaml` | `go` or `python` checks, then `image`: Docker build on PRs, push from `main` and/or `v*` tags |
| `.github/workflows/actionlint.yaml` | actionlint (with shellcheck) when a PR touches workflows |
| `.github/workflows/secret-scan.yaml` | betterleaks over the commits a PR (or a push to `main`) adds |
| `.github/workflows/privacy.yaml` | `bin/privacy-check` on the tree, the commits and identities, and the PR title and body |
| `.github/workflows/privacy-text.yaml` | the same check on comments, reviews and issues; deletes a comment with a hit |
| `.github/workflows/changelog.yaml` | a Changie fragment, unless labeled `skip-changelog` or opened by a bot |
| `.github/workflows/release.yaml` | Changie batch, release commit, `vX.Y.Z` tag, GitHub Release, image build for the tag |
| `.github/workflows/meta-labeler.yaml`, `.github/labeler.yaml` | `area/*` labels from the changed paths |
| `.github/workflows/meta-sync-labels.yaml`, `.github/labels.yaml` | labels synced from the file (others deleted) |
| `.github/renovate.json5` | extends this repo's presets at the release the repo is on |
| `.pre-commit-config.yaml` | betterleaks, privacy-check (staged files, git identity, commit message), the formatter |
| `.betterleaks.toml` | upstream rules plus a prose-credential rule |
| `.editorconfig`, `.gitattributes` | LF, UTF-8, two spaces; tabs for Go, four spaces for Python |
| `.changie.yaml`, `.changes/header.tpl.md` | changelog fragments and their format |
| `Taskfile.yml` | `precommit:init`, `precommit:run`, `privacy:check`, `privacy:sync`, `changelog:new`, `template:update`, `test`, plus `lint` (Go) or `format` (Python) |
| `bin/privacy-check` | the denylist check |
| `bin/install-hooks` | `pre-commit install --install-hooks`, then checks that the denylist loads |
| `AGENTS.md` | how to work in the repo, the privacy rules, no emdashes in prose |
| `.copier-answers.yml` | the answers and the template release (`_commit`); edit only through Copier |

Created once, then owned by the repo (`_skip_if_exists`): `README.md`
(with a "Workflow" section), `CHANGELOG.md`, `.gitignore`, `.golangci.yml`
(Go) and `.github/privacy-allow`.

Not managed: the code, `Dockerfile`, `go.mod`, requirements, tests, and
any extra workflow a repo adds for itself.

### Questions

`copier copy` asks a few things; the answers go to `.copier-answers.yml`.

| Question | Default | Used for |
| --- | --- | --- |
| `language` | (none) | `go` or `python`: the CI job, hooks, labels, editorconfig, Taskfile, Renovate preset |
| `has_image` | `true` | whether `build.yaml` has the `image` job (and release dispatches it) |
| `image_name` | empty | the image; empty means `ghcr.io/<owner>/<repo>` from the repo CI runs in |
| `image_push_main` | `true` | `true`: push `:main` and `:sha-<short>` from `main`. `false`: only `v*` tags push, and the push gate checks the whole history |
| `go_version` | `go.mod` | `go.mod` reads the go directive; anything else pins setup-go |
| `python_version` | `3.12` | setup-python for the host-side Python checks |
| `extra_test_command` | empty | bash after the language tests (Go: after `go test`; Python: after compileall) |
| `image_test_command` | empty | bash against the locally built image before any push; `$IMAGE_REF` is its tag |

The repo name is not a question. It is the destination directory's name,
and only fills the files created once (`README.md`, `.gitignore`), so no
repo name is ever stored in `.copier-answers.yml`.

## Reusable workflows

Called as `uses: iT3E/app-template/.github/workflows/<file>@<sha> # vX.Y.Z`.
Check names in the app repo are `<caller job> / <job>`, e.g. `go / lint`.
Secrets are passed explicitly (`secrets: PRIVACY_TERMS: ...`), never
`inherit`. The caller job grants the permissions listed.

| Workflow | Jobs | Inputs (default) | Secrets | Caller grants |
| --- | --- | --- | --- | --- |
| `go.yaml` | `lint`, `test` | `go-version` (empty: from `go-version-file`), `go-version-file` (`go.mod`), `working-directory` (`.`), `golangci-lint-version` (pinned), `test-command` (`go test -race ./...`), `extra-test-command` (empty) | none | `contents: read` |
| `python.yaml` | `check` | `python-version` (`3.12`), `working-directory` (`.`), `ruff-check` (`false`), `ruff-format` (`false`), `ruff-version` (pinned), `test-command` (empty) | none | `contents: read` |
| `image.yaml` | `image` | `image` (empty: `ghcr.io/<owner>/<repo>`), `context` (`.`), `dockerfile` (`Dockerfile`), `platforms` (empty), `build-args` (empty), `push-main` (`true`), `privacy-gate` (`tree` or `history`), `test-command` (empty). Output: `digest` | `PRIVACY_TERMS` | `contents: read`, `packages: write` |
| `privacy.yaml` | `privacy` | `allow-missing-terms` (`false`) | `PRIVACY_TERMS` | `contents: read` |
| `privacy-text.yaml` | `privacy-text` | none | `PRIVACY_TERMS` | `contents: read`, `issues: write`, `pull-requests: write` |
| `secret-scan.yaml` | `betterleaks` | `config` (`.betterleaks.toml`), `betterleaks-version` (pinned) | none | `contents: read` |
| `actionlint.yaml` | `actionlint` | `actionlint-version` (pinned) | none | `contents: read` |
| `changelog.yaml` | `changelog` | `skip-label` (`skip-changelog`), `changie-version` (pinned) | none | `contents: read` |
| `labeler.yaml` | `labeler` | `configuration-path` (`.github/labeler.yaml`) | none | `contents: read`, `pull-requests: write` |
| `sync-labels.yaml` | `sync-labels` | `config-file` (`.github/labels.yaml`), `delete-other-labels` (`true`) | none | `contents: read`, `issues: write` |
| `release.yaml` | `release` | `bump` (`auto`), `first-version` (`v0.1.0`), `build-workflow` (`build.yaml`; empty: none), `changie-version` (pinned). Output: `version` | none | `contents: write`, `actions: write` |

Each file's header comment has the details. The workflows run the caller's
own `bin/privacy-check`, so the matching is whatever release the repo is
on. Commands given as inputs run as bash scripts with `set -euo pipefail`.

The triggers stay in the callers: `pull_request`, `push` to `main`, `v*`
tags and `workflow_dispatch` for `build.yaml`; the comment and issue events
for `privacy-text.yaml`; a `paths:` filter for `actionlint.yaml` and
`release.yaml`; `concurrency` for `build.yaml` and `release.yaml`.

## Create a new app repo

```bash
gh repo create iT3E/<repo> --private --clone
cd <repo>
copier copy --vcs-ref vX.Y.Z https://github.com/iT3E/app-template.git .
```

Use the `https://github.com/iT3E/app-template.git` source (not `gh:` or a
local path): it is what `_src_path` records, and Renovate's copier manager
looks the tags up from it. Leave `--vcs-ref` out to get the latest
release. "Use this template" on GitHub (it is a template repository) gives
a copy of this repo, not an app repo; run `copier copy` in the new repo
either way.

Then:

1. Set the `PRIVACY_TERMS` Actions secret from a tnwks-ops checkout
   (`task privacy:sync`, see [Privacy](#privacy)) and run
   `bin/privacy-check`. Anything it finds is renamed, not allow-listed.
2. `task precommit:init` in every clone.
3. Fill in `README.md` and the "What this is" section of `AGENTS.md`.
4. Add a Changie fragment (`changie new`), open everything as one PR, get
   every check green, and merge with `gh-merge-safe.sh <pr> iT3E/<repo>`.
5. Grant the Renovate app access to the repo (see [Renovate](#renovate)).
6. If tnwks-ops deploys it, add the repo to tnwks-ops'
   `docs/development/app-repos.md`.

## Move an existing repo onto the template

On a branch, with a clean tree:

```bash
copier copy --overwrite --defaults --vcs-ref vX.Y.Z \
  --data language=go --data image_push_main=true \
  https://github.com/iT3E/app-template.git .
git diff
```

`--overwrite` replaces the managed files with the rendered ones. Review
the diff and put back what is repo-specific (the "What this is" section of
`AGENTS.md`, rules in `.github/renovate.json5` below the `extends`,
extra tasks or hooks). Copier keeps those edits on later updates. Files
in `_skip_if_exists` are left alone if they exist. Commit it all with
`.copier-answers.yml` as one PR.

## How updates flow

1. A change merges here with a Changie fragment, and `self-release.yaml`
   tags `vX.Y.Z` (see [Releasing the template](#releasing-the-template)).
2. In each app repo, Renovate's copier manager sees a newer tag than the
   `_commit` in `.copier-answers.yml`, runs `copier update --vcs-ref
   vX.Y.Z` (no `--trust`; the template needs none), and opens a PR titled
   `chore(template): update app-template to vX.Y.Z`, labeled
   `renovate/template`. One PR per release, never grouped with other
   updates.
3. That PR rewrites the managed files. Every caller's `@<sha> # vX.Y.Z`
   moves to the new release's commit (the template renders it from
   `_copier_conf.vcs_ref_hash`), and so does the `#vX.Y.Z` on the Renovate
   presets. The repo's own CI then runs the new reusable workflows on the
   PR itself.
4. It is reviewed and merged by hand with `gh-merge-safe.sh` (no
   automerge: it changes CI, so a green run alone proves little).

Without Renovate, or for a specific release: `task template:update` (the
latest) or `task template:update -- --vcs-ref vX.Y.Z`, on a branch, then
a PR. A conflict between a local edit and the template shows up as
conflict markers or a `.rej` file in that PR.

Renovate does not open separate PRs for the `iT3E/app-template` workflow
pins or for the versions inside managed files (hook revs, CI tools in the
Taskfile): `renovate/default.json5` turns those off in app repos, since
the template's own Renovate bumps them here and they arrive with the next
release.

## CI on pull requests

Every check must pass before a merge. `gh-merge-safe.sh` waits for all of
them, and Renovate's `ignoreTests: false` does the same for its own PRs.

- **Go** (`go / lint`, `go / test`): `gofmt -l` must be empty, then `go vet
  ./...`, golangci-lint (`.golangci.yml`, v2 `standard` linters; on PRs and
  only for issues the PR introduces), and `go test -race ./...`. The Go
  version comes from `go.mod`.
- **Python** (`python / check`): `python -m compileall` (every file must
  parse; no dependencies needed), then ruff and host-side tests where the
  repo turns them on. Tests that need the app's environment run in the
  image instead (`image_test_command`).
- **Image** (`image / image`): the Docker build, then `image_test_command`
  against it, all without pushing.
- **All**: `actionlint` (workflow changes only), `betterleaks`, `privacy`,
  `changelog`, `labeler`.

## Releases and image tags

Each PR adds a Changie fragment (or carries `skip-changelog`), and merging
fragments to `main` runs `release.yaml`. It batches them into
`.changes/<version>.md` and `CHANGELOG.md`, pushes a `chore(release):
vX.Y.Z` commit, tags it, and publishes a GitHub Release.
Fixed/Security bump the patch, Added/Changed/Deprecated/Removed the minor,
Breaking the major. `first-version` (`v0.1.0`) applies only while
`.changes/` has no `v*.md`; a repo with older hand-made tags backfills them
there so the next release follows.

- **No release app.** Private repos on GitHub Free have no rulesets, so
  `GITHUB_TOKEN` (`contents: write`) can push the release commit and tag.
- **The build is dispatched.** A tag pushed with `GITHUB_TOKEN` starts no
  workflow, so `release.yaml` ends with `gh workflow run build.yaml --ref
  vX.Y.Z` (`actions: write`). A tag pushed by hand triggers `build.yaml`
  directly, but the next Changie release won't know about it, so tags come
  from `release.yaml`.

Image tags must stay what the deploy repo pins:

| Event | `image_push_main: true` | `image_push_main: false` |
| --- | --- | --- |
| PR | build only (`pr-<n>` stays local) | build only |
| push to `main` | `:main`, `:sha-<short>` | build only |
| `v*` tag | `:<version>`, `:sha-<short>` | `:<version>`, `:sha-<short>` |

`metadata-action` runs with `latest=false`, so nothing pushes `:latest`.
Before a push, `image.yaml` runs `bin/privacy-check` on the exact tree
(`privacy-gate: tree`) or on every commit the tag ships (`history`, the
tag-only repos). A repo deployed as `:main` with `imagePullPolicy: Always`
ships each merge on the next pod restart; the release commit triggers
nothing, so `:main` stays on the merge commit's image (same code). A
tag-only repo reaches the cluster when a deploy-repo PR moves its
`:<version>@sha256` pin.

## Renovate

An app repo's `.github/renovate.json5`:

```json5
extends: [
  "github>iT3E/app-template//renovate/default.json5#vX.Y.Z",
  "github>iT3E/app-template//renovate/go.json5#vX.Y.Z",   // or python.json5
],
```

[`renovate/default.json5`](renovate/default.json5) starts from the same
base as tnwks-ops (`config:best-practices`, `docker:enableMajor`,
`:semanticCommits`, `:enablePreCommit`, `helpers:pinGitHubActionDigests`,
the dashboard) and extends three of its shared presets (`commitMessage`,
`labels`, `semanticCommits`). It adds:

- **The copier manager**, with template updates as their own
  `chore(template)` PRs, labeled `renovate/template`, never automerged.
- **Template-owned files off**: no Renovate updates in the managed
  workflows, `.pre-commit-config.yaml` or `Taskfile.yml`, and none for the
  `iT3E/app-template` workflow pins anywhere.
- **Automerge** once every check is green: GitHub Actions
  minor/patch/digest (3 days after release, none for `actions/*`),
  pre-commit hooks, and CI tools pinned with `# renovate:` comments.
- **Reviewed** (left open): majors and 0.x minors.
- `platformAutomerge: false` (GitHub Free has no auto-merge on private
  repos; Renovate merges, squash), `schedule: ["before 7am on monday"]`
  (Actions minutes on private repos are metered), `docker:pinDigests` on
  (base images are pinned `tag@sha256`).

[`renovate/go.json5`](renovate/go.json5): `renovate/go` labels and `go`
commit scopes, `gomodTidy` and `gomodUpdateImportPaths`, the go directive
and the `golang` image as one "Go toolchain" group, and automerge for Go
module and base image patch/digest and non-0.x minors.

[`renovate/python.json5`](renovate/python.json5): `renovate/python` labels
and `python` scopes, `constraints.txt` as a requirements file, automerge
for package and base image patch/digest and non-0.x package minors, and
Python image minors reviewed.

Rules only one repo needs (version caps, say) go in its own
`renovate.json5` below the `extends`.

The Renovate GitHub App only sees the repos it is granted. Adding a repo
is a UI-only step: <https://github.com/settings/installations>, Renovate,
Configure, Repository access, "Select repositories", add the repo, Save.
With a config already in place, the first run opens the "Renovate
Dashboard 🤖" issue (and a "Pin dependencies" PR if anything is unpinned)
instead of an onboarding PR. Job logs: `https://developer.mend.io/github/iT3E/<repo>`.

## Hooks

`.pre-commit-config.yaml` installs pre-commit and commit-msg hooks
(`default_install_hook_types`), via `bin/install-hooks` / `task
precommit:init`:

- `betterleaks` (staged changes, `.betterleaks.toml`)
- `privacy-check` (`bin/privacy-check --cached`: staged files and your git
  identity) and `privacy-check-message` (the commit message)
- the formatter: `gofmt -l -w` as a `language: golang` hook (pre-commit
  downloads Go if it isn't installed), or `ruff-format`. A repo whose code
  isn't formatted yet can move ruff to `stages: [manual]` locally until a
  formatting PR lands.

golangci-lint runs in CI only (`task lint` runs it in Docker).

## Privacy

Images built from app repos are public, and this repo is public, so the
privacy rules of tnwks-ops apply in full: no family names, personal email
addresses, or anything else that says who is who. The denylist has one
source, `.github/privacy-terms.sops.yaml` in tnwks-ops, and is never
committed anywhere else or printed.

- **CI** reads it from each repo's `PRIVACY_TERMS` Actions secret, passed
  to the reusable workflows explicitly. Set or refresh it from a tnwks-ops
  checkout (`task privacy:sync` in the app repo):

  ```bash
  sops -d --extract '["terms"]' .github/privacy-terms.sops.yaml \
    | gh secret set PRIVACY_TERMS --repo iT3E/<repo>
  ```

  Re-sync every app repo (and this one) after the list changes. In a
  private app repo a missing secret fails the check. This public repo
  warns instead on fork PRs, which get no secrets.
- **Locally**, `bin/privacy-check` reads `$PRIVACY_TERMS`, or else
  decrypts the list from a tnwks-ops checkout at `../tnwks-ops` (or
  `$TNWKS_OPS_DIR`). The hooks fail closed if it can't load.
- **Text you post** (PR bodies, comments): write it to a file, run
  `bin/privacy-check --text < file`, and post with `--body-file`.
- **Exceptions** are the owner's call and name-free:
  `.github/privacy-allow` (pathspec globs, files only) or `privacy:allow`
  on a line. Commit messages, PR text and comments have none.

## Merging without branch protection

Rulesets and branch protection need GitHub Pro on private repos, so
nothing enforces review or green checks on an app repo's `main`:

- Every change is a PR, squash-merged. No direct pushes to `main`, by
  people or agents.
- Merge with `gh-merge-safe.sh <pr> iT3E/<repo>`: it waits for every check
  and refuses unless all passed or were skipped.
- Renovate automerges only with every check green (`ignoreTests: false`,
  `platformAutomerge: false`).
- The only direct push is `release.yaml`'s release commit, which touches
  only `.changes/` and `CHANGELOG.md`.

If a repo goes public or the account moves to Pro, add a `main` ruleset
with these checks required (and a release app, since `GITHUB_TOKEN` can't
bypass it).

## Releasing the template

The template releases itself the way app repos do, with Changie:

1. In the PR, add a fragment: `changie new` (Fixed for a fix, Added or
   Changed for something new, Breaking for a renamed or removed input,
   job or question). A PR that app repos shouldn't get labels
   `skip-changelog` and releases nothing.
2. Merge it (`gh-merge-safe.sh <pr> iT3E/app-template`). `self-release.yaml`
   batches the fragments, commits `chore(release): vX.Y.Z`, tags it and
   publishes the GitHub Release.
3. The tag is the release. App repos get it as a Renovate PR (see
   [How updates flow](#how-updates-flow)), or with `task template:update`.

Tags are plain `vX.Y.Z` (Renovate's copier manager reads git tags with
pep440 versioning). Never move or delete a tag: app repos pin its commit.
To re-run a release by hand, use the workflow's `bump` input.

## Working on the template

- `template/` is the Copier subdirectory (`_subdirectory`). Jinja there
  uses `[[ ]]`, `[% %]` and `[# #]` (`_envops`), so `${{ }}` and `{{ }}`
  pass through untouched; only `*.jinja` files are rendered.
- Keep `copier.yml` free of unsafe features (`_tasks`, `_migrations`,
  `_jinja_extensions`, `_external_data`). Renovate's hosted copier manager
  runs without `--trust` and refuses them; `template-ci.yaml` checks.
- `task render` renders each `tests/*.yml` answers set into `out/` from
  the last commit.
- `template-ci.yaml` renders the same sets in CI and runs actionlint (with
  the reusable workflows swapped in as local files, so callers are checked
  against their inputs), `pre-commit validate-config`, yamllint and
  shellcheck on them, checks the `@<sha>` pins, applies `copier update`
  from the latest tag and compares it with a fresh render, and runs
  `renovate-config-validator` on the presets.
- This repo calls its own reusable workflows from `self-*.yaml` (privacy,
  secret scan, changelog, labels, release), so a PR here runs the version
  it changes.
- Reusable workflow inputs, secrets, job names and Copier questions are an
  interface: add with defaults (minor), don't rename or remove without a
  Breaking fragment.
