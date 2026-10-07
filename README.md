# ContractOpsRAG

ContractOpsRAG est un projet de copilote contractuel dédié aux contrats français. Son objectif est d'aider à retrouver des informations dans un corpus contractuel, comparer des versions et examiner les différences entre clauses, tout en conservant des références vers les sources utilisées.

> **Statut du dépôt :** ce dépôt présente actuellement le concept et l'architecture cible. Aucun code exécutable ni procédure de déploiement n'est encore versionné.

## Fonctionnalités prévues

- questions-réponses sur des contrats avec citations des passages sources ;
- recherche augmentée par génération (RAG) sur un corpus documentaire ;
- comparaison de versions de contrats ;
- identification et redlining des modifications de clauses ;
- contrôle d'accès par rôles et journalisation des actions sensibles.

## Architecture cible

| Composant | Rôle envisagé |
| --- | --- |
| ASP.NET Core | API métier et orchestration des opérations contractuelles |
| FastAPI | Services de traitement documentaire et d'intelligence artificielle |
| PostgreSQL + pgvector | Stockage des données et recherche vectorielle |
| OAuth2 / OpenID Connect | Authentification et autorisation |
| Docker | Environnements reproductibles pour les services |
| GitHub Actions | Intégration et déploiement continus |
| Azure Container Registry | Stockage des images de conteneurs |
| Azure Container Apps ou AKS | Hébergement des services |
| OpenTelemetry + Azure Monitor | Traces, métriques et supervision |

## Parcours fonctionnel envisagé

1. Importer et indexer un ensemble de contrats.
2. Découper les documents en passages traçables.
3. Rechercher les passages pertinents pour une question donnée.
4. Générer une réponse accompagnée de citations vérifiables.
5. Comparer deux versions et présenter les changements de clauses.

## Principes de sécurité

Le traitement de documents contractuels exige une attention particulière à la confidentialité. L'implémentation devra notamment prévoir :

- une séparation stricte des données entre utilisateurs ou organisations ;
- des autorisations explicites pour chaque document ;
- un journal d'audit des consultations et modifications ;
- une gestion des secrets hors du dépôt ;
- une politique de conservation et de suppression des données ;
- la validation des citations avant toute utilisation opérationnelle.

## Feuille de route initiale

- définir les formats de documents pris en charge ;
- documenter le modèle de données et les limites d'accès ;
- créer un premier pipeline d'ingestion et d'indexation ;
- exposer une API de recherche avec citations ;
- ajouter des tests sur l'isolation des données et la traçabilité ;
- mettre en place l'observabilité et un déploiement de démonstration.

## Limites

ContractOpsRAG est un outil d'assistance envisagé, pas un service de conseil juridique. Toute analyse produite devra être vérifiée par une personne qualifiée avant de guider une décision contractuelle.
