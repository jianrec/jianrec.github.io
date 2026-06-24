---
title: "KuaiRand 跨域多场景精排项目面试复习：从 DIN 到 PSRG/PCRG"
date: 2026-06-24 19:20:00 +0800
tags: [KuaiRand, 推荐系统, 精排, DIN, 跨域推荐, 多兴趣, PSRG, PCRG, GAUC, 面试复习]
main_category: "LLM4Rec"
discipline: "LLM4Rec"
course: "KuaiRand 跨域精排"
material_type: "项目复盘"
description: "整理 KuaiRand 跨域多场景精排项目的业务问题、建模假设、模块设计、代码口径、评估指标和面试追问答案。"
---

这篇是 KuaiRand 跨域多场景精排项目的面试复习稿。重点不是背模块名，而是把“为什么有这个问题、为什么这样建模、每个模块解决什么、证据在哪里、面试时怎么收敛口径”讲清楚。

<!--more-->

## 一句话定位

这个项目是基于 KuaiRand-27K 曝光日志的 pointwise 短视频精排任务。每次曝光构成一个样本，模型利用用户此前跨 tab 的历史行为、当前候选视频、用户画像和当前场景上下文，预测用户是否会对候选视频产生长观看。

稳定面试口径：

> 我做的是 KuaiRand-27K 多场景短视频精排项目，主任务是预测 `label_long_view`。Baseline 是 DIN，用候选 item 对历史序列做 target attention。核心问题是用户历史来自多个 tab，而当前预测发生在某个目标 tab，不同 tab 的用户意图、内容分布和反馈规律不同。因此我用共享 embedding 复用跨场景共性，再用 `d_ctx`、PSRG 和 PCRG 做目标域适配和多兴趣抽取，最后用 AUC 和用户级 GAUC 评估。

## 先分清场景和域

面试里最容易混的是“跨场景”和“跨域”。我建议这样说：

| 概念 | 含义 | 在项目里的例子 |
| --- | --- | --- |
| 场景 | 业务入口或 tab | 推荐、关注、附近、商城等入口 |
| 域 | 某个入口下形成的数据分布 | 关注页的数据分布、附近页的数据分布 |
| 跨场景现象 | 用户历史行为可能来自多个入口 | 用户在关注页看美妆，在推荐页看测评 |
| 跨域建模问题 | 不同入口的数据分布不同，需要共享又要适配 | 关注页更受作者关系影响，商城页更受商品意图影响 |

域不是单独的一列特征，而是一类样本的整体概率分布。按 tab 划分后，不同 tab 在用户、内容、曝光机制和反馈规律上存在差异，因此每个 tab 可以看作一个数据域。

可以用这个例子解释：

```text
历史行为：
关注页看过美妆博主
推荐页看过粉底测评

当前请求：
在商城页预测用户是否长观看粉底商品视频
```

同一个历史视频或作者，在不同当前 tab 下可能有不同预测语义。关注页里的美妆博主可能表示关系链偏好，商城页下它可能更像商品购买意图的旁证。

## 数据和反馈目标

当前项目是单目标长观看预测，主标签是 `label_long_view`。数据里还有点击、点赞、播放比例等反馈，但它们在当前训练主线里不是多任务 label。

按代码口径拆开说：

| 信号 | 作用 |
| --- | --- |
| `long_view` | 主预测目标，生成 `label_long_view` |
| `is_click` | 历史序列入池信号之一 |
| `is_like` | pre 期候选统计特征之一 |
| `play_time_ms / duration_ms` | 播放比例统计和历史播放比例分桶 |
| pre 期 CTR/LV/Like/PlayRatio | 候选侧统计特征，避免直接泄漏测试期反馈 |

真实代码里，样本 label 和历史入池逻辑在 `preprocess/step3_generate_samples_v2.py`：

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

```python
# preprocess/step3_generate_samples_v2.py:142
if row['is_click'] == 1 or row['long_view'] == 1:
    curr_time = row['time_ms']
    prev_time = history[-1]['time'] if history else curr_time
    delta_t = curr_time - prev_time
    play_ratio = min(row['play_time_ms'] / max(row['duration_ms'], 1), 3.0)
```

候选统计特征只用 pre 期日志计算，避免把实验期反馈泄漏进训练：

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

面试时要避免说成“点击、长观看、点赞、播放比例都是训练目标”。更准确的说法是：

> 当前主任务是 pointwise 长观看预测。点击和长观看用于构造历史正反馈序列，点赞和播放比例主要进入候选统计或历史辅助特征，最终 loss 仍然是围绕 `label_long_view` 的 BCE。

## 整体模型链路

代码主线是：

```text
DIN baseline
-> ADS: PSRG + PCRG
-> TransformerFusion
-> MBCNet
-> PPNet
```

逻辑主线是：

```text
共享基础 embedding
-> 复用不同 tab 的视频/作者共性

当前场景生成 d_ctx
-> 表示当前请求所在的目标域

PSRG
-> 把跨 tab 历史重映射到当前目标域语义下

PCRG
-> 根据当前候选和目标域生成多个 query，抽取候选相关多兴趣

TransformerFusion
-> 让多个兴趣 token 先交互，再用候选做 target attention

MBCNet / PPNet
-> 在最终 head 侧加强特征交叉和个性化调制

最终预测
-> 当前场景下用户是否长观看候选视频
```

## Baseline DIN 解决什么

Baseline 选 DIN 是因为精排任务不是只建用户静态画像，而是要判断“当前候选视频”和“用户历史行为”是否相关。

代码中候选和历史使用同一套 video/author embedding 表，这一点很关键：

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

DIN attention 的打分方式是经典 `[q, h, q-h, q*h]`：

```python
# src/models/modules/target_attention_dnn.py:125
q = query.unsqueeze(1).expand(-1, num_tokens, -1)
att_input = torch.cat([q, keys, q - keys, q * keys], dim=-1)
att_scores = self.mlp(att_input).squeeze(-1)

att_weights, all_pad = masked_softmax(att_scores, mask, dim=-1)
user_interest = torch.bmm(att_weights.unsqueeze(1), keys).squeeze(1)
```

Baseline 的三个局限：

1. 历史 item embedding 是共享静态表示，对当前 tab 不敏感。
2. DIN 是单 query，只能把历史压成一个兴趣向量。
3. 最后的 concat + MLP head 对显式特征交叉的结构先验不足。

## 为什么共享 embedding 还不够

我不是为每个 tab 学一套独立 embedding，而是让候选侧、历史侧和不同 tab 样本共用同一张 video/author embedding 表。

这样做的好处：

- 复用跨域共性，避免每个 tab 数据变少。
- 保证候选和历史处于同一个向量空间。
- 让低频视频、低频作者能从其他 tab 的行为里获得训练信号。

但共享 embedding 只解决“共性复用”，不解决“当前域下如何重新解释历史”。所以项目的问题不是 embedding 参数不更新，而是原始 item embedding 对当前域不敏感。

面试短答：

> 底层共享 embedding 学视频和作者跨场景的稳定共性，PSRG 不替代共享 embedding，而是在共享表示之上加入当前场景条件，让同一历史行为在不同当前场景下呈现不同预测语义。

## d_ctx 表示当前目标域

`d_ctx` 由当前请求上下文编码而来，不是简单把 tab id 拼到最后。代码里 `DomainContextEncoder` 接收已经 lookup 好的上下文 embedding，再经过 MLP 和 LayerNorm：

```python
# src/models/modules/domain_context.py:71
def forward(self, ctx_concat: torch.Tensor) -> torch.Tensor:
    d_ctx = self.encoder(ctx_concat)
    d_ctx = self.layernorm(d_ctx)
    return d_ctx
```

主配置中 `d_ctx` 使用 `tab`、`hour_of_day`、`day_of_week`，输出维度是 48。它被 PSRG 和 PCRG 共享。

面试短答：

> `d_ctx` 表示当前请求所在目标域。它来自 tab 和时间上下文，是后续 PSRG 重映射历史、PCRG 生成 query 的共同条件。

## PSRG 解决同物不同义

PSRG 解决的是历史表示对当前域不敏感的问题。同一个历史 item 在不同当前 tab 下，应该允许被重解释。

当前代码是 `PSRG-lite`，不是完整 HyperNetwork，不为每个样本生成一整套 private MLP 权重。它支持 FiLM 和 gated residual，主配置默认是 `gated_residual`：

```python
# src/models/modules/psrg.py:95
gate = torch.sigmoid(self.gate_mlp(merged)).reshape(B, L, D)
delta = self.delta_mlp(merged).reshape(B, L, D)
hist_out = hist_repr + gate * delta

hist_out = self.layernorm(hist_out)
```

为什么这样做：

- 完整动态生成大矩阵成本高，训练也更不稳定。
- FiLM / gated residual 保留了“域条件调制”的核心思想。
- 每个历史位置都能根据 `hist_repr` 和 `d_ctx_seq` 得到不同调制。
- padding 位被 mask 保护，避免无效历史污染注意力。

面试短答：

> PSRG 改的是历史序列表示。它用当前场景 `d_ctx` 对每条历史 item 做轻量动态重映射，让历史行为先变成目标域下的语义，再交给 DIN 或 PCRG 做 attention。

## 如何回答“为什么历史 emb 在不同域下应该不一样”

这里不要说“我能先验证明”。更严谨的回答是：

> 我不能先验地证明它一定应该不一样，这是一个建模假设。但我有四层证据来支撑它。

四层证据：

1. 业务逻辑上，同一个视频在关注 tab 和发现 tab 表达的偏好语义不同。
2. 数据分布上，同一 item 在不同 tab 下的点击、长观看、播放比例分布可能分裂，它对候选的预测力也会随 tab 变化。
3. 消融上，把 `d_ctx` 置零或只保留重映射层容量，如果增益明显缩水，就说明收益来自域条件，而不是多加一层网络。
4. 表征上，训练后比较同一历史 item 在不同 `d_ctx` 下的 PSRG 输出，如果向量距离被拉开，说明模型确实学到了域相关重解释。

这段回答的关键是承认“这是可检验假设”，然后给出业务、数据、消融、表征四类证据。

## PCRG 解决单 query 压平多兴趣

DIN 的单 query 会把历史聚合成一个向量。用户历史里可能同时有美妆、探店、影视、体育等兴趣，一个向量容易压平。

PCRG 的做法是由当前候选和当前目标域生成多个 query：

```python
# src/models/modules/pcrg.py:168
q_input = torch.cat([cand_item_repr, d_ctx], dim=-1)
q = self.query_gen(q_input).reshape(B, self.num_queries, self.query_dim)
q_item = self.query_to_item(q)
```

然后每个 query 分别 attend 历史：

```python
# src/models/modules/pcrg.py:173
scores = self._score_queries_with_hist(q_item, hist_repr)
mask_3d = hist_mask.unsqueeze(1).expand(-1, self.num_queries, -1)
attn, all_pad = _masked_softmax(scores, mask_3d, dim=-1)
z = torch.einsum("bgl,bld->bgd", attn, hist_repr)
```

区分两个概念：

| 概念 | 含义 |
| --- | --- |
| query | 由候选和场景生成的检索视角 |
| interest token | query attend 历史后得到的兴趣结果 |

面试短答：

> PCRG 不是预先存一组用户兴趣原型，而是用当前候选和当前场景动态生成多个 query。每个 query 从 PSRG 重映射后的跨 tab 历史里抽取一个兴趣 token，所以它是 candidate-aware + domain-aware 的多兴趣提取。

## 和 MIND / ComiRec 的区别

MIND 和 ComiRec 也做多兴趣，但它们更偏用户级静态多兴趣：从用户历史中聚合多个兴趣，然后候选侧再路由或匹配。

PCRG 的区别：

| 维度 | MIND / ComiRec | 本项目 PCRG |
| --- | --- | --- |
| 兴趣生成 | 主要从用户历史生成 | 由候选和场景生成 query 后检索历史 |
| 是否候选相关 | 候选主要用于后续匹配 | query 生成阶段就引入候选 |
| 是否域相关 | 不一定显式建模当前 tab | 显式引入 `d_ctx` |
| 适合位置 | 召回或粗排多兴趣表达 | 精排中的候选相关兴趣抽取 |

面试短答：

> ComiRec/MIND 的多兴趣更像用户级静态多兴趣；PCRG 是 candidate-aware + domain-aware 的动态多 query。精排本来就是候选相关任务，所以让 query 由当前候选和当前场景生成更贴合。

## 和 STAR / PEPNet / PEP 的关系

STAR、PEPNet、PEP 都属于多域或个性化条件调制思想。区别不在原则，而在作用位置。

| 方法 | 核心思想 | 作用位置 |
| --- | --- | --- |
| STAR | 共享权重和域特定权重做组合 | 网络权重或 tower |
| PEPNet / PEP | 生成缩放、偏置或门控参数 | embedding、tower 或 head |
| PSRG | 用域上下文调制历史 item 表示 | 历史序列逐位置重映射 |
| PCRG | 用候选和域上下文生成多个 query | 多兴趣抽取 query 侧 |

面试短答：

> 我的 PSRG 和 STAR/PEPNet 属于同一思想家族，都是 shared backbone + domain-conditioned modulation。但 STAR/PEPNet 多作用在整个网络或 head，PSRG 把 domain-conditioning 下沉到历史序列逐位置重映射，PCRG 则把域条件带进 query 生成，用于序列兴趣抽取。

## TransformerFusion 解决兴趣 token 之间缺少交互

PCRG 输出多个 interest token，但它们默认相互独立。TransformerFusion 让兴趣 token 之间先做 self-attention，再用候选做 target attention。

代码里默认是对 interest tokens 做 Transformer，不是对 50 条原始历史直接做：

```python
# src/models/modules/transformer_fusion.py:237
if self.n_layers > 0:
    padding_mask = token_mask <= 0
    for layer in self.encoder_layers:
        x = layer(x, padding_mask=padding_mask)
```

然后用候选作为 query 做 target attention：

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

为什么先 PCRG 再 Transformer：

- 原始历史长度是 50，PCRG interest token 通常是 4 个，计算更轻。
- PCRG 已经引入候选和域条件，先筛出更干净的兴趣 token。
- Self-attention 负责兴趣之间的相似、互补、冲突关系。
- Target attention 负责“当前候选最相关哪个兴趣”。

面试短答：

> DIN 的 target attention 关注候选和历史是否相关；self-attention 关注历史内部或兴趣 token 内部如何相互影响。在我的项目里，TransformerFusion 主要作用在 PCRG 输出的多个兴趣 token 上，让多兴趣先交互，再由候选视频做 target attention 聚合出当前候选最相关的兴趣表示。

## MBCNet / FGC 解决 head 侧交叉不足

前面的模块主要解决序列兴趣建模。最终精排还依赖很多非序列特征：

- `user_interest`
- `cand_repr`
- 用户 sparse / dense 画像
- 当前 tab / 时间上下文
- 候选侧属性
- 候选 pre 期统计特征

Baseline 的 head 是 concat + MLP。MLP 能隐式学习交叉，但没有显式结构先验。

MBCNet 的三个分支：

| 分支 | 解决问题 | 直觉 |
| --- | --- | --- |
| FGC | 语义 group 内显式交叉 | 在用户兴趣、候选、用户画像、上下文等组内做轻量交叉 |
| Low-rank Cross | 全局显式交叉 | 用低秩方式近似全局二阶交叉 |
| Deep | 隐式高阶组合 | 保留 MLP 泛化能力 |

FGC 的 group 内交叉支持 `cross1`、`bilinear`、`gated`：

```python
# src/models/modules/mbcnet.py:86
if self.mode == "cross1":
    self.param = nn.Linear(self.group_dim, 1, bias=True)
elif self.mode in {"bilinear", "gated"}:
    self.param = nn.Linear(self.group_dim, self.group_dim, bias=True)
```

特征 group 来自 head input 的语义切片：

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

如果去掉 FGC，只保留全局低秩交叉和 Deep 分支，可以称为 Low-rank Cross-Deep Head。本质是全局低秩显式交叉 + DNN 隐式交叉。

面试短答：

> FGC 一方面按语义把 head input 分成用户兴趣、候选、用户画像、场景、候选属性等 group；另一方面在每个 group 内做轻量显式交叉。这样比普通 MLP 更有结构先验，能增强特定字段组的记忆能力。

## PPNet 是代码主线的最后一步

当前仓库 README 的最终链路是 `+PPNet`。如果简历里没有写 PPNet，可以把它作为代码主线补充，不要和其他离线扩展混成一件事。

PPNet 的目标是让不同场景、用户类型、活跃度状态下的 head 输入被不同方式调制。`p_ctx` 来自场景、用户属性和活跃度代理：

```python
# src/models/modules/personal_context.py:159
if context_embs is not None and context_embs.shape[-1] > 0:
    parts.append(context_embs)
```

默认实现是 group-wise FiLM：

```python
# src/models/modules/ppnet.py:179
params = self.out_proj(self.backbone(p_ctx))
gamma, beta = torch.chunk(params, 2, dim=-1)

x_norm_before = x.norm(dim=-1).mean()
x_mod = self._apply_group_film(x=x, gamma=gamma, beta=beta)
```

面试短答：

> PPNet 不改变上游兴趣抽取，而是在 head 侧根据当前场景、用户画像和活跃度代理生成轻量调制参数。它让同一套特征在不同用户和不同 tab 下有不同权重。

## 评估指标和边界

当前项目可以稳定说 AUC 和 GAUC。

全局 AUC：曝光样本级，把所有样本放一起算。

用户级 GAUC：按 `user_id` 分组，每个用户内部算 AUC，再按样本数加权。代码会跳过只有正样本或只有负样本的用户：

```python
# src/metrics/metrics.py:230
if n_pos == 0 or n_neg == 0:
    continue
```

为什么不能严格算请求级 NDCG：

> 当前项目有曝光样本级候选、预测分数、真实标签和 `user_id`，所以可以计算全局 AUC 和用户级 GAUC。但缺少请求级候选分组，也就是同一个 request 下的一组候选列表，因此不能计算严格的 request-level NDCG。

为什么重点看 `test_random`：

> standard log 来自线上策略分发，样本选择偏置更强；random log 来自随机曝光，虽然样本更小，但更适合观察模型真实排序能力。

README 记录的 full track 结果：

| 阶段 | test_standard GAUC | test_random GAUC | random 增量 |
| --- | ---: | ---: | ---: |
| DIN baseline | 0.6761 | 0.5525 | - |
| +ADS | 0.6854 | 0.5698 | +0.0173 |
| +Transformer | 0.6880 | 0.5745 | +0.0047 |
| +MBCNet | 0.6905 | 0.5775 | +0.0030 |
| +PPNet | 0.6918 | 0.5815 | +0.0040 |

面试里要注意：

- `0.6918` 是 `test_standard GAUC`，不要说成 random。
- `test_random GAUC` 最终是 `0.5815`。
- full track random 从 `0.5525` 到 `0.5815`，绝对提升 `0.0290`。

## 高频追问短答

### 1. 这个项目到底解决什么问题

> 解决多 tab 精排中的跨域兴趣复用和场景适配问题。共享 embedding 复用视频/作者跨场景共性，PSRG 用当前场景重解释历史行为，PCRG 用候选和场景生成多个 query 抽取多兴趣，最终预测当前场景下的长观看概率。

### 2. 为什么不为每个 tab 训练一套 embedding

> 每个 tab 单独训练会割裂数据，长尾视频和作者更稀疏，也会让候选和历史表示空间不统一。共享 embedding 学共性，PSRG/PCRG 在共享表示上做目标域适配，是共享和差异之间的折中。

### 3. PSRG 是不是生成完整 private weight

> 当前代码不是完整 HyperNetwork，而是 PSRG-lite。共享 MLP 根据 `hist_repr` 和 `d_ctx` 生成 `gamma/beta` 或 `gate/delta` 这类轻量调制参数，对每个历史位置做动态重映射。

### 4. PCRG 生成的是 query 还是 interest token

> 先生成 query，再由 query attend 历史得到 interest token。query 是检索视角，interest token 是从历史里检索出来的兴趣结果。

### 5. TransformerFusion 的输入是历史序列吗

> 主配置默认输入 PCRG 产生的 interest tokens，不是 50 长度原始历史。这样计算更省，噪声更少，也更贴合“多兴趣融合”这个目标。

### 6. Self-attention 和 DIN attention 有什么区别

> Self-attention 建模 token 内部关系，让兴趣之间相互交互；DIN target attention 用候选作为 query，判断当前候选和哪些历史或兴趣最相关。前者是内部融合，后者是候选对齐。

### 7. MBCNet 是不是只处理非序列特征

> 不是。MBCNet 处理最终 `head_input`，里面既有上游得到的 `user_interest`，也有候选、用户画像、上下文和候选属性。它是最终打分前的特征交叉 head。

### 8. 为什么用 GAUC

> 全局 AUC 容易被曝光量大的用户主导。GAUC 先在用户内部算排序质量，再按用户样本数加权，更贴近推荐场景里的用户级排序效果。

### 9. 为什么不能算严格 NDCG

> 缺少 request 级候选分组。没有同一次请求下的候选列表，就不能定义每个请求内的排序位置和 DCG。

### 10. 怎么证明增益不是单纯来自参数变多

> 要做反证消融。比如保留 PSRG/PCRG 的网络容量但置零或打乱 `d_ctx`，如果增益缩水，说明收益主要来自域条件；再做 `G=1` 和 `G>1` 对比，可以隔离多 query 多兴趣的增益。

## 最终复习主线

背项目时按这条线讲：

```text
业务现象：
用户历史来自多个 tab，当前请求发生在一个目标 tab。

建模问题：
不同 tab 的数据分布不同，需要共享跨场景共性，同时保留目标域差异。

Baseline：
DIN 用候选对历史做 target attention，但历史表示静态、单 query、head 交叉弱。

共享 embedding：
学习视频和作者跨场景共性，保持历史和候选同空间。

d_ctx：
编码当前 tab/time 目标域。

PSRG：
用 d_ctx 对每条历史 item 做场景条件化重映射。

PCRG：
用候选和 d_ctx 生成多个 query，从跨 tab 历史里抽取多兴趣。

TransformerFusion：
多兴趣 token 先 self-attention，再由候选 target attention 聚合。

MBCNet / PPNet：
加强最终 head 的显式交叉和个性化条件调制。

评估：
pointwise BCE 训练，曝光样本 AUC 和用户级 GAUC 评估，重点看 test_random GAUC。
```

一句压轴：

> 这个项目不是简单把多场景历史拼起来，而是在共享 embedding 的基础上，用当前目标域重新解释历史行为，再用候选和目标域动态抽取多兴趣，从而预测用户在当前场景下是否会长观看候选视频。
