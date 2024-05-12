+++
title = 'You need a personal dev docs DB (GitBook)'
description = 'Create a free personalized dev docs database'
image = "cover.png"
date = 2023-11-30
categories = ["DevOps & Infrastructure"]
tags = ["Documentation", "Knowledge Sharing", "GitBook"]
draft = false
+++

{{< youtube o-Tml_PAAeM >}}

At [Fleet](https://fleetdm.com/), our developer documentation is spread out throughout the codebase,
contained in a multitude of README and Markdown files. Much of the documentation is hosted
on [our webpage](https://fleetdm.com/docs/get-started/why-fleet), but not all of it.

As developers, we need to be able to quickly search project documentation to find answers to
specific questions, such as:

- How to do a database migration
- How to run integration tests
- How to deploy a development version of to a specific OS

One solution is to use **grep** or the IDE environment to search for these answers. Unfortunately,
such search methods are not optimized for text search -- they frequently generate no relevant
results or too many results that we must manually wade through to find the most appropriate.
Specialized documentation search tools, on the other hand, prioritize headings and whole words,
search for plural versions of the search terms, and offer other conveniences.

The lack of good search capability for engineering docs must be solved in order to scale engineering
efforts. It is an issue because of the following side effects:

- Engineers are discouraged from writing documentation
- Documentation may be duplicated
- Senior developers are frequently interrupted when people can’t find relevant documentation

One solution is to use a documentation service, such as a team
wiki, [Confluence](https://www.atlassian.com/software/confluence),
or [GitBook](https://www.gitbook.com/). GitBook
integrates with git repositories, and can push documentation changes. GitBook is free for personal
use, which makes it easy to use for open source projects such
as [fleet](https://github.com/fleetdm/fleet) and [osquery](https://github.com/osquery/osquery). That
said,
GitBook is a newcomer to the space, and is still reaching maturity.

To set up a personal GitBook, make a fork of the open source projects that contain documentation
you’d like to search, and integrate them into GitBook spaces. After indexing is complete, you’ll be
able to effectively search the documentation.

{{< figure src="GitBook-1.png" alt="Search for database migrations in Fleet's GitBook" >}}

To keep the forks in sync with the parent repositories, we use Github Actions. Github Actions are
free for open source projects. Searching GitHub for **sync-fork** returned several examples. We
ended up using the following:

```yaml
name: Sync Fork

on:
  schedule:
    - cron: '55 * * * *'
  workflow_dispatch: # on button click

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.WORKFLOW_TOKEN }}
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config --global user.name "GitHub Actions Bot"
          git config --global user.email "actions@github.com"

      - name: Merge upstream
        run: |
          git remote add upstream https://github.com/fleetdm/fleet.git
          git fetch upstream main
          git checkout main
          git merge upstream/main
          git push origin main
```

The **WORKFLOW_TOKEN** above is a GitHub personal access token (PAT) that allows reading and writing
workflows in this repository. This token is not needed for repositories without workflows.

In addition to project documentation, GitBook can be used to synchronize personal documentation
that’s being held in a private repository. There are several git-based notebook applications on the
market. In addition, Markdown notes from the popular note-taking
app [Obsidian](https://obsidian.md/) can be kept in GitHub. This turns GitBook into a true
personalized developer documentation database -- one place to search through developer docs as well
as your own private notes.
