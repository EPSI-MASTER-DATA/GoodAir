# Benchmark des solutions de pipeline et justification de l’architecture retenue

## Contexte

Dans le cadre du projet GoodAir, un pipeline de données doit être mis en place afin de collecter, transformer, stocker et restituer les données issues des APIs externes (AQICN et OpenWeatherMap).

Le cahier des charges impose notamment :

- la collecte de données issues d’APIs externes ;
- l’historisation horaire des données temps réel ;
- une modélisation normalisée ;
- un stockage permettant de requêter efficacement des volumes croissants ;
- la mise en place d’un data lake ;
- la création d’un data warehouse ;
- la surveillance de la qualité des données ;
- la conformité RGPD ;
- la restitution via un outil de data visualisation.

Ces éléments répondent directement aux attentes du Bloc 3, qui demande notamment une architecture de collecte, de stockage, de data lake, de data warehouse et de processus ETL/ELT . Ils répondent également aux attentes du Bloc 5, qui insiste sur la préparation, le nettoyage, la qualité, la traçabilité, la data visualisation et les usages statistiques ou de data science .

## Architecture retenue

L’architecture retenue est une architecture open source et locale, composée des éléments suivants :

```text
APIs AQICN / OpenWeatherMap
        ↓
Apache Airflow
        ↓
Scripts Python d’extraction
        ↓
MinIO — Data Lake / Bronze
        ↓
Python — Transformations techniques
        ↓
PostgreSQL — ODS normalisé / Silver
        ↓
dbt / SQL — Transformations analytiques
        ↓
PostgreSQL — Data Warehouse / Gold
        ↓
Metabase / Apache Superset
```

Cette architecture est retenue comme solution principale pour le MVP du projet GoodAir.

## Critères de benchmark

Le benchmark s’appuie sur les critères suivants :

| Critère                   | Description                                                      |
| ------------------------- | ---------------------------------------------------------------- |
| Couverture du besoin MSPR | Capacité à répondre aux attentes Bloc 3 et Bloc 5                |
| Maîtrise technique        | Facilité de prise en main par l’équipe                           |
| Coût                      | Capacité à limiter les coûts du projet                           |
| Démontrabilité            | Possibilité de présenter une solution fonctionnelle au jury      |
| Scalabilité               | Capacité à absorber une hausse du volume de données              |
| Traçabilité               | Capacité à conserver l’historique et les données brutes          |
| Qualité des données       | Capacité à contrôler, nettoyer et valider les données            |
| Sécurité                  | Gestion des accès, secrets, authentification                     |
| Évolutivité               | Capacité à évoluer vers une architecture cloud ou industrialisée |

# 1. Benchmark de l’orchestrateur

## Technologies comparées

| Technologie    | Avantages                                                                          | Limites                                                                              |
| -------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Apache Airflow | Très utilisé en data engineering, DAGs clairs, retries, logs, planification native | Configuration initiale plus lourde                                                   |
| Prefect        | Moderne, simple à prendre en main, bonne interface                                 | Moins standard dans certains contextes pédagogiques                                  |
| Dagster        | Très robuste pour les pipelines data modernes                                      | Courbe d’apprentissage plus élevée                                                   |
| Cron           | Très simple                                                                        | Pas de vraie supervision, pas de visualisation des dépendances, peu adapté au projet |

## Choix retenu : Apache Airflow

Apache Airflow est retenu car il permet de représenter clairement les étapes du pipeline sous forme de DAG.

Il répond bien aux besoins suivants :

- planifier une ingestion horaire ;
- superviser les tâches ;
- gérer les retries ;
- journaliser les erreurs ;
- déclencher des alertes ;
- rendre le pipeline démontrable.

## Justification

Airflow est particulièrement adapté au projet GoodAir, car le cahier des charges demande une récupération régulière des données temps réel et un système capable d’alerter en cas de problème sur le pipeline .

# 2. Benchmark de l’extraction des données

## Technologies comparées

| Technologie             | Avantages                                                   | Limites                                              |
| ----------------------- | ----------------------------------------------------------- | ---------------------------------------------------- |
| Python + requests/httpx | Simple, lisible, adapté aux APIs REST, très utilisé en data | Nécessite de coder la logique de retry et validation |
| Node.js + axios/fetch   | Performant pour les appels API                              | Moins naturel dans un pipeline data Python           |
| Bash + curl             | Très simple pour tester                                     | Peu maintenable pour un pipeline structuré           |
| Airbyte                 | Connecteurs prêts à l’emploi                                | Surdimensionné pour deux APIs REST simples           |

## Choix retenu : Python

Python est retenu pour les scripts d’extraction.

## Justification

Python est adapté car :

- les APIs AQICN et OpenWeatherMap retournent du JSON ;
- les premiers contrôles de validité des réponses peuvent être écrits dans le même écosystème ;
- Python est également utilisable ensuite pour la data quality et les analyses statistiques.

Ce choix répond aussi au Bloc 5, qui attend l’utilisation de langages adaptés comme Python pour produire des pipelines et des jeux de données nettoyées ou améliorées .

# 3. Benchmark du Data Lake

## Technologies comparées

| Technologie                  | Avantages                                                             | Limites                                     |
| ---------------------------- | --------------------------------------------------------------------- | ------------------------------------------- |
| MinIO                        | Compatible S3, open source, simple en local, adapté au stockage objet | Moins complet qu’un cloud provider managé   |
| Amazon S3                    | Standard cloud, scalable, robuste                                     | Coût, dépendance cloud, configuration IAM   |
| Azure Data Lake Storage Gen2 | Très adapté à une architecture Azure                                  | Dépendance Azure, coût, configuration       |
| Google Cloud Storage         | Scalable et robuste                                                   | Dépendance GCP                              |
| HDFS                         | Technologie Big Data historique                                       | Plus lourd à déployer et maintenir en local |

## Choix retenu : MinIO

MinIO est retenu comme Data Lake local.

## Justification

MinIO permet de stocker les données brutes des APIs sans transformation, dans une logique de data lake.

Exemple d’organisation :

```text
/raw/aqicn/YYYY/MM/DD/HH/
/raw/openweather/YYYY/MM/DD/HH/
```

Ce choix permet :

- de conserver les JSON originaux ;
- de rejouer les traitements en cas d’erreur ;
- d’assurer la traçabilité ;
- de démontrer concrètement la création d’un data lake.

Cela répond aux attentes du Bloc 3, qui demande la création d’un lac de données pour collecter des données brutes , et du Bloc 5, qui demande la manipulation d’architectures de type Data Lake et la gestion du cycle de vie des données .

# 4. Benchmark de la transformation technique

## Technologies comparées

| Technologie            | Avantages                                                              | Limites                                                        |
| ---------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------- |
| Python                 | Simple, lisible, adapté au parsing JSON, très maîtrisable par l’équipe | Moins performant que Spark sur de très gros volumes            |
| PySpark / Apache Spark | Scalable, adapté aux gros volumes et traitements distribués            | Surdimensionné pour le MVP, configuration plus lourde          |
| SQL intermédiaire      | Simple pour des données déjà structurées                               | Peu adapté au parsing de JSON bruts complexes                  |
| dbt                    | Versionnable, tests intégrés, documentation                            | Plus adapté aux transformations analytiques qu’au parsing brut |
| Apache Beam            | Puissant, compatible batch/streaming                                   | Complexité élevée pour le besoin du projet                     |

## Choix retenu : Python

Python est retenu pour les transformations techniques entre le Data Lake et l’ODS.

## Traitements réalisés

Les traitements techniques correspondent au passage de la couche Bronze vers la couche Silver :

```text
Data Lake / Bronze
        ↓
Transformations techniques
        ↓
ODS / Silver
```

Ils incluent :

- parsing des fichiers JSON ;
- extraction des champs utiles ;
- nettoyage des valeurs invalides (`"-"` → `NULL`) ;
- conversion des types ;
- normalisation des timestamps ;
- gestion des champs dynamiques, notamment les polluants AQICN ;
- transformation des structures imbriquées en tables relationnelles ;
- déduplication ;
- contrôles de cohérence avant insertion dans l’ODS.

## Justification

Les APIs AQICN et OpenWeatherMap retournent des données JSON hétérogènes, parfois incomplètes ou dynamiques. Avant leur stockage dans l’ODS, ces données doivent donc être nettoyées, typées, structurées et validées.

Python est le choix le plus adapté au MVP, car il permet de traiter simplement les JSON, d’appliquer des règles de nettoyage précises et de s’intégrer facilement dans les DAGs Airflow.

Spark pourra être envisagé dans une évolution future si les volumes deviennent réellement importants.

# 4. Benchmark de l’ODS / couche Silver

## Technologies comparées

| Technologie     | Avantages                                                                            | Limites                                                  |
| --------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------- |
| PostgreSQL      | Robuste, open source, SQL standard, JSONB disponible, très adapté à la normalisation | Moins spécialisé analytique qu’un DWH dédié              |
| MySQL / MariaDB | Simple et connu                                                                      | Moins riche pour certains usages analytiques avancés     |
| SQL Server      | Très complet                                                                         | Plus lourd, dépendance Microsoft                         |
| MongoDB         | Flexible pour JSON                                                                   | Moins adapté à une modélisation relationnelle normalisée |

## Choix retenu : PostgreSQL

PostgreSQL est retenu pour l’ODS normalisé.

## Justification

L’ODS sert à stocker les données déjà nettoyées, structurées et normalisées à l’issue des transformations techniques.

Il ne réalise pas directement les transformations, mais garantit que les données stockées sont cohérentes, historisées et exploitables.

Il permet de gérer :

- le référentiel des villes ;
- le référentiel des stations ;
- les mesures de qualité de l’air ;
- les polluants dynamiques ;
- les sources ;
- les données techniques associées aux traitements.

PostgreSQL est adapté car il permet une modélisation relationnelle propre, tout en offrant la possibilité de stocker du JSON si nécessaire.

Le cahier des charges impose une base normalisée capable d’être interrogée efficacement malgré une hausse progressive du volume de données.

# 6. Benchmark de la transformation analytique

## Technologies comparées

| Technologie            | Avantages                                                                        | Limites                                             |
| ---------------------- | -------------------------------------------------------------------------------- | --------------------------------------------------- |
| dbt                    | Très adapté aux transformations SQL, versionnable, tests intégrés, documentation | Nécessite un modèle SQL clair                       |
| SQL pur                | Simple, directement exécutable dans PostgreSQL                                   | Moins maintenable sans framework                    |
| Python / pandas        | Flexible pour analyses et prototypes                                             | Moins adapté à la construction maintenable d’un DWH |
| Apache Spark / PySpark | Scalable, adapté aux gros volumes                                                | Surdimensionné pour le MVP                          |

## Choix retenu : dbt + SQL

dbt est retenu pour les transformations analytiques entre l’ODS et le Data Warehouse.

## Traitements réalisés

Les transformations analytiques correspondent au passage de la couche Silver vers la couche Gold :

```text
ODS / Silver
        ↓
Transformations analytiques
        ↓
Data Warehouse / Gold
```

Elles incluent :

- agrégation des données ;
- pivot des polluants si nécessaire ;
- enrichissement métier ;
- création des dimensions ;
- création des tables de faits ;
- préparation des indicateurs pour la data visualisation.

## Justification

dbt est pertinent pour transformer les données propres de l’ODS vers un modèle orienté BI.

Il permet de construire un Data Warehouse en étoile de manière versionnée, documentée et testable.

Contrairement aux transformations techniques, qui visent à nettoyer et structurer les données brutes, les transformations analytiques visent à produire des tables métier optimisées pour les requêtes, les indicateurs et les dashboards.

# 6. Benchmark du Data Warehouse

Contrairement à l’ODS, qui conserve une structure normalisée proche des sources, le Data Warehouse est orienté métier et optimisé pour les analyses, les agrégations et la data visualisation.

## Technologies comparées

| Technologie   | Avantages                                                           | Limites                                                  |
| ------------- | ------------------------------------------------------------------- | -------------------------------------------------------- |
| PostgreSQL    | Simple, déjà utilisé pour l’ODS, suffisant pour un MVP, open source | Moins performant qu’un DWH spécialisé à très gros volume |
| ClickHouse    | Très performant pour les séries temporelles et l’analytique         | Technologie moins connue, complexité supplémentaire      |
| BigQuery      | Très scalable, cloud-native                                         | Coût et dépendance GCP                                   |
| Snowflake     | Très performant et cloud-native                                     | Coût, complexité, dépendance fournisseur                 |
| Azure Synapse | Très adapté à Azure                                                 | Nécessite une architecture cloud Azure                   |

## Choix retenu : PostgreSQL pour le MVP

PostgreSQL est retenu également pour le Data Warehouse du MVP, avec une séparation logique ou physique de l’ODS.

## Justification

Pour un projet pédagogique, PostgreSQL permet de démontrer un Data Warehouse orienté métier sans multiplier inutilement les technologies.

Le Data Warehouse pourra être modélisé en étoile :

```text
fact_air_quality
dim_city
dim_station
dim_time
dim_pollutant
```

Ces tables seront alimentées à partir des données propres présentes dans l’ODS, via les transformations analytiques dbt / SQL.

Ce choix permet de répondre à l’exigence de création d’un entrepôt unique à partir du référentiel établi, tout en restant réaliste pour une équipe projet et un MVP .

# 7. Benchmark de la qualité des données

## Technologies comparées

| Technologie        | Avantages                                            | Limites                                |
| ------------------ | ---------------------------------------------------- | -------------------------------------- |
| Python checks      | Simple, flexible, rapide à mettre en place           | Documentation manuelle nécessaire      |
| dbt tests          | Intégré au processus de transformation, versionnable | Principalement orienté tests SQL       |
| Great Expectations | Très complet, documentation automatique              | Mise en place plus lourde              |
| Soda               | Bon outil de data quality                            | Technologie supplémentaire à maîtriser |

## Choix retenu : Python checks + dbt tests

La solution retenue combine :

- des contrôles Python lors de l’extraction et des transformations techniques ;
- des tests dbt lors de la transformation analytique.

## Contrôles prévus

- `status = ok` pour les APIs ;
- `aqi != "-"` ou conversion en `NULL` ;
- fraîcheur de la donnée ;
- absence de doublons ;
- cohérence des timestamps ;
- détection de valeurs aberrantes ;
- présence d’une station active ;
- nombre de mesures attendues par ville.

## Justification

Le cahier des charges demande explicitement une surveillance de la qualité des données, un éventuel nettoyage, et des alertes en cas de problème sur le pipeline ou la disponibilité des données .

# 8. Benchmark de l’alerting et du monitoring

## Technologies comparées

| Technologie            | Avantages                               | Limites                      |
| ---------------------- | --------------------------------------- | ---------------------------- |
| Airflow logs + alertes | Intégré à l’orchestration, simple       | Monitoring système limité    |
| Grafana + Prometheus   | Très bon monitoring technique           | Configuration supplémentaire |
| ELK Stack              | Très puissant pour les logs             | Lourd pour un MVP            |
| Sentry                 | Très bon suivi des erreurs applicatives | Moins orienté pipeline data  |
| Webhook Discord/Slack  | Simple et efficace pour l’équipe        | Moins institutionnel         |

## Choix retenu : Airflow logs + alertes + webhook

Pour le MVP, le monitoring repose sur :

- logs Airflow ;
- retries Airflow ;
- alertes email ou webhook ;
- tableau de suivi des échecs.

## Justification

Cette approche est suffisante pour démontrer :

- la supervision du pipeline ;
- l’identification des erreurs ;
- l’alerte en cas de problème ;
- la capacité de l’équipe à réagir.

# 9. Benchmark de la data visualisation

## Technologies comparées

| Technologie     | Avantages                                                    | Limites                                 |
| --------------- | ------------------------------------------------------------ | --------------------------------------- |
| Metabase        | Simple, rapide, open source, adapté aux utilisateurs métiers | Moins avancé que Power BI               |
| Apache Superset | Open source, puissant, adapté aux dashboards analytiques     | Plus complexe à configurer              |
| Power BI        | Très complet et professionnel                                | Moins open source, dépendance Microsoft |
| Grafana         | Très bon pour séries temporelles et monitoring               | Moins orienté BI métier classique       |

## Choix retenu : Metabase ou Superset

Deux options sont retenues :

- Metabase pour un MVP simple et rapide ;
- Superset si l’équipe veut une solution open source plus avancée.

## Justification

GoodAir demande que les données soient accessibles via un outil de data visualisation et exportables pour des études avancées . Le Bloc 5 attend aussi des rapports ou visualisations compréhensibles pour aider les métiers à la décision .

# 10. Benchmark sécurité

## Technologies comparées

| Besoin          | Solution MVP                                     | Solution évolutive                |
| --------------- | ------------------------------------------------ | --------------------------------- |
| Secrets API     | Variables d’environnement / `.env` non versionné | Vault / Azure Key Vault           |
| Accès BI        | Comptes Metabase/Superset                        | SSO / OAuth / Entra ID            |
| Accès base      | Comptes PostgreSQL dédiés                        | RBAC avancé                       |
| Chiffrement     | HTTPS en reverse proxy                           | TLS complet + gestion certificats |
| Localisation UE | Hébergement local ou VPS UE                      | Cloud région France/UE            |

## Choix retenu pour le MVP

- secrets stockés hors dépôt Git ;
- accès base limité par utilisateur ;
- outil BI avec authentification ;
- services déployés sur une infrastructure localisée en France ou dans l’Union Européenne ;
- reverse proxy HTTPS si exposition externe.

## Justification

Le cahier des charges impose que les traitements et entrepôts soient localisés en France ou dans l’Union Européenne et que l’accès aux données soit sécurisé .

# 13. Limites de l’architecture retenue

L’architecture retenue présente néanmoins certaines limites :

- scalabilité inférieure à une solution cloud managée ;
- sécurité à renforcer si exposition externe ;
- supervision moins avancée qu’une architecture cloud complète ;
- maintenance assurée par l’équipe projet ;
- gestion DLM à formaliser manuellement.

Ces limites sont acceptables dans le cadre d’un MVP pédagogique.

# 14. Évolution possible

À moyen terme, l’architecture pourra évoluer vers une solution cloud Azure :

```text
Airflow → Azure Data Factory
MinIO → Azure Data Lake Storage Gen2
PostgreSQL ODS → Azure SQL Database
PostgreSQL DWH → Azure Synapse Analytics
Metabase/Superset → Power BI
Secrets .env → Azure Key Vault
```

Cette trajectoire permet de présenter l’architecture actuelle comme un MVP robuste, tout en anticipant une industrialisation future.

# Conclusion

Le benchmark réalisé met en évidence qu’une architecture de pipeline basée sur des technologies open source déployées en local constitue une réponse pertinente aux exigences du projet GoodAir dans le cadre du MSPR.

Cette approche permet de couvrir l’ensemble des besoins identifiés :

- collecte de données via APIs externes ;
- stockage des données brutes dans un data lake ;
- mise en place de transformations techniques pour fiabiliser les données ;
- structuration des données dans un ODS normalisé ;
- construction d’un data warehouse orienté métier via des transformations analytiques ;
- contrôle de la qualité des données ;
- restitution via des outils de data visualisation.

Elle présente également plusieurs avantages majeurs :

- une bonne maîtrise technique par l’équipe ;
- un coût limité ;
- une forte démontrabilité auprès du jury ;
- une architecture modulaire et évolutive.

Enfin, cette architecture constitue une base solide pour une montée en charge future vers des environnements cloud ou des architectures plus industrialisées, tout en respectant les contraintes de traçabilité, de qualité des données et de conformité attendues dans le cadre du projet.
