
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


# ❓ Comment Cert-Manager traite une demande de certificat sans IP ni URL dans un ClusterIssuer de type `ca`

Dans la définition suivante :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-pki-clusterissuer
spec:
  ca:
    secretName: my-ca-key-pair
```

➡️ Il **n'y a pas d'adresse IP, ni de serveur distant** spécifié.  
Alors comment Cert-Manager fonctionne dans ce cas ?

---

## ✅ Réponse : Cert-Manager signe localement avec la clé privée dans le Secret

Ce `ClusterIssuer` est de type `ca`, ce qui signifie que :

1. Le Secret `my-ca-key-pair` contient **le certificat (ca.crt)** et **la clé privée (ca.key)** d’une autorité intermédiaire.
2. Cert-Manager utilise **cette clé privée** pour **signer directement** les demandes de certificats.
3. **Aucune requête externe n’est effectuée** (pas d’appel à un serveur CA distant).
4. Le certificat est généré et signé **en local dans le cluster Kubernetes**.

---

## 📦 Exemple de création du Secret

```bash
kubectl create secret tls my-ca-key-pair \
  --cert=intermediate-ca.crt \
  --key=intermediate-ca.key \
  -n cert-manager
```

- `intermediate-ca.crt` : peut contenir uniquement l'intermédiaire, ou la chaîne complète (intermédiaire + root)
- `intermediate-ca.key` : la clé privée correspondante

---

## 🧠 Ce qu’il faut comprendre

| Élément                      | Rôle                                                                 |
|-----------------------------|----------------------------------------------------------------------|
| `ca.crt` dans le Secret     | Le certificat de l’Intermediate CA utilisé pour signer              |
| `ca.key` dans le Secret     | La clé privée correspondante utilisée pour signer                   |
| ClusterIssuer `ca`          | Utilise ce Secret pour générer localement les certificats           |
| Pas d’IP / URL              | Car Cert-Manager ne contacte personne, tout est fait *dans le cluster* |

---

## 🔗 Et ensuite ?

- Tu peux configurer Cert-Manager pour **inclure la chaîne complète** dans les certificats générés.
- Ou bien utiliser une **CA d’entreprise distante** comme :
  - Microsoft ADCS
  - EJBCA
  - HashiCorp Vault PKI

##

# 🔐 Intégrer CMPv2 avec Cert-Manager dans Kubernetes

Le protocole **CMPv2 (Certificate Management Protocol v2)** est utilisé dans les PKI d’entreprise pour gérer les certificats (délivrance, révocation, renouvellement). Voici comment l’utiliser dans Kubernetes avec Cert-Manager.

---

## ⚙️ Qu’est-ce que CMPv2 ?

CMPv2 est un protocole standardisé pour :

- 🔑 Génération de clés
- 📩 Envoi de CSR (Certificate Signing Request)
- 🛡 Authentification (challenge, signature…)
- 📜 Réception de certificats signés
- 🔁 Renouvellement, révocation, mise à jour

---

## ❗ CMPv2 et Cert-Manager : pas de support natif

Cert-Manager **ne supporte pas directement CMPv2**.  
Mais il peut l’utiliser via un **Webhook ExternalIssuer** personnalisé.

---

## 🧩 Architecture avec CMPv2

[ Cert-Manager ] │ ▼ 
[ External Issuer (webhook) ] │ ▼
[ CMPv2 Gateway / Proxy ] │ ▼
[ CA Interne (EJBCA, ADCS...) ]

---

## 🛠 Étapes pour intégrer CMPv2

### 1. CMP Gateway / Client CMP
Tu as besoin d’un composant qui sait parler CMPv2 :
- EJBCA avec CMP Adapter
- OpenXPKI
- Proxy CMP fait maison

### 2. Webhook Cert-Manager ExternalIssuer

Créer un webhook (Go, Python…) qui :
- Reçoit les `CertificateRequest`
- Envoie la requête via CMPv2 à la CA
- Récupère le certificat signé
- Retourne le certificat à Cert-Manager

### 3. Déclaration dans Kubernetes :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cmp-cluster-issuer
spec:
  external:
    apiGroup: example.com
    kind: CMPIssuer
    name: my-cmp-issuer
