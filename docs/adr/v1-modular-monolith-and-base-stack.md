# 第一版模块化单体与基础技术栈

## 简要概括

错题局第一版需要在现有小型服务器和约 50 人私测目标下，尽快验证照片录题、OCR、错题卡和语音复习闭环。第一版采用 `React + TypeScript + Vite PWA`、FastAPI 模块化单体、PostgreSQL、数据库任务表加单个 Python Worker，以及普通私有腾讯 COS；先控制服务数量，并为 AI/OCR 实验和后续替换保留清晰边界。

## 状态

已接受

## 上下文

### 要解决的问题

两段旧对话留下了互相冲突的工程方案：

- `React + TypeScript + FastAPI + PostgreSQL`，并曾建议 Redis 队列；
- `Next.js + NestJS + Prisma + MySQL`，并曾建议 MySQL `ai_jobs` 或 Redis/BullMQ。

这些内容当时都只是 Agent 建议。用户现已明确选择适合第一版的轻量方案，因此需要确定可以作为后续开工前提的基础技术栈，同时防止把尚未验证的容量、版本和部署细节误写成事实。

### 决策依据

- 优先验证第一版核心学习闭环，而不是提前建设通用平台；
- 尽量减少现有 `4 vCPU / 4 GB / 40 GB` 服务器上的常驻服务；
- 方便直接进行 OCR、图片处理和 AI 结构化结果实验；
- 保持前后端 API 契约明确，并支持 AI Coding；
- 适应结构化学习数据与仍会变化的 AI 衍生结果；
- 对异步任务提供持久状态、重试、幂等和成本审计；
- 允许在真实负载或维护团队发生变化后替换局部组件。

### 考虑过的选项

#### 选项 A：Vite PWA + FastAPI + PostgreSQL + 数据库 Worker

- 做法：静态部署 React PWA；FastAPI 承担模块化单体 API；PostgreSQL 保存业务数据和任务状态；单个 Python Worker 消费 `ai_jobs`；图片存入私有腾讯 COS。
- 优点：常驻服务少，适合当前服务器和私测规模；Python 与 OCR/AI 实验直接衔接；数据库任务状态便于查询、重试、幂等和审计。
- 缺点：前后端使用两种语言；数据库任务表的吞吐和调度能力低于专业消息队列；需要约束 Python 依赖和 Worker 并发。
- 适用条件：核心目标是低成本验证闭环，后台任务量尚未被证明需要独立队列。

#### 选项 B：Next.js + NestJS + Prisma + MySQL + Redis/BullMQ

- 做法：前后端主要使用 TypeScript，以 Next.js 和 NestJS 分别承担 Web 与 API，并用 Redis/BullMQ 处理任务。
- 优点：开发语言统一；NestJS 模块约束清晰；BullMQ 适合成熟的并发任务处理。
- 缺点：Next.js 服务端、NestJS、Redis 和 Worker 增加部署与资源成本；当前产品不依赖 SEO 或服务端渲染；本地 OCR/AI 实验仍可能引入 Python。
- 适用条件：长期维护者主要是 TypeScript 工程师，AI/OCR 已全部外部服务化，并且 SSR 或 Redis 队列已有真实需求。

#### 选项 C：NestJS 主后端加独立 Python AI/OCR 服务

- 做法：业务 API 使用 NestJS，另建 Python 服务运行 OCR 与 AI 处理。
- 优点：业务团队可以保持 TypeScript，Python 能力独立扩缩。
- 缺点：第一版就需要维护两个后端运行时、服务协议、部署和故障边界，复杂度与当前规模不匹配。
- 适用条件：AI/OCR 负载和团队边界已经独立到值得拆分服务。

## 决策

我们将采用选项 A：

- Web 前端使用 `React + TypeScript + Vite PWA`；
- 业务后端使用 `Python + FastAPI + Pydantic + SQLAlchemy + Alembic`，保持模块化单体；
- 由后端 OpenAPI 契约生成 TypeScript API 客户端；
- 主数据库使用 PostgreSQL；
- 第一版后台任务使用 PostgreSQL `ai_jobs` 表和单个 Python Worker；
- 图片使用普通私有腾讯 COS，关系数据库只保存对象 Key 和业务元数据；
- AI、OCR 和语音能力继续遵守服务端薄适配边界。

第一版不引入 Next.js 服务端渲染、NestJS、MySQL、Redis、BullMQ、Dramatiq、微服务或独立 Python OCR 服务。该决策只确定基础架构方向，不代表精确依赖版本、服务器容量、部署拓扑、数据模式、供应商和成本已经验证。

## 后果

### 正面后果

- 第一版可以用较少的常驻组件启动，降低部署、监控和故障排查成本；
- OCR、图片处理和 AI 实验可以直接使用 Python 生态；
- OpenAPI 客户端生成减轻前后端不同语言造成的接口漂移；
- PostgreSQL 同时承载关系业务数据、可演进的结构化结果和初期任务状态；
- `ai_jobs` 能把任务状态、重试、幂等和成本审计保存在同一权威数据底座；
- 各 AI 能力仍可通过适配器替换供应商。

### 负面后果与代价

- 维护者需要理解 TypeScript 与 Python 两套工具链；
- Python OCR/AI 依赖可能发生版本和平台兼容问题，需要锁定版本并隔离 Worker 依赖；
- 数据库任务队列不适合无限扩展，后续增加并发消费者时需要验证锁竞争和失败恢复；
- Vite PWA 不提供 Next.js 的服务端渲染能力；如果未来出现公开内容和 SEO 需求，需要重新评估；
- 腾讯 COS 的区域、权限、签名 URL、生命周期和删除策略仍需单独设计。

### 风险、缓解措施与后续验证

- 风险：现有服务器不足以同时承载 API、PostgreSQL、Worker 和 OCR。
  - 缓解：限制 Worker 并发，把原图放入对象存储，并允许后续把数据库或 Worker 迁出。
  - 验证：用真实高中题图片、语音链路和约 50 人目标负载完成端到端压测，记录内存、CPU、磁盘和排队时间。
- 风险：数据库任务表逐渐变成难以维护的自制消息队列。
  - 缓解：第一版只实现明确的状态机、租约或锁、重试上限、幂等键和失败记录，不复制专业队列的全部功能。
  - 验证：当任务积压、并发消费者、优先级、延迟任务或运维恢复需求被真实数据证明超出数据库方案时，新建 ADR 比较 Redis/BullMQ、Dramatiq 或云任务队列。
- 风险：双语言降低贡献者上手速度。
  - 缓解：以 OpenAPI 作为唯一接口契约，固定生成客户端、开发命令、代码检查和依赖锁定流程。
  - 验证：如果主要维护者长期变成 TypeScript 团队且 AI/OCR 全部通过外部 HTTP 服务提供，重新比较 NestJS 路线。
- 风险：过早认为框架选择等于部署方案已经完成。
  - 缓解：把版本、数据模式、备份恢复、监控、密钥和供应商评测继续列为后期实施债。
  - 验证：正式开发前逐项形成可执行设计和验收标准。
