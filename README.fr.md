# LeanTimeMachine — Manuel utilisateur

🇬🇧 [English version](README.md)

Garder des sauvegardes **Time Machine horaires** tout en gardant **de la place sur le disque du Mac**.

LeanTimeMachine fait trois choses, avec les outils natifs de macOS (`tmutil`, `launchd`), et se pilote avec une seule commande, `leantimemachine` :

1. **Exclure des sauvegardes** ce qui est régénérable (VM, conteneurs, modèles IA, caches, `node_modules`...).
2. **Exclure automatiquement les dossiers générés par les outils de développement** (`node_modules`, `target`, `build`, `.venv`...), qui apparaissent à des chemins imprévisibles à chaque nouveau projet.
3. **Ne garder qu'un seul snapshot local** (le plus récent) au lieu de 24 snapshots horaires.

Time Machine reste **activé normalement** dans Réglages Système : c'est lui qui sauvegarde, dès qu'un disque de sauvegarde est branché. LeanTimeMachine ne remplace pas Time Machine, il l'allège.

> **Statut et avertissement.** Projet jeune, développé et testé sur une seule machine (macOS 26, Apple silicon). `install` met en place un agent en arrière-plan qui tourne **toutes les heures sous votre propre compte (sans root)** et **supprime des snapshots locaux Time Machine, sauf le plus récent**. `sudo` ne sert qu'à appliquer les exclusions de chemins fixes et à activer Time Machine. Les sauvegardes déjà présentes sur votre disque externe ne sont jamais modifiées, mais lisez ce manuel et lancez `leantimemachine status` / `doctor` avant d'installer. Fourni « tel quel », sans garantie (voir [LICENSE](LICENSE)).

---

## 1. Comprendre le problème

### Pourquoi des snapshots locaux ?

Sur un Mac en APFS, Time Machine **crée toujours un snapshot local** du volume avant chaque sauvegarde : c'est la source figée de la copie. Il n'existe aucun réglage pour sauvegarder « sans snapshot local ».

Par défaut, macOS **conserve** ces snapshots : un par heure pendant environ 24 h, plus le dernier snapshot de chaque disque de sauvegarde non branché récemment. Un snapshot ne contient que les blocs modifiés depuis sa création, mais sur une machine où des VM, des conteneurs ou des modèles IA changent souvent, 24 snapshots retiennent beaucoup de blocs.

### Pourquoi les exclusions seules ne suffisent pas

Un snapshot est une photo du **volume entier**, exclusions comprises. Si un fichier exclu (disque de VM, modèle) est modifié ou supprimé, le snapshot garde quand même ses anciens blocs.

| Mécanisme | Réduit | Ne réduit pas |
| :--- | :--- | :--- |
| Exclusions | La taille et la durée de la sauvegarde sur le disque externe | L'espace pris par les snapshots locaux |
| Suppression des anciens snapshots | L'espace local pris par les snapshots | — |

D'où les deux parties du projet : exclusions **et** nettoyage des snapshots.

### La solution retenue : un seul snapshot, le plus récent

On laisse macOS créer ses snapshots et sauvegarder normalement, puis un agent supprime toutes les heures tous les snapshots sauf le plus récent. Le snapshot restant ne contient que ce qui a changé depuis la dernière sauvegarde : il reste petit.

Choix écartés :

| Piste | Pourquoi écartée |
| :--- | :--- |
| `tmutil disable` + script qui lance `tmutil startbackup` | Réinvente ce que Time Machine fait déjà, et sort du fonctionnement normal de macOS |
| Supprimer aussi le dernier snapshot | Les sauvegardes incrémentales perdent leur point de référence (sauvegardes plus lentes, comportement non vérifié) |
| Configurer le nom du disque dans un script | Fragile : le disque peut être renommé ou remplacé. Ici, aucune configuration : n'importe quelle destination configurée dans Time Machine convient |

> **Limites connues.** L'agent ne tourne que pendant votre session. Un snapshot peut apparaître entre deux passages de l'agent : il est supprimé au passage suivant (au plus 1 h plus tard). Je n'ai trouvé aucune source officielle sur une évolution de `tmutil deletelocalsnapshots` dans macOS 27, et une erreur « Stale NFS file handle » (qui signifie en pratique « snapshot introuvable », observée ici en visant une date inexistante) a été rapportée sous macOS 26.1 : les échecs sont donc consignés dans le journal (voir Dépannage). Les snapshots locaux sont aussi « purgeables » : macOS peut les libérer lui-même quand l'espace manque.

---

## 2. Vue d'ensemble

```text
  leantimemachine install ──► copie le programme dans ~/Library/Application Support/LeanTimeMachine
                              + agent launchd "com.leantimemachine.maintenance" (toutes les heures, sous votre compte)
                              + leantimemachine apply (exclusions de config/rules.conf)
                              + active Time Machine s'il était désactivé

  Toutes les heures (agent, sans root) : leantimemachine run
     ├─► exclut les dossiers de développement des projets (node_modules, target, build...)
     └─► supprime les snapshots locaux, sauf le plus récent (rien pendant une sauvegarde)

  Time Machine (macOS, activé normalement) : sauvegarde horaire quand le disque est branché
```

| Besoin | Outil | Quand |
| :--- | :--- | :--- |
| Exclure des chemins fixes (`~/.cache`, `~/.tart`...) | `leantimemachine apply` + `config/rules.conf` | Fait par `install` ; à relancer à la main quand vous ajoutez une règle |
| Exclure les dossiers de développement des projets | agent (`run`) | Automatique, toutes les heures |
| Limiter les snapshots locaux à un seul | agent (`run`) | Automatique, toutes les heures |
| Savoir si tout va bien | `leantimemachine doctor` | Quand vous voulez |

```sh
leantimemachine install      # installe l'automatisation (sudo)
leantimemachine doctor       # fait le point et indique quoi corriger
leantimemachine uninstall    # retire l'automatisation (sans sudo, sauf avec --remove-exclusions)
```

---

## Langue du programme

Les messages s'affichent dans la langue du système macOS (français ou anglais). Pour forcer une langue : `LEANTIMEMACHINE_LANG=en leantimemachine doctor` (ou `fr`). L'aide (`leantimemachine help`) suit la même règle. Le **journal** (`~/Library/Logs/LeanTimeMachine.log`) reste toujours en anglais, car son format est lu par `doctor`.

---

## 2bis. Intégration au terminal

Le projet reste dans son dossier ; le terminal y accède par un **lien symbolique** et une **autocomplétion zsh**. Une modification du projet est donc immédiatement prise en compte (rien à recopier, sauf pour l'agent : `leantimemachine install`).

```sh
# 1. la commande, partout (~/.local/bin est dans le PATH)
ln -sfn "$HOME/code/projects/me/LeanTimeMachine/bin/leantimemachine" ~/.local/bin/leantimemachine

# 2. l'autocomplétion : dans ~/.zshrc, AVANT `source $ZSH/oh-my-zsh.sh` (qui lance compinit)
fpath=("$HOME/code/projects/me/LeanTimeMachine/completions" $fpath)   # adaptez au dossier où vous avez cloné le projet
```

Puis ouvrez un nouveau terminal : `leantimemachine <Tab>` propose les commandes et leurs options.

Le programme suit le lien pour retrouver `config/rules.conf` dans le projet. Pour retirer l'intégration : `rm ~/.local/bin/leantimemachine` et supprimez la ligne `fpath=` de `~/.zshrc`.

Si vous déplacez le projet, refaites le lien et corrigez la ligne `fpath=`.

---

## 3. Exclusions de chemins fixes

### Principe

Vous listez des chemins dans `config/rules.conf`. `leantimemachine apply` les exclut avec `tmutil addexclusion -p` (exclusion à chemin fixe, inscrite dans `SkipPaths`).

Avantages de l'exclusion à chemin fixe :

* elle survit à la suppression et à la recréation du dossier ;
* elle apparaît dans **Réglages Système → Général → Time Machine → Options** ;
* **elle s'applique même si le chemin n'existe pas** : un logiciel désinstallé (UTM) ou pas encore installé (Tart, OrbStack...) est déjà exclu le jour où il revient.

Une fois exclu, c'est exclu : il n'y a rien à « mettre à jour », sauf quand vous ajoutez une règle.

### Installation de la CLI

Aucune dépendance (Bash + outils macOS). Pour l'utilisation dans le terminal, voir la section 2bis ; sinon, depuis le dossier du projet : `./bin/leantimemachine help`.

### Utilisation rapide

```sh
leantimemachine status      # 1. voir ce qui est sauvegardé à tort (sans sudo)
leantimemachine apply       # 2. appliquer les exclusions (demande le mot de passe sudo)
leantimemachine list        # 3. vérifier SkipPaths
```

### Commandes

| Commande | Rôle | sudo |
| :--- | :--- | :---: |
| `status` | Affiche l'état et la taille de chaque règle, plus le volume total à économiser | non |
| `apply` | Exclut chaque chemin (présent **ou absent**, relançable sans risque), puis vérifie avec `tmutil isexcluded` | oui |
| `remove` | Retire les exclusions des chemins de `rules.conf` (restauration), y compris les chemins absents | oui |
| `list` | Affiche `SkipPaths` (lu sans sudo, via les préférences) | non |
| `run` | Lance maintenant le passage horaire de l'agent | non |
| `diff` | Montre les règles présentes sur disque mais pas encore exclues (préfixe `+`) | non |
| `install` | Installe l'agent horaire, applique `rules.conf`, active Time Machine si besoin (voir section 6) | oui |
| `uninstall` | Retire l'agent (voir section 7) | seulement avec `--remove-exclusions` |
| `doctor` | Fait le point et indique quoi corriger (voir section 8) | non |
| `help` | Aide | non |

| Option | Effet |
| :--- | :--- |
| `-c`, `--config FICHIER` | Utilise un autre fichier de règles |
| `--no-size` | `status` : ne calcule pas les tailles (plus rapide) |
| `--remove-exclusions` | `uninstall` : retire aussi les exclusions |
| `--create` | `apply` : crée les dossiers absents avant de les exclure (par défaut, un chemin absent est inscrit sans créer le dossier) |

### Lire la sortie de `status`


```text
[INCLUS]   154.1G   ~/Library/Application Support/com.apple.container
[EXCLU]         -   ~/.cargo/registry
[ABSENT]        -   ~/.orbstack
```

* `[INCLUS]` : le chemin existe et est sauvegardé. La taille indique ce que vous économiserez.
* `[EXCLU]` : exclu (d'après `tmutil isexcluded`), par LeanTimeMachine ou nativement par macOS.
* `[ABSENT]` : le chemin n'existe pas. `status` ne vérifie pas `SkipPaths` pour ces chemins ; `apply` les inscrit quand même, et `list` le confirme.

Le calcul des tailles peut prendre plusieurs secondes sur de gros dossiers. Utilisez `--no-size` pour un affichage instantané.

### Écrire ses règles : `config/rules.conf`

```text
# commentaire
~/.cache
~/Library/Application Support/com.apple.container
```

* Une ligne = un chemin ; les lignes vides et celles commençant par `#` sont ignorées.
* `~` et `$HOME` sont développés.
* Les espaces sont acceptés sans guillemets.
* Un chemin sur un dossier exclut aussi ses sous-dossiers.
* Pas de caractères génériques (`*`) : pour les chemins variables, voir la partie suivante.

Contenu par défaut : conteneurs et VM (Docker, Colima, OrbStack, Tart, UTM, Parallels, VirtualBox, Rancher, Incus...), modèles IA (Ollama, LM Studio, omlx, MacWhisper, ModelScope...), caches de package managers (npm, Cargo, Gradle, Maven, sbt, conda...), Xcode, caches d'IDE, `IndexedDB` de Chrome.

### Ce qui est volontairement NON exclu

Des dossiers contiennent aussi de la configuration ou des secrets. LeanTimeMachine n'exclut donc que leurs sous-dossiers de cache :

| Exclu | Conservé |
| :--- | :--- |
| `~/.lmstudio/models` | conversations et presets LM Studio |
| `~/.ollama/models` | clés `id_ed25519` |
| `~/.omlx/models`, `~/.omlx/cache` | `api_key`, `settings.json`, statistiques |
| `~/.cargo/registry`, `~/.cargo/git` | `~/.cargo/bin` (outils installés) |
| `~/.gradle/caches`, `wrapper`, `daemon` | `gradle.properties`, `init.d` |
| — | `~/.docker` (`config.json`, contextes) |
| — | **Homebrew** (`/opt/homebrew`, 11,5 Go) : gardé pour qu'une restauration fonctionne directement |

> **Pourquoi Homebrew n'est pas exclu.** `brew update && brew upgrade` ne **réinstalle pas** des paquets manquants : il met à jour ceux qui sont déjà installés. Si `Cellar` était exclu, brew ne verrait rien après une restauration. Il faudrait un `Brewfile` (`brew bundle dump`, puis `brew bundle install`) à maintenir. Pour 11,5 Go, le choix retenu est de ne rien exclure.

### Attention : conteneurs, VM et navigateur

* Les dossiers de conteneurs et de VM sont exclus par défaut. **Si vous restaurez depuis Time Machine, ils n'y seront pas**, y compris les volumes nommés (bases de données, etc.) et le contenu des VM. Les images se re-téléchargent, pas ces données. Exportez ce qui compte avant de dépendre de la restauration.
* `Chrome/Default/IndexedDB` est exclu (22 Go sur la machine d'origine) : les données locales que des sites y stockent (accès hors ligne, brouillons non synchronisés) ne seront pas restaurées. Mots de passe, favoris et extensions ne sont pas concernés.

---

## 4. Exclusions automatiques des dossiers de développement

### Périmètre exact

**Seulement les dossiers générés par des outils de développement**, reconnus par leur nom *et* confirmés par un fichier du projet. Cela ne concerne pas tout dossier de « projet » au sens large : documents, photos, projets musicaux (Ableton...), vidéos, etc. ne sont **jamais** touchés. Si vous ne développez pas, cette fonction ne trouvera simplement rien.

| Langage / écosystème | Dossiers | Exclu seulement si... |
| :--- | :--- | :--- |
| JavaScript, TypeScript | `node_modules`, `.next`, `.nuxt`, `.turbo`, `.parcel-cache`, `.svelte-kit` | toujours |
| | `.angular` | le dossier parent contient `angular.json` |
| | `dist` | le parent contient `package.json` ou `tsconfig.json` (ou `setup.py` / `pyproject.toml`) |
| HTML/CSS (outils) | `.sass-cache` | toujours |
| Python | `.tox`, `.nox`, `.pytest_cache`, `.mypy_cache`, `.ruff_cache`, `.ipynb_checkpoints` | toujours |
| | `.venv`, `venv`, `env` | il contient `pyvenv.cfg` |
| | `*.egg-info` | le parent contient `setup.py` ou `pyproject.toml` |
| Java, Kotlin, Scala | `.gradle` | toujours |
| | `target` | le parent contient `pom.xml` ou `build.sbt` |
| | `.kotlin` | le parent contient `build.gradle.kts` ou `settings.gradle.kts` |
| | `out` | le parent contient un dossier `.idea` (sortie IntelliJ) |
| Rust | `target` | le parent contient `Cargo.toml` |
| C, C++ | `build` | le parent contient `CMakeLists.txt`, `Makefile` ou `meson.build` |
| | `cmake-build-*`, `CMakeFiles` | `CMakeLists.txt` dans le parent / toujours |
| | `builddir` | le parent contient `meson.build` |
| | `.cache` | le parent contient `CMakeLists.txt` ou `compile_commands.json` (index clangd) |
| Tout projet Gradle / Maven / JS / Python | `build` | le parent contient `build.gradle(.kts)`, `settings.gradle(.kts)`, `pom.xml`, `package.json`, `setup.py` ou `pyproject.toml` |
| PHP, Go, Ruby | `vendor` | le parent contient `composer.json`, `go.mod` ou `Gemfile` |
| Swift | `.build` | le parent contient `Package.swift` |
| | `Pods` | le parent contient un `Podfile` |
| Lua | `lua_modules` | toujours |
| Infra / Dart | `.terraform`, `.dart_tool` | toujours |
| **SQL, Bash/Shell** | *rien* | pas de dossier généré à exclure ; les fichiers de base de données et les scripts ne sont jamais touchés |

Langages couverts : JavaScript, TypeScript, outils HTML/CSS, Python, Java, Kotlin, Scala, C, C++, PHP, Go, Rust, Swift, Ruby, Lua, Dart/Flutter et Terraform.

Pour les noms ambigus (`build`, `dist`, `target`, `vendor`, `env`...), le fichier « marqueur » évite d'exclure par erreur un dossier simplement ainsi nommé qui contiendrait autre chose que du code généré.

Ces dossiers sont régénérables (`npm install`, `mvn package`...). Les exclure ne perd rien, sauf si vous aviez *modifié à la main* des fichiers à l'intérieur.

### Où il cherche

Les dossiers **visibles** du home (`code`, `Desktop`, `Documents`, `Downloads`...), de l'utilisateur. Sont ignorés : `Library`, `Movies`, `Music`, `Pictures`, `Applications`, `Public`, les dossiers cachés (`.cache`, `.colima`...), les dépôts `.git` et tout dossier de premier niveau déjà exclu en entier. Un dossier à nom générique (`vendor`, `env`, `build`...) sans marqueur n'est pas exclu, mais les projets qu'il contient sont quand même parcourus. Ces zones contiennent des centaines de Go de VM et de modèles déjà exclus et rendraient le scan lent. Conséquence : les `node_modules` cachés (extensions VS Code, par exemple) restent sauvegardés, ce qui est négligeable.

`__pycache__` est volontairement ignoré : minuscule, mais présent par milliers (un appel à `tmutil` par dossier rendrait le scan très lent).

### Fonctionnement

Chaque dossier repéré reçoit une exclusion « sticky » (attribut posé sur le dossier lui-même, qui le suit s'il est déplacé). Le scan est refait toutes les heures, donc un nouveau `node_modules` créé à 10 h est exclu avant la sauvegarde de 11 h. Durée mesurée : environ 3 à 6 s par passage.

---

## 5. Nettoyage des snapshots locaux

À chaque passage, l'agent :

1. ne touche à aucun snapshot si une sauvegarde est en cours (`tmutil status`) ;
2. liste les snapshots locaux (`tmutil listlocalsnapshotdates /`), triés par date ;
3. supprime tous sauf le plus récent avec `tmutil deletelocalsnapshots <date>` ;
4. note chaque succès et chaque échec dans `~/Library/Logs/LeanTimeMachine.log` (limité aux 500 dernières lignes).

Vérifier à la main :

```sh
tmutil listlocalsnapshots /          # doit en montrer un seul (ou deux juste après une sauvegarde)
tail ~/Library/Logs/LeanTimeMachine.log
```

---

## 6. Installation

### Prérequis

* macOS avec APFS (développé sous macOS 26 ; aucune autre version testée), Bash 3.2 (celui du système), au moins une destination Time Machine configurée.
* Droits d'administrateur (`sudo`) pour l'installation et pour appliquer les exclusions.
* **Accès complet au disque pour votre terminal** (Réglages Système → Confidentialité et sécurité → Accès complet au disque → ajoutez Terminal / iTerm / Ghostty / Warp..., puis quittez-le complètement et rouvrez-le). Sans cela, `tmutil` refuse `addexclusion -p`, `removeexclusion` et `enable` (`tmutil: addexclusion requires Full Disk Access privileges`). `apply`, `remove` et `install` le vérifient d'abord et expliquent quoi faire.

### Lancer l'installation

Depuis le dossier du projet :

```sh
./bin/leantimemachine install
```

`install` demande le mot de passe sudo, puis :

1. copie le programme dans `~/Library/Application Support/LeanTimeMachine/` ;
2. écrit l'**agent** `launchd` `~/Library/LaunchAgents/com.leantimemachine.maintenance.plist` et le charge (au login, puis toutes les heures, **sous votre compte, sans root**) ;
3. applique les exclusions de `config/rules.conf` (équivalent de `leantimemachine apply` ; **demande sudo**, car `tmutil addexclusion -p` l'exige) ;
4. active Time Machine s'il était désactivé (**sudo** également).

La commande est **relançable** : elle remplace l'ancienne version. Relancez-la après toute modification du programme (l'agent exécute la copie installée, pas celle du projet). Modifier `config/rules.conf` ne demande que `leantimemachine apply`.

Ensuite : `leantimemachine doctor`.

---

## 7. Désinstallation

```sh
./bin/leantimemachine uninstall
```

Arrête l'agent et supprime le plist, le programme installé et le journal. **Time Machine n'est pas modifié** (il reste activé : c'est votre outil de sauvegarde, pas celui du projet) et les exclusions sont **conservées**.

Pour revenir complètement à l'état d'origine, retirer aussi les exclusions :

```sh
./bin/leantimemachine uninstall --remove-exclusions
```

Cela retire les exclusions de `config/rules.conf` et celles posées sur les dossiers de développement. Attention : ce dernier retrait concerne tous les dossiers qui correspondent aux critères de la section 4, **y compris ceux que vous auriez exclus vous-même à la main**.

Après désinstallation, macOS recommence à conserver ses snapshots horaires (environ 24 h). Les snapshots déjà supprimés ne reviennent pas ; les sauvegardes présentes sur le disque externe ne sont pas modifiées.

Pour supprimer le projet : effacez le dossier `LeanTimeMachine` et l'éventuel alias dans `~/.zshrc`.

---

## 8. Diagnostic : `leantimemachine doctor`

```sh
leantimemachine doctor
```

Sans sudo. Affiche `✓` (ok), `!` (avertissement) ou `✗` (problème), et pour chaque anomalie **la commande qui la corrige**. Le code de sortie est 1 s'il y a au moins un `✗`.

| Vérification | Si ce n'est pas bon |
| :--- | :--- |
| Sauvegarde automatique activée | `✗` : `sudo tmutil enable` ou Réglages Système |
| Pour chaque disque de sauvegarde : branché ou non, date et âge de la dernière sauvegarde (lus dans les préférences Time Machine, **sans monter le disque**) | `✗` si aucune destination ; `!` si aucune sauvegarde ou si la dernière date de plus de 7 jours |
| Accès complet au disque pour ce terminal | `!` : voir Prérequis / Dépannage |
| Nombre de snapshots locaux (objectif : 1) | `!` : normal juste après une sauvegarde ; sinon lire le journal |
| Espace disque libre | information |
| Agent installé, programme installé vous appartenant et non modifiable par d'autres | `✗` : `leantimemachine install` |
| Copie installée identique à celle du projet | `!` : relancer `install` |
| Agent chargé dans `launchd` | `✗` : `launchctl bootstrap gui/<uid> ...` |
| Dernier passage il y a moins de ~2 h | `✗` : `launchctl kickstart gui/<uid>/com.leantimemachine.maintenance` |
| Erreurs des dernières 24 h dans le journal (y compris dossiers que l'agent n'a pas pu scanner) | `!` : voir ci-dessous |
| Chemins absents de `rules.conf` pas encore inscrits | `!` : `leantimemachine apply` |
| Chemins de `rules.conf` existants mais encore sauvegardés | `!` : `leantimemachine apply` |

### Autres pannes

* **`tmutil` refuse l'opération / `Operation not permitted`** : donnez l'**Accès complet au disque** à votre terminal. Pour l'agent, ajoutez `/bin/bash` (Réglages Système → Confidentialité et sécurité → Accès complet au disque) : sans cela, macOS lui refuse `Desktop` et `Documents` (`Operation not permitted`).
* **`tmutil: ... requires Full Disk Access privileges`** : votre terminal n'a pas l'Accès complet au disque (voir Prérequis). `leantimemachine doctor` le signale.
* **Un chemin reste `[INCLUS]` après `apply`** : relancez `apply` et lisez le message d'erreur de `tmutil` affiché avant `[ECHEC]`.
* **Plus d'un snapshot subsiste** : lisez le journal (lignes `FAILED`, le journal est toujours en anglais). Un snapshot de plus peut exister jusqu'au passage suivant.
* **Vérification manuelle d'un chemin** : `tmutil isexcluded "$HOME/.colima"`.
* **Tester un passage sans attendre** : `leantimemachine run` (sans sudo), puis `tail ~/Library/Logs/LeanTimeMachine.log`. Attention : un `run` manuel utilise les permissions de *votre terminal*, pas celles de l'agent. Pour tester l'agent lui-même : `launchctl kickstart -k gui/$(id -u)/com.leantimemachine.maintenance`, attendre ~10 s, puis `leantimemachine doctor`.

---

## 9. Structure du projet

```text
LeanTimeMachine/
├── README.md              # documentation en anglais
├── README.fr.md           # ce manuel (français)
├── LICENSE                # MIT
├── docs/fr/, docs/en/     # architecture technique et décisions de conception (français / anglais)
├── config/rules.conf      # règles d'exclusion (chemins fixes)
├── completions/_leantimemachine  # autocomplétion zsh
└── bin/leantimemachine    # programme unique : install, uninstall, doctor, status, apply, remove, list, diff
                           # (+ commande interne `run`, utilisée par l'agent)
```
