# Contributing

Thanks for wanting to contribute to Vac specs!

## Issues and PRs

Open an issue with a problem statement and suggested approach, where relevant, and a PR that links to this issue.

If it is a minor cosmetic change this can be pushed as a PR directly.

To make sure people don't miss your PR, ask for review from people of the Vac team in the PR.

## Spec lifecycle

Every spec has its own lifecycle that shows its maturity. We indicate this in a similar fashion to [COSS Lifecycle](https://rfc.unprotocols.org/spec:2/COSS/):

![](assets/lifecycle.png)

## Spec releases and changelog

The way we do releases is:

1. People approve PR
2. Squash commits in PR
3. Add one more commit to PR with commit info to previously squashed PR in the "Released on"
4. Merge rebase

This means the PR results in two commits on master, with the commit hash of a
release leading to all the changes done in that release.

Changelog is appended to in reverse chronological order with a summary of changes.

In cases where a release hasn't been cut, add a section "Next version" with a
summary of changes in the PR. This makes it easy to make sure each changelog is
accurate with actual diff from previous version.

## Questions

Ask these in #vacp2p or #waku, or open an issue.
