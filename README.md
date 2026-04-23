# StockAlert API

API de gestion d'alertes de stock pour une boutique en ligne.
Une alerte est automatiquement creee quand le stock d'un produit passe sous son seuil.

## Stack

- Runtime: Node.js 18 (sans framework)
- Stockage: en memoire (pas de base de donnees)

## Structure

```text
stockalert/
|-- app.js
|-- app.test.js
|-- package.json
|-- Dockerfile
|-- k8s/
|   |-- namespace.yaml
|   |-- deployment.yaml
|   |-- service.yaml
|   |-- hpa.yaml
|   '-- kustomization.yaml
'-- .github/workflows/ci.yml
```

## API

| Methode | Route | Description |
|---|---|---|
| GET | `/health` | Etat de l'app et statistiques |
| GET | `/products` | Liste tous les produits |
| POST | `/products` | Ajouter un produit `{ name, stock, threshold? }` |
| PATCH | `/products/:id/stock` | Mettre a jour le stock `{ stock }` |
| GET | `/alerts` | Liste toutes les alertes |
| GET | `/alerts/active` | Alertes non resolues uniquement |
| PATCH | `/alerts/:id/resolve` | Resoudre une alerte |
| DELETE | `/alerts/:id` | Supprimer une alerte |

Severites: `critical` (stock = 0), `warning` (stock < threshold)

## Lancer en local

```bash
npm install
npm start
# http://localhost:3000
```

## Tests

```bash
npm test
npm run lint
```

## Variables d'environnement

| Variable | Defaut | Description |
|---|---|---|
| `PORT` | `3000` | Port du serveur |
| `APP_ENV` | `development` | Environnement |
| `APP_VERSION` | `1.0.0` | Version |
| `DEFAULT_THRESHOLD` | `10` | Seuil d'alerte par defaut |

## Deploiement Kubernetes

Les manifests sont dans `k8s/`:

- `namespace.yaml`: namespace `stockalert`
- `deployment.yaml`: 2 replicas, rolling update, readiness/liveness probes
- `service.yaml`: service interne sur le port `80`
- `hpa.yaml`: autoscaling CPU de 2 a 6 pods

### 1) Adapter l'image Docker Hub

Dans `k8s/deployment.yaml`, remplace:

- `DOCKERHUB_USERNAME`
- `v1.0.2` par ton tag (ex: `v1.0.4`)

### 2) Deployer

```bash
kubectl apply -k k8s
kubectl -n stockalert get all
kubectl -n stockalert get hpa
```

### 3) Tester l'app

```bash
kubectl -n stockalert port-forward svc/stockalert 8080:80
# puis ouvre http://localhost:8080/health
```

### 4) Mise a jour sans downtime

Apres push d'une nouvelle image sur Docker Hub:

```bash
kubectl -n stockalert set image deployment/stockalert stockalert=docker.io/<dockerhub_user>/stockalert:<new_tag>
kubectl -n stockalert rollout status deployment/stockalert
```

### 5) Self-healing (verification)

```bash
kubectl -n stockalert get pods
kubectl -n stockalert delete pod <pod_name>
kubectl -n stockalert get pods -w
```

Kubernetes recree automatiquement le pod supprime.

---

Projet support du TP CI/CD - DevOps Bachelor 3
