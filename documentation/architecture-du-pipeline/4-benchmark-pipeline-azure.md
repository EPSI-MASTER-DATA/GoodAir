# Benchmark Azure pour le pipeline GoodAir — Architecture retenue pour le MVP

## Contexte et décision

Suite à l'analyse des contraintes du projet GoodAir, l'équipe a décidé d'adopter directement pour une architecture Azure comme solution de production dès le MVP, plutôt que de passer par une stack open source locale.

Cette décision repose sur plusieurs facteurs :

- la disponibilité d'un abonnement Azure dans le cadre du projet ;
- la nécessité de démontrer une architecture industrielle et sécurisée auprès du jury ;
- la volonté d'éviter la dette technique liée à une migration open source → cloud a posteriori ;
- la conformité RGPD facilitée par les certifications Azure (régions Europe, données localisées en France).

Ce document justifie les choix de composants Azure retenus pour chaque brique du pipeline, en répondant aux attentes du Bloc 3 (collecte, data lake, data warehouse, ETL/ELT) et du Bloc 5 (qualité des données, traçabilité, data visualisation, data science).

## Architecture retenue (MVP Azure)

```text
AQICN / OpenWeatherMap
          ↓
Azure Data Factory
(orchestration + déclenchement des jobs)
          ↓
Azure Functions (Python)
(extraction des APIs, écriture Bronze)
          ↓
Azure Data Lake Storage Gen2 — Bronze / Raw
(JSON bruts partitionnés par date et heure)
          ↓
Azure Databricks + Delta Lake
(nettoyage, normalisation, Silver)
(agrégations analytiques, Gold)
          ↓
Azure Synapse Analytics — Serverless SQL
(serving Data Warehouse, requêtes Power BI)
          ↓
Power BI
(dashboards, restitution métier)

─────────────────────────────────────────
Azure Monitor + Log Analytics         (transversal — supervise toutes les couches)
Azure Key Vault                       (transversal — secrets, clés API)
Microsoft Entra ID                    (transversal — identités, RBAC)
```

## Critères de benchmark

Les choix sont évalués selon les critères suivants, appliqués à chaque brique :

| Critère                     | Description                                                                 |
| --------------------------- | --------------------------------------------------------------------------- |
| Adéquation fonctionnelle    | Couverture complète des besoins ETL/ELT, DWH, qualité, data viz             |
| Scalabilité                 | Capacité à absorber la croissance des volumes et des usages                 |
| Coût maîtrisé (MVP)         | Capacité à limiter les coûts pour un premier déploiement                    |
| Exploitabilité              | Simplicité d'opérations, monitoring, maintenance                            |
| Gouvernance et sécurité     | IAM, secrets, chiffrement, traçabilité, résidence des données               |
| Cohérence de la suite Azure | Intégration native entre les composants, pas de friction d'interopérabilité |
| Réponse MSPR                | Couverture des attentes Bloc 3 et Bloc 5                                    |

# 1. Benchmark de l'orchestration

## Options comparées

| Solution                      | Avantages                                                                     | Limites                                                                    |
| ----------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Azure Data Factory            | Service managé, triggers horaires, retry, monitoring intégré, CI/CD ARM/Bicep | Coût des activités à surveiller, logique métier limitée dans les pipelines |
| Synapse Pipelines             | Identique à ADF, intégré dans Synapse Analytics                               | Couplage fort au workspace Synapse, moins flexible en dehors               |
| Airflow sur Azure (VM ou AKS) | Flexibilité maximale, portable, DAGs Python                                   | Charge d'exploitation importante (ops, sécurité, mises à jour)             |

## Choix retenu : Azure Data Factory

**Justification**

- **Adéquation fonctionnelle** : ADF orchestre le pipeline complet (déclenchement des extractions, enchaînement des jobs Databricks, supervision) sans écrire de code d'infrastructure.
- **Coût maîtrisé** : facturation à l'exécution, pas de ressource permanente à payer.
- **Exploitabilité** : interface visuelle, historique d'exécution, alertes natives, intégration Azure Monitor.
- **Cohérence Azure** : s'intègre nativement avec Azure Functions, Databricks, ADLS Gen2 et Key Vault.
- **Réponse MSPR** : répond à l'exigence de planification horaire, de supervision du pipeline et d'alertes en cas de problème.

# 2. Benchmark de l'extraction des données

## Options comparées

| Solution                         | Avantages                                                              | Limites                                                          |
| -------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------- |
| Azure Functions (Python)         | Léger, serverless, parfait pour des appels API REST, déclenché par ADF | Timeout à gérer pour des volumes très importants                 |
| Databricks notebooks             | Puissant, déjà présent dans la stack                                   | Surdimensionné pour un simple appel HTTP REST                    |
| ADF Copy Activity / Web Activity | Simple pour des cas basiques                                           | Logique Python complexe (retry, parsing, validation) peu adaptée |
| Container App / VM               | Contrôle total                                                         | Charge d'exploitation élevée                                     |

## Choix retenu : Azure Functions (Python)

**Justification**

- **Adéquation fonctionnelle** : les APIs AQICN et OpenWeatherMap retournent du JSON, facilement traité en Python avec `requests` ou `httpx`.
- **Coût maîtrisé** : serverless, facturation uniquement à l'exécution.
- **Exploitabilité** : déclenchée par ADF, les logs sont centralisés dans Azure Monitor.
- **Cohérence Azure** : les secrets (clés API) sont lus depuis Azure Key Vault, les fichiers écrits directement sur ADLS Gen2.
- **Réponse MSPR** : répond à l'attente Bloc 5 d'utilisation de Python pour produire des pipelines et des jeux de données nettoyés.

# 3. Benchmark du Data Lake (couche Bronze)

## Options comparées

| Solution                     | Avantages                                                                     | Limites                                                             |
| ---------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Azure Data Lake Storage Gen2 | Standard Azure, ACL fines, hiérarchique, excellent avec Databricks et Synapse | Coût stockage + transactions à piloter                              |
| Azure Blob Storage           | Économique                                                                    | Moins riche en gouvernance data lake (pas de hiérarchie ACL réelle) |
| MinIO (local)                | Open source, compatible S3                                                    | Non scalable en production, gouvernance limitée                     |

## Choix retenu : Azure Data Lake Storage Gen2

**Justification**

- **Adéquation fonctionnelle** : stockage des JSON bruts partitionnés, conservation de l'intégralité des données originales.
- **Scalabilité** : conçu pour absorber des volumes croissants sans reconfiguration.
- **Gouvernance et sécurité** : ACL au niveau dossier/fichier, chiffrement au repos, conformité RGPD avec résidence en région Europe.
- **Cohérence Azure** : compatible nativement avec Databricks, Synapse, Purview et ADF.
- **Réponse MSPR** : répond à l'exigence Bloc 3 de création d'un data lake pour collecter des données brutes et au Bloc 5 de gestion du cycle de vie des données.

Organisation de l'arborescence :

```text
/bronze/aqicn/year=YYYY/month=MM/day=DD/hour=HH/
/bronze/openweather/year=YYYY/month=MM/day=DD/hour=HH/
```

# 4. Benchmark des transformations (Bronze → Silver → Gold)

## Options comparées

| Solution                      | Avantages                                                                         | Limites                                                  |
| ----------------------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------- |
| Azure Databricks + Delta Lake | Très scalable, excellent pour JSON hétérogène, ACID, time travel, schéma évolutif | Coût compute à bien dimensionner                         |
| Synapse Spark                 | Intégré à Synapse                                                                 | Écosystème et ergonomie moins matures que Databricks     |
| Azure Functions + SQL         | Léger pour de petits flux simples                                                 | Peu adapté aux transformations volumineuses et complexes |
| dbt on Synapse                | Versionnable, testable, documenté                                                 | Moins adapté au parsing de JSON bruts complexes          |

## Choix retenu : Azure Databricks + Delta Lake

**Justification**

- **Adéquation fonctionnelle** : parsing JSON, normalisation, déduplication, gestion du schéma évolutif (polluants dynamiques AQICN), enrichissement analytique.
- **Scalabilité** : architecture Spark, montée en charge transparente.
- **Exploitabilité** : Delta Lake garantit ACID, upsert et time travel, utiles pour la rejouabilité et l'audit.
- **Cohérence Azure** : lecture/écriture native sur ADLS Gen2, intégration ADF pour le déclenchement, logs dans Azure Monitor.
- **Réponse MSPR** : répond à l'exigence Bloc 3 d'ETL/ELT et au Bloc 5 de préparation, nettoyage et qualité des données.

Couverture des traitements :

**Bronze → Silver (technique)**

- parsing des fichiers JSON ;
- extraction des champs utiles ;
- nettoyage des valeurs invalides (`"-"` → `NULL`) ;
- conversion des types et normalisation des timestamps ;
- gestion des champs dynamiques (polluants AQICN) ;
- déduplication et contrôles de cohérence.

**Silver → Gold (analytique)**

- agrégations et enrichissements métier ;
- construction des dimensions et tables de faits ;
- préparation des indicateurs pour Power BI.

# 5. Benchmark du Data Warehouse (couche Gold)

## Options comparées

| Solution                                 | Avantages                                                                    | Limites                                        |
| ---------------------------------------- | ---------------------------------------------------------------------------- | ---------------------------------------------- |
| Azure Synapse Analytics — Serverless SQL | Paiement à la requête, requête directe sur Delta/ADLS, aucun cluster à gérer | Moins performant pour des charges BI soutenues |
| Azure Synapse — Dedicated SQL Pool       | DWH MPP managé, très performant pour la BI à grande échelle                  | Coût fixe important, overkill pour un MVP      |
| Azure SQL Database                       | Simple, connu, SQL standard                                                  | Limites sur de très gros volumes analytiques   |
| Databricks SQL Warehouse                 | Très performant, unifié avec la stack Databricks                             | Coût compute élevé si mal dimensionné          |

## Choix retenu : Azure Synapse Analytics — Serverless SQL

**Justification**

- **Coût maîtrisé** : pas de ressource permanente, facturation uniquement au volume de données scannées.
- **Adéquation fonctionnelle** : requêtes SQL directes sur les tables Delta Gold stockées dans ADLS Gen2, sans duplication de données.
- **Exploitabilité** : connecteur natif Power BI, pas de gestion de cluster.
- **Cohérence Azure** : intégré à l'espace de travail Synapse, visible depuis Azure Monitor.
- **Réponse MSPR** : répond à l'exigence Bloc 3 de création d'un entrepôt unique orienté métier.

Modèle en étoile retenu :

```text
fact_air_quality
dim_city
dim_station
dim_time
dim_pollutant
```

# 6. Benchmark de la qualité des données

## Options comparées

| Solution                               | Avantages                                                    | Limites                                          |
| -------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| Contraintes Delta Lake                 | Natif, léger, validations lors de l'écriture                 | Contrôles limités aux règles d'intégrité de base |
| Great Expectations (dans Databricks)   | Très complet, documentation automatique, rapports de qualité | Mise en place plus lourde                        |
| dbt tests (sur Synapse)                | Intégré aux transformations SQL, versionnable                | Principalement orienté tests SQL                 |
| Règles Python dans les jobs Databricks | Flexible, adapté aux contrôles métier spécifiques            | Documentation manuelle nécessaire                |

## Choix retenu : règles Python dans les jobs Databricks + contraintes Delta

**Justification**

- **Adéquation fonctionnelle** : les contrôles Python couvrent les cas spécifiques au projet (fraîcheur des données, valeurs aberrantes, polluants manquants).
- **Cohérence Azure** : exécutés dans les jobs Databricks déjà présents dans la stack.
- **Réponse MSPR** : répond à l'exigence Bloc 3 de surveillance de la qualité des données et au Bloc 5 de nettoyage, validation et traçabilité.

Contrôles prévus :

- `status = ok` pour les réponses API ;
- `aqi != "-"` ou conversion en `NULL` ;
- fraîcheur de la donnée (timestamp < 2h) ;
- absence de doublons ;
- cohérence des timestamps ;
- détection de valeurs aberrantes ;
- présence d'une station active ;
- nombre de mesures attendues par ville.

Les enregistrements invalides sont dirigés vers une zone de quarantaine dédiée sur ADLS Gen2, historisée pour audit.

# 7. Benchmark du monitoring et de l'alerting

## Options comparées

| Solution                        | Avantages                                                                                | Limites                                         |
| ------------------------------- | ---------------------------------------------------------------------------------------- | ----------------------------------------------- |
| Azure Monitor + Log Analytics   | Centralise les signaux de toutes les couches (ADF, Databricks, Synapse), alertes natives | Nécessite une configuration initiale des règles |
| Grafana + Prometheus            | Très bon pour le monitoring technique                                                    | Configuration supplémentaire hors Azure         |
| Databricks jobs monitoring seul | Simple pour les jobs Spark                                                               | Silotage, ne couvre pas ADF ni Synapse          |
| Application Insights            | Très bon pour les applications, Azure Functions                                          | Moins orienté data pipeline au sens large       |

## Choix retenu : Azure Monitor + Log Analytics + alertes

**Justification**

- **Adéquation fonctionnelle** : supervise toutes les couches du pipeline (ADF, Azure Functions, Databricks, Synapse, ADLS) depuis un point unique.
- **Exploitabilité** : tableaux de bord, règles d'alerte, intégration email et webhook.
- **Cohérence Azure** : natif, aucun outil supplémentaire à déployer.
- **Réponse MSPR** : répond à l'exigence d'alerte en cas de problème sur le pipeline ou la disponibilité des données.

Cas d'alerte couverts :

- échec d'un pipeline ADF ou d'un job Databricks ;
- API externe indisponible ou quota dépassé ;
- fraîcheur des données dépassée ;
- dérive de coût anormale.

# 8. Benchmark sécurité, gouvernance et conformité

## Composants retenus

| Besoin                        | Solution retenue                                      | Justification                                 |
| ----------------------------- | ----------------------------------------------------- | --------------------------------------------- |
| Secrets et clés API           | Azure Key Vault                                       | Aucun secret dans le code ou le dépôt Git     |
| Authentification et accès     | Microsoft Entra ID + RBAC                             | Principe du moindre privilège, auditabilité   |
| Réseau et exposition          | Private endpoints + VNet integration                  | Réduction de la surface d'exposition publique |
| Chiffrement                   | Chiffrement au repos et en transit natif ADLS/Synapse | Conformité RGPD                               |
| Gouvernance des données       | Microsoft Purview (phase 2)                           | Catalogue, lignage, classification            |
| Séparation des environnements | Groupes de ressources dev/test/prod                   | Fiabilisation des déploiements                |

**Justification**

- **Gouvernance et sécurité** : le principe du moindre privilège et la traçabilité des accès sont assurés nativement par Entra ID et Key Vault, sans configuration manuelle supplémentaire.
- **Conformité RGPD** : Azure garantit la résidence des données en région Europe (France Centre disponible), répondant à l'exigence de localisation en France ou dans l'Union Européenne.
- **Réponse MSPR** : répond aux exigences Bloc 3 de sécurisation des accès et au Bloc 5 de traçabilité et gouvernance des données.

# 9. Benchmark du moteur de recherche élastique

## Contexte

La grille d'évaluation MSPR (TPRE843 — Bloc 3) demande explicitement, dans la compétence "Créer un Data Lake", de préciser le **moteur de recherche élastique de données non structurées ou semi-structurées**. Elle cite Elasticsearch (ES) comme exemple de référence dans une architecture AWS.

Ce composant est la quatrième brique obligatoire du Data Lake, en complément du serveur de données (ADLS Gen2), du catalogue (Purview) et du moteur de requêtes (Synapse SQL).

## Options comparées

| Solution | Avantages | Limites |
| --- | --- | --- |
| Azure Cognitive Search | Service managé Azure, indexation native ADLS/Blob, recherche full-text, sémantique et vectorielle, intégration Power BI | Coût à l'usage selon le volume indexé |
| Elasticsearch (autohébergé) | Standard du marché, très puissant, open source | Charge d'exploitation importante, hors écosystème Azure natif |
| Elastic Cloud sur Azure | Elasticsearch managé, hébergé sur Azure | Coût élevé, dépendance fournisseur supplémentaire |
| Azure AI Search (anciennement Cognitive Search) | Même service, renommé 2024, capacités IA enrichies (vecteurs, sémantique) | Idem Azure Cognitive Search |

## Choix retenu : Azure Cognitive Search

**Justification**

- **Adéquation fonctionnelle** : indexe directement les fichiers JSON bruts stockés dans ADLS Gen2 (couche Bronze), permettant la recherche full-text sur les noms de stations, de villes, les polluants et les attributions de sources.
- **Couverture MSPR** : répond à l'exigence explicite d'un moteur de recherche élastique sur les données non structurées et semi-structurées du Data Lake.
- **Cohérence Azure** : intégration native avec ADLS Gen2 (indexeur Blob), authentification Entra ID, coût à l'usage sans infrastructure à gérer.
- **Complémentarité avec Synapse SQL** : les deux moteurs coexistent avec des rôles distincts — Synapse SQL pour les requêtes analytiques structurées sur Silver/Gold, Azure Cognitive Search pour l'exploration et la recherche sur Bronze.
- **Scalabilité** : montée en charge automatique selon le volume de données indexées.

## Positionnement dans le Data Lake

Azure Cognitive Search intervient sur la **couche Bronze**, en lecture seule depuis ADLS Gen2. Il n'écrit pas de données — il produit un index de recherche.

```text
ADLS Gen2 — Bronze (JSON bruts)
        ↓
Azure Cognitive Search
(indexeur Blob → index full-text)
        ↓
Recherche et exploration des données semi-structurées
```

## Complémentarité des moteurs de requêtes

| Besoin | Moteur | Couche |
| --- | --- | --- |
| Requêtes analytiques SQL structurées | Synapse Serverless SQL | Gold |
| Exploration full-text, recherche par station ou polluant | Azure Cognitive Search | Bronze |
| Transformations et agrégations | Azure Databricks | Bronze → Silver → Gold |

# 10. Benchmark de la data visualisation

## Options comparées

| Solution        | Avantages                                                               | Limites                                                 |
| --------------- | ----------------------------------------------------------------------- | ------------------------------------------------------- |
| Power BI        | Connecteur natif Synapse/Azure, très professionnel, standard entreprise | Licence Pro requise pour le partage                     |
| Apache Superset | Open source, puissant                                                   | Configuration supplémentaire, hors écosystème Azure     |
| Metabase        | Simple et rapide                                                        | Moins intégré à Azure, moins professionnel pour le jury |
| Grafana         | Très bon pour les séries temporelles                                    | Moins orienté BI métier classique                       |

## Choix retenu : Power BI

**Justification**

- **Cohérence Azure** : connecteur natif avec Synapse Serverless SQL, authentification Entra ID, déploiement dans le tenant Azure du projet.
- **Adéquation fonctionnelle** : dashboards interactifs, analyses temporelles, export des données.
- **Réponse MSPR** : répond à l'exigence Bloc 3 de restitution via un outil de data visualisation et au Bloc 5 de production de rapports compréhensibles pour les métiers.

# Synthèse de l'architecture retenue

| Brique                          | Composant Azure retenu                   | Rôle                                                  |
| ------------------------------- | ---------------------------------------- | ----------------------------------------------------- |
| Orchestration                   | Azure Data Factory                       | Planification, déclenchement, supervision             |
| Extraction API                  | Azure Functions (Python)                 | Appels REST, écriture Bronze                          |
| Data Lake                       | ADLS Gen2                                | Stockage Reference/Bronze/Silver/Gold                 |
| Transformations                 | Azure Databricks + Delta Lake            | Nettoyage, normalisation, agrégation                  |
| Moteur de requêtes analytiques  | Synapse Analytics — Serverless SQL       | Serving analytique structuré, requêtes Power BI       |
| Moteur de recherche élastique   | Azure Cognitive Search                   | Recherche full-text sur données Bronze semi-structurées |
| Catalogue de données            | Microsoft Purview                        | Lignage, classification, gouvernance                  |
| Qualité des données             | Jobs Databricks + contraintes Delta      | Contrôles, quarantaine, KPIs qualité                  |
| Monitoring                      | Azure Monitor + Log Analytics            | Supervision transversale, alertes                     |
| Sécurité                        | Key Vault + Entra ID + Private endpoints | Secrets, RBAC, réseau                                 |
| Data visualisation              | Power BI                                 | Dashboards, restitution métier                        |

# Risques et points de vigilance

- **Dérive de coûts** : Azure est facturable à l'usage. Sans gouvernance FinOps, les coûts peuvent déraper rapidement (principalement sur Databricks compute et Synapse scans).
- **Dimensionnement Databricks** : les clusters auto-terminating et le choix du tier (Standard vs Premium) ont un impact direct sur le budget.
- **IaC dès le départ** : la gestion des ressources Azure doit être versionnée (Bicep ou Terraform) pour garantir la reproductibilité et la séparation des environnements.
- **CI/CD des pipelines ADF** : les pipelines ADF doivent être versionnés dans Git (intégration native ADF/GitHub) pour éviter la configuration manuelle.

Actions recommandées :

- tagging des ressources par domaine (ingestion, compute, BI) pour le suivi des coûts ;
- politique d'auto-stop des clusters Databricks hors plages utiles ;
- définition des SLO pipeline (fraîcheur < 2h, taux de succès > 99%, temps de traitement) ;
- revue mensuelle architecture et coûts.
