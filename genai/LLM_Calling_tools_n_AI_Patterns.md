
# Demystifying “LLMs Calling Tools” and Agentic AI Patterns

Over the past year, there’s been a lot of excitement—and confusion—around LLMs (Large Language Models) “calling tools” and the different patterns of agentic AI systems. 
After diving into multiple resources  like [Anthropic’s guide](https://www.anthropic.com/engineering/building-effective-agents)and other blogs on this subject,  I want to share two key clarifications that reshaped my understanding.

---

## 1️LLMs Don’t Call Tools—They Prescribe Them

When I first heard that LLMs can call tools, my immediate thought was:

> How are permissions and credentials managed? How does the LLM authenticate itself to invoke APIs or run commands?

The truth is, **LLMs never actually execute tools**. They only **prescribe which tools to call and the arguments for those calls**. The execution happens in a separate layer—commonly called the **agent, middleware, or workflow layer**—which has access to credentials, API keys, or system functions.

In other words, the process looks like this:

```
User → LLM (decides which tool + arguments) → Agent/Middleware (executes tool with proper permissions) → Tool → Result → LLM → User
```

This distinction is critical: the LLM is **advising**, not **acting**. All security, authentication, and execution logic lives in the middleware. This explains why tools can be safely exposed to LLM-driven workflows without the LLM ever needing direct access to credentials.


## All Patterns Are Variations of the Same Core System

Across the literature, there are many names and patterns for how LLMs are used:

* Prompt Chaining, Routing, Parallelization, Orchestrator-Workers, Evaluator-Optimizer
* Basic Responder, Router, Tool Calling, Multi-Agent, Autonomous
* ReAct, Planning, Multi-Agent  ,Reflection, Tool Use,     



At first glance, this diversity feels overwhelming. But at the core, they all describe the **same architecture**:

1. An **LLM** that generates suggestions, tool calls, or reasoning steps.
2. A **middle layer** (agent/middleware/workflow) that mediates between the LLM and the external world—executing tasks, validating inputs, managing context, and enforcing security.

The differences are mostly about **how much logic, orchestration, validation, and reasoning you put into this middle layer**. For example:

* A **basic responder** or **simple tool calling pattern** is almost a one-shot execution—LLM suggests, middleware runs.
* A **multi-agent or autonomous pattern** involves loops, dynamic planning, and multiple LLM calls across steps, but the principle remains: LLM advises, middleware executes.


