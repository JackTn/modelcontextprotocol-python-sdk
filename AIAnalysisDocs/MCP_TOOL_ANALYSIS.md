5. 使用 Sampling 进行智能工具选择# MCP Tool Analysis & Implementation Guide

本文档记录了对 MCP Python SDK 中工具(Tool)属性的分析，以及如何实现智能工具选择和执行前判断的方法。

## 问题 1: Tool 中包含哪些属性

### MCP 协议中的 Tool 类 (`mcp.types.Tool`)

MCP 协议标准的工具定义，继承自 `BaseMetadata`，包含以下属性：

#### 从 BaseMetadata 继承的属性：
- **`name`** (str): 工具的程序化名称
- **`title`** (str | None): 可选的人类可读标题，用于UI和最终用户上下文

#### Tool 特有的属性：
- **`description`** (str | None): 工具的人类可读描述
- **`inputSchema`** (dict[str, Any]): 定义工具期望参数的 JSON Schema 对象
- **`outputSchema`** (dict[str, Any] | None): 可选的 JSON Schema 对象，定义工具输出的结构
- **`icons`** (list[Icon] | None): 可选的图标列表
- **`annotations`** (ToolAnnotations | None): 可选的附加工具信息
- **`meta`** (dict[str, Any] | None): 元数据字段（别名为 `_meta`）

### FastMCP 内部的 Tool 类 (`mcp.server.fastmcp.tools.base.Tool`)

FastMCP 框架内部使用的工具注册信息，包含以下属性：

- **`fn`** (Callable): 工具对应的函数（不包含在序列化中）
- **`name`** (str): 工具名称
- **`title`** (str | None): 人类可读的工具标题
- **`description`** (str): 工具功能描述
- **`parameters`** (dict[str, Any]): 工具参数的 JSON schema
- **`fn_metadata`** (FuncMetadata): 函数元数据，包括参数模型
- **`is_async`** (bool): 工具是否为异步函数
- **`context_kwarg`** (str | None): 应接收上下文的参数名称
- **`annotations`** (ToolAnnotations | None): 可选的工具注解
- **`icons`** (list[Icon] | None): 可选的图标列表
- **`output_schema`** (属性): 从 fn_metadata 获取的输出模式

### ToolAnnotations 类

工具注解提供额外的提示信息：

- **`title`** (str | None): 人类可读的标题
- **`readOnlyHint`** (bool | None): 工具是否为只读
- **`destructiveHint`** (bool | None): 工具是否可能执行破坏性更新
- **`idempotentHint`** (bool | None): 工具是否具有幂等性
- **`openWorldHint`** (bool | None): 工具是否与"开放世界"的外部实体交互

## 问题 2: 工具执行前判断和背景添加的实现方法

### 1. 使用 Prompts 进行工具选择引导

MCP 提供了 Prompts 功能，可以用来引导 LLM 更好地选择和使用工具：

```python
from mcp.server.fastmcp import FastMCP
from mcp.server.fastmcp.prompts import base

mcp = FastMCP("智能工具选择器")

# 创建工具选择引导提示
@mcp.prompt(title="工具选择助手")
def tool_selection_guide(task: str, available_tools: str) -> str:
    return f"""
根据用户任务和可用工具，请分析并选择最合适的工具组合：

用户任务：{task}

可用工具：
{available_tools}

请按以下格式分析：
1. 任务分解：将复杂任务分解为子任务
2. 工具映射：为每个子任务选择合适的工具
3. 执行顺序：确定工具调用的优先级和顺序
4. 依赖关系：分析工具间的数据依赖
5. 风险评估：识别可能的问题和替代方案
"""

# 创建背景信息提示
@mcp.prompt(title="任务背景分析")
def task_context_analyzer(task: str, user_context: str = "") -> list[base.Message]:
    return [
        base.UserMessage(f"任务描述：{task}"),
        base.UserMessage(f"用户背景：{user_context}"),
        base.AssistantMessage("我会分析任务背景，为您提供最佳的工具使用策略。"),
    ]
```

### 2. 实现工具分类和元数据系统

使用工具注解 (ToolAnnotations) 来为工具添加分类和元数据：

```python
from mcp.server.fastmcp import FastMCP
from mcp.types import ToolAnnotations

mcp = FastMCP("分类工具服务器")

# 数据处理类工具
@mcp.tool(annotations=ToolAnnotations(
    title="数据分析工具",
    readOnlyHint=True,  # 只读工具
    openWorldHint=False  # 封闭域工具
))
def analyze_data(data: str) -> str:
    """分析数据并生成报告"""
    return f"数据分析结果：{data}"

# 系统操作类工具
@mcp.tool(annotations=ToolAnnotations(
    title="系统操作工具",
    destructiveHint=True,  # 可能有破坏性
    openWorldHint=True  # 开放域工具
))
def system_operation(command: str) -> str:
    """执行系统操作"""
    return f"执行命令：{command}"

# 工具分类器
@mcp.tool()
def classify_tools_for_task(task_description: str) -> dict[str, list[str]]:
    """根据任务描述对工具进行分类"""
    
    tool_categories = {
        "data_tools": ["analyze_data", "process_data"],
        "system_tools": ["system_operation", "file_operation"],
        "communication_tools": ["send_email", "post_message"]
    }
    
    # 根据任务描述智能选择相关工具类别
    relevant_categories = {}
    if "数据" in task_description or "分析" in task_description:
        relevant_categories["data_tools"] = tool_categories["data_tools"]
    if "系统" in task_description or "操作" in task_description:
        relevant_categories["system_tools"] = tool_categories["system_tools"]
    
    return relevant_categories
```

### 3. 使用 Context 实现工具执行前的验证和准备

利用 Context 对象在工具执行前进行检查和准备：

```python
from mcp.server.fastmcp import Context, FastMCP
from mcp.server.session import ServerSession

mcp = FastMCP("智能工具执行器")

# 工具执行前置检查器
@mcp.tool()
async def execute_with_validation(
    tool_name: str, 
    arguments: dict, 
    ctx: Context[ServerSession, None]
) -> str:
    """在执行工具前进行验证和背景分析"""
    
    # 1. 记录执行背景
    await ctx.info(f"准备执行工具：{tool_name}")
    await ctx.debug(f"参数：{arguments}")
    
    # 2. 工具分类判断
    tool_category = get_tool_category(tool_name)
    await ctx.info(f"工具类别：{tool_category}")
    
    # 3. 风险评估
    risk_level = assess_tool_risk(tool_name, arguments)
    if risk_level == "high":
        # 请求用户确认
        confirmation = await ctx.elicit(
            message=f"工具 {tool_name} 具有高风险，是否继续执行？",
            schema={"type": "object", "properties": {"confirm": {"type": "boolean"}}}
        )
        if not confirmation.data.get("confirm"):
            return "用户取消了高风险工具的执行"
    
    # 4. 执行工具
    result = await execute_actual_tool(tool_name, arguments, ctx)
    
    # 5. 记录执行结果
    await ctx.info(f"工具执行完成：{tool_name}")
    
    return result

def get_tool_category(tool_name: str) -> str:
    """获取工具类别"""
    categories = {
        "data": ["analyze_data", "process_data"],
        "system": ["system_operation", "file_operation"],
        "communication": ["send_email", "post_message"]
    }
    
    for category, tools in categories.items():
        if tool_name in tools:
            return category
    return "unknown"

def assess_tool_risk(tool_name: str, arguments: dict) -> str:
    """评估工具风险等级"""
    high_risk_tools = ["system_operation", "delete_file"]
    if tool_name in high_risk_tools:
        return "high"
    return "low"

async def execute_actual_tool(tool_name: str, arguments: dict, ctx: Context) -> str:
    """实际执行工具的逻辑"""
    # 这里实现实际的工具调用逻辑
    return f"执行了工具 {tool_name}，参数：{arguments}"
```

### 4. 实现工具链和依赖管理

创建一个工具链管理器来处理工具间的依赖关系：

```python
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class ToolStep:
    tool_name: str
    arguments: dict
    dependencies: List[str] = None  # 依赖的前置工具
    condition: str = None  # 执行条件

class ToolChainManager:
    def __init__(self):
        self.execution_results = {}
    
    def plan_execution(self, task: str) -> List[ToolStep]:
        """根据任务规划工具执行链"""
        if "数据分析" in task:
            return [
                ToolStep("load_data", {"source": "database"}),
                ToolStep("analyze_data", {"method": "statistical"}, 
                        dependencies=["load_data"]),
                ToolStep("generate_report", {"format": "pdf"}, 
                        dependencies=["analyze_data"])
            ]
        return []
    
    async def execute_chain(self, steps: List[ToolStep], ctx: Context) -> dict:
        """按依赖顺序执行工具链"""
        results = {}
        
        for step in steps:
            # 检查依赖
            if step.dependencies:
                for dep in step.dependencies:
                    if dep not in results:
                        await ctx.error(f"依赖工具 {dep} 未执行")
                        continue
            
            # 执行工具
            await ctx.info(f"执行工具：{step.tool_name}")
            result = await self.execute_single_tool(step, ctx)
            results[step.tool_name] = result
            
        return results
    
    async def execute_single_tool(self, step: ToolStep, ctx: Context) -> str:
        """执行单个工具"""
        return f"执行了 {step.tool_name}"

# 在 FastMCP 中使用
mcp = FastMCP("工具链管理器")
chain_manager = ToolChainManager()

@mcp.tool()
async def execute_task_chain(task: str, ctx: Context[ServerSession, None]) -> dict:
    """执行任务工具链"""
    # 1. 分析任务并规划工具链
    steps = chain_manager.plan_execution(task)
    await ctx.info(f"规划了 {len(steps)} 个工具步骤")
    
    # 2. 执行工具链
    results = await chain_manager.execute_chain(steps, ctx)
    
    return {
        "task": task,
        "steps_executed": len(steps),
        "results": results
    }
```

### 5. 使用 Sampling 进行智能工具选择

利用 MCP 的 Sampling 功能让 LLM 协助进行工具选择：

```python
from mcp.types import SamplingMessage, TextContent

@mcp.tool()
async def intelligent_tool_selector(
    user_request: str, 
    available_tools: List[str],
    ctx: Context[ServerSession, None]
) -> dict:
    """使用 LLM 智能选择工具"""
    
    prompt = f"""
用户请求：{user_request}
可用工具：{', '.join(available_tools)}

请分析用户请求并选择最合适的工具组合。
返回 JSON 格式：
{{
    "selected_tools": ["tool1", "tool2"],
    "execution_order": [1, 2],
    "reasoning": "选择原因"
}}
"""
    
    # 使用 Sampling 请求 LLM 分析
    result = await ctx.session.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(type="text", text=prompt)
            )
        ],
        max_tokens=200
    )
    
    if result.content.type == "text":
        # 解析 LLM 的响应
        import json
        try:
            analysis = json.loads(result.content.text)
            return analysis
        except json.JSONDecodeError:
            return {"error": "LLM 响应格式错误"}
    
    return {"error": "无法获取 LLM 分析"}
```

## 总结

### Tool 属性总结

MCP 工具系统提供了丰富的属性来描述和控制工具行为：
- 基本信息（名称、标题、描述）
- 输入输出模式定义
- 执行提示（只读、破坏性、幂等性等）
- 元数据和图标支持

### 智能工具选择实现方法

基于 MCP Python SDK 的功能，可以通过以下方式实现工具调用前的判断和背景添加：

1. **Prompts** - 创建引导性提示来帮助工具选择
2. **ToolAnnotations** - 为工具添加分类和元数据
3. **Context** - 在工具执行前进行验证和准备
4. **工具链管理** - 实现复杂的工具依赖和执行顺序
5. **Sampling** - 利用 LLM 进行智能工具选择

这些方法可以单独使用或组合使用，根据具体需求选择最合适的实现方案。

## MCP Planner 相关信息

根据对 MCP Python SDK 代码库的全面搜索和分析，**没有发现任何关于 "MCP planner" 的具体信息或实现**。

### 搜索结果
- 没有文件名包含 "planner" 或 "plan"
- 代码中没有 "planner" 关键字
- 没有相关的规划、任务编排或多步骤任务管理的组件

### 代码库主要组件
MCP Python SDK 主要提供以下核心功能：
- MCP 服务器实现 (FastMCP, 低级服务器)
- MCP 客户端 (连接到 MCP 服务器)
- 工具 (Tools) - 执行特定功能的函数
- 资源 (Resources) - 提供数据访问
- 提示 (Prompts) - 交互模板
- 传输层 (stdio, SSE, Streamable HTTP)
- 认证支持 (OAuth)
- CLI 工具 (开发和部署)

### 结论
MCP Python SDK 本身**没有内置的 planner 功能**。如果需要规划功能，可以：
1. 在 MCP 服务器之上构建自己的规划逻辑
2. 使用其他专门的任务规划库
3. 在客户端实现规划算法来协调多个 MCP 工具的调用

---

*文档创建时间：2025年10月15日*