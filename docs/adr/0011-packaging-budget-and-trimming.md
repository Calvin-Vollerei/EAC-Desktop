# ADR 0011：打包体积与裁剪预算

日期：2026-10-18
状态：已接受。取代 6.0.1 的「随包固定版 WebView2」方案。

## 计数批注

本文所有体积数字均为 **2026-10-18 对 `EAC-Release-v6.0.1-20261007/lite` 便携包的实测**，
不是估算。压缩后字节取自 zip 中央目录的 `CompressedLength`，原始字节取自 `Length`。
**任何后续版本的预算必须重新实测，不得沿用本文数字当验收结论。**

## 背景

### 6.0.1 lite 实测规模

| 指标 | 值 |
| --- | --- |
| portable.zip | 536.7 MB |
| Setup.exe | 482.8 MB |
| 解压后原始体积 | 1371.4 MB |
| 文件数 | 35424 |

按顶层目录拆：

| 顶层 | 文件数 | 原始 | 压缩后 | 占 zip |
| --- | ---: | ---: | ---: | ---: |
| `dsh-desktop/` | 35132 | 689.7 MB | 230.4 MB | 42.9% |
| `webview2/` | 257 | 668.7 MB | 294.7 MB | **54.9%** |
| 其余（壳/`dpx`/`sidecar`/皮肤管理器） | 35 | 12.9 MB | 3.9 MB | 0.7% |

`dsh-desktop/` 内部：

| 子目录 | 文件数 | 原始 | 压缩后 |
| --- | ---: | ---: | ---: |
| `node_modules/` | 32320 | 550.0 MB | 168.8 MB |
| `vendor/` | 2694 | 137.7 MB | 60.9 MB |

`vendor/` 明细：`node.exe` 88.2 MB（压缩 33.0）、`kernel/` 19.9 MB（327 个 tgz）、
`pnpm/` 17.8 MB、`npm/` 11.7 MB。

`node_modules/` 前六名：

| 包 | 原始 | 压缩后 |
| --- | ---: | ---: |
| `@deepseek-ai/libreoffice-kit-win32-x64` | 182.0 MB | 65.6 MB |
| `@typescript/typescript-win32-x64` | 27.1 MB | 9.3 MB |
| `node-pty` | 26.8 MB | 7.3 MB |
| `sherpa-onnx-win-x64` | 22.4 MB | 8.3 MB |
| `@img/sharp-win32-x64` | 18.5 MB | 7.9 MB |
| `pnpm` | 17.8 MB | 4.4 MB |

### 问题一：294.7 MB 的 WebView2 随了包，但从未被使用

6.0.1 的配置是：

```json
"webviewInstallMode": { "type": "skip" },
"resources": { ..., "staged-resources/webview2/": "webview2/" }
```

`skip` 表示安装器不处理 WebView2；固定版运行时（`Microsoft.WebView2.FixedVersionRuntime.154.0.4258.53.x64`，
cab 293.7 MB，解压 668.7 MB）被手工拷进 `staged-resources/webview2/` 并映射为资源。

但没有任何代码指向它：

- wry 0.55.1（`src/webview2/mod.rs:343`）调用
  `CreateCoreWebView2EnvironmentWithOptions(PCWSTR::null(), ...)` ——
  第一个参数 `browserExecutableFolder` 传 `null`，语义是「用系统已安装的运行时」。
- 全仓 `WEBVIEW2_BROWSER_EXECUTABLE_FOLDER` / `fixedRuntime` / `browserExecutableFolder` /
  `CoreWebView2EnvironmentOptions` / `msedgewebview2` 均 **0 命中**。
- `installer-hooks.nsh` 无任何 WebView2 相关环境变量或注册表写入。
- 未使用 Tauri 的 `FixedRuntime` 模式（该模式会把路径传进 `browserExecutableFolder`）。

**结论：这 294.7 MB 是死重。** 它是 `stage-resources.mjs` 里嵌在
`if (fullPack 存在)` 块内的 `cpSync(runtime, staged/webview2)` 顺带拷进 full staged 的，
`make-lite-staged.mjs` 的 `LITE_RESOURCE_DIRS` 又保留了 `'webview2'`，于是跟着进了 lite。

### 问题二：近半数文件是运行期零用途的死重

按扩展名统计（全包）：

| 扩展名 | 文件数 | 原始 | 压缩后 |
| --- | ---: | ---: | ---: |
| `.ts` | 8772 | 48.0 MB | 10.1 MB |
| `.map` | 4746 | 47.5 MB | 10.3 MB |
| `.md` | 1504 | 11.6 MB | 4.4 MB |
| `.mts` | 1185 | 4.7 MB | 1.0 MB |
| `.cts` | 166 | 1.6 MB | 0.3 MB |
| **合计** | **16373** | **113.4 MB** | **26.0 MB** |

只占压缩后体积 4.8%，但占**文件数 46.2%**。`node_modules/` 里 16162 / 32405 个文件是这五类。

安装慢的主因是文件数（NSIS 逐文件写入），不是字节数。所以死重文件对安装时长的
伤害远大于它对包体积的贡献。

> 口径说明：上表是**全包**统计，与「排除 dev-only 后的死重」不是同一口径。
> `dev-only` 包内部本身就含 725 个死重文件，两者有重叠。决策 §4 的预算表
> 用的是互斥口径（`deadOutside` 15437 文件），比这里的 16373 少 936，
> 差值即 dev-only 包内部的死重 + `dsh-desktop` 根与 `vendor/` 下的同类文件。

### 问题三：win32 分支没有平台裁剪

`stage-resources.mjs` 有 `if (targetPlatform === 'linux')`（`:562`）与
`if (targetPlatform === 'darwin')`（`:580`）两个裁剪块，各自 `rmSync` 掉不可达的
跨平台 payload 并调用 `pruneLinuxPayloads` / `pruneDarwinPayloads`。

**`win32` 分支只有 `WebView2Loader.dll` 前置检查（`:124`）和装配（`:659`），
没有任何裁剪。**

由于交叉编译被显式禁止（`--target` 必须等于 `process.platform`，见 `:43-46`），
这三个分支各自只在本机平台运行时生效。因此 Windows 产物里若存在 macOS/Linux
专属 payload，没有任何代码会剔除它。

**实测后果**（2026-10-18，win32/x64 源树）：

| 包 | 保留 `win32-x64` | 应剔除的异平台 prebuilds |
| --- | ---: | ---: |
| 合计 | 9 文件 / 12.24 MB | **48 文件 / 15.92 MB** |

其中 `node-pty/prebuilds/win32-arm64` 单目录 **11.22 MB / 6 文件** ——
在 x64 机器上永远不会被加载，却随 Windows 产物一起发出。
另有 `bare-fs` / `bare-path` / `bare-url` 各 12 个异平台目录（含 android、ios、
darwin、linux）。

**注意 `assertNoUnexpectedNativePlatform` 挡不住这一类。** 它（`stage-node-modules.mjs:170`）
按**包名**里的平台 token 判定，而 `prebuilds/` 是**包内子目录**，包名里没有平台词。
实测 `assertNoUnexpectedNativePlatform(nm, 'win32', 'x64')` 返回 PASS —— 这 48 个
异平台 prebuilds 全部漏网。

### 问题三之二：`pruneNonDarwinPrebuilds` 硬编码 `darwin-arm64`

`stage-platform-prune.mjs:125`：

```js
if (platformDir.isDirectory() && platformDir.name !== 'darwin-arm64') {
```

对比同文件的 `pruneNonLinuxPrebuilds:57`，它用的是 `linux-${arch}`（参数化）。
darwin 版本写死了 arm64，**在 darwin-x64 上会把该保留的 prebuilds 删掉、
把 arm64 的留下**。

这条目前不影响 Windows 产物（darwin 分支只在 macOS 上跑），但它是同一类缺陷：
平台裁剪按字面量而非按 target 推导。补 win32 分支时必须同时修掉它。

### 问题四：打包脚本按旧假设硬编码，与当前代码不一致

`make-lite-staged.mjs` 与 `make-portable-lite.mjs` 位于任务根
（`EAC-Full-Build-20261004/`，**不在 EAC-Desktop 仓库内**），仍停留在 6.0.1 的形态：

| 位置 | 硬编码 | 当前实际 | 后果 |
| --- | --- | --- | --- |
| `make-lite-staged.mjs:130` | `LITE_RESOURCE_DIRS` 含 `'webview2'`（5 项） | `tauri.conf.json` 只有 4 项 | 键集断言失败 |
| `make-lite-staged.mjs:334` | `if (builtin !== 9)` | `BUILTIN_PLUGIN_DIRS` 10 项 | 断言失败 |
| `make-lite-staged.mjs:336` | 必需 `webview2/msedgewebview2.exe` | 不再装配该目录 | fail |
| `make-portable-lite.mjs:135,175` | 必需 + 门禁 G8f `webview2/msedgewebview2.exe` | 同上 | fail |

实测 `node make-lite-staged.mjs --dry-run` 当前第一步即失败：
`FAIL: 来源 staged-resources 不存在`。

## 决策

### 1. WebView2 不得随包

`webviewInstallMode` 只允许以下取值：

| 取值 | 增加体积 | 断网可用 | 实际生效 |
| --- | ---: | --- | --- |
| `downloadBootstrapper`（当前，默认） | ~0 | 否 | 是 |
| `offlineInstaller` | +127 MB | 是 | 是 |
| `embedBootstrapper` | +1.8 MB | 否 | 是 |
| ~~`skip` + 手工拷固定版目录~~ | +295 MB | 是 | **否** |
| `fixedRuntime`（Tauri 原生模式） | +180 MB | 是 | 是 |

- **禁止**再使用「`skip` + 手工把固定版运行时拷进 `resources`」的组合。
  这条路径只搬文件、不接线，产出的体积换不到任何收益。
- 确需离线安装时用 `offlineInstaller`（+127 MB），它把 WebView2 装进系统，
  之后所有应用受益，不是塞一份死副本。
- 若未来确实要用固定版运行时，必须用 Tauri 的 `fixedRuntime` 模式（它会把路径
  传进 `browserExecutableFolder`），并在 `stage-resources.mjs` 里校验
  `msedgewebview2.exe` 的 SHA-256 与 `downloads/webview2/runtime-lock.json` 一致。
  **只搬目录、不配 `fixedRuntime` 的写法一律拒绝。**

### 2. 死重文件不进 staged 树

在 `stage-node-modules.mjs` 的 `includeSource()` 里增加排除，与既有的
`devOnlyPaths` / `EXCLUDED_TOP_LEVEL` 并列：

排除清单：

- `.d.ts` / `.d.mts` / `.d.cts` —— 类型声明，运行期从不加载
- `.map` —— sourcemap
- `.md` / `.markdown` —— 文档
- `.ts` / `.mts` / `.cts` —— **仅当未被该包 `exports` 引用时**排除

**例外规则（必须实现，不得省略）：**

1. **`exports` 引用检查**：若某包的 `exports` / `main` / `module` / `browser`
   把任何条件指向 `.ts` 文件，该文件保留。当前实测 13 处（`zod` 的 `@zod/source`、
   `eventsource` 的 `source` 等），运行期虽不命中，但规则必须显式覆盖。
2. **`REQUIRED_FILES` 检查**：`verify-staged-runtime.mjs` 的 `REQUIRED_FILES`
   中若出现被排除的扩展名，该文件保留。当前 43 项内无此类项（实测 0 命中），
   但规则必须显式检查，否则未来加项会静默断。
3. **`vendor/` 不适用本条**：只作用于 `node_modules`。

**裁剪只作用于 staging 目标树，绝不修改 `dsh-desktop/node_modules` 源树** ——
源树要供 `tsc`、`node --test`、`tsx` 等开发期工具使用。

`stage-node-modules.mjs` 必须打印排除统计（文件数 + 字节数），否则排障时
无法区分「被规则排除」与「拷贝静默丢文件」。

同样处理 `vendor/kernel/`：那 327 个 tgz 只供 `npm install` 解析 `file:` 依赖，
运行期不读。**但保留判断留给后续任务**（见「未决」）。

### 3. 平台裁剪必须三平台对称

`stage-resources.mjs` 必须为 `win32` 补齐与 `linux` / `darwin` 等价的裁剪块：
删除目标平台不可达的 `.node` / `prebuilds/` / 平台专属包。

判定依据不得依赖目录名猜测：`.node` 按文件魔数（ELF / Mach-O / PE）识别，
与既有 `stage-platform-prune.mjs` 一致；`prebuilds/` 按目录名 `win32-${arch}` 保留、
其余删除。

**同时修掉 `pruneNonDarwinPrebuilds:125` 的 `'darwin-arm64'` 字面量**，
改为 `darwin-${arch}`，与 `pruneNonLinuxPrebuilds:57` 的写法对齐。

**新增平台专属大包时必须同时补齐三平台分支**，否则该包会在另外两个平台的产物里
白占体积。

**`assertNoUnexpectedNativePlatform` 不能替代本决策** —— 它只看包名里的平台 token，
对包内 `prebuilds/` 子目录无效（实测 48 个异平台 prebuilds 全部漏网）。
两者是互补关系，不是替代关系。

### 4. 体积与文件数双预算

两个门禁必须同时满足，任一超限即拒绝出包：

| 门禁 | lite | full |
| --- | ---: | ---: |
| portable.zip | ≤ 260 MB | ≤ 1.1 GB |
| 文件数 | ≤ 20000 | ≤ 40000 |

基线对照（6.0.1 lite）：536.7 MB / 35424 文件，**两项均超**。

`node_modules` 是唯一需要动的部分。2026-10-18 对当前源树实测，按**互斥四类**切分
（各类无重叠，加总 = 全量）：

| 分类 | 文件数 | 原始 | 处理 |
| --- | ---: | ---: | --- |
| dev-only 包（`package-lock.json` 标 `dev: true`） | 2571 | 70.0 MB | 排除 |
| 死重文件（dev-only 包之外，且未被 `exports` 引用） | 15424 | 100.7 MB | 排除 |
| 被 `exports` 引用的 `.ts`（`source` 条件） | 13 | ~0 MB | **保留** |
| 保留 | 14397 | 392.3 MB | 随包 |
| **合计** | **32405** | **563.0 MB** | |

排除合计 **17995 文件 / 170.7 MB 原始**。

按实测压缩率折算（`node_modules` 整体 30.7%，死重扩展名 22.9%）：

| 动作 | 压缩后变化 | 文件数变化 | 依据 |
| --- | ---: | ---: | --- |
| 移除随包 WebView2 | −294.7 MB | −257 | zip 中央目录实测 |
| 排除 dev-only 依赖 | −21.5 MB | −2571 | 已由 `d00ff202` 实现 |
| 排除死重文件 | −23.1 MB | −15424 | 本 ADR 新增 |
| 剔除异平台 prebuilds | −约 4 MB | −48 | 源树实测 15.92 MB 原始 |
| **合计** | **约 −343 MB** | **−18300** | |

**预期落点：约 193 MB / 约 17124 文件。**

体积门禁在第一步就已满足；**文件数门禁只有做完死重排除才能满足**
（只做前两步是 32596 文件，仍超 20000）。这是有意设计：字节数靠砍一个大文件
就能达标，文件数才反映真实的安装成本。

> **`d00ff202` 的 dev-only 排除只在 staging 生效，源树仍有这 2571 个文件。**
> 所以「排除 dev-only」不是待办项，而是已生效项 —— 上面的 −21.5 MB / −2571
> 是相对 6.0.1 产物的差值，不是相对当前代码的增量。

**`.d.ts` 与 `.ts` 必须分开处理。** 实测 647 个包里有 850 处 `types` /
`exports.*.types` 指向 `.d.ts` —— 这些是类型声明，运行期从不加载，可以整体排除。
但有 13 处 `exports` 把 `source` 条件指向真实 `.ts`（`zod` 的 `@zod/source`、
`eventsource` 的 `source` 等）。Node 默认不解析自定义条件，运行期同样不会命中，
但**排除规则必须显式检查 `exports` 引用**，否则未来某个包把 `default` 条件指向
`.ts` 时会静默产出加载失败的树。

### 5. 打包脚本必须从配置派生，不得硬编码

任务根的编排脚本（`make-lite-staged.mjs` / `make-portable-lite.mjs` /
`bundle-lite.mjs`）**不得**再硬编码以下任何一项：

- `tauri.conf.json` 的 `bundle.resources` 键集 → 从配置读取并断言「键集 = 4 项」，
  或直接遍历实际键集
- 随包内置插件数 → 从 `BUILTIN_PLUGIN_DIRS` 读取，不得写字面量
- 资源目录清单 → 从 `LITE_RESOURCE_DIRS` 与实际 `resources` 求交集后校验

硬编码的代价已经在本次暴露：代码把 WebView2 去掉、把 builtin 从 9 改到 10 之后，
四个断言全部失效，构建链第一步就断，而且**没有任何测试会提前发现**
（这些脚本在仓库外，`npm test` 覆盖不到）。

**因此：`tauri-shell/` 内新增任何影响产物形态的配置项时，必须同步检查任务根脚本。**
把这条写进 `references/updates-and-packaging.md`。

## 实施清单

按依赖顺序：

1. **`stage-node-modules.mjs` 增加死重排除**（含 `exports` 引用与 `REQUIRED_FILES` 例外检查，并打印排除统计）
2. **`stage-platform-prune.mjs` 增加 `pruneNonWindowsPrebuilds(dir, arch)`**，同时修掉 `pruneNonDarwinPrebuilds:125` 的 `'darwin-arm64'` 字面量
3. **`stage-resources.mjs` 补 `win32` 裁剪块**，与 `:562` / `:580` 两个既有块对称
4. **修任务根 4 处硬编码**（`LITE_RESOURCE_DIRS` / builtin 数 / 两处 webview2 必需项）
5. **加体积与文件数门禁**到 `make-portable-lite.mjs` 的 G8 系列，与既有 G8a–G8f 并列
6. 重新出包并**实测**，把新数字回填到本文「计数批注」

第 1 与第 2–3 步互不依赖，可并行；第 4 步必须在前三步之后（否则出的包还是旧的）。

## 验证

单靠 `verify-staged-runtime.mjs` 不够 —— 它只校验必需文件与闭包完整性，
**从不检查多余文件，也不校验体积**。必须补充：

| 检查 | 做法 |
| --- | --- |
| 死重已排除 | staged 树内 `.d.ts/.map/.md` 计数为 0；`.ts` 仅剩 `exports` 引用项 |
| 平台裁剪正确 | staged 树内 `prebuilds/` 只剩 `win32-${arch}` |
| 裁剪未破坏加载 | 真实内核 boot，确认插件激活数与 `BUILTIN_PLUGIN_DIRS` 一致 |
| 体积门禁 | 实测 `portable.zip` 字节与条目数，与预算比对 |
| 安装可用 | **解压后的运行树**启动验证，不能只看 `target/release` |

**最后一条是硬要求**：`target/release` 与安装后目录树不是一回事，
`tauri-build` 的 `copy_resources()` 只逐文件复制、没有删除逻辑，
上一轮中间物会静默残留并混进产物。

**裁剪类改动必须先证明「没删错」再证明「删够了」。** 建议顺序：先在源树跑一遍
排除规则，统计将删/将留的文件数，与本文表格核对；再对 staged 树做真实内核 boot；
最后才看体积。反过来做（先看体积达标就收工）会把加载失败留到用户机器上。

## 后果

### 正面

- lite 体积从 536.7 MB 降到约 193 MB，文件数从 35424 降到约 17124
- 安装时长随文件数线性下降（文件数减少 51.7%）
- 三平台产物不再夹带对方平台的二进制

### 代价

- 首次安装需要联网（`downloadBootstrapper`）。断网场景必须改用 `offlineInstaller`（+127 MB）
- 死重排除会让 staged 树与源树内容不一致，排障时容易误判「文件丢了」
  —— 因此 `stage-node-modules.mjs` 必须打印排除统计

### 未决

- **`libreoffice-kit`（压缩后 65.6 MB）是否随包**：它只服务 office→PDF 转换。
  需要先确认缺失时相关 skill 是优雅降级还是硬崩。在结论出来前**保持随包**，
  不得为了体积擅自移除。
- **`vendor/kernel/`（19.9 MB，327 个 tgz）能否在打包后删除**：运行期不读，
  但删掉后用户无法离线重装内核依赖。需评估离线修复链路是否依赖它。
- **`node_modules/pnpm` 与 `vendor/pnpm` 字节级重复**（各 454 文件 / 21.1 MB）：
  代码只从 `vendor/pnpm` 取（`package-manager.ts:15`），`node_modules/pnpm` 是
  `devDependencies` 的副产物。已由 `devOnlyPaths` 排除，**无需单独处理** ——
  此处记录以免重复排查。

## 参考

- 实测数据来源：`EAC-Release-v6.0.1-20261007/lite/ARTIFACT-AUDIT-lite.json` 与 zip 中央目录
- WebView2 取值语义：`tauri-utils-2.10.0/src/config.rs` 的 `WebviewInstallMode`
- wry 环境创建：`wry-0.55.1/src/webview2/mod.rs:343`
- 固定版运行时锁：`EAC-Full-Build-20261004/downloads/webview2/runtime-lock.json`
- 分发边界：`docs/adr/0008-plugin-distribution-boundary.md`
- 打包操作说明：`.agents/skills/deepseek-harness-eac-dev/references/updates-and-packaging.md`
