# LAB-13-Bypass-de-la-D-tection-de-Root-Android-avec-Objections

Prérequis
PC Windows/macOS/Linux avec droits admin/sudo.
Python 3.8+ et pip.
ADB (Android Platform Tools): https://developer.android.com/tools/releases/platform-tools
Un appareil Android 8.0+ avec Options développeur + Débogage USB activés.
Frida côté PC et frida-server côté Android (versions alignées). Si besoin, référez-vous à votre lab d’installation Frida précédent.
Un APK cible qui fait une root detection (ex.: RootBeer Sample, DevAdvance RootChecker, ou votre app). Exemples de package ci‑dessous: com.example.rootcheck.


Vérifications rapides:
python --version
pip --version
adb devices
frida --version

![Import OVA](https://github.com/user-attachments/assets/e5e1cfa7-45d1-48aa-9434-671dcf8893df)

---

Étape 1 — Installer Objection
Objection est une CLI Python.
Installation (choisissez une méthode):
# Méthode recommandée via pipx (isolation)
pip install --user pipx
pipx ensurepath
pipx install objection

# Ou via pip classique
pip install --upgrade objection
Vérifications:
objection --version
objection --help
Conseil Windows: exécutez dans PowerShell. Assurez-vous que le dossier Scripts de Python (p. ex. %USERPROFILE%\AppData\Roaming\Python\Python311\Scripts) est dans le PATH si la commande n’est pas trouvée.

![Import OVA](https://github.com/user-attachments/assets/18e5267f-bee1-4973-932c-0948e4c2da90)


---

Étape 2 — Préparer l’appareil et démarrer frida-server
Identifier l’ABI pour télécharger le bon frida-server (si ce n’est pas déjà fait):
adb shell getprop ro.product.cpu.abi
Télécharger frida-server-<version>-android-<arch>.xz depuis https://github.com/frida/frida/releases et le décompresser.

Pousser et lancer:
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"

Option (selon appareil):
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
Vérifier la visibilité:
frida-ps -Uai

![Import OVA](https://github.com/user-attachments/assets/64e6224e-123e-4951-9772-864fd2317dc9)

---


Étape 3 — Démarrer Objection sur l’app cible
Deux stratégies: démarrer l’app sous Objection (spawn) ou s’attacher à une app déjà ouverte (attach).
Spawn dès le lancement (recommandé pour appliquer les hooks tôt):
objection -g com.example.rootcheck explore --startup-command "android root disable"
Attach à une app déjà en cours (utile si l’app n’aime pas être hookée trop tôt):
# Lancez l’app normalement sur le téléphone, puis
objection -g com.example.rootcheck explore
# Dans la console Objection qui s’ouvre, exécutez:
android root disable
Remarques:
Selon la version d’Objection, les commandes exactes peuvent varier. Tapez help android root dans la console Objection pour voir la liste et l’intitulé précis des commandes disponibles sur votre version. Les plus courantes: android root disable et parfois des variantes ciblées pour bibliothèques de détection populaires.
Si votre version propose l’option SSL pinning, elle est hors scope ici mais sachez qu’elle existe (android sslpinning disable).


---

Étape 4 — Que fait android root disable ? (Compréhension rapide)
Derrière, Objection installe des hooks Java via Frida, qui typiquement:
Forcent android.os.Build.TAGS à renvoyer une valeur inoffensive (ex.: release-keys).
Font échouer des recherches de fichiers sensibles (java.io.File.exists() pour /system/xbin/su, busybox, etc.).
Neutralisent des exécutions de commandes su / which su via Runtime.getRuntime().exec(...).
Désactivent des méthodes de libs comme RootBeer (ex.: isRooted()), quand Objection contient des patches pour celles‑ci.
Cela couvre beaucoup de root checks « Java side ». Des checks purement natifs C/C++ peuvent nécessiter des actions supplémentaires (voir Étape 7).


---

Étape 5 — Valider le bypass
Avant Objection: ouvrez l’app sans instrumentation et observez « Root detected » / blocage.
Avec Objection + android root disable:
L’app devrait afficher « Not rooted » ou ne plus bloquer.
Dans la console Objection, vous devriez voir des logs indiquant que les hooks ont été installés et parfois appelés.
Commandes utiles dans la console Objection pour explorer:
android hooking search classes root
android hooking search methods isRoot
android intent launch_activity <ActivityName>



