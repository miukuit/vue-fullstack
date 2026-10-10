# AI Agent — LangChain / LangGraph / DeepAgent + FastAPI

Agent 服务独立为 `apps/agent`（Python），职责边界清晰：**不做业务主数据写入**，业务数据读写通过调用主 API 或只读查询完成。

## 选型

| 场景 | 用 |
|---|---|
| 简单链式调用、工具调用、RAG | langchain (`langchain-core` + 社区 integration) |
| 有状态、多步骤、循环、人工介入 | **langgraph** |
| 复杂推理/规划/多子 agent 编排 | deepAgent（在 langgraph 之上） |
| 对外 HTTP 服务 | **FastAPI** |

## 目录结构

```
apps/agent/
├── app/
│   ├── main.py            # FastAPI 入口
│   ├── api/routes/        # 路由
│   ├── graph/             # langgraph 图定义（nodes/edges/state）
│   ├── chains/            # 简单 chain
│   ├── tools/             # 工具函数（@tool）
│   ├── prompts/           # 提示词模板（版本化）
│   ├── models/            # LLM/embedding 客户端工厂
│   ├── schemas/           # pydantic 模型（入参/出参）
│   └── core/              # 配置、日志、异常
├── tests/
├── pyproject.toml         # 用 uv 管理
└── Dockerfile
```

## FastAPI 约定

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="agent")

class ChatIn(BaseModel):
    session_id: str = Field(min_length=1)
    message: str = Field(min_length=1, max_length=4000)

class ChatOut(BaseModel):
    session_id: str
    reply: str
    citations: list[str] = []

@app.post("/chat", response_model=ChatOut)
async def chat(body: ChatIn) -> ChatOut:
    ...
```

- 入参/出参一律 pydantic 模型，与 zod 对齐字段命名（snake_case 在 Python 侧，前端转换）。
- 长任务（>30s）不阻塞 HTTP：提交任务返回 `task_id`，用 SSE 或轮询取结果；流式用 `StreamingResponse` + SSE。
- 统一异常处理 `@app.exception_handler`，返回 `{ code, message }` 与主 API 同构。
- 日志用 `structlog` 或 stdlib logging 输出 JSON，字段与 pino 对齐（`requestId`、`userId`、`durationMs`），同样推 Loki。
- 配置用 `pydantic-settings`：`OPENAI_API_KEY`、`OPENAI_BASE_URL`、`MODEL_NAME`、`CHECKPOINT_DB_URL`。
- 依赖管理用 `uv`（`uv sync`、`uv run`），不用 requirements.txt 手写。

## LangGraph 约定

- State 用 `TypedDict` 显式声明，含 `messages`、`context`、`next` 等字段；用 `Annotated[list, add_messages]` 管理消息合并。
- 检查点（持久化）用 `PostgresSaver`，复用主库另一 schema，实现会话续跑与人工中断。
- 每个 node 是纯函数：输入 state → 输出 partial state，不藏副作用；有副作用的写库操作显式标注。
- 循环必须有终止条件（最大步数 / token 预算），防止死循环烧钱。
- 人工介入用 `interrupt()`，配合前端的确认交互。
- 编译图 `graph.compile(checkpointer=...)` 暴露为单例，不要每次请求重新编译。

```python
from langgraph.graph import StateGraph, END

builder = StateGraph(AgentState)
builder.add_node("plan", plan_node)
builder.add_node("act", act_node)
builder.set_entry_point("plan")
builder.add_conditional_edges("plan", route, {"act": "act", "end": END})
builder.add_edge("act", "plan")
graph = builder.compile(checkpointer=checkpointer)
```

## 工具与安全

- 工具用 `@tool` 装饰器，docstring 即工具描述（直接影响模型选择），写清楚用途与参数。
- 工具入参用 pydantic 校验，工具内做权限判断（当前 userId 能否操作该资源）。
- **禁止**把 SQL 拼接、shell 执行、任意 URL 抓取直接暴露为工具；需要时走白名单 + 参数化。
- 涉及写操作的工具必须幂等或带确认（human-in-the-loop）。
- Prompt 模板放 `prompts/` 并版本化，改动要能追溯。
- 输出必须结构化时用 `with_structured_output(PydanticModel)`，不要靠字符串解析。

## RAG

- 向量库：pgvector（复用 PG，优先）或独立向量库；不引入额外数据库除非确实需要。
- 文档切分按语义（标题/段落）而非固定字符数；chunk 带元数据（文档 id、页码）便于引用。
- 检索结果必须能引用溯源（返回 `citations`），不确定时明确说"未找到依据"，不编造。

## 可观测

- 每轮对话记录：`session_id`、`user_id`、耗时、token 用量、工具调用链、失败原因。
- 接 LangSmith 或自建 trace 表；生产默认关闭 prompt 全量落库（隐私），只记元数据。
- 成本监控：按模型分别统计 token，设日预算告警。
