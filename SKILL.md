---
name: zxcsexe
description: "SCADA 系列工程的安装包出包流程（Build-Installer.ps1 → App_Output 里的 Setup-<仓库名>-YYYY.MM.DD.HHmm.exe）：定位仓库、跑前五项前置校验、后台执行、跑后读日志尾并核销产物。当用户要求出安装包、打包、重新发版 setup.exe、或问这套流程怎么跑时使用。不用于只构建前端调试包（那用 npm run build 即可），不用于安装后的运行排障。"
---

# SCADA 安装包出包（Setup-*.exe）

一句话口径：**只走 `Build-Installer.ps1` 这一个入口，成败以日志尾和产物核销为准 —— 不看 exit code 就报错。**

## 0. 先定位坐标，不要假设路径

**每个仓库的 `Build-Installer.ps1` 头部就是这台机器上的权威坐标。** 任何路径都要从脚本里读出来，不要凭记忆或从别的仓库套用 —— 三个仓库的写法并不一致。

脚本头部普遍长这样（以 `Dsh_Scada` 为例）：

```powershell
$repoRoot = Split-Path -Parent $MyInvocation.MyCommand.Path   # 仓库根：脚本自己推导，永远正确
$installerNameTag = Split-Path -Leaf $repoRoot                # 产物名里的工程标识 = 仓库文件夹名
$installerOutputRoot = '<共享输出目录>'                        # ← 写死的绝对路径，逐仓库核对
$webNodeModulesStore = '<依赖实体存放点>'                       # ← 写死的绝对路径，且并非每个仓库都有
$apiProject = Join-Path $repoRoot '<Api 工程>.csproj'          # ← 工程名逐仓库不同
$setupProject = Join-Path $repoRoot 'Scada.Setup\Scada.Setup.csproj'
```

定位步骤：

1. **找仓库**：从当前工作区向上找 `Build-Installer.ps1`；找不到就在用户机器上的 SCADA 工程目录里枚举同名脚本，把候选列给用户选。**不要猜仓库名。**
2. **读头部**：读该脚本前 30 行，抄出 `$installerOutputRoot`、`$webNodeModulesStore`（若有）、`$apiProject`、`$setupProject` 的真实值。
3. **判适用**：脚本会 derive `$repoRoot`，所以换机器/换目录时**只有头部那几个绝对路径需要改**。若读出的路径在本机不存在，停下报告并让用户确认，不要自行改成猜测路径。

各仓库的差异（仅作核对提示，不代替读脚本）：

| 仓库 | `$apiProject` | `$webNodeModulesStore` |
| --- | --- | --- |
| `Dsh_Scada` | `Scada.Api\Scada.Api.csproj` | 有 |
| `Design` | `Scada.Api\Scada.Api.csproj` | 有（用 `Join-Path` 拼） |
| `Dsh_RestAPI` | `Scada.RestApi\Scada.RestApi.csproj` | **无**（不涉及前端依赖库） |

## 1. 跑前校验（缺一都会白跑，按顺序核）

1. **dotnet 8 SDK 在**：`dotnet --version` 应报 8.x（实证跑通的是 8.0.412）。
2. **前端依赖就位**（仅涉及前端的仓库）：`scada-web\node_modules` 存在，且是指向 `$webNodeModulesStore` 的**目录联接（Junction）**。确认方式：`(Get-Item -Force '<link>').LinkType` 应报 `Junction`，`(Get-Item -Force '<link>').Target` 应等于存放点。若它是实体目录（`npm ci` 之后、或 OneDrive 同步把它还原成了实体），脚本会自己整体搬到存放点再建联接，不删你新装的依赖 —— 不需要手工处理。
3. **字体齐**：`scada-web\public\fonts` 必须有 `HarmonyOS_Sans_SC_Regular.ttf`、`_Medium.ttf`、`_Bold.ttf` 加 `LICENSE.txt`。脚本自己会校验并硬失败，这一步只是提前说清。
4. **安装器工程在**：`$setupProject` 指向的 `Scada.Setup.csproj` 存在（个别工程可能是别的名字）。它的 `PreparePayload` 目标会把 Api 的 publish 产物加 Launcher 打成 `payload.zip` 内嵌进 EXE，并把 `wwwroot/fonts/*.ttf` 缺失当错误。
5. **工作树状态已知**：安装包编的是**当前源码状态**，未提交的改动会进包。先看 `git status`，跟用户确认再打。

## 2. 执行

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "<repoRoot>\Build-Installer.ps1"
```

- **换工程只改路径。** 不要自己拼 `npm run build` 加 `dotnet publish` 复刻流程；脚本内部的顺序（前端 → Api publish → payload.zip → Setup 单文件自包含 → 拷到输出目录 → finally 清理）才是真的。
- 耗时数分钟：后台跑，输出落盘。先把输出和错误一起写进日志，别只依赖 stdout：

```powershell
New-Item -ItemType Directory -Force -Path "<repoRoot>\runlogs" | Out-Null
powershell -NoProfile -ExecutionPolicy Bypass -File "<repoRoot>\Build-Installer.ps1" *> "<repoRoot>\runlogs\build-installer-<时间戳>.log"
```

- `runlogs/`、`*.log` 都在 `.gitignore` 里，落盘不留痕。
- 脚本 `finally` 会自动清 `bin`/`obj`/`dist`、临时 publish 目录、`scada-api-9001.log`，**但保留 node_modules 联接**（旧版本连它一起删，会导致之后 `npm run build` 报 `'tsc' 不是内部或外部命令`）。

## 3. 跑后核销（这一步才是结论）

1. **读日志尾，把 `Installer created:` 当成功凭证**：脚本用了 `$ErrorActionPreference='Stop'`，且**只在 EXE 真的拷进输出目录之后**才打印这一行 —— 反过来，没看到这行就不算成功。返回码只是参考，不是成功证明（后台任务的收尾命令可能把它洗成 0）。
2. **核产物**：列 `<installerOutputRoot>`，确认新文件存在、时间戳是本次、大小与上一版同量级（异常小基本是 payload 没打进去）。
3. **验载体真进了包**：清理前 `<api 工程>\wwwroot\fonts` 下应有三个 ttf（由前端构建从 `public/fonts` 复制而来）。
4. **OneDrive 复核**：同步盘上的改动可能被回滚并生成 `*-设备名` 冲突副本。跑完再看一次 `git status` 并重新列一次输出目录，确认产物和源码都还在。

## 4. 常见报错与处置

- `HarmonyOS Sans SC 字体不全` / `缺少字体许可文件`：先在 `scada-web` 跑一次构建，让字体进 wwwroot，再打包。别改脚本里的校验。
- `Missing dependency store '<store>'`：在 `scada-web` 跑 `npm ci`（让依赖落地），再重跑打包脚本 —— 脚本自己会把实体目录搬到存放点并建联接。若机器上另有 relink 脚本，用它也可以，但**不要**把 `npm install` 的实体目录留在同步盘里（会把几万个文件重新拉进 OneDrive）。
- 打包后 `npm run build` 报 `'tsc' 不是内部或外部命令`：`node_modules` 联接没了，按上一条重建。
- 中文报错乱码并中途退出（PowerShell 5.1 把 UTF-8 当 ANSI 读）：`.ps1` 存成 **UTF-8 with BOM**。
- 带双引号的 PowerShell 内联命令在 Git Bash 里被吞掉：落盘成 `.ps1` 后用 `-File` 跑，别拼内联。

## 5. 边界（不做的事）

- 不改父级文件名、不删输出目录里的旧产物（改名或清理必须用户点名）。
- 不注册计划任务、不加定时/守护出包脚本：只在用户要求时跑一次。
- 不自动 `commit`、不 `push` 打包产生的变化；提交必须用户明确要求，提交说明写中文。
- `wwwroot`、`payload.zip`、`runlogs` 等是生成物，留在 `.gitignore` 范围内，别手工进版本库。
