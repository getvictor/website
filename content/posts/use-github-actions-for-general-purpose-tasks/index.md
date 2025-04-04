+++
title = "Use GitHub Actions for general-purpose tasks"
description = "Using GitHub Actions for good"
image = "GitHub-action.png"
date = 2024-01-11
categories = ["DevOps & Infrastructure"]
tags = ["GitHub", "GitHub Actions", "Cloud Computing"]
draft = false
+++

## What are GitHub Actions?

GitHub Actions are a way to automate your software development workflows. They are similar to CI/CD tools like Jenkins,
CircleCI, and TravisCI. However, GitHub Actions are built into GitHub.

GitHub Actions are not entirely free, but they have very high usage limits for open-source projects. For private
repositories, you can run up to 2,000 minutes per month for free. After that, you will be charged.

## GitHub Actions for non-CI/CD tasks

However, GitHub Actions are not just for CI/CD. You can use them for many general-purpose tasks. For example, you can
use them as an extension of your application to perform tasks such as:

- generating aggregate reports
- updating a database
- sending notifications
- general data processing
- and many others

A GitHub Action can run arbitrary code, taking inputs from multiple sources such as API calls, databases, and files.

{{< figure src="GitHub-action.png" alt="GitHub Action block diagram" >}}

You can use a GitHub Action as a worker for your application. For example, you can use it to process data from a
database and then send a notification to a user. Or you can use it to generate a report and upload it to a file server.

Although GitHub Actions in open-source repositories are public, they can still use secrets that are not accessible to
the public. For example, secrets can be API keys and database access credentials.

## A real-world GitHub Action doing data processing

Below is an example GitHub Action that does general data processing. It uses API calls to download data from NVD
(National Vulnerability Database), generates files from this data, and then creates a release. Subsequently, the
application can download these files and use them directly without making the API calls or processing the data itself.

GitHub gist: {{< gist getvictor 5b708d408ec5508fbc5f1b3487e8f8a9 >}}

The GitHub Action does a checkout of our application code and runs a script _cmd/cve/generate.go_ to generate the files.
Then, it publishes the generated files as a new release. As a final step, it deletes any old releases.

A note of caution. GitHub monitors for cryptocurrency mining and other abusive behavior. So, keep that in mind and be
careful with process-intensive actions.

## Use GitHub Actions for general-purpose tasks video

{{< youtube y4Jct7eWLmY >}}

## Other articles related to GitHub

- [How to reuse workflows and steps in GitHub Actions](../github-reusable-workflows-and-steps/)
- [What happens in a GitHub pull request after a `git merge`](../git-merges-and-pull-requests/)
- [How to create a custom GitHub Action using TypeScript](../typescript-github-action/)

_Note:_ If you want to comment on this article, please do so on the YouTube video.
