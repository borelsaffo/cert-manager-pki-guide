
# 🔐 Utiliser Cert-Manager avec une PKI interne (CA interne)

Ce guide montre comment brancher Cert-Manager sur une autorité de certification privée (PKI interne), au lieu de Let's Encrypt. C'est utile pour les environnements d'entreprise ou en local sans Internet.

---

## ⚙️ 1. Brancher Cert-Manager sur une PKI interne

Cert-Manager peut s’interfacer avec plusieurs types d'autorités de certification :

### A. Si la PKI expose une API de type **ACME**
→ Utiliser un `ClusterIssuer` avec `acme`.

### B. Si la PKI expose une **API HTTP privée**
→ Utiliser un `CA` Issuer ou `Venafi` Issuer.

### C. Si vous avez une chaîne de certificat **(Root / Intermediate CA)**
→ Créer un `CA` Issuer local avec le certificat + clé privée.

---

### 🧪 Exemple : Utilisation d'un ClusterIssuer basé sur une CA locale

#### Supposons que vous avez :

- Un certificat intermédiaire : `ca.crt`
- Sa clé privée : `ca.key`

#### 1. Créer un Secret Kubernetes :

```bash
kubectl create namespace cert-manager

kubectl create secret tls my-ca-key-pair \
  --cert=ca.crt \
  --key=ca.key \
  -n cert-manager
```

#### 2. Définir un `ClusterIssuer` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-pki-clusterissuer
spec:
  ca:
    secretName: my-ca-key-pair
```

---

## 📡 2. Commandes utiles avec Cert-Manager

### 🔍 Lister les certificats :

```bash
kubectl get certificates -A
```

### 📝 Voir les demandes de certificats :

```bash
kubectl get certificaterequests -A
```

### 🔎 Inspecter un certificat :

```bash
kubectl describe certificate <nom> -n <namespace>
```

### 📄 Lister les Issuers :

```bash
kubectl get clusterissuers
kubectl get issuers -A
```

---

## 📥 Exemple de certificat géré par Cert-Manager

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: default
spec:
  secretName: myapp-tls
  duration: 2160h # 90 jours
  renewBefore: 360h # 15 jours avant expiration
  commonName: myapp.local
  dnsNames:
    - myapp.local
  issuerRef:
    name: my-pki-clusterissuer
    kind: ClusterIssuer
```

Appliquer avec :

```bash
kubectl apply -f myapp-cert.yaml
```

---

## 🔧 Interagir via l'API Kubernetes

Cert-Manager expose ses ressources via l’API K8s :

### Exemple avec `kubectl proxy` :

```bash
kubectl proxy
```

Puis :

```bash
curl http://localhost:8001/apis/cert-manager.io/v1/namespaces/default/certificates
```

---

## 🧩 Intégrations avancées

Cert-Manager peut aussi être intégré avec :

- **Microsoft ADCS** (via webhook ou via un intermédiaire HTTP)
- **EJBCA**
- **HashiCorp Vault PKI** (plugin officiel cert-manager + Vault)

Souhaitez-vous un guide spécifique pour l’un de ces systèmes ?

---
