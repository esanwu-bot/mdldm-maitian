# CHANGELOG

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

- [ ] Reviewer 对 P0 设计出具结论
- [ ] 项目负责人关闭 P0-B1..B9
- [ ] 确认 P1 测试 Page 与内容审批人
- [ ] 确认 Morrishk 站点事件/CRM 接口
- [ ] 确认 OIDC 身份源
- [ ] 确认 X 预算（若进入 P4）