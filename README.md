# task-service

Service de gestion des tâches (bounded context `tasks`) de la plateforme. Architecture hexagonale, DDD,
Java / Spring Boot, PostgreSQL. Voir `../platform-architecture/` pour le catalogue, les standards et les contrats.

## Statut
Dépôt initialisé : aucun code applicatif pour l'instant. Le premier plan est l'initialisation technique du service.

## Prérequis
Dépôts clonés côte à côte : `platform-ai`, `platform-architecture`, `task-service`.
Outils : JDK (version à figer par ADR), Maven, Docker, `jq` (hook de formatage), Claude Code.

## Travailler avec l'équipe d'agents
Depuis la racine de ce dépôt :
```
claude --add-dir ../platform-architecture
/plugin marketplace add ../platform-ai
/plugin install dev-team@platform-ai
```
Puis : `/dev-team:plan <demande>` → validation du plan → `/dev-team:implement <slug>` → relecture du diff → push par le CTO.
Pour itérer sur les agents sans publier : `claude --plugin-dir ../platform-ai/plugins/dev-team`, puis `/reload-plugins`.
