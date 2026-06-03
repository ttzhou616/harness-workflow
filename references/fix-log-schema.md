# fix-log JSON Schema (v3.0)

跨轮经验传递的核心数据结构。每轮 Worker 读写此文件，PASS 后归档 MemPalace。

## 文件命名

`{OUTPUT_DIR}/fix-log-{FEATURE_SLUG}.json`

## 完整 Schema

```json
{
  "design_decisions": {
    "语言/框架": "Python 3.11 + FastAPI",
    "数据库": "SQLite（单机足够）",
    "认证方案": "JWT（非Session，跨服务无状态）",
    "排除的备选方案": ["Session：需Redis"]
  },
  "known_risks": ["token过期处理需覆盖边缘情况"],
  "cumulative_lessons": [
    "令牌验证应统一中间件层",
    "修复前优先验证 known_risks",
    "边缘用例三种场景都需覆盖"
  ],
  "rounds": [
    {
      "round": 1,
      "role": "tester",
      "failures": ["test_auth:42 — token过期返回500"],
      "compared_to_previous": "初始测试",
      "qualitative_assessment": "单点bug，根因在业务逻辑层",
      "lesson": "边缘用例覆盖不足"
    },
    {
      "round": 1,
      "role": "developer",
      "failures_from": "test_auth:42 token过期500",
      "analysis": "jwt.decode 未捕获 ExpiredSignatureError",
      "attempted": ["方案A：全局中间件 → 影响面太大"],
      "fix": "方案C：统一异常处理器 + try/except",
      "files_changed": ["src/auth/login.py"],
      "known_risks": "Exception 捕获可能太宽",
      "lesson": "令牌验证应统一拦截"
    }
  ]
}
```

## 字段说明

| 字段 | 写入者 | 时机 | 含义 |
|------|--------|------|------|
| `design_decisions` | Designer | 设计完成 | 为什么选X、排除了Y |
| `known_risks` | Designer | 设计完成 | 架构风险点 |
| `cumulative_lessons` | Dev + Tester | 每轮追加 | 跨轮沉淀的核心教训 |
| `rounds[].failures` | Tester | FAIL时 | 具体失败测试+原因 |
| `rounds[].compared_to_previous` | Tester | FAIL时 | 修好了什么/还挂什么 |
| `rounds[].qualitative_assessment` | Tester | FAIL时 | 单点bug还是架构问题 |
| `rounds[].analysis` | Developer | 修复完成 | 根因分析 |
| `rounds[].attempted` | Developer | 修复完成 | 放弃的方案及原因 |
| `rounds[].fix` | Developer | 修复完成 | 最终采用的方案 |
| `rounds[].known_risks` | Developer | 修复完成 | 修复后仍存在的风险 |
| `rounds[].lesson` | Dev + Tester | 每轮结束 | 可复用经验 |

## 生命周期

```
Designer 完成 → 创建 fix-log（含 design_decisions + round:[]）
Dev/Tester 每轮 → 追加 rounds[] + cumulative_lessons
PASS → Tester 归档 MemPalace → 删除 fix-log
3轮 FAIL → 保留 fix-log（供人工诊断）
```
