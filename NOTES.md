# dsh-archive-manager — DSH 0.2.1-alpha.1 兼容化改造记录

- 上游：`MichengAI/dsh-archive-manager`，包 `@michengai/dsh-archive-manager@1.0.11`
- Fork：`AEmbers/dsh-archive-manager`（`github:AEmbers/dsh-archive-manager`）
- 本地克隆：`C:\Sophia\_compat021\work\dsh-archive-manager`
- 本次版本：`1.0.11` → `1.0.12`
- 提交：`21e0a98 compat: declare DSH 0.2.1-alpha.1 support and ship lib/ in the git tree`（已 push 到 `origin/main`）

## 1. 被拒原因（沙箱实测）

`C:\Sophia\_compat021\reports\survey-20261004-105901.json` 里该包的判定是
`REJECTED-INCOMPATIBLE`，原因是 `peerDependencies` 用的**逐版本白名单**只到 `0.2.0-rc.2`：

```
"@deepseek-ai/dsh-session":"0.1.2-rc.1 || 0.1.5-rc.1 || 0.1.5-rc.2 || 0.1.5-rc.3
 || 0.1.7-rc.1 || 0.1.7-rc.2 || 0.2.0-rc.1 || 0.2.0-rc.2"
```

17 个 peer 全是这个形状，没有任何一支覆盖 `0.2.1-alpha.1`。

## 2. 改了什么

### 2.1 `package.json`

| 位置 | 改前 | 改后 |
|---|---|---|
| `version` | `1.0.11` | `1.0.12` |
| `engines.dsh` | **不存在** | `">=0.1.2-alpha.1"` |
| `dsh.compatibility` | **不存在** | `{ "dsh": ">=0.1.2-alpha.1", "dshReleases": { … } }` |
| `dsh.compatibility.dshReleases` | — | `0.1.2-rc.1` / `0.1.5-rc.1` / `0.1.5-rc.2` / `0.1.5-rc.3` / `0.1.7-rc.1` / `0.1.7-rc.2` / `0.2.0-rc.1` / `0.2.0-rc.2` / **`0.2.1-alpha.1`**，全部 `"compatible"` |
| `peerDependencies`（17 个 `@deepseek-ai/dsh-*` + `cordis`） | 逐版本白名单 | 一律 `"*"` |

`0.2.0-rc.1` 与 `0.2.0-rc.2` 两个键**保留**，`0.2.1-alpha.1` 是**新增**——即两代核心都显式声明为兼容，
不是把旧的换成新的。区间下界取 `>=0.1.2-alpha.1`（本插件历史上支持的最早一线），因此比只写
`>=0.2.0-rc.1` 覆盖更宽，老宿主也不会被自己的声明挡掉。

`peerDependenciesMeta` 与 `dsh.client.inject` **未改动**（`dsh-client-runtime` / `dsh-session-format-catalog` /
`dsh-client-store` 仍标 `optional: true`）。

### 2.2 `.gitignore`：`lib/` 纳入版本控制

`.gitignore` 原本有 `/lib/`。但本包**没有 `prepare` 脚本**（只有 `prepack` / `prepublishOnly`），
`npm`/`pnpm` 装 git 依赖时只跑 `prepare`，也就是说从 GitHub 源码安装**不会**触发构建。
`lib/` 不入库 ⇒ `main: lib/index.js` 解析不到 ⇒ 插件装不起来。

因此把 `/lib/` 从 `.gitignore` 移除并 `git add` 了 11 个构建产物：
`archive-discovery.js`、`archive-experience.js`、`archive-organizer.js`、`client.js`、`contracts.js`、
`index.js`、`plugin-updater.js`、`projcache.js`、`session-repair.js`、`tombstone.js`、`workspace.js`。
（与 `dsh-codekin` 的做法一致——它也是把 `lib/` 入库以便 GitHub 源码安装。）

### 2.3 `CHANGELOG.md` / `CHANGELOG.zh-CN.md`

新增 `## 1.0.12 - 2026-10-04` 条目，说明兼容声明放宽范围与 `lib/` 入库两件事。

## 3. 破坏了什么？（0.2.0-rc.2 → 0.2.1-alpha.1）

按 `TEAM-BRIEF.md` §3 的**被删符号表**逐条查过（`dsh-invariants`、`dsh-experimental-schedule-bundle`、
`StatsPills`、`ToolCallHookContext`、`registerScheduleTools`、`providerForOpenStep`、
`scopedSubjectResolverFor`、`livePresetMounts`、`UseToolCallArgumentsPartial`、`PartialAccumulator`、
`isVisibleAssistantChunk`、`bindToolCallArgumentsPartial`、`ToolCallInjected`、`JoinedPresetMount`、
`standingMountFor`、`serviceForAgent`）：

- **源码里一处都没有引用**（全仓 grep，排除 `node_modules` 与 `.git`）。
  唯一命中是 `package.json` 的 `dsh-invariants` **devDependency** 声明。
- devDependency 不参与安装闸门判定，也不进运行时；且 `0.2.0-rc.2` 这条 devDep 工具链
  （`pnpm install --frozen-lockfile` + `pnpm build`）本来就跑得通，所以**保持原样没动**——
  动它只会给构建引入风险，而构建产物与环境无关（`bundle: false`，`@deepseek-ai/*` 全部外部化由宿主提供）。

### 3.1 单独查证：`dsh-client-runtime`（0.2.1-alpha.1 已不再发布）

确认 `C:\Sophia\_compat021\node_modules\@deepseek-ai\` 下**没有** `dsh-client-runtime`。
本包有两处提到它，逐条查证后判定**不是阻塞点**：

- `src/client.ts:28` —
  `_deepseek_ai_dsh_client_store = require("@deepseek-ai/dsh-client-runtime/client");`
  位于 `try { require("@deepseek-ai/dsh-client-store") } catch { … }` 的 **catch 分支**里，
  是给 DSH ≤ 0.1.1 的兼容回退（注释原文："DSH <= 0.1.1 owns the store engine in client-runtime"）。
  0.2.1-alpha.1 有 `dsh-client-store`，`try` 成功，这行**不会被执行**。
- `src/client-types.ts:13` — 仅在 `interface ClientModules` 里做类型声明
  `"@deepseek-ai/dsh-client-runtime/client": typeof import("@deepseek-ai/dsh-client-store");`。
  `tsc --noEmit` 实测无报错。

两处都保留原样（保留了老宿主支持），并以「构建 + 类型检查通过」和「浏览器半边真被挂载」为证据。

## 4. 验证（沙箱实测，原始命令与输出）

### 4.1 构建

```
> cd C:\Sophia\_compat021\work\dsh-archive-manager
> pnpm install --frozen-lockfile     # Done in 10.3s, exit 0
> pnpm build
$ pnpm typecheck && node scripts/build.mjs && node scripts/check-package.mjs
$ tsc --noEmit
[dsh-archive-manager] 已从 src 生成 Host 与客户端发布产物
ok: package.json
ok: lib/index.js
ok: lib/contracts.js
ok: cordis.patch.yml
ok: lib/workspace.js
ok: lib/projcache.js
ok: lib/client.js
ok: lib/tombstone.js
ok: lib/plugin-updater.js
ok: lib/archive-experience.js
ok: lib/archive-organizer.js
ok: lib/archive-discovery.js
ok: lib/session-repair.js
ok: LICENSE
ok: README.md
ok: README.zh-CN.md
ok: CHANGELOG.md
ok: CHANGELOG.zh-CN.md
package structure OK
```

### 4.2 沙箱 A = DSH 0.2.1-alpha.1（`C:\Sophia\_compat021`，profile `web`，端口 8911）

安装（**没有** `incompatible` 拒绝）：

```
> $env:DSH_HOME="C:\Sophia\_compat021\home"
> node C:\Sophia\_compat021\node_modules\@deepseek-ai\dsh\lib\bin.js plugin --profile web add `
    github:AEmbers/dsh-archive-manager github:AEmbers/dsh-codekin
+ @michengai/dsh-archive-manager  (git+https://github.com/AEmbers/dsh-archive-manager.git)
+ @nath-vikky/dsh-codekin         (git+https://github.com/AEmbers/dsh-codekin.git)
Done in 1m 15.2s using pnpm v11.24.0
```

装上的是 fork 的 `1.0.12`，且安装副本里 11 个 `lib/*.js` 全在。
功能级验证（Lead 的 `verify-client-bundles.ps1`，**服务端 HTML 的客户端模块清单**为准）：

```
boot page: HTTP 200, 50379 bytes
client modules mounted: 67
@michengai/dsh-archive-manager -> True
VERDICT: PASS
report: C:\Sophia\_compat021\reports\clientbundle-20261004-112141.json
boot log: C:\Sophia\_compat021\reports\boot-20261004-112123-fixgate-p8911.log
```

boot 日志的 `--- suspicious lines ---` 里**没有**该插件的激活失败（只有 bili fetch 链的自噪声）。

### 4.3 沙箱 B = DSH 0.2.0-rc.2（`C:\Sophia\_compat020`，profile `fixgate020`，端口 8912）

验证「过渡期」不被自己锁死：Desktop 宿主打包的核心仍是 0.2.0-rc.2。

```
> $env:DSH_HOME="C:\Sophia\_compat020\home"
> node C:\Sophia\_compat020\node_modules\@deepseek-ai\dsh\lib\bin.js plugin --profile fixgate020 add `
    github:AEmbers/dsh-archive-manager github:AEmbers/dsh-codekin
Packages: +8
Done in 1m 8.4s using pnpm v11.24.0        # 同样没有 incompatible 拒绝
```

```
boot page: HTTP 200, 35641 bytes
client modules mounted: 67
@michengai/dsh-archive-manager -> True
VERDICT: PASS
report: C:\Sophia\_compat020\reports\clientbundle-20261004-112313.json
boot log: C:\Sophia\_compat020\reports\boot-20261004-112304-fixgate-p8912.log
```

### 4.4 复现命令

```powershell
# 沙箱 A（0.2.1-alpha.1）
pwsh -NoProfile -File C:\Sophia\_compat021\work\fixgate-boot-verify.ps1 `
  -Root C:\Sophia\_compat021 -Port 8911 -Profile web
# 沙箱 B（0.2.0-rc.2）
pwsh -NoProfile -File C:\Sophia\_compat021\work\fixgate-boot-verify.ps1 `
  -Root C:\Sophia\_compat020 -Port 8912 -Profile fixgate020
```

（该脚本起沙箱 → 从启动日志取 token → **在服务端存活期间**调 `verify-client-bundles.ps1` → 关沙箱。
注意用 `Start-Process -RedirectStandardOutput <file>` 轮询日志取 token，不能直接读
`Process.StandardOutput.ReadToEndAsync().Result`——那会在子进程退出前一直阻塞。）

## 5. 遗留 / 未做

- 未向上游提 PR（按铁律）。
- 未发布到 npm；交付物是 fork 的 GitHub 源码（`github:AEmbers/dsh-archive-manager`）。
- 未改 `C:\Users\Administrator\.dsh\`。
- `repository` / `homepage` / `bugs` 仍指向上游，便于溯源与反馈；如需改指 fork 请告知。
