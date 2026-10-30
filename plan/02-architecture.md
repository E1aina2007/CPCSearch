# 架构设计

更新日期：2026-10-30。状态：逻辑设计，生产 DDL 与运行行为待核验。返回 [计划目录](README.md)。

## 1. 部署与模块边界

继续使用现有 Spring Boot 应用、MySQL 和 Vue 工程。后端复用冒险家协会的只读搜索、数据查询与 OJ Mapper；前端新增独立推搜功能，不挂载在协会布局下。默认不增加服务进程、前端工程或通用 Agent 平台，Redis 仅在实测需要时缓存可重建结果。

~~~mermaid
flowchart TD
    UI["features/recommendations 独立推搜页面"] --> C["Controller + 现有鉴权"]
    C --> S["共享 SearchService"]
    C --> R["RecommendationService"]
    R --> S
    R --> P["DataQueryService / Profile DTO"]
    C --> H["Hot 查询与题单反馈"]
    P --> DB["MySQL"]
    S --> DB
    R --> DB
    H --> DB
    IN["现有 CF 采集 / 洛谷上传"] --> V["入库校验与幂等写入"]
    V --> DB
    CL["独立 CleaningService / Spring 调度"] --> DB
    AU["二期对话入口"] -.-> A["AssistantService"]
    A -.-> R
    A -.-> S
    A -.-> H
    A -.-> MC["ModelClient"]
    MC -.-> API["供应商 API"]
    A -.-> DB
~~~

实线为一期基线，虚线为二期 Agent。Agent → 推搜 Service 是单向依赖；基线推搜不依赖会话、供应商 Key 或模型可用性。外部采集适配完成后写本地库，推荐请求不实时访问 OJ。可选语义支路另见本节末尾，不作为图中实线的启动依赖。

| 责任 | 现有落点与建议改造 |
| --- | --- |
| 用户身份 | 复用 AuthInterceptor / AuthContextHolder；业务关联 user.id |
| 公共搜索 | 在 Service/data/SearchServiceImpl 中整理条件组合，保留旧方法兼容 |
| 事实和统计 | 在 Service/data/DataQueryServiceImpl 及对应 Mapper 增加字段投影、去重聚合 |
| 规范化 | 协会内增加 CleaningService、词表配置及必要 Mapper；采集器保留原归属位置 |
| 推荐与反馈 | 协会内增加 RecommendationService、题单/事件持久层与 Controller |
| 前端 | 新增 newfe/src/features/recommendations 的页面、API 与组件；建议 /main/problems 路由，复用通用鉴权与 HTTP |
| 二期 | 协会内增加 AssistantService、ModelClient、受限工具注册和版本化任务说明 |
| 可选语义增强 | 同一 Java 应用内的 EmbeddingService/相似题候选适配；CPU ONNX 或供应商 Embedding API 二选一评测，默认关闭 |

上述名称表示职责，不要求拆 Maven 模块。源目录大小写沿用仓库现状。后端先保留现有包与方法兼容；只有发现具体耦合时才提取共享只读层，不为新页面重构整个协会。

可选语义链为“题目文本批量编码 → 向量/相似题缓存 → 候选 ID → RecommendationService 硬过滤与编排”。自然语言语义查询额外生成查询向量；已有题作为种子时复用向量或相似题 ID。模型只提供候选，不能跳过当前 AC、反馈、归属、平台和难度校验。

Agent 开关与语义增强开关独立。两者关闭时无需 Key、模型文件或 ONNX 原生库即可启动并使用一期功能。即使未来启用 Embedding，Agent 的 LLM 仍走供应商 API；不把 CPU 小模型评测解释为本机部署对话大模型。

## 2. 已有表与关联

| 表 | 源码可确认的关键字段 | 推搜用途与限制 |
| --- | --- | --- |
| user | id、user_code、role_id、deleted、cf_account、luogu_uid | 登录后的业务主体、现役范围及账号绑定 |
| user_auth | id、user_code 等 | 认证身份；其 id 不能替代 user.id |
| oj_account | user_id、cf_account、luogu_uid 等 | 已有账号冗余；须与 user 及历史绑定核对 |
| cf_all_submissions | sub_id、account、problem_id、verdict、rating、creation_time | CF 来源事实；实体对主键的描述不完全一致，实际键以 DDL 为准 |
| luogu_all_submissions | sub_id、uid、problem_id、is_pass、difficulty、creation_time | 洛谷观察记录；实体未表达完整采集覆盖 |
| cf_official_problems | problem_id、title、rating、points、tags | CF 题目目录；当前实体无题面字段 |
| luogu_all_problems | pid、title、difficulty、tags、created_at | 洛谷题目目录；当前实体无题面字段 |
| cf_team_problems | team_contest_id、official_contest_id、problem_name 等 | 集训题映射线索，必须核验实际字段值和证据 |
| task / task_item | 任务定义；内容项含 platform、external_id、url、title、rating、sort_order | 正式任务与题目引用；TaskItemDO.problemId 是非持久字段 |
| task_assign | task_id、user_id、pass_count、status、due_at 等 | 正式派发及可变进度，不能用来重建全历史逐题 AC |
| team / team_member | 团队、成员、团队角色与成员状态 | 正式团队权限；团队队长与全局 user.role_id 不同 |
| task_user_contribution | user_id、date、count、level 等 | 贡献展示；无题目标识，不能用作去重过题集合 |

逻辑关联为 user.id → 经核验的 OJ 账号 → 来源记录 → 题目引用 → 已核验别名 / 规范标签。user_code 只参与已有认证到业务用户的解析，不替换推搜的业务主键。

个人题单独立于正式任务存储。本期暂缓任务表单预填和发布衔接；现有任务进度同步可能发放奖励，不能由推荐触发。共享只读服务的修改仍需维持原任务调用方兼容。

## 3. 统一对象和口径

### 3.1 ProblemRef

统一 DTO 至少包含 platform、source_problem_id、source_problem_key、canonical_problem_key、title、source_url、平台难度、规范标签与元数据版本。

常规 CF 示例：源题号 1850A，源 key 为 cf:1850:A；洛谷示例：P1000、luogu:P1000。CF 的 B1/B2 等索引整体保留。特殊题库与训练赛按来源格式单独解析，不能把任意字符串强行套成常规比赛题号。

没有已核验别名时 canonical_problem_key 等于 source_problem_key。同题映射不会丢弃展示平台和原始题号；原任务表单仍接收自己的 externalId 格式。

可推荐性由题目目录的可核验状态决定，不要求已有队员 AC。所有召回方式返回可解析到目录的题目引用；目录外题先异步补齐元数据，再进入候选。Embedding 索引也可能只覆盖目录子集，其缺失不使结构化搜索中的题目消失。

### 3.2 时间与覆盖

- creation_time：核验后的平台提交时间，不等于上传时间或最终判题时间。
- received_at / source_checked_at：建议在兼容扩展字段或关联采集元数据中保留接收/核对时间。
- first_observed_ac_at：全部已知有效历史中，当前仍被判定为 AC 的提交创建时间最小值。
- last_observed_ac_at：相同范围内的最大提交创建时间。
- observed_ac_count：有效来源提交去重后的 AC 次数；不同于去重题数。
- coverage：按账号和来源记录 complete_in_scope / partial / unknown，并附可访问范围、历史补齐进度与最后成功同步时间。
- data_as_of：本次查询采用的截止时间；来源同步状态另列，不能把当前服务器时间充当来源已同步时间。

complete_in_scope 只表示所声明的可访问范围已经补齐，不代表全部私人、历史或未授权数据完整。洛谷仅监听且未补历史时采用 partial；最早观察日期不证明该日期之后连续无缺口。

库内时间存储方式先审计再定，所有范围转换使用明确时区，界面与“本周”等用户语义使用 Asia/Shanghai。

### 3.3 Profile DTO

每次请求按当前有效绑定查询已入库事实，生成去重已做题集合及各标签的题数、近期新增、最近练习、平台难度分布、标签覆盖比例与来源覆盖。

标签覆盖比例以本次有效已做题中能映射到规范标签的题数为分子，全部有效已做题数为分母；分母为零时返回无样本。它不是历史采集完整率。

画像不持久化是第一版默认。去重、标签统计尽量在 SQL 中完成或用必要字段分批聚合；不从大表加载所有列，不按每道题发起额外查询。

## 4. 新增或复用的持久数据

以下是逻辑契约，字段类型、迁移编号与具体索引在 M0 后确定。表名为建议；复用现有结构必须满足同样语义及权限。

| 对象 | 必须表达的内容 | 阶段 |
| --- | --- | --- |
| oj_sync_state 或等价采集元数据 | 用户、平台、账号/绑定版本、补齐状态、检查点、最后成功时间、覆盖、失败状态 | M1A；优先复用已有 |
| problem_tag | 来源题号、规范 tag_code、来源、状态、词表版本；同来源题与标签唯一 | M1B |
| problem_alias | 来源题 key、规范 key、证据、状态、版本；一个来源仅一个有效映射，禁止循环 | M1B；只写已核验映射 |
| clean_run 或等价运行元数据 | run_id、输入边界、规则版本、检查点、计数、隔离引用、运行状态 | M1B；优先复用兼容记录 |
| recommendation_slate | id、user_id、parent_id、request_id、body_hash、约束、有序题目 JSON、依据、数据时间、版本 | M2A |
| recommendation_event | event_id、user_id、slate_id、题目 key、类型、值、时间、被撤销事件引用 | M2A |
| assistant_session | id、user_id、state_json、current_slate_id、version、更新时间 | A1 |
| assistant_message | 会话、顺序、角色、文本、request_id、结果引用、时间 | A1 |
| assistant_preference | user_id、namespace、key、类型化值、来源、expires_at、status、version | A1 |
| 请求/动作执行记录 | user_id、request_id、body_hash、动作 ID/参数摘要/结果引用、状态、恢复信息 | A1；与消息持久层复用或设 assistant_request |
| user_problem_progress | 用户—规范题聚合、来源覆盖与规则版本 | 可选优化；不是一期前置表 |
| problem_text 或等价文本存储 | 来源题 key、语言、文本、来源/许可信息、内容 hash、更新时间及状态 | 仅 E0 语义实验需要，现有题库实体没有完整题面 |
| problem_embedding 或等价可重建存储 | 来源题 key、embedding_profile_id、内容 hash、维度、向量、生成状态及时间 | 仅 E0；模型/分词/前缀/池化/归一化/文本处理版本由 profile 固定 |
| 相似题预计算结果 | 种子来源题 key、同一向量空间版本、候选 key/分数、数据版本 | E0 可选缓存；读取后仍检查当前元数据与已做题 |

请求/动作记录必须有一个可唯一定位的请求控制实体，不能仅靠扫描聊天正文推断执行状态。如果消息结构无法可靠承载，使用小型 assistant_request 表保存动作状态 JSON；不因此建设工作流平台。

题单同时保存 source_problem_key 与 canonical_problem_key。一期题单无需 session_id 或任何 assistant_* 外键；二期由会话引用既有题单。

持久写入的幂等键绑定当前用户。同键同内容返回原结果，同键不同内容报冲突；最终由数据库唯一约束和短事务保证，先查再插本身不能保证并发幂等。

E0 的存储仅在实验决定实施时确定，不混入一期必需迁移。不同模型或编码约定的向量不直接混算；模型升级建立新版本并在完成校验后切换，不能把新旧版本混在同一索引。向量是可重建检索数据，不是用户偏好或 AC 的权威存储。

## 5. 主要接口契约

新增接口沿用 Result / PageResult 和当前鉴权。平台枚举与协会现有定义适配；用户身份从登录态注入，不接受模型或客户端替换主体。

以下路径是待实现契约。保留 /api/adventurers 前缀表示后端复用边界，不要求前端使用协会页面；独立前端通过通用 HTTP 封装访问，普通队员入口不得改走 /api/admin/**。

| 阶段 | 拟定接口 | 主要内容 |
| --- | --- | --- |
| 一期 | GET /api/adventurers/problems/search | query、platform、tags、各平台难度、excludeSolved、分页 |
| 一期 | GET /api/adventurers/problems/hot | window=7d/30d、platform、分页 |
| 一期 | POST /api/adventurers/recommendations | request_id、平台、主题、难度、数量、巩固/探索配额 |
| 一期 | GET /api/adventurers/recommendations | 本人历史快照分页 |
| 一期 | GET /api/adventurers/recommendations/{slateId} | 本人快照与当前有效性 |
| 一期 | POST /api/adventurers/recommendations/{slateId}/replace | 请求标识、明确题目 key/位置、允许修改的约束 |
| 一期 | POST /api/adventurers/recommendations/{slateId}/events | 事件标识、题目、类型及撤销引用 |
| 二期 | POST /api/adventurers/assistant/sessions | 创建本人会话 |
| 二期 | POST /api/adventurers/assistant/sessions/{id}/messages | request_id、version、消息 |
| 二期 | GET /api/adventurers/assistant/sessions/{id} | 恢复本人会话 |
| 二期 | GET /api/adventurers/assistant/preferences | 查看有效偏好 |
| 二期 | PUT /api/adventurers/assistant/preferences/{key} | 校验后的偏好及有效期 |
| 二期 | DELETE /api/adventurers/assistant/preferences/{key} | 删除并更新遗忘边界 |

响应包含实际采用条件、题目及顺序、依据、来源同步范围、不足或降级原因。内部规则版本进入诊断字段，普通页面以可理解说明呈现。

原 /api/admin/tasks 搜索契约保留，通过共享 Service 复用能力；不直接把管理端路径开放给普通队员。个人题单、事件、会话与偏好均在 Service 校验归属，以覆盖 Agent 的内部调用。

## 6. 一致性、调度与故障行为

采集成功确认只在对应记录持久化后返回；分页进度不能越过未提交数据。判题变化按可信来源和新鲜度更新，不能采用“AC 永远优先”或用更晚的客户端上传时间证明判题更新。

历史清洗保留来源、检查点和规则版本。初次运行采用仅启用清洗的配置，避免启动正式任务派发、奖励和贡献调度。多实例使用数据库协调运行归属，限制批量和连接占用。

查询采用确定的版本与截止时间；返回前复查候选最新已知 AC 和映射。无法保证还未采集的外站 AC 已被排除，页面据实说明。

| 故障或变化 | 处理 |
| --- | --- |
| 采集暂时失败 | 保留已入库事实，更新失败和覆盖状态，不把失败标为完成 |
| 绑定、判题、标签或别名修正 | 下次查询生效；有缓存/聚合时按版本失效或重算 |
| 归属冲突、过题事实不可读 | 停止相关个性化保证；仅在题库可读时提供标明范围的普通搜索 |
| Redis 不可用 | 对可重建缓存限流回源；不得因此漏掉个人反馈与已做题校验 |
| MySQL 不可用 | 返回可重试错误，不能用旧快照声称完成新的推荐 |
| Agent 关闭或供应商失败 | 保留全部一期功能；已有有效工具结果可按模板返回 |
| 可选 Embedding 超时、缺向量或版本不匹配 | 跳过该候选来源、保留规则推荐；语义查询明确提示能力暂不可用，不把关键词结果冒充语义命中 |

CPU 试验先在独立进程验证原生库和模型兼容，再决定是否嵌入业务 JVM；原生库崩溃不能靠普通 Java 异常降级保证隔离。清洗、离线编码与在线业务分别限制并发，验收包含已有排名、采集和数据库查询的资源影响。硬件快照与验证流程见 [03](03-technology-options.md)、[14](14-e0-cpu-model-evaluation.md)。

## 7. 源码证据入口

- [后端依赖](../be/pom.xml)、[前端依赖](../newfe/package.json)、[前端 HTTP](../newfe/src/core/api/http.js)。
- [用户实体](../be/src/main/java/gxuicpc/qingluan/modules/user/DO/UserDO.java)、[账号实体](../be/src/main/java/gxuicpc/qingluan/entity/OJAccount.java)、[登录解析](../be/src/main/java/gxuicpc/qingluan/modules/auth/AuthInterceptor.java)、[角色拦截](../be/src/main/java/gxuicpc/qingluan/modules/auth/RoleAuthorizationInterceptor.java)。
- [CF 来源实体](../be/src/main/java/gxuicpc/qingluan/entity/CFAllSubmission.java)、[洛谷来源实体](../be/src/main/java/gxuicpc/qingluan/entity/LuoguAllSub.java)、[CF 协会实体](../be/src/main/java/gxuicpc/qingluan/modules/Adventurers/DO/oj/CfSubmissionDO.java)。
- [CF 题库](../be/src/main/java/gxuicpc/qingluan/modules/Adventurers/DO/oj/CfOfficalProblemsDO.java)、[洛谷题库](../be/src/main/java/gxuicpc/qingluan/modules/Adventurers/DO/oj/LuoguProblemDO.java)、[共享数据读取](../be/src/main/java/gxuicpc/qingluan/modules/Adventurers/Service/data/DataQueryServiceImpl.java)。
- [任务内容项](../be/src/main/java/gxuicpc/qingluan/modules/Adventurers/DO/task/TaskItemDO.java)、[派发进度](../be/src/main/java/gxuicpc/qingluan/modules/Adventurers/DO/task/TaskAssignDO.java)、[任务创建](../be/src/main/java/gxuicpc/qingluan/modules/Adventurers/Service/task/TaskServiceImpl.java)、[进度结算](../be/src/main/java/gxuicpc/qingluan/modules/Adventurers/Service/task/TaskProgressServiceImpl.java)。

实体映射能证明代码采用了这些字段，不能证明生产表已具备唯一键、索引或数据完整性。
