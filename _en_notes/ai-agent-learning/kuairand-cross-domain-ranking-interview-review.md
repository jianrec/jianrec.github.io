---
title: "KuaiRand Cross-Domain Multi-Scenario Ranking Project — Interview Review: From DIN to PSRG/PCRG"
date: 2026-06-24 19:20:00 +0800
tags: [KuaiRand, Recommender Systems, Ranking, DIN, Cross-Domain Recommendation, Multi-Interest, PSRG, PCRG, GAUC, Interview Review]
main_category: "Paper Reading"
discipline: "LLM4Rec"
course: "KuaiRand Cross-Domain Ranking"
sub_category: "LLM4Rec · Project Retrospective"
material_type: "Project Retrospective"
description: "A consolidated write-up of the business problem, model design, ablation protocol, and the follow-up questions most likely to come up in interviews for the KuaiRand cross-domain multi-scenario ranking project."
lang: en
ref: "kuairand-cross-domain-ranking-interview-review"
---

These notes are for reviewing the KuaiRand-27K multi-scenario ranking project. When I talk about this project in an interview, I don't want to start by piling up module names — I want to state the problem clearly first: the user's history comes from multiple tabs, the current prediction happens in one specific tab, and the data distributions behind different tabs are not the same.

The model changes all revolve around this: a shared bottom embedding preserves what is common across scenarios, then the current scenario context is used to re-interpret historical behavior and to extract multiple interests relevant to the current candidate.

<!--more-->

## The project in one sentence

This is a pointwise short-video ranking task built on KuaiRand-27K exposure logs. Each exposure is one sample; the model input includes the candidate video, the user profile, the current tab/time context, and up to 50 of the user's prior behaviors, and the output is whether the user will watch the candidate video for a long time.

The main line of the code:

```text
DIN baseline
-> ADS: PSRG + PCRG
-> TransformerFusion
-> MBCNet
-> PPNet
```

In an interview I would frame it like this:

```text
First, build a candidate-relevant interest baseline with DIN.
Then handle the two problems introduced by multiple tabs:
1. The semantics of historical behavior change under different current scenarios.
2. A single query struggles to express a user's multiple interests.

PSRG re-interprets the history.
PCRG uses the candidate and the scenario to generate multiple queries and extract multiple interests.
The subsequent TransformerFusion, MBCNet, and PPNet add interest interaction, feature crossing, and personalized modulation respectively.
```

## Don't conflate scenario and domain

A scenario is a business entry point, e.g. Recommended, Following, Nearby, or Mall. A domain is not a field; it is the overall distribution of a class of samples.

Once you split by tab, the users, content, exposure mechanism, and feedback patterns in each tab may all differ:

```text
P(user features, video features, historical behavior, feedback | Following page)
!=
P(user features, video features, historical behavior, feedback | Nearby page)
```

So I make the distinction this way:

| Concept | Meaning in this project |
| --- | --- |
| Scenario | A tab or business entry point |
| Domain | The data distribution formed under that entry point |
| Cross-scenario | The user's history may come from multiple tabs |
| Cross-domain modeling | Different tabs have different distributions, so we must both share information and adapt to the scenario |

An example:

```text
Historical behavior:
Watched a beauty influencer on the Following page
Watched a foundation review on the Recommended page

Current request:
On the Mall page, predict whether the user will long-view a foundation product video
```

The same historical behavior means different things under different current tabs. Beauty behavior on the Following page may lean toward a creator relationship, while on the Mall page it looks more like product interest.

## Data and label definitions

The main task is long-view prediction, with `label_long_view` as the label. Clicks, likes, and play ratio are not turned into multi-task labels; they are used for history filtering or as candidate statistical features.

In the code, the sample label comes from `long_view`:

```python
# preprocess/step3_generate_samples_v2.py:62
sample = {
    'label_long_view': int(row['long_view']),
    'user_id_raw': int(user_id),
    'sample_time_ms': int(row['time_ms']),
    'meta_is_rand': int(row.get('is_rand', 0)),
    'meta_log_source': str(row.get('log_source', 'UNKNOWN')),
}
```

The entry condition for the history sequence is a click or a long view:

```python
# preprocess/step3_generate_samples_v2.py:142
if row['is_click'] == 1 or row['long_view'] == 1:
    curr_time = row['time_ms']
    prev_time = history[-1]['time'] if history else curr_time
    delta_t = curr_time - prev_time
    play_ratio = min(row['play_time_ms'] / max(row['duration_ms'], 1), 3.0)
```

Candidate statistical features are computed only from the pre-period logs, to avoid leaking test-period feedback:

```python
# preprocess/step2_compute_stats.py:43
stats = log_pre.groupby('video_id').agg(
    pre_show_cnt=('video_id', 'count'),
    pre_ctr=('is_click', 'mean'),
    pre_lv_rate=('long_view', 'mean'),
    pre_like_rate=('is_like', 'mean'),
    pre_play_ratio=('play_ratio', 'mean'),
).reset_index()
```

This point often draws follow-up questions in interviews. My answer stays fairly contained: the current model is a single-objective BCE long-view prediction, and the other feedback signals mainly feed the history sequence and the statistical features.

## DIN baseline

I chose DIN as the baseline because the question ranking has to answer is "is the current candidate relevant to the user's history?" DIN uses the candidate item as the query and applies target attention over the historical items.

The candidate and the history share the same video/author embedding tables:

```python
# src/models/din.py:945
def _build_cand_item_repr(self, batch: Dict[str, torch.Tensor]) -> torch.Tensor:
    cand_vid_emb = self.video_id_emb(batch[self.cand_video_col])
    cand_aid_emb = self.author_id_emb(batch[self.cand_author_col])
    cand_repr = torch.cat([cand_vid_emb, cand_aid_emb], dim=-1)
```

```python
# src/models/din.py:957
def _build_hist_item_repr(self, batch: Dict[str, torch.Tensor]) -> torch.Tensor:
    hist_vid_emb = self.video_id_emb(batch["hist_video_id"])
    hist_aid_emb = self.author_id_emb(batch["hist_author_id"])
    hist_repr = torch.cat([hist_vid_emb, hist_aid_emb], dim=-1)
```

The interaction input to DIN attention is `[q, h, q-h, q*h]`:

```python
# src/models/modules/target_attention_dnn.py:125
q = query.unsqueeze(1).expand(-1, num_tokens, -1)
att_input = torch.cat([q, keys, q - keys, q * keys], dim=-1)
att_scores = self.mlp(att_input).squeeze(-1)

att_weights, all_pad = masked_softmax(att_scores, mask, dim=-1)
user_interest = torch.bmm(att_weights.unsqueeze(1), keys).squeeze(1)
```

DIN is a good enough baseline, but in this project it has three obvious gaps:

- The historical item representation is static; it does not know which tab the current request is in.
- A single query can only aggregate into one interest vector at the end, flattening multiple interests.
- The head is concat + MLP, so explicit feature crossing is not direct enough.

## Why not learn a separate embedding per tab

This project does not train separate video/author embeddings per tab. Doing so would fragment the data, make low-frequency videos and authors even sparser, and break the shared vector space between candidates and history.

Shared embeddings solve the reuse of commonality:

```text
The same video / author
on the candidate side, on the history side, and across samples from different tabs
all land in the same base representation space
```

But shared embeddings do not solve "how to interpret the history under the current domain." The same historical behavior may carry different predictive semantics on the Following page and on the Mall page. What PSRG/PCRG do is scenario adaptation on top of the shared representation.

## d_ctx

`d_ctx` is the domain context of the current request. It is not a single tab id; it concatenates context embeddings such as tab/time and passes them through a lightweight MLP:

```python
# src/models/modules/domain_context.py:71
def forward(self, ctx_concat: torch.Tensor) -> torch.Tensor:
    d_ctx = self.encoder(ctx_concat)
    d_ctx = self.layernorm(d_ctx)
    return d_ctx
```

In the main config, `d_ctx` uses `tab`, `hour_of_day`, and `day_of_week`, with a 48-dimensional output. Both PSRG and PCRG downstream use the same `d_ctx`.

## PSRG: interpreting history inside the current domain

PSRG handles the key/value side, i.e. the history sequence representation. What it does is not "swap in a different embedding table," but add a current-scenario condition on top of the shared embedding.

The current code is `PSRG-lite`, not a full HyperNetwork. The default is a gated residual:

```python
# src/models/modules/psrg.py:95
gate = torch.sigmoid(self.gate_mlp(merged)).reshape(B, L, D)
delta = self.delta_mlp(merged).reshape(B, L, D)
hist_out = hist_repr + gate * delta

hist_out = self.layernorm(hist_out)
```

This is far more stable than generating a full private matrix, and it is expressive enough to capture "the same historical behavior means different things under different current scenarios."

When asked "why should historical embeddings differ across domains," I would not claim this is a mathematically a priori conclusion. It is a modeling assumption, and it needs evidence:

- Business-wise, the same video expresses a different preference in the Following tab versus the Mall tab.
- Data-wise, the click, long-view, and play-ratio distributions of the same item may split across tabs.
- Ablation-wise, you can keep the capacity of the remapping layer but zero out or shuffle `d_ctx` and see whether the gain shrinks.
- Representation-wise, you can check whether the PSRG outputs of the same historical item are pushed apart under different `d_ctx`.

This kind of answer holds up much better than simply asserting "they should be different."

## PCRG: extracting multiple interests from the candidate and the scenario

PCRG handles the query side. It does not pre-store several user interest vectors; it dynamically generates multiple queries from the current candidate and the current domain:

```python
# src/models/modules/pcrg.py:168
q_input = torch.cat([cand_item_repr, d_ctx], dim=-1)
q = self.query_gen(q_input).reshape(B, self.num_queries, self.query_dim)
q_item = self.query_to_item(q)
```

Each query attends over the history sequence separately, yielding multiple interest tokens:

```python
# src/models/modules/pcrg.py:173
scores = self._score_queries_with_hist(q_item, hist_repr)
mask_3d = hist_mask.unsqueeze(1).expand(-1, self.num_queries, -1)
attn, all_pad = _masked_softmax(scores, mask_3d, dim=-1)
z = torch.einsum("bgl,bld->bgd", attn, hist_repr)
```

It is important to keep these apart:

| Name | Meaning |
| --- | --- |
| query | A retrieval perspective generated from the candidate and the scenario |
| interest token | The interest vector obtained after a query attends over the history |

Compared with MIND and ComiRec, PCRG is closer to ranking. The multi-interest representations in MIND/ComiRec are user-level and static; PCRG's queries carry the candidate and the current scenario from the very start, making them candidate-aware + domain-aware.

## Relationship to STAR / PEPNet

PSRG/PCRG belong to the same family of ideas as STAR and PEPNet: a shared backbone plus conditional modulation.

The difference lies in where they act:

| Method | Primary point of action |
| --- | --- |
| STAR | Domain-specific combination of network weights or towers |
| PEPNet | Gating / scaling of embeddings, towers, or the head |
| PSRG | Representation remapping at every position of the history sequence |
| PCRG | Multi-interest query generation |

I would not describe PSRG as a replacement for STAR. A more accurate phrasing is: it pushes the principle of domain conditioning down into the sequence interest extraction layer.

## TransformerFusion

After PCRG produces multiple interest tokens, they have not yet interacted with each other. By default TransformerFusion does not operate on the 50 raw history entries; it operates on the small number of interest tokens produced by PCRG.

First self-attention:

```python
# src/models/modules/transformer_fusion.py:237
if self.n_layers > 0:
    padding_mask = token_mask <= 0
    for layer in self.encoder_layers:
        x = layer(x, padding_mask=padding_mask)
```

Then target attention with the candidate:

```python
# src/models/modules/transformer_fusion.py:242
if self.use_target_attention:
    u_target, attn_debug = self.target_attention(
        query=q,
        keys=x,
        mask=token_mask,
        return_debug=True,
    )
```

The division of labor at this layer is fairly clear:

```text
PCRG: pull multiple interests out of the history
TransformerFusion: let those interests interact with each other first
TargetAttention: then let the current candidate pick the most relevant combination of interests
```

DIN target attention cares about whether the candidate and the history are related; self-attention cares about the relationships among interest tokens — similarity, complementarity, conflict.

## MBCNet / FGC

The preceding layers mainly handle sequence interests. The final score also has to account for the candidate, the user profile, the context, and candidate statistical features. The original concat + MLP can learn implicit crossing, but has no explicit structure.

MBCNet's head has three paths:

| Branch | Role |
| --- | --- |
| FGC | Local explicit crossing within semantic groups |
| Low-rank Cross | Global explicit crossing in a low-rank form |
| Deep | Retains the high-order nonlinear expressiveness of an MLP |

The feature grouping comes from semantic slices of the head input:

```python
# src/models/modules/feature_slices.py:22
DEFAULT_MBCNET_GROUPS: List[str] = [
    "user_interest",
    "cand_repr",
    "user_profile_sparse_embs",
    "user_dense",
    "context_embs",
    "candidate_side_embs",
]
```

Within-group crossing in FGC supports `cross1`, `bilinear`, and `gated`:

```python
# src/models/modules/mbcnet.py:86
if self.mode == "cross1":
    self.param = nn.Linear(self.group_dim, 1, bias=True)
elif self.mode in {"bilinear", "gated"}:
    self.param = nn.Linear(self.group_dim, self.group_dim, bias=True)
```

If you drop FGC and keep only Low-rank Cross and Deep, you can interpret it as global low-rank explicit crossing + DNN implicit crossing.

## PPNet

The last step in the current repo's README is PPNet. It does not change the upstream interest extraction; it performs lightweight personalized modulation on the head side.

`p_ctx` comes from the scenario, user attributes, and an activity proxy:

```python
# src/models/modules/personal_context.py:159
if context_embs is not None and context_embs.shape[-1] > 0:
    parts.append(context_embs)
```

The default is group-wise FiLM:

```python
# src/models/modules/ppnet.py:179
params = self.out_proj(self.backbone(p_ctx))
gamma, beta = torch.chunk(params, 2, dim=-1)

x_norm_before = x.norm(dim=-1).mean()
x_mod = self._apply_group_film(x=x, gamma=gamma, beta=beta)
```

Intuitively, the Following page may weight creator relationships more heavily while the Mall page weights product attributes more heavily; highly active and low-activity users also differ in how much they rely on historical interests. PPNet brings this kind of conditioning into the final head.

## Metrics

For this project I can reliably discuss AUC and GAUC.

Global AUC is computed over all exposure samples pooled together. GAUC groups by user, computes AUC within each user, and then weights by sample count. Users with only one-sided labels are skipped:

```python
# src/metrics/metrics.py:230
if n_pos == 0 or n_neg == 0:
    continue
```

Strict request-level NDCG cannot be computed, because the data does not group the candidate list under a single request. What we have is exposure-sample-level candidates, prediction scores, labels, and `user_id`.

The full-track results in the README:

| Stage | test_standard GAUC | test_random GAUC | random delta |
| --- | ---: | ---: | ---: |
| DIN baseline | 0.6761 | 0.5525 | - |
| +ADS | 0.6854 | 0.5698 | +0.0173 |
| +Transformer | 0.6880 | 0.5745 | +0.0047 |
| +MBCNet | 0.6905 | 0.5775 | +0.0030 |
| +PPNet | 0.6918 | 0.5815 | +0.0040 |

Be careful about the definitions here: `0.6918` is standard, not random. Random ends at `0.5815`, an absolute improvement of `0.0290` over the baseline's `0.5525`.

## Follow-up questions likely in interviews

**Does PSRG generate full private weights?**

No. The current implementation is PSRG-lite: a shared MLP generates `gate/delta` or FiLM parameters that modulate each historical position; it does not generate a full dynamic matrix.

**Does PCRG generate queries or interest tokens?**

It first generates queries, then uses those queries to attend over the history to obtain interest tokens. A query is a retrieval perspective; an interest token is the retrieval result.

**Why not just apply self-attention directly over the history?**

You could, but the default path first uses PCRG to compress the 50 history entries into a small number of candidate-relevant interests, and then applies self-attention over those interest tokens. It is cheaper computationally and easier to interpret.

**Does MBCNet only handle non-sequential features?**

No. MBCNet handles the final head input, which contains the upstream `user_interest` as well as the candidate, user profile, context, and candidate-side attributes.

**How do you prove the gain isn't just from more parameters?**

Through ablations. For example, keep the capacity of the PSRG remapping layer but zero out or shuffle `d_ctx` and see whether the gain disappears; for PCRG, compare `G=1` against `G>1`.

## Tell it along this line when reviewing

```text
Business observation:
The user's history spans tabs, and the current request happens in one specific tab.

Modeling problem:
Different tabs are different data domains; we must both share cross-scenario commonality and adapt to the current target domain.

DIN:
Apply target attention from the candidate over the history to establish a ranking baseline.

Shared embeddings:
Reuse the cross-scenario base representations of videos and authors.

d_ctx:
Encode the current tab/time target domain.

PSRG:
Map historical behaviors into the current target domain and re-interpret them.

PCRG:
Generate multiple queries from the candidate and the current domain to extract multiple interests.

TransformerFusion:
Let the multiple interest tokens interact, then let the candidate apply target attention.

MBCNet / PPNet:
Strengthen head-side feature crossing and personalized modulation.

Evaluation:
Train with BCE, evaluate with AUC and user-level GAUC, focusing on test_random GAUC.
```

The most important thing about this project is not "how many modules were used," but that every step maps to a specific gap DIN has in multi-scenario ranking: history that does not change with the scenario, a single query flattening interests, a lack of interaction among interests, insufficiently explicit crossing in the head, and feature weights that differ across users and scenarios.