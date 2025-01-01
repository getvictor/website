+++
title = "Go modules and packages — which one to use and when"
description = "What's the difference between Go modules and packages? How to use multiple modules in one repository."
authors = ["Victor Lyuboslavsky"]
image = "go-modules-and-packages.png"
date = 2024-12-04
categories = ["Software Development"]
tags = ["Golang", "Dependency Management"]
draft = false
+++

## What is a Go package?

A Go package is a collection of Go source files in the same directory that are compiled together. It can contain
functions, types, and variables. Go packages organize code and provide a way to share code between different program
parts.

By convention, the package name is the same as the last element of the import path. For example, the package name for
the `fmt` package in the `fmt` directory is `fmt`. You can name the package differently from the directory name, but it
is not recommended.

When a Go application is compiled, the Go compiler compiles all the packages imported by the main package. The main
package contains the `main` function and is the entry point of the program. However, all the imported packages can be
compiled separately as well, even if they do not contain the `main` function. That's what happens when we run unit tests
for all packages in a Go project — each package is compiled separately and can be tested independently.

The directory structure does not have to match the package dependencies. For example, given the following directory
structure:

```
service/
  service.go
  api/
    api.go
  db/
    db.go
```

The packages `service`, `api`, and `db` can be completely independent. The `service` package does not include the
`api.go` and `db.go` files because they are in a different directory. Go's package dependency graph has no relation to
the directory structure.

## What is a Go module?

A [Go module](https://go.dev/ref/mod), introduced in Go 1.11, is a collection of Go packages that are versioned
together. A Go module is defined by a `go.mod` file located at the module's root. The `go.mod` file contains the module
name and the versions of the dependencies the module uses.

A Go module can contain multiple packages, and the packages do not need to be related to each other.

{{< figure src="go-modules-and-packages.svg" title="Sample project with two modules" alt="Go project directory structure, containing one module at the top level with two packages in a subdirectory. Another module exists in a third subdirectory.">}}

A Go module can exist at any level of the directory structure, even nested within another module. The Go toolchain
treats all Go modules as independent entities, regardless of their location in the directory structure.

## How to use multiple modules in one Go workarea

### 1. Create a new Go workarea:

```bash
mkdir -p go-modules
cd go-modules
go mod init example.com/go-modules
```

### 2. Create a simple Go file in the root of the workarea:

```go
package main

import "fmt"

func main() {
    fmt.Printf("Hello, world.\n")
}
```

### 3. Create a new Go module in a subdirectory:

```bash
mkdir -p adder
cd adder
go mod init example.com/go-modules/adder
```

With the following Go file in the `adder` directory:

```go
package adder

func Add(a, b int) int {
    return a + b
}
```

### 4. Use the `adder` package in the `main` package:

You cannot simply import the `adder` package in the `main` package because they are in different modules.

One way to use the `adder` package is to publish it and use `go get` to download it as the `main` dependency. However,
this is impractical for local development and doesn't make sense when we explicitly want to use multiple modules in one
repository.

The proper way is to use a [workspace, declared in the go.work file](https://go.dev/ref/mod#workspaces), introduced in
Go 1.18.

In the top level of the workarea, create a `go.work` file with all the modules in the subdirectories:

```bash
go work use -r .
```

The resulting `go.work` file will look like this:

```text
go 1.23.2

use (
    .
    ./adder
)
```

Now, you can import the `adder` package in the `main` package:

```go
package main

import (
    "fmt"

    "example.com/go-modules/adder"
)

func main() {
    fmt.Printf("Hello, world.\n")
    fmt.Print(myadder.Add(1, 2))
}
```

## Should I use multiple Go modules in one repository?

Generally, it is not recommended to use multiple Go modules in one repository. However, there are some cases where it
makes sense, like:

- Temporarily pull in a third-party module to add a feature before this feature is merged upstream.
- Work on multiple interdependent modules that can be versioned and released independently.

Usually, it is better to use packages within a single module to organize and decouple code.

## Further reading

- Recently, we covered [method overriding in Go](../method-overriding-in-go/).
- We also wrote about [using the staticcheck linter on a large Go project](../staticcheck-go-linter/).
- Previously, we described [how to use Go to unmarshal JSON null, set, and missing fields](../go-json-unmarshal/).
- We also published an article on [accurately measuring the execution time of Go tests](../go-test-execution-time/).

## Multiple modules in one Go project on GitHub

[Example Go project with multiple modules](https://github.com/getvictor/go-modules)

## Watch an explanation of Go modules and packages, along with an example of using multiple modules in one repository

{{< youtube EdV1rx5613g >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
