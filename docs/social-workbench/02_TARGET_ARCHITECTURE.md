# 目标架构设计

## 1. 设计原则

1. **官方 API 优先**：发布、状态、Analytics 只走平台官方 API/SDK；不引入聚合发布平台。
2. **内容与发布分离**：内容先审核、再排期、再提交；提交动作不可由生成动作直接触发。
3. **事实可追溯**：每条对外技术表述都能回溯到 `product_identity`、`source_reference`、`source_version`、`evidence_references`。
4. **状态可恢复**：任何超时都进入 `unknown` 并先对账，不盲目重试。
5. **能力显式声明**：平台不支持的能力返回 `UNSUPPORTED`，不伪造成功。
6. **原始数据优先**：Analytics 先保存平台原始响应，再标准化；`unavailable ≠ 0`。
7. **最小权限**：内部系统、用户、外部平台三层授权彼此隔离。
8. **渐进替换**：Chrome 路径保留为回退，但新核心不依赖它；每个平台完成 `REAL_ACCOUNT_PASS` 后再切换。

## 2. 总体架构

```text
┌────────────────────────────── Morrishk 侧 ──────────────────────────────┐
│  Morrishk 商品主数据 API            Ds Bit 技术数据 API                  │
│  - MPN / Brand / 标题 / 分类         - Datasheet 解析结果                 │
│  - 详情 URL / 图片 / 卖点            - 结构化参数 / 页码证据              │
│  - RELEASE_APPROVED 状态             - 技术说明 / 应用场景                │
└───────────────┬──────────────────────────────┬──────────────────────────┘
                │ OAuth2 Client Credentials    │ 独立 Client Credentials
                │ scope: product.read          │ scope: technical.read
                ▼                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      内容源适配层 source/                                │
│  product-source.adapter  dsbit-source.adapter  release-gate             │
│  - 只读；不修改上游                                                      │
│  - 只接收 RELEASE_APPROVED 的商品事实                                    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    内容中控 content/                                     │
│  content.service  content.repository  approval.service                  │
│  - 平台无关内容模型（见 §4）                                             │
│  - 内容版本、审核记录、发布门禁                                          │
│  - 生成平台变体，但不提交                                                │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                  发布编排 publish/                                       │
│  scheduler  state-machine  dedup  job-runner  reconciler                 │
│  - 计划、排队、幂等、重试、对账、恢复                                    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                  平台适配层 providers/                                   │
│  provider.interface                                                     │
│  linkedin.adapter  youtube.adapter  facebook.adapter                    │
│  x.adapter  reddit.adapter                                              │
│  - 只调用官方 API                                                        │
│  - 显式 capability matrix                                                │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│               外部平台官方 API（LinkedIn / Google / Meta / X / Reddit）  │
└─────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────── 横切层 ───────────────────────────────────┐
│ auth/        OAuth 管理器、Token 加密存储、轮换、吊销                     │
│ analytics/   原始采集、标准化、映射、归因                                 │
│ web/         工作台 UI、RBAC、审批、审计                                  │
│ notify/      飞书通知                                                    │
│ storage/     SQLite + 迁移 + 备份                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. 模块划分

```text
src/
  providers/
    provider.interface.mjs
    linkedin.adapter.mjs
    youtube.adapter.mjs
    facebook.adapter.mjs
    x.adapter.mjs
    reddit.adapter.mjs
  auth/
    oauth.manager.mjs
    token.store.mjs
    internal-client.mjs
    user-session.mjs
  content/
    content.service.mjs
    content.repository.mjs
    approval.service.mjs
    release-gate.mjs
  source/
    product-source.adapter.mjs
    dsbit-source.adapter.mjs
  publish/
    publish.scheduler.mjs
    publish.state-machine.mjs
    publish.dedup.mjs
    publish.job-runner.mjs
    publish.reconciler.mjs
  analytics/
    analytics.collector.mjs
    analytics.normalizer.mjs
    analytics.attribution.mjs
    metrics.registry.mjs
  storage/
    db.mjs
    migrations/
  web/
    server.mjs
    routes/
    rbac.mjs
  notify/
    feishu.mjs
```

目录可按实际实现调整，但**层次边界不得打破**：

- `providers/*` 不得直接读写 SQLite。
- `publish/*` 不得包含平台特定字段。
- `content/*` 不得调用平台 API。
- `analytics/*` 不得改写发布状态。
- `web/*` 不得直接拼平台请求。

## 4. 平台无关内容模型

`publish.json` 保留为导入/导出格式。运行时以 `content_items` 为准。

```json
{
  "content_id": "cnt_20261120_mpn_abc123",
  "content_type": "datasheet_interpretation",
  "product_identity": {
    "mpn": "ABC123",
    "brand": "Example",
    "product_title": "Example ABC123 ...",
    "category": "MOSFET",
    "product_url": "https://www.morrishk.net/...",
    "image_url": "https://...",
    "application_areas": ["industrial control"],
    "selling_points": ["low Rds(on)"]
  },
  "source_reference": {
    "product_source": "morrishk",
    "technical_source": "dsbit",
    "document_id": "doc_...",
    "datasheet_revision": "Rev.C"
  },
  "source_version": "dsbit:2026-11-18T08:00:00Z",
  "content_language": "en",
  "campaign_id": "cmp_2026_q4_sic",
  "title": "How to read the ABC123 switching losses",
  "summary": "...",
  "core_message": "...",
  "technical_facts": [
    {
      "name": "Rds(on)",
      "value": "12 mΩ",
      "condition": "Vgs=10V, Id=20A",
      "evidence": { "page": 4, "table": "Electrical Characteristics" }
    }
  ],
  "evidence_references": [
    { "document_id": "doc_...", "page": 4, "quote": "Rds(on) max 12 mΩ" }
  ],
  "target_platforms": ["linkedin", "youtube"],
  "platform_specific_content": {
    "linkedin": {
      "text": "...",
      "hashtags": ["#Semiconductor", "#MOSFET"],
      "cta": "Review the product page",
      "media": ["asset_..."],
      "target_url": "https://www.morrishk.net/...?utm_source=linkedin&utm_medium=social&utm_campaign=cmp_2026_q4_sic&utm_content=cnt_20261120_mpn_abc123"
    }
  },
  "media_assets": [
    { "asset_id": "asset_...", "type": "image", "path": "...", "sha256": "..." }
  ],
  "target_url": "https://www.morrishk.net/...",
  "utm_parameters": {
    "utm_source": "linkedin",
    "utm_medium": "social",
    "utm_campaign": "cmp_2026_q4_sic",
    "utm_content": "cnt_20261120_mpn_abc123"
  },
  "review_status": "RELEASE_APPROVED",
  "approval_record": {
    "approved_by": "user_...",
    "approved_at": "2026-11-20T10:00:00Z",
    "scope": "public_marketing",
    "content_version": 3
  },
  "scheduled_at": "2026-11-21T09:00:00Z"
}
```

### 4.1 内容类型

| 类型 | 说明 | 发布平台 |
|---|---|---|
| `product_intro` | 产品技术介绍 | LinkedIn / Facebook |
| `datasheet_interpretation` | Datasheet 技术解读 | LinkedIn / 视频 |
| `application_case` | 应用案例 | LinkedIn / Facebook / YouTube |
| `selection_guide` | 产品选型建议 | LinkedIn |
| `technical_article` | 技术知识文章 | LinkedIn / X |
| `product_comparison` | 产品对比 | LinkedIn |
| `video_content` | YouTube 视频内容 | YouTube |
| `reddit_draft` | Reddit 技术讨论草稿 | Reddit（半自动） |

### 4.2 审批门禁

```text
DRAFT
  └─ 内容生成/人工编辑
       └─ CONTENT_REVIEW
            ├─ PIPELINE_PASS：结构、字段、媒体校验通过（不等于可发布）
            └─ CONTENT_APPROVED：内容负责人确认文案与事实
                 └─ RELEASE_APPROVED：商品事实来自已批准版本，允许对外
                      └─ SCHEDULED / PUBLISHED
```

硬规则：

1. 只有 `RELEASE_APPROVED` 的 `content_version` 可以生成发布任务。
2. `PIPELINE_PASS` 不得写入 `approval_record`。
3. 技术事实缺失证据（页码/表格）时，只能生成内部草稿。
4. 修改 `technical_facts` 或 `target_url` 会使已有审批失效，必须重新审批。
5. Ds Bit 技术分类只用于技术检索，不得直接映射为 Morrishk 业务分类。

## 5. Provider 接口契约

```js
// provider.interface.mjs
export class Provider {
  static id = 'provider-id';
  static capabilities = {
    connectAccount: true,
    getAccountStatus: true,
    validateContent: true,
    publish: true,
    schedule: false,
    getPublishStatus: true,
    getPostUrl: true,
    deletePost: true,
    analytics: true,
    idempotencyKey: false,
    nativeScheduling: false
  };

  async connectAccount({ userId, redirectUri }) { throw new Error('UNSUPPORTED'); }
  async getAccountStatus({ accountId }) { throw new Error('UNSUPPORTED'); }
  async validateContent({ accountId, content }) { throw new Error('UNSUPPORTED'); }
  async publish({ accountId, content, idempotencyKey }) { throw new Error('UNSUPPORTED'); }
  async schedule({ accountId, content, scheduledAt }) { throw new Error('UNSUPPORTED'); }
  async getPublishStatus({ accountId, providerPostId }) { throw new Error('UNSUPPORTED'); }
  async getPostUrl({ accountId, providerPostId }) { throw new Error('UNSUPPORTED'); }
  async deletePost({ accountId, providerPostId }) { throw new Error('UNSUPPORTED'); }
}
```

每个 Adapter 必须提供 `capabilities`、`errorMap`、`rateLimitPolicy`、`tokenPolicy`、`mediaPolicy`。不支持的接口统一抛 `UNSUPPORTED`，由上层显示“不支持”，不得回退到浏览器自动化。

### 5.1 统一返回

```js
{
  ok: true,
  state: 'processing' | 'published' | 'failed' | 'unknown',
  providerRequestId: '...',
  providerPostId: '...',
  postUrl: '...',
  raw: { ... },          // 原始响应，脱敏后入库
  retryable: false
}
```

## 6. 发布编排

```text
scheduler
  - 读取 approved 且到期的 content
  - 展开为 publish_jobs（content × platform × account）
  - 计算 payload_hash，写入本地唯一键
job-runner
  - 领取 queued job（租约）
  - submitting：写 publish_attempts，调 provider.publish
  - 成功：processing 或 published
  - 明确失败：failed（按 errorMap 决定可重试）
  - 超时/网络中断：unknown，交给 reconciler
reconciler
  - unknown：按 provider 查询最近帖子/请求 ID
  - 找到：published/scheduled
  - 未找到且超过安全窗口：failed（允许人工重试）
  - 找到多条：unknown + 告警，禁止自动重试
```

`publish_jobs` 的本地唯一键：

```text
UNIQUE(content_id, content_version, platform, account_id, payload_hash)
```

平台不支持幂等键时：

1. 本地任务锁 + 租约，防止并发。
2. 提交前查询最近 N 条同内容记录。
3. 提交后按 `provider_post_id` 或内容指纹查询确认。
4. 超时进入 `unknown`，人工确认前不重试。

## 7. Analytics 与归因

```text
platform API
  └─ analytics.collector：按账号/帖子/日期拉取
       ├─ analytics_raw：原始 JSON + 请求参数 + 采集时间
       └─ analytics.normalizer：标准指标 + 映射版本
            └─ analytics_daily：平台/账号/帖子/日期/指标
                 └─ attribution：UTM + 站点事件 + 询盘
```

- 原始表只追加，不覆盖。
- 标准化表按 `(platform, account_id, provider_post_id, metric_date, metric_name)` 唯一。
- `metric_registry` 记录每个平台每个指标的定义、口径、单位、可用性与生效版本。
- `unavailable` 单独存 `availability_reason`，不得写 0。

## 8. 工作台界面

新增页面/能力：

| 页面 | 能力 |
|---|---|
| 内容库 | 内容列表、版本、来源、审批状态、平台变体 |
| 内容详情 | 事实/证据对照、审批记录、发布预览 |
| 审批队列 | 待审、驳回、批准、重新审批 |
| 账号中心 | 平台账号、授权状态、Scope、到期时间、撤销 |
| 发布计划 | 日历、队列、失败/未知、对账 |
| 发布详情 | 每次尝试、请求 ID、平台帖子、原始响应（脱敏） |
| Analytics | 平台原始/标准化指标、可用性说明 |
| 归因 | campaign/content → 访问 → 询价 → 有效询盘 |
| 审计 | 登录、授权、审批、发布、删除、配置变更 |

RBAC 角色：`admin`、`editor`、`reviewer`、`analyst`、`viewer`。发布与审批必须由不同角色完成（至少 reviewer 不能批准自己创建的内容，具体规则待项目确认）。

## 9. 部署形态

P0 建议单节点部署，避免过早引入分布式复杂度：

```text
[Web/API 进程] ── [Scheduler/Worker 进程] ── SQLite
        │                    │
        └──── 飞书通知 ───────┘
```

- 运行环境：Linux 容器优先；macOS 仅作为现有回退环境。
- 单实例 SQLite + WAL；任务租约用数据库行锁/事务实现。
- 备份：每日 `VACUUM INTO` 或文件快照；迁移前强制备份。
- 若未来需要多实例，再评估 PostgreSQL；P0 不引入。

## 10. 非目标

- 不做互动自动化（点赞、评论、私信、关注）。
- 不做批量灌水、伪造讨论、绕过平台风控。
- 不做商业聚合发布平台。
- 不在 P0 实现代码。
- 不承诺任何平台权限一定能获批。
- 不把 Mock/Sandbox 结果当作生产可用。