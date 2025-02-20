# Label Pull Requests with Merge Conflicts Automatically

[![MIT](https://img.shields.io/github/license/prince-chrismc/label-merge-conflicts-action)](https://github.com/prince-chrismc/label-merge-conflicts-action/blob/main/LICENSE)

## Purpose

This action _intuitively_ checks open pull request(s) for merge conflicts and marks them with a [label][label-guide] and optionally leaves a
[comment][pr-comments] to alter the author.

[label-guide]: https://guides.github.com/features/issues/#filtering
[pr-comments]: https://docs.github.com/en/github/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/commenting-on-a-pull-request#about-pull-request-comments

## Add it to Your Project

### Create a Label

You'll need to decide on a label and manually [create it][create-label] if it does not already exist.

[create-label]: https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels#creating-a-label

### Setup a Workflow

```yml
name: Auto Label Conflicts
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  auto-label:
    runs-on: ubuntu-latest
    steps:
      - uses: goonstation/label-merge-conflicts-action@v3
        with:
          conflict_label_name: "has conflict"
          github_token: ${{ github.token }}

          # --- Optional Inputs ---
          # To make sure the merge commit exactly matches the branch
          detect_merge_changes: false # or true to handle as conflicts
          # By default a comment will be left, adding `conflict_comment: ''` will disable comments
          # The optional `${author}` will be replaced with the username of the pull request
          conflict_comment: |
            :wave: Hi, @${author},
            I detected conflicts against the base branch :speak_no_evil:
            You'll want to sync :arrows_counterclockwise: your branch with upstream!
```

## Limitations

1. GitHub does not reliably compute the `mergeable` status which is used by this action to detect merge conflicts.
If the base branch (e.g. `main`) changes then the mergeable status is unknown until someone (like this action) requests it.
[GitHub then tries to compute the status with an async job.](https://stackoverflow.com/a/30620973). This is usually quick
and simple, but there are no guarantees and GitHub might have issues. You can tweak `max_retries` and `wait_ms` to increase
the timeout before giving up on the Pull Request.
2. GitHub does not run actions on pull requests which have conflicts - which will prevent this action from working.
When there is a conflict it prevents the merge commit from being calculated.
[See this thread](https://github.community/t/run-actions-on-pull-requests-with-merge-conflicts/17104) for more details.
This is required for the [`mergeable`](https://docs.github.com/en/graphql/reference/enums#mergeablestate) as per the
[API documentation](https://docs.github.com/en/rest/reference/pulls#get-a-pull-request).

## FAQ - What are _Merge Changes_?

When [merging a pull request](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-request-merges), no matter the
[strategy](https://git-scm.com/docs/merge-strategies), there may _inadvertently be changes_ (i.e
[git blobs](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) which over lap and need to be recombined) which can have negative side effects. For example...

> I was working on an app with a friend and [...] I ran `git pull`. There were no merge conflicts, but _git added duplicate functions_ to a file after merge.
> I spent an hour trying to figure our what the problem was before realizing that **git had made a mistake** while merging.
> [ref](https://news.ycombinator.com/item?id=9871042)

## FAQ - How do I fix _"Resource not accessible by integration"_?

> ℹ️ _This is an evolving topic_. Feel free to [open an issue](https://github.com/prince-chrismc/label-merge-conflicts-action/issues/new?title=Question:%20Permissions&labels=help%20wanted) if there's any problems.

If a user without write access opens a 'Pull Request' from their fork then GitHub will not grant adequate permissions to set the labels
([detailed here](https://github.blog/changelog/2021-04-20-github-actions-control-permissions-for-github_token/#setting-permissions-in-the-workflow)).
Hence the _not accessible_ error. Try the following steps:

* using the ([potentially dangerous](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)) event `pull_request_target`
* setting the [workflow permissions](https://github.blog/changelog/2021-04-20-github-actions-control-permissions-for-github_token/)
by adding the minimum permission required to add labels or comments

  ```yml
  permissions:
    issues: write
    pull-requests: write
  ```

It boils down to the GitHub Action's permissions current behavior.
The [default permissions](https://docs.github.com/en/actions/reference/authentication-in-a-workflow#permissions-for-the-github_token) for any
[event type](https://docs.github.com/en/actions/reference/events-that-trigger-workflows) is
[`read` only](https://docs.github.com/en/actions/reference/authentication-in-a-workflow#permissions-for-the-github_token). The default can be
[adjusted for the repository](https://docs.github.com/en/github/administering-a-repository/disabling-or-limiting-github-actions-for-a-repository) or
[set for each workflow](https://github.blog/changelog/2021-04-20-github-actions-control-permissions-for-github_token/) explicitly. However, as per
the documentation...

> Pull requests from public forks are still considered a special case and will receive a read token regardless of these settings.

See [actions/labeler#12](https://github.com/actions/labeler/issues/12),
[actions/first-interaction#10](https://github.com/actions/first-interaction/issues/10),
and [Actions are severely limited](https://github.community/t/github-actions-are-severely-limited-on-prs/18179#M9249)
for more information and some history on the issue.
