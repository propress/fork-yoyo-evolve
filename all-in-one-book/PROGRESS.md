# yoyo 实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 0 | 序言：一只小章鱼的成长实验 | ch00-preface.md | 项目定位 / 架构全景图 / 核心概念词典 / 代码库地图 / 极简全流程 | ✅ |
| 1 | 数据流全景：一次完整交互的旅程 | ch01-data-flow.md | 用户输入 → CLI解析 → Agent构建 → LLM调用 → 工具执行 → 响应渲染，每步数据形态变化 | ✅ |
| 2 | 核心引擎：Agent 构建与工具体系 | ch02-agent-engine.md | AgentConfig / yoagent集成 / 工具包装三层架构(Guard→Confirm→Truncate) / 自定义工具(Bash/AskUser/Todo) | ✅ |
| 3 | REPL 交互循环 | ch03-repl.md | rustyline集成 / Tab补全 / 多行输入 / 命令路由 / 会话状态管理 | ✅ |
| 4 | 提示执行与容错 | ch04-prompt-execution.md | run_prompt调用链 / 自动重试 / 上下文溢出恢复 / 审计日志 / 文件变更追踪与撤销 | ✅ |
| 5 | 命令系统全解 | ch05-commands.md | 81条命令的分类架构 / Git集成 / 文件操作 / 项目分析 / 搜索体系 / 会话管理 | ⏳ |
| 6 | 上下文窗口管理 | ch06-context-management.md | 双阈值压缩策略 / Checkpoint模式 / yoagent CompactionStrategy / 子代理上下文隔离 | ⏳ |
| 7 | 自我进化流水线 | ch07-evolution-pipeline.md | evolve.sh全流程 / 三阶段(评估→规划→实施) / 赞助者系统 / Issue处理 / 学习记录 | ✅ |
| 8 | 记忆与学习系统 | ch08-memory-system.md | 双层记忆架构 / JSONL存档 / 活跃上下文合成 / 社交学习 / 项目记忆(.yoyo/) | ✅ |
| 9 | 项目演进史：从200行到3万行 | ch09-evolution-history.md | Day 1-4原型期 / Day 5-12基础设施期 / Day 13-20成熟期 / Day 21-28精炼期 / 架构演进图 | ✅ |
| 10 | 端到端追踪 | ch10-end-to-end.md | 场景一：用户在REPL提问 / 场景二：自动进化一次 / 场景三：社区Issue处理，串联全书 | ⏳ |

## 章节规划说明

认知路径：先懂什么 → 才能懂什么

1. **序言**（第0章）：建立全局画面。读者需要先知道 yoyo 是什么、整体长什么样、有哪些核心概念，才能进入任何细节。
2. **数据流全景**（第1章）：用一次真实交互串起整个系统，让读者有"地图感"。复杂节点标注"详见第N章"。
3. **核心引擎**（第2章）：理解了全流程后，深入最核心的模块——Agent 是怎么构建的、工具系统如何运转。这是后续所有章节的基础。
4. **REPL 交互循环**（第3章）：有了 Agent 的理解，才能看懂 REPL 如何驱动 Agent。从用户按下回车到命令被路由。
5. **提示执行与容错**（第4章）：REPL 把自然语言输入交给 prompt 系统后发生了什么。重试、压缩、审计、撤销。
6. **命令系统**（第5章）：REPL 把斜杠命令交给命令系统后发生了什么。按功能分组深入。
7. **上下文窗口管理**（第6章）：跨越 prompt 和 session 的上下文管理策略，需要前面的基础才能理解。
8. **自我进化流水线**（第7章）：跳出 CLI 本身，看它是怎么进化自己的。这是 yoyo 最独特的部分。
9. **记忆与学习系统**（第8章）：进化流水线的"大脑"，理解了进化流程才能理解记忆为什么这样设计。
10. **项目演进史**（第9章）：从 commit 历史还原 yoyo 的成长过程。需要前面对各模块的理解作为背景。
11. **端到端追踪**（第10章）：收尾，用完整场景串联全书，验收读者的理解。

## 状态说明
- ✅ 已完成
- 🔄 进行中
- ⏳ 待开始

## 术语约定

### 行业标准术语
- **LLM**（Large Language Model）：大语言模型
- **REPL**（Read-Eval-Print Loop）：交互式命令行循环
- **Token**：LLM 处理文本的最小单位
- **Context Window**：LLM 单次对话能"看到"的最大 token 数
- **Tool Use / Function Calling**：LLM 调用外部工具的能力
- **MCP**（Model Context Protocol）：模型上下文协议，标准化 LLM 与外部工具的交互
- **CLI**（Command-Line Interface）：命令行界面

### 项目特有术语
- **yoagent**：yoyo 底层依赖的 Agent 框架库（类比：Flask 之于 Web 应用）
- **Evolution Session**：一次自我进化会话（类比：CI/CD Pipeline 的一次运行）
- **Skill**：以 Markdown 形式定义的能力模块（类比：系统提示词模板）
- **GuardedTool / ConfirmTool / TruncatingTool**：工具包装层（类比：中间件 Middleware）
- **Session Changes**：文件变更追踪器（类比：Git 暂存区的运行时版本）
- **SpawnTracker**：子代理任务管理器

## 下次续写指引
### 从哪里继续
从第0章（序言）开始写作。

### 交接备忘
- 已完成深度阅读：17个Rust源文件、所有脚本、技能文件、身份文件、日志、commit历史(796条)
- 项目总规模：~35K行Rust代码，17个源文件，1346个测试
- 关键认知：yoyo = CLI Agent + 自我进化流水线 + 社区交互 + 记忆系统
- 架构核心：yoagent 提供 Agent/Tool/Event 基础，yoyo 在其上构建 REPL、命令、进化

### 待验证项
- evolve.sh 中 Phase A 和 Phase B 的具体超时参数
- yoagent 的 CompactionStrategy 三级压缩具体实现
- SubAgentTool 的上下文隔离机制细节
