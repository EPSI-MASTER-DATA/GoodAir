# Audit de la structure d’un retour API AQICN

## Contexte

Notre audit de structure JSON porte sur le retour de l’endpoint suivant :
[`https://api.waqi.info/feed/@5722/?token=TOKEN_API`](#retour-json-feed-5722)

Il s’agit d’un retour issu de notre endpoint cible, à savoir `feed/@station_id`. C’est cet endpoint que notre pipeline ETL interrogera toutes les heures pour historiser les mesures de qualité de l’air.

L’endpoint `search` sera audité séparément, uniquement pour constituer le référentiel initial des stations.

## Le statut

Le premier champ retourné par l’API est le champ `status`. Lorsque ce champ retourne `ok`, cela indique que l’appel est valide et que l’API a répondu correctement.

Dans notre ETL, ce champ devra être contrôlé systématiquement. Si le statut est différent de `ok`, le pipeline devra :

- journaliser l’erreur ;
- ne pas ingérer les données ;
- réessayer l’appel selon une stratégie de retry ;
- lever une alerte si l’erreur se répète.

## Les différents blocs de données

Le retour JSON contient les blocs suivants :

```text
data
 ├── aqi
 ├── idx
 ├── attributions
 ├── city
 ├── dominentpol
 ├── iaqi
 ├── time
 ├── forecast
 └── debug
```

On remarque que les données retournées sont riches, mais partiellement hétérogènes.
La structure globale est stable, mais certains champs métier peuvent varier selon les stations.

## Audit des données métier

### AQI global

Le champ `aqi` vaut `38`. Cette valeur semble cohérente et exploitable directement comme indicateur global de qualité de l’air.

À première vue, si on compare cette valeur au champ `pm25`, qui vaut `61`, on pourrait penser qu’il existe une incohérence. En réalité, cela signifie que l’AQI global est déjà calculé par la source selon sa propre méthodologie.

Il ne faut donc pas recalculer l’AQI, mais le conserver comme KPI principal.

### Polluant dominant

Le champ `dominentpol` indique le polluant dominant. Dans notre exemple, sa valeur est `o3`.

Ce champ est intéressant à récupérer, car il permet d’identifier le polluant qui contribue le plus à la dégradation de la qualité de l’air d’une ville à un instant donné.

### Polluants détaillés

Les champs présents dans le bloc `iaqi` contiennent les mesures détaillées des polluants :

- `pm25`
- `pm10`
- `no2`
- `o3`
- `co`
- `so2`

Ces données semblent cohérentes et exploitables.

Cependant, ces champs sont dynamiques et peuvent être absents ou incomplets selon les stations.

Ils doivent donc être :

- modélisés comme `NULLABLE` ;
- parsés dynamiquement lors de l’ingestion.

### Données météo

L’endpoint retourne également des données météorologiques :

- température (`t`)
- humidité (`h`)
- pression (`p`)
- vent (`w`)

Dans le cadre de ce projet, nous disposons déjà d’une source dédiée à la météo avec OpenWeatherMap.

Il n’est donc pas nécessaire d’exploiter ces données dans le MVP.

Cela permet d’éviter :

- les redondances ;
- les incohérences entre sources ;
- une complexification inutile du modèle de données.

### Données de localisation

Les données de localisation sont présentes dans le bloc `city` :

- nom de la ville ;
- coordonnées géographiques.

Ces données sont utiles pour constituer le référentiel des stations.

Cependant, dans le cadre de l’ingestion horaire, elles n’ont pas vocation à être stockées à chaque appel, car elles seront déjà présentes dans une table dédiée aux stations.

### Données temporelles

Plusieurs champs relatifs au temps sont présents dans la réponse.

Le champ le plus pertinent à conserver est `time.iso`, car il représente la date et l’heure d’observation de la mesure.

Il est recommandé de distinguer deux champs :

- `observed_at` : issu de `time.iso` (date de mesure)
- `collected_at` : généré par le système (`NOW()`) au moment de l’ingestion

Cette distinction permet :

- d’assurer la traçabilité des données ;
- de détecter les retards de collecte ;
- d’améliorer le data lineage.

### Données de prévision

Le bloc `forecast` contient des données prévisionnelles sur plusieurs jours.

Ces données présentent un intérêt pour des analyses avancées ou des modèles prédictifs.

Cependant, elles ne sont pas nécessaires pour le MVP.

Elles pourront être intégrées dans une itération ultérieure.

### Données d’attribution

Le bloc `attributions` contient les sources des données, comme :

- AirParif
- European Environment Agency
- World Air Quality Index Project

Ces données sont importantes pour :

- la traçabilité ;
- la transparence des sources ;
- la crédibilité scientifique des analyses.

Elles doivent être conservées dans le système.

## Conclusion de l’audit

L’API AQICN fournit des données riches et exploitables pour le projet GoodAir.

Cependant, plusieurs points de vigilance ont été identifiés :

- la structure des champs métier est partiellement dynamique ;
- certains champs peuvent être absents selon les stations ;
- les données météo sont redondantes avec une autre source ;
- plusieurs champs temporels existent et nécessitent une sélection rigoureuse.

En conséquence, le pipeline ETL devra être conçu de manière robuste, avec :

- un parsing dynamique des données ;
- une gestion des valeurs nulles ;
- une séparation claire entre les données observées et collectées ;
- une conservation des sources pour assurer la traçabilité.

Cette approche permettra de garantir une intégration fiable et évolutive des données dans la plateforme Big Data.

<a id="retour-json-feed-5722"></a>
## Retour JSON de l'appel `https://api.waqi.info/feed/@5722/?token=TOKEN_API`

```json
{
  "status": "ok",
  "data": {
    "aqi": 38,
    "idx": 5722,
    "attributions": [
      {
        "url": "https://www.airparif.asso.fr/",
        "name": "AirParif - Association de surveillance de la qualité de l'air en Île-de-France",
        "logo": "Paris-Air-Parif.png"
      },
      {
        "url": "http://www.eea.europa.eu/themes/air/",
        "name": "European Environment Agency",
        "logo": "Europe-EEA.png"
      },
      {
        "url": "https://waqi.info/",
        "name": "World Air Quality Index Project"
      }
    ],
    "city": {
      "geo": [48.856614, 2.3522219],
      "name": "Paris",
      "url": "https://aqicn.org/city/paris",
      "location": ""
    },
    "dominentpol": "o3",
    "iaqi": {
      "co": {
        "v": 0.1
      },
      "h": {
        "v": 22.5
      },
      "no2": {
        "v": 14.4
      },
      "o3": {
        "v": 37.7
      },
      "p": {
        "v": 1023.8
      },
      "pm10": {
        "v": 19
      },
      "pm25": {
        "v": 61
      },
      "so2": {
        "v": 0.6
      },
      "t": {
        "v": 20.5
      },
      "w": {
        "v": 1.7
      }
    },
    "time": {
      "s": "2026-04-26 12:00:00",
      "tz": "+02:00",
      "v": 1777204800,
      "iso": "2026-04-26T12:00:00+02:00"
    },
    "forecast": {
      "daily": {
        "o3": [
          {
            "avg": 14,
            "day": "2026-04-26",
            "max": 24,
            "min": 5
          },
          {
            "avg": 14,
            "day": "2026-04-27",
            "max": 24,
            "min": 5
          },
          {
            "avg": 15,
            "day": "2026-04-28",
            "max": 22,
            "min": 9
          },
          {
            "avg": 19,
            "day": "2026-04-29",
            "max": 25,
            "min": 15
          },
          {
            "avg": 19,
            "day": "2026-04-30",
            "max": 23,
            "min": 14
          },
          {
            "avg": 19,
            "day": "2026-05-01",
            "max": 19,
            "min": 18
          }
        ],
        "pm10": [
          {
            "avg": 11,
            "day": "2026-04-26",
            "max": 18,
            "min": 7
          },
          {
            "avg": 14,
            "day": "2026-04-27",
            "max": 17,
            "min": 7
          },
          {
            "avg": 18,
            "day": "2026-04-28",
            "max": 24,
            "min": 11
          },
          {
            "avg": 13,
            "day": "2026-04-29",
            "max": 15,
            "min": 10
          },
          {
            "avg": 11,
            "day": "2026-04-30",
            "max": 13,
            "min": 10
          },
          {
            "avg": 11,
            "day": "2026-05-01",
            "max": 11,
            "min": 10
          }
        ],
        "pm25": [
          {
            "avg": 36,
            "day": "2026-04-26",
            "max": 58,
            "min": 20
          },
          {
            "avg": 38,
            "day": "2026-04-27",
            "max": 48,
            "min": 19
          },
          {
            "avg": 53,
            "day": "2026-04-28",
            "max": 70,
            "min": 31
          },
          {
            "avg": 41,
            "day": "2026-04-29",
            "max": 52,
            "min": 27
          },
          {
            "avg": 38,
            "day": "2026-04-30",
            "max": 43,
            "min": 31
          },
          {
            "avg": 37,
            "day": "2026-05-01",
            "max": 38,
            "min": 35
          }
        ],
        "uvi": [
          {
            "avg": 1,
            "day": "2026-04-26",
            "max": 5,
            "min": 0
          },
          {
            "avg": 1,
            "day": "2026-04-27",
            "max": 5,
            "min": 0
          },
          {
            "avg": 1,
            "day": "2026-04-28",
            "max": 5,
            "min": 0
          },
          {
            "avg": 1,
            "day": "2026-04-29",
            "max": 5,
            "min": 0
          },
          {
            "avg": 1,
            "day": "2026-04-30",
            "max": 5,
            "min": 0
          },
          {
            "avg": 0,
            "day": "2026-05-01",
            "max": 0,
            "min": 0
          }
        ]
      }
    },
    "debug": {
      "sync": "2026-04-26T21:43:21+09:00"
    }
  }
}
```
