# Décisions de conception

🇬🇧 [English version](../en/DESIGN-DECISIONS.md)

Notes techniques sur les choix faits dans LeanTimeMachine, pour ne pas avoir à les re-débattre.

## Exclusion « sticky » ou « path-based » ?

`tmutil addexclusion <chemin>` (sans `-p`) pose une exclusion **sticky** : un attribut étendu `com.apple.metadata:com_apple_backup_excludeItem` sur l'élément lui-même. Elle survit au déplacement et à la copie de l'élément, y compris vers un autre volume. Elle ne demande pas `sudo` et n'apparaît pas dans Réglages Système.

`tmutil addexclusion -p <chemin>` pose une exclusion **path-based** : le chemin est inscrit dans la clé `SkipPaths` de `/Library/Preferences/com.apple.TimeMachine.plist`. C'est la liste affichée dans Réglages Système → Général → Time Machine → Options → « Exclure des sauvegardes ».

| Situation | Sticky (xattr) | Path-based (`-p`) |
| :--- | :--- | :--- |
| Dossier déplacé ou renommé | l'exclusion suit | perdue (sans effet au nouveau chemin) |
| Dossier supprimé puis recréé | **perdue** (l'attribut disparaît avec l'inode) | **conservée**, active dès que le chemin réapparaît |
| Chemin inexistant | impossible | possible (voir ci-dessous) |
| Visible dans Réglages Système | non | oui |
| Privilèges | aucun | `sudo` |

**Choix retenu :**

* **Chemins fixes** (`config/rules.conf`) → `-p`. Beaucoup de ces dossiers (`~/.colima`, caches, stockage des conteneurs) sont supprimés puis recréés par des daemons : seule l'exclusion par chemin survit. Elle permet aussi d'exclure un logiciel désinstallé ou pas encore installé.
* **Dossiers de projets** (`node_modules`, `target`, `build`...) → sticky. Leur chemin est imprévisible et il faut que l'exclusion suive le dossier s'il est déplacé.

## Lire `SkipPaths`

Plusieurs approches ont été écartées ou limitées :

* `defaults read` + `tr` : casse les chemins contenant des espaces.
* Expression régulière sur les guillemets : `defaults read` ne met entre guillemets que les chaînes contenant des caractères spéciaux, donc la plupart des chemins sont invisibles.
* `PlistBuddy` : lit le fichier directement, en contournant le cache `cfprefsd` ; les lectures peuvent être périmées et **les écritures peuvent être écrasées**. Pire, le fichier est protégé : sans Accès complet au disque il ne peut même pas être ouvert (`PlistBuddy` répond « File Doesn't Exist, Will Create »). **Plus utilisé.**
* `defaults export` + `plutil` : passe par `cfprefsd`, ne demande ni `sudo` ni Accès complet au disque et renvoie la valeur à jour. **Utilisé partout** (`list`, `status`, `diff`, `apply`, `remove`, `doctor`).

La source de vérité pour savoir si un chemin est exclu est `tmutil isexcluded`, qui ne demande aucun privilège.

## Chemins inexistants

Vérifié sous macOS 26 (Accès complet au disque accordé au terminal) : `sudo tmutil addexclusion -p ~/un-chemin-absent` **réussit** et le chemin apparaît dans `SkipPaths` ; `sudo tmutil removeexclusion -p` sur le même chemin absent **réussit** et le retire. Les chemins absents peuvent donc être inscrits *et* retirés avec `tmutil` seul, ce que font `apply` et `remove` (tous deux relisent `SkipPaths` ensuite pour vérifier le résultat et signalent `[ECHEC]` avec le message de `tmutil` sinon). Il n'y a volontairement **aucun repli qui écrit directement les préférences** (un repli `defaults write ... -array-add` ajouterait des doublons à chaque exécution). Sans Accès complet au disque, `tmutil` refuse tout (« requires Full Disk Access privileges »), d'où la vérification préalable dans `apply`, `remove` et `install`.

## Exécution sous `sudo`

Lancer un script avec `sudo ./script` peut faire pointer `$HOME` vers `/var/root`. La commande `leantimemachine` se lance donc **sans** `sudo` et élève elle-même les commandes qui en ont besoin (`sudo tmutil ...`).

## Pourquoi un agent utilisateur et pas un daemon root ?

La première version utilisait un `LaunchDaemon` root. Elle a été remplacée après avoir vérifié, sous macOS 26, que la tâche horaire n'a **besoin d'aucun privilège** :

* `tmutil localsnapshot`, `tmutil deletelocalsnapshots <date>` et `tmutil addexclusion` « sticky » réussissent en simple utilisateur, y compris lancés par un **agent** `launchd` (testé avec un agent temporaire).
* Seuls `tmutil addexclusion -p` et `tmutil enable` exigent root (documenté dans `man tmutil`). Ce sont des actions ponctuelles, passées par `sudo` quand on lance `apply` ou `install`.

Un agent utilisateur évite tout code exécuté en root à intervalle régulier, toute copie du programme appartenant à root à protéger, et réduit fortement la surface d'attaque. Contreparties : il ne tourne que pendant la session de l'utilisateur, et ne couvre que son dossier personnel.

À noter : `tmutil deletelocalsnapshots` sur une date qui n'existe pas échoue avec `Stale NFS file handle` (POSIX 70). Ce message signifie donc « snapshot introuvable », pas un problème de droits.

Comme tout processus lancé par `launchd`, l'agent n'hérite pas de l'Accès complet au disque de votre terminal : `Desktop` et `Documents` sont refusés (`Operation not permitted`) tant que `/bin/bash` n'est pas ajouté à l'Accès complet au disque.

## Exclure ne supprime rien rétroactivement

Les exclusions n'effacent pas ce qui est déjà sur le disque de sauvegarde. Les anciennes copies disparaissent par le nettoyage normal de Time Machine.

## Exclusions sticky résiduelles

Si des exclusions sticky ont été posées à la main sur les mêmes chemins que `rules.conf`, les deux mécanismes coexistent sans conflit, mais avec deux sources à auditer. Pour en retirer une : `tmutil removeexclusion <chemin>`.
