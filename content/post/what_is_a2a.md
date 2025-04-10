---
title: "A First Look at Agent-to-Agent (A2A)"
date: "2025-04-10"
toc: true
#author: "Kevin Hoffman"
type: post
description: "Taking a look at Google's A2A and how it impacts developers"
subtitle: "What's in it for me?"
categories: 
    - "a2a"
    - "ai"
    - "google"
---
_A look at what A2A is, how it works, and why it matters_

<!--more-->

Roughly two days ago (April 8th, 2025), Google introduced [A2A](https://google.github.io/A2A/#/). A2A is Google's brand new open protocol for supporting collaborative and client/server communication between AI agents.

I think it is a bit telling (and somewhat frightening) that this protocol was unveiled to the public just two days ago and I've had so many questions on it already I feel compelled to write a blog post. Back before the war, when it was just microservices and dinosaurs, we had weeks or months of lead time before a _"standard"_ was worth digging into.

First, it's worth re-iterating that **A2A** is not **MCP** nor is it a competitor to it. I'm oversimplifying here, but think of MCP as a way of plugging arbitrary tools into a human-in-the-loop agent session. While A2A can support humans, the protocol is designed for agent-to-agent comms.

## Overview
As mentioned, A2A enables communication and collaboration between agents. At the low level, this is done using HTTP, SSE, and JSON-RPC (which is, coincidentally, the same set of building blocks MCP uses).

The first thing we get from A2A is discovery. It defines a JSON format for an _Agent Card_. Agent cards are metadata about the agent, which is treated very much like an enterprise app or service. This metadata includes name, description, longer documentation, the agent's URL, the authorization scheme, and, most importantly, the list of **_skills_**.

At this point it would be really easy to confuse A2A's _skills_ with MCP's _tools_. If you haven't been down that rabbit hole yet, MCP's _tools_ are a list of functions and expected parameters that an LLM can use to match _intent_. In other words, the LLM decodes the user's intent to use a function and it uses MCP to advance the agent flow.

> **tl;dr**: _MCP_ allows for tool-level function calling using structured parameters, while _A2A_ allows agents to communicate with each other using natural language prompts. 

A2A _skills_ have a name, description, tags, and a definition of the input and output _modes_ (mime types). A2A skills aren't invoked because an LLM discovered intent. They're invoked because an agent determined that it needs to utilize another agent (which, of course, could be _informed by_ the execution of LLM prompts). Where MCP tools accept structured data as input (as decoded by the LLM), A2A skills actually accept _prompts_ like "what is the weather in New York, NY?" or "What is the status of Kevin Airlines flight 12?"

## Uses
Let's say you're building a training companion or assistant. The user enters a prompt like the following:

> _"Set up and route my daily run using my previous running history to pick my next "road to 5k" exercise."_

The "my previous running history" part of the prompt makes it pretty obvious that we're going to start a RAG flow for this. The agent could just draw out a route made up of GPS waypoints, but this is the age of the AI assistant. We can do way more for the customer than that, and A2A can help us.

There are a couple of things we can do to enhance the running route planner. We might envision a sequence like this (invoked by calling a service that fronts our agent, this is _not_ local):

1. Get the LLM to generate the GPS run based on the RAG context.
1. Ask the **`weather`** agent: _"What is the weather for 90210 today"_. The agent returns both text _and_ structured data as a separate payload. We can extract the hourly temperature forecast.
1. Ask the same agent, in the same session: "What is the UV index for today in this location?"
1. Ask the **`road_conditions`** agent if there is any construction along the GPS route.
    1. If so, generate a new GPS route (return to step 2)
1. Ask the LLM _"What is the optimal time to go for a run based on the hourly temperature, humidity, and UV index in this context"_
1. Return all of the run information so the app can accept it

One of the first things we notice here is that A2A supports _multi-turn_ conversations with these remote agents. The other is that these agents are logically named. The protocol doesn't require it, but it assumes that one good way to use it is to use DNS to look up the endpoint for a named agent, and then pull its `.well-known/agent-card.json` file to get the metadata.

In the above flow, **`weather`**, and **`road_conditions`** are A2A servers discovered and used by the client. This flow shows the simplest way to interact via A2A--by sending a task and getting a result. The protocol is built around the notion of potentially _very_ long-running, asynchronous tasks. In this way, we could spin off and orchestrate any number of sub-tasks (which in turn could also use A2A to spin up more sub-tasks).

As such, not only can we hit a route that will give us task-specific SSE data, but we can also tell remote agents a URL they should hit to report status changes and progress. We can also cancel any long-running tasks we started by supplying the task ID.

If we expand the scope of our thinking a bit, we can see how an agent having a long-running, durable, distributed conversation with a human can result in the creation, management, and completion of all kinds of agents and sub-flows. Google's [example](https://google.github.io/A2A/#/topics/a2a_and_mcp?id=example) shows extensive conversations with a customer that potentially involve other humans, agents, and tools.

Because of this, A2A feels more to me like a distributed task orchestration protocol than just a simple JSON-RPC protocol. I think we will likely see tooling integration in the near future that supports A2A-based workflows and task management. Of course, the protocol is literally only 48 hours old, so there's no telling if the protocol is here to stay or not, especially given Google's history of abandoning standards, products, and services.

## Security Concerns
The documentation says that A2A supports the [OpenAPI](https://swagger.io/docs/specification/v3_0/authentication/) specification for authorization (not to be confused with **OpenAI**). The _agent card_ specifies which authorization schemes the service supports such as `Basic` or `Bearer`.

The protocol punts on the means of token acquisition, login, etc and assumes that those tasks will be done out of band. So if the **`road_conditions`** service requires an API key, you'll have to acquire that yourself and then ensure that when your agent calls it, it supplies the right bearer token.

I think the thing that worries me about A2A is also a concern I have about MCP. It can be _really_ easy to accidentally leak PII into the context of a long-running conversation with an LLM, and an agent acting on your behalf can easily leak your PII without knowing it. Now imagine an agentic flow that's orchestrating a number of nested sub-tasks. We may not know how many of them might have captured the PII.

Sure, we can do everything in our power to ensure that the context we retrieve for RAG doesn't get sent to the other agents (that's actually a primary goal of this protocol), but if a human accidentally puts PII into their own question, we can't really know that.[^1]

## Pluggable Agents
I've seen some conversation out on the interwebs about A2A making it so you can support pluggable agents. In other words, if I write an agent that consumes _a_ **`road_conditions`** service, I could always swap the provider of that service later without changing my code.

I don't think that's a practical assumption or something we really want to do. Since agents on the other side of an A2A link are accepting natural language and they are returning both text _and_ structured data, we'll end up relying on the schema of that structured data. This schema will vary from provider to provider. Further, prompts are very finicky, and the results we get from two different providers with the same prompt text could vary wildly. 

What I think is far more likely to happen is that A2A-based agents become like _agentic microservices_, and an enterprise will build any number of these and applications will be built by composing traditional (hopefully _event-sourced_) application components with agents.

## What to Do
As I said in my [Demystifying AI, LLMS, and RAG](https://akka.io/blog/demystifying-ai-llms-and-rag) video, _don't get distracted_. This protocol has been public for less time than a mayfly typically lives. It's good to keep an eye on the evolution of these things, and experiment in spare time _if you have it_. But this isn't something (yet) that should be causing seismic shifts in your roadmap, though it deserves some investigation slots in your strategy.

---
[^1]: We could always use another model to analyze the prompt for PII risk 😊



