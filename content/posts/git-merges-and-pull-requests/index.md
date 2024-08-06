+++
title = "How git merge works with GitHub pull requests"
description = "Keep feature branches up to date before merging a pull request"
authors = ["Victor Lyuboslavsky"]
image = "git-merges-and-pull-requests-feature.png"
date = 2024-06-26
categories = ["Software Development"]
tags = ["git", "GitHub", "Pull Request"]
draft = false
+++

This article covers how `git merge` works with GitHub pull requests. We will focus on the use case where developers want
to keep their feature branches updated with the main branch. After completing the feature work, developers create a pull
request to merge their feature branch into the main branch.

- [git merge](#git-merge)
- [Pull request after a merge](#pull-request-after-merge)
- [Updating a protected feature branch with a pull request](#update-protected-branch)

## What is a merge in version control?

Git is a distributed version control system that allows multiple developers to work on the same codebase. When
developers work on different branches, they must merge their changes into the main branch. A merge is the process of
combining changes from one branch into another branch, resulting in a single branch that contains the changes from both
branches.

## git merge {#git-merge}

The standard `git merge` command takes each commit from one branch and applies it to another. The final commit has two
parent commits: one from the current branch and one from the merged branch.

In the following example, we have a `branch` that we want to merge into the `main` branch:

{{< figure src="git-merge-two-branches-1.svg" title="git merge of two branches before merge"
alt="git merge of two branches before merge" >}}

The `git log` of the `main` branch shows the commit history:

```
commit e493ac8fea4e0efe125a561b9014313bec41a489 (HEAD -> main, origin/main)
Author: Victor on Software <>
Date:   Sat Jun 22 17:31:12 2024 -0500

    m3

commit b79e810cb86405061dc979ce4fc05fe36a724256
Author: Victor on Software <>
Date:   Sat Jun 22 17:29:41 2024 -0500

    m2

commit eaccad9476b472dbfb3cdfbd17088425be75b7b1
Author: Victor on Software <>
Date:   Sat Jun 22 17:26:55 2024 -0500

    m1

commit 4265c0a30b7b6f03d93331bc112261393c97ee1d
Author: Victor on Software <>
Date:   Sat Jun 22 17:25:40 2024 -0500

    first commit
```

And the `git log` of the `branch` shows the commit history:

```
commit 2afb078875a84095327ab2ef7c83711534c5eef8 (HEAD -> branch, origin/branch)
Author: Victor on Software <>
Date:   Sat Jun 22 17:30:20 2024 -0500

    b2

commit 9fc53c2ca9a58637a5d433de4c6150b832d4d275
Author: Victor on Software <>
Date:   Sat Jun 22 17:28:25 2024 -0500

    b1

commit eaccad9476b472dbfb3cdfbd17088425be75b7b1
Author: Victor on Software <>
Date:   Sat Jun 22 17:26:55 2024 -0500

    m1

commit 4265c0a30b7b6f03d93331bc112261393c97ee1d
Author: Victor on Software <>
Date:   Sat Jun 22 17:25:40 2024 -0500

    first commit
```

We merge the `branch` into the `main` branch:

```shell
git checkout main
git merge branch
```

The resulting commit history shows all the commits from both branched as well as the final empty merge commit pointing
to the two parent commits: `e493ac8 2afb078`:

```
commit 09d569bd079162643462dde112246f4167f14889 (HEAD -> main)
Merge: e493ac8 2afb078
Author: Victor on Software <>
Date:   Sat Jun 22 21:03:01 2024 -0500

    Merge branch 'branch'

commit e493ac8fea4e0efe125a561b9014313bec41a489 (origin/main)
Author: Victor on Software <>
Date:   Sat Jun 22 17:31:12 2024 -0500

    m3

commit 2afb078875a84095327ab2ef7c83711534c5eef8 (origin/branch, branch)
Author: Victor on Software <>
Date:   Sat Jun 22 17:30:20 2024 -0500

    b2

commit b79e810cb86405061dc979ce4fc05fe36a724256
Author: Victor on Software <>
Date:   Sat Jun 22 17:29:41 2024 -0500

    m2

commit 9fc53c2ca9a58637a5d433de4c6150b832d4d275
Author: Victor on Software <>
Date:   Sat Jun 22 17:28:25 2024 -0500

    b1

commit eaccad9476b472dbfb3cdfbd17088425be75b7b1
Author: Victor on Software <>
Date:   Sat Jun 22 17:26:55 2024 -0500

    m1

commit 4265c0a30b7b6f03d93331bc112261393c97ee1d
Author: Victor on Software <>
Date:   Sat Jun 22 17:25:40 2024 -0500

    first commit
```

{{< figure src="git-merge-two-branches-2.svg" title="git merge of two branches after merge" alt="git merge of two branches after merge" >}}

The result would be the same if **instead** we merged the `main` branch into the `branch` branch:

```shell
git checkout branch
git merge main
```

except the final merge commit would be slightly different:

```
commit 06713a38ed38c12f599c6e810ee50d4cacfe2de7 (HEAD -> branch)
Merge: 2afb078 e493ac8
Author: Victor on Software <>
Date:   Sat Jun 22 17:32:01 2024 -0500

    Merge branch 'main' into branch
```

The merged changes on `branch` can be pushed to the remote repository without issues because the remote branch can be
fast-forwarded to the new commit.

```shell
git push origin branch
```

## What is a fast-forward merge?

A fast-forward merge is a merge where the base branch (target branch) has no new commits. In this case, git moves the
target branch to the commit of the source branch. It is a fast-forward merge because the target branch is moved forward
to the new commit.

A fast-forward merge does not lose any history -- it is always possible to undo a fast-forward merge.

`git push` does not, by default, allow a merge that is not a fast-forward. Use the'- force' option to enable a merge
that is not a fast-forward.

## Undo a git merge

The above merge can be undone by resetting the branch to the commit before the merge, which is one of the parent commits
of the merge commit.

This command resets the `branch` to the commit before the merge:

```shell
git reset --hard 2afb078875a84095327ab2ef7c83711534c5eef8
```

## git rebase

Another way to combine changes from one branch into another is to use `git rebase`. This command applies the changes
from the source branch to the target branch by reapplying the commits from the source branch to the target branch.

The `git rebase` command will modify the commit history of the source branch. In our pull request examples, we will use
`git merge` instead of `git rebase` to preserve all the commit histories.

## Pull request after a merge {#pull-request-after-merge}

When working on a feature branch, developers often want to update their branch with the latest changes from the main to
make sure their feature works with the newest code. We start this process with the above-described `git merge` command,
where we merge the `main` branch into the `branch` branch.

After the merge, the developer can create a GitHub pull request to merge the `branch` into the `main` branch.

{{< figure src="github-pull-request.png" title="GitHub pull request after merge" alt="GitHub pull request after merge" >}}

Note that the commit history only shows the commits from the `branch` and the merge commit. The `main` commits are not
shown in the pull request.

GitHub shows a few options for merging the pull request:

{{< figure src="merge-pull-request-options.png" title="Merge pull request" alt="Merge pull request" >}}

- **Create a merge commit**: This option creates a new merge commit combining the changes from the `branch` and the
  `main` branches. This is the default option.
- **Squash and merge**: This option combines all the commits from the `branch` into a single commit and merges that
  commit into the `main` branch.
- **Rebase and merge**: This option applies the changes from the `branch` onto the `main` branch by rebasing the commits
  from the `branch` onto the `main` branch.

Selecting **Create a merge commit** results in the following commit history:

{{< figure src="commit-history-after-pull-request.png" title="Commit history after pull request" alt="Commit history after pull request" >}}

The last two commits are both merge commits.

```
commit 1eb0af8c0e9ad16a0267d8abd1ce667f125ab7e8 (HEAD -> main, origin/main)
Merge: e493ac8 22d2107
Author: Victor Lyuboslavsky <******>
Date:   Sun Jun 23 07:35:57 2024 -0500

    Merge pull request #1 from getvictor/branch

    My pull request

commit 22d2107b49bca56e67b7d4e800d93f93378a0956 (origin/branch, branch)
Merge: 2afb078 e493ac8
Author: Victor on Software <>
Date:   Sat Jun 22 21:25:39 2024 -0500

    Merge branch 'main' into branch
```

The PR merge commit points to the previous merge commit and the last commit on `main`.

{{< figure src="pull-request-commit-history.svg" title="Diagram of commit history after pull request"
alt="Diagram of commit history after pull request" >}}

## Updating a protected feature branch {#update-protected-branch}

We have two branches in this example: `main` and `feature`. Both branches are protected, meaning that changes to them
must be made through a pull request. We want to update the `feature` branch with the latest changes from the `main`
branch.

We can do this by merging the `main` branch into the `feature` branch, creating a new branch, and creating a pull
request.

```shell
git checkout feature
git merge main
git checkout -b feature-update
git push origin feature-update
```

And create a pull request to merge `feature-update` into `feature`.

{{< figure src="github-pull-request-branches.png" title="Create a PR to merge into feature branch"
alt="Create a PR to merge into feature branch" >}}

This pull request shows all the commits from the `main` branch and the merge commit. This commit history is problematic
because the PR may trigger a code review from the
[code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
of the files that were already reviewed in previous pull requests to the `main` branch.

{{< figure src="github-pull-request-commits.png" title="Commits from feature-update branch" alt="Commits from feature-update branch" >}}

After the merge, the `feature` branch commit history looks like:

{{< figure src="commit-history-of-feature-after-pr.png" title="Commit history of feature branch after PR"
alt="Commit history of feature branch after PR" >}}

Now, we create a pull request to merge the update `feature` branch into the `main` branch.

{{< figure src="github-pull-request-feature.png" title="PR to merge feature branch into main" alt="PR to merge feature branch into main" >}}

After the merge, the `main` branch commit history looks like:

{{< figure src="commit-history-of-main-after-feature-pr.png" title="Commit history of main after PR from feature branch"
alt="Commit history of main after PR from feature branch" >}}

The last three commits are merge commits.

```
commit 59caaf1cc5103099f850c32f1729c5ffe3525404 (HEAD -> main, origin/main)
Merge: e493ac8 dcbc117
Author: Victor Lyuboslavsky <******>
Date:   Sun Jun 23 08:56:53 2024 -0500

    Merge pull request #3 from getvictor/feature

    Feature

commit dcbc117e5811e683f1074d947bc25da21b5fa5f6 (origin/feature)
Merge: 2afb078 373dd82
Author: Victor Lyuboslavsky <******>
Date:   Sun Jun 23 08:45:44 2024 -0500

    Merge pull request #2 from getvictor/feature-update

    Feature update

commit 373dd82672a4879bfcf3b29c4feb97004359adfe (origin/feature-update, feature-update, feature)
Merge: 2afb078 e493ac8
Author: Victor on Software <>
Date:   Sun Jun 23 08:03:22 2024 -0500

    Merge branch 'main' into feature
```

{{< figure src="pull-request-commit-history-2.svg" title="Diagram of commit history after two pull requests"
alt="Diagram of commit history after two pull requests" >}}

### Merging a pull request with **Squash and merge**

If the final pull request is merged with **Squash and merge**, the commit history will look like:

{{< figure src="commit-history-after-squash-and-merge.png" title="Commit history of main after squash and merge"
alt="Commit history of main after squash and merge" >}}

The last commit is a single commit that combines all the changes from the `feature` branch. The merge commits and all
other commits are eliminated.

The downside of **Squash and merge** is that the commit history is lost. The commit history is useful for debugging,
understanding the changes made, and keeping ownership of the changes when multiple developers work on the same feature
branch.

## Further reading

Recently, we explained
[how to find the minimum required code owner approvers for a pull request](../find-code-owners-for-pull-request).

## Watch how git merge works with GitHub pull requests

{{< youtube Djpc7ymvuzU >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
