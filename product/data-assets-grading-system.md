---
type: concept
---

# 批阅系统数据资产盘点（aireview-grading-system）

> 调研对象：`~/src/indievolve/aireview-grading-system`（main 分支，2026-08）。本文是[记忆系统提案](./memory-system-proposal.md)的前置调研，目标是回答「这个系统里到底有哪些数据、各自长什么样、哪些能被记忆/数据架构复用」。
> 盘点方法：通读 `backend/modules/*/models.py` 全部 24 个域目录（共 64 张表）、`app/infra/oss.py`、`docs/OSS_NAMESPACE.md`、`db/dev_seed.py` 及关键 service 代码；「是否被业务/LLM 复用」一列直接引用 [API 与业务流程](./api-and-process-grading-system.md)（下称「API 文档」）的既有结论，不重复调研。与既有文档不一致处以 ⚠️ 标注。

## 0. 全局事实（先看这部分）

- **存储格局**：结构化数据全部在单库 MySQL（SQLAlchemy 2.0 async + aiomysql）；非结构化文件在共享对象存储桶 `indievolve-grade`（MinIO SDK 走 S3 协议，生产为阿里云 OSS，本地/演示为 MinIO），key 规范见仓库 `docs/OSS_NAMESPACE.md`；Redis 只做缓存/熔断/幂等，不承载业务事实。
- **表结构铁律**（`app/common/base.py`）：所有表由 `IdMixin`（雪花 BIGINT 主键）+ `TenantMixin`（tenant_id 打头索引，0=平台共享）+ `TimestampMixin`（含软删哨兵 deleted_at）组合；不建 DB 外键。只有两类表**刻意不带** TimestampMixin：只追加审计表（`score_audits`、`exam_events`）与平台级全局表（`api_logs`、`feedbacks`、`content_moderation_records`、`llm_*`、`system_configs` 等，无租户行级守卫）。
- **数据量级总体判断**：本机没有可查询的本系统数据库（docker 里唯一的 MySQL 是其他项目的 e2e 容器），量级只能给两个口径：
  - **演示数据**：`backend/db/dev_seed.py`——1 个租户、高三 3 个班、每班约 5 名学生、数场考试与少量掌握度样例行，纯演示级。
  - **生产规模估算**：`docs/TESTPLAN-批阅链路稳定性与并发.md` P2 基线——单场考试 8 班 × 60 人 ≈ **480 卷**，扫描峰值 1 卷/秒，同时段并发 3 场，全天推卷 ≈ **5000 卷**（文档自承是按打印机速度推算、非实测）。按高中单科 15–25 题估算，一场考试产生 `student_answers` 约 0.7–1.2 万行，`score_audits` 视复核量数千行量级。
- **对个人化/记忆系统最重要的一句话**（与 API 文档 §5 结论一致并扩展）：`student_answers` 的 ai_*/manual_* 双列留痕、`score_audits` 只追加审计、`recognized_answer` 作答原文、`ai_feedback` 评语，是沉淀最厚的个体数据，目前**完全不进入本系统任何 LLM prompt**；`knowledge_mastery`/`error_attributions`/`student_tasks` 三张个人化学情表**存在 schema 且被读，但本仓库没有任何写入方**（详见 §1.4 ⚠️）。

## A. 结构化数据盘点

「复用现状」列：✅=已被业务逻辑读取消费；🔶=存在但消费面窄/读时实算；❌=无任何消费方；⚠️=有异常（如无写入方）。LLM 复用细节统一见 API 文档 §3/§5，此处只给结论。

### A.1 组织与人（modules/org、modules/iam）

| 表 | 域 | 关键字段（教学/个人化相关） | 量级 | 复用现状 |
|---|---|---|---|---|
| `students` | org | `student_no`（学籍号，租户内唯一，注释明示「终身锚点」）、`name`、`class_id`（当前班）、`status`（active/transferred/graduated） | 演示 ~15 人；生产一校约 1–3 千人（未实测） | ✅ 全系统人事实体；花名册进入 scan_recognition 的 LLM prompt（学号+姓名） |
| `classes` | org | `grade`（高一/高二/高三）、`name`、`head_teacher_id`（班主任）、`class_type=admin`（行政班）；`academic_year` 已停用（写空串哨兵） | 演示 3 班；生产一校数十班 | ✅ 数据范围/聚合统计的组织单元 |
| `semesters` | org | `name`、`start_date/end_date`、`is_current` | 每校每学期 1 行 | ✅ 考试批次挂靠 |
| `exam_students` | org | **本场应到名单快照**（绑 `exam_batch_id`，导入时冻结 `class_id/class_name/student_name/student_no` 快照列，转班不漂移） | ≈考试批次×应到人数 | ✅ 应到人数唯一定义；扫描认人比对基准 |
| `exam_no_mappings` | org | `exam_batch_id × student_id ↔ exam_no` 考号映射 | 同上场次 | ✅ 扫描认人、报告对外接口（按 exam_no 查报告） |
| `users` | iam | `user_type`（teacher/student/school_admin）、`name`、`phone`、`student_id`（学生账号↔students）、三个微信 openid、`last_login_at` | ≈教师数+开通账号学生数 | ✅ 登录/RBAC；无画像字段 |
| `teaching_assignments` | org | `teacher_user_id × class_id × subject` 任教关系 | 每校数百行 | ✅ 数据范围解析（scope_resolver）与复核分工的依据 |
| `roles` / `user_roles` / `role_menus` / `delegations` | iam | `roles.data_scope` 六档；`user_roles.grade/subject/class_ids`（JSON 显式班级清单）；delegations 资源级临时授权（scope JSON + 起止时间） | 每校数十至数百行 | ✅ 权限判定；🔶 `class_ids` JSON 是少有的「个性化范围」数据，但仅用于授权 |
| `tenants` | iam | `code`（OSS 命名空间/外部下推标识）、`name`、`edition`、`settings`（JSON 租户级配置） | 全平台个位数–数十 | ✅ 租户隔离锚点 |

### A.2 考试与题目（modules/exam、modules/knowledge、modules/tiku、modules/scan）

| 表 | 域 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|---|
| `exam_batches` / `exams` | exam | 批次（学期/考试类型/日期）；场次三正交状态机 `status/extract_status/annotate_status`，`grade_mode/grade_locked/marking_identity`（阅卷模式与匿名/实名，管理端可锁），`pass_line_pct/excellent_line_pct` | 演示数场；生产见 §0 | ✅ 主链路核心实体 |
| `exam_classes` | exam | `exam_id × class_id`，`reviewer_user_id`（联考复核分工） | ≈场次×参考班级 | ✅ 复核分工 |
| `exam_questions` | exam | `content/standard_answer/analysis`（Text 题目/答案/解析）、`options/scoring_points/region/extract_raw/tags`（JSON）、`question_type`、`full_score`、`difficulty_prior` | ≈场次×题数（每场数十行） | ✅ 评分/讲评输入；`tags` 知识点标签是 `_local_kp_rates` 聚合依据 |
| `exam_cards` | exam | `regions/student_region/qno_regions`（JSON 答题卡区域坐标）、`confirm_status/confirmed_by` | 每场 1 行 | ✅ 切割/留痕渲染依据 |
| `exam_files` | exam | `exam_id × kind × file_id`（试卷/答案/答题卡 PDF 挂载） | 每场数行 | ✅ 材料管理 |
| `exam_events` | exam | 只追加事件审计：`event`、`operator_id`、`detail`（JSON） | 随操作量增长 | 🔶 审计留痕，无分析消费 |
| `knowledge_points` | knowledge | **知识点图谱主数据（本系统为权威源）**：`code`（科目.级.级.级）、`path` 物化路径、`level`（章/节/考点）、`textbook_version/module`（教材版本定位）、`core_competency`、`aliases/extra`（JSON）；`tenant_id=0` 平台内置 | 未知（有 smartedu 导入/打标脚本，无实测数） | ✅ 对内外提供知识树读取；被题目关联表引用 |
| `exam_question_knowledge` | knowledge | 题目↔知识点：`kp_id + kp_name_raw`（原始名留痕）、`match_type`（auto/人工）、`step_no`、`is_combo` | ≈场次×题×考点 | ✅ 考点掌握度聚合的锚点 |
| `questions`（tiku） | tiku | 校本/平台题库题目：`stem/answer/explanation`（HTML Text）、`type/difficulty`、`source_type`（gaokao/exercise/mock）、`source_channel/external_id`（幂等溯源）、`textbook_version/module/unit_name/case_name`（教材章节定位）、`extra`（来源原始元数据 JSON 快照） | 未知（smartedu 渠道批量导入） | ✅ 组卷抽题；qbank 模块复用本表（tenant_id=0 平台共享题库，`modules/qbank/routes.py:5`） |
| `question_knowledge` / `papers` / `paper_questions` / `paper_layouts` / `paper_templates` | tiku | 题↔知识点（kp_code 冗余）；组卷结果与抽题配置快照（`type_config/difficulty_config/kp_codes` JSON）；canvas 布局 JSON | 未知 | ✅ 组卷链路 |
| `scan_papers` / `scan_batches` / `paper_attendance` | scan | 扫描件：`object_keys`（原图 key 数组 JSON）、`pushed_identity`（推送方识别身份 JSON）、`raw_payload`（原始报文 JSON）、`card_pdf_key`、`region_images`（区域切图 JSON）、`match_status/matched_student_id/anomaly_reason`；考勤处置 | ≈卷数（生产日增数千行） | ✅ 认人/重卷/缺考处置；`region_images` 是预切图下推批阅服务的载体 |

### A.3 批阅结果与复核交互（modules/marking）——个体数据沉淀最厚的一组

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `student_papers` | 状态机 pending→…→confirmed；**`objective_score/subjective_score/total_score`（AI 原始）与 `final_score`（复核后）永不同列**；`class_rank/grade_rank`；`confirmed_by/at`；`annotated_pdf_file_id` | ≈卷数 | ✅ 学生端报告/排名；LLM 复用 ❌（API 文档 §5） |
| `student_answers` | **逐题双列留痕**：`recognized_answer`（作答原文 Text，「原始层只存不改」）、`chosen_option`；ai 侧 `ai_score/ai_feedback`（评语 Text）`/ai_confidence/confidence_level/scoring_point_results`(JSON)；快扫 V1 `review_level/calibrated_confidence/reason_codes/risk_flags/evidence_consistent`；人工侧 `manual_score/manual_scoring_points/override_reason`（改分原因，选填自由文本）`/review_status`；`final_score/reviewer_id/reviewed_at`；`region_image_url`（作答区切图） | ≈卷数×题数（单场约万行级，见 §0） | ✅ 复核工作台/报告/讲评下钻；LLM 复用 ❌——**记忆系统提案点名的第一候选数据源** |
| `score_audits` | **只追加、无 updated_at/deleted_at（模型层保证不可改删）**：`action`（adopt/override/point_change/to_manual/manual_score/undo/anonymous_toggle/rebatch）、`before_score/after_score`、`detail` JSON、`operator_id` | 随复核操作量增长 | ✅ 撤销快照游标、留痕查询；🔶 记忆系统提案 §6 点名「天然的 AI vs 人工差异日志，可蒸馏教师评分尺度偏好」，当前无任何此类消费 |
| `review_flags` | 疑点标记：`reason`（255 字自由文本）、`status`、`resolved_by/at` | 少量 | ✅ 复核协同 |
| `review_completions` | 「完成批阅」可撤销快照：`snapshot`（JSON 全量分数快照）、`audit_cursor`（score_audits 最大 id，拒绝有后续操作的撤销） | 每场 1–数行 | ✅ 撤销与打印失效联动 |

### A.4 学情聚合与班级/学校统计（modules/analytics、modules/manage、modules/tasks）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `exam_score_summaries` | 班级/年级口径汇总：`avg/max/min/pass_rate/excellent_rate`、`score_distribution/question_stats/common_issues`（JSON）、`source`（local/analysis） | ≈场次×班级 | ✅ 讲评挑重灾题（`common_issues`）、管理端报表 |
| `knowledge_mastery` | **学生×学科×知识点掌握度**：`mastery_score`（0–1 连续值）、`level`（stable/unstable/weak）、`main_error_layer`（concept/method/execution/habit 错因层）、`evidence` JSON、`source_exam_id` | 未知 | ⚠️ **本仓库无写入方**——全仓 grep 仅 models 定义、`exam/service.py` 软删清理、`db/dev_seed.py` 演示写入；消费方是 `student/service.py`（成就徽章）与 `manage/overview_service.py`（班级薄弱考点）。写入者应为外部学情分析系统直连或预留，需与学情系统侧核实 |
| `error_attributions` | 结构化错因：`cause_layer/cause_code/cause_text`、`confidence`、`evidence_level`、`source`（analysis/teacher） | 未知 | ⚠️ 同上：本仓库无写入方，仅删除与定义 |
| `student_tasks` | 学生个性化任务：`title/action_text/goal_text/time_text`、`status`（open/done）、`kp_id` | 未知 | ⚠️ 同上：无写入方；`student/service.py` 读它算成就 |
| `review_modes` / `manual_marking_permissions` | 租户级阅卷模式（grade_mode + 全校锁定）、人工批阅权限矩阵（角色×授权×范围） | 每租户数行 | ✅ 服务端复核授权判定 |
| `external_tasks` | 外部任务统一事实表：`kind`（grade_paper/grade_extract/card_annotate/analysis/tenant_backup）、`remote_task_id`、`dedup_key`（防重复付费）、`cost_cny`、`request_snapshot/response_snapshot`（JSON 留痕）、`superseded_by` 重批链 | ≈卷数+场次级任务 | ✅ worker 三循环驱动；`cost_cny` 是成本核算数据源 |
| （无表）班级/学校聚合 | — | — | 🔶 `manage/overview_service.py` 的全校概览/教学质量/教师发展是**读时多域 JOIN 实算**，不落任何快照表——无历史趋势可回溯 |

### A.5 校本库（modules/kb）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `kb_resources` | `kind`（lessons/slides/items/papers/custom）、`title`、`file_id→files`、`uploader_id/uploader_name`（快照）、`status/review_stage/reject_reason`（private/pending/pub/rejected + 初审/终审） | 未知 | ✅ 两级审核入库；⚠️ 无知识点/学科标签字段——与[数据架构](./data-architecture.md) L2「校本库与知识图谱双向打通」的差距在此 |
| `kb_custom_libraries` | 教师自定义库（仅创建者可见，可分享审核） | 未知 | ✅ |
| `kb_review_records` | 审核留痕：`stage/action/reviewer_id/reviewer_name/reason` | 未知 | ✅ 记名留痕 |

### A.6 教师个人配置/偏好（modules/prefs、modules/lecture、modules/feedback）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `user_nav_prefs` | `nav_order`（JSON 有序功能 key 列表）——**全系统唯一的显式「用户偏好」表** | 每用户≤1 行 | 🔶 仅 UI 导航排序，与 LLM/教学无关（API 文档 §5 已结论） |
| `lecture_notes` / `lecture_items` | 讲评单（每场一份）与条目：`talk_points`（Text，LLM 生成 + **教师可编辑改写**）、`error_tag`、`loss_rate/wrong_count/answer_count`（失分热度快照）、`included`（是否收进课件）、`presenter` | ≈场次×重灾题数 | ✅ 讲评 PPTX 导出；❌ 教师对 LLM 产物的改写**未回流为任何偏好数据**（记忆系统提案点名的蒸馏原料） |
| `feedbacks` | 教师「想让 AI 帮我做什么」心愿自由文本 `content`（平台级表，跨租户可查） | 少量 | 🔶 运营人工处理；是现成的需求挖掘语料 |

### A.7 学生端（modules/student）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `student_checkins` | 每日打卡（一人一天一条，`checkin_date` 本地日） | ≈活跃学生×天数 | ✅ 连续打卡阶梯、成就徽章 |
| （无表）成就徽章/班级森林 | — | — | 🔶 `student/service.py:149-200` **读时实算**（打卡连胜 + 成绩连涨/新高 + `knowledge_mastery` stable 计数 + `student_tasks` 完成率），不落库——行为事件只有打卡一种被持久化 |

### A.8 平台配置与运行留痕（与教学弱相关，但对数据架构重要）

| 表 | 说明 |
|---|---|
| `system_configs` | 平台级 KV 配置：含 `scene.<scene>.provider/model/fallback_*`（LLM 场景配置，改即生效）、`grading.auto_grade_mode`、`oss.*` 等 |
| `llm_providers` / `llm_api_keys` / `llm_call_logs` | LLM 网关（平台级，Key Fernet 加密）；`llm_call_logs` 记 scene/provider/model/token/耗时/成败，`cost` 列预留暂空 |
| `api_logs` | 全局 API 切面日志（出入参快照脱敏+截断）；有 `scripts/purge_api_logs.py` 清理脚本（有保留期意识） |
| `open_api_call_logs` | 开放 API 调用审计（预处理推送等机器调用） |
| `content_moderation_records` | 阿里云内容审核审计：**不存原文只存 sha256**，含 labels 快照/decision/策略版本 |
| `files` | 文件元数据（`object_key/sha256/mime/biz_type`：exam_paper/exam_answer/blank_card/scan_paper/scan_marked/blank_marked/lecture_pptx/import_excel 等） |
| `print_jobs` | 留痕批改卷打印任务：`params`（纸型/单双面 JSON）、`artifact_file_id` |
| `dict_items` / `menus` / `tenant_menus` / `platform_admins` / `open_api_keys` / `devices` / `device_credentials` / `app_versions` / `tenant_backup_policies` | 字典/权限目录/运营与设备/版本/备份策略（系统类，从略） |

## B. 非结构化数据盘点

### B.1 原始文件（对象存储）

**存储位置与格式**：统一在共享桶 `indievolve-grade`（生产阿里云 OSS，本地 MinIO；`app/infra/oss.py` 用 minio-py 走 S3 协议），key 规范的权威文档是仓库 `docs/OSS_NAMESPACE.md`：

```
{租户code}/exam/{exam_id}/material/     建考材料 PDF（paper/card/answer，版本化不覆盖）
{租户code}/exam/{exam_id}/student_paper/ 学生卷原图 JPG + 整卡 PDF
{租户code}/exam/{exam_id}/student_paper_cut/ 区域/单题切图
{租户code}/exam/{exam_id}/marked/       留痕批改卷 ZIP + 逐生回打件 PDF（key 含随机 token 防枚举）
{租户code}/exam/{exam_id}/report/       学生评阅报告 md/html/pdf（批阅服务生成，待对齐）
{租户code}/kb_resource/                 校本库资源文件
{租户code}/upload/                      建考暂存区（孤儿靠生命周期清理）
{租户code}/paper_pdf/                   组卷导出 PDF
shared/app_apk/  shared/question_images/  平台共享（App 包 / 题库题目内联图）
sample-lib/  task/                      批阅服务自有域
+ 存量遗留前缀（grading_system/t{n}/、t{n}/、paper/、exam/ 桶根等，只读不增）
```

| 文件类型 | 写入方 | 能否结构化 / 现有流水线 |
|---|---|---|
| 学生扫描件原图（JPG）+ 整卡 PDF | 预处理系统推送 / 教师本地上传 | **已有流水线**：①预处理系统切分回传 `region_images`（区域切图 key）；②`scan_recognition` VLM 把学生信息区图+花名册 → `matched_student_id`；③整卷下推外部批阅服务 → `student_answers` 结构化。潜在未建：卷面书写质量/图像质量特征提取（当前无） |
| 试卷/答案/答题卡 PDF | 教师建考上传 | **已有流水线**：①`adaptive_cutting` VLM 把答题卡 PDF 页 → `exam_cards.regions` 区域坐标 JSON；②外部批阅服务 5-Agent 提取题目 → `exam_questions`（`extract_raw` 留原始结构） |
| 校本库上传资源（教案/课件/试题/整卷，任意格式） | 教师上传 | **无结构化流水线**：入库只有人工标题+人工审核，无内容抽取、无知识点打标。可建流水线：资源文本抽取 → LLM 打学科/知识点/类型标签 → 与 knowledge_points 关联（对应[数据架构](./data-architecture.md) L2 的「双向打通」缺口） |
| 留痕批改卷 ZIP/PDF、讲评 PPTX、组卷 PDF、逐生评阅报告 | 本系统 printing/lecture/tiku、批阅服务 | 终端产物，价值在分发与存档，**结构化价值不大**（其内容全部可由结构化数据重新生成） |
| 题库题目内联图 | 平台运营（smartedu 导入） | 已被 `questions.stem` HTML 引用，属题目内容的一部分 |
| 头像 | — | **不存在**：全仓无 avatar 字段/上传路径，用户体系不收集头像 |
| 导入 Excel（学生/教师/任课/考号） | 学校管理员 | 一次性结构化入口（`org/importer.py`），导入即入结构化表，无需常驻流水线 |

### B.2 自然语言文本（存在数据库 Text/大字段里的）

| 文本 | 位置 | 产生方 | 能否结构化 / 现有流水线 |
|---|---|---|---|
| **学生作答原文** | `student_answers.recognized_answer`（「原始层只存不改」） | 外部批阅服务 OCR/VLM | **已有结构化**：批阅服务已产出 ai_score/采分点得分。可再建：按错误类型聚合作答原文语料 → 学生个人错误模式（对应 `error_attributions` 的设计意图，但该表本仓库无写入方） |
| **AI 逐题评语** | `student_answers.ai_feedback` | 外部批阅服务 | **无流水线**。可建：评语 → LLM 抽取高频失分短语/知识点 → 回灌班级学情聚合与讲评（当前讲评只用失分率数值，不用评语文本） |
| **讲评要点（LLM 生成 + 教师改写）** | `lecture_items.talk_points`（本系统 `lecture_talk` scene 生成） | LLM + 教师编辑 | **无回流流水线**：教师改写前的 AI 原文未留版本，改写行为本身也未记录——这是蒸馏「教师讲评风格」的最直接原料，当前只能靠改库前快照丢失（⚠️ 建议最小改造：生成时把 AI 原文另存一列） |
| **教师复核自由文本** | `student_answers.override_reason`（改分原因，≤64 字，选填）、`review_flags.reason`（≤255 字） | 教师 | **无流水线**，量小但价值密度高（改分动机）。可建：LLM 归一化为原因码枚举，与系统侧 `reason_codes` 对齐成同一套码表 |
| **教师心愿/反馈** | `feedbacks.content` | 教师 | 无流水线；运营人工读。可作需求挖掘语料（LLM 聚类） |
| 题目/答案/解析 HTML | `exam_questions.content/standard_answer/analysis`、`questions.stem/answer/explanation` | 批阅服务提取 / smartedu 导入 / 人工录入 | **已有部分结构化**：`exam_question_knowledge`/`question_knowledge` 挂知识点（match_type=auto 为自动匹配）；离线脚本 `scripts/tag_smartedu_knowledge_llm.py` 用本系统 LLM 网关批量打标——这是仓库内**唯一已存在的「文本→LLM→结构化标签」离线流水线** |
| 错因文本 | `error_attributions.cause_text` | 外部学情分析系统（推定） | 表本身即结构化产物（cause_layer/cause_code 枚举化），但 ⚠️ 写入方不在本仓库 |
| 学生评阅报告 md/html | OSS `report/` 前缀 | 批阅服务 | 生成式产物；本系统无任何 LLM 生成学生自然语言报告（API 文档 §2.7 结论） |

### B.3 日志 / 事件流

| 流 | 位置/格式 | 能否结构化 / 现状 |
|---|---|---|
| API 切面日志 | `api_logs` 表（结构化入库，含出入参快照脱敏+截断）；`scripts/purge_api_logs.py` 定期清理 | **已是结构化**；可做使用行为分析（功能热度、教师操作路径），当前仅运营排障查询 |
| 开放 API 调用审计 | `open_api_call_logs` 表 | 已结构化，机器调用对账用 |
| LLM 调用账本 | `llm_call_logs` 表（scene/provider/model/token/耗时；cost 列预留空） | 已结构化；⚠️ 只覆盖本系统 3 个 scene，批阅主链路的模型调用走 grading_service 自有账本，两账不互通（API 文档 §4.2） |
| 考试事件审计 | `exam_events` 表（只追加） | 已结构化，无分析消费 |
| 评分操作留痕 | `score_audits` 表（只追加，模型层禁改删） | 已结构化；**记忆系统提案点名的教师偏好蒸馏源**（adopt/override 差异 → 评分尺度画像），当前无此消费 |
| 外部任务留痕 | `external_tasks.request_snapshot/response_snapshot` JSON | 已结构化（JSON 快照），排障用 |
| 内容审核审计 | `content_moderation_records`（不存原文，只存 sha256+标签） | 已结构化；合规约束下**不应**建「还原原文」的流水线 |
| 前端埋点 / 浏览器行为事件 | **不存在**：frontend/src 全仓无任何埋点/分析 SDK（无 sentry/posthog/友盟等） | 如需要须从零埋点；当前教师端「已收进/换说法」等交互只有结果落库（lecture_items），无过程事件流 |
| 进程日志 | stdout（worker `logging.basicConfig` 带时间戳，uvicorn access 日志），无落盘文件配置 | 运维采集侧的事，不进数据资产 |

### B.4 非结构化配置 / JSON 字段（库内 JSON，实有结构）

全库 JSON 字段遵循「弹性快照、不参与 SQL 聚合」的 schema 铁律。按可结构化价值排序：

| 字段 | 实际结构 | 说明 |
|---|---|---|
| `exam_cards.regions` / `student_region` / `qno_regions` | 答题卡区域坐标 `[{qno,type,page,region:{x,y,w,h%}}]` | VLM 标注 + 教师确认的强结构数据，切割/渲染直接消费 |
| `exam_questions.scoring_points` / `extract_raw` | 采分点数组 / 批阅服务原始提取结构 | 组卷场次本地为权威源；extract_raw 是重提取的兜底 |
| `student_answers.scoring_point_results` / `manual_scoring_points` | 逐采分点得分明细 | 「分项之和 vs 总分一致性」校验（`evidence_consistent`）的依据 |
| `student_answers.reason_codes` / `risk_flags` | 快扫判定原因码、硬性风险标记数组 | 已是枚举化结构（review_rules.py），可直接聚合统计 |
| `review_completions.snapshot` | 完成时刻全量分数快照 | 撤销恢复用，不宜作分析源 |
| `exam_score_summaries.score_distribution` / `question_stats` / `common_issues` | 分段分布 / 逐题失分率 / 共性错因 | 班级/场次聚合的落库形态，讲评直接消费 |
| `scan_papers.object_keys` / `pushed_identity` / `raw_payload` / `region_images` | 原图 key 数组 / 推送方识别身份 / 原始报文 / 切图清单 | raw_payload 是幂等排障兜底 |
| `system_configs`（KV）/`scene.*` | LLM 场景模型配置、批阅开关 | 配置即数据，运营后台可视化编辑 |
| `user_roles.class_ids` / `delegations.scope` / `tenants.settings` | 显式班级清单 / 委派范围 / 租户设置 | 权限语义 JSON，结构化需求低 |
| `knowledge_points.aliases/extra` / `questions.extra` / `exam_events.detail` / `score_audits.detail` / `llm` 各 JSON / `print_jobs.params` / `paper_layouts.layout` / `papers.*_config` / `user_nav_prefs.nav_order` / `content_moderation_records.labels` | 别名/来源快照/事件详情/排版与抽题配置快照等 | 留痕与快照性质，结构化价值低 |

## C. 与记忆系统提案的映射（盘点视角的结论）

按[记忆系统提案](./memory-system-proposal.md) §6 的三档分类核对本仓库实况：

1. **现成可用（零改造）**：`score_audits`（AI vs 人工差异日志）✅ 确认；`student_checkins` ✅。⚠️ 提案 §5 L1 表把 `student_answers`/`score_audits` 归属标为「grading-service」——实际两张表都在**本仓库**（`modules/marking/models.py`），批阅服务侧的同名概念需另行核实，提案引用时应修正归属。
2. **需要蒸馏（数据在、缺抽取层）**：`student_answers` 双列留痕 → 学生个人错误模式 ✅ 确认可行；`override_reason`/`review_flags.reason` 教师自由文本 → 改分动机码表（本文 B.2）；`lecture_items` 教师改写 → 讲评风格（⚠️ 需先补「AI 原文留版」的最小改造，否则蒸馏原料不完整）。
3. **当前完全不存在**：教师/学生画像表、行为事件流（前端无埋点）、头像等基础画像字段、`knowledge_mastery` 等三张个人化学情表的**本仓库写入方**（记忆系统若要用它们做学生个人学业记忆，需先与学情分析系统核实写入链路归属）。

## D. 与既有文档的不一致标注

- ⚠️ 与[记忆系统提案](./memory-system-proposal.md)：§5 L1 数据源归属标注「grading-service」的 `student_answers`/`score_audits`，实测属本仓库 grading-system（见本文 A.3）。
- ⚠️ 与[API 与业务流程](./api-and-process-grading-system.md) §5「考点掌握不落个性化表」：准确描述本系统的写路径，但需补充——`knowledge_mastery`/`error_attributions`/`student_tasks` 三张个性化表 schema 存在且被 student/manage 模块读取，本仓库无写入方（仅 dev_seed 与软删清理），写入链路指向外部学情分析系统，归属待核实。
- ⚠️ 与[软件架构](./architecture-grading-system.md)「25 个域模块」：实测 `backend/modules/` 下 24 个域目录（admin/analytics/device/exam/feedback/files/iam/kb/knowledge/lecture/llm/manage/marking/moderation/org/platform/prefs/printing/qbank/scan/student/syslog/tasks/tiku），差异可能来自路由挂载口径，不影响本文按表盘点。
- ⚠️ 与[数据架构](./data-architecture.md) L2：校本库表结构确认**没有知识点标签字段**，「与 L1 知识图谱双向打通」需要给 `kb_resources` 增加标签列或关联表——这是本盘点对那份提案的具体补充。
- ✅ 与[API 与业务流程](./api-and-process-grading-system.md) §5 总结论一致：本系统无用户画像/记忆机制，个人历史数据厚但零 LLM 复用。

相关：[数据架构](./data-architecture.md) · [API 与业务流程](./api-and-process-grading-system.md) · [记忆系统提案](./memory-system-proposal.md)
