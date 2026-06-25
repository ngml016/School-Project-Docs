# 🎒 学生端 AI 开发提示词（超详细版）

> 纯语言提示词，**不包含任何代码**。
> 
> **执行前：先把以下两个文档发给 AI 作为上下文：**
> 1. `学生端对接文档.md` — 包含 17 个 MCP 工具的完整参数和返回值
> 2. `学生端项目开发文档.md` — 包含技术栈、页面清单、本地数据库设计、MCP 对接关系表
>
> AI 需要阅读这些文档了解 MCP 工具和数据表结构，然后自己生成全部代码。
> 按顺序执行，每个提示词完成后验证成功再继续。

---

## 🔧 提示词 1：创建项目骨架

> 📖 **参考文档：** `学生端项目开发文档.md` → 技术栈表格、目录结构、本地数据库表设计
> 
> **角色：** 你是一个全栈工程师。

**任务：** 在指定目录下创建 school-student 项目，包含 frontend 和 backend 两个子目录。

### 前端（frontend）

用 Vite 创建 React + TypeScript 项目：

1. 在 frontend/ 目录下执行 vite 创建命令，选择 react-ts 模板
2. 进入 frontend/，安装所有依赖
3. 额外安装 react-router-dom
4. 删除 src/ 下的所有默认文件（App.css、index.css、assets 等）
5. 创建 vite.config.ts，配置 proxy，将 /api 开头的请求转发到 http://localhost:8002
6. 创建 src/main.tsx，这是应用的入口文件，挂载 App 组件到 root 元素
7. 创建 src/App.tsx，这是一个空的根组件，先返回一个简单的 hello 文本
8. 创建 src/vite-env.d.ts，添加 Vite 类型声明

### 后端（backend）

用 Python FastAPI 创建后端：

1. 创建 backend/ 目录
2. 创建虚拟环境 .venv（python3 -m venv）
3. 安装依赖包：fastapi、uvicorn[standard]、sqlalchemy、httpx、python-jose[cryptography]、pydantic
4. 创建 backend/data/ 目录（SQLite 数据库文件存放位置）
5. 创建 backend/database/__init__.py（空文件）
6. 创建 backend/database/database.py 文件：
   - 用 SQLAlchemy 连接 SQLite（数据库文件在 data/student.db）
   - 创建 SessionLocal 会话工厂
   - 创建 Base 模型基类
   - 创建 get_db 依赖函数
   - 创建 init_db 函数，调用 Base.metadata.create_all 建表
7. 创建 backend/database/models.py 文件，定义三张表：
   - **OperationLog 表**（operation_logs）：
     - id：自增主键
     - action_type：字符串，记录操作类型（checkin / homework_submit / leave / login）
     - content：字符串，操作内容描述
     - result：字符串，可为空，操作结果
     - created_at：日期时间，默认当前时间
   - **NotificationHistory 表**（notification_history）：
     - id：自增主键
     - campus_notif_id：整型，可为空，校园端通知的 ID
     - title：字符串，通知标题
     - content：文本，通知内容
     - notif_type：字符串，通知类型（grade_notice / leave_result / announcement 等）
     - is_read：布尔，默认 false，是否已读
     - received_at：日期时间，默认当前时间，收到时间
   - **Memo 表**（memos）：
     - id：自增主键
     - title：字符串，标题
     - content：文本，内容
     - is_important：布尔，默认 false，是否重要
     - created_at：日期时间，默认当前时间
     - updated_at：日期时间，默认当前时间，更新时自动更新
8. 创建 backend/main.py：
   - 创建 FastAPI 应用
   - 标题设为"学生端 API"
   - 添加 CORS 中间件，允许 http://localhost:5173
   - 在 startup 事件中调用 init_db 初始化数据库
   - 创建 GET /api/health 健康检查接口，返回 {"status": "ok"}
   - main 入口用 uvicorn 启动，端口 8002，带 reload
9. 创建 backend/router/__init__.py（空文件）
10. 创建 backend/mcp_client.py：
    - 创建 CampusMCPClient 类
    - __init__ 接收 server_url 参数，默认 http://localhost:8001
    - 定义 session_id 属性，初始 None
    - 定义 _client 属性，初始 None
    - 实现 async connect() 方法：
      - 创建 httpx.AsyncClient
      - 用 GET 请求连接 {server_url}/sse
      - 遍历响应行，找到 data: 开头且包含 session_id 的行
      - 解析 JSON 提取 session_id 并保存
      - 如果连接失败抛出异常
    - 实现 async call_tool(tool_name, args) 方法：
      - 如果未连接或 session_id 为空，抛出异常
      - 构造 JSON-RPC 请求体：jsonrpc 2.0、id 1、method "tools/call"、params 包含 name 和 arguments
      - 用 POST 请求 {server_url}/messages/?session_id=xxx
      - 解析响应 JSON
      - 如果响应中有 result.content，取第一条 content 的 text 字段返回
      - 否则返回整个响应的字符串表示
    - 实现 async close() 方法，关闭 HTTP 客户端
    - 在模块级别创建全局单例 mcp_client

**验证方式：**
- 前端执行 npm run dev，浏览器打开 http://localhost:5173 能看到页面
- 后端执行 python main.py，终端显示 "Uvicorn running on http://0.0.0.0:8002"
- 浏览器打开 http://localhost:8002/docs 能看到 Swagger 文档
- 访问 http://localhost:8002/api/health 返回 {"status": "ok"}

---

## 🔧 提示词 2：后端 MCP 连接 + 注册登录 + 学生信息接口

> 📖 **参考文档：**
> - `学生端对接文档.md` → 查看 get_class_list、student_register、student_login、get_student_info、get_schedule、get_courses 的参数和返回值
> - `学生端项目开发文档.md` → MCP 对接关系一览表
>
> **角色：** 你是一个全栈工程师。

**前提：** 项目骨架已创建，包括前端、后端、数据库、MCP 客户端。

**任务：** 实现后端 MCP 连接初始化和三个路由模块。

### 第一步：修改 main.py

1. 在 main.py 的 startup 事件中，在 init_db 之后，添加连接 MCP 的代码：
   - 导入 mcp_client
   - 尝试调 mcp_client.connect()
   - 如果成功，打印成功日志
   - 如果失败，打印警告日志，不阻止应用启动
2. 导入并注册三个路由模块：auth_router、student_router、checkin_router、homework_router、leave_router、notification_router、local_router
   - 注意这里先注册好所有路由的占位，路由文件后面逐个创建

### 第二步：创建 router/auth.py

注册登录路由模块，包含三个接口：

1. **GET /api/classes**
   - 调用 mcp_client.call_tool("get_class_list")
   - 将 MCP 返回的文本直接作为 result 字段返回
   - 返回格式：{"result": "文本内容"}

2. **POST /api/register**
   - 接收请求体：phone（字符串）、password（字符串）、name（字符串）、class_id（整型，默认0）
   - 调用 mcp_client.call_tool("student_register", {phone, password, name, class_id})
   - 将 MCP 返回的文本作为 result 返回
   - 返回格式：{"result": "文本内容"}

3. **POST /api/login**
   - 接收请求体：account（字符串，可以是手机号或学号）、password（字符串）
   - 调用 mcp_client.call_tool("student_login", {account, password})
   - 将 MCP 返回的文本作为 result 返回
   - 返回格式：{"result": "文本内容"}

### 第三步：创建 router/student.py

学生信息路由模块，包含三个接口，每个接口都需要解析 MCP 返回的文本：

1. **GET /api/student/info**
   - 接收查询参数：student_id（字符串）
   - 调用 mcp_client.call_tool("get_student_info", {student_id})
   - 解析返回文本：
     - 文本格式类似 "姓名：张三\n学号：20260001\n班级：高一(1)班\n手机：138xxxx"
     - 按换行符分割，每行按冒号（中文冒号或英文冒号）分割成 key 和 value
     - 去掉 key 和 value 两端的空格
     - 组合成 JSON 对象
   - 返回格式：{"data": {"name": "张三", "student_id": "20260001", ...}}

2. **GET /api/schedule/today**
   - 接收查询参数：student_id（字符串）、day_of_week（整型，可选，默认0表示今天）
   - 调用 mcp_client.call_tool("get_schedule", {student_id, day_of_week})
   - 返回格式：{"data": "MCP返回的文本"}

3. **GET /api/courses**
   - 接收查询参数：student_id（字符串）
   - 调用 mcp_client.call_tool("get_courses", {student_id})
   - 返回格式：{"data": "MCP返回的文本"}

### 第四步：注册路由

在 main.py 中：
- 从 router 导入 auth、student
- 创建 router 实例并注册

---

## 🔧 提示词 3：后端签到 + 作业 + 请假接口

> 📖 **参考文档：**
> - `学生端对接文档.md` → 查看 do_checkin、get_checkin_records、get_checkin_stats、get_pending_homeworks、get_all_homeworks、get_homework_detail、submit_homework、get_homework_grades、apply_leave 的参数和返回值
>
> **角色：** 你是一个全栈工程师。

**前提：** 后端已有 MCP 连接、注册登录、学生信息接口。数据库有 operation_logs 表。

**任务：** 创建三个路由模块，每个操作成功后都要写入本地操作日志。

### 第一步：创建 router/checkin.py

签到路由模块，包含四个接口：

1. **POST /api/checkin**
   - 接收请求体：student_id（字符串）、code（字符串，6位签到码）
   - 调用 mcp_client.call_tool("do_checkin", {student_id, code})
   - 不管成功失败，都写入 operation_logs 表：
     - action_type = "checkin"
     - content = f"签到码：{code}"
     - result = MCP 返回的文本（截取前100字符）
   - 返回格式：{"result": "MCP返回的文本"}

2. **GET /api/checkin/records**
   - 接收查询参数：student_id（字符串）、days（整型，可选，默认7）
   - 调用 mcp_client.call_tool("get_checkin_records", {student_id, days})
   - 返回格式：{"data": "MCP返回的文本"}

3. **GET /api/checkin/stats**
   - 接收查询参数：student_id（字符串）、days（整型，可选，默认30）
   - 调用 mcp_client.call_tool("get_checkin_stats", {student_id, days})
   - 返回格式：{"data": "MCP返回的文本"}

### 第二步：创建 router/homework.py

作业路由模块，包含五个接口：

1. **GET /api/homeworks/pending**
   - 接收查询参数：student_id（字符串）
   - 调用 mcp_client.call_tool("get_pending_homeworks", {student_id})
   - 返回格式：{"data": "MCP返回的文本"}

2. **GET /api/homeworks/all**
   - 接收查询参数：student_id（字符串）
   - 调用 mcp_client.call_tool("get_all_homeworks", {student_id})
   - 返回格式：{"data": "MCP返回的文本"}

3. **GET /api/homeworks/detail**
   - 接收查询参数：homework_id（整型）、student_id（字符串，可选）
   - 调用 mcp_client.call_tool("get_homework_detail", {homework_id, student_id})
   - 返回格式：{"data": "MCP返回的文本"}

4. **POST /api/homeworks/submit**
   - 接收请求体：student_id（字符串）、homework_id（整型）、content（字符串）
   - 调用 mcp_client.call_tool("submit_homework", {student_id, homework_id, content})
   - 写入 operation_logs：
     - action_type = "homework"
     - content = f"提交作业ID：{homework_id}"
     - result = MCP 返回文本（前100字符）
   - 返回格式：{"result": "MCP返回的文本"}

5. **GET /api/homeworks/grades**
   - 接收查询参数：student_id（字符串）
   - 调用 mcp_client.call_tool("get_homework_grades", {student_id})
   - 返回格式：{"data": "MCP返回的文本"}

### 第三步：创建 router/leave.py

请假路由模块，包含一个接口：

1. **POST /api/leave**
   - 接收请求体：student_id（字符串）、course_id（整型）、reason（字符串）
   - 调用 mcp_client.call_tool("apply_leave", {student_id, course_id, reason})
   - 写入 operation_logs：
     - action_type = "leave"
     - content = f"请假课程ID：{course_id}，原因：{reason}"
     - result = MCP 返回文本（前100字符）
   - 返回格式：{"result": "MCP返回的文本"}

### 第四步：注册路由

在 main.py 中注册 checkin、homework、leave 的路由。

---

## 🔧 提示词 4：后端通知轮询 + 本地功能接口

> 📖 **参考文档：** `学生端项目开发文档.md` → 查看本地数据库表结构（notification_history、memos、operation_logs 的字段定义）
>
> **角色：** 你是一个全栈工程师。

**前提：** 数据库有 notification_history 和 memos 表。operation_logs 已有数据。

**任务：** 创建两个路由模块。

### 第一步：创建 router/notification.py

通知路由模块，包含三个接口：

1. **GET /api/notifications**
   - 接收查询参数：student_id（字符串）、since_id（整型，可选，默认0）
   - 调用 mcp_client.call_tool("get_notifications", {student_id})
   - 将 MCP 返回的文本缓存到本地 notification_history 表：
     - 创建一条记录，title="新通知"，content=MCP返回文本（截取前500字符），notif_type="notification"
   - 注意缓存时用数据库会话，用完关闭
   - 返回格式：{"result": "MCP返回文本", "list": []}
   - list 字段先返回空数组，后续前端轮询使用

2. **GET /api/notifications/history**
   - 查询 notification_history 表，按 id 降序，取最近50条
   - 将每条记录转成字典返回：
     - id、title、content（截取前100字符）、type（原名 notif_type）、is_read、time（received_at 格式化 %m-%d %H:%M）
   - 返回格式：{"data": [每条通知的字典]}

3. **POST /api/notifications/read**
   - 接收查询参数或请求体：notif_id（整型）
   - 查询 notification_history 表找到该记录，将 is_read 设为 true
   - 提交修改
   - 返回格式：{"status": "ok"}

### 第二步：创建 router/local.py

本地功能路由模块，包含五个接口：

1. **GET /api/local/logs**
   - 接收查询参数：action_type（字符串，可选，默认空字符串表示全部）
   - 查询 operation_logs 表，按 id 降序，取最近100条
   - 如果 action_type 不为空且不为 "all"，按 action_type 过滤
   - 每条记录转成字典：id、action_type、content、result、created_at（格式化为 %m-%d %H:%M）
   - 返回格式：{"data": [每条日志的字典]}

2. **GET /api/local/memos**
   - 查询 memos 表，按 id 降序
   - 每条记录转成字典：id、title、content、is_important、created_at（格式化）
   - 返回格式：{"data": [每条备忘的字典]}

3. **POST /api/local/memos**
   - 接收请求体：title（字符串）、content（字符串）、is_important（布尔，可选）
   - 创建一条 memo 记录存入数据库
   - 刷新后返回新记录的 id
   - 返回格式：{"status": "ok", "id": 新ID}

4. **DELETE /api/local/memos/{memo_id}**
   - 接收路径参数：memo_id（整型）
   - 从数据库删除该记录
   - 返回格式：{"status": "ok"}

5. **POST /api/local/logs**（可选，用于测试或离线记录）
   - 接收请求体：action_type、content、result
   - 直接写入 operation_logs 表
   - 返回格式：{"status": "ok"}

### 第三步：注册路由

在 main.py 中注册 notification 和 local 的路由。

---

## 🔧 提示词 5：前端路由配置 + 登录页 + 注册页

> 📖 **参考文档：** `学生端对接文档.md` → 查看 student_register、student_login、get_class_list 的说明（理解登录注册的数据来源）
>
> **角色：** 你是一个全栈工程师。

**前提：** 所有后端接口已实现。前端 Vite + React + TypeScript 项目已创建。

**任务：** 配置前端路由，实现登录页（三种状态）和注册页（三步流程）。

### 第一步：配置路由（修改 src/App.tsx）

1. 安装 react-router-dom（如果还没装）
2. 创建 routes 配置：
   - /login → Login 页面
   - /register → Register 页面
   - /home → Home 页面（需要登录）
   - /checkin → Checkin 页面（需要登录）
   - /homework → Homework 页面（需要登录）
   - /homework/:id → HomeworkDetail 页面（需要登录）
   - /schedule → Schedule 页面（需要登录）
   - /leave → Leave 页面（需要登录）
   - /notifications → Notifications 页面（需要登录）
   - /logs → Logs 页面（需要登录）
   - /profile → Profile 页面（需要登录）
   - / → 重定向到 /home

3. 创建 RequireAuth 组件：
   - 从 localStorage 读取 token
   - 如果 token 不存在，重定向到 /login
   - 如果 token 存在，渲染子组件
   - 用于包裹所有需要登录的页面路由

4. 先创建所有页面的占位组件（返回一个带页面名称的 div），确保路由配置不报错

### 第二步：实现登录页（src/pages/Login.tsx）

页面包含以下元素和功能：

**页面布局：**
- 居中布局，最大宽度 400px，上下左右 padding 40px
- 顶部标题："🎒 校园助手"（大字号）
- 副标题："学生端"（小字号，灰色）
- 账号输入框：placeholder "学号 / 手机号"
- 密码输入框：type password，placeholder "密码"
- 登录按钮：蓝色背景，白色文字，"登录"文字
- "没有账号？去注册" 链接，点击跳转到 /register

**三种状态处理：**

状态一：初始状态
- 输入框为空
- 登录按钮可点击
- 没有错误提示

状态二：输入验证
- 如果账号为空，提示"请输入账号"
- 如果密码为空，提示"请输入密码"
- 验证不通过时，在按钮上方显示红色错误文字
- 验证通过后，发起登录请求

状态三：登录请求中
- 按钮文字变为"登录中..."
- 按钮禁用，防止重复提交
- 调后端 POST /api/login，发送 {account, password}

状态四：登录成功
- 解析返回的 result 文本
- 从文本中提取 Token 值（格式：Token：xxx）
- 保存 token 到 localStorage
- 保存 student_id 到 localStorage（就是输入的 account）
- 跳转到 /home

状态五：登录失败
- 解析返回的 result 文本
- 如果包含"❌"，显示错误信息
- 按钮恢复可点击
- 文字恢复为"登录"

状态六：网络错误
- 如果 fetch 抛出异常，提示"无法连接服务器"
- 按钮恢复可点击

### 第三步：实现注册页（src/pages/Register.tsx）

**三步注册流程：**

**第一步：手机号和密码**
- 标题："注册"
- 手机号输入框：placeholder "手机号"
- 密码输入框：type password，placeholder "密码（至少6位）"
- 验证：手机号不为空、密码至少6位
- "下一步" 按钮
- "已有账号？去登录" 链接
- 错误提示区域（红色文字）
- 点击下一步时验证输入，通过后进入第二步

**第二步：姓名和班级选择**
- 标题："完善信息"
- 姓名输入框：placeholder "请输入姓名"
- 班级下拉选择框：
  - 进入此步时自动调 GET /api/classes 获取班级列表
  - 请求期间显示"加载班级列表..."
  - 解析返回的 result 文本：
    - 按换行分割，找到包含 📋 的行
    - 每行格式：📋 1. 班级名称（年级）
    - 提取数字 ID 和中文名称
    - 如果文本包含"暂无班级"，显示提示信息，下拉框显示"暂无可选班级"
  - 下拉框第一个选项："请选择班级"（value=0）
  - 后续选项从班级列表生成
- "完成注册" 按钮
- "上一步" 按钮，点击回到第一步（保留已填信息）
- 错误提示区域
- 点击完成注册时调 POST /api/register
  - 请求体：{phone, password, name, class_id}
  - 请求期间按钮禁用，文字变"注册中..."
  - 成功后解析 result，提取"学号：xxx" 中的学号

**第三步：注册成功**
- 标题："🎉 注册成功！"
- 学号大字显示（系统生成的学号）
- 提示文字："这是你的学号，请保存好"
- "去登录" 按钮，点击跳转到 /login
- 如果注册失败，显示错误信息，可返回修改

---

## 🔧 提示词 6：前端首页 + 签到页 + 作业页 + 课表页

> 📖 **参考文档：** `学生端对接文档.md` → 查看 do_checkin、get_checkin_records、get_checkin_stats、get_pending_homeworks、get_all_homeworks、get_schedule 的说明（理解页面数据从哪来）
>
> **角色：** 你是一个全栈工程师。

**前提：** 路由已配置，登录注册页面已完成。

**任务：** 实现四个核心功能页面。

### 第一步：首页（src/pages/Home.tsx）

**页面加载：**
- 进入页面时，从 localStorage 读取 student_id
- 同时发起三个请求（用 Promise.all）：
  - GET /api/student/info?student_id=xxx
  - GET /api/schedule/today?student_id=xxx
  - GET /api/homeworks/pending?student_id=xxx
- 三个请求全部完成后，更新页面数据

**页面布局（从上到下）：**

个人信息卡片：
- 浅蓝背景，圆角卡片
- 显示返回数据中的 name 和 student_id
- 如果数据还没加载完，显示"加载中..."

今日课表卡片：
- 灰色背景，圆角卡片
- 标题："📅 今日"
- 显示课表文本
- 如果课表文本为空或包含"没有课程"，显示"今天没有课 🎉"

待办作业卡片：
- 灰色背景，圆角卡片
- 标题："📝 作业"
- 显示待交作业文本

快捷操作按钮（一行三个，等宽）：
- 📍 签到 → 蓝色背景，点击跳转到 /checkin
- 📝 作业 → 紫色背景，点击跳转到 /homework
- 📅 课表 → 绿色背景，点击跳转到 /schedule

次级操作按钮（一行三个，等宽，白色背景，灰色边框）：
- 📋 请假 → 点击跳转到 /leave
- 🔔 通知 → 点击跳转到 /notifications
- 📋 记录 → 点击跳转到 /logs

### 第二步：签到页（src/pages/Checkin.tsx）

**页面布局：**

顶部：返回按钮（← 返回），点击回到上一页
标题："📍 签到"

签到码输入区：
- 一个输入框，最大长度6位
- 居中对齐，大字号（20px），字间距加宽
- placeholder "输入6位签到码"
- 下方是"确认签到"按钮，蓝色背景

签到结果展示：
- 签到完成后，在按钮下方显示结果文字
- 成功（包含 ✅）显示绿色
- 失败显示灰色

签到记录区：
- 分隔线
- 小标题："📋 签到记录"
- 进入页面时自动加载最近7天记录（GET /api/checkin/records）
- 签到成功后刷新记录
- 显示文本（保持换行格式）

**三种状态：**
- 初始状态：输入框为空，没结果，有记录列表
- 签到中：按钮禁用，文字"签到中..."
- 签到完成：显示结果，刷新记录，按钮恢复

### 第三步：作业页（src/pages/Homework.tsx）

**页面布局：**

顶部：返回按钮
标题："📝 作业"

Tab 切换栏（两个按钮）：
- "待提交" → 激活时蓝色背景白色文字
- "全部" → 激活时蓝色背景白色文字
- 未激活时灰色背景
- 切换 Tab 重新加载数据
- 默认选中"待提交"

作业列表区：
- 白色背景，圆角卡片
- 显示 MCP 返回的文本（保持换行）
- Tab 切换时重新调接口：
  - "待提交" → GET /api/homeworks/pending
  - "全部" → GET /api/homeworks/all
- 加载中显示"加载中..."

底部按钮：
- "📊 查看成绩" → 紫色背景，点击跳转到作业成绩页或弹窗显示 GET /api/homeworks/grades 的结果

### 第四步：课表页（src/pages/Schedule.tsx）

**页面布局：**

顶部：返回按钮
标题："📅 课表"

星期选择器（横向滚动）：
- 8个按钮：今天、周一、周二、周三、周四、周五、周六、周日
- 选中项蓝色背景白色文字
- 未选中灰色背景
- 圆角胶囊形状
- 默认选中"今天"

课程展示区：
- 灰色背景，圆角卡片
- 显示 MCP 返回的课表文本
- 切换星期时重新调 GET /api/schedule/today?student_id=xxx&day_of_week=N
  - day_of_week 参数：0=今天，1=周一...7=周日
- 加载中显示"加载中..."

---

## 🔧 提示词 7：前端请假页 + 通知页 + 操作记录页

> 📖 **参考文档：**
> - `学生端对接文档.md` → 查看 apply_leave、get_notifications 的说明
> - `学生端项目开发文档.md` → 查看本地数据库操作记录的设计
>
> **角色：** 你是一个全栈工程师。

**前提：** 核心页面（首页、签到、作业、课表）已完成。

**任务：** 实现三个功能页面。

### 第一步：请假页（src/pages/Leave.tsx）

**页面布局：**

顶部：返回按钮
标题："📋 请假"

表单区域：
- 课程ID输入框：placeholder "课程ID"，数字输入
- 请假原因输入框：多行文本，高度100px，placeholder "请假原因"
- "提交请假" 按钮，蓝色背景

**三种状态：**
- 初始状态：输入框为空，按钮可点击
- 提交中：按钮禁用，文字"提交中..."
- 提交完成：显示结果文字
  - 成功（包含 ✅）绿色
  - 失败灰色

**数据流：**
1. 用户填写课程ID和原因
2. 如果任一为空，提示"请填写完整信息"
3. 调 POST /api/leave
4. 请求体：{student_id, course_id, reason}
5. 显示结果

### 第二步：通知页（src/pages/Notifications.tsx）

**页面布局：**

顶部：返回按钮
标题："🔔 通知"

通知列表：
- 进入页面时调 GET /api/notifications/history
- 如果返回空数据，显示"暂无通知"
- 每条通知是一个卡片：
  - 未读：浅蓝背景（#f0f4ff）
  - 已读：浅灰背景（#f9f9f9）
  - 显示标题（加粗）
  - 显示内容摘要（小字号，灰色）
  - 显示时间（最小字号，浅灰）
  - 点击通知：
    - 调 POST /api/notifications/read?notif_id=xxx
    - 切换为已读样式

**数据流：**
1. 进入页面调 GET /api/notifications/history
2. 同时调一次 GET /api/notifications 轮询新通知（同步缓存到本地）
3. 后续新通知通过全局 Toast 展示（在提示词8实现）

### 第三步：操作记录页（src/pages/Logs.tsx）

**页面布局：**

顶部：返回按钮
标题："📋 操作记录"

筛选栏（横向滚动）：
- 4个胶囊按钮：全部、📍签到、📝作业、📋请假
- 选中项蓝色背景白色文字
- 默认选中"全部"
- 切换时重新请求数据

记录列表：
- 调 GET /api/local/logs?action_type=xxx
  - action_type 参数：all / checkin / homework / leave
- 每条记录：
  - 图标（签到 📍、作业 📝、请假 📋）
  - 内容文字
  - 结果文字（小字号，灰色）
  - 时间（最小字号，浅灰）
  - 灰色背景卡片
- 如果数据为空，显示"暂无记录"

---

## 🔧 提示词 8：前端全局通知轮询 + Toast 展示

> 📖 **参考文档：** `学生端项目开发文档.md` → 查看校园通知（新增）章节，了解通知轮询的设计思路
>
> **角色：** 你是一个全栈工程师。

**前提：** 所有页面已完成。

**任务：** 实现后台轮询和右上角 Toast 弹出。

### 第一步：创建轮询模块

创建 src/api/notifications.ts 文件：

1. 定义变量：
   - lastNotifId：从 localStorage 读取，默认0
   - queue：存储待显示通知的数组
   - isShowing：布尔，当前是否有 Toast 正在显示

2. 实现 startPolling 函数：
   - 设置定时器，每10秒执行一次 poll
   - 首次延迟3秒后执行一次 poll
   - 该函数会在 main.tsx 中调用

3. 实现 poll 异步函数：
   - 从 localStorage 读取 student_id
   - 如果没有 student_id，直接返回（未登录）
   - 调 GET /api/notifications?student_id=xxx&since_id=lastNotifId
   - 解析响应的 list 字段
   - 如果有新通知，逐个加入 queue 数组
   - 更新 lastNotifId 并保存到 localStorage
   - 调 showNext 开始展示队列
   - 如果请求失败，静默处理

4. 实现 showNext 函数：
   - 如果 isShowing 为 true 或 queue 为空，直接返回
   - isShowing 设为 true
   - 从 queue 取出第一个通知
   - 调 showToast 显示
   - showToast 接收一个回调函数，展示完成后调回调
   - 回调中：isShowing 设为 false，再调 showNext

5. 实现 showToast 函数：
   - 创建 div 元素
   - 用内联 style 设置以下样式：
     - 固定定位，top: 64px，right: 16px
     - z-index: 9999
     - flex 布局，align-items: center，gap: 12px
     - 背景：紫色渐变 linear-gradient(135deg, #667eea, #764ba2)
     - 白色文字
     - 圆角 10px
     - padding: 10px 20px
     - 最小宽度 300px
     - 阴影
     - 初始 transform: translateX(120%)，transition 0.4s
   - 内部结构：
     - 图标（🔔，大字）
     - 文字区域：
       - 标题（加粗）
       - 内容（半透明，单行溢出省略）
   - 添加到 body
   - 下一帧添加 show 类（transform: translateX(0)）
   - 5秒后移除 show 类（transform: translateX(120%)）
   - 0.4秒后移除 DOM 元素，调完成回调

### 第二步：启动轮询

修改 src/main.tsx：
- 导入 startPolling
- 在 React.StrictMode 之后调用 startPolling()

### 测试方式

手动向 notification_history 表插入一条数据来测试：
```sql
INSERT INTO notification_history (title, content, notif_type) VALUES ('测试通知', '这是一条测试通知', 'notification');
```
然后刷新前端页面，等待10秒内应该能看到 Toast 弹出。

---

## ✅ 最终验证清单

| 功能 | 验证方式 |
|:-----|:---------|
| 登录 | 输入学号和密码，能登录成功跳转到首页 |
| 注册 | 三步注册流程正常，能选班级，显示学号 |
| 首页 | 显示个人信息、今日课表、待办作业 |
| 签到 | 能输入签到码提交，显示签到记录 |
| 作业 | Tab 切换正常，能看详情提交作业 |
| 课表 | 切换星期显示不同课表 |
| 请假 | 能填写提交，本地记录写入 |
| 通知 | 显示通知历史，已读未读可切换 |
| 操作记录 | 筛选正常，显示本地操作历史 |
| Toast | 新通知自动弹出右上角，5秒消失 |

> 文档版本：v2.0 | 2026-06-25
