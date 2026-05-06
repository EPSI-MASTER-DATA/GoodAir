
L’objectif est d’obtenir un jeu de données cohérent et exploitable, en regroupant les informations utiles et en standardisant les formats.

## Champs correspondants

| Donnée métier | OpenWeather | AQICN | Choix final |
|--------------|-------------|-------|-------------|
| Ville | name | city.name | OpenWeather |
| Latitude | coord.lat | city.geo | OpenWeather |
| Longitude | coord.lon | city.geo | OpenWeather |
| Température | main.temp | iaqi.t | OpenWeather |
| Humidité | main.humidity | iaqi.h | OpenWeather |
| Pression | main.pressure | iaqi.p | OpenWeather |
| Vent | wind.speed | iaqi.w | OpenWeather |
| AQI (pollution) | ----- | data.aqi | AQICN |
| Polluants (PM2.5, NO2…) | ----- | iaqi.* | AQICN |
| Date | dt | time.iso | AQICN (plus précis) |

---

## Logique d’harmonisation

· Utiliser OpenWeather comme source principale pour la météo ;  
· Utiliser AQICN comme source principale pour la qualité de l’air ;  
· Associer les données via :  
· la ville  
· ou les coordonnées GPS  
· Convertir les formats ;  
· Gérer les données manquantes ;  

---

# Conclusion

Les deux APIs sont complémentaires mais différentes.  
L’harmonisation permet de combiner leurs données pour obtenir une vision complète : météo + qualité de l’air.  

Cette étape est essentielle pour garantir la cohérence des données avant leur stockage et leur exploitation.