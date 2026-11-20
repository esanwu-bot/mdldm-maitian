# Morrishk 社媒与视频平台账号资产盘点

| 项 | 内容 |
|---|---|
| 文档 | `13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md` |
| 版本 | P0-R1 初版 |
| 盘点日期 | 2026-11-20 |
| 盘点范围 | LinkedIn、Facebook、X、Reddit、Instagram、TikTok、YouTube、Vimeo，以及视频分发渠道 |
| 本轮性质 | 只盘点与取证；不注册账号、不申请授权、不修改账号、不发布内容、不购买套餐 |
| 结论状态 | **没有取得任何 Morrishk 真实社媒账号的账号名、主页 URL 或管理员证据**；台账以待确认占位记录建立 |

## 1. 执行摘要

本轮对工作区现有资料、Morrishk 官网公开页面和平台官方能力进行了核对，结论如下：

1. **工作区没有 Morrishk 社媒账号清单。** 项目资料只写到“社媒账号与视频素材由谁提供、归谁所有、谁审核”待确认，没有账号名称、URL、管理员或开发者应用记录。
2. **Morrishk 官网页脚只有平台首页链接**（`https://www.linkedin.com/`、`https://www.facebook.com/`、`https://www.instagram.com/`、`https://twitter.com/`），**没有指向任何 Morrishk 账号的 URL**。因此不能用官网证明账号存在，也不能据此推断账号不存在。
3. **没有发现任何已完成的 OAuth 授权、开发者应用或 API 发布证据。** 当前仓库的 API 预研 Demo 使用 Mock，真实发布状态为 `BLOCKED_CREDENTIAL`；本轮不改变该状态。
4. **P1 LinkedIn 无法选定真实账号。** 缺少 Company Page URL、组织 ID/URN、管理员角色、开发者应用与测试内容审批人。
5. **本轮不能回答“Morrishk 究竟有哪些账号”。** 按任务要求，缺少真实资料时建立待确认台账，不虚构账号；所有账号级结论标记 `UNKNOWN`。

**对 P1 的影响：** P1 的代码与 Mock 测试可以准备，但 `REAL_ACCOUNT_PASS` 在账号资产与管理员权限确认前不可能完成。

## 2. 盘点方法与证据等级

### 2.1 证据来源

| 来源 | 用途 | 证据等级 |
|---|---|---|
| 工作区项目资料 | 是否已有账号/管理员/开发者应用记录 | 内部资料 |
| `https://www.morrishk.net/` 公开首页 | 是否有公开账号链接 | 公开网页（2026-11-20 抓取） |
| 平台官方 API 文档 | 平台能力、Scope、配额 | 官方文档（见 `03`） |
| 项目方提供的账号/后台截图 | 账号存在性、归属、管理员 | **本轮未提供** |
| 真实 OAuth/API 调用 | 实际发布与 Analytics | **本轮未执行** |

### 2.2 证据等级

| 标记 | 含义 |
|---|---|
| `PUBLIC_EVIDENCE` | 公开网页/官方文档可直接核验 |
| `INTERNAL_STATEMENT` | 项目资料中的陈述，未被账号后台复核 |
| `PROJECT_PROVIDED` | 项目方提供的截图/后台信息（本轮无） |
| `DOC_VERIFIED` | 只有官方文档，未做真实账号授权测试 |
| `REAL_ACCOUNT_PASS` | 真实账号完成授权、发布/读取验证（本轮无） |
| `UNKNOWN` | 没有足够证据判断 |

**规则：** 账号存在不等于 API 可用；OAuth 成功不等于发布权限获批；文档存在不等于真实账号可调用。

### 2.3 五个独立核验层次

每个账号必须分别判断，不能合并成一个“已开通”：

| 平台 | 1. 账号存在性 | 2. 所有权 | 3. 管理权限 | 4. 开发者 API 条件 | 5. 实际 API 能力 |
|---|---|---|---|---|---|
| LinkedIn | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | 未验证 |
| Facebook | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | 未验证 |
| X | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | 未验证 |
| Reddit | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | 未验证；协议阻断 |
| Instagram | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | 未验证 |
| TikTok | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | 未验证 |
| YouTube | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | 未验证 |
| Vimeo | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | `UNKNOWN` | 未验证 |

说明：第 1–3 层需要项目方后台证据；第 4 层需要开发者后台/审核邮件；第 5 层需要真实授权后的 API 调用。本轮三层都没有证据，因此不能进入第 5 层。

## 3. 社交媒体平台总清单

| 平台 | 主要用途 | 账号类型 | 官方 API 能力 | Morrishk 账号证据 | 当前结论 |
|---|---|---|---|---|---|
| LinkedIn | 企业品牌、技术内容、B2B 获客 | Company Page / Personal | Posts API、Share Statistics | 无账号 URL/URN/管理员证据 | `UNKNOWN` |
| Facebook | 企业品牌、产品内容、广告流量 | Page | Page Feed/Photos/Video、Insights | 无 Page ID/管理员证据 | `UNKNOWN` |
| X | 技术资讯、品牌传播 | Brand/Personal | `POST /2/tweets`、媒体上传 | 无 handle/开发者项目证据 | `UNKNOWN` |
| Reddit | 技术社区、工程师讨论 | User | `/api/submit`、`/api/info` | 无用户名/subreddit/商业协议 | `UNKNOWN`；未签协议前阻断 |
| Instagram | 品牌视觉、图片/短视频 | Business/Creator | Content Publishing、Insights | 无 Professional Account/Page 绑定证据 | `UNKNOWN` |
| TikTok | 产品短视频、品牌曝光 | Business/Creator | Content Posting API | 无账号/开发者应用/审核证据 | `UNKNOWN` |
| Bluesky（补充评估） | 技术/开源社区 | Personal/Brand | AT Protocol；非本阶段官方 API 范围 | 无账号证据 | 不纳入首期 |
| Threads（补充评估） | Meta 生态文字/短内容 | Professional | 依赖 Meta 权限与账号 | 无账号证据 | 不纳入首期 |

## 4. 视频平台与分发渠道总清单

| 平台/渠道 | 内容形态 | 账号前提 | 上传能力 | 处理状态 | Analytics | 证据 | 结论 |
|---|---|---|---|---|---|---|---|
| YouTube | 长视频/短视频 | Channel / Brand Account | `videos.insert`；100 次/日 | `videos.list` + `processingDetails` | YouTube Analytics API | 官方文档 | `DOC_VERIFIED`，账号未知 |
| LinkedIn Video | 企业技术视频 | Company Page 管理员 | Videos API + Posts API | 帖子 `lifecycleState` | Share/Video Analytics | 官方文档 | `DOC_VERIFIED`，账号未知 |
| Facebook Video / Reels | Page 视频 | Page + `CREATE_CONTENT` | Video API / Page endpoints | Post 查询 | Page Insights | 官方文档 | `DOC_VERIFIED`，账号未知 |
| Instagram Reels | 短视频 | Professional Account + Meta 授权 | Content Publishing | 媒体容器状态 | Insights | 官方文档 | `DOC_VERIFIED`，账号未知 |
| TikTok | 短视频 | Business/Creator + 审核 | Content Posting API | 发布状态查询 | 平台分析（权限待核验） | 官方文档 | `DOC_VERIFIED`，账号未知 |
| Vimeo | 企业视频展示 | 企业账号 + 付费套餐 | Vimeo API（套餐相关） | 待核验 | 待核验 | 未调研到账号/套餐 | `UNKNOWN`，是否必要待确认 |

> 平台与分发渠道分开建模：例如 Facebook 是平台，Page 是账号，Facebook Video/Reels 是分发渠道；不重复创建账号记录。

## 5. 账号资产台账

### 5.1 说明

本轮**没有取得任何真实账号名称或 URL**。为避免虚构，以下每个平台只建立 **1 条待确认占位记录**，`account_name` 与 `account_url` 明确写“待项目方提供/未知”。项目方提供资料后，再按真实账号逐条展开，不能把占位记录当成真实账号。

字段取值：`UNKNOWN` = 无证据；`否` = 未核实；`未授权` = 当前没有 OAuth；`未验证` = 没有真实 API 调用。

账号状态枚举（后续按此维护）：

| 状态 | 含义 |
|---|---|
| `VERIFIED_ACTIVE` | 账号真实存在，归属及管理权限已核实 |
| `ACTIVE_UNVERIFIED` | 账号存在，归属或管理权限未核实 |
| `ACCESS_PENDING` | 账号存在，尚未获得管理访问权限 |
| `DEVELOPER_SETUP_PENDING` | 账号已确认，开发者应用未配置 |
| `OAUTH_PENDING` | 开发者应用存在，授权未完成 |
| `API_PERMISSION_PENDING` | 授权完成，发布/数据权限未获批 |
| `READY_FOR_POC` | 满足指定 PoC 的全部前置条件 |
| `SUSPENDED` | 账号受限或暂停 |
| `NOT_FOUND` | 已核实不存在 |
| `UNKNOWN` | 没有足够证据判断 |

### 5.2 身份与归属

| platform | account_name | account_url | account_type | account_owner | account_admin | account_status | login_status | ownership_verified | admin_access_verified |
|---|---|---|---|---|---|---|---|---|---|
| linkedin | 待项目方提供 | 未知 | Company Page（待确认） | 待确认 | 待确认 | `UNKNOWN` | 未核验 | 否 | 否 |
| facebook | 待项目方提供 | 未知 | Page（待确认） | 待确认 | 待确认 | `UNKNOWN` | 未核验 | 否 | 否 |
| x | 待项目方提供 | 未知 | Brand/Personal（待确认） | 待确认 | 待确认 | `UNKNOWN` | 未核验 | 否 | 否 |
| reddit | 待项目方提供 | 未知 | User（待确认） | 待确认 | 待确认 | `UNKNOWN` | 未核验 | 否 | 否 |
| instagram | 待项目方提供 | 未知 | Business/Creator（待确认） | 待确认 | 待确认 | `UNKNOWN` | 未核验 | 否 | 否 |
| tiktok | 待项目方提供 | 未知 | Business/Creator（待确认） | 待确认 | 待确认 | `UNKNOWN` | 未核验 | 否 | 否 |
| youtube | 待项目方提供 | 未知 | Channel/Brand Account（待确认） | 待确认 | 待确认 | `UNKNOWN` | 未核验 | 否 | 否 |
| vimeo | 待项目方提供 | 未知 | 企业账号（待确认） | 待确认 | 待确认 | `UNKNOWN` | 未核验 | 否 | 否 |

### 5.3 API、授权与发布

| platform | developer_app | oauth_status | granted_scopes | publishing_status | analytics_status | api_tier | cost | last_verified_at | evidence | blockers | next_action |
|---|---|---|---|---|---|---|---|---|---|---|---|
| linkedin | 未知（未核验） | 未授权 | 无 | 未验证 | 未验证 | 未确认 | API 调用不按次收费；审核门槛见 03 | 2026-11-20（仅资料核对） | 官网页脚无账号链接；工作区无账号记录 | Page URL/URN、管理员、应用、审批人均缺 | 提供 Company Page URL/URN、管理员角色、应用 ID、测试 Page |
| facebook | 未知（未核验） | 未授权 | 无 | 未验证 | 未验证 | 未确认 | 无按次费用；审核/企业验证 | 2026-11-20 | 同上 | Page ID、Meta App、企业验证、审核状态 | 提供 Page ID/管理员/App ID/审核状态 |
| x | 未知（未核验） | 未授权 | 无 | 未验证 | 未验证 | 未确认 | 按量：$0.015/帖，带 URL $0.200/帖；读取 $0.005 | 2026-11-20 | 同上 | handle、开发者项目、预算 | 提供 handle、项目 ID、月度预算 |
| reddit | 未知（未核验） | 未授权 | 无 | 未验证 | 无发布级 Analytics | 未确认 | 商业用途需单独协议 | 2026-11-20 | 官方页本环境 403；工作区无账号 | 账号、subreddit、商业协议 | 先签协议/获许可，再提供账号 |
| instagram | 未知（未核验） | 未授权 | 无 | 未验证 | 未验证 | 未确认 | 无按次费用；Meta 审核 | 2026-11-20 | 工作区无账号记录 | Professional Account、Page 绑定、App | 提供 IG 账号/Page 绑定/App |
| tiktok | 未知（未核验） | 未授权 | 无 | 未验证 | 未验证 | 未确认 | 免费额度/付费规则待核验 | 2026-11-20 | 工作区无账号记录 | 账号、开发者应用、Content Posting 审核 | 提供账号类型、App、审核状态 |
| youtube | 未知（未核验） | 未授权 | 无 | 未验证 | 未验证 | 未确认（上传 100/日；其他 10,000 units/日） | 配额内免费 | 2026-11-20 | 工作区无频道记录 | Channel ID、Brand Account、GCP 项目、审核 | 提供频道 ID、GCP 项目、OAuth 状态 |
| vimeo | 未知（未核验） | 未授权 | 无 | 未验证 | 未验证 | 未确认 | 套餐相关，待核验 | 2026-11-20 | 无账号/套餐证据 | 是否有实际需求未确认 | 先确认是否需要 Vimeo |

### 5.4 分发渠道（与账号分离）

| channel_id | platform | account | content_type | publish_capability | analytics_capability | status |
|---|---|---|---|---|---|---|
| linkedin-page-post | linkedin | 待确认 | 文本/图片/视频/文章 | 文档支持；未实测 | Share Statistics；未实测 | `UNKNOWN` |
| facebook-page-feed | facebook | 待确认 | 文本/链接/图片 | 文档支持；未实测 | Page Insights；未实测 | `UNKNOWN` |
| facebook-video-reels | facebook | 待确认 | 视频/Reels | 文档支持；未实测 | Page/Video Insights；未实测 | `UNKNOWN` |
| instagram-reels | instagram | 待确认 | Reels | 文档支持；未实测 | Insights；未实测 | `UNKNOWN` |
| x-post | x | 待确认 | 文本/图片/视频 | 文档支持；未实测 | 公开指标；未实测 | `UNKNOWN` |
| youtube-video | youtube | 待确认 | 长视频/短视频 | `videos.insert`；未实测 | YouTube Analytics；未实测 | `UNKNOWN` |
| tiktok-video | tiktok | 待确认 | 短视频 | Content Posting；未实测 | 待核验 | `UNKNOWN` |
| linkedin-video | linkedin | 待确认 | 视频 | Videos API；未实测 | Video Analytics；未实测 | `UNKNOWN` |
| reddit-post | reddit | 待确认 | 讨论帖 | `submit`；未实测 | 无发布级 Analytics | `BLOCKED_TERMS` |

## 6. 平台账号状态统计

> 无数据写 `未知`，不写 0。下表的“账号数”不是 8；8 是待确认占位记录数，真实账号数量未知。

| 平台 | 账号数 | 已核实 | 待核实 | API 已授权 | 可发布 | 可读 Analytics | 下一步 |
|---|---:|---:|---:|---:|---:|---:|---|
| LinkedIn | 未知 | 未知 | 未知 | 未知 | 未知 | 未知 | 提供 Company Page URL/URN、管理员、开发者应用 |
| Facebook | 未知 | 未知 | 未知 | 未知 | 未知 | 未知 | 提供 Page ID、管理员、Meta App/审核 |
| X | 未知 | 未知 | 未知 | 未知 | 未知 | 未知 | 提供 handle、开发者项目、预算 |
| Reddit | 未知 | 未知 | 未知 | 未知 | 未知 | 未知 | 先签商业 API 协议，再提供账号/subreddit |
| Instagram | 未知 | 未知 | 未知 | 未知 | 未知 | 未知 | 提供 Professional Account、Page 绑定、App |
| TikTok | 未知 | 未知 | 未知 | 未知 | 未知 | 未知 | 提供账号类型、App、Content Posting 审核 |
| YouTube | 未知 | 未知 | 未知 | 未知 | 未知 | 未知 | 提供 Channel ID、Brand Account、GCP 项目 |
| Vimeo | 未知 | 未知 | 未知 | 未知 | 未知 | 未知 | 确认是否有实际需求与套餐 |

## 7. 管理员及归属待确认清单

| 平台 | 待确认项 | 需要的证据 | 不能做的事 |
|---|---|---|---|
| LinkedIn | Company Page 所属主体、管理员、组织 URN | 后台截图/Page URL/组织 ID | 不猜测 Page slug，不用个人号代替组织 |
| Facebook | Page 所属主体、`CREATE_CONTENT`/`ANALYZE` 角色 | Page 设置截图/Page ID | 不使用个人号发 Page 内容 |
| X | 品牌账号归属、开发者项目所有者 | 账号设置/项目截图 | 不使用个人账号冒充品牌 |
| Reddit | 运营账号、目标 subreddit、版规、商业协议 | 账号信息/协议 | 不批量注册、不跨版灌帖 |
| Instagram | Professional Account、与 Page 的绑定 | 账号类型截图/Page ID | 不把个人号当企业号 |
| TikTok | 账号类型、Content Posting 审核 | 后台/审核邮件 | 未审审核不假设可公开发布 |
| YouTube | Channel 所有者、Brand Account、GCP 项目 | 频道 URL/ID、项目 ID | 不把个人频道当企业频道 |
| Vimeo | 企业账号、套餐、API 需求 | 账号/套餐截图 | 不在无需求时采购 |

## 8. Developer App 状态清单

| 平台 | 是否已有 App | App 名称/ID | 审核状态 | 可发布 | 可读 Analytics | 证据 |
|---|---|---|---|---|---|---|
| LinkedIn | 未知 | 未知 | 未知 | 未知 | 未知 | 本轮未提供 |
| Facebook | 未知 | 未知 | 未知 | 未知 | 未知 | 本轮未提供 |
| X | 未知 | 未知 | 未知 | 未知 | 未知 | 本轮未提供 |
| Reddit | 未知 | 未知 | 未申请/未核验 | 未知 | 无发布级 Analytics | 本轮未提供 |
| Instagram | 未知 | 未知（随 Meta App） | 未知 | 未知 | 未知 | 本轮未提供 |
| TikTok | 未知 | 未知 | 未知 | 未知 | 未知 | 本轮未提供 |
| YouTube | 未知 | 未知 | 未知 | 未知 | 未知 | 本轮未提供 |
| Vimeo | 未知 | 未知 | 未知 | 未知 | 未知 | 本轮未提供 |

## 9. OAuth 与 API 权限状态

| 平台 | OAuth 状态 | 已授予 Scope | 发布权限 | Analytics 权限 | Token 状态 | 备注 |
|---|---|---|---|---|---|---|
| LinkedIn | 未授权 | 无 | 未验证 | 未验证 | 无 Token | 需要 Company Page 管理员与 Community Management 审核 |
| Facebook | 未授权 | 无 | 未验证 | 未验证 | 无 Token | 需要 Page Token、App Review、企业验证 |
| X | 未授权 | 无 | 未验证 | 未验证 | 无 Token | OAuth2 PKCE；按量计费 |
| Reddit | 未授权 | 无 | 未验证 | 无发布级 Analytics | 无 Token | 商业协议未签，阻断 |
| Instagram | 未授权 | 无 | 未验证 | 未验证 | 无 Token | 需 Meta App 与 Professional Account |
| TikTok | 未授权 | 无 | 未验证 | 未验证 | 无 Token | Content Posting 审核未确认 |
| YouTube | 未授权 | 无 | 未验证 | 未验证 | 无 Token | 需 GCP 项目与 OAuth 审核 |
| Vimeo | 未授权 | 无 | 未验证 | 未验证 | 无 Token | API/套餐待确认 |

## 10. 视频发布与 Analytics 能力矩阵

### 10.1 视频资产清单

| 平台 | 频道/账号 | 视频内容类型 | 当前视频数量 | 发布权限 | 上传 API | 处理状态 API | Analytics API | 状态 |
|---|---|---|---|---|---|---|---|---|
| YouTube | 待项目方提供 | Datasheet 解读/产品介绍/应用视频 | 未知 | 未知 | `videos.insert`（文档） | `processingDetails`（文档） | YouTube Analytics（文档） | `UNKNOWN` |
| LinkedIn Video | 待项目方提供 | 企业技术视频 | 未知 | 未知 | Videos API（文档） | 帖子状态（文档） | Share/Video Analytics（文档） | `UNKNOWN` |
| Facebook Video/Reels | 待项目方提供 | 产品/应用视频 | 未知 | 未知 | Video API（文档） | Post 查询（文档） | Page Insights（文档） | `UNKNOWN` |
| Instagram Reels | 待项目方提供 | 产品短视频 | 未知 | 未知 | Content Publishing（文档） | 容器状态（文档） | Insights（文档） | `UNKNOWN` |
| TikTok | 待项目方提供 | 产品短视频 | 未知 | 未知 | Content Posting（文档） | 待核验 | 待核验 | `UNKNOWN` |
| Vimeo | 待项目方提供 | 企业展示视频 | 未知 | 未知 | 待核验 | 待核验 | 待核验 | `UNKNOWN` |

### 10.2 能力矩阵

| 能力 | YouTube | LinkedIn Video | Facebook Video | Instagram Reels | TikTok | Vimeo |
|---|---|---|---|---|---|---|
| 视频上传 API | `videos.insert`（文档） | Videos API（文档） | Video API（文档） | Content Publishing（文档） | Content Posting（文档） | 待核验 |
| 处理状态 API | `processingDetails`（文档） | 帖子状态（文档） | Post 查询（文档） | 容器状态（文档） | 发布状态（待核验） | 待核验 |
| 封面上传 | 文档支持；未实测 | 文档支持；未实测 | 文档支持；未实测 | 文档支持；未实测 | 待核验 | 待核验 |
| 字幕 | 文档支持；未实测 | 待核验 | 待核验 | 待核验 | 待核验 | 待核验 |
| 视频 URL 回收 | 可构造/查询；未实测 | 可构造；未实测 | 可查询；未实测 | 可查询；未实测 | 待核验 | 待核验 |
| Analytics | YouTube Analytics（文档） | Share/Video Analytics（文档） | Page Insights（文档，指标退役） | Insights（文档） | 待核验 | 待核验 |
| 配额 | 上传 100/日；其他 10,000 units/日 | LinkedIn 开发层 500/应用/日、100/成员/日 | 动态 Platform/BUC | Meta 动态限流 | 待核验 | 套餐相关 |
| 真实账号证据 | **无** | **无** | **无** | **无** | **无** | **无** |

> 视频上传成功不等于 Analytics 权限已获得；本表全部为 `DOC_VERIFIED` 或 `UNKNOWN`，没有 `REAL_ACCOUNT_PASS`。

## 11. API 成本及配额摘要

| 平台 | 计费/配额 | 对本项目的影响 |
|---|---|---|
| LinkedIn | API 不按次收费；开发层 500/应用/日、100/成员/日；Standard 审核 | 账号/审核是主要瓶颈，不是费用 |
| YouTube | 上传 100 次/日；其他端点 10,000 units/日 | 视频量在配额内；Analytics 配额另计 |
| Facebook | 免费；动态 Platform/BUC | 审核、企业验证、指标退役是主要风险 |
| X | 按量：$0.015/帖；带 URL $0.200/帖；读取 $0.005 | 带链接帖子成本高；预算闸门 |
| Reddit | 商业协议；未获批前无额度 | 未签协议前阻断 |
| Instagram/TikTok/Vimeo | 免费额度/套餐待核验 | 账号与审核优先 |

详细见 `03_PLATFORM_API_MATRIX.md` 与 `08_COST_AND_RISK.md`。

## 12. 平台接入优先级

| 优先级 | 平台 | 理由 | 当前门禁 |
|---|---|---|---|
| P1 | LinkedIn | B2B 获客最直接；Posts/Share Statistics 能力完整 | 账号资产、Company Page 管理员、开发者应用、Community Management 审核 |
| P2 | YouTube | 技术视频与搜索长尾；API 能力明确 | Channel/Brand Account、GCP 项目、OAuth 审核 |
| P2 | Facebook | Page 发布与视频；Meta 生态 | Page 管理员、App Review、企业验证、Insights 指标退役 |
| P3 | Analytics/UTM | 归因闭环的基础 | 站点事件、CRM、各平台账号授权 |
| P4 | X | 技术传播；按量成本 | handle、开发者项目、预算 |
| P4 | Reddit | 工程师社区；无 Analytics | 商业协议、账号、subreddit 规则 |
| 评估 | Instagram/TikTok/Vimeo | 视觉/短视频；Vimeo 需求未确认 | Professional/Business 账号、审核、套餐 |

## 13. 项目方待提供信息

请按平台提供以下信息（**不要发送密码、Cookie、Token 或 Client Secret**）：

| 平台 | 需要提供 |
|---|---|
| LinkedIn | Company Page URL、组织数字 ID/URN、管理员姓名与角色、开发者应用 ID、Community Management 层级、测试 Page |
| Facebook | Page URL/ID、管理员、Meta App ID、企业验证状态、App Review 状态、`CREATE_CONTENT`/`ANALYZE` 角色 |
| Instagram | Professional Account 用户名/ID、与 Facebook Page 的绑定关系、Meta App ID、权限审核状态 |
| X | 品牌 handle、开发者项目 ID、当前 credits/预算、是否接受带 URL 帖子成本 |
| Reddit | 运营用户名、目标 subreddit、版规确认、商业 API 协议/许可 |
| TikTok | 账号类型、开发者应用 ID、Content Posting API 审核状态、可发布范围 |
| YouTube | Channel URL/ID、Brand Account 所有者、GCP 项目 ID、OAuth 同意屏幕状态、API 审核/配额状态 |
| Vimeo | 是否有实际需求、企业账号、套餐、API 用途 |

同时提供：

- 账号归属主体（公司/品牌/个人）与书面授权；
- 内容审批人（`SOCIAL_CONTENT_APPROVED`）与商品公开状态维护人（`PUBLIC_ELIGIBLE`）；
- 测试内容与测试账号的发布授权；
- 账号安全联系人。

## 14. 证据链接及核验日期

### 14.1 平台官方文档

| 平台 | 官方文档 URL | 核验日期 | 等级 |
|---|---|---|---|
| LinkedIn | `https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api` | 2026-11-20 | `DOC_VERIFIED` |
| YouTube | `https://developers.google.com/youtube/v3/docs/videos/insert`；`https://developers.google.com/youtube/analytics` | 2026-11-20 | `DOC_VERIFIED` |
| Facebook | `https://developers.facebook.com/docs/pages-api/posts`；`https://developers.facebook.com/docs/graph-api/reference/page/insights/` | 2026-11-20 | `DOC_VERIFIED` |
| X | `https://docs.x.com/x-api/posts/creation-of-a-post` | 2026-11-20 | `DOC_VERIFIED` |
| Reddit | `https://www.reddit.com/dev/api/` | 2026-11-20 | 本环境 403；待复核 |
| Instagram | `https://developers.facebook.com/docs/instagram-platform/content-publishing` | 2026-11-20 | `DOC_ONLY`，账号未验证 |
| TikTok | `https://developers.tiktok.com/doc/content-posting-api-get-started` | 2026-11-20 | `DOC_ONLY`，账号未验证 |
| Vimeo | `https://developer.vimeo.com/api/reference` | 2026-11-20 | `DOC_ONLY`，需求未确认 |

### 14.2 项目内证据

| 证据 | URL/位置 | 核验日期 | 结论 |
|---|---|---|---|
| Morrishk 官网首页 | `https://www.morrishk.net/` | 2026-11-20 | 页脚仅有 `linkedin.com`、`facebook.com`、`instagram.com`、`twitter.com` 平台首页链接，无账号 URL |
| 项目范围 | `00_project_baseline/project_scope.md:100` | 2026-11-20 | 社媒账号与视频素材由谁提供/归谁所有/谁审核，待确认 |
| 增长计划 | `01_business_plan/morrishk_growth_plan_v1.1.md:930` | 2026-11-20 | 社媒账号归属与权限待客户提供 |
| API 预研待确认 | `research/api-integration-poc/docs/OPEN_QUESTIONS.md` OQ-02/OQ-22 | 2026-11-20 | 账号所有权、管理员、LinkedIn 组织 URN 未提供 |
| 授权要求 | `research/api-integration-poc/docs/AUTH_AND_PERMISSION.md` §2.2 | 2026-11-20 | 组织身份不能从个人 token 推断，必须提供组织 URN/管理员 |
| 平台能力 | `docs/social-workbench/03_PLATFORM_API_MATRIX.md` | 2026-11-20 | 各平台官方 API/配额/权限 |

> 官网页脚链接只证明“页面有平台首页入口”，**不证明** Morrishk 拥有对应账号。

## 15. 未核实事项与风险

| 编号 | 事项 | 风险 | 处理 |
|---|---|---|---|
| INV-01 | 真实账号数量与名称 | 无法做账号级发布/分析 | 项目方提供清单 |
| INV-02 | 账号归属与管理员 | 发错账号/越权 | 书面授权 + 后台截图 |
| INV-03 | LinkedIn 组织 URN | 无法组织发帖 | 提供 Page URL/URN |
| INV-04 | 开发者应用与审核 | API 不可用 | 提供 App ID/审核状态 |
| INV-05 | OAuth 授权 | 无法读取/发布 | 按 06/12 实施 |
| INV-06 | Reddit 商业协议 | 法务/账号风险 | 未签协议前阻断 |
| INV-07 | 视频频道与 Brand Account | 上传到错误频道 | 提供 Channel ID/所有者 |
| INV-08 | 官网页脚链接误读 | 把平台首页当账号 | 本文件明确不据此认定账号存在 |
| INV-09 | 账号被限流/封禁 | 发布中断 | 合规运营、监控、备份渠道 |
| INV-10 | 个人号冒充企业号 | 品牌/合规风险 | 账号类型与归属核验 |

## 16. 对最终问题的直接回答

| # | 问题 | 当前回答 |
|---|---|---|
| 1 | Morrishk 目前究竟有哪些社交媒体账号？ | **无法确认**；工作区无账号清单，官网只有平台首页链接 |
| 2 | 哪些账号归 Morrishk 所有？ | **无法确认**；无归属证据 |
| 3 | 哪些账号已有管理员权限？ | **无法确认**；无后台证据 |
| 4 | 哪些账号可以申请官方 API？ | 平台层面均可申请；账号层面**未知**，取决于账号类型与开发者应用 |
| 5 | 哪些账号已完成 OAuth？ | **无**；本轮未发现任何 OAuth 记录 |
| 6 | 哪些账号能够真实发布内容？ | **无证据**；没有 `REAL_ACCOUNT_PASS` |
| 7 | 哪些账号能够读取 Analytics？ | **无证据**；仅文档能力，未真实调用 |
| 8 | 哪些视频频道可以通过 API 上传和管理视频？ | **无法确认**；YouTube 等能力有文档，但频道/授权未知 |
| 9 | 当前最大的账号和授权阻塞是什么？ | **账号资产（名称/URL/归属/管理员）未提供**；其次是开发者应用与平台审核 |
| 10 | 下一阶段 LinkedIn PoC 应该使用哪个真实账号？ | **无法选定**；需项目方提供 Company Page URL/URN、管理员、测试 Page 与内容审批人 |

## 17. 与现有设计文档的关系

- `02_TARGET_ARCHITECTURE.md`：账号资产是内容源与发布目标的前置；未确认账号不得创建 `publish_jobs`。
- `03_PLATFORM_API_MATRIX.md`：本文件只回答“Morrishk 是否有账号”，平台能力与配额以 `03` 为准。
- `06_OAUTH_SECURITY.md` / `12_OAUTH2_AUTHORIZATION_ARCHITECTURE.md`：账号绑定时必须记录 `platform/account_id/account_name/owner`，不能只用平台名。
- `08_COST_AND_RISK.md`：Reddit 商业协议、X 预算、LinkedIn 审核均为账号级门禁。
- `09_IMPLEMENTATION_PLAN.md`：P1-A3/A5/A6 依赖本文件；账号未确认时 P1 只能 `MOCK_PASS`。
- `10_ACCEPTANCE_TESTS.md`：`REAL_ACCOUNT_PASS` 必须引用本文件中已核实的账号记录。

## 18. 账号安全要求与本轮边界

- **不要求任何人向 Agent 发送密码、Cookie、Session、Refresh Token 或 Client Secret。**
- 登录状态由项目方在安全环境中确认；本轮不尝试登录。
- 管理员姓名/联系方式未获授权不猜测、不填写。
- 未注册任何账号或开发者应用。
- 未申请任何平台权限或 API 套餐。
- 未登录任何 Morrishk 账号。
- 未收集任何密码、Cookie、Session、Token 或 Client Secret。
- 未发布任何内容。
- 未把公开平台首页链接当成 Morrishk 账号证据。
- 未虚构账号名称、URL、管理员或粉丝数据。
