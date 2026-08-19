---
type: concept
---

# 批阅服务数据资产盘点（aireview-grading-service）

> 调研对象：`~/src/indievolve/aireview-grading-service`（main 分支，2026-08）。本文是[批阅系统数据资产盘点](./data-assets-grading-system.md)（姐妹仓库侧）的对侧互补，同为[记忆系统提案](./memory-system-proposal.md)与[数据架构](./data-architecture.md)的前置调研。
> 盘点方法：通读 `core/db/` 全部模型文件（`exam_models.py` + `models.py`，共 26 张表）、`core/cache/redis_client.py`、`core/file/storage_keys.py`、`core/subjective/rubric_optimizer.py` + `optimization_shared.py`、`core/log_collector.py`、`docs/oss_storage_key_convention.md` 及关键调用点；「是否被 LLM 调用复用」一列直接引用[批阅服务 API 与业务流程](./api-and-process-grading-service.md)（下称「API 文档」）的既有结论，不重复调研。与既有文档不一致处以 ⚠️ 标注。

## 0. 全局事实（先看这部分）

- **存储格局**：三存储分工——MySQL 是终态主存与配置主存（唯一业务真值，连接失败拒绝启动）；Redis 是队列 + 缓存 + 投影（全部可重建，分层 TTL：预计算缓存 30 天 / 在途任务 24h / 投影 7 天，`config/settings.py:197-199`）；MinIO/阿里云 OSS 是图片对象存储（与 grading-system、预处理、教师端共享同一桶与 `{tenant_code}/exam/{exam_id}/{业务域}/` 命名空间，权威约定见仓库 `docs/oss_storage_key_convention.md`，真值源 `core/file/storage_keys.py:exam_domain_key`）。结论来源：[软件架构](./architecture-grading-service.md) §5。
- **表结构口径**：26 张表全部定义在两个文件——`core/db/exam_models.py`（14 张：考试/整卷/单题/优化历史/仪表盘/配置/API Key）与 `core/db/models.py`（12 张：样本库/回归/抽查/裁定/看板产物）。无 Alembic，schema 演进靠 `init_db()` 启动时幂等补丁加列加索引（`core/db/engine.py`）。时间戳/数值统一 `DOUBLE` 存 epoch 秒（Float 会把 10 位 epoch 量化到 ~128s，破坏与 Redis zindex 排序一致）。大字段一律拆重表（`*_details`）避免拖慢列表查询。
- **数据量级总体判断**：本机没有可查询的本系统数据库（docker 里的 MySQL 是其他项目 e2e 容器），量级只能给两个口径：
  - **实测锚点**（代码注释与文档中的真实数字）：`question_config_snapshots` 注释记载「38.6k 任务 → ~1k distinct，37:1 压缩比」（`core/db/exam_models.py:227`）——说明盘点时点 `grading_tasks` 量级在数万行；2026-08-13 实测一批 90 份整卷（`paper_pipeline.py` 整卷收敛补偿注释）；单卷成本报告一场高三英语卷 12 道主观题 + 55 道客观题（仓库根 `AI批阅服务单卷成本分析报告.md`）。
  - **生产规模推算**：引用姐妹篇[批阅系统数据资产盘点](./data-assets-grading-system.md) §0 的 TESTPLAN 基线——单场考试约 480 卷、全天约 5000 卷。按单场 15–25 题估算，一场考试对应 `grading_tasks` 约 0.7–1.2 万行、`paper_tasks` 约 480 行，与上面 38.6k 的实测锚点自洽（数十场考试的积累量）。
- **对个人化/记忆系统最重要的一句话**：本服务的定位是「AI 算法层、内部系统」，学生身份（`student_id`/`student_name`/`student_no`）在 6 张表里有字段，但全部只是**下推元数据的落库与检索**，服务内没有任何按学生聚合的计算；沉淀最厚的个体数据是 `grading_task_details.result_json` 里的逐题评分结果（含 AI 评语 `feedback`/`feedback_detail`、分步得分 `step_scores`、推理痕迹 `reasoning_trace`）——这是 grading-system 侧 `student_answers` 双列留痕的**上游源头**，目前已有一个现成的按学生聚合读接口（`GET /api/v1/learning/grading-results?student_id=`，API 文档 §6）。学生姓名进入 LLM 请求体仅两处历史性例外（花名册识别、细则优化样本标题），其中细则优化样本摘要文本**只存在于内存，从不落库**（详见 B.2 重点分析）。

## A. 结构化数据盘点

「复用现状」列：✅=已被业务逻辑读取消费；🔶=存在但消费面窄/读时实算；❌=无任何消费方；⚠️=有异常。LLM 复用细节统一见 API 文档 §2/§3，此处只给结论。

### A.1 考试与题目（core/db/exam_models.py）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `exams` | `exam_id`（主键）、`subject/grade`、`tenant_code/tenant_name`（租户唯一真值源，向下游冗余传播）、`status`（pending/processing/ready/failed）、三个 PDF URL、`usage_json`（建考/提取期资源消耗） | ≈考试场次数（数十至数百行） | ✅ 主链路核心实体 |
| `exam_details` | `processed_exam_images_json`（处理后试卷页图 URL 列表）、`questions_json`（题目列表完整 JSON）、`regions_json`（区域标注） | 每场 1 行（MEDIUMTEXT） | ✅ 详情页与预计算输入 |
| `exam_questions` | **题目与预计算三件套**：`question_text/reference_answer/answer_analysis`（题目级内容）、`knowledge_points_json`（知识点 JSON 数组）、`question_analysis_json`（Agent1 产物）、`rubric_json`（Agent3 产物）、`extraction_rules`（Agent2 预计算产物）、`rubric_version/extraction_version`（优化版本号） | ≈场次×题数 | ✅ 批阅主链路唯一读取的题目数据源（Redis 缓存的 MySQL 持久副本，`redis_client.py` 双写回源）；⚠️ `knowledge_points_json` 只是 AI 提取出的知识点**名称字符串数组**，无知识点图谱 ID——与[数据架构](./data-architecture.md) L1「知识点权威源」存在打通缺口 |

### A.2 批阅任务与结果（个体数据沉淀最厚的一组）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `paper_tasks` | 整卷汇总：`student_id/student_name/student_no`、`class_id/class_name/exam_room`、`status`、四个分数列（total/objective/subjective/max）、`total_tokens/total_cost_usd/total_calls/total_elapsed`；索引 `idx_paper_exam_student(exam_id, student_id)` | ≈卷数（日增数千行，见 §0） | ✅ 轮询/列表/结算；`student_*` 仅元数据落库与模糊搜索（API 文档 §6）；LLM 复用 ❌ |
| `paper_task_details` | `sub_tasks_json`、`results_json`（LONGTEXT 完整结果）、`student_info_json`（学生信息识别结果）、`student_info_image_url`（信息区切图留存，供重识别）、`usage_json`（整卷用量明细含 by_model 分项）、`request_json`（原始提交，大键走 request_blobs 占位） | ≈卷数 | ✅ 详情/重算/对外结算的数据源 |
| `grading_tasks` | 单题任务：`paper_task_id/exam_id/question_number`、`student_id/student_name`（冗余自整卷）、`score/confidence/is_rejected`、**`teacher_score/diff`（教师评分与 AI 分差）**、`extraction_model/grading_model`、`total_cost_usd/total_tokens`、客观题专属 `matched_count/accuracy`；索引 `idx_task_exam_student` | ≈卷数×题数（实测锚点 38.6k+ 行） | ✅ 列表/复核/统计；🔶 `teacher_score` 靠 `/backfill-teacher-scores` 事后回填，覆盖不全，但已是一个**稀疏的 AI vs 人工差异数据源**（grading-system 侧 `score_audits` 的上游雏形） |
| `grading_task_details` | **`result_json`（LONGTEXT，完整评分结果：score/step_scores/reasoning_trace/feedback/feedback_detail/confidence/validation）**、`agent_details_json`（五 Agent 执行详情，题目级两键走 question_config_snapshots 占位）、`usage_breakdown_json`、`history_json`（每次重跑前的评分快照）、`request_json`（去 base64 图，供重跑重建）、客观题三 JSON | ≈卷数×题数 | ✅ 详情/重跑/细则优化数据源；**AI 评语自由文本的实际存放处**，LLM 复用仅在细则优化（批次级摘要）与评阅报告（当卷个体）两处 |

### A.3 内容寻址去重表（无 GC，含 PII 留存）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `request_blobs` | `blob_hash`（MD5 主键）、`content`（class_students 花名册 / card_regions_json 的 canonical JSON） | 去重后数百至数千行 | ✅ 请求大键去重；⚠️ **无 GC，花名册 PII（全班学号+姓名）永久留存**（模型注释自承，`exam_models.py:207-225`） |
| `question_config_snapshots` | `snap_hash`（MD5）、`kind`（question_analysis/grading_rubric）、`content`（批阅时生效的题目配置快照） | 实测 38.6k 任务 → ~1k 行（37:1） | ✅ 任务快照与 `exam_questions` 现值 95% 漂移（细则持续优化），存快照而非引用；⚠️ 同无 GC |

### A.4 成本/用量/报告（对外结算的数据基础）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `paper_report_logs` | 每次报告生成追加一行：`model/cost_usd/cost_cny/tokens/calls/pages/duration_ms`、`params_json`、`report_urls_json {md,html,pdf}` | ≈报告生成次数 | ✅ 报告成本审计与缓存命中（重生成先查本表） |
| `dashboard_stats` | `stat_type`（paper/cost/time/model/score_dist/exam）×`stat_key`×`stat_date`×`hour` 的通用统计 KV（count/float/json 三列） | 随聚合任务增长 | ✅ 仪表盘读时聚合的落库形态 |
| `paper_tasks.total_cost_usd` + `paper_task_details.usage_json` + `grading_tasks.total_cost_usd` + `grading_task_details.usage_breakdown_json` | 成本四级落点：单题 → 整卷（by_model 分项 + scenes 场景维）→ 批次/考试聚合 | 同任务量 | ✅ **对外结算接口（`api/routes/external_api.py`）即读这套数据**；采集机制见[软件架构](./architecture-grading-service.md) §7（contextvars 隔离账本 + CNY 阶梯价表 + USD OpenRouter 实际扣费） |

### A.5 细则优化历史（题目级演进留痕）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `optimization_history` | `opt_type`（rubric/extraction）、`old_content_json/new_content_json`（优化前后细则）、`notes`（LLM 生成的优化说明）、`trigger_reason`、`sample_count`、`version` | ≈场次×题×优化轮次（默认 3 份卷触发一次，`settings.rubric_optimization_threshold=3`） | ✅ 优化回溯与版本管理；双写自 Redis LIST（保留最近 10 条，`rubric_optimizer.py:_save_history`）；🔶 `notes` 是现成的「题目级评分规律」文本，目前只用于展示，无进一步结构化消费 |

### A.6 样本库 / 回归 / 抽查 / 问题样本裁定（core/db/models.py，12 张）

| 表 | 关键字段 | 量级 | 复用现状 |
|---|---|---|---|
| `samples` | 样本库单表：`kind`（accuracy/error）、`student_id/student_name`、`verified_score/verified_answer`（人工核准真值）vs `ai_score/ai_answer/ai_confidence/ai_model`、**`error_type`（extraction/grading/objective/student_info）+ `error_pattern`（错误模式枚举，逗号多选）**、`review_accuracy/review_diff_count`（AI vs 人工字符级差异）、`identity_key`（学生×题目身份键）、`question_context_json`（题目快照，变量隔离回归用） | 未知（文档记载一次审计 128/140 条样本的量级） | ✅ 人工核对工作台与回归基线；**`error_pattern` 是全仓库唯一的「错误模式枚举标签」列**——即记忆系统想要的「不含姓名的错误模式标签」已经以人工标注形态存在，缺点是依赖人工、覆盖仅限入库样本 |
| `sample_audit` | `verified_answer` 改动审计（field/old_value/new_value） | 随核对操作量 | ✅ 留痕 |
| `discarded_tasks` | 随机审核废弃任务（含 `student_id/student_name/reason/discarded_by`） | 少量 | ✅ 审核剔除留痕 |
| `bench_runs` / `bench_items` | 回归测试：runs 记 `mode/filter/models/total_cost_usd/total_cost_cny/summary_json/roster_json`；items 记逐条 `exact_match/score_match/char_sim/diff_count/corrected/cost/detail_json` | ≈回归次数×候选数 | ✅ Prompt/模型变更的回归验证体系（配置叠加走 `config:test:*` 隔离命名空间） |
| `spotcheck_assignments` / `spotcheck_records` / `spotcheck_review_events` / `spotcheck_adjudications` | 核对质量抽查：分配快照（冻结抽查率分母）→ B/C/D 逐人判定（verdict/corrected_answer，token 尾 8 位记名、完整 token 不落库）→ 只追加事件 → A 最终裁决回写真值 | 随抽查批次 | ✅ 外部核对人质量治理闭环；records 不直接回写 samples，裁决采纳才写 |
| `fails_settlement_records` | 问题样本逐人独立裁定：`fail_class`（A/B/C/D 主归因）+ `fail_classes`（多选全集）、`fail_why`（归因说明自由文本）、`evidence_sha256/detail_sha256`（证据指纹，证据变了旧裁定失效）、`adopted` | 随裁定批次 | ✅ 达标看板问题样本的归因裁定；**`fail_class` 是第二套错误归因标签体系**（面向系统性失败而非个体错误） |
| `fails_report_bundle` / `dashboard_bundle` | 看板产物整包入库（HTML+图映射+上下文 JSON），`is_active` 标记对外服务版本 | 单行/少数行 | ✅ 免文件同步的产物分发，与业务数据弱相关 |

### A.7 配置与凭证（系统类，从略）

| 表 | 说明 |
|---|---|
| `config_overrides` | 动态配置 MySQL 主存（settings/strategies/confidence/prompts 四命名空间，与 Redis hash 逐字一致双写）；Prompt 覆盖也在此——**Prompt 模板版本本身是一种高价值配置数据**（回归体系证明其可测） |
| `api_keys` | 受限 API Key 主存（明文 key + scopes JSON，Redis 双写防丢） |

### A.8 学生相关字段确认（任务点名的核查项）

grading-service 自己的 MySQL 里 **有** `student_id/student_name`（及 `student_no`）字段的表共 6 张，全部只存**当次提交的元数据粒度**（学号/姓名/班级/考场），不存在任何学生属性扩展字段（无性别/年龄/历史标签）：

| 表 | 学生字段 | 粒度与用途 |
|---|---|---|
| `paper_tasks` | `student_id/student_name/student_no` + `class_id/class_name/exam_room` | 整卷级；下推身份或识别结果的落库，按学号/姓名模糊搜索（`exam_query.py:578`） |
| `grading_tasks` | `student_id/student_name` + `class_name/exam_room` | 单题级冗余（避免 join）；`idx_task_exam_student` 以 exam_id 前导 |
| `samples` | `student_id/student_name` + `identity_key`（学生×题目键） | 样本级；检索同一学生同题样本 |
| `discarded_tasks` | `student_id/student_name` | 废弃任务留痕 |
| `paper_task_details` | `student_info_json/student_info_image_url` | 学生信息**识别结果与切图**（原始模式产物） |
| `request_blobs` | class_students 花名册 JSON（全班 学号:姓名 清单） | 请求去重内容；⚠️ 无 GC，**全班级 PII 永久留存** |

与姐妹篇的互补结论：两仓库**没有任何同义表重复**——`student_answers`/`score_audits` 确认只在 grading-system；grading-service 侧最接近的对应物是 `grading_task_details.result_json`（作答提取文本 + AI 评分 + 评语，是 `student_answers` 的上游源头）与 `grading_tasks.teacher_score/diff`（稀疏的 AI vs 人工差异，远比 `score_audits` 薄）。跨考试按学生聚合目前只有 `learning` 接口的 SQL 过滤（无 student_id 前导索引，量大后是慢查询点，API 文档 §6 已记）。

## B. 非结构化数据盘点

### B.1 原始文件（对象存储）

**存储位置与格式**：与 grading-system 共享同一桶（生产阿里云 OSS，本地 MinIO），key 规范 `{tenant_code}/exam/{exam_id}/{业务域}/`（业务域白名单：`student_paper/student_paper_cut/material/report/marked`，`core/file/storage_keys.py:EXAM_MEDIA_DOMAINS`），另有存量桶根前缀 `paper/ exam/ task/ report/ sample-lib/`（媒体守卫 `_ALLOWED_MEDIA_PREFIXES` 放行，`minio_client.py:31`）。

| 文件类型 | 写入方 | 能否结构化 / 现有流水线 |
|---|---|---|
| 答题区域切图（`student_paper_cut/{卷标识}/{区域}.png`） | 预处理服务下推 / 本服务原图模式自切 | **已有流水线**：区域图 → Agent2 提取 → `grading_task_details` 结构化结果；这是批阅主链路的输入 |
| 学生卷原图 + 整卡 PDF（`student_paper/`） | 硬件上传 / 教师直传 | **已有流水线**（原始模式）：VLM 学生信息识别 + 区域坐标切割 → 区域图。预切割主链路本服务不接收原图 |
| 建考材料 PDF（`material/{exam|answer|card}.pdf` + `processed_page_N.jpg`） | 建考上传（base64 才上传，裸 key 只登记） | **已有流水线**：`question_extractor` 两步提取（题干 + 答案合并，可选多模型投票）→ `exam_questions` |
| OCR 增强图 / 预处理图（`student_paper_cut/{卷标识}/{task_id}_{roi_reference|preprocessed}.jpg`） | `preprocessed_image_store.py`（fire-and-forget，失败不阻塞） | 调试/详情页对比展示用；**结构化价值不大**（是中间产物，增强参数已固化在代码里）。注意：增强图默认**不喂 LLM**（`attach_ocr_enhanced_image_to_llm=false`，强模型下增强图反引入伪影，`answer_extractor.py:463-469`） |
| 学生信息区切图（`student_info_image_url` 指向） | 原始模式切割 | 留存供 `recognize-student` 重识别；一次性消费 |
| 样本冻结图（`sample-lib/{sample_id}.{ext}`） | `core/sample_lib/image_clone.py`（入库即克隆，防源图清理后失效） | 回归测试的输入资产，随样本库治理 |
| 学生评阅报告 md/html/pdf（`report/{paper_id}/{名}-{ts}.ext`） | `core/reports/paper_report.py`（多版本各存一份） | 生成式终端产物，结构化价值不大（内容可由 `grading_task_details` 重新生成） |
| 留痕批改产物（`marked/`） | grading-system 侧为主 | 本服务不写，从略 |

### B.2 自然语言文本（DB 大字段 / Redis JSON 里的）

| 文本 | 位置 | 产生方 | 能否结构化 / 现有流水线 |
|---|---|---|---|
| **AI 逐题评语** | `grading_task_details.result_json` 的 `feedback`/`feedback_detail`（UGP validator 归一化为 markdown，`universal_grading_protocol.py:143-205`） | Agent4 grader | **无独立结构化流水线**。可建：评语文本 → LLM 抽取常见失分短语/错误模式 → 结构化标签表（与五 Agent 同技术栈：structured_completion + Pydantic 契约 + 学科路由）。⚠️ 但注意已有两条**部分替代品**：①grader 输出的 `step_scores[].feedback`（采分点级扣分原因，半结构化）；②`samples.error_pattern`（人工标注的错误模式枚举）。新建流水线的增量价值在于覆盖**全量任务**而非入库样本 |
| **学生作答提取文本** | 同上 `result_json.extracted_answer`（经 `sanitize_student_answer_text` 清洗） | Agent2 | **已是结构化产物的核心字段**（评分直接消费）。可再建：按题目聚合作答语料 → 常见错误答案聚类（对细则优化有直接价值，见下） |
| **评分推理痕迹** | `result_json.reasoning_trace` | Agent4 | 半结构化；细则优化已截取前 300 字进摘要（`rubric_optimizer.py:_collect_grading_data`） |
| **题目分析文本** | `exam_questions.question_analysis_json` / Redis `exam:{id}:q:{qn}:question_analysis`（JSON） | Agent1（预计算） | **已是结构化**（UGP `QuestionContext` Pydantic 契约） |
| **参考答案/评分细则** | `exam_questions.rubric_json` / Redis `...:rubric`（JSON） | Agent3（预计算） | **已是结构化**（UGP `ReferenceAnswer` 契约：rubric 采分点分值 dict + key_concepts + scoring_levels + answer_equivalence） |
| **提取细则文本** | `exam_questions.extraction_rules` / Redis `...:rules`（纯文本） | Agent2 预计算（GPT-5.2 Pro） | 半结构化（300 字内四段式约定，`optimization_shared.py:EXTRACTION_OPTIMIZATION_PROMPT`）；可读性差但消费方只有 Agent2 prompt，结构化价值低 |
| **细则优化说明** | `optimization_history.notes` | rubric_optimizer LLM | 题目级「评分规律」分析文本；可作教研侧洞察语料，量小 |
| **评阅报告文本** | OSS `report/` md/html + `paper_report_logs.report_urls_json` | `paper_report.py`（当卷个体作答 + AI 评语 → 逐题失分分析） | 生成式产物；是**评语二次加工的唯一下游**（API 文档 §3.6） |
| **问题样本归因说明** | `fails_settlement_records.fail_why`（自由文本） | 裁定人 | 已有结构化主归因 `fail_class`；自由文本可 LLM 归一化补充，量小价值密度高 |
| **样本备注/抽查备注** | `samples.note`、`spotcheck_records.note` | 核对人 | 量小，同人工复核自由文本，可并入统一的原因码蒸馏 |

**重点分析：rubric_optimizer 样本摘要文本（任务点名项）**

实测结论（`core/subjective/rubric_optimizer.py` + `optimization_shared.py` 逐行确认）：

1. **存在哪里**：样本摘要文本（`format_grading_summary`/`format_extraction_samples` 产出，标题 `### {student_name} - 得分...`）**只存在于优化调用的内存中**——数据源是 Redis 在途投影（`paper_task:{id}` + `grading:task:{task_id}` 的 result/agent_details 字段，`_collect_grading_data:279` / `_collect_extraction_data:318`），格式化成 prompt 后即弃。**不落任何库表**：`_save_history` 只持久化 `old/new rubric/rules + optimization_notes`（Redis LIST 保留 10 条 + 双写 MySQL `optimization_history`），学生姓名不在持久化字段里。
2. **保留多久**：摘要文本本身零保留（函数栈生命周期）；其输入数据（Redis 任务投影）随终态驱逐 + TTL 消亡（在途 24h/投影 7d），MySQL 侧的 `result_json`（姓名在轻表、结果在大表）永久保留但不含拼接后的摘要形态。
3. **姓名暴露面**：姓名确实进入了 `rubric_optimization_model`/`extraction_optimization_model` 的 LLM 请求体（API 文档 §2.2 已记为合规事实），但**不进入任何持久存储**——这比「落库含姓名文本」的风险等级低一档。
4. **能否结构化为不含姓名的错误模式标签**：**完全可行，且原料现成**。建议流水线：`_collect_grading_data` 的聚合点（天然按 exam×question 分批）→ 把 `step_scores[].criterion + feedback`（采分点级扣分原因，**本身不含姓名**）经 LLM 聚类为错误模式标签 → 写入新增结构化表（如 `question_error_patterns(exam_id, question_number, pattern, count, sample_task_ids[])`）。关键观察：**优化器其实已经丢弃了姓名以外的所有个体标识需求**——`### {name}` 标题只是给 LLM 区分样本的分隔符，改成 `### 样本{i}` 即可零成本去 PII（一行改动，`optimization_shared.py:177/216`）。这与[记忆系统提案](./memory-system-proposal.md) §8「不把姓名拼进 LLM 请求」的约束直接对齐，建议作为该提案 P0 的最小改造项。
5. **当前是否已有类似流水线**：**没有**。仓库内唯一的「文本→LLM→结构化」离线流水线在 grading-system（smartedu 知识点打标脚本），本服务侧所有 LLM 调用都是在线同步路径；`samples.error_pattern` 是人工标注而非抽取产物。

### B.3 日志 / 事件流

| 流 | 位置/格式 | 能否结构化 / 现状 |
|---|---|---|
| 任务队列状态变化 | Redis hash `grading:task:{id}`（status/progress/result）+ `grading:status_counts` 计数器 + MySQL 终态行 | **已是结构化**（状态机字段化）；状态迁移历史不留（终态即驱逐），只有 `grading_task_details.history_json` 保留**重跑前评分快照** |
| 成本追踪记录 | contextvars 隔离账本 → `usage_breakdown_json`/`usage_json`/`total_cost_*` 列 → `dashboard_stats` 聚合 | **已是结构化**（by_model + scenes 场景维，[软件架构](./architecture-grading-service.md) §7）；是对外结算的直接数据源 |
| 告警事件 | `core/alerting.py` 飞书 webhook（内存危险/节点判死/重试超限，冷却去重，best-effort） | **无持久化**——发出去就完了，无告警历史可查；如需审计须自建 |
| 应用日志 | `core/log_collector.py`：**内存环形缓冲区（deque maxlen=2000）+ SSE 广播**，按 task_id/exam_id/level 过滤（contextvars 自动注入） | **无落盘**——重启即失，仅支撑前端实时日志流；进程 stdout 日志是运维采集侧的事，不进数据资产 |
| 节点心跳/扩缩容事件 | Redis `grading:nodes`/`node_heartbeat:*`（TTL 自动离线） | 投影态，可重建，非资产 |
| 复核/核对/裁定事件 | `sample_audit`、`spotcheck_review_events`、`spotcheck_adjudications`（均只追加） | **已是结构化**事件表（模型层注释明示「只追加不覆盖」），可直接做行为分析，当前仅治理闭环自用 |
| 前端埋点 | **不存在**（管理后台/复核门户无埋点 SDK），与姐妹系统一致 | 如需要须从零埋点 |

### B.4 非结构化配置 / JSON 字段（库内 JSON 与 Redis JSON，实有结构）

| 字段 | 实际结构 | 说明 |
|---|---|---|
| Redis `exam:{exam_id}:q:{qn}:question_analysis/rubric/rules`（预计算三件套） | UGP `QuestionContext`/`ReferenceAnswer` Pydantic JSON + 纯文本规则 | **考试×题目级共享缓存**（TTL 30d，`redis_cache_ttl=2592000`），成本结构核心；MySQL `exam_questions` 有持久副本；**粒度完全不含学生维度**（API 文档 §7） |
| `paper_task_details.usage_json` / `grading_task_details.usage_breakdown_json` | by_model 分项 + scenes 场景维（学生信息识别/客观题/主观填空/作文）+ 各 Agent 耗时 + 内存/CPU 快照 | 成本分析的最细粒度留痕，已支撑单卷成本报告的产出 |
| `grading_task_details.result_json` | UGP `GradeResult`+`ValidationResult`+`StudentResponse` 合并 JSON | 个体数据最厚处（见 A.2/B.2） |
| `grading_task_details.agent_details_json` | 五 Agent 执行详情；`question_analysis/grading_rubric` 两键以 `{"$snap": md5}` 占位 | 内容寻址去重（37:1）；运行期附件 `_usage_stats/_fallback_extraction` 等也在此 |
| `paper_task_details.request_json` / `request_blobs.content` | 原始提交（大键 `{"$blob": md5}` 占位） | 重跑/排障兜底；含花名册 PII（A.3 ⚠️） |
| `exam_details.questions_json/regions_json`、`samples.question_context_json`、`bench_runs.roster_json/summary_json`、`fails_report_bundle.bundle_json`、`dashboard_bundle.data_json` | 题目快照/区域坐标/候选项清单/看板产物 | 快照与打包性质，结构化价值低 |
| `config_overrides`（四命名空间）+ Redis `config:*` | settings/strategies/confidence/prompts 逐字双写 | 配置即数据；`config:test:*` 隔离命名空间是回归测试的配置叠加机制 |
| Redis `config:model_subject_overrides` / `config:subject_group_definitions` | 学科×角色覆盖 + 学科组定义 | 三级路由数据源（`_source` 字段标注命中层级） |

## C. 与记忆系统提案的映射（盘点视角的结论）

按[记忆系统提案](./memory-system-proposal.md) §6 的三档分类核对本仓库实况：

1. **现成可用（零改造）**：
   - ✅ `learning` 接口按学生聚合读（提案已列，本盘点确认其数据源就是 `grading_tasks`/`paper_tasks` 的 student_id 过滤）。
   - ✅ `optimization_history.notes` + `question_config_snapshots`——题目级评分标准演进留痕，是「学科评分标准基线」（`subject_group_memory`）的现成素材。
   - ✅ `samples.error_pattern` + `fails_settlement_records.fail_class`——**两套已存在的错误归因标签体系**（人工标注，覆盖窄但质量高），可直接作为蒸馏输出的对齐码表。
2. **需要蒸馏（数据在、缺抽取层）**：
   - `grading_task_details.result_json`（全量 AI 评语 + 分步得分 + 提取文本）→ 不含姓名的错误模式标签——**提案点名的第一候选在 grading-service 侧的对应物**（姐妹篇的 `student_answers` 是其下游落地）。推荐抽取点：rubric_optimizer 的聚合位（见 B.2 重点分析），顺带完成 `### {name}` → `### 样本{i}` 的去 PII 一行改造。
   - `grading_tasks.teacher_score/diff` → 教师评分尺度（稀疏，远不如 grading-system `score_audits`，两源应合并使用）。
   - `usage_breakdown_json` 成本明细 → 题目级批阅成本画像（对数据架构的成本归因有价值）。
3. **当前完全不存在**：学生/教师画像表、行为事件流（无埋点）、告警历史、日志持久化、按 student_id 的跨考试前导索引（慢查询风险，提案 §8 已记）。

## D. 与既有文档的不一致标注

- ✅ 与[批阅系统数据资产盘点](./data-assets-grading-system.md)：其 §C 对 `student_answers`/`score_audits` 归属的勘误（属 grading-system）经本盘点对侧确认——grading-service 26 张表中无此两表，亦无同义表；两仓库数据边界为「grading-service 产原始评分与评语 → grading-system 落地为 student_answers 双列 + score_audits 审计」。
- ✅ 与[API 文档](./api-and-process-grading-service.md) §6 学生字段盘点：本盘点 A.8 在其基础上补全了 `samples`/`discarded_tasks` 两张样本域表的学生字段，结论一致（全部元数据粒度，无画像）。
- ⚠️ 补充 API 文档 §2.2 一处表述精度：其称细则优化「学生姓名作为样本标签进 prompt」属实，但需补注——**该姓名仅存在于优化调用的内存 prompt 中，不落任何持久存储**（`_save_history` 只存细则与 notes），且改成 `### 样本{i}` 可零成本去除（本盘点 B.2）。
- ⚠️ 与[数据架构](./data-architecture.md) L1：`exam_questions.knowledge_points_json` 只是 AI 提取的知识点名称字符串数组，无图谱 ID/编码——批阅服务侧与「知识点权威源」的打通需要在提取或建考环节增加知识点对齐步骤，这是本盘点对该提案的具体补充（与姐妹篇 D 节校本库无标签字段的发现同性质）。
- ✅ 与[记忆系统提案](./memory-system-proposal.md)：提案 P0「learning 接口 + score_audits」路径与本盘点结论兼容；本盘点新增一个 P0 候选最小改造（rubric_optimizer 样本标题去 PII + 错误模式标签抽取）。

相关：[数据架构](./data-architecture.md) · [API 与业务流程](./api-and-process-grading-service.md) · [批阅系统数据资产盘点](./data-assets-grading-system.md) · [记忆系统提案](./memory-system-proposal.md)
