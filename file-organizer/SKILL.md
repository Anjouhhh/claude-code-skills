---
name: file-organizer
description: File organization agent that scans a directory, classifies files by type/name/age, suggests a folder structure, and moves files safely. Use when the user wants to organize their Desktop, Downloads, or any messy folder. Supports --preview (dry run), --auto (skip confirmation), and --dir /path/to/dir (custom target). TRIGGER when the user says "organize my files", "clean up my desktop", "sort my downloads", or similar.
---

# File Organizer

Scan a directory, classify files by type, name patterns, and age, then move them into a clean folder structure — safely, without ever deleting anything.

## Arguments

| Argument | Default | Behavior |
|----------|---------|----------|
| `--dir /path/to/dir` | `~/Desktop` | Directory to organize |
| `--preview` | off | Show the move plan and stop — nothing is moved |
| `--auto` | off | Skip the confirmation step and move immediately |

`--preview` takes precedence over `--auto`. If the resolved directory does not exist or is not readable/writable, report the error and stop immediately.

## File Taxonomy

Apply classification rules **in this order** for each file:

### 1. Name-Pattern Rules (highest priority)

| Pattern | Destination |
|---------|-------------|
| Starts with `Screenshot`, `screen`, `snap`, `Capture` (case-insensitive) | `Images/Screenshots/` |
| Starts with `IMG_`, `DSC_`, `DSCN_`, `photo`, `pic` (case-insensitive) | `Images/Photos/` |
| Name contains `download` (case-insensitive) | `Downloads/` |

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
| `.zip` `.tar` `.gz` `.bz2` `.xz` `.rar` `.7z` `.tar.gz` `.tar.bz2` | `Archives/` |
| `.dmg` `.pkg` `.exe` `.msi` `.deb` `.rpm` `.appimage` | `Installers/` |
| `.js` `.ts` `.py` `.rb` `.go` `.rs` `.java` `.c` `.cpp` `.h` `.sh` `.bash` `.zsh` `.swift` `.kt` | `Code/` |

### 3. Age Rule (applied after category is determined)

If a file's modification time is **older than 30 days**, prepend `Archive/` to its destination.

Examples: `Archive/PDFs/`, `Archive/Images/Photos/`, `Archive/Videos/`

### 4. Fallback

Any file that matches no rule above → `Misc/`

## Safety Rules

- **Never use `rm` or delete any file** under any circumstances.
- Hidden files (name starts with `.`) → skip, record reason in report.
- Symlinks → skip, record reason in report.
- Subdirectories → skip, record reason in report (no recursion by default).
- **Duplicate destination**: if the destination path already exists on disk or two source files map to the same path, append `_1`, `_2`, etc. before the extension (e.g., `report_1.pdf`).
- On any individual move error: log the error, continue with remaining files. Never abort the full run.

## Workflow

Make a todo list for all tasks in this workflow and complete them in order.

### 1. Parse Arguments

Extract `--dir`, `--preview`, and `--auto` from the invocation arguments.

- Resolve `--dir` to an absolute path. Default to `~/Desktop` if not provided; if `~/Desktop` does not exist, fall back to `~/Downloads`.
- Verify the resolved path exists and is readable and writable. If not, report the error and stop.
- If both `--preview` and `--auto` are passed, note that `--preview` takes precedence.

### 2. Scan the Directory

```bash
ls -la <target-dir>
```

Enumerate all top-level entries. For each entry record: name, type (file / directory / symlink), size, and modification timestamp.

Immediately partition entries into two lists:
- **To classify**: regular files that are not hidden
- **Skipped**: hidden files, symlinks, subdirectories — record each with its skip reason

### 3. Classify Each File

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

### 4. Resolve Duplicate Destinations

For each destination path in the classification map:
1. Check whether the file already exists at that path on disk.
2. Check whether two source files map to the same destination.
3. Where a conflict exists, append `_1`, `_2`, etc. before the file extension. Update the classification map with the resolved paths.

### 5. Display the Move Plan

Output a formatted table:

```
FILE ORGANIZATION PLAN — <target-dir>  (N files to move, N skipped)

 SOURCE                              DESTINATION
 ──────────────────────────────────  ──────────────────────────────────────────────────
 Screenshot 2026-01-15.png           Images/Screenshots/Screenshot 2026-01-15.png
 quarterly_report.pdf                Archive/PDFs/quarterly_report.pdf          [>30d]
 setup.dmg                           Installers/setup.dmg
 .DS_Store                           [hidden — skipped]
 mystery.xyz                         Misc/mystery.xyz

Folders to create: Images/Screenshots, Archive/PDFs, Installers, Misc
Files to move: N   |   Skipped: N
```

If `--preview` was passed, **stop here**. Print:
> "Preview complete. No files were moved. Run without --preview to execute."

### 6. Confirm (skip this step if --auto)

Ask the user:

> "Proceed with this plan? **yes** / **no** / **edit**"

- **yes** → continue to step 7
- **no** → abort. Print: "Aborted. No files were moved." Stop.
- **edit** → let the user describe which files to redirect to different folders. Update the classification map accordingly, re-display the updated plan (step 5), and ask again.

### 7. Create Directories and Move Files

For each unique destination folder, run:
```bash
mkdir -p "<target-dir>/<destination-folder>"
```

Then for each file in the classification map, run:
```bash
mv "<target-dir>/<source-filename>" "<target-dir>/<destination-path>"
```

Move files one at a time. On any individual failure (permissions error, disk full, etc.), log the error and continue. **Never use `rm`.**

### 8. Generate Summary Report

Print the final report:

```
FILE ORGANIZATION COMPLETE
──────────────────────────────────
Target directory : <target-dir>
Completed at     : <ISO timestamp>

RESULTS
  Moved           :  N files
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

ERRORS  (if any)
  locked-file.pdf  →  Permission denied
  ...

NOTES
  - Files placed in Archive/ were last modified more than 30 days ago.
  - Files placed in Misc/ had unrecognized extensions: <list extensions>
```

## Wrap Up

In your final message to the user, provide:

1. A one-sentence summary (e.g., "Moved 42 files into 8 folders on your Desktop.")
2. The full RESULTS block from the summary report.
3. If any files landed in `Misc/`, list them explicitly and suggest the user inspect them manually.
4. If any errors occurred, explain each one and suggest remediation (check permissions, free disk space, etc.).
5. Remind the user: **no files were deleted** — everything is either in its new location or untouched in the original directory.
