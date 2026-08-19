# LangChain 与 LangGraph

**核心差异一句话**：写大模型应用时，LLM 只是一颗"算力核心"，真正复杂的是**怎么把模型、提示词、检索、工具、外部 API 编排成一条可靠的工作流**。LangChain 是"积木拼装"（链式串联、灵活自由），LangGraph 是"指挥调度"（把状态机化、可控可回滚）；前者让你快速搭起来，后者让你搭得稳、难失控。

> 串起全文的主线：**从"链"到"图"，从"自由拼装"到"有纪律的编排"。** 先讲清 LangChain 解决了什么、它的核心原子（LCEL 与链）；再重点展开 LangGraph——为什么要有它、它的 State/Node/Edge 三要素、循环与条件分支如何实现、持久化与人工介入怎么落地；最后补生态（LangSmith/LangServe）与同类框架对比。它与_agent 笔记_里的 ReAct、workflow 编排直接呼应——LangGraph 正是实现那种"循环 + 分岔 + 断点续跑"的工程底座。

## 目录

> **一、框架概览**
> - [LangChain 是什么](#langchain-是什么)
> - [LangChain vs LangGraph](#langchain-vs-langgraph)
>
> **二、LangChain 核心（链与 LCEL）**
> - [模型 I/O](#模型-io)
> - [数据连接](#数据连接)
> - [LCEL：可组合的表达式语言](#lcel可组合的表达式语言)
> - [传统 LangChain 的局限](#传统-langchain-的局限)
>
> **三、LangGraph 核心概念**
> - [为什么要造图](#为什么要造图)
> - [三个基本元素：State / Node / Edge](#三个基本元素state--node--edge)
> - [三种边的类型](#三种边的类型)
> - [State 与 Reducer](#state-与-reducer)
>
> **四、编排原语与进阶能力**
> - [流式、持久化与断点](#流式持久化与断点)
> - [人工介入与时间旅行](#人工介入与时间旅行)
> - [一个最小 ReAct 图](#一个最小-react-图)
>
> **五、工程化与生态**
> - [LangSmith：可观测平台](#langsmith可观测平台)
> - [LangServe：把应用暴露成 API](#langserve把应用暴露成-api)
> - [同类框架对比](#同类框架对比)
>
> - [阅读资料](#阅读资料)

---

## 一、框架概览

### LangChain 是什么

LangChain 是构建 LLM 应用的主流开源框架（Python / JS），目标是把常见工作"积木化"，避免每个应用从零再写一遍胶水代码。它围绕 LLM 提供三大类能力：

- **模型 I/O**：统一接入各类大模型，管理提示词与输出解析。
- **数据连接**：对接文档加载、切块、向量库、检索，做 RAG。
- **链与 Agent**：把上述原子串成可复用的工作流，或交给 Agent 自主驱动工具。

一句话实质：**给"调 API"这件事套上一层工程化的积木系统。**

### LangChain vs LangGraph

| 维度 | LangChain | LangGraph |
| ---- | ---- | ---- |
| 定位 | 链式编排工具 | 图式状态机编排工具 |
| 抽象 | 链（Chain）、可组合表达式 | 图（Graph）上的节点与边 |
| 控制流 | 线性为主，循环/分支靠外挂 | **原生支持循环、条件分支、并行** |
| 状态 | 基本靠传参、可随意覆盖 | **显式状态机（State），节点显式读写** |
| 可控性 | 高自由度，但也容易失控 | 受控，天然适合"需回滚/断点/人工介入" |
| 适用途 | 快速原型、RAG、简单问答 | 稳定生产、复杂流程、对每步有要求 |

> **关系不是替代而是演进**：LangChain 仍在（负责"积木"），LangGraph 是官方重点发展的"编排层"——在 LangGraph 里依然可以用 LangChain 的模型、检索器等组件当节点内容。

---

## 二、LangChain 核心（链与 LCEL）

### 模型 I/O

- **两类大模型，按场景选**：
  - **LLM（文本生成模型）**：一段文本进、一段文本出，如 Llama、Qwen，适合翻译/总结等简单任务。
  - **ChatModel（对话模型）**：接收一组带角色标记的"对话消息"，返回一条消息，如 GPT-4o，适合多轮对话——因为它能更好地理解对话上下文的逻辑。
- **Prompts**：`PromptTemplate`（变量插值）、`ChatPromptTemplate`（多角色消息模板）。
- **Output Parsers**：把自由文本解析成结构化结果（`StrOutputParser`、JSON、Pydantic）。

#### 消息角色：不只是"谁在说话"

> 初学者常误以为 `system` / `user` / `assistant` 只是区分发言者身份。实际上，**角色的核心作用是表达不同层级的约束关系**：

- **system**：不参与具体问答，为模型设定整体行为规则（身份、风格、行为边界）。
- **user**：本轮用户希望完成的具体任务（问题、指令或补充）。
- **assistant**：本质不是身份标识，而是**对话历史的一部分**，靠它保持上下文连续性（再配合 `tool` 回填工具结果）。

### 数据连接

为 RAG 提供完整管线：`Document Loader`（加载）→ `TextSplitter`（切块）→ `embedding` + `VectorStore`（向量化入库）→ `Retriever`（检索）。这与_rag 笔记_里的"索引→检索→生成"三阶段一一对应。

### LCEL：可组合的表达式语言

LCEL（LangChain Expression Language）用管道符 `|` 把组件链起来，是 LangChain 的核心语法——**一切皆可"投喂 → 产出 → 传下一步"**：

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("用一句话总结：{text}")
model = ChatOpenAI()
llm_chain = prompt | model | StrOutputParser()   # 管道式组合

print(llm_chain.invoke({"text": "LangChain 是构建 LLM 应用的框架。"}))
```

- **组合即复用**：`|` 连接的整条链本身又是一个可再组合的组件。
- **内置批处理/流式/异步**：`.invoke()`、`.stream()`、`.batch()`、`.ainvoke()` 等接口统一，切换开销低。

#### 常用 Runnable 组件

| 组件 | 作用 |
| ---- | ---- |
| **RunnableSequence** | 线性流转，`A | B | C` 顺序执行 |
| **RunnableBranch** | 动态路由，按条件/LLM 理解匹配目标链 |
| **RunnableMap** | 并行容器，同时执行多个 Runnable，合并结果 |
| **RunnablePassthrough** | "透传"，原样传递输入（或据此派生数据），常用于保留原始输入 |
| **RunnableWithMessageHistory** | 让链具备对话记忆（配合 `BaseChatMessageHistory`） |

**路由链（新版基于 Runnable 范式）**：`目标链（多种场景 Runnable）` + `路由选择器（条件/LLM 判断）` + `默认链（兜底）`。路由选择器决定"这个输入走哪条支线"，匹配不上就走默认兜底。

#### 对话记忆：history 无限增长怎么办

多轮会话里 `history` 无限增长，必须做**上下文管理**，常用方案：

1. **全量记忆（最简）**：完整保存所有历史，适合短对话，长对话 token 爆炸。
2. **滑动窗口（窗口记忆）**：只保留最近 N 轮，控 token，适合普通聊天。
3. **自动摘要（摘要记忆）**：历史过长时用 LLM 总结旧对话，用摘要替代原始历史，兼顾连贯与效率。
4. **短期 + 长期记忆（Agent 常用）**：短期存最近几轮（Memory/窗口），长期把重要信息存数据库——向量库（Qdrant/Milvus）存知识、图数据库（Neo4j）存用户关系/偏好/事实。

生产环境通常是**组合**：
`System Prompt + 最近 N 轮对话 + 历史摘要 + 长期记忆检索结果`，而不是永久保存完整 `history`。

> Memory 的两个核心动作：**存储（Save）**——把每轮 Human/AI 消息写入介质；**提取（Load）**——新轮次前把历史取出注入 Prompt（即"记忆 = 对小窗口的检索"）。

### 传统 LangChain 的局限

- 链是**线性**的，遇到"要循环直到完成、要根据条件走不同分支"就力不从心。
- 早期 `AgentExecutor` 虽能循环，但**状态容易随意覆盖、难以持久化、出错难回滚**——一旦生产要求"断点续跑 / 人工审批 / 精确回退"，就撑不住。

> 这正是 LangGraph 登场的原因：**把自由度收编进"图状态机"，让循环、分支、回滚都有清晰落点。**

---

## 三、LangGraph 核心概念

### 为什么要造图

LangGraph（LangChain 团队）把应用建模为**有向图 + 显式状态**。相比自由拼链，它带来三个硬收益：

1. **循环成第一公民**：Agent 的"思考→行动→观察"天然是环，图上直接画回边即可，不用再硬套线性链。
2. **状态可控**：所有节点只朝同一份 `State` 读写，谁改了啥、改了哪一版一目了然。
3. **可持久化可回滚**：配合 checkpointer，每条执行是一条"可控的生命线"，随时暂停/恢复。

### 三个基本元素：State / Node / Edge

- **State（状态）**：整个图共享的数据结构。常见用 `TypedDict` 或 Pydantic 定义，例如 `{"messages": [...]}`。**它是图的"内存"，节点读它、改它、传递它。**
- **Node（节点）**：一个可调用的函数，输入 `state`，处理后返回"要更新的字段的字典"。节点就是"一步动作"（调用一次模型、执行一次检索、运行一段工具）。
- **Edge（边）**：定义节点之间的流转关系，决定"做完这步去哪里"。

### 三种边的类型

| 边类型 | 作用 | 例子 |
| ---- | ---- | ---- |
| **普通边（Edge）** | 一走完 A 就固定去 B | A 执行完必然进 B |
| **条件边（Conditional Edge）** | 根据当前状态动态决定下一个节点 | 模型输出决定"再查一次"还是"Finish" |
| **入口边 / 结束点** | 标记图的开始和结束 | 从 `START` 进，到 `END` 出 |

```python
from langgraph.graph import StateGraph, START, END

def call_llm(state):
    # 读 state、调模型、返回要更新的字段
    return {"messages": state["messages"] + ["..."]}

graph = StateGraph(State)
graph.add_node("assistant", call_llm)
graph.add_edge(START, "assistant")            # 入口
graph.add_conditional_edges(                  # 条件边：决定下一步
    "assistant",
    lambda s: "tool" if s["needs_tool"] else END,
    {"tool": "tool_node", END: END}
)
app = graph.compile()
```

### State 与 Reducer

- 节点更新 State 时默认是**整体覆盖**该 key。但很多场景要"追加"（如不断往 `messages` 里加消息），于是引入 **Reducer**：
  - 定义 State 的字段时附一个 reducer 函数（如 `operator.add` / `add_messages`），同 key 多份更新时按 reducer 合并，而不是互相覆盖。
  - `add_messages` 是内置最常用的 reducer，让多路并行节点都能把消息正确追加。
- 这是 LangGraph 比普通链路"状态纪律化"的关键：**谁会写、如何合并、冲突怎么办，都由图定义讲清楚。**
- **节点是"纯函数"**：读 State → 执行 → 返回状态补丁（只含要更新的字段，不必返回完整状态）。例如只更新 `progress` 就返回 `{"progress": 1}`，LangGraph 自动合并进全局 State。

#### 字段设计的三个原则

- **最小必要原则**：只定义工作流必须的字段，避免冗余数据占内存。
- **可更新原则**：只有需要跨节点传递/修改的数据才设为状态字段，固定不变的配置不放入。
- **清晰命名原则**：字段名直观反映含义，如 `user_query` / `generated_text` / `quality_score`。

---

## 四、编排原语与进阶能力

### 流式、持久化与断点

- **流式（Streaming）**：节点级、token 级实时输出中间状态，长任务体验好。
- **持久化（Persistence）**：通过 `checkpointer`（内存、SQLite、Redis、Postgres）在每一步保存状态快照。
- **断点（Interrupt）**：在图的关键位置设断点暂停执行，允许"审批通过后再继续"，无需从头跑。
- **时间旅行（Time Travel）**：因为有检查点，可以**回滚到历史任意一个状态**重新分岔，调试与"重跑"非常自然。

### 人工介入与时间旅行

- **人工介入（Human-in-the-loop）**：当模型要执行"不可逆"的高风险动作（付款、删库、发邮件）时，在动作节点前暂停，交由人确认——这正是_agent 笔记_里 Harness"安全与越权护栏"的落地形态。
- **时间旅行**：配合 checkpointer 可回看、回退、修复状态后重放，是生产级调试的关键。

### 一个最小 ReAct 图

把 RAG 与工具调用收编进一个可控循环：

```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, START, END, add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

def agent(state):        # 节点：模型决定下一步
    ...
def tool(state):         # 节点：执行工具
    ...

def route(state) -> Literal["tool", END]:   # 条件边
    return "tool" if state["messages"][-1].tool_calls else END

builder = StateGraph(State)
builder.add_node("agent", agent)
builder.add_node("tool", tool)
builder.add_edge(START, "agent")            # 入口
builder.add_conditional_edges("agent", route, {"tool": "tool", END: END})  # 循环 + 出口
builder.add_edge("tool", "agent")           # 工具结果回到 agent（回边构成循环）
app = builder.compile()
```

> 这就是把 _agent 笔记_里的 **ReAct / Agentic Loop** 画成了图：`agent` 与 `tool` 之间的环 = 思考与行动的循环，`route` = 谁来结束它，checkpointer = 谁能收回重来。

---

## 五、工程化与生态

### LangSmith：可观测平台

LangChain 官方的一体化平台，用于**追踪、评估、监控**：

- 记录每次执行的完整轨迹（输入/输出、每步耗时、token 用量、工具调用结果）。
- 提供 Prompt 管理、数据集、回归评测（对应_agent 笔记_里的"Golden Dataset + LLM 裁判"）。
- 生产环境定位"为什么模型这次回答差了"的主力工具。

### LangServe：把应用暴露成 API

把编译好的 LangGraph 应用一键部署成 REST API 服务，客户端可流式订阅结果，省去自写服务层。适合从"脚本原型"到"可用服务"的过渡。

### 同类框架对比

| 框架 | 定位 | 特点 |
| ---- | ---- | ---- |
| **LangGraph** | 有向图状态机编排 | 循环/分支/断点/时间旅行，可控性最强 |
| **CrewAI** | 多 Agent 角色协作 | 类团队分工（领队/执行者），上手快 |
| **AutoGen（Microsoft）** | 多 Agent 对话 | 强调 Agent 之间互相"交流/辩论"求解 |
| **LlamaIndex** | 数据框架 | 侧重 RAG 与数据接入，检索栈成熟 |
| **原生手写** | 无框架 | 最灵活，但循环/状态/持久化都要自己造轮子 |

> 一句话选型：**要细粒度可控、生产级循环编排 → LangGraph；要快速多角色协作 → CrewAI；主打检索数据接入 → LlamaIndex；只有极简线性问答 → 直接 LangChain 甚至不引框架。**

---

## 阅读资料

[LangGraph 官方文档](https://langchain-ai.github.io/langgraph/)

[LangChain 官方文档](https://python.langchain.com/)

[LangSmith 平台](https://www.langchain.com/langsmith)