# Vault — HashiCorp Vault sur cluster-payload

## Déploiement

Vault est déployé via FluxCD (HelmRelease) depuis le chart officiel HashiCorp.

- **Namespace** : `vault`
- **Chart** : `hashicorp/vault` v0.29.*
- **Storage** : `file` (local, sans PVC externe)
- **TLS** : désactivé (HTTP uniquement)
- **HA** : désactivé (mode standalone)

## Accès

URL : http://vault.homelab.local

DNS configuré dans Pi-hole : `vault.homelab.local → 192.168.1.201`

## Initialisation (première fois)

```bash
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=1 \
  -key-threshold=1 -format=json
```

Sauvegarder le `unseal_keys_b64[0]` et le `root_token`.

## Unseal après redémarrage

Vault se re-scelle à chaque redémarrage du pod. Il faut le desceller manuellement :

```bash
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY>
```

Vérifier le statut :

```bash
kubectl exec -n vault vault-0 -- vault status
```

## Connexion UI

Se connecter avec le **Root Token** sur https://vault.homelab.local

## Structure FluxCD

```
apps/
  base/vault/
    namespace.yaml       # namespace vault
    source.yaml          # HelmRepository hashicorp
    release.yaml         # HelmRelease vault
    kustomization.yaml
  cluster-payload/vault/
    kustomization.yaml   # référence base + patches
```

---

## Vault Secrets Operator (VSO)

Le VSO synchronise les secrets Vault vers des Kubernetes Secrets automatiquement.

### Composants déployés par FluxCD

```
infrastructure/
  base/vso/                          # HelmRelease VSO
  cluster-payload/vso/               # VaultConnection + VaultAuth VSO (namespace system)
  cluster-payload/vso-config/        # VaultConnection + VaultAuth pour les apps
```

### VaultConnection (commune à tout le cluster)

Définie dans `infrastructure/cluster-payload/vso-config/vault-connection.yaml` :

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: vault-connection
  namespace: vault-secrets-operator-system
spec:
  address: https://vault.homelab.local
  skipTLSVerify: true
```

### Flow complet pour synchroniser un secret

```
Vault KV                    VSO                        Kubernetes
kv/warap/staging/           VaultStaticSecret    →     Secret postgres-credentials
  clusterpayload/warap/     (namespace: warap)          (namespace: warap)
  postgres-credentials
```

1. Stocker le secret dans Vault (voir `staging/vault-homelab/README.md`)
2. Créer une `VaultAuth` dans le namespace applicatif
3. Créer une `VaultStaticSecret` → VSO crée le Kubernetes Secret

Voir `datacenter-rancher-configuration/staging/vault-homelab/README.md` pour le détail complet.
