# meeting-ingest · 会议转录入库

[中文](#中文) | [English](#english)

---

## 中文

把语音转文字的会议记录，自动纠正错误、对照知识库去重、提炼真正有价值的信息，写入 wiki。

### 它解决什么问题

语音转录的会议记录通常有三个痛点：
1. **转写错误太多**——人名、产品名、术语被转得面目全非
2. **90% 是已知信息**——会上说的东西，wiki 里早就记过了
3. **执行细节混在战略判断里**——操作指引、截止日期、现场情绪充斥全文

meeting-ingest 用一个 7 步管线解决这三个问题：

```mermaid
flowchart LR
    A["📝 转录文件"] --> B["Step 0\n归档"]
    B --> C["Step 1\n纠错 pass"]
    C --> D["Step 2\n搜索 wiki"]
    D --> E["Step 3\n读完整页面"]
    E --> F["Step 4\n三道筛子过滤"]
    F --> G["Step 5\n报摘要"]
    G --> H["Step 6\n写回 wiki"]
    H --> I["Step 7\nLint 检查"]
    
    style A fill:#1F497D,color:#fff
    style I fill:#4BACC6,color:#fff
```

### 核心能力

- **强制纠错**：逐字对照 wiki 中已记录的人名、产品名、术语，把转写错误揪出来
- **全文对比**：搜索 wiki 后必须读完匹配页面的**完整内容**，不能只看搜索片段就判"已知"
- **三道筛子**：
  1. 已知 vs 新 —— wiki 已有的，剔除
  2. 一周测试 —— 一周后还重要吗？
  3. 持久层 vs 执行层 —— 一次 60 分钟会议，真正入库的通常 ≤3 个要点

### 快速开始

```bash
git clone https://github.com/stanley6635/meeting-ingest.git ~/.claude/skills/meeting-ingest
```

然后编辑 `skill.md` 里的**配置**部分，设置你的知识库路径。详见 [SETUP.md](SETUP.md)。

### 依赖

- Python 3（stdlib only，无需 pip install）
- 结构化 wiki 知识库（包含 `people/`、`products/`、`mechanisms/` 等子目录）
- `file-ingest` skill（可选，用于自动归档）

---

## English

A Claude Code / OpenCode skill that processes voice-to-text meeting transcripts into structured wiki knowledge.

### Pipeline

```mermaid
flowchart LR
    A["📝 Transcript"] --> B["Step 0\nArchive"]
    B --> C["Step 1\nError Fix"]
    C --> D["Step 2\nWiki Search"]
    D --> E["Step 3\nRead Full Pages"]
    E --> F["Step 4\nThree Filters"]
    F --> G["Step 5\nSummarize"]
    G --> H["Step 6\nWrite-back"]
    H --> I["Step 7\nLint"]
    
    style A fill:#1F497D,color:#fff
    style I fill:#4BACC6,color:#fff
```

### Key Features

| Step | What | Why |
|------|------|-----|
| 1. Error Correction | Fix ASR mangling of names, products, terms | Garbage in → garbage out |
| 2. Wiki Search | Cross-reference every topic domain | Don't re-record what you already know |
| 3. Full Page Read | Read matched pages completely | Snippet match ≠ already covered |
| 4. Three Filters | Known/new, one-week test, durable vs execution | 90% of meeting content doesn't belong in long-term memory |
| 5-6. Write-back | Structured updates with time layers | Preserve history, don't flatten |
| 7. Lint | Verify source paths and cross-links | No fabricated references |

### Quick Start

```bash
git clone https://github.com/stanley6635/meeting-ingest.git ~/.claude/skills/meeting-ingest
```

Edit the **配置** section in `skill.md` to match your knowledge base paths. See [SETUP.md](SETUP.md) for details.

### Requirements

- Python 3 (stdlib only)
- Structured wiki knowledge base
- `file-ingest` skill (optional)

### License

MIT
