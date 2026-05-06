# Qualités des données

L’objectif est d’évaluer la qualité des données récupérées depuis l’API OpenWeather afin d’anticiper les traitements nécessaires avant de les utiliser dans le projet.

## Données manquantes 

- Certaines informations peuvent ne pas être présentes dans toutes les réponses de l’API 
- Par exemple, le vent ou certaines descriptions météo peuvent être absents selon la ville ou le moment 
- Cela peut poser problème pour les analyses si on ne gère pas les valeurs manquantes 

## Formats incohérents 

- La température peut être en Kelvin si on ne précise pas les unités, ce qui n’est pas directement compréhensible 
- La date (champ dt) est sous forme de timestamp (nombre), donc il faut la convertir en date lisible 
- Certaines données sont numériques alors que pour les utilisateurs, il faudra les afficher autrement 

## Données imbriquées 

- Les données sont organisées sous forme de JSON avec plusieurs niveaux (ex : main.Temp, weather.description) 
- Cela veut dire qu’il faut “aller chercher” les informations à l’intérieur d’autres objets 
- Pour les exploiter correctement, il faudra transformer ces données en format plus simple (tableau ou base de données) 

## Problèmes potentiels 

- Dépendance à l’API : si l’API ne fonctionne pas, on ne peut pas récupérer les données 
- Limite de requêtes (rate limit) qui peut bloquer l’accès si on fait trop d’appels 
- Risque d’avoir des données incohérentes ou manquantes selon les situations