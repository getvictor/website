+++
title = "Use GitHub actions for general-purpose tasks"
description = "Using GitHub actions for good"
image = "GitHub-action.png"
date = 2024-01-11
tags = ["github", "githubactions", "cloudcomputing"]
draft = false
+++

{{< youtube y4Jct7eWLmY >}}

## What are GitHub actions?

GitHub actions are a way to automate your software development workflows. They are similar to CI/CD tools like Jenkins, CircleCI,
and TravisCI. However, GitHub actions are built into GitHub.

GitHub actions are not entirely free, but they have very high usage limits for open-source projects. For private repositories, you can
run up to 2,000 minutes per month for free. After that, you will be charged.

## GitHub actions for non-CI/CD tasks

However, GitHub actions are not just for CI/CD. You can use them for many general-purpose tasks. For example, you can use them as an
extension of your application to perform tasks such as:

- generating aggregate reports
- updating a database
- sending notifications
- general data processing
- and many others

A GitHub action can run arbitrary code, taking inputs from multiple sources such as API calls, databases, and files.

{{< figure src="GitHub-action.png" alt="GitHub action block diagram" >}}

You can use a GitHub action as a worker for your application. For example, you can use it to process data from a database and then
send a notification to a user. Or you can use it to generate a report and upload it to a file server.

Although GitHub actions in open-source repositories are public, they can still use secrets that are not accessible to the public.
For example, secrets can be API keys and database access credentials.

## A real-world GitHub action doing data processing

Below is an example GitHub action that does general data processing. It uses API calls to download data from NVD (National Vulnerability
Database), generates files from this data, and then creates a release. Subsequently, the application can download these files and use them
directly without making the API calls or processing the data itself.

GitHub gist:
{{< gist getvictor 5b708d408ec5508fbc5f1b3487e8f8a9 >}}

The GitHub action does a checkout of our application code and runs a script *cmd/cve/generate.go* to generate the files. Then, it publishes
the generated files as a new release. As a final step, it deletes any old releases.

A note of caution. GitHub monitors for cryptocurrency mining and other abusive behavior. So, keep that in mind and be careful with
process-intensive actions.
