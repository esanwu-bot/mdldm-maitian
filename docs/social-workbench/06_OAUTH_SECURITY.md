# OAuth 与 Token 安全设计（外部平台）

本文件聚焦外部社交平台的 OAuth 实现与 Token 安全；内部系统 Client Credentials 与 Morrishk 用户登录见 `12_OAUTH2_AUTHORIZATION_ARCHITECTURE.md`。

## 1. 威胁模型

| 威胁 | 场景 | 控制 |
|---|---|---|
| 授权码拦截 | 回调被窃听/开放重定向 | 严格 redirect URI、`state`、一次性 code、短期有效 |
| CSRF | 攻击者诱导用户绑定其账号 | `state` 绑定会话；回调校验 |
| Token 泄漏 | 日志、错误页、Git、前端 | 加密存储、日志脱敏、禁止前端持有 |
| Token 重放 | 备份/调试文件泄漏 | 密钥版本、轮换、撤销、最小 Scope |
| 账号混淆 | 多 Page/多频道 | `provider_account_id` + `account_name` 双确认 |
| 越权发布 | 个人号误发到公司 Page | 账号类型校验 + 审批 + 发布前展示目标账号 |
| 恶意内容 | 被篡改的文案/链接 | `payload_hash` + 审批记录 + 发布前二次校验 |
| 供应链 | 第三方 SDK 窃取 Token | 仅官方 SDK/REST；依赖锁定与许可证审查 |

## 2. 各平台授权流程

| 平台 | 流程 | PKCE | State | 备注 |
|---|---|---|---|---|
| LinkedIn | Authorization Code（3-legged） | 官方文档未要求 | 必须 | 用 Client Secret 换 Token；回调需 HTTPS 绝对地址 |
| YouTube | Google OAuth 2.0 Authorization Code | 支持/推荐 | 必须 | 可离线刷新；Scope 需审核 |
| Facebook | Facebook Login + Authorization Code | 支持（以当前版本文档为准） | 必须 | 换取长期用户 Token，再换 Page Token |
| X | OAuth 2.0 Authorization Code + PKCE | **必须** | 必须 | Access Token 2 小时；`offline.access` 才有 refresh |
| Reddit | OAuth 2.0 Authorization Code | 待核验 | 必须 | installed app / script；待注册核验 |

### 2.1 通用回调

```text
GET /oauth/{platform}/callback?code=...&state=...
  ├─ 校验 state（会话绑定、一次性）
  ├─ 校验错误参数（error/error_description）
  ├─ 用 code 换 token（服务端，绝不经过浏览器）
  ├─ 读取账号身份（/me、/v2/userinfo、Page/Channel 列表）
  ├─ 让用户选择要绑定的账号（多 Page/多频道时必须显式选择）
  ├─ 加密入库
  └─ 跳回账号中心（成功/失败原因）
```

### 2.2 账号身份确认

| 平台 | 身份接口 | 绑定字段 |
|---|---|---|
| LinkedIn | OIDC：`GET /v2/userinfo`（`openid profile email`）；旧版成员资料：`GET /v2/me`（`r_liteprofile`/实际批准的 `r_basicprofile`） | `sub`、`urn:li:organization:{id}` |
| YouTube | `channels.list?mine=true` | channel ID、title |
| Facebook | `/me/accounts` | Page ID、name、tasks |
| X | `GET /2/users/me` | user ID、username |
| Reddit | `GET /api/v1/me` | username、id |

绑定前必须展示：平台、账号名、账号类型、将获得的权限。用户确认后才保存。账号资产状态见 `13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md`；未核实归属/管理员的账号不得绑定或发布。

## 3. Token 存储

### 3.1 加密

- 算法：AES-256-GCM。
- 每条记录独立 IV（12 bytes），随机生成。
- AAD：`platform | provider_account_id | key_version`。
- 密文格式：`version(1) | iv(12) | tag(16) | ciphertext`。
- KEK 来自环境变量/KMS，不写入仓库；支持多版本，旧版本只解密不加密。
- Token 明文只在内存中存在，生命周期最短；不写临时文件。

### 3.2 数据

```text
social_accounts: account_id, platform, provider_account_id, account_name, account_type,
                 status, granted_scopes, token_expires_at, last_refresh_at, authorization_owner
oauth_tokens:    account_id, access_token_enc, refresh_token_enc, expires_at, scopes, key_version
```

### 3.3 刷新

| 平台 | Access Token | Refresh | 策略 |
|---|---|---|---|
| LinkedIn | 60 天 | 仅部分伙伴支持程序化刷新 | 到期前 7 天提醒；不支持刷新则要求重新授权 |
| YouTube | 1 小时 | 有 | 到期前 10 分钟刷新；失败重试 3 次 |
| Facebook | 短期/长期用户 Token | 长期用户 Token | 换 Page Token；按平台有效期刷新 |
| X | 2 小时 | `offline.access` 提供 | 到期前 10 分钟刷新；并发刷新加锁 |
| Reddit | 平台指定 | 平台指定 | 待核验 |

刷新实现要求：

1. 单账号刷新使用数据库行锁，避免并发刷新导致 refresh token 失效。
2. 刷新成功：更新密文、`expires_at`、`last_refresh_at`，保留旧 `key_version` 记录用于审计。
3. 刷新失败 `invalid_grant`：账号置 `reauth_required`，停止该账号的发布任务，发告警。
4. 刷新失败 5xx/网络：指数退避，不立即标记账号失效。

## 4. Scope 最小化

| 平台 | 发布 | 分析 | 身份 | 禁止申请 |
|---|---|---|---|---|
| LinkedIn | `w_organization_social` | `rw_organization_admin`、`r_organization_social` | OIDC：`openid profile email`；旧版：`r_liteprofile`（`r_basicprofile` 仅在已获批时） | `r_member_social`（closed） |
| YouTube | `youtube.upload` | `yt-analytics.readonly` | `youtube.readonly` | 无必要的宽 Scope |
| Facebook | `pages_manage_posts`、`pages_read_engagement` | `read_insights` | `pages_show_list` | 用户好友/私信权限 |
| X | `tweet.write`、`media.write` | `tweet.read`、`users.read` | `users.read` | DM 权限 |
| Reddit | `submit` | `read` | `identity` | `privatemessages` 等 |

规则：

- 发布与 Analytics 使用**独立授权记录**（同一平台账号可有两条 Token，Scope 不同）。
- 申请新 Scope 必须重新走授权，不得静默扩大。
- Scope 变化必须使旧审批失效（因为能力边界变了）。
- LinkedIn 身份读取不得混用：OIDC UserInfo 用 `openid profile email`；旧版成员资料 API 才用 `r_liteprofile`/`r_basicprofile`，且以后台实际批准为准。

## 5. 日志与错误脱敏

禁止记录：

- `access_token`、`refresh_token`、`client_secret`、`code_verifier`、`Authorization` 头。
- 完整回调 URL（含 code）。
- 平台返回的 `WWW-Authenticate` 原文（可能含 token 片段）。

允许记录：

- `account_id`、平台、HTTP 状态、平台错误码、请求 ID。
- Token 的**指纹**（如 SHA-256 前 8 位）用于关联，不用于还原。

错误页不得展示平台原始响应；只展示“授权失败，请重新授权”与本地错误 ID。

## 6. 撤销与轮换

### 6.1 用户撤销

```text
账号中心 → 撤销授权
  ├─ 调平台 revoke 端点（支持时）
  ├─ 删除 oauth_tokens 密文
  ├─ social_accounts.status = revoked
  ├─ 取消该账号所有 queued jobs
  ├─ 写 audit_events
  └─ 通知管理员
```

### 6.2 密钥轮换

- 新增 KEK 版本；新写入用新版本；旧密文按需重加密。
- 轮换脚本必须支持 dry-run、分批、校验。
- 轮换失败不影响旧版本解密。

### 6.3 Client Secret 轮换（内部系统）

见 `12_OAUTH2_AUTHORIZATION_ARCHITECTURE.md` §7。

## 7. 环境隔离

| 环境 | 平台应用 | 账号 | Token 库 | 说明 |
|---|---|---|---|---|
| dev | 开发者应用（测试模式） | 测试账号 | 本地加密库 | 不连生产 Page |
| staging | 独立应用或测试模式 | 测试 Page/频道 | 独立库 | 用于验收 |
| prod | 生产应用 | Morrishk 官方账号 | 生产库 | 需审批后启用 |

禁止：

- dev/staging 使用生产 Token。
- 把生产 Token 复制到测试库。
- 在 CI 中执行真实发布。

## 8. 安全验收清单

- [ ] Token 密文入库，数据库转储中无明文。
- [ ] 日志、错误页、审计中无 Token/Secret/Code。
- [ ] `state` 一次性且与会话绑定。
- [ ] redirect URI 白名单精确匹配。
- [ ] 刷新并发有锁。
- [ ] `invalid_grant` 正确置 `reauth_required`。
- [ ] 撤销后 queued job 不再提交。
- [ ] Scope 与功能矩阵一致。
- [ ] dev/staging/prod 隔离。
- [ ] 密钥轮换演练通过。
- [ ] 依赖许可证与来源审查通过。
