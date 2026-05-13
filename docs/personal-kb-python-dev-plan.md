# 个人 AI 知识库（Python 自研）详细开发方案

> 适用场景：个人积累的知识、常用命令、问题排查思路、设计与开发思路等；**单机、本地优先**，资料以 **Markdown / 文本** 为主（PDF 等可作为后续扩展）。

---

## 1. 项目定位与目标

### 1.1 定位

面向个人的「可检索 + 可对话」知识系统：把碎片化的命令、踩坑记录、设计思路写成条目或文档，系统负责 **分块、建索引、检索**，并 **可选** 用大模型做总结与追问。

### 1.2 非目标（第一期刻意不做）

- 多租户、复杂权限、团队协作（可预留接口，第一期不实现）。
- 企业级 OCR、复杂版式 PDF（可作为二期插件）。

### 1.3 成功标准（建议写入项目 README）

- 新增或修改一条知识后，在约定时间内可被检索到。
- 检索结果能展示 **来源标题、文件路径、片段位置**（或章节路径）。
- 若启用 LLM：**无检索命中** 时明确拒答或提示「知识库中未找到相关内容」，避免胡编。

---

## 2. 用户故事与功能范围

### 2.1 MVP（建议 3～5 个迭代内完成）

| 能力 | 说明 |
|------|------|
| 知识写入 | 支持 Markdown 文件入库；支持单条笔记通过 API/CLI 追加 |
| 解析分块 | 按标题、空行或固定长度分块；可配置 chunk 大小与 overlap |
| 向量化 | 调用一种嵌入模型（本地或 API），生成向量 |
| 持久化 | 元数据与向量落盘；重启后无需全量重建（除非更换嵌入模型） |
| 检索 | 向量 Top-K；可选简单关键词过滤 |
| 展示 | CLI 或极简本地 Web（二选一即可）：列出命中片段与来源 |
| 配置 | YAML/TOML：模型、路径、chunk 参数、是否启用 LLM |

### 2.2 二期

- **混合检索**：向量 + BM25（如 `rank_bm25` 或 SQLite FTS5）。
- **Rerank**：用小交叉编码器对 Top-K 重排（可选云端 API）。
- **多格式**：PDF、HTML；代码块感知分块。
- **对话**：多轮对话 + 引用约束的 prompt 模板。
- **同步**：监视目录，自动增量索引。

### 2.3 三期（按需）

- 图谱、标签体系、时间线视图。
- 与 Obsidian、Git 工作流结合（仅索引 vault 子集等）。

---

## 3. 技术栈建议（Python）

### 3.1 语言与工程化

- **Python**：建议 3.11+。
- **包管理**：`uv` 或 `poetry`。
- **CLI**：`typer` 或 `click`。
- **配置**：`pydantic-settings` + YAML。
- **日志**：`structlog` 或标准库 `logging`。
- **测试**：`pytest` + 少量「黄金」检索用例。

### 3.2 存储组合（推荐个人场景）

**方案 A（简单、依赖少）**

- 原文/Markdown：文件系统（按日期或 UUID）或 **SQLite** 表 `documents` / `chunks`。
- 向量：**Chroma** 持久化目录，或 **sqlite-vec**（与 SQLite 同库）。
- 全文（二期）：**SQLite FTS5** 与向量并行，做混合检索。

**方案 B（向量与查询能力更强）**

- **LanceDB** 或本地 **Qdrant**（Docker），在数据量增大时考虑。

**MVP 建议**：SQLite（元数据）+ Chroma 持久化 **或** 单用 LanceDB 存 metadata + vector（二选一，避免早期双写复杂度过高）。

### 3.3 嵌入与生成（可插拔）

- **嵌入**：`sentence-transformers`（本地）；或 OpenAI 兼容 API（官方 `openai` SDK，`base_url` 可指向国产/自建网关）。
- **生成（可选）**：同一套 SDK 调 Chat；无 API 时 MVP 仅做「检索 + 展示片段」亦可交付价值。

**抽象接口（设计层面）**

```text
Embedder.embed(texts: list[str]) -> list[list[float]]
LLM.chat(messages) -> str          # 可选
Chunker.split(doc) -> list[Chunk]
Retriever.search(query, k) -> list[Hit]
```

---

## 4. 系统架构（逻辑分层）

```text
┌─────────────────────────────────────────┐
│  UI 层：CLI / FastAPI + 简单前端         │
└─────────────────────────────────────────┘
                      │
┌─────────────────────────────────────────┐
│  应用层：入库、检索、问答编排（Orchestrator） │
└─────────────────────────────────────────┘
                      │
      ┌───────────────┼───────────────┐
      ▼               ▼               ▼
   Ingest         Index/Store      Retrieve
  （解析分块）    （向量+元数据）   （Top-K+过滤）
```

### 4.1 建议包结构

```text
kb/
  config.py              # 配置加载
  models.py              # Document, Chunk, Hit 等数据类
  ingest/
    markdown.py          # MD 解析、YAML front matter
    chunker.py           # 分块策略
  index/
    store.py             # 抽象：upsert / delete
    chroma_backend.py    # 或 lance_backend.py
  embed/
    local_st.py          # sentence-transformers
    openai_api.py        # 兼容 OpenAI 的嵌入/对话 API
  retrieve/
    vector_search.py
    hybrid.py            # 二期混合检索
  rag/
    prompts.py           # 系统提示、引用格式
    pipeline.py          # retrieve -> 拼上下文 -> llm
  cli.py                 # kb ingest / kb search / kb ask
  api.py                 # 可选 FastAPI
```

---

## 5. 数据模型

### 5.1 Document（文档）

| 字段 | 说明 |
|------|------|
| `id` | 唯一标识 |
| `source_path` / `source_uri` | 来源路径或 URI |
| `title` | 标题 |
| `content_raw` | 原始全文（或仅存路径，按需） |
| `created_at` / `updated_at` | 时间戳 |
| `tags` | 可选，JSON 或关联表 |
| `content_hash` | SHA256，用于变更检测 |

### 5.2 Chunk（块）

| 字段 | 说明 |
|------|------|
| `id` | 唯一标识 |
| `document_id` | 所属文档 |
| `text` | 块文本 |
| `start_offset` / `end_offset` | 在原文中的偏移（可选） |
| `heading_path` | 章节路径，如 `["部署","Docker"]` |
| `embedding_id` | 与向量库主键对应 |

### 5.3 Hit（检索命中）

| 字段 | 说明 |
|------|------|
| `chunk_id` | 块 ID |
| `score` | 相似度分数 |
| `snippet` | 展示用片段 |
| `document_title` | 文档标题 |
| `source_path` | 来源路径 |
| `heading_path` | 章节路径 |

### 5.4 变更与一致性

- 文档级 `content_hash` 变化：删除该文档旧 chunks → 重新分块与嵌入 → upsert。
- 文件删除：级联删除向量与元数据。

---

## 6. 核心流程设计

### 6.1 入库（Ingest）

1. 扫描配置的根目录（例如 `~/notes/kb/**/*.md`）或读取单文件。
2. 解析 Markdown：提取 YAML front matter（tags、title 等）。
3. `Chunker`：优先按二级标题切分；过长段落再按字符/token 窗口切分，带 overlap。
4. 批量调用 `Embedder`（注意 API 速率与 batch size；本地模型注意显存/内存）。
5. 写入向量库与元数据表；记录 `last_indexed_at`。

### 6.2 检索（Retrieve）

1. 将查询文本转为向量。
2. 向量检索 Top-K（K 建议 8～20，个人库可先取 10）。
3. （二期）BM25 取 Top-M，融合或 rerank。
4. 返回 `Hit` 列表，**必须包含可追溯字段**。

### 6.3 问答（RAG，可选）

1. 用同一查询检索 → 拼接上下文（限制总 token，例如 3000～6000）。
2. 系统提示词要求示例：
   - 仅依据上下文回答；
   - 重要结论后附 `[来源: 标题 > 章节]`；
   - 上下文不足时明确说明。
3. 若做 Web：流式输出可用 SSE，与业务编排分离。

---

## 7. 接口设计（CLI 优先）

第一期以 CLI 为主，降低前端成本。命令名仅为设计目标示例：

```bash
kb init --data-dir ~/.local/share/mykb
kb ingest path/to/notes              # 增量索引目录
kb ingest --file ./snippet.md        # 单文件
kb search "docker compose 重启"      # 仅检索
kb ask "docker compose 如何滚动更新？"  # RAG（需配置 LLM）
kb doctor                            # 检查模型、路径、向量维度等
```

可选 **FastAPI**：`POST /ingest`、`POST /search`、`POST /chat`，便于后续替换任意 UI。

---

## 8. 配置项清单（建议在 `config.yaml` 中固化）

| 配置项 | 说明 |
|--------|------|
| `data_dir` | 数据根目录 |
| `vault_paths` | 待索引的 Markdown 根路径列表 |
| `chunk_size` / `chunk_overlap` | 分块参数 |
| `embedding` | `provider`（`local` \| `openai_compatible`）、`model`、`api_base`；`api_key` 从环境变量读取 |
| `llm` | 同上，可选 |
| `vector_backend` | `chroma` \| `lance` 等 |
| `top_k` | 默认检索条数 |

**安全**：密钥只来自环境变量或本地密钥链，**不要**写入仓库。

---

## 9. 非功能需求

| 类别 | 建议做法 |
|------|----------|
| 性能 | 批量嵌入；本地模型自动检测 GPU/CPU；大目录索引用异步或任务队列 |
| 可靠 | 索引进度与日志；失败 chunk 写入 `errors.jsonl` |
| 备份 | 定期打包 `data_dir`；若笔记本身用 Git 管理，向量库仍建议单独备份 |
| 隐私 | 默认全本地；若使用云端 API，在 README 中说明哪些文本会出境 |

---

## 10. 测试策略

- **单元测试**：`Chunker` 对固定 Markdown 的切块边界；`content_hash` 变更检测逻辑。
- **集成测试**：内存或临时目录向量库 + 假 Embedder（固定维度）跑通 ingest → search。
- **回归**：3～5 条真实笔记构造「查询 → 某标题必须出现在 Top-3」的断言。

---

## 11. 开发里程碑（任务分解）

| 阶段 | 内容 |
|------|------|
| **M1：骨架** | 项目结构、配置、`Document`/`Chunk` 模型、假 embedder（固定维度） |
| **M2：索引闭环** | Markdown 读取 → 分块 → 真实嵌入 → 持久化 → 重启后可 search |
| **M3：检索体验** | `kb search`：分数、标题、路径、片段预览；支持按 tag 过滤（若有 front matter） |
| **M4：RAG（可选）** | `kb ask`、prompt 模板、拒答行为、引用格式 |
| **M5：打磨** | 增量索引、`kb doctor`、错误处理、README「5 分钟上手」 |

---

## 12. 风险与对策

| 风险 | 对策 |
|------|------|
| 更换嵌入模型 | 维度变化需「重建索引」子命令，清空旧 collection |
| 中英文混合笔记 | 选用多语言 embedding 模型，并在文档中写明推荐型号 |
| 笔记中大量命令行 | 分块时识别 fenced code block（` ``` `），避免从中间切断 |

---

## 13. 扩展路线（与内容类型对应）

- **命令速查**：使用 tag（如 `#cmd`）+ 检索；后续可增加「命令模板」结构化字段。
- **问题排查思路**：鼓励固定模板（现象 / 假设 / 验证 / 结论），仍以 Markdown 存储即可。
- **开发思路、架构长文**：标题优先分块 + 略大的 `chunk_size`，减少上下文被切碎。

---

## 14. 交付物清单（自检）

- [ ] 可安装的 Python 包或完整的 `pyproject.toml`
- [ ] `config.example.yaml`（不含密钥）
- [ ] `README`：安装步骤、嵌入配置、首次 `ingest` 与 `search` 示例
- [ ] 数据目录说明与备份建议
- [ ] 若提供 Web/API：可选 `docker-compose`（按需）

---

## 15. 文档修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0 | 2026-05-13 | 初稿：个人本地知识库 Python 自研方案 |

---

*本文档为开发规划说明，不包含具体业务代码实现。*
