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

Se connecter avec le **Root Token** sur http://vault.homelab.local

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
    ingress.yaml         # Ingress nginx → vault:8200
```
