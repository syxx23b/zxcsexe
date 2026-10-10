# zxcsexe — SCADA 安装包出包 Skill

给任意 AI agent 用的流程抓手：把 SCADA 系列工程的 `Setup-*.exe` 出包动作收敛到一个入口（`Build-Installer.ps1`），并要求跑后读日志、核销产物，不拿 exit code 充证据。

本 skill 是**纯文档**（只有 `SKILL.md` + 本 README），不含脚本。真正干活的 `Build-Installer.ps1` 在各工程仓库里。

## 安装

skill 就是一个目录，里面放 `SKILL.md`。把目录放进你所用 agent 的 skill 目录即可 —— **不限定任何特定 agent**：

```bash
git clone https://github.com/syxx23b/zxcsexe.git <你的 agent 的 skill 目录>/zxcsexe
```

常见的 skill 目录（按你的 agent 选，不要照抄全部）：

| Agent / 运行时 | 用户级 skill 目录 | 项目级 skill 目录 |
| --- | --- | --- |
| Claude Code | `~/.claude/skills/` | `<项目>/.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `<项目>/.cursor/skills/` |
| Qoder | `~/.qoder/skills/` | `<项目>/.qoder/skills/` |
| DSH | `~/.dsh/skills/` | `<项目>/.dsh/skills/` |

Windows PowerShell / cmd 举例：

```powershell
git clone https://github.com/syxx23b/zxcsexe.git "$env:USERPROFILE\.claude\skills\zxcsexe"
```

没有 git 时也可直接下载 zip 解压到对应目录。

**两条硬要求**（多数 agent 都按此约定加载）：

1. 目录名必须与 `SKILL.md` frontmatter 里的 `name` 一致，本 skill 两者都是 `zxcsexe`。
2. `SKILL.md` 顶部必须有 `---` 包起来的 `name` 与 `description`。`description` 决定 agent 何时触发它。

## 怎么用

装好后直接对 agent 说：「用出安装包的那个 skill 打最新的安装包」「重新打包 Dsh_RestAPI」。skill 会先定位仓库并读 `Build-Installer.ps1` 头部的坐标，再依次做五项前置校验、后台跑打包并把输出落 `runlogs/`、最后读日志尾 + 核销输出目录里的产物。

产物命名：`Setup-<仓库文件夹名>-YYYY.MM.DD.HHmm.exe`，统一输出到多工程共享目录（靠文件名里的工程标识区分）。

## 前置条件

- Windows + PowerShell 5.1（含中文的 `.ps1` 要存成 UTF-8 with BOM，否则报错信息乱码并中途退出）。
- .NET 8 SDK、Node + npm。
- 目标仓库根有 `Build-Installer.ps1` 与安装器工程（`Scada.Setup`）。
- 前端依赖：`scada-web/node_modules` 以目录联接指向 OneDrive 外的依赖实体，不在同步盘放实体目录（脚本会自行处理，见 SKILL.md §1）。

## 机器坐标

路径**没有**写死在 SKILL.md 里。每个仓库 `Build-Installer.ps1` 的头部（`$installerOutputRoot`、`$webNodeModulesStore`、`$apiProject`、`$setupProject`）才是权威坐标，skill 会现场读取。换机器时改脚本头部，不用改 skill。

需要额外覆盖时，可在 skill 目录放一份 `SKILL.local.md`（同目录，已被 `.gitignore` 忽略），写法见 [`SKILL.local.md.example`](SKILL.local.md.example)。

## 维护

流程有变动就改 `SKILL.md`，保持单一入口；产物命名口径变更时要同步工程里的 `INSTALL_REQUIREMENTS.md`。
