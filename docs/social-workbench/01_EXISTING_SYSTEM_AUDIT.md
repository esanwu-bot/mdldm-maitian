# 原系统技术审计

审计对象：`CzzzzzzJ/mdldm-maitian`，审计基线 `575e2998b9667ae2c61b175d6e8e17f94b3c7621`。  
方法：完整阅读 `src/*.mjs`、`src/web/*`、`skills/maitian-publish/SKILL.md`、`config.example.json`、`README.md`；未运行任何平台发布动作。

## 1. 项目形态

| 项 | 事实 | 证据 |
|---|---|---|
| 运行时 | Node.js ≥ 22.5，ESM | `package.json` |
| 依赖 | **零 npm 运行时依赖**；SQLite 使用内置 `node:sqlite` | `package.json`；`src/db.mjs:2,9` |
| 入口 | `maitian` → `src/hub.mjs` | `package.json` |
| 平台 | README 标注 macOS-only；代码含 Windows Chrome 路径与安装脚本 | `README.md`；`src/chrome-shim.mjs:9-11` |
| 发布引擎 | **不在本仓库**，运行时从专用 Chrome 的商店插件目录加载 | `src/plugin.mjs:16-33`；`src/driver.mjs:11-19` |
| 数据 | `data/hub.db`（可用 `MAITIAN_DB` 覆盖） | `src/db.mjs:6-9` |
| 配置 | `config.json`，不入 Git | `src/feishu.mjs:3-5`；`.gitignore` |
| 许可 | MIT（仅覆盖本仓库；商店插件代码不在仓库内） | `LICENSE` |

## 2. 模块清单

| 模块 | 行数 | 职责 | 与 Chrome/平台的耦合 |
|---|---:|---|---|
| `src/hub.mjs` | 471 | CLI 与守护进程：发布、同步、体检、采集、计划、复盘触发 | 高：所有执行都经 `driver` |
| `src/driver.mjs` | 83 | 驱动选择（direct/plugin）、批次执行、登录检查、取链接、截图 | **极高**：direct 直接 import 插件 `lib/publish.js`/`lib/cdp.js` |
| `src/chrome-shim.mjs` | 116 | 启动专用 Chrome（CDP 9333），模拟 `chrome.tabs`/`chrome.debugger` | **极高**：DOM 自动化底座 |
| `src/plugin.mjs` | 39 | 定位商店插件版本目录 | 高：依赖 `Extensions/<id>/<ver>_0` |
| `src/checks.mjs` | 182 | 发布前查重、填写后回读、自动提交（X/YouTube/Instagram/知识站/抖音） | **极高**：全部是 DOM 选择器 |
| `src/metrics.mjs` | 77 | 7 平台播放/互动采集 | **极高**：页面内 `fetch(..., credentials:'include')` 或 DOM |
| `src/followers.mjs` | 86 | 7 平台粉丝采集与历史回填 | **极高**：同上 |
| `src/db.mjs` | 65 | SQLite schema、任务状态、日志、指标 | 无 |
| `src/web.mjs` | 187 | 本地 Web 看板/API/口令与飞书验证码登录/小组件 | 中：健康检查探测 CDP；操作以子进程调 `hub` |
| `src/make-publish.mjs` | 134 | Markdown 发布素材 → `publish.json` | 高：字段规则来自插件 `lib/bundle.js` |
| `src/validate.mjs` | 39 | 校验 `publish.json`、封面、faststart | 高：同上 |
| `src/board.mjs` | 100 | 飞书多维表格同步；站点数据读取 | 中：站点数据用浏览器登录态 |
| `src/site.mjs` | 19 | 自有站点接口契约 | 低：仅配置映射 |
| `src/review.mjs` | 125 | 第 1/3/7 天复盘、周报 | 无（读库） |
| `src/feishu.mjs` | 59 | 飞书卡片通知 | 无 |
| `src/bridge.mjs` | 38 | 插件开发桥模式 | 高：依赖插件 `tools/bridge-server.mjs` |
| `src/login.mjs` / `access.mjs` | 94 / 34 | Web 飞书验证码 / Cloudflare Access | 无 |
| `src/init.mjs` / `demo.mjs` / `tunnel.mjs` / `learn.mjs` | 91 / 74 / 36 / 7 | 安装向导 / 演示 / 隧道 / 教程链接 | 低 |
| `src/web/board.html`、`pages.html`、`mt.js` | — | 单页看板与详情 | 无 |

## 3. 关键调用链

### 3.1 发布链

```text
hub run <dir>
  └─ validateProject(dir)                         src/hub.mjs:124
       └─ import <plugin>/lib/bundle.js           src/validate.mjs:7
  └─ upsertEpisode / saveSnapshot                 src/hub.mjs:127
  └─ 防重 1：本地 LOCKED 状态                     src/hub.mjs:137-139
  └─ drv.ensure() → 启动/连接专用 Chrome          src/hub.mjs:142；driver.mjs:21
  └─ 防重 2：平台页面查重                         src/hub.mjs:145-154
       └─ xExisting / ytRows / igExisting / zszCourses / douyinTopics
  └─ drv.checkLogin(todo)                         src/hub.mjs:169
  └─ 按 config.batches 分批                        src/hub.mjs:176-186
       └─ drv.runBatch
            ├─ direct: import <plugin>/lib/publish.js
            │          pub.run(project, platforms, onChange)
            └─ plugin: bridge.panel('startRaw') + 轮询 job
  └─ after(): 截图 → 回读 → 自动提交 → setTask     src/hub.mjs:66-88
       └─ xSubmit / ytSubmit / igSubmit（DOM 点击）
  └─ dev 通知 + boardSync
```

### 3.2 链接回收与状态同步链

```text
hub sync / daemon 定时
  └─ drv.links(project, ids)
       └─ <plugin>/lib/publish.js collectLinks(...)   src/driver.mjs:71-74
  └─ 根据返回 status/url 映射 scheduled/published       src/hub.mjs:213-216
  └─ 写 <工程>/发布记录.json                            src/hub.mjs:228
  └─ 回填发布素材 md 的「已发布」行                     src/hub.mjs:230-236
```

### 3.3 数据采集链

```text
hub metrics / daemon 每日 10 点后
  └─ collectMetrics(ep)
       └─ collectors[platform]                       src/metrics.mjs:15-77
            └─ drv.withTab(平台后台 URL)
                 └─ CDP Runtime.evaluate
                      ├─ 页面 DOM 文本/React fiber
                      └─ fetch(内部接口, credentials:'include')
  └─ saveMetric(day, episode, platform, m)           src/db.mjs:52-58
  └─ 飞书日报 / boardSync

hub followers
  └─ followerCollectors[platform]                    src/followers.mjs:16-61
  └─ saveFollowers(...)                              src/followers.mjs:68-77
```

### 3.4 Web 操作链

```text
浏览器 POST /api/action
  └─ 校验口令 cookie / 飞书会话 / CF Access         src/web.mjs:125-130
  └─ 只允许 sync|metrics|doctor|board|followers|mark src/web.mjs:164-168
  └─ spawn(node hub.mjs ...)                         src/web.mjs:172
```

## 4. Chrome 插件耦合点

| 编号 | 耦合点 | 位置 | 严重度 | 说明 |
|---|---|---|---|---|
| C-01 | 发布执行完全在插件内 | `driver.mjs:16-17,42-50` | 阻断 | 本仓库没有 LinkedIn/Facebook 适配器，也没有可替换的 provider 层 |
| C-02 | 校验规则在插件 bundle | `validate.mjs:7`；`make-publish.mjs:21` | 阻断 | 未安装插件时连 `publish.json` 都无法校验 |
| C-03 | 链接回收在插件内 | `driver.mjs:71-74` | 阻断 | 新工作台必须自己实现 `getPostUrl` |
| C-04 | 发布后回读/提交用 DOM | `checks.mjs:26-51,67-97,155-182` | 高 | API 发布后不再需要，但查重逻辑可借鉴 |
| C-05 | 登录态保存在专用 Chrome Profile | `chrome-shim.mjs:7-8,23-31` | 高 | 以 Cookie 代替 Token，不可审计、不可轮换 |
| C-06 | 页面 DOM 选择器随平台改版失效 | `checks.mjs`、`metrics.mjs`、`followers.mjs` | 高 | 维护成本高，且无法保证幂等 |
| C-07 | 商店插件源码与版本不可锁定 | `plugin.mjs:16-21` | 高 | 运行时读取“最新版本”，行为可能漂移 |
| C-08 | 开发桥依赖插件 `tools/` | `bridge.mjs:25,29` | 中 | 仅本地调试用 |
| C-09 | 健康检查探测 CDP 端口 | `web.mjs:53` | 低 | 新架构改探 API/Token 状态 |

## 5. 现有状态与防重

### 5.1 状态集合

`src/db.mjs:28`：

```text
planned / pending / running / filled / awaiting_user /
scheduled / published / failed / not_logged_in / stopped / skipped
```

`LOCKED = filled, awaiting_user, scheduled, published`（`db.mjs:27`）。

### 5.2 与目标状态机的差距

| 目标状态 | 现状 | 差距 |
|---|---|---|
| `draft` | 无 | 内容与任务未分离 |
| `approved` | 无 | 没有审批记录 |
| `queued` | 部分 `planned`/`pending` | 没有队列语义与优先级 |
| `submitting` | 部分 `running` | 不能区分“已发出请求”和“正在等待响应” |
| `processing` | 无 | YouTube 等异步处理无法表达 |
| `published` | 有 | 但依赖 DOM 回读 |
| `failed` | 有 | 不区分可重试/不可重试 |
| `unknown` | **无** | 超时/网络中断后无法安全恢复 |
| `cancelled` | 部分 `stopped` | 没有取消语义 |

### 5.3 防重机制评估

- 本地：`tasks` 主键 `(episode_id, platform)`，靠 `LOCKED` 跳过（`hub.mjs:137-139`）。
- 平台侧：按标题/首行文本在页面查重（`checks.mjs:8-23,54-64,115-151`）。
- 缺陷：
  1. 没有 `payload_hash`，文案改了仍会被当成同一任务。
  2. 没有 `idempotency_key`/`provider_request_id`。
  3. 标题匹配会漏判（平台改名、同标题多期）。
  4. `filled` 也被视为锁定，人工改稿后无法重跑，只能 `--force`，而 `--force` 会绕过所有本地保护。

## 6. SQLite 数据模型

`src/db.mjs:11-23`：

| 表 | 主键 | 用途 | 对新架构的限制 |
|---|---|---|---|
| `episodes` | `id` | 一期内容 | `id` 来自目录名；无产品/活动维度 |
| `tasks` | `(episode_id, platform)` | 平台任务状态 | **无法支持同平台多账号/多帖子** |
| `metrics` | `(day, episode_id, platform)` | 每日指标 | 无法保存多帖子、多账号、原始响应 |
| `samples` | `(ts, episode_id, platform)` | 分钟级采样 | 同分钟冲突；无账号维度 |
| `board_records` | `(tbl, key)` | 飞书记录映射 | 可保留 |
| `site_daily` | `day` | 站点每日注册/会员/订单/收入 | 无询盘维度 |
| `snapshots` | `episode_id` | `publish.json` 快照 | 无版本号/审批状态 |
| `site_campaigns` | `campaign` | 按 campaign 的转化 | 无 `utm_content`、无商品/询盘 |
| `site_channels` | `source` | 按渠道转化 | 只有注册/会员/订单/收入 |
| `meta` | `key` | 守护进程时间戳等 | 可保留 |
| `logs` | 自增 | 操作日志 | 无操作者、无审批、无请求 ID |

**迁移风险：**

1. 没有 `PRAGMA user_version`，也没有迁移脚本；只有 `CREATE TABLE IF NOT EXISTS`。新增列只能 `ALTER TABLE`，且无法回滚。
2. 现有主键与目标“多账号、多帖子、多版本”冲突；必须新增表，不能原地改主键。
3. `metrics` 的 `views` 等字段是跨平台合并后的整数；拆分会改变现有看板/复盘查询。
4. `tasks.detail` 与 `logs.result` 截断到 2000 字符（`db.mjs:47`），不适合存放原始 API 响应。
5. `node:sqlite` 的 `DatabaseSync` 是同步 API；大批量原始 JSON 写入会阻塞事件循环。

## 7. 指标与归因现状

### 7.1 指标

- 统一字段：`views/likes/comments/favorites/shares`（`db.mjs:14`）。
- 平台差异被塞进 `extra` JSON（如 B 站投币/弹幕、视频号推荐）。
- `review.mjs:30-32` 直接把各平台 `views` 相加；`board.mjs:75-77` 同样相加。
- 问题：
  - LinkedIn 的 `impressionCount` 与 YouTube 的 `views` 不是同一概念。
  - Facebook 的 `post_impressions` 与 X 的 `views` 也不可直接相加。
  - 缺失值用 `0` 或跳过，无法区分“真的 0”和“API 不提供”。

### 7.2 UTM 与转化

- 仅从 `publish.json` 文本里正则提取 `utm_campaign`（`web.mjs:32`；`review.mjs:44`；`board.mjs:45`）。
- 站点接口只返回注册/会员/订单/收入（`site.mjs:5-7`）。
- 没有 `utm_content`、没有商品页访问、没有询价点击/提交、没有有效询盘状态。
- 结论：现有链路只能做“活动级粗略归因”，不能做“内容 → 商品 → 询价 → 有效询盘”的闭环。

## 8. 可保留 / 必须重构

### 8.1 可直接保留（改造成本低）

| 能力 | 位置 | 处理 |
|---|---|---|
| SQLite 持久化与 WAL | `db.mjs:9-10` | 保留，新增表 |
| 守护进程调度骨架 | `hub.mjs:342-430` | 保留循环结构，替换任务执行 |
| 计划扫描 | `hub.mjs:328-339` | 保留，改为读内容状态 |
| 状态变更日志 | `db.mjs:46-49` | 保留，补充操作者/请求 ID |
| 飞书通知 | `feishu.mjs` | 保留 |
| Web 看板 | `web.mjs`、`web/*` | 保留信息架构，增加审批与账号页 |
| 复盘/周报 | `review.mjs` | 保留算法，改用标准化指标 |
| `publish.json` 导入 | `make-publish.mjs` | 保留为导入器，不再作为运行时事实源 |
| faststart 校验 | `validate.mjs:14-22` | 保留 |
| 站点数据接口 | `site.mjs` | 扩展字段 |

### 8.2 必须重构（高成本/阻断）

| 能力 | 原因 |
|---|---|
| 发布执行 | 必须改为官方 API Adapter；插件代码不可维护、不可审计 |
| 发布后核验/提交 | DOM 点击不适用于 API；改为 API 响应 + 查询确认 |
| 链接回收 | 改为 Adapter `getPostUrl()` |
| 指标/粉丝采集 | 改为官方 Analytics API；禁止 Cookie 注入与内部接口抓取 |
| 状态机 | 增加 `submitting/processing/unknown/cancelled` 与幂等字段 |
| 内容模型 | 增加产品身份、来源版本、证据、审批、平台变体 |
| 数据模型 | 新增原始数据、账号、任务尝试、审批、归因表 |
| 认证 | 新增 OAuth2/OIDC/Token 加密存储 |
| 审批 | 新增三层门禁：Ds Bit 技术事实 `RELEASE_APPROVED`、商品 `PUBLIC_ELIGIBLE`、社媒内容 `SOCIAL_CONTENT_APPROVED`，并记录审计 |
| 部署 | 去掉 macOS 专属命令依赖，明确 Linux/容器方案 |

## 9. 运行与部署审计

| 项 | 事实 | 风险 |
|---|---|---|
| 默认模式 | `direct`：中控进程直接加载插件代码 | 插件版本漂移、无沙箱 |
| Chrome | 专用 Profile + 9333 调试端口 | 端口暴露在本机；Cookie 即权限 |
| NAS | `ensureNas()` 使用 `/Volumes`、`diskutil`、`osascript` | Windows/Linux 不可用 |
| 常驻 | `daemon` 每 5 分钟循环 | 单进程；重启后依赖 SQLite 恢复 |
| Web | 监听 `0.0.0.0:8787` | 依赖口令；远程需隧道 + Access |
| Web 操作 | 子进程调用 `hub`，内存态 `actions` | 重启丢失；无持久任务队列 |
| 日志 | 控制台 + `logs` 表 | 无脱敏层；未来不得写 Token |
| 配置 | `config.json` 明文 | 未来密钥必须走 env/KMS |

## 10. 审计结论

1. **中控骨架可复用，执行层必须重写。**
2. **Chrome 插件不是“可选依赖”，而是当前发布、校验、取链接、采集的共同底座；移除前必须逐条替换并做对账。**
3. **现有状态机与数据模型不足以支撑官方 API 的异步、幂等和审计要求，必须增量扩展。**
4. **现有指标口径不能直接用于 Morrishk 的 B2B 决策，必须重建标准化层。**
5. **现有 UTM 只能到 campaign 级，必须扩展到 content/product/inquiry。**
6. **P1 应只做 LinkedIn，并以真实测试 Page 的发布、查询、删除/保留策略和 Token 失效处理为验收核心。**
