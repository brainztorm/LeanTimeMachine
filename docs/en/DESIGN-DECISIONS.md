# Design decisions

🇫🇷 [Version française](../fr/DESIGN-DECISIONS.md)

Technical notes on the choices made in LeanTimeMachine, so they do not have to be debated again.

## "Sticky" or "path-based" exclusion?

`tmutil addexclusion <path>` (without `-p`) sets a **sticky** exclusion: an extended attribute `com.apple.metadata:com_apple_backup_excludeItem` on the item itself. It survives the item being moved or copied, even to another volume. It needs no `sudo` and does not show up in System Settings.

`tmutil addexclusion -p <path>` sets a **path-based** exclusion: the path is stored in the `SkipPaths` key of `/Library/Preferences/com.apple.TimeMachine.plist`. This is the list shown in System Settings → General → Time Machine → Options → "Exclude from Backups".

| Situation | Sticky (xattr) | Path-based (`-p`) |
| :--- | :--- | :--- |
| Folder moved or renamed | the exclusion follows | lost (no effect at the new path) |
| Folder deleted then recreated | **lost** (the attribute dies with the inode) | **kept**, active again as soon as the path reappears |
| Non-existent path | impossible | possible (see below) |
| Visible in System Settings | no | yes |
| Privileges | none | `sudo` |

**Choice made:**

* **Fixed paths** (`config/rules.conf`) → `-p`. Many of these folders (`~/.colima`, caches, container storage) are deleted and recreated by daemons: only a path-based exclusion survives. It also lets you exclude software that is uninstalled or not installed yet.
* **Project folders** (`node_modules`, `target`, `build`...) → sticky. Their path is unpredictable and the exclusion must follow the folder if it is moved.

## Reading `SkipPaths`

Several approaches were rejected or restricted:

* `defaults read` + `tr`: breaks paths containing spaces.
* Regular expression on quotes: `defaults read` only quotes strings with special characters, so most paths are invisible.
* `PlistBuddy`: reads the file directly, bypassing the `cfprefsd` cache; reads can be stale and **writes can be overwritten**. Worse, the file is protected: without Full Disk Access it cannot even be opened (`PlistBuddy` answers "File Doesn't Exist, Will Create"). **No longer used.**
* `defaults export` + `plutil`: goes through `cfprefsd`, needs neither `sudo` nor Full Disk Access and returns the live value. **Used everywhere** (`list`, `status`, `diff`, `apply`, `remove`, `doctor`).

The source of truth for whether a path is excluded is `tmutil isexcluded`, which needs no privileges.

## Non-existent paths

Verified on macOS 26 (with Full Disk Access granted to the terminal): `sudo tmutil addexclusion -p ~/some-absent-path` **succeeds** and the path appears in `SkipPaths`; `sudo tmutil removeexclusion -p` on the same absent path **succeeds** and removes it. So absent paths can be registered *and* removed with `tmutil` alone, which is what `apply` and `remove` do (both re-read `SkipPaths` afterwards to check the result and report `[FAILED]` with the `tmutil` message otherwise). There is deliberately **no fallback that writes the preferences directly** (a `defaults write ... -array-add` fallback would add duplicates on every run). Without Full Disk Access, `tmutil` refuses everything ("requires Full Disk Access privileges"), which is why `apply`, `remove` and `install` check it first.

## Running under `sudo`

Running a script with `sudo ./script` can make `$HOME` point to `/var/root`. The `leantimemachine` command is therefore run **without** `sudo` and elevates by itself the commands that need it (`sudo tmutil ...`).

## Why a user agent and not a root daemon?

The first design used a root `LaunchDaemon`. It was replaced after checking, on macOS 26, that the hourly job needs **no privileges**:

* `tmutil localsnapshot`, `tmutil deletelocalsnapshots <date>` and sticky `tmutil addexclusion` all succeed as a normal user, including when started by a `launchd` **agent** (tested with a temporary agent).
* Only `tmutil addexclusion -p` and `tmutil enable` require root (documented in `man tmutil`). They are one-off actions, run through `sudo` when you execute `apply` or `install`.

A user agent means no code running as root on a schedule, no root-owned copy of the program to protect, and a much smaller attack surface. Trade-offs: it runs only while the user is logged in, and it only covers that user's home folder.

Note: `tmutil deletelocalsnapshots` on a date that does not exist fails with `Stale NFS file handle` (POSIX 70). That message therefore means "snapshot not found", not a permissions problem.

Like any process started by `launchd`, the agent does not inherit your terminal's Full Disk Access: `Desktop` and `Documents` are denied (`Operation not permitted`) unless `/bin/bash` is added to Full Disk Access.

## Excluding does not delete anything retroactively

Exclusions do not erase what is already on the backup disk. Old copies disappear through Time Machine's normal thinning.

## Leftover sticky exclusions

If sticky exclusions were set by hand on the same paths as `rules.conf`, both mechanisms coexist without conflict, but with two sources to audit. To remove one: `tmutil removeexclusion <path>`.
