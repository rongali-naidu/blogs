## Understanding LLM Benchmarks

As large language models (LLMs) continue to evolve rapidly, it’s increasingly important to understand **how their capabilities are measured**. Benchmarks provide objective, comparable ways to assess reasoning, coding, tool use, and domain-specific tasks across different model providers like OpenAI, Anthropic, Google, and more.

Knowing these benchmarks helps users:

* **Compare models fairly**, beyond marketing claims.
* **Understand strengths and limitations**, from general reasoning to specialized skills like SQL or multi-step tool use.
* **Make informed decisions** for research, product development, or deployment.

---

### Comprehensive LLM Benchmark List (2025)

#### **1. Classic / Academic Benchmarks**

| **Benchmark**                   | **Capability**                          | **Notes / Adoption**                              | **Link**                                                                    |
| ------------------------------- | --------------------------------------- | ------------------------------------------------- | --------------------------------------------------------------------------- |
| MMLU                            | General knowledge / multitask reasoning | Widely used by major providers                    | [MMLU](https://arxiv.org/abs/2009.03300)                                    |
| GSM8K                           | Math reasoning (grade-school)           | Popular for coding/math reasoning                 | [GSM8K](https://github.com/openai/grade-school-math)                        |
| MATH                            | Advanced math                           | Competition-level problems                        | [MATH](https://github.com/hendrycks/math)                                   |
| ARC / ARC‑AGI‑2                 | Scientific reasoning                    | Multiple-choice; widely cited                     | [ARC](https://allenai.org/data/arc)                                         |
| BBH (Big-Bench Hard)            | Hard reasoning                          | Subset of Big-Bench; emerging SOTA                | [BBH](https://github.com/google/BIG-bench)                                  |
| HellaSwag                       | Commonsense reasoning                   | Scenario-based evaluation                         | [HellaSwag](https://rowanzellers.com/hellaswag/)                            |
| HumanEval                       | Code generation                         | Functional correctness; used by OpenAI, Anthropic | [HumanEval](https://github.com/openai/human-eval)                           |
| MBPP                            | Programming basics                      | Mostly simple coding tasks                        | [MBPP](https://github.com/google-research/google-research/tree/master/mbpp) |
| STEPWISE‑CODEX‑Bench (SX‑Bench) | Complex coding                          | Multi-function / control flow reasoning           | [SX‑Bench](https://arxiv.org/abs/2508.05193)                                |

---

#### **2. Agentic / Tool Use Benchmarks**

| **Benchmark**    | **Capability**             | **Notes / Adoption**                       | **Link**                                                                                        |
| ---------------- | -------------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| AgentBench       | Multi-step agentic actions | Measures autonomous tool use               | [AgentBench](https://arxiv.org/abs/2410.14255)                                                  |
| ToolBench / BFCL | Tool / API usage           | Evaluates function-calling and integration | [ToolBench](https://evalscope.readthedocs.io/en/v0.16.3/get_started/supported_dataset/llm.html) |

---

#### **3. Meta / Evaluation Benchmarks**

| **Benchmark**        | **Capability**             | **Notes / Adoption**        | **Link**                                                                    |
| -------------------- | -------------------------- | --------------------------- | --------------------------------------------------------------------------- |
| TruthfulQA           | Factual accuracy           | Detects hallucinations      | [TruthfulQA](https://github.com/sylvainbarraud/truthfulqa)                  |
| Winogrande           | Coreference resolution     | Contextual understanding    | [Winogrande](https://leaderboard.allenai.org/winogrande/submissions/public) |
| Humanity’s Last Exam | Reasoning / exam-style     | Broad reasoning tasks       | [HLE](https://arxiv.org/abs/2307.12241)                                     |
| MM‑Eval              | Meta-evaluation            | LLM-as-judge / multilingual | [MM‑Eval](https://arxiv.org/abs/2410.17578)                                 |
| MetaBench            | Compact multi-task         | Combines ARC, MMLU, GSM8K   | [MetaBench](https://arxiv.org/abs/2407.12844)                               |
| BiGGen Bench         | Text generation evaluation | Fine-grained evaluation     | [BiGGen](https://arxiv.org/abs/2406.05761)                                  |

---

#### **4. Continuous / Rolling Benchmarks**

| **Benchmark** | **Capability**                    | **Notes / Adoption**                                                                                                                                                                                                                         | **Link**                                                   |
| ------------- | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| LiveBench     | Multi-task / practical evaluation | Rolling / contamination-free; math, code, reasoning; GitHub pipeline supports OpenAI, Anthropic, Google. **Updated regularly to reflect real-world tasks, avoid test contamination, and evaluate models on fresh, multi-domain challenges.** | [LiveBench](https://livebench.ai/)                         |
|               |                                   | Updated monthly for real-world evaluation                                                                                                                                                                                                    | [LiveBench GitHub](https://github.com/LiveBench/LiveBench) |

---

#### **5. Domain-Specific / Specialized Benchmarks**

| **Benchmark** | **Capability**               | **Notes / Adoption**                                       | **Link**                                      |
| ------------- | ---------------------------- | ---------------------------------------------------------- | --------------------------------------------- |
| BIRD‑bench    | Text-to-SQL / database tasks | Multi-turn SQL; used by Google, Tencent, Ant Group, Amazon | [BIRD‑bench](https://bird-bench.github.io/)   |
| BIRD-Critic   | SQL debugging / diagnostics  | Evaluates LLM’s ability to fix SQL errors                  | [BIRD-Critic](https://bird-critic.github.io/) |

