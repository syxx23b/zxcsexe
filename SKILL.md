---
name: zxcsexe
description: "SCADA 系列工程共用的安装包出包流程（Build-Installer.ps1 → App_Output 里的 Setup-<project>-日期.时间.exe）：一条标准命令、跑前五项前置校验、跑后必须看日志尾并核产物。当用户要求出安装包、打包、重新发布 setup.exe、或问这套流程怎么跑时使用。不用于只构建前端调试包（那用 npm run build 即可），不用于安装后的运行排障。"
---

# SCADA 安装包出包（Setup-*.exe）

一句话口径：**只走 `Build-Installer.ps1` 这一个入口，成败以日志和产物为准，不拿 exit code 充证据。**

## 0. 适用工程与坐标

- 共用这套脚本的仓库（各自仓库根都有同名 `Build-Installer.ps1`）：`Dsh_Scada`、`Design`、`Dsh_RestAPI`、`ModbusTCP`（均在 `C:\Users\syxxz\OneDrive\SCADA\` 下）。
- 统一输出目录：`C:\Users\syxxz\OneDrive\SCADA\App_Output`（多工程共享，靠文件名里的工程标识区分，禁止落到仓库根）。
- 命名规则：`Setup-<project>-YYYY.MM.DD.HHmm.exe`，`<project>` 是仓库文件夹名（脚本里 `$installerNameTag = Split-Path -Leaf $repoRoot`），例：`Setup-Dsh_Scada-2026.10.07.0310.exe`。
- 口径来源：`INSTALL_REQUIREMENTS.md` §8（Installer Output）。

## 1. 跑前校验（缺一都会白跑，按顺序核）

1. **dotnet 8 SDK 在**：`dotnet --version`（实测本机 8.0.425）。
2. **依赖联接在**：`cmd //c "dir <repo>\scada-web /al"` 要看到 `<JUNCTION> node_modules` 指向 `C:\node_modules\<project>\scada-web\node_modules`。依赖实体存放在 OneDrive 外，仓库里只是联接（INSTALL_REQUIREMENTS §10）。
3. **字体齐**：`<repo>\scada-web\public\fonts` 必须有 `HarmonyOS_Sans_SC_{Regular,Medium,Bold}.ttf` 加 `LICENSE.txt`；脚本自己会校验并直接失败，缺了说明前端没重新构建。
4. **安装器工程在**：`Scada.Setup\Scada.Setup.csproj`（ModbusTCP 是 `ModbusTcp.Setup`）。它的 `PreparePayload` 会拷 `Scada.Api\bin\Release\net8.0\win-x64\publish` 加 Launcher 进 payload，压成 `payload.zip` 当 EmbeddedResource，并把 `wwwroot/fonts/*.ttf` 缺失当错误。
5. **工作树已提交或明确未提交也能接**：安装包嵌的是当前源码状态，未提交的改动会进包但事后无从对应；先确认再打包。

## 2. 执行

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Users\syxxz\OneDrive\SCADA\Dsh_Scada\Build-Installer.ps1"
```

- 换工程只改路径。**不要**自己拼 `npm run build` 加 `dotnet publish` 复制流程；脚本内部顺序（前端→Api payload→Setup 单文件自包含→拷贝到 App_Output）才是准的。
- 耗时数分钟，用后台跑并把输出落进仓库已忽略的日志目录（`runlogs/`、`*.log` 都在 .gitignore 里）：先 `mkdir -p runlogs`，再把标准输出和错误一起重定向到 `runlogs/build-installer-<时间>.log`。
- 脚本 `finally` 会自动 `Clear-BuildOutputs`：清 `bin/obj/dist`、临时 publish 目录、`scada-api-9001.log`，**保留 node_modules 联接**（旧版本删它会让打包后的 `npm run build` 报 'tsc' 不是内部命令）。

## 3. 跑后核销（这一步才是结论）

1. **读日志尾**，不是只看返回码：后台任务末尾的 echo 或收尾命令会把退出码洗成 0，exit code 不可信。日志里应出现 `Installer created: C:\...\App_Output\Setup-<project>-<时间>.exe`。
2. **核产物**：`cmd //c dir "C:\Users\syxxz\OneDrive\SCADA\App_Output"`，确认新文件存在、时间戳是本次、大小与上一版同量级（几十 MB；异常小基本是 payload 没打进去）。
3. **验字体真进了包**：清理前的 `Scada.Api\wwwroot\fonts` 应有三个 ttf（由前端构建从 `public/fonts` 复制而来）。
4. **OneDrive 复核**：本工作区多设备同步会回冲本地改动并生成 `*-设备名` 冲突副本；跑完再 `git status` 加重新列一次 `App_Output`，确认产物和源码都还在。

## 4. 常见报错与处置

- `HarmonyOS Sans SC 字体不全` / `缺少字体许可文件`：先在 `scada-web` 跑一次构建让字体进 wwwroot，再打包；别去改脚本里的校验。
- `Missing dependency store 'C:\node_modules\...'`：在 `scada-web` 跑 `npm ci`，再 `powershell -NoProfile -ExecutionPolicy Bypass -File C:\node_modules\relink-node-modules.scada.ps1`。**不要**在同步盘里 `npm install` 留实体目录（会把 2.5 万文件重新拉进 OneDrive）。
- 打包后 `npm run build` 报 `'tsc' 不是内部或外部命令`：node_modules 联接被删了，跑同一个 relink 脚本恢复。
- 中文报错信息乱码并中途退出（PowerShell 5.1 把 UTF-8 当 ANSI 读）：`.ps1` 存成 **UTF-8 with BOM**。
- 带美元符的 PowerShell 内联命令在 Git Bash 里被吃掉：落成 `.ps1` 再 `-File` 跑，别拼内联。

## 5. 边界（不做的事）

- 不改历史包文件名、不删 `App_Output` 里的旧产物（改名或清理要用户点名）。
- 不注册计划任务、不加定时/守护出包脚本：只在用户要求时跑一次。
- 不自动 commit、不 push 打包产生的变化；提交由用户明确要求，且提交说明写中文。
- `wwwroot`、`payload.zip` 等是生成物，已在 .gitignore 范围内，别手工塞进版本库。
