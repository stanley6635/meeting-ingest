---
name: meeting-ingest
description: Use when the user provides a voice-to-text meeting transcript or call recording transcript and says "处理一下""按规则处理""帮我整理这个通话/会议"。Handles ASR transcription correction, wiki cross-reference, and handoff to wiki-builder for structured write-back. Key symptom: the file is a .txt/.md transcript of spoken dialogue with known voice-to-text inaccuracies in speakers, names, products, organizations, terms, or abbreviations.
---

# Meeting Ingest Skill

## 配置

| 变量 | 说明 | 示例 |
|------|------|------|
| `$MEETINGS_DIR` | 会议转录文件存放目录 | `raw/meetings/` |
| `$WIKI_DIR` | wiki 页面目录 | `wiki/` |
| `$INDEX_FILE` | wiki 顶层导航文件 | `index.md` |

**依赖**：本 skill 依赖 `file-ingest` skill 做文件归档，依赖 `pro-workflow:wiki-builder` 做自动写回。wiki 搜索通过 agentmemory `memory_smart_search` 完成。

## 触发条件

用户提供语音转录文档（.txt / .md），且说了以下任一：
- "处理一下这个通话记录/会议记录"
- "这个转录帮我处理一下"
- "按规则处理这个对话"
- "帮我整理下这个会议"

**核心信号**：文件内容是口语对话转录，天然存在语音识别错误。

## 不触发的情况

- 文件是公众号推文 → 用 `article-ingest`
- 文件是 PDF/学术文献 → 用 `academic-briefing`
- 文件是图片 → 用 `image-ocr`
- 用户只是让我读一下，不要求入库

## 执行流程

以下步骤**严格按顺序执行**，不可跳步。Step 1-3 由 meeting-ingest 执行（纠错 + wiki 对照），Step 4 交给 wiki-builder 统一写回。

### Step 0: 归档文件

委托 `file-ingest` skill 将文件移入 `$MEETINGS_DIR`，标准化命名。

### Step 1: 纠错 pass（强制执行）

语音转录文档的纠错是强制性前置步骤。纠错分两轮：先做说话人实名化，再做内容纠错。

#### 1A: 说话人实名化

将 ASR 自动分配的 `发言人1/2/3...` 替换为参会人真实姓名。

**做法**：

1. **建立姓名基准**：
   - 用 agentmemory `memory_smart_search` 搜索 wiki `people/` 下已记录的参会人员，提取标准姓名。
   - 如存在与会议直接相关的补充数据表（CSV 等），其标准名可作为交叉验证，但最终以 wiki 记录和会议上下文为准。
   - 从转录中识别主持人点名、汇报顺序、互相称呼和固定角色。
   - 注意：ASR 的说话人编号**跨段不绑定人**，同一编号可能在不同时段指代不同人。
2. **逐段映射**：按会议结构，为每个可确认段落建立 `(时间段, 发言人编号, 标准姓名, 依据)`。
3. **处理 ASR 分裂**：同一人可能被 ASR 拆成多个编号，需按上下文合并到同一标准姓名。
4. **保留不确定项**：无法确认身份的，不强行实名，保留原编号并在纠错表标注“待确认”。

#### 1B: 内容纠错

**做法**：

1. **提取关键实体**：从原文中提取所有人名、产品名、组织名、医院名、术语、缩写、日期、价格、数量和关键项目名称。
2. **对照 wiki 纠错**：
   - 用 agentmemory `memory_smart_search` 搜索每个实体。
   - 找到 wiki 已有记录 → 采用 wiki 的标准写法。
   - 找不到 → 保留原转录，标注“未确认”。
3. **只纠正可证明的 ASR 错误**：纠错依据必须来自 wiki、项目文件、会议上下文、参会人名单、补充表格、原始音视频或 Stanley 明确确认。不要凭“看起来更顺”改写。
4. **高风险字段必须进入确认表**：说话人实名、人名、医院名、产品/型号、组织名、监管/招采术语、价格、数量、日期、承诺量/报量/挂网等会影响后续判断的字段，必须列入纠错表。

#### 1C: 确认阀门（强制）

在改写转录文件前，必须先向 Stanley 展示纠错表并等待确认。

```markdown
| 位置 | 转录原文 | 建议改为 | 类型 | 依据 | 置信度 | 处理 |
|------|----------|----------|------|------|--------|------|
| L42 / 12:31 | ... | ... | 人名/产品/说话人/术语 | wiki / 上下文 / 用户确认 | high/medium/low | 改 / 保留 / 待确认 |
```

**确认规则**：

- Stanley 确认“改”的项，才能写回转录文件。
- Stanley 要求“保留”的项，不改写，但可在后续 wiki 写回中标注“转录待确认”。
- 低置信度或无依据项不得自动替换。
- 不允许跨文件全局替换；每次替换必须限定在当前会议和已确认上下文内。

#### 1D: 回写转录文件

经 Stanley 确认后，将已确认的纠错直接修改进 `$MEETINGS_DIR` 下的会议转录文件。

**TARS 例外规则**：`raw/meetings/` 中由 ASR 生成的会议纪要/逐字稿可以被纠错改写，因为未纠正的转录错误会污染后续 wiki 判断。此例外只适用于语音转录生成的会议文本，不适用于 raw/ 下其他原始来源。

回写后保留一个简短“已确认纠错表”段落或相邻记录，说明本次改动依据；无法确认的内容不改。

### Step 2: 全文搜索 wiki

针对原文中提到的**每个主题域**（人、产品、项目、机制），使用 agentmemory `memory_smart_search` 搜索。

覆盖所有主题域，不遗漏。搜索结果不足时读 `$INDEX_FILE` 获取全貌。

### Step 3: 读 wiki 页面（完整读，不是看 snippet）

**必须读完**搜索结果中匹配的 wiki 页面的**完整内容**，不能只靠搜索 snippet 做判断。

snippet 匹配 ≠ 页面已记录，必须读全文才能真正确认“已有 vs 新”。

### Step 4: 写回（交给 wiki-builder）

调用 `pro-workflow:wiki-builder`，传入：
- 纠错后的转录文件（已在 `$MEETINGS_DIR`）
- wiki 搜索结果和已读页面

写入规则由 wiki-builder 统一管理，按 CONVENTIONS.md 规范执行：更新已有页面、必要时新建、交叉链接、时间分层保留。写回完成后展示实际改动清单。

## 常见错误类别

| 类别 | 模式 | 检测方法 |
|------|------|----------|
| **说话人编号匿名** | ASR 分配 `发言人1/2/3...`，且编号不稳定 | 对照参会人、点名顺序、会议结构、wiki people/ 和上下文逐段映射 |
| **人名误转** | 中文同音/近音错字、英文名音译、姓/名拆分错误 | 提取所有疑似人名 → wiki 搜索 → 对照标准姓名 |
| **产品名/术语误转** | 品牌名、产品型号、专业缩写被转成常见词或近音词 | 提取专有名词 → 对照 wiki products/mechanisms/projects/ 和项目文件 |
| **医院名/组织名误转** | 简称、地区名、机构名被转成同音或近音词 | 对照 wiki、项目文件、参会上下文和补充表格 |
| **数字/时间/金额误转** | 日期、价格、数量、比例、承诺量等被识别错误 | 结合上下文、表格、邮件、会议材料交叉验证；无法验证则待确认 |
| **以搜索片段替代完整阅读** | 看到搜索 snippet 匹配就判定“已有” | 强制 Step 3：读完整页面 |

## 与其他 Skill 的关系

```
用户提供转录文件
  → file-ingest（Step 0：归档）
  → meeting-ingest（Step 1-3：纠错 + wiki 对照）
  → wiki-builder（Step 4：统一写回）
```
