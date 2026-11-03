# app-repo-template

The standard setup for app repos: the repos that hold an application's
source and build the container image a GitOps repo (tnwks-ops) deploys.
Two public repos define it:

- **This repo**, the scaffolding: a [Copier](https://copier.readthedocs.io)
  template in [`template/`](template) that renders an app repo's thin
  caller workflows and the rest of its per-repo files (hooks, labels,
  Renovate config, Taskfile, `AGENTS.md`, `bin/privacy-check`). Each repo
  records the release it is on in `.copier-answers.yml`.
- **[iT3E/github-workflows](https://github.com/iT3E/github-workflows)**,
  what runs at CI time: the reusable workflows (`on: workflow_call`, all
  the CI logic) and the Renovate presets. Its README documents every
  workflow, input and secret.

The template pins one github-workflows release (`github_workflows` in
[`copier.yml`](copier.yml)), and every rendered caller and preset uses
it. A template release reaches every app repo as a `copier update` PR,
opened by Renovate, that the repo's own CI tests. A github-workflows
release travels the same way, through a template release (see
[How updates flow](#how-updates-flow)).

Contents: [What an app repo gets](#what-an-app-repo-gets) ·
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
the 3-way merge, but keep those rare). Every workflow here is a thin
caller of the github-workflows workflow of the same purpose, pinned
`@<sha> # vX.Y.Z`:

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
| `.github/renovate.json5` | extends the github-workflows presets at the release the template pins |
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
| `image_test_postgres` | `false` | a throwaway Postgres for `image_test_command`: CI's service on `127.0.0.1:5432`, `task test`'s container on `127.0.0.1:55432`, both as `$TEST_POSTGRES_URL` (containers reach it with `--network host`) |

The repo name is not a question. It is the destination directory's name,
and only fills the files created once (`README.md`, `.gitignore`), so no
repo name is ever stored in `.copier-answers.yml`.

## Create a new app repo

```bash
gh repo create iT3E/<repo> --private --clone
cd <repo>
copier copy --vcs-ref vX.Y.Z https://github.com/iT3E/app-repo-template.git .
```

Use the `https://github.com/iT3E/app-repo-template.git` source (not `gh:`
or a local path): it is what `_src_path` records, and Renovate's copier
manager looks the tags up from it. Leave `--vcs-ref` out to get the latest
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
  https://github.com/iT3E/app-repo-template.git .
git diff
```

`--overwrite` replaces the managed files with the rendered ones. Review
the diff and put back what is repo-specific (the "What this is" section of
`AGENTS.md`, rules in `.github/renovate.json5` below the `extends`,
extra tasks or hooks). Copier keeps those edits on later updates. Files
in `_skip_if_exists` are left alone if they exist. Commit it all with
`.copier-answers.yml` as one PR.

## How updates flow

Both repos release with Changie (`vX.Y.Z` tags). Each app repo gets every
change, whether to the template or to the workflows, as **one Renovate
PR**: `chore(template): update app-repo-template to vX.Y.Z`, labeled
`renovate/template`.

**A template change:**

1. A PR here merges with a Changie fragment, and `release.yaml` tags
   `vX.Y.Z` (see [Releasing the template](#releasing-the-template)).
2. In each app repo, Renovate's copier manager sees a newer tag than the
   `_commit` in `.copier-answers.yml`, runs `copier update --vcs-ref
   vX.Y.Z` (no `--trust`; the template needs none), and opens the PR. One
   PR per release, never grouped with other updates.
3. The repo's own CI runs on it, with the workflows it now calls, and
   Renovate merges it once every check is green (every update type; the
   repo's CI running the new callers is the test). It changes only repo
   tooling, so it releases nothing in the app repo.

**A github-workflows change** takes two hops, both automatic:

1. A PR in github-workflows merges with a fragment, and it tags `vA.B.C`.
   A Renovate merge there that changes a reusable workflow or a preset
   tags a patch by itself.
2. Here, Renovate opens one PR (label `renovate/github-workflows`) moving
   `github_workflows` in `copier.yml` (a custom regex manager), this
   repo's own callers and its preset `#vA.B.C`. Template CI lints the
   rendered callers against the workflows of `vA.B.C`, and the PR merges
   itself when green (majors wait for review). The merge moves
   `copier.yml`, so `release.yaml` runs, adds a fragment for the bump
   (`Fixed` for a patch release of the workflows, `Changed` otherwise)
   and tags a template release.
3. That template release reaches the app repos as above. Every caller's
   `@<sha> # vA.B.C` and the presets' `#vA.B.C` move together in that one
   PR.

**Why the template owns the pins.** App repos never bump a
github-workflows pin themselves: `renovate/default.json5` turns off the
`iT3E/github-workflows` updates (workflow pins and presets) and every
managed file there. So a managed caller line in an app repo is always
exactly what the template rendered last time, and `copier update`'s 3-way
merge takes the new line without a conflict. The other design, Renovate
bumping the pins in each app repo directly, would be one hop shorter, but
a repo pin that moved ahead of the template's makes every caller line a
conflict on the next `copier update` (both sides changed the same line to
different values). Tested on a rendered repo: with the pin bumped locally
to a newer release than the template's, `copier update` left conflict
markers in every caller and the presets; with the template owning the pin
it applied cleanly.

Without Renovate, or for a specific release: `task template:update` (the
latest) or `task template:update -- --vcs-ref vX.Y.Z`, on a branch, then
a PR. A conflict between a local edit and the template shows up as
conflict markers or a `.rej` file in that PR.

## CI on pull requests

The jobs come from the github-workflows release the repo pins; its README
has each workflow's details. Every check must pass before a merge. `gh-merge-safe.sh` waits for all of
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
  against it (with a throwaway Postgres if `image_test_postgres`), all
  without pushing. A long or changing test belongs in a script the repo
  owns (say `bin/image-test`), with `image_test_command` just calling it,
  so editing a test doesn't mean re-answering Copier.
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

- **Dependency updates release themselves** (repos with an image).
  Renovate PRs carry no fragment, so `release.yaml` runs on every push to
  `main` and passes `auto-fragment: "Dependency updates"`: when no
  fragment is pending and a Renovate (or Dependabot) commit since the last
  tag changed anything outside docs, `.github/` and the repo tooling this
  template manages (`.copier-answers.yml`, `Taskfile.yml`,
  `.pre-commit-config.yaml`, `bin/install-hooks`, `bin/privacy-check`, the
  dotfiles), it writes a `Fixed` fragment naming those commits and tags a
  patch. So a merged Go module, Python package or base image update
  becomes an image and a deploy-repo PR. Template updates and CI-only
  bumps release nothing. A person's PR still needs its own fragment; one
  labeled `skip-changelog` releases nothing. The paths are github-workflows'
  `ship-paths` default.

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
`:<version>@sha256` pin. tnwks-ops deploys every app that way now, and
its Renovate automerges those PRs behind its own CI.

## Renovate

An app repo's `.github/renovate.json5` extends the presets in
iT3E/github-workflows at the release the template pins:

```json5
extends: [
  "github>iT3E/github-workflows//renovate/default.json5#vX.Y.Z",
  "github>iT3E/github-workflows//renovate/go.json5#vX.Y.Z",   // or python.json5
],
```

What they do (the copier manager, automerge tiers, the weekly schedule,
what is reviewed) is in the
[github-workflows README](https://github.com/iT3E/github-workflows#renovate-presets).
Rules only one repo needs (version caps, say) go in its own
`renovate.json5` below the `extends`; copier update keeps them.

This repo's own `.github/renovate.json5` extends the same default preset
(at the pinned release), keeps Renovate on for its own workflows and
`template/` sources, and adds the `github-workflows` group described in
[How updates flow](#how-updates-flow).

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
  to the github-workflows reusable workflows explicitly. Set or refresh it from a tnwks-ops
  checkout (`task privacy:sync` in the app repo):

  ```bash
  sops -d --extract '["terms"]' .github/privacy-terms.sops.yaml \
    | gh secret set PRIVACY_TERMS --repo iT3E/<repo>
  ```

  Re-sync every app repo (and this repo and github-workflows) after the
  list changes. In a private app repo a missing secret fails the check.
  The two public repos warn instead on fork PRs, which get no secrets.
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
2. Merge it (`gh-merge-safe.sh <pr> iT3E/app-repo-template`).
   `release.yaml` batches the fragments, commits `chore(release): vX.Y.Z`,
   tags it and publishes the GitHub Release. Renovate's github-workflows
   PR needs no fragment: `release.yaml` writes one when a push moves the
   `github_workflows` pin. Any other Renovate merge that changes
   `template/` or `copier.yml` (a hook rev, a pinned tool) releases a
   patch through `auto-fragment`; one that only touches this repo's own
   CI releases nothing.
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
- `github_workflows` in `copier.yml` is the github-workflows release, as
  `"vX.Y.Z@<the tag's commit>"`. Templates use `[[ github_workflows_ref ]]`
  and `[[ github_workflows_sha ]]` (derived, never stored in
  `.copier-answers.yml`). To move it by hand, change it together with
  this repo's callers and `.github/renovate.json5`.
- `template-ci.yaml` checks that pin (the tag's commit, and the same
  release in this repo's callers and preset), renders the same sets as
  `task render` and checks the rendered pins, runs actionlint with the
  reusable workflows of the pinned release swapped in as local files (so
  callers are checked against their inputs), `pre-commit
  validate-config`, yamllint and shellcheck, applies `copier update` from
  the newest tag the commit descends from and compares it with a fresh
  render, and runs `renovate-config-validator` on this repo's and the
  rendered configs. The reusable workflows have their own tests in
  github-workflows.
- This repo's own workflows (privacy, secret scan, changelog, labels,
  release, actionlint) call github-workflows at the pinned release, like
  an app repo.
- Copier questions are an interface: add with defaults (minor), don't
  rename or remove without a Breaking fragment. The workflow interface is
  github-workflows' (its README, "Versions and pinning").
