---
type: concept
---

# 批阅服务代码库调研报告（aireview-grading-service）

对 `aireview-grading-service` 仓库（内部批阅服务，即[部署架构](./deployment.md)中"批阅服务：后端×1 + 批阅Worker×3"所指的系统）的全面代码调研。调研时间 2026-08-18，基于仓库 main 分支最新提交（`f3dddc8`，2026-08-17）。

## 1. 项目定位与整体架构

**定位**：统一的 AI 试卷批阅服务（内部版本号 v4.1），承载全部批阅逻辑，不直接对客户开放。上游接收批阅系统下推的预切割试卷图片与题库，下游把各题得分与批语写回，由批阅系统的同步 Worker 回拉（与[部署架构](./deployment.md)主链路一致）。

**架构形态**：单代码库、三角色运行——

```
单机模式：  [Master (API + Worker, role=all)]  ←→  Redis / MinIO / MySQL
分布式模式：[Master (纯 API, role=master)]  ←→  Redis / MinIO / MySQL
                ↑ 共享同一 Redis 队列 + 对象存储
            [Worker-01]  [Worker-02]  [Worker-03] ...（role=worker，无状态水平扩展）
```

- 角色由 `config/settings.py` 的 `server_role`（`all`/`master`/`worker`）控制，Docker Compose 用 `mono`/`split` 两个互斥 profile 切换（`docker-compose.yml`）。
- 同一进程内 API 与 Worker 并存（all 模式）；分布式模式下 Worker 从 Redis 队列抢任务，加入/退出对 Master 透明（`docs/distributed-deploy.md`）。
- **存储三分工**：MySQL 是终态主存（连接失败拒绝启动，`main.py` `_init_mysql_or_fail`）；Redis 是任务队列 + 缓存 + 动态配置投影；MinIO/OSS 存试卷图片。Redis 投影缺失时由 Worker 以 MySQL `pending` 行作为 durable outbox 对账补投，终态写 fence key 防重跑（`CLAUDE.md` Redis Key 规范）。

## 2. 技术栈

| 层 | 技术 | 来源 |
|---|---|---|
| 后端 | Python 3.12 + FastAPI + Uvicorn（端口 7732），Pydantic v2 / pydantic-settings | `requirements.txt`、`main.py` |
| 前端 | Vue 3（Composition API）+ Vite + Vue Router + KaTeX，SPA 含监控页 `/monitor`、配置页 `/config` | `frontend/package.json`、`CLAUDE.md` |
| 队列 | Redis 自研任务管理器（**无 Celery**），协程并发 + 信号量限流 + 宕机恢复 | `core/queue/task_manager.py`（约 5700 行） |
| 关系库 | MySQL（SQLAlchemy 2 + aiomysql），终态存储、配置主存 | `core/db/`、`requirements.txt` |
| 对象存储 | MinIO（主）+ 阿里云 OSS（迁移方向，`oss2`） | `requirements.txt`、`core/` |
| LLM 接入 | OpenRouter（主聚合网关）+ Gemini 直连 + 通义千问 DashScope + 硅基流动，四 provider | `core/provider_manager.py` |
| 图像/PDF | PyMuPDF 切割、OpenCV（CLAHE/降噪/歪斜矫正）、Pillow | `requirements.txt`、`CLAUDE.md` |
| 其他 | tenacity 重试、json-repair（模型输出容错）、sympy（数学等价）、rdkit（化学结构式判等）、paramiko（SSH 部署 Worker）、Chromium（评阅报告 HTML→PDF） | `requirements.txt`、`Dockerfile` |

**异步模型**：全链路 `async/await`；CPU 密集（图像预处理、PDF 渲染）跑默认线程池，启动时按容量模型扩池（`main.py` lifespan + `core/capacity.py`）。

## 3. 目录结构与模块划分

| 目录 | 职责 |
|---|---|
| `api/routes/` | 19 个 FastAPI 路由：exam（考试/题库）、paper（整卷批阅）、grading、task、dashboard、deploy（SSH 部署 Worker）、auth、report、samples（样本库）、external_api（对外结算）、learning（学情）、review_portal/spotcheck_portal（复核/抽检查）、fails_admin/fails_settlements（问题样本裁定）等 |
| `api/schemas/` | Pydantic 请求/响应模型 |
| `config/` | `settings.py`（Pydantic BaseSettings，全部配置项带默认值）+ `.env` |
| `core/objective/` | 客观题：VLM 识别 + 自适应投票（recognizer/grader/answer_loader/data_quality_monitor） |
| `core/subjective/` | 主观题：5-Agent 流水线（`agents/`：question_analyzer→answer_extractor→reference_generator→grader→validator）+ 编排器 `grading_service.py` + 细则自动优化 `rubric_optimizer.py` |
| `core/ai/` | LLM 客户端（OpenAI 兼容客户端、Gemini 客户端、key 池轮询、schema 工具、学生信息识别） |
| `core/scheduler/` | 整卷批阅管道 `paper_pipeline.py`（约 2700 行）：切分子任务→投递→汇聚→触发细则优化 |
| `core/queue/` | Redis 异步任务队列 + 进度追踪 + 宕机恢复 |
| `core/strategies/` | 学科自适应策略、置信度传播 |
| `core/chem/`、`core/defenses/`、`core/security/` | 化学结构判等、参考答案校验/注入防护、安全 |
| `core/db/`、`core/cache/`、`core/file/`、`core/exports/`、`core/reports/`、`core/sample_lib/` | MySQL 引擎、Redis 客户端、存储、导出、评阅报告、样本库 |
| `core/provider_manager.py` | 服务商与"模型角色"（MODEL_ROLES，19 个角色）管理 |
| `core/subject_model_router.py` | 学科×模型角色路由（单学科 > 学科组 > 角色默认三级 fallback），支持随学科下发思考开关 |
| `core/prompt_manager.py` | 提示词注册中心，`shared:*` 共享命名空间（注入防护/LaTeX 规范/OCR 纠错/涂改规则） |
| `data/` | few-shot 示例仓库、问题样本复核数据 |
| `frontend/` + `static/dist/` | Vue 3 SPA 源码与构建产物（多阶段 Docker 构建注入） |
| `scripts/` | 大量运维/评测脚本（默认 gitignore，白名单例外如 `install_worker.sh`、`update_model_cost_md.py`） |
| `docs/` | 60+ 篇设计/复盘文档（迭代记录、误差分析、规格、迁移方案） |
| `tests/` | 140+ 个 pytest 测试文件 + `regression/` 回归测试框架 |

## 4. 核心批阅流程

**任务流**（`CLAUDE.md`）：`submit_paper → paper_pipeline 切分子任务 → task_manager 异步执行 → 结果汇聚`。

1. **建考试/题库**：`POST /exam/create-with-questions`（外部题库下推）或 `POST /exam/extract-questions`（从试卷/答案 PDF 用 AI 多模型投票提取题目），随后后台**预计算**主观题的题目分析 + 评分细则 + 提取细则，缓存到 Redis（`exam:{exam_id}:q:{qn}:rubric` 等）。
2. **提交试卷**：`POST /exam/submit-precut-paper`，上游（Java 侧批阅系统）按题切好的区域图片 URL 下推，创建 paper 任务并入分布式队列。
3. **客观题**：VLM 识别答题卡图片 → 自适应投票（默认单次识别，实测单次 99.86% ≈ 5 次投票 99.88%，成本仅 40%）→ 与标准答案比对。低置信度题触发交叉验证。
4. **主观题 5-Agent 流水线**：题目分析（Agent1，预计算）→ 答案提取（Agent2，视觉 OCR，Pass1/Pass2 双独立提取 + 仲裁）→ 参考答案生成（Agent3，预计算）→ 自适应评分（Agent4，按置信度决定 1/3/5 次采样）→ 质量验证（Agent5，交叉验证，学科策略决定是否启用）。填空题可"一步直评"合并提取+评分省一次调用。
5. **汇聚与优化**：整卷完成后汇总得分、token 与费用；N 份（默认 3）完成后自动触发评分细则优化（`rubric_optimizer.py`）。

**模型调用**（与[部署架构](./deployment.md)对照，详见第 10 节）：

- 代码层面 provider 是**四家并存**：OpenRouter（聚合 Claude/GPT/Gemini 等）、Gemini 直连、**通义千问 DashScope**、硅基流动（`core/provider_manager.py` `PROVIDER_DEFINITIONS`）。默认模型角色以 OpenRouter 上的 Claude/GPT/Gemini 为主（如评分默认 `openai/gpt-5.2-pro`、验证默认 `anthropic/claude-opus-4.6`）。
- **千问确实在册且承担生产角色**：5 月批次迭代中客观题固定用 `qwen3.6-plus`（`docs/model_cost_iteration_summary.md`）；价格表内置 `qwen3.6/3.7/3.8` 全系列定价（`core/ai/openai_compatible_client.py`）。
- **学科级自定义模型已实现**：`core/subject_model_router.py` 支持按学科/学科组覆盖任意模型角色，2026-08-17 合并的 `feat/subject-level-thinking-config`（`9c75175`）支持**学科级思考开关**——同一模型在不同学科可分别开关思考（代码注释原话："`qwen3.7-plus` 语文开、化学关"）。
- **思考模式开关**：OpenRouter 走 `reasoning` 字段；千问 DashScope 走 `enable_thinking`/`thinking_budget`，且**关闭时必须显式传 `{"enable_thinking": false}`**——qwen3.x-plus 默认开思考，省略参数即开启（`openai_compatible_client.py` `build_thinking_payload`）。

**产出**：各题得分 + 批语（feedback）+ 置信度 + 每模型 token/费用（usage），写入 Redis 并落 MySQL；`GET /paper/paper-results/{paper_task_id}` 返回 `total_score`/`objective_score`/`subjective_score` 与 `usage.total_cost_cny`。

## 5. 成本与性能

### 5.1 单卷成本分析报告（`AI批阅服务单卷成本分析报告.md`）

对一份高三英语卷（150 分制，55 道客观题 + 10 道填空 + 应用文 + 续写）同卷批阅 3 次验证配置调整：

- **结论**：三次批阅总分完全一致（118.0），12 道主观题每题得分零偏差；通过三项配置变更，**单卷成本从 $0.47 降至 $0.25（约 ¥1.8），降幅 47%**，且不影响评分质量。
- **成本结构**（第 3 次）：学生信息识别 $0.0296（11.8%）、客观题 $0.0618（24.7%）、主观填空 $0.1201（47.9%）、作文 $0.0390（15.6%）。主观题答案提取（Claude Sonnet 4.6）是最大单项支出；评分用 Gemini Flash Lite 极低成本（每题约 $0.0014）。
- **三项优化**：客观题投票 5 次→1 次（省 80%）；学生信息投票模型 Opus→Haiku；花名册 prompt 19 班→1 班。
- **耗时**：单卷约 56-67 秒。

### 5.2 与"客观题 1-2 分钱/份"的对照

- 上述英语卷报告的客观题成本 $0.0618 ≈ **¥0.44/份**，明显高于周会口径的 1–2 分钱——但该报告用的是 OpenRouter 上的 Claude Sonnet 4.6 主识别 + Gemini 3.1 Pro 交叉验证。
- 5 月批次迭代中客观题固定用 **qwen3.6-plus**：`docs/model_cost_iteration_summary.md` 五次迭代客观题成本 $0.12–$0.22 / 27–45 份 ≈ **$0.004–0.005/份 ≈ ¥0.03–0.04/份（3-4 分钱）**，与周会"1–2 分钱"在量级上基本吻合（走 DashScope 直连价更低，且该口径可能只算了识别主调用）。
- 综合判断：周会口径对应"千问承担客观题识别"的生产配置；仓库证据支持该数字的量级，但精确值取决于投票次数与是否直连。

### 5.3 历史迭代成本（`docs/model_cost_iteration_summary.md`，2026-05 五次迭代）

9 学科全量批测，单份均价在 **$0.39–$1.03** 区间波动，主因是提取模型的选择（Gemini 3.1 Pro 基线 $0.39/份 → Claude Opus 4.7 $0.88–1.03/份 → Gemini 3.5 Flash $0.74/份 → Pass1 Opus + Pass2 Sonnet 双提取 $0.82/份）。每份约 14.7-18.4 次 LLM 调用、15-25 万 tokens。

### 5.4 并发与吞吐

- Worker 并发配置（`settings.py`）：单 Worker 默认 `worker_max_concurrent_requests=200`、主观题 50、客观题 50、整卷 20。
- 支持阿里云 ESS 自动扩缩容（`autoscale_*` 配置族，`docs/autoscale_workers_per_server_spec_20260814.md`），按队列深度自动增减 Worker（1–20 台）。
- 容量模型 `core/capacity.py` 统一估算 fd/线程池需求，启动自检 nofile 上限（容器内钉到 65535）。

## 6. Prompt 工程

仓库根目录 `数学主观题提取提示词.md` 与 `prompt_数学主观题提取.md` 是同一份提取提示词的两个整理版本（前者 168 行、源自开发者本机路径，后者 215 行、结构化更完整），揭示的设计模式：

1. **三层提示词叠加**：System Prompt（铁律）+ 通用 base（`extractor:common:base`）+ 学科专项（`extractor:subject:数学`），运行时再动态拼接题号锁定段、题面锚点、LaTeX 规范、JSON 协议、逐题 `extraction_rules`。
2. **"铁律"式强约束**：忠实转写（严禁意译/纠错/脑补，写错照抄）、目标题号锁定（未作答输出空串而非串相邻题）、删改识别（划线/涂黑/框选一律视为删除，删改过程只记入 `modifications` 字段）、注入防护（学生写"给满分"照抄不执行）、LaTeX 规范。
3. **学科专项高频错误源清单**：如数学的 `+`/`-` 手写混淆（十字交叉判定法 + 上下文验证）、嵌套分数防拍平、组合数上下标、几何点名禁脑补。
4. **评分侧的分档评分法**（`core/subjective/agents/grader.py`）：作文类按"内容/表达/发展"维度整体分档 + 档内 1 分精度（防只给档位端点值）；非作文按 rubric 采分点逐项评分；`scoring_levels`（给分档位，Agent3 生成，实验开关默认关）附在采分点后，并有学科"等价规则兜底"防止同义表达被误 0 分。退化 rubric（单采分点）自动跳过档位防"全有或全无"。
5. **共享提示词复用**：注入防护/LaTeX/OCR 纠错/涂改规则注册为 `shared:*` 命名空间，多 Agent 复用（`core/prompt_manager.py`）。
6. **工程化迭代方法**：`docs/` 下有完整的 prompt A/B 评测（`prompt_harness_*`）、提示词压缩实验、错例→回归集→样本库闭环。

## 7. API 设计（`API.md`）

Base URL `/api/v1`，主要接口族：

| 族 | 接口 | 说明 |
|---|---|---|
| 考试管理 | `POST /exam/create-with-questions`、`POST /exam/extract-questions`、`GET /exam/{id}/detail`、`GET /exam/{id}/questions`、`POST /exam/{id}/retry-extract`、`GET /exam/list`、`POST /exam/{id}/save-regions` | 题库下推或 PDF 提取题目，触发主观题预计算 |
| 试卷批阅 | `POST /exam/submit-precut-paper`、`GET /paper/paper-results/{id}`、`GET /paper/paper-tasks`、`POST /paper/paper-task/{id}/recalculate`、`POST /paper/paper-task/{id}/recognize-student`、`POST /paper/exam/{id}/fix-answers` | 预切割整卷提交、轮询结果（含分数/批语/费用）、客观题答案批量修正重算 |
| 任务管理 | `GET /task/paper-groups` 等 | 服务端分页、按批次/状态/学科筛选 |

典型流程：建考试（提取题目+预计算）→ 保存答题区域 → 循环提交学生试卷 → 轮询结果。返回中包含完整的模型用量与人民币成本（`usage.total_cost_cny`），支持按批次聚合成本——这正是成本报告的数据来源。另有内部运维接口族（dashboard/deploy/logs/config/auth/samples/复核门户/问题样本裁定/对外结算）散见于 `api/routes/` 的 19 个路由文件。

## 8. 部署与运维

- **Docker 多阶段构建**（`Dockerfile`）：Node 20 构建 Vue 前端（构建期注入 `VITE_BASE_PATH=/grade-server/` 反代子路径）→ Python 3.12-slim 运行时（含 Chromium + noto-cjk 字体用于评阅报告 PDF 渲染）。
- **Compose 双 profile**（`docker-compose.yml`）：`mono`（单容器 all）与 `split`（master + worker×N）互斥；显式钉死容器子网防与内网 172.x 宿主机网段冲突（注释记录了"整机对外失联"事故教训）；nofile 65535；worker 不映射宿主机端口、master 按容器 IP 直连。
- **update.sh**：一键更新 = `git reset --hard origin/main` + 自更新重执行 + 前端构建自检（校验资源前缀，防 404）+ 形态切换前清理旧容器/旧 nohup 进程 + 网段自动避让。带 `-m/-w` 参数即角色拆分模式，如 `./update.sh -m 1 -w 3` 即"后端×1 + 批阅Worker×3"。
- **watchdog.sh**：进程级守护——持续监控 `python3 main.py`，正常退出/SIGTERM 5s 后重启，崩溃（含 OOM Kill 137）15s 起指数退避至 60s，日志 50MB 轮转。**说明的运维假设**：服务预期会崩溃/OOM（高并发下内存压力大，`main.py` 启动日志也明示"内存保护已启用（切割阈值60%，批阅阈值75%，watchdog重启）"），用快速自动重启 + Redis 队列宕机恢复（重启扫描 processing 任务重新入队）+ MySQL outbox 对账来保证任务不丢。
- **Worker 扩容**：Web 界面填 SSH 信息一键部署（Master 通过 paramiko 远程安装），或手动 `install_worker.sh` 一行命令；支持阿里云 ESS 自动弹性伸缩。
- **告警**：飞书 webhook（`feishu_webhook_url`）+ `core/alerting.py`。

## 9. 测试与 CI

- **测试**：`tests/` 下 140+ 个 pytest 文件（pytest + pytest-asyncio），覆盖面相当广：LaTeX 清洗/修复、多模型投票、千问思考参数与成本核算（`test_qwen_thinking_payload.py`、`test_qwen_cost_accounting.py`）、任务/试卷投递恢复、配置快照、权限中间件、OSS 路由、复核门户、样本库、A/B 结算、自动扩缩容等。另有 `tests/regression/` 自带回归框架（jsonl 用例 + evaluator + runner）。运行方式为 `pytest`（仓库无 pytest.ini/pyproject 配置，按默认发现规则）。
- **CI**：`.github/` 下**只有 ISSUE_TEMPLATE**（bug_report/feature_request），**没有任何 GitHub Actions 工作流**——测试与发布不在 CI 中强制，发布靠 `update.sh` 强制同步 origin/main + 前端产物自检。这是一个值得注意的工程缺口。

## 10. 与知识库现有文档的对照

对照[部署架构](./deployment.md)（2026-08-17 周会口径）逐项核验：

| 周会/文档口径 | 代码证据 | 结论 |
|---|---|---|
| 批阅服务：后端×1 + 批阅Worker×3 | `update.sh -m 1 -w 3` / `docker compose --profile split --scale grading-master=1 --scale grading-worker=3` 完全对应 | ✅ 吻合 |
| 内部系统，不对客户开放，承载全部批阅逻辑 | API 设计面向"Java 侧批阅系统"下推（`paper_id` 注释即"Java 侧试卷 ID"），另设 `external_api` 路由做对外结算 | ✅ 吻合 |
| 调用千问大模型完成批阅 | 千问（DashScope）是四个 provider 之一，5 月迭代中客观题固定用 qwen3.6-plus；但**默认模型角色以 OpenRouter 上的 Claude/GPT/Gemini 为主**（评分默认 GPT-5.2 Pro、验证默认 Claude Opus 4.6） | ⚠️ **部分不吻合**：周会口径把批阅服务描述为"调千问"，实际是多模型体系，千问主要承担客观题识别；主观题提取/评分/验证大量使用 Claude/Gemini/GPT（经 OpenRouter）。建议 deployment.md 修正为"多模型路由：客观题以千问为主，主观题经 OpenRouter 调 Claude/GPT/Gemini，支持学科级模型与思考开关配置" |
| 千问 3.7 Plus | 价格表内置 `qwen3.7-plus` 定价；学科路由注释以 qwen3.7-plus 为例 | ✅ 吻合（代码支持该型号，具体生效配置在 Redis/MySQL 动态配置中，仓库内不可见） |
| 分学科自定义模型与评分细则 | `core/subject_model_router.py` 学科×模型角色三级路由 + 每题独立评分细则（`exam:{exam_id}:q:{qn}:rubric`） | ✅ 吻合 |
| 语文/数学开启思考模式 | 2026-08-17 合并的 `feat/subject-level-thinking-config` 支持学科级思考开关，注释举例即"qwen3.7-plus 语文开"；DashScope 侧关闭需显式传参 | ✅ 吻合（机制就绪且刚上线，与周会同日） |
| 客观题约 1–2 分钱/份 | qwen3.6-plus 承担客观题时约 ¥0.03–0.04/份；用 Claude/Gemini 时约 ¥0.44/份 | ✅ 量级吻合（对应千问配置） |
| 入库 50–60 份/分钟，峰值 75 份/分钟 | 仓库内无直接压测数字；单 Worker 默认整卷并发 20、请求并发 200，3 Worker × 20 = 60 份并发额度，与 50–60 份/分钟（单份约 60s）在稳态吞吐上自洽 | ✅ 自洽（代码无法直接证实，但容量模型支持） |
| Redis + MySQL | Redis 队列/缓存/配置投影 + MySQL 终态主存，分层 TTL 与对账机制完善 | ✅ 吻合且比文档描述更细致 |

**建议对 deployment.md 的修正**：外部依赖一节"千问 3.7 Plus 大模型"可补注批阅服务实际是多 provider 多模型路由（OpenRouter/Claude/GPT/Gemini + 千问 DashScope），千问主要承担客观题识别与部分学科角色，学科级模型与思考开关可在配置页动态调整（2026-08-17 上线学科级思考开关）。

## 11. 其他值得记录的事实

- **无 CI**：`.github/` 仅 issue 模板，测试/发布无流水线强制，质量靠本地 pytest + 样本库回归 + `update.sh` 自检。
- **运维哲学偏"自愈"**：watchdog 退避重启 + 队列宕机恢复 + MySQL durable outbox + 终态 fence，层层兜底"进程随时可能死"的假设。
- **成本可观测性好**：每次 LLM 调用记录 token/费用并聚合到试卷/批次粒度，内置全模型价格表（含千问各代际人民币定价与缓存价），支撑了仓库内多份成本报告的产出。
- **治理功能超出"批阅"本身**：样本库、人工复核门户、抽检查裁定、问题样本在线裁定（默认关）、对外 API key 结算等，说明该服务同时是批阅质量的回归与运营平台。
- `README.md` 的"模型配置"表（GPT-5.2 Pro / Gemini 3 Pro / Claude Opus 4.5）与 `provider_manager.py` 的当前默认值（Opus 4.6 等）略有滞后，README 更新不如代码快。

相关：[部署架构](./deployment.md) · [角色功能清单](./roles-and-features.md) · [数据架构](./data-architecture.md)
