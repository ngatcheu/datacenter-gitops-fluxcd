# Gateway API

Installation des CRDs Gateway API (standard Kubernetes officiel) depuis le repo `kubernetes-sigs/gateway-api`.

## Ce qui est installé

Ce n'est **pas un chart Helm** — FluxCD récupère directement les manifests YAML depuis le repo officiel et applique le dossier `./config/crd/standard`.

### CRDs installées

| CRD | Rôle |
|-----|------|
| `GatewayClass` | Définit le type de gateway (implémentation : Traefik) |
| `Gateway` | Instance de gateway (IP, ports, TLS) |
| `HTTPRoute` | Règles de routing HTTP par hostname/path |
| `ReferenceGrant` | Autorise les références cross-namespace |

## Séquence de déploiement

```
gateway-api-source      GitRepository kubernetes-sigs/gateway-api v1.2.1
       ↓ dependsOn
gateway-api             Applique ./config/crd/standard → CRDs installées
       ↓ dependsOn
traefik                 Démarre en tant que controller Gateway API
```

Les deux étapes sont nécessaires car FluxCD ne peut pas réconcilier
un GitRepository externe (kubernetes-sigs) sans l'avoir d'abord enregistré
comme source dans le cluster.

## Relation avec Traefik

Les CRDs seules ne font rien — c'est Traefik qui implémente le controller :

```
GatewayClass (traefik)
       ↓
Gateway (traefik-gateway)  ← IP Cilium LB, ports 80/443, cert wildcard TLS
       ↓
HTTPRoute (par app)        ← hostname → service
```

Traefik surveille les objets `Gateway` et `HTTPRoute` et configure
automatiquement ses entrypoints et routers en conséquence.

## Version

`v1.2.1` — Gateway API standard channel (stable).

Pour mettre à jour, changer le tag dans `source.yaml` :

```yaml
spec:
  ref:
    tag: v1.3.0   # nouvelle version
```