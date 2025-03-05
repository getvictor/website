+++
title = "Top code complexity metrics every software dev should know"
description = "Ways to measure software code complexity to improve code quality and readability"
authors = ["Victor Lyuboslavsky"]
image = "code-complexity-headline.png"
date = 2025-03-05
categories = ["Software Development"]
tags = ["DevTools", "Linting", "Developer Experience", "Engineering Management", "Technical Debt"]
draft = false
+++

_This article is part of a series on **technical debt**. Check out the previous articles:_

- [Ways to improve your code for readability](../readable-code/)
- [How to scale a codebase with evolutionary architecture](../scaling-codebase-evolutionary-architecture/)

## Intro to code complexity metrics

- [Code style](#code-style)
- [Code size](#code-size)
- [Cyclomatic complexity](#cyclomatic-complexity)
- [Cognitive complexity](#cognitive-complexity)

In the previous article on [readable code](../readable-code/), we discussed a few metrics for measuring unreadable code.
In this article, we will expand on some of those ideas and specifically focus on code complexity.

Code complexity primarily refers to the difficulty of understanding a piece of code or a piece of the codebase, such as
a module. Complex code is difficult to modify because engineers must spend considerable mental energy to understand it.
Frequently, engineers will not understand the code well enough, so they'll make a change to fix a bug, and the change
will introduce a new bug somewhere else. Lack of understanding also leads to
[a fear of refactoring](https://victoronsoftware.com/posts/common-refactorings/#why-are-engineers-afraid-of-refactoring),
because engineers don't want to break the codebase.

## Code complexity metrics

Many code complexity measures overlap since they all try to measure the same thing.

### Code style

A standard code style is helpful for readability. For example, if I opened a file and saw that it had no indentation,
the max line length was 20, and somebody named all the variables with a leading `iwuzhere`, I would be confused. I would
have to stop and carefully process the file. I would not have to slow down if the code style were consistent.

The metric to track is the number of code style violations or the number of files violating the code style. Most
companies enforce a code style with their CI pipeline. Modern tooling can automatically reformat code to match the
agreed-upon code style, so code style should no longer be a complexity or readability issue.

### Code size

How much code is there? The more code there is, the longer it takes to read and understand it. The common metrics are:

- program size or lines of code (LOC)
  - in a function
  - in a file
- number of functions/classes/modules/files

The motivation for tracking these metrics is to help engineers split their functions/files/projects into smaller, more
manageable pieces. James Lewis from ThoughtWorks said that "a microservice should be as big as my head." His idea is
that one person should be able to understand the entire codebase. The smaller the piece of code, the easier it is to
understand.

[Halstead introduced a set of software complexity measures](https://en.wikipedia.org/wiki/Halstead_complexity_measures)
in 1977, and one of his metrics was the Halstead volume, which is directly related to code size. We can approximate the
Halstead volume by ignoring all comments and whitespace, then multiplying the average code line length by the number of
lines of code. This approximation is a good enough metric for our purposes.

### Cyclomatic complexity

[Cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity) measures the number of linearly independent
paths through a program's source code. It is often used as the master metric for code complexity, uncovering
maintainability and hard-to-test parts of the codebase.

A typical calculation of cyclomatic complexity is as follows:

- 1 is the base complexity for a function
- for each `if`, `case`, `while`, `for`, or other branching statement, add 1

A good cyclomatic complexity is 10 or less. A score of 20 or more is generally considered difficult to understand. This
metric encourages us to write smaller functions.

### Cognitive complexity

An alternative to cyclomatic complexity is
[cognitive complexity](https://www.sonarsource.com/docs/CognitiveComplexity.pdf). This metric tries to adjust the
cyclomatic complexity metric to focus on the human reader's mental load -- on the maintainability, and not on the
testability, of the code.

The key differences in the calculation are:

- for nested structures, extra incremental penalties are added
- recursion is penalized
- jumps to labels, such as `goto LABEL`, are penalized
- `switch` is preferred over nested `if`
- groups of similar logical operators are NOT penalized
  - for example, `a && b && c && d` is easier to understand than `a && b || c && d`

This metric is more difficult to calculate than cyclomatic complexity, but it is generally considered a better
approximation of code complexity. Many companies are adopting this metric.

## Tool and language-specific considerations

Modern tools can help with code maintainability issues. For example, AI tools that index the codebase can help explain
how a piece of code (or a feature) works. IDEs can also help by collapsing boilerplate code or improving readability in
other ways.

In the Go programming language, the idiomatic way to check for errors is:

```go
if err != nil {
    return err
}
```

The above code is repeated everywhere and is typically collapsed by modern IDEs. However, cyclomatic complexity and
cognitive complexity metrics penalize it.

We need a complexity tool where the user can adjust the penalties. This way, an engineering team can agree on what is
considered complex code based on their experience, language, and code style.

### Go complexity metrics

For measuring cyclomatic complexity, Go has [gocyclo](https://github.com/fzipp/gocyclo). For measuring cognitive
complexity, there is [gocognit](https://github.com/uudashr/gocognit).

## Further reading

- Recently, we wrote [an overview of using AI in software development](../ai-for-software-developers/).

## Watch us discuss and show examples of code complexity metrics

{{< youtube HzZQqrhX3cg >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
