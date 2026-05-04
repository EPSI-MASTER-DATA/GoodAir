# Choix du mode de traitement des données : Batch vs Streaming

## Contexte

Dans le cadre du projet GoodAir, une question structurante concerne le mode de traitement des données :

- traitement **batch** (par lots, à intervalles réguliers) ;
- traitement **streaming** (en continu, en temps réel).

Ce choix impacte directement l’architecture du pipeline, les technologies utilisées et la complexité globale du système.

---

## Définition des approches

### Traitement batch

Le traitement batch consiste à récupérer et traiter les données à intervalles réguliers.

Exemple dans le projet :

```text
Collecte toutes les heures des données AQICN et OpenWeather
```

Caractéristiques :

- exécution planifiée ;
- traitement par lots ;
- simplicité de mise en œuvre ;
- cohérence des données à un instant donné.

---

### Traitement streaming

Le traitement streaming consiste à traiter les données en continu, au fil de leur arrivée.

Exemple théorique :

```text
Réception en temps réel des mesures de qualité de l’air via un flux continu
```

Caractéristiques :

- traitement en temps réel ;
- faible latence ;
- architecture plus complexe ;
- nécessité d’outils spécifiques (Kafka, Spark Streaming, Flink, etc.).

---

## Analyse des besoins du projet

Le projet GoodAir repose sur les éléments suivants :

- récupération de données via des APIs REST ;
- mise à disposition de données mises à jour régulièrement (pas en temps réel strict) ;
- besoin d’historisation horaire ;
- usage orienté analyse et reporting (data visualisation, études scientifiques).

Le sujet précise que les données doivent être **collectées régulièrement (ex : toutes les heures)** .

---

## Contraintes liées aux APIs

Les APIs utilisées (AQICN, OpenWeatherMap) présentent des contraintes importantes :

- pas de flux temps réel natif (pas de streaming) ;
- accès via requêtes HTTP ;
- limitation du nombre d’appels (quotas) ;
- données mises à jour à fréquence régulière (et non en continu).

👉 Ces éléments rendent une approche streaming peu adaptée.

---

## Comparaison des approches

| Critère                   | Batch                | Streaming         |
| ------------------------- | -------------------- | ----------------- |
| Adaptation aux APIs REST  | ✔️ Très adapté       | ❌ Peu adapté     |
| Complexité technique      | ✔️ Faible            | ❌ Élevée         |
| Coût                      | ✔️ Faible            | ❌ Élevé          |
| Latence                   | ⏳ Moyenne (ex : 1h) | ⚡ Très faible    |
| Besoin métier GoodAir     | ✔️ Suffisant         | ❌ Surdimensionné |
| Facilité de mise en œuvre | ✔️ Simple            | ❌ Complexe       |
| Démonstration MSPR        | ✔️ Adaptée           | ⚠️ Risquée        |

---

## Choix retenu : traitement batch

Le projet GoodAir adopte une approche **batch**, avec une fréquence d’ingestion horaire.

### Fonctionnement

```text
Toutes les heures :
    ↓
Appel des APIs AQICN et OpenWeather
    ↓
Stockage des données brutes (Data Lake)
    ↓
Transformation et nettoyage (ODS)
    ↓
Alimentation du Data Warehouse
```

---

## Justification

### 1. Alignement avec les sources de données

Les APIs ne fournissent pas de flux en continu.

👉 Le batch est donc naturellement adapté.

---

### 2. Adéquation au besoin métier

Les chercheurs n’ont pas besoin de données en temps réel à la seconde près.

👉 Une mise à jour horaire est suffisante pour :

- analyser les tendances ;
- produire des rapports ;
- suivre l’évolution de la qualité de l’air.

---

### 3. Simplicité et robustesse

Le batch permet :

- une architecture plus simple ;
- une meilleure maîtrise du pipeline ;
- une réduction des risques techniques.

---

### 4. Respect des contraintes MSPR

Le batch permet de répondre efficacement aux exigences :

- collecte régulière des données ;
- historisation ;
- traitement ETL/ELT ;
- stockage et analyse.

---

### 5. Optimisation des coûts

Le batch limite :

- le nombre d’appels API ;
- les ressources nécessaires ;
- la complexité de l’infrastructure.

---

## Pourquoi le streaming n’est pas retenu

Le streaming n’est pas retenu pour les raisons suivantes :

- absence de flux temps réel côté API ;
- complexité technique élevée (Kafka, Spark Streaming, etc.) ;
- surdimensionnement par rapport au besoin ;
- difficulté de mise en œuvre dans un cadre pédagogique.

---

## Ouverture : évolution possible vers du streaming

Bien que non retenu dans le cadre du MVP, une architecture streaming pourrait être envisagée si :

- des capteurs IoT fournissant des données en continu étaient intégrés ;
- des besoins en temps réel strict apparaissaient (alertes instantanées, monitoring critique).

Dans ce cas, des technologies comme :

- Apache Kafka ;
- Apache Flink ;
- Spark Streaming ;

pourraient être utilisées.

---

## Conclusion

Le choix d’un traitement batch pour le projet GoodAir est justifié par :

- la nature des sources de données ;
- les besoins métiers ;
- les contraintes techniques ;
- les exigences du projet MSPR.

Cette approche permet de construire un pipeline robuste, simple et adapté, tout en restant évolutif vers des architectures plus complexes si nécessaire.
