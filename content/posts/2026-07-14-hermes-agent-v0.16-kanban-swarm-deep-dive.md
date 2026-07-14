---
title: "Hermes Agent v0.16 Kanban Swarm：多智能体协作的新范式"
date: 2026-07-14
tags: ["Hermes Agent", "多智能体", "Kanban", "Swarm", "AI Agent", "开源"]
author: "66sunny-D"
description: "深度解析 Hermes Agent v0.16 全新 Kanban Swarm 架构——为什么说它不是另一个 Agent 框架，而是一种基于 SQLite 持久化看板的智能体协作操作系统。"
---

# Hermes Agent v0.16 Kanban Swarm：多智能体协作的新范式

> **一句话概括：** Kanban Swarm 不是又一个 Agent 编排框架，而是一个基于 SQLite 持久化看板的智能体协作操作系统——智能体之间不对话，只读写看板。

当 OpenAI Swarm、CrewAI、AutoGen 还在研究"智能体之间怎么聊天"时，Hermes Agent v0.16 给出了一个截然不同的答案：**他们不应该聊天。他们应该共享一块板子。**

---

## 一、从 RPC 到持久化队列：设计哲学的转变

### 1.1 delegate_task 的局限

在 v0.16 之前，Hermes Agent 的多智能体协作主要依赖 `delegate_task`——一个 RPC 风格的子代理调用工具。父代理 spawn 子代理，阻塞等待子代理返回结果，然后继续执行。

这个模型对于"帮我查个天气"这种简单任务足够了，但在真实工程场景中暴露出几个结构性问题：

| 痛点 | 表现 |
|------|------|
| **无状态** | 子代理挂了就挂了，结果全丢 |
| **无持久化** | 父代理压缩上下文时，子代理的产出消失 |
| **无人类介入** | 子代理卡住了没法问人 |
| **单次绑定** | 一个任务对应一个子代理，没有重试/多阶段概念 |
| **层级耦合** | 父代理必须活着等子代理回来 |

### 1.2 Kanban 的解答：智能体之间不对话

Kanban Swarm 的核心洞察是：**智能体协作不该通过对话完成，而应通过共享状态完成。**

每个智能体是一台独立的"机器"，从看板上取任务、干活、写结果、放回看板。机器之间不需要互相认识，不需要知道对方在干什么——它们只需要读写同一块共享内存。

```python
# 智能体 A：调研北美市场
kanban_show()  # 读取任务描述
# ... 做调研 ...
kanban_complete(summary="北美 AI 融资活跃，种子轮均额 $2.5M")

# 智能体 B：调研欧洲市场（完全独立，并行执行）
kanban_show()  # 读取任务描述
# ... 做调研 ...
kanban_complete(summary="欧洲侧重监管合规，GDPR 合规成融资前提")

# 智能体 C：综合撰写（等 A 和 B 都完成后自动触发）
kanban_show()  # 读取两个父任务的产出
# 现在 C 手握 A 和 B 的结构化交付物，开始写文章
```

这三个智能体可以运行在不同的进程中、不同的主机上、不同的时间段——它们永远不会直接对话。这就是 Kanban Swarm 的第一性原理。

---

## 二、架构深度解析

### 2.1 数据模型：SQLite 即一切

整个 Kanban 系统建立在 SQLite 之上。不是 Redis、不是消息队列、不是 gRPC——就是 `/home/user/.hermes/kanban.db` 里的几张表。

```
┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│   tasks     │────▶│  task_runs    │     │  task_links  │
│             │     │               │     │              │
│ id          │     │ task_id       │     │ parent_id    │
│ title       │     │ status        │     │ child_id     │
│ body        │     │ worker        │     └──────────────┘
│ assignee    │     │ started_at    │     ┌──────────────┐
│ status      │     │ summary       │     │ task_events  │
│ priority    │     │ metadata      │     │              │
│ tenant      │     │ error         │     │ task_id      │
│ max_retries │     └───────────────┘     │ event_type   │
└─────────────┘                           │ payload      │
                                          └──────────────┘
```

关键设计决定：

- **任务不是状态机，而是事件流。** 每次状态变更（创建、认领、完成、阻塞、重试）都写入 `task_events` 表。看板仪表盘通过 WebSocket 消费这个事件流实现实时更新。
- **重试是一等公民。** 每次失败不会覆盖前一次的执行记录，而是向 `task_runs` 追加一行。新的子代理在 `kanban_show()` 时能看到完整的历史上下文。
- **元数据是结构化交付物。** `kanban_complete(metadata={...})` 写的不是自然语言总结，而是结构化的机器可读数据（changed_files、tests_run、decisions 等），下游智能体可以直接消费。

### 2.2 调度器：60 秒一次的心跳

调度器（Dispatcher）是 Kanban Swarm 的心脏。它运行在 Gateway 进程中（而不是独立的守护进程），默认每 **60 秒** 执行一次调度循环：

```
每个 tick：
  1. 回收过期认领 — kill(pid, 0) 检测崩溃的 worker
  2. 推进依赖 — parent 完成后自动将 child 从 todo → ready
  3. 认领并 spawn — 原子性认领 ready 任务，启动 worker 进程
  4. 断路器检查 — 连续 N 次失败后自动 blocked
```

调度器的关键设计：

- **原子认领**：使用 SQLite 的事务和行级锁。在 `kanban.db` 上执行 `UPDATE tasks SET status='claimed', claimed_by=pid WHERE status='ready' AND id=?`，并检查 affected_rows 是否为 1。与 RabbitMQ 的 basic.get 是同一个模式，只是数据库是 SQLite。
- **PID 检测**：`os.kill(pid, 0)` 检查 worker 是否还活着。如果 pid 消失了但任务状态还是 running，调度器收回任务、重新放回 ready 队列。
- **断路器**：`kanban.failure_limit = 2`（默认值）。连续 2 次 spawn 失败后，任务自动进入 blocked 状态，等待人类干预。防止在"配置错误"这种永久性故障上无限重试。
- **粘性重试**：--max-retries 3 允许覆盖全局限制。每次重试是新的 task_runs 行，旧的历史不会丢失。

### 2.3 Worker 生命周期

Worker 不是通过 shell 命令与看板交互的。调度器 spawn worker 时设置 `HERMES_KANBAN_TASK=t_xxx` 环境变量，这个变量会触发 Hermes 为该会话动态注入 `kanban_*` 工具集。

Worker 的标准生命周期：

```
1. kanban_show()                    ← 读取任务上下文
2. cd $HERMES_KANBAN_WORKSPACE      ← 进入工作目录
3. (执行真正的任务逻辑)
4. kanban_heartbeat(note="...")     ← 长时间任务的心跳（可选）
5. kanban_complete() 或 kanban_block()  ← 终结
```

三个核心设计决策：

**为什么用工具而不是 CLI？** 如果 worker 在 Docker/Modal/SSH 远程终端上运行，shell 里没有 `hermes` 命令，也没有 `~/.hermes/kanban.db`。但 `kanban_*` 工具运行在 agent 自身的 Python 进程中，永远能访问本地 SQLite 文件，与终端后端无关。

**为什么需要心跳？** 调度器在 `dispatch_stale_timeout_seconds`（默认 4 小时）后回收没有心跳的任务。如果你有个跑 3 小时的 CI 构建，但你每 5 分钟发了心跳，调度器就知道你还活着。

**协议违规检测：** 如果 worker 进程正常退出（exit code 0）但任务状态还是 running——说明模型直接输出了文本回复而没有调用 `kanban_complete()` 或 `kanban_block()`——调度器会将该任务 auto-block 并标记 protocol_violation。这强制 worker 遵守看板协议，而不是像普通聊天那样随意输出。

### 2.4 Orchestrator：不干活的设计师

Kanban Swarm 中有一个特殊角色：**Orchestrator**。它的工作不是"做任务"，而是"分解任务"。

```
Orchestrator 的一条黄金法则：
你可以创建任务、链接任务、评论任务——但绝不能自己去实现任务。
```

Orchestrator 的标准工作流：

```python
# 用户说："写一篇关于 AI 融资的文章"
kanban_create("调研北美 AI 融资", assignee="researcher-a")
kanban_create("调研欧洲 AI 融资", assignee="researcher-b")
kanban_create(
    "综合成文",
    assignee="writer",
    parents=["t_r1", "t_r2"]  # 两个调研都完成才触发
)
kanban_complete("分解完成：2 调研 → 1 写作")
```

Orchestrator 的注入系统提示包含：

- **反诱惑规则**：不要自己去实现任务，只做分解和协调
- **Profile 感知**：启动时通过 `hermes kanban profiles` 发现可用的 profile 列表，只将任务分配给实际存在的 profile
- **分解模式库**：预置了串行流水线、并行农场、扇出聚合、审核回路等模式

### 2.5 看板 Dashboard

除了 CLI 和工具接口，Kanban 还提供一个内嵌的 Web Dashboard：

```
hermes dashboard  # 打开 http://127.0.0.1:9119
```

Dashboard 的特性：

- **六列看板**：Triage → Todo → Ready → Running → Blocked → Done
  - Triage 是原始想法池。默认开启 Auto-Decompose，调度器会自动调用 decomposer 将其拆分；也可切换为 Manual 模式手动分解
  - Running 列支持"按 Profile 分组"，一眼看出每个 worker 在干什么
- **拖放操作**：拖拽卡片切换状态，状态变更通过 PATCH API 走与 CLI 相同的 kanban_db 层
- **实时更新**：通过 WebSocket 消费 task_events 表，秒级刷新
- **文件附件**：支持上传最高 25MB 的文件（PDF、图片、源代码），worker 启动时自动将这些文件的绝对路径注入上下文
- **批量操作**：Ctrl/Shift 多选卡片后批量完成、归档、阻塞

---

## 三、Kanban vs delegate_task：何时用哪个？

| 维度 | delegate_task | Kanban |
|------|-------------|--------|
| 调用模式 | RPC call (fork → join) | 消息队列 + 状态机 |
| 父级行为 | 阻塞等待 | 创建即遗忘 |
| 重试 | 无——失败即失败 | 自动重试 + 断路器 |
| 人类介入 | 不支持 | 注释/解阻塞，随时介入 |
| 审计追踪 | 上下文压缩时丢失 | SQLite 中永久保存 |
| 协调模型 | 层级化（调用者→被调用者） | 对等——任何 profile 读写任何任务 |
| 超时 | 默认无超时（之前版本有硬限制） | 4 小时 + 心跳扩展 |
| 适用场景 | 短推理、不需要人看结果 | 跨 agent 边界、需持久化、需人类介入 |

**可以混合使用：** 一个 Kanban worker 在干活时可以内部调用 `delegate_task` 做子推理。两者不互斥，只是抽象层次不同。

---

## 四、协作模式的八种范式

Kanban Swarm 的设计文档定义了八种协作模式。这里列出最常用的四种：

### 4.1 串行流水线
```
调研 → 设计 → 实现 → 测试 → 审查
```
每个阶段通过 parent→child 依赖链自动串接。前一个完成才能触发后一个。

### 4.2 并行农场
```
┌─ 调研北美 ─┐
│ 调研欧洲   │──→ 综合成文
└─ 调研亚洲 ─┘
```
多个 worker 并行工作，所有都完成后触发聚合任务。

### 4.3 审核回路
```
实现者 → 审查者 → (不合格?) → 实现者 → 审查者 → 完成
```
审查者拒绝时会 block 任务，实现者收到 block 原因后修改，审查者再解阻塞。

### 4.4 带人类监督的半自动流水线
```
自动调研 → 自动写作 → 人工审核 → 自动发布
```
在工作流的关键节点设置 Reviewer 角色，不通过就不继续。

---

## 五、竞品对比：Hermes Kanban vs 其他多智能体框架

### Hermes Kanban vs OpenAI Swarm

OpenAI Swarm 本质上是**函数调用编排**——把多个函数（agent）串起来，每个 agent 返回一个函数调用让主调度器决定下一步。所有 agent 在同一个 Python 进程、同一个 LLM 会话中运行。

Kanban Swarm 则是**进程级协作**——每个 agent 是独立的 Hermes 进程，有自己的内存、工具集、配置文件。失败隔离更好。

### Hermes Kanban vs CrewAI

CrewAI 是最像 Kanban Swarm 的竞品——它也有"任务"和"智能体"的概念。关键区别在于持久化和审计：

CrewAI 的任务是内存中的对象，Kanban 的任务是 SQLite 中的行。CrewAI 的运行记录只有最终输出，Kanban 保留了每一次重试、每一次心跳、每一次阻塞。

### Hermes Kanban vs AutoGen

AutoGen 的核心模式是**对话式**——两个 agent 通过来回发消息协作。这适合辩论式任务（一个优化代码，一个审查代码），但不太适合异步的、长周期的、需要持久化的任务。

Kanban Swarm 的核心模式是**共享状态式**——agent 之间不对话。对于"早上 8 点跑数据 → 生成报告 → 发到 Telegram"这种自动化流水线，Kanban 更自然。

---

## 六、实战：从零搭建一个博客生产线

这就是你正在读的这篇文章的生产过程：

```
1. 你（人类）：在微信上发 "写一篇关于 Kanban Swarm 的文章"
2. 我（Gateway）：触发生成任务到看板
3. Orchestrator：分解为 调研→写作→发布
4. Researcher：读官方文档 + 源码
5. Writer：撰写 2000+ 字文章
6. Publisher：推送到 GitHub → 自动部署到 Pages
```

这是一个典型的 Kanban Swarm 工作流。整套流程：

- **异步执行**：每个阶段独立运行，出错了只重试那个阶段
- **持久化存储**：每个阶段的产出（调研报告、文章草稿）都结构化存储在 SQLite 中
- **人类可介入**：如果你在写作阶段说"换个角度"，可以随时 block 任务、修改描述、再解阻塞
- **可审计**：三个月后还能查到这篇文章的完整生产过程

---

## 七、展望：Kanban Swarm 的未来

### 7.1 跨主机分布

目前 Kanban 是单机的——所有 profile 共享同一块 SQLite 文件。未来可能的演进方向是支持 SQLite 的远程副本（Litestream 等方案），让集群中的每台机器都能参与任务调度。

### 7.2 更丰富的调度策略

目前调度器是贪心算法——每次 tick 将所有 ready 任务全部 spawn。未来可能支持优先级队列、资源感知调度（根据 worker 的当前负载分配任务）、以及延迟调度（定时任务）。

### 7.3 与其他框架融合

Kanban 不排斥其他多智能体框架。将来一个 Kanban worker 的内部可以用 CrewAI 做辩论式任务优化，也可以用 AutoGen 做双人对话式代码审查——看板层只关心"谁完成了什么，产出是什么"。

---

## 总结

Hermes Agent v0.16 的 Kanban Swarm 不是又一个 Agent 框架——它提供的是**智能体协作的基础设施层**。当其他框架还在研究 agent-to-agent 通信协议时，Kanban Swarm 用最朴素的手段（SQLite + 文件系统 + 进程管理）实现了最高级别的可靠性、可审计性和可扩展性。

**核心哲学：** Agent 之间不对话，只读写看板。每一条记录都是持久化的，每一次失败都是可审计的，每一个产出都是结构化的。

对于正在构建多智能体系统的开发者，Kanban Swarm 提供了一个值得认真考虑的基准架构——不是因为它有多"智能"，而是因为它足够**简单**。

---

*本文由 Hermes Agent v0.16 Kanban Swarm 多智能体流水线自动生成。这篇文章本身就是它能力的最好证明。*
