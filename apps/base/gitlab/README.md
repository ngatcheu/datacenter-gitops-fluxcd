# GitLab

GitLab déployé via **GitLab Operator** (v2.x) sur le cluster `cluster-payload`.

## Architecture

```
FluxCD
├── Kustomization: gitlab-operator   → installe l'operator (CRDs + controller)
└── Kustomization: gitlab            → déploie la CR GitLab (dependsOn: gitlab-operator)
                                         ↓
                                  GitLab Operator
                                         ↓
                          PostgreSQL · Redis · MinIO · Gitaly
                          Webservice (Puma + Workhorse) · Registry
                          Sidekiq · Shell · KAS
```

## Accès

| Service  | URL                              |
|----------|----------------------------------|
| GitLab   | https://gitlab.homelab.local     |
| Registry | https://registry.homelab.local   |

**User** : `root`

**Password initial** :
```bash
kubectl get secret gitlab-gitlab-initial-root-password -n gitlab-system \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

## Structure des fichiers

```
apps/base/gitlab-operator/
├── namespace.yaml        # Namespace gitlab-system
├── source.yaml           # HelmRepository https://charts.gitlab.io
├── operator-release.yaml # HelmRelease gitlab-operator 2.x
└── kustomization.yaml

apps/base/gitlab/
├── gitlab-cr.yaml        # CR GitLab (instance gérée par l'operator)
└── kustomization.yaml

apps/cluster-payload/gitlab/
├── httproute.yaml        # HTTPRoutes gitlab + registry
└── kustomization.yaml

apps/cluster-payload/gitlab-operator/
└── kustomization.yaml
```

## Composants déployés

| Pod | Rôle |
|-----|------|
| `gitlab-webservice` | Puma (Rails) + Workhorse (proxy HTTP) |
| `gitlab-sidekiq` | Jobs asynchrones (emails, CI, etc.) |
| `gitlab-gitaly` | Stockage Git (repos) |
| `gitlab-postgresql` | Base de données principale |
| `gitlab-redis` | Cache et queues Sidekiq |
| `gitlab-minio` | Stockage objet (artifacts, uploads, registry) |
| `gitlab-registry` | Container registry Docker |
| `gitlab-shell` | Accès SSH |
| `gitlab-kas` | GitLab Agent for Kubernetes |

## Stockage (Longhorn)

| PVC | Taille |
|-----|--------|
| Gitaly (repos Git) | 10Gi |
| PostgreSQL | 8Gi |
| MinIO (objets) | 10Gi |
| Redis | 1Gi |

## Troubleshooting

### Webservice 1/2 Running

Puma prend 3-5 min à démarrer. Surveille :
```bash
kubectl logs -n gitlab-system -l app=webservice -c webservice --tail=20
```
Si les probes tuent le pod avant le boot complet, vérifier `initialDelaySeconds` dans la CR.

### Job migrations échoué (DeadlineExceeded)

L'operator bloque sur un job de migration échoué. Supprimer le job pour qu'il soit relancé :
```bash
kubectl get jobs -n gitlab-system | grep migrations
kubectl delete job <nom-du-job> -n gitlab-system
```

### Bucket MinIO manquant (NoSuchBucket)

Supprimer le job `minio-create-buckets` échoué pour qu'il soit relancé :
```bash
kubectl delete job <nom-du-job> -n gitlab-system
kubectl annotate gitlab gitlab -n gitlab-system reconcile=$(date +%s) --overwrite
```

### Trop de replicas webservice

L'operator peut recréer des replicas. Forcer à 1 :
```bash
kubectl scale deployment gitlab-webservice-default -n gitlab-system --replicas=1
```
La CR est configurée avec `minReplicas: 1 / maxReplicas: 1` pour le homelab.

### Récupérer le mot de passe root après changement

Utiliser la toolbox GitLab :
```bash
kubectl exec -n gitlab-system -it gitlab-toolbox-<id> -- gitlab-rails runner \
  "user = User.find_by_username('root'); user.password = 'NewPassword123!'; user.password_confirmation = 'NewPassword123!'; user.save!"
```