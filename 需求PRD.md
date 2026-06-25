# campus_assistant — 校园AI助手 需求PRD

> 版本：v1.0
> 日期：2026-06-25
> 状态：已定稿

---

## 1. 产品定位

### 1.1 核心理念

**双端分离，Agent 驱动。**

- **校园端**：教师通过 Chat 与 AI Agent 对话，完成课程、班级、签到、作业等管理。
- **学生端**：学生通过 MCP 协议调用校园端工具，完成签到、提交作业、请假等事务。

两套独立的项目，通过 MCP 协议互联。

### 1.2 双端角色

| 端 | 使用者 | 交互方式 | 项目 |
|:--|:-------|:---------|:-----|
| 🏫 校园端 | 教师 / 管理员 | Chat + Agent 对话 | `school-admin` |
| 🎒 学生端 | 学生 | 独立 App / 小程序，走 MCP | `school-student` |

### 1.3 用户角色

| 角色 | 归属 | 能做什么 |
|:-----|:-----|:---------|
| 👨‍🏫 教师 | 校园端 | 管理班级/课程，发起签到，布置/批改作业，发公告，查数据 |
| 👩‍🎓 学生 | 学生端 | 签到，交作业，请假，查课表/成绩/通知（通过 MCP） |

---

## 2. 系统架构

### 2.1 总体架构

```
┌──────────────────────────────┐    ┌──────────────────────────────┐
│        🏫 校园端              │    │        🎒 学生端              │
│                              │    │                              │
│  ┌──────────────────┐       │    │  ┌──────────────────┐        │
│  │  Chat 页面（教师用）│       │    │  │  React 前端页面   │        │
│  └────────┬─────────┘       │    │  └────────┬─────────┘        │
│           │ 对话             │    │           │ HTTP              │
│  ┌────────▼─────────┐       │    │  ┌────────▼─────────┐        │
│  │  🧠 主Agent       │       │    │  │  学生端后端        │        │
│  │  (Orchestrator)   │       │    │  │  (FastAPI)        │        │
│  └────────┬─────────┘       │    │  └────────┬─────────┘        │
│           │ 路由             │    │           │ MCP 协议          │
│  ┌────────▼─────────┐       │    │           │ (SSE)             │
│  │  7 个子 Agent     │       │    │  ┌────────▼─────────┐        │
│  │  (23 个工具)      │       │    │  │  MCP 客户端       │        │
│  └────────┬─────────┘       │    │  └──────────────────┘        │
│           │                  │    │                              │
│  ┌────────▼─────────┐       │    │  ┌──────────────────┐        │
│  │  SQLite 数据库     │       │    │  │  本地 SQLite      │        │
│  │  (12 张表)        │       │    │  │  (操作记录/备忘等)  │        │
│  └──────────────────┘       │    │  └──────────────────┘        │
│                              │    │                              │
│  ┌──────────────────┐       │    │                              │
│  │  MCP Server       │◄──────┼────┤                              │
│  │  (17 个工具)      │       │    │                              │
│  └──────────────────┘       │    │                              │
└──────────────────────────────┘    └──────────────────────────────┘
```

### 2.2 校园端技术栈

| 层次 | 技术选型 | 说明 |
|:-----|:---------|:------|
| 前端 | **纯静态 HTML + CSS + JS** | 由 FastAPI 直接托管，零构建 |
| 后端 | **FastAPI (Python 3.11+)** | 异步框架，自动 API 文档 |
| 数据库 | **SQLite (WAL 模式)** | 12 张表，零配置 |
| ORM | **SQLAlchemy 2.0** | 类型安全 |
| LLM | **DeepSeek V4 Flash** | Agent 大脑，支持 function calling |
| 鉴权 | **JWT (python-jose)** | HS256 签名 |
| MCP 协议 | **MCP Python SDK** | SSE 模式，端口 8001 |

### 2.3 学生端技术栈

| 层次 | 技术选型 | 说明 |
|:-----|:---------|:------|
| 前端 | **React 18 + Vite + TypeScript** | 独立项目 |
| 后端 | **FastAPI (Python 3.11+)** | 学生端自建后端，端口 8002 |
| 本地数据库 | **SQLite** | 操作记录、通知缓存、备忘 |
| MCP 对接 | **MCP Python SDK** | 连接校园端 MCP Server（8001） |
| LLM | **DeepSeek V4 Flash** | 学生端 Agent（可选） |

---

## 3. 校园端功能

### 3.1 Agent 体系

```
🧠 主Agent (Orchestrator)
├─ 意图识别 + 任务拆解
├─ 多轮对话上下文管理
└─ 路由到子Agent

📦 7 个子 Agent · 23 个工具

├─ 📋 班级管理 Agent  (3工具)
│   ├─ 创建班级
│   ├─ 查询班级列表
│   └─ 删除班级
│
├─ 📚 课程管理 Agent  (4工具)
│   ├─ 创建课程
│   ├─ 查询课程列表
│   ├─ 课程详情
│   └─ 删除课程
│
├─ 👨‍🏫 教师管理 Agent  (2工具)
│   ├─ 添加教师
│   └─ 查询教师列表
│
├─ 📍 签到管理 Agent  (3工具)
│   ├─ 发起签到（生成签到码）
│   ├─ 查询签到记录
│   └─ 出勤统计
│
├─ 📝 作业管理 Agent  (4工具)
│   ├─ 布置作业
│   ├─ 查看作业列表
│   ├─ 查看学生提交
│   └─ 批改评分
│
├─ 📢 公告管理 Agent  (2工具)
│   ├─ 发布公告（班级/全校）
│   └─ 查看公告
│
└─ 📊 数据查询 Agent  (5工具)
    ├─ 学生列表
    ├─ 学生详情（基本信息+出勤+作业）
    ├─ 出勤报表
    ├─ 作业提交统计
    └─ 学生综合报告
```

### 3.2 校园端前端页面

| 页面 | 功能 | 说明 |
|:-----|:-----|:------|
| **登录页** | 教师登录 | 账号（手机号/工号）+ 密码 |
| **注册页** | 教师注册 | 分步注册：手机号→验证码→姓名+工号 |
| **Chat 页** | Agent 对话 | 唯一管理界面，跟 Agent 说话完成所有操作 |

### 3.3 实时活动推送

当学生通过 MCP 执行操作时，校园端 Chat 页面右上角实时弹出 Toast：

| 学生操作 | 校园端显示 |
|:---------|:-----------|
| 签到 | 📍 测试学生1 签到《高等数学》 |
| 提交作业 | 📝 测试学生1 提交作业《课后习题》 |
| 请假 | 📋 测试学生1 请假《大学英语》：发烧 |

- Toast 从右侧滑入，5秒后自动消失
- 队列依次显示，前一个消失后下一个再弹出

### 3.4 教师操作自动通知学生

| 教师操作 | 学生收到通知 |
|:---------|:------------|
| 批改作业 | 📝 你的《高等数学》作业已批改：85分 |
| 审批请假 | 📋 你的请假申请已批准 |
| 发布公告 | 📢 班级新公告 |

---

## 4. 学生端功能

### 4.1 MCP 工具（17 个）

通过 MCP 协议调用校园端功能，学生端不需要开发后端业务逻辑。

#### 注册 & 登录（3 工具）

| 工具 | 说明 |
|:-----|:------|
| `get_class_list()` | 获取班级列表（注册时选班） |
| `student_register(phone, pwd, name, class_id)` | 注册，**学号自动生成** |
| `student_login(account, pwd)` | 登录，返回 token |

#### 信息 & 课表（3 工具）

| 工具 | 说明 |
|:-----|:------|
| `get_student_info(student_id)` | 个人信息 |
| `get_schedule(student_id, day)` | 课表（0=今天，1-7=周一到周日） |
| `get_courses(student_id)` | 已选课程 |

#### 签到（3 工具）

| 工具 | 说明 |
|:-----|:------|
| `do_checkin(student_id, code)` | 输入6位签到码签到 |
| `get_checkin_records(student_id, days)` | 签到记录 |
| `get_checkin_stats(student_id, days)` | 出勤统计 |

#### 作业（5 工具）

| 工具 | 说明 |
|:-----|:------|
| `get_pending_homeworks(student_id)` | 待交作业 |
| `get_all_homeworks(student_id)` | 全部作业（分类） |
| `get_homework_detail(homework_id, student_id)` | 作业详情 |
| `submit_homework(student_id, homework_id, content)` | 提交作业（文字） |
| `get_homework_grades(student_id)` | 作业成绩 |

#### 请假（1 工具）

| 工具 | 说明 |
|:-----|:------|
| `apply_leave(student_id, course_id, reason)` | 提交请假 |

#### 通知（2 工具）

| 工具 | 说明 |
|:-----|:------|
| `get_notifications(student_id)` | 通知列表 |
| `get_announcements(class_id)` | 班级公告 |

### 4.2 学生端前端页面（8 页）

| 页面 | 功能 |
|:-----|:------|
| **登录页** | 学号/手机号 + 密码登录 |
| **注册页** | 三步：手机号密码 → 选班级 → 显示学号 |
| **首页** | 个人信息 + 今日课表 + 待办作业 + 快捷操作 |
| **签到页** | 输入签到码 + 签到记录 + 出勤统计 |
| **作业页** | Tab 切换：待交/全部 + 详情 + 提交 + 成绩 |
| **课表页** | 星期选择器 + 每日课程 |
| **请假页** | 选择课程 + 填写原因 |
| **通知页** | 通知历史 + 已读/未读 |

### 4.3 学生端本地功能

| 功能 | 说明 |
|:-----|:------|
| **操作记录** | 自己在学生端的所有操作历史（本地 SQLite） |
| **通知缓存** | 校园端推送的通知自动存本地可查 |
| **备忘录** | 学习备忘/待办（本地） |

### 4.4 学生端通知轮询

- 每 10 秒调 `get_notifications` 检查新通知
- 有新通知时右上角弹出 Toast（样式同校园端）
- 通知自动缓存到本地 `notification_history` 表

---

## 5. 数据库设计

### 5.1 校园端（12 张表）

| 表 | 说明 | 核心字段 |
|:---|:-----|:---------|
| `users` | 用户（教师/学生） | id, name, role, phone, password_hash, student_id, teacher_id, class_id |
| `classes` | 班级 | id, name, grade, is_active |
| `courses` | 课程 | id, name, teacher_id, location, day_of_week, start_time, end_time |
| `course_classes` | 课程-班级关联 | course_id, class_id |
| `checkins` | 签到记录 | id, user_id, course_id, status, checkin_time |
| `checkin_codes` | 签到码 | id, course_id, code, expires_at, is_active |
| `leaves` | 请假 | id, user_id, course_id, reason, status |
| `homeworks` | 作业 | id, course_id, title, description, due_at |
| `submissions` | 作业提交 | id, homework_id, user_id, content, score, comment |
| `announcements` | 公告 | id, title, content, scope, class_id |
| `notifications` | 通知 | id, user_id, type, title, content, is_read |
| `student_activities` | 学生活动记录 | id, user_id, action_type, content, created_at |

### 5.2 学生端本地（3 张表）

| 表 | 说明 | 核心字段 |
|:---|:-----|:---------|
| `operation_logs` | 操作记录 | id, action_type, content, result, created_at |
| `notification_history` | 通知缓存 | id, campus_notif_id, title, content, notif_type, is_read |
| `memos` | 备忘录 | id, title, content, is_important, created_at |

---

## 6. 接口设计

### 6.1 API 返回格式

所有接口统一返回格式：
```json
{
  "status": "success | error | fail",
  "data": {},
  "msg": "操作成功"
}
```

### 6.2 MCP 对接方式

学生端通过 MCP 协议（SSE 模式）连接校园端 MCP Server：

```
MCP Server URL: http://<校园服务器IP>:8001/sse
```

MCP 调用流程：
1. SSE 连接获取 session_id
2. JSON-RPC 格式调 tools/call
3. 通过 SSE 接收返回结果

---

## 7. 部署说明

### 7.1 校园端启动

```bash
# FastAPI（管理后台）
cd school-admin/service
source .venv/bin/activate
uvicorn main:app --host 0.0.0.0 --port 8000 --reload

# MCP Server（学生端对接）
python mcp_server.py --transport sse --port 8001
```

### 7.2 学生端启动

```bash
# 学生端后端
cd school-student/backend
source .venv/bin/activate
uvicorn main:app --host 0.0.0.0 --port 8002 --reload

# 学生端前端
cd school-student/frontend
npm install
npm run dev
```

---

## 8. 优先级规划

### Phase 1 ✅ 已完成

| 端 | 功能 |
|:---|:-----|
| 校园端 | 教师注册/登录、Chat 页面、主Agent + 7个子Agent（23工具） |
| 校园端 | MCP Server（17工具）、活动实时推送（Toast） |
| 校园端 | 教师操作自动通知学生（批改/审批/公告） |
| 学生端 | 文档已出，待开发 |

### Phase 2 📝 待开发

| 端 | 功能 |
|:---|:-----|
| 学生端 | 前端页面（React）开发 |
| 学生端 | 后端 MCP 对接 |
| 学生端 | 本地数据库、操作记录、通知轮询 |
| 校园端 | 请假审批 Agent 工具 |
| 校园端 | 学生批量导入 |

### Phase 3 🔮 规划中

| 端 | 功能 |
|:---|:-----|
| 校园端 | 作业查重（LanceDB） |
| 校园端 | WebSocket 流式对话 |
| 学生端 | LLM Agent 集成（DeepSeek V4 Flash） |
| 校园端 | 自动签到规则 |
