# LAB18
# 🔥 CTF Writeup — FireStorm (PwnSec)



---

## 🎯 Objectif

L'application Android contient une méthode `Password()` qui génère un mot de passe pour se connecter à une base de données Firebase. Cette méthode n'est **jamais appelée** dans le flux normal de l'application.

L'objectif est de :
1. Analyser l'APK avec **Jadx** (reverse statique)
2. Forcer l'exécution de `Password()` avec **Frida** (hooking dynamique)
3. S'authentifier sur **Firebase** avec le mot de passe obtenu
4. Récupérer le **flag** depuis la Realtime Database

---

## 🛠️ Outils utilisés

| Outil | Usage |
|-------|-------|
| Jadx-GUI | Décompilation de l'APK |
| Frida 17.9.1 | Hooking dynamique Java |
| ADB | Communication avec l'émulateur Android |
| Python + pyrebase4 | Authentification Firebase |
| Android Studio Emulator | Environnement d'exécution |

---

## 📋 Étape 1 — Préparation de l'environnement

Vérification de tous les outils et de la connexion à l'émulateur :

```powershell
adb version
adb devices
adb shell getprop ro.product.cpu.abi
adb shell pm list packages | Select-String "pwnsec"
frida-ps -U
```

![Environnement](prem.png)

**Résultats obtenus :**
- Émulateur `emulator-5554` connecté ✅
- Architecture `x86` ✅
- Package `com.pwnsec.firestorm` installé ✅
- Frida-server actif, `Firestorm` visible dans les processus ✅

---

## 📋 Étape 2 — Analyse statique avec Jadx

Ouverture de l'APK dans **Jadx-GUI** pour décompiler le code Java.

**Éléments importants trouvés :**

Dans `MainActivity`, la méthode `Password()` est définie mais **jamais appelée** :
- Elle combine des chaînes statiques depuis `strings.xml`
- Elle appelle `generateRandomString()` — une fonction native dans `libfirestorm.so`

Dans `strings.xml` on extrait :
- **Email Firebase :** `TK757567@pwnsec.xyz`
- **API Key :** `AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY`
- **Database URL :** `https://firestorm-9d3db-default-rtdb.firebaseio.com`

---

## 📋 Étape 3 — Script Frida

Le script `frida_firestorm.js` utilise `Java.choose` pour trouver l'instance de `MainActivity` en mémoire et appeler manuellement `Password()` :

![Script Frida](script.png)

```javascript
Java.perform(function() {

    function getPassword() {
        console.log("[*] Recherche d'instances de MainActivity...");

        Java.choose('com.pwnsec.firestorm.MainActivity', {
            onMatch: function(instance) {
                console.log("[+] Instance trouvée : " + instance);
                try {
                    var pass = instance.Password();
                    console.log("[+] Mot de passe : " + pass);
                } catch (e) {
                    console.log("[-] Erreur : " + e);
                }
            },
            onComplete: function() {
                console.log("[*] Recherche terminée.");
            }
        });
    }

    setTimeout(getPassword, 3000);
});
```

**Lancement du script :**

```powershell
adb shell am force-stop com.pwnsec.firestorm
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js
```

![Mot de passe obtenu via Frida](deux.png)

**Mot de passe obtenu :** `C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC`

> ⚠️ Ce mot de passe change à chaque lancement car il contient une partie générée dynamiquement par la fonction native `generateRandomString()`.

---

## 📋 Étape 4 — Script Python pour récupérer le flag

Le script `get_flag.py` s'authentifie sur Firebase avec les credentials obtenus, puis lit la Realtime Database :

![Script Python](python.png)

```python
import pyrebase

config = {
    "apiKey": "AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY",
    "authDomain": "firestorm-9d3db.firebaseapp.com",
    "databaseURL": "https://firestorm-9d3db-default-rtdb.firebaseio.com",
    "storageBucket": "firestorm-9d3db.appspot.com",
    "projectId": "firestorm-9d3db"
}

firebase = pyrebase.initialize_app(config)
auth = firebase.auth()

email = "TK757567@pwnsec.xyz"
password = "C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC"

user = auth.sign_in_with_email_and_password(email, password)
print("Connexion reussie. Token obtenu.")

db = firebase.database()
flag_data = db.get(user['idToken'])
print("FLAG recupere :")
print(flag_data.val())
```

**Installation de pyrebase :**
```powershell
pip install pyrebase4
```

**Exécution :**
```powershell
python get_flag.py
```

![Flag obtenu](flag.png)

---

## 🏆 Flag

```
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
```

---

## 🔄 Résumé du flux d'attaque

```
APK (FireStorm.apk)
    │
    ├── Jadx ──→ strings.xml ──→ email + config Firebase
    │
    ├── Jadx ──→ MainActivity.Password() ──→ méthode cachée identifiée
    │
    ├── Frida ──→ Java.choose ──→ instance.Password() forcée
    │                          └──→ mot de passe dynamique obtenu
    │
    └── Python + pyrebase ──→ Firebase Auth ──→ Realtime Database ──→ FLAG 🏆
```

---

## 💡 Points clés appris

- **Reverse Android avec Jadx** — décompiler un APK et identifier des méthodes cachées
- **Hooking dynamique avec Frida** — forcer l'exécution de code mort non appelé normalement
- **`Java.choose`** — parcourir la mémoire pour trouver des instances de classes Android
- **Firebase Auth** — s'authentifier programmatiquement pour accéder à une base de données protégée
- Le mot de passe est **dynamique** — il faut lancer Frida et Python dans la foulée
