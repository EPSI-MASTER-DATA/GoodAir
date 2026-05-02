# Proposition de modélisation après audit des endpoints AQICN

## Contexte

Suite à l’audit des deux endpoints AQICN utilisés dans le projet GoodAir, deux types de données ont été identifiés :

- les données référentielles, issues de l’endpoint `search` ;
- les données de mesure, issues de l’endpoint `feed/@station_id`.

L’endpoint `search` sert à identifier et sélectionner les stations à suivre.
L’endpoint `feed/@station_id` sert ensuite à historiser les mesures de qualité de l’air pour les stations validées.

# 1. Référentiel des stations

## Table `station`

La table `station` constitue le référentiel des stations AQICN retenues pour le projet.

Elle est alimentée à partir de l’endpoint `search`, après une phase de filtrage, de scoring et de validation humaine.

```text
station
```

| Champ            | Description                                            |
| ---------------- | ------------------------------------------------------ |
| id (PK)          | Identifiant interne de la station                      |
| aqicn_station_id | Identifiant externe AQICN (`uid`)                      |
| city_name        | Ville cible suivie dans le projet                      |
| station_name     | Nom de la station retourné par AQICN                   |
| latitude         | Latitude de la station                                 |
| longitude        | Longitude de la station                                |
| country          | Code pays, nullable                                    |
| aqicn_url        | URL ou identifiant URL AQICN                           |
| last_known_aqi   | Dernier AQI connu au moment de la sélection            |
| last_seen_at     | Date de la dernière mesure connue lors de la sélection |
| is_active        | Indique si la station est utilisée par le pipeline     |
| created_at       | Date d’insertion en base                               |
| updated_at       | Date de dernière mise à jour                           |

## Justification

Cette table permet de ne pas stocker les identifiants de stations “en dur” dans le code.

En production, le pipeline interrogera uniquement les stations actives :

```sql
SELECT aqicn_station_id
FROM station
WHERE is_active = true;
```

Puis il appellera l’endpoint :

```text
feed/@station_id
```

# 2. Mesures de qualité de l’air

## Table `air_quality_measurement`

Cette table permet de stocker les mesures de qualité de l’air dans le temps.

```text
air_quality_measurement
```

| Champ           | Description                       |
| --------------- | --------------------------------- |
| id (PK)         | Identifiant unique de la mesure   |
| station_id (FK) | Référence vers la table `station` |
| aqi             | Indice global de qualité de l’air |
| dominentpol     | Polluant dominant                 |
| observed_at     | Date de mesure fournie par l’API  |
| collected_at    | Date de collecte par le pipeline  |
| created_at      | Date d’insertion en base          |

# 3. Polluants détaillés

## Table `pollution_measure`

Les polluants étant dynamiques et variables selon les stations, ils sont stockés dans une table dédiée.

```text
pollution_measure
```

| Champ               | Description                                          |
| ------------------- | ---------------------------------------------------- |
| id (PK)             | Identifiant unique                                   |
| measurement_id (FK) | Référence vers `air_quality_measurement`             |
| pollutant           | Type de polluant (`pm25`, `pm10`, `no2`, `o3`, etc.) |
| value               | Valeur mesurée                                       |
| unit                | Unité de mesure, optionnelle                         |

Cette structure permet d’ajouter ou retirer des polluants sans modifier le schéma de base de données.

# 4. Sources des données

## Table `data_source`

Cette table stocke les sources des données issues du bloc `attributions`.

```text
data_source
```

| Champ      | Description        |
| ---------- | ------------------ |
| id (PK)    | Identifiant unique |
| name       | Nom de la source   |
| url        | URL de la source   |
| logo       | Logo éventuel      |
| created_at | Date d’insertion   |

## Table `measurement_source`

Une mesure pouvant être rattachée à plusieurs sources, une table de liaison est nécessaire.

```text
measurement_source
```

| Champ               | Description              |
| ------------------- | ------------------------ |
| measurement_id (FK) | Référence vers la mesure |
| source_id (FK)      | Référence vers la source |

# 5. Données brutes

## Table `raw_data`

Cette table permet de conserver les données JSON brutes retournées par l’API.

```text
raw_data
```

| Champ               | Description              |
| ------------------- | ------------------------ |
| id (PK)             | Identifiant unique       |
| measurement_id (FK) | Référence vers la mesure |
| raw_json            | Données JSON complètes   |
| created_at          | Date d’insertion         |

Cette table est utile pour :

- assurer la traçabilité ;
- faciliter le debugging ;
- permettre une ré-exploitation future des données.

# 6. Schéma relationnel global

```text
station
   │
   └── air_quality_measurement
           │
           ├── pollution_measure
           ├── measurement_source ─── data_source
           └── raw_data
```

# 7. Conclusion

La modélisation proposée distingue clairement :

- le référentiel des stations ;
- les mesures historisées ;
- les polluants détaillés ;
- les sources ;
- les données brutes.

Cette approche permet de construire un modèle normalisé, évolutif et traçable, adapté à l’ingestion horaire des données AQICN dans le cadre du projet GoodAir.
