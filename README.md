# Joint Decomposition of Prompt, Paraphrase, and Item-Level Noise in LLM Benchmark Evaluation

Course project for **ST5230: Applied Natural Language Processing** (NUS, Semester 2, AY2025/2026).

This repository contains the full code, data, and analysis pipeline for an
empirical study that decomposes the noise in LLM benchmark evaluation into
three sources — prompt wording, test-set wording, and item-level instability —
across four instruction-tuned models (7B to 72B) on ARC-Challenge and MMLU-Pro.

## Authors

- Pengyu Li (A0326905B)
- Siyuan Teng (A0310203H)
- Tan Yang (A0333423N)

## Repository structure

```
ST5230/
├── README.md                              this file
├── PROJECT_BRIEFING.md                    full project context and per-experiment summary
├── LITERATURE_REVIEW_AND_GAP_ANALYSIS.md  literature review
├── ST5230_group project.pdf               course requirements (instructor)
├── Project proposal_Group 1.pdf           original proposal
├── references.bib                         bibliography for the report
├── Papers/                                17 reference papers (PDFs)
│
├── exp1/                                  Experiment I: Prompt Perturbation
│   ├── run_experiment1.py                 100-variant runner (two-phase async)
│   ├── prompt_variants.py                 5-dimensional prompt design
│   ├── analyze_experiment1.py             accuracy/variance/OLS analysis
│   ├── visualize_experiment1.py           per-experiment figure generation
│   ├── arc_challenge_300.json             ARC-Challenge 300-question pool
│   ├── mmlu_pro_300.json                  MMLU-Pro 300-question pool
│   ├── results_exp1/                      raw API responses (8 files)
│   └── analysis_exp1/                     processed analysis JSON + summary CSV
│
├── exp2/                                  Experiment II: Paraphrase Resampling
│   ├── generate_paraphrases_gpt4o.py      dual-source paraphrase generator
│   ├── run_experiment2_async.py           paraphrase evaluation runner
│   ├── analyze_experiment2.py             cross-source analysis with BH correction
│   ├── visualize_experiment2.py           per-experiment figure generation
│   ├── *_paraphrased_gpt4o.json           GPT-4o paraphrases (2 files)
│   ├── *_paraphrased_qwen.json            Qwen2.5-72B paraphrases (2 files)
│   ├── exp2_*_gpt4o.json                  evaluation results, GPT-4o source (8 files)
│   ├── exp2_*_qwen.json                   evaluation results, Qwen source (8 files)
│   ├── analysis_exp2/                     processed analysis output
│   ├── EXPERIMENT2_REPORT.md              standalone Exp II writeup
│   └── qc/                                paraphrase quality validation
│       ├── manual_qc_50.py                50-paraphrase manual rubric
│       ├── semantic_faithfulness.py       1,800-paraphrase bidirectional NLI
│       ├── faithfulness_*.json            NLI judgements
│       └── paraphrase_qc_*.csv/.json      manual QC results
│
├── exp3/                                  Experiment III: Noise Item Analysis
│   ├── run_experiment3.py                 per-question noise scoring
│   ├── analyze_experiment3.py             threshold sweep analysis
│   ├── visualize_experiment3.py           per-experiment figure generation
│   ├── noise_data/                        noise scores (shared 150-question subset)
│   └── analysis_exp3/                     threshold-sweep analysis output
│
├── exp4/                                  Experiment IV: Bradley Terry Ranking
│   ├── README.md                          BT-specific notes
│   ├── run_bradley_terry.py               BT MLE + bootstrap + sample-size simulation
│   ├── visualize_bt.py                    BT-specific figure generation
│   └── bt_results_*.json                  BT log-strengths and posteriors
│
├── exp5/                                  Experiment V: Run-to-Run Validation
│   ├── README.md                          stability check overview
│   ├── run_stability.py                   2,000-trial repeated-run runner
│   ├── analyze_stability.py               TARr@5 + run/prompt comparison
│   ├── results_exp5/                      raw repeated-run trials
│   └── analysis_exp5/                     stability analysis output
│
└── figures/                               all figures used in the report
    ├── generate_report_figures.py         main-text figure generator
    ├── generate_appendix_figures.py       appendix figure generator
    ├── fig1_accuracy_distribution.png     main: per-variant accuracy boxplot
    ├── fig2_bt_sample_size.png            main: BT sample-size simulation
    ├── fig3_three_way_variance.png        main: three-way variance decomposition
    ├── fig_a1_ols_coefficients.png        appendix: OLS coefficient heatmap
    ├── fig_a2_dim_variance_per_model.png  appendix: per-model dimension variance
    ├── fig_a3_surface_vs_full.png         appendix: surface vs full comparison
    ├── fig_a4_cross_source.png            appendix: cross-source paraphrase
    ├── fig_a5_noise_correlation.png       appendix: noise correlation heatmap
    ├── fig_a6_category_sensitivity.png    appendix: MMLU-Pro category sensitivity
    ├── fig_a7_bt_posterior.png            appendix: BT forest plot + rank posterior
    ├── fig_a8_threshold_sweep.png         appendix: noise removal threshold sweep
    └── fig_a10_tarr_distribution.png      appendix: TARr@5 distribution
```

## Quick start

### Prerequisites

```bash
pip install httpx numpy pandas matplotlib seaborn scipy
```

You also need an [OpenRouter](https://openrouter.ai) API key set as an
environment variable:

```bash
export OPENROUTER_API_KEY=sk-or-v1-...
```

### Reproducing the figures

All raw API responses needed to reproduce every figure and table are
already in this repository, so you do not need to issue any new API
calls. Run:

```bash
# Main-text figures (3 figures)
python figures/generate_report_figures.py

# Appendix figures (10 figures)
python figures/generate_appendix_figures.py
```

Each script writes its outputs into the `figures/` directory.

### Reproducing each experiment from scratch

If you want to re-run the experiments end-to-end (this will issue
~131,600 API calls and take several hours), follow this order.

```bash
# Experiment I: 100 prompt variants on the shared 150-question subset
python exp1/run_experiment1.py
python exp1/analyze_experiment1.py
python exp1/visualize_experiment1.py

# Experiment II: dual-source paraphrase resampling
python exp2/generate_paraphrases_gpt4o.py     # generates GPT-4o + Qwen paraphrases
python exp2/run_experiment2_async.py          # evaluates 4 models on both sources
python exp2/analyze_experiment2.py
python exp2/visualize_experiment2.py
python exp2/qc/manual_qc_50.py                # 50-paraphrase manual QC
python exp2/qc/semantic_faithfulness.py       # 1,800-paraphrase NLI check

# Experiment III: noise item analysis (no API calls, reanalysis only)
python exp3/run_experiment3.py --shared-only
python exp3/analyze_experiment3.py --noise-tag _shared150
python exp3/visualize_experiment3.py --noise-tag _shared150

# Experiment IV: Bradley Terry ranking (no API calls, reanalysis only)
python exp4/run_bradley_terry.py
python exp4/visualize_bt.py

# Experiment V: run-to-run validation
python exp5/run_stability.py
python exp5/analyze_stability.py

# Final figures for the report
python figures/generate_report_figures.py
python figures/generate_appendix_figures.py
```

## Models evaluated

| Model | Size | API identifier |
|---|---|---|
| LLaMA-3.1-8B-Instruct | 8B  | `meta-llama/llama-3.1-8b-instruct` |
| Qwen2.5-7B-Instruct   | 7B  | `qwen/qwen-2.5-7b-instruct` |
| Qwen3-32B             | 32B | `qwen/qwen3-32b` |
| Qwen2.5-72B-Instruct  | 72B | `qwen/qwen-2.5-72b-instruct` |

All models are accessed through OpenRouter with `temperature=0.0` and
`seed=42`. For Qwen3-32B we append `/no_think` to suppress its internal
chain-of-thought trace.

## Datasets

| Dataset | Source | Sampled |
|---|---|---|
| ARC-Challenge | 1,172 questions, AI2 Challenge test split | 150 (shared subset across all experiments) |
| MMLU-Pro      | 12,032 questions, 14 subject categories   | 150 (stratified by category, shared subset) |

The 150-question shared subset is identical across Experiments I–V to
allow clean variance decomposition without confounding from differing
item samples.

## Key findings

1. **Prompt wording is the dominant noise source**, accounting for
   58–94% of total accuracy variance depending on model and benchmark.
2. **Answer format alone explains over 70%** of prompt-induced variance.
3. **Larger models are not uniformly more robust**: Qwen2.5-72B has the
   highest accuracy on ARC yet the widest prompt-sensitivity range
   (33.3 percentage points).
4. **Reliable top-1 ranking requires ~2,000 pairwise comparisons** under
   a Bradley Terry model, far more than typical leaderboards provide.
5. **Run-to-run noise is consistently below half of prompt-induced
   noise** (median ratio 0.18), confirming that the prompt-sensitivity
   findings are not artifacts of API stochasticity at temperature zero.

See `PROJECT_BRIEFING.md` for the full per-experiment summary.

## Reproducibility notes

- All stochastic operations use **`seed=42`** (prompt sampling, bootstrap
  resampling, BT subsampling, NumPy `RandomState`).
- All raw API responses are committed to this repository (`exp1/results_exp1/`,
  `exp2/exp2_*.json`, `exp5/results_exp5/`), so figures and tables can be
  regenerated without making any new API calls.
- The two-phase pipeline used in Experiment I retries unparseable
  responses with a larger token budget. Final unparseable rate is below
  1% on every model and benchmark.
- Bootstrap CIs use B = 10,000 resamples for all main-text statistics.

## License

Code is released for academic and educational use. The reference papers
in `Papers/` are subject to their respective licenses.
