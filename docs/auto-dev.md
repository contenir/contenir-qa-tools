# Auto-dev

`.github/workflows/auto-dev.yml` is a reusable workflow that turns issues
into draft pull requests:

1. **Triage.** A new issue, or a comment on one, is read against the
   repository's code, docs and existing issues. The verdict is posted as a
   comment and a label: `triage:accepted`, `triage:needs-info`,
   `triage:needs-human` or `triage:rejected`. Nothing is closed
   automatically.
2. **Implement.** Accepted issues with low or medium risk get the
   `auto:implement` label, which builds the change on `auto/issue-N`. The
   workflow re-runs the check command itself and opens a draft PR only if it
   passes, with the maintainer requested for review. If it can't, the issue
   gets `auto:failed`, the check output and the attempted diff.
3. **Review.** Every auto-dev PR gets an independent review comment.
4. **Respond.** Mentioning `@claude` in a PR comment, inline comment or
   review asks for changes, which are pushed to the PR branch. On a closed
   or merged PR it only replies that the PR is closed; open an issue
   instead. On an issue, the mention re-runs triage with the comment as new
   input.

Only people with write, maintain or admin permission on the repository can
start triage from an issue or comment, or get a response. Labels work for
anyone who can apply them.

## Adding it to a repository

Create `.github/workflows/auto-dev.yml`:

```yaml
name: "Auto-dev"

on:
  issues:
    types: [opened, labeled]
  issue_comment:
    types: [created]
  pull_request:
    types: [opened, labeled]
  pull_request_review_comment:
    types: [created]
  pull_request_review:
    types: [submitted]

permissions:
  contents: read

jobs:
  auto-dev:
    uses: contenir/contenir-qa-tools/.github/workflows/auto-dev.yml@0.1.x
    secrets: inherit
```

Pass `with:` inputs only where the repository differs from the defaults:

| Input | Default | Use |
|---|---|---|
| `reviewers` | `simon-mundy` | Logins requested to review auto-dev PRs. |
| `base-branch` | default branch | Branch auto-dev PRs target. |
| `php-version` | `8.3` | PHP version for implementing and checking. |
| `php-extensions` | none | Extra extensions for setup-php. |
| `apt-packages` | none | Ubuntu packages the tests need, as in `continuous-integration.yml`. |
| `check-command` | `composer check` | Must pass before a PR is opened. |
| `guidance` | none | Repository-specific conventions, added to the Contenir conventions built into the workflow. |
| `branch-prefix` | `auto/issue-` | Prefix for auto-dev branches. |
| `commit-user-login` | `simon-mundy` | Account auto-dev commits as, by its noreply address. |
| `commit-user-id` | `46739456` | Numeric id of `commit-user-login`. |

The workflow takes effect once the caller is on the default branch, because
GitHub runs issue workflows from there.

## Organisation setup

These are shared by every repository and are already in place for the
contenir org:

- The `contenir-autodev` GitHub App, installed on all repositories with
  Contents, Issues and Pull requests read and write. Its token makes the
  workflow's PRs and pushes trigger CI and lets it request reviews.
- `vars.AUTODEV_APP_ID` and `secrets.AUTODEV_APP_PRIVATE_KEY` for the App.
- `secrets.CLAUDE_CODE_OAUTH_TOKEN`, a subscription token.

## Labels

Created on first use:

| Label | Meaning |
|---|---|
| `auto:triage` | Run or re-run triage, including on issues filed before auto-dev |
| `auto:implement` | Build it; added by triage, or by hand to overrule |
| `auto:skip` | Never triage this issue |
| `auto:review` | Run or re-run the review on an auto-dev PR |
| `auto:pr-open`, `auto:failed`, `auto:generated` | Status |
| `triage:*` | The latest triage verdict |

## AI attribution

The `attributions` CI job rejects PR text, commit messages and commit
identities that name the assistant. Auto-dev's commits, PR titles and PR
descriptions are written to pass it. Comments are not checked, so mentions in
comments are fine.

Every commit auto-dev makes, whether from the implementer, the `@claude`
responder or the draft-PR step, is authored and committed as
`commit-user-login` at its GitHub noreply address. The action would otherwise
commit as `claude[bot]`. The `auto/` branch, the `auto:generated` label and the
App that opens the PR still show the change came from auto-dev.
