# Lab 2 : Rooting & Contrôles d'Intégrité Android

> Laboratoire de sécurité mobile — Comprendre le rooting et ses impacts sur les protections Android

[![Android](https://img.shields.io/badge/Android-API%2029+-green.svg)](https://developer.android.com)
[![OWASP](https://img.shields.io/badge/OWASP-MASVS%20%7C%20MASTG-orange.svg)](https://mas.owasp.org/)

---

## 📋 Table des matières

- [Fiche périmètre](#-fiche-périmètre)
- [Prérequis](#-prérequis)
- [Étape 1 : Rooter l'AVD](#-étape-1--rooter-lavd)
- [Étape 2 : Installation de l'app de test](#-étape-2--installation-de-lapp-de-test)
- [Scénarios de test](#-scénarios-de-test)
- [Concepts de sécurité Android](#-concepts-de-sécurité-android)
- [Verified Boot & AVB](#-verified-boot--avb)
- [Définition du Rooting](#-définition-du-rooting)
- [Matrice de risques & Mesures défensives](#-matrice-de-risques--mesures-défensives)
- [OWASP MASVS & MASTG](#-owasp-masvs--mastg)
- [Commandes de référence](#-commandes-de-référence)
- [Fiche environnement & Traçabilité](#-fiche-environnement--traçabilité)
- [Remise à zéro](#-remise-à-zéro)
- [Ressources & Glossaire](#-ressources--glossaire)

---

## 📄 Fiche périmètre

| Champ | Valeur |
|-------|--------|
| **App + version** | [Nom de l'app] + [Version] |
| **Support** | AVD / Device laboratoire |
| **Objectif** | Comprendre le rooting et impacts sur les contrôles d'intégrité |
| **Données** | Fictives |
| **Réseau** | Test |

> La fiche périmètre définit clairement ce qui est testé et dans quelles conditions. Sans périmètre défini, pas d'audit fiable.

---

## 🔧 Prérequis

- **AVD démarré** (Android Studio → Device Manager → Start)
- **ADB fonctionnel** : `adb devices` doit détecter l'émulateur
- **AVD propre** : écran d'accueil Android, aucun compte personnel
- **API 29+** recommandé pour observer les mécanismes de sécurité modernes

![AVD Démarré - Écran d'accueil](screenshots/01_avd_home.png)

> ⚠️ **Erreur à éviter** : Réutiliser un AVD « sale » avec des applications résiduelles — vos résultats seront faussés.

---

## 📲 Étape 1 : Rooter l'AVD

### Pourquoi ?

Disposer des privilèges élevés sur un environnement jetable pour observer l'impact du root et des contrôles d'intégrité. Un environnement rooté permet d'accéder à des zones normalement protégées du système.

### Commandes (émulateur uniquement)

```bash
adb root          # Active le serveur ADB en mode root
adb remount       # Monte /system en lecture/écriture si verity est permissif
```

### Vérifications

```bash
adb shell id                                    # Chercher uid=0(root)
adb shell getprop ro.boot.verifiedbootstate    # Souvent "orange" ou "yellow" si verity désactivé
adb shell getprop ro.boot.veritymode            # État de verity
adb shell getprop ro.boot.vbmeta.device_state   # État du vbmeta
adb shell "su -c id"                            # Tester si su est disponible
```

### Interprétation des résultats

| Résultat | Signification |
|----------|---------------|
| `uid=0(root)` | Privilèges root confirmés |
| `verifiedbootstate` = orange/yellow | Intégrité du système non garantie |
| `su -c id` fonctionne | Accès superutilisateur disponible |

![Résultat adb root et adb shell id](screenshots/02_adb_root_id.png)

### Option permissive (si `adb root` échoue)

```bash
adb disable-verity
adb reboot
adb remount
```

### Journalisation

```bash
adb logcat -d | tail -n 200 > logcat_root_check.txt
```

> 🔒 **Concept** : Verity vérifie l'intégrité du système de fichiers. Le désactiver supprime la garantie d'intégrité — comme retirer le scellé de sécurité d'un produit.

---

## 📱 Étape 2 : Installation de l'app de test

```bash
adb install app-debug.apk
```

Ou via Android Studio : **Run** sur le projet.

### Vérifications

- L'application s'ouvre correctement
- Un scénario simple est réalisable
- **Noter la version** dans le rapport (crucial pour la reproductibilité)

![App lancée](screenshots/03_app_launched.png)

---

## 🎯 Scénarios de test

### Scénario 1 : Vérification de l'état root et intégrité système

**Objectif** : Confirmer l'obtention des privilèges root et l'impact sur l'état de Verified Boot.

**Étapes** :
1. Lancer `adb root` puis `adb remount`
2. Exécuter `adb shell id` → doit afficher `uid=0(root)`
3. Vérifier `adb shell getprop ro.boot.verifiedbootstate` → attendu : orange ou yellow après root
4. Comparer avec l'état initial (green) sur AVD non rooté

**Résultat attendu** : Accès root confirmé ; état verifiedbootstate modifié indiquant une perte de garantie d'intégrité.

![Scénario 1 - Vérification root](screenshots/scenario1_root_verification.png)

---

### Scénario 2 : Accès aux données de l'application avec privilèges root

**Objectif** : Observer si les données sensibles de l'app sont accessibles depuis un shell root.

**Étapes** :
1. Utiliser l'app (connexion, saisie de données fictives)
2. Avec shell root : `adb shell` puis `su` ou `adb shell "su -c ..."`
3. Explorer `adb shell "su -c ls -la /data/data/[package_name]/shared_prefs/"`
4. Vérifier si des fichiers contiennent des données sensibles en clair

**Résultat attendu** : Identification de données potentiellement exposées (préférences, tokens, etc.) ou confirmation d'un stockage sécurisé.

![Scénario 2 - Accès données app](screenshots/scenario2_data_access.png)

---

### Scénario 3 : Comportement de l'app sur appareil rooté vs non rooté

**Objectif** : Déterminer si l'application détecte le root et adapte son comportement.

**Étapes** :
1. Tester l'app sur AVD propre (non rooté) — fonctionnalités de base
2. Rooter l'AVD
3. Relancer l'app et refaire les mêmes actions
4. Observer : refus d'exécution, message d'avertissement, ou comportement inchangé ?
5. Capturer `adb logcat` pendant l'utilisation pour détecter des logs sensibles

**Résultat attendu** : Rapport sur la détection (ou non) du root et les mesures prises par l'app.

![Scénario 3 - Comportement app rooté](screenshots/scenario3_root_detection.png)

---

## 🔐 Concepts de sécurité Android

La sécurité Android repose sur plusieurs couches :

1. **Sandboxing des applications** : Chaque app est isolée des autres
2. **Modèle de permissions** : Contrôle d'accès aux ressources sensibles
3. **Isolation et intégrité globale du système** : Protection contre les modifications non autorisées

> *Analogie* : Le sandboxing, c'est mettre chaque application dans sa propre salle de classe fermée. Le modèle de permissions, c'est demander l'autorisation avant d'utiliser certains équipements. L'intégrité système, c'est verrouiller le bâtiment entier.

**Ressource** : [Android Security - source.android.com](https://source.android.com/docs/security)

---

## 🛡️ Verified Boot & AVB

### Verified Boot

**Objectif principal** : Garantir que le système qui démarre est celui prévu par le fabricant, sans modifications malveillantes.

**Chain of trust** : Série de vérifications où chaque composant vérifie l'authenticité du suivant avant de lui faire confiance. Comme une chaîne de gardiens où chacun vérifie l'identité du suivant.

**Pourquoi c'est critique ?** Si le démarrage est compromis, toutes les protections ultérieures peuvent être contournées — comme une forteresse dont la porte principale serait ouverte.

### Interprétation des couleurs (verifiedbootstate)

| Couleur | Signification |
|---------|---------------|
| **Green** | Tout est normal, système vérifié et intègre |
| **Yellow/Orange** | Avertissement, système modifié mais fonctionnel |
| **Red** | Danger, intégrité compromise |

```bash
adb shell getprop ro.boot.verifiedbootstate  # Attendu "green" sur image signée
```

![État Verified Boot](screenshots/04_verified_boot_state.png)

### AVB (Android Verified Boot)

AVB est la version 2.0 de Verified Boot, plus moderne et flexible.

- Vérification d'intégrité moderne
- Protection contre le rollback (empêche d'installer d'anciennes versions vulnérables)
- Flexibilité accrue pour les fabricants

**Protection anti-rollback** : Empêche d'installer d'anciennes versions du système contenant des failles connues — comme empêcher le remplacement d'une serrure moderne par un ancien modèle facilement crochetable.

---

## 🔑 Définition du Rooting

- **Root** = privilèges super-utilisateur sur le système
- Cela modifie les protections et la confiance du système
- Utile en laboratoire pour observer certains comportements
- Risqué → nécessite isolement, traçabilité et reset

> *Analogie* : Le rooting, c'est avoir un passe-partout pour toutes les portes d'un bâtiment. Très utile pour la maintenance, mais dangereux si mal utilisé.

**En labo, un environnement privilégié peut aider à** :
- Observer des artefacts système normalement inaccessibles
- Analyser les comportements runtime de l'app à bas niveau
- Tester la robustesse du stockage face à un attaquant privilégié

> ⚠️ **Labo autorisé uniquement.** Ne jamais rooter un appareil personnel.

---

## ⚠️ Matrice de risques & Mesures défensives

### 8 Risques

| # | Risque |
|---|--------|
| 1 | Intégrité non garantie → conclusions biaisées sur la sécurité réelle |
| 2 | Surface d'attaque accrue si l'appareil sort du labo → exposition à des menaces externes |
| 3 | Données sensibles exposées si présentes → violation potentielle de confidentialité |
| 4 | Instabilité système → tests non reproductibles et résultats incohérents |
| 5 | Mélange comptes perso/test → fuite possible d'informations personnelles |
| 6 | Mauvais nettoyage fin de séance → persistance de données sensibles |
| 7 | Réseau non isolé → effets involontaires sur systèmes externes |
| 8 | Traçabilité insuffisante → impossible de reproduire ou d'auditer les tests |

### 8 Mesures défensives

| # | Mesure |
|---|--------|
| 1 | Réseau isolé pour éviter toute communication non contrôlée |
| 2 | Données fictives uniquement pour éliminer tout risque de fuite réelle |
| 3 | Device/AVD dédié exclusivement aux tests de sécurité |
| 4 | Snapshots ou wipe en fin de séance pour ne laisser aucune trace |
| 5 | Journal de configuration détaillé pour assurer la reproductibilité |
| 6 | Aucun compte personnel pour éviter tout mélange de données |
| 7 | Contrôle strict des APK installées pour limiter les risques |
| 8 | Horodatage + captures des étapes pour une traçabilité complète |

---

## 📚 OWASP MASVS & MASTG

### MASVS (2 exigences)

| Exigence | Description |
|----------|-------------|
| **STORAGE-1** | Les données sensibles (API keys, mots de passe, tokens) doivent être stockées de manière sécurisée avec un chiffrement approprié |
| **NETWORK-1** | Les communications réseau doivent utiliser TLS avec une configuration correcte et vérification des certificats |

### MASTG (2 idées de tests)

| Test | Procédure |
|------|-----------|
| **Examen SharedPreferences** | Vérifier si les fichiers dans `/data/data/[package_name]/shared_prefs/` contiennent des informations sensibles en clair |
| **Analyse des logs** | Utiliser `adb logcat` pour détecter des fuites d'informations sensibles pendant l'exécution |

> Si le **MASVS** dit « quoi » vérifier, le **MASTG** explique « comment » le vérifier.

---

## 📋 Commandes de référence

### Rooting (synthèse)

```bash
adb devices
adb root
adb remount
adb shell id
adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.vbmeta.device_state
adb shell "su -c id"
```

### Option permissive

```bash
adb disable-verity
adb reboot
adb remount
adb logcat -d | tail -n 200 > logcat_root_check.txt
```

### Fastboot (labo uniquement — device déverrouillé)

```bash
fastboot oem device-info
fastboot getvar avb_boot_state
fastboot boot magisk_patched.img   # Boot temporaire, PAS de flash
```

> ⚠️ **Dépannage** : Si `adb root` échoue avec « adbd cannot run as root in production builds », utiliser un émulateur ou un device de labo configuré pour le root.

> ⚠️ **Avertissement critique** : Ne jamais flasher ni manipuler un device personnel. Manipuler le bootloader peut rendre l'appareil inutilisable (« brick »).

---

## 📊 Fiche environnement & Traçabilité

| Champ | Valeur |
|-------|--------|
| **Date / Auteur** | [Date] / [Nom] |
| **Support** | AVD / Device labo |
| **Version Android / API** | [Ex: Android 13 / API 33] |
| **App + version** | [Nom] [Version] |
| **Scénario 1** | [Observations factuelles] |
| **Scénario 2** | [Observations factuelles] |
| **Scénario 3** | [Observations factuelles] |
| **Limites** | [Ex: émulateur uniquement, pas de device physique] |
| **Reset effectué** | Oui / Non + preuve |

### Captures à inclure

- [ ] App lancée
- [ ] Résultat de `adb root`
- [ ] Résultat de `getprop ro.boot.verifiedbootstate`
- [ ] Accès aux données (scénario 2)
- [ ] Écran de remise à zéro (assistant initial)

---

## 🔄 Remise à zéro

### AVD (obligatoire en fin de séance)

**Via UI** : Android Studio → Device Manager → **Wipe Data** (ou Delete puis Recreate)

**Via commandes** :
```bash
adb emu avd stop
adb emu avd wipe-data
```

### Device labo (si utilisé)

1. Paramètres système → Réinitialisation usine → Redémarrer
2. Vérifier absence de comptes/profils/certificats
3. **Preuve** : assistant de configuration initial

**Option fastboot (labo uniquement)** :
```bash
fastboot erase userdata
# Puis redémarrer
```

> ⚠️ Ne pas réinitialiser, c'est comme laisser un laboratoire avec des produits chimiques sur les tables — dangereux pour la prochaine session.

---

## 📖 Ressources & Glossaire

### Glossaire

| Terme | Définition |
|-------|------------|
| **ADB** | Android Debug Bridge — outil CLI pour communiquer avec un appareil Android |
| **AVD** | Android Virtual Device — émulateur Android |
| **Bootloader** | Programme qui charge le système d'exploitation au démarrage |
| **Fastboot** | Mode spécial permettant de flasher les partitions système |
| **Partition** | Section du stockage dédiée à un usage (system, data, boot) |
| **Root** | Utilisateur avec privilèges administrateur complets |
| **Sandbox** | Environnement isolé où une application s'exécute |
| **Verity** | Système de vérification d'intégrité des partitions |
| **AVB** | Android Verified Boot — vérification du démarrage |

### Architecture de sécurité Android (simplifiée)

```
[Matériel sécurisé] → [Bootloader vérifié] → [Kernel] → [Système Android] → [Apps sandboxées]
```

### Impact du rooting sur les protections

```
Normal :   App → Sandbox → Permissions → Système (protégé)
Rooté :    App → Sandbox → Permissions → Système (modifiable) ← Root
```

### Liens utiles

- [Android Security](https://source.android.com/docs/security)
- [Verified Boot](https://source.android.com/docs/security/features/verifiedboot)
- [AVB](https://source.android.com/docs/security/features/verifiedboot/avb)
- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [OWASP MASTG](https://mas.owasp.org/MASTG/)

---

## 📁 Structure des captures d'écran

Créez le dossier `screenshots/` et ajoutez vos captures :

```
screenshots/
├── 01_avd_home.png
├── 02_adb_root_id.png
├── 03_app_launched.png
├── 04_verified_boot_state.png
├── scenario1_root_verification.png
├── scenario2_data_access.png
└── scenario3_root_detection.png
```

---

**Lab réalisé dans le cadre du cours de sécurité mobile — Environnement de test uniquement.**
