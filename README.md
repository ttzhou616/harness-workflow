# Harness Workflow

多 Agent 自动开发工作流——一句话需求 → Board → Dispatcher → 3 Worker 协作，全自动产出可提交的代码。

## 工作方式

```
你说「hermes harness 实现 XXX」
        │
        ▼
    编导（你当前的会话）
    创建 Board + 放置 3 个任务
        │
        ▼
    Kanban Dispatcher 自动调度
        │
   ┌────┼────┐
   ▼    ▼    ▼
Designer  Developer  Tester
(设计)   (TDD编码)  (三维验证)
   │       │        │
   └───┬───┘        │
       │   fix-log  │
       │  管道状态机 │
       └────┬───────┘
            ▼
        PASS → git commit
        FAIL → 自动修复循环(≤3轮)
```

## 三个 Worker

| Worker | Profile | 职责 |
|--------|---------|------|
| Designer | harness-designer | 读 PRD → 查 Supermemory 历史经验 → 写 DESIGN.md |
| Developer | harness-developer | TDD 编码 + 修复模式（最多 3 轮自动修复） |
| Tester | harness-tester | 单元测试 + 验收对照 + 代码质量 → PASS 归档 or FAIL 创建修复任务 |

## 核心机制

- **管道状态机**：fix-log.json 驱动 Worker 分流（design_done → dev_done → test_pass / test_fail）
- **跨轮经验**：Supermemory 知识图谱存储 DO/DON'T/PREFER/PATTERNS，新项目自动复用
- **修复循环**：Tester 发现 FAIL → 自动创建修复任务 → Developer 读 fix-log 重建上下文 → 修完复测，全程不等人工
- **异步持久**：Kanban Board 持久化调度，关闭终端 Worker 继续跑

## 安装

```bash
hermes skills install https://github.com/ttzhou616/harness-workflow
```

## 首次使用

创建 3 个 Worker Profile（只需一次）：

```bash
hermes profile create harness-designer --clone
hermes profile create harness-developer --clone
hermes profile create harness-tester --clone
```

配置 Supermemory memory provider：

```bash
hermes config set memory.provider supermemory
```

## 使用

```
hermes harness 实现一个用户登录系统
```

然后用「查看 harness 进度」查看状态。

## 前置依赖

- Hermes Agent（Kanban 版）或 Claude Code v2.1.32+（Agent Teams 版）
- Supermemory（memory provider，免费额度 $5/月）
- Hermes 版需 3 个 Worker Profile

## 项目结构

```
harness-workflow/
├── SKILL.md                    # 编导：建 Board + 监控
├── worker-contexts/
│   ├── designer.md             # 设计 Worker 上下文
│   ├── developer.md            # 开发 Worker 上下文
│   └── tester.md               # 测试 Worker 上下文
├── templates/
│   └── supermemory.json        # Supermemory 配置模板
└── references/
    ├── pipeline-state-machine.md
    ├── fix-log-schema.md
    └── kanban-cli-gotchas.md
```

## Claude Code Agent Teams 版

> ⚠️ 限制：Agent Teams 是同步模式，关闭 CLI 团队即销毁。适合开着终端盯完的场景。需要持久异步调度请用 Hermes 原版。

### 前置条件

- Claude Code v2.1.32+
- 开启实验特性：`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
- 建议 tmux（分屏模式体验更好）

```json
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### 架构

```
Lead（你当前会话，加载 harness-workflow SKILL.md 作为编导）
  │
  ├─ Designer Teammate    ← 角色 prompt: worker-contexts/designer.md
  │   读 PRD → 查历史 → 写 DESIGN.md + fix-log.json
  │
  ├─ Developer Teammate   ← 角色 prompt: worker-contexts/developer.md
  │   读 DESIGN + fix-log → TDD 编码 → CHANGES.txt
  │
  └─ Tester Teammate      ← 角色 prompt: worker-contexts/tester.md
      三维验证 → test-report.md
      PASS → git commit / FAIL → Lead 创建修复任务
```

### 与 Hermes 原版的区别

| | Hermes 原版 | Claude Code 版 |
|---|---|---|
| 调度 | Kanban Dispatcher 自动 | Lead 手动协调 |
| 持久性 | ✅ 关终端继续跑 | ❌ 团队随会话销毁 |
| 修复循环 | Tester 自治创建修复任务 | Lead 手动 spawn 修复队友 |
| Worker 通信 | Board comment + fix-log.json | Agent Teams 直接消息 |
| 记忆 | Supermemory auto recall | 手动查 Supermemory MCP |
| 任务队列 | Kanban Board (SQLite) | 共享任务列表（文件锁） |

### 使用方式

```
1. 加载 skill 到 Lead 会话
2. 说「实现 XXX，创建 Agent Team：Designer、Developer、Tester」
3. Lead 创建共享任务列表，spawn 3 个 Teammate
4. Designer/Developer/Tester 按序执行，Lead 监控进度
5. Tester PASS → Lead 汇总结果 / FAIL → Lead spawn 修复队友（≤3轮）
```

### 安装 Supermemory MCP（Claude Code 记忆层）

```json
// ~/.claude/mcp.json
{
  "mcpServers": {
    "supermemory": {
      "command": "npx",
      "args": ["supermemory", "mcp"],
      "env": {
        "SUPERMEMORY_API_KEY": "sm_..."
      }
    }
  }
}
```
