---
title: "TAAC-2026: A Retrospective on Model Improvements over the Baseline"
date: 2026-05-24 20:55:00 +0800
tags: [TAAC, Recommender Systems, CVR, Model Retrospective]
main_category: "Paper Reading"
sub_category: "LLM4Rec"
discipline: "LLM4Rec"
course: "TAAC-2026"
material_type: "Competition Retrospective"
description: "A record of the main changes the TAAC-2026 code makes relative to the baseline, the current scores, and the ablation directions to follow."
lang: en
ref: "taac-2026-baseline-improvements"
---

This post records the main changes I made to the TAAC-2026 code on top of the official baseline, and puts the current scores up front so they are easy to look back at. "Official baseline" here refers to the original `baseline/train` code shipped with the competition; `0.8289*` is the score after the first round of improvements on the official baseline, not the score of the baseline itself.

<!--more-->

## Performance

| Version | Score | Notes |
| --- | --- | --- |
| Official baseline | `0.813` | The original baseline provided by the competition |
| Improved baseline | `0.8289*` | Recorded after the first round of improvements on the official baseline |
| Current code | `0.830368`, roughly `0.8303*` | The Best Score in the screenshot |

![Screenshot of the Best Score for the current TAAC-2026 code](/assets/images/taac-2026-performance.jpeg)

The full code is on GitHub: [zliaaan/TAAC-2026](https://github.com/zliaaan/TAAC-2026).

## Why change the baseline

The baseline's HyFormer backbone can handle multiple behavior sequences, but the feature semantics are still fairly coarse:

- The user-side fields contain pair-type features; treating them as ordinary user features directly loses the `category id + value/score` pairing relationship.
- The time signal mostly relies on buckets — recency, gaps, hour of day, day of week, and coverage within the sequence are not modeled adequately.
- The number of NS tokens is tightly coupled to the grouping, so the representational granularity is not flexible enough.
- The matching between the target item and historical behaviors is not direct enough.
- The training configuration depends on runtime arguments, so checkpoints easily end up structurally inconsistent in the inference environment.

So the focus of this version of the code is not to keep stacking on a bigger model, but to cleanly separate user, item, sequence, time, and training stability.

## Main improvements

### Modeling pair features separately

The following fields were split out of the ordinary user features:

```python
PAIR_FEATURES = [62, 63, 64, 65, 66, 89, 90, 91]
```

On the data side I added `pair_int_schema` and `pair_dense_schema`; on the model side I added `CrossRankMixerNSTokenizer`. Pair int and pair dense are modeled with positional alignment, then injected into the user NS tokenizer as an extra embedding.

Among them, the dense values for `62-66` go through `log1p` compression to reduce the influence of long-tail counts and outliers; `89-91` keep their similarity- or score-type dense semantics.

### Hash embeddings for high-cardinality fields

I added the following arguments to handle high-cardinality fields:

```bash
--emb_skip_threshold
--hash_bucket_size
--pair_hash_bucket_size
--pair_hash_fields
--pair_hash_bucket_map
```

When a field exceeds the embedding threshold, a fixed-bucket `HashEmbedding` can retain part of the ID signal instead of zeroing it out entirely.

### RankMixer NS tokenizer

The current main line defaults to:

```bash
--ns_tokenizer_type rankmixer
--user_ns_tokens 12
--item_ns_tokens 2
```

The RankMixer tokenizer first concatenates all group embeddings, applies a sample-level rescaling through an LHUC gate, and then splits the result into a tunable number of NS tokens. This way the user/item NS token counts are no longer tightly coupled to `ns_groups.json`.

### Splitting user_dense into two semantic tokens

`user_dense` no longer goes through a single projection as a whole; it is split into:

- A user embedding-type dense token.
- A historical item embedding-type dense token.

The historical item embedding is first split into blocks, then aggregated over the valid blocks with learnable attention pooling, which reduces the contamination of the dense representation by all-zero users and users with weak history.

### Time feature enhancement

The original time bucket is kept, and in addition each sequence position gets 8 float time feature dimensions:

| Channel | Meaning |
| --- | --- |
| `ch0` | `log1p(diff_days)` |
| `ch1` | A domain-specific time scale: months for `seq_c`, hours for `seq_d`, days for `seq_a/b` |
| `ch2` | `log1p(diff_hours)` |
| `ch3/ch4` | cos/sin of hour-of-day |
| `ch5/ch6` | cos/sin of day-of-week |
| `ch7` | inter-event gap, with a different scale per domain |

Each sequence domain also gets 6 statistical features:

- `log1p(max_diff)`
- `log1p(min_diff)`
- `log1p(mean_diff)`
- `count_<=15min`
- `count_<=1h`
- `count_<=1d`

These statistical features feed into the query generator and the sequence domain gate.

### No aggressive time filtering

When constructing the sequence time features, I preserve the original padded-slot alignment and no longer forcibly compact/filter events by `sample_ts`; only negative time differences are clipped. This avoids misalignment between sequence fields and time fields, and avoids breaking the original positional structure through filtering.

### DIN target-aware branch

With `--use_din` enabled, the target item attends over the historical tokens of each sequence domain, producing a target-aware history context. Finally the HyFormer output, the DIN context, and the item target are fed together into the output tower.

This branch does not replace the HyFormer backbone; it only augments the output side.

## Training stability

On the training side, the main additions are:

- `bce_pairwise`: `BCEWithLogits + pairwise_lambda * batch_pairwise_loss`
- `label_smoothing=0.01`
- `ModelEMA`
- Dual optimizers: AdamW + Adagrad
- bf16 autocast
- Gradient clipping
- warmup + cosine decay
- Checkpoints ship with `schema.json` and `train_config.json`

Self-contained checkpoints matter quite a bit here. `infer.py` first reads `train_config.json` from the checkpoint directory, then strictly rebuilds the model and loads the weights, avoiding structural mismatch between training and inference.

## Version branches

| Directory | Role |
| --- | --- |
| `./` | Main line: pair features, time enhancement, RankMixer, DIN, pairwise loss, EMA |
| `variant-omega-fusion/` | Combines CrossNet, SE-Net, NS self-attention, NS output fusion, target-aware attention, and a TIME_MATCH residual |
| `variant-omega-fusion-focus-ad/` | Adds a sequence focus gate for `seq_a` and `seq_d` on top of the omega fusion version |

## Ablation order going forward

I will run ablations in roughly this order:

1. Turn off `--use_din` to verify whether the DIN output-side enhancement contributes stably.
2. Tune `pairwise_lambda` to `0.01/0.05` to judge whether the ranking auxiliary term is too strong.
3. Turn off EMA to confirm whether the gains come from the model itself or from weight smoothing.
4. Tune `user_ns_tokens` to `8/10/12` to find the balance between NS token granularity and parameter count.
5. Turn off the 8-dimensional float time features and keep only the buckets, to verify whether the time enhancement generalizes.
6. For the omega version, turn off CrossNet, SE-Net, and NS output fusion one at a time to look for redundant injection of static signals.
7. For the focus-ad version, turn off `seq_focus_*` to verify whether the `seq_a/seq_d` prior is genuinely effective.

Going forward I would not recommend stacking many modules at once. In this task several modules express similar signals, and combining them can produce offline gains that are unstable online.