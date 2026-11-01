# Anti-Local-LLM-Deleted-Your-Computer

> **使用每個本地 AI 加上任何 Harness，都請在你的 Harness 加上文中這些防護規則。你可能會感謝我，也希望你永遠不會遇到這種事情。**

本專案提供可直接部署於各類 AI Agent 框架（如 Hermes、OpenClaw、Claude Code 等）的系統提示詞（`AGENTS.md` / `MEMORY.md`）與安全檔案操作技能（`SKILL.md`），防止本地端中小型語言模型在執行磁碟清理或檔案管理時發生毀滅性的誤刪災難。

---

## 目錄

- [技術風險提示：本地端 AI Agent 誤刪案例分析](#技術風險提示本地端-ai-agent-誤刪案例分析)
- [防範架構設計：為什麼不在 Prompt 裡寫「禁止指令」？（防粉紅大象效應）](#防範架構設計為什麼不在-prompt-裡寫禁止指令防粉紅大象效應)
- [建議部署之安全防護措施](#建議部署之安全防護措施)
- [Windows 防護配置（通用版）](#windows-防護配置通用版)
  - [1. AGENTS.md / MEMORY.md 正向記憶規範 (Windows)](#1-agentsmd--memorymd-正向記憶規範-windows)
  - [2. SKILL.md 安全清理技能與底層攔截器 (Windows)](#2-skillmd-安全清理技能與底層攔截器-windows)
- [macOS 防護配置（通用版）](#macos-防護配置通用版)
  - [1. AGENTS.md / MEMORY.md 正向記憶規範 (macOS)](#1-agentsmd--memorymd-正向記憶規範-macos)
  - [2. SKILL.md 安全清理技能與底層攔截器 (macOS)](#2-skillmd-安全清理技能與底層攔截器-macos)

---

## 技術風險提示：本地端 AI Agent 誤刪案例分析

近期在 Windows 環境中，測試由本地端 27B 參數規模之大型語言模型（LLM）驅動 AI Agent 進行磁碟空間清理與檔案搬移作業時，發生非預期的系統環境損毀與桌面檔案遺失事件。在該次操作中，Agent 除了清除目標快取外，亦誤將使用者桌面（`Desktop`）清空，並刪除了系統已安裝之 `.NET Desktop Runtime`、`MSI Afterburner` 部分組件及 `uv` 工具鏈目錄，同時清空了系統資源回收筒。

經檢視 Agent 的推理紀錄（Thinking Log）與其生成的 PowerShell 腳本，可歸納出本地端模型在執行檔案系統操作時存在以下四項潛在風險：

### 1. 多輪對話與分批執行之上下文對齊偏差
當使用者以項目編號（例如指定保留特定編號、將某編號內容搬移至 `Downloads`）授權操作時，模型在跨輪次與分批（Batch 1 至 Batch 4）執行過程中容易出現上下文記憶混淆，於推理階段無法確切對應編號所屬之實際路徑。此外，模型雖於前期檢查發現 `.NET Runtime` 為已安裝之系統執行環境而非廢棄安裝包，但在後續批次執行時未能維持該判斷，導致系統組件遭移除並引發常駐程式啟動異常。

### 2. 推理階段（Thinking）對腳本瑕疵的合理化傾向
從推理紀錄可見，模型在發送執行指令前已察覺腳本邏輯存在瑕疵（例如在執行 `uv cache clean` 後，腳本接著刪除了整個 `AppData\Local\uv` 母目錄）。然而，中小型模型在識別出潛在風險時，有時未能主動中斷流程並修正程式碼，而是傾向在推理過程中自行合理化既有操作並照常執行。

### 3. 腳本編碼差異與路徑變數退化問題
Agent 工具預設寫入的無 BOM UTF-8 `.ps1` 腳本，在透過舊版 Windows PowerShell 5.1（`powershell.exe`）執行時，若涉及中文路徑（如桌面下的中文子資料夾），容易因預設語系（CP950/Big5）解碼不相容而導致配對失敗或變數為空值。一旦子目錄變數失效，原本針對特定子資料夾的操作便可能退化為對整個桌面根目錄（`Desktop`）執行，進而清空整個桌面（本次案例中，桌面上僅存內部包含 Windows 系統保留檔名 `nul` 之資料夾因底層限制而未被移除）。

### 4. 預設採用繞過資源回收筒之強制刪除指令
在未受明確規則限制的情況下，模型生成的清理腳本傾向定義 `Remove-Item -Recurse -Force -ErrorAction SilentlyContinue` 函式以強制遞迴刪除目錄，並附加 `Clear-RecycleBin -Force` 指令清空資源回收筒。此類指令直接繞過作業系統的回收筒緩衝機制，在 SSD 環境下會觸發 TRIM 指令，大幅增加事後資料救援的難度。

---

## 防範架構設計：為什麼不在 Prompt 裡寫「禁止指令」？（防粉紅大象效應）

在設計 AI Agent 的安全規則時，若在常駐的系統提示詞（`AGENTS.md` / `MEMORY.md`）中寫入大量負向禁令（例如：`NEVER use Remove-Item -Force`、`NEVER delete Desktop`），在 Transformer 的自注意力機制（Self-Attention）下反而會引發**「負向約束失效（Negative Constraint Failure / 粉紅大象效應）」**：

1. **詞元誘導（Token Priming）：** 只要危險指令或敏感路徑名稱出現在 System Prompt 中，這些詞元就會進入 KV Cache 並處於高活躍狀態。當對話輪次拉長、模型對否定詞（`NEVER` / `NOT`）的注意力權重被稀釋時，模型反而更容易直接複製並輸出這些危險指令與路徑。
2. **大腦與手腳職責分離（本專案解決方案）：**
   - **在 `AGENTS.md`（給 LLM 看的常駐記憶）：** 採用**「100% 純正向行為錨定（Positive-Only Framing）」**。完全不提及任何危險刪除指令與敏感目錄名稱，而是規定唯一允許呼叫的安全封裝函式（`Move-ToRecycleBin` / `move_to_trash`）與結構化路徑檢查表。
   - **在 `SKILL.md`（給作業系統跑的底層腳本）：** 將具體的系統目錄黑名單（`Desktop`、`Downloads`、`Program Files` 等）封裝在 PowerShell / Bash 的 `Assert-SafePath` 攔截函式中。由作業系統執行 100% 確定性的路徑比對，一旦命中保護目錄即刻拋出例外中斷執行，徹底杜絕模型幻覺。

---

## 建議部署之安全防護措施

1. **以完整路徑映射表取代編號授權：** 進行檔案刪除或搬移前，避免僅使用數字代號確認；要求 Agent 在當前回合輸出 `編號 -> 完整絕對路徑` 對照表，確認無誤後才允許執行。
2. **單一安全回收筒 API 通道：** 規範所有刪除操作僅能透過封裝好的系統回收筒函式（Windows 的 `SendToRecycleBin` 或 macOS 的 `/usr/bin/trash`）執行，保留完整還原空間。
3. **底層腳本核心目錄攔截：** 在 `SKILL.md` 腳本範本中實作動態路徑攔截機制，自動鎖死當前電腦的所有磁碟根目錄與使用者核心目錄。
4. **強化編碼相容性與空變數斷言：** 在腳本中強制檢驗路徑變數不得為空值，並統一使用 `-LiteralPath`（Windows）與 `set -euo pipefail`（macOS），防止因中文編碼解析失敗或萬用字元展開而波及母目錄。

---

## Windows 防護配置（通用版）

以下設定採用「正向提示詞 + 底層腳本攔截」雙層架構，適用於所有 Windows 機台。

### 1. `AGENTS.md` / `MEMORY.md` 正向記憶規範 (Windows)

可直接複製以下區塊內容，貼入專案根目錄的 `AGENTS.md`、`MEMORY.md`，或設定檔中的 `system_prompt`（此區塊已採用零負向誘導設計，不出現任何危險指令字眼）：

````markdown
# FILE SYSTEM EXECUTION PROTOCOL (WINDOWS POWERSHELL)

When performing file cleanup, deletion, or directory movement on Windows, you MUST strictly follow this positive execution protocol across every turn and batch:

## 1. EXCLUSIVE SAFE REMOVAL API
- Perform all file and directory removals EXCLUSIVELY by calling the `Move-ToRecycleBin` wrapper function backed by `Microsoft.VisualBasic.FileIO.FileSystem` (`SendToRecycleBin`).
- Perform all directory movements EXCLUSIVELY by calling the `Move-ItemSafely` wrapper function.
- Clean package manager or tool caches EXCLUSIVELY using their native built-in cache subcommands (such as `uv cache clean`).
- Keep all system runtime directories and application installations intact; limit installer cleanup strictly to standalone `.exe` or `.msi` files inside temporary download folders.

## 2. MANDATORY LITERAL PATH MAPPING TABLE
- Whenever the user selects items by index numbers (e.g., "1, 3, 6"), you MUST first output an explicit `Index -> Full Literal Path` table in the current turn before generating or running any script.
- Verify that every resolved path points to a specific, intended leaf target (a specific child subfolder or individual file).
- If any item index or path mapping is ambiguous, pause immediately and ask the user to confirm the exact literal path.

## 3. DEFENSIVE POWERSHELL SCRIPT STANDARDS
- Start every file-operation `.ps1` script with `$ErrorActionPreference = 'Stop'` and load the safe wrapper functions from the `safe-disk-cleanup` skill. - Always pass `-LiteralPath` to file system cmdlets to ensure exact string matching without wildcard expansion. - Before executing any operation on a path variable, assert that the variable is non-empty (`-not [string]::IsNullOrWhiteSpace($target)`) and passes `Assert-SafePath`.

## 4. IMMEDIATE HALT ON REASONING ANOMALIES
- If during your reasoning process you detect any script bug, path discrepancy, encoding issue, or scope mismatch, halt execution immediately, rewrite the script to fix the flaw, and re-verify all target paths before running.
````

### 2. `SKILL.md` 安全清理技能與底層攔截器 (Windows)

請在 Agent 的 `skills` 目錄下建立 `safe-disk-cleanup/SKILL.md`，並貼入以下內容：

````markdown
---
name: safe-disk-cleanup
description: Universal Windows file operation skill with dynamic environment detection. Enforces Recycle Bin safety, protects all OS/User root paths on any Windows PC, and prevents wildcard/encoding collapse.
---

# Universal Windows Safe Disk Operations Skill

When executing cleanup, file deletion, or directory moves on any Windows machine, you MUST utilize the provided PowerShell template below.

## Operational Directives

1. **Dynamic Environment Binding:** Resolve paths dynamically using environment variables (`$env:USERPROFILE`, `$env:LOCALAPPDATA`, etc.).
2. **Runtime Preservation:** Only remove standalone installer files (`.exe`/`.msi`) when cleaning downloads. Preserve all installed system runtimes and package manager root directories.
3. **Encoding Safety:** Always use `-LiteralPath` and validate path resolution via `Assert-SafePath` before acting on folders containing CJK/Unicode characters.
4. **Mandatory Recycle Bin:** Always use `Move-ToRecycleBin` so all removed items can be restored from the Windows Recycle Bin.

## Universal PowerShell Execution Template

When writing `.ps1` scripts for file maintenance, ALWAYS include this defensive header:

```powershell
$ErrorActionPreference = 'Stop'
Add-Type -AssemblyName Microsoft.VisualBasic

# Dynamically construct protected path boundaries for ANY Windows system
$ProtectedPaths = [System.Collections.Generic.HashSet[string]]::new([System.StringComparer]::OrdinalIgnoreCase)

# 1. Add all system drive roots (C:\, D:\, etc.)
Get-CimInstance -ClassName Win32_LogicalDisk | ForEach-Object {
    [void]$ProtectedPaths.Add([System.IO.Path]::GetFullPath($_.DeviceID + "\").TrimEnd('\'))
}

# 2. Add system and program files roots
@(
    $env:SystemRoot,
    $env:ProgramFiles,${env:ProgramFiles(x86)},
    $env:ProgramData,$env:PUBLIC
) | Where-Object { -not [string]::IsNullOrWhiteSpace($_) } | ForEach-Object {
    [void]$ProtectedPaths.Add([System.IO.Path]::GetFullPath($_).TrimEnd('\'))
}

# 3. Add current user profile and critical shell directories
if (-not [string]::IsNullOrWhiteSpace($env:USERPROFILE)) {
    [void]$ProtectedPaths.Add([System.IO.Path]::GetFullPath($env:USERPROFILE).TrimEnd('\'))$UserSpecialDirs = @('Desktop', 'Downloads', 'Documents', 'Pictures', 'Videos', 'Music', 'AppData', 'AppData\Local', 'AppData\Roaming', 'AppData\Local\uv')
    foreach ($dir in $UserSpecialDirs) {$p = Join-Path $env:USERPROFILE$dir
        [void]$ProtectedPaths.Add([System.IO.Path]::GetFullPath($p).TrimEnd('\'))
    }
}

function Assert-SafePath {
    param([Parameter(Mandatory=$true)][string]$LiteralPath)

    if ([string]::IsNullOrWhiteSpace($LiteralPath)) {
        throw "SECURITY ABORT: Target path is null or whitespace."
    }

    if (-not (Test-Path -LiteralPath $LiteralPath)) {
        Write-Output "[SKIP] Path does not exist: $LiteralPath"
        return $null
    }

    $resolved = [System.IO.Path]::GetFullPath((Get-Item -LiteralPath$LiteralPath -Force).FullName).TrimEnd('\')

    if ($ProtectedPaths.Contains($resolved)) {
        throw "SECURITY ABORT: Operation rejected. Target is a protected system or root directory: $resolved"
    }

    return $resolved
}

function Move-ToRecycleBin {
    param(
        [Parameter(Mandatory=$true)][string]$LiteralPath,
        [string]$Tag = ""
    )
    $target = Assert-SafePath -LiteralPath$LiteralPath
    if (-not $target) { return }

    $item = Get-Item -LiteralPath$target -Force
    if ($item.PSIsContainer) {
        [Microsoft.VisualBasic.FileIO.FileSystem]::DeleteDirectory(
            $target,
            [Microsoft.VisualBasic.FileIO.UIOption]::OnlyErrorDialogs,
            [Microsoft.VisualBasic.FileIO.RecycleOption]::SendToRecycleBin
        )
    } else {
        [Microsoft.VisualBasic.FileIO.FileSystem]::DeleteFile(
            $target,
            [Microsoft.VisualBasic.FileIO.UIOption]::OnlyErrorDialogs,
            [Microsoft.VisualBasic.FileIO.RecycleOption]::SendToRecycleBin
        )
    }
    Write-Output "[RECYCLED] $Tag ->$target"
}

function Move-ItemSafely {
    param(
        [Parameter(Mandatory=$true)][string]$SourceLiteralPath,
        [Parameter(Mandatory=$true)][string]$DestinationDir
    )
    $src = Assert-SafePath -LiteralPath$SourceLiteralPath
    if (-not $src) { return }

    if (-not (Test-Path -LiteralPath $DestinationDir -PathType Container)) {
        throw "TARGET ERROR: Destination directory does not exist: $DestinationDir"
    }

    Move-Item -LiteralPath $src -Destination$DestinationDir -ErrorAction Stop
    Write-Output "[MOVED] $src ->$DestinationDir"
}
```
````

---

## macOS 防護配置（通用版）

以下為適用於所有 macOS（Apple Silicon / Intel）之正向記憶規範與底層攔截腳本，支援 macOS 原生垃圾桶（Trash）與外接卷宗保護。

### 1. `AGENTS.md` / `MEMORY.md` 正向記憶規範 (macOS)

可直接貼入任何 Mac 上的 `MEMORY.md`、工作目錄的 `AGENTS.md` 或 `system_prompt`：

````markdown
# FILE SYSTEM EXECUTION PROTOCOL (macOS ZSH / BASH)

When performing file cleanup, deletion, or directory movement on macOS, you MUST strictly follow this positive execution protocol across every turn and batch:

## 1. EXCLUSIVE SAFE REMOVAL API
- Perform all file and directory removals EXCLUSIVELY by calling the `move_to_trash` wrapper function (which routes items to the macOS Trash via `/usr/bin/trash` or Finder AppleScript).
- Perform all directory movements EXCLUSIVELY by calling the `move_item_safely` wrapper function.
- Clean package manager or developer caches EXCLUSIVELY using their native built-in subcommands (such as `uv cache clean` or `brew cleanup`).
- Keep all application bundles (`.app`) and toolchain root directories intact unless explicitly requested by exact full path.

## 2. MANDATORY LITERAL PATH MAPPING TABLE
- Whenever the user selects items by index numbers (e.g., "1, 3, 6"), you MUST first output an explicit `Index -> Full POSIX Path` table in the current turn before generating or running any script.
- Verify that every resolved path points to a specific, intended leaf target (a specific child subfolder or individual file).
- If any item index or path mapping is ambiguous, pause immediately and ask the user to confirm the exact path.

## 3. DEFENSIVE SHELL SCRIPT STANDARDS
- Start every `.sh` or `.zsh` file-operation script with `set -euo pipefail` and include the safe wrapper functions from the `safe-disk-cleanup` skill.
- Always enforce non-empty parameter expansion (`${TARGET:?Error: Variable is empty}`) and canonical path resolution via `realpath` within `assert_safe_path` to prevent empty-variable directory collapse and handle APFS Unicode NFD/NFC filenames safely.

## 4. IMMEDIATE HALT ON REASONING ANOMALIES
- If during your reasoning process you detect any script bug, path discrepancy, Unicode normalization mismatch, or scope error, halt execution immediately, rewrite the script to fix the flaw, and re-verify all target paths before running.
````

### 2. `SKILL.md` 安全清理技能與底層攔截器 (macOS)

請在 Mac 的 Agent `skills` 目錄下建立 `safe-disk-cleanup/SKILL.md`，並貼入以下內容：

````markdown
---
name: safe-disk-cleanup
description: Universal macOS file operation skill with dynamic environment detection. Enforces macOS Trash safety, protects all macOS System/User/Homebrew root paths, and prevents empty-variable collapse.
---

# Universal macOS Safe Disk Operations Skill

When executing disk cleanup, file deletion, or directory moves on any Mac, you MUST use the defensive `bash`/`zsh` template below.

## Operational Directives

1. **Dynamic Environment Binding:** Always use `$HOME` and resolve symlinks dynamically via `realpath`.
2. **Toolchain & App Bundle Preservation:** Preserve `.app` bundles and toolchain roots (`uv`, `homebrew`, `conda`, `nvm`). Use built-in cache clean commands (`uv cache clean`, `brew cleanup`).
3. **Unicode NFD/NFC Safety:** Always verify path existence with `[[ -e "$target" ]]` inside `assert_safe_path` before acting on CJK/Unicode folder names.
4. **Mandatory macOS Trash:** Always use the `move_to_trash` function below so items are safely placed in the macOS Trash.

## Universal macOS Shell Execution Template

When writing `.sh` or `.zsh` scripts for file maintenance, ALWAYS include this defensive header:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Dynamically build protected path list for ANY macOS machine
PROTECTED_PATHS=(
    "/"
    "/System"
    "/Library"
    "/Applications"
    "/Users"
    "/Volumes"
    "/opt"
    "/opt/homebrew"
    "/usr"
    "/usr/local"
    "/bin"
    "/sbin"
    "/var"
    "/private"
    "/private/var"
    "/private/tmp"
    "/private/etc"
    "/etc"
    "/tmp"
    "${HOME:?HOME is not set}"
    "$HOME/Desktop"
    "$HOME/Downloads"
    "$HOME/Documents"
    "$HOME/Pictures"
    "$HOME/Movies"
    "$HOME/Music"
    "$HOME/Public"
    "$HOME/Applications"
    "$HOME/Library"
    "$HOME/Library/Application Support"
    "$HOME/Library/Caches"
    "$HOME/Library/Preferences"
    "$HOME/.Trash"
    "$HOME/.local"
    "$HOME/.cache"
    "$HOME/.local/share/uv"
    "$HOME/Library/Caches/uv"
)

assert_safe_path() {
    local input_path="${1:-}"

    if [[ -z "${input_path// /}" ]]; then
        echo "SECURITY ABORT: Target path is empty or whitespace!" >&2
        exit 1
    fi

    if [[ ! -e "$input_path" && ! -L "$input_path" ]]; then
        echo "[SKIP] Path does not exist: $input_path" >&2
        return 1
    fi

    local resolved_path
    resolved_path="$(realpath "$input_path" 2>/dev/null || python3 -c 'import os,sys; print(os.path.realpath(sys.argv[1]))' "$input_path")"
    resolved_path="${resolved_path%/}"
    [[ -z "$resolved_path" ]] && resolved_path="/"

    for protected in "${PROTECTED_PATHS[@]}"; do
        local norm_protected="${protected%/}"
        [[ -z "$norm_protected" ]] && norm_protected="/"
        if [[ "$resolved_path" == "$norm_protected" ]]; then
            echo "SECURITY ABORT: Operation rejected. Target is a protected macOS directory: $resolved_path" >&2
            exit 1
        fi
    done

    # Also protect the top-level mount point of any external drive in /Volumes/*
    if [[ "$resolved_path" =~ ^/Volumes/[^/]+$ ]]; then
        echo "SECURITY ABORT: Cannot delete or move the root of a mounted volume: $resolved_path" >&2
        exit 1
    fi

    printf "%s" "$resolved_path"
}

move_to_trash() {
    local target_input="${1:?Missing target path}"
    local label="${2:-Item}"

    local safe_target
    if ! safe_target="$(assert_safe_path "$target_input")"; then
        return 0
    fi

    # 1. Prefer native macOS /usr/bin/trash (macOS 14 Sonoma and newer)
    if [[ -x "/usr/bin/trash" ]]; then
        /usr/bin/trash "$safe_target"
    # 2. Fallback to Finder AppleScript (supports 'Put Back' across all macOS versions)
    elif osascript -e "tell application \"Finder\" to delete POSIX file \"$safe_target\"" >/dev/null 2>&1; then
        :
    # 3. Headless fallback: move directly into ~/.Trash with timestamp to avoid overwrite
    else
        local base_name
        base_name="$(basename "$safe_target")"
        local dest="$HOME/.Trash/${base_name}_$(date +%Y%m%d_%H%M%S)"
        mv -n "$safe_target" "$dest"
    fi

    echo "[TRASHED] $label -> $safe_target"
}

move_item_safely() {
    local src_input="${1:?Missing source path}"
    local dest_dir="${2:?Missing destination directory}"

    local safe_src
    if ! safe_src="$(assert_safe_path "$src_input")"; then
        return 0
    fi

    if [[ ! -d "$dest_dir" ]]; then
        echo "TARGET ERROR: Destination directory does not exist: $dest_dir" >&2
        exit 1
    fi

    # Prevent overwriting existing files with -n
    mv -n "$safe_src" "$dest_dir/"
    echo "[MOVED] $safe_src -> $dest_dir/"
}
```
````

---

以上內容希望能幫助到正在使用 AI Agent 的每個人。
