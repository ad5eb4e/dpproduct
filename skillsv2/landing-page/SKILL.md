---
name: landing-page
description: 高转化落地页文案生成（内置Writer-Reviewer 3轮质检循环）。基于product-research调研报告，自动生成完整11个Section落地页文案（英文），由独立Reviewer Agent审查3轮后输出终稿。Use when user mentions "落地页", "文案", "landing page", "copy", or after completing product research.
disable-model-invocation: false
argument-hint: "[report-file-path] or paste research report"
---

# 落地页文案生成系统（Writer-Reviewer 3轮循环）

## 概述

基于 product-research 调研报告，通过 Writer-Reviewer 角色分离的3轮质检循环，生成高转化落地页文案（英文）。

## 使用方式

在完成 `/product-research` 后运行：
```
/landing-page
/landing-page C:\Users\adao\dpproduct\产品名_市场调研报告.html
```

## 输入来源

1. 如果用户指定了文件路径 → 读取该文件作为调研报告
2. 如果当前对话中已有调研报告输出 → 直接使用
3. 如果都没有 → 提示用户先运行 /product-research 或提供调研报告

**⚠️ 文件格式要求**：优先使用 `.txt` 纯文本版调研报告（由 `/product-research` 自动生成）。如果只有 `.html` 版本，先用 Python 转成 `.txt` 再传给 Writer Agent，因为单行HTML文件 Agent 无法高效读取。

---

## 核心架构：Writer-Reviewer 3轮循环

本skill采用**角色分离**的质检机制：Writer Agent 和 Reviewer Agent 是独立的角色，各自有独立的指令集，交替执行3轮，确保文案质量。

### 指令文件

| 文件 | 角色 | 用途 |
|------|------|------|
| `WRITER.md` | Writer Agent | 完整的文案创作指令（11 Section + Effect Data + Before/After） |
| `REVIEWER.md` | Reviewer Agent | 完整的审核标准（字数检查 + 格式验证 + 6维度打分 + 反模板化） |

---

## 执行流程

### Stage 1：Writer 生成初稿

1. 读取本skill目录下的 `WRITER.md` 文件
2. 读取用户提供的调研报告
3. 严格按照 WRITER.md 中的指令，完整执行：
   - 第一阶段：深度报告解析与核心内容提取（6个执行步骤 + 智能适配）
   - 第二阶段：核心创作原则与灵活应用
   - 第三阶段：完整11个Section创作 + Effect Data + Before/After
   - 第四阶段：Writer自带的创作质量保证（A/B/C/D四级自检）
4. 将初稿保存为工作目录下的 `{产品名}_Landing_Page_Copy_Draft.md` 文件

### Stage 2：Reviewer 第1轮审查

1. 使用 Agent 工具 spawn 一个独立的 **Reviewer Agent**
2. 在 Agent prompt 中指示它：
   - 读取本skill目录下的 `REVIEWER.md` 文件，作为审查标准
   - 读取 Stage 1 输出的初稿文件
   - 执行 REVIEWER.md 中的完整审查流程：
     - Step 0.5：全量字数/格式检查（逐字段）
     - Step 3：6维度逐句打分（吸引力/情绪共鸣/自然度/内心触动/对话感/号召力）
     - 反模板化检查
     - 元素完整性检查
   - **不需要询问用户意见**（跳过 REVIEWER.md 中的 Step 5 用户交互环节）
   - 直接输出：问题清单 + 每个问题的具体修改指令
3. 收集 Reviewer Agent 返回的审查结果

### Stage 3：Writer 第1轮修改

1. 使用 Agent 工具 spawn 一个独立的 **Writer Agent**
2. 在 Agent prompt 中指示它：
   - 读取 `WRITER.md` 作为写作标准参考
   - 读取当前稿件
   - 读取 Reviewer 的审查反馈
   - 针对每个问题逐一修改，保持未被指出问题的部分不变
   - 输出修改后的完整文案（二稿）
3. 将二稿保存覆盖草稿文件

### Stage 4：Reviewer 第2轮审查

重复 Stage 2 的流程，审查二稿。
- 重点关注：上一轮指出的问题是否已修复
- 继续检查是否有新的问题

### Stage 5：Writer 第2轮修改

重复 Stage 3 的流程，根据第2轮审查反馈修改。输出三稿。

### Stage 6：Reviewer 终审

重复 Stage 2 的流程，对三稿做终审。
- 输出：终审报告（通过/剩余问题标注）
- 如果仍有 ❌ 严重问题，在终审报告中标注，但不再循环

### Stage 7：输出终稿

1. 将终稿以规范格式输出到对话中
2. 同时将完整文案写入工作目录的 HTML 文件：`{产品名}_Landing_Page_Copy.html`
3. 附上终审报告摘要：
   - ✅ 通过的检查项数量
   - ⚠️ 仍有轻微问题的项（如有）
   - 3轮循环的改进轨迹

---

## Agent Prompt 模板

### Reviewer Agent Prompt 模板

```
你是一个独立的文案审核专家（Reviewer）。你的任务是审查落地页文案的质量。

**审核标准**：请读取以下文件作为你的完整审核指令：
{REVIEWER.md 的完整路径}

**待审文案**：请读取以下文件：
{草稿文件路径}

**执行要求**：
1. 严格按照审核指令中的 Step 0.5 执行全量字数/格式检查
2. 对每个Section执行6维度打分
3. 执行反模板化检查
4. 跳过"询问用户意见"步骤，直接输出审查结果
5. 输出格式：
   - 【字数/格式检查结果】（按REVIEWER.md中的格式）
   - 【6维度打分结果】（8分以下的逐条列出）
   - 【具体修改指令】（每个问题给出：位置 + 当前内容 + 修改建议）

当前是第 {N} 轮审查（共3轮）。
{如果是第2/3轮}：上一轮指出的问题清单如下，请重点验证是否已修复：
{上轮问题清单}
```

### Writer Agent Prompt 模板

```
你是一个独立的文案写手（Writer）。你的任务是根据审核反馈修改落地页文案。

**写作标准**：请读取以下文件作为你的写作规范参考：
{WRITER.md 的完整路径}

**当前稿件**：请读取以下文件：
{当前稿件路径}

**审核反馈**：以下是 Reviewer 提出的问题清单和修改指令：
{Reviewer 的审查结果}

**执行要求**：
1. 逐条处理每个审核问题
2. 修改时严格遵循 WRITER.md 中的创作规则
3. 未被指出问题的部分保持不变
4. 输出完整的修改后文案
5. 在文案末尾附上修改日志：列出每个修改点（位置 + 旧内容 → 新内容）
```

---

## 交互后续模式

终稿输出后，用户可以：
- "优化Section X" → 对该Section单独跑一轮 Writer-Reviewer
- "全部再优化一轮" → 再跑一轮完整的 Reviewer→Writer 循环
- "导出HTML" → 重新生成格式化HTML文件
- `/copy-compare` → 拿终稿和竞品对比

修改时只改指定部分，保持其他Section不变。
