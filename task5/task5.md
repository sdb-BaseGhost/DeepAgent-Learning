### 子Agent
解耦，只需要提供给主Agent他想要的消息，具体怎么做主Agent不关心
所以核心设计动机就是上下文隔离

### 子 Agent 最佳实践
1. 描述要具体
主 Agent 靠 description 决定何时以及委派给谁。描述越具体，路由越准确：

# ✅ 好的描述
"description": "执行深度网络研究，需要多次搜索、信息交叉验证和综合分析时使用"

# ❌ 差的描述
"description": "做研究"
2. System Prompt 要详细
特别是要包含输出格式要求和字数限制——这直接影响返回给主 Agent 的内容质量：

"system_prompt": """你是一位研究员。

工具使用指导：
- 用 internet_search 搜索信息，每次查询不同的关键词
- 搜索 3-5 次以获取全面的信息

输出格式：
- 摘要（2-3 段）
- 关键发现（要点列表）
- 信息来源（附 URL）

重要：返回结果控制在 500 字以内。"""
3. 工具集要精简
最小权限原则——只给子 Agent 它需要的工具：

# ✅ 精简：只给研究相关的工具
research_agent = {"tools": [internet_search]}

# ❌ 冗余：给了不需要的工具
research_agent = {"tools": [internet_search, send_email, delete_file, execute_code]}
4. 不同子 Agent 用不同模型
根据任务特点选择合适的模型：

示意片段：internet_search、statistical_analysis、ChatOpenAI 和 os 沿用应用已有定义，代码只对比不同子 Agent 的模型选择。
5. 返回结果要精简
在 system_prompt 中明确要求子 Agent 只返回核心内容：