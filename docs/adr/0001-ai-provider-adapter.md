# ADR 0001：AI Provider Adapter 架构

## 状态

已取代

本文由旧 ChatGPT 对话直接创建并标为 `Accepted`，但没有按当前 ADR 模板记录用户确认边界。用户后来明确确认需要 AI Adapter 和成本架构，因此本决策已由 [`ai-capability-adapters-and-cost-aware-routing.md`](ai-capability-adapters-and-cost-aware-routing.md) 取代。以下原始内容保留，用于追溯当时的建议。

## 原始背景

AI 模型和供应商变化速度快。如果业务代码直接绑定单一供应商，会导致未来迁移困难。

## 原始决策

建立 AI Adapter 层。业务层只调用 OCR Provider、LLM Provider、Speech Provider 和 Embedding Provider 等能力接口，底层可以替换 DeepSeek、Qwen、讯飞、OpenAI 和 Claude。

## 原始后果

- 优点：降低供应商锁定，支持成本优化，方便模型评测；
- 代价：初期增加接口设计成本。
