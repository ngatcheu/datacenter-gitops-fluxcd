# Cilium LB IPAM — LoadBalancer IP Management

## Structure

```
infrastructure/base/cilium-lb/
├── ippool.yaml         # Pool d'IPs pour les services LoadBalancer
├── l2policy.yaml       # Politique d'annonce ARP (L2)
├── rbac.yaml           # ClusterRole/ClusterRoleBinding pour les leases
└── kustomization.yaml
```

---

## Pourquoi Cilium LB IPAM

Cilium, deja deploye comme CNI sur le cluster RKE2, integre nativement la gestion des IPs LoadBalancer. Il remplace des solutions externes comme MetalLB. Lorsqu'un Service de type `LoadBalancer` est cree, Cilium lui assigne automatiquement une IP depuis le pool configure et l'annonce sur le reseau local via ARP (L2).

---

## Architecture

```
Service type: LoadBalancer cree dans Kubernetes
        │
        ▼
CiliumLoadBalancerIPPool
(plage: 192.168.1.211 - 192.168.1.250)
        │
        ▼
IP assignee (ex: 192.168.1.211 pour Traefik)
        │
        ▼
CiliumL2AnnouncementPolicy
→ Annonce ARP sur eth0 depuis les nodes worker
→ Le LAN resout l'IP via ARP
```

---

## Prerequis Cilium

L2 Announcements doit etre active dans la configuration Cilium. Verifier dans le ConfigMap :

```bash
kubectl get configmap -n kube-system cilium-config \
  -o jsonpath='{.data.enable-l2-announcements}'
# Doit retourner : true
```

Si la valeur est absente ou `false`, activer via le role Ansible RKE2 ou patcher manuellement :

```bash
kubectl patch configmap -n kube-system cilium-config \
  --type merge \
  -p '{"data":{"enable-l2-announcements":"true"}}'

kubectl rollout restart daemonset -n kube-system cilium
```

---

## Configuration

### Pool d'IPs (ippool.yaml)

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: homelab-pool
spec:
  blocks:
    - start: "192.168.1.211"
      stop:  "192.168.1.250"
```

Plage disponible : **192.168.1.211 → 192.168.1.250** (40 IPs).

Pour modifier la plage, editer `start` et `stop` en s'assurant que ces IPs ne sont pas deja utilisees sur le reseau local.

---

### Politique d'annonce L2 (l2policy.yaml)

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default
spec:
  interfaces:
    - eth0            # Adapter si l'interface reseau est differente (ex: ens18, ens192)
  loadBalancerIPs: true
  externalIPs: true
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist  # Annonce uniquement depuis les nodes worker
```

> Verifier le nom de l'interface reseau sur les nodes : `ip addr show` ou `ip link`

---

### RBAC (rbac.yaml)

Cilium a besoin de creer des `leases` dans `coordination.k8s.io` pour elire le node qui annonce l'IP (leader election L2). Sans ce RBAC, Cilium ne peut pas acquerir les leases et l'annonce ARP echoue.

```yaml
# ClusterRole : acces aux leases
# ClusterRoleBinding : lie au ServiceAccount cilium dans kube-system
```

Erreur typique sans ce RBAC :
```
leases.coordination.k8s.io is forbidden: User "system:serviceaccount:kube-system:cilium"
cannot get resource "leases"
```

---

## IPs actuellement assignees

| Service | Namespace | IP |
|---------|-----------|-----|
| traefik | traefik | 192.168.1.211 |

Pour voir toutes les IPs assignees :

```bash
kubectl get svc -A --field-selector spec.type=LoadBalancer
```

---

## Commandes utiles

```bash
# Verifier le pool et les IPs allouees
kubectl get ciliumloadbalancerippool homelab-pool -o yaml

# Verifier la politique L2
kubectl get ciliuml2announcementpolicy default -o yaml

# Voir les leases Cilium (leader election L2)
kubectl get leases -n kube-system | grep cilium

# Verifier les logs Cilium pour les annonces ARP
kubectl logs -n kube-system -l k8s-app=cilium --tail=50 | grep -i "l2\|arp\|announce\|lease"

# Tester qu'une IP est bien annoncee sur le LAN (depuis Windows)
ping 192.168.1.211
arp -a | findstr 192.168.1.211
```

---

## Depannage

| Symptome | Cause probable | Solution |
|----------|---------------|----------|
| Service LoadBalancer reste en `<pending>` | Pool vide ou L2 non active | Verifier `CiliumLoadBalancerIPPool` et `enable-l2-announcements: true` |
| IP assignee mais pas joignable | Annonce ARP echouee | Verifier logs Cilium, RBAC leases, interface `eth0` |
| `leases is forbidden` dans les logs | RBAC manquant | Appliquer `rbac.yaml` |
| IP repondue depuis un mauvais node | Comportement normal | Cilium fait la leader election, un seul node annonce l'IP a la fois |
| Ping repond mais depuis une IP inattendue | Conflit ARP sur le LAN | Verifier qu'aucune autre machine n'utilise les IPs du pool |