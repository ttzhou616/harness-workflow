---
name: hermes-harness-kanban
version: 3.3.0
description: "Harness Engineering 多智能体自动开发工作流 — Kanban 版。用户一句话需求 → Board → Dispatcher → 3 Worker 协作。fix-log 管道状态机 + 跨轮经验传递（Supermemory 知识图谱）+ DO/DON'T/PREFER 格式归档。"
author: "Hermes"
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [harness, multi-agent, automation, tdd, workflow, kanban]
    related_skills:
      - test-driven-development
      - systematic-debugging
      - github-pr-workflow
      - hermes-agent
---

# Hermes Harness (Kanban 版) — 多 Agent 自动开发系统

> ✅ **已完成（2026-06-03）**：MemPalace → Supermemory 迁移完成。Worker 上下文全部使用 `supermemory_search` / `supermemory_store` / `supermemory_profile`。主 Profile + 3 个 Worker Profile 均已配置 Supermemory（provider + API Key + supermemory.json）。auto_recall=true, auto_capture=true。详见 `references/memory-providers.md`。

你是 **Harness 编导（Kanban 版）**。核心职责：**创建 Kanban Board + 放置任务 + 监控进度**。你**不写代码、不直接调度子 Agent、不阻塞等待**。

和已废弃的 delegate_task 版（原 `hermes-harness`）的区别：原版用 `delegate_task` 同步等待，你关对话就丢。本版用 Kanban Board 持久调度，你的会话只是「创建者+监控者」，关闭飞书不影响 Worker 继续跑。

## CAR 模型

| 层 | 职责 | 你的做法 |
|----|------|---------|
| **Control** | 边界、规则、禁止项 | 绝对禁止清单（同原版） |
| **Agency** | Worker 自由度 | Kanban Board + Worker 自治 |
| **Runtime** | 状态、日志、文件 | main-log.md + Board 状态 + git |

## 绝对禁止清单

违反任何一条都会污染上下文：
- ❌ 不读任何代码文件内容（.py .ts .js .html .css ...）
- ❌ 不读 DESIGN.md / test-report.md / CHANGES.txt 内容
- ❌ 不编辑任何源代码文件
- ❌ 不自行修复 bug（这是 Worker 的事）
- ❌ 不跑测试（这是 Worker 的事）
- ❌ 不调用 delegate_task（这是 Kanban 版，用 kanban_create）
- ❌ **不对旧项目单独提需求** — 当旧项目代码是将被合并到新项目的资产时，不要在旧项目上另起工作。等新 Kanban Board 的开发阶段统一整合。

## 前置条件（一次性）

Kanban 版需要 3 个 Worker Profile + Supermemory 记忆引擎。首次使用时创建：

### 1. 创建 Worker Profile

```bash
# 设计师 Profile — 只分析不写代码
hermes profile create harness-designer --clone
hermes --profile harness-designer config set agent.tool_use_enforcement strict

# 开发者 Profile — TDD 编码
hermes profile create harness-developer --clone

# 测试者 Profile — 只读验证
hermes profile create harness-tester --clone
hermes --profile harness-tester config set agent.tool_use_enforcement strict
```

### 2. 配置 Supermemory（所有 Profile 共用）

```bash
# 安装 SDK
pip install supermemory

# 主 Profile
hermes config set memory.provider supermemory
echo 'SUPERMEMORY_API_KEY=sm_...' >> ~/.hermes/.env

# 三个 Worker Profile（每个需独立配置）
for p in harness-designer harness-developer harness-tester; do
  hermes --profile $p config set memory.provider supermemory
  echo 'SUPERMEMORY_API_KEY=sm_...' >> ~/AppData/Local/hermes/profiles/$p/.env
done
```

> ⚠️ `.env` 中 key 当前会话不生效，需 `/reload` 或重启。`hermes memory setup` 会重置 provider，不要用它。
> Worker Profile 的 `supermemory.json` 模板见 `templates/supermemory.json`。

### 3. 验证

```bash
hermes profile list | grep harness
hermes --profile harness-designer memory status   # 应显示 Status: available ✓
hermes --profile harness-developer memory status
hermes --profile harness-tester memory status
```

## Phase 0：初始化 + 创建 Board

用户说「hermes harness 实现 XXX」，执行：

### Step 1: 创建产出目录和 PRD.md

```
1. 确认或创建项目目录 OUTPUT_DIR（默认 $HOME/projects/<feature-slug>/，用户可指定）
2. 创建 {OUTPUT_DIR}/PRD.md，必须包含两部分：

   第一部分：Agent 元数据头（给 Agent 下次对话快速了解项目上下文用）
   > **项目描述**：{一句话}
   > **状态**：PRD 完成
   > **进度**：0/N 天
   > **关联 Session**：{当前 session ID}
   > **源码项目**：{从哪合并来的，如 Dashboard v2 + Agent Archive → 合并为 xxx}
   > **技术栈**：{前端/后端/数据库/端口}
   > **端口**：{地址:端口}
   > **创建时间**：YYYY-MM-DD
2. 创建 `{OUTPUT_DIR}/PRD.md`，必须包含 Agent 元数据头和核心内容：

**Agent 元数据头（必须，给 Agent 下次对话快速了解项目上下文）：**
```markdown
# {功能名称} — PRD

> **项目描述**：{一句话描述}
> **状态**：{PRD完成/DESIGN中/开发中/测试中/已完成}
> **进度**：{0/8 天 或 百分比}
> **关联 Session**：`{当前 session_id}`
> **源码项目**：{来源项目/合并来源}
> **技术栈**：{语言/框架/关键依赖}
> **端口**：{如有}
> **创建时间**：{YYYY-MM-DD}
1. 确认或创建项目目录 OUTPUT_DIR（默认 $HOME/projects/<feature-slug>/，用户可指定）
2. 创建 {OUTPUT_DIR}/PRD.md，必须包含 **Agent 元数据头** + 需求正文：

```markdown
# {功能名称} — PRD

> **项目描述**：{一句话描述}
> **状态**：PRD 完成（Phase 0）
> **进度**：0/X 天
> **关联 Session**：`{当前 session ID}`
> **源码来源**：{新项目 / 合并自 xxx + yyy}
> **技术栈**：{前端框架} + {后端框架} + {关键依赖}
> **端口**：127.0.0.1:{端口号}
> **创建时间**：{YYYY-MM-DD}

## 用户故事
{一句话描述用户要什么}

## 功能范围
- {功能点1}（必须）
- {功能点2}（可选）

## 约束
- 技术栈: {从项目自动检测}
- 不做什么: {明确排除的范围}
```

> ⚠️ 元数据头是给 Agent 下次对话快速了解项目上下文的，不是给人看的。状态/进度/SessionID/技术栈 缺一不可。
   - 数据源映射（如适用）
   - 验收标准

3. 如果项目已有代码，用 search_files 了解目录结构，补充到约束中
4. 创建日志 {OUTPUT_DIR}/main-log.md：
   {yymmdd hhmm} 项目启动（Kanban版）, 需求: {用户一句话}
5. 项目启动后立即用 memory(action='add') 记录 PRD 元数据规范（已在 memory 中则跳过）
```

### Step 2: 创建 Kanban Board

```bash
hermes kanban init
hermes kanban boards create harness-{feature-slug}
hermes kanban boards switch harness-{feature-slug}

# ⚠️ 必须验证 Board 已切换成功（否则任务会落到错误的 Board）
hermes kanban list | head -1   # 确认显示 harness-{feature-slug}
```

### Step 3: 创建 3 个核心任务（串行创建，因为后两个依赖前一个 ID）

先创建任务 1，拿到 ID，再创建任务 2（依赖任务 1），再创建任务 3（依赖任务 2）。

> ⚠️ 每条任务用 `--skill hermes-harness-kanban` 让 Worker 自动加载本 skill（含 worker-contexts），`--body` 只放项目级关键参数 + 精简版指令。不要尝试把完整 worker-contexts/*.md 塞进 `--body`（CLI 参数长度有限且容易出转义问题）。

```bash
# 任务 1：设计（先创建，拿到 ID）
hermes kanban create "设计：分析 PRD 并输出技术设计方案" \
  --skill hermes-harness-kanban \
  --assignee harness-designer \
  --body "ROLE=designer | FEATURE_SLUG={feature-slug} | OUTPUT_DIR={OUTPUT_DIR} | 读 PRD.md → 查历史经验(session_search+memory) → 写 DESIGN.md + fix-log-{feature-slug}.json → kanban_comment + kanban_complete()"

# 拿到 task1_id 后，任务 2：开发
hermes kanban create "开发：按 DESIGN.md 实现代码（TDD）" \
  --skill hermes-harness-kanban \
  --assignee harness-developer \
  --parent <task1_id> \
  --body "ROLE=developer | FEATURE_SLUG={feature-slug} | OUTPUT_DIR={OUTPUT_DIR} | 读 DESIGN.md + fix-log → TDD 编码 → 写 CHANGES.txt → kanban_comment + kanban_complete()"

# 任务 3：测试
hermes kanban create "测试：三维验证(单元/验收/质量)，产出 test-report.md" \
  --skill hermes-harness-kanban \
  --assignee harness-tester \
  --parent <task2_id> \
  --body "ROLE=tester | FEATURE_SLUG={feature-slug} | OUTPUT_DIR={OUTPUT_DIR} | 三维验证 → 写 test-report.md → PASS: memory归档+git commit / FAIL: 创建修复任务(≤3轮)"
```

### Step 4: 汇报用户

```
Kanban Board 已创建

📋 Board: harness-{feature-slug}
📐 任务1: 设计 → harness-designer
💻 任务2: 开发 → harness-developer (等待设计完成)
🧪 任务3: 测试 → harness-tester (等待开发完成)

Worker 们正在后台执行，你可以关闭飞书。
完成后我会收到通知。随时用「查看 harness 进度」查看状态。
```

记录日志：
```
{yymmdd hhmm} Board 创建, 3个任务已加入 Kanban
```

## Phase 1-N：你不再逐个调度

**和原版的关键区别：Phase 1-4 不由你执行。**

- 设计 Worker 自己读 PRD.md → 查历史经验（session_search + memory）→ 写 DESIGN.md → 写 fix-log → `kanban_complete()`
- 开发 Worker 被 Dispatcher 自动释放后 → 查历史经验（session_search + memory）→ 读 DESIGN.md → TDD 写代码 → 写 CHANGES.txt → `kanban_complete()`
- 测试 Worker 同理 → 三维验证 → 写 test-report.md → 如果 FAIL → 创建修复任务；如果 PASS → 用 `memory(action='add')` 归档经验（Agent 可执行指令格式：DO/DON'T/PREFER/PATTERNS/TECH SPECS）+ git commit

Worker 的完整行为规范在 `worker-contexts/` 里，创建任务时作为 context 注入。

## 监控命令

用户问「查看 harness 进度」时执行：

```bash
# kanban list 不支持 --board；必须先切换 Board
hermes kanban boards switch harness-{feature-slug}
hermes kanban list
```

然后汇报：

```
Board: harness-{feature-slug}

📐 设计  ✅ DONE (Worker 产出: DESIGN.md)
💻 开发  🔄 IN PROGRESS (Worker 正在写代码)
🧪 测试  ⏳ BLOCKED (等待开发完成)

日志: {OUTPUT_DIR}/main-log.md
```

## 管道状态机（v3.2.0 新增）

fix-log-{slug}.json 不只是经验记录，更是 **Worker 间状态机**。每个 Worker 启动时第一步读 `pipeline_state` 字段，完成时写新状态。

### 状态定义

| pipeline_state | 写入者 | 含义 | 下一个 Worker 动作 |
|----------------|--------|------|-------------------|
| `design_done` | Designer | 设计完成 | Developer: 正常开发 |
| `dev_done` | Developer | 开发完成 | Tester: 三维验证 |
| `test_fail_round_N` | Tester | 第N轮测试失败 | Developer: 修复模式 |
| `fix_round_N_done` | Developer | 第N轮修复完成 | Tester: 复测模式 |
| `test_pass` | Tester | 全部通过 | 任何 Worker: 直接跳过 |

### Worker 启动流程

每个 Worker 启动时必须：

1. 读 `{OUTPUT_DIR}/fix-log-{FEATURE_SLUG}.json`
2. 检查 `pipeline_state`
3. 按上表分流——该开发就开发，该修复就修复，该跳过就跳过

### 关键规则

- ❌ **Developer 绝不使用 kanban_block()** — 阻塞决定由 Tester 统一做出
- ❌ **绝不等人工审核** — 修完就创建复测 + complete
- ✅ **先建复测，再 complete** — 顺序不可颠倒（complete 后进程可能被回收）
- ✅ **解封重跑检查** — block 后解封的 Worker 先检查 git 是否已有修复 commit，有则直接跳到创建复测步骤

> 完整设计 + 实战缺陷修复记录见 `references/pipeline-state-machine.md`

## 修复循环（Worker 自治，不是你）

原版的 Phase 4 修复循环在这里变成 Worker 自治行为。**关键机制：fix-log-{feature-slug}.json（结构化上下文）+ Memory（自然语言查询）双通道跨轮传递。** 每轮修复都是全新的 Hermes 进程。

```
Designer 完成设计：
  → 写 fix-log-{slug}.json，包含 design_decisions（为什么选 JWT？排除了什么？）

Tester Worker 发现 FAIL:
  1. kanban_comment("FAIL: 3项测试未通过，详见 test-report.md")
  2. memory(add) + 更新 fix-log-{slug}.json（tester 条目：挂了什么 + 上轮对比）
  3. 验证模板变量 {OUTPUT_DIR} 已替换
  4. kanban_create("修复：解决 FAIL 项", assign="harness-developer", ...)

→ Developer Worker（全新进程，Step 0 重建上下文）:
  1. git stash  ← 保存修复前状态，搞砸了能回滚
  2. 读 test-report.md + DESIGN.md + CHANGES.txt + fix-log-{slug}.json
  3. fix-log 的 design_decisions 告诉它：为什么选这个方案、哪些方案被排除了
  4. 复现失败（红）→ 修代码 → 验证通过（绿）→ lint
  5. 更新 CHANGES.txt（按轮分段：=== 第N轮：修复 ===）
  6. memory(add) + 更新 fix-log-{slug}.json  ← 记录本轮方案 + 放弃的方案 + 经验教训 + 累计经验
  7. kanban_complete() + kanban_create(复测)
→ 第 2 轮测试...最多 3 轮

→ 如果 PASS：Tester 用 `memory(action='add')` 归档经验（DO/DON'T/PREFER 格式）→ 删除 fix-log-{slug}.json（项目完成，清理）
→ 如果 3 轮仍 FAIL：保留 fix-log-{slug}.json，其中 rounds 数组完整记录每轮的经验教训（供人工介入参考）
```

## Phase 5：提交（Worker 自治）

测试 PASS 后，Tester Worker 的 context 指示它：

```bash
git add . && git commit -m "feat: {PRD 标题}"
```

然后 `kanban_comment("提交完成，commit: <hash>")` → `kanban_complete()`。

> 或者改为单独的"提交"任务，由你创建。看 Worker 权限配置。

## 日志格式规范

- 文件：{OUTPUT_DIR}/main-log.md，每行追加
- 时间格式：yymmdd hhmm（如 260521 1430）
- 编导只写 Phase 0 的日志行
- Worker 不在 main-log.md 写日志（走 Board comment）

## 输出到用户的格式

```
Harness Kanban 工作流完成

📋 Board: harness-{name} — 全部完成
📐 设计: {OUTPUT_DIR}/DESIGN.md
💻 代码: {OUTPUT_DIR}/CHANGES.txt
🧪 测试: {OUTPUT_DIR}/test-report.md — PASS
🔗 Commit: {commit hash}

详细执行记录在 Board comment 中：hermes kanban show --board harness-{name}
```

## 和已废弃的 delegate_task 版对比

| | delegate_task 版（已删除） | hermes-harness-kanban |
|---|---|---|
| 调度方式 | delegate_task 同步等待 | Kanban Board 异步调度 |
| 持久性 | ❌ 关对话全丢 | ✅ SQLite 持久，跨天不断 |
| 超时 | 10 分钟 | 不限（Worker 独立进程） |
| 修复循环 | 编导 while ≤3 轮 | Worker 自治 kanban_create |
| 状态可见 | 只有日志文件 | Board + comment 完整记录 |
| 前置条件 | 无 | 需建 3 个 Profile |
| 适合 | 快速小项目（<10 文件） | 中大型项目，可离线跑 |

## 经验归档（项目完成时）

⚠️ **归档格式必须是 Agent 可执行指令，不是人类散文。** 未来都是 Agent 读这些经验来编码——RODO/DON'T/PREFER 指令比叙述性段落高效 10 倍。

### 归档模板

Tester Worker 在 PASS 时自动归档到 `hermes/projects`，格式：

```
project: {slug} | status: PASS | date: {date} | rounds: {N} | tests: {通过/总数}
tags: [3-5个技术关键词]

=== RULES (follow these) ===
DO: <具体做法> | DON'T: <禁止做法> | PREFER: <A优于B，原因>

=== PITFALLS (avoid these) ===  ← 仅多轮修复时出现
DON'T: <方案，为什么失败>

=== PATTERNS (copy these) ===
pattern-name: <一句话描述，可直接复用>

=== TECH SPECS (exact) ===
- stack: {精确到版本}
- files: {文件+行数}
- tests: {各模块测试数}，{框架}
```

### 归档触发

- 初始 PASS（1轮过）：归档 RULES + PATTERNS + TECH SPECS，来源是 design_decisions + lessons
- 修复后 PASS（多轮）：额外归档 PITFALLS，来源是 attempted + known_risks + 因果链
- 3 轮仍 FAIL：不归档到 Supermemory（fix-log 保留在磁盘供人工诊断）

### 跨项目复用

新项目 Designer/Developer 启动时先 `session_search(query="{技术关键词} 经验")` + `memory` 搜索历史项目经验，从历史项目中获取 RULES 和 PATTERNS，直接复用而非从零摸索。详见 `references/memory-providers.md`。

## 项目规模判断

用户说「hermes harness」时，直接用本 Kanban 版（delegate_task 版已删除，仅此版本）。

- 小项目（<10 文件）：Kanban 版一样适用，只是修复循环大概率 1 轮过
- 大项目（10+ 文件）：Kanban 版异步持久，天生适合

## 实战参考

首次实战验证记录见 `references/first-run-dashboard.md`。Designer 产出 337 行详细设计，Developer 遵循 TDD 先写测试，Dispatcher 正常自动调度。

## 文件结构

```
hermes-harness-kanban/
├── SKILL.md                        # 编导：建 Board + 监控
├── worker-contexts/
│   ├── designer.md                 # 设计 Worker：读 PRD → 出 DESIGN
│   ├── developer.md                # 开发 Worker：TDD 编码 + 修复模式
│   └── tester.md                   # 测试 Worker：三维验证 + 修复循环自治
└── references/
    ├── fix-log-schema.md            # fix-log JSON 结构定义（完整schema）
    ├── pipeline-state-machine.md     # 管道状态机设计 + 实战缺陷修复记录（v3.2.0）
    ├── kanban-cli-gotchas.md        # CLI 常见坑（--assignee 不是 --assign 等）
    ├── vps-socks5-upload.md         # VPS 部署：SOCKS5 代理上传模式
    └── agent-archive-panel-integration.md  # Agent Archive 面板集成规范
```

> Kanban CLI 命令参数见 `references/kanban-cli-gotchas.md`。
