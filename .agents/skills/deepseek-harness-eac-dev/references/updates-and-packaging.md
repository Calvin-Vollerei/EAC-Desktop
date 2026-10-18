# 更新、安装与打包规范

## 三类更新

1. EAC 客户端更新。
2. DSH Agent overlay 更新。
3. 内置或市场插件更新。

三者不得共用模糊状态或相互覆盖目录。

## 客户端

- 安装版通过 NSIS Setup 完成替换。
- 便携版按完整目录树 staging、交换和重启。
- 下载支持校验、断点恢复和镜像回退。
- 安装失败保留旧程序、更新文件和诊断日志。
- 旧 Electron 版本接管不得删除用户数据。

### 便携整树事务

修改便携版目录树交换时必须考虑：

- helper 接管成功后主程序才退出。
- 壳 PID 超时仍存活时中止，不触碰安装树。
- staging 内容、目标版本、磁盘空间和固定顶层项先校验。
- 使用唯一事务目录和持久 journal 记录每一步。
- 交换顺序固定，失败按 journal 逆序恢复。
- helper 被杀、断电或系统重启后，下一次启动能识别并恢复未完成事务。
- last-good 只在新 sidecar、`boot.start` 和 Web 探针确认健康后清理。
- 回滚失败时保留新旧树、journal 和日志。
- `%APPDATA%`、`DSH_HOME`、profile、会话和下载缓存不参与程序树交换。
- 未知顶层文件、reparse point 和符号链接采用明确的保留、拒绝或隔离策略。

## Agent

- 更新安装到 overlay/staging。
- 新版本未确认健康前保留 previousAgent。
- 启动失败支持回退。
- npm registry 和镜像切换必须有超时和明确错误。

## 插件

- 更新前执行兼容门槛、冲突检查和 guard 快照。
- staging 验证通过后再替换内置覆盖层。
- 更新失败清理 staging，但保留当前可用版本。

## 打包

- `stage-resources.mjs` 是 Tauri 资源装配入口。
- 新增运行文件必须进入资源清单和完整性测试。
- 同步版本号到产品配置和发布产物命名。
- NSIS 与便携包都要验证真实目录结构。
- 不能只验证 `target/release`，必须验证安装后或解压后的运行树。
- **体积与裁剪预算见 [ADR 0011](../../../../docs/adr/0011-packaging-budget-and-trimming.md)**：
  lite ≤ 260 MB / ≤ 20000 文件，full ≤ 1.1 GB / ≤ 40000 文件。改产物形态前先读该 ADR。

### 体积红线

- **WebView2 不得随包。** 只允许 `downloadBootstrapper`（默认）、`offlineInstaller`
  （+127 MB，可断网）或 `fixedRuntime`（+180 MB，Tauri 原生模式）。
  禁止「`skip` + 手工拷固定版运行时目录」—— 那条路径只搬文件不接线，
  6.0.1 因此白占 294.7 MB（wry 传 `PCWSTR::null()`，永远用系统运行时）。
- **平台裁剪三平台对称。** `stage-resources.mjs` 的 `linux` / `darwin` 分支各自
  裁剪跨平台 payload，`win32` 分支曾长期缺失，实测 Windows 产物夹带 48 个
  异平台 prebuilds（15.92 MB），其中 `node-pty/prebuilds/win32-arm64` 单目录 11.22 MB。
- **`assertNoUnexpectedNativePlatform` 只看包名**，对包内 `prebuilds/` 子目录无效，
  不能当作平台裁剪的替代品。
- **死重文件不进 staged 树**：`.d.ts` / `.map` / `.md` 直接排除；`.ts` 仅当未被
  `exports` 引用时排除（当前 13 处例外）。裁剪只作用于 staging，不改源树。

### 编排脚本必须从配置派生

任务根的 `make-lite-staged.mjs` / `make-portable-lite.mjs` / `bundle-lite.mjs`
**在仓库外，`npm test` 覆盖不到**，因此更容易腐化。它们不得硬编码：

- `tauri.conf.json` 的 `bundle.resources` 键集
- 随包内置插件数（`BUILTIN_PLUGIN_DIRS` 长度）
- 资源目录清单（`LITE_RESOURCE_DIRS`）

硬编码的代价已经发生过一次：代码把 WebView2 去掉、builtin 从 9 改到 10 之后，
四处断言全部失效，构建链第一步就断，而没有任何测试能提前发现。

**规则：`tauri-shell/` 内新增任何影响产物形态的配置项时，必须同步检查任务根脚本。**

## 关键文件

- EAC 客户端更新核心：`dsh-desktop/client-updater.js`
- Tauri 客户端更新适配：`lib/desktop/client-update.ts`
- Agent 更新：`dsh-desktop/updater.js`
- 内置插件更新：`dsh-desktop/plugin-updater.js`
- Tauri 资源装配：`tauri-shell/stage-resources.mjs`
- 生产依赖装配与裁剪：`tauri-shell/stage-node-modules.mjs`
- 平台裁剪原语：`tauri-shell/stage-platform-prune.mjs`
- 便携包：`tauri-shell/make-portable.mjs`
- NSIS 钩子：`tauri-shell/installer-hooks.nsh`
- Tauri 配置：`tauri-shell/tauri.conf.json`
- 发布工作流：`.github/workflows/release-tauri.yml`
- 体积预算：`docs/adr/0011-packaging-budget-and-trimming.md`

## 测试组

- `client-updater-*.test.ts`
- `client-update-platform.test.ts`
- `updater-*.test.ts`
- `update-mirror-chain.test.ts`
- `plugin-updater.test.ts`
- `bundle-integrity.test.ts`
- `bundled-files.test.ts`
- `installer-*.test.ts`
- `verify-dist-fresh.test.ts`

便携更新不能只测成功路径。还要覆盖逐步骤失败、文件锁、进程残留、helper 中断、健康检查失败、重复恢复和用户数据哈希不变。
