---
title: "KuaiRand 跨域多场景精排项目面试复习：从 DIN 到 PSRG/PCRG"
date: 2026-06-24 19:20:00 +0800
tags: [KuaiRand, 推荐系统, 精排, DIN, 跨域推荐, 多兴趣, PSRG, PCRG, GAUC, 面试复习]
main_category: "LLM4Rec"
discipline: "LLM4Rec"
course: "KuaiRand 跨域精排"
material_type: "项目复盘"
description: "整理 KuaiRand 跨域多场景精排项目的业务问题、模型设计、消融口径和面试中容易被追问的点。"
---

这份笔记用来复习 KuaiRand-27K 多场景精排项目。面试讲这个项目时，我不想从模块名开始堆，而是先把问题讲清楚：用户历史来自多个 tab，当前预测发生在某个 tab，不同 tab 背后的数据分布并不一样。

模型改造围绕这件事展开：共享底层 embedding 保留跨场景共性，再用当前场景上下文重新解释历史行为，并抽取当前候选相关的多兴趣。

<!--more-->

## 项目一句话

这是一个基于 KuaiRand-27K 曝光日志的 pointwise 短视频精排任务。每一次曝光是一条样本，模型输入包括候选视频、用户画像、当前 tab/time 上下文，以及用户此前最多 50 条历史行为，输出是用户是否会对候选视频产生长观看。

代码主线：

```text
DIN baseline
-> ADS: PSRG + PCRG
-> TransformerFusion
-> MBCNet
-> PPNet
```

我面试时会把它讲成：

```text
先用 DIN 建一个候选相关兴趣 baseline。
再处理多 tab 带来的两个问题：
1. 历史行为在不同当前场景下语义会变。
2. 单 query 很难表达用户多兴趣。

PSRG 负责重解释历史。
PCRG 负责用候选和场景生成多个 query 抽取多兴趣。
后面的 TransformerFusion、MBCNet、PPNet 分别补兴趣交互、特征交叉和个性化调制。
```

## 场景和域不要混

场景是业务入口，比如推荐、关注、附近、商城。域不是一个字段，而是一类样本的整体分布。

按 tab 切开后，每个 tab 的用户、内容、曝光机制和反馈规律都可能不同：

```text
P(用户特征, 视频特征, 历史行为, 反馈 | 关注页)
!=
P(用户特征, 视频特征, 历史行为, 反馈 | 附近页)
```

所以我会这样区分：

| 概念 | 项目里的含义 |
| --- | --- |
| 场景 | tab 或业务入口 |
| 域 | 这个入口下形成的数据分布 |
| 跨场景 | 用户历史可能来自多个 tab |
| 跨域建模 | 不同 tab 分布不同，既要共享信息，又要做场景适配 |

例子：

```text
历史行为：
关注页看过美妆博主
推荐页看过粉底测评

当前请求：
商城页预测用户是否长观看粉底商品视频
```

同一个历史行为在不同当前 tab 下含义不一样。关注页里的美妆行为可能偏作者关系，商城页下可能更像商品兴趣。

## 数据和标签口径

主任务是长观看预测，label 是 `label_long_view`。点击、点赞和播放比例没有被做成多任务 label，而是用于历史筛选或候选统计特征。

代码里样本 label 来自 `long_view`：

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

历史序列的入池条件是点击或长观看：

```python
# preprocess/step3_generate_samples_v2.py:142
if row['is_click'] == 1 or row['long_view'] == 1:
    curr_time = row['time_ms']
    prev_time = history[-1]['time'] if history else curr_time
    delta_t = curr_time - prev_time
    play_ratio = min(row['play_time_ms'] / max(row['duration_ms'], 1), 3.0)
```

候选统计特征只用 pre 期日志算，避免把测试期反馈泄漏进去：

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

这一点面试里容易被追问。我的说法会比较收敛：当前模型是单目标 BCE 长观看预测，其他反馈信号主要进历史序列和统计特征。

## DIN baseline

Baseline 选 DIN，是因为精排要回答的是“当前候选和用户历史是否相关”。DIN 用候选 item 作为 query，对历史 item 做 target attention。

候选和历史共用同一套 video/author embedding 表：

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

DIN attention 的交互输入是 `[q, h, q-h, q*h]`：

```python
# src/models/modules/target_attention_dnn.py:125
q = query.unsqueeze(1).expand(-1, num_tokens, -1)
att_input = torch.cat([q, keys, q - keys, q * keys], dim=-1)
att_scores = self.mlp(att_input).squeeze(-1)

att_weights, all_pad = masked_softmax(att_scores, mask, dim=-1)
user_interest = torch.bmm(att_weights.unsqueeze(1), keys).squeeze(1)
```

DIN 够适合做 baseline，但在这个项目里有三个明显缺口：

- 历史 item 表示是静态的，不知道当前请求在哪个 tab。
- 单 query 最后只能聚合出一个兴趣向量，多兴趣会被压平。
- head 是 concat + MLP，显式特征交叉不够直接。

## 为什么不为每个 tab 学一套 embedding

这个项目没有给每个 tab 单独训练 video/author embedding。这样做会让数据被切碎，低频视频和作者更稀疏，也会破坏候选和历史共用的向量空间。

共享 embedding 解决的是共性复用：

```text
同一个视频 / 作者
在候选侧、历史侧、不同 tab 样本里
都落在同一个基础表示空间
```

但共享 embedding 不解决“当前域下如何解释历史”。同一条历史行为，在关注页和商城页里的预测语义可能不同。PSRG/PCRG 做的是共享表示之上的场景适配。

## d_ctx

`d_ctx` 是当前请求的域上下文。它不是单独一个 tab id，而是把 tab/time 等上下文 embedding 拼起来，再过一个轻量 MLP：

```python
# src/models/modules/domain_context.py:71
def forward(self, ctx_concat: torch.Tensor) -> torch.Tensor:
    d_ctx = self.encoder(ctx_concat)
    d_ctx = self.layernorm(d_ctx)
    return d_ctx
```

主配置里 `d_ctx` 用了 `tab`、`hour_of_day`、`day_of_week`，输出 48 维。后面 PSRG 和 PCRG 都用同一份 `d_ctx`。

## PSRG：把历史放到当前域里解释

PSRG 处理 key/value 侧，也就是历史序列表示。它做的不是“换一套 embedding”，而是在共享 embedding 上加当前场景条件。

当前代码是 `PSRG-lite`，不是完整 HyperNetwork。默认是 gated residual：

```python
# src/models/modules/psrg.py:95
gate = torch.sigmoid(self.gate_mlp(merged)).reshape(B, L, D)
delta = self.delta_mlp(merged).reshape(B, L, D)
hist_out = hist_repr + gate * delta

hist_out = self.layernorm(hist_out)
```

这比生成完整 private matrix 稳很多，也够表达“同一历史行为在不同当前场景下语义不同”。

被问“为什么历史 embedding 在不同域下应该不一样”时，我不会说这是数学上先验成立的结论。它是一个建模假设，需要用证据支撑：

- 业务上，同一视频在关注 tab 和商城 tab 里表达的偏好不同。
- 数据上，同一 item 的点击、长观看、播放比例分布可能随 tab 分裂。
- 消融上，可以保留重映射层容量但置零或打乱 `d_ctx`，看增益是否缩水。
- 表征上，可以看同一历史 item 在不同 `d_ctx` 下的 PSRG 输出是否被推开。

这套回答比直接说“应该不一样”更稳。

## PCRG：用候选和场景抽多兴趣

PCRG 处理 query 侧。它不是提前存几组用户兴趣，而是根据当前候选和当前域动态生成多个 query：

```python
# src/models/modules/pcrg.py:168
q_input = torch.cat([cand_item_repr, d_ctx], dim=-1)
q = self.query_gen(q_input).reshape(B, self.num_queries, self.query_dim)
q_item = self.query_to_item(q)
```

每个 query 分别 attend 历史序列，得到多个 interest token：

```python
# src/models/modules/pcrg.py:173
scores = self._score_queries_with_hist(q_item, hist_repr)
mask_3d = hist_mask.unsqueeze(1).expand(-1, self.num_queries, -1)
attn, all_pad = _masked_softmax(scores, mask_3d, dim=-1)
z = torch.einsum("bgl,bld->bgd", attn, hist_repr)
```

这里要分清：

| 名称 | 含义 |
| --- | --- |
| query | 候选和场景生成的检索视角 |
| interest token | query attend 历史之后得到的兴趣向量 |

和 MIND、ComiRec 相比，PCRG 更贴近精排。MIND/ComiRec 的多兴趣偏用户级静态表达；PCRG 的 query 从一开始就带候选和当前场景，是 candidate-aware + domain-aware。

## 和 STAR / PEPNet 的关系

PSRG/PCRG 和 STAR、PEPNet 是同一类思想：共享主干，再做条件化调制。

区别在作用位置：

| 方法 | 主要作用位置 |
| --- | --- |
| STAR | 网络权重或 tower 的域特定组合 |
| PEPNet | embedding、tower 或 head 的门控 / 缩放 |
| PSRG | 历史序列每个位置的表示重映射 |
| PCRG | 多兴趣 query 生成 |

我不会把 PSRG 说成替代 STAR。更准确的说法是：把 domain-conditioning 这个原则下沉到了序列兴趣抽取层。

## TransformerFusion

PCRG 得到多个 interest token 后，它们之间还没有交互。TransformerFusion 默认不是直接处理 50 条历史，而是处理 PCRG 输出的少量兴趣 token。

先 self-attention：

```python
# src/models/modules/transformer_fusion.py:237
if self.n_layers > 0:
    padding_mask = token_mask <= 0
    for layer in self.encoder_layers:
        x = layer(x, padding_mask=padding_mask)
```

再用候选做 target attention：

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

这一层的分工比较清楚：

```text
PCRG：从历史里取出多个兴趣
TransformerFusion：让这些兴趣之间先交互
TargetAttention：再让当前候选选择最相关的兴趣组合
```

DIN target attention 关注的是候选和历史是否相关；self-attention 关注的是兴趣 token 之间的关系，比如相似、互补、冲突。

## MBCNet / FGC

前面几层主要处理序列兴趣。最后打分还要看候选、用户画像、上下文、候选统计特征。原来的 concat + MLP 能学隐式交叉，但没有明确结构。

MBCNet 的 head 有三条路：

| 分支 | 作用 |
| --- | --- |
| FGC | 按语义 group 做局部显式交叉 |
| Low-rank Cross | 用低秩方式做全局显式交叉 |
| Deep | 保留 MLP 的高阶非线性表达 |

特征分组来自 head input 的语义切片：

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

FGC 组内交叉有 `cross1`、`bilinear`、`gated`：

```python
# src/models/modules/mbcnet.py:86
if self.mode == "cross1":
    self.param = nn.Linear(self.group_dim, 1, bias=True)
elif self.mode in {"bilinear", "gated"}:
    self.param = nn.Linear(self.group_dim, self.group_dim, bias=True)
```

如果去掉 FGC，只保留 Low-rank Cross 和 Deep，可以把它解释成全局低秩显式交叉 + DNN 隐式交叉。

## PPNet

当前仓库 README 的最后一步是 PPNet。它不改变前面的兴趣抽取，而是在 head 侧做轻量个性化调制。

`p_ctx` 来自场景、用户属性和活跃度代理：

```python
# src/models/modules/personal_context.py:159
if context_embs is not None and context_embs.shape[-1] > 0:
    parts.append(context_embs)
```

默认是 group-wise FiLM：

```python
# src/models/modules/ppnet.py:179
params = self.out_proj(self.backbone(p_ctx))
gamma, beta = torch.chunk(params, 2, dim=-1)

x_norm_before = x.norm(dim=-1).mean()
x_mod = self._apply_group_film(x=x, gamma=gamma, beta=beta)
```

直觉上，关注页可能更重作者关系，商城页可能更重商品属性；高活跃用户和低活跃用户对历史兴趣的依赖也不同。PPNet 就是把这类条件带到最终 head。

## 指标

这个项目能稳定讲 AUC 和 GAUC。

全局 AUC 是所有曝光样本放在一起算。GAUC 是按用户分组，每个用户内部算 AUC，再按样本数加权。只有单边标签的用户会被跳过：

```python
# src/metrics/metrics.py:230
if n_pos == 0 or n_neg == 0:
    continue
```

严格的 request-level NDCG 不能算，因为数据里没有同一次请求下的候选列表分组。现在有的是曝光样本级候选、预测分数、label 和 `user_id`。

README 里的 full track 结果：

| 阶段 | test_standard GAUC | test_random GAUC | random 增量 |
| --- | ---: | ---: | ---: |
| DIN baseline | 0.6761 | 0.5525 | - |
| +ADS | 0.6854 | 0.5698 | +0.0173 |
| +Transformer | 0.6880 | 0.5745 | +0.0047 |
| +MBCNet | 0.6905 | 0.5775 | +0.0030 |
| +PPNet | 0.6918 | 0.5815 | +0.0040 |

这里要注意口径：`0.6918` 是 standard，不是 random。random 最终是 `0.5815`，从 baseline 的 `0.5525` 绝对提升 `0.0290`。

## 面试里容易被追问的点

**PSRG 是不是生成完整 private weight？**

不是。当前实现是 PSRG-lite，用共享 MLP 生成 `gate/delta` 或 FiLM 参数，对每个历史位置做调制，不生成完整动态矩阵。

**PCRG 生成的是 query 还是 interest token？**

先生成 query，再用 query attend 历史，得到 interest token。query 是检索视角，interest token 是检索结果。

**为什么不直接对历史做 self-attention？**

可以做，但默认路径先用 PCRG 把 50 条历史压成少量候选相关兴趣，再在这些兴趣 token 上做 self-attention。计算更省，也更容易解释。

**MBCNet 是不是只处理非序列特征？**

不是。MBCNet 处理最终 head input，里面包含上游得到的 `user_interest`，也包含候选、用户画像、上下文和候选侧属性。

**怎么证明不是参数变多带来的提升？**

要看消融。比如保留 PSRG 重映射层容量但置零或打乱 `d_ctx`，看增益是否消失；PCRG 可以看 `G=1` 和 `G>1` 的差异。

## 复习时按这条线讲

```text
业务现象：
用户历史跨 tab，当前请求在某一个 tab。

建模问题：
不同 tab 是不同数据域，既要共享跨场景共性，也要适配当前目标域。

DIN：
用候选对历史做 target attention，建立精排 baseline。

共享 embedding：
复用视频和作者的跨场景基础表示。

d_ctx：
编码当前 tab/time 目标域。

PSRG：
把历史行为映射到当前目标域下重新解释。

PCRG：
由候选和当前域生成多个 query，抽取多兴趣。

TransformerFusion：
让多个兴趣 token 交互，再由候选做 target attention。

MBCNet / PPNet：
加强 head 侧特征交叉和个性化调制。

评估：
BCE 训练，AUC 和用户级 GAUC 评估，重点看 test_random GAUC。
```

这套项目最重要的不是“用了多少模块”，而是每一步都对应 DIN 在多场景精排里的一个缺口：历史不随场景变、单 query 压平兴趣、兴趣之间缺少交互、head 交叉不够明确、不同用户和场景下特征权重不同。
