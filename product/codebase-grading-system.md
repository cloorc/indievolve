---
type: concept
---

# 批阅系统代码库调研报告（aireview-grading-system）

> 调研对象：`~/src/indievolve/aireview-grading-system`（GitHub: IndievolveLabs/aireview-grading-system，main 分支，最近提交 2026-08-18，约 1158 次提交）。
> 本报告基于代码事实整理，对应[部署架构](./deployment.md)中的「批阅系统」（学校租户面：建考/批阅/讲评）。文末第 8 节逐项对照知识库既有文档，标注吻合点与不一致点。

## 1. 项目定位与整体架构

**定位**：多租户智能批阅 SaaS 系统（内部代号 `grading_system`），业务闭环为「建考 → 题目提取/卷面确认 → 接收扫描件 → 自动批阅 → 三模式复核 → 学情 → 讲评 → 留痕打印」（仓库根 `CLAUDE.md`）。它本身**不含 AI 批阅算法**——AI 能力全部在外部的批阅服务（grading_service，独立代码库，见[批阅服务代码库调研](./codebase-grading-service.md)）；本系统的本质是「考试全生命周期的业务编排 + 教学数据资产沉淀」（`docs/DESIGN.md` §0 原话）。

**架构形态**：模块化单体（Modular Monolith）+ 独立异步 Worker 进程，单仓（monorepo）。`docs/DESIGN.md` 明确拒绝微服务化（「团队规模与一期周期不支持微服务」）。分期范围：

- **P0 多租户基建**：租户开通（运营后台）、账号体系、组织管理、文件直传、权限与数据范围、知识点种子。
- **P1 核心闭环**：建考→卷面确认→扫描接收→自动批阅→三模式复核→学情→讲评（在线+PPTX）→留痕批改卷 PDF。
- **不在范围**：出题组卷/知识库（P2）、学生端（P3）、管理空间/校园空间（P4）、原型标「规划预览」的功能。

注意：代码中实际已存在 P2–P4 的部分实现（见第 8 节对照），CLAUDE.md 的「不在范围」口径落后于代码现状。

**三个外部系统**（ADR-001，全经 `backend/app/infra/` 适配层）：

1. **预处理系统**（开放 API 推送整份学生扫描卷）；
2. **批阅服务 grading_service**（REST 提交 → 轮询）；
3. **学情分析系统**（同款 REST 对接）。

被调方向统一走 `external_tasks` 任务骨架：入队 → 限流提交 → 轮询 → 落库 → 重试 ≤3 → failed，用 `dedup_key` 做活跃去重（防止「自动批阅」重复付费）。数据底座口径：扫描件 → 批阅服务 → 本系统 `student_answers` → confirmed 后回流 event_raw 底座，底座不可直连批阅服务取数。

## 2. 技术栈（实际依赖文件读取）

### 后端（`backend/requirements.txt`，Python 3.12）

| 类别 | 依赖 |
|---|---|
| Web 框架 | FastAPI ≥0.104 + uvicorn[standard] |
| ORM/迁移 | SQLAlchemy 2.0 asyncio + aiomysql + Alembic（62 个迁移版本） |
| 校验 | pydantic v2 + pydantic-settings |
| 缓存/队列 | redis[hiredis] ≥5（注意：**未实际使用 ARQ**，设计文档写 ARQ，实现是自研 worker 轮询 `external_tasks` 表，见第 4 节） |
| 对象存储 | minio 客户端（S3 协议，生产对接阿里云 OSS） |
| 文档处理 | PyMuPDF（PDF 切页）、openpyxl（Excel 导入）、python-pptx（讲评 PPTX）、Pillow、numpy、opencv-python（答题卡图像处理） |
| 渲染 | playwright（组卷试卷→PDF 服务端渲染，无头 Chromium + 中文字体） |
| LLM | openai SDK（经 `modules/llm` 网关路由多模型） |
| 安全 | PyJWT（显式替代 python-jose，注释注明后者有 CVE-2024-33663/33664）、argon2-cffi（密码哈希）、cryptography（Fernet 加密 LLM key） |
| 内容审核 | alibabacloud-green20220302==3.2.4（阿里云内容安全，钉死版本防依赖冲突） |
| 测试 | pytest + pytest-asyncio + httpx + fakeredis |

### 前端（`frontend/package.json`）

- **Vue 3.5 + TypeScript + Vite 5 + vue-tsc**；UI：**shadcn-vue（reka-ui）+ Tailwind 3**（CLAUDE.md 铁律：一个项目一套 UI 库）。
- 状态/数据：Pinia 3、@tanstack/vue-query 5；表单：vee-validate + zod。
- 特色依赖：vue-konva/konva（批阅画布标注）、KaTeX（公式渲染）、@vue-office/*（docx/excel/pdf/pptx 在线预览）、html2canvas + jspdf（前端 PDF 导出）、dompurify（XSS 防护）。
- **桌面/移动壳**：Tauri 2（Windows exe）+ Capacitor 8（Android apk），同一份 Vue 代码多端出包（与[部署架构](./deployment.md)「一体机触屏 Web 端」「多端接入」对应）。
- 测试：vitest + @vue/test-utils（单测）、playwright（e2e，8 个用例文件）。

### 基础设施

MySQL 8 + Redis 7 + 阿里云 OSS（S3 协议）；本地开发用 docker-compose 起 mysql:13306 / redis / minio。设计文档提到 ECharts，但 package.json 中未见该依赖（前端现状未引入）。

## 3. 目录结构与模块划分

```
├── backend/                  # FastAPI 后端（438 个 .py）
│   ├── app/{config,infra,common,helpers}   # 基建：settings、OSS/DB/外部服务适配、雪花发号、错误信封
│   ├── modules/<域>/         # 业务模块（功能域切分，非类型分层），25 个域：
│   │   ├── platform  运营后台/租户开通/open API（含 open_routes、v2）
│   │   ├── iam       账号/登录（密码、短信、微信小程序/App/Web 扫码 ×4 种微信登录）
│   │   ├── org       组织（年级/班级/学生导入）
│   │   ├── exam      建考/卷面确认（orientation 姿态矫正、annotate_tasks 标注）
│   │   ├── scan      扫描件接收（open_routes 供预处理系统推送）
│   │   ├── marking   批阅/复核（grading_remote、review_rules、completion_guard、annotations）
│   │   ├── manage    管理空间（overview_service）
│   │   ├── analytics 学情、lecture 讲评、printing 留痕打印、kb 校本库
│   │   ├── knowledge 知识点（本系统是公司知识图谱属主，编号 SUBJ-NN-NN-NN）
│   │   ├── qbank/tiku 题库、llm LLM 网关、device 设备凭证、files 文件直传
│   │   ├── student 学生端、feedback 反馈、prefs 偏好、moderation 内容审核
│   │   ├── admin/syslog/tasks 平台管理/API 日志/外部任务
│   ├── db/{migrations,seeds} # Alembic 62 个迁移 + 幂等种子（菜单/字典/内置角色）
│   ├── tests/ (166 个测试文件) + tests_unit/ + loadtest/（locust）
├── frontend/                 # Vue3 SPA（96 个 .vue + 89 个 .ts）
│   ├── src/modules/<域>/     # 与后端同域：auth/exam/scan/marking/analytics/lecture/
│   │                         #   manage/campus/kb/student/admin/platform/teach/compose…
│   ├── src-tauri/（Windows 壳）、android/ 由 Capacitor 生成、e2e/（playwright）
├── deploy/                   # 泛域名证书签发脚本 + 宿主 nginx 配置（含真实域名规划）
├── load/                     # k6 压测 harness（KR1–KR5 验收标准）
├── scripts/                  # 批量建考、smartedu 题库探针/重建
├── docs/                     # 27 篇文档（DESIGN/SPEC/API 对接/测试计划/评审记录）+ plans/specs/superpowers
└── docker-compose.yml / docker-compose.scale.yml / update.sh / dc.sh  # 部署编排
```

## 4. 核心业务流程与关键代码入口

**入口**：`backend/app/main.py`（`create_app()` 逐一挂载 25 个模块的路由，区分内部 routes 与对外 open_routes）；中间件链：CORS（放行 localhost/私网/Tauri/Capacitor scheme）→ `TenantContextMiddleware`（按 Host 二级域名解析租户）→ `ApiLogMiddleware`（API 日志落库）。探针：`/health`（存活）与 `/readyz`（就绪，探 DB/Redis/OSS）。

**主链路**（与[部署架构](./deployment.md)主链路一致，代码落点如下）：

1. **建考/卷面确认**：`modules/exam`（建考异步化 + 自适应标注状态拆分有专项方案文档）。
2. **扫描接收**：预处理系统经 `/open/v1/scan/*` 推送（租户级 API Key + 幂等键），`modules/scan/open_routes.py`；硬件固件内置 URL 由宿主 nginx `/t/<租户>/prod-api/scan/upload/*` 注入 API Key 转发（`deploy/nginx-grade.indievolve.com.conf`）。
3. **自动批阅**：`modules/marking/grading_remote.py` 提交 grading_service；**异步 worker**（`backend/app/worker.py`）消费 `external_tasks`（kind=grade_paper）：pending→submit→submitted→poll→completed/failed，单副本运行、跨租户扫描、每 5 秒一轮（`GRADE_WORKER_INTERVAL`）。
4. **三模式复核**：「客观题确认 → AI 疑点 → 快速扫描」共用一套按「学生 × 题」推进的工作台；完成整场批阅必须写可查询记录（`review_completions` 表），撤销只允许最新一条且恢复快照、使既有打印产物失效（`modules/marking/completion_guard.py`，有专项单测 `tests_unit/test_completion_guard.py`）。
5. **学情/讲评/打印**：`modules/analytics`（调外部学情系统）、`modules/lecture`（在线讲评 + python-pptx 生成 PPTX）、`modules/printing`（留痕批改卷 PDF，`print_jobs` 表）。

**权限模型**（ADR-002，与[角色功能清单](./roles-and-features.md)交叉验证）：

- 有效权限 = `role_menus ∩ tenant_menus`（菜单两层：平台运营分配上限 / 学校管理员校内分配）+ `roles.data_scope`（年级×学科×班级数据范围）+ `delegations`（临时授权）。菜单与数据范围**正交、都要判**；JWT 携带已 ∩ 的菜单码集合。
- 人工批阅的角色开关是 `manage` 域专用授权：队列入口必须调 `resolve_manual_permission`，不能只靠前端显隐——这正是角色清单中「阅卷设置/人工批阅权限矩阵」的服务端实现。
- 初始密码服务端强随机生成、仅创建时返回一次，禁止从手机号派生；机器凭证（`open_api_keys` 系统级 / `device_credentials` 设备级）共用「前缀→哈希→status→scopes」鉴权、即时吊销。
- 登录方式 4 种：密码、短信验证码、微信小程序/App、Web 扫码（2026-08 新增，含学校域名隔离）；另有运营后台可配的**白名单固定密码**（`login.whitelist_phones` / `login.whitelist_password`，`modules/iam/service.py:72`）——角色清单中 demo 固定验证码 `250528` 即此机制。

**数据库铁律**（CLAUDE.md §数据库，与[数据架构](./data-architecture.md)相关）：全表 `tenant_id` 行级隔离且索引 `tenant_id` 打头；不建 DB 外键；软删哨兵 `'1970-01-01 00:00:00'`（禁止 `IS NULL`）；主键 BIGINT + 应用层雪花发号（41ms+5机器+6序列，≤ JS MAX_SAFE_INTEGER）；学生按 `student_no` upsert 终身一条。知识点 `knowledge_points` 是公司知识图谱主数据（编号 `POL-01-02-03` 式），迁移源为旧 continuow_admin 库的「爬来的那套」，政治学科试点先行——印证了数据架构提案中「本系统是知识点权威源」的判断。

## 5. 部署与运维

**部署形态**：单机 Docker Compose 多环境共存（test/demo/prod 三套同机，独立 compose project `grading_<env>` + 独立镜像 tag + 独立库/端口）。服务：migrate（一次性 alembic + 幂等种子）→ backend（多副本，nginx 经 Docker DNS 轮询）→ worker（单副本 profile）→ frontend（nginx 托管 SPA + 反代 /api）。MySQL/Redis/OSS 全部外部化（不在 compose 内）。

- **`update.sh <env>`**：一键更新——切分支 git pull → build → up -d --remove-orphans → 清理悬空镜像 → 跟踪日志。处理了 `--remove-orphans` 误删扩容副本的坑（SCALE_WEB=1 时必须叠加 scale 文件）。
- **`dc.sh`**：docker compose 参数包装（-p/--env-file/-f/--profile 四件套封装，避免手工漏参串环境）。
- **`docker-compose.scale.yml`**：横向扩容到 2 副本（backend 实例0 + backend-1），每副本独立 `SNOWFLAKE_WORKER_ID`（0/1，worker 用 4）防雪花撞号。副本数定为 2 是 2026-08-13 压测实测结论（283 份并发推卷四副本合计仅 1.33 核，2 副本各 67% 留余量），注释明确「4 副本纯浪费，且进程内缓存一致性变差」。
- **`docker-entrypoint.sh` 护栏**：`UVICORN_WORKERS>1` 且固定 `SNOWFLAKE_WORKER_ID` 时启动即拒，逼向多副本正确姿势。
- **生产环境守卫**（`app/config/settings.py`）：`APP_ENV=production` 时强制校验 JWT_SECRET 非默认且 ≥32 字节、LLM_ENC_KEY 非默认、签名密钥非默认，否则启动失败。
- **App 签名更新**：Ed25519 签名的更新清单（fail-closed，缺配置直接 503），下载域名白名单 + 证书 SHA-256 钉扎。
- **nginx/证书**（`deploy/`）：全站一张多 SAN 泛域名证书（`*.grade.indievolve.com` / `*.grade-demo.indievolve.com`），acme.sh + 阿里云 DNS 签发，自动续期并 scp 同步到中继服务器 47.111.107.63 双端 reload。租户识别在应用层（Host 二级域名），nginx 零租户逻辑、新开租户不改配置。
- **CI/CD**（`.github/workflows/build-apps.yml`）：仅一个 workflow，手动触发，一次产出 4 个包（prod/demo × Windows exe(Tauri)/Android apk(Capacitor)），发布到 GitHub Release `desktop-mobile-latest`。**没有自动化测试/ lint 的 CI**——pytest/vitest 不在流水线里跑。

**线上环境**（CLAUDE.md）：MySQL/Redis 在阿里云**新加坡**实例（白名单限制，本机不可直连），demo 库 `grading_system_demo` / prod 库 `grading_system`，Redis demo 用 DB2；配置在服务器 `8.219.170.206:/data/aireview-grading-system/.env.{demo,prod}`。涉及线上库的脚本一律在服务器上用独立临时容器跑，禁止 `docker exec` 进运行中容器跑长任务（曾连带打挂 backend/worker 致 demo 中断）。

## 6. 压力测试/负载测试情况

两套 harness + 完整验收标准（`load/README.md`）：

- **`load/kr12.js`（k6）**：KR1 普通接口 P99<500ms、KR2 300 RPS 错误率<0.1%，只压同步 CRUD 类接口（重接口单列）；支持 ramping-arrival-rate 到 300 RPS。
- **`backend/loadtest/locustfile.py`（locust）**：热接口（/health、/readyz、/auth/me、批阅进度轮询、分页 papers）P99<500ms、300 RPS 不宕机。
- **KR3 异常熔断**：已落地全局兜底异常处理器（统一错误信封 `{success,code,message}`、生产隐藏 detail、不裸抛栈）；故障注入用例待补。
- **KR4 容错**：围绕 external_tasks 状态机的故障注入矩阵（幂等/断点续/重试），规划中。
- **KR5 长效稳定**：72h soak（RSS/MySQL 连接/pool/Redis 无泄漏），规划中。
- 另有 `docs/TESTPLAN-500并发-执行计划.html`、`docs/TESTPLAN-批阅链路稳定性与并发.*` 与 `docs/deploy/scaling-and-load-test.md`（2026-08-13 实测数据，见第 5 节副本数决策）。

**观察**：压测工具与指标定义齐备且有实测数据支撑，但 KR3–KR5 的故障注入与 soak 仍是待办；压测入口靠手工跑，未进 CI。

## 7. 代码质量观察（仅报告事实）

**正面**：

- 测试规模可观：后端 `tests/` 166 个文件、约 1020 个测试函数（含租户隔离、登录限流、完成守护等专项），前端 vitest 单测 + 8 个 playwright e2e。但**无 CI 强制**，测试是否全绿靠自觉。
- 安全意识较好：argon2 密码哈希、PyJWT 替代有 CVE 的 python-jose、Fernet 加密 LLM key、生产密钥守卫、内部 Swagger 默认关闭（secure-by-default）、`EXPOSE_OPEN_DOCS` 默认关闭、软删/审计三件套、API 日志切面、初始密码强随机、机器凭证即时吊销、App 更新 Ed25519 签名 fail-closed。
- 文档密度高：27 篇 docs + 部署手册 + 三份测试计划 + 专项方案（答题卡定位配准、快扫判定规则、标注状态拆分等），关键坑（remove-orphans、雪花撞号、容器内跑长任务）都有复盘注释。
- 数据库纪律严格且成文（雪花发号、软删哨兵、无外键、tenant_id 打头索引）。

**问题/技术债**：

1. **真实 API Key 硬编码进仓库**：`deploy/nginx-grade.indievolve.com.conf` 第 75/86/96 行明文提交预处理系统 API Key `main-2cgxnaxqrm.5f74bf7e6d2c6b9c90dcaf1b417bdb3e`（注入 X-API-Key 头转发给 ingest 服务）。该文件还暴露了内网拓扑（172.17.0.1 各端口、中继 IP 47.111.107.63、服务器 8.219.170.206）。建议轮换该 Key 并移入部署期注入。
2. **demo 白名单固定密码机制**（`login.whitelist_password`）：运营后台可配的固定验证码/固定密码绕过了短信验证，是 demo 便利与生产风险的灰色地带——若该配置被带到 prod 即成为后门（当前由运营后台配置项控制，代码层面 prod 无强制禁用）。
3. **设计文档与实现漂移**：`docs/DESIGN.md`/`SPEC-P0-P1.md` 写 ARQ 队列、ECharts、uv/ruff/pnpm 工具链，实际是 DB 表轮询 worker、无 ECharts、pip/npm。旧文档未归档标注，新人易被误导。
4. **无自动化 CI 测试流水线**：唯一 workflow 只构建桌面/移动安装包；lint（ruff/eslint）配置存在但不强制。
5. **KR3–KR5 验收项未完成**（故障注入、72h soak），「当前无入站限流」（load/README 自述）。
6. 本地 `Settings()` 默认指向已废弃旧库（CLAUDE.md 自述的坑：本地跑写库脚本会静默写错地方），虽已成文警告，但默认值本身仍是陷阱。
7. `backend/tests_unit/` 仅 1 个文件，与 `tests/` 的双目录结构职责不清晰。

## 8. 与知识库现有文档的对照

### 与[部署架构](./deployment.md)

| 部署文档描述 | 代码事实 | 结论 |
|---|---|---|
| 批阅系统「后端 ×2 负载」 | docker-compose.scale.yml 固定 2 副本（backend + backend-1），注释注明 2026-08-13 压测实测定 2 | ✅ 吻合 |
| 「同步 Worker ×1（从批阅服务回拉结果）」 | worker 单副本 profile，`app/worker.py` poll_results | ✅ 吻合 |
| 批阅服务「内部，不对外开放」 | grading_service 仅经宿主 nginx `/grade-server/` 与后端出站调用 | ✅ 吻合 |
| 「RDS MySQL / 云 Redis」双可用区 | CLAUDE.md：阿里云新加坡 RDS MySQL + Redis | ⚠️ 部署文档未提**新加坡**区域，且未提「双可用区」在代码侧无法验证 |
| 「千问 3.7 Plus 大模型」 | 本系统不直接调大模型，经 `modules/llm` 网关 + 外部 grading_service；与[批阅服务调研](./codebase-grading-service.md)已指出的「多模型路由」一致 | ⚠️ 沿用批阅服务报告的修正口径 |
| 主链路「扫描→预处理→批阅系统→批阅服务→MySQL→同步 Worker 回拉」 | 与第 4 节代码链路逐环对应 | ✅ 吻合 |
| 多端「Web（三套环境）」 | test/demo/prod 三套 compose project 同机部署 | ✅ 吻合 |
| ALB/NLB 负载均衡、多 ECS、弹性伸缩 | 代码侧为**单机 Docker Compose** 部署（nginx + host.docker.internal + 中继服务器），未见多 ECS/云 LB 的 IaC 证据 | ⚠️ **不一致或待补充**：代码反映的部署形态是单机 + 中继，部署文档描述的云上多可用区形态可能在中继层/云上手工配置，仓库内无对应编排 |

### 与[角色功能清单](./roles-and-features.md)

| 角色清单描述 | 代码事实 | 结论 |
|---|---|---|
| 阅卷模式设置（AI 辅助/人工）+ 全校锁定 | `modules/manage` 专用授权 `resolve_manual_permission`，服务端强制，非前端显隐 | ✅ 吻合（且有 ADR-002 背书） |
| 校本库两级审核（初审/终审） | `modules/kb` 存在 | ✅ 吻合 |
| 管理空间 5 模块 + 校园空间 | `frontend/src/modules/manage`、`frontend/src/modules/campus` 均存在 | ⚠️ 与 CLAUDE.md「管理空间/校园空间 P4 不在范围」矛盾——**代码已先行实现**，范围口径需更新 |
| 学生端 | `modules/student` + `frontend/src/modules/student` 存在 | ⚠️ 同上（P3 口径落后于代码） |
| 备课/命题组卷（教师端） | `frontend/src/modules/teach`、`compose` 与后端 `qbank`/`tiku` 存在 | ⚠️ 同上（P2 口径落后于代码） |
| demo 固定验证码 250528 | `login.whitelist_password` 白名单机制（iam/service.py） | ✅ 机制吻合 |
| 「教务主任被路由守卫重定向」疑似缺陷 | 前端有 `ForbiddenView.vue` 与空间路由逻辑；属 demo 权限配置问题，代码机制本身支持细粒度菜单 ∩ | ➡️ 维持原结论（待产品复核配置） |

### 对[数据架构](./data-architecture.md)的补充

- 提案假设「知识点图谱是新建」，代码事实是：**本系统已是知识图谱属主**（`knowledge_points` 主数据 + 编号签发 + 政治学科迁移试点），L1 的权威源定位已被代码确认。
- 提案的 ClickHouse/向量检索在代码中**无任何踪迹**（分析查询仍走 MySQL），选型提案仍属前瞻。
- 「不建 DB 外键、软删哨兵、雪花 ID」等数据库铁律是数据架构落地必须遵守的既有约束。

相关：[部署架构](./deployment.md) · [角色功能清单](./roles-and-features.md) · [数据架构](./data-architecture.md)
