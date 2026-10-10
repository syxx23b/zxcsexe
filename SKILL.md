---
name: zxcsexe
description: "安装包出包流程（单入口 Build-Installer.ps1 → Setup-<project>-日期.时间.exe）：先发现本工程坐标、跑前六项校验（含安装器图标必须用规范图标、不得自造）、只走一个入口，跑后必须看日志尾并核产物，不拿 exit code 充证据。当用户要求出安装包、打包、重新发布 setup.exe，或问这套流程怎么跑、怎么接新工程、安装器图标该用哪个时使用。不用于只构建前端调试包（那用 npm run build 即可），不用于安装后的运行排障，不用于改动 Build-Installer.ps1 的内部顺序。"
whenToUse: "用户要求出安装包/打包/重新发布 setup.exe，需要把新工程接入这套出包流程，或需要确认/补齐安装器图标时。"
---

# 安装包出包（Setup-*.exe）

一句话口径：**先发现本工程坐标，再只走 `Build-Installer.ps1` 这一个入口，成败以日志和产物为准，不拿 exit code 充证据。**

## 0. 本技能与坐标的边界

**坐标属于工程，不属于本技能。** 本技能不含任何机器路径，也不假设工程叫什么名字。每个工程的差异只落在三处：

| 差异点 | 落在哪 | 说明 |
| --- | --- | --- |
| 主项目 / 安装器工程 / 依赖联接 / 输出目录 | 工程根的 `Build-Installer.ps1` 顶部几行 | 出包脚本自己持有，权威 |
| 可选声明 | 工程根的 `installer.config.json` | 便于技能发现与核销；缺失时按下文默认值探测 |
| 流程顺序 | 不落任何地方 | 脚本内部顺序是准的，任何工程都一样 |

**不要**在本技能里写死工程名、盘符或用户目录；需要时去读工程。

### 通用默认值（无 `installer.config.json` 时按此探测）

| 项 | 默认 | 来源 |
| --- | --- | --- |
| 入口脚本 | `<项目根>/Build-Installer.ps1` | 约定 |
| 输出目录 | `installerOutputRoot`（读脚本第 6 行附近的赋值） | 脚本 |
| 包名 | `Setup-<仓库文件夹名>-YYYY.MM.DD.HHmm.exe` | 脚本 `$installerNameTag = Split-Path -Leaf $repoRoot` |
| 日志尾成功标志 | `Installer created:` | 脚本 |
| 工程坐标 | 脚本顶部 `$scadaWebDir` / `$apiProject` / `$setupProject` / `$webNodeModulesStore` | 脚本 |

若工程放了 `installer.config.json`，优先用它做发现与核销（字段见 `installer.config.example.json`）。**声明与脚本冲突时以脚本为准**，并把差异报给用户。

## 1. 跑前校验（缺一都会白跑，按顺序核）

按 `installer.config.json` 的开关决定是否执行某项；没有该文件时按脚本实际引用判断，跳过与工程无关的项。

1. **工具链在**：`dotnet --version`（需要 8.x）。工程有前端时再核 `node --version`。
2. **依赖联接在**（仅当脚本引用了依赖实体仓；典型是 `$webNodeModulesStore`）：`cmd //c "dir <工程>\scada-web /al"` 要看到 `<JUNCTION> node_modules` 指向同步盘之外的实体仓。没有联接要素的工程跳过此项。
3. **字体齐**（仅当脚本校验字体时）：`<工程>\scada-web\public\fonts` 必须有脚本点名的那几个 ttf 加许可文件；缺失说明前端没重新构建。
4. **安装器工程在**：脚本的 `$setupProject` 指向的 `.csproj` 存在。它的 `PreparePayload` 负责把主项目 publish 产物（加 Launcher，若有）压进 payload，并把 `wwwroot` 资源缺失当错误。
5. **图标是规范图标**：安装器工程的 `Assets\launcher.ico` 必须与全系列规范图标同源（比对 SHA256，见第 5 节）。缺失或不一致时**先从规范源复制**，不要出包、更不要自己画。
6. **工作树状态明确**：安装包嵌的是**当前源码状态**，未提交的改动会进包但事后无从对应。先 `git status` 确认，再打包。

## 2. 执行

```powershell
# 只把 <工程> 换成你这次要出包的那个工程根，其余照抄
powershell -NoProfile -ExecutionPolicy Bypass -File "<工程>\Build-Installer.ps1"
```

- 换工程**只换路径**。**不要**自己拼 `npm run build` 加 `dotnet publish` 复制流程；脚本内部顺序（前端 → 主项目 payload → Setup 单文件自包含 → 拷贝到输出目录）才是准的。
- 耗时数分钟，用后台跑并把标准输出与错误一起落进仓库已忽略的日志目录（`runlogs/`、`*.log` 通常都在 `.gitignore` 里）：先建 `runlogs`，再重定向到 `runlogs/build-installer-<时间>.log`。
- 安装器 exe 是**单文件自包含**发布（`--self-contained true` + `PublishSingleFile`）；主项目 payload 通常 `--self-contained false`。以脚本实际参数为准。
- 脚本 `finally` 一般会自动清理 `bin/obj/dist` 与临时 publish 目录，并**保留依赖联接**（旧版本删它会让打包后的 `npm run build` 报 `'tsc' 不是内部或外部命令`）。清理范围以脚本为准。

## 3. 跑后核销（这一步才是结论）

1. **读日志尾**，不是只看返回码：后台任务末尾的 echo 或收尾命令会把退出码洗成 0，exit code 不可信。日志里应出现 `Installer created: <输出目录>\Setup-<工程>-<时间>.exe`。
2. **核产物**：列出输出目录，确认新文件存在、时间戳是本次、大小与上一版**同量级**（异常小基本是 payload 没打进去）。
3. **验静态资源真进了包**：清理前主项目的 `wwwroot` 应有前端产物（若有字体校验，字体也应在）。
4. **同步盘复核**：同步盘工作区会回冲本地改动并生成 `*-设备名` 冲突副本；跑完再看一次 `git status` 与输出目录列表，确认产物和源码都还在。

## 4. 常见报错与处置

- **字体不全 / 缺许可文件**：先在 `scada-web` 跑一次构建让资源进 `wwwroot`，再打包；别去改脚本里的校验。
- **Missing dependency store**：在 `scada-web` 跑一次安装补依赖，再用工程约定的 relink 脚本重建联接。**不要**在同步盘里留实体 `node_modules`（会把几万文件拉进同步盘）。
- **打包后 `npm run build` 报 `'tsc' 不是内部或外部命令`**：依赖联接被删了，用同一个 relink 脚本恢复。
- **中文报错乱码并中途退出**（PowerShell 5.1 把 UTF-8 当 ANSI 读）：`.ps1` 要存成 **UTF-8 with BOM**。
- **带 `$` 的 PowerShell 内联命令在 Git Bash 里被吃掉**：落成 `.ps1` 再 `-File` 跑，别拼内联。

## 5. 图标（安装器 logo）规范

**一句话口径：图标全系列只有一份，从既有规范复制并核哈希；不要自己画、不要自己生成。**

安装器的 `ApplicationIcon` 与桌面快捷方式图标必须用同一份规范图标。它由本地既有安装器工程提供，是唯一权威来源。

### 规范坐标

| 项 | 值 |
| --- | --- |
| 逻辑名 | 各安装器工程的 `Assets\launcher.ico` |
| 属性名 | `.csproj` 的 `<ApplicationIcon>Assets\launcher.ico</ApplicationIcon>` |
| SHA256 前缀 | `C2A3AC94E1F7C3E5` |
| 大小 | 372526 bytes |
| 内含帧 | 16 / 24 / 32 / 48 / 64 / 128 / 256，全 32bpp |

**规范源**：在本地既有的 `Scada.Setup\Assets\launcher.ico` 里任取一份（多工程共用同一份，哈希一致）。**不要**把某个项目路径写死在新工程里当作唯一依据 —— 先扫描比对，再选一个一致的源。

### 接入步骤

1. **盘点**：扫描本地既有安装器工程，收集所有 `Assets\launcher.ico`，按 SHA256 分组。
2. **判定规范**：多数一致的那一组即规范图标。若出现两个以上不同哈希，**停下问用户**，不要自行择一。
3. **复制**：把规范图标复制到新工程的 `<Setup 工程>\Assets\launcher.ico`。
4. **核验**：复制后重新计算 SHA256，必须与规范一致；用 `System.Drawing.Icon` 试解析，确认能读出且尺寸档位齐全。

### 禁止事项

- **不要自己画、也不要程序生成图标**（包括用 SVG 光栅化、品牌色 + 文字拼接等）——除非用户明确要求并提供了设计稿。
- **不要把前端素材当安装器图标**：`scada-web\public` 下的 `favicon.svg` / `sidebar-brand-logo.svg` 是网页素材，SVG 不能直接做 Windows 图标；需要光栅化时先问用户。
- **不要把图标文件提交进版本库**：它是复制来的资产，从规范源重新获取即可，与本技能一样保持"不写死工程路径"。

## 6. 接入一个新工程

本技能对任何工程都可用，前提是该工程补齐这套出包基建：

1. 在工程根放 `Build-Installer.ps1`，照现有工程的样板改顶部坐标（主项目、安装器工程、依赖联接、输出目录），**流程顺序保持不变**。
2. 有安装器工程（如 `Scada.Setup`），负责把 payload 压成 EmbeddedResource 并产出单文件自包含 exe。
3. 有前端时，让构建产物进主项目 `wwwroot`；需要字体校验就在脚本里显式校验。
4. **按第 5 节放置规范图标**（`Assets\launcher.ico`），并在跑前校验里比对哈希。
5. 可选：在工程根放 `installer.config.json`（照 `installer.config.example.json`），便于技能自动发现与核销。

## 7. 边界（不做的事）

- 不改历史包文件名、不删输出目录里的旧产物（改名或清理要用户点名）。
- 不注册计划任务、不加定时/守护出包脚本：只在用户要求时跑一次。
- 不自动 commit、不 push 打包产生的变化；提交由用户明确要求，且提交说明写中文。
- `wwwroot`、`payload.zip` 等是生成物，应在 `.gitignore` 范围内，别手工塞进版本库。
- 不修改 `Build-Installer.ps1` 的内部步骤顺序来"绕过"报错。
