---
name: run-all-guided
description: 半自动产品营销流水线（含确认暂停）。输入竞品URL，分3阶段执行：调研→文案→竞品对比，每阶段结束后暂停等用户确认再继续。Use when user mentions "半自动", "分步跑", "run guided", or wants to review output between stages.
disable-model-invocation: false
argument-hint: "<competitor-url> [competitor-url-2] [目标市场] [销售平台]"
---

# 半自动产品营销流水线（含确认暂停）

## 概述

输入竞品落地页URL，串联3个skill分阶段执行，每阶段结束后暂停等用户确认：
1. `/product-research` → 分析竞品页面，生成调研报告 → **⏸️ 用户确认**
2. `/landing-page` → 基于调研报告，生成我方文案（Writer-Reviewer 3轮循环）→ **⏸️ 用户确认**
3. `/copy-compare` → 将我方文案与竞品原始页面对比 → 完成

## 核心逻辑

**竞品URL = 调研对象 = 最终对比对象。** 我们卖的是同类/同款产品，所以：
- 从竞品页面提取产品信息（成分、价格、卖点、评价等）
- 基于提取的信息生成我们自己的文案
- 最后拿我们的文案和竞品原始页面做销售力对比

## 使用方式

```
/run-all-guided https://competitor-url.com
/run-all-guided https://competitor-url.com/adv https://competitor-url.com/product
/run-all-guided https://competitor-url.com/adv https://competitor-url.com/product 澳洲市场 Facebook
```

## 输入参数

- **必须**：至少1个竞品URL
- **可选**：第2个竞品URL（竞品有2个页面时，如Advertorial页+产品页，全部提供）、目标市场、销售平台、补充说明
- **注意**：竞品可能有2个页面（广告文章页 + 产品详情页），构成完整销售漏斗，如果有请全部提供

## ⚠️ 半自动模式说明

- **保留所有暂停点**：调研的2个确认暂停 + 阶段间2个确认暂停 = 共4个暂停点
- **适用场景**：新产品、不熟悉的市场、需要中途调整方向
- **优势**：可以在调研后调整人群/痛点方向，再进入文案阶段

---

## 执行流程

### Stage 1：产品调研

1. 读取 `skills/product-research/SKILL.md` 获取调研指令
2. spawn **Research Agent**，传入：
   - 产品URL（用户提供的所有产品相关URL）
   - 目标市场、销售平台等补充信息
   - **保留暂停点**：按 product-research/SKILL.md 原始设计执行
3. Agent 执行调研流程：
   - Step 0：产品信息提取 → **⏸️ 暂停1：向用户展示提取结果，等待确认**
   - Step 1：Phase 1 市场格局 → **⏸️ 暂停2：向用户展示人群定位、竞品选择、痛点方向，等待确认**
   - Step 2-4：Phase 2/3/4 连续输出
4. 确认以下文件已生成：
   - `{产品名}_市场调研报告.txt`（**必须**，后续流程依赖此文件）
   - `{产品名}_市场调研报告.html`（可选，方便人阅读）

### ⏸️ 暂停3：调研完成确认

向用户展示调研报告摘要，询问：

```
调研报告已完成，文件已保存。

📋 报告摘要：
- 目标人群：[摘要]
- TOP 3痛点：[摘要]
- 核心差异化：[摘要]
- 对标竞品：[摘要]

请确认：
A. 满意，继续生成文案 → 进入Stage 2
B. 需要调整某个方向 → 告诉我调整什么
C. 停在这里，后续手动跑 → 结束
```

等待用户回复后再继续。

### Stage 2：落地页文案

1. 读取 `skills/landing-page/SKILL.md` 获取编排指令
2. 按 landing-page/SKILL.md 的7个Stage执行完整的Writer-Reviewer 3轮循环：
   - Stage 1: spawn Writer Agent → 生成初稿
   - Stage 2-6: 3轮 Reviewer↔Writer 循环
   - Stage 7: 输出终稿
3. 确认终稿文件已生成：`{产品名}_Landing_Page_Copy_Final.md`

### ⏸️ 暂停4：文案完成确认

向用户展示文案概要和3轮审改轨迹，询问：

```
文案已完成3轮Writer-Reviewer循环，终稿已保存。

📋 审改轨迹：
Round 1: X个问题 → Round 2: X个问题 → Round 3: X个问题

📋 终稿包含：
- 11个Section + Effect Data + Before/After
- 共X行

请确认：
A. 满意，继续跑竞品对比 → 进入Stage 3
B. 某个Section需要调整 → 告诉我哪个Section要改什么
C. 再跑一轮Writer-Reviewer → 额外优化
D. 停在这里，后续手动跑 → 结束
```

等待用户回复后再继续。

### Stage 3：竞品对比

1. 读取 `skills/copy-compare/SKILL.md` 获取对比指令
2. spawn **Compare Agent**，传入：
   - 我方文案：`{产品名}_Landing_Page_Copy_Final.md`
   - 竞品URL：用户提供的竞品URL（可以多个）
3. Agent 完成后，确认对比报告已生成：`{产品名}_竞品对比报告.md`

### Stage 4：输出总结

全部完成后，输出流水线执行总结：

```
## 半自动流水线执行完成

### 生成的文件
| 文件 | 用途 |
|------|------|
| {产品名}_市场调研报告.txt | 调研报告（核心文件，Agent可读） |
| {产品名}_市场调研报告.html | 调研报告（可选，人看的格式化版） |
| {产品名}_Landing_Page_Copy_Final.md | 终稿文案（经3轮审改） |
| {产品名}_竞品对比报告.md | 竞品对比（100分制打分） |

### 3轮审改轨迹
Round 1: X个问题 → Round 2: X个问题 → Round 3: X个问题

### 竞品对比得分
版本A（我方）: X/100 | 版本B（竞品）: X/100
```

最后询问：
```
还需要什么？
A. 根据竞品对比结果优化文案
B. 针对某个Section深入优化
C. 导出为HTML格式
D. 全部完成
```

---

## 容错处理

- **HTML无TXT**：如果调研只输出了HTML，自动用Python转换为TXT后再进入Stage 2
- **竞品URL 403**：按 copy-compare/SKILL.md 的3级容错路径（WebFetch → 浏览器 → 提示粘贴）
- **Agent超时/失败**：报错并告知用户在哪个Stage失败，用户可以手动从该Stage继续
- **文件未生成**：每个Stage结束后检查输出文件是否存在，不存在则报错
- **用户要求调整**：在任何暂停点，用户可以要求修改后重新执行当前Stage
