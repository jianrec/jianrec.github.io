---
title: "TAAC-2026 复盘：从稀疏特征、时间信号到 target-aware 序列建模"
date: 2026-06-24 21:00:00 +0800
tags: [TAAC, 推荐系统, PCVR, HyFormer, RankMixer, DIN, 特征工程, 模型复盘]
main_category: "LLM4Rec"
discipline: "LLM4Rec"
course: "TAAC-2026"
material_type: "竞赛复盘"
description: "整理 TAAC-2026 项目中围绕 sparse-dense pair、dense 拆分、时间特征、sequence gate、DIN target-aware 分支和训练稳定性的设计复盘。"
---

上一篇文章记录了我在 TAAC-2026 baseline 上做过哪些模块改造。这篇更偏“为什么这么改”：PCVR 任务里的稀疏性到底体现在哪些地方，为什么没有直接复刻更重的统一 Transformer 路线，以及我怎么在 HyFormer 框架里补上 pair 特征、时间信号和候选相关兴趣。

<!--more-->

相关的第一阶段总览在这里：[TAAC-2026：基于 baseline 的模型改进复盘]({{ '/notes/ai-agent-learning/taac-2026-baseline-improvements/' | relative_url }})。

## 问题不是单纯正样本少

PCVR 的稀疏性不只是“正样本少”。真正麻烦的是三件事叠在一起：

- 标签侧，负样本数量多，且存在转化延迟和负样本不确定性。
- 特征侧，高基数 ID 低频，很多字段如果直接 skip，相当于把一部分区分度扔掉。
- 序列侧，四个历史 domain 的覆盖率、时间新鲜度和相关性差异很大，固定平均或简单 concat 会把噪声也带进来。

所以我的改造思路不是单点换一个模型，而是分三层处理：

| 层次 | 主要问题 | 处理方式 |
| --- | --- | --- |
| 标签侧 | easy negative 主导训练，AUC 更关心排序 | `BCE + Pairwise`，小权重排序辅助项 |
| 特征侧 | 高基数、低频、sparse-dense 对齐信息容易丢 | hash embedding、缺失指示、pair-aware 建模 |
| 序列侧 | 多域历史质量不均，时间状态差异大 | event-level time feature、domain-level stat、sequence gate、DIN |

这也是我没有盲目堆大模型的原因。比赛数据规模和线上工业数据不是一个量级，很多工业论文里的统一 token 化、KV cache、复杂 mask 设计很有启发，但直接复刻未必划算。

## 为什么没有直接复刻 OneTrans / MTGR

我理解 OneTrans 的核心是把 sequential 和 non-sequential 特征统一 tokenizer 成 token sequence，再用一个 Transformer backbone 同时做序列建模和特征交互。它的优势是统一、可扩展、端到端，也可以配合 causal attention 和 cross-request KV cache 做推理优化。

但它的工程门槛也很高：

- token 设计复杂；
- mask 设计复杂；
- KV cache 和线上服务形态强绑定；
- 推理延迟和显存压力更大。

MTGR 这类 generative recommendation 路线也给了我一个重要提醒：生成式推荐和 scaling 有价值，但不能为了统一建模，把传统推荐里有效的 cross feature 全部丢掉。

所以这次我更倾向于保留 HyFormer 的轻量主干：用 query 读取多域历史序列，再用 RankMixer / DIN 补足非序列 token 交互和 candidate-history 匹配。这样做的目标不是架构最“统一”，而是在匿名特征、小数据比赛和显存限制下，让每一类信号都进入合适的位置。

## sparse-dense pair 不应该拆散

baseline 对用户 dense 的处理比较粗，容易把不同语义、不同尺度的 dense 字段直接 concat 后统一投影。我重点处理了两类结构：

- `fid61`、`fid87` 这种大块 dense；
- `62-66`、`89-91` 这种同时出现在 `user_int` 和 `user_dense` 里的 sparse-dense pair。

后者的关键不是 dense value 本身，而是同一位置上的 `(id, value)` 关系。value 不是孤立连续值，而是某个匿名 ID 对应的强度或统计信号。如果 int 和 dense 分开处理，模型就不知道第 `k` 个 value 属于第 `k` 个 ID。

我的处理方式是：

```text
pair 原始字段
-> 从 user_int / user_dense 中拆出 pair_int_feats 和 pair_dense_feats
-> 按 fid / offset / length 保持逐元素对齐
-> sparse id 走 embedding，dense scalar 走 Linear 投影
-> 用同一 valid mask 做 pooling
-> 拼入 coverage、mean、max、std 等统计
-> 得到 pair_emb，注入 user NS tokenizer
```

这里有两个细节很重要。

第一，`62-66` 的 dense 更像长尾强度或计数类信号，我会先做 NaN 置 0、非负裁剪和 `log1p` 压缩，降低极端值对 Linear 投影和梯度的支配。`89-91` 的数值尺度相对没那么长尾，因此保留原值更稳。

第二，pair 特征不是写回 `user_int`，也不是直接新增一个独立 NS token。它本质是用户侧 sparse 的补充信息，所以我把 `pair_emb` 作为 `extra_emb` 拼入 user RankMixer tokenizer 的输入，在切块生成 user NS token 之前经过 LHUC gate 做样本级重标定。这样 pair 信息能分布到多个 user token 里，又不会单独改变 HyFormer 的 token 结构。

## 大块 dense 要拆语义，不要全塞进一个 Linear

`fid61` 是 256 维，`fid87` 是 320 维。它们明显不是普通小维连续特征。如果把它们和 62-66、89-91 这类 pair dense 混在一起整体投影，会有两个问题：

- 大维度字段支配投影，早期训练容易压过小维字段。
- 不同来源和尺度的 dense 语义混杂，模型需要自己先做拆分再学习交互。

所以我把 user dense 拆成三条路径：

```text
fid61              -> 用户画像 dense token
fid87              -> 历史兴趣 dense token
fid62-66 / 89-91   -> sparse-dense pair embedding
```

其中 `fid87` 的 320 维可以自然拆成 `10 x 32`。我不能 100% 证明它的业务语义就是 10 个历史 item 摘要，但从维度结构上看，它更像多个同构 block 拼接而成。因此我给它加了一个轻量 attention pooling：

```text
fid87 [B, 320]
-> reshape [B, 10, 32]
-> block valid mask
-> Linear(32, 1) 得到 block score
-> softmax 得到 block 权重
-> 加权求和为 [B, 32]
-> 投影成 history dense token
```

这里的 attention pooling 不是 multi-head self-attention，没有 Q/K/V，也没有 block 间两两交互。它只是一个 learnable weighted pooling，让模型判断 10 个 dense block 里哪些更重要。

## RankMixer 有两个语境

项目里 RankMixer 容易混淆，因为它至少有两个语境：

- HyFormer block 内部的 RankMixer / Query Boosting，用于短 token 序列之间的 mixing。
- 输入侧的 `ns_tokenizer_type=rankmixer`，用于把原始 sparse embedding 构造成 NS tokens。

我这里讨论的重点是第二个。RankMixer tokenizer 会把各个 fid embedding 先整体 concat，再经过 LHUC/gate 重标定，最后 split 成固定数量的 chunk，并投影为 user/item NS tokens。

主目录把 `user_ns_tokens` 从 baseline 的 5 提到 12，原因是用户侧特征维度和语义复杂度变高了，尤其是 pair-aware embedding 被注入 user tokenizer 之后，如果仍压成 5 个 user token，很容易形成信息瓶颈。12 个 token 不是手工对应 12 类语义，而是给 RankMixer tokenizer 更多子空间，让它自己学习字段组合。

这里 LHUC gate 的作用是做样本级 feature-wise 重标定：

```text
cat_emb' = cat_emb * 2 * sigmoid(MLP(cat_emb))
```

它不是新增复杂交叉结构，而是在切块前对 user sparse + pair embedding 的长向量做一次校准。pair 特征表达力更强，也更可能带来噪声；LHUC 可以学习什么时候强化 pair，什么时候压低不可靠维度。

需要注意的是，`fid61/fid87` 拆出的两个 dense token 不在这 12 个 user NS token 里面。它们是单独编码成 dense NS token，再和 user/item NS tokens concat。

## 时间信号分成 event-level 和 domain-level

我把时间信息分成两层。

event-level 是每条历史行为位置上的时间特征。除了离散 time bucket，我还为每个行为构造 8 维细粒度时间特征，包括 recency、小时/星期周期、inter-event gap 等。它们会投影后加到每个 seq token 上，让历史行为 token 自己带有时间语境。

domain-level 是整条序列的质量和新鲜度统计，比如：

- `max / min / mean recency`
- `15min / 1h / 1d` 内行为数
- 有效长度和 coverage
- time bucket embedding 的 masked mean pooling

这些统计进入两个地方：

1. query generation，让 query 在 cross-attention 前就带有该 domain 的时间新鲜度和活跃度。
2. sequence gate，让模型判断四个历史 domain 当前哪个更值得信任。

sequence gate 不是固定平均四个 domain，而是每个 domain 先得到一个整体兴趣表示，再结合时间统计和覆盖率生成动态权重：

```text
seq_repr = mean(q_tokens_domain)
stat_emb = Linear + LayerNorm + SiLU(seq_stats)
time_pool = masked_mean(time_bucket_embedding)
gate_input = concat(seq_repr, stat_emb, time_pool)
```

我没有把长度、coverage、时间统计直接原样拼进去，而是先投影成 `stat_emb`。原因很实际：这些统计是低维连续值，和 `[B, D]` 的 query embedding 在维度和尺度上不匹配。先经过 `Linear + LayerNorm + SiLU`，可以让模型学习“长度很长但很旧”“长度很短但很新”这类组合语义，也能让 gate 训练更稳定。

gate 里还需要防止早期塌缩。训练初期如果 gate 过早把权重集中到某一两个 domain，其他分支权重接近 0，梯度就很弱，后续很难学回来。因此我会在 softmax gate 权重中混入少量 uniform mixing：

```text
w_final = (1 - alpha) * w_gate + alpha / N
```

它不是为了最终平均融合，而是给每个 domain 保留一点探索空间，避免早期分支被饿死。

## DIN 是输出侧 target-aware 补充分支

HyFormer 主干更像是在学习通用的多域历史兴趣，但 PCVR 最终要回答的是：这个用户对当前候选广告是否可能转化。因此我额外加了 DIN 风格的 target-aware 分支。

先构造候选广告的 `candidate_anchor`：

```text
item sparse fields -> embedding / pooling -> concat -> target_item_proj -> candidate_anchor
```

然后在每个 domain 内，把候选 item 和历史行为 token 做交互：

```text
score_i = MLP([q, h_i, q - h_i, q * h_i])
alpha_i = softmax(score_i)
context_domain = sum(alpha_i * h_i)
```

这里的 `alpha_i` 是 DIN attention 权重，表示第 `i` 条历史行为和当前候选广告的相关性。真正的兴趣表示是 `context_domain`，也就是当前候选广告相关的历史兴趣，而不是全局平均兴趣。

最后四个 domain 的 DIN context 再通过 sequence gate 加权融合，和 HyFormer output、candidate anchor 一起进入输出侧融合。这个分支不替换主干，而是补足候选相关匹配能力。

我对 residual 和 concat 两种融合都做过思考：

- `output + gate * residual` 更像对主干表示做修正，简单、可控，但 residual 噪声大时会扰动主干。
- `concat(output, din_context, item_target)` 更像特征级融合，不强行假设 DIN 是主干 output 的修正方向，更灵活，但参数更多。

在比赛里，我更倾向把 DIN 当作输出侧补充分支，而不是完全推翻 HyFormer 主干。

## 损失函数和 EMA 更偏稳定性

训练侧我最后更认可 `BCE + Pairwise`，而不是把 Focal 或 Asymmetric Loss 作为主路线。

Focal Loss 的优点是简单，能在 BCE 基础上降低 easy sample 权重，让模型关注 hard sample。但 PCVR 任务不仅看排序，也看概率质量；如果叠加 label smoothing 和 Focal，可能会削弱 hard sample 聚焦或影响校准。

主目录正式配置更像这样：

```text
L = BCEWithLogits(label_smoothing=0.01) + lambda * softplus(s_neg - s_pos)
```

BCE 保留概率学习和校准，pairwise 项贴近 AUC 排序目标。pairwise 项用小权重，是为了避免排序辅助项压过主任务。

EMA 的收益也更多来自稳定性，而不是表达能力增强。广告 PCVR 数据噪声大，高基数 ID embedding 更新稀疏，gate/attention 模块又容易受 batch 分布影响，单个 checkpoint 会有 step-level 波动。EMA 相当于低成本 checkpoint ensemble。

但全量 EMA 会复制一份很大的 sparse embedding，显存和存储成本都高。因此更合理的方向是 dense-only EMA：只平滑 HyFormer、projection、fusion 和 prediction head 等 dense 参数，保留 sparse embedding 的当前学习结果。

## 验证集最好更接近“用过去预测未来”

row-group split 很快，因为 Parquet 本身就是按 RowGroup 组织数据。如果原始数据写入时基本按时间排序，RowGroup tail split 可以近似时间切分；但如果 RowGroup 内部时间混杂，它就不是真正的未来验证集。

timestamp-based validation split 更严谨：显式用样本时间切分，较早时间进 train，较晚时间进 valid。它更接近线上广告预测里的“用过去预测未来”，也更能暴露时间外推和分布漂移问题。

代价是实现和读取更麻烦，验证窗口太窄时指标方差也可能更大。比赛里这两种切分都值得看，但最终判断模型泛化时，我更信任时间切分。

## 这次复盘最大的收获

第一，PCVR 的稀疏性要拆开看。正样本少只是表面，高基数 ID 低频、pair 对齐关系、历史 domain 覆盖不均，都会让模型学不到稳定信号。

第二，unified tokenization 是趋势，但 unified 不等于把所有特征粗暴丢进 Transformer。真正难点在 token 设计、mask 设计、长序列效率、缓存和延迟约束。小数据比赛里，轻量 HyFormer + targeted feature modeling 反而更可控。

第三，时间信号不能只当 recency bucket。event-level 时间特征告诉模型每条行为是什么时间状态；domain-level 统计告诉 gate 这个历史域整体是否新鲜、是否可靠。

第四，DIN 的价值不在于替代序列主干，而在于明确回答“当前候选广告和哪些历史行为相关”。这件事对 PCVR 比泛化的用户兴趣平均更直接。

后面如果继续做，我不会再一次性堆很多模块，而会把消融拆得更细：pair-aware、dense split、event time、sequence gate、DIN、EMA、pairwise loss 分别验证，再看它们是否表达了重复信号。比赛里的提升最终不是模块列表越长越好，而是每个新增分支都能解释它解决的稀疏性问题。
