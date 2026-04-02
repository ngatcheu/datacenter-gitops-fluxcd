# Cilium LB IPAM — LoadBalancer IP Management

## Structure

```
infrastructure/base/cilium-lb/
├── ippool.yaml         # Pool d'IPs de base (surchargé par cluster)
├── l2policy.yaml       # Politique d'annonce ARP (L2)
├── rbac.yaml           # ClusterRole/ClusterRoleBinding pour les leases
└── kustomization.yaml

infrastructure/cluster-rancher/cilium-lb/
├── kustomization.yaml
└── ippool-patch.yaml   # Override pool : 192.168.1.221 - 192.168.1.230

infrastructure/cluster-payload/cilium-lb/
├── kustomization.yaml
└── ippool-patch.yaml   # Override pool : 192.168.1.231 - 192.168.1.250
```

---

## Architecture

```
Service type: LoadBalancer cree dans Kubernetes
        │
        ▼
CiliumLoadBalancerIPPool
(plage par cluster, voir ci-dessous)
        │
        ▼
IP assignee (ex: 192.168.1.221 pour Traefik sur cluster-rancher)
        │
        ▼
CiliumL2AnnouncementPolicy
→ Annonce ARP sur eth0 depuis tous les nodes
→ Le LAN resout l'IP via ARP
```

---

## Pools d'IPs par cluster

| Cluster | Plage | IPs disponibles | Traefik |
|---------|-------|-----------------|---------|
| cluster-rancher | `192.168.1.221 - 192.168.1.230` | 10 | `192.168.1.221` |
| cluster-payload | `192.168.1.231 - 192.168.1.250` | 20 | `192.168.1.231` |

> Les plages sont separees pour eviter tout overlap (conflit ARP sur le LAN).

---

## Prerequis Cilium

L2 Announcements doit etre active dans la configuration Cilium :

```bash
kubectl get configmap -n kube-system cilium-config \
  -o jsonpath='{.data.enable-l2-announcements}'
# Doit retourner : true
```

Si absent ou `false` :

```bash
kubectl patch configmap -n kube-system cilium-config \
  --type merge \
  -p '{"data":{"enable-l2-announcements":"true"}}'

kubectl rollout restart daemonset -n kube-system cilium
```

---

## Points d'attention

### Cluster sans worker (control-plane only)

Si le cluster n'a pas de nodes worker dedies (tous les nodes sont `control-plane`),
le `nodeSelector` de la L2 policy doit etre ouvert a tous les nodes :

```yaml
# l2policy.yaml
nodeSelector:
  matchLabels: {}   # Tous les nodes
```

Avec un filtre `DoesNotExist` sur `control-plane`, aucun node n'annonce les IPs
et les services LoadBalancer restent injoignables depuis le LAN.

### Interface reseau

Verifier le nom de l'interface sur les nodes :

```bash
ssh root@<node-ip> ip link show
```

L'interface configuree dans `l2policy.yaml` est `eth0`. Adapter si necessaire (`ens18`, `ens192`, etc.).

---

## Applications exposees

### cluster-rancher (Traefik : `192.168.1.221`)

| App | URL |
|-----|-----|
| Rancher | `https://rancher.homelab.local` |
| ArgoCD | `https://argocd-rancher.homelab.local` |

### cluster-payload (Traefik : `192.168.1.231`)

| App | URL |
|-----|-----|
| ArgoCD | `https://argocd.homelab.local` |
| Vault | `https://vault.homelab.local` |
| n8n | `https://n8n.homelab.local` |
| phpIPAM | `https://phpipam.homelab.local` |
| Grafana | `https://grafana.homelab.local` |
| Prometheus | `https://prometheus.homelab.local` |
| Alertmanager | `https://alertmanager.homelab.local` |
| Longhorn | `https://longhorn.homelab.local` |
| Kafka UI | `https://kafka.homelab.local` |
| Adminer | `https://adminer.homelab.local` |

### hosts Windows (`C:\Windows\System32\drivers\etc\hosts`)

```
192.168.1.221  rancher.homelab.local
192.168.1.221  argocd-rancher.homelab.local

192.168.1.231  traefik.homelab.local
192.168.1.231  argocd.homelab.local
192.168.1.231  grafana.homelab.local
192.168.1.231  prometheus.homelab.local
192.168.1.231  alertmanager.homelab.local
192.168.1.231  vault.homelab.local
192.168.1.231  n8n.homelab.local
192.168.1.231  phpipam.homelab.local
192.168.1.231  longhorn.homelab.local
192.168.1.231  kafka.homelab.local
192.168.1.231  adminer.homelab.local
```

---

## Commandes utiles

```bash
# Verifier le pool et les IPs allouees
kubectl get ciliumloadbalancerippool homelab-pool -o yaml

# Verifier la politique L2
kubectl get ciliuml2announcementpolicy default -o yaml

# Voir les leases Cilium (leader election L2)
kubectl get leases -n kube-system | grep cilium-l2

# Verifier les logs Cilium
kubectl logs -n kube-system -l k8s-app=cilium --tail=50 | grep -i "l2\|arp\|announce\|lease"

# IPs assignees aux services LoadBalancer
kubectl get svc -A --field-selector spec.type=LoadBalancer

# Tester la connectivite depuis Windows (PowerShell)
arp -a | findstr 192.168.1.221
curl.exe -k https://rancher.homelab.local -o nul -w "%{http_code}"
```

---

## Depannage

| Symptome | Cause probable | Solution |
|----------|---------------|----------|
| Service LoadBalancer reste en `<pending>` | Pool vide ou L2 non active | Verifier `CiliumLoadBalancerIPPool` et `enable-l2-announcements: true` |
| IP assignee mais pas joignable depuis le LAN | Aucun node ne selectionne par la L2 policy | Verifier `nodeSelector` — si cluster sans worker, utiliser `matchLabels: {}` |
| Ping repond "Destination Host Unreachable" | Normal si ICMP non gere par Cilium eBPF | Tester avec `curl` plutot que `ping` |
| `leases is forbidden` dans les logs | RBAC manquant | Appliquer `rbac.yaml` |
| 404 sur l'IP directe de Traefik | Normal — Traefik requiert un Host header | Utiliser le nom DNS (`rancher.homelab.local`) |
| IP repondue depuis un mauvais node | Comportement normal | Cilium fait la leader election, un seul node annonce l'IP a la fois |
