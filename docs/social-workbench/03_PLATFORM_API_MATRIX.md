# 平台官方 API / SDK 调研矩阵

调研日期：2026-11-20。  
证据等级：**A** = 本次直接抓取官方文档并引用原文；**B** = 官方文档存在但本环境无法直接抓取/仅抓到摘要；**C** = 需注册后以实际响应核验。  
本阶段**没有注册应用、没有调用任何平台 API、没有购买任何额度**。

> 平台能力存在不等于 Morrishk 已有账号。真实账号、管理员、归属与开发者应用状态见 `13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md`；本文件只回答“平台能不能接”，不回答“Morrishk 有没有这个账号”。

## 1. 总矩阵

| 平台 | 官方文档 | 发布 API | 状态/链接 | Analytics API | 认证 | 发布权限 | 账号前提 | 速率/配额 | 费用 | 证据 |
|---|---|---|---|---|---|---|---|---|---|---|
| LinkedIn | `learn.microsoft.com/linkedin/marketing/...` | `POST /rest/posts` | `GET /rest/posts/{urn}`；`x-restli-id` | `organizationalEntityShareStatistics` | OAuth 2.0 3-legged | `w_organization_social` / `w_member_social` | Company Page 管理员；Community Management API 审核 | 开发层 500 次/应用/日、100 次/成员/日 | API 不按次收费 | A |
| YouTube | `developers.google.com/youtube/v3` | `videos.insert` | `videos.list` + `processingDetails` | YouTube Analytics API | OAuth 2.0 | `youtube.upload` | Google Cloud 项目 + YouTube 频道 | 上传 100 次/日；其他 10,000 units/日 | 配额内免费 | A |
| Facebook | `developers.facebook.com/docs/pages-api` | `POST /{page-id}/feed`、`/photos`、`/videos` | `GET /{post-id}` | Page Insights | OAuth 2.0 + Page Token | `pages_manage_posts`、`pages_read_engagement` | Page 管理员；应用审核/企业验证 | 动态限流（Platform/BUC 取决于 Token；响应头监测） | 免费 | A |
| X | `docs.x.com` | `POST /2/tweets` | `GET /2/tweets/{id}` | 公开指标/自有内容读取 | OAuth 2.0 PKCE | `tweet.write`、`media.write` | X 开发者账号；按量 credits | 端点级限流，429 | **按量计费**：$0.015/帖；带 URL $0.200/帖 | A |
| Reddit | `reddit.com/dev/api` | `POST /api/submit` | `GET /api/info` | **无发布级 Analytics API** | OAuth 2.0 | `submit`、`read` | Reddit 账号、应用注册、subreddit 规则 | 约 100 QPM/客户端（待核验） | 商业用途需单独协议；未获批前阻断 API | B/C |

## 2. LinkedIn

### 2.1 官方依据

- Posts API：`https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api`
- Community Management 概览：`https://learn.microsoft.com/en-us/linkedin/marketing/community-management/community-management-overview`
- Increasing Access（权限与层级）：`https://learn.microsoft.com/en-us/linkedin/marketing/increasing-access`
- 3-legged OAuth：`https://learn.microsoft.com/en-us/linkedin/shared/authentication/authorization-code-flow`
- Rate limits：`https://learn.microsoft.com/en-us/linkedin/shared/api-guide/concepts/rate-limits`
- Share Statistics：`https://learn.microsoft.com/en-us/linkedin/marketing/community-management/organizations/share-statistics`

### 2.2 发布

| 项 | 结论 |
|---|---|
| 端点 | `POST https://api.linkedin.com/rest/posts` |
| 必需头 | `Authorization: Bearer`、`Linkedin-Version: YYYYMM`、`X-Restli-Protocol-Version: 2.0.0`、`Content-Type: application/json` |
| 成功响应 | `201`，响应头 `x-restli-id` 为 Post URN |
| 内容类型 | 文本、图片、视频、文档、文章、MultiImage、Poll（organic） |
| 媒体 | 先经 Images/Videos/Documents API 上传，拿到 `urn:li:image` / `urn:li:video` / `urn:li:document` 再发帖 |
| 组织帖 | `author` 为 `urn:li:organization:{id}` |
| 个人帖 | `author` 为 `urn:li:person:{id}`，需 `w_member_social` |
| 原生定时 | **本次抓取的 Posts API 文档未确认 `scheduledAt` 字段**；按“工作台定时、到点提交”设计 |
| 删除 | `DELETE /rest/posts/{urn}`，幂等，返回 `204` |
| 查询 | `GET /rest/posts/{urn}?viewContext=AUTHOR`；`lifecycleState` 可为 `DRAFT/PUBLISHED/PUBLISH_REQUESTED/PUBLISH_FAILED` |

### 2.3 权限与访问

| 权限 | 用途 | 角色限制 |
|---|---|---|
| `w_organization_social` | 代表组织发帖/评论/点赞 | Page `ADMINISTRATOR`、`DIRECT_SPONSORED_CONTENT_POSTER`、`CONTENT_ADMIN` |
| `r_organization_social` | 读取组织帖子与互动 | 同上 |
| `rw_organization_admin` | 读取 Page 统计 | `ADMINISTRATOR` |
| `w_member_social` | 代表个人发帖 | 成员本人 |
| `r_member_social` | 读取个人帖子 | **closed**，官方明确暂不接受申请 |
| `r_member_postAnalytics` | 个人帖子分析 | Community Management API |
| `openid profile email` | OIDC 身份读取（`/v2/userinfo`） | LinkedIn OIDC |
| `r_liteprofile` / `r_basicprofile` | 旧版成员资料 API（`/v2/me`） | 按实际 API 与后台批准；不得与 OIDC 混用 |

访问层级（官方原文要点）：

- Community Management API 是 **Vetted Product**，分 Development / Standard 两级。
- Development：所有 API **500 次/应用/24h**，**100 次/成员/24h**；BATCH_GET 不允许；Social Actions webhook 推送禁用；须在 12 个月内完成集成与测试。
- Standard：无上述限制；申请时需提交**每个用例的 screencast**。
- LinkedIn 保留审核与选择伙伴的权利，即使满足最低要求也可能不升级。

### 2.4 Analytics

`GET https://api.linkedin.com/rest/organizationalEntityShareStatistics?q=organizationalEntity&organizationalEntity={urn}`

- 指标：`impressionCount`、`uniqueImpressionsCount`、`clickCount`、`likeCount`、`commentCount`、`shareCount`、`engagement`。
- 仅 organic；赞助内容需 Ad Analytics。
- 仅覆盖**滚动 12 个月**。
- 时间维度最小为 `DAY`；可按 `shares` / `ugcPosts` 查询单帖，但**单帖不支持 time-bound**。
- 权限：`rw_organization_admin`。

### 2.5 Token

- 授权码换 Token：`POST https://www.linkedin.com/oauth/v2/accessToken`。
- Access Token 当前为 **60 天**（`expires_in=5184000`）。
- **程序化 refresh token 仅对有限伙伴开放**；普通应用到期后需重新走授权。
- OAuth 错误页显示授权码 30 分钟有效；必须校验 `state`。

### 2.6 P1 结论

LinkedIn 是唯一一个“发布、查询、分析、删除”都能用同一套官方 REST API 完成、且不需要视频转码链路的平台。适合作为端到端 PoC。  
最大风险不是技术，而是**应用审核、Page 管理员权限、Standard Tier 升级与 60 天 Token 再授权**。

## 3. YouTube

### 3.1 官方依据

- Data API 配额：`https://developers.google.com/youtube/v3/determine_quota_cost`
- 上传指南：`https://developers.google.com/youtube/v3/guides/uploading_a_video`
- Analytics API：`https://developers.google.com/youtube/analytics`
- Analytics 指标：`https://developers.google.com/youtube/analytics/metrics`

### 3.2 发布

| 项 | 结论 |
|---|---|
| 端点 | `videos.insert`（`part=snippet,status`） |
| 上传方式 | 支持 resumable upload；瞬时错误用指数退避 |
| OAuth Scope | `https://www.googleapis.com/auth/youtube.upload` |
| 读取 | `videos.list`（`part=status,processingDetails`） |
| 状态 | 上传成功后进入异步处理；需轮询 `processingDetails.processingStatus` |
| 定时 | `status.publishAt` + `privacyStatus=private` 实现定时公开 |
| 配额（2026-11-20 官方页） | `videos.insert` **100 次/日**；`search.list` **100 次/日**；其他端点合计 **10,000 units/日**；`videos.list` 1 unit |
| 费用 | 配额内不收费 |

> 注意：本次抓取到的官方配额页已不是旧的“`videos.insert` 1600 units”表述，而是按端点给出每日次数。实现前需在 Google Cloud Console 再次核对项目实际配额。

### 3.3 Analytics

- 独立 API：YouTube Analytics API。
- 常用指标：views、likes、comments、shares、subscribers gained/lost、estimated minutes watched、average view duration。
- **impressions 不是本次抓取页面的核心指标**；若需要曝光/CTR，必须单独确认报表与权限，不能假定可用。
- Scope：`https://www.googleapis.com/auth/yt-analytics.readonly`。
- Analytics API 配额与 Data API 分开；本次未取得明确数值，标记 C，实施前在 Cloud Console 核实。

### 3.4 账号与审核

- Google Cloud 项目 + 启用 YouTube Data API v3 / YouTube Analytics API。
- OAuth 同意屏幕；未验证应用仅限测试用户，公开使用需完成 Google 验证流程。
- 上传公开视频可能触发 YouTube API Services 审核；以 Google 实际要求为准。

## 4. Facebook（Meta）

### 4.1 官方依据

- Pages API Posts：`https://developers.facebook.com/docs/pages-api/posts`
- Page Overview：`https://developers.facebook.com/docs/pages/overview`
- Page Insights：`https://developers.facebook.com/docs/graph-api/reference/page/insights/`
- Rate limiting：`https://developers.facebook.com/docs/graph-api/overview/rate-limiting/`

### 4.2 发布

| 内容 | 端点 | 说明 |
|---|---|---|
| 文本/链接 | `POST /{page-id}/feed` | `message`、`link` |
| 图片 | `POST /{page-id}/photos` | `url` 或 `source` |
| 视频 | Video API / `POST /{page-id}/videos` | 建议使用官方 Video API |
| 定时 | `published=false` + `scheduled_publish_time` | 官方文档：10 分钟至 30 天 |
| 权限 | `pages_manage_posts`、`pages_read_engagement` | Page Token |
| Page 任务 | `CREATE_CONTENT` | 发帖；`ANALYZE` 读洞察 |
| 审核 | Facebook Login + App Review（advanced access） | 上线前必须完成 |
| 企业验证 | 需要 | 以 Meta 后台要求为准 |

### 4.3 Analytics

- `GET /{page-id}/insights?metric=...`。
- 权限：`read_insights`、`pages_read_engagement`；用户需 `ANALYZE` 任务。
- 常见帖子指标：`post_impressions`、`post_impressions_unique`、`post_clicks`、`post_reactions_*`、`post_video_views` 等。
- **重要变更**：官方页面标注，2026-06-15 起多项 Page Insights 指标将在所有 API 版本退役；部分 unique 指标在 Graph API v25 以上已标记 deprecated。
- 数据限制：Page 需 100+ likes；仅两年历史；unique 值为近似值。
- 结论：P2 开始前必须锁定“当前版本仍可用”的指标清单，不能照搬旧教程。

### 4.4 速率（动态限流，不写死额度）

Meta Graph API 的限流按应用、用户、Page 等维度计算；Page 请求可能适用 Platform 或 BUC 限额，**取决于 Token 类型与调用方**。官方文档给出的公式（如 `200 × Users`、`4800 × Engaged Users`）是口径示例，不应作为固定配额硬编码。

实现要求：

1. 每次响应读取 `X-App-Usage`、`X-Business-Use-Case-Usage`，按百分比动态控制。
2. 建议阈值：80% 告警、90% 降速、95% 暂停该账号发布并等待窗口重置。
3. 识别限流错误码（如 4/17/32/613 等，按当前版本文档）并指数退避。
4. 不把“200×用户 / 4800×活跃用户”写成保证值或验收标准。
5. 发布与 Insights 采集分别计量，避免互相挤占。

## 5. X

### 5.1 官方依据

- 创建帖子：`https://docs.x.com/x-api/posts/creation-of-a-post`
- 媒体上传：`https://docs.x.com/x-api/media/upload-media`
- OAuth 2.0 PKCE：`https://docs.x.com/fundamentals/authentication/oauth-2-0/authorization-code`
- 定价：`https://docs.x.com/x-api/getting-started/pricing`

### 5.2 发布

| 项 | 结论 |
|---|---|
| 端点 | `POST https://api.x.com/2/tweets` |
| 认证 | OAuth 2.0 user context |
| Scope | `tweet.write`；读取需 `tweet.read`、`users.read`；刷新需 `offline.access` |
| 媒体 | `POST https://api.x.com/2/media/upload`，scope `media.write`；图片 5 MB（`tweet_image`） |
| 视频 | 上限取决于账号 Premium 状态，不取决于 API 套餐 |
| 引用帖 | 官方文档标注 quote posting 需 Enterprise |
| 原生定时 | **无**；由工作台调度器到点提交 |
| Access Token | 默认 2 小时；`offline.access` 才有 refresh token |

### 5.3 成本（官方定价页，2026-11-20）

- 模式：**pay-per-usage credits，无订阅**。
- 创建帖子：`$0.015 / request`。
- **带 URL 的帖子：`$0.200 / request`**。
- 普通读取：`$0.005 / resource`。**自有内容读取 `$0.001 / resource` 本次未能从价格表确认，标记 C，不纳入预算。**
- 新用户：保存卡片并首次购买后可获 `$20` credits。
- 媒体上传的单独计费未在定价页列出，标记 C，需在账单中核验。

**对 Morrishk 的影响**：UTM 外链是 B2B 转化的关键，但每条带 URL 帖子 `$0.20`。若每月 30 条带链接帖子，仅发帖约 `$6.00`；读取按当前可确认的 `$0.005/resource` 保守估算，30 天 × 30 帖 × `$0.005` ≈ `$4.50`。合计约 `$10.50/月`（不含媒体与重试）。必须设置月度预算与“无链接/带链接”策略。

### 5.4 速率与幂等

- 端点级限流，超限 `429`；具体窗口以响应头为准。
- `POST /2/tweets` 不支持 idempotency key。
- 超时后必须用“最近自有帖子”查询确认，不能直接重试。

## 6. Reddit

### 6.1 官方依据与取证限制

- API 文档：`https://www.reddit.com/dev/api/`
- Data API Terms：`https://redditinc.com/policies/data-api-terms`
- 本次从当前网络环境访问以上页面均返回 **403/仅导航**，**未取得可引用的官方原文**。因此本节除明确标注外均为 **B/C 级**，必须在注册应用后用实际响应头与文档复核。
- Reviewer 指出的关键事实：Data API Terms 于 **2026-07-20** 修订；商业用途需要**单独协议**；平台保留自行设定 API 使用限制的权利。Morrishk 属于商业营销场景，**在获得明确许可/协议前不得正式实施 Reddit API**。

### 6.2 能力判断

| 项 | 结论 | 证据 |
|---|---|---|
| 发布 | `POST /api/submit`（`kind=self` 或 `kind=link`） | 公开 API 惯例，待核验 |
| Scope | `submit`、`read` | 待核验 |
| 认证 | OAuth 2.0（installed app / script） | 待核验 |
| 速率 | 约 **100 queries/minute per OAuth client**（未认证约 10 QPM） | 社区与二手资料；官方页本环境不可读 |
| 发布级 Analytics | **没有** impressions/reach API | 官方能力范围判断 |
| 可读数据 | `GET /api/info` 可读 score、comments 等公开字段 | 待核验 |
| 商业使用 | Data API Terms（2026-07-20 修订）要求商业用途单独协议；未获批前阻断 | 法务确认/协议 |
| 自动化边界 | 只允许“生成草稿 + 人工确认 + 人工发布/确认后发布” | 项目约束 |

### 6.3 Reddit 结论

Reddit 不进入自动发布主链路。P4 只做：

0. **前置门禁**：取得商业 API 协议/明确许可，并完成法务确认；未完成前不注册生产应用、不调用 API、不采集数据。
1. 生成讨论草稿与证据引用。
2. 人工选择 subreddit、阅读版规、确认账号身份。
3. 人工点击发布，或在工作台内由授权人二次确认后调用 `submit`。
4. 发布后只记录 URL、score、comment count；不承诺 impressions。
5. 禁止批量、跨版重复、伪造用户口吻。

## 7. SDK 选择

| 平台 | 官方 SDK | 本项目选择 |
|---|---|---|
| LinkedIn | 无官方 Node.js SDK | 官方 REST 封装（`fetch`） |
| YouTube | Google 官方 `googleapis` | 使用 `googleapis` 或 REST |
| Facebook | Meta 官方 `facebook-nodejs-business-sdk` | 优先官方 SDK；亦可 REST |
| X | 无官方 Node.js SDK | 官方 REST 封装 |
| Reddit | 无官方 Node.js SDK | 官方 REST 封装 |

约束：新增 npm 依赖必须单独提交依赖清单与许可证审查；P0 不安装。

## 8. 证据缺口

| 编号 | 缺口 | 处理 |
|---|---|---|
| E-01 | LinkedIn Posts API 是否支持原生定时 | 在 P1 沙箱/真实账号验证；默认按无原生定时设计 |
| E-02 | LinkedIn Standard Tier 实际审核周期 | 以提交后 LinkedIn 回复为准 |
| E-03 | YouTube Analytics API 具体配额数值 | 在 Cloud Console 核实 |
| E-04 | YouTube 公开上传是否需要 API audit | 以 Google 审核邮件为准 |
| E-05 | Meta Insights 2026-06-15 后仍可用的指标清单 | P2 启动前用目标 Graph 版本实测 |
| E-06 | X 媒体上传是否单独计费 | 以 X 账单/控制台为准 |
| E-07 | Reddit 当前官方速率、商业协议与数据保留要求 | 注册前先取得协议；以官方页面与响应头复核 |
| E-08 | 各平台是否提供 idempotency key | P1/P2 实测；默认按不支持设计 |
