# Traefik — Ingress Controller avec Kubernetes Gateway API

## Structure

```
infrastructure/base/
├── traefik/
│   ├── namespace.yaml          # Namespace traefik
│   ├── source.yaml             # HelmRepository helm.traefik.io/traefik
│   ├── release.yaml            # HelmRelease traefik (chart 39.0.7)
│   └── kustomization.yaml
│
└── gateway-api/
    ├── source.yaml             # GitRepository kubernetes-sigs/gateway-api v1.2.1
    └── kustomization.yaml

infrastructure/cluster-payload/
└── traefik/
    └── kustomization.yaml      # Pointe vers ../../base/traefik
```

---

## Architecture

```
Internet / LAN
      │
      ▼
LoadBalancer IP: 192.168.1.211  (Cilium LB IPAM)
      │
      ▼
Traefik (2 replicas, namespace: traefik)
      │
      ├── Listener web      (port 80)  → redirect HTTPS permanent
      └── Listener websecure (port 443) → TLS Terminate (wildcard-tls)
                │
                ▼
         HTTPRoutes (tous namespaces)
                │
                ▼
         Services applicatifs
```

Traefik fonctionne exclusivement en mode **Kubernetes Gateway API** (pas d'Ingress). Il cree automatiquement une `GatewayClass` et un `Gateway` nomme `traefik-gateway`.

---

## Ordre de deploiement (FluxCD)

```
gateway-api-source  →  gateway-api (CRDs)  →  traefik
                                               (dependsOn: gateway-api + kube-prometheus-stack)
```

Les CRDs Gateway API doivent etre installes avant Traefik. Deux Kustomizations FluxCD gerent cela :

```yaml
# Etape 1 : deploie le GitRepository gateway-api dans le cluster
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: gateway-api-source
spec:
  path: ./infrastructure/cluster-payload/gateway-api
  wait: true

# Etape 2 : installe les CRDs depuis le repo kubernetes-sigs/gateway-api
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: gateway-api
spec:
  dependsOn:
    - name: gateway-api-source
  sourceRef:
    kind: GitRepository
    name: gateway-api       # pointe vers le repo kubernetes-sigs
  path: ./config/crd/standard
  prune: false              # ne jamais supprimer les CRDs automatiquement
```

---

## Configuration cle (release.yaml)

### Providers

```yaml
providers:
  kubernetesIngress:
    enabled: false    # Ingress desactive, on utilise uniquement Gateway API
  kubernetesGateway:
    enabled: true
```

### Gateway et listeners

```yaml
gatewayClass:
  enabled: true         # Cree la GatewayClass "traefik"

gateway:
  enabled: true
  listeners:
    web:
      port: 8000
      protocol: HTTP
      namespacePolicy:
        from: All       # HTTPRoutes acceptes depuis tous les namespaces
    websecure:
      port: 8443
      protocol: HTTPS
      namespacePolicy:
        from: All
      certificateRefs:
        - name: wildcard-tls
          namespace: traefik    # Secret TLS genere par cert-manager
      mode: Terminate           # TLS termine au niveau de Traefik
```

### Redirection HTTP → HTTPS

```yaml
ports:
  web:
    port: 8000
    exposedPort: 80
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true   # 301 permanent
  websecure:
    port: 8443
    exposedPort: 443
```

### Service LoadBalancer

```yaml
service:
  type: LoadBalancer    # IP assignee par Cilium LB IPAM (192.168.1.211)
```

---

## Exposer une nouvelle application

Pour exposer une app, creer un `HTTPRoute` dans le namespace de l'application :

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mon-app
  namespace: mon-app
spec:
  parentRefs:
    - name: traefik-gateway
      namespace: traefik    # toujours pointer vers traefik-gateway dans traefik
  hostnames:
    - mon-app.homelab.local
  rules:
    - backendRefs:
        - name: mon-app-service   # nom du Service Kubernetes
          port: 8080
```

Le certificat wildcard `*.homelab.local` s'applique automatiquement via le listener `websecure` — aucune configuration TLS supplementaire n'est necessaire dans l'HTTPRoute.

### Ajouter l'entree DNS (Windows hosts)

```
192.168.1.211   mon-app.homelab.local
```

Fichier : `C:\Windows\System32\drivers\etc\hosts`

---

## HTTPRoutes existantes

| Application | Namespace | Hostname | Service | Port |
|-------------|-----------|----------|---------|------|
| grafana | monitoring | grafana.homelab.local | kube-prometheus-stack-grafana | 80 |
| prometheus | monitoring | prometheus.homelab.local | kube-prometheus-stack-prometheus | 9090 |
| alertmanager | monitoring | alertmanager.homelab.local | kube-prometheus-stack-alertmanager | 9093 |
| longhorn | longhorn-system | longhorn.homelab.local | longhorn-frontend | 80 |
| argocd | argocd | argocd.homelab.local | argocd-server | 80 |
| gitlab | gitlab | gitlab.homelab.local | gitlab-webservice-default | 8181 |
| gitlab-registry | gitlab | registry.homelab.local | gitlab-registry | 5000 |
| vault | vault | vault.homelab.local | vault | 8200 |
| n8n | n8n | n8n.homelab.local | n8n | 5678 |
| phpipam | phpipam | phpipam.homelab.local | phpipam-web | 80 |

---

## Dashboard Traefik

Le dashboard est accessible via IngressRoute integre au chart :

```
http://traefik.homelab.local/dashboard/
```

> Le dashboard est expose sur le listener `websecure` (HTTPS). Le certificat wildcard `*.homelab.local` s'applique automatiquement.

---

## Commandes utiles

```bash
# Etat du HelmRelease
kubectl get helmrelease -n traefik traefik

# IP LoadBalancer assignee
kubectl get svc -n traefik traefik

# Voir le Gateway et son etat
kubectl get gateway -n traefik
kubectl describe gateway -n traefik traefik-gateway

# Voir toutes les HTTPRoutes
kubectl get httproute -A

# Verifier qu'une HTTPRoute est bien attachee au Gateway
kubectl describe httproute <name> -n <namespace>
# Chercher : Parents: traefik/traefik-gateway - Accepted: True

# Logs Traefik
kubectl logs -n traefik -l app.kubernetes.io/name=traefik --tail=50

# Redemarrer Traefik
kubectl rollout restart deployment -n traefik traefik
```

---

## Depannage

| Symptome | Cause probable | Solution |
|----------|---------------|----------|
| 404 sur toutes les routes | HTTPRoute reference mauvais gateway | Verifier `name: traefik-gateway` et `namespace: traefik` dans `parentRefs` |
| HTTPS ne repond pas | Secret `wildcard-tls` absent | `kubectl get secret -n traefik wildcard-tls` |
| Pas d'IP LoadBalancer | Cilium LB IPAM non configure | Verifier `CiliumLoadBalancerIPPool` et L2 policy |
| HelmRelease en erreur | CRDs Gateway API manquantes | Verifier que `gateway-api` Kustomization est `Ready` avant Traefik |
| HTTPRoute non acceptee | Namespace non autorise | Verifier `namespacePolicy: from: All` dans le listener |