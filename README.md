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

## Modèle minimal d'autorisation

L'autorisation devra être évaluée côté serveur à partir de l'identité authentifiée,
de l'espace documentaire et de l'action demandée. Un identifiant d'organisation
ou un rôle fourni uniquement dans le corps de la requête ne constitue jamais une
preuve d'accès.

| Rôle | Lire et rechercher | Importer une version | Valider un résultat | Supprimer un document | Gérer les membres |
| --- | --- | --- | --- | --- | --- |
| `lecteur` | oui, dans son espace | non | non | non | non |
| `contributeur` | oui, dans son espace | oui | non | non | non |
| `réviseur` | oui, dans son espace | non | oui | non | non |
| `administrateur_espace` | oui, dans son espace | oui | oui | oui, avec confirmation | oui |
| `service_ingestion` | non interactif | oui, via une identité technique dédiée | non | non | non |

Cette matrice constitue un minimum : une organisation pourra restreindre davantage
les droits par dossier ou document, mais aucun rôle ne devra obtenir implicitement
un accès à un autre espace. Les contrôles suivants seront obligatoires :

- refuser par défaut toute action absente de la politique et vérifier l'autorisation
  à chaque requête, tâche asynchrone et reprise ;
- filtrer les documents autorisés avant la recherche lexicale ou vectorielle ;
- propager `workspace_id`, l'identité du sujet et la version de la politique dans
  les travaux en arrière-plan, sans accepter leur remplacement par le client ;
- inclure l'espace et une version des droits dans les clés de cache, puis invalider
  les résultats et liens temporaires lorsqu'un accès est retiré ;
- séparer les identités techniques des comptes humains et leur accorder uniquement
  les permissions requises pour une tâche déterminée ;
- journaliser les décisions d'autorisation accordées ou refusées sans enregistrer
  le texte contractuel, les jetons ou les secrets.

Les tests devront couvrir au minimum la lecture inter-espace refusée, l'élévation
de rôle par paramètre client, la révocation d'un membre, un cache créé avant la
révocation et une tâche d'ingestion tentant d'accéder à un autre espace.

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

## Frontière de confiance des modèles et fournisseurs

Les services d'embedding, de reranking et de génération sont des destinataires
potentiels de données contractuelles. Avant d'activer un fournisseur externe,
l'architecture devra documenter précisément quelles données quittent la plateforme,
dans quelle région elles sont traitées et pendant combien de temps elles sont
conservées. Un fournisseur non approuvé ou une clé API fournie par le client ne
devra jamais recevoir automatiquement un document ou un passage.

Les contrôles minimaux seront les suivants :

- maintenir une liste blanche versionnée des fournisseurs, modèles, régions et
  usages autorisés pour chaque espace documentaire ;
- désactiver l'entraînement sur les données envoyées et obtenir des garanties
  contractuelles sur la rétention, les sous-traitants et la suppression ;
- transmettre uniquement les passages nécessaires, après suppression des
  métadonnées et identifiants qui ne sont pas utiles à la tâche ;
- appliquer les mêmes règles aux embeddings, rerankers, modèles de génération,
  outils d'observabilité et mécanismes de secours ;
- interdire les appels sortants lorsqu'un espace exige un traitement local ou
  lorsqu'aucun fournisseur compatible avec sa politique n'est disponible ;
- isoler les clés par environnement et fournisseur, limiter leurs droits, les
  conserver hors des journaux et prévoir leur rotation ;
- journaliser le fournisseur, le modèle, la région, la politique appliquée et les
  identifiants des passages transmis, sans recopier leur contenu.

L'interface devra indiquer clairement lorsqu'une opération utilise un service
externe. Les tests devront vérifier qu'un espace configuré en traitement local ne
génère aucun appel sortant, qu'un fournisseur non autorisé est refusé et qu'un
mécanisme de repli ne contourne jamais la politique de l'espace.

## Sécurité des fichiers importés

L'import d'un contrat doit être traité comme une frontière de sécurité distincte du pipeline RAG. Un fichier reste en quarantaine tant que tous les contrôles ci-dessous ne sont pas terminés ; son nom, son extension et le type MIME annoncé par le client ne suffisent jamais à l'autoriser.

- maintenir une liste blanche versionnée de formats acceptés et vérifier la signature binaire ainsi que la structure réelle du document ;
- imposer des limites sur la taille du fichier, le nombre de pages, le nombre d'entrées d'une archive et sa taille décompressée ;
- refuser les fichiers chiffrés, les archives récursives, les chemins traversant des répertoires et tout dépassement de limite ;
- exécuter une analyse antimalware avant l'extraction, puis consigner dans l'audit la version de l'outil et l'empreinte du fichier ;
- isoler l'OCR, la conversion et le parsing sans accès réseau sortant, avec des limites de temps, de mémoire, de CPU et d'espace disque ;
- ne jamais exécuter les macros, scripts, objets embarqués, liens externes ou autres contenus actifs ;
- chiffrer le document en transit et au repos, puis supprimer les fichiers temporaires à la fin du traitement.

Un fichier refusé ne produit aucun texte indexable ni embedding. L'API retourne au client un message générique et un code d'erreur stable, tandis que le journal d'audit conserve le motif technique sans recopier le contenu du contrat.

Les tests d'acceptation couvrent au minimum une fausse extension, un fichier tronqué, une archive avec traversée de chemin, une archive surdimensionnée, un contenu actif et un dépassement de délai du parseur.

## Cycle d'ingestion et de versionnage

Chaque import devra créer une version immuable rattachée à un document et à un
espace documentaire. Remplacer un fichier existant créera un nouveau
`version_id` ; aucune réindexation ne modifiera silencieusement une version
déjà citée.

| État | Signification |
| --- | --- |
| `reçue` | Fichier accepté, contrôles de taille et de type terminés |
| `extraction_en_cours` | Texte et structure en cours d'extraction |
| `indexation_en_cours` | Passages et embeddings en cours de création |
| `indexée` | Version publiée et accessible à la recherche |
| `échec` | Import interrompu, avec étape et erreur consultables |
| `suppression_en_cours` / `supprimée` | Version bloquée puis purgée selon le cycle décrit ci-dessous |

Une version ne deviendra `indexée` qu'après validation conjointe du texte
extrait, des positions de passages, du `content_hash` et du nombre d'entrées
écrites dans l'index vectoriel. Avant cette transition, aucun passage de la
version ne devra être visible par la recherche. La publication devra être
atomique : soit tous les passages validés deviennent accessibles, soit aucun.

L'import acceptera une clé d'idempotence liée à l'espace documentaire. Répéter
la même requête avec la même clé et le même contenu retournera la version déjà
créée ; réutiliser cette clé avec un contenu différent sera rejeté. Le hash du
fichier original et la configuration d'extraction seront enregistrés afin de
distinguer un doublon exact d'une nouvelle version.

En cas d'échec, les artefacts partiels (texte, passages, embeddings et fichiers
temporaires) seront supprimés ou marqués comme non publiés. Une reprise
contrôlée pourra réutiliser le fichier original, mais créera une nouvelle
tentative traçable sans masquer l'erreur précédente.

Les tests d'intégration devront vérifier au minimum :

- qu'une version partiellement indexée n'apparaît jamais dans les résultats ;
- que deux imports identiques avec la même clé ne créent pas deux versions ;
- qu'une même clé associée à un contenu différent est refusée ;
- qu'une nouvelle version ne modifie ni les citations ni les hashes historiques ;
- qu'un échec nettoie les artefacts partiels sans toucher aux autres espaces.

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
