# AGENTS.md

How to work in this repo: the app-repo template (README.md has the full
standard). It is public, and every app repo's CI runs code from it.

## Change workflow

- Every change goes through a PR, squash-merged, with every check green
  (`gh-merge-safe.sh <pr> iT3E/app-template`). Never push to `main`.
- A PR adds a Changie fragment (`changie new`) or carries `skip-changelog`.
  Merging a fragment tags a template release (`self-release.yaml`), and
  that tag is what app repos update to. A change that app repos should get
  needs a fragment.
- Reusable workflows (`.github/workflows/*.yaml` without `self-` or
  `template-ci`) are called by app repos pinned to a release commit. Treat
  their inputs, secrets and job names as an interface: adding an input
  with a default is a minor change, renaming or removing one is Breaking.
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
  or print the list.
- Before posting a PR body, comment or review, write it to a file, run
  `bin/privacy-check --text < file`, and post with `--body-file`.
- No secrets in any file. Workflows get them only as `secrets:` from the
  caller.

## Writing

Plain prose, short sentences. No emdashes (use a comma, colon or
parentheses). No AI-attribution trailers in commits.
