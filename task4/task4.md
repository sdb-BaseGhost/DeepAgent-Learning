## 目标
让 Agent 在长任务里始终知道“我要做什么、做到哪了、下一步干什么”。长程任务Agent会容易丢失目标和忘记下一步，解决方案就是给Agent一个**TodoList**

上下文长的时候会进行压缩，或者写入第二章提到的虚拟文件系统，但是TodoList不受影响，感觉是类似状态栏的东西

## Middleware
Deep Agents 本质上是给 Agent 挂了一堆 Middleware，把各种工程能力组合起来

## TodoList放在那里
TodoListMiddleware
① 给 Agent 注入 write_todos 工具

② Agent State 增加 todos 字段

③ 给模型增加规划相关提示词

langchain有两种插逻辑方式：
1、Node-style       直接加节点
2、Wrap-style       类似AOP，把执行代码包起来


PatchToolCallsMiddleware
→ 修消息历史：工具调用“有请求没响应”怎么办？
补一条类似：
ToolMessage:
call_1 上次调用被取消，没有拿到响应

TodoListMiddleware
→ 管任务主线：Agent 做到哪、下一步干嘛？

SummarizationMiddleware
→ 管上下文长度：历史太长塞不进模型怎么办？
