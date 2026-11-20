# 验收与测试计划

## 1. 验收等级

| 等级 | 含义 | 能证明什么 | 不能证明什么 |
|---|---|---|---|
| `CODE_PASS` | 单元/静态/构建通过 | 代码逻辑符合预期 | 平台 API 可用 |
| `MOCK_PASS` | 本地 mock 服务契约通过 | 请求/响应/错误处理正确 | 真实权限、真实发布 |
| `SANDBOX_PASS` | 平台沙箱/测试模式通过 | 平台接口在测试环境可用 | 生产账号可用 |
| `REAL_ACCOUNT_PASS` | 真实测试账号完成授权、发布、查询 | 官方 API 端到端可用 | 生产规模/长期稳定性 |

**Mock 通过不得写成真实 API 已打通。**  
**Sandbox 通过不得写成生产可用。**  
**未取得真实账号证据时，只能标记 `MOCK_PASS` 并列出阻塞。**

## 2. 测试分层

```text
test/
  unit/          纯函数、状态机、加密、映射
  contract/      Adapter 对 mock HTTP server 的契约
  integration/   SQLite + worker + 本地 mock
  security/      日志/DB/错误脱敏
  real/          真实账号脚本（手动触发，不进 CI）
```

命令建议（Node 内置测试）：

```bash
node --test test/unit
node --test test/contract
node --test test/integration
node --test test/security
# real 测试不自动运行，需显式环境变量与人工确认
```

## 3. P1 测试矩阵

### 3.1 OAuth

| ID | 用例 | 预期 | 等级 |
|---|---|---|---|
| OAuth-01 | 正常授权 | 回调成功，账号入库，Token 加密 | REAL |
| OAuth-02 | `state` 不匹配 | 401，不换 Token | MOCK |
| OAuth-03 | 用户取消 | 友好错误，无账号记录 | MOCK |
| OAuth-04 | code 过期/重复使用 | 明确错误，提示重新授权 | MOCK |
| OAuth-05 | Scope 缺少 `w_organization_social` | 账号状态 `insufficient_scope`，禁止发布 | MOCK/REAL |
| OAuth-06 | 非 Page 管理员 | 不显示该 Page 或禁止绑定 | REAL |
| OAuth-07 | Token 过期 | 401 → 刷新/`reauth_required` | MOCK |
| OAuth-08 | 撤销授权 | Token 删除，queued job 取消 | MOCK |

### 3.1A 三层门禁（R1 新增）

| ID | 用例 | 预期 | 等级 |
|---|---|---|---|
| GATE-01 | Ds Bit 事实非 `RELEASE_APPROVED` | 拒绝创建 job | UNIT |
| GATE-02 | 商品非 `PUBLIC_ELIGIBLE` | 拒绝创建 job | UNIT |
| GATE-03 | 社媒内容非 `SOCIAL_CONTENT_APPROVED` | 拒绝创建 job | UNIT |
| GATE-04 | 三项全部满足 | 允许创建 job | UNIT |
| GATE-05 | 仅 `RELEASE_APPROVED` | **拒绝**（不能替代商品/社媒） | UNIT |
| GATE-06 | 仅 `SOCIAL_CONTENT_APPROVED` | **拒绝**（不能替代技术/商品） | UNIT |
| GATE-07 | 商品状态在排队后撤销 | 取消 queued job；已发布走删除流程 | UNIT |
| GATE-08 | 社媒内容修改 | 旧审批失效，需重新 `SOCIAL_CONTENT_APPROVED` | UNIT |

### 3.2 发布

| ID | 用例 | 预期 | 等级 |
|---|---|---|---|
| PUB-01 | 文本帖成功 | 201 + URN → published | REAL |
| PUB-02 | 图片帖成功 | 图片 URN + 帖子 URN | REAL（可选） |
| PUB-03 | 400 参数错误 | failed，不重试 | MOCK |
| PUB-04 | 401 | 刷新一次；失败 `reauth_required` | MOCK |
| PUB-05 | 403 角色不足 | failed，提示 Page 角色 | REAL/MOCK |
| PUB-06 | 429 | 按 Retry-After 回 queued | MOCK |
| PUB-07 | 5xx | unknown → 对账 | MOCK |
| PUB-08 | 超时 | unknown，不自动重试 | MOCK |
| PUB-09 | 重复提交 | 第二次不产生新帖 | REAL |
| PUB-10 | 内容版本变更 | 新 job，旧 job 不复用 | UNIT |
| PUB-11 | 社媒内容未审批/商品不合格 | 拒绝创建 job | UNIT |
| PUB-12 | Page 与账号不匹配 | 拒绝发布 | UNIT/REAL |
| PUB-13 | 删除帖子 | 204，审计记录 | REAL（可选） |

### 3.3 状态机

| ID | 用例 | 预期 |
|---|---|---|
| SM-01 | 合法转移 | 全部按图转移 |
| SM-02 | 非法转移 | 抛错/拒绝 |
| SM-03 | `unknown` 自动重试 | 禁止 |
| SM-04 | 租约过期恢复 | submitting → unknown |
| SM-05 | 对账唯一匹配 | published |
| SM-06 | 对账多条 | unknown + 告警 |
| SM-07 | 对账 0 条且窗口过 | failed |
| SM-08 | 取消已发布 | 拒绝 |
| SM-09 | 重试上限 | 超过 max_attempts 不再自动 |

### 3.4 Token 与安全

| ID | 用例 | 预期 |
|---|---|---|
| SEC-01 | DB 转储扫描 | 无 `access_token`/`refresh_token` 明文 |
| SEC-02 | 日志扫描 | 无 Token/Secret/Code |
| SEC-03 | 错误页 | 不展示平台原始响应 |
| SEC-04 | 加密 AAD 错误 | 解密失败，不返回明文 |
| SEC-05 | 密钥轮换 | 新旧版本均可解密，新写入用新版本 |
| SEC-06 | 撤销后发布 | 拒绝 |

### 3.5 数据与迁移

| ID | 用例 | 预期 |
|---|---|---|
| DB-01 | 首次迁移 | 新表创建，`user_version=1` |
| DB-02 | 重复迁移 | 幂等，不报错 |
| DB-03 | 迁移前备份 | 备份文件存在且可打开 |
| DB-04 | 旧表数据 | 不修改、不删除 |
| DB-05 | 大数据原始响应 | 不阻塞/可归档 |
| DB-06 | 事务失败 | 回滚，服务停止 |
| DB-07 | 平台保留策略 | 按平台执行；Reddit 未获批时无数据 |

### 3.6 Analytics 与归因

| ID | 用例 | 预期 |
|---|---|---|
| AN-01 | 字段映射 | 原生字段 → 标准名，registry 有记录 |
| AN-02 | 权限不足 | `unavailable` + reason，值 NULL |
| AN-03 | 平台返回 0 | `available` + 0 |
| AN-04 | 原始追溯 | daily 值可查 raw |
| AN-05 | 跨平台合并 | 不把不同口径直接相加 |
| AN-06 | LinkedIn 12 个月边界 | 超出返回空/明确提示 |
| AN-07 | Meta 指标退役 | 标记 deprecated，不写 0 |
| AN-08 | Reddit 未签协议 | 不采集、不持久化；仅草稿 |
| UTM-01 | 生成 UTM | campaign/content 与 DB 一致 |
| UTM-02 | 跳转保留参数 | 不丢失 |
| AT-01 | last-touch | 询盘前最后 content 正确 |
| AT-02 | first-touch | 首次 campaign 正确 |
| AT-03 | 有效询盘 | 仅 `qualified` 计入 |
| AT-04 | 表单未合格 | 不计入有效 |

### 3.7 真实账号验收（P1）

必须保存以下证据（脱敏）：

1. 应用 ID、Page URN（必须来自 `13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md` 中已核实的账号）。
2. 授权时间、Scope、账号类型。
3. 发布请求：时间、HTTP 状态、Post URN、Post URL。
4. 查询结果：`lifecycleState`。
5. 第二次提交：无新帖子（列表/URN 对比）。
6. Token 失效：模拟/真实过期后的行为。
7. 429/超时：Mock 或真实触发记录。
8. 测试帖处置：保留或删除，及审计记录。

通过条件：

- `REAL_ACCOUNT_PASS`：1、3、4、5 全部有证据。
- 若 1/3 无法完成：只能 `MOCK_PASS`，并列出 P1-A1..A8 阻塞项。

## 4. P2 测试重点

| 平台 | 重点 |
|---|---|
| YouTube | 断点续传、处理状态轮询、`publishAt`、配额、视频过大、版权/处理失败 |
| Facebook | Page Token、`CREATE_CONTENT`、定时 10 分钟–30 天、动态限流响应头、指标退役、100 likes 限制 |
| 通用 | 媒体上传失败、异步状态、跨平台不合并 |

## 5. P3 测试重点

- 原始/标准化一致性。
- 采集窗口与平台限制。
- UTM 全链路（帖子 → 落地页 → 商品页 → 询价）。
- CRM 有效询盘回写。
- 归因可复算（同一事件序列结果稳定）。

## 6. P4 测试重点

- X 预算上限、带 URL 帖子成本、超时不重复计费。
- Reddit 人工确认、版规、无 Analytics 不伪造。
- **Reddit 前置门禁**：未取得商业 API 协议/明确许可时，P4 只能验收“草稿 + 人工发布”，不得调用 `submit` 或采集 API 数据。
- Facebook 限流：不得以固定 200×/4800× 作为验收值；必须验证响应头驱动的降速/暂停。
- RBAC：无权限用户不能审批/发布。
- 审计：所有关键动作可追溯。

## 7. 缺陷分级

| 级别 | 定义 | 处理 |
|---|---|---|
| P0 | 重复发布、越权、Token 泄漏、数据损坏 | 阻断，立即修复 |
| P1 | 真实发布失败、状态不可恢复、审批绕过 | 阻断 |
| P2 | 指标错误、UI 错误、非关键失败 | 修复后通过 |
| P3 | 文案/体验问题 | 可排期 |

## 8. 测试报告模板

```text
阶段：
分支：
Commit：
环境：dev/staging/prod
测试命令：
结果：通过 x / 失败 y / 跳过 z
失败用例：
证据位置：
未完成：
验收等级：
结论：不通过 / 有条件通过 / 通过
```
