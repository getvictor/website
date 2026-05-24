+++
title = "Comprehension debt: the hidden cost of AI-generated code"
description = "AI coding agents can ship working code before the team has built the mental model to maintain it. That gap is comprehension debt."
authors = ["Victor Lyuboslavsky"]
image = "comprehension-debt-headline.png"
date = 2026-05-24
categories = ["Software Development"]
tags = ["AI", "Agentic AI", "Developer Experience", "Engineering Management", "Technical Debt"]
draft = false
+++

AI coding agents can ship working code before the team has built the mental model to maintain it. I think of that gap as
comprehension debt, and it behaves differently from the technical debt engineers are used to.

## What it is

Comprehension debt is the distance between code that runs and code the team can still reason about.

It is not the same as messy code. Messy code announces itself. You open the file, and the pain is obvious. Comprehension
debt is sneakier. It can sit inside code that looks clean, passes tests, and ships without drama. The danger is that
nobody can explain why it works, what assumptions it depends on, or what is likely to break when it changes.

AI did not invent maintainability problems. Engineers were writing sloppy, hard-to-maintain code for decades before AI.
What AI changed is the speed and scale. It is now much cheaper to produce code that looks finished before anyone has
really understood it.

{{< figure src="code-iceberg.png" title="Comprehension debt hides below the surface of code that compiles and ships">}}

## How it differs from classic tech debt

Classic tech debt usually has visible friction. The code is tangled. The tests are brittle. The team knows the area is
painful. Comprehension debt hides longer. The code looks serviceable. The tests pass. The feature ships. But the team
accepted the change without deeply understanding it. Future work depends on a mental model the team did not build.

The cost usually shows up in behavior before it shows up in code:

- Nobody wants to touch a module unless they have to.
- A tiny change triggers a huge amount of regression testing because nobody trusts the blast radius.
- Design discussions turn into archaeology, searching commit messages, review threads, and old plans to infer what the
  system was supposed to do.
- A one-line change needs a senior engineer, a full regression run, and a long Slack thread before anyone feels safe
  merging it.
- Work that should take an hour quietly stretches much longer.

The codebase still works. The team just no longer moves through it with confidence.

## A small example

I ran into this recently on a disk encryption feature in our product. Our key escrow flow was taking multiple hours,
which was too long. On the surface, it looked like a small bug hunt.

I did what a lot of engineers would do now. I asked the agent to investigate first. It came back with root causes and
possible fixes, but they did not hold up. Some did not fit the behavior I was seeing. At one point the agent was blaming
the customer's environment, which was a bad read of the situation.

So I did the real work myself. I read the docs. I tried the product. I traced the flow through the code. I used the
agent as a helper for local understanding, to summarize files, explain branches, and point me toward likely paths. But I
was still reading much of the implementation myself and building the mental model on my own. It still took hours. In the
end, I found that another module was touching disk encryption on a schedule. The fix was a documentation update. The
system was behaving as designed, but that design lived in the code and in engineers' heads, not in documentation.

That investigation produced almost no code. But it paid down debt the team did not realize it had taken on.

## Inquiry vs delegation

There is a healthy way to use AI and a dangerous one.

- **Inquiry** is using AI to ask questions, explore unfamiliar code, compare paths, and build a mental model faster.
- **Delegation** is using AI to make the problem go away without understanding it.

Both can produce working code. Only one produces understanding.

A [January 2026 Anthropic study](https://www.anthropic.com/research/AI-assistance-coding-skills) found that developers
using AI scored 17 percent lower on a comprehension quiz after learning a new Python library. Digging into the data,
people who used AI to ask conceptual questions tended to do much better. People who delegated the work and skipped the
mental model did much worse. Inquiry built understanding. Delegation bypassed it.

An [April 2026 JetBrains study](https://blog.jetbrains.com/research/2026/04/ai-impact-developer-workflows/) adds a
related warning. Developers reported that AI improved their code quality and readability. The behavioral signal the
study used (debugging sessions started, an imperfect proxy for quality) did not show the same improvement. What
developers feel is improving and what their workflow shows may not always line up.

When the mental model shrinks while code volume grows, the bottleneck does not go away. It gets worse.

## Who pays the bill

If something goes wrong in production, the model does not get blamed. A human does. Humans run the postmortems. Humans
explain what happened and what should change next. So humans still need understanding, even if they did not write every
line by hand.

{{< figure src="human-and-robot-pointing-fingers.png" title="When something goes wrong in production, humans are still the ones who get blamed">}}

AI making code generation cheaper does not change that. It just makes it easier for the team to accumulate work it
cannot defend.

AI can also make the debt feel safer for a while. It can summarize unfamiliar code, propose edits, run tests. That
helps. But safety is not the same as understanding. If the team still cannot explain what changed and what might break
next, it has not paid down the debt. It has only delayed the bill.

## What to do about it

The fix is not "review more carefully" or "work harder." That does not scale. It was already hard enough when humans
wrote all the code by hand. The fix is making sure understanding gets built somewhere in the workflow before the code
lands. A few things that help:

- Use small phases. Do not let the agent run further than you can review.
- Write the implementation plan with the agent, then read it fully before any code lands.
- Use AI for inquiry, to summarize files and explain branches, not to make the problem go away.
- Before merging, require a short behavioral summary: what changed, what stayed invariant, what side effects exist, and
  what might break next.

Generation got cheaper. Comprehension did not. The teams that move fastest will be the ones that notice the difference.

## Further reading

- **[Will AI agents replace software developers?](../will-ai-agents-replace-developers/)**  
  A realistic look at where AI coding agents help across the SDLC and where human judgment still has to do the work.

- **[Multitasking with AI agents: When it works and when it fails](../multitasking-with-ai-agents/)**  
  How to balance focus and parallel agent work without losing the context that makes review meaningful.

- **[What is readable code and why is it important?](../readable-code/)**  
  Why code that is easy to understand is also code that is safe to change, with or without AI in the loop.

## Watch the full talk

This article covers one idea from a longer conference talk on writing maintainable code with AI. The talk also gets into
PR review, behavioral contracts, AI-generated tests, and architecture fitness functions.

{{< youtube 7iQthHou_4M >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
