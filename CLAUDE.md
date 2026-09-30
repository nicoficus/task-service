# task-service

## Identité
Tu travailles dans le dépôt **task-service**.
Rôle : gère le cycle de vie des tâches (création, statuts, échéances). Cœur du domaine de la plateforme.
Bounded context : `tasks`. Données : PostgreSQL, schéma `tasks` (accès exclusif de ce service).

## Référentiel d'architecture
Le dépôt `platform-architecture` est cloné à côté de celui-ci : `../platform-architecture/`.
Lis-y avant de planifier, d'implémenter ou de réviser :
- `catalog/services.yaml` : rôle, contrats et dépendances de chaque service (le tien et ses voisins)
- `standards/engineering-standards.md` : règles d'architecture, qualité, définition de « terminé »
- `ubiquitous-language.md` : vocabulaire métier
- `adr/`, `contracts/`
Si ce dossier est absent, arrête-toi et demande au CTO : ne suppose rien sur les autres services.

## État du projet
Dépôt vide : aucune structure Maven ni code pour l'instant. Les versions (Java, Spring Boot) et le runtime
sont à figer par ADR (`../platform-architecture/adr/0002-architecture-cible.md`) avant tout code.
Le premier plan prévu est l'initialisation technique du service.

## Stack et commandes (à confirmer après l'initialisation technique)
- Build et contrôles : `mvn verify`
- Formatage : `mvn spotless:apply`
- Local : `docker compose up --build`

## Workflow
`/dev-team:plan <demande>` → validation du CTO → `/dev-team:implement <slug>` → `/dev-team:review`.
Le CTO relit le diff et pousse lui-même.

## Interdits
- Jamais de `git push`, de fusion, ni de travail direct sur `main`.
- Jamais de commande AWS, `terraform apply` ou `cdk deploy`.
- Jamais de lecture ou d'écriture de `.env` ni de secret.
- Jamais de modification d'un autre dépôt depuis cette session.
- Jamais d'implémentation sans plan au statut `validé`.
- Jamais de test ou de règle de lint désactivé pour passer au vert : le signaler.
- Doute métier ou architectural : poser la question au CTO.

## Commits
Conventional Commits, un commit par tâche cohérente.
