# Analytics 与 UTM 归因设计

核心目标不是“播放量”，而是：**哪条内容、哪个平台、哪个商品，带来了有效的 B2B 采购询盘。**

## 1. 指标分层

```text
平台原始响应（analytics_raw）
  └─ 平台原生指标（native_name）
       └─ 标准指标（metric_name）
            ├─ 可跨平台比较的“展示/互动”层
            └─ 平台特有指标（不合并）
```

三层必须同时保存：

1. **原始响应**：平台返回的完整 JSON + 请求参数 + API 版本 + 采集时间。
2. **原生指标**：平台字段名与数值，保留单位与口径。
3. **标准指标**：映射到统一名称，但**不改变口径**。

### 1.1 标准指标命名

| 标准名 | 含义 | 可跨平台合并 |
|---|---|---|
| `impressions` | 内容被展示的次数 | 谨慎：各平台定义不同 |
| `reach` | 去重触达人数（平台提供时） | 谨慎 |
| `views` | 内容被观看次数 | 谨慎：视频/帖子口径不同 |
| `likes` | 点赞/喜欢 | 可合并为“互动”子项，但需标注 |
| `comments` | 评论/回复 | 可合并为“互动”子项 |
| `shares` | 转发/分享/转帖 | 可合并为“互动”子项 |
| `clicks` | 内容内链接点击 | 仅部分平台提供 |
| `followers` | 粉丝/订阅总数 | 不合并 |
| `subscribers_gained` | 新增订阅 | 不合并 |
| `watch_time_seconds` | 观看时长 | 仅视频平台 |

**禁止**：把 LinkedIn `impressionCount`、YouTube `views`、Facebook `post_impressions`、X `impression_count` 直接相加成“总曝光”。UI 必须分别展示，或使用明确标注的“平台加权口径”。

### 1.2 平台指标可用性

| 平台 | 可用 | 不可用/待确认 |
|---|---|---|
| LinkedIn | impressions、unique impressions、clicks、likes、comments、shares、engagement | 视频观看完成率需 Video Analytics；仅 12 个月 |
| YouTube | views、likes、comments、shares、subscribers、watch time | impressions/CTR 待确认；Analytics 配额待确认 |
| Facebook | post_impressions、reach、clicks、reactions、comments、shares、video views | 多项指标 2026-06-15 退役；仅 100+ likes、两年 |
| X | impression_count、like_count、reply_count、retweet_count、quote_count、bookmark_count | URL 点击、转化不提供；仅自有内容可靠 |
| Reddit | score、comment count、upvote ratio | **无 impressions/reach/点击** |

## 2. 采集设计

### 2.1 调度

| 数据 | 频率 | 说明 |
|---|---|---|
| 发布状态 | 提交后 1m/5m/15m/1h，之后每日 | 由 reconciler/状态机驱动 |
| 帖子级指标 | 发布后 1h、6h、24h、第 3/7/30 天 | 与复盘节点对齐 |
| 账号级指标 | 每日一次 | 粉丝、订阅 |
| 历史回填 | 首次接入时 | 平台窗口内（LinkedIn 12m、Meta 2y） |

### 2.2 原始保存

```text
analytics_raw:
  platform, account_id, provider_post_id, metric_date, endpoint,
  request_params, payload_json, payload_hash, collected_at, api_version
```

- 只追加；同一请求重复采集用 `payload_hash` 去重。
- 原始响应脱敏后保存；不得包含 Token、Cookie、个人信息。
- 保存 API 版本，便于口径变化追溯。

### 2.3 标准化

```text
analytics_daily:
  platform, account_id, provider_post_id, content_id,
  metric_date, metric_name, value_num, availability,
  unavailable_reason, unit, source_raw_id
```

规则：

1. 平台返回 `null`/字段缺失 → `availability='unavailable'`，`value_num=NULL`。
2. 平台返回 0 → `availability='available'`，`value_num=0`。
3. 字段被平台标记 deprecated → `unavailable_reason='deprecated_by_platform'`。
4. 权限不足 → `unavailable_reason='permission_denied'`。
5. 数据延迟 → `unavailable_reason='data_delayed'`，下次重试。
6. 每个值必须能追溯到 `source_raw_id`。

## 3. UTM 规范

统一参数：

```text
utm_source   = linkedin | facebook | youtube | x | reddit
utm_medium   = social
utm_campaign = {campaign_id}
utm_content  = {content_id}
```

可选：

```text
utm_term = {product_mpn}      # 仅用于商品级细分时
```

规则：

1. `utm_campaign` 与 `campaign_id` 一一对应，不使用自由文本。
2. `utm_content` 与 `content_id` 一一对应；同一内容多平台变体仍共用 `content_id`。
3. 链接必须由系统生成，不允许运营手写。
4. 短链/跳转必须保留 UTM；不得在跳转中丢失参数。
5. 不在不支持外链的平台强行放链接（如 Instagram 文案、Reddit 版规禁止时）。

### 3.1 平台链接位置

| 平台 | 链接位置 | 备注 |
|---|---|---|
| LinkedIn | 帖子正文/评论/文章 | 正文链接可能影响触达，需 A/B |
| Facebook | 帖子正文 | 可配合链接卡片 |
| YouTube | 视频描述/置顶评论 | 描述首行 |
| X | 帖子正文 | **带 URL 帖子单价 $0.200**，需预算闸门 |
| Reddit | 评论/正文（版规允许时） | 默认不放，人工判断 |

## 4. 归因链路

```text
社媒发布
  └─ provider_post_id → publish_jobs → content_id → campaign_id
       └─ 目标 URL + UTM
            └─ Morrishk 商品详情页访问（page_view）
                 └─ 询价按钮点击（cta_click）
                      └─ 询价提交（inquiry_submit）
                           └─ 销售确认有效（inquiry_qualified）
```

### 4.1 站点侧需要的事件

| 事件 | 必需字段 | 说明 |
|---|---|---|
| `page_view` | `occurred_at`、`url`、`utm_*`、`session_id`、`product_mpn` | 首次落地页与商品页都要记 |
| `cta_click` | `content_id`/`utm_content`、`product_mpn`、`session_id` | 询价按钮/表单打开 |
| `inquiry_submit` | `inquiry_id`、`product_mpn`、`utm_*`、`session_id` | 表单提交成功 |
| `inquiry_qualified` | `inquiry_id`、`qualified`、`qualified_at`、`reason` | 由 CRM/销售回写 |

### 4.2 有效询盘定义

“有效采购询盘”不是表单提交数。建议由业务定义并配置：

```text
inquiry_qualified = 已联系 + 有明确产品/数量/应用 + 联系信息可验证 + 非垃圾/竞品
```

具体字段与判定人由 Morrishk 销售负责人确认；工作台只存 `inquiry_id`、状态、时间、理由，不复制 CRM 明细。

### 4.3 归因模型

| 模型 | 用途 | 规则 |
|---|---|---|
| Last-touch（内容） | 评估“哪条内容促成询价” | 询盘前最后一次 `utm_content` |
| First-touch（活动） | 评估“哪个活动带来新客” | 会话/线索首次 `utm_campaign` |
| 多触点 | 辅助 | 保存事件序列，按需计算 |

默认展示 last-touch 与 first-touch 两列，**不合并成一个“功劳”数字**。

### 4.4 指标

| 指标 | 公式 | 说明 |
|---|---|---|
| 平台访问量 | `count(distinct session_id)` by platform | 站点事件，不是社媒 impressions |
| 商品页访问量 | `page_view` where `product_mpn` | 可按商品/内容 |
| 询价点击量 | `cta_click` | 去重按 session |
| 询价提交量 | `inquiry_submit` | 去重按 inquiry_id |
| 有效询盘量 | `inquiry_qualified` | 业务确认 |
| 有效率 | `qualified / submits` | 分平台/内容 |
| 访问→询价率 | `submits / visits` | 分平台/内容 |
| 询价→有效 rate | `qualified / submits` | 销售质量 |

## 5. 数据关联

### 5.1 社媒帖子 → 内容

```sql
SELECT j.content_id, j.campaign_id, p.provider_post_id
FROM publish_jobs j
JOIN publish_attempts p ON p.job_id = j.job_id
WHERE p.provider_post_id = ?;
```

### 5.2 站点事件 → 内容

```sql
SELECT content_id, campaign_id
FROM attribution_events
WHERE utm_content = ? AND event_type = 'page_view'
ORDER BY occurred_at
LIMIT 1;
```

### 5.3 询盘 → 内容

```sql
SELECT inquiry_id, content_id, campaign_id, platform,
       MIN(occurred_at) AS first_seen
FROM attribution_events
WHERE inquiry_id = ?
GROUP BY inquiry_id, content_id, campaign_id, platform;
```

## 6. 反模式

1. **用社媒 API 的 impressions 推断询盘转化**：社媒 API 不知道站点行为。
2. **把不同平台的 views/impressions 相加**：口径不同。
3. **把 unavailable 填 0**：会拉低均值、掩盖权限问题。
4. **只记 campaign 不记 content**：无法评估单条内容。
5. **用短链覆盖 UTM**：跳转丢参数。
6. **把表单提交当有效询盘**：B2B 噪声高。
7. **跨设备强行 identity**：无登录时只做会话级，不拼接个人身份。

## 7. 隐私与合规

- 工作台不存访客姓名、电话、邮箱；只存 `session_id`、`inquiry_id`（伪匿名）。
- `session_id` 使用随机 UUID，不使用指纹。
- 询盘明细留在 Morrishk CRM；工作台通过受控接口读状态。
- 遵守站点隐私政策与 Cookie 同意；UTM 参数不包含个人信息。
- 数据保留 ≥ 24 个月；到期聚合后删除明细。

## 8. 验收

| 测试 | 断言 |
|---|---|
| 字段映射 | 每个平台原生字段能映射到标准名，且有 registry 记录 |
| unavailable | 权限不足时值为 NULL，reason 正确 |
| 原始可追溯 | 每个 daily 值能查到 raw |
| UTM 生成 | campaign/content 与数据库一致，无手写 |
| 归因 | 给定事件序列，last/first-touch 结果正确 |
| 有效询盘 | 只有 `inquiry_qualified` 计入 |
| 跨平台 | 不同平台指标不被错误相加 |
| 时间窗 | LinkedIn 12 个月、Meta 2 年边界处理正确 |
