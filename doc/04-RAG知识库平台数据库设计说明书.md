# RAG 知识库平台数据库设计说明书

> 项目名称：RAG Base  
> 文档版本：V1.3
> 文档状态：数据库设计评审稿  
> 编制日期：2026-09-22  
> 需求基线：《RAG 知识库平台需求说明书》V1.25
> 确认基线：《RAG 知识库平台需求确认单》V1.4
> 系统设计基线：《RAG 知识库平台系统设计说明书》V1.7
> 修订日期：2026-09-23

> V1.3 修订：明确纯索引重建不迁移文档版本、不回退知识库当前 Schema；区分快照构建配置与文档实际分块配置，补充 REPROCESS 版本及新 Embedding 配置的事务路径，并更新上游评审稿基线。
>
> V1.2 修订：对齐 UPLOADED 删除迁移、Rerank 评测后切换和三份上游评审稿的新版本与校验值。
>
> V1.1 修订：对齐 V1.5 的上传预留、对象生命周期、不可变向量复用、可见性代次、任务执行、删除和一致恢复点；修正主外键、唯一性、发布顺序及数据清除规则。

## 1. 文档说明

### 1.1 编制目的

本文定义 RAG Base MVP 的 PostgreSQL/pgvector 数据模型、表结构、主外键、唯一约束、索引、状态字段、事务边界、数据保留和派生索引映射，用于指导 SQLAlchemy 模型、Alembic 迁移、数据访问代码和数据库测试。

### 1.2 设计范围

本文仅覆盖上游定义的数据库设计内容；基线状态见 1.4 节：

- 单租户人工账号、API 调用账号、API Key、会话、角色和知识库直接授权；
- 知识库、元数据 Schema、文档、文档版本、七牛云对象引用、解析产物、内容块和引用；
- 入库任务、执行记录、事务 Outbox 和写操作幂等；
- 索引版本、完整快照成员、Embedding 指纹和 pgvector 向量；
- 检索参数集、脱敏检索追踪、评测集、评测运行和上线门禁；
- 审计日志、删除墓碑、对象备份恢复点及必要的运行配置；
- PostgreSQL 与 Elasticsearch、Redis、七牛云 Kodo、追加写墓碑账本之间的数据边界。

### 1.3 非本文范围

本文不设计聊天、会话问答、答案生成、工作流、计费、多租户、组织层级、GraphRAG、外部数据源同步、音视频处理或其他需求外数据表；不规定服务器规格、数据库高可用产品、Elasticsearch 分片数量和七牛云 Bucket 创建流程。

### 1.4 基线状态

本文对照 01 V1.25、02 V1.4 和 03 V1.7 的当前内容编制。需求状态矩阵已允许 `UPLOADED -> DELETING -> DELETED`；Embedding 升级需新建索引并全量重建，Rerank 升级在通过冻结评测回归后切换查询期版本，不重建索引。确认单记录的评审稿校验值与当前需求说明书一致；上游文档仍未签署，本稿为数据库设计评审稿。

本次核对的文件 SHA-256：01 为 `aee1da5ab30ff11ca94d9853758ab0fdaefc44ed4b2ac304ca0b65321c097fa8`，02 为 `afcfebed17bc107b1dcba266e11624816b89802c2625a2dfbaa818cad720af42`，03 为 `587b56bc808c640afd46fb82aa5ac930b5251c1bcfc8e48d74be072c40928d95`。上游变更后重新进行影响分析；本稿不代表已签署基线。

## 2. 数据库技术基线与通用约定

### 2.1 技术基线

| 项目 | 设计约定 |
| --- | --- |
| 数据库 | PostgreSQL，精确版本在开发前技术基线中冻结 |
| 向量扩展 | pgvector，向量维度固定为 1024 |
| ORM 与迁移 | SQLAlchemy、Alembic |
| 主键 | 核心实体使用应用生成的 UUIDv7，数据库类型为 `UUID` |
| 时间 | `TIMESTAMPTZ`，统一写入 UTC，展示层转换为 `Asia/Shanghai` |
| 半结构数据 | 仅在结构需要版本化或字段由业务 Schema 定义时使用 `JSONB` |
| 向量距离 | L2 归一化向量，使用余弦距离/相似度 |
| 字符编码 | UTF-8 |

开发前至少验证并冻结 PostgreSQL、pgvector、SQLAlchemy 和 Alembic 的兼容版本。生产迁移不得依赖 `latest` 镜像或未锁定依赖。

### 2.2 命名约定

- 表、列、约束和索引使用小写 `snake_case`；
- 主键统一命名为 `id`，外键统一命名为 `<对象>_id`；
- 创建、更新时间统一为 `created_at`、`updated_at`；
- 业务状态使用可读字符串并配合 `CHECK` 约束，不使用难以演进的 PostgreSQL ENUM；
- 金额类字段本期不存在，不新增金额类型；
- 平台 SHA-256 和 HMAC-SHA256 使用 64 位小写十六进制字符串并校验格式；Kodo hash 和自适应密码哈希保留原算法编码，不套用此约定；
- JSONB 空集合使用空对象或空数组；“尚未生成的结果”允许 SQL NULL，并由具体字段定义限定；JSONB 必须同时校验顶层类型和字段白名单。

### 2.3 通用字段规则

第 6 章每张实体表均包含 `id UUID PRIMARY KEY` 和 `created_at TIMESTAMPTZ NOT NULL`，由应用生成 UUIDv7、数据库记录创建时间；表内重列同名字段不代表重复列。除下述追加写表外，各表同时包含 `updated_at` 和 `lock_version`，用于状态或受控生命周期更新：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `UUID` | 应用生成 UUIDv7；数据库不生成随机 UUID |
| `created_at` | `TIMESTAMPTZ` | 创建时间，不可回写 |
| `updated_at` | `TIMESTAMPTZ` | 最后更新时间 |
| `lock_version` | `BIGINT` | 乐观锁版本，从 0 开始，每次受控更新加 1 |

`retrieval_trace`、`audit_log`、`release_gate_record` 为追加写表，不设置 `updated_at` 和 `lock_version`。模型指纹和向量产物的内容不可修改，但保留通用字段用于统一映射；任务执行记录允许 QUEUED/RUNNING 期间更新，进入终态后不可覆盖。本文的 `lock_version` 对应系统设计中的 `row_version`，不与配置版本或可见性代次混用。

所有状态/类型枚举均建立 CHECK；计数、代次和耗时非负，业务版本号从 1 开始。空值为“否”的字段必须 NOT NULL；跨行归属不能用 CHECK 子查询模拟，使用外键、唯一约束和受控事务。全库只有一个固定 tenant，含 tenant_id 的表均引用它，无 tenant_id 的表沿所属实体关系确定该固定租户。除明确声明为历史快照或多态引用外，表中 UUID 关系字段均建立对应外键；普通外键的索引按查询路径配置。

### 2.4 扩展与初始化数据

数据库初始化启用 `vector` 扩展。初始数据仅包含：

- 固定租户记录；
- `PLATFORM_ADMIN`、`ACCOUNT_ADMIN`、`USER` 三个角色定义；
- 通过安全初始化流程创建的唯一初始平台管理员，并授予 `PLATFORM_ADMIN`、`ACCOUNT_ADMIN`；如需由其创建知识库，再显式授予 `USER`；
- 固定模型标识和只读模型基线由部署流程写入，不在迁移脚本中写入密钥、服务地址或通用默认密码。

## 3. 数据职责与边界

| 数据类型 | 权威存储 | 说明 |
| --- | --- | --- |
| 账号、权限、状态、版本、任务、内容块、评测 | PostgreSQL | 唯一业务事实来源 |
| 解析原始文本、清洗文本 | PostgreSQL `document_artifact` | 压缩保存于公司私有数据库，不写入七牛云 |
| 内容块向量 | PostgreSQL/pgvector | 可由内容块和模型指纹重建 |
| 关键词倒排索引 | Elasticsearch | 可重建派生数据，不作为状态和权限事实来源 |
| 原文件、暂存文件、受保护预览 | 七牛云 Kodo 私有空间 | PostgreSQL 仅保存对象键、摘要、状态和校验结果，不保存永久或签名 URL |
| Celery 消息、短期缓存、限流计数 | Redis | 不保存唯一业务事实 |
| 删除账本 | PostgreSQL 墓碑记录 + 独立追加写账本 | PostgreSQL 保存同步状态和账本序号，不保存被删除正文 |
| 备份对象清单 | 公司私有备份设施 | PostgreSQL 仅保存恢复点、清单引用、摘要和一致性水位 |

任何签名下载 URL、上传 Token、下载 Token、七牛云 AccessKey/SecretKey、API Key 明文、密码明文和模型密钥均不得写入数据库。在线原始检索问题不落库；经批准脱敏的评测问题仅保存在 evaluation_case，不写入检索追踪或审计。

## 4. 逻辑数据模型

### 4.1 身份与授权

~~~mermaid
erDiagram
    tenant ||--o{ user_account : contains
    tenant ||--o{ api_account : contains
    user_account ||--o{ user_role : has
    role_definition ||--o{ user_role : assigns
    user_account ||--o{ login_session : opens
    user_account ||--o{ password_reset_credential : resets
    api_account ||--o{ api_key : owns
    user_account o|--o{ knowledge_permission : receives
    api_account o|--o{ knowledge_permission : receives
    knowledge_base ||--o{ knowledge_permission : grants
    api_key ||--o{ api_key_scope : limits
    knowledge_base ||--o{ api_key_scope : scopes
~~~

### 4.2 知识内容与索引

~~~mermaid
erDiagram
    tenant ||--o{ knowledge_base : contains
    knowledge_base ||--o{ metadata_schema_version : defines
    metadata_schema_version ||--o{ metadata_field_definition : contains
    knowledge_base ||--o{ document : contains
    document ||--o{ document_version : versions
    document_version ||--o{ document_object : references
    document_version ||--o{ document_artifact : produces
    document_artifact ||--o{ document_artifact_part : splits
    document_version ||--o{ chunk : contains
    document_version ||--o{ document_relation : parent
    document ||--o{ document_relation : child
    chunk o|--o| citation : identifies
    knowledge_base ||--o{ index_version : publishes
    index_version ||--o{ index_snapshot_member : contains
    chunk ||--o{ index_snapshot_member : snapshots
    embedding_model_fingerprint ||--o{ index_version : binds
    embedding_artifact ||--o{ index_snapshot_member : supplies
    embedding_model_fingerprint ||--o{ embedding_artifact : defines
    knowledge_base ||--o{ chunking_config_version : defines
    chunking_config_version ||--o{ index_version : binds
    upload_reservation o|--o| document_object : consumed_as
~~~

### 4.3 任务、评测与治理

~~~mermaid
erDiagram
    document_version ||--o{ ingestion_task : processes
    ingestion_task ||--o{ task_execution : executes
    tenant ||--o{ outbox_event : records
    knowledge_base o|--o{ retrieval_parameter_set : owns
    knowledge_base ||--o{ evaluation_set : owns
    evaluation_set ||--o{ evaluation_set_version : versions
    evaluation_set_version ||--o{ evaluation_case : contains
    evaluation_set_version ||--o{ evaluation_run : runs
    index_version ||--o{ evaluation_run : freezes
    retrieval_parameter_set ||--o{ evaluation_run : uses
    tenant ||--o{ retrieval_trace : records
    tenant ||--o{ audit_log : audits
    tenant ||--o{ deletion_tombstone : protects
    deletion_tombstone ||--|| deletion_task : drives
    deletion_task ||--o{ deletion_execution : executes
    deletion_tombstone ||--o{ deleted_access_grant : preserves
    tenant ||--o{ backup_recovery_point : checkpoints
~~~

## 5. 表清单

| 分组 | 表名 | 用途 |
| --- | --- | --- |
| 身份 | `tenant` | 固定单租户身份 |
| 身份 | `user_account` | 人工登录账号 |
| 身份 | `role_definition` | 固定平台角色 |
| 身份 | `user_role` | 人工账号角色关系 |
| 身份 | `login_session` | 可撤销登录会话 |
| 身份 | `password_reset_credential` | 一次性密码重置凭证 |
| 身份 | `api_account` | 非登录 API 调用主体 |
| 身份 | `api_key` | API Key 摘要与状态 |
| 权限 | `knowledge_permission` | 人工/API 账号的知识库直接授权 |
| 权限 | `api_key_scope` | API Key 自身知识库范围 |
| 内容 | `knowledge_base` | 知识库配置与活动版本引用 |
| 内容 | `metadata_schema_version` | 元数据 Schema 版本 |
| 内容 | `metadata_field_definition` | Schema 字段定义 |
| 内容 | `document` | 稳定业务文档身份 |
| 内容 | `document_version` | 文件和正文版本 |
| 内容 | `document_object` | 七牛云对象引用与生命周期 |
| 内容 | `document_relation` | EML 附件等父子文档关系 |
| 内容 | `document_artifact` | 解析和清洗文本产物 |
| 内容 | `document_artifact_part` | 大型解析产物的有序分段内容 |
| 内容 | `chunk` | 版本化内容块 |
| 内容 | `citation` | 稳定引用标识 |
| 任务 | `ingestion_task` | 入库任务权威状态 |
| 任务 | `task_execution` | 每次实际执行历史 |
| 任务 | `outbox_event` | 事务事件可靠投递 |
| 任务 | `idempotency_record` | 管理写操作幂等结果 |
| 索引 | `embedding_model_fingerprint` | 不可变 Embedding 空间指纹 |
| 索引 | `index_version` | 知识库完整索引快照版本 |
| 索引 | `index_snapshot_member` | 快照包含的不可变内容块清单 |
| 索引 | `embedding_artifact` | 按文本与模型指纹复用的不可变 1024 维向量 |
| 内容 | `upload_reservation` | Kodo 写入前的上传预留与孤儿回收保护 |
| 内容 | `chunking_config_version` | 不可变分块配置版本 |
| 治理 | `deletion_task` | 删除编排、租约和阶段进度 |
| 治理 | `deletion_execution` | 删除执行历史 |
| 权限 | `deleted_access_grant` | 删除引用所需历史授权交集 |
| 检索 | `retrieval_parameter_set` | 命名检索参数集 |
| 检索 | `retrieval_trace` | 脱敏检索链路摘要 |
| 评测 | `evaluation_set` | 评测集稳定身份 |
| 评测 | `evaluation_set_version` | 冻结的评测集版本 |
| 评测 | `evaluation_case` | 问题与期望命中 |
| 评测 | `evaluation_run` | 一次批量评测运行 |
| 评测 | `evaluation_case_result` | 单用例评测结果摘要 |
| 评测 | `release_gate_record` | 首次上线质量门禁证据 |
| 配置 | `model_runtime_config` | 本地模型运行参数版本 |
| 配置 | `security_policy` | 登录和密码安全策略版本 |
| 治理 | `audit_log` | 追加写审计日志 |
| 治理 | `deletion_tombstone` | 删除墓碑与账本同步状态 |
| 治理 | `backup_recovery_point` | 数据库、对象和墓碑的一致恢复点 |

## 6. 表结构设计

### 6.1 `tenant`

MVP 只允许存在一条固定租户记录，不提供创建、切换或停用接口。

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `id` | `UUID` | 否 | 固定租户 UUIDv7 |
| `code` | `VARCHAR(64)` | 否 | 固定代码，唯一 |
| `name` | `VARCHAR(128)` | 否 | 展示名称 |
| `status` | `VARCHAR(16)` | 否 | 固定为 `ACTIVE` |
| `created_at` | `TIMESTAMPTZ` | 否 | 创建时间 |

约束：`UNIQUE(code)`、`CHECK(status='ACTIVE')`，另建 `CREATE UNIQUE INDEX uq_tenant_singleton ON tenant ((true));` 保证至多一行。初始化写入固定记录，业务数据库角色无 INSERT/DELETE 权限，启动验证该记录存在；“至多一行”与“必须存在”分别验证。

### 6.2 `user_account`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `tenant_id` | `UUID` | 否 | 固定租户 |
| `email` | `VARCHAR(320)` | 否 | 原始展示邮箱 |
| `normalized_email` | `VARCHAR(320)` | 否 | 去首尾空格并统一小写 |
| `display_name` | `VARCHAR(128)` | 否 | 显示名称 |
| `password_hash` | `TEXT` | 否 | 自适应密码哈希完整编码结果 |
| `status` | `VARCHAR(16)` | 否 | `ACTIVE/LOCKED/DISABLED` |
| `failed_login_count` | `INTEGER` | 否 | 连续失败次数，默认 0 |
| `locked_until` | `TIMESTAMPTZ` | 是 | 临时锁定截止时间 |
| `session_version` | `BIGINT` | 否 | 强制全部会话失效时递增 |
| `authorization_version` | `BIGINT` | 否 | 直接授权或角色变更时递增，默认 0 |
| `must_change_password` | `BOOLEAN` | 否 | 是否必须修改临时密码 |
| `last_login_at` | `TIMESTAMPTZ` | 是 | 最后成功登录时间 |
| `disabled_at` | `TIMESTAMPTZ` | 是 | 禁用时间 |
| `created_by` | `UUID` | 是 | 创建人；初始管理员可空 |

索引与约束：UNIQUE(tenant_id,normalized_email)；CHECK(normalized_email=lower(btrim(email)))；按 (tenant_id,status) 建管理列表索引；failed_login_count ≥ 0。登录成功更新 last_login_at，失败不更新；必须改密的会话仅允许改密与退出。

### 6.3 `role_definition` 与 `user_role`

`role_definition`：

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `id` | `UUID` | 否 | 角色编号 |
| `code` | `VARCHAR(32)` | 否 | 三个固定角色之一 |
| `name` | `VARCHAR(64)` | 否 | 角色名称 |
| `description` | `TEXT` | 是 | 角色说明 |
| `created_at` | `TIMESTAMPTZ` | 否 | 创建时间 |

`user_role`：

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `id` | `UUID` | 否 | 关系编号 |
| `user_account_id` | `UUID` | 否 | 人工账号 |
| `role_id` | `UUID` | 否 | 平台角色 |
| `granted_by` | `UUID` | 是 | 授权人；初始化可空 |
| `created_at` | `TIMESTAMPTZ` | 否 | 授权时间 |
| `revoked_at` | `TIMESTAMPTZ` | 是 | 撤销时间 |
| `revoked_by` | `UUID` | 是 | 撤销人 |

约束：角色代码唯一并限制为 `PLATFORM_ADMIN/ACCOUNT_ADMIN/USER`；对 `(user_account_id, role_id)` 建未撤销部分唯一索引。角色关系不物理删除，平台角色不产生知识库正文访问权。

### 6.4 `login_session`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `id` | `UUID` | 否 | 同时作为访问凭证中的会话编号 |
| `user_account_id` | `UUID` | 否 | 所属人工账号 |
| `session_version` | `BIGINT` | 否 | 创建时捕获账号会话版本 |
| `credential_hmac` | `VARCHAR(64)` | 否 | Cookie 中高熵会话凭证的 HMAC-SHA256 校验值 |
| `hmac_key_id` | `VARCHAR(64)` | 否 | 会话校验密钥引用编号 |
| `csrf_token_hmac` | `VARCHAR(64)` | 否 | 会话绑定 CSRF Token 的校验值，明文不落库 |
| `issued_at` | `TIMESTAMPTZ` | 否 | 签发时间 |
| `expires_at` | `TIMESTAMPTZ` | 否 | 绝对失效时间 |
| `revoked_at` | `TIMESTAMPTZ` | 是 | 撤销时间 |
| `revoke_reason` | `VARCHAR(64)` | 是 | 退出、密码修改、禁用或强制下线 |
| `last_seen_at` | `TIMESTAMPTZ` | 是 | 最近使用时间 |

索引：`(user_account_id, revoked_at, expires_at)`；`UNIQUE(credential_hmac)`；`CHECK(expires_at > issued_at)`。访问有效性必须同时验证凭证、账号状态、账号 session_version、会话撤销和过期时间。会话凭证使用系统设计规定的安全 Cookie；MVP 不另建刷新令牌机制。

### 6.5 `password_reset_credential`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `user_account_id` | `UUID` | 否 | 目标人工账号 |
| `credential_hash` | `TEXT` | 否 | 临时密码的自适应哈希完整编码，含独立随机盐 |
| `hash_algorithm` | `VARCHAR(32)` | 否 | 与人工账号密码相同的自适应哈希算法版本 |
| `status` | `VARCHAR(16)` | 否 | `ACTIVE/USED/REVOKED/EXPIRED` |
| `expires_at` | `TIMESTAMPTZ` | 否 | 短期有效截止时间 |
| `used_at` | `TIMESTAMPTZ` | 是 | 首次成功使用时间 |
| `created_by` | `UUID` | 否 | 发起重置的管理员 |

约束：对 user_account_id 建 `WHERE status='ACTIVE'` 部分唯一索引；失效由 expires_at 和 status 共同判定，新建前在锁定账号的事务中撤销旧凭证。重置事务签发临时密码、设置 must_change_password=true、递增 session_version 并撤销旧会话，旧正式密码随即不再可用。临时密码首次验证成功时将凭证置 USED，只签发受 must_change_password 限制的改密会话；提交新密码时写 user_account.password_hash、清除强制改密标记并再次撤销此前会话。不得先消耗临时密码再要求用同一临时密码重新登录。完整临时密码只交付一次。

### 6.6 `api_account`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `tenant_id` | `UUID` | 否 | 固定租户 |
| `name` | `VARCHAR(128)` | 否 | 调用账号名称 |
| `purpose` | `TEXT` | 否 | 调用用途 |
| `owner_name` | `VARCHAR(128)` | 否 | 负责人 |
| `owner_contact` | `VARCHAR(320)` | 是 | 负责人联系方式 |
| `status` | `VARCHAR(16)` | 否 | `ACTIVE/DISABLED` |
| `status_version` | `BIGINT` | 否 | 禁用和恢复时递增，供鉴权复核 |
| `authorization_version` | `BIGINT` | 否 | 账号知识库授权变更时递增，默认 0 |
| `disabled_at` | `TIMESTAMPTZ` | 是 | 禁用时间 |
| `last_called_at` | `TIMESTAMPTZ` | 是 | 最近成功调用时间 |
| `created_by` | `UUID` | 否 | 创建人 |

约束：`UNIQUE(tenant_id, name)`；状态索引 `(tenant_id, status)`。该表不含密码且不能创建登录会话。

### 6.7 `api_key`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `api_account_id` | `UUID` | 否 | 所属 API 调用账号 |
| `key_prefix` | `VARCHAR(24)` | 否 | 用于快速定位和脱敏展示的非秘密前缀 |
| `key_hash` | `VARCHAR(64)` | 否 | API Key 的 HMAC-SHA256 校验值 |
| `hash_algorithm` | `VARCHAR(32)` | 否 | 固定 HMAC-SHA256 |
| `hmac_key_id` | `VARCHAR(64)` | 否 | 校验密钥版本编号，不是明文密钥 |
| `status_version` | `BIGINT` | 否 | 禁用、恢复或删除时递增 |
| `authorization_version` | `BIGINT` | 否 | 自身 Scope 变更时递增 |
| `name` | `VARCHAR(128)` | 否 | 密钥用途名称 |
| `status` | `VARCHAR(16)` | 否 | `ACTIVE/DISABLED/DELETED` |
| `last_used_at` | `TIMESTAMPTZ` | 是 | 最后成功鉴权使用时间 |
| `disabled_at` | `TIMESTAMPTZ` | 是 | 禁用时间 |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 删除时间 |
| `created_by` | `UUID` | 否 | 创建人 |

约束：`UNIQUE(key_prefix)`、`UNIQUE(hmac_key_id, key_hash)`；不设置 expires_at。HMAC 密钥轮换保留仍有有效 API Key 引用的旧校验密钥，通过 hmac_key_id 定位；不得因密钥托管轮换隐式使永久 API Key 失效。完整 Key 只在创建事务成功后返回一次，幂等重放只返回密钥编号及“明文已交付”标志，不重放秘密值。

### 6.8 `knowledge_permission`

一行表示一个主体的一项知识库权限。人工账号和 API 账号共用表，但通过两个可空外键保持真实引用。

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `knowledge_base_id` | `UUID` | 否 | 目标知识库 |
| `user_account_id` | `UUID` | 是 | 人工账号 |
| `api_account_id` | `UUID` | 是 | API 调用账号 |
| `permission_code` | `VARCHAR(16)` | 否 | `MANAGE/EDIT/RETRIEVE/EVALUATE` |
| `grant_version` | `BIGINT` | 否 | 该授权变更版本 |
| `granted_by` | `UUID` | 否 | 授权人 |
| `effective_at` | `TIMESTAMPTZ` | 否 | 生效时间 |
| `revoked_at` | `TIMESTAMPTZ` | 是 | 撤销时间；空表示当前有效 |
| `revoked_by` | `UUID` | 是 | 撤销人 |

关键约束：

- `CHECK ((user_account_id IS NOT NULL)::int + (api_account_id IS NOT NULL)::int = 1)`；
- API 调用账号记录必须为 `RETRIEVE`，通过 `CHECK(api_account_id IS NULL OR permission_code='RETRIEVE')` 保证；
- 分别对 `(knowledge_base_id,user_account_id,permission_code)` 和 `(knowledge_base_id,api_account_id,permission_code)` 建立主体非空且 revoked_at IS NULL 的部分唯一索引；
- 撤销不物理删除，用于审计和删除引用的历史授权判断。

### 6.9 `api_key_scope`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `api_key_id` | `UUID` | 否 | API Key |
| `knowledge_base_id` | `UUID` | 否 | 被允许知识库 |
| `created_by` | `UUID` | 否 | 配置人 |
| `revoked_at` | `TIMESTAMPTZ` | 是 | 撤销时间 |

对 `(api_key_id, knowledge_base_id)` 建未撤销部分唯一索引。写入时在同一事务中锁定 API Key 和所属 API 账号授权，禁止把 Scope 扩大到账号当前 `RETRIEVE` 授权之外；读取时仍取两者实时交集。

### 6.10 `knowledge_base`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `tenant_id` | `UUID` | 否 | 固定租户 |
| `name` | `VARCHAR(128)` | 否 | 全平台唯一名称 |
| `description` | `TEXT` | 是 | 知识库说明 |
| `default_language` | `VARCHAR(16)` | 否 | 默认 `zh-CN`，不破坏英文、数字和业务代码 |
| `status` | `VARCHAR(16)` | 否 | `DRAFT/ACTIVE/DISABLED/DELETING/DELETED` |
| `current_chunking_config_version_id` | `UUID` | 是 | 当前入库配置；启用前必须有有效配置 |
| `visibility_revision` | `BIGINT` | 否 | 文档/块启停及删除覆盖的全库代次，默认 0 |
| `active_metadata_schema_version_id` | `UUID` | 是 | 当前 Schema 版本 |
| `active_index_version_id` | `UUID` | 是 | 当前活动索引版本 |
| `created_by` | `UUID` | 否 | 创建人 |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 删除完成时间 |

约束：UNIQUE(tenant_id,name)；名称去首尾空白、不能为空，使用确定性比较。活动 Schema、分块配置和索引均属于本知识库，使用 7.2 的复合外键及事务状态校验。删除后名称不复用；创建知识库、初始配置和四项直接权限同事务完成，生产初始化不预置业务知识库。

### 6.11 `metadata_schema_version`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `knowledge_base_id` | `UUID` | 否 | 所属知识库 |
| `version_no` | `INTEGER` | 否 | 单库递增版本号 |
| `name` | `VARCHAR(128)` | 否 | Schema 名称 |
| `template_code` | `VARCHAR(64)` | 是 | 可选 `K12` 等业务模板标识 |
| `status` | `VARCHAR(16)` | 否 | `DRAFT/ACTIVE/RETIRED` |
| `schema_digest` | `VARCHAR(64)` | 否 | 规范化字段定义 SHA-256 |
| `created_by` | `UUID` | 否 | 创建人 |
| `activated_at` | `TIMESTAMPTZ` | 是 | 生效时间 |

约束：`UNIQUE(knowledge_base_id, version_no)`；每库最多一个 ACTIVE 的部分唯一索引。Schema 一旦被 READY/ACTIVE/RETIRED 快照引用，其字段定义全部冻结；字段停用也通过新 Schema 版本表达，历史版本保留原定义和值。KnowledgeBase 当前 Schema 供新入库使用，在线检索使用捕获 IndexVersion 的 Schema，不混用两者。

### 6.12 `metadata_field_definition`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `schema_version_id` | `UUID` | 否 | Schema 版本 |
| `field_name` | `VARCHAR(64)` | 否 | 字段名 |
| `display_name` | `VARCHAR(128)` | 否 | 展示名 |
| `data_type` | `VARCHAR(16)` | 否 | `STRING/NUMBER/BOOLEAN/DATE/DATETIME/STRING_ARRAY` |
| `required` | `BOOLEAN` | 否 | 是否必填 |
| `filterable` | `BOOLEAN` | 否 | 是否允许业务过滤 |
| `response_visible` | `BOOLEAN` | 否 | 是否允许返回调用方 |
| `chunk_overridable` | `BOOLEAN` | 否 | 内容块是否可覆盖文档值 |
| `status` | `VARCHAR(16)` | 否 | `ACTIVE/DISABLED` |
| `disabled_at` | `TIMESTAMPTZ` | 是 | 字段停用时间 |
| `validation_rule` | `JSONB` | 否 | 枚举、长度、数值范围等受控规则 |
| `sort_order` | `INTEGER` | 否 | 展示顺序 |

约束：`UNIQUE(schema_version_id, field_name)`；field_name 禁止使用平台保留名。已索引字段删除前在新 Schema 中标为 DISABLED；类型变化必须使用新字段名并重建，不得在新 Schema 中沿用同名字段静默改变类型。字段定义物理清除必须无保留版本或快照引用。

元数据值保存在 DocumentVersion 和 Chunk 的 JSONB 中，但不是无约束自由 JSON：写入前必须按对应 SchemaVersion 校验名称、类型、必填、枚举和长度。文档值默认复制到 Chunk，只有 `chunk_overridable=true` 的字段可以覆盖。K12 模板只预置需求确认的 `content_type`、`course_id`、`grade_code`、`subject_code`、`campus_id`、`term_code`、`class_type`、`valid_from`、`valid_to`、`sales_stage`、`channel`、`tags`；不得加入 `department_ids` 或 `security_level`。`valid_from/valid_to` 使用 UTC 时间，可空，且校验 `valid_to > valid_from`；检索采用左闭右开区间。

### 6.13 `document`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `knowledge_base_id` | `UUID` | 否 | 所属知识库 |
| `name` | `VARCHAR(512)` | 否 | 业务文档名 |
| `current_published_version_id` | `UUID` | 是 | 当前发布版本 |
| `created_by` | `UUID` | 否 | 创建人 |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 删除完成时间 |

同一知识库允许重名文档；列表索引为 `(knowledge_base_id, updated_at DESC)`。Document 不另设业务状态枚举，状态由 DocumentVersion 和删除墓碑决定。current_published_version_id 保存最近选定的发布版本，人工停用时保留以支持恢复，删除完成后清空；有 PUBLISHED 版本时必须指向该版本，仅指针非空不代表可检索。

### 6.14 `document_version`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `document_id` | `UUID` | 否 | 稳定文档身份 |
| `version_no` | `INTEGER` | 否 | 文档内递增版本号 |
| `source_version_id` | `UUID` | 是 | 从已发布版本复制草稿时的来源版本 |
| `change_type` | `VARCHAR(16)` | 否 | `UPLOAD/REPLACE/EDIT/REPROCESS/ROLLBACK`；REPROCESS 表示复用已校验原文件重新解析/分块，不表示上传替换文件 |
| `change_reason` | `TEXT` | 是 | 上传、编辑或回滚原因 |
| `original_filename` | `VARCHAR(512)` | 否 | 原始文件名，仅展示，不用于对象键 |
| `file_format` | `VARCHAR(16)` | 否 | 已确认支持格式 |
| `file_size_bytes` | `BIGINT` | 否 | 文件大小 |
| `file_sha256` | `VARCHAR(64)` | 否 | 文件摘要 |
| `mime_type` | `VARCHAR(255)` | 否 | 服务端识别 MIME |
| `status` | `VARCHAR(24)` | 否 | `UPLOADED/PROCESSING/REVIEW_REQUIRED/PUBLISHED/DISABLED/FAILED/DELETING/DELETED` |
| `disable_reason` | `VARCHAR(16)` | 是 | `MANUAL/SUPERSEDED` |
| `metadata_schema_version_id` | `UUID` | 否 | 校验本版本元数据的 Schema |
| `chunking_config_version_id` | `UUID` | 否 | 本次处理固定使用的分块配置 |
| `attachment_aggregate_status` | `VARCHAR(24)` | 是 | EML 为 PENDING/COMPLETE/PARTIAL_FAILED；非 EML 为空 |
| `metadata` | `JSONB` | 否 | 文档业务元数据 |
| `parser_version` | `VARCHAR(128)` | 是 | 解析器版本 |
| `cleaning_rule_version` | `VARCHAR(128)` | 是 | 清洗规则版本 |
| `ocr_model_version` | `VARCHAR(128)` | 是 | OCR 模型版本 |
| `quality_flags` | `JSONB` | 否 | OCR 低质量、隐藏工作表等告警 |
| `published_at` | `TIMESTAMPTZ` | 是 | 发布时间 |
| `disabled_at` | `TIMESTAMPTZ` | 是 | 停用时间 |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 删除完成时间 |
| `created_by` | `UUID` | 否 | 创建人 |

关键约束与索引：

- `UNIQUE(document_id, version_no)`；
- `REPROCESS` 必须具有同一 Document 的 `source_version_id`；沿该来源版本的同文档 source_version_id 链定位仍保留的 FORMAL ORIGINAL DocumentObject，链缺失、成环、跨文档或命中墓碑均拒绝。新版本不重复创建 ORIGINAL 对象键，来源原文件在新版本处理及保留期内不得被独立清理；
- `file_size_bytes > 0`，格式限制为 PDF、DOCX、XLSX、PPTX、TXT、MD、HTML、CSV、JSON、EML、PNG、JPG、JPEG；
- `disable_reason` 仅在 `DISABLED` 时非空；
- CHECK 明确 DISABLED 必须携带 MANUAL/SUPERSEDED，其他状态必须为空；进入 DELETING 时清空原因，历史原因由审计保留；
- `CREATE UNIQUE INDEX ... ON document_version(document_id) WHERE status='PUBLISHED'`；
- 状态索引 `(document_id, status)` 和 `(status, updated_at)`；
- 发布/回滚事务同步发布指针和索引引用；停用保留指针，只更新状态、disable_reason 和 visibility_revision；处理状态变更不修改活动索引。具体迁移见第 7、9 章。

### 6.15 `document_object`

仅保存七牛云对象引用，不保存 URL 或访问 Token。

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `document_version_id` | `UUID` | 否 | 所属文档版本 |
| `object_kind` | `VARCHAR(24)` | 否 | `ORIGINAL/PREVIEW`，与生命周期分离 |
| `bucket_code` | `VARCHAR(64)` | 否 | 配置中的 Bucket 逻辑编号，不保存密钥 |
| `staged_object_key` | `VARCHAR(1024)` | 否 | 平台生成、永久不复用的暂存键 |
| `object_key` | `VARCHAR(1024)` | 否 | 预先确定、永久不复用的正式键 |
| `upload_reservation_id` | `UUID` | 是 | 原文件对应上传预留；预览为空 |
| `content_sha256` | `VARCHAR(64)` | 否 | 内容摘要 |
| `qiniu_etag` | `VARCHAR(128)` | 否 | Kodo 上传/stat 的 hash 原编码，不等于 content_sha256 |
| `size_bytes` | `BIGINT` | 否 | 对象大小 |
| `storage_status` | `VARCHAR(24)` | 否 | `STAGED/FORMAL/REJECTED/DELETED` |
| `source_locator` | `JSONB` | 否 | 预览页码/幻灯片等定位；原文件为空对象 |
| `last_verified_at` | `TIMESTAMPTZ` | 是 | 最近存在性/摘要验证时间 |
| `operation_attempt_count` | `INTEGER` | 否 | 最终化、验证或删除累计尝试次数 |
| `delete_requested_at` | `TIMESTAMPTZ` | 是 | 删除请求时间 |
| `delete_verified_at` | `TIMESTAMPTZ` | 是 | Kodo 确认对象不存在的时间 |
| `last_error_code` | `VARCHAR(64)` | 是 | 稳定错误码 |

约束与处理规则：

- `UNIQUE(bucket_code, object_key)`、`UNIQUE(bucket_code, staged_object_key)`、`UNIQUE(upload_reservation_id)`；暂存与正式键前缀不重叠；
- 正式键由 `tenant_id/knowledge_base_id/document_id/document_version_id/object_kind` 等不可变标识确定，重复最终化写入同一键；
- 同一文档版本最多一个 ORIGINAL（按 object_kind 建部分唯一索引）；预览可按页或格式存在多条，定位反映在唯一对象键中。草稿复制时 source_version_id 追溯来源原文件，不复制或共享可被单独删除的对象键；读取来源链前复核墓碑，删除 Document 时覆盖其全部版本；
- 对象删除只有在 Kodo 返回成功或已不存在，并记录 `delete_verified_at` 后才能标记 `DELETED`；
- STAGED 重试同时探测暂存键和正式键，正式键存在且大小/hash 一致时确认转正，解决移动成功但数据库未提交的窗口；平台 SHA-256 在读取流中复核。REJECTED 仅在拒收暂存对象确认删除后写入，FORMAL 解析失败不删除原文件；
- 孤儿扫描必须同时确认超过安全窗口、无 DocumentObject 引用、无未到期 UploadReservation、无有效任务；本表有引用的 STAGED 不能按年龄自动清理；
- 签名 URL 按请求临时生成，不写本表、缓存持久层、追踪或审计。

### 6.16 `document_relation`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `parent_document_version_id` | `UUID` | 否 | EML 等父文档版本 |
| `child_document_id` | `UUID` | 否 | 附件形成的独立文档 |
| `relation_type` | `VARCHAR(24)` | 否 | MVP 固定为 `ATTACHMENT` |
| `source_locator` | `JSONB` | 否 | 附件名、序号等结构定位，不保存正文 |

约束：UNIQUE(parent_document_version_id,child_document_id,relation_type)；父子同库且不是同一 Document。锁定知识库后检查无环再写入。EML 聚合状态为：存在未结束附件任务则 PENDING，全部成功则 COMPLETE，全部结束且任一失败/取消则 PARTIAL_FAILED；不支持附件只记录告警。

### 6.17 `document_artifact`

解析产物明确存放在 PostgreSQL，不引入未确认的第二对象存储。考虑到单文件解压后上限可达 1 GB，正文不得作为单个超大 BYTEA 行写入，而采用“产物元数据 + 有序分段”保存。

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `document_version_id` | `UUID` | 否 | 所属文档版本 |
| `artifact_type` | `VARCHAR(24)` | 否 | `EXTRACTED_TEXT/CLEANED_TEXT/STRUCTURE` |
| `content_encoding` | `VARCHAR(32)` | 否 | 如 `utf-8` |
| `compression` | `VARCHAR(16)` | 否 | `NONE/GZIP/ZSTD`，以技术基线支持为准 |
| `content_sha256` | `VARCHAR(64)` | 否 | 解压后内容摘要 |
| `content_length` | `BIGINT` | 否 | 解压后的字节数 |
| `part_count` | `INTEGER` | 否 | 有序分段数量 |
| `write_status` | `VARCHAR(16)` | 否 | WRITING/READY/FAILED，只有 READY 可读取或索引 |
| `producer_version` | `VARCHAR(128)` | 否 | 解析器或清洗规则版本 |
| `is_deleted` | `BOOLEAN` | 否 | 显式删除后为 true |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 内容清除时间 |

约束：`UNIQUE(document_version_id, artifact_type, producer_version)`；`content_length >= 0`、`part_count >= 0`。读取必须同时满足 write_status=READY 且 is_deleted=false；`is_deleted=true` 时 deleted_at 非空且不得存在关联分段，长度、分段数和摘要仅保留为删除前的历史校验数据。正文读取只用于受控管理、重建和解析诊断，不参与外部接口直接返回。

### 6.18 `document_artifact_part`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `artifact_id` | `UUID` | 否 | 所属解析产物 |
| `part_no` | `INTEGER` | 否 | 从 0 开始的连续分段号 |
| `content_bytes` | `BYTEA` | 否 | 独立压缩的分段内容 |
| `uncompressed_offset` | `BIGINT` | 否 | 解压后起始字节偏移 |
| `uncompressed_length` | `INTEGER` | 否 | 解压后分段长度 |
| `part_sha256` | `VARCHAR(64)` | 否 | 解压后分段摘要 |

约束：`UNIQUE(artifact_id, part_no)`；偏移和长度均非负。应用按 `part_no` 流式写入和读取，分段大小由技术容量基线冻结并保持显著低于 PostgreSQL 单字段上限。完成写入后校验分段连续、累计长度、完整内容摘要和 `part_count`，通过后才允许任务进入下一阶段。显式删除时物理删除分段行，再保留不含正文的 Artifact 元数据。

### 6.19 `chunk`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `document_version_id` | `UUID` | 否 | 所属文档版本 |
| `chunk_no` | `INTEGER` | 否 | 版本内稳定顺序号 |
| `parent_chunk_id` | `UUID` | 是 | 父子分块中的父块 |
| `chunk_type` | `VARCHAR(24)` | 否 | `GENERAL/SECTION/QA/TABLE/PARENT/CHILD` |
| `content` | `TEXT` | 是 | 可检索正文；显式删除后清空 |
| `content_sha256` | `VARCHAR(64)` | 否 | 内容摘要 |
| `title_path` | `JSONB` | 否 | 标题路径字符串数组 |
| `source_locator` | `JSONB` | 否 | 页码、幻灯片、工作表、边界框等 |
| `metadata` | `JSONB` | 否 | 继承并校验后的业务元数据 |
| `manual_keywords` | `JSONB` | 否 | 人工补充关键词数组 |
| `token_count` | `INTEGER` | 否 | 冻结 tokenizer 下的 Token 数 |
| `chunker_version` | `VARCHAR(128)` | 否 | 分块器和规则版本 |
| `tokenizer_version` | `VARCHAR(128)` | 否 | Token 计数版本 |
| `enabled` | `BOOLEAN` | 否 | PostgreSQL 权威可用状态 |
| `target_enabled` | `BOOLEAN` | 否 | 最近一次请求的目标状态 |
| `visibility_revision` | `BIGINT` | 否 | 本块最近一次启停捕获的全库代次 |
| `enabled_changed_at` | `TIMESTAMPTZ` | 否 | 权威状态生效时间 |
| `enable_sync_status` | `VARCHAR(16)` | 否 | `SYNCED/ENABLING/DISABLING/FAILED` |
| `edited_from_chunk_id` | `UUID` | 是 | 新草稿块的来源块 |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 显式删除清除时间 |

约束与索引：

- `UNIQUE(document_version_id, chunk_no)`；
- 父块属于同一 document_version_id，由 7.2 复合外键保证；CHILD 必须引用 PARENT，PARENT 的 parent_chunk_id 为空，不允许多级循环；
- `token_count >= 0`；未删除块 `content IS NOT NULL`，已删除块 `content IS NULL`；
- `(document_version_id, enabled)`、`(parent_chunk_id)` 和 GIN `metadata jsonb_path_ops` 索引；
- READY 快照引用的正文、关键词、定位和业务元数据均不可原地更新；编辑复制新版本。运营更新仅限 enabled、target_enabled、visibility_revision、enabled_changed_at 和 enable_sync_status；显式删除是清除敏感内容的例外，按第 12 章执行。

### 6.20 `citation`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `id` | `UUID` | 否 | 不可猜测稳定 `citation_id` |
| `knowledge_base_id` | `UUID` | 否 | 来源知识库 |
| `document_id` | `UUID` | 否 | 来源文档稳定编号 |
| `document_version_id` | `UUID` | 是 | 来源版本；物理清理后可置空 |
| `chunk_id` | `UUID` | 是 | 来源块；物理清理后可置空 |
| `source_version_no` | `INTEGER` | 否 | 保留的原版本号 |
| `source_document_name` | `VARCHAR(512)` | 否 | 允许保留的来源名称 |
| `source_locator` | `JSONB` | 否 | 最小来源定位，不含正文 |
| `status` | `VARCHAR(16)` | 否 | `CURRENT/HISTORICAL/DELETED` |

约束：`CREATE UNIQUE INDEX uq_citation_chunk ON citation(chunk_id) WHERE chunk_id IS NOT NULL;`。所有来源 ID 必须沿同一知识库/文档/版本链；status 为展示投影，不能替代实时状态和墓碑复核，返回状态映射为 CURRENT/HISTORICAL_UNAVAILABLE/SOURCE_DELETED。历史或删除引用不保存正文和预览地址；删除引用授权使用 6.41 的历史交集及当前撤权信息。

### 6.21 `ingestion_task`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `document_version_id` | `UUID` | 否 | 处理目标 |
| `task_type` | `VARCHAR(24)` | 否 | MVP 为 `INGEST`，评测使用独立表 |
| `status` | `VARCHAR(24)` | 否 | `PENDING/RUNNING/SUCCEEDED/RETRY_WAIT/FAILED/CANCEL_REQUESTED/CANCELLED/CLEANUP_PENDING/CLEANUP_RETRY_WAIT/CLEANUP_FAILED` |
| `current_stage` | `VARCHAR(32)` | 否 | 03 的十一阶段：VALIDATE/FINALIZE/PARSE/CLEAN/CHUNK/PREVIEW/EMBED/VECTOR_WRITE/KEYWORD_WRITE/VERIFY/FINISH；取消清理为 CLEANUP |
| `index_version_id` | `UUID` | 是 | 本次任务构建的索引版本，创建后关联 |
| `progress_percent` | `SMALLINT` | 否 | 0～100 |
| `parameter_snapshot` | `JSONB` | 否 | 分块、Schema、解析和模型参数快照 |
| `idempotency_key` | `VARCHAR(128)` | 否 | 文档版本与处理配置的幂等键 |
| `attempt_count` | `INTEGER` | 否 | 已开始执行次数 |
| `lease_owner` | `VARCHAR(128)` | 是 | 当前 Worker 实例 |
| `execution_generation` | `BIGINT` | 否 | 每次领取递增的 fencing 代次，默认 0 |
| `lease_expires_at` | `TIMESTAMPTZ` | 是 | 租约截止时间 |
| `heartbeat_at` | `TIMESTAMPTZ` | 是 | 最近心跳 |
| `next_retry_at` | `TIMESTAMPTZ` | 是 | 下次可重试时间 |
| `last_error_code` | `VARCHAR(64)` | 是 | 稳定错误码 |
| `last_error_summary` | `TEXT` | 是 | 脱敏错误摘要 |
| `started_at` | `TIMESTAMPTZ` | 是 | 首次开始时间 |
| `finished_at` | `TIMESTAMPTZ` | 是 | 最终结束时间 |
| `created_by` | `UUID` | 否 | 发起人 |

索引与约束：

- `UNIQUE(idempotency_key)`；
- 对同一文档版本在 PENDING/RUNNING/RETRY_WAIT/CANCEL_REQUESTED/CLEANUP_PENDING/CLEANUP_RETRY_WAIT/CLEANUP_FAILED 状态建立部分唯一索引；
- 调度索引 `(status, next_retry_at)`、租约索引 `(status, lease_expires_at)`；
- RUNNING 必须具有租约持有者、到期时间和心跳；CANCEL_REQUESTED 可保留停止前的运行租约，CLEANUP_PENDING 可由清理执行持租约。其余状态清空租约；三项租约字段必须同时为空或同时非空；
- `SUCCEEDED` 只表示产物校验完成并进入 `REVIEW_REQUIRED`，不表示已发布。

### 6.22 `task_execution`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `ingestion_task_id` | `UUID` | 否 | 所属任务 |
| `execution_no` | `INTEGER` | 否 | 任务内递增执行次数 |
| `worker_id` | `VARCHAR(128)` | 是 | QUEUED 时为空，领取后填写 |
| `execution_type` | `VARCHAR(16)` | 否 | `INGEST/CLEANUP` |
| `execution_generation` | `BIGINT` | 是 | 领取时绑定任务代次，QUEUED 时为空 |
| `status` | `VARCHAR(16)` | 否 | `QUEUED/RUNNING/SUCCEEDED/FAILED/CANCELLED/LOST` |
| `started_at` | `TIMESTAMPTZ` | 是 | 领取后填写 |
| `heartbeat_at` | `TIMESTAMPTZ` | 是 | 最近心跳 |
| `finished_at` | `TIMESTAMPTZ` | 是 | 结束时间 |
| `stage_timings` | `JSONB` | 否 | 各阶段耗时，不含正文 |
| `error_code` | `VARCHAR(64)` | 是 | 稳定错误码 |
| `error_summary` | `TEXT` | 是 | 脱敏摘要 |

约束：`UNIQUE(ingestion_task_id, execution_no)`；同一任务最多一条 QUEUED/RUNNING 执行记录。调度事务先创建 QUEUED，领取更新同一条，不重复创建；RUNNING 要求 worker_id、started_at、execution_generation 非空；终态要求 finished_at 非空。终态记录不可修改。

### 6.23 `outbox_event`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `aggregate_type` | `VARCHAR(64)` | 否 | 聚合类型 |
| `aggregate_id` | `UUID` | 否 | 聚合编号 |
| `event_type` | `VARCHAR(64)` | 否 | 事件类型 |
| `business_key` | `VARCHAR(160)` | 否 | 业务幂等键 |
| `payload` | `JSONB` | 否 | 仅包含编号、版本和事件参数，不含正文、Key 或 Token |
| `status` | `VARCHAR(16)` | 否 | `PENDING/DELIVERED/RETRY/DEAD` |
| `attempt_count` | `INTEGER` | 否 | 投递次数 |
| `available_at` | `TIMESTAMPTZ` | 否 | 可投递时间 |
| `claimed_by` | `VARCHAR(128)` | 是 | 投递器实例 |
| `claimed_until` | `TIMESTAMPTZ` | 是 | 投递租约 |
| `delivered_at` | `TIMESTAMPTZ` | 是 | Broker 接受时间 |
| `last_error_code` | `VARCHAR(64)` | 是 | 稳定错误码 |

约束：`UNIQUE(event_type, business_key)`；投递扫描索引 `(status, available_at)`。投递器在短事务中用 `FOR UPDATE SKIP LOCKED` 领取到期且无有效租约的 PENDING/RETRY 行，写入 claimed_by/claimed_until 后提交；事务外投递，回写时比较领取时的 lock_version，防止旧投递器覆盖新租约。Broker 接受成功后标记 DELIVERED，消费者仍必须幂等。失败进入 RETRY，耗尽重试进入 DEAD 并告警；受控重投将原行改为 RETRY，保留 business_key 和业务任务编号，不另建业务任务。DELIVERED 但业务任务未被领取时，由持久化任务状态触发同编号补投；领取与回写均清晰区分投递成功和业务执行成功。

### 6.24 `idempotency_record`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `actor_type` | `VARCHAR(16)` | 否 | `USER/API_ACCOUNT/SYSTEM` |
| `actor_id` | `UUID` | 否 | 操作主体 |
| `operation` | `VARCHAR(64)` | 否 | 发布、删除、授权等操作 |
| `idempotency_key` | `VARCHAR(128)` | 否 | 客户端或服务端幂等键 |
| `request_fingerprint` | `VARCHAR(64)` | 否 | 白名单请求的 HMAC-SHA256，不把秘密字段生成可离线猜测的裸摘要 |
| `hmac_key_id` | `VARCHAR(64)` | 否 | 幂等指纹校验密钥版本 |
| `status` | `VARCHAR(16)` | 否 | `PROCESSING/SUCCEEDED/FAILED` |
| `response_status` | `INTEGER` | 是 | 首次响应状态码 |
| `response_summary` | `JSONB` | 是 | 不含敏感正文的首次响应摘要 |
| `expires_at` | `TIMESTAMPTZ` | 是 | 仅控制幂等复用窗口，不代表业务数据删除 |

约束：`UNIQUE(actor_type, actor_id, operation, idempotency_key)`。相同 Key 但请求指纹不同返回冲突，不得执行第二次写操作。

PROCESSING 记录与业务变更使用同一事务；成功时一起写 SUCCEEDED 和不含秘密的结果引用。事务失败全部回滚，不留下永久 PROCESSING。异步操作在接受时即记 SUCCEEDED、响应保存 task_id，实际执行结果由任务表承担；轮询或重复消息不重新执行业务。到期仅关闭复用窗口，不能靠删除幂等记录允许旧请求重复删除或重复发布。

### 6.25 `embedding_model_fingerprint`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `model_name` | `VARCHAR(128)` | 否 | 固定 `BAAI/bge-m3` |
| `model_version` | `VARCHAR(128)` | 否 | 模型版本 |
| `model_file_sha256` | `VARCHAR(64)` | 否 | 模型文件摘要 |
| `vector_dimension` | `INTEGER` | 否 | 固定 1024 |
| `normalized` | `BOOLEAN` | 否 | 固定 true |
| `distance_metric` | `VARCHAR(16)` | 否 | 固定 `COSINE` |
| `query_prefix_rule` | `TEXT` | 否 | 查询前缀规则 |
| `document_prefix_rule` | `TEXT` | 否 | 文档前缀规则 |
| `text_normalization_version` | `VARCHAR(64)` | 否 | 模型输入规范化规则，参与指纹及产物键计算 |
| `fingerprint_sha256` | `VARCHAR(64)` | 否 | 全部规范化字段摘要 |
| `created_at` | `TIMESTAMPTZ` | 否 | 创建时间 |

约束：`UNIQUE(fingerprint_sha256)`；`CHECK(vector_dimension=1024 AND normalized=true AND distance_metric='COSINE')`。记录一旦被索引版本引用即不可修改。

### 6.26 `index_version`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `knowledge_base_id` | `UUID` | 否 | 所属知识库 |
| `version_no` | `INTEGER` | 否 | 单库递增版本号 |
| `status` | `VARCHAR(16)` | 否 | `BUILDING/READY/ACTIVE/RETIRED/FAILED/DELETED` |
| `base_active_index_version_id` | `UUID` | 是 | 构建时捕获的活动版本 |
| `base_visibility_revision` | `BIGINT` | 否 | 构建时捕获的全库可见性代次 |
| `validated_visibility_revision` | `BIGINT` | 是 | 最近双索引验证的代次，发布时必须等于权威值 |
| `publishable` | `BOOLEAN` | 否 | 默认 true，过期 READY 设 false 后不可再次发布 |
| `metadata_schema_version_id` | `UUID` | 否 | 冻结 Schema |
| `embedding_fingerprint_id` | `UUID` | 否 | 冻结 Embedding 空间 |
| `chunking_config_version_id` | `UUID` | 否 | 构建时知识库当前分块配置外键；不表示快照全部文档均按此配置重切，各文档实际配置由 DocumentVersion 记录 |
| `index_schema_version` | `VARCHAR(64)` | 否 | ES/pgvector 映射版本 |
| `snapshot_digest` | `VARCHAR(64)` | 是 | BUILDING 未完成时为空，READY 前必须计算 |
| `member_count` | `BIGINT` | 否 | 快照块数 |
| `es_index_name` | `VARCHAR(255)` | 否 | ES 物理索引名 |
| `validation_status` | `VARCHAR(16)` | 否 | `PENDING/PASSED/FAILED` |
| `validation_evidence` | `JSONB` | 否 | 数量、维度、抽样查询和校验摘要 |
| `built_at` | `TIMESTAMPTZ` | 是 | 构建完成时间 |
| `activated_at` | `TIMESTAMPTZ` | 是 | 激活时间 |
| `retired_at` | `TIMESTAMPTZ` | 是 | 退役时间 |
| `created_by` | `UUID` | 否 | 创建人 |

约束与索引：

- `UNIQUE(knowledge_base_id, version_no)`、`UNIQUE(es_index_name)`；
- 每个知识库最多一个 `ACTIVE` 的部分唯一索引；
- `READY/ACTIVE` 必须 `validation_status='PASSED'` 且成员数、快照摘要非空；
- 发布时锁定知识库行并验证 `base_active_index_version_id` 仍等于当前活动版本；
- 活动切换和旧索引退役必须在同一短事务中完成；仅快照包含文档内容版本变更时，文档版本切换也在该事务中完成。纯 Embedding/Schema/索引修复重建不改变 DocumentVersion 发布状态；事务内不调用 Elasticsearch。

### 6.27 `index_snapshot_member`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `index_version_id` | `UUID` | 否 | 索引版本 |
| `chunk_id` | `UUID` | 否 | 不可变内容块 |
| `document_id` | `UUID` | 否 | 冗余过滤字段 |
| `document_version_id` | `UUID` | 否 | 冗余版本字段 |
| `content_sha256` | `VARCHAR(64)` | 否 | 加入快照时的内容摘要 |
| `metadata_digest` | `VARCHAR(64)` | 否 | 加入快照时的元数据摘要 |
| `embedding_artifact_id` | `UUID` | 是 | 参与向量召回的块必须有产物；仅作上下文的父块可空 |
| `retrievable` | `BOOLEAN` | 否 | CHILD/普通块为 true，纯上下文 PARENT 为 false |
| `chunk_enabled` | `BOOLEAN` | 否 | 可见性投影，不纳入内容摘要 |
| `document_status` | `VARCHAR(24)` | 否 | PUBLISHED/DISABLED/DELETING/DELETED 检索投影 |
| `visibility_revision` | `BIGINT` | 否 | 本成员已同步的可见性代次 |

约束：`UNIQUE(index_version_id, chunk_id)`；索引 `(index_version_id, document_id)`。不可变字段必须与 Chunk 来源一致，READY 后不能改写；正文和元数据通过固定 Chunk 外键取得，不再复制第二份正文。可见性三字段独立更新，不纳入 snapshot_digest。READY/ACTIVE 中 retrievable=true 的成员必须有同模型指纹的已验证产物；隐藏成员仍保留。父块必须作为同快照成员存在，避免子块命中后取得快照之外的父块。

未激活快照的 document_status 表示预期发布后的投影：待发布目标可预置 PUBLISHED，但绝不因此修改 DocumentVersion 的权威状态或允许在线访问。候选快照只有验证全体成员与所用 Schema、模型和索引映射兼容后才能 READY；成员保留其来源 DocumentVersion 的分块配置，IndexVersion 的分块配置记录本次构建基线，不宣称继承成员已被重新切分。

### 6.28 `embedding_artifact`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `tenant_id` | `UUID` | 否 | 固定租户，复用安全边界 |
| `normalized_text_sha256` | `VARCHAR(64)` | 否 | 模型实际输入的规范化文本摘要，不保存正文副本 |
| `embedding_fingerprint_id` | `UUID` | 否 | 向量空间指纹 |
| `embedding` | `VECTOR(1024)` | 否 | L2 归一化向量 |
| `embedding_sha256` | `VARCHAR(64)` | 否 | 标准化向量摘要 |
| `verified_at` | `TIMESTAMPTZ` | 否 | 维度、有限数值和归一化校验通过时间 |
| `created_at` | `TIMESTAMPTZ` | 否 | 创建时间 |

约束：`UNIQUE(tenant_id, normalized_text_sha256, embedding_fingerprint_id)`；向量必须为 1024 维、非零、有限且按模型基线完成 L2 归一化。规范化规则属于模型指纹；摘要使用固定编码的 float32 向量字节计算。产物校验完成后一次性插入，正文、模型或向量内容均不可原地更新；只有不存在任何快照引用时才可清理。检索通过 index_snapshot_member 联接 embedding_artifact 和 chunk 施加快照、权限、状态及业务过滤，不在共享产物上保存单一文档状态。索引策略见 8.3。

### 6.29 `retrieval_parameter_set`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `knowledge_base_id` | `UUID` | 是 | 空表示跨库测试参数集 |
| `name` | `VARCHAR(128)` | 否 | 参数集名称 |
| `version_no` | `INTEGER` | 否 | 同名参数版本 |
| `parameters` | `JSONB` | 否 | 需求定义的模式、候选数、RRF、Rerank、阈值、top_n、预算 |
| `parameter_digest` | `VARCHAR(64)` | 否 | 规范化参数摘要 |
| `status` | `VARCHAR(16)` | 否 | `DRAFT/FROZEN/RETIRED` |
| `created_by` | `UUID` | 否 | 创建人 |

约束：知识库非空时 UNIQUE(knowledge_base_id,name,version_no)，空时 UNIQUE(name,version_no) WHERE knowledge_base_id IS NULL；FROZEN 后参数及摘要不可修改。参数固定校验模式 KEYWORD/VECTOR/HYBRID（默认 HYBRID）、双路 candidate_k=1～200（默认50）、rrf_k=1～200（默认60，仅 HYBRID）、rerank 默认 true、rerank_threshold=0～1（默认0.50）、top_n=1～50（默认10）、max_context_tokens=1000～32000（默认8000）。不适用参数记录但不执行；参数集只供测试评测使用。

### 6.30 `retrieval_trace`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `id` | `UUID` | 否 | `trace_id` |
| `tenant_id` | `UUID` | 否 | 固定租户 |
| `request_id` | `VARCHAR(128)` | 否 | 调用方请求编号 |
| `protocol` | `VARCHAR(16)` | 否 | `HTTP/MCP/UI/EVALUATION` |
| `api_account_id` | `UUID` | 是 | API/MCP 调用主体 |
| `api_key_id` | `UUID` | 是 | 使用的密钥编号 |
| `user_account_id` | `UUID` | 是 | 管理端检索测试主体 |
| `query_hmac` | `VARCHAR(64)` | 否 | HMAC-SHA256 指纹 |
| `hmac_key_id` | `VARCHAR(64)` | 否 | 指纹密钥编号，不是密钥 |
| `normalization_version` | `VARCHAR(32)` | 否 | 查询规范化规则版本 |
| `query_char_count` | `INTEGER` | 否 | 查询字符数 |
| `knowledge_base_ids` | `JSONB` | 否 | 有序去重后的知识库 UUID 数组 |
| `document_ids` | `JSONB` | 否 | 可选文档范围 UUID 数组 |
| `effective_parameters` | `JSONB` | 否 | 实际生效参数及模型/tokenizer 版本；启用 Rerank 时记录已捕获配置编号、模型版本和文件校验值 |
| `requested_parameters` | `JSONB` | 否 | 请求参数白名单，不含 query 正文 |
| `index_versions` | `JSONB` | 否 | 每库 knowledge_base_id、index_version_id、Schema、Embedding 指纹及 visibility_revision |
| `effective_fusion` | `JSONB` | 否 | 实际融合算法、常量与版本；VECTOR 内部 RRF 不使用调用方 rrf_k |
| `snapshot_captured_at` | `TIMESTAMPTZ` | 否 | 请求捕获快照的数据库时间 |
| `stage_timings_ms` | `JSONB` | 否 | 阶段耗时 |
| `hit_summaries` | `JSONB` | 否 | 块编号、排名和得分，不含正文 |
| `degraded` | `BOOLEAN` | 否 | 是否降级 |
| `degraded_stage` | `VARCHAR(32)` | 是 | 降级阶段 |
| `degraded_reason` | `VARCHAR(64)` | 是 | 稳定原因码 |
| `result_status` | `VARCHAR(24)` | 否 | `SUCCESS/EMPTY/ERROR/CANCELLED` |
| `error_code` | `VARCHAR(64)` | 是 | 稳定错误码 |
| `total_duration_ms` | `INTEGER` | 否 | 总耗时 |
| `created_at` | `TIMESTAMPTZ` | 否 | 调用时间 |

索引：`(api_account_id, request_id, created_at DESC)`、`(user_account_id, request_id, created_at DESC)`、`(created_at DESC)`、`(result_status, created_at DESC)` 及 `knowledge_base_ids` GIN。`request_id` 用于关联和诊断，不设全局唯一，调用方重试可产生新的 `trace_id`。原始查询、完整正文、完整向量和签名 URL 不得进入任何 JSONB 字段。

成功鉴权的调用主体必须二选一：user_account_id 非空且两个 API 字段均空；或 api_account_id、api_key_id 同时非空且 user_account_id 为空，密钥必须属于该账号。认证失败在安全审计记录，不伪造 RetrievalTrace 主体。未提供 request_id 时由入口生成。JSONB 的每库上下文使用固定白名单；只记录已获授权的资源，未完成授权的输入编号不进入命中摘要。

### 6.31 评测表

#### `evaluation_set`

保存 knowledge_base_id(UUID 外键)、name(VARCHAR(128))、description(TEXT，可空)、status(VARCHAR(16)，ACTIVE/ARCHIVED)、created_by(UUID 外键)，除 description 外非空；UNIQUE(knowledge_base_id,name)。

#### `evaluation_set_version`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `evaluation_set_id` | `UUID` | 否 | 评测集 |
| `version_no` | `INTEGER` | 否 | 递增版本 |
| `status` | `VARCHAR(16)` | 否 | `DRAFT/FROZEN/RETIRED` |
| `corpus_digest` | `VARCHAR(64)` | 是 | 验收语料和版本摘要 |
| `annotation_version` | `VARCHAR(64)` | 否 | 相关性标注版本 |
| `case_count` | `INTEGER` | 否 | 用例数 |
| `frozen_at` | `TIMESTAMPTZ` | 是 | 冻结时间 |
| `created_by` | `UUID` | 否 | 创建人 |

约束：UNIQUE(evaluation_set_id,version_no)；FROZEN 后禁止常规增删改用例；显式删除或安全处置清除问题内容时按 12.1 例外处理，版本同步转 RETIRED，不再发起新运行，历史结果保留为当时证据。

#### `evaluation_case`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `evaluation_set_version_id` | `UUID` | 否 | 评测版本 |
| `case_no` | `INTEGER` | 否 | 版本内序号 |
| `query_text` | `TEXT` | 是 | 有效用例必填经批准脱敏的问题；清除时置空 |
| `redacted_at` | `TIMESTAMPTZ` | 是 | 显式清除时间，非空即不可运行 |
| `expected_targets` | `JSONB` | 否 | 期望文档/块编号及可选相关等级 |
| `metadata_filter` | `JSONB` | 否 | 可选业务过滤条件 |
| `notes` | `TEXT` | 是 | 备注 |

约束：UNIQUE(evaluation_set_version_id,case_no)；query_text 与 redacted_at 必须恰有一个非空。expected_targets 固定为目标类型 DOCUMENT/CHUNK、编号、相关等级和粒度，冻结时验证同库；不保存答案正文。访问受 EVALUATE 控制。

#### `evaluation_run`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `evaluation_set_version_id` | `UUID` | 否 | 冻结评测版本 |
| `index_version_id` | `UUID` | 否 | 批量运行启动时捕获的活动快照；管理单条 READY 调试不创建本表记录 |
| `parameter_set_id` | `UUID` | 否 | 冻结参数集 |
| `embedding_fingerprint_id` | `UUID` | 否 | Embedding 指纹 |
| `rerank_config_id` | `UUID` | 是 | 启用 Rerank 时所评测的运行配置外键；可引用已核验、尚未激活的候选配置 |
| `rerank_model_version` | `VARCHAR(128)` | 是 | 启用 Rerank 时冻结的模型版本 |
| `rerank_model_file_sha256` | `VARCHAR(64)` | 是 | 启用 Rerank 时冻结的模型文件校验值 |
| `tokenizer_version` | `VARCHAR(128)` | 否 | tokenizer 版本 |
| `code_version` | `VARCHAR(128)` | 否 | 执行代码版本 |
| `lease_owner` | `VARCHAR(128)` | 是 | 运行 Worker，未领取时为空 |
| `lease_expires_at` | `TIMESTAMPTZ` | 是 | 执行租约截止 |
| `execution_generation` | `BIGINT` | 否 | 领取代次，默认 0 |
| `run_fingerprint` | `VARCHAR(64)` | 否 | 所有冻结输入的摘要 |
| `status` | `VARCHAR(16)` | 否 | `PENDING/RUNNING/SUCCEEDED/FAILED/CANCELLED` |
| `metrics` | `JSONB` | 否 | Recall、Precision、MRR、nDCG、空结果率 |
| `permission_case_pass_rate` | `NUMERIC(6,5)` | 是 | 权限用例通过率 |
| `started_at` | `TIMESTAMPTZ` | 是 | 开始时间 |
| `finished_at` | `TIMESTAMPTZ` | 是 | 结束时间 |
| `created_by` | `UUID` | 否 | 发起人 |

索引：run_fingerprint 普通索引。运行只引用同一知识库的 FROZEN 评测集版本、FROZEN 参数集、已验证 IndexVersion，且 Embedding 指纹一致；参数集可为该库或无库归属的通用配置。Rerank 开启时三个 Rerank 字段均非空，配置为固定模型且版本/文件校验值必须一致；关闭时三个字段均为空。候选配置的模型文件须先完成部署校验，评测运行可引用 DRAFT 配置；配置一经评测引用，模型身份、endpoint_ref 和 runtime_parameters 不可更新，变更须创建新配置版本并重新评测。启动后冻结字段不可修改。创建运行和 Outbox 同事务，条件领取租约并递增代次，结果写入校验代次；租约过期记 FAILED，重新运行创建新 evaluation_run 保留旧结果。同输入可重复运行；SUCCEEDED 要求每个冻结用例都有结果且没有执行错误，错误不能算作零分用例掩盖。

#### `evaluation_case_result`

保存 evaluation_run_id(UUID)、evaluation_case_id(UUID)、status(VARCHAR(16)，SUCCESS/EMPTY/ERROR)、returned_targets(JSONB 数组)、metric_values(JSONB 对象)、trace_id(UUID，可空)、error_code(VARCHAR(64)，可空)、duration_ms(BIGINT 非负)，其余字段非空。`UNIQUE(evaluation_run_id,evaluation_case_id)`；用例必须属于运行引用的评测集版本。returned_targets 只保存编号、排名、得分和相关性判断，不复制正文。

#### `release_gate_record`

保存 evaluation_run_id(UUID 外键)、gate_type(VARCHAR(32)，固定 INITIAL_PRODUCTION)、status(VARCHAR(16)，PASSED/FAILED)、thresholds(JSONB)、evidence(JSONB)、approved_by(UUID 外键 user_account)、approved_at(TIMESTAMPTZ)、evidence_ref(TEXT)，均非空。PASSED 要求运行成功、脱敏批准语料、不少于 100 个用例、默认混合参数集、Recall@10 ≥ 0.80、MRR@10 ≥ 0.60、权限用例通过率=1；跨表证据在受控事务中复核。记录追加写，不允许把历史 FAILED 改成 PASSED。

### 6.32 `model_runtime_config`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `model_type` | `VARCHAR(16)` | 否 | `EMBEDDING/RERANK/OCR` |
| `config_version` | `INTEGER` | 否 | 同模型指纹内递增配置版本 |
| `embedding_fingerprint_id` | `UUID` | 是 | EMBEDDING 必填外键，其他类型为空 |
| `model_name` | `VARCHAR(128)` | 否 | 只读模型名 |
| `model_version` | `VARCHAR(128)` | 否 | 只读模型版本 |
| `model_file_sha256` | `VARCHAR(64)` | 否 | 文件摘要 |
| `endpoint_ref` | `VARCHAR(255)` | 否 | 配置中心引用或内部服务标识，不含凭证 |
| `runtime_parameters` | `JSONB` | 否 | 超时、并发、批次和计算设备 |
| `status` | `VARCHAR(16)` | 否 | `DRAFT/ACTIVE/RETIRED` |
| `created_by` | `UUID` | 否 | 配置人 |
| `activated_at` | `TIMESTAMPTZ` | 是 | 生效时间 |

EMBEDDING 对 (embedding_fingerprint_id,config_version) 建部分唯一索引；其他类型对 (model_type,model_name,model_version,model_file_sha256,config_version) 建部分唯一索引，两类谓词分别为 model_type='EMBEDDING' 和 model_type<>'EMBEDDING'。EMBEDDING 的 ACTIVE 唯一索引以 embedding_fingerprint_id 为键；RERANK 的 ACTIVE 唯一索引以固定 model_type 为键，保证全平台至多一个供新请求选择的配置；OCR 的 ACTIVE 唯一索引按模型名称、版本及文件摘要建键。EMBEDDING 的名称、版本和文件摘要必须与所引指纹一致；RERANK 的 model_name 固定为 `BAAI/bge-reranker-v2-m3`，配置一经评测引用，其模型身份、endpoint_ref 和 runtime_parameters 均不可改写。每个 Embedding 指纹最多一个 ACTIVE 运行配置，升级期间可同时服务不同指纹的新旧索引。外部新请求按捕获 IndexVersion 的 Embedding 指纹定位运行配置；启用 Rerank 时捕获唯一 ACTIVE 的 Rerank 配置编号及模型身份。Rerank 候选配置经文件核验和最小推理，记录同环境新旧版本 P50/P95/P99 耗时，并通过同一冻结验收知识库、IndexVersion、默认混合参数集和不少于 100 题的评测，达到 Recall@10≥0.80、MRR@10≥0.60、权限泄露用例 100% 通过后，切换事务复核所引用的 SUCCEEDED EvaluationRun 与候选配置一致，先把旧 ACTIVE 改 RETIRED，再将候选 DRAFT 改 ACTIVE；失败整体回滚，旧配置保留至在途请求和评测结束，回滚沿同一互斥路径执行。OCR 仅记录部署基线，不增加需求外 OCR 管理功能；旧 Embedding 配置在被活动/保留快照引用期间不可移除。

### 6.33 `security_policy`

保存版本化登录安全策略：version_no(INTEGER 唯一)、password_min_length(INTEGER 默认 8)、login_failure_threshold(INTEGER)、lock_duration_seconds(INTEGER)、session_ttl_seconds(INTEGER)、status(VARCHAR(16)，DRAFT/ACTIVE/RETIRED)、created_by(UUID 外键)、activated_at(TIMESTAMPTZ，生效前可空)。数值均为正整数，除 activated_at 外非空；最多一个 ACTIVE。策略修改创建新版本，已生效数值不覆盖；不保存密码、盐或密钥。

### 6.34 `audit_log`

追加写记录，字段包括：

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `id` | `UUID` | 否 | 审计编号 |
| `tenant_id` | `UUID` | 否 | 固定租户 |
| `actor_type` | `VARCHAR(16)` | 否 | `USER/API_ACCOUNT/SYSTEM` |
| `actor_id` | `UUID` | 是 | 系统任务可空 |
| `action` | `VARCHAR(64)` | 否 | 登录、授权、发布、删除、查看敏感正文等 |
| `resource_type` | `VARCHAR(64)` | 否 | 资源类型 |
| `resource_id` | `UUID` | 是 | 资源编号 |
| `knowledge_base_id` | `UUID` | 是 | 知识库相关操作的历史归属编号，不设外键；非知识库操作为空 |
| `result` | `VARCHAR(16)` | 否 | `SUCCESS/DENIED/FAILED` |
| `request_id` | `VARCHAR(128)` | 是 | 请求关联编号 |
| `trace_id` | `UUID` | 是 | 检索关联编号 |
| `before_summary` | `JSONB` | 否 | 脱敏变更前摘要 |
| `after_summary` | `JSONB` | 否 | 脱敏变更后摘要 |
| `client_context` | `JSONB` | 否 | 脱敏 IP、User-Agent 等 |
| `created_at` | `TIMESTAMPTZ` | 否 | 发生时间 |
| `previous_integrity_hash` | `VARCHAR(64)` | 是 | 同一审计链前一条摘要；链首可空 |
| `chain_sequence` | `BIGINT` | 否 | 固定租户内严格递增的提交链序号 |
| `integrity_hash` | `VARCHAR(64)` | 否 | 当前记录防篡改摘要 |

索引：`(resource_type, resource_id, created_at DESC)`、`(knowledge_base_id, created_at DESC)`、`(actor_type, actor_id, created_at DESC)`、`(action, created_at DESC)`。生产业务账号对审计表只授予 INSERT/SELECT，不授予 UPDATE/DELETE；链摘要定期校验并输出受控证据。禁止记录密码、API Key、Token、签名 URL、密钥、完整正文和原始检索问题。

`UNIQUE(tenant_id,chain_sequence)`；写审计时取得固定租户的事务级审计锁，读取已提交链尾，按固定字段顺序与编码计算摘要并插入，再随业务事务提交，避免并发分叉。正文编辑的前后值由不可变文档版本保存，审计只保存版本/块引用、摘要、操作者和时间；正文删除后不可借审计恢复内容。

### 6.35 `deletion_tombstone`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `tenant_id` | `UUID` | 否 | 固定租户 |
| `resource_type` | `VARCHAR(32)` | 否 | `KNOWLEDGE_BASE/DOCUMENT`，不增加独立版本删除入口 |
| `resource_id` | `UUID` | 否 | 被删除资源 |
| `deletion_version` | `BIGINT` | 否 | 同资源递增删除序号 |
| `deleted_at` | `TIMESTAMPTZ` | 否 | 逻辑删除时间 |
| `deleted_by` | `UUID` | 否 | 发起人 |
| `reason_code` | `VARCHAR(64)` | 否 | 删除原因码 |
| `cleanup_status` | `VARCHAR(24)` | 否 | `PENDING/RUNNING/VERIFIED/FAILED` |
| `cleanup_summary` | `JSONB` | 否 | 各存储清理状态、数量和摘要，不含正文 |
| `ledger_status` | `VARCHAR(16)` | 否 | `PENDING/SYNCED/FAILED` |
| `ledger_sequence` | `BIGINT` | 是 | 独立追加写账本序号 |
| `ledger_synced_at` | `TIMESTAMPTZ` | 是 | 账本同步时间 |
| `verified_at` | `TIMESTAMPTZ` | 是 | 全部清理复核完成时间 |

约束：`UNIQUE(resource_type, resource_id, deletion_version)`；恢复环境开放流量前，必须把恢复点之后的账本序号重放到 PostgreSQL，并再次清理对象、正文和派生索引。

### 6.36 `backup_recovery_point`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `tenant_id` | `UUID` | 否 | 固定租户 |
| `recovery_point_code` | `VARCHAR(128)` | 否 | 恢复点编号 |
| `status` | `VARCHAR(24)` | 否 | `CREATING/READY/FAILED/EXPIRED`；演练通过保持 READY |
| `postgres_backup_ref` | `TEXT` | 是 | 创建中可空，READY 必填 |
| `postgres_wal_lsn` | `PG_LSN` | 是 | 数据库一致性水位，READY 必填 |
| `object_manifest_ref` | `TEXT` | 是 | 含每个对象备份副本引用的清单，READY 必填 |
| `object_manifest_sha256` | `VARCHAR(64)` | 是 | 清单摘要，READY 必填 |
| `config_manifest_ref` | `TEXT` | 是 | 索引/运行配置备份引用，READY 必填 |
| `key_custody_ref` | `TEXT` | 是 | 独立密钥托管恢复引用，READY 必填，不保存明文材料 |
| `object_count` | `BIGINT` | 否 | 清单对象数 |
| `tombstone_ledger_sequence` | `BIGINT` | 否 | 恢复点已包含的墓碑账本水位 |
| `index_backup_ref` | `TEXT` | 是 | 为满足 RTO 保留的派生索引备份引用 |
| `created_at` | `TIMESTAMPTZ` | 否 | 创建时间 |
| `verified_at` | `TIMESTAMPTZ` | 是 | 恢复演练验证时间 |
| `verification_summary` | `JSONB` | 否 | 恢复验证摘要和各阶段耗时 |

约束：`UNIQUE(recovery_point_code)`。只有 PostgreSQL 备份/WAL、对象副本与清单、墓碑水位、配置及独立密钥托管引用均固化并验证后才能进入 READY；季度演练只更新 verified_at 和验证记录，不改变 READY 的可恢复含义。清单包含 WAL 截止点所有被引用的 STAGED/FORMAL 原文件及预览，并记录暂存/正式键、实际副本、大小、平台 SHA-256 和 Kodo hash；不能仅备份键清单。恢复点清单同时保存在数据库备份之外，数据库故障时仍能找到它。

### 6.37 `upload_reservation`

包含通用字段及 user_account_id(UUID 外键)、knowledge_base_id(UUID 外键)、bucket_code(VARCHAR(64))、staged_object_key(VARCHAR(1024))、formal_object_key(VARCHAR(1024))、expires_at(TIMESTAMPTZ)、status(VARCHAR(16)，RESERVED/CONSUMED/EXPIRED)，以上非空；consumed_at(TIMESTAMPTZ，可空)。两个对象键分别 UNIQUE(bucket_code,键)，前缀不重叠。预留先于对象写入落库；消费预留与创建 DocumentObject、版本、任务和 Outbox 同事务。过期回收与消费锁同一预留行，持有效预留或已消费为对象引用时禁止孤儿回收。

### 6.38 `chunking_config_version`

包含通用字段及 knowledge_base_id(UUID 外键)、version_no(INTEGER 正整数)、strategy(VARCHAR(16)，GENERAL/SECTION/QA/TABLE/PARENT_CHILD)、parameters(JSONB 对象)、config_digest(VARCHAR(64))、created_by(UUID 外键)，均非空；UNIQUE(knowledge_base_id,version_no)。parameters 固定包含目标/最大 Token 数、重叠长度、分隔符及父子策略参数，校验目标不超过最大值、重叠非负且小于目标值。创建后内容不可更新；KnowledgeBase 保存当前配置引用，DocumentVersion 和 IndexVersion 保存各自冻结引用。

### 6.39 `deletion_task`

包含通用字段及 tombstone_id(UUID UNIQUE 外键)、status(VARCHAR(16)，PENDING/RUNNING/RETRY_WAIT/SUCCEEDED/FAILED)、current_stage(VARCHAR(32)，LEDGER/SUCCESSOR_SNAPSHOT/CONTENT_CLEANUP/INDEX_CACHE_CLEANUP/VERIFY)、progress_percent(SMALLINT 0～100)、attempt_count(INTEGER 非负)、execution_generation(BIGINT 非负)，均非空。lease_owner(VARCHAR(128))、lease_expires_at/heartbeat_at/next_retry_at/finished_at(TIMESTAMPTZ)、last_error_code(VARCHAR(64))、successor_index_version_id(UUID 外键) 可空。删除目标由唯一墓碑确定，重复删除复用未完成任务，不另建并行清理。

RUNNING 必须具备有效租约，非 RUNNING 清空租约；状态允许 PENDING→RUNNING→SUCCEEDED/RETRY_WAIT/FAILED、RETRY_WAIT→PENDING、FAILED→PENDING（人工重试）。租约丢失按 RUNNING→RETRY_WAIT/FAILED 恢复；SUCCEEDED 无后继。索引为 (status,next_retry_at)、(status,lease_expires_at)。墓碑 ledger_sequence 确认、关联入库执行停止、必要后继快照切换及全量清理复核是 SUCCEEDED 的前置条件。

### 6.40 `deletion_execution`

包含通用字段及 deletion_task_id(UUID 外键)、execution_no(INTEGER 正整数)、status(VARCHAR(16)，QUEUED/RUNNING/SUCCEEDED/FAILED/LOST)，均非空；worker_id(VARCHAR(128))、execution_generation(BIGINT)、started_at/finished_at(TIMESTAMPTZ)、error_code(VARCHAR(64)) 可空，stage_results(JSONB 对象，非空) 只存编号、进度和校验摘要。UNIQUE(deletion_task_id,execution_no)，每任务最多一条 QUEUED/RUNNING。调度时建 QUEUED，Worker 领取时绑定租约代次并改 RUNNING；RUNNING 要求 worker_id、execution_generation、started_at 非空，所有终态要求 finished_at 非空且不可覆盖。

### 6.41 `deleted_access_grant`

删除知识库前，在删除事务中保存最小历史授权交集；本表不是新的在线授权来源。包含通用字段、tombstone_id(UUID 外键)、knowledge_base_id(UUID 外键)、permission_id(UUID 外键 knowledge_permission)、permission_grant_version(BIGINT)，这些字段非空；user_account_id/api_account_id/api_key_id(UUID 外键)、api_key_scope_id(UUID 外键)、scope_lock_version(BIGINT)、revoked_at(TIMESTAMPTZ) 可空，按下述互斥约束填写。

人工记录仅 user_account_id 非空，API 记录必须 api_account_id 和 api_key_id 同时非空且 user_account_id 为空；人工记录的 scope 引用和版本为空，API 记录必须同时具备 scope 引用和版本。permission_id 必须是本主体、本知识库在删除时有效的 RETRIEVE 授权；API Key 必须属于所记 API 账号，Scope 必须属于该 Key 和知识库且删除时有效。分别建立 (tombstone_id,user_account_id) 和 (tombstone_id,api_key_id) 主体非空部分唯一索引。历史关系不物理删除；知识库删除本身不伪造人工撤权，在线访问由墓碑阻断，后续显式撤权同时更新本表 revoked_at。读取时检查当前账号/Key/会话有效性和撤权，已知 citation_id 才可返回最小删除状态。

## 7. 状态、唯一性与引用完整性

### 7.1 核心部分唯一索引

以下约束必须由 Alembic 明确创建并由迁移测试验证：

~~~sql
CREATE UNIQUE INDEX uq_document_one_published_version
ON document_version (document_id)
WHERE status = 'PUBLISHED';

CREATE UNIQUE INDEX uq_kb_one_active_index
ON index_version (knowledge_base_id)
WHERE status = 'ACTIVE';

CREATE UNIQUE INDEX uq_kb_one_active_schema
ON metadata_schema_version (knowledge_base_id)
WHERE status = 'ACTIVE';

CREATE UNIQUE INDEX uq_document_one_live_ingestion
ON ingestion_task (document_version_id)
WHERE status IN ('PENDING', 'RUNNING', 'RETRY_WAIT', 'CANCEL_REQUESTED',
                 'CLEANUP_PENDING', 'CLEANUP_RETRY_WAIT', 'CLEANUP_FAILED');

CREATE UNIQUE INDEX uq_task_one_running_execution
ON task_execution (ingestion_task_id)
WHERE status IN ('QUEUED', 'RUNNING');
~~~

人工账号授权和 API 账号授权分别建立未撤销部分唯一索引，避免空值导致唯一约束失效。

### 7.2 外键删除策略

| 关系 | 删除策略 | 原因 |
| --- | --- | --- |
| 租户到核心业务表 | `RESTRICT` | MVP 固定租户不可删除 |
| 知识库到文档、权限、索引 | `RESTRICT` | 必须走受控删除和墓碑流程 |
| 文档到文档版本 | `RESTRICT` | 保留版本和审计关系 |
| 文档版本到任务、块、对象 | `RESTRICT` | 删除前显式清理，禁止级联误删 |
| 评测集版本到用例和运行 | `RESTRICT` | 冻结证据不可被级联删除 |
| Citation 到版本/块 | 可空外键、ON DELETE SET NULL | 删除正文后仍保留最小来源状态 |
| 审计/追踪中的历史资源编号 | 不设业务外键，按历史快照保留 | 业务清理不能破坏历史日志 |
| 快照成员到 Chunk/EmbeddingArtifact | RESTRICT | 先清理失效快照引用，后回收无引用产物 |
| 删除任务、执行、历史授权到墓碑 | RESTRICT | 删除和恢复证据不可级联删除 |

业务删除禁止依赖无条件 `ON DELETE CASCADE`。级联只可用于尚未发布且无审计价值的纯从属临时数据，并需在迁移评审中单独证明。

为防止“引用存在但归属错误”，以下关系使用复合候选键和复合外键，而不是只引用目标 `id`：

- `document(id, current_published_version_id)` 引用 `document_version(document_id, id)`；
- `knowledge_base(id, active_index_version_id)` 引用 `index_version(knowledge_base_id, id)`；
- `knowledge_base(id, active_metadata_schema_version_id)` 引用 `metadata_schema_version(knowledge_base_id, id)`；
- `knowledge_base(id, current_chunking_config_version_id)` 引用 `chunking_config_version(knowledge_base_id, id)`；
- `chunk(document_version_id, parent_chunk_id)` 引用 `chunk(document_version_id, id)`。

上述目标列组合分别建立显式 UNIQUE 候选键，循环外键在基础表创建后补充，固定使用 DEFERRABLE INITIALLY DEFERRED；它们校验归属，不证明目标处于发布状态。source_version_id/edited_from_chunk_id 只能引用同一 Document 的更早版本；快照的 Schema、分块配置、基础索引必须属于同一知识库。快照成员的 document_id/document_version_id/chunk_id 必须沿同一归属链，产物模型指纹必须等于 IndexVersion 指纹。这些无法仅靠已有列复合外键表达的约束在受控写入事务中锁定聚合、校验后写入，并由一致性巡检复核。

### 7.3 状态迁移执行规则

- 状态字段不可通过通用更新接口修改；
- Service 在事务内执行 `UPDATE ... WHERE id=? AND status=? AND lock_version=?`；受影响行数为 0 时返回 409；
- 文档发布锁定知识库，验证 READY 索引及基线版本后同时更新 DocumentVersion、Document、IndexVersion 和 KnowledgeBase；
- 内容块停用先提交 `chunk.enabled=false` 和 Outbox；启用时保持 false，双索引确认后再置 true；
- 初始删除事务写入墓碑、DeletionTask 和 Outbox；知识库同时进入 DELETING，单文档立即由墓碑屏蔽。DocumentVersion 随关联入库执行停止后按合法迁移进入 DELETING；UPLOADED 版本须先取消未领取任务、阻止新领取或写入并确认无有效租约，暂存/正式原文件纳入清理。全部清理复核后才进入 DELETED，任务为 SUCCEEDED、墓碑为 VERIFIED。

核心迁移严格采用 01 第 8.3 节和 03 第 6 章：KnowledgeBase 为 DRAFT→ACTIVE、ACTIVE↔DISABLED、DRAFT/ACTIVE/DISABLED→DELETING→DELETED；DocumentVersion 为 UPLOADED→PROCESSING/DELETING、PROCESSING→REVIEW_REQUIRED/FAILED、FAILED→PROCESSING/DELETING、REVIEW_REQUIRED→PUBLISHED/DELETING、PUBLISHED→DISABLED/DELETING、DISABLED→PUBLISHED/DELETING、DELETING→DELETED；IndexVersion 为 BUILDING→READY/FAILED、FAILED→BUILDING/DELETED、READY→ACTIVE/DELETED、ACTIVE→RETIRED、RETIRED→ACTIVE/DELETED。未列出的迁移拒绝。可见性同步和墓碑变更不伪造文档状态迁移。

## 8. 索引与访问路径

### 8.1 必要关系索引

| 主要查询 | 索引 |
| --- | --- |
| 邮箱登录 | `user_account(tenant_id, normalized_email)` 唯一 |
| API Key 鉴权 | `api_key(key_prefix)` 唯一，定位后验证摘要 |
| 账号有效授权 | 两个主体维度的未撤销部分唯一索引 |
| 知识库文档列表 | `document(knowledge_base_id, updated_at DESC)`，联查版本状态与墓碑 |
| 文档版本列表 | `document_version(document_id, version_no DESC)` |
| 入库调度 | `ingestion_task(status, next_retry_at)` |
| 租约恢复 | `ingestion_task(status, lease_expires_at)` |
| Outbox 投递 | `outbox_event(status, available_at)` |
| 活动索引读取 | `index_version(knowledge_base_id)` 的 ACTIVE 部分唯一索引 |
| 快照成员读取 | `(index_version_id,chunk_id)` 唯一；`(embedding_artifact_id)` 用于反向引用与回收；`(index_version_id,document_status,chunk_enabled)` 用于过滤 |
| 检索追踪筛选 | 按账号、状态、时间建立组合索引 |
| 审计查询 | 按资源/主体与时间建立组合索引 |
| 删除恢复 | `deletion_tombstone(ledger_status, deleted_at)` |

### 8.2 JSONB 索引原则

- `chunk.metadata` 使用 GIN 支持受控元数据过滤；
- `retrieval_trace.knowledge_base_ids` 可使用 GIN 支持管理端按知识库过滤；
- 低频展示 JSONB 不建通用 GIN，避免写放大；
- 只对已在 Metadata Schema 中标记 `filterable=true` 且经真实基线证明必要的高频字段增加表达式索引；
- 不允许调用方传入原始 SQL/JSONPath，过滤必须由受控 AST 编译并使用绑定参数。

### 8.3 pgvector 索引原则

MVP 使用 embedding_artifact 的 VECTOR(1024)。正确性基准为先按快照成员、业务元数据和实时权限过滤，再执行精确余弦距离排序；共享向量的一次命中必须展开为全部获授权的成员，不能把同文本不同来源静默合并。只有性能基线证明需要时才增加 embedding 上的余弦 HNSW 索引，并验证过滤后召回率、候选不足和多模型指纹隔离。ANN 过滤可能发生在候选扫描后，不能把“已有过滤条件”写成足量召回保证；候选不足时扩大受控扫描或回退精确过滤排序。向量检索必须：

1. 使用活动 IndexVersion 对应的 Embedding 指纹生成查询向量；
2. 强制限定租户、知识库、活动索引版本、文档范围和 `chunk_enabled=true`；
3. 取候选后回查 PostgreSQL 的知识库、文档版本和 Chunk 权威状态；
4. 不把 ANN 结果当作权限或发布状态事实。

### 8.4 分区策略

当前文档量、追踪量和审计量尚无确认数值，MVP 不预先分区，避免增加运维复杂度。上线基线形成后，如单表规模、写入速率或维护窗口证明需要，再对 `retrieval_trace`、`audit_log` 等追加写大表按时间进行在线迁移；不允许因分区自动删除需求规定长期保留的数据。

## 9. 事务、并发与一致性

### 9.1 上传与任务创建

七牛云暂存写入无法加入 PostgreSQL 事务，采用以下顺序：

1. API 完成入口校验并先提交 UploadReservation，记录暂存键、正式键和有效截止时间；
2. 流式写入 Kodo 暂存键，计算平台 SHA-256，另存 Kodo hash 与大小；
3. 锁定并消费有效预留，在一个 PostgreSQL 事务中创建 Document/DocumentVersion、document_object(STAGED)、IngestionTask、QUEUED TaskExecution 和 OutboxEvent；
4. 提交后由投递器发送 `task_id`；
5. Worker 取得任务租约后执行深度文件特征和安全校验，再以确定性正式键完成对象最终化；
6. 事务失败时预留仍保护已上传对象；回收需超过最大上传时间加提交重试窗口，且无对象引用、有效预留和有效任务。转正崩溃恢复检查暂存/正式双键，数据库提交不能被对象移动成功代替。

对象最终化和重试以 `(bucket_code, object_key)` 唯一约束、内容摘要和 Kodo ETag 共同判定幂等。

重处理路径不创建 UploadReservation 或新的 ORIGINAL DocumentObject。事务内锁定当前来源版本，沿同文档 source_version_id 链定位并校验未被墓碑排除的 FORMAL ORIGINAL，创建 `REPROCESS/UPLOADED` DocumentVersion、IngestionTask、QUEUED TaskExecution 和 OutboxEvent；Worker 在 VALIDATE 阶段复核已解析来源对象键、大小、Kodo hash 和平台 SHA-256，跳过仅针对 STAGED 新上传对象的 FINALIZE，随后执行解析、分块及索引。来源链引用在新版本处理和保留期间阻止独立清理；整篇文档删除仍先以墓碑阻断并停止关联任务，再清理全部版本和对象。

### 9.2 发布事务

发布事务只执行 PostgreSQL 操作，不包含 Elasticsearch 或七牛云网络调用：

1. 确认 IndexVersion 为 READY、校验证据 PASSED；
2. 锁定 `knowledge_base` 行；
3. 使用 `IS NOT DISTINCT FROM` 比较可空 base_active_index_version_id 与当前活动版本，复核 publishable、validated_visibility_revision、知识库 ACTIVE、适用的目标文档状态和删除墓碑；同步不足时退出事务、完成双索引投影同步后重新校验；
4. 仅当快照发布包含文档内容版本变更时，先将旧 PUBLISHED 版本改为 DISABLED/SUPERSEDED，再将新版本改为 PUBLISHED，避免立即生效的部分唯一索引冲突；纯 Embedding/Schema/索引修复重建跳过本步；
5. 仅当文档内容版本发生变更时更新 Document 当前发布版本；
6. 新快照使用尚未在线生效的 Embedding 指纹时，先复核对应 DRAFT 运行配置已通过本地核验，并在同一事务使该指纹配置 ACTIVE；旧指纹 ACTIVE 配置保留给在途请求和保留快照。随后先将旧 ACTIVE 索引改为 RETIRED，再将新索引改为 ACTIVE；任一失败整体回滚。KnowledgeBase 当前 Schema 引用由 Schema 激活操作独立切换，索引发布不将其回退；
7. 更新 KnowledgeBase 活动索引引用并写 AuditLog、OutboxEvent；
8. 提交。并发基线已变化时整体回滚并返回 409。

### 9.3 任务租约

调度事务创建且仅创建一条 QUEUED TaskExecution 与 Outbox。Worker 锁定任务条件领取，递增 execution_generation、绑定 lease_owner/lease_expires_at/heartbeat_at，将任务及既有执行改 RUNNING；文档只在 UPLOADED/FAILED 时转 PROCESSING，重试中已有 PROCESSING 则保持原状态。

所有阶段提交和产物晋级必须同时校验任务编号、租约持有者、未过期租约与 execution_generation；旧 Worker 的远端残留写入不能被确认为有效产物。Scheduler 锁定过期任务，把旧执行记为 LOST、失效租约，再按 RUNNING→RETRY_WAIT/FAILED 收尾；重新调度遵循 RETRY_WAIT→PENDING，不能跳过状态矩阵。

取消遵循 03 的 CLEANUP_PENDING/CLEANUP_RETRY_WAIT/CLEANUP_FAILED 路径，清理执行另建 QUEUED、execution_type=CLEANUP 的记录。清理未完成不能把 IngestionTask 置 CANCELLED；未运行且文档仍为 UPLOADED 的取消保留 UPLOADED，已运行的 PROCESSING 取消后进入 FAILED，原 FAILED 保持不变。索引校验成功时 IndexVersion READY、DocumentVersion REVIEW_REQUIRED、任务/执行 SUCCEEDED 同事务提交；重试 FAILED 索引先合法恢复 BUILDING，不能 FAILED 直接变 READY。

### 9.4 权限变更

授权/角色/Scope 变更在同一事务中递增对应主体 authorization_version、写审计及缓存失效 Outbox；禁用/撤销更新 status_version 或 session_version。请求入口权威复核版本，缓存版本不一致或失效事件尚未送达时回源；撤权同步撤销 deleted_access_grant，不能因删除知识库后无当前授权行而漏撤历史访问。API Key 可用知识库集合为：

`当前有效 api_key_scope ∩ 当前有效 api_account RETRIEVE knowledge_permission`。

### 9.5 内容块启停

- 两类操作锁定知识库、递增 visibility_revision，将该代次写入 Chunk 和 Outbox；事件只更新对应块的投影，不改变快照不可变字段。
- 停用先设置 enabled=false、target_enabled=false、DISABLING；末端复核立即阻断。启用先设置 enabled=false、target_enabled=true、ENABLING，双后端同步成功后仅在该块目标代次仍匹配时提交 enabled=true、SYNCED。
- 旧事件不得覆盖该块更新代次；不能仅因另一块使全库代次增大就永久丢弃本块事件。Scheduler 根据持久化目标状态补偿；发布校验全库代次时必须验证所有相关块投影已收敛。失败保持目标状态和 FAILED 以供补偿。

### 9.6 快照保留、恢复与回滚

后继快照包含 MANUAL DISABLED 文档和停用块的隐藏成员，恢复只更新可见性；缺成员时必须建新 IndexVersion。单文档回滚以当前活动快照构建新快照，仅替换目标文档；RETIRED→ACTIVE 只用于显式整库回滚，事务同步全部受影响文档指针和合法状态，禁止重启墓碑内容。

请求记录捕获时刻与每库快照；仅因捕获后发布变为 SUPERSEDED 的版本允许完成在途请求，人工停用、撤权和删除实时拒绝。旧索引回收同时满足回滚保留期和最大请求时限加安全窗口；未完成评测运行引用的版本同样不得物理清理。过期 READY 设 publishable=false，清理后转 DELETED，重建创建新 BUILDING。

SUPERSEDED 不将保留快照成员的检索投影改为人工停用；在途请求按捕获快照检索，再以 PostgreSQL 当前状态和停用原因复核。人工停用、块停用和删除则必须作用于全部仍可被请求使用的快照，不能仅修改当前活动索引。

## 10. Elasticsearch 与 Redis 数据映射

### 10.1 Elasticsearch 文档

每个 ES 索引文档至少包含 tenant_id、knowledge_base_id、index_version_id、document_id、document_version_id、chunk_id、chunk_enabled、document_status、visibility_revision、正文、人工关键词、标题路径、来源定位、允许过滤的业务元数据和内容摘要。ES `_id` 使用稳定的 index_version_id:chunk_id 派生值。正文等不可变数据对应固定快照，状态字段是可更新投影。

ES 不保存账号、API Key、密码、审计事实或唯一发布状态。ES 索引丢失时从 IndexSnapshotMember、Chunk 和活动模型/Schema 指纹重建。

### 10.2 Redis 键边界

Redis Broker 只承载含 `task_id` 的 Celery 消息；Redis Cache 只保存可回源的权限、配置和短期辅助数据；限流键以 API Key 编号和 API Account 编号为主体。两者使用独立逻辑库或键前缀。任何 Redis 数据丢失不得造成授权扩大、任务永久丢失或发布状态倒退。

## 11. 安全与隐私

- 数据库连接强制 TLS，并使用 API、Worker、Scheduler、只读观测等不同最小权限账号；
- 生产应用账号不拥有建库、安装扩展、修改迁移历史或绕过审计的权限；
- 密码只保存自适应哈希，API Key 只保存带独立密钥版本的 HMAC-SHA256；
- 数据库不保存七牛云密钥、模型密钥、HMAC 明文密钥、签名 URL 和上传/下载 Token；
- RetrievalTrace 不保存原始查询，AuditLog 不保存完整正文；
- JSONB 写入前执行字段白名单和敏感信息检查；
- 数据库备份、WAL、日志和临时文件启用静态加密；密钥材料与备份分开管理；
- 查看敏感正文必须经过额外授权，并产生 AuditLog；
- 运维查询和数据导出使用只读受控账号，导出评测片段和正文必须审计。

## 12. 删除、保留与恢复

### 12.1 显式删除

删除流程先使资源对新检索不可见，再异步清理：

- PostgreSQL：物理删除 DocumentArtifactPart，标记 Artifact 已清理；清空 Chunk 的 content、title_path、source_locator、metadata、manual_keywords 等可承载正文的字段（JSONB 置空集合），清空 DocumentVersion 的业务 metadata、quality_flags 和可含正文的变更说明；保留允许的最小标识与来源信息。清理所有失效快照成员引用，再物理删除无引用的 EmbeddingArtifact，不以仅“失效向量”冒充物理删除；
- Elasticsearch：删除对应活动和保留索引中的文档并验证查询不可见；
- 七牛云：删除原文件和预览，对每个 DocumentObject 执行不存在性确认；
- Redis：失效权限、配置和查询辅助缓存；
- Citation/RetrievalTrace/AuditLog：只保留允许的最小标识、版本、得分、耗时和操作结果，不保留可恢复正文。

开始不可逆清理前必须 ledger_status=SYNCED 且有账本序号，停止关联入库写入，并按 03 完成必要的后继快照切换。UPLOADED 版本可经 DELETING 清理；其关联预留、暂存/正式原文件和孤儿对象均须按引用与键核验清理。清理覆盖所有目标版本、保留快照、预览和归档副本；共享向量若仍被未删除文档合法引用则保留，但目标文档的所有关联必须移除。FAILED/READY 索引清理后转 DELETED，ACTIVE 先 RETIRED 后 DELETED；从不直接 ACTIVE→DELETED。

DeletionTask、DeletionExecution 记录阶段和失败，只有墓碑 cleanup_status=VERIFIED 且账本已同步，才在同一事务写任务 SUCCEEDED、全部目标 DocumentVersion/KnowledgeBase DELETED 和 document.deleted_at。失败保持屏蔽、重试并告警。任务 JSONB、评测用例备注等也需按数据白名单检查，不能成为被删正文副本；冻结评测问题若包含需删除内容则保留用例编号并受控清除正文、标记失效，禁止再作为有效验收语料。

### 12.2 数据保留

- 原文件、检索追踪和审计日志不按时间自动过期；
- 显式删除、法规删除和安全事件处置优先于长期保留；
- 历史文档版本可为回滚保留，但对外不得返回旧正文或预览；
- 墓碑账本保留期不得短于最长业务备份生命周期加恢复缓冲期；
- IdempotencyRecord 的复用窗口可到期，但其对应业务结果和 AuditLog 不随之删除。
- 分层归档保留业务编号、可定位清单、当前权限校验和完整性摘要，读路径仍经业务服务；归档副本属于显式删除清理范围，不得用归档保留已删除正文。

### 12.3 备份恢复一致性

一个可用恢复点固定 PostgreSQL WAL/备份、对象副本及清单、墓碑水位、配置和密钥托管引用，只有 READY 可选。恢复期间关闭业务流量，数据库与对象必须恢复到同一清单所对应的截止点。顺序为：

1. 恢复墓碑账本和密钥访问能力；
2. 恢复 PostgreSQL 到指定恢复点；
3. 按同一恢复点清单恢复并核对全部 STAGED/FORMAL 原文件、预览副本；
4. 重放该恢复点水位之后的删除墓碑，并恢复处理数据库内尚未完成的墓碑任务，重新清理恢复出的正文、对象和所有派生关联；
5. 重建或验证 pgvector 与 Elasticsearch 派生索引；
6. 验证账号、授权、API Key 状态、文档状态和活动索引版本；
7. 执行抽样检索并确认 RPO ≤ 24 小时、RTO ≤ 8 小时后开放流量。

## 13. 数据库迁移与回滚

### 13.1 迁移原则

- 所有结构变化通过 Alembic 管理，禁止生产手工改表后补迁移；
- 每个迁移具有唯一 revision、明确前置版本、升级说明和回滚说明；
- 破坏性变更采用“扩展—迁移—切换—收缩”，兼容窗口内新旧应用均可运行；
- 大表增加非空列时先增加可空列或默认值，回填验证后再收紧约束；
- 创建大索引优先使用 PostgreSQL 支持的在线方式，并在迁移外受控执行不支持事务的步骤；
- 向量维度、Embedding 指纹或索引结构变化不得原地改写活动数据，必须建立新 IndexVersion；
- 迁移前完成备份和恢复可用性检查，失败时回退应用和数据库兼容版本。

### 13.2 数据库验收检查

上线前至少验证：

1. UUIDv7 格式和时间排序特性；
2. 规范化邮箱唯一、三个固定角色和固定单租户；
3. API Key 明文不落库、范围交集和禁用即时生效；
4. 同一文档最多一个 PUBLISHED 版本；
5. 同一知识库最多一个 ACTIVE IndexVersion；
6. 并发发布只有一个成功，过期快照返回 409；
7. 重复 Outbox/Celery 消息不产生重复可见块；
8. Worker 租约过期可以恢复且历史执行不覆盖；
9. 内容块停用立即不可返回，启用在双索引确认后才生效；
10. 历史引用和删除引用不返回正文；
11. RetrievalTrace 无原始查询，日志与数据库无 Key、Token 和签名 URL；
12. 七牛云暂存最终化、重复删除和不存在确认均幂等；
13. 删除墓碑重放后已删除内容不会因恢复重新上线；
14. pgvector 与 ES 可从 PostgreSQL 和对象数据重建；
15. 代表性数据量下完成备份恢复并满足 RPO/RTO。

还须覆盖本次修订的数据库约束：上传预留消费与回收互斥；跨知识库复合外键拒绝错误归属；共享向量不丢失多来源成员；旧可见性事件和过期执行代次不可回写；UPLOADED 删除停止入库写入并清理对象；Rerank 候选配置可冻结评测、唯一 ACTIVE 配置在事务内切换且旧配置支持在途请求；删除后历史授权可撤销；评测问题清除后不能再次运行该失效版本。以上均为待执行验收项，不是本次文档审核的运行结果。

## 14. 需求与设计追溯

| 数据库设计内容 | 上游依据 |
| --- | --- |
| 固定租户、人工账号、角色、会话、API 账号、API Key | FR-AUTH-001～006；C-AUTH-01～09 |
| KnowledgePermission、ApiKeyScope 范围交集 | FR-AUTH-002/003/006；C-AUTH-04/07/08 |
| KnowledgeBase、Document、DocumentVersion、状态唯一性 | FR-KB-001～003；FR-DOC-001～004；C-DOC-01～08 |
| MetadataSchemaVersion、JSONB 元数据和块级覆盖 | FR-DOC-002；FR-SEARCH-006；C-META-01～06 |
| DocumentObject 与七牛云私有对象边界 | FR-DOC-001；需求说明书 12.3、12.7；C-NFR-03；系统设计 4.4、10.2、11.4 |
| IngestionTask、TaskExecution、Outbox、租约 | FR-INGEST-001～005；AC-001/009；系统设计 4.4、6.4、6.5 |
| Chunk、Citation 和历史正文限制 | FR-CHUNK-001～004；FR-CITE-001/002；C-DOC-05/06/08 |
| IndexVersion、快照成员、模型指纹、双索引切换 | FR-INDEX-001～003；C-DOC-03/04；C-SEARCH-02/03 |
| pgvector 1024 维向量和权威复核 | FR-SEARCH-003/006；C-META-03；系统设计 4.5～4.7 |
| RetrievalParameterSet、RetrievalTrace | FR-SEARCH-009；FR-ADMIN-003；C-NFR-05 |
| EvaluationSet/Case/Run、ReleaseGateRecord | FR-EVAL-001～003；AC-008；确认单 8.2 |
| ModelRuntimeConfig 的 Rerank 候选评测与活动配置切换 | FR-ADMIN-001；C-SEARCH-09；系统设计 4.6、4.9 |
| AuditLog、DeletionTombstone、BackupRecoveryPoint | 需求说明书 12.2～12.7；C-NFR-04～07；AC-007/012 |

## 15. 设计结论

本设计以 PostgreSQL 为业务事实来源，以 pgvector 和 Elasticsearch 为可重建派生索引；原文件和受保护预览仅存放于公司七牛云 Kodo 私有空间，数据库只保存可核验对象引用；Redis 仅承担消息、缓存和限流。数据库结构覆盖三份基线要求的身份权限、版本化内容、可靠任务、原子发布、检索追踪、评测、审计和删除恢复，不引入需求范围外的业务实体。

上述覆盖基于第 1.4 节所列评审稿；编号或章节覆盖不能替代相关方签署或实际数据库迁移验证。UPLOADED 删除与 Rerank 升级已有设计规则，仍需按第 13.2 节执行数据库及业务验收。本文为数据库设计修订，不包含已执行的建表、迁移或运行测试结果。

## 16. 关联文档

- [RAG 知识库平台需求说明书 V1.25](./01-RAG知识库平台需求说明书.md)
- [RAG 知识库平台需求确认单 V1.4](./02-RAG知识库平台需求确认单.md)
- [RAG 知识库平台系统设计说明书 V1.7](./03-RAG知识库平台系统设计说明书.md)

技术语义核对：[PostgreSQL 约束说明](https://www.postgresql.org/docs/current/ddl-constraints.html)、[pgvector 官方索引与过滤说明](https://github.com/pgvector/pgvector)。正式迁移以冻结的兼容版本验证。
