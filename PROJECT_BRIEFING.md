# Project Briefing: Reliability of LLM Benchmark Evaluation

> This document is a complete briefing for anyone joining the project
> (e.g. to help prepare a PPT presentation). It covers: what the course
> requires, what our research question is, what experiments we ran, what
> we found, and where the key files live.

---

## 1. Course Requirements (ST5230, NUS)

**Theme chosen**: Theme 1 — Reliability of LLM Evaluation

**What the instructor wants**:
- An empirical research study (NOT a software project, NOT leaderboard chasing)
- A clearly stated research question
- A reproducible experimental pipeline
- Statistical analysis beyond reporting a single mean
- Conference-style report (8-10 pages), with:
  Introduction / Experimental Setup / Results / Analysis & Discussion / Conclusion

**Grading weights**:
| Criterion | Weight |
|-----------|--------|
| Problem formulation & clarity | 20% |
| Statistical reasoning & design | 30% |
| Empirical rigor & analysis | 25% |
| Clarity, writing, reproducibility | 15% |
| Insight & originality | 10% |

**Difficulty tiers**:
- Baseline: clear question + reproducible setup + correct metrics
- Core: controlled design + statistical analysis + literature context
- Stretch: failure modes, robustness insights, actionable recommendations

**Team**: 3 members (Pengyu Li, Siyuan Teng, Tan Yang)

---

## 2. Our Research Question

> **To what extent do LLM benchmark evaluation results remain stable under
> prompt perturbations, equivalent test-set resampling, and the presence
> of high-noise items?**

We decompose "evaluation noise" into three sources and measure each:
1. **Prompt wording noise** — same question, different prompt format
2. **Test-set wording noise** — same meaning, paraphrased question stem
3. **Item-level noise** — inherently unstable questions that amplify noise

---

## 3. Models and Datasets

**Models** (4 models across 7B–72B scale):
| Model | Size | API endpoint |
|-------|------|-------------|
| LLaMA-3.1-8B-Instruct | 8B | meta-llama/llama-3.1-8b-instruct |
| Qwen2.5-7B-Instruct | 7B | qwen/qwen-2.5-7b-instruct |
| Qwen3-32B | 32B | qwen/qwen3-32b |
| Qwen2.5-72B-Instruct | 72B | qwen/qwen-2.5-72b-instruct |

**Datasets**:
- ARC-Challenge: 300 questions (science reasoning, 4 options)
- MMLU-Pro: 300 questions (multi-domain knowledge, 10 options A-J)

**Infrastructure**: OpenRouter API, temperature=0.0, seed=42

---

## 4. Experiments Overview

We have **4 experiments + 1 quality validation module**:

```
Exp I  (Prompt Perturbation)     — How much does prompt wording affect scores?
Exp II (Paraphrase Resampling)   — How much does question wording affect scores?
Exp III (Noise Item Analysis)    — Can we identify and remove noisy questions?
Exp IV (Bradley-Terry Ranking)   — How many comparisons to stabilize rankings?
QC     (Paraphrase Validation)   — Are our paraphrases semantically faithful?
```

---

## 5. Experiment I: Prompt Perturbation

**Location**: `exp1/`

**Design**: 18 prompt variants along 4 orthogonal dimensions:
| Dimension | Levels | Examples |
|-----------|--------|---------|
| Instruction phrasing | 3 | "Choose the correct answer" / "Select the best answer" / "Which is correct?" |
| Answer format | 3 | letter-only / "Answer: [X]" / with explanation |
| Option formatting | 3 | A. text / (A) text / A) text |
| Contextual framing | 2 | no prefix / "You are a knowledgeable assistant" |

**Structure**: 1 base + 7 OFAT + 10 factorial = 18 variants
**Scale**: 18 × 300 questions × 4 models × 2 datasets = 43,200 API calls

**Key results (ARC-Challenge)**:
| Model | Mean Acc | Std | Range | Flip Rate |
|-------|----------|-----|-------|-----------|
| LLaMA-3.1-8B | 79.2% | 3.5% | 10.3% | 35.0% |
| Qwen2.5-7B | 86.9% | 5.3% | 20.3% | 41.3% |
| Qwen3-32B | 91.7% | 5.2% | 14.7% | 30.3% |
| Qwen2.5-72B | 89.6% | 11.5% | **33.3%** | 48.7% |

**Key results (MMLU-Pro)**:
| Model | Mean Acc | Std | Range | Flip Rate |
|-------|----------|-----|-------|-----------|
| LLaMA-3.1-8B | 39.9% | 3.8% | 12.4% | 66.0% |
| Qwen2.5-7B | 45.3% | 3.0% | 10.3% | 67.0% |
| Qwen3-32B | 58.2% | 3.3% | 13.1% | 65.7% |
| Qwen2.5-72B | 54.5% | 5.3% | 21.4% | 62.7% |

**Core insights**:
1. `with_explanation` answer format dominates variance (especially for larger models)
   - Qwen2.5-72B ARC range drops from 33.3% → 1.3% when excluding explanation variants
2. MMLU-Pro has ~2× higher flip rates than ARC across all models
3. Bigger models are NOT always more robust (Qwen2.5-72B is most volatile on ARC)

**Analysis methods**:
- OFAT main effects (per-dimension delta from base)
- Main-effects OLS regression with dummy coding (R²_adj = 0.72–0.98)
- Variance decomposition: Var(prompt) vs Var(sampling) via bootstrap
- Dimension-level variance attribution
- MMLU-Pro category-level sensitivity heatmap

**Key figures**: `exp1/figures_exp1/fig1-fig9`

---

## 6. Experiment II: Paraphrase Resampling

**Location**: `exp2/`

**Design**: Dual-source paraphrasing to avoid "model grades its own test" bias
- **GPT-4o** generates 3 paraphrases per question (external model, not evaluated)
- **Qwen2.5-72B** generates 3 paraphrases (as contrast source)
- Only question stem changed; answer choices unchanged
- Fixed base prompt (isolates test-set noise from prompt noise)

**Scale**: 150 questions × 4 versions × 4 models × 2 datasets × 2 sources = 9,600 calls

**Key results (ARC, GPT-4o source)**:
| Model | Mean Acc | Std | Flip Rate |
|-------|----------|-----|-----------|
| LLaMA-3.1-8B | 70.8% | 2.3% | 22.0% |
| Qwen2.5-7B | 86.0% | 1.4% | 14.0% |
| Qwen3-32B | 92.7% | 1.3% | 10.7% |
| Qwen2.5-72B | 95.3% | 0.5% | 4.0% |

**Cross-source agreement**: **100%** on both ARC and MMLU
(all pairwise significance conclusions are identical between GPT-4o and Qwen sources)

**Statistical methods**: Bootstrap CI (10,000 resamples) + Benjamini-Hochberg correction

**MMLU-Pro parse failure issue**: LLaMA and Qwen3-32B give long explanations
instead of letter answers on harder questions. Handled by:
- Primary analysis: full set (parse failures count as wrong)
- Supplementary analysis: clean subset (only questions with no parse failures)

**Key figures**: `exp2/figures_exp2/fig1-fig9`

---

## 7. Experiment III: High-Noise Item Analysis

**Location**: `exp3/`

**Design**: Combine Exp I + Exp II item-level results to build a per-question noise score:

```
Noise(q) = 1 - |2c(q) - N(q)| / N(q)
```
- c(q) = number of conditions answered correctly
- N(q) = total conditions tested
- Noise = 0 when always correct OR always wrong
- Noise = 1 when exactly 50/50

**Key design choice**: **shared-only mode** — only the 150 questions that appear
in BOTH Exp I and Exp II are included, avoiding bias from unequal observation counts.

**Thresholds**: Remove top 10% / 20% / 30% noisiest items, re-evaluate all metrics.

**Key results**:
| Benchmark | Mean Noise | % above 0.5 | Exp1 noise | Exp2 noise |
|-----------|-----------|-------------|-----------|-----------|
| ARC-Challenge | 0.235 | 22.0% | 0.236 | 0.224 |
| MMLU-Pro | 0.448 | 39.3% | 0.456 | 0.253 |

**Core insight**: Prompt noise > paraphrase noise on both benchmarks.
MMLU-Pro is 2× noisier than ARC overall.

**Additional analyses**:
- Three-way variance decomposition: Var(prompt) vs Var(sampling) vs Var(test-set)
- Model noise correlation heatmap (moderate agreement: r ≈ 0.3–0.4)
- Noise vs difficulty relationship
- MMLU-Pro category-level noise analysis
- Ranking reversal reduction under noise removal

**Key figures**: `exp3/figures_exp3_shared150/fig1-fig11`

---

## 8. Experiment IV: Bradley-Terry Ranking Stability

**Location**: `exp4_bt/`

**Design**: Converts item-level correctness from Exp I + Exp II into pairwise
model comparisons, then fits a Bradley-Terry model.

For each (question, condition) cell and model pair (A, B):
- A correct, B wrong → A wins
- B correct, A wrong → B wins
- Ties skipped

**Scale**: ~6,200 condition cells per benchmark → ~37,000 directed outcomes

**Methods**:
1. BT MLE fit via Zermelo iteration
2. 10,000 bootstrap resamples for 95% CIs on log-strengths
3. Rank posterior: Pr(model holds rank k) for k = 1,2,3,4
4. Sample-size simulation: subsample N comparisons, measure top-k stability

**Key results — BT Ranking (identical on both benchmarks)**:
| Rank | Model | ARC logit | MMLU logit |
|------|-------|-----------|-----------|
| 1 | Qwen3-32B | -0.90 | -0.92 |
| 2 | Qwen2.5-72B | -1.16 | -1.11 |
| 3 | Qwen2.5-7B | -1.63 | -1.86 |
| 4 | LLaMA-3.1-8B | -2.47 | -2.17 |

**Sample-size requirements for ≥95% stability**:
| Metric | ARC | MMLU |
|--------|-----|------|
| Top-1 correct | **2,000 comparisons** | **2,000 comparisons** |
| Top-2 set correct | 100 comparisons | 100 comparisons |

**Core insight**: The top-2 set locks in quickly (~100 comparisons), but
distinguishing the #1 from #2 model requires ~2,000 comparisons. Arena-style
leaderboards with <500 comparisons between close competitors are unreliable.

**Key figures**: `exp4_bt/figures_bt/fig1-fig3`

---

## 9. Paraphrase Quality Validation

**Location**: `exp2/qc/`

### 9.1 Manual-Style QC on 50 Samples
- 25 from GPT-4o source + 25 from Qwen source (stratified by dataset)
- GPT-4o as judge, 3-dimension rubric (semantic equivalence, answer invariance, info preservation)
- **Result: 50/50 pass (100%)**, mean scores: 4.98 / 5.00 / 5.00 out of 5

### 9.2 Bidirectional NLI on All 1,800 Paraphrases
- GPT-4o checks both directions: paraphrase→original AND original→paraphrase
- Faithful = both directions return "entailment"
- **Result: 1,739/1,800 (96.6%) faithful**
- 11 genuine meaning-flip contradictions identified
- By source: GPT-4o 95.7%, Qwen 97.6%

---

## 10. How the 4 Experiments Connect (Story Arc)

```
Exp I:  "Prompt wording matters — accuracy swings 10-33% across variants"
   ↓
Exp II: "Question wording also matters, but less — accuracy swings 1-5%"
   ↓
Exp III: "We can identify which questions are most unstable and remove them.
         Removing top 30% noisiest items reduces flip rate by 10-15pp."
   ↓
Exp IV: "Even after pooling all conditions, you need ~2,000 comparisons
         to reliably identify the top-1 model via Bradley-Terry."
```

**The central narrative**:
> Benchmark scores are not as reliable as they appear. Prompt choice is the
> dominant noise source, followed by test-set wording. By identifying and
> removing high-noise items, we can make benchmarks more stable. And even with
> clean data, ranking the top model requires far more comparisons than typical
> leaderboards provide.

---

## 11. Five Key Takeaways for the PPT

1. **Prompt choice dominates evaluation noise**: On ARC, answer format alone
   explains >90% of cross-prompt accuracy variance for larger models.

2. **Bigger models ≠ more robust**: Qwen2.5-72B has the highest accuracy on
   ARC but also the highest prompt sensitivity (range = 33.3%).

3. **MMLU-Pro is fundamentally noisier than ARC**: 2× higher mean noise
   scores, 2× higher flip rates across all models.

4. **Paraphrase source doesn't matter**: 100% cross-source significance
   agreement between GPT-4o and Qwen paraphrases on both benchmarks.

5. **Ranking stability requires ~2,000 comparisons**: The top-2 set is easy
   to identify, but distinguishing #1 from #2 is fragile with <500 samples.

---

## 12. Comparison with Prior Work (Novelty)

| What others did | What we did differently |
|-----------------|----------------------|
| Study prompt sensitivity OR paraphrase OR noise in isolation | **Joint framework** decomposing 3 noise sources |
| Single paraphrase source | **Dual-source** (GPT-4o + Qwen) with cross-source validation |
| Report accuracy changes without statistical testing | **Bootstrap CI + BH correction** on all pairwise comparisons |
| No item-level analysis | **Per-question noise scoring + threshold-based cleaning** |
| No sample-size guidance | **BT model with sample-size simulation** for ranking stability |

---

## 13. File Map

```
ST5230/
├── exp1/                          # Experiment I: Prompt Perturbation (18 variants)
│   ├── prompt_variants.py         # Variant definitions
│   ├── run_experiment1_async.py   # Runner (2-phase)
│   ├── analyze_experiment1.py     # Analysis pipeline
│   ├── visualize_experiment1.py   # 9 figures
│   ├── results_exp1/             # Raw results (JSON per model×dataset)
│   ├── analysis_exp1/            # Analysis outputs
│   └── figures_exp1/             # fig1-fig9 PNG
│
├── exp1_extended/                 # Experiment I Extended (100 variants, running)
│   ├── prompt_variants_100.py     # 100-variant design (5 dimensions)
│   └── run_experiment1_extended.py
│
├── exp2/                          # Experiment II: Paraphrase Resampling
│   ├── generate_paraphrases_gpt4o.py  # Dual-source generation
│   ├── run_experiment2_async.py       # Runner with Phase 2 retry
│   ├── analyze_experiment2.py         # BH correction + cross-source
│   ├── visualize_experiment2.py       # 9 figures
│   ├── *_paraphrased_gpt4o.json      # GPT-4o paraphrases
│   ├── *_paraphrased_qwen.json       # Qwen paraphrases
│   ├── exp2_*_gpt4o.json             # Results (GPT-4o source)
│   ├── exp2_*_qwen.json              # Results (Qwen source)
│   ├── analysis_exp2/                # Analysis outputs
│   ├── figures_exp2/                 # fig1-fig9 PNG
│   └── qc/                           # Quality validation
│       ├── manual_qc_50.py            # 50-sample QC
│       ├── semantic_faithfulness.py   # Bidirectional NLI
│       ├── paraphrase_qc_manual.csv   # QC results
│       └── faithfulness_summary.json  # NLI summary
│
├── exp3/                          # Experiment III: Noise Analysis
│   ├── run_experiment3.py         # Noise score computation
│   ├── analyze_experiment3.py     # Threshold analysis
│   ├── visualize_experiment3.py   # 11 figures
│   ├── noise_data/               # Noise scores (shared150)
│   ├── analysis_exp3/            # Analysis outputs
│   └── figures_exp3_shared150/   # fig1-fig11 PNG
│
├── exp4_bt/                       # Experiment IV: Bradley-Terry
│   ├── run_bradley_terry.py       # BT fit + bootstrap + simulation
│   ├── visualize_bt.py            # 3 figures
│   ├── bt_results_*.json          # BT ratings + rank posteriors
│   └── figures_bt/               # fig1-fig3 PNG
│
├── Papers/                        # Reference literature (17 papers)
├── DETAILED_PROJECT_REPORT.md     # Internal debug report (Chinese)
├── LITERATURE_REVIEW_AND_GAP_ANALYSIS.md
├── PROJECT_BRIEFING.md            # ← THIS FILE
└── ST5230_group project.pdf       # Course requirements
```

---

## 14. Status of Ongoing Work

| Item | Status |
|------|--------|
| Exp I (18 variants) | ✅ Complete |
| Exp I Extended (100 variants) | 🔄 Running (~80% done, LLaMA/ARC) |
| Exp II (dual-source paraphrases) | ✅ Complete |
| Exp III (noise analysis, shared150) | ✅ Complete |
| Exp IV (Bradley-Terry) | ✅ Complete |
| Paraphrase QC (50 manual + 1800 NLI) | ✅ Complete |
| English conference-style report | ❌ Not yet written |
| PPT presentation | 🔄 In progress |
