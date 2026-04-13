# 研究背景、研究目的与文献差距分析

> 基于 Papers/ 文件夹中 17 篇文献，结合项目当前实验设计（Exp1–Exp3）撰写。

---

## 一、研究背景（Research Background）

### 1.1 LLM 评测的核心地位与潜在危机

大语言模型（LLMs）的快速发展使得基准评测（benchmark evaluation）成为衡量模型能力、指导研发方向和部署决策的关键手段 (Eleuther AI lm-eval, 2405.14782)。然而，越来越多的研究表明，**当前基准评测体系的可靠性远低于普遍预期**，已发表的性能差异中有相当比例可能来源于评测过程本身的噪声，而非模型真实能力的差异。
 
### 1.2 评测噪声的多重来源

近年文献已从多个角度揭示了评测噪声的严重性：

**（1）Prompt 敏感性（Prompt Sensitivity）**

- Zhu et al. (2306.04528, PromptRobust) 在 8 个任务、13 个数据集上系统测试了字符、词、句子和语义四个层级的 prompt 扰动，发现 LLM 对提示词的微小变化高度敏感，即使语义不变的同义替换也可导致准确率大幅波动。
- Cao et al. (2406.10248, RobustAlpacaEval) 进一步引入"最差提示性能"的概念，发现 Llama-2-70B-chat 最优与最差提示之间的准确率差距高达 **45.48%**，最差性能仅为 9.38%，且现有 prompt engineering 方法对改善最差表现效果有限。
- Razavi et al. (2502.06065, PromptSET) 提出了"Prompt Sensitivity Prediction"任务，发现当前方法（包括 LLM 自评估、文本分类和查询性能预测）均难以有效预测哪些提示变体会导致性能退化。
- Brittlebench (2603.13285) 对 LLM 的鲁棒性进行了量化，发现模型对选项顺序、上下文可用性等都十分敏感。
- 医学领域研究 (2603.25960) 发现 Chain-of-Thought 提示反而降低 MedGemma 准确率 5.7%，few-shot 样例使性能下降 11.9%，打乱答案选项顺序导致 **59.1%** 的预测发生改变，说明通用模型上验证过的 prompt 技巧在领域模型上并不可迁移。

**（2）测试集表述噪声（Test-set Wording Noise）**

- Mousavi et al. (2506.23864, "Garbage In, Reasoning Out?") 对 SocialIQa、FauxPas-EAI、ToMi 三个推理基准进行系统审计，发现**大量结构性、语义性和语用性缺陷**（如重复题目、模糊措辞、不合理选项），模型分数提升往往源于表面表述变化而非推理能力改善。
- Platinum Benchmarks (2502.03461) 在 15 个流行基准上发现广泛的标签错误（label errors），这些错误掩盖了模型的真实失败模式。即使是前沿 LLM，在经过仔细校正后的"铂金基准"上仍然在小学数学题等简单任务上犯错。

**（3）基准饱和与设计缺陷（Benchmark Saturation）**

- Akhtar et al. (2602.16763) 分析了 60 个 LLM 基准，发现**近半数已呈现饱和**，无法区分顶级模型之间的差异。专家策划的基准比众包基准更耐饱和，而隐藏测试数据（公开 vs 私有）对抗饱和无保护效果。
- PSN-IRT (2505.15055) 提出基于项目反应理论（IRT）的增强框架，对 11 个基准共 41,871 道题进行分析，揭示了现有基准在测量质量上的**显著且多样的不足**。

**（4）数据污染（Data Contamination）**

- 多篇综述与方法论论文 (2406.04244, 2502.00678) 指出，预训练数据中的基准泄露（Benchmark Data Contamination）是影响评测公平性的重要因素。Choi et al. 提出的 Kernel Divergence Score (KDS) 通过比较微调前后嵌入矩阵的散度来量化污染程度，与污染水平呈近乎完美的相关。

### 1.3 评测不稳定性对模型排名的影响

**（1）确定性幻觉（Illusion of Determinism）**

- Atil et al. (2408.04667, "LLM Stability") 发现即使在 temperature=0 的确定性配置下，6 个 LLM 在 5 次相同运行中准确率波动可达 **10%**，且没有任何 LLM 能在所有任务上稳定复现。他们提出了 TARr@N 和 TARa@N 等稳定性度量指标，建议将其纳入排行榜。
- Blackwell et al. (2410.03492) 同样关注 LLM 的随机性，提出了一种成本效率较高的不确定性量化方法，建议进行重复实验以报告置信区间而非单一分数。

**（2）排名脆弱性（Ranking Fragility）**

- Huang & Shen (2508.11847) 证明在 Chatbot Arena 和 MT-Bench 上，仅移除 **0.02%** 的评测数据即可改变排名第一的模型。MT-Bench 因使用专家标注更鲁棒，而众包人类评估与 LLM-as-Judge 同样脆弱。
- Language Model Council (2406.08598) 提出用 20 个 LLM 互相评估的民主协作框架，发现比单一 LLM 评委（如 GPT-4o）产生的排名更稳定、更一致、更接近人工评估。

**（3）评测框架层面的可靠性保障**

- lm-eval harness (2405.14782) 总结了三年评测经验，系统梳理了评测设置敏感性、跨方法比较困难和可复现性缺失等常见挑战。
- MaP 框架 (2510.09295) 从预训练动态评估角度出发，将不稳定性归因于**参数不稳定性**（训练随机性）和**评测不稳定性**（噪声测量协议），提出 checkpoint merging + Pass@k 的双管齐下策略。

### 1.4 文献总结

综合以上文献，当前 LLM 评测面临的核心挑战可归纳为：

| 噪声维度 | 代表文献 | 核心发现 |
|---|---|---|
| Prompt 扰动 | PromptRobust, RobustAlpacaEval, PromptSET, Brittlebench | 同一题目不同提示间准确率差距可达 45%+ |
| 测试集措辞 | "Garbage In, Reasoning Out?", Platinum Benchmarks | 基准题目本身存在广泛缺陷和歧义 |
| 运行随机性 | LLM Stability, Blackwell et al. | Temperature=0 下仍有 10% 波动 |
| 排名脆弱性 | Huang & Shen, Language Model Council | 移除 0.02% 数据即可翻转排名 |
| 数据污染 | BDC Survey, KDS | 预训练泄露使分数虚高 |
| 基准饱和 | Akhtar et al., PSN-IRT | 近半数基准已无法区分顶级模型 |

**然而，现有文献大多聚焦于某一单一维度的噪声**（仅研究 prompt 或仅研究 test-set），**缺乏将多维噪声统一量化、并在题目层面构建综合噪声图谱的系统性框架**。这正是本项目的切入点。

---

## 二、研究目的（Research Objectives）

基于以上文献背景，本项目的核心研究问题为：

> **LLM 基准评测中观测到的性能差异，有多少反映了模型的真实能力差异，又有多少源于评测过程本身引入的噪声？**

围绕这一问题，项目设定三个递进的具体目标：

### 目标 1：量化 Prompt Wording Noise（Exp1）

在固定测试集内容的前提下，系统变化 prompt 的指令措辞、输出格式、选项标记和角色设定，测量同一模型在同一题目上因 prompt 变化导致的**准确率波动幅度**和**题目级别翻转率（item flip rate）**。

> **对应文献**：PromptRobust (2306.04528)、RobustAlpacaEval (2406.10248)、PromptSET (2502.06065)
> **差异化贡献**：文献多采用对抗性扰动或自然语言改写，本项目采用结构化的多维度 prompt 变体设计（instruction × format × option × framing），更适合做方差分解分析。

### 目标 2：量化 Test-set Wording Noise（Exp2）

在固定 prompt 的前提下，对题干进行语义保持（meaning-preserving）的改写（paraphrase），测量改写后模型答案的**稳定性**，并通过双来源（GPT-4o vs Qwen）对照降低 paraphrase 生成偏差。

> **对应文献**："Garbage In, Reasoning Out?" (2506.23864)、Platinum Benchmarks (2502.03461)
> **差异化贡献**：文献侧重于识别和修复基准缺陷，本项目则主动生成等价重写来测试模型对"合理的措辞变化"的敏感度。

### 目标 3：构建题目级噪声图谱并评估去噪效果（Exp3）

将 Exp1 和 Exp2 的题目级波动合并为综合 noise score，识别高噪声题目，评估过滤这些题目后基准评测的**稳定性提升**和**排名一致性变化**。

> **对应文献**：PSN-IRT (2505.15055)、LLM Stability (2408.04667)、MaP (2510.09295)
> **差异化贡献**：PSN-IRT 基于 IRT 建模题目参数，MaP 从训练动态角度处理评测噪声。本项目采用更直接的经验性方法——基于多条件观测的翻转率构造噪声分数，计算简单、可解释性强，且同时涵盖 prompt 和 test-set 两个维度。

---

## 三、项目实现方式相比文献存在的不足

基于对 17 篇文献的深入分析和项目当前实验设计的对比，以下从方法论、实验规模、统计严谨性和评估完整性四个方面逐条分析不足之处。

### 3.1 Prompt 扰动设计的系统性不足

| 方面 | 文献标准 | 本项目现状 | 差距 |
|---|---|---|---|
| 变体数量 | PromptRobust: 4,788 个对抗性 prompt；RobustAlpacaEval: 语义等价的 case-level 变体；课程要求建议 50–200 个 | **18 个变体**（1 base + 7 OFAT + 10 随机组合） | 数量偏少，可能不足以充分覆盖 prompt 空间 |
| 设计结构 | PromptRobust 按字符/词/句/语义四层级系统扰动 | 4 维度（instruction, format, option, framing）但**非正交设计** | 主效应与交互项可能混杂，无法做严格的因子分解 |
| 扰动层级 | 文献涵盖字符级（typo）、词级（同义替换）、句级（重排）、语义级（改写） | 仅涉及句级和语义级（指令措辞变化） | 缺少字符级和词级扰动，如 typo、标点变化等 |

**建议**：
- 增加变体数量至 50+ 以满足课程要求，或至少补充说明 18 个变体在统计功效（statistical power）上的合理性。
- 考虑引入正交或近正交的拉丁方设计（Latin Square），使得可以分离各维度的主效应。
- 若资源充裕，增加字符级（如 "Choose" → "Chooze"）和词级（如 "correct" → "right"）扰动。

### 3.2 Paraphrase 语义保真验证缺失

| 方面 | 文献标准 | 本项目现状 | 差距 |
|---|---|---|---|
| 语义等价验证 | RobustAlpacaEval 使用 GPT-4 自动验证语义等价性；Platinum Benchmarks 使用人工标注确认标签正确性 | **无独立验证**——仅依赖生成 prompt 的指令约束 | 关键环节缺失：部分准确率波动可能源于 paraphrase 的 semantic drift 而非模型脆弱性 |
| 人工抽检 | "Garbage In, Reasoning Out?" 使用系统性人工标注验证基准缺陷 | 无人工抽检 | 缺乏 ground truth 校准 |
| 答案不变性检查 | Platinum Benchmarks 一题一题验证正确答案是否因改写而改变 | 未检查改写后正确答案是否仍然成立 | 可能引入假阳性噪声 |
| Paraphrase 多样性 | VarBench (文献引用) 使用变量扰动确保语义严格等价 | 有多样性统计，但无等价性打分 | 知道 paraphrase 们"不同"，但不知道它们是否"等价" |

**建议**：
- 增加一个轻量级的语义等价验证流水线：用 GPT-4o 或人工对随机抽取的 50–100 对（原题, paraphrase）打分（如 1-5 分等价度），报告 inter-rater agreement。
- 对高噪声题目的 paraphrase 进行人工审查，排除 semantic drift 导致的假阳性。
- 增加"答案不变性验证"：检查每个 paraphrase 版本的标准答案是否仍然是原题的正确答案。

### 3.3 运行随机性未被量化

| 方面 | 文献标准 | 本项目现状 | 差距 |
|---|---|---|---|
| 重复运行 | Atil et al. (LLM Stability): 每个配置**重复 5 次**；Blackwell et al.: 多次重复实验 | **每个条件仅运行 1 次** | 无法区分"确定性 API 返回不同结果"与"prompt/paraphrase 引起的真实差异" |
| 稳定性指标 | TARr@N (total agreement rate at N runs)、TARa@N | 未使用 | 缺乏稳定性基线 |
| temperature=0 验证 | LLM Stability 明确验证 temperature=0 并不保证确定性 | 假设 temperature=0 即确定性 | 可能低估了 sampling variance |

**建议**：
- 对至少一个代表性子集（如 50 题 × 1 prompt × 2 模型）重复运行 3–5 次，测量 temperature=0 下的实际波动。
- 如果波动可忽略，则可在论文中明确报告这一验证结果作为假设支撑；如果不可忽略，则需将 sampling variance 纳入方差分解。

### 3.4 实验规模与模型覆盖

| 方面 | 文献标准 | 本项目现状 | 差距 |
|---|---|---|---|
| 数据集规模 | PromptRobust: 8 任务 13 数据集；PSN-IRT: 11 基准 41,871 题；Benchmark Saturation: 60 基准 | **2 个数据集，每个 300 题（Exp2 仅 150 题）** | 规模有限，结论的外部效度受限 |
| 模型数量 | LLM Stability: 6 模型；Language Model Council: 20 模型；PromptRobust: 13 模型 | **4 个模型**（同系列为主：3 个 Qwen + 1 个 LLaMA） | 模型多样性不足，Qwen 系列占比过高可能引入系列偏差 |
| 模型家族多样性 | 文献通常覆盖 GPT、Claude、LLaMA、Mistral 等多家族 | Qwen（3 个）+ LLaMA（1 个） | 缺少 GPT、Mistral、Gemma 等代表性家族 |

**建议**：
- 在论文中明确声明实验规模为课程项目约束下的合理选择，并讨论外部效度的局限性。
- 如有预算，补充 1–2 个非 Qwen 家族模型（如 Mistral-7B 或 Gemma-2）以增加多样性。
- 考虑在 Exp2 中将 150 题扩展至与 Exp1 一致的 300 题（项目报告中已提及此计划）。

### 3.5 统计分析方法的严谨性

| 方面 | 文献标准 | 本项目现状 | 差距 |
|---|---|---|---|
| 方差分解模型 | PSN-IRT 使用项目反应理论参数化模型；MaP 使用 checkpoint merging + Pass@k | 简单的三源方差分解（prompt / sampling / test-set） | 缺乏更精细的统计建模 |
| 不确定性量化 | Blackwell et al. 报告预测区间（prediction interval）；LLM Stability 使用多次运行的统计检验 | 仅报告 mean、std、range | 缺少置信区间、bootstrap 区间或贝叶斯不确定性 |
| 多重比较校正 | 标准统计实践要求 Bonferroni/FDR 校正 | 报告中提到"显著性校正标准未统一" | 可能存在假阳性膨胀 |
| 效应量报告 | 文献普遍报告 Cohen's d、Cramér's V 等 | 仅报告 flip rate 和百分比差异 | 不便跨研究对比 |

**建议**：
- 为方差分解增加 bootstrap 置信区间。
- 报告标准化效应量（如 Cohen's d），使结果可与文献比较。
- 对多个模型 × 数据集的统计检验应用 FDR 校正。

### 3.6 数据污染与基准饱和未被考虑

| 方面 | 文献标准 | 本项目现状 | 差距 |
|---|---|---|---|
| 数据污染检测 | KDS (2502.00678)、BDC Survey (2406.04244) 提供系统性检测方法 | **完全未涉及** | 如果参评模型的预训练数据中包含 ARC/MMLU 题目，则 Exp1/Exp2 的"正确率"本身就不可靠 |
| 基准饱和考量 | Akhtar et al. 分析基准饱和因素 | 未讨论 | ARC-Challenge 上 Qwen2.5-72B 已达 95%+，接近饱和，在此区间测量噪声的意义可能受限 |

**建议**：
- 在论文的 Limitation 部分明确讨论数据污染的潜在影响：ARC 和 MMLU 作为广泛使用的基准，很可能已在多数模型的预训练数据中出现。
- 引用 KDS 等方法说明这一风险，即使不实际运行污染检测，也应作为 acknowledged limitation。
- 针对 ARC 上 Qwen2.5-72B 的接近饱和表现，讨论天花板效应（ceiling effect）对噪声测量的影响。

### 3.7 Noise Score 设计的理论基础

| 方面 | 文献标准 | 本项目现状 | 差距 |
|---|---|---|---|
| 题目质量建模 | PSN-IRT: 基于 IRT 的区分度（discrimination）、难度（difficulty）参数化 | 简单公式：$Noise(q) = 1 - \|2c(q) - N(q)\| / N(q)$ | 只衡量稳定性，不区分题目难度、区分度等多维属性 |
| "噪声"与"难度"的区分 | IRT 框架天然区分"难题"（低正确率但稳定）与"noisy 题"（不稳定） | 项目报告已指出但未在方法上解决 | 困难但稳定的题（全错）和简单且稳定的题（全对）都被标为 noise=0，中等难度题天然更"noisy" |

**建议**：
- 在论文中明确定义 noise score 度量的是"稳定性"而非"质量"，并讨论这一设计选择的利弊。
- 考虑引入 IRT 框架或至少报告 point-biserial correlation 作为补充的题目质量指标。
- 分析 noise score 与题目难度之间的关系（是否存在"中难度题天然更 noisy"的混淆）。

### 3.8 缺乏与排名稳定性的直接连接

| 方面 | 文献标准 | 本项目现状 | 差距 |
|---|---|---|---|
| 排名稳定性分析 | Huang & Shen: 最小数据移除导致排名翻转；Language Model Council: 鲁棒排名 | Exp3 有去噪后的结果比较，但**未显式量化排名翻转** | 缺少"噪声导致排名翻转"的直接证据 |
| 排名模型 | Bradley-Terry 模型、Elo 评分 | 未使用 | 无法与 Chatbot Arena 等排名体系做对比 |

**建议**：
- 在 Exp3 中增加排名翻转分析：在不同噪声条件下（不同 prompt、不同 paraphrase），模型排名发生翻转的频率。
- 可参照 Huang & Shen 的方法，计算最少需要移除多少题才能在当前 4 个模型间引发排名翻转。

---

## 四、总结：项目定位与改进优先级

### 项目的核心优势（相比文献）

1. **多维噪声统一框架**：文献大多聚焦单一噪声维度，本项目首次将 prompt noise 和 test-set wording noise 合并为题目级噪声图谱。
2. **双来源 paraphrase 对照**：引入 GPT-4o 和 Qwen 双源生成，部分缓解了 source bias。
3. **递进式实验结构**：Exp1 → Exp2 → Exp3 逻辑清晰，每一步都建立在前一步的基础上。
4. **完善的工程实践**：parse failure 两阶段处理、shared-only 模式、full-set 主分析等设计体现了严谨的工程思维。

### 改进优先级排序

| 优先级 | 改进点 | 预计工作量 | 影响程度 |
|---|---|---|---|
| 🔴 高 | 增加 paraphrase 语义保真验证 | 中 | 直接影响 Exp2 结论可信度 |
| 🔴 高 | 增加 temperature=0 重复运行验证 | 低 | 补充噪声分解基线 |
| 🟡 中 | 增加 bootstrap 置信区间 | 低 | 提升统计严谨性 |
| 🟡 中 | 补充排名翻转分析 | 中 | 增强与排名稳定性文献的对话 |
| 🟡 中 | 讨论数据污染风险（Limitation 章节） | 低 | 完善论文的自我批评 |
| 🟢 低 | 扩大 prompt 变体至 50+ | 高 | 满足课程建议 |
| 🟢 低 | 增加非 Qwen 家族模型 | 高 | 提高外部效度 |
| 🟢 低 | 引入 IRT 框架补充 noise score | 中 | 增强理论基础 |

---

## 文献引用索引

| 编号 | 类别 | 标题 | ArXiv ID |
|---|---|---|---|
| 1 | Evaluation Noise | Benchmark Data Contamination of Large Language Models: A Survey | 2406.04244 |
| 2 | Evaluation Noise | How Contaminated Is Your Benchmark? Quantifying Dataset Leakage with KDS | 2502.00678 |
| 3 | Evaluation Noise | Garbage In, Reasoning Out? Why Benchmark Scores are Unreliable | 2506.23864 |
| 4 | Evaluation Noise | When AI Benchmarks Plateau: A Systematic Study of Benchmark Saturation | 2602.16763 |
| 5 | Ranking Stability | Language Model Council: Democratically Benchmarking Foundation Models | 2406.08598 |
| 6 | Ranking Stability | LLM Stability: A Detailed Analysis with Some Surprises | 2408.04667 |
| 7 | Ranking Stability | PSN-IRT: Rethinking LLM Benchmarking with Item Response Theory | 2505.15055 |
| 8 | Ranking Stability | Dropping Just a Handful of Preferences Can Change Top LLM Rankings | 2508.11847 |
| 9 | Benchmark Reliability | lm-eval: Reproducible Evaluation of Language Models | 2405.14782 |
| 10 | Benchmark Reliability | Towards Reproducible LLM Evaluation: Quantifying Uncertainty | 2410.03492 |
| 11 | Benchmark Reliability | Do Large Language Model Benchmarks Test Reliability? (Platinum Benchmarks) | 2502.03461 |
| 12 | Benchmark Reliability | MaP: Reliable Evaluation of Pre-training Dynamics | 2510.09295 |
| 13 | Prompt Sensitivity | PromptRobust: Towards Evaluating the Robustness of LLMs | 2306.04528 |
| 14 | Prompt Sensitivity | On the Worst Prompt Performance of LLMs (RobustAlpacaEval) | 2406.10248 |
| 15 | Prompt Sensitivity | Benchmarking Prompt Sensitivity in LLMs (PromptSET) | 2502.06065 |
| 16 | Prompt Sensitivity | Brittlebench: Quantifying LLM Robustness | 2603.13285 |
| 17 | Prompt Sensitivity | When Chain-of-Thought Backfires: Medical LLM Prompt Sensitivity | 2603.25960 |
