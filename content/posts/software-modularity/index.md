+++
title = "6 business benefits of software modularity and cohesion"
description = "Why modularity and cohesion are important for building modern scalable software systems"
authors = ["Victor Lyuboslavsky"]
image = "software-modularity-headline.png"
date = 2025-03-19
categories = ["Software Development"]
tags = ["Software Architecture", "Developer Experience", "Technical Debt"]
draft = false
+++

_This article is part of a series on **technical debt**. Check out the previous articles:_

- [Why readable code is essential](../readable-code/)
- [Incrementally scaling a codebase with evolutionary architecture](../scaling-codebase-evolutionary-architecture/)
- [Top software code complexity metrics](../code-complexity-metrics/)

## Introduction

Modularity and cohesion are key software engineering concepts that software engineers frequently misunderstand.
Frequently, software developers know that modularity is good in some vague general sense, but they can't quantify the
benefits. This lack of understanding leads to poor decisions when adding new functionality or fixing system bugs, often
leading to the proverbial big ball of mud codebase. Sometimes, engineers feel like it is their manager or software
architect's job to define the modules of a system, and they are simply responsible for implementing the details in the
quickest way they know how.

This article provides
[specific reasons why using or adding modularity to your existing growing codebase is a good business decision](#business-benefits-of-a-modular-codebase).
Sometimes, engineers intuitively know how to do a good job, but they can't explain it to business stakeholders. We try
to explain.

## It's all about complexity

One of the challenging problems in software engineering is managing complexity. Modularity is a tool for managing
complexity. When we speak of complexity, we are referring to a mature, growing codebase with 10 or more software
developers. A young codebase with a couple of developers will benefit from modularity but not as much as a bigger, more
complex codebase.

Bigger organizations must manage the complexity of both people and systems. People must be able to work independently at
maximum velocity without being slowed down by others. Systems must be simple enough to think about without being
overwhelmed. Complexity increases the cost of ownership of software because:

- engineers cannot move at maximum velocity due to coupling to other teams or parts of the codebase
- engineers cannot understand the code, leading to slow development and more bugs

### The "hero engineer" anti-pattern

Complex codebases often have "hero developers" who know the codebase and become the go-to people for solving critical
issues or implementing complex features. The prevalence of such heroes may be a sign that your codebase is too complex
and that you must change things to scale your business.

The "hero engineer" may be contributing to the complexity issue by:

- focusing on quick fixes rather than long-term solutions
- insisting that the current codebase is just fine because "that's how we've always done it"

## What is modularity and cohesion

Modularity is the process of breaking up a complex system into smaller, independent, and interchangeable modules. Small
means small enough to easily understand. Independent means that we can compile and test the module independently of all
the other modules. Interchangeable means that we can substitute other implementations of modules in our system, which
often happens during testing.

Cohesion refers to the degree to which the functionality inside a module belongs together. It is the metric we use for
creating modules. A good module has high cohesion. Logical changes in that module should generally not leak to other
modules. This metric means that a module comprising random functions is not a good one.

## Business benefits of a modular codebase

### 1. Faster development and easier maintenance

Modularity increases engineering velocity, which in turn lowers labor costs and reduces time to market. Since modules
are cohesive and easy to understand, engineers focus their changes on a limited set of modules without wading through
the whole codebase.

Since modules are independent and interchangeable, they are easier to test. Writing tests becomes faster and easier.

### 2. Risk reduction

A frequent occurrence in a complex codebase is that a change in one place introduces a bug in another seemingly
unrelated functionality. Because modules are cohesive and independent, a bug in one module is less likely to bring down
the entire system. Also, since modules are more straightforward to test, they are less likely to have bugs in the first
place.

### 3. Organizational scalability

Management expects the software output to scale proportionally as the engineering organization scales. However, this is
not always the case. Adding more people to a codebase causes merge conflicts, ownership confusion, duplicated effort,
and communication overheads.

Modularity allows engineers to work in parallel. Each person or team can work in parallel on their modules, minimizing
organizational coupling.

The main reason microservices have become so popular is that engineers can work on them in parallel. The organizational
scalability benefits outweigh the added complexity of microservices.

### 4. Faster onboarding

When new developers join the team, the engineering velocity often dips as senior developers help with onboarding.

Since modules are small, they can "fit in your head" without having to understand all other parts of the system. New
developers can contribute more quickly by focusing their initial contributions on a limited set of modules. This
"simplicity" of the codebase means fewer distractions and less hand-holding for senior staff.

### 5. Flexibility

Modularity gives the business more options for the product's future direction since small modules are more
straightforward to modify or replace with new functionality.

Modularity also allows engineers to experiment with newer and potentially better approaches. For example, an engineer
can try a different JSON library on one module. Engineering management would not consider such a library change in a
monolithic codebase since it would pose too much risk to existing functionality.

### 6. Professional growth for software engineers

What about the engineers who have been with the company for years and are comfortable with (or used to) the current
monolithic approach?

Modular software architectures are becoming the norm in the software industry. Building and maintaining a genuinely
modular codebase provides a valuable experience that engineers can carry to future projects within and beyond the
current company. If we interviewed a candidate whose preferred working style was to minimize the number of modules in
the codebase, that would be a serious red flag.

## Downsides of a modular codebase

### 1. Initial module creation overhead

Creating a new module requires defining its interface and directory structure and writing a new test harness. These
steps require more up-front work than simply dumping the code into an existing package.

### 2. CI complexity

We can compile and test modules independently. To maximize the development speed of modules, each one can have its own
CI run. However, as the number of modules grows, this process can become complicated over time.

### 3. Cross-cutting features can become trickier

Some features, such as security and auditing, affect multiple modules. Ensuring consistency while keeping modules
independent can require extra thought and coordination.

## Further reading

- Check out our other articles in the **technical debt** series. Links are at the top of this article.
- Previously, we [explained the difference between Go modules and Go packages](../go-modules-and-packages/).

## Watch us explain the business benefits of software modularity

{{< youtube UmSsmGFufTg >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
