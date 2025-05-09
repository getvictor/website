+++
title = "Introducing MCP: Lessons from building an AI-driven server"
description = "How we taught an AI agent to use our product with MCP"
authors = ["Victor Lyuboslavsky"]
image = "introducing-mcp-headline.png"
date = 2025-05-08
categories = ["Software Development"]
tags = ["AI", "Agentic AI", "DevTools"]
draft = false
+++

Recently, we developed a proof of concept using an MCP(Model Context Protocol) server to control
[Fleet's device management server](https://fleetdm.com/). This server made a successful
[demo video](https://youtu.be/eKoOvT-fr2I). The takeaways from the video were:

- 🧠 Natural language understanding — no commands to memorize
- 🤖 Actionable context awareness — the AI remembers what you're asking about
- 🔗 Tool selection and orchestration — chooses the proper action automatically

In this article, we'll talk about our process for integrating our API with agentic AI, mistakes made, and lessons
learned.

## What is MCP, and why do I need one?

MCP (Model Context Protocol) is a client-server framework that enables AI agents to interact with your tools and APIs
through natural language. MCP was introduced by [Anthropic](https://www.anthropic.com/news/model-context-protocol) in
November 2024. Its primary use case was to use local tools available on your local machine, such as databases.

But, some may ask why we need another framework—aren't AI agents good enough to use those tools directly? Yes, it is
true that by providing AI agents with instructions on how to use an arbitrary tool, the AI agent should be able to use
it. However, MCP standardizes this approach. Thus, an MCP server created for one tool can be reused by many developers
and by many AI agents.

Instead of telling your AI agent about every tool, imagine that the AI agent has access to a library of MCP servers that
it can install whenever you want to "teach" it about a tool. Think of MCP as an intelligent middleware that allows AI
agents to understand and use your system's capabilities appropriately. With MCP, the AI can dynamically learn and adapt
to your system's interface.

### The MCP protocol

The MCP protocol consists of several key components:

1. Tools: These are functions or actions that can be performed. Tools are defined by name, description, and parameters
   they accept. They represent the capabilities your system provides to AI agents.

2. Resources: These are data objects that tools can operate on. Resources have properties and relationships with other
   resources. For example, a `User` resource might have properties like name and email.

3. Prompts: These are reusable prompt templates and workflows intended to standardize common LLM interactions.

4. Schema Definition: The protocol uses a schema to describe available tools and resources, making it easy for AI agents
   to understand possible actions.

The main distinction between tools and resources is that tools are actions (verbs), while resources are objects (nouns)
on which these actions may operate. For example, a `createUser` tool might operate on a `User` resource. In practice,
however, you can create a `User` tool that returns a user and not worry about associating it with a resource. The
semantics of tools are simpler than those of resources, so it might be faster only to use tools if your goal is to get
something working quickly.

## Building an MCP server with agentic AI

Our biggest mistake was asking our AI agent ([Roo Code](https://roocode.com/)) to build an MCP server without fully
understanding the MCP details ourselves. To summarize, this process was not successful.

MCP has SDKs available for multiple languages. We decided to build our MCP server with TypeScript. Our AI agent pulled
in the right [TypeScript MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk). However, it couldn't get the
compile to work, so it removed the SDK and implemented the MCP protocol from scratch. We allowed our agent to proceed
since we didn't know any better.

After implementing a basic tool, we couldn't get the MCP server to work reliably. It worked using `curl`, but the
[MCP inspector](https://github.com/modelcontextprotocol/inspector) debug tool couldn't connect to our MCP server.
Another mistake we made was not telling the AI agent to use liberal debug messages up front, which would have sped up
the debug effort.

After letting our AI agent waste our API credits for too long, we told it to rewrite the code using the TypeScript SDK.
We copied and pasted the examples that our agent could reference. After that, things went smoothly.

Another engineer suggested that using the Python SDK would have been more effective and required less handholding.

Our proof of concept [MCP server for Fleet API](https://github.com/getvictor/fleet-mcp) is on GitHub. Here's a code
example for the `install_software` tool:

```typescript
// Register the install_software tool
this.mcpServer.tool(
  'install_software',
  'Install software on a host managed by Fleet',
  {
    host_id: z.string().describe('Required. The host ID'),
    software_id: z.string().describe('Required. The software title ID')
  },
  async (params: { host_id: string; software_id: string }) => {
    try {
      console.log(`Installing software ID ${params.software_id} on host ID ${params.host_id}`);

      const url = `/api/v1/fleet/hosts/${params.host_id}/software/${params.software_id}/install`;
      const response = await this.axiosInstance.post(url);
      console.log('Fleet API install request successful');

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(response.data, null, 2),
          },
        ],
      };
    } catch (error) {
      console.error('Fleet API error:', error);
      throw {
        code: 'internal_error',
        message: `Fleet API error: ${error instanceof Error ? error.message : String(error)}`,
      };
    }
  }
);
```

As you can see above, the code is pretty straightforward. We're simply wrapping an existing API call with an MCP tool.

### Overall impressions

AI agent interactions with our MCP server are complex to test and challenging to debug. First, as an MCP server
developer, you must be familiar with AI agents, MCP protocol, and the tool interface you're trying to use. For example,
in one case, the AI agent didn't use the proper parameter to call the tool and came up with wrong conclusions. Since we
are dealing with AI, it is hard to tell how often such mistakes will happen and also hard to fix or prevent them. Also,
we don't know exactly how customers may use the MCP server, so the QA test space is quite large.

Our product already has an API, a CLI, and a UI interface. Adding MCP to the mix feels hard to justify—are customers
going to use all these ways to interact with our product? For a small product team, investing in MCP support might not
make sense until a customer is ready to pay for it.

## Further reading

We recently covered [what every software engineer needs to know about AI right now](../ai-for-software-developers/).

## Watch how we taught an AI agent to use our product with MCP

{{< youtube 1RLh3SpfytQ >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
