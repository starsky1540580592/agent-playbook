# DSH 宿主插件与档案运维手册

本文件承接原 AGENTS.md §7 的全部内容，以及 §5 里的工作区 ACL 回滚命令。**做 DSH 插件/档案相关任务时才读它**，其它任务不必读——AGENTS.md 每轮硬注入，所以那里只留每轮都适用的硬规则，一次性配置记录和已装清单搬到这里。

---

## 一、档案目录与快照

- 档案目录 `{{USERHOME}}\.dsh\profiles\desktop`。
- **改配置前先快照**到工作区 `_backups/desktop-profile-<时间戳>-<原因>/`，快照四个文件：`package.json`、`cordis.patch.yml`、`pnpm-workspace.yaml`、`pnpm-lock.yaml`。

## 二、开关插件

- **插件管理器 CLI（`@loveebb/dsh-plugin-manager`）在 desktop 档案上不可用**：它依赖 `dsh --profile <p> --dump-config`，而 desktop 档案拒绝该命令（`profile "desktop" is managed exclusively by the Electron application`）→ `list` / `update` / `disable` 全部失败。**禁用/启用只能手写用户层 patch**。
- 禁用插件：在 `cordis.patch.yml` 追加两行
  ```yaml
  - id: <插件id>
    disabled: true
  ```
  不动 pnpm、可逆（删掉这两行即恢复）。
- **插件 id 不等于包名**，看各包自带的 `cordis.patch.yml`。已确认对照：`@anweat/dsh-browser` → `browser`，`@nanmicoder/dsh-agent-teams` → `agent-teams`，`@anionex/dsh-vision-toolkit` → `vision-toolkit`。
- 注意：`web-ui-pet`（不是包名 `@linxin666/dsh-web-all`）行已加 `disabled: true`，见第六节。

## 三、安装与更新

- 更新插件：`dsh plugin --profile desktop add <包>@<版本>`（会同时把版本钉进 `package.json` 的 `dependencies`）。
- **`dsh plugin add` 只给「新增依赖」自动追加 `dsh.profile.bundles`**；既有依赖升级不会进启用列表（meow-memory、vision-toolkit 都踩过），要手动往 bundles 里补一行。
- **通用网络环境（自 AGENTS.md §3 外移，2026-10-30）**：沙箱受限令牌下 Schannel 拿不到凭据 → PowerShell / curl.exe 的 HTTPS 一律失败；**Python 自带 OpenSSL 正常**（可直连，也可用本机代理 `http://{{PROXY}}`），下载、装包一律走 Python。`cdn.jsdelivr.net` 直连被对端重置（10054 / ECONNRESET），走 7890 代理才通；宿主（Node fetch，不走代理）拉不到 jsdelivr 资源。
- **GitHub 直装插件**（npm registry 404 的包，如 `@dsh-external/dsh-conv-search`）：
  1. 先 `git ls-remote` 测通路——沙箱下 git 默认走 schannel 会报 `SEC_E_NO_CREDENTIALS`；
  2. 用环境变量让 git（含 pnpm 内部 spawn 的）改用 OpenSSL 后端：`GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=http.sslBackend GIT_CONFIG_VALUE_0=openssl`；
  3. `dsh plugin --profile desktop add github:<user>/<repo>`（依赖钉为 `github:<user>/<repo>` 规格）；
  4. tarball 兜底：`codeload.github.com` 经 Python 可达；
  5. 装完核对 `package.json` 的 `dependencies` 与 `dsh.profile.bundles` 都有该包（新依赖会自动追加 bundles，升级才需要手补）。
- **pnpm 11 的构建脚本开关**在 `pnpm-workspace.yaml` 的 `allowBuilds`：缺项会报 `ERR_PNPM_IGNORED_BUILDS`，让每次安装/更新在最后一步失败。当前放行 `@google/genai`、`protobufjs`，其余 false。

## 四、兼容性判定与现状（2026-10-22 快照）

判定某版本是否兼容：看它 `peerDependencies` 里 `@deepseek-ai/dsh-*` 的范围是否含 `0.2.1-rc.2`。

两个常踩的坑：
- **别把 `@deepseek-ai/cordis`（cordis 4.x，另一个包）当宿主版本**；
- 别把 `^` 范围按「同大版本即可」放宽，要收上界。

各插件现状：

| 插件 | 版本 | 结论 |
|---|---|---|
| `@nanmicoder/dsh-agent-teams` | 0.1.26（peer 新增 `0.2.1-rc.2`） | 已解禁，bundles 内，随重启加载 |
| `@anionex/dsh-vision-toolkit` | 0.1.47（peer 含 `0.2.1-rc.2`） | 已启用，bundles 内共 18 个 |
| `@anweat/dsh-browser` | 无兼容版本（latest=next=0.2.1，peer `>=0.2.0-rc.2 <0.2.1-0` 正好排除 rc.2） | 用户拍板保持 `disabled: true`，等作者发兼容版，**不做精确豁免** |

- 豁免命令形如 `dsh plugin allow-version <包>@<版本> --dsh-version 0.2.1-rc.2 --accept-risk`，DSH 会警告可能崩溃/丢数据——**仅记录，未使用**。

## 五、插件市场「索引源」

宿主条目 id 是 **`plugin-market`**（应用级 bundle `@deepseek-ai/dsh-experimental-plugin-market-bundle` 插入，服务端 `@deepseek-ai/dsh-plugin-market`）。

索引列表 = 该条目的 **volatile 配置 `config.indexes`**（`Config.indexes: z.array(url).default([])`），UI 在「插件 → 发现 → 索引源」，保存走 `settings.update(entryId, {indexes})` → 落到**档案用户层 `cordis.patch.yml`**。

手动加源就在 patch 层写：

```yaml
- id: plugin-market
  name: '@deepseek-ai/dsh-plugin-market'
  config:
    indexes:
      - https://cdn.jsdelivr.net/npm/dsh-plugin-catalog/index.json
```

**四个关键约束**：

1. 该条目是**应用级**的，不会出现在档案 `cordis.yml` 合成树里，但用户层 patch 对应用级条目同样生效（`ui-*` 各行就是证明）。
2. **宿主取数用 `fetch(url, { redirect: "error", headers: { accept: "application/json" } })` + 15s 超时**（`@deepseek-ai/dsh-plugin-market/lib/index.js`），**任何 3xx 都直接报 `unexpected redirect`**——所以 unpkg 等会 302 到版本路径的地址不可用（除非写固定版本 `@<ver>/index.json`）。
3. **本网络下 `cdn.jsdelivr.net` 和 `raw.githubusercontent.com` 直连都被对端重置**（宿主报 `fetch failed: read ECONNRESET`），但 jsdelivr 的**其它边缘域名 `gcore.jsdelivr.net` / `fastly.jsdelivr.net` 可用**（同内容、不同 IP 段，宿主同款 fetch 选项实测 200 且不跳转）。
4. **验证某地址宿主能否拉，必须用宿主同款 fetch 选项测**（`{{LOCAL_PATH}}` + `redirect:"error"` + `accept:application/json`）；Python/默认 Node fetch 会跟随重定向从而误判。

档案现配三条（**只换坏掉的两条，`dshopen.org` 原样未动**）：

1. `https://gcore.jsdelivr.net/gh/realguan/dsh-plugin-market@dist/dsh-index.json`（原第 1 条同文件，5989 个）
2. `https://dshopen.org/index.json`（原第 2 条）
3. `https://fastly.jsdelivr.net/npm/dsh-plugin-catalog/index.json`（新增 npm 目录，3213 个）

增删源 = 改这一行的 `indexes` 列表；改完宿主热重载，重开「插件 → 发现」即可。

## 六、已装插件清单（非标准安装的那些）

- **宠物插件**：`@linxin666/dsh-web-all` 会插入 `web-ui-pet`（鲸鱼女孩桌宠），与 `dsh-whale-widget` 余额小鲸鱼都占右下角 → 已给 `web-ui-pet` 行加 `disabled: true`（**保留其 config**：petId/size/right/bottom 都在，随时可恢复）。
- **记忆插件**：**meow-memory 在用**（memory-evolve 已停用，dependency 保留可恢复）。模型文案语言在 `cordis.patch.yml` 的 `- id: meow-memory` 条目里设 `promptLang: 'zh'`；数据落在工作区 `.dsh-meow/`（已 gitignore）。
- **`{{EMAIL}}`**：右下角 DeepSeek 余额小鲸鱼挂件。含 `assets/whale-widget.js` 客户端脚本 + 贴图/音效，`lib/accounting.mjs` 取余额。它**无 peerDependencies**，故不受 0.2.1-rc.2 兼容闸门限制；属 server bundle，**装完要重启 DSH 才生效**。
- **`@smalltailqwq/{{EMAIL}}`**：深海女仆工坊皮肤（两位鲸鱼娘女仆 + 深海蓝蕾丝 + Q 版侧栏）。它是**整页 web 皮肤**而非挂件——`dsh.client.inject: ["@deepseek-ai/dsh-client-ui-layout"]`，`lib/client.js` 约 300KB + `assets/runtime` 约 5.7MB 贴图；`skin.json` 自带 `dshCompatibility: "0.2.1rc2"`、accent `#c5a468`，靠给 body 加 `data-dsh-maid-atelier` 属性 + 注入 CSS 生效。peer 只声明主包 `@deepseek-ai/dsh: >=0.2.0-rc.2 <0.3.0-0`（覆盖 0.2.1-rc.2 ✓），是少数不逐个列 `dsh-*` 的插件。
- **`@dsh-external/{{EMAIL}}`**：对话内文本搜索（Ctrl+F），`github:beijingwahw/dsh-conv-search`，web 客户端插件，`dsh.client.inject: ["@deepseek-ai/dsh-client-runtime"]`，peer 只需 `dsh-storage-domain >=0.0.1-0`，宿主 0.2.1-rc.2 满足。
- v1.0.1 之后新增的插件见记忆库 `dsh插件` 项目的 overview 条目（清单会漂，本文件不重复维护）。

## 六点五、PowerShell 7 并存（2026-10-30）

- **7.6.9 已装**：`{{LOCAL_PATH}} Files\PowerShell\7\pwsh.exe`，机器 PATH 已注册，新终端敲 `pwsh` 即进；开始菜单/Windows Terminal 自动出现入口。与 5.1 **完全并存**（`powershell` 命令仍指 5.1）。
- **DSH 的 `pwsh` 工具仍走 5.1**（System32 路径），未改宿主。混合调用约定：常规走 5.1（脚本纪律已适配），5.1 真报错时显式调 `& '{{LOCAL_PATH}} Files\PowerShell\7\pwsh.exe' ...`。
- **执行策略**：两版均 RemoteSigned；用工具写入的 `.ps1` 可能被当「未签名远程文件」拒绝——绕法：不走脚本文件，直接内联 `-Command` 执行（不受执行策略限制）。
- **下载安装包**：GitHub releases 的 MSI（约 117MB）走 Python+代理下载（`_asr/tmp/` 曾存一份作重装备份）。

## 七、工作区 ACL（回滚手段已不存在）

工作区根目录上有一条经用户批准的 `{{HOST}}\{{USER}}:(F)` 完全控制项，当初是为了修复 Windows 文件权限（否则沙箱无法在本目录写入；仅作用于本目录、不继承、不涉及工作区外）。**该 ACE 目前仍然生效**，工作区内写入正常。

**⚠️ 但它已经没有任何回滚手段了（2026-10-29 实测）**：回滚脚本与备份 JSON 原本在 `{{LOCAL_PATH}}`，该目录已随 D 盘旧工作区清理**一并删除**（`ls` 返回 No such file or directory）。而且 `acl-recovery` 当初就不是 git 仓库、没有版本兜底，所以无法从别处找回。

**原命令仅作历史记录，现在跑会扑空，不要再照抄**：

```
pwsh -NoProfile -File '{{LOCAL_PATH}}' -Path '{{LOCAL_PATH}}' -AllowRoot '{{LOCAL_PATH}}' -Restore '{{LOCAL_PATH}}'
```

**将来若要重建回滚能力**：用 `icacls <工作区> /save <新json> /t` 做一次基线快照，**存进工作区内**（如 `_backups/acl-<日期>/`）而不是工作区外的临时目录——这次丢的根因就是存到了会被清理的地方。
