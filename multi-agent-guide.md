# LobsterAI 多 Agent 协作开发指南

## 一、架构概览

```
你（主会话，手动调度 + 最终验收）
│
├── Worker-A（模块A → 独立子会话）
├── Worker-B（模块B → 独立子会话）
├── Worker-C（模块C → 独立子会话）
└── ...（按需扩展）

共享文档（项目根目录）
├── ARCHITECTURE.md  → 项目架构、技术栈、目录结构
├── INTERFACES.md    → 接口约定、数据格式
└── TASKS.md         → 任务清单、进度、依赖关系
```

**核心原则：** 主会话做调度，Worker 做执行，共享文档做协调。

---

## 二、Token 消耗对比

| 方式 | 每次对话消耗 | 说明 |
|------|------------|------|
| 主会话直接写代码 | 285k tokens | 加载全部上下文 |
| spawn Worker 子会话 | 2-5万 tokens | 只加载任务描述 |

**省 token 原理：** Worker 是独立会话，不共享主会话的 285k 上下文，只读你给它的任务片段。

---

## 三、操作流程

### 步骤 1：准备共享文档（你亲自做，约 15 分钟）

#### ARCHITECTURE.md（项目大脑）

```markdown
# 项目架构说明
- 技术栈：[框架、组件库、语言]
- 目录结构：
  - module-a/ → [模块A说明]
  - module-b/ → [模块B说明]
  - shared/ → 公共组件（你统一管理）
- 命名规范：[文件命名、变量命名规则]
- 已完成模块：[当前进度]
```

#### INTERFACES.md（接口契约）

```markdown
# 接口约定
## 数据格式
用户：{ id, name, email, role }
订单：{ id, userId, items, status, createdAt }

## API 约定
GET /api/users?page=1 → { total, list }
POST /api/users → { id }

## 规则
- 返回格式：{ code, data, message }
- 字段命名：驼峰命名
- 认证方式：Bearer Token
```

#### TASKS.md（任务清单）

```markdown
# 任务进度

## Worker-A（模块A）
- [ ] 任务1：[描述]
- [ ] 任务2：[描述]

## Worker-B（模块B）
- [ ] 任务1：[描述]
- [ ] 任务2：[描述]

## 依赖关系
第一层（可并行）：任务1-A, 任务1-B
第二层（依赖第一层）：任务2-A → 依赖任务1-A

## 问题清单
（阻塞问题记录在这里）
```

---

### 步骤 2：调度 Worker（主会话操作）

**你说：**
> "spawn Worker-A，任务是完成商品列表页，需要读 module-a/README.md"

**我帮你组装的 task 描述：**
```
你是 Worker-A，负责商品模块。

## 开始前
读取以下文件了解项目：
- ARCHITECTURE.md（项目架构）
- INTERFACES.md（接口约定）
- TASKS.md（你的任务）

## 本次任务
在 module-a/ 下完成商品列表页

## 规则
- 只操作 module-a/ 下的文件
- 使用 INTERFACES.md 中约定的数据格式
- 公共组件需要修改时，记录到 TASKS.md 问题清单

## 完成后
汇报：
1. 创建/修改了哪些文件
2. 代码摘要（100字以内）
3. 遇到的问题（如有）
```

---

### 步骤 3：Worker 执行（自动，无需你干预）

Worker 子会话会：
1. 读取任务描述中的文件内容
2. 执行开发任务
3. 保存文件到指定目录
4. 汇报结果

**耗时：** 通常 5-30 秒
**Token 消耗：** 2-5 万 tokens

---

### 步骤 4：你验收产出

Worker 完成后，你会收到：
- 文件路径
- 代码摘要
- 遇到的问题

**你决定：**
- ✅ 通过 → 告诉我继续下一个任务
- ❌ 不通过 → 告诉我哪里有问题，spawn 修正 Worker
- 🔧 手动调整 → 你自己改代码，然后继续

---

## 四、省 Token 技巧

### 1. Worker 只读必要文件
不要让 Worker 读全部文档，只给它需要的部分：
```
task: "读 module-a/README.md 了解模块结构，然后完成..."
```

### 2. 用 mode: "run" 一次性执行
Worker 干完就结束，不保持会话：
```json
{ "mode": "run" }
```

### 3. 主会话只做调度
主会话不写代码，只 spawn Worker + 验收，保持上下文精简。

### 4. 批量 spawn 并行 Worker
如果任务无依赖，可以同时 spawn 多个 Worker：
```
"同时 spawn Worker-A 和 Worker-B，分别完成..."
```

### 5. Worker 输出精简
要求 Worker 只汇报摘要，不返回完整代码：
```
"完成后只输出文件路径和100字摘要"
```

---

## 五、实际案例

### 场景：开发一个待办事项应用

**你说：**
> "帮我开发一个待办事项应用，前端用 React，后端用 Express，数据库用 SQLite"

**我做的：**

1. 创建共享文档（ARCHITECTURE.md、INTERFACES.md、TASKS.md）
2. 拆分任务：
   - Worker-A：后端 API（Express + SQLite）
   - Worker-B：前端页面（React）
3. spawn Worker-A 执行后端任务
4. 验收 Worker-A 产出
5. spawn Worker-B 执行前端任务
6. 验收 Worker-B 产出
7. 集成测试

**预估 Token 消耗：**
- 主会话调度：~10 万 tokens
- Worker-A（后端）：~5 万 tokens
- Worker-B（前端）：~5 万 tokens
- **总计：~20 万 tokens**（比主会话直接写省 90%）

---

## 六、注意事项

1. **不要让 Worker 修改 shared/ 目录** — 公共组件由你统一管理
2. **每次只 spawn 一个 Worker** — 避免文件冲突（除非任务无依赖）
3. **Worker 任务控制在 1-2 轮对话内完成** — 任务太大会拆分
4. **遇到阻塞问题记录到 TASKS.md** — 不要卡住，先跳过
5. **定期更新 ARCHITECTURE.md** — 记录新增文件和变更
6. **检查点1 包含技能匹配审查和复杂场景评估** — 每次任务开始前，Main Agent 会审查技能映射是否匹配、是否涉及数据库迁移/跨服务事务等非常规场景，如有需要会在检查点1和你商讨专门指引
7. **极简任务走快速通道** — 只涉及1个模块或用户说"直接做"的任务，跳过检查点1

---

## 七、快速参考

### spawn Worker 命令模板

```
spawn Worker-[名称]，任务是[具体描述]，需要读[文件列表]，完成后汇报[输出格式]
```

### 常用参数

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| mode | 执行模式 | "run"（一次性） |
| runtime | 运行时 | "subagent"（内置） |
| label | 任务标签 | "Worker-[名称]-[任务]" |

### Token 预估

| 角色 | 每次消耗 | 说明 |
|------|---------|------|
| 主会话调度 | 5-10 万 | 读文档 + 组装任务 |
| Worker 执行 | 2-5 万 | 只读任务描述 + 写代码 |
| 你验收 | 0 | 你自己看代码 |

---

**文档版本：** v1.0
**创建时间：** 2026-06-19
**适用平台：** LobsterAI OpenClaw
