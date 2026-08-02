# 错题局 · 技术讨论文档（公开副本）

这里收录「错题局」项目的产品与技术讨论文档，公开目的是展示这个项目如何做决策——每一个选择背后的候选方案、理由和放弃的东西。

> ⚠️ 说明：
> - 本目录是主仓库文档的**公开副本**，权威版本以主仓库为准。
> - `PRODUCT-BLUEPRINT.md` 公开时对涉及个人隐私的表述做了脱敏。
> - 文中 `DEC-xxxx`、`SRC-xxxx` 等编号引用的是主仓库私有目录，此处链接不可用属正常。
> - 价格、成本类数字均标注了日期或为假设估算，不构成采购或定价承诺。

## 目录

### 产品蓝图

- [PRODUCT-BLUEPRINT.md](PRODUCT-BLUEPRINT.md) — 产品定义、核心闭环、AI 边界、模型选型对比、竞品分析、收费与开源候选、决策总账

### 架构决策记录（ADR）

| ADR | 主题 | 状态 |
|---|---|---|
| [flutter-android-primary-client.md](adr/flutter-android-primary-client.md) | 主客户端从 React PWA 反转为 Flutter Android APK | accepted |
| [v1-modular-monolith-and-base-stack.md](adr/v1-modular-monolith-and-base-stack.md) | 模块化单体 + 基础栈（前端条款已被上者取代，原位保留） | accepted |
| [ai-capability-adapters-and-cost-aware-routing.md](adr/ai-capability-adapters-and-cost-aware-routing.md) | AI 能力薄适配 + 成本感知路由 | accepted |
| [self-controlled-business-data.md](adr/self-controlled-business-data.md) | 自有业务数据底座，供应商只提供处理能力 | accepted |
| [official-hosted-backend-managed-ai.md](adr/official-hosted-backend-managed-ai.md) | 官方托管 + Managed AI，拒绝第一天做 BYOK | accepted |
| [0001-ai-provider-adapter.md](adr/0001-ai-provider-adapter.md) | 旧对话产物（已被取代，保留以存档决策治理过程） | superseded |
| [template.md](adr/template.md) | ADR 模板 | — |

### 架构文档

- [v1-implementation-plan.md](architecture/v1-implementation-plan.md) — V1 工程实施总纲：模块边界、两阶段入题、复习调度、仪表盘契约
- [ai-cost-architecture.md](architecture/ai-cost-architecture.md) — AI 成本架构（历史候选，非权威）
- [technology-stack-decision.md](architecture/technology-stack-decision.md) — 技术栈候选（历史候选，非权威）

## 未包含的文档

主仓库的 `PROJECT-PLAN.md`（活的项目地图）与 `Docs/testing/`（TDD 证据）未公开：前者含有基础设施与账号标识等不适合公开的信息，后者是内部工程记录。需要了解的可以在 [取舍笔记](../tradeoffs.html) 里看到它们的要点。
