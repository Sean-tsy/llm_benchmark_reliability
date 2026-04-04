# LLM Benchmark Reliability 项目详细报告

## 1. 报告目的与当前状态

本报告基于仓库当前可见的代码与结果文件，对项目的三组实验进行一次完整复盘，覆盖：

- 研究目标与整体实验设计
- 三个实验的具体实现与分析口径
- 当前已有结果的主要发现
- 调试与修复过程中遇到的关键问题
- 当前仍存在的限制与后续建议

本报告使用的主要结果文件包括：

- `exp1/analysis_exp1/analysis_arc.json`
- `exp1/analysis_exp1/analysis_mmlu.json`
- `exp2/analysis_exp2/analysis_arc.json`
- `exp2/analysis_exp2/analysis_mmlu.json`
- `exp3/noise_data/noise_arc_shared150.json`
- `exp3/noise_data/noise_mmlu_shared150.json`
- `exp3/analysis_exp3/analysis_arc_shared150.json`
- `exp3/analysis_exp3/analysis_mmlu_shared150.json`

当前项目已经完成多轮修复，尤其在 `Exp2` 的 parse failure 处理、`Exp3` 的 shared-only 模式、双 paraphrase source 合并、以及 `Exp2` 主分析口径统一方面有明显进展。整体上，项目已经从“实验原型”推进到了“可用于正式写作和展示的分析框架”，但仍保留若干方法学限制，尤其是 paraphrase 的语义保真验证尚未真正落地。

---

## 2. 研究问题与总体思路

本项目的核心问题是：

**LLM benchmark 上观测到的性能差异，有多少来自模型真实能力差异，又有多少来自评测过程本身的噪声。**

为回答这个问题，项目把噪声拆成三类来源：

1. `Exp1`：Prompt wording noise  
   同一道题，在不同 prompt 形式下，模型结果会不会变化。

2. `Exp2`：Test-set wording noise  
   保持题意不变，只重写题干表述，模型结果会不会变化。

3. `Exp3`：Item-level noise identification  
   将 `Exp1` 和 `Exp2` 的波动合并到题目层面，识别高噪声题，再观察过滤这些题后 benchmark 会发生什么变化。

这三者组成了一个清晰的递进结构：

- `Exp1` 测 prompt 侧噪声
- `Exp2` 测测试集表述噪声
- `Exp3` 把前两者合并成题目级噪声地图

---

## 3. 实验设计总览

### 3.1 数据、模型与接口

- 数据集：
  - ARC-Challenge 300 题
  - MMLU-Pro 300 题
- 参评模型：
  - LLaMA-3.1-8B
  - Qwen2.5-7B
  - Qwen3-32B
  - Qwen2.5-72B
- 推理接口：OpenRouter API
- 评测温度：`temperature=0.0`
- Qwen3 特殊处理：追加 `/no_think`，尽量关闭 thinking 模式

### 3.2 三个实验的角色分工

| 实验 | 测量对象 | 基本单位 | 核心输出 |
|------|----------|----------|----------|
| Exp1 | Prompt 变体敏感性 | question x prompt-variant | 准确率波动、flip rate、方差分解 |
| Exp2 | Paraphrase 敏感性 | question x paraphrase-version | 准确率波动、flip rate、cross-source 对比 |
| Exp3 | 题目级噪声 | question | noise score、过滤阈值、清洗后稳定性 |

---

## 4. Exp1：Prompt Perturbation

### 4.1 设计

`Exp1` 的目标是测量：**同一题目在不同 prompt 写法下，模型结果是否稳定。**

项目把 prompt 拆成 4 个维度：

1. `Instruction`
   - `Choose the correct answer.`
   - `Select the best answer.`
   - `Which is correct?`

2. `Answer format`
   - 仅输出字母
   - `Answer: [X]`
   - 解释后再作答

3. `Option format`
   - `A. text`
   - `(A) text`
   - `A) text`

4. `Framing`
   - 无额外角色设定
   - `You are a knowledgeable assistant.`

理论上全因子空间是 `3 x 3 x 3 x 2 = 54` 种，但项目实际采用：

- 1 个 base prompt
- 7 个 OFAT 变体
- 10 个随机 factorial 组合

最终形成 18 个 prompt 变体。

### 4.2 运行机制

`Exp1` 的一个很重要的设计优点是它从一开始就显式考虑了 parse failure。

运行分为两阶段：

- Phase 1：`max_tokens=200`
- Phase 2：对 `is_correct=None` 的 parse failure 样本重跑，`max_tokens=1024`

这意味着 `Exp1` 的基本哲学是：

- 先尽量解析答案
- 无法解析时再补救
- 不轻易把题整道删掉

这也是后来把 `Exp2` 改成类似口径的重要参考。

### 4.3 当前结果

#### ARC-Challenge

| 模型 | mean acc | std | range | item flip rate |
|------|----------|-----|-------|----------------|
| LLaMA-3.1-8B | 0.7919 | 0.0351 | 0.1033 | 0.3500 |
| Qwen2.5-7B | 0.8692 | 0.0525 | 0.2033 | 0.4133 |
| Qwen3-32B | 0.9165 | 0.0521 | 0.1467 | 0.3033 |
| Qwen2.5-72B | 0.8956 | 0.1145 | 0.3333 | 0.4867 |

主要观察：

- `Qwen3-32B` 在 ARC 上的平均表现最好。
- `Qwen2.5-72B` 虽然总体准确率很高，但对 prompt 扰动的波动也最大之一，`std=0.1145`、`range=0.3333`。
- 更大模型不一定更稳，这说明“平均正确率高”和“对 prompt 不敏感”不是同一件事。

#### MMLU-Pro

| 模型 | mean acc | std | range | item flip rate |
|------|----------|-----|-------|----------------|
| LLaMA-3.1-8B | 0.3994 | 0.0377 | 0.1237 | 0.6600 |
| Qwen2.5-7B | 0.4531 | 0.0296 | 0.1027 | 0.6700 |
| Qwen3-32B | 0.5816 | 0.0335 | 0.1314 | 0.6567 |
| Qwen2.5-72B | 0.5452 | 0.0529 | 0.2142 | 0.6267 |

主要观察：

- `Qwen3-32B` 在 MMLU 上也给出最高平均准确率。
- 四个模型在 MMLU 上的 item flip rate 都很高，约在 `0.63-0.67`，说明题目对 prompt 非常敏感。
- 与 ARC 相比，MMLU 不仅更难，也明显更不稳定。

### 4.4 Exp1 的方法学意义

`Exp1` 最重要的贡献是证明了：

- benchmark 上的单次分数并不完全稳定
- prompt wording 自身就是一个实质性的噪声源
- “更大模型一定更稳”并不成立

同时，`Exp1` 也留下了两个已知限制：

- 18 个变体不是正交设计，主效应和交互项可能混杂
- answer parsing fallback 会随被测 answer format 变化

---

## 5. Exp2：Test-Set Resampling via Paraphrasing

### 5.1 设计

`Exp2` 的目标是测量：**同一道题在题意不变、题干改写后，模型结果是否稳定。**

为了避免和 `Exp1` 混淆，`Exp2` 固定使用 base prompt，只改变题目表述。

每道题有 4 个版本：

- 原始题干
- paraphrase 1
- paraphrase 2
- paraphrase 3

### 5.2 Paraphrase source 设计

目前项目使用双 source：

- `gpt4o`
  - 外部模型，不是参评模型家族
  - 主要用于降低“模型自己给自己改题”的偏差
- `qwen`
  - 用 Qwen2.5-72B 生成 paraphrase
  - 作为对照 source

生成 prompt 明确要求：

- 保持原意、难度和技术细节不变
- 只改写题干，不修改选项
- 输出 3 个彼此不同的 paraphrase

并且从旧版“独立三次生成”改成了“单次 prompt 生成 3 条”，温度为 `0.7`，这显著减少了重复 paraphrase 的问题。

### 5.3 当前分析口径

这部分是近期最重要的修复之一。

现在 `Exp2` 采用：

- `full set` 作为主分析
  - parse failure 记为错误
- `clean subset` 作为补充分析
  - 只保留对所有模型都无 parse failure 的题

这种做法比旧版更接近 `Exp1` 的实验哲学，也更适合把 parse failure 当作鲁棒性的一部分，而不是静默删题。

### 5.4 当前结果：ARC-Challenge

#### GPT-4o paraphrases

| 模型 | mean acc | std | flip rate | parse failure rate |
|------|----------|-----|-----------|--------------------|
| LLaMA-3.1-8B | 0.7083 | 0.0227 | 0.2200 | 0.0000 |
| Qwen2.5-7B | 0.8600 | 0.0144 | 0.1400 | 0.0000 |
| Qwen3-32B | 0.9267 | 0.0133 | 0.1067 | 0.0000 |
| Qwen2.5-72B | 0.9533 | 0.0054 | 0.0400 | 0.0000 |

#### Qwen paraphrases

| 模型 | mean acc | std | flip rate | parse failure rate |
|------|----------|-----|-----------|--------------------|
| LLaMA-3.1-8B | 0.7233 | 0.0258 | 0.2000 | 0.0000 |
| Qwen2.5-7B | 0.8633 | 0.0128 | 0.1067 | 0.0000 |
| Qwen3-32B | 0.9250 | 0.0126 | 0.0933 | 0.0000 |
| Qwen2.5-72B | 0.9517 | 0.0064 | 0.0600 | 0.0000 |

ARC 上的主要观察：

- 两个 source 的结论非常一致，`cross-source agreement = 1.0`。
- `Qwen2.5-72B` 是 ARC paraphrase 鲁棒性最强的模型。
- ARC 上 parse failure 基本已被控制到接近 0。
- 与 Exp1 相比，Exp2 的波动整体更小，说明在 ARC 上，prompt 扰动比题干改写更容易带来不稳定。

### 5.5 当前结果：MMLU-Pro

#### GPT-4o paraphrases

| 模型 | mean acc | std | flip rate | parse failure rate |
|------|----------|-----|-----------|--------------------|
| LLaMA-3.1-8B | 0.3483 | 0.0167 | 0.2600 | 0.0233 |
| Qwen2.5-7B | 0.3917 | 0.0252 | 0.2867 | 0.0000 |
| Qwen3-32B | 0.5517 | 0.0114 | 0.3200 | 0.1450 |
| Qwen2.5-72B | 0.5350 | 0.0213 | 0.1600 | 0.0000 |

#### Qwen paraphrases

| 模型 | mean acc | std | flip rate | parse failure rate |
|------|----------|-----|-----------|--------------------|
| LLaMA-3.1-8B | 0.3383 | 0.0191 | 0.2333 | 0.0233 |
| Qwen2.5-7B | 0.3950 | 0.0114 | 0.2133 | 0.0000 |
| Qwen3-32B | 0.5483 | 0.0137 | 0.2733 | 0.1333 |
| Qwen2.5-72B | 0.5283 | 0.0184 | 0.1667 | 0.0000 |

MMLU 上的主要观察：

- `Qwen3-32B` 和 `Qwen2.5-72B` 是表现最强的两档模型。
- `Qwen3-32B` 在 paraphrase 设定下的准确率最高，但 parse failure 率也明显偏高，约 `13%-15%`。
- `LLaMA-3.1-8B` 在 MMLU 上也保留了少量 parse failure。
- clean subset 规模仍明显小于 full set：
  - GPT-4o source：`150 -> 110`
  - Qwen source：`150 -> 117`
- 这说明即使主分析口径已经修复，MMLU 上的解析问题仍是结果解释中的重要组成部分。

### 5.6 Exp2 的关键解释

当前 `Exp2` 告诉我们三件事：

1. benchmark 的 wording instability 是真实存在的；
2. 在 ARC 上，这种 instability 较温和；
3. 在 MMLU 上，题目改写、输出格式稳定性和知识边界更容易混在一起。

同时需要强调：

- `Exp2` 的 parse failure 处理已明显改善；
- 但 paraphrase 的**语义保真验证**仍然没有单独落地；
- 因此当前 `Exp2` 更准确地说是在测：
  - “模型对这些 paraphrase 产物的鲁棒性”
  - 而不完全是“模型对严格语义等价改写的鲁棒性”

---

## 6. Exp3：High-Noise Item Analysis

### 6.1 设计

`Exp3` 把 `Exp1` 和 `Exp2` 的题目级波动合并，构造每道题的 noise score：

`Noise(q) = 1 - |2c(q) - N(q)| / N(q)`

其中：

- `c(q)` 是该题在所有条件下被答对的次数
- `N(q)` 是该题总共被测了多少次

这个定义衡量的是**稳定性**而不是**质量**：

- 全对：noise = 0
- 全错：noise = 0
- 一半对一半错：noise 接近 1

### 6.2 shared-only 模式

这次修复后，`Exp3` 的关键改进是采用 `shared-only`：

- 只保留同时在 `Exp1` 和 `Exp2` 中出现的题
- 也就是共享的 150 题

这样避免了旧版“前 150 题有 Exp1+Exp2，后 150 题只有 Exp1”的不公平比较。

当前 `Exp3` 的结果文件也明确是基于：

- `shared-only`
- `exp2-source both`

### 6.3 当前结果：ARC-Challenge

`noise_arc_shared150.json` 显示：

- 题量：`150`
- combined mean noise：`0.2350`
- `pct_above_0.5`：`22.0%`
- `exp1_only mean`：`0.2360`
- `exp2_only mean`：`0.2238`

过滤阈值：

- top 10% cutoff：`0.75`
- top 20% cutoff：`0.538462`
- top 30% cutoff：`0.288462`

ARC 的 top noisy items 前几项：

- `Mercury_SC_402276`：`0.961538`
- `Mercury_SC_412374`：`0.942308`
- `Mercury_7212888`：`0.942308`

这些题的共同特点是整体准确率非常接近 `0.5`，说明不同模型和不同条件下的结果高度不稳定。

### 6.4 当前结果：MMLU-Pro

`noise_mmlu_shared150.json` 显示：

- 题量：`150`
- combined mean noise：`0.4477`
- `pct_above_0.5`：`39.33%`
- `exp1_only mean`：`0.4560`
- `exp2_only mean`：`0.2526`

过滤阈值：

- top 10% cutoff：`0.916667`
- top 20% cutoff：`0.807692`
- top 30% cutoff：`0.652632`

MMLU 的 top noisy items 前几项：

- `9692`：`1.0`
- `5911`：`0.980769`
- `3588`：`0.980392`

这些题的准确率也几乎都在 `0.48-0.52` 附近，说明模型在这些题上的表现极不稳定。

### 6.5 Exp3 的核心结论

目前最清楚的结论是：

1. **MMLU 比 ARC 明显更 noisy。**
   - ARC mean noise：`0.2350`
   - MMLU mean noise：`0.4477`

2. **Prompt 噪声仍然比 paraphrase 噪声更强。**
   - ARC：`exp1_only 0.2360 > exp2_only 0.2238`
   - MMLU：`exp1_only 0.4560 > exp2_only 0.2526`

3. **高噪声题在 MMLU 上更集中、更极端。**
   - 30% removal cutoff 在 ARC 仅 `0.288462`
   - 在 MMLU 高达 `0.652632`

4. **模型之间对“哪些题 noisy”的看法只有中等相关。**
   例如：
   - ARC 中最高相关约为 `0.3860`
   - MMLU 中最高相关约为 `0.4322`

这说明“高噪声题”并不是一个完全模型无关的客观属性，而是带有一定模型相对性。

### 6.6 三源方差分解的含义

`analysis_*_shared150.json` 中的 `variance_decomposition_3way` 显示：

- 在 ARC 上，prompt variance 对多数模型占主导，尤其较强模型更明显。
- 在 MMLU 上，prompt variance 与 sampling variance 往往更接近，说明该 benchmark 同时受到 prompt 和 test-set wording 的双重影响。

例如 baseline 下：

- ARC / `qwen2.5-72b`
  - `pct_prompt = 94.55%`
  - `pct_sampling = 5.22%`
  - `pct_testset = 0.23%`

- MMLU / `qwen2.5-7b`
  - `pct_prompt = 36.62%`
  - `pct_sampling = 60.65%`
  - `pct_testset = 2.74%`

这进一步支持一个重要判断：

**ARC 更像是“prompt 敏感型 benchmark”，MMLU 更像是“多源不稳定型 benchmark”。**

---

## 7. Debug 与修复过程复盘

本项目的一个重要特点是：不仅跑实验，也持续修正实验设计本身。下面按问题类型复盘。

### 7.1 Exp2：Qwen3 thinking 开关与 max_tokens

早期 `Exp2` 的两个关键问题是：

- Qwen3 使用了错误的指令 `/nothink`
- `max_tokens=128` 太小

其后果是：

- MMLU 上出现大量 parse failure
- 尤其是 `Qwen3-32B` 和 `LLaMA-3.1-8B`
- 早期结果中 parse failure 一度达到 `16%-24%`

修复后做了三件事：

- 改成 `/no_think`
- Phase 1 提高到 `200`
- 新增 Phase 2，对 parse failure 用 `1024 tokens` 重跑

这一步是整个项目最关键的 engineering fix 之一，因为它直接决定 `Exp2` 的结果是否可信。

### 7.2 Exp2：从 clean subset 主分析改为 full-set 主分析

这一步是最重要的方法学修复之一。

旧版逻辑的问题在于：

- parse failure 一出现，就整题从分析里被剔除
- 结果就变成了“只在能被顺利解析的题上分析”

修复后的逻辑是：

- 主分析：full set，parse failure 当作错误
- 补充分析：clean subset

这样一来：

- `Exp1` 和 `Exp2` 的哲学更一致
- parse failure 不再悄悄改变分析对象
- clean subset 从“默认口径”降级成“敏感性检验”

### 7.3 Exp2：paraphrase source 偏差

用户一开始提出的一个关键观察是：

**用参评模型来生成 paraphrase 是否合理。**

这是非常准确的问题。因为如果 `qwen2.5-72b` 既是 paraphrase generator，又是参评模型家族之一，那么 `Exp2` 的结果就可能混入 source bias。

项目后续加入 `gpt4o` 作为外部 paraphrase source 后，这个问题已经从“严重设计偏差”降为“需要说明的对照设置”。

当前的最佳理解应该是：

- `gpt4o` source 是主参考
- `qwen` source 是对照条件
- 两个 source 的一致率目前为 `1.0`

### 7.4 Exp2：顺序截取前 150 题

旧版 `Exp2` 直接取 `items[:150]`，存在顺序采样偏差。

后续修复改成：

- MMLU：按 `category` 做分层随机抽样
- ARC：固定种子随机抽样

这一步在设计上是明显进步，但要注意：

**如果要把这套新采样逻辑作为最终标准，整个 Exp2 与 Exp3 最好从 paraphrase generation 开始完整重跑一次。**

### 7.5 Exp3：300 题与 150 题混用问题

早期 `Exp3` 的一个根本问题是：

- `Exp1` 用 300 题
- `Exp2` 只用 150 题
- 两者直接合并后，后 150 题天然少一个噪声维度

这会让后半部分题目因为“被观测得更少”而系统性更不 noisy。

修复方案是 `shared-only`：

- 只保留共享的 150 题
- 并把输出文件明确标成 `_shared150`

这一步让 `Exp3` 的噪声比较真正可解释。

### 7.6 Exp3：双 source 整合与 tagged analysis

早期 `Exp3` 还有两个流程问题：

- 只读一个 `Exp2` source
- `analyze/visualize` 不能自然接住 tagged 结果

后续修复包括：

- `run_experiment3.py` 支持 `--exp2-source both`
- `analyze_experiment3.py` 支持 `--noise-tag`
- `visualize_experiment3.py` 支持读取 tagged 文件并输出单独图目录

这使得 `Exp3` 从“局部能跑”变成了“端到端完整闭环”。

---

## 8. 当前仍存在的问题

尽管项目已经大幅改善，但以下问题仍然存在。

### 8.1 paraphrase 语义保真仍未验证

这是当前最重要的未解决问题。

现在有：

- 生成 prompt 中要求保持原意
- 多样性统计

但还没有：

- 语义等价打分
- 答案不变性验证
- 人工抽检

因此，`Exp2` 里的一部分波动仍可能混入 semantic drift。

### 8.2 paraphrase 质量元数据不足

现在的 paraphrase 文件没有完整记录：

- 是否使用了 fallback
- 生成是否严格按 JSON 成功解析
- 哪些题的 3 个 paraphrase 实际不够理想

这会影响后续按质量分层分析。

### 8.3 Exp1 设计不是正交实验

`Exp1` 当前的 18 个变体在工程上是合理的，但不构成严格正交设计，因此：

- 主效应估计可能混入交互项
- OLS 结果更适合做探索性解释，而不是严格因果识别

### 8.4 Exp3 的 noise score 衡量稳定性，不等于 benchmark 质量

当前定义下：

- 一道题如果全对，noise=0
- 一道题如果全错，noise=0

所以 `Exp3` 清洗掉的是“不稳定题”，不是“坏题”。

这在论文里必须明确区分。

---

## 9. 综合结论

从当前结果看，本项目已经支持以下较强结论：

1. **Benchmark 分数存在显著评测噪声。**
   这种噪声不仅来自模型本身，也来自 prompt 与 test-set wording。

2. **Prompt wording noise 是一个非常强的噪声源。**
   尤其在 `Exp1` 中，四个模型在 MMLU 上都表现出非常高的 item flip rate。

3. **Test-set wording noise 也真实存在，但强度通常弱于 prompt noise。**
   这一点在 `Exp3` 的 `exp1_only` 与 `exp2_only` 对比中很清楚。

4. **MMLU-Pro 比 ARC-Challenge 更 noisy。**
   这在 `Exp1`、`Exp2`、`Exp3` 三个层面都得到了支持。

5. **高准确率不自动意味着高稳定性。**
   一些更强模型在某些 benchmark 上分数更高，但对 prompt 或 parsing 也更敏感。

6. **清洗高噪声题是有意义的分析方向。**
   `Exp3` 已经给出了题目级噪声分数与 removal set，后续可以继续用于“去噪 benchmark”的下游分析。

---

## 10. 后续建议

如果要把这个项目进一步推进到论文或正式答辩版本，我建议按下面顺序继续完善。

### 高优先级

1. 增加 paraphrase 语义保真检查模块
2. 为 paraphrase 文件加入 QC 元数据
3. 如果采纳新的分层采样代码，完整重跑 `Exp2 -> Exp3`

### 中优先级

1. 统一 `Exp1` 和 `Exp2` 的显著性校正标准
2. 为 `Exp1` 设计矩阵加入混淆度说明，例如 VIF
3. 在报告中明确区分“稳定性”与“质量”

### 低优先级

1. 改善 figure 脚本中的配置复用
2. 为结果文件增加版本标识，避免“新代码 + 旧结果”混淆
3. 增加一份专门的 reproduction checklist

---

## 11. 最终评价

从当前仓库状态看，这个项目已经不只是“跑了几组 benchmark”，而是形成了一个比较完整的**benchmark reliability analysis framework**：

- `Exp1` 提供 prompt 扰动分析
- `Exp2` 提供 paraphrase 重采样分析
- `Exp3` 提供题目级噪声定位与清洗

项目最有价值的部分，不只是得到了一些数字，而是逐步把“实验结果不稳定”这件事拆解成了可分析、可修复、可解释的多个来源。  
在经历多轮 debug 和设计修复后，这套框架现在已经具备较强的方法论说服力。它的下一步重点不再是“再多跑一些结果”，而是把剩余的验证环节补齐，特别是 paraphrase 语义保真和结果版本一致性。
