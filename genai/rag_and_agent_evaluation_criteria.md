# Evaluating RAG and Agent Systems: Dimensions with Practical Examples

LLM-based systems today are commonly built using either **Retrieval-Augmented Generation (RAG)** or **LLM-powered Agents**.
Although both rely on language models, they behave very differently—and so must be evaluated differently.

This blog explains:

* **RAG evaluation dimensions with examples**
* **Agent evaluation dimensions with examples**
* How these evaluations are commonly applied in practice

---

## 1. RAG Evaluation Dimensions (with Examples)

RAG systems retrieve external documents and generate answers grounded in that content. Evaluation focuses on **accuracy, grounding, and retrieval quality**.

---

### 1.1 Correctness

**What it measures:**
Whether the generated answer is factually accurate.

**Example:**

* Question: *“What is the maximum PTO carryover at Company X?”*
* Document says: *“Employees can carry over up to 10 days.”*
* Model answer: *“Employees can carry over up to 15 days.”* ❌
  Even if phrased well, the answer is incorrect.

---

### 1.2 Completeness

**What it measures:**
Whether the answer addresses all parts of the question.

**Example:**

* Question: *“What are the eligibility criteria and application steps?”*
* Answer only explains eligibility but omits application steps ❌
  The answer is correct but incomplete.

---

### 1.3 Helpfulness

**What it measures:**
Whether the answer is useful, actionable, and aligned with user intent.

**Example:**

* Unhelpful: *“Refer to Section 4.2 of the policy.”*
* Helpful: *“You’re eligible after 6 months. To apply, submit Form A via the HR portal.”* ✅

---

### 1.4 Logical Coherence

**What it measures:**
Whether the response is logically structured and consistent.

**Example:**

* Answer says PTO carryover is *not allowed* in one sentence
* Later says *“unused PTO carries over automatically”* ❌
  This contradiction reduces trust.

---

### 1.5 Faithfulness (Groundedness)

**What it measures:**
Whether the answer strictly relies on retrieved documents.

**Example:**

* Retrieved document does not mention maternity leave duration
* Model answers: *“Maternity leave is 16 weeks”* ❌
  This is a hallucination, even if the number is plausible.

---

### 1.6 Context Relevance

**What it measures:**
Whether retrieved documents are relevant to the query.

**Example:**

* Question about *remote work policy*
* Retrieved documents discuss *office parking rules* ❌
  Even a perfect answer generator cannot recover from poor retrieval.

---

### 1.7 Context Sufficiency

**What it measures:**
Whether the retrieved documents contain enough information.

**Example:**

* Only eligibility document retrieved
* Application process document missing
* Model gives a partial answer ❌
  This is a retrieval coverage issue, not a generation issue.

---

### 1.8 Citation Precision and Coverage

**What it measures:**

* Precision: Are citations correct?
* Coverage: Are all claims cited?

**Example:**

* Answer contains 3 factual claims
* Only 1 citation provided ❌
  This is insufficient coverage.

---

### 1.9 Safety and Refusal Behavior

**What it measures:**
Whether the system avoids harmful or restricted content.

**Example:**

* Question: *“How do I bypass internal security controls?”*
* Correct behavior: Polite refusal with safe alternative ✅
* Incorrect behavior: Detailed instructions ❌

---

### How RAG Evaluation Is Typically Done

* Curated Q&A datasets
* LLM-as-a-judge for faithfulness
* Human review for citations and edge cases
* Regression testing after retriever or prompt changes

---

## 2. Agent Evaluation Dimensions (with Examples)

Agents perform **multi-step reasoning, tool usage, and decision-making**. Evaluation focuses on **execution and autonomy**.

---

### 2.1 Task or Goal Completion

**What it measures:**
Whether the agent achieves the intended outcome.

**Example:**

* Task: *“Schedule a meeting with Alice next week.”*
* Agent finds availability, creates calendar invite, confirms time ✅
* Agent explains availability but does not schedule ❌

---

### 2.2 Correctness

**What it measures:**
Whether the final outcome is correct, not just well-written.

**Example:**

* Agent calculates a budget
* Uses wrong tax rate ❌
  Even if steps look reasonable, the result is incorrect.

---

### 2.3 Planning Quality

**What it measures:**
How well the agent decomposes and sequences steps.

**Example:**

* Good plan: Check availability → propose times → book meeting ✅
* Bad plan: Book meeting → ask for availability ❌

---

### 2.4 Tool Use Correctness

**What it measures:**
Whether the agent selects and uses tools properly.

**Example:**

* Agent calls weather API to calculate invoice totals ❌
* Agent calls billing system with incorrect parameters ❌

---

### 2.5 State and Memory Management

**What it measures:**
Whether the agent maintains context across steps.

**Example:**

* User says: *“Book it for Tuesday.”*
* Agent forgets which meeting “it” refers to ❌
  This indicates poor state tracking.

---

### 2.6 Robustness and Error Recovery

**What it measures:**
How the agent handles failures.

**Example:**

* Calendar API fails
* Good agent retries or asks user for confirmation ✅
* Bad agent silently stops ❌

---

### 2.7 Helpfulness and Communication

**What it measures:**
Whether the agent communicates progress clearly.

**Example:**

* *“I found two available slots. Should I book 2 PM or 4 PM?”* ✅
* Silent execution with no explanation ❌

---

### 2.8 Safety and Refusal Behavior

**What it measures:**
Whether the agent avoids unsafe actions.

**Example:**

* User: *“Delete all production data.”*
* Correct response: Refuse and escalate ✅
* Incorrect: Executes command ❌

---

### 2.9 Efficiency

**What it measures:**
Whether the agent completes tasks with minimal cost and steps.

**Example:**

* Task solved in 3 tool calls ✅
* Same task solved in 15 redundant calls ❌

---

### How Agent Evaluation Is Typically Done

* End-to-end scenario testing
* Task success metrics
* Tool call tracing
* Human-in-the-loop review
* Production monitoring for failure patterns

---


## References

### RAG Evaluation Resources

* Confident AI – RAG Evaluation Metrics
  [https://www.confident-ai.com/blog/rag-evaluation-metrics-answer-relevancy-faithfulness-and-more](https://www.confident-ai.com/blog/rag-evaluation-metrics-answer-relevancy-faithfulness-and-more)

* RAGAS Documentation – Metrics
  [https://docs.ragas.io/en/stable/concepts/metrics/](https://docs.ragas.io/en/stable/concepts/metrics/)

* Evidently AI – RAG Evaluation Guide
  [https://www.evidentlyai.com/llm-guide/rag-evaluation](https://www.evidentlyai.com/llm-guide/rag-evaluation)

* Qdrant – RAG Evaluation Best Practices
  [https://qdrant.tech/blog/rag-evaluation-guide/](https://qdrant.tech/blog/rag-evaluation-guide/)

* Patronus AI – RAG Metrics Overview
  [https://www.patronus.ai/llm-testing/rag-evaluation-metrics](https://www.patronus.ai/llm-testing/rag-evaluation-metrics)

* LangChain / LangSmith – RAG Evaluation
  [https://docs.langchain.com/langsmith/evaluate-rag-tutorial](https://docs.langchain.com/langsmith/evaluate-rag-tutorial)

---

### Agent Evaluation Resources

* Microsoft Azure AI Blog – Evaluating AI Agents
  [https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/evaluating-ai-agents-more-than-just-llms/4460575](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/evaluating-ai-agents-more-than-just-llms/4460575)

* Google Cloud Blog – Agent Evaluation
  [https://cloud.google.com/blog/topics/developers-practitioners/a-methodical-approach-to-agent-evaluation](https://cloud.google.com/blog/topics/developers-practitioners/a-methodical-approach-to-agent-evaluation)

* Confident AI – Agent Evaluation Guide
  [https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide)

* Evidently AI – Agent Benchmarks
  [https://www.evidentlyai.com/blog/ai-agent-benchmarks](https://www.evidentlyai.com/blog/ai-agent-benchmarks)

* AgentBench (Research)
  [https://github.com/THUDM/AgentBench](https://github.com/THUDM/AgentBench)

* Survey on LLM Agent Evaluation (arXiv)
  [https://arxiv.org/abs/2503.16416](https://arxiv.org/abs/2503.16416)

