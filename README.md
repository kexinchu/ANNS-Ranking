# Research Gap Map for Budget-Aware Multimodal Late-Interaction Retrieval

## 问题定义

现有 retrieval 系统已经开始学习如何在固定候选集上做 truncation、adaptive reranking 或 late-interaction pruning，但多数方法仍把 `retrieval` 和 `reranking` 视为两个松耦合阶段。它们通常默认第一阶段已经给出了一个可接受的候选集，后续工作只是决定 rerank 多深、算多少、何时停止。

这个设定在文本检索中已经开始暴露局限，在多模态 late-interaction 场景下问题会更明显。因为预算不再只是 “rerank 前 50 个还是 100 个”，而是要同时决定 `candidate budget`、`interaction budget`、`modality budget`，以及这些预算是否应该随 query 难度和证据结构动态变化。

## 核心判断

**Problem.** 现有方法大多只在固定候选上做 truncation 或 adaptive rerank，缺少 retrieval 与 reranking 的闭环预算控制。

**Why now.** late-interaction 与 multimodal retrieval 让预算控制从单一的 rerank depth 问题，升级为候选选择、交互揭示、模态分配与停止判定的联合决策问题。

**Claim.** 真正的 research gap 不是“再做一个更聪明的 truncation heuristic”，而是让计算对最终 `top-k ranking fidelity` 或 `evidence utility` 负责，并把 retrieval、reranking 与 modality-specific computation 放进同一个预算分配框架。

## Research Gap Map

下表中的缩写见文末 `References`。

| Gap Layer | Representative Work | What Existing Work Solves | What Remains Open | Why This Becomes Harder in Multimodal Late-Interaction | Paperable Opportunity |
| --- | --- | --- | --- | --- | --- |
| Closed-loop control is missing | RLT, GenRT, AcuRank, Col-Bandit, RGS | 已经开始学习 query-adaptive truncation、reranking 深度控制或在 reranking 内部按不确定性分配计算；RGS 进一步让 reranker 偏好影响搜索次序。 | 仍缺少统一策略来回答：什么时候应该回头扩大 retrieval、补更多候选、切换模态、或增加 ANN probing，而不是只在当前候选集内继续算。 | 多模态场景中的漏召回不是均匀噪声；漏掉一个表格区域、patch 或 OCR 证据，可能直接改变最终 top-k。闭环控制必须能决定“去哪里补证据”。 | 构建 `closed-loop retrieval-reranking controller`，以 ranking risk 或 evidence deficit 为信号，联合决定扩召、补证据和停止。 |
| Budget allocation is still depth-centric, not candidate-/evidence-centric | RLT, AcuRank, RGS, Col-Bandit | 现有方法多数已经会学“rerank 到多深”或“把计算集中到不确定候选”。RGS 证明在固定 reranker budget 下，选谁来 rerank 本身就是关键问题。 | 预算仍主要表现为 cut-off depth，而不是对 candidate、evidence unit 或 modality 的选择性暴露。系统仍不擅长回答“预算应该花在谁身上”。 | late-interaction 的计算单元不是整篇文档，而是 token、patch、region、frame 等局部证据；多模态下最优预算对象不一定是 top-N 文档，而可能是少量高风险交互。 | 从 `depth control` 升级为 `selective budget allocation`：联合选择 candidate、证据单元和模态，而不只是机械增加 rerank depth。 |
| Objective is still ranking quality, not downstream utility / coverage | GenRT, IGP, SetR, ToolRerank | 大多数方法优化的是 ranking quality、ranking uncertainty 或 quality-cost trade-off；IGP 和 SetR 已开始强调 generator-aligned utility、coverage 与 set-level quality。 | 预算控制的目标仍多停留在“保住排序”，而不是“最大化最终 answer utility、evidence coverage、harm filtering”。 | 多模态 RAG 往往需要一组互补证据，而不是单篇最相关文档。图像、表格和文本之间还存在互补与冲突关系，单文档 relevance 更难代表最终效用。 | 把预算控制目标从 `rank preservation` 扩展到 `utility-aware evidence selection`，显式建模 coverage、complementarity 与 harmful evidence filtering。 |
| Uncertainty exists but is poorly calibrated and weakly transferable | AcuRank, Col-Bandit, RLT | 已有方法广泛使用 ranking uncertainty、score bounds 或稳定性信号来控制计算量。 | 仍不清楚这些 uncertainty 在不同 retriever、reranker、domain、模态组合之间是否可比较、可迁移、可校准。 | 多模态 late-interaction 的 uncertainty 来自多个来源：ANN miss、模态缺失、局部交互未揭示、跨模态 grounding 失败。不同来源的风险难以统一到同一量纲。 | 研究 `pipeline-agnostic uncertainty abstraction`，让风险信号能跨模型、跨模态、跨数据集迁移，并真正指导预算分配。 |
| Evaluation is not budget-aware enough for multimodal late-interaction | RLT, RGS, Col-Bandit, SetR | 现有工作已经开始报告 reranking FLOPs、latency、Reranker@k、ranking fidelity 等效率指标。 | 仍缺少统一评测框架来衡量 retrieval budget、reranking budget、interaction budget、modality budget 如何共同影响最终质量。 | 多模态系统的预算不仅是 reranker calls，还包括 ANN probes、revealed interactions、modality-specific compute 与 evidence coverage；单一 NDCG 或单一 FLOPs 指标不足以刻画代价。 | 建立 `budget-aware evaluation framework`，同时衡量 ranking fidelity、evidence utility、coverage、budget split 与 modality-aware compute。 |

## Research Positioning

本文档最终主张的问题不是：

- 只做更好的 truncation
- 只做 adaptive reranking
- 只做 subset selection

而是：

> **在固定总预算下，系统应如何闭环地、风险感知地、按模态联合决定：retrieve who、rerank who、allocate compute to which modality、when to stop、以及 when to go back for more evidence。**

这一定义与现有工作的边界如下：

- 相对 `RLT / GenRT`：我们关心的不只是截断位置，而是 retrieval 与 reranking 的双向联动。
- 相对 `AcuRank / Col-Bandit`：我们关心的不只是 reranking 内部如何省算，而是预算能否回流到 candidate generation 与 modality allocation。
- 相对 `SetR / IGP`：我们认同 utility-aware 目标，但重点放在预算控制策略如何服务于 evidence utility，而不是单独研究 set selection。
- 相对 `RGS`：我们认可“选谁 rerank”是关键一步，但目标是把这一步进一步扩展到多模态 late-interaction 的统一预算抽象。

## Synthesis: What the Field Still Cannot Do

当前文献已经能在固定候选集上更聪明地省 reranking 成本，也开始意识到 query difficulty、uncertainty 和 set-level utility 的重要性。但整个领域仍未形成一个真正统一的预算控制框架。

更具体地说，现有方法还不能稳定地做到以下三点：

1. 在同一套策略中联合决定 retrieval expansion、candidate admission、reranking depth、interaction reveal 和 modality allocation。
2. 将 query 的“难在什么地方”映射为不同的预算动作，例如补更多候选、换模态、做更多局部交互，或提前停止。
3. 用同一套 evaluation protocol 同时度量 `ranking fidelity`、`evidence utility`、`coverage` 与 `budget split`，并解释不同预算维度之间的替代关系。

## Priority Research Directions

### 1. Closed-loop retrieval-reranking control

让 reranking uncertainty 或 evidence deficit 反向驱动 retrieval expansion、ANN probing 深化或模态切换，而不是只在固定 shortlist 上继续精排。

### 2. Selective budget allocation over candidates, evidence, and modalities

把预算从“截断到多深”升级为“预算应该投给谁、哪类证据、哪种模态、哪种交互”。这比单纯增加 rerank depth 更符合 late-interaction 的真实计算结构。

### 3. Utility-aware evidence selection beyond rank preservation

把目标从保护 top-k 排序扩展到保护 evidence utility，包括 coverage、complementarity 和 harmful evidence filtering，尤其适用于 reasoning-heavy 与 multimodal RAG。

### 4. Unified abstraction for multimodal late-interaction budgeting

建立一个统一抽象，把 `candidate budget`、`interaction budget`、`modality budget` 和 `stopping budget` 放进同一个控制空间，而不是分别做局部 heuristic。

## Working Problem Statement for This Repo

本仓库当前聚焦的问题可以收束为一句话：

> **Under a fixed total budget, how should a system jointly allocate retrieval, reranking, and modality-specific computation to maximize final ranking fidelity or evidence utility?**

把它展开后，至少包含五个操作层面的子问题：

- 应该 `retrieve who`，而不是默认只接受 first-stage 的固定候选。
- 应该 `rerank who`，而不是默认 rerank top-N retrieved。
- 应该把额外计算投给 `which modality / which evidence unit`，而不是平均分配给所有 token、patch 或文档。
- 应该 `when to stop`，而不是固定 compute cap。
- 应该 `when to go back for more evidence`，而不是把 retrieval 与 reranking 严格割裂。

这一定义默认面向 `multimodal + late-interaction + budget coupling`，而不是一般性的 RAG pipeline 综述。

## Recommended Benchmark, Dataset, and Metric Stack

### 选择标准

对这个问题，最合适的 benchmark 不应只满足“多模态”或“RAG”两个标签，而应尽量同时覆盖以下属性：

- 检索对象本身是视觉丰富或跨模态的文档，而不是纯文本 passage。
- 任务重点在 retrieval / reranking，而不是把检索误差淹没在生成误差里。
- 数据集足够大，能让 `candidate budget`、`ANN budget`、`interaction budget` 的差异真正显现出来。
- 至少覆盖以下一种困难来源：表格 / 图表 / OCR / layout grounding、query rephrasing、reasoning-intensive retrieval、长链路 evidence planning。
- 最好支持切片分析，而不是只给一个总分。

基于这个标准，我不建议只用单一 benchmark。更合理的是采用“**主 benchmark + 补充 stress test + 可选 downstream benchmark**”的组合。

### 推荐组合

#### 最小可发表组合

- `ViDoRe V3`
- `REAL-MM-RAG`
- `MM-BRIGHT`

这三个 benchmark 的覆盖面最互补：

- `ViDoRe V3` 负责测文档级多模态 grounding 和 visually rich retrieval。
- `REAL-MM-RAG` 负责测 query rephrasing robustness 和 table-heavy retrieval。
- `MM-BRIGHT` 负责测 reasoning-intensive multimodal retrieval。

如果论文只做 retrieval / reranking 层，不强行宣称 end-to-end agentic MM-RAG 收益，这个三件套已经足够有说服力。

#### 完整论文组合

- 主 benchmark：`ViDoRe V3` + `REAL-MM-RAG` + `MM-BRIGHT`
- 文本 reasoning control：`BRIGHT`
- 可选 downstream benchmark：`MC-Search`

这里的逻辑是：

- `BRIGHT` 用来区分“你的方法提升来自多模态预算控制”还是“只是一般 reasoning retrieval 变强了”。
- `MC-Search` 只在你要额外 claim `downstream evidence utility`、`planning fidelity` 或 `agentic MM-RAG` 改善时加入；否则会把论文主线从 retrieval policy 拉向 end-to-end agent design。

### Benchmark / Dataset 推荐表

| Role | Benchmark / Dataset | Why it fits this problem | Recommended use |
| --- | --- | --- | --- |
| Main benchmark | `ViDoRe V3` | 官方数据卡说明其包含 10 个数据集、约 26,000 页、3,099 个查询、6 种语言，并提供人工验证的 relevant pages、bounding boxes 和参考答案。它最适合测 page-level retrieval、OCR / table / figure / layout grounding，以及 late-interaction 对局部证据的利用能力。 | 作为主结果 benchmark。优先报告 ranking fidelity、evidence hit 和 modality-aware compute。 |
| Main benchmark | `REAL-MM-RAG` | ACL 2025 论文明确把目标设为更接近真实多模态检索，强调四点：多模态文档、更高难度、realistic RAG queries、准确标注；同时提供基于 query rephrasing 的多难度设置。它特别适合暴露 query rewrite robustness 与 table-heavy failure modes。 | 作为鲁棒性 benchmark。重点分析 paraphrase stability、table-heavy slice、rephrase levels。 |
| Main hard benchmark | `MM-BRIGHT` | 官方站点将其定位为 reasoning-intensive multimodal retrieval benchmark，包含 2,803 个真实查询、29 个技术域、四种任务，且 `Task 2 (MM→Text)` 与 `Task 4 (MM→MM)` 直接对应多模态查询条件下的难检索场景。它非常适合测 modality selection、hard-query budget allocation 和 reasoning-heavy retrieval。 | 作为 hardest slice benchmark。优先使用 `Task 2` 和 `Task 4`，必要时用 `Task 1` 做 text-only control。 |
| Text-only control | `BRIGHT` | BRIGHT 是 reasoning-intensive retrieval 的强文本基准，1,398 个真实查询跨多个专业域。它能帮助你判断收益是否主要来自 reasoning-aware retrieval，而不是多模态预算控制本身。 | 作为控制实验，不必成为主表 benchmark。 |
| Optional downstream benchmark | `MC-Search` | MC-Search 是 agentic MM-RAG benchmark，包含 3,333 个带 step-wise reasoning chain 的例子，并提供 process-level metrics。它更适合评估“检索策略是否提升后续 reasoning / planning fidelity”，而不是单纯的 top-k 排序质量。 | 仅在论文声称 downstream utility 改善时加入。否则可不做。 |

### 数据集层面的具体建议

#### 1. ViDoRe V3

推荐直接使用 `full benchmark`，不要只挑少数简单子集。原因是这个 benchmark 的价值就在于它把视觉文档检索中的多种真实困难压在一起，包括表格、图表、版式、页面图像和 OCR 痕迹。

如果算力有限，优先保留那些视觉结构更强的子集，而不是退回到几乎可由 OCR 文本解决的子集。因为你的问题本质上是在研究：当证据以局部视觉结构存在时，预算应如何分配给 `candidate retrieval` 与 `late interaction`。

#### 2. REAL-MM-RAG

推荐至少覆盖以下三个公开子集：

- `FinSlides`
- `FinReport`
- `TechReport`

原因很直接：

- `FinSlides` 以 table-heavy financial presentations 为主，最能暴露图表和表格证据的 miss。
- `FinReport` 同时包含文本和结构化表格，适合测 text/table 混合文档上的预算联动。
- `TechReport` 偏文本但带 visual elements 和 structured tables，适合看 query rephrasing 后的语义鲁棒性。

如果只选一个子集，优先选 `FinSlides`；如果选两个，优先 `FinSlides + FinReport`。

#### 3. MM-BRIGHT

推荐优先做：

- `Task 2: Query+Image → Documents`
- `Task 4: Query+Image → Documents+Images`

原因是这两项最贴近“多模态 late-interaction retrieval under budget”的设定：

- `Task 2` 能看系统是否真的会利用 query 中的视觉上下文，而不是退化成文本检索。
- `Task 4` 则直接测试系统能否在 multimodal query 和 multimodal corpus 同时存在时，做出正确的 candidate admission 与 reranking。

`Task 1` 可以保留为文本 reasoning control，但不建议把精力主要放在 `Task 3`，因为你的核心问题不是纯 image retrieval。

### 推荐指标

我建议把指标分成五组，而不是只报 `Recall@k` 或只报 `nDCG`。

| Metric Group | Recommended Metrics | Why they are necessary |
| --- | --- | --- |
| Ranking quality | `nDCG@10`, `Recall@10`, `Recall@50`, `MRR@10` | `nDCG@10` 应作为主 headline metric，因为你的目标是最终 top-k 质量而不是纯召回；`Recall@10/50` 用来分解 first-stage candidate sufficiency；`MRR@10` 适合观察 top-1 / top-few 是否更稳定。 |
| Ranking stability | `Top-k overlap@10` with exhaustive reranking, `Kendall tau` or `Spearman rho`, `Top-k inversion rate`, `Paraphrase stability` | 这个问题的核心不是平均 relevance，而是 budget 变化后 top-k 是否翻转。尤其在 `REAL-MM-RAG` 上，paraphrase stability 应是主指标之一。 |
| Budget / efficiency | `P50/P95 latency`, `FLOPs`, `ANN probes`, `candidate set size`, `revealed interaction ratio`, `modality-specific compute share` | 你的方法主张的是联合预算控制，因此必须把 retrieval 成本和 reranking 成本拆开，而不是只报总 latency。`revealed interaction ratio` 对 late-interaction 方法尤其关键。 |
| Calibration / control quality | `Brier score` or `ECE` on stop / risk prediction, `quality-cost frontier` or `AUC under budget curve` | 如果方法显式建模 risk 或 stopping，就必须评估“知道何时停”是否真的被校准。单点预算结果不足以说明控制策略是否可靠。 |
| Evidence utility | `Evidence hit@k`, `bbox hit rate` on ViDoRe V3, optional `answer EM/F1` on downstream benchmarks | 如果你要 claim utility-aware retrieval，就需要证明系统不仅排得更对，还更常把真正的关键证据拿进 top-k。ViDoRe V3 的 bounding boxes 很适合做 evidence-level evaluation。 |

### 这些指标里，哪些是必须的

如果目标是一篇聚焦 retrieval policy 的论文，最低限度我建议每个 benchmark 都报告：

- `nDCG@10`
- `Recall@10`
- `Top-k overlap@10` with exhaustive reranking
- `P50/P95 latency`
- `ANN probes`
- `revealed interaction ratio`

然后再加两个与你方法最匹配的附加指标：

- 如果方法强调 `risk-aware stopping`，加 `ECE` 或 `Brier score`
- 如果方法强调 `evidence utility`，加 `bbox hit rate` 或 `Evidence hit@k`

### 我最推荐的论文主指标组合

如果只能选一组最有代表性的 headline metrics，我会建议：

- 主质量指标：`nDCG@10`
- 主稳定性指标：`Top-k overlap@10` with exhaustive reranking
- 主效率指标：`revealed interaction ratio` + `P95 latency`
- 主鲁棒性指标：`paraphrase stability` on `REAL-MM-RAG`
- 主证据指标：`bbox hit rate` on `ViDoRe V3`

这组指标最能直接对应你的研究问题：

> 在固定总预算下，系统是否能更稳定地保住 top-k，并把计算更准确地投向真正关键的多模态证据？

## Executable Experiment Matrix

### 通用执行协议

为了让不同 benchmark 上的实验可以横向比较，建议先固定一套通用协议：

- 所有 `policy comparison` 尽量共享同一套 encoder、同一套索引和同一候选生成后端，只改变预算控制策略。
- 所有结果默认返回 `top-10`，主 headline metric 统一为 `nDCG@10`。
- 对于大规模语料上无法真正做全量 exhaustive reranking 的情况，用 `high-budget oracle run` 代替，并在论文中写清楚它的定义。
- 建议把 `1.0x budget` 定义为：每个 backbone 在开发集上达到候选召回基本饱和的高预算设置，然后把所有更低预算都归一化到这个成本上。
- 所有随机策略或 learned controller 至少跑 `3` 个 seed；latency 采用 warm-up 后的 `P50/P95`。

推荐统一使用四类预算轴：

- `Total budget ratio`：`{0.1, 0.2, 0.4, 0.6, 0.8, 1.0}`
- `Candidate budget K`：`{20, 50, 100, 200, 500}`
- `Interaction reveal ratio r`：`{0.1, 0.2, 0.4, 0.6, 0.8, 1.0}`
- `Modality split`：`{100/0, 75/25, 50/50, 25/75, 0/100}`

这里的 `modality split` 指额外预算在文本/OCR 与视觉证据之间的分配比例；如果方法支持更细粒度模态，例如 table / chart / patch / OCR，可在附录里再做更细切分。

### 最小 baseline 包

如果目标是尽快开跑并形成第一版论文主表，至少跑以下 5 组：

- `Fixed retrieval + fixed reranking`
- `Fixed retrieval + adaptive reranking`
- `Equal-modality budget baseline`
- `High-budget oracle run`
- `Proposed closed-loop method`

如果算力允许，再补：

- `Candidate-selection baseline`
- `Text-only control on BRIGHT`

### Run Matrix

| Run ID | Goal | Benchmarks | Settings to run | Budget sweep | Must-save outputs |
| --- | --- | --- | --- | --- | --- |
| `Run-0` | 标定 oracle 与预算归一化 | ViDoRe V3, REAL-MM-RAG, MM-BRIGHT, BRIGHT | 每个 backbone 跑一组 `high-budget oracle run`，记录候选召回饱和点、full interaction 成本、P95 latency。 | 不扫预算，只确定 `1.0x`。 | Oracle nDCG@10、Recall@50、P95 latency、candidate recall saturation、interaction cost。 |
| `Run-1` | 主结果 frontier | ViDoRe V3, REAL-MM-RAG, MM-BRIGHT | 对所有核心 baseline 和你的方法，在相同 `total budget ratio` 下跑完整 frontier。 | `B_total ∈ {0.1,0.2,0.4,0.6,0.8,1.0}` | nDCG@10、Top-k overlap@10、P95 latency、ANN probes、revealed interaction ratio。 |
| `Run-2` | 分解 budget 到各个动作 | ViDoRe V3, REAL-MM-RAG, MM-BRIGHT | 固定总预算后，分别只扫一个预算轴，分析收益来自哪里。 | `K`, `r`, `modality split` 各自单独扫。 | 候选数、interaction ratio、modality compute share、质量变化曲线。 |
| `Run-3` | 鲁棒性与 slice 分析 | REAL-MM-RAG, MM-BRIGHT, ViDoRe V3 | 按 rephrase difficulty、table-heavy、visual grounding、hard-query slice 分组。 | 只跑 `0.2, 0.4, 0.8, 1.0` 四个预算点。 | Paraphrase stability、top-k inversion、bbox hit rate、slice-level nDCG@10。 |
| `Run-4` | 控制实验 | BRIGHT | 在 text-only reasoning benchmark 上复现主 frontier。 | `B_total ∈ {0.2,0.4,0.8,1.0}` | nDCG@10、P95 latency、candidate budget 敏感性。 |
| `Run-5` | 可选 downstream transfer | MC-Search | 只对最终方法和 1-2 个强 baseline 跑 end-to-end agentic MM-RAG。 | 只跑 `0.4, 0.8, 1.0`。 | Answer EM/F1、step retrieval accuracy、planning fidelity、retrieval latency。 |

### Benchmark-by-Benchmark 设置

| Benchmark | What to run | Primary budget axis | Secondary budget axis | Primary metrics | Why this is the right stress test |
| --- | --- | --- | --- | --- | --- |
| `ViDoRe V3` | Full benchmark；优先保留视觉结构强的子集 | `B_total`, `r` | `K`, `modality split` | `nDCG@10`, `Top-k overlap@10`, `bbox hit rate`, `revealed interaction ratio` | 最适合暴露 page-level grounding、layout/table/chart miss，以及局部视觉证据是否真正被分到预算。 |
| `REAL-MM-RAG` | `FinSlides`, `FinReport`, `TechReport`；原始 query + rephrased levels | `B_total`, `K` | `r` | `nDCG@10`, `paraphrase stability`, `top-k inversion`, `P95 latency` | 最适合测 query rewriting robustness 和 table-heavy 文档上的候选预算错误。 |
| `MM-BRIGHT` | 重点跑 `Task 2` 与 `Task 4`；`Task 1` 作为 control | `B_total`, `modality split` | `K`, `r` | `nDCG@10`, `Top-k overlap@10`, `modality compute share`, `hard-slice nDCG@10` | 最适合测 reasoning-heavy、多模态 query 条件下预算是否被正确投向对应模态。 |
| `BRIGHT` | Full benchmark | `B_total`, `K` | 无需主扫 `modality split` | `nDCG@10`, `Recall@10`, `P95 latency` | 用来证明收益不是“任何 reasoning retrieval 都会提升”，而是你的 multimodal budget policy 确实带来额外价值。 |
| `MC-Search` | 仅最终模型与少量强 baseline | `B_total` | 可选 `K` | `answer EM/F1`, `step retrieval accuracy`, `planning fidelity` | 只在需要支撑 downstream utility claim 时有必要。 |

### 每个 Run 该怎么组织

#### Run-0: Oracle calibration

目标不是出论文表，而是确定后续所有预算 sweep 的归一化基准。

建议输出：

- 每个 benchmark 的 `1.0x` 成本定义
- 在高预算下的候选召回是否已经基本饱和
- full interaction 是否已经接近质量上限

如果某个 benchmark 在 `K=200` 后已经基本饱和，就不要继续把 `K=500` 作为主实验预算点，只把它留在附录。

#### Run-1: Main frontier

这是主论文结果的核心。每个 benchmark 都跑：

- `Fixed retrieval + fixed reranking`
- `Fixed retrieval + adaptive reranking`
- `Equal-modality budget baseline`
- `Proposed method`
- `High-budget oracle run`

所有方法都在相同 `total budget ratio` 上对齐，而不是在相同 `K` 或相同 `probe count` 上对齐。因为论文主张的是总预算控制，而不是单组件优化。

#### Run-2: Component sweeps

这一组主要回答“收益从哪里来”，建议拆成三张小表或三组图：

- 固定 `B_total=0.4`，只扫 `K`
- 固定 `B_total=0.4`，只扫 `r`
- 固定 `B_total=0.4`，只扫 `modality split`

如果你的方法支持 closed-loop 回补 retrieval，再额外做一组：

- 固定总预算，比较 `one-way rerank-only policy` vs `closed-loop retrieval backoff`

#### Run-3: Slice and robustness

这一组应该重点做：

- `REAL-MM-RAG` 上的 rephrase difficulty 分层
- `ViDoRe V3` 上的 table / chart / OCR / layout slice
- `MM-BRIGHT` 上的 hard reasoning domains 或 multimodal tasks 分层

这组结果最容易支持“平均提升之外， hardest slice 收益更大”的 claim。

### 建议的预算点选择

如果你希望第一版实验更轻量，我建议先跑下面这组最稳的预算点：

- 主 frontier：`0.2, 0.4, 0.8, 1.0`
- 主 candidate depth：`50, 100, 200`
- 主 interaction reveal：`0.2, 0.4, 0.8, 1.0`
- 主 modality split：`75/25, 50/50, 25/75`

等第一版结果出来后，再扩展到完整的六点或五点 sweep。

### 主表和附表怎么组织

#### 主表

`Table 1: Matched-budget main results`

- 数据集：ViDoRe V3, REAL-MM-RAG, MM-BRIGHT
- 行：各 baseline + proposed method
- 列：`nDCG@10`, `Top-k overlap@10`, `P95 latency`, `ANN probes`, `revealed interaction ratio`
- 预算：固定 `B_total=0.4` 或 `0.8`

`Table 2: Frontier summary`

- 数据集：同上
- 行：各方法
- 列：`AUC under budget curve`, `best nDCG@10 @ 0.4x`, `best nDCG@10 @ 0.8x`

`Table 3: Robustness and evidence quality`

- REAL-MM-RAG：`paraphrase stability`
- ViDoRe V3：`bbox hit rate`
- MM-BRIGHT：`hard-slice nDCG@10`

#### 主图

`Figure 1`

- 三个 benchmark 的 `quality-efficiency frontier`

`Figure 2`

- `modality split` 热力图，展示不同 benchmark / slice 的最优预算分配不同

`Figure 3`

- `paraphrase stability vs budget` 或 `top-k inversion vs budget`

#### 附表

`Appendix Table A1`

- Run-0 oracle calibration：每个 benchmark 的 `1.0x` 定义与成本

`Appendix Table A2`

- Component sweeps：`K`, `r`, `modality split`

`Appendix Table A3`

- Slice 结果：table-heavy、OCR-heavy、visual-grounding、hard reasoning

`Appendix Table A4`

- BRIGHT control results

`Appendix Table A5`

- 如果方法显式建模风险，报告 `ECE` / `Brier score`

### 最简开跑版本

如果你现在只想尽快启动第一轮实验，我建议直接跑这个最小矩阵：

- Benchmark：`ViDoRe V3`, `REAL-MM-RAG (FinSlides + FinReport)`, `MM-BRIGHT (Task 2 + Task 4)`
- 方法：`fixed-fixed`, `adaptive-rerank-only`, `equal-modality`, `proposed`, `oracle`
- 预算：`0.2, 0.4, 0.8, 1.0`
- 指标：`nDCG@10`, `Top-k overlap@10`, `P95 latency`, `ANN probes`, `revealed interaction ratio`
- Slice：`REAL-MM-RAG paraphrase levels`, `ViDoRe bbox hit`, `MM-BRIGHT hard tasks`

这一版已经足够形成：

- 一张主结果表
- 一张 frontier 图
- 一张 robustness / evidence quality 表

也足够判断这个方向是否值得继续扩展到更完整的 closed-loop 论文版本。

## References

- [RLT] [Ranked List Truncation for Large Language Model-based Re-Ranking](https://arxiv.org/abs/2404.18185)
- [GenRT] [List-aware Reranking-Truncation Joint Model for Search and Retrieval-augmented Generation](https://arxiv.org/abs/2402.02764)
- [AcuRank] [AcuRank: Uncertainty-Aware Adaptive Computation for Listwise Reranking](https://openreview.net/forum?id=H918WyPf0s)
- [Col-Bandit] [Col-Bandit: Zero-Shot Query-Time Pruning for Late-Interaction Retrieval](https://arxiv.org/abs/2602.02827)
- [RGS] [Beyond Sequential Reranking: Reranker-Guided Search Improves Reasoning Intensive Retrieval](https://openreview.net/forum?id=Fjw3OqdjTj)
- [IGP] [Less is More for RAG: Information Gain Pruning for Generator-Aligned Reranking and Evidence Selection](https://arxiv.org/abs/2601.17532)
- [SetR] [Shifting from Ranking to Set Selection for Retrieval Augmented Generation](https://aclanthology.org/2025.acl-long.861/)
- [ToolRerank] [ToolRerank: Adaptive and Hierarchy-Aware Reranking for Tool Retrieval](https://aclanthology.org/2024.lrec-main.1413/)
- [ViDoRe V3] [ViDoRe V3 benchmark dataset card](https://huggingface.co/datasets/vidore/vidore_v3_computer_science)
- [REAL-MM-RAG] [REAL-MM-RAG: A Real-World Multi-Modal Retrieval Benchmark](https://aclanthology.org/2025.acl-long.1528/)
- [MM-BRIGHT] [MM-BRIGHT: A Multi-Task Multimodal Benchmark for Reasoning-Intensive Retrieval](https://mm-bright.github.io/)
- [BRIGHT] [BRIGHT: A Realistic and Challenging Benchmark for Reasoning-Intensive Retrieval](https://openreview.net/forum?id=ykuc5q381b)
- [MC-Search] [MC-Search: Evaluating and Enhancing Multimodal Agentic Search with Structured Long Reasoning Chains](https://openreview.net/forum?id=JEGDp1E4OH)
