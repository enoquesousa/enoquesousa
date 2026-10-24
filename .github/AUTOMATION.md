# Contribution animation automation

This repository is a static GitHub profile. It has no application login,
password store, session, or API server. Authentication is used only by
`.github/workflows/snake.yml` to read contribution data and maintain the generated
SVG branch.

## Flow and token permissions

| Job | Runs when | Token access | Result |
| --- | --- | --- | --- |
| `generate` | Daily schedule, manual dispatch, or a PR changing this workflow/document | `contents: read`; other configurable permissions are not granted | Queries the owner's contribution calendar and uploads three SVGs as a GitHub artifact |
| `publish` | Successful generation from a scheduled/manual run on `refs/heads/main` | `contents: write`; other configurable permissions are not granted | Downloads that run's artifact and updates `output` |

Pull requests generate a reviewable artifact and skip publication. A manual run
on another branch also skips publication. The workflow has no
`pull_request_target` or `workflow_run` path that could promote a PR artifact to
a write-capable job. Publication uses the artifact ID returned by its own
generation job and requires digest verification.

The generator uses the SVG-only variant of Platane/snk, pinned to a reviewed
commit. It accepts the automatic `github.token` input and calls GitHub's GraphQL
API for the contribution calendar. It does not receive the publisher job's
write token. The three existing filenames, palettes, and daily UTC schedule
remain the workflow's output contract.

The publisher uses its job's automatic `GITHUB_TOKEN`. It makes incremental
commits to the configured `output` branch, avoids empty commits, and does not
force-push. Existing files outside the three generated SVGs are retained by
this publishing mode. `contents: write` is a repository permission, not a
credential cryptographically restricted to the `output` branch; the fixed
destination, event condition, pinned action, and repository protection rules
remain relevant controls.

## Credential handling and trust

No personal access token or long-lived deployment secret is needed. GitHub
creates a token for each job, with that job's permissions. Actions can access
the token through the `github.token` context even without an explicit input,
so permission separation must happen between jobs, not merely by moving an
environment variable between steps. See
[GitHub's token documentation](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token).

Neither job needs a checkout of this profile repository. The publisher creates
a temporary Git repository and can store an authenticated remote URL in its
temporary `.git/config`. Do not treat the token as memory-only. That directory
is not part of the artifact: the upload explicitly selects only three SVG
files. Do not enable verbose credential debugging or upload the job workspace.

All actions are pinned by full commit SHA. Generation and publication use
Node 24 variants. Pinning prevents a moved version tag from silently changing
the code, but the pinned third-party actions are still trusted code and must
be reviewed when updated. The SVGs are public profile output, not a place for
secrets or private repository metadata.

## Verification and changes

The [baseline scheduled run on 2026-10-24](https://github.com/enoquesousa/enoquesousa/actions/runs/40042928108)
used one job with broad token permissions for both generation and publication;
its log includes `Contents: write` and `Actions: write`. The split limits
generation to read access and grants write access only to the publishing job.

Workflow changes should pass `actionlint` and a normal PR run. That PR run must
generate all three nonempty SVGs, retain the artifact, and mark `publish` as
skipped. It must not move the `output` branch. Testing a PR does not require a
manual dispatch or an actual publication.

Updating this file or merging the PR does not itself publish the SVGs; there
is no push trigger. Scheduled/manual publication uses the workflow present on
the selected branch and the condition above. Branch-protection settings are
managed separately from this file.
