# Fully automated pull request processing

This repo demonstrates how to create, automatically approve and merge pull
requests in Github without human intervention (except for starting the whole
process).

## Wait, but why?

Sometimes it is desired to have a possibility to automate a process that does
changes to code, while keeping mandatory code reviews for the rest of PRs.
For example, at my company we have decided to give our translations team
a button that would update site translations from [Phrase](https://phrase.com)
without requiring developer's attention. Another use-case could be
automatically accepting dependabot's suggestions (though I personally
considering this usage in this scenario more harmful than having the standard
review).

## Prerequisites

This workflow relies on master branch being protected as follows:

- At least one code review is required
- A required protection check is configured
  - either as a github workflow named 'CI'
  - or using Travis CI (or other CI tool, but that's untested)

Otherwise, you would need to tweak the
[automerge](https://github.com/marketplace/actions/auto-merge-pull-request)
action to allow merging into unprotected branches.

## Setup

For demonstration purposes this repository consists of trivial `Rakefile` that
pretends to do some testing job by sleeping for some time.

Travis CI is used as a build system because I needed to test handling of
`check_suite` event in github actions.

## Workflows

Three github workflows are orchestrated together in order to achieve the
automated PR process:

[auto_create_pr](.github/workflows/auto_create_pr.yml) provides a workflow
that could be triggered manually from `Actions` pane in github (or, with some
tweaking, via a webhook event). This workflow does some dummy changes to the
checkout, and then invokes
[create-pull-request](https://github.com/marketplace/actions/create-pull-request)
action. This workflow needs a `GH_REPO_TOKEN` secret which is a github
personal access token with `repo` or `public_repo` scope, depending on whether
you're automating in a private or a public repository. This token can not be
replaced with standard `GITHUB_SECRET` token, because using `GITHUB_SECRET`
token would prevent triggering other workflows, including CI builds.

[auto_approve_pr](.github/workflows/auto_approve_pr.yml) listens for PR
creation events and auto-approves once it sees matching PR. If this workflow
fail, you'll have to approve the PR manually (or, if you're the only
developer, than the only option would be to override branch protection rules
and merge the PR - as you cannot review your own PRs),
as I had not found a way to trigger
[automatic-pull-request-review](https://github.com/marketplace/actions/automatic-pull-request-review)
action for a specific PR

[auto_merge_pr](.github/workflows/auto_merge_pr.yml) is waiting for test suite
completion, reported either via `check_suite` event or completion of a github
workflow named `CI`. Once triggered, it checks if the branch name matches
the pattern allowed for auto-merge, and completes the merging process.
This workflow also uses `GH_REPO_TOKEN` secret to allow further workflows to
run (e.g. triggering autodeploy action).

It could be possible to use several access tokens for different steps, just
make sure to use access tokens that belong to different accounts for PR
creation and PR review step.
