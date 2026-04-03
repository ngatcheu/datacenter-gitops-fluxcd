# CloudNativePG (CNPG) — Opérateur PostgreSQL

## Structure

```
infrastructure/base/cnpg/
├── namespace.yaml      # cnpg-system
├── source.yaml         # HelmRepository : https://cloudnative-pg.github.io/charts
├── release.yaml        # HelmRelease : cloudnative-pg v0.23.0
└── kustomization.yaml

infrastructure/cluster-payload/cnpg/
└── kustomization.yaml

apps/base/warap/
├── namespace.yaml      # warap
├── postgres.yaml       # Cluster CNPG : warap-db
└── kustomization.yaml

apps/cluster-payload/warap/
└── kustomization.yaml
```

---

## Pourquoi CNPG

CloudNativePG est un opérateur Kubernetes qui gère le cycle de vie complet de clusters PostgreSQL :
provisioning, haute disponibilité, sauvegardes, monitoring. Il remplace un PostgreSQL déployé
manuellement ou via un chart générique.

---

## Architecture

```
Flux déploie l'opérateur CNPG
        │
        ▼
cnpg-cloudnative-pg (Deployment)
→ Surveille les ressources kind: Cluster
        │
        ▼
Cluster warap-db (namespace: warap)
→ warap-db-1 (pod primary PostgreSQL)
        │
        ├── warap-db-rw  (ClusterIP — lecture/écriture)
        ├── warap-db-ro  (ClusterIP — lecture seule)
        └── warap-db-r   (ClusterIP — toutes instances)
```

---

## Clusters PostgreSQL déployés

| Cluster | Namespace | Base | User | Taille | Status |
|---------|-----------|------|------|--------|--------|
| warap-db | warap | warap | warap | 5Gi | healthy |

---

## Connexion depuis une application

```yaml
# Variables d'environnement à injecter dans l'app
DB_HOST: warap-db-rw.warap.svc.cluster.local
DB_PORT: 5432
DB_NAME: warap
DB_USER: warap
DB_PASSWORD: <secret warap-db-secret>
```

### Services disponibles

| Service | Usage |
|---------|-------|
| `warap-db-rw.warap.svc.cluster.local` | Lecture / écriture (primary) |
| `warap-db-ro.warap.svc.cluster.local` | Lecture seule (replicas) |
| `warap-db-r.warap.svc.cluster.local` | Toutes les instances |

---

## Secret requis

Le secret doit être créé manuellement avant le déploiement du cluster :

```bash
# Créer le namespace d'abord
kubectl create namespace warap

# Créer le secret
kubectl create secret generic warap-db-secret \
  --namespace warap \
  --from-literal=username=warap \
  --from-literal=password=<mot-de-passe>
```

> Le secret doit exister avant que Flux réconcilie le Cluster CNPG,
> sinon le cluster reste en erreur.

---

## Monitoring

Le `podMonitor` est activé — Prometheus scrape automatiquement les métriques PostgreSQL.
Un dashboard Grafana est déployé dans le namespace `monitoring`.

```bash
# Vérifier que le podMonitor est créé
kubectl get podmonitor -n cnpg-system
```

---

## Commandes utiles

```bash
# Statut de l'opérateur
kubectl get pods -n cnpg-system

# Statut des clusters PostgreSQL
kubectl get cluster -A

# Détail d'un cluster
kubectl describe cluster warap-db -n warap

# Se connecter au primary (debug)
kubectl exec -it warap-db-1 -n warap -- psql -U warap -d warap

# Voir les services créés par CNPG
kubectl get svc -n warap
```

---

## Backup via Longhorn (Volume Snapshots)

CNPG utilise l'API `VolumeSnapshot` de Kubernetes pour déclencher des snapshots Longhorn
directement sur les PVCs PostgreSQL. Pas de transfert réseau, pas d'agent externe.

### Prérequis

Longhorn crée automatiquement une `VolumeSnapshotClass` nommée `longhorn` :

```bash
kubectl get volumesnapshotclass
```

### Configuration dans le Cluster CNPG

```yaml
spec:
  backup:
    volumeSnapshot:
      className: longhorn
```

### Backup planifié (ScheduledBackup)

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: <app>-db-backup
  namespace: <app>
spec:
  schedule: "0 2 * * *"   # Tous les jours à 2h
  backupOwnerReference: self
  cluster:
    name: <app>-db
  method: volumeSnapshot
```

### Commandes utiles

```bash
# Voir les backups planifiés
kubectl get scheduledbackup -n warap

# Voir les backups effectués
kubectl get backup -n warap

# Déclencher un backup manuel
kubectl cnpg backup warap-db -n warap --method volumeSnapshot

# Voir les VolumeSnapshots créés par CNPG
kubectl get volumesnapshot -n warap
```

---

## Restauration depuis un Volume Snapshot

### Étape 1 — Identifier le snapshot à restaurer

```bash
# Lister les VolumeSnapshots disponibles
kubectl get volumesnapshot -n warap

# Exemple de sortie :
# NAME                          READYTOUSE   SOURCEPVC        AGE
# warap-db-1-20260403020000     true         warap-db-1       1d
```

### Étape 2 — Supprimer le cluster existant (si nécessaire)

> Si tu restaures sur le même cluster, il faut d'abord le supprimer.
> Flux va le recréer — suspends la réconciliation d'abord.

```bash
# Suspendre Flux pour éviter qu'il recrée tout de suite
flux suspend kustomization warap -n flux-system

# Supprimer le cluster existant
kubectl delete cluster warap-db -n warap

# Attendre que les pods disparaissent
kubectl get pods -n warap -w
```

### Étape 3 — Créer un fichier de restauration

Crée un fichier temporaire `restore.yaml` (ne pas commiter) :

```yaml
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: warap-db
  namespace: warap
spec:
  instances: 1
  storage:
    size: 5Gi
  bootstrap:
    recovery:
      volumeSnapshots:
        storage:
          name: warap-db-1-20260403020000   # nom du snapshot à restaurer
          kind: VolumeSnapshot
          apiGroup: snapshot.storage.k8s.io
  backup:
    volumeSnapshot:
      className: longhorn
```

```bash
kubectl apply -f restore.yaml
```

### Étape 4 — Vérifier la restauration

```bash
# Suivre le statut du cluster
kubectl get cluster warap-db -n warap -w

# Vérifier les données
kubectl exec -it warap-db-1 -n warap -- psql -U warap -d warap -c "\dt"
```

### Étape 5 — Remettre Flux en marche

Une fois la restauration validée, remplace le `postgres.yaml` de base (initdb → recovery),
puis reprends la réconciliation :

```bash
flux resume kustomization warap -n flux-system
```

---

## Ajouter un nouveau cluster PostgreSQL

Pour chaque nouvelle application qui a besoin d'une base, créer les fichiers suivants :

**1. `apps/base/<app>/postgres.yaml`**

```yaml
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: <app>-db
  namespace: <app>
spec:
  instances: 1
  storage:
    size: 5Gi
  bootstrap:
    initdb:
      database: <app>
      owner: <app>
      secret:
        name: <app>-db-secret
```

**2. Créer le namespace et le secret manuellement**

```bash
kubectl create namespace <app>

kubectl create secret generic <app>-db-secret \
  --namespace <app> \
  --from-literal=username=<app> \
  --from-literal=password=<mot-de-passe>
```

**3. Ajouter dans `apps/base/<app>/kustomization.yaml`**

```yaml
resources:
  - namespace.yaml
  - postgres.yaml
```

**4. Ajouter dans `clusters/cluster-payload/apps.yaml`**

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: <app>
  namespace: flux-system
spec:
  dependsOn:
    - name: cnpg
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/cluster-payload/<app>
  prune: true
```

> Le `dependsOn: cnpg` garantit que l'opérateur est prêt avant de créer le cluster.

**5. Mettre à jour le tableau des clusters dans ce README**

---

## Dépannage

| Symptome | Cause probable | Solution |
|----------|---------------|----------|
| Cluster en erreur au démarrage | Secret `warap-db-secret` manquant | Créer le secret avant de pusher |
| Pod en `CrashLoopBackOff` | Problème de PVC | Vérifier Longhorn et le StorageClass |
| App ne peut pas se connecter | Mauvais hostname | Utiliser `warap-db-rw.warap.svc.cluster.local` |
| Cluster en `Configuring` | Bootstrap en cours | Attendre 1-2 minutes |
