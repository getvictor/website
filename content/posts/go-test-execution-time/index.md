+++
title = "How to measure the execution time of Go tests accurately"
description = "Improve developer experience by optimizing Go test execution time using detailed performance data"
authors = ["Victor Lyuboslavsky"]
image = "crash-test-dummy-headline.png"
date = 2024-09-04
categories = ["Software Development"]
tags = ["Golang", "Unit Testing", "Developer Experience"]
draft = false
+++

- [Accurately measuring test execution time](#accurately-measuring-test-execution-time)

## Why measure test execution time?

By speeding up your test suite, you're improving developer experience and productivity. Faster tests mean faster
feedback, which leads to quicker iterations and better code quality.

When you run tests, you want to know how long they take to execute. This information can help you optimize your test
suite and make it run faster. By measuring the execution time of your tests, you can identify slow tests and improve
their performance.

## Problems with current measurement tools

We have yet to find a tool that provides detailed, actionable insights into the performance of Go tests.

For example, running the `gotestsum tool slowest` command from the
[gotestsum](https://github.com/gotestyourself/gotestsum) tool gave us the following output for our test suite:

```
github.com/fleetdm/fleet/v4/server/datastore/mysql TestMDMApple 6m9.65s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestSoftware 4m8.9s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestPolicies 3m31s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestActivity 2m16.67s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestMDMWindows 2m14.85s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestMDMShared 2m10.27s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestVulnerabilities 2m7.98s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestPacks 1m59.2s
github.com/fleetdm/fleet/v4/server/worker TestAppleMDM 1m55.11s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestTeams 1m47.82s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestAppConfig 1m42.81s
github.com/fleetdm/fleet/v4/server/datastore/mysql TestHosts 1m41.79s
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240709183940 1m36.43s
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240709132642 1m36.34s
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240725182118 1m35.95s
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240730171504 1m35.73s
github.com/fleetdm/fleet/v4/server/vulnerabilities/nvd TestTranslateCPEToCVE/recent_vulns 1m34.87s
...
```

The first thing to notice is that the numbers don't add up. Our test suite takes around 14 minutes to run, but the times
in the report add up to more than 14 minutes. This discrepancy makes it hard to identify the slowest tests.

The second thing to notice is that our tests contain many subtests. The `TestMDMApple` test contains over 40 subtests.
We want to know the execution time of each subtest, not just the total time for the test.

The third thing to notice is that the output does not provide any information regarding parallelism. We want to know if
our tests run in parallel and how many run concurrently. We want to run tests in parallel when possible to speed up the
test suite.

## Understanding parallelism in Go tests

Before measuring the execution time of our tests, we need to understand how Go tests run in parallel.

{{< figure src="go-test-parallelism.svg" alt="Sequence diagram of a go test run with two packages, two tests, and two subtests." >}}

When you run `go test`, Go compiles each package in your test suite in a separate binary. It then runs each binary in
parallel. The tests in different packages run concurrently. This behavior is controlled by the `-p` flag, which defaults
to `GOMAXPROCS`, the number of CPUs on your machine.

Within a package, tests run sequentially by default -- the tests in the same package run one after the other. However,
you can run tests in parallel within a package by calling `t.Parallel()` in your test functions. This behavior is
controlled by the `-parallel` flag, which also defaults to `GOMAXPROCS`. So, in a system with 8 CPUs, running a test
suite with many packages and parallel tests will run 8 packages concurrently and 8 tests within each package
concurrently, for a total of 64 tests running concurrently.

Each test function may have multiple subtests, which may have their own subtests, and so on. Subtests run sequentially
by default. However, you can also run subtests in parallel by calling `t.Parallel()` in your subtest functions.

## Accurately measuring test execution time {#accurately-measuring-test-execution-time}

To measure the execution time of your tests, we must use the `-json` flag with the `go test` command. This flag outputs
test results in JSON format, which we can parse and analyze.

The `Action` field in the JSON output shows the start and end times of each test and subtest.

```json
{
  "Time": "2024-08-26T19:22:51.969606869Z",
  "Action": "run",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB"
}
{
  "Time": "2024-08-26T19:22:51.96984165Z",
  "Action": "run",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB/test1"
}
{
  "Time": "2024-08-26T19:22:51.969928132Z",
  "Action": "pause",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB/test1"
}
{
  "Time": "2024-08-26T19:22:51.969983777Z",
  "Action": "run",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB/test2"
}
{
  "Time": "2024-08-26T19:22:51.970052987Z",
  "Action": "pause",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB/test2"
}
{
  "Time": "2024-08-26T19:22:51.970090377Z",
  "Action": "cont",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB/test1"
}
{
  "Time": "2024-08-26T19:22:51.973464469Z",
  "Action": "cont",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB/test2"
}
{
  "Time": "2024-08-26T19:22:52.015505184Z",
  "Action": "pass",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB/test1",
  "Elapsed": 0.04
}
{
  "Time": "2024-08-26T19:22:52.015523238Z",
  "Action": "pass",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB/test2",
  "Elapsed": 0.04
}
{
  "Time": "2024-08-26T19:22:52.015527907Z",
  "Action": "pass",
  "Package": "github.com/fleetdm/fleet/v4/cmd/cpe",
  "Test": "TestCPEDB",
  "Elapsed": 0
}
```

While parsing the JSON output, we can track how many tests are running in parallel. We can then adjust the execution
time of each test by dividing the total time by the number of tests running concurrently. Since we don't have access to
the actual CPU time each test used, this is the best approximation we can get.

When tests run in parallel, we typically see the `pause` and `cont` actions. If we see these actions, we know that the
test or subtest is running in parallel.

We created a parser called [goteststats](https://github.com/getvictor/goteststats) that does these calculations.

## Accurate test execution time measurement in practice

By running our [goteststats](https://github.com/getvictor/goteststats) parser on the JSON output of our test suite, we
gained actionable insights into our tests' performance.

```
WARNING: Stopped test not found in running tests: TestGenerateMDMApple/successful_run
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240709132642: 8.142s (total: 2m12.806s parallel: 16)
github.com/fleetdm/fleet/v4/server/cron TestCalendarEvents1KHosts: 7.853s (total: 36.158s parallel: 4)
github.com/fleetdm/fleet/v4/server/cron TestEventForDifferentHost: 7.853s (total: 36.158s parallel: 4)
github.com/fleetdm/fleet/v4/cmd/fleet TestCronVulnerabilitiesCreatesDatabasesPath: 6.878s (total: 30.232s parallel: 4)
github.com/fleetdm/fleet/v4/server/vulnerabilities/nvd TestTranslateCPEToCVE/find_vulns_on_cpes: 6.849s (total: 1m34.89s parallel: 13)
github.com/fleetdm/fleet/v4/server/vulnerabilities/nvd TestTranslateCPEToCVE/recent_vulns: 6.849s (total: 1m34.89s parallel: 13)
github.com/fleetdm/fleet/v4/server/vulnerabilities/oval TestOvalAnalyzer/#load/invalid_vuln_path: 5.844s (total: 1m25.152s parallel: 14)
github.com/fleetdm/fleet/v4/server/vulnerabilities/oval TestOvalAnalyzer/analyzing_RHEL_software: 5.844s (total: 1m25.152s parallel: 14)
github.com/fleetdm/fleet/v4/server/vulnerabilities/oval TestOvalAnalyzer/analyzing_Ubuntu_software: 5.844s (total: 1m25.151s parallel: 14)
github.com/fleetdm/fleet/v4/cmd/fleet TestAutomationsSchedule: 5.699s (total: 14.213s parallel: 2)
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240725182118: 5.623s (total: 1m37.577s parallel: 17)
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240709183940: 5.588s (total: 1m36.771s parallel: 17)
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240709124958: 5.52s (total: 1m35.622s parallel: 17)
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240730171504: 5.517s (total: 1m35.74s parallel: 17)
github.com/fleetdm/fleet/v4/server/datastore/mysql/migrations/tables TestUp_20240726100517: 5.418s (total: 1m33.987s parallel: 17)
...
```

For a given test, we provide the adjusted time, the total time, and the average number of tests running concurrently
with this test. The adjusted time is the time the test took to execute, which is also the time saved if we removed this
test from the suite.

The first thing to notice is that the numbers add up. The total time for the test suite is around 14 minutes, and the
times in the report add up to around 14 minutes.

The second thing to notice is that we now have the execution time of each subtest. This information is crucial for
identifying slow tests and improving their performance.

The third thing to notice is that we now have information about parallelism. We can see how many tests are running
concurrently and how many tests are running in parallel. If we see a test with a low parallelism number, we know that
this test is a bottleneck and should parallelized.

The WARNING message indicates that the JSON output did not contain the start time of the test. This issue can happen if
the console output of the code under test does not include a new line and gets mixed with the output of Go's testing
package. For example:

```json
{
  "Time": "2024-08-26T19:23:17.8084601Z",
  "Action": "output",
  "Package": "github.com/fleetdm/fleet/v4/cmd/fleetctl",
  "Test": "TestGenerateMDMApple/CSR_API_call_fails",
  "Output": "requesting APNs CSR: GET /api/latest/fleet/mdm/apple/request_csr received status 502 Bad Gateway: FleetDM CSR request failed: bad request=== RUN   TestGenerateMDMApple/successful_run\n"
}
```

## `goteststats` on GitHub

[goteststats](https://github.com/getvictor/goteststats) is available on GitHub. You can use it to get detailed
performance data for your Go test suite.

## Further reading

- Recently, we wrote about [optimizing the performance of Go applications](../optimizing-performance-of-go-app).
- We also explored [fuzz testing with Go](../fuzz-testing-with-go).
- In addition, we showed [how to create an EXE installer for a Go program](../exe-installer).
- We also published an article on [using Go modules and packages](../go-modules-and-packages).
- And we wrote about [automatically tracking engineering metrics with Go](../track-engineering-metrics/).

## Watch how to measure the execution time of Go tests accurately

{{< youtube caTDvS5vCjA >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
