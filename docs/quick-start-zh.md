# REA 中文快速指南

用本地源码构建 REA，连接 Codex MCP，再分析一个只有两个文件的 JavaScript 示例。以下命令适用于 macOS／Linux 的 Bash 或 Zsh；静态 JavaScript 分析无需 Hopper、Ghidra 或 IDA，也不执行目标应用。

## 1. 下载与本地安装

先准备 Git、Node.js 和 npm。REA 支持 Node.js `^22.19.0 || ^24.11.0 || >=26.0.0`；仓库开发工具链固定为 Node.js **24.18.0**（`.nvmrc`）和 npm **11.16.0**（`package.json`）。请用自己的工具链管理方式准备版本，REA 设置流程不会安装或升级 Node.js、npm 或 Homebrew。

检查版本，然后在希望保存项目的目录运行：

```bash
git --version
node --version
npm --version

git clone https://github.com/nings/rea.git
cd rea
npm ci
npm run build:cached

node scripts/rea.mjs --version
node scripts/rea.mjs --help
```

已有仓库时，在该目录执行 `git pull --ff-only`，再运行安装和构建命令。本文后续终端命令都在 `rea` 仓库根目录执行，无需全局安装或创建别名。保留仓库及构建产物：Codex 将从这里启动本地 MCP 服务。

这里使用 `nings/rea` 的本地构建；`npx rea-agents@latest` 使用 npm 发布版本，不保证包含本仓库尚未发布的修复。

## 2. 配置 Codex MCP

预览配置计划，确认其中的 REA 启动路径指向刚构建的本地仓库：

```bash
node scripts/rea.mjs setup --client codex --dry-run
```

确认预览计划只包含预期的 Codex 注册和 skill 后，执行：

```bash
node scripts/rea.mjs setup --client codex --yes
```

`--yes` 表示批准所选 Codex 的设置变更；这里没有使用 `--all-detected` 或 `--install-hopper`。设置会注册 `rea` MCP 服务并默认安装配套 `reverse-engineer-anything` skill。Codex 配置默认位于 `~/.codex/config.toml`；设置了非空 `CODEX_HOME` 时使用该目录。修改已有配置时保留其他设置和注释，并创建 `.rea.backup` 备份；已有备份保留最初快照。

设置完成后，**重启 Codex**，检查注册和 skill：

```bash
node scripts/rea.mjs doctor --client codex --skill --json
```

关注报告中的 `scope_checks` 与 `healthy`；未配置的可选引擎可能出现在其他诊断中。如果已安装 Codex CLI，还可以查看注册：

```bash
codex mcp get rea --json
```

注册检查不等于工具调用成功；下一步在 Codex 中实际分析示例。若终端可运行但桌面 Codex 无法启动服务，检查注册的启动命令、Node.js 路径和桌面进程环境；版本管理器的终端 `PATH` 不一定与桌面进程一致。诊断入口见 [客户端配置](./client-configuration.md#codex-and-devin) 和 [安装与设置](./installation.md)。不要移动或删除已注册的仓库；移动后应从新位置重新预览并执行设置。

## 3. 第一个分析示例

在临时目录创建模块声明和一个函数。证据文件放在输入目录之外，避免后续分析将输出当作输入：

```bash
REA_DEMO=$(mktemp -d)

cat > "$REA_DEMO/package.json" <<'EOF'
{
  "name": "rea-demo",
  "type": "module"
}
EOF

cat > "$REA_DEMO/index.js" <<'EOF'
export function greet(name) {
  return `Hello, ${name}!`;
}
EOF

node scripts/rea.mjs analyze-javascript-application "$REA_DEMO" --json \
  > "${REA_DEMO}-evidence.json"

printf '示例绝对路径：%s\n证据文件：%s\n' \
  "$REA_DEMO" "${REA_DEMO}-evidence.json"
```

CLI 结果包含应用图、来源位置和 Evidence。打开打印出的证据文件，检查 `index.js`、导出的 `greet` 函数及解析覆盖；Evidence ID 随输入身份和参数变化，无需与其他人的结果相同。

复制终端打印的**实际绝对路径**，在重启后的 Codex 聊天中发送下面的提示，将占位符替换为该路径。聊天不会自动展开终端变量 `$REA_DEMO`。

> 使用 REA MCP 的 `analyze_javascript_application` 分析 `/替换为实际示例绝对路径`，使用 `format: "directory"` 和 `detail: "summary"`。解释模块结构，确认是否观测到导出的 `greet` 函数，并给出 Evidence ID、来源位置和覆盖限制；如摘要没有函数细节，请查询保留证据的相关视图。

验证成功的依据是 **Codex 实际调用工具并返回有效 Evidence**，且结果能定位到示例的 `index.js`／`greet`。仅显示 MCP 已注册或已连接不足以证明分析调用成功。静态结果不证明函数运行过，也不证明目标行为已经动态验证。

## 4. 分析自己的项目

把示例路径替换为 JavaScript 项目目录或 Electron 的 `app.asar`：

```bash
node scripts/rea.mjs analyze-javascript-application "/absolute/path/to/app" --json \
  > /tmp/my-app-evidence.json

node scripts/rea.mjs analyze-javascript-application --help
```

大型项目的完整证据可能很大。在 Codex 中优先要求摘要，再按文件或模块查询，检查已分析范围与未知项。原生二进制深度分析需要另外配置相应 provider；JavaScript 静态分析可以独立使用。

继续阅读：[中文架构与 MCP 流程](./architecture-zh.md) · [安装与设置](./installation.md) · [CLI 与 Evidence](./cli.md)。
