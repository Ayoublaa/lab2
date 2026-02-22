# Lab 2 : Rooting & Sécurité Android

> Comprendre le rooting et tester les vulnérabilités OWASP sur une app Android

## Fiche périmètre

| Champ | Valeur |
|-------|--------|
| **App + version** | [Nom] + [Version] |
| **Support** | AVD / Device labo |
| **Objectif** | Rooting et impacts sur les contrôles d'intégrité |
| **Données** | Fictives |
| **Réseau** | Test |

## Rooting AVD

```bash
adb devices
adb root
adb remount
adb shell id   # → uid=0(root)
```

![ADB Device](adbDevice.png)
![Root - uid](rootetuid.png)

## Scénarios de test

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

## Concepts clés

- **Root** : Privilèges super-utilisateur — modifie les protections système
- **Sécurité Android** : Sandboxing, permissions, intégrité système
- **Verified Boot** : Garantit que le système démarre sans modification (chain of trust)

## Risques & Mesures

| Risques | Mesures |
|---------|---------|
| Intégrité non garantie | Réseau isolé, données fictives |
| Données exposées | AVD dédié, wipe en fin de séance |
| Traçabilité insuffisante | Journal + captures d'écran |

## OWASP MASVS

- **STORAGE-1** : Données sensibles stockées de façon sécurisée
- **NETWORK-1** : TLS correctement configuré

## Remise à zéro

```bash
adb emu avd stop
adb emu avd wipe-data
```

Ou : Android Studio → Device Manager → Wipe Data
