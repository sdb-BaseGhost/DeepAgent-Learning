## 异步子 Agent
### 主 Agent 怎么管理子 Agent
1. 跑的进度怎么样
2. 子 Agent 有异常了怎么办
3. 子 Agent 返回的格式是什么
4. 怎么启停子 Agent

注册工具遥控 Agent

### Agent Server 是什么
异步意味着主 Agent 已经返回了，那谁继续帮你跑 Researcher？必须得有一个**独立于当前这次调用生命周期**的东西托管它。
它负责：
    创建任务
    运行 Agent
    保存状态
    查询状态
    取消任务

主 Agent = 调度者
Agent Server = 任务运行平台
子 Agent = 真正干活的人

### thread / run / task_id 到底是什么？
thread = 子 Agent 的会话
run    = 这个会话上的某一次执行
task_id ≈ thread_id

### async_tasks channel
有点像状态栏？DeepAgent 重要信息单独放在 Graph state


### 子 Agent 注册
```python

    from deepagents import AsyncSubAgent, create_deep_agent

    async_subagents = [
        AsyncSubAgent(
            name="researcher",
            description="深度研究 Agent，用于多次搜索 + 信息综合的调研任务",
            graph_id="researcher",
            # 不传 url → 使用 ASGI 进程内传输（与主 Agent 同部署在一个 server）
        ),
        AsyncSubAgent(
            name="coder",
            description="编码 Agent，用于代码生成、改写与代码评审",
            graph_id="coder",
            # url="https://coder-deployment.langsmith.dev"   # 可选：远程 HTTP 传输
        ),
    ]

    agent = create_deep_agent(
        model="google_genai:gemini-3.5-flash",
        subagents=async_subagents,
    )

```

### langgraph.json
```python

  {
    "dependencies": ["./"],
    "graphs": {
      "supervisor": "./graphs/supervisor.py:graph",
      "researcher": "./graphs/researcher.py:graph"
    },
    "env": "./.env"
  }

  它告诉 langgraph dev
  项目依赖在哪里？
  → ./

  我要启动哪些 Agent？
  → supervisor
  → researcher

  它们的代码在哪里？
  → ./graphs/supervisor.py 里的 graph
  → ./graphs/researcher.py 里的 graph
```

### ASGI 和 HTTP

ASGI 全称：Asynchronous Server Gateway Interface
是 Python Web 世界的一套接口规范，正常情况浏览器发起 HTTP 请求后经过uvicorn，按照 ASGI 规范调用 python web应用

本文提到的是：进程内调用 Agent Server vs 通过网络调用 Agent Server
同一 Agent Server 进程内通过 ASGI？为啥要提到这个异步的协议？
HTTP则是子 Agent 不在同一个进程里

**为什么要用 ASGI 呢**，最核心的价值：**统一接口。**
可以根据本地调用和远程调用共享同一套语义。
```
    ASGI 本地：

    Supervisor
    ↓
    POST /threads
    POST /runs
    ↓
    Agent Server


    HTTP 远程：

    Supervisor
    ↓
    POST https://xxx/threads
    POST https://xxx/runs
    ↓
    Agent Server
    
```

TODO:
第六章的最佳实践