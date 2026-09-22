# RAG 知识库平台数据库设计说明书

> 项目名称：RAG Base  
> 文档版本：V1.0  
> 文档状态：数据库设计评审稿  
> 编制日期：2026-09-22  
> 需求基线：《RAG 知识库平台需求说明书》V1.23  
> 确认基线：《RAG 知识库平台需求确认单》V1.2  
> 系统设计基线：《RAG 知识库平台系统设计说明书》V1.3

## 1. 文档说明

### 1.1 编制目的

本文定义 RAG Base MVP 的 PostgreSQL/pgvector 数据模型、表结构、主外键、唯一约束、索引、状态字段、事务边界、数据保留和派生索引映射，用于指导 SQLAlchemy 模型、Alembic 迁移、数据访问代码和数据库测试。

### 1.2 设计范围

本文仅覆盖已确认的数据库设计内容：

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

需求说明书仍为“评审稿”，需求确认单仍为“待确认”，因此本文同样为评审稿。基线签署或上游文档变更后，应执行数据库影响分析并更新本文版本。

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
- 所有摘要使用十六进制小写字符串，SHA-256 字段长度固定为 64；
- JSONB 字段默认值使用空对象或空数组，不使用 SQL `NULL` 表示“空集合”。

### 2.3 通用字段规则

除表定义明确使用自然/复合主键外，第 6 章中的业务表均包含 `id UUID PRIMARY KEY`，由应用生成 UUIDv7。除追加写表和不可变定义表外，可变业务表同时包含 `created_at`、`updated_at` 和 `lock_version`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `UUID` | 应用生成 UUIDv7；数据库不生成随机 UUID |
| `created_at` | `TIMESTAMPTZ` | 创建时间，不可回写 |
| `updated_at` | `TIMESTAMPTZ` | 最后更新时间 |
| `lock_version` | `BIGINT` | 乐观锁版本，从 0 开始，每次受控更新加 1 |

追加写表不设置 `updated_at` 和 `lock_version`。所有外键列按访问路径决定是否建立普通索引，PostgreSQL 不会自动为外键创建索引。

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

任何签名下载 URL、上传 Token、下载 Token、七牛云 AccessKey/SecretKey、API Key 明文、密码明文、原始检索问题和模型密钥均不得写入数据库。

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
    user_account ||--o{ knowledge_permission : receives
    api_account ||--o{ knowledge_permission : receives
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
    chunk ||--o{ citation : identifies
    knowledge_base ||--o{ index_version : publishes
    index_version ||--o{ index_snapshot_member : contains
    chunk ||--o{ index_snapshot_member : snapshots
    embedding_model_fingerprint ||--o{ index_version : binds
    index_version ||--o{ index_chunk_vector : stores
    chunk ||--o{ index_chunk_vector : embeds
~~~

### 4.3 任务、评测与治理

~~~mermaid
erDiagram
    document_version ||--o{ ingestion_task : processes
    ingestion_task ||--o{ task_execution : executes
    ingestion_task ||--o{ outbox_event : dispatches
    knowledge_base ||--o{ retrieval_parameter_set : owns
    knowledge_base ||--o{ evaluation_set : owns
    evaluation_set ||--o{ evaluation_set_version : versions
    evaluation_set_version ||--o{ evaluation_case : contains
    evaluation_set_version ||--o{ evaluation_run : runs
    index_version ||--o{ evaluation_run : freezes
    retrieval_parameter_set ||--o{ evaluation_run : uses
    tenant ||--o{ retrieval_trace : records
    tenant ||--o{ audit_log : audits
    tenant ||--o{ deletion_tombstone : protects
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
| 索引 | `index_chunk_vector` | 按索引版本保存的 1024 维向量 |
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

约束：`UNIQUE(code)`；`CHECK(status='ACTIVE')`。数据库不能仅靠表约束限制为一行，应用初始化和迁移测试必须验证固定租户唯一。

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
| `must_change_password` | `BOOLEAN` | 否 | 是否必须修改临时密码 |
| `last_login_at` | `TIMESTAMPTZ` | 是 | 最后成功登录时间 |
| `disabled_at` | `TIMESTAMPTZ` | 是 | 禁用时间 |
| `created_by` | `UUID` | 是 | 创建人；初始管理员可空 |

索引与约束：`UNIQUE(tenant_id, normalized_email)`；按 `(tenant_id, status)` 建管理列表索引；`failed_login_count >= 0`。

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
| `refresh_token_hash` | `VARCHAR(128)` | 是 | 如使用刷新凭证，仅保存摘要 |
| `issued_at` | `TIMESTAMPTZ` | 否 | 签发时间 |
| `expires_at` | `TIMESTAMPTZ` | 否 | 绝对失效时间 |
| `revoked_at` | `TIMESTAMPTZ` | 是 | 撤销时间 |
| `revoke_reason` | `VARCHAR(64)` | 是 | 退出、密码修改、禁用或强制下线 |
| `last_seen_at` | `TIMESTAMPTZ` | 是 | 最近使用时间 |

索引：`(user_account_id, revoked_at, expires_at)`。访问有效性必须同时验证账号状态、账号 `session_version`、会话撤销和过期时间。

### 6.5 `password_reset_credential`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `user_account_id` | `UUID` | 否 | 目标人工账号 |
| `credential_hash` | `VARCHAR(128)` | 否 | 一次性临时凭证的不可逆摘要 |
| `hash_algorithm` | `VARCHAR(32)` | 否 | 摘要算法及版本 |
| `status` | `VARCHAR(16)` | 否 | `ACTIVE/USED/REVOKED/EXPIRED` |
| `expires_at` | `TIMESTAMPTZ` | 否 | 短期有效截止时间 |
| `used_at` | `TIMESTAMPTZ` | 是 | 首次成功使用时间 |
| `created_by` | `UUID` | 否 | 发起重置的管理员 |

约束：每个账号最多一个 ACTIVE 凭证；完整临时凭证只在创建时交付一次。使用成功时在同一事务中改为 USED、更新密码、设置 `must_change_password=true` 并递增 `session_version`。

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
| `disabled_at` | `TIMESTAMPTZ` | 是 | 禁用时间 |
| `last_called_at` | `TIMESTAMPTZ` | 是 | 最近成功调用时间 |
| `created_by` | `UUID` | 否 | 创建人 |

约束：`UNIQUE(tenant_id, name)`；状态索引 `(tenant_id, status)`。该表不含密码且不能创建登录会话。

### 6.7 `api_key`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `api_account_id` | `UUID` | 否 | 所属 API 调用账号 |
| `key_prefix` | `VARCHAR(24)` | 否 | 用于快速定位和脱敏展示的非秘密前缀 |
| `key_hash` | `VARCHAR(128)` | 否 | API Key 不可逆摘要 |
| `hash_algorithm` | `VARCHAR(32)` | 否 | 摘要算法及版本 |
| `name` | `VARCHAR(128)` | 否 | 密钥用途名称 |
| `status` | `VARCHAR(16)` | 否 | `ACTIVE/DISABLED/DELETED` |
| `last_used_at` | `TIMESTAMPTZ` | 是 | 最后成功调用时间 |
| `disabled_at` | `TIMESTAMPTZ` | 是 | 禁用时间 |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 删除时间 |
| `created_by` | `UUID` | 否 | 创建人 |

约束：`UNIQUE(key_prefix)`、`UNIQUE(key_hash)`；不设置 `expires_at`，符合永久有效直至禁用、删除或所属账号失效的规则。完整 Key 只在创建事务成功后返回一次，任何响应摘要和日志都不得保存完整值。

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
- 对人工账号和 API 账号分别建立“未撤销授权”的部分唯一索引；
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
| `chunk_config` | `JSONB` | 否 | 分块策略和参数 |
| `chunk_config_version` | `INTEGER` | 否 | 分块配置版本，从 1 开始 |
| `active_metadata_schema_version_id` | `UUID` | 是 | 当前 Schema 版本 |
| `active_index_version_id` | `UUID` | 是 | 当前活动索引版本 |
| `created_by` | `UUID` | 否 | 创建人 |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 删除完成时间 |

约束：`UNIQUE(tenant_id, name)`；活动 Schema 和活动索引必须属于本知识库，该跨行约束由事务服务锁定校验并由一致性测试覆盖。删除状态不复用名称，避免墓碑和审计歧义。创建知识库与为创建人写入四项直接权限在同一事务中完成。

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

约束：`UNIQUE(knowledge_base_id, version_no)`；每库最多一个 `ACTIVE` 的部分唯一索引。参与索引后的版本不可原地修改字段类型，只能创建新版本。

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

约束：`UNIQUE(schema_version_id, field_name)`；`field_name` 禁止使用 `tenant_id`、权限、状态、索引版本等保留名称。已用于索引的字段删除前先置 DISABLED，类型变化必须新建 Schema 版本。

元数据值保存在 DocumentVersion 和 Chunk 的 JSONB 中，但不是无约束自由 JSON：写入前必须按对应 SchemaVersion 校验名称、类型、必填、枚举和长度。文档值默认复制到 Chunk，只有 `chunk_overridable=true` 的字段可以覆盖。K12 模板只预置需求确认的 `content_type`、`course_id`、`grade_code`、`subject_code`、`campus_id`、`term_code`、`class_type`、`valid_from`、`valid_to`、`sales_stage`、`channel`、`tags`；不得加入 `department_ids` 或 `security_level`。`valid_from/valid_to` 使用 UTC 时间，可空，且校验 `valid_to > valid_from`；检索采用左闭右开区间。

### 6.13 `document`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `knowledge_base_id` | `UUID` | 否 | 所属知识库 |
| `name` | `VARCHAR(512)` | 否 | 业务文档名 |
| `status` | `VARCHAR(16)` | 否 | `ACTIVE/DELETING/DELETED` |
| `current_published_version_id` | `UUID` | 是 | 当前发布版本 |
| `created_by` | `UUID` | 否 | 创建人 |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 删除完成时间 |

同一知识库允许业务上重名文档，因此不对名称建立唯一约束；列表索引为 `(knowledge_base_id, status, updated_at DESC)`。`current_published_version_id` 必须属于本 Document。

### 6.14 `document_version`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `document_id` | `UUID` | 否 | 稳定文档身份 |
| `version_no` | `INTEGER` | 否 | 文档内递增版本号 |
| `source_version_id` | `UUID` | 是 | 从已发布版本复制草稿时的来源版本 |
| `change_type` | `VARCHAR(16)` | 否 | `UPLOAD/REPLACE/EDIT/ROLLBACK` |
| `change_reason` | `TEXT` | 是 | 上传、编辑或回滚原因 |
| `original_filename` | `VARCHAR(512)` | 否 | 原始文件名，仅展示，不用于对象键 |
| `file_format` | `VARCHAR(16)` | 否 | 已确认支持格式 |
| `file_size_bytes` | `BIGINT` | 否 | 文件大小 |
| `file_sha256` | `VARCHAR(64)` | 否 | 文件摘要 |
| `mime_type` | `VARCHAR(255)` | 否 | 服务端识别 MIME |
| `status` | `VARCHAR(24)` | 否 | `UPLOADED/PROCESSING/REVIEW_REQUIRED/PUBLISHED/DISABLED/FAILED/DELETING/DELETED` |
| `disable_reason` | `VARCHAR(16)` | 是 | `MANUAL/SUPERSEDED` |
| `metadata_schema_version_id` | `UUID` | 否 | 校验本版本元数据的 Schema |
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
- `file_size_bytes > 0`，格式限制为 PDF、DOCX、XLSX、PPTX、TXT、MD、HTML、CSV、JSON、EML、PNG、JPG、JPEG；
- `disable_reason` 仅在 `DISABLED` 时非空；
- `CREATE UNIQUE INDEX ... ON document_version(document_id) WHERE status='PUBLISHED'`；
- 状态索引 `(document_id, status)` 和 `(status, updated_at)`；
- 状态变更必须同时更新 `document.current_published_version_id` 和相关索引版本，不允许直接改单列。

### 6.15 `document_object`

仅保存七牛云对象引用，不保存 URL 或访问 Token。

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `document_version_id` | `UUID` | 否 | 所属文档版本 |
| `object_kind` | `VARCHAR(24)` | 否 | `STAGED_ORIGINAL/ORIGINAL/PREVIEW` |
| `bucket_code` | `VARCHAR(64)` | 否 | 配置中的 Bucket 逻辑编号，不保存密钥 |
| `object_key` | `VARCHAR(1024)` | 否 | 平台生成的对象键 |
| `source_object_id` | `UUID` | 是 | 正式对象从暂存对象转化时的来源 |
| `content_sha256` | `VARCHAR(64)` | 否 | 内容摘要 |
| `qiniu_etag` | `VARCHAR(128)` | 是 | 七牛云对象 ETag |
| `size_bytes` | `BIGINT` | 否 | 对象大小 |
| `storage_status` | `VARCHAR(24)` | 否 | `STAGED/FINALIZING/ACTIVE/DELETE_PENDING/DELETED/ORPHANED` |
| `source_locator` | `JSONB` | 否 | 预览页码/幻灯片等定位；原文件为空对象 |
| `last_verified_at` | `TIMESTAMPTZ` | 是 | 最近存在性/摘要验证时间 |
| `operation_attempt_count` | `INTEGER` | 否 | 最终化、验证或删除累计尝试次数 |
| `delete_requested_at` | `TIMESTAMPTZ` | 是 | 删除请求时间 |
| `delete_verified_at` | `TIMESTAMPTZ` | 是 | Kodo 确认对象不存在的时间 |
| `last_error_code` | `VARCHAR(64)` | 是 | 稳定错误码 |

约束与处理规则：

- `UNIQUE(bucket_code, object_key)`；
- 正式键由 `tenant_id/knowledge_base_id/document_id/document_version_id/object_kind` 等不可变标识确定，重复最终化写入同一键；
- 同一文档版本最多一个未删除 `ORIGINAL`，预览可按页或格式存在多条，需在对象键中体现定位；
- 对象删除只有在 Kodo 返回成功或已不存在，并记录 `delete_verified_at` 后才能标记 `DELETED`；
- 孤儿扫描只删除超过安全窗口且不存在有效 `document_object` 引用的暂存对象；
- 签名 URL 按请求临时生成，不写本表、缓存持久层、追踪或审计。

### 6.16 `document_relation`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `parent_document_version_id` | `UUID` | 否 | EML 等父文档版本 |
| `child_document_id` | `UUID` | 否 | 附件形成的独立文档 |
| `relation_type` | `VARCHAR(24)` | 否 | MVP 固定为 `ATTACHMENT` |
| `source_locator` | `JSONB` | 否 | 附件名、序号等结构定位，不保存正文 |

约束：`UNIQUE(parent_document_version_id, child_document_id, relation_type)`；禁止循环父子关系，由应用层在创建时检测。

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
| `producer_version` | `VARCHAR(128)` | 否 | 解析器或清洗规则版本 |
| `is_deleted` | `BOOLEAN` | 否 | 显式删除后为 true |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 内容清除时间 |

约束：`UNIQUE(document_version_id, artifact_type, producer_version)`；`content_length >= 0`、`part_count >= 0`。`is_deleted=true` 时不得存在关联分段。正文读取只用于受控管理、重建和解析诊断，不参与外部接口直接返回。

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
| `enable_sync_status` | `VARCHAR(16)` | 否 | `SYNCED/ENABLING/DISABLING/FAILED` |
| `edited_from_chunk_id` | `UUID` | 是 | 新草稿块的来源块 |
| `deleted_at` | `TIMESTAMPTZ` | 是 | 显式删除清除时间 |

约束与索引：

- `UNIQUE(document_version_id, chunk_no)`；
- 父块必须属于同一 `document_version_id`，由复合外键或应用事务校验；
- `token_count >= 0`；未删除块 `content IS NOT NULL`，已删除块 `content IS NULL`；
- `(document_version_id, enabled)`、`(parent_chunk_id)` 和 GIN `metadata jsonb_path_ops` 索引；
- 已发布正文或业务元数据不能原地更新；编辑时复制文档版本和块；仅 `enabled`、`enable_sync_status` 可按启停流程受控更新。

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

约束：一个 Chunk 对应一个稳定 Citation，`UNIQUE(chunk_id)` 采用 `NULLS NOT DISTINCT` 之外的部分唯一索引实现。历史或删除状态不保存正文和预览地址；访问时仍执行当前或允许的历史授权校验。

### 6.21 `ingestion_task`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `document_version_id` | `UUID` | 否 | 处理目标 |
| `task_type` | `VARCHAR(24)` | 否 | MVP 为 `INGEST`，评测使用独立表 |
| `status` | `VARCHAR(24)` | 否 | `PENDING/RUNNING/SUCCEEDED/RETRY_WAIT/FAILED/CANCEL_REQUESTED/CANCELLED/CLEANUP_PENDING` |
| `current_stage` | `VARCHAR(32)` | 否 | 固定九阶段或 `CLEANUP` |
| `progress_percent` | `SMALLINT` | 否 | 0～100 |
| `parameter_snapshot` | `JSONB` | 否 | 分块、Schema、解析和模型参数快照 |
| `idempotency_key` | `VARCHAR(128)` | 否 | 文档版本与处理配置的幂等键 |
| `attempt_count` | `INTEGER` | 否 | 已开始执行次数 |
| `lease_owner` | `VARCHAR(128)` | 是 | 当前 Worker 实例 |
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
- 对同一文档版本在 `PENDING/RUNNING/RETRY_WAIT/CANCEL_REQUESTED` 状态建立部分唯一索引；
- 调度索引 `(status, next_retry_at)`、租约索引 `(status, lease_expires_at)`；
- RUNNING 必须同时具有租约持有者、到期时间和心跳；非 RUNNING 不保留活动租约；
- `SUCCEEDED` 只表示产物校验完成并进入 `REVIEW_REQUIRED`，不表示已发布。

### 6.22 `task_execution`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `ingestion_task_id` | `UUID` | 否 | 所属任务 |
| `execution_no` | `INTEGER` | 否 | 任务内递增执行次数 |
| `worker_id` | `VARCHAR(128)` | 否 | Worker 实例 |
| `status` | `VARCHAR(16)` | 否 | `RUNNING/SUCCEEDED/FAILED/CANCELLED/LOST` |
| `started_at` | `TIMESTAMPTZ` | 否 | 开始时间 |
| `heartbeat_at` | `TIMESTAMPTZ` | 是 | 最近心跳 |
| `finished_at` | `TIMESTAMPTZ` | 是 | 结束时间 |
| `stage_timings` | `JSONB` | 否 | 各阶段耗时，不含正文 |
| `error_code` | `VARCHAR(64)` | 是 | 稳定错误码 |
| `error_summary` | `TEXT` | 是 | 脱敏摘要 |

约束：`UNIQUE(ingestion_task_id, execution_no)`；同一任务最多一条 `RUNNING` 执行记录。历史记录只追加，不覆盖失败信息。

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

约束：`UNIQUE(event_type, business_key)`；投递扫描索引 `(status, available_at)`。投递采用 `FOR UPDATE SKIP LOCKED` 或等价条件更新；Broker 接受成功后标记 DELIVERED，消费者仍必须幂等。

### 6.24 `idempotency_record`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `actor_type` | `VARCHAR(16)` | 否 | `USER/API_ACCOUNT/SYSTEM` |
| `actor_id` | `UUID` | 否 | 操作主体 |
| `operation` | `VARCHAR(64)` | 否 | 发布、删除、授权等操作 |
| `idempotency_key` | `VARCHAR(128)` | 否 | 客户端或服务端幂等键 |
| `request_fingerprint` | `VARCHAR(64)` | 否 | 规范化请求摘要 |
| `status` | `VARCHAR(16)` | 否 | `PROCESSING/SUCCEEDED/FAILED` |
| `response_status` | `INTEGER` | 是 | 首次响应状态码 |
| `response_summary` | `JSONB` | 是 | 不含敏感正文的首次响应摘要 |
| `expires_at` | `TIMESTAMPTZ` | 是 | 仅控制幂等复用窗口，不代表业务数据删除 |

约束：`UNIQUE(actor_type, actor_id, operation, idempotency_key)`。相同 Key 但请求指纹不同返回冲突，不得执行第二次写操作。

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
| `metadata_schema_version_id` | `UUID` | 否 | 冻结 Schema |
| `embedding_fingerprint_id` | `UUID` | 否 | 冻结 Embedding 空间 |
| `chunk_config_version` | `INTEGER` | 否 | 冻结分块配置版本 |
| `index_schema_version` | `VARCHAR(64)` | 否 | ES/pgvector 映射版本 |
| `snapshot_digest` | `VARCHAR(64)` | 否 | 有序成员清单摘要 |
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
- 活动切换、文档版本切换和旧索引退役必须在同一短事务中完成，事务内不调用 Elasticsearch。

### 6.27 `index_snapshot_member`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `index_version_id` | `UUID` | 否 | 索引版本 |
| `chunk_id` | `UUID` | 否 | 不可变内容块 |
| `document_id` | `UUID` | 否 | 冗余过滤字段 |
| `document_version_id` | `UUID` | 否 | 冗余版本字段 |
| `content_sha256` | `VARCHAR(64)` | 否 | 加入快照时的内容摘要 |
| `metadata_digest` | `VARCHAR(64)` | 否 | 加入快照时的元数据摘要 |

约束：`UNIQUE(index_version_id, chunk_id)`；索引 `(index_version_id, document_id)`。冗余字段必须与 Chunk 来源一致，写入后不可修改；它们用于完整性校验和重建，不取代 PostgreSQL 权威状态复核。

### 6.28 `index_chunk_vector`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `index_version_id` | `UUID` | 否 | 索引版本 |
| `tenant_id` | `UUID` | 否 | 固定租户强制过滤字段 |
| `knowledge_base_id` | `UUID` | 否 | 强制过滤字段 |
| `document_id` | `UUID` | 否 | 文档过滤字段 |
| `document_version_id` | `UUID` | 否 | 当前版本过滤字段 |
| `chunk_id` | `UUID` | 否 | 内容块 |
| `chunk_enabled` | `BOOLEAN` | 否 | 与启停流程同步的派生过滤值 |
| `document_status` | `VARCHAR(16)` | 否 | 构建时固定为 `PUBLISHED`，查询仍做权威复核 |
| `metadata` | `JSONB` | 否 | 经 Schema 校验的可过滤业务元数据 |
| `embedding_fingerprint_id` | `UUID` | 否 | 向量空间指纹 |
| `embedding` | `VECTOR(1024)` | 否 | L2 归一化向量 |
| `embedding_sha256` | `VARCHAR(64)` | 否 | 标准化向量摘要 |
| `created_at` | `TIMESTAMPTZ` | 否 | 创建时间 |

约束：`UNIQUE(index_version_id, chunk_id)`；普通过滤索引 `(tenant_id, knowledge_base_id, index_version_id, document_status, chunk_enabled)`、`(index_version_id, document_id)` 及业务元数据 GIN；向量索引使用 pgvector 支持的余弦 HNSW 方案，具体参数由性能基线确定，不在本文预设。查询必须同时限定固定租户、活动 `index_version_id`、知识库、当前文档版本、`document_status='PUBLISHED'` 和 `chunk_enabled=true`，返回前再复核 `chunk` 与 `document_version` 权威状态。

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

约束：参数字段必须满足需求默认值和范围；分别为“知识库内参数集”和 `knowledge_base_id IS NULL` 的跨库参数集建立唯一索引，避免 PostgreSQL 空值语义绕过唯一性。参数集仅供测试和评测，不成为调用方不可覆盖的线上知识库配置。

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
| `query_hmac` | `VARCHAR(128)` | 否 | HMAC-SHA256 指纹 |
| `hmac_key_id` | `VARCHAR(64)` | 否 | 指纹密钥编号，不是密钥 |
| `normalization_version` | `VARCHAR(32)` | 否 | 查询规范化规则版本 |
| `query_char_count` | `INTEGER` | 否 | 查询字符数 |
| `knowledge_base_ids` | `JSONB` | 否 | 有序去重后的知识库 UUID 数组 |
| `document_ids` | `JSONB` | 否 | 可选文档范围 UUID 数组 |
| `effective_parameters` | `JSONB` | 否 | 实际生效参数及模型/tokenizer 版本 |
| `index_versions` | `JSONB` | 否 | 各知识库捕获的活动索引版本 |
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

### 6.31 评测表

#### `evaluation_set`

保存评测集稳定身份：`knowledge_base_id`、`name`、`description`、`status(ACTIVE/ARCHIVED)`、`created_by`。约束 `UNIQUE(knowledge_base_id, name)`。

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

约束：`UNIQUE(evaluation_set_id, version_no)`；冻结后禁止增删改用例。

#### `evaluation_case`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `evaluation_set_version_id` | `UUID` | 否 | 评测版本 |
| `case_no` | `INTEGER` | 否 | 版本内序号 |
| `query_text` | `TEXT` | 否 | 经批准和脱敏的真实问题 |
| `expected_targets` | `JSONB` | 否 | 期望文档/块编号及可选相关等级 |
| `metadata_filter` | `JSONB` | 否 | 可选业务过滤条件 |
| `notes` | `TEXT` | 是 | 备注 |

约束：`UNIQUE(evaluation_set_version_id, case_no)`。该表是需求明确允许保存的问题数据，不复用检索追踪；访问受 EVALUATE 权限控制。

#### `evaluation_run`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `evaluation_set_version_id` | `UUID` | 否 | 冻结评测版本 |
| `index_version_id` | `UUID` | 否 | 冻结活动索引版本 |
| `parameter_set_id` | `UUID` | 否 | 冻结参数集 |
| `embedding_fingerprint_id` | `UUID` | 否 | Embedding 指纹 |
| `rerank_model_version` | `VARCHAR(128)` | 否 | Rerank 版本 |
| `tokenizer_version` | `VARCHAR(128)` | 否 | tokenizer 版本 |
| `code_version` | `VARCHAR(128)` | 否 | 执行代码版本 |
| `run_fingerprint` | `VARCHAR(64)` | 否 | 所有冻结输入的摘要 |
| `status` | `VARCHAR(16)` | 否 | `PENDING/RUNNING/SUCCEEDED/FAILED/CANCELLED` |
| `metrics` | `JSONB` | 否 | Recall、Precision、MRR、nDCG、空结果率 |
| `permission_case_pass_rate` | `NUMERIC(6,5)` | 是 | 权限用例通过率 |
| `started_at` | `TIMESTAMPTZ` | 是 | 开始时间 |
| `finished_at` | `TIMESTAMPTZ` | 是 | 结束时间 |
| `created_by` | `UUID` | 否 | 发起人 |

索引：`run_fingerprint` 普通索引，用于比较相同输入的重复运行；运行开始后冻结字段不可修改。相同输入允许重复执行，以验证结果稳定性和性能差异。

#### `evaluation_case_result`

保存 `evaluation_run_id`、`evaluation_case_id`、`status`、`returned_targets`、`metric_values`、`trace_id`、`error_code` 和耗时；`UNIQUE(evaluation_run_id, evaluation_case_id)`。`returned_targets` 只保存编号、排名、得分和相关性判断，不复制正文。

#### `release_gate_record`

保存 `evaluation_run_id`、`gate_type=INITIAL_PRODUCTION`、`status(PASSED/FAILED)`、固定阈值快照、指标证据、审批人、审批时间和证据位置；上线门禁记录追加写，不允许通过修改历史记录把失败改为通过。

### 6.32 `model_runtime_config`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `model_type` | `VARCHAR(16)` | 否 | `EMBEDDING/RERANK/OCR` |
| `config_version` | `INTEGER` | 否 | 类型内递增版本 |
| `model_name` | `VARCHAR(128)` | 否 | 只读模型名 |
| `model_version` | `VARCHAR(128)` | 否 | 只读模型版本 |
| `model_file_sha256` | `VARCHAR(64)` | 否 | 文件摘要 |
| `endpoint_ref` | `VARCHAR(255)` | 否 | 配置中心引用或内部服务标识，不含凭证 |
| `runtime_parameters` | `JSONB` | 否 | 超时、并发、批次和计算设备 |
| `status` | `VARCHAR(16)` | 否 | `DRAFT/ACTIVE/RETIRED` |
| `created_by` | `UUID` | 否 | 配置人 |
| `activated_at` | `TIMESTAMPTZ` | 是 | 生效时间 |

每种 `model_type` 最多一个 ACTIVE。该表不允许切换到需求外模型供应商；服务密钥仅存密钥管理系统引用，不存明文。

### 6.33 `security_policy`

保存版本化登录安全策略：`version_no`、`password_min_length`（默认不低于 12）、`login_failure_threshold`、`lock_duration_seconds`、`session_ttl_seconds`、`status(DRAFT/ACTIVE/RETIRED)`、`created_by` 和 `activated_at`。每次最多一个 ACTIVE；修改策略创建新版本，不覆盖历史。该表只保存规则数值，不保存密码、盐或密钥。

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
| `result` | `VARCHAR(16)` | 否 | `SUCCESS/DENIED/FAILED` |
| `request_id` | `VARCHAR(128)` | 是 | 请求关联编号 |
| `trace_id` | `UUID` | 是 | 检索关联编号 |
| `before_summary` | `JSONB` | 否 | 脱敏变更前摘要 |
| `after_summary` | `JSONB` | 否 | 脱敏变更后摘要 |
| `client_context` | `JSONB` | 否 | 脱敏 IP、User-Agent 等 |
| `created_at` | `TIMESTAMPTZ` | 否 | 发生时间 |
| `previous_integrity_hash` | `VARCHAR(64)` | 是 | 同一审计链前一条摘要；链首可空 |
| `integrity_hash` | `VARCHAR(64)` | 否 | 当前记录防篡改摘要 |

索引：`(resource_type, resource_id, created_at DESC)`、`(actor_type, actor_id, created_at DESC)`、`(action, created_at DESC)`。生产业务账号对审计表只授予 INSERT/SELECT，不授予 UPDATE/DELETE；链摘要定期校验并输出受控证据。禁止记录密码、API Key、Token、签名 URL、密钥、完整正文和原始检索问题。

### 6.35 `deletion_tombstone`

| 字段 | 类型 | 空值 | 说明 |
| --- | --- | --- | --- |
| `tenant_id` | `UUID` | 否 | 固定租户 |
| `resource_type` | `VARCHAR(32)` | 否 | `KNOWLEDGE_BASE/DOCUMENT/DOCUMENT_VERSION` |
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
| `status` | `VARCHAR(24)` | 否 | `CREATING/READY/VERIFIED/FAILED/EXPIRED` |
| `postgres_backup_ref` | `TEXT` | 否 | 私有备份系统中的数据库备份引用 |
| `postgres_wal_lsn` | `PG_LSN` | 否 | 数据库一致性水位 |
| `object_manifest_ref` | `TEXT` | 否 | 私有备份系统中的七牛云对象清单引用 |
| `object_manifest_sha256` | `VARCHAR(64)` | 否 | 清单摘要 |
| `object_count` | `BIGINT` | 否 | 清单对象数 |
| `tombstone_ledger_sequence` | `BIGINT` | 否 | 恢复点已包含的墓碑账本水位 |
| `index_backup_ref` | `TEXT` | 是 | 为满足 RTO 保留的派生索引备份引用 |
| `created_at` | `TIMESTAMPTZ` | 否 | 创建时间 |
| `verified_at` | `TIMESTAMPTZ` | 是 | 恢复演练验证时间 |
| `verification_summary` | `JSONB` | 否 | 恢复验证摘要和各阶段耗时 |

约束：`UNIQUE(recovery_point_code)`。只有数据库备份、对象清单摘要和墓碑水位都成功固化后才能进入 READY；季度恢复演练验证后进入 VERIFIED。清单引用不得指向公开地址，也不包含七牛云签名 Token。

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
WHERE status IN ('PENDING', 'RUNNING', 'RETRY_WAIT', 'CANCEL_REQUESTED');

CREATE UNIQUE INDEX uq_task_one_running_execution
ON task_execution (ingestion_task_id)
WHERE status = 'RUNNING';
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
| Citation 到版本/块 | `SET NULL` 或清理前显式断开 | 删除正文后仍保留最小来源状态 |
| 审计/追踪到业务对象 | 不设置强制外键或使用可空外键 | 业务删除不能破坏历史日志 |

业务删除禁止依赖无条件 `ON DELETE CASCADE`。级联只可用于尚未发布且无审计价值的纯从属临时数据，并需在迁移评审中单独证明。

为防止“引用存在但归属错误”，以下关系使用复合候选键和复合外键，而不是只引用目标 `id`：

- `document(id, current_published_version_id)` 引用 `document_version(document_id, id)`；
- `knowledge_base(id, active_index_version_id)` 引用 `index_version(knowledge_base_id, id)`；
- `knowledge_base(id, active_metadata_schema_version_id)` 引用 `metadata_schema_version(knowledge_base_id, id)`；
- `chunk(document_version_id, parent_chunk_id)` 引用 `chunk(document_version_id, id)`。

上述循环外键在基础表创建后由后续迁移步骤补充，可声明为可延迟检查，但应用发布事务仍需显式验证目标状态。DocumentVersion 引用的 MetadataSchemaVersion 也必须属于同一 KnowledgeBase，该约束由写入事务和一致性检查共同保证。

### 7.3 状态迁移执行规则

- 状态字段不可通过通用更新接口修改；
- Service 在事务内执行 `UPDATE ... WHERE id=? AND status=? AND lock_version=?`；受影响行数为 0 时返回 409；
- 文档发布锁定知识库，验证 READY 索引及基线版本后同时更新 DocumentVersion、Document、IndexVersion 和 KnowledgeBase；
- 内容块停用先提交 `chunk.enabled=false` 和 Outbox；启用时保持 false，双索引确认后再置 true；
- 删除先把业务对象置为 DELETING 并写墓碑和 Outbox；只有 PostgreSQL 正文、pgvector、Elasticsearch、Redis、七牛云原文件和预览均完成清理复核后才置 DELETED/VERIFIED。

## 8. 索引与访问路径

### 8.1 必要关系索引

| 主要查询 | 索引 |
| --- | --- |
| 邮箱登录 | `user_account(tenant_id, normalized_email)` 唯一 |
| API Key 鉴权 | `api_key(key_prefix)` 唯一，定位后验证摘要 |
| 账号有效授权 | 两个主体维度的未撤销部分唯一索引 |
| 知识库文档列表 | `document(knowledge_base_id, status, updated_at DESC)` |
| 文档版本列表 | `document_version(document_id, version_no DESC)` |
| 入库调度 | `ingestion_task(status, next_retry_at)` |
| 租约恢复 | `ingestion_task(status, lease_expires_at)` |
| Outbox 投递 | `outbox_event(status, available_at)` |
| 活动索引读取 | `index_version(knowledge_base_id)` 的 ACTIVE 部分唯一索引 |
| 快照成员读取 | `index_snapshot_member(index_version_id, chunk_id)` 唯一 |
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

MVP 使用 `VECTOR(1024)` 和余弦 HNSW 索引。HNSW 的构建参数、查询参数和候选放大倍数必须通过代表性语料性能基线确定，不在数据库文档中虚构固定值。向量检索必须：

1. 使用活动 IndexVersion 对应的 Embedding 指纹生成查询向量；
2. 强制限定租户、知识库、活动索引版本、文档范围和 `chunk_enabled=true`；
3. 取候选后回查 PostgreSQL 的知识库、文档版本和 Chunk 权威状态；
4. 不把 ANN 结果当作权限或发布状态事实。

### 8.4 分区策略

当前文档量、追踪量和审计量尚无确认数值，MVP 不预先分区，避免增加运维复杂度。上线基线形成后，如单表规模、写入速率或维护窗口证明需要，再对 `retrieval_trace`、`audit_log` 等追加写大表按时间进行在线迁移；不允许因分区自动删除需求规定长期保留的数据。

## 9. 事务、并发与一致性

### 9.1 上传与任务创建

七牛云暂存写入无法加入 PostgreSQL 事务，采用以下顺序：

1. API 完成权限、扩展名、MIME、流式大小等入口校验；
2. 使用平台生成键流式写入 Kodo 暂存前缀；
3. 在一个 PostgreSQL 事务中创建 Document/DocumentVersion、`document_object(STAGED)`、IngestionTask 和 OutboxEvent；
4. 提交后由投递器发送 `task_id`；
5. Worker 取得任务租约后执行深度文件特征和安全校验，再以确定性正式键完成对象最终化；
6. API 在数据库事务失败时记录可识别暂存键供孤儿扫描；孤儿扫描只清理超过安全窗口且数据库无有效引用的对象。

对象最终化和重试以 `(bucket_code, object_key)` 唯一约束、内容摘要和 Kodo ETag 共同判定幂等。

### 9.2 发布事务

发布事务只执行 PostgreSQL 操作，不包含 Elasticsearch 或七牛云网络调用：

1. 确认 IndexVersion 为 READY、校验证据 PASSED；
2. 锁定 `knowledge_base` 行；
3. 比较 `base_active_index_version_id` 与当前活动版本；
4. 条件更新新文档版本为 PUBLISHED、旧版本为 DISABLED/SUPERSEDED；
5. 更新 Document 当前发布版本；
6. 新 IndexVersion 改为 ACTIVE，旧版本改为 RETIRED；
7. 更新 KnowledgeBase 活动索引引用并写 AuditLog、OutboxEvent；
8. 提交。并发基线已变化时整体回滚并返回 409。

### 9.3 任务租约

Worker 通过条件更新从 PENDING 获取任务，将状态改为 RUNNING，同时写入 `lease_owner`、`lease_expires_at`、`heartbeat_at` 并创建 TaskExecution。心跳只允许租约持有者更新。租约过期后 Scheduler 把旧执行标记 LOST，再按失败分类转入 RETRY_WAIT、PENDING 或 FAILED；禁止两个 Worker 同时持有有效租约。

### 9.4 权限变更

授权、撤销、API 账号禁用、API Key 禁用与权限缓存失效事件在同一事务中写入。请求入口始终复核 PostgreSQL 的主体状态版本；权限缓存超过版本或失效事件异常时回源数据库。API Key 可用知识库集合为：

`当前有效 api_key_scope ∩ 当前有效 api_account RETRIEVE knowledge_permission`。

### 9.5 内容块启停

- 停用：事务更新 `chunk.enabled=false`、`enable_sync_status=DISABLING` 并写 Outbox；末端数据库复核立即阻止返回，双索引同步后置为 SYNCED。
- 启用：先保持 `enabled=false`、状态 ENABLING；pgvector 与 Elasticsearch 均更新并验证可查询后，事务将 `enabled=true`、状态 SYNCED。任一后端失败则保持 false 并置 FAILED。

## 10. Elasticsearch 与 Redis 数据映射

### 10.1 Elasticsearch 文档

每个 ES 索引文档至少包含：`tenant_id`、`knowledge_base_id`、`index_version_id`、`document_id`、`document_version_id`、`chunk_id`、`chunk_enabled`、正文、标题路径、来源定位、允许过滤的业务元数据和内容摘要。ES `_id` 使用稳定的 `index_version_id:chunk_id` 派生值。

ES 不保存账号、API Key、密码、审计事实或唯一发布状态。ES 索引丢失时从 IndexSnapshotMember、Chunk 和活动模型/Schema 指纹重建。

### 10.2 Redis 键边界

Redis Broker 只承载含 `task_id` 的 Celery 消息；Redis Cache 只保存可回源的权限、配置和短期辅助数据；限流键以 API Key 编号和 API Account 编号为主体。两者使用独立逻辑库或键前缀。任何 Redis 数据丢失不得造成授权扩大、任务永久丢失或发布状态倒退。

## 11. 安全与隐私

- 数据库连接强制 TLS，并使用 API、Worker、Scheduler、只读观测等不同最小权限账号；
- 生产应用账号不拥有建库、安装扩展、修改迁移历史或绕过审计的权限；
- 密码只保存自适应哈希，API Key 只保存不可逆摘要；
- 数据库不保存七牛云密钥、模型密钥、HMAC 明文密钥、签名 URL 和上传/下载 Token；
- RetrievalTrace 不保存原始查询，AuditLog 不保存完整正文；
- JSONB 写入前执行字段白名单和敏感信息检查；
- 数据库备份、WAL、日志和临时文件启用静态加密；密钥材料与备份分开管理；
- 查看敏感正文必须经过额外授权，并产生 AuditLog；
- 运维查询和数据导出使用只读受控账号，导出评测片段和正文必须审计。

## 12. 删除、保留与恢复

### 12.1 显式删除

删除流程先使资源对新检索不可见，再异步清理：

- PostgreSQL：物理删除 DocumentArtifactPart，标记 Artifact 已清理，清空 Chunk 正文，删除或失效向量及快照派生关系，保留最小标识、状态、审计和墓碑；
- Elasticsearch：删除对应活动和保留索引中的文档并验证查询不可见；
- 七牛云：删除原文件和预览，对每个 DocumentObject 执行不存在性确认；
- Redis：失效权限、配置和查询辅助缓存；
- Citation/RetrievalTrace/AuditLog：只保留允许的最小标识、版本、得分、耗时和操作结果，不保留可恢复正文。

只有 `deletion_tombstone.cleanup_status=VERIFIED` 且账本已同步，业务对象才能完成 DELETED。失败保持可重试状态并告警。

### 12.2 数据保留

- 原文件、检索追踪和审计日志不按时间自动过期；
- 显式删除、法规删除和安全事件处置优先于长期保留；
- 历史文档版本可为回滚保留，但对外不得返回旧正文或预览；
- 墓碑账本保留期不得短于最长业务备份生命周期加恢复缓冲期；
- IdempotencyRecord 的复用窗口可到期，但其对应业务结果和 AuditLog 不随之删除。

### 12.3 备份恢复一致性

一个可用恢复点同时固定 PostgreSQL WAL/备份引用、七牛云对象清单摘要和墓碑账本水位。恢复顺序为：

1. 恢复墓碑账本和密钥访问能力；
2. 恢复 PostgreSQL 到指定恢复点；
3. 重放恢复点之后的删除墓碑；
4. 按对象清单恢复并核对七牛云对象；
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
| AuditLog、DeletionTombstone、BackupRecoveryPoint | 需求说明书 12.2～12.7；C-NFR-04～07；AC-007/012 |

## 15. 设计结论

本设计以 PostgreSQL 为业务事实来源，以 pgvector 和 Elasticsearch 为可重建派生索引；原文件和受保护预览仅存放于公司七牛云 Kodo 私有空间，数据库只保存可核验对象引用；Redis 仅承担消息、缓存和限流。数据库结构覆盖三份基线要求的身份权限、版本化内容、可靠任务、原子发布、检索追踪、评测、审计和删除恢复，不引入需求范围外的业务实体。

## 16. 关联文档

- [RAG 知识库平台需求说明书 V1.23](./01-RAG知识库平台需求说明书.md)
- [RAG 知识库平台需求确认单 V1.2](./02-RAG知识库平台需求确认单.md)
- [RAG 知识库平台系统设计说明书 V1.3](./03-RAG知识库平台系统设计说明书.md)
