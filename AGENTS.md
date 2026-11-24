# AGENTS.md — Règles d'Ingénierie pour Amazonia : Dernier Rempart (BGA Studio)

Ce dépôt contient le portage Board Game Arena (BGA Studio) du jeu coopératif asymétrique **Amazonia : Dernier Rempart** (schéma canonique v6.2+).
Toute modification opérée par un agent IA doit respecter impérativement les règles ci-dessous.

## 1. Sources de Vérité et Fichiers Générés (SSOT)
- Les règles canoniques, invariants physiques et définitions des tuiles résident EXCLUSIVEMENT dans `contracts/*.json` (synchronisés depuis `../amazonia-sim-lab/contracts/`).
- **INTERDICTION ABSOLUE** d'éditer manuellement les fichiers situés dans `modules/php/Generated/` et `src/ts/generated/`. Toute mise à jour passe par `python3 scripts/sync_contracts.py`.
- Dès qu'un fichier figure dans `contracts/kernel_locks.json`, il est **VERROUILLÉ** : ne jamais le modifier sans instruction explicite de déverrouillage de l'utilisateur. Vérifier systématiquement avec `python3 scripts/check_locks.py`.

## 2. Standard BGA Moderne Exclusif (PHP 8.4 + ESM/TypeScript)
- **Serveur PHP :** Tous les fichiers PHP commencent par `declare(strict_types=1);` sous le namespace PSR-4 du jeu dans `modules/php/`.
- **API BGA Actuelle Uniquement :**
  - Étendre `\Bga\GameFramework\Table` dans `modules/php/Game.php`.
  - Utiliser les classes d'états dans `modules/php/States/*.php` étendant `\Bga\GameFramework\States\GameState` avec l'attribut `#[PossibleAction]` et l'enum `\Bga\GameFramework\StateType`.
  - Utiliser `$this->bga->globals`, `$this->bga->notify`, `$this->bga->tableOptions`, `$this->bga->playerScore`, `$this->bga->tableStats` et `\Bga\GameFramework\UserException`.
  - Pour tout tirage aléatoire côté BGA, passer par `RngPort` qui encapsule `bga_rand(int $min, int $max)` en production (jamais `rand()`, `mt_rand()` ni `RAND()` SQL).
- **BANNISSEMENT DU LEGACY BGA :** Ne JAMAIS créer ni référencer `states.inc.php`, `material.inc.php`, `*.action.php`, `*.view.php`, `*.tpl`, l'ancien module `Deck`, `self::ajaxcall`, ni `dojo.declare`.
- **Isolation Serveur BGA :** Zéro appel réseau externe, zéro exécution shell au runtime PHP, zéro dépendance Composer requise sur le serveur BGA (`vendor/` sert uniquement aux outils locaux hors SFTP).
- **Client UI :** Coder dans `src/ts/*.ts` et compiler vers l'unique bundle ESM `modules/js/Game.js` en utilisant l'API `this.bga.*`.

## 3. Architecture Découplée & Pile de Résolution
- **Frontière `Domain/` vs BGA :** Aucune classe dans `modules/php/Domain/` ne doit importer `\Bga\GameFramework\*` ni exécuter de SQL. Toute la logique métier opère sur `GameStateData`, utilise `RngPort`, et émet des `DomainEvent` typés.
- **Zéro Récursion d'États :** Les étapes 3 à 6 et le balayage gauche -> droite des 6 icônes Gardiens (`[Parade, Piégeage, Sabotage, Rêve, Insecte, Résonance]`) doivent passer exclusivement par `ResolutionQueue` sérialisée dans `GameStateData`. Auto-résoudre tout arbitrage ayant 0 ou 1 seule option légale, sauf effet explicitement facultatif au contrat.

## 4. Pare-feu de Silence Tactique & Anti-Triche Bots
- **Étapes 1 et 2 (Paravents fermés) :** `PlayerViewFilter` et toute notification envoyée avant l'étape 3.2 ne doivent JAMAIS exposer :
  1. L'ordre ou le détail des jetons restants dans le sac opaque (seul l'effectif total restant est public).
  2. Le jeton pioché par un autre joueur ni son lancer de dé.
  3. Le jeton caché sous le Perchoir de Vigie à un joueur autre que le Peuple Oiseaux.
- **Lancer de dés universel :** À l'étape 1.2, générer un dé 1-6 pour TOUS les joueurs (marqué `ignored: true` pour le propriétaire s'il a tiré Sabotage ou Écaille) afin d'éliminer toute fuite structurelle ou temporelle.
- **Bots loyaux :** Aucun code dans `modules/php/Domain/Bot/` ne lit les champs secrets de `GameStateData` ; le bot ne reçoit que `PlayerViewFilter::forBot(...)`.

## 5. Protocole d'Exécution & Portes de Qualité (Quality Gates)
- Toujours utiliser des commandes CLI et Git non-interactives (`git --no-pager ...`), jamais d'éditeur bloquant. Ne jamais lire ni versionner `.env.bga`.
- Aucune tâche n'est terminée tant que ces vérifications locales ne passent pas à 100 % :
  1. `find modules tests -name "*.php" -print0 | xargs -0 -n1 php -l`
  2. `python3 scripts/sync_contracts.py --check`
  3. `python3 scripts/check_locks.py`
  4. `php tests/smoke.php` (puis `fuzz_game.php` et `verify_canon.php` dès leur ajout)
  5. `npm run build`
