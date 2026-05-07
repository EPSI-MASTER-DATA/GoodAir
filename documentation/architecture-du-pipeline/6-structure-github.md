# Structure du dépôt GitHub — Projet GoodAir

## Principe général

Le dépôt GitHub contient **tout ce qui se déploie et se versionne** : le code, la configuration, l'infrastructure et la documentation. Il ne contient jamais de données ni de secrets.

Ce qui vit ailleurs :

| Élément                        | Où                                  |
| ------------------------------ | ----------------------------------- |
| Données Bronze / Silver / Gold | ADLS Gen2                           |
| Référentiel des stations       | ADLS Gen2 `/reference/`             |
| Clés API, secrets, credentials | Azure Key Vault                     |
| State Terraform                | ADLS Gen2 (backend distant dédié)   |
| Dashboards Power BI            | Service Power BI (gestion manuelle) |

## Arborescence complète MVP

```text
goodair/
│
├── .github/
│   └── workflows/
│       ├── deploy-infra.yml
│       └── quality-check.yml
│
├── infra/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── backend.tf
│   └── modules/
│       ├── adls/
│       ├── databricks/
│       ├── synapse/
│       ├── functions/
│       ├── adf/
│       ├── keyvault/
│       └── monitoring/
│
├── functions/
│   ├── extract_aqicn/
│   │   ├── __init__.py
│   │   └── function.json
│   ├── extract_openweather/
│   │   ├── __init__.py
│   │   └── function.json
│   ├── shared/
│   │   ├── adls_client.py
│   │   ├── keyvault_client.py
│   │   └── http_client.py
│   ├── requirements.txt
│   └── host.json
│
├── databricks/
│   ├── jobs/
│   │   ├── bronze_to_silver.py
│   │   └── silver_to_gold.py
│   ├── shared/
│   │   ├── delta_utils.py
│   │   └── quality_checks.py
│   └── config/
│       ├── job_bronze_to_silver.json
│       └── job_silver_to_gold.json
│
├── adf/
│   └── pipelines/
│       ├── pl_ingest_hourly.json
│       ├── pl_bronze_to_silver.json
│       └── pl_silver_to_gold.json
│
├── reference/ (à mettre à jour en fonction de l'audit OpenWeather si nécessaire)
│   ├── bootstrap_stations.py
│   └── stations_schema.json
│
├── monitoring/
│   └── grafana/
│       ├── dashboards/
│       │   ├── pipeline_overview.json
│       │   └── data_quality.json
│       └── provisioning/
│           └── datasources.yml
│
├── tests/
│   ├── functions/
│   │   ├── test_extract_aqicn.py
│   │   └── test_extract_openweather.py
│   └── databricks/
│       ├── test_bronze_to_silver.py
│       └── test_silver_to_gold.py
│
├── documentation/
│
├── .env.example
├── .gitignore
└── README.md
```

## Détail de chaque dossier

### `.github/workflows/`

Contient les pipelines GitHub Actions. Chaque fichier correspond à un périmètre de déploiement distinct.

| Fichier             | Déclencheur      | Ce qu'il fait                                                    |
| ------------------- | ---------------- | ---------------------------------------------------------------- |
| `deploy-infra.yml`  | Push sur `main`  | Terraform plan + apply — provisionne toutes les ressources Azure |
| `quality-check.yml` | PR sur `develop` | Tests unitaires, lint Python, validation des schémas             |

Les secrets nécessaires aux workflows (credentials Azure, tenant ID, client ID) sont stockés dans les **GitHub Actions Secrets**, jamais dans le code.

### `infra/`

Code Terraform pour provisionner l'ensemble de l'infrastructure Azure.

| Fichier / dossier     | Contenu                                                    |
| --------------------- | ---------------------------------------------------------- |
| `main.tf`             | Point d'entrée Terraform, appel des modules                |
| `variables.tf`        | Variables paramétrables (région, noms, tailles)            |
| `outputs.tf`          | Valeurs exportées après provisioning (URLs, IDs)           |
| `backend.tf`          | Configuration du backend distant (ADLS Gen2) pour le state |
| `modules/adls/`       | Compte ADLS Gen2, containers, ACL                          |
| `modules/databricks/` | Workspace Databricks, clusters, configuration              |
| `modules/synapse/`    | Workspace Synapse, Serverless SQL                          |
| `modules/functions/`  | Plan Azure Functions, application, identity managée        |
| `modules/adf/`        | Data Factory, linked services                              |
| `modules/keyvault/`   | Key Vault, politiques d'accès                              |
| `modules/monitoring/` | Container App Grafana, OpenTelemetry collector             |

### `functions/`

Code Python des Azure Functions. Chaque fonction est isolée dans son propre dossier.

| Dossier                     | Rôle                                                                                                                                                         |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `extract_aqicn/`            | Lit les `station_id` actifs depuis `/reference/stations.json` sur ADLS, appelle `feed/@station_id` pour chaque station, écrit les JSON bruts sur ADLS Bronze |
| `extract_openweather/`      | Appelle l'API OpenWeatherMap pour chaque ville, écrit les JSON bruts sur ADLS Bronze                                                                         |
| `shared/adls_client.py`     | Client ADLS Gen2 réutilisable (authentification Managed Identity)                                                                                            |
| `shared/keyvault_client.py` | Lecture des secrets depuis Key Vault                                                                                                                         |
| `shared/http_client.py`     | Client HTTP avec retry et gestion des quotas                                                                                                                 |

La liste des `station_id` à collecter n'est **jamais codée en dur** dans les fonctions. Elle est lue dynamiquement depuis `/reference/stations.json` sur ADLS à chaque exécution.

### `databricks/`

Jobs Python Spark pour les transformations Bronze → Silver → Gold.

| Fichier                    | Rôle                                                                                                                                  |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `jobs/bronze_to_silver.py` | Parsing JSON, nettoyage (`"-"` → NULL), normalisation timestamps, gestion polluants dynamiques, déduplication, écriture Delta Silver  |
| `jobs/silver_to_gold.py`   | Agrégations, construction du modèle en étoile (fact_air_quality, dim_city, dim_station, dim_time, dim_pollutant), écriture Delta Gold |
| `shared/delta_utils.py`    | Fonctions utilitaires Delta Lake (upsert, time travel, compaction)                                                                    |
| `shared/quality_checks.py` | Règles de qualité des données (fraîcheur, nullité, doublons, plages de valeurs), quarantaine des enregistrements invalides            |
| `config/*.json`            | Configuration des jobs Databricks (cluster, schedule, paramètres) déployée via GitHub Actions                                         |

### `adf/`

Pipelines ADF exportés au format JSON et versionnés dans Git. ADF supporte nativement la synchronisation Git — les pipelines sont édités dans l'interface ADF et committé automatiquement dans ce dossier.

| Pipeline                   | Rôle                                                                                                                                  |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `pl_ingest_hourly.json`    | Trigger horaire : déclenche en parallèle extract_aqicn et extract_openweather, attend la fin, enchaîne Bronze→Silver puis Silver→Gold |
| `pl_bronze_to_silver.json` | Déclenche le job Databricks Bronze→Silver, surveille l'exécution                                                                      |
| `pl_silver_to_gold.json`   | Déclenche le job Databricks Silver→Gold, surveille l'exécution                                                                        |

### `reference/`

Scripts de récupération du référentiel des stations. Ce dossier ne contient pas les données elles-mêmes (qui vivent sur ADLS) mais les scripts qui les produisent.

| Fichier                 | Rôle                                                                                                                                                                                    |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bootstrap_stations.py` | Appelle l'endpoint `search` de l'API WAQI, filtre les stations par critères géographiques et qualité, produit le fichier `stations.json` à uploader manuellement sur ADLS `/reference/` |
| `stations_schema.json`  | Schéma JSON attendu pour le fichier de référentiel (station_id, status, city, coordinates)                                                                                              |

Ce script est exécuté **une seule fois** au démarrage du projet, puis lors de mises à jour occasionnelles du référentiel des stations.

### `monitoring/`

Dashboards Grafana et configuration OpenTelemetry, versionnés dans Git.

| Fichier                                     | Rôle                                                                                            |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `grafana/dashboards/pipeline_overview.json` | Vue globale du pipeline : taux de succès, latence, dernière exécution par étape                 |
| `grafana/dashboards/data_quality.json`      | KPIs qualité des données : taux de nullité, doublons, fraîcheur, enregistrements en quarantaine |
| `grafana/provisioning/datasources.yml`      | Configuration des sources de données Grafana (Azure Monitor, Prometheus)                        |

Les dashboards Grafana étant des fichiers JSON, ils sont modifiables dans l'interface puis exportés et committé dans Git.

### `tests/`

Tests unitaires Python pour les Azure Functions et les jobs Databricks.

| Dossier             | Ce qui est testé                                                                                                              |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `tests/functions/`  | Parsing des réponses API, gestion des erreurs (status != ok, timeout), construction des chemins ADLS                          |
| `tests/databricks/` | Transformations Bronze→Silver (nettoyage, typage, déduplication), transformations Silver→Gold (agrégations, modèle en étoile) |

Les tests sont exécutés automatiquement par le workflow `quality-check.yml` à chaque Pull Request.

### `documentation/`

Documentation technique du projet, déjà structurée dans le dépôt.

### Fichiers racine

| Fichier        | Contenu                                                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `.env.example` | Template des variables d'environnement nécessaires au développement local (noms des ressources Azure, région, etc.) — jamais le `.env` réel |
| `.gitignore`   | Exclut `.env`, `__pycache__`, `.terraform/`, fichiers de secrets                                                                            |
| `README.md`    | Présentation du projet, prérequis, guide de démarrage rapide                                                                                |

## Ce qui ne va jamais dans le dépôt

| Élément                                | Raison                                                   |
| -------------------------------------- | -------------------------------------------------------- |
| `.env`                                 | Contient des secrets                                     |
| `terraform.tfstate`                    | State Terraform — stocké sur ADLS Gen2 (backend distant) |
| `*.json` de données Bronze/Silver/Gold | Les données vivent sur ADLS, pas dans Git                |
| Clés API, tokens, passwords            | Stockés dans Azure Key Vault                             |
| Fichiers `.pbix` Power BI              | Gérés manuellement via le service Power BI               |
