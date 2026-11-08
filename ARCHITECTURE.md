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

## 3.1 文件级职责清单

### 根目录文件

- `.gitignore`：指定不提交到 Git 的缓存、模型、运行产物和本地配置。
- `README.md`：项目介绍、安装步骤、Milvus 启动方式、模型下载方式和 Demo 启动命令。
- `ARCHITECTURE.md`：本项目的模块、文件、数据流和部署边界说明。
- `requirements.txt`：Python 依赖版本约束，重点包括 `vllm`、`pymilvus`、`transformers` 和 ColPali Git 版本。
- `standalone_embed.sh`：Milvus Standalone 启动脚本，依赖 Docker。

### `demo/` 文件

- `__init__.py`：将 `demo` 声明为 Python 包。
- `agent_prompt.md`：Agent 的提示词模板，约束 Router 输出工具调用、检索问题和病理分区。
- `wrapper.py`：封装 OpenAI 兼容客户端，统一模型名、服务地址和 API key 的传入方式。
- `vrag_agent.py`：VRAG 检索编排器，接收文本问题、图片和分区信息，调用检索服务并迭代返回检索结果。
- `patho_agenticrag.py`：端到端入口。
  - `MyAgent.__init__()`：初始化 Router、VRAG 和 Patho-R1 地址。
  - `encode_image()`：将输入图片编码为 Base64，供 HTTP 多模态请求使用。
  - `tool()`：调用 Router，得到是否检索以及检索参数。
  - `verify_rag_result()`：让 Router 判断检索内容是否有帮助。
  - `call_specialist_model()`：将图片、问题和检索上下文发送给 Patho-R1。
  - `get_rag_answer_from_generator()`：从 VRAG 生成器中提取最终检索内容。
  - `run()`：串联 Router、检索、结果验证和专家模型。

### `search_engine/` 文件

- `__init__.py`：将 `search_engine` 声明为 Python 包。
- `milvus.py`：Milvus 访问层。负责客户端初始化、collection 创建、字段定义、分区、插入、查询和文档去重。
- `milvus_ingestion.py`：离线建库入口。遍历图片目录，加载 ColQwen2，提取页面嵌入并写入 Milvus；支持按病理主题建立分区。
- `milvus_search_engine.py`：在线检索逻辑。负责将查询图片/文本转成向量，在 Milvus 中搜索并整理页面路径、分数和上下文。
- `milvus_search_engine_api.py`：HTTP 服务层。把 `milvus_search_engine.py` 的查询能力暴露给 `demo/vrag_agent.py`。
- `offline_rerank.py`：离线重排序工具，用于对初始检索结果进一步排序或评估。

### `multimodal_fusion/` 文件

该目录主要服务于多模态融合、样本选择和实验分析，不是最小 Demo 的必经路径。

- `__init__.py`：将目录声明为 Python 包。
- `basic_functions.py`：共享基础函数，例如数据处理、日志、评估或通用计算工具。
- `archive.py`：实验文件归档和历史结果整理。
- `code_evaluation.py`：对候选代码、融合策略或实验结果进行评估。
- `ea.py`：进化算法主流程，用于搜索或优化融合策略。
- `gp_evolution.py`：遗传编程/进化搜索相关逻辑。
- `gp_util.py`：遗传编程的辅助函数、表达式和适应度工具。
- `instance_selection.py`：从样本集合中选择用于训练、检索或评估的实例。
- `selection.py`：通用候选选择和筛选逻辑。
- `similarity_discovery.py`：计算或发现多模态样本之间的相似关系。
- `surrogate_model.py`：代理模型，用较低成本近似评估融合策略。
- `ndcg.py`：计算 NDCG 等排序质量指标。
- `notification.py`：实验过程中的通知或状态提示。
- `time_util.py`：时间统计、计时和时间格式化工具。
- `analysis/clean_log.py`：清理和规范化实验日志。
- `analysis/fitness_analysis.py`：分析进化过程中的适应度变化。
- `analysis/log_plot_tool.py`：读取日志并绘制分析图。

### `docs/` 和 `images/`

- `docs/agent_demo_detailed.ipynb`：Agent Demo 的交互式说明和实验笔记本。
- `docs/casestudy.png`：README 和案例说明使用的示意图。
- `images/demo.jpg`：`patho_agenticrag.py` 默认读取的示例输入图片。

### `pdf_datasets/` 文件内容

该目录按病理主题保存页面图片。目录名是检索分区名，图片文件是 Milvus 的原始文档。它们不是 Python 包，也不是预构建数据库。

```text
pdf_datasets/<topic>/*.jpg
  -> milvus_ingestion.py
  -> ColQwen2/ColPali 向量
  -> Milvus collection/partition
```

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
