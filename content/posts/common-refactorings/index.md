+++
title = "Top code refactorings every software engineer should know"
description = "Overcoming the fear of refactoring: How small, smart changes improve code without risk"
authors = ["Victor Lyuboslavsky"]
image = "common-refactorings-headline.png"
date = 2025-02-12
categories = ["Software Development"]
tags = ["Golang", "Technical Debt"]
draft = false
+++

- [Extract method (aka extract function)](#extract-method-aka-extract-function)
- [Inline variable](#inline-variable)
- [Extract variable](#extract-variable)
- [Inline method (aka inline function)](#inline-method-aka-inline-function)

## What is code refactoring?

Code refactoring involves modifying existing software code without changing its observable behavior. Refactoring improves the code base's readability and maintainability. See our previous article on [why readable code is important](../readable-code/).

## Why are engineers afraid of refactoring?

Refactoring is essential to software development and should be done regularly as part of day-to-day work. Unfortunately, many engineers are afraid of refactoring, don't know how to do it, or don't consider it part of their job responsibilities.

Some engineers fear that refactoring will introduce bugs or break existing functionality. The root cause of this fear is the lack of automated tests. Without automated tests, ensuring that the refactored code behaves as expected is difficult. A code base without automated tests is a ticking time bomb and cannot be maintained by any sane engineer. Before refactoring code in such a code base, you should add automated tests for the targeted code.

Other engineers fear that refactoring will take too much time. This fear is often unfounded, as refactoring can be done incrementally and in small steps. For example, after refactoring the code for an hour or less, merge your changes to your main branch and, if needed, continue doing the next small refactoring steps. Your organization should never allocate weeks of development for "large refactorings."

Engineers may also fear refactoring because they don't want to make too many changes to the code, making it difficult for reviewers to review the changes. The issue is that many current code review systems don't understand the code changes' semantics (i.e., the meaning). These systems only understand line changes and are frequently confused by relocated code. In this case, the coder should explain the changes to the reviewer. Alternatively, the organization can adopt a better code review tool. For a further discussion of [issues with GitHub code reviews, see our previous article](../github-code-review-issues/).

This article will show some common refactorings you can safely do with automation from your IDE (Integrated Development Environment).

## Extract method (aka extract function)

The `extract method` refactoring takes a piece of code and moves it into a new method. There are several reasons to do this, all of which improve the readability and maintainability of the code:

- The code is too long and must be broken into smaller, more manageable pieces.
- The code is duplicated in multiple places and needs to be consolidated into a single method.
- We want to separate the code implementation from the code intention. The code implementation is what the code does, and the code intention is why it does it. Move the code implementation into its own method and name the new method based on the code intention.

For example, consider the following code:

```go
func (pt PackageTest) expandPackages(pkgs []string) []string {
    if !slices.ContainsFunc(pkgs, func(p string) bool {
       return strings.Contains(p, "...")
    }) {
       return pkgs
    }

    // lots more code ...
```

When entering the `expandPackages` method, the reader is immediately confronted with a complex expression. They must stop and think about what the code does. Even though the amount of code is small, it still hampers readability. The code implementation is mixed with the code intention. One way to improve the situation is to add a comment. A better way is to extract the code into its own method and name the new method based on the code intention.

```go
func (pt PackageTest) expandPackages(pkgs []string) []string {
    if !needExpansion(pkgs) {
       return pkgs
    }

    // lots more code ...
}

func needExpansion(packages []string) bool {
    return slices.ContainsFunc(packages, func(p string) bool {
       return strings.Contains(p, "...")
    })
}
```

Most IDEs automatically perform this refactoring. Highlight the code you want to extract, open the refactoring menu, and select the `Extract Method` option.

## Inline variable

Every variable should have a purpose and a good explanatory name that describes its intent. As the number of variables grows in a method, it becomes increasingly difficult to understand the code. One way to improve the readability of the code is to inline variables. Inlining a variable is replacing the variable with the right-hand side of the assignment.

For example, consider the following code:

```go
func (pd *packageDependency) chain() (string, int) {
    name := pd.name

    if pd.parent == nil {
       return name + "\n", 1
    }

    // lots more code ...
```

The variable `name` does not add any value to the code. It is simply a copy of the `pd.name` field. We can inline the variable to improve the readability of the code:

```go
func (pd *packageDependency) chain() (string, int) {
    if pd.parent == nil {
        return pd.name + "\n", 1
    }

    // lots more code ...
```

Many IDEs automatically perform this refactoring. Highlight the variable you want to inline, open the refactoring menu, and select the `Inline` option.

## Extract variable

The `extract variable` refactoring takes a complex expression and moves it into a new variable. Mechanically, it is the opposite of the above `inline variable` refactoring. There are several reasons to do this:

- The expression is complex and must be broken into smaller, more manageable pieces.
- The meaning of the expression is unclear and needs to be clarified with a descriptive variable name.

Sometimes, you have a choice between extracting a method or a variable. In general, you should extract a method to make the code more readable. However, if the method is only used once and the parent function is not complex, it may be better to extract a variable.

For example, consider the same code from our `extract method` example above:

```go
func (pt PackageTest) expandPackages(pkgs []string) []string {
    if !slices.ContainsFunc(pkgs, func(p string) bool {
       return strings.Contains(p, "...")
    }) {
       return pkgs
    }

    // lots more code ...
```

We can extract the complex expression into a variable to improve the readability of the code:

```go
func (pt PackageTest) expandPackages(pkgs []string) []string {
    needExpansion := slices.ContainsFunc(pkgs, func(p string) bool {
       return strings.Contains(p, "...")
    })
    if !needExpansion {
       return pkgs
    }

    // lots more code ...
```

Many IDEs automatically perform this refactoring. Highlight the expression you want to extract, open the refactoring menu, and select the `Extract Variable` or `Introduce Variable` option.

## Inline method (aka inline function)

The `inline method` refactoring takes a method and moves its code into the caller. Mechanically, it is the opposite of the `extract method` refactoring. There are several reasons to do this:

- The method is too simple, and its body is as clear as its name.
- We want to simplify code and remove a level of indirection.
- We must regroup code into a single method before proceeding with a better refactoring.

For example, consider the following code:

```go
    // method code ...
    if dep.has(expandedPackages) {
    // more coe ...

func (pd *packageDependency) has(pkgs []string) bool {
    return slices.Contains(pkgs, pd.name)
}
```

We can inline the `has` method:

```go
    // method code ...
    if slices.Contains(expandedPackages, pd.name) {
    // more coe ...
```

Many IDEs automatically perform this refactoring. Highlight the method call you want to inline, open the refactoring menu, and select the `Inline Function/Method` option.

## Further reading

- The code examples above are from our article on [finding package dependencies of a Go package](../go-package-dependencies/).
- We also discussed [how to scale your codebase with evolutionary architecture](../scaling-codebase-evolutionary-architecture/).
- And [how to analyze Go build times](../analyze-go-build/).

## Watch examples of top code refactorings

{{< youtube Mzj9XQlieHk >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
