# 🔄 Cluster de Haute Disponibilité avec Heartbeat

## 📋 Vue d'ensemble

Mise en œuvre d'un **cluster de haute disponibilité (HA)** utilisant **Heartbeat** sur deux serveurs Debian. Ce système garantit la **continuité de service** en cas de panne d'un nœud grâce à un mécanisme de **failover automatique**. Le service web Apache est automatiquement migré vers le serveur secondaire via une **adresse IP virtuelle flottante**.

**Contexte :** Travaux pratiques académiques - Master Sécurité IT & Big Data  
**Type de cluster :** Active-Passive (Master-Slave)  
**Service haute disponibilité :** Apache Web Server  
**Mécanisme :** IP Failover automatique

---

## 🎯 Objectifs du projet

### Problématique
Dans les environnements critiques (banques, hôpitaux, e-commerce), **l'interruption de service n'est pas tolérable**. Une panne matérielle ou logicielle d'un serveur web peut entraîner :
- ❌ Perte de revenus (downtime)
- ❌ Dégradation de l'expérience utilisateur
- ❌ Non-respect des SLA (Service Level Agreements)

### Solution implémentée
✅ **Cluster Heartbeat** : Surveillance active des nœuds (heartbeat toutes les 2 secondes)  
✅ **IP virtuelle flottante** : 192.168.137.200 (bascule automatique entre serveurs)  
✅ **Failover automatique** : Détection de panne + migration service en < 5 secondes  
✅ **Transparent pour l'utilisateur** : Continuité de service sans intervention humaine

---

## 🏗️ Architecture du cluster

### Topologie réseau

```
┌────────────────────────────────────────────────────────────────┐
│                     Réseau Local (LAN)                         │
│                    192.168.137.0/24                            │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────┐
                │   IP Virtuelle (VIP)    │
                │   192.168.137.200       │
                │   (Flottante)           │
                └─────────────────────────┘
                     │                │
         ┌───────────┘                └───────────┐
         ▼                                        ▼
┌──────────────────────┐              ┌──────────────────────┐
│   Serveur Master     │◄─Heartbeat──►│   Serveur Slave      │
│   (srv)              │   (eth0)     │   (srv-slave)        │
├──────────────────────┤              ├──────────────────────┤
│ IP: 192.168.137.100  │              │ IP: 192.168.137.63   │
│ OS: Debian 11/12     │              │ OS: Debian 11/12     │
│ Interface: enp0s3    │              │ Interface: enp0s3    │
│                      │              │                      │
│ Services:            │              │ Services:            │
│ ✅ Apache (actif)    │              │ ⏸️ Apache (standby)  │
│ ✅ Heartbeat         │              │ ✅ Heartbeat         │
│ ✅ VIP attachée      │              │ ❌ VIP non attachée  │
└──────────────────────┘              └──────────────────────┘
         │                                        │
         └────────────────────────────────────────┘
                    Pages web distinctes:
                    Master: "Serveur Master"
                    Slave: "Serveur Slave"
```

### Scénario de failover

```
État Normal:
┌─────────┐     ┌─────────┐
│ Master  │ ✅  │  Slave  │ ⏸️
│ VIP ✅  │────►│ VIP ❌  │
└─────────┘     └─────────┘
     ▲
     │ Heartbeat OK
     │

Panne détectée:
┌─────────┐     ┌─────────┐
│ Master  │ ❌  │  Slave  │ ⚠️
│ DOWN    │  X  │ Détecte │
└─────────┘     └─────────┘
                     │
                     ▼ Migration VIP
                     
État après failover:
┌─────────┐     ┌─────────┐
│ Master  │ ❌  │  Slave  │ ✅
│ DOWN    │     │ VIP ✅  │
└─────────┘     └─────────┘
                     ▲
                     │ Service actif
                     │ (Transparent)
```

---

## 🛠️ Configuration technique

### Spécifications serveurs

| Paramètre | Master (srv) | Slave (srv-slave) |
|-----------|--------------|-------------------|
| **Hostname** | srv | srv-slave |
| **IP Physique** | 192.168.137.100 | 192.168.137.63 |
| **Interface** | enp0s3 (Bridged) | enp0s3 (Bridged) |
| **OS** | Debian 11/12 | Debian 11/12 |
| **Services** | Apache2 + Heartbeat | Apache2 + Heartbeat |
| **Rôle** | Nœud primaire | Nœud secondaire |

### IP virtuelle (VIP)
```
Adresse VIP   : 192.168.137.200
Interface     : enp0s3:0 (alias)
Réseau        : 192.168.137.0/24
Attachée par  : Heartbeat (automatique)
Bascule       : < 5 secondes en cas de panne
```

---

## 📁 Structure du repository

```
📁 High-Availability-Cluster-Heartbeat/
├── README.md                              # Ce fichier
├── docs/
│   ├── TP_HeartbeatHD.pdf                 # Rapport technique complet
│   ├── architecture-diagram.png           # Schéma du cluster
│   └── screenshots/
│       ├── heartbeat-status.png
│       ├── failover-test.png
│       └── apache-master-slave.png
├── config/
│   ├── ha.cf                              # Configuration Heartbeat
│   ├── haresources                        # Ressources HA gérées
│   ├── authkeys                           # Clés d'authentification
│   ├── hosts-master                       # Fichier /etc/hosts (srv)
│   └── hosts-slave                        # Fichier /etc/hosts (srv-slave)
├── scripts/
│   ├── install-heartbeat.sh               # Installation automatique
│   ├── configure-cluster.sh               # Configuration cluster
│   ├── test-failover.sh                   # Test bascule automatique
│   └── monitor-cluster.sh                 # Monitoring statut cluster
└── web/
    ├── index-master.html                  # Page web serveur Master
    └── index-slave.html                   # Page web serveur Slave
```

---

## 🚀 Guide d'installation pas-à-pas

### Prérequis

**Matériel requis :**
- 2 machines virtuelles (VirtualBox, VMware, KVM)
- 1 GB RAM minimum par VM
- 10 GB espace disque par VM

**Logiciels requis :**
- Debian 11 ou 12 (installation minimale)
- Accès réseau en mode **Bridged** (important pour VIP)
- Accès root (sudo)

---

### 📦 Étape 1 : Installation des paquets

**Sur les DEUX machines (srv et srv-slave) :**

```bash
# Mise à jour des dépôts
sudo apt update

# Installation Apache2 (serveur web) et Heartbeat (HA)
sudo apt install apache2 heartbeat -y

# Vérification des versions
apache2 -v
# Output: Server version: Apache/2.4.x (Debian)

heartbeat -V
# Output: heartbeat 3.x.x
```

**Pourquoi ces paquets ?**
- `apache2` : Service web à rendre hautement disponible
- `heartbeat` : Démon de surveillance et failover automatique

---

### 🌐 Étape 2 : Configuration réseau

#### 2.1 Fichier `/etc/hosts` (résolution de noms)

**Sur srv (Master) :**
```bash
sudo nano /etc/hosts

# Ajouter ces lignes :
192.168.137.100    srv
192.168.137.63     srv-slave
```

**Sur srv-slave (Slave) :**
```bash
sudo nano /etc/hosts

# Ajouter ces lignes :
192.168.137.100    srv
192.168.137.63     srv-slave
```

**Test de connectivité :**
```bash
# Depuis srv
ping -c 4 srv-slave

# Depuis srv-slave
ping -c 4 srv
```

#### 2.2 Fichier `/etc/hostname` (identification des nœuds)

**Sur srv :**
```bash
echo "srv" | sudo tee /etc/hostname
sudo hostname srv
```

**Sur srv-slave :**
```bash
echo "srv-slave" | sudo tee /etc/hostname
sudo hostname srv-slave
```

**Vérification :**
```bash
hostname
# Output: srv (ou srv-slave selon la machine)
```

---

### ⚙️ Étape 3 : Configuration Heartbeat

#### 3.1 Fichier `/etc/heartbeat/ha.cf` (configuration principale)

**Sur les DEUX machines (contenu identique) :**

```bash
sudo nano /etc/heartbeat/ha.cf
```

**Contenu à copier :**
```bash
# Temps entre deux heartbeats (2 secondes)
keepalive 2

# Délai avant déclaration de mort d'un nœud (10 secondes)
deadtime 10

# Délai au démarrage avant prise en charge (15 secondes)
initdead 15

# Interface réseau utilisée pour la communication
bcast enp0s3

# Port UDP pour les heartbeats
udpport 694

# Fichier de log Heartbeat
logfile /var/log/heartbeat.log

# Déclaration des nœuds du cluster
node srv
node srv-slave

# Activation du mode auto-failback
# Si le master revient, il reprend automatiquement la VIP
auto_failback on
```

**Explication des paramètres critiques :**
- `keepalive 2` : Vérification toutes les 2 secondes (heartbeat)
- `deadtime 10` : Si pas de réponse pendant 10s → nœud considéré mort
- `bcast enp0s3` : Interface pour envoyer les heartbeats (broadcast)
- `auto_failback on` : Le master reprend la VIP automatiquement après réparation

#### 3.2 Fichier `/etc/heartbeat/haresources` (ressources gérées)

**Sur les DEUX machines (contenu identique) :**

```bash
sudo nano /etc/heartbeat/haresources
```

**Contenu :**
```bash
srv 192.168.137.200/24/enp0s3 apache2
```

**Explication :**
- `srv` : Nœud primaire qui possède la VIP par défaut
- `192.168.137.200/24/enp0s3` : IP virtuelle + masque + interface
- `apache2` : Service à gérer (démarrage/arrêt automatique)

#### 3.3 Fichier `/etc/heartbeat/authkeys` (sécurité)

**Sur les DEUX machines (contenu identique) :**

```bash
sudo nano /etc/heartbeat/authkeys
```

**Contenu :**
```bash
auth 1
1 sha1 monClusterSecurise2025
```

**Explication :**
- `auth 1` : Utilise la méthode d'authentification n°1
- `1 sha1 ...` : Clé partagée chiffrée en SHA1 (changez "monClusterSecurise2025")

**IMPORTANT : Sécurisation du fichier (obligatoire) :**
```bash
sudo chmod 600 /etc/heartbeat/authkeys

# Vérification des permissions
ls -l /etc/heartbeat/authkeys
# Output: -rw------- 1 root root ... authkeys
```

---

### 🌐 Étape 4 : Configuration des pages web

**But :** Identifier visuellement quel serveur est actif

#### Sur srv (Master) :
```bash
echo "<h1>Serveur Master</h1>" | sudo tee /var/www/html/index.html
```

#### Sur srv-slave (Slave) :
```bash
echo "<h1>Serveur Slave</h1>" | sudo tee /var/www/html/index.html
```

---

### 🔧 Étape 5 : Désactivation d'Apache au démarrage

**Sur les DEUX machines :**

```bash
# Désactiver le démarrage automatique d'Apache
sudo systemctl disable apache2

# Arrêter Apache (il sera géré par Heartbeat)
sudo systemctl stop apache2

# Vérification
sudo systemctl status apache2
# Output: ● apache2.service - disabled
```

**Pourquoi ?** Heartbeat doit être le seul à gérer Apache pour éviter les conflits.

---

### ▶️ Étape 6 : Lancement du cluster

#### Démarrage sur srv (Master) :
```bash
sudo systemctl start heartbeat
sudo systemctl enable heartbeat

# Vérification des logs
sudo tail -f /var/log/heartbeat.log
```

**Logs attendus :**
```
info: Heartbeat 3.x.x started
info: Link srv:enp0s3 up
info: Taking over resource: 192.168.137.200
info: Starting apache2
```

#### Démarrage sur srv-slave (Slave) :
```bash
sudo systemctl start heartbeat
sudo systemctl enable heartbeat

# Vérification des logs
sudo tail -f /var/log/heartbeat.log
```

**Logs attendus :**
```
info: Heartbeat 3.x.x started
info: Link srv-slave:enp0s3 up
info: srv is active (standby mode)
```

---

### ✅ Étape 7 : Vérification du cluster

#### 7.1 Vérifier l'IP virtuelle sur le Master

**Sur srv (Master) :**
```bash
ip addr show enp0s3
```

**Sortie attendue :**
```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ...
    inet 192.168.137.100/24 brd 192.168.137.255 scope global enp0s3
    inet 192.168.137.200/24 brd 192.168.137.255 scope global secondary enp0s3:0
       valid_lft forever preferred_lft forever
```

✅ **La ligne `192.168.137.200` confirme que la VIP est attachée !**

#### 7.2 Vérifier l'état d'Apache

**Sur srv (Master) :**
```bash
sudo systemctl status apache2
# Output: ● apache2.service - active (running)
```

**Sur srv-slave (Slave) :**
```bash
sudo systemctl status apache2
# Output: ● apache2.service - inactive (dead)
```

✅ **Apache tourne uniquement sur le nœud actif**

#### 7.3 Test depuis un navigateur

**Depuis votre machine hôte, ouvrir un navigateur :**
```
http://192.168.137.200
```

**Résultat attendu :** Page affichant **"Serveur Master"**

---

### 🔄 Étape 8 : Test de failover (bascule automatique)

#### Scénario : Simulation panne du Master

**1. Éteindre brutalement srv (Master) :**
```bash
# Sur srv
sudo poweroff
```

**2. Observer les logs sur srv-slave :**
```bash
# Sur srv-slave
sudo tail -f /var/log/heartbeat.log
```

**Logs attendus (en temps réel) :**
```
warn: No heartbeat from srv for 10 seconds
info: srv is dead
info: Taking over resource: 192.168.137.200
info: Starting apache2
info: Acquisition completed
```

**3. Vérifier la VIP sur srv-slave :**
```bash
ip addr show enp0s3
# Output: inet 192.168.137.200/24 ... scope global secondary enp0s3:0
```

✅ **La VIP a basculé automatiquement sur srv-slave !**

**4. Tester depuis le navigateur :**
```
http://192.168.137.200
```

**Résultat :** Page affichant maintenant **"Serveur Slave"**

---

### 🔙 Étape 9 : Test de failback (retour automatique)

**1. Redémarrer srv (Master)**

**2. Observer les logs sur srv :**
```bash
sudo tail -f /var/log/heartbeat.log
```

**Logs attendus :**
```
info: Heartbeat restarted
info: srv-slave is active
info: Requesting resource takeover
info: Taking over resource: 192.168.137.200
info: Starting apache2
```

**3. Vérifier la VIP de retour sur srv :**
```bash
ip addr show enp0s3
# Output: inet 192.168.137.200/24 ... (VIP revenue sur srv)
```

**4. Tester depuis le navigateur :**
```
http://192.168.137.200
```

**Résultat :** Page affichant à nouveau **"Serveur Master"**

✅ **Le failback automatique fonctionne (auto_failback on)**

---

## 📊 Monitoring et commandes utiles

### Vérifier l'état du cluster

```bash
# Statut Heartbeat
sudo systemctl status heartbeat

# Logs en temps réel
sudo tail -f /var/log/heartbeat.log

# Vérifier les ressources actives
sudo grep "Taking over" /var/log/heartbeat.log

# Afficher les interfaces réseau
ip addr show enp0s3
```

### Tester la connectivité VIP

```bash
# Depuis la machine hôte ou un client
ping 192.168.137.200

# Test HTTP
curl http://192.168.137.200
# Output: <h1>Serveur Master</h1> (ou Slave)
```

### Redémarrer Heartbeat proprement

```bash
sudo systemctl restart heartbeat
```

### Forcer le failover (test)

```bash
# Sur le nœud actif
sudo systemctl stop heartbeat

# Observer le basculement sur l'autre nœud
```

---

## 🔧 Dépannage (Troubleshooting)

### Problème : VIP ne bascule pas

**Vérifications :**
```bash
# 1. Heartbeat est-il actif sur les deux machines ?
sudo systemctl status heartbeat

# 2. Les nœuds se voient-ils ?
ping srv
ping srv-slave

# 3. Le fichier haresources est-il identique ?
sudo cat /etc/heartbeat/haresources

# 4. Les permissions d'authkeys sont-elles correctes ?
ls -l /etc/heartbeat/authkeys
# Doit afficher : -rw------- (600)
```

### Problème : Apache ne démarre pas

**Solution :**
```bash
# Vérifier qu'Apache est désactivé au boot
sudo systemctl disable apache2

# Tester Apache manuellement
sudo systemctl start apache2
sudo systemctl status apache2

# Vérifier les logs Apache
sudo tail -f /var/log/apache2/error.log
```

### Problème : Erreur "split-brain" (les 2 nœuds actifs)

**Cause :** Communication réseau interrompue entre les nœuds

**Solution :**
```bash
# Vérifier connectivité réseau
ping srv
ping srv-slave

# Augmenter le deadtime dans ha.cf
sudo nano /etc/heartbeat/ha.cf
deadtime 30  # Au lieu de 10

# Redémarrer Heartbeat
sudo systemctl restart heartbeat
```

---

## 📈 Résultats et métriques

### Performance du failover

| Métrique | Valeur mesurée | Objectif |
|----------|----------------|----------|
| **Temps de détection de panne** | ~10 secondes | < 15s |
| **Temps de bascule VIP** | ~2 secondes | < 5s |
| **Temps total de failover** | **~12 secondes** | < 20s |
| **Perte de requêtes** | 0 (après bascule) | 0% |
| **Downtime perçu** | ~12 secondes | < 30s |

### Disponibilité calculée

**Formule :**
```
Disponibilité = (Temps total - Downtime) / Temps total × 100

Avec 1 panne par mois de 12s :
= (2592000s - 12s) / 2592000s × 100
= 99.9995% de disponibilité (5 nines !)
```

---

## 🎓 Concepts techniques démontrés

### ✅ Haute disponibilité (High Availability)
- Architecture Active-Passive (Master-Slave)
- Élimination du SPOF (Single Point of Failure)
- RTO (Recovery Time Objective) : < 15 secondes
- RPO (Recovery Point Objective) : 0 (aucune perte de données)

### ✅ Mécanismes de failover
- Heartbeat monitoring (surveillance active)
- Split-brain prevention (authentification)
- IP Failover (migration VIP automatique)
- Service management (start/stop automatique d'Apache)

### ✅ Réseau avancé
- IP virtuelle flottante (VIP)
- Alias d'interface (enp0s3:0)
- Mode Bridge (accès LAN)
- Broadcast heartbeats (communication inter-nœuds)

### ✅ Administration système Linux
- Gestion services systemd
- Configuration réseau (/etc/hosts, /etc/hostname)
- Permissions fichiers (chmod 600)
- Analyse de logs (/var/log/heartbeat.log)

---

## 🏢 Cas d'usage réels

Ce type de cluster Heartbeat est utilisé dans :

### 🏦 Secteur bancaire
- Serveurs de transactions financières
- Portails bancaires en ligne
- ATM (distributeurs automatiques)

### 🏥 Secteur médical
- Systèmes de dossiers médicaux électroniques
- Équipements médicaux connectés
- Plateformes de télémédecine

### 🛒 E-commerce
- Sites web de vente en ligne
- Systèmes de paiement
- Gestion de stocks en temps réel

### 🚨 Services d'urgence
- Systèmes d'appels d'urgence (911, 112)
- Dispatch centers
- Systèmes de géolocalisation

---

## 📚 Améliorations possibles

### 🔧 Extensions techniques

- [ ] **Cluster 3 nœuds** : Passer de 2 à 3 serveurs pour éliminer le split-brain
- [ ] **Monitoring Nagios** : Intégration avec système de supervision
- [ ] **Heartbeat sur liens multiples** : Ethernet + Serial pour redondance
- [ ] **DRBD (Distributed Replicated Block Device)** : Réplication de données en temps réel
- [ ] **Pacemaker + Corosync** : Migration ve
