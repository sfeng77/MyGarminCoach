# MyGarminCoach – Implementation Plan (Python / Local Deploy / Official Garmin APIs)

目标：实现一个本地部署的 “Garmin Coach” 系统，支持至少 2 个用户，使用 **官方 Garmin 数据接口**同步 activities/health/training 记录到本地存储；每天定时生成滚动训练计划（当前聚焦：**隔天一次室内骑行 + 室内无器械体能/力量**，每次 ≤ 60min）；在需要恢复时建议取消训练；并将训练计划导出为可在手表侧执行的内容（MVP：ICS 日历 + 训练说明；增强：Workout 文件，如 FIT，取决于设备/导入链路可用性）。

> 重要约束：在“只用官方接口”的前提下，通常无法做到 100% 自动“下发 workout 到 Connect/手表”。因此本计划将“导出/导入”作为 V1 可交付路径，并把“自动下发”留作可插拔扩展（未来可选非官方 Connect 接口或其他方式）。

---

## 1) 非功能性要求（必须达标）

- **多用户隔离**：任意数据查询/同步/导出都必须以 `user_id` 作用域隔离。
- **幂等与可重跑**：同步、计划生成、导出都必须可重复执行且结果可追溯。
- **本地部署**：`docker compose up` 一键启动（API + Worker + DB）。
- **可观测性**：所有后台 job 写 `job_runs`，失败可定位、可重试。
- **安全**：Garmin 凭据/refresh token 等敏感信息必须加密存储；日志不得泄露敏感字段。

---

## 2) 推荐技术栈（Python）

- API：FastAPI
- DB：Postgres
- ORM/Migrations：SQLAlchemy + Alembic
- Worker/Jobs：
  - MVP：APScheduler（单 worker 进程定时任务，最简单）
  - 进阶：Celery + Redis（需要水平扩展或更复杂队列时再上）
- 加密：`cryptography`（Fernet / AES-GCM 封装）
- 导出：
  - ICS：`icalendar`
  - Markdown：标准库即可
  - FIT（增强）：先预留接口与测试样例，等你确认真实导入链路后再实现
- 质量：
  - 测试：pytest
  - 格式/静态检查：ruff（可选但建议）

---

## 3) 仓库结构（coding agent 按此创建）

建议采用单仓库 + 多应用目录：

- `apps/api/`：HTTP API（Auth、数据查询、计划、导出、手动触发 job）
- `apps/worker/`：定时任务入口（sync、plan、export）
- `packages/core/`：领域模型与规则引擎（训练模板 DSL、计划生成、diff）
- `packages/garmin/`：官方 API client + 数据映射（raw -> normalized）
- `packages/storage/`：DB session、repo 层、加密工具
- `packages/exporter/`：ICS/Markdown/FIT 导出
- `docs/`：部署、手工导入步骤、运维手册
- `docker-compose.yml`：本地一键启动

DoD：
- `apps/api` 与 `apps/worker` 可分别启动；
- `packages/*` 不依赖具体运行时（可单测）。

---

## 4) 数据模型（DB schema / Alembic）

### 4.1 用户与 Garmin 绑定

- `users`
  - `id` (pk)
  - `email`/`username`（唯一）
  - `password_hash`
  - `created_at`
- `garmin_links`
  - `id` (pk)
  - `user_id` (fk)
  - `status`（linked/unlinked/error）
  - `encrypted_credentials`（加密 blob，包含 token/refresh 等）
  - `last_sync_at`
  - `created_at`, `updated_at`

DoD：
- 支持至少 2 个用户分别绑定各自 Garmin；
- 任意用户无法读取其他用户的 `garmin_links`。

### 4.2 骑行心率分区（必须版本化 + 可冻结）

- `hr_zone_sets`
  - `id` (pk)
  - `user_id` (fk)
  - `sport`（固定 `"cycling"`）
  - `effective_from`, `effective_to`（Garmin 分区变更时切版本）
  - `zones_json`（数组：`[{zone_index, low_bpm, high_bpm, label?}, ...]`）
  - `raw_payload_json`（可选，便于追溯）
  - `created_at`

DoD：
- 同步到分区变更时，新建 `hr_zone_sets` 版本，旧版本补 `effective_to`；
- 计划导出时可引用某个 `zone_set_id` 解析并冻结目标 bpm。

### 4.3 活动与健康数据（raw + normalized）

- `activities_raw`
  - `id` (pk)
  - `user_id` (fk)
  - `provider`（"garmin"）
  - `provider_activity_id`（唯一：`(user_id, provider, provider_activity_id)`）
  - `start_time`
  - `payload_json`
  - `payload_hash`
  - `updated_at`
- `activities`
  - `id` (pk)
  - `user_id` (fk)
  - `provider_activity_id`
  - `sport`（cycling/strength/other）
  - `duration_seconds`
  - `avg_hr_bpm`, `max_hr_bpm`（若可得）
  - `distance_m`（骑行可得则存）
  - `created_at`, `updated_at`

- `health_daily`
  - `id` (pk)
  - `user_id` (fk)
  - `date`（唯一：`(user_id, date)`）
  - `sleep_seconds?`, `hrv?`, `resting_hr?`, `stress?`（按官方能拿到的字段逐步补齐）
  - `payload_json`（可选）
  - `updated_at`

DoD：
- sync 可重复执行不重复写；
- raw 与 normalized 可对账（保留 `provider_activity_id` 关联）。

### 4.4 计划、版本、导出、任务日志

- `plan_versions`
  - `id` (pk)
  - `user_id` (fk)
  - `start_date`, `end_date`
  - `constraints_nl`（自然语言约束原文）
  - `reason`（daily_roll/manual/recovery_override）
  - `created_at`
- `plan_items`
  - `id` (pk)
  - `plan_version_id` (fk)
  - `user_id` (fk)
  - `date`
  - `status`（scheduled/optional/cancelled/completed/skipped）
  - `template_id`
  - `params_json`（如：时长、强度缩放、repeat 次数等）
  - `zone_set_id`（引用生成时生效的 `hr_zone_sets.id`）
  - `resolved_targets_json`（导出时冻结：step_path -> bpm_range）
  - `notes`

- `exports`
  - `id` (pk)
  - `user_id` (fk)
  - `plan_version_id` (fk)
  - `kind`（ics/markdown/fit/zip）
  - `path`
  - `content_hash`
  - `created_at`

- `job_runs`
  - `id` (pk)
  - `user_id` (fk, nullable：系统级 job 可为空)
  - `job_name`（sync_user_data/generate_plan/export_plan）
  - `status`（running/success/failed）
  - `started_at`, `finished_at`
  - `error_message`（短文本）
  - `metrics_json`（拉取条数、写入条数、耗时等）

DoD：
- 能通过 `job_runs` 定位失败原因；
- 计划生成与导出均可回溯到具体 `plan_version_id`。

---

## 5) Garmin Connector（官方接口封装与同步）

### 5.1 Client 设计

- `GarminClient`：只做“官方 API 请求 + 认证刷新 + rate limit/backoff”
- `GarminMapper`：把官方返回映射为 internal schema（raw + normalized）

### 5.2 同步 Job：`sync_user_data(user_id)`

按顺序：
1. 同步 `cycling` HR zones（若官方可得）：检测变化 -> `hr_zone_sets` 版本化
2. 同步 activities：首次回填 + 后续增量
3. 同步 `health_daily` / `training_daily`（如官方有）：按日期窗口增量

幂等策略：
- raw：`(user_id, provider, provider_activity_id)` 唯一
- normalized：以 `provider_activity_id` upsert
- health_daily：`(user_id, date)` upsert

Mock/fixtures：
- 提供 `GARMIN_MODE=mock`：使用本地 JSON fixtures，保证 CI 与本地离线可跑通。

DoD：
- 同一用户 sync 连续跑 2 次，第二次写入条数应为 0 或仅更新有变化的记录；
- 两用户并行 sync 不互相影响。

---

## 6) 训练模板 DSL（保证可导出）

### 6.1 核心结构（存 JSON）

- `WorkoutTemplate`
  - `template_id`, `name`
  - `sport`: `"cycling"` 或 `"cardio"`（力量/体能先用 cardio 表达，兼容性更高）
  - `defaults`: `{ total_minutes <= 60, intensity_hint }`
  - `steps`: `WorkoutStep[]`（支持 `repeat`）
- `WorkoutStep`
  - `kind`: warmup/work/recovery/cooldown/rest
  - `duration`: `{ type:"time", seconds }`
  - `target`: `{ type:"hr", mode:"zone"|"open", zone_index? }`
  - `notes`（动作提示/技术要点）
  - `repeat`: `{ count, steps: WorkoutStep[] }`（可选）

### 6.2 模板库（MVP）

骑行（目标只用 zone_index）：
- `bike_endurance_50`：WU 10' Z1–Z2 → 35' Z2 → CD 5' Z1
- `bike_tempo_55`：WU 10' Z1–Z2 → 3×(8' Z3 + 3' Z2) → CD 7' Z1–Z2
- `bike_intervals_50`：WU 10' Z1–Z2 → 6×(2' Z4 + 2' Z2) → 10' Z2 → CD 5' Z1

力量/体能（cardio + notes，目标 open 或最多 Z3）：
- `strength_full_body_45`：热身 → 循环动作（深蹲/俯卧撑/臀桥/核心）+ 休息 → 放松
- `conditioning_mobility_40`：低中强度循环 + 活动度

DoD：
- 模板可被实例化为 `PlanItem`（可调时长/循环次数）；
- 导出层不需要 LLM 才能生成可执行的训练说明。

---

## 7) 计划引擎 v0（规则生成 + 可取消）

### 7.1 目标

- 生成未来 `horizon_days=14` 的滚动计划
- 交替安排：Bike / Strength / Bike / Strength…
- 每 7 天最多 2 次高强度骑行（含 Z4 work 段）
- 每次训练 ≤ 60min
- 当恢复不足时：建议取消（把 `PlanItem.status="cancelled"`，并写明原因与替代建议）

### 7.2 输入与输出

输入：
- 最近 2–6 周活动统计（bike 训练量、强度、完成度）
- `health_daily` 趋势（HRV、睡眠、静息心率，能拿到哪些用哪些）
- 最新的 `hr_zone_set_id`（cycling）
- `constraints_nl`（自然语言只做记录；规则只消费结构化上限：60min/交替/高强度次数）

输出：
- `plan_versions` + `plan_items`
- 计划生成时写入 `plan_items.zone_set_id`（锁定分区版本）

DoD：
- 能生成 14 天交替计划；
- 在“恢复差”的 mock 数据下，能把最近的高强度骑行改为 endurance 或 cancelled。

---

## 8) 导出（MVP：ICS + Markdown；增强：FIT）

### 8.1 MVP 导出：ICS

- 每个 `PlanItem` 生成一个日历事件（标题含：Bike/Strength + 强度 + 时长）
- 描述包含：
  - steps 概要
  - 心率目标（以 zone_index 表达，并可附“解析后的 bpm 范围”）
  - 强度提示与注意事项

### 8.2 冻结心率目标（保证可追溯）

导出时：
- 读取 `plan_items.zone_set_id` 对应的 zones
- 将每个 step 的 `zone_index -> (low_bpm, high_bpm)` 写入 `plan_items.resolved_targets_json`
- 若 zone 数据缺失：降级为 `target=open` 并在 notes 标记

### 8.3 变更同步（导出 diff）

- 计划每天生成新 `plan_version`
- 对比“最新版本”与“上一版本”在未来 7 天的 `PlanItem`（按 date + template_id + params hash）
- 仅对变化的日期重新生成 ICS/Markdown（或生成一个 zip 包）

DoD：
- 连续两天计划完全一致时，导出 hash 不变；
- 只有某天变化时，只更新那天的导出产物/说明（或产出变更清单）。

---

## 9) API（最小可用）

- Auth：
  - `POST /auth/register`
  - `POST /auth/login`
- Garmin：
  - `POST /garmin/link`（绑定）
  - `POST /garmin/unlink`
  - `POST /jobs/sync`（手动触发）
- 数据：
  - `GET /activities?from=&to=`
  - `GET /health/daily?from=&to=`
- 计划：
  - `POST /plan/generate`（手动触发）
  - `GET /plan/latest`
  - `POST /plan/items/{id}/status`（completed/skipped）
- 导出：
  - `POST /exports/latest`（生成并返回下载链接）
  - `GET /exports/{id}/download`

DoD：
- 两用户登录后，接口返回数据互不交叉；
- 所有写操作写入 `job_runs` 或相应审计字段。

---

## 10) Worker（定时任务）

默认调度（本地时区）：
- 每 3 小时：对每个 linked 用户跑 `sync_user_data`
- 每天 03:00：对每个 linked 用户跑 `generate_daily_plan`，随后触发 `export_plan`

DoD：
- worker 重启后能继续按 schedule 运行；
- 单用户失败不影响其他用户 job。

---

## 11) 测试策略（必须覆盖的点）

- 单元测试：
  - 模板实例化（repeat、时长裁剪 ≤ 60min）
  - zone 解析冻结（zone_index -> bpm range）
  - 计划引擎（交替训练、高强度上限、取消逻辑）
- 集成测试（mock garmin）：
  - sync -> plan -> export 全链路
  - 多用户并行（至少 2 用户）

DoD：
- `pytest` 可在无外部网络/无真实 Garmin 凭据情况下跑通（依赖 fixtures）。

---

## 12) 文档（必须写在 docs）

- `docs/deployment.md`：本地启动方式（docker compose）、环境变量（MASTER_KEY 等）
- `docs/garmin_linking.md`：官方接口接入与回调（若需要）
- `docs/export_to_watch.md`：如何把导出产物导入/同步到手表（你实际验证的步骤）
- `docs/ops.md`：常见问题（sync 失败、分区缺失、导出失败）与排障

DoD：
- 新环境按文档 30 分钟内能跑起 demo（mock 模式）。

---

## 13) 分阶段交付（coding agent 可按里程碑提交）

### Milestone A（闭环骨架）

- scaffold + DB migrations + auth + job_runs
- mock garmin + activities sync（raw + normalized）

### Milestone B（健康与分区 + 计划 v0）

- hr zones 版本化同步（cycling 独立）
- health_daily 同步
- 规则计划引擎 + cancelled 支持

### Milestone C（导出可执行）

- ICS + Markdown 导出
- 变更 diff + 导出冻结 targets

### Milestone D（增强）

- FIT 导出（以你验证的设备/导入链路为准）
- 更强的恢复判定与自动调参
- 本地模型用于解释与建议（不参与关键数值）

