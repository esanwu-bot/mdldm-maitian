# CHANGELOG

## 2026-11-20 · P0-R1 修订 + 账号资产盘点

### 分支与基线

- 分支：`docs/morrishk-social-workbench-design-r1`
- 父提交：`69ad5b8e1dec9591b059b253c5756314df535483`（P0）
- 未修改 P0 历史，未 amend，未强推。
- 未修改业务代码、数据库、生产配置。

### R1 修订

| 编号 | 修订 |
|---|---|
| R1-01 | 拆分三层门禁：Ds Bit 技术事实 `RELEASE_APPROVED`、商品 `PUBLIC_ELIGIBLE`、社媒内容 `SOCIAL_CONTENT_APPROVED`；任何一项不能替代另外两项 |
| R1-02 | 明确 Token 签发方前提（已有 OAuth / 仅 API Key-JWT / 都没有）；`audience` 改为 RFC 8707 `resource`（或记录 AS 扩展）；Secret 哈希归 AS，工作台只存 `secret_ref` |
| R1-03 | LinkedIn 身份 Scope 分离：OIDC `openid profile email`；旧版成员资料 `r_liteprofile`/`r_basicprofile`，不得混用 |
| R1-04 | Facebook 改为动态 Platform/BUC 限流，不写死 200×/4800×；以响应头为准 |
| R1-05 | X 自有读取 `$0.001` 标记未确认，不纳入预算；按 `$0.005/resource` 保守估算 |
| R1-06 | Reddit 商业 API 协议门禁（2026-07-20 修订条款）；未获批前阻断 API；数据保留按平台策略，不默认 24 个月 |
| R1-07 | 修正行数统计口径：P0 的 3,015 = 13 份正文 3,002 行（git numstat）+ `SHA256SUMS.txt` 13 行；此前 3,001 为 `wc -l` 少算 1 行且未计校验和文件 |
| R1-08 | 账号盘点中的 OAuth/Scope/Token 状态改为 `UNKNOWN`，区分“未发现记录”与“已确认未授权”；只有检查平台授权后台后才可用 `NOT_AUTHORIZED`/`UNAUTHORIZED` |
| R1-09 | 在 04/06/12 明确当前实施平台与后续评估平台；`platform_registry`、`account_type`、`capabilities_json` 预留扩展，不把 5 个平台写成封闭集合 |

### 新增文件

- `13_SOCIAL_PLATFORM_AND_ACCOUNT_INVENTORY.md`：社媒/视频平台与账号资产盘点。
- R1 后文档共 **14 份**（00–13）；提交新增行数以 `git show --numstat` 为准，不以 `wc -l` 估算。
- 结论：未取得任何 Morrishk 真实账号名称、URL、管理员或开发者应用证据；8 个平台仅为待确认占位记录；P1 无法选定真实测试 Page。

### 未做

- 未注册账号/开发者应用。
- 未申请权限、未购买套餐/额度。
- 未连接或发布真实账号。
- 未收集密码/Cookie/Token/Secret。
- 未虚构账号名称、URL 或管理员。

## 2026-11-20 · P0 设计交付

### 新增

```text
docs/social-workbench/00_EXECUTIVE_SUMMARY.md
docs/social-workbench/01_EXISTING_SYSTEM_AUDIT.md
docs/social-workbench/02_TARGET_ARCHITECTURE.md
docs/social-workbench/03_PLATFORM_API_MATRIX.md
docs/social-workbench/04_DATA_MODEL.md
docs/social-workbench/05_PUBLISH_STATE_MACHINE.md
docs/social-workbench/06_OAUTH_SECURITY.md
docs/social-workbench/07_ANALYTICS_ATTRIBUTION.md
docs/social-workbench/08_COST_AND_RISK.md
docs/social-workbench/09_IMPLEMENTATION_PLAN.md
docs/social-workbench/10_ACCEPTANCE_TESTS.md
docs/social-workbench/11_CHANGELOG.md
docs/social-workbench/12_OAUTH2_AUTHORIZATION_ARCHITECTURE.md
```

### 基线

- 仓库：`CzzzzzzJ/mdldm-maitian`
- 审计提交：`575e2998b9667ae2c61b175d6e8e17f94b3c7621`
- 分支：`docs/morrishk-social-workbench-design`
- 审计范围：`src/*.mjs`、`src/web/*`、`skills/maitian-publish/SKILL.md`、`config.example.json`、`README.md`
- 未修改任何业务代码、数据库、生产配置。

### 调研

- LinkedIn / YouTube / Facebook / X 官方文档：本次直接抓取并引用。
- Reddit 官方页面：本环境 403，标记 B/C 级，待注册后核验。
- 价格核对：2026-11-20；X 为按量计费，带 URL 帖子 $0.200/请求。

### 关键结论

1. 中控骨架可复用；发布、核验、取链接、采集必须重写。
2. Chrome 插件是当前共同底座，移除前必须逐条替换并对账。
3. 现有状态机/数据模型不足，需新增状态、幂等、原始分析、归因、审计表。
4. 官方 API 可行，但 LinkedIn 审核与 Token、Meta 指标退役、X 成本、Reddit 规则是主要风险。
5. P1 只做 LinkedIn，以真实测试 Page 的 `REAL_ACCOUNT_PASS` 为通过条件。

### 未做

- 未注册开发者应用。
- 未申请权限。
- 未购买任何服务/额度。
- 未连接或发布真实账号。
- 未实现代码。
- 未宣布设计冻结或进入 P1。

### 待审核

- [ ] Reviewer 对 P0-R1 设计出具结论
- [ ] 项目负责人关闭 P0-B1..B10
- [ ] 提供 Morrishk 真实账号资产（名称/URL/归属/管理员/开发者应用）
- [ ] 确认 P1 测试 Page 与内容审批人
- [ ] 确认 Morrishk 站点事件/CRM 接口
- [ ] 确认内部 OAuth AS/网关能力
- [ ] 确认 X 预算（若进入 P4）
