# Flutter Android 主客户端

## 简要概括

第一版完整闭环依赖相机、图片临时文件、麦克风和持续语音会话，Android APK 是主要交付终端。项目选择 Flutter 构建 Android 主客户端，Web 降级为静态宣传页；本 ADR 取代 [`v1-modular-monolith-and-base-stack.md`](v1-modular-monolith-and-base-stack.md) 中 `React + TypeScript + Vite PWA` 及 OpenAPI 生成 TypeScript 客户端的前端条款，其余后端、数据库、Worker、COS 和能力适配决策继续有效。

## 状态

`accepted`

## 日期

`2026-08-01`

## 上下文

### 要解决的问题

第一版纵切已经明确包含拍照、多题边界确认、图片处理、OCR 校对、语音输入和复习中的实时反馈。旧 ADR 选择 `React + TypeScript + Vite PWA`，当时目标是用较少常驻服务快速验证 Web/PWA 闭环；现在主交互明显依赖 Android 的相机、麦克风、应用临时目录、前后台生命周期和 APK 分发，原前端条款不再匹配主终端。

后端基础栈没有因此改变。FastAPI OpenAPI 仍是客户端契约的唯一来源，但生成目标需要从 TypeScript 改为 Dart。旧 ADR 还承载 FastAPI 模块化单体、PostgreSQL、`ai_jobs`、单 Python Worker、私有腾讯云 COS 和能力薄适配等已接受结论，因此不能整体覆盖或改写旧文件。

### 决策依据

- 是否稳定支持 Android 相机、麦克风、文件与前后台生命周期；
- 是否能以单个主客户端跑通完整纵切；
- 是否适合 APK 私测分发和真实设备验证；
- 是否保留以后扩展 iOS 的可能，而不增加第一版交付范围；
- 是否继续以 FastAPI OpenAPI 消除客户端与后端契约漂移；
- 是否避免为了客户端引入 Node 与 pnpm 工具链；
- 是否控制第一版 Web、SSR 和多端并行开发成本。

### 考虑过的选项

#### 选项 A：Flutter + Android APK 主终端

- 做法：使用 Flutter 开发业务客户端，第一版只把 Android APK 作为主终端；Dart 依赖由 Flutter/Dart 工具管理，API 客户端从 OpenAPI 生成。
- 优点：相机、麦克风、文件和应用生命周期有明确的移动端实现路径；同一 UI 与状态管理代码以后可以评估 iOS；不需要 Node 或 pnpm。
- 缺点：团队需要维护 Dart 与 Python 两种语言；仍需处理 Android 权限、机型差异、APK 签名和系统版本兼容。
- 适用条件：核心产品体验以 Android 手机的拍照和持续语音交互为中心。

#### 选项 B：继续 React + TypeScript + Vite PWA

- 做法：保留旧 ADR 的 PWA 主端，并通过浏览器媒体 API 接入相机与麦克风。
- 优点：静态部署简单，链接即可访问，已有 Web 生态成熟。
- 缺点：移动浏览器权限、后台行为、文件生命周期和持续语音体验更受浏览器限制；还会保留 Node 与前端包管理工具链，而这些不再属于项目选择。
- 适用条件：主要交互是表单和浏览，拍照与语音只承担非关键增强。

#### 选项 C：原生 Kotlin + Jetpack Compose

- 做法：使用 Kotlin 和 Jetpack Compose 只开发 Android 客户端。
- 优点：Android 平台能力、性能调优和系统 API 集成最直接。
- 缺点：如果以后增加 iOS，需要重新实现主要客户端；对当前纵切而言，平台专用代码与团队学习成本没有形成足够收益。
- 适用条件：产品长期只支持 Android，或深度依赖 Flutter 插件无法覆盖的 Android 专属能力。

#### 选项 D：React Native 或 Expo

- 做法：使用 JavaScript/TypeScript 跨平台移动框架构建客户端。
- 优点：跨平台生态成熟，Web 开发者迁移成本较低。
- 缺点：需要 Node 包管理和对应构建工具链，与本项目“不引入 Node 与 pnpm”的约束冲突；相机与语音仍要处理原生桥接和生命周期。
- 适用条件：维护团队以 TypeScript 为主，且已经接受 Node 移动端工具链。

## 决策

我们将使用 Flutter 构建客户端，并把 Android APK 作为第一版唯一主业务终端，因为完整纵切中的相机、图片文件、麦克风和持续语音会话需要可靠的移动端生命周期与权限控制。第一版不承诺 iOS 交付；Flutter 只保留以后评估 iOS 的技术可能。

Web 降级为静态宣传页，不承载登录后的录题、OCR 确认、错题卡或语音复习，不建立 React PWA 业务端。

本 ADR 明确取代 [`v1-modular-monolith-and-base-stack.md`](v1-modular-monolith-and-base-stack.md) 的以下前端条款：

- `React + TypeScript + Vite PWA` 改为 Flutter + Android APK 主客户端；
- 后端 OpenAPI 生成 TypeScript API Client 改为生成 Dart API Client。

旧 ADR 原位保留且不改写，以保存当时的上下文和理由。除上述两条外，旧 ADR 中 FastAPI + Pydantic v2 + SQLAlchemy + Alembic 模块化单体、PostgreSQL、`ai_jobs`、单 Python Worker、私有腾讯云 COS 和能力薄适配等后端决策不受影响。

项目不引入 Node 与 pnpm。Python 依赖继续只由 `uv` 管理；Flutter/Dart 依赖由 Flutter/Dart 自身工具管理。FastAPI OpenAPI 仍是接口唯一权威，生成的 Dart 客户端进入 `client/lib/api/generated/` 且禁止手工修改。

## 后果

### 正面后果

- 第一版可围绕真实 Android 相机、麦克风、临时文件和前后台切换完成闭环验证；
- APK 便于在受控私测设备上安装并复现权限与机型问题；
- FastAPI OpenAPI 到 Dart 的生成链继续约束前后端接口漂移；
- 项目不需要维护 React 业务端，也不需要 Node 与 pnpm；
- 如果以后确认 iOS 有真实需求，可以复用大部分 Flutter 业务 UI 与状态管理代码再做平台验证。

### 负面后果与代价

- 维护者需要掌握 Dart/Flutter 与 Python 两套语言和调试工具；
- Android 权限、摄像头实现、音频编码、WebSocket、应用恢复和 APK 签名都需要真实设备测试；
- Flutter 插件升级可能带来 Android Gradle、SDK 和机型兼容工作；
- Web 用户不能直接使用核心业务流程，静态宣传页也不能替代 APK 的体验；
- OpenAPI 生成链要验证 Dart model 的可空性、枚举、日期和错误模型，生成代码不能作为未经测试的黑盒。

### 风险、缓解措施与后续验证

- 风险：低端 Android 设备处理照片时内存峰值过高。
  - 缓解：客户端限制上传像素与文件体积，图片重处理放到单并发 Python Worker，客户端只保留当前流程所需临时文件。
  - 验证：在至少一台低内存真实 Android 设备和一台模拟器上完成上限图片的录入闭环并记录峰值内存。
- 风险：麦克风权限、来电、锁屏或网络切换中断复习会话。
  - 缓解：把语音状态设计为可显式结束和可放弃，服务端为每个 WebSocket 提供 `finally` 清理与空闲超时。
  - 验证：自动化覆盖拒绝权限、断网和恢复，真实设备覆盖前后台切换与系统中断。
- 风险：OpenAPI 生成的 Dart 客户端与后端 schema 漂移。
  - 缓解：CI 先导出固定 `openapi.json`，再用固定版本生成器生成 Dart，并拒绝未提交的 diff；日期、枚举和错误响应增加 contract test。
  - 验证：每次后端 API 变更都运行 Dart 客户端生成与 Flutter 编译测试。
- 风险：Flutter 被误解为第一版同时交付 Android、iOS 和 Web。
  - 缓解：验收和发布流程只包含 Android APK，Web 明确只做静态宣传页，iOS 必须由新的范围决策启动。
  - 验证：发布清单中不存在 iOS 或 Flutter Web 业务构建任务。

### 被放弃的备选方案

- 放弃 `React + TypeScript + Vite PWA` 作为业务主端，因为它不再是相机与持续语音闭环的最佳主终端，并且会保留已明确排除的 Node 前端工具链。
- 放弃 Kotlin + Jetpack Compose，因为当前没有必须依赖原生专属实现的证据，而它会关闭未来复用客户端代码到 iOS 的路径。
- 放弃 React Native/Expo，因为它依赖 Node 生态，与本项目已确认的工具链边界直接冲突。
