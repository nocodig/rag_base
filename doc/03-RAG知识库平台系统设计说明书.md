# RAG 知识库平台系统设计说明书

> 项目名称：RAG Base
> 文档版本：V1.7
> 文档状态：设计评审稿
> 编制日期：2026-09-19
> 修订日期：2026-09-23
> 需求基线：《RAG 知识库平台需求说明书》V1.25
> 确认基线：《RAG 知识库平台需求确认单》V1.4

> V1.7 修订：对齐内容块启停的生效条件；区分纯索引重建与文档内容发布，明确快照分块配置和成员实际配置的关系，补全原文件重处理草稿及新 Embedding 配置与索引的事务切换。
>
> V1.6 修订：对齐 UPLOADED 删除状态迁移与 Embedding/Rerank 升级边界；复核需求评审稿校验值并保留未签署状态。
>
> V1.5 修订：完成需求追溯、运行角色、七牛云对象生命周期、状态并发、检索降级、安全、健康检查和备份恢复的全面一致性修正；删除含糊和重复表述，不增加需求外能力。
>
> V1.4 修订：修正完整索引快照与文档删除的衔接、明确逻辑快照和不可变向量复用边界、拆分取消清理失败状态、固定解析产物存储、修正核心对象关系及删除引用授权优先级；七牛云相关设计保持不变。
>
> V1.3 修订：文档原文件和受保护预览改为存储到公司七牛云账号下的 Kodo 私有空间；同步调整上传链路、访问控制、安全、部署、备份恢复和需求追溯。
>
> V1.2 修订：按需求说明书 V1.22 和需求确认单 V1.1 复核系统边界；补充 React、FastAPI、LlamaIndex、Celery、Redis 等项目技术栈；收敛实现级细节，并修正异步任务、索引模型绑定、管理接口和浏览器兼容性表述。
>
> V1.1 修订：补充可靠任务投递与任务状态机、知识库快照并发发布、索引与模型版本绑定、内容块即时启停、删除墓碑恢复、账号会话与权限失效、解析产物追溯、引用删除语义、评测门禁、安全运维和逐项需求追溯设计。

## 1. 文档说明

### 1.1 编制目的

本文定义 RAG Base MVP 的系统架构、模块边界、关键流程、状态与一致性机制、数据和接口边界、安全设计、部署设计及非功能设计，用于指导后续详细设计、开发、测试和部署。

### 1.2 基线状态

当前需求说明书状态为“评审稿”，需求确认单状态为“待确认”，因此本文属于设计评审稿。两份上游文档版本分别为 V1.25 和 V1.4；需求说明书当前 SHA-256 为 `aee1da5ab30ff11ca94d9853758ab0fdaefc44ed4b2ac304ca0b65321c097fa8`，与确认单记录的评审稿 SHA-256 一致。需求状态矩阵已补齐 `UPLOADED -> DELETING -> DELETED`；FR-ADMIN-001 已明确 Embedding 升级需要索引重建、Rerank 升级经评测后切换查询期版本。两份上游文档尚未完成相关方签署，本文不把评审稿称为已签署实施基线；签署前的正文变更须重新核对校验值和设计影响。

### 1.3 设计范围

本文覆盖：

- 总体逻辑架构和部署架构；
- 身份权限、知识库、文档、入库、索引、检索、引用、评测和运维模块；
- 文档入库、发布、检索、编辑、停用和删除等关键流程；
- 知识库、文档版本和索引版本状态控制；
- PostgreSQL、pgvector、Elasticsearch、对象存储和模型服务的数据职责；
- HTTP、MCP、Python 客户端及管理前端的接口边界；
- 与需求约束一致的项目技术栈及各组件职责；
- 安全、可靠性、可观测性、备份恢复和容量设计；
- 功能需求、非功能需求和验收场景的设计追溯。

### 1.4 非本文范围

本文不展开：

- 数据库表字段、DDL、索引语句和迁移脚本；
- 完整 HTTP/MCP 请求响应 Schema；
- 管理页面原型和视觉设计；
- 详细类图、函数级设计和源代码目录；
- 除已确认的七牛云 Kodo 外，其他云厂商、硬件厂商、服务器数量和资源规格；
- 项目计划、人员安排、预算和采购；
- 需求明确排除的聊天、答案生成、工作流、GraphRAG、多租户等能力。

上述内容分别由数据库设计、接口设计、详细设计、部署方案或项目计划承载，不在本系统设计中扩展。

## 2. 设计原则与固定约束

### 2.1 设计原则

1. **业务事实单一来源**：PostgreSQL 保存身份、权限、状态、版本、任务和内容块等业务事实。
2. **共享核心能力**：HTTP、MCP、Python 客户端、LangChain 和 LangGraph 最终调用同一检索应用服务。
3. **版本化发布**：文档内容、向量和倒排索引先构建、后校验、再切换，不允许半成品参与检索。
4. **权限前置且不可降级**：认证、授权、活动版本和状态过滤失败时直接失败，不能通过降级绕过。
5. **读写路径隔离**：在线检索与耗时入库分开执行，批量入库不得阻塞在线查询。
6. **派生数据可重建**：pgvector 向量和 Elasticsearch 索引均能从 PostgreSQL 业务数据及原文件重建。
7. **最小架构**：MVP 采用模块化单体核心，不拆分无必要的业务微服务。
8. **不生成业务答案**：平台仅返回内容块、得分、元数据和引用，不调用聊天模型生成最终答案。
9. **跨系统写入可恢复**：PostgreSQL、对象存储、任务队列和 Elasticsearch 之间不假设分布式事务，通过事务 Outbox、幂等消费、状态复核和一致性巡检实现最终一致。
10. **快照与 Embedding 强绑定**：每个活动索引版本固定绑定内容清单、Embedding 模型、Schema、分块和索引结构指纹，查询不得使用与活动向量空间不兼容的 Embedding 配置。
11. **安全状态默认拒绝**：权限缓存、状态同步、引用鉴权或密钥状态无法确认时拒绝访问，不以缓存命中或降级结果绕过业务事实。

### 2.2 需求固定技术约束

| 领域 | 设计约束 |
| --- | --- |
| 后端 | FastAPI |
| 关系数据库 | PostgreSQL |
| 向量能力 | PostgreSQL pgvector，1024 维、L2 归一化、余弦相似度 |
| 关键词检索 | Elasticsearch；中文索引使用 ik_max_word、查询使用 ik_smart，英文和业务代码保留基础 Token 分析 |
| 入库编排 | 本地开源 LlamaIndex Loader、Node Parser、Transform、Ingestion Pipeline |
| Embedding | 私有化部署 BAAI/bge-m3 |
| Rerank | 私有化部署 BAAI/bge-reranker-v2-m3 |
| MCP | 2026-07-28，无状态 Streamable HTTP |
| 主键 | 核心实体统一使用 UUIDv7 |
| 部署 | 核心计算、数据库、搜索和模型服务在公司环境内私有化部署；原文件与受保护预览使用公司七牛云 Kodo 私有空间 |

### 2.3 项目技术栈

技术栈只服务于本期已确认功能，不引入微服务、独立工作流平台或额外向量数据库。

| 层次 | 技术选型 | 系统职责与边界 |
| --- | --- | --- |
| 管理前端 | React、TypeScript、Vite、Ant Design、React Router、TanStack Query | 实现需求范围内的管理页面；通过私有管理 API 访问系统，不直连数据库、搜索引擎或对象存储 |
| 后端 API | Python、FastAPI、Pydantic、SQLAlchemy、Alembic、Uvicorn | 提供外部检索 API、私有管理 API、鉴权、业务编排和数据库迁移 |
| RAG 入库 | LlamaIndex Core、本地 Readers、Node Parser、Transform、Ingestion Pipeline | 仅用于本地解析、转换、分块和入库流水线；不负责在线检索、权限和对外契约 |
| 异步任务 | Celery Worker、Celery Beat、Scheduler | Worker 执行入库、索引、清理和评测；Beat 只产生周期触发；Scheduler 运行 Outbox 投递、恢复扫描和巡检；业务状态以 PostgreSQL 为准 |
| 消息队列 | Redis Broker | 承载 Celery 任务消息；结合事务 Outbox、幂等消费和任务租约保证可恢复投递，不作为业务状态来源 |
| 缓存 | Redis Cache | 缓存权限、配置和短期辅助数据；与 Broker 使用独立命名空间，缓存失效不得绕过 PostgreSQL 权威校验 |
| 关系与向量数据 | PostgreSQL、pgvector、SQLAlchemy、Alembic | 保存业务事实及 1024 维向量；向量索引属于可重建派生数据 |
| 关键词检索 | Elasticsearch、IK 分词插件、官方 Python 客户端 | 负责关键词召回、过滤和高亮，不保存唯一业务事实 |
| 对象存储 | 七牛云对象存储 Kodo、七牛云官方 Python SDK | 使用公司七牛云账号下的私有空间保存原文件、暂存对象和受保护预览；解析文本仍保存在公司私有环境 |
| 模型服务 | 本地 BAAI/bge-m3、本地 BAAI/bge-reranker-v2-m3 | 提供 Embedding 与 Rerank；不调用公网模型推理 API |
| 对外适配 | MCP Python SDK、轻量 Python Client、LangChain、LangGraph | MCP 只做协议适配；LangChain/LangGraph 位于调用端，不进入核心检索服务 |
| 测试 | pytest、pytest-asyncio、Vitest、React Testing Library、Playwright | 覆盖后端、前端、接口契约和核心端到端验收路径 |

Redis Broker 与 Redis Cache 在 MVP 可共用同一 Redis 部署，但必须使用独立逻辑库或键前缀、独立访问配置和容量监控；后续是否拆分只依据真实容量与可用性数据决定。

### 2.4 待技术基线冻结项

第 2.3 节所列前后端、任务、存储、检索、模型和适配组件的精确版本、镜像摘要与锁文件尚未在需求中给出，应在开发前通过兼容性验证后形成独立技术版本基线。本文不替代该基线，生产环境禁止使用 `latest` 标签或未锁定依赖。

## 3. 总体架构

### 3.1 架构风格

系统采用“模块化单体核心 + 独立进程角色”的架构：

- API 进程承载对外检索 REST API 和内网管理 API，两类路由、鉴权方式与 OpenAPI 文档分离；
- MCP 进程只承担协议校验和转换，复用同一检索应用服务；
- Celery Worker 进程执行解析、分块、Embedding、索引、清理和评测等异步任务；
- Scheduler 运行 Outbox 投递器、失败任务恢复、一致性巡检、容量检查和定时维护，Celery Beat 仅按配置产生周期触发；
- 各进程使用同一领域模型与应用服务代码，不复制业务规则。

### 3.2 逻辑架构

~~~mermaid
flowchart TB
    subgraph Clients[调用与管理端]
        AdminUI[管理前端]
        HttpClient[业务系统 / Python Client]
        LC[LangChain / LangGraph]
        Inspector[MCP Inspector]
    end

    subgraph Access[接入层]
        PrivateIngress[内网管理入口]
        RetrievalIngress[检索入口（公网暴露需审批）]
        AdminAPI[FastAPI 私有管理 API]
        RetrievalAPI[FastAPI 检索 REST API]
        MCP[MCP 协议适配入口]
    end

    subgraph Application[应用与领域层]
        IAM[身份与权限]
        KB[知识库与文档]
        Ingest[入库编排]
        Index[索引与发布]
        Retrieve[统一检索服务]
        Citation[引用服务]
        Eval[检索测试与评测]
        Audit[审计与追踪]
        Admin[运行管理]
    end

    subgraph Async[异步执行层]
        Outbox[事务 Outbox]
        Queue[Redis Broker]
        Worker[Celery Worker]
        Scheduler[Scheduler 运行角色（含 Outbox 投递 / Celery Beat）]
    end

    subgraph Infra[数据与基础设施]
        PG[(PostgreSQL + pgvector)]
        ES[(Elasticsearch)]
        Object[(七牛云 Kodo 私有空间)]
        Embed[本地 Embedding 服务]
        Rerank[本地 Rerank 服务]
        Cache[(Redis Cache)]
        Tombstone[(删除墓碑账本)]
        Obs[日志 / 指标 / 追踪 / 告警]
    end

    AdminUI --> PrivateIngress --> AdminAPI
    HttpClient --> RetrievalIngress
    LC --> RetrievalIngress
    Inspector --> RetrievalIngress
    RetrievalIngress -->|/api/v1/retrieval/*| RetrievalAPI
    RetrievalIngress -->|/mcp| MCP

    AdminAPI --> IAM
    AdminAPI --> KB
    AdminAPI --> Eval
    AdminAPI --> Admin
    AdminAPI --> Retrieve
    RetrievalAPI --> Retrieve
    MCP --> Retrieve
    Retrieve --> IAM
    Retrieve --> Citation
    IAM --> Outbox
    KB --> Outbox
    Ingest --> Outbox
    Eval --> Outbox
    Outbox --> Scheduler --> Queue
    Queue --> Worker
    Worker --> Ingest
    Worker --> Index
    Worker --> Eval

    IAM --> PG
    KB --> PG
    Worker --> Tombstone
    Ingest --> PG
    Ingest --> Object
    Index --> PG
    Index --> ES
    Index --> Embed
    Retrieve --> PG
    Retrieve --> ES
    Retrieve --> Embed
    Retrieve --> Rerank
    Retrieve --> Cache
    Scheduler --> Cache
    Citation --> PG
    Citation --> Object
    Eval --> PG
    Audit --> PG
    Audit --> Obs
    Admin --> Obs
    Worker --> Obs
    Scheduler --> Obs
    Scheduler --> PG
    Scheduler --> ES
    Scheduler --> Object
~~~

### 3.3 模块职责

| 模块 | 核心职责 | 不负责 |
| --- | --- | --- |
| 身份与权限 | 人工账号、API 调用账号、API Key、角色、知识库直接授权、会话失效 | IAM/SSO、多租户、组织继承 |
| 知识库与文档 | 知识库、元数据 Schema、文档身份、文档版本、内容块运营状态 | 外部业务数据生产 |
| 入库编排 | 文件校验、解析、清洗、分块、Embedding 调用和任务状态 | 在线检索 |
| 索引与发布 | 双索引写入、校验、版本切换、回滚、清理和一致性检查 | 业务正文编辑 |
| 统一检索 | 权限过滤、三种召回、融合、Rerank、裁剪、降级和结果映射 | 最终答案生成 |
| 引用服务 | 稳定引用、来源定位、权限复查、历史和删除状态返回 | 永久公开下载 |
| 检索测试与评测 | 单条调试、评测集、批量指标、参数对比 | LLM Judge、在线 A/B |
| 审计与追踪 | 管理操作审计、脱敏检索追踪、查询指纹和链路关联 | 保存原始查询正文 |
| 运行管理 | 健康、队列、容量、模型、错误和告警视图 | 读取未授权正文 |

事务 Outbox、删除墓碑账本和权限/状态复核属于逻辑能力，不引入新的业务微服务。它们由模块化单体中的独立运行角色实现，但不得退化为仅依赖进程内存的临时机制。

### 3.4 部署架构

~~~mermaid
flowchart LR
    Qiniu[(七牛云 Kodo 私有空间)]
    subgraph Corp[公司网络]
        subgraph AccessZone[接入区]
            RP[反向代理 / TLS]
        end

        subgraph AppZone[应用区]
            API[FastAPI API 实例]
            MCPD[MCP 适配实例]
            W[Celery Worker 实例]
            S[Outbox 投递器 / Celery Beat / Scheduler]
        end

        subgraph DataZone[数据区]
            PG[(PostgreSQL / pgvector)]
            ES[(Elasticsearch)]
            MQ[(Redis Broker)]
            C[(Redis Cache)]
            DL[(追加写删除墓碑账本)]
        end

        subgraph ModelZone[模型区]
            E[BAAI/bge-m3]
            R[BAAI/bge-reranker-v2-m3]
        end

        subgraph OpsZone[运维区]
            O[日志 / 指标 / 追踪 / 告警]
            B[备份存储]
        end
    end

    User[管理用户 / 调用系统] --> RP
    RP --> API
    RP --> MCPD
    API --> PG
    API --> ES
    API -->|HTTPS / 官方 SDK| Qiniu
    API --> C
    API --> E
    API --> R
    MCPD --> PG
    MCPD --> ES
    MCPD --> C
    MCPD --> E
    MCPD --> R
    MQ --> W
    W --> PG
    W --> ES
    W -->|HTTPS / 官方 SDK| Qiniu
    W --> E
    S --> MQ
    S --> PG
    S --> ES
    S -->|HTTPS / 官方 SDK| Qiniu
    W --> DL
    API --> O
    MCPD --> O
    W --> O
    S --> O
    PG --> B
    ES -.可选索引快照（仅重建超 RTO 时）.-> B
    Qiniu -.对象副本与清单备份.-> B
    DL --> B
~~~

管理入口仅允许公司内网访问；检索入口是否配置公网入口由独立安全审批决定。除文档原文件和受保护预览使用公司七牛云账号下的 Kodo 私有空间外，核心服务、关系数据、解析文本、索引、日志、密钥和模型保留在公司私有环境。访问七牛云通过 HTTPS、固定域名白名单和独立访问密钥执行。

## 4. 核心模块设计

### 4.1 接入层

#### 4.1.1 对外检索 REST API

- 路径统一使用 /api/v1/retrieval 前缀；
- 仅提供知识库列表、搜索、内容块详情和引用详情；
- 使用 Bearer API Key；
- 请求进入应用服务前完成鉴权、限流、参数上限和基础 Schema 校验；
- 对外 OpenAPI 与内部管理 OpenAPI 分开生成。

#### 4.1.2 私有管理 API

- 仅服务管理前端并限制在公司内网；
- 使用人工账号登录后的访问凭证，不接受 API Key；
- 所有写操作执行权限、状态迁移、并发版本和审计校验；可能产生重复副作用的操作增加幂等保护；
- 不作为外部长期兼容接口。

#### 4.1.3 MCP 适配入口

- 默认路径为 /mcp；
- 固定使用 MCP `2026-07-28` 无状态 Streamable HTTP，校验 Bearer API Key、`Mcp-Protocol-Version`、`Mcp-Method`、`Mcp-Name` 及请求 `_meta` 中的协议版本、客户端信息和能力声明；
- 不执行 `initialize`/`initialized`，不维护 `Mcp-Session-Id`，不实现 OAuth；支持直接调用 `tools/list`、`tools/call`，能力发现使用 `server/discover`；
- MVP 不实现传统 HTTP+SSE 传输，不在 `/mcp` 混用旧协议生命周期；后续如需兼容旧版，必须使用独立端点并另行评审；
- 提供 search_knowledge、list_knowledge_bases、get_chunk；
- 仅做协议转换，不自行实现权限、检索、排序或引用逻辑；
- 与 REST API 使用同一错误枚举、限流计数、追踪规范和结构化领域结果。

#### 4.1.4 管理前端

- 使用 React + TypeScript 构建单页管理应用，Vite 负责构建，Ant Design 提供基础管理组件；
- React Router 管理页面路由，TanStack Query 管理服务端数据、缓存失效和请求状态；
- 页面范围严格对应需求中的知识库、文档、内容块、知识检索、账号、密钥、评测、追踪和平台运行管理，不增加聊天、答案生成或工作流页面；
- 前端仅调用私有管理 API，不保存业务事实，不直连 PostgreSQL、Elasticsearch、Redis 或七牛云 Kodo，也不持有七牛云 AccessKey/SecretKey；
- 登录态、页面按钮和路由控制只改善交互，后端仍必须执行完整认证、授权、状态和审计校验。

### 4.2 身份与权限模块

#### 4.2.1 人工账号

- 邮箱经去除首尾空格并统一大小写后作为唯一登录名；
- MVP 默认不开放自助注册，人工账号由 `ACCOUNT_ADMIN` 创建；
- 密码使用自适应哈希与独立随机盐；默认最小长度为 8 个字符，密码策略由平台管理员配置并版本化；
- 人工账号状态为 ACTIVE、LOCKED、DISABLED；登录失败计数、临时锁定截止时间和解锁操作均保存于 PostgreSQL 并写入审计；
- 人工账号记录最近一次成功登录时间；API Key 记录最近一次成功鉴权使用时间，失败请求不得更新这两个时间；
- 管理员密码重置签发一次性临时密码，首次登录必须修改；临时密码只保存安全哈希，不得作为长期会话使用；
- 登录成功创建可撤销的会话记录，访问凭证携带会话编号和明确 expires_at，有效时长由版本化安全配置固定；退出、密码修改、账号禁用和管理员强制下线通过会话版本或撤销时间使已有凭证立即失效；
- 管理前端的会话凭证固定保存在 `HttpOnly; Secure; SameSite=Lax` Cookie 中，状态变更请求同时校验后端签发的 CSRF Token；会话凭证不写入 Web Storage；
- 全局角色为 PLATFORM_ADMIN、ACCOUNT_ADMIN、USER；
- 首次部署通过只允许执行一次的安全初始化流程创建唯一初始管理员，不内置通用默认密码；该账号同时具有 `PLATFORM_ADMIN` 和 `ACCOUNT_ADMIN`，创建、保管人指定、交接、重置和停用均写入审计；
- 全局角色不隐含知识正文读取权限。

#### 4.2.2 API 调用账号与 API Key

- API 调用账号为非登录主体，只用于 HTTP 和 MCP；
- API 调用账号保存名称、用途、负责人、ACTIVE/DISABLED 状态、授权知识库、创建时间和最后调用时间；DISABLED 立即使其全部 API Key 失效；
- API Key 使用具有足够熵的不可预测随机值，完整值只在创建时返回一次；服务端使用独立密钥计算并保存 HMAC-SHA256 校验值和 key_id，不保存可恢复明文；日志只记录密钥编号和脱敏标识；
- API Key 不自动过期，禁用、删除或所属 API 调用账号禁用后立即失效；
- 实际知识库范围取“API Key 自身范围”与“所属 API 调用账号当前授权范围”的交集；API Key 范围使用独立关系保存，不以内嵌字符串代替；
- 轮换时新旧密钥同时有效，调用方完成切换后由管理员显式禁用旧密钥；未显式禁用前平台不按存续时间自动使任一密钥失效。

#### 4.2.3 知识库权限

- 权限仅为 MANAGE、EDIT、RETRIEVE、EVALUATE；
- 人工账号和 API 调用账号与知识库直接授权，不存在用户组和组织继承；
- 多知识库请求必须先完成全量授权校验，任一目标不存在或无权访问时整次返回 403 FORBIDDEN_SCOPE；
- 每项管理操作必须同时满足平台角色和知识库权限矩阵；MANAGE、EDIT、RETRIEVE、EVALUATE 互不隐含，EVALUATE 只允许读取评测执行所需的受控结果，不自动授予通用正文浏览能力；
- 知识库授权缓存只作为加速手段，有效期不超过 5 分钟；授权事务递增主体的 authorization_version 并通过 Outbox 主动失效缓存，缓存条目的版本与权威版本不一致或存在未完成失效事件时直接回源 PostgreSQL；
- API 调用账号、API Key 禁用和会话撤销不得等待缓存自然过期，鉴权记录携带状态版本并在请求入口执行权威复核；
- 读取内容块和引用时重新执行当前权限检查；
- 私有管理 API 同时提供“按账号查看其知识库权限”和“按知识库查看已授权账号”两种查询视图，两者读取同一 KnowledgePermission 事实，不维护独立授权副本；
- API 调用账号只能获得 RETRIEVE 权限，不能获得 MANAGE、EDIT、EVALUATE 或登录管理前端。

平台角色与知识库权限的组合规则如下：

| 操作 | 平台角色要求 | 知识库权限要求 |
| --- | --- | --- |
| 账号/API 调用账号生命周期 | ACCOUNT_ADMIN | 无，不获得正文权限 |
| 平台模型、节点、配额管理 | PLATFORM_ADMIN | 无，不获得正文权限 |
| 创建知识库 | USER | 创建后自动获得四项直接权限 |
| 修改知识库配置、授权、启停、删除 | 可登录人工账号 | MANAGE |
| 上传、编辑、发布、停用、删除文档 | 可登录人工账号 | EDIT |
| 检索、读取当前块和引用 | 人工账号或 API 调用账号 | RETRIEVE |
| 管理评测集、运行评测 | 可登录人工账号 | EVALUATE |

### 4.3 知识库与文档模块

- KnowledgeBase 保存名称、描述、默认语言、状态、分块配置、元数据 Schema 和活动索引版本引用；
- Document 作为稳定业务身份，指向当前发布版本；文档是否可检索仍由当前 DocumentVersion 状态承载；
- DocumentVersion 保存文件版本、来源版本、变更原因、处理/发布状态和 disable_reason；DISABLED 必须区分 MANUAL 和 SUPERSEDED；
- Chunk 保存内容、位置、标题路径、父子关系、元数据、版本和启用状态；
- MetadataSchemaVersion 独立版本化，记录字段名称、类型、必填、枚举/格式校验、可过滤、可由内容块覆盖、可在响应中返回和字段状态；已索引字段不允许原地修改类型；
- 业务元数据基础类型为字符串、数字、布尔、日期和字符串列表；删除字段先转为 DISABLED，历史文档保留原值，完成索引重建和清理后才能删除字段定义；
- 文档元数据默认继承到内容块，只有 chunk_overridable 字段可以覆盖；
- K12 模板为可选业务模板，预置 `content_type`、`course_id`、`grade_code`、`subject_code`、`campus_id`、`term_code`、`class_type`、`valid_from`、`valid_to`、`sales_stage`、`channel`、`tags`；未启用该模板的知识库不创建这些业务字段；
- `tenant_id`、`knowledge_base_id`、`document_id`、`document_version`、`index_version`、`document_status`、`chunk_enabled` 是平台权限/状态字段，只能由服务端生成，调用方不能提交或覆盖。

生产初始化脚本和数据库迁移不得预置首批业务知识库；平台上线后由业务团队人工账号通过私有管理 API 自行创建和维护。预发布验收知识库与生产数据隔离，不属于生产预置业务知识库。

文档停用把当前 PUBLISHED 版本改为 DISABLED，并记录 disable_reason=MANUAL；新版本发布把旧版本改为 DISABLED，并记录 disable_reason=SUPERSEDED。DISABLED 版本重新进入 PUBLISHED 仅允许两类路径：按下述时序恢复当前人工停用版本，或以当前活动快照为基线构建新快照完成单文档历史版本回滚；两者均写入审计，不得直接修改状态或直接激活历史整库快照。

知识库创建后为 DRAFT；持有 MANAGE 权限的人工账号完成配置校验后显式执行 DRAFT -> ACTIVE，空知识库可为 ACTIVE 但检索返回空结果。首次文档发布不隐式激活知识库；验收流程必须先激活知识库，再发布文档。

人工停用文档的恢复使用“先可见性投影、后权威状态”时序：先确认目标 MANUAL DISABLED 版本仍是当前活动快照的隐藏成员，将双检索后端的可见性投影同步为待启用并验证，再在 PostgreSQL 中将其恢复为 PUBLISHED；失败时保持 DISABLED。若活动快照缺少该隐藏成员，必须构建新的完整 IndexVersion，禁止向不可变成员清单原地追加。单文档历史版本回滚必须以当前活动快照为基础，只替换该文档成员并构建新的完整 IndexVersion，不得直接激活可能包含其他文档旧状态的历史整库快照。

### 4.4 入库编排模块

#### 4.4.1 任务模型

管理 API 对每个上传文件独立处理：先完成权限、扩展名、声明 MIME 和空文件校验，在 PostgreSQL 创建携带平台暂存键和有效截止时间的 UploadReservation，再使用七牛云官方 Python SDK 通过 HTTPS 流式写入 Kodo 私有空间的暂存前缀，同时强制单文件大小上限并计算平台内容 `content_sha256`；另行记录 SDK 上传响应返回的 Kodo 对象 `hash` 和对象大小，二者不得与 SHA-256 混用。对象键由平台根据 UUIDv7 标识生成，不使用用户文件名，正式键不允许覆盖已有对象。上传完成后，API 在同一 PostgreSQL 事务中消费 UploadReservation，创建 DocumentVersion、DocumentObject（STAGED）、IngestionTask、状态为 QUEUED 的首个 TaskExecution 和 OutboxEvent；事务提交后由 Scheduler 中的 Outbox 投递器把 task_id 发送到 Redis Broker。

Worker 取得任务租约后先调用 Kodo 对象信息查询校验对象键、大小和 Kodo `hash`；读取暂存对象执行深度校验时重新计算 `content_sha256` 并与平台记录值比较，同时完成文件特征、解压后大小、页数、工作表、单元格、附件数和安全深检。校验通过后才将对象转为不可变正式对象：正式键不存在时执行移动（如底层需复制则复制后再删暂存对象）并再次查询对象信息；正式键已存在且大小、Kodo `hash` 均与预期一致时按幂等成功处理；任一值不一致时以稳定冲突错误失败。只有 Kodo 对象信息复核和平台 SHA-256 复核均通过后才把 DocumentObject 改为 FORMAL，然后进入解析。

深度校验因格式不支持、文件损坏或资源超限而确定性失败时，该文件未成为可接受原文件：Worker 删除暂存对象、再查询确认不存在，将 DocumentObject 标记为 REJECTED，并使 DocumentVersion/IngestionTask 以稳定错误码失败；不得将该对象转为 FORMAL。FORMAL 后发生的解析或依赖失败不自动删除原文件，以便人工诊断和重试。

暂存对象只允许在同时满足以下条件时由孤儿扫描删除：超过配置的孤儿最小年龄，PostgreSQL 中无 DocumentObject 引用、无活动上传预留且无有效任务。该最小年龄必须大于最大上传超时加数据库提交重试窗口；队列长时间不可用不得导致仍被任务引用的原文件提前过期。单批文件数量使用外置配置且不得无界；批量上传中每个文件使用独立的对象、数据库事务、任务和 Outbox 事件，单文件失败不回滚同批已成功文件。

受保护预览使用独立的平台对象键和 DocumentObject，执行与原文件相同的 STAGED -> FORMAL、不覆盖、对象信息复核和孤儿回收规则；预览地址只在请求时经权限复核后签发，不作为对象事实保存。

重处理不创建 UploadReservation 或新的 ORIGINAL 对象：锁定当前来源 DocumentVersion，沿同文档 source_version_id 链找到未被墓碑排除的 FORMAL ORIGINAL 并复核可读性，同一 PostgreSQL 事务创建 `change_type=REPROCESS`、状态 UPLOADED 的新 DocumentVersion、IngestionTask、QUEUED TaskExecution 和 OutboxEvent。Worker 的 VALIDATE 阶段复核已解析来源对象键、大小、Kodo `hash` 和平台 SHA-256，再直接进入解析；FINALIZE 阶段只对新上传的 STAGED 对象执行。来源原文件在重处理任务和新版本保留期间不得独立清理；删除整个 Document 时仍按墓碑先停止关联任务、再清理全部版本与对象。

任务至少记录任务编号、文档版本、参数版本、当前状态、当前阶段、进度、执行次数、租约持有者、心跳时间、下次重试时间、开始/结束时间和稳定错误码。初始执行及每次自动或人工重试各创建且仅创建一条 QUEUED TaskExecution；Worker 领取后将该记录转为 RUNNING，不覆盖历史失败执行。

阶段固定为：

1. 对象完整性、文件特征与资源上限校验；
2. 正式对象幂等确认；
3. 安全解析；
4. 文本清洗；
5. 分块；
6. 受保护预览生成；
7. Embedding；
8. pgvector 写入；
9. Elasticsearch 写入；
10. 双索引校验；
11. 任务收尾并将 DocumentVersion 转入 REVIEW_REQUIRED。

Celery 任务采用至少一次投递语义：消息只携带 task_id，Worker 每次执行前在 PostgreSQL 通过条件更新取得带单调递增 execution_generation 的有期租约。所有阶段提交、状态迁移和产物晋级均必须同时校验 task_id、租约持有者和 execution_generation；租约失效的旧执行记录为 LOST，其后续写入不得转为可见结果。OutboxEvent 在 Broker 确认接收后标记已投递；Scheduler 中的 Outbox 投递器持续扫描未投递事件，Scheduler 恢复扫描由 Celery Beat 触发，处理超过派发时限仍为 PENDING 的任务和过期租约。Outbox 投递器启动只依赖 PostgreSQL，Redis 不可用时保留未投递状态并继续重试。

#### 4.4.2 解析适配

- 入口只接受 PDF、DOCX、XLSX、PPTX、TXT、MD、HTML、CSV、JSON、EML、PNG、JPG/JPEG；其他扩展名或文件特征统一以稳定格式不支持错误拒绝；
- MVP 单文件服务端硬上限为 200 MB，默认有效上限同为 200 MB，管理员只能下调；解压后内容硬上限为 1 GB，PDF 默认最多 2,000 页，PPTX 默认最多 1,000 张幻灯片，XLSX 默认最多 100 个可见工作表且非空单元格总数最多 1,000,000，EML 默认最多处理 100 个附件；管理员可下调格式专属默认上限，超限时整个单文件任务失败并返回稳定错误码；
- 《支持格式矩阵》固定记录每类文件的扩展名、MIME、解析器、资源上限、OCR 规则和验收样例，作为发布物随版本冻结；
- 每类文件使用独立解析适配器，统一输出平台中间文档结构；
- 解析以中文内容为主，但必须原样保留已有英文、数字和业务代码，并保持标题层级、段落、列表和表格文本的合理阅读顺序；
- LlamaIndex 只位于内部适配层，用于本地 Loader、Node Parser、Transform 与 Ingestion Pipeline 编排；不调用 LlamaCloud、LlamaParse 云 API 或其他托管解析服务；
- LlamaIndex 对象必须转换为平台 DocumentVersion、Chunk 和元数据对象；
- 解析产物包含原始提取文本、清洗后文本、结构位置、解析器版本、清洗规则版本、OCR 模型版本、OCR 置信度和告警；产物固定保存在 PostgreSQL，不引入未定义的第二私有文件存储；大产物按有序分段保存和流式读取，不写入七牛云，并由 DocumentVersion、分段清单和完整产物校验值稳定关联；
- PDF 保留页码；PPTX 保留幻灯片编号、正文、表格和演讲者备注；XLSX 保留工作表名称；EML 保留主题、发件人、收件人和发送时间；图片保留图片编号。隐藏工作表和不支持附件形成结构化告警；
- EML 正文任务与每个受支持附件的子 Document/IngestionTask 独立执行；父文档记录所有子任务及聚合状态 PENDING、COMPLETE 或 PARTIAL_FAILED。任一受支持附件失败时，父正文仍可进入 REVIEW_REQUIRED，但聚合状态必须为 PARTIAL_FAILED，界面必须展示失败附件和子任务错误，不得表示全部附件已入库；不支持附件记录告警并跳过；
- HTML 移除脚本、样式和不可见内容，禁止加载外部 URL 或资源；XLSX 只读取已保存值且不执行或重新计算公式；扫描 PDF、PNG 和 JPG/JPEG 使用本地 OCR；解析环境禁止宏、脚本、嵌入对象和外部命令；
- 清洗去除多余空白、重复空行和不可见控制字符，保留标题、列表编号、表格行列标识和链接文本，抑制重复页眉页脚及纯装饰内容；清洗前文本、清洗后文本和规则版本保持可追溯；
- 解析失败返回稳定错误码，不得生成空索引并标记成功；
- OCR 低于配置阈值时，DocumentVersion 进入 REVIEW_REQUIRED 并携带低质量标记，不能通过自动流程静默发布；
- 大文件采用流式读取或分段处理，不一次性载入应用进程内存。

#### 4.4.3 分块设计

| 策略 | 系统语义 |
| --- | --- |
| 通用分块 | 按标题、段落和 Token 上限切分普通文档、网页和 Markdown |
| 章节分块 | 优先保留制度、手册和长文档的章节层级及标题上下文 |
| Q&A 分块 | 将一问一答作为一个语义单元 |
| 表格分块 | 保留表头，CSV/XLSX 按行组切分 |
| 父子分块 | 子块参与匹配，命中后按父块返回完整上下文 |

- 分块配置包含目标 Token 数、最大 Token 数、重叠长度和分隔符，并随快照冻结配置版本；
- 分块不得从 Unicode 字符、Markdown 链接或表格单元格中间截断；父块和子块均保留原文定位，子块记录父块编号；
- 修改分块策略或参数时重新生成内容块和索引，旧活动快照持续服务至新完整快照校验并原子切换。
- 对已发布文档使用新解析或分块配置时，从当前版本沿同一文档的 source_version_id 链定位经校验仍可读取的 FORMAL 原文件，并创建独立重处理草稿；不要求调用方重复上传相同文件，不覆盖原 DocumentVersion 或内容块。来源链缺失、跨文档、成环、命中墓碑或对象不可验证时拒绝重处理。处理前提示已有人工修改须在新草稿重新合并；重处理完成仍需人工审核与受控发布，不能因配置激活而自动替换线上版本。

#### 4.4.4 失败、取消和重试

- 确定性错误不自动重试；
- 临时依赖错误按配置执行有限次数退避重试；
- 自动或人工重试在任务重新进入 PENDING 时创建一条新的 QUEUED TaskExecution 并写入 Outbox，保留原失败执行；Worker 领取时只把该条执行记录转为 RUNNING，不再重复创建。DocumentObject=STAGED 时必须重新执行对象信息和平台 SHA-256 复核后再继续转正，FORMAL 时复核正式对象后从安全阶段恢复；DocumentObject=REJECTED 必须重新上传；
- Worker 使用带 execution_generation 的 PostgreSQL 任务租约识别异常中断，服务恢复后可重新调度；
- PENDING 且从未写入派生数据的任务按 `PENDING -> CANCEL_REQUESTED -> CANCELLED` 迁移，并在同一事务中将尚未领取的 QUEUED TaskExecution 改为 CANCELLED，关联 DocumentVersion 保持 UPLOADED；RUNNING 任务的取消请求先转为 CANCEL_REQUESTED，Worker 在阶段边界停止、将本次 INGEST TaskExecution 记为 CANCELLED 并转入 CLEANUP_PENDING，只有确认本次未发布派生数据已清理后才将 IngestionTask 转为 CANCELLED，同时将 DocumentVersion 从 PROCESSING 转为 FAILED 并记录稳定取消错误码；
- 临时清理失败进入 CLEANUP_RETRY_WAIT 并按退避策略只重试清理，超过自动重试次数后进入 CLEANUP_FAILED 并告警；CLEANUP_RETRY_WAIT 或 CLEANUP_FAILED 期间不得提前标记 CANCELLED，人工重试 CLEANUP_FAILED 只能恢复清理，不能重新执行入库；
- 每次计划清理尝试创建一条 QUEUED TaskExecution，并以 execution_type=CLEANUP 与普通 INGEST 执行区分；Worker 领取后更新同一记录，清理重试保留全部执行和错误历史；
- 同一文档版本同一处理阶段通过幂等键和并发约束防止重复写入。

### 4.5 索引与发布模块

#### 4.5.1 索引组织

- Elasticsearch 使用知识库和索引版本可定位的物理索引；
- pgvector 中的 EmbeddingArtifact 是不可变向量产物，在固定租户边界内按“规范化文本摘要 + Embedding 模型指纹”唯一定位；新文档版本中文本未变的 Chunk 可引用已校验产物，但仍使用自己的 chunk_id 与引用语义。IndexSnapshotMember 关联 index_version、chunk_id 和 EmbeddingArtifact，并固化其引用 Chunk 的正文、关键词、业务元数据及稳定系统定位字段；除 visibility_revision 控制的可见性投影外不得原地改写。正文、关键词或业务元数据变化必须创建新的 DocumentVersion 和 IndexVersion。document_status、chunk_enabled 和墓碑排除属于同步到双检索后端的可见性投影；主体授权不固化到成员中，每次请求按 PostgreSQL 当前 KnowledgePermission 与 ApiKeyScope 求值；
- 两侧均写入业务过滤字段和服务端权限过滤字段；
- 每个 IndexVersion 表示知识库在一个发布时点的完整逻辑内容快照，新版本必须通过不可变成员清单包含该知识库全部应发布内容，不能只记录本次变化文档；逻辑完整不等于为所有未变化内容重复生成向量；
- IndexVersion 必须记录 base_active_index_version、base_visibility_revision、快照内容清单摘要、元数据 Schema 版本、构建时知识库当前分块配置版本、Embedding 模型指纹和索引结构版本；同一完整快照可保留按旧配置处理但内容未变化的文档，各文档实际使用的分块配置以其 DocumentVersion 绑定值为准，不能把快照配置编号误认为全部成员均已重切；Rerank 与 tokenizer 属于查询期配置，记录在 RetrievalTrace 和 EvaluationRun；Rerank 升级不触发索引重建；
- Embedding 模型指纹至少由模型名称、模型版本、模型文件校验值、向量维度、归一化规则和查询前缀规则组成；查询向量必须按活动 IndexVersion 的指纹生成；
- 未变化内容块复用已校验的文本和 EmbeddingArtifact，新快照通过显式成员关系引用不可变产物，不依赖对象“当前状态”动态拼出内容快照；内容清单摘要只覆盖不可变成员。文档停用、删除墓碑和内容块启停每次递增知识库 visibility_revision，作为 PostgreSQL 实时权威覆盖，不改变快照正文或内容摘要；
- 后继完整快照必须继承当前 MANUAL DISABLED 文档版本及 `chunk_enabled=false` 内容块作为隐藏成员，SUPERSEDED 版本除外；隐藏成员参与不可变成员清单和内容摘要，但始终被可见性投影排除。恢复只更新可见性投影，不得向活动快照原地新增成员；
- IndexVersion 管理 BUILDING、READY、ACTIVE、RETIRED、FAILED、DELETED 状态；
- 每个知识库只有一个 active_index_version。

#### 4.5.2 发布策略

1. 读取并记录当前 active_index_version，创建 BUILDING 索引版本和快照内容清单；
2. 对变化内容生成新的 EmbeddingArtifact，对未变化内容复用已校验产物并写入新快照成员关系；基于完整成员清单构建 Elasticsearch 新物理索引，批量复用未变化内容的已校验解析结果；
3. 校验数量、标识、向量维度、权限字段、业务过滤字段、可查询性和构建时的 visibility_revision；
4. 校验成功后将索引标记为 READY；
5. 发布前复核 READY 版本的校验证据；双索引任一侧不可用或证据不完整时不得进入切换事务；
6. 对知识库取得数据库级互斥锁，校验当前 active_index_version 仍等于 base_active_index_version，并复核目标文档状态、构建后新增墓碑及 visibility_revision。若仅 visibility_revision 改变，必须先将最新启停/停用覆盖同步到新双索引并重新校验，再以最新 revision 执行条件切换；命中 DELETING/DELETED、已生效墓碑或基础活动版本改变时禁止激活，返回 409；
7. 在短 PostgreSQL 事务中切换活动索引版本并退役旧索引；若此次快照发布包含文档内容版本变更，同一事务还须完成新文档版本 PUBLISHED、旧版本 DISABLED 和 Document 当前版本引用更新；纯 Embedding/Schema/索引修复重建不伪造文档版本迁移。活动版本更新采用条件更新并受唯一约束保护，事务内不执行 Elasticsearch 等远程网络调用；
   若新快照使用尚未供在线查询的 Embedding 指纹，同一事务须确认其已核验的本地运行配置可用并将该指纹配置置为 ACTIVE；旧指纹配置继续保留供在途请求和保留快照使用。任一前置条件失败时旧活动索引和模型配置均保持不变；
8. 新检索请求读取新的活动版本；已开始请求继续使用请求开始时捕获的版本；
9. 旧索引进入 RETIRED 后按回滚保留策略异步清理；物理清理必须同时满足回滚保留期结束，且从退役时刻起已经过“在线检索强制最大执行时限 + 回收安全窗口”，以保证捕获该版本的在途请求已结束；新请求不得选择 RETIRED 版本。

活动版本是在线检索的唯一发布开关。Elasticsearch 别名可以用于运维定位，但不能取代 PostgreSQL 中的活动版本事实。

任何包含已生效删除墓碑目标的 READY 或 RETIRED IndexVersion 均不得激活。`RETIRED -> ACTIVE` 只用于显式的整库快照回滚：执行者必须具有 MANAGE，界面展示全部受影响文档并二次确认，切换前重新校验墓碑、当前 visibility_revision 和双索引完整性，并在同一 PostgreSQL 事务中同步受影响 DocumentVersion 发布状态及当前版本引用。单文档回滚按 4.3 节构建新快照，不使用此迁移。

因并发发布冲突而过期的 READY 版本标记为不可再发布，其派生数据完成可追踪清理后按 `READY -> DELETED` 收尾；重试必须基于最新活动版本创建新的 BUILDING 版本，不复用原 READY 版本。

同一知识库同一时刻只允许一个发布动作进入切换临界区，构建任务可以并行，但过期快照不得发布。完整快照是逻辑内容清单，不要求复制全部 PostgreSQL 正文或重复计算未变化向量；Elasticsearch 新物理索引仍按完整成员清单构建。详细设计必须定义成员复用、批量构建、垃圾回收和容量基线，避免把一次局部编辑退化为全库解析和全量 Embedding。

#### 4.5.3 一致性巡检

巡检任务按知识库和索引版本核对：

- PostgreSQL 内容块与 pgvector 向量是否缺失、重复或维度错误；
- PostgreSQL 内容块与 Elasticsearch 文档是否缺失、重复或版本错误；
- 活动版本是否在两侧完整可用；
- IndexVersion 快照成员、基础活动版本和模型指纹是否一致；
- 文档删除是否已生成排除目标内容的后继快照；处于 DELETING 且墓碑清理未完成的内容按权威覆盖处理，不作为普通索引缺失告警；
- 每个 STAGED/FORMAL DocumentObject 对应的 Kodo 对象是否存在，且对象键、大小和 Kodo `hash` 是否与 PostgreSQL 记录一致；STAGED 超过安全窗口时同时核对 UploadReservation 和任务状态；
- 平台暂存/正式前缀中超过孤儿安全窗口的对象是否缺少 DocumentObject、有效 UploadReservation 和有效任务引用；只有同时满足 4.4.1 节孤儿条件时才允许进入可追踪清理；
- REJECTED/DELETED DocumentObject 对应对象是否仍残留，以及已删除业务对象是否存在正文、向量、倒排索引、缓存或预览孤儿数据；
- 发现问题后记录告警，禁止自动把未校验版本切为活动版本。

### 4.6 统一检索模块

统一检索服务按固定流水线执行：

~~~mermaid
flowchart LR
    A[参数校验] --> B[主体鉴权与 PrincipalContext]
    B --> C[全量知识库授权校验]
    C --> D[读取活动版本与 Schema]
    D --> E[编译固定条件与业务过滤]
    E --> F{检索模式}
    F -->|KEYWORD| G[Elasticsearch 召回]
    F -->|VECTOR| H[查询向量 + pgvector 召回]
    F -->|HYBRID| G
    F -->|HYBRID| H
    G --> I[候选汇总]
    H --> I
    I --> J[融合或单路稳定排序与去重]
    J --> K{rerank}
    K -->|启用| L[本地 Rerank]
    K -->|关闭或降级| M[保留当前排序]
    L --> N[阈值过滤]
    M --> X[父块映射与去重]
    N --> X
    X --> O[结果裁剪]
    O --> P[引用与响应映射]
    P --> Q[写入脱敏追踪]
~~~

关键规则：

- 外部 REST/MCP 由接入层校验 API Key，管理界面知识检索或评测由私有 API 校验人工会话；两者均生成统一 PrincipalContext，检索核心不复制身份逻辑；
- 请求开始时捕获统一 UTC 时间，以及每个目标知识库的 index_version、metadata_schema_version、embedding_model_fingerprint 和 visibility_revision；启用 Rerank 时同时捕获当前 ACTIVE Rerank 配置的模型版本与文件校验值，整个请求保持已捕获版本；
- 多知识库 VECTOR 请求在 Embedding 指纹不一致时按指纹分组生成查询向量并分别召回，再使用服务端版本化的固定 `cross_fingerprint_rrf_k=60` 对各排名列表执行 RRF；此时 effective_fusion 明确记录 `CROSS_FINGERPRINT_RRF`、内部常量和版本，score_type 为 RRF。调用方参数 `rrf_k` 仍不参与 VECTOR；任一目标指纹无法加载时整个 VECTOR 请求返回 503；
- HYBRID 中某个 Embedding 指纹不可用时，只移除该指纹对应的向量排名列表，对应知识库仍使用关键词列表；所有向量和关键词列表均不可用时返回 503，否则以 RRF 融合有效列表并记录降级；KEYWORD 不依赖 Embedding；
- 固定权限条件由服务端注入，业务过滤不能覆盖；
- 多知识库字段在执行前检查存在性和类型兼容性；
- 单知识库 KEYWORD 和 VECTOR 不执行 RRF；多知识库 VECTOR 仅在存在多个 Embedding 指纹时使用上述内部固定融合规则，不读取请求 `rrf_k`；HYBRID 把每个知识库的关键词列表和向量列表视为独立排名列表，对全部有效列表执行 RRF，并使用请求实际生效的 `rrf_k`；
- Elasticsearch 中文索引和查询分别使用 ik_max_word 与 ik_smart，英文、数字和业务代码使用基础 Token 分析，避免精确型号、编号和专有名词在分词阶段丢失；
- 同一内容块只保留一条，并记录各路得分和排名；
- 并列结果以稳定 chunk_id 确定顺序，知识库传入顺序不影响结果；
- Elasticsearch 和 pgvector 返回候选标识、得分及必要定位信息，统一检索服务在最终返回前从 PostgreSQL 获取权威内容与状态并完成复核；仅因请求开始后新版本发布而变为 SUPERSEDED 的版本可继续服务该请求捕获的快照，MANUAL 停用、DELETING/DELETED、知识库停用、内容块停用和撤权始终按当前状态拒绝；无法证明时序时返回可重试的快照冲突，不返回错误空结果；
- Rerank 只处理候选集前 50 条；
- 固定 `BAAI/bge-reranker-v2-m3` 的部署版本升级时，先在隔离配置上完成最小推理，记录同环境新旧版本 P50/P95/P99 耗时，并使用相同的冻结验收知识库、IndexVersion、默认混合参数集和不少于 100 题比较新旧版本；评测一旦引用候选配置，该配置的模型身份、服务地址引用和运行参数冻结，更改须创建新配置并重新评测。新版本达到 Recall@10≥0.80、MRR@10≥0.60、权限泄露用例 100% 通过后，在 PostgreSQL 事务内复核评测所用配置与待激活配置一致，先将旧 ACTIVE 配置退役、再激活新配置。新请求读取新版本，旧配置保留到在途请求和评测运行结束；切换失败保持旧 ACTIVE 配置，回滚沿同一互斥路径执行，不创建新的 IndexVersion；
- Rerank 原始输出保存为 rerank_logit，使用固定 Sigmoid 转换得到 rerank_score；rerank_threshold 只作用于转换后的 rerank_score；
- Token 数使用平台冻结的 tokenizer 名称、版本和配置计算，并在响应生效参数中返回 tokenizer_version；超出预算时按完整返回块裁剪；
- Rerank 和阈值处理作用于召回的子块候选；之后将命中子块映射到父块，同一父块合并为一个结果，得分取最佳命中子块的最终得分，并保留全部命中子块编号和位置；top_n 和 Token 预算均按去重后的父块计算；
- 客户端断开、请求取消或到达强制超时时，取消信号向 Elasticsearch、pgvector 查询、Embedding 和 Rerank 调用传播；不再需要的下游结果不写入响应或缓存；
- 关键词高亮信息作为可选结果字段保留；返回元数据只允许 Schema 中标记为 response_visible 的业务字段，权限字段和内部状态字段不得透出；
- 空结果返回空数组和明确原因，不生成补充内容。

### 4.7 过滤设计

过滤分为两层：

1. **服务端强制过滤**：tenant_id、已授权 knowledge_base_id、当前 document_version、当前 index_version、document_status=PUBLISHED、chunk_enabled=true；指定文档时再校验 document_id。
2. **业务元数据过滤**：仅允许知识库 Schema 中声明且可过滤的字段，支持等于、不等于、包含、范围、IN、AND 和 OR。

过滤处理流程：

- 将 metadata_filter 解析为受控抽象语法树；
- 校验字段、类型、操作符及固定上限：嵌套深度 5、叶子条件 20、单个 IN 值 100、单个字符串值 256 个字符；日期值必须为 UTC ISO-8601；超限或类型错误返回 422，多知识库字段缺失或类型不兼容返回 `FILTER_FIELD_INCOMPATIBLE`；
- 将同一抽象语法树分别编译为 Elasticsearch DSL 和 PostgreSQL 条件；
- 禁止用户文本直接拼接 SQL 或 Elasticsearch DSL；
- 查询结束前对候选结果执行 PostgreSQL 权威状态复核，覆盖 Elasticsearch 状态更新的短暂延迟窗口。

有效期使用请求开始时的 UTC 时间，并应用：

valid_from 为空或 valid_from 小于等于当前时间；且 valid_to 为空或 valid_to 大于当前时间。

内容块启停采用非对称一致性流程：

- 每次启停在 PostgreSQL 中递增知识库 visibility_revision，Outbox 事件携带该 revision；消费者拒绝小于当前 revision 的乱序事件；
- 停用时先在 PostgreSQL 将权威 `chunk_enabled` 改为 false、记录目标状态与同步中状态并写入 Outbox；即使索引尚未同步，末端权威复核也立即阻止返回；
- 启用时先在 PostgreSQL 持久化“目标启用、当前仍停用、同步中、visibility_revision”并写入 Outbox，再同步更新 pgvector 和 Elasticsearch 中既有隐藏成员的可见性投影；两侧验证可查询且该 revision 仍为最新时，最后才将 PostgreSQL `chunk_enabled` 改为 true；任一后端更新失败时保持停用并返回可重试错误；
- 两侧查询应用 `chunk_enabled=true`，末端再复核 PostgreSQL 权威状态；只有启用操作成功返回后才承诺新请求可检索，失败不得出现“界面已启用但实际不可召回”；
- Scheduler 扫描 PostgreSQL 中持久化的同步中状态，重试未完成同步并对长期不一致告警，不依赖进程内存判断待补偿操作。

### 4.8 引用模块

- citation_id 稳定指向具体文档版本和内容块版本；
- 当前发布版本在再次鉴权后可以返回引用片段和受保护预览；
- 历史版本只返回文档名称、原版本号和“历史版本不可查看”，不返回正文或预览；
- 引用记录独立保存不可猜测 citation_id、知识库、文档和版本标识，但不复制正文；
- 来源删除后，只有调用方提交已知 citation_id，且满足以下条件之一，引用接口才返回“来源已删除”和最小标识：目标知识库仍存在时调用方当前授权有效；目标知识库自身已删除、无法继续保留当前授权关系时，删除事务在移除授权前保存的不含正文或密钥的历史授权快照证明该主体当时有权访问，且该关系未被后续撤销。人工通路快照保存 UserAccount 与直接 KnowledgePermission 的编号和版本；API Key 通路只有在删除时 ApiKeyScope 与所属 ApiAccount 的 KnowledgePermission 同时有效且均覆盖该知识库时才写入，并保存 ApiAccount、ApiKey 及两条关系的编号和版本。查询时必须使用同一 UserAccount 或同一 ApiKey，重新确认账号、密钥和会话当前有效；任一相关授权或密钥范围后续显式撤销均优先返回 403。检索、块详情、列表或任意编号探测仍统一返回 403；
- 预览地址必须短期有效并在生成前检查权限，不提供永久公开 URL。

### 4.9 检索测试与评测模块

- 持有 RETRIEVE 的人工账号可通过管理前端的“知识检索”和私有 API 调用统一检索服务，该通路只读取活动快照，不要求 EVALUATE；
- 单条检索测试要求 EVALUATE，复用统一召回、排序、过滤和权限实现，并额外展示各阶段候选、得分、过滤和耗时；管理测试可显式选择同一知识库中与 REVIEW_REQUIRED 文档对应的 READY IndexVersion，但不得选择 BUILDING、FAILED、RETIRED 或 DELETED 版本；结果标记 test_snapshot=true 且不改变 active_index_version，外部 REST/MCP 始终只能读取活动快照；
- 测试参数只对本次测试生效，不修改知识库在线配置；
- EvaluationSet 通过界面维护或 CSV 导入问题、期望命中文档/内容块、可选相关等级和备注；EvaluationSetVersion 保存不可变版本，EvaluationCase 归属于具体版本，EvaluationRun 冻结并引用具体版本，禁止运行期间跟随草稿变化；
- 批量评测通过异步任务执行，记录语料版本、文档版本、索引版本、模型版本和实际参数，并支持两个冻结参数集的结果对比；
- 计算 Recall@K、Precision@K、MRR@K、nDCG 和无结果率；
- EvaluationRun 启动时冻结评测集版本、活动索引版本、参数集、模型指纹、相关性标注版本和代码版本，生成不可变 run_fingerprint；启用 Rerank 时另冻结待评测的 Rerank 配置编号、模型版本及文件校验值，允许对尚未激活但已核验的部署版本执行回归；运行期间不跟随线上索引或 Rerank 配置切换；
- 父子块评测以 EvaluationCase 声明的期望文档/块粒度计算；返回父块时，只有其命中子块或父块标识满足该用例的期望集合才算命中，禁止在运行后改变计分口径；
- EVALUATE 权限允许查看该知识库的评测用例和受控命中片段，但不自动开放通用正文浏览；评测导出和查看敏感片段均审计；
- 评测 Worker 使用独立队列或并发配额，不能挤占在线查询和入库资源；
- 预发布环境建立与生产知识库隔离的验收知识库，只使用经公司批准且已脱敏的代表性语料；首次上线前冻结不少于 100 个真实业务问题、期望命中、语料/文档版本和默认混合检索参数集；
- 验收知识库上的默认混合检索参数集未达到 Recall@10 ≥ 0.80、MRR@10 ≥ 0.60 或权限用例未 100% 通过时，生成失败门禁记录并阻止平台首次生产发布；门禁不阻止已上线环境的业务文档发布，记录包含 run_fingerprint、指标、审批人和证据位置。

### 4.10 审计与检索追踪模块

- AuditLog 记录登录、密码重置、账号禁用、API Key、授权、状态变化、发布、删除和人工内容修改；
- RetrievalTrace 记录 request_id、trace_id、协议、调用主体、knowledge_base_ids、每库 index/schema/embedding/visibility 上下文、实际生效的 Rerank 配置编号/模型版本/文件校验值、参数、命中标识、得分、阶段耗时、降级信息和结果状态；未启用 Rerank 时相应版本字段为空；
- 原始查询正文不进入检索追踪，改用独立可轮换密钥计算 HMAC-SHA256 指纹，并记录字符数、hmac_key_id 和规范化规则版本；密钥轮换后新记录使用新 key_id，历史记录不重新计算；
- 日志不记录密码、API Key、完整向量、完整短期签名 URL、URL 中的 `e`/`token`、上传凭证、下载凭证、对象管理凭证或大段正文；对象操作只记录对象编号、空间代码、操作类型和结果；
- AuditLog 按追加写模式保存，应用运行身份对历史记录无 UPDATE/DELETE 权限；每条记录保存前一条与当前完整性校验值，定期校验并将结果写入受保护存储；降级、权限缓存失效失败、墓碑重放和管理端敏感正文查看均写入审计或安全事件日志。

## 5. 关键业务流程

### 5.1 文档入库流程

~~~mermaid
sequenceDiagram
    actor Operator as 运营人员
    participant API as 管理 API
    participant PG as PostgreSQL
    participant OS as 七牛云 Kodo
    participant Q as 任务队列
    participant D as Scheduler / Outbox 投递器
    participant W as Worker
    participant EMB as Embedding
    participant ES as Elasticsearch

    Operator->>API: 上传文件与元数据
    API->>API: 权限、扩展名、声明 MIME、空文件与流式大小校验
    API->>PG: 创建 UploadReservation 与平台暂存键
    API->>OS: HTTPS 使用平台键保存私有暂存对象
    API->>PG: 同一事务消费预留并创建 DocumentVersion、DocumentObject、IngestionTask、QUEUED Execution、OutboxEvent
    API-->>Operator: 返回任务编号
    D->>PG: 领取未投递 OutboxEvent
    D->>Q: 提交 task_id
    Q-->>D: 确认接收
    D->>PG: 标记事件已投递
    Q->>W: 分派任务
    W->>PG: 以租约代次领取任务
    W->>PG: 同一事务将 Task 和既有 QUEUED Execution 转 RUNNING，DocumentVersion 转 PROCESSING
    W->>OS: stat 复核，完成文件特征/资源/安全深检
    W->>OS: 幂等确认不可变正式对象并再次 stat
    W->>PG: DocumentObject 转 FORMAL
    W->>W: 安全解析、清洗、分块、受保护预览生成
    W->>EMB: 批量生成向量
    W->>PG: 写入草稿内容块、EmbeddingArtifact 与 BUILDING IndexVersion
    W->>ES: 写入版本化倒排索引
    W->>W: 双索引校验
    alt 校验成功
        W->>PG: 同一事务：IndexVersion READY，DocumentVersion REVIEW_REQUIRED，Task/Execution SUCCEEDED
    else 确定性失败
        W->>PG: 同一事务：已创建索引和 DocumentVersion/Task/Execution 转 FAILED
    else 临时依赖失败
        W->>PG: Task 转 RETRY_WAIT，Execution 记录失败，未校验索引不得转 READY
    end
~~~

### 5.2 文档发布与更新流程

发布前要求知识库处于 ACTIVE、文档版本处于 REVIEW_REQUIRED，且对应 IndexVersion 为 READY 并已完成双索引校验。发布事务同时完成：

- 对知识库取得发布互斥锁，并按 4.5.2 节校验 READY 索引的 base_active_index_version、visibility_revision、目标状态和墓碑；
- 新版本转为 PUBLISHED；
- 旧发布版本转为 DISABLED，并记录 disable_reason=SUPERSEDED；
- Document 更新当前发布版本引用；
- KnowledgeBase 更新 active_index_version；
- 新索引版本转为 ACTIVE，旧活动版本转为 RETIRED；
- 写入审计日志。

事务冲突或基础版本已变化返回 409。任何校验失败或事务失败均保持旧版本继续在线；过期 READY 索引不得直接重试发布，必须基于最新活动版本重新生成快照。

### 5.3 已发布内容块编辑

1. 复制当前发布版本及全部内容块形成新草稿；
2. 在副本中修改正文、关键词或允许覆盖的元数据；
3. 只为变化内容块重新生成向量；
4. 构建并校验新的 Elasticsearch 与 pgvector 索引版本；
5. 按发布流程原子切换；
6. 失败时继续使用原发布版本。

只有替换原文件或修改解析/分块策略时才重新解析和分块；操作前必须提示已有人工修改需要在新草稿中重新合并。正文、关键词或元数据编辑以及内容块启用/停用均记录操作者、时间和变更前后值。

内容块启用/停用不创建文档内容版本。停用先更新 PostgreSQL 权威状态，依靠末端复核立即阻断，再异步同步索引；启用只更新活动快照中既有隐藏成员的双索引可见性投影并确认可查询，最后更新 PostgreSQL 有效状态。任一同步失败时启用操作不得显示成功；活动快照缺少成员时必须先重建新快照，不得原地添加。

### 5.4 在线检索流程

1. 接入层完成协议和请求上限校验；
2. 接入层按入口鉴别 API Key 或人工会话，生成 PrincipalContext 并验证主体当前状态；
3. API Key 通路计算密钥与所属调用账号的知识库范围交集，人工会话通路使用当前直接授权；
4. 对全部目标知识库执行全有或全无授权校验；
5. 获取知识库状态、活动索引版本和元数据 Schema；
6. 编译服务端强制过滤与业务过滤；
7. 按模式执行关键词、向量或双路召回；
8. 完成融合、去重、稳定排序、可选 Rerank 和阈值处理；
9. 完成父块映射与去重，再按 top_n 与 max_context_tokens 裁剪；
10. 生成稳定引用和结构化结果；
11. 写入脱敏检索追踪并返回。

### 5.5 停用与删除流程

#### 停用

- KnowledgeBase 或当前 PUBLISHED DocumentVersion 先在 PostgreSQL 中改为 DISABLED；DocumentVersion 同时记录 disable_reason=MANUAL；
- 新请求读取状态后立即排除；
- 索引状态异步同步，但不能作为唯一阻断条件；
- 恢复时仅允许执行状态迁移矩阵中定义的路径。

#### 删除

每个已获需求状态矩阵支持的删除请求创建独立 DeletionTask，不复用 IngestionTask。DeletionTask 保存删除目标、PENDING/RUNNING/RETRY_WAIT/SUCCEEDED/FAILED 状态、当前固定阶段、进度、执行次数、租约持有者、execution_generation、心跳、下次重试时间、稳定错误码和墓碑账本序号；每次计划执行创建 DeletionExecution，Worker 领取后绑定租约，旧代次不得提交。阶段固定为墓碑落账、后继快照、对象与正文清理、向量/倒排索引/缓存清理、完整性复核；失败从当前幂等阶段重试，人工重试按 `FAILED -> PENDING` 创建新执行记录。

1. 管理 API 完成权限、状态矩阵前置检查和二次确认；删除知识库时，确认界面展示当前文档数、内容块数和已授权 API Key 数量。锁定目标并在初始事务中写入墓碑、DeletionTask 和 OutboxEvent；知识库删除还同步改为 DELETING，单文档删除立即由墓碑屏蔽。关联 PENDING/RETRY_WAIT 入库任务按既有取消状态迁移停止领取，已有运行执行收到取消请求；Worker 领取和提交阶段均复核墓碑；
2. DocumentVersion 不得跳过第 6.2 节状态矩阵：无有效入库执行的 UPLOADED、FAILED、REVIEW_REQUIRED、PUBLISHED、DISABLED 可转 DELETING；PROCESSING 必须等待关联执行停止、租约失效和未发布派生数据清理完成，再按 `PROCESSING -> FAILED -> DELETING` 迁移。删除单个 Document 不为其稳定身份引入需求外状态。UPLOADED 版本的暂存/正式原文件纳入后续对象清理；
3. Worker 在 DeletionExecution 的有效租约下，把墓碑按 tombstone_id 幂等追加到独立于普通业务备份保留周期的只追加账本，记录对象类型、对象编号、删除时间、删除版本、清理范围和可验证账本序号；追加未成功时保持删除屏蔽并重试，不得开始不可逆物理清理。Scheduler 只负责任务投递、租约恢复和巡检，不另行执行墓碑追加；
4. 新请求立即通过状态过滤拒绝访问；
5. 删除单个文档且目标内容存在于当前活动 IndexVersion 时，基于当前活动版本构建排除目标文档的后继完整逻辑快照；双索引校验后按发布互斥和 base_active_index_version 条件原子切换。未进入活动快照的草稿/失败版本以及删除整个知识库时无需生成后继快照；
6. 后继快照构建期间，DELETING 状态和墓碑覆盖继续保证目标内容对新请求不可见；构建失败保持不可见并进入可重试状态，不恢复目标内容；
7. 已取得可验证账本序号、关联入库执行已停止且租约失效，并且需要后继快照的目标已完成快照切换后，异步清理每个 DocumentObject 记录的原文件和预览对象键、解析产物、内容块正文、已无快照引用的 EmbeddingArtifact、旧倒排索引和缓存；第 5 步明确无需后继快照的目标跳过切换门禁。Kodo 删除中“对象不存在”按幂等成功处理，每个键删除后再查询对象信息确认不存在，记录校验时间和结果；
8. 全部原文件/预览对象、正文和派生数据均已清理并复核后，在同一 PostgreSQL 事务中将知识库或相关 DocumentVersion 改为 DELETED、DeletionTask 改为 SUCCEEDED，并记录 cleanup_completed_at 和账本序号；
9. 墓碑落账、后继快照、清理或复核失败时，DeletionExecution 记录阶段和稳定错误，DeletionTask 进入 RETRY_WAIT 并按退避时间重投；超过自动重试次数后进入 FAILED 并告警，人工重试创建新执行记录；
10. 审计和检索追踪只保留允许的标识与操作信息，不能恢复正文；
11. 备份恢复时先将业务数据置于不可对外服务状态，再从恢复点之后的墓碑账本水位重放删除，完成复核后方可恢复检索流量。

删除墓碑账本至少跨越全部业务备份生命周期，并采用追加写、完整性校验和独立备份。墓碑重放按 tombstone_id 幂等执行；账本不可用或水位无法确认时，恢复环境保持只读且不得上线。

## 6. 状态与并发控制

### 6.1 KnowledgeBase 状态

~~~mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> ACTIVE
    ACTIVE --> DISABLED
    DISABLED --> ACTIVE
    DRAFT --> DELETING
    ACTIVE --> DELETING
    DISABLED --> DELETING
    DELETING --> DELETED
~~~

只有 ACTIVE 知识库中的 PUBLISHED 文档可以参与检索。

DRAFT -> ACTIVE 要求 MANAGE 权限并通过默认语言、分块配置和元数据 Schema 校验；DISABLED -> ACTIVE 前必须确认当前活动 IndexVersion 的双索引仍完整可查询，无活动版本的空库可恢复但检索返回空结果。

### 6.2 DocumentVersion 状态

~~~mermaid
stateDiagram-v2
    [*] --> UPLOADED
    UPLOADED --> PROCESSING
    UPLOADED --> DELETING
    PROCESSING --> REVIEW_REQUIRED
    PROCESSING --> FAILED
    FAILED --> PROCESSING
    FAILED --> DELETING
    REVIEW_REQUIRED --> PUBLISHED
    REVIEW_REQUIRED --> DELETING
    PUBLISHED --> DISABLED
    PUBLISHED --> DELETING
    DISABLED --> PUBLISHED
    DISABLED --> DELETING
    DELETING --> DELETED
~~~

数据库必须保证同一 Document 最多一个 PUBLISHED 版本。

DISABLED --> PUBLISHED 不是普通状态恢复：disable_reason=MANUAL 时仅允许恢复该 Document 当前版本，并按 4.3 节先恢复双索引可查询状态；disable_reason=SUPERSEDED 时只能执行显式单文档回滚，以当前活动快照为基线构建新的完整 IndexVersion。两类操作均须校验双索引、visibility_revision、墓碑和并发版本；直接修改状态一律拒绝。

### 6.3 IndexVersion 状态

~~~mermaid
stateDiagram-v2
    [*] --> BUILDING
    BUILDING --> READY
    BUILDING --> FAILED
    FAILED --> BUILDING
    FAILED --> DELETED
    READY --> ACTIVE
    READY --> DELETED
    ACTIVE --> RETIRED
    RETIRED --> ACTIVE
    RETIRED --> DELETED
~~~

数据库必须保证同一 KnowledgeBase 最多一个 ACTIVE IndexVersion。

READY 版本因基础活动版本、可见性或墓碑冲突而过期时，只能清理后转 DELETED，不得回到 BUILDING。RETIRED --> ACTIVE 仅允许 4.5.2 节定义的整库快照回滚。

### 6.4 IngestionTask 状态

~~~mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> RUNNING
    RUNNING --> SUCCEEDED
    RUNNING --> RETRY_WAIT
    RETRY_WAIT --> PENDING
    RUNNING --> FAILED
    FAILED --> PENDING : 人工重试
    PENDING --> CANCEL_REQUESTED : 用户取消
    RUNNING --> CANCEL_REQUESTED
    RETRY_WAIT --> CANCEL_REQUESTED
    CANCEL_REQUESTED --> CANCELLED : 无派生数据
    CANCEL_REQUESTED --> CLEANUP_PENDING : 已有派生数据
    CLEANUP_PENDING --> CANCELLED
    CLEANUP_PENDING --> CLEANUP_RETRY_WAIT
    CLEANUP_RETRY_WAIT --> CLEANUP_PENDING
    CLEANUP_RETRY_WAIT --> CLEANUP_FAILED
    CLEANUP_FAILED --> CLEANUP_PENDING : 人工重试清理
~~~

- PENDING 表示任务已落库，可尚未成功进入队列；Scheduler 根据未投递 OutboxEvent 或超过派发时限的 PENDING 状态补投；
- RUNNING 必须持有带 execution_generation 的有期租约并定期心跳；租约过期后由 Scheduler 判断是否重试，旧 TaskExecution 转为 LOST 且不得继续提交；
- RETRY_WAIT 保存下次执行时间和稳定失败分类；确定性错误直接进入 FAILED；
- FAILED 仅允许通过人工重试回到 PENDING；该调度事务创建一条新的 QUEUED TaskExecution 和 OutboxEvent；
- CLEANUP_RETRY_WAIT 只重试未发布派生数据清理，不执行解析、Embedding 或索引构建；CLEANUP_FAILED 人工重试只能回到 CLEANUP_PENDING；
- 每次初始调度、RETRY_WAIT 到期重调度或 FAILED 人工重试各创建且仅创建一条 QUEUED TaskExecution；PENDING -> RUNNING 只为该记录绑定租约和 execution_generation 并改为 RUNNING，历史执行不可覆盖；
- 每次计划进入 CLEANUP_PENDING 时创建 execution_type=CLEANUP、状态为 QUEUED 的 TaskExecution；Worker 领取时更新该记录，不能复用或覆盖原入库执行记录；
- SUCCEEDED 仅表示入库产物校验完成；双索引校验成功时，IndexVersion -> READY、DocumentVersion -> REVIEW_REQUIRED、IngestionTask/TaskExecution -> SUCCEEDED 在同一 PostgreSQL 事务中完成，不表示已发布；
- 确定性失败时，已创建 IndexVersion、DocumentVersion、IngestionTask 和 TaskExecution 分别转为各自 FAILED；临时依赖失败只将 IngestionTask 转为 RETRY_WAIT 并结束本次 TaskExecution，未校验产物不得转 READY；
- 有派生数据的取消任务只有在清理完成后才能转为 CANCELLED；CLEANUP_RETRY_WAIT 或 CLEANUP_FAILED 均不是取消终态。

取消以文档实际状态和是否存在派生数据为准：重试调度后的 PENDING 不能视为从未执行。仅 DocumentVersion=UPLOADED 且无派生数据时保持 UPLOADED；已进入 PROCESSING 的版本取消后转 FAILED，已有 FAILED 版本保持 FAILED。RETRY_WAIT 取消同样执行派生数据清理判定，不创建新的入库执行；未领取的 QUEUED 执行标记 CANCELLED，已结束的失败执行保留原结果。

### 6.5 并发与幂等

- 状态变更采用数据库事务和条件更新，提交时验证目标聚合的 expected row_version，涉及配置时同时验证 expected config_version；
- 发布、回滚、删除、人工编辑、账号/授权变更等写操作要求幂等键，并记录首次响应摘要；
- 同一文档版本不允许两个入库任务同时写入；
- 重复任务消息可以重复消费，但不能产生重复可见内容块；
- 同一知识库发布切换使用数据库级互斥和 base_active_index_version 条件更新；构建可并行，切换必须串行；
- 非法状态迁移、并发发布和配置版本冲突统一返回 409；
- 检索请求只读，客户端安全重试不产生业务副作用。

## 7. 数据架构

### 7.1 存储职责

| 存储 | 数据职责 | 一致性定位 |
| --- | --- | --- |
| PostgreSQL | 账号、权限、知识库、文档、版本、内容块、任务、Outbox、活动索引、评测、追踪、审计和墓碑操作记录 | 业务事实来源 |
| pgvector | 内容块向量和向量索引 | PostgreSQL 内可重建派生数据 |
| Elasticsearch | 关键词倒排索引、高亮字段和可过滤字段 | 可重建派生数据 |
| 七牛云 Kodo | 暂存原文件、不可变正式原文件和受保护预览资源 | 公司七牛云账号下的私有空间；对象键由平台生成；不保存解析文本、向量或查询数据 |
| 删除墓碑账本 | 跨业务备份恢复点保留的追加写删除事件和账本水位 | 恢复安全依据，不保存正文 |
| 缓存 | 权限、配置和查询辅助缓存 | 仅性能优化，不作为唯一判断依据 |

### 7.2 核心对象关系

~~~mermaid
erDiagram
    Tenant ||--o{ UserAccount : contains
    Tenant ||--o{ ApiAccount : contains
    Tenant ||--o{ KnowledgeBase : isolates
    Tenant ||--o{ RoleDefinition : defines
    Tenant ||--o{ OutboxEvent : records
    ApiAccount ||--o{ ApiKey : owns
    ApiKey ||--o{ ApiKeyScope : limits
    KnowledgeBase ||--o{ ApiKeyScope : scopes
    UserAccount o|--o{ KnowledgePermission : may_receive
    ApiAccount o|--o{ KnowledgePermission : may_receive
    KnowledgeBase ||--o{ KnowledgePermission : grants
    UserAccount ||--o{ LoginSession : opens
    UserAccount ||--o{ UploadReservation : initiates
    UserAccount ||--o{ UserRole : assigned
    RoleDefinition ||--o{ UserRole : grants
    KnowledgeBase ||--o{ Document : contains
    KnowledgeBase ||--o{ IndexVersion : publishes
    KnowledgeBase ||--o{ MetadataSchemaVersion : defines
    KnowledgeBase ||--o{ ChunkingConfigVersion : defines
    KnowledgeBase ||--o{ RetrievalParameterSet : evaluates
    KnowledgeBase ||--o{ EvaluationSet : owns
    Document ||--o{ DocumentVersion : versions
    DocumentVersion ||--o{ DocumentObject : owns
    UploadReservation o|--o| DocumentObject : consumed_as
    DocumentVersion ||--o{ DocumentArtifact : produces
    DocumentVersion ||--o{ Chunk : contains
    DocumentVersion ||--o{ IngestionTask : processes
    IngestionTask ||--o{ TaskExecution : executes
    IndexVersion ||--o{ IndexSnapshotMember : contains
    Chunk ||--o{ IndexSnapshotMember : snapshots
    EmbeddingArtifact ||--o{ IndexSnapshotMember : supplies
    EmbeddingModelFingerprint ||--o{ IndexVersion : binds
    MetadataSchemaVersion ||--o{ IndexVersion : binds
    ChunkingConfigVersion ||--o{ IndexVersion : binds
    Chunk o|--o{ Citation : may_identify
    EvaluationSet ||--o{ EvaluationSetVersion : versions
    EvaluationSetVersion ||--o{ EvaluationCase : contains
    EvaluationSetVersion ||--o{ EvaluationRun : executes
    IndexVersion ||--o{ EvaluationRun : freezes
    RetrievalParameterSet ||--o{ EvaluationRun : uses
    ApiAccount o|--o{ RetrievalTrace : may_invoke
    UserAccount o|--o{ RetrievalTrace : may_invoke
    ApiKey o|--o{ RetrievalTrace : may_authenticate
    Tenant ||--o{ AuditLog : records
    Tenant ||--o{ DeletionTombstone : protects
    Tenant ||--o{ DeletionTask : schedules
    DeletionTombstone ||--|| DeletionTask : drives
    DeletionTask ||--o{ DeletionExecution : executes
~~~

关键对象语义：

- Tenant 在 MVP 中只有一条固定记录，不提供创建、切换或停用租户的接口；
- KnowledgePermission 的主体必须且只能是 UserAccount 或 ApiAccount 之一，不能同时关联两者；ApiKeyScope 表达密钥自身知识库范围，实际范围仍与 ApiAccount 的 KnowledgePermission 求交集；
- DocumentObject 只保存平台对象标识、对象键、校验信息和 STAGED/FORMAL/REJECTED/DELETED 生命周期状态，不保存签名 URL 或访问凭证；DocumentArtifact 表示保存在 PostgreSQL 中的分段解析产物；
- UploadReservation 只用于封闭“对象已上传但业务事务未提交”窗口；成功上传时在业务事务中消费，超时且未转为 DocumentObject 时才允许孤儿扫描处理；
- IndexSnapshotMember 固化一个 IndexVersion 包含的 Chunk 版本及正文、关键词、业务元数据和稳定定位字段，不从对象“当前状态”动态推导；可变授权不写入成员，可见性投影按 visibility_revision 单独同步；
- EmbeddingArtifact 保存可跨文档版本复用的不可变向量，Chunk 和 IndexSnapshotMember 只保存引用；
- 每个 IndexVersion 固定绑定一个 EmbeddingModelFingerprint，同一指纹可以被多个 IndexVersion 复用；Rerank 和 tokenizer 版本记录在查询追踪及评测运行中；
- EvaluationSetVersion 是评测问题、期望命中和相关性标注的不可变版本边界，EvaluationRun 必须引用具体版本；
- RetrievalTrace 的调用主体只能为两种组合之一：人工通路关联 UserAccount，外部通路同时关联 ApiAccount 和 ApiKey；
- Citation 只保存稳定定位和允许保留的来源标识，不复制正文；删除后允许块关联为空，不用强制外键阻止正文清理；
- DeletionTombstone 同时保存在 PostgreSQL 操作记录和独立追加写账本中；
- DeletionTask 只承载知识库或文档删除编排，目标类型和目标编号必须且只能指向一个删除聚合；DeletionExecution 保存每次调度的执行历史，不覆盖失败记录；
- OutboxEvent 通过聚合类型和聚合编号表达入库、授权、删除、评测和状态同步等通用来源，不专属于 IngestionTask。

### 7.3 关键数据库约束

- 核心主键为 UUIDv7；
- 人工账号规范化邮箱唯一；
- 知识库名称全平台唯一；
- 同一主体与知识库不得存在语义冲突的多条有效授权；
- 同一 ApiKey 与 KnowledgeBase 最多一条有效 ApiKeyScope，且范围不得超过所属 ApiAccount 的当前授权；
- 同一 Document 最多一个 PUBLISHED DocumentVersion；
- 同一 KnowledgeBase 最多一个 ACTIVE IndexVersion；
- READY/ACTIVE IndexVersion 必须具有完整快照成员、Embedding 模型指纹、Schema 版本和可验证内容摘要；
- IndexVersion 必须保存构建基础活动版本和可见性代次，切换时执行条件复核；
- DocumentObject 的对象键唯一，FORMAL 对象不允许覆盖；
- 同一文档版本最多一个处于 PENDING、RUNNING、RETRY_WAIT、CANCEL_REQUESTED、CLEANUP_PENDING、CLEANUP_RETRY_WAIT 或 CLEANUP_FAILED 的有效 IngestionTask；清理未完成前不得创建新的入库任务；
- 同一删除目标最多一个 PENDING、RUNNING、RETRY_WAIT 或 FAILED 且尚未收尾的 DeletionTask；每次执行必须校验租约和 execution_generation；
- OutboxEvent 使用业务类型和业务幂等键唯一约束防止重复创建；
- Citation、AuditLog、RetrievalTrace 和 DeletionTombstone 不通过业务对象物理删除破坏历史关系，但不得保留可恢复正文；
- 内容块标识在文档版本范围内稳定且可追溯；
- 状态迁移、发布和删除均保留审计关系，不通过物理删除破坏历史记录；
- 具体字段、唯一索引、外键、分区和 DDL 在数据库设计中定义。

## 8. 检索策略设计

### 8.1 三种模式

| 模式 | 执行路径 | 必需依赖不可用 |
| --- | --- | --- |
| KEYWORD | Elasticsearch 关键词召回 | 返回 503 RETRIEVAL_UNAVAILABLE |
| VECTOR | 本地查询 Embedding + pgvector 召回；多模型指纹时分组召回后 RRF | 任一目标指纹的 Embedding 或 pgvector 不可用时，整次返回 503 RETRIEVAL_UNAVAILABLE |
| HYBRID | 每知识库两路召回 + RRF | 移除失败的排名列表并使用剩余列表；无任何有效列表时返回 503，否则返回带 degraded 信息的结果 |

Rerank 不可用时返回召回或融合排序，并设置 degraded、degraded_stage 和 degraded_reason。认证、权限、活动版本或状态过滤失败时禁止降级。

### 8.2 调用参数

| 参数 | 默认值 | 范围 |
| --- | --- | --- |
| retrieval_mode | HYBRID | KEYWORD、VECTOR、HYBRID |
| keyword_candidate_k | 50 | 1～200 |
| vector_candidate_k | 50 | 1～200 |
| rrf_k | 60 | 1～200 |
| rerank | true | true/false |
| rerank_threshold | 0.50 | 0～1 |
| top_n | 10 | 1～50 |
| max_context_tokens | 8000 | 1000～32000 |

HTTP 与 MCP 共享上述参数定义、默认值、范围和响应中的实际生效参数。`rrf_k` 仅在 HYBRID 模式生效；VECTOR 跨多个 Embedding 指纹时使用 4.6 节定义的服务端固定融合常量，不读取该请求参数。知识库不保存独立在线检索参数。

请求级固定限制为：query 最多 4,000 个字符，单次最多 20 个 knowledge_base_id 和 100 个 document_id；过滤 DSL 限制见 4.7 节。HTTP 和 MCP 使用同一校验器，超限时不执行检索。

服务端通过 per_knowledge_base_context 列表返回每个实际检索知识库的 knowledge_base_id、index_version、metadata_schema_version、embedding_model_fingerprint 和 visibility_revision，查询级单独返回 rerank_model_fingerprint 和 tokenizer_version。参数合法但受故障降级影响未执行时，响应分别记录 requested_parameters 与 effective_parameters，不能把降级后的行为伪装成请求参数已完整执行。

### 8.3 得分语义

- Rerank 成功时保留模型原始 rerank_logit，并按 `1 / (1 + exp(-rerank_logit))` 计算 rerank_score；score 为 rerank_score，score_type 为 RERANK；
- 关键词模式未 Rerank 时使用关键词得分，score_type 为 KEYWORD；
- 向量模式未 Rerank 时使用余弦相似度，score_type 为 VECTOR；
- 多知识库 VECTOR 模式因多个 Embedding 指纹执行跨列表融合时，最终 score 使用 RRF 得分，score_type 为 RRF，各候选仍保留原向量得分和排名；
- 混合模式未 Rerank 时使用 RRF 得分，score_type 为 RRF；
- 不同 score_type 的 score 不允许直接跨类型比较；
- 响应保留可获得的原始得分、排名、融合得分和重排前后位置。

阈值比较、稳定排序和评测均使用未格式化的内部数值；JSON 展示精度不反向参与排序。不同知识库或不同模型指纹的原始向量得分不得直接合并，跨库统一排序依赖排名融合或 Rerank。

## 9. 接口边界设计

### 9.1 对外 REST 能力

| 方法与路径 | 责任 |
| --- | --- |
| GET /api/v1/retrieval/knowledge-bases | 返回当前 API 调用账号可检索知识库 |
| POST /api/v1/retrieval/search | 执行统一检索 |
| GET /api/v1/retrieval/chunks/{chunk_id} | 返回当前发布版本且有权读取的内容块 |
| GET /api/v1/retrieval/citations/{citation_id} | 返回当前引用或历史/删除状态 |

### 9.2 MCP 能力

| 工具 | 对应领域能力 |
| --- | --- |
| list_knowledge_bases | 可检索知识库列表 |
| search_knowledge | 统一检索 |
| get_chunk | 内容块或引用读取 |

HTTP 与 MCP 的领域请求和响应模型从同一契约映射，禁止各自维护不同的排序、权限和错误逻辑。

`search_knowledge` 返回超过 MCP 或上下文限制时只按完整内容块裁剪，并返回 `truncated=true` 及实际返回的 Token 数；未裁剪时返回 `truncated=false`。

### 9.3 Python 与框架适配

- 轻量 Python 客户端提供同步和异步 `retrieve(query, knowledge_base_ids, filters, options)`，交付最小接入、引用传递和空结果处理示例；
- RagBaseRetriever 符合 LangChain BaseRetriever 语义，支持 invoke 和 ainvoke；默认通过 HTTP 调用，调用方可显式选择 MCP；两种传输映射同一领域请求、响应、权限、错误、得分和引用语义，语义不一致视为兼容测试失败；
- 每个检索结果映射为 LangChain Document：page_content 是内容块文本，metadata 包含知识库、文档、版本、页码/幻灯片/工作表、块编号、得分、引用编号和 trace_id；
- LangGraph 提供可组合的异步检索节点与 MCP 调用示例，节点只返回可合并到业务图状态的 Document 列表、引用列表和 trace_id，不定义业务项目的 State Schema、路由或会话状态；
- 客户端只负责调用、超时、错误映射和结果转换，不执行 Embedding 或直接访问搜索引擎；
- LangChain MCP 集成使用当前维护的 `langchain.mcp` 能力，不把已停止维护的 `langchain-mcp-adapters` 作为基础依赖；
- 集成包与服务端独立发布并声明 Python、langchain-core、LangGraph 及服务端的兼容版本矩阵，对支持范围的最低/最高版本执行自动化验证；示例只使用占位 API Key。

### 9.4 私有管理能力

私有管理 API 按以下领域分组，均使用 `/internal/api/v1` 前缀并生成独立 OpenAPI：

| 领域 | 核心能力 | 关键控制 |
| --- | --- | --- |
| 会话与人工账号 | 登录、退出、改密、重置、锁定、禁用、恢复 | 登录限流、会话撤销、ACCOUNT_ADMIN、审计 |
| API 调用账号与密钥 | 创建、授权、禁用、轮换、删除 | 密钥只展示一次、范围交集、并存轮换、审计 |
| 知识库与授权 | 创建、配置、启停、授权、删除；按账号/知识库双向查看授权 | MANAGE、同一授权事实、配置版本、二次确认、幂等 |
| 知识检索 | 人工账号对活动快照执行普通检索 | RETRIEVE、当前授权、脱敏追踪 |
| 文档与任务 | 上传、任务查询、取消、重试、审核、发布、停用、删除 | EDIT、状态机、Outbox、发布锁、审计 |
| 元数据 Schema | 新版本、字段停用、迁移和重建 | MANAGE、类型不可原地修改、兼容性校验 |
| 内容块 | 预览、草稿编辑、启停 | EDIT、正文版本化、运营状态同步 |
| 检索测试与评测 | 活动/READY 快照单条测试、评测集版本、CSV 导入、运行、对比、门禁记录 | EVALUATE、测试快照限制、快照冻结、资源隔离 |
| 平台运维 | 模型运行参数、节点、队列、容量、追踪、告警 | PLATFORM_ADMIN、计算设备外置配置、模型基线只读、正文最小暴露 |

重复提交可能产生重复副作用的管理写操作接受 Idempotency-Key；全部管理接口在 OpenAPI 中声明所需平台角色、知识库权限、前置状态、成功后状态、并发版本和稳定错误码。管理前端不得绕过 API 直接修改 PostgreSQL、Elasticsearch 或对象存储。

### 9.5 错误模型

| HTTP 状态 | 领域语义 |
| --- | --- |
| 400/422 | 参数、过滤、类型或资源上限错误 |
| 401 | 认证凭证无效：人工会话过期/撤销，或 API Key/所属调用账号无效 |
| 403 | FORBIDDEN_SCOPE；不存在、删除、无权访问对外统一处理；已知 citation_id 的受控墓碑查询按下述例外处理 |
| 404 | RESOURCE_NOT_FOUND；仅限具备管理范围的内部管理 API |
| 409 | 并发、非法状态迁移、配置版本或发布冲突 |
| 429 | 配额、频率或并发超限 |
| 503 | 当前操作的必需依赖不可用；检索依赖按 8.1 节先判断能否降级，上传、任务或管理依赖返回对应稳定不可用错误码 |
| 504 | 在线检索或当前操作的下游依赖超时 |

错误响应统一包含 trace_id、稳定错误码、简短说明和是否可重试。MCP 将相同内部错误枚举映射为协议错误对象，不暴露堆栈、路径或资源存在性。

引用例外只适用于引用详情或 MCP get_chunk 的 citation_id 形态，并严格使用 4.8 节的判定顺序：先复核主体、会话/API Key 和显式撤权状态，再校验当前或未撤销的历史授权。通过后可返回 CURRENT、HISTORICAL_UNAVAILABLE 或 SOURCE_DELETED；任一优先拒绝条件成立时返回 403。知识库列表、搜索、普通 chunk_id 读取和任意资源编号探测不适用该例外。

## 10. 安全设计

### 10.1 信任边界

- 管理前端和私有管理 API 位于内网管理边界；
- HTTP/MCP 调用经过检索入口、TLS、API Key 鉴权和限流；
- 所有网络通信使用 TLS；内部调用同时使用受信网络和服务身份认证；
- Worker 编排进程只能按白名单访问 PostgreSQL、Redis、Kodo、Elasticsearch 和本地模型服务；不可信文件的具体格式解析在无数据库/密钥凭证且禁止出站网络的隔离子进程或容器中执行；
- 数据区和模型区只允许应用与任务执行单元按最小权限访问；
- API、MCP、Worker、Scheduler、模型服务和基础设施之间使用受信网络叠加服务身份认证；仅处于同一内网不能替代服务认证；
- 对外出站流量统一通过代理、域名白名单和审计；七牛云 Kodo 仅允许访问已配置的上传、管理和私有下载域名；每次外部调用记录目标服务、调用类型、耗时和结果，不记录密钥、凭证或完整正文。

HTTP 与 MCP 共用以 API Key 编号和 ApiAccount 编号为主键的分布式限流计数，知识库维度和全局并发作为附加限制。限流存储不可用时，外部 REST/MCP 检索统一失败关闭，返回 503 `RATE_LIMIT_UNAVAILABLE` 并写安全事件；健康检查和内部恢复使用与外部密钥无关的独立、固定上限配额。不得因切换 HTTP/MCP 协议获得新额度。

### 10.2 数据保护

- 数据库、七牛云 Kodo、日志和备份启用静态加密；
- 密钥与业务数据分离保管，不写入代码、镜像或普通配置；
- 密钥通过公司批准的密钥管理或独立密钥托管机制引用；备份保存的是加密数据和受控恢复所需的密钥引用/托管材料，不把可直接解密业务数据的明文密钥与同一份备份共同存放；
- 密钥轮换记录 key_id、启用时间、退役时间和用途；API 签名、HMAC 指纹、七牛云 AccessKey/SecretKey 和备份密钥采用相互独立的用途域；
- 七牛云 Kodo 使用私有空间和服务端加密；首次接受上传前必须验证空间为私有且服务端加密已启用。该加密只对启用后新上传的对象生效；如已有对象早于加密开启时间，必须在开启后重新写入并验证后才可通过上线检查；
- 原文件与预览只允许在业务服务完成当前权限校验后生成短期有效私有下载地址，不提供永久公开 URL；签名 URL 和其中的到期时间/令牌不写入日志、数据库或任务消息；
- 七牛云访问密钥仅由 API、Worker 和 Scheduler 按角色通过密钥管理机制读取，并使用相互独立、最小权限的用途凭证：API 负责受控上传/下载签发，Worker 负责对象校验、预览写入、转正和删除，Scheduler 只读巡检并投递清理任务；管理前端、日志和任务消息不得包含 SecretKey；
- 文档、查询和候选结果不得发送给外部 Embedding 或 Rerank 服务；
- 日志分级脱敏，禁止记录密码、API Key、模型密钥和完整正文；
- 删除覆盖原文件、解析文本、内容块、向量、倒排索引、缓存和预览。

### 10.3 输入安全

- 校验扩展名、MIME、文件特征、大小、页数、解压大小和格式专属上限；
- 防止路径穿越、压缩炸弹、恶意文档、SSRF、越权下载和查询注入；
- HTML 不访问外部 URL，文档中的宏、脚本、命令和嵌入对象不执行；
- metadata_filter 经受控语法树编译，查询文本不直接拼接后端查询；
- 查看正文、预览和引用均在请求时重新鉴权。

管理入口限制可信 Origin 和 Host，登录与敏感写操作执行 CSRF 防护、重放保护和频率限制。格式解析隔离环境使用非特权身份、只读基础镜像、临时工作目录及文件系统/CPU/内存/执行时间限制，不得继承 Worker 编排进程的数据库、Kodo 或密钥权限。

## 11. 非功能设计

### 11.1 性能与容量

- API 进程与 Worker 分开扩展，在线检索不消费入库工作池；
- Embedding、Rerank、解析和索引写入配置独立并发上限；
- 请求并发、队列长度、超时和限流均外置配置；
- 上传和解析采用流式或分段方式；
- 普通管理 API、上传接口和异步任务分开记录性能，不用异步任务耗时掩盖在线接口问题；
- 检索指标区分关键词、向量、混合和 Rerank 开关；对相同查询集与参数分别记录 HTTP/MCP 总耗时和协议转换耗时；
- 当前不承诺固定 QPS、延迟和可用性数值，先形成可复现基线；
- 性能基线报告必须同时记录数据量、查询集、并发量、模型、硬件和索引配置，否则不作为容量或扩容依据；
- 容量使用率达到 70%、85%、95% 时分别产生逐级升级的容量告警；三个触发点作为需求固定值，不由性能基线改写。容量压力不触发已发布内容的自动删除。

### 11.2 可用性与故障处理

- API、MCP、Worker、Scheduler 和模型服务分别提供 liveness、startup 与能力级 readiness。liveness 只判断进程存活，不访问重依赖；startup 只校验该运行角色的本地配置、初始化和数据库 Schema 兼容性，不把 Kodo、搜索或模型的临时故障误判为进程启动失败；
- 健康响应分别报告 PostgreSQL/pgvector、Elasticsearch、Redis Broker/Cache、Kodo、Embedding 和 Rerank，并输出下表能力的 READY、DEGRADED 或 UNREADY，不用单一总状态替代能力判断。Kodo 检查通过官方 SDK查询部署时预置的不可变认证探针对象，不写入业务对象；

| 能力 | readiness 必需依赖 | 失败影响 |
| --- | --- | --- |
| 管理基础 | PostgreSQL | 管理基础能力 UNREADY，不连带判定独立检索进程失活 |
| 上传 | PostgreSQL、Kodo 认证探针、空间私有性和服务端加密校验 | 上传 UNREADY；无需 Kodo 的检索能力独立判断 |
| KEYWORD | PostgreSQL、限流存储、Elasticsearch | KEYWORD UNREADY；HYBRID 按向量链路决定 DEGRADED 或 UNREADY |
| VECTOR | PostgreSQL/pgvector、限流存储、Embedding | VECTOR UNREADY；HYBRID 按关键词链路决定 DEGRADED 或 UNREADY |
| HYBRID | KEYWORD/VECTOR 至少一条召回链路可用 | 单链可用为 DEGRADED，两链均不可用为 UNREADY；具体请求仍按 8.1 节处理 |
| Rerank | Rerank 服务 | 仅 Rerank 能力 UNREADY，召回能力保持 READY/DEGRADED，并在响应中降级 |
| Worker | PostgreSQL、Redis Broker 和当前任务阶段所需依赖 | 只停止领取依赖不满足的任务，既有在线检索不受影响 |
| Scheduler | PostgreSQL；Redis Broker 单独报告投递状态 | PostgreSQL 不可用时编排 UNREADY；Redis 不可用时投递 UNREADY，但进程存活且 Outbox 留在 PostgreSQL |

- Embedding 与 Rerank 调用配置超时、并发隔离和熔断；
- 入库任务支持异常中断识别、有限重试和人工重试；
- 索引切换失败继续使用旧活动版本；
- 任务队列或 Worker 故障不阻断已发布内容的在线检索；
- Elasticsearch、pgvector 和七牛云 Kodo 异常分别产生可定位告警；
- MVP 同一环境只允许一个 Scheduler 持有 PostgreSQL 领导锁；锁丢失后立即停止新投递与新巡检，周期任务仍使用幂等键防止重复触发，不引入独立调度集群；
- 进程优雅停机时先退出就绪状态，停止领取新任务，在宽限期内完成或释放任务租约，并取消不再需要的下游模型请求。

### 11.3 可观测性

统一观测维度包括：

- request_id、trace_id、task_id、knowledge_base_ids 和每库 index_version/visibility_revision；
- HTTP 路径或 MCP 工具名、调用主体和结果状态；
- 权限过滤、查询向量、关键词召回、向量召回、融合、Rerank 和返回阶段耗时；
- QPS、成功率、P50/P95/P99、无结果率、降级率和错误码分布；
- 队列长度、任务阶段、重试次数、索引大小、存储使用量和模型调用状态。

告警至少覆盖检索错误率上升、延迟异常、降级率异常、任务积压、索引不一致、模型不可用和存储不足。

每类告警必须配置责任模块、严重级别、抑制/聚合规则、诊断入口和处置手册。告警通过测试环境故障注入产生过真实事件后，才能计入上线证据。

### 11.4 备份与恢复

- 每个 BackupRecoveryPoint 必须同时绑定 PostgreSQL 备份及 WAL 截止点、该截止点上全部已被 DocumentObject 引用的 Kodo STAGED/FORMAL 原文件与预览对象清单（对象编号/键、状态、大小、校验值、对象信息标识及备份副本引用）、墓碑账本水位、索引/运维配置和密钥引用/受控托管材料；所有部分均持久化并通过摘要校验后才标记 READY，恢复只能选择 READY 恢复点；
- 七牛云原文件/预览备份副本和对象清单保存在公司控制的私有备份基础设施中；Elasticsearch 和 pgvector 是可重建派生索引，只在验证全量重建无法满足 RTO 时才将索引快照纳入 BackupRecoveryPoint；
- 恢复后验证账号、授权、API Key 状态、文档状态、活动索引版本和删除墓碑；
- 目标为 RPO ≤ 24 小时、RTO ≤ 8 小时；
- 至少每季度完成一次恢复演练并保留结果；
- 升级前执行兼容性检查和备份，失败时回滚到上一可用版本；
- 恢复顺序固定为：关闭业务流量，恢复墓碑账本和密钥访问能力，按同一 READY BackupRecoveryPoint 恢复 PostgreSQL 和对象备份副本，再从该恢复点的账本水位之后幂等重放墓碑并完成所有权威/派生存储的物理清理，然后重建或验证派生索引、验证权限与活动版本、执行抽样检索，最后开放流量；任一恢复点组成部分缺失或墓碑水位无法验证时保持离线；
- RTO 演练分别记录各阶段耗时，若全量重建派生索引无法在 8 小时内完成，必须保留满足恢复目标的索引备份或缩短重建路径；不得只声明目标而不验证容量条件。

### 11.5 数据保留

- 原文件、检索追踪和审计日志不按时间自动过期；
- 经授权的显式删除、法规删除和安全事件处置按删除流程执行；
- 删除后的追踪只保留标识、版本、得分、耗时和操作结果，不保留可恢复正文；
- 备份按既定生命周期自然过期，恢复时重放删除墓碑；
- 删除墓碑账本的保留期不得短于最长业务备份生命周期加恢复缓冲期；账本完整性校验失败时禁止恢复环境上线；
- 永久数据可在公司批准的存储层之间分层归档，但不改变业务编号、保留语义或 PostgreSQL 中的可定位清单；归档迁移必须记录审计并校验完整性和可恢复性；
- 归档读取和审计查询仍经业务服务执行当前权限校验，不能通过归档路径绕过鉴权；显式删除、法规删除和安全处置必须覆盖归档副本，尚未完成物理清理时由删除墓碑阻止恢复或读取正文。

### 11.6 前端兼容性

- React 管理前端支持 Chrome、Edge、Safari 最近两个稳定主版本；
- 关键管理流程以真实浏览器执行兼容性测试，不以组件测试或构建成功代替；
- 浏览器不支持时给出明确提示，不通过降低后端权限、安全或数据校验规则兼容旧环境。

## 12. 配置与发布设计

### 12.1 外置配置

以下配置不得写死在代码或镜像：

- 数据库、Elasticsearch、七牛云空间/区域/域名/暂存前缀/正式前缀、任务队列和缓存连接；
- Embedding 与 Rerank 服务地址、超时、并发、批次和计算设备；
- 单批文件数量，以及不高于需求硬上限的上传大小、解压大小、页数、工作表、单元格、附件等有效资源限制；
- API 限流、任务并发、重试、队列长度、熔断阈值、暂存孤儿最小年龄和在途请求回收安全窗口；
- TLS、代理、外网白名单和内部 DNS；
- 七牛云 AccessKey/SecretKey 的密钥引用、短期下载地址有效期和日志脱敏策略；
- Embedding 查询前缀规则属于 IndexVersion 指纹；tokenizer 名称与版本、Rerank 配置编号/模型版本/文件校验值、评分转换版本和父子块聚合策略属于 RetrievalTrace 与 EvaluationRun 的查询期版本记录。

模型名称、模型版本、文件校验值和向量维度属于只读部署基线，运行时不得静默更新。

模型文件通过受控外网下载后，必须核验来源、版本、许可证和文件校验值，再保存到公司私有环境并冻结部署基线；运行进程只能加载已核验的本地模型文件。

### 12.2 启动依赖

建议启动顺序：

1. PostgreSQL/pgvector、Elasticsearch 和 Redis；
2. 七牛云 Kodo 私有空间连通性与认证读探针、空间私有性和服务端加密验证，以及本地 Embedding 与 Rerank 服务；
3. 数据库迁移和初始化检查；
4. FastAPI API 与 MCP 适配进程；
5. Celery Worker 与含 Outbox 投递器的 Celery Beat/Scheduler；
6. 健康检查、最小模型推理和索引连通性验证；
7. 管理前端和调用方连通性验证。

“建议启动顺序”不代表具体容器编排产品选型。

启动顺序不能替代能力级 readiness。各进程对临时不可用依赖执行有限退避重试，相应能力保持 UNREADY，不得无限阻塞或在必需依赖未验证时接受该类业务流量。Kodo 空间私有性或服务端加密未通过时只阻断上传及依赖 Kodo 的对象操作；检索、管理和其他能力按 11.2 节矩阵独立判定。

### 12.3 升级与回滚兼容窗口

- 数据库迁移采用扩展—迁移—收缩策略；新旧应用并存期间先增加兼容字段/表，再回填并切换读取，确认旧版本退出后才删除旧结构；
- 对外 REST 主版本、MCP Schema、内部 OpenAPI 和 Python 集成包分别维护兼容矩阵；新增字段保持向后兼容，字段语义不得静默改变；
- Elasticsearch Mapping 或索引结构变化通过新 IndexVersion 滚动重建，不原地执行不可回滚变更；
- 应用回滚前检查数据库、任务消息、索引版本和配置是否仍被旧版本识别；不可逆迁移必须在发布前显式阻断自动回滚并提供恢复方案；
- 发布过程执行数据库迁移检查、配置校验、最小推理、索引连通性、权限抽样和检索冒烟，失败不得切入流量。

### 12.4 发布物

- 版本化应用镜像；
- Python 依赖锁文件和前端依赖锁文件；
- 镜像摘要与第三方依赖清单；
- 配置模板和初始化脚本；
- 外部检索 OpenAPI、内部管理 OpenAPI 和 MCP 工具 Schema；
- 版本化《支持格式矩阵》及对应黄金验收样例清单；
- Python 客户端与 LangChain/LangGraph 示例；
- 部署、升级、回滚、备份恢复和运维手册。

## 13. 设计决策与待详细设计项

### 13.1 已确定设计决策

| 编号 | 决策 |
| --- | --- |
| DD-001 | 核心业务采用 FastAPI 模块化单体，不按业务模块拆分微服务 |
| DD-002 | API、MCP、Celery Worker、含 Outbox 投递器的 Celery Beat/Scheduler 为独立运行角色，共享领域与应用代码 |
| DD-003 | PostgreSQL 为业务事实来源，Elasticsearch 与向量索引均为版本化派生数据 |
| DD-004 | PostgreSQL 活动索引版本是在线发布开关 |
| DD-005 | HTTP 与 MCP 从同一领域契约映射并共享检索服务 |
| DD-006 | 权限、版本和状态异常禁止降级 |
| DD-007 | 文档处理异步执行，在线检索与入库资源隔离 |
| DD-008 | 不引入答案生成、工作流、多租户和需求外数据源 |
| DD-009 | PostgreSQL 事务 Outbox + 至少一次投递 + Worker 租约实现可靠异步处理 |
| DD-010 | IndexVersion 是完整逻辑内容快照，通过成员关系复用未变化的不可变文本和 EmbeddingArtifact；发布必须同时校验 base_active_index_version、visibility_revision 和墓碑并串行切换 |
| DD-011 | IndexVersion 固定绑定 Embedding、Schema、分块和索引结构指纹；Rerank 与 tokenizer 版本记录在查询追踪和评测运行中；Rerank 升级通过评测后切换查询期配置，不重建索引 |
| DD-012 | 内容块停用先改权威状态，启用先同步双索引再改有效状态 |
| DD-013 | 删除墓碑同时进入 PostgreSQL 与独立追加写账本，恢复后先重放墓碑再开放流量 |
| DD-014 | 已删除引用只在已知 citation_id、主体/密钥未失效、未显式撤权且通过当前或未撤销历史授权时返回最小墓碑状态 |
| DD-015 | 管理前端采用 React 技术栈；后端采用 FastAPI；入库采用 LlamaIndex；Celery + Redis 承担异步任务与消息投递；Redis 同时提供隔离的缓存空间 |
| DD-016 | 文档原文件和受保护预览使用公司七牛云账号下的 Kodo 私有空间；解析文本、内容块、索引、查询和模型数据不写入七牛云 |
| DD-017 | Kodo 使用平台键的暂存到不可变正式对象生命周期；对象转正、删除和孤儿回收均可幂等复核 |
| DD-018 | 备份恢复只使用同时绑定 PostgreSQL、Kodo 对象清单/副本和墓碑水位的 READY BackupRecoveryPoint |

### 13.2 待后续详细设计确定

下列内容不影响系统边界和核心语义，留待详细设计或部署方案确定：

- 容器编排、日志、指标和追踪产品；
- 物理节点数量、CPU/GPU、内存和磁盘规格；
- 逻辑/物理表结构、约束和索引由《RAG 知识库平台数据库设计说明书》承载，并按本 V1.7 系统设计复核；数据库分区和物理索引参数可依性能基线调整；
- Elasticsearch 分片、副本、生命周期和物理索引命名；
- 模型推理服务框架与批处理实现；
- 技术组件精确版本、镜像摘要和锁文件；
- Outbox、任务租约、墓碑账本和权限缓存失效的数据结构与编码细节；
- Embedding 指纹、tokenizer 版本记录、快照内容摘要和审计完整性校验的具体编码格式。

这些细节不得改变本文定义的业务边界、状态语义、权限规则、接口契约和验收要求。任务投递可靠性、快照并发保护、Embedding 绑定、即时启停、墓碑恢复和权限失效属于已确定系统语义，不能以下沉到详细设计为由省略。

## 14. 需求追溯

### 14.1 功能需求追溯

| 需求编号 | 设计章节 | 关键设计落点 | 主要验证 |
| --- | --- | --- | --- |
| FR-AUTH-001、FR-AUTH-004 | 4.2.1、6.5、10 | 关闭自助注册、初始管理员、密码策略、账号状态、登录锁定、临时密码、可撤销会话和审计 | 初始化唯一性、登录失败锁定、改密/禁用后旧会话失效 |
| FR-AUTH-002、FR-AUTH-003、FR-AUTH-006 | 4.2.3、7.2、9.4 | 全有或全无授权、角色/权限矩阵、直接授权、缓存失效 | 跨库拒绝、平台角色不读取正文、授权变更生效 |
| FR-AUTH-005 | 4.2.2、7.2、10.1 | API 调用账号、ApiKeyScope、范围交集、轮换、共享限流 | 新旧密钥并存、禁用立即失效、HTTP/MCP 不绕限流 |
| FR-KB-001、FR-KB-002 | 4.3、4.5、6.1 | 创建权限、固定模型、活动快照与 Embedding 指纹 | 空库无结果、模型变更触发重建 |
| FR-KB-003 | 5.5、7.2、11.4 | 二次确认、DeletionTask、墓碑账本、异步清理和恢复重放 | 含 UPLOADED 版本的知识库仍立即不可见，停止入库写入后按合法状态迁移清理，旧备份不复活 |
| FR-DOC-001 | 4.4.1、4.4.2、9.4、12.1、12.4 | 平台对象键、暂存/正式对象、固定格式、资源上限、格式矩阵和批次单文件隔离 | 逐格式黄金样例、超限错误、批量部分失败、暂存孤儿回收 |
| FR-DOC-002 | 4.3、4.7、7.2 | Schema 版本、继承/覆盖、权限字段服务端生成 | K12 模板可选、字段迁移和双后端过滤 |
| FR-DOC-003、FR-DOC-004 | 4.3、4.5、5.2、5.5、6.2 | 单发布版本、disable_reason、快照 CAS、停用和删除；UPLOADED→DELETING→DELETED | V2 构建时 V1 在线、并发发布、显式回滚；UPLOADED 删除停止任务写入并清理暂存文件 |
| FR-INGEST-001、FR-INGEST-004 | 4.4、5.1、6.4、6.5 | Outbox、租约、执行记录、取消、重试和清理 | 队列故障补投、Worker 崩溃接管、取消无半成品 |
| FR-INGEST-002、FR-INGEST-003 | 4.4.2、7.1 | 中间产物、格式定位、OCR 质量、清洗版本和告警 | 黄金样例、低置信度审核、清洗前后追溯 |
| FR-INGEST-005 | 3.3、4.4.2、4.5 | LlamaIndex 适配层、平台对象映射、禁止绕过发布 | 本地流水线与生产索引隔离 |
| FR-CHUNK-001、FR-CHUNK-002、FR-CHUNK-003 | 4.3、4.4.3、4.6、7.2 | 五种分块策略、参数与边界保护、父子关系、快照成员和定位 | 策略黄金样例、Unicode/Markdown/表格边界、父子引用、配置变更重建 |
| FR-CHUNK-004 | 5.3、4.7、6.2 | 草稿复制、变化块重算、非对称即时启停 | 编辑失败不影响线上、停用/启用时序 |
| FR-INDEX-001 | 4.5.1、4.6、12.1 | 模型指纹绑定、查询按活动指纹执行 | 模型升级期间新旧索引空间不混用 |
| FR-INDEX-002、FR-INDEX-003 | 4.5、5.2、11.3 | 完整快照、双索引校验、CAS 切换、巡检和清理 | 单侧失败、过期快照拒绝、孤儿检测 |
| FR-SEARCH-001、FR-SEARCH-002、FR-SEARCH-003、FR-SEARCH-004 | 4.6、4.7、8.1、8.3 | 三种模式、多库分组、RRF 列表定义、稳定去重 | KEYWORD/VECTOR/HYBRID、跨库顺序稳定 |
| FR-SEARCH-005 | 4.6、8.3 | 本地 Rerank、logit、Sigmoid、阈值和降级 | 开关、阈值边界、超时保留原排序 |
| FR-SEARCH-006 | 4.7、7.2 | 受控 AST、强制过滤、Schema 类型和有效期 | 双后端同条件、跨库字段不兼容 |
| FR-SEARCH-007 | 4.6、8.2 | tokenizer 版本、父块聚合、top_n 和完整块预算 | 父块去重、Token 边界和 1～50 |
| FR-SEARCH-008、FR-SEARCH-009 | 4.6、8.1、8.2、9.5 | 空结果、降级、请求/生效参数和稳定错误 | 单路/双路故障、越界参数、空结果原因 |
| FR-CITE-001、FR-CITE-002 | 4.8、7.2、9.1、9.5 | 稳定 Citation、当前/历史/删除状态与再鉴权 | 历史无正文、删除最小墓碑、撤权后 403 |
| FR-API-001、FR-API-002、FR-API-003、FR-API-004、FR-API-005 | 4.1.1、4.6、8、9.1、9.5 | 统一检索服务、响应语义、取消、错误和追踪 | 四个 REST 能力、只读重试、真实调用 |
| FR-API-006 | 4.1.2、9.4 | 私有管理域、角色/权限、幂等、状态和独立 OpenAPI | 前端契约测试、越权和重复提交 |
| FR-MCP-001、FR-MCP-002、FR-MCP-003 | 4.1.3、9.2、9.5、10.1 | 无状态协议适配、共享领域模型、限流和追踪 | Inspector/客户端兼容、HTTP/MCP 一致 |
| FR-FRAMEWORK-001、FR-FRAMEWORK-002、FR-FRAMEWORK-003 | 9.3、12.3、12.4 | 同步/异步客户端、Retriever、LangGraph 示例和兼容矩阵 | 最低/最高版本与 HTTP/MCP 真实调用 |
| FR-EVAL-001、FR-EVAL-002、FR-EVAL-003 | 4.9、6.4、7.2 | 活动/READY 快照单条调试、CSV、不可变运行快照、双参数集对比、验收库与门禁 | 不少于 100 问题、批准脱敏语料、结果复现、失败阻止首次上线 |
| FR-ADMIN-001、FR-ADMIN-002、FR-ADMIN-003、运营管理界面 | 3.3、4.1.4、4.6、4.10、9.4、11、12.1 | 模型运行参数、计算设备、运行/诊断模块、React 管理前端、管理 API 和最小正文暴露；Embedding 升级全量重建，Rerank 升级经评测后切换查询期配置 | 管理页面契约、浏览器兼容、追踪查询、健康和告警；Rerank 升级验证新旧版本切换、在途保持和失败回退 |

### 14.2 非功能与验收追溯

| 需求范围 | 设计章节 | 证据要求 |
| --- | --- | --- |
| NFR-PERF-001、NFR-PERF-002、NFR-PERF-003、NFR-PERF-004、NFR-PERF-005、NFR-PERF-006 | 11.1、11.3 | 固定数据、参数、模型、硬件和并发的 HTTP/MCP 性能报告 |
| 安全与隐私 | 4.2、4.7、10、11.4 | 权限攻击用例、密钥/日志扫描、解析隔离、加密和恢复证据 |
| 可用性与可靠性 | 4.4、4.5、6、11.2 | 队列/Worker/模型/搜索故障注入和恢复记录 |
| 容量、保留与私有化 | 11.1、11.4、11.5、12 | 容量基线、墓碑恢复、RPO/RTO、网络边界和依赖清单 |
| AC-001、AC-006、AC-009 | 4.3～4.5、5、6 | 入库发布、更新、并发和故障恢复端到端证据 |
| AC-007 | 5.5、6.2 | 对全部合法状态路径验证立即不可见、完整清理、重试和无孤儿，覆盖 UPLOADED 版本 |
| AC-002、AC-003、AC-004、AC-005 | 4.6～4.8、8、9 | 真实 API、权限、三种模式、过滤和引用证据 |
| AC-008 | 4.9 | 冻结评测运行及门禁记录 |
| AC-010、AC-011 | 4.1.3、9.2、9.3 | MCP、LangChain、LangGraph 兼容和真实调用证据 |
| AC-012 | 10～12 | 核心服务全新部署、七牛云 Kodo 私有存储接入、升级回滚、备份恢复和网络边界证据 |

### 14.3 需求确认单追溯

| 确认项 | 设计章节 | 一致性结论 |
| --- | --- | --- |
| C-AUTH-01、C-AUTH-02、C-AUTH-03、C-AUTH-04、C-AUTH-05、C-AUTH-06、C-AUTH-07、C-AUTH-08、C-AUTH-09 | 4.1、4.2、6.5、9.4、10 | 独立账号、固定角色、直接授权、API Key 范围交集和权限失效方式一致 |
| C-DOC-01 | 4.3、4.9 | 生产不预置首批业务知识库，由业务团队上线后自行创建；验收知识库与生产隔离 |
| C-DOC-02、C-DOC-03、C-DOC-04、C-DOC-05、C-DOC-06、C-DOC-08 | 4.3～4.5、5.2、5.3、6 | UUIDv7、文档版本、双索引发布、内容块编辑/启停和历史引用语义一致 |
| C-DOC-07 | 5.5、6.2 | 停用、删除和清理语义一致；UPLOADED 删除须先阻止入库写入并完成原文件清理 |
| C-META-01、C-META-02、C-META-03、C-META-04、C-META-05、C-META-06 | 4.3、4.7、7 | 单租户固定字段、K12 可选模板、Schema 版本和继承规则一致 |
| C-INGEST-01、C-INGEST-02、C-INGEST-03 | 2.3、4.4、5.1、6.4 | Celery 异步执行不改变任务语义；LlamaIndex 仅用于本地入库适配，不调用托管解析服务 |
| C-SEARCH-01、C-SEARCH-02、C-SEARCH-03、C-SEARCH-04、C-SEARCH-05、C-SEARCH-06、C-SEARCH-07、C-SEARCH-08 | 4.6～4.8、8、9 | 三种检索模式、双检索后端、本地模型、降级、空结果、引用和私有管理边界一致 |
| C-SEARCH-09 | 4.5、4.6、4.9、12.1 | Embedding 升级重建索引；Rerank 升级固定评测输入、校验门槛后切换查询期配置，在途请求保留旧版本 |
| C-MCP-01、C-FRAMEWORK-01 | 4.1.3、9.2、9.3 | MCP 无状态协议适配和 LangChain/LangGraph 调用端边界一致 |
| C-NFR-01、C-NFR-02、C-NFR-03、C-NFR-04、C-NFR-05、C-NFR-06、C-NFR-07、C-NFR-08 | 10～12 | 性能基线、可靠性、TLS/加密、查询指纹、保留、墓碑恢复和浏览器兼容一致 |

## 15. 设计验收检查

系统设计评审至少确认：

- 需求确认单的评审稿 SHA-256 与当前需求说明书一致；正式签署时仍须按确认单流程填写最终签署校验值；
- UPLOADED DocumentVersion 的删除迁移已明确，验收须覆盖单文档和知识库删除时停止入库写入、文件清理及失败重试；
- Rerank 升级按冻结评测集验证后切换查询期版本，验收须覆盖评测门槛、在途请求版本保持及切换失败回退；
- 架构未超出需求确认的 MVP 边界；
- HTTP、MCP 和框架适配共享同一检索核心；
- PostgreSQL、pgvector、Elasticsearch 和对象存储职责明确；
- 七牛云 Kodo 使用公司账号私有空间、HTTPS、服务端加密和短期下载地址；暂存转正、重试、孤儿回收和删除均幂等可验证，前端不持有存储密钥；
- 权限、文档版本和索引版本不会被缓存或降级绕过；
- 新版本发布校验基础活动版本，并发冲突或切换失败时旧版本继续可用；
- Embedding、Schema、分块和索引结构指纹随活动快照冻结，Rerank 与 tokenizer 版本随查询和评测记录；
- 停用和删除能够立即阻止新访问；
- 内容块启用只有在双索引同步成功后才对外显示成功；
- 历史引用和已删除引用不会返回正文；
- 入库任务可追踪、可恢复且不产生重复可见数据；
- 数据库提交而队列不可用时可由 Outbox 补投，取消和租约超时具有明确终态；
- 租约代次能阻止过期 Worker 提交，READY 发布会复核基础活动版本、可见性代次和墓碑；
- 三种检索模式、参数、得分、过滤和降级语义与需求一致；
- 审计、追踪、加密、日志脱敏和查询指纹满足安全要求；
- BackupRecoveryPoint 同时绑定 PostgreSQL、Kodo 对象副本/清单和墓碑水位，备份恢复、删除墓碑和 RPO/RTO 有可验证设计落点；
- 从早于删除时间的备份恢复后，墓碑账本仍能阻止已删除正文重新上线；
- 除第 2.3 节已确定的项目技术栈外，未指定的基础设施产品和容量数值没有被擅自固化；
- 需求追溯表逐项给出设计落点和主要验证，不以章节范围代替实际覆盖。

## 16. 关联文档

- [RAG 知识库平台需求说明书 V1.25](./01-RAG知识库平台需求说明书.md)
- [RAG 知识库平台需求确认单 V1.4](./02-RAG知识库平台需求确认单.md)
- [RAG 知识库平台数据库设计说明书 V1.3](./04-RAG知识库平台数据库设计说明书.md)
