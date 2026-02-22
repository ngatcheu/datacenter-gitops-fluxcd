# Cert-manager - Infrastructure

Deploiement de cert-manager via FluxCD.

## Structure

```
infrastructure/base/
├── cert-manager/           # Installation cert-manager
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── source.yaml         # HelmRepository jetstack
│   └── release.yaml        # HelmRelease cert-manager
│
└── cert-manager-issuers/   # CA et Issuers (deploye apres cert-manager)
    ├── kustomization.yaml
    ├── selfsigned-issuer.yaml
    ├── homelab-ca.yaml      # CA racine + ClusterIssuer
    └── wildcard-certificate.yaml
```

## Activation sur un cluster

Ajouter dans `clusters/<votre-cluster>/cert-manager.yaml` :

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-manager
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/base/cert-manager
  prune: true
  wait: true
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-manager-issuers
  namespace: flux-system
spec:
  dependsOn:
    - name: cert-manager
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/base/cert-manager-issuers
  prune: true
  wait: true
```

## Utilisation dans vos applications

### Via annotation Ingress (automatique)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app
  annotations:
    cert-manager.io/cluster-issuer: "homelab-ca-issuer"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - mon-app.home.lab
      secretName: mon-app-tls
  rules:
    - host: mon-app.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mon-app
                port:
                  number: 80
```

### Via certificat wildcard (reference le secret existant)

```yaml
spec:
  tls:
    - hosts:
        - mon-app.home.lab
      secretName: wildcard-homelab-tls-secret
```

## Issuers disponibles

| Issuer | Type | Usage |
|--------|------|-------|
| `selfsigned-issuer` | ClusterIssuer | Creation de la CA uniquement |
| `homelab-ca-issuer` | ClusterIssuer | Signer les certificats pour *.home.lab |

## Verification

```bash
# Verifier cert-manager
kubectl get pods -n cert-manager

# Verifier les issuers
kubectl get clusterissuers

# Verifier les certificats
kubectl get certificates -A

# Verifier le secret CA
kubectl get secret homelab-ca-secret -n cert-manager
```
