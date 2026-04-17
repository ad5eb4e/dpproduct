---
name: run-all-max
description: 迭代优化流水线。在 run-all-auto 基础上自动追加最多2轮"优化→对比"循环，直到我方文案明显领先竞品或边际收益收敛。Use when user mentions "迭代优化", "跑到最好", "run all max", or wants the full pipeline with auto-iteration until victory.
disable-model-invocation: false
argument-hint: "<competitor-url> [competitor-url-2] [目标市场] [销售平台]"
---

# 迭代优化流水线（run-all-max）— v2.1 多样本聚合版

## v2.1 升级摘要

本版本（skillsv2.1）相比 skillsv2（v2.0）的核心升级：
- **所有 Compare 环节默认启用多样本聚合模式 N=3**：3 个独立 Agent 并行打分取中位数，解决单样本 ±6 分漂移问题
- **CLEAR_WIN 判定加稳定性校验**：除中位数领先 >=8 外，还要求 >=2/3 样本领先 >=5
- **新增两个停机原因**：LOW_CONFIDENCE（打分不稳定）、MEDIAN_WIN_UNSTABLE（中位数领先但不稳定）
- **三条防护硬约束**：防递归爆炸、防文件覆盖、防老格式误读

## 概述

在 `/run-all-auto` 的基础上，自动执行"优化→对比"循环，最多 2 轮。**所有 Compare 环节默认启用 copy-compare 的多样本聚合模式（aggregation_mode=3）**，用 3 个独立 Agent 并行打分取中位数，降低单次主观判断 + WebFetch 切片差异带来的评分噪声。

停机条件（命中任一即停止迭代）：
1. **CLEAR_WIN**：中位数领先 >= 8 分 **AND** 至少 2/3 样本领先 >= 5 分（稳定性校验）
2. **PLATEAU**：连续 2 轮中位数提分 < 2 分（边际收益收敛）
3. **MAX_ROUNDS**：达到最大迭代轮数 2（兜底封顶）
4. **LOW_CONFIDENCE**：连续 2 轮竞品样本离散度 > 6 分（打分不稳定，建议人工介入）

最终输出 3 个版本（v1/v2/v3）的完整文案 + 聚合对比报告 + 得分追踪表，供用户选择最优版本投放。

## 与 run-all-auto 的区别

| 项目 | `/run-all-auto` | `/run-all-max` |
|------|----------------|----------------|
| 调研 + 文案 v1 + 对比 v1 | ✅ | ✅ |
| 自动迭代 v2 / v3 | ❌ | ✅（最多2轮） |
| 总耗时估算 | ~24 分钟 | ~36 分钟 |
| 输出文件数 | 4 份 | 7~9 份 |
| 适用场景 | 快速试水、测试竞品 | 正式项目、准备投放 |

**旧命令 `/run-all-auto` 不受影响，继续正常工作。**

## 使用方式

```
/run-all-max https://competitor-url.com
/run-all-max https://competitor-url.com/adv https://competitor-url.com/product
/run-all-max https://competitor-url.com/adv https://competitor-url.com/product 美国市场 Facebook
```

参数与 `/run-all-auto` 完全一致，只是命令名不同。

## 输入参数

- **必须**：至少1个竞品URL
- **可选**：第2个竞品URL（Advertorial + 产品页）、目标市场、销售平台、补充说明

---

## 执行流程

### Stage 1-2：复用 run-all-auto 的 Stage 1-2

1. 读取 `skillsv2.1/run-all-auto/SKILL.md` 的 Stage 1-2 指令
2. 按其完整执行：
   - Stage 1：产品调研 → `{产品名}_市场调研报告.txt` + `.html`
   - Stage 2：文案生成 → `{产品名}_Landing_Page_Copy_Final.md`（此为 v1）
3. 将 Stage 2 输出拷贝为 `{产品名}_Copy_v1.md`（保持 run-all-auto 兼容路径不动）

### Stage 3：v1 竞品对比（多样本聚合模式）

⚠️ **不复用 run-all-auto 的 Stage 3（单样本模式）**，改为直接调用 copy-compare 的**多样本聚合模式**。

1. 读取 `skillsv2.1/copy-compare/SKILL.md`
2. spawn **Compare Orchestrator Agent**，传入：
   - 我方文案：`{产品名}_Copy_v1.md`
   - 竞品 URL：用户提供的 URL 列表
   - **aggregation_mode = 3**（关键参数）
3. Orchestrator 按 copy-compare/SKILL.md 的"多样本聚合模式"章节执行：
   - 并行 spawn 3 个独立 Compare sub-agent
   - **严格遵守 copy-compare SKILL.md 中 Step A 的"强制约束"**（防护 1+2）：
     - 不传 aggregation_mode 给 sub-agent
     - 显式指定每个 sub-agent 的输出路径为 `_sample{i}.md`
   - 主 Orchestrator 聚合 3 份结果（中位数 + 离散度 + 共识信号）
4. 产出：
   - 主文件：`{产品名}_对比报告_v1.md`（包含多样本分布表 + 中位数得分 + 置信度，**必含字符串"多样本打分分布"**）
   - 附件：`{产品名}_对比报告_v1_sample1.md` / `_sample2.md` / `_sample3.md`

### Stage 4：决策点 ①（首次停止判定）

#### ⚠️ 4.0 报告格式兼容性检查（防护 3）

在判定前**必须先**检查 `{产品名}_对比报告_v1.md` 的格式：

```
读取 {产品名}_对比报告_v1.md 前 200 行内容
搜索关键字符串 "多样本打分分布"

IF 包含 "多样本打分分布":
    → 新格式（v2.1 多样本）：继续执行下面的 4.1
ELSE:
    → 老单样本格式（可能来自 /run-all-auto 或 skillsv2 v2.0）
    → 输出警告："检测到单样本对比报告，无法进行稳定性校验。
                 建议重跑 Stage 3 以生成多样本报告；
                 当前将降级为单样本阈值判定（旧 v2.0 逻辑）。"
    → 降级逻辑：用 (我方总分 - 竞品总分) >= 8 做判定
    → stop_reason 前缀加 "[LEGACY] " 标识本次未使用稳定性校验
```

#### 4.1 多样本判定（新格式）

读取 `{产品名}_对比报告_v1.md`，提取：
- 我方中位数总分、竞品中位数总分
- 3 个样本各自的我方/竞品得分（用于稳定性判定）
- 置信度等级（高/中/低）

```
lead_median = 我方中位数 - 竞品中位数
samples_leading_5plus = 3 个样本中差值 >= +5 的样本数（0/1/2/3）
competitor_dispersion = max(竞品样本) - min(竞品样本)

IF lead_median >= 8 AND samples_leading_5plus >= 2:
    stop_reason = "CLEAR_WIN"
    → 跳到 Stage 7（已稳定明显领先，无需迭代）
ELSE IF lead_median >= 8 AND samples_leading_5plus < 2:
    stop_reason = "MEDIAN_WIN_UNSTABLE"
    → 进入 Stage 5 继续迭代（中位数领先但不稳定，需要更稳的版本）
ELSE:
    → 进入 Stage 5（开始迭代循环）
```

### Stage 5：迭代循环（最多 2 轮）

初始化变量：
```
iterate_count = 0
max_rounds = 2
last_gain = +infinity
prev_gain = +infinity
current_version = "v1"
```

循环条件（全部满足才继续）：
- `iterate_count < max_rounds`
- `(我方得分 - 竞品得分) < 8`
- NOT `(last_gain < 2 AND prev_gain < 2)`  ← 收敛检测

#### 每轮循环内容：

**Sub-Stage 5.A：优化**

1. 读取 `skillsv2.1/copy-optimize/SKILL.md`
2. spawn **Optimize Agent**，传入：
   - 当前文案文件：`{产品名}_Copy_{current_version}.md`
   - 当前对比报告：`{产品名}_对比报告_{current_version}.md`
   - 目标版本号：`{next_version}`（v1→v2→v3）
3. Agent 按 copy-optimize/SKILL.md 执行完整优化流程
4. 产出：`{产品名}_Copy_{next_version}.md`

**Optimize Agent Prompt 模板**：
```
你是文案优化专家。请读取以下文件作为你的完整优化指令：
{copy-optimize/SKILL.md 路径}

当前文案：{产品名}_Copy_{current_version}.md
对比报告：{产品名}_对比报告_{current_version}.md
目标版本号：{next_version}

严格按 SKILL.md 的 Step 1-5 执行，输出优化后的完整文案到：
{产品名}_Copy_{next_version}.md
```

**Sub-Stage 5.B：验证（多样本聚合模式）**

1. 读取 `skillsv2.1/copy-compare/SKILL.md`
2. spawn **Compare Orchestrator Agent**，**aggregation_mode = 3**（与 Stage 3 同规格）：
   - 我方文案：`{产品名}_Copy_{next_version}.md`
   - 竞品 URL：原始竞品 URL 列表
3. Orchestrator 并行 spawn 3 个独立 Compare sub-agent，**严格执行 copy-compare 中的防护 1+2**（不传 aggregation_mode、强制指定 sample 路径）
4. 产出：
   - 主文件：`{产品名}_对比报告_{next_version}.md`（中位数得分 + 多样本分布表 + 置信度，必含"多样本打分分布"字符串）
   - 附件：`{产品名}_对比报告_{next_version}_sample{1,2,3}.md`
5. 提取本轮中位数得分和稳定性指标，计算 `score_gain = 本轮我方中位数 - 上轮我方中位数`
6. 更新：
   ```
   prev_gain = last_gain
   last_gain = score_gain
   prev_dispersion = last_dispersion
   last_dispersion = competitor_dispersion
   iterate_count++
   current_version = next_version
   ```

**Sub-Stage 5.C：决策点 ②（循环内停止判定）**

按优先级检查 4 条停止条件（任一命中就跳出循环）：

```
lead_median = 本轮我方中位数 - 本轮竞品中位数
samples_leading_5plus = 本轮 3 个样本中差值 >= +5 的样本数

IF lead_median >= 8 AND samples_leading_5plus >= 2:
    stop_reason = "CLEAR_WIN"                # 稳定领先
    break

ELSE IF iterate_count >= max_rounds:
    stop_reason = "MAX_ROUNDS"                # 兜底封顶
    break

ELSE IF last_gain < 2 AND prev_gain < 2:
    stop_reason = "PLATEAU"                   # 边际收益收敛
    break

ELSE IF last_dispersion > 6 AND prev_dispersion > 6:
    stop_reason = "LOW_CONFIDENCE"            # 连续 2 轮打分不稳定
    break

ELSE:
    continue loop
```

### Stage 6：Copy_Final 软链接

将得分最高的版本拷贝到 `{产品名}_Landing_Page_Copy_Final.md`（覆盖 run-all-auto 的原始 Final 文件），保证下游工具引用此文件总能拿到最佳版本。

### Stage 7：输出总结

```
## 迭代优化流水线执行完成（多样本聚合模式 N=3）

### 版本得分追踪（中位数）
| 版本 | 我方(中位数) | 竞品(中位数) | 差值 | 本轮提分 | 3样本分布(我方) | 3样本分布(竞品) | 置信度 |
|------|------------|------------|------|---------|---------------|---------------|-------|
| v1  | X   | X   | ±X | -    | X/X/X | X/X/X | 高/中/低 |
| v2  | X   | X   | ±X | +X   | X/X/X | X/X/X | 高/中/低 |
| v3  | X   | X   | ±X | +X   | X/X/X | X/X/X | 高/中/低 |
| **推荐** | **v{N}** | - | **±X** | - | - | - | ⭐ 最高中位数 |

### 稳定性校验
- v{N} 中 3 个样本领先差值：{X, X, X}
- 领先 >= +5 分的样本数：{0/1/2/3}
- 竞品分数离散度：{X 分}（<= 3 分为高置信度）

### 停机原因
{根据 stop_reason 输出对应说明}
- CLEAR_WIN：✅ 中位数领先 >= 8 AND 至少 2/3 样本领先 >= 5 → 稳定明显领先
- MEDIAN_WIN_UNSTABLE：⚠️ 中位数领先 >= 8 但稳定性不够 → 已继续迭代
- PLATEAU：⏹️ 连续 2 轮中位数提分 < 2 → 边际收益收敛
- MAX_ROUNDS：🔒 达到最大迭代轮数 2 → 兜底停机
- LOW_CONFIDENCE：⚠️ 连续 2 轮竞品样本离散度 > 6 分 → 打分不稳定，建议人工复核
- [LEGACY] ...：⚠️ 检测到 v1 是单样本老格式，已降级为旧 v2.0 判定逻辑

### 生成的文件
| 文件 | 用途 |
|------|------|
| {产品名}_市场调研报告.txt / .html | 调研报告 |
| {产品名}_Copy_v1.md / v2.md / v3.md | 各版本文案 |
| {产品名}_Landing_Page_Copy_Final.md | 推荐版本副本 |
| {产品名}_对比报告_v1.md / v2.md / v3.md | 各版本聚合对比报告（中位数 + 分布） |
| {产品名}_对比报告_v{1,2,3}_sample{1,2,3}.md | 每轮 3 个样本的独立报告（查证用） |

### 推荐使用版本：v{N}
理由：{根据停机原因和得分自动生成}

### 下一步建议
{根据停机原因给出不同建议}
- CLEAR_WIN：v{N} 已稳定明显领先，建议进入 A/B 投放测试阶段。
- PLATEAU：文案微调已达边际收益，用真实投放数据做最终判定。
- MAX_ROUNDS (且仍未稳定领先)：建议手动检查调研报告定位，或继续跑 /copy-optimize 尝试突破。
- LOW_CONFIDENCE：3 次打分差距过大，建议把 aggregation_mode 提到 5 重跑一轮，或人工读 3 份 sample 报告手动判定。
- [LEGACY]：建议重跑一次 Stage 3 拿多样本数据，当前判定可信度仅等同 v2.0。
```

---

## 容错处理

- **Stage 1-3 任一失败**：按 run-all-auto 原有的失败处理逻辑，报错并告知用户在哪个Stage失败
- **优化循环中 Agent 失败**：保留上一版本文件不覆盖，直接跳到 Stage 6/7 输出总结（用已有版本），告知用户失败的是第几轮
- **对比 Agent 失败**：保留已生成的文案版本，跳出循环，输出部分总结
- **无进展情况**：如果 v2/v3 得分反而下降（优化反效果），仍正常记录并输出，让用户看到完整轨迹自行判断
- **用户中断**：按 Ctrl+C 中断后，已生成的文件全部保留，用户可以从任意 vN 手动继续

---

## 设计说明

### 为什么采用多样本聚合（N=3）

实测发现单样本 Compare 模式下，同一个竞品在 3 次独立 Agent 打分中最大漂移达 **6 分**，来源包括：
- Agent 主观判断差异（5 维度权重的个体解读不同）
- WebFetch 抓取切片差异（每次返回的摘要内容可能不完全相同）
- 相对比较的锚定效应（我方分数变化会牵动 Agent 对竞品的隐性重定位）

对 "+8 分 = CLEAR_WIN" 这种阈值决策，±6 分漂移意味着单样本判定可能直接翻车。多样本聚合通过：
- 并行 3 次独立打分 + 取中位数 → 消除单个 Agent 的主观偏差
- 离散度统计 → 显式暴露置信度，低置信度触发停机或人工介入
- 稳定性条件（2/3 样本领先 >= 5） → 防止中位数被一次异常高分拉高

并行执行使墙钟时间与单样本基本相同，仅 token 成本变 3 倍，性价比极高。

### 三条防护硬约束（v2.1 相比 v2.0 的加固）

v2.1 在指令里明文封堵 3 个已知风险：

| 防护 | 场景 | 实施位置 |
|-----|------|---------|
| **防护 1：禁止递归** | Orchestrator 不许把 aggregation_mode 传给 sub-agent，否则会 N² 倍成本或无限递归 | copy-compare SKILL.md Step A 强制约束 1 |
| **防护 2：强制输出路径** | 3 个 sub-agent 若都按 Step 4 默认路径保存会互相覆盖，必须显式指定 `_sample{i}.md` | copy-compare SKILL.md Step A 强制约束 2 |
| **防护 3：老格式降级** | 读到 v2.0 生成的单样本报告时自动降级为旧逻辑 + 加 [LEGACY] 标记 | run-all-max Stage 4.0 格式兼容性检查 |

### 为什么最多 2 轮

基于实测数据：
- v1→v2 平均提分 +10~13（高 ROI）
- v2→v3 平均提分 +4~6（中 ROI）
- v3→v4 边际收益预估 +2~3（低 ROI）

到 v3 后继续迭代，还不如用真实投放数据做判断。因此 2 轮（生成到 v3）是性价比最高的默认值。

### 为什么 "+8 分 + 稳定性校验" 作为 CLEAR_WIN 阈值

5 维度 100 分制单样本噪声 ±3~6 分，多样本中位数能压到 ±1~2 分。+8 分超过 3 倍标准差，判定为"显著领先"而非随机波动。叠加"2/3 样本领先 >= 5"的稳定性条件，确保中位数不是被单次异常值顶上来的，而是跨样本共识。

### 为什么本 skill 做了侵入式修改

原本 run-all-max 是"只加不改，完全复用 run-all-auto"。但多样本聚合无法通过叠加实现——必须在 Compare 阶段就启用，所以 Stage 3 从"复用 run-all-auto Stage 3"改成"独立调用 copy-compare 多样本模式"。Stage 1-2 仍然复用 run-all-auto。这是一个**必要的范围收窄**，不影响 `/run-all-auto` 单独使用时的行为。

### v2.1 与 v2.0 的共存策略

- **v2.0（skillsv2/）** 保留不动：所有行为与之前一致，单样本对比
- **v2.1（skillsv2.1/）** 默认推荐：多样本对比 + 3 防护
- 用户**二选一**安装到 `~/.claude/skills/`，同时装两个会因 skill name 重复冲突
- 如果已经在 v2.0 下跑了一部分（比如生成了老格式 v1 对比报告），切换到 v2.1 继续跑时，防护 3 会自动识别并降级，确保不崩
