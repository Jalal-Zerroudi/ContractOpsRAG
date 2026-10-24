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

## Traçabilité minimale des citations

Chaque passage indexé devra conserver suffisamment de métadonnées pour retrouver précisément sa source et vérifier une réponse :

| Champ | Utilité |
| --- | --- |
| `document_id` | Identifiant stable du contrat |
| `version_id` | Version exacte ayant servi à la réponse |
| `source_name` | Nom de fichier ou référence lisible de la source |
| `page_number` | Page d'origine lorsqu'elle est disponible |
| `char_start` / `char_end` | Position du passage dans le texte extrait |
| `excerpt` | Extrait effectivement présenté comme preuve |
| `content_hash` | Contrôle d'intégrité du contenu indexé |
| `retrieval_score` | Score utilisé pour classer le passage |

L'extrait cité devra provenir directement de la version indexée, et l'interface devra permettre de revenir au passage correspondant. Une réindexation ne devra pas écraser les références historiques utilisées par des réponses déjà enregistrées.

### Conventions de localisation et d'intégrité

Pour le prototype, `page_number` commencera à 1 et sera absent si le format source
ne fournit pas de pages. Les positions `char_start` et `char_end` désigneront un
intervalle semi-ouvert `[début, fin)`, compté en points de code Unicode dans le
texte extrait complet de la version, et non en octets du PDF ou en unités UTF-16.

Le texte extrait de référence sera conservé sans modification après indexation.
`content_hash` sera le SHA-256 hexadécimal de ce texte complet encodé en UTF-8,
sans normalisation supplémentaire au moment de la vérification. L'extrait devra
être exactement égal à la tranche désignée par les positions. Un changement de
texte extrait imposera une nouvelle version ; un hash ou un extrait incohérent
bloquera la validation de la citation plutôt que de déplacer silencieusement ses positions.

## Principes de sécurité

Le traitement de documents contractuels exige une attention particulière à la confidentialité. L'implémentation devra notamment prévoir :

- une séparation stricte des données entre utilisateurs ou organisations ;
- des autorisations explicites pour chaque document ;
- un journal d'audit des consultations et modifications ;
- une gestion des secrets hors du dépôt ;
- une politique de conservation et de suppression des données ;
- la validation des citations avant toute utilisation opérationnelle.

## Risques propres au pipeline RAG

Les contrats importés devront être traités comme des contenus non fiables. Une instruction présente dans un document constitue une donnée à analyser et ne doit jamais modifier les règles du système, les autorisations ou le comportement des outils.

Les contrôles suivants devront être appliqués dès le premier prototype :

- filtrer les documents autorisés avant la recherche vectorielle, et non après la génération ;
- propager l'identifiant de l'organisation et les droits d'accès à chaque passage indexé ;
- isoler et délimiter clairement les extraits transmis au modèle ;
- refuser toute réponse lorsque les passages autorisés ne fournissent pas de preuve suffisante ;
- valider les types, la taille et le contenu des fichiers avant leur indexation ;
- tester des documents adversariaux contenant de fausses instructions, des demandes d'exfiltration ou des références trompeuses ;
- vérifier qu'une citation prouve la réponse sans révéler un document inaccessible à l'utilisateur.

## Cycle de suppression et de désindexation

La suppression d'un contrat devra rendre immédiatement toutes ses versions
indisponibles à la recherche, même si la purge physique s'exécute ensuite de
manière asynchrone. Une demande autorisée créera d'abord un état logique
`suppression_en_cours` bloquant toute nouvelle récupération ou citation.

La purge devra couvrir l'ensemble des dérivés liés au document :

- fichier original et versions importées ;
- texte extrait, clauses et passages découpés ;
- embeddings et entrées de l'index vectoriel ;
- caches de recherche ou de génération ;
- brouillons, exports et pièces temporaires ;
- liens de téléchargement ou URL signées encore actives.

Les réponses historiques ne devront pas conserver silencieusement le texte d'un
document supprimé. Selon la politique de conservation applicable, elles seront
supprimées ou réduites à des métadonnées non sensibles indiquant que leur source
n'est plus disponible. Le journal d'audit pourra conserver l'identifiant, le
demandeur, le motif, les horodatages et le résultat de la purge, mais pas le
contenu contractuel effacé.

Chaque opération devra produire un rapport de suppression avec le nombre
d'éléments attendus et effectivement supprimés par stockage. L'état final ne
deviendra `supprimé` qu'après vérification de l'absence du document dans le
stockage principal, l'index vectoriel et les caches. Un échec partiel restera
visible, bloquera toute réindexation avec le même `version_id` et déclenchera
une reprise contrôlée.

Les tests devront démontrer qu'un document supprimé ne peut plus être retrouvé
par identifiant, recherche lexicale, recherche vectorielle, citation existante
ou cache, tout en confirmant que les documents d'autres espaces restent intacts.

## Feuille de route initiale

- définir les formats de documents pris en charge ;
- documenter le modèle de données et les limites d'accès ;
- créer un premier pipeline d'ingestion et d'indexation ;
- exposer une API de recherche avec citations ;
- ajouter des tests sur l'isolation des données et la traçabilité ;
- mettre en place l'observabilité et un déploiement de démonstration.


## Règles de comparaison et de redlining

La comparaison devra recevoir deux identifiants de version immuables appartenant au même dossier contractuel. Elle ne devra jamais modifier les textes sources.

Chaque clause sera classée avec un statut explicite : `inchangée`, `ajoutée`, `supprimée`, `modifiée` ou `déplacée`. L'alignement utilisera en priorité un identifiant stable, puis la numérotation, le titre et la similarité du texte. Une correspondance incertaine restera visible au lieu d'être fusionnée automatiquement.

| Champ de sortie | Rôle |
| --- | --- |
| `source_clause_id` / `target_clause_id` | Références des clauses comparées, avec une valeur absente pour un ajout ou une suppression |
| `change_type` | Type de changement parmi les cinq statuts définis |
| `source_text` / `target_text` | Textes exacts des deux versions |
| `diff` | Segments ajoutés et supprimés sans reformulation générative |
| `alignment_confidence` | Niveau de confiance de la correspondance |
| `review_status` | État `à_valider`, `validé` ou `rejeté` et identité du réviseur |

Les correspondances sous le seuil de confiance, les déplacements ambigus et toute suggestion de reformulation devront être validés par un humain. Les explications du modèle resteront séparées du diff déterministe afin qu'une interprétation générée ne puisse pas être présentée comme une modification réellement observée.

## Protocole d'évaluation

Le prototype devra être évalué sur un jeu de questions versionné, construit uniquement à partir des contrats de démonstration. Chaque exemple devra contenir la question, la réponse attendue ou le statut `sans_reponse`, les passages de référence et l'espace documentaire autorisé.

Pour éviter d'ajuster le système sur ses propres résultats, les questions seront séparées en ensembles de développement et de test. La version du corpus, les paramètres de découpage, le modèle d'embedding, la configuration de recherche et le modèle génératif devront être enregistrés avec chaque exécution.

| Dimension | Mesure | Objectif du contrôle |
| --- | --- | --- |
| Recherche | `Recall@k` | Vérifier qu'au moins un passage de référence figure parmi les `k` premiers résultats |
| Classement | `MRR` | Mesurer la position du premier passage pertinent |
| Citations | Précision des citations | Vérifier que chaque extrait cité soutient réellement l'affirmation associée |
| Fidélité | Taux de réponses entièrement étayées | Compter les réponses dont toutes les affirmations importantes sont couvertes par une preuve |
| Abstention | Précision et rappel sur `sans_reponse` | Mesurer les refus corrects et les réponses injustifiées |
| Isolation | Tests de fuite inter-espace | Exiger zéro passage ou citation provenant d'un espace non autorisé |
| Performance | Latence `p50` et `p95` | Suivre le temps de réponse typique et les cas lents |

Les résultats devront être comparés à une baseline simple de recherche lexicale. Toute régression sur l'isolation des données ou la validité des citations sera bloquante, même si les scores de recherche ou la latence s'améliorent.

## Critères d'acceptation du premier prototype

Le premier jalon pourra être considéré comme démontrable lorsque les conditions suivantes seront vérifiées :

- un corpus composé uniquement de contrats synthétiques ou librement publiables peut être ingéré de manière reproductible ;
- chaque passage indexé conserve l'identifiant du document et sa position dans la source ;
- chaque réponse générée cite les passages qui la justifient ;
- le système signale explicitement l'absence de preuve suffisante au lieu d'inventer une réponse ;
- des tests automatisés confirment qu'un utilisateur ne peut pas consulter les documents d'un autre espace ;
- les opérations d'import, de consultation et de suppression sont inscrites dans un journal d'audit ;
- les journaux techniques ne contiennent ni texte contractuel complet, ni secret, ni jeton d'accès ;
- une procédure locale documentée permet de lancer et d'arrêter tous les services.

## Données de démonstration

Aucun contrat réel ou confidentiel ne doit être ajouté au dépôt. Les exemples, tests et démonstrations devront utiliser des documents synthétiques, anonymisés ou explicitement publics. Les originaux et les exports contenant des données sensibles devront rester dans un stockage contrôlé, hors de Git.

## Limites

ContractOpsRAG est un outil d'assistance envisagé, pas un service de conseil juridique. Toute analyse produite devra être vérifiée par une personne qualifiée avant de guider une décision contractuelle.
