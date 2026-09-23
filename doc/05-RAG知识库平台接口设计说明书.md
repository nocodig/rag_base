# RAG 知识库平台接口设计说明书

> 项目名称：RAG Base
> 文档版本：V1.1
> 文档状态：接口设计评审稿
> 编制日期：2026-09-23
> 需求依据：《RAG 知识库平台需求说明书》V1.25
> 确认依据：《RAG 知识库平台需求确认单》V1.4
> 系统设计依据：《RAG 知识库平台系统设计说明书》V1.7
> 数据库设计依据：《RAG 知识库平台数据库设计说明书》V1.3

> V1.1 修订：补齐重处理草稿、纯索引重建与整库回滚、模型探测和运维只读接口；收紧权限、发布及 Embedding 配置切换边界，对齐内容块启停、文件上限和上游文档版本。

## 1. 文档边界

本文定义已在上述文档中确定的对外检索 REST、MCP、管理前端私有 REST 和调用端适配契约。接口设计细化路径、字段、状态、权限、错误和并发语义；不增加业务能力。没有聊天、答案生成、工作流、多租户、外部数据源同步或对外管理 API。

当前四份依据均为评审稿；需求确认单尚未由相关方签署。本文可用于评审和联调设计，不表示接口已实现、通过兼容测试或正式验收。本轮修改前五份文档的逐份备份保存在 `doc/backups/`。本轮依据文件 SHA-256 依次为：01 `aee1da5ab30ff11ca94d9853758ab0fdaefc44ed4b2ac304ca0b65321c097fa8`、02 `afcfebed17bc107b1dcba266e11624816b89802c2625a2dfbaa818cad720af42`、03 `587b56bc808c640afd46fb82aa5ac930b5251c1bcfc8e48d74be072c40928d95`、04 `235a3a3b1687bc672385efa1ae14f4058317769ad666c91c5d18bb71fc783d62`。

## 2. 通用契约

### 2.1 入口与认证

| 入口 | 路径 | 身份 | 暴露范围 |
| --- | --- | --- | --- |
| 对外 REST | `/api/v1/retrieval` | `Authorization: Bearer <API Key>` | 仅四个只读检索接口 |
| MCP | `/mcp` | 同一个 Bearer API Key | 仅三个检索工具 |
| 私有管理 REST | `/internal/api/v1` | 人工账号会话 Cookie | 仅公司内网和管理前端 |

所有入口使用 TLS。外部 API Key 不可用于管理接口；人工账号会话不可用于对外 REST 或 MCP。服务端注入固定 `tenant_id`，任何接口均不接受客户端传入租户编号。权限判断以 PostgreSQL 当前账号、API Key、直接授权和密钥范围为准；平台角色不自动赋予知识库正文权限。HTTP 与 MCP 使用同一检索应用服务、授权范围、限流计数、结果与错误语义。

管理会话 Cookie 使用 `HttpOnly; Secure; SameSite=Lax`；登录成功同时提供与会话绑定的 CSRF Token。除登录和安全的 GET 外，管理请求必须发送 `X-CSRF-Token`。临时密码首次登录只允许改密和退出；账号禁用、会话撤销、密码修改后原会话立即失效。`ACCOUNT_ADMIN` 管理账号但不因此读取知识正文；`PLATFORM_ADMIN` 管理运行配置但不因此读取知识正文。

### 2.2 数据格式与并发

- JSON 使用 UTF-8；编号为 UUIDv7 字符串；时间为带 `Z` 的 UTC ISO-8601 字符串；整数、布尔值和数值不以字符串代替。
- 未提供的可选字段使用默认值；明确可空字段才允许 JSON `null`。服务端拒绝未声明的可写字段、`tenant_id` 及权限/状态投影字段；响应中的可空字段用 `null`，不借缺字段表达不同语义。
- 查询列表使用 `limit`（1～100，默认 20）和不透明 `cursor`；按稳定主键打破同时间排序并列。响应为 `{trace_id, items, next_cursor}`；无下一页时 `next_cursor=null`。检索结果不使用此分页规则。
- 管理详情和任务响应包含 `lock_version`。实体更新、状态切换和删除请求以 `If-Match: "<lock_version>"` 携带目标资源当前版本；会话退出和本人改密按 5.1 节执行，不使用实体版本头。需要版本头却缺失时返回 422 `VERSION_REQUIRED`，过期或并发冲突返回 409 `VERSION_CONFLICT`。服务端同时锁定相关聚合并验证状态、活动索引和墓碑；单个版本号不是发布成功的唯一条件。
- 会产生重复副作用的管理 POST/PATCH/PUT/DELETE 请求必须携带 `Idempotency-Key`（1～128 个可打印非空字符）；登录、本人改密、退出、只读管理检索/检索测试、模型连通性探测以及比较已有评测运行除外。服务端自首次受理起保留至少 24 小时的复用记录，并在成功响应中返回 `idempotency_expires_at`；相同主体、操作和键在该时间前以同请求指纹重放时返回原结果引用，不同指纹返回 409 `IDEMPOTENCY_CONFLICT`。到期后调用方必须先查询业务资源/任务状态，不得把原键重试当作不会重复创建的保证。API Key 完整明文仅首次成功创建响应显示，幂等重放仅返回密钥编号和 `secret_delivered=true`，不得再次返回明文。
- 同一写请求的业务变更、任务/执行记录及 Outbox 在 PostgreSQL 事务内提交；`202 Accepted` 表示已持久化受理，不表示 Worker 完成。客户端通过任务详情查询终态。失败重试不产生第二个可见版本或重复删除。
- 私有管理成功响应至少含 `trace_id` 与操作资源编号；有状态资源同时返回 `status` 和 `lock_version`。创建返回 201，立即完成的读取或状态操作返回 200，异步受理返回 202。异步响应返回对应持久操作编号与状态：入库/删除为 `task_id/task_status`，评测为 `run_id/status`，索引构建为 `index_version_id/status`，块可见性同步为 `chunk_id/sync_status`；没有实际任务时不伪造 `task_id`。

### 2.3 错误模型

错误 JSON 固定为 `{ "trace_id": "UUID", "code": "STABLE_CODE", "message": "简短说明", "retryable": false }`。`trace_id` 由入口生成或沿已建立的请求上下文传递。`message` 不含堆栈、文件路径、密钥、正文、原始查询或资源存在性。MCP 工具错误保存同一 `code` 和 `retryable` 语义，以工具结构化错误返回；协议帧错误与业务错误区分。

| HTTP | 固定语义与主要错误码 | 可重试 |
| --- | --- | --- |
| 400/422 | JSON、参数、过滤 DSL 或资源上限错误；`INVALID_ARGUMENT`、`FILTER_FIELD_INCOMPATIBLE`、`VERSION_REQUIRED` | 否，修改请求后重试 |
| 401 | 会话、API Key 或所属账号无效；`UNAUTHENTICATED` | 否，重新认证 |
| 403 | 范围或状态禁止；外部资源不存在、已删除、无权访问统一 `FORBIDDEN_SCOPE`；管理端权限不足 `FORBIDDEN_SCOPE` | 否 |
| 404 | `RESOURCE_NOT_FOUND`，仅已通过对应管理范围校验的私有管理读取 | 否 |
| 409 | `VERSION_CONFLICT`、`INVALID_STATE`、`INDEX_SNAPSHOT_CONFLICT`、`IDEMPOTENCY_CONFLICT` | 仅冲突可在读取新状态后重试 |
| 429 | `RATE_LIMITED`，API Key/账号共享配额或并发上限 | 是，按限流信息退避 |
| 503 | `RETRIEVAL_UNAVAILABLE`、`RATE_LIMIT_UNAVAILABLE` 或当前操作所需依赖不可用 | 是，按依赖恢复情况 |
| 504 | `DEPENDENCY_TIMEOUT`，本次检索或必需下游超时 | 是，按幂等语义 |

认证和权限失败先于资源详情返回。外部检索及 MCP 对未知、已删除或无权访问的知识库、文档、块统一返回 403，不返回失败编号；多知识库请求任一目标失败，整次不检索。已知 `citation_id` 的受控历史/删除状态例外仅适用引用详情，判定顺序见 3.7 节。无结果是成功响应，不能用 403、503 或空结果掩盖权限/依赖故障。Redis 限流不可用时，对外 REST 与 MCP 返回 503 `RATE_LIMIT_UNAVAILABLE`，不放行请求。

## 3. 对外检索 REST

### 3.1 路由清单

| 方法与路径 | 功能 | 成功 |
| --- | --- | --- |
| `GET /api/v1/retrieval/knowledge-bases` | 当前 API Key 可检索知识库列表 | 200 |
| `POST /api/v1/retrieval/search` | 统一检索 | 200 |
| `GET /api/v1/retrieval/chunks/{chunk_id}` | 当前发布内容块详情 | 200 |
| `GET /api/v1/retrieval/citations/{citation_id}` | 当前引用或受控历史/删除状态 | 200 |

### 3.2 知识库列表

请求查询参数为 `limit`、`cursor`、可选 `name`。仅列出当前 API Key Scope 与所属 API 账号有效 `RETRIEVE` 授权的交集内、状态为 ACTIVE 的知识库；不列出 DRAFT、DISABLED、DELETING、DELETED。每项为 `{knowledge_base_id,name,description}`，不包含文档正文、平台角色或其他知识库标识。响应遵循 2.2 节分页模型；空授权范围返回空列表。

### 3.3 统一检索请求

`POST /api/v1/retrieval/search` 的 JSON 字段如下；MCP `search_knowledge` 复用同一字段、默认值和校验器。

| 字段 | 必填 | 类型与规则 |
| --- | --- | --- |
| `query` | 是 | 非空字符串，最多 4,000 个 Unicode 字符；不在追踪中保存正文 |
| `knowledge_base_ids` | 是 | 1～20 个不重复 UUID；任一目标不存在或无权则整次 403 |
| `document_ids` | 否 | 1～100 个不重复 UUID；每个须属于已授权目标知识库且可用于当前检索；未传表示不限制 |
| `retrieval_mode` | 否 | `KEYWORD/VECTOR/HYBRID`，默认 HYBRID |
| `keyword_candidate_k` | 否 | 整数 1～200，默认 50；KEYWORD/HYBRID 生效 |
| `vector_candidate_k` | 否 | 整数 1～200，默认 50；VECTOR/HYBRID 生效 |
| `rrf_k` | 否 | 整数 1～200，默认 60；仅 HYBRID 使用该调用参数 |
| `rerank` | 否 | 布尔值，默认 true |
| `rerank_threshold` | 否 | 数值 0～1，默认 0.50；仅 Rerank 成功时应用 |
| `top_n` | 否 | 整数 1～50，默认 10 |
| `metadata_filter` | 否 | 第 3.4 节受控业务字段过滤树；未传表示只应用强制过滤 |
| `max_context_tokens` | 否 | 整数 1,000～32,000，默认 8,000 |
| `request_id` | 是 | 非空字符串，最多 128 个字符；用于调用链关联，不作为检索幂等键 |

所有输入先进行认证、参数及全量范围校验，再读取活动快照。服务端固定注入租户、知识库、当前文档版本、活动索引版本、`PUBLISHED`、`chunk_enabled=true` 和有效期条件；客户端不得覆盖。多 Embedding 指纹的 VECTOR 请求按指纹分组召回并以服务端固定 `cross_fingerprint_rrf_k=60` 合并，`effective_fusion=CROSS_FINGERPRINT_RRF`，不读取请求的 `rrf_k`。HYBRID 对全部有效关键词/向量排名列表使用请求的 `rrf_k`。不同指纹的原始向量分数不得直接相加。

### 3.4 业务过滤 DSL

`metadata_filter` 为以下两种节点之一：逻辑节点 `{ "op": "AND|OR", "children": [节点, 节点, ...] }`，或叶子节点 `{ "field": "Schema字段名", "op": "EQ|NE|CONTAINS|GT|GTE|LT|LTE|IN", "value": 值 }`。逻辑节点至少两个子节点；`IN` 的 value 必须是非空数组，其余运算符的 value 为单个与 Schema 类型匹配的值。范围由两个比较叶子组成的 AND 表达。`CONTAINS` 仅用于标记为 `filterable=true` 的 STRING 或 STRING_ARRAY 字段。`valid_from/valid_to` 为 UTC ISO-8601 时间，输入比较按时间类型执行；当前有效区间固定为 `[valid_from,valid_to)`，不接受历史时间点参数。

过滤树最多 5 层、20 个叶子，单个 IN 最多 100 值，单个字符串值最多 256 字符。字段必须在每个目标知识库捕获的 Schema 中存在、类型兼容且允许过滤；否则整次 422 `FILTER_FIELD_INCOMPATIBLE`。不允许权限字段、`tenant_id`、文档/索引状态、任意 SQL 或 Elasticsearch DSL 进入树。相同树分别编译到 Elasticsearch 和 PostgreSQL/pgvector，末端再按 PostgreSQL 权威状态复核。

### 3.5 检索成功响应

根对象固定字段：`trace_id`、`request_id`、`knowledge_base_ids`、`retrieval_mode`、`requested_parameters`、`effective_parameters`、`ignored_parameters`、`effective_fusion`、`per_knowledge_base_context`、`rerank_model_fingerprint`、`tokenizer_version`、`degraded`、`degraded_stage`、`degraded_reason`、`total_duration_ms`、`stage_timings_ms`、`results`、`empty_reason`、`truncated`、`returned_tokens`。`requested_parameters` 保存校验通过后的请求参数及默认值；`per_knowledge_base_context` 对每个实际检索知识库给出 `knowledge_base_id/index_version/metadata_schema_version/embedding_model_fingerprint/visibility_revision`；未使用 Rerank 时 `rerank_model_fingerprint=null`。`results` 始终为数组；有结果时 `empty_reason=null`，无结果时为 `NO_MATCH`、`BELOW_RERANK_THRESHOLD` 或 `TOKEN_BUDGET_EXHAUSTED`。未降级时两个降级字段为 null；降级时均非空且使用稳定阶段/原因码。`effective_parameters` 仅列本次实际执行的参数，`ignored_parameters` 逐项给出已传但不适用的参数及原因；`effective_fusion` 列实际融合方法和常量。`stage_timings_ms` 只包含本次执行阶段的非负毫秒数。

每条 `results` 记录包含：

| 字段组 | 字段和约束 |
| --- | --- |
| 身份与正文 | `knowledge_base_id/name`、`document_id`、`document_version`、`document_name`、`chunk_id`、`parent_chunk_id`、`content`、`citation_id`；父块返回保留 `matched_child_chunk_ids` |
| 得分 | `score`、`score_type`、`keyword_score/rank`、`vector_score/rank`、`rrf_score`、`fusion_rank`、`rerank_logit/score/rank`；未执行的阶段字段为 null |
| 来源 | `page_number`、`slide_number`、`sheet_name`、`title_path`、`source_locator`；来源不适用时对应字段为 null |
| 元数据与预览 | `metadata` 仅含当前 Schema 允许响应的业务字段；`preview_available` 为布尔值，`preview_ref` 为受控引用编号或 null，不包含永久公开地址 |

`score_type` 固定为 `KEYWORD/VECTOR/RRF/RERANK`。Rerank 成功时 `score=rerank_score`，并以 Sigmoid 转换 `rerank_logit`；未启用或降级时分别使用当前模式的关键词、余弦相似度或 RRF 得分，且不执行 `rerank_threshold`。排序并列按稳定 chunk_id；候选先经过权限和状态复核，再映射父块去重，最后按 `top_n` 与 Token 预算裁剪完整块，不截断 Unicode 文本。`truncated` 仅表示因预算或协议限制未返回本可返回的完整块，`returned_tokens` 为实际返回内容的 Token 总数。

KEYWORD 必需的 Elasticsearch 不可用，或 VECTOR 必需的 Embedding/pgvector 不可用时返回 503；HYBRID 仅某一路不可用时按有效列表降级，并设置降级字段；两路均不可用返回 503。Rerank 不可用时保留当前召回/融合排序并标记降级。认证、权限、活动版本和状态过滤失败不得降级。本期只按相同 chunk_id 与父块规则去重；不合并不同来源的相邻块，以免丢失引用位置。取消或超时不伪装空结果。

### 3.6 当前内容块详情

`GET /api/v1/retrieval/chunks/{chunk_id}` 只接受 chunk_id 路径参数，不接受知识库或版本覆盖参数。服务端定位所属知识库后校验 Key/账号范围、当前 ACTIVE 知识库、当前 PUBLISHED 版本、活动 IndexVersion 成员和块启用状态；任一无法确认统一返回 403。成功字段复用 3.5 节的身份、正文、来源和允许响应的元数据，并给出 citation_id、preview_available/preview_ref；不返回检索得分。历史块或已删块不得通过普通 chunk_id 获得正文。

### 3.7 引用详情

`GET /api/v1/retrieval/citations/{citation_id}` 的成功响应为 `{trace_id,citation_id,status,knowledge_base_id,document_id,document_name,document_version,chunk_id,content,source_locator,preview_download_url,preview_expires_at}`。状态只允许 `CURRENT/HISTORICAL_UNAVAILABLE/SOURCE_DELETED`。

- `CURRENT`：再次校验当前账号、Key Scope、直接授权、文档发布状态和块状态；返回当前引用片段与可用来源。仅当前有预览且已获授权时临时生成短期签名 `preview_download_url` 和到期时间，否则两者为 null。
- `HISTORICAL_UNAVAILABLE`：只返回最小来源标识和原版本号；`chunk_id`、`content`、`source_locator`、预览地址及到期时间均为 null。
- `SOURCE_DELETED`：仅已知 citation_id、主体/会话/密钥仍有效、未显式撤权且具备当前授权或删除前保存的同主体历史授权交集时返回最小删除状态；`chunk_id`、`content`、`source_locator`、预览地址及到期时间均为 null。

上述优先拒绝条件或引用不存在/不可确认时统一 403 `FORBIDDEN_SCOPE`。历史/删除状态不能通过知识库列表、检索或普通 chunk_id 探测。签名 URL 不写数据库、检索追踪、审计或业务日志；URL 只在请求时生成。

## 4. MCP 工具契约

`/mcp` 使用基线冻结的 `2026-07-28` 无状态 Streamable HTTP。请求必须携带 Bearer API Key，并校验 `Mcp-Protocol-Version`、`Mcp-Method`、适用时的 `Mcp-Name` 与 JSON-RPC 正文一致；`_meta` 的协议版本、客户端信息和能力声明按本平台契约校验。不执行 `initialize/initialized`，不维护会话 ID，不实现 OAuth 或传统 HTTP+SSE。支持 `server/discover`、`tools/list`、`tools/call`；协议帧由 MCP SDK 编解码，本文只规定工具的业务输入输出和错误映射，不另造协议生命周期。

| 工具 | 输入 | 结构化输出 |
| --- | --- | --- |
| `list_knowledge_bases` | `limit`、`cursor`、可选 `name` | 与 3.2 相同的 `items/next_cursor`，含 trace_id |
| `search_knowledge` | 3.3 的完整搜索输入，含必填 `request_id` | 与 3.5 相同的领域结果，含 `truncated/returned_tokens` |
| `get_chunk` | `chunk_id` 与 `citation_id` 恰有一个非空 | chunk_id 复用 3.6；citation_id 复用 3.7 |

工具 JSON Schema 必须对必填、类型、枚举、默认值、数值范围、互斥参数和 `additionalProperties=false` 作同样约束。`search_knowledge` 的 `metadata_filter` 也执行 3.4 节业务校验。调用方无需解析自然语言文本取得结果。MCP 错误对象保留 2.3 节稳定错误码、可重试语义和 trace_id；协议无效与业务拒绝分别标识。HTTP 与 MCP 使用相同 Key/账号限流计数和脱敏追踪；客户端断开向下游检索和 Rerank 传播取消信号。

### 4.1 Python 与框架适配

轻量 Python 客户端提供同步和异步 `retrieve(query, knowledge_base_ids, filters, options)`；`filters` 映射 3.4 节，`options` 只能映射 3.3 节声明的可选字段，客户端生成必填 `request_id`，不在本地执行 Embedding 或直接访问检索存储。默认走对外 REST；调用方显式选择 MCP 时只换传输层，不改变领域请求、权限、限流、结果或错误映射。两种传输均将 `code/trace_id/retryable` 暴露为结构化错误，不把 403、503 或 504 当作空结果。

LangChain `RagBaseRetriever` 支持 `invoke/ainvoke`，把每条结果转换为 `Document`：`page_content=content`，`metadata` 至少包含知识库/文档/版本/块编号、来源页码或表格位置、`score/score_type`、`citation_id` 和 `trace_id`。LangGraph 异步检索节点只输出 Document 列表、引用列表与 trace_id，调用方自行定义业务 State、路由和会话。集成包与服务端独立发布，声明 Python、`langchain-core`、LangGraph 和服务端兼容范围；发布前对所声明最低/最高版本分别验证 REST/MCP 映射。示例 Key 只能用占位值。

## 5. 私有管理 REST

本章路径均以 `/internal/api/v1` 为根，表内省略该前缀。管理接口只供 React 前端在公司内网使用，使用人工账号会话和 CSRF；不接受 API Key。每个接口在独立内部 OpenAPI 声明参数、请求/响应 Schema、平台角色、知识库权限、前置/后置状态、错误码和 `If-Match`/`Idempotency-Key` 要求。下表的 `USER` 表示持有该全局角色的人工账号；知识库权限互不隐含。

路径中的 `kb_id`、`document_id`、`version_id`、`chunk_id`、`task_id`、`run_id` 均为 UUIDv7。列表查询采用 2.2 节分页；嵌套资源必须验证与路径上级实体的真实归属。账号管理员即使能查看授权关系，也不能因为平台角色而查看知识正文。更新/删除同一实体时使用该实体的 `lock_version`；父子发布还按知识库活动索引、可见性代次和墓碑执行权威 CAS。

### 5.1 会话与人工账号

| 方法与路径 | 请求核心字段 | 成功与权限 |
| --- | --- | --- |
| `POST /sessions` | `email`、`password` | 200；邮箱按去首尾空白和大小写规范化；成功设置会话 Cookie 并返回 `csrf_token`、`expires_at`、`must_change_password`；失败统一 401，不暴露邮箱存在性 |
| `GET /sessions/current` | 无 | 200；当前人工账号编号、角色、状态、到期时间及 `must_change_password` |
| `DELETE /sessions/current` | CSRF | 200；撤销当前会话，重复退出仍返回成功；不需要 `If-Match` |
| `POST /sessions/current/password` | `current_password`、`new_password`、CSRF | 200；本人改密，临时密码会话只允许此操作；撤销包括本次在内的全部旧会话并清除 Cookie，须用新密码重新登录；不返回密码 |
| `GET /users`、`GET /users/{user_id}` | 列表筛选邮箱/状态 | 200；`ACCOUNT_ADMIN`；列表/详情只含账号资料、状态、角色和最后登录时间，不含密码哈希或会话凭证 |
| `POST /users` | `email`、`display_name`、全局角色集合 | 201；`ACCOUNT_ADMIN`；默认无自助注册，邮箱规范化后唯一，一次性临时密码仅首次成功响应交付，首次登录必须改密 |
| `PATCH /users/{user_id}` | `display_name`、全局角色集合 | 200；`ACCOUNT_ADMIN`，角色变更后权限版本立即更新；需 `If-Match` |
| `POST /users/{user_id}/disable`、`POST /users/{user_id}/enable`、`POST /users/{user_id}/unlock` | 无 | 200；`ACCOUNT_ADMIN`；disable 为 ACTIVE/LOCKED→DISABLED，enable 为 DISABLED→ACTIVE，unlock 为 LOCKED→ACTIVE；禁用立即撤销会话，恢复不自动恢复旧会话；需 `If-Match` |
| `POST /users/{user_id}/reset-password` | 无 | 200；`ACCOUNT_ADMIN`；`temporary_password` 只在本次成功响应展示一次，强制首次改密并撤销既有会话；需 `If-Match` |
| `POST /users/{user_id}/sessions/revoke` | 无 | 200；`ACCOUNT_ADMIN`；强制下线，已有会话立即失效；需 `If-Match` |
| `GET /users/{user_id}/permissions` | `limit`、`cursor` | 200；`ACCOUNT_ADMIN` 可看账号授权关系，当前主体也可看自身授权；不返回知识正文 |

账号状态只允许 ACTIVE、LOCKED、DISABLED，临时锁定由登录安全策略控制。管理员恢复 LOCKED/禁用状态时必须经过已授权管理操作并审计；API 不接受客户端直接写 `failed_login_count`、`locked_until` 或 `authorization_version`。管理员重置/创建凭证的幂等重放不能重新展示一次性秘密；如首次响应丢失，须再次执行新的受控重置。密码至少符合当前版本化安全策略，默认最小长度 8；任何接口不接受或返回密码哈希。

### 5.2 API 调用账号、API Key 与直接授权

| 方法与路径 | 请求核心字段 | 成功与权限 |
| --- | --- | --- |
| `GET /api-accounts`、`GET /api-accounts/{account_id}` | 列表名称/状态 | 200；`ACCOUNT_ADMIN`；返回用途、负责人、状态、最后调用时间 |
| `POST /api-accounts` | `name`、`purpose`、`owner_user_id` | 201；`ACCOUNT_ADMIN`；初始 ACTIVE，不自带知识库权限 |
| `PATCH /api-accounts/{account_id}` | `name`、`purpose`、`owner_user_id` | 200；`ACCOUNT_ADMIN`，需 `If-Match` |
| `POST /api-accounts/{account_id}/disable`、`POST /api-accounts/{account_id}/enable` | 无 | 200；`ACCOUNT_ADMIN`，需 `If-Match`；禁用立即使全部 Key 失效 |
| `GET /api-accounts/{account_id}/keys` | `limit`、`cursor` | 200；`ACCOUNT_ADMIN`；仅返回 Key 编号、名称、脱敏前缀、状态、创建/最后使用时间、Scope 编号和 `lock_version`，不返回完整 Key |
| `POST /api-accounts/{account_id}/keys` | `name`、`knowledge_base_ids` | 201；`ACCOUNT_ADMIN`；Scope 必须属于账号当前有效 RETRIEVE 授权；只在首次响应返回完整 `api_key`，并返回 `key_id` |
| `POST /api-keys/{key_id}/disable`、`DELETE /api-keys/{key_id}` | 无 | 200；`ACCOUNT_ADMIN`，需 `If-Match`；立即失效，不物理删除历史审计引用 |
| `PUT /api-keys/{key_id}/scope` | `knowledge_base_ids` | 200；`ACCOUNT_ADMIN`，需 `If-Match`；整体替换 Key 自身范围，不能超过账号当前 RETRIEVE 授权 |
| `GET /api-accounts/{account_id}/permissions` | `limit`、`cursor` | 200；`ACCOUNT_ADMIN`；仅返回直接授权关系、授权编号及 `lock_version`，不返回正文 |
| `GET /knowledge-bases/{kb_id}/permissions` | `limit`、`cursor` | 200；该库 `MANAGE`；按库查看同一授权事实，返回授权编号及 `lock_version` |
| `POST /knowledge-bases/{kb_id}/permissions` | `subject_type` 取 USER 或 API_ACCOUNT、`subject_id`、`permission_codes` | 201；该库 `MANAGE`；API_ACCOUNT 只允许 RETRIEVE，重复有效授权合并或 409，不能产生冲突记录 |
| `DELETE /knowledge-bases/{kb_id}/permissions/{grant_id}` | 无 | 200；该库 `MANAGE`，需 `If-Match`；显式撤销并写审计，历史删除引用授权也同步失效 |

轮换 Key 使用“创建新 Key → 调用方切换 → 显式禁用旧 Key”；新旧在切换窗口同时有效且没有时间自动过期。API Key 的实际可检索范围始终是 Key Scope 与所属 API 账号当前直接 RETRIEVE 授权的交集；账号授权撤销后，即使 Key Scope 尚有旧记录也不能继续访问。管理前端不能复制或再次读取完整 Key。

### 5.3 知识库、Schema 与分块配置

| 方法与路径 | 请求核心字段 | 成功与权限 |
| --- | --- | --- |
| `GET /knowledge-bases`、`GET /knowledge-bases/{kb_id}` | 列表名称/状态 | 200；仅返回当前人工账号有任一直接权限且状态为 DRAFT/ACTIVE/DISABLED 的库；详情不因为 PLATFORM_ADMIN/ACCOUNT_ADMIN 自动展示正文 |
| `POST /knowledge-bases` | `name`、`description`、`default_language`、初始分块策略/参数 | 201；全局 `USER`；创建 DRAFT、初始配置和创建者四项直接权限同事务完成；授权其他主体须使用授权接口 |
| `PATCH /knowledge-bases/{kb_id}` | `name`、`description`、`default_language` | 200；该库 `MANAGE`，需 `If-Match`；名称全平台唯一，不能通过此接口直接改活动索引/模型指纹 |
| `POST /knowledge-bases/{kb_id}/enable`、`POST /knowledge-bases/{kb_id}/disable` | 无 | 200；该库 `MANAGE`，需 `If-Match`；DRAFT/ACTIVE/DISABLED 按合法状态迁移，恢复前校验活动索引或空库条件 |
| `POST /knowledge-bases/{kb_id}/deletion` | `confirmation_name`，须与当前名称一致 | 202；该库 `MANAGE`，需 `If-Match`；二次确认前页面展示文档数、块数、已授权 Key 数；事务落墓碑/DeletionTask/Outbox 并立即阻断检索 |
| `GET /knowledge-bases/{kb_id}/schemas`、`GET /schemas/{schema_id}` | `limit`、`cursor` | 200；该库 `MANAGE` 或 `EDIT`；返回版本、状态、字段类型/枚举/必填、可覆盖和可响应属性 |
| `POST /knowledge-bases/{kb_id}/schemas` | `name`、可选 `template_code`、`fields[]` | 201；该库 `MANAGE`；创建 DRAFT 新版本，已用于索引的字段类型变化必须用新字段名 |
| `POST /schemas/{schema_id}/activate` | 无 | 200；该库 `MANAGE`，需 `If-Match`；切换当前 Schema；已发布快照仍使用其冻结 Schema，需重建的变化通过新 IndexVersion 发布 |
| `GET /knowledge-bases/{kb_id}/chunking-configs` | `limit`、`cursor` | 200；该库 `MANAGE` 或 `EDIT`；每项包含配置编号、版本、参数与 `lock_version` |
| `POST /knowledge-bases/{kb_id}/chunking-configs` | `strategy`、`target_tokens`、`max_tokens`、`overlap_tokens`、`separators`、策略专属参数 | 201；该库 `MANAGE`；创建不可变配置版本，目标≤最大、重叠小于目标 |
| `POST /chunking-configs/{config_id}/activate` | 无 | 200；该库 `MANAGE`，需 `If-Match`；仅影响后续处理；已发布块不原地重切 |
| `GET /knowledge-bases/{kb_id}/index-versions`、`GET /index-versions/{index_version_id}` | 状态筛选/分页或单项 | 200；该库 `MANAGE`；只返回版本、绑定的 Schema/分块/Embedding 指纹、基础活动版本、校验摘要、`publishable`、状态与 `lock_version`，不返回正文 |
| `POST /knowledge-bases/{kb_id}/index-versions/rebuild` | `reason` 取 EMBEDDING_UPGRADE、SCHEMA_REINDEX、INDEX_REPAIR 或 HIDDEN_MEMBER_RESTORE；按原因提交目标字段 | 202；该库 `MANAGE`；从当前权威文档版本和不可变内容块构建新的完整 BUILDING 快照，返回 `index_version_id/status`；不自动发布，不用于改变分块正文 |
| `POST /index-versions/{index_version_id}/activate` | 无 | 200；目标库 `MANAGE`，需目标 IndexVersion 的 `If-Match`；知识库须 ACTIVE，只受理上一行纯重建形成的 READY 快照，不得包含待发布的新文档内容版本；`publishable=true` 且双索引证据完整时，复核基础活动版本、可见性代次和墓碑并在短事务内切换；含新文档版本的 READY 快照只能由目标文档的 `EDIT` 授权经 `/document-versions/{version_id}/publish` 发布；过期版本 409 且不可重复激活 |
| `GET /index-versions/{index_version_id}/rollback-preview`、`POST /index-versions/{index_version_id}/rollback` | 预览无请求体；回滚提交 `confirmation_name`、预览返回的 `affected_documents_digest` 与 `expected_active_index_version_id` | 200；目标库 `MANAGE`；仅可用 RETIRED 完整快照，提交需目标库 `If-Match`；先列出全部受影响文档供二次确认，再复核墓碑、可见性和双索引，在同一事务同步文档发布指针及活动快照；冲突 409，不允许借此恢复已删除内容 |

`POST /knowledge-bases/{kb_id}/deletion` 对含 UPLOADED 版本的目标仍可受理；初始事务立即屏蔽检索，关联入库任务按取消路径停止领取或写入，待租约失效后再按 `UPLOADED→DELETING→DELETED` 清理。失败保留墓碑与任务，状态由删除任务接口查询。Schema 字段停用须创建新 Schema 版本；K12 字段仅为可选模板。客户端不能提交 `active_index_version_id`、`visibility_revision` 或 Embedding 模型名来绕过受控发布。

重建请求按 `reason` 限定参数：`EMBEDDING_UPGRADE` 必填经平台核验的 `target_embedding_fingerprint_id`；`SCHEMA_REINDEX` 必填目标库当前 ACTIVE 的 `target_schema_version_id`；`INDEX_REPAIR` 不接受目标配置，沿用当前活动快照的指纹和 Schema；`HIDDEN_MEMBER_RESTORE` 必填同库、当前 MANUAL DISABLED 且不在活动快照中的 `target_document_version_id`，重建时仍保持该成员不可见。互不适用的字段返回 422。四种重建均捕获当前活动版本作为并发基线，不得直接覆盖线上索引或改变文档发布指针。回滚预览返回全部受影响文档编号/名称、`affected_documents_digest`、`expected_active_index_version_id` 与当前 `visibility_revision`；提交时任一值已变化须重新预览并返回 409。

分块配置变化时，已发布文档不得只换配置编号而保留旧块；对受影响文档调用 5.4 节重处理接口，基于已校验原文件生成新草稿，界面先提示已有人工修改需在新草稿重新合并，重新解析、分块和双索引校验完成后仍须人工发布。`index-versions/rebuild` 只复用已有不可变内容块，不能代替这一重处理流程。Embedding 指纹升级通过全库快照重建新指纹向量；新快照激活时，已核验的对应本地运行配置与活动索引在同一事务生效，旧指纹配置留给在途请求和保留快照，任一失败保持旧配置与旧快照。Schema 切换不改写已发布快照；需要让已有内容按新 Schema 检索时使用受控重建并校验所有成员的过滤字段。

`fields[]` 的每项固定为 `field_name`、`display_name`、`data_type`、`required`、`filterable`、`response_visible`、`chunk_overridable`、`status`、`validation_rule` 和 `sort_order`；`data_type` 只允许 `STRING/NUMBER/BOOLEAN/DATE/DATETIME/STRING_ARRAY`，`status` 只允许 `ACTIVE/DISABLED`。同一版本字段名不得重复或使用平台保留名；`validation_rule` 仅接受与字段类型相容的枚举、长度或数值范围约束。上传或人工改写元数据时，按该文档版本绑定的 Schema 校验字段名、类型、必填项与约束；块级覆盖只接受 `chunk_overridable=true` 的字段。过滤与响应分别只使用 `filterable`、`response_visible` 的活动字段。分块 `strategy` 只允许 `GENERAL/SECTION/QA/TABLE/PARENT_CHILD`；目标/最大/重叠 Token 数均为非负整数且目标与最大值为正数，`target_tokens≤max_tokens`、`overlap_tokens<target_tokens`。`separators` 为有序非空字符串数组；`PARENT_CHILD` 另要求父块目标 Token 数大于子块目标 Token 数。配置创建后不可改写，修改需创建新版本；新策略对已发布内容生效须重建并发布完整索引快照。

### 5.4 文档、任务、内容块和预览

| 方法与路径 | 请求核心字段 | 成功与权限 |
| --- | --- | --- |
| `GET /knowledge-bases/{kb_id}/documents`、`GET /documents/{document_id}` | 列表名称/版本状态 | 200；该库 `EDIT`；仅当前权限允许的文档资料，不向无权主体暴露存在性 |
| `GET /knowledge-bases/{kb_id}/ingestion-tasks` | 状态筛选、分页 | 200；该库 `EDIT`；列出该库入库任务编号、状态、阶段和稳定错误码 |
| `POST /knowledge-bases/{kb_id}/documents/uploads` | `multipart/form-data`：`file`、`metadata` JSON、可选文档名 | 202；该库 `EDIT`；服务端校验当前有效单文件上限（默认 200 MB、仅可下调，服务端硬上限 200 MB）、格式/声明 MIME/空文件，先创建上传预留再流式写入七牛云暂存对象；返回新文档、UPLOADED 版本和入库 task_id |
| `POST /documents/{document_id}/versions/uploads` | 同上，另有 `change_reason` | 202；该库 `EDIT`，需文档 `If-Match`；创建 REPLACE 草稿，不影响当前 PUBLISHED 版本 |
| `POST /documents/{document_id}/versions/reprocess` | 必填非空 `change_reason` | 202；该库 `EDIT`，需文档 `If-Match`；从当前 PUBLISHED 版本沿同文档来源链定位并校验 FORMAL ORIGINAL，链缺失、成环、跨文档或命中墓碑时拒绝；创建 REPROCESS/UPLOADED 草稿及入库 task_id，冻结知识库当前解析/分块/Schema 配置，不重复上传或覆盖原文件；处理前告知已有人工修改需在新草稿重新合并，完成后仍走 REVIEW_REQUIRED 审核发布 |
| `GET /documents/{document_id}/versions`、`GET /document-versions/{version_id}` | 无 | 200；该库 `EDIT`；返回版本、状态、元数据、质量告警、任务编号和引用，不直接返回可恢复历史正文 |
| `GET /ingestion-tasks/{task_id}` | 无 | 200；目标库 `EDIT`；状态、阶段、进度、次数、稳定错误、开始/结束时间和执行历史，不返回租约秘密 |
| `POST /ingestion-tasks/{task_id}/cancel` | 无 | 无派生数据且未运行时 200/CANCELLED；已有派生数据时 202/CANCEL_REQUESTED 或 CLEANUP_PENDING；目标库 `EDIT`，需 `If-Match`；清理完成前不得称为 CANCELLED |
| `POST /ingestion-tasks/{task_id}/retry` | 无 | 202；目标库 `EDIT`，需 `If-Match`；FAILED→PENDING 创建新的 QUEUED 入库执行，CLEANUP_FAILED→CLEANUP_PENDING 只创建清理执行；RETRY_WAIT 由 Scheduler 按到期时间重投，保留历史执行 |
| `POST /document-versions/{version_id}/publish` | `ready_index_version_id` | 200；目标库 `EDIT`，需 `If-Match`；仅 REVIEW_REQUIRED 且 READY 双索引校验通过，按活动版本/visibility CAS 切换，冲突 409 |
| `POST /document-versions/{version_id}/disable`、`POST /document-versions/{version_id}/restore` | 无 | 200；目标库 `EDIT`，需 `If-Match`；停用先更新权威状态并立即阻断新检索；仅当前 MANUAL DISABLED 版本且仍为活动快照隐藏成员时，恢复接口先同步并验证双索引可见性投影、再提交 PUBLISHED；同步失败仍为 DISABLED；缺少隐藏成员返回 409，须另行构建完整快照；SUPERSEDED 只能走显式回滚 |
| `POST /documents/{document_id}/rollback` | `target_version_id` | 202；目标库 `EDIT`，需文档 `If-Match`；返回新建 `index_version_id/status`，基于当前活动快照构建新完整快照，校验后发布；不复活墓碑内容 |
| `POST /documents/{document_id}/deletion` | `confirmation_name`，须与当前文档名一致 | 202；目标库 `EDIT`，需文档 `If-Match`；初始事务写墓碑并立即屏蔽目标，异步清理全部版本/对象/索引 |
| `GET /deletion-tasks/{task_id}` | 无 | 200；目标库仍存在时校验 `MANAGE`（库删除）或 `EDIT`（文档删除）；库已删除时仅删除发起人本人或 `PLATFORM_ADMIN` 可看最小任务进度，绝不返回正文/对象键；返回阶段、进度、状态、稳定错误和脱敏清理摘要 |
| `POST /deletion-tasks/{task_id}/retry` | 无 | 202；目标库仍存在且调用方具有对应 `MANAGE/EDIT` 时可用；需 `If-Match`，仅 FAILED 可人工重试，复用墓碑并建新执行记录；库已完成删除时任务不可能仍为 FAILED，此接口不提供已删除库的重试特例 |
| `GET /document-versions/{version_id}/chunks`、`GET /chunks/{chunk_id}` | 分页或单块 | 200；目标库 `EDIT`；当前 REVIEW_REQUIRED 草稿或 PUBLISHED 版本可读受控正文并审计；历史 DISABLED/DELETED 仅返回来源标识与状态，不返回旧正文或预览 |
| `POST /chunks/{chunk_id}/draft-edit` | 可选 `content`、`manual_keywords`、允许覆盖的 `metadata`、`change_reason` | 202；目标库 `EDIT`，需当前 PUBLISHED 源块 `If-Match`；复制整个当前发布版本成新草稿，只在副本修改并为变化块重算向量 |
| `POST /chunks/{chunk_id}/disable`、`POST /chunks/{chunk_id}/enable` | 无 | 停用 200、启用 202；目标库 `EDIT`，需 `If-Match`；停用先写权威 false，启用返回 `chunk_id/sync_status`，双索引验证前保持 false，完成后通过块详情确认启用 |
| `GET /documents/{document_id}/preview` | 可选当前发布版本的预览定位 | 200；目标库 `RETRIEVE` 且版本/状态当前有效，或 `EDIT` 的受控管理预览；临时签发短期地址与到期时间，记录敏感访问审计 |

上传响应的 `202` 只确认数据库已保存 UploadReservation、DocumentVersion、DocumentObject(STAGED)、IngestionTask、QUEUED TaskExecution 和 Outbox；深度文件特征、安全、解压、页数及格式专属上限由 Worker 验证，失败反映在任务状态，不返回伪造成功索引。正式对象未验证前不得解析为可发布内容。EML 附件形成独立文档关系和任务；批量上传由管理前端逐文件调用单文件上传接口并分别展示成功/失败，单文件失败不回滚其他文件。

文档状态只使用 DocumentVersion 的 `UPLOADED/PROCESSING/REVIEW_REQUIRED/PUBLISHED/DISABLED/FAILED/DELETING/DELETED`；Document 本身不另设状态。删除中的知识库/文档以及人工停用块立即在权威状态层阻断新检索。`restore` 仅适用 `DISABLED` 且 disable_reason=MANUAL 的当前版本；目标库 DISABLED、墓碑存在或活动快照缺失所需成员时拒绝直接恢复。`rollback` 是独立的版本化发布流程，不直接把历史 SUPERSEDED 版本改为线上版本。

### 5.5 管理检索、评测和门禁

| 方法与路径 | 请求核心字段 | 成功与权限 |
| --- | --- | --- |
| `POST /knowledge-bases/{kb_id}/search` | 3.3 字段，knowledge_base_ids 固定只含路径库 | 200；该库 `RETRIEVE`；仅活动快照，结果同 3.5，人工会话产生脱敏追踪 |
| `POST /knowledge-bases/{kb_id}/retrieval-tests` | 3.3 字段及可选 `ready_index_version_id` | 200；该库 `EVALUATE`；选 READY 快照只用于测试，返回各阶段候选、分数、过滤原因和 `test_snapshot`，不改活动版本 |
| `GET /knowledge-bases/{kb_id}/evaluation-sets`、`GET /evaluation-sets/{set_id}` | 列表/详情 | 200；该库 `EVALUATE`；评测问题、期望目标和片段访问受控、审计 |
| `POST /knowledge-bases/{kb_id}/evaluation-sets` | `name`、描述 | 201；该库 `EVALUATE`，同事务创建评测集和初始 DRAFT 版本，返回 `set_id/set_version_id` |
| `GET /evaluation-sets/{set_id}/versions`、`POST /evaluation-sets/{set_id}/versions` | 列表分页；创建时可选 `source_version_id` | 200/201；该库 `EVALUATE`；创建需评测集 `If-Match`，新建递增 DRAFT 版本，源版本须属于同一评测集且未清除问题；传入源版本时复制其用例及标注，不传时创建空版本，不改动原版本 |
| `POST /evaluation-set-versions/{set_version_id}/cases` | `query_text`、`expected_targets`、可选 `metadata_filter/notes` | 201；该库 `EVALUATE`，需 DRAFT 版本 `If-Match`；`expected_targets` 每项固定目标类型 DOCUMENT/CHUNK、同库目标编号、相关等级和粒度 |
| `PATCH /evaluation-cases/{case_id}`、`DELETE /evaluation-cases/{case_id}` | 可修改问题、期望目标、业务过滤或备注；删除无请求体 | 200；该库 `EVALUATE`，仅 DRAFT 版本可操作，需用例 `If-Match`；删除仅从该草稿移除，不改变任何冻结版本 |
| `POST /evaluation-set-versions/{set_version_id}/cases/import` | CSV 文件 | 200；该库 `EVALUATE`，需 DRAFT 版本 `If-Match`；逐行校验问题、目标、同库归属和重复项；全部通过后同事务导入并返回导入数量，任一错误返回 422 和脱敏行号/错误码，不产生部分用例 |
| `POST /evaluation-set-versions/{set_version_id}/freeze` | 无 | 200；该库 `EVALUATE`，需版本 `If-Match`；把指定 DRAFT 版本冻结，不覆盖既有 FROZEN 版本 |
| `GET /evaluation-set-versions/{set_version_id}` | 无 | 200；该库 `EVALUATE`；返回指定版本状态、问题、目标和标注，DRAFT/FROZEN 均只读返回 |
| `POST /knowledge-bases/{kb_id}/evaluation-runs` | `set_version_id`、`parameter_set_id`、可选经核验 `rerank_config_id` | 202；该库 `EVALUATE`；评测集版本必须为本库 FROZEN、参数集必须为 FROZEN 且属于本库或通用配置；冻结活动 IndexVersion/输入/模型与代码版本，生成 run_fingerprint，返回 `run_id/status`；候选 Rerank 配置可为 DRAFT |
| `GET /evaluation-runs/{run_id}`、`GET /evaluation-runs/{run_id}/results` | 无或分页 | 200；该库 `EVALUATE`；状态、指标、错误和逐题标识/得分，正文片段只在受控查看时返回并审计 |
| `POST /evaluation-runs/compare` | 两个 `run_id` | 200；两运行的知识库均须有 `EVALUATE`；比较冻结输入和指标，不改变历史运行 |
| `GET /knowledge-bases/{kb_id}/parameter-sets`、`POST /knowledge-bases/{kb_id}/parameter-sets` | 列表；创建时 `name`，以及 3.3 的 `retrieval_mode/keyword_candidate_k/vector_candidate_k/rrf_k/rerank/rerank_threshold/top_n/max_context_tokens` | 200/201；该库 `EVALUATE`；参数集只保存检索参数，不保存 query、目标编号或 request_id；仅供测试/评测，不改变在线请求默认值 |
| `POST /parameter-sets/{parameter_set_id}/freeze` | 无 | 200；该库 `EVALUATE`，需 `If-Match`；冻结后内容不可修改 |
| `POST /release-gates/initial-production` | `run_id`、`evidence_ref` | 201；对验收知识库有 `EVALUATE` 的技术验收人员；复核批准脱敏语料、不少于 100 题、默认混合参数、Recall@10≥0.80、MRR@10≥0.60、权限用例 100%；写不可覆盖的 PASSED/FAILED 门禁记录和审批人，FAILED 不允许首次生产发布 |

管理检索不把 `EVALUATE` 当作 `RETRIEVE`；单条测试只可选择同库 READY 或当前活动快照，不能选择 BUILDING、FAILED、RETIRED 或 DELETED。评测集版本、参数集、索引版本、Rerank 候选配置在运行启动后固定，失败不能算作零分成功用例；运行失败保留历史，重跑创建新运行。门禁只约束平台首次上线，不反向阻断已上线环境的业务文档发布。Rerank 升级使用相同冻结验收知识库、IndexVersion、默认混合参数集和不少于 100 题比较新旧版本；候选配置一经评测引用，其模型身份、服务地址引用和运行参数不可改写。

### 5.6 平台运行与追踪

| 方法与路径 | 请求核心字段 | 成功与权限 |
| --- | --- | --- |
| `GET /operations/overview` | 无 | 200；`PLATFORM_ADMIN`；任务各状态数量、队列积压、文档/块数、索引大小与存储量、检索 QPS/成功率/P50/P95/P99、Embedding/搜索/Rerank 阶段耗时和错误率、HTTP/MCP 分项调用指标、热门知识库及无结果率，不含正文 |
| `GET /operations/health` | 无 | 200；`PLATFORM_ADMIN`；按 API/MCP/Worker/Scheduler 与 PostgreSQL/pgvector、ES、Redis、七牛云、本地模型报告能力级 READY/DEGRADED/UNREADY |
| `GET /operations/alerts` | 类型/严重度/时间筛选、分页 | 200；`PLATFORM_ADMIN`；返回现有监控告警的编号、来源、严重度、时间、状态和脱敏摘要，不返回正文或密钥；本接口不新增业务告警写入或确认流程 |
| `GET /operations/traces`、`GET /operations/traces/{trace_id}` | 脱敏筛选/分页 | 200；`PLATFORM_ADMIN`；请求指纹、授权后知识库编号、命中编号、耗时、降级和错误，不返回原始 query |
| `GET /operations/ingestion-tasks`、`GET /operations/deletion-tasks` | 状态/时间筛选 | 200；`PLATFORM_ADMIN`；只显示任务编号、状态、阶段和脱敏错误，正文查看仍按知识库权限 |
| `GET /operations/model-configs`、`GET /operations/model-configs/{config_id}` | 模型类型筛选 | 200；`PLATFORM_ADMIN`；模型名称、版本、文件摘要只读展示，不返回密钥 |
| `POST /operations/model-configs` | 固定模型类型、已核验部署基线引用、服务地址引用、超时/并发/批次/设备参数 | 201；`PLATFORM_ADMIN`；创建 DRAFT 配置，不能选择任意外部模型供应商 |
| `POST /operations/model-configs/{config_id}/probe` | 无 | 200；`PLATFORM_ADMIN`；只对公司内已配置服务做连接、模型身份和最小推理检查，返回 `reachable/identity_matched/inference_passed/duration_ms` 及脱敏错误码；不激活配置，不保存测试正文 |
| `POST /operations/model-configs/{config_id}/activate` | Rerank 候选必须提供 `evaluation_run_id`；Embedding/OCR 不接受该接口 | 200；`PLATFORM_ADMIN`，需 `If-Match`；仅 Rerank 按第 5.5 节质量/权限门槛切换查询期配置；Embedding 运行配置随对应指纹的索引构建/发布路径受控生效，不提供直接切换模型身份的接口 |

Rerank 激活事务复核 `evaluation_run_id` 的 SUCCEEDED 状态、候选 config_id、模型文件摘要、冻结评测集、索引和参数及门槛；先退役旧 ACTIVE，再激活新配置，失败整体回滚。新请求捕获新配置编号，已有请求使用旧编号；旧配置保留到在途请求和评测完成。性能对比记录同环境 P50/P95/P99，不另设本期数值型切换门槛。Embedding 激活不提供绕过重建的直接模型切换路径。运维接口不允许直接修改模型名称、版本、文件摘要、向量维度或已被评测引用的配置字段。

## 6. 契约核验清单

| 验证点 | 必须得到的接口结果 | 依据 |
| --- | --- | --- |
| HTTP/MCP 同一 Key、查询和参数 | 同样的授权结论、排序/得分语义、引用与错误码，限流计数共享 | FR-API-001～005；FR-MCP-001～003；03 第 9 章 |
| 多知识库任一无权/不存在 | 整次 403，无部分结果和失败编号 | C-AUTH-08；FR-AUTH-002 |
| KEYWORD/VECTOR/HYBRID 与依赖故障 | 单路可用时仅 HYBRID 降级；必需依赖均不可用返回 503 | FR-SEARCH-004/008；03 第 8.1 节 |
| Rerank 开关、阈值和得分 | 仅成功重排时用阈值与 RERANK 得分；降级保留原模式排序 | FR-SEARCH-005；03 第 8.3 节 |
| 元数据 DSL 与权限覆盖 | 无效/跨库不兼容 422，不能覆盖强制过滤 | FR-SEARCH-006；C-META-03 |
| 当前块、历史/已删除引用 | 普通块只返回当前可见正文；历史/删除引用仅返回允许的最小状态 | FR-CITE-001/002；03 第 4.8 节 |
| 上传、取消、重试、发布 | 202 对应持久任务；旧 Worker 不能提交；并发发布 409 且旧版本可用 | FR-INGEST-001/004；FR-INDEX-002；04 第 7、9 章 |
| UPLOADED 删除 | 墓碑立即阻断；取消关联任务；对象及派生数据清理完成才 DELETED | C-DOC-07；03 第 5.5 节 |
| 内容块停用/启用 | 停用立即不可见；启用双索引校验前不对外宣称成功 | C-DOC-06；03 第 4.7 节 |
| Rerank 升级 | 冻结候选配置、评测达标、事务切换，新旧在途版本分离且不重建索引 | C-SEARCH-09；03 第 4.6 节 |
| 密钥与会话 | Key 明文只展示一次；禁用立即失效；临时密码必须改密；角色不获正文权限 | FR-AUTH-001～006；04 第 6.2～6.9 节 |

本文仅完成接口契约设计。验收仍须以冻结后的 OpenAPI/MCP Schema、最小/最高兼容版本真实调用、浏览器联调、数据库迁移和完整业务链路测试为证；文档结构检查不能代替运行验证。

## 7. 关联文档

- [RAG 知识库平台需求说明书 V1.25](./01-RAG知识库平台需求说明书.md)
- [RAG 知识库平台需求确认单 V1.4](./02-RAG知识库平台需求确认单.md)
- [RAG 知识库平台系统设计说明书 V1.7](./03-RAG知识库平台系统设计说明书.md)
- [RAG 知识库平台数据库设计说明书 V1.3](./04-RAG知识库平台数据库设计说明书.md)
