-------
# PKI – Projet TP4 M2SSI

**Auteur :** DAMBE Lamboni
**Master :** 2SSI
**Module :** Cryptographie Appliquée
**Date :** Lundi 09 Décembre 2024
**Email :** [dlamboni31@gmail.com](mailto:dlamboni31@gmail.com)

---

## 1. Objectif

Ce projet TP4 a pour objectif la **mise en place d’une infrastructure PKI complète**, incluant :

* Diffusion et vérification de certificats racines
* Création et gestion d’une **autorité intermédiaire**
* Génération et signature de certificats utilisateurs et serveurs
* Gestion des **listes de révocation (CRL)**
* Création d’enveloppes **PKCS#12** pour importation dans les clients (ex. Thunderbird)

---

## 2. Exercice 1 : Diffusion de l’autorité

### Rôle et importance du fingerprint

* **Rôle :** Garantir l’intégrité et l’authenticité d’un certificat via un condensat cryptographique unique.
* **Importance :** Protection contre les attaques MITM et vérification de l’origine du certificat.

### Importation et vérification du certificat racine

```bash
# Vérification du fingerprint
openssl x509 -in m2ssi_24.crt -noout -fingerprint -sha256

# Ajout dans Ubuntu
sudo cp m2ssi_24.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Vérification de la validité
openssl x509 -in /etc/ssl/certs/m2ssi.pem -text -noout
openssl verify -CAfile /etc/ssl/certs/m2ssi_24.pem /etc/ssl/certs/m2ssi_24.pem
```

### Importation dans le navigateur

* Firefox → Paramètres → Vie privée → Certificats → Import

### Contenu d’un certificat

* **Informations claires :** version, numéro de série, émetteur, sujet, période de validité, clé publique, extensions
* **Données cryptographiques :** signature numérique
* **Empreintes SHA-1/SHA-256 :** vérification rapide de l’intégrité

### Usages possibles et champs

* Signatures CRL, OCSP, S/MIME, Time Stamp
* **Issuer :** Autorité émettrice (CA)
* **Subject :** Entité à laquelle le certificat est attribué
* **Certificat racine :** Auto-signé, sommet de la chaîne de confiance

---

## 3. Exercice 2 : Autorité intermédiaire

### Création d’une bi-clé RSA

```bash
openssl genrsa -out dambelam_tp4.pem 2048
openssl rsa -in dambelam_tp4.pem -pubout -out dambelam_tp4_pub.pem
```

### Création de la CSR

```bash
openssl req -new -key dambelam_tp4.pem -out CSR_dambelam_tp4.req
openssl req -in CSR_dambelam_tp4.req -text -noout
```

### Envoi à l’autorité racine

* Certificat signé reçu et vérifié :

```bash
openssl verify -CAfile m2ssi_24.crt dambelam_tp4.crt
```

---

## 4. Exercice 3 : Mise en place de l’autorité intermédiaire

### Arborescence

```text
ca/intermediate/
├── certs/
├── newcerts/
├── crl/
├── csr/
├── private/
├── index.txt
├── serial
└── crlnumber
```

### Déplacement des fichiers

```bash
cp ../dambelam_tp4.pem intermediate/private/
cp ../dambelam_tp4.crt intermediate/certs/
```

### Configuration `openssl.cnf`

* Mise à jour des chemins vers les répertoires et fichiers spécifiques à l’autorité intermédiaire

---

## 5. Exercice 4 : Certificats utilisateurs/serveurs

### Configuration des extensions dans `openssl.cnf`

* S/MIME, TLS serveur/client, VPN IPsec, OpenVPN serveur/client
* `keyUsage` et `extendedKeyUsage` adaptés à chaque usage

### Génération des clés privées et CSR

```bash
mkdir -p users/public users/csr users/private

openssl genrsa -out users/private/smime_client.key 2048
openssl genrsa -out users/private/server_tls.key 2048
openssl genrsa -out users/private/ipsec_vpn.key 2048
openssl genrsa -out users/private/openvpn_client.key 2048

openssl req -new -key private/smime_client.key -out csr/smime_client.csr -subj "/C=FR/ST=Seine-Maritime/O=M2_SSI/CN=smime-client.m2ssi_tp4"
# idem pour les autres types
```

### Signature avec l’autorité intermédiaire

```bash
openssl ca -config /etc/ssl/openssl.cnf -extensions smime_cert -in csr/smime_client.csr -out public/smime_client.crt
# idem pour les autres types
```

---

## 6. Exercice 5 : Enveloppe PKCS#12

```bash
openssl pkcs12 -export \
-inkey users/private/smime_client.key \
-in users/public/smime_client.crt \
-certfile ca/intermediate/certs/dambelam_tp4.crt \
-out users/public/smime_client.p12 \
-name "Client S/MIME M2SSI_TP4"

# Vérification
openssl pkcs12 -info -in users/public/smime_client.p12
```

* Importation dans Thunderbird pour signature de messages

---

## 7. Exercice 6 : Listes de révocation CRL

### Création et signature d’un certificat utilisateur

```bash
openssl genrsa -out users/private/revoked_user.key 2048
openssl req -new -key users/private/revoked_user.key -out users/csr/revoked_user.csr -subj "/C=FR/ST=Seine-Maritime/O=M2_SSI/CN=revoked-user.m2ssi_tp4"
openssl ca -config /etc/ssl/openssl.cnf -extensions client_tls_cert -in users/csr/revoked_user.csr -out users/public/revoked_user.crt
```

### Révocation et génération CRL

```bash
openssl ca -config /etc/ssl/openssl.cnf -revoke users/public/revoked_user.crt
openssl ca -config /etc/ssl/openssl.cnf -gencrl -out ca/intermediate/crl/intermediate.crl
```

* Certificat révoqué identifié dans la CRL et rejeté par les applications vérifiant son état

---

## Références

* [Linux Tricks – Ajouter des certificats auto-signés](https://www.linuxtricks.fr/wiki/certificats-ajouter-des-certificats-autosignes-sur-linux)
* [Dell – Validation et conversion de certificats SSL](https://www.dell.com/support/kbdoc/fr-tn/000211907/proc%C3%A9dure-generale-comment-valider-et-convertir-certificat-ssl)
* [Certificat.fr – Certificat intermédiaire](https://www.certificat.fr/fr/support/faq/quest-ce-que-est-certificat-intermediaire)
* [Stormshield – Authentification VPN IPsec par certificat](https://documentation.stormshield.com/SNS/v4/fr/Content/HowTo_-_IPSec_VPN_-_Authentication_by_certificate/Setup-Main-Site-10-Creating-PKI.htm)

---

