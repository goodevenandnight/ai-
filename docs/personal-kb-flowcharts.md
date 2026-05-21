# 个人 AI 知识库 — 流程图

本文档与 `personal-kb-python-dev-plan.md` 第 4～6 节对应，使用 [Mermaid](https://mermaid.js.org/) 绘制，可在 GitHub、VS Code（插件）或任意支持 Mermaid 的 Markdown 预览中渲染。

---

## 1. 逻辑分层（概览）

```mermaid
flowchart TB
    subgraph UI["UI 层"]
        CLI[CLI]
        WEB[FastAPI + 简单前端]
    end

    subgraph APP["应用层"]
        ORC[编排：入库 / 检索 / 问答]
    end

    subgraph CORE["核心能力"]
        ING[Ingest 解析分块]
        IDX[Index/Store 向量 + 元数据]
        RET[Retrieve Top-K + 过滤]
    end

    UI --> ORC
    ORC --> ING
    ORC --> RET
    ING --> IDX
    RET --> IDX
```

---

## 2. 入库（Ingest）

```mermaid
flowchart TD
    A[开始：扫描根目录或单文件] --> B[解析 Markdown]
    B --> C[提取 YAML front matter<br/>tags / title 等]
    C --> D[Chunker 分块]
    D --> E{块过长？}
    E -->|是| F[按字符或 token 窗口切分<br/>带 overlap]
    E -->|否| G[批量调用 Embedder]
    F --> G
    G --> H[写入向量库 + 元数据表]
    H --> I[记录 last_indexed_at]
    I --> J[结束]
```

**说明**：分块策略为「优先按二级标题切分；过长段落再按窗口切分」（见开发方案 6.1）。

---

## 3. 检索（Retrieve）

```mermaid
flowchart TD
    A[开始：用户查询文本] --> B[查询文本向量化]
    B --> C[向量检索 Top-K]
    C --> D{二期：混合检索？}
    D -->|否| F[组装 Hit 列表]
    D -->|是| E[BM25 Top-M + 融合或 rerank]
    E --> F
    F --> G[每条 Hit 含可追溯字段<br/>标题 / 路径 / 章节路径等]
    G --> H[结束]
```

---

## 4. 问答（RAG，可选）

```mermaid
flowchart TD
    A[开始：用户问题] --> B[同查询执行检索]
    B --> C{有可用上下文？}
    C -->|否| D[明确拒答或提示<br/>知识库未找到相关内容]
    C -->|是| E[拼接上下文<br/>限制总 token]
    E --> F[系统提示：仅依据上下文<br/>引用格式 / 不足时说明]
    F --> G[调用 LLM 生成回答]
    G --> H{Web 流式？}
    H -->|是| I[SSE 输出]
    H -->|否| J[一次性返回文本]
    D --> K[结束]
    I --> K
    J --> K
```

---

## 5. 文档变更与一致性（摘要）

```mermaid
flowchart TD
    A[检测文档 content_hash] --> B{相对上次索引是否变化？}
    B -->|否| Z[跳过或仅更新元数据]
    B -->|是| C[删除该文档旧 chunks]
    C --> D[重新分块与嵌入]
    D --> E[upsert 向量与元数据]

    F[源文件已删除] --> G[级联删除向量与元数据]
```

---

*修订：与开发方案文档同步维护即可。*
