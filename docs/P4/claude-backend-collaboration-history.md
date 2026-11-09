# 后端与 Claude 协作记录（Mac 备份）

本文件归档 Mac 备份中 backend/CLAUDE.md 的原文。该文件最后修改于 2026-06-10，记录了当时的后端开发进度、设计取舍和协作经验。下文的“当前状态”、待办和验证结果均属于历史记录；现行状态以仓库代码、正式文档和 AGENTS.md 为准。

同一备份中的未提交标志设计稿保存在 [CampusHub 标志概念图](asset/campushub-neon-logo-concept.png)。

## 备份原文

@AGENTS.md

# 后端工程补充

## 当前状态

- 后端正在从零重建，严格按 P3 设计文档（`docs/P3/class_design.md`、`database_design.md`、`api_design.yaml`）实现
- 放弃 `origin/fix/multiple-bugfixes` 上的不规范代码，仅保留 `pom.xml` 和 `application.yaml` 骨架
- 后端负责人：谢易轩

## 工作方式

每一步动手前，先简短说明要做什么，等用户批准再执行。不用写详细 plan，口头说清楚即可。

### 提交后更新进度快照（开发流程第 5 步的执行细则，强制）

**这是「开发流程 → B. 单模块内部节奏」第 5 步的具体做法**——每次完成一批代码提交后，立即把这一批追加进本文件「进度快照」章节，作为提交的法定收尾动作，不用等用户单独要求，也不单独问"要不要更新"。`git commit` 拿到 hash → 紧接着写快照，一气呵成。具体：

1. 在「进度快照」末尾按批次新增一个 `### 第 N 批：<模块名>（commit `<hash>`，分支 `<branch>`，从 dev 切出）` 块，写清：本批做了什么接口/DTO/service、关键业务决策、共用增强、`./mvnw test` 通过数，以及**本批沉淀的设计准则**（踩过的坑、边界判断）。
2. 同步更新「下一步逐模块打通」清单：把已完成项划掉（`~~...~~ ✅`）。
3. 同步更新「待办」：补未 push/未合 dev 的新分支、新引入的待处理项。
4. **本文件 `backend/CLAUDE.md` 被 `.gitignore` 忽略（`.gitignore:14-15` `CLAUDE.md` / `**/CLAUDE.md`），是本地专属文件、不进版本库**——所以更新它**不产生提交**，只在本地工作树累积（历次「进度快照」都是这样攒下来的）。无需也无法为它单独 commit；它不会污染任何 feature 的 diff。团队共享的指令源是已跟踪的 `AGENTS.md`（根 + backend），`CLAUDE.md` 通过 `@AGENTS.md` 引用它。

> 为什么是「约定」而非真正的 git hook：进度快照要**读懂这次提交干了什么、再写成有意义的中文总结 + 提炼设计准则**，这是判断性工作，机械 hook 脚本写不出有价值的内容。故由 AI 在提交收尾时主动执行。又因本文件 gitignore、更新不入库，hook 也无从「在提交里带上它」。若将来想要"提交时强制提醒补快照"，可在 `settings.json` 配 `PostToolUse`（匹配 git commit）/`Stop` hook 做轻量文字提示（不自动生成内容），需用户另行批准（走 update-config skill）。

## 技术学习

用户刚学完 Spring Boot，借这个项目熟悉后端开发流程。用到便利技术时（Lombok `@Data`、MyBatis Plus `BaseMapper`、Spring 依赖注入、`@Transactional` 等），主动解释它省了什么模板代码、背后实际发生了什么。不要默认用户熟悉这些。

## 开发流程

### A. 宏观顺序（地基先行，再纵向打通业务模块）

整体分两个阶段，**先把所有业务模块都要依赖的地基铺好，再逐个业务模块纵向打通**：

1. **脚手架**（`100f82b`）—— entity/enums/mapper/MyBatis 配置/建表脚本，先立数据与持久层骨架。
2. **Common 公共层 + 鉴权基础设施**（`09c648d`/`f01bd48`）—— 统一响应、错误码、异常、分页、JWT 拦截、`@CurrentUser`。**这层被后面每个 Controller/Service 复用，所以必须先于任何业务模块**。
3. **业务模块逐个纵向打通**（按 Sprint P0 顺序）：用户 → 文件 → 代取流转 → 支付 → 评价 → 通知。
   - 用户模块排第一，因为它是所有受保护接口的前置，也是地基的首次真实集成验证。
   - 有依赖关系的（如头像/实名材料上传依赖文件模块）按依赖顺序排，遇到被依赖项未就绪就拆出、留到对应模块。

> 一句话：**先地基（脚手架→公共层→鉴权），后业务（一次一个模块，纵向到底）**。不横向铺半成品。

### B. 单模块内部节奏（纵向打通）

每个业务模块按固定节奏推进（**五步缺一不可，第 5 步是提交的法定收尾，不是可选项**）：

1. **列设计决策** —— 对着 `api_design.yaml` 把该模块的契约（路径/字段/枚举/错误码）和待定决策（DTO 划分、校验落点、字段归属、鉴权边界等）列清，等用户批准再写。
2. **分小步实现** —— 顺序：dto → service 接口 → service 实现 → controller → test。每小步先口头说要做什么，写完即 `./mvnw compile` 验证再进下一步。
3. **测试收尾** —— Service 单元测试（mock mapper）+ Web 切片测试（`@WebMvcTest` 不连库）；`./mvnw test` 全绿才算完成。
4. **提交** —— 走 `feature/*` 分支，英文 Conventional Commit，显式 staged，不含 gitignored/敏感文件（见 p4-git-workflow skill）。
5. **更新进度快照（提交后立即做，不等用户要求）** —— commit 一拿到 hash，立刻把这一批追加进本文件「进度快照」章节。**这一步未做 = 这一批没收尾**，下一批不开工。具体动作见下方「提交后更新进度快照」小节。把它当成提交流程的一部分：`git commit` 成功 → 写快照，连续完成，中间不插别的事。

带文件上传的接口依赖文件模块，遇到时拆出、留到文件模块再做。

## 进度快照（截至上次会话）

当前分支：`feature/notification-module`（从 `dev` 切出，本批已提交 `d51a9a1`，**尚未 push、未合 dev**）。
分支链：第五批 `feature/pickup-core`、配置对齐批次 `fix/api-prefix-and-time-fill`(`33f36d4`)、毛嘉和测试交付 `feature/p4-testing-cicd-delivery` 均已合入 `dev`。另有开发种子脚本 `feature/dev-seed-data`(`471bdef`，`backend/src/main/resources/seed-dev.sql`：清空重建 + admin/alice/bob/carol 账户 + 示例代取/待审核认证，密码用项目 BCryptPasswordEncoder 真实生成的 hash)，未合 dev。
代码提交走 `feature/*`/`fix/*` 分支，不直接提交在 `dev`/`P4`。

### 第一批：脚手架（commit `100f82b`）
- `entity/enums/` — 12 个枚举（对照数据库设计 §7）
- `entity/` — 7 个实体，**富领域模型**（带 markXxx/canXxx 等领域方法，非贫血）
- `mapper/` — 7 个 Mapper（`extends BaseMapper`，空接口）
- `config/MybatisPlusConfig` — 分页插件（已加 `mybatis-plus-jsqlparser` 依赖）
- `config/TimeFieldFillHandler` — 时间字段自动填充
- `resources/schema.sql` — 7 张表建表脚本（对照数据库设计 §6，无 version 列）
- `pom.xml` — 依赖整理（删 actuator，加 jjwt + security-crypto + jsqlparser），版本集中到 `<properties>`
- `application.yaml` — 配置外部化（`${DB_PASSWORD:}` 占位符），删了不符设计的逻辑删除配置
- `application-example.yaml`（模板，提交）/ `application-local.yaml`（真实密码，gitignore）

### 第二批：Common 公共层 + 鉴权基础设施（commit `09c648d` + 修正 `f01bd48`）
- `common/` — 统一响应与异常地基（严格对齐 `api_design.yaml` 契约）：
  - `ApiResponse<T>` — 成功响应（`code/message/data`，data 为 null 也序列化）；`ErrorResponse` — 错误响应（四字段全输出，`errors` 无详情时输出 null，契约 required）
  - `ErrorCode`（枚举，`@Getter`）— 7 个错误码集中管理，前 3 位 = HTTP 状态
  - `BusinessException` — 持 `ErrorCode` + 可选 `errors`，Service 抛它表达业务冲突
  - `PageQuery`（`@Min/@Max` 校验）/ `PageResult<T>`（契约字段是 `list`，**不是** MP 的 `records`，`of(IPage)` 隔离框架对象）
  - `GlobalExceptionHandler`（`@RestControllerAdvice`）— 翻译业务/参数校验/兜底异常为统一响应 + 正确 HTTP 状态
  - `CurrentUserContext` — 不可变身份上下文（`currentUserId/role/authStatus`）
- `security/` — JWT 鉴权（拦截器方案，**不引 Spring Security 过滤器链**）：
  - `JwtProperties`（`@ConfigurationProperties(prefix="jwt")`）/ `JwtUtil`（jjwt 0.12.x，secret 外部化、空 secret fail-fast）
  - `CurrentUser` 注解 + `CurrentUserArgumentResolver` — Controller 参数直接拿 `CurrentUserContext`
  - `AuthInterceptor` — 解析 `Bearer` Token → 放入 request attribute；只验签不查库
- `config/` — `SecurityConfig`（`BCryptPasswordEncoder` Bean）/ `WebMvcConfig`（注册拦截器+解析器+CORS，白名单放行 `/auth/**`）
- `application.yaml` 加 `jwt.secret/expiration` 外部化；example/local 同步占位符与真实值
- 测试：`ApiResponseTest`（4，纯 Jackson 验序列化契约）/ `JwtUtilTest`（4，round-trip + 篡改/过期/空 secret）；`./mvnw test` 全绿（含 `contextLoads`）

### 第三批：用户模块（commit `663b9da`，分支 `feature/user-auth`）
本批只做 3 个**纯 JSON** 接口，打通注册→登录→鉴权闭环（带文件上传的 `PUT /users/me/profile`、`POST /users/me/verification` 依赖文件模块，**留到文件模块**）：
- `dto/user/` — `RegisterRequest`/`LoginRequest`（`@Data` + Bean Validation）、`LoginSession`/`UserMeResponse`（不可变，`@Getter`+`@RequiredArgsConstructor`/`@Builder`）
- `service/` `AuthService`/`UserService` 接口（签名收原始字段/`userId`，与 Web DTO 解耦）；`service/impl/` 两实现
- `controller/` `AuthController`（`/auth/register`、`/auth/login`）/ `UserController`（`/users/me`，`@CurrentUser` 注入身份）
- 已落地业务决策：注册重复 → 40901 `USERNAME_ALREADY_EXISTS`（先 `selectCount` 预检 + DB 唯一约束兜底 `DuplicateKeyException`）；登录失败 → 40101 `INVALID_CREDENTIALS`（**不区分**用户不存在/密码错，防枚举）；注册 `nickname` 默认取 `username`；学号脱敏前 2+后 2；`/users/me` **读库**返回最新（不用 JWT 快照）
- 测试：`AuthServiceImplTest`（单元 5，Mockito mock mapper/encoder/jwt）+ `UserModuleWebTest`（**`@WebMvcTest` 切片 5，不连库**）；`./mvnw test` → 19 passed

### 第四批：文件模块 + 实名认证/审核 + 资料编辑（commit `1bd29cb`，分支 `feature/file-and-verification`，从 dev 切出）

> 起因：上一会话 codex 一次性把这三块都写了（绕过了"逐模块、先列决策"的节奏）。本会话对它逐轮审查 + 返工，沉淀出上面「第四批设计准则」。最终 `./mvnw test` → **39 passed**。

- **文件模块**：`FileStorageService`（`uploadImage` 返回 `Long fileId`、`loadFile` 受信读取、`updateBusinessTrace` 回填溯源）+ 本地存储实现（路径分层、SHA256、元数据落库失败回滚磁盘文件、路径遍历防护）；`util/ImageValidator`（JPG/PNG + 大小由框架 multipart 配置兜底）；`service/dto/StoredFileContent`；`entity/enums/FileBusinessType`。
- **实名认证 + 审核**：`VerificationReviewService`（`submitVerification` 提交、`queryReviews`/`handleReview`/`loadReviewImage` 管理员侧）；`AdminVerificationReviewController`（`/admin/verification-reviews`，契约已定义，非越界）；状态变更经 `UserService.markVerificationSubmitted/Approved/Rejected`。
- **资料编辑**：`PUT /users/me/profile`（昵称/头像/学院/联系方式部分更新）；`POST /users/me/verification` 委托审核服务。
- **共用地基增强**：`common/ErrorReason`（reason 枚举）+ `BusinessException(ErrorCode, ErrorReason)` 便利构造器；`JwtProperties` 迁到 `config` + `PropertiesConfig` 集中注册；`util/StudentIdMasker`。
- **遗留 doc-drift（未改文档，待批准）**：`class_design.md` 把 `submitVerification` 放 `UserService` 且签名收 `materialFileId`，实现归到了 `VerificationReviewService` 且收 `MultipartFile`；`UserSummary`（含 ratingSummary）与本批新建的 `UserBrief`（userId/username/nickname，供审核列表）并存；`markAuthSubmitted` 旧三参描述。

关键设计决策（已确认）：
- 富领域模型，状态转换调实体方法而非裸 setter
- Service 依赖注入用**接口**不用实现类（纠正队友错误）
- Service 返回 **DTO** 不用 `Map<String,Object>`（纠正队友错误）
- 错误码用**枚举集中管理**（`common/ErrorCode`），不用魔数（已落地）
- 并发控制用**状态条件更新**（`WHERE status=?`），不加 version 列、不引乐观锁插件
- 无逻辑删除（文档确认无此需求）
- **统一响应拆双类 `ApiResponse`/`ErrorResponse`**，与 OpenAPI 两个 schema 一一对应（曾合并为单类，但 `ErrorResponse.required` 含 `errors`，单类 `@JsonInclude(NON_NULL)` 无法同时满足"成功不带 errors / 错误必带 errors 键且可为 null"，故拆回）
- **身份传递用 `@CurrentUser` + `HandlerMethodArgumentResolver`**，不用 ThreadLocal（无并发/清理问题）；Service 签名显式收 `userId`，不信任前端传入
- **JWT 载荷含 `userId+role+authStatus` 快照**，拦截器只验签不查库；需要权威认证状态的门槛走 Service 读库（如发布代取前的 `ensureCertified`），快照略滞后不影响正确性
- **已引入 Lombok，能用就用**（用户明确约束）：减少手写样板——可变请求 DTO 用 `@Data`，不可变对象用 `@Getter`+`@RequiredArgsConstructor`（多字段/含可空用 `@Builder`），构造注入用 `@RequiredArgsConstructor`。不手写 getter/setter/构造器
- **测试策略**：Web 层验证用 `@WebMvcTest` 切片（`@MockitoBean` 替换 Service/JwtUtil，**不连库**、快、CI 友好）；Service 落库逻辑用 Mockito mock mapper 的单元测试覆盖。真实 SQL 集成测试待引入 H2/Testcontainers 再补（当前 `@SpringBootTest` 会连本地真实 MySQL）。**新代码用 `@MockitoBean`**，不用已废弃的 `@MockBean`
- **评价详情接口（`GET /evaluations/{evaluationId}`）有意不做**：`class_design.md` 定义了 `EvaluationDetail`/`queryEvaluationDetail`，但 `api_design.yaml` 未暴露对应接口——这不是文档漏写，是 MVP 取舍。详情比公开列表（`/users/{userId}/evaluations`）多出的只有 `content` 全文和内嵌 `pickupSummary`（代取服务背景）；做成公开会泄露列表刻意隐藏的业务背景，做成非公开则路人（信誉展示的主要受众）看不到、参与者又无信息增量，故无真实使用场景。公开列表已满足信誉展示需求，评价又是 Should 模块，不补此接口

### 第四批沉淀的设计准则（文件 + 实名认证模块返工得出，已落地）

这批是审查/返工 codex 代码时反复确认的边界准则，**后续每个业务模块都按这些来，不要重蹈覆辙**：

- **校验分层：格式校验归 DTO 声明式，业务校验留 Service**。长度/正则/必填/枚举这类「请求是否合规」用 Bean Validation 注解 + Controller `@Valid` 入口拦截；Service 只做「业务是否允许」（已占用/状态不符/至少改一项）。**不要把格式校验下沉到 Service 手写**（codex 的典型坏味道）。
  - multipart 表单：文本字段封 DTO 用 `@Valid @ModelAttribute` 接收，文件单独 `@RequestPart`；跨字段约束（如 REJECT 时 reason 必填）用 `@AssertTrue`。
  - 跨 DTO+文件的判断（如「至少改一项」要同时看 avatar 文件）DTO 表达不了，才留 Service。
- **错误 reason 用枚举集中（`common/ErrorReason`），不写裸字符串**；取值**按 `api_design.yaml` 契约收敛**，不要自造契约外的细码（codex 造了 `INVALID_CREDENTIALS`/`USERNAME_ALREADY_EXISTS` 等，契约其实统一用 `DUPLICATE_OR_CONFLICTED_OPERATION`）。`BusinessException(ErrorCode, ErrorReason)` 便利构造器自动组装 `{"reason": name()}`。
  - 参数校验的 errors 用**字段名**作 key（`{"file": "..."}`）；业务状态错误才用 `reason` 作 key。两者别混。
  - 契约缺口（如登录失败 reason 文档没定义）：保留现用值 + 在枚举 javadoc 标注「契约缺口」，改契约需用户批准。
- **跨模块不传 entity，不暴露敏感字段**（`class_design.md` §315 / FR-UM-03）：模块间互调返回业务层 DTO（如 `service/dto/UserBrief`），不返回 `User` 实体。返回 entity 会泄露 passwordHash、明文 studentId 等。
  - 用户实体的读取/状态变更**收口到 owner service**：其他模块不注入 `UserMapper`，改调 `UserService` 的领域方法（`getUserBriefs`/`markVerificationApproved` 等）。状态变更经 owner service。
  - 批量展示场景用**批量查询 + 只查需要的列**（`selectList + .select(...)`），避 N+1，也别 `selectByIds` 拉全表再丢弃。`selectBatchIds` 已废弃（MP 3.5.4+），用 `selectByIds` 或带 `.select()` 的 `selectList`。
- **模块归属看业务本质，不看 URL 前缀**：`POST /users/me/verification` 路径在 users 下，但「提交认证=创建审核记录」本质归 `VerificationReviewService`，Controller 委托过去。依赖方向保持单向（VerificationReview→User），避免循环依赖。
- **文件模块只返回 fileId，不返回 `StoredFile`**：四个上传场景（头像/认证材料/取件凭证/完成凭证）都只消费 id。`loadFile(fileId)` 是**受信方法**——不做权限判断，调用方必须先鉴权，javadoc 已强调，禁止把前端 fileId 直传。
- **DTO/工具的去重与归置**：重复逻辑抽公共（脱敏 `util/StudentIdMasker`、图片校验 `util/ImageValidator`，跨模块共享才抽）；无状态 helper 统一放 `util`；service 层数据载体放 `service/dto`。删没人读的字段（YAGNI）。
- **`@ConfigurationProperties` 用 `@Getter @Setter` 不用 `@Data`**：`@Data` 的 `@ToString` 会把 secret 打进日志（JwtProperties 尤危）。配置类集中放 `config` 包，统一用 `@EnableConfigurationProperties`（`config/PropertiesConfig`）注册，不用 `@Component` 混搭。有安全默认的字段给缺省值，敏感字段（secret）不给。
- **admin 身份用 JWT 快照判定即可，不读库**：token 有 HMAC 签名，篡改 role 会验签失败。`requireAdmin` 读 `CurrentUserContext.role` 是安全的——只有「动态降级权限」场景才需读库，本项目 admin 固定，无此需求。

> 一句话提炼：**校验分层、reason 枚举对齐契约、跨模块传 DTO 不传 entity、文件只回 id、状态变更经 owner service**。

### 第五批：代取请求核心流转（commit `4178972`，分支 `feature/pickup-core`，从 dev `1bd29cb` 切出）

本批打通代取闭环：**无报酬全闭环 + 有报酬到 `WAITING_PAYMENT` 占位**（真实支付宝支付/回调/退款/结算留第六批）。最终 `./mvnw test` → **54 passed**（新增 14：`PickupServiceImplTest` 10 + `PickupModuleWebTest` 5 - 移走的 1 + `UserModuleWebTest` +1）。

- **10 个接口**（全部对齐 `api_design.yaml`）：`POST/GET /pickup-requests`（发布/大厅）、`GET /pickup-requests/{id}`（详情）、`/credential`（取件凭证）、`/accept`、`/completion-proof`(POST上传/GET读)、`/completion-confirmation`、`/cancel`，以及 `GET /users/me/pickup-requests`（我的发布/接单）。
- `dto/pickup/` — 请求 DTO（`PickupCreateRequest` multipart，`@AssertTrue` 管 PAID↔金额跨字段约束；`PickupCancelRequest`）+ 不可变响应 DTO（`PickupCreateResult`/`PickupAcceptResult`/`PickupRequestSummary`/`PickupRequestDetail`/`PickupSummary`/`CompletionConfirmResult`/`PickupCancelResult`/`UserSummary`）。
- `service/PickupService` + impl — 状态流转用**条件更新**（`WHERE id=? AND status=?`，影响行数 0 视为并发冲突），不加 version 列。`@Transactional` 加在发布/确认/取消。
- `service/PaymentService` + **占位实现** `PaymentServiceImpl` — 接口签名按 class_design 定稿；本批 `createPrepay` 落 WAITING_PAY 记录 + 占位 `payEntry`，`cancelWaitingPayment`/`refundPayment`/`settlePayment` 仅状态标记（无真实资金）。`PickupService` 依赖此接口，第六批填支付宝零返工。
- 共用增强：`common/ErrorReason` +4（`NOT_PICKUP_PARTICIPANT`/`PICKUP_STATUS_NOT_ALLOWED`/`ACCEPT_DEADLINE_EXPIRED`/`COMPLETION_PROOF_NOT_AVAILABLE`，均契约内）；`UserService.ensureCertified`（读库认证门槛）；`enums/FileBusinessType` +`PICKUP_REQUEST`（凭证溯源）；`security/AuthInterceptor` 加**可选鉴权路径**（公开 GET 无/无效 token 放行、写接口仍 401）。
- 已落地业务决策：有报酬 `expireAt`=now+3min；接单超 `acceptDeadline` → 先流转 CANCELLED(`ACCEPT_DEADLINE_EXPIRED`) 再 409；发布方不能接自己单；确认完成要求 IN_PROGRESS+已传完成凭证；取消仅 WAITING_PAYMENT(close)/WAITING_ACCEPT(无报酬直接取消/有报酬 refund)，IN_PROGRESS/COMPLETED/CANCELLED 不可取消。

### 第五批沉淀的设计准则：Controller 按 URL 路径前缀划分，跨模块靠委托

会话中发现并对齐的边界准则（**后续评价/通知模块按此，不要按业务模块拆 controller**）：

- **Controller 按 URL 路径前缀分，不按业务模块分**：一个 controller 一个类级 `@RequestMapping(前缀)`，与既有 `AuthController(/auth)`/`UserController(/users)`/`AdminVerificationReviewController(/admin/verification-reviews)` 一致。
- **跨模块的接口按路径归属，由该 controller 委托到 owner service**：业务逻辑不落 controller。例：`POST /users/me/verification`（路径在 /users）→ `UserController` 委托 `VerificationReviewService`；`GET /users/me/pickup-requests`（路径在 /users）→ `UserController` 委托 `PickupService`。`PickupController` 只管 `/pickup-requests/**`。
- **后续可预见的同类情况**：评价 `/users/{userId}/evaluations`（归 UserController 委托 EvaluationService）+ `/pickup-requests/{id}/evaluations`（归 PickupController 或新 EvaluationController）；通知 `/users/me/notifications`（归 UserController 委托 NotificationService）。按路径前缀就位，别按模块新建 controller 打散路径。
- **同路径公开 GET + 鉴权写接口共存**（如 `GET /pickup-requests` 公开浏览 vs `POST /pickup-requests` 发布）：用 `AuthInterceptor` 的「可选鉴权路径」——仅对该 GET 放宽（无/无效 token 放行、不还原身份），同路径写接口因 `@CurrentUser` 解析不到上下文仍得 401，鉴权语义不丢。新增此类公开 GET 时把路径加进 `OPTIONAL_AUTH_PATHS`，注意通配符 `/x/*` 只匹配一段（受保护子路径 `/x/{id}/credential` 不受影响）。

### 配置对齐与联调修复（commit `33f36d4`，分支 `fix/api-prefix-and-time-fill`，从 dev 切出）

本次为联调前的配置对齐 + 两个阻断性 bug 修复（非新业务模块）：

- **base path 对齐 `/api/v1`**：`application.yaml` 的 `server.servlet.context-path` 原为 `/api`，与 `api_design.yaml` 的 `servers`（`/api/v1`）和前端 axios `baseURL`（`/api/v1`）不符——真机启动前端会全 404（`@WebMvcTest` 用 MockMvc 不走 servlet 容器、不剥 context-path，故 54 测试一直绿没暴露）。改为 `/api/v1`，控制器 `@RequestMapping`（`/auth`/`/users`/`/pickup-requests`，相对 context-path）与拦截器白名单均不动。
- **`VerificationReview.submittedAt` 插入填充缺失**：实体已标 `@TableField(fill=INSERT)`，但 `TimeFieldFillHandler.insertFill` 只 `strictInsertFill` 了 `createdAt`/`updatedAt`——strict 填充须**逐字段名显式列出**，注解只是开关。漏填导致 insert 写 `submitted_at=NULL` 撞 NOT NULL 约束（联调报错）。补一行 `strictInsertFill(..., "submittedAt", ...)`，与 `createdAt` 同一时刻。
- **未实现 API 误报 500 → 改返回统一 40401**：路径未命中原落到 `@ExceptionHandler(Exception.class)` 兜底 = 500，误导联调（像后端崩了而非没实现）。新增 `@ExceptionHandler({NoResourceFoundException, NoHandlerFoundException})`（放在 `Exception` 兜底**之前**）翻成 `ErrorCode.NOT_FOUND`（40401）。**绝不重定向 API**——API 是给程序调的，404 该回 JSON。`NoResourceFoundException` 在 Spring Boot 3.x 默认即抛、开箱生效；`NoHandlerFoundException` 仅作防御性兜底，**未**配 `throw-exception-if-no-handler-found`。

本次确立的部署/联调约定（后续按此，别再走回头路）：

- **前后端分离同源，靠路径前缀隔离两类"未命中"**：`/api/v1/**` → 后端 API，未命中回 **40401 JSON**；其余 GET（前端路由如 `/pickup/5`）→ **SPA fallback 回 `index.html`**（forward 语义、URL 不变，**不是** 302 重定向）。
- **本地演示用 Vite proxy，后端只出 API、不托管前端**：`frontend/vite.config.js` 已配 `proxy '/api' → localhost:8080`；前端 axios `baseURL='/api/v1'` 是**相对路径**，浏览器只与 `localhost:5173` 同源对话、proxy 在服务端转发到 8080——**全程不触发 CORS**。生产用 nginx 同源（架构文档 P2：nginx `:80` 静态资源 + `/api` 反代），原理相同。**SPA fallback 由 Vite/nginx 负责，后端一行都不写**。后端 `WebMvcConfig` 那份 `allowedOriginPatterns("*")` CORS 是冗余安全网（直连后端/Postman 时才用得到），留着无害。
- **启动联调**：两个终端 `cd backend && ./mvnw spring-boot:run` + `cd frontend && npm run dev`，浏览器只开 `http://localhost:5173`。前置：MySQL 在跑 + `campushub` 库 7 表已建 + `application-local.yaml` 填好密码/JWT secret（均已就绪）。
- **codex 临时同源部署的残留已清理**：`backend/src/main/resources/static/` 前端产物已删；`FrontendForwardController`（同源 SPA fallback controller）源文件已删（`target/` 旧 `.class` 待 `mvn clean` 清）；其悬空测试 `FrontendForwardControllerTest` 不纳入版本控制。改用 Vite proxy 后后端不再做 SPA fallback。

### 第六批：通知模块（commit `d51a9a1`，分支 `feature/notification-module`，从 dev 切出）

本批实现站内通知模块本体 + 在已实现业务模块的状态变更点接入发通知调用。最终 `./mvnw test` → **72 passed**（新增 7：`NotificationServiceImplTest`）。

- **3 个接口**（全部对齐 `api_design.yaml`，均在 `/users/me` 下、均需鉴权）：`GET /users/me/notifications`（分页列表）、`GET /users/me/notifications/unread-count`（未读数）、`POST /users/me/notifications/{id}/read`（标记已读）。按"路径前缀归属"约定挂在 `UserController` 委托 `NotificationService`，不新建 NotificationController。
- `service/NotificationService` + impl — `createNotice`（业务模块调用，单 insert 不抛异常，不阻断主流程）/`queryMyNotices`（分页，按 createdAt 倒序）/`countUnread`/`markRead`（404 不存在 / 403 非接收者 / 已读幂等）。
- `dto/notification/` — `NotificationItem`（响应 VO，含 entity→VO 的 `from`）、`UnreadCountResponse`、`ReadStatus`（UNREAD/READ）。
- **业务触发点接入**（改动已完成模块，最小化）：`VerificationReviewServiceImpl.handleReview`（审核通过/驳回→通知申请人，type=VERIFICATION）；`PickupServiceImpl` 的 acceptPickup/uploadCompletionProof/confirmComplete/cancelPickup（→通知对手方，type=PICKUP）。评价触发点（EVALUATION）留到评价模块。
- 共用增强：`common/ErrorReason` +`NOT_NOTIFICATION_RECEIVER`（契约 403）。
- 测试：`NotificationServiceImplTest`（7，mock mapper）；现有 3 个 Web/集成测试（UserModuleWebTest/UserFileAndVerificationWebTest/CoreFlowIntegrationTest）补 `NotificationService` 的 `@MockitoBean`；Pickup/Verification 测试加 verify 触发点交互。

本批沉淀的设计准则（已落地）：
- **VO 专用枚举放 `dto`，不放 `entity.enums`**：`ReadStatus` 无数据库列映射（已读状态库里是 `Boolean is_read`），是纯对外枚举，归 `dto/notification`；`entity.enums` 只放有真实列映射的领域枚举。
- **契约里是枚举的字段，VO 出口必须用枚举不用 String**：`NotificationItem.businessType` 用 `entity.enums.BusinessType`（契约定义为 enum），不用裸 String，避免契约漂移；entity 的 `businessType` 仍是通用 String 溯源字段，转换在 VO 出口做。
- **DTO 数值字段统一用包装类型**：与全项目一致（`Long` 而非 `long`）。
- **MyBatis Plus + SQL 保留字坑（重要）**：实体字段名取了 SQL 保留字（如 `read`）时，MP 在 SELECT 会以 Java 属性名作列别名（`is_read AS read`），裸保留字无反引号触发语法错误——**mock 单测发现不了，真机 SELECT 才暴露**。解法：字段改名避开保留字（`read`→`isRead`），列名仍由 `@TableField("is_read")` 绑定不变。

### 第七批：评价模块（commit `f0b8fda`，分支 `feature/evaluation-module`，从 dev 切出）

本批打通代取完成后的双方互评 + 信誉展示，并补齐用户公开主页（之前因依赖评价摘要而留白）。最终 `./mvnw test` → **92 passed**（新增 20：`EvaluationServiceImplTest` 11 + Pickup/User Web 切片 5 + 各 Web/集成测试补 `EvaluationService` @MockitoBean 后照常绿）。

- **5 个接口**（全部对齐 `api_design.yaml`）：`POST /pickup-requests/{id}/evaluations`（提交评价）、`GET /pickup-requests/{id}/evaluation-eligibility`（评价资格）归 `PickupController` 委托 `EvaluationService`；`GET /users/{userId}/profile`（公开主页，`includeRating` 控评价摘要）、`/rating-summary`（好评率）、`/users/{userId}/evaluations`（收到的评价）归 `UserController` 委托。
- `dto/evaluation/` — 请求 DTO（`EvaluationCreateRequest`，`@AssertTrue` 管「BAD 时 content 必填」跨字段约束）+ 不可变响应 DTO（`EvaluationEligibility`/`EvaluationHistorySummary`/`EvaluationSubmitResult`/`RatingSummary`/`RatingRoleSummary`）+ VO 枚举 `EvaluationEligibilityReason`（归 dto，第六批准则）；`dto/user/UserPublicProfile`。
- `service/EvaluationService` + impl — 提交校验（参与者/已完成/未重复评价，唯一约束 `DuplicateKeyException` 兜底并发）、**被评价人后端推导**（我评对手方，角色据其发布方/接单方定 ACCEPTOR/PUBLISHER）、好评率动态聚合（按 `reviewee_id+reviewee_role` 分组，分母 0 时 rate=0.0，不缓存）、发评价通知（type=EVALUATION）。
- 共用增强：`common/ErrorReason` +`PICKUP_EVALUATION_NOT_ALLOWED`（契约 409）；`PickupService.queryPickupEvaluationContext`（回 `service/dto/PickupEvaluationContext`，跨模块不传 entity）；`UserService.getPublicProfile`；`dto/pickup/UserSummary` 补回 `ratingSummary` 字段（原注释「留到评价模块」）。
- `seed-dev.sql` 补评价种子：新增已双方互评的历史 COMPLETED 代取（id=4）+ 配套凭证文件（id=6/7）；evaluations 3 条（id=3 仅 bob→alice 一条、**留 alice→bob 入口空着演示提交**；id=4 双方互评齐全演示好评率）；notifications 3 条 EVALUATION 通知。

本批沉淀的设计准则（已落地）：
- **公开 GET 与受保护写接口同前缀共存时，用 GET-only 的「可选鉴权路径」而非 WebMvcConfig 白名单**：`/users/{userId}/profile` 公开 GET vs `/users/me/profile` 受保护 PUT 共用 `/users/*` 前缀，白名单按路径放行无法区分方法（单段通配 `*` 会匹配 `me`，把 PUT 一起放行）。改把三个公开读路径加进 `AuthInterceptor.OPTIONAL_AUTH_PATHS`（仅放宽 GET），PUT 因 `@CurrentUser` 解析不到上下文仍 401。这是对第五批「可选鉴权」准则的延用，注意它**只对 GET 生效**正是关键。
- **跨模块组合在 Controller 完成，避免 owner service 反向依赖**：`profile?includeRating=true` 需公开资料（UserService）+ 评价摘要（EvaluationService），在 `UserController` 组合两者，不让 `UserService` 反向依赖 `EvaluationService`（依赖方向保持 Evaluation→User/Pickup 单向）。
- **纯 Mockito 单测下避免 MP lambda 列投影**：`.select(Entity::getX)` 会在 wrapper 构建期触发 MP `TableInfo` 缓存查找，无 Spring/MP 引导的纯单测会抛 `can not find lambda cache`。聚合查询去掉 `.select()` 投影（评价行很窄，全列读取代价可忽略）；`.eq(lambda)` 因 SQL 在 mock 处被拦截、不会真正解析故不受影响。
- **`BaseMapper.insert` 重载致 Mockito `insert(any())` 歧义**：MP `insert(T)` 与 `insert(Collection)` 重载，裸 `any()` 编译期不明确，须写 `any(Evaluation.class)` 显式定型。

### 文档批次：积分体系替换支付宝沙箱（commit `ceffd72`，分支 `docs/p3-points-system`，从 dev 切出，**未 push、未合 dev**）

> 这是**纯 P3 契约文档改造**（不含代码），起因见下。代码尚未跟改——**当前代码与文档已不一致**，第六批改为「积分模块」（不再是支付宝）。

**起因（重要决策反转）**：第六批本要填第五批的支付桩接支付宝沙箱，深挖后确认**沙箱无法跑通完整闭环**——下单/回调/退款沙箱支持，但「平台结款给接单方」依赖分账/转账资质，沙箱商户结算账户无法充值，结款这环跑不通=闭环断=支付模块无展示价值；纯 Mock 又无意义。团队据此**反转 SRS v1.6 的人工决策**（当初「废除积分、接支付宝」），改回 **P2 时期的积分体系**：报酬用平台内虚拟积分计价，闭环变成平台内余额加减，必然自洽、零外部依赖、零合规风险。规则权威来源 = P2 分支 `srs_ieee830.md`（认证送 100/学号一次、签到 +5/日、确认完成积分转移）。

**已确认的落地决策**：
- 积分**纯虚拟、不可充值提现**（避开 SRS v1.5 被废除的「兑换比例无共识」坑）。
- 数据模型：`users` 加 `point_balance` 列 + 新建 `point_transactions` 流水表（**不引入 frozen 冻结列**）。
- 报酬流转：发布即从余额**扣减** → 完成转给接单方 → 取消退回发布方（**发布即扣，完成才入账**，非冻结模型）。
- 有报酬 amount **必须 >0**（PAID→amount≥1，UNPAID→空，不允许 0，否则与无报酬重复）—— 这是相对 P2「报酬可为 0」的 **P4 收敛**。
- 签到本批纳入（每日 +5、按 `last_check_in_date` 去重）。
- 积分流水分页支持按 `type` 筛选。

**改了 4 份 P3 文档**（一致性 grep 全检通过：无意外支付残留、`PointTransactionType` 五值四文档齐全、`PickupStatus` 四态、YAML 合法无悬空 ref）：
- `api_design.yaml`：删 3 接口（`/payments/alipay/notify`、`/payments/{id}`、`/pickup-requests/{id}/payment-entry`）；`/users/me/transactions`→`/users/me/point-transactions`（加 `type` 筛选）；新增 `GET /users/me/point-balance`、`POST /users/me/check-in`；`UserMe` 加 `pointBalance`；删 schema `PaymentRecord`/`PickupPaymentResult`/`TransactionPage`/`PaymentStatus`，新增 `PointTransactionItem`/`PointTransactionPage`/`PointTransactionType`/`CheckInResult`；删枚举 `WAITING_PAYMENT`/`PAYMENT_EXPIRED`；`PickupCreateResult` 删 payEntry/expireAt；发布加 409 `INSUFFICIENT_POINTS`；确认完成/取消响应删 `paymentStatus`。**`BusinessType` 保留**（通知/评价/文件仍复用）。**`NotificationType.PAYMENT` 保留**（语义改积分/报酬类，三文档枚举一致、零代码连锁）。
- `database_design.md`：删 `payment_records` 表 + `pickup_requests.payment_id`/相关 FK/UK；`users` 加 `point_balance BIGINT DEFAULT 0`（`CHECK>=0`）+ `last_check_in_date DATE`；新建 `point_transactions` 表（`type/amount/balance_after/related_pickup_id`，含 `idx_point_tx_user_created`/`_user_type_created`）；ER 图/枚举§7/索引§9/隐私§10/追踪§11/AI审查§12 全同步。
- `class_design.md`：`PaymentService`/`PaymentGateway`/`AlipaySandboxPaymentGateway` → `PointService`；§3.2 适配器模式 → **积分变动模板方法**（保留「适配器已移除」演进说明）；`User` 加 `pointBalance`/`lastCheckInDate` + `deductPoints`/`addPoints`；`PickupRequest` 删 `paymentId`/`markWaitingPayment`；SOLID 表/追踪表/协作图同步。
- `detailed_design.md`（**原方案漏了、grep 自检时发现补改**——它是 P3 整合版摘要，60 处支付命中）：§三范围、§四类表、§5 SOLID/模式、§六 API 分组（Payments→Points）、§七 数据库、§八流程闭环（§8.3 发布有偿改积分扣减、§8.6 结款、§8.7 取消退回）、§9 AI审查、§十一致性结论 全对齐。

**未做（明确留待）**：**代码返工**——第五批 `PaymentService`/`PaymentServiceImpl` 桩、`PaymentRecord` 实体/mapper、`PickupServiceImpl` 4 处支付调用点（createPrepay/cancelWaitingPayment/refundPayment/settlePayment）、相关测试、`seed-dev.sql`、entity 枚举（`PaymentStatus` 删、新增 `PointTransactionType`）全部要改。好消息：`PickupService` 只依赖 `PaymentService` 接口，换 `PointService` 时调用点改动可控。未改 P1/P2 及 `prompt_log.md`/`ai_reflection_log.md`（历史过程记录，刻意保持原样）。

> **以上「未做」已在第八批完成（见下）。**

### 第八批：积分模块（替代支付桩，commit `45b41c0`，分支 `feature/point-module`，从 dev 切出）

本批把第五批的「支付桩」整套替换为平台虚拟积分体系，落地上面「文档批次」已先行改完的 P3 契约。最终 `./mvnw test` → **99 passed**（新增 8：`PointServiceImplTest`；净增 7，含 Pickup 测试改写后的发布扣减/取消退回/余额不足用例）。

- **新建积分地基**：`entity/PointTransaction` + `entity/enums/PointTransactionType`（五值）+ `mapper/PointTransactionMapper`；`User` 加 `pointBalance`/`lastCheckInDate` 字段 + `deductPoints`/`addPoints` 领域方法（含防御性非负校验）。
- **`PointService` + impl**：`grantInitialPoints`(+100)/`checkIn`(+5，按 `lastCheckInDate` 去重)/`getBalance`/`spendForPublish`/`refundForCancel`/`transferOnComplete`/`queryTransactions`。内部模板 `applyChange` 固定「读用户→改余额（领域方法）→**条件更新**落库→写流水（含 balanceAfter）」骨架；余额落库用 `WHERE id=? AND point_balance=<旧值>` 条件更新防并发超扣（与代取状态条件更新一致，不引 version 列）。
- **3 个接口**（全对齐 `api_design.yaml`，均挂 `UserController`/`/users/me` 下、委托 `PointService`，遵「路径前缀归属」准则）：`GET /users/me/point-balance`、`POST /users/me/check-in`、`GET /users/me/point-transactions`（`type` 筛选 + 分页）。流水分页直接返回 `PageResult<PointTransactionItem>`（契约 `PointTransactionPage` = PageResult + list，与通知模块同模式）。
- **`dto/point/`**：`PointBalanceResponse`/`CheckInResult`/`PointTransactionItem`（entity→VO 的 `from`，`type` 用枚举不用 String，第六批准则）。`UserMeResponse` 加 `pointBalance`（契约 `UserMe` 新增字段）。
- **接入触发点**：`VerificationReviewServiceImpl.approve` 审核通过后调 `grantInitialPoints`（每学号一次：审核通过本身每学号唯一，故一次性赠送；驳回不赠送）。
- **代取返工**（`PickupServiceImpl`）：
  - 发布有报酬：先 insert 拿 pickupId（流水需 relatedPickupId）→ `markWaitingAccept` → `spendForPublish`，**直接进 WAITING_ACCEPT，无待支付态**；余额不足 `spendForPublish` 抛 409 `INSUFFICIENT_POINTS`、事务整体回滚。
  - 确认完成有报酬：`transferOnComplete(publisher, acceptor, amount, pickupId)` 只对**接单方**入账 `INCOME_COMPLETE`（发布方扣减已在发布时记 `SPEND_PUBLISH`，不二次扣）。
  - 取消：**仅 WAITING_ACCEPT 可取消**（无待支付态）；有报酬 `refundForCancel` 退回发布方 `REFUND_CANCEL`。
  - 金额→积分转换 `toPoints`：`rewardAmount.longValueExact()`（DTO 校验保证 1-200 整数元）。
- **删支付桩**：`PaymentService`/`PaymentServiceImpl`/`PaymentRecord`/`PaymentRecordMapper`/`PaymentStatus`/`PrepayResult` 全删；`PickupStatus` 删 `WAITING_PAYMENT`（剩 4 态）、`PickupCancelReason` 删 `PAYMENT_EXPIRED`；`PickupCreateResult` 删 payEntry/expireAt、`CompletionConfirmResult`/`PickupCancelResult` 删 paymentStatus（契约这两个响应分别为 `{status,completedAt}`/`{status,cancelReason}`）。
- **共用增强**：`common/ErrorReason` +`INSUFFICIENT_POINTS`/`ALREADY_CHECKED_IN_TODAY`（均契约 409）。`NotificationType.PAYMENT` 保留（语义改积分/报酬类，文档已定）。
- **schema.sql**：删 `payment_records` 表；`users` 加 `point_balance BIGINT NOT NULL DEFAULT 0`(CHECK>=0)/`last_check_in_date DATE`；新建 `point_transactions` 表；`pickup_requests` 删 `payment_id`/FK/UK，status & cancel_reason CHECK 收敛为 4 态/3 因。**seed-dev.sql**：删 payment_records TRUNCATE + payment_id 列；alice/bob 预置 `point_balance=100` + 各一条 `EARN_VERIFICATION` 流水。

本批沉淀的设计准则（已落地）：
- **积分余额并发安全用「余额条件更新」**：`update(entity, wrapper.eq(point_balance, 旧值))`，影响行数 0 视为并发改变抛 409——MP `update(et, wrapper)` 的 SET 取 et 非空字段（新余额）、WHERE 取 wrapper（带旧余额），与代取 `updateOnStatus` 同模式。签到去重同理用「旧签到日期」作条件，双保险防同日重复签到。
- **发布有报酬必须先 insert 再扣积分**：流水 `relatedPickupId` 需要 pickupId，故顺序是 insert→markWaitingAccept→spendForPublish；三者同一 `@Transactional`，扣减失败整体回滚（不会留下已发布但没扣款的脏单）。
- **完成只对接单方入账，不二次扣发布方**：发布即扣（`SPEND_PUBLISH`），完成只记接单方 `INCOME_COMPLETE`。`transferOnComplete` 的 `payerId` 仅作语义对齐，实现不动发布方余额。对应 class_design「SPEND 已在发布时记，完成记 INCOME_COMPLETE」。
- **金额(DECIMAL)→积分(long) 用 `longValueExact`**：DTO 已校验 1-200，`longValueExact` 对含非零小数兜底抛异常；`10.00` 这类整数值（scale=2 但无小数部分）可正常转换。

### 第八批边界修正：PointService 不再注入 UserMapper（commit `3015587`，同分支 `feature/point-module`）

> 起因：用户审查时指出 `PointServiceImpl` 直接注入 `UserMapper` 读写 `users.point_balance`/`last_check_in_date`，违反第四批准则「用户表读取/变更收口 owner service，其他模块不注入 UserMapper」。张力：积分余额字段物理上长在 users 表（P3 既定，不拆表）。`./mvnw test` → **107 passed**（新增 8：PointServiceImplTest +1、UserServiceImplTest +7）。

- **`UserService` 加 2 个积分原语**（owner service 收口用户表写入）：
  - `getPointBalance(userId)`（读）。
  - `applyPointChange(userId, delta, checkInDate)` → 返回 `balanceAfter`：读用户→`User.add/deductPoints`（领域方法）→**条件更新**（`WHERE point_balance=旧值`；`checkInDate!=null` 再带「旧签到日期」条件）。余额不足→409 `INSUFFICIENT_POINTS`、重复签到→409 `ALREADY_CHECKED_IN_TODAY`、并发→409、用户不存在→404，**全在此翻译**。复用既有 `requireUser` + `User` 富领域方法。
- **`PointServiceImpl` 删 `UserMapper`、改注 `UserService`**：各变动方法（赠送/签到/发布扣减/退回/完成入账）退为「调 `applyPointChange` 拿 balanceAfter → 写流水」；删 `requireUser`/`requireBalanceUpdated`/`applyChange` 私有方法。只剩注入 `UserService` + `PointTransactionMapper`（积分流水是本模块自有表）。
- **范围**：只改 PointService 链路。`AuthServiceImpl` 的 `UserMapper`（登录/注册查用户，User 尚未创建、属身份根服务）作为合理特例**保留不动**。现 `UserMapper` 仅 `UserServiceImpl`(owner) + `AuthServiceImpl`(特例) 注入。
- **零契约/行为变更**：不动 schema、API、DTO 字段、事务边界（`applyPointChange` 自身 `@Transactional`，经 REQUIRED 并入外层 pickup/verification 事务，原子性与重构前一致）。

本批沉淀的设计准则（已落地）：
- **「字段物理寄存在某表」≠「该表的 owner 拥有该字段的所有写入语义」要分开看**：积分余额列在 users 表 ⇒ 账户级写入（条件更新、余额非负、签到去重）归 `UserService`（users 的 owner）；但「为什么变、变多少、写什么流水」的积分业务语义归 `PointService`。跨模块协作：业务模块调 owner 的**完整领域原语**（`applyPointChange` 一次往返完成读+改+条件落库并回 balanceAfter），**不要**把「读余额 / 条件更新」拆成两个原语让调用方跨 service 拼装——那样破坏原子性、把并发条件（旧值）暴露到模块边界外。
- **owner service 的积分原语用「带签到日期参数」区分两种变动语义**：`applyPointChange(.., checkInDate)` 一个方法同时覆盖普通积分变动（checkInDate=null，仅余额条件）和签到变动（checkInDate≠null，余额+签到日期同笔、带旧日期条件防重复签到），避免为签到单开一个近乎重复的原语。

Common 公共层 + 鉴权基础设施已就绪，下一步按 Sprint 任务板 **P0 顺序逐模块纵向打通**（每个模块：dto + service 接口&实现 + controller + test）：

1. ~~**用户模块**~~ —— ✅ 第三批（注册/登录/`/users/me`）+ 第四批（资料编辑/实名认证）已完成。
2. ~~**文件上传**~~ —— ✅ 第四批已完成（文件存储元数据 + 上传 + 受控读取）。
3. ~~**代取请求核心流转**~~ —— ✅ 第五批已完成（发布→浏览→接单→完成→取消；有报酬到 WAITING_PAYMENT 占位）。**注意：WAITING_PAYMENT 桩将随积分改造移除**。
4. ~~**积分模块（替代原「支付/支付宝沙箱」）**~~ ✅ 第八批已完成（`PointService` + 流水/余额、签到、3 接口、代取返工、删支付桩）。**代码现已与「文档批次」改完的 P3 契约一致**。
5. ~~**评价**~~ —— ✅ 第七批已完成（双方互评 + 好评率聚合 + 用户公开主页/评价列表）。
6. ~~**站内通知**~~ —— ✅ 第六批已完成（列表/未读数/标记已读 + 认证审核/代取流转触发点）。

**待办（非阻塞，需用户批准才动）**：
- **本文档批次分支 `docs/p3-points-system`（`ceffd72`）未 push、未合 dev**。
- **代码已跟随文档完成积分返工**（第八批 `feature/point-module` `45b41c0`，未 push、未合 dev）——**代码与积分契约已一致**。合 dev 时注意：本批改 `seed-dev.sql`、删 `payment_records`/`PickupStatus.WAITING_PAYMENT`，与 `docs/p3-points-system` 文档批次配套合入。
- 未 push、未合 dev 的分支：`feature/notification-module`（`d51a9a1`，第六批）、`feature/dev-seed-data`（`471bdef`，种子脚本）、`feature/evaluation-module`（`f0b8fda`，第七批）、`docs/p3-points-system`（`ceffd72`，文档批次）、`feature/point-module`（积分模块 `45b41c0` + 边界修正 `3015587`，第八批）。注意第六/七批 + 积分返工都改了 `seed-dev.sql`，合并留意。
- `.gitignore` 第 40 行写的是 `backend/uploads/`，但联调产生的目录在**仓库根** `uploads/`，未被忽略；新出现根目录 `deploy/`、`backend/Dockerfile` 同样未跟踪。需补忽略项。
- 第四批 doc-drift 清单见第四批进度块末尾。

动手前先对着 `docs/P3/api_design.yaml` 列清该模块的设计决策，等用户批准再写。**新模块直接套用「第四批设计准则」**，不要再犯校验下沉、返回 entity、裸字符串 reason 等坑。
