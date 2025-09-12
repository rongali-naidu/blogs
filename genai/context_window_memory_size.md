# When the normal Laptop has Gigabytes of RAM — Why LLMs Only Handle 20k Tokens

When I first started working with large language models (LLMs), one thing puzzled me. Models advertised context windows of **20k or 32k tokens** — just a few dozen pages of text. My immediate reaction was:

> *“But my laptop has 16 GB of RAM. If normal computers can handle gigabytes of memory, why are LLMs restricted to such a small window?”*

That question led me down a rabbit hole. The answer has less to do with system RAM and more to do with how transformers — the architecture behind modern LLMs — process sequences. Along the way, I also learned why both input and output tokens contribute to the same context memory, how to tell whether input or output is causing overflow, and what practical tricks help avoid it.

This post shares what I learned in the process. As you might guess, I leaned on LLMs themselves to understand the limitations of context windows.

---

## What Is Context Overflow?

Every LLM has a **context window**, which is the maximum number of tokens it can process at once. Tokens are chunks of text — usually 3–4 characters, roughly ¾ of a word.

The context window must fit:

* **System and developer prompts** (hidden and explicit instructions).
* **Conversation history** (all prior user and assistant turns).
* **Supporting data** (documents, schemas, query results).
* **The new user input**.
* **And the model’s output itself**.

If the total number of tokens exceeds the model’s limit (e.g., 20k, 32k, or 200k in modern models), you get **context overflow**.

---

## Why Is the Context Window So Small Compared to RAM?

At first glance, it feels wrong — if a laptop handles gigabytes of RAM, why should an LLM choke on a few megabytes of text?

The reason lies in how **transformers** process input:

1. Each token is converted into a **high-dimensional vector** (often 4,096 floats).
2. The model uses **self-attention**, comparing *every token with every other token*.
3. Memory and compute costs grow **quadratically** with sequence length:

O(n<sup>2</sup> × d)



   where $n$ = number of tokens, $d$ = embedding size.

So:

* 10k tokens → 100 million comparisons.
* 100k tokens → 10 billion comparisons.
* Multiply by dozens of layers → memory explodes.

That’s why context is small compared to system RAM — it’s not about storage, it’s about the **compute and memory cost of attention**.

---

## A Surprising and Puzzling Point: Why Input and Output Tokens Both Count

One of the most counterintuitive aspects of context overflow is this: **both the tokens you feed into the model and the tokens it generates count toward the same context memory**, and there’s a specific reason for this.

Unlike humans, LLMs generate text **one token at a time**, and each new token is predicted based on **everything that has come before**, including:

1. The **original input** (system prompt, instructions, chat history, etc.)
2. The **tokens already generated in this response**

Formally, at each step $t$, the model computes:

P(y<sub>t</sub> | x, y_<sub><t</sub>)


Where:

* $x$ = all input tokens
* $y_{<t}$ = all previously generated tokens

This means the model must keep the **input tokens in memory for the entire generation process**. They are not “used up” or discarded after reading — each next token is conditioned on the full input.

### Example

* Suppose you are using a **20k-token model**.
* Your input (system + chat history + user query) = 10k tokens.
* When the model generates the first output token, it considers all **10k input tokens**.
* When generating the second output token, it considers all **10k input tokens + the first output token**.
* This continues until either the model reaches the maximum context length or finishes generating.

So the context memory is **shared and cumulative**: input tokens are continuously “active” while generating output, which is why both input and output contribute to the context window.

This is why even a modestly long input can drastically limit how much output the model can generate before hitting the token limit.

---

## How to Tell Whether Input or Output Is the Problem

When context overflow happens, it helps to diagnose whether **input** or **output** caused it:

* **Input too large:** Model fails immediately — no response at all. Prompt (history + instructions + data) already exceeded the limit.
* **Output too large:** Model starts responding but cuts off mid-generation. Input fit, but combined input + output exceeds context.

You can check this practically:

* Tokenize the prompt before sending it (libraries like `tiktoken` for OpenAI).
* Set `max_output_tokens` to limit output size.

