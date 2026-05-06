L’objectif est de définir quelles informations doivent être stockées, et lesquelles peuvent être ignorées, afin d’éviter de garder trop de données inutiles.

## Données brutes retenus :

### API OpenWeather

· name : nom de la ville  
· coord.lat : latitude  
· coord.lon : longitude  
· main.temp : température  
· main.humidity : humidité  
· main.pressure : pression  
· weather.description : description météo  
· wind.speed : vitesse du vent  
· dt : date de la mesure  
· sys.country : pays  

### API AIQCN

· status : statut de la réponse API  
· data.idx : identifiant de la station  
· data.aqi : indice global de qualité de l’air  
· data.city.name : nom de la ville/station  
· data.city.geo : coordonnées GPS  
· data.dominentpol : polluant dominant  
· data.iaqi.pm25 : particules fines PM2.5  
· data.iaqi.pm10 : particules PM10  
· data.iaqi.no2 : dioxyde d’azote  
· data.iaqi.o3 : ozone  
· data.iaqi.co : monoxyde de carbone  
· data.iaqi.so2 : dioxyde de soufre  
· data.time.iso : date et heure de la mesure  
· data.attributions : sources des données  

---

## Champs exclus :

| API | Champ | Raison de l’exclusion |
|-----|------|----------------------|
| OpenWeather | weather.icon | Utile uniquement pour l’affichage (interface), pas pour l’analyse |
| OpenWeather | base | Champ technique sans intérêt métier |
| OpenWeather | visibility | Donnée non prioritaire pour le projet |
| OpenWeather | clouds.all | Information secondaire, peu utile pour les analyses |
| OpenWeather | sys.sunrise / sys.sunset | Non nécessaire pour le MVP |
| OpenWeather | id | Identifiant interne sans utilité pour le projet |
| AQICN | debug | Champ technique, utilisé uniquement pour le debug |
| AQICN | forecast | Données de prévision non utilisées dans le MVP |
| AQICN | iaqi.t, iaqi.h, iaqi.p, iaqi.w | Données météo déjà couvertes par OpenWeather |
| AQICN | city.url | Non utile pour l’analyse des données |
| AQICN | attributions.logo | Utile pour affichage, pas pour le traitement |

---

Les champs retenus sont ceux qui répondent directement au besoin du projet GoodAir :

· suivre la qualité de l’air ;  
· analyser les conditions météo ;  
· comparer les villes ou stations ;  
· historiser les mesures dans le temps ;  
· créer des dashboards et rapports.  

Les champs inutiles ou trop techniques ont été exclus afin de garder un modèle de données plus simple et plus clair.

---