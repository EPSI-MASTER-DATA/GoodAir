# Identification des composants du pipeline de données

## Contexte

Dans le cadre du projet GoodAir, un pipeline de données doit être mis en place afin de collecter, transformer, stocker et restituer les données issues des APIs externes (AQICN et OpenWeatherMap).

L’objectif de ce ticket est d’identifier les différents composants nécessaires à la construction de ce pipeline, ainsi que leur rôle dans l’architecture globale, tout en proposant plusieurs technologies possibles pour chaque brique.

## Vue d’ensemble du pipeline

```text
API AQICN / OpenWeather
        ↓
Orchestrateur
        ↓https
Extracteurs API
        ↓
Zone Raw / Data Lake
        ↓
Transformation technique
        ↓
ODS (Base normalisée)
        ↓
Transformation analytique
        ↓
Data Warehouse
        ↓
Contrôle qualité + alerting
        ↓
Data Viz / restitution
```

## 1. Sources de données

### Rôle

Fournir les données nécessaires au projet.

### Composants

- API AQICN / WAQI : données de qualité de l’air ;
- API OpenWeatherMap : données météorologiques ;
- table `city` : référentiel des villes ;
- table `aq_station` : référentiel des stations AQICN.
- table `wf_station` : référentiel des stations OpenWeather.

## 2. Orchestrateur

### Rôle

Planifier et superviser les traitements du pipeline.

### Technologies possibles

- Apache Airflow
- Prefect
- Dagster
- Cron

### Responsabilités

- planification des jobs (ex : toutes les heures) ;
- gestion des retries ;
- centralisation des logs ;
- gestion des dépendances entre tâches ;
- alerting.

## 3. Extracteurs API

### Rôle

Récupérer les données depuis les APIs externes.

### Technologies possibles

- Python (requests, httpx)
- Node.js (axios, fetch)
- Bash + curl (MVP simple)

### Responsabilités

- appeler les différents endpoints ;
- vérifier le champ `status` de l'endpoint `feed/@station_id`;
- gérer les quotas API ;
- récupérer les données au format JSON ;
- transmettre les données vers le stockage brut.

## 4. Data Lake (implémentation de la couche Bronze)

### Rôle

Stocker les données brutes sans transformation.

### Technologies possibles

- MinIO
- Amazon S3
- Google Cloud Storage
- Azure Blob Storage
- HDFS

### Organisation

```text
/raw/aqicn/YYYY/MM/DD/HH/
/raw/openweather/YYYY/MM/DD/HH/
```

### Intérêt

- traçabilité ;
- audit ;
- rejeu des traitements ;
- conservation du JSON original.

## 5. Transformation technique (Bronze → Silver)

### Rôle

Transformer les données brutes issues du Data Lake en données propres, cohérentes et structurées avant leur stockage dans l’ODS.

Cette couche réalise les traitements techniques nécessaires à la fiabilisation des données.

### Technologies possibles

- Python (recommandé)
- Apache Spark / PySpark
- dbt (cas simple)
- SQL intermédiaire

### Traitements réalisés

- parsing des données JSON ;
- extraction des champs utiles ;
- conversion des types (`string → int`, gestion des `NULL`) ;
- nettoyage des données (`"-"` → `NULL`) ;
- normalisation des timestamps ;
- gestion des données dynamiques (polluants variables) ;
- transformation des structures imbriquées en tables relationnelles ;
- déduplication des données ;
- contrôle de cohérence des enregistrements.

### Responsabilités

- transformer les données brutes en données exploitables ;
- garantir la qualité des données avant insertion dans l’ODS ;
- préparer une structure stable pour la couche Silver ;
- isoler les traitements techniques des traitements analytiques.

## 6. ODS — Base de données normalisée (implémentation de la couche Silver)

### Rôle

Stocker les données nettoyées et structurées issues du datalake. l'ODS ne réalise pas les traitements que sont la transformation mais constitue la couche de stockage des données déjà nettoyées et structurées.

### Technologies possibles

- PostgreSQL
- MySQL / MariaDB
- SQL Server
- MongoDB

### Responsabilités

- garantir que les données stockées sont nettoyées et cohérentes ;
- assurer la normalisation du schéma de données ;
- gérer les données dynamiques (polluants variables) ;
- historiser les données dans le temps ;
- stocker les données techniques issues des traitements.

## 7. Transformation analytique (Silver → Gold)

### Rôle

Préparer les données pour le Data Warehouse.

### Technologies possibles

- Python
- Apache Spark
- Hadoop
- dbt
- PySpark

### Traitements réalisés

- agrégation ;
- pivot des données ;
- enrichissement ;
- préparation des faits et dimensions.

## 8. Data Warehouse (implémentation de la couche Gold)

### Rôle

Stocker les données dans un modèle orienté métier.

### Technologies possibles

- PostgreSQL
- Snowflake
- Google BigQuery
- Amazon Redshift
- ClickHouse

### Modèle

- schéma en étoile (star schema).

### Tables typiques

- `fact_air_quality`
- `dim_city`
- `dim_station`
- `dim_time`
- `dim_pollutant`

## 9. Contrôle qualité

### Rôle

Garantir la fiabilité des données.

### Technologies possibles

- Python (assertions, scripts)
- Great Expectations
- Soda
- dbt tests

### Contrôles

- validité API ;
- cohérence des données ;
- fraîcheur ;
- valeurs aberrantes ;
- doublons.

## 10. Alerting et monitoring

### Rôle

Surveiller le pipeline et alerter.

### Technologies possibles

- Airflow alerts
- Grafana + Prometheus
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Sentry
- Slack / Discord webhooks

### Cas d’alerte

- API down ;
- quota dépassé ;
- pipeline en échec ;
- données manquantes.

## 11. Data Visualisation et restitution

### Rôle

Restituer les données aux utilisateurs métiers.

### Technologies possibles

- Metabase (recommandé)
- Apache Superset
- Power BI
- Tableau
- Grafana

### Fonctionnalités

- dashboards ;
- analyses temporelles ;
- comparaisons ;
- export ;
- alertes métier.

## Conclusion

Le pipeline GoodAir repose sur une architecture modulaire et évolutive composée de plusieurs briques techniques.

Chaque composant peut être implémenté avec différentes technologies, selon :

- les contraintes du projet ;
- le niveau de complexité souhaité ;
- les ressources disponibles.

Cette approche permet de construire un système :

- robuste ;
- scalable ;
- maintenable ;
- adapté à une plateforme Big Data moderne.
