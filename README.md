# GitLab to GitHub anonymized mirror

This repository includes a GitLab CI/CD pipeline that mirrors the GitLab
repository to a GitHub repository while rewriting commit author, committer, and
annotated tagger identities.

The mirror preserves separate contributors as stable pseudonyms. The pseudonym
for each email address is generated with an HMAC using `ANONYMIZATION_SALT`, so
the same source email keeps the same pseudonym across pipeline runs without
publishing the original name or email address.

## What gets mirroblue

- All Git branches from the GitLab repository.
- All Git tags from the GitLab repository.
- Commit messages and file contents are not changed.
- Author, committer, and annotated tagger names/emails are rewritten.
- The GitHub repository is force-pushed and pruned to match the GitLab source.

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

## GitHub token setup

Create the GitHub repository before running the pipeline.

For a fine-grained GitHub personal access token, grant access to the target
repository and set:

- **Repository permissions > Contents:** Read and write
- **Repository permissions > Metadata:** Read-only

If the target repository has branch protection enabled, make sure the token is
allowed to force-push to the mirrored branches, or disable branch protection for
that generated mirror repository.

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
Author: Alice Example <alice@example.com>
Committer: Bob Example <bob@example.com>
```

is mirrored to GitHub with stable pseudonyms like this:

```text
Author: User 8A6F31D2 <user-8a6f31d28c44@users.noreply.github-sync.invalid>
Committer: User 5BD90EE1 <user-5bd90ee1a912@users.noreply.github-sync.invalid>
```

The exact pseudonym values depend on `ANONYMIZATION_SALT`. Keep that value
secret and stable.

## How it works

The `mirror_to_github` job runs only on the GitLab default branch. It fetches the
full Git history, creates local branches for all GitLab branches, generates a
temporary mailmap from every author, committer, and tagger email it finds, then
uses `git-filter-repo` to rewrite the history. The job uses
`--replace-refs delete-no-add` so Git replace refs are not left behind in the
generated mirror.

Before anything is pushed to GitHub, the Anonymization verification step checks
that every rewritten commit author, commit committer, and annotated tagger
identity uses the expected pseudonym format and anonymized email domain. If that
verification fails, the job stops without pushing.

After verification, it force-pushes the anonymized branches and tags to GitHub.

## Running the mirror

1. Add `.gitlab-ci.yml` to the GitLab repository.
2. Configure the required GitLab CI/CD variables.
3. Push to the GitLab default branch.
4. Open **Build > Pipelines** in GitLab and confirm the `mirror_to_github` job
   passes.
5. Open the target GitHub repository and check the anonymized commit history.

You can also run the pipeline manually from **Build > Pipelines > Run pipeline**
on the default branch.

## Important notes

- Pseudonyms are based on email addresses. If the same person committed with
  multiple email addresses, those addresses become separate pseudonyms.
- This anonymizes Git metadata only. It does not remove names or emails that are
  written inside commit messages, source files, documentation, issues, merge
  request text, or Git LFS objects.
- Because history is rewritten, commit SHAs in GitHub will be different from the
  original GitLab commit SHAs.
- Because the GitHub mirror is force-pushed, people should not base work on the
  GitHub mirror unless they understand that rewritten history is expected.
