# Audit de la structure d’un retour API AQICN — Endpoint `search`

## Contexte

Notre audit de structure JSON porte sur le retour de l’endpoint suivant :
[`https://api.waqi.info/search/?keyword=paris&token=TOKEN_API`](#retour-json-search-paris)

Cet endpoint est utilisé dans le cadre du projet pour constituer le référentiel initial des stations pour la qualité de l'air. Contrairement à l’endpoint `feed/@station_id`, il n’a pas vocation à être utilisé pour l’ingestion horaire des données.

## Le statut

Comme pour les autres endpoints AQICN, le champ `status` est présent en première position.

Lorsque sa valeur est `ok`, cela indique que l’appel API a réussi.

Dans notre ETL, ce champ devra être contrôlé systématiquement afin de :

- détecter les erreurs d’appel ;
- éviter d’ingérer des données invalides ;
- déclencher des mécanismes de retry et d’alerte si nécessaire.

## Les différents blocs de données

Le retour JSON contient les éléments suivants :

```text
status
data[]
 ├── uid
 ├── aqi
 ├── time
 │    ├── tz
 │    ├── stime
 │    └── vtime
 └── station
      ├── name
      ├── geo
      ├── url
      └── country (optionnel)
```

Contrairement à l’endpoint `feed`, le champ `data` est ici un tableau contenant plusieurs stations correspondant au mot-clé recherché (ici ça sera une ville).

## Audit des données métier

### Identifiant de station

Le champ `uid` correspond à l’identifiant unique de la station dans AQICN.

Il s’agit d’un élément clé, car il sera utilisé pour interroger l’endpoint `feed/@station_id` lors de l’ingestion des données.

Ce champ doit être conservé comme clé externe dans la table `station`.

### AQI initial

Le champ `aqi` représente une valeur indicative de la qualité de l’air pour la station.

Cependant, ce champ présente plusieurs particularités :

- il est de type chaîne de caractères (`"53"`) ;
- il peut contenir la valeur `"-"` lorsque la donnée n’est pas disponible.

Il est donc nécessaire de :

- convertir la valeur en entier lorsque cela est possible ;
- considérer `"-"` comme une valeur nulle.

Ce champ peut être utilisé comme indicateur de disponibilité des données mais ne constitue pas une donnée fiable pour l’analyse.

### Données temporelles

Le bloc `time` contient les informations temporelles associées à la dernière mesure connue :

- `stime` : date et heure sous forme de chaîne ;
- `vtime` : timestamp Unix ;
- `tz` : fuseau horaire.

Ces informations permettent d’évaluer la fraîcheur des données.

Ce champ est particulièrement important pour filtrer les stations :

- les stations avec des données trop anciennes doivent être exclues du référentiel.

### Données de station

Le bloc `station` contient les informations descriptives de la station :

- `name` : nom de la station ;
- `geo` : coordonnées géographiques (latitude, longitude) ;
- `url` : identifiant URL AQICN ;
- `country` : code pays (optionnel).

Ces données sont essentielles pour construire le référentiel des stations.

## Points de vigilance

### Multiplicité des résultats

Une recherche par mot-clé retourne plusieurs stations.

Dans le cas de `paris`, on obtient :

- des stations dans Paris intra-muros ;
- des stations en périphérie (Saint-Denis, Bobigny, etc.) ;
- des stations éloignées (ex : Rouen).

Une règle de sélection est donc nécessaire.

### Données non homogènes

Certains champs sont optionnels :

- `country` n’est pas toujours présent ;
- la structure des noms de stations est variable ;
- les coordonnées sont toujours présentes mais doivent être validées.

### Données obsolètes

Certaines stations retournées contiennent des données très anciennes :

ex : 2021 ou 2025

Ces stations doivent être filtrées lors de la constitution du référentiel.

### AQI non exploitable

La valeur `aqi` peut être :

```text
"-"
```

Cela indique une absence de données.

Ces stations ne doivent pas être retenues pour l’ingestion.

## Stratégie d’utilisation de l’endpoint `search`

Comme nous l'avons dit plus haut, l’endpoint `search` ne doit pas être utilisé comme source d’ingestion automatique mais comme un outil de constitution du référentiel des stations.

Une approche **semi-automatisée** est recommandée.

### Phase 1 — Découverte automatique

L’endpoint `search` est utilisé pour récupérer une liste de stations candidates à partir d’un mot-clé (ex : une ville).

### Phase 2 — Filtrage et scoring

Les stations sont évaluées selon des critères objectifs :

- présence d’un AQI exploitable (`aqi != "-"`) ;
- fraîcheur des données (`time.stime` récent) ;
- cohérence géographique avec la ville cible ;
- présence du code pays (`country = "FR"` si disponible) ;
- cohérence du nom de station.

Un système de scoring peut être utilisé pour prioriser les stations les plus pertinentes.

### Phase 3 — Validation humaine

Les stations retenues sont validées par l’équipe projet afin de garantir leur pertinence.

Cette étape est essentielle pour éviter les erreurs liées à l’automatisation complète.

### Phase 4 — Stockage dans le référentiel

Les stations validées sont enregistrées dans une table `station` en base de données.

### Phase 5 — Exploitation en production

En production, le pipeline n’utilise plus l’endpoint `search`.

Il interroge uniquement les stations validées via l’endpoint :

```text
feed/@station_id
```

## Conclusion de l’audit

L’endpoint `search` permet d’identifier les stations associées à une ville, mais présente plusieurs limites :

- il retourne plusieurs résultats non filtrés ;
- certaines stations sont obsolètes ou hors périmètre ;
- les données sont partiellement hétérogènes.

En conséquence, cet endpoint doit être utilisé uniquement pour constituer et maintenir un référentiel de stations.

Une logique de filtrage, de scoring et de validation est indispensable afin de garantir la qualité des données. Pour chaque ville concernée par ce projet, l'idéal serait d'avoir au moins une station de fall-back en cas d'indipsonibilté temporaire.

On pourrait aussi envisager, dans le cadre d'une évolution de notre projet, de s'assurer d'avoir des données de qualité de l'air en centre ville et en périphérie.

L’ingestion des mesures sera ensuite réalisée exclusivement via l’endpoint `feed/@station_id` qui est plus stable et adapté à l’historisation.

<a id="retour-json-search-paris"></a>

## Retour JSON de l'appel `https://api.waqi.info/search/?keyword=paris&token=TOKEN_API`

```json
{
  "status": "ok",
  "data": [
    {
      "uid": 5722,
      "aqi": "53",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 18:00:00",
        "vtime": 1777392000
      },
      "station": {
        "name": "Paris",
        "geo": [48.856614, 2.3522219],
        "url": "paris"
      }
    },
    {
      "uid": 12763,
      "aqi": "62",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "Paris 1er Les Halles, Paris",
        "geo": [48.8621, 2.34462],
        "url": "france/paris/paris-1er-les-halles",
        "country": "FR"
      }
    },
    {
      "uid": 3089,
      "aqi": "53",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 18:00:00",
        "vtime": 1777392000
      },
      "station": {
        "name": "Boulevard Haussmann, Paris",
        "geo": [48.8733, 2.32957],
        "url": "france/paris/boulevard-haussmann",
        "country": "FR"
      }
    },
    {
      "uid": 3086,
      "aqi": "51",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 18:00:00",
        "vtime": 1777392000
      },
      "station": {
        "name": "Avenue Des Champs Elysees, Paris",
        "geo": [48.8686, 2.31166],
        "url": "france/paris/avenue-des-champs-elysees",
        "country": "FR"
      }
    },
    {
      "uid": 3088,
      "aqi": "48",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 18:00:00",
        "vtime": 1777392000
      },
      "station": {
        "name": "Boulevard Peripherique Est, Paris",
        "geo": [48.8386, 2.41278],
        "url": "france/paris/boulevard-peripherique-est",
        "country": "FR"
      }
    },
    {
      "uid": 3098,
      "aqi": "44",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 18:00:00",
        "vtime": 1777392000
      },
      "station": {
        "name": "Autoroute A1 - Saint-denis, Paris",
        "geo": [48.9251, 2.35654],
        "url": "france/paris/autoroute-a1-saint-denis",
        "country": "FR"
      }
    },
    {
      "uid": 3103,
      "aqi": "42",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "Vitry-sur-seine, Paris",
        "geo": [48.7759, 2.37577],
        "url": "france/paris/vitry-sur-seine",
        "country": "FR"
      }
    },
    {
      "uid": 3082,
      "aqi": "39",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "Paris 18eme, Paris",
        "geo": [48.8917, 2.34563],
        "url": "france/paris/paris-18eme",
        "country": "FR"
      }
    },
    {
      "uid": 3092,
      "aqi": "35",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 18:00:00",
        "vtime": 1777392000
      },
      "station": {
        "name": "Place De Lopera, Paris",
        "geo": [48.8704, 2.33241],
        "url": "france/paris/place-de-lopera",
        "country": "FR"
      }
    },
    {
      "uid": 3091,
      "aqi": "30",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 18:00:00",
        "vtime": 1777392000
      },
      "station": {
        "name": "Place Victor Basch, Paris",
        "geo": [48.8277, 2.3267],
        "url": "france/paris/place-victor-basch",
        "country": "FR"
      }
    },
    {
      "uid": 6937,
      "aqi": "25",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "Paris Stade Lenglen, Paris",
        "geo": [48.8303, 2.26972],
        "url": "france/paris/paris-stade-lenglen",
        "country": "FR"
      }
    },
    {
      "uid": 3100,
      "aqi": "-",
      "time": {
        "tz": "+02:00",
        "stime": "2025-10-29 10:00:00",
        "vtime": 1761724800
      },
      "station": {
        "name": "Route Nationale 2 - Pantin, Paris",
        "geo": [48.9022, 2.3907],
        "url": "france/paris/route-nationale-2-pantin",
        "country": "FR"
      }
    },
    {
      "uid": 3090,
      "aqi": "-",
      "time": {
        "tz": "+02:00",
        "stime": "2021-02-26 19:00:00",
        "vtime": 1614358800
      },
      "station": {
        "name": "Paris Centre, Paris",
        "geo": [48.8593, 2.35101],
        "url": "france/paris/paris-centre",
        "country": "FR"
      }
    },
    {
      "uid": 10946,
      "aqi": "38",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 20:00:00",
        "vtime": 1777399200
      },
      "station": {
        "name": ", Quai De Paris - Trafic, Rouen, France",
        "geo": [49.4366914897979, 1.09855494496119],
        "url": "france/rouen/--quai-de-paris-trafic"
      }
    },
    {
      "uid": 3093,
      "aqi": "42",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "Lognes, Paris",
        "geo": [48.8403, 2.63463],
        "url": "france/paris/lognes"
      }
    },
    {
      "uid": 3099,
      "aqi": "37",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "Bobigny, Paris",
        "geo": [48.9024, 2.45261],
        "url": "france/paris/bobigny"
      }
    },
    {
      "uid": 3097,
      "aqi": "53",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "La Defense, Paris",
        "geo": [48.8913, 2.24064],
        "url": "france/paris/la-defense"
      }
    },
    {
      "uid": 3105,
      "aqi": "15",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 18:00:00",
        "vtime": 1777392000
      },
      "station": {
        "name": "Gonesse, Paris",
        "geo": [48.9908, 2.44461],
        "url": "france/paris/gonesse"
      }
    },
    {
      "uid": 12762,
      "aqi": "44",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "Rambouillet, Paris",
        "geo": [48.6337, 1.83043],
        "url": "france/paris/rambouillet"
      }
    },
    {
      "uid": 3085,
      "aqi": "35",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "Gennevilliers, Paris",
        "geo": [48.9302, 2.29421],
        "url": "france/paris/gennevilliers"
      }
    },
    {
      "uid": 3104,
      "aqi": "41",
      "time": {
        "tz": "+02:00",
        "stime": "2026-04-28 19:00:00",
        "vtime": 1777395600
      },
      "station": {
        "name": "Cergy-pontoise, Paris",
        "geo": [49.0459, 2.04104],
        "url": "france/paris/cergy-pontoise"
      }
    }
  ]
}
```
