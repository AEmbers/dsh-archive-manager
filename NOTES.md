# dsh-archive-manager — DSH 0.2.1-alpha.1 兼容化改造记录

- 上游：`MichengAI/dsh-archive-manager`，包 `@michengai/dsh-archive-manager@1.0.11`
- Fork：`AEmbers/dsh-archive-manager`（`github:AEmbers/dsh-archive-manager`）
- 本地克隆：`C:\Sophia\_compat021\work\dsh-archive-manager`
- 本次版本：`1.0.11` → `1.0.12`（兼容声明 + `lib/` 入库）→ `1.0.13`（死引用清理）
- 提交：`21e0a98 compat: declare DSH 0.2.1-alpha.1 support and ship lib/ in the git tree`
  → `3522f57 compat: drop the retired dsh-client-runtime fallback (1.0.13)`（均已 push 到 `origin/main`）

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
| `version` | `1.0.11` | `1.0.12` → `1.0.13` |
| `engines.dsh` | **不存在** | `">=0.1.2-alpha.1"` |
| `dsh.compatibility` | **不存在** | `{ "dsh": ">=0.1.2-alpha.1", "dshReleases": { … } }` |
| `dsh.compatibility.dshReleases` | — | `0.1.2-rc.1` / `0.1.5-rc.1` / `0.1.5-rc.2` / `0.1.5-rc.3` / `0.1.7-rc.1` / `0.1.7-rc.2` / `0.2.0-rc.1` / `0.2.0-rc.2` / **`0.2.1-alpha.1`**，全部 `"compatible"` |
| `peerDependencies`（16 个 `@deepseek-ai/dsh-*` + `cordis`） | 逐版本白名单 | 一律 `"*"`；1.0.13 又删掉了 `@deepseek-ai/dsh-client-runtime` |

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

新增 `## 1.0.12 - 2026-10-04`（兼容声明放宽 + `lib/` 入库）与 `## 1.0.13 - 2026-10-04`
（清除已下线的 `@deepseek-ai/dsh-client-runtime`：不再静默回退，改为具名报错）两条条目。

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

### 3.1 `dsh-client-runtime`（0.2.1-alpha.1 已不再发布）—— 1.0.13 已清除

确认 `C:\Sophia\_compat021\node_modules\@deepseek-ai\` 下**没有** `dsh-client-runtime`。
本包原有两处提到它：

- `src/client.ts:28` —
  `_deepseek_ai_dsh_client_store = require("@deepseek-ai/dsh-client-runtime/client");`
  位于 `try { require("@deepseek-ai/dsh-client-store") } catch { … }` 的 **catch 分支**里，
  是给 DSH ≤ 0.1.1 的兼容回退（注释原文："DSH <= 0.1.1 owns the store engine in client-runtime"）。
- `src/client-types.ts:13` — 仅在 `interface ClientModules` 里做类型声明
  `"@deepseek-ai/dsh-client-runtime/client": typeof import("@deepseek-ai/dsh-client-store");`。

**1.0.12 时我判定它们不阻塞**（`tsc --noEmit` 干净、`try` 在 0.2.1-alpha.1 上成功、浏览器半边真被挂载），
因此当时保留原样。复核后 1.0.13 把它们**删掉了**，理由是这个回退分支在**整段声明支持范围**
（`>=0.1.2-alpha.1`）内**根本不可达**：

- 拆分出来的 `@deepseek-ai/dsh-client-store` 是 **0.1.2 起**才有的，而本包 1.0.8 起就不支持 0.1.0/0.1.1，
  所以「`dsh-client-store` 解析失败」这件事，对一个受支持的宿主来说不会发生；
- 也就是说那条 catch 只会把错误伪装成「加载已下线包」，而不是把真实原因报出来。

改法（而不是「删掉了事」）：

```ts
// src/client.ts —— 1.0.13
try {
  _deepseek_ai_dsh_client_store = require("@deepseek-ai/dsh-client-store");
} catch (reason) {
  throw new Error(
    `@deepseek-ai/dsh-client-store is not available in this host (requires dsh >= 0.1.2-rc.1): `
    + (reason instanceof Error ? reason.message : String(reason)),
  );
}
```

即**把原始错误带出来再抛**（含原始 `reason.message`），而不是回退到一个已下线的包。
同时清掉了：`exports.__test` 里已无用的 `hasSplitClientStore` 标志、`src/client-types.ts:13` 的类型行、
`package.json` 里 `peerDependencies` 与 `peerDependenciesMeta` 的 `@deepseek-ai/dsh-client-runtime`。
测试同步改写（原来的 legacy-fallback 测试断言的就是被删的行为，必然变红），并**新增一条**
「宿主没有 `dsh-client-store` 时必须显式报错、且绝不加载 `dsh-client-runtime`」。

**有意不动**的两处（附理由）：

- `test/helpers/client-store.mjs:24,30` —— 测试基础设施，只在 `dsh-client-store` 解析不到时才走，
  它演练的是测试脚手架，不是发布出去的 bundle。
- `scripts/test-version-matrix.mjs:43` —— 那里的两个 `profile.runtime` 档案是 `0.1.0-rc.8` / `0.1.1-rc.2`，
  **本包 1.0.8 起就已声明不支持**（低于 `>=0.1.2-alpha.1` 下限）。改它属于上游发布矩阵的范围决策，
  且需要联网装 5 个旧宿主才能验证 —— 在此无法验证，故保持原样并在此标注。

顺带修掉的既存不一致：1.0.12 把 peer 白名单改成 `"*"` 之后，仓库自带的那条 manifest 测试
（原第 397 行，断言旧白名单）**其实已经是红的**；1.0.13 一并改正。

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

### 4.5 1.0.13 重验（死引用清理之后，全部重跑）

因为 `lib/` 是**人工提交的构建产物**（不是构建期生成再发布），改源码后必须重建 + 重验，
而且**没有复用 1.0.12 的 commit**（`21e0a98` → 新的 `3522f57`），让证据链对得上。

```
> pnpm build     # exit 0；tsc --noEmit 干净；12 个 lib/*.js；package structure OK
> pnpm test      # tests 270 / pass 270 / fail 0 / duration_ms 121795
```

两侧都用 **GitHub 源码安装**（不是本地 link）重装到 `1.0.13`，再跑：

```
############ 021  (core=C:\Sophia\_compat021, profile=uiverify, port=8921) ############
overlays cleared: 1; settings opened: true; nav probed: 6 [通用设置 / 模型 / 内置插件 / 归档会话 / Agent 预设 / 码灵]
[PASS] archive-manager: #dsham-archive-panel rendered in settings tab "归档会话" with 24 chars of UI text (id=dsham-archive-panel)
console errors: 0, page errors: 0, >=400 responses: 0

############ 020  (core=C:\Sophia\_compat020, profile=fixgate020, port=8922) ############
[PASS] archive-manager: #dsham-archive-panel rendered in settings tab "归档会话" with 24 chars of UI text
console errors: 0, page errors: 0, >=400 responses: 0
```

（这就是「浏览器里真能用」那一层：不是在设置页里找一段字符串，而是**打开设置 → 点「归档会话」这一节
→ 断言 `#dsham-archive-panel` 这个插件自己的面板真被渲染出来**，并且整个过程中 console 0 error。）

额外一侧的 bundle 级复核（端口 8923，profile `uiverify`）：

```
boot page: HTTP 200, 36387 bytes
client modules mounted: 68
@michengai/dsh-archive-manager   clientHalfMounted = True
@nath-vikky/dsh-codekin          clientHalfMounted = True
VERDICT: PASS
report: C:\Sophia\_compat021\reports\clientbundle-20261004-121952.json
```

一键复跑：

```powershell
pwsh -NoProfile -File C:\Sophia\_compat021\work\_uiverify\ui-verify.ps1 -All     # exit 0，两侧都跑
```

### 4.6 写操作端到端（归档 → 恢复往返，0.2.1-alpha.1）

渲染级验证管不到"按下去写没写进去"，所以另跑了一次**真的状态变更**（只在沙箱 profile 里造/删数据）：

```powershell
pwsh -NoProfile -File C:\Sophia\_compat021\work\_uiverify\write-op.ps1 -Root C:\Sophia\_compat021 `
  -Port 8931 -Profile uiverify -Label 021-write-archive   -Action archive
pwsh -NoProfile -File C:\Sophia\_compat021\work\_uiverify\write-op.ps1 -Root C:\Sophia\_compat021 `
  -Port 8931 -Profile uiverify -Label 021-write-unarchive -Action unarchive
```

| | 归档前 | 归档后 | 恢复后 |
|---|---|---|---|
| 「未归档」行数 | 12 | **11** | 12 |
| 「已归档」行数 | 0 | **1** | 0 |
| `home\storages\workspace.json` | 670 B，`archivedSessionIds: []` | **728 B**，`["session-f0160d13-…"]` | 670 B，回到 `[]` |

```
[PASS] one session moved (archive):   unarchived 12 -> 11, archived 0 -> 1, via "归档" + confirm "归档"
[PASS] one session moved (unarchive): unarchived 11 -> 12, archived 1 -> 0, via "恢复"
```

三个互相独立的见证：面板行数动了 / 磁盘上的 `archivedSessionIds` 真写进去了 / 恢复后原样回来。
归档要点两下（面板「归档」+ 弹窗「归档」），恢复只有一下（「恢复」无确认弹窗）—— 两条都读出来记在 `clicks` 日志里。

**0.2.0-rc.2 侧是「无法断言」**：`fixgate020` 这个 profile 的未归档页签本来就是 0 行（空状态「暂无未归档会话。」），
没有东西可归档。另跑 `probe-session-visibility.mjs` 确认**宿主自己的会话界面在该 profile 里同样空**
（整页正文 `MOCK_TURN_2_OK` 出现 0 次）⇒ 是该沙箱 profile 的**数据条件**，不是本插件的缺陷。写路径只在 0.2.1-alpha.1 上走通。

## 5. 遗留 / 未做

- 未向上游提 PR（按铁律）。
- 未发布到 npm；交付物是 fork 的 GitHub 源码（`github:AEmbers/dsh-archive-manager`）。
- 未改 `C:\Users\Administrator\.dsh\`。
- `repository` / `homepage` / `bugs` 仍指向上游，便于溯源与反馈；如需改指 fork 请告知。
- **写路径**：0.2.1-alpha.1 上「归档 → 恢复」往返已端到端验证（§4.6，含磁盘见证）；但 0.2.0-rc.2 侧没测到
  （该 profile 无会话可归档，原因见 §4.6），**不算通过**。真正删除会话、正文检索导出这些写路径也没测。
- `scripts/test-version-matrix.mjs:43` 仍指向 0.1.0-rc.8 / 0.1.1-rc.2 两个已声明不支持的档案（理由见 §3.1）。
