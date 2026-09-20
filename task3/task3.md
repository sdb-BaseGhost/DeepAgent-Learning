### 1、为什么要用虚拟文件系统，只靠文件系统不行吗？

只使用真实文件系统当然可以保存数据，但它存在一个问题：**LLM 会直接和具体的存储方式耦合**。

例如同样是保存一个文件：

- 有时我们希望它只是当前 Agent 运行过程中的临时数据；
- 有时希望它真正写到本地磁盘；
- 有时希望长期保存到数据库；
- 有时还希望保存到沙箱环境中。

如果只使用真实文件系统，那么所有内容都必须真正落盘，而且 Agent 还需要知道具体文件保存在哪里、底层是什么存储方式。

Deep Agents 使用虚拟文件系统，就是在 LLM 和具体存储之间增加了一层抽象。

对于 LLM 来说，它只需要使用统一的文件工具，例如：

```text
write_file
read_file
edit_file
ls
grep
```

并操作类似：

```text
/workspace/a.md
/memories/user.md
```

这样的虚拟路径。

至于这些“文件”实际上存在哪里，由 Backend 决定。

例如：

```text
/workspace/a.md
        ↓
StateBackend
        ↓
LangGraph State
```

也可能是：

```text
/workspace/a.md
        ↓
FilesystemBackend
        ↓
本地磁盘
```

因此虚拟文件系统的核心作用不是“替代真实文件系统”，而是：

> **给 LLM 提供统一的文件操作接口，同时屏蔽底层具体存储实现。**

这样 Agent 只关注“我要读取什么、保存什么”，而不需要关注这些数据到底存在 State、本地磁盘还是数据库中。

------

### 2、上下文如何自动管理

Deep Agents 引入虚拟文件系统，一个很重要的目的就是进行 **Context Engineering**。

LLM 的上下文窗口是有限的。如果所有历史对话、工具调用结果、搜索结果、中间过程都一直保存在 messages 中，上下文会越来越长。

因此 Deep Agents 会把：

```text
当前真正需要的信息
```

放在 LLM Context 中，

而将：

```text
暂时不需要，但未来可能还会使用的信息
```

放到虚拟文件系统中。

整体思路可以理解为：

```text
LLM Context = 内存

Virtual Filesystem = 外部存储
```

需要时再通过 `read_file`、`grep` 等工具读取。

------

#### 上下文过长怎么办

随着 Agent 执行时间变长，messages 中会不断增加：

```text
UserMessage
AIMessage
ToolMessage
UserMessage
AIMessage
...
```

当上下文长度达到一定阈值后，Deep Agents 的 `SummarizationMiddleware` 会触发自动总结。

流程大致是：

```text
messages 越来越长
        ↓
检测 token 数量
        ↓
超过上下文阈值
        ↓
取出较早的历史消息
        ↓
使用 LLM 生成 Summary
        ↓
保留 Summary + 最近消息
```

例如原来的上下文：

```text
消息1
消息2
消息3
...
消息200
消息201
消息202
```

经过总结以后可能变成：

```text
Summary：
用户正在开发某项目，
已经完成 A、B，
当前遇到 C 问题。

最近消息201
最近消息202
```

这样就可以大幅减少 Context 中占用的 Token。

所以：

> **是否需要总结通常由程序根据上下文长度判断，而总结内容本身由 LLM 生成。**

也就是：

```text
什么时候总结 → 框架判断
总结成什么 → LLM 判断
```

------

#### 外部结果返回过长怎么办

Agent 调用搜索、网页读取、数据库查询等工具时，也可能一次返回非常大的结果。

例如：

```text
search()
↓
返回 30000 tokens
```

如果这些内容全部直接塞入 messages，会迅速占满 Context。

因此 Deep Agents 会对大型工具结果进行 **offload（卸载）**。

流程类似：

```text
工具返回大型结果
        ↓
Middleware 检查大小
        ↓
结果超过阈值
        ↓
完整结果写入虚拟文件系统
        ↓
Context 中只留下：
文件地址 + 少量预览
```

例如完整结果可能保存成：

```text
/workspace/search_result.md
```

LLM 此时只看到：

```text
完整搜索结果已保存到：
/workspace/search_result.md
```

如果后面需要其中一部分，再执行：

```text
grep
↓
找到位置
↓
read_file(offset, limit)
```

按需读取。

这就避免了：

```text
把 30000 tokens 全部塞进上下文
```

而变成：

```text
先保存
↓
需要多少读多少
```

这也是虚拟文件系统在 Context Engineering 中最重要的作用之一。

------

### 3、虚拟文件都有哪些文件，这些文件都存在哪里？

虚拟文件并不是一种固定类型的文件。

只要 Agent 通过虚拟文件系统保存的信息，都可以表现成“文件”。

例如可能包括：

```text
/workspace/plan.md
/workspace/search_result.md
/workspace/code.txt
/workspace/report.md

/memories/user_preferences.md

/conversation_history/history.md
```

这些文件可以大致分成几类。

第一类是临时工作文件，例如：

```text
plan.md
todo.md
research.md
```

通常保存 Agent 当前任务中的：

- 中间结果
- 草稿
- TODO
- 搜索结果
- 临时分析

这类文件可以使用 `StateBackend`。

实际存储位置是：

```text
LangGraph State
```

因此它虽然看起来是：

```text
/workspace/plan.md
```

但宿主机磁盘上不一定真的存在这个文件。

------

第二类是真实项目文件，例如：

```text
main.py
config.yaml
README.md
```

这类文件可以使用：

```text
FilesystemBackend
```

实际存储在：

```text
本地文件系统
```

例如虚拟路径：

```text
/workspace/backend_test.md
```

可能映射到真实路径：

```text
./fs_data/workspace/backend_test.md
```

这种 Backend 比较适合：

- Coding Agent
- 修改项目代码
- 修改配置文件
- 本地开发任务

------

第三类是长期数据，例如：

```text
/memories/preferences.md
```

可能保存：

- 用户偏好
- 长期记忆
- 跨任务信息

这种文件可以使用：

```text
StoreBackend
```

实际数据可能保存到：

```text
持久化 Store
数据库
其他长期存储系统
```

从而支持不同 conversation/thread 之间继续使用。

------

此外，还可以通过：

```text
CompositeBackend
```

让不同虚拟路径使用不同 Backend。

例如：

```text
/
├── workspace/
│   └── plan.md
│       ↓
│   StateBackend
│
└── memories/
    └── user.md
        ↓
    StoreBackend
```

因此最终可以总结为：

```text
LLM看到的是统一的虚拟文件路径

        ↓

Backend 决定实际存储位置

        ↓

StateBackend
→ LangGraph State

FilesystemBackend
→ 本地磁盘

StoreBackend
→ 持久化 Store / 数据库

Sandbox Backend
→ 沙箱环境
```

所以最重要的一点是：

> **虚拟文件只是 LLM 看到的逻辑表示，文件实际存在哪里取决于配置的 Backend。**

```python
from pathlib import Path
import os
from langchain_openai import ChatOpenAI
from deepagents import create_deep_agent
from deepagents.backends import StateBackend, FilesystemBackend

from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

# 加载当前项目下的 .env
load_dotenv()

model = ChatOpenAI(
    model=os.getenv("MODEL_NAME", "Qwen/Qwen2.5-7B-Instruct"),
    api_key=os.getenv("SILICONFLOW_API_KEY"),
    base_url=os.getenv(
        "SILICONFLOW_BASE_URL",
        "https://api.siliconflow.cn/v1",
    ),
)


TASK = """
请严格完成下面的文件操作实验：

1. 使用 write_file 创建 /workspace/backend_test.md
   内容为：
   # Deep Agents Backend Test
   Deep Agents 使用虚拟文件系统管理上下文。
   当前状态：待修改

2. 使用 ls 查看 /workspace 目录。

3. 使用 grep 搜索文件中的字符串 "Deep Agents"。

4. 使用 edit_file，把：
   当前状态：待修改
   修改为：
   当前状态：实验完成

5. 最后使用 read_file 读取 /workspace/backend_test.md，
   并告诉我最终文件内容。

必须实际调用文件工具完成，不要只在回答中模拟。
"""


def print_result(result):
    """只打印 Agent 最后一条回复。"""
    messages = result["messages"]
    print(messages[-1].content)


# =========================================================
# 实验 A：StateBackend
# =========================================================

print("\n" + "=" * 60)
print("实验 A：StateBackend")
print("=" * 60)

state_agent = create_deep_agent(
    model=model,
    backend=StateBackend(),
)

state_result = state_agent.invoke(
    {
        "messages": [
            {
                "role": "user",
                "content": TASK,
            }
        ]
    }
)

print_result(state_result)

print("\n检查宿主机：")

local_state_file = Path("./workspace/backend_test.md")

print(
    "宿主机是否存在 ./workspace/backend_test.md:",
    local_state_file.exists(),
)


# =========================================================
# 实验 B：FilesystemBackend
# =========================================================

print("\n" + "=" * 60)
print("实验 B：FilesystemBackend")
print("=" * 60)

root_dir = Path("./fs_data")
root_dir.mkdir(exist_ok=True)

filesystem_agent = create_deep_agent(
    model=model,
    backend=FilesystemBackend(
        root_dir=root_dir,
        virtual_mode=True,
    ),
)

filesystem_result = filesystem_agent.invoke(
    {
        "messages": [
            {
                "role": "user",
                "content": TASK,
            }
        ]
    }
)

print_result(filesystem_result)


# =========================================================
# 检查真实磁盘
# =========================================================

real_file = root_dir / "workspace" / "backend_test.md"

print("\n检查宿主机：")
print("真实文件路径：", real_file.resolve())
print("文件是否存在：", real_file.exists())

if real_file.exists():
    print("\n磁盘中的真实内容：")
    print(real_file.read_text(encoding="utf-8"))
```

