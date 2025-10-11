
# Understanding AWS Q Developer, Q CLI, CodeWhisperer, and Kiro

Amazon (via AWS) now offers multiple AI-assisted tools for developers. To make sense of them, here’s a comparison of **Q Developer**, **Q CLI**, **CodeWhisperer**, and the newer **Kiro** — what they do, how they relate, and where they differ.

---

## Amazon Q Developer — The Unified AI Assistant

**Amazon Q Developer** is AWS’s main generative AI assistant for software developers. It works in your IDE, via chat, in the AWS Console — and its capabilities include:

* Generating, explaining, or refactoring code
* Debugging / diagnosing AWS errors
* Helping with architectural decisions, resource usage, security, etc.
* Acting as a “knowledge-aware” assistant around AWS services

Under the hood, Q Developer is powered by foundation models via **Amazon Bedrock**, and it layers AWS-specific knowledge, safety rules, and context integration on top.
It also inherits all the AI coding features that the previous **CodeWhisperer** tool used to provide (which are now merged into Q).

---

## Q CLI — Bring Q into the Terminal

**Q CLI** (Amazon Q for the command line) is the terminal interface to Q Developer’s capabilities. In your shell, you can:

* Chat with Q naturally (e.g. “create an S3 bucket and list contents”)
* Get autocomplete suggestions for `git`, `aws`, `docker`, etc., based on context
* Translate natural-language instructions into actual shell commands
* Use “agentic behaviors” — e.g. writing files, reading outputs, integrating with MCP servers
* Command utilities like `q login`, `q doctor`, `q chat`, etc.

The key point: **Q CLI and Q Developer share the same AI brain** (the same underlying LLMs via Bedrock). They are just different interfaces into that same system — one is GUI/chat/IDE-based, the other is CLI-based.

Because both rely on Bedrock for model infrastructure, features like model choice (e.g. Claude Sonnet 4, Sonnet 3.7) apply across both.
Additionally, Q supports Model Context Protocol (MCP) integrations, which let you plug in external knowledge sources or data services (docs, DBs, APIs) so Q can reason with more context.

---

## CodeWhisperer — The Legacy Component

**CodeWhisperer** was AWS’s original AI code assistant, focused primarily on inline completions inside IDEs. Over time, as Q Developer matured, CodeWhisperer’s core features were integrated into Q. It no longer exists as a separate product — everything it did lives in Q Developer now.

---

## Kiro — AI-Native Agentic IDE (Spec-Driven Development)

**Kiro** is AWS’s newer offering: an **agentic, spec-first IDE** built to go beyond mere autocomplete. It’s designed to help you move from **concept → spec → code → production** with more structure and control. ([Kiro][1])

Here’s a breakdown of what Kiro brings to the table, and how it relates to (or differs from) Q Developer / Q CLI.

### What is Kiro?

* Kiro is an **AI-native IDE** (built on a fork of VS Code / Code OSS) that emphasizes **spec-driven development**. 
* Rather than jumping immediately into code, Kiro starts by turning prompts into structured artifacts:

  1. **Requirements / specs** (user stories, acceptance criteria)
  2. **Design / architecture** (data models, APIs, module structure)
  3. **Tasks / implementation plan** (code tasks, tests)
  4. Then the **agents execute** the plan, generate code, tests, docs, etc. 
* Kiro also supports **agent hooks**: triggers that react to file events (save, changes) or other signals to run background tasks (e.g. generate tests, update docs). 
* It uses **Model Context Protocol (MCP)** integrations: you can plug in external systems (APIs, databases, docs) to give the IDE more context. 


### How Kiro relates to Q Developer / Q CLI — similarities & differences

| Aspect                           | Commonalities                                                                                     | Differences / Unique Features                                                                                                                                                  |
| -------------------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Backend & models**             | Kiro also uses foundation models (e.g. Claude) under the hood. It is built to integrate with MCP. | Unlike Q, Kiro is not just a chat or code assistant — it’s an **IDE with agents and structured workflows**. Q is more general-purpose; Kiro is focused on structured dev flow. |
| **Interface**                    | Both are developer-centric tools tied to coding / dev workflows.                                  | Q Developer is UI / chat / IDE plugin; Q CLI is terminal. Kiro is a full IDE environment (like VS Code, but augmented).                                                        |
| **Scope & purpose**              | All aim to boost developer productivity using AI.                                                 | Q is broad (AWS, cloud, code, architecture, chat). Kiro is narrower (agentic code + project workflows).                                                                        |
| **Integration with AWS / cloud** | Q is deeply integrated with AWS, tailored to AWS users.                                           | Kiro is intended to be more cloud-agnostic. But AWS sees it as complementary to Q (Q for cloud tasks, Kiro for building). ([GeekWire][7])                                      |

So, while Q Developer / Q CLI and Kiro may overlap in coding support, Kiro introduces a higher-level structured workflow with specs, agents, and lifecycle automation that complements (rather than replaces) Q.

