# Architecture hybride GoodAir — Réponse aux risques de dépendance fournisseur

## Contexte et problématique

L'architecture Azure initiale (documentée dans `4-benchmark-pipeline-azure.md`) repose entièrement sur des services Microsoft.

Cette approche crée des dépendances fortes sur trois composants transversaux critiques :

- **le monitoring** : Azure Monitor et Log Analytics sont 100 % propriétaires Microsoft ;
- **le CI/CD** : si Azure DevOps est utilisé, le déploiement est également verrouillé sur l'écosystème Microsoft ;
- **l'IaC et les rollbacks d'infrastructure** : ARM et Bicep sont des langages Azure-only, non portables.

> **Note :** l’Infrastructure as Code (IaC) consiste à décrire et déployer l’infrastructure via du code automatisé. Les _rollbacks d’infrastructure_ permettent de revenir rapidement à une version précédente en cas d’erreur ou d’échec de déploiement.

Par soucis de portabilité, nous allons ici proposer une architecture dites 'hybride' proposée ici conserve les services Azure à haute valeur ajoutée (stockage, calcul, serving) tout en substituant les composants transversaux par des alternatives open source portables.

## Principe de décision

| Type de composant                                | Stratégie                                                                             |
| ------------------------------------------------ | ------------------------------------------------------------------------------------- |
| Stockage et calcul (ADLS, Databricks, Synapse)   | Lock-in accepté — valeur ajoutée élevée, migration coûteuse mais rare                 |
| Composants transversaux (monitoring, CI/CD, IaC) | Lock-in refusé — ces composants conditionnent l'opérabilité et la portabilité globale |
| Rollback des données (Delta Lake)                | Déjà open source — aucun changement nécessaire                                        |

> **Note :** le _lock-in fournisseur_ désigne une dépendance forte à un service ou à une technologie propriétaire, compliquant une éventuelle migration vers une autre plateforme.

## Architecture hybride retenue

```text
AQICN / OpenWeatherMap
          ↓
Azure Data Factory
(orchestration + déclenchement)
          ↓
Azure Functions (Python)
(extraction, validation, écriture Bronze)
          ↓
Azure Data Lake Storage Gen2
├── /reference/   (référentiel des stations)
├── /bronze/      (JSON bruts)
├── /silver/      (Delta Lake — nettoyé)
└── /gold/        (Delta Lake — analytique)
          ↓
Azure Databricks + Delta Lake
(transformations Bronze→Silver→Gold)
          ↓
Azure Synapse Analytics — Serverless SQL
(serving Data Warehouse)
          ↓
Power BI
(dashboards, rapports métier)

─────────────────────────────────────────────────
Grafana + OpenTelemetry    (monitoring transversal — portable)
GitHub Actions             (CI/CD — portable)
Terraform                  (IaC — multi-cloud)
Azure Key Vault            (secrets — conservé)
Microsoft Entra ID         (identités, RBAC — conservé)
```

# 1. Monitoring — Grafana + OpenTelemetry

## Problème avec Azure Monitor

Azure Monitor et Log Analytics sont des services entièrement propriétaires. Les dashboards, règles d'alerte et requêtes KQL (Kusto Query Language) ne sont pas portables. En cas de migration vers un autre cloud ou on-premise, l'ensemble du dispositif de supervision est à reconstruire.

## Solution retenue : Grafana + OpenTelemetry

| Composant            | Rôle                                                                  |
| -------------------- | --------------------------------------------------------------------- |
| **OpenTelemetry**    | Standard ouvert CNCF pour l'instrumentation (traces, métriques, logs) |
| **Grafana**          | Plateforme de visualisation open source, portable, multi-source       |
| **Grafana Alerting** | Règles d'alerte configurées en YAML, versionnables dans Git           |

### Pourquoi ce choix

- **Portabilité** : Grafana se connecte à Azure Monitor, Prometheus, Loki, Datadog, InfluxDB — le même dashboard fonctionne quel que soit le cloud sous-jacent.
- **OpenTelemetry** est un standard CNCF adopté par tous les fournisseurs cloud. Les traces et métriques émises par les jobs Databricks et les Functions sont collectées indépendamment d'Azure.
- **Versionnabilité** : les dashboards Grafana et les règles d'alerte sont des fichiers JSON/YAML stockables dans Git.

### Ce qui est instrumenté

- pipelines ADF (via connecteur Azure Monitor → Grafana) ;
- jobs Databricks (via OpenTelemetry SDK Python) ;
- Azure Functions (via OpenTelemetry SDK Python) ;
- Synapse Serverless SQL (via connecteur Azure Monitor → Grafana).

### Déploiement

Grafana est déployé dans un conteneur sur **Azure Container Apps** ce qui le maintient dans l'écosystème Azure sans dépendre de ses services propriétaires de monitoring.

# 2. CI/CD — GitHub Actions

## Problème avec Azure DevOps

Azure DevOps est un outil propriétaire Microsoft. Les pipelines YAML, les environnements, les release gates et les approbations sont stockés dans l'infrastructure Microsoft. En cas de changement de fournisseur, le CI/CD est à reconstruire intégralement.

## Solution retenue : GitHub Actions

### Pourquoi ce choix

- **Portabilité** : GitHub Actions est indépendant d'Azure. Les workflows YAML fonctionnent quel que soit le cloud cible.
- **Intégration Azure native** : des actions officielles existent pour déployer sur Azure (ADF, Databricks, Terraform, Azure Functions) sans dépendre d'Azure DevOps.
- **Versionnabilité** : les workflows sont dans le dépôt Git, au même titre que le code.
- **Standard de marché** : très largement adopté, facilite l'onboarding de nouveaux développeurs.

### Rôle concret de GitHub Actions dans l’architecture

#### MVP

Dans le cadre du MVP, GitHub Actions servira principalement à automatiser les tâches essentielles de validation et de déploiement afin de sécuriser les premières mises en production.

Les workflows permettront de :

- vérifier automatiquement la qualité du code et des traitements ;
- tester les composants avant mise en production ;
- assurer une traçabilité complète des déploiements.

#### Dans une future itération

À terme, GitHub Actions pourra également automatiser l’ensemble de la chaîne de déploiement cloud et data :

- déployer automatiquement l’infrastructure Azure via Terraform ;
- publier les pipelines Azure Data Factory ;
- déployer les notebooks et jobs Databricks ;
- mettre à jour les Azure Functions.

Cette approche permettra de standardiser les mises en production, réduire les erreurs humaines et faciliter la collaboration grâce à une chaîne CI/CD centralisée et versionnée.

### Pipelines définis

#### MVP

| Pipeline            | Déclencheur     | Actions                                             |
| ------------------- | --------------- | --------------------------------------------------- |
| `quality-check.yml` | PR sur develop  | Tests unitaires, lint, validation des schémas Delta |
| `deploy-infra.yml`  | Push sur `main` | Terraform plan + apply (provisioning Azure)         |

#### Dans une future itération

| Pipeline                | Déclencheur     | Actions                                      |
| ----------------------- | --------------- | -------------------------------------------- |
| `deploy-adf.yml`        | Push sur `main` | Publication des pipelines ADF via ARM export |
| `deploy-functions.yml`  | Push sur `main` | Build et déploiement des Azure Functions     |
| `deploy-databricks.yml` | Push sur `main` | Déploiement des notebooks et jobs Databricks |

# 3. IaC et rollback infrastructure — Terraform

## Problème avec ARM / Bicep

ARM et Bicep sont des langages de templating Azure-only. Ils ne sont pas réutilisables sur un autre cloud et créent une dépendance directe sur le plan d'infrastructure.

## Solution retenue : Terraform

### Pourquoi ce choix

- **Multi-cloud** : les mêmes patterns Terraform fonctionnent sur Azure, AWS et GCP.
- **Rollback** : `terraform plan` permet de visualiser les changements avant application ; `terraform state` permet de revenir à un état antérieur de l'infrastructure.
- **Écosystème** : large communauté, modules officiels pour tous les services Azure utilisés (ADLS, Databricks, Synapse, ADF, Key Vault).
- **Séparation des environnements** : gestion native des workspaces dev/test/prod.

### Ressources gérées par Terraform

- compte ADLS Gen2 et containers ;
- workspace Azure Databricks ;
- workspace Azure Synapse Analytics ;
- Azure Data Factory ;
- Azure Functions (plan et application) ;
- Azure Key Vault et politiques d'accès ;
- règles de réseau (Private endpoints, VNet) ;
- instance Grafana sur Azure Container Apps.

## Rollback des données — Delta Lake (inchangé)

Le rollback des données repose sur le **time travel Delta Lake**, qui est un standard open source Apache. Il n'est pas lié à Azure et fonctionnerait de la même manière sur AWS ou GCP. Aucun changement n'est nécessaire sur ce point.

```sql
-- Lecture d'une version antérieure d'une table Delta
SELECT * FROM silver.air_quality VERSION AS OF 42;

-- Restauration
RESTORE TABLE silver.air_quality TO VERSION AS OF 42;
```

# 4. Ce qui est conservé d'Azure et pourquoi

| Service Azure conservé       | Justification                                                                          |
| ---------------------------- | -------------------------------------------------------------------------------------- |
| ADLS Gen2                    | Lock-in acceptable — valeur élevée, Delta Lake portable si migration nécessaire        |
| Azure Databricks             | Multi-cloud natif (Azure, AWS, GCP) — lock-in limité                                   |
| Azure Synapse Serverless SQL | Serving léger, remplaçable par Databricks SQL ou Athena en cas de migration            |
| Azure Data Factory           | Orchestration managée — remplaçable par Airflow si migration, coût de portage maîtrisé |
| Azure Functions              | Facilement remplaçable par AWS Lambda ou Cloud Functions — logique Python standard     |
| Azure Key Vault              | Conservé pour les secrets — remplaçable par HashiCorp Vault si nécessaire              |
| Microsoft Entra ID           | Conservé pour le RBAC — standard entreprise, remplaçable par Okta ou Keycloak          |

# Synthèse des changements

| Composant        | Architecture initiale         | Architecture hybride    | Gain                                      |
| ---------------- | ----------------------------- | ----------------------- | ----------------------------------------- |
| Monitoring       | Azure Monitor + Log Analytics | Grafana + OpenTelemetry | Portabilité totale, dashboards versionnés |
| CI/CD            | Azure DevOps (implicite)      | GitHub Actions          | Portabilité, standard de marché           |
| IaC              | ARM / Bicep                   | Terraform               | Multi-cloud, rollback infrastructure      |
| Rollback données | Delta Lake (déjà open source) | Inchangé                | —                                         |
| Stockage         | ADLS Gen2                     | Inchangé                | Lock-in accepté et justifié               |
| Calcul           | Azure Databricks              | Inchangé                | Multi-cloud natif                         |
| Serving          | Synapse Serverless SQL        | Inchangé                | Lock-in limité et justifié                |
| Data viz         | Power BI                      | Inchangé                | Lock-in accepté                           |

# Risques résiduels

Malgré ces ajustements, des dépendances résiduelles subsistent :

- **ADLS Gen2** reste Azure-specific pour le stockage. Une migration vers S3 ou GCS nécessiterait un travail de reconfiguration (chemins, credentials, connecteurs Databricks).
- **Power BI** reste propriétaire Microsoft. Une alternative open source serait Apache Superset, déjà évalué dans le benchmark.
- **Azure Data Factory** reste propriétaire pour l'orchestration. Apache Airflow (déployé via Terraform sur AKS) constituerait l'alternative la plus portable.

Ces risques sont connus et documentés. Ils sont considérés acceptables dans le cadre du MVP, avec un chemin de migration identifié pour chaque composant.
