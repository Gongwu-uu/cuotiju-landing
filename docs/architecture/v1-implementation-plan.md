---
title: 第一版完整闭环纵切实施计划
status: active
last_updated: 2026-08-01
authority: engineering-implementation-plan
---

# 第一版完整闭环纵切实施计划

> 本文是工程实现依据。实施时严格执行 `RED → GREEN → REFACTOR`，每个工作块保存失败与通过证据；未经用户另行授权，不安装依赖、不提交、不推送、不调用真实付费供应商。

**目标：** 在 Android APK 上跑通“批量拍照加题 → 逐题图片处理 → 批注 OCR 与用户确认 → 存错题卡 → 流式语音复习 → AI 方向判断 → 用户确认 → 固定间隔调度”的完整单链路。

**基础栈：** Flutter Android 客户端；Python `3.13.13`、FastAPI、Pydantic v2、SQLAlchemy、Alembic；PostgreSQL `18.4`；`uv 0.11.14`；私有腾讯云 COS；PostgreSQL `ai_jobs` 加单个 Python Worker。Python 依赖只由 `uv` 管理，项目不引入 Node 或 pnpm。

## 1. 范围与验收

### 1.1 本纵切实现

- 入题分两阶段。第一阶段是批量连拍：Android 客户端只生成灰度加对比度增强的黑白预览，服务端检测题目候选框，用户点加号将题加入待处理列表；拍完即可离开，不强制当场完成逐题处理。第一阶段一旦结束，会话立即封口；同一张原图不能再加题，也不存在“重开”状态。
- 第二阶段按题处理：系统预填题目边界、批注区域及其题目归属，用户只在不正确时调整，然后核对批注文字，并强制录入、确认标准答案；不做任何标签操作。没有已确认的标准答案，单题不得完成第二阶段。
- 题干不做 OCR 或公式识别，长期仅以裁剪图保存。普通文字题可展示服务端二值化裁剪图；几何题等需要保留图形细节的题保留灰度或彩色裁剪图，不做二值化。
- OCR 只提取学生手写批注，并必须返回原文、坐标、行级置信度和行级手写标记；真实供应商候选能力为 TextIn `lines[].handwritten`、百度 `words_type` 或阿里 `recClassify=2`。界面显著标出低置信度行。
- VLM 在 MVP 只以 OCR 批注结果为证据做批注类型分类，不判断知识点，也不作为批注原文的唯一来源；用户可修改系统预填类型。每片批注区域默认就是一个批注块，不做行级合并或拆分；每块小图以 `annotation_crop` 长期与批注文字共存。
- 标准答案不从拍照中提取，并在第二阶段强制录入。用户自己打字或语音说出正确思路为首选；用户不会时，可由 AI 基于题干图辅助生成步骤。两种来源都须经用户确认，以带 `schema_version` 的 `solution_graphs.graph_payload` JSONB 保存，并支持 `linear` 和含等号边界的 `branching`。
- 系统创建私人错题卡，保存用户的初始思路、错因、下次注意事项、批注和复习卡点摘要，并建立第一轮复习日期。
- 复习时只展示题干图，批注收在信息栏且默认收起；MVP 不做应用内画板，中间演算使用纸笔，交互保持纯口述。Flutter 将 `16 kHz`、`16-bit`、单声道 PCM 通过 WebSocket 流式上传，服务端转发流式 ASR；句子最终转写出来后，LLM 才与已确认步骤精确对比。判断不对时只返回表情和“再想想”，不揭示任何内容；只有用户主动点“不会”才展示整题标准答案，每次判断结果和“不会”的触发位置都要记录。
- 服务端用完即删音频，不保存逐字稿；LLM 检测到长停顿、重复或“不知道”时立即把“哪道题、哪一步卡住”写为 `card_notes.note_type = 'stuck_point'`，供用户事后查看。
- 复习每组 `10` 题；一组首轮全部完成后，本组做错题立即各重做一遍。组内重做只用于当次强化，不改变调度结果；调度层仍按 `1/3/7/14/30` 天推进，首轮任一题做错都重置为第一轮并安排次日，即使组内重做答对也不撤销重置，完成第 30 天轮次后进入 `sequence_completed`。
- ASR、OCR、VLM 与 ObjectStorage 分别从四个窄能力端口接入；第一版只使用确定性 Mock 适配器与离线评测样本，后续替换真实供应商不改业务代码。ASR 真实选型优先评估讯飞或阿里云的国内流式服务。
- 同张 `original_photo` 上所有在阶段一结束前已加入的题目都完成第二阶段后，该原图即可删除。会话封口后没有新增题或重开分支；任一第二阶段未完成的会话原图不自动删除，界面显示待处理数量和原图存储占用，只有用户主动丢弃未完成会话时才可清理其原图。

### 1.2 本纵切不实现

- 不实现辨识题、变式题或通用搜题；只在用户明确表示不会时提供需用户确认的 AI 标准步骤辅助生成。
- 不实现社区分享、互惠访问、`share_consents` 或 `public_problem_copies`；但 `problems.visibility` 与 `assets.visibility` 从第一条迁移开始存在且第一版只写入 `private`。
- 不实现 Q 版形象的复杂表情、骨骼动画或动效打磨；客户端只保留静态状态切换所需的占位视觉状态。
- 不实现任何收费、计费、套餐、支付或付费供应商调用。
- 不实现 Web 业务端；Web 只允许静态宣传页，且不进入本纵切代码路径。
- 不实现学科标签、知识点标签和“看了答案才会/老师讲过/独立做对”等个人标记；相关字段如为后续兼容而保留，MVP 必须留空。
- 不实现应用内画板、手写板或演算笔迹保存；用户用纸笔演算，应用只承载口述复习。
- 不实现 Redis、独立消息队列、微服务、独立 OCR 服务、Node 或 pnpm 工具链。
- 不把原始音频、逐字转写或供应商原始语音响应写入 PostgreSQL、日志、对象存储、崩溃报告或分析事件。

### 1.3 跑通标准

- [ ] `python --version` 输出 `Python 3.13.13`，`uv --version` 输出 `uv 0.11.14`，集成测试数据库输出 `PostgreSQL 18.4`。
- [ ] Alembic 从空库执行 `upgrade head` 后得到第 4 节全部表、约束和索引；应用启动路径中不存在 `create_all`。
- [ ] Android APK 能申请相机权限、拍照并通过签名上传地址上传一张不超过 `10 MiB`、不超过 `12 MP` 的 JPEG。
- [ ] 用户可在同一会话批量连拍，客户端显示灰度加对比度预览，服务端返回候选框，点加号后即可结束第一阶段并离开；结束后会话封口，同图加题请求被拒绝且没有重开入口或状态。
- [ ] 第二阶段可逐题调整系统预填边界、修改批注归属并核对批注文字；不出现学科、知识点或个人标记入口，正确预填项无需重新操作。
- [ ] 服务端的光照归一化始终早于二值化，两步只有一份 Python 实现；`detail_preserving` 题干只生成灰度或彩色裁剪图。
- [ ] 题干仅保存图片引用，数据库不存在题干 OCR 或公式识别结果。
- [ ] OCR Mock 只产生手写批注行的原文、坐标、行级置信度和手写标记；低置信度行在界面标出，用户校对文本与供应商原文分字段保存，每块 `annotation_crop` 可长期读取。
- [ ] VLM Mock 仅基于 OCR 批注证据预填批注类型，不判断学科或知识点，不能覆盖 OCR 原文或成为唯一文本来源；一片批注区域默认形成一个块，不做行级合并拆分。
- [ ] 用户打字或语音输入的正确思路可确认为 `solution_graphs`；用户不会时可改用 AI 生成草稿。`linear` 和 `branching` 均经 Pydantic v2 写入/读出校验，且 `branching` 覆盖 `<`、`=`、`>` 边界；没有已确认步骤图时，阶段二完成请求被拒绝。
- [ ] 复习默认只显示题干图，批注信息栏默认收起；用户能看到 `initial_thought`、`error_reason`、`next_caution`、`annotation` 和 `stuck_point` 等卡片备注。
- [ ] 语音复习按 `16 kHz`/PCM16 流式上传；partial 转写只触发跑题趋势事件，final 转写才触发与 `solution_graphs` 的精确步骤对比。判断不对只返回表情和“再想想”，用户主动点“不会”才展示整题答案，判断结果与触发位置均可审计。
- [ ] 长停顿、重复或“不知道”会在复习当下写入 `card_notes.note_type = 'stuck_point'`；结束口述后，服务端音频与逐字转写被清空，数据库与 COS 中检索不到二者。
- [ ] 同张原图的全部已加入题目完成第二阶段前，原图无过期时间且任何自动删除均被拒绝；待处理数量和存储占用可查，用户可主动丢弃未完成会话。
- [ ] AI 的方向判断不能直接推进调度，最终调度仍以用户确认为准。
- [ ] 复习按 `10` 题成组；本组首轮做错题在组末立即重做一次，重做结果不覆盖首轮已经产生的调度推进或重置。
- [ ] 复习页不存在应用内画板或笔迹保存路径，用户可用纸笔演算并完成纯口述闭环。
- [ ] 对任一轮确认 `incorrect` 后，`stage_index` 变为 `0` 且 `next_review_on` 为次日；连续确认正确时依次使用 `3/7/14/30` 天，最后进入 `sequence_completed`。
- [ ] 同一复习确认请求重放不会重复推进；同一 `ai_jobs.idempotency_key` 重放只产生一个任务。
- [ ] 单元、集成、契约和 Android 端到端测试全部通过；每个实施块都保存对应的 RED 与 GREEN 输出。

## 2. 仓库目录结构

```text
.
├── pyproject.toml
├── uv.lock
├── .python-version
├── app/
│   ├── main.py
│   ├── api/
│   ├── shared/
│   ├── identity/
│   ├── assets/
│   ├── ingestions/
│   ├── problems/
│   ├── reviews/
│   ├── jobs/
│   └── capabilities/
├── worker/
├── migrations/
│   └── versions/
├── scripts/
├── openapi/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   ├── e2e/
│   ├── fixtures/
│   └── evidence/
└── client/
    ├── pubspec.yaml
    ├── android/
    ├── lib/
    │   ├── app/
    │   ├── api/generated/
    │   ├── core/
    │   └── features/
    │       ├── capture/
    │       ├── ingestion_review/
    │       ├── cards/
    │       └── voice_review/
    ├── test/
    └── integration_test/
```

### 2.1 客户端导航与页面映射

- 应用外壳常驻三栏底部导航，固定映射为：`home` → 主页、`dashboard` → 仪表盘、`settings` → 设置；Tab 高度和视觉状态由统一设计 Token 控制。
- 主页承载小菊问候、开始复习、拍照录题和错题本入口；仪表盘只展示连续天数、本周点阵、本周题数/时长对比与到期题建议，不展示学科横条；设置页承载账号、提醒、匿名分享预留、吉祥物开关、数据导出和关于入口。
- `capture`、`ingestion_review`、`voice_review`、复习完成页和错题本详情均为全屏流程页，不显示底部 Tab。流程完成后一律回到主页，不落到中间 Tab 或流程栈中的上一页。

- `pyproject.toml`：声明后端、Worker、测试和静态检查依赖，并要求 Python `==3.13.13`。
- `uv.lock`：由 `uv 0.11.14` 生成并作为唯一 Python 依赖锁文件。
- `.python-version`：固定本地与 CI 使用 Python `3.13.13`。
- `app/main.py`：创建 FastAPI 应用并只做路由、依赖和生命周期装配。
- `app/api/`：定义 `/v1` 路由、鉴权上下文、请求/响应模型与统一错误映射，不写业务规则。
- `app/shared/`：放置数据库会话、配置、时钟、事务边界和通用错误，不承载具体业务。
- `app/identity/`：解析认证主体并提供 `user_id`，拥有 `users` 表的仓储。
- `app/assets/`：管理私有对象元数据、签名 URL、图片处理、原图保护与清理授权。
- `app/ingestions/`：编排连拍会话、上传、服务端候选区域、两阶段入题、批注 OCR 和原图删除资格。
- `app/problems/`：管理题干图、第二阶段强制确认的标准步骤图和错题卡创建；MVP 不提供标签能力。
- `app/reviews/`：管理到期列表、语音会话、用户最终确认和固定间隔调度。
- `app/jobs/`：提供通用 `ai_jobs` 仓储、领取、租约、重试与状态机，不包含具体供应商代码。
- `app/capabilities/`：定义四个 Protocol、统一 DTO、Mock 适配器和以后供应商适配器的位置。
- `worker/`：运行唯一 Python Worker，领取任务并在组合根中把任务类型映射到各模块处理器。
- `migrations/`：保存 Alembic 环境与迁移链；迁移链是数据库结构唯一权威。
- `migrations/versions/`：按实施工作块追加可回滚迁移，不允许用 ORM 自动建表替代。
- `scripts/`：保存 OpenAPI 导出、Dart 客户端生成和离线评测入口；脚本不得依赖 Node 或 pnpm。
- `openapi/`：保存由 FastAPI 导出的受版本控制的 `openapi.json`。
- `tests/unit/`：测试纯状态机、Pydantic schema、图片尺寸估算和无 I/O 业务规则。
- `tests/integration/`：针对 PostgreSQL `18.4`、Alembic、文件型 ObjectStorage Mock 和 FastAPI 测试真实事务与约束。
- `tests/contract/`：以同一套用例验证四个端口的 Mock 适配器，并验证 OpenAPI 与生成的 Dart 客户端一致。
- `tests/e2e/`：从 HTTP/WebSocket 层覆盖完整后端闭环。
- `tests/fixtures/`：保存脱敏的离线题图、区域、批注 OCR、ASR 事件和 VLM 响应；不放真实未成年人数据。
- `tests/evidence/`：按工作块保存带时间戳的 RED 失败输出和 GREEN 通过输出。
- `client/pubspec.yaml`：声明 Flutter 与 Dart 依赖，Dart 依赖只由 `dart pub`/Flutter 工具管理。
- `client/android/`：保存 Android Manifest、相机与麦克风权限、APK 构建和网络安全配置。
- `client/lib/app/`：负责导航、依赖装配、主题和应用级状态。
- `client/lib/api/generated/`：保存从 FastAPI OpenAPI 生成的 Dart API 客户端，禁止手工编辑。
- `client/lib/core/`：保存客户端错误显示、本地临时文件、权限和网络基础设施。
- `client/lib/features/capture/`：负责相机采集、上传进度和设备临时原图生命周期。
- `client/lib/features/ingestion_review/`：负责第一阶段加题并在结束后封口，以及第二阶段边界、批注 OCR/归属与强制关键步骤确认。
- `client/lib/features/cards/`：负责错题卡详情和到期列表。
- `client/lib/features/voice_review/`：负责十题分组、PCM 录音帧、WebSocket、partial 趋势、final 方向判断、“不会”触发整题答案、组内错题重做和最终确认；不提供画板。
- `client/test/`：测试 Flutter 状态管理、表单校验、权限失败和调度结果展示。
- `client/integration_test/`：在 Android 模拟器上驱动 Mock 后端完成拍照替身输入到复习确认的端到端流程。

## 3. 模块边界与依赖方向

允许的依赖方向如下：

```text
Flutter client
      ↓ OpenAPI / WebSocket
app.api
      ↓ application services
identity   assets   ingestions → problems → reviews
              ↘        ↓          ↓          ↓
               jobs ← handlers at worker composition root
                         ↓
                 capability Protocols
                         ↑
                  Mock / vendor adapters
```

- `app.api` 可以依赖各模块公开的 application service 与 API schema；模块不得反向导入路由。
- `app.ingestions` 可以依赖 `app.assets` 和 `app.jobs` 的公开服务；它只向 `app.problems` 暴露已完成第二阶段的题干图引用、已确认批注和已确认标准步骤，不暴露 MVP 后置的标签值。
- `app.problems` 可以按 ID 读取已完成 ingestion 结果，可以依赖 `app.assets` 和 `app.jobs`，但不能修改候选框、批注 OCR 原文或原图删除历史。
- `app.reviews` 可以读取已就绪的 problem、solution graph、card 和 asset 展示信息，并依赖 ASR/VLM Protocol；它不得改变题干图、已确认步骤或批注 OCR 结果。
- `app.jobs` 只依赖 `app.shared`，不了解业务表；具体 handler 在 `worker/` 组合根注入，因此通用队列不会反向依赖业务模块。
- `app.capabilities` 的 Protocol 和 DTO 不依赖业务 ORM；适配器只实现端口并返回统一 DTO，不直接写业务表。
- 每个模块拥有自己的 ORM model 和 repository；跨模块写操作必须经过对方 application service，禁止直接导入并修改对方 ORM 实体。
- 所有 JSONB 在 repository 入口和出口经过 Pydantic v2；禁止把未校验的供应商字典直接写入数据库或返回客户端。
- Flutter 只通过生成的 Dart 客户端和明确的 WebSocket 协议访问后端；禁止直接访问 COS 私有 Key、数据库或供应商 API。
- 禁止 domain/application 层依赖 FastAPI、具体 COS SDK、具体 OCR/ASR/VLM SDK 或 Worker 进程代码。
- 禁止循环依赖、跨模块级联删除、运行时 `create_all`、Node 代码生成器和 pnpm 脚本。

## 4. 数据模型

> **演进记录（2026-08-01）：** 本节下方完整 DDL 是 `0001_create_v1_schema` 的基线快照，不逐句重写迁移后的 DDL。最终运行 schema 以 Alembic 链为准：`0002` 将 `display_mode` 收敛为 `binarized/grayscale/color` 并给 `problem_regions` 增加 `has_geometry`；`0003` 给 `assets` 增加 `upload_expires_at`，并把 `deletion_reason` 扩展为 `phase_two_completed/session_discarded/superseded/upload_failed`；`0004` 给 `review_sessions` 增加四档 `self_rating`；`0005` 增加 `activated_at`、`speech_opened_at`、`speech_stream_expires_at`、`speech_stream_id`、`schedule_revision_after`，并建立 `card_notes_stuck_point_idempotency_idx`。R2 通过 `0006` 新增 `daily_sessions`，再通过 `0007` 给 `review_groups` 固定 `session_date`、绑定快照 owner 并保证每组只有一条学习记录。阶段二的 `0008` 新增 `personal_access_tokens`，将 `problems.subject` 启用为四科可选单选字段，并给 `ai_jobs.job_type` 增加 `export_data`；字段说明与第 6 节均按迁移后的最终 schema 描述。

### 4.1 完整 DDL

以下 DDL 使用 PostgreSQL `18.4` 内置 `uuidv7()`。所有时间使用 `timestamptz`，复习到期使用用户可理解的日历日期 `date`。SQLAlchemy model 必须与迁移一致，但只有 Alembic 迁移可以执行结构变更。

```sql
CREATE TABLE users (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    identity_subject text NOT NULL UNIQUE,
    status text NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'disabled', 'deleted')),
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    deleted_at timestamptz,
    CHECK ((status = 'deleted') = (deleted_at IS NOT NULL))
);

CREATE TABLE assets (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    owner_user_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    parent_asset_id uuid REFERENCES assets(id) ON DELETE RESTRICT,
    asset_type text NOT NULL
        CHECK (asset_type IN (
            'original_photo',
            'problem_bw_crop',
            'problem_detail_crop',
            'annotation_crop'
        )),
    object_key text NOT NULL UNIQUE,
    mime_type text NOT NULL,
    width_px integer NOT NULL CHECK (width_px > 0),
    height_px integer NOT NULL CHECK (height_px > 0),
    byte_size bigint NOT NULL CHECK (byte_size > 0 AND byte_size <= 10485760),
    sha256 char(64) NOT NULL CHECK (sha256 ~ '^[0-9a-f]{64}$'),
    visibility text NOT NULL DEFAULT 'private'
        CHECK (visibility IN ('private', 'unlisted', 'public')),
    lifecycle_status text NOT NULL DEFAULT 'upload_pending'
        CHECK (lifecycle_status IN ('upload_pending', 'active', 'pending_delete', 'deleted')),
    deletion_eligible_at timestamptz,
    deletion_reason text
        CHECK (deletion_reason IN ('phase_two_completed', 'session_discarded')),
    deleted_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    CHECK (object_key !~ '(^/|(^|/)\.\.(/|$))'),
    CHECK (
        asset_type <> 'original_photo'
        OR deletion_eligible_at IS NOT NULL
        OR (
            lifecycle_status IN ('upload_pending', 'active')
            AND deleted_at IS NULL
        )
    ),
    CHECK ((deletion_eligible_at IS NULL) = (deletion_reason IS NULL)),
    CHECK (
        (lifecycle_status = 'deleted' AND deleted_at IS NOT NULL)
        OR (lifecycle_status <> 'deleted' AND deleted_at IS NULL)
    )
);

CREATE INDEX assets_owner_created_idx ON assets (owner_user_id, created_at DESC);
CREATE INDEX assets_cleanup_idx ON assets (deletion_eligible_at)
    WHERE lifecycle_status = 'active' AND deletion_eligible_at IS NOT NULL;

CREATE FUNCTION prevent_asset_row_delete()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE EXCEPTION 'asset rows are immutable audit records; use lifecycle_status';
END;
$$;

CREATE TRIGGER assets_no_hard_delete
BEFORE DELETE ON assets
FOR EACH ROW EXECUTE FUNCTION prevent_asset_row_delete();

CREATE TABLE capture_sessions (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    owner_user_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    status text NOT NULL DEFAULT 'capturing'
        CHECK (status IN (
            'capturing',
            'awaiting_phase_two',
            'processing_phase_two',
            'completed',
            'discarded'
        )),
    phase_one_completed_at timestamptz,
    phase_two_completed_at timestamptz,
    discarded_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    CHECK (status <> 'capturing' OR phase_one_completed_at IS NULL),
    CHECK ((status = 'completed') = (phase_two_completed_at IS NOT NULL)),
    CHECK ((status = 'discarded') = (discarded_at IS NOT NULL))
);

CREATE INDEX capture_sessions_owner_status_idx
    ON capture_sessions (owner_user_id, status, created_at DESC);

CREATE TABLE ingestions (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    capture_session_id uuid NOT NULL REFERENCES capture_sessions(id) ON DELETE RESTRICT,
    owner_user_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    original_asset_id uuid NOT NULL UNIQUE REFERENCES assets(id) ON DELETE RESTRICT,
    status text NOT NULL DEFAULT 'upload_pending'
        CHECK (status IN (
            'upload_pending',
            'detecting_regions',
            'awaiting_selection',
            'awaiting_phase_two',
            'processing_phase_two',
            'completed',
            'failed',
            'cancelled'
        )),
    failure_code text,
    selection_completed_at timestamptz,
    phase_two_completed_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    CHECK (phase_two_completed_at IS NULL OR selection_completed_at IS NOT NULL),
    CHECK ((status = 'completed') = (phase_two_completed_at IS NOT NULL))
);

CREATE INDEX ingestions_owner_status_idx
    ON ingestions (owner_user_id, status, created_at DESC);
CREATE INDEX ingestions_capture_session_idx
    ON ingestions (capture_session_id, created_at);

CREATE TABLE problem_regions (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    ingestion_id uuid NOT NULL REFERENCES ingestions(id) ON DELETE RESTRICT,
    source_asset_id uuid NOT NULL REFERENCES assets(id) ON DELETE RESTRICT,
    candidate_order smallint NOT NULL CHECK (candidate_order >= 0),
    x_px integer NOT NULL CHECK (x_px >= 0),
    y_px integer NOT NULL CHECK (y_px >= 0),
    width_px integer NOT NULL CHECK (width_px > 0),
    height_px integer NOT NULL CHECK (height_px > 0),
    selection_source text NOT NULL DEFAULT 'system'
        CHECK (selection_source IN ('system', 'user')),
    selection_status text NOT NULL DEFAULT 'suggested'
        CHECK (selection_status IN ('suggested', 'added', 'dismissed')),
    boundary_review_status text NOT NULL DEFAULT 'system_prefilled'
        CHECK (boundary_review_status IN ('system_prefilled', 'confirmed')),
    display_mode text NOT NULL DEFAULT 'binarized'
        CHECK (display_mode IN ('binarized', 'detail_preserving')),
    boundary_revision integer NOT NULL DEFAULT 0 CHECK (boundary_revision >= 0),
    render_result_revision integer NOT NULL DEFAULT 0 CHECK (render_result_revision >= 0),
    bw_crop_asset_id uuid REFERENCES assets(id) ON DELETE RESTRICT,
    detail_crop_asset_id uuid REFERENCES assets(id) ON DELETE RESTRICT,
    selected_at timestamptz,
    boundary_confirmed_at timestamptz,
    phase_two_completed_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    UNIQUE (ingestion_id, candidate_order),
    CHECK ((selection_status = 'added') = (selected_at IS NOT NULL)),
    CHECK ((boundary_review_status = 'confirmed') = (boundary_confirmed_at IS NOT NULL)),
    CHECK (phase_two_completed_at IS NULL OR boundary_confirmed_at IS NOT NULL),
    CHECK (
        (display_mode = 'binarized' AND detail_crop_asset_id IS NULL)
        OR (display_mode = 'detail_preserving' AND bw_crop_asset_id IS NULL)
    )
);

CREATE INDEX problem_regions_ingestion_idx
    ON problem_regions (ingestion_id, candidate_order);

CREATE TABLE problem_annotations (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    problem_region_id uuid NOT NULL REFERENCES problem_regions(id) ON DELETE RESTRICT,
    source_asset_id uuid NOT NULL REFERENCES assets(id) ON DELETE RESTRICT,
    candidate_order smallint NOT NULL CHECK (candidate_order >= 0),
    x_px integer NOT NULL CHECK (x_px >= 0),
    y_px integer NOT NULL CHECK (y_px >= 0),
    width_px integer NOT NULL CHECK (width_px > 0),
    height_px integer NOT NULL CHECK (height_px > 0),
    assignment_source text NOT NULL DEFAULT 'system'
        CHECK (assignment_source IN ('system', 'user')),
    assignment_revision integer NOT NULL DEFAULT 0 CHECK (assignment_revision >= 0),
    annotation_crop_asset_id uuid REFERENCES assets(id) ON DELETE RESTRICT,
    crop_result_revision integer NOT NULL DEFAULT 0 CHECK (crop_result_revision >= 0),
    classification_payload jsonb NOT NULL DEFAULT '{}'::jsonb,
    classification_revision integer NOT NULL DEFAULT 0 CHECK (classification_revision >= 0),
    confirmed_annotation_type text,
    confirmed_knowledge_points text[] NOT NULL DEFAULT '{}',
    review_status text NOT NULL DEFAULT 'system_prefilled'
        CHECK (review_status IN ('system_prefilled', 'confirmed', 'rejected')),
    confirmed_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    UNIQUE (problem_region_id, candidate_order),
    CHECK (jsonb_typeof(classification_payload) = 'object'),
    CHECK (cardinality(confirmed_knowledge_points) = 0),
    CHECK (
        (review_status = 'confirmed'
            AND annotation_crop_asset_id IS NOT NULL
            AND confirmed_annotation_type IS NOT NULL
            AND confirmed_at IS NOT NULL)
        OR (review_status <> 'confirmed' AND confirmed_at IS NULL)
    )
);

CREATE INDEX problem_annotations_region_idx
    ON problem_annotations (problem_region_id, candidate_order);

CREATE TABLE annotation_ocr_results (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    annotation_id uuid NOT NULL REFERENCES problem_annotations(id) ON DELETE RESTRICT,
    revision integer NOT NULL DEFAULT 1 CHECK (revision > 0),
    provider text NOT NULL,
    model text NOT NULL,
    raw_text text NOT NULL,
    line_results jsonb NOT NULL,
    corrected_text text,
    review_status text NOT NULL DEFAULT 'candidate'
        CHECK (review_status IN ('candidate', 'confirmed', 'rejected')),
    confirmed_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    UNIQUE (annotation_id, revision),
    UNIQUE (id, annotation_id),
    CHECK (jsonb_typeof(line_results) = 'array'),
    CHECK (
        (review_status = 'confirmed' AND corrected_text IS NOT NULL AND confirmed_at IS NOT NULL)
        OR (review_status <> 'confirmed' AND confirmed_at IS NULL)
    )
);

CREATE UNIQUE INDEX annotation_ocr_one_confirmed_idx
    ON annotation_ocr_results (annotation_id)
    WHERE review_status = 'confirmed';

CREATE TABLE problems (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    owner_user_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    problem_region_id uuid NOT NULL UNIQUE REFERENCES problem_regions(id) ON DELETE RESTRICT,
    subject text CHECK (subject IS NULL),
    tags text[] NOT NULL DEFAULT '{}',
    display_mode text NOT NULL
        CHECK (display_mode IN ('binarized', 'detail_preserving')),
    display_asset_id uuid NOT NULL REFERENCES assets(id) ON DELETE RESTRICT,
    status text NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'ready', 'archived')),
    visibility text NOT NULL DEFAULT 'private'
        CHECK (visibility IN ('private', 'unlisted', 'public')),
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    CHECK (cardinality(tags) = 0)
);

CREATE INDEX problems_owner_status_idx ON problems (owner_user_id, status, created_at DESC);

CREATE TABLE solution_graphs (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    problem_id uuid NOT NULL UNIQUE REFERENCES problems(id) ON DELETE RESTRICT,
    source_type text NOT NULL
        CHECK (source_type IN ('user_text', 'user_voice', 'ai_generated')),
    graph_type text NOT NULL CHECK (graph_type IN ('linear', 'branching')),
    schema_version integer NOT NULL CHECK (schema_version > 0),
    graph_payload jsonb NOT NULL,
    status text NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'confirmed')),
    confirmed_by_user_id uuid REFERENCES users(id) ON DELETE RESTRICT,
    confirmed_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    CHECK (jsonb_typeof(graph_payload) = 'object'),
    CHECK (
        (status = 'confirmed' AND confirmed_by_user_id IS NOT NULL AND confirmed_at IS NOT NULL)
        OR (status = 'draft' AND confirmed_by_user_id IS NULL AND confirmed_at IS NULL)
    )
);

CREATE TABLE mistake_cards (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    owner_user_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    problem_id uuid NOT NULL UNIQUE REFERENCES problems(id) ON DELETE RESTRICT,
    status text NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'sequence_completed', 'archived')),
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    archived_at timestamptz,
    CHECK ((status = 'archived') = (archived_at IS NOT NULL))
);

CREATE INDEX mistake_cards_owner_status_idx
    ON mistake_cards (owner_user_id, status, created_at DESC);

CREATE TABLE review_schedules (
    card_id uuid PRIMARY KEY REFERENCES mistake_cards(id) ON DELETE RESTRICT,
    stage_index smallint NOT NULL DEFAULT 0 CHECK (stage_index BETWEEN 0 AND 4),
    next_review_on date,
    last_outcome text CHECK (last_outcome IN ('correct', 'incorrect')),
    sequence_completed_at timestamptz,
    revision integer NOT NULL DEFAULT 0 CHECK (revision >= 0),
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    CHECK (
        (sequence_completed_at IS NULL AND next_review_on IS NOT NULL)
        OR (sequence_completed_at IS NOT NULL AND next_review_on IS NULL)
    )
);

CREATE INDEX review_schedules_due_idx ON review_schedules (next_review_on, card_id)
    WHERE sequence_completed_at IS NULL;

CREATE TABLE review_groups (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    owner_user_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    target_size smallint NOT NULL DEFAULT 10 CHECK (target_size = 10),
    state text NOT NULL DEFAULT 'active'
        CHECK (state IN ('active', 'reinforcing', 'completed', 'abandoned')),
    initial_pass_completed_at timestamptz,
    completed_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    CHECK (state <> 'reinforcing' OR initial_pass_completed_at IS NOT NULL),
    CHECK ((state = 'completed') = (completed_at IS NOT NULL))
);

CREATE INDEX review_groups_owner_state_idx
    ON review_groups (owner_user_id, state, created_at DESC);

CREATE TABLE review_sessions (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    review_group_id uuid NOT NULL REFERENCES review_groups(id) ON DELETE RESTRICT,
    card_id uuid NOT NULL REFERENCES mistake_cards(id) ON DELETE RESTRICT,
    group_position smallint NOT NULL CHECK (group_position BETWEEN 0 AND 9),
    attempt_kind text NOT NULL DEFAULT 'initial'
        CHECK (attempt_kind IN ('initial', 'reinforcement')),
    parent_review_session_id uuid REFERENCES review_sessions(id) ON DELETE RESTRICT,
    state text NOT NULL DEFAULT 'active'
        CHECK (state IN ('active', 'awaiting_confirmation', 'completed', 'abandoned')),
    started_at timestamptz NOT NULL DEFAULT current_timestamp,
    speech_ended_at timestamptz,
    ephemera_purged_at timestamptz,
    answer_revealed_at timestamptz,
    final_outcome text CHECK (final_outcome IN ('correct', 'incorrect')),
    schedule_stage_before smallint CHECK (schedule_stage_before BETWEEN 0 AND 4),
    schedule_stage_after smallint CHECK (schedule_stage_after BETWEEN 0 AND 4),
    next_review_on date,
    confirmed_at timestamptz,
    ended_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    UNIQUE (review_group_id, group_position, attempt_kind),
    CHECK (
        (attempt_kind = 'initial' AND parent_review_session_id IS NULL)
        OR (attempt_kind = 'reinforcement' AND parent_review_session_id IS NOT NULL)
    ),
    CHECK (speech_ended_at IS NULL OR ephemera_purged_at IS NOT NULL),
    CHECK (
        state <> 'completed'
        OR (
            final_outcome IS NOT NULL
            AND confirmed_at IS NOT NULL
            AND ended_at IS NOT NULL
            AND ephemera_purged_at IS NOT NULL
        )
    )
);

CREATE INDEX review_sessions_card_started_idx
    ON review_sessions (card_id, started_at DESC);
CREATE UNIQUE INDEX review_sessions_one_open_per_card_idx
    ON review_sessions (card_id)
    WHERE state IN ('active', 'awaiting_confirmation');

CREATE TABLE review_events (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    review_session_id uuid NOT NULL REFERENCES review_sessions(id) ON DELETE RESTRICT,
    sequence_no integer NOT NULL CHECK (sequence_no >= 0),
    event_type text NOT NULL CHECK (event_type IN ('assessment', 'cannot_solve')),
    assessment_outcome text CHECK (assessment_outcome IN ('on_track', 'off_track')),
    solution_node_id text,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    UNIQUE (review_session_id, sequence_no),
    CHECK (
        (event_type = 'assessment' AND assessment_outcome IS NOT NULL)
        OR (event_type = 'cannot_solve' AND assessment_outcome IS NULL)
    )
);

CREATE INDEX review_events_session_idx
    ON review_events (review_session_id, sequence_no);

CREATE TABLE card_notes (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    card_id uuid NOT NULL REFERENCES mistake_cards(id) ON DELETE RESTRICT,
    review_session_id uuid REFERENCES review_sessions(id) ON DELETE RESTRICT,
    annotation_id uuid REFERENCES problem_annotations(id) ON DELETE RESTRICT,
    solution_node_id text,
    note_type text NOT NULL
        CHECK (note_type IN (
            'initial_thought',
            'error_reason',
            'next_caution',
            'stuck_point',
            'annotation'
        )),
    content text NOT NULL CHECK (length(btrim(content)) > 0),
    created_by text NOT NULL CHECK (created_by IN ('user', 'system', 'llm')),
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    CHECK (note_type <> 'stuck_point' OR review_session_id IS NOT NULL),
    CHECK (note_type <> 'annotation' OR annotation_id IS NOT NULL)
);

CREATE INDEX card_notes_card_created_idx
    ON card_notes (card_id, created_at DESC);
CREATE INDEX card_notes_session_idx
    ON card_notes (review_session_id, created_at)
    WHERE review_session_id IS NOT NULL;

CREATE TABLE daily_sessions (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    user_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    review_group_id uuid NOT NULL,
    date date NOT NULL,
    items jsonb NOT NULL,
    total_sec integer NOT NULL CHECK (total_sec >= 0),
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    FOREIGN KEY (user_id, review_group_id)
        REFERENCES review_groups(owner_user_id, id) ON DELETE RESTRICT,
    UNIQUE (review_group_id),
    UNIQUE (user_id, date, review_group_id),
    CHECK (jsonb_typeof(items) = 'array')
);

CREATE INDEX daily_sessions_user_date_idx
    ON daily_sessions (user_id, date DESC);

CREATE TABLE ai_jobs (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    job_type text NOT NULL
        CHECK (job_type IN (
            'detect_regions',
            'render_region',
            'render_annotation_crop',
            'run_annotation_ocr',
            'classify_annotation',
            'generate_solution_graph',
            'purge_asset'
        )),
    target_type text NOT NULL,
    target_id uuid NOT NULL,
    payload jsonb NOT NULL DEFAULT '{}'::jsonb,
    status text NOT NULL DEFAULT 'queued'
        CHECK (status IN ('queued', 'running', 'retry_wait', 'succeeded', 'failed', 'cancelled')),
    idempotency_key text NOT NULL,
    attempt_count smallint NOT NULL DEFAULT 0 CHECK (attempt_count >= 0),
    max_attempts smallint NOT NULL DEFAULT 3 CHECK (max_attempts BETWEEN 1 AND 10),
    available_at timestamptz NOT NULL DEFAULT current_timestamp,
    leased_by text,
    lease_expires_at timestamptz,
    last_error_code text,
    last_error_message text,
    started_at timestamptz,
    finished_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    updated_at timestamptz NOT NULL DEFAULT current_timestamp,
    UNIQUE (job_type, idempotency_key),
    CHECK (attempt_count <= max_attempts),
    CHECK (jsonb_typeof(payload) = 'object'),
    CHECK (
        (status = 'running' AND leased_by IS NOT NULL AND lease_expires_at IS NOT NULL)
        OR (status <> 'running' AND leased_by IS NULL AND lease_expires_at IS NULL)
    ),
    CHECK (
        (status IN ('succeeded', 'failed', 'cancelled') AND finished_at IS NOT NULL)
        OR (status NOT IN ('succeeded', 'failed', 'cancelled') AND finished_at IS NULL)
    )
);

CREATE INDEX ai_jobs_claim_idx ON ai_jobs (available_at, created_at)
    WHERE status IN ('queued', 'retry_wait');
CREATE INDEX ai_jobs_lease_idx ON ai_jobs (lease_expires_at)
    WHERE status = 'running';

CREATE TABLE capability_runs (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    ai_job_id uuid REFERENCES ai_jobs(id) ON DELETE RESTRICT,
    target_type text NOT NULL,
    target_id uuid NOT NULL,
    capability text NOT NULL
        CHECK (capability IN ('ocr', 'asr', 'vlm', 'object_storage')),
    provider text NOT NULL,
    model text NOT NULL,
    outcome text NOT NULL CHECK (outcome IN ('succeeded', 'failed')),
    latency_ms integer NOT NULL CHECK (latency_ms >= 0),
    usage_json jsonb NOT NULL DEFAULT '{}'::jsonb,
    cost_amount numeric(18, 8) NOT NULL DEFAULT 0 CHECK (cost_amount >= 0),
    currency char(3) NOT NULL DEFAULT 'CNY',
    error_code text,
    started_at timestamptz NOT NULL,
    finished_at timestamptz NOT NULL,
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    CHECK (jsonb_typeof(usage_json) = 'object'),
    CHECK (finished_at >= started_at),
    CHECK ((outcome = 'succeeded') = (error_code IS NULL))
);

CREATE INDEX capability_runs_target_idx
    ON capability_runs (target_type, target_id, created_at DESC);
CREATE INDEX capability_runs_job_idx
    ON capability_runs (ai_job_id, created_at) WHERE ai_job_id IS NOT NULL;
```

#### 阶段二 `0008` 增量

```sql
CREATE TABLE personal_access_tokens (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    user_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    token_hash char(64) NOT NULL UNIQUE
        CHECK (token_hash ~ '^[0-9a-f]{64}$'),
    scopes text[] NOT NULL,
    name text NOT NULL CHECK (length(btrim(name)) BETWEEN 1 AND 100),
    created_at timestamptz NOT NULL DEFAULT current_timestamp,
    last_used_at timestamptz,
    revoked_at timestamptz,
    CHECK (cardinality(scopes) BETWEEN 1 AND 4),
    CHECK (scopes <@ ARRAY[
        'problems:read',
        'annotations:read',
        'reviews:read',
        'export:read'
    ]::text[])
);

CREATE INDEX personal_access_tokens_user_created_idx
    ON personal_access_tokens (user_id, created_at DESC);

ALTER TABLE problems DROP CONSTRAINT problems_subject_check;
ALTER TABLE problems ADD CONSTRAINT problems_subject_check
    CHECK (subject IN ('math', 'physics', 'chemistry', 'biology'));

ALTER TABLE ai_jobs DROP CONSTRAINT ai_jobs_job_type_check;
ALTER TABLE ai_jobs ADD CONSTRAINT ai_jobs_job_type_check
    CHECK (job_type IN (
        'detect_regions', 'render_region', 'render_annotation_crop',
        'run_annotation_ocr', 'classify_annotation', 'generate_solution_graph',
        'purge_asset', 'export_data'
    ));
```

### 4.2 字段说明

#### `users`

- `id`：用户的稳定 UUIDv7 主键。
- `identity_subject`：认证层提供的唯一主体标识，业务表不保存密码或供应商凭据。
- `status`：用户当前是可用、停用还是已逻辑删除。
- `created_at`：用户记录创建时间。
- `updated_at`：用户记录最后更新时间，由 repository 在每次修改时显式写入。
- `deleted_at`：用户完成逻辑删除的时间，未删除时为空。

#### `personal_access_tokens`

- `id`：个人访问令牌的 UUIDv7 主键。
- `user_id`：令牌所属用户；所有 PAT 查询、吊销和调用都以此为 owner 边界。
- `token_hash`：原始高熵令牌的 SHA-256 库存值；明文只在创建响应中展示一次。
- `scopes`：限定为 `problems:read`、`annotations:read`、`reviews:read`、`export:read` 的非空数组。
- `name`：用户可识别的令牌名称，长度 `1..100`。
- `created_at` / `last_used_at` / `revoked_at`：令牌创建、最近一次成功认证和吊销时间；吊销后不再可认证。

#### `assets`

- `id`：对象元数据的 UUIDv7 主键。
- `owner_user_id`：资产的私人所有者。
- `parent_asset_id`：衍生图所对应的上游资产，例如裁剪图指向原图。
- `asset_type`：区分整页原图、二值化题干裁剪图、保细节题干裁剪图和批注区域小图。`annotation_crop` 通常只有几十 KiB，与批注文字长期共存。
- `object_key`：私有 COS 中的对象 Key，数据库不保存公开 URL。
- `mime_type`：对象实际 MIME 类型。
- `width_px`：图片像素宽度。
- `height_px`：图片像素高度。
- `byte_size`：对象字节数，并在数据库层拒绝超过 `10 MiB` 的单文件。
- `sha256`：上传内容的 SHA-256，用于完整性、去重判断和 Mock 固定响应。
- `visibility`：资产的逻辑可见性，第一版只写 `private`。
- `lifecycle_status`：对象处于等待上传、可用、待删还是已从 COS 删除的状态。
- `deletion_eligible_at`：整张原图满足删除前置条件的时间；正常路径只能在该图阶段一封口前已加入的全部题目完成第二阶段时写入，另保留用户主动丢弃未完成会话的显式路径。
- `upload_expires_at`：尚未完成上传确认的 `original_photo` 上传票据过期时间；派生资产必须为空，上传确认或终态清理后清空。
- `deletion_reason`：解释删除条件来自 `phase_two_completed`、`session_discarded`、派生资产被新修订替代的 `superseded`，还是上传失败的 `upload_failed`；未完成且未丢弃的有效会话原图必须为空。
- `deleted_at`：COS 对象删除成功并完成元数据状态更新的时间。
- `created_at`：资产元数据创建时间。
- `updated_at`：资产元数据最后更新时间。

#### `capture_sessions`

- `id`：一次批量连拍会话的 UUIDv7 主键。
- `owner_user_id`：发起连拍的用户。
- `status`：会话处于拍摄中、待第二阶段、处理中、已完成或已主动丢弃；状态机没有 `reopened`，离开 `capturing` 后禁止回到该状态。
- `phase_one_completed_at`：用户结束连拍并可离开第一阶段的时间，也是会话封口时间；写入后拒绝同一会话或其原图的新增题请求。
- `phase_two_completed_at`：会话中阶段一封口前已加入的全部题目完成边界、批注核对和标准答案确认的时间。
- `discarded_at`：用户主动丢弃未完成会话的时间。
- `created_at`：会话创建时间。
- `updated_at`：会话最后更新时间。

#### `ingestions`

- `id`：连拍会话中一张照片的 UUIDv7 主键。
- `capture_session_id`：该照片所属的批量连拍会话。
- `owner_user_id`：发起录入的用户。
- `original_asset_id`：本批次唯一且受保护的上传原图。
- `status`：单张照片在上传、服务端候选检测、第一阶段加题、第二阶段或终态中的当前状态。
- `failure_code`：可机器判断且不含敏感内容的失败代码。
- `selection_completed_at`：该照片的候选题已在第一阶段加入或忽略的时间；写入后选择集合冻结，不再接受新增、恢复或重开。
- `phase_two_completed_at`：该照片上所有已加入题目都完成第二阶段的时间；这是该 `original_photo` 可删除的正常前置。
- `created_at`：录入批次创建时间。
- `updated_at`：录入批次最后更新时间。

#### `problem_regions`

- `id`：题目候选区域的 UUIDv7 主键。
- `ingestion_id`：该区域所属的录入批次。
- `source_asset_id`：区域坐标所基于的 EXIF 方向已规范化原图资产；该资产的 `width_px`/`height_px` 定义权威像素坐标系。
- `candidate_order`：候选区域在同一照片中的稳定显示顺序。
- `x_px`：区域左上角相对原图的横坐标。
- `y_px`：区域左上角相对原图的纵坐标。
- `width_px`：区域在原图坐标系中的宽度。
- `height_px`：区域在原图坐标系中的高度。
- `selection_source`：候选框是服务端系统检测还是用户手动新建。
- `selection_status`：第一阶段中该候选框是待选、已点加号加入还是已忽略。
- `boundary_review_status`：第二阶段边界仍为系统预填还是已由用户确认；用户只在预填不对时修改坐标。
- `display_mode`：最终枚举为 `binarized`、`grayscale`、`color`；几何题至少使用 `grayscale`，确需颜色证据时使用 `color`。
- `has_geometry`：系统或用户确认题目是否包含必须保留的几何细节；为真时不得选择 `binarized`。
- `boundary_revision`：区域框每次系统预填或用户修改后单调递增；异步裁剪结果只能 compare-and-set 写回相同修订。
- `render_result_revision`：题干裁剪处理结果的单调修订号；必须对应当前 `boundary_revision`，旧结果不得改写当前裁剪引用。
- `bw_crop_asset_id`：光照归一化后再二值化的题干裁剪资产。
- `detail_crop_asset_id`：不二值化、保留几何和颜色信息的灰度或彩色题干裁剪资产。
- `selected_at`：用户点加号将该题加入待处理列表的时间。
- `boundary_confirmed_at`：用户确认系统预填边界或完成边界修改的时间。
- `phase_two_completed_at`：该题的边界、批注文字、批注归属和标准答案均已确认的时间；标签不参与 MVP 完成条件。
- `created_at`：候选区域创建时间。
- `updated_at`：候选区域最后更新时间。

#### `problem_annotations`

- `id`：一块学生手写批注区域的 UUIDv7 主键。MVP 中“一片批注区域 = 一个块”，OCR 行只属于块内文本，不触发块的合并或拆分。
- `problem_region_id`：系统预填或用户修改后的批注题目归属。
- `source_asset_id`：批注坐标所基于的整页原图。
- `candidate_order`：同一道题内批注的稳定顺序。
- `x_px` / `y_px` / `width_px` / `height_px`：批注区域在原图坐标系中的边界。
- `assignment_source`：当前归属是系统预填还是用户修改。
- `assignment_revision`：批注框及题目归属每次变化后单调递增；旧归属修订的 OCR、裁剪或分类结果不得覆盖新值。
- `annotation_crop_asset_id`：与批注文字长期共存的 `annotation_crop` 资产。
- `crop_result_revision`：批注裁剪处理结果的单调修订号；必须对应当前 `assignment_revision`。
- `classification_payload`：VLM 只基于 OCR 证据生成的批注类型候选，不包含对原文的替换或知识点判断。
- `classification_revision`：批注类型处理结果的单调修订号；结果落库时必须与任务携带的预期修订 compare-and-set。
- `confirmed_annotation_type`：系统预填且允许用户修改后的批注类型。
- `confirmed_knowledge_points`：为后续版本保留；MVP 不实现知识点标签，数据库约束其始终为空数组。
- `review_status`：批注是系统预填、已确认还是已拒绝。
- `confirmed_at`：用户完成该批注的文字、类型和归属核对的时间。

#### `annotation_ocr_results`

- `id`：一次批注 OCR 候选结果的 UUIDv7 主键。
- `annotation_id`：该 OCR 结果对应的手写批注区域。
- `revision`：同一区域 OCR 候选的递增版本号。
- `provider`：产生结果的适配器标识，Mock 阶段固定为 `mock`。
- `model`：适配器报告的模型或规则集版本。
- `raw_text`：供应商返回的批注原文，不被用户校对覆盖。
- `line_results`：行数组，每行必须有 `text`、`bbox`、`confidence` 和 `handwritten`；适配器必须映射 TextIn `lines[].handwritten`、百度 `words_type` 或阿里 `recClassify=2` 等真实字段。低于统一阈值的行由 API 标记为低置信度。
- `corrected_text`：用户最终核对的批注文字。
- `review_status`：候选结果是待处理、已确认还是已拒绝。
- `confirmed_at`：用户确认该 OCR 版本的时间。
- `created_at`：OCR 候选写入时间。

#### `problems`

- `id`：题目事实记录的 UUIDv7 主键。
- `owner_user_id`：题目的私人所有者。
- `problem_region_id`：题目来自哪个已确认区域。
- `subject`：阶段二可选的单选学科，值只能是 `math`、`physics`、`chemistry`、`biology` 或 `NULL`；不兼任知识点或自由标签。
- `tags`：为后续知识点/通用标签保留；MVP 不实现，数据库约束其始终为空数组。个人标记不建字段。
- `display_mode`：题干图最终使用 `binarized`、`grayscale` 或 `color`，与当前展示资产的处理模式一致。
- `display_asset_id`：题干的唯一长期内容引用；题干不保存 OCR 文本或公式识别结果。复习主区只展示该图。
- `status`：题目处于草稿、可建卡或归档状态。
- `visibility`：题目的逻辑可见性，第一版只写 `private`。
- `created_at`：题目创建时间。
- `updated_at`：题目最后更新时间。

#### `solution_graphs`

- `id`：关键步骤图的 UUIDv7 主键。
- `problem_id`：该关键步骤图所属的唯一题目。
- `source_type`：标准步骤来自用户打字 `user_text`、用户语音 `user_voice` 或用户不会时请求的 `ai_generated`；前两者优先。
- `graph_type`：步骤图是 `linear` 还是 `branching`。
- `schema_version`：`graph_payload` 所遵循的 Pydantic schema 正整数版本。
- `graph_payload`：节点、条件边、期望证据和等号边界组成的完整 JSONB 对象。
- `status`：步骤图是等待用户校对的草稿还是已确认版本。
- `confirmed_by_user_id`：最终确认该步骤图的用户。
- `confirmed_at`：用户确认步骤图的时间。
- `created_at`：步骤图创建时间。
- `updated_at`：步骤图最后更新时间。

阶段二完成前必须已有 `status = 'confirmed'` 的唯一 `solution_graphs`；没有标准答案的题不能写入 `problem_regions.phase_two_completed_at`，也不能触发原图删除资格。

`graph_payload` 只通过 `SolutionGraphV1` 写入和读出。`branching` 必须先有计算节点，再由结构化条件边分支；条件运算符固定为 `<`、`<=`、`=`、`>=`、`>`，所以 `Δ = 0` 不会被两个开区间吞掉。例如：

```json
{
  "entry_node_id": "compute_delta",
  "nodes": [
    {"id": "compute_delta", "kind": "compute", "instruction": "计算判别式 Δ", "expected_evidence": {"left": "Δ", "operator": "=", "right": "b²-4ac"}},
    {"id": "two_roots", "kind": "continue", "instruction": "按两个不等实根继续"},
    {"id": "double_root", "kind": "continue", "instruction": "按两个相等实根继续"},
    {"id": "no_real_root", "kind": "continue", "instruction": "按无实根继续"}
  ],
  "edges": [
    {"from": "compute_delta", "to": "two_roots", "condition": {"left": "Δ", "operator": ">", "right": 0}},
    {"from": "compute_delta", "to": "double_root", "condition": {"left": "Δ", "operator": "=", "right": 0}},
    {"from": "compute_delta", "to": "no_real_root", "condition": {"left": "Δ", "operator": "<", "right": 0}}
  ]
}
```

Pydantic v2 校验器还必须验证：ID 唯一、`entry_node_id` 存在、边的两端存在、线性图无条件分叉、分支图至少有一组条件边、从入口可达所有节点、无环、每个条件组至多一个 `=` 边，并拒绝无法解释等号边界的重叠区间。repository 从数据库读出时再次 `model_validate`；旧 `schema_version` 只能经显式升级函数转换，不能静默按新 schema 解释。

#### `mistake_cards`

- `id`：用户私人错题卡的 UUIDv7 主键。
- `owner_user_id`：错题卡的私人所有者。
- `problem_id`：该卡引用的唯一题目。
- `status`：卡片处于复习序列中、已走完序列或已归档。
- `created_at`：错题卡创建时间。
- `updated_at`：错题卡最后更新时间。
- `archived_at`：用户归档卡片的时间。

#### `review_schedules`

- `card_id`：被调度卡片的主键，同时保证每卡只有一个当前调度。
- `stage_index`：当前应完成的固定间隔索引，`0..4` 对应 `1/3/7/14/30` 天。
- `next_review_on`：下一次到期的日历日期，完成序列后为空。
- `last_outcome`：最近一次用户最终确认的正确或错误结果。
- `sequence_completed_at`：用户完成第 30 天轮次的时间。
- `revision`：每次原子调度更新递增，用于并发控制和响应审计。
- `updated_at`：调度最后更新时间。

#### `review_groups`

- `id`：一次十题复习组的 UUIDv7 主键。
- `owner_user_id`：该复习组所属用户。
- `session_date`：创建组时经服务端日期窗口校验后固定的用户日历日；后续确认必须与之一致，不得改写调度日期。
- `target_size`：MVP 固定为 `10`，由服务在首轮开始前冻结题目顺序。
- `state`：组处于首轮、组内强化、已完成或已放弃。
- `initial_pass_completed_at`：本组十题首轮全部完成的时间；此后立即进入错题重做。
- `completed_at`：本组所有需强化的错题各重做一次后的完成时间。
- `created_at` / `updated_at`：复习组创建和最后更新时间。

#### `review_sessions`

- `id`：一次复习会话的 UUIDv7 主键。
- `review_group_id`：本次复习所属十题组。
- `card_id`：本次复习对应的错题卡。
- `group_position`：题目在本组冻结顺序中的 `0..9` 位置。
- `attempt_kind`：`initial` 参与调度，`reinforcement` 只做本组即时强化，不再次推进或撤销调度。
- `parent_review_session_id`：强化复习指向同题首轮会话，首轮为空。
- `state`：会话处于口述中、等待用户确认、已完成或已放弃。
- `started_at`：会话开始时间。
- `activated_at`：客户端真正激活该题开始作答的时间；完成摘要和 `daily_sessions` 从该时间（为空时回退 `started_at`）计算用时。
- `speech_opened_at`：当前语音流 claim 首次成功建立的时间。
- `speech_stream_expires_at`：当前语音流 claim 的租约过期时间，支持进程异常后的可恢复判定。
- `speech_stream_id`：当前持有语音流的 UUID；仅在语音打开时间和租约同时存在时非空。
- `speech_ended_at`：客户端停止发送音频且 ASR 会话关闭的时间。
- `ephemera_purged_at`：服务端确认音频缓冲和逐字转写已清空的时间。
- `answer_revealed_at`：标准过程首次向用户展示的时间。
- `self_rating`：四档用户自评的具名持久值：`passed`、`understood`、`understood_with_ai_hint`、`still_dont_understand`，逻辑档位依次为 `1..4`。
- `final_outcome`：用户最终确认的正确或错误结果，而不是 AI 粗判。
- `schedule_stage_before`：确认前调度所处的阶段索引。
- `schedule_stage_after`：确认后调度所处的阶段索引。
- `next_review_on`：本次确认产生的下一复习日期，完成序列时为空。
- `schedule_revision_after`：本次确认完成后看到的调度 revision；首轮等于请求 revision 加一，强化轮保持不变，用于幂等重放。
- `confirmed_at`：用户提交最终确认的时间。
- `ended_at`：复习业务会话完成或放弃的时间。
- `created_at`：复习会话记录创建时间。

#### `review_events`

- `id`：单次复习审计事件的 UUIDv7 主键。
- `review_session_id`：事件所属的首轮或强化复习会话。
- `sequence_no`：事件在单题复习中的稳定触发位置，可在不保存逐字稿的前提下定位第几次判断或“不会”。
- `event_type`：记录每次句子判断 `assessment`，或用户主动点“不会”的 `cannot_solve`。
- `assessment_outcome`：只保存 `on_track`/`off_track` 方向结果，不保存用户原话或模型正文。
- `solution_node_id`：判断或“不会”发生时对应的已确认标准步骤节点；尚未定位节点时可为空。
- `created_at`：事件发生时间。

#### `card_notes`

- `id`：一条卡片备注的 UUIDv7 主键。
- `card_id`：备注所属的错题卡。
- `review_session_id`：复习中生成的备注所属会话；`stuck_point` 必填。
- `annotation_id`：批注备注对应的手写批注区域；`annotation` 必填。
- `solution_node_id`：卡点或其他备注对应的 `solution_graphs` 步骤，可为空。
- `note_type`：至少支持 `initial_thought`、`error_reason`、`next_caution`、`stuck_point` 和 `annotation`。
- `content`：人类可读的简短备注；`stuck_point` 只记录哪道题、哪一步卡住，不保存逐字稿。
- `created_by`：备注来自用户、系统还是 LLM。
- `created_at` / `updated_at`：备注创建和最后更新时间。

`card_notes_stuck_point_idempotency_idx` 对 `(review_session_id, solution_node_id, content) NULLS NOT DISTINCT` 建立仅限 `stuck_point` 的唯一索引，保证并发检测同一卡点时只保存一条记录。

#### `daily_sessions`

- `id`：学习记录的 UUIDv7 主键。
- `user_id`：记录所属用户；所有仪表盘聚合都必须以此作为所有权边界。
- `review_group_id`：触发该记录的已完成复习组；全局唯一约束使一个组只能产生一条记录，与 `user_id` 的复合外键强制该组属于同一 owner。
- `date`：直接取复习组固定的 `session_date`，不在完成时接受独立日期改写。
- `items`：JSONB 数组，只快照首轮每题的 `questionId`、向上取整的 `durationSec` 和逻辑自评档位 `grade = 1..4`；不保存逐字稿、音频或模型正文。
- `total_sec`：`items[].durationSec` 的非负总和。
- `created_at`：复习组进入完成态并在同一事务写入学习记录的时间。

#### `ai_jobs`

- `id`：后台任务的 UUIDv7 主键。
- `job_type`：任务是服务端题区检测、题干/批注裁剪、批注 OCR、VLM 批注分类、标准步骤生成、资产清理还是用户全量导出 `export_data`。
- `target_type`：目标业务实体类型，供 handler 选择 repository。
- `target_id`：目标业务实体 UUID。
- `payload`：经对应 Pydantic v2 job schema 校验的最小参数，不包含音频或逐字转写。
- `status`：任务在排队、执行、等待重试、成功、失败或取消中的当前状态。
- `idempotency_key`：同类业务动作的稳定去重键。
- `attempt_count`：已开始的执行次数，领取任务时递增。
- `max_attempts`：允许的最大执行次数，第一版默认三次。
- `available_at`：任务允许被 Worker 领取的最早时间。
- `leased_by`：当前持有租约的 Worker 实例标识。
- `lease_expires_at`：当前租约过期时间。
- `last_error_code`：最后一次失败的稳定错误码。
- `last_error_message`：去除敏感输入后的最后一次错误摘要。
- `started_at`：任务首次开始执行的时间。
- `finished_at`：任务进入成功、失败或取消终态的时间。
- `created_at`：任务创建时间。
- `updated_at`：任务最后更新时间。

#### `capability_runs`

- `id`：一次外部或 Mock 能力调用的 UUIDv7 主键。
- `ai_job_id`：异步调用所属任务；实时语音等同步调用可以为空。
- `target_type`：该能力调用对应的业务实体类型。
- `target_id`：该能力调用对应的业务实体 UUID。
- `capability`：调用 OCR、ASR、VLM 还是 ObjectStorage 能力。
- `provider`：适配器或供应商标识。
- `model`：适配器报告的模型、协议或规则版本。
- `outcome`：调用最终成功或失败。
- `latency_ms`：从适配器调用开始到结束的毫秒数。
- `usage_json`：经 Pydantic v2 校验的页数、秒数或 token 等用量，不含用户原始内容。
- `cost_amount`：本次调用归集的非负成本，Mock 固定为零。
- `currency`：成本币种的三字符代码，第一版为 `CNY`。
- `error_code`：失败时的稳定错误映射，成功时为空。
- `started_at`：能力调用开始时间。
- `finished_at`：能力调用结束时间。
- `created_at`：审计记录写入时间。

### 4.3 明确不建的表

本纵切不创建 `solution_nodes`、`solution_edges`、`review_transcripts`、`audio_assets`、`share_consents`、`public_problem_copies`、价格、支付、独立标签、个人标记、画像、辨识题或变式题表。阶段二只启用 `problems.subject` 的四科可选单选；`problems.tags` 与 `problem_annotations.confirmed_knowledge_points` 仍分别固定为空数组和空数组，不实现知识点、自由标签、多选或独立标签表。题干仅有 `problems.display_asset_id` 图片引用，不建题干 OCR 或公式识别字段；关键步骤统一存在 `solution_graphs.graph_payload`；语音原文和逐字稿不落库；分享与收费不在范围内。

## 5. 图片流水线

> **修订说明（2026-08-01）：** 用户已确认入题采用“批量连拍加题 → 事后逐题处理”两阶段；阶段一结束即封口，同张原图不能再加题且没有重开状态；同张整页彩色原图在阶段一封口前已加入的全部题完成第二阶段后即可删除，未完成会话不自动删原图；阶段二不打标签，但必须确认标准答案；题干不做 OCR 或公式识别，长期只保存裁剪图；服务端图片处理固定先光照归一化、后二值化。

### 5.1 阶段预算

| 阶段 | 格式与分辨率上限 | 单张预估体积 | 存放位置 | 清理时机 |
|---|---|---:|---|---|
| Android 拍摄原图 | JPEG，最长边不超过 `4032 px`，总像素不超过 `12 MP`，上传硬上限 `10 MiB` | 常见 `2–6 MiB`，最坏 `10 MiB` | 设备应用临时目录，随后直传私有 COS | 上传完成且可重试后删设备副本；服务端原图按第 5.3 节保护 |
| 客户端黑白预览 | 仅灰度加对比度增强，不做光照归一化或二值化 | 随预览尺寸，不上传 | Flutter 内存/应用临时目录 | 退出拍摄页或完成上传后删除 |
| 服务端解码 | RGB/RGBA 内存像素，不写对象存储 | `12 MP × 3/4 bytes`，约 `36–48 MiB` | 单 Worker 进程内存 | 当前处理步骤结束即释放，Worker 并发固定为 `1` |
| 光照归一化整页工作图 | `8-bit` 灰度 PNG，最长边不超过 `2480 px` | 编码约 `0.3–2.0 MiB`，解码约 `5.9 MiB` | Worker 独占临时目录 | 二值化/裁剪完成即删；崩溃遗留由一小时 TTL 清理 |
| 二值化整页工作图 | 对归一化结果做自适应阈值 PNG | 约 `0.2–1.5 MiB` | Worker 独占临时目录 | `binarized` 题干裁剪完成即删；不作为 `detail_preserving` 输入 |
| 二值化题干裁剪图 | lossless WebP，最长边和最长宽均不超过 `2000 px` | 常见 `80–800 KiB`，最坏约 `1.5 MiB` | 私有 COS，登记为 `problem_bw_crop` | 作为题干唯一长期内容之一，随题目明确删除流程清理 |
| 保细节题干裁剪图 | 灰度 lossless WebP 或 JPEG quality `90`，最长边不超过 `2000 px`，不二值化 | 常见 `150 KiB–1.5 MiB` | 私有 COS，登记为 `problem_detail_crop` | 作为几何题等题干的唯一长期内容，随题目明确删除流程清理 |
| 批注区域小图 | lossless WebP，仅覆盖已确认批注区域 | 每块通常几十 KiB | 私有 COS，登记为 `annotation_crop` | 与批注文字和卡片长期共存，不随整页原图删除 |
| 批注 OCR 适配输入 | 从 `annotation_crop` 解码或临时转 PNG，不额外复制到 COS | 通常几十 KiB | Worker 临时目录 | 单次 OCR 调用的 `finally` 块立即删除 |

以上是工程预算而非计费承诺。每个处理任务在 `capability_runs.usage_json` 记录输入字节、输出字节和像素数，端到端压测要求单 Worker 处理一张上限照片时新增常驻内存不超过 `96 MiB`。图片库必须在解码前检查文件头、像素数和压缩炸弹限制，禁止仅相信客户端 MIME 与尺寸。

### 5.2 处理顺序

所有坐标统一使用“已应用 EXIF 方向后的原图像素”作为唯一权威坐标系；客户端预览缩放和服务端检测缩放都只能从该坐标系派生。任何含坐标的 API 必须同时携带客户端预览宽高和坐标修订号，服务端据此换算、校验并拒绝旧修订。

1. 客户端创建 `capture_session`。每次拍摄都先应用 EXIF 方向并规范化为 JPEG，以规范化后的像素宽高建立权威坐标系、计算 `sha256`，同时仅在客户端生成灰度加对比度增强的黑白预览。客户端不实现光照归一化或二值化。
2. 客户端为每张照片创建 `ingestion` 并通过单次签名 URL 直传私有 COS。API 用 ObjectStorage `head_object` 校验字节数、内容类型、hash 和对象存在性后，把 `original_photo` 置为 `active`。
3. `detect_regions` 在服务端 Python Worker 中下载规范化原图；检测缩放结果必须映射回权威原图像素，并连同源尺寸和修订号返回。客户端只按预览尺寸映射渲染，不运行候选检测模型。
4. 系统候选框默认为 `suggested`。用户点加号后写为 `added`，也可手动新建题框；此时只选题，不强制精调边界。
5. 用户结束连拍时写 `phase_one_completed_at`，会话进入 `awaiting_phase_two` 并永久封口。用户可立即离开；原图、已加入题和待处理计数都保留，后续对该会话或原图的加题、恢复选择或重开请求一律拒绝。
6. 第二阶段逐题返回系统预填边界、批注区域和批注题目归属。用户只在预填不正确时移动/缩放边界或改归属。
7. `render_region` 的唯一 Python 图片实现先解码并转灰度，再做光照归一化。`binarized` 题干在此之后才二值化并生成 `problem_bw_crop`；`detail_preserving` 题干不进入二值化，从灰度或彩色图生成 `problem_detail_crop`。
8. `render_annotation_crop` 为每块系统预填/用户确认的批注生成 `annotation_crop`。MVP 将每片批注区域直接视为一个块；`run_annotation_ocr` 只读批注小图，统一返回原文与行数组，每行必须有坐标、行级置信度和手写标记。OCR 行只用于核对块内文字，界面对低置信度行加醒目标记，不据此合并或拆分批注块。
9. VLM 只接收 OCR 批注证据及题目归属 ID，预填批注类型，不判断学科或知识点，也不生成或覆盖批注原文。用户只核对文字、类型和归属；阶段二不提供任何标签入口。
10. 用户必须在阶段二通过打字或语音录入正确思路；不会时可请求 AI 基于题干图生成步骤。任一来源都经用户确认后写入 `solution_graphs`，没有 `status = 'confirmed'` 的标准答案时不能完成该题。完成后 `problems.display_asset_id` 只引用题干裁剪图，不写入题干 OCR 文本或公式识别结果。
11. 一张照片上阶段一封口前全部 `selection_status = 'added'` 的题都有 `phase_two_completed_at` 后，同一事务写入 `ingestions.phase_two_completed_at`、`assets.deletion_eligible_at` 和 `deletion_reason = 'phase_two_completed'`，再排入 `purge_asset`；不需要也不得检查重开分支。
12. `purge_asset` 先把资产原子改为 `pending_delete`，再删 COS 对象，最后改为 `deleted`；失败由同一幂等任务重试。会话中全部已加入题完成后，`capture_sessions.status` 才进入 `completed`。

> **待迭代（E22，优先）：** MVP 暂定“一片批注区域默认一个块”，不做行级合并拆分。用户对该粒度有异议；产品实际跑起来后必须优先依据真实使用调整批注块粒度，不能把当前规则视为长期定案。

### 5.3 原图保护的代码落实

- 上传后的原图是用户提交给系统的原始 JPEG，不以相机传感器 RAW 为定义；服务端不得二次压缩后冒充原图。
- 同张原图上任一已加入题尚未完成第二阶段时，`deletion_eligible_at` 和 `deletion_reason` 均必须为空，原图必须保持 `active`。
- 自动清理查询只能选择 `deletion_reason = 'phase_two_completed'` 的原图；不得按上传时间、会话闲置时间或存储压力为未完成会话设置过期时间。
- `assets` 禁止物理删行，业务删除只能走 lifecycle 状态机，因此审计记录不会随 COS 对象消失。
- 只有 `AssetCleanupService` 注入带删除权限的 ObjectStorage；API、OCR handler 和图片 handler 只获得读写但无删除能力的窄包装。
- COS 生命周期规则不得匹配 `original/` 前缀；原图删除只由 `purge_asset` 任务发起，避免云端规则绕过用户确认。
- 最后一道已加入题完成第二阶段时，在数据库事务中锁定源照片对应 ingestion 和原图资产，并用 `NOT EXISTS` 检查是否仍有“阶段一已加入且未完成第二阶段”的题；只有查询为空才能写删除资格。会话已封口，因此无需重开分支。
- 用户未完成第二阶段、离线或流程失败时，原图无限期保持保护状态。界面必须显示待处理会话数、待处理题数与原图字节占用，但不能自动删除。
- 用户可主动丢弃未完成会话。服务端必须显式确认所有权与影响范围，将会话置为 `discarded`，再对其原图写入 `deletion_reason = 'session_discarded'` 并排入清理；不得把“显示待处理数量”视为删除授权。
- `purge_asset` 每次尝试或重试都必须重新锁定并执行同一 `NOT EXISTS` 核验，同时读取资产状态、所有者、`deletion_eligible_at` 和 `deletion_reason`；核验失败立即撤销清理。ObjectStorage 返回“对象不存在”按幂等成功处理，再写 `deleted_at`。

## 6. 语音复习链路

> **修订说明（2026-08-01）：** 复习语音采用 Flutter PCM → WebSocket → 服务端 → 流式 ASR 的全链路流式方案；中间转写只做趋势检测，句子最终转写才与标准步骤精确对比。MVP 提示不分级：判断不对只给表情和“再想想”，用户主动点“不会”才展示整题标准答案，并记录每次判断和“不会”的触发位置。复习十题一组，组末立即重做本组错题；不做应用内画板。标准答案在阶段二强制确认，来源优先级为用户打字/语音正确思路，其次才是用户不会时的 AI 辅助生成。

### 6.1 客户端职责

- 复习主区只展示 `problems.display_asset_id` 题干图；批注放在信息栏并默认收起，用户主动展开时才显示批注小图与文字。
- 每次从到期队列冻结 `10` 题为一组；十题首轮完成后立即按原组顺序重做本组首轮做错题一次。
- 复习页顶部始终展示 `remaining_today` 和“本批 10 题”，每个 card 使用复习组响应中的 `stage_index` 显示当前轮次；流程页全屏且不显示底部 Tab。
- 在开始复习前申请麦克风权限，拒绝权限时仍允许用户查看题目并用手动最终确认完成复习。
- 将音频编码为 `16 kHz`、`16-bit`、单声道 PCM 小帧，通过已认证 WebSocket 连续发送；不创建系统录音库文件。
- 展示连接状态、趋势状态、句子级方向判断与静态表情，并明确标注 AI 不是最终调度裁决者。MVP 不提供应用内画板，中间演算由用户在纸上完成，应用内交互保持纯口述。
- 对 `partial` ASR 事件只更新“可能跑题”等趋势状态；对 `final` 句子事件才显示 LLM 判断。判断为 `off_track` 时只显示表情和固定文案“再想想”，不显示步骤、关键词或分级提示。
- 标准答案默认始终隐藏；只有用户主动点“不会”时才一次性展示整题已确认 `solution_graphs`，并提交当前题内事件序号和已定位的 `solution_node_id`。用户随后从四档按钮提交 `self_rating`：逻辑档位 `1` 你通过了、`2` 你懂了、`3` 在 AI 的提示下懂了、`4` 你还完全不懂；第 3 档只描述用户自评，AI 仍不提供分级提示。
- 组内强化题使用独立的 `reinforcement` 会话；其结果只完成本组，不改写首轮已产生的调度结果。
- 收到服务端的 `ephemera_purged` 事件后删除应用内存中的音频缓存；应用崩溃恢复时也清理孤立缓存。

### 6.2 服务端职责与生命周期

1. `POST /v1/review-groups` 从到期题冻结 `10` 题及顺序；每题 `POST /v1/cards/{card_id}/review-sessions` 创建带 `review_group_id`、`group_position` 和 `attempt_kind` 的 `active` 会话，但表中没有音频或 transcript 字段。
2. WebSocket handler 把 PCM 帧立即转发给 `ASRPort`，只在当前进程的有界 ring buffer 中保留发送所需的短暂音频；音频不写磁盘、COS、PostgreSQL 或 `ai_jobs`。
3. ASR 适配器返回 `partial` 和句子级 `final` 事件。第一版用 Mock；接真实供应商时优先实测讯飞与阿里云的国内端到端延迟、教育术语识别率、断句稳定性和服务端零留存配置。
4. `partial` 文本只存在会话内存中，仅用于识别明显跑题关键词等趋势；不传入精确步骤对比，不写入长期数据，也不以抢先 `200–500 ms` 为实施目标。
5. 每当 ASR 产生一句 `final` 文本，`VLMPort.assess_review_sentence` 才将该句与已确认 `solution_graphs.graph_payload` 对比。服务端先按单调 `sequence_no` 写入只含 `on_track`/`off_track`、`solution_node_id` 和时间的 `review_events`；`off_track` 响应只允许表情和固定文案“再想想”，不得携带任何答案内容。请求、转写和模型正文禁止被日志或事件表记录。
6. 同一次对比还检测长时间停顿、连续重复和“不知道”。命中时不等到会话结束，立即幂等写入 `card_notes`：`note_type = 'stuck_point'`、`review_session_id`、`solution_node_id` 与不含逐字原文的“哪道题、哪一步卡住”摘要。
7. 用户停止口述后，handler 依次执行 ASR `finish()`、处理最后的 `final` 句子、关闭 ASR 会话、覆盖并释放音频 buffer、清空 partial/final transcript 列表，然后写 `speech_ended_at` 和 `ephemera_purged_at`。
8. 用户主动点“不会”时，服务端在展示答案前写 `event_type = 'cannot_solve'` 的 `review_events`，以 `sequence_no` 和当时的 `solution_node_id` 记录触发位置，再返回整题已确认标准答案；系统不得从判断结果自动揭示答案。
9. 首轮用户最终确认后，服务端先保存四档 `self_rating`，再独立派生调度用 `final_outcome`：档位 `1/2 → correct`，档位 `3/4 → incorrect`。AI 的 `on_track/off_track` 判断不参与该映射，也不能直接推进调度。十题首轮全部结束后，服务端仅为首轮 `incorrect` 的题创建一次 `reinforcement` 会话；强化结果不得更新 `review_schedules`。
10. 复习组进入 `completed` 时，在更新组完成时间的同一数据库事务中插入一条 `daily_sessions`；唯一约束与 `ON CONFLICT DO NOTHING` 共同保证确认重放不会重复生成学习记录。
11. WebSocket 异常、客户端离线和 handler 取消都在 `finally` 中关闭端口并清空内存；十分钟无帧的会话由进程内 idle watchdog 关闭并标记 `abandoned`。进程崩溃不留下音频或 transcript，启动恢复只需把超时 `active` 会话标记 `abandoned`。

日志过滤器必须拒绝字段名 `audio`、`audio_bytes`、`transcript`、`raw_text` 出现在 review 路由的结构化日志中。`capability_runs` 对 ASR 只记录秒数、时延、供应商、成功状态和零成本 Mock 用量。以后启用真实供应商前，必须单独验证其服务端留存条款与关闭留存的配置。

### 6.3 长期保存内容

- 保存用户最终选择的四档 `self_rating` 及其独立派生的 `final_outcome`；前者解释完成页自评，后者是调度权威输入。
- 保存每次句子判断的 `on_track`/`off_track`、题内 `sequence_no` 和可选 `solution_node_id`，以及用户主动点“不会”的同类触发位置；不保存产生判断的逐字文本或模型正文。
- 保存 `card_notes.note_type = 'stuck_point'` 的简短卡点摘要，只含题目、`solution_node_id` 和卡住的含义，不含用户逐字话语。
- 保存答案展示时间、十题组/题内位置、首轮或强化标记、调度前后阶段和下一日期，以解释本轮为何推进或重置。
- 不长期保存音频、partial/final 逐字稿、ASR 原始响应、LLM 逐句请求/响应、提示正文或短暂表情状态。

### 6.4 标准答案与步骤来源

1. 标准答案是阶段二必填项。默认先让用户自己给出正确思路；打字输入记为 `source_type = 'user_text'`，语音输入记为 `source_type = 'user_voice'`，语音仍经流式 ASR，转成结构化步骤后立即清理音频和逐字稿。
2. 用户不会时，允许 `source_type = 'ai_generated'`。VLM 可基于题干图生成步骤草稿；这是用户主动触发的解题辅助，不是题干 OCR，也不能将生成文本当作题干原文。
3. 无论来源如何，保存前都必须由用户确认，并通过带 `schema_version` 的 Pydantic v2 schema 写入 `solution_graphs.graph_payload`。同时支持 `linear` 和 `branching`，分支条件必须完整覆盖等号边界；确认前阶段二完成接口必须返回校验错误。
4. 复习时 LLM 只把用户的句子级 `final` 转写与这份已确认步骤对比。标准答案默认隐藏，仅用户主动点“不会”才整题展示；不从拍摄题图中提取或保存“答案原件”，也不为标准答案创建图片资产。

### 6.5 小菊客户端状态机

客户端只使用以下六个稳定状态枚举，素材名、语义和触发事件一一对应：

| 枚举 | 中文状态 | 客户端事件映射 |
|---|---|---|
| `greeting` | 打招呼 | 主页进入、复习完成后回主页 |
| `listening` | 倾听中 | 语音流成功打开、持续接收 PCM 帧 |
| `thinking` | 思考中 | 结束一句口述后等待 final 评估 |
| `correct` | 答对啦 | `on_track` 或自评档位 `1/2` 的短暂反馈 |
| `stuck` | 卡住了 | `off_track`、`stuck_point_recorded` 或用户点“不会” |
| `encouraging` | 别灰心 | 自评档位 `3/4` 与组内强化开始 |

状态机只控制静态素材、点头与文字泡，不携带或推断调度结果。服务端 AI 事件只可触发 `thinking/stuck/correct` 的临时展示；最终自评事件可覆盖临时状态，且任何状态都不得展示分级答案提示。

## 7. 能力端口契约

端口 DTO 都是不可变 Pydantic v2 model；下列四个窄端口位于 `app/capabilities/ports.py`。业务层只依赖 `OCRPort`、`ASRPort`、`VLMPort` 和 `ObjectStoragePort`，第一版注入 Mock，以后在组合根替换供应商适配器，不修改业务代码。`LocalImage` 指向 Worker 私有临时文件，避免把上限图片整体复制成 Python `bytes`。题目候选框检测由服务端内部 Python `RegionDetector` 执行，不放到客户端，也不复用 OCR 或 VLM 端口。

```python
from __future__ import annotations

from pathlib import Path
from types import TracebackType
from typing import Protocol, Self


class OCRPort(Protocol):
    async def recognize_annotation(
        self,
        *,
        image: LocalImage,
        request: AnnotationOCRRequest,
        idempotency_key: str,
    ) -> AnnotationOCRResult: ...


class ASRStream(Protocol):
    async def send(self, chunk: PCM16AudioChunk) -> tuple[ASREvent, ...]: ...
    async def finish(self) -> ASRFinalResult: ...
    async def close(self) -> None: ...
    async def __aenter__(self) -> Self: ...
    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        tb: TracebackType | None,
    ) -> None: ...


class ASRPort(Protocol):
    async def open_stream(
        self,
        *,
        config: ASRConfig,
        correlation_id: str,
    ) -> ASRStream: ...


class VLMPort(Protocol):
    async def classify_annotation(
        self,
        *,
        request: AnnotationClassificationRequest,
        idempotency_key: str,
    ) -> AnnotationClassificationResult: ...

    async def generate_solution_graph(
        self,
        *,
        image: LocalImage,
        request: SolutionGenerationRequest,
        idempotency_key: str,
    ) -> SolutionGraphV1: ...

    async def assess_review_sentence(
        self,
        *,
        request: ReviewSentenceAssessmentRequest,
        correlation_id: str,
    ) -> ReviewSentenceAssessmentResult: ...


class ObjectStoragePort(Protocol):
    async def create_upload(
        self,
        *,
        request: CreateUploadRequest,
    ) -> PresignedUpload: ...

    async def head_object(self, *, object_key: str) -> ObjectMetadata: ...

    async def download_to(self, *, object_key: str, destination: Path) -> None: ...

    async def upload_file(
        self,
        *,
        object_key: str,
        source: Path,
        content_type: str,
        metadata: dict[str, str],
    ) -> ObjectMetadata: ...

    async def create_download(
        self,
        *,
        object_key: str,
        expires_in_seconds: int,
    ) -> PresignedDownload: ...

    async def delete_object(self, *, object_key: str) -> DeleteObjectResult: ...
```

`AnnotationOCRResult.lines` 的每个 `AnnotationOCRLine` 必须包含 `text`、原图或批注小图坐标系中的 `bbox`、`confidence` 和 `handwritten`。真实 OCR 适配器必须使用带行级手写标记的供应商能力：TextIn 映射 `lines[].handwritten`，百度映射 `words_type`，阿里映射 `recClassify=2`。适配器同时保留供应商原文，并按统一阈值为 API 生成 `is_low_confidence`；业务层不从 VLM 回填原文。

`AnnotationClassificationRequest` 只含 `annotation_id`、已归属的 `problem_region_id`、OCR `raw_text`/行证据和用户已校对文本，不包含用来让 VLM 重做原文转写的批注图。`VLMPort.classify_annotation` 在 MVP 只返回批注类型候选，禁止返回学科或知识点。`generate_solution_graph` 是显式例外：用户在阶段二不会录入正确思路时可基于题干图生成标准步骤草稿，且必须经用户确认后才能完成阶段二。

`ASRConfig` 固定为 `sample_rate_hz = 16000`、`signed_16_bit = true`、`channels = 1`、`interim_results = true`。`ASREvent` 明确区分 `partial` 和 `final`；只有 `final` 文本可构造 `ReviewSentenceAssessmentRequest`，`partial` 只进入不持久的趋势检测器。

Mock 适配器统一遵守以下约定：

- 不访问网络、不读取环境中的供应商密钥、不产生费用，也不在调用结束后保留输入。
- 服务端 `RegionDetector` 按整页图片 SHA-256 查找候选框 fixture，且始终在服务端运行。
- OCR 按 `annotation_crop` SHA-256 查找 `tests/fixtures/ocr/manifest.json`，返回固定批注原文及带 `bbox`、`confidence`、`handwritten` 的行数组；未知 hash 返回稳定 `fixture_not_found`。OCR Mock 不接收题干图。
- ASR 按测试发送的帧序号返回固定 `partial`/句子级 `final` 事件；`close()` 可重复调用，并立即清空帧与 transcript 内存。
- VLM 按 `fixture_id` 返回批注类型候选、已通过 Pydantic 的 `SolutionGraphV1` 或句子级 review 判断；批注分类 fixture 必须证明其输入来自 OCR 证据而非原图转写，且 schema 拒绝学科/知识点输出。句子判断只返回 `on_track`/`off_track`、`solution_node_id` 和表情状态，`off_track` 的用户文案由业务层固定为“再想想”。
- ObjectStorage 使用每个测试独享的临时目录模拟私有 bucket，拒绝 `..` 与绝对路径，签名 URL 只生成测试 token；删除不存在对象返回幂等成功。
- 每个 Mock 都报告 `provider = "mock"`、固定 `model` 版本和 `cost_amount = 0`，使 capability 审计链在不接供应商时已可验证。

## 8. `ai_jobs` 任务状态机

### 8.1 状态与流转

```text
queued ──领取──> running ──成功──> succeeded
                    │
                    ├─可重试且未耗尽──> retry_wait ──到时领取──> running
                    ├─永久错误或重试耗尽──> failed
                    └─租约过期──> retry_wait / failed

queued ──目标取消──> cancelled
retry_wait ──目标取消──> cancelled
```

- `queued`：业务事务已提交任务，且 `available_at <= now()` 后可领取。
- `running`：Worker 在同一事务用 `FOR UPDATE SKIP LOCKED` 领取一条任务、递增 `attempt_count` 并写入租约。
- `retry_wait`：可重试失败已记录，等待新的 `available_at`。
- `succeeded`：业务结果已幂等落库，任务成功终止。
- `failed`：错误不可重试或 `attempt_count = max_attempts`，必须保留稳定错误码。
- `cancelled`：目标 ingestion 已由用户取消，且任务尚未成功。

### 8.2 幂等、租约与重试

- 幂等键由业务动作稳定构造，例如 `run_annotation_ocr:{annotation_id}:{revision}`、`render_region:{problem_region_id}:{coordinates_hash}`、`purge_asset:{asset_id}`；`UNIQUE (job_type, idempotency_key)` 拒绝重复排队。
- 区域框用 `boundary_revision`、批注框及归属用 `assignment_revision`、OCR/分类/裁剪等处理结果用各自单调 `result_revision`；任务创建时把预期修订写入经校验的 payload。handler 落库前必须 compare-and-set 目标修订，旧修订结果直接作废并以 `cancelled`/`stale_revision` 终止，绝不覆盖新修订。
- handler 写业务结果时以目标表唯一约束再次兜底，compare-and-set、结果写入和任务成功/作废标记位于同一数据库事务；对象存储操作使用含目标修订的确定性 object key，旧修订对象不得被设为当前引用。
- 单个 Worker 默认租约 `60 s`，每 `20 s` 心跳续租；完成后清空 `leased_by` 与 `lease_expires_at`。
- 恢复器把过期 `running` 任务转为 `retry_wait`；若已经耗尽次数则直接转 `failed`。
- 三次尝试的延迟分别为约 `5 s`、`30 s`、`120 s`，加入不超过 `20%` 的确定性 jitter；永久校验错误、对象越权和 fixture 缺失不重试。
- Worker 只并发执行一个图片或 AI 任务，以控制 `4 GB` 服务器内存；领取逻辑仍保留租约，保证进程崩溃可恢复。
- `failed`、`succeeded`、`cancelled` 都是终态；人工重跑必须创建新 idempotency key，并在 payload 中记录原任务 ID，不得把终态行改回 `queued`。

## 9. API 端点清单

所有端点以认证用户为所有权边界，返回的资产地址是最长十分钟有效的私有签名 URL。HTTP 请求与响应由 FastAPI Pydantic model 产生 OpenAPI，再生成 Dart 客户端；WebSocket 事件使用同名 Pydantic model 生成 JSON schema 并由 Dart 手写一层稳定的 stream wrapper。

| 方法 | 路径 | 请求模型 | 响应模型 | 用途 |
|---|---|---|---|---|
| `POST` | `/v1/personal-access-tokens` | `CreatePersonalAccessTokenRequest` | `CreatedPersonalAccessTokenResponse` | 用交互式身份令牌创建 PAT；明文只在该次 `201` 响应中展示。 |
| `GET` | `/v1/personal-access-tokens` | 无 | `PersonalAccessTokenListResponse` | 列出当前用户 PAT 的名称、scope、使用时间和吊销时间，不返回明文或 hash。 |
| `DELETE` | `/v1/personal-access-tokens/{token_id}` | 无 | `PersonalAccessTokenResponse` | 吊销当前用户的 PAT；跨 owner ID 按不存在处理。 |
| `POST` | `/mcp` | MCP Streamable HTTP JSON-RPC | MCP Streamable HTTP JSON-RPC | 用 Bearer PAT 认证的只读 MCP 端点，提供 `list_problems`、`get_problem`、`list_annotations`、`get_review_stats`、`get_dashboard_summary` 五个 tool。 |
| `GET` | `/v1/export` | `job_id` 可选查询参数 | `ExportResponse` | 无 `job_id` 时返回 `202` 并触发 `export_data` 任务；轮询成功后返回十分钟有效的 ZIP 签名下载地址。 |
| `POST` | `/v1/capture-sessions` | `CreateCaptureSessionRequest` | `CaptureSessionResponse` | 创建一次批量连拍会话。 |
| `GET` | `/v1/capture-sessions/pending-summary` | 无 | `PendingCaptureSummaryResponse` | 返回未完成第二阶段的会话数、题数、原图数与总 `byte_size`，用于界面展示待处理数量和存储占用。 |
| `GET` | `/v1/capture-sessions/{session_id}` | 无 | `CaptureSessionDetailResponse` | 查询连拍照片、候选框、已加入题和第二阶段进度。 |
| `POST` | `/v1/capture-sessions/{session_id}/ingestions` | `CreateIngestionRequest` | `CreateIngestionResponse` | 为连拍中的一张照片创建 ingestion、`original_photo` 和一次性签名上传 URL。 |
| `POST` | `/v1/ingestions/{ingestion_id}/upload-complete` | `CompleteUploadRequest` | `IngestionResponse` | 校验 COS 元数据与 hash，并幂等排入区域检测。 |
| `GET` | `/v1/ingestions/{ingestion_id}` | 无 | `IngestionResponse` | 查询照片上传、服务端候选检测、任务错误和短期原图 URL。 |
| `PUT` | `/v1/ingestions/{ingestion_id}/region-selections` | `SaveRegionSelectionsRequest` | `RegionSelectionsResponse` | 仅在第一阶段保存用户点加号加入、忽略的系统候选框以及用户新建框；此接口不要求精调边界，会话封口后调用返回冲突。 |
| `POST` | `/v1/capture-sessions/{session_id}/finish-capture` | `FinishCaptureRequest` | `CaptureSessionResponse` | 结束第一阶段并永久封口，转入 `awaiting_phase_two`；同一会话不能重开或从其原图继续加题。 |
| `POST` | `/v1/capture-sessions/{session_id}/discard` | `DiscardCaptureSessionRequest` | `DiscardCaptureSessionResponse` | 由用户明确主动丢弃未完成会话，写 `session_discarded` 后排入其原图清理；系统不自动调用。 |
| `GET` | `/v1/capture-sessions/{session_id}/phase-two-items` | `PhaseTwoItemsQuery` | `PhaseTwoItemsResponse` | 按题列出待调边界、核对批注和录入标准答案的第二阶段队列；不返回标签任务。 |
| `GET` | `/v1/problem-regions/{region_id}/phase-two` | 无 | `ProblemPhaseTwoResponse` | 返回题干图、系统预填边界、批注区域/归属、OCR 行与 `is_low_confidence`、VLM 批注类型候选和标准答案状态；不返回学科、知识点或个人标记。 |
| `PUT` | `/v1/problem-regions/{region_id}/boundary` | `ConfirmProblemBoundaryRequest` | `ProblemRegionResponse` | 确认系统预填边界，或仅在不对时提交调整后坐标与 `display_mode`。 |
| `PUT` | `/v1/problem-annotations/{annotation_id}` | `ConfirmAnnotationRequest` | `ProblemAnnotationResponse` | 核对批注文字，必要时修改系统预填的题目归属和批注类型；保留 OCR 原文和 `annotation_crop`，不接收知识点或标签。 |
| `POST` | `/v1/problem-regions/{region_id}/phase-two-completion` | `CompleteProblemPhaseTwoRequest` | `ProblemResponse` | 仅在边界、批注及标准答案都已确认时完成单题第二阶段；可选提交 `math/physics/chemistry/biology` 单选 `subject`；若该原图阶段一已加入题均完成，在同一事务解除原图保护并排入删除。 |
| `POST` | `/v1/problems/{problem_id}/solution-graph/drafts` | `CreateSolutionGraphDraftRequest` | `SolutionGraphResponse` | 用户打字正确思路，或在明确 `user_cannot_solve = true` 时请求 AI 基于题干图生成步骤草稿。 |
| `POST` | `/v1/problems/{problem_id}/solution-voice-sessions` | `StartSolutionVoiceSessionRequest` | `SolutionVoiceSessionResponse` | 创建用户口述正确思路的临时流式 ASR 会话。 |
| `WS` | `/v1/solution-voice-sessions/{session_id}/speech` | `PCM16AudioChunk` | `ASREvent` | 流式转写用户自己的正确思路；音频和逐字稿只作临时输入。 |
| `POST` | `/v1/solution-voice-sessions/{session_id}/finish` | `FinishSolutionVoiceRequest` | `SolutionGraphResponse` | 完成 `source_type = 'user_voice'` 草稿并立即清理音频/逐字稿。 |
| `GET` | `/v1/problems/{problem_id}/solution-graph` | 无 | `SolutionGraphResponse` | 获取经 Pydantic 校验的步骤图草稿或确认版本。 |
| `PUT` | `/v1/problems/{problem_id}/solution-graph/confirmation` | `ConfirmSolutionGraphRequest` | `SolutionGraphResponse` | 用户修改并确认 `linear` 或 `branching` 关键步骤图。 |
| `POST` | `/v1/problems/{problem_id}/cards` | `CreateMistakeCardRequest` | `MistakeCardResponse` | 在 `solution_graphs` 确认后创建私人错题卡、`initial_thought`/`error_reason`/`next_caution`/`annotation` 备注和第一轮调度。 |
| `GET` | `/v1/cards/{card_id}` | 无 | `MistakeCardDetailResponse` | 获取题干图、默认收起的批注信息栏、`card_notes`、已确认步骤和当前调度；复习主区不展开批注。 |
| `GET` | `/v1/reviews/due` | `DueReviewsQuery` | `DueReviewsResponse` | 列出 `next_review_on` 不晚于用户当前日期的活动卡片。 |
| `POST` | `/v1/review-groups` | `CreateReviewGroupRequest` | `ReviewGroupResponse` | 从到期队列冻结一组 `10` 题及顺序，首轮结束后返回本组需立即强化的错题。 |
| `GET` | `/v1/review-groups/{group_id}` | 无 | `ReviewGroupResponse` | 返回组进度、`remaining_today`，并为每个 card 返回 `stage_index` 与 `schedule_revision`，供复习页顶部展示。 |
| `POST` | `/v1/cards/{card_id}/review-sessions` | `StartReviewSessionRequest` | `ReviewSessionResponse` | 在十题组内创建唯一开放复习会话；请求明确 `group_position` 与首轮/强化类型，强化会话不锁定或修改调度修订。 |
| `WS` | `/v1/review-sessions/{session_id}/speech` | `PCM16AudioChunk` | `ReviewStreamEvent` | 流式发送 `16 kHz`/PCM16 音频；接收 `partial_trend`、`sentence_assessment`、`stuck_point_recorded`、表情和清理状态。`off_track` 事件文案固定为“再想想”，不含提示内容。 |
| `POST` | `/v1/review-sessions/{session_id}/finish-speech` | `FinishSpeechRequest` | `ReviewSpeechFinishedResponse` | 调用 ASR `finish()`，处理最后一句，立即清除音频与逐字稿 ephemera，并返回已记录卡点 ID。 |
| `POST` | `/v1/review-sessions/{session_id}/reveal-answer` | `RevealAnswerRequest` | `RevealAnswerResponse` | 仅响应用户主动点“不会”；先记录题内事件序号/步骤位置，再返回整题已确认 `solution_graphs`，不返回拍摄的答案原件。 |
| `PUT` | `/v1/review-sessions/{session_id}/confirmation` | `ConfirmReviewRequest` | `ConfirmReviewResponse` | 保存四档 `self_rating`，按 `1/2 → correct、3/4 → incorrect` 派生调度结果；首轮在同一事务推进或重置调度，强化只记录结果而不碰调度。逐字稿不保存，AI 方向判断仅以无正文事件保存。 |
| `GET` | `/v1/dashboard/summary` | `DashboardSummaryQuery` | `DashboardSummaryResponse` | 以用户本地 `current_date` 聚合连续复习天数、当周七天点阵、本周/上周题数与时长、当前到期题数量；`subject_accuracy` 只统计已选学科题目的首轮最终自评正确率。 |

`SaveRegionSelectionsRequest`、`ConfirmProblemBoundaryRequest` 和 `ConfirmAnnotationRequest` 都必须提交 EXIF 已规范化原图尺寸、客户端预览尺寸与当前坐标修订号；服务端只以规范化原图像素为权威换算并验证每个框完全位于图内，题框不小于 `64 × 64 px`。区域框、批注归属和处理结果分别携带其单调修订号，写入前 compare-and-set；修订不匹配返回 `409 stale_revision`，旧结果不得覆盖。

`CompleteProblemPhaseTwoRequest` 必须先验证唯一 `solution_graphs.status = 'confirmed'`，再在锁中重新计数同张原图阶段一封口前的已加入/已完成题，不信任客户端传入的“全部完成”布尔值。`ConfirmReviewRequest` 的首轮请求携带启动会话时取得的 `schedule_revision`；若修订号已变化则返回 `409 schedule_conflict`，不得二次推进，强化请求不更新调度。

PAT 与交互式身份 token 共存，但 PAT 只能调用明确声明 scope 的只读能力；现有写端点一律拒绝 PAT。MCP 工具不接收 owner 参数，而是从通过官方 SDK 验证的 PAT 上下文取得 `user_id`，并复用 application service 的 owner 隔离。PAT 调用成功、业务失败和 scope 越权均写结构化审计事件。全量导出的公开格式见 [`Docs/export-format.md`](../export-format.md)；压缩包只含用户自有结构化 JSON 和资产签名 URL 清单，不内嵌图片二进制。

## 10. 复习调度算法

### 10.1 十题组内强化

MVP 每次从到期队列冻结 `10` 题及其 `group_position`。先依次完成十题首轮；首轮最后一题确认后，收集本组 `final_outcome = 'incorrect'` 的题，并立即按原组顺序各创建一次 `attempt_kind = 'reinforcement'` 的重做会话。没有错题则直接完成本组。

组内强化与固定调度是两层独立机制：首轮确认照常推进或重置 `review_schedules`；强化会话只记录当次结果并完成 `review_groups`，既不再次推进，也不撤销首轮调度。因此某题首轮做错后已回到第一轮，即使组内重做答对，也仍按次日第一轮复习。

```text
function finish_initial_pass(group):
    wrong_sessions = initial sessions in group where final_outcome == "incorrect"
    group.initial_pass_completed_at = now
    if wrong_sessions is empty:
        group.state = "completed"
        group.completed_at = now
    else:
        group.state = "reinforcing"
        create one reinforcement session for each wrong session in group_position order

function confirm_reinforcement(session, outcome, now):
    assert session.attempt_kind == "reinforcement"
    persist session.final_outcome = outcome and complete session
    do not read or write review_schedules
    if all reinforcement sessions in group are completed:
        group.state = "completed"
        group.completed_at = now
```

### 10.2 固定间隔调度

定义 `INTERVAL_DAYS = [1, 3, 7, 14, 30]`，`stage_index` 表示当前正在完成哪一个间隔轮次。建卡当天不立即算完成第一轮，而是安排第一轮在一天后。

```text
function create_schedule(card, today):
    card.status = "active"
    schedule.stage_index = 0
    schedule.next_review_on = today + 1 day
    schedule.last_outcome = null
    schedule.sequence_completed_at = null
    schedule.revision = 0

function confirm_review(session_id, expected_revision, outcome, today, now):
    begin transaction
    session = select review_session for update
    schedule = select review_schedule for update
    card = select mistake_card for update

    if session.state == "completed":
        return previously persisted result
    if session.state != "awaiting_confirmation":
        raise invalid_session_state
    if session.attempt_kind != "initial":
        raise reinforcement_does_not_update_schedule
    if schedule.revision != expected_revision:
        raise schedule_conflict
    if session.ephemera_purged_at is null:
        raise speech_ephemera_not_purged

    before = schedule.stage_index

    if outcome == "incorrect":
        after = 0
        schedule.stage_index = 0
        schedule.next_review_on = today + 1 day
        schedule.last_outcome = "incorrect"
        schedule.sequence_completed_at = null
        card.status = "active"
    else if before < 4:
        after = before + 1
        schedule.stage_index = after
        schedule.next_review_on = today + INTERVAL_DAYS[after] days
        schedule.last_outcome = "correct"
        schedule.sequence_completed_at = null
        card.status = "active"
    else:
        after = 4
        schedule.stage_index = 4
        schedule.next_review_on = null
        schedule.last_outcome = "correct"
        schedule.sequence_completed_at = now
        card.status = "sequence_completed"

    schedule.revision += 1
    session.final_outcome = outcome
    session.schedule_stage_before = before
    session.schedule_stage_after = after
    session.next_review_on = schedule.next_review_on
    session.confirmed_at = now
    session.ended_at = now
    session.state = "completed"
    commit transaction
```

“做错重置”不保留先前阶段作为下一轮起点；它总是回到 `stage_index = 0` 并在次日复习，组内强化答对也不能覆盖该结果。完成 30 天轮次只表示固定序列完成，状态名必须是 `sequence_completed`，不得写作 `mastered`；模式掌握等待以后辨识题链路验证。

## 11. 测试策略

### 11.1 RED → GREEN 推进顺序

每个行为先写一个最小失败测试并运行到预期失败，再写最小实现通过，最后只在全部相关测试为 GREEN 时重构。证据以 `tests/evidence/<work-block>/<timestamp>-red.txt` 和 `...-green.txt` 保存；RED 文件必须包含失败测试名与符合预期的失败原因，GREEN 文件必须包含同一测试及其相邻回归测试的通过摘要。

1. 先为 Pydantic schema 和纯状态机写单元 RED：关键步骤分支边界、图片限制、EXIF/预览/检测坐标换算、原图删除保护、修订 CAS、任务状态机、十题组内强化与复习推进/重置。
2. 再为 Alembic 和 repository 写 PostgreSQL 集成 RED：UUIDv7 默认值、标签保留字段强制留空、JSONB round-trip、唯一约束、租约领取、旧修订作废、原图 `NOT EXISTS` 锁定核验、十题组与并发确认。
3. 再为 OCR、ASR、VLM、ObjectStorage 四个 Mock 端口写契约 RED：确定性结果、超时/错误映射、幂等关闭、零留存和零成本审计。
4. 再按 API 顺序写 FastAPI RED：连拍会话、上传、服务端候选检测、第一阶段加题/封口、第二阶段边界/批注 OCR/强制标准答案、建卡、十题分组、语音结束、“不会”揭示和用户最终确认。
5. 后端闭环为 GREEN 后导出 OpenAPI，用固定容器的 OpenAPI Generator `dart-dio` target 生成 Dart 客户端，并用 contract test 拒绝未提交的生成差异。
6. Flutter 先写 feature controller/widget RED，再实现相机替身输入、连拍加题/封口、第二阶段边界/批注 OCR/标准答案表单、无画板的流式口述、固定“再想想”、“不会”揭示、十题组与最终确认。
7. 最后在 Android 模拟器和 Mock 后端运行端到端 RED/GREEN，验证会话封口后同图不能加题，一张双题图片的两题都完成含标准答案的第二阶段前原图不删，并完成十题首轮、错题重做与独立调度。

### 11.2 测试层级

- 单元测试：`SolutionGraphV1` 图约束、`advance_schedule`、十题组内强化、job backoff、权威坐标换算、修订 CAS、文件体积预算、日志字段过滤和状态转换。
- PostgreSQL 集成测试：全部 DDL check/unique/FK、Alembic 升降级、`FOR UPDATE SKIP LOCKED`、租约恢复、旧修订落库拒绝、原图 `NOT EXISTS` 核验、确认接口幂等和事务回滚。
- 端口契约测试：同一测试套件运行 Mock，并为以后真实适配器保留标记；业务测试只 mock Protocol，不 mock repository 或状态机。
- API 集成测试：使用真实 FastAPI、PostgreSQL 和文件 ObjectStorage Mock；只在四个能力边界使用 Mock。
- 隐私测试：扫描数据库 dump、COS Mock 目录、结构化日志和 `ai_jobs.payload`，断言测试音频字节与唯一 transcript canary 均不存在。
- 图片压力测试：用 `12 MP / 10 MiB` fixture 记录峰值 RSS、临时目录峰值和各阶段对象大小，断言单 Worker 新增内存不超过 `96 MiB`。
- Flutter 单元与 widget 测试：权限拒绝、批量上传恢复、封口后加题拒绝、第二阶段边界、批注归属与低置信度行、无标签/无画板、默认收起信息栏、`off_track` 只显示“再想想”、“不会”才揭示答案、十题组错题重做、WebSocket 断线、AI 与用户结果冲突。
- Android 端到端测试：Mock 相机连拍 → 服务端候选框 → 点加号并封口离开 → 逐题边界/批注 OCR → 用户正确思路/步骤确认 → 阶段二完成 → 建卡 → 十题到期组 → 流式 ASR Mock → 句子精确对比 → 用户确认 → 组内错题重做 → 验证首轮产生的新日期不变。
- 离线评测：至少包含数学线性题、数学三分支且含 `=` 的题、物理几何图题、化学方程式题、生物长题干和多题照片；供应商后置阶段只替换适配器，不改业务测试。

### 11.3 必须覆盖的失败路径

- 上传 hash 不符、超限图片、压缩炸弹、EXIF/预览尺寸不一致、越界候选框、旧坐标修订和全部区域被拒绝；阶段一封口后的新增/恢复/重开请求均被拒绝。
- 同图尚有未完成第二阶段的题却被请求自动清理、`purge_asset` 重试时 `NOT EXISTS` 复核失败、COS 删除瞬时失败和“对象已不存在”的幂等恢复。
- 批注 OCR fixture 缺失、缺少行级 `handwritten`/坐标/置信度、VLM 尝试覆盖原文或返回知识点、旧归属/分类修订尝试覆盖、用户重复确认两个 OCR revision。
- solution graph 节点悬空、循环、分支缺等号边界、`schema_version` 不支持和数据库脏 JSONB 回读失败。
- Worker 在结果写入前后崩溃、租约过期、可重试错误耗尽和永久错误立即失败。
- 没有已确认标准答案却请求完成阶段二、AI 标准步骤未经用户确认、保留标签字段被写入非空值。
- ASR WebSocket 断线、`finish()` 重放、会话空闲超时、服务端取消和 transcript canary 泄漏扫描；`off_track` 响应夹带提示内容、非用户主动触发答案揭示或漏记判断/“不会”位置都必须失败。
- AI 建议 `on_track` 而用户确认 `incorrect` 时必须按用户结果重置；两个客户端同时确认时只能一个事务推进。组内强化答对仍不得改写首轮重置，强化确认不得更新 `schedule_revision`。

## 12. 实施顺序

| 工作块 | 前置依赖 | RED 重点 | 完成标志 |
|---|---|---|---|
| 0. 工具链与骨架 | 本计划与 Flutter ADR 已接受 | 版本检查失败、应用健康检查和 OpenAPI 导出测试先失败 | Python `3.13.13`、uv `0.11.14`、Flutter Android 骨架、PostgreSQL `18.4` 测试服务可运行，仓库无 Node/pnpm 文件 |
| 1. Schema 与迁移 | 工作块 0 | 空库迁移、全部约束、UUIDv7、标签字段留空、修订号、十题组/事件、JSONB Pydantic round-trip | Alembic `upgrade head`/`downgrade base` 集成测试通过，应用无 `create_all` |
| 2. `ai_jobs` 与四个 Mock 端口 | 工作块 1 | 状态流转、租约恢复、幂等键、旧修订 CAS 作废、端口契约与零成本记录 | 单 Worker 能确定性完成/重试/作废/失败 fixture 任务，四端口契约测试通过 |
| 3. 资产上传与原图保护 | 工作块 1、2 的 ObjectStorage Mock | EXIF 权威坐标、超限拒绝、hash 校验、阶段一封口、同图多题未全完成时 `NOT EXISTS` 删除拒绝、用户主动丢弃 | Android 连拍可私有上传，封口后不能同图加题，未完成原图的事务与服务双重删除测试通过，待处理数与存储占用可查 |
| 4. 两阶段图片、切题与批注 OCR | 工作块 2、3 | 服务端候选框、先归一化后二值化、保细节图、默认一区一块、批注归属修订、行级手写/置信度、`annotation_crop` | 双题 fixture 可拍完离开后逐题确认，低置信度行可见，VLM 只基于 OCR 预填批注类型，临时文件清零 |
| 5. 题干图与阶段二关键步骤图 | 工作块 1、2、4 | 题干无 OCR、无标签写入、用户打字/语音优先、AI 兜底、linear/branching、等号边界、schema 版本、阶段二必填 | 三种 `source_type` 和两类步骤图均经 Pydantic 写入/读出；只有用户确认标准答案后题目才完成阶段二并变为 `ready` |
| 6. 错题卡、十题组与固定调度 | 工作块 5 | 建卡首日、十题冻结、组内错题重做、五轮推进、任意轮错误重置、强化不改调度、完成态命名 | 纯函数和 PostgreSQL 并发测试通过，卡片可列入 due 列表并完成两层复习机制 |
| 7. 流式语音复习与隐私清理 | 工作块 2、5、6 | PCM16 WebSocket、partial 仅趋势、final 精确对比、固定“再想想”、“不会”才揭示、判断位置事件、即时 `stuck_point`、ephemera 清理 | 用户能完成无画板的 ASR/VLM Mock 口述复习，判断/“不会”位置和卡点可审计，canary 在 DB/COS/log/job 中均不存在 |
| 8. Flutter 完整流程 | 工作块 3–7、Dart 客户端生成 | 权限、断网、封口、候选框、校对、无标签/无画板、十题组、答案揭示和修订冲突提示 | Android 模拟器从相机 fixture 走到十题首轮、组内强化与调度响应，全程使用生成 Dart 客户端 |
| 9. 闭环验证与重构 | 工作块 0–8 | 完整 E2E、迁移重放、峰值内存、失败恢复和 OpenAPI diff | 第 1.3 节全部勾选，最新测试证据保存；完成独立代码评审，因涉及未成年人数据、外部存储与权限再完成独立安全评审 |

每个工作块独立完成 RED、GREEN、REFACTOR 和证据归档后才进入下一个工作块。未获得用户明确授权前不安装依赖、不连接真实 COS 或 AI 供应商、不提交、不推送；Mock 与离线样本足以完成工作块 0–9 的核心验收。
