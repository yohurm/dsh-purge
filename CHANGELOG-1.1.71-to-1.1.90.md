# dsh-purge 1.1.71 → 1.1.90 更新说明（旧版 → 新版）

> 面向：把 yohurm/dsh-purge 公开旧版（1.1.71）与本地已安装的新版（1.1.90）对齐，并把新版集成进我们的 fork 仓库（HunYiSage/dsh-purge）。

## 0. 事实基础与口径

- **旧版（云端公开仓库）**：`https://github.com/yohurm/dsh-purge`，`master` 最后提交 `8df4406`（2026-10-19），`package.json` version = **1.1.71**。作者在 README 里写明：2026-10-26 后闭源，GitHub 不再发新版，新版本只走 Telegram 频道。
- **新版（本地已安装）**：`~/.dsh/profiles/desktop/{vendor,node_modules}/dsh-purge`，`package.json` version = **1.1.90**，构建文件时间 2026-11-03；安装记录 `installed-rev = apply:1.1.90`。
- **关键事实**：两棵树的 `release-notes.md` **内容完全一致**（都只写到 `# 1.1.71`）。也就是说 1.1.72–1.1.90 这 19 个版本**官方没有留下更新日志**。因此下面的更新内容是通过 **源码逐文件对比 + 代码内联 issue 编号/注释** 反向整理出来的，不是照抄 changelog。
- 新版代码注释里直接自证：`lib/anchor.js` 写「两棵树（本地 1.1.75 / 云端 1.1.71）的 context 形状一致」，`lib/profile-insert.js` 引用 `@1.1.81` 版本键 —— 说明 1.1.75、1.1.81 都是本地私有迭代版本，最终收敛到 1.1.90。

## 1. 一句话结论（版本跨度带来什么）

1. **宿主适配前移**：从 dsh `0.2.0-rc.2` 直接适配到 `0.2.1-rc.2`（并声明兼容 `0.2.1-rc.5`；README 明示 `0.2.1-rc.3+` 不支持）。
2. **许可证与主体换血**：从作者「商业软件许可」改为 **Apache-2.0**，版权主体改为 **dsh-purge maintainers / HunYiSage**，README 从中文第一人称改为「维护者团队」英文优先。
3. **新增 9 个功能模块**，删除 4 个历史文件，约 100 个文件被改写（含 3 个上十万字节的 `client.js / core.js / redteam/*` 大文件）。
4. **修掉一批硬伤**：Web 自动重启留下孤儿服务进程导致 `dsh web` 起不来（#197）、CLI 应用不打应用戳导致重复清洗（#202）、0.2.1 组装管线过滤导致旧补丁全部失配（#132/#142）等。

## 2. 宿主适配与版本管理

- `lib/core.js` 新增导出：`HOST_TARGET`、`HOST_TARGETS`、`parseHostVersion`、`readHostVersion`、`hostCompat`，统一做宿主版本判定与兼容性映射。
- `bin/dsh-purge.js` 的 `status` 增加：宿主版本行 `宿主版本 / host <ver>  ← 未适配：插件声明 <target>`；补丁状态新增 `!`（重编译漂移 drifted）。
- `package.json`：`dshTarget: 0.2.0-rc.2` → `dshTarget: 0.2.1-rc.2` + `dshTargets: ["0.2.1-rc.2","0.2.1-rc.5"]`。
- **#132 / #142**：0.2.1-rc.2 在 `assemble` 之后新增 `requiredContexts` 过滤，旧的注入 pattern 一条都命中不了 → 重写 pattern 以在新管线里继续生效。
- client 注入面新增 `@deepseek-ai/dsh-client-ui-conversation`（配合会话区 UI 改造）。

## 3. 打包 / 许可 / 元数据

| 项 | 旧（1.1.71） | 新（1.1.90） |
|---|---|---|
| `version` | 1.1.71 | **1.1.90** |
| `license` | `SEE LICENSE IN LICENSE`（商业许可，首行「dsh-purge 商业软件许可 / 保留所有权利 / 不是开源软件」） | **`Apache-2.0`**（LICENSE 全文替换为 Apache License 2.0） |
| `author` | 无 | **`dsh-purge maintainers`** |
| `repository / homepage / bugs` | `YuJunZhiXue/dsh-purge` | **`HunYiSage/dsh-purge`** |
| `files` | 含 `skills`、`README.en.md`、`LICENSE-COMMERCIAL.md` | 移除 `skills`、`README.en.md`、`LICENSE-COMMERCIAL.md`；新增 `NOTICE` |
| `scripts` | `embed-prompt`、`build:client` | 新增 **`postinstall: ensure-profile-bundle.mjs`**（安装即把本体 insert 写进用户层） |
| 安装源 | YuJunZhiXue archive | **codeload HunYiSage tar.gz / zip / git** |

> 注意两处打包不一致（本次集成一并修）：`package.json` 与 README 都要求保留 `NOTICE`，但 1.1.90 压缩包里**实际没有 `NOTICE` 文件**；README 提到 `README.ja.md` 但仓库内也没有该文件。集成时补了一份标准 Apache `NOTICE`，`README.ja.md` 作为已知缺口记录。

## 4. 新增模块（9 个）

| 文件 | 职责 |
|---|---|
| `lib/anchor.js` | 尾部锚点。注入正文落在 system prompt 最前，末尾再补一小段把「身份 + 格式锁」放回 recency 位；主锚用 section（每轮重渲染），子锚用 context（排在宿主 `subagent:delegation` order 120 之后抵消「审批自动拒绝」提示）。自带跨版本子会话判定，不 import 仓库函数。 |
| `lib/subagent-control.js` | 子代理审批接管。宿主给子会话写死 `approvalPolicy: "never"`（`decide()` 直接 `return "rejected"`），本模块先把 policy 翻成 `ask`，再在 waterfall 里放行；只作用于子会话，主会话零影响。 |
| `lib/apply-progress.js` | Apply/还原/卸载的**内存态进度**，只给 `/dsh-purge/status` 轮询，不落盘。 |
| `lib/profile-insert.js` | 用户层 `profile/cordis.patch.yml` 的本体 insert（#154/#159/#191）。包内 overlay 只改已入队行的 inject/config；本模块负责「行不存在就写一条」，并能识别文件本身是完整文档（空 flow `[]`）时替换而非追加。 |
| `lib/bin-preflight.js` | #197：把「启动前回收残留 web server」钩子挂到宿主 `bin.js` 顶部，必须早于 Cordis 激活（0.2.x 里 webserver 绑端口失败是 fatal）。 |
| `lib/web-preflight-entry.js` | 被复制进宿主 lib 并由 `bin.js` import 的 sidecar；Cordis 激活前回收占用端口的 `dsh web` 孤儿进程。 |
| `lib/web-reclaim.js` | #197 的共用实现：pid 记录文件 `$DSH_HOME/dsh-purge/web-server-<port>.json`、按端口反查监听 pid（win netstat / *nix lsof）、只杀 `@deepseek-ai/dsh/lib/bin.js … web` 自己人。 |
| `lib/redteam/atomic-write.js` | 先写临时文件 → fsync → rename 的原子写，避免写一半崩溃留下半截 JSON 把用户设置吞掉。 |
| `lib/redteam/evidence-ledger.mjs` | 红队证据账本（借鉴 REA Evidence 思想）：`claimed/observed/derived/inferred` 四档 + `ev_<sha256>` 内容寻址 + `ResidualUnknown`，基于 `node:sqlite`。 |

## 5. 主要修复 / 特性（按主题）

### 5.1 Web 自动重启孤儿进程 / EADDRINUSE（#197）
- 触发：Apply→自动重启用 detached 子进程接管 server，父进程 250ms 后退出；关窗/关终端杀不到它，下次 `dsh web` 在 webserver 绑端口时撞 `EADDRINUSE`，0.2.x 把 required 插件激活失败当 fatal，宿主直接起不来。
- 修复：新增 `web-reclaim.js` + `bin-preflight.js` + `web-preflight-entry.js`，在 Cordis 激活前按记录文件/端口回收；`restart-web.js`、`web.js`、`desktop.js`、`core.js` 同步接入。

### 5.2 Apply 状态、补丁漂移与应用戳（#202、drifted）
- 面板点「应用」会打应用戳（applied stamp），但 **CLI 路径没打**，下次启动 `settle` 以为没对齐又整轮重打一遍（#202）→ CLI 补 `reapply.markPatchesApplied()`。
- 新增补丁状态 **`drifted`（漂移：官方原文已改、pattern 需更新）**，`filePatchOutcome`、`rollupPatchStatus`、`committedOverrideRevision` 等导出统一状态计算。
- Apply 完成时若仍有失败补丁，CLI 打印「完成但有 N 个补丁失败」并以退出码 1 结束，不再假装全部成功。
- 命令行卸载补上「删除插件目录」步骤（界面卸载交给独立助手等端口释放后再删）。

### 5.3 身份 / 提示词装配（#204、#29、#194、#162）
- **#204**：优先把真实 cwd 字面量写进 `assembly.variables`，这样 0.2.1 即使不注册 `{{cwd}}` 也能保留工作目录提示（`resolveCwdFromContext`、`cwdVariableSupported`）。
- **#29**：不再把上一条 system/message 全文钉回 sections。
- **#194**：宿主主线程同步调用装配函数，一次全量 resolve 被磁盘/解密拖住会卡桌面宿主 → 增加超时熔断 + 仅在正文真变时重算。
- **#162**：逐命令注册，撞名只丢那一条并可退到备用名。
- 新增 `peelChatmlEnvelope`、`normalizeChatmlLayout`、`stripChatmlTags`、`escapeDshInterpolation`、`adaptInjectForMinimalPreset`、`modePromptText`、`overlayOfficialIdentity`、`splitRedteamInject`、`stripRegionLecture`、`withoutCtfContract`、`withoutRedteamLeadIdentity` 等，处理 ChatML 信封剥离、最小预设适配、区域说教清理、CTF 契约注入等。

### 5.4 用户层 profile 自愈 insert（#154 / #159 / #191）
- 见 `profile-insert.js`。`postinstall` 与演练台子行都走同一处：行不存在就写一条；避免 Hub 已入队再 insert 报 `already queued`；`[]` 完整文档时整体替换，绝不追加成第二个 YAML 文档（否则宿主启动解析失败）。

### 5.5 子代理审批接管 + 尾部锚点
- 见 `anchor.js` / `subagent-control.js`。配合 `presets/redteam/anchor-gate.mjs`（更新）。

### 5.6 桌面 / 社区 Electron 宿主识别（#133 + surface.js）
- `surface.js` 新增 `isCommunityElectronHost`：识别 `DS Bot / DeepSeek Bot / dsh next` 等社区 Electron（profile 可能是 `ds-bot`，`DSH_SURFACE` 还可能被写成 `web`），扩展 `dsh-desktop-next`、`dsh next.exe` 等判定。
- **#133**：换包助手不能再用 Electron 本体跑（`Stop-HarnessInstall`/`taskkill` 按 exe 名会先把自己杀掉，导致 `app.asar` 永远挪不开）→ 改用非 Electron 进程；`restart-desktop.js` 把助手自身排除出待杀名单；`desktop.js` 增加 `ASAR_SWAP_PS1`、`desktopCliEntryPatched`。

### 5.7 文件权限 / 软链 / 解包（#155、#157、#119、#150、#118）
- **#155/#157**：写文件会变成 0644 导致后续 Permission denied → 写前记原权限、写完补回；apply 曾把软链 launcher 覆盖成 `bin.js` 拷贝 → 能找到官方入口就还原软链。
- **#119**：补同步版实现，避免漏 `await` 变未处理 rejection。
- **#150**：`extract-asar` 处理 fd（数字）不能被当路径做 asar 探测。
- **#118**：`file:<本地 tar>` 是归档不是目录，按目录扫会 `ENOTDIR`。

### 5.8 Office 技能运行时自愈
- `core.js` 新增 `healOfficeSkillsRuntimeOnDisk`、`officeSkillAssetSources`；`desktop.js` 新增 `healOfficeSkillsAt` / `healOfficeSkillsRuntimeOnDisk`：安装/更新后自愈 Office 技能运行时资产。

### 5.9 网络范围 / 作弊任务闸门（net-scope.js）
- 新增 `isCheatTask`、`ownServerNote`、`publicHostHit`、`execArgText`、`latestToolText`：开源版把攻击任务限制在 loopback / RFC1918 私网，域名先解析、任一解析地址为公网即拒绝；并显式拦截外挂类（自瞄/透视/改内存/反作弊绕过）。

### 5.10 红队演练台（lib/redteam）
- 新增 `evidence-ledger.mjs`（证据账本）、`atomic-write.js`（原子写）。
- `tools.js`（+约 10KB）、`store-core.js`、`client.js`、`index.js`、`schema.js`、`skill-fixes.js` 等大改：证据账本接入、技能可用性/修复、平台配置、评分规则、报告回放更新。
- `presets/redteam/agent.cordis.yml` 更新（本地留有一份旧 `.bak`，集成时剔除）。

### 5.11 客户端 UI（client.js）
- **#208**：undo 按钮不再注册进 `conversation.input.right`。
- **#169**：工具条按钮继承图标字体/font-size:0/透明色导致「字在 DOM 但看不见」的修复。
- **#196**：暗色宿主 `--dsw-alias-brand-primary` 解析为近白色导致对比度问题。
- **#347**：`dusk` 主题主按钮 hover 配色。
- **#121**：`session-controller` 普通发送走 followup、插话 steer 仍打断当前步（Ctrl+Enter 不能改成 followup）。
- rewind / 撤回交互修正。

### 5.12 其它核心修复
- **#207**：dir 模式静默退出时挂生命周期日志，记录真实退出码与原因。
- **#61/#59**：web 自愈重启不再弹浏览器、关窗后不被反复拉起新窗口。
- **#80**：hooks deny/ask 旁路开关与官方形态反向写回。
- **#74/#87**：`reapply` 补丁 marker 重打，避免 agent-loop 文件被整体替换后 hang-inject 全路径哑火。
- **#32**：注释标记与 #53 在 rc.3+ 的文件键适配。

## 6. 文件变更清单（摘要）

**新增（9）**：`lib/anchor.js`、`lib/apply-progress.js`、`lib/bin-preflight.js`、`lib/profile-insert.js`、`lib/subagent-control.js`、`lib/web-preflight-entry.js`、`lib/web-reclaim.js`、`lib/redteam/atomic-write.js`、`lib/redteam/evidence-ledger.mjs`

**删除（4）**：`docs/修改文档.md`、`docs/注入方式改造方案-1.1.34化.md`、`lib/default-prompt-inject.md.bak`、`lib/mnemon-settings.js`

**修改（49 个，重点）**：`lib/{core,index,identity,desktop,net-scope,reapply,rules,skills,update,extract-asar,restart-desktop,restart-web,web,surface,host-clean,hide-console,preset-default,capability-protocol}.js`、`lib/redteam/{client,index,tools,store-core,schema,score-rules,platform-config,preset-composition,report-replay,role-section,skill-availability,skill-fixes,validate,ui,settings,ip-utils}.js`、`client.js`、`bin/dsh-purge.js`、`package.json`、`cordis.patch.yml`、`README*.md`、`screenshots.json`、`scripts/*`、`skills/redteam/*`、`LICENSE` 等。

> 精确的逐文件 `+/-` 行数与完整清单见仓库提交说明末尾附录（`git diff --stat 8df4406..HEAD`）。

## 7. 集成方式与验证

- 集成基线：以本机已安装的 **1.1.90 vendor 树**为准覆盖到 fork（`master` 分支），保留上游 `8df4406`（1.1.71）完整 git 历史，新增一个「集成本体」提交，便于回溯与回滚。
- 排除本地临时文件：`presets/redteam/agent.cordis.yml.bak-20261105-091738`（本地编辑备份，不入库）。
- 补齐打包缺失：新增 `NOTICE`（Apache-2.0 署名）。
- 验证：`node --check` 全部 `lib/**/*.js` 与 `bin/dsh-purge.js` 通过；`package.json` 可解析；文件清单与 1.1.90 一致。
- 回滚：`git reset --hard 8df4406` 回到 1.1.71；或使用本地备份 `~/.dsh/dsh-purge-fix-backup-20261020` 与 `vendor` 现有副本。

---

## 附录：本次集成文件变更（git name-status，共 64 个路径）

### 新增 11

- `CHANGELOG-1.1.71-to-1.1.90.md`
- `NOTICE`
- `lib/anchor.js`
- `lib/apply-progress.js`
- `lib/bin-preflight.js`
- `lib/profile-insert.js`
- `lib/redteam/atomic-write.js`
- `lib/redteam/evidence-ledger.mjs`
- `lib/subagent-control.js`
- `lib/web-preflight-entry.js`
- `lib/web-reclaim.js`

### 修改 49

- `LICENSE`
- `README.md`
- `README.zh-CN.md`
- `bin/dsh-purge.js`
- `client.js`
- `cordis.patch.yml`
- `docs/banner.svg`
- `docs/drill-auth-legal.html`
- `lib/asset-table.js`
- `lib/capability-protocol.js`
- `lib/continue-retry.js`
- `lib/core.js`
- `lib/default-prompt-inject.en.md`
- `lib/default-prompt-inject.zh.md`
- `lib/desktop.js`
- `lib/extract-asar.js`
- `lib/hide-console.js`
- `lib/host-clean.js`
- `lib/identity.js`
- `lib/index.js`
- `lib/net-scope.js`
- `lib/official-update.js`
- `lib/preset-default.js`
- `lib/reapply.js`
- `lib/redteam/client.js`
- `lib/redteam/index.js`
- `lib/redteam/platform-config.js`
- `lib/redteam/report-replay.js`
- `lib/redteam/role-section.js`
- `lib/redteam/score-rules.js`
- `lib/redteam/settings.js`
- `lib/redteam/skill-fixes.js`
- `lib/redteam/store-core.js`
- `lib/redteam/tools.js`
- `lib/redteam/ui.js`
- `lib/redteam/validate.js`
- `lib/restart-desktop.js`
- `lib/restart-web.js`
- `lib/rewind.js`
- `lib/rules.js`
- `lib/skills.js`
- `lib/surface.js`
- `lib/uninstall-restart.js`
- `lib/uninstall.js`
- `lib/update.js`
- `lib/web.js`
- `lib/zip-extract.js`
- `package.json`
- `presets/redteam/agent.cordis.yml`

### 删除 4

- `docs/修改文档.md`
- `docs/注入方式改造方案-1.1.34化.md`
- `lib/default-prompt-inject.md.bak`
- `lib/mnemon-settings.js`
