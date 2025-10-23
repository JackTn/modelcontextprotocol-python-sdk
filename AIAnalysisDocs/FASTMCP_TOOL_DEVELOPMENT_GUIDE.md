# FastMCP Tool 开发指南

## 目录
- [概述](#概述)
- [FastMCP 版本信息](#fastmcp-版本信息)
- [FastMCP 架构说明](#fastmcp-架构说明)
- [Tool 开发最佳实践](#tool-开发最佳实践)
- [参数描述与验证](#参数描述与验证)
- [多列表处理工具](#多列表处理工具)
- [ToolManager vs FastMCP](#toolmanager-vs-fastmcp)
- [实际示例](#实际示例)

## 概述

FastMCP 是 MCP (Model Context Protocol) Python SDK 中的高级接口，提供了一个更加人性化和便于使用的服务器开发框架。它封装了底层的 MCP 协议细节，让开发者能够通过简洁的装饰器语法快速构建功能强大的 MCP 服务器。

## FastMCP 版本信息

### 版本管理方式
- **FastMCP 不是独立的版本化组件**，它是 MCP Python SDK 的一部分
- 项目名称是 `mcp`（不是单独的 fastmcp）
- 使用动态版本管理：`dynamic = ["version"]`
- 版本通过 Git 标签自动生成：`vcs = "git"`, `style = "pep440"`

### 版本获取方法
```bash
# 方法1：通过 pip 查看已安装的 mcp 包版本
pip show mcp

# 方法2：通过 Git 标签查看
git describe --tags

# 方法3：在 Python 中导入查看
python -c "import mcp; print(mcp.__version__)" # 如果有的话
```

### 项目结构层级
- **Low-level Server**: `mcp.server.lowlevel` - 底层接口
- **FastMCP**: `mcp.server.fastmcp` - 高级接口
- **Client**: `mcp.client` - 客户端

## FastMCP 架构说明

### 核心文件功能
`src/mcp/server/fastmcp/server.py` 是 FastMCP 框架的核心服务器实现，提供了更友好的 MCP 服务器接口。

### 主要类说明

#### 1. `Settings` 类
- **作用**: FastMCP 服务器的配置管理类
- **功能**: 
  - 管理服务器设置（调试模式、日志级别、主机端口等）
  - 支持环境变量配置（前缀 `FASTMCP_`）
  - HTTP 传输设置（SSE、StreamableHTTP 路径配置）
  - 认证和安全设置

#### 2. `FastMCP` 类（核心类）
- **作用**: FastMCP 框架的主要服务器类
- **功能**:
  - 封装底层 MCP 服务器（`MCPServer`）
  - 管理工具、资源、提示的注册和执行
  - 提供装饰器接口（`@tool`, `@resource`, `@prompt`）
  - 支持多种传输协议（stdio、SSE、StreamableHTTP）
  - 处理认证和权限管理
  - 提供服务器生命周期管理

**主要方法**:
- `tool()`: 注册工具的装饰器
- `resource()`: 注册资源的装饰器  
- `prompt()`: 注册提示的装饰器
- `run()`: 启动服务器
- `custom_route()`: 添加自定义 HTTP 路由

#### 3. `Context` 类
- **作用**: 为工具和资源函数提供运行时上下文
- **功能**:
  - 提供日志记录接口（`debug()`, `info()`, `warning()`, `error()`）
  - 进度报告（`report_progress()`）
  - 资源访问（`read_resource()`）
  - 用户交互（`elicit()`）
  - 获取请求信息（`request_id`, `client_id`）

## Tool 开发最佳实践

### 1. 基础工具示例
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Demo Server")

@mcp.tool()
def sum(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
```

### 2. 带参数描述的工具（推荐）
```python
from pydantic import Field
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Parameter Descriptions Server")

@mcp.tool()
def greet_user(
    name: str = Field(description="The name of the person to greet"),
    title: str = Field(description="Optional title like Mr/Ms/Dr", default=""),
    times: int = Field(description="Number of times to repeat the greeting", default=1),
) -> str:
    """Greet a user with optional title and repetition"""
    greeting = f"Hello {title + ' ' if title else ''}{name}!"
    return "\n".join([greeting] * times)
```

### 3. 异步工具与 Context（高级）
```python
import asyncio
from mcp.server.fastmcp import Context, FastMCP

mcp = FastMCP("Echo Server with logging and progress updates")

@mcp.tool()
async def echo(text: str, ctx: Context) -> str:
    """Echo the input text sending log messages and progress updates during processing."""
    await ctx.report_progress(progress=0, total=100)
    await ctx.info("Starting to process echo for input: " + text)
    
    await asyncio.sleep(2)
    
    await ctx.info("Halfway through processing echo for input: " + text)
    await ctx.report_progress(progress=50, total=100)
    
    await asyncio.sleep(2)
    
    await ctx.info("Finished processing echo for input: " + text)
    await ctx.report_progress(progress=100, total=100)
    
    return text
```

### 4. 结构化输出工具（最佳实践）
```python
from datetime import datetime
from pydantic import BaseModel, Field
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Weather Service")

class WeatherData(BaseModel):
    """Structured weather data response"""
    temperature: float = Field(description="Temperature in Celsius")
    humidity: float = Field(description="Humidity percentage (0-100)")
    condition: str = Field(description="Weather condition (sunny, cloudy, rainy, etc.)")
    wind_speed: float = Field(description="Wind speed in km/h")
    location: str = Field(description="Location name")
    timestamp: datetime = Field(default_factory=datetime.now, description="Observation time")

@mcp.tool()
def get_weather(city: str) -> WeatherData:
    """Get current weather for a city with full structured data"""
    return WeatherData(
        temperature=22.5, 
        humidity=65.0, 
        condition="partly cloudy", 
        wind_speed=12.3, 
        location=city
    )
```

## 参数描述与验证

### `description` vs `annotations` 区别

| 属性 | `description` | `annotations` |
|------|--------------|---------------|
| **性质** | 人类可读文本 | 结构化元数据 |
| **用途** | 说明工具功能 | 描述工具特征 |
| **读者** | 人类用户 + LLM | 主要是 LLM 系统 |
| **内容** | 详细功能说明、用法示例 | 行为特征、安全提示 |
| **影响** | 帮助理解工具用途 | 影响工具使用策略 |
| **格式** | 自由文本（支持 Markdown） | 固定的布尔值字段 |

### ToolAnnotations 属性

```python
class ToolAnnotations(BaseModel):
    title: str | None = None                # 人类可读的标题
    readOnlyHint: bool | None = None        # 是否只读（不修改环境）
    destructiveHint: bool | None = None     # 是否可能有破坏性更新
    idempotentHint: bool | None = None      # 是否具有幂等性
    openWorldHint: bool | None = None       # 是否与"开放世界"交互
```

### 完整示例：数据查询工具
```python
from mcp.types import ToolAnnotations

@mcp.tool(
    name="query_database",
    title="数据库查询工具",
    description="""
    查询数据库并返回结果。支持 SQL 查询语句，但仅限于 SELECT 操作。
    
    示例用法：
    - 查询用户信息: SELECT * FROM users WHERE age > 18
    - 统计订单数量: SELECT COUNT(*) FROM orders WHERE status = 'completed'
    
    注意：此工具是只读的，不会修改任何数据。
    """,
    annotations=ToolAnnotations(
        title="安全数据库查询",        # 覆盖上面的 title
        readOnlyHint=True,           # 只读，不修改环境
        destructiveHint=False,       # 非破坏性
        idempotentHint=True,         # 相同查询返回相同结果
        openWorldHint=False          # 封闭域，只与数据库交互
    )
)
def query_database(sql: str, ctx: Context) -> dict:
    """执行数据库查询"""
    # 实际的查询逻辑...
    return {"rows": [], "count": 0}
```

### Pydantic Field 详解

#### Field 的作用
`Field` 是 Pydantic 库提供的函数，用于为模型字段或函数参数添加额外的元数据信息。

#### Field 常用参数

| 参数 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `description` | str | 参数描述 | `description="用户姓名"` |
| `default` | any | 默认值 | `default="匿名用户"` |
| `ge` | int/float | 大于等于 | `ge=0` |
| `le` | int/float | 小于等于 | `le=100` |
| `gt` | int/float | 大于 | `gt=0` |
| `lt` | int/float | 小于 | `lt=100` |
| `min_length` | int | 最小长度 | `min_length=1` |
| `max_length` | int | 最大长度 | `max_length=50` |
| `pattern` | str | 正则表达式 | `pattern=r'^[a-zA-Z]+$'` |

#### 高级参数示例
```python
@mcp.tool()
def create_user_profile(
    # 必填字符串参数
    username: str = Field(
        description="用户名，用于登录和标识用户",
        min_length=3,
        max_length=20,
        pattern=r'^[a-zA-Z0-9_]+$'  # 只允许字母、数字、下划线
    ),
    
    # 必填邮箱参数
    email: str = Field(
        description="用户邮箱地址，用于接收通知和密码重置",
        pattern=r'^[^@]+@[^@]+\.[^@]+$'
    ),
    
    # 可选年龄参数
    age: int = Field(
        description="用户年龄，用于个性化推荐",
        ge=13,    # 最小13岁
        le=120,   # 最大120岁
        default=18
    ),
) -> dict[str, any]:
    """创建用户档案"""
    # 实现逻辑...
    pass
```

## 多列表处理工具

### 使用 Pydantic Field 定义列表参数

```python
@mcp.tool()
def process_multiple_lists(
    numbers: list[int] = Field(description="List of numbers to process"),
    strings: list[str] = Field(description="List of strings to process"), 
    flags: list[bool] = Field(description="List of boolean flags")
) -> dict[str, Any]:
    """Process multiple lists and return summary."""
    return {
        "number_sum": sum(numbers),
        "string_count": len(strings),
        "true_flags": sum(flags),
        "total_items": len(numbers) + len(strings) + len(flags)
    }
```

### 列表参数验证
```python
from typing import Annotated

@mcp.tool()
def validate_lists(
    ids: Annotated[list[int], Field(min_length=1, description="List of IDs (at least one required)")],
    names: Annotated[list[str], Field(max_length=100, description="List of names (max 100 items)")],
) -> dict[str, Any]:
    """Tool with validated list parameters."""
    return {
        "processed_pairs": [{"id": id_val, "name": name} for id_val, name in zip(ids, names)],
        "total_processed": min(len(ids), len(names))
    }
```

### 带进度报告的批量处理
```python
@mcp.tool()
async def batch_process_with_progress(
    items: list[str] = Field(description="List of items to process"),
    ctx: Context
) -> str:
    """Process a list with progress reporting."""
    total = len(items)
    results = []
    
    for i, item in enumerate(items):
        # 报告进度
        await ctx.report_progress(i + 1, total, f"Processing item {i + 1}/{total}")
        
        # 处理项目
        processed = f"Processed: {item}"
        results.append(processed)
    
    return f"Completed processing {total} items:\n" + "\n".join(results)
```

## ToolManager vs FastMCP

### 层级关系
- **基本上相同**：两者的 `add_tool` 方法参数和功能相同
- **层级关系**：FastMCP 内部调用 ToolManager

```python
# FastMCP.add_tool() 实际上调用 ToolManager.add_tool()
def add_tool(self, fn, **kwargs):
    self._tool_manager.add_tool(fn, **kwargs)  # 委托给 ToolManager
```

### ToolManager 功能
1. **工具注册**：`add_tool()` 方法注册新工具
2. **工具列表**：`list_tools()` 方法返回可用工具列表  
3. **工具执行**：`call_tool()` 方法调用指定工具
4. **工具查找**：根据名称查找和管理工具

### MCP 中没有内置 Planner
- MCP SDK 专注于工具、资源、提示的管理，不提供任务规划能力
- 所有工具调用都是直接的、单次的执行
- 真正的"编排"在客户端层面实现（由 LLM 决定）

## 实际示例

### 文件读取描述示例
```python
def read_tool_description(tool_name: str) -> str:
    """从 Markdown 文件读取工具描述"""
    try:
        with open(f"descriptions/{tool_name}.md", 'r', encoding='utf-8') as f:
            return f.read().strip()
    except FileNotFoundError:
        return f"Description for {tool_name} not found"

@mcp.tool(
    name="calculate_sum",
    description=read_tool_description("calculate_sum")  # 从文件读取
)
def calculate_sum(a: int, b: int) -> int:
    return a + b
```

### 用户交互工具
```python
from pydantic import BaseModel, Field

class ConfirmationSchema(BaseModel):
    confirmed: bool = Field(description="用户确认结果")
    reason: str = Field(description="确认或拒绝的原因", default="")

@mcp.tool(
    description="执行需要用户确认的危险操作",
    annotations=ToolAnnotations(destructiveHint=True)
)
async def dangerous_operation(
    operation: str,
    target: str,
    ctx: Context
) -> str:
    """执行危险操作前请求用户确认"""
    
    # 请求用户确认
    result = await ctx.elicit(
        message=f"确认执行操作 '{operation}' 在目标 '{target}' 上？这个操作不可逆！",
        schema=ConfirmationSchema
    )
    
    if result.action == "accept" and result.data and result.data.confirmed:
        await ctx.warning(f"用户确认执行: {operation}")
        # 执行实际操作...
        return f"已执行 {operation} 在 {target}"
    else:
        await ctx.info("用户取消了危险操作")
        return "操作已取消"
```

## 最佳实践总结

1. **使用 Pydantic Field** 为参数提供详细描述
2. **结构化输出** 使用 Pydantic 模型或 TypedDict
3. **异步操作** 配合 Context 提供进度和日志
4. **用户交互** 使用 `ctx.elicit()` 进行确认
5. **错误处理** 合理使用 try/catch 和日志
6. **类型注解** 充分利用 Python 类型系统
7. **文档字符串** 提供清晰的功能说明
8. **ToolAnnotations** 为系统提供工具行为提示
9. **description** 写给人看，要详细、有示例
10. **annotations** 写给系统看，要准确、反映实际行为

---

*本文档基于 MCP Python SDK 源代码分析整理，涵盖了 FastMCP 工具开发的核心概念和最佳实践。*