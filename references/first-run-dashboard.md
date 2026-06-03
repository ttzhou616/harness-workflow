# 实战验证记录 — Hermes Dashboard 项目

**日期**: 2026-05-21  
**项目**: Hermes Dashboard（仿 hermes dashboard，Phase 1: WebSocket + 配置查看）  
**Board**: `hermes-dashboard`  
**状态**: 设计完成，开发进行中

## 验证结果

| 阶段 | Worker | 结果 |
|------|--------|------|
| Phase 0 | 编导 | ✅ 创建 PRD.md + Kanban Board + 3 任务 + 依赖链 |
| Phase 1 | harness-designer | ✅ 产出 DESIGN.md (15KB, 337行，含架构图/模块拆分/API设计/组件树) |
| Phase 2 | harness-developer | 🔄 进行中（TDD: tests/ 目录已建，正在写代码） |
| Phase 3 | harness-tester | ⏳ 等待开发完成 |

## 关键观察

1. **Dispatcher 确实在工作**: Gateway PID 36404，每 60s 扫描，Task 1 被自动认领并完成
2. **依赖链正确**: Task 2 在 Task 1 完成后自动变为 ready（之前是 todo/blocked）
3. **Designer Worker 产出质量高**: 337 行详细设计，含架构图、模块拆分、API 签名、前端组件树
4. **Developer Worker 遵循 TDD**: 先建 tests/ 目录，再写实现
5. **Profile 创建简单**: `hermes profile create XXX --clone` 一键克隆默认配置，自动继承 .env 和 API key

## 第二次运行（2026-05-21，React 版）

用户要求在 Phase 0 明确说明技术选型：

**关键决策过程**：
1. 用户确认使用 React（非原生 JS），在动手前要求呈现完整工作流
2. 编导先展示 React vs 原生 JS 对比（用 header+blockquote 格式，避免飞书表格崩坏）
3. 确认后按标准流程：PRD.md → Board → 3 任务（含 --parent 依赖链）
4. Board `harness-dashboard`，Gateway PID 36404 运行中

**和第一次的区别**：
- 前端：原生 JS → React 19 + TypeScript + Vite 7
- PRD 中明确写入技术栈约束（不含 @nous-research/ui、不含 Tailwind）
- 用户在 Phase 0 阶段深度参与技术决策，确认后才开工

## 常见问题

### 新建 Profile 后需要确认
Profile 克隆后自动继承默认的 .env（含 API key），无需额外配置。但建议验证：
```bash
hermes --profile harness-designer config get model.default
# 应输出: deepseek-v4-pro
```

### Board 依赖链的创建
`--parent` 参数在创建任务时设置依赖，但任务状态初始为 `todo`（不是 `blocked`）。
Dispatcher 在扫描时会自动评估依赖：父任务未完成 → 子任务不会被认领。
依赖满足后子任务自动变为 `ready`。
