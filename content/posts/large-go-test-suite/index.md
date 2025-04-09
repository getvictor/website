+++
title = "How to speed up a large Go test suite"
description = "Speed up your CI test time by parallelizing a large Go test suite"
authors = ["Victor Lyuboslavsky"]
image = "large-go-test-suite-headline.png"
date = 2025-04-09
categories = ["Software Development"]
tags = ["Golang", "Unit Testing", "Developer Experience"]
draft = false
+++

Fast Continuous Integration (CI) test results are crucial for maintaining a good developer velocity. Quick test results
give developers immediate feedback on their changes, resulting in a more enjoyable development process. For critical
changes, slow tests can become a bottleneck, delaying deployments.

Previously, we covered [how to accurately measure the execution time of Go tests](../go-test-execution-time/). This
article will demonstrate one approach to breaking apart a large Go test suite and running each part in parallel. This
approach should reduce the CI cycle time, benefitting developers and the organization.

- [Split up the Go test suite](#split-up-the-go-test-suite)
- [Create parallel Go test jobs in CI](#create-parallel-go-test-jobs-in-ci)

## Understanding your Go test suite

The standard way to run all the tests in your Go project is with the following command:

```bash
go test ./...
```

This command will compile all the Go packages and their tests, each compiling to a separate binary. The Go toolchain
will then run each binary in parallel. The `-p` flag controls this parallel test behavior, which defaults to the number
of CPUs on your machine.

Splitting up the test suite makes sense if you have a lot of packages to test. If you have few packages or if you have 1
or 2 packages that dominate your project's test run time, then a simple split may not help much. You may need to
refactor your code or split the tests in a single Go package across multiple CI jobs. Splitting a single package is
generally inefficient since each CI job must compile the same package separately, and we will not cover this approach.

To find all the packages in your project, you can list them with the following command:

```bash
go list ./...
```

To identify time-consuming packages, you can run your test suite with the `-json` switch and save the results. Then,
find the elapsed time of each package and sort the times. This operation can be done with [jq](https://jqlang.org/):

```bash
cat test-result.json | jq -s 'map(select(has("Test") | not)) | group_by(.Package) | map({package: .[0].Package, elapsed: (map(.Elapsed) | add)}) | sort_by(.elapsed) | reverse'
```

## Split up the Go test suite

You can specify the packages at the end of the `go test` command to run a subset of packages in a CI job. For example:

```bash
go test ./cmd/fleetctl/... ./server/service
```

Next, manually create groups of packages and identify them with a name.

To create a catchall group for packages that were not explicitly assigned to a group, you can use the Linux
[comm](https://linux.die.net/man/1/comm) command to generate the remaining packages. For example:

```bash
comm -23 <(go list ./... | sort) <({ go list ./cmd/fleetctl/... && go list ./server/service/... ;} | sort)
```

The above command returns the packages unique to the first list, which includes all the packages (`./...`).

The following is a real-world example from Fleet's Makefile that creates test suite groups with identifiers:

```Makefile
# Set up packages for CI testing.
DEFAULT_PKGS_TO_TEST := ./cmd/... ./ee/... ./orbit/pkg/... ./orbit/cmd/orbit ./pkg/... ./server/... ./tools/...
# fast tests are quick and do not require out-of-process dependencies (such as MySQL, etc.)
FAST_PKGS_TO_TEST := \
    ./ee/tools/mdm \
    ./orbit/pkg/cryptoinfo \
    ./orbit/pkg/dataflatten \
    ./orbit/pkg/keystore \
    ./server/goose \
    ./server/mdm/apple/appmanifest \
    ./server/mdm/lifecycle \
    ./server/mdm/scep/challenge \
    ./server/mdm/scep/x509util \
    ./server/policies
FLEETCTL_PKGS_TO_TEST := ./cmd/fleetctl/...
MYSQL_PKGS_TO_TEST := ./server/datastore/mysql/... ./server/mdm/android/mysql
SCRIPTS_PKGS_TO_TEST := ./orbit/pkg/scripts
SERVICE_PKGS_TO_TEST := ./server/service
VULN_PKGS_TO_TEST := ./server/vulnerabilities/...
ifeq ($(CI_TEST_PKG), main)
    # This is the bucket of all the tests that are not in a specific group. We take a diff between DEFAULT_PKG_TO_TEST and all the specific *_PKGS_TO_TEST.
    CI_PKG_TO_TEST=$(shell /bin/bash -c "comm -23 <(go list ${DEFAULT_PKGS_TO_TEST} | sort) <({ \
    go list $(FAST_PKGS_TO_TEST) && \
    go list $(FLEETCTL_PKGS_TO_TEST) && \
    go list $(MYSQL_PKGS_TO_TEST) && \
    go list $(SCRIPTS_PKGS_TO_TEST) && \
    go list $(SERVICE_PKGS_TO_TEST) && \
    go list $(VULN_PKGS_TO_TEST) \
    ;} | sort)")
else ifeq ($(CI_TEST_PKG), fast)
    CI_PKG_TO_TEST=$(FAST_PKGS_TO_TEST)
else ifeq ($(CI_TEST_PKG), fleetctl)
    CI_PKG_TO_TEST=$(FLEETCTL_PKGS_TO_TEST)
else ifeq ($(CI_TEST_PKG), mysql)
    CI_PKG_TO_TEST=$(MYSQL_PKGS_TO_TEST)
else ifeq ($(CI_TEST_PKG), scripts)
    CI_PKG_TO_TEST=$(SCRIPTS_PKGS_TO_TEST)
else ifeq ($(CI_TEST_PKG), service)
    CI_PKG_TO_TEST=$(SERVICE_PKGS_TO_TEST)
else ifeq ($(CI_TEST_PKG), vuln)
    CI_PKG_TO_TEST=$(VULN_PKGS_TO_TEST)
else
    CI_PKG_TO_TEST=$(DEFAULT_PKGS_TO_TEST)
endif
```

## Create parallel Go test jobs in CI

The major CI tools provide a way to start multiple jobs in parallel. In GitHub, this is done with a
[matrix strategy](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow).

In the previous step, we gave an example of named test suites. Now, we feed those names into the GitHub matrix job:

```yaml
jobs:
  test-go:
    strategy:
      matrix:
        suite: ["fast", "fleetctl", "main", "mysql", "scripts", "service", "vuln"]
        os: [ubuntu-latest]
    runs-on: ${{ matrix.os }}

    steps:
    - name: Checkout Code
      uses: actions/checkout@c85c95e3d7251135ab7dc9ce3241c5835cc595a9 # v3.5.3

    - name: Install Go
      uses: actions/setup-go@0a12ed9d6a96ab950c8f026ed9f722fe0da7ef32 # v5.0.2
      with:
        go-version-file: 'go.mod'

    - name: Run Go Tests
      run: CI_TEST_PKG=${{ matrix.suite }} make test-go
```

The above workflow runs our test suites in parallel, speeding up our overall CI cycle time.

## Further reading

Recently, we covered [analyzing Go build times](../analyze-go-build/).

In the past, we reviewed [the state of fuzz testing in Go](../fuzz-testing-with-go/).

## Watch how to break apart a large Go test suite

{{< youtube AFVlbf5LZwc >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
