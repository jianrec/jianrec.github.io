---
title: "MiniOneRec Project Interview Review: A Generative Recommendation Loop from Semantic ID to GRPO"
date: 2026-06-25 21:30:00 +0800
tags: [MiniOneRec, Generative Recommendation, LLM4Rec, Semantic ID, RQ-VAE, SFT, GRPO, Constrained Decoding, HR, NDCG, Interview Review]
main_category: "Paper Reading"
discipline: "LLM4Rec"
course: "MiniOneRec"
sub_category: "LLM4Rec · Project Retrospective"
material_type: "Interview Review"
description: "How I present the MiniOneRec reproduction project in interviews: turning sequential recommendation into LLM generation of Semantic IDs, doing SFT, GRPO, and constrained beam search, and how to defend the metrics and the boundaries of my contribution."
lang: en
---

This is the interview review script I prepared for the MiniOneRec project. It is neither a paper summary nor an API guide; it is organized the way I would present it in an interview: state the problem first, then string SID construction, SFT, GRPO, constrained decoding, and metric defense into one complete chain.

When I talk about this project, I want to do more than say "I got a generative recommendation framework running" — I want to be able to explain the input, output, training objective, and risk of every step.

<!--more-->

## The 30-second version

MiniOneRec is a generative sequential recommendation project. Traditional sequential recommendation typically scores and ranks candidate items; here items are first encoded into hierarchical Semantic IDs, and then an LLM directly generates the next item's SID from the user's history of SIDs.

The main line I reproduced and worked through is:

```text
Amazon interactions and item metadata
-> Qwen encodes item title / description
-> RQ-VAE produces three-level Semantic IDs
-> Extend the Qwen tokenizer, adding SID tokens to the vocabulary
-> Multi-task SFT: history SID to next SID, plus bidirectional Title/SID alignment
-> GRPO: generate multiple candidates for the same context, optimize with hit and ranking rewards
-> Constrained Beam Search: only allow generation of valid SIDs
-> Offline evaluation with HR / NDCG / CC
```

In an interview I proactively emphasize two things.

First, this is not simply swapping item ids for strings. The value of an SID is that it compresses item text semantics into discrete tokens an LLM can generate, while preserving hierarchical structure.

Second, the biggest engineering risk in generative recommendation is validity. If the LLM generates an item that does not exist, high HR/NDCG is meaningless, so both training and evaluation must be constrained around the valid SID space.

## The overall pipeline: turning ranking into constrained generation

This project addresses sequential recommendation: given a user's chronologically ordered interaction history, predict the next item they are likely to interact with.

The traditional route is:

```text
user history + candidate item -> score(candidate)
```

MiniOneRec's route is:

```text
user history SID -> LLM generates next item SID
```

This brings three changes:

| Question | Traditional recommendation | How MiniOneRec handles it |
| --- | --- | --- |
| Item representation | item id / embedding | title + description encoded by Qwen, then quantized into a Semantic ID |
| Training objective | Scoring candidates or next-item classification | causal LM generation of the next item's SID |
| Inference validity | The candidate set is valid by construction | Prefix constraints are needed to prevent generating nonexistent SIDs |

In the local Industrial data, `Industrial_and_Scientific.index.json` has 3686 items, which after deduplication adds 560 SID sub-tokens; Office Products has 3459 items and adds 600 SID sub-tokens. That number is also a reminder: the number of new tokens is not the total number of SID combinations, but the deduplicated count of the `<a_*>/<b_*>/<c_*>` sub-tokens at each level.

## SID construction: from item text to hierarchical discrete codes

The first step of SID construction is turning item text into continuous vectors. The code reads `.item.json`, concatenates `title` and `description`, tokenizes them, feeds them into Qwen, and then applies masked mean pooling over the valid tokens.

```python
# /Users/zhoujian/Desktop/MiniOneRec/rq/text2emb/amazon_text2emb.py:94
encoded_sentences = tokenizer(
    batch_texts,
    max_length=args.max_sent_len,
    truncation=True,
    return_tensors='pt',
    padding=True
).to(accelerator.device)

outputs = model(input_ids=input_ids, attention_mask=attention_mask)
last_hidden = outputs.last_hidden_state
mask_expanded = attention_mask.unsqueeze(-1).expand(last_hidden.size()).float()
sum_embeddings = torch.sum(last_hidden * mask_expanded, dim=1)
sum_mask = torch.clamp(mask_expanded.sum(dim=1), min=1e-9)
mean_output = sum_embeddings / sum_mask
```

Here I explain why we use mean pooling rather than simply taking the last token. Qwen is a decoder-only model; it has no naturally trained CLS sentence vector the way BERT does, and the last token is easily affected by templates, truncation, or punctuation. Masked mean pooling at least guarantees padding does not participate in the average, yielding a stable item-level vector.

The second step is RQ-VAE. It has three parts: an encoder, a ResidualVectorQuantizer, and a decoder.

```python
# /Users/zhoujian/Desktop/MiniOneRec/rq/models/rqvae.py:46
self.encode_layer_dims = [self.in_dim] + self.layers + [self.e_dim]
self.encoder = MLPLayers(layers=self.encode_layer_dims,
                         dropout=self.dropout_prob,bn=self.bn)

self.rq = ResidualVectorQuantizer(num_emb_list, e_dim,
                                  beta=self.beta,
                                  kmeans_init=self.kmeans_init,
                                  kmeans_iters=self.kmeans_iters,
                                  sk_epsilons=self.sk_epsilons,
                                  sk_iters=self.sk_iters,)

self.decoder = MLPLayers(layers=self.decode_layer_dims,
                         dropout=self.dropout_prob,bn=self.bn)
```

The encoder compresses the Qwen embedding into a latent space; the ResidualVectorQuantizer quantizes the latent hierarchically; the decoder is only used during training to reconstruct the original embedding, forcing the discrete codes to retain semantic information. At deployment or evaluation time we do not use the decoder to recover items from SIDs.

The core logic of residual quantization is short:

```python
# /Users/zhoujian/Desktop/MiniOneRec/rq/models/rq.py:43
x_q = 0
residual = x
for quantizer in self.vq_layers:
    x_res, loss, indices = quantizer(residual, use_sk=use_sk)
    residual = residual - x_res
    x_q = x_q + x_res
    all_indices.append(indices)
```

The first level finds the nearest code for `z`, the second level quantizes the remaining residual, and the third level continues to fill in the residual. The result is a hierarchical SID like `<a_104><b_118><c_176>`.

The RQ-VAE loss consists of a reconstruction loss and a quantization loss:

```python
# /Users/zhoujian/Desktop/MiniOneRec/rq/models/rqvae.py:74
if self.loss_type == 'mse':
    loss_recon = F.mse_loss(out, xs, reduction='mean')
loss_total = loss_recon + self.quant_loss_weight * quant_loss
```

The quantization layer has two more points that often come up: codebook loss, commitment loss, and the STE.

```python
# /Users/zhoujian/Desktop/MiniOneRec/rq/models/vq.py:89
commitment_loss = F.mse_loss(x_q.detach(), x)
codebook_loss = F.mse_loss(x_q, x.detach())
loss = codebook_loss + self.beta * commitment_loss

# preserve gradients
x_q = x + (x_q - x).detach()
```

The interview answer can be wrapped up like this: `codebook_loss` moves the codebook toward the data distribution, `commitment_loss` constrains the encoder output to stay close to the selected code; and `x + (x_q - x).detach()` is the Straight-Through Estimator, which makes the forward pass use the hard quantized result while approximately passing gradients back to the encoder.

## SFT: learn valid expression first, then learn recommendation

RQ-VAE outputs SIDs in string form, but what the model actually processes are the new token ids assigned by the tokenizer. The project first collects the SID sub-tokens that actually appear in `index.json`, then extends the tokenizer and the model embeddings.

```python
# /Users/zhoujian/Desktop/MiniOneRec/sft.py:41
def get_new_tokens(self):
    if self.indices is None:
        self._load_data()

    self.new_tokens = set()
    for index in self.indices.values():
        for token in index:
            self.new_tokens.add(token)
    self.new_tokens = sorted(list(self.new_tokens))
    return self.new_tokens
```

```python
# /Users/zhoujian/Desktop/MiniOneRec/sft.py:149
if sid_index_path and os.path.exists(sid_index_path):
    token_extender = TokenExtender(
        data_path=os.path.dirname(sid_index_path),
        dataset=os.path.basename(sid_index_path).split('.')[0]
    )
    new_tokens = token_extender.get_new_tokens()
    if new_tokens:
        tokenizer.add_tokens(new_tokens)
        model.resize_token_embeddings(len(tokenizer))
```

The interview point here is: you cannot use RQ-VAE's integer codes directly as Qwen token ids. Token ids in Qwen's original vocabulary already have their own subword meanings, and a `42` at one level does not mean the same thing as a `42` at another. So they have to be wrapped into tokens with hierarchical prefixes such as `<a_42>` and `<b_42>`.

SFT is not a single task. `sft.py` combines three kinds of data into one `ConcatDataset`:

```python
# /Users/zhoujian/Desktop/MiniOneRec/sft.py:193
train_data1 = SidSFTDataset(...)
train_datasets.append(train_data1)
train_data2 = SidItemFeatDataset(...)
train_datasets.append(train_data2)
train_data3 = FusionSeqRecDataset(...)
train_datasets.append(train_data3)
train_data = ConcatDataset(train_datasets)
```

The three tasks have different responsibilities:

| Dataset | What it trains | Purpose |
| --- | --- | --- |
| `SidSFTDataset` | history SID -> next SID | The main recommendation task |
| `SidItemFeatDataset` | title -> SID, SID -> title | Establishes bidirectional alignment between natural language and SIDs |
| `FusionSeqRecDataset` | history SID -> next item title | Preserves item text semantics, avoiding learning only SID co-occurrence |

The supervised fine-tuning loss is computed on the response only, not the prompt. The code sets the prompt portion's labels to `-100`:

```python
# /Users/zhoujian/Desktop/MiniOneRec/data.py:453
golden_tokens = self.tokenizer.encode(target_item, bos=False, eos=True)
input_prompt_len = len(tokens)
tokens = tokens + golden_tokens
attention_mask = [1] * len(tokens)
labels = [-100] * input_prompt_len + tokens[input_prompt_len:]
```

This is an easy question to get asked. My answer: the prompt is the instruction and the user's historical context; the model should not be trained to parrot it back. What it really needs to learn is "given the prompt, generate the correct SID or title." If the prompt also contributes to the loss, the fixed template would dilute the recommendation objective.

## GRPO: putting recommendation metrics back into the training objective

After SFT the model can generate SIDs, but SFT's token-level CE is not equivalent to HR/NDCG. GRPO's role is to compare multiple candidates within a group under the same user context, turning hit and ranking signals into rewards.

`rl.py` has two basic rewards:

```python
# /Users/zhoujian/Desktop/MiniOneRec/rl.py:160
def ndcg_rule_reward(prompts, completions):
    ...
    if completion.strip("\n\"") == targets[i].strip("\n\""):
        flag = True
        lis.append(0.0)
    else:
        lis.append(ndcg_rewards[i % num_generations])
```

```python
# /Users/zhoujian/Desktop/MiniOneRec/rl.py:186
def rule_reward(prompts, completions):
    ...
    if completion.strip("\n\" ") == targets[i].strip("\n\" "):
        rewards.append(1.0)
    else:
        rewards.append(0.0)
```

`rule_reward` corresponds to a hit, approximating HR; `ndcg_rule_reward` cares about the position of the target SID among the in-group candidates, approximating ranking quality. With `reward_type="ranking"` the two are used in combination.

Why doesn't GRPO need a critic? Because it makes relative comparisons among multiple completions of the same prompt. The trainer first repeats each sample `num_generations` times:

```python
# /Users/zhoujian/Desktop/MiniOneRec/minionerec_trainer.py:109
indexes = [
    idx
    for idx in torch.randperm(self.num_samples, generator=self.generator).tolist()
    for _ in range(self.repeat_count)
]
```

Then it normalizes the rewards within the group to get the advantage:

```python
# /Users/zhoujian/Desktop/MiniOneRec/minionerec_trainer.py:963
mean_grouped_rewards = rewards.view(-1, self.num_generations).mean(dim=1)
std_grouped_rewards = rewards.view(-1, self.num_generations).std(dim=1)
mean_grouped_rewards = mean_grouped_rewards.repeat_interleave(self.num_generations, dim=0)
std_grouped_rewards = std_grouped_rewards.repeat_interleave(self.num_generations, dim=0)
advantages = (rewards - mean_grouped_rewards) / (std_grouped_rewards + 1e-4)
```

Finally the update uses the current policy's logprob, the advantage, and a KL penalty against the reference model:

```python
# /Users/zhoujian/Desktop/MiniOneRec/minionerec_trainer.py:1044
per_token_logps = self._get_per_token_logps(model, input_ids, attention_mask, logits_to_keep)
ref_per_token_logps = inputs["ref_per_token_logps"]
per_token_kl = torch.exp(ref_per_token_logps - per_token_logps) - (ref_per_token_logps - per_token_logps) - 1

per_token_loss = torch.exp(per_token_logps - per_token_logps.detach()) * advantages.unsqueeze(1)
per_token_loss = -(per_token_loss - self.beta * per_token_kl)
```

The way to explain the KL penalty should be simple: RL may distort the output distribution too much in pursuit of reward, breaking the SID format and semantic alignment learned during SFT. KL is the constraint that pulls the current actor back toward the reference/SFT model. If `beta` is too large the model barely learns; if it is too small you get format drift or reward over-optimization.

## Constrained Beam Search: not post-hoc filtering

The biggest fear in generative recommendation is the model emitting invalid items. MiniOneRec's approach is not to filter after generation, but to restrict the next token during generation to come only from valid prefixes.

`evaluate.py` first reads the valid SIDs from `info_file` and builds a mapping from prefix to allowed tokens:

```python
# /Users/zhoujian/Desktop/MiniOneRec/evaluate.py:87
hash_dict = dict()
for index, ID in enumerate(prefixID):
    ID.append(tokenizer.eos_token_id)
    for i in range(prefix_index, len(ID)):
        if i == prefix_index:
            hash_number = get_hash(ID[:i])
        else:
            hash_number = get_hash(ID[prefix_index:i])
        if hash_number not in hash_dict:
            hash_dict[hash_number] = set()
        hash_dict[hash_number].add(ID[i])
```

At inference time, `ConstrainedLogitsProcessor` sets the logits of invalid tokens to `-inf`:

```python
# /Users/zhoujian/Desktop/MiniOneRec/LogitProcessor.py:45
scores = torch.nn.functional.log_softmax(scores, dim=-1)
mask = torch.full_like(scores, float('-inf'))
...
prefix_allowed_tokens = self._prefix_allowed_tokens_fn(batch_id, hash_key)
...
mask[batch_id * self._num_beams + beam_id, prefix_allowed_tokens] = 0
scores = scores + mask
```

This is also an engineering detail I emphasize in interviews. Post-hoc filtering causes two problems: first, there may not be enough candidates to fill Top-K; second, the ranking has already been contaminated by invalid candidates. Constraining the process restricts the search space directly to the valid SID prefix tree, which suits both offline evaluation and online candidate generation better.

## Metrics: HR, NDCG, and CC must be read together

MiniOneRec's evaluation cannot look only at HR/NDCG; it must also look at CC.

| Metric | Meaning | Interview explanation |
| --- | --- | --- |
| HR@K | Whether the Top-K contains the true item | Measures coverage and recall |
| NDCG@K | Whether a hit is ranked highly | Measures ranking quality |
| CC | invalid candidate count | Measures whether the generated outputs are valid |

In `calc.py`, if a predicted item is not in the valid set, CC is incremented:

```python
# /Users/zhoujian/Desktop/MiniOneRec/calc.py:60
for i in range(len(sample)):
    if sample[i] not in item_dict:
        CC += 1
        print(sample[i])
        print(target_item)
    if sample[i] == target_item:
        minID = i
        break
```

Here is how I explain the experimental results: in my local retrospective materials, GRPO raised `NDCG@20` from `0.0937` to `0.1012` and `HR@1` from `0.0514` to `0.0666` relative to SFT, but `HR@20` dropped slightly from `0.1648` to `0.1604`. This shows that RL tends to push the correct answer toward the front, at a small cost in Top-20 coverage.

This is not a story of "RL always improves everything," but a trade-off between the reward and the evaluation metrics. The current ranking reward focuses more on in-group hits and positions, and when `num_generations` is small the reward is also sparse: if the target SID never appears in the candidate group, that group has no effective positive signal.

If an interviewer asks how to fix it, I answer in priority order:

1. First make sure SFT and RL use the same tokenizer, SID mapping, `info.txt`, and evaluation script.
2. Log HR/NDCG/CC for every checkpoint and early-stop on the validation objective.
3. Increase `num_generations` to reduce the proportion of all-zero reward groups.
4. If the business cares more about HR@20, add coverage/recall/listwise rewards instead of only optimizing head ranking.
5. Grid-search the `learning_rate` and the KL `beta` to avoid overly aggressive RL updates.

## Be clear about the boundaries of your contribution

The generative recommendation framework and core ideas of this project come from the open-source MiniOneRec work. In an interview you cannot present the paper's or the open-source framework's contributions as your own.

I frame my own work as "reproduction, adaptation, modification, and a closed experimental loop":

- Getting the end-to-end flow working: Amazon data processing, text vectorization, SID construction, SFT, GRPO/RL, and constrained evaluation.
- Adapting local single-GPU scripts and a Qwen2.5-0.5B experimental path to reduce debugging cost.
- Working out the inputs and outputs of `data/`, `rq/`, `sft.py`, `rl.py`, `minionerec_trainer.py`, `evaluate.py`, and `calc.py`.
- Understanding and debugging issues such as SID/tokenizer inconsistency, invalid generation, sparse rewards, and the HR@20 drop after RL.

This framing holds up better. It shows I actually read the code, ran the pipeline, and understand the training and evaluation risks, rather than just running the open-source README once.

## Quick answers to frequent follow-ups

**Why can't SIDs just be item ids?**

An item id has no semantic structure, so it is hard for an LLM to generalize from the token itself. Through item text embeddings and residual quantization, SIDs compress similar items into nearby hierarchical discrete spaces, so they work as generation tokens while retaining some semantic relationships.

**Is the RQ-VAE decoder used in production?**

No. The decoder is a reconstruction constraint during training, meant to make SIDs retain the information in the original embedding. The recommendation stage uses the item-to-SID mapping and the valid SID prefix tree.

**Are collision and CC the same thing?**

No. A collision means multiple items share the same complete SID, indicating the encoding is not unique; CC means the model generated a candidate that is not in the valid item/SID set, indicating invalid generation. The former is a SID construction quality problem, the latter a decoding validity problem.

**Why are prompt labels set to `-100`?**

Because the causal LM loss ignores `-100`. That way the model only learns the response portion, i.e. the correct SID/title, and does not treat large amounts of fixed instruction and user input as targets to parrot back.

**Why does GRPO cause HR@20 to drop?**

RL optimizes the reward you defined; it does not guarantee that all offline metrics improve together. The current reward leans toward exact match and in-group ranking, so it may push correct answers to earlier positions while losing some Top-20 coverage.

**How would you do this online?**

I would not have a large model autoregressively generate over the full item space. A more realistic setup is to have traditional candidate generation supply the candidate set first, then have a small model or a quantized LLM do constrained generation/re-ranking within that candidate space, combined with KV Cache, batched trie lookups, and continuous batching to control P95 latency.

## How to remember it in the end

MiniOneRec can be remembered in one sentence:

```text
First compress item text into Semantic IDs that an LLM can generate,
then use SFT to learn the format and semantic alignment,
then use GRPO to pull recommendation metrics into training as rewards,
and finally use prefix constraints to guarantee the generated output always lands in the valid item set.
```

In interviews, what really sets you apart is not reciting names like RQ-VAE, SFT, and GRPO, but being able to explain why each module exists: SID solves item representation, SFT solves generation format and semantic alignment, GRPO solves the mismatch between token loss and recommendation metrics, constrained decoding solves generation validity, and HR/NDCG/CC together determine whether the evaluation is trustworthy.