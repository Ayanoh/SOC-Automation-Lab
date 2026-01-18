# 📤 How to Upload This Project to GitHub

This guide explains **step-by-step** how to upload all documentation files to your GitHub repository.

---

## ✅ ÉTAPE 1 : Télécharger Tous les Fichiers

1. **Télécharge tous les fichiers** que je t'ai créés depuis ce chat
2. **Place-les sur ton PC** dans un dossier, par exemple : `~/Documents/SOC-Automation-Lab/`

**Structure attendue** :
```
SOC-Automation-Lab/
├── README.md
├── PROJECT_STATUS.md
├── ARCHITECTURE.md
├── ROADMAP.md
├── TROUBLESHOOTING.md
├── .gitignore
├── docs/
├── configs/
├── scripts/
└── images/
```

---

## ✅ ÉTAPE 2 : Créer le Repository GitHub

### Via le Site Web GitHub

1. Va sur https://github.com/Ayanoh
2. Clique sur **"New"** (bouton vert en haut à droite)
3. Remplis :
   - **Repository name** : `SOC-Automation-Lab`
   - **Description** : `Enterprise SOC Lab with automated threat detection, SOAR orchestration, and incident response`
   - **Public** ✅ (coché)
   - **Add a README** ❌ (décoché - on a déjà notre README)
   - **Add .gitignore** ❌ (décoché - on a déjà notre .gitignore)
   - **Choose a license** ❌ (décoché pour l'instant)
4. Clique sur **"Create repository"**

GitHub va te montrer une page avec des instructions. **Ne les suis PAS encore**, continue ici.

---

## ✅ ÉTAPE 3 : Initialiser Git Localement

Ouvre un terminal et exécute :

```bash
# Va dans le dossier du projet
cd ~/Documents/SOC-Automation-Lab/

# Initialise Git
git init

# Vérifie que tous les fichiers sont là
ls -la

# Ajoute tous les fichiers
git add .

# Vérifie ce qui va être committé
git status

# Tu devrais voir :
# - README.md
# - PROJECT_STATUS.md
# - ARCHITECTURE.md
# - ROADMAP.md
# - TROUBLESHOOTING.md
# - .gitignore
# - docs/ (si tu as des fichiers dedans)
# - configs/ (si tu as des fichiers dedans)
# - images/ (tes screenshots)
```

---

## ✅ ÉTAPE 4 : Premier Commit

```bash
# Configure ton nom et email (si pas déjà fait)
git config --global user.name "Oussama EL Maskaoui"
git config --global user.email "oussama.elmaskaoui@gmail.com"

# Crée le premier commit
git commit -m "Initial commit: SOC Automation Lab documentation"

# Vérifie que le commit est créé
git log
# Tu devrais voir ton commit avec le message
```

---

## ✅ ÉTAPE 5 : Connecter au Repository GitHub

```bash
# Ajoute le remote (remplace [username] par ton nom d'utilisateur)
git remote add origin https://github.com/Ayanoh/SOC-Automation-Lab.git

# Vérifie que le remote est ajouté
git remote -v
# Tu devrais voir :
# origin  https://github.com/Ayanoh/SOC-Automation-Lab.git (fetch)
# origin  https://github.com/Ayanoh/SOC-Automation-Lab.git (push)
```

---

## ✅ ÉTAPE 6 : Push vers GitHub

```bash
# Renomme la branche en 'main' (GitHub utilise 'main' maintenant)
git branch -M main

# Push vers GitHub
git push -u origin main

# GitHub va te demander de t'authentifier :
# Option 1 : Username + Password (ou Personal Access Token)
# Option 2 : SSH key (si configurée)
```

### 🔑 Si GitHub demande un Personal Access Token

1. Va sur https://github.com/settings/tokens
2. Clique sur **"Generate new token"** → **"Generate new token (classic)"**
3. Remplis :
   - **Note** : "SOC Lab Project"
   - **Expiration** : 90 days (ou plus)
   - **Scopes** : ✅ `repo` (toutes les permissions repo)
4. Clique sur **"Generate token"**
5. **COPIE LE TOKEN** (tu ne le reverras plus !)
6. Utilise ce token comme **password** quand Git te le demande

---

## ✅ ÉTAPE 7 : Vérifier sur GitHub

1. Va sur https://github.com/Ayanoh/SOC-Automation-Lab
2. Tu devrais voir :
   - ✅ Tous les fichiers uploadés
   - ✅ README.md affiché automatiquement
   - ✅ Badges et structure visible

**Si tout est là → SUCCÈS ! 🎉**

---

## ✅ ÉTAPE 8 : Ajouter les Images

### Télécharge tes Screenshots

1. **Copie tous tes screenshots** dans le dossier `images/`
2. **Renomme-les** avec des noms descriptifs :
   - `architecture-complete.png` (ton diagramme)
   - `n8n-workflow.png`
   - `thehive-alert.png`
   - `cortex-enrichment.png`
   - `slack-notifications.png`

### Upload les Images

```bash
# Retourne dans le dossier du projet
cd ~/Documents/SOC-Automation-Lab/

# Ajoute les nouvelles images
git add images/

# Commit
git commit -m "Add project screenshots and architecture diagram"

# Push
git push

# Vérifie sur GitHub que les images sont là
```

---

## ✅ ÉTAPE 9 : Mettre à Jour le README (Si Besoin)

Si les images ne s'affichent pas correctement dans le README :

```bash
# Édite README.md
nano README.md

# Vérifie que les chemins des images sont corrects :
![SOC Lab Architecture](images/architecture-complete.png)
# OU
![SOC Lab Architecture](images/labwazuh.png)

# Si tu as changé quelque chose :
git add README.md
git commit -m "Fix image paths in README"
git push
```

---

## 🔄 Commandes Git Utiles Pour Après

### Ajouter de Nouveaux Fichiers

```bash
# Ajoute un nouveau fichier
git add [nom-du-fichier]

# Ou ajoute tous les changements
git add .

# Commit avec message
git commit -m "Description du changement"

# Push vers GitHub
git push
```

### Vérifier l'État du Repo

```bash
# Voir les fichiers modifiés
git status

# Voir l'historique des commits
git log --oneline

# Voir les différences
git diff
```

### Revenir en Arrière (Si Erreur)

```bash
# Annuler le dernier commit (garde les changements)
git reset --soft HEAD~1

# Annuler les changements non commités
git checkout -- [fichier]

# Annuler TOUT (DANGER !)
git reset --hard HEAD
```

---

## 📋 Checklist Finale

Avant de partager ton repo en entretien :

```
□ README.md s'affiche correctement sur GitHub
□ Toutes les images sont visibles
□ Pas de credentials/API keys dans les fichiers
□ Tous les liens internes fonctionnent
□ .gitignore empêche les fichiers sensibles
□ Repository est Public (pas Private)
□ Description du repo est remplie
□ Topics/Tags ajoutés (optional) : soc, siem, soar, cybersecurity
```

---

## 🎯 Ajouter des Topics au Repository

1. Va sur ton repo : https://github.com/Ayanoh/SOC-Automation-Lab
2. Clique sur **⚙️** à côté de "About"
3. Ajoute ces topics :
   - `soc`
   - `siem`
   - `soar`
   - `cybersecurity`
   - `wazuh`
   - `thehive`
   - `threat-intelligence`
   - `incident-response`
   - `automation`
4. Sauvegarde

---

## 📱 Partager ton Projet

### Lien Direct

```
https://github.com/Ayanoh/SOC-Automation-Lab
```

### Sur LinkedIn

```
🚀 Excited to share my latest project: SOC Automation Lab

I built a complete Security Operations Center lab demonstrating:
✅ Automated threat detection (Wazuh)
✅ SOAR orchestration (n8n)
✅ Threat intelligence enrichment (Cortex)
✅ Incident management (TheHive)

The project includes full documentation, architecture diagrams, and troubleshooting guides.

Check it out: https://github.com/Ayanoh/SOC-Automation-Lab

#Cybersecurity #SOC #ThreatDetection #Automation #InfoSec
```

### Dans ton CV

```
Projects:
- SOC Automation Lab (https://github.com/Ayanoh/SOC-Automation-Lab)
  Enterprise-grade security operations lab with automated detection,
  orchestration, and incident response capabilities
```

---

## 🆘 Si tu Rencontres un Problème

### Erreur : "Permission denied (publickey)"

**Solution** : Utilise HTTPS au lieu de SSH
```bash
git remote set-url origin https://github.com/Ayanoh/SOC-Automation-Lab.git
```

### Erreur : "fatal: remote origin already exists"

**Solution** :
```bash
git remote remove origin
git remote add origin https://github.com/Ayanoh/SOC-Automation-Lab.git
```

### Erreur : "Updates were rejected"

**Solution** :
```bash
# Récupère les changements du remote
git pull origin main --rebase

# Puis push à nouveau
git push origin main
```

---

## ✅ C'est Tout !

Tu as maintenant un projet GitHub professionnel et complet ! 🎉

**Prochaines étapes** :
1. Partage le lien dans ton CV
2. Mentionne-le sur LinkedIn
3. Utilise-le en entretien comme portfolio
4. Continue à l'améliorer avec le ROADMAP

---

**Besoin d'aide ?** Contacte-moi ou pose une question sur GitHub Discussions.
