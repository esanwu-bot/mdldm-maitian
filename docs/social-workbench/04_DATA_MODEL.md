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

CREATE TABLE IF NOT EXISTS internal_clients (
  client_id          TEXT PRIMARY KEY,      -- morrishk-social-workbench-dsbit 等
  client_secret_hash TEXT NOT NULL,         -- Argon2id/bcrypt，不存明文
  audience           TEXT NOT NULL,         -- dsbit|morrishk
  scopes             TEXT NOT NULL,         -- 空格分隔
  status             TEXT NOT NULL DEFAULT 'active',
  created_at         TEXT NOT NULL,
  rotated_at         TEXT,
  revoked_at         TEXT
);
```

### 2.3 外部平台账号与 Token

```sql
CREATE TABLE IF NOT EXISTS social_accounts (
  account_id          TEXT PRIMARY KEY,     -- 内部 UUID
  platform            TEXT NOT NULL,        -- linkedin|youtube|facebook|x|reddit
  provider_account_id TEXT NOT NULL,        -- 平台侧 ID / URN
  account_name        TEXT NOT NULL,
  account_type        TEXT NOT NULL,        -- organization|member|page|channel
  status              TEXT NOT NULL DEFAULT 'active', -- active|reauth_required|revoked|error
  granted_scopes      TEXT NOT NULL,
  token_expires_at    TEXT,
  last_refresh_at     TEXT,
  authorization_owner TEXT,                 -- user_id
  created_at          TEXT NOT NULL,
  updated_at          TEXT NOT NULL,
  UNIQUE (platform, provider_account_id)
);

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
  status           TEXT NOT NULL DEFAULT 'draft', -- draft|in_review|approved|released|archived
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
  review_status   TEXT NOT NULL,           -- DRAFT|PIPELINE_PASS|CONTENT_APPROVED|RELEASE_APPROVED
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
  stage           TEXT NOT NULL,           -- CONTENT_APPROVED|RELEASE_APPROVED
  decision        TEXT NOT NULL,           -- approved|rejected|revoked
  decided_by      TEXT NOT NULL,
  decided_at      TEXT NOT NULL,
  reason          TEXT,
  scope           TEXT,                    -- public_marketing
  FOREIGN KEY (content_id, version) REFERENCES content_versions(content_id, version)
);

CREATE INDEX IF NOT EXISTS idx_approvals_content ON content_approvals(content_id, version);
```

硬约束（由应用层与数据库触发器共同保证）：

1. 无 `RELEASE_APPROVED` 的版本不得创建 `publish_jobs`。
2. `content_versions.payload_hash` 变化必须产生新版本，不允许原地改。
3. `content_approvals` 只追加；撤销用 `decision='revoked'`，不删除记录。

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
| Analytics 原始 | ≥ 24 个月 | LinkedIn 12 个月、Meta 2 年；保留原始以便重算 |
| Token | 仅当前有效 | 撤销/过期后删除密文，保留审计事件 |
| 站点归因事件 | ≥ 24 个月 | 询盘 ID 脱敏 |
| 操作审计 | ≥ 36 个月 | 授权、审批、发布、删除 |

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