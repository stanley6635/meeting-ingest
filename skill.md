---
name: meeting-ingest
description: Use when the user provides a voice-to-text meeting transcript or call recording transcript and says "处理一下""按规则处理""帮我整理这个通话/会议"。Handles transcription error correction, wiki cross-reference, net new identification, and structured write-back. Key symptom: the file is a .txt transcript of spoken dialogue with known voice-to-text inaccuracies (wrong names, products, terms).
---

# Meeting Ingest Skill

## 配置

使用前，根据你的知识库结构设置以下路径。路径可为绝对路径或相对于工作区根目录的相对路径。

| 变量 | 说明 | 示例 |
|------|------|------|
| `$MEETINGS_DIR` | 会议转录文件存放目录 | `raw/meetings/` |
| `$WIKI_DIR` | wiki 页面目录 | `wiki/` |
| `$WIKI_SEARCH` | wiki 搜索脚本路径 | `scripts/wiki_search.py` |
| `$WIKI_LINT` | wiki 结构检查脚本路径 | `scripts/wiki_lint.py` |
| `$INDEX_FILE` | wiki 顶层导航文件 | `index.md` |
| `$LOG_FILE` | 操作日志文件 | `log.md` |

**依赖**：本 skill 假设存在一个结构化的 wiki 知识库（包含 `people/`、`products/`、`mechanisms/`、`projects/`、`judgments/` 等子目录），以及对应的搜索和 lint 脚本。另依赖 `file-ingest` skill 做文件归档（如未安装，需手动将转录文件放入 `$MEETINGS_DIR`）。

## 触发条件

用户提供语音转录文档（.txt / .md），且说了以下任一：
- "处理一下这个通话记录/会议记录"
- "这个转录帮我处理一下"
- "按规则处理这个对话"
- "帮我整理下这个会议"

**核心信号**：文件内容是口语对话转录，天然存在语音识别错误。

## 不触发的情况

- 文件是公众号推文 → 用 `article-ingest`
- 文件是 PDF/学术文献 → 用 `academic-briefing` 或 `pdf-ocr`
- 文件是图片 → 用 `image-ocr`
- 用户只是让我读一下，不要求入库

## 执行流程

以下步骤**严格按顺序执行**，不可跳步。

### Step 0: 归档文件

委托 `file-ingest` skill 将文件移入 `$MEETINGS_DIR`，标准化命名。

如未安装 `file-ingest`，手动将文件移入 `$MEETINGS_DIR`，按 `YYYY-MM-DD_简短描述.ext` 格式命名。

### Step 1: 纠错 pass（必须先做）

语音转录文档的纠错是**强制性前置步骤**——不做纠错就不能进入 net new 判断。

**做法**：

1. **提取关键实体**：从原文中提取所有人名、产品名、组织名、术语、缩写
2. **对照 wiki 纠错**：
   - 用 `$WIKI_SEARCH` 搜索每个实体
   - 找到 wiki 已有记录 → 采用 wiki 的正确写法
   - 找不到 → 标注"未在 wiki 中出现，保留原转录"
3. **特别注意**：
   - 中文名同音/近音错字（竺向佳→朱项家、张顺华→张润华）
   - 英文名/缩写误转（Subha→舒曼/舒巴、Puresee→PUC、ESCRS→ESCLS）
   - 产品名拼写（PY-60AD→PY60AD、AF-1→AFone）
   - 组织名简称（五官科→EENT/Fudan Eye & ENT）
4. **输出纠错表**：

```markdown
| 转录原文 | 应为 | 依据 |
|---------|------|------|
| 朱向佳 | 竺向佳 | wiki people/ 已记录 Zhu Xiangjia |
| 舒巴 | Subha | wiki 多人页面已记录 |
| PUC | Puresee | 产品名纠正 |
```

5. **用户确认**纠错表的准确性。

**不确认不能进入 Step 2。**

6. **回写纠错至转录文件**：用户确认后，将每条纠错直接修改进原始转录文件。不改的错误会在后续分析中持续传播。修改方式：逐条 `replaceAll` 替换原转录文本中的错误写法为正确写法。修改后标注文件 mod time 不变更说明（不破坏原始时间戳语义）。

### Step 2: 全文搜索 wiki

针对原文中提到的**每个主题域**（人、产品、项目、机制），执行 wiki 搜索：

```bash
python3 $WIKI_SEARCH "<query>" --top 5
```

覆盖所有主题域，不遗漏。搜索结果不足时读 `$INDEX_FILE` 获取全貌。

### Step 3: 读 wiki 页面（完整读，不是看 snippet）

**必须读完**搜索结果中匹配的 wiki 页面的**完整内容**，不能只靠搜索 snippet 做判断。

snippet 匹配≠页面已记录，必须读全文才能真正确认"已有 vs 新"。

### Step 4: 净新增判断（三道筛子）

逐条对照原文内容，用三道筛子过滤：

**筛子 1 — 已知 vs 新**：wiki 已明确记录的 → 剔除。不确定 → 标注"待确认"。

**筛子 2 — 一周测试**：一周后还重要吗？执行细节、现场情绪、操作指引 → 不入库。

**筛子 3 — 持久层 vs 执行层**：会议中 90% 的内容是执行协调。只有持久决策、战略判断、机制变化才入库。通常一次 60 分钟会议，真正入库的不超过 3 个要点。

**转录来源标注**：所有基于转录的信息标注"转录来源，待确认"。不确定的地方直接问用户，不猜。

### Step 5: 报摘要 + 写回计划

输出结构化的写回计划：

```markdown
## Net New 摘要

### 战略级（入库 judgment / 项目主页面）
- 要点 1 — 为什么重要
- 要点 2

### 信息级（入库 meeting log / tracker）
- 要点 3
- 要点 4

### 不入库
- 要点 5 — 原因（执行细节/已知信息/一周后不重要）

## 写回计划

| 页面 | 写入内容 |
|------|---------|
| $WIKI_DIR/people/X.md | ... |
| $WIKI_DIR/projects/Y.md | ... |
```

等用户确认后执行写回。

### Step 6: 写回

1. 更新目标页面的 frontmatter（`updated` 日期 + `sources` 新增转录文件路径）
2. 写入内容时保留时间分层、标注"转录来源，待确认"
3. 交叉链接相关页面
4. 有新增页面时更新 `$INDEX_FILE`
5. 追加 `$LOG_FILE`

### Step 7: Lint

```bash
python3 $WIKI_LINT $WIKI_DIR --check-sources
```

确保无 fabricated source paths。

## 常见错误类别

以下类别描述的是**模式**而非具体案例。每次转录产生的具体错误不同，但都落入这些类别之一。

| 类别 | 模式 | 检测方法 |
|------|------|----------|
| **人名误转** | 中文同音/近音错字、英文名被转成中文发音近似字、姓 vs 名拆分错误 | 提取所有人名 → `$WIKI_SEARCH` 搜索 → 对照 wiki 正确写法 |
| **产品名/术语误转** | 缩写被拆散、专业术语被转成常见词、品牌名拼写变形 | 提取所有专有名词 → 对照 `$WIKI_DIR/products/` 和 `$WIKI_DIR/mechanisms/` |
| **以搜索片段替代完整阅读** | 看到搜索 snippet 匹配就判定"已有"，但完整页面可能记录的是不同方面 | 强制 Step 3：读匹配页面的**完整内容** |
| **跳过筛子直接膨胀 net new** | 未经一周测试 + 持久层判断，把大量执行细节和已知信息标为"新" | 强制 Step 4 三道筛子**按顺序**执行 |
| **执行层内容混入持久层** | 操作指引（怎么填系统、截止日期）、个人八卦、现场情绪被当成 wiki 素材 | 筛子 3：区分持久决策 vs 执行协调。一次会议通常 ≤3 个要点入库 |

**核心原则**：skill 不预判具体会错什么——它只强制**纠错→搜索→读全文→筛子**的序列，让流程本身堵住漏洞，不依赖对特定错误的记忆。

## 与其他 Skill 的关系

```
用户提供转录文件
  → file-ingest（Step 0：归档 to $MEETINGS_DIR）
  → meeting-ingest（Step 1-7：内容处理）
```

本 skill 假设 `file-ingest` 已完成归档，或文件已在 `$MEETINGS_DIR` 中。
