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

- [architecture-grading-service.md](architecture-grading-service.md) — 批阅服务软件架构深化（七层分层与依赖方向、五 Agent 流水线接口契约与编排、自研 Redis 队列引擎与 MySQL outbox/fence 一致性设计、MODEL_ROLES 多 Provider 路由、三存储分工、Prompt 注册中心、成本追踪、技术债）
- [api-and-process-grading-system.md](api-and-process-grading-system.md) — 批阅系统 API 清单与核心业务流程（完整内部/开放 API 分域清单、7 条业务路径端到端流程、每条路径到 LLM 调用点追踪、modules/llm 网关与 grading_service provider_manager 关系、用户上下文/记忆现状排查）

- [api-and-process-grading-service.md](api-and-process-grading-service.md) — 批阅服务 API 清单与核心业务流程（19 个路由文件约 205 端点分族清单含 scope 鉴权、submit-precut-paper 学生身份字段接收与去向逐项追踪、五 Agent + 题目提取/学生信息识别/细则优化/评阅报告全部 LLM 调用点的文件+行号与 prompt 组成、prompt_manager 动态插入能力评估、MySQL 学生字段盘点、题目级缓存与个性化关系分析；结论：学生身份仅作元数据落库，除花名册识别与细则优化样本标签外不进任何 prompt）

- [memory-system-proposal.md](memory-system-proposal.md) — 分层记忆管理系统提案（对齐角色功能清单的 8 类真实角色；记忆分三组——教育类/个人偏好类/学生个人学业记忆，附组织层级传导规则图：校长→学科组→备课组→任课教师逐级广播+默认值继承，班主任跨学科横向广播，学生记忆严格点对点隔离不进入广播体系；含每角色记忆点清单、L1-L3 分层架构、user prompt 尾部注入设计、可复用数据盘点与分期建议）
- [data-assets-grading-system.md](data-assets-grading-system.md) — 批阅系统全量数据资产盘点（64 张表分域清单：组织与人/考试题目/批阅复核/学情聚合/校本库/偏好/学生端；非结构化四类：OSS 文件与 key 规范、自然语言文本、日志事件流、JSON 字段，逐类评估可否流水线结构化及现状；量级估算；与记忆系统提案的数据源映射与归属勘误）
- [data-assets-grading-service.md](data-assets-grading-service.md) — 批阅服务全量数据资产盘点（26 张表分组清单：考试题目/批阅任务结果/内容寻址去重/成本用量报告/优化历史/样本回归抽查裁定/配置凭证，含 6 张表 student_id/student_name 字段逐表确认——全部元数据粒度无画像；非结构化四类：OSS 文件与共享桶 key 规范、自然语言文本、日志事件流、JSON 字段，逐类评估可否流水线结构化及现状；量级锚点 38.6k 任务实测；rubric_optimizer 样本摘要文本重点分析——### {student_name} 标题仅在内存 prompt 不落库，改样本{i} 可零成本去 PII；与记忆系统提案的数据源映射及两处补充标注）
