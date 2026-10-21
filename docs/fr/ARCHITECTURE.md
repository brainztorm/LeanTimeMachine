# Architecture

🇬🇧 [English version](../en/ARCHITECTURE.md)

Vue technique de LeanTimeMachine. Pour l'utilisation, voir le [README](../../README.md) (anglais) ou le [manuel](../../README.fr.md) (français).. Les raisons des choix techniques sont dans [DESIGN-DECISIONS.md](DESIGN-DECISIONS.md).

## Objectif

Garder des sauvegardes Time Machine horaires sans que les données régénérables et les snapshots locaux APFS n'occupent le disque du Mac.

## Composants

```text
LeanTimeMachine/
├── bin/leantimemachine         # programme unique (Bash 3.2, aucune dépendance)
├── config/rules.conf           # exclusions de chemins fixes (déclaratif)
├── completions/_leantimemachine# autocomplétion zsh
├── docs/en, docs/fr            # architecture et décisions de conception (anglais / français)
├── README.md / README.fr.md    # documentation (anglais / français)
└── LICENSE                     # MIT
```

`bin/leantimemachine` joue plusieurs rôles selon la commande :

| Commande | Rôle | Privilèges |
| :--- | :--- | :---: |
| `status`, `diff`, `list`, `doctor` | lecture et diagnostic | non |
| `run` | passage horaire de l'agent, lançable à la main (verrou `lockf` contre les passages simultanés) | non |
| `apply`, `remove` | `tmutil addexclusion -p` / `removeexclusion -p` pour chaque chemin de `rules.conf` | `sudo` |
| `install` | installe l'agent `launchd`, puis lance `apply` et `tmutil enable` | `sudo` seulement pour ces deux étapes |
| `uninstall` | retire l'agent (`--remove-exclusions` : aussi les exclusions) | `sudo` seulement avec `--remove-exclusions` |

## Flux

```text
install ─► copie du programme dans ~/Library/Application Support/LeanTimeMachine
        ─► ~/Library/LaunchAgents/com.leantimemachine.maintenance.plist (au login + toutes les heures, sous l'utilisateur)
        ─► apply (rules.conf, sudo) ─► sudo tmutil enable si Time Machine est désactivé

agent launchd (toutes les heures, sans root) ─► leantimemachine run
   1. scan_build_dirs add : exclusions « sticky » des dossiers de développement
   2. prune_snapshots     : tmutil deletelocalsnapshots, sauf le plus récent
                            (rien si une sauvegarde est en cours)
   3. journal ~/Library/Logs/LeanTimeMachine.log (500 dernières lignes)

Time Machine (macOS) ─► sauvegarde horaire, indépendamment du projet
```

## Deux mécanismes d'exclusion

| | Chemins fixes | Dossiers de projets |
| :--- | :--- | :--- |
| Source | `config/rules.conf` | détection dans les dossiers visibles de `$HOME` |
| Type | path-based (`-p`, `SkipPaths`) | sticky (attribut étendu sur le dossier) |
| Déclenchement | `apply` (manuel, aussi lancé par `install`) | agent, toutes les heures |
| Chemin absent | exclu quand même | sans objet |

Détection des dossiers de projets : nom reconnu (`SCAN_NAMES`) **et**, pour les noms ambigus, fichier marqueur dans le dossier parent (`pom.xml`, `package.json`, `Cargo.toml`, `go.mod`, `composer.json`, `Gemfile`, `pyvenv.cfg`...). Langages couverts : JavaScript, TypeScript, outils HTML/CSS, Python, Java, Kotlin, C, C++, PHP, Go, Rust, Swift, Ruby, Lua, Scala, Dart/Flutter, Terraform ; rien à exclure pour SQL et Bash/Shell. Pour ajouter un écosystème : ajouter le nom à `SCAN_NAMES` et un cas dans `has_marker`. `Library`, `Movies`, `Music`, `Pictures`, `Applications`, `Public`, les dossiers cachés et `.git` ne sont pas parcourus ; `__pycache__` est ignoré (trop nombreux, un appel `tmutil` chacun).

## Format de `config/rules.conf`

* un chemin par ligne ; `#` = commentaire ; lignes vides ignorées ;
* `~` et `$HOME` sont développés ; les espaces sont acceptés sans guillemets ;
* pas de caractères génériques ;
* un chemin de dossier exclut aussi ses sous-dossiers.

Principe : exclure uniquement les sous-dossiers régénérables (`~/.ollama/models`, jamais `~/.ollama`), car le dossier parent contient souvent de la configuration ou des secrets.

## Sécurité

* **Pas de daemon root.** La tâche horaire est un agent `launchd` qui tourne sous l'utilisateur ; `tmutil localsnapshot` / `deletelocalsnapshots` / `addexclusion` « sticky » fonctionnent sans privilèges (vérifié aussi depuis un agent launchd). `sudo` n'est nécessaire que pour `addexclusion -p` et `enable`, quand vous lancez `apply` / `install`. Voir [DESIGN-DECISIONS.md](DESIGN-DECISIONS.md).
* L'agent n'exécute que sa propre copie dans `~/Library/Application Support/LeanTimeMachine` ; `doctor` vérifie qu'elle vous appartient et n'est pas modifiable par d'autres.
* Le programme ne lit ni n'écrit rien en dehors de `tmutil`, des préférences Time Machine (lues via `defaults export` ; jamais écrites directement), de son journal et de son plist `launchd`.
* La suppression de snapshots (`tmutil deletelocalsnapshots`) est irréversible pour ces snapshots ; les sauvegardes sur disque externe ne sont pas touchées.

## Limites connues

* macOS uniquement ; APFS.
* Non testé : `deletelocalsnapshots` et le comportement de `tmutil` sous les versions de macOS postérieures à 26. Le programme n'a été testé que sous macOS 26.
* Un snapshot peut subsister jusqu'au prochain passage de l'agent (au plus 1 h). L'agent ne tourne que pendant la session de l'utilisateur.
* Les messages du programme suivent la langue du système (français / anglais, forçable avec `LEANTIMEMACHINE_LANG`) ; le journal est toujours en anglais. Commentaires du code et documentation sont bilingues.

## Pistes futures

* Scanner aussi les dossiers cachés du home (ignorés aujourd'hui pour la vitesse) à la recherche de dossiers de projets.
* D'autres écosystèmes (C#/.NET, Elixir...) en ajoutant des noms à `SCAN_NAMES` et un marqueur dans `has_marker`.
