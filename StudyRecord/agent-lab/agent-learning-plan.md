# Agent Lab 学习计划 —— 从零到实习 Offer

> **启动日期**：2026年7月27日（周一）
> **目标截止**：2026年12月中旬
> **总周数**：20 周
> **目标岗位**：中大厂 AI Agent / 大模型应用开发 实习
>
> **每日时间分配**：
> - 上午 2h：LeetCode 刷题
> - 下午 3h：项目开发
> - 晚上 1h：LLM 理论 / 技术阅读 / 笔记输出

---

## 总览时间线

```
         7月      8月      9月      10月     11月     12月
项目：   ████░░░░ ████████ ████████ ████████ ████████ ██░░░░░░
        Bootcamp  MVP聊天  工具使用  代码助手  部署打磨  面试冲刺
                       ↑                ↑         ↑
                    可演示V1         可演示V2   上线版本

算法：   数组哈希  双指针   链表树    DP专题   回溯贪心  模拟面试

理论：   Karpathy  Agent    Prompt    RAG概览  系统设计  面经
        视频      设计文    工程深入
```

---

## 项目路线图

---

### 第 1-2 周（7/27 - 8/9）：技能预热 —— 让前后端跑通

#### 本周目标
从零到能写出一个 React 页面 + FastAPI 接口，并让它们通信。

#### 具体任务

**React 前端（第 1 周重点，约 18h）**

| 天 | 任务 | 产出 |
|----|------|------|
| Day 1-2 | [react.dev](https://react.dev/learn) "Quick Start" 章节 | 理解组件、JSX、Props |
| Day 3-4 | "Adding Interactivity" 章节 | 理解 useState、事件处理 |
| Day 5-6 | "Managing State" 章节 | 理解 useReducer、状态提升 |
| Day 7 | 写一个静态聊天界面（无后端） | 组件树：App → Sidebar + ChatArea → MessageList + InputBox |
| Day 8-9 | 学习 `fetch` / `EventSource`（SSE）| 理解 HTTP 请求和流式响应 |
| Day 10 | 把静态聊天界面改成从 mock 数据渲染 | 消息列表能滚动，能发新消息 |

**FastAPI 后端（第 2 周重点，约 12h）**

| 天 | 任务 | 产出 |
|----|------|------|
| Day 1-2 | [FastAPI 官方教程](https://fastapi.tiangolo.com/zh/tutorial/) 前 6 章 | 路径操作、Pydantic、async |
| Day 3 | 写 `GET /api/health` 和 `POST /api/chat` | `/chat` 接收 `{message}`，返回固定回复 |
| Day 4 | 配置 CORS 中间件 | 前端能跨域调通后端 |
| Day 5-6 | 接入 DeepSeek API（httpx + 环境变量管理 API Key） | `/chat` 能真正调用 LLM 返回结果 |

**前后端联调（第 2 周末，约 6h）**

- React 调 FastAPI `/api/chat`，发消息 → 调 LLM → 显示回复
- 完成第一个"能用"的闭环

**LLM 理论（夜间 1h，共约 12h）**

| 内容 | 时间 | 笔记要点 |
|------|------|----------|
| [Karpathy: Let's build GPT from scratch](https://www.youtube.com/watch?v=kCc8FmEb1nY) | 2 晚 | Transformer 的结构直觉、Self-Attention 做了什么 |
| [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | 1 晚 | Agent 设计的核心原则（简单 > 复杂、工作流 vs 自主Agent） |
| OpenAI Prompt Engineering Guide | 1 晚 | System prompt、few-shot、CoT 的实践技巧 |
| 注册 DeepSeek / 各平台 API，玩一遍 Playground | 1 晚 | 直观感受不同参数（temperature、top_p）的效果 |

#### 第 1-2 周里程碑
> ✅ 浏览器输入文字 → 后端调用 LLM → 返回回复 → 前端展示
>
> ✅ 理解了 Transformer + Agent 设计原则

---

### 第 3-5 周（8/10 - 8/30）：MVP —— 单 Agent 流式对话

#### 本周目标
做出一个有流式输出、多角色、对话持久化的聊天应用。**这是你第一个可以给人 demo 的版本。**

#### 后端任务

**流式输出（SSE）—— 这是 Agent 应用的标配（约 12h）**

```python
# 核心实现思路（手写，不调包）
from fastapi.responses import StreamingResponse
import httpx

async def stream_chat(message: str, history: list[dict]):
    async with httpx.AsyncClient() as client:
        async with client.stream(
            "POST",
            "https://api.deepseek.com/v1/chat/completions",
            headers={"Authorization": f"Bearer {API_KEY}"},
            json={
                "model": "deepseek-chat",
                "messages": history + [{"role": "user", "content": message}],
                "stream": True,
            },
        ) as response:
            async for chunk in response.aiter_lines():
                if chunk.startswith("data: "):
                    data = json.loads(chunk[6:])
                    delta = data["choices"][0]["delta"].get("content", "")
                    if delta:
                        yield f"data: {json.dumps({'content': delta})}\n\n"
```

关键点：
- 用 `httpx` 而非 `requests`（需要真正的异步）
- 理解 SSE 协议：`data: xxx\n\n` 格式
- 错误处理：API 超时、token 耗尽、网络中断

**Agent 管理系统（约 10h）**

```
数据模型（Pydantic + SQLite）：
Agent:
  - id: str
  - name: str           # "代码助手"、"旅游规划师"
  - system_prompt: str  # 角色设定
  - model: str          # deepseek-chat / gpt-4o-mini
  - temperature: float
  - created_at: datetime

Conversation:
  - id: str
  - agent_id: str       # 外键
  - title: str
  - created_at: datetime

Message:
  - id: str
  - conversation_id: str
  - role: str           # user / assistant
  - content: str
  - created_at: datetime
```

**预置 3 个 Agent 角色（约 6h）**

| Agent | System Prompt 要点 |
|-------|-------------------|
| **代码助手** | 资深软件工程师，擅长 Python/JS，解释代码、debug、优化建议；回复使用 Markdown 代码块 |
| **旅游规划师** | 热情细心的旅行达人，根据预算/天数/偏好制定行程；推荐小众景点和地道美食 |
| **苏格拉底导师** | 用提问引导思考，不直接给答案；帮助理解概念的本质而非记忆 |

#### 前端任务

**核心组件开发（约 18h）**

```
组件树：
App
├── Sidebar
│   ├── AgentList          ← 展示所有 Agent，可切换
│   │   └── AgentCard      ← 头像 + 名称 + 简介
│   └── ConversationList   ← 当前 Agent 的历史对话
│       └── ConversationItem
├── ChatArea
│   ├── ChatHeader         ← 当前 Agent 名称 + 操作按钮
│   ├── MessageList        ← 自动滚动到底部
│   │   └── MessageBubble  ← 用户消息（右）/ AI消息（左）
│   └── InputBox           ← 输入框 + 发送按钮 + 新建对话
```

关键实现：
- **流式渲染**：后端 SSE 推到哪，前端字就显示到哪（打字机效果）
- **Markdown 渲染**：安装 `react-markdown` + `react-syntax-highlighter`，代码块要高亮
- **自动滚动**：新消息来时自动滚到底，但如果用户往上翻看历史则不强制滚动
- **消息状态**：pending（等待中）→ streaming（逐字输出中）→ done（完成）

**状态管理（约 10h）**

```
用 React Context + useReducer 管理全局状态：

state = {
  agents: Agent[],              // 所有可用 Agent
  activeAgentId: string,        // 当前选中的 Agent
  conversations: Conversation[], // 当前 Agent 的对话列表
  activeConversationId: string,  // 当前对话
  messages: Message[],          // 当前对话的消息
  streaming: boolean,           // 是否正在流式输出
}
```

> **为什么不直接用 Redux / Zustand？**
> 先用 Context + useReducer 手写，理解状态管理的本质后再加库。面试时你能说清楚"为什么需要状态管理库"比"我会用 Redux"更重要。

#### 第 3-5 周里程碑
> ✅ 可切换 3 个 Agent 角色
> ✅ 流式输出（打字机效果）
> ✅ 对话历史持久化（刷新页面不丢失）
> ✅ Markdown + 代码高亮
> 🎥 **录制第一个 demo 视频，发到 B站/知乎 记录进度**

---

### 第 6-10 周（8/31 - 10/4）：工具使用 —— Agent 真正"能做事"

> ⚠️ 这是整个项目最核心、面试最加分的技术阶段。做好它，你的项目就比 90% 的课程作业强。

#### 本周目标
实现一套通用的 Tool Use 框架，让 Agent 能调用工具完成任务。**这是 Agent 区别于聊天机器人的本质。**

#### 核心设计：Tool 抽象层（约 15h）

```python
# 工具定义的 Pydantic 模型
from pydantic import BaseModel
from typing import Any, Callable

class ToolParameter(BaseModel):
    """单个参数定义"""
    name: str
    type: str          # "string" | "number" | "boolean"
    description: str
    required: bool = True

class Tool(BaseModel):
    """工具定义 —— 这是你设计的核心抽象"""
    name: str                    # "search_web", "execute_python"
    description: str             # LLM 看这个决定何时调用
    parameters: list[ToolParameter]

    # 实际执行逻辑（子类覆写）
    async def execute(self, **kwargs) -> str:
        raise NotImplementedError

    # 生成发给 LLM 的 function definition（OpenAI 格式）
    def to_openai_function(self) -> dict:
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": {
                    "type": "object",
                    "properties": {
                        p.name: {"type": p.type, "description": p.description}
                        for p in self.parameters
                    },
                    "required": [p.name for p in self.parameters if p.required],
                },
            },
        }
```

**设计原则**：
- 抽象层跟具体的 LLM API 格式解耦（即使 OpenAI 格式是事实标准）
- 每个 Tool 是独立类，方便添加新工具
- `execute` 方法返回纯文本（LLM 只读文本）

#### 实现的工具（约 30h）

**1. WebSearchTool —— 网络搜索（约 6h）**

```python
class WebSearchTool(Tool):
    """搜索网络信息，返回标题+摘要+URL"""
    name = "search_web"
    description = "搜索互联网获取最新信息。用于查找文档、新闻、技术资料。"
    parameters = [
        ToolParameter(name="query", type="string",
                      description="搜索关键词，用英文或中文"),
        ToolParameter(name="num_results", type="number",
                      description="返回结果数量，默认5"),
    ]

    async def execute(self, query: str, num_results: int = 5) -> str:
        # 选项1: 用 Bing Search API（有免费额度）
        # 选项2: 用 SerpAPI（稳定但收费）
        # 选项3: 用 DuckDuckGo（免费但结果少）
        # 返回格式化文本：标题\n摘要\nURL\n---\n...
        pass
```

**2. PythonExecutorTool —— 代码执行（约 10h）**

```python
class PythonExecutorTool(Tool):
    """在沙箱中执行 Python 代码，返回 stdout/stderr"""
    name = "execute_python"
    description = "执行 Python 代码。用于计算、数据分析、测试代码逻辑。"
    parameters = [
        ToolParameter(name="code", type="string",
                      description="要执行的 Python 代码"),
    ]

    async def execute(self, code: str) -> str:
        # 沙箱方案（按安全性递进）：
        # 1. subprocess.run + timeout（简单，不够安全）
        # 2. Docker 容器执行（推荐，学习 Docker 的好机会）
        #    每次创建新容器：docker run --rm --network=none
        #    --memory=256m python:3.11-slim python -c "code"
        # 3. RestrictedPython / PyPy 沙箱（复杂，不推荐）
        pass
```

**安全问题（面试必问，必须能说清楚）**：
- 网络隔离（`--network=none`）
- 内存限制（`--memory=256m`）
- 超时机制（`timeout=30s`）
- 禁止的文件系统操作
- **你知道这些风险 + 知道怎么控制 = 面试加分**

**3. FileReaderTool —— 文件读取（约 6h）**

```python
class FileReaderTool(Tool):
    """读取项目目录下的文件内容，用于代码审查"""
    name = "read_file"
    description = "读取指定文件的内容。用于查看代码、配置文件等。"
    parameters = [
        ToolParameter(name="filepath", type="string",
                      description="文件路径（相对于项目根目录）"),
    ]

    async def execute(self, filepath: str) -> str:
        # 安全检查：
        # 1. 路径不能包含 ..（防止目录穿越）
        # 2. 文件必须在允许的目录内
        # 3. 文件大小限制（如 1MB）
        # 4. 只允许特定扩展名（.py, .js, .json, .md 等）
        pass
```

**4. FileWriterTool —— 文件写入（约 6h）**

```python
class FileWriterTool(Tool):
    """将内容写入文件，用于代码生成场景"""
    name = "write_file"
    description = "将内容写入文件。用于保存生成的代码。"
    parameters = [
        ToolParameter(name="filepath", type="string",
                      description="输出文件路径"),
        ToolParameter(name="content", type="string",
                      description="要写入的内容"),
    ]

    async def execute(self, filepath: str, content: str) -> str:
        # 安全检查同 FileReaderTool
        # 额外：写入前先备份原文件
        pass
```

#### Agent Loop —— 核心推理循环（约 15h）

这是整个项目最重要的代码，值得反复打磨：

```python
from enum import Enum

class StopReason(Enum):
    FINISHED = "finished"         # LLM 认为任务完成，直接回复
    TOOL_CALL = "tool_call"       # LLM 需要调用工具
    MAX_ITERATIONS = "max_iter"   # 超过最大循环次数，强制终止
    ERROR = "error"               # 发生错误

class AgentLoopResult(BaseModel):
    content: str                   # 最终回复内容
    stop_reason: StopReason
    iterations: int                # 循环了多少次
    tool_calls: list[dict]         # 每次工具调用的记录（用于前端展示）

async def agent_loop(
    message: str,
    tools: list[Tool],
    history: list[dict],
    max_iterations: int = 10,      # 防止无限循环
) -> AgentLoopResult:
    """Agent 核心推理循环"""
    messages = history + [{"role": "user", "content": message}]
    tool_call_log = []             # 记录所有工具调用

    for i in range(max_iterations):
        # 1. 调用 LLM（带工具定义）
        response = await llm.chat(
            messages=messages,
            tools=[t.to_openai_function() for t in tools],
        )

        # 2. 判断 LLM 的意图
        if response.has_tool_calls():
            # LLM 想调工具
            for tc in response.tool_calls:
                tool = find_tool(tc.name, tools)
                error = None
                try:
                    result = await tool.execute(**tc.arguments)
                except Exception as e:
                    result = f"Error: {str(e)}"
                    error = str(e)

                # 记录（前端展示用）
                tool_call_log.append({
                    "tool": tc.name,
                    "arguments": tc.arguments,
                    "result": result[:2000],  # 截断过长结果
                    "error": error,
                })

                # 工具结果喂回 LLM
                messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": result,
                })
            # 继续循环，让 LLM 决定下一步
            continue

        else:
            # 3. LLM 直接回复 → 结束
            return AgentLoopResult(
                content=response.content,
                stop_reason=StopReason.FINISHED,
                iterations=i + 1,
                tool_calls=tool_call_log,
            )

    # 4. 超过最大循环 → 让 LLM 总结当前结果
    messages.append({
        "role": "user",
        "content": "已达到最大执行次数。请基于当前所有信息，给用户一个总结性的回复。"
    })
    last_response = await llm.chat(messages=messages)
    return AgentLoopResult(
        content=last_response.content,
        stop_reason=StopReason.MAX_ITERATIONS,
        iterations=max_iterations,
        tool_calls=tool_call_log,
    )
```

**这段代码需要深刻理解的几个点**：
1. **为什么要循环**：一次工具调用可能不够，LLM 可能连续调多个工具
2. **为什么限制循环次数**：防止无限循环（token 在烧钱）
3. **工具结果截断**：搜索结果可能很长，不截断会超出 context window
4. **错误处理**：工具执行失败不应该让整个 Agent 崩溃

#### 前端新增（约 20h）

**ToolCallCard 组件**（面试时 demo 效果最好的部分）：

```
聊天界面中每条 AI 消息的展开：
┌────────────────────────────────────┐
│ 🤖 代码助手                         │
│                                    │
│ 让我先搜索一下这个报错的原因...       │
│                                    │
│ 🔧 调用工具: search_web    ⏱ 1.2s  │ ← ToolCallCard
│   参数: "Python KeyError 解决方法"  │
│   ┌──────────────────────────────┐ │
│   │ [搜索结果摘要]                │ │  ← 可折叠
│   └──────────────────────────────┘ │
│                                    │
│ 🔧 调用工具: read_file       ⏱ 0.3s │
│   参数: "src/main.py"             │
│                                    │
│ 根据搜索结果和你的代码...            │ ← 最终回复
└────────────────────────────────────┘
```

组件需求：
- 每个工具调用一个可折叠的卡片
- 显示工具名、参数、耗时
- 如果出错了显示红色错误信息
- 流式过程中实时更新（边调工具边显示）

**Agent 思考过程动画**（加分项）：
- 调用工具时，聊天框里显示 "思考中..." 的跳动动画
- 如果有多个工具调用，显示步骤数 "第 2/3 步"

#### 第 6-10 周里程碑
> ✅ 4 个工具全部可用（搜索、执行代码、读文件、写文件）
> ✅ Agent Loop 稳定运行（不会无限循环）
> ✅ 前端展示工具调用过程
> 🎥 **录制第二个 demo：Agent 搜索 + 写代码解决一个实际问题**

---

### 第 11-14 周（10/5 - 11/1）：代码助手深度打磨

> 只做一个场景，但做到极致。面试官看到一个功能做深了，比十个浅尝辄止的功能强得多。

#### 本周目标
围绕"代码助手"这一个场景，实现完整的多步骤工作流，并打磨 UI 体验。

#### 代码审查工作流（约 30h）

```
用户提交代码/指定仓库
        ↓
  ┌─────────────────┐
  │ 1. 代码分析Agent  │  read_file 逐文件读取
  │   理解项目结构    │  生成项目结构图
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ 2. 代码审查Agent  │  检查：
  │   多维度审查      │  - 逻辑错误 / Bug
           │          │  - 安全问题
           │          │  - 性能问题
           │          │  - 代码风格
  └────────┬────────┘  - 最佳实践偏离
           ↓
  ┌─────────────────┐
  │ 3. 建议生成Agent  │  为每个问题生成：
  │   生成修复建议    │  - 问题描述
           │          │  - 严重程度（critical/warning/info）
           │          │  - 修复代码（diff 格式）
  └────────┬────────┘  - 解释为什么这样改
           ↓
  ┌─────────────────┐
  │ 4. 测试验证Agent  │  执行修复后的代码
  │   验证修复正确性  │  对比修复前后的输出
  └─────────────────┘
           ↓
       生成审查报告（Markdown 格式）
```

**关键技术点**：
- **代码 Diff 展示**：前端用 `diff` 库渲染 GitHub 风格的 diff 视图
- **审查报告**：带目录、分级、代码示例的 Markdown 文档，支持一键导出
- **上下文窗口管理**：大型项目不能全塞进去，需要设计文件选择策略

#### 多 Agent 协作抽象（约 20h）

从代码助手场景中提炼出通用的多 Agent 模式，但不追求抽象的完美——能跑通这个场景就行：

```python
# 最简单的多Agent模式：流水线
class Pipeline:
    """顺序执行多个 Agent，每个的输出是下一个的输入"""
    def __init__(self, agents: list[Agent]):
        self.agents = agents

    async def run(self, input: str, context: dict) -> str:
        result = input
        for agent in self.agents:
            result = await agent.run(result, context=context)
        return result

# 代码审查的实例化
code_review_pipeline = Pipeline([
    CodeAnalyzerAgent(),    # 分析项目结构
    CodeReviewerAgent(),    # 多维度审查
    SuggestionAgent(),      # 生成修复建议
    VerificationAgent(),    # 验证修复
])
```

#### 前端打磨（约 25h）

**代码对比视图**：
```
┌─────────────────────────────────────────┐
│ 📄 src/utils.py                  [+3 -2]│
├─────────────────────────────────────────┤
│  12  def process_data(items):           │
│  13      result = []                    │
│ -14      for item in items:             │
│ -15          result.append(item * 2)    │
│ +14      return [item * 2 for item ...] │
│  16      return result                  │
├─────────────────────────────────────────┤
│ 💡 建议：使用列表推导式更简洁、更快      │
│ ⚠️ 严重程度：info                       │
└─────────────────────────────────────────┘
```

**审查报告面板**：
- 左侧：文件树（哪些文件有问题，标记颜色）
- 中间：代码 diff 视图
- 右侧：问题列表（按严重程度排序）

**通用 UI 改进**：
- 暗色/亮色主题切换（代码审查暗色更舒服）
- 响应式布局（面试时可以手机打开展示）
- Loading 状态和空状态的设计

#### 第 11-14 周里程碑
> ✅ 代码审查工作流完整可用（分析 → 审查 → 建议 → 验证）
> ✅ 代码 Diff 视图
> ✅ 审查报告生成 + 导出
> 🎥 **录制第三个 demo：提交一个真实项目，Agent 自动审查并给出修复建议**

---

### 第 15-18 周（11/2 - 11/29）：部署上线 + 项目包装

#### 本周目标
把项目部署到线上，写好文档，让它成为一个"正经产品"。

#### Docker 容器化（约 15h）

```
项目结构（最终）：
agent-lab/
├── backend/
│   ├── app/
│   │   ├── main.py           # FastAPI 入口
│   │   ├── config.py         # 配置管理（环境变量）
│   │   ├── models/           # Pydantic + SQLAlchemy 模型
│   │   ├── routes/           # API 路由
│   │   ├── agents/           # Agent 核心逻辑
│   │   │   ├── base.py       # Agent 基类
│   │   │   ├── loop.py       # Agent Loop
│   │   │   └── tools/        # 所有工具实现
│   │   └── services/         # 业务逻辑层
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── components/       # React 组件
│   │   ├── contexts/         # React Context
│   │   ├── hooks/            # 自定义 Hooks
│   │   └── pages/            # 页面
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yml        # 一键启动
├── .env.example              # 环境变量模板
└── README.md                 # 项目文档
```

**docker-compose.yml**：
```yaml
services:
  backend:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      - DATABASE_URL=sqlite:///data/app.db   # 演示用 SQLite
      - LLM_API_KEY=${LLM_API_KEY}
    volumes:
      - ./data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock  # 代码执行需要

  frontend:
    build: ./frontend
    ports: ["3000:80"]
    depends_on: [backend]
```

#### 云部署（约 10h）

| 方案 | 适合场景 | 费用 |
|------|---------|------|
| **阿里云 ECS 学生优惠** | 正经项目，面试加分 | 约 10元/月（学生认证后） |
| **腾讯云轻量应用服务器** | 同上 | 有免费额度 |
| **Railway / Fly.io** | 国外平台，部署简单 | 有免费额度 |
| **Vercel(前端) + Railway(后端)** | 最小配置成本 | 免费 |

**推荐方案**：阿里云 ECS（学生认证）+ Docker Compose 部署
- 学生认证后 ECS 约 10 元/月
- 同时学习基础的 Linux 运维

#### 文档 + 演示（约 15h）

**README.md 结构**（面试官看这个的时间不超过 3 分钟）：

```markdown
# Agent Lab —— 多 Agent 实验平台

## 一句话介绍
一个可以创建、配置、使用多种 AI Agent 的 Web 平台，支持工具调用和多 Agent 协作。

## Demo
[GIF/视频链接：展示代码审查工作流]

## 核心特性
- 🎭 多角色 Agent（代码助手 / 旅游规划 / 苏格拉底导师）
- 🔧 工具使用（网络搜索 / 代码执行 / 文件读写）
- 🔄 多 Agent 协作（代码审查流水线）
- ⚡ 流式输出 + 工具调用可视化
- 🐳 Docker 一键部署

## 技术架构
[架构图：React → FastAPI → LLM API / Tools]

## 快速开始
```bash
git clone ...
cp .env.example .env    # 填 API Key
docker-compose up -d
# 打开 http://localhost:3000
```

## 项目结构
[目录树]

## 技术亮点
- 手写 Agent Loop（非框架），深入理解 Tool Use 机制
- Docker 沙箱执行代码，包含安全隔离措施
- 流式传输（SSE）+ 实时渲染
- 多 Agent 流水线编排

## 开发日志
[链接到 StudyRecord 或博客]
```

#### 第 15-18 周里程碑
> ✅ 线上可访问的完整应用
> ✅ 一份专业的 README
> ✅ Docker 一键部署
> 🎥 **录制最终 demo 视频（3-5 分钟，配解说）**

---

### 第 19-20 周（11/30 - 12/13）：面试冲刺

- 回顾项目中的技术决策（每个都能说清楚"为什么这样做"）
- 准备 3 分钟项目介绍（电梯演讲）
- 准备常见面试问题：
  - "你的项目中最大的技术挑战是什么？"
  - "Agent Loop 为什么要循环？怎么防止无限循环？"
  - "代码执行的沙箱是怎么做的？有哪些安全考虑？"
  - "为什么不用 LangChain？"
- 模拟面试
- 投递简历

---

## LeetCode 路线图

> 每日 2h，按分类刷，每道题先想 15min → 看题解 → 自己写 → 第二天复习。
> 推荐资源：**代码随想录**（中文，讲解好）、**labuladong 的算法小抄**

### 第一阶段：基础数据结构（第 1-4 周）

| 周 | 专题 | 题量 | 重点题 |
|----|------|------|--------|
| 1 | 数组 | 8 题 | 两数之和(1)、三数之和(15)、多数元素(169)、移动零(283) |
| 2 | 哈希表 | 7 题 | 字母异位词分组(49)、最长连续序列(128)、和为K的子数组(560) |
| 3 | 双指针 | 7 题 | 盛水最多的容器(11)、接雨水(42)、无重复字符的最长子串(3) |
| 4 | 滑动窗口 | 8 题 | 找到字符串中所有字母异位词(438)、最小覆盖子串(76) |

### 第二阶段：中级数据结构（第 5-9 周）

| 周 | 专题 | 题量 | 重点题 |
|----|------|------|--------|
| 5 | 链表 | 8 题 | 反转链表(206)、环形链表(141,142)、合并有序链表(21)、LRU缓存(146) |
| 6 | 栈与队列 | 7 题 | 有效的括号(20)、每日温度(739)、字符串解码(394) |
| 7-8 | 二叉树 | 12 题 | 前中后序遍历(144,94,145)、层序遍历(102)、最大深度(104)、对称二叉树(101)、最近公共祖先(236)、路径总和III(437) |
| 9 | DFS/BFS | 8 题 | 岛屿数量(200)、课程表(207)、单词搜索(79)、全排列(46) |

### 第三阶段：动态规划（第 10-14 周）

> DP 是面试最高频的 hard 考点，AI 岗要求比 SDE 低但也不能完全不会

| 周 | 专题 | 题量 | 重点题 |
|----|------|------|--------|
| 10 | DP 入门 | 8 题 | 爬楼梯(70)、打家劫舍(198)、最大子数组和(53)、不同路径(62) |
| 11 | 背包问题 | 7 题 | 分割等和子集(416)、零钱兑换(322)、组合总和IV(377) |
| 12 | 序列DP | 7 题 | 最长递增子序列(300)、最长公共子序列(1143)、编辑距离(72) |
| 13-14 | DP 复习 | — | 二刷以上所有 DP 题 + 做变体 |

### 第四阶段：冲刺复习（第 15-20 周）

| 周 | 内容 |
|----|------|
| 15 | 回溯算法（7 题）：子集(78)、组合总和(39)、N皇后(51) |
| 16 | 贪心 + 排序（7 题）：跳跃游戏(55)、合并区间(56) |
| 17-18 | Hot 100 二刷（重点题 + 之前做错的题） |
| 19-20 | 模拟面试（限时做题 + 边写边讲思路） |

### 每日刷题 SOP

```
15min   自己想思路，写下伪代码
  ↓
10min   看题解（代码随想录 / labuladong），对比自己的思路差在哪
  ↓
20min   合上书，自己完整写一遍代码（不要抄！）
  ↓
10min   写注释解释关键逻辑 + 分析复杂度
  ↓
5min    记录到错题本（不是抄题，是记"为什么错/卡在哪"）
  ↓
第二天  花 10min 默写前一天做过的题
```

---

## LLM 理论学习清单（夜间 1h）

### 必读（按顺序）

| 序号 | 内容 | 预计时间 | 完成标记 |
|------|------|----------|----------|
| 1 | [Karpathy: Let's build GPT from scratch](https://www.youtube.com/watch?v=kCc8FmEb1nY) | 2晚 | ☐ |
| 2 | [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | 1晚 | ☐ |
| 3 | OpenAI Prompt Engineering Guide | 1晚 | ☐ |
| 4 | [Lilian Weng: LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/) | 2晚 | ☐ |
| 5 | [Chip Huyen: Building A Generative AI Platform](https://huyenchip.com/2024/07/25/genai-platform.html) | 2晚 | ☐ |

### 选读（有时间就看）

| 内容 | 说明 |
|------|------|
| OpenAI Function Calling 官方文档 | 理解 Tool Use 的 API 层 |
| LangChain 源码（只看 Agent 部分） | 等你写完后对比，理解"原来框架就是封装了这些" |
| Anthropic Context Engineering 博客 | 进阶：如何管理 Agent 的上下文 |
| RAG 综述论文 | 了解 Embedding → 检索 → 生成的完整链路 |

---

## 每周检查清单模板

在 `StudyRecord/` 下每周建一个文件（如 `week1-checkin.md`）：

```markdown
# Week X 检查（日期 - 日期）

## 项目进度
- [ ] 任务1
- [ ] 任务2

## 算法进度
本周完成题数：___ / 计划___
卡住的题：___
新学到的技巧：___

## 理论/阅读
读了：___
一句话收获：___

## 遇到的问题
（技术问题、思路卡点、时间管理...）

## 下周调整
（哪些地方需要改进）
```

---

## 最后的提醒

### 做减法的勇气

这个计划已经是最小可行版本了。过程中你会不断想加新功能——忍住。完成比完美重要，一个做完的简单项目比一个放弃的"完美构想"强一百倍。

### "不会"是常态

你会在每个阶段遇到完全不会的东西。React 报错看不懂、FastAPI 的 async 让人困惑、LeetCode 的 DP 题想不出来——**这都是正常的**。区别只在于你是停下来怀疑自己，还是去搜、去问、继续试。

### 用好 Claude Code

你现在用的 Claude Code 就是最好的结对编程伙伴。遇到不会的，直接把报错贴给它，把代码扔给它分析。但记住：**先自己想了再问，看完解释自己再写一遍**。

### 记录在公开的地方

知乎/掘金/博客园/个人博客——每周至少发一篇记录。12 月面试时，你把「持续记录了 20 周的开发日志」链接放进简历，面试官会认真看的。这比你简历上写"熟练掌握 React、FastAPI"有说服力得多。
