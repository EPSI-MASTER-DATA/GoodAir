# Proposition de modélisation après audit des endpoints AQICN

## Contexte

Suite à l’audit des deux endpoints AQICN utilisés dans le projet GoodAir, deux types de données ont été distingués :

- les données référentielles, issues de l’endpoint `search` ;
- les données de mesure, issues de l’endpoint `feed/@station_id`.

L’endpoint `search` permet d’identifier et de sélectionner les stations à suivre.  
L’endpoint `feed/@station_id` permet ensuite d’historiser les mesures de qualité de l’air pour les stations validées.

# 1. Référentiel des villes et des stations

## Table `station`

La table `station` constitue le référentiel des stations AQICN retenues pour le projet.

Elle est alimentée à partir de l’endpoint `search`, après une phase de filtrage, de scoring et de validation humaine.

```text
station
```

| Champ            | Description                                             |
| ---------------- | ------------------------------------------------------- |
| id (PK)          | Identifiant interne de la station                       |
| city_id (FK)     | Ville cible du projet (`city.id`)                       |
| aqicn_station_id | Identifiant externe AQICN (`uid`)                       |
| station_name     | Nom de la station retourné par AQICN                    |
| latitude         | Latitude de la station                                  |
| longitude        | Longitude de la station                                 |
| country          | Code pays, nullable                                     |
| aqicn_url        | URL ou identifiant URL AQICN                            |
| last_seen_at     | Date de la dernière mesure connue lors de la sélection  |
| status           | Indique comment la station est utilisée par le pipeline |
| created_at       | Date d’insertion en base                                |
| updated_at       | Date de dernière mise à jour                            |

**Précisions concernant le champ `status` :**

Le champ `status` sert à gérer le cycle de vie des stations. Plutôt que de supprimer les stations qui ne sont plus utilisées, on les conserve en base avec un statut explicite, ce qui améliore la gouvernance des données et la flexibilité du pipeline.

Ainsi, une station peut être :

- ACTIVE : station utilisée pour l’ingestion principale ;
- FALLBACK : station utilisée en cas de défaillance d’une station principale ;
- INACTIVE : station temporairement désactivée ;
- DEPRECATED : station obsolète conservée à des fins historiques.

## Justification

Cette table évite de figer les identifiants AQICN des stations dans le code ou la configuration sans traçabilité.

En production, le pipeline n’interroge que les stations prises en charge pour l’ingestion (statuts métier pertinents) :

```sql
SELECT aqicn_station_id
FROM station
WHERE status IN ('ACTIVE', 'FALLBACK');
```

Les statuts `INACTIVE` et `DEPRECATED` restent stockés mais sont exclus du cycle d’ingestion courant.

Ensuite le pipeline appellera l’endpoint :

```text
feed/@station_id
```

## La table `city`

La table `city` constitue le référentiel des villes suivies dans le cadre du projet GoodAir.

Contrairement aux stations, qui peuvent être multiples pour une même ville, cette table permet de définir un point de référence unique pour chaque ville cible. Elle est utilisée notamment pour :

- structurer les données métier ;
- centraliser les informations géographiques de référence ;
- faciliter la sélection des stations pertinentes (via la distance) ;
- préparer l’intégration avec d’autres sources de données (ex : OpenWeatherMap).

```text
city
```

| Champ      | Description                                         |
| ---------- | --------------------------------------------------- |
| id (PK)    | Identifiant interne de la ville                     |
| name       | Nom de la ville cible (ex : Paris, Lyon, Marseille) |
| country    | Pays de la ville (ex : FR)                          |
| latitude   | Latitude du centre-ville                            |
| longitude  | Longitude du centre-ville                           |
| created_at | Date d’insertion en base                            |
| updated_at | Date de dernière mise à jour                        |

## Justifications

Une table `city` vise à représenter explicitement les **villes cibles du projet**, et non uniquement les points de mesure renvoyés par AQICN. Cela évite d’encoder la notion de périmètre urbain sous forme de chaînes de caractères ou de duplications dans chaque ligne `station`.

Sur le plan relationnel, une ville fait office d’entité pivot : **plusieurs stations** (principale, secondaire, secours ou candidates issues de `search`) **se rattachent à une seule ville**, ce qui permet l’agrégation, le filtrage géographique et une cohérence métier stable (par exemple scoring ou distance par rapport au centroïde défini dans `city`).

```text
city (1) → (N) station
```

Métier et normalisation sont alignés : la **station** est le lieu instrumenté de mesure ; la **ville** est la zone analytique suivie dans GoodAir (nom, pays, centroïde géographique de référence). Séparer ces deux niveaux évite les redondances, clarifie les clés étrangères (`station.city_id`) et facilite l’évolution du référentiel lorsque les stations changent alors que les villes cibles restent stables.

# 2. Mesures de qualité de l’air

## Table `air_quality_measurement`

Cette table permet de stocker les mesures de qualité de l’air dans le temps.

```text
air_quality_measurement
```

| Champ           | Description                                  |
| --------------- | -------------------------------------------- |
| id (PK)         | Identifiant unique de la mesure              |
| station_id (FK) | Référence vers la table `station`            |
| aqi             | Indice global de qualité de l’air            |
| dominentpol     | Polluant dominant (nom repris du JSON AQICN) |
| observed_at     | Date de mesure fournie par l’API             |
| collected_at    | Date de collecte par le pipeline             |
| created_at      | Date d’insertion en base                     |

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

Cette table stocke les sources issues du bloc `attributions` du retour API.

```text
data_source
```

| Champ      | Description        |
| ---------- | ------------------ |
| id (PK)    | Identifiant unique |
| name       | Nom de la source   |
| url        | URL de la source   |
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
city
 │
 └── station
         │
         └── air_quality_measurement
                 │
                 ├── pollution_measure
                 ├── measurement_source ─── data_source
                 └── raw_data
```

# 7. Conclusion

La modélisation proposée distingue clairement :

- le référentiel des villes puis des stations ;
- les mesures historisées ;
- les polluants détaillés ;
- les sources ;
- les données brutes.

Elle permet de construire un modèle normalisé, évolutif et traçable, adapté à l’ingestion horaire des données AQICN dans le cadre du projet GoodAir.
