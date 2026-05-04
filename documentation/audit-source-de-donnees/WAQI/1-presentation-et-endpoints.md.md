# L'API AQICN / WAQI - World Air Quality Index API

## Présentation

**L'API AQICN** sert à récupérer des données de qualité de l'air en temps réel pour différentes villes ou stations. Elle répond directement à un besoin de Good Air puisqu'elle **permettra de suivre la qualité de l'air dans les principales villes de France et d'historiciser les mesures chaque heure**.

En se basant strictement sur la documentation, c'est une **API REST JSON** qui donne accès à :

- des **données par** station ou par **ville** ;
- **l’AQI global** ;
- **les polluants** : PM2.5, PM10, NO2, CO, SO2, Ozone ;
- le nom de la station ;
- les coordonnées géographiques ;
- **l’organisme source** ;
- certaines données météo ;
- **des prévisions qualité de l’air** sur plusieurs jours.

\*AQI : pour Air Quality Index.

En termes d'accès maintenant, l'API fonctionne avec un **token d'accès obligatoire**. Elle est aussi soumise à un **quota** qui, par défaut, est **de 1000 requêtes par seconde**.

## Analyse des endpoints

L'API propose plusieurs endpoints pour **récupérer les données de qualité de l'air pour nos villes françaises** :

- par nom de ville ;
- par données GPS ;
- **par station**.

L'API propose aussi une **fonctionnalité de recherche de stations** :

- par recherche **par nom de ville / mot-clé** ;
- par bbox (récupère les stations dans une zone géographique précise).

### La recherche par ville

La recherche par ville permet de récupérer les données de qualité de l'air d'une ville.

Cependant, cet endpoint fonctionne avec une sélection automatique de station. Il convient donc d'être prudent dans la mesure où une ville peut en avoir plusieurs. Aussi, un nom de ville peut être ambigu et est soumis à une marge d'erreur.

Cet endpoint est donc utile pour un MVP mais doit être abandonné par la suite par souci de fiabilité.

### La recherche coordonnées GPS

La recherche par coordonnées GPS récupère la station la plus pertinente autour d’un point GPS.

Elle est donc plus fiable géographiquement que la recherche par nom de ville.

Ici il faut faire attention au fait que la station retournée peut ne pas être exactement dans la ville, mais simplement au plus proche du point GPS donné.

### La recherche par identifiant de station

La recherche par identifiant de station permet de retourner les données de qualité de l'air d'une station précise.

Elle est la plus fiable pour une historicisation dans le temps. Une fois que l'on a identifié les stations à suivre pour chaque ville, on stocke leurs IDs et on interroge toujours les mêmes.

Cet endpoint peut être retenu comme cible finale pour la production.

### La recherche de station par ville / mot-clé

La recherche de station par ville / mot-clé permet de retourner les stations correspondant à une ville ou un mot-clé.

Elle peut être très utile au démarrage pour construire le référentiel des stations.

### La recherche par stations dans une zone (bbox)

La recherche par stations dans une zone (bbox) permet de récupérer des stations dans une zone géographique.

Cela peut être intéressant si l'on souhaite auditer plusieurs stations autour d’une région ou d’une ville, mais c'est plus complexe.

### Conclusion

Pour conclure, la stratégie la plus robuste consiste d'abord à utiliser `search` pour identifier et qualifier les stations pertinentes par zone urbaine. Ensuite, les stations retenues sont stockées dans une table de référence, puis l'ingestion se fait de manière stable via `feed/@station_id`, afin d'historiser les mesures toutes les heures.

Cette approche répond directement au besoin de Good Air : **suivre la qualité de l'air dans les principales villes de France et historiciser les mesures chaque heure**. Elle permet également d'exploiter de manière fiable les informations clés exposées par cette **API REST JSON**, notamment **l’AQI global**, **les polluants** (PM2.5, PM10, NO2, CO, SO2, Ozone), **l’organisme source** et **des prévisions qualité de l’air** sur plusieurs jours.

Enfin, le dispositif reste compatible avec les contraintes techniques de l'API, en particulier le **token d'accès obligatoire** et le **quota** associé, ce qui en fait une base crédible pour un passage en production.
