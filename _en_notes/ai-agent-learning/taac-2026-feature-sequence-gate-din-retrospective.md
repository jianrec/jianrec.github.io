---
title: "TAAC-2026 Retrospective: From Sparse Features and Time Signals to Target-Aware Sequence Modeling"
date: 2026-06-24 17:00:00 +0800
tags: [TAAC, Recommender Systems, PCVR, HyFormer, RankMixer, DIN, Feature Engineering, Model Retrospective]
main_category: "Paper Reading"
sub_category: "LLM4Rec"
discipline: "LLM4Rec"
course: "TAAC-2026"
material_type: "Competition Retrospective"
description: "A design retrospective on the TAAC-2026 project covering sparse-dense pairs, dense splitting, time features, the sequence gate, the DIN target-aware branch, and training stability."
lang: en
---

The previous post recorded which modules I changed on top of the TAAC-2026 baseline. This one leans more toward "why I changed them that way": where exactly the sparsity in the PCVR task shows up, why I did not directly replicate a heavier unified Transformer route, and how I added pair features, time signals, and candidate-relevant interests inside the HyFormer framework.

<!--more-->

The related first-stage overview is here: [TAAC-2026: A Retrospective on Model Improvements over the Baseline]({{ '/notes/ai-agent-learning/taac-2026-baseline-improvements/' | relative_url }}).

## The problem is not simply too few positives

PCVR sparsity is not just "too few positive samples." What really makes it hard is three things stacking up:

- On the label side, there are many negatives, plus conversion delay and negative-sample uncertainty.
- On the feature side, high-cardinality IDs are low-frequency; skipping many fields outright throws away part of the discriminative power.
- On the sequence side, the four historical domains differ greatly in coverage, temporal freshness, and relevance, so a fixed average or a naive concat drags noise along with it.

So my approach was not to swap in a single different model, but to handle it at three levels:

| Level | Main problem | How it is handled |
| --- | --- | --- |
| Label side | Easy negatives dominate training, while AUC cares about ranking | `BCE + Pairwise`, with a small-weight ranking auxiliary term |
| Feature side | High cardinality, low frequency, and easily lost sparse-dense alignment information | hash embedding, missingness indicators, pair-aware modeling |
| Sequence side | Uneven quality across multi-domain histories, large differences in temporal state | event-level time features, domain-level statistics, sequence gate, DIN |

This is also why I did not blindly stack a big model. The competition data scale is not on the same order as production industrial data; the unified tokenization, KV cache, and complex mask designs in many industry papers are very inspiring, but replicating them directly is not necessarily worth it.

## Why I didn't directly replicate OneTrans / MTGR

My understanding of OneTrans's core idea is that it unifies sequential and non-sequential features through a tokenizer into a token sequence, and then uses a single Transformer backbone to do both sequence modeling and feature interaction. Its advantages are that it is unified, scalable, and end-to-end, and it can be combined with causal attention and cross-request KV cache for inference optimization.

But the engineering bar is also high:

- Token design is complex;
- Mask design is complex;
- The KV cache is tightly coupled to the online serving form;
- Inference latency and memory pressure are higher.

Generative recommendation routes like MTGR also gave me an important reminder: generative recommendation and scaling are valuable, but you should not throw away all the cross features that work in traditional recommendation just for the sake of unified modeling.

So this time I preferred to keep HyFormer's lightweight backbone: use queries to read the multi-domain history sequences, then use RankMixer / DIN to fill in non-sequential token interaction and candidate-history matching. The goal is not the most "unified" architecture, but to put every kind of signal in the right place under anonymized features, small competition data, and memory constraints.

## Sparse-dense pairs should not be torn apart

The baseline handles user dense features rather crudely, easily concatenating dense fields of different semantics and different scales and then projecting them uniformly. I focused on two structures:

- Large dense blocks like `fid61` and `fid87`;
- Sparse-dense pairs like `62-66` and `89-91` that appear in both `user_int` and `user_dense`.

For the latter, the key is not the dense value itself but the `(id, value)` relationship at the same position. The value is not an isolated continuous number; it is the intensity or statistical signal corresponding to some anonymized ID. If int and dense are handled separately, the model has no way of knowing that the `k`-th value belongs to the `k`-th ID.

My approach is:

```text
raw pair fields
-> split pair_int_feats and pair_dense_feats out of user_int / user_dense
-> maintain element-wise alignment by fid / offset / length
-> sparse ids go through embeddings, dense scalars go through a Linear projection
-> pool with the same valid mask
-> concatenate statistics such as coverage, mean, max, std
-> obtain pair_emb and inject it into the user NS tokenizer
```

Two details matter here.

First, the dense values for `62-66` look more like long-tail intensity or count signals, so I first set NaNs to 0, clip to non-negative, and apply `log1p` compression, to reduce how much extreme values dominate the Linear projection and the gradients. The numerical scale of `89-91` is relatively less long-tailed, so keeping the original values is more stable.

Second, pair features are not written back into `user_int`, nor do they directly become a separate standalone NS token. They are essentially supplementary information for the user-side sparse features, so I feed `pair_emb` as `extra_emb` into the input of the user RankMixer tokenizer, where it goes through an LHUC gate for sample-level rescaling before being chunked into user NS tokens. This way the pair information can spread across multiple user tokens without changing HyFormer's token structure on its own.

## Split large dense blocks by semantics, don't stuff them all into one Linear

`fid61` is 256-dimensional and `fid87` is 320-dimensional. They are clearly not ordinary low-dimensional continuous features. Mixing them with pair dense features like 62-66 and 89-91 and projecting everything together causes two problems:

- The large-dimensional fields dominate the projection and easily overwhelm the small-dimensional fields early in training.
- Dense semantics from different sources and scales get mixed together, so the model has to disentangle them itself before it can learn interactions.

So I split user dense into three paths:

```text
fid61              -> user profile dense token
fid87              -> historical interest dense token
fid62-66 / 89-91   -> sparse-dense pair embedding
```

The 320 dimensions of `fid87` naturally factor as `10 x 32`. I cannot prove with 100% certainty that its business semantics really are summaries of 10 historical items, but from the dimensional structure it looks more like a concatenation of multiple homogeneous blocks. So I gave it a lightweight attention pooling:

```text
fid87 [B, 320]
-> reshape [B, 10, 32]
-> block valid mask
-> Linear(32, 1) to get a block score
-> softmax to get block weights
-> weighted sum into [B, 32]
-> project into a history dense token
```

The attention pooling here is not multi-head self-attention; there is no Q/K/V and no pairwise interaction between blocks. It is just a learnable weighted pooling that lets the model judge which of the 10 dense blocks matter more.

## RankMixer has two contexts

RankMixer is easy to confuse in this project, because it has at least two contexts:

- The RankMixer / Query Boosting inside a HyFormer block, used for mixing among a short token sequence.
- The input-side `ns_tokenizer_type=rankmixer`, used to construct NS tokens from raw sparse embeddings.

What I discuss here is the second one. The RankMixer tokenizer first concatenates all the fid embeddings, then rescales them through LHUC/gate, and finally splits the result into a fixed number of chunks, projecting them into user/item NS tokens.

The main directory raises `user_ns_tokens` from the baseline's 5 to 12, because the dimensionality and semantic complexity of the user-side features grew — especially after the pair-aware embedding was injected into the user tokenizer. Compressing all of that into 5 user tokens would easily create an information bottleneck. The 12 tokens are not hand-mapped to 12 semantic categories; they simply give the RankMixer tokenizer more subspaces so it can learn field combinations itself.

The LHUC gate here performs sample-level feature-wise rescaling:

```text
cat_emb' = cat_emb * 2 * sigmoid(MLP(cat_emb))
```

It does not add a complex crossing structure; it just calibrates the long vector of user sparse + pair embeddings once before chunking. Pair features are more expressive but also more likely to bring noise; LHUC can learn when to amplify pair signals and when to suppress unreliable dimensions.

Note that the two dense tokens split out of `fid61/fid87` are not among these 12 user NS tokens. They are encoded separately into dense NS tokens and then concatenated with the user/item NS tokens.

## Time signals split into event-level and domain-level

I split time information into two levels.

Event-level means the time features at the position of each historical behavior. Besides the discrete time bucket, I build 8-dimensional fine-grained time features for each behavior, including recency, hour/weekday periodicity, inter-event gap, and so on. They are projected and added onto each seq token, so that the historical behavior tokens carry their own temporal context.

Domain-level means quality and freshness statistics for the whole sequence, for example:

- `max / min / mean recency`
- The number of behaviors within `15min / 1h / 1d`
- Valid length and coverage
- Masked mean pooling of the time bucket embedding

These statistics feed into two places:

1. Query generation, so that the query carries that domain's temporal freshness and activity level before cross-attention.
2. The sequence gate, so the model can judge which of the four historical domains is currently more trustworthy.

The sequence gate does not fix a uniform average over the four domains; instead each domain first gets an overall interest representation, and then dynamic weights are generated by combining the time statistics and the coverage:

```text
seq_repr = mean(q_tokens_domain)
stat_emb = Linear + LayerNorm + SiLU(seq_stats)
time_pool = masked_mean(time_bucket_embedding)
gate_input = concat(seq_repr, stat_emb, time_pool)
```

I did not concatenate length, coverage, and time statistics in raw form; I project them into `stat_emb` first. The reason is practical: these statistics are low-dimensional continuous values that do not match the `[B, D]` query embedding in dimensionality or scale. Passing them through `Linear + LayerNorm + SiLU` lets the model learn combined semantics such as "very long but very stale" or "very short but very fresh," and also makes gate training more stable.

The gate also needs protection against early collapse. If early in training the gate concentrates weight on one or two domains too soon, the other branches' weights approach 0, the gradients become very weak, and it is hard to recover later. So I mix a small amount of uniform mixing into the softmax gate weights:

```text
w_final = (1 - alpha) * w_gate + alpha / N
```

This is not meant to produce an averaged fusion in the end; it just keeps a little exploration room for each domain so that branches are not starved early on.

## DIN is an output-side target-aware supplementary branch

The HyFormer backbone is more about learning general multi-domain historical interests, but what PCVR ultimately has to answer is: is this user likely to convert on the current candidate ad. So I added a DIN-style target-aware branch.

First construct the candidate ad's `candidate_anchor`:

```text
item sparse fields -> embedding / pooling -> concat -> target_item_proj -> candidate_anchor
```

Then, within each domain, make the candidate item interact with the historical behavior tokens:

```text
score_i = MLP([q, h_i, q - h_i, q * h_i])
alpha_i = softmax(score_i)
context_domain = sum(alpha_i * h_i)
```

Here `alpha_i` is the DIN attention weight, representing the relevance between the `i`-th historical behavior and the current candidate ad. The actual interest representation is `context_domain`, i.e. the historical interest relevant to the current candidate ad, rather than a globally averaged interest.

Finally the DIN contexts of the four domains are fused with weights from the sequence gate, and enter the output-side fusion together with the HyFormer output and the candidate anchor. This branch does not replace the backbone; it fills in the candidate-relevance matching capability.

I thought about both residual and concat fusion:

- `output + gate * residual` is more like a correction to the backbone representation — simple and controllable, but when the residual is noisy it perturbs the backbone.
- `concat(output, din_context, item_target)` is more like feature-level fusion; it does not force the assumption that DIN is a correction direction for the backbone output, so it is more flexible, but has more parameters.

In the competition setting, I lean toward treating DIN as an output-side supplementary branch rather than fully overturning the HyFormer backbone.

## The loss function and EMA lean toward stability

On the training side I ended up preferring `BCE + Pairwise` over making Focal or Asymmetric Loss the main route.

Focal Loss has the advantage of simplicity: on top of BCE it downweights easy samples so the model focuses on hard ones. But the PCVR task cares about probability quality as well as ranking; stacking label smoothing and Focal together may weaken the hard-sample focus or hurt calibration.

The official configuration in the main directory looks more like this:

```text
L = BCEWithLogits(label_smoothing=0.01) + lambda * softplus(s_neg - s_pos)
```

BCE preserves probability learning and calibration, while the pairwise term aligns with the AUC ranking objective. The pairwise term gets a small weight so the ranking auxiliary term does not overwhelm the main task.

EMA's benefit also comes more from stability than from added expressiveness. Ad PCVR data is noisy, high-cardinality ID embeddings are updated sparsely, and gate/attention modules are easily affected by batch distribution, so a single checkpoint has step-level fluctuation. EMA is effectively a low-cost checkpoint ensemble.

But full EMA copies a very large sparse embedding, which is expensive in both memory and storage. So a more reasonable direction is dense-only EMA: only smooth dense parameters such as HyFormer, projections, fusion, and the prediction head, while keeping the current learned state of the sparse embeddings.

## The validation set should be closer to "predict the future from the past"

Row-group split is fast, because Parquet organizes data by RowGroup in the first place. If the original data was written roughly in time order, a RowGroup tail split approximates a temporal split; but if the times are mixed within a RowGroup, it is not a true future validation set.

A timestamp-based validation split is more rigorous: split explicitly by sample time, with earlier times going into train and later times into valid. It is closer to "predict the future from the past" in online ad prediction, and better exposes temporal extrapolation and distribution drift problems.

The cost is that it is more troublesome to implement and read, and when the validation window is too narrow the metric variance can be larger. Both splits are worth looking at in a competition, but when finally judging model generalization I trust the temporal split more.

## The biggest takeaways from this retrospective

First, PCVR sparsity has to be broken down. Too few positives is only the surface; low-frequency high-cardinality IDs, pair alignment relationships, and uneven coverage across historical domains all prevent the model from learning stable signals.

Second, unified tokenization is a trend, but "unified" does not mean crudely dumping all features into a Transformer. The real difficulties lie in token design, mask design, long-sequence efficiency, caching, and latency constraints. In a small-data competition, a lightweight HyFormer + targeted feature modeling is actually more controllable.

Third, time signals should not be treated only as recency buckets. Event-level time features tell the model the temporal state of each behavior; domain-level statistics tell the gate whether this historical domain is fresh and reliable overall.

Fourth, DIN's value is not in replacing the sequence backbone, but in explicitly answering "which historical behaviors are relevant to the current candidate ad." For PCVR that is more direct than a generic average of user interests.

If I keep working on this, I will not stack many modules at once again; I will break the ablations down more finely: validate pair-aware, dense split, event time, sequence gate, DIN, EMA, and pairwise loss separately, and then see whether they express redundant signals. Gains in a competition ultimately do not come from a longer module list, but from each added branch being able to explain the sparsity problem it solves.