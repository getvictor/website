+++
title = "Top 3 issues with GitHub code review process"
description = "GitHub code review process is not scalable and provides a poor developer experience"
authors = ["Victor Lyuboslavsky"]
image = "developer-on-tightrope-headline.png"
date = 2024-09-15
categories = ["Software Development"]
tags = ["GitHub", "Developer Experience"]
draft = false
+++

- [CODEOWNERS is not scalable](#issue-1-codeowners-is-not-scalable)
- [Re-approvals for every push](#issue-2-re-approvals-for-every-push)
- [Impractical for protected feature branches](#issue-3-impractical-to-maintain-a-protected-feature-branch)

Our team has been using GitHub to review the code for our open-source product. We have encountered several issues with
GitHub code reviews. The default GitHub code review process is not scalable and provides a poor developer experience.

## How to set up a GitHub code review process

GitHub admins can create a
[branch protection rule](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule)
that requires a code review before merging the code to the main branch.

Here's a representative branch protection rule:

{{< figure src="pr-branch-protection-rule.png" title="PR branch protection rule" alt="Branch protection rule requiring code review before merging" >}}

When a developer creates a pull request, GitHub requires code reviews from all relevant owners specified by the
`CODEOWNERS` file in the repository. If someone makes a new push to the PR, all the owners need to re-approve the PR.

In a previous article, we covered [how to find the required code owners for a PR](../find-code-owners-for-pull-request).
This is another issue, but we will not discuss it in this article.

## Issue 1: CODEOWNERS is not scalable

GitHub uses a
[`CODEOWNERS` file to define individuals or teams responsible for each file in the repository](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners).

The `CODEOWNERS` file format favors fine-grained code ownership, where the last matching pattern takes precedence over
previous patterns. Here is an example from the GitHub documentation:

```plaintext
# In this example, @octocat owns any file in the `/apps`
# directory in the root of your repository except for the `/apps/github`
# subdirectory, as this subdirectory has its own owner @doctocat
/apps/ @octocat
/apps/github @doctocat
```

The `CODEOWNERS` file is not scalable for medium-to-large and even small organizations. As the number of code owners
grows, each pull request is likely to require approval from more code owners. Each code owner may request changes,
potentially leading to cycles and cycles of re-approvals.

Tracking down multiple people to approve and re-approve a PR can be time-consuming and frustrating for developers. This
results in longer PR turnaround times, slower development velocity, and missed commitments.

From a developer experience perspective, we want to make the code review process as smooth and efficient as possible,
which means one reviewer for one PR. This approach is feasible by manually inverting the
`last matching pattern takes precedence` rule in the `CODEOWNERS` file by always including the owner(s) from the
previous pattern. For example, we would rewrite the above owners as:

```plaintext
/apps/ @octocat
/apps/github @octocat @doctocat
```

Keeping the `CODEOWNERS` file in this format may be cumbersome to do manually, but it can be done with a script.

## Issue 2: Re-approvals for every push

When a developer makes a new push to a PR, all the code owners need to re-approve it. This is a poor developer
experience, as it requires the code owners to review potentially the same code changes multiple times.

The issue stems from the lack of fine-grained control over the following option:

{{< figure src="dismiss-stale-pr-approvals.png" alt="Dismiss stale pull request approvals when new commits are pushed" >}}

With multiple code owners, every code owner must re-approve every change.

A code owner should not need to re-review code that didn't change -- this is a waste of time and effort.

With a single code owner, the reviewer must re-approve trivial or irrelevant changes, such as:

- fixing a typo in a comment
- fully accepting a suggestion from the reviewer
- re-generating an auto-generated file, such as documentation

The required re-approvals can be frustrating and time-consuming for developers and code owners. They make developers
feel untrusted and inefficient.

The main argument for requiring re-approvals is security—we don't want to merge potentially malicious code. If that's
the case, we should have a security review process in place, not a code review process. A security review can be done by
a separate individual and improved by automated tools.

In addition, we should be able to completely exclude some files/directories from the code review process. For example,
generated files, such as documentation based on code changes, should not require code review. Other generated files,
such as testing mocks, may have CI/CD checks that ensure they are generated correctly, and they should not require code
review either.

## Issue 3: Impractical to maintain a protected feature branch

A protected feature branch requires code reviews before merging. Since all the commits on the feature branch have
already been reviewed and approved, it is considered safe to merge into the main branch.

The main issue is that the developer cannot simply update this feature branch with the latest changes on the main
branch. They need PR approval from all the code owners who have already approved the same changes on the main branch.
This busy work is another example of a waste of time and effort.

In addition, a feature branch may be long-lived and introduce changes across multiple areas of the code base. This means
that it may require approval from many code owners, which can be time-consuming and frustrating.

## Solution: Custom GitHub Action to manage code reviews

Instead of relying on the default GitHub code review process, we can create a custom GitHub Action to manage code
reviews. The custom GitHub Action can:

- automatically identify a single reviewer for a PR (or identify a small group of reviewers, each of whom can approve
  the PR)
- automatically exclude specific files/directories from the code review process
- automatically maintain the approval state of the PR when new commits meeting explicit criteria are pushed
- enable a usable and practical protected feature branch

Here is an example [GitHub Action to replace GitHub's pull request review process](../typescript-github-action/).

## Further reading

- **[Why transparency beats everything else in engineering](../engineering-transparency/)**  
  How making work visible transforms teams from frustrated to high-performing through organizational transparency.

- **[How to track engineering metrics with GitHub Actions](../track-engineering-metrics/)**  
  Implement automated tracking of engineering performance metrics using GitHub's built-in tools and APIs.

- **[How git merge works with PRs](../git-merges-and-pull-requests)**  
  Understand the mechanics behind pull request merges and how to optimize your git workflow.

- **[How to reuse GitHub workflows and steps](../github-reusable-workflows-and-steps)**  
  Build maintainable CI/CD pipelines by creating reusable components for your GitHub Actions.

- **[What is clean, readable code and why it matters?](../readable-code/)**  
  Discover why code readability directly impacts team productivity and how to measure it effectively.

- **[How to scale your codebase with evolutionary architecture](../scaling-codebase-evolutionary-architecture/)**  
  Learn architectural patterns that help teams maintain velocity as codebases and organizations grow.

- **[Set up a remote dev environment](../remote-development-environment)**  
  Configure development environments that enhance team collaboration and reduce onboarding friction.

## Watch the top 3 issues with GitHub code reviews

{{< youtube RWnJ84vTK48 >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
