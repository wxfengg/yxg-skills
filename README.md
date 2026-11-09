# YXG Skills

用于项目集成的 Agent Skills。目前包含 `yxg-mqtt-use`，按 `@yxg/mqtt@0.1.0` 的实际接口指导网页、微信小程序及 App 接入。

公开技能仓库：[wxfengg/yxg-skills](https://github.com/wxfengg/yxg-skills)。安装 skill 与安装 MQTT 库相互独立。

## 安装

使用 Vercel 的 [Skills CLI](https://github.com/vercel-labs/skills)，需要 Node.js 22.20 或以上。它不是本项目的运行依赖，不需要修改业务项目的依赖清单。

个人全局安装：

```sh
npx skills add wxfengg/yxg-skills --skill yxg-mqtt-use -a codex -g
```

去掉 `-g` 则安装到当前项目；去掉 `-a codex` 可交互选择 CLI 支持的工具。其他工具的调用方式遵循该工具约定。

本地试用（把路径替换成此目录的真实路径）：

```sh
npx skills add /path/to/yxg-skills --list
npx skills add /path/to/yxg-skills --skill yxg-mqtt-use -a codex
```

建议在临时空目录执行本地试用，避免覆盖已有技能。安装后在目标项目打开新的 Codex 会话；若未被发现，重启客户端后重试。

不使用 CLI 时，也可将 `skills/yxg-mqtt-use` 完整复制到自己的 Codex skills 目录，保留 `agents/` 与 `references/`。

## 准备 MQTT 库

安装技能不会安装 `@yxg/mqtt`。从有权限访问的 [yxg-mqtt Release](https://github.com/wxfengg/yxg-mqtt/releases/tag/v0.1.0) 或维护者取得 `yxg-mqtt-0.1.0.tgz` 和 SHA-256 校验记录。

库仓库为私有，公开技能不会授予源码或附件访问权限。不将访问令牌写入命令示例、技能或项目。不要把 GitHub 自动生成的 Source code 压缩包当作 npm tgz。

```sh
npm install /actual/path/yxg-mqtt-0.1.0.tgz
```

也可提供 tgz 路径，由技能在获准的接入范围内沿用项目原包管理器完成安装。

## 使用

在目标项目中输入：

```text
$yxg-mqtt-use
将 @yxg/mqtt 接入当前项目，目标平台为微信小程序。
安装包位于 /actual/path/yxg-mqtt-0.1.0.tgz。
保留现有登录流程和业务 Handler，完成接线、构建、测试和文档更新，先不要提交。
```

网页项目可将平台改为 Vue / React / H5。App 项目请说明 uni-app App、WebView 或原生桥的实际情况；技能会核对运行环境，缺少原生 MQTT 桥时不会生成假的实现。

只需评估时使用：

```text
使用 $yxg-mqtt-use，只读分析当前项目接入 @yxg/mqtt 的改动范围，不修改文件。
```

凭据沿用项目现有认证模块，只提供代码位置，不粘贴密码或 token。技能会检查登录退出、订阅就绪、消息收发与清理；微信重点检查延迟加载。自动测试不替代真实 Broker、微信手机与设备业务 ACK 验证。

## 更新与移除

```sh
npx skills check
npx skills update
npx skills remove yxg-mqtt-use -a codex -g
```

`update` 可能更新其他已安装技能；按 CLI 提示检查范围。项目级安装移除时省略 `-g`。更新技能不会更新业务项目中的 MQTT 依赖。

## 目录与维护

```text
skills/yxg-mqtt-use/
├── SKILL.md
├── agents/openai.yaml
└── references/
    ├── api-contract.md
    ├── browser.md
    ├── weixin.md
    ├── app.md
    ├── communication.md
    └── validation.md
```

入口负责平台分流；参考文件维护 API、生命周期、微信加载顺序及证据边界。库版本更新后核对实际 exports、声明和实现，再更新技能；不能仅替换版本字符串。完整使用文档见库的 `docs/guide/skill-integration.md`。

每次更新应验证 CLI 能发现并安装完整目录，检查相对链接、环境专属路径和凭据。本仓库不包含库的二进制制品；目前未指定开源许可证。

## 安装验证记录

2026-11-09：skill-creator 结构检查通过；Skills CLI 1.7.6 能发现唯一技能，并在临时项目完成 Codex 安装。安装后的 8 个技能文件逐一与源文件核对一致，相对参考链接全部存在。验证使用临时项目，未安装到个人全局目录、未替换现有技能。

远程验证：已从 GitHub 公开仓库执行以下命令，在临时项目安装成功；8 个技能文件与发布源逐一核对一致，并生成 `skills-lock.json`。GitHub 匿名 API 确认仓库为 Public。验证未修改个人全局技能。

```sh
npx --yes skills add wxfengg/yxg-skills --skill yxg-mqtt-use -a codex -y
```

这里省略 `-g` 是为了隔离测试；个人跨项目使用可按上方全局安装命令执行。测试覆盖 Codex 安装与文件完整性，不代表验证了所有 Agent 工具或真实设备接入。
