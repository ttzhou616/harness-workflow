# Memory Provider — Supermemory 迁移记录

> 2026-06-03: MemPalace 已卸载，迁移到 Supermemory（Hermes 原生 memory provider）。

## 迁移映射

| MemPalace | Supermemory | 说明 |
|-----------|-------------|------|
| `mempalace_search(wing, room, query)` | `supermemory_search` | hybrid 搜索（记忆+文档），container_tag 按 FEATURE_SLUG 隔离 |
| `mempalace_add_drawer(wing, room, content)` | `supermemory_store(content, container_tag)` | DO/DON'T/PREFER 格式不变，container_tag="{FEATURE_SLUG}" |
| Wing/Room 空间隐喻 | container_tag 命名空间 | 每个项目独立 namespace |

## 配置清单

### 主 Profile（default）
```bash
pip install supermemory
hermes config set memory.provider supermemory
echo 'SUPERMEMORY_API_KEY=sm_xxx' >> ~/.hermes/.env
```

`~/.hermes/supermemory.json`:
```json
{
  "container_tag": "hermes",
  "auto_recall": true,
  "auto_capture": true,
  "max_recall_results": 10,
  "search_mode": "hybrid"
}
```

### Worker Profile（harness-designer/developer/tester）
```bash
for p in harness-designer harness-developer harness-tester; do
  hermes --profile $p config set memory.provider supermemory
  echo 'SUPERMEMORY_API_KEY=sm_xxx' >> ~/AppData/Local/hermes/profiles/$p/.env
done
```

每个 Worker Profile 的 `supermemory.json`:
```json
{
  "container_tag": "hermes",
  "auto_recall": true,
  "auto_capture": true,
  "max_recall_results": 5,
  "search_mode": "hybrid"
}
```

## Worker 工具映射

| Worker | 旧工具 | 新工具 |
|--------|--------|--------|
| Designer | `mempalace_search` | `supermemory_search` + `supermemory_profile` |
| Developer | `mempalace_search` | `supermemory_search` + `supermemory_profile` |
| Tester | `mempalace_search` + `mempalace_add_drawer` | `supermemory_search` + `supermemory_store` |

## MemPalace 遗留数据

- `~/projects/agent-archive/data/mempalace.json` — 9 条结构化记忆（Harness 开发历史）
- `~/.openclaw/workspace/mempalace_data/conversations/` — 对话日志
- ChromaDB 向量库已丢失（ONNX 模型文件在 `.cache/chroma/onnx_models/`，但 SQLite 数据库不在）

### 导入到 Supermemory

用 Python SDK 批量导入，container_tag 按来源隔离：

```python
from supermemory import Supermemory
client = Supermemory(api_key='sm_xxx')
for entry in legacy_entries:
    client.add(content=entry['content'], container_tag='harness-legacy')
```

导入后处于 `queued` 状态，需等几分钟索引完成才能搜到。验证：`client.search.memories(q='Harness', container_tag='harness-legacy')`。

## Auto-capture 策略

Worker Profile 的 `auto_capture` 开/关各有取舍：

| auto_capture | 好处 | 代价 |
|-------------|------|------|
| true | Worker 每轮对话自动入 Supermemory，无需显式调用 | 消耗额度，Worker 的中间技术细节也被记录 |
| false | 只有 Tester PASS 时显式 `supermemory_store` 才记录 | Worker 启动时的 `auto_recall` 搜不到本轮对话上下文 |

当前默认 `true`（全量记录），如果额度紧张可改为 `false`。

## Hermes 备份与数据恢复

Hermes 的 `state-snapshots/` 目录（`hermes update` 前自动生成）包含完整的 state.db 快照，可用来恢复丢失的会话数据：

```bash
# 列出快照
ls ~/AppData/Local/hermes/state-snapshots/

# 查询快照中的会话
sqlite3 <snapshot>/state.db "SELECT id, title, datetime(started_at, 'unixepoch') FROM sessions"

# 搜索特定内容
sqlite3 <snapshot>/state.db "SELECT COUNT(*) FROM messages WHERE content LIKE '%mempalace%'"
```

## Supermemory 定价

- Free: $5/月额度，Hermes 插件可用，超了暂停
- 存储文本: $0.005/1K tokens
- 搜索: $0.005/1K 次查询
- 字节级去重: 重复内容不计费

## 常见坑

1. **`.env` 不生效** — Hermes 启动时读 `.env`，中途加的 Key 需要用 `/reload` 或重启。`hermes memory status` 检查的是环境变量，不是文件。
2. **`hermes memory setup` 会重置 provider** — 交互式 setup 会覆盖 `memory.provider` 为 built-in。配置 Supermemory 用 `hermes config set memory.provider supermemory` + 手动写 `.env`。
3. **Worker Profile 独立配置** — 每个 Profile 有独立的 `.env` 和 `config.yaml`，Supermemory 需要逐个配置。
