# OAuth 2.0 统一授权体系架构

适用范围：社媒工作台访问 Ds Bit、Morrishk API 的内部授权；Morrishk 用户进入工作台的身份授权；LinkedIn / Facebook / YouTube / X / Reddit 的外部平台授权。  
本文件是设计，不包含生产实施；未经 Reviewer 审核通过，不进入 OAuth 生产实施。

## 1. 总体架构

```text
┌────────────────────────── Morrishk 统一身份（OIDC） ──────────────────────────┐
│  /authorize  /token  /jwks  /revoke  /userinfo                                 │
│  用户登录 → Authorization Code + PKCE → 工作台 BFF 会话                        │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │ OIDC
                                ▼
┌────────────────────────── 社媒工作台（BFF + Worker） ─────────────────────────┐
│  user-session       RBAC（admin/editor/reviewer/analyst/viewer）               │
│  oauth.manager      外部平台授权（LinkedIn/Meta/Google/X/Reddit）              │
│  token.store        AES-256-GCM 加密                                            │
│  internal-client    Client Credentials（Ds Bit / Morrishk 各自独立）           │
└───────┬───────────────────────────────┬───────────────────────────────┬───────┘
        │ Client Credentials            │ Client Credentials            │ 外部 OAuth
        ▼                               ▼                               ▼
┌──────────────────┐          ┌──────────────────┐          ┌──────────────────────┐
│ Ds Bit API       │          │ Morrishk API     │          │ LinkedIn / Meta /    │
│ resource=dsbit   │          │ resource=morrishk│          │ Google / X / Reddit  │
│ scope=technical  │          │ scope=product /  │          │ 各自独立 Token        │
│       .read      │          │ attribution      │          │                      │
└──────────────────┘          └──────────────────┘          └──────────────────────┘
```

三层授权**互不共用 Token**：

1. 内部系统 Token 只用于服务间 API。
2. 用户 Token 只用于工作台会话。
3. 外部平台 Token 只用于对应平台 API。

## 2. 内部系统：Client Credentials

### 2.0 前置：Token 由谁签发

**在实施前必须确认 Ds Bit 与 Morrishk 是否已经具备 OAuth 2.0 Authorization Server、Token Endpoint 和 Resource Server 校验能力。** 工作台只是 OAuth Client，不能通过新增一张配置表“实现 OAuth2”。

| 情况 | 现状 | 方案 | 不能做的事 |
|---|---|---|---|
| A | 已有 OAuth 2.0 AS / Token Endpoint / RS | 在 AS 注册独立客户端；工作台按标准流程获取 Token | 不得共用客户端或 Token |
| B | 只有 API Key / 静态 JWT / 网关签名 | 建设兼容认证网关，或迁移到 OAuth；明确过渡期与风险 | 不得把 API Key 包装成“OAuth2 已完成” |
| C | 两边都没有 OAuth 服务 | 单独建设内部授权服务（AS + Token Endpoint + JWKS + 客户端管理 + 审计），再接入工作台 | 不得只在工作台加配置表就宣称完成 |

Token 签发方负责：

- 客户端注册、Secret 哈希、Scope 授予、轮换与吊销；
- Token 签发、有效期、签名/加密、JWKS；
- Resource Server 侧的 `iss`、`aud/resource`、`scope`、`exp` 校验。

工作台负责：

- 从 Secret Manager/环境变量取**出站**客户端凭证；
- 获取并缓存 Token；
- 按最小 Scope 调用 API；
- 不保存 Secret 哈希（哈希属于 AS）。

> `audience` 不是 OAuth 2.0 的通用标准参数。优先使用 RFC 8707 的 `resource` 参数；如果实际 AS 只支持 `audience` 扩展，必须在对接文档中明确记录，不能假定所有 AS 都接受。

### 2.1 客户端

| Client ID | Resource（RFC 8707） | Scope | 用途 |
|---|---|---|---|
| `morrishk-social-workbench` | `workbench` | `workbench.read` | 工作台自身服务（可选） |
| `morrishk-social-workbench-dsbit` | `https://<dsbit-host>/api` | `technical.read` | 读取 Ds Bit 技术数据 |
| `morrishk-social-workbench-morrishk` | `https://<morrishk-host>/api` | `product.read` | 读取商品主数据 |
| `morrishk-social-workbench-attribution` | `https://<morrishk-host>/api` | `attribution.write` | 回传询盘归因事件（独立客户端） |

实际客户端数量可按部署调整，但 **Ds Bit 与 Morrishk 必须使用不同的 Client ID、Secret、Scope 和 Token**。归因写入使用独立客户端与 Scope，即使该客户端被攻破也不能读取商品/技术数据。

客户端密钥不存放在工作台数据库：

| 项 | 存放位置 |
|---|---|
| Client Secret 明文 | Secret Manager / 环境变量；仅运行时内存使用 |
| Client Secret 哈希 | Authorization Server 的客户端注册表 |
| 工作台配置 | 只存 `client_id`、`resource`、`token_endpoint`、`scopes`、`secret_ref` |

### 2.2 流程

```text
Workbench                         Authorization Server
   │  POST <token_endpoint>           │
   │  grant_type=client_credentials   │
   │  client_id=...                   │
   │  client_secret=...（从 Secret Manager 取） │
   │  resource=https://<dsbit-host>/api │
   │  scope=technical.read            │
   ├─────────────────────────────────▶│
   │                                  │ 校验 client、resource、scope
   │  200 { access_token, expires_in }│
   │◀─────────────────────────────────┤
   │ 缓存至 expires_in - 60s           │
   │ 调 API：Authorization: Bearer     │
```

要求：

- Token 有效期建议 **15 分钟**；不签发 refresh token。
- 工作台在内存缓存；到期前 60 秒重新获取；不得把 Token 写入日志/数据库/前端。
- 并发获取使用 single-flight，避免风暴。
- Token 为 JWT 时，API 校验 `iss`、`aud`/`resource`、`exp`、`nbf`、`scope`、签名（JWKS）。
- 不透明 Token 时，API 通过 introspection 校验。
- API 端必须校验 Resource/Audience 绑定，防止 Ds Bit Token 被 Morrishk API 接受。
- 若 AS 使用 `audience` 扩展而非 `resource`，在配置与文档中记录为“AS 扩展”，不得写成 OAuth 标准。

### 2.3 权限边界

| 系统 | 允许 | 禁止 |
|---|---|---|
| Ds Bit API | `technical.read` 读取已发布技术数据、参数、页码证据 | 任何写入；读取未审核/内部草稿 |
| Morrishk API | `product.read` 读取商品主数据与商品公开状态 `PUBLIC_ELIGIBLE` | 修改商品、价格、库存、分类 |
| 归因接口 | `attribution.write` 写入 `inquiry_id`/事件 | 读取 CRM 明细、修改询盘状态 |

硬规则：

1. 工作台**不得**修改 Ds Bit 技术数据。
2. 工作台**不得**修改 Morrishk 商品主数据。
3. 只有同时满足 Ds Bit `RELEASE_APPROVED`、商品 `PUBLIC_ELIGIBLE`、社媒内容 `SOCIAL_CONTENT_APPROVED`，才可创建发布任务；任何一项不能替代另外两项。
4. 归因写入使用独立客户端/Scope；即使被攻破也不能改商品/技术数据。

### 2.4 客户端密钥

- Secret 使用高熵随机值（≥ 32 bytes），只在 Secret Manager/环境变量中。
- **Secret 哈希由 Authorization Server 保存**；工作台只保存 `secret_ref`，不保存 Secret 或哈希。
- 禁止写入 Git、前端、日志、镜像层。
- 轮换时由 AS 支持两个 active secret 并存；工作台通过 `secret_ref` 切换到新值。
- 紧急吊销：AS 立即吊销客户端；工作台进入只读并告警。

## 3. Morrishk 用户登录（OIDC）

### 3.1 流程

```text
浏览器                 工作台 BFF                 Morrishk OIDC
  │  GET /login           │                           │
  │──────────────────────▶│ 生成 state/nonce/PKCE     │
  │  302 /authorize       │                           │
  │──────────────────────────────────────────────────▶│
  │  登录 + 同意          │                           │
  │  302 callback?code    │                           │
  │──────────────────────▶│ 服务端换 token（含 secret）│
  │                       │──────────────────────────▶│
  │                       │ 校验 id_token（iss/aud/   │
  │                       │ nonce/exp/signature）      │
  │                       │ 建立会话（HttpOnly Cookie）│
  │  302 /               │                           │
  │◀──────────────────────│                           │
```

要求：

- Web 应用使用 **Authorization Code + PKCE**。
- `client_secret` 只存在 BFF；浏览器不持有。
- `state`、`nonce` 一次性，绑定会话。
- ID Token 校验签名、`iss`、`aud`、`exp`、`nonce`。
- 会话 Cookie：`HttpOnly; Secure; SameSite=Lax`；会话服务端存储。
- 登出：清除本地会话；必要时调 IdP `end_session_endpoint`。
- 若 Morrishk 暂无 OIDC，先实现适配接口与本地临时登录，但生产前必须接入统一身份；不得自建长期密码库。

### 3.2 RBAC

| 角色 | 内容创建 | 审批 | 发布 | 账号授权 | 分析 | 审计 |
|---|---|---|---|---|---|---|
| `admin` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `editor` | ✓ | ✗ | ✗ | ✗ | ✓ | 仅自己 |
| `reviewer` | ✗ | ✓ | ✗ | ✗ | ✓ | 仅相关内容 |
| `analyst` | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ |
| `viewer` | ✗ | ✗ | ✗ | ✗ | 只读 | ✗ |

约束：

- 审批人与创建人不得为同一人（至少 `reviewer` 不能批准自己创建的内容；最终规则待项目确认）。
- 发布动作必须由 `admin` 或明确授权的 `publisher` 执行。
- 账号授权（OAuth 连接）只允许 `admin`。
- 所有拒绝写审计。

## 4. 外部社交平台授权

外部平台流程、Scope、Token 生命周期详见 `06_OAUTH_SECURITY.md` 与 `03_PLATFORM_API_MATRIX.md`。本节只定义统一边界。

| 平台 | 授权类型 | Token 存储 | 发布 Scope | 分析 Scope |
|---|---|---|---|---|
| LinkedIn | Authorization Code | 加密 | `w_organization_social` | `rw_organization_admin`、`r_organization_social` |
| YouTube | Google OAuth | 加密 | `youtube.upload` | `yt-analytics.readonly` |
| Facebook | Facebook Login | 加密 | `pages_manage_posts`、`pages_read_engagement` | `read_insights` |
| X | OAuth2 + PKCE | 加密 | `tweet.write`、`media.write` | `tweet.read`、`users.read` |
| Reddit | OAuth2 | 加密 | `submit` | `read` |

> Reddit 的商业用途需单独协议（Data API Terms 2026-07-20 修订）；未取得明确许可/协议前，不启用 Reddit 外部授权与 API 调用。

规则：

1. 账号资产状态见 `13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md`；未核实归属/管理员的账号不得绑定。
2. 每个平台独立保存 `platform/account_id/account_name/access_token/refresh_token/expires_at/granted_scopes/token_status/last_refresh_at/authorization_owner`。
3. 发布与分析可使用独立授权记录；分析授权不得自动获得发布能力。
4. 账号绑定必须展示平台、账号名、类型、Scope，由用户确认。
5. 撤销必须同时停用该账号的 queued jobs。
6. 不得把外部平台 Token 用于内部 API。

## 5. Token 生命周期

```text
[未授权] ──授权──▶ [active] ──到期前刷新──▶ [active]
                     │                         │
                     ├─ invalid_grant ────────▶ [reauth_required]
                     ├─ 用户撤销 ─────────────▶ [revoked]
                     └─ 多次失败 ─────────────▶ [error]
```

| 阶段 | 内部 Token | 用户会话 | 外部 Token |
|---|---|---|---|
| 签发 | Client Credentials | OIDC Code | 平台 OAuth |
| 有效期 | 15 分钟 | 建议 8 小时/可配置 | 平台决定（X 2h、LinkedIn 60d） |
| 刷新 | 重新获取 | 滑动/重新登录 | refresh token 或重新授权 |
| 存储 | 内存 | 服务端会话 | AES-256-GCM |
| 撤销 | 客户端禁用/轮换 | 登出/会话失效 | 平台 revoke + 本地删除 |
| 监控 | 获取失败率 | 会话异常 | 到期、刷新失败、Scope 变化 |

## 6. Scope 权限矩阵

### 6.1 内部

| Scope | 持有者 | 读 | 写 | 数据范围 |
|---|---|---|---|---|
| `technical.read` | dsbit 客户端 | Ds Bit 已发布技术数据 | ✗ | 仅 `RELEASE_APPROVED` 关联文档 |
| `product.read` | morrishk 客户端 | 商品主数据 | ✗ | MPN/Brand/标题/分类/URL/图片/卖点/状态 |
| `attribution.write` | attribution 客户端 | ✗ | 归因事件 | 仅 `inquiry_id`/事件/UTM |
| `workbench.read` | workbench 自身 | 工作台配置 | ✗ | 非敏感 |

### 6.2 外部

| 平台 | 最小发布 Scope | 最小分析 Scope | 明确不申请 |
|---|---|---|---|
| LinkedIn | `w_organization_social` | `rw_organization_admin`、`r_organization_social` | `r_member_social`（closed） |
| YouTube | `youtube.upload` | `yt-analytics.readonly` | 宽泛账号权限 |
| Facebook | `pages_manage_posts`、`pages_read_engagement` | `read_insights` | 私信/好友 |
| X | `tweet.write`、`media.write` | `tweet.read`、`users.read` | DM |
| Reddit | `submit` | `read` | 私信 |

## 7. OAuth 安全设计

1. **State/Nonce/PKCE**：一次性、绑定会话、服务端校验。
2. **Redirect URI**：精确白名单，HTTPS，禁止通配符与开放重定向。
3. **Client Secret**：仅服务端；环境变量/密钥管理；不写前端/Git/日志。
4. **Token 加密**：AES-256-GCM，AAD 绑定账号与密钥版本。
5. **日志脱敏**：Token、Secret、Code、Authorization 头一律不记录。
6. **最小权限**：发布与分析分离；按需申请。
7. **CSRF**：所有状态变更请求校验 CSRF token/Origin。
8. **会话**：HttpOnly/Secure/SameSite；空闲超时；登出失效。
9. **速率限制**：登录、回调、Token 获取、发布接口分别限流。
10. **SSRF 防护**：目标 URL 只允许 Morrishk 域名白名单；媒体 URL 只允许受控存储。
11. **审计**：授权、刷新、撤销、审批、发布、删除、配置变更全部记录。
12. **依赖**：仅官方 SDK/REST；新增依赖需许可证与来源审查。
13. **告警**：刷新失败、Scope 变化、连续 401/403、异常授权、撤销、密钥轮换失败必须告警。

## 8. 密钥轮换与吊销

### 8.1 Client Secret 轮换

```text
1. 在 Auth Server 为客户端新增第二个 active secret
2. 更新工作台密钥配置（新 secret）
3. 重启/热加载，验证 Token 获取成功
4. 观察 24h，确认无旧 secret 调用
5. 吊销旧 secret
6. 写审计
```

要求：

- 支持两个 active secret 并存。
- 轮换窗口内不得中断发布。
- 轮换脚本 dry-run + 分批。
- 紧急情况可立即吊销全部 secret，工作台进入只读。

### 8.2 外部 Token 吊销

- 用户撤销：调平台 revoke（支持时）→ 删除密文 → 账号 `revoked` → 取消 queued jobs。
- 管理员强制撤销：同上，通知账号所有者。
- Scope 变更：旧 Token 标记失效，要求重新授权。

### 8.3 KEK 轮换

- 新增密钥版本；新写入用新版本；旧密文按需重加密。
- 保留旧版本解密能力；轮换失败可回退。
- 密钥不落盘明文；使用 KMS/环境变量注入。

## 9. 错误与授权失败处理

| 场景 | HTTP | 处理 |
|---|---|---|
| Client 认证失败 | 401 `invalid_client` | 不重试；告警；检查 secret |
| Scope 不足 | 403 `insufficient_scope` | 不重试；提示重新授权 |
| Token 过期 | 401 `invalid_token` | 内部 Token 重新获取；外部 Token 刷新一次 |
| Refresh 失败 `invalid_grant` | 400 | 账号 `reauth_required`；停止发布 |
| 用户取消授权 | 回调 `error=access_denied` | 友好提示；不创建账号 |
| State 不匹配 | 401 | 拒绝；安全告警 |
| 平台 429 | 429 | 退避；不标记账号失效 |
| 平台 5xx | 5xx | 退避；外部 Token 保持 |
| JWKS 不可用 | 503 | 缓存公钥；超过 TTL 则拒绝并告警 |

原则：

- 认证失败不自动循环重试。
- 授权失败不降级为浏览器自动化。
- 任何失败都不得泄漏 Token/Secret。
- 账号状态变化必须可见、可审计、可恢复。

## 10. 授权边界与数据访问矩阵

| 主体 | Ds Bit 技术数据 | Morrishk 商品数据 | 询盘/归因 | 社媒账号 | 工作台配置 |
|---|---|---|---|---|---|
| 工作台（dsbit 客户端） | 读（已发布） | ✗ | ✗ | ✗ | ✗ |
| 工作台（morrishk 客户端） | ✗ | 读 | ✗ | ✗ | ✗ |
| 工作台（attribution 客户端） | ✗ | ✗ | 写 | ✗ | ✗ |
| 工作台用户（admin） | 经工作台读 | 经工作台读 | 经工作台读 | 连接/撤销 | 读写 |
| 工作台用户（editor） | 经工作台读 | 经工作台读 | ✗ | ✗ | 内容 |
| 工作台用户（reviewer） | 经工作台读 | 经工作台读 | ✗ | ✗ | 审批 |
| Ds Bit | 自有 | ✗ | ✗ | ✗ | ✗ |
| Morrishk 主站 | ✗ | 自有 | 自有 | ✗ | ✗ |
| 外部平台 | ✗ | ✗ | ✗ | 自有 | ✗ |

**不可跨越的边界：**

1. 工作台不能写 Ds Bit 技术数据。
2. 工作台不能写 Morrishk 商品主数据。
3. 外部平台 Token 不能访问内部 API。
4. 用户会话不能直接持有外部平台 Token。
5. 归因写入不能读取 CRM 明细。
6. 任何系统不得共享同一组 Client Secret/Token。

## 11. 实施顺序（OAuth 部分）

```text
O-0 确认 Ds Bit/Morrishk 的 AS/Token Endpoint/JWKS/RS 能力；按 A/B/C 三种情况选择方案
O-1 注册内部客户端（dsbit / morrishk / attribution 独立）
O-2 实现 Client Credentials 获取与缓存（resource 参数）
O-3 API 侧校验 iss/resource 或 aud/scope/exp
O-4 接入 Morrishk OIDC（Code + PKCE）
O-5 建立 RBAC 与会话
O-6 实现外部平台 OAuth（P1 先 LinkedIn）
O-7 Token 加密存储与刷新
O-8 撤销、轮换、审计
O-9 安全测试
```

门禁：O-0..O-3 未完成，不进入真实平台发布；O-4..O-5 未完成，不开放多用户工作台。

## 12. 待确认

| 编号 | 事项 | 影响 |
|---|---|---|
| OQ-01 | Morrishk 是否已有 OIDC Issuer | 用户登录方案 |
| OQ-02 | Ds Bit / Morrishk 是否已有 OAuth 2.0 AS、Token Endpoint、JWKS 与 Resource Server 校验；若没有，选兼容网关、迁移还是自建授权服务 | 内部授权能否实施 |
| OQ-03 | 归因写入接口归属（Morrishk 主站/CRM） | Scope 与客户端 |
| OQ-04 | 审批人与创建人是否必须分离 | RBAC |
| OQ-05 | 生产部署是否可接 KMS/Secret Manager | Token/密钥安全 |
| OQ-06 | 外部平台应用由谁持有（公司/个人） | 资产归属与审核 |
| OQ-07 | 是否允许个人 LinkedIn 账号作为测试账号 | P1 测试范围 |
