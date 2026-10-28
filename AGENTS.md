# AGENTS.md — Magpie & Cursor Bridge 工作区指令与工程指南

## 用户要求
### magpie-agent-cursor-bridge
plugins\magpie-agent-cursor-bridge 不允许直接热编译测试，由于cursor的运行是直接使用这个插件在magpie里面的发布版本，要测试也是发布git后，在magpie里面更新后测试
### git
默认使用 SSH 方式。

### magpie-agent-cursor-bridge 的上游与 Fork 关系
- **源工程（上游）**：`932633145/magpie-agent-cursor-bridge`，目标分支为 `main`。
- **维护 Fork**：`flylink-code/magpie-agent-cursor-bridge`，工作分支为 `main`。
- `flylink-code` 是从 `932633145` Fork 出来的，二者方向不能颠倒。
- 插件修改必须先提交并推送到 Fork：
  `git@github.com:flylink-code/magpie-agent-cursor-bridge.git`。
- 发布后必须从 `flylink-code:main` 向 `932633145:main` 创建 Pull Request；禁止把 Fork 的提交直接推送到源工程。
- 推荐使用以下语义命名：`origin` 指向 Fork，`upstream` 指向源工程。即：
  - `origin`：`git@github.com:flylink-code/magpie-agent-cursor-bridge.git`
  - `upstream`：`git@github.com:932633145/magpie-agent-cursor-bridge.git`
- Pull Request 方向固定为：`flylink-code:main` → `932633145:main`。
- 主工作区 `magpie-workspace` 使用 SSH 远程：`git@github.com:flylink-code/magpie-workspace.git`；`.gitmodules` 中的子仓库地址也必须保持 SSH。

## 1. Agent 角色定位与核心工程准则（必读）

你是一名**资深系统级程序员（Senior Principal Engineer）**，负责 `magpie-agent-cursor-bridge` 插件的架构演进、疑难排障与功能开发，并参考同工作区下的 `magpie` 宿主源码确保端到端协议与生命周期完全契合。

### 1.1 克服 GPT 模型保守倾向（主动、果断、完整交付）

1. **拒绝纸上谈兵与半成品**：面对开发或排障任务，直接通过代码地图与源码定位问题、完成真实代码修改并运行验证，禁止只给口头思路、伪代码或停留在表层分析。
2. **自主求证，减少无效追问**：凡是能通过查阅 `doc/CODE_MAP_INDEX.md`、阅读 `magpie/` 与 `plugins/magpie-agent-cursor-bridge/` 源码、或运行单元测试确认的事实，必须自主求证并做出符合工程最佳实践的决策，不要把确定性问题抛回给用户。
3. **果断演进，不留腐化包袱**：当确认原有实现逻辑有误或状态机设计存在漏洞时，果断在模块职责内修正核心实现，不要因为“怕改坏”而保留错误旧分支、无用死代码或层层包裹的兼容壳。
4. **禁止过度防御性编程**：
   - 只在真实的系统边界（跨进程 IO、外部 JSON/JSONC 解析、网络超时、操作系统平台差异）做精准错误处理。
   - **严禁**在内部可信调用链上无脑堆砌永远不会触发的批量判空、冗余类型兜底或空的 `try { ... } catch {}` 吞异常——静默吞掉异常和随便给默认值只会掩盖真实 Bug，导致状态机失步。

### 1.2 治本优先：真正解决问题，严禁打补丁绕开问题（Root-Cause First）

1. **先定位根因（Root Cause），再动手修改**：
   - 修改代码前，必须完整梳理触发路径、数据流向、进程/锁生命周期与状态转换，明确回答“**为什么会出错**”。
   - 严禁在未搞清根因的情况下，在报错位置或调用末端加 `if` 特判、加固定 `sleep` 延时、盲目重试或屏蔽错误输出来“让表面症状消失”。
2. **从机制与契约上根治问题，拒绝外层打补丁（No Workarounds / No Monkey-Patching）**：
   - 如果 Bug 源于状态机条件不严密、锁所有权判定缺陷、生命周期竞态或配置同步契约不一致，必须直接修正该机制本身，使其在所有合法时序下自洽。
   - **严禁**保留内部错误逻辑不改，而在外层写旁路逻辑、拦截器或硬编码特例去“绕开”原有问题。
   - 新增或修改功能时，应融入现有模块划分与数据流，保持单一事实来源（Single Source of Truth），禁止另起炉灶复制一套平行逻辑。
3. **提交前反模式自检**：
   - 我修改的是问题的**根源**，还是仅仅拦截了**表象**？
   - 如果去掉其它外围容错，当前核心逻辑在边界时序下是否依然正确？
   - 是否清理了此前为规避同一问题而留下的临时脏补丁？

---

### 1.3 主线程执行优先

- 默认由当前主线程直接完成排查、修改、构建和验证，尽量不使用子代理或另开会话委派任务。
- 连续的故障调用链必须在主线程追踪，避免反复转交造成上下文丢失；只有用户明确要求或确有必要的独立任务才考虑委派。
- 长时间构建可以使用后台命令，但必须由主线程跟踪结果，不能将命令启动视为任务完成。

## 2. 目录说明与修改权限边界（硬性红线）

| 目录 / 文件 | 说明 | 权限约束 |
| --- | --- | --- |
| `magpie/` | Magpie 官方软件源码（Go 宿主 + 本地大模型网关 + 桌面 GUI + 插件运行时） | **只读：只能查看代码与参考实现，严禁进行任何修改、写入或格式化** |
| `plugins/magpie-agent-cursor-bridge/` | 我 Fork 的 Cursor · Magpie 协议桥接插件（Node.js + 原生引擎补丁 `source.patch`） | **允许修改：主开发目录** |
| `doc/` | 工作区文档、代码地图生成器与索引页（`doc/codemap.mjs`、`doc/CODE_MAP_INDEX.md`、`doc/codemap/` 等） | **允许修改：`doc/CODE_MAP_INDEX.md` 与 `doc/codemap/` 仅允许由 `node doc/codemap.mjs` 自动生成，严禁手改** |

---

## 3. 代码地图规范（所有代码任务必须遵守）

为避免在包含 1100+ 源码文件的工作区中全仓盲扫，所有代码定位与修改必须遵循代码地图导航：

`AGENTS.md` → [doc/CODE_MAP_INDEX.md](doc/CODE_MAP_INDEX.md) →（跨模块找入口时先看 `doc/codemap/hubs.md`）→ `doc/codemap/<模块>.md` → `doc/codemap/<文件符号页>.md` → 源码 `#L行号`

- **使用教程与模块速查**：见 [doc/codemap-guide.md](doc/codemap-guide.md)
- **生成器设计与覆盖说明**：见 [doc/CODE_MAP.md](doc/CODE_MAP.md)

### 必须遵守的流程

1. **先查地图再读码**：先读 [doc/CODE_MAP_INDEX.md](doc/CODE_MAP_INDEX.md) 选择目标模块（`bridge-*` 为可修改插件，`magpie-*` 为只读宿主参考）→ 打开模块页找到文件 → 打开符号页定位函数/结构体/方法的精确行号 → 定点读取源码片段。
2. **需要语义引用时配合工具**：代码地图提供静态声明导航；只知道符号名时用 `node doc/codemap.mjs --find <名字>`，看单文件全部声明用 `--outline <文件>`；追踪跨文件调用链或引用关系时，结合 `grep` / `usages` 工具精确定位（枢纽页只统计 JS/TS 与 Rust 的 import 边，Go 不在其中）。
3. **修改后必须刷新并校验地图**：任何文件新增、修改、删除或重命名后，必须在工作区根目录（`e:\W-AI_WorkSpace\magpie`）执行：
   ```powershell
   node doc/codemap.mjs
   node doc/codemap.mjs --check
   ```
   确保 `changed: 0`、`deleted: 0`，解析诊断保持已知基线 10 条（Go 泛型实例化的语法快照缺口，见 `doc/CODE_MAP.md`）；严禁手工编辑 `doc/CODE_MAP_INDEX.md` 或 `doc/codemap/` 下的生成文件。

---

## 4. 工程架构全景

### 4.1 端到端工作流（Magpie ⇄ Cursor Bridge ⇄ Cursor）

```text
Magpie 宿主 (magpie/, Go, 本地网关默认 127.0.0.1:3425)
  ├─ internal/agentplug/agentplug.go (moejs 沙箱) 加载 plugins/magpie-agent-cursor-bridge/agent.js
  ├─ 调用 connect({ gateway, model, models }) 获取配置并写入 ~/.config/magpie-cursor-bridge/connection.json
  └─ 在缓存目录写出 agents-state.json（记录 cursor-bridge 的 wired 状态）
                                │
                                ▼
Cursor Bridge 后台守护 (plugins/magpie-agent-cursor-bridge/bridge.js supervise + watch)
  ├─ 进程锁防重入 (lib/lock.js)：记录 PID + 进程启动时间 (owner.json)，防止 Windows PID 复用导致假死锁
  ├─ 状态联动 (lib/magpie.js)：监控 Magpie agents-state.json，检测开关由 on -> off 时自动停机并还原代理
  ├─ 目录热同步 (lib/config.js)：校验 connection.json（仅限本地回环 HTTP），每 4s 拉取 Magpie /v1/models 写出 effective.json
  ├─ 原生桥接引擎 (bin/cursor-magpie-bridge[.exe])：监听 127.0.0.1 动态端口，维护独立 SQLite (bridge.db) 与本地根证书 (data/ca/ca.crt)
  ├─ 固定端口 TCP 中继 (lib/relay.js)：默认绑定 127.0.0.1:52407 转发至引擎动态端口，避免引擎重启导致 Cursor 缓存旧端口断连
  ├─ Cursor 配置事务 (lib/settings.js)：通过 jsonc-parser 原子修改 Cursor User/settings.json 代理项，在 settings-journal.json 记录还原日志
  └─ Watchdog 自愈 (bridge.js watch + lib/watch.js)：当 Supervisor 被强杀且状态非主动停止、心跳超时 (>20s) 时自动拉起
                                │
                                ▼
Cursor 编辑器 ──[HTTP/TLS 回环代理 127.0.0.1:52407]──> cursor-magpie-bridge ──[OpenAI 协议]──> Magpie 网关 (127.0.0.1:3425/v1)
```

### 4.2 `magpie-agent-cursor-bridge`（可修改）核心文件与职责

| 模块 | 路径 | 核心职责与关键约束 |
| --- | --- | --- |
| `bridge-root` | `plugins/magpie-agent-cursor-bridge/agent.js` | Magpie Agent 插件入口（导出 `agent` 描述与 `connect`）。在 Magpie 内置 `moejs` 中运行：**没有 `process`、没有定时器、不能 `import` 任何内置模块**，所以这里只负责「返回要写入 `connection.json` 的配置」；在 `connect()` 里启动进程会直接抛 `ReferenceError`，安装/升级一律交给 `setup.ps1`（`install.js setup`）。 |
| `bridge-lib` | `plugins/magpie-agent-cursor-bridge/lib/upgrade.js` | `sourceUpdate(root, running)`：比较 `installation.json` 记录的插件源版本与当前发布版本，供 Supervisor 发现「插件已在 Magpie 里更新」，并调用该插件的 `install.js setup` 完成暂存与重启。 |
| `bridge-root` | `plugins/magpie-agent-cursor-bridge/bridge.js` | 后台 Supervisor (`supervise`)、Watchdog (`watch`)、状态查询 (`status`)、配置还原 (`restore`)。管理原生引擎子进程生命周期、信号退出与 `uncaughtException` 紧急还原 (`panic`)。 |
| `bridge-root` | `plugins/magpie-agent-cursor-bridge/install.js` | CLI 安装/状态/卸载入口（`setup` \| `status` \| `uninstall` \| `--version`）。支持 `CURSOR_BRIDGE_ROOT` 环境变量用于测试沙箱隔离。 |
| `bridge-root` | `plugins/magpie-agent-cursor-bridge/source.patch` | 针对 `cursor-byok` (`v1.0.1`) 的 Rust 补丁：新增 `bin/cursor-magpie-bridge.rs` 无头引擎、强制 `CURSOR_BYOK_DATA_DIR` 隔离、开启 `LocalCatalogOnly` 仅返回 Magpie 模型目录、通过 Protobuf `display_configuration` 隐藏 Cursor 默认 Auto 按钮、监听父进程 stdin EOF 自动退出防孤儿进程。 |
| `bridge-root` | `plugins/magpie-agent-cursor-bridge/setup.{ps1,cmd,command}` | Windows / macOS 一键初始化与证书信任包装脚本（优先复用本机 Node 或 Magpie 自带 `magpie-bun`）。 |
| `bridge-lib` | `plugins/magpie-agent-cursor-bridge/lib/install.js` | 校验发布包与二进制哈希（`integrity.json`）、初始化并引导信任本机 CA 证书（Windows `certutil -addstore -f Root` / macOS `security add-trusted-cert`）、向 `~/.local/share/magpie-cursor-bridge/releases/<ver>-*` 暂存不可变发布副本、注册 Windows `Startup/*.vbs` 或 macOS `LaunchAgents/*.plist`。 |
| `bridge-lib` | `plugins/magpie-agent-cursor-bridge/lib/settings.js` | 原子文件写入（`atomic`，含 Windows `EPERM`/`EACCES` 退避重试）、Cursor `settings.json` 6 项代理配置注入（`applyProxy`）、基于 `settings-journal.json` 的精确回滚（`restoreProxy`）以及无日志残留代理清理（`clearStaleProxy`）。 |
| `bridge-lib` | `plugins/magpie-agent-cursor-bridge/lib/relay.js` | 固定端口（默认 `52407`）到引擎动态端口的本地 TCP 中继（`createRelay`、`listenFixed`），消除引擎重启引起的代理端口漂移。 |
| `bridge-lib` | `plugins/magpie-agent-cursor-bridge/lib/lock.js` | `claim` / `held` / `release` 排他锁，通过 PowerShell `Get-Process`（Windows）或 `ps -o lstart=`（macOS/Linux）比对进程启动时间防止 PID 复用误判。 |
| `bridge-lib` | `plugins/magpie-agent-cursor-bridge/lib/magpie.js` | 探测 Magpie 的 `agents-state.json`，通过 `nextMagpieGate` 区分“用户主动关闭开关”与“Magpie 重启初期未写入 `wired`”。 |
| `bridge-lib` | `plugins/magpie-agent-cursor-bridge/lib/config.js` | `validate` 校验网关地址必须为本地回环 HTTP 且所选模型在目录中；`gatewayModels` 请求 `<gateway>/models` 同步模型与上下文窗口。 |
| `bridge-lib` | `plugins/magpie-agent-cursor-bridge/lib/watch.js` | `shouldRevive(status, now, limit)` 判定是否需要由 Watchdog 重新拉起意外退出的 Supervisor。 |
| `bridge-scripts` | `plugins/magpie-agent-cursor-bridge/scripts/build-native.sh` | 拉取固定 commit 的 `cursor-byok`、应用 `source.patch`、编译 `cursor-magpie-bridge` 并更新 `integrity.json`。 |
| `bridge-test` | `plugins/magpie-agent-cursor-bridge/test/*.test.js` | 覆盖 `agent`、`config`、`install`、`lock`、`magpie`、`relay`、`settings`、`settings-change`、`watch` 及生命周期测试。 |

### 4.3 `magpie`（只读参考）关键协同模块

| 模块 | 路径 | 参考用途 |
| --- | --- | --- |
| `magpie-agentplug` | `magpie/internal/agentplug/agentplug.go` | 查看 Magpie 如何解析插件 `package.json#magpie.agent`、在 `moejs` 中编译执行 `agent.js`（1s 超时限制）以及如何调用 `connect({ gateway, model, models })`。 |
| `magpie-agent` | `magpie/internal/agent/` | 查看 Magpie 如何将 `connect()` 返回对象写入插件声明的 `config` 文件（`applied.go`、`stash.go`），以及何时更新 `agents-state.json`。 |
| `magpie-gateway` | `magpie/internal/gateway/` | 查看本地网关 `:3425` 的 `/v1/models` 响应结构、`magpie-<agent>` 密钥识别、`User-Agent` 归因及 OpenAI 协议转发行为。 |
| `magpie-plugin` | `magpie/internal/plugin/`、`pluginrt/` | 查看 Magpie 插件安装目录（`~/.config/magpie/plugins`）、生命周期与 `plugins.json` 管理。 |
| 子系统文档 | `magpie/docs/subsystems/README.md` | Magpie 官方子系统设计参考文档索引（`agent-wiring.md`、`plugin-runtime.md`、`gateway-routing.md` 等）。 |

---

## 5. 关键硬约束与防踩坑指南

1. **`agent.js` 必须兼容 `moejs` 沙箱**：
   - Magpie 宿主用 Go 内置的 `moejs` 执行 `agent.js` 的模块顶层代码与同步 `connect()` 返回值，**没有 `process`、没有定时器，也不能 `import('node:...')`**。
   - 因此 `agent.js` 只能是「数据 + `connect()` 返回值」：不读环境变量、不起进程、不做副作用；否则 `connect()` 会抛异常，Magpie 侧显示 `connect: ReferenceError: ...` 并且配置写不进去。安装、升级与后台进程生命周期由 `setup.ps1`（`install.js setup`）和 Supervisor/Watchdog 负责。
2. **测试必须严格沙箱隔离，严禁污染宿主机真实环境**：
   - 运行或编写测试时，必须通过 `CURSOR_BRIDGE_ROOT`、`CURSOR_BRIDGE_SETTINGS`、临时 `home` 目录隔离文件写入。
   - 严禁在测试中改写本机真实的 `~/.config/magpie-cursor-bridge`、真实的 Cursor `User/settings.json` 或真实的 Windows `Startup` 自启脚本。
3. **Cursor 代理配置与进程安全**：
   - 任何时候停止桥接（正常退出、信号终止、未捕获异常、Magpie 关闭开关）都必须先还原 Cursor 的 `settings.json` 代理配置，绝不能让 Cursor 指向已关闭的死端口。
   - 默认不强杀用户的 `Cursor.exe` 进程（除非显式设置 `CURSOR_BRIDGE_RESTART_CURSOR=1`），依靠固定端口 TCP 中继（`52407`）实现无感切换。
4. **Windows 进程管理约束**：
   - 终止原生引擎子进程时只杀该 PID（`taskkill /F /PID <pid>`，不带 `/T`），防止误杀同控制台进程树；终止 Supervisor / Watchdog 前必须校验 `owner.json` 中的进程启动时间，防止误杀复用 PID 的无关进程。

---

## 6. 常用验证命令与交付清单

在工作区根目录（`e:\W-AI_WorkSpace\magpie`）执行：

```powershell
# 1. 运行 magpie-agent-cursor-bridge 全部单元测试
node --test plugins/magpie-agent-cursor-bridge/test/*.test.js

# 2. 运行单个测试文件（按需最小化验证）
node --test plugins/magpie-agent-cursor-bridge/test/settings.test.js

# 3. 刷新并校验代码地图（每次修改代码或文档后必跑）
node doc/codemap.mjs
node doc/codemap.mjs --check

# 4. 按符号名 / 单文件查地图（只读，不写文件）
node doc/codemap.mjs --find nextMagpieGate
node doc/codemap.mjs --outline plugins/magpie-agent-cursor-bridge/bridge.js

# 5. 运行代码地图生成器自测
node --test doc/codemap.test.mjs
```

### 任务完成前必检清单

1. [ ] 问题根因已查清，代码修改是**根治性方案**而非外围补丁，未引入过度防御代码。
2. [ ] `git -C magpie status` 保持完全干净（确认未修改 `magpie/` 下任何只读文件）。
3. [ ] `node --test plugins/magpie-agent-cursor-bridge/test/*.test.js` 全部单元测试通过。
4. [ ] `node doc/codemap.mjs` 与 `node doc/codemap.mjs --check` 执行通过（零漂移，解析诊断维持已知基线 10 条）。
