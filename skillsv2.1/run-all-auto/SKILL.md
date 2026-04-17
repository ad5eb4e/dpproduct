---
name: run-all-auto
description: 全自动产品营销流水线。输入竞品URL，一键跑完：调研→文案（含3轮Writer-Reviewer循环）→竞品对比，中间不暂停。Use when user mentions "全自动", "一键跑", "run all auto", or wants the full pipeline without interruption.
disable-model-invocation: false
argument-hint: "<competitor-url> [competitor-url-2] [目标市场] [销售平台]"
---

# 全自动产品营销流水线

## 概述

输入竞品落地页URL，一键串联3个skill，全程无暂停自动执行：
1. `/product-research` → 分析竞品页面，生成产品调研报告
2. `/landing-page` → 基于调研报告，生成我方落地页文案（Writer-Reviewer 3轮循环）
3. `/copy-compare` → 将我方生成的文案与竞品原始页面对比

## 核心逻辑

**竞品URL = 调研对象 = 最终对比对象。** 我们卖的是同类/同款产品，所以：
- 从竞品页面提取产品信息（成分、价格、卖点、评价等）
- 基于提取的信息生成我们自己的文案
- 最后拿我们的文案和竞品原始页面做销售力对比

## 使用方式

```
/run-all-auto https://competitor-url.com
/run-all-auto https://competitor-url.com/adv https://competitor-url.com/product
/run-all-auto https://competitor-url.com/adv https://competitor-url.com/product 澳洲市场 Facebook
```

## 输入参数

- **必须**：至少1个竞品URL
- **可选**：第2个竞品URL（竞品有2个页面时，如Advertorial页+产品页，全部提供）、目标市场、销售平台、补充说明
- **注意**：竞品可能有2个页面（广告文章页 + 产品详情页），构成完整销售漏斗，如果有请全部提供

## ⚠️ 全自动模式说明

- **跳过所有暂停点**：product-research 的2个确认暂停（产品信息确认、Phase 1方向确认）自动跳过
- **适用场景**：你对产品和目标市场已经很熟悉，不需要中途调整方向
- **风险**：如果调研方向偏了，后面的文案和对比都会跟着偏

---

## 执行流程

### Stage 1：产品调研

1. 读取 `skillsv2.1/product-research/SKILL.md` 获取调研指令
2. spawn **Research Agent**，传入：
   - 竞品URL（用户提供的所有竞品页面URL）
   - 目标市场、销售平台等补充信息
   - **特殊指令**：跳过所有暂停点，自动连续完成Phase 1-4
3. Agent 完成后，确认以下文件已生成：
   - `{产品名}_市场调研报告.txt`（**必须**，后续流程依赖此文件）
   - `{产品名}_市场调研报告.html`（可选，方便人阅读）
4. 从Agent返回中提取产品名，用于后续文件命名

**Research Agent Prompt 模板**：
```
你是产品市场调研专家。请读取以下文件作为调研指令：
{product-research/SKILL.md 路径}

竞品URL：{用户提供的URL}
目标市场：{用户指定或默认}
销售平台：{用户指定或默认}

⚠️ 全自动模式：跳过所有暂停点，全程自动连续输出，直到完成所有4个Phase。

最终输出：
1. 纯文本报告 → {产品名}_市场调研报告.txt（必须，流水线核心文件）
2. HTML报告 → {产品名}_市场调研报告.html（可选，方便人阅读）
```

### Stage 2：落地页文案

1. 读取 `skillsv2.1/landing-page/SKILL.md` 获取编排指令
2. 读取 `skillsv2.1/landing-page/WRITER.md` 和 `skillsv2.1/landing-page/REVIEWER.md` 的路径
3. 按 landing-page/SKILL.md 的7个Stage执行：
   - Stage 1: spawn Writer Agent → 读WRITER.md + TXT报告 → 生成初稿
   - Stage 2: spawn Reviewer Agent → 读REVIEWER.md + 初稿 → 第1轮审查
   - Stage 3: spawn Writer Agent → 根据反馈修改 → 二稿
   - Stage 4: spawn Reviewer Agent → 第2轮审查
   - Stage 5: spawn Writer Agent → 修改 → 三稿
   - Stage 6: spawn Reviewer Agent → 终审
   - Stage 7: 输出终稿
4. 确认终稿文件已生成：`{产品名}_Landing_Page_Copy_Final.md`

### Stage 3：竞品对比

1. 读取 `skillsv2.1/copy-compare/SKILL.md` 获取对比指令
2. spawn **Compare Agent**，传入：
   - 我方文案：`{产品名}_Landing_Page_Copy_Final.md`
   - 竞品URL：用户提供的竞品URL（可以多个）
   - 对比标准：copy-compare/SKILL.md 中的直接回应文案专家5维度打分
3. Agent 完成后，确认对比报告已生成：`{产品名}_竞品对比报告.md`

**Compare Agent Prompt 模板**：
```
你是直接回应文案专家。请读取以下文件作为对比标准：
{copy-compare/SKILL.md 路径}

版本A（我方文案）：{终稿文件路径}
版本B（竞品）：{竞品URL列表}

抓取竞品页面后，严格按照SKILL.md的5维度100分制打分。
输出完整对比报告，保存到：{产品名}_竞品对比报告.md
```

### Stage 4：输出总结

全部完成后，输出流水线执行总结：

```
## 全自动流水线执行完成

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

---

## 容错处理

- **HTML无TXT**：如果调研只输出了HTML，自动用Python转换为TXT后再进入Stage 2
- **竞品URL 403**：按 copy-compare/SKILL.md 的3级容错路径（WebFetch → 浏览器 → 提示粘贴）
- **Agent超时/失败**：报错并告知用户在哪个Stage失败，用户可以手动从该Stage继续
- **文件未生成**：每个Stage结束后检查输出文件是否存在，不存在则报错
