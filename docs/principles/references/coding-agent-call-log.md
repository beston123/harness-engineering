# Claude Code 执行原理模拟Log

## 前置知识：Coding Agent 的核心架构

Claude Code 本质上是一个 **ReAct 循环**（Reason + Act），围绕 Claude API 构建：

```
┌─────────────────────────────────────────────────┐
│                  Claude Code REPL               │
│                                                 │
│  1. 构造 Prompt（系统指令 + CLAUDE.md + 工具定义 + 对话历史）  │
│  2. 调用 Claude API                             │
│  3. 解析响应                                     │
│     ├─ 纯文本 → 返回给用户                        │
│     └─ 包含 tool_use → 执行工具 → 将结果喂回 → 回到步骤2  │
│  4. 上下文管理（压缩、记忆）                       │
└─────────────────────────────────────────────────┘
```

**关键组件**：
- **Prompt 构造层**：拼接系统指令、CLAUDE.md 内容、工具定义、对话历史
- **ReAct 循环**：LLM 推理 → 工具调用 → 结果回注 → 继续推理
- **工具执行层**：Bash、Read、Write、Edit、Agent 等
- **上下文管理**：对话压缩（超长时自动摘要）、记忆系统（memory files）

---

## 场景设定

假设 CLAUDE.md 内容：
```
## 阅读原则

**必须阅读**：
- `docs/project/architecture.md`: 全局架构，包括技术栈，模块/目录分层
```

用户第一次发送：`"帮我看看这个项目的架构"`

---

## 第一次对话：完整执行过程

### 阶段 1：Prompt 构造

Claude Code 运行时将以下内容组装成 API 请求的 messages 数组：

#### Message 1 — system（系统消息）

这是所有对话的"底座"，每次 API 调用都会携带：

```
角色: system（或作为第一条 user 消息的隐藏前缀）

内容包含:
1. 身份声明："You are Claude Code, Anthropic's official CLI for Claude."
2. 系统行为指令：
   - 工具使用规范（优先用 Read/Edit/Write 而非 cat/sed）
   - Git 操作安全协议
   - 任务跟踪（TaskCreate/TaskUpdate）
   - 代码风格（不加多余注释、不加 emoji 等）
   - 安全要求（不引入 XSS、SQL 注入等漏洞）
3. 工具定义（JSON Schema）：
   - Read: { file_path, offset?, limit? }
   - Edit: { file_path, old_string, new_string }
   - Write: { file_path, content }
   - Bash: { command, description }
   - Agent: { description, prompt, subagent_type? }
   - TaskCreate / TaskUpdate / TaskList / TaskGet
   - WebFetch / WebSearch
   - ... 所有可用工具
4. CLAUDE.md 内容注入：
   → 从 ~/.claude/CLAUDE.md（全局）和项目目录 ./CLAUDE.md（项目级）读取
   → 合并到系统消息中
   → 本场景中包含 "必须阅读 docs/project/architecture.md" 的指令
5. 环境信息：
   - 工作目录: /Users/beston/projects/my-project
   - 平台: darwin
   - Shell: zsh
   - 当前日期: 2026/05/14
6. 记忆系统内容（如果存在）：
   → 读取 ~/.claude/projects/.../memory/MEMORY.md 及引用的记忆文件
```

**实际发送给大模型时的格式**（简化示意）：

```json
{
  "system": "You are Claude Code, Anthropic's official CLI for Claude.\n\nYou are an interactive agent that helps users with software engineering tasks...\n\n## Behavioral Guidelines\n\n## 阅读原则\n\n**必须阅读**：\n- `docs/project/architecture.md`: 全局架构，包括技术栈，模块/目录分层\n\n...(完整系统指令 + 工具定义)...",
  "messages": [...]
}
```

#### Message 2 — user（用户第一条消息）

```json
{
  "role": "user",
  "content": "帮我看看这个项目的架构"
}
```

### 阶段 2：第一次 API 调用

**发送给 Claude API 的完整请求**：

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 16384,
  "system": "<上面完整的系统消息，包含 CLAUDE.md 的阅读原则>",
  "messages": [
    {
      "role": "user",
      "content": "帮我看看这个项目的架构"
    }
  ],
  "tools": [
    { "name": "Read", "description": "Reads a file from the local filesystem...", "input_schema": {...} },
    { "name": "Bash", "description": "Executes a given bash command...", "input_schema": {...} },
    { "name": "Edit", ... },
    { "name": "Write", ... },
    { "name": "Agent", ... },
    ...所有工具定义
  ]
}
```

**LLM 的推理过程**（内部思维，不可见）：

> 用户要看项目架构。CLAUDE.md 明确要求必须阅读 `docs/project/architecture.md`。
> 我应该先用 Read 工具读取这个文件。

### 阶段 3：LLM 返回 tool_use（第一次工具调用）

**LLM 的响应**：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "让我先读取项目的架构文档。"
    },
    {
      "type": "tool_use",
      "id": "toolu_01ABC123",
      "name": "Read",
      "input": {
        "file_path": "/Users/beston/projects/my-project/docs/project/architecture.md"
      }
    }
  ]
}
```

### 阶段 4：工具执行 + 结果回注

Claude Code 运行时执行 Read 工具，得到文件内容，然后构造 tool_result 消息：

**第二次 API 调用的 messages**：

```json
{
  "system": "<同上，完整系统消息>",
  "messages": [
    {
      "role": "user",
      "content": "帮我看看这个项目的架构"
    },
    {
      "role": "assistant",
      "content": [
        { "type": "text", "text": "让我先读取项目的架构文档。" },
        { "type": "tool_use", "id": "toolu_01ABC123", "name": "Read", "input": { "file_path": "/Users/beston/projects/my-project/docs/project/architecture.md" } }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01ABC123",
          "content": "     1\t# 项目架构\n     2\t\n     3\t## 技术栈\n     4\t- 后端: Java 17 + Spring Boot 3.2\n     5\t- 前端: React 18 + TypeScript\n     6\t- 数据库: MySQL 8.0 + Redis 7\n     7\t\n     8\t## 模块结构\n     9\t```\n    10\t├── api-gateway/        # API 网关\n    11\t├── user-service/       # 用户服务\n    12\t├── order-service/      # 订单服务\n    13\t├── payment-service/    # 支付服务\n    14\t└── shared-libs/        # 共享库\n    15\t```\n    16\t\n    17\t## 分层架构\n    18\t每个服务遵循: Controller → Service → Repository → Entity\n    19\t..."
        }
      ]
    }
  ]
}
```

### 阶段 5：LLM 继续推理并生成最终回复

LLM 收到 architecture.md 的内容后，生成给用户的回复：

```json
{
  "role": "assistant",
  "content": "根据架构文档，这个项目的情况如下：\n\n**技术栈**：Java 17 + Spring Boot 3.2 后端，React 18 + TypeScript 前端，MySQL 8.0 + Redis 7 存储。\n\n**模块结构**：微服务架构，包含 API 网关、用户服务、订单服务、支付服务和共享库。\n\n**分层**：每个服务遵循 Controller → Service → Repository → Entity 的标准分层。\n\n你想深入了解哪个模块？"
}
```

**此时对话循环结束，文本回复展示给用户。**

### 第一次对话完整消息流总结

```
┌──────────────────────────────────────────────────────────────┐
│ API Call #1                                                  │
│                                                              │
│ system: [完整系统指令 + CLAUDE.md + 工具定义]                 │
│ messages:                                                    │
│   [user] "帮我看看这个项目的架构"                             │
│                                                              │
│ → LLM 返回: text + tool_use(Read, architecture.md)           │
├──────────────────────────────────────────────────────────────┤
│ 工具执行: Read → 获取文件内容                                 │
├──────────────────────────────────────────────────────────────┤
│ API Call #2 (同一轮对话内的 ReAct 继续)                      │
│                                                              │
│ system: [同上]                                               │
│ messages:                                                    │
│   [user] "帮我看看这个项目的架构"                             │
│   [assistant] text + tool_use(Read)                          │
│   [user] tool_result(Read 的文件内容)                        │
│                                                              │
│ → LLM 返回: 纯文本回复给用户                                 │
└──────────────────────────────────────────────────────────────┘

关键点：
- 一次用户对话可能触发多次 API 调用（因为有工具调用循环）
- 每次调用都携带完整的 system + 全部历史 messages
- CLAUDE.md 的指令被注入到 system 消息中，LLM 在推理时会"遵守"这些指令
```

---

## 第二次对话：完整执行过程

用户继续发送：`"订单服务用了哪些技术？我想加个消息队列"`

### 阶段 1：Prompt 构造

此时对话历史已包含第一轮的所有消息。Claude Code 构造的 messages 数组更长：

#### Message 列表

```json
{
  "system": "<同上，完整系统消息，包含 CLAUDE.md>",
  "messages": [
    // === 第一轮对话历史 ===
    {
      "role": "user",
      "content": "帮我看看这个项目的架构"
    },
    {
      "role": "assistant",
      "content": [
        { "type": "text", "text": "让我先读取项目的架构文档。" },
        { "type": "tool_use", "id": "toolu_01ABC123", "name": "Read", "input": { "file_path": "/Users/beston/projects/my-project/docs/project/architecture.md" } }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01ABC123",
          "content": "     1\t# 项目架构\n     2\t\n     3\t## 技术栈\n     4\t- 后端: Java 17 + Spring Boot 3.2\n     5\t- 前端: React 18 + TypeScript\n     6\t- 数据库: MySQL 8.0 + Redis 7\n     7\t\n     8\t## 模块结构\n     9\t```\n    10\t├── api-gateway/\n    11\t├── user-service/\n    12\t├── order-service/\n    13\t├── payment-service/\n    14\t└── shared-libs/\n    15\t```\n    16\t\n    17\t## 分层架构\n    18\t每个服务遵循: Controller → Service → Repository → Entity\n    19\t..."
        }
      ]
    },
    {
      "role": "assistant",
      "content": "根据架构文档，这个项目的情况如下：\n\n**技术栈**：Java 17 + Spring Boot 3.2 后端，React 18 + TypeScript 前端，MySQL 8.0 + Redis 7 存储。\n\n**模块结构**：微服务架构，包含 API 网关、用户服务、订单服务、支付服务和共享库。\n\n**分层**：每个服务遵循 Controller → Service → Repository → Entity 的标准分层。\n\n你想深入了解哪个模块？"
    },

    // === 第二轮用户输入 ===
    {
      "role": "user",
      "content": "订单服务用了哪些技术？我想加个消息队列"
    }
  ]
}
```

### 阶段 2：LLM 推理 + 工具调用

**LLM 的推理**（内部思维）：

> 用户问订单服务的技术细节，还想加消息队列。
> architecture.md 只给了宏观信息，我需要深入到 order-service 目录查看具体实现。
> CLAUDE.md 的阅读原则要求先看 architecture.md（已读），现在应该读订单服务的代码。

**LLM 响应**（可能并行调用多个工具）：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "让我查看订单服务的具体实现。"
    },
    {
      "type": "tool_use",
      "id": "toolu_02DEF456",
      "name": "Bash",
      "input": {
        "command": "ls /Users/beston/projects/my-project/order-service/src/main/java/com/example/order/",
        "description": "列出订单服务的源码目录结构"
      }
    },
    {
      "type": "tool_use",
      "id": "toolu_02GHI789",
      "name": "Read",
      "input": {
        "file_path": "/Users/beston/projects/my-project/order-service/pom.xml"
      }
    }
  ]
}
```

> **注意**：两个工具调用可以并行执行（它们之间无依赖），这是 Claude Code 提升效率的关键设计。

### 阶段 3：工具执行 + 结果回注

运行时并行执行两个工具，然后构造两条 tool_result：

```json
{
  "system": "<同上>",
  "messages": [
    // ... 前面所有历史 ...
    { "role": "user", "content": "订单服务用了哪些技术？我想加个消息队列" },
    {
      "role": "assistant",
      "content": [
        { "type": "text", "text": "让我查看订单服务的具体实现。" },
        { "type": "tool_use", "id": "toolu_02DEF456", "name": "Bash", "input": { "command": "ls ..." } },
        { "type": "tool_use", "id": "toolu_02GHI789", "name": "Read", "input": { "file_path": ".../pom.xml" } }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_02DEF456",
          "content": "controller/\nservice/\nrepository/\nentity/\nconfig/\nmessaging/"
        },
        {
          "type": "tool_result",
          "tool_use_id": "toolu_02GHI789",
          "content": "<?xml version=\"1.0\"?>\n<project>\n  <dependencies>\n    <dependency>spring-boot-starter-web</dependency>\n    <dependency>spring-boot-starter-data-jpa</dependency>\n    <dependency>spring-boot-starter-data-redis</dependency>\n    <dependency>spring-kafka</dependency>\n    ...\n  </dependencies>\n</project>"
        }
      ]
    }
  ]
}
```

### 阶段 4：LLM 继续推理，可能再次调用工具

**LLM 推理**：

> pom.xml 已经显示用了 Kafka（spring-kafka），目录里也有 messaging/ 文件夹。
> 用户想加消息队列，但项目已经在用 Kafka 了。让我看看 messaging 目录的实现。

**LLM 再次调用工具**：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "项目已经在用 Kafka，让我看看现有的消息处理代码。"
    },
    {
      "type": "tool_use",
      "id": "toolu_03JKL012",
      "name": "Bash",
      "input": {
        "command": "ls /Users/beston/projects/my-project/order-service/src/main/java/com/example/order/messaging/",
        "description": "列出订单服务消息模块的文件"
      }
    }
  ]
}
```

### 阶段 5：工具结果 + 最终回复

```json
// 工具结果
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_03JKL012",
      "content": "OrderEventProducer.java\nOrderEventConsumer.java\nKafkaTopicConfig.java"
    }
  ]
}

// LLM 最终回复
{
  "role": "assistant",
  "content": "订单服务的技术栈：\n\n- **Web**: Spring Boot Starter Web\n- **数据**: Spring Data JPA + MySQL\n- **缓存**: Spring Data Redis\n- **消息**: Spring Kafka（已有 Kafka 集成）\n\n项目已经在使用 Kafka 作为消息队列，有 `OrderEventProducer` 和 `OrderEventConsumer` 两个类处理消息收发。\n\n你想加的消息队列是补充 Kafka，还是想替换为 RabbitMQ/RocketMQ 等其他方案？或者你只是想新增某个具体的消息场景？"
}
```

### 第二次对话完整消息流总结

```
┌──────────────────────────────────────────────────────────────┐
│ API Call #1                                                  │
│                                                              │
│ system: [完整系统指令 + CLAUDE.md + 工具定义]                 │
│ messages:                                                    │
│   [user] "帮我看看这个项目的架构"                             │
│   [assistant] text + tool_use(Read, architecture.md)         │
│   [user] tool_result(文件内容)                               │
│   [assistant] 纯文本回复                                     │
│   [user] "订单服务用了哪些技术？我想加个消息队列"             │
│                                                              │
│ → LLM 返回: text + tool_use(Bash, ls) + tool_use(Read, pom)  │
├──────────────────────────────────────────────────────────────┤
│ 工具并行执行: Bash(ls) + Read(pom.xml)                       │
├──────────────────────────────────────────────────────────────┤
│ API Call #2                                                  │
│                                                              │
│ system: [同上]                                               │
│ messages:                                                    │
│   [全部历史 + 新的工具调用和结果]                             │
│                                                              │
│ → LLM 返回: text + tool_use(Bash, ls messaging/)             │
├──────────────────────────────────────────────────────────────┤
│ 工具执行: Bash(ls messaging/)                                │
├──────────────────────────────────────────────────────────────┤
│ API Call #3                                                  │
│                                                              │
│ system: [同上]                                               │
│ messages:                                                    │
│   [全部历史 + 最新工具结果]                                   │
│                                                              │
│ → LLM 返回: 纯文本回复给用户                                 │
└──────────────────────────────────────────────────────────────┘

关键点：
- 第二次对话携带了第一次的全部历史（messages 越来越长）
- LLM 能"记住"之前读过的 architecture.md 内容（因为它在消息历史中）
- 工具调用可以并行（一次响应中包含多个 tool_use）
- 一个用户消息可能触发 2-3 次 API 调用（ReAct 循环多次）
```

---

## 核心原理总结

### 1. CLAUDE.md 如何生效

CLAUDE.md **不是魔法**，它只是被注入到 system 消息中的文本：

```
系统消息 = 运行时系统指令 + CLAUDE.md内容 + 工具定义 + 环境信息
```

LLM 之所以"遵守"CLAUDE.md，是因为：
- System 消息权重高，LLM 倾向于遵循
- 指令足够具体（如"必须阅读 X 文件"），LLM 会在推理时引用
- 违背 system 指令的输出在训练中被弱化

### 2. ReAct 循环的本质

```
用户输入 → 构造消息 → API调用 → 解析响应
                                    ↓
                            有 tool_use？
                           /           \
                          是            否
                         ↓              ↓
                    执行工具        返回文本给用户
                         ↓
                    结果回注为 tool_result
                         ↓
                    回到"构造消息"继续
```

每次 API 调用都发送**完整的对话历史**，这就是为什么：
- 对话越长，API 调用越贵（token 随消息累积增长）
- 超过上下文窗口时需要"压缩"（自动摘要早期对话）
- LLM 理论上能"看到"所有历史，但实际注意力分配不均

### 3. 上下文增长与压缩

| 对话轮次 | 大致 token 消耗 |
|---------|---------------|
| 第1轮 (2次API调用) | ~5K-10K |
| 第2轮 (3次API调用) | ~15K-25K（含第1轮历史）|
| 第5轮 | ~50K-100K |
| 触发压缩 | 超过上下文窗口时自动摘要早期消息 |

### 4. 工具调用并行的条件

Claude Code 支持在**一次 assistant 响应**中返回多个 tool_use，运行时会：
- 识别它们之间是否独立
- 独立的工具调用并行执行（如同时 Read 两个文件）
- 有依赖的串行执行

### 5. Prompt Cache 优化

Claude API 支持 **prompt caching**：
- system 消息前缀不变时，可复用缓存
- CLAUDE.md + 工具定义通常不变，适合缓存
- 缓存 TTL 为 5 分钟，减少重复计算的 token 消耗

这就是为什么多次 API 调用的成本不是简单的线性增长——缓存命中时，不变的 system 前缀部分只需计算一次。
