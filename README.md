# 🔐 Module 06 : Autorité de Certification (AD CS WS2026) & Service DHCP

## 🎯 Objectifs
- Déployer une **PKI d'Entreprise** avec le rôle **Active Directory Certificate Services (AD CS)** sous **Windows Server 2025** (`oclock-WS2026-CA`).
- Sécuriser l'accès à l'intranet web via un certificat SSL/TLS valide (port 443) sans avertissement de sécurité sur les clients.
- Expérimenter le mécanisme de **révocation des certificats** (liste CRL et cache `certutil`).
- Déployer et autoriser le rôle **Serveur DHCP** sur Windows Server à la place du routeur OPNsense.
- Analyser la chaîne de négociation des baux **DORA** et valider le passage d'une adresse **APIPA** (`169.254.x.x`) à un bail d'adressage IP local (`10.0.200.x`).

---

## 📋 Concepts Clés

### 1. PKI & Chaîne de Confiance (AD CS WS2026)
- **Chaîne de Confiance** : Un client ne fait pas confiance à un certificat isolément, mais à l'Autorité de Certification (AC) qui l'a émis.
- **AC d'Entreprise vs AC Autonome** : L'AC d'entreprise s'intègre à l'Active Directory, publie automatiquement son certificat racine dans le magasin des clients du domaine et autorise l'inscription automatique (*Auto-enrollment*).
- **Répertoire de configuration AD** : La racine de l'AC est répliquée dans la partition de configuration AD (`CN=Public Key Services`), dispensant de la création d'une GPO manuelle d'import.
- **Révocation (CRL)** : Publication de la liste des certificats révoqués accessible via les points de distribution CDP.

### 2. Service DHCP & Adressage Réseau
- **Cycle DORA** : Négociation en 4 étapes (*Discover*, *Offer*, *Request*, *Acknowledge*).
- **Options DHCP obligatoires** :
  - `003` : Passerelle par défaut (*Gateway*).
  - `006` : Serveurs DNS du domaine.
  - `015` : Suffixe DNS du domaine (`oclock.lan`).
  - `066/067` : Paramètres de boot réseau pour PXE/WDS.
- **Autorisation dans AD** : Un serveur DHCP Windows membre d'un domaine Active Directory doit être explicitement autorisé par un administrateur pour pouvoir distribuer des baux.

---

## 🛠️ Étapes de Configuration (Lab & Manipulations)

### 1️⃣ Déploiement d'AD CS sous Windows Server 2025
1. Installation du rôle **Services de certificats Active Directory** (`certsrv.msc`).
2. Configuration de l'autorité racine d'entreprise :
   - Type : **AC d'entreprise** / **AC racine**.
   - Clé privée : RSA 2048 / SHA256.
   - Nom de l'AC : `oclock-WS2026-CA` (Durée de validité : 5 ans).
3. Emissions et demandes de certificats :
   - Génération d'un certificat de domaine depuis la console IIS pour `intranet.oclock.lan`.
   - Création de la liaison HTTPS sur le port `443` avec le certificat émis.

### 2️⃣ Déploiement et basculement du Service DHCP
1. Désactivation du service DHCP sur l'interface LAN de l'équipement OPNsense.
2. Installation du rôle **Serveur DHCP** sur Windows Server (`dhcpmgmt.msc`).
3. Exécution de l'assistant d'**Autorisation Active Directory** (`OCLOCK\Administrateur`).
4. Création d'une nouvelle étendue d'adressage IPv4 :
   - Plage : `10.0.200.10` à `10.0.200.50` (Masque `255.255.0.0`).
   - Options d'étendue : Passerelle (`10.0.0.1`), DNS (`10.0.0.10`), Suffixe (`oclock.lan`).

---

## 🧪 Validation & Troubleshooting

1. **Test HTTPS Client** :
   - Navigation sur `https://intranet.oclock.lan` depuis la VM cliente.
   - Validation du cadenas vert fermé sans avertissement de sécurité (Racine `oclock-WS2026-CA` reconnue).
2. **Test de Révocation** :
   - Révocation du certificat intranet sous `certsrv.msc` (Motif : *Suspension*).
   - Publication de la nouvelle liste CRL et inspection du cache avec `certutil -urlcache CRL delete` & `certutil -urlfetch -verify`.
3. **Démonstration APIPA -> Bail DHCP** :
   - Exécution de `ipconfig /release` sur le client alors que l'étendue est inactive -> Obtention d'une adresse APIPA (`169.254.x.x`).
   - Activation de l'étendue DHCP + `ipconfig /renew` -> Obtention d'un bail valide (`10.0.200.10`).

---

## ⚠️ Sécurité & Bonnes Pratiques
- **Architecture PKI à 2 Niveaux** : En environnement de production réels, l'AC Racine (*Root CA*) doit être installée sur un serveur autonome hors ligne (*offline*) déconnecté du réseau, signant une AC subordonnée d'entreprise.
- **DHCP Snooping** : L'autorisation AD protège contre la mise en place involontaire d'un serveur Windows DHCP non déclaré, mais ne bloque pas un équipement ou routeur rogue. La protection contre le DHCP pirate s'exécute au niveau des commutateurs (Switchिंग / Switching).
