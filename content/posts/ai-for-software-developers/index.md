+++
title = "How to use AI for software development (2025)"
description = "What every software engineer needs to know about AI right now: code completion, generation, and AI agents"
authors = ["Victor Lyuboslavsky"]
image = "ai-building-software.png"
date = 2025-02-26
categories = ["Software Development"]
tags = ["AI", "Developer Experience"]
draft = false
+++

The AI landscape is changing rapidly. A new tool seems to come out weekly, or we hear about a breakthrough. As a
software engineer, it is hard to keep up with all the latest developments and even harder to figure out what is relevant
to your day-to-day work. Many tech leaders claim that AI is helping them achieve greater engineering velocity, improving
the quality of their code, and enhancing the engineering experience, to name a few. But how can you leverage AI to
improve your work as a software engineer? And, most importantly, what is the most effective way to use AI right now?

We have followed AI since the early 2000s, when Jeff Hawkins published
[On Intelligence](https://en.wikipedia.org/wiki/On_Intelligence). AI has come a long way since then. Today, most
software engineers can benefit from some AI.

The top AI use cases for software engineers are:

- [Code completion](#code-completion)
- [Code generation](#code-generation)
- [Summarizing or explaining code](#summarizing-or-explaining-code)
- [Agentic coding](#agentic-coding)

Some software developers have other use cases for AI, such as image generation, but this article will focus on the above
ones.

## Code completion

The original GitHub Copilot popularized AI code completion. It is a tool that can be integrated into your IDE and can
perform multi-line code completions. GitHub Copilot and similar tools have been extremely popular with software
developers. We found AI code completion extremely helpful, even though it occasionally generates function signatures and
other code that is not quite right.

For example, after creating the `PubSubMessage` struct in our code, Copilot recommended the `PubSubMessageAttributes`
struct. Opening the completions window shows other multi-line suggestions.

{{< figure src="copilot-example.png" title="GitHub Copilot code completion example">}}

The code completion is not only for software code. We used it for documentation, such as writing a user guide for a new
feature. AI is most useful for boilerplate introductory content. However, beware that some search engines may be able to
detect the use of AI in your documentation and reduce your search ranking.

## Code generation

Code generation is similar to code completion, but the desired result is a complete function or code block. Typically,
engineers use a chat interface such as ChatGPT and provide it with a description of the desired function. Providing a
detailed description along with inputs and outputs is key to getting the best results.

For example, given the following prompt:

> Write a Golang function that takes path.URL, makes a GET request to that URL, parses the JSON response, and returns
> the "attributes" and "data" fields of the response.

ChatGPT o1 model generated the following code:

```go
// FetchAttributesAndData makes a GET request to the provided URL, parses
// the JSON response, and returns the "attributes" and "data" fields.
func FetchAttributesAndData(u *url.URL) (map[string]interface{}, interface{}, error) {
    resp, err := http.Get(u.String())
    if err != nil {
        return nil, nil, err
    }
    defer resp.Body.Close()

    // You may also check resp.StatusCode for error handling if desired
    if resp.StatusCode != http.StatusOK {
        return nil, nil, fmt.Errorf("unexpected status code: %d", resp.StatusCode)
    }

    // Define a struct to capture the JSON structure
    var result struct {
        Attributes map[string]interface{} `json:"attributes"`
        Data       interface{}            `json:"data"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, nil, err
    }

    return result.Attributes, result.Data, nil
}
```

ChatGPT also provided some documentation and a usage example.

Previously, software engineers would accomplish this task by searching Google or StackOverflow for similar code
snippets. Now, they can use AI chatbots to generate this code. This approach is most useful when you need to generate
code that's not your area of expertise.

Some programming languages have dedicated AI code-generation tools. For example, you can create Go functions and
packages online with [sketch.dev](https://sketch.dev/).

## Summarizing or explaining code

When software engineers work on a large codebase, they often need to understand code written by other engineers, some of
whom may have left the company, and they may also need to find a specific piece of code in the codebase.

AI can help by reading the codebase and adding it to its context. This is an example of Retrieval-Augmented Generation
(RAG), where AI combines its general knowledge with the specific context of the codebase.

To understand the codebase, the AI tool needs to add the code to its context or to index the codebase. Then, the tool
can use a combination of AI and deterministic search to find the relevant code. Below is an example of a question we
asked [Cursor](https://www.cursor.com/) IDE about our codebase:

> In which Go files is macOS disk encryption code located?

Cursor provided a list of files and additional information on the top hits.

{{< figure src="cursor-example.png" title="Cursor codebase search example">}}

Codebase-wide queries are a great way to find information yourself before asking the broader team. They are also a
decent way to learn the codebase.

## Agentic coding

Agentic coding refers to using an AI agent to write code and perform tasks on your behalf. Using agents is a more
advanced use case, requiring you to know the AI tools, processes, and LLMs well. A good AI agent can:

- Write code, including creating and moving files
- Write and run tests, including Browser UI tests
- Write, read, and follow documentation
- Do terminal operations such as installing applications
- Do Git operations such as pushing
- Connect to other servers with SSH

Currently, the top agentic coding tools are:

- [Roo Code](https://github.com/RooVetGit/Roo-Code) (VSCode plugin)
- [Cline](https://cline.bot/) (VSCode plugin)
- [Cursor](https://www.cursor.com/) (IDE built on top of VSCode)

There are many other tools and platforms available. GitHub Copilot also announced Agent mode, which is available in
preview as of this writing. JetBrains has announced [Junie](https://www.jetbrains.com/junie/), which is only available
via the Early Access Program.

As the agentic coding tools are still in their early stages, changing rapidly, and require a lot of handholding, it is
reasonable to wait 6 to 12 months before revisiting them.

### AI coding agent workflows

The following are some workflow suggestions for using an AI coding agent to create a small application.

First, start with some context regarding what you want to build. Create a README or a plan outlining how you want to
structure the application and the steps to implement it. You can use another general-purse AI, such as ChatGPT, to help
you create the high-level plan.

For example, we asked ChatGPT to create a high-level plan with the following prompt:

> We want to create a mock Google Android Management API server using an AI agent. The server is written in Golang and
> will interact with our MDM solution during testing. It should hold the state for enterprises, profiles, and fake
> devices enrolled in it. The server should have a mock PubSub webhook that will push notifications regarding ENROLLMENT
> and STATUS_REPORTs. Please create a plan that another AI agent can implement in several steps.

Next, ask the AI agent to read the plan, update it, and create a more detailed plan. It may make sense to break the plan
into smaller parts and treat each part as a separate project. In effect, you act as the AI agent's project manager.

Make sure to have documentation and have the AI agent update it regularly. In addition to the README, you can have API
specs, secrets, and other documentation files.

Tell the AI agent to initialize a git repo, create the project structure, and start implementing the plan. For each
step, ask the AI agent to create tests. After each step, ask the AI agent to update the documentation and commit the
changes. This way, you can easily rollback if the AI agent gets stuck or goes off the rails.

Try to be as precise as possible in your prompts.

When adding a new feature, you can start a new session with the AI agent and ask it to read all the documentation. This
will "initialize" the AI agent with the project context.

Work in small development iterations with your AI agent.

{{< figure src="ai-agent-workflow.png" title="AI agent workflow">}}

Learn about your AI agent's specific features to level up your skills. Often, there are ways to provide context to the
agent or give special meaning to certain words or files.

At some point, you may want to take over the maintenance of the code from the AI agent. For example, check the code into
your main repository and maintain it as any other human-written code.

### AI coding agent issues

The main issue with AI coding agents is that they make mistakes. If you spot their mistake, you can tell them about it,
and they will generally correct it. However, if you can't spot their mistake, the agent may end up in a loop where it
keeps trying to fix the issue, but your application still doesn't work. That is why it is essential to work in small
iterations where you can roll back and start over.

The other issue is that AI agents are slow. Often, they need to take several steps to make progress, and the human is
left waiting—being there just in case they need help or go off track. Theoretically, a single human could manage
multiple AI agents, but in practice, it is hard for people to frequently switch between multiple cognitively demanding
tasks.

## Further reading

- For a deeper look at AI agents' impact on software development, check out [Will AI agents replace software developers?](../will-ai-agents-replace-developers/)
- We just published [our lessons learned from building an AI-driven MCP server](../introducing-mcp/).
- We recently discussed
  [how to scale your codebase with incremental design](../scaling-codebase-evolutionary-architecture/).
- We also wrote about [the importance of readable code](../readable-code/).

## Watch how to use AI for software development

{{< youtube PVdqwcwc2_o >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
