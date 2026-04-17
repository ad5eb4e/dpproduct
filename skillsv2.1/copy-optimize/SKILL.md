---
name: copy-optimize
description: 单轮落地页文案优化器。基于 copy-compare 生成的对比报告，按"补短板+抄优点"策略自动修改文案，输出优化版本。Use when user mentions "优化文案", "根据对比改", "optimize copy", or after completing copy-compare and wanting to iterate.
disable-model-invocation: false
argument-hint: "<current-copy-file> <compare-report-file> [version-tag]"
---

# 单轮文案优化器

## 概述

读取 copy-compare 生成的对比报告，针对报告中的"补短板+抄优点"建议对当前文案进行定向优化，输出下一个版本的文案。

这个 skill 的存在目的：把"根据对比结果改文案"这个操作**固化为可重用流程**，而不是每次手动组装 prompt。

## 使用方式

```
/copy-optimize ./MaxErect_Copy_v1.md ./MaxErect_对比报告_v1.md v2
/copy-optimize ./MaxErect_Copy_v2.md ./MaxErect_对比报告_v2.md v3
```

## 输入参数

- **必须**：当前文案文件路径（如 `MaxErect_Copy_v1.md`）
- **必须**：最新对比报告文件路径（如 `MaxErect_对比报告_v1.md`）
- **可选**：版本标签（默认根据文件名自增，v1→v2→v3...）

## 前置条件

1. 当前文案必须是 `/landing-page` 或 `/copy-optimize` 生成的完整11 Section格式
2. 对比报告必须是 `/copy-compare` 生成的5维度打分格式
3. 对比报告中必须包含"第四部分：我方版本优化建议"

---

## 执行流程

### Step 1：读取输入与规划

1. 读取当前文案文件
2. 读取对比报告文件，提取以下信息：
   - 当前得分（我方 vs 竞品）
   - 5个维度的失分分布
   - 第四部分：补短板建议
   - 第四部分：抄优点建议
3. 规划优化清单（从报告抽取最多 5 条高价值建议，按预估提分排序）

### Step 2：spawn Writer Agent 执行优化

使用 Agent 工具 spawn 一个独立的 **Writer Agent**，传入以下 prompt：

```
你是顶级直接回应文案专家（Writer Agent）。你的任务是根据对比报告的建议
对现有落地页文案进行定向优化。

## 输入文件

**当前文案**：请读取：{当前文案文件路径}
**对比报告**：请读取：{对比报告文件路径}
**写作规范**：请读取：C:/Users/adao/productcr/skillsv2.1/landing-page/WRITER.md

## 优化执行要求

1. 从对比报告的"第四部分：我方版本优化建议"中提取所有改写建议：
   - 补短板部分的具体改写方案
   - 抄优点部分的嫁接建议
2. 逐条执行这些修改，精准定位到文案中对应位置
3. 未被指出的部分保持不变
4. 严格遵守 WRITER.md 的所有字数/格式/反模板约束
5. 保留之前版本的所有优化（如这是 v3，必须保留 v2 的改动基础上继续）

## 关键禁令（来自 WRITER.md）

- 严禁副作用暗示、退款原因说明、产品局限性、负面用户体验
- 严禁"skeptical"、"honestly"、"I won't lie" 等套路化表达
- 所有字数必须严格遵守 WRITER.md 中的上下限
- SEO部分纯文本无加粗
- 关键数据和情感词汇必须加粗

## 输出

1. 将完整优化后文案保存为新文件：{输出文件路径}
2. 在文件末尾附优化日志：
   - 每条改动：位置 + v_current 原内容 → v_next 新内容 + 新字数
   - 标明每条改动对应对比报告中的哪条建议
```

### Step 3：验证输出文件

1. 确认输出文件已生成
2. 文件不为空且包含完整11 Section结构
3. 如验证失败，保留当前版本不覆盖，报错退出

### Step 4：同步 Copy_Final 指针

将输出文件拷贝一份到 `{产品名}_Landing_Page_Copy_Final.md`（或更新该软拷贝），使下游工具总能引用最新版本。

### Step 5：返回摘要

输出以下摘要到对话：

```
## 优化完成：v_current → v_next

### 执行的优化（共 X 条）
1. [位置] 改写摘要
2. [位置] 改写摘要
...

### 输出文件
- {产品名}_Landing_Page_Copy_Final_{version-tag}.md
- {产品名}_Landing_Page_Copy_Final.md（已更新为最新版本）

### 下一步
建议运行 /copy-compare 验证 v_next 的效果得分。
```

---

## 容错处理

- **对比报告格式异常**：如果无法从第四部分抽取建议，使用 LLM 理解能力兜底解析整份报告
- **建议冲突**：如两条建议修改同一字段，只执行提分预估更高的那条
- **Writer Agent 失败**：保留当前版本不覆盖原文件，报错并告知用户失败原因
- **优化后字数超标**：Writer Agent 内部自检，如超标则自动精简至合规范围
- **版本号冲突**：如目标文件已存在，在文件名加时间戳后缀（避免覆盖历史版本）

---

## 设计说明

本 skill 是 `run-all-max` 迭代流水线的核心组件之一。也可以独立使用：
- 手动跑完 `/run-all-auto` 后，如想继续优化 → 用 `/copy-optimize`
- 对特定文案做有针对性的迭代，而不想重跑整个流水线
