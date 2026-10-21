# Architecture

🇫🇷 [Version française](../fr/ARCHITECTURE.md)

Technical overview of LeanTimeMachine. For usage see the [README](../../README.md) (English) or the [manual](../../README.fr.md) (French). The reasons behind the technical choices are in [DESIGN-DECISIONS.md](DESIGN-DECISIONS.md).

## Goal

Keep hourly Time Machine backups without regenerable data and APFS local snapshots filling the Mac's disk.

## Components

```text
LeanTimeMachine/
├── bin/leantimemachine          # the single program (Bash 3.2, no dependency)
├── config/rules.conf            # fixed-path exclusions (declarative)
├── completions/_leantimemachine # zsh completion
├── docs/en, docs/fr             # architecture and design decisions (English / French)
├── README.md / README.fr.md     # documentation (English / French)
└── LICENSE                      # MIT
```

`bin/leantimemachine` plays several roles depending on the command:

| Command | Role | Privileges |
| :--- | :--- | :---: |
| `status`, `diff`, `list`, `doctor` | read-only and diagnostics | no |
| `run` | the agent's hourly job, can be run by hand (locked against concurrent runs with `lockf`) | no |
| `apply`, `remove` | `tmutil addexclusion -p` / `removeexclusion -p` for each path of `rules.conf` | `sudo` |
| `install` | installs the `launchd` agent, then runs `apply` and `tmutil enable` | `sudo` only for those two steps |
| `uninstall` | removes the agent (`--remove-exclusions`: also the exclusions) | `sudo` only with `--remove-exclusions` |

## Flow

```text
install ─► copy the program to ~/Library/Application Support/LeanTimeMachine
        ─► ~/Library/LaunchAgents/com.leantimemachine.maintenance.plist (at login + every hour, as the user)
        ─► apply (rules.conf, sudo) ─► sudo tmutil enable if Time Machine is disabled

launchd agent (every hour, no root) ─► leantimemachine run
   1. scan_build_dirs add : "sticky" exclusions of development folders
   2. prune_snapshots     : tmutil deletelocalsnapshots, except the most recent
                            (nothing while a backup is running)
   3. log to ~/Library/Logs/LeanTimeMachine.log (last 500 lines)

Time Machine (macOS) ─► hourly backup, independent of this project
```

## Two exclusion mechanisms

| | Fixed paths | Project folders |
| :--- | :--- | :--- |
| Source | `config/rules.conf` | detection in the visible folders of `$HOME` |
| Type | path-based (`-p`, `SkipPaths`) | sticky (extended attribute on the folder) |
| Triggered by | `apply` (manual, also run by `install`) | the agent, every hour |
| Absent path | excluded anyway | not applicable |

Project-folder detection: a recognised name (`SCAN_NAMES`) **and**, for ambiguous names, a marker file in the parent folder (`pom.xml`, `package.json`, `Cargo.toml`, `go.mod`, `composer.json`, `Gemfile`, `pyvenv.cfg`...). Covered: JavaScript, TypeScript, HTML/CSS tooling, Python, Java, Kotlin, C, C++, PHP, Go, Rust, Swift, Ruby, Lua, Scala, Dart/Flutter, Terraform; nothing to exclude for SQL and Bash/Shell. To add an ecosystem: add the name to `SCAN_NAMES` and a case to `has_marker`. `Library`, `Movies`, `Music`, `Pictures`, `Applications`, `Public`, hidden folders and `.git` are not walked; `__pycache__` is ignored (too many, one `tmutil` call each).

## `config/rules.conf` format

* one path per line; `#` = comment; blank lines ignored;
* `~` and `$HOME` are expanded; spaces are allowed without quotes;
* no wildcards;
* a folder path also excludes its sub-folders.

Principle: exclude only regenerable sub-folders (`~/.ollama/models`, never `~/.ollama`), because the parent folder often holds configuration or secrets.

## Security

* **No root daemon.** The hourly job is a `launchd` agent running as the user, and `tmutil localsnapshot` / `deletelocalsnapshots` / sticky `addexclusion` were verified to work unprivileged (also from a launchd agent). `sudo` is needed only for `addexclusion -p` and `enable`, when you run `apply` / `install`. See [DESIGN-DECISIONS.md](DESIGN-DECISIONS.md).
* The agent executes only its own copy in `~/Library/Application Support/LeanTimeMachine`; `doctor` checks that it belongs to you and is not writable by others.
* The program touches nothing outside `tmutil`, the Time Machine preferences (read through `defaults export`; never written directly), its log and its `launchd` plist.
* Deleting snapshots (`tmutil deletelocalsnapshots`) is irreversible for those snapshots; backups on the external disk are untouched.

## Known limits

* macOS only; APFS.
* Untested: `deletelocalsnapshots` and `tmutil` behaviour on macOS versions after 26. The program was tested on macOS 26 only.
* A snapshot may remain until the agent's next run (at most 1 hour). The agent only runs while the user is logged in.
* Program messages follow the system language (English / French, override with `LEANTIMEMACHINE_LANG`); the log is always English. Code comments and documentation are bilingual.

## Future ideas

* Scanning hidden folders of the home (currently skipped for speed) for project folders.
* More ecosystems (C#/.NET, Elixir...) by adding names to `SCAN_NAMES` and a marker to `has_marker`.
