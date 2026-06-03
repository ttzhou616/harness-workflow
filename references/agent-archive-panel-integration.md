# Agent Archive 面板集成模式

向 agent-archive 项目添加新数据面板的标准流程（以 Kanban 面板为例）。

## 文件改动清单（7 个文件）

| # | 文件 | 改动 | 关键点 |
|---|------|------|--------|
| 1 | `scan.py` | 新增 `scan_xxx()` + 在 `scan_all()` 调用 | 子进程调用 CLI，JSON 解析 |
| 2 | `web/js/panels/xxx.js` | **新文件**：面板渲染器 | IIFE 模块，`render(container, data)` |
| 3 | `web/index.html` | Tab 按钮 + panel section + script 引用 | 3 处改动 |
| 4 | `web/js/data-loader.js` | `FILES` 数组 + `data.xxx` 映射 | **最容易漏：第 58 行 data 对象** |
| 5 | `web/js/app.js` | `PANEL_RENDERERS` + `_updateBadges` | 2 处改动 |
| 6 | `web/css/components.css` | 新面板样式 | 追加在文件末尾 |
| 7 | `web/data/xxx.json` | 扫描生成的数据 | `_stripMeta` 过滤 `_generated_at` |

## 踩坑

### 坑 1：忘了 data 对象映射

```js
// ❌ 只改了 FILES，忘了改 data 对象
const FILES = [..., 'kanban.json'];  // 加了一行

// ❌ data 对象里没加 → badge 永远是 0
const data = {
  ...
  mempalace: _stripMeta(results['mempalace.json']),
  // 少了: kanban: _stripMeta(results['kanban.json']),
};
```

**症状**：Tab badge 显示 0，但 JSON 文件数据正确。

**教训**：新增数据源时必须同时改 `FILES` 数组（第 12 行）和 `data` 映射对象（第 58 行）。

### 坑 2：JS 命名空间不一致

```js
// ❌ 自己发明命名空间
})(window.__AGENT_ARCHIVE__ || ...);

// ✅ 必须匹配 app.js 中已有的
})(window.AgentArchive || (window.AgentArchive = {}));
```

### 坑 3：CLI `--json` 字段名和直觉不同

参见 `references/kanban-cli-gotchas.md`：`kanban boards list --json` 返回 `name`（不是 `display_name`），`total`（不是 `task_count`）。

## 面板 JS 模板

```js
(function (ns) {
  'use strict';
  if (!ns.Panels) ns.Panels = {};

  function render(container, data) {
    if (!container) return;
    container.innerHTML = '';
    if (!data || data.length === 0) {
      container.innerHTML = '<div class="panel__empty">...</div>';
      return;
    }
    // 渲染逻辑...
    ns.emit('panel:rendered', { panel: 'xxx', count: data.length });
  }

  ns.Panels.xxx = { render };
})(window.AgentArchive || (window.AgentArchive = {}));
```

## scan.py 添加子扫描函数模板

```python
def scan_xxx() -> list[dict]:
    hermes_bin = _find_hermes_cli()
    try:
        result = subprocess.run(
            [hermes_bin, "xxx", "list", "--json"],
            capture_output=True, text=True, timeout=10,
        )
        if result.returncode == 0:
            return json.loads(result.stdout)
    except Exception as e:
        print(f"[WARN] xxx 扫描失败: {e}", file=sys.stderr)
    return []
```

然后在 `scan_all()` 中调用：

```python
xxx = scan_xxx()
xxx.append({"_generated_at": timestamp})
write_json(xxx, "xxx.json")
print(f"[OK] xxx.json — {len(xxx) - 1} 条记录")
```
