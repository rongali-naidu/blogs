# How LLMs Picks the Next Word?

This is covered in many blogs but i wanted to calrfiy a few questions so gathered these points as my notes.To make this concrete, we'll follow one real request from start to finish:

> **The user types:** *"What is the color of the sky?"*

The model has already read that question and is now generating its reply. It has produced the words *"The sky is"* and is about to choose the **next** word. Let's trace exactly how it picks it.

> **One idea to hold onto:** The model's only real output is a *score for every possible next word*. Everything after that — temperature, top-p — just reshapes and trims those scores before one is randomly picked. The settings never add knowledge; they only change how the existing scores are used.

---

## Step 1: Tokenization — Text Becomes Numbers

Computers do math on numbers, not letters. So the user's question *"What is the color of the sky?"* — plus the reply-so-far *"The sky is"* — is first broken into **tokens**.

- **What it does:** A tokenizer splits text into chunks — whole words, sub-words, or punctuation — using an algorithm like Byte-Pair Encoding (BPE). Common words become a single token; rarer words get split.
  - `"The"` → `464`
  - `" sky"` → `5824`
  - `" is"` → `318`
  - `" refrigerator"` would split into `" refriger"` + `"ator"` → two tokens
- **Embedding lookup:** Each token ID is mapped to a **vector** — a long list of numbers that places the token's meaning in a high-dimensional space, where related concepts sit near each other ("sky" near "cloud," far from "invoice").

*This is why token count, not word count, drives API cost and context limits.*

---

## Step 2: The Model — Vectors Become Scores (Logits)

The vectors for the whole conversation so far flow through the neural network's **weights**: billions of numbers fixed during training.

1. **Where weights come from:** During training, the model read enormous amounts of text and repeatedly predicted the next token. Each mistake nudged the weights via **backpropagation**. After training, the weights are frozen.
2. **What weights actually are:** Not a database of facts — a compressed set of *statistical patterns* about grammar, style, and how concepts relate. (This is also why models can confidently state things that are wrong: there's no fact table to look up, only learned correlations.)
3. **The engine — attention:** As the vectors pass through many **transformer layers**, a mechanism called **self-attention** lets each token "look at" the others. This is where the model connects the user's word *"color"* with *"sky"* and figures out that a **color word** should come next.
4. **The output — logits:** At the final layer, the model produces one raw score (a **logit**) for *every single token in its vocabulary* — typically 50,000–200,000 of them — **not just the relevant ones.**

This is the part most explanations skip, and it answers the natural question *"how does the model know which words to consider?"* — **It doesn't choose. It scores them all, every time.** It assigns a number to "blue," "gray," "the," "banana," and "refrigerator" simultaneously. "Relevance" isn't about *which* words get scored (they all do); it's reflected in *how high* each score is — and the attention step above is what pushed the sensible ones up and the nonsense down.

> **Scores after "The sky is…" (showing 4 of ~100,000 tokens):**
>
> | Token | Logit | Why |
> |---|---|---|
> | **blue** | `10` | Attention linked "color" + "sky" → a color is expected |
> | **gray** | `8` | Also a plausible sky color |
> | **purple** | `5` | Possible but less common |
> | **refrigerator** | `-42` | Grammatically possible, semantically absurd → scored near-zero |
> | *(…99,996 more tokens, almost all with very low scores…)* | | |
>
> The model didn't *pick* these four — it scored the entire vocabulary and these simply rose to the top. This is exactly why the filtering steps below exist: to discard the massive tail of near-zero tokens so the model only samples from the sensible few.

At this point the model's job is **done**. Everything that follows is *sampling* — and it happens in the inference software, not the model weights.

---

## Step 2.5: Why the Model Re-Reads the Whole Question *and* Its Own Partial Answer

Here's something that surprises people: to generate **each** word, the model re-reads the *entire* text so far — your question *and* every word it has already written. Why repeat all that work?

**Because the model is stateless — it has no memory between tokens.** It keeps no internal notepad that carries over. Each time it's asked for one more token, it's handed the whole text and asked, fresh:

> "Given ALL of this text — the question and everything generated so far — what's the single most likely next token?"

It answers with one token, that token is appended, and the model is called **again** with the now-longer text. It doesn't know it just answered; it only ever sees a passage of text and predicts what comes next. Re-reading everything isn't a choice — **it's the only input the model ever gets.**

**Why it needs the question:** without it, the model has no topic. *"The sky is ___"* only leads to "blue" because the question established what's being asked.

**Why it needs its own partial answer (the subtle part):** each new word must stay grammatically and logically consistent with the words already written — and the only way to ensure that is to see them. Watch the per-token calls, each one *independent*:

```
Call 1:  "What is the color of the sky? "            → predicts "The"
Call 2:  "What is the color of the sky? The"         → predicts " sky"
Call 3:  "What is the color of the sky? The sky"     → predicts " is"
Call 4:  "What is the color of the sky? The sky is"  → predicts " blue"
Call 5:  "...The sky is blue"                        → predicts "."
```

At Call 4, the model must see *"The sky is"* to know a **color** should come next, that it fits after "is," and that it hasn't already said it. Strip the partial answer away and Call 4 wouldn't know if it's at the start, middle, or end of a sentence — producing incoherent fragments.

**"Isn't re-reading everything each time wasteful?"** This is addressed with the **KV cache**: the model saves the intermediate attention math for tokens it already processed, so each new call only does the heavy computation for the *one new* token and reuses the cached work for the rest. Conceptually it still considers the whole text every step; mechanically it doesn't recompute it from scratch. This is a big reason responses stream quickly.

**And this is exactly what the "context window" is:** all the text the model re-reads on every token — system prompt + your question + the answer-so-far + earlier history. A "200K-token context window" means that's the most text it can re-read per step. Run past it and the oldest text falls out of view — which is why a model can "forget" the start of a very long conversation.

---

## Step 3: Temperature — Reshaping the Gaps

Raw logits aren't probabilities yet. Two things turn them into percentages:

- **Temperature** divides each logit by a number `T`. This *stretches or compresses the gaps* between scores.
- **Softmax** then converts the adjusted scores into probabilities that add up to 100% — across *all* ~100,000 tokens (most end up at essentially 0%).

(A subtle but important point: *temperature scales, softmax converts.* Temperature alone doesn't make percentages.)

For our sky question, looking at the top candidates:

- **Low temperature (`0.5`):** Gaps widen → the top word dominates → focused, predictable answer.
  - **blue:** 85% | **gray:** 14% | **purple:** 1%
- **High temperature (`1.5`):** Gaps flatten → probability spreads out → more variety.
  - **blue:** 45% | **gray:** 35% | **purple:** 20%
- **Temperature = 0:** A special case — the model always takes the single highest-scoring token (greedy). Fully deterministic; the same question gives the same answer every time. This is what you want for strict, repeatable, factual output.

---

## Step 4: Top-P (and Top-K) — Trimming the Candidates

Softmax just produced a probability for **all ~100,000 tokens**, most of them effectively 0%. Top-P and Top-K's whole job is to cut that giant tail down to the realistic handful before sampling — so the model never accidentally blurts out "refrigerator."

- **Top-P (nucleus sampling):** Sort candidates high to low, add them up until the cumulative probability reaches a threshold (say 0.50), then discard the rest.
- **Top-K (its sibling):** Simply keep the *K* highest candidates, ignoring cumulative probability. Often used alongside Top-P.

Top-P with a **0.50** threshold on our sky answer:

| Temperature | Probability pool | Top-P (0.50) action | Survivors |
|---|---|---|---|
| **Low** | `blue: 85%` | 85% already exceeds 50% → stop immediately | **blue** only |
| **High** | `blue: 45%, gray: 35%` | 45% < 50%, add gray → 80% → stop | **blue** + **gray** |

---

## Step 5: The Final Selection

From the survivors, the model does a **weighted random draw**:

- Only **blue** survived → the answer continues *"The sky is blue."*
- **blue (45%)** and **gray (35%)** survived → the model picks between them in proportion to their probabilities, so it might say *"blue"* this time and *"gray"* next time.

The chosen token is appended to the reply, and **the entire loop runs again** to pick the word after that — one token at a time. This is literally why you see the answer stream in word by word.

## Why there is limit on context-window when the tokens are processed in sequence?

"processed in sequence" does not mean "one at a time, independently."** If it did, there'd be no limit. The limit exists precisely because every token must interact with every other token. Three reasons:

### 1. Attention is all-to-all — cost grows with the *square* of length

This ties straight back to the attention section. Self-attention means **every token looks at every other token.** So for `n` tokens, the model computes roughly `n × n` interactions.

```
100 tokens   →   100 × 100   =   10,000 interactions
1,000 tokens →  1,000 × 1,000 = 1,000,000 interactions
100,000 tokens → 100,000²    = 10,000,000,000 interactions
```

Double the context, **quadruple** the attention compute. This `O(n²)` scaling is the core reason there's a ceiling — at some point it's too slow and too expensive to be practical. "In sequence" doesn't save you, because each token still has to attend to all the others.

### 2. Memory (the KV cache) grows linearly and must all stay resident

the saved key/value vectors for every prior token? That cache lives in GPU memory, and it grows with **every token in the window**. A longer window means a bigger cache that must fit in finite GPU RAM, *simultaneously*, for the whole session. Run out of GPU memory → hard wall. So even though tokens arrive in sequence, the model must hold the representation of *all* of them at once to do attention.

### 3. The model was trained with a fixed maximum length

This is the subtle one. The limit isn't only about compute — it's baked into the model. During training, the model only ever saw sequences up to some maximum length (say 200K tokens). Its **positional encodings** — the mechanism that tells it "this is token #5, this is token #50,000" — were only ever learned up to that length. Feed it position #250,000 and it's in territory it never trained on; behavior degrades badly. So the number is a **declared, trained-in constant**, not an arbitrary software cap you can just raise.

### Putting it together — why not "just accept more"?

Because accepting more means:
- **quadratically** more attention compute (slower, costlier),
- **linearly** more GPU memory for the KV cache (can exhaust hardware),
- operating **beyond the positions the model was trained for** (quality collapses).

"Processed in sequence" describes the *order* tokens are consumed — but the model still needs the *entire* sequence co-resident and mutually attending, which is exactly what makes the window finite.




