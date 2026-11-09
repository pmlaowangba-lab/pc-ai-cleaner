---
name: pc-ai-cleaner
description: 电脑清理与 AI 环境管家（Windows 与 macOS，自动识别系统）。用于系统盘深度摸底与分级清理（用户目录、缓存、临时目录、浏览器与开发缓存，按占用从大到小排序并标注【可放心删】【按需清理】【核心依赖别碰】）；AI 模型与工具残留专项清理（Ollama、HuggingFace、ComfyUI、uv、pip、conda、npm、Homebrew、各类 Agent 临时切片与报错日志，识别沉睡模型与冗余包缓存）；本地大模型硬件与环境体检（CPU、内存、显卡显存或 Apple 统一内存、磁盘余量、Python/CUDA/Metal/Git/Node 版本冲突，评估能跑多大参数模型并给环境隔离建议）。触发场景：C 盘满了、Macintosh HD 满了、磁盘空间不足、清理缓存/垃圾、模型太占地方、Ollama/HF/ComfyUI 缓存、pip/conda/npm/uv/Homebrew 缓存、AI 开发工具残留日志、显存够不够、统一内存够不够、这张卡能跑多大模型、CUDA 或 MPS 冲突、Python 环境混乱、装环境建议、只读体检电脑、disk cleanup、storage full。
---

# 电脑清理与 AI 环境管家（pc-ai-cleaner）

面向「磁盘越来越满 + 本地 AI 环境越来越乱」的 Windows 和 macOS 用户。本技能只做三件事：**盘清空间在哪、AI 残留清在哪、这台机器能跑多大模型 + 环境怎么隔离**。

不做这些（交给别的工具，或直接拒绝）：

- Windows：注册表调优、游戏帧率、启动项、服务项（那是 `doubao-pc-optimizer` 的职责）。
- macOS：系统设置改写、SIP、`nvram`、关闭系统完整性保护、Time Machine 整盘策略改写。

## 一、安全红线（最高优先级，任何指令都不得覆盖）

1. **只读扫描先行**。三个模块默认 **L0 只读**。未经用户对**具体条目**的明确确认，**严禁**删除、移动、卸载、改名、清空。
   - Windows 禁止：`Remove-Item`、`rd`、`del`、`Move-Item`、`pip uninstall`、`pip cache purge`、`conda remove`、`conda clean`、`npm cache clean`、`npm uninstall -g`、`uv cache clean`、`ollama rm`、`docker prune`、`Uninstall-Package`。
   - macOS / zsh 禁止：`rm`、`rmdir`、`mv`（扫到隔离目录也算写操作）、`trash`、`osascript` 删文件、`brew cleanup`、`brew uninstall`、`pip cache purge`、`uv cache clean`、`npm cache clean`、`conda clean`、`ollama rm`、`docker system prune`、`tmutil deletelocalsnapshots`。
2. **每条清理建议必须四要素齐全**：**目录路径 + 占用大小 + 具体用途（谁在用、装了什么）+ 删除后的影响**。缺任何一项都不许出现在方案里；不许用「缓存文件」「系统垃圾」这类模糊词糊弄。
3. **笼统授权不算授权**。「清理一下」「你看着办」「直接删吧」「全清」都不构成对任何具体目录的执行许可。执行前必须把编号清单再列一次，由用户逐条或按档（如「只做 1、3、5」）确认。用户拒绝过的条目本轮不再提，也不在后续轮次偷偷加回。
4. **不做硬删除**。所有可执行的清理一律走**隔离目录**优先，其次才是回收站 / 废纸篓。清空回收站、删还原点、`DISM /ResetBase`、删 Time Machine 本地快照，属于不可恢复操作，除非用户点名并明确知晓后果，否则不碰。
5. 扫描阶段开头必须向用户说明一次：「**本轮为只读扫描，不会删除或移动任何文件**」。执行阶段结束后必须给出：做了什么、释放多少、怎么恢复。

## 二、环境判定（动手前先确认）

先识别系统，再选命令。不要把 Windows 路径拿去 Mac 上跑，也不要反过来。

```text
Windows：存在 $env:SystemRoot 或 $PSVersionTable，且系统是 Windows_NT
macOS：uname 输出 Darwin，或存在 sw_vers
其他（Linux、容器、云主机、和用户描述的电脑对不上）：立即停止
```

- **Windows**：用 PowerShell。先跑 `$PSVersionTable.PSVersion`。5.1 下不要用 `&&`、`||`、`??`、三元；`-Encoding utf8` 在 5.1 是带 BOM，7.x 不带。给人看的脚本统一 `utf8BOM`。路径一律 `-LiteralPath`。
- **macOS**：用 zsh。先跑 `sw_vers` 和 `uname -m`（`arm64` 是 Apple 芯片，`x86_64` 是 Intel）。路径含空格必须加引号。只读扫描里权限被 TCC 拒绝的目录记为 `denied`，不要为了扫到它们去改隐私权限。
- **对不上**（例如用户说的是自己的 Mac，当前环境却是 Linux 容器）：不要把探测结果当成那台电脑的结论。说明原因，交付一份只读脚本让用户自己跑，再基于回传结果做第二轮分析。Windows 交付 `.ps1`（UTF-8 BOM）；macOS 交付 `.sh`（`bash`/`zsh` 均可，无 `rm`/`mv`）。

## 三、四步工作流（固定顺序，不许跳步）

1. **只读摸底**：按模块 1 / 2 / 3 收集数据，一次跑完，别反复打断用户。
2. **分级报告**：按占用从大到小排序，用下面这张固定表输出，等级只用三个标签。
3. **逐项确认**：把「可放心删」和「按需清理」分别编号，给出建议默认值，请用户按编号确认。
4. **执行 + 复核**：只执行被确认的条目，结束后前后对比并给恢复方式。

### 报告固定格式

| # | 路径 | 占用 | 是什么 / 谁在用 | 删除后的影响 | 等级 |
|---|------|------|----------------|-------------|------|

三档判定标准（写报告时按这条硬标准归类，不要凭感觉）：

- **【可放心删】**：纯可再生缓存，删掉只意味着下次重新生成或重新下载，不丢配置、不丢产物、不坏软件。例：临时目录、pip/uv/npm 包缓存、Homebrew 下载缓存、浏览器 Cache、缩略图缓存、崩溃转储、运行日志、Xcode DerivedData。
- **【按需清理】**：删了要付时间成本（重新下载几个 GB / 重装环境）或与业务资产相关。例：模型权重、`.venv`/conda 环境、`node_modules`、旧版安装包、下载目录里的大文件、微信聊天缓存、ComfyUI 的 output。
- **【核心依赖别碰】**：删了软件直接坏、数据直接丢、系统起不来。例：`site-packages` 已安装包、conda base、正在使用的 env、ComfyUI `custom_nodes`、`.git`、程序配置、浏览器 Profile（登录数据、书签、历史）、`Program Files`、`/System`、`/usr`、正在被引用的 Ollama blobs。

## 四、模块 1：系统盘深度摸底与分级清理

统计结果要提示一句「**合计可能大于实际可释放**」：硬链接（conda `pkgs`、HuggingFace cache）、同一文件被多处引用都会重复计数。量之前先确认目标存在，不存在不要报成 0。

### 4.1 Windows

| 范围 | 典型路径 | 常见等级 |
|------|---------|---------|
| 用户临时目录 | `%USERPROFILE%\AppData\Local\Temp`、`C:\Windows\Temp` | 可放心删 |
| 系统缓存 | `AppData\Local\Microsoft\Windows\INetCache`、`Explorer`（缩略图）、`CrashDumps`、`DeliveryOptimization` | 可放心删 |
| 浏览器缓存 | `...\Google\Chrome\User Data\Default\Cache`、`Code Cache`、`GPUCache`；Edge 同结构 | 可放心删（**不碰** Profile 数据） |
| Office 残留 | `AppData\Local\Microsoft\Office\*.tmp`、`UnsavedFiles`、`AppData\Roaming\Microsoft\Office\Recent` | tmp 可放心删；UnsavedFiles 按需 |
| 通讯软件缓存 | 微信 `Documents\WeChat Files\*\FileStorage\{Cache,Video,Image}`、QQ 同类 | 按需（会丢聊天里的图片视频） |
| 用户目录大件 | `Downloads`、`Desktop`、`Documents` 里的旧安装包/压缩包/镜像 | 按需，**只报告不删** |
| Roaming 配置 | `AppData\Roaming\<软件名>` | 核心依赖别碰（多为配置与数据） |
| UWP 缓存 | `AppData\Local\Packages\<包名>\LocalCache` | 按需 |

```powershell
$ErrorActionPreference = 'SilentlyContinue'

# 1) 用户目录下各一级文件夹的大小排行
$rows = Get-ChildItem -LiteralPath $env:USERPROFILE -Force -Directory | ForEach-Object {
  $sum = (Get-ChildItem -LiteralPath $_.FullName -Recurse -File -Force | Measure-Object -Property Length -Sum).Sum
  [pscustomobject]@{ GB = [math]::Round($sum / 1GB, 2); Path = $_.FullName }
}
$rows | Sort-Object GB -Descending | Select-Object -First 15 | Format-Table -AutoSize

# 2) 指定缓存目录逐个量体积
$targets = @(
  "$env:USERPROFILE\AppData\Local\Temp",
  "$env:USERPROFILE\AppData\Local\Microsoft\Windows\INetCache",
  "$env:USERPROFILE\AppData\Local\CrashDumps"
)
foreach ($t in $targets) {
  if (Test-Path -LiteralPath $t) {
    $s = (Get-ChildItem -LiteralPath $t -Recurse -File -Force | Measure-Object -Property Length -Sum).Sum
    [pscustomobject]@{ GB = [math]::Round($s / 1GB, 2); Path = $t }
  }
}

"权限拒绝/占用跳过的报错数：$($Error.Count)（不影响其余统计）"
$Error.Clear(); exit 0
```

- 大目录（`AppData\Local` 这种）深扫慢，可用只读的 `robocopy "$env:USERPROFILE\AppData\Local" NULL /L /S /NJH /BYTES /XJ /R:0 /W:0` 取总字节数，比 `Get-ChildItem` 快数倍。

### 4.2 macOS

| 范围 | 典型路径 | 常见等级 |
|------|---------|---------|
| 用户缓存 | `~/Library/Caches` 下各应用子目录 | 多数可放心删；先看子目录名，整棵 `Caches` 不要一次挪走 |
| 用户日志 | `~/Library/Logs` | 可放心删 |
| 临时目录 | `$TMPDIR`（`echo "$TMPDIR"`，一般在 `/var/folders/.../T/`） | 可放心删 |
| Homebrew 下载缓存 | `brew --cache` 的返回路径 | 可放心删（执行时用 `brew cleanup`，扫描阶段只跑 `brew cleanup -n`） |
| Xcode 编译产物 | `~/Library/Developer/Xcode/DerivedData` | 可放心删（下次编译会重做） |
| 浏览器缓存 | `~/Library/Caches/Google/Chrome`、`~/Library/Caches/com.apple.Safari` | 只清 Cache / Code Cache；**不碰** `~/Library/Application Support/Google/Chrome` 里的 Profile |
| 模拟器与设备支持 | `~/Library/Developer/CoreSimulator`、`~/Library/Developer/Xcode/iOS DeviceSupport` | 按需（重下很慢） |
| 通讯软件 | `~/Library/Containers/com.tencent.xinWeChat` | 按需（会丢聊天里的图片视频） |
| 用户目录大件 | `~/Downloads`、`~/Desktop`、`~/Documents` | 按需，**只报告不删** |
| 应用配置与数据 | `~/Library/Application Support`、`~/Library/Preferences`、`~/Library/Keychains` | 核心依赖别碰 |
| iCloud | `~/Library/Mobile Documents` | 核心依赖别碰 |
| 系统 | `/System`、`/Library`、`/usr`、`/bin`、`/private/var/vm` | 核心依赖别碰 |

```bash
sw_vers
uname -m
df -h /
diskutil info / | grep -E "Volume Name|Disk Size|Container Free Space|Volume Free Space"

echo "TMPDIR=$TMPDIR"
du -hd 1 "$HOME" 2>/dev/null | sort -hr | head -n 20

targets=(
  "$HOME/Library/Caches"
  "$HOME/Library/Logs"
  "$TMPDIR"
  "$HOME/Library/Developer/Xcode/DerivedData"
  "$HOME/Downloads"
)
for t in "${targets[@]}"; do
  if [ -e "$t" ]; then
    du -sh "$t" 2>/dev/null
  fi
done

# 有 Homebrew 时只预览，不清理
if command -v brew >/dev/null 2>&1; then
  echo "brew cache: $(brew --cache)"
  du -sh "$(brew --cache)" 2>/dev/null
  brew cleanup -n
fi
```

- `du` 扫 `~/Library` 会碰到权限拒绝，把 denied 的路径单独列出来，并写明「这部分没算进合计」。
- `~/Library/Caches` 先列一级子目录占用，不要把整棵目录标成一条「可放心删」。里面偶尔有应用把自己的离线数据放进来。
- Homebrew 缓存如果已经算进 `~/Library/Caches`，报告里注明不要重复加总。

## 五、模块 2：AI 模型与工具缓存专项

路径以工具自己报告的为准，表里的位置只是默认值。扫描时优先跑：`uv cache dir`、`pip cache dir`、`npm config get cache`、`conda info --base`、`brew --cache`、`ollama list`。命令不存在就记「未安装」，不要当成 0 GB。

### 各工具路径与处置方式

| 工具 | Windows | macOS | 里面是什么 | 等级 | 正确清理方式 / 注意 |
|------|---------|-------|-----------|------|-------------------|
| Ollama 模型 | `%USERPROFILE%\.ollama\models` | `~/.ollama/models` | 权重 blob + manifests | 按需 | 先 `ollama list`、`ollama ps`；卸载用 `ollama rm <model>`，**不要手删 blobs** |
| Ollama 日志 | `%LOCALAPPDATA%\Ollama\*.log`、`%USERPROFILE%\.ollama\logs` | `~/Library/Logs/Ollama`、`~/.ollama/logs` | 运行日志 | 可放心删 | 只删日志 |
| HuggingFace | `%USERPROFILE%\.cache\huggingface` | `~/.cache/huggingface` | 模型/数据集分片 | 按需 | 可用 `huggingface-cli delete-cache`；手删后下次整包重下 |
| ComfyUI 模型 | `<ComfyUI>\models\...` | 同左，位置由用户安装决定 | **用户的工作资产** | 按需 | 绝不能当垃圾；只报告同名重复文件 |
| ComfyUI 节点 | `<ComfyUI>\custom_nodes`、`python_embeded` | `<ComfyUI>/custom_nodes`、自带 venv | 节点与内嵌环境 | 核心依赖别碰 | 删了工作流直接报节点缺失 |
| ComfyUI 临时 | `temp`、`output`、`input` | 同左 | 中间图 / 成品图 | temp 可放心删；output/input 按需 | output 默认不清 |
| uv | `%LOCALAPPDATA%\uv\cache` | `uv cache dir`（常见 `~/Library/Caches/uv`） | wheel 缓存 | 可放心删 | `uv cache clean`；项目 `.venv` 是核心依赖 |
| pip | `%LOCALAPPDATA%\pip\cache` | `pip cache dir`（常见 `~/Library/Caches/pip`） | wheel/http 缓存 | 可放心删 | `pip cache purge` |
| conda 包缓存 | `<conda>\pkgs` | `$(conda info --base)/pkgs` | tarball + 解压副本 | 可放心删 | `conda clean --all`，**不要手删 `pkgs`**（可能被 env 硬链接） |
| conda 环境 | `<conda>\envs\*` | `$(conda info --base)/envs/*` | 一套套 Python | 按需 | `conda env list`；不用的用 `conda env remove -n <name>` |
| npm | `%LOCALAPPDATA%\npm-cache\_cacache` | `~/.npm/_cacache` | tarball 缓存 | 可放心删 | `npm cache clean --force`；全局 `node_modules` 别碰 |
| pnpm / yarn | `%LOCALAPPDATA%\pnpm\store`、`Yarn\Cache` | `pnpm store path`、`~/Library/Caches/Yarn` | 内容寻址仓库 | 按需 | `pnpm store prune` |
| Homebrew | 无 | `brew --cache` | 已下载的 bottle | 可放心删 | 确认后 `brew cleanup`；**不要** `brew uninstall` |
| Agent 日志与轨迹 | `%USERPROFILE%\.codex`、`.claude`、`.cursor`、`.doubao` | `~/.codex`、`~/.claude`、`~/.cursor`、`~/.doubao` | 会话轨迹、临时切片、日志 | 日志可放心删；会话轨迹按需 | 轨迹可能是复盘依据，先按时间和大小列出 |

### 沉睡模型与冗余包识别

- 沉睡判定（三条同时满足才标）：体积 **≥ 1GB** + 最后修改时间 **> 90 天** + `ollama list` 或项目目录里查不到引用。Windows 可能禁用 LastAccessTime，macOS 的 `du`/`stat` 也经常只能拿到 mtime。报告里写明「按修改时间估算」。
- 冗余包判定：pip 缓存里同名包的多个 wheel、conda `pkgs` 里同一包多版本、HuggingFace `hub` 下同一 repo 的多个 snapshot。
- 「沉睡模型」单列一节，按大小排序，标注重下成本（如 8GB @ 100Mbps ≈ 11 分钟）。

Windows 只读列目录：

```powershell
$roots = @(
  "$env:USERPROFILE\.ollama\models",
  "$env:USERPROFILE\.cache\huggingface\hub",
  "$env:LOCALAPPDATA\uv\cache",
  "$env:LOCALAPPDATA\pip\cache"
)
foreach ($r in $roots) {
  if (Test-Path -LiteralPath $r) {
    Get-ChildItem -LiteralPath $r -Force -Directory | ForEach-Object {
      $s = (Get-ChildItem -LiteralPath $_.FullName -Recurse -File -Force | Measure-Object Length -Sum).Sum
      [pscustomobject]@{ GB = [math]::Round($s/1GB,2); LastWrite = $_.LastWriteTime.ToString('yyyy-MM-dd'); Path = $_.FullName }
    } | Sort-Object GB -Descending | Select-Object -First 8
  }
}
```

macOS 只读列目录：

```bash
ollama list 2>/dev/null || echo "ollama: 未安装"
python3 - <<'PY'
import os, time
roots = [
    os.path.expanduser("~/.ollama/models"),
    os.path.expanduser("~/.cache/huggingface/hub"),
]
for r in roots:
    if not os.path.isdir(r):
        print(f"MISSING {r}")
        continue
    rows = []
    for name in os.listdir(r):
        p = os.path.join(r, name)
        if not os.path.isdir(p):
            continue
        total = 0
        mtime = 0
        for dirpath, _, files in os.walk(p):
            for fn in files:
                fp = os.path.join(dirpath, fn)
                try:
                    st = os.stat(fp)
                except OSError:
                    continue
                total += st.st_size
                mtime = max(mtime, st.st_mtime)
        rows.append((total, mtime, p))
    rows.sort(reverse=True)
    for total, mtime, p in rows[:8]:
        gb = total / (1024**3)
        day = time.strftime("%Y-%m-%d", time.localtime(mtime)) if mtime else "-"
        print(f"{gb:8.2f} GB  {day}  {p}")
PY
uv cache dir 2>/dev/null || true
pip3 cache dir 2>/dev/null || pip cache dir 2>/dev/null || true
npm config get cache 2>/dev/null || true
conda info --base 2>/dev/null || echo "conda: 未安装"
```

uv / pip 的真实缓存目录用上面命令的输出再去 `du -sh`，不要猜。

## 六、模块 3：本地大模型硬件与环境体检

### 6.1 Windows 硬件

```powershell
Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors
Get-CimInstance Win32_ComputerSystem | Select-Object TotalPhysicalMemory, Manufacturer, Model
Get-CimInstance Win32_VideoController | Select-Object Name, DriverVersion, AdapterRAM
nvidia-smi --query-gpu=name,memory.total,memory.used,driver_version --format=csv
Get-PSDrive -PSProvider FileSystem | Select-Object Name,
  @{n='总GB';e={[math]::Round(($_.Used+$_.Free)/1GB,1)}}, @{n='剩余GB';e={[math]::Round($_.Free/1GB,1)}}
```

- `AdapterRAM` 对 NVIDIA 常读错（32 位溢出），**显存必须用 `nvidia-smi`**。没有 NVIDIA 时退化为 `Get-CimInstance`，并说明精度有限。
- 内存同时报物理总容量和当前可用，并检查是否开了虚拟内存（CPU offload 依赖它）。

Windows 显存对照（Q4_K_M，上下文 4k–8k）：

| 模型规模 | 权重占用 | 加上 KV 与桌面占用后的建议显存 |
|---------|---------|--------------------------------|
| 3B | 约 2–2.5GB | 4GB 起 |
| 7–8B | 约 4.5–5.5GB | 8GB 起 |
| 13–14B | 约 8–9.5GB | 12GB 起 |
| 32B | 约 18–20GB | 24GB 起 |
| 70B | 约 40–45GB | 48GB 起（消费级基本靠 CPU offload） |

判定口径：可用显存 **≥ 权重 + 20% 余量 + 1.5GB 桌面占用** → 流畅；**仅 ≥ 权重 + 5%** → 勉强（降上下文、降量化、关浏览器）；否则 → CPU offload，速度通常掉一个数量级，直接说清不建议。同时给「最舒服的档位」和「勉强能试的档位」。

### 6.2 macOS 硬件

```bash
sysctl -n machdep.cpu.brand_string
sysctl -n hw.memsize
sysctl -n hw.ncpu
uname -m
system_profiler SPHardwareDataType SPDisplaysDataType
memory_pressure 2>/dev/null | head -n 20
df -h /
```

- `uname -m` 为 `arm64`：内存是统一内存，CPU 和 GPU 共用。不要把 `system_profiler` 里的 VRAM 数字再加一次到总内存上。给模型档位时，先扣掉大约 **8GB** 给系统和桌面，剩下的才是能放权重的空间。
- `x86_64`：按独显显存算；没有独显就按 CPU 推理，直接说明会很慢。
- 只有真的有 NVIDIA（极少见）才跑 `nvidia-smi`。Apple 芯片不看 CUDA。

Apple 芯片统一内存对照（Ollama / llama.cpp Metal，Q4_K_M，上下文 4k–8k，已考虑系统占用）：

| 统一内存 | 最舒服 | 勉强能试 |
|---------|--------|----------|
| 8GB | 3B | 7B 很紧，不建议当日常 |
| 16GB | 7B–8B | 14B，要降上下文 |
| 24GB | 13B–14B | 32B Q4 |
| 32GB | 32B Q4 | 70B 需更狠的量化 |
| 64GB 及以上 | 70B Q4 | 更大参数看量化 |

装了 PyTorch 时补一条只读检查：`python3 -c "import torch; print(torch.__version__, getattr(torch.backends, 'mps', None) and torch.backends.mps.is_available())"`。MPS 可用就说明能走 Apple GPU；不可用就说明当前解释器里的 torch 不是 Metal 构建。

### 6.3 环境版本与冲突

Windows：

```powershell
py -0p
where.exe python; where.exe node; where.exe git
python -c "import sys; print(sys.version); print(sys.executable)"
python -c "import torch; print(torch.__version__, torch.version.cuda)"
nvcc --version; nvidia-smi
node -v; npm -v; git --version
$env:Path -split ';' | Where-Object { $_ } | Group-Object | Where-Object Count -gt 1
```

macOS：

```bash
which -a python3 || true
which -a node || true
which -a git || true
python3 -c "import sys; print(sys.version); print(sys.executable)"
python3 -c "import torch; print(torch.__version__)" 2>/dev/null || echo "当前 python3 未装 torch"
node -v 2>/dev/null || echo "node: 未安装"
npm -v 2>/dev/null || true
git --version 2>/dev/null || echo "git: 未安装"
command -v brew >/dev/null 2>&1 && brew --prefix || echo "Homebrew: 未安装"
```

典型冲突：

- **多个 Python**：`where` / `which -a` 出现 2 条以上，且 `sys.executable` 不是用户以为的那个 → 包装错解释器。macOS 上 `/usr/bin/python3` 是系统自带的，受 PEP 668 限制，**不要往里面 pip install**。
- **Windows CUDA 三件套**：`nvidia-smi` 是驱动支持的最高 CUDA，`nvcc` 是本地 toolkit，`torch.version.cuda` 是 wheel 自带 runtime。判据是**驱动 ≥ torch 的 cu 版本**。只有要自己编译扩展时才需要本地 toolkit。
- **macOS 没有这套 CUDA 判据**。Apple 芯片看 Metal / MPS；Intel Mac 若无独显，就告诉用户本地大模型会走 CPU。
- **全局污染**：conda base 或系统 Python 里堆了大量项目包、同时存在多个 torch → 今天能跑明天崩。
- **Node 多版本**：`node -v` 和项目 `package.json` 的 engines 不符，且没有 nvm / fnm。
- **PATH 顺序**：同名命令重复，调用到的不是预期版本。

### 环境隔离安装建议（给方案，不代装）

- Python：每个项目一个 `uv venv`，或 `conda create -n <项目名> python=3.11`。不往 base、系统 Python、macOS 自带 `/usr/bin/python3` 装项目包。
- Windows CUDA / PyTorch：优先用官方 wheel（自带 runtime），按驱动选 cu 标签。
- macOS：优先用带 MPS 的官方 wheel，或 Ollama 自带的 Metal 推理。不要为了 CUDA 去装一套用不上的 toolkit。
- ComfyUI：独立 env + 固定 torch 版本；`custom_nodes` 升级前先备份。
- Node：用 fnm 或 nvm 切版本，不把项目依赖装进全局。
- 建议写成「清单 + 命令示例」，等用户确认后再代执行。安装属于写操作。

## 七、执行与复核（第四步细则）

1. 执行前把确认条目重排成执行清单：`编号 | 路径 | 预计释放 | 方式`，并声明「只处理这些，其余不动」。
2. 删除方式优先级：**移到隔离目录** > 回收站或废纸篓 > 直接删（仅在用户点名「直接删，我知道不可恢复」时）。
   - Windows 隔离目录：`%USERPROFILE%\_ai_cleaner_quarantine\<YYYYMMDD>\<原路径结构>`
   - macOS 隔离目录：`~/_ai_cleaner_quarantine/<YYYYMMDD>/<原路径结构>`
3. 逐条写 manifest（隔离目录下的 `manifest.csv`：时间、原路径、字节数、方式、状态），这是恢复依据。
4. macOS 上移入废纸篓用 Finder，不用 `rm`：

```bash
osascript -e 'tell application "Finder" to delete POSIX file "/替换成真实绝对路径"'
```

5. 复核：前后磁盘剩余对比、目标目录剩余体积、失败条目（占用中 / 权限拒绝）单独列出并写明「未删除」。不许重试强删，不许改权限硬来，不许 `sudo`。
6. 收尾话术：做了什么 → 释放多少（人话，如「腾出 12.4GB」）→ 怎么恢复（隔离目录路径，建议保留 7 天）→ 1–2 条无法代劳的建议（加硬盘、换更大内存的机器、把模型目录迁到数据盘并做链接）。

## 八、绝对不做（即使用户要求也要拒绝并解释）

- 盘符根、用户主目录根、`AppData` 根、`~/Library` 根、`Program Files`、`Windows` / `System32` / `WinSxS`、`/System`、`/usr`、`/bin`、注册表、`defaults delete`。
- 用户个人文件（桌面 / 文档 / 图片 / 下载 / 照片图库）——只报告，不代删。
- 正在被进程占用的文件、正在运行的模型（先 `ollama ps`，提示关掉再动）。
- `site-packages`、conda base、正在使用的 env、`custom_nodes`、`.git`、项目 `.venv` / `node_modules`（只报告）。
- 浏览器 Profile、钥匙串、密码库、OneDrive / iCloud / 坚果云同步目录。
- 安全软件、驱动、系统组件、SIP、`nvram`、`csrutil`。
- 未逐条确认的 `pip uninstall`、`conda remove`、`npm -g uninstall`、`brew uninstall`、`ollama rm`。
- `DISM /ResetBase`、清空回收站、删系统还原点、删 Time Machine 本地快照、`bcdedit` 的任何写操作。
- 替用户下载安装「清理大师 / 加速器」类第三方程序。

## 九、沟通与话术模板

确认请求（一次问完）：

> 扫描完成，共发现 **N 项**可处理，合计约 **X GB**（实际释放可能略少，存在硬链接重复计数）。
> **A 档【可放心删】**（建议全清，不影响使用）：1 … 2 … 3 …
> **B 档【按需清理】**（请挑）：4 … 5 …
> **C 档【核心依赖别碰】**（仅告知，不会动）：…
> 回复编号或「A 档全清 + 4」即可；默认不动 B/C 档。

结论用大白话加数字，不堆命令输出。不承诺具体提速倍数。硬件瓶颈如实说：8GB 显存跑 32B、16GB 统一内存跑 32B，都要说「能加载但体验差」或直接不建议。

## 十、常见坑

- Windows PowerShell 5.1 不支持 `&&`、`??`、三元。只读扫描用 `$Error.Clear(); exit 0` 收尾，并在报告里说明有 N 个目录因权限跳过。
- 中文路径：Windows 给人看的脚本存 `utf8BOM`，避免把中文路径拼进 `cmd /c`。macOS 终端默认 UTF-8，路径仍要加引号。
- 路径超长或 `du` 权限不足：报告「无法完整统计」，不要报 0。
- 符号链接 / junction（HuggingFace、conda 常见）会重复计数。Windows 必要时加 `-Attributes !ReparsePoint`。macOS `du` 默认不重复计算硬链接，但同一缓存若用两个父目录各扫一次仍会在合计里加两次，报告里要扣掉。
- 目录大小 ≠ 可释放空间。Windows 的还原点、休眠文件、页面文件，macOS 的本地快照、`sleepimage`、可清除空间（Purgeable），都不在这套「删目录」统计里，需要单独提一句，而且默认不动。
- 模型文件删了但 `ollama list` 仍显示 → manifests 残留，用 `ollama rm`，不要手删。
- macOS 上 `~/Library/Caches` 不等于「全是垃圾」。先列子目录，再分级。

## 十一、技能协作

- Windows 上的系统级性能、游戏帧率、启动项、服务、注册表 → 交给 `doubao-pc-optimizer`，本技能不越界。
- 要把扫描结果落成表格台账 → 用用户已有的表格技能，便于长期跟踪。
- 用户要求「生成一个我能自己跑的只读扫描脚本」→ Windows 输出 `.ps1`（UTF-8 BOM，无 `pause`，结尾打印摘要）；macOS 输出 `.sh`。两份脚本都只能是只读，里面不得出现删除或移动语句。
