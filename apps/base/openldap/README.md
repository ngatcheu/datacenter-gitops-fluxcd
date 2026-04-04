# OpenLDAP — Annuaire LDAP + phpLDAPadmin

## Structure

```
apps/base/openldap/
├── namespace.yaml          # Namespace openldap
├── source.yaml             # HelmRepository ldap-stack
├── release.yaml            # HelmRelease ldap-stack (openldap + phpldapadmin)
└── kustomization.yaml

apps/cluster-payload/openldap/
├── secrets.yaml            # Credentials admin OpenLDAP
├── httproute.yaml          # ldap.homelab.local → phpldapadmin
└── kustomization.yaml
```

---

## Chart

**ldap-stack** v1.3.1 — `https://start-codex.github.io/ldap-stack-helm-chart`

Chart tout-en-un incluant :
- **OpenLDAP** (startcodex/openldap:2.0.0)
- **phpLDAPadmin** (osixia/phpldapadmin)
- ~~Keycloak~~ (desactive)

---

## Configuration

| Parametre | Valeur |
|-----------|--------|
| Organisation | Home Lab |
| Domain | `homelab.local` |
| Base DN | `dc=homelab,dc=local` |
| Admin DN | `cn=admin,dc=homelab,dc=local` |
| TLS intra-cluster | Desactive (Traefik gere le TLS externe) |
| Stockage | Longhorn Retain (2Gi data + 100Mi config) |

### OUs creees au bootstrap

```
dc=homelab,dc=local
├── ou=users      # Utilisateurs (inetOrgPerson)
├── ou=groups     # Groupes (groupOfNames)
└── ou=services   # Comptes de service
```

---

## Acces

| Service | URL |
|---------|-----|
| phpLDAPadmin | `https://ldap.homelab.local` |
| OpenLDAP (intra-cluster) | `openldap-ldap-stack-openldap.openldap.svc.cluster.local:389` |

### Login phpLDAPadmin

- **Login DN** : `cn=admin,dc=homelab,dc=local`
- **Password** : voir `secrets.yaml`

---

## Secret

```yaml
# apps/cluster-payload/openldap/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: openldap-secret
  namespace: openldap
stringData:
  admin-password: "<mot-de-passe>"
  config-password: "<mot-de-passe>"
```

> Ne pas commiter le mot de passe en clair en production. Utiliser SealedSecrets ou Vault Secret Operator.

---

## Connexion depuis une application

Pour connecter une application (Grafana, GitLab, ArgoCD...) a OpenLDAP :

```
Host:    openldap-ldap-stack-openldap.openldap.svc.cluster.local
Port:    389
Base DN: dc=homelab,dc=local
Bind DN: cn=admin,dc=homelab,dc=local
```

---

## Groupes Rancher

Les groupes suivants sont utilises par Rancher pour les role bindings :

| Groupe | Role Rancher | DN |
|--------|-------------|-----|
| `devops` | admin global | `cn=devops,ou=groups,dc=homelab,dc=local` |
| `middleware` | admin global | `cn=middleware,ou=groups,dc=homelab,dc=local` |
| `usilo` | user | `cn=usilo,ou=groups,dc=homelab,dc=local` |

---

## Ajouter un utilisateur

### Via phpLDAPadmin (interface graphique)

1. Ouvrir `https://ldap.homelab.local`
2. Login → `cn=admin,dc=homelab,dc=local` / mot de passe admin
3. Dans l'arbre a gauche → `ou=users` → **Create a child entry**
4. Choisir **Generic: User Account**
5. Remplir les champs : `uid`, `First name`, `Last name`, `Email`, `Password`
6. Cliquer **Commit** pour valider

**Ajouter l'utilisateur a un groupe :**

1. Cliquer sur le groupe cible sous `ou=groups`
2. **Add new attribute** → `member` → entrer le DN complet de l'utilisateur
3. Cliquer **Update Object**

### Via ldapadd (depuis kubectl)

```bash
LDAP_PASS=$(kubectl get secret -n openldap openldap-secret -o jsonpath='{.data.admin-password}' | base64 -d)

cat <<EOF | kubectl exec -i -n openldap deploy/openldap-ldap-stack-openldap -- \
  ldapadd -x -H ldap://localhost:389 -D "cn=admin,dc=homelab,dc=local" -w "$LDAP_PASS"
dn: uid=jdoe,ou=users,dc=homelab,dc=local
objectClass: inetOrgPerson
uid: jdoe
cn: John Doe
sn: Doe
mail: jdoe@homelab.local
userPassword: <password>
EOF
```

### Ajouter un utilisateur a un groupe existant

```bash
LDAP_PASS=$(kubectl get secret -n openldap openldap-secret -o jsonpath='{.data.admin-password}' | base64 -d)

cat <<EOF | kubectl exec -i -n openldap deploy/openldap-ldap-stack-openldap -- \
  ldapmodify -x -H ldap://localhost:389 -D "cn=admin,dc=homelab,dc=local" -w "$LDAP_PASS"
dn: cn=devops,ou=groups,dc=homelab,dc=local
changetype: modify
add: member
member: uid=jdoe,ou=users,dc=homelab,dc=local
EOF
```

## Ajouter un groupe

```bash
LDAP_PASS=$(kubectl get secret -n openldap openldap-secret -o jsonpath='{.data.admin-password}' | base64 -d)

cat <<EOF | kubectl exec -i -n openldap deploy/openldap-ldap-stack-openldap -- \
  ldapadd -x -H ldap://localhost:389 -D "cn=admin,dc=homelab,dc=local" -w "$LDAP_PASS"
dn: cn=nouveau-groupe,ou=groups,dc=homelab,dc=local
objectClass: groupOfNames
cn: nouveau-groupe
member: uid=jdoe,ou=users,dc=homelab,dc=local
EOF
```

> **Note** : `groupOfNames` exige au minimum un `member`. Utiliser un utilisateur existant comme membre initial.

---

## Commandes utiles

```bash
# Etat des pods
kubectl get pods -n openldap

# Etat du HelmRelease
kubectl get helmrelease -n openldap openldap

# Tester la connexion LDAP depuis le cluster
kubectl run ldap-test --rm -it --image=bitnami/openldap --restart=Never -- \
  ldapsearch -x -H ldap://openldap-ldap-stack-openldap.openldap.svc.cluster.local \
  -D "cn=admin,dc=homelab,dc=local" -w <password> -b "dc=homelab,dc=local"

# Voir les logs OpenLDAP
kubectl logs -n openldap -l app.kubernetes.io/component=openldap --tail=50

# Voir les logs phpLDAPadmin
kubectl logs -n openldap -l app.kubernetes.io/component=phpldapadmin --tail=50
```

---

## Depannage

| Symptome | Cause probable | Solution |
|----------|---------------|----------|
| HTTP 500 sur phpLDAPadmin | Mauvais nom de service dans HTTPRoute | Verifier `kubectl get svc -n openldap` |
| Connexion LDAP refusee | Mauvais Bind DN ou password | Verifier `cn=admin,dc=homelab,dc=local` |
| Pod OpenLDAP en CrashLoop | Secret manquant ou mauvaise cle | Verifier `openldap-secret` avec les cles `admin-password` et `config-password` |
| PVC en Pending | StorageClass indisponible | Verifier `longhorn-retain` sur le cluster |