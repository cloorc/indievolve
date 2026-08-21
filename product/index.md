# Product — Indievolve 主要产品

- [overall-architecture.md](overall-architecture.md) — **全局软件架构总览（建议从这里开始阅读）**：2 个团队自研系统（批阅系统/批阅服务）+ 2 个外部依赖系统（预处理系统/学情数据系统）的目标定位、两个交叉点详解（提交-轮询契约、共享基础设施但不共享数据）、记忆系统在全局中的落点
- [indievolve-product.md](indievolve-product.md) — Indievolve 主要产品（首篇）
- [deployment.md](deployment.md) — 部署架构图：学校侧边缘设备 + 阿里云中心云
- [roadmap.md](roadmap.md) — 产品待办（2026-08-17 周会纪要）
- [roles-and-features.md](roles-and-features.md) — 演示站角色功能清单与权限对比（教师端/管理端/学生端）
- [data-architecture.md](data-architecture.md) — 数据架构提案（试卷库 / 知识点库 / 教材考纲；§3.5 新增基于两仓库代码实测的数据资产盘点总览，含对提案本身的修正与勘误记录）

- [codebase-grading-service.md](codebase-grading-service.md) — 批阅服务代码库调研报告（aireview-grading-service：架构/技术栈/批阅流程/成本/Prompt/部署）
- [codebase-grading-system.md](codebase-grading-system.md) — 批阅系统代码库调研报告（aireview-grading-system：架构/技术栈/核心流程/部署/压测/代码质量/与既有文档对照）
- [architecture-grading-system.md](architecture-grading-system.md) — 批阅系统软件架构深化（分层与依赖方向、25 域模块依赖图与 4 组循环依赖、数据模型、external_tasks 任务骨架、权限与多租户、前端架构、安全设计、技术债）
- [grading-system-exam-workflow.md](grading-system-exam-workflow.md) — 高中大型考试全流程用例清单（基于 aireview-grading-system 的 `modules/exam`/`scan`/`marking`/`manual_marking`/`analytics`/`lecture`/`printing` 七模块路由与状态机代码实测，2026-08-20 更新纳入此前遗漏的人工批阅模块）：按建考准备（含批阅方式AI/人工选择）→答题卡标注确认→现场扫描接收→**两条平行批阅路径**（阶段四/五：AI批阅+三模式人工核对；阶段四A：独立的人工批阅模块——题块分发/双评仲裁/租约领题/质量监控/问题卷处理，31个路由端点，27条用户用例）→学情分析→讲评→留痕打印共 9 个阶段，53 条用户用例，每条给出用户操作/预期结果/数据变化三要素；附贯穿全流程的权限拦截/状态机拦截/幂等去重/撤销强约束/人工批阅双层授权五类异常护栏，及人工批阅模块 4 项已确认的实现缺口（权限校验粒度不足/问题卷未真正隔离/草稿文案与实现不符/总分收敛两套口径）
- [manual-marking-rehearsal-checklist.md](manual-marking-rehearsal-checklist.md) — 交付前全流程演练分层检查清单（人工批阅·单日版·定稿v5）：与考试全流程用例清单的"阶段四A：人工批阅"配套，是该章节全部用例的一次端到端真人演练排期（10:30–18:00，5人分工+1名机动，覆盖语文/数学/英语/理综物化合卷4场考试、2班60人规模），按人员拆分任务表格并补全起止时间（部分时刻为原文档硬性节点：11:15/14:00检查点①/14:30精评分表截止/16:10检查点②，其余按时间区块估算切分）；含5类人为制造的异常卷（缺考/空白卷/客观涂改/单选多涂/问题卷）及其预期归宿、6项风险预案与剧本表字段规格

- [architecture-grading-service.md](architecture-grading-service.md) — 批阅服务软件架构深化（七层分层与依赖方向、五 Agent 流水线接口契约与编排、自研 Redis 队列引擎与 MySQL outbox/fence 一致性设计、MODEL_ROLES 多 Provider 路由、三存储分工、Prompt 注册中心、成本追踪、技术债）
- [api-and-process-grading-system.md](api-and-process-grading-system.md) — 批阅系统 API 清单与核心业务流程（完整内部/开放 API 分域清单、7 条业务路径端到端流程、每条路径到 LLM 调用点追踪、modules/llm 网关与 grading_service provider_manager 关系、用户上下文/记忆现状排查）

- [api-and-process-grading-service.md](api-and-process-grading-service.md) — 批阅服务 API 清单与核心业务流程（19 个路由文件约 205 端点分族清单含 scope 鉴权、submit-precut-paper 学生身份字段接收与去向逐项追踪、五 Agent + 题目提取/学生信息识别/细则优化/评阅报告全部 LLM 调用点的文件+行号与 prompt 组成、prompt_manager 动态插入能力评估、MySQL 学生字段盘点、题目级缓存与个性化关系分析；结论：学生身份仅作元数据落库，除花名册识别与细则优化样本标签外不进任何 prompt）

- [memory-system-proposal.md](memory-system-proposal.md) — 分层记忆管理系统提案（对齐角色功能清单的 8 类真实角色；记忆分三组——教育类/个人偏好类/学生个人学业记忆，附组织层级传导规则图：校长→学科组→备课组→任课教师逐级广播+默认值继承，班主任跨学科横向广播，学生记忆严格点对点隔离不进入广播体系；含每角色记忆点清单、L1-L3 分层架构、user prompt 尾部注入设计、可复用数据盘点与分期建议）
- [data-assets-grading-system.md](data-assets-grading-system.md) — 批阅系统全量数据资产盘点（64 张表分域清单：组织与人/考试题目/批阅复核/学情聚合/校本库/偏好/学生端；非结构化四类：OSS 文件与 key 规范、自然语言文本、日志事件流、JSON 字段，逐类评估可否流水线结构化及现状；量级估算；与记忆系统提案的数据源映射与归属勘误）
- [data-assets-grading-service.md](data-assets-grading-service.md) — 批阅服务全量数据资产盘点（26 张表分组清单：考试题目/批阅任务结果/内容寻址去重/成本用量报告/优化历史/样本回归抽查裁定/配置凭证，含 6 张表 student_id/student_name 字段逐表确认——全部元数据粒度无画像；非结构化四类：OSS 文件与共享桶 key 规范、自然语言文本、日志事件流、JSON 字段，逐类评估可否流水线结构化及现状；量级锚点 38.6k 任务实测；rubric_optimizer 样本摘要文本重点分析——### {student_name} 标题仅在内存 prompt 不落库，改样本{i} 可零成本去 PII；与记忆系统提案的数据源映射及两处补充标注）

- [multi-tenant-data-architecture.md](multi-tenant-data-architecture.md) — 100 校规模多租户数据架构提案（PostgreSQL 优先）：基于已验证量级锚点推算容量（年增约 3,840 万行）；Hash(tenant_id) 两级分区 + RANGE 学期子分区 + Whale 租户逃生舱；行级安全(RLS)作为应用层 tenant_guard 的数据库级第二道防线；JSONB/GIN 承接现有半结构化字段，pgvector/HNSW 一个扩展覆盖相似题查重/错误模式聚类/校本库语义检索三个需求；OLTP/OLAP 三级递进分离策略（只读副本物化视图 → 列存扩展 → 独立 ClickHouse 触发式升级，非默认必需），并标注与记忆系统 L2 蒸馏任务的技术交集

- [adaptive-learning-engine.md](adaptive-learning-engine.md) — 自适应学习引擎（Adaptive Learning Engine，高中全学科自适应教学系统）产品线调研，2026-08-20 完成章节结构重组（原 `self-learning-engine.md` 更名并按主题重新分章）。§1 产品概览：目标/客群/优劣势（新增内容，转引"对外产品理念介绍""利益相关者与市场结构""师生双主体用户画像"三份原始文档：一句话理念"让每一个县城孩子都有人告诉他下一步该练什么"、四步证据链定义"学会了"、四个易混淆的市场单位竞争/采购/续费/数据可得、11类利益相关者KPI冲突矩阵、四条主动约束不承诺提分/不替代老师判断/不贴标签/教师数据不用于考核、真实落地规模2026-08甘肃定西驻校验证5所付费学校）；§2 三条演进线的真实关系与当前定位（原§6审计发现+§7.1-7.3定位判断合并，先事实后结论；§2.4 platform-data-foundation-service 与早期主线真实集成关系的完整更正说明+目标架构三通道通信模型（同步查询/异步Outbox+CloudEvents事件/批量manifest）vs当前实现差距对照表（事件通道仅底座侧已实现、ALE侧未见订阅消费代码；学校上传入口/MinIO对象存储中转均查无实据，系用户对目标架构的合理推断而非文档记载；outbox+CloudEvents是云厂商中立的分布式一致性设计非AWS/Azure特定选型，项目实际基础设施为阿里云）+API参考+§2.4.1专属运行时进程图；§2.5 五项遗留风险清单去重合并）；§3 Adaptive Learning Engine 代码与产品实现（§3.1 顶层45份方案文档主题分组8组，提升为本章第一节；§3.2 v1早期主线文档体系+本地engine试点代码；§3.3 v2 adaptive-learning-engine-lab完整内容——隔离编排纪律/engine代码现状/九微服务逐一实现现状〔L2-L6分层6739行核心代码〕/产品设计文档/产品交互形态含§3.3.5.1 static-pages业务功能清单/§3.3.6 platform-data-foundation-service公司数据底座完整介绍）；§4 核心算法与运行时架构（六轴模型/决策路由P0-P16/五分量选题公式/3+N聚类/五条设计宪法、"推荐系统"概括澄清、七阶段输入输出详表、算法实现细节含半衰期遗忘模型与Kish有效样本量精确公式、v2运行时权威进程视图15进程）；§5 与批阅系统产品线关系（2026-08-20更正：此前"未见直接耦合"结论不准确，经核实 `rubric-svc` 自称"上游防腐层"只读镜像批阅评分结构、`pipelines/ingest_grading.py` 真实调用批阅系统 `/grade-server/api/v1/learning/grading-results` 端点摄入评分结果为ALE证据，`source_task_id`复用批阅`task_id`做幂等键——实际是证据层单向摄入管道而非零耦合；知识资产层仍确认零共享，§5.1-5.4分层说明组织独立/证据层摄入/资产层零耦合/三层级关系区分）；§6 教育领域AI工作台候选框架选型调研（Work Buddy/千问办公闭源排除+OpenCode/Goose/DeepSeek Harness开源候选优劣势对比，推荐顺序Goose>OpenCode>DeepSeek Harness）。配套5张SVG图表同步改名为 `adaptive-learning-engine-*.svg` 前缀。
