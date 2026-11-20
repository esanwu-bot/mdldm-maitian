# 发布任务状态机

## 1. 状态定义

| 状态 | 含义 | 是否终态 | 允许自动重试 |
|---|---|---|---|
| `draft` | 内容或任务尚未提交审批 | 否 | 否 |
| `approved` | 三层门禁通过且社媒内容已 `SOCIAL_CONTENT_APPROVED`，任务已生成但未排队 | 否 | 否 |
| `queued` | 已进入队列，等待 worker 领取 | 否 | 否 |
| `submitting` | 已发出请求或即将发出，结果未知 | 否 | 否（先对账） |
| `processing` | 平台已接受，异步处理中（视频转码等） | 否 | 否（查询） |
| `published` | 平台确认已发布/已定时 | **是** | 否 |
| `failed` | 平台明确拒绝或本地明确失败 | **是** | 仅人工/策略允许 |
| `unknown` | 超时、断网、响应不可判定 | **是（临时）** | **禁止自动重试** |
| `cancelled` | 人工取消 | **是** | 否 |

> `unknown` 不是“失败”。它表示“平台可能已经创建了内容”，必须对账后才能决定。

## 2. 状态转移

```text
draft ──approve──▶ approved ──enqueue──▶ queued
                                          │
                                          ▼
                                    submitting
                                      │  │  │
                     success(immediate)│  │  │timeout/ambiguous
                                      │  │  └──────────────▶ unknown
                                      │  │                        │
                                      │  └─accepted(async)──▶ processing
                                      │                           │
                                      │                     ┌─────┴─────┐
                                      ▼                     ▼           ▼
                                  published             published     failed
                                                          
queued/submitting/processing ──cancel──▶ cancelled
failed ──manual retry（新 attempt）──▶ queued
unknown ──reconcile found──▶ published
unknown ──reconcile not found + window passed──▶ failed
unknown ──human decision──▶ cancelled
```

禁止的转移：

- `published → queued`（同一 job 不得复活；重发必须新建 job）
- `unknown → submitting`（无对账不得重试）
- `failed → submitting`（必须回到 `queued` 并记录新 attempt）
- 任何状态 → `published`，除非有 `provider_post_id` 或平台查询确认

## 3. 不变量

1. 一个 `job_id` 只对应一个 `content_version` 与一个 `payload_hash`。
2. 创建 job 前必须同时满足：Ds Bit 技术事实 `RELEASE_APPROVED`、商品 `PUBLIC_ELIGIBLE`、社媒内容 `SOCIAL_CONTENT_APPROVED`；缺一不可。
3. 内容版本或 payload 变化必须新建 job；旧 job 不得修改。
4. `submitting` 状态必须持有未过期租约；租约过期视为崩溃。
5. 每次状态变化写 `audit_events`；每次请求写 `publish_attempts`。
6. `published` 必须有 `provider_post_id`（平台不提供时，必须有可验证的查询证据）。
7. `unknown` 必须有 `last_error` 与对账截止时间。
8. `cancelled` 只能从非终态进入；已发布内容只能走平台删除流程，不能改成 `cancelled`。

## 4. 幂等与防重

### 4.1 键

```text
idempotency_key = sha256(content_id | content_version | platform | account_id | payload_hash)
```

本地唯一约束：

```sql
UNIQUE (content_id, content_version, platform, account_id, payload_hash)
```

### 4.2 平台不支持幂等键时

| 步骤 | 动作 |
|---|---|
| 1 | 领取 job 时获取数据库租约（`lease_owner`、`lease_expires_at`） |
| 2 | 提交前查询平台最近帖子（时间窗 + 内容指纹/标题/首行） |
| 3 | 若找到唯一匹配，直接转 `published`，不提交 |
| 4 | 若找到多条，转 `unknown` 并告警，禁止提交 |
| 5 | 提交后保存 `provider_request_id`/`provider_post_id` |
| 6 | 超时进入 `unknown`；对账找到唯一匹配才转 `published` |
| 7 | 对账窗口内 0 条，且确认请求未到达平台，才允许转 `failed` |

### 4.3 内容指纹

用于查询匹配的指纹应稳定、可解释，不依赖平台返回：

```text
fingerprint = sha256(normalize(title + "\n" + first_paragraph) )
```

平台查询结果只作为“候选”，最终必须结合发布时间窗、账号、媒体哈希确认。

## 5. 超时与错误分类

| 情况 | 状态 | 处理 |
|---|---|---|
| `2xx` 且返回帖子 ID | `published` / `processing` | 保存 ID；异步平台转 `processing` |
| `400/422` 参数错误 | `failed` | 不可重试；提示修正内容 |
| `401` Token 过期 | `failed`（本次 attempt） | 刷新一次；失败则账号 `reauth_required`，job 回 `queued` 等授权 |
| `403` 权限/角色不足 | `failed` | 不可重试；提示重新授权或换账号 |
| `404` 资源不存在 | `failed` | 不可重试 |
| `409` 冲突 | `unknown` | 先查询；确认未创建再重试 |
| `429` 限流 | `queued` | 按 `Retry-After` 退避；不计入失败次数 |
| `5xx` | `unknown` | 先查询；确认未创建后按退避重试 |
| 连接超时/断网 | `unknown` | 不重试；对账 |
| 平台异步失败 | `failed` | 保存平台错误；不可盲目重传 |

重试策略：

- 最大 3 次；指数退避（1m、5m、15m）+ 随机抖动。
- `429` 不受 3 次限制，但受最大等待时间（如 24h）限制。
- `unknown` 不计入重试次数，必须人工/对账关闭。

## 6. 崩溃恢复

启动时扫描：

```sql
SELECT * FROM publish_jobs
WHERE state IN ('submitting','processing')
  AND (lease_expires_at IS NULL OR lease_expires_at < now());
```

处理：

1. `submitting` 且租约过期 → `unknown`，记录“worker 崩溃/租约过期”。
2. `processing` 且租约过期 → 保持 `processing`，交给 reconciler 查询；超过平台处理上限转 `unknown`。
3. 不自动重新提交。
4. 恢复动作写审计。

## 7. 对账（reconciler）

输入：`unknown` job + 账号 + 时间窗 + 内容指纹。

```text
查询平台最近 N 条内容
  ├─ 唯一匹配 → published（保存 post_id/url）
  ├─ 多条匹配 → unknown + 告警（人工选择）
  ├─ 0 条且已过安全窗口 → failed（允许人工重试）
  └─ 查询失败 → 保持 unknown，退避重试查询
```

安全窗口（建议值，P1 实测后调整）：

| 平台 | 提交后查询窗口 | 说明 |
|---|---|---|
| LinkedIn | 0–10 分钟 | 同步创建，201 即成功 |
| Facebook | 0–10 分钟 | 基本同步 |
| X | 0–10 分钟 | 同步创建 |
| YouTube | 0–24 小时 | 上传 + 转码异步 |
| Reddit | 0–30 分钟 | 人工确认场景不自动对账 |

## 8. 平台状态映射

| 本地状态 | LinkedIn | YouTube | Facebook | X | Reddit |
|---|---|---|---|---|---|
| `submitting` | 请求中 | 上传中/初始化 | 请求中 | 请求中 | 人工确认后请求 |
| `processing` | 一般不出现 | 转码/处理中 | 一般不出现 | 媒体处理中 | 不适用 |
| `published` | `201` + Post URN | `videos.list` 可见且公开/定时 | Post ID 可查 | Tweet ID 可查 | Post ID 可查 |
| `failed` | 4xx/5xx 明确错误 | 处理失败/版权/配额 | 4xx 明确错误 | 4xx 明确错误 | 规则拒绝 |
| `unknown` | 超时/409 | 上传中断/处理超时 | 超时 | 超时 | 人工未确认 |

## 9. 取消与删除

- 取消：仅允许 `draft/approved/queued/submitting/processing`。
- 已发布：走 `deletePost()`（平台支持时），并新建 `delete` 审计记录；不得把 `published` 改成 `cancelled`。
- 平台不支持删除时，标记 `delete_unsupported`，由人工到平台处理。

## 10. 测试要点

| 测试 | 断言 |
|---|---|
| 重复提交 | 同一 job 第二次提交被本地唯一键/租约拒绝 |
| 超时 | 状态进入 `unknown`，不自动重试 |
| 对账唯一匹配 | 转 `published`，保存 post_id |
| 对账多条 | 保持 `unknown` 并告警 |
| Token 过期 | 刷新一次；失败后账号 `reauth_required` |
| 429 | 按 `Retry-After` 回到 `queued` |
| 崩溃恢复 | 过期租约的 `submitting` 转 `unknown` |
| 版本变更 | 旧 job 不可复用，新 job 生成 |
| 取消 | 已发布不可取消 |
