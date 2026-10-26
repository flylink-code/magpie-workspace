# Magpie 工作区

[English](README.md) | 简体中文

面向 **Cursor · Magpie Bridge** 插件的开发工作区，通过 Git 子模块整合桥接插件、Magpie 上游源码与自动生成的代码地图。本仓库负责协调这些项目，**不是 Magpie 官方应用仓库，也不是独立的插件发布包**。

桥接插件通过本地协议桥接，让 Cursor 使用 Magpie 中配置的模型与路由组。

## 仓库结构

| 路径 | 用途 | 来源仓库 |
| --- | --- | --- |
| [`magpie/`](magpie/) | Magpie 上游源码，在本工作区中仅供**只读参考** | [yetone/magpie](https://github.com/yetone/magpie) |
| [`plugins/magpie-agent-cursor-bridge/`](plugins/magpie-agent-cursor-bridge/) | 桥接插件实现、原生引擎补丁、初始化脚本与测试 | [flylink-code/magpie-agent-cursor-bridge](https://github.com/flylink-code/magpie-agent-cursor-bridge) |
| [`doc/`](doc/) | 代码地图生成器、生成索引与导航指南 | [flylink-code/magpie-doc](https://github.com/flylink-code/magpie-doc) |
| [`AGENTS.md`](AGENTS.md) | 工作区工程规范与修改边界 | 本仓库 |

## 克隆与初始化

以下工作区操作需要 Git 和 Node.js。插件运行配置还需要安装 Cursor 与 Magpie，具体见[插件安装指南](plugins/magpie-agent-cursor-bridge/README.md)。

```powershell
git clone --recurse-submodules https://github.com/flylink-code/magpie-workspace.git
Set-Location magpie-workspace
```

对于已有的克隆，将子模块初始化到主仓库记录的版本：

```powershell
git submodule update --init --recursive
```

该命令不会自动跟踪子模块分支的最新版本。更新前先检查本地改动，不要使用强制选项丢弃未保存的工作。

## 使用桥接插件

普通使用场景可直接在 Magpie → **Plugins** 中添加：

```text
github:flylink-code/magpie-agent-cursor-bridge
```

Windows/macOS 初始化、本地证书信任、模型连接、状态查询与卸载步骤，详见[插件 README](plugins/magpie-agent-cursor-bridge/README.md)。仅克隆本工作区不会安装或启动桥接服务。

## 开发流程

1. 阅读 [`AGENTS.md`](AGENTS.md)，了解工程约束。
2. 从[代码地图索引](doc/CODE_MAP_INDEX.md)选择模块，经文件及符号页定位源码；另见[导航教程](doc/codemap-guide.md)与[生成器设计说明](doc/CODE_MAP.md)。
3. 在 [`plugins/magpie-agent-cursor-bridge/`](plugins/magpie-agent-cursor-bridge/) 中修改插件，保持 [`magpie/`](magpie/) 不变。
4. 先运行针对性测试，交付前运行插件完整测试。测试必须隔离配置、用户目录与 Cursor 设置，禁止将真实机器上的初始化或卸载当作测试。
5. 文件变更后刷新并校验代码地图，禁止手动修改生成的地图文件。

在工作区根目录执行：

```powershell
# 插件单元测试
node --test .\plugins\magpie-agent-cursor-bridge\test\*.test.js

# 刷新并校验代码地图
node .\doc\codemap.mjs
node .\doc\codemap.mjs --check

# 修改生成器时运行其测试
node --test .\doc\codemap.test.mjs
```

地图校验应显示 `changed: 0`、`diagnostics: 0`。

## 子模块协作

每个子模块都有独立的提交历史、远程仓库与工作区。应先在对应子模块中提交并推送改动，再在主仓库提交更新后的子模块版本指针。**推送主仓库不会同时推送子模块提交。**

常用检查命令：

```powershell
git submodule status
git status --short
git -C doc status --short --branch
git -C plugins\magpie-agent-cursor-bridge status --short --branch
git -C magpie status --short
```

不要提交本机专用代理配置、凭据或运行数据。各组件的许可证与发布信息请查阅其对应仓库。
