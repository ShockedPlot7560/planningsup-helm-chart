# PlanningSup — Helm Chart

[![Helm](https://img.shields.io/badge/Helm-v3-0F1689)](https://helm.sh)
[![Chart version](https://img.shields.io/badge/Chart-0.4.9-blue)](https://github.com/kernoeb/PlanningSup)
[![App version](https://img.shields.io/badge/App-3.3.2-green)](https://github.com/kernoeb/PlanningSup)

Un chart Helm pour déployer **[PlanningSup](https://planningsup.app)** sur Kubernetes.

PlanningSup est un planning universitaire moderne (PWA) réalisé par [@kernoeb](https://github.com/kernoeb) :
calendrier au format ICS converti en événements, vues jour / semaine / mois, couleurs par catégorie ou UE,
thèmes clair/sombre, et fonctionnement hors connexion. L'application est disponible en public sur
[planningsup.app](https://planningsup.app).

## Ce que déploie le chart

| Composant | Description |
|---|---|
| **Webapp** | L'application PlanningSup complète (API Elysia + PWA Vue 3), image `ghcr.io/kernoeb/planningsup`. |
| **PostgreSQL** | Base de données intégrée (optionnelle), avec volume persistant pour les données. |
| **Plannings** | Des fichiers JSON décrivant les calendriers ICS de vos établissements, injectés via un ConfigMap. |

Le chart se charge aussi de créer les ConfigMaps nécessaires : `DATABASE_URL`, `NODE_ENV`, `RUN_JOBS`,
`PLANNINGS_LOCATION`, `PUBLIC_ORIGIN` et `TRUSTED_ORIGINS` pour la webapp, ainsi que les identifiants
`POSTGRES_USER`, `POSTGRES_PASSWORD` et `POSTGRES_DB` pour la base.

## Prérequis

- Un cluster Kubernetes (toute distribution supportée par Helm)
- [Helm](https://helm.sh/docs/intro/install/) ≥ 3.8 (pour l'installation depuis un registre OCI)
- Un `StorageClass` avec accès `ReadWriteOnce` (si vous utilisez le PostgreSQL intégré)

## Installation

Le chart est publié sur le registre OCI GitHub Container Registry (GHCR) :

```bash
helm install planningsup \
  oci://ghcr.io/shockedplot7560/planningsup-helm-chart/charts/planningsup \
  --version 0.4.9
```

> Le registre GHCR exige des chemins en minuscules, d'où le `shockedplot7560` dans l'URL
> (le dépôt est `ShockedPlot7560/planningsup-helm-chart`).

### Installation avec une configuration personnalisée

Créez un fichier `values.yaml` avec votre configuration (plannings, identifiants, origine publique…) :

```bash
helm install planningsup \
  oci://ghcr.io/shockedplot7560/planningsup-helm-chart/charts/planningsup \
  --version 0.4.9 \
  --namespace planningsup \
  --create-namespace \
  --values values.yaml
```

Exemple minimal de `values.yaml` :

```yaml
publicOrigin: "https://planningsup.exemple.fr"

postgres:
  user: "planningsup"
  password: "un-mot-de-passe-robuste"
  dbName: "planningsup"

plannings:
  mon-universite.json: |
    {
      "title": "Mon Université",
      "group": "Ma Ville",
      "children": [
        {
          "id": "cours-sciences",
          "title": "Sciences",
          "url": "https://edt.univ-exemple.fr/calendrier.ics"
        }
      ]
    }
```

### Mise à niveau et rollback

```bash
helm upgrade planningsup \
  oci://ghcr.io/shockedplot7560/planningsup-helm-chart/charts/planningsup \
  --version <nouvelle-version> \
  --values values.yaml

helm history planningsup        # liste les révisions
helm rollback planningsup 1     # revenir à une révision précédente si besoin
```

### Désinstallation

```bash
helm uninstall planningsup
```

> La suppression de la release supprime le `PersistentVolumeClaim` et donc **les données PostgreSQL**
> associées. Sauvegardez la base au préalable si vous comptez la réutiliser.

## Configuration

| Paramètre | Description | Défaut |
|---|---|---|
| `publicOrigin` | Origine publique du déploiement (utilisée pour `PUBLIC_ORIGIN` et `TRUSTED_ORIGINS`). | `https://planningsup.app` |
| `imagePullSecrets` | Secrets à utiliser pour tirer l'image depuis un registre privé. | `[]` |
| `webapp.replicaCount` | Nombre de réplicas de la webapp. | `1` |
| `webapp.image.repository` | Image Docker de la webapp. | `ghcr.io/kernoeb/planningsup` |
| `webapp.image.tag` | Tag de l'image (par défaut : `Chart.appVersion`). | — |
| `webapp.image.pullPolicy` | Politique de pull de l'image. | `Always` |
| `webapp.service.type` | Type du Service de la webapp. | `ClusterIP` |
| `webapp.service.port` | Port exposé par le Service. | `20000` |
| `webapp.env` | Variables d'environnement supplémentaires (liste Kubernetes). | `[]` |
| `webapp.envFrom` | Sources d'environnement supplémentaires (ConfigMap/Secret refs). | `[]` |
| `webapp.command` | Commande de démarrage du conteneur (override). | — |
| `webapp.args` | Arguments du conteneur (override). | — |
| `postgres.enabled` | Active le déploiement du PostgreSQL intégré. | `true` |
| `postgres.replicaCount` | Nombre de réplicas de PostgreSQL. | `1` |
| `postgres.user` | Utilisateur PostgreSQL (`POSTGRES_USER`). | `user` |
| `postgres.password` | Mot de passe PostgreSQL (`POSTGRES_PASSWORD`). | `password` |
| `postgres.dbName` | Nom de la base (`POSTGRES_DB`). | `dbname` |
| `postgres.image.repository` | Image PostgreSQL. | `postgres` |
| `postgres.image.tag` | Tag de l'image PostgreSQL. | `18` |
| `postgres.image.pullPolicy` | Politique de pull de l'image PostgreSQL. | `IfNotPresent` |
| `postgres.service.port` | Port interne du PostgreSQL (5432 par convention). | `5432` |
| `postgres.persistence.size` | Taille du volume persistant (`PersistentVolumeClaim`). | `5Gi` |
| `plannings` | Map `nom-de-fichier.json` → contenu JSON des plannings, montée dans `/app/plannings`. | — |

> **Réservé :** les valeurs `timezone` et `webapp.service.nodePort` sont présentes dans `values.yaml`
> mais ne sont pas encore exploitées par les templates du chart.

### Plannings personnalisés

PlanningSup lit des fichiers JSON (un par établissement) décrivant un arbre de calendriers ICS.
Le chart permet de fournir ces fichiers directement dans `values.yaml` : chaque clé de la map `plannings`
devient un fichier d'un ConfigMap, monté dans `/app/plannings` (la variable `PLANNINGS_LOCATION`
pointe dessus). Une modification des plannings redéploie la webapp (checksum du ConfigMap).

```yaml
plannings:
  iut-exemple.json: |
    {
      "title": "IUT Exemple",
      "group": "Exemple",
      "children": [
        {
          "id": "info-annee-1",
          "title": "Info 1A",
          "url": "https://edt.iut-exemple.fr/info-1a.ics"
        },
        {
          "id": "info-annee-2",
          "title": "Info 2A",
          "url": "https://edt.iut-exemple.fr/info-2a.ics"
        }
      ]
    }
```

Le format est décrit dans le repository [kernoeb/PlanningSup](https://github.com/kernoeb/PlanningSup)
(dossier `resources/plannings`). Vous pouvez aussi contribuer vos propres établissements en amont.

### Utiliser une base PostgreSQL externe

Si vous préférez votre propre base de données, désactivez le PostgreSQL intégré et fournissez `DATABASE_URL`
via `webapp.env` (ou `webapp.envFrom`) :

```yaml
postgres:
  enabled: false

webapp:
  env:
    - name: DATABASE_URL
      value: "postgres://utilisateur:motdepasse@db.externe.example:5432/planningsup"
```

### Exposer l'application

Le chart n'expose que des Services internes (`ClusterIP`). Pour rendre PlanningSup accessible de l'extérieur,
utilisez un `port-forward` en développement :

```bash
kubectl port-forward svc/planningsup-webapp 20000:20000
# puis ouvrez http://localhost:20000
```

ou créez un Ingress vers le Service `planningsup-webapp` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: planningsup
spec:
  ingressClassName: nginx
  rules:
    - host: planningsup.exemple.fr
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: planningsup-webapp
                port:
                  number: 20000
```

## Ressources

La webapp demande `10m` de CPU / `100Mi` de mémoire et est limitée à `200m` de CPU / `150Mi` de mémoire.
PostgreSQL est déployé avec des sondes de readiness/liveness (`pg_isready`).

## Publier une nouvelle version (mainteneurs)

La publication est automatisée par GitHub Actions (`.github/workflows/helm.yaml`) : pousser un tag
`sémantique` déclenche l'empaquetage et le push du chart vers
`oci://ghcr.io/shockedplot7560/planningsup-helm-chart/charts`.

```bash
git tag v0.5.0
git push origin v0.5.0
```

Pensez à aligner la `version` du chart dans `Chart.yaml` sur le tag poussé.

## Liens

- [PlanningSup (application)](https://planningsup.app)
- [kernoeb/PlanningSup (code source)](https://github.com/kernoeb/PlanningSup)
- [planningsup-helm-chart (ce dépôt)](https://github.com/ShockedPlot7560/planningsup-helm-chart)