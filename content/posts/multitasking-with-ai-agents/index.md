+++
title = "Multitasking with AI agents: When it works and when it fails"
description = "A guide to balancing focus and efficiency when working with AI agents, including tips on single-tasking vs multitasking"
authors = ["Victor Lyuboslavsky"]
image = "multitasking-with-ai-agents-headline.png"
date = 2025-08-19
categories = ["Software Development"]
tags = ["AI", "Agentic AI", "Developer Experience", "Engineering Management"]
draft = false
+++

AI agents are changing the way we work. They can handle coding, debugging, and research tasks while we focus on other
things. But just because an AI can work in parallel does not mean you should. The real question is: when is it worth
multitasking, and when will it hurt your results?

I'm a firm believer that humans cannot multitask. Every popular psychology book I have read, and I have read quite a
few, says that humans cannot multitask, even though many believe they can. Studies show that when humans do two tasks in
series, they finish faster and with higher quality than when switching between them.

But that's not the whole story. Humans can multitask in particular situations. For example, when driving a car and
talking on the phone (with a Bluetooth headset, of course), we are clearly multitasking. Would our driving and
conversational quality be better if we did these tasks separately? Probably. But the difference, depending on the
person, might not be that big. Driving is habitual for many people, so we can pair it with another cognitive task
without dramatically dropping performance.

What about throwing AI agents in the mix? It is Wednesday morning. You told your AI agent to do a job. You hope it will
only take one minute, but it might take up to 15 minutes. What do you do now?

_Note:_ We will use the term "task" to refer to work a human does, and "job" to refer to work an AI agent does.

## Task switching

Research from the American Psychological Association suggests that switching between tasks can cost up to 40% of
someone's productive time. A University of Michigan study found it takes an average of 23 minutes and 15 seconds to
fully return to a task after an interruption. Even brief mental blocks from shifting between tasks can cost significant
time.

So, if your AI agent is running, can you do something else while waiting to check the results? If you switch to another
task, such as looking at another bug or feature, you will spend time ramping up on that second task, and then even more
time switching back to the first. Yes, people do it all the time, but you will not be fully engaged in either task. You
are more likely to miss something important, like the agent using the wrong method or skipping a critical feature
requirement. And yes, that happens in real life.

## When not to task switch

Avoid switching when working on a significant, important feature or bug. You need your full mental capacity in the
context of that task to deliver the best software.

Instead of switching, consider these options.

### Plan ahead

After the agent completes its job, what will you do? Will you check the results and then start the next phase? Is that
next phase directly related to the current one? Can you spin up a parallel environment and have another agent start the
next step?

If waiting for the agent is your bottleneck, parallelize jobs or ensure the agent is always working on something useful.
Keeping the agent busy may mean reviewing the agent's work while it jumps ahead and works on the next phase.

### Make sure the agent is efficient

Watch the agent's progress to make sure it is not doing unnecessary work. If it is trying to find information you
already know, stop it and provide the answer. If you do not know the answer, you may find it faster than the agent.

For example, as of August 2025, Claude Code does not create a vector database of the codebase, making it slower for
certain queries about the codebase. For example, asking where a specific feature is tested. You might ask Cursor, which
does a semantic index of the codebase, to find the answer faster and then feed the answer back to Claude Code.

Agents can go off track. Many use brute-force approaches, repeatedly trying solutions until something works, even if the
"working solution" is not optimal. By monitoring progress, you can redirect the agent when needed.

### Prepare for manual QA

Once the feature is ready for manual testing, is your environment ready? Do you need to populate database data, deploy
other services, or sign up for cloud accounts? Prepare these ahead of time.

### Fix CI and code review issues

Commit regularly, ideally after each agent job run. This lets you run CI checks and even have AI review your code. Fix
any lint issues or failing tests, and address review feedback early.

## When to task switch

Task switching is fine for small, low-risk work, such as:

- Fixing a single failing unit test
- Resolving a simple bug
- Addressing straightforward code review comments
- Minor refactoring
- Adding logging statements

These tasks require little mental setup, so switching costs are minimal. Any quality issues are easier to spot.

## Work modes: single-tasking vs multitasking

Categorize tasks before starting them to decide whether they need full attention or can run in parallel.

| Aspect                | 🎯 Single-Task Mode                | 🔄 Multitask Mode                     |
| --------------------- | ---------------------------------- | ------------------------------------- |
| **Focus Level**       | 🧠 Deep focus required             | 😌 Light mental load                  |
| **Task Types**        | 🏗️ Complex features                | 🔧 Minor refactoring                  |
|                       | 🚨 Production bug fixes            | 📝 Code review comments               |
|                       | 🏛️ Architecture decisions          | ✅ Simple bug fixes                   |
|                       | ⚡ Performance optimization        | 📊 Adding logging                     |
| **Context Switching** | ❌ Avoid at all costs              | ✅ Switch freely                      |
| **Agent Strategy**    | 🎪 One or more agents on same task | 🎭 Multiple agents on different tasks |
| **Monitoring**        | 👁️ Watch progress closely          | 📱 Check periodically                 |
| **Risk Level**        | ⚠️ High - mistakes costly          | 🟢 Low - easy to spot issues          |
| **Mental Setup**      | 🏔️ Significant ramp-up time        | 🏃 Quick to start                     |
| **Quality Impact**    | 💎 Quality critical                | 👍 Quality easier to verify           |
| **Best For**          | 🚀 Mission-critical work           | 📋 Routine repeatable work            |

**Single-task mode** is for deep-focus work. Run one or more AI agent jobs focused on this task. Monitor progress while
planning ahead or preparing test environments. Keep your context intact.

**Multitask mode** is for routine, well-defined work. Run multiple AI agent jobs in parallel across different branches.
Switch freely between these smaller tasks as agents finish.

## Further reading

- **[AI for software developers](../ai-for-software-developers/)**  
  Explore how AI tools are changing developer workflows and when to embrace AI assistance versus maintaining human
  control.

- **[Will AI agents replace software developers?](../will-ai-agents-replace-developers/)**  
  Understand the evolving role of developers in an AI-driven world and why human judgment remains irreplaceable.

## Watch us discuss multitasking with AI agents

{{< youtube HSDrhYvE3g0 >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
