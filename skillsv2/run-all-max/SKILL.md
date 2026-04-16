---
name: run-all-max
description: 迭代优化流水线。在 run-all-auto 基础上自动追加最多2轮"优化→对比"循环，直到我方文案明显领先竞品或边际收益收敛。Use when user mentions "迭代优化", "跑到最好", "run all max", or wants the full pipeline with auto-iteration until victory.
disable-model-invocation: false
argument-hint: "<competitor-url> [competitor-url-2] [目标市场] [销售平台]"
---

# 迭代优化流水线（run-all-max）

## 概述

在 `/run-all-auto` 的基础上，自动执行"优化→对比"循环，最多 2 轮，直到命中任一停止条件：
1. 我方领先竞品 +8 分以上（明显领先）
2. 连续 2 轮提分 < 2 分（边际收益收敛）
3. 达到最大迭代轮数 2（兜底封顶）

最终输出 3 个版本（v1/v2/v3）的完整文案 + 对比报告 + 得分追踪表，供用户选择最优版本投放。

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

### Stage 1-3：复用 run-all-auto 的完整流程

1. 读取 `skills/run-all-auto/SKILL.md` 的 Stage 1-3 指令
2. 按其完整执行：
   - Stage 1：产品调研 → `{产品名}_市场调研报告.txt` + `.html`
   - Stage 2：文案生成 → `{产品名}_Landing_Page_Copy_Final.md`（此为 v1）
   - Stage 3：竞品对比 → `{产品名}_竞品对比报告.md`（此为对比_v1）

3. **规范化文件命名**：
   - 将 Stage 2 输出拷贝为 `{产品名}_Copy_v1.md`
   - 将 Stage 3 输出拷贝为 `{产品名}_对比报告_v1.md`
   - 原文件保留不动（保持 run-all-auto 的向后兼容）

### Stage 4：决策点 ①（首次停止判定）

读取 `{产品名}_对比报告_v1.md`，提取我方得分和竞品得分：

```
IF (我方得分 - 竞品得分) >= 8:
    → 跳到 Stage 7（已明显领先，无需迭代）
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

1. 读取 `skills/copy-optimize/SKILL.md`（注意：位于 skillsv2 目录或同级 skills 目录）
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

**Sub-Stage 5.B：验证**

1. 读取 `skills/copy-compare/SKILL.md`（同 run-all-auto 的 Stage 3）
2. spawn **Compare Agent**，对比新版本文案 vs 竞品：
   - 我方文案：`{产品名}_Copy_{next_version}.md`
   - 竞品URL：原始竞品URL列表
3. 产出：`{产品名}_对比报告_{next_version}.md`
4. 提取本轮得分，计算 `score_gain = new_score - prev_score`
5. 更新：
   ```
   prev_gain = last_gain
   last_gain = score_gain
   iterate_count++
   current_version = next_version
   ```

**Sub-Stage 5.C：决策点 ②（循环内停止判定）**

检查 3 条停止条件（任一命中就跳出循环）：

```
IF (我方得分 - 竞品得分) >= 8:
    stop_reason = "CLEAR_WIN"
    break
ELSE IF iterate_count >= max_rounds:
    stop_reason = "MAX_ROUNDS"
    break
ELSE IF last_gain < 2 AND prev_gain < 2:
    stop_reason = "PLATEAU"
    break
ELSE:
    continue loop
```

### Stage 6：Copy_Final 软链接

将得分最高的版本拷贝到 `{产品名}_Landing_Page_Copy_Final.md`（覆盖 run-all-auto 的原始 Final 文件），保证下游工具引用此文件总能拿到最佳版本。

### Stage 7：输出总结

```
## 迭代优化流水线执行完成

### 版本得分追踪
| 版本 | 得分 | vs 竞品 | 本轮提分 | 状态 |
|------|-----|--------|---------|------|
| v1  | X   | ±X     | -       | 初稿 |
| v2  | X   | ±X     | +X      | 1轮优化后 |
| v3  | X   | ±X     | +X      | 2轮优化后 |
| **推荐** | **v{N}** | **±X** | - | ⭐ 最高得分 |

### 停机原因
{根据 stop_reason 输出对应说明}
- CLEAR_WIN：✅ 我方 X 分 > 竞品 X 分 + 8 分 → 明显领先
- PLATEAU：⏹️ 连续 2 轮提分 < 2 分 → 边际收益收敛
- MAX_ROUNDS：🔒 达到最大迭代轮数 2 → 兜底停机

### 生成的文件
| 文件 | 用途 |
|------|------|
| {产品名}_市场调研报告.txt / .html | 调研报告 |
| {产品名}_Copy_v1.md / v2.md / v3.md | 各版本文案 |
| {产品名}_Landing_Page_Copy_Final.md | 推荐版本副本 |
| {产品名}_对比报告_v1.md / v2.md / v3.md | 各版本对比报告 |

### 推荐使用版本：v{N}
理由：{根据停机原因和得分自动生成}

### 下一步建议
{根据停机原因给出不同建议}
- CLEAR_WIN：v{N} 已明显领先竞品，建议进入 A/B 测试阶段。
- PLATEAU：文案微调已达边际收益，用真实投放数据做最终判定。
- MAX_ROUNDS (且仍未领先)：建议手动检查调研报告定位，或继续跑 /copy-optimize 尝试突破。
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

### 为什么是"只加不改"

本 skill 不修改 `run-all-auto`、`landing-page`、`copy-compare` 中的任何一个。这保证：
- 用户继续用 `/run-all-auto` 行为完全不变
- 如果本 skill 出问题，直接不用即可，不影响其他工作流
- 代码路径清晰：`run-all-max` 只是调用了其他 skill 的完整流程，不做侵入性修改

### 为什么最多 2 轮

基于实测数据：
- v1→v2 平均提分 +10~13（高 ROI）
- v2→v3 平均提分 +4~6（中 ROI）
- v3→v4 边际收益预估 +2~3（低 ROI）

到 v3 后继续迭代，还不如用真实投放数据做判断。因此 2 轮（生成到 v3）是性价比最高的默认值。

### 为什么 "+8 分" 作为明显领先阈值

评分系统本身有 ±2~3 分的噪声（不同 Reviewer 的主观差异），+8 分超过 3 倍标准差，可以较为可靠地判定为"显著领先"而非随机波动。
