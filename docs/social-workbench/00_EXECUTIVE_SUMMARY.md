# Morrishk 海外社媒工作台改造 · P0 执行摘要

| 项 | 内容 |
|---|---|
| 阶段 | P0：原仓库技术审计与改造设计 |
| 基线仓库 | `https://github.com/CzzzzzzJ/mdldm-maitian` |
| 审计基线提交 | `575e2998b9667ae2c61b175d6e8e17f94b3c7621`（main，2026-11-20 拉取） |
| 设计分支 | `docs/morrishk-social-workbench-design` |
| 调研截止 | 2026-11-20 |
| 本阶段性质 | 设计与审计，不含代码重构、不含第三方采购、不含真实账号发布 |

## 1. 结论

**可以做，但不能按“换一套发布适配器就完成”来估算。**

麦田的可复用价值集中在三块：以 `publish.json` 为中心的内容工程、SQLite 任务/状态/日志、以及守护进程式的计划与同步机制。它的发布执行、发布后核验、链接回收、流量采集和粉丝采集，全部建立在“专用 Chrome + 平台创作者后台 DOM/内部接口”之上；真正的平台填写代码不在本仓库，而是运行时从 Chrome 应用商店插件目录加载。因此：

1. 官方 API 化不是替换 `src/providers/` 下的几个文件，而是要把**发布执行、状态确认、数据采集**三条链路一起替换。
2. 现有状态机不足以支撑 API 发布：没有 `submitting` / `processing` / `unknown`，没有幂等键、平台请求 ID、平台帖子 ID，也没有“超时后先查再决定是否重试”的路径。
3. 现有数据模型把不同平台统一成 `views/likes/comments/favorites/shares`，并直接相加。官方 API 的指标定义并不一致，必须拆成“原始数据 + 标准化指标 + 平台差异声明”。
4. 现有转化只看 `utm_campaign` 和站点侧的注册/会员/订单/收入，没有 `utm_content`、商品页访问、询价点击、询价提交和“有效采购询盘”的业务状态。
5. OAuth 需要新建。当前系统没有平台 Token，只有浏览器登录态；内部系统（Ds Bit / Morrishk）也没有 Client Credentials 体系。

**建议的推进顺序不变：先 LinkedIn 官方 API 端到端 PoC，拿到真实账号的 `REAL_ACCOUNT_PASS`，再扩展 YouTube/Facebook，最后做 Analytics/UTM 与 X/Reddit。**

## 2. 官方 API 可行性判断

| 平台 | 发布 | 发布后状态 | Analytics | 主要门槛 | 结论 |
|---|---|---|---|---|---|
| LinkedIn | 可行：Posts API | 可行：按 Post URN 查询 | 可行：Share Statistics | Community Management API 审核；开发层 500 次/应用/日、100 次/成员/日；Token 60 天且程序化刷新仅限部分伙伴 | **P1 首选** |
| YouTube | 可行：`videos.insert` | 异步处理，需查 `videos.list` | 可行：YouTube Analytics API | OAuth 审核；上传 100 次/日；Analytics 配额另计 | P2 |
| Facebook | 可行：Page Feed/Photos/Videos | 基本同步，可查 Post | 可行但指标正在退役 | 应用审核、企业验证、Page 管理员；Insights 多项指标 2026-06-15 退役 | P2，需先锁定指标 |
| X | 可行：`POST /2/tweets` | 可查 Post | 有限：公开指标/自有内容读取 | 按量计费；带 URL 的帖子单价显著更高；无原生定时 | P4，先做成本闸门 |
| Reddit | 可行但须半自动 | 可查 Post 数据 | **无发布级 Analytics API** | 商业 API 协议、应用注册、规则与反垃圾限制；官方文档在本环境不可直接抓取 | P4，人工确认后发布；未签协议前阻断 API |

关键成本事实（2026-11-20 官方页面）：

- LinkedIn：API 调用本身不按次收费，但开发层有硬性日调用上限；生产需要 Standard Tier 审核。
- YouTube：Data API 默认配额为“其他端点合计 10,000 units/日”，`videos.insert` 与 `search.list` 各为 100 次/日；上传本身不单独收费。
- Facebook：Graph API 不按次收费；受限的是权限、审核和速率。
- X：已改为 **pay-per-usage credits，无订阅**。官方定价页列出：创建帖子 `$0.015/请求`，**带 URL 的帖子 `$0.200/请求`**，普通读取 `$0.005/资源`；自有内容读取 `$0.001/资源` 本次未能从价格表确认，**不纳入确定预算**；新用户有 `$20` 赠金。该价格必须在采购前复核。
- Reddit：商业用途需单独协议（Data API Terms 2026-07-20 修订）；未取得明确许可/协议前，**阻断 Reddit API 正式实施**，只允许人工草稿流程。

> 账号资产另见 `13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md`：当前没有取得任何 Morrishk 真实账号名称、URL、管理员或开发者应用证据；8 个平台仅为待确认占位记录，不能据此进入 P1 真实发布。OAuth、Scope、Token 状态在没有检查平台后台前一律记 `UNKNOWN`，不写成“未授权”。

## 3. 本阶段交付

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
  13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md
```

## 4. 核心架构决策（待 Reviewer 审核）

1. **保留麦田的“内容工程 + 调度 + 状态 + 看板 + 飞书”骨架，替换执行与采集。**
2. **新增平台无关内容模型**，`publish.json` 只作为导入/导出格式，不再作为运行时唯一事实源。
3. **新增三层独立门禁**：Ds Bit 技术事实 `RELEASE_APPROVED`、Morrishk 商品公开状态 `PUBLIC_ELIGIBLE`、社媒内容 `SOCIAL_CONTENT_APPROVED`；三者同时满足才可创建发布任务。
4. **新增可恢复状态机**，以 `content_id + platform + account_id + payload_hash` 作为本地幂等键。
5. **平台适配器只做官方 API**；不支持的接口显式返回 `UNSUPPORTED`，不模拟成功。
6. **Analytics 先存原始响应，再做标准化**；平台没有的指标记 `unavailable`，不填 0。
7. **UTM 归因以 Morrishk 站点事件为准**，社媒 API 数据不能推断询盘转化。
8. **OAuth 分三层**：内部 Client Credentials、Morrishk 用户 OIDC + PKCE、外部平台独立 OAuth。
9. **SQLite 采用增量迁移**，保留旧表和旧数据；不做破坏性 schema 重写。
10. **Chrome 插件保留为过渡/回退路径，但不进入新核心依赖**；删除前必须先完成调用链替换与对账。

> `RELEASE_APPROVED` 只表示 Ds Bit 技术事实可作为对外来源，不表示商品可公开，也不表示社媒文案已批准。商品状态与社媒内容审批必须分别校验。

## 5. 主要阻塞与未决事项

| 编号 | 事项 | 影响 | 责任方 |
|---|---|---|---|
| P0-B1 | LinkedIn 开发者应用与 Community Management API 开发层是否已具备 | P1 无法真实发布 | 项目负责人 |
| P0-B2 | Morrishk LinkedIn Company Page 是否存在、当前管理员是谁 | 组织帖授权 | 项目负责人 |
| P0-B3 | Standard Tier 申请材料（录屏、用例）谁提供 | 生产可用时间 | 项目负责人 |
| P0-B4 | Ds Bit 哪些字段属于 `RELEASE_APPROVED`；商品 `PUBLIC_ELIGIBLE` 由谁维护；`SOCIAL_CONTENT_APPROVED` 审批人是谁 | 内容合规/门禁 | Ds Bit / 商品 / 内容负责人 |
| P0-B5 | Morrishk 站点是否已有 `utm_content`、询价事件、有效询盘状态接口 | 归因闭环 | Morrishk 站点 |
| P0-B6 | X 的月度预算上限与“带 URL 帖子”策略 | 是否进入 P4 | 项目负责人 |
| P0-B7 | Reddit 商业 API 协议、目标 subreddit、账号身份；未获协议前不启用 API | 是否允许 API 接入 | 运营/法务负责人 |
| P0-B8 | 工作台部署环境（macOS 常开机器 / Linux / Windows） | 守护进程与 Keychain 方案 | 运维 |
| P0-B9 | Morrishk 统一身份认证中心是否存在（OIDC Issuer） | 用户登录授权 | 平台负责人 |
| P0-B10 | Morrishk 各平台真实账号名称、URL、归属、管理员与开发者应用均未提供 | P1 无法选定真实测试 Page；所有账号级能力为 `UNKNOWN` | 项目/市场负责人 |

## 6. 本阶段明确没有做的事

- 没有修改任何业务代码、数据库、路由、生产配置。
- 没有注册任何开发者应用，没有申请任何权限，没有购买 X credits。
- 没有连接或发布任何真实 Morrishk 账号内容。
- 没有把任何 Mock 结果写成真实 API 已打通。
- 没有宣布架构冻结或进入 P1 实施。

## 7. 审核请求

请 Reviewer 重点裁定：

1. 是否接受“保留中控、替换执行与采集”的总体路线。
2. 是否接受 Ds Bit 技术事实、Morrishk 商品状态、社媒内容三层独立门禁，且任何一层都不能替代另外两层。
3. 是否接受状态机、幂等键与 `unknown` 处理方案。
4. 是否接受 OAuth 三层边界与 Token 加密存储方案。
5. 是否接受 P1 只做 LinkedIn、且以真实测试 Page 的 `REAL_ACCOUNT_PASS` 为通过条件。
6. 是否同意 P0 阻塞项由项目负责人先关闭，再进入 P1。
7. 是否接受“账号资产未提供时，P1 只能准备代码与 Mock，不得选定或连接真实账号”的结论（见 `13`）。
