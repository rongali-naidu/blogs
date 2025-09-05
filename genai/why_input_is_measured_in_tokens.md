# Why LLMs Input is measured in Tokens, Not Words — And Why It Matters

We often hear LLM usage limits in **tokens**. So, it’s natural to ask: why can’t they specify limits in terms of words?
This question led me to understand **what a token is**, and in the process, I also discovered that the same text can be converted into tokens in different ways by different LLMs.

> **A token is a number representing a word or part of a word.**
> A token is usually defined as a chunk of text that can represent a single word, a part of a word, or even a punctuation mark. A number is assigned to each chunk.

Sounds straightforward, right? But understanding tokens is key to knowing **why LLMs can’t specify prompt length in words like Word or Notepad**, and why they instead specify **length in tokens**.


## 1. Why Words Alone Don’t Work

Machine learning models operate on **numbers**, not raw text. So, any text we feed into an LLM must be **converted to numbers**.

You might wonder: why not simply assign a **sequence number to each word**?

```
"cat" → 1
"dog" → 2
"house" → 3
```

This is simple approach doesnt work due to:

* Word inflections: run, runs, running, ran
* Grammar variations: different tenses, plurals
* Spelling mistakes: color vs colour

Clearly, we need a **set of rules** to break text into a manageable set of units. This is where **tokenizers** and **vocabularies** come in.



## 2. Tokenizers: Breaking Text into Tokens

A **tokenizer** is a piece of logic that:

1. Splits text into **subwords or words**.
2. Assigns each token a **unique numeric ID** in the vocabulary.

Each LLM uses its **own tokenizer**:

* **ChatGPT / GPT models:** BPE (Byte Pair Encoding, implemented in `tiktoken`)
* **Claude:** SentencePiece

These tokenizers define:

* **Vocabulary:** the known set of tokens (subwords, special tokens)
* **Rules:** how to break new words into existing tokens

> Note: Both tokenizers always map text to **numeric token IDs** internally. Textual token strings are just a human-readable representation for debugging.

---

## 3. Example: Tokenizing the Same Text

Let’s use the text:

```
"ChatGPT is amazing!"
```

#### Using OpenAI GPT Tokenizer (BPE via `tiktoken`)

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")
text = "ChatGPT is amazing!"

tokens_ids = enc.encode(text)  # numeric token IDs
decoded_tokens = [enc.decode([t]) for t in tokens_ids]  # human-readable
print("Token IDs:", tokens_ids)
print("Decoded Tokens:", decoded_tokens)
```

**Example Output:**

```
Token IDs: [85097, 374, 2898, 0]
Decoded Tokens: ['Chat', 'GPT', ' is', ' amazing', '!']
```

---

#### Using Hugging Face Tokenizer

```python
from transformers import GPT2Tokenizer

tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
text = "ChatGPT is amazing!"

tokens_ids = tokenizer.encode(text)
tokens_text = tokenizer.convert_ids_to_tokens(tokens_ids)
print("Token IDs:", tokens_ids)
print("Tokens Text:", tokens_text)

```

**Example Output:**

```
Token IDs: [30820, 38, 11571, 318, 4998, 0] 
Token Text: ['Chat', 'G', 'PT', 'Ġis', 'Ġamazing', '!'']
```



## 4. From Tokens to Embeddings

Once the tokenizer generates **numeric token IDs**, the model converts them into **embeddings**, which are **high-dimensional vectors**:

* Embeddings capture **semantic meaning**: “cat” and “dog” vectors are closer than “cat” and “carrot.”
* The model uses these vectors for downstream computations:

  * **Text generation**
  * **Similarity search**
  * **Context understanding**

So the **tokenizer + vocabulary**, **embedding model**, and **model weights** together form the **secret sauce of LLMs**.

---

## 5. Flow in Action

When you type a question into an LLM:

1. User sends text to the LLM API or middle layer.
2. The tokenizer (BPE for GPT, SentencePiece for Claude) converts the text into **numeric token IDs**.
3. For text generation: token IDs are mapped to **embeddings**, which are fed into the transformer network for processing and generation.
4. For retrieval or similarity search: embeddings may be **computed and compared externally** to other document embeddings.

> Key clarification: tokenization → numeric IDs → embeddings → transformer for generation, while similarity search is embedding-based and often outside the transformer network.

---

## 6. Why Understanding Tokens Matters

* **Prompt Length**: LLMs measure input length in **tokens**, not words.
* **Cost & Limits**: API usage and context window depend on **token count**, not word count.
* **Precision**: Different tokenizers produce different token counts for the same text.

So **conclusion**: all large language models (LLMs) use tokens instead of words because tokens are a more **efficient, flexible, and consistent** way to process and understand human language. Tokens are the fundamental units of text that the model breaks language down into before it can process the information.

---

**References & Tools:**

* OpenAI `tiktoken` (BPE): [GitHub](https://github.com/openai/tiktoken)
* Claude / SentencePiece: [GitHub](https://github.com/google/sentencepiece)

