<div align="center">

# 🔎 SOC Home Lab : Wazuh (SIEM)

### Déploiement d'un SIEM, détection d'activités malveillantes mappées MITRE ATT&CK et script de supervision PowerShell intégré

![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-3AB7CB?logo=wazuh&logoColor=white)
![VMware](https://img.shields.io/badge/VMware%20Workstation-17-607078?logo=vmware&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu%20Server-Wazuh%20server-E95420?logo=ubuntu&logoColor=white)
![Windows](https://img.shields.io/badge/Windows-H%C3%B4te%20surveill%C3%A9-0078D6?logo=windows&logoColor=white)
![Kali](https://img.shields.io/badge/Kali%20Linux-Attaquant-557C94?logo=kalilinux&logoColor=white)
![MITRE](https://img.shields.io/badge/MITRE%20ATT%26CK-mapp%C3%A9-red)
![PowerShell](https://img.shields.io/badge/PowerShell-Script-5391FE?logo=powershell&logoColor=white)

> 🇬🇧 [English version below](#-english-summary) · 🇫🇷 Version française ci-dessous

</div>

---

> ⚠️ **Périmètre autorisé.** Toutes les manipulations ont été réalisées dans un **réseau de laboratoire isolé**, sur des machines virtuelles et des comptes **fictifs** créés pour l'exercice.

---

## 📌 Résumé du projet

J'ai monté un **laboratoire SOC personnel** pour pratiquer en dehors des cours. Le principe : déployer le SIEM **Wazuh**, générer sur un poste Windows des activités typiques d'un attaquant, **analyser les alertes** produites, puis écrire mon **propre script PowerShell de supervision** qui remonte ses résultats dans le SIEM.

```
 Activité malveillante simulée ──► Journaux Windows / FIM ──► Agent Wazuh ──► Manager (règles)
        (poste Windows)                                                         │
                                                                                ▼
                                          Dashboard : alerte + niveau + MITRE ATT&CK ──► Analyse / triage
```

> 🎯 **Pourquoi ce projet ?** C'est le quotidien d'un analyste SOC de niveau 1 : recevoir une alerte, comprendre **ce qui s'est passé**, **à quel point c'est grave** et **à quelle technique d'attaque** elle correspond.

---

## 🎯 Objectifs

- ✅ Déployer un SIEM (Wazuh) pour collecter et analyser les journaux de sécurité d'un poste Windows
- ✅ Simuler des actions malveillantes (échecs de connexion, création d'un compte administrateur, persistance) et confirmer la détection
- ✅ Écrire mon propre script de supervision PowerShell et l'intégrer à Wazuh

---

## 🗺️ Architecture du laboratoire

| Machine | Rôle | Système |
| --- | --- | --- |
| **VM1** | Serveur Wazuh : indexer, manager, dashboard | Ubuntu Server |
| **VM2** | Cible / poste surveillé (agent Wazuh) | Windows |
| **VM3** | Machine attaquante | Kali Linux |

> **Hyperviseur :** VMware Workstation Pro 17. Toutes les machines sont sur le **même réseau isolé**.

![Schéma réseau](screenshots/00-schema_reseau_wazuh.png)

**Les composants de Wazuh, en bref :**

| Composant | Rôle |
| --- | --- |
| **Agent** | Installé sur la machine surveillée ; collecte les journaux et surveille l'intégrité des fichiers |
| **Manager** | Reçoit les événements, les compare aux **règles de détection** et génère les alertes |
| **Indexer** | Stocke et indexe les alertes pour les rechercher rapidement |
| **Dashboard** | Interface web pour visualiser, filtrer et analyser les alertes |

---

## 🛠️ Ce que j'ai fait

### 1️⃣ Déploiement de Wazuh

Installation de l'**indexer**, du **manager** et du **dashboard** sur Ubuntu Server avec le **script d'installation officiel**.

**Pourquoi ?** Le script officiel installe et relie les trois composants avec des certificats générés automatiquement, ce qui évite les erreurs de configuration manuelle.

### 2️⃣ Agent Windows

Installation de l'**agent** sur le poste Windows et vérification de sa connexion au manager. Le journal de l'agent confirme :

```
Connected to the server
```

**Pourquoi ?** Sans agent connecté, aucun événement ne remonte. Vérifier ce message est le premier réflexe quand « rien n'apparaît dans le dashboard ».

### 3️⃣ Tests de détection

Sur le poste surveillé, j'ai généré **trois types d'activité** et confirmé chacun dans le dashboard :

| Test | Action | Règle Wazuh | MITRE ATT&CK |
| --- | --- | --- | --- |
| **A** | Échecs de connexion répétés (style force brute) | `60122` | **T1110** : Brute Force |
| **B** | Création d'un compte local ajouté au groupe Administrateurs | `60154` (niveau **12**) | **T1136** / **T1098** |
| **C** | Dépôt d'un fichier dans le dossier de démarrage (persistance) | `554` (FIM) | **T1547** |

#### 🅰️ Test A : échecs de connexion répétés

- **Ce que je simule :** une personne qui essaie plusieurs mots de passe sur un compte.
- **Ce que Windows enregistre :** des événements d'**échec d'ouverture de session** (Event ID **4625**).
- **Ce que Wazuh détecte :** la règle **60122**, associée à la technique **T1110 (Brute Force)**.
- **Comment je l'analyse :** je regarde le **compte ciblé**, la **fréquence** des échecs et l'**origine** de la tentative pour décider s'il s'agit d'une erreur de saisie ou d'une attaque.

#### 🅱️ Test B : création d'un compte administrateur

- **Ce que je simule :** un attaquant qui crée un compte local et l'élève au rang d'administrateur pour garder un accès.
- **Ce que Wazuh détecte :** la règle **60154**, de **niveau 12** (alerte à fort impact), liée aux techniques **T1136** (création de compte) et **T1098** (manipulation de compte).
- **Pourquoi c'est critique :** un nouveau compte administrateur inattendu est un signal classique de compromission.

#### 🅲 Test C : persistance par le dossier de démarrage

- **Ce que je simule :** un fichier placé dans le dossier **Startup**, qui s'exécutera à chaque ouverture de session.
- **Ce que Wazuh détecte :** la règle **554**, produite par le module **FIM** (*File Integrity Monitoring*, surveillance de l'intégrité des fichiers), liée à **T1547**.
- **Pourquoi c'est important :** la persistance permet à un attaquant de survivre à un redémarrage.

### 4️⃣ Script de supervision PowerShell

J'ai écrit un script PowerShell qui :

- ✔️ vérifie l'état des **services** ;
- ✔️ contrôle l'**espace disque** ;
- ✔️ mesure le **CPU et la mémoire** ;
- ✔️ relève les **erreurs système** ;
- ✔️ génère un **rapport HTML** ;
- ✔️ **écrit un événement dans le journal Windows**.

Wazuh récupère cet événement (règle **60602**), ce qui ferme la boucle : *mon* script alimente *mon* SIEM. Je l'ai aussi **planifié toutes les heures** avec le planificateur de tâches.

**Pourquoi ?** Passer par le journal Windows est simple et fiable : l'agent Wazuh lit déjà ce journal, donc aucune intégration complexe n'est nécessaire.

📸 **Captures d'écran :** voir le dossier `/screenshots`. La documentation complète étape par étape est dans `/docs`.

---

## 🧰 Outils et compétences

`Wazuh (SIEM)` · `VMware` · `Windows` · `Ubuntu Server` · `PowerShell` · `Planificateur de tâches` · `Active Directory (bases)` · `MITRE ATT&CK` · `FIM` · `Analyse de logs` · `Triage d'incidents`

---

## 🧩 Ce que j'ai appris

- **Le rôle de chaque composant** de Wazuh (indexer, manager, dashboard, agent) et comment ils communiquent
- **Lire un log** pour réparer un service qui ne démarre pas
- **Analyser une alerte** : règle, niveau de gravité, Event ID Windows, puis la rattacher à une technique **MITRE ATT&CK**
- Qu'une alerte n'a de valeur que si on **comprend le contexte** : qui, quoi, quand, depuis où

---

## 🚀 Prochaines étapes

- [ ] Écrire ma **propre règle de détection personnalisée** dans Wazuh
- [ ] Tester le lab avec des **attaques réseau depuis la machine Kali**, dans un environnement isolé

---

## 🔐 Hygiène de sécurité du dépôt

- Environnement **isolé**, machines virtuelles et comptes **fictifs**
- **Aucun** mot de passe, jeton, clé ou donnée personnelle publié

---

## 📂 Structure du dépôt

```
📦 wazuh-soc-lab
 ┣ 📂 screenshots   # Captures et schéma réseau
 ┣ 📂 docs          # Documentation complète étape par étape
 ┗ 📄 README.md
```

---

## 👤 À propos

**Youssef** : étudiant au **Cégep de Saint-Hyacinthe** (DEC en informatique / réseaux et cybersécurité). Je vise une carrière en **cybersécurité (SOC)**. J'apprends vite et j'aime comprendre comment les choses fonctionnent, c'est pourquoi j'ai monté ce laboratoire par moi-même.

🔎 À la recherche d'un **stage** · 💼 *[LinkedIn](LIEN_VERS_TON_LINKEDIN)*

---

<a id="-english-summary"></a>

## 🇬🇧 English summary

**SOC Home Lab: Wazuh (SIEM)** is a home lab where I deployed the Wazuh SIEM, generated attacker-style activity on a Windows machine, analyzed the resulting alerts, and wrote my own PowerShell monitoring script that reports back into the SIEM.

**Architecture:** Ubuntu Server (Wazuh indexer + manager + dashboard) · Windows (monitored host with agent) · Kali Linux (attacker), all on an isolated VMware Workstation Pro 17 network.

**Detection tests (confirmed in the dashboard):**

| Test | Action | Wazuh rule | MITRE ATT&CK |
| --- | --- | --- | --- |
| A | Repeated failed logons (brute-force style) | 60122 | T1110 – Brute Force |
| B | Create a local account and add it to Administrators | 60154 (level 12) | T1136 / T1098 |
| C | Drop a file in the Startup folder (persistence) | 554 (FIM) | T1547 |

**Monitoring script:** a PowerShell script checks services, disk space, CPU/memory and system errors, builds an HTML report and writes an event to the Windows log. Wazuh picks it up (rule 60602). It runs hourly through a scheduled task.

**Skills:** Wazuh · VMware · Windows · Ubuntu Server · PowerShell · MITRE ATT&CK · log analysis · incident triage.

**Next steps:** a custom Wazuh detection rule, network attacks from the Kali machine in an isolated environment.

> All activity was performed in an isolated lab, on virtual machines and fictitious accounts. No secrets or personal data are published.
