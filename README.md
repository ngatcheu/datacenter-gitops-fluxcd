# GitOps Apps

Repository GitOps pour la gestion des applications et infrastructures Kubernetes via FluxCD.

## Structure

```
gitops-apps/
├── apps/                          # Applications
│   ├── base/                      # Définitions de base
│   │   └── rancher/               # Rancher Manager
│   └── cluster-rancher/           # Overlay cluster rancher
│       └── rancher/
├── clusters/                      # Configuration par cluster
│   ├── cluster-payload/           # Cluster Payload
│   │   ├── flux-system/
│   │   ├── apps.yaml
│   │   ├── infrastructure.yaml
│   │   └── cert-manager.yaml
│   └── cluster-rancher/           # Cluster Rancher
│       ├── flux-system/
│       ├── apps.yaml
│       └── infrastructure.yaml
└── infrastructure/                # Composants d'infrastructure
    └── base/
        ├── cert-manager/          # Gestionnaire de certificats
        └── cert-manager-issuers/  # Issuers et certificats
```

## Clusters

| Cluster | Description | Chemin |
|---------|-------------|--------|
| cluster-payload | Cluster applicatif | `clusters/cluster-payload` |
| cluster-rancher | Cluster Rancher Manager | `clusters/cluster-rancher` |

## Composants déployés

### Infrastructure

- **cert-manager** : Gestion automatique des certificats TLS
- **cert-manager-issuers** : CA auto-signée et certificat wildcard

### Applications

- **Rancher** : Interface de gestion Kubernetes (cluster-rancher uniquement)

## Bootstrap

Le bootstrap de FluxCD est géré via Terraform dans :
- `../flux-configuration/flux-configuration-github-payload/`
- `../flux-configuration/flux-configuration-github-rancher/`

## Ajouter une nouvelle application

1. Créer les manifests de base dans `apps/base/<app-name>/`
2. Créer un overlay par cluster dans `apps/<cluster-name>/<app-name>/`
3. Référencer l'application dans `clusters/<cluster-name>/apps.yaml`

## Ajouter un composant d'infrastructure

1. Créer les manifests dans `infrastructure/base/<component>/`
2. Référencer dans `clusters/<cluster-name>/infrastructure.yaml`

## Commandes utiles

```bash
# Vérifier le statut de Flux
flux get all

# Réconcilier manuellement
flux reconcile source git flux-system
flux reconcile kustomization flux-system

# Voir les logs
flux logs

# Suspendre/reprendre une réconciliation
flux suspend kustomization <name>
flux resume kustomization <name>
```
