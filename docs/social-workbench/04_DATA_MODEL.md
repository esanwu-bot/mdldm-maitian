# 数据模型设计

目标：在不破坏现有 `data/hub.db` 的前提下，新增账号、内容、审批、发布任务、原始分析、归因与审计模型。  
原则：**新增表、双写过渡、不原地改主键、不删除旧表**。

## 1. 现有模型与目标模型映射

| 现有表 | 目标表 | 处理 |
|---|---|---|
| `episodes` | `content_items` + `campaigns` | 保留旧表；新内容进入 `content_items` |
| `tasks` | `publish_jobs` + `publish_attempts` | 旧表只读；新任务写新表 |
| `metrics` | `analytics_raw` + `analytics_daily` | 保留旧表供旧看板；新采集写新表 |
| `samples` | `analytics_raw`（按采集时间） | 逐步停止写旧表 |
| `snapshots` | `content_versions` | 导入时生成版本 |
| `site_campaigns` | `attribution_daily` | 扩展 `utm_content`、询盘字段 |
| `site_daily` | `site_daily` | 保留，增加询盘统计视图 |
| `site_channels` | `attribution_daily` | 保留旧表 |
| `logs` | `audit_events` | 保留旧表；新审计写新表 |
| `meta` | `meta` | 保留 |
| `board_records` | `board_records` | 保留 |

## 2. 新表

### 2.1 迁移与版本

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
  version     INTEGER PRIMARY KEY,
  name        TEXT NOT NULL,
  applied_at  TEXT NOT NULL,
  checksum    TEXT NOT NULL
);
```

- 启动时检查 `PRAGMA user_version` 与 `schema_migrations`。
- 每个迁移在事务中执行；失败回滚并停止服务。
- 迁移前自动备份到 `data/backups/hub-YYYYMMDD-HHMMSS.db`。

### 2.2 用户、角色与内部客户端

```sql
CREATE TABLE IF NOT EXISTS user_profiles (
  user_id        TEXT PRIMARY KEY,          -- OIDC sub
  issuer         TEXT NOT NULL,
  email          TEXT,
  display_name   TEXT,
  status         TEXT NOT NULL DEFAULT 'active', -- active|disabled
  created_at     TEXT NOT NULL,
  last_login_at  TEXT
);

CREATE TABLE IF NOT EXISTS user_roles (
  user_id     TEXT NOT NULL,
  role        TEXT NOT NULL,                -- admin|editor|reviewer|analyst|viewer
  granted_by  TEXT NOT NULL,
  granted_at  TEXT NOT NULL,
  PRIMARY KEY (user_id, role)
);

CREATE TABLE IF NOT EXISTS internal_client_config (
  client_id       TEXT PRIMARY KEY,        -- morrishk-social-workbench-dsbit 等
  resource        TEXT NOT NULL,           -- RFC 8707 resource，例如 https://dsbit.example/api
  token_endpoint  TEXT NOT NULL,           -- 实际 Token Endpoint
  scopes          TEXT NOT NULL,           -- 空格分隔
  secret_ref      TEXT NOT NULL,           -- env:DSBIT_CLIENT_SECRET / secretmanager://...
  status          TEXT NOT NULL DEFAULT 'active',
  created_at      TEXT NOT NULL,
  rotated_at      TEXT,
  revoked_at      TEXT
);
```

> `secret_ref` 只保存“去哪里取密钥”，不保存 Secret 或 Secret 哈希。客户端凭证的哈希由 Token 签发方（Authorization Server）保存；工作台作为 OAuth Client，出站时从 Secret Manager/环境变量取明文并只在内存使用。

### 2.3 外部平台账号与 Token

```sql
CREATE TABLE IF NOT EXISTS social_accounts (
  account_id          TEXT PRIMARY KEY,     -- 内部 UUID
  platform            TEXT NOT NULL,        -- 可扩展：linkedin|youtube|facebook|x|reddit|instagram|tiktok|vimeo|...
  provider_account_id TEXT NOT NULL,        -- 平台侧 ID / URN
  account_name        TEXT NOT NULL,
  account_type        TEXT NOT NULL,        -- 可扩展：organization|member|page|channel|business|creator|brand|...
  capabilities_json   TEXT,                 -- 该账号实际可用的发布/媒体/分析能力
  status              TEXT NOT NULL DEFAULT 'active', -- active|reauth_required|revoked|error
  granted_scopes      TEXT NOT NULL,
  token_expires_at    TEXT,
  last_refresh_at     TEXT,
  authorization_owner TEXT,                 -- user_id
  created_at          TEXT NOT NULL,
  updated_at          TEXT NOT NULL,
  UNIQUE (platform, provider_account_id)
);

-- 平台范围注册表：区分当前实施、后续评估、范围外；不把 5 个平台写成封闭 CHECK
CREATE TABLE IF NOT EXISTS platform_registry (
  platform             TEXT PRIMARY KEY,
  display_name         TEXT NOT NULL,
  implementation_scope TEXT NOT NULL,       -- current|evaluation|out_of_scope
  implementation_phase TEXT,                -- P1|P2|P4|...
  auth_type            TEXT,
  capabilities_json    TEXT,
  status               TEXT NOT NULL DEFAULT 'active',
  updated_at           TEXT NOT NULL
);
```

平台范围（本表为配置数据，不是数据库 CHECK 枚举）：

| platform | implementation_scope | 当前阶段 | 说明 |
|---|---|---|---|
| linkedin | `current` | P1 | 端到端 PoC |
| youtube | `current` | P2 | 视频上传/Analytics |
| facebook | `current` | P2 | Page 发布/Insights |
| x | `current` | P4 | 成本门禁后实施 |
| reddit | `current` | P4 | 商业协议后实施；未签协议只做草稿 |
| instagram | `evaluation` | 后续 | 与 Meta 账号/权限一起评估 |
| tiktok | `evaluation` | 后续 | Content Posting 审核后再评估 |
| vimeo | `evaluation` | 后续 | 先确认是否存在实际需求 |
| 其他平台 | 待登记 | 后续 | 通过 `platform_registry` 新增，不改表结构 |

```sql
CREATE TABLE IF NOT EXISTS oauth_tokens (
  token_id        TEXT PRIMARY KEY,
  account_id      TEXT NOT NULL,
  access_token_enc  BLOB NOT NULL,          -- AES-256-GCM
  refresh_token_enc BLOB,
  token_type      TEXT NOT NULL DEFAULT 'Bearer',
  expires_at      TEXT,
  scopes          TEXT NOT NULL,
  key_version     INTEGER NOT NULL,
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,
  FOREIGN KEY (account_id) REFERENCES social_accounts(account_id)
);

CREATE INDEX IF NOT EXISTS idx_oauth_tokens_account ON oauth_tokens(account_id);
```

> `social_accounts` 的真实账号来源与核实状态见 `13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md`；未核实归属/管理员的账号不得写入 `publish_jobs`。

约束：

- `access_token_enc`/`refresh_token_enc` 使用 AES-256-GCM；AAD 绑定 `account_id + platform + key_version`。
- 日志、错误信息、审计事件不得出现明文 Token。
- 一个账号保留当前 Token + 最多 1 个待轮换 Token；旧 Token 在刷新成功后删除。

### 2.4 内容与版本

```sql
CREATE TABLE IF NOT EXISTS campaigns (
  campaign_id   TEXT PRIMARY KEY,
  name          TEXT NOT NULL,
  objective     TEXT,
  starts_at     TEXT,
  ends_at       TEXT,
  status        TEXT NOT NULL DEFAULT 'active',
  created_at    TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS content_items (
  content_id       TEXT PRIMARY KEY,
  content_type     TEXT NOT NULL,
  product_mpn      TEXT NOT NULL,
  product_brand    TEXT NOT NULL,
  campaign_id      TEXT,
  content_language TEXT NOT NULL,
  current_version  INTEGER NOT NULL DEFAULT 1,
  status           TEXT NOT NULL DEFAULT 'draft', -- draft|in_review|social_approved|released|archived
  created_by       TEXT NOT NULL,
  created_at       TEXT NOT NULL,
  updated_at       TEXT NOT NULL,
  FOREIGN KEY (campaign_id) REFERENCES campaigns(campaign_id)
);

CREATE INDEX IF NOT EXISTS idx_content_product ON content_items(product_mpn, product_brand);
CREATE INDEX IF NOT EXISTS idx_content_status ON content_items(status);

CREATE TABLE IF NOT EXISTS content_versions (
  content_id      TEXT NOT NULL,
  version         INTEGER NOT NULL,
  payload_json    TEXT NOT NULL,           -- 见 02 §4
  payload_hash    TEXT NOT NULL,           -- sha256(canonical json)
  source_version  TEXT,
  review_status   TEXT NOT NULL,           -- DRAFT|PIPELINE_PASS|SOCIAL_CONTENT_APPROVED
  created_by      TEXT NOT NULL,
  created_at      TEXT NOT NULL,
  PRIMARY KEY (content_id, version),
  FOREIGN KEY (content_id) REFERENCES content_items(content_id)
);

CREATE TABLE IF NOT EXISTS content_evidence (
  content_id   TEXT NOT NULL,
  version      INTEGER NOT NULL,
  document_id  TEXT NOT NULL,
  page         INTEGER,
  table_name   TEXT,
  quote        TEXT,
  source_hash  TEXT,
  PRIMARY KEY (content_id, version, document_id, page, table_name, quote)
);

CREATE TABLE IF NOT EXISTS content_approvals (
  approval_id     TEXT PRIMARY KEY,
  content_id      TEXT NOT NULL,
  version         INTEGER NOT NULL,
  stage           TEXT NOT NULL,           -- SOCIAL_CONTENT_APPROVED
  decision        TEXT NOT NULL,           -- approved|rejected|revoked
  decided_by      TEXT NOT NULL,
  decided_at      TEXT NOT NULL,
  reason          TEXT,
  scope           TEXT,                    -- public_marketing
  FOREIGN KEY (content_id, version) REFERENCES content_versions(content_id, version)
);

CREATE INDEX IF NOT EXISTS idx_approvals_content ON content_approvals(content_id, version);

-- 发布前门禁快照：三个维度分别记录，禁止用一个状态代表全部
CREATE TABLE IF NOT EXISTS content_gate_checks (
  content_id            TEXT NOT NULL,
  version               INTEGER NOT NULL,
  dsbit_release_status  TEXT NOT NULL,     -- RELEASE_APPROVED | NOT_RELEASED | REVOKED
  product_publish_status TEXT NOT NULL,    -- PUBLIC_ELIGIBLE | NOT_ELIGIBLE | REVOKED
  social_content_status TEXT NOT NULL,     -- SOCIAL_CONTENT_APPROVED | NOT_APPROVED | REVOKED
  source_version        TEXT,
  checked_at            TEXT NOT NULL,
  checked_by            TEXT NOT NULL,
  PRIMARY KEY (content_id, version)
);
```

硬约束（由应用层与数据库触发器共同保证）：

1. 创建 `publish_jobs` 前必须同时满足：
   - `dsbit_release_status = 'RELEASE_APPROVED'`
   - `product_publish_status = 'PUBLIC_ELIGIBLE'`
   - `social_content_status = 'SOCIAL_CONTENT_APPROVED'`
2. `RELEASE_APPROVED` 只属于 Ds Bit 技术事实，不能替代商品状态或社媒内容审批。
3. `content_versions.payload_hash` 变化必须产生新版本，不允许原地改。
4. `content_approvals` 与 `content_gate_checks` 只追加；撤销用 `decision='revoked'` 或状态置 `REVOKED`，不删除记录。
5. 商品 `PUBLIC_ELIGIBLE` 被撤销时，所有未发布的 `queued`/`submitting` job 必须取消或转 `unknown` 并对账。

### 2.5 媒体资产

```sql
CREATE TABLE IF NOT EXISTS media_assets (
  asset_id     TEXT PRIMARY KEY,
  content_id   TEXT,
  version      INTEGER,
  media_type   TEXT NOT NULL,              -- image|video
  local_path   TEXT,
  sha256       TEXT NOT NULL,
  bytes        INTEGER,
  width        INTEGER,
  height       INTEGER,
  duration_ms  INTEGER,
  status       TEXT NOT NULL DEFAULT 'ready', -- ready|missing|invalid
  created_at   TEXT NOT NULL
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_media_sha ON media_assets(sha256);
```

### 2.6 发布任务

```sql
CREATE TABLE IF NOT EXISTS publish_jobs (
  job_id           TEXT PRIMARY KEY,
  content_id       TEXT NOT NULL,
  content_version  INTEGER NOT NULL,
  platform         TEXT NOT NULL,
  account_id       TEXT NOT NULL,
  payload_hash     TEXT NOT NULL,
  idempotency_key  TEXT NOT NULL,
  state            TEXT NOT NULL,          -- draft|approved|queued|submitting|processing|published|failed|unknown|cancelled
  scheduled_at     TEXT,
  priority         INTEGER NOT NULL DEFAULT 100,
  attempt_count    INTEGER NOT NULL DEFAULT 0,
  max_attempts     INTEGER NOT NULL DEFAULT 3,
  lease_owner      TEXT,
  lease_expires_at TEXT,
  last_error       TEXT,
  created_at       TEXT NOT NULL,
  updated_at       TEXT NOT NULL,
  UNIQUE (content_id, content_version, platform, account_id, payload_hash)
);

CREATE INDEX IF NOT EXISTS idx_jobs_state ON publish_jobs(state, scheduled_at);
CREATE INDEX IF NOT EXISTS idx_jobs_account ON publish_jobs(account_id, platform);

CREATE TABLE IF NOT EXISTS publish_attempts (
  attempt_id          TEXT PRIMARY KEY,
  job_id              TEXT NOT NULL,
  attempt_no          INTEGER NOT NULL,
  started_at          TEXT NOT NULL,
  finished_at         TEXT,
  outcome             TEXT NOT NULL,       -- success|failed|timeout|unknown
  http_status         INTEGER,
  provider_request_id TEXT,
  provider_post_id    TEXT,
  post_url            TEXT,
  error_code          TEXT,
  error_message       TEXT,               -- 脱敏，≤2000 字符
  raw_response_json   TEXT,               -- 脱敏，单独表可按需归档
  retryable           INTEGER NOT NULL DEFAULT 0,
  FOREIGN KEY (job_id) REFERENCES publish_jobs(job_id)
);

CREATE INDEX IF NOT EXISTS idx_attempts_job ON publish_attempts(job_id, attempt_no);

CREATE TABLE IF NOT EXISTS provider_requests (
  request_id      TEXT PRIMARY KEY,        -- 本地请求 ID
  job_id          TEXT NOT NULL,
  platform        TEXT NOT NULL,
  endpoint        TEXT NOT NULL,
  method          TEXT NOT NULL,
  request_hash    TEXT NOT NULL,
  sent_at         TEXT NOT NULL,
  response_status INTEGER,
  provider_request_id TEXT,
  FOREIGN KEY (job_id) REFERENCES publish_jobs(job_id)
);
```

### 2.7 Analytics

```sql
CREATE TABLE IF NOT EXISTS metric_registry (
  platform        TEXT NOT NULL,
  metric_name     TEXT NOT NULL,           -- 标准名，如 impressions
  native_name     TEXT NOT NULL,           -- 平台原字段
  unit            TEXT NOT NULL,           -- count|ratio|seconds
  definition      TEXT NOT NULL,
  availability    TEXT NOT NULL,           -- available|unavailable|deprecated
  effective_from  TEXT,
  effective_to    TEXT,
  api_version     TEXT,
  PRIMARY KEY (platform, metric_name, native_name, api_version)
);

CREATE TABLE IF NOT EXISTS analytics_raw (
  raw_id          TEXT PRIMARY KEY,
  platform        TEXT NOT NULL,
  account_id      TEXT NOT NULL,
  provider_post_id TEXT,
  metric_date     TEXT,
  endpoint        TEXT NOT NULL,
  request_params  TEXT,
  payload_json    TEXT NOT NULL,
  payload_hash    TEXT NOT NULL,
  collected_at    TEXT NOT NULL,
  api_version     TEXT,
  UNIQUE (platform, account_id, provider_post_id, metric_date, endpoint, payload_hash)
);

CREATE INDEX IF NOT EXISTS idx_raw_lookup ON analytics_raw(platform, account_id, metric_date);

CREATE TABLE IF NOT EXISTS analytics_daily (
  platform         TEXT NOT NULL,
  account_id       TEXT NOT NULL,
  provider_post_id TEXT NOT NULL,
  content_id       TEXT,
  metric_date      TEXT NOT NULL,
  metric_name      TEXT NOT NULL,
  value_num        REAL,
  value_text       TEXT,
  availability     TEXT NOT NULL,          -- available|unavailable
  unavailable_reason TEXT,
  unit             TEXT,
  source_raw_id    TEXT NOT NULL,
  normalized_at    TEXT NOT NULL,
  PRIMARY KEY (platform, account_id, provider_post_id, metric_date, metric_name)
);

CREATE INDEX IF NOT EXISTS idx_daily_content ON analytics_daily(content_id, metric_date);
```

规则：

- `availability='unavailable'` 时 `value_num` 必须为 `NULL`，禁止写 0。
- 每个 `value_num` 必须能追溯到 `source_raw_id`。
- 不同平台同名指标不得自动相加；聚合必须在明确的“标准指标”层完成，并在 UI 标注口径。

### 2.8 归因

```sql
CREATE TABLE IF NOT EXISTS attribution_events (
  event_id       TEXT PRIMARY KEY,
  event_type     TEXT NOT NULL,            -- page_view|cta_click|inquiry_submit|inquiry_qualified
  occurred_at    TEXT NOT NULL,
  content_id     TEXT,
  campaign_id    TEXT,
  platform       TEXT,
  utm_source     TEXT,
  utm_medium     TEXT,
  utm_campaign   TEXT,
  utm_content    TEXT,
  product_mpn    TEXT,
  product_brand  TEXT,
  session_id     TEXT,
  inquiry_id     TEXT,
  qualified      INTEGER,
  metadata_json  TEXT
);

CREATE INDEX IF NOT EXISTS idx_attr_content ON attribution_events(content_id, occurred_at);
CREATE INDEX IF NOT EXISTS idx_attr_inquiry ON attribution_events(inquiry_id);

CREATE TABLE IF NOT EXISTS attribution_daily (
  day             TEXT NOT NULL,
  platform        TEXT NOT NULL,
  campaign_id     TEXT,
  content_id      TEXT,
  visits          INTEGER NOT NULL DEFAULT 0,
  product_views   INTEGER NOT NULL DEFAULT 0,
  cta_clicks      INTEGER NOT NULL DEFAULT 0,
  inquiry_submits INTEGER NOT NULL DEFAULT 0,
  qualified_inquiries INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (day, platform, campaign_id, content_id)
);
```

### 2.9 审计

```sql
CREATE TABLE IF NOT EXISTS audit_events (
  audit_id     TEXT PRIMARY KEY,
  occurred_at  TEXT NOT NULL,
  actor_type   TEXT NOT NULL,              -- user|system|external_client
  actor_id     TEXT,
  action       TEXT NOT NULL,              -- oauth.connect|content.approve|publish.submit|token.revoke|config.change
  object_type  TEXT,
  object_id    TEXT,
  ip           TEXT,
  user_agent   TEXT,
  result       TEXT NOT NULL,              -- success|failure
  detail_json  TEXT                        -- 脱敏
);

CREATE INDEX IF NOT EXISTS idx_audit_time ON audit_events(occurred_at);
CREATE INDEX IF NOT EXISTS idx_audit_object ON audit_events(object_type, object_id);
```

## 3. 迁移计划

### 3.1 阶段 M0（P1 之前）

1. 备份 `data/hub.db`。
2. 新增上述表；不改旧表。
3. 写入 `schema_migrations` 与 `user_version=1`。
4. 只对 LinkedIn 新链路启用新表；旧 Chrome 链路继续写旧表。

### 3.2 阶段 M1（P1 期间）

1. `episodes` → `content_items` 的只读映射视图，用于旧看板。
2. LinkedIn 发布写 `publish_jobs`/`publish_attempts`；旧 `tasks` 不再写 LinkedIn。
3. Analytics 采集写 `analytics_raw`/`analytics_daily`；旧 `metrics` 保留。

### 3.3 阶段 M2（P2 后）

1. 旧看板查询切换到新表；旧表改为只读归档。
2. `samples` 停止写入。
3. 观察 30 天后，再评估是否归档旧表；**不删除**。

## 4. 数据保留与合规

| 数据 | 保留 | 说明 |
|---|---|---|
| 内容与审批 | 永久 | 审计需要 |
| 发布任务/尝试 | ≥ 24 个月 | 与平台 Analytics 窗口对齐 |
| Analytics 原始 | 平台级策略，见下表 | 不得用一个统一期限覆盖所有平台 |
| Token | 仅当前有效 | 撤销/过期后删除密文，保留审计事件 |
| 站点归因事件 | ≥ 24 个月 | 询盘 ID 脱敏 |
| 操作审计 | ≥ 36 个月 | 授权、审批、发布、删除 |

### 4.1 平台级分析数据保留

| 平台 | 默认保留 | 依据/限制 |
|---|---|---|
| LinkedIn | 24 个月 | API 仅提供滚动 12 个月；保留原始用于重算 |
| YouTube | 24 个月 | 受 YouTube API Services Terms 约束 |
| Facebook | 24 个月 | API 仅提供 2 年历史；指标退役后保留原始 |
| X | 24 个月 | 受 X Developer Agreement 约束 |
| Reddit | **未获批前为 0** | Data API Terms（2026-07-20 修订）；商业用途需单独协议；获批后按协议与获批用途设定，不默认套用 24 个月 |

规则：

1. 平台条款与本地策略冲突时，取更严格者。
2. Reddit 未签署商业 API 协议前，不调用 API、不采集、不持久化 Reddit 数据；只允许人工草稿。
3. 平台调整条款/保留要求时，必须复审并更新本表。
4. 删除请求必须同时覆盖 `analytics_raw`、`analytics_daily` 与导出副本。

PII 最小化：工作台只存 OIDC `sub`、邮箱、显示名；不存访客姓名/电话；询盘只存 `inquiry_id` 与业务状态，明细留在 Morrishk CRM。

## 5. 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| 同步 SQLite 阻塞 | Web/Worker 卡顿 | 原始 JSON 批量异步写；必要时拆独立采集进程 |
| 迁移无回滚 | 数据损坏 | 迁移前备份 + 事务 + 校验和 |
| 旧表主键冲突 | 多账号无法表达 | 新表独立，不改旧主键 |
| Token 密文与密钥丢失 | 无法发布 | KEK 版本化 + 备份 + 轮换演练 |
| 原始响应过大 | 库膨胀 | 压缩/归档；单条上限；定期 `VACUUM` |
| 指标口径漂移 | 报表失真 | `metric_registry` 版本化；平台 API 版本入键 |
