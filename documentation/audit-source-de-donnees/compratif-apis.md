# 1. Comparatif des deux APIs :

L’objectif est de voir les différences entre les deux APIs, mais aussi de montrer qu’elles sont complémentaires.

| Élément comparé | API OpenWeather | API AQICN |
|----------------|----------------|-----------|
| Type de données | Données météorologiques | Données de qualité de l’air |
| Objectif | Connaître la météo d’une ville | Suivre la pollution de l’air |
| Données principales | Température, humidité, pression, vent, description météo | AQI, PM2.5, PM10, NO2, CO, SO2, O3 |
| Format de réponse | JSON | JSON |
| Accès | Clé API obligatoire | Token obligatoire |
| Exemple d’usage | Récupérer la météo de Paris | Récupérer la qualité de l’air d’une station à Paris |
| Fiabilité géographique | Recherche par ville simple | Plus fiable avec un identifiant de station |
| Données temporelles | Champ dt | Champ time.iso |
| Difficulté | Plutôt simple à utiliser | Plus complexe car il faut choisir les bonnes stations |

## Différences clés identifiées

· OpenWeather sert surtout à récupérer les conditions météo alors que AQICN sert à récupérer les mesures de pollution de l’air.  
· OpenWeather est plus simple à utiliser car on peut faire une recherche directement par ville tandis que AQUICN demande plus de précaution car une ville peut avoir plusieurs stations.  
· Pour AQICN, il est préférable d’utiliser un identifiant de station pour avoir des données plus fiables dans le temps.  

## Complémentarité des deux APIs :

· OpenWeather permet de connaître le contexte météo.  
· AQICN permet de mesurer la qualité de l’air.  
· En combinant les deux, on peut analyser si la météo influence la pollution (le vent, la température ou l’humidité peuvent avoir un impact sur la dispersion des polluants)
