# 错题局技术栈历史候选

> 状态：历史候选，非当前权威决策。本文由旧 ChatGPT 对话直接写入 GitHub，当时没有经过用户明确拍板，并与后来出现的 `Next.js + NestJS + MySQL` 建议冲突。当前技术状态以 [`PROJECT-PLAN.md`](../../PROJECT-PLAN.md) 为准。

## 当时的背景

错题局面向中国高中生。第一阶段目标不是构建大规模基础设施，而是验证错题录入、AI 整理、复习调度和语音复盘闭环。

当时使用的技术选择原则是：

1. 支持 AI Coding 开发模式；
2. 适合中国大陆部署；
3. 控制早期成本；
4. 保留替换供应商和扩展能力。

## 当时建议的候选架构

- 前端：React + TypeScript
- 后端：Python FastAPI
- 数据库：PostgreSQL
- 缓存/任务：Redis
- 图片存储：腾讯 COS / 阿里 OSS
- AI 接口：Adapter 层统一封装

## 当前处理

- `React + TypeScript + PWA` 是已确定方向，`Vite` 与 `Next.js` 尚待比较；
- `FastAPI + PostgreSQL` 与 `NestJS + MySQL` 都是候选，用户尚未选择；
- Redis 不是第一版已确定依赖，数据库任务表 + 单 Worker 也是候选；
- 模块化单体是 Agent 推荐，仍需用户确认；
- AI 能力薄适配与成本意识已经由用户明确确认，现由 [`Docs/adr/ai-capability-adapters-and-cost-aware-routing.md`](../adr/ai-capability-adapters-and-cost-aware-routing.md) 记录。

本文只用于解释旧对话曾提出过什么，不得作为开工依据。
