# Harness Engineering

Claude Code Agent 工程实践中心，探索如何高效地为 Coding Agent 设计信息架构与工作流。

## 核心理念

**AGENTS.md 是地图，而非手册。**

把知识存储在专用文档中，用 AGENTS.md 作为路由索引，让 Agent 在需要时按需读取，避免注意力稀释。

## 目录结构

```
harness-engineering/
├── AGENTS.md                    # Agent 导航地图（精简路由，非手册）
├── docs/
│   └── principles/              # 工程原则
│       └── 01_harness-agents-md-map-not-manual.md
├── examples/
│   └── harness-repo/            # 示例项目（展示地图模式实践）
│       ├── AGENTS.md
│       └── .harness/             # 知识库
└── LICENSE
```

## 核心原则

1. **Think Before Coding** — 不臆测，不隐藏困惑
2. **Simplicity First** — 最少量代码解决问题
3. **Surgical Changes** — 精准修改，不做预防性改动
4. **Goal-Driven Execution** — 定义成功标准，循环验证

## 提交规范

遵循 [AngularJS Git Commit 规范](https://gist.github.com/stephenparish/9941e89d80e2bc58a15e)

## 相关资源

- [AGENTS.md 设计原则详解](./docs/principles/01_harness-agents-md-map-not-manual.md)
