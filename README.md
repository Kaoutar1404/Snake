# 🐍 CTF PwnSec — Snake (Hard) — Android Reverse Engineering Writeup

> **Auteur :** Ayoub Laafar  
> **Difficulté :** ⭐⭐⭐ Hard  
> **Catégorie :** Android Reverse Engineering  
> **Flag :** `PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}`

---

## 🎯 Objectif du Challenge

Analyser et bypasser les protections d'une application Android, puis exploiter une vulnérabilité **CVE-2022-1471 (SnakeYAML)** pour déclencher l'exécution d'une fonction native JNI qui génère le flag.

---

## 🔧 Environnement

- Émulateur : **Pixel 3 API 28 x86** (Google APIs)
- Outils : `apktool 3.0.1`, `Jadx-GUI`, `ADB`, `VS Code`, `PowerShell`

---

## Étape 1 : Analyse statique avec Jadx

On ouvre l'APK avec **Jadx-GUI** pour analyser le code source.

![Ouverture du fichier snake.apk dans Jadx](jadx.png)

On identifie dans `MainActivity` :
- La méthode `isDeviceRooted()` qui vérifie si l'appareil est rooté
- La méthode `C()` qui contient le flux principal : si l'Intent contient `SNAKE=BigBoss`, l'app lit `/sdcard/Snake/Skull_Face.yml` et le parse avec SnakeYAML

![Analyse de MainActivity - méthodes de détection root](main.png)

![Flux principal : SNAKE=BigBoss → lecture Skull_Face.yml → SnakeYAML](C.png)

---

## Étape 2 : Décompilation avec apktool

On décompile l'APK pour accéder au code Smali :

```powershell
java -jar apktool_3.0.1.jar d snake.apk -o snake_smali_clean
```

![Structure du dossier snake_smali après décompilation](dossier.png)

---

## Étape 3 : Patch Smali — Bypass isDeviceRooted()

On remplace le corps de `isDeviceRooted()` pour qu'elle retourne toujours `false` :

```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .registers 1
    const/4 v0, 0x0
    return v0
.end method
```

![Patch de isDeviceRooted dans VS Code](koutar.png)

---

## Étape 4 : Patch Smali — Correction du chemin fichier YAML

On corrige le chemin de lecture du fichier YAML vers `/sdcard/Snake/Skull_Face.yml` :

![Patch du chemin "Snake" dans le Smali](changecode.png)

---

## Étape 5 : Patch AndroidManifest.xml

On active l'extraction des librairies natives :

```xml
android:extractNativeLibs="true"
```

![Patch extractNativeLibs dans AndroidManifest](libsnative.png)

---

## Étape 6 : Recompilation et signature

```powershell
# Recompile
java -jar apktool_3.0.1.jar b snake_smali_clean -o snake_patched.apk

# Zipalign + Signature
zipalign -f -v 4 snake_patched.apk snake_final.apk
apksigner sign --ks my-key.jks --out snake_signed.apk snake_final.apk
```

![Recompilation avec apktool et zipalign](javajar.png)

![Vérification successful](verificatiobseccesful.png)

---

## Étape 7 : Installation et création du payload YAML

```powershell
adb install snake_signed.apk
```

![Installation de l'APK patché](install.png)

On crée le fichier `Skull_Face.yml` qui exploite **CVE-2022-1471** (désérialisation non sécurisée SnakeYAML) :

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```

![Contenu du payload Skull_Face.yml](snake.png)

```powershell
adb shell mkdir -p /sdcard/Snake
adb push Skull_Face.yml /sdcard/Snake/Skull_Face.yml
adb shell pm grant com.pwnsec.snake android.permission.READ_EXTERNAL_STORAGE
```

---

## Étape 8 : Lancement et récupération du flag

```powershell
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
adb logcat -d | Select-String -Pattern "PWNSEC|BigBoss"
```

![Flag obtenu dans logcat](Flag.png)

---

## 🏁 Flag

```
PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}
```

---

## 🔑 Concepts clés

- **CVE-2022-1471** : Désérialisation arbitraire via SnakeYAML `new Yaml().load()` qui instancie n'importe quelle classe Java
- **Smali patching** : Modification du bytecode Dalvik pour bypasser les vérifications de sécurité
- **APK repackaging** : apktool → patch → zipalign → apksigner
