+++
title = "Is staticcheck linter useful for my Go project?"
description = "First impressions enabling the staticcheck linter in our large Go application"
authors = ["Victor Lyuboslavsky"]
image = "staticcheck-go-linter-headline.png"
date = 2024-11-06
categories = ["Software Development"]
tags = ["DevTools", "Golang", "Linting"]
draft = false
+++

## What is staticcheck?

[Staticcheck](https://staticcheck.dev/) is a Go linter that checks your Go code for bugs and performance issues. It is a
powerful tool that can help you find issues in your code before they become problematic. Staticcheck is one of the
default linters in the [golangci-lint](https://golangci-lint.run/) tool.

## Run staticcheck on your Go project

In this example, we will enable staticcheck via the `golangci-lint` tool in a large Go project. The `golangci-lint` tool
is a lint runner that runs many linters in parallel. It is a great tool to use in your CI/CD pipeline to catch issues
early.

### Install golangci-lint

To install the `golangci-lint` tool, you can use one of the options in
[golangci-lint install documentation](https://golangci-lint.run/welcome/install/). We install it using the following
command:

```bash
go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.61.0
```

Although the documentation does not recommend this way of installing from source, we use it to ensure that our version
of `golangci-lint` is compiled using the same Go version as our project. We previously encountered issues with
`golangci-lint` compiled with a different Go version.

Check the version of `golangci-lint`:

```bash
golangci-lint --version
```

Sample output:

```
golangci-lint has version v1.61.0 built with go1.23.1 from (unknown, modified: ?, mod sum: "h1:VvbOLaRVWmyxCnUIMTbf1kDsaJbTzH20FAMXTAlQGu8=") on (unknown)
```

### Run golangci-lint with staticcheck

You can run `staticcheck` using the `golangci-lint` tool. In the root of your Go project, run the following command:

```bash
golangci-lint run --disable-all --enable staticcheck
```

This command turns off all default linters and enables only the `staticcheck` linter. You can view the complete list of
run options with:

```bash
golangci-lint run --help
```

For our project, we add a few more flags to the `golangci-lint run` command:

```bash
golangci-lint run --disable-all --enable staticcheck --timeout 10m --max-same-issues 0 --max-issues-per-linter 0 --exclude-dirs ./node_modules
```

## Analyze and fix staticcheck issues

### [SA1019 - Using a deprecated function, variable, constant or field](https://staticcheck.dev/docs/checks#SA1019)

After running the linter, the first thing we notice is a considerable number of
[SA1019](https://staticcheck.dev/docs/checks#SA1019) fails flagging deprecations, such as:

```
cmd/osquery-perf/agent.go:2574:2: SA1019: rand.Seed has been deprecated since Go 1.20 and an alternative has been available since Go 1.0: As of Go 1.20 there is no reason to call Seed with a random value. Programs that call Seed with a known value to get a specific sequence of results should use New(NewSource(seed)) to obtain a local random generator. (staticcheck)
        rand.Seed(*randSeed)
        ^
```

or

```
server/service/appconfig.go:970:5: SA1019: customSettings[i].Labels is deprecated: the Labels field is now deprecated, it is superseded by LabelsIncludeAll, so any value set via this field will be transferred to LabelsIncludeAll. (staticcheck)
                                customSettings[i].Labels = nil
                                ^
```

The first fail flags a Go library depreciation issue. Although we could fix it, we are not worried because of Go's
commitment to backward compatibility.

The second `SA1019` deprecation fail flags an internal depreciation within our app. However, we must maintain many
deprecated functions within our app for backward compatibility until they can be removed with the next major release.
So, many of these failures cannot be fixed. We could waive each one, but that would be a lot of busy work.

Enabling `SA1019` as a default `staticcheck` rule is a mistake. We suspect many potential users of `staticcheck` will be
turned off by the sheer number of these fails and will simply turn off `staticcheck` in their projects.

We decide to suppress them for now by creating a custom configuration file:

```yaml
linters-settings:
  staticcheck:
    checks: ["all", "-ST1000", "-ST1003", "-ST1016", "-ST1020", "-ST1021", "-ST1022", "-SA1019"]
```

We use the [default staticcheck checks](https://staticcheck.dev/docs/configuration/#example-configuration) and turn off
the `SA1019` check.

We then run `golangci-lint` with the custom configuration file:

```bash
golangci-lint run --disable-all --enable staticcheck --config staticcheck.yml
```

### [SA1032 - Wrong order of arguments to errors.Is](https://staticcheck.dev/docs/checks/#SA1032)

After rerunning the linter, we saw a `SA1032` fail:

```
server/datastore/mysql/vpp.go:1090:6: SA1032: arguments have the wrong order (staticcheck)
                if errors.Is(sql.ErrNoRows, err) {
                   ^
```

This failure is a good catch and a potential bug. We fix it by swapping the arguments:

```go
if errors.Is(err, sql.ErrNoRows) {
```

### [SA4005 - Field assignment that will never be observed. Did you mean to use a pointer receiver?](https://staticcheck.dev/docs/checks/#SA4005)

Another fail we saw was `SA4005`:

```
server/mail/users.go:44:2: SA4005: ineffective assignment to field PasswordResetMailer.CurrentYear (staticcheck)
        r.CurrentYear = time.Now().Year()
        ^
```

The relevant Go code is:

```go
    r.CurrentYear = time.Now().Year()
    t, err := server.GetTemplate("server/mail/templates/password_reset.html", "email_template")
    if err != nil {
       return nil, err
    }

    var msg bytes.Buffer
    if err = t.Execute(&msg, r); err != nil {
       return nil, err
    }
```

In this case, the `CurrentYear` field was used in our template, but the linter could not detect it. We spent a few
minutes testing the template to ensure that the `CurrentYear` field was being populated correctly. To waive this
failure, we add a comment:

```go
    r.CurrentYear = time.Now().Year() // nolint:staticcheck // SA4005 false positive for Go templates
```

### [SA4006 - A value assigned to a variable is never read before being overwritten. Forgotten error check or dead code?](https://staticcheck.dev/docs/checks/#SA4006)

We saw a lot of `SA4006` fails in our codebase. It was the most common `staticcheck` fail we encountered. Here is an
example:

```
ee/fleetctl/updates_test.go:455:2: SA4006: this value of `repo` is never used (staticcheck)
        repo, err = openRepo(tmpDir)
        ^
```

This is a bug or a potential bug. The developer assigned a value to `repo` but never used it. We fix it by removing the
assignment:

```go
_, err = openRepo(tmpDir)
```

### [SA4009 - A function argument is overwritten before its first use](https://staticcheck.dev/docs/checks/#SA4009)

Another fail we saw was `SA4009`. Here is an example:

```
orbit/pkg/installer/installer.go:288:37: SA4009: argument ctx is overwritten before first use (staticcheck)
func (r *Runner) runInstallerScript(ctx context.Context, scriptContents string, installerPath string, fileName string) (string, int, error) {
                                    ^
```

This is another bug or potential bug. A function argument is passed in but then immediately overwritten and never used.
This issue could be challenging to fix because it requires specific code knowledge.

### Other fails

We found a few other fails that were not as critical as the ones mentioned above. We fixed them as we went along. See
the video below for more details.

## Overall impressions

Overall, we like the `staticcheck` linter. It found many bugs or potential bugs and provided a lot of value.

We did have to ignore the `SA1019` check and encountered an `SA4005` false positive.

We will enable it in our CI/CD pipeline and continue to use it in our project.

## Further reading

- Recently, we wrote about
  [finding performance issues with OpenTelemetry and Jaeger in your Go project](../opentelemetry-with-jaeger/).
- We also wrote about [optimizing the performance of your Go code](../optimizing-performance-of-go-app/).
- We also published an article on [Go modules and packages](../go-modules-and-packages/).

## Example code on GitHub

[Fleet repo we used when enabling staticcheck (as of this writing)](https://github.com/fleetdm/fleet/tree/b4a5a1fb49666dd3b10cfd11ccf26190ad9d2902)

## Watch us enable staticcheck in our Go project

{{< youtube oqmVtN-Soig >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
