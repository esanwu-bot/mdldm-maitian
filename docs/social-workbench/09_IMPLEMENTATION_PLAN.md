# 实施计划

## 1. 阶段总览

| 阶段 | 目标 | 通过条件 |
|---|---|---|
| P0 | 审计、API 调研、架构、OAuth、P1 计划 | Reviewer 批准本设计 |
| P1 | LinkedIn 端到端 PoC | `REAL_ACCOUNT_PASS` 或明确的权限阻塞清单 |
| P2 | YouTube + Facebook | 两平台各自 `REAL_ACCOUNT_PASS` |
| P3 | Analytics + UTM + 询盘归因 | 数据可追溯、归因可复算 |
| P4 | X + Reddit + 完整工作台 | 成本/规则/权限均通过 |

本文件重点展开 P0/P1；P2–P4 只给范围与门禁。

## 2. P0 交付（本轮）

### 2.1 文件

```text
docs/social-workbench/
  00_EXECUTIVE_SUMMARY.md
  01_EXISTING_SYSTEM_AUDIT.md
  02_TARGET_ARCHITECTURE.md
  03_PLATFORM_API_MATRIX.md
  04_DATA_MODEL.md
  05_PUBLISH_STATE_MACHINE.md
  06_OAUTH_SECURITY.md
  07_ANALYTICS_ATTRIBUTION.md
  08_COST_AND_RISK.md
  09_IMPLEMENTATION_PLAN.md
  10_ACCEPTANCE_TESTS.md
  11_CHANGELOG.md
  12_OAUTH2_AUTHORIZATION_ARCHITECTURE.md
```

### 2.2 P0 完成标准

- [x] 原仓库模块与调用链审计
- [x] Chrome 耦合点清单
- [x] 发布/Analytics 耦合点清单
- [x] 可保留/必须重构清单
- [x] SQLite 模型与迁移风险
- [x] 目标架构图
- [x] 五平台 API/SDK 矩阵（含证据等级）
- [x] 成本与权限风险
- [x] P1 详细计划
- [x] OAuth2 三层授权架构
- [ ] Reviewer 审核通过（待）

## 3. P1：LinkedIn 端到端 PoC

### 3.1 范围

只做 LinkedIn，只做官方 API；只绑定一个测试 Company Page；只发布人工批准的测试内容。

不做：

- 不做 YouTube/Facebook/X/Reddit。
- 不做批量发布、自动互动、评论。
- 不做真实 Morrishk 官方账号（除非项目负责人明确授权测试内容）。
- 不做完整工作台 UI（只做最小账号/发布/日志页）。

### 3.2 前置条件（人工，非代码）

| 编号 | 条件 | 责任 | 验收证据 |
|---|---|---|---|
| P1-A1 | LinkedIn 开发者账号 | 项目负责人 | 开发者后台截图/应用 ID |
| P1-A2 | 创建应用 | 项目负责人 | Client ID（Secret 不进 Git） |
| P1-A3 | 关联 Morrishk Company Page | 项目负责人 | Page 管理员可见应用 |
| P1-A4 | 申请 Community Management API 开发层 | 项目负责人 | 批准邮件/后台状态 |
| P1-A5 | 测试 Page（建议独立测试页） | 项目负责人 | Page URN |
| P1-A6 | 测试内容与审批链 | 内容/商品负责人 | Ds Bit `RELEASE_APPROVED` + 商品 `PUBLIC_ELIGIBLE` + 社媒 `SOCIAL_CONTENT_APPROVED` 三项记录 |
| P1-A7 | Redirect URI 白名单 | 开发 | `https://<host>/oauth/linkedin/callback` |
| P1-A8 | 部署环境与域名 | 运维 | HTTPS 可访问 |
| P1-A9 | 内部 OAuth 能力（仅当 P1 需要读商品/技术数据） | 平台/安全 | 按 `12` 的 A/B/C 三种情况之一提供 AS/网关与凭证引用 |

> `P1-A3/A5/A6` 依赖 `13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md` 中已核实的真实账号；账号未提供时，P1 只能做到 `MOCK_PASS`，不能选定或连接真实 Page。

### 3.3 实施步骤

#### Step 1：配置与密钥

- `config` 只存非敏感项：Client ID、redirect URI、API 版本、Page 候选。
- `LINKEDIN_CLIENT_SECRET` 从环境变量读取。
- 启动时校验：缺失则禁用 LinkedIn 功能并显示原因，不崩溃。

交付：配置示例、环境变量清单、启动校验。

#### Step 2：OAuth

- `GET /oauth/linkedin/start`：生成 `state`，绑定会话，跳转授权页。
- `GET /oauth/linkedin/callback`：校验 `state`、错误参数、code；服务端换 Token。
- 读取身份与组织访问角色：OIDC 使用 `/v2/userinfo` + `openid profile email`；旧版成员资料使用 `/v2/me` + `r_liteprofile`/实际批准的 `r_basicprofile`；不得混用。
- 账号选择页：展示可管理的 Page，用户确认绑定。
- 保存加密 Token。

测试：

- state 不匹配 → 401。
- code 过期 → 明确错误。
- 用户取消 → 友好提示。
- 多 Page → 必须选择。

#### Step 3：Token 存储与刷新

- AES-256-GCM 加密；KEK 从环境变量读取。
- 60 天到期提醒（提前 7 天）。
- 无法程序化刷新时，账号状态置 `reauth_required`，阻止发布。

测试：

- 加密/解密往返。
- 数据库转储无明文。
- 过期 Token 调用 → 401 → 状态变更。
- 并发刷新（若后续支持）不破坏 Token。

#### Step 4：账号状态与能力

- `getAccountStatus()`：Token 有效、Scope、Page 角色、可发布性。
- `validateContent()`：文本长度、媒体类型、链接、必填字段。
- 明确返回 `UNSUPPORTED` 的能力（如原生定时，若确认不支持）。

#### Step 5：发布文本帖

- `POST /rest/posts`，`author=urn:li:organization:{page}`。
- 保存 `x-restli-id`、HTTP 状态、原始响应（脱敏）。
- 状态：`submitting → published`。
- 生成 `post_url`：`https://www.linkedin.com/feed/update/{urn}/`。

测试：

- 201 + URN → published。
- 400 参数错误 → failed，不重试。
- 401 → 刷新/重新授权。
- 403 → 角色/Scope 提示。
- 429 → 退避。

#### Step 6：图片帖（可选，P1 最小）

- Images API 上传 → `urn:li:image` → Posts API。
- 校验图片格式/大小；失败不发布文本替代。

#### Step 7：状态查询与对账

- `GET /rest/posts/{urn}?viewContext=AUTHOR`。
- `lifecycleState` 映射：`PUBLISHED` / `PUBLISH_REQUESTED` / `PUBLISH_FAILED`。
- 超时 → `unknown`；按 URN/最近帖子对账。

#### Step 8：幂等与防重

- `payload_hash` + 本地唯一键。
- 提交前查询最近组织帖子（时间窗 + 文本指纹）。
- 重复请求返回已有 job，不再次提交。
- `--force` 不进入新工作台；重发必须新建版本/审批。

#### Step 9：失败与恢复

- 401/403/429/5xx/超时分别处理。
- Worker 崩溃后租约过期 → `unknown`。
- 对账唯一匹配 → published；多条 → 人工。
- 所有动作写审计。

#### Step 10：最小工作台

页面：

- 账号中心：连接、状态、Scope、到期、撤销。
- 内容详情：审批状态、平台预览、发布按钮。
- 发布详情：job 状态、attempts、请求 ID、post URL、错误。
- 审计：授权/审批/发布。

权限：`admin` 连接账号；`editor` 创建内容；`reviewer` 审批；`viewer` 只读。

#### Step 11：测试

- 单元：状态机、加密、payload hash、错误映射。
- 契约：用本地 mock HTTP server 模拟 LinkedIn 201/400/401/403/429/5xx/超时。
- 集成：真实测试 Page 发布一条文本帖。
- 安全：日志/数据库无 Token 明文。

#### Step 12：真实验收

必须记录：

- 应用 ID、Page URN（脱敏）。
- 授权时间、Scope。
- 发布请求时间、HTTP 状态、Post URN、Post URL。
- 查询状态结果。
- 重复提交保护的证据（第二次不产生新帖）。
- Token 失效模拟结果。
- 限流/超时模拟结果。
- 测试帖处理（保留/删除）与审计。

### 3.4 P1 分支与提交

```text
分支：feat/morrishk-linkedin-api-poc
提交 1：docs: add P1 LinkedIn PoC plan（如设计有更新）
提交 2：feat(auth): add OAuth manager and encrypted token store
提交 3：feat(linkedin): add provider adapter
提交 4：feat(publish): add job state machine and dedup
提交 5：feat(web): add minimal account/content/publish pages
提交 6：test: add unit/contract/integration tests
提交 7：docs: add P1 test report and evidence
```

设计文件与代码分开提交；每个提交附变更文件清单、测试命令、结果。

### 3.5 P1 不通过的条件

- Mock 通过但无真实账号证据，却宣称 P1 完成。
- 用浏览器自动化替代官方 API。
- 重复提交产生两条帖子。
- 超时后自动重试导致重复。
- Token 明文入库/入日志。
- 未审批内容被发布。
- 账号/Page 混淆。

## 4. P2：YouTube + Facebook

### 4.1 YouTube

- OAuth + `youtube.upload`。
- `videos.insert` resumable upload。
- 视频处理状态轮询（`processingDetails`）。
- 定时：`publishAt` + `private`。
- 配额监控（上传 100/日；其他 10,000 units/日）。
- Analytics：views/likes/comments/shares/subscribers/watch time；impressions 待确认。

### 4.2 Facebook

- Page Token + `pages_manage_posts`。
- 文本/链接/图片/视频。
- 定时：`published=false` + `scheduled_publish_time`。
- Insights：P2 启动前锁定 2026-06-15 后仍可用的指标。
- 应用审核/企业验证。

### 4.3 通过条件

- 两平台各自真实测试账号 `REAL_ACCOUNT_PASS`。
- 视频异步状态可恢复。
- 不把 YouTube `views` 与 Facebook `impressions` 相加。

## 5. P3：Analytics 与归因

- 采集器按平台/账号/帖子拉取。
- 原始 + 标准化 + registry。
- UTM 生成器（campaign/content）。
- Morrishk 站点事件接入。
- 询盘状态回写。
- 归因报表（last/first-touch）。
- 验收：每个值可追溯；unavailable 不为 0。

## 6. P4：X、Reddit 与完整工作台

- X：预算闸门、无链接优先、超时对账、成本对账。
- Reddit：草稿 + 人工确认；无 Analytics 不伪造。**未取得商业 API 协议/明确许可前，不注册生产应用、不调用 API、不采集数据。**
- 完整工作台：内容库、审批队列、日历、账号中心、审计、RBAC。
- 通过条件：成本不超预算；Reddit 商业协议/规则合规；X 无重复发帖。

## 7. 每阶段交付模板

```text
阶段：
分支：
Commit SHA：
变更文件清单：
关键 diff：
测试命令：
实际测试结果：
未完成事项：
风险与限制：
验收等级：CODE_PASS / MOCK_PASS / SANDBOX_PASS / REAL_ACCOUNT_PASS
Reviewer 结论：待审核
```

## 8. 里程碑与门禁

```text
P0 设计审核
  └─ P1 前置条件关闭
       └─ P1 实施 + 测试
            └─ P1 独立审核（真实证据）
                 └─ P2
                      └─ P2 审核
                           └─ P3
                                └─ P3 审核
                                     └─ P4
                                          └─ 最终验收
```

任何阶段未通过，不得进入下一阶段；不得用 Mock 结果替代真实账号证据。
