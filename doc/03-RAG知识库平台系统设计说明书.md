# RAG 知识库平台系统设计说明书

> 项目名称：RAG Base  
> 文档版本：V1.3  
> 文档状态：设计评审稿  
> 编制日期：2026-09-19  
> 修订日期：2026-09-22  
> 需求基线：《RAG 知识库平台需求说明书》V1.23  
> 确认基线：《RAG 知识库平台需求确认单》V1.2

> V1.3 修订：文档原文件和受保护预览改为存储到公司七牛云账号下的 Kodo 私有空间；同步调整上传链路、访问控制、安全、部署、备份恢复和需求追溯。
>
> V1.2 修订：按需求说明书 V1.22 和需求确认单 V1.1 复核系统边界；补充 React、FastAPI、LlamaIndex、Celery、Redis 等项目技术栈；收敛实现级细节，并修正异步任务、索引模型绑定、管理接口和浏览器兼容性表述。
>
> V1.1 修订：补充可靠任务投递与任务状态机、知识库快照并发发布、索引与模型版本绑定、内容块即时启停、删除墓碑恢复、账号会话与权限失效、解析产物追溯、引用删除语义、评测门禁、安全运维和逐项需求追溯设计。

## 1. 文档说明

### 1.1 编制目的

本文定义 RAG Base MVP 的系统架构、模块边界、关键流程、状态与一致性机制、数据和接口边界、安全设计、部署设计及非功能设计，用于指导后续详细设计、开发、测试和部署。

### 1.2 基线状态

当前需求说明书状态为“评审稿”，需求确认单状态为“待确认”，因此本文属于设计评审稿。需求基线正式签署后，如确认内容发生变化，应按需求变更流程同步评估和修订本文。

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
| 关键词检索 | Elasticsearch，中文索引使用 ik_max_word，查询使用 ik_smart |
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
| 异步任务 | Celery Worker、Celery Beat | 执行入库、索引、清理、评测和周期巡检；业务任务状态仍以 PostgreSQL 为准 |
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
- Celery Beat 触发周期任务，Scheduler 执行失败任务恢复、一致性巡检、容量检查和定时维护；
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
        PublicIngress[检索入口]
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
        Dispatcher[可靠投递器]
        Queue[Redis Broker]
        Worker[Celery Worker]
        Scheduler[Celery Beat / Scheduler]
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
    HttpClient --> PublicIngress
    LC --> PublicIngress
    Inspector --> PublicIngress --> MCP
    PublicIngress --> RetrievalAPI

    AdminAPI --> IAM
    AdminAPI --> KB
    AdminAPI --> Eval
    AdminAPI --> Admin
    RetrievalAPI --> Retrieve
    MCP --> Retrieve
    Retrieve --> IAM
    Retrieve --> Citation
    KB --> Ingest --> Outbox --> Dispatcher --> Queue
    Eval --> Outbox
    Queue --> Worker
    Scheduler --> Queue
    Worker --> Ingest
    Worker --> Index

    IAM --> PG
    KB --> PG
    KB --> Tombstone
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
            S[Celery Beat / Scheduler]
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
    W --> MQ
    W --> PG
    W --> ES
    W -->|HTTPS / 官方 SDK| Qiniu
    W --> E
    S --> MQ
    S --> PG
    S --> ES
    S -->|HTTPS / 官方 SDK| Qiniu
    API --> DL
    W --> DL
    API --> O
    MCPD --> O
    W --> O
    S --> O
    PG --> B
    ES --> B
    Qiniu -.对象备份/恢复验证.-> B
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
- 校验 Bearer API Key、Mcp-Protocol-Version、Mcp-Method、Mcp-Name 和请求 _meta；
- 不执行 initialize/initialized，不维护会话 ID，不实现 OAuth；
- 提供 search_knowledge、list_knowledge_bases、get_chunk；
- 仅做协议转换，不自行实现权限、检索、排序或引用逻辑；
- 与 REST API 使用同一错误枚举、限流计数、追踪规范和结构化领域结果。

#### 4.1.4 管理前端

- 使用 React + TypeScript 构建单页管理应用，Vite 负责构建，Ant Design 提供基础管理组件；
- React Router 管理页面路由，TanStack Query 管理服务端数据、缓存失效和请求状态；
- 页面范围严格对应需求中的知识库、文档、内容块、账号、密钥、评测、追踪和平台运行管理，不增加聊天、答案生成或工作流页面；
- 前端仅调用私有管理 API，不保存业务事实，不直连 PostgreSQL、Elasticsearch、Redis 或七牛云 Kodo，也不持有七牛云 AccessKey/SecretKey；
- 登录态、页面按钮和路由控制只改善交互，后端仍必须执行完整认证、授权、状态和审计校验。

### 4.2 身份与权限模块

#### 4.2.1 人工账号

- 邮箱经去除首尾空格并统一大小写后作为唯一登录名；
- 密码使用自适应哈希与独立随机盐；
- 人工账号状态为 ACTIVE、LOCKED、DISABLED；登录失败计数、临时锁定截止时间和解锁操作均保存于 PostgreSQL 并写入审计；
- 管理员密码重置签发一次性临时凭证，首次登录必须修改密码；临时凭证不得作为长期会话使用；
- 登录成功创建可撤销的会话记录，访问凭证携带会话编号和短有效期；退出、密码修改、账号禁用和管理员强制下线通过会话版本或撤销时间使已有凭证立即失效；
- 管理前端采用安全 Cookie 时启用 HttpOnly、Secure、SameSite 和 CSRF 防护；采用 Authorization Header 时不得把凭证保存到可被脚本长期读取的不安全存储；
- 全局角色为 PLATFORM_ADMIN、ACCOUNT_ADMIN、USER；
- 全局角色不隐含知识正文读取权限。

#### 4.2.2 API 调用账号与 API Key

- API 调用账号为非登录主体，只用于 HTTP 和 MCP；
- API Key 完整值只在创建时返回一次，服务端仅保存不可逆摘要或安全等价物；
- API Key 使用不可预测随机值，服务端通过不可逆摘要完成校验；日志只记录密钥编号和脱敏标识；
- API Key 不自动过期，禁用、删除或所属 API 调用账号禁用后立即失效；
- 实际知识库范围取“API Key 自身范围”与“所属 API 调用账号当前授权范围”的交集；API Key 范围使用独立关系保存，不以内嵌字符串代替；
- 轮换时允许新旧密钥短期并存，由调用方切换后显式禁用旧密钥；平台不自动延长或缩短永久有效规则。

#### 4.2.3 知识库权限

- 权限仅为 MANAGE、EDIT、RETRIEVE、EVALUATE；
- 人工账号和 API 调用账号与知识库直接授权，不存在用户组和组织继承；
- 多知识库请求必须先完成全量授权校验，任一目标不存在或无权访问时整次返回 403 FORBIDDEN_SCOPE；
- 每项管理操作必须同时满足平台角色和知识库权限矩阵；MANAGE、EDIT、RETRIEVE、EVALUATE 互不隐含，EVALUATE 只允许读取评测执行所需的受控结果，不自动授予通用正文浏览能力；
- 权限缓存只作为加速手段，设置不超过 5 分钟的有效期并在授权变化时通过事务 Outbox 主动失效；缓存失效事件失败时，新请求回源 PostgreSQL，不能继续信任超过授权版本的缓存；
- API 调用账号、API Key 禁用和会话撤销不得等待缓存自然过期，鉴权记录携带状态版本并在请求入口执行权威复核；
- 读取内容块和引用时重新执行当前权限检查；
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

- KnowledgeBase 保存知识库状态、分块配置、元数据 Schema 和活动索引版本引用；
- Document 作为稳定业务身份，指向当前发布版本；文档是否可检索仍由当前 DocumentVersion 状态承载；
- DocumentVersion 保存文件版本、来源版本、变更原因、处理/发布状态和 disable_reason；DISABLED 必须区分 MANUAL 和 SUPERSEDED；
- Chunk 保存内容、位置、标题路径、父子关系、元数据、版本和启用状态；
- MetadataSchemaVersion 独立版本化，记录字段定义、过滤能力、块级覆盖能力和状态；已索引字段不允许原地修改类型；
- 文档元数据默认继承到内容块，只有 chunk_overridable 字段可以覆盖；
- K12 模板为可选业务模板，权限字段由服务端生成，调用方不能提交或覆盖。

文档停用把当前 PUBLISHED 版本改为 DISABLED，并记录 disable_reason=MANUAL；新版本发布把旧版本改为 DISABLED，并记录 disable_reason=SUPERSEDED。DISABLED 版本重新进入 PUBLISHED 仅允许两类路径：恢复当前人工停用版本，或通过显式回滚流程激活双索引仍完整的历史版本；两者均需重新校验活动快照并写审计。

### 4.4 入库编排模块

#### 4.4.1 任务模型

管理 API 接收上传流后，使用七牛云官方 Python SDK 通过 HTTPS 把文件写入 Kodo 私有空间的暂存前缀，管理前端不直接访问 Kodo。随后在同一 PostgreSQL 事务中创建 DocumentVersion、IngestionTask 和 OutboxEvent；事务提交后由投递器把 task_id 发送到 Redis Broker，Celery Worker 首次确认任务时再把对象转为正式对象。暂存对象只允许在“超过安全保留窗口且 PostgreSQL 中不存在有效引用”时由孤儿扫描删除，队列长时间不可用不得导致仍被任务引用的原文件提前过期。

任务至少记录任务编号、文档版本、参数版本、当前状态、当前阶段、进度、执行次数、租约持有者、心跳时间、下次重试时间、开始/结束时间和稳定错误码。每次实际运行独立记录 TaskExecution，不覆盖历史失败执行。

阶段固定为：

1. 文件与资源上限校验；
2. 安全解析；
3. 文本清洗；
4. 分块；
5. Embedding；
6. pgvector 写入；
7. Elasticsearch 写入；
8. 双索引校验；
9. 转入 REVIEW_REQUIRED。

Celery 任务采用至少一次投递语义：消息只携带 task_id，Worker 每次执行前在 PostgreSQL 通过条件更新取得租约；重复消息、Worker 重启或投递器重试不得产生第二份可见结果。OutboxEvent 在 Broker 投递调用成功后标记已投递；Celery Beat 周期扫描未投递事件、超过派发时限仍为 PENDING 的任务和过期租约，即使 Redis 消息丢失也能从 PostgreSQL 业务状态重新投递。

#### 4.4.2 解析适配

- 每类文件使用独立解析适配器，统一输出平台中间文档结构；
- LlamaIndex 只位于内部适配层，用于本地读取、转换、分块与 Ingestion Pipeline 编排；
- LlamaIndex 对象必须转换为平台 DocumentVersion、Chunk 和元数据对象；
- 解析产物包含原始提取文本、清洗后文本、结构位置、解析器版本、清洗规则版本、OCR 模型版本、OCR 置信度和告警；产物保存在 PostgreSQL 或公司私有文件存储，不写入七牛云，并由 DocumentVersion 和产物校验值稳定关联；
- PDF、PPTX、XLSX、EML 和图片保留需求规定的页码、幻灯片、工作表、邮件头或图片编号；PPTX 保留演讲者备注，隐藏工作表和不支持附件形成结构化告警；
- EML 中受支持附件创建关联子 Document 和独立 IngestionTask，父任务记录附件任务编号，但附件失败不伪装为父文档解析成功；
- HTML 禁止加载外部资源，XLSX 只读取已保存值且不执行公式，解析环境禁止宏、脚本和外部命令；
- OCR 低于配置阈值时，DocumentVersion 进入 REVIEW_REQUIRED 并携带低质量标记，不能通过自动流程静默发布；
- 大文件采用流式读取或分段处理，不一次性载入应用进程内存。

#### 4.4.3 失败、取消和重试

- 确定性错误不自动重试；
- 临时依赖错误按配置执行有限次数退避重试；
- 人工重试创建新的执行记录，保留原失败记录；
- Worker 使用任务租约或等价机制识别异常中断，服务恢复后可重新调度；
- 取消请求将任务改为 CANCEL_REQUESTED；Worker 在阶段边界停止并清理本次未发布派生数据后改为 CANCELLED，清理失败进入 CLEANUP_PENDING 并告警；
- 同一文档版本同一处理阶段通过幂等键和并发约束防止重复写入。

### 4.5 索引与发布模块

#### 4.5.1 索引组织

- Elasticsearch 使用知识库和索引版本可定位的物理索引；
- pgvector 每条向量携带 knowledge_base_id、document_id、document_version、index_version 和固定权限状态字段；
- 两侧均写入业务过滤字段和服务端权限过滤字段；
- 每个 IndexVersion 表示知识库在一个发布时点的完整可检索快照，新版本必须包含该知识库全部应发布内容，不能只包含本次变化文档；
- IndexVersion 必须记录 base_active_index_version、快照内容清单摘要、元数据 Schema 版本、分块配置版本、Embedding 模型指纹和索引结构版本；Rerank 与 tokenizer 属于查询期配置，记录在 RetrievalTrace 和 EvaluationRun，不触发无关的索引重建；
- Embedding 模型指纹至少由模型名称、模型版本、模型文件校验值、向量维度、归一化规则和查询前缀规则组成；查询向量必须按活动 IndexVersion 的指纹生成；
- 未变化内容块可以复用已校验的文本和向量，但新快照必须通过显式成员关系引用这些不可变产物，不能依赖“当前状态”动态拼出快照；
- IndexVersion 管理 BUILDING、READY、ACTIVE、RETIRED、FAILED、DELETED 状态；
- 每个知识库只有一个 active_index_version。

#### 4.5.2 发布策略

1. 读取并记录当前 active_index_version，创建 BUILDING 索引版本和快照内容清单；
2. 写入 pgvector 新版本数据和 Elasticsearch 新物理索引；
3. 校验数量、标识、向量维度、权限字段、业务过滤字段和可查询性；
4. 校验成功后将索引标记为 READY；
5. 发布前复核 READY 版本的校验证据；双索引任一侧不可用或证据不完整时不得进入切换事务；
6. 对知识库取得数据库级互斥锁，并校验当前 active_index_version 仍等于 base_active_index_version；不相等表示快照过期，返回 409 并要求基于新活动版本重建，禁止覆盖其他已发布内容；
7. 在短 PostgreSQL 事务中完成新文档版本 PUBLISHED、旧版本 DISABLED、活动索引版本切换和旧索引 RETIRED；活动版本更新采用条件更新并受唯一约束保护，事务内不执行 Elasticsearch 等远程网络调用；
8. 新检索请求读取新的活动版本；已开始请求继续使用请求开始时捕获的版本；
9. 旧索引进入 RETIRED 后按回滚保留策略异步清理，清理前仍不得被新请求使用。

活动版本是在线检索的唯一发布开关。Elasticsearch 别名可以用于运维定位，但不能取代 PostgreSQL 中的活动版本事实。

同一知识库同一时刻只允许一个发布动作进入切换临界区，构建任务可以并行，但过期快照不得发布。完整快照方案保持业务语义简单，但可能带来较高存储和构建成本；详细设计必须定义不可变产物复用、增量拷贝、垃圾回收和容量基线，不能每次人工编辑无条件重复计算全部向量。

#### 4.5.3 一致性巡检

巡检任务按知识库和索引版本核对：

- PostgreSQL 内容块与 pgvector 向量是否缺失、重复或维度错误；
- PostgreSQL 内容块与 Elasticsearch 文档是否缺失、重复或版本错误；
- 活动版本是否在两侧完整可用；
- IndexVersion 快照成员、基础活动版本和模型指纹是否一致；
- 已删除对象是否存在孤儿数据；
- 发现问题后记录告警，禁止自动把未校验版本切为活动版本。

### 4.6 统一检索模块

统一检索服务按固定流水线执行：

~~~mermaid
flowchart LR
    A[参数校验] --> B[API Key 鉴权]
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
    M --> O[结果裁剪]
    N --> O
    O --> P[引用与响应映射]
    P --> Q[写入脱敏追踪]
~~~

关键规则：

- 请求开始时捕获统一 UTC 时间和各知识库活动索引版本，整个请求保持不变；
- 同时捕获各活动索引的 Embedding 模型指纹和元数据 Schema 版本，并记录本次使用的 Rerank 与 tokenizer 版本；多个知识库 Embedding 指纹不一致时按指纹分组生成查询向量并分别召回，再按统一 RRF 规则融合，无法加载所需模型时明确拒绝或按模式降级；
- 固定权限条件由服务端注入，业务过滤不能覆盖；
- 多知识库字段在执行前检查存在性和类型兼容性；
- KEYWORD 和 VECTOR 模式不执行 RRF；HYBRID 模式把“每个知识库的关键词列表”和“每个知识库的向量列表”视为独立排序列表，对全部有效列表执行 RRF，默认 rrf_k=60；
- 同一内容块只保留一条，并记录各路得分和排名；
- 并列结果以稳定 chunk_id 确定顺序，知识库传入顺序不影响结果；
- Elasticsearch 和 pgvector 返回候选标识、得分及必要定位信息，统一检索服务在最终返回前从 PostgreSQL 获取权威内容与状态并完成复核；
- Rerank 只处理候选集前 50 条；
- Rerank 原始输出保存为 rerank_logit，使用固定 Sigmoid 转换得到 rerank_score；rerank_threshold 只作用于转换后的 rerank_score；
- Token 数使用平台冻结的 tokenizer 名称、版本和配置计算，并在响应生效参数中返回 tokenizer_version；超出预算时按完整返回块裁剪；
- 父子分块命中多个子块但返回同一父块时合并为一个结果，父块得分取最佳命中子块的最终得分，保留全部命中子块编号和位置；top_n 和 Token 预算均按去重后的父块计算；
- 关键词高亮信息作为可选结果字段保留；返回元数据只允许 Schema 中标记为 response_visible 的业务字段，权限字段和内部状态字段不得透出；
- 空结果返回空数组和明确原因，不生成补充内容。

### 4.7 过滤设计

过滤分为两层：

1. **服务端强制过滤**：tenant_id、已授权 knowledge_base_id、当前 document_version、当前 index_version、document_status=PUBLISHED、chunk_enabled=true；指定文档时再校验 document_id。
2. **业务元数据过滤**：仅允许知识库 Schema 中声明且可过滤的字段，支持等于、不等于、包含、范围、IN、AND 和 OR。

过滤处理流程：

- 将 metadata_filter 解析为受控抽象语法树；
- 校验字段、类型、操作符、嵌套深度、条件数量和值长度；
- 将同一抽象语法树分别编译为 Elasticsearch DSL 和 PostgreSQL 条件；
- 禁止用户文本直接拼接 SQL 或 Elasticsearch DSL；
- 查询结束前对候选结果执行 PostgreSQL 权威状态复核，覆盖 Elasticsearch 状态更新的短暂延迟窗口。

有效期使用请求开始时的 UTC 时间，并应用：

valid_from 为空或 valid_from 小于等于当前时间；且 valid_to 为空或 valid_to 大于当前时间。

内容块启停采用非对称一致性流程：

- 停用时先在 PostgreSQL 将权威 `chunk_enabled` 改为 false 并写入 Outbox；即使索引尚未同步，末端权威复核也立即阻止返回；
- 启用时先保持 PostgreSQL `chunk_enabled=false`，同步更新 pgvector 过滤字段和 Elasticsearch 文档并等待可查询确认，最后才将 PostgreSQL `chunk_enabled` 改为 true；任一后端更新失败时保持停用并返回可重试错误；
- 两侧查询应用 `chunk_enabled=true`，末端再复核 PostgreSQL 权威状态；只有启用操作成功返回后才承诺新请求可检索，失败不得出现“界面已启用但实际不可召回”；
- Scheduler 重试未完成的状态同步并对长期不一致告警。

### 4.8 引用模块

- citation_id 稳定指向具体文档版本和内容块版本；
- 当前发布版本在再次鉴权后可以返回引用片段和受保护预览；
- 历史版本只返回文档名称、原版本号和“历史版本不可查看”，不返回正文或预览；
- 引用记录独立保存不可猜测 citation_id、知识库、文档和版本标识，但不复制正文；
- 来源删除后，只有调用方提交已知 citation_id，且其当前授权或删除时保留的历史授权关系允许识别该来源时，引用接口才返回“来源已删除”和最小标识；检索、块详情、列表或任意编号探测仍统一返回 403；权限已撤销时引用接口也返回 403；
- 预览地址必须短期有效并在生成前检查权限，不提供永久公开 URL。

### 4.9 检索测试与评测模块

- 单条检索测试直接调用统一检索服务，并额外展示各阶段候选、得分、过滤和耗时；
- 测试参数只对本次测试生效，不修改知识库在线配置；
- EvaluationSet、EvaluationCase 和 EvaluationRun 均版本化；
- 批量评测通过异步任务执行，记录语料版本、文档版本、索引版本、模型版本和实际参数；
- 计算 Recall@K、Precision@K、MRR@K、nDCG 和无结果率；
- EvaluationRun 启动时冻结评测集版本、活动索引版本、参数集、模型指纹、相关性标注版本和代码版本，生成不可变 run_fingerprint；运行期间不跟随线上索引切换；
- 父子块评测以 EvaluationCase 声明的期望文档/块粒度计算；返回父块时，只有其命中子块或父块标识满足该用例的期望集合才算命中，禁止在运行后改变计分口径；
- EVALUATE 权限允许查看该知识库的评测用例和受控命中片段，但不自动开放通用正文浏览；评测导出和查看敏感片段均审计；
- 评测 Worker 使用独立队列或并发配额，不能挤占在线查询和入库资源；
- 默认混合检索参数集未达到 Recall@10 ≥ 0.80、MRR@10 ≥ 0.60 或权限用例未 100% 通过时，生成失败门禁记录并阻止平台首次生产发布；门禁记录包含 run_fingerprint、指标、审批人和证据位置。

### 4.10 审计与检索追踪模块

- AuditLog 记录登录、密码重置、账号禁用、API Key、授权、状态变化、发布、删除和人工内容修改；
- RetrievalTrace 记录 request_id、trace_id、协议、调用账号、知识库、参数、命中标识、得分、阶段耗时、降级信息和结果状态；
- 原始查询正文不进入检索追踪，改用独立可轮换密钥计算 HMAC-SHA256 指纹，并记录字符数、hmac_key_id 和规范化规则版本；密钥轮换后新记录使用新 key_id，历史记录不重新计算；
- 日志不记录密码、API Key、完整向量、永久下载地址或大段正文；
- 审计日志采用追加写或等价防篡改方式；降级、权限缓存失效失败、墓碑重放和管理端敏感正文查看均写入审计或安全事件日志。

## 5. 关键业务流程

### 5.1 文档入库流程

~~~mermaid
sequenceDiagram
    actor Operator as 运营人员
    participant API as 管理 API
    participant PG as PostgreSQL
    participant OS as 七牛云 Kodo
    participant Q as 任务队列
    participant D as Outbox 投递器
    participant W as Worker
    participant EMB as Embedding
    participant ES as Elasticsearch

    Operator->>API: 上传文件与元数据
    API->>API: 权限、格式、大小和文件特征校验
    API->>OS: HTTPS 保存私有暂存对象
    API->>PG: 同一事务创建 DocumentVersion、IngestionTask、OutboxEvent
    API-->>Operator: 返回任务编号
    D->>PG: 领取未投递 OutboxEvent
    D->>Q: 提交 task_id
    Q-->>D: 确认接收
    D->>PG: 标记事件已投递
    Q->>W: 分派任务
    W->>PG: 条件取得任务租约并创建 TaskExecution
    W->>OS: 校验对象并登记正式引用
    W->>PG: 状态改为 RUNNING / PROCESSING
    W->>W: 安全解析、清洗、分块
    W->>EMB: 批量生成向量
    W->>PG: 写入草稿内容块与版本化向量
    W->>ES: 写入版本化倒排索引
    W->>W: 双索引校验
    alt 校验成功
        W->>PG: 状态改为 REVIEW_REQUIRED
    else 失败
        W->>PG: 状态改为 FAILED，记录稳定错误码
    end
~~~

### 5.2 文档发布与更新流程

发布前要求文档版本处于 REVIEW_REQUIRED，且对应双索引均完整可用。发布事务同时完成：

- 对知识库取得发布互斥锁，并校验 READY 索引的 base_active_index_version 与当前活动版本相同；
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

内容块启用/停用不创建文档内容版本。停用先更新 PostgreSQL 权威状态，依靠末端复核立即阻断，再异步同步索引；启用先同步双索引并确认可查询，最后更新 PostgreSQL 有效状态。任一同步失败时启用操作不得显示成功。

### 5.4 在线检索流程

1. 接入层完成协议和请求上限校验；
2. 鉴别 API Key，并验证所属 API 调用账号状态；
3. 计算 API Key 与调用账号知识库范围交集；
4. 对全部目标知识库执行全有或全无授权校验；
5. 获取知识库状态、活动索引版本和元数据 Schema；
6. 编译服务端强制过滤与业务过滤；
7. 按模式执行关键词、向量或双路召回；
8. 完成融合、去重、稳定排序、可选 Rerank 和阈值处理；
9. 按 top_n 与 max_context_tokens 裁剪；
10. 生成稳定引用和结构化结果；
11. 写入脱敏检索追踪并返回。

### 5.5 停用与删除流程

#### 停用

- 知识库或文档状态先在 PostgreSQL 中改变；文档版本同步记录 disable_reason=MANUAL；
- 新请求读取状态后立即排除；
- 索引状态异步同步，但不能作为唯一阻断条件；
- 恢复时仅允许执行状态迁移矩阵中定义的路径。

#### 删除

1. 管理 API 完成权限和二次确认；
2. PostgreSQL 状态改为 DELETING，并在同一事务中生成删除墓碑和 OutboxEvent；
3. 墓碑投递器把墓碑追加到独立于普通业务备份保留周期的只追加账本，记录 tombstone_id、对象类型、对象编号、删除时间、删除版本、清理范围和账本序号；
4. 新请求立即通过状态过滤拒绝访问；
5. 异步清理对象文件、预览、解析产物、内容块正文、向量、倒排索引和缓存；
6. 清理成功后状态改为 DELETED，并记录 cleanup_completed_at 和账本序号；
7. 清理失败进入重试并告警；
8. 审计和检索追踪只保留允许的标识与操作信息，不能恢复正文；
9. 备份恢复时先将业务数据置于不可对外服务状态，再从恢复点之后的墓碑账本水位重放删除，完成复核后方可恢复检索流量。

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

### 6.2 DocumentVersion 状态

~~~mermaid
stateDiagram-v2
    [*] --> UPLOADED
    UPLOADED --> PROCESSING
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

DISABLED --> PUBLISHED 不是普通状态恢复：disable_reason=MANUAL 时仅允许恢复该 Document 当前版本；disable_reason=SUPERSEDED 时只能通过显式回滚操作，并要求目标 IndexVersion 双索引仍完整、当前活动版本未发生并发变化。直接修改状态一律拒绝。

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
    PENDING --> CANCEL_REQUESTED
    RUNNING --> CANCEL_REQUESTED
    RETRY_WAIT --> CANCEL_REQUESTED
    CANCEL_REQUESTED --> CANCELLED
    CANCEL_REQUESTED --> CLEANUP_PENDING
    CLEANUP_PENDING --> CANCELLED
    CLEANUP_PENDING --> FAILED
~~~

- PENDING 表示任务已落库，可尚未成功进入队列；Scheduler 根据未投递 OutboxEvent 或超过派发时限的 PENDING 状态补投；
- RUNNING 必须持有有期限租约并定期心跳；租约过期后由 Scheduler 判断是否重试；
- RETRY_WAIT 保存下次执行时间和稳定失败分类；确定性错误直接进入 FAILED；
- FAILED 仅允许通过人工重试回到 PENDING，并创建新的 TaskExecution；
- 每次 PENDING -> RUNNING 创建新的 TaskExecution，历史执行不可覆盖；
- SUCCEEDED 仅表示入库产物校验完成并进入 REVIEW_REQUIRED，不表示已发布；
- CANCELLED 前必须确认本次未发布派生数据已清理或已进入可追踪清理任务。

### 6.5 并发与幂等

- 状态变更采用数据库事务和条件更新，提交时验证当前状态与配置版本；
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
| 七牛云 Kodo | 暂存/正式原文件和受保护预览资源 | 公司七牛云账号下的私有空间；不保存解析文本、向量或查询数据 |
| 删除墓碑账本 | 跨业务备份恢复点保留的追加写删除事件和账本水位 | 恢复安全依据，不保存正文 |
| 缓存 | 权限、配置和查询辅助缓存 | 仅性能优化，不作为唯一判断依据 |

### 7.2 核心对象关系

~~~mermaid
erDiagram
    Tenant ||--o{ UserAccount : contains
    Tenant ||--o{ ApiAccount : contains
    Tenant ||--o{ KnowledgeBase : isolates
    ApiAccount ||--o{ ApiKey : owns
    ApiKey ||--o{ ApiKeyScope : limits
    KnowledgeBase ||--o{ ApiKeyScope : scopes
    UserAccount ||--o{ KnowledgePermission : receives
    ApiAccount ||--o{ KnowledgePermission : receives
    KnowledgeBase ||--o{ KnowledgePermission : grants
    UserAccount ||--o{ LoginSession : opens
    KnowledgeBase ||--o{ Document : contains
    KnowledgeBase ||--o{ IndexVersion : publishes
    KnowledgeBase ||--o{ MetadataSchemaVersion : defines
    KnowledgeBase ||--o{ RetrievalParameterSet : evaluates
    KnowledgeBase ||--o{ EvaluationSet : owns
    Document ||--o{ DocumentVersion : versions
    DocumentVersion ||--o{ Chunk : contains
    DocumentVersion ||--o{ IngestionTask : processes
    IngestionTask ||--o{ TaskExecution : executes
    IndexVersion ||--o{ IndexSnapshotMember : contains
    Chunk ||--o{ IndexSnapshotMember : snapshots
    IndexVersion ||--|| EmbeddingModelFingerprint : binds
    Chunk ||--o{ Citation : identifies
    EvaluationSet ||--o{ EvaluationCase : contains
    EvaluationSet ||--o{ EvaluationRun : executes
    IndexVersion ||--o{ EvaluationRun : freezes
    RetrievalParameterSet ||--o{ EvaluationRun : uses
    ApiAccount ||--o{ RetrievalTrace : invokes
    ApiKey ||--o{ RetrievalTrace : authenticates
    Tenant ||--o{ AuditLog : records
    Tenant ||--o{ DeletionTombstone : protects
    IngestionTask ||--o{ OutboxEvent : dispatches
~~~

关键对象语义：

- ApiKeyScope 表达密钥自身知识库范围，实际范围仍与 ApiAccount 的 KnowledgePermission 求交集；
- IndexSnapshotMember 固化一个 IndexVersion 包含的文档版本和内容块，不从对象“当前状态”动态推导；
- EmbeddingModelFingerprint 是随 IndexVersion 冻结的不可变值对象；Rerank 和 tokenizer 版本记录在查询追踪及评测运行中；
- Citation 只保存稳定定位和允许保留的来源标识，不复制正文；
- DeletionTombstone 同时保存在 PostgreSQL 操作记录和独立追加写账本中；
- OutboxEvent 是跨数据库、队列、缓存和墓碑账本协作的可靠事件来源。

### 7.3 关键数据库约束

- 核心主键为 UUIDv7；
- 人工账号规范化邮箱唯一；
- 知识库名称全平台唯一；
- 同一主体与知识库不得存在语义冲突的多条有效授权；
- 同一 ApiKey 与 KnowledgeBase 最多一条有效 ApiKeyScope，且范围不得超过所属 ApiAccount 的当前授权；
- 同一 Document 最多一个 PUBLISHED DocumentVersion；
- 同一 KnowledgeBase 最多一个 ACTIVE IndexVersion；
- READY/ACTIVE IndexVersion 必须具有完整快照成员、Embedding 模型指纹、Schema 版本和可验证内容摘要；
- 同一文档版本最多一个处于 PENDING、RUNNING、RETRY_WAIT 或 CANCEL_REQUESTED 的有效 IngestionTask；
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
| VECTOR | 本地查询 Embedding + pgvector 召回 | 返回 503 RETRIEVAL_UNAVAILABLE |
| HYBRID | 两路召回 + RRF | 单路故障时降级到另一条链路；两路均不可用返回 503 |

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

HTTP 与 MCP 共享上述参数定义、默认值、范围和响应中的实际生效参数。知识库不保存独立在线检索参数。

服务端同时返回本次实际使用的 index_version、metadata_schema_version、embedding_model_fingerprint、rerank_model_fingerprint 和 tokenizer_version。参数合法但受故障降级影响未执行时，响应分别记录 requested_parameters 与 effective_parameters，不能把降级后的行为伪装成请求参数已完整执行。

### 8.3 得分语义

- Rerank 成功时保留模型原始 rerank_logit，并按 `1 / (1 + exp(-rerank_logit))` 计算 rerank_score；score 为 rerank_score，score_type 为 RERANK；
- 关键词模式未 Rerank 时使用关键词得分，score_type 为 KEYWORD；
- 向量模式未 Rerank 时使用余弦相似度，score_type 为 VECTOR；
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

### 9.3 Python 与框架适配

- 轻量 Python 客户端提供同步和异步 retrieve；
- RagBaseRetriever 符合 LangChain BaseRetriever 语义，支持 invoke 和 ainvoke；
- LangGraph 提供可组合的异步检索节点与 MCP 调用示例；
- 客户端只负责调用、超时、错误映射和结果转换，不执行 Embedding 或直接访问搜索引擎；
- 集成包与服务端独立发布并声明兼容版本矩阵。

### 9.4 私有管理能力

私有管理 API 按以下领域分组，均使用 `/internal/api/v1` 前缀并生成独立 OpenAPI：

| 领域 | 核心能力 | 关键控制 |
| --- | --- | --- |
| 会话与人工账号 | 登录、退出、改密、重置、锁定、禁用、恢复 | 登录限流、会话撤销、ACCOUNT_ADMIN、审计 |
| API 调用账号与密钥 | 创建、授权、禁用、轮换、删除 | 密钥只展示一次、范围交集、并存轮换、审计 |
| 知识库与授权 | 创建、配置、启停、授权、删除 | MANAGE、配置版本、二次确认、幂等 |
| 文档与任务 | 上传、任务查询、取消、重试、审核、发布、停用、删除 | EDIT、状态机、Outbox、发布锁、审计 |
| 元数据 Schema | 新版本、字段停用、迁移和重建 | MANAGE、类型不可原地修改、兼容性校验 |
| 内容块 | 预览、草稿编辑、启停 | EDIT、正文版本化、运营状态同步 |
| 评测 | 评测集版本、运行、对比、门禁记录 | EVALUATE、快照冻结、资源隔离 |
| 平台运维 | 模型、节点、队列、容量、追踪、告警 | PLATFORM_ADMIN、正文最小暴露 |

重复提交可能产生重复副作用的管理写操作接受 Idempotency-Key；全部管理接口在 OpenAPI 中声明所需平台角色、知识库权限、前置状态、成功后状态、并发版本和稳定错误码。管理前端不得绕过 API 直接修改 PostgreSQL、Elasticsearch 或对象存储。

### 9.5 错误模型

| HTTP 状态 | 领域语义 |
| --- | --- |
| 400/422 | 参数、过滤、类型或资源上限错误 |
| 401 | API Key 或所属调用账号无效 |
| 403 | FORBIDDEN_SCOPE；不存在、删除、无权访问对外统一处理；已知 citation_id 的受控墓碑查询按下述例外处理 |
| 404 | RESOURCE_NOT_FOUND；仅限具备管理范围的内部管理 API |
| 409 | 并发、非法状态迁移、配置版本或发布冲突 |
| 429 | 配额、频率或并发超限 |
| 503 | 必需检索依赖不可用且不能降级 |
| 504 | 在线检索或下游依赖超时 |

错误响应统一包含 trace_id、稳定错误码、简短说明和是否可重试。MCP 将相同内部错误枚举映射为协议错误对象，不暴露堆栈、路径或资源存在性。

引用例外只适用于引用详情或 MCP get_chunk 的 citation_id 形态：调用方提交已知且不可猜测的 citation_id，并通过当前或保留的历史授权校验时，可返回 CURRENT、HISTORICAL_UNAVAILABLE 或 SOURCE_DELETED 状态。知识库列表、搜索、普通 chunk_id 读取和任意资源编号探测不适用该例外，仍统一返回 403。

## 10. 安全设计

### 10.1 信任边界

- 管理前端和私有管理 API 位于内网管理边界；
- HTTP/MCP 调用经过检索入口、TLS、API Key 鉴权和限流；
- 解析 Worker 处理不可信文档，部署在受限执行环境；
- 数据区和模型区只允许应用与任务执行单元按最小权限访问；
- API、MCP、Worker、Scheduler、模型服务和基础设施之间使用受信网络叠加服务身份认证；仅处于同一内网不能替代服务认证；
- 对外出站流量统一通过代理、域名白名单和审计；七牛云 Kodo 仅允许访问已配置的上传、管理和私有下载域名。

HTTP 与 MCP 共用以 API Key 编号和 ApiAccount 编号为主键的分布式限流计数，知识库维度和全局并发作为附加限制。限流存储不可用时执行可配置的失败策略：外部检索默认保守拒绝高风险突发请求，健康检查和内部恢复流量使用独立配额；不得因切换协议获得新的额度。

### 10.2 数据保护

- 数据库、七牛云 Kodo、日志和备份启用静态加密；
- 密钥与业务数据分离保管，不写入代码、镜像或普通配置；
- 密钥通过公司批准的密钥管理或独立密钥托管机制引用；备份保存的是加密数据和受控恢复所需的密钥引用/托管材料，不把可直接解密业务数据的明文密钥与同一份备份共同存放；
- 密钥轮换记录 key_id、启用时间、退役时间和用途；API 签名、HMAC 指纹、七牛云 AccessKey/SecretKey 和备份密钥采用相互独立的用途域；
- 七牛云 Kodo 使用私有空间和服务端加密；原文件与预览只允许在业务服务完成当前权限校验后生成短期有效下载地址，不提供永久公开 URL；
- 七牛云访问密钥仅由后端和 Worker 通过密钥管理机制读取，管理前端、日志和任务消息不得包含 SecretKey；
- 文档、查询和候选结果不得发送给外部 Embedding 或 Rerank 服务；
- 日志分级脱敏，禁止记录密码、API Key、模型密钥和完整正文；
- 删除覆盖原文件、解析文本、内容块、向量、倒排索引、缓存和预览。

### 10.3 输入安全

- 校验扩展名、MIME、文件特征、大小、页数、解压大小和格式专属上限；
- 防止路径穿越、压缩炸弹、恶意文档、SSRF、越权下载和查询注入；
- HTML 不访问外部 URL，文档中的宏、脚本、命令和嵌入对象不执行；
- metadata_filter 经受控语法树编译，查询文本不直接拼接后端查询；
- 查看正文、预览和引用均在请求时重新鉴权。

管理入口限制可信 Origin 和 Host，登录与敏感写操作执行 CSRF 防护、重放保护和频率限制。解析 Worker 使用非特权身份、只读基础镜像、临时工作目录、文件系统/CPU/内存/执行时间限制及默认禁止出站网络；格式解析器不得继承 API 进程的数据库或密钥权限。

## 11. 非功能设计

### 11.1 性能与容量

- API 进程与 Worker 分开扩展，在线检索不消费入库工作池；
- Embedding、Rerank、解析和索引写入配置独立并发上限；
- 请求并发、队列长度、超时和限流均外置配置；
- 上传和解析采用流式或分段方式；
- 指标记录关键词、向量、混合、Rerank 开关及 HTTP/MCP 协议差异；
- 当前不承诺固定 QPS、延迟和可用性数值，先形成可复现基线；
- 容量达到 70%、85%、95% 时分别告警，不自动删除已发布内容。

### 11.2 可用性与故障处理

- API、MCP、Worker 和模型依赖分别提供健康检查；
- 健康检查区分 liveness、readiness 和 startup：存活检查不访问重依赖，就绪检查验证提供当前角色所必需的依赖，启动检查允许模型和迁移在限定时间内完成；
- Embedding 与 Rerank 调用配置超时、并发隔离和熔断；
- 入库任务支持异常中断识别、有限重试和人工重试；
- 索引切换失败继续使用旧活动版本；
- 任务队列或 Worker 故障不阻断已发布内容的在线检索；
- Elasticsearch、pgvector 和七牛云 Kodo 异常分别产生可定位告警；
- MVP 同一环境只保持一个有效 Scheduler；周期任务执行前仍通过 PostgreSQL 锁或幂等键防止重启、重复触发造成重复任务，不引入独立调度集群；
- 进程优雅停机时先退出就绪状态，停止领取新任务，在宽限期内完成或释放任务租约，并取消不再需要的下游模型请求。

### 11.3 可观测性

统一观测维度包括：

- request_id、trace_id、task_id、knowledge_base_id 和 index_version；
- HTTP 路径或 MCP 工具名、调用账号和结果状态；
- 权限过滤、查询向量、关键词召回、向量召回、融合、Rerank 和返回阶段耗时；
- QPS、成功率、P50/P95/P99、无结果率、降级率和错误码分布；
- 队列长度、任务阶段、重试次数、索引大小、存储使用量和模型调用状态。

告警至少覆盖检索错误率上升、延迟异常、降级率异常、任务积压、索引不一致、模型不可用和存储不足。

每类告警必须配置责任模块、严重级别、抑制/聚合规则、诊断入口和处置手册。告警通过测试环境故障注入产生过真实事件后，才能计入上线证据。

### 11.4 备份与恢复

- 备份覆盖 PostgreSQL、七牛云对象及对象清单、索引配置、密钥引用/受控托管材料、追加写墓碑账本和必要运维配置；
- Elasticsearch 和 pgvector 派生索引可重建，但恢复流程仍需验证活动版本一致；
- 恢复后验证账号、授权、API Key 状态、文档状态、活动索引版本和删除墓碑；
- 目标为 RPO ≤ 24 小时、RTO ≤ 8 小时；
- 至少每季度完成一次恢复演练并保留结果；
- 升级前执行兼容性检查和备份，失败时回滚到上一可用版本；
- 恢复顺序固定为：恢复墓碑账本和密钥访问能力、恢复 PostgreSQL、应用恢复点后的墓碑、恢复对象数据、重建或验证派生索引、验证权限与活动版本、执行抽样检索，最后开放流量；
- RTO 演练分别记录各阶段耗时，若全量重建派生索引无法在 8 小时内完成，必须保留满足恢复目标的索引备份或缩短重建路径；不得只声明目标而不验证容量条件。

### 11.5 数据保留

- 原文件、检索追踪和审计日志不按时间自动过期；
- 经授权的显式删除、法规删除和安全事件处置按删除流程执行；
- 删除后的追踪只保留标识、版本、得分、耗时和操作结果，不保留可恢复正文；
- 备份按既定生命周期自然过期，恢复时重放删除墓碑；
- 删除墓碑账本的保留期不得短于最长业务备份生命周期加恢复缓冲期；账本完整性校验失败时禁止恢复环境上线。

### 11.6 前端兼容性

- React 管理前端支持 Chrome、Edge、Safari 最近两个稳定主版本；
- 关键管理流程以真实浏览器执行兼容性测试，不以组件测试或构建成功代替；
- 浏览器不支持时给出明确提示，不通过降低后端权限、安全或数据校验规则兼容旧环境。

## 12. 配置与发布设计

### 12.1 外置配置

以下配置不得写死在代码或镜像：

- 数据库、Elasticsearch、七牛云空间/区域/域名、任务队列和缓存连接；
- Embedding 与 Rerank 服务地址、超时、并发和批次；
- 上传、解压、页数、工作表、单元格、附件等资源上限；
- API 限流、任务并发、重试、队列长度和熔断阈值；
- TLS、代理、外网白名单和内部 DNS；
- 七牛云 AccessKey/SecretKey 的密钥引用、短期下载地址有效期和日志脱敏策略；
- Embedding 查询前缀规则属于 IndexVersion 指纹；tokenizer 名称与版本、Rerank 版本、评分转换版本和父子块聚合策略属于 RetrievalTrace 与 EvaluationRun 的查询期版本记录。

模型名称、模型版本、文件校验值和向量维度属于只读部署基线，运行时不得静默更新。

### 12.2 启动依赖

建议启动顺序：

1. PostgreSQL/pgvector、Elasticsearch 和 Redis；
2. 七牛云 Kodo 私有空间连通性、本地 Embedding 与 Rerank 服务；
3. 数据库迁移和初始化检查；
4. FastAPI API 与 MCP 适配进程；
5. Celery Worker 与 Celery Beat/Scheduler；
6. 健康检查、最小模型推理和索引连通性验证；
7. 管理前端和调用方连通性验证。

“建议启动顺序”不代表具体容器编排产品选型。

启动顺序不能替代依赖就绪判断。各进程必须在依赖暂时不可用时有限重试并保持未就绪，不得无限阻塞或在依赖未验证时接受业务流量。

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
- Python 客户端与 LangChain/LangGraph 示例；
- 部署、升级、回滚、备份恢复和运维手册。

## 13. 设计决策与待详细设计项

### 13.1 已确定设计决策

| 编号 | 决策 |
| --- | --- |
| DD-001 | 核心业务采用 FastAPI 模块化单体，不按业务模块拆分微服务 |
| DD-002 | API、MCP、Celery Worker、Celery Beat/Scheduler 为独立运行角色，共享领域与应用代码 |
| DD-003 | PostgreSQL 为业务事实来源，Elasticsearch 与向量索引均为版本化派生数据 |
| DD-004 | PostgreSQL 活动索引版本是在线发布开关 |
| DD-005 | HTTP 与 MCP 从同一领域契约映射并共享检索服务 |
| DD-006 | 权限、版本和状态异常禁止降级 |
| DD-007 | 文档处理异步执行，在线检索与入库资源隔离 |
| DD-008 | 不引入答案生成、工作流、多租户和需求外数据源 |
| DD-009 | PostgreSQL 事务 Outbox + 至少一次投递 + Worker 租约实现可靠异步处理 |
| DD-010 | IndexVersion 是完整快照，发布必须校验 base_active_index_version 并串行切换 |
| DD-011 | IndexVersion 固定绑定 Embedding、Schema、分块和索引结构指纹；Rerank 与 tokenizer 版本记录在查询追踪和评测运行中 |
| DD-012 | 内容块停用先改权威状态，启用先同步双索引再改有效状态 |
| DD-013 | 删除墓碑同时进入 PostgreSQL 与独立追加写账本，恢复后先重放墓碑再开放流量 |
| DD-014 | 已删除引用只在已知 citation_id 且通过当前/历史授权校验时返回最小墓碑状态 |
| DD-015 | 管理前端采用 React 技术栈；后端采用 FastAPI；入库采用 LlamaIndex；Celery + Redis 承担异步任务与消息投递；Redis 同时提供隔离的缓存空间 |
| DD-016 | 文档原文件和受保护预览使用公司七牛云账号下的 Kodo 私有空间；解析文本、内容块、索引、查询和模型数据不写入七牛云 |

### 13.2 待后续详细设计确定

下列内容不影响系统边界和核心语义，留待详细设计或部署方案确定：

- 容器编排、日志、指标和追踪产品；
- 物理节点数量、CPU/GPU、内存和磁盘规格；
- PostgreSQL 分区、索引和表结构细节；
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
| FR-AUTH-001、FR-AUTH-004 | 4.2.1、6.5、10 | 账号状态、登录锁定、临时密码、可撤销会话、审计 | 登录失败锁定、改密/禁用后旧会话失效 |
| FR-AUTH-002、FR-AUTH-003、FR-AUTH-006 | 4.2.3、7.2、9.4 | 全有或全无授权、角色/权限矩阵、直接授权、缓存失效 | 跨库拒绝、平台角色不读取正文、授权变更生效 |
| FR-AUTH-005 | 4.2.2、7.2、10.1 | API 调用账号、ApiKeyScope、范围交集、轮换、共享限流 | 新旧密钥并存、禁用立即失效、HTTP/MCP 不绕限流 |
| FR-KB-001、FR-KB-002 | 4.3、4.5、6.1 | 创建权限、固定模型、活动快照与 Embedding 指纹 | 空库无结果、模型变更触发重建 |
| FR-KB-003 | 5.5、7.2、11.4 | 二次确认、墓碑账本、异步清理和恢复重放 | 删除立即不可见、清理重试、旧备份恢复不复活 |
| FR-DOC-001 | 4.4.1、4.4.2、9.4、12.1 | 暂存上传、格式适配、资源限制、批次单文件隔离 | 支持格式矩阵、超限错误、批量部分失败 |
| FR-DOC-002 | 4.3、4.7、7.2 | Schema 版本、继承/覆盖、权限字段服务端生成 | K12 模板可选、字段迁移和双后端过滤 |
| FR-DOC-003、FR-DOC-004 | 4.3、4.5、5.2、5.5、6.2 | 单发布版本、disable_reason、快照 CAS、停用和删除 | V2 构建时 V1 在线、并发发布、显式回滚 |
| FR-INGEST-001、FR-INGEST-004 | 4.4、5.1、6.4、6.5 | Outbox、租约、执行记录、取消、重试和清理 | 队列故障补投、Worker 崩溃接管、取消无半成品 |
| FR-INGEST-002、FR-INGEST-003 | 4.4.2、7.1 | 中间产物、格式定位、OCR 质量、清洗版本和告警 | 黄金样例、低置信度审核、清洗前后追溯 |
| FR-INGEST-005 | 3.3、4.4.2、4.5 | LlamaIndex 适配层、平台对象映射、禁止绕过发布 | 本地流水线与生产索引隔离 |
| FR-CHUNK-001、FR-CHUNK-002、FR-CHUNK-003 | 4.3、4.4.2、4.6、7.2 | 分块配置版本、父子关系、快照成员和定位 | 策略样例、父子引用、配置变更重建 |
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
| FR-EVAL-001、FR-EVAL-002、FR-EVAL-003 | 4.9、6.4、7.2 | 单条调试、不可变运行快照、指标口径、门禁记录 | 100 问题、结果复现、失败阻止首次上线 |
| FR-ADMIN-001、FR-ADMIN-002、FR-ADMIN-003、运营管理界面 | 3.3、4.1.4、4.10、9.4、11 | 模型/运行/诊断模块、React 管理前端、管理 API 和最小正文暴露 | 管理页面契约、浏览器兼容、追踪查询、健康和告警 |

### 14.2 非功能与验收追溯

| 需求范围 | 设计章节 | 证据要求 |
| --- | --- | --- |
| NFR-PERF-001、NFR-PERF-002、NFR-PERF-003、NFR-PERF-004、NFR-PERF-005、NFR-PERF-006 | 11.1、11.3 | 固定数据、参数、模型、硬件和并发的 HTTP/MCP 性能报告 |
| 安全与隐私 | 4.2、4.7、10、11.4 | 权限攻击用例、密钥/日志扫描、解析隔离、加密和恢复证据 |
| 可用性与可靠性 | 4.4、4.5、6、11.2 | 队列/Worker/模型/搜索故障注入和恢复记录 |
| 容量、保留与私有化 | 11.1、11.4、11.5、12 | 容量基线、墓碑恢复、RPO/RTO、网络边界和依赖清单 |
| AC-001、AC-006、AC-007、AC-009 | 4.3～4.5、5、6 | 入库发布、更新删除、并发和故障恢复端到端证据 |
| AC-002、AC-003、AC-004、AC-005 | 4.6～4.8、8、9 | 真实 API、权限、三种模式、过滤和引用证据 |
| AC-008 | 4.9 | 冻结评测运行及门禁记录 |
| AC-010、AC-011 | 4.1.3、9.2、9.3 | MCP、LangChain、LangGraph 兼容和真实调用证据 |
| AC-012 | 10～12 | 核心服务全新部署、七牛云 Kodo 私有存储接入、升级回滚、备份恢复和网络边界证据 |

### 14.3 需求确认单追溯

| 确认项 | 设计章节 | 一致性结论 |
| --- | --- | --- |
| C-AUTH-01、C-AUTH-02、C-AUTH-03、C-AUTH-04、C-AUTH-05、C-AUTH-06、C-AUTH-07、C-AUTH-08、C-AUTH-09 | 4.1、4.2、6.5、9.4、10 | 独立账号、固定角色、直接授权、API Key 范围交集和权限失效方式一致 |
| C-DOC-01、C-DOC-02、C-DOC-03、C-DOC-04、C-DOC-05、C-DOC-06、C-DOC-07、C-DOC-08 | 4.3～4.5、5.2、5.3、5.5、6 | 文档版本、双索引发布、内容块编辑/启停、删除和历史引用语义一致 |
| C-META-01、C-META-02、C-META-03、C-META-04、C-META-05、C-META-06 | 4.3、4.7、7 | 单租户固定字段、K12 可选模板、Schema 版本和继承规则一致 |
| C-INGEST-01、C-INGEST-02、C-INGEST-03 | 2.3、4.4、5.1、6.4 | Celery 异步执行不改变任务语义；LlamaIndex 仅用于本地入库适配，不调用托管解析服务 |
| C-SEARCH-01、C-SEARCH-02、C-SEARCH-03、C-SEARCH-04、C-SEARCH-05、C-SEARCH-06、C-SEARCH-07、C-SEARCH-08 | 4.6～4.8、8、9 | 三种检索模式、双检索后端、本地模型、降级、空结果、引用和私有管理边界一致 |
| C-MCP-01、C-FRAMEWORK-01 | 4.1.3、9.2、9.3 | MCP 无状态协议适配和 LangChain/LangGraph 调用端边界一致 |
| C-NFR-01、C-NFR-02、C-NFR-03、C-NFR-04、C-NFR-05、C-NFR-06、C-NFR-07、C-NFR-08 | 10～12 | 性能基线、可靠性、TLS/加密、查询指纹、保留、墓碑恢复和浏览器兼容一致 |

## 15. 设计验收检查

系统设计评审至少确认：

- 架构未超出需求确认的 MVP 边界；
- HTTP、MCP 和框架适配共享同一检索核心；
- PostgreSQL、pgvector、Elasticsearch 和对象存储职责明确；
- 七牛云 Kodo 使用公司账号私有空间、HTTPS、服务端加密和短期下载地址，前端不持有存储密钥；
- 权限、文档版本和索引版本不会被缓存或降级绕过；
- 新版本发布校验基础活动版本，并发冲突或切换失败时旧版本继续可用；
- Embedding、Schema、分块和索引结构指纹随活动快照冻结，Rerank 与 tokenizer 版本随查询和评测记录；
- 停用和删除能够立即阻止新访问；
- 内容块启用只有在双索引同步成功后才对外显示成功；
- 历史引用和已删除引用不会返回正文；
- 入库任务可追踪、可恢复且不产生重复可见数据；
- 数据库提交而队列不可用时可由 Outbox 补投，取消和租约超时具有明确终态；
- 三种检索模式、参数、得分、过滤和降级语义与需求一致；
- 审计、追踪、加密、日志脱敏和查询指纹满足安全要求；
- 备份恢复、删除墓碑和 RPO/RTO 有明确设计落点；
- 从早于删除时间的备份恢复后，墓碑账本仍能阻止已删除正文重新上线；
- 除第 2.3 节已确定的项目技术栈外，未指定的基础设施产品和容量数值没有被擅自固化；
- 需求追溯表逐项给出设计落点和主要验证，不以章节范围代替实际覆盖。

## 16. 关联文档

- [RAG 知识库平台需求说明书 V1.23](./01-RAG知识库平台需求说明书.md)
- [RAG 知识库平台需求确认单 V1.2](./02-RAG知识库平台需求确认单.md)
