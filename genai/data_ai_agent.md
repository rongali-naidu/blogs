# **AI-Powered Data Analysis: From Text-to-SQL to Multi-Tool Agents**

In my [previous post](https://medium.com/@rongalinaidu/basic-text-to-sql-for-amazon-athena-metatadata-enrichment-8fb41c019ee5), I built a simple **Text-to-SQL** pipeline for Amazon Athena.
It worked like this:

* User asked a question in plain English.
* The system retrieved metadata from the Data Catalog.
* I used an LLM to turn the question + metadata into SQL.
* SQL was executed in Athena.
* Results were returned to the user.

It was a great proof of concept.
But it had one limitation: it was *single-purpose*.
What if tomorrow I wanted the same chatbot to:

* Query Redshift instead of Athena?
* Pull metrics from QuickSight?
* Send an alert to Slack?

That’s where **Agentic AI** and **MCP (Model Context Protocol)** come in.

---

## **What’s a Data AI Agent?**

An **AI Agent** is more than just an LLM.
It’s the *orchestrator brain* that:

1. Understands the user’s request.
2. Decides what steps to take.
3. Uses the right tools or APIs at the right time.
4. Combines results into a final answer.

In a data analysis scenario, the Agent might:

* Fetch metadata from a Data Catalog.
* Ask the LLM to Generate the SQL.
* Execute it in Athena.
* Ask the LLM to Analyze the results.
* Return a clear, business-friendly answer.

The LLM is **part** of the Agent — it’s the reasoning engine.
But the Agent also knows *when not to think* and instead *go do something* (like run a query).

---

## **Enter MCP: Model Context Protocol**

If the Agent is the brain, **MCP Servers** are the hands and tools.
They provide real-world capabilities the Agent can use.

If this sounds like an **API server**, you’re not wrong — it is.
But MCP is designed specifically for AI-driven agents, and that makes it special.

---

### **MCP Server vs API Server**

| Feature             | Traditional API Server                    | MCP Server                                                  |
| ------------------- | ----------------------------------------- | ----------------------------------------------------------- |
| **Purpose**         | Expose functionality for software clients | Expose functionality for AI Agents                          |
| **Tool Discovery**  | Developer reads docs, hardcodes endpoints | Agent can dynamically list tools and their parameters       |
| **Protocol**        | HTTP/REST, gRPC, custom                   | MCP protocol (WebSocket, stdio)                             |
| **Schema**          | Custom per API                            | Standardized JSON schemas for tools, inputs, outputs        |
| **Adaptability**    | Client logic must be coded per API        | Any MCP Client can use tools without special code           |
| **AI Friendliness** | Not context-aware for LLMs                | Designed to be discoverable and usable by LLM-driven agents |

---

## **The Data AI Agent Flow**

Here’s how a **Data AI Agent** could work for business data analysis:

1. **User Interaction**

   * User types a question in a **chatbot UI** (browser app).
   * The chatbot sends the request to the Agent.

2. **Agent Orchestration**

   * Agent decides:

     * “I need metadata context.” → Calls **Data Catalog MCP Server**.


3. **Query Generation**

   * Agent sends:

     * User’s question
     * Schema from Data Catalog MCP Server
     * Context from Knowledge Base
       to the LLM.
   * LLM returns Athena SQL.

4. **Query Execution**

   * Agent calls **Athena MCP Server** to run the SQL.
   * MCP Server executes the query in Athena and returns data.

5. **Data Analysis**

   * Agent sends query results to the LLM for summarization and insight extraction.

6. **Response**

   * Agent formats the output and sends it back to the chatbot UI.

---

## **Architecture Overview**

```
[Browser Chatbot UI]
        │
        ▼
[MCP Client]
   └─ Agent (Orchestrator)
       ├─ LLM (Claude/ChatGPT)
       ├─ Knowledge Base (RAG)
       ├─ MCP Server (Data Catalog - AWS Glue)
       └─ MCP Server (Athena Queries)
```

**Key roles:**

* **MCP Client** — the environment hosting the Agent and connecting to MCP Servers.
* **Agent** — decides when to use LLM vs MCP tools.
* **LLM** — handles reasoning, SQL generation, and analysis.
* **MCP Servers** — provide standardized access to services like Glue and Athena.

---

## **Why MCP Fits This Pattern**

With MCP:

* Adding a new tool (e.g., QuickSight dashboard generation) is as easy as connecting another MCP Server.
* The Agent doesn’t need new code — it just discovers the new tool and starts using it.
* You can replace Athena with another query engine without touching the Agent logic.

In short: **MCP gives AI Agents plug-and-play superpowers**.

