# Changelog

Notable changes to this repository, newest first. Each PR adds a Changie
fragment under `.changes/unreleased/`; merging to `main` releases them.

## v0.5.0 - 2026-11-03

### Added

* language: node, for npm app repos such as an Angular frontend. CI calls github-workflows node.yaml (npm ci, then the package's lint, prettier --check, test and build scripts), with the Node version from .nvmrc unless node_version pins one, and node_lint / node_format_check to turn the first two off. A prettier hook using the repo's own prettier, node labels, editorconfig, .gitignore, Taskfile (test, format, lint) and the renovate/node.json5 preset.

### Changed

* Rendered workflow callers and Renovate presets: iT3E/github-workflows v1.1.0 to v1.2.0 (https://github.com/iT3E/github-workflows/releases/tag/v1.2.0).

## v0.4.0 - 2026-11-03

### Changed

* Rendered workflow callers and Renovate presets: iT3E/github-workflows v1.0.2 to v1.1.0 (https://github.com/iT3E/github-workflows/releases/tag/v1.1.0).
* Releases and template updates no longer need a hand step. The rendered release.yaml (repos with an image) runs on every push to main with auto-fragment "Dependency updates": a Renovate merge that changed what goes into the image releases a patch by itself, while template and CI-only updates release nothing. Template update PRs automerge once green (iT3E/github-workflows v1.1.0 presets). This repo releases a patch when a Renovate merge changes template/ or copier.yml.

## v0.3.1 - 2026-11-03

### Fixed

* Rendered workflow callers and Renovate presets: iT3E/github-workflows v1.0.0 to v1.0.2 (https://github.com/iT3E/github-workflows/releases/tag/v1.0.2).

## v0.3.0 - 2026-11-03

### Changed

* The reusable workflows and Renovate presets moved to iT3E/github-workflows (v1.0.0, same names, inputs and jobs), and this repo is now iT3E/app-repo-template with only the Copier template. Rendered callers pin iT3E/github-workflows and the rendered renovate.json5 extends its presets, both at the release in copier.yml's github_workflows. A github-workflows release now reaches app repos as a template release (Renovate bumps the pin here, and merging it releases the template).

## v0.2.1 - 2026-11-03

### Fixed

* renovate/default.json5: the renovate-config manager no longer opens its own PR for the #vX.Y.Z on the app-template presets; the copier update PR moves them.

## v0.2.0 - 2026-11-03

### Added

* image.yaml: postgres and postgres-version inputs start a throwaway Postgres service for test-command ($TEST_POSTGRES_URL). Copier question image_test_postgres turns it on, and task test starts a matching container on 127.0.0.1:55432.

## v0.1.1 - 2026-11-03

### Fixed

* Taskfile.yml: a blank line again between the template:update task and the next one.

## v0.1.0 - 2026-11-03

### Added

* First release: reusable workflows (Go, Python, image, privacy, secret scan, actionlint, changelog, labels, release), the Copier template for app repos, and Renovate presets.
