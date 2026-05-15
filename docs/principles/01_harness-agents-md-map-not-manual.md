# Harness 工程原则 — AGENTS.md 是地图，而非手册

> AGENTS.md 设计原则：**保持精简：地图，而非手册**，具体规范和知识分散在各文档中，AGENTS.md 仅负责引导 Agent 到正确的位置。

---

## 一、引言：一个反直觉的事实

你写了一份 3000 行的 AGENTS.md，把项目所有的编码规范、架构决策、API 约定、部署流程、Git 规范、测试策略……全部塞进去。你满怀期待地启动 Agent，却发现它：

- 漏读了后半段的关键规则
- 把两条不同上下文的规范混为一谈
- 在一个简单任务上反复"回忆"大量无关指令，拖慢推理

问题不在 Agent 的能力——问题在于你给它的是**手册**，而不是**地图**。

这是 Harness 工程原则中最精炼也最深刻的一条：**AGENTS.md 是地图，而非手册**。六个字，道出了 Coding Agent 与 LLM 交互的核心设计哲学。

---

## 二、理论篇：为什么"手册"模式会失效

### 2.1 LLM 的注意力不是均匀分布的

很多人误以为：只要把信息放进 system 消息，LLM 就会"同等重视"每一条。实际上，LLM 的注意力机制存在**稀释效应**：

```
信息量少、指令明确 → 注意力集中 → 遵循率高
信息量大、指令模糊 → 注意力分散 → 遵循率下降
```

这和人类阅读同理——一本 10 页的文档你能记住 80%，一本 500 页的手册你只能记住目录。

### 2.2 System 消息的 Token 经济学

Claude Code 的每次 API 调用都会携带完整的 system 消息。来看实际的数据流：

```
┌──────────────────────────────────────────────────────────┐
│ 每次 API 请求                                             │
│                                                          │
│ system: [运行时系统指令 + CLAUDE.md/AGENTS.md + 工具定义]  │
│ messages: [对话历史 + 工具调用结果]                        │
└──────────────────────────────────────────────────────────┘
```

**每一次 ReAct 循环**（LLM 推理 → 工具调用 → 结果回注 → 继续推理），system 消息都会被完整发送。如果你的 AGENTS.md 有 3000 行：

| 场景 | system 消息 token | 单轮对话 API 调用次数 | 总 token 浪费 |
|------|-------------------|---------------------|--------------|
| 精简 AGENTS.md (50行) | ~2K | 3次 | ~6K |
| 臃肿 AGENTS.md (3000行) | ~15K | 3次 | ~45K |

**臃肿的 AGENTS.md 不仅降低遵循率，还直接烧钱。** 每次调用都重复传输那些当前任务根本用不到的规范。

### 2.3 ReAct 循环的天然优势：按需加载

Coding Agent 的核心架构是 ReAct（Reason + Act）循环：

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

这个循环意味着 Agent **天然支持按需加载**——它可以在需要时用 Read 工具读取任何文件。你不需要在 AGENTS.md 里预加载所有知识，只需要告诉它"知识在哪里"。

### 2.4 手册模式 vs 地图模式：一个对比

| 维度 | 手册模式 | 地图模式 |
|------|---------|---------|
| 信息存放 | 全部堆在 AGENTS.md | 分散在各专业文档中 |
| AGENTS.md 体积 | 几千行 | 几十到几百行 |
| 注意力分布 | 稀释，关键指令被淹没 | 聚焦，每条指令都有高权重 |
| Token 成本 | 每次调用重复传输大量无用信息 | system 消息精简，按需读取 |
| 维护成本 | 一处改动影响全局，难以 review | 各文档独立维护，职责清晰 |
| Agent 行为 | "知道很多但做不好" | "知道去哪找，做得很准" |

---

## 三、实践篇：如何把 AGENTS.md 写成地图

### 3.1 核心公式

```
AGENTS.md = 索引 + 路由规则 + 极少量全局约束
```

- **索引**：告诉 Agent "每个领域的知识在哪个文件"
- **路由规则**：告诉 Agent "遇到什么场景应该去读什么"
- **全局约束**：仅放真正全局适用的、简短的硬规则（3-5条）

### 3.2 反面示例：手册式 AGENTS.md

```markdown
# 项目规范

## 编码规范
- 所有 Java 类必须有 Javadoc
- 变量命名使用 camelCase
- 常量使用 UPPER_SNAKE_CASE
- 方法不超过 30 行
- 类不超过 500 行
- 使用 Lombok 的 @Data 注解
- 不允许使用 System.out.println，必须用 SLF4J
- 异常必须包含错误码
- DTO 必须实现 Serializable
- ...（还有 50 条）

## API 规范
- RESTful 风格
- URL 使用 kebab-case
- 响应格式统一为 { code, message, data }
- 分页参数必须使用 page 和 size
- 认证使用 Bearer Token
- ...（还有 30 条）

## 数据库规范
- 表名使用 snake_case
- 必须有 id, created_at, updated_at 字段
- 索引命名 idx_表名_字段名
- 禁止使用外键约束
- ...（还有 20 条）

## 部署规范
- 使用 Docker 部署
- 镜像基于 eclipse-temurin:17-jre
- 健康检查端口 8080/actuator/health
- ...（还有 15 条）
```

**问题**：115+ 条规则塞在一个文件里。Agent 处理"给订单加个字段"这种小任务时，99% 的规则都是噪音。

### 3.3 正面示例：地图式 AGENTS.md

```markdown
# 项目导航

## 阅读原则

**必须阅读**（按场景选择）：
- `docs/architecture.md`：全局架构、技术栈、模块分层
- `docs/api-conventions.md`：API 设计规范与响应格式
- `docs/database-conventions.md`：数据库命名与迁移规范
- `docs/deployment.md`：部署流程与 Docker 配置

## 路由规则

- 改后端代码 → 先读 `docs/architecture.md` 了解分层，再按模块读对应规范
- 改 API 接口 → 必读 `docs/api-conventions.md`
- 改数据库 → 必读 `docs/database-conventions.md`
- 部署相关 → 必读 `docs/deployment.md`

## 全局硬规则

1. 提交信息遵循 AngularJS Git 规范
2. 不引入新的安全漏洞（XSS、SQL 注入、命令注入）
3. 改动超过 3 个文件时先写计划再动手
```

**效果**：
- AGENTS.md 只有 ~20 行，system 消息极度精简
- 每条路由规则都有高注意力权重，Agent 遵循率接近 100%
- Agent 按需读取专业文档，当前任务不需要的规范零 token 消耗
- 各专业文档可以独立演进，改数据库规范不影响 API 规范

### 3.4 第一次对话：地图如何驱动 Agent 行为

用户发送：`"帮我看看这个项目的架构"`

**Agent 的内部推理**（基于 ReAct 循环）：

```
Step 1: 读取 AGENTS.md → 发现路由规则："改后端代码 → 先读 docs/architecture.md"
        同时发现"必须阅读"列表中包含 architecture.md

Step 2: 调用 Read 工具读取 docs/architecture.md

Step 3: 基于 architecture.md 的内容，生成回复
```

对应到 API 调用层面：

```
API Call #1
  system: [..., AGENTS.md地图内容, ...]
  messages:
    [user] "帮我看看这个项目的架构"

  → LLM 返回: tool_use(Read, docs/architecture.md)

工具执行: Read → 获取文件内容

API Call #2
  system: [同上]
  messages:
    [user] "帮我看看这个项目的架构"
    [assistant] tool_use(Read, docs/architecture.md)
    [user] tool_result(architecture.md 的完整内容)

  → LLM 返回: 基于文档的架构概述
```

**关键洞察**：AGENTS.md 里没有一行架构知识，但 Agent 准确地找到了所有架构信息。这就是"地图"的力量——它不存储知识，它路由知识。

### 3.5 第二次对话：路由规则的分流价值

用户发送：`"给订单表加个 status 字段"`

**Agent 的内部推理**：

```
Step 1: 读取 AGENTS.md → 匹配路由规则："改数据库 → 必读 docs/database-conventions.md"

Step 2: 调用 Read 工具读取 docs/database-conventions.md
        （注意：它不会去读 api-conventions.md，因为当前任务不需要）

Step 3: 基于数据库规范，生成符合约定的迁移脚本
```

**对比手册模式**：如果所有规范都堆在 AGENTS.md 里，Agent 会被 115+ 条规则淹没，可能把 API 响应格式的规则错误地应用到数据库迁移脚本上。地图模式通过路由规则，让 Agent 只加载当前任务需要的知识上下文。

---

## 四、原理解析篇：地图模式背后的深层机制

### 4.1 信息检索的 Two-Phase 模式

地图模式实际上把 Agent 的信息获取分成了两个阶段：

```
Phase 1: 索引查找（轻量）
  - 输入：AGENTS.md（~20行，始终在 system 消息中）
  - 输出：目标文档路径
  - 成本：极低，每次 API 调用都执行
  - 类比：查地图，找到目标建筑的坐标

Phase 2: 内容获取（按需）
  - 输入：Read 工具读取目标文档
  - 输出：具体规范内容
  - 成本：仅在实际需要时产生
  - 类比：到达建筑，阅读门牌和说明
```

**手册模式只有 Phase 2，没有 Phase 1**——所有内容都预加载了，但 Agent 无法区分"当前任务需要哪些"，导致注意力稀释。

### 4.2 注意力密度的数学直觉

假设 LLM 对 system 消息的"遵循力"是一个有限资源 $A$（总量固定），而 system 消息中有 $N$ 条有效指令。那么每条指令获得的平均注意力为：

$$\text{注意力密度} = \frac{A}{N}$$

| 模式 | N (有效指令数) | 注意力密度 | 遵循效果 |
|------|--------------|-----------|---------|
| 地图 (20行) | ~5 条核心指令 | 高 | 每条指令都被准确遵循 |
| 手册 (3000行) | ~100+ 条指令 | 低 | 关键指令被淹没 |

这不是说 LLM 无法处理长上下文——而是说，**在决定行为的关键指令上，密度越高，遵循率越好**。

### 4.3 与 Prompt Cache 的协同

Claude API 的 Prompt Cache 机制为地图模式提供了额外的性能加成：

```
┌─────────────────────────────────────────────────────────┐
│ Prompt Cache 工作原理                                    │
│                                                         │
│ system 消息被分为：                                      │
│   [固定前缀 | 可变后缀]                                   │
│                                                         │
│ 固定前缀：运行时指令 + 工具定义 + AGENTS.md               │
│   → 缓存命中时直接复用，不重新计算                        │
│   → TTL 5 分钟                                          │
│                                                         │
│ 可变后缀：对话历史                                        │
│   → 每次新增内容需要重新计算                              │
└─────────────────────────────────────────────────────────┘
```

**精简的 AGENTS.md → 固定前缀更短 → 缓存命中率更高 → 更省 token 更快**。

臃肿的 AGENTS.md 会让固定前缀变大，虽然也能缓存，但：
1. 首次计算成本高
2. 缓存失效后重新计算的成本也高
3. 缓存中大量内容是当前任务用不到的

### 4.4 地图模式与 ReAct 循环的天然契合

ReAct 循环的本质是：**先思考需要什么信息，再去获取，再基于信息行动**。地图模式完美匹配这个节奏：

```
┌───────────────────────────────────────────────────────┐
│ ReAct + 地图模式                                      │
│                                                       │
│ Reason: "用户要改数据库，AGENTS.md 说去读               │
│         docs/database-conventions.md"                 │
│                    ↓                                  │
│ Act:    Read("docs/database-conventions.md")          │
│                    ↓                                  │
│ Observe: 获取到完整的数据库规范                          │
│                    ↓                                  │
│ Reason: "根据规范，字段命名用 snake_case，               │
│         必须有 updated_at，迁移脚本放在 migrations/"     │
│                    ↓                                  │
│ Act:    编写符合规范的迁移脚本                           │
└───────────────────────────────────────────────────────┘
```

**手册模式破坏了这个节奏**——Agent 在 Reason 阶段就被大量无关信息干扰，无法精准定位"我需要什么"。

### 4.5 路由规则 vs 单纯索引

地图模式不只是"列一个文件目录"——它包含**路由规则**，这是关键区别：

**单纯索引**（不够好）：
```markdown
## 文档索引
- docs/architecture.md
- docs/api-conventions.md
- docs/database-conventions.md
```

**路由规则**（正确姿势）：
```markdown
## 路由规则
- 改后端代码 → 先读 docs/architecture.md
- 改 API 接口 → 必读 docs/api-conventions.md
- 改数据库 → 必读 docs/database-conventions.md
```

区别在于：
- 索引只告诉 Agent "有这些文件"，Agent 需要自己判断读哪个（增加了推理负担和出错概率）
- 路由规则直接告诉 Agent "什么场景读什么"，消除了判断环节，遵循率更高

**这就像真实地图的区别**：一张只标注了城市名的地图 vs 一张标注了"你要去机场就走这条高速"的地图——后者才是真正的导航。

---

## 五、进阶：地图模式的工程实践

### 5.1 分层地图

大型项目可以采用分层地图：

```markdown
# AGENTS.md（一级地图）

## 入口路由
- 所有任务 → 先读 `docs/README.md` 了解项目概况
- 后端任务 → 读 `docs/backend/MAP.md`
- 前端任务 → 读 `docs/frontend/MAP.md`
- DevOps 任务 → 读 `docs/devops/MAP.md`

## 全局硬规则
1. Git 提交遵循 AngularJS 规范
2. 不引入安全漏洞
```

```markdown
# docs/backend/MAP.md（二级地图）

## 路由规则
- 新增 API → 读 `docs/backend/api-conventions.md` + `docs/backend/error-codes.md`
- 改数据库 → 读 `docs/backend/database-conventions.md` + `docs/backend/migration-guide.md`
- 加定时任务 → 读 `docs/backend/scheduler-guide.md`

## 领域硬规则
1. Service 层禁止直接调用其他服务的 Repository
2. 所有外部调用必须走 Feign Client
```

Agent 的 ReAct 循环会自动逐层展开：

```
Read AGENTS.md → 发现 "后端任务 → 读 backend/MAP.md"
  → Read backend/MAP.md → 发现 "改数据库 → 读 database-conventions.md"
    → Read database-conventions.md → 获取完整规范
```

每一层都很精简，但 Agent 能精准定位到任何角落的知识。

### 5.2 动态路由：基于条件的地图

```markdown
## 路由规则

- 改 Java 代码 → 读 `docs/java-conventions.md`
- 改 TypeScript 代码 → 读 `docs/ts-conventions.md`
- 涉及支付流程 → **额外**读 `docs/payment-security.md`（合规要求）
- 涉及用户数据 → **额外**读 `docs/gdpr-compliance.md`（法律要求）
```

条件路由让 Agent 在不同场景下加载不同的知识组合，避免"一份手册打天下"的粗暴方式。

### 5.3 地图的维护原则

| 原则 | 说明 |
|------|------|
| **AGENTS.md 只写路由，不写规范** | 规范内容属于专业文档，不属于地图 |
| **路由规则要具体到场景** | "改数据库 → 读 X" 比 "X 文件包含数据库规范" 更有效 |
| **全局硬规则不超过 5 条** | 超过 5 条就不再是"硬规则"，考虑拆分到专业文档 |
| **专业文档自成体系** | 每个文档独立完整，不依赖 AGENTS.md 才能理解 |
| **定期清理死链** | 文件被删除或移动后，及时更新地图 |

---

## 六、总结：地图思维的本质

**地图思维的本质是：承认 Agent 的能力边界，优化信息的投递方式。**

Agent 不缺能力——它有 ReAct 循环、有工具调用、有长上下文窗口。它缺的是**精准的信息投递**：在正确的时刻，把正确的信息，放到注意力的焦点上。

手册模式假设"信息越多越好"，但 LLM 的注意力是有限的。地图模式理解"信息越精准越好"，通过路由机制实现按需加载。

```
手册思维：我把所有知识都告诉你，你自己找
地图思维：我告诉你去哪找，你到了再细看

手册思维：Agent 是读者，要在 500 页手册里找答案
地图思维：Agent 是旅行者，地图指引方向，到了再看细节
```

这不是一个微小的写作风格差异——这是两种完全不同的 Agent 工程范式。在 Agent 越来越强的今天，**如何为 Agent 设计信息架构**，和**如何为 Agent 设计工具**同等重要。

AGENTS.md 是地图，而非手册。六个字，值得每一个 Agent 工程师刻在键盘上。

---

## 附录：从 API 调用看地图 vs 手册的 token 差异

以下是一次完整对话的实际 token 流对比（用户请求："给订单表加 status 字段"）：

### 地图模式

```
API Call #1
  system: [...运行时指令... + AGENTS.md(20行地图) + 工具定义]
  messages: [user] "给订单表加 status 字段"
  → system token: ~3K

  → LLM 返回: Read("docs/database-conventions.md")

API Call #2
  system: [同上, ~3K, 缓存命中]
  messages: [历史 + tool_result(database-conventions.md 内容)]
  → 新增 token: ~1.5K (tool_result)

  → LLM 返回: 符合规范的迁移脚本

总 token: ~4.5K (system 缓存后仅计算一次)
```

### 手册模式

```
API Call #1
  system: [...运行时指令... + AGENTS.md(3000行手册) + 工具定义]
  messages: [user] "给订单表加 status 字段"
  → system token: ~15K

  → LLM 返回: 迁移脚本 (但可能混入 API 规范的无关约束)

总 token: ~15K+ (且遵循率更低)
```

**地图模式节省 70% 的 token，同时提高遵循率。** 这不是理论推测——这是 ReAct 架构 + 注意力机制 + Prompt Cache 三者协同的必然结果。

> 参考[Claude Code 执行原理模拟Log](./references/coding-agent-call-log.md)