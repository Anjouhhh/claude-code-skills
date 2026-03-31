---
name: file-organizer
description: >
  High-side-effect skill that scans a directory and moves files into a categorized
  folder structure. Invoke ONLY when the user has explicitly asked to organize files
  or has directly invoked this skill by name. Do NOT trigger from vague mentions of
  messiness, clutter, or a busy folder — ask for explicit confirmation of intent first.
  Requires explicit target confirmation before any files are moved. Supports --preview
  (dry run), --auto (requires --dir), and --dir /absolute/path. Non-interactive mode
  never guesses or infers a target directory.
---

# File Organizer

Scan a directory, classify files by type, name patterns, and age, then move them into a clean folder structure — safely, without ever deleting anything.

## Critical Safety Rules

These rules are mandatory and non-negotiable. They override any interpretation of convenience, helpfulness, or efficiency.

1. **Never substitute one target directory for another.** If the user said "Desktop" and Desktop cannot be resolved, the result is an error — not "use Downloads instead."
2. **`UNKNOWN` is a hard stop, not a candidate.** If platform-specific resolution returns `UNKNOWN`, stop immediately and ask the user for an explicit `--dir`. Do not map `UNKNOWN` to any other folder.
3. **Never proceed past path resolution without showing the resolved absolute path.** Scanning, classification, and moving must not begin until the full absolute resolved path has been displayed to the user.
4. **User intent and resolved path must be confirmed together before any action.** The user must confirm the resolved absolute path as their intended target — not just the move plan.
5. **Ambiguity is always an error.** If the target cannot be determined with certainty, stop. Never guess.
6. **Never operate on root, system, or home directories.** See the Dangerous Path Blocklist below.
7. **This tool moves files. Treat it as a high-side-effect operation.** When in doubt between stopping and guessing, always stop.
8. **Fallback is not a feature.** There is no fallback directory. There is no default directory. The absence of a resolvable target is an error condition, always.

## Invocation Requirements

This skill must only run when the user has **explicitly** asked to organize files or has directly invoked the `file-organizer` skill by name.

- Indirect statements such as "my desktop is messy", "I have too many downloads", or "things are getting cluttered" are **not** sufficient to begin execution. Respond by asking: "Would you like me to run the file organizer on a specific folder?" — do not proceed until the answer is clearly yes.
- Ambiguous intent → ask for clarification before doing anything.
- If the environment supports skill invocation metadata, treat this skill as `manual-only`.

## Execution Flow

The following is the required order of operations. Steps 1–9 must complete before step 10 begins. No step may be skipped or reordered.

```
1.  Capture the raw user input — do not resolve or validate paths yet.
2.  Classify the input type:
      a. Explicit absolute path provided via --dir /absolute/path
      b. Named location intent ("Desktop", "Downloads", "Documents")
      c. Nothing provided
3.  If --auto is set and --dir is absent → STOP immediately (hard error, see Step 1).
4.  If no explicit --dir and mode is non-interactive → STOP immediately.
5.  If interactive and no --dir:
      → Resolve each named location using platform-aware APIs.
      → Present a numbered menu with the full resolved absolute path and file count for each.
      → Mark unresolvable locations as [not available on this system] — never make them selectable.
      → Let the user select a candidate or enter a custom absolute path.
6.  Normalize the selected/provided path to its absolute form.
7.  Validate: exists / is a directory / readable / writable / not on blocklist.
8.  Check for sync folder or network share. Warn if detected; require explicit confirmation.
9.  Show the resolved absolute path. Ask: "You are about to organize files in: <absolute path>. Is this the correct target? [yes / no / choose again]"
10. Scan the directory (top-level files only).
11. Classify files using the taxonomy rules.
12. Resolve destination collisions.
13. Display the move plan.
14. If --preview → stop. Print: "Preview complete. No files were moved."
15. If not --auto → ask: "Proceed with this plan? [yes / no / edit]"
16. Execute moves (mkdir -p then mv, absolute paths only, no shell expansion).
17. Print summary report.
```

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

## Target Selection States

Path selection passes through four distinct states. These are never combined into a single step.

| State | Definition |
|-------|------------|
| **Intent** | What the user stated: "Desktop", "my downloads folder", or a literal path string |
| **Resolved path** | The absolute path produced by platform-aware resolution (named locations) or normalization (explicit paths) |
| **Validated** | The resolved path has passed all checks: exists, is a directory, readable, writable, not on the blocklist |
| **Approved** | The user has explicitly confirmed that the resolved absolute path is the correct target |

Rules:
- A `UNKNOWN` resolved path is a terminal state — never map it to another folder.
- Explicit path input and named location input are **different trust classes**: named locations require platform-aware API resolution; explicit paths are normalized as provided.
- A path that exists on disk is not automatically validated or approved — all checks must still run.
- **Approval at Step 9 (target path confirmation) is separate from approval at Step 15 (move plan confirmation).** Both are required. Neither substitutes for the other.

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
| Network share (Windows) | Path starts with `\\` (UNC path) |
| Network share (Unix/macOS) | Path is under `/net/`, `/mnt/`, `/media/`, or the mount entry in `/proc/mounts` lists type `nfs`, `cifs`, or `smbfs` |

If a sync folder is detected, print a **warning** before proceeding:

```
⚠ Warning: This folder is inside a [OneDrive / iCloud / Dropbox / Google Drive] sync directory.
  Moving files here will trigger sync activity and may affect other devices.
  Resolved path: <absolute path>
```

If a network share is detected, print:

```
⚠ Warning: This folder appears to be on a network share.
  File operations may be slow, unreliable, or affect other users.
  Resolved path: <absolute path>
```

Require explicit confirmation even if `--auto` is set — for both sync folders and network shares.

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

Non-interactive mode does not resolve named locations under any circumstances. Even if the platform could successfully resolve "Desktop", `--auto` mode requires an explicit `--dir`. Do not attempt resolution as a convenience fallback.

---

### Step 2 — Resolve Target Path

This step is separate from parsing. Resolve the user's intent to an absolute path.

**If `--dir` was provided:** normalize it to an absolute path. Do not attempt any fallback — if it is wrong, it is wrong.

**If `--dir` was not provided (interactive mode only):** detect available locations using platform-specific APIs (see Platform-Specific Path Resolution above). Present a numbered menu:

```
Which folder would you like to organize?

  [1] Desktop    →  /Users/alice/Desktop                           (23 files)
  [2] Downloads  →  /Users/alice/Library/CloudStorage/OneDrive/Downloads  (12 files) [⚠ OneDrive]
  [3] Documents  →  /Users/alice/Documents                         (342 files)
  [4] Enter a custom path
  [x] Desktop (Windows)  →  [not available on this system]

  [q] Quit
```

- Show the full resolved absolute path next to each label.
- Show the file count for each candidate so the user can sanity-check.
- If a sync folder or network share is detected for a candidate, show a warning tag inline (e.g., `[⚠ OneDrive]`, `[⚠ network share]`).
- If a location cannot be resolved, show it as `[not available on this system]` and make it **unselectable**. Entering an unresolvable option's number must produce an explicit error — not a silent skip, not a fallback.
- If the user selects the custom path option, ask them to type an absolute path. Relative paths must be rejected with an error.

**Non-interactive mode note:** If `--auto` is set, this entire branch is unreachable — `--auto` without `--dir` is caught and rejected in Step 1. This step runs only in interactive mode.

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
6. **Sync folder and network share check**: check against the Sync Folder Detection table above. If matched → print warning (see above). Require explicit confirmation before continuing.
7. **File count**: count top-level regular files. If count > 500 → print warning and require explicit confirmation even if `--auto` is set.

---

### Step 4 — Confirm Target (interactive mode only, skip if --auto with valid --dir)

Show the confirmed target to the user before any scanning. The user is confirming the **resolved absolute path** as their intended target — not the move plan. Make this explicit:

```
You are about to organize files in: /Users/alice/Desktop
  Platform : macOS
  Files    : 23 regular files (4 hidden, 2 symlinks, 3 subdirectories — all skipped)
  Mode     : interactive

Is this the correct target? [yes / no / choose again]
```

- **yes** → continue to Step 5
- **no** → stop. Print: "Aborted. Nothing was changed."
- **choose again** → return to Step 2

In `--auto` mode with a valid `--dir`, print the target info to stdout before scanning begins. This is mandatory, not optional — it is the only record of the resolved target for log review:

```
Target: /Users/alice/Desktop  (23 files)
Mode: auto — target confirmation skipped
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

## Never Do These Things

- **Never** substitute one named location for another. Desktop → Downloads is never acceptable. Neither is any other substitution.
- **Never** proceed when path resolution returns `UNKNOWN` or is in any way ambiguous.
- **Never** scan a directory that the user has not explicitly confirmed as the target.
- **Never** operate on `/`, `~`, `C:\`, `D:\`, or any equivalent root or home directory (see Dangerous Path Blocklist).
- **Never** treat the existence of a path on disk as proof it matches the user's intent.
- **Never** skip the target path confirmation step (Workflow Step 4 / Execution Flow step 9) for a move operation.
- **Never** use `~`, relative paths, or any shell expansion in `mv` or `mkdir` commands.
- **Never** infer that `--auto` or any non-interactive mode grants permission to guess or resolve the target directory.
- **Never** continue execution after a blocklist match — refuse and stop, no confirmation prompt.
- **Never** delete files under any circumstances.

## Wrap Up

In your final message to the user, provide:

1. A one-sentence summary (e.g., "Moved 42 files into 8 folders on your Desktop at `/Users/alice/Desktop`.")
2. The full RESULTS block from the summary report.
3. If any files landed in `Misc/`, list them explicitly and suggest the user inspect them manually.
4. If any errors occurred, explain each one and suggest remediation (check permissions, free disk space, close the file in another app, etc.).
5. Remind the user: **no files were deleted** — everything is either in its new location or untouched in the original directory.
