+++
title = "6 lessons from literate programming"
description = "What can the literate programming approach teach today's software developers"
authors = ["Victor Lyuboslavsky"]
image = "literate-programming-headline.png"
date = 2025-03-12
categories = ["Software Development"]
tags = ["Developer Experience", "Technical Debt", "Hello World"]
draft = false
+++

This article examines the literate programming paradigm introduced in 1984 by
[Donald Knuth](https://en.wikipedia.org/wiki/Donald_Knuth). We go through a "Hello World" example and extract the key
lessons relevant to making today's software more readable and maintainable.

- [Literate programming example](#literate-programming-hello-world-example)
- [Key takeaways from literate programming](#key-takeaways-from-literate-programming)

## What is literate programming

Literate programming is a paradigm in which a computer program is written in a natural language, such as English. The
programming language source code is embedded into the program's description. The aim was to create an artifact that a
human can easily read without jumping back and forth between different sections of the code file. The writer completely
controls the flow of the document, which can be reorganized in any fashion.

Knuth called his implementation of literate programming [WEB](<https://en.wikipedia.org/wiki/Web_(programming_system)>)
to emphasize that a computer program is built from many different pieces. He picked the name before the World Wide Web
was prominent. To produce source code, the user runs the **tangle** command. To create documentation, the user runs the
**weave** command.

## Literate programming "Hello World" example

### Writing a literate program

To demonstrate literate programming, we will use the [noweb](https://github.com/nrnrnr/noweb) literate programming tool
to write a simple program in Go.

We create a `hello.nw` file and start it with:

```
This program teaches us how to print to the screen using:
<<print>>=
fmt.Println(message)
@

To print "Hello World", pass a literal string to the function:
<<message>>=
"Hello World"
@
```

We wrote the program in text with embedded code starting with `<<name>>=` and ending with `@`. The `<<name>>` sections
are macros that we can reuse in other sections of the document, such as:

```
Now, we can create a function that prints a message:
<<mypackage_print>>=
func Print(message string) {
    <<print>>
}
@

Finally, we can call this function from the main function:
<<main_call>>=
mypackage.Print(<<message>>)
@
```

See [the complete literate program](https://github.com/getvictor/noweb_example/blob/main/hello.nw) on GitHub.

### Generating code and documentation from the literate program

To install the noweb tool with `brew` on macOS, run the following:

```bash
brew install noweb
```

To generate the Go source code (tangle):

```bash
notangle -Rgo.mod hello.nw > go.mod
mkdir -p mypackage
notangle -R'mypackage/mypackage.go' hello.nw > mypackage/mypackage.go
notangle -Rmain.go hello.nw > main.go
```

Now we can run the program: `go run main.go`

To generate the HTML documentation (weave):

```bash
noweave -html hello.nw > hello.html
```

We can open the `hello.html` documentation in our web browser.

## Key takeaways from literate programming

### 1. The developer orders the code for maximum readability

Most of today's programming languages were not designed with readability as their top guiding principle. They often
require the developer to put code in specific file sections, distracting the reader trying to understand the code. Some
examples include:

- imports
- function and variable declarations, including nested functions
- error handling

Today's IDEs (Integrated Development Environments) have tried to help with the situation by automatically collapsing
boilerplate sections. However, we have not seen them take the next step of entirely hiding or virtually relocating
distracting code. This area is where today's programming languages and IDEs need to improve.

### 2. Comments are first-class citizens

In literate programming, comments (natural language) are the main body of the program. Source code, on the other hand,
is delegated to macros. Comments are easy to write and can be enhanced with additional processing, such as Markdown,
Mermaid diagrams, etc.

Many of today's language toolchains also have processors that generate HTML documentation from the comments. However,
none can mix arbitrary pieces of code with their documentation.

Linting requirements to include comments often lead to meaningless comments that make the code less readable:

```go
// This is a class.
```

Today, the closest mainstream approaches to literate programming are computational notebooks such as
[Jupyter](https://jupyter.org/) and various online tutorials. These are great for sharing examples and small programs
with others but not sufficient for larger software projects.

Some IDEs support rendering comments in a different style than the rest of the code, including rendering diagrams.
However, no standard works across IDEs and version control hosting systems like GitHub.

### 3. Code from multiple source files can be present in one place

Literate programming allows us to include arbitrary source files in one program file. This behavior is helpful when you
want to keep related code in one place, such as an interface (abstract class) and its implementation.

Modern IDEs can find all the implementations of an interface and often have a quick shortcut, allowing the developer to
jump between the two.

Developers may also write their own preprocessors that split a single file into several modules or compile units.

Although having more code in one file is sometimes useful, today's developers typically have issues with splitting and
decoupling code files that have become too large and are no longer scalable.

### 4. Writing code is more difficult

The main issue with literate programming is that it makes writing code much more difficult for the developer. It
introduces another level of abstraction and another set of tools and concepts that the software developer must be
familiar with.

This general lesson applies to any system that tries to enhance the coding experience by adding another layer between
the user and the code. The new system must provide overwhelming benefits for software developers to switch to it.
TypeScript is an example of a successful layer over JavaScript.

### 5. Macros make reading code more difficult

The literate program contains macros with their own names, adding to the namespace of functions and variables already
present in the computer program. These additional names increase the cognitive load of both reading and creating
literate programs.

Today's standard guidance is to make your variable and function names descriptive so the reader knows what they do
without additional comments. We can effectively replicate much of literate programming by replacing the literate
programming macros with our own well-named functions and ordering these functions in a file for maximum comprehension.

### 6. No tooling support

Since literate programming is not widely used, it has little to no tooling support, syntax highlighting, or IDE support.
There is also no standard build system. Instead, the literate programming user must maintain their own custom build
system for the "tangle" and "weave" flows.

## See literate programming example code on GitHub

- [Literate programming example using noweb](https://github.com/getvictor/noweb_example)

## Further reading

- Previously, we explained [what readable code is and why it is important](../readable-code/).
- We also reviewed [the top code complexity metrics](../code-complexity-metrics/).

## Watch the literate programming example and takeaways

{{< youtube 8cwxxioVbfA >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
