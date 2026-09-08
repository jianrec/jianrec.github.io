---
title: "Tokenization in Large Language Models and Some Interesting Phenomena: Paper Notes"
date: 2026-06-26 13:00:10 +0800
tags: [Large Models, Tokenizer, Tokenization, BPE, SentencePiece, Vocabulary, Scaling Law, Recommender Systems]
main_category: "Paper Reading"
sub_category: "Large Language Models"
discipline: "LLM"
course: "Paper Reading"
material_type: "Paper Notes"
description: "Notes on why large models need a tokenizer, mainstream subword algorithms, vocabulary size trade-offs, low-frequency token generation failures, and what tokenization suggests for recommender systems."
ref: "llm-tokenization-interesting-phenomena"
lang: en
---

Source PDF: `大模型分词技术以及相关的有趣现象.pdf`  

## 0. One-Sentence Summary

A tokenizer is not a simple text preprocessing module. It is a major component of a large model's efficiency, semantic expressiveness, multilingual ability, generation stability, and even of how recommender systems are modeled. Vocabulary size, tokenization algorithm, low-frequency token coverage, and input/output vocabulary design all affect training and inference behavior.

## 0.1 Structured Reading Card

### What Problem Does It Solve?

This article is mainly about: **why large models must tokenize, and how the tokenizer affects model efficiency, training cost, multilingual ability and generation stability**.

Concretely, it covers five kinds of problems:

1. Text cannot be fed into a neural network directly; it must first become token IDs.
2. Character-level splitting produces sequences that are too long, while word-level splitting produces a vocabulary that is too large — hence subword tokenization.
3. Different tokenizers have different vocabulary sizes, which leads to differences in compression ratio, memory footprint and inference speed.
4. A large vocabulary brings the problem of under-trained low-frequency tokens, for example the "马嘉祺" case.
5. User behavior sequences in recommender systems face a similar tokenization problem: they cannot be represented by static item IDs alone.

### What Methods Are Used?

The article introduces and connects the following methods:

1. **Subword tokenization methods**: BPE, WordPiece, Unigram, SentencePiece.
2. **Vocabulary size analysis**: analyzing tokenizer efficiency and cost via compression ratio, vocabulary size, and embedding / LM Head parameter counts.
3. **Tokenizer Scaling Law**: the relationship between model size, training FLOPs, and the optimal vocabulary size.
4. **Decoupling input and output vocabularies**: using the Over-Tokenized Transformer to show that scaling the input vocabulary and the output vocabulary pay off differently.
5. **Case debugging**: using the "马嘉祺" case to explain how low-frequency tokens can cause generation failures in SFT, the LM Head, and the sampling stage.
6. **Transfer to recommender systems**: ActionPiece carries the BPE-style token merging idea over to user behavior sequence modeling.

### What Is the Key Takeaway?

The key takeaway is:

> The tokenizer is part of the model's capability, not an irrelevant preprocessing step.

More specifically:

- The larger the vocabulary, the higher the text compression ratio usually is, but the harder low-frequency tokens are to train well.
- Once a tokenizer is fixed, it is bound to the model parameters, the training data and the inference ecosystem, so large models rarely swap out the vocabulary between iterations.
- A multilingual model's quality depends not only on training data but also on how efficiently each language is segmented by the tokenizer.
- When a model "knows a word but cannot say it," the cause may not be missing knowledge but token generation probability, post-training coverage, and LM Head issues.
- Recommender systems can borrow the tokenizer idea and upgrade item IDs into context-aware behavior tokens.

### What Is the Author Trying to Say?

The article's message is:

> The tokenizer determines how a model "sees" and "generates" text. It affects sequence length, compute cost, vocabulary parameters, low-frequency word learning, multilingual fairness and downstream task modeling, so it should be treated as an important part of large model system design.

In other words, a tokenizer does not just chop up strings — it is the underlying protocol of a large model's input and output space.

### What Is the Conclusion?

It can be summed up in three sentences:

1. Large models need a tokenizer because models can only process numbers, and subword tokenization strikes a balance between the character level and the word level.
2. A tokenizer's vocabulary size affects compression ratio, training cost and low-frequency token stability, so there is a trade-off between efficiency and model capacity.
3. From LLMs to recommender systems, the essence of tokenization is turning raw objects into discrete semantic units that are easier for a model to learn.

## 1. Why Do We Need Tokenization?

Natural language is originally text, but neural networks can only process numbers. So before entering the model, text must be converted into a sequence of integer IDs:

```text
Text → Tokenizer → Token sequence → Token ID sequence → Embedding → Model
```

The two most direct approaches both have obvious problems:

| Approach | How it works | Problem |
|---|---|---|
| Character-level splitting | Every character, letter and punctuation mark is a token | Sequences get too long, compute cost is high, and a single character carries weak semantics |
| Word-level splitting | Each complete word is one token | The vocabulary is enormous, and unseen words cause OOV problems |

Modern large models mainly use **subword tokenization**:

- Frequent words or common fragments are kept as complete tokens;
- Rare and unusual words are split into smaller subwords;
- A compromise is reached among vocabulary size, sequence length, semantic expressiveness and generalization.

The core goal is:

> Represent an open vocabulary with a finite vocabulary, while keeping the number of tokens as small as possible.

## 2. Mainstream Tokenization Algorithms

Modern tokenizers usually first normalize text into a Unicode character sequence and then perform subword learning. This way Chinese, English, punctuation, emoji and so on can all be handled uniformly, without the underlying algorithm depending on language-specific rules.

### 2.1 BPE: Byte Pair Encoding

The core idea of BPE is:

> Start at the character level and repeatedly merge the most frequent adjacent token pair in the training corpus.

Basic procedure:

1. Initialization: every character is a token;
2. Count the frequency of all adjacent token pairs;
3. Merge the most frequent token pair;
4. Update the token sequences in the corpus;
5. Repeat until the vocabulary reaches the target size.

Example:

```text
lower, lowest
may learn: low, er, est
```

Characteristics:

- Simple and efficient;
- Greedy merging, so only local optimality is guaranteed;
- Commonly used in the GPT series, RoBERTa, and similar models.

### 2.2 WordPiece

WordPiece is similar to BPE in that it also grows a small vocabulary step by step, but the merge criterion is not raw frequency:

> Choose the subword that maximizes the likelihood of the training data.

A common convention is to mark non-word-initial subwords with `##`:

```text
playing → play + ##ing
```

Characteristics:

- Places more emphasis on language model probability than BPE;
- Used by BERT, DistilBERT and similar models;
- Fairly friendly to English affix and root structure.

### 2.3 Unigram Language Model

Unigram goes in the opposite direction from BPE / WordPiece:

> First construct a very large candidate vocabulary, then prune it step by step, removing the subwords that contribute least.

Basic procedure:

1. Initialize a candidate vocabulary that is as complete as possible;
2. Use a language model and the EM algorithm to estimate the importance of each subword;
3. Remove the batch of subwords with the smallest impact on overall likelihood;
4. Repeat the pruning until the vocabulary reaches the target size.

Characteristics:

- Supports probabilistic segmentation;
- The same sentence can have multiple reasonable segmentation paths;
- The algorithm picks the best segmentation based on probability;
- Models such as T5 and XLM have used this family of methods.

### 2.4 SentencePiece

Traditional BPE often applies pre-tokenization first, for example splitting on whitespace into words and then running BPE inside each word. This is natural for English but unfriendly to whitespace-free languages such as Chinese and Japanese.

The core innovation of SentencePiece is:

> Learn tokenization directly on raw text, without relying on whitespace as a word boundary.

It also encodes spaces explicitly as ordinary characters, commonly in the form `▁`. This way the tokenization result can be losslessly restored to the original text.

Characteristics:

- Does not depend on language-specific rules;
- More robust for Chinese, Japanese and mixed multilingual text;
- Mainstream open-source models such as LLaMA and Mistral commonly use SentencePiece or a similar idea.

### 2.5 Comparison of the Four Algorithms

| Method | Core idea | Typical models | Advantages | Limitations |
|---|---|---|---|---|
| BPE | Merge frequent adjacent token pairs | GPT, RoBERTa | Simple and efficient, good compression ratio | Greedy merging, possibly only locally optimal |
| WordPiece | Maximize training data likelihood | BERT | More principled probabilistic objective | Implementation and training are relatively complex |
| Unigram | Start from a large vocabulary and prune | T5, XLM | Supports multi-path probabilistic segmentation | Higher training cost |
| SentencePiece | Operate directly on raw text, no whitespace dependency | LLaMA, Mistral | Language agnostic, multilingual friendly | Actual quality still depends on corpus and vocabulary design |

## 3. How Tokenization Affects Large Models

### 3.1 Vocabulary Size and Compression Ratio

Compression ratio can be understood as:

$$
\text{压缩率} = \frac{\text{原始文本字符数}}{\text{分词后 token 数}}
$$

For the same piece of text:

```text
Fewer tokens → shorter sequence → higher compression ratio
```

A large vocabulary usually improves the compression ratio because it can include more common long fragments.

For example:

```text
人工智能
```

A small vocabulary might split it into:

```text
人 / 工 / 智 / 能
```

A large vocabulary might keep it as:

```text
人工智能
```

So the intuitive effect of a large vocabulary is:

> The dictionary is thicker, so common words or phrases can be looked up directly instead of being broken into pieces.

One note: where the original text says "finer-grained common words are kept whole," it would be more accurate to say "coarser-grained common fragments are kept whole."

### 3.2 The Cost of a Large Vocabulary

Vocabulary does not expand for free. With vocabulary size \(V\) and hidden dimension \(d\), the input embedding has roughly:

$$
V \times d
$$

parameters. The output LM Head is usually tied to vocabulary size as well. The larger the vocabulary:

- the more embedding parameters;
- the more LM Head parameters;
- the larger the softmax output dimension;
- the greater the training communication and memory pressure;
- the harder it is to train low-frequency tokens adequately.

So tokenizer design is fundamentally a trade-off:

| Vocabulary size | Advantages | Disadvantages |
|---|---|---|
| Small vocabulary | Fewer parameters, cheaper output layer, fewer low-frequency tokens | Text is chopped up finely, sequences are longer |
| Large vocabulary | High compression ratio, shorter sequences, more multilingual friendly | More parameters, low-frequency tokens are hard to learn |

### 3.3 Model Vocabulary Sizes Given in the PDF

The table below is excerpted from the PDF; in real engineering work you should rely on the official tokenizer configuration.

| Model | Vocabulary size | Notes |
|---|---:|---|
| Gemma | 256K | Large vocabulary |
| GPT-4o / o1 | 200K | Better compression ratio for non-English languages |
| Qwen 2.5 | about 152K | Optimized for multilingual use |
| Qwen 3.5 | 248K | Figure given in the PDF |
| DeepSeek-V3 | about 129K | Larger than V2 |
| Llama 3 | 128K | A big jump from Llama 2's 32K |
| GLM3 | 65,024 | The PDF says this is a reduction from an earlier 150K |
| GLM4 | 151,328 | Large vocabulary |
| GLM5.1 | 154,880 | Figure given in the PDF |

### 3.4 Why Don't Large Models Usually Iterate on the Vocabulary?

There are two main reasons.

First, **ecosystem inertia**.

A tokenizer is bound to:

- the token-id mapping;
- embedding parameters;
- LM Head parameters;
- preprocessed training data;
- LoRA / adapters;
- inference engines;
- quantization and deployment toolchains.

Changing the vocabulary amounts to changing the model's low-level interface.

Second, **diminishing returns**.

Once the vocabulary is already above 100K, further expansion yields limited compression gains, while the parameter, training, compatibility and low-frequency token problems remain.

So large models generally settle on a tokenizer before pretraining and stick with it in later versions. Adding a few special tokens is fine, but a large-scale vocabulary change usually only happens when a new generation of the model is retrained from scratch.

### 3.5 Tokenization and Multilingual Ability: Token Tax

Token Tax refers to the fact that different languages get split into different numbers of tokens under the same tokenizer, and therefore bear different compute costs.

For example:

- English and other Latin-script languages usually tokenize efficiently;
- Non-Latin-script languages and morphologically complex languages may need more tokens;
- The more tokens, the longer the sequence needed to express the same meaning, and the higher the training and inference cost.

A related concept:

> Token Fertility: how many tokens each word is split into on average.

Higher token fertility means a language is chopped up more finely, the model's processing cost is higher, and quality may degrade as well.

## 4. Is There a Scaling Law for Tokenizers?

The central question of this section is:

> As models get bigger, should vocabulary size and tokenizer design scale along with them?

The answer is: yes, this should be considered — and you cannot look at model parameter count alone; you also need to consider the input vocabulary, the output vocabulary and the training FLOPs.

### 4.1 Scaling Laws with Vocabulary

The core claim:

> Larger models generally need larger vocabularies.

Traditional scaling laws focus mainly on:

- model parameter count;
- training data volume;
- training FLOPs.

But this work brings vocabulary size into the scaling law, arguing that for a given FLOPs budget there exists an optimal vocabulary size.

If the vocabulary is too small:

- text gets chopped up finely;
- sequences get longer;
- the context window is wasted;
- compute efficiency drops.

If the vocabulary is too large:

- embedding / LM Head parameters increase;
- the output softmax becomes more expensive;
- low-frequency tokens are under-trained.

So the optimal vocabulary size depends on the compute budget and the model scale.

### 4.2 Over-Tokenized Transformer

The key insight of this work is:

> The input vocabulary and the output vocabulary play different roles and should be analyzed separately.

| Location | Role | Effect of enlarging the vocabulary |
|---|---|---|
| Input vocabulary | Encodes text into embeddings | Mainly adds embedding lookups, relatively cheap |
| Output vocabulary | Predicts the next-token probability distribution | Adds LM Head and softmax cost, much more expensive |

So a large input vocabulary is usually a good deal, while a large output vocabulary is not always beneficial and can even hurt small models.

#### Over-Encoding

Enlarge the input vocabulary, for example by adding more n-gram input tokens, so the input representation becomes more compact.

Effects:

- shorter sequences;
- higher input information density;
- relatively controllable cost.

#### Over-Decoding

Enlarge the supervision on the output side, for example by predicting n-gram tokens or introducing richer output supervision.

Effects:

- gives the model finer-grained or higher-order generation signals;
- may be effective for large models;
- may increase learning difficulty for small models.

### 4.3 Scaling Laws for Visual Tokenizers

Image generation models also have a tokenizer — it is just not a text BPE but a visual tokenizer such as a VAE, VQ-VAE or latent encoder.

Its role is:

```text
Image pixels → latent token → generative model
```

An ordinary visual tokenizer that only optimizes reconstruction tends to overfocus on low-level pixel detail, whereas generative models need high-level semantics more.

Related work from MiniMax emphasizes:

- image-text contrastive learning;
- self-supervised learning;
- reconstruction loss;
- latent representations oriented toward generation tasks.

Core conclusion:

> Tokenizer training itself is worth scaling; better latent tokens improve the downstream generative model.

## 5. In the News: Why Doesn't the Model Recognize Ma Jiaqi?

This section covers a classic tokenizer case:

> The model knows an entity, but cannot type out the corresponding characters when generating.

### 5.1 The Phenomenon

The model can describe Ma Jiaqi's career in detail, yet may be unable to reliably output the two characters "嘉祺".

This suggests the problem is not necessarily missing knowledge, but possibly:

```text
The semantic knowledge is there, but the token generation channel is broken
```

### 5.2 Key Findings from MiniMax's Debugging

"嘉祺" is tokenized as a single standalone token.

If the model's actual path when generating "马嘉祺" is:

```text
马 / 嘉祺
```

rather than:

```text
马 / 嘉 / 祺
```

then training coverage of this single "嘉祺" token becomes critical.

The PDF mentions:

- during pretraining, this token's embedding and LM Head norm look normal;
- similar tokens include "亚轩, 千禧, 祺, 耀文, 嘉" and others;
- this suggests the pretraining stage did learn some semantics;
- the problem mainly comes from the post-training SFT stage.

If "嘉祺" appears extremely rarely in the SFT data, the output weights of low-frequency Chinese tokens can get squeezed, making the generation probability very low.

### 5.3 Top-p Sampling Amplifies the Low-Frequency Token Problem

If top-p sampling is used at generation time, for example \(p=0.95\), the model only keeps the candidate tokens making up the top 95% of cumulative probability.

If "嘉祺" has too low a probability, it can be filtered out before sampling even happens, no matter how much knowledge the model has.

The symptom is:

```text
The model knows this person, but cannot type the name
```

### 5.4 Solutions

The PDF offers two directions.

#### Option 1: Recite the Entire Vocabulary

Split the whole vocabulary into chunks of, say, about 8K tokens each and construct a simple supervised task:

```text
Please repeat the following: <a batch of tokens>
```

The output is the same tokens.

Purpose:

> Ensure every token is covered at least once during SFT, preventing the generation ability for low-frequency tokens from degrading.

#### Option 2: Delete Tail-Distribution Tokens

For tail tokens that have almost no use case in post-training or in practice, consider leaving them out of the vocabulary, so under-trained whole tokens do not interfere with generation.

Fundamentally, this is a problem caused by a mismatch between the BPE merge results and the training data distribution.

### 5.5 The LM Head Gradient Bottleneck

The PDF connects this to the paper "Lost in Backpropagation: The LM Head is a Gradient Bottleneck."

A language model usually ends with an LM Head layer:

$$
\mathbb{R}^D \rightarrow \mathbb{R}^V
$$

where:

- \(D\): hidden size;
- \(V\): vocabulary size.

When \(V\) is much larger than \(D\), gradients get compressed as they backpropagate through the LM Head. Low-frequency tokens already receive few gradients, and after this bottleneck it is even harder for them to learn stable representations.

This explains why tail tokens in a large vocabulary tend to suffer from:

- slow training;
- low generation probability;
- poor output discriminability;
- generation failures for rare Chinese characters, personal names and foreign-language tokens.

## 6. Implications for Recommender Systems

Section six discusses transferring NLP tokenization ideas to recommender systems; the representative method is ActionPiece.

### 6.1 Problems with Traditional Generative Recommendation

Generative recommendation usually treats a user behavior sequence as:

```text
item_1, item_2, item_3, ...
```

and then predicts the next item the way a language model predicts the next word.

The problem is:

> The same item can carry different semantics in different user contexts.

Take the same T-shirt. In:

```text
Running shoes → gym leggings → T-shirt
```

it feels more like "athletic wear" semantics.

Whereas:

```text
Necklace → midi skirt → T-shirt
```

feels more like "fashion styling" semantics.

If you always represent that product with a fixed item ID, you ignore the contextual semantics.

### 6.2 The Core Idea of ActionPiece

ActionPiece does not treat an item as a single token; instead it represents a user action as a feature set:

```text
{Brand: Nike, Category: T-shirt, Color: White, Price: Medium, Style: Athletic}
```

and then borrows from BPE:

```text
Frequently co-occurring features or behavior fragments → merged into higher-level tokens
```

This makes it possible to learn:

- feature combinations within an item;
- transition patterns between adjacent actions;
- higher-order semantics in the context of user behavior.

### 6.3 Two Kinds of Token Pair Merging

ActionPiece counts two kinds of co-occurrence.

First, **feature co-occurrence within a single action**:

```text
Nike + running shoes
White + T-shirt
High price + luxury brand
```

Second, **feature co-occurrence between adjacent actions**:

```text
Running shoes → gym leggings
Phone → phone case
Baby formula → diapers
```

The tokens obtained this way are not just product IDs but behavior fragments with contextual meaning.

### 6.4 SPR: Set Permutation Regularization

Set Permutation Regularization can be rendered as:

> set permutation regularization / random reordering of elements within a set as regularization

The reason is that the feature set of an action is inherently unordered:

```text
{Brand: Nike, Category: T-shirt, Color: White}
```

and:

```text
{Color: White, Brand: Nike, Category: T-shirt}
```

mean the same thing.

SPR randomly shuffles the order of features inside a set to generate multiple equivalent training samples, preventing the model from wrongly relying on feature ordering.

### 6.5 Training and Inference

Training stage:

1. Convert user behavior sequences into sequences of feature sets;
2. Use ActionPiece for context-aware tokenization;
3. Construct multiple equivalent sequences via SPR;
4. Train autoregressively with a Transformer.

Inference stage:

1. Generate multiple SPR variants of the same input;
2. Run inference on each;
3. Ensemble them for a more stable recommendation result.

### 6.6 Core Implications for Recommender Systems

| NLP tokenization | ActionPiece in recommender systems |
|---|---|
| Characters, subwords | Product features |
| Words, phrases | Feature combinations |
| Sentence context | User behavior context |
| BPE merging frequent subwords | Merging frequent behavior feature combinations |
| Predicting the next token | Predicting the next item / action |

Core idea:

> Tokenization in a recommender system should not be just a static item ID mapping; it should account for context, attribute combinations and behavior transition patterns.

## 7. Key Concepts at a Glance

| Concept | Meaning |
|---|---|
| Token | The basic unit a model uses to process text; can be a character, subword, word, symbol or special marker |
| Vocabulary | The set of all tokens a tokenizer can recognize |
| Token ID | A token's integer index in the vocabulary |
| OOV | Out-of-Vocabulary, a word outside the vocabulary |
| Subword | A subword unit, between a character and a full word |
| Compression ratio | Number of characters in the raw text / number of tokens |
| Token Tax | Different languages bear different compute costs because of differing tokenization efficiency |
| Token Fertility | How many tokens each word is split into on average |
| LM Head | The output layer mapping hidden states to vocabulary logits |
| SPR | Set permutation regularization: randomly reordering an unordered feature set to improve robustness |

## 8. My Own Conclusions

1. The tokenizer determines the basic unit through which a model sees the world.  
   It is not an irrelevant preprocessing step but part of the model architecture and training system.

2. A large vocabulary improves the compression ratio but also brings tail-token learning problems.  
   Compression ratio, parameter count, softmax cost and low-frequency token coverage must all be considered together.

3. The vocabulary is usually fixed before pretraining begins.  
   Later minor versions generally do not change the tokenizer, at most appending a few special tokens.

4. A multilingual model's tokenizer directly affects language fairness.  
   The more finely a language is chopped up, the higher its training and inference cost, and the worse the model may perform on it.

5. Tokenizers have a scaling problem too.  
   Large models may need larger vocabularies, but the input vocabulary and the output vocabulary should be designed separately.

6. A generation failure on a low-frequency token is not necessarily missing knowledge.  
   It can also be the combined result of insufficient SFT coverage, the LM Head gradient bottleneck, and the sampling strategy.

7. Recommender systems can borrow ideas from tokenization.  
   User behavior does not have to be represented only as item IDs; it can be constructed as context-aware behavior tokens.

The tokenizer is the "input/output interface layer" of a large model: it determines both how text is compressed into a sequence and how the model's output space is organized. A good tokenizer must balance compression ratio, vocabulary size, multilingual fairness, low-frequency token learning, and downstream ecosystem compatibility.