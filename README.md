# Lab 2 : Rooting & Sécurité Android

> Comprendre le rooting et tester les vulnérabilités OWASP sur une app Android

---

## Étape 1 : Fiche périmètre

| Champ | Valeur |
|-------|--------|
| **Support** | AVD / Device labo |
| **Objectif** | Rooting et impacts sur les contrôles d'intégrité |
| **Données** | Fictives |
| **Réseau** | Test |

---

## Étape 2 : Démarrer un AVD propre

- Android Studio → Device Manager → **Start** (ou créer un AVD avec API 29+)
- Vérifier : écran d'accueil Android, **aucun compte personnel**
- `adb devices` doit détecter l'émulateur

> Préférer les versions récentes (API 29+) pour observer les mécanismes de sécurité modernes.

---

## Étape 3 : Rooter l'AVD

```bash
adb devices
adb root
adb remount
adb shell id   # → uid=0(root)
```

**Vérifications :**
```bash
adb shell getprop ro.boot.verifiedbootstate   # green/orange/yellow
adb shell "su -c id"                          # Tester su
```

**Option si adb root échoue :** `adb disable-verity` → `adb reboot` → `adb remount`

![ADB Device](adbDevice.png)
![Root - uid](rootetuid.png)

---

## Étape 4 : Installer et lancer l'app

```bash
adb install app-debug.apk
```

Ou via Android Studio : **Run**. Vérifier que l'app s'ouvre et noter la version.

---

## Étape 5 : Scénarios de test

### Scénario 1 : SQL Injection

Test d'injection SQL dans les champs de saisie de l'app.

![SQL Injection](sql-injection.png)

---

### Scénario 2 : Insecure Data Storage

Vérification du stockage des données sensibles (SharedPreferences, fichiers) avec accès root.

![Insecure Data Storage](insecuredatastorage.png)

---

### Scénario 3 : Input Validation (URL)

Test de validation des URLs et failles d'input (XXE, injection via URLs).

![Input Validation - URL](url.png)

---

## Étape 6 : Android Security (résumé)

1. **Sandboxing** : Chaque app est isolée des autres
2. **Modèle de permissions** : Contrôle d'accès aux ressources sensibles
3. **Intégrité système** : Protection contre les modifications non autorisées

---

## Étape 7 : Verified Boot

**Objectif** : Garantir que le système qui démarre est celui prévu par le fabricant, sans modification malveillante.

**Chain of trust** : Chaque composant vérifie l'authenticité du suivant avant de lui faire confiance.

| Couleur | Signification |
|---------|---------------|
| Green | Système vérifié et intègre |
| Yellow/Orange | Système modifié |
| Red | Intégrité compromise |

---

## Étape 8 : AVB (Android Verified Boot)

AVB = version 2.0 de Verified Boot. Ajoute vérification d'intégrité moderne + **protection anti-rollback** (empêche d'installer d'anciennes versions vulnérables).

---

## Étape 9 : Définition du Rooting

- **Root** = privilèges super-utilisateur. Modifie les protections et la confiance du système.
- Utile en labo pour observer comportements à bas niveau, tester le stockage face à un attaquant privilégié.
- Risqué → isolement + traçabilité + reset. **Labo autorisé uniquement.**

---

## Étape 10 : Matrice de risques (8 risques)

1. Intégrité non garantie → conclusions biaisées
2. Surface d'attaque accrue → exposition aux menaces
3. Données sensibles exposées → violation confidentialité
4. Instabilité système → tests non reproductibles
5. Mélange comptes perso/test → fuite d'informations
6. Mauvais nettoyage → persistance de données sensibles
7. Réseau non isolé → effets sur systèmes externes
8. Traçabilité insuffisante → impossible d'auditer

---

## Étape 11 : Mesures défensives (8 mesures)

1. Réseau isolé
2. Données fictives uniquement
3. Device/AVD dédié aux tests
4. Snapshots ou wipe en fin de séance
5. Journal de configuration détaillé
6. Aucun compte personnel
7. Contrôle strict des APK installées
8. Horodatage + captures pour traçabilité

---

## Étape 12 : OWASP MASVS (2 exigences)

- **STORAGE-1** : Données sensibles stockées de façon sécurisée (chiffrement approprié)
- **NETWORK-1** : TLS avec configuration correcte et vérification des certificats

---

## Étape 13 : OWASP MASTG (2 idées de tests)

- Examiner `/data/data/[package]/shared_prefs/` pour données sensibles en clair
- Analyser `adb logcat` pour détecter fuites d'informations pendant l'exécution

---

## Étape 14 : Commandes de référence

```bash
adb devices
adb root
adb remount
adb shell id
adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.vbmeta.device_state
adb shell "su -c id"
```

**Journalisation :** `adb logcat -d | tail -n 200 > logcat_root_check.txt`

---

## Étape 15 : Fiche environnement (traçabilité)

| Champ | Valeur |
|-------|--------|
| Date / Auteur | |
| Support | AVD / Device labo |
| Version Android / API | |
| App + version | |
| Scénario 1-2-3 | Observations factuelles |
| Limites | |
| Reset effectué | Oui/Non + preuve |

---

## Étape 17 : Remise à zéro AVD (obligatoire fin de séance)

**Via UI :** Android Studio → Device Manager → Wipe Data (ou Delete puis Recreate)

**Via commandes :**
```bash
adb emu avd stop
adb emu avd wipe-data
```

**Pourquoi c'est crucial :** Ne pas réinitialiser l'environnement, c'est comme laisser un laboratoire avec des produits chimiques sur les tables — dangereux pour la prochaine personne et source potentielle de contamination des résultats futurs.

**Preuve :** Au redémarrage, assistant initial ou état "neuf" visible.

**Astuce :** Prenez une capture d'écran de l'écran d'accueil initial ou de l'assistant de configuration comme preuve de réinitialisation réussie.

![Wipe Data](wipedata.png)
![Résultat du wipe](wipeResuit.png)

---

## Étape 18 : Remise à zéro device labo (si utilisé)

- Paramètres système → Réinitialisation usine → Redémarrer
- Vérifier absence de comptes/profils/certificats
- **Preuve :** assistant de configuration initial

**Vérification supplémentaire :** Après réinitialisation, vérifiez l'absence de certificats racine personnalisés dans les paramètres de sécurité (qui pourraient permettre l'interception de trafic HTTPS).

**Option fastboot (labo uniquement) :**
```bash
fastboot erase userdata
# Puis redémarrer
```
Capturer l'écran de l'assistant initial.

> **Différence technique :** La réinitialisation via les paramètres efface les données utilisateur mais peut laisser des traces dans certaines partitions. La commande fastboot effectue un effacement de bas niveau plus complet.

---

## Étape 19 : Livrables (1–2 pages)

À inclure dans le rapport :

| Élément | Contenu |
|---------|---------|
| Définition rooting | 4 phrases |
| Schéma | Chaîne de confiance Verified Boot / AVB |
| Risques & mesures | 8 risques + 8 mesures défensives |
| MASVS | 2 exigences résumées |
| MASTG | 2 idées de tests |
| Fiche environnement | Remplie |
| Checklist reset | Signée + preuves (captures commandes/wipe) |

> Utilisez des tableaux, listes à puces et schémas simples pour rendre le rapport lisible. Un bon rapport de sécurité doit être compréhensible même par des non-spécialistes.

---

## Étape 20 : Checklist finale

**Début :**
- [ ] Périmètre écrit
- [ ] AVD neuf
- [ ] App test installée
- [ ] 3 scénarios notés
- [ ] Versions Android/app notées

**Fin :**
- [ ] Données de test supprimées
- [ ] Reset effectué (wipe AVD ou reset device)
- [ ] Preuve du reset
- [ ] Rapport + traçabilité sauvegardés
- [ ] Aucun compte personnel utilisé

> Cette checklist suit le principe **Plan-Do-Check-Act (PDCA)** : planification avant les tests, exécution contrôlée, vérification des résultats, et actions de nettoyage.
