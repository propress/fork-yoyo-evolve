# 第十章：端到端追踪

前面九章分别讲了 yoyo 的各个部件。这一章把它们串起来——用三个完整场景，从第一个字节到最后一个字节，走完每一步。

---

## 10.1 场景一：用户在 REPL 中修复一个 Bug

**情境**：开发者在项目目录下运行 yoyo，输入 "帮我修复 src/parser.rs 中第 42 行的 panic"。

### 完整路径

```
1. 启动（第 1 章）
   CLI: parse_args() → Config { model: "claude-sonnet-4-20250514", provider: "anthropic", ... }
   main.rs: build_agent() → AnthropicProvider → configure_agent()
     — 注册 9 个工具（bash/read/write/edit/list/search/rename/ask/todo）
     — 三层包装：TruncatingTool → GuardedTool → ConfirmTool
     — 上下文窗口：200K token × 80% = 160K 交给 yoagent
   main.rs: → 检测为交互模式 → run_repl()

2. REPL 循环（第 3 章）
   repl.rs: 打印 banner（model、cwd、git branch）
   repl.rs: readline() → "帮我修复 src/parser.rs 中第 42 行的 panic"
   repl.rs: 不以 / 开头 → 自然语言路径
   repl.rs: 保存 last_input，记录 TurnSnapshot

3. Prompt 执行（第 4 章）
   prompt.rs: run_prompt_auto_retry()
     → run_prompt_with_changes()
       → proactive_compact_if_needed()  ← 新会话，0%，跳过
       → run_prompt_once()
         → agent.prompt("帮我修复...")
         → handle_prompt_events():

4. Agent 工具循环（第 2 章）
   yoagent: 发送消息到 Anthropic API
   LLM: "我来看看这个文件" → 请求 read_file("src/parser.rs")
     → TruncatingTool → GuardedTool（允许） → ReadFileTool → 返回文件内容
   LLM: "第 42 行有个 unwrap()，让我修复" → 请求 edit_file(...)
     → TruncatingTool → GuardedTool → ConfirmTool:
       "⚠ Allow: edit: src/parser.rs (1 → 3 lines) ? (y/n/always)"
       用户输入 "y" → EditFileTool 执行
   LLM: "验证修复" → 请求 bash("cargo test")
     → TruncatingTool → StreamingBashTool → 实时输出测试结果
   LLM: "所有测试通过，修复完成。"

5. 渲染和善后（第 1 章）
   format.rs: Markdown 渲染 LLM 响应 → 彩色终端输出
   prompt.rs: auto_compact_if_needed()  ← 用了约 3%，跳过
   prompt.rs: print_usage("↳ 2.1K in / 1.5K out · ~$0.01")
   format.rs: maybe_ring_bell()  ← 耗时 8 秒 > 3 秒，发铃声
   repl.rs: session_total += usage，等待下一次输入
```

### 数据变化追踪

| 阶段 | 数据形态 |
|------|---------|
| 用户输入 | `"帮我修复 src/parser.rs 中第 42 行的 panic"` |
| 发送给 API | `{messages: [{role:"user", content:"帮我修复..."}], tools: [...], model: "claude-sonnet-4-20250514"}` |
| LLM 请求读文件 | `{tool: "read_file", params: {path: "src/parser.rs"}}` |
| 文件内容返回 | `"fn parse(...) {\n  data.unwrap()  // line 42\n..."` |
| LLM 请求编辑 | `{tool: "edit_file", params: {path: "src/parser.rs", old_text: "data.unwrap()", new_text: "data.unwrap_or_default()"}}` |
| 编辑确认 | 用户输入 "y" → GuardedTool 放行 → ConfirmTool 放行 → 文件被修改 |
| SessionChanges 记录 | `[{path: "src/parser.rs", kind: Edit}]` |
| 最终输出 | Markdown 渲染的彩色文本："修复完成，`unwrap()` 改为 `unwrap_or_default()`…" |

---

## 10.2 场景二：一次自动进化

**情境**：GitHub Actions cron 在凌晨 4 点触发 evolve.sh。距上次进化已超过 8 小时。

### 完整路径

```
1. 门控检查（第 7 章 §7.3）
   evolve.sh: 检查 git log → 最近 wrap-up 在 10 小时前 → 通过门控
   evolve.sh: 加载赞助者信息 → 无加速额度需要处理

2. 构建验证（第 7 章 §7.2）
   evolve.sh: cargo build → 成功 ✓
   evolve.sh: 检查 CI → 上次通过 ✓

3. Issue 获取（第 7 章 §7.5）
   format_issues.py: gh issue list → 选取最多 5 个 Issue
     — 赞助者 Issue 优先
     — 按得票数排序
     — 日期种子的随机抽取（公平性）
   → 写入 ISSUES_TODAY.md

4. Phase A1: 评估（第 7 章 §7.4）
   evolve.sh: 启动 yoyo（单次模式, --prompt "评估指令..."）
   yoyo: 读取所有 .rs 文件、JOURNAL.md、git log
   yoyo: 执行 cargo build, cargo test（自测）
   yoyo: 写 session_plan/assessment.md
   → 发现: "commands_project.rs 的 /extract 命令不处理泛型参数"

5. Phase A2: 规划（第 7 章 §7.5）
   evolve.sh: 启动新的 yoyo（带评估结果 + Issues + 学习记忆）
   yoyo: 制定 3 个任务:
     task_01.md: 修复 /extract 泛型参数问题
     task_02.md: 响应 Issue #201 (配置路径问题)
     task_03.md: 更新文档

6. Phase B: 实施循环（第 7 章 §7.6）

   任务 1: 修复 /extract
     evolve.sh: 保存 PRE_TASK_SHA
     evolve.sh: 启动 yoyo（--context-strategy checkpoint, 15 分钟超时）
     yoyo: 读 commands_project.rs → 修改 extract_symbol() → 添加测试
     yoyo: cargo fmt && cargo test → 通过
     evolve.sh: 三重验证:
       门控 1: 受保护文件未变 ✓
       门控 2: cargo build + test ✓
       门控 3: 评估 Agent 审批 → "Verdict: PASS" ✓
     → 保留修改

   任务 2: Issue #201
     evolve.sh: 保存新的 PRE_TASK_SHA
     yoyo: 修改 cli.rs → 添加 ~/.yoyo.toml 支持 → 添加测试
     三重验证 → 全部通过 → 保留

   任务 3: 文档更新
     yoyo: 更新 docs/src/configuration/models.md
     三重验证 → 全部通过 → 保留

7. 构建验证 + 修复（第 7 章 §7.7）
   evolve.sh: cargo build + cargo test + cargo clippy → 全部通过 ✓

8. 日志记录（第 8 章）
   evolve.sh: 启动 Journal Agent
   yoyo: 写 JOURNAL.md 条目:
     "## Day 28 — 04:07 — /extract 泛型修复与配置路径改进
      修复了 /extract 不处理泛型参数的问题，响应了 #201..."
   evolve.sh: 启动反思 Agent
   yoyo: 审视本次会话 → 追加 learnings.jsonl:
     {"type":"lesson", "title":"泛型处理需要回溯到类型定义", ...}

9. Issue 响应（第 7 章 §7.9）
   evolve.sh: 启动 Issue Agent
   yoyo: 检查 Issue #201 → 本次已修复 → 调用 gh issue comment + gh issue close

10. Git Push（第 7 章 §7.2）
    evolve.sh: git push → 代码上线
```

### 安全机制触发顺序

```
进化开始
  → 构建验证（确保在良好状态上开始）
  → 每个任务执行后:
    → 受保护文件检查（不能改自己的规则）
    → 构建 + 测试（不能破坏现有功能）
    → 独立评估 Agent（不能偏离任务）
  → 全部任务后:
    → 再次构建验证（检测任务间交互问题）
    → 最多 3 次修复尝试
    → 修不好 → 全部回滚
进化结束
```

---

## 10.3 场景三：社区 Issue 的完整生命周期

**情境**：用户在 GitHub 上提了一个 Issue："yoyo 在处理非 ASCII 文件名时 panic"。

### 从提交到解决

```
Day 0: 用户提交 Issue #250
  → GitHub Issue 系统记录

Day N (下次进化):
  1. format_issues.py 抓取 Issue #250
     — 计算得票数、分类为 "new"
     — 写入 ISSUES_TODAY.md

  2. Phase A2 规划 Agent 看到 Issue #250
     — 安全检查: 分析 Issue 内容（不可信输入）
     — 判断优先级: bug 类 → 高于 UX 改进
     — 生成 task_01.md: "修复非 ASCII 文件名 panic (Issue #250)"

  3. Phase B 实施
     — yoyo 复现: 创建含中文名的文件 → 确认 panic
     — 定位: commands_file.rs 中 Path::display() 的编码假设
     — 修复: 使用 to_string_lossy() 替代
     — 添加测试: 中文、日文、emoji 文件名
     — 三重验证 → 通过

  4. Issue 响应 Agent
     — 检测到 Issue #250 被本次提交修复
     — gh issue comment: "Fixed in today's session! The panic was caused by..."
     — gh issue close

Day N+1 (用户侧):
  — 用户收到关闭通知
  — 用户 pull 最新代码 → 验证修复
```

### 关键路径中的模块参与

| 步骤 | 参与模块 | 章节 |
|------|---------|------|
| Issue 获取 | format_issues.py | 第 7 章 |
| 任务规划 | evolve.sh Phase A2 | 第 7 章 |
| Bug 复现 | StreamingBashTool | 第 2 章 |
| 代码定位 | SearchTool / ReadFileTool | 第 2 章 |
| 代码修改 | EditFileTool + ConfirmTool | 第 2 章 |
| 测试编写 | WriteFileTool + bash | 第 2 章 |
| 三重验证 | evolve.sh 门控 | 第 7 章 |
| 日志记录 | Journal Agent | 第 8 章 |
| Issue 回复 | gh CLI + Issue Agent | 第 7 章 |

---

## 10.4 全书回顾

这本书从"yoyo 是什么"开始（第 0 章），沿着数据流走了一遍全程（第 1 章），打开了核心引擎（第 2 章）、REPL（第 3 章）、prompt 系统（第 4 章）这三个主要黑盒，然后跳出 CLI 本身，看了进化流水线（第 7 章）、记忆系统（第 8 章），回顾了从 200 行到 35,000 行的成长史（第 9 章），最后用三个场景把所有知识串在一起。

yoyo 不只是一个编程助手——它是一个关于"AI 能否自主改进自己"的实验。28 天、796 次提交、从 200 行长到 35,000 行——这个实验仍在继续。

---

### 质检报告

**讲解节奏**
- [x] 每个场景从触发条件开始，按时间线完整走完

**周边知识**
- [x] 每个步骤标注了对应章节，方便回溯

**讲透了吗**
- [x] 场景一覆盖了 CLI→REPL→Prompt→Agent→Tool→Render 全路径
- [x] 场景二覆盖了进化流水线的所有阶段
- [x] 场景三覆盖了 Issue 从提交到解决的完整生命周期

**代码纪律**
- [x] 全章代码片段 0 处

**流程图准确性**
- [x] 所有调用路径基于前 9 章已验证的源码分析
- [x] 无需新增流程图（引用前文的分析）

**过渡自然吗**
- [x] 三个场景覆盖：用户交互、自动进化、社区参与
- [x] 全书回顾简洁收尾

**准确吗**
- [x] 所有模块引用与前文一致
- [x] 数据形态变化与第 1 章的分析一致

**读得下去吗**
- [x] 场景式叙述，具象化
- [x] 表格辅助理解模块参与

**勘误建议**
- 无
