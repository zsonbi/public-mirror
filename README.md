# GitLab to GitHub anonymized mirror

This repository includes a GitLab CI/CD pipeline that mirrors the GitLab
repository to a GitHub repository while rewriting private commit author,
committer, and annotated tagger identities.

By default, only identities whose email address ends with `@mptrdev.com` are
anonymized. Public contributor identities from other domains are preserved. Each
private email address is converted to a stable pseudonym generated with an HMAC
using `ANONYMIZATION_SALT`, so the same private email keeps the same pseudonym
across pipeline runs without publishing the original name or email address.

## What gets mirroblue

- All Git branches from the GitLab repository.
- All Git tags from the GitLab repository.
- Git notes from `refs/notes/*`, when present and safely retargetable.
- Commit messages and file contents are not changed.
- Author, committer, and annotated tagger names/emails ending with
  `@mptrdev.com` are rewritten.
- Author, committer, and annotated tagger names/emails from other domains are
  preserved.
- The GitHub repository is force-pushed and pruned to match the GitLab source.
- `.gitlab-ci.yml` and `anonymize_git_history.py` are removed from the GitHub
  mirror history.

Treat the GitHub repository as generated output. Do not make manual-only changes
there, because the next successful GitLab pipeline can overwrite them.

## Required GitLab CI/CD variables

Create these variables in GitLab under **Settings > CI/CD > Variables**:

| Variable | Example | Notes |
| --- | --- | --- |
| `GITHUB_USERNAME` | `octocat` | GitHub user or organization that owns the target repository. |
| `GITHUB_REPOSITORY` | `public-mirror` | Target GitHub repository name, without `.git`. |
| `GITHUB_TOKEN` | `github_pat_...` | GitHub token with read/write contents access to the target repository. Mark it as masked and protected when possible. |
| `ANONYMIZATION_SALT` | `use-a-long-random-secret-value` | Secret salt used to create stable pseudonyms. Mark it as masked and protected. Changing it changes every pseudonym. |

Optional variable:

| Variable | Default | Notes |
| --- | --- | --- |
| `ANONYMIZED_EMAIL_DOMAIN` | `users.noreply.github-sync.invalid` | Domain used for rewritten commit email addresses. |
| `ANONYMIZE_EMAIL_DOMAINS` | `mptrdev.com` | Comma-separated list of private email domains to anonymize. Do not include `@`. |

## GitHub token setup

Create the GitHub repository before running the pipeline.

For a fine-grained GitHub personal access token, grant access to the target
repository and set:

- **Repository permissions > Contents:** Read and write
- **Repository permissions > Metadata:** Read-only

If the target repository has branch protection enabled, make sure the token is
allowed to force-push to the mirrored branches, or disable branch protection for
that generated mirror repository.

## GitHub PR import setup

The GitHub mirror includes `.github/workflows/import-github-pr-to-gitlab.yml`.
When a GitHub pull request is merged, this workflow imports the PR commits into a
GitLab branch and opens or updates a GitLab merge request. GitLab remains the
canonical repository: merge the generated GitLab MR, then the GitLab mirror
publishes the accepted change back to GitHub.

Configure these GitHub Actions secrets in the GitHub mirror repository:

| Secret | Example | Notes |
| --- | --- | --- |
| `GITLAB_TOKEN` | `glpat-...` | GitLab token with `api` and `write_repository` access to the source GitLab project. |
| `GITLAB_PROJECT_PATH` | `my-group/my-project` | GitLab project path used for clone and API calls. |
| `GITLAB_URL` | `https://gitlab.com` | Optional. Defaults to `https://gitlab.com`. |
| `GITLAB_TARGET_BRANCH` | `main` | Optional. Defaults to the GitHub PR base branch name. |

The workflow uses `pull_request_target` only after a PR is merged. It does not
run code from the pull request. It fetches the PR commits, builds a patch series,
applies that series to a GitLab branch named `github-pr-<number>`, pushes that
branch to GitLab, and opens or updates a GitLab MR.

The import workflow refuses automatic imports that change `.gitlab-ci.yml` or
`anonymize_git_history.py`, because those files are GitLab-only mirror
automation. Handle those changes directly in GitLab.

## Example configuration

For this target GitHub repository:

```text
https://github.com/octocat/public-mirror
```

set these GitLab CI/CD variables:

```text
GITHUB_USERNAME=octocat
GITHUB_REPOSITORY=public-mirror
GITHUB_TOKEN=github_pat_XXXXXXXXXXXXXXXX
ANONYMIZATION_SALT=9c25f0d8b31b4f6b9bb2a0cdb5b8e2df
```

With the default email domain, an original commit like this:

```text
Author: Alice Internal <alice@mptrdev.com>
Committer: Bob Internal <bob@mptrdev.com>
```

is mirrored to GitHub with stable pseudonyms like this:

```text
Author: User 8A6F31D2 <user-8a6f31d28c44@users.noreply.github-sync.invalid>
Committer: User 5BD90EE1 <user-5bd90ee1a912@users.noreply.github-sync.invalid>
```

An original public contribution like this:

```text
Author: Jane Contributor <jane@users.noreply.github.com>
Committer: Jane Contributor <jane@users.noreply.github.com>
```

is preserved as:

```text
Author: Jane Contributor <jane@users.noreply.github.com>
Committer: Jane Contributor <jane@users.noreply.github.com>
```

The exact pseudonym values depend on `ANONYMIZATION_SALT`. Keep that value
secret and stable.

## How it works

The `mirror_to_github` job can run from any GitLab branch. It fetches the full
Git history, tags, and Git notes, creates local branches for all GitLab
branches, then runs `anonymize_git_history.py`.

The helper script generates a temporary mailmap for identities whose email
addresses end with the domains listed in `ANONYMIZE_EMAIL_DOMAINS`, then uses
`git-filter-repo` to rewrite the history. It also removes `.gitlab-ci.yml` and
`anonymize_git_history.py` from the rewritten mirror history so the GitHub
repository does not receive the GitLab-only mirror automation. The job uses
`--replace-refs delete-no-add` so Git replace refs are not left behind in the
generated mirror.

Git notes are exported before the rewrite and restored after the rewrite onto
the new object IDs when a safe mapping exists. Notes whose target objects were
removed from the mirrored history are skipped.

Before anything is pushed to GitHub, the Anonymization verification step checks
that no private-domain commit author, commit committer, or annotated tagger
email remains in the rewritten history. It also checks that generated pseudonyms
use the expected format and anonymized email domain. If verification fails, the
job stops without pushing.

After verification, it force-pushes the anonymized branches and tags to GitHub.
It also force-pushes `refs/notes/*` when Git notes are present.

## Running the mirror

1. Add `.gitlab-ci.yml` to the GitLab repository.
2. Configure the required GitLab CI/CD variables.
3. Push to the GitLab repository.
4. Open **Build > Pipelines** in GitLab and confirm the `mirror_to_github` job
   passes.
5. Open the target GitHub repository and check the anonymized commit history.

You can also run the pipeline manually from **Build > Pipelines > Run pipeline**.

## Handling GitHub contributions

1. A contributor opens a pull request against the GitHub mirror.
2. Review and merge the PR on GitHub.
3. The GitHub workflow creates or updates a GitLab merge request from the PR
   commits.
4. Review and merge the generated MR in GitLab.
5. The GitLab mirror rewrites private `@mptrdev.com` identities, preserves public
   contributor identities, and force-pushes the canonical result back to GitHub.

If the import workflow cannot apply the PR cleanly to GitLab, it fails without
pushing to GitLab. Import the contribution manually in GitLab in that case.

## Example mirrored contents

If the GitLab repository contains:

```text
.gitlab-ci.yml
anonymize_git_history.py
README.md
src/app.py
```

the GitHub mirror contains:

```text
README.md
src/app.py
```

The automation files stay in GitLab only.

## Important notes

- Pseudonyms are based on private email addresses. If the same person committed
  with multiple private email addresses, those addresses become separate
  pseudonyms.
- This anonymizes Git metadata only. It does not remove names or emails that are
  written inside commit messages, source files, documentation, issues, merge
  request text, or Git LFS objects.
- Because history is rewritten, commit SHAs in GitHub will be different from the
  original GitLab commit SHAs.
- Because the GitHub mirror is force-pushed, people should not base work on the
  GitHub mirror unless they understand that rewritten history is expected.
- GitLab-only metadata such as issues, merge requests, CI pipeline history,
  comments, project settings, and GitLab releases are not Git refs. They are not
  mirrored by this pipeline.
