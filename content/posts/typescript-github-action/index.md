+++
title = "How to create a custom GitHub Action using TypeScript"
description = "Create a GitHub Action using TypeScript to replace GitHub's pull request review process"
authors = ["Victor Lyuboslavsky"]
image = "typescript-github-action-headline.png"
date = 2024-10-23
categories = ["DevOps & Infrastructure"]
tags = ["GitHub", "GitHub Actions", "Developer Experience"]
draft = false
+++

In this article, we'll create a custom reusable GitHub Action using TypeScript. As covered in our article on
[reusing GitHub workflows and steps](../github-reusable-workflows-and-steps/), GitHub Actions allow you to automate your
software development workflows. By creating a custom GitHub Action, you can extend the functionality of GitHub Actions
to suit your specific needs.

We will create a simple GitHub Action to replace
[GitHub's broken Pull Request review process](../github-code-review-issues/). This custom GitHub Action will
automatically approve Pull Requests that meet specific criteria.

## Start with GitHub's Action template

GitHub provides a
[template for creating a new GitHub Action using TypeScript](https://github.com/actions/typescript-action). You can use
this template to get started quickly. The template includes the necessary files and structure to create a new GitHub
Action. Anything unnecessary can be removed or modified to suit your requirements.

Follow the instructions in the template's README to set up your action.

1. Click the **Use this template** button at the top of the repository
1. Select **Create a new repository**
1. Select an owner and name for your new repository
1. Click **Create repository**
1. Clone your new repository

Check out your new repository, go to the cloned repo directory, and make sure your node version matches the one in the
`.node-version` file. If you're using [nvm (Node Version Manager)](https://github.com/nvm-sh/nvm/blob/master/README.md),
you can run:

```bash
cp .node-version .nvmrc
nvm install
node --version
```

**Note:** Our example is based on
[this commit of the template repository](https://github.com/actions/typescript-action/tree/0467e124e81b527a246cabdaef9c3b433febf9a8).

## Implement your custom GitHub Action

Install the template's dependencies with `npm install`. In addition, install the `@actions/github` package with
`npm install @actions/github`.

The template provides a basic structure for your GitHub Action. Update the `action.yml` file with your action's name,
description, author, and updated inputs/outputs.

```yaml
name: 'Code review'
description: 'Improved code review process'
author: 'Victor on Software'

# Add your action's branding here. This will appear on the GitHub Marketplace.
branding:
  icon: 'heart'
  color: 'red'

# Define your inputs here.
inputs:
  github-token:
    description: 'GitHub secret token'
    required: true

runs:
  using: node20
  main: dist/index.js
```

In this example, our new action will have a single input, `github-token`, the GitHub secret token to use the GitHub API.

Next, we can modify the code in the `src/main.ts` file to implement our custom logic.

```typescript
import * as core from '@actions/core'
import * as github from '@actions/github'
import { readFileSync } from 'fs'

/**
 * The main function of the action.
 * @returns {Promise<void>} Resolves when the action is complete.
 */
export async function run(): Promise<void> {
  try {
    // Get the PR number from the payload. This action is only intended for PRs.
    const prNumber = github.context.payload.pull_request?.number
    if (!prNumber) {
      return
    }

    // Get the required reviewer from the REVIEWERS file.
    // This simplified example assumes that the REVIEWERS file contains a single reviewer.
    // In a real-world scenario, we would need to parse the REVIEWERS file at the top directory
    // of our changed files to get the reviewers.
    const reviewer = readFileSync('REVIEWERS', 'utf8').trim()
    if (!reviewer) {
      core.setFailed('No reviewer found in REVIEWERS file')
      return
    }

    // Get all the reviews for this PR.
    const githubToken = core.getInput('github-token')
    const octokit = github.getOctokit(githubToken)
    const reviews = await octokit.rest.pulls.listReviews({
      owner: github.context.repo.owner,
      repo: github.context.repo.repo,
      pull_number: prNumber
    })

    // Check if the required reviewer has approved the PR.
    // This action does not require the reviewer to re-approve the PR if new changes are pushed.
    let approved = false
    reviews.data.forEach(review => {
      if (review.user?.login === reviewer && review.state === 'APPROVED') {
        approved = true
      }
    })

    // Fail the workflow run if the required reviewer has not approved the PR.
    if (!approved) {
      core.setFailed(`Reviewer ${reviewer} needs to approve the PR`)
      return
    }

  } catch (error) {
    // Fail the workflow run if an error occurs
    if (error instanceof Error) core.setFailed(error.message)
  }
}
```

We read the `REVIEWERS` file to get the required reviewer for the PR. We then retrieve all the reviews for the pull
request and check if the needed reviewer has approved the PR. If the reviewer has not approved the PR, we fail the run.

## Build your custom GitHub Action

To build your action, run `npm run bundle`. This will compile the TypeScript code in the `src/` directory and output the
JavaScript code in the `dist/` directory. The exact command for `bundle` is defined in the `package.json` file.

The `bundle` step is required before you can test your action locally or use it in a workflow because the `action.yml`
file points at the `dist/index.js` bundled version of your code.

This step can easily be forgotten, and you may wonder why your changes are not reflected in the action. You can
configure our IDE to do `npm run bundle` automatically on save, create a check as a
[git pre-commit hook](https://git-scm.com/book/ms/v2/Customizing-Git-Git-Hooks) to ensure the `dist/` directory is
up-to-date, or rely on the `Check Transpiled Javascript` workflow in the `.github/workflows/` directory.

Now, commit your changes and push them to your repository.

**Note:** We will not cover testing in this article, but you should add tests to the `__tests__/` directory for your
source code. For this example, we must refactor the code to make it more testable.

## Test your custom GitHub Action in another repository

### Add workflows and REVIEWERS file

You can use the `uses` keyword in a workflow file to try your action in another repository. Create a new workflow file
in the repository's `.github/workflows/` directory where you want to test your action. For example:

```yaml
name: Code review

on:
  workflow_dispatch: # Manual (for debug)
  pull_request:

jobs:
  code-review:
    name: Code review
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        id: checkout
        uses: actions/checkout@v4
      - name: Code review action
        id: test-action
        uses: getvictor/code-review-demo@main # Your action goes here
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

We are using `@main` as the version of the action to test the latest code on the main branch. You can replace this with
a specific version tag or branch name.

In addition, create a `REVIEWERS` file at the repository's root with the required reviewer's GitHub username.

Notice that the above workflow runs on `pull_request` events but not on `pull_request_review` events. This is because
GitHub Actions treats the `pull_request` workflow runs and the `pull_request_review` workflow runs as distinct. Instead,
we want the same workflow run to be triggered by both events. This is a common pitfall when working with GitHub Actions.
We need to add another workflow file that listens for `pull_request_review` events and triggers the `pull_request`
workflow run to fix this.

```yaml
name: Rerun checks after review

on:
  pull_request_review:
    types:
      - submitted
      - dismissed

jobs:
  rerun_checks:
    name: Rerun specified checks
    permissions:
      actions: write
    runs-on: ubuntu-latest
    steps:
      - name: Rerun Checks
        uses: shqear93/rerun-checks@v3
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          check-names: 'Code review'
```

### Configure a GitHub rule to require code review

Create a branch protection rule in the repository settings that requires the above `Code review` workflow to pass in a
PR before merging to your default branch.

{{< figure src="github-code-review-rule.png" alt="Require a pull request before merging. Require the Code review status check to pass." >}}

### Create a pull request

Commit your changes to a new branch and create a pull request. The `Code review` workflow should run automatically and
show the Failed status.

Once the required reviewer approves the PR, the `Rerun checks after review` workflow will run and trigger the
`Code review` workflow. The rerun should pass, and you should be able to merge the PR.

## Clean up your custom GitHub Action repo

As an optional step, you can clean up your custom GitHub Action repository:

- Update README.md with instructions on how to use your new action
- Remove the `src/wait.ts` file and associated tests
- Update workflows in the `.github/workflows/` directory

## Further reading

In a previous article, we described
[what happens in a GitHub pull request after a `git merge`](../git-merges-and-pull-requests/).

In another article, we covered
[how to use GitHub Actions for general-purpose tasks](../use-github-actions-for-general-purpose-tasks/).

## Example code on GitHub

The code for our simple GitHub Action is available on GitHub: https://github.com/getvictor/code-review-demo

## Watch how to create a custom GitHub Action using TypeScript

{{< youtube NFIwPxz5La8 >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
