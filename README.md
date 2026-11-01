# Anti-Local-LLM-Deleted-Your-Computer

> **使用每個本地 AI 加上任何 Harness，都請在你的 Harness 加上文中這些防護規則。你可能會感謝我，也希望你永遠不會遇到這種事情。**

本專案提供可直接部署於各類 AI Agent 框架（如 Hermes、OpenClaw、Claude Code 等）的系統提示詞（`AGENTS.md` / `MEMORY.md`）與安全檔案操作技能（`SKILL.md`），防止本地端中小型語言模型在執行磁碟清理或檔案管理時發生毀滅性的誤刪災難。

---

## 目錄

- [技術風險提示：本地端 AI Agent 誤刪案例分析](#技術風險提示本地端-ai-agent-誤刪案例分析)
- [建議部署之安全防護措施](#建議部署之安全防護措施)
- [Windows 防護配置（通用版）](#windows-防護配置通用版)
  - [1. AGENTS.md / MEMORY.md 記憶規範 (Windows)](#1-agentsmd--memorymd-記憶規範-windows)
  - [2. SKILL.md 安全清理技能 (Windows)](#2-skillmd-安全清理技能-windows)
- [macOS 防護配置（通用版）](#macos-防護配置通用版)
  - [1. AGENTS.md / MEMORY.md 記憶規範 (macOS)](#1-agentsmd--memorymd-記憶規範-macos)
  - [2. SKILL.md 安全清理技能 (macOS)](#2-skillmd-安全清理技能-macos)

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

## 建議部署之安全防護措施

為避免類似情況發生，建議在 AI Agent 的系統提示詞（`AGENTS.md` / `MEMORY.md`）與技能範本（`SKILL.md`）中導入以下四項防護規範：

1. **以完整路徑取代編號授權：** 進行檔案刪除或搬移前，避免僅使用數字代號確認；應要求 Agent 在當前回合重新列出目標的完整絕對路徑，確認無誤後才允許執行。
2. **禁用永久刪除指令，強制改用系統回收筒 API：** 明文禁止模型使用 `Remove-Item -Force`、`Clear-RecycleBin`（Windows）及 `rm -rf`（macOS/Linux）。所有刪除操作應統一透過系統 API 移至資源回收筒（如 Windows 的 `SendToRecycleBin` 或 macOS 的 `/usr/bin/trash`），保留還原空間。
3. **建立核心目錄保護清單：** 在腳本範本中實作路徑檢查機制，將使用者根目錄、`Desktop`、`Downloads`、`Documents`、`AppData`、`Program Files` 等關鍵系統與使用者目錄設為保護名單，一旦解析出的目標路徑為上述根目錄本身，即刻觸發例外並中斷腳本。
4. **強化編碼相容性與空變數驗證：** 在腳本中強制檢驗路徑變數不得為空值，並統一使用 `-LiteralPath` 取代 `-Path`，防止因中文編碼解析失敗或萬用字元展開而波及母目錄。

---

## Windows 防護配置（通用版）

以下為適用於所有 Windows 機台之動態防護設定，會自動解析當前使用者路徑與所有掛載磁碟。

### 1. `AGENTS.md` / `MEMORY.md` 記憶規範 (Windows)

可直接複製以下區塊內容，貼入專案根目錄的 `AGENTS.md`、`MEMORY.md`，或設定檔中的 `system_prompt`：

````markdown
# CRITICAL FILE SYSTEM SAFETY RULES (CROSS-PLATFORM WINDOWS POWERSHELL)

You are running on a Windows workstation. Any file deletion mistake causes irreversible data loss. You MUST obey these rules without exception across any machine or user profile:

## 1. BANNED COMMANDS (ZERO TOLERANCE)
- **NEVER** use `Remove-Item -Force` or `Remove-Item -Recurse -Force` on directories.
- **NEVER** use `Clear-RecycleBin`, `del /f /s /q`, `rd /s /q`, or raw command-line directory deletion.
- **ALL** file and directory deletions MUST be sent to the Windows Recycle Bin using `Microsoft.VisualBasic.FileIO.FileSystem` (`SendToRecycleBin`). Permanent deletion is strictly forbidden.

## 2. UNIVERSAL DYNAMIC PATH BLACKLIST (NEVER DELETE OR MOVE ROOT PATHS)
Regardless of the current username or drive letter, you MUST NEVER target any of the following paths for deletion or root movement:
- Current user profile: `$env:USERPROFILE`
- Standard user shell folders: Desktop, Downloads, Documents, Pictures, Videos, Music under `$env:USERPROFILE` - Application data roots: `$env:APPDATA` (`AppData\Roaming`), `$env:LOCALAPPDATA` (`AppData\Local`) - Package manager root folders: `$env:LOCALAPPDATA\uv` (Use `uv cache clean` ONLY; NEVER delete the `uv` parent directory)
- Operating system & Program roots: Any drive root (e.g., `C:\`, `D:\`), `$env:SystemRoot` (`C:\Windows`), `$env:ProgramFiles`, `$env:ProgramFiles(x86)`, `$env:ProgramData` - Active workspace roots: The root directory where your agent or project is hosted.  ## 3. ZERO NUMBER-ONLY DISPATCH & CROSS-TURN AMNESIA SHIELD - When the user approves cleanup tasks using numbers (e.g., "Delete 1, 3, 5; move 2"), **NEVER** guess or retrieve paths from long conversational memory. - Before executing ANY deletion or move, you MUST re-list the exact absolute path (`C:\...`) for each item in the current turn. If you are even 1\% uncertain about which path an index corresponds to, **STOP and ask for explicit confirmation**.  ## 4. POWERSHELL ENCODING & NULL-VARIABLE COLLAPSE DEFENSE - Windows PowerShell 5.1 (`powershell.exe`) defaults to non-UTF8 (ANSI/Big5) for `.ps1` scripts without BOM. Non-ASCII/CJK characters (such as Chinese directory names) will corrupt into empty strings or garbage, causing `"$dir\$subfolder"` to collapse to `"$dir\"` and wipe the entire parent directory (e.g., Desktop/Downloads)! - **RULE:** Always use `-LiteralPath` instead of `-Path` (to prevent accidental wildcard expansion). - **RULE:** Always validate `[string]::IsNullOrWhiteSpace($targetPath)` and ensure the resolved path does NOT equal any parent directory before issuing file system commands.

## 5. THINKING-STAGE BUG ABORT PROTOCOL
- If during your internal reasoning/thinking process you spot ANY flaw, scope over-extension, or accidental system directory deletion (e.g., "Wait, this deletes the parent folder too"), you MUST **ABORT EXECUTION IMMEDIATELY** and fix the script. NEVER proceed with "it should be fine" or "safe to execute anyway".
````

### 2. `SKILL.md` 安全清理技能 (Windows)

請在 Agent 的 `skills` 目錄下建立 `safe-disk-cleanup/SKILL.md`，並貼入以下內容：

````markdown
---
name: safe-disk-cleanup
description: Universal Windows file operation skill with dynamic environment detection. Enforces Recycle Bin safety, protects all OS/User root paths on any Windows PC, and prevents wildcard/encoding collapse.
---

# Universal Windows Safe Disk Operations Skill

When executing cleanup, file deletion, or directory moves on any Windows machine, you MUST adhere to the dynamic path resolution rules and utilize the provided PowerShell template below.

## Operational Directives

1. **Dynamic Environment Binding:** Never hardcode specific usernames (e.g., `C:\Users\John`) or static drives. Always resolve paths dynamically using environment variables (`$env:USERPROFILE`, `$env:LOCALAPPDATA`, etc.).
2. **Runtime Preservation:** Distinguish standalone installer packages (`.exe`/`.msi` inside Downloads or Temp) from installed system runtimes inside `Program Files` or package manager cores. Never delete installed runtimes.
3. **Encoding Safety:** Use literal paths and guard against string truncation when handling folders containing CJK/Unicode characters.
4. **Mandatory Recycle Bin:** Always use Windows Shell API to send files to the Recycle Bin so accidental deletions can be restored.

## Universal PowerShell Execution Template

When writing `.ps1` scripts for file maintenance, ALWAYS wrap your logic with this defensive header:

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

以下為適用於所有 macOS（Apple Silicon / Intel）之動態防護設定，支援 macOS 原生垃圾桶（Trash）與外接卷宗根目錄保護。

### 1. `AGENTS.md` / `MEMORY.md` 記憶規範 (macOS)

可直接貼入任何 Mac 上的 `MEMORY.md`、工作目錄的 `AGENTS.md` 或 `system_prompt`：

````markdown
# CRITICAL FILE SYSTEM SAFETY RULES (UNIVERSAL macOS / ZSH / BASH)

You are running on a macOS workstation. Any file deletion mistake on APFS/SSD causes immediate TRIM and irreversible data loss. You MUST obey these rules without exception across any Mac or user account:

## 1. BANNED COMMANDS (ZERO TOLERANCE)
- **NEVER** use `rm -rf`, `rm -fr`, `rm -r`, `unlink`, or `find ... -delete`.
- **NEVER** empty the macOS Trash (`rm -rf ~/.Trash/*` or via AppleScript `empty trash`).
- **ALL** file and directory deletions MUST be sent to the macOS Trash using the native `/usr/bin/trash` command or Finder AppleScript (`osascript`), NEVER permanently deleted.

## 2. UNIVERSAL DYNAMIC PATH BLACKLIST (NEVER DELETE OR MOVE ROOT PATHS)
Regardless of the current macOS username or mounted volume, you MUST NEVER target any of the following paths for deletion or root movement:
- Current user home: `$HOME` (`/Users/$USER`)
- Standard user folders: `$HOME/Desktop`, `$HOME/Downloads`, `$HOME/Documents`, `$HOME/Pictures`, `$HOME/Movies`, `$HOME/Music`, `$HOME/Public`
- macOS Library & App Data roots: `$HOME/Library`, `$HOME/Library/Application Support`, `$HOME/Library/Preferences`, `$HOME/Library/Caches` (NEVER delete `Caches` as a whole)
- Developer & Toolchain roots: `/opt/homebrew`, `/usr/local`, `$HOME/.local`, `$HOME/.cache`, `$HOME/.cargo`, `$HOME/.pyenv`, `$HOME/.nvm` (For `uv`, run `uv cache clean` ONLY; NEVER delete the `uv` directory or environment root)
- System & Volume roots: `/`, `/System`, `/Applications`, `/Library`, `/Users`, `/Volumes`, `/private`, `/var`, `/etc`, `/tmp`
- Active workspace root: The root directory where the agent or current project is located.

## 3. ZERO NUMBER-ONLY DISPATCH & CROSS-TURN AMNESIA SHIELD
- When the user approves cleanup tasks using numbers (e.g., "Delete 1, 3, 5; move 2"), **NEVER** guess or retrieve paths from conversational memory.
- Before executing ANY deletion or move, you MUST re-list the exact absolute POSIX path (`/Users/...`) for each item in the current turn. If you are even 1% uncertain which path an index corresponds to, **STOP and ask the user**.

## 4. SHELL UNSET-VARIABLE COLLAPSE & NFD/NFC UNICODE DEFENSE
- In `zsh`/`bash`, if a subfolder variable `$SUBDIR` is empty, unset, or fails to match due to macOS APFS Unicode NFD/NFC normalization (common with Chinese/Japanese/accented folder names), `"$HOME/Desktop/$SUBDIR"` collapses into `"$HOME/Desktop/"` and wipes the entire Desktop!
- **RULE:** Always put `set -euo pipefail` at the top of every shell script.
- **RULE:** Always use `${VAR:?Error: Variable is empty}` parameter expansion so an empty variable immediately aborts the script.
- **RULE:** Always resolve canonical paths with `realpath` and verify the target is NOT equal to `$HOME/Desktop` or any parent directory before acting.

## 5. THINKING-STAGE BUG ABORT PROTOCOL
- If during your internal reasoning/thinking process you spot ANY bug, unintended directory scope, or path mismatch in a script you wrote (e.g., "Wait a sec — there's a bug in the script"), you MUST **ABORT EXECUTION IMMEDIATELY** and fix the script. NEVER rationalize a bug with "OK, safe. Execute".
````

### 2. `SKILL.md` 安全清理技能 (macOS)

請在 Mac 的 Agent `skills` 目錄下建立 `safe-disk-cleanup/SKILL.md`，並貼入以下內容：

````markdown
---
name: safe-disk-cleanup
description: Universal macOS file operation skill with dynamic environment detection. Enforces macOS Trash safety, blocks rm -rf, protects all macOS System/User/Homebrew root paths, and prevents empty-variable collapse.
---

# Universal macOS Safe Disk Operations Skill

When executing disk cleanup, file deletion, or directory moves on any Mac, you MUST follow these rules and use the defensive `bash`/`zsh` template below.

## Operational Directives

1. **Dynamic Environment Binding:** Never hardcode specific usernames (e.g., `/Users/john`). Always use `$HOME` and resolve symlinks dynamically via `realpath`.
2. **Toolchain & App Bundle Preservation:** Never delete `.app` bundles inside `/Applications` or `~/Applications` unless explicitly requested by exact app name. Never delete `/opt/homebrew` or toolchain roots (`uv`, `conda`, `nvm`). For package managers, use their built-in cache clean commands (`uv cache clean`, `brew cleanup`).
3. **Unicode NFD/NFC Safety:** macOS APFS stores filenames in Unicode NFD (or normalization-insensitive) form. Never rely on unverified string concatenation for CJK/Chinese folder names; always verify path existence with `[[ -e "$target" ]]` before acting.
4. **Mandatory macOS Trash:** Never use `rm -rf`. Always use the `move_to_trash` function below so items go to the macOS Trash.

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
