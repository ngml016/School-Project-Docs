# 🎒 学生端对话 Agent 实现文档

> **核心思路：最小 Agent，前端驱动，单一循环。**
>
> 学生端不搞校园端的多 Agent 分层（Orchestrator + 7 个子 Agent），而是用**单 Agent + 前端 Function Calling** 实现对话。学生端只做一件事：**用户说话 → LLM 理解意图 → 调 MCP 工具 → 返回结果**。

---

## 🏗️ 整体架构

```
┌─────────────────────────────────────────┐
│             学生端 H5 前端                │
│                                         │
│  ┌───────────┐    ┌──────────────────┐  │
│  │  Chat.tsx  │    │  Agent Loop      │  │  ← 主页即聊天页
│  │  聊天界面   │◄──▶│  (前端 Orchestrator)│  │
│  └───────────┘    └────────┬─────────┘  │
│                            │             │
│  ┌─────────────────────────▼──────────┐ │
│  │  api/llm.ts                        │ │  ← 调 DeepSeek V4 Flash（Function Call）
│  │  POST /api/chat (SSE stream)       │ │
│  └───────────────────────┬────────────┘ │
└──────────────────────────┼──────────────┘
                           │
              ┌────────────▼──────────────┐
              │     学生端后端 (FastAPI)    │
              │                            │
              │  ┌──────────────────────┐  │
              │  │  router/chat.py      │  │  ← SSE 聊天接口
              │  │  LLM + MCP 编排      │  │
              │  └──────────┬───────────┘  │
              │             │               │
              │  ┌──────────▼───────────┐  │
              │  │  mcp_client.py       │  │  ← 调校园端 MCP 工具
              │  └──────────┬───────────┘  │
              └─────────────┼──────────────┘
                            │ MCP (SSE)
              ┌─────────────▼──────────────┐
              │ 校园端 MCP Server (:8001)   │ ← 17 个工具
              └────────────────────────────┘
```

### 与校园端的核心区别

| 对比维度 | 🏫 校园端（教师） | 🎒 学生端 |
|:---------|:-----------------|:---------|
| Agent 数量 | Orchestrator + 7 个子 Agent | **1 个 Agent（纯前端编排）** |
| LLM 调用方 | Python 后端（OpenAI SDK） | **前端直接调 / 后端转发** |
| 工具来源 | 直接操作本地 SQLite 数据库 | **走 MCP 协议调校园端工具** |
| 多轮上下文 | 后端维护 session 级历史 | **前端维护消息列表，每次全量发送** |
| 复杂度 | 高（路由、子 Agent 调度） | **低（单一循环，无路由）** |
| 部署依赖 | 必须同后端运行 | **只需 DeepSeek API Key + MCP Server** |

---

## 🔄 Agent 工作流程

### 一次对话的完整生命线

```
学生: "帮我签到"
  │
  ├──► 1. 前端 Chat.tsx 捕捉输入
  │       将消息加入历史 → 调 POST /api/chat (SSE)
  │
  ├──► 2. 后端 /api/chat 收到请求
  │       拼接 System Prompt + 消息历史
  │       用 DeepSeek V4 Flash 的 function calling
  │
  ├──► 3. LLM 识别意图 → 返回 tool_call
  │       tool_call: do_checkin(student_id="20260001", code="582641")
  │
  ├──► 4. 后端调用 MCP 工具
  │       mcp_client.call_tool("do_checkin", {...})
  │       → 校园端返回 "✅ 签到成功！高等数学"
  │
  ├──► 5. 后端将 tool_result 喂回 LLM
  │       LLM 生成自然语言回复
  │
  └──► 6. SSE 流式返回给前端
        前端渲染气泡
```

### 最小 Agent 循环（伪代码）

```python
# 后端 router/chat.py — 核心 Agent 循环
async def chat_loop(messages, student_id):
    # 注入系统提示词
    system_prompt = build_system_prompt(student_id)

    # Step 1: 调 LLM（带 function calling）
    response = await llm.chat(
        model="deepseek-v4-flash",
        messages=[{"role": "system", "content": system_prompt}, *messages],
        tools=MCP_TOOL_DEFINITIONS,  # 17 个 MCP 工具转为 function schema
    )

    # Step 2: 如果有 tool_call，执行 MCP 工具
    while response.choices[0].finish_reason == "tool_calls":
        tool_call = response.choices[0].message.tool_calls[0]

        # 调校园端 MCP
        tool_result = await mcp_client.call_tool(
            tool_call.function.name,
            json.loads(tool_call.function.arguments),
        )

        # 把结果喂回 LLM
        messages.append(response.choices[0].message)
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": tool_result,
        })
        response = await llm.chat(messages=messages, tools=MCP_TOOL_DEFINITIONS)

    # Step 3: 返回最终回复（SSE 流式输出）
    return response.choices[0].message.content
```

---

## 📦 最小实现方案

### 方案 A：后端集中式（推荐最小改动）

**适用场景：** 已有 FastAPI 后端，不想动前端架构。

后端新增一个接口 `POST /api/chat`（SSE 流式），前端只需一个 Chat.tsx 页面。

```
新增文件（最小集）：
  backend/router/chat.py     # SSE 聊天接口 + Agent 循环
  backend/llm_client.py      # DeepSeek V4 Flash 客户端封装
  frontend/src/pages/Chat.tsx # 聊天主页
  frontend/src/agent/orchestrator.ts  # （可选）前端直调 LLM 的编排
```

#### 后端 chat.py 结构

```python
# backend/router/chat.py
from fastapi import APIRouter
from sse_starlette.sse import EventSourceResponse
from llm_client import DeepSeekClient
from mcp_client import mcp_client

router = APIRouter()
llm = DeepSeekClient()

# 17 个 MCP 工具 → OpenAI function calling schema
MCP_TOOLS = [  # ← 固定写死，不从 MCP 动态获取
    {
        "type": "function",
        "function": {
            "name": "get_student_info",
            "description": "获取学生个人信息（姓名、班级、手机号）",
            "parameters": {
                "type": "object",
                "properties": {
                    "student_id": {"type": "string", "description": "学号"},
                },
                "required": ["student_id"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "do_checkin",
            "description": "学生签到，输入6位签到码",
            "parameters": {
                "type": "object",
                "properties": {
                    "student_id": {"type": "string"},
                    "code": {"type": "string", "description": "6位签到码"},
                },
                "required": ["student_id", "code"],
            },
        },
    },
    # ... 其余 15 个 MCP 工具同理
]

SYSTEM_PROMPT = """你是校园AI学生助手，运行在学生端。

## 可用功能
- 📍 签到 — 输入签到码即可
- 📅 课表 — 查看今日或某天的课程
- 📝 作业 — 查作业、交作业（支持文件附件）
- 📊 成绩 — 查看作业批改分数
- 📋 请假 — 提交请假申请
- 🔔 通知 — 查看最新通知和公告
- 👤 个人信息 — 查看我的信息

## 工作方式
1. 学生说一句话，你判断需要做什么
2. 需要调用工具时，使用 function calling
3. 需要参数（如签到码、课程ID）时，先问学生
4. 用 emoji 让回复更友好
5. 每次只调用一个工具，等结果出来再决定下一步

当前学生学号：{student_id}
当前时间：{current_time}"""

@router.post("/api/chat")
async def chat(body: dict):
    """SSE 流式聊天接口"""
    messages = body["messages"]       # 前端传来的消息历史
    student_id = body.get("student_id", "")

    system_prompt = SYSTEM_PROMPT.format(
        student_id=student_id,
        current_time=datetime.now().strftime("%Y-%m-%d %H:%M"),
    )

    async def generate():
        # 第一步：调 LLM
        full_messages = [{"role": "system", "content": system_prompt}, *messages]
        response = await llm.chat_stream(full_messages, tools=MCP_TOOLS)

        # tool_call 循环
        while True:
            choice = response.choices[0]
            if choice.finish_reason == "tool_calls":
                # 3. 执行 MCP 工具
                tool_call = choice.message.tool_calls[0]
                yield {"event": "tool_call", "data": json.dumps({
                    "name": tool_call.function.name,
                    "args": json.loads(tool_call.function.arguments),
                })}

                result = await mcp_client.call_tool(
                    tool_call.function.name,
                    json.loads(tool_call.function.arguments),
                )

                # 4. 喂回 LLM
                full_messages.append(choice.message)
                full_messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": str(result),
                })
                response = await llm.chat_stream(full_messages, tools=MCP_TOOLS)
            else:
                # 5. 流式输出最终回复
                async for chunk in response:
                    if chunk.choices[0].delta.content:
                        yield {"event": "message", "data": chunk.choices[0].delta.content}
                break

    return EventSourceResponse(generate())
```

#### 前端 Chat.tsx 最小骨架

```tsx
// frontend/src/pages/Chat.tsx
function ChatPage() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState("");

  async function sendMessage() {
    const userMsg = { role: "user", content: input };
    const newMessages = [...messages, userMsg];
    setMessages(newMessages);
    setInput("");

    // SSE 流式请求
    const resp = await fetch("/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        messages: newMessages,
        student_id: localStorage.getItem("student_id"),
      }),
    });

    const reader = resp.body!.getReader();
    const decoder = new TextDecoder();
    let agentReply = "";

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const text = decoder.decode(value);
      // 解析 SSE 事件，逐字追加 agentReply
      // 每收到一段 content 就 setMessages 触发渲染
    }

    setMessages([...newMessages, { role: "assistant", content: agentReply }]);
  }

  return (
    <div className="chat-page">
      <div className="chat-header">🏫 校园AI助手</div>
      <div className="chat-messages">
        {messages.map((m, i) => (
          <div key={i} className={`bubble ${m.role}`}>
            {m.content}
          </div>
        ))}
      </div>
      <div className="chat-input-bar">
        <input value={input} onChange={e => setInput(e.target.value)} />
        <button onClick={sendMessage}>发送</button>
      </div>
    </div>
  );
}
```

---

### 方案 B：前端纯 Agent（更轻量，无需后端改动）

**适用场景：** 学生端已有后端接口（`/api/classes`、`/api/checkin` 等），不想新增 `/api/chat`。

前端直接调 DeepSeek API，工具调用走已有的后端 REST 接口。

```
无需新增后端文件，只需：
  frontend/src/api/llm.ts        # DeepSeek 客户端
  frontend/src/agent/index.ts    # Agent 循环
  frontend/src/pages/Chat.tsx    # 聊天主页
```

#### Agent 循环（前端）

```typescript
// frontend/src/agent/index.ts
const DEEPSEEK_API = "https://api.deepseek.com/v1/chat/completions";

const TOOLS = [
  {
    type: "function",
    function: {
      name: "do_checkin",
      description: "学生签到",
      parameters: {
        type: "object",
        properties: {
          code: { type: "string", description: "6位签到码" },
        },
        required: ["code"],
      },
    },
  },
  // ... 其他工具
];

async function agentLoop(messages: Message[]): Promise<string> {
  // 1. 调 LLM
  const resp = await fetch(DEEPSEEK_API, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${DEEPSEEK_KEY}`,
    },
    body: JSON.stringify({
      model: "deepseek-v4-flash",
      messages: [{ role: "system", content: SYSTEM_PROMPT }, ...messages],
      tools: TOOLS,
    }),
  });
  const data = await resp.json();
  const choice = data.choices[0];

  // 2. 如果有 tool_call
  if (choice.finish_reason === "tool_calls") {
    const tool = choice.message.tool_calls[0];

    // 3. 调已有后端 REST 接口
    const backendResp = await fetch(`/api/${tool.function.name}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: tool.function.arguments,
    }).then(r => r.json());

    // 4. 把结果喂回 LLM
    messages.push(choice.message);
    messages.push({
      role: "tool",
      tool_call_id: tool.id,
      content: JSON.stringify(backendResp),
    });

    // 5. 递归直到 LLM 直接回复
    return agentLoop(messages);
  }

  // 6. LLM 直接回复
  return choice.message.content;
}
```

---

## 🧩 MCP 工具 → Function Calling 映射表

学生端 17 个 MCP 工具中，**只有需要对话交互的部分**才暴露给 LLM：

| MCP 工具 | 暴露给 LLM？ | LLM 函数名 | 说明 |
|:---------|:------------|:----------|:-----|
| `get_student_info` | ✅ | `get_student_info` | "我的信息" |
| `get_schedule` | ✅ | `get_schedule` | "今天有什么课" |
| `get_courses` | ✅ | `get_courses` | "我的课程" |
| `do_checkin` | ✅ | `do_checkin` | "签到"+等待码 |
| `get_checkin_records` | ✅ | `get_checkin_records` | "签到记录" |
| `get_checkin_stats` | ✅ | `get_checkin_stats` | "出勤率" |
| `get_pending_homeworks` | ✅ | `get_pending_homeworks` | "有作业吗" |
| `get_all_homeworks` | ✅ | `get_all_homeworks` | "所有作业" |
| `get_homework_detail` | ✅ | `get_homework_detail` | 看某个作业详情 |
| `submit_homework` | ✅ | `submit_homework` | "交作业" |
| `upload_file` | ❌ | — | 前端文件选择器处理 |
| `get_homework_grades` | ✅ | `get_homework_grades` | "成绩" |
| `apply_leave` | ✅ | `apply_leave` | "请假" |
| `get_notifications` | ✅ | `get_notifications` | "通知" |
| `get_announcements` | ✅ | `get_announcements` | "公告" |
| `student_login` | ❌ | — | 登录页处理 |
| `student_register` | ❌ | — | 注册页处理 |
| `get_class_list` | ❌ | — | 注册页处理 |

**关键规则：**
- `upload_file` 不给 LLM —— 文件选择用前端 UI（📎按钮），上传完成后把 URL 传给 `submit_homework`
- 登录/注册/班级列表归传统表单页，不走 Agent
- LLM 不存 `student_id` —— 由前端从 localStorage 读取，每次调用自动注入

---

## 📋 系统提示词（最小版）

```typescript
const SYSTEM_PROMPT = `你是校园AI学生助手，运行在学生端 H5。

## 当前学生
学号：{student_id}

## 你可以做这些事
- 📍 签到 — 输入签到码即可
- 📅 课表 — 查看今日或某天的课程
- 📝 作业 — 查作业、交作业
- 📊 成绩 — 查看作业批改分数
- 📋 请假 — 提交请假申请
- 🔔 通知 — 查看最新通知和公告
- 👤 我的信息 — 查看个人信息和班级

## 你的工作方式
1. 学生说一句话，你判断需要做什么
2. 需要参数时先问学生（比如签到码、课程ID、请假原因）
3. 每次只调用一个工具，等结果出来再决定下一步
4. 用 emoji 让回复更友好
5. 如果学生说"不知道做什么"，主动提示可用功能

## 重要规则
- 不要替学生做决定，重要操作（交作业/请假）要让用户确认
- 签到码必须学生提供，你不能编造
- 文件上传请提示学生点击输入框左侧的📎按钮

当前时间：{current_time}`;
```

---

## 🚀 最小部署依赖

| 依赖 | 用途 | 必选？ |
|:-----|:-----|:------|
| DeepSeek V4 Flash API Key | LLM 推理 | ✅ |
| 校园端 MCP Server (:8001) | 调用 17 个工具 | ✅ |
| 学生端 FastAPI (:8002) | 方案A 需要，方案B 可选 | ⚠️ 方案A |
| `openai` Python 包 | 后端调 DeepSeek | ⚠️ 方案A |
| `sse-starlette` | SSE 流式输出 | ⚠️ 方案A |

### 环境变量

```bash
# 后端（方案A）
DEEPSEEK_API_KEY=sk-xxx
DEEPSEEK_BASE_URL=https://api.deepseek.com/v1
MCP_SERVER_URL=http://localhost:8001

# 前端（方案B）
VITE_DEEPSEEK_KEY=sk-xxx
VITE_DEEPSEEK_BASE_URL=https://api.deepseek.com/v1
```

---

## 📐 对话场景示例

### 签到（需要参数——多轮）

```
学生：签到
Agent：📍 请输入你的签到码～
学生：582641
Agent：(调 do_checkin)
       ✅ 签到成功！高等数学 9:00 @ 教学楼301
```

### 查课表（无需参数——直接返回）

```
学生：今天有什么课
Agent：(调 get_schedule)
       📅 周三 6月26日
       08:30-10:00 高等数学 @ 教学楼301
       10:20-11:50 大学英语 @ 教学楼205
```

### 交作业（文件走前端 UI——分步）

```
学生：我要交高数作业
Agent：📝 《高数第四章习题》 截止06-28 23:59
       请输入文字内容，或点击📎按钮上传文件～
学生：（打字 + 点📎选文件）
Agent：(调 upload_file → 调 submit_homework)
       ✅ 作业已提交！
```

---

## 🧪 验证清单

| 验证项 | 预期结果 |
|:-------|:---------|
| 打开应用 → 聊天页 | 显示快捷卡片 + 欢迎消息 |
| 说"签到" | Agent 回复"请输入签到码" |
| 输入签到码 | 签到成功，显示课程信息 |
| 说"今天有什么课" | 显示今日课表 |
| 说"作业" | 显示待交作业列表 |
| 说"帮我请假" | Agent 问哪节课、什么原因 |
| 点📎上传文件 | 文件显示在输入框中 |
| 说"我的信息" | 显示姓名、学号、班级 |

---

> 文档版本：v1.0 | 2026-06-29
> 设计理念：**最小 Agent，前端驱动，单循环无路由，MCP 工具即 Function**
