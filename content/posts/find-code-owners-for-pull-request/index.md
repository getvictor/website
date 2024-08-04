+++
title = 'Find required code owner approvers for a PR in 3 steps'
description = "How to find the minimum required reviewers who need to approve a GitHub pull request"
authors = ["Victor Lyuboslavsky"]
image = "codeowners-headline.png"
date = 2024-07-31
categories = ["Software Development"]
tags = ["Pull Request", "GitHub"]
draft = true
+++

- [Find code owners from the command line](#find-code-owners-from-command-line)

## What are GitHub code owners?

GitHub
[CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
is a file that defines the individuals or teams responsible for code in a repository. When a user creates a pull
request, GitHub uses the `CODEOWNERS` file to suggest the appropriate reviewers for the pull request. This process helps
ensure that the right people review the code changes.

Repository owners can enable branch protection rules that require the code owner of each changed file to approve the
pull request.

## The problem: too many files and too many code owners

It can be challenging to determine who needs to review a pull request in a large repository with many files and many
code owners. This challenge is especially true when the pull request touches many files.

Many times, I've asked another engineer to approve my PR, and they approved it, but GitHub said that the PR still needed
approval from another code owner. I needed another approval because my PR changed another file with another code owner,
and I didn't know about it.

## Steps to find the minimum required code owners

To find the minimum required code owners for a pull request, we can use these steps:

1. Get the list of changed files in the pull request.
2. For each changed file, get the list of code owners.
3. Find the minimum set of code owners that covers all the above lists.

## Finding the code owners manually

The above steps can be done manually by opening the pull request in GitHub and hovering over the blue CODEOWNERS icon
for each changed file to see the code owners. However, this can be time-consuming and error-prone.

{{< figure src="codeowners-hover.png" title="See the code owners on hover" alt="List of files, with code owners showing when hovering over the blue icon next to the file" >}}

## Finding the code owners from the command line {#find-code-owners-from-command-line}

To automate the above steps, we will need the following prerequisites:

- [GitHub CLI](https://cli.github.com/) installed and logged in
- Command-line JSON processor [jq](https://jqlang.github.io/jq/download/) installed
- A CODEOWNERS parser installed, such as https://github.com/hmarr/codeowners

From the top directory containing your git repository, run the following:

```bash
gh pr view $MY_PR_NUMBER --json files | jq -r '.files[] .path' \
| xargs codeowners | tr -s ' ' | cut -f2- -d ' ' | sort -u
```

Where `$MY_PR_NUMBER` is the number of your pull request.

The first part of the command, `gh pr view $MY_PR_NUMBER --json files`, gets the list of changed files in the pull
request in JSON format. The second part, `jq -r '.files[] .path'`, extracts the file paths from the JSON. The third
part, `xargs codeowners`, runs the `codeowners` command for each file. The fourth optional part, `tr -s ' '`, removes
extra spaces. The fifth part, `cut -f2- -d ' '`, removes the first column. The last part, `sort -u`, sorts and removes
duplicates.

The output will be the list of code owners for the changed files in the pull request. For example:

```
(unowned)
@fleetdm/go
@getvictor @lucasmrod @roperzh @mostlikelee
@lucasmrod @getvictor @jacobshandling
@roperzh @gillespi314 @lucasmrod @getvictor
```

At this point, we can do additional processing, such as excluding some code owners.

By visually inspecting the output, we can determine the minimum set of code owners that need to review the pull request.

## Further reading

Recently, we wrote [how merges work with GitHub pull requests](../git-merges-and-pull-requests).

Previously, we explained
[how to create reusable workflows and steps in GitHub Actions](../github-reusable-workflows-and-steps).

## Watch how to find the minimum required code owners for a pull request

{{< youtube RHW-ZELnuSg >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
