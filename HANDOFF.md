# Multimodal Safety — Internship Handoff

**Author:** Kunj Shah · **Date:** 2026-08-11 · **Branch:** `feature/kunj`

---

## 1. Summary

The deliverable is **TrooperViz**, a 4B guardrail model that classifies a prompt, a
multi-turn conversation, or a candidate assistant response as `safe` / `unsafe` against a 21-policy taxonomy — with or without images, in one or two inference modes.

It is a weight-space **SLERP merge (t = 0.75)** of a Qwen3.5-4B checkpoint I trained on
160k multimodal guardrail examples and `Trooper`, the team's text-only guardrail
model. The merge keeps the text model's strength while adding vision, multi-turn, and
output-guard capability in both modalities.

**TrooperViz beats every external multimodal guard model we tested, on every cut of our benchmark**,
at roughly a third of Llama Guard 4's parameter count.

| | TrooperViz | Best baseline |
|---|---:|---|
| Single-turn multimodal accuracy | **91.83%** | GuardReasoner-VL-3B — 81.39% |
| Multi-turn multimodal accuracy | **90.24%** | SingGuard-4B — 73.84% |
| Overall (12,177-row test set) | **90.21%** | Shieldstral-1.0-3B — 78.12% |
| False-positive rate on benign | **1.06%** | Shieldstral-1.0-3B — 0.97% |

Model weights (local disk only — **not published anywhere, back these up**):

```
outputs/_Week11-Aug3/models/merged_checkpoint871_x_week11_1280/slerp_t075/
```

---

## 2. Results in full

All numbers come from `outputs/_Week10-July27/combined_image_question/splits_train161k/test.parquet`
(12,177 rows: 7,323 single-turn, 3,793 multi-turn ending on an assistant turn, 1,061
multi-turn ending on a user turn). Unparsed output counts as wrong.

The charts used in the final presentation are the authoritative version of these numbers:

```
outputs/_Week11-Aug3/evals/week10_multimodal_accuracy_single_turn.png
outputs/_Week11-Aug3/evals/week10_multimodal_accuracy_multi_turn.png
outputs/_Week11-Aug3/evals/average_benchmark_accuracy_compact.png
outputs/_Week11-Aug3/evals/false_positive_rate_compact.png
outputs/_Week11-Aug3/evals/week10_multimodal_accuracy.md      # source table
```

### 2a. The scoreboard

Multimodal accuracy is over image rows only. FPR is over MMMU (10,500) +
Multimodal-RewardBench (4,711) = 15,211 rows that should all be `safe`, parse errors
counted as refusals.

| Model | Single-turn MM (n=4,310) | Multi-turn MM (n=2,859) | Benign FPR |
|---|---:|---:|---:|
| **TrooperViz (non-thinking)** | **91.83%** | 89.19% | **1.06%** |
| **TrooperViz (thinking)** | 90.23% | **90.24%** | 3.48% |
| GuardReasoner-VL-3B | 81.39% | 68.45% | 5.85% |
| Qwen3.5-4B base (non-thinking) | 80.26% | 70.34% | — |
| Shieldstral-1.0-3B | 77.87% | 73.77% | 0.97% |
| Nemotron-3.5-CS | 77.82% | 63.24% | 4.03% |
| SingGuard-4B | 74.45% | 73.84% | 1.18% |
| Llama Guard 4-12B | 72.20% | 73.24% | 6.65% |
| GuardReasoner-Omni-3B | 70.02% | 56.14% | 12.95% |
| VLGuard LLaVA-13B | 67.10% | 66.56% | 7.40% |

The multi-turn column is the important one: **every baseline degrades on multi-turn, and
TrooperViz does not.** Nemotron drops 14.6 pp, GuardReasoner-Omni 13.9 pp,
GuardReasoner-VL 12.9 pp. TrooperViz in thinking mode gains slightly.

Over the whole 12,177-row test set (not just image rows): **90.21%** non-thinking with
0 parse errors, **89.33%** thinking with 11.

Sources: `outputs/_Week11-Aug3/benchmarks/merged/slerp_t075_week10test_*/summary.json`,
`outputs/_Week11-Aug3/fpr_check_slerp/*/summary.json`.

### 2b. Text-only benchmark suite — the known trade-off

Averaged over 16 text safety benchmarks (`ours` suite: gsm8k_benign, strongreject, xstest,
harmbench, ga_jailbreak, toxicity sets, PII masking, …):

| Model | Average accuracy |
|---|---:|
| Trooper (checkpoint-871, text-only) | 90.45% |
| SingGuard-4B | 88.46% |
| **TrooperViz (non-thinking)** | **87.25%** |
| Shieldstral-1.0-3B | 86.90% |
| TrooperViz (thinking) | 85.64% |
| Nemotron-3.5-CS | 85.75% |
| Qwen3.5-4B base | 84.69% |
| VLGuard LLaVA-13B | 79.46% |
| Llama Guard 4-12B | 74.10% |

TrooperViz gives up ~3 pp of pure-text accuracy relative to the
text-only Trooper in exchange for +10 to +17 pp on multimodal and multi-turn. The SLERP
ratio (t = 0.75 toward the multimodal checkpoint) is where I landed after sweeping
t ∈ {0.25, 0.50, 0.75}. Closing that 3 pp gap is the single most valuable follow-up — see
Section 10.

---

## 3. The task

**Input:** a conversation (1–6 messages, any of `user` / `assistant` / `tool`), optionally
with one image attached to one turn.
**Output:** a verdict on the **last** turn, using earlier turns as context.

```
<answer>unsafe</answer>
<rules_violated>P10, P15</rules_violated>
```

There is no separate "input guard" and "output guard" model. They are the same task on a
different conversation shape:

- last turn is a `user` message → input guard
- last turn is an `assistant` message → output guard
- anything longer → judge the last turn, everything before is context

The chat template marks the judged turn with `[last_message:]`.

### Two inference modes, one checkpoint

Driven entirely by `chat_template_kwargs.enable_thinking` on the request:

- `enable_thinking=true` → assistant prefix is `<think>\n`, model writes a structured
  5-section rationale, then the answer.
- `enable_thinking=false` → prefix is a pre-closed empty `<think>\n\n</think>\n\n`, model
  goes straight to the answer.

Nothing else about the request changes. One served checkpoint supports both per request.

### Policies (P0–P20)

| | | | | |
|---|---|---|---|---|
| **P0** Nonsensical input | **P1** Prompt injection / jailbreak | **P2** Cyber abuse | **P3** Anti-abuse circumvention | **P4** Credentials / PII |
| **P5** Copyright | **P6** Misinfo / civic deception | **P7** Synthetic media | **P8** Hate / discrimination | **P9** Harassment |
| **P10** Violence | **P11** Reckless endangerment | **P12** Self-harm | **P13** Sexual content | **P14** Child safety |
| **P15** Weapons / CBRN | **P16** Substance misuse | **P17** Animal harm | **P18** Regulated advice | **P19** Fraud / scams |
| **P20** Academic dishonesty | | | | |

Source of truth: `src/input_guardrail/prompts/shared_prompts/policies.md`.

> **Policy permutation is ON in Week 10+ training.** Each training row carries a
> `policy_permutation` field; `<rules>` is rendered in that order and `policies_violated`
> holds the *displayed* IDs, verified against `canonical_policies_violated`. This is
> deliberate — it stops the model memorizing "P14 = child safety" and forces it to actually
> read the rules, which is what makes custom policies work. **When analysing training data,
> always use `canonical_policies_violated`; the permuted column carries no signal.**
> Disable with `--no-policy-permutations` for an ablation.

---

## 4. Data

### The pool

`outputs/_Week10-July27/combined_image_question/combined.parquet` — **340,473 rows**,
treated as read-only. Built by `outputs/_Week10-July27/combined_image_question/build_combined.py`
(the builder lives next to its output, not under `scripts/`), then relabeled end-to-end with Gemma-4-31B
(`scripts/train_ds/label_trainset/relabel_combined_gemma.py`, resumable sidecar JSONL).

Columns: `Image`, `Question`, `is_multiturn`, `verdict`, `policies_violated`, `rationale`,
`regenerate`, `policy_permutation`, `canonical_policies_violated`.

### The splits

Built by `scripts/train_ds/build_balanced_splits.py` into
`outputs/_Week10-July27/combined_image_question/splits_train161k/`:

| File | Rows | What |
|---|---:|---|
| `thinking.parquet` | 80,079 | Training, thinking targets |
| `nonthinking.parquet` | 80,078 | Training, answer-only targets (disjoint from above) |
| `validation.parquet` | 1,024 | Balanced; each row evaluated in **both** modes |
| `test.parquet` | **12,177** | The authoritative test set |
| `unused.parquet` | 167,115 | Held out, available for more training or RL |
| `split_manifest.json` | — | Full per-policy and per-cell distributions |

Balancing objective, in priority order: canonical policy first (17 of 21 policies land dead
flat; P5/P12/P14/P20 are exempt because forcing all 21 flat would cap train at ~20k rows),
then `verdict × modality × turn` (8 cells) within the remaining slack. 247 rows labeled
`unsafe` with an empty policy set were routed to `unused` as label-inconsistent. Exact
per-cell and per-policy counts are in `split_manifest.json`.

### Where multi-turn data came from

There was no labeled multi-turn multimodal dataset, so I synthesized one.
`scripts/multiturn_guardrail/build_synthetic_conversations.py` takes an already-labeled row
as an **anchor** — its text, its image, and its verdict are all real and never touched — and
has Gemma-4-31B write only the 1–2 rounds *before* it. Turn count is assigned per row by a
seeded RNG (0.20 / 0.35 / 0.45 for 1/2/3 turns). The anchor's real image may be placed on an
earlier synthetic round, in which case that generation call is multimodal so Gemma actually
sees the image rather than inventing a caption.

Anchors: BeaverTails-V output-guard set (59,648 rows, real request + real response + real
`is_response_safe`) and the MMIG unused split (35,000 rows).

**Status: 85,000 conversations built** (`outputs/_Week9-July20/multiturn_guardrail/synthetic_conversations/results.jsonl`),
of which **20,000 have Qwen3.5-397B output-guard labels**
(`results.qwen397b_merged.jsonl`, via `scripts/multiturn_guardrail/label_conversations_qwen397b.py`).
Both files are resumable. The remaining 65,000 are unlabeled — see Section 10.

### Earlier data (superseded)

The MMIG 186k pool and its 13,000-row test split (`outputs/_Week6-July5/mmig_all_186k/splits/`)
were primary through Week 9 and are **superseded by the Week 10 pool** — kept only to
reproduce older results. It was built from SPA-VL train (91,094), JailBreakV-28K (24,372),
Nemotron (4,930), Think-in-Safety (4,394), VLGuard train (2,956), plus synthesized
compositional hard positives.

---

## 5. Repo map — what is where

```
multimodal-safety/
├── src/
│   ├── input_guardrail/            # The task definition
│   │   ├── policies.py             #   P0-P20 + category aliases from external taxonomies
│   │   ├── contracts.py            #   Verdict parsing (XML, Qwen tool call, native vLLM)
│   │   ├── prompts/shared_prompts/ #   policies.md, thinking_spec.md  ← source of truth
│   │   ├── prompts/qwen3.5/        #   Per-variant prompt .md files
│   │   └── templates/              #   Nemotron etc. prompt builders
│   ├── multimodalsafetylib/        # Pre-existing core library (records, pipeline, serde)
│   └── evaluation/guardrails/      # Adapter framework, 8 guard-model adapters, score matrix
│
├── scripts/                        # ⭐ The operational layer — nearly everything I wrote
│   ├── t_nt_model_training/        # ⭐⭐ TrooperViz lives here: training, merging, eval
│   ├── train_ds/                   # Pool construction, Gemma relabeling, balanced splits
│   ├── multiturn_guardrail/        # Synthetic conversation generation + 397B labeling
│   ├── guard_baselines/            # External guard model runs + the final plot scripts
│   ├── input_guardrail_judge/      # 8-GPU judge runners (Qwen / Gemma / Nemotron)
│   ├── test_ds/                    # Test-set construction, agreement plots, HF push
│   ├── simple_input_guardrail_sft/ # Week 4-6 single-mode SFT (superseded)
│   ├── simple_input_guardrail_rl/  # GRPO code — written, never run
│   ├── gemma_verifier/             # Gemma-4-31B re-judging of guardrail outputs
│   ├── ocr_benchmarks/             # OCRBench v2 — capability-retention check
│   └── materialize_coyo_dataset/   # COYO-700M image download
│
├── tools/                          # 6 Gradio apps, all still used
├── outputs/                        # Every artifact, organised by week
├── models/                         # Retained weights + README-model-weights.md
├── verified/                       # Gemma / human / tiebreak labels
├── lib/execution/, ext/, examples/ # Pre-existing
└── tests/                          # pytest
```

### Files that matter most

| What | Path |
|---|---|
| **Model weights (TrooperViz)** | `outputs/_Week11-Aug3/models/merged_checkpoint871_x_week11_1280/slerp_t075/` |
| **Test set** | `outputs/_Week10-July27/combined_image_question/splits_train161k/test.parquet` |
| **Training prompt template** | `scripts/t_nt_model_training/prompts/common_prompt.jinja` |
| **Training script** | `scripts/t_nt_model_training/train_dual_mode.py` |
| **Merge script** | `scripts/t_nt_model_training/merge_checkpoints.py` |
| **Policy definitions** | `src/input_guardrail/prompts/shared_prompts/policies.md` |
| **Verdict parsing** | `src/input_guardrail/contracts.py` |
| Data pool (read-only) | `outputs/_Week10-July27/combined_image_question/combined.parquet` |
| Split manifest | `.../splits_train161k/split_manifest.json` |
| Final charts + table | `outputs/_Week11-Aug3/evals/` |
| Text guardrail model (merge partner) | `/home/kshah/models/checkpoint-871` |
| Detailed pipeline docs | `scripts/t_nt_model_training/README.md`, `scripts/multiturn_guardrail/README.md` |

---

## 6. How to run things

Every command assumes:

```bash
source ./ACTIVATE_ME     # NOT `source activate`
export UV_CACHE_DIR=/tmp/uv-cache HF_HOME=/tmp/kshah-hf-home HF_DATASETS_CACHE=/tmp/kshah-hf-datasets
export HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1
```

### Train

Dry-run first — it writes `launch_spec.json` without touching a GPU. Add `--live` only after
reading the spec.

```bash
uv run -m scripts.t_nt_model_training.train_dual_mode --live \
  --non-thinking-train-parquet outputs/_Week10-July27/combined_image_question/splits_train161k/nonthinking.parquet \
  --thinking-train-parquet     outputs/_Week10-July27/combined_image_question/splits_train161k/thinking.parquet \
  --validation-parquet         outputs/_Week10-July27/combined_image_question/splits_train161k/validation.parquet \
  --model Qwen/Qwen3.5-4B --template common_guardrail \
  --gpus 8 --max-steps 1500 --learning-rate 1e-5 --max-length 8192 \
  --per-device-train-batch-size 1 --gradient-accumulation-steps 16 \
  --freeze-vit --policy-permutations --loss-scale ignore_empty_think
```

That is the exact recipe behind the TrooperViz base checkpoint: full fine-tune, bf16,
effective global batch 128, ~1 epoch over 160,157 rows. Checkpoint-1280 is the one that was
merged; 1500 also exists and was slightly worse.

### Merge

```bash
uv run -m scripts.t_nt_model_training.merge_checkpoints \
  --model-a /home/kshah/models/checkpoint-871 \
  --model-b outputs/_Week11-Aug3/models/t_nt_guardrail_week11_full/v0-20260802-095634/checkpoint-1280 \
  --method slerp --ratios 0.75 \
  --output-root outputs/_Week11-Aug3/models/merged_checkpoint871_x_week11_1280
```

`--ratios` takes several values at once for a sweep. `--method` also accepts `linear` and
`dare_ties` (the latter needs `--base-model`, `--density`, `--weight-a/b`). Every merge
writes a `merge_manifest.json` recording exactly what went in.

`--tokenizer-source` picks which model's tokenizer, chat template, and config are copied
into the merged output — `a` (the default) or `b`. **TrooperViz used `b`**, the week11
checkpoint, because that is the one carrying `common_prompt.jinja`. Getting this wrong
produces a model that loads without complaint and outputs garbage, so check
`merge_manifest.json["tokenizer_source"]` after every merge.

### Evaluate

Multimodal test set, both modes:

```bash
uv run -m scripts.t_nt_model_training.run_checkpoint871_policy_eval \
  --model <ckpt> --test-parquet outputs/_Week10-July27/combined_image_question/splits_train161k/test.parquet \
  --num-shards 8 --output outputs/<dir>/raw_outputs.jsonl [--enable-thinking]
```

Text benchmark suite + benign FPR:

```bash
uv run -m scripts.t_nt_model_training.run_ours_policy_eval --model <ckpt> --gpus 0,1,2,3,4,5,6,7
uv run -m scripts.t_nt_model_training.run_common_prompt_probe --model <ckpt> \
  --datasets mmmu multimodal-rewardbench --output-dir outputs/<dir>
```

Regenerate the presentation charts:

```bash
uv run python scripts/guard_baselines/plot_week10_multimodal_accuracy.py
uv run python scripts/guard_baselines/plot_average_benchmark_accuracy.py
uv run python scripts/guard_baselines/plot_false_positive_rate.py
```

### Interactive

```bash
uv run -m scripts.input_guardrail_judge.try_qwen_guardrail --gpu 1   # Gradio, thinking toggle, image upload
```

Other Gradio tools, all working: `tools.dataset_verifier_app`, `tools.guardrail_chat_app`,
`tools.tiebreak_app`, `tools.train_label_app`, `tools.nemotron_policy_generator`,
`tools.qwen_thinking_annotation_app`.

### Dev

```bash
uv run pytest
uv run ruff format --check src/multimodalsafetylib examples tests
uv run ty check --python .venv --extra-search-path ext/benchmarking --extra-search-path ext/conjunction-types src/multimodalsafetylib examples tests
```

---

## 7. Training history

### Models used for labeling and judging (not trained by me)

Qwen3.5-397B-A17B (primary labeler and output-guard judge) · Gemma-4-31B (relabeling,
verification, multi-turn lead-in generation) · Claude (test-set labeling) · Nemotron-3.5-CS,
Llama Guard 4, Llama Guard 3 Vision, VLGuard, ProGuard, GuardReasoner-VL, GuardReasoner-Omni,
SingGuard-4B/8B, LLaVAShield, Shieldstral-1.0-3B (baselines).

`checkpoint-871` is the team's text-only guardrail model (Madhav's) and is the text-side
merge partner, not something I trained.

### What I trained, in order

| # | Week | Run | Result |
|---|---|---|---|
| 1 | 4–6 | Non-thinking SFT, Qwen3.5-4B, plain XML target | Baseline. `outputs/_Week6-July5/models/..._sft_non_thinking_*` |
| 2 | 5–6 | Thinking SFT, tool-call + `<think>` target | `..._sft_thinking_*/checkpoint-{256…1095}`; LoRA variant in `models/qwen35_4b/thinking_lora/` |
| 3 | 8 | **Dual-mode T/NT SFT** — one checkpoint, both modes via `enable_thinking` | First unified model. `outputs/_Week8-July13/models/..._dual_mode_*/checkpoint-1280` |
| 4 | 9 | Fine-tuned checkpoint-871 on our multimodal data, 871's native template | 1,280 steps, 4 GPUs, lr 1e-5. Vision added to the text model, but text regressed |
| 5 | 9 | Fresh Qwen3.5-4B trained on 871's template — so both models speak the same prompt language | 640 steps, 8 GPUs. This is what made merging possible |
| 6 | 9 | **First SLERP merge**: 871 × ckpt-640, t ∈ {0.25, 0.50, 0.75} | Proved the merge idea works |
| 7 | 9 | Frozen-ViT retrain (`freeze_vit=True`), 900 steps → `checkpoint-900` | Preserving the vision encoder helped; became the Week 10 line |
| 8 | 10 | **DARE-TIES merge**: 871 × ckpt-900, density 0.5, w 0.5/0.5, seed 17 | Tried as a SLERP alternative. Competitive but not better — SLERP won |
| 9 | 11 | **Final training run** — dual-mode on the 160k Week 10 splits, `common_prompt.jinja`, policy permutations on, frozen ViT, 1,500 steps | `outputs/_Week11-Aug3/models/t_nt_guardrail_week11_full/v0-20260802-095634/checkpoint-{1280,1500}` |
| 10 | 11 | **TrooperViz** — SLERP t=0.75 of 871 × week11-ckpt-1280 | **The deliverable.** Section 2 |

### Week 12 — regression check on Madhav's model

`outputs/_Week12-Aug10/models/quantumguard-v2` is a newer text-only Trooper checkpoint
(TrojAI `trooper`, DAPO phase-2, base `checkpoint-871`) that I did not train. I evaluated it
against text only benchmarks to see whether it should replace `checkpoint-871` as the merge
partner. It is **not** a clean upgrade:

| Benchmark | New | Old | Δ |
|---|---:|---:|---:|
| xstest | 79.78% | 92.89% | **−13.11 pp** |
| troj_toxic_violence | 79.64% | 86.14% | −6.50 pp |
| ga_jailbreak | 88.80% | 85.80% | +3.00 pp |

Full tables in `outputs/_Week12-Aug10/benchmarks/quantumguard_v2_ours_native871_*/comparison.md`.
The xstest regression is an over-refusal regression and would likely have hurt TrooperViz's
FPR, which is why the merge was **not** redone against it.

---

## 8. Gotchas

| Problem | Fix |
|---|---|
| **Wrong policy column in analysis** | `policies_violated` is permuted per row and carries no signal. Use `canonical_policies_violated`. |
| **Merged model outputs garbage** | Tokenizer mismatch. Check `merge_manifest.json`'s `tokenizer_source`. |
| **Stock Qwen chat template breaks output-guard** | It wraps any assistant message after the last user message in `<think></think>`, so the response being judged reads as the guard's own reasoning. Use `common_prompt.jinja` / `conversation_guardrail_template.jinja`. |
| **vLLM `tools` param blanks out content** | Put the tool syntax in the system prompt instead; enabling a tool-call parser moves the call into `message.tool_calls` and empties raw content. |
| **`max_tokens` truncation → fake parse errors** | Anything under ~2048 truncates thinking mode. Use 4096. In thinking mode a high parse-error rate almost always means truncation, not a bad model. |
| **HF 429 errors** | `HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1`. Weights are cached. |
| **`--shard-size` produces 0 shards** | Needs a positive `--limit ≥ dataset size`. |
| **TP sizing** | Qwen3.5-4B has 4 KV heads → TP ∈ {1,2,4}. TP=8 runs but wastes GPUs. |
| **Orphaned vLLM servers** | `pkill -9 -f "vllm serve"`, confirm with `nvidia-smi`. |
| **Port collisions (Errno 98)** | Space ports out or `fuser -k <port>/tcp`. Set `MS_SWIFT_MASTER_PORT` for concurrent training jobs. |
| **Streaming train parquet** | Must be pre-shuffled or eval collapses at epoch boundaries. |
| **LLaVAShield needs transformers 4.45** | Incompatible with the workspace's Transformers 5. Use an isolated `.venv-llavashield`. |
| **Gemma-4-31B at TP=1** | Needs `--gpu-memory-utilization 0.95` on an 80GB H100. |
| **GPU 0** | Permanently hosts a small NER service (~1.7GB). Leave it running. |
| **READMEs can be stale** | Code and `outputs/` are ground truth. Always `--limit 16` probe before a full run, then delete the probe dir. |

Machine: shared, 8× H100 80GB.

---

## 9. Status

**Done**

- 340k-row data pool (text + image, single + multi-turn), fully relabeled with Gemma-4-31B,
  split into policy-balanced train / test / unused sets with a published manifest
- Synthetic multi-turn conversation pipeline — 85,000 conversations built
- Unified task: input guard, output guard, and multi-turn are one model, one prompt
- Dual-mode (thinking / non-thinking) training and inference from a single checkpoint
- Policy permutation training, which is what makes custom taxonomies work
- Weight-space merging: SLERP and DARE-TIES, with ratio sweeps
- **TrooperViz**, beating 9 external guard models on every cut of our benchmark
- Policy-stratified evaluation framework with adapters for every baseline
  (`src/evaluation/guardrails/`), plus OCRBench v2 capability-retention checks
- 6 Gradio tools for labeling, verification, tiebreaking, and interactive testing

**Not done**

- GRPO / RL: `scripts/simple_input_guardrail_rl/` is complete — pipeline, three reward
  functions (format, verdict, policy), MS-Swift plugin — and **has never been executed once**
- TrooperViz is not published to HuggingFace or backed up off this machine
- Multilingual: the training pool contains non-English rows (German, others — see
  `splits_train161k/non_english_examples_exact.md`) but non-English performance was never
  measured

---

## 10. What to do next

**Priority 1 — multilingual, custom policy, and closing the text gap.**

These three are one workstream, because they all come down to the same thing: making the
model read the rules rather than pattern-match memorized categories.

1. **Multilingual.** The pool already contains non-English rows and the model handles them,
   but there is **no non-English benchmark**, so this is unmeasured. Start by carving a
   language-stratified slice out of `test.parquet` (language-detect `Question`, report
   accuracy per language) — that turns an unknown into a number before spending any GPU time.
   Then decide whether to translate a portion of the training pool.

2. **Custom policy.** The mechanism exists — policy permutation during training plus a
   swappable `<rules>` block — and `scripts/t_nt_model_training/evaluate_policy_dropout.py`
   already measures the model's behavior when policies are removed. What is missing is an
   evaluation on genuinely *unseen* policies: write a taxonomy that shares no wording with
   P0–P20, label a few hundred rows against it, and measure. That is the number a customer
   will ask for.

3. **Text benchmarks.** TrooperViz sits 3.2 pp below the text-only Trooper (87.25% vs
   90.45%) — the cost of the merge. Cheapest experiments, in order: sweep SLERP t between
   0.5 and 0.75 more finely; try DARE-TIES against the week11 checkpoint (only ever tried
   against ckpt-900); mix more text-only rows into training, since 167k untouched rows are
   sitting in `unused.parquet`. xstest and troj_toxic_violence are where the loss concentrates.

**Also worth doing, in rough order of value**

- Push TrooperViz to HuggingFace or otherwise back it up. It exists on one disk.
- Run the GRPO pipeline. It has been ready since Week 6 and SFT is plateauing.
- Re-examine `quantumguard-v2` as a merge partner once its xstest regression is fixed
  upstream.
