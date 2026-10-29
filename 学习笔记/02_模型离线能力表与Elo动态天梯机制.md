# vLLM Semantic Router 学习笔记 02：模型离线能力表与 Elo 动态天梯机制

> **源码事实源**：
> - `src/semantic-router/pkg/selection/elo.go` (RouteLLM & Bradley-Terry 胜率推算)
> - `src/semantic-router/pkg/selection/lookuptable/builder.go` (从历史调用回放中统计 QualityGap)
> - `src/semantic-router/pkg/selection/lookuptable/table.go` (查找表接口与缓存)
> **学术依据**：RouteLLM 论文 (arXiv:2406.18665) 及 LMSYS Chatbot Arena 评测天梯

---

## 一、核心问题：各模型能力差距（QualityGap）从哪里来？自己打分能准吗？

在上一篇笔记中，我们学到了模型切换必须计算：
$$\text{新模型最终分} = \text{基础分} + \text{真实能力差距 (QualityGap)} - \text{换乘代价} - \text{缓存报废扣分} \dots$$

师弟提出了一个非常关键的敏锐问题：
> **“这个能力差距（QualityGap）如果是人工凭感觉打分，那能打准吗？业界有没有公开测评可以借鉴？”**

在 `vLLM Semantic Router` 源码中，答案非常清晰：**它绝不依赖人工拍脑袋瞎打分！**
它的能力分由**三大支柱**构成：

---

## 二、三大支柱来源（源码 1:1 对照）

### 支柱 1：引入权威学术界 Elo 天梯（RouteLLM & Chatbot Arena）
源码 `elo.go` 第 96 行直接注明：
> `// Based on RouteLLM paper (arXiv:2406.18665) using Bradley-Terry model`

1. **什么是 Elo 积分？**
   - 就像国际象棋或王者荣耀排位赛，每个模型都有一个基准分（默认 1500 分，区间 1000~2000 分）。
   - 行业有公开的大模型竞技场（如 **LMSYS Chatbot Arena**、**HumanEval 代码榜**、**GSM8K 数学榜**），上万真实用户盲测投票打出来的客观战力榜。
2. **Bradley-Terry 胜率预测公式（源码第 319 行）**：
   $$P(\text{模型 A 战胜 模型 B}) = \frac{10^{\text{Rating}_A / 400}}{10^{\text{Rating}_A / 400} + 10^{\text{Rating}_B / 400}}$$
   - 系统能以极高的数学严密性，预测出“在当前代码任务下，DeepSeek-V3 战胜 16B 小模型的概率是 87.5%”。
   - 这就完全消除了人工打分的主观偏差！

---

### 支柱 2：从真实流量回放中自动推导（Router Replay Builder）
源码 `lookuptable/builder.go` 第 78~84 行：
```go
// quality_gap: group records by Category (task family). For each category,
// compute the average ConfidenceScore per SelectedModel. The quality gap from
// model A to model B is avgScore(B) - avgScore(A).
```

1. **真实数据说话**：
   - 每次用户使用 AI 时，系统在后台日志里记录：任务类型（代码/闲聊/数理）、用了哪个模型、这次回答的置信度评分（ConfidenceScore）。
2. **批量自动聚合计算（PopulateFromRecords）**：
   - 定期（如每天或每周）把历史日志聚合：
   $$\text{QualityGap(模型 A } \to \text{ 模型 B)} = \overline{\text{Score}}_B - \overline{\text{Score}}_A$$
   - 跑过几千次真实业务调用后，系统自己就会从真实业务场景中“算出”真正的能力差值，比实验室跑分更贴合我们的真实用户！

---

### 支柱 3：新模型冷启动配置兜底（Config Override）
源码 `table.go`：
- 当我们训练了一个全新的自建模型、或者刚接了一个新模型时，因为既没有公开测评、也没有历史调用日志；
- 此时允许管理员在配置文件里手动给一个**初始及格分**（0.0 ~ 1.0）；
- 只要系统开始接单，随着真实调用和 Elo 胜负反馈，初始分会被真实的动态数据逐步校正覆盖。

---

## 三、换乘代价与会话预测的自动推导

`builder.go` 不仅推导能力差距，还自动从历史中推导两项神级数据：

1. **真实换乘网络损耗（Handoff Penalty，源码第 85 行）**：
   - 统计同一个用户在同一个 30 分钟会话内，从模型 A 切到模型 B 时**真实多花的时间与成本差距**；
   - 用 EMA（指数移动平均，$\alpha = 0.2$）平滑计算，测出最真实的跨机网络换乘代价。
2. **会话剩余轮次预测（Remaining Turn Prior，源码第 92 行）**：
   - 统计特定任务类型（比如写长代码通常要聊 6 轮，查天气通常只有 1 轮）；
   - 提前预知“这个任务用户大概率还要聊几轮”，从而决定是“咬牙换大模型”还是“继续凑合用小模型”。

---

## 四、学习总结与沉淀启示

1. **不要手工拍脑袋给模型定死分**：借鉴公开测评榜（LMSYS / HuggingFace OpenLLM）作为初始战力值；
2. **闭环日志自进化**：业务运行产生的数据是最好的裁判，通过历史回放差值自动更新各模型的能力差距表；
3. **能力差与换乘代价天平**：只有当公开能力差距明显，且能覆盖真实换乘开销时，智能选型才真正下达换车指令。
