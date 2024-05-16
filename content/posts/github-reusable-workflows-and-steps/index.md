+++
title = 'How to reuse workflows and steps in GitHub Actions (2024)'
description = "How to create reusable workflows and reusable steps in GitHub Actions"
image = "GitHub Actions thumbnail.png"
date = 2024-05-01
categories = ["DevOps & Infrastructure"]
tags = ["GitHub", "GitHub Actions", "Code Reuse"]
draft = false
+++

- [GitHub reusable workflows](#reusable-workflows)
- [GitHub reusable steps (composite action)](#reusable-steps-composite-action)

## Introduction

[GitHub Actions](https://github.com/features/actions) is a way to automate your software development workflows. The approach is similar to CI/CD tools like Jenkins, CircleCI, and TravisCI. However, GitHub Actions are built into GitHub.

{{< figure src="GitHub Actions workflow.svg" alt="High level diagram of GitHub Actions" title="High level diagram of GitHub Actions" >}}

The entry point for GitHub Actions is the `.github/workflows` directory in your repository. This directory contains one or more YAML files that define your workflows. A workflow is an automated process made up of one or more jobs. Each job runs on a separate runner. A runner is a server that runs the job. A job contains one or more steps. Each step runs a separate command.

## Why reuse?

[Code reuse](https://en.wikipedia.org/wiki/Code_reuse) is a fundamental principle of software development. Reusing GitHub Actions code allows you to:
- Improve maintainability by keeping common code in one place and reducing the amount of code
- Increase consistency since multiple workflows can use the same code
- Promote best practices
- Increase productivity
- Reduce errors

Examples of reusable GitHub Actions code include:
- Code signing
- Uploading artifacts to cloud services
- Security checks
- Notifications and reports
- Data processing
- and many others

## Reusable workflows {#reusable-workflows}

A reusable workflow replaces a job in the main workflow.

{{< figure src="GitHub Actions reusable workflow.svg" alt="GitHub Actions reusable workflow" title="GitHub Actions reusable workflow" >}}

A reusable workflow may be shared across repositories and run on a different platform than the main workflow.

For file sharing, 'build artifacts' must be used to share files with the main workflow. The reusable workflow does not inherit environment variables. However, it accepts inputs and secrets from the calling workflow and may use outputs to pass data back to the main workflow.

Here is an example of a reusable workflow. It uses the same schema as a regular workflow.

```yaml
name: Reusable workflow

on:
  workflow_call:
    inputs:
      reusable_input:
        description: 'Input to the reusable workflow'
        required: true
        type: string
      filename:
        required: true
        type: string
    secrets:
      HELLO_WORLD_SECRET:
        required: true
    outputs:
      # Map the workflow output(s) to job output(s)
      reusable_output:
        description: 'Output from the reusable workflow'
        value: ${{ jobs.reusable-workflow-job.outputs.job_output }}

defaults:
  run:
    shell: bash

jobs:
  reusable-workflow-job:
    runs-on: ubuntu-20.04
    # Map the job output(s) to step output(s)
    outputs:
      job_output: ${{ steps.process-step.outputs.step_output }}
    steps:
      - name: Process reusable input
        id: process-step
        env:
          HELLO_WORLD_SECRET: ${{ secrets.HELLO_WORLD_SECRET }}
        run: |
          echo "reusable_input=${{ inputs.reusable_input }}"
          echo "HELLO_WORLD_SECRET=${HELLO_WORLD_SECRET}"
          echo "step_output=${{ inputs.reusable_input }}_processed" >> $GITHUB_OUTPUT
      - uses: actions/download-artifact@v4
        with:
          name: input_file
      - name: Process file
        run: |
          echo "Processing file: ${{ inputs.filename }}"
          echo "file processed" >> ${{ inputs.filename }}
      - uses: actions/upload-artifact@v4
        with:
          name: output_file
          path: ${{ inputs.filename }}
```

The reusable workflow is triggered `on: workflow_call`. It accepts an input called `reusable_input` and generates an output called `reusable_output`. It also downloads an artifact called `input_file`, processes a file, and uploads an artifact called `output_file`.

The main workflow calls the reusable workflow using the `uses` keyword.

```yaml
  job-2:
    needs: job-1
    # We do not need to check out the repository to use the reusable workflow
    uses: ./.github/workflows/reusable-workflow.yml
    with:
      reusable_input: "job-2-input"
      filename: "input.txt"
    secrets:
      # Can also implicitly pass the secrets with: secrets: inherit
      HELLO_WORLD_SECRET: TERCES_DLROW_OLLEH
```

A successful run of the main workflow looks like this on GitHub:

{{< figure src="GitHub Actions reusable workflow success.png" alt="GitHub Actions reusable workflow success" title="GitHub Actions reusable workflow success" >}}

## Reusable steps (composite action) {#reusable-steps-composite-action}

Reusable steps replace a regular step in a job. We will use a `composite action` for reusable steps in our example.

{{< figure src="GitHub Actions reusable steps.svg" alt="GitHub Actions reusable steps (composite action)" title="GitHub Actions reusable steps (composite action)" >}}

Like a reusable workflow, a composite action may be shared across repositories, it accepts inputs, and it may use outputs to pass data back to the main workflow.

Unlike a reusable workflow, a composite action inherits environment variables. However, it does not inherit secrets. Secrets must be passed explicitly as inputs or environment variables. Also, there is no need to use 'build artifacts' to share files since the reusable steps run on the same runner and in the same work area as the main job.

Here is an example of a composite action. It uses a different schema than a workflow. Also, the file must be named `action.yml` or similar.

```yaml
name: Reusable steps (AKA composite action)
description: Demonstrate how to use reusable steps in a workflow
# Schema: https://json.schemastore.org/github-action.json

inputs:
  reusable_input:
    description: 'Input to the reusable workflow'
    required: true
  filename:
    required: true
outputs:
  # Map the action output(s) to step output(s)
  reusable_output:
    description: 'Output from the reusable workflow'
    value: ${{ steps.process-step.outputs.step_output }}

runs:
  using: 'composite'
  steps:
    - name: Process reusable input
      id: process-step
      # Shell must explicitly specify the shell for each step. https://github.com/orgs/community/discussions/18597
      shell: bash
      run: |
        echo "reusable_input=${{ inputs.reusable_input }}"
        echo "HELLO_WORLD_SECRET=${HELLO_WORLD_SECRET}"
        echo "step_output=${{ inputs.reusable_input }}_processed" >> $GITHUB_OUTPUT
    - name: Process file
      shell: bash
      run: |
        echo "Processing file: ${{ inputs.filename }}"
        echo "file processed" >> ${{ inputs.filename }}
```

The composite action is called via the `uses` setting on a step. Our action accepts an input called `reusable_input` and generates an output called `reusable_output`. It also processes a file called `filename`.

The following code snippet shows how to use the composite action in a job.

```yaml
  - name: Use reusable steps
    id: reusable-steps
    uses: ./.github/reusable-steps # To use this syntax, we must have the repository checked out
    with:
      reusable_input: "job-2-input"
      filename: "input.txt"
    env:
      HELLO_WORLD_SECRET: TERCES_DLROW_OLLEH
```

A successful run of the main workflow with reusable steps looks like this on GitHub:

{{< figure src="GitHub Actions composite action success.png" alt="GitHub Actions composite action success" title="GitHub Actions composite action success" >}}

## Conclusion

Reusable workflows and steps are powerful tools for improving the maintainability, consistency, and productivity of your GitHub Actions. They allow you to reuse code across repositories and workflows and promote best practices. They are a great way to reduce errors and increase productivity.

For larger units of work, a reusable workflow should be used. A composite action should be used for smaller units of work that may run on the same runner and share the same work area.

## Example code on GitHub

The example code is available on GitHub at: https://github.com/getvictor/github-reusable-workflows-and-steps

## GitHub Actions reusable workflows and steps video

{{< youtube ciHJzV6TZB8 >}}

## Other articles related to GitHub Actions

- [Use GitHub actions for general-purpose tasks](../use-github-actions-for-general-purpose-tasks/)

*Note:* If you want to comment on this article, please do so on the YouTube video.
