# zxcsexe — 安装包出包 Skill

给 AI agent 用的流程技能：把工程的 `Setup-*.exe` 出包动作收敛到**一个入口**（工程根的 `Build-Installer.ps1`），并要求跑后以日志和产物核销，不拿 exit code 充证据。

**不绑定任何机器路径、任何工程名、任何单一 agent。** 坐标由工程自己持有（见[每工程坐标](#每工程坐标)）。

## 安装

```bash
npx skills add syxx23b/zxcsexe -g --agent '*'
```

- `-g`（`--global`）装到**用户目录**，对所有项目可用；不加则只装进当前项目。
- `--agent '*'` 装到**所有已装 agent**；也可以只指定几个：`-a claude-code -a cursor`。
- `--all` 是 `--skill '*' --agent '*' -y` 的简写，非交互。

**`-g` 和 `--agent` 是两件独立的事**，别混淆：

| 维度 | 由谁决定 | 含义 |
| --- | --- | --- |
| 安装**范围** | `-g` | 用户目录（跨项目） vs 项目目录（随项目提交） |
| 接收 **agent** | `--agent` | 装到哪些 agent 各自的技能目录 |

所以 `--agent qoder -g` 是「装在用户目录下、但只进 Qoder 的技能目录」——**不是**所有 agent 通用。要通用就用 `'*'`。

装完想确认：`npx skills ls -g`。

### 没有 Node/npx 时手动装

```powershell
# Windows PowerShell
git clone https://github.com/syxx23b/zxcsexe.git "$env:USERPROFILE\.agents\skills\zxcsexe"
```

```bash
# bash / Git Bash
git clone https://github.com/syxx23b/zxcsexe.git ~/.agents/skills/zxcsexe
```

目录名必须全小写 `zxcsexe`，与 `SKILL.md` 里的 `name` 一致，否则不会被识别。

## 关键 agent 的技能目录

`skills` CLI 支持 80+ agent，各自目录不同。常用的：

| Agent | `--agent` | 全局路径 |
| --- | --- | --- |
| **通用 / 多 agent 共享** | `universal`、`amp`、`replit` | `<项目>/.agents/skills/`、`~/.config/agents/skills/` |
| **Codex / Cursor / Cline / Gemini CLI / Copilot / Droid / Kilo / Zed / Warp / Pi 等一批** | `codex`、`cursor`、`cline`、`gemini-cli` … | `~/.agents/skills/` |
| Claude Code | `claude-code` | `~/.claude/skills/` |
| Qoder | `qoder` | `~/.qoder/skills/` |
| Qoder CN | `qoder-cn` | `~/.qoder-cn/skills/` |
| Antigravity | `antigravity` | `~/.gemini/antigravity/skills/` |
| OpenCode | `opencode` | `~/.config/opencode/skills/` |
| Windsurf | `windsurf` | `~/.codeium/windsurf/skills/` |
| GitHub Copilot | `github-copilot` | `~/.copilot/skills/` |
| Trae / Trae CN | `trae`、`trae-cn` | `~/.trae/skills/`、`~/.trae-cn/skills/` |

完整对照表见 [vercel-labs/skills](https://github.com/vercel-labs/skills#supported-agents)，或直接 `npx skills add syxx23b/zxcsexe --list`。

### DeepSeek Harness（DSH）

DSH 不在 `skills` CLI 的官方列表里，但它**直接扫描磁盘技能根**，所以能读到本技能。它的发现顺序（`@deepseek-ai/dsh-skill-filesystem`，rank 小的优先）：

| Rank | 来源 | 路径 |
| --- | --- | --- |
| 100 | project-dsh | `<项目根>/.dsh/skills` |
| 200 | project-agents | `<项目根>/.agents/skills` |
| 300 | custom | `Config.customSkillDirs` |
| 400 | user-dsh | `<DSH_HOME>/skills`（默认 `~/.dsh/skills`，跳过其 `.system`） |
| 500 | user-agents | `<DSH_AGENTS_HOME>/skills`（默认 `~/.agents/skills`） |

**最省事的接法：装到 `~/.agents/skills/`。** 这既是 DSH 的 `user-agents` 根，也是上面一大批 agent 的共用目录 —— 一条命令同时喂饱 DSH 和 Codex/Cursor/Cline 等：

```bash
# 装到共享 agent 根（DSH 与多数 agent 都扫这里）
git clone https://github.com/syxx23b/zxcsexe.git ~/.agents/skills/zxcsexe
```

DSH 侧无需重启：技能根由 Chokidar 监视，新增/改名/删除会即时到达会话。
注意 DSH 的发现深度是**一层**（只认 `<root>/<name>/SKILL.md` 与 `<root>/<name>.md`），不支持嵌套的 `**/SKILL.md`。

## 怎么用

装好后直接对 agent 说：「用出安装包的那个 skill 打最新的安装包」「重新打包 Dsh_RestAPI」。

skill 会依次做：发现本工程坐标 → 五项前置校验 → 后台跑工程的 `Build-Installer.ps1` 并把输出落进 `runlogs/` → 跑后读日志尾 + 核输出目录里的产物。

## 每工程坐标

坐标属于**工程**，不属于本技能。差异只落在三处：

1. 工程根的 `Build-Installer.ps1` 顶部几行（主项目、安装器工程、依赖联接、输出目录）—— **权威**。
2. 可选的工程根 `installer.config.json`，便于技能自动发现与核销。样板见 [`installer.config.example.json`](./installer.config.example.json)。与脚本冲突时**以脚本为准**。
3. 流程顺序——**不落任何地方**，脚本内部顺序对所有工程一致，不要改。

接入一个新工程的做法见 `SKILL.md` 第 6 节。

## 安装器图标（logo）

**全系列只有一份图标，从既有安装器工程复制，不要自己画。**

| 项 | 值 |
| --- | --- |
| 位置 | 各安装器工程的 `Assets\launcher.ico` |
| SHA256 前缀 | `C2A3AC94E1F7C3E5` |
| 大小 | 372526 bytes |
| 内含帧 | 16 / 24 / 32 / 48 / 64 / 128 / 256（全 32bpp） |

出包前的校验会比对 SHA256；不一致或缺失就**先复制规范图标再出包**，不要用程序生成，也不要把前端的 SVG 素材（`favicon.svg` / `sidebar-brand-logo.svg`）当安装器图标 —— 那些是网页素材，且 SVG 不能直接做 Windows 图标。细节见 `SKILL.md` 第 5 节。

## 前置条件

- Windows + PowerShell 5.1（含中文的 `.ps1` 要存成 **UTF-8 with BOM**，否则报错信息乱码并中途退出）。
- .NET 8 SDK；工程有前端时另需 Node + npm。
- 工程根有 `Build-Installer.ps1` 与安装器工程（如 `Scada.Setup`）。
- 有前端且脚本引用依赖实体仓时，`scada-web/node_modules` 以目录联接指向同步盘之外的实体，不在同步盘里放实体目录。

## 维护

流程有变动就改 `SKILL.md`，保持单一入口。**改了流程一定要同步改本 README 的安装段**（`--agent` 与目录口径会随 `skills` CLI 版本变化，改前先 `npx skills add syxx23b/zxcsexe --list` 核对）。产物命名口径变更时要同步工程里的 `INSTALL_REQUIREMENTS.md` §8。
