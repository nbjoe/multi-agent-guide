# LobsterAI 多 Agent 协作开发指令

基于 [LobsterAI OpenClaw](https://github.com/openclaw/openclaw) 的多 Agent 协作开发框架。通过主 Agent 调度 + Worker Agent 执行的架构，在不切换平台的前提下实现分工协作，大幅降低单次对话的 token 消耗。

---

## 操作说明

**两步开始：**

1. 打开 LobsterAI 对话窗口
2. 把 `MAIN-AGENT-INSTRUCTIONS.md` 和 `WORKER-AGENT-INSTRUCTIONS.md` 拖进对话框

然后告诉 Main Agent 你要做什么，它会自动按流程调度 Worker 完成任务。

**就是这么简单。** 不需要额外配置、不需要安装插件、不需要切换平台。

---

## 为什么需要这个？

单 Agent 开发大型项目时，每次对话都要加载完整上下文（通常 20-30 万 tokens），成本高且容易丢失焦点。

本方案将任务拆分给独立的 Worker Agent，每个 Worker 只加载自己需要的上下文（2-5 万 tokens），主 Agent 只做调度和验收，**token 消耗降低 80% 以上**。

---

## 架构

```
你（主会话，手动调度 + 最终验收）
│
├── Worker-A（模块A → 独立子会话）
├── Worker-B（模块B → 独立子会话）
├── Worker-C（模块C → 独立子会话）
└── ...（按需扩展）

共享文档
├── ARCHITECTURE.md  → 项目架构、技术栈、目录结构
├── INTERFACES.md    → 接口约定、数据格式
├── TASKS.md         → 任务清单、进度、依赖关系
└── LOG.md           → 执行日志
```

---

## 文件说明

| 文件 | 用途 | 适用角色 |
|------|------|---------|
| `MAIN-AGENT-INSTRUCTIONS.md` | 主 Agent 系统指令 | Main Agent |
| `WORKER-AGENT-INSTRUCTIONS.md` | Worker Agent 行为规范 | Worker Agent |
| `multi-agent-guide.md` | 使用指南（人类可读） | 人类用户 |
| `README.md` | 项目说明 | 所有人 |

---

## 技能映射

Worker 的技能不是固定的，Main Agent 会根据任务类型动态分配。以下是 LobsterAI 内置 skill 的映射参考：

| 任务类型 | 必选技能 | 推荐技能 |
|---------|---------|---------|
| 前端开发 | frontend-design | code-review |
| 后端 API | code-review | documentation |
| 数据库 | data-analysis | code-review |
| 测试 | testing-strategy | code-review, debug |
| 文档 | documentation | article-writer |
| 数据可视化 | create-viz | data-visualization |
| Excel/Word/PPT | xlsx, docx, pptx | documentation |

**特殊场景处理：** 如果任务涉及数据库迁移、跨服务事务等复杂场景，Main Agent 会在检查点1 和你商讨专门的指引，而不是自行猜测。

---

## 快速开始

### 1. 准备共享文档

在项目根目录创建：

```markdown
# ARCHITECTURE.md — 项目架构
- 技术栈：[框架、语言]
- 目录结构：[各模块说明]
- 命名规范：[文件/变量命名规则]
```

```markdown
# INTERFACES.md — 接口约定
- 数据格式：[JSON 结构定义]
- API 约定：[REST/gRPC 规范]
```

```markdown
# TASKS.md — 任务清单
## Worker-A
- [ ] 任务1：[描述]
## Worker-B
- [ ] 任务2：[描述]
```

### 2. 使用流程

```
你说任务 → 检查点1（理解确认）→ 检查点2（拆分方案）→ 检查点3（Worker设立）→ spawn Worker → 验收
```

简单任务走**快速通道**（跳过检查点1）。

### 3. spawn Worker 示例

```
spawn Worker-A，任务是完成商品列表页，需要读 module-a/README.md
```

Worker 会自动读取共享文档、执行开发任务、汇报结果。

---

## 核心机制

### 三道检查点

| 检查点 | 内容 | 产出 |
|--------|------|------|
| 1 | 任务理解 + 技能匹配审查 + 复杂场景评估 | 用户确认理解正确 |
| 2 | 子任务拆分 + 依赖关系 + 执行顺序 | 用户确认拆分方案 |
| 3 | Worker 身份、技能、任务分配 | 用户确认 Worker 配置 |

### Worker 失败处理

1. **不达标** → spawn 修复 Worker
2. **崩溃/报错** → 检查文件冲突后重试
3. **两次失败** → 上报用户，人工介入

### 并行模式

满足以下条件时可并行 Worker：
- 各 Worker 操作目录完全不重叠
- 任务之间无依赖
- 用户明确同意

---

## Token 消耗对比

| 方式 | 每次对话消耗 | 说明 |
|------|------------|------|
| 主会话直接写代码 | ~285k tokens | 加载全部上下文 |
| spawn Worker | 2-5万 tokens | 只加载任务描述 |

**典型项目总消耗：**
- 主会话调度：~10 万 tokens
- Worker-A：~5 万 tokens
- Worker-B：~5 万 tokens
- **总计：~20 万 tokens**（比直接写省 80%+）

---

## 适用场景

- 多模块 Web 应用开发
- 前后端分离项目
- 需要多角色协作的开发任务（前端、后端、测试、文档）

## 局限性

- 技能映射矩阵基于 LobsterAI 内置 skill，不适用于所有技术栈
- 数据库迁移、跨服务事务等复杂场景需要额外指引
- Worker 是一次性执行，不适合需要持续迭代的长期任务

---

**平台：** [LobsterAI OpenClaw](https://github.com/openclaw/openclaw)
**文档版本：** v2.1
