# Kanban CLI 踩坑记录

## `kanban list` 不支持 `--board`

```bash
# ❌ 报错: unrecognized arguments: --board demo-test
hermes kanban list --board demo-test --json

# ✅ 正确: 先切换，再列举
hermes kanban boards switch demo-test
hermes kanban list --json
```

`kanban list` 始终作用于当前激活的 Board。跨 Board 查询时必须 `boards switch`，用完后再切回去。

## `boards switch` 后 `create` 可能落到错误 Board

`hermes kanban boards switch <slug>` 的输出显示 "Active board is now '<slug>'" 但后续 `hermes kanban create` 创建的任务**可能落在之前的 Board 上**。这是 `--board` 参数的竞态问题——`switch` 和 `create` 之间的激活状态可能未刷新。

```bash
# ❌ 不安全：switch 后立即 create 可能落到旧 Board
hermes kanban boards switch new-board
hermes kanban create "任务" --assignee worker ...

# ✅ 安全：每次 create 显式指定 --board
hermes kanban boards switch new-board
hermes kanban create "任务" --assignee worker --board new-board ...

# ✅ 安全：create 后用 show 确认 workspace 路径
hermes kanban show t_xxx  # 检查 workspace 路径是否在正确的 Board 下
```

验证任务归属：
```bash
hermes kanban show <task_id> | grep workspace
# 应该显示: .../boards/<正确board>/workspaces/...
# 如果是:   .../boards/<错误board>/workspaces/... → 建错了 Board
```

## `kanban boards list --json` 字段名

| 直觉中的名字 | 实际字段名 |
|-------------|-----------|
| `display_name` | `name` |
| `task_count` | `total`（顶层）或 `counts`（包含分状态计数） |

示例输出片段：
```json
{
  "slug": "demo-test",
  "name": "Demo Test",
  "total": 3,
  "counts": {"done": 1, "running": 1, "todo": 1}
}
```

## `kanban create` 参数名

| 直觉中的名字 | 实际参数名 |
|-------------|-----------|
| `--assign` | `--assignee` |
| `--depends-on` | `--parent`（可重复，支持多个父任务） |
| `--context` | `--body`（任务描述文本，不是文件路径） |

Worker context 应放进 `--body`，不是 `--context`：
```bash
# ✅ 正确
hermes kanban create "设计" --assignee harness-designer --body "读 PRD.md，输出 DESIGN.md"

# ❌ 错误
hermes kanban create "设计" --assign harness-designer --context "..."
```

## `kanban init` 必须先于 `kanban boards create`

```bash
# ❌ 直接 boards create 会报错（没有 kanban.db）
hermes kanban boards create demo-test

# ✅ 正确顺序
hermes kanban init          # 创建 ~/.hermes/kanban.db
hermes kanban boards create demo-test
hermes kanban boards switch demo-test
```

## `--body` 不支持文件路径引用

`--body` 是纯文本参数。如果想注入 `worker-contexts/*.md` 的内容，需要在 Hermes 对话中先 `read_file` 读取模板内容，然后拼接到 `--body` 参数中。`--skill` 参数可以强制加载一个 skill 给 Worker，适用于 skill 级别的上下文。但文件级别的 context 仍需自己拼。`--skill` 是 Worker 端的等效，编导侧不需要（因为编导已经 load 了 hermes-harness-kanban）。

> ⚠️ 实测：`--body` 里放完整 worker context（~2KB+）会因为 CLI 参数转义出问题。**正确做法**：`--skill hermes-harness-kanban` 让 Worker 自动加载本 skill（含 worker-contexts），`--body` 只放项目级参数：`ROLE=designer | FEATURE_SLUG=xxx | OUTPUT_DIR=C:\...`。Worker 根据 ROLE 值从 skill 中匹配对应的 worker-context。

## `boards switch` 后务必验证

```bash
# ❌ 常见错误：switch 后直接 create，任务落到错误 Board
hermes kanban boards switch harness-hermes-web
hermes kanban create "设计" --assignee harness-designer --body "..."
# → 任务可能落在之前的 Board 上！

# ✅ 正确：switch 后先 list 确认，再看 Board 名
hermes kanban boards switch harness-hermes-web
hermes kanban list | head -1        # 确认 Board: harness-hermes-web
hermes kanban create "设计" ...
```

如果任务已创建到错误 Board，用 `hermes kanban show <task_id>` 查看 workspace 路径确认归属。

## `--parent` 创建的任务初始状态是 `todo` 而非 `blocked`

```bash
hermes kanban create "任务2" --parent t_xxx
# 输出: Created t_yyy (todo, assignee=harness-developer)
```

设置了父依赖，但任务状态显示 `todo` 而非 `blocked`。Dispatcher 会在扫描时检查依赖：如果父任务未完成，子任务不会被提升为 `ready`。因此 `todo` + 未满足的 `parents` ≈ `blocked`。

在 Agent Archive 面板中，依赖未满足的任务会被渲染在"待处理"列，而非"阻塞"列。如需渲染在阻塞列，需在 JS 面板中根据 `parents` 字段判断。

## Gateway 必须运行

Kanban Dispatcher 运行在 Gateway 进程内。没有 Gateway，任务永远停在 `ready` 状态。

```bash
hermes gateway status  # 确认运行中
hermes gateway start   # 如果未运行
```

## Gateway 崩溃快速诊断

如果 Worker 反复 crash 或任务不推进，先查 Gateway 日志：

```bash
tail -50 ~/.hermes/logs/gateway.log
# 常见根因：
#   ModuleNotFoundError: No module named 'pydantic'  → 依赖丢失
#   ModuleNotFoundError: No module named 'xxx'       → 类似，uv pip install xxx==版本
#   Transient agent failure                          → 可能是 MCP 服务器崩溃
```

修复依赖丢失（Hermes 用 uv 管理 venv）：
```bash
uv pip install --python ~/AppData/Local/hermes/hermes-agent/venv/Scripts/python.exe pydantic==2.13.4
hermes gateway start
```

### MCP 服务器崩溃导致 Agent 初始化失败

症状：飞书能收发但每轮会话重建、新开 CLI 卡住、Gateway 日志反复出现 `Transient agent failure`。

诊断：
```bash
tail -50 ~/.hermes/logs/errors.log
# 找 MCP 相关错误：
#   Failed to parse JSONRPC message from server
#   MCP server 'xxx' initial connection failed (attempt N/3)
#   CancelledError
```

修复：在 ~/.hermes/config.yaml 中给出问题的 MCP server 加 `enabled: false`，然后 `hermes gateway restart`。之后找时间修复 MCP 脚本本身。

### Config YAML 缩进错误

症状：`hermes` 命令输出 `Failed to parse config.yaml: expected <block end>, but found '-'`，所有用户配置被忽略，回退到默认值。

常见于 `platform_toolsets` 或 `mcp_servers` 段落——列表项缩进不对（少 2 格或 4 格）。YAML 严格要求子级缩进必须一致。

诊断：
```bash
# 错误信息会告诉具体行号
hermes config show  # 或任何 hermes 命令，看第一行警告
# → "in config.yaml, line 510, column 3"
```

修复：`read_file offset=505 limit=15` 找到出错行，确认缩进层级。

诊断流程：`gateway status` → `tail gateway.log` → 找 `ModuleNotFoundError` / `Transient agent failure` → 修 → `gateway start`。
