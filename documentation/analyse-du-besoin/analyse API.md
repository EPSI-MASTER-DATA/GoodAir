# Analyse de l’API météorologique

L’objectif est de savoir quelles informations peuvent être récupérées et comment les utiliser dans le projet.

## Endpoint utilisé

- L’API est accessible via une requête HTTP avec une URL 

### Exemple utilisé pour récupérer la météo :

```bash
https://api.openweathermap.org/data/2.5/weather?q=Paris&appid=dd07e5a6415da382783d98973a7cc1af&units=metric


##Champs principaux
name → nom de la ville
main.temp → température
main.humidity → humidité
main.pressure → pression
weather.description → description du temps
wind.speed → vitesse du vent
dt → date de la mesure
Contraintes d’accès identifiées
L’accès à l’API nécessite une clé API (paramètre appid)
Sans cette clé, une erreur 401 est retournée
Il existe une limite de requêtes (rate limit)
Les données sont retournées en format JSON
Certaines données doivent être transformées (ex : date, unités)