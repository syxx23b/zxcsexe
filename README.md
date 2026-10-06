# zxcsexe — SCADA 安装包出包 Skill

给 AI agent 用的流程技能：把 SCADA 系列工程（`Dsh_Scada` / `Design` / `Dsh_RestAPI` / `ModbusTCP`）的 `Setup-*.exe` 出包动作收敛到一个入口，并要求跑后以日志和产物核销，不拿 exit code 充证据。

## 安装

```bash
npx skills add syxx23b/zxcsexe -g --agent qoder
```

- `-g` 装到用户目录（全局），不加则只装进当前项目。
- `--agent qoder` 指定目标 agent；Qoder 的 skill 目录是 `.qoder/skills/`（项目级）与 `~/.qoder/skills/`（全局）。
- 装完应看到 `~/.qoder/skills/zxcsexe/SKILL.md`。目录名必须全小写 `zxcsexe`，与 SKILL.md 里的 `name` 一致，否则不会被识别。

没有 Node/npx 时手动装：

```powershell
# Windows PowerShell
git clone https://github.com/syxx23b/zxcsexe.git "$env:USERPROFILE\.qoder\skills\zxcsexe"
```

```bash
# bash / Git Bash
git clone https://github.com/syxx23b/zxcsexe.git ~/.qoder/skills/zxcsexe
```

## 怎么用

装好后直接对 agent 说：「用出安装包的那个 skill 打最新的安装包」「重新打包 Dsh_RestAPI」。skill 会依次做五前置校验、后台跑 `Build-Installer.ps1` 并把输出落进 `runlogs/`、跑后读日志尾 + 核 `App_Output` 里的产物。

产物命名：`Setup-<仓库文件夹名>-YYYY.MM.DD.HHmm.exe`，统一输出到多工程共享目录（靠文件名里的工程标识区分）。

## 前置条件

- Windows + PowerShell 5.1（含中文的 `.ps1` 要存成 UTF-8 with BOM，否则报错信息乱码并中途退出）。
- .NET 8 SDK、Node + npm。
- 仓库根有 `Build-Installer.ps1` 与安装器工程（`Scada.Setup`，`ModbusTCP` 是 `ModbusTcp.Setup`）。
- `scada-web/node_modules` 以目录联接指向 OneDrive 外的依赖实体，不在同步盘里放实体目录。

## 本机坐标需要改

SKILL.md 第 0 节和排错条目里写的是这台机器的实际路径（OneDrive 下的 SCADA 目录、`C:\node_modules\` 依赖仓、共享输出目录）。换机器或换目录时，先改这几处再照着跑。

## 维护

流程有变动就改 `SKILL.md`，保持单一入口；产物命名口径变更时要同步工程里的 `INSTALL_REQUIREMENTS.md` §8。
