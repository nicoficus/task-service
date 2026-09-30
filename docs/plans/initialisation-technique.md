---
statut: validé   # à valider | validé | réalisé  (seul le CTO passe à « validé »)
slug: initialisation-technique
portee: local       # local (un seul service) | transverse (plusieurs services)
services: [task-service]
---

# Initialisation technique de `task-service`

## Objectif
Poser le socle technique de `task-service`, sans aucune fonctionnalité métier, pour que le plan validé
`gestion-taches` puisse démarrer : squelette Maven, packages hexagonaux vérifiés par ArchUnit, PostgreSQL et
Flyway testés avec Testcontainers, formatage Spotless, démarrage local par `docker compose`, santé liveness /
readiness, journaux JSON avec identifiant de corrélation, horloge UTC injectable, et point d'entrée Lambda
conforme au runtime retenu par l'ADR 0002 (Spring Boot sur Lambda, Java + SnapStart).

Hors périmètre : agrégat `Task`, table métier, contrat `task-api.yaml`, sécurité JWT (plan `gestion-taches`,
T5), format d'erreur `problem+json` (T6), outil de tests de contrat (T7), infrastructure Terraform.

## Modèle de domaine
- Bounded context : `tasks`. Ce plan ne touche pas au modèle.
- Agrégats et entités : aucun (les packages `domain` et `application` restent vides).
- Invariants : aucun.
- Commandes : aucune.
- Événements de domaine : aucun.
- Termes ajoutés à `ubiquitous-language.md` : aucun (identifiant de corrélation, santé, horloge sont des
  notions techniques, pas du langage métier).

## Contrats impactés
- OpenAPI : aucun. Les points de santé sont techniques et ne figurent pas dans `task-api.yaml` (voir Q10).
- Événements : aucun.
- Catalogue : aucune modification (`exposes`, `publishes` restent vides, `status: planned`,
  `contracts_version: unreleased`).
- Référentiel (tâches R1 et R2, session séparée sur `platform-architecture`) :
  - nouvel ADR `adr/0004-socle-technique-services-java-lambda.md` (statut `proposé`) qui fige ce que
    l'ADR 0002 laisse ouvert ;
  - `standards/engineering-standards.md`, section 8 : nom de l'en-tête de corrélation et champs de journal
    communs à tous les services.

## Conséquences du runtime Lambda (analyse)
L'ADR 0002 retient Spring Boot sur Lambda avec SnapStart et « exécution locale uniquement ». Conséquences
pour l'initialisation, chacune reliée à une question ouverte :
1. **Adaptateur Lambda / HTTP** : un gestionnaire (handler) doit traduire les événements API Gateway en
   requêtes HTTP pour Spring MVC. Retenu : adaptateur AWS de Spring Cloud Function (Q2) ; sa compatibilité
   avec Spring Boot 4.1.1 et Java 25 est **à vérifier** en R1. Le format de l'événement dépend du type d'API
   Gateway : HTTP API, charge utile v2 (Q2).
2. **Conditionnement** : SnapStart ne s'applique pas aux fonctions déployées en image de conteneur ;
   l'artefact Lambda doit donc être une archive zip (ou un jar avec dépendances). Avec LocalStack (Q3), ce
   même zip est déployé en local : pas d'image Docker propre au service.
3. **SnapStart et JDBC** : tout ce qui est ouvert pendant la phase d'initialisation est figé dans
   l'instantané. Des connexions PostgreSQL ouvertes avant l'instantané (pool Hikari, migration Flyway au
   démarrage) seraient invalides après restauration. Il faut soit n'ouvrir aucune connexion avant
   l'instantané, soit les fermer par un crochet de point de contrôle (CRaC, `org.crac`, que Spring prend en
   charge pour le pool Hikari). L'ADR 0002 mentionne RDS Proxy côté AWS (Q4).
4. **Migrations Flyway** : lancées au démarrage, elles ouvrent des connexions pendant l'initialisation et
   s'exécutent à chaque démarrage à froid, potentiellement en parallèle. Stratégie à décider (Q4).
5. **Santé liveness / readiness** : sur Lambda, aucun orchestrateur n'interroge les sondes ; elles restent
   utiles en local et pour un contrôle de santé externe, mais leur sémantique change (Q10).
6. **Aléa et unicité après restauration** : les identifiants générés (futur `TaskId`, identifiant de
   corrélation) doivent rester uniques entre environnements restaurés depuis le même instantané. À garder en
   tête pour `gestion-taches` ; ce plan génère des identifiants de corrélation par `UUID.randomUUID()`
   (`SecureRandom`), à confirmer dans l'ADR 0004.
7. **Exécution locale** : LocalStack (Q3) émule Lambda et API Gateway, mais pas SnapStart ni RDS Proxy.
   Le `docker compose` démarre PostgreSQL et LocalStack, qui exécute le zip Lambda (Q16, Q17).
8. **IaC Terraform** (ADR 0002, point 2) : la fonction Lambda, SnapStart, API Gateway, RDS Proxy relèvent de
   Terraform et sont **hors de ce plan**. Ce plan ne fixe que l'interface avec l'IaC : nom de la classe du
   handler, format et emplacement de l'artefact, variables d'environnement attendues. L'emplacement du code
   Terraform reste à décider (Q6).

## Proposition pour l'ADR 0004 (à rédiger en R1, statut `proposé`)
Contenu attendu, chaque valeur **à confirmer par le CTO** (aucune version n'est inventée ici) :
- Versions figées : Java 25 (confirmé par l'ADR 0002), Spring Boot `4.1.1` (Q1, tranché), Maven et Maven
  Wrapper, PostgreSQL majeur (Q7), bibliothèque d'adaptation Lambda et, le cas échéant, Spring Cloud (Q2).
- Vérifications à consigner : disponibilité du runtime géré Lambda Java 25 et de SnapStart pour ce runtime ;
  compatibilité de l'adaptateur Lambda avec la version de Spring Boot retenue.
- Type d'API Gateway et format d'événement (Q2).
- Conditionnement : une seule archive zip, pour AWS comme pour LocalStack.
- Gestion des connexions JDBC autour de l'instantané SnapStart et stratégie de migration (Q4, Q17).
- Mode d'exécution locale : LocalStack, édition et version épinglée (Q3, Q16) ; aucun déploiement AWS
  pendant le POC.
- Format des journaux JSON et en-tête de corrélation (Q9).
- Technologie de persistance (Q8), même si elle n'est utilisée qu'à partir de `gestion-taches` T4.

## Ordre de réalisation
Plan local ; R1 et R2 se font dans `platform-architecture`, **dans une session ouverte sur ce dépôt**
(le `CLAUDE.md` de `task-service` interdit de modifier un autre dépôt depuis sa session).
1. `platform-architecture` : R1 (ADR 0004, puis acceptation par le CTO) → R2.
2. `task-service` : T1 → T2 → T3 → T4 → T5 → T6 → T7 → T8 → T9 → T10 → T11.
   T8 (Lambda) peut être réalisé après T9 si le CTO veut d'abord un démarrage local ; il reste requis avant de
   passer ce plan à `réalisé`.

Compatibilité : aucun contrat ni consommateur ; aucune contrainte de transition.

## Tâches

### platform-architecture — R1 : ADR 0004, socle technique des services Java sur Lambda
- Fichiers : `adr/0004-socle-technique-services-java-lambda.md` (gabarit `templates/adr.md`).
- Tests attendus : `./bin/validate` vert.
- Critères d'acceptation :
  - [ ] L'ADR reprend les rubriques de la section « Proposition pour l'ADR 0004 » avec les réponses du CTO
        aux questions Q1 à Q4, Q7 à Q9, Q16 et Q17.
  - [ ] Chaque version est exacte (aucune plage du type « 4.x ») et accompagnée de la date de vérification.
  - [ ] Les alternatives écartées sont listées (adaptateur Lambda, mode local, stratégie de migration).
  - [ ] Statut `proposé` à la rédaction ; **T1 ne commence qu'après acceptation par le CTO**.

### platform-architecture — R2 : standard d'observabilité
- Fichiers : `standards/engineering-standards.md` (section 8).
- Tests attendus : `./bin/validate` vert.
- Critères d'acceptation :
  - [ ] Le nom de l'en-tête de corrélation (proposition : `X-Correlation-Id`) et les règles de propagation
        (reprise si présent et valide, génération sinon, renvoi dans la réponse) sont décrits.
  - [ ] Les champs communs des journaux JSON sont listés (horodatage, niveau, message, logger, nom du
        service, identifiant de corrélation, et sur Lambda l'identifiant de requête AWS).
  - [ ] Renvoi vers l'ADR 0004.

### task-service — T1 : squelette Maven et packages hexagonaux
- Modules / fichiers : `pom.xml` (module unique), Maven Wrapper, `.editorconfig`, compléments de
  `.gitignore`, classe principale Spring Boot à la racine du package de base (Q5), packages vides
  `domain`, `application`, `adapter.in`, `adapter.out`, `config` (chacun avec `package-info.java` décrivant
  son rôle).
- Tests attendus : test de démarrage du contexte Spring (sans base, voir T4 pour la base) ; `mvn verify` vert.
- Critères d'acceptation :
  - [ ] Versions de Java, Spring Boot et des plugins conformes à l'ADR 0004, toutes épinglées (aucune
        version `SNAPSHOT`, `LATEST` ou plage).
  - [ ] Règles `maven-enforcer-plugin` : version de Java et de Maven minimales, interdiction des dépendances
        `SNAPSHOT`.
  - [ ] Dépendances gérées par la BOM Spring Boot ; versions hors BOM déclarées en propriétés.
  - [ ] Aucune classe métier, aucune dépendance AWS SDK.
  - [ ] Groupe Maven, artefact et package de base conformes à la réponse à Q5.

### task-service — T2 : formatage (Spotless)
- Modules / fichiers : `pom.xml` (plugin Spotless, formateur selon Q14), fichiers existants reformatés.
- Tests attendus : `mvn verify` échoue sur un fichier mal formaté et passe après `mvn spotless:apply`.
- Critères d'acceptation :
  - [ ] `spotless:check` lié à la phase `verify`.
  - [ ] Java, `pom.xml` et fichiers texte courants (YAML, Markdown si l'outil le permet) couverts.
  - [ ] Version du formateur épinglée.

### task-service — T3 : tests d'architecture (ArchUnit)
- Modules / fichiers : `src/test/java/.../architecture/HexagonalArchitectureTest.java` (nom indicatif),
  dépendance ArchUnit épinglée.
- Tests attendus : tests ArchUnit exécutés par `mvn verify`, plus des classes de test volontairement
  fautives prouvant que chaque règle détecte une violation.
- Critères d'acceptation :
  - [ ] `domain` ne dépend que du JDK : aucune dépendance à Spring, Jakarta Persistence, Hibernate, AWS SDK,
        Jackson ni à une autre bibliothèque de sérialisation.
  - [ ] Sens des dépendances : `adapter.*` → `application` → `domain` ; jamais `domain` → `application` ou
        `adapter`, jamais `application` → `adapter`.
  - [ ] `adapter.in` et `adapter.out` ne dépendent pas l'un de l'autre.
  - [ ] Absence de cycles entre packages.
  - [ ] Traitement des règles sur packages encore vides conforme à la réponse à Q15, documenté dans le test,
        avec une note indiquant que `gestion-taches` T1 le lève.

### task-service — T4 : PostgreSQL, Flyway et Testcontainers
- Modules / fichiers : `pom.xml` (pilote PostgreSQL, Flyway et son module PostgreSQL, Testcontainers),
  configuration de la source de données par variables d'environnement, configuration Flyway (schéma
  `tasks`, emplacement `db/migration`, **aucune migration**), classe de base des tests d'intégration.
- Tests attendus : test d'intégration Testcontainers (PostgreSQL réel, image épinglée à la version majeure de
  Q7) vérifiant que le contexte démarre et que Flyway initialise le schéma.
- Critères d'acceptation :
  - [ ] URL, utilisateur et mot de passe de la base lus exclusivement dans des variables d'environnement ;
        aucune valeur par défaut pointant vers un environnement réel.
  - [ ] Après démarrage, le schéma `tasks` existe et contient la table d'historique Flyway ; aucune autre
        table n'est créée.
  - [ ] Le nom `V1__...` reste libre pour `gestion-taches` T4.
  - [ ] Aucune génération de DDL par un ORM (si un ORM est ajouté selon Q8 : génération désactivée).
  - [ ] Le comportement de Flyway suit Q4 : activé au démarrage dans les tests ; désactivé au démarrage en
        profil Lambda (LocalStack compris), la migration passant par la fonction dédiée de T8 (Q17).
  - [ ] Aucune base embarquée (H2 ou autre) dans les dépendances.

### task-service — T5 : santé liveness / readiness
- Modules / fichiers : Spring Boot Actuator, configuration dans `config` / `application.yaml`.
- Tests attendus : test d'intégration HTTP (Testcontainers) sur les deux points de santé.
- Critères d'acceptation :
  - [ ] Groupes de santé liveness et readiness activés explicitement (ils ne le sont pas par défaut hors
        Kubernetes), chemins documentés (proposition : `/actuator/health/liveness` et
        `/actuator/health/readiness`, voir Q10).
  - [ ] Readiness inclut la base ; liveness ne dépend pas de la base.
  - [ ] Base arrêtée (conteneur stoppé dans le test) → readiness `DOWN` (503), liveness `UP` (200).
  - [ ] Seul l'endpoint `health` est exposé ; aucun détail de composant exposé sans autorisation.

### task-service — T6 : journaux JSON et identifiant de corrélation
- Modules / fichiers : configuration de la journalisation structurée (format selon Q9), filtre HTTP de
  corrélation dans `adapter.in` (alimente le contexte de journalisation, MDC).
- Tests attendus : tests d'intégration capturant la sortie des journaux et les en-têtes de réponse.
- Critères d'acceptation :
  - [ ] Chaque ligne de journal est un objet JSON valide sur la sortie standard, avec les champs définis
        en R2.
  - [ ] En-tête de corrélation présent et valide → réutilisé, présent dans chaque ligne de journal de la
        requête et renvoyé dans la réponse.
  - [ ] En-tête absent → identifiant généré, journalisé et renvoyé.
  - [ ] En-tête invalide (trop long ou caractères hors liste autorisée) → remplacé par un identifiant
        généré (protection contre l'injection dans les journaux).
  - [ ] Contexte de journalisation nettoyé en fin de requête (aucune fuite entre requêtes, vérifié par test).
  - [ ] Aucun en-tête `Authorization`, aucun corps de requête ni de réponse journalisé.

### task-service — T7 : horloge UTC injectable
- Modules / fichiers : bean `java.time.Clock` (UTC) dans `config`.
- Tests attendus : test du contexte vérifiant la présence du bean et son fuseau UTC ; exemple de
  remplacement par une horloge fixe dans un test.
- Critères d'acceptation :
  - [ ] Une seule horloge applicative, en UTC, indépendante du fuseau de la JVM.
  - [ ] Aucun port applicatif créé ici : le port `Clock` de `application` (instant courant et date du jour
        UTC) reste à la charge de `gestion-taches` T3, qui l'implémente en s'appuyant sur ce bean.

### task-service — T8 : point d'entrée Lambda et préparation SnapStart
- Modules / fichiers : handler Lambda de l'adaptateur AWS Spring Cloud Function (Q2) dans `config` ou
  `adapter.in`, fonction de migration `migrate` (Q4, Q17), profil de configuration Lambda, construction de
  l'artefact zip (plugin Maven), prise en charge du point de contrôle pour le pool JDBC (`org.crac` et/ou
  réglages du pool, Q4), repli de l'identifiant de corrélation sur l'identifiant de requête API Gateway.
- Tests attendus : tests d'intégration (Testcontainers) appelant directement le handler avec des événements
  API Gateway enregistrés en fichiers (format de Q2), sans AWS ni émulateur.
- Critères d'acceptation :
  - [ ] Un événement `GET` vers le point de liveness renvoie une réponse API Gateway 200 au format attendu.
  - [ ] L'identifiant de corrélation de l'événement est repris dans la réponse et les journaux ; à défaut,
        l'identifiant de requête API Gateway est utilisé.
  - [ ] Aucune connexion JDBC n'est ouverte à la fin de l'initialisation en profil Lambda, ou elles sont
        fermées par le crochet de point de contrôle (vérifié par test sur le pool : zéro connexion active
        ou inactive).
  - [ ] `mvn verify` (ou `mvn package`) produit l'artefact zip Lambda, à un emplacement et sous un nom
        stables, documentés pour l'IaC, avec le nom complet de la classe du handler.
  - [ ] Le serveur HTTP embarqué n'est pas démarré en mode Lambda.
  - [ ] L'invocation de la fonction `migrate` applique les migrations Flyway (vérifié par Testcontainers)
        et est idempotente (seconde invocation sans effet).
  - [ ] Aucun identifiant AWS, ARN ou URL d'environnement dans le dépôt.

### task-service — T9 : `docker compose` avec LocalStack
- Modules / fichiers : `compose.yaml` (services `postgres` et `localstack`, images épinglées, contrôles de
  santé, réseau commun permettant à la Lambda d'atteindre PostgreSQL), script d'initialisation LocalStack
  (`ready.d`) qui crée la fonction à partir du zip de T8, invoque `migrate` puis crée l'API Gateway (Q16,
  Q17). Pas de `Dockerfile` pour le service (Q3).
- Tests attendus : démarrage local vérifié manuellement ; `mvn verify` vert.
- Critères d'acceptation :
  - [ ] Après `mvn package`, `docker compose up` démarre la base et LocalStack, déploie la fonction dans
        LocalStack et applique les migrations (schéma `tasks` et historique Flyway présents).
  - [ ] Liveness et readiness répondent `UP` via l'URL de l'API Gateway de LocalStack (chemins selon Q10).
  - [ ] Journaux de la Lambda (visibles via LocalStack) au format JSON avec identifiant de corrélation.
  - [ ] Le script n'appelle que LocalStack : aucun point de terminaison, identifiant ni profil AWS réel.
  - [ ] Aucun fichier `.env` ; identifiants de la base locale selon Q12 ; jeton LocalStack lu dans l'environnement du poste (Q16).
  - [ ] Image PostgreSQL épinglée à la même version que les tests Testcontainers.
  - [ ] `docker compose down -v` remet l'environnement à zéro.

### task-service — T10 : intégration continue (conditionnée à Q11)
- Modules / fichiers : `.github/workflows/ci.yml`.
- Tests attendus : exécution du workflow sur la PR (vérifiée par le CTO, l'agent ne pousse pas).
- Critères d'acceptation :
  - [ ] `mvn verify` (tests unitaires, d'intégration avec Testcontainers, ArchUnit, Spotless) sur chaque PR.
  - [ ] Analyse des vulnérabilités des dépendances avec l'outil retenu en Q11, sans secret dans le dépôt.
  - [ ] Actions GitHub épinglées par version ou empreinte.

### task-service — T11 : documentation
- Modules / fichiers : `README.md`, section « Stack et commandes » du `CLAUDE.md`.
- Tests attendus : relecture ; commandes du README exécutées manuellement.
- Critères d'acceptation :
  - [ ] Le README donne les commandes exactes : construire, tester, formater, démarrer, arrêter, produire
        l'artefact Lambda.
  - [ ] Variables d'environnement documentées (nom, rôle, obligatoire ou non), sans valeur réelle.
  - [ ] Structure hexagonale et règles ArchUnit résumées, avec renvoi vers les standards.
  - [ ] Interface avec l'IaC décrite : artefact, handler, variables, stratégie de migration.
  - [ ] Le `CLAUDE.md` ne porte plus la mention « à confirmer après l'initialisation technique » et indique
        les versions figées par l'ADR 0004.
  - [ ] Définition de « terminé » des standards respectée.

## Questions ouvertes
1. ~~**Version exacte de Spring Boot**~~ — **Tranché (CTO, 2026-09-30)** : Spring Boot **4.1.1**. Reste à
   vérifier en R1 : runtime géré Lambda Java 25 et SnapStart disponibles pour ce runtime.
2. ~~**Adaptateur Lambda / HTTP**~~ — **Tranché (CTO, 2026-09-30)** : adaptateur AWS de **Spring Cloud
   Function** (version du train Spring Cloud compatible avec Spring Boot 4.1.1, à figer et vérifier en R1).
   Type d'API Gateway : **HTTP API, charge utile v2** (tranché, CTO, 2026-09-30) ; il fixe le format des
   événements de test de T8.
3. ~~**Mode d'exécution locale**~~ — **Tranché (CTO, 2026-09-30)** : **LocalStack** (Lambda et API Gateway
   émulés), **aucun déploiement AWS** pendant le POC. SnapStart n'est donc jamais exercé réellement.
4. ~~**Migrations et connexions sous SnapStart**~~ — **Tranché (CTO, 2026-09-30)** : proposition retenue.
   Flyway au démarrage dans les tests (Testcontainers) ; en mode Lambda (LocalStack compris), Flyway désactivé
   au démarrage et exécuté par un mécanisme dédié (voir Q17), pool sans connexion ouverte avant l'instantané
   et crochet de point de contrôle.
5. ~~**Coordonnées Maven et package de base**~~ — **Tranché (CTO, 2026-09-30)** : `groupId` `net.versmerch`,
   `artifactId` `task-service`, package de base `net.versmerch.task`.
6. **[Non bloquante] Emplacement de l'IaC Terraform.** Dépôt `infra` (envisagé au catalogue) ou dossier
   `infra/` dans `task-service`, et plan qui le portera. Ce plan se limite à l'interface (artefact, handler,
   variables).
7. ~~**Version majeure de PostgreSQL**~~ — **Tranché (CTO, 2026-09-30)** : **PostgreSQL 18**, identique en
   local, en test et sur la cible RDS (ADR 0004).
8. **[Non bloquante pour ce plan, à trancher avant `gestion-taches` T4] Technologie de persistance** :
   Spring Data JDBC (plus léger au démarrage à froid, proposée) ou JPA / Hibernate. Ce plan n'ajoute que la
   source de données et Flyway.
9. **[Non bloquante] Journaux et corrélation.** Format JSON : ECS ou Logstash (proposition : ECS, pris en
   charge nativement par Spring Boot). En-tête : `X-Correlation-Id` (proposé). Faut-il aussi activer le
   format de journal JSON de Lambda, ou la sortie JSON de l'application suffit-elle ?
10. **[Non bloquante] Points de santé sur Lambda.** Chemins proposés : `/actuator/health/liveness` et
    `/actuator/health/readiness`. Doivent-ils être routés par API Gateway (publiquement, sans jeton), ou
    rester accessibles uniquement en local ? Ils ne figurent pas dans `task-api.yaml`.
11. **[Non bloquante] Intégration continue.** Inclure T10 dans ce plan ? Outil d'analyse des vulnérabilités :
    OWASP Dependency-Check (clé API NVD nécessaire, donc secret de CI), OSV-Scanner, Trivy ou Dependabot /
    dependency review de GitHub.
12. **[Non bloquante] Identifiants de la base locale.** Valeurs de développement non sensibles écrites
    directement dans `compose.yaml` (sans `.env`), à confirmer au regard de la règle « aucun secret dans le
    dépôt ».
13. **[Non bloquante] Cohérence de l'ADR 0002.** Statut « accepté », mais sections intitulées « Décision
    proposée » et « Points ouverts », point 4 (périmètre du POC) non tranché et Spring Boot non figé.
    Proposition : l'ADR 0004 fige les versions ; le CTO met à jour la rédaction de l'ADR 0002.
14. **[Non bloquante] Formateur Spotless** : google-java-format ou palantir-java-format.
15. **[Non bloquante] Règles ArchUnit sur packages vides.** `domain` et `application` sont vides jusqu'à
    `gestion-taches` T1 : les règles échouent par défaut sur un ensemble vide. Proposition : autoriser
    explicitement l'ensemble vide (`allowEmptyShould`) règle par règle, commenté, et retirer cette tolérance
    dans `gestion-taches` T1 ; l'efficacité des règles est prouvée par les classes fautives de test. Le
    standard interdisant d'assouplir un test, la décision revient au CTO.
16. ~~**Édition de LocalStack**~~ — **Tranché (CTO, 2026-09-30)** : **édition payante** acceptée (HTTP API v2
    émulée). À vérifier et consigner en R1 : version de l'image épinglée, prise en charge du runtime Lambda
    Java 25 et du réseau entre la Lambda et PostgreSQL. Le jeton d'authentification LocalStack est un
    secret : il est lu dans une variable d'environnement du poste (ex. `LOCALSTACK_AUTH_TOKEN`), transmise
    par `compose.yaml` sans valeur, documentée dans le README ; jamais de `.env` ni de valeur dans le dépôt.
17. ~~**Provisionnement local et migrations**~~ — **Tranché (CTO, 2026-09-30)** : proposition retenue ; le
    script LocalStack (et `awslocal` ciblant uniquement LocalStack) est autorisé, il ne relève pas de
    l'interdit « jamais de commande AWS ». Proposition retenue :
    - la fonction Lambda et l'API Gateway sont créées dans LocalStack par un script d'initialisation
      (`ready.d`) versionné dans `task-service`, qui n'appelle que LocalStack, jamais AWS ; l'IaC Terraform
      reste hors plan (Q6) ;
    - le « mécanisme dédié » de migration de Q4 est une seconde fonction du même artefact zip (fonction
      Spring Cloud Function `migrate`), invoquée par ce script avant d'exposer l'API, et réutilisable telle
      quelle par l'IaC sur AWS.

## Risques et compromis
- **Versions non vérifiables par l'architecte** : l'adaptateur Lambda (compatibilité avec Spring Boot 4.1.1) et le runtime Java 25
  sur Lambda doivent être vérifiés à la date de R1 ; un adaptateur non compatible pourrait imposer un autre
  outil, voire une révision de l'ADR 0002.
- **Écart local / cible** (LocalStack, Q3) : Lambda et API Gateway sont émulés avec le même zip qu'en
  cible, mais SnapStart et RDS Proxy ne le sont pas, et aucun déploiement AWS n'est prévu : la restauration
  d'instantané n'est couverte que par le test du pool autour du point de contrôle (T8).
- **Coût de LocalStack** : démarrage et boucle de développement plus lents qu'un serveur embarqué (chaque
  modification demande `mvn package` et un redéploiement), dépendance à l'édition de LocalStack et à son
  jeton (Q16). En contrepartie, un seul artefact, sans divergence de configuration.
- **Migrations séparées sur Lambda** (Q4, Q17) : plus sûr, mais ajoute une fonction `migrate` à invoquer
  par le script LocalStack puis par l'IaC ; l'alternative (au démarrage) était plus simple mais exposait à
  des migrations concurrentes.
- **Tolérance ArchUnit temporaire** (Q15) : écart ponctuel au standard « aucun test assoupli », borné dans le
  temps et tracé ; sans elle, il faudrait des classes factices dans `domain`, contraire au « sans métier ».
- **Horloge limitée à un bean** (T7) : le port applicatif reste à `gestion-taches` T3 pour ne pas créer de
  port sans cas d'usage ; léger travail restant pour ce plan.
- **Outil de tests de contrat non fixé ici** : conforme à `gestion-taches` T7, mais l'initialisation ne
  prépare pas cette dépendance.
- **Coordination entre dépôts** : R1 et R2 exigent une session séparée sur `platform-architecture` et
  l'acceptation de l'ADR 0004 avant T1, ce qui retarde le premier code.
- **Surface exposée** : les points de santé sans jeton (confirmés par `gestion-taches` T5) révèlent l'état de
  la base s'ils sont routés publiquement (Q10) ; les détails de composants restent masqués.
