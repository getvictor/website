+++
title = "How to find package dependencies of a Go package"
description = "2 ways to find and check that your Go package is dependent on specific other packages"
authors = ["Victor Lyuboslavsky"]
image = "go-dependencies-headline.png"
date = 2025-02-05
categories = ["Software Development"]
tags = ["Software Architecture", "Golang", "Dependency Management"]
draft = false
+++

- [Find package dependencies using `go list`](#find-package-dependencies-using-go-list)
- [Find package dependencies using Go code](#find-package-dependencies-using-go-code)

## What are package dependencies and module dependencies?

A package dependency is another package that your Go package imports. When you import a package in Go, you create a
dependency on that package. The Go compiler will not compile your package if it cannot find and compile the package you
depend on.

On the other hand, a module dependency is a dependency on a module. A module is a collection of related Go packages that
are versioned together. You declare your module dependencies in your `go.mod` file. Your code may use one or more
packages from your module dependencies.

## Why are package dependencies important?

Understanding your package dependencies is essential because they:

- indicate the amount of internal coupling in your codebase
- help you understand the structure of your codebase
- help you avoid too many dependencies
- help you avoid circular dependencies
- help you optimize your build times

As your codebase grows, keeping track of package dependencies is vital to ensure that the codebase remains maintainable.
Many developers import dependencies without considering the consequences. In modern IDE tools, they quickly click
`Import` in a pop-up to make the squiggly lines go away. In some cases, IDEs add imports without even asking the
developer. However, code with many dependencies becomes coupled to other potentially unrelated code. This entanglement
makes the codebase harder to understand, test, and maintain. For additional details, see
[the list of problems with a coupled architecture](../scaling-codebase-evolutionary-architecture/#problems-with-the-current-architecture)
from our previous article.

### What is an architectural test?

An architectural test is a test that makes sure your code follows the architectural rules that you have defined.
Codebases tend to devolve into a Big Ball of Mud as time passes. Architectural tests are one way to keep your codebase
clean.

In our example below, we will check to ensure that our Go package is NOT dependent on another package in our codebase.
This is a common scenario when you want to refactor your codebase and remove a dependency or add a new package and want
to ensure that it is not dependent on other parts of the codebase.

## Find package dependencies using `go list`

`go list` is a powerful tool that you can use to list information about Go packages. You can use the `-deps` flag with
`go list` to find package dependencies. Here is an example:

```bash
go list -deps ./server/android...
```

The result is a list of all the direct and indirect package dependencies of the `./server/android` and its subpackages.
To filter out standard library packages and sort the list, you can use the following command on macOS:

```bash
go list -deps ./server/android... | grep -E '^[^\/]*\.[^\/]*\/' | sort
```

The above regular expression looks for packages with a `.` before the first `/` in the package path. This regex filters
out standard library packages. The `sort` command sorts the list alphabetically.

To check if a package is dependent on another package, you can use the following command:

```bash
! (go list -deps ./server/android... | grep -q 'github.com/fleetdm/fleet/v4/server/mdm/nanomdm/mdm')
```

The leading `!` inverts the command's exit status. If the package is dependent on the specified package, the command
will return `1`; if it is not, the command will return `0`. You can use this command in your CI/CD pipelines to ensure
that your package is not dependent on a specific package.

## Find package dependencies using Go code

[packages](https://pkg.go.dev/golang.org/x/tools/go/packages) is a Go package that allows one to load, parse,
type-check, and import Go packages. We will use the `Load` function to get a list of `Package` values. In addition, we
will use [Context.Import method from build package](https://pkg.go.dev/go/build#Context.Import) to recursively find
dependencies.

Below is an example architecture test you can add to your test suite.

{{< gist getvictor 17f495021211dcca087b16cd2d4b24d1 >}}

The above example is based on https://github.com/matthewmcnew/archtest. You can
[jump to the code example section](https://youtu.be/yIZcbTQvpCE?t=440&si=T2AqNGc9_YMjbCTx) of the video below for a full
explanation.

A failing run of our architecture test will look like this:

```
=== RUN   TestPackageDependencies
    arch_test.go:41: Error: package dependency not allowed. Dependency chain:
        github.com/fleetdm/fleet/v4/server/android/service
        	github.com/fleetdm/fleet/v4/server/fleet
        		github.com/fleetdm/fleet/v4/server/mdm/nanomdm/mdm
--- FAIL: TestPackageDependencies (14.66s)
```

## Further reading

- In the previous article, we discussed
  [how to scale your codebase with evolutionary architecture](../scaling-codebase-evolutionary-architecture/).
- Before that, we [explained the difference between Go modules and Go packages](../go-modules-and-packages/).

## Watch how to find package dependencies of a Go package

{{< youtube yIZcbTQvpCE >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
