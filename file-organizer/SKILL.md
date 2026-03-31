---
name: file-organizer
description: File organization agent that scans a directory, classifies files by type/name/age, suggests a folder structure, and moves files safely. Use when the user wants to organize their Desktop, Downloads, or any messy folder. Supports --preview (dry run), --auto (skip confirmation), and --dir /absolute/path (required in non-interactive mode). TRIGGER when the user says "organize my files", "clean up my desktop", "sort my downloads", or similar.
---

# File Organizer

Scan a directory, classify files by type, name patterns, and age, then move them into a clean folder structure — safely, without ever deleting anything.

## Arguments

| Argument | Behavior |
|----------|----------|
| `--dir /absolute/path` | Target directory. Required in non-interactive (`--auto`) mode. In interactive mode, prompts to choose if omitted. |
| `--preview` | Show the move plan and stop — nothing is moved. Takes precedence over `--auto`. |
| `--auto` | Skip the move-plan confirmation step. Requires `--dir`. |

**Important:** There is no default directory. The tool never silently switches from one folder to another. If the target cannot be confidently resolved, it stops and asks.

## Platform-Specific Path Resolution

When the user names a location like "Desktop" or "Downloads" rather than providing an absolute path, resolve it using the correct API for the platform — never rely on shell expansion (`~/Desktop`) alone.

| Platform | Desktop | Downloads |
|----------|---------|-----------|
| **Windows** | `SHGetKnownFolderPath(FOLDERID_Desktop)` via `ctypes` or registry key `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders\Desktop` — correctly handles OneDrive redirection | `SHGetKnownFolderPath(FOLDERID_Downloads)` |
| **macOS** | `Path.home() / "Desktop"` (reliable — Apple does not redirect Desktop) | `Path.home() / "Downloads"` |
| **Linux** | `xdg-user-dir DESKTOP` subprocess call — respects XDG config | `xdg-user-dir DOWNLOAD` |

If the platform-specific resolution fails or the resolved path does not exist, the result is `UNKNOWN`. `UNKNOWN` is not a fallback — it means stop and ask the user to provide an explicit `--dir`.

## Dangerous Path Blocklist

Refuse to operate on any path that matches:
- Root directories: `/`, `C:\`, `D:\`, or any single-component absolute path
- System directories: `/usr`, `/etc`, `/bin`, `/sbin`, `/lib`, `/System`, `/Windows`, `C:\Windows`, `C:\Program Files`, `C:\Program Files (x86)`
- The user's home directory itself: `~` / `$HOME` / `%USERPROFILE%` (top-level home is too broad)
- Any path with fewer than 2 path components below the filesystem root

On any blocklist match: stop immediately with a clear error. Never ask for confirmation — just refuse.

## Sync Folder Detection

Before scanning, check whether the resolved path is inside a known sync folder:

| Sync service | Detection |
|---|---|
| OneDrive (Windows) | Path contains `\OneDrive\` |
| iCloud Drive (macOS) | Path contains `/Library/Mobile Documents/` |
| Dropbox | Path contains `/Dropbox/` or `\Dropbox\` |
| Google Drive | Path contains `/Google Drive/` or `\Google Drive\` |

If a sync folder is detected, print a **warning** before proceeding:

```
⚠ Warning: This folder is inside a [OneDrive / iCloud / Dropbox / Google Drive] sync directory.
  Moving files here will trigger sync activity and may affect other devices.
  Resolved path: <absolute path>
```

Require explicit confirmation even if `--auto` is set.

## File Taxonomy

Apply classification rules **in this order** for each file (first match wins):

### 1. Name-Pattern Rules (highest priority)

| Pattern (case-insensitive) | Destination |
|----------------------------|-------------|
| Starts with `Screenshot`, `Screen Shot`, `screen`, `snap`, `Capture`, `scr_` | `Images/Screenshots/` |
| Starts with `IMG_`, `DSC_`, `DSCN_`, `DCIM`, `photo`, `pic` | `Images/Photos/` |
| Name contains `download` | `Downloads/` |

### 2. Extension Rules

| Extensions | Destination |
|------------|-------------|
| `.jpg` `.jpeg` `.png` `.gif` `.svg` `.webp` `.heic` `.bmp` `.tiff` | `Images/Graphics/` |
| `.mp4` `.mov` `.avi` `.mkv` `.m4v` `.wmv` `.flv` | `Videos/` |
| `.mp3` `.m4a` `.wav` `.flac` `.ogg` `.aac` `.wma` | `Audio/` |
| `.pdf` | `PDFs/` |
| `.doc` `.docx` `.txt` `.odt` `.pages` `.rtf` `.md` | `Documents/` |
| `.xls` `.xlsx` `.csv` `.numbers` `.ods` | `Spreadsheets/` |
| `.ppt` `.pptx` `.key` `.odp` | `Presentations/` |
| `.zip` `.tar` `.gz` `.bz2` `.xz` `.rar` `.7z` | `Archives/` |
| `.dmg` `.pkg` `.exe` `.msi` `.deb` `.rpm` `.appimage` | `Installers/` |
| `.js` `.ts` `.py` `.rb` `.go` `.rs` `.java` `.c` `.cpp` `.h` `.sh` `.bash` `.zsh` `.swift` `.kt` | `Code/` |

### 3. Age Rule (applied after category is determined)

If a file's modification time is **older than 30 days**, prepend `Archive/` to its destination.

Examples: `Archive/PDFs/`, `Archive/Images/Photos/`, `Archive/Videos/`

If a file's modification time cannot be read, log a warning and treat it as recent (do not archive it).

### 4. Fallback

Any file that matches no rule above → `Misc/`

## Safety Rules

- **Never use `rm` or delete any file** under any circumstances.
- **Never silently switch from one target directory to another.**
- Hidden files (name starts with `.`) → skip, record reason in report.
- Symlinks → skip, record reason in report.
- Subdirectories → skip, record reason in report (no recursion).
- **Duplicate destination**: if the destination path already exists on disk or two source files map to the same path, append `_1`, `_2`, etc. before the extension (e.g., `report_1.pdf`). Log every rename — never overwrite silently.
- On any individual move error: log the error, continue with remaining files. Never abort the full run.
- If file count exceeds 500, print a warning and require explicit confirmation even if `--auto` is set.

## Workflow

Make a todo list for all tasks in this workflow and complete them in order.

---

### Step 1 — Parse Arguments (Intent Capture)

Extract `--dir`, `--preview`, and `--auto` from the invocation arguments.

Record what the user said. Do **not** resolve or validate paths yet — that happens in Step 2.

If `--auto` is set and `--dir` is not provided, stop immediately:

```
Error: --auto requires an explicit --dir argument.
Non-interactive mode does not resolve named locations like "Desktop".
Usage: file-organizer --auto --dir /absolute/path/to/folder
```

---

### Step 2 — Resolve Target Path

This step is separate from parsing. Resolve the user's intent to an absolute path.

**If `--dir` was provided:** normalize it to an absolute path. Do not attempt any fallback — if it is wrong, it is wrong.

**If `--dir` was not provided (interactive mode only):** detect available locations using platform-specific APIs (see Platform-Specific Path Resolution above). Present a numbered menu:

```
Which folder would you like to organize?

  [1] Desktop    →  /Users/alice/Desktop              (23 files)
  [2] Downloads  →  /Users/alice/Downloads            (156 files)
  [3] Documents  →  /Users/alice/Documents            (342 files)
  [4] Enter a custom path

  [q] Quit
```

- Show the full resolved absolute path next to each label.
- Show the file count for each candidate so the user can sanity-check.
- If a location cannot be resolved, show it as `[not found on this system]` and make it unselectable.
- If the user selects [4], ask them to type the absolute path. Do not accept relative paths.

**If resolution fails for all candidates and no `--dir` was given:** stop with:

```
Error: Could not locate Desktop or Downloads on this system.
Please provide an explicit path: file-organizer --dir /path/to/folder
```

---

### Step 3 — Validate Target Path

Before scanning anything, validate the resolved path:

1. **Exists**: path must exist on disk. If not → stop with error.
2. **Is a directory**: must not be a file or broken symlink. If not → stop with error.
3. **Is readable**: must be able to list its contents. If not → stop with error.
4. **Is writable**: must be able to create files inside it. If not → stop with error.
5. **Not on blocklist**: check against the Dangerous Path Blocklist above. If matched → stop with error. Do not ask for confirmation — just refuse.
6. **Sync folder check**: check against the Sync Folder Detection table above. If matched → print warning (see above). Require explicit confirmation before continuing.
7. **File count**: count top-level regular files. If count > 500 → print warning and require explicit confirmation even if `--auto` is set.

---

### Step 4 — Confirm Target (interactive mode only, skip if --auto with valid --dir)

Show the confirmed target to the user before any scanning:

```
Target confirmed: /Users/alice/Desktop
  Platform : macOS
  Files    : 23 regular files (4 hidden, 2 symlinks, 3 subdirectories — all skipped)
  Mode     : interactive

Proceed with this folder? [yes / no / choose again]
```

- **yes** → continue to Step 5
- **no** → stop. Print: "Aborted. Nothing was changed."
- **choose again** → return to Step 2

In `--auto` mode with a valid `--dir`, print the target info to stdout for logging but do not prompt:

```
Target: /Users/alice/Desktop  (23 files)
Mode: auto — confirmation skipped
```

---

### Step 5 — Scan the Directory

List all top-level entries in the target directory. For each entry record: name, type (file / directory / symlink), size, and modification timestamp.

Immediately partition entries:
- **To classify**: regular files that are not hidden
- **Skipped**: hidden files (name starts with `.`), symlinks, subdirectories — record each with its skip reason

---

### Step 6 — Classify Each File

For every file in the classify list, apply the taxonomy rules in order (name-pattern → extension → age → fallback). Build a classification map:

```
{ source_filename → relative_destination_path }
```

Example entries:
```
Screenshot 2026-01-15.png  →  Images/Screenshots/Screenshot 2026-01-15.png
quarterly_report.pdf       →  Archive/PDFs/quarterly_report.pdf   [>30 days]
setup.dmg                  →  Installers/setup.dmg
mystery.xyz                →  Misc/mystery.xyz
```

---

### Step 7 — Resolve Duplicate Destinations

For each destination path in the classification map:
1. Check whether the file already exists at that path on disk.
2. Check whether two source files map to the same destination.
3. Where a conflict exists, append `_1`, `_2`, etc. before the file extension. Update the classification map with the resolved paths. Log every rename.

---

### Step 8 — Display the Move Plan

Output a formatted table:

```
FILE ORGANIZATION PLAN
Target : /Users/alice/Desktop  (N files to move, N skipped)

 SOURCE                              DESTINATION
 ──────────────────────────────────  ────────────────────────────────────────────────────
 Screenshot 2026-01-15.png           Images/Screenshots/Screenshot 2026-01-15.png
 quarterly_report.pdf                Archive/PDFs/quarterly_report.pdf          [>30d]
 setup.dmg                           Installers/setup.dmg
 report.pdf → report_1.pdf           PDFs/report_1.pdf                          [renamed]
 .DS_Store                           [hidden — skipped]
 mystery.xyz                         Misc/mystery.xyz

Folders to create : Images/Screenshots, Archive/PDFs, Installers, PDFs, Misc
Files to move     : N
Skipped           : N
Renamed           : N  (collision)
```

If `--preview` was passed, **stop here**. Print:
```
Preview complete. No files were moved.
Run without --preview to execute.
```

---

### Step 9 — Confirm Move Plan (skip if --auto)

Ask the user:

> "Proceed with this plan? **yes** / **no** / **edit**"

- **yes** → continue to Step 10
- **no** → abort. Print: "Aborted. No files were moved." Stop.
- **edit** → let the user describe which files to redirect to different folders conversationally. Update the classification map, re-display the updated plan (Step 8), and ask again.

---

### Step 10 — Create Directories and Move Files

For each unique destination folder:
```bash
mkdir -p "<absolute-target-dir>/<destination-folder>"
```

Then for each file in the classification map:
```bash
mv -- "<absolute-target-dir>/<source-filename>" "<absolute-target-dir>/<destination-path>"
```

Move files one at a time. On any individual failure (permissions error, disk full, file in use, etc.), log the error and continue with remaining files. **Never use `rm`. Never overwrite.**

Always use the full absolute path in `mv` commands. Never use `~` or relative paths in shell commands.

---

### Step 11 — Generate Summary Report

Print the final report:

```
FILE ORGANIZATION COMPLETE
──────────────────────────────────────────────
Target directory : /Users/alice/Desktop
Completed at     : <ISO 8601 timestamp>

RESULTS
  Moved           :  N files
  Renamed         :  N files  (destination collision)
  Skipped         :  N files  (N hidden, N symlinks, N subdirectories)
  Errors          :  N files
  Folders created :  N

MOVED FILES BY CATEGORY
  Images/Screenshots : N
  Images/Photos      : N
  PDFs               : N
  Archive/PDFs       : N
  Misc               : N
  ...

SKIPPED FILES
  .DS_Store        →  hidden file — skipped
  my-project/      →  subdirectory — skipped
  ...

RENAMES
  report.pdf  →  report_1.pdf  (PDFs/ already contained report.pdf)
  ...

ERRORS  (if any)
  locked-file.pdf  →  Permission denied — file may be in use

NOTES
  - Files in Archive/ were last modified more than 30 days ago.
  - Files in Misc/ had unrecognized extensions: <list>
  - No files were deleted. Everything is either in its new location
    or untouched in the original directory.
```

---

## Wrap Up

In your final message to the user, provide:

1. A one-sentence summary (e.g., "Moved 42 files into 8 folders on your Desktop at `/Users/alice/Desktop`.")
2. The full RESULTS block from the summary report.
3. If any files landed in `Misc/`, list them explicitly and suggest the user inspect them manually.
4. If any errors occurred, explain each one and suggest remediation (check permissions, free disk space, close the file in another app, etc.).
5. Remind the user: **no files were deleted** — everything is either in its new location or untouched in the original directory.
