# ToW Reproducibility Report

This repository contains the code and notebooks for a scaled reproducibility study of **Thoughts of Words (ToW)** and a SciCite citation-intent extension.

The reproduced paper is:

> Xu et al. (2025), *ToW: Thoughts of Words Improve Reasoning in Large Language Models.*

The project has two parts:

1. **Scaled reproduction** of the original Raw vs. ToW continual-training comparison using the authors' public `fine-nwp` codebase.
2. **SciCite extension** testing whether ToW-style annotations transfer to scientific citation-intent classification.

The submitted report uses the following final terminology:

| Name | Abbreviation | Description |
|---|---|---|
| Citation-Cue ToW | CC-ToW | Task-oriented citation cue explanations appended to classifier input. |
| Inline Word-Level ToW | IW-ToW | Clean word-level ToW inserted inline before selected target words. |
| ToW-MLM Adaptation | ToW-MLM | Clean IW-ToW text used only for MLM adaptation before raw SciCite classification. |

---

## Repository Structure

```text
tow-reproducibility-report/
├── README.md
├── notebooks/
│   ├── 01_tow_reproduction_qwen25_3b_raw_vs_tow.ipynb
│   ├── 02_scicite_citation_cue_tow_annotation.ipynb
│   ├── 03_scicite_inline_wordlevel_tow_generator.ipynb
│   ├── 04_scicite_direct_input_tow_extensions_CC_IW.ipynb
│   └── 05_scicite_tow_mlm_adaptation.ipynb
├── results/
│   ├── reproduction_summary.csv
│   ├── scicite_extension_AB_results.csv
│   └── scicite_extension_B2_results.csv
└── figures/
    ├── scicite_extension_AB_scores.pdf
    └── scicite_extension_B2_scores.pdf
```

The file names `scicite_extension_AB_results.csv` and `scicite_extension_B2_results.csv` come from the experiment scripts. In the report, these correspond to:

- `scicite_extension_AB_results.csv`: direct-input Raw vs. CC-ToW vs. IW-ToW results.
- `scicite_extension_B2_results.csv`: ToW-MLM Adaptation results.

---

## Environment

The experiments were run in Google Colab Pro.

### Reproduction hardware

- GPU: NVIDIA A100-SXM4 80GB
- System RAM: high-RAM runtime, approximately 167GB
- Precision: bf16
- Training backend: DeepSpeed ZeRO-3
- Evaluation backend: vLLM

### Extension hardware

The SciCite extension experiments are much lighter and can run on a Colab GPU runtime. The direct-input SciBERT experiments train quickly. The ToW-MLM adaptation notebook is also small because it uses the cleaned SciCite subset.

---

## Required Secrets

Only the annotation-generation notebooks require an OpenAI API key.

In Colab, add this secret:

```text
OPENAI_API_KEY
```

Notebooks that require the key:

```text
02_scicite_citation_cue_tow_annotation.ipynb
03_scicite_inline_wordlevel_tow_generator.ipynb
```

Evaluation notebooks can be run without calling the API if the cached JSONL annotation files are already present.

Notebooks that do **not** require the API when cached data exists:

```text
04_scicite_direct_input_tow_extensions_CC_IW.ipynb
05_scicite_tow_mlm_adaptation.ipynb
```

---

## How to Run

### 1. Scaled Raw vs. ToW reproduction

Run:

```text
notebooks/01_tow_reproduction_qwen25_3b_raw_vs_tow.ipynb
```

This notebook:

1. Clones the official `ARC-ASU/fine-nwp` repository.
2. Applies Colab compatibility patches.
3. Uses the released `Raw.jsonl` and `ToW.jsonl` pretraining files.
4. Trains Qwen2.5-3B under Raw and ToW conditions.
5. Evaluates on ARC-Challenge and CommonsenseQA.
6. Saves a reproduction summary CSV/JSON.

The report run used:

```text
Model: Qwen/Qwen2.5-3B
Training steps: 30
Max sequence length: 768
Benchmarks: ARC-Challenge, CommonsenseQA
Evaluation examples: 50 per benchmark
```

Final reproduction results:

| Model | Condition | Benchmark | Exact Match |
|---|---|---|---:|
| Qwen2.5-3B | Raw | ARC-Challenge | 0.70 |
| Qwen2.5-3B | ToW | ARC-Challenge | 0.70 |
| Qwen2.5-3B | Raw | CommonsenseQA | 0.66 |
| Qwen2.5-3B | ToW | CommonsenseQA | 0.74 |

---

### 2. Generate Citation-Cue ToW annotations

Run:

```text
notebooks/02_scicite_citation_cue_tow_annotation.ipynb
```

This notebook creates the **CC-ToW** dataset. It loads SciCite examples and uses a GPT model to produce citation-function cue explanations.

It saves cached JSONL files such as:

```text
scicite_train_tow.jsonl
scicite_validation_tow.jsonl
scicite_test_tow.jsonl
```

These files are used by the direct-input extension notebook.

This notebook requires `OPENAI_API_KEY`.

---

### 3. Generate clean Inline Word-Level ToW annotations

Run:

```text
notebooks/03_scicite_inline_wordlevel_tow_generator.ipynb
```

This notebook creates the final clean **IW-ToW** dataset used in the report. It selects meaningful target words in SciCite contexts and rejects poor targets such as:

- author names
- years and numbers
- `et` / `al`
- short words and stopwords
- words inside parenthetical citations
- citation-format artifacts

It then generates word-level ToW annotations and saves cached JSONL files such as:

```text
scicite_train_wordlevel_tow.jsonl
scicite_validation_wordlevel_tow.jsonl
scicite_test_wordlevel_tow.jsonl
```

This notebook requires `OPENAI_API_KEY`.

---

### 4. Evaluate direct-input SciCite extensions

Run:

```text
notebooks/04_scicite_direct_input_tow_extensions_CC_IW.ipynb
```

This notebook evaluates:

1. Raw SciBERT
2. CC-ToW as an appended classifier input feature
3. IW-ToW inserted directly into the citation context

It uses common IDs across all three conditions so the comparison is fair.

Final direct-input extension results:

| Condition | Accuracy | Macro F1 |
|---|---:|---:|
| Raw SciBERT | 0.8333 | 0.8296 |
| CC-ToW | 0.8788 | 0.8758 |
| IW-ToW | 0.8788 | 0.8793 |

Per-class F1:

| Condition | Background | Method | Result |
|---|---:|---:|---:|
| Raw SciBERT | 0.7895 | 0.8421 | 0.8571 |
| CC-ToW | 0.8500 | 0.8718 | 0.9057 |
| IW-ToW | 0.8421 | 0.9231 | 0.8727 |

Outputs:

```text
results/scicite_extension_AB_results.csv
figures/scicite_extension_AB_scores.pdf
```

---

### 5. Evaluate ToW-MLM Adaptation

Run:

```text
notebooks/05_scicite_tow_mlm_adaptation.ipynb
```

This notebook evaluates the most faithful training-time extension.

The workflow is:

1. Load clean IW-ToW text.
2. Continue MLM adaptation of SciBERT on clean IW-ToW text.
3. Fine-tune the adapted SciBERT model on raw SciCite citation contexts.
4. Evaluate on raw SciCite citation contexts.

The classifier does **not** see `<ToW>` tags at test time.

Final ToW-MLM results:

| Condition | Accuracy | Macro F1 |
|---|---:|---:|
| Raw SciBERT | 0.8636 | 0.8609 |
| ToW-MLM Adaptation | 0.8788 | 0.8789 |

Outputs:

```text
results/scicite_extension_B2_results.csv
figures/scicite_extension_B2_scores.pdf
```

---

## Quick No-API Rerun

To reproduce the reported SciCite evaluation without regenerating annotations:

1. Ensure the cached CC-ToW and IW-ToW JSONL files are available in the expected Google Drive folders.
2. Run notebook 04 for direct-input Raw/CC-ToW/IW-ToW evaluation.
3. Run notebook 05 for ToW-MLM Adaptation.

This avoids any OpenAI API calls.

---

## Notes on Reproducibility

The original ToW paper used larger models, more training steps, and more benchmarks. This repository is a scaled reproduction designed for a single Colab A100 runtime. Therefore, the reproduction results should be interpreted as a constrained pipeline reproduction, not a full replication of every number in the original paper.

The SciCite extension uses a small common-ID test set. The final common-ID direct-input experiment used:

```text
Train: 187
Validation: 67
Test: 66
```

Because the test set is small, the results should be interpreted as pilot evidence rather than a definitive benchmark result.

---

## Code Relationship to the Original Repository

The reproduction notebook uses the authors' official `fine-nwp` repository as the base implementation. The notebook adds Colab-specific setup, dependency pins, compatibility patches, scaled settings, and result export logic.

The SciCite extension notebooks are standalone additions written for this project. They generate and evaluate new ToW-style annotations for citation-intent classification.

---

## Citation

If referring to the original method, cite Xu et al. (2025):

```bibtex
@inproceedings{xu2025tow,
  title     = {ToW: Thoughts of Words Improve Reasoning in Large Language Models},
  author    = {Xu, Zhikun and Shen, Ming and Dineen, Jacob and Li, Zhaonan and Ye, Xiao and Lu, Shijie and RRV, Aswin and Baral, Chitta and Zhou, Ben},
  booktitle = {Proceedings of the 2025 Conference of the North American Chapter of the Association for Computational Linguistics},
  pages     = {3057--3075},
  year      = {2025}
}
```

