# LeanTimeMachine

🇫🇷 [Version française](README.fr.md)

Keep **hourly Time Machine backups** without letting them eat your Mac's disk space.

LeanTimeMachine does three things, using only native macOS tools (`tmutil`, `launchd`):

1. **Excludes regenerable data from backups**: VM disks, container storage, AI models, caches, package-manager stores.
2. **Automatically excludes development build folders** (`node_modules`, `target`, `build`, `.venv`...) in your projects, wherever they appear.
3. **Keeps a single local snapshot** (the most recent one) instead of up to 24 hourly ones.

Time Machine itself stays enabled and unchanged in Settings: it keeps doing the backups. LeanTimeMachine only makes them lighter.

> **Status and disclaimer.** Young project, developed and tested on a single machine (macOS 26, Apple silicon). `install` sets up a background agent that runs **every hour under your own user account (no root)** and **deletes local Time Machine snapshots except the most recent one**. `sudo` is only used to apply the fixed-path exclusions and to enable Time Machine. Backups already on your external disk are never modified, but read this README and run `leantimemachine status` / `doctor` before installing. Provided "as is", without warranty (see [LICENSE](LICENSE)).

## Why

On APFS, Time Machine **always creates a local snapshot** before each backup (the frozen source of the copy). There is no setting to disable this. macOS then keeps these snapshots, roughly one per hour for 24 hours. On a machine where VM disks, containers or model files change a lot, those snapshots can pin tens of gigabytes.

Exclusions alone don't fix it: a snapshot is a picture of the **whole volume**, excluded files included. So LeanTimeMachine does both:

| | Reduces | Does not reduce |
| :--- | :--- | :--- |
| Exclusions | backup size and duration on the external disk | space used by local snapshots |
| Deleting old snapshots | space used by local snapshots | — |

You cannot get "hourly backups without a local snapshot", but you can keep just one: the latest, which only holds what changed since the last backup.

## Language

Program messages follow the macOS system language (English or French). To force one: `LEANTIMEMACHINE_LANG=fr leantimemachine doctor` (or `en`). The **log** (`~/Library/Logs/LeanTimeMachine.log`) is always in English because `doctor` parses its format.

## Requirements

* macOS with APFS (developed on macOS 26; no other version tested).
* Bash 3.2 (the system one) — no other dependency.
* At least one Time Machine destination configured in System Settings.
* Administrator rights (`sudo`) for installation and for applying exclusions.
* **Full Disk Access for your terminal** (System Settings → Privacy & Security → Full Disk Access → add Terminal / iTerm / Ghostty / Warp..., then quit and reopen it). `tmutil` refuses `addexclusion -p`, `removeexclusion` and `enable` without it (`tmutil: addexclusion requires Full Disk Access privileges`). `apply`, `remove` and `install` check this first and explain what to do.

## Quick start

```sh
git clone https://github.com/brainztorm/LeanTimeMachine.git
cd LeanTimeMachine

./bin/leantimemachine status      # what is currently backed up needlessly (no sudo)
./bin/leantimemachine install     # install the hourly agent + apply exclusions (sudo for the exclusions)
./bin/leantimemachine doctor      # check that everything works
```

`install` asks for your password, then:

1. copies the program to `~/Library/Application Support/LeanTimeMachine/`;
2. creates the `launchd` **agent** `com.leantimemachine.maintenance` in `~/Library/LaunchAgents/` (runs at login, then every hour, **as you, without root**);
3. applies the exclusions listed in `config/rules.conf` (**asks for sudo**: `tmutil addexclusion -p` requires it);
4. enables Time Machine if it was disabled (**sudo** as well).

Re-run `install` after updating the program: the agent runs the installed copy, not the one in the project folder.

### Use it from any terminal (optional)

```sh
ln -sfn "$PWD/bin/leantimemachine" ~/.local/bin/leantimemachine     # ~/.local/bin must be in your PATH

# zsh completion: add to ~/.zshrc, BEFORE compinit / oh-my-zsh is loaded
fpath=("/path/to/LeanTimeMachine/completions" $fpath)
```

The program follows the symlink to find `config/rules.conf` in the project.

## Commands

| Command | What it does | sudo |
| :--- | :--- | :---: |
| `install` | Installs the hourly agent, applies `rules.conf`, enables Time Machine if needed | yes |
| `uninstall` | Removes the agent. Time Machine is left as is. `--remove-exclusions` also removes the exclusions | only with `--remove-exclusions` |
| `doctor` | Health check: what is fine, what is wrong, and the command to fix it | no |
| `status` | State and size of every rule in `rules.conf` | no |
| `apply` | Excludes every path of `rules.conf` (existing **or not**; safe to re-run) | yes |
| `remove` | Removes the exclusions of `rules.conf` | yes |
| `list` | Shows the exclusions stored in Time Machine (`SkipPaths`) | no |
| `run` | Runs the agent's hourly job now (what the agent does every hour) | no |
| `diff` | Rules to apply / already excluded / absent | no |
| `help` | Help | no |

Options: `-c, --config FILE` (alternate rules file), `--no-size` (`status` without sizes, faster), `--create` (`apply` creates missing folders first), `--remove-exclusions` (with `uninstall`).

## Fixed-path exclusions: `config/rules.conf`

```text
# comment
~/.cache
~/Library/Application Support/com.apple.container
```

* One path per line; `#` starts a comment; blank lines are ignored.
* `~` and `$HOME` are expanded; spaces are allowed without quotes; no wildcards.
* A folder excludes its sub-folders too.
* Paths are registered with `tmutil addexclusion -p` (stored in Time Machine's `SkipPaths`, visible in **System Settings → General → Time Machine → Options**). They survive the folder being deleted and recreated, and they apply **even if the software is not installed** (yet or anymore).

Edit the file, then run `leantimemachine apply`. Once excluded, a path stays excluded.

The default file covers containers and VMs (Docker, Colima, OrbStack, Tart, UTM, Parallels, VirtualBox, Rancher, Incus, Lima...), AI models (Ollama, LM Studio, MLX...), package-manager caches (npm, pnpm, Cargo, Gradle, Maven, sbt, conda...), Xcode, and IDE caches.

### Deliberately NOT excluded

Many tool folders also hold configuration or secrets, so only their cache sub-folders are listed:

| Excluded | Kept |
| :--- | :--- |
| `~/.ollama/models` | the rest of `~/.ollama` (keys) |
| `~/.lmstudio/models` | conversations and presets |
| `~/.cargo/registry`, `~/.cargo/git` | `~/.cargo/bin` (installed tools) |
| `~/.gradle/caches` | `gradle.properties`, `init.d` |
| — | `~/.docker` (config, contexts) |
| — | Homebrew: `brew upgrade` does not reinstall missing packages, so excluding `Cellar` would require maintaining a `Brewfile` |

> **Warning — containers and VMs.** They are excluded by default: after restoring from Time Machine they will **not** be there, including named volumes (databases...) and VM contents. Images can be pulled again, that data cannot. Export what matters.

## Automatic exclusion of development folders

Every hour the agent looks for folders generated by development tools and excludes them. The scope is **development projects only**: documents, photos, music or video projects are never touched. A folder is excluded only if its name matches **and**, for any ambiguous name (`build`, `dist`, `target`, `vendor`, `env`...), a marker file of the project confirms it is generated. Covered: JavaScript, TypeScript, HTML/CSS tooling, Python, Java, Kotlin, Scala, C, C++, PHP, Go, Rust, Swift, Ruby, Lua, Dart/Flutter and Terraform.

| Language / ecosystem | Folders | Excluded when... |
| :--- | :--- | :--- |
| JavaScript, TypeScript | `node_modules`, `.next`, `.nuxt`, `.turbo`, `.parcel-cache`, `.svelte-kit` | always |
| | `.angular` | the parent contains `angular.json` |
| | `dist` | the parent contains `package.json` or `tsconfig.json` (or `setup.py` / `pyproject.toml`) |
| HTML/CSS (tooling) | `.sass-cache` | always |
| Python | `.tox`, `.nox`, `.pytest_cache`, `.mypy_cache`, `.ruff_cache`, `.ipynb_checkpoints` | always |
| | `.venv`, `venv`, `env` | it contains `pyvenv.cfg` |
| | `*.egg-info` | the parent contains `setup.py` or `pyproject.toml` |
| Java, Kotlin, Scala | `.gradle` | always |
| | `target` | the parent contains `pom.xml` or `build.sbt` |
| | `.kotlin` | the parent contains `build.gradle.kts` or `settings.gradle.kts` |
| | `out` | the parent contains an `.idea` folder (IntelliJ output) |
| Rust | `target` | the parent contains `Cargo.toml` |
| C, C++ | `build` | the parent contains `CMakeLists.txt`, `Makefile` or `meson.build` |
| | `cmake-build-*`, `CMakeFiles` | `CMakeLists.txt` in the parent / always |
| | `builddir` | the parent contains `meson.build` |
| | `.cache` | the parent contains `CMakeLists.txt` or `compile_commands.json` (clangd index) |
| Any Gradle / Maven / JS / Python project | `build` | the parent contains `build.gradle(.kts)`, `settings.gradle(.kts)`, `pom.xml`, `package.json`, `setup.py` or `pyproject.toml` |
| PHP, Go, Ruby | `vendor` | the parent contains `composer.json`, `go.mod` or `Gemfile` |
| Swift | `.build` | the parent contains `Package.swift` |
| | `Pods` | the parent contains a `Podfile` |
| Lua | `lua_modules` | always |
| Infra / Dart | `.terraform`, `.dart_tool` | always |
| **SQL, Bash/Shell** | *nothing* | no generated folders to exclude. Database files and scripts are never touched |

* It scans the **visible** folders of your home (`code`, `Desktop`, `Documents`, `Downloads`...). Skipped: `Library`, `Movies`, `Music`, `Pictures`, `Applications`, `Public`, hidden folders, `.git` and any top-level folder already excluded as a whole. Generic names (`vendor`, `env`, `build`...) without a marker are not excluded, but the projects inside them are still scanned.
* `__pycache__` is deliberately ignored (tiny but numerous; one `tmutil` call each would make the scan slow).
* Exclusions are "sticky" (an extended attribute on the folder itself), so they follow the folder if it is moved. A scan takes a few seconds.

## Snapshot cleanup

On every run, unless a backup is in progress, the agent lists local snapshots (`tmutil listlocalsnapshotdates /`) and deletes all but the most recent with `tmutil deletelocalsnapshots`. Each deletion and each failure is logged to `~/Library/Logs/LeanTimeMachine.log` (last 500 lines kept).

```sh
tmutil listlocalsnapshots /        # should show one (two right after a backup)
tail ~/Library/Logs/LeanTimeMachine.log
```

**Limits:** the agent runs while you are logged in. A new snapshot may appear between two runs and is removed at the next one (at most an hour later). Deleted snapshots do not come back, but backups on the external disk are untouched. macOS also treats local snapshots as purgeable and may free them itself when space is low. Behaviour on macOS versions after 26 is untested.

## `doctor`

```sh
leantimemachine doctor
```

No sudo needed. It prints `✓` (fine), `!` (warning) or `✗` (problem), and the **command that fixes** each issue. It checks: automatic backup enabled; each backup disk (plugged in or not, date and age of its last backup, read from preferences **without mounting the disk**); number of local snapshots and free space; whether your terminal has Full Disk Access; agent installed, loaded, owned by you (not writable by others) and up to date; time of the last run and errors of the last 24 hours (including folders the agent could not scan); rules not yet applied, including absent paths not yet registered. Exit code is 1 if there is at least one `✗`.

## Uninstall

```sh
./bin/leantimemachine uninstall                       # removes the agent, program copy and log
./bin/leantimemachine uninstall --remove-exclusions   # also removes all exclusions
```

Time Machine itself is never disabled. With `--remove-exclusions`, the removal of project-folder exclusions applies to **every** folder matching the rules above, including ones you may have excluded by hand. After uninstalling, macOS goes back to keeping its hourly snapshots. To remove the terminal integration: delete `~/.local/bin/leantimemachine` and the `fpath=` line.

## Troubleshooting

* **`Operation not permitted`**: give **Full Disk Access** to your terminal; for the agent, add `/bin/bash` (System Settings → Privacy & Security → Full Disk Access). Without it macOS denies `Desktop` and `Documents` to the agent (`Operation not permitted`).
* **`tmutil: ... requires Full Disk Access privileges`**: your terminal lacks Full Disk Access (see Requirements). `leantimemachine doctor` tells you.
* **A path stays `[INCLUDED]` after `apply`**: re-run `apply` and read the `tmutil` error printed under `[FAILED]`.
* **More than one snapshot remains**: read the log (`FAILED` lines); a snapshot may exist until the next run.
* **Run the job now**: `leantimemachine run` (no sudo), then `tail ~/Library/Logs/LeanTimeMachine.log`. Note that a manual `run` uses *your terminal's* permissions, not the agent's. To test the agent itself: `launchctl kickstart -k gui/$(id -u)/com.leantimemachine.maintenance`, wait ~10 s, then `leantimemachine doctor`.
* **Check one path**: `tmutil isexcluded "$HOME/.colima"`.

## Project layout

```text
bin/leantimemachine          the program (install, uninstall, doctor, status, apply, remove, list, diff)
config/rules.conf            fixed-path exclusion rules
completions/_leantimemachine zsh completion
docs/en/, docs/fr/           architecture and design decisions (English / French)
README.fr.md                 full manual (French)
```

## License

[MIT](LICENSE)
