# cert-manager — Gestion des certificats TLS

## Structure

```
infrastructure/base/
├── cert-manager/                   # Installation cert-manager
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── source.yaml                 # HelmRepository jetstack
│   └── release.yaml                # HelmRelease cert-manager
│
└── cert-manager-issuers/           # CA et Issuers (deploye apres cert-manager)
    ├── kustomization.yaml
    ├── selfsigned-issuer.yaml
    ├── homelab-ca.yaml             # CA racine + ClusterIssuer
    └── wildcard-certificate.yaml
```

---

## Architecture de la chaine de certificats

```
selfsigned-issuer (ClusterIssuer)
    └── homelab-ca (Certificate, namespace: cert-manager)
            └── homelab-ca-secret (Secret avec la cle CA)
                    └── homelab-ca-issuer (ClusterIssuer)
                            └── wildcard-tls (Certificate *.homelab.local, namespace: traefik)
```

La CA racine (`homelab-ca`) est auto-signee. Tous les certificats du homelab sont emis par `homelab-ca-issuer`. Pour que les navigateurs et systemes fassent confiance aux certificats, la CA racine doit etre importee dans chaque magasin de confiance (voir section 3).

---

## 1. Activation sur un cluster

Ajouter dans `clusters/<votre-cluster>/infrastructure.yaml` :

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-manager
  namespace: flux-system
spec:
  interval: 1h
  retryInterval: 1m
  timeout: 10m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/cluster-payload/cert-manager
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
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/cluster-payload/cert-manager-issuers
  prune: true
  wait: true
```

---

## 2. Travailler avec cert-manager

### Les 3 objets principaux

**ClusterIssuer** — definit qui signe les certificats. Disponible dans tous les namespaces (contrairement a `Issuer` qui est limite a un namespace).

**Certificate** — declare le certificat souhaite. cert-manager le cree et le renouvelle automatiquement avant expiration.

**Secret** — cree automatiquement par cert-manager. Contient `tls.crt` et `tls.key`.

---

### Certificat wildcard (setup actuel)

Un seul certificat couvre toutes les apps `*.homelab.local`. Il est stocke dans le namespace `traefik` et reference directement par le Gateway listener `websecure`.

```
*.homelab.local → Secret wildcard-tls (namespace: traefik)
→ Traefik l'utilise pour toutes les apps via le Gateway
```

Avec ce setup, **aucune action supplementaire n'est necessaire** pour exposer une nouvelle app — l'HTTPRoute suffit.

---

### Creer un certificat dedie pour une app

Si tu as besoin d'un certificat specifique (hors wildcard) :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-app-tls
  namespace: mon-app          # doit etre dans le namespace de l'app
spec:
  secretName: mon-app-tls     # cert-manager cree ce Secret automatiquement
  duration: 8760h             # 1 an
  renewBefore: 720h           # renouvellement 30 jours avant expiration
  dnsNames:
    - mon-app.homelab.local
  issuerRef:
    name: homelab-ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```

---

### Issuers disponibles

| Issuer | Type | Usage |
|--------|------|-------|
| `selfsigned-issuer` | ClusterIssuer | Creation de la CA uniquement |
| `homelab-ca-issuer` | ClusterIssuer | Signer les certificats pour *.homelab.local |

---

### Commandes utiles

```bash
# Lister tous les certificats
kubectl get certificate -A

# Voir les etapes d'emission (CertificateRequest)
kubectl get certificaterequest -A

# Diagnostiquer un certificat en erreur
kubectl describe certificate <name> -n <namespace>
kubectl describe certificaterequest -n <namespace>

# Forcer le renouvellement immediat
kubectl delete secret <secretName> -n <namespace>
# cert-manager recrée le Secret automatiquement

# Verifier le contenu d'un certificat
kubectl get secret wildcard-tls -n traefik \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```

---

## 3. Chaine de confiance — importer la CA racine

La CA `homelab-ca` etant auto-signee, les navigateurs et OS ne lui font pas confiance par defaut. Il faut importer le certificat CA dans chaque magasin de confiance.

### Exporter la CA depuis le cluster

```bash
kubectl get secret -n cert-manager homelab-ca-secret \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > homelab-ca.crt

# Verifier le contenu
openssl x509 -in homelab-ca.crt -noout -subject -dates
```

---

### Windows

**Via l'interface graphique :**

1. Double-clic sur `homelab-ca.crt`
2. Clic sur **Installer le certificat** → **Ordinateur local**
3. **Placer tous les certificats dans le magasin suivant** → Parcourir → **Autorites de certification racines de confiance**
4. Terminer → Oui
5. Redemarrer le navigateur

**Via PowerShell (en tant qu'administrateur) :**

```powershell
Import-Certificate -FilePath "homelab-ca.crt" -CertStoreLocation Cert:\LocalMachine\Root
```

---

### Linux — Debian / Ubuntu

```bash
sudo cp homelab-ca.crt /usr/local/share/ca-certificates/homelab-ca.crt
sudo update-ca-certificates
```

### Linux — RHEL / CentOS / Rocky

```bash
sudo cp homelab-ca.crt /etc/pki/ca-trust/source/anchors/homelab-ca.crt
sudo update-ca-trust
```

---

### Firefox

Firefox maintient son propre magasin de certificats, independant de celui du systeme d'exploitation.

1. Ouvrir `about:preferences#privacy`
2. Section **Certificats** → **Afficher les certificats**
3. Onglet **Autorites** → **Importer**
4. Selectionner `homelab-ca.crt`
5. Cocher **Confirmer cette AC pour identifier des sites web**

---

### Nodes Kubernetes

Necessaire pour les applications qui effectuent des appels HTTPS internes vers d'autres services du cluster.

```bash
# Sur chaque node RKE2
sudo cp homelab-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

---

### Automatisation Ansible

```yaml
- name: Recuperer le certificat CA homelab
  command: >
    kubectl get secret -n cert-manager homelab-ca-secret
    -o jsonpath='{.data.tls\.crt}'
  register: ca_b64
  delegate_to: localhost

- name: Importer la CA sur tous les nodes
  copy:
    content: "{{ ca_b64.stdout | b64decode }}"
    dest: /etc/pki/ca-trust/source/anchors/homelab-ca.crt
  notify: update-ca-trust

handlers:
  - name: update-ca-trust
    command: update-ca-trust
```

---

### Verifier que la confiance est en place

```bash
# Depuis un node Linux
curl -v https://grafana.homelab.local 2>&1 | grep -i "verify\|ssl\|cert"
# Doit afficher : SSL certificate verify ok
```