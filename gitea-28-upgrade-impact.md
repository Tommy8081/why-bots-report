# Gitea 1.23.5 → 28.4.0 升级影响清单

> 面向：自建 Gitea，已有仓库、PR 历史、Actions/CI 接入。
> 依据：Gitea 28.0.0 官方发布说明（breaking changes）与官方升级文档（28 版）。1.24–1.27 各自的破坏性变更需按第 7 节逐版本核对。

---

## 0. 一句话结论

**数据基本不动，配置和 CI 语义要动。** 仓库、PR、issue、评论、review、标签、wiki、LFS、包、SSH 公钥都会原样保留（数据库自动迁移）；真正需要改的是 `app.ini` 里几组键、反向代理的 WebSocket、以及 Actions 工作流里少数写法。唯一会"丢"的东西是 **Actions 运行历史超过 400 天的部分**（含作业、日志、产物），默认在升级后第一次午夜清理时删除。

---

## 1. 完全不受影响（放心）

| 东西 | 说明 |
| --- | --- |
| 仓库与文件内容 | 存在 git 目录里，升级不动 |
| PR / issue / 评论 / review / 审批记录 | 存在数据库，迁移只改结构不改数据 |
| 合并记录、protected branch 规则、CODEOWNERS 文件 | 保留；规则本身新增了"必须 code owner 批准"可选开关，不打开即维持原样 |
| 标签、里程碑、看板、wiki | 保留 |
| LFS 对象、包仓库、release 附件 | 保留 |
| SSH 公钥、令牌（PAT/OAuth2 授权） | 保留（实现细节变化见 5.4 / 5.5） |
| 用户、组织、团队、协作者 | 保留（生效权限可能变化，见第 6 节） |

---

## 2. 会消失 / 会被改动的数据

### 2.1 Actions 运行历史（默认保留 400 天）

- 新增 `[actions] RUN_RETENTION_DAYS`，默认 **400**。
- 新增定时任务 `cleanup_action_runs`（默认午夜执行），删除超过保留期的 run，**作业、日志、产物一并删除**，无法自动恢复。
- **`0` 的含义改了**：`RUN_RETENTION_DAYS` / `LOG_RETENTION_DAYS` / `ARTIFACT_RETENTION_DAYS` 设 `0` 现在是"永久保留"；1.27 及以前设 `0` 会被当成删除。
- 想原样保留全部历史，**升级前**先写：

```ini
[actions]
RUN_RETENTION_DAYS = 0
LOG_RETENTION_DAYS = 0
ARTIFACT_RETENTION_DAYS = 0
```

### 2.2 已弃用的配置项

控制面板里挂了很久的"配置已弃用"警告，在 28 里可能直接导致**启动失败**（官方文档：未解决的弃用告警可能让下个版本拒绝启动）。升级前把 `/admin` 顶部告警全部清掉。

### 2.3 自定义模板

改过 `custom/templates` 的，模板结构在五个大版本里动过，升级后可能出现 50x、组件缺失、样式错乱。先备份 custom 目录，出问题先整体移走再逐个恢复。

---

## 3. 必须改的配置（app.ini）

### 3.1 出口规则（最大的一处不兼容）

Git 的网络操作（迁移、镜像、检查远程 wiki 等）现在**走内部代理**，在连接时应用出口策略。

```ini
[security]
EGRESS_MODE       = strict          ; 缺省 lax
ALLOWED_HOST_LIST = github.com,gitlab.com,private,loopback
BLOCKED_HOST_LIST = 1.2.3.4
```

要点：

- `external` 预设被删除。
- 默认 `lax` 模式下，`[security] ALLOWED_HOST_LIST` **不再限制公网主机**——只是黑名单 + 私有网段限制。原来靠它做白名单的，必须显式写 `EGRESS_MODE = strict`，否则等于放开了 webhook / OAuth2 的出站目标。
- 只写域名的条目，在 strict 下**只放行 80 / 443**；内部 Git 若在 3000 端口，要写 `10.0.0.15/32:3000`。
- IP 条目不再支持通配符，`*` 不是合法条目。
- 域名按 curl 语法：`example.com` 含所有子域；`*.example.com` 只含子域；`example.*` 非法。
- `[migrations] ALLOWED_DOMAINS` / `BLOCKED_DOMAINS` / `ALLOW_LOCALNETWORKS` 已被 `[migrations] ALLOWED_HOST_LIST` / `BLOCKED_HOST_LIST` 取代；`BLOCKED_HOST_LIST` 写错**直接启动失败**。

对照表：

| 旧键 | 新键 |
| --- | --- |
| `[migrations] ALLOWED_DOMAINS` | `[migrations] ALLOWED_HOST_LIST` |
| `[migrations] BLOCKED_DOMAINS` | `[migrations] BLOCKED_HOST_LIST` |
| `[migrations] ALLOW_LOCALNETWORKS = true` | `ALLOWED_HOST_LIST` 里加 `private,loopback` |
| `[migrations] ALLOWED_DOMAINS = external` | 删除；用 `EGRESS_MODE = strict` + 列表 |

### 3.2 实例域名

- `[server] DOMAIN` **不再被读取**，实例域名与默认 SSH 域名一律来自 `ROOT_URL`。
- 生成的克隆地址、webhook 里的链接、邮件里的链接都可能跟着变，升级前确认 `ROOT_URL` 是你想要的那个。

### 3.3 注册

- 自助注册默认关闭。要开必须显式 `[service] DISABLE_REGISTRATION = false`。

### 3.4 反向代理（不配就降级，不会挂）

- 通知从 SSE（`/user/events`）改为 **WebSocket `/-/ws`**。
- nginx 缺 `proxy_http_version 1.1` 会一直 `426 Upgrade Required`，页面自动退回轮询（能用，但通知不再实时）。
- 多进程部署需要 `[websocket] PUBSUB_TYPE = redis` + Redis。
- `[ui.notification] EVENT_SOURCE_UPDATE_TIME` 已移除。

验证：

```bash
curl -s --max-time 5 -o /dev/null -w '%{http_code}\n' --http1.1 \
  -H 'Connection: Upgrade' -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' \
  https://你的域名/-/ws
# 101 = 通了；426 = 代理没转发 upgrade
```

### 3.5 Git 版本

- 要求 **Git ≥ 2.25.0**，低了直接起不来。

### 3.6 可选新配置

```ini
[audit]
RECORD_OUTPUT  = database     ; 审计日志默认关闭，开了才记
RETENTION_DAYS = 30           ; 0 = 永久

[redis]
CONN_STR = redis://127.0.0.1:6379/0   ; 统一给 cache/session/queue/lock/websocket 用

[actions]
ARTIFACT_PREVIEW_MAX_SIZE = 10   ; MiB，0 关预览，-1 不限制
```

---

## 4. CI/CD（Gitea Actions）需要改什么

### 4.1 工作流写法（会真的报错/行为变）

| 变化 | 影响 | 处理 |
| --- | --- | --- |
| 作业级 `if:` 在矩阵展开**之前**求值，且只能用 `github` `gitea` `needs` `vars` `inputs` 上下文 | 用 `matrix.xxx` 写作业级 `if` 的会失效/报错 | 把条件挪到 `strategy.matrix.include/exclude` 或步骤级 `if:` |
| `strategy.fail-fast` 现在真正生效 | 以前一个矩阵作业失败其余继续，现在会被取消 | 需要全跑完的加 `strategy: fail-fast: false` |
| 矩阵组合数 > 256 直接拒绝 | 超大矩阵的构建会失败 | 拆分或减少组合 |
| 公共仓库不能调用私有仓库的可复用工作流；嵌套调用不能放大令牌权限 | 依赖这种"借权限"的流程会失败 | 调整引用方式 |
| 无效工作流文件现在会生成一条失败记录 | 以前静默 | 无（更清楚） |

### 4.2 Fork PR 的审批门（安全修复带来的行为收紧）

- 取消后再重跑一个"等待审批"的 fork PR 运行，会被视为**显式批准**（并记录批准人）。
- 批准不再重启已经取消的作业。
- 未批准的 fork 运行不会再取消同 concurrency 组的可信运行。
- 审批检查现在同时看**事件触发者**和 **PR 作者**：维护者触发的 PR 事件也会拦住不可信 fork 的工作流。

→ 如果你们的 CI 依赖"维护者手动触发 fork PR 就跑"，升级后可能停在等待审批。

### 4.3 运行器

- 建议 runner 升到 4.1.0（2026-10-01），新增 Kubernetes 后端。
- 旧 runner 一般仍可用，但上面的审批/并发语义由服务端决定。

### 4.4 新增能力（不用改，只是能用）

构建队列视图（管理员全局 + 仓库级）、运行列表自动刷新、产物在线预览（文本/图片/PDF/HTML，默认 ≤10 MiB）、动态矩阵（由上一步输出生成）、`strategy.max-parallel`、`$self` / `$/` 复用引用、强制取消运行 API、运行日志 API。

### 4.5 外部 CI（Jenkins / GitLab / 自研勾子）走的是 webhook + API

- webhook 出站受 `[security] EGRESS_MODE` 管（见 3.1）。
- 仓库改名现在也会触发 `repository` 事件（新增），接收方若按"仓库名不变"假设处理会收到额外事件。
- 新增 API：项目看板、运行管理、包列表；`/repos/{owner}/{repo}/commits/{ref}` 与 GitHub 兼容。
- 没有删除现有 API，旧调用应继续可用。

---

## 5. 外部集成逐个看

### 5.1 镜像仓库 / 从外部迁移

走内部代理 + 出口策略，strict 模式下目标必须在白名单里，端口要写清楚。列表写错会启动失败。

### 5.2 Webhook

`[security] EGRESS_MODE = strict` 时目标主机必须在白名单；lax 模式下私有/环回地址需要单独放行。

### 5.3 OAuth2 提供方 / 登录

出站同样受 `[security]` 出口策略约束。

### 5.4 SSH

- 内置 SSH 服务改为**按指纹**查公钥（修复大小写变体 RSA 伪造），你自己生成、正常注册的密钥不受影响。
- 若有人手工往数据库塞过非标准格式的公钥，需重新添加。

### 5.5 令牌与部署凭据

- PAT 现在可以**原地轮换**（同名同权限换值），泄露时不用删了重建。
- 新增仓库级 **HTTPS 部署令牌**（只对一个仓库，读或读写，可当 Git/LFS 的密码用）。
- **用部署密钥推送时，钩子以"部署凭据"自己的身份和权限运行**，受 protected tag / push option 权限检查约束。以前部署密钥"等于仓库主人"的用法会失效 → 依赖它绕过保护规则或推 protected tag 的脚本要改。

### 5.6 Bot 账号

`gitea admin user change-type` 可把本地账号转成 bot 或转回。bot 只能 token 认证、不能交互登录、不收通知/邮件。给 CI 用的服务账号可以换成 bot，安全性更好。

### 5.7 二次验证

新增 `disable-2fa` 命令行（管理员给对方关 2FA 用）。

---

## 6. 权限模型：升级后要抽查

几处安全修复改变了**生效权限**，不是数据丢失，而是"以前默认宽松、现在严格"：

- 通过 API 降低团队权限时，以前旧的按单元权限会残留 → 现在会被真正撤销（成员可能突然失去不该有的访问，也可能保住本该撤销的）。
- 团队访问改为继承所有单元（`inherit team access for all units`）。
- 团队访问、仓库删除、解绑包都强制做**仓库范围**授权校验。
- 失败的仓库转移不再保留接收方的临时读权限。
- 部署密钥推送不再继承仓库主身份（见 5.5）。

→ 升级后抽查：私有仓库的可见范围、团队对 packages/Actions 的权限、协作者名单。

---

## 7. 你跨了 1.24 / 1.25 / 1.26 / 1.27，各自的破坏性变更要单独过一遍

Gitea 官方把 breaking changes 写在各版本发布说明里，没有一份"跨版本汇总"。**升级前必须逐条读**：

- 1.24.0：https://blog.gitea.com/release-of-1.24.0/
- 1.25.0：https://blog.gitea.com/release-of-1.25.0/
- 1.26.0：https://blog.gitea.com/release-of-1.26.0/
- 1.27.0：https://blog.gitea.com/release-of-1.27.0/
- 28.0.0：https://blog.gitea.com/release-of-28.0.0/

重点看每篇里的 "Major Breaking Changes" 小节，通常涉及配置键改名/移除、默认值变化、Actions 语法、API 行为。本文档只覆盖了 28.0.0 那部分（已核对原文）。

---

## 8. 上线前检查表

```text
[ ] 1. 停服，备份：数据库 dump + app.ini + data 目录 + 外部存储（快照更快）
[ ] 2. 记录当前版本与 git 目录校验，写进变更单
[ ] 3. git --version ≥ 2.25
[ ] 4. 清掉 /admin 顶部的配置弃用告警
[ ] 5. 重写出口规则：[security]/[migrations] 的 EGRESS_MODE 与 ALLOWED/BLOCKED_HOST_LIST
[ ] 6. 决定 Actions 保留天数（要全留就 RUN_RETENTION_DAYS = 0）
[ ] 7. 确认 ROOT_URL 正确（DOMAIN 不再生效）
[ ] 8. 确认是否需要开注册（DISABLE_REGISTRATION）
[ ] 9. 反向代理加 proxy_http_version 1.1，多进程加 [websocket] PUBSUB_TYPE = redis
[ ] 10. 换 28.4.0（二进制 / 镜像标签 / 包），确认架构（32 位 x86、gogit 变体已停发）
[ ] 11. 首次启动等数据库迁移跑完（期间服务不可用）
[ ] 12. 验证：/api/healthz、/api/v1/version、启动日志告警、/-/ws 返回 101
[ ] 13. 抽查：私有仓库可见性、团队权限、webhook 投递、镜像同步、一条 CI 流水线
[ ] 14. 观察一个清理周期（午夜）后 Actions 历史是否符合预期
```

---

## 9. 回滚

数据库升级是**单向**的：跨大版本后无法用旧版本二进制启动同一份数据库。回滚只能靠备份恢复，所以第 8 节第 1 条是硬前提。同一 `x.y` 内的补丁版本（如 28.4.0 ↔ 28.4.1）才是可来回换的。

---

## 10. 参考

- Gitea 28.0.0 发布说明（含 Major Breaking Changes 与 20 个 CVE）：https://blog.gitea.com/release-of-28.0.0/
- 官方升级文档（28 版）：https://docs.gitea.com/installation/upgrade-from-gitea
- 配置项速查（含 egress、actions、audit、websocket）：https://docs.gitea.com/administration/config-cheat-sheet/
