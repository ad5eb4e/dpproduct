# skillsv2.1 — 多样本聚合版流水线

`skillsv2.1` 是 `skillsv2`（v2.0）的加固版本，**完整自包含**，可独立使用。

## v2.1 相比 v2.0 的升级

1. **多样本聚合打分（核心）**：每次竞品对比并行跑 3 个独立 Agent，取中位数 + 离散度 + 共识信号，把单样本 ±6 分漂移压到 ±1~2 分
2. **CLEAR_WIN 加稳定性校验**：不再只看中位数差值，还要求 >=2/3 样本领先 >=5 分
3. **新增 2 个停机原因**：`LOW_CONFIDENCE`（打分不稳定）、`MEDIAN_WIN_UNSTABLE`（中位数赢但不稳）
4. **三条防护硬约束**：防递归爆炸、防 3 个 sub-agent 文件互相覆盖、防误读老版本单样本报告

## v2.0 / v2.1 如何选择

| 场景 | 推荐版本 |
|------|---------|
| 日常跑一次试水，对分数精度不敏感 | `skillsv2`（v2.0）成本低 1/3 |
| 正式项目，要用分数做 CLEAR_WIN 阈值决策 | `skillsv2.1`（v2.1）分数更可靠 |
| 只用 `/copy-compare`、`/landing-page`、`/product-research` 等单个 skill | 两个版本行为完全一致，随便选 |

⚠️ **二选一**：两个版本里的 skill name 相同（`copy-compare`、`run-all-max` 等），同时装两个会冲突。安装前任选其一放到 `~/.claude/skills/`。

主要升级基线（继承自 v2）：新增"对比→优化→对比"的自动迭代循环能力。

## 与 skills 的区别

| 对比项 | skills | skillsv2 |
|--------|--------|----------|
| 模块数量 | 5 个 | 7 个（新增 2 个） |
| 自动迭代 | ❌ | ✅ |
| 向后兼容 | 原命令 | 原命令 + 新命令 |

## 文件清单

### 继承自 skills（行为一致）
| 目录 | 命令 | 作用 |
|------|------|------|
| `product-research/` | `/product-research` | 产品深度调研（4阶段） |
| `landing-page/` | `/landing-page` | 落地页文案生成（3轮Writer-Reviewer循环） |
| `copy-compare/` | `/copy-compare` | 我方 vs 竞品文案对比打分 |
| `run-all-auto/` | `/run-all-auto` | 全自动流水线（调研→文案→对比） |
| `run-all-guided/` | `/run-all-guided` | 半自动流水线（每阶段暂停确认） |

### 新增（v2 核心能力）
| 目录 | 命令 | 作用 |
|------|------|------|
| `copy-optimize/` | `/copy-optimize` | 单轮文案优化器（基于对比报告） |
| `run-all-max/` | `/run-all-max` | 迭代流水线（自动跑到领先竞品或收敛） |

## 怎么用

### 新手推荐：直接用 `/run-all-max`

```
/run-all-max <竞品URL> [第2个竞品URL] [目标市场] [销售平台]
```

示例：
```
/run-all-max https://competitor.com/lander 美国市场 Facebook
```

跑完后拿到 3 份文案（v1/v2/v3）+ 3 份对比报告 + 1 份推荐版本软拷贝。

### 进阶：单独调用 copy-optimize

如果已有文案和对比报告，想手动迭代一次：

```
/copy-optimize ./MyCopy_v1.md ./MyReport_v1.md v2
```

### 老习惯：继续用 skills 的命令

所有原 skills 的命令在 skillsv2 中行为完全一致（修改仅限内部路径引用），可以无感切换。

## 停止条件

`/run-all-max` 会在以下任一条件命中时自动停机（所有打分基于 **多样本聚合模式 N=3** 的中位数）：

1. **CLEAR_WIN**：中位数领先 >= 8 分 **AND** 至少 2/3 样本领先 >= 5 分（稳定性校验）→ 停
2. **PLATEAU**：连续 2 轮中位数提分 < 2 分（边际收益收敛）→ 停
3. **MAX_ROUNDS**：已执行 2 轮迭代 → 停
4. **LOW_CONFIDENCE**：连续 2 轮竞品样本离散度 > 6 分（打分不稳定，建议人工介入）→ 停

## 多样本聚合模式（v2.1 核心升级）

单样本 Compare 实测发现同一个竞品的得分在 3 次独立打分中最大漂移 **6 分**，来自 Agent 主观差异 + WebFetch 抓取切片差异 + 锚定效应。

`/run-all-max` 现在每次 Compare 都：
- 并行 spawn 3 个独立 Compare sub-agent（墙钟时间与单次相同）
- 主 Orchestrator 聚合 3 份结果取**中位数**
- 计算**离散度**显式标注置信度（高/中/低）
- 提取**共识短板/亮点**（在 >= 2/3 样本里被提到的才算共识）

这能把 ±6 分的单样本漂移压到 ±1~2 分的中位数偏差，让 +8 分阈值判定真正可靠。详见 `copy-compare/SKILL.md` 的"多样本聚合模式"章节。

## 三条防护（v2.1 加固）

| 防护 | 问题 | 位置 |
|-----|------|------|
| 禁止递归 | Orchestrator 传 aggregation_mode 给 sub-agent → N² 爆炸 | `copy-compare/SKILL.md` Step A 约束 1 |
| 强制输出路径 | 3 个 sub-agent 都按默认路径写 → 互相覆盖 | `copy-compare/SKILL.md` Step A 约束 2 |
| 老格式降级 | v1 是 v2.0 单样本报告 → 读不到多样本字段 | `run-all-max/SKILL.md` Stage 4.0 检查 |

## 安装

把整个 `skillsv2/` 文件夹放到你的 Claude skills 目录下（如 `~/.claude/skills/` 或项目本地 `.claude/skills/`），即可使用。

## 文件关系图

```
skillsv2/
├── product-research/     ◄─┐
├── landing-page/         ◄─┤
├── copy-compare/         ◄─┼─ 被 run-all-auto / run-all-max 调用
├── copy-optimize/        ◄─┤
├── run-all-auto/         ◄─┘  （调研 + 文案 + 对比）
├── run-all-guided/            （半自动版）
└── run-all-max/               （run-all-auto + 迭代循环）
```
