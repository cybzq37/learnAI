## 一、核心架构与基础概念

**Q1：FastAPI 由哪几个库构建？各自负责什么？**

FastAPI 建立在两个核心库之上：**Starlette** 负责 Web 层能力（路由、Request/Response、中间件、WebSocket、静态文件、异常处理、测试客户端、线程池处理同步端点），**Pydantic** 负责数据校验、类型转换和 JSON Schema 生成。

面试中如果只答“基于 Starlette 和 Pydantic”会被追问分工。完整回答应说清：**Uvicorn 是 ASGI 服务器**，负责监听端口、接收 HTTP/WebSocket 连接，按 ASGI 规范调用应用；**Starlette 是 Web 框架底座**；**FastAPI 在 Starlette 之上加了类型驱动的 API 开发体验**；**Pydantic 是数据校验引擎**。

**Q2：ASGI 和 WSGI 的本质区别是什么？**

WSGI 应用是**同步 callable**，一个请求对应一个线程/进程，对 WebSocket、长连接、流式响应支持不自然。ASGI 把连接抽象为 `scope + receive + send` 三个参数，使框架可以处理普通 HTTP、WebSocket 长连接、长轮询、流式响应和异步 IO 任务。

关键追问：“为什么 WSGI 不支持 WebSocket？”——因为 WSGI 的接口是“请求进来 → 一次性返回响应”的同步模型，没有持久双向通信的抽象。

**Q3：FastAPI 的快体现在哪里？**

FastAPI 的高性能主要来自两个层面：底层是 Starlette 的 ASGI 异步处理能力，上层是 Pydantic V2 用 Rust 重写的核心校验引擎。
但面试官真正想听的是：**async 的优势主要体现在高并发 IO，不是让 CPU 计算自动并行**。`async def` 里必须一路使用可 await 的库，否则就是伪异步；同步库用 `def` 或显式线程池。

## 二、Pydantic 与数据验证

**Q4：Pydantic 如何让类型提示在运行时真正生效？**

Python 解释器本身不强制类型提示。Pydantic 在**类定义阶段**读取 `__annotations__`，为每个字段生成校验器；在**实例化阶段**按校验器逐字段转换和校验。类型不匹配时抛出 `ValidationError`，FastAPI 捕获后自动返回 422 响应。

**Q5：为什么请求模型、存储模型、响应模型应该分开？**

这是一个高频的高级问题。用一个 `User` 模型同时做请求体和响应体，会导致 `hashed_password` 通过 `response_model` 泄露。正确做法是定义 `UserCreate`（含明文密码）、`UserInDB`（含哈希）、`UserPublic`（不含敏感字段），FastAPI 的 `response_model=UserPublic` 会自动过滤输出。Pydantic V2 的 `model_config` 中还可以用 `from_attributes=True` 配合 ORM 对象。

**Q6：Pydantic V2 的 lax 和 strict 模式有什么区别？**

lax 模式（默认）允许类型强制转换，比如字符串 `"123"` 转 `int`；strict 模式拒绝任何隐式转换。**coercion 的危险场景**：API 接收 `{"age": "not_a_number"}` 时 lax 模式报错，但如果传入 `{"age": true}`，lax 模式可能把 `True` 转成 `1`，这在业务上通常是 bug。

**Q7：如何实现超出 Pydantic 默认能力的自定义校验？**

使用 `@field_validator`（Pydantic V2）或 `@model_validator` 实现跨字段校验。例如密码长度校验、密码与确认密码一致性校验。自定义校验适用于业务规则复杂、输入需要预处理、安全强制检查等场景。


## 三、异步编程与并发

**Q8：`async def` 端点和 `def` 端点在 FastAPI 中的调度方式有何不同？**

FastAPI 根据 endpoint 类型决定调用方式：`async def` 直接在事件循环中 await；普通 `def` 会被自动放到线程池执行。

**Q9：`async def` 里调用了阻塞库（如 `requests.get`）会发生什么？**

会**阻塞整个事件循环**，其他所有请求都无法处理。这是面试中的经典陷阱题。正确做法是改用 `httpx.AsyncClient` 或 `aiohttp`。如果必须用同步库，端点上不要写 `async def`，让 FastAPI 自动丢线程池。

**Q10：`await` 到底做了什么？**

`await` 暂停当前协程，将控制权交还给事件循环，直到被 await 的对象就绪。在等待期间，事件循环可以执行其他协程。这是 async 提升吞吐量的核心机制。

**Q11：什么时候 async 不会提升性能？**

CPU 密集型场景。async 解决的是 IO 等待期间的并发利用问题，不解决计算并行。CPU 密集任务应该用 `def`（自动进线程池）或 `ProcessPoolExecutor`/Celery 分发到独立进程。


## 四、依赖注入（Depends）

**Q12：Depends 系统内部如何工作？**

FastAPI 在启动时分析路径操作函数的签名，构建依赖树。每个请求到来时，按拓扑序执行依赖：先执行最底层依赖，再逐层向上。如果一个依赖被多个路径函数引用，同一请求内默认只执行一次（`use_cache=True`）。

**Q13：依赖缓存什么时候会出问题？**

数据库会话缓存是合理的——同一请求内多个函数共享一个 Session。但“获取当前时间”“获取实时配置”这类依赖需要 `use_cache=False`，否则同一请求内多次注入会拿到同一个旧值。

**Q14：基于类的依赖有什么优势？**

类作为依赖时，`__init__` 的参数会被 FastAPI 像路径函数参数一样解析（支持 Query、Path、Header 等）。适合需要状态的依赖，比如带配置的限流器、带认证级别的权限检查器。

**Q15：依赖注入和中间件的区别？**

依赖注入作用于**路由级别**，返回值给视图函数使用，可嵌套、可缓存、可按路由启用。中间件作用于**全局请求/响应**，包裹整个生命周期，适合日志、CORS、计时等横切关注点。执行顺序上，中间件在依赖之前执行。


## 五、中间件与生命周期

**Q16：中间件的洋葱模型怎么理解？**

添加顺序与实际执行顺序**相反**。最后添加的中间件最先执行请求处理，最后执行响应处理。例如先 `add_middleware(CORSMiddleware)` 再 `add_middleware(LoggingMiddleware)`，请求路径是 Logging → CORS → 路由，响应路径是 CORS → Logging。

**Q17：startup/shutdown 和 lifespan 有什么区别？**

旧版 FastAPI 用 `@app.on_event("startup")` 和 `@app.on_event("shutdown")`。推荐用 **lifespan 上下文管理器**，把启动和关闭逻辑收在一个 `async with` 块中，保证资源获取与释放的对称性，避免忘记关闭连接池。

**Q18：如何在中间件中正确处理异常？**

中间件中 `call_next` 抛出的异常会向上传播。如果中间件有自己的 try/except，需要注意不要吞掉 `HTTPException`——应重新 raise。FastAPI 的 `ExceptionMiddleware` 在中间件链的最内层处理 `HTTPException`。


## 六、流式响应与实时通信

**Q19：SSE 和 WebSocket 分别适用于什么场景？**

SSE（Server-Sent Events）是**单向**的服务器到客户端推送，基于 HTTP 长连接，适合 LLM token 逐字输出、实时通知。WebSocket 是**全双工**通信，适合聊天、协作编辑等需要客户端持续发送消息的场景。

**Q20：FastAPI 中如何实现流式响应？**

使用 `StreamingResponse` 配合异步生成器。FastAPI 2.0 对流式响应做了底层重构，原生支持 `AsyncGenerator` 类型提示驱动的流式输出。在大模型服务中，典型模式是 `async def generate(): async for chunk in llm.stream(prompt): yield f"data: {chunk}\n\n"`。

**Q21：WebSocket 的认证怎么做？**

FastAPI 的 WebSocket 没有内置的中间件认证。典型做法是在 `websocket.accept()` 之前校验 token（从 query 参数或首条消息中获取），校验失败则 `await websocket.close(code=1008)`。FastAPI 的 WebSocket 支持依赖注入，可以复用 HTTP 端点的认证依赖。


## 七、安全认证

**Q22：FastAPI 的 JWT 认证完整流程？**

1）定义 `SECRET_KEY` 和算法（HS256 或 RS256）；2）登录端点用 `OAuth2PasswordRequestForm` 接收凭证，校验密码哈希后生成 JWT；3）创建 `OAuth2PasswordBearer` 依赖，在每个受保护端点上通过 `Depends` 注入，解码 token 并校验 `exp` 声明。

**Q23：JWT 的安全规则有哪些必须遵守？**

用 `openssl rand -hex 32` 生成强密钥；access token 过期时间设 15-30 分钟；始终校验 `exp` 声明；算法只用 HS256 或 RS256，**绝不能用 `none`**。

**Q24：FastAPI 没有全局认证中间件，这合理吗？**

合理。FastAPI 的设计哲学是“每个端点显式声明自己的认证依赖”。这比全局中间件更安全——开发者不会因为忘记加中间件而意外暴露端点。通过 `APIRouter` 的 `dependencies=[Depends(verify_token)]` 可以在路由组级别批量应用。


## 八、数据库集成

**Q25：FastAPI 中数据库会话的标准生命周期管理？**

用 `yield` 依赖：

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

异步场景用 `AsyncSession` 配合 `async with`。关键点：**会话的创建和关闭必须对称**，`yield` 之后的代码保证请求结束后执行清理。

**Q26：异步 SQLAlchemy 和同步 SQLAlchemy 在 FastAPI 中分别怎么用？**

异步驱动（`asyncpg` + `AsyncSession`）配合 `async def` 端点，适合高并发。同步驱动（`psycopg2` + `Session`）配合 `def` 端点，FastAPI 自动丢线程池。**不要混用**：`async def` 端点里用同步 Session 会阻塞事件循环。


## 九、测试

**Q27：TestClient 和真实 HTTP 请求的区别？**

`TestClient` 基于 `httpx`，在**进程内**直接调用 ASGI 应用，不经过网络栈，速度快、无需启动服务器。适合单元测试和集成测试。真实 HTTP 请求用于端到端测试，验证部署环境。

**Q28：如何隔离测试中的数据库依赖？**

用 `app.dependency_overrides[get_db] = get_test_db` 替换真实依赖，让测试使用内存数据库或独立测试库。pytest fixture 中在 setup 阶段创建表、在 teardown 阶段回滚或删除。

**Q29：异步端点的测试怎么写？**

用 `httpx.AsyncClient` 配合 `ASGITransport`，或用 `pytest-asyncio` 的 `@pytest.mark.asyncio`。`TestClient` 本身是同步的，测试异步端点时需要确保事件循环正确管理。


## 十、生产部署与性能优化

**Q30：Uvicorn 和 Gunicorn 在生产环境中如何配合？**

Gunicorn 作为**进程管理器**，Uvicorn 作为 **worker**。`gunicorn -k uvicorn.workers.UvicornWorker -w 4 app:app`。worker 数量建议等于 CPU 核心数，最小化上下文切换开销。

**Q31：FastAPI 性能优化的三个核心方向？**

**路由模块化拆分 + 依赖按需注入**（减少不必要的依赖构建开销）；**异步数据库连接池 + 并行查询**（避免 N+1 和串行 IO 等待）；**热点数据内存缓存 + 响应压缩**（Redis 缓存 + GZipMiddleware）。

**Q32：CORS 配置的常见陷阱？**

`allow_origins=["*"]` 和 `allow_credentials=True` **不能同时使用**。浏览器规范禁止 credential 请求使用通配符 origin。生产环境必须显式列出允许的 origin。

---

## 十一、高频率追问与场景题

**Q33：FastAPI 中如何设计一个限流器？**

用**依赖注入**实现：基于类的依赖，在 `__init__` 中接收 `rate_limit` 和 `window` 参数，使用 Redis 的滑动窗口或令牌桶算法。比中间件更灵活——可以针对不同路由设置不同限流策略。

**Q34：BackgroundTasks 和 Celery 如何选择？**

`BackgroundTasks` 适合**响应后执行、丢失可接受**的任务（发送邮件、写日志、更新统计）。Celery 适合**必须可靠执行、需要重试和持久化**的任务（支付处理、文件转换、批量计算）。BackgroundTasks 同进程运行，进程崩溃任务丢失。

**Q35：FastAPI 微服务之间如何通信？**

REST API（HTTP 端点同步调用）、消息队列（RabbitMQ/Kafka 异步解耦）、服务发现（动态定位服务实例）。设计原则：独立部署、独立数据库、明确的 API 契约、集中式认证和日志。

**Q36：如何实现零停机部署？**

蓝绿部署或滚动更新。FastAPI 的 **lifespan 中在 startup 阶段完成数据库迁移检查**，但不在启动时执行迁移（迁移应由独立 Job 完成）。Kubernetes 中用 liveness probe（检测进程是否存活）和 readiness probe（检测是否准备好接收流量），两者实现必须不同——readiness 失败时停止路由流量，liveness 失败时重启 Pod。

**Q37：FastAPI 在生产中常见的性能反模式有哪些？**

在 `async def` 中调用同步库阻塞事件循环；`response_model` 定义不当导致序列化开销过大；数据库查询未使用连接池；中间件链中执行了同步阻塞操作；依赖的 `use_cache` 误用导致重复创建昂贵资源。

---

如果需要针对某个模块深入（如完整的 JWT + OAuth2 实现代码、异步 SQLAlchemy 的 Repository 模式、SSE 流式大模型服务的架构设计），可以继续追问。