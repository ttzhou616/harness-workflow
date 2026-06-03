# Pipeline State Machine — 设计文档

> 版本: v3.2.0 | 日期: 2026-05-22

## 问题背景

v3.1 及之前：Worker 之间通过 fix-log 传递经验数据，但没有明确的管道状态。导致：
- 修复 Worker block 后解封重跑，不知道代码已修完，要么重复工作要么卡住
- 没有「已完成，跳过」的快速路径
- 编导需要手动补建遗漏的复测任务

## 解决方案

fix-log 新增 `pipeline_state` 字段，每个 Worker 把 fix-log 当状态机读/写：

```json
{
  "pipeline_state": "design_done",
  "design_decisions": {...},
  "rounds": [...],
  "cumulative_lessons": [...]
}
```

## 状态转换图

```
design_done → dev_done → (test_pass | test_fail_round_1)
                              ↑                    ↓
                              │              fix_round_1_done
                              │                    ↓
                              │         (test_pass | test_fail_round_2)
                              │                    ↓
                              │              fix_round_2_done
                              │                    ↓
                              │         (test_pass | test_fail_round_3)
                              └────────────────────┘        ↓
                                                    kanban_block()
```

## 实战触发

本次对话中发现的 3 个流程缺陷及修复：

1. **Developer 自己 block 等审核** → 加铁律：Developer 不准用 kanban_block()
2. **kanban_complete() 后 kanban_create() 丢失** → 顺序修复：先建复测再 complete
3. **解封重跑不知道已完工** → 加 pipeline_state 检测 + "解封重跑模式" 章节
