# Patho-AgenticRAG 代码架构

## 1. 总体架构

```text
病理图片 + 用户问题
        |
        v
demo/patho_agenticrag.py
        |
        +--> Agentic-Router：判断是否需要检索并生成检索请求
        +--> VRAG：调用图文检索服务
        |       +--> ColQwen2/ColPali：生成嵌入
        |       +--> Milvus：返回相关教材页面
        +--> 检索结果验证和上下文拼接
        +--> Patho-R1：生成最终答案
```

官方示例通过 OpenAI 兼容接口访问模型：

```text
Agentic-Router -> http://localhost:8002/v1
VRAG-Agent     -> http://localhost:8003/v1
Patho-R1       -> http://localhost:8004/v1/chat/completions
```

## 2. 运行流程

```text
启动 Milvus
  -> milvus_ingestion.py 建立向量库
  -> milvus_search_engine_api.py 提供检索 API
  -> 启动三个模型服务
  -> demo/patho_agenticrag.py
```

一次请求的调用链是：

```text
run(query, image)
  -> Agentic-Router 生成 tool_call
  -> VRAG 执行检索
  -> Milvus 返回病理教材页面
  -> 验证并合并检索上下文
  -> Patho-R1 生成答案
```

## 3. 目录职责

### `demo/`

- `patho_agenticrag.py`：主入口和 `MyAgent` 编排器。
  - `tool()`：判断是否需要检索。
  - `verify_rag_result()`：验证检索结果。
  - `call_specialist_model()`：调用 Patho-R1。
  - `run()`：串联完整流程。
- `vrag_agent.py`：调用检索服务并组织 VRAG 流程。
- `wrapper.py`：OpenAI 兼容 API 封装。
- `agent_prompt.md`：Agent 工具调用提示词。

### `search_engine/`

- `milvus_ingestion.py`：读取 `pdf_datasets` 图片，使用 ColQwen2/ColPali 生成嵌入并写入 Milvus。
- `milvus.py`：封装 collection、插入、查询、分区和去重。
- `milvus_search_engine.py`：执行文本/图片查询。
- `milvus_search_engine_api.py`：将检索器暴露为 HTTP API。
- `offline_rerank.py`：离线重排序逻辑。

### `pdf_datasets/`

病理教材页面图片，按病理主题分组。它是向量库的原始输入，不是现成的 Milvus 数据库。

```text
pdf_datasets/<topic>/*.jpg
  -> milvus_ingestion.py
  -> Milvus collection
```

### `multimodal_fusion/`

多模态融合、样本选择、相似度分析、进化算法和日志分析代码，不是最小 Agent Demo 的必经路径。

### 其他文件

- `images/`：Demo 示例图片。
- `standalone_embed.sh`：启动 Milvus Standalone 的脚本。
- `requirements.txt`：vLLM、PyMilvus、ColPali、Transformers 等依赖。

## 4. 模型和数据

| 组件 | 作用 |
|---|---|
| Agentic-Router | 判断是否检索、生成检索问题和分区信息 |
| Qwen2.5-VL-7B-VRAG | 视觉检索相关推理 |
| Patho-R1-7B | 结合图片和检索上下文回答 |
| ColQwen2/ColPali | 生成多模态检索嵌入 |
| Milvus | 保存和搜索页面向量 |

## 5. 本地运行边界

RTX 4060 Laptop 8GB 可以完成：

- 代码阅读、修改和语法检查；
- 少量图片建库和检索测试；
- Milvus 或 FAISS 小规模检索；
- Agent 调度逻辑调试；
- 使用云端视觉模型或小型量化模型测试端到端流程。

不适合直接完成：

- 三个官方模型同时常驻；
- 官方多 GPU vLLM 部署；
- 全量病理图片建库；
- 论文指标的严格复现。

官方示例将模型分配为 1 张 GPU、2 张 GPU、2 张 GPU，严格并发部署约需 5 张 GPU。单卡可通过量化、依次启动模型或替换模型验证流程，但属于简化部署。

## 6. 修改时需要同步检查

1. 模型服务端口和模型名称：`patho_agenticrag.py`、`wrapper.py`、vLLM 命令。
2. 检索 API 请求格式：`vrag_agent.py`、`milvus_search_engine_api.py`。
3. Milvus collection、字段和向量形状：`milvus.py`、`milvus_ingestion.py`、`milvus_search_engine.py`。
4. Agent 工具调用格式：`agent_prompt.md`、`patho_agenticrag.py`。
5. 图片路径：Demo、建库脚本和服务器工作目录。

## 7. 常见问题

- `pdf_datasets` 必须先经过 ingestion 才能被 Agent 检索。
- Windows 对带结尾空格的目录名不兼容，完整建库建议在 Linux 服务器执行。
- 只有模型服务启动后，`patho_agenticrag.py` 才能完成端到端推理。
- 云端模型可以验证工程流程，但不能严格复现论文模型结果。
- 真实患者数据不应直接上传到公共云服务器或云端模型 API。
