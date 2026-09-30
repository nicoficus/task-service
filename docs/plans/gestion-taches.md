---
statut: validé   # à valider | validé | réalisé  (seul le CTO passe à « validé »)
slug: gestion-taches
portee: local       # local (un seul service) | transverse (plusieurs services)
services: [task-service]
---

# Gestion des tâches (« todo »)

## Objectif
Permettre à un utilisateur authentifié de gérer ses propres tâches : les créer, les consulter, les lister,
modifier leur titre et leur échéance, faire avancer leur statut (`TODO` → `IN_PROGRESS` → `DONE`),
les mettre en pause, les rouvrir et les supprimer. C'est la première fonctionnalité métier de `task-service`.

### Interprétation de « todo »
Dans `ubiquitous-language.md`, `TODO` n'est pas un concept autonome : c'est une valeur de `TaskStatus`
(« à faire »). Ce plan interprète donc « gestion des todo » comme **la gestion du cycle de vie de l'agrégat
`Task`**, sans introduire de nouveau concept (**confirmé par le CTO le 2026-09-30**, Q1). Deux autres
lectures sont **hors périmètre** :
- des sous-tâches ou une liste de contrôle (checklist) à l'intérieur d'une tâche ;
- des listes de tâches (regroupement nommé de tâches).

Le mot « todo » n'est pas ajouté au langage omniprésent, pour éviter un synonyme de « Tâche ».

## Prérequis (bloquants, hors de ce plan)
1. **ADR 0002 accepté** sur les points 1 à 3 (runtime, IaC, versions Java / Spring Boot). L'ADR indique :
   « Tant que les points 1 à 3 ne sont pas tranchés, aucun code de service n'est écrit. »
2. **Plan `initialisation-technique` validé puis réalisé** dans ce dépôt (squelette Maven, packages
   hexagonaux, ArchUnit, Testcontainers, Flyway, Spotless, `docker compose`, santé liveness/readiness,
   journaux JSON avec identifiant de corrélation). Plan séparé et préalable, décidé par le CTO (Q2) ;
   il n'existe pas encore et doit être produit avant la réalisation de celui-ci.

Ce plan ne fixe aucune version ni aucun outil : il s'appuie sur ce que l'initialisation aura figé.

## Modèle de domaine
- Bounded context : `tasks` (seul). Identité (Cognito) n'est traversée que par `OwnerId`.
- Agrégats et entités :
  - Agrégat racine `Task` (seul agrégat, pas d'entité interne ; ni sous-tâche, ni checklist, ni liste — Q1).
  - Objets-valeurs :
    - `TaskId` : UUID généré par le service.
    - `TaskTitle` : seul texte de la tâche, pas de description longue (Q4).
    - `TaskStatus` : `TODO`, `IN_PROGRESS`, `DONE`.
    - `DueDate` : date calendaire seule, sans heure ni fuseau (Q5). L'objet-valeur ne connaît pas
      l'horloge : la règle « pas dans le passé » est vérifiée par l'agrégat avec la date du jour fournie.
    - `OwnerId` : identifiant `sub` du jeton Cognito.
  - Attributs techniques : `createdAt`, `updatedAt` (fournis par une horloge injectée), `version`
    (verrouillage optimiste, non exposé comme concept métier).
- Invariants (protégés dans `Task`, jamais dans les contrôleurs) :
  - I1. `TaskTitle` obligatoire, espaces de début et fin retirés, non vide, 200 caractères au maximum (Q4).
  - I2. Une tâche est créée au statut `TODO`.
  - I3. Transitions de statut (Q3). Chaque transition est portée par une commande dédiée :

    | Commande | Depuis | Vers |
    |---|---|---|
    | Démarrer (`StartTask`) | `TODO` | `IN_PROGRESS` |
    | Mettre en pause (`PauseTask`) | `IN_PROGRESS` | `TODO` |
    | Terminer (`CompleteTask`) | `TODO` ou `IN_PROGRESS` | `DONE` |
    | Rouvrir (`ReopenTask`) | `DONE` | `TODO` |

    - Interdit : `DONE` → `IN_PROGRESS` (on rouvre, puis on redémarre).
    - Commande dont le statut cible est déjà le statut courant : **sans effet**, succès (200), aucun
      événement, aucune écriture (Q3). Exemples : `CompleteTask` sur une tâche `DONE`, `StartTask` sur
      une tâche `IN_PROGRESS`, `PauseTask` ou `ReopenTask` sur une tâche `TODO`.
    - Toute autre combinaison est refusée (exception de domaine). Chaque commande n'accepte que ses
      statuts de départ (Q10) : `PauseTask` sur une tâche `DONE` et `ReopenTask` sur une tâche
      `IN_PROGRESS` sont refusés (409).
  - I4. Une tâche `DONE` n'est plus modifiable (titre, échéance) tant qu'elle n'est pas rouverte.
    La suppression reste possible (I7).
  - I5. `OwnerId` est fixé à la création et immuable. Un seul propriétaire, sans partage ni
    assignation (Q8).
  - I6. `DueDate` est facultative ; elle peut être définie, modifiée ou retirée. Une échéance
    **antérieure à la date du jour est refusée**, à la création comme à la replanification (Q5).
    L'échéance du jour est acceptée. Une échéance déjà dépassée n'empêche aucune autre opération
    (renommer, changer de statut, supprimer). Pas de concept « en retard » dans ce plan (Q5).
    La date du jour de référence est la date UTC de l'horloge du service (Q11). La règle ne
    s'applique que si l'échéance change : une échéance passée renvoyée à l'identique est acceptée (Q12).
  - I7. Une tâche peut être supprimée quel que soit son statut, `DONE` compris. La suppression est
    définitive : ni archivage ni suppression logique (Q6).
  - Règle d'autorisation (niveau ressource, appliquée en couche application) : seul le propriétaire
    voit, modifie ou supprime sa tâche. Une tâche d'un autre propriétaire est traitée comme inexistante
    (404), pour ne pas révéler son existence.
- Commandes (cas d'usage) :
  - `CreateTask(ownerId, title, dueDate?)`
  - `RenameTask(taskId, title)`
  - `RescheduleTask(taskId, dueDate?)` (valeur absente = retrait de l'échéance)
  - `StartTask(taskId)`, `PauseTask(taskId)`, `CompleteTask(taskId)`, `ReopenTask(taskId)`
  - `DeleteTask(taskId)` (suppression définitive, tout statut)
  - Requêtes : `GetTask(taskId)`, `ListTasks(ownerId, status?, page, size)` triées par échéance
    croissante (sans échéance en dernier), puis date de création.
- Événements de domaine (enregistrés par l'agrégat, **non publiés hors du service** — Q7) :
  `TaskCreated`, `TaskRenamed`, `TaskRescheduled`, `TaskStarted`, `TaskPaused`, `TaskCompleted`,
  `TaskReopened`, `TaskDeleted`. Ils sont exposés par l'agrégat (liste d'événements en attente) et testés,
  sans broker, sans outbox et sans schéma dans `contracts/events/` tant qu'aucun consommateur n'est au
  catalogue. Une commande sans effet (I3) n'enregistre aucun événement.
- Termes ajoutés à `ubiquitous-language.md` (tâche R1) :

  | Terme métier | Identifiant de code | Définition |
  |---|---|---|
  | Créer une tâche | `CreateTask` | Enregistre une nouvelle tâche au statut `TODO` pour son propriétaire. |
  | Renommer | `RenameTask` | Change le titre d'une tâche non terminée. |
  | Replanifier | `RescheduleTask` | Définit, modifie ou retire l'échéance d'une tâche non terminée ; l'échéance ne peut pas être passée. |
  | Démarrer | `StartTask` | Passe une tâche de `TODO` à `IN_PROGRESS`. |
  | Mettre en pause | `PauseTask` | Repasse une tâche de `IN_PROGRESS` à `TODO`. |
  | Terminer | `CompleteTask` | Passe une tâche de `TODO` ou `IN_PROGRESS` à `DONE`. |
  | Rouvrir | `ReopenTask` | Repasse une tâche `DONE` à `TODO`. |
  | Supprimer | `DeleteTask` | Retire définitivement une tâche, quel que soit son statut. |

  Plus la section « Règles métier connues » (I1 à I7, règle d'autorisation, commandes sans effet) et la
  liste des événements de domaine internes, marqués « non publiés ».

## Contrats impactés
- OpenAPI : création de `contracts/openapi/task-api.yaml` (version `1.0.0`) dans `platform-architecture`.
  Consommateurs : aucun au catalogue aujourd'hui (futur `frontend`). Ressources :
  - `POST /tasks` → 201 + `Location` ; `GET /tasks?status=&page=&size=` → 200 page ;
    `GET /tasks/{taskId}` → 200.
  - `PATCH /tasks/{taskId}` (champs `title`, `dueDate` ; `dueDate: null` retire l'échéance) → 200.
  - Transitions explicites, reflet des commandes du domaine plutôt qu'un `PATCH` du statut :
    `POST /tasks/{taskId}/start`, `/pause`, `/complete`, `/reopen` → 200 avec la représentation de la
    tâche. Une commande sans effet (statut cible déjà atteint) renvoie aussi 200, tâche inchangée.
  - `DELETE /tasks/{taskId}` → 204, quel que soit le statut ; 404 si la tâche n'existe pas ou
    n'appartient pas à l'appelant.
  - Sécurité : schéma `bearerAuth` (JWT Cognito) sur toutes les opérations ; 401 sans jeton valide.
  - Erreurs : format unique `application/problem+json` (RFC 9457) avec un champ `code` stable :
    - `VALIDATION_ERROR` (400 : corps ou paramètres mal formés ; 422 : titre invalide) ;
    - `DUE_DATE_IN_PAST` (422) ;
    - `TASK_NOT_FOUND` (404) ;
    - `INVALID_STATUS_TRANSITION` (409, par exemple `/start` sur une tâche `DONE`) ;
    - `TASK_COMPLETED` (409, modification d'une tâche `DONE`) ;
    - `CONCURRENT_MODIFICATION` (409).
  - Représentation `Task` : `id`, `title`, `status`, `dueDate` (format `date`, facultatif), `createdAt`,
    `updatedAt`. Ni `ownerId` (implicite dans le jeton), ni description, ni indicateur « en retard ».
  - Lint Spectral (`.spectral.yaml`) vert.
- Événements : aucun schéma créé (Q7).
- Catalogue (`catalog/services.yaml`, entrée `task-service`) :
  - `exposes: [contracts/openapi/task-api.yaml]` (dans la même PR que le contrat, `bin/validate` exige
    l'existence du fichier) ;
  - `publishes` reste vide (Q7) ;
  - `contracts_version` : tag du référentiel posé par le CTO après fusion du contrat (ex. `v0.1.0`) ;
  - `status: planned` inchangé jusqu'à la mise en service (passage à `active` décidé par le CTO).

## Ordre de réalisation
Plan local, mais les tâches R1 à R3 se font dans `platform-architecture`, **dans une session ouverte sur ce
dépôt** (le `CLAUDE.md` de `task-service` interdit de modifier un autre dépôt depuis sa session).
1. Prérequis : ADR 0002 accepté, plan `initialisation-technique` réalisé.
2. `platform-architecture` : R1 → R2 → R3, puis tag des contrats par le CTO.
3. `task-service` : T1 → T8.

Compatibilité : premier contrat, aucun consommateur existant ; aucune contrainte de transition.

## Tâches

### platform-architecture — R1 : langage omniprésent
- Fichiers : `ubiquitous-language.md`.
- Tests attendus : `./bin/validate` vert.
- Critères d'acceptation :
  - [ ] Les termes du tableau ci-dessus (dont « Mettre en pause »), les règles I1 à I7, la règle
        d'autorisation, la règle des commandes sans effet et les huit événements internes sont ajoutés.
  - [ ] Aucun terme « description », « en retard », « todo » ni « archiver » n'est introduit.
  - [ ] La mention « Proposition initiale, à valider » reste tant que le CTO ne l'a pas retirée.

### platform-architecture — R2 : contrat `task-api.yaml`
- Fichiers : `contracts/openapi/task-api.yaml`.
- Tests attendus : Spectral vert ; `./bin/validate` vert.
- Critères d'acceptation :
  - [ ] Toutes les opérations listées dans « Contrats impactés » sont décrites (dont `/pause`), avec
        requêtes, réponses, codes d'erreur et exemples.
  - [ ] Contraintes de validation reportées dans le schéma (`title` : `minLength` 1, `maxLength` 200 ;
        `dueDate` : `format: date`, nullable en `PATCH` ; `size` : maximum 100, défaut 20).
  - [ ] La règle « échéance non passée » est documentée dans la description de `dueDate`, avec
        l'erreur `DUE_DATE_IN_PAST` (elle ne s'exprime pas en JSON Schema).
  - [ ] Les descriptions des transitions précisent le comportement sans effet (200, tâche inchangée).
  - [ ] `DELETE` documente la suppression définitive, quel que soit le statut.
  - [ ] Un unique schéma `Problem` est référencé par toutes les réponses d'erreur.

### platform-architecture — R3 : catalogue
- Fichiers : `catalog/services.yaml`.
- Tests attendus : `./bin/validate` vert.
- Critères d'acceptation :
  - [ ] `task-service.exposes` référence `contracts/openapi/task-api.yaml` ; `publishes` reste vide.
  - [ ] `contracts_version` mis à jour une fois le tag posé par le CTO.

### task-service — T1 : objets-valeurs du domaine
- Modules / fichiers : package `domain` : `TaskId`, `TaskTitle`, `TaskStatus`, `DueDate`, `OwnerId`,
  exception(s) de domaine.
- Tests attendus : tests unitaires sans framework.
- Critères d'acceptation :
  - [ ] Objets immuables, égalité par valeur.
  - [ ] `TaskTitle` refuse `null`, vide, blanc, plus de 200 caractères ; retire les espaces de bord.
  - [ ] `DueDate` encapsule une date seule (aucune heure ni fuseau) et expose une comparaison
        « est antérieure à » une date donnée, sans lire d'horloge.
  - [ ] `OwnerId` et `TaskId` refusent `null` et la valeur vide.
  - [ ] ArchUnit : aucune dépendance Spring / JPA / sérialisation dans `domain`.

### task-service — T2 : agrégat `Task`
- Modules / fichiers : `domain.Task`, événements de domaine (`TaskCreated`, `TaskRenamed`,
  `TaskRescheduled`, `TaskStarted`, `TaskPaused`, `TaskCompleted`, `TaskReopened`, `TaskDeleted`).
- Tests attendus : tests unitaires couvrant la table de transitions complète (4 commandes × 3 statuts :
  autorisée, sans effet ou refusée), I1 à I7, l'enregistrement des événements.
- Critères d'acceptation :
  - [ ] Création au statut `TODO` avec événement `TaskCreated`.
  - [ ] Les transitions du tableau I3 réussissent et enregistrent l'événement correspondant.
  - [ ] Une commande dont le statut cible est le statut courant ne modifie rien et n'enregistre aucun
        événement.
  - [ ] `StartTask` sur une tâche `DONE` lève l'exception de transition invalide, de même que toute
        combinaison hors tableau et hors cas sans effet (Q10).
  - [ ] Création ou replanification avec une échéance antérieure à la date du jour fournie → exception
        dédiée ; l'échéance du jour est acceptée ; une échéance passée inchangée est acceptée (Q12) ;
        le retrait de l'échéance est toujours accepté
        (hors tâche `DONE`).
  - [ ] Une tâche dont l'échéance est déjà dépassée peut être renommée, changer de statut et être
        supprimée.
  - [ ] Renommer ou replanifier une tâche `DONE` est refusé.
  - [ ] La suppression est acceptée pour les trois statuts et enregistre `TaskDeleted`.
  - [ ] Chaque commande effective enregistre exactement un événement ; un échec ou une commande sans
        effet n'en enregistre aucun.
  - [ ] Les dates (instant courant, date du jour) sont passées en paramètre (aucun `now()` implicite).

### task-service — T3 : cas d'usage et ports
- Modules / fichiers : `application` : un cas d'usage par commande (dont `PauseTask`) et par requête,
  port sortant `TaskRepository` (charger par id et propriétaire, enregistrer, supprimer, lister paginé),
  port `Clock` si non fourni par l'initialisation (fournit l'instant courant et la date du jour UTC, Q11).
- Tests attendus : tests unitaires avec un dépôt en mémoire (faux adaptateur de test) et une horloge fixe.
- Critères d'acceptation :
  - [ ] Une commande = une transaction = un agrégat.
  - [ ] Accès à la tâche d'un autre propriétaire → « tâche introuvable », y compris en suppression.
  - [ ] Une commande sans effet ne déclenche aucune sauvegarde et renvoie la tâche inchangée.
  - [ ] `DeleteTask` supprime physiquement la tâche, quel que soit son statut.
  - [ ] La date du jour transmise à l'agrégat provient du port `Clock`.
  - [ ] `ListTasks` filtre par propriétaire et statut facultatif, pagine et trie comme spécifié.
  - [ ] Les événements enregistrés sont vidés après sauvegarde (pas de publication externe).

### task-service — T4 : persistance PostgreSQL
- Modules / fichiers : migration Flyway `V1__create_task_table.sql` (schéma `tasks`, table `task`,
  colonne `due_date` de type `date` nullable, contrainte `CHECK` sur le statut, index
  `(owner_id, status, due_date)`, pas de colonne de suppression logique ni de description),
  adaptateur `adapter.out` implémentant `TaskRepository`, verrouillage optimiste par colonne `version`.
- Tests attendus : tests d'intégration Testcontainers (PostgreSQL réel).
- Critères d'acceptation :
  - [ ] Aller-retour complet d'une tâche (avec et sans échéance), la date d'échéance restant identique
        quel que soit le fuseau de la JVM et de la base.
  - [ ] Isolation par propriétaire vérifiée au niveau des requêtes.
  - [ ] La suppression retire la ligne (`DELETE` physique).
  - [ ] Deux mises à jour concurrentes → la seconde échoue en conflit (mappé plus tard en 409).
  - [ ] Pagination et tri vérifiés, y compris les tâches sans échéance en dernier.
  - [ ] Aucun DDL généré par l'ORM.

### task-service — T5 : authentification et `OwnerId`
- Modules / fichiers : `config` (validation des JWT : émetteur, audience et clé publique ou JWKS par
  variables d'environnement), extraction de `OwnerId` depuis la revendication `sub` en `adapter.in`.
- Tests attendus : tests d'intégration avec des jetons signés par une clé de test locale, sans appel ni
  émulation de Cognito (Q9).
- Critères d'acceptation :
  - [ ] Requête sans jeton, jeton expiré ou mal signé → 401 au format `problem+json`.
  - [ ] Points de santé accessibles sans jeton.
  - [ ] Aucune URL, identifiant Cognito ni clé en dur ; la clé de test ne sert qu'aux tests et au local.

### task-service — T6 : adaptateur REST
- Modules / fichiers : `adapter.in` : contrôleur(s), DTO conformes à `task-api.yaml`, validation des
  entrées, gestionnaire d'erreurs unique vers `problem+json`.
- Tests attendus : tests d'intégration HTTP (Testcontainers) pour chaque opération, cas nominaux, sans
  effet et d'erreur (400, 401, 404, 409, 422).
- Critères d'acceptation :
  - [ ] Chaque opération du contrat est implémentée (dont `/pause`), aucune opération hors contrat.
  - [ ] Commande de transition sans effet → 200 avec la tâche inchangée (`updatedAt` et `version`
        inchangés).
  - [ ] Transition interdite (ex. `/start` sur une tâche `DONE`) → 409 `INVALID_STATUS_TRANSITION` ;
        modification d'une tâche `DONE` → 409 `TASK_COMPLETED` ; conflit de version → 409
        `CONCURRENT_MODIFICATION`.
  - [ ] Échéance passée en `POST` ou `PATCH` → 422 `DUE_DATE_IN_PAST`.
  - [ ] `PATCH` renvoyant à l'identique une échéance déjà dépassée (par exemple pour renommer) → 200 (Q12).
  - [ ] `DELETE` d'une tâche `DONE` → 204.
  - [ ] Les journaux ne contiennent ni jeton ni titre de tâche (donnée potentiellement personnelle).

### task-service — T7 : tests de contrat
- Modules / fichiers : tests validant requêtes et réponses réelles contre `task-api.yaml` à la version
  déclarée dans `contracts_version` (outil fixé dans le `pom` lors de l'implémentation, version épinglée).
- Tests attendus : suite de contrat exécutée par `mvn verify`.
- Critères d'acceptation :
  - [ ] Toutes les réponses des tests T6 sont validées contre le contrat, y compris les réponses
        sans effet et `DUE_DATE_IN_PAST`.
  - [ ] Une divergence volontaire du contrat fait échouer la suite.

### task-service — T8 : documentation et démarrage local
- Modules / fichiers : `README.md` (exemples d'appels, génération d'un jeton de test signé par la clé
  locale), éventuel complément `docker compose` (configuration de la clé publique locale).
- Tests attendus : démarrage local vérifié manuellement ; `mvn verify` vert.
- Critères d'acceptation :
  - [ ] `docker compose up --build` démarre le service et la base ; migration appliquée.
  - [ ] Le README explique comment obtenir un jeton local, sans Cognito.
  - [ ] Le README décrit un scénario complet : créer, lister, démarrer, mettre en pause, redémarrer,
        terminer, rouvrir, supprimer.
  - [ ] Définition de « terminé » des standards respectée.

## Questions ouvertes
1. ~~**Sens de « todo »**~~ — **Tranché (CTO, 2026-09-30)** : « todo » désigne l'agrégat `Task`.
   Sous-tâches, checklist et listes de tâches sont hors périmètre.
2. ~~**Initialisation technique**~~ — **Tranché (CTO, 2026-09-30)** : plan séparé, préalable à celui-ci. Reste à faire hors de ce
   plan : trancher les points 1 à 3 de l'ADR 0002 (le runtime conditionne T4 et T8).
3. ~~**Transitions de statut**~~ — **Tranché (CTO, 2026-09-30)** : autorisées `TODO` → `IN_PROGRESS`,
   `IN_PROGRESS` → `DONE`, `TODO` → `DONE`, `DONE` → `TODO` (rouvrir) et `IN_PROGRESS` → `TODO` (mettre en
   pause). Interdit : `DONE` → `IN_PROGRESS`. Transition vers le statut courant : sans effet, 200, aucun
   événement.
4. ~~**Titre**~~ — **Tranché (CTO, 2026-09-30)** : obligatoire, 200 caractères maximum, pas de description longue.
5. ~~**Échéance**~~ — **Tranché (CTO, 2026-09-30)** : date seule, facultative, refusée si passée à la
   création comme à la replanification. Pas de concept « en retard » dans ce plan.
6. ~~**Suppression**~~ — **Tranché (CTO, 2026-09-30)** : définitive, sans archivage ni suppression logique,
   autorisée quel que soit le statut, `DONE` compris.
7. ~~**Événements**~~ — **Tranché (CTO, 2026-09-30)** : aucun événement publié hors du service tant qu'aucun consommateur
   n'est au catalogue. Pas de broker ni d'outbox dans ce plan.
8. ~~**Partage**~~ — **Tranché (CTO, 2026-09-30)** : un seul propriétaire, sans partage ni assignation.
9. ~~**Authentification en local**~~ — **Tranché (CTO, 2026-09-30)** : jetons de test signés par une clé locale, pas d'émulation
   de Cognito.
10. ~~**Transitions par commande**~~ — **Tranché (CTO, 2026-09-30)** : chaque commande n'accepte que ses statuts de
    départ : `PauseTask` sur une tâche `DONE` et `ReopenTask` sur une tâche `IN_PROGRESS` → 409.
11. ~~**Date du jour de référence**~~ — **Tranché (CTO, 2026-09-30)** : date UTC de l'horloge du service.
12. ~~**`PATCH` avec une échéance passée inchangée**~~ — **Tranché (CTO, 2026-09-30)** : la règle « pas dans le passé » ne
    s'applique que si l'échéance change.

## Risques et compromis
- **Blocage par les prérequis** : tant que l'ADR 0002 et l'initialisation ne sont pas faits, ce plan ne
  peut pas démarrer. Choix du CTO : socle technique et fonctionnel relus séparément.
- **Événements internes non publiés** (décidé, Q7) : on garde la modélisation des faits métier sans coût
  d'infrastructure ; il faudra ajouter un outbox et des schémas d'événements quand un consommateur
  apparaîtra (migration additive, sans rupture).
- **API par transitions explicites** (`/start`, `/pause`, `/complete`, `/reopen`) plutôt qu'un
  `PATCH status` : plus fidèle au domaine et plus simple à autoriser, mais quatre opérations dans le
  contrat.
- **Commandes sans effet en 200** (décidé, Q3) : les rejeux et doubles clics sont sans danger, mais une
  erreur du client (par exemple une tâche déjà terminée par un autre onglet) passe inaperçue ; le client
  doit lire le statut renvoyé.
- **Échéance non passée et fuseau** (décidé, Q11) : la « date du jour » est la date UTC du service ;
  entre minuit local et minuit UTC, un utilisateur peut voir refuser l'échéance du jour (à l'est de UTC)
  ou saisir celle de la veille (à l'ouest).
- **Échéance dépassée non signalée** (décidé, Q5) : pas de concept « en retard » ; le client peut le
  calculer lui-même, au risque de divergences entre clients.
- **404 plutôt que 403** pour la tâche d'un autre propriétaire : ne révèle pas l'existence de la
  ressource, au prix d'un diagnostic un peu moins direct.
- **Suppression définitive** (décidé, Q6) : simple, mais sans historique ni annulation, y compris pour
  les tâches terminées ; un besoin d'audit ultérieur exigerait un changement de modèle.
- **Contrat avant consommateur** : `task-api.yaml` est conçu sans frontend existant ; des ajustements
  rétrocompatibles sont probables à l'arrivée du frontend.
- **Tri par échéance puis création** : suppose un index adapté (T4) ; à réévaluer si les volumes par
  propriétaire deviennent importants (pagination par curseur).
- **Tâches R1 à R3 dans un autre dépôt** : elles demandent une session séparée sur `platform-architecture`,
  ce qui ajoute une étape de coordination pour le CTO.
