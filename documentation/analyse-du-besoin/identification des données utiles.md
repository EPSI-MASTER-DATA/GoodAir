# Identification des donnnées utiles

L’objectif de cette étape est d’identifier les données utiles dans la réponse de l’API OpenWeather.On doit donc que conserver les informations nécessaires pour le projet, afin de simplifier le stockage et faciliter les analyses.

## Réponse de la requête :

```json id="i8zzx7"
{"coord":{"lon":2.3488,"lat":48.8534},"weather":[{"id":800,"main":"Clear","description":"clear sky","icon":"01d"}],"base":"stations","main":{"temp":16.17,"feels_like":14.81,"temp_min":15.28,"temp_max":17.92,"pressure":1020,"humidity":37,"sea_level":1020,"grnd_level":1011},"visibility":10000,"wind":{"speed":6.69,"deg":80},"clouds":{"all":0},"dt":1777539367,"sys":{"type":2,"id":2012208,"country":"FR","sunrise":1777523527,"sunset":1777575802},"timezone":7200,"id":2988507,"name":"Paris","cod":200}
```

À partir de la réponse JSON, les champs suivants ont été sélectionnés :
name → nom de la ville
main.temp → température
main.humidity → humidité
main.pressure → pression
weather.description → description du temps
wind.speed → vitesse du vent
dt → date de la mesure

Ces champs ont été retenus car ils sont directement utiles pour analyser les conditions météo et leur impact sur la qualité de l’air.

La température, l’humidité et la pression permettent de décrire les conditions climatiques
La description météo donne une information simple (pluie, ciel clair, etc.)
La vitesse du vent peut influencer la dispersion de la pollution
La date permet de suivre l’évolution dans le temps
La ville permet de comparer plusieurs zones