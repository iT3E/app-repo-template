# AGENTS.md

How to work in this repo: the Copier template for app repos (README.md has
the full standard). It is public, and every app repo is rendered from it.
The reusable workflows and Renovate presets those repos run live in
[iT3E/github-workflows](https://github.com/iT3E/github-workflows); change
them there.

## Change workflow

- Every change goes through a PR, squash-merged, with every check green
  (`gh-merge-safe.sh <pr> iT3E/app-repo-template`). Never push to `main`.
- A PR adds a Changie fragment (`changie new`) or carries `skip-changelog`.
  Merging a fragment tags a template release (`release.yaml`), and that
  tag is what app repos update to. A change that app repos should get
  needs a fragment.
- The github-workflows release the template uses is one value,
  `github_workflows` in `copier.yml` (`"vX.Y.Z@<the tag's commit>"`).
  This repo's own callers in `.github/workflows/` and its
  `.github/renovate.json5` use the same release; template CI fails if
  they differ. Renovate moves all of them in one PR, and merging it
  releases the template by itself.
- Copier questions are an interface: add with a default (minor), don't
  rename or remove without a Breaking fragment.
- Template files (`template/`) render with `[[ ]]`, `[% %]` and `[# #]`
  (copier.yml `_envops`), so `${{ }}` and `{{ }}` pass through as is. Keep
  copier.yml free of `_tasks`, `_migrations`, `_jinja_extensions` and
  `_external_data`: Renovate's copier manager refuses templates that need
  `--trust`.
- `task render` renders `tests/*.yml` into `out/`; `template-ci.yaml` does
  the same in CI and lints the result.

## Privacy

This repo is public. No family names, personal email addresses, store or
business names, household details, or anything else that says who is who
or where, in any file, path, commit message, PR or issue text, or comment.
Examples use `example-*` names.

- `bin/privacy-check` (a link to `template/bin/privacy-check`) checks the
  tnwks-ops denylist: CI reads the `PRIVACY_TERMS` secret, locally it reads
  `$PRIVACY_TERMS` or a tnwks-ops checkout at `../tnwks-ops`. Never commit
  or print the list. github-workflows has a copy; change both together.
- Before posting a PR body, comment or review, write it to a file, run
  `bin/privacy-check --text < file`, and post with `--body-file`.
- No secrets in any file. Workflows get them only as `secrets:` from the
  caller.

## Writing

Plain prose, short sentences. No emdashes (use a comma, colon or
parentheses). No AI-attribution trailers in commits.
