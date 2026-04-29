# 🛡️ Azure Cybersecurity Dashboard — Mini SOC Cloud

> **Technologies** : Microsoft Azure · Linux · Fail2ban · Azure Monitor · Microsoft Sentinel · Power BI  
> **Objectif** : Construire un mini SOC (Security Operations Center) dans le cloud Azure

---

## 📋 Table des matières

1. [Prérequis](#-prérequis)
2. [Architecture du projet](#-architecture-du-projet)
3. [Étape 1 — Créer l'infrastructure Azure](#-étape-1--créer-linfrastructure-azure)
4. [Étape 2 — Connexion SSH & configuration de la VM](#-étape-2--connexion-ssh--configuration-de-la-vm)
5. [Étape 3 — Simuler des activités et générer des logs](#-étape-3--simuler-des-activités-et-générer-des-logs)
6. [Étape 4 — Collecter les logs avec Azure Monitor](#-étape-4--collecter-les-logs-avec-azure-monitor)
7. [Étape 5 — Activer Microsoft Sentinel (SIEM)](#-étape-5--activer-microsoft-sentinel-siem)
8. [Étape 6 — Créer des règles de détection](#-étape-6--créer-des-règles-de-détection)
9. [Étape 7 — Construire le Dashboard](#-étape-7--construire-le-dashboard)
10. [Étape 8 — (Avancé) Automatiser la réponse aux incidents](#-étape-8--avancé-automatiser-la-réponse-aux-incidents)
11. [Résultat final attendu](#-résultat-final-attendu)
12. [Dépannage fréquent](#-dépannage-fréquent)

---

## ✅ Prérequis

Avant de commencer, assure-toi d'avoir :

- [x] Un compte **Microsoft Azure** actif ([créer un compte gratuit](https://azure.microsoft.com/free/))
  - Le compte gratuit offre **200$ de crédits** valables 30 jours
- [x] Un terminal avec un client SSH installé :
  - **Windows** : PowerShell, Windows Terminal, ou [PuTTY](https://putty.org/)
  - **macOS / Linux** : Terminal natif
- [x] Un navigateur web moderne (Chrome, Edge recommandé pour Azure Portal)
- [x] (Optionnel) **Power BI Desktop** installé sur ta machine → [Télécharger](https://powerbi.microsoft.com/fr-fr/downloads/)

---

## 🏗️ Architecture du projet

```
┌─────────────────────────────────────────────────────────┐
│                     AZURE CLOUD                         │
│                                                         │
│  ┌─────────────┐     logs      ┌──────────────────────┐ │
│  │  VM Ubuntu  │ ──────────── ▶│  Log Analytics       │ │
│  │  (Linux)    │               │  Workspace           │ │
│  │             │               └──────────┬───────────┘ │
│  │ fail2ban    │                          │             │
│  │ auth.log    │               ┌──────────▼───────────┐ │
│  │ syslog      │               │  Microsoft Sentinel  │ │
│  └─────────────┘               │  (SIEM)              │ │
│                                │  - Détection         │ │
│                                │  - Alertes           │ │
│                                └──────────┬───────────┘ │
│                                           │             │
│                                ┌──────────▼───────────┐ │
│                                │  Dashboard           │ │
│                                │  Azure / Power BI    │ │
│                                └──────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## 🔷 Étape 1 — Créer l'infrastructure Azure

### 1.1 — Créer un groupe de ressources

> Un groupe de ressources est un conteneur logique pour organiser tes ressources Azure.

1. Connecte-toi sur [portal.azure.com](https://portal.azure.com)
2. Dans la barre de recherche en haut, tape **"Groupes de ressources"**
3. Clique sur **➕ Créer**
4. Remplis les champs :
   - **Abonnement** : Azure (ton abonnement gratuit)
   - **Nom du groupe** : `rg-cybersecurity-lab`
   - **Région** : `France Central` (ou la plus proche de toi)
5. Clique sur **Vérifier + Créer**, puis **Créer**

---

### 1.2 — Créer une Machine Virtuelle (VM) Ubuntu

1. Dans la barre de recherche, tape **"Machines virtuelles"**
2. Clique sur **➕ Créer** → **Machine virtuelle Azure**
3. Remplis l'onglet **"Informations de base"** :

   | Champ | Valeur |
   |-------|--------|
   | Groupe de ressources | `rg-cybersecurity-lab` |
   | Nom de la machine virtuelle | `vm-soc-lab` |
   | Région | `France Central` |
   | Image | `Ubuntu Server 24.04 LTS` |
   | Taille | `Standard_B1s` (1 vCPU, 1 Go RAM — niveau gratuit) |
   | Type d'authentification | `Clé publique SSH` |
   | Nom d'utilisateur | `azureuser` |
   | Nom de la paire de clés | `soc-lab-key` |

4. Dans l'onglet **"Disques"** : laisser les valeurs par défaut

5. Dans l'onglet **"Réseau"** :
   - Laisser Azure créer un nouveau réseau virtuel
   - **Ports entrants publics** : Autoriser les ports sélectionnés
   - **Ports** : cocher `SSH (22)`

6. Clique sur **Vérifier + Créer**, puis **Créer**

7. Quand la fenêtre **"Générer une nouvelle paire de clés"** apparaît :
   - Clique sur **Télécharger la clé privée et créer la ressource**
   - ⚠️ **Sauvegarde le fichier `.pem` téléchargé** — tu en auras besoin pour te connecter

8. Attends la fin du déploiement (environ 2 minutes)

---

## 🔷 Étape 2 — Connexion SSH & configuration de la VM

### 2.1 — Récupérer l'adresse IP de la VM

1. Va dans **Machines virtuelles** → clique sur `vm-soc-lab`
2. Dans l'onglet **"Vue d'ensemble"**, note l'**Adresse IP publique**
   - Exemple : `20.93.145.12` (74.234.186.100)

---

### 2.2 — Se connecter en SSH

**Sur macOS / Linux :**

```bash
# Donne les bonnes permissions à ta clé
chmod 400 ~/Téléchargements/soc-lab-key.pem

# Connecte-toi à la VM
ssh -i ~/Téléchargements/soc-lab-key.pem azureuser@<TON_IP_PUBLIQUE>
```

**Sur Windows (PowerShell) :**

```powershell
# Connexion SSH
ssh -i C:\Users\TonNom\Downloads\soc-lab-key.pem azureuser@<TON_IP_PUBLIQUE>
```

> ✅ Si tu vois `azureuser@vm-soc-lab:~$` → tu es connecté !

---

### 2.3 — Mettre à jour le système

```bash
# Mettre à jour les paquets
sudo apt update && sudo apt upgrade -y

# Vérifier que SSH est actif (il l'est par défaut)
sudo systemctl status ssh
```

---

### 2.4 — Installer Fail2ban 

> Fail2ban surveille les tentatives de connexion échouées et bannit les IPs suspectes. Il génère des logs que Sentinel va analyser.

```bash
# Installer fail2ban
sudo apt install fail2ban -y

# Activer et démarrer le service
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Vérifier qu'il tourne
sudo systemctl status fail2ban
```

**Configurer Fail2ban pour SSH :**

```bash
# Créer un fichier de configuration local
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# Editer la configuration
sudo nano /etc/fail2ban/jail.local
```

Dans le fichier, trouve la section `[sshd]` et assure-toi qu'elle contient :

```ini
[sshd]
enabled  = true
port     = ssh
maxretry = 3
bantime  = 3600
findtime = 600
```

Enregistre avec `Ctrl+O`, puis quitte avec `Ctrl+X`.

```bash
# Redémarrer fail2ban
sudo systemctl restart fail2ban
```

---

## 🔷 Étape 3 — Simuler des activités et générer des logs

> On va simuler des comportements suspects pour avoir des données à analyser dans Sentinel.

### 3.1 — Simuler des tentatives de connexion échouées

Depuis ta machine locale (pas dans la VM), ouvre un **second terminal** et exécute :

```bash
# Tentatives de connexion avec un mauvais mot de passe (depuis ta machine locale)
# Remplace <TON_IP_PUBLIQUE> par l'IP de ta VM
for i in {1..5}; do
  ssh -o ConnectTimeout=5 fakeuser@<TON_IP_PUBLIQUE> 2>/dev/null
  echo "Tentative $i effectuée"
done
```

### 3.2 — Générer des logs d'activité dans la VM

De retour dans ta VM (premier terminal) :

```bash
# Simuler des accès fichiers
ls -la /etc/
cat /etc/passwd
cat /var/log/auth.log | tail -50

# Créer des fichiers de test
mkdir -p ~/lab-test
touch ~/lab-test/sensitive_data.txt
echo "Simulation accès fichier sensible" >> ~/lab-test/sensitive_data.txt

# Vérifier les logs d'authentification
sudo tail -f /var/log/auth.log
```

### 3.3 — Vérifier que Fail2ban a détecté les tentatives

```bash
# Voir les IPs bannies
sudo fail2ban-client status sshd

# Voir les logs de fail2ban
sudo tail -20 /var/log/fail2ban.log
```

> 📝 Tu devrais voir les tentatives de connexion enregistrées dans les logs.

---

## 🔷 Étape 4 — Collecter les logs avec Azure Monitor

### 4.1 — Créer un Log Analytics Workspace

1. Dans le portail Azure, recherche **"Log Analytics Workspace"**
2. Clique sur **➕ Créer**
3. Remplis :
   - **Groupe de ressources** : `rg-cybersecurity-lab`
   - **Nom** : `law-soc-lab`
   - **Région** : `France Central`
4. Clique sur **Vérifier + Créer** → **Créer**
5. Attends le déploiement (1-2 minutes)

---

### 4.2 — Connecter la VM au Workspace

1. Va dans ta VM : **Machines virtuelles** → `vm-soc-lab`
2. Dans le menu de gauche, descends jusqu'à **"Surveillance"**
3. Clique sur **"Insights"** → **Activer**
4. Sélectionne **"Log Analytics Workspace"** : choisis `law-soc-lab`
5. Clique sur **"Configurer"**

> ⏳ L'agent Azure Monitor va s'installer automatiquement sur la VM (5-10 minutes)

---

### 4.3 — Configurer la collecte des logs Linux

1. Va dans **"Log Analytics Workspaces"** → `law-soc-lab`
2. Dans le menu de gauche → **"Paramètres"** → **"Agents"**
3. Clique sur **"Linux syslog"**
4. Clique sur **"Ajouter une facilité"** et ajoute :
   - `auth` (INFO et supérieur)
   - `syslog` (WARNING et supérieur)
   - `kern` (ERROR et supérieur)
5. Clique sur **"Appliquer"**

---

### 4.4 — Vérifier que les logs arrivent

1. Dans `law-soc-lab` → **"Journaux"** (dans le menu de gauche)
2. Dans l'éditeur de requêtes, entre :

```kusto
Syslog
| take 10
```

3. Clique sur **Exécuter**

> ✅ Si tu vois des lignes de logs → la collecte fonctionne !  
> ⏳ Si aucun résultat → attends 10-15 minutes, les logs peuvent mettre du temps à apparaître.

---

## 🔷 Étape 5 — Activer Microsoft Sentinel (SIEM)

### 5.1 — Activer Sentinel

1. Recherche **"Microsoft Sentinel"** dans le portail Azure
2. Clique sur **➕ Créer**
3. Sélectionne ton workspace : `law-soc-lab`
4. Clique sur **"Ajouter Microsoft Sentinel"**
5. Attends quelques minutes pour l'activation

---

### 5.2 — Connecter les sources de données

1. Dans Microsoft Sentinel → menu de gauche → **"Connecteurs de données"**

**Connecter les événements de sécurité Linux :**

2. Dans la barre de recherche, tape **"Syslog"**
3. Clique sur le connecteur **"Syslog"**
4. Clique sur **"Ouvrir la page du connecteur"**
5. Clique sur **"Installer l'agent sur une machine virtuelle Azure Linux"**
6. Sélectionne ta VM `vm-soc-lab` → clique **"Connecter"**

**Connecter les événements Microsoft Defender for Cloud :**

7. Retourne dans les connecteurs, cherche **"Microsoft Defender for Cloud"**
8. Clique sur le connecteur → **"Ouvrir la page du connecteur"**
9. Dans la section **"Configuration"**, active ton abonnement Azure

> ⏳ La synchronisation peut prendre 15-30 minutes.

---

### 5.3 — Vérifier la réception des données

1. Dans Sentinel → **"Journaux"**
2. Entre la requête :

```kusto
SecurityEvent
| summarize count() by EventID
| order by count_ desc
| take 10
```

3. Exécute la requête pour voir les types d'événements collectés.

---

## 🔷 Étape 6 — Créer des règles de détection

> On va créer des règles d'alerte dans Sentinel pour détecter des comportements suspects.

### 6.1 — Règle 1 : Tentatives de connexion SSH multiples échouées

1. Dans Sentinel → menu de gauche → **"Configuration"** → **"Règles analytiques"**
2. Clique sur **"➕ Créer"** → **"Règle de requête planifiée"**

**Onglet "Général" :**

| Champ | Valeur |
|-------|--------|
| Nom | `Brute Force SSH - Multiples échecs de connexion` |
| Description | `Détecte plus de 5 tentatives de connexion SSH échouées en 10 minutes depuis la même IP` |
| Gravité | `Moyenne` |
| MITRE ATT&CK | `Credential Access > Brute Force` |
| Statut | `Activé` |

**Onglet "Définir la logique de règle" :**

Colle cette requête KQL :

```kusto
Syslog
| where Facility == "auth"
| where SyslogMessage contains "Failed password" or SyslogMessage contains "Invalid user"
| extend IPAddress = extract(@"from (\d+\.\d+\.\d+\.\d+)", 1, SyslogMessage)
| where isnotempty(IPAddress)
| summarize FailedAttempts = count(), FirstAttempt = min(TimeGenerated), LastAttempt = max(TimeGenerated)
    by IPAddress, bin(TimeGenerated, 10m)
| where FailedAttempts >= 5
| project TimeGenerated, IPAddress, FailedAttempts, FirstAttempt, LastAttempt
```

**Planification de la requête :**
- Exécuter la requête toutes les : `5 minutes`
- Rechercher les données des dernières : `10 minutes`

**Seuil d'alerte :**
- Générer une alerte quand le nombre de résultats est supérieur à : `0`

3. Clique sur **"Vérifier + Créer"** → **"Créer"**

---

### 6.2 — Règle 2 : Connexion réussie après plusieurs échecs (Password Spray)

1. Crée une nouvelle règle analytique avec ces paramètres :

**Nom** : `Password Spray - Connexion suspecte après échecs`

**Requête KQL :**

```kusto
let FailedLogins = Syslog
| where Facility == "auth"
| where SyslogMessage contains "Failed password"
| extend IPAddress = extract(@"from (\d+\.\d+\.\d+\.\d+)", 1, SyslogMessage)
| summarize FailedCount = count() by IPAddress, bin(TimeGenerated, 30m);
let SuccessLogins = Syslog
| where Facility == "auth"
| where SyslogMessage contains "Accepted password" or SyslogMessage contains "Accepted publickey"
| extend IPAddress = extract(@"from (\d+\.\d+\.\d+\.\d+)", 1, SyslogMessage);
SuccessLogins
| join kind=inner (FailedLogins | where FailedCount >= 3) on IPAddress
| project TimeGenerated, IPAddress, FailedCount
| order by TimeGenerated desc
```

**Gravité** : `Haute`  
**Planification** : toutes les `10 minutes`, données des `30 dernières minutes`

---

### 6.3 — Règle 3 : Activité en dehors des heures normales

1. Crée une nouvelle règle :

**Nom** : `Connexion SSH hors heures ouvrables`

**Requête KQL :**

```kusto
Syslog
| where Facility == "auth"
| where SyslogMessage contains "Accepted"
| where hourofday(TimeGenerated) < 7 or hourofday(TimeGenerated) > 20
| extend IPAddress = extract(@"from (\d+\.\d+\.\d+\.\d+)", 1, SyslogMessage)
| extend User = extract(@"for (\w+) from", 1, SyslogMessage)
| project TimeGenerated, IPAddress, User, SyslogMessage
```

**Gravité** : `Faible`

---

## 🔷 Étape 7 — Construire le Dashboard

### Option A — Dashboard Azure (intégré)

1. Dans Sentinel → **"Gestion des menaces"** → **"Classeurs"**
2. Clique sur **"Ajouter un classeur"**
3. Clique sur **"Modifier"** (icône crayon)
4. Supprime le contenu par défaut
5. Clique sur **"Ajouter"** → **"Ajouter une requête"**

**Widget 1 — Tentatives de connexion par heure :**

```kusto
Syslog
| where Facility == "auth"
| where SyslogMessage contains "Failed password" or SyslogMessage contains "Accepted"
| extend Status = iff(SyslogMessage contains "Failed", "Echec", "Succès")
| summarize Count = count() by bin(TimeGenerated, 1h), Status
| order by TimeGenerated asc
```
→ Visualisation : **Graphique en courbes**

**Widget 2 — Top 10 des IPs suspectes :**

```kusto
Syslog
| where Facility == "auth"
| where SyslogMessage contains "Failed password"
| extend IPAddress = extract(@"from (\d+\.\d+\.\d+\.\d+)", 1, SyslogMessage)
| where isnotempty(IPAddress)
| summarize Tentatives = count() by IPAddress
| order by Tentatives desc
| take 10
```
→ Visualisation : **Graphique à barres**

**Widget 3 — Alertes Sentinel déclenchées :**

```kusto
SecurityAlert
| summarize count() by AlertName, bin(TimeGenerated, 1d)
| order by TimeGenerated desc
```
→ Visualisation : **Grille**

**Widget 4 — Ratio Succès / Échec :**

```kusto
Syslog
| where Facility == "auth"
| where SyslogMessage contains "Failed password" or SyslogMessage contains "Accepted password"
| extend Status = iff(SyslogMessage contains "Failed", "Echec", "Succès")
| summarize Count = count() by Status
```
→ Visualisation : **Camembert (Pie chart)**

6. Clique sur **"Enregistrer"** → Donne le nom `Dashboard-SOC-Lab`

---

### Option B — Dashboard Power BI (recommandé pour le CV)

1. Dans `law-soc-lab` → **"Vue d'ensemble"** → note l'**ID de l'espace de travail**
2. Ouvre **Power BI Desktop**
3. **Obtenir les données** → **Azure** → **Azure Log Analytics**
4. Entre ton **ID d'espace de travail**
5. Connecte-toi avec ton compte Azure
6. Crée les visuels suivants :
   - 📊 Graphique en courbes : tentatives par heure
   - 🥧 Camembert : échecs vs succès
   - 📋 Tableau : top IPs suspectes
   - 🔢 Carte KPI : nombre total d'alertes
7. **Publie** le rapport sur Power BI Service

---

## 🔷 Étape 8 — (Avancé) Automatiser la réponse aux incidents

### 8.1 — Créer un Playbook (Logic App) pour bannir une IP

1. Dans Sentinel → **"Automatisation"** → **"Règles d'automatisation"**
2. Clique sur **"➕ Créer"** → **"Playbook"**
3. Sélectionne le modèle **"Block IP - Azure Firewall"** si disponible
4. Ou crée une Logic App personnalisée :
   - **Déclencheur** : Quand une alerte Sentinel est créée
   - **Action 1** : Envoyer un email de notification (via Office 365)
   - **Action 2** : Ajouter l'IP à une règle de blocage NSG (Network Security Group)

### 8.2 — Envoyer une alerte email automatique

Dans la Logic App :

1. Déclencheur : **"Microsoft Sentinel - Quand une réponse à une alerte Microsoft Sentinel est déclenchée"**
2. Ajouter une action : **"Envoyer un e-mail (V2)"** — Outlook ou Gmail
3. Corps de l'email :

```
🚨 ALERTE SÉCURITÉ

Alerte : @{triggerBody()?['object']?['properties']?['alertDisplayName']}
Gravité : @{triggerBody()?['object']?['properties']?['severity']}
Heure : @{triggerBody()?['object']?['properties']?['timeGenerated']}
Description : @{triggerBody()?['object']?['properties']?['description']}
```

4. **Enregistre** le playbook
5. Associe-le à tes règles analytiques dans Sentinel

---

## 🎯 Résultat final attendu

À la fin de ce TP, tu dois avoir :

- [x] Une **VM Linux** sur Azure qui génère des logs de sécurité
- [x] Un **Log Analytics Workspace** qui centralise tous les logs
- [x] **Microsoft Sentinel** activé avec des connecteurs de données configurés
- [x] **3 règles de détection** actives (brute force, password spray, hors horaires)
- [x] Un **Dashboard** avec 4 visuels (tentatives, ratio, top IPs, alertes)
- [ ] (Bonus) Un **Playbook d'automatisation** pour les alertes email

---

## 🔧 Dépannage fréquent

| Problème | Cause probable | Solution |
|----------|---------------|----------|
| `Permission denied (publickey)` | Mauvais fichier `.pem` ou permissions | `chmod 400 ton-fichier.pem` |
| Aucun log dans Log Analytics | L'agent n'est pas encore installé | Attendre 15-20 min après l'activation |
| Requête KQL sans résultat | Pas encore assez de données | Générer plus d'activité sur la VM |
| Sentinel ne reçoit pas les alertes | Connecteur mal configuré | Vérifier le statut dans "Connecteurs de données" |
| VM inaccessible en SSH | Le port 22 est bloqué | Vérifier les règles NSG dans Azure |

### Commandes utiles de diagnostic dans la VM

```bash
# Vérifier les logs d'auth en temps réel
sudo tail -f /var/log/auth.log

# Voir les connexions actives
who

# Voir les dernières connexions
last | head -20

# Statut des services
sudo systemctl status fail2ban
sudo systemctl status ssh

# Voir les IPs bannies par fail2ban
sudo fail2ban-client status sshd

# Tester la génération de logs manuellement
logger -p auth.warning "Test log - tentative connexion suspecte"
```

**🧠 Compétences démontrées :** Cloud Azure · Cybersécurité (SIEM) · Linux · KQL · Power BI · DevSecOps

---

## 📚 Ressources complémentaires

- [Documentation Microsoft Sentinel](https://learn.microsoft.com/fr-fr/azure/sentinel/)
- [Référence du langage KQL](https://learn.microsoft.com/fr-fr/azure/data-explorer/kusto/query/)
- [Fail2ban Documentation](https://www.fail2ban.org/wiki/index.php/Main_Page)
- [Azure Monitor Overview](https://learn.microsoft.com/fr-fr/azure/azure-monitor/overview)
- [Power BI + Log Analytics](https://learn.microsoft.com/fr-fr/azure/azure-monitor/logs/log-powerbi)

---

*TP réalisé dans le cadre d'un projet de formation en cybersécurité cloud — Mini SOC Azure*#
