# LAB 16 : Inspection HTTPS Android : Désactivation du SSL Pinning avec Objection + Proxy (Burp/mitmproxy)
## Objectifs du lab

- Installer et utiliser **Objection** (surcouche Frida) pour désactiver l’SSL pinning d’une app Android.
- Mettre en place un proxy (Burp/mitmproxy) et installer le certificat CA sur l’appareil.
- Lancer l’app avec Objection et exécuter `android sslpinning disable` efficacement (spawn ou attach).
- Valider la capture du trafic HTTPS en clair et dépanner les cas courants.

---

## Prérequis

- **Python 3.8+** et **pip**.
- **ADB** (Android Platform Tools) : [Télécharger](https://developer.android.com/tools/releases/platform-tools)
- **Frida** côté PC et **frida-server** côté Android, versions alignées.
- Un appareil **Android 8+** avec **Débogage USB** activé.
- **Burp Suite** (ou mitmproxy) sur le PC.

### Vérifications rapides

```bash
python --version
pip --version
adb version
```

---

## Étape 1 — Installer Objection et Frida côté PC

**Option isolée (recommandée) :**

```bash
pip install --user pipx
pipx ensurepath
pipx install objection
```

**Ou via pip classique :**

```bash
pip install --upgrade objection frida frida-tools
```

**Vérifiez :**

```bash
objection --version
frida --version
python -c "import frida; print(frida.__version__)"
```

> **Remarque Windows** : si `objection` n’est pas reconnu, ajoutez le dossier `Scripts` de Python au PATH (ex: `%USERPROFILE%\AppData\Roaming\Python\Python311\Scripts`).

---

## Étape 2 — Préparer l’appareil et démarrer frida-server

1. **Activer le Débogage USB** sur l’appareil (Options développeur), brancher en USB, accepter l’empreinte :

```bash
adb devices
```

2. **Identifier l’architecture CPU** (pour télécharger le bon binaire) :

```bash
adb shell getprop ro.product.cpu.abi
```

3. **Télécharger** sur [https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)  
   `frida-server-<même_version>-android-<arch>.xz` et le décompresser (7‑Zip sous Windows).

4. **Pousser et lancer frida-server** :

```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

5. **Option (selon appareil) :**

```bash
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

6. **Vérifier que l’appareil est visible par Frida :**

```bash
frida-ps -Uai
```

> **Astuce versions** : `frida --version` (PC) doit correspondre exactement à la version du frida-server téléchargé.

---

## Étape 3 — Configurer le proxy et installer la CA

1. Lancer **Burp** (ou mitmproxy) sur le PC et noter l’adresse/port (ex: `192.168.X.Y:8080`).
2. Sur le téléphone, configurer le **proxy Wi‑Fi** du réseau vers l’adresse/port ci‑dessus.
3. **Télécharger et installer la CA** du proxy sur le téléphone :

   - **Burp** : visiter `http://burp` depuis le téléphone pour récupérer le certificat.
   - **mitmproxy** : visiter `http://mitm.it` → Android → installer.

---

## Étape 4 — Lancer l’app avec Objection et désactiver le pinning

1. **Identifiez le package de l’app cible :**

```bash
frida-ps -Uai | Select-String -Pattern https,ssl,okhttp   # PowerShell
```

2. **Deux stratégies** selon le timing des vérifications :

### Injection au démarrage (spawn) — recommandé

```bash
objection -g com.example.app explore --startup-command "android sslpinning disable"
```

### Attache à une app déjà ouverte (attach)

```bash
# Ouvrez l’app normalement sur le téléphone, puis :
objection -g com.example.app explore
# Une fois dans la console Objection :
android sslpinning disable
```

### Résultat attendu

- Dans la console Objection, messages confirmant l’installation des hooks.
- L’application ne présente plus d’erreurs de certificat.
- Le proxy affiche les requêtes HTTPS (URL, en‑têtes, parfois corps).

### Commandes utiles dans la console Objection

```text
help android sslpinning
android hooking search classes pin
```

---

## Étape 5 — Validation

- Utilisez des écrans internes de l’app (login, requêtes API) pour générer du trafic.
- Dans Burp/mitmproxy, vérifiez que :
  - Les requêtes HTTPS de l’app apparaissent.
  - Les réponses sont visibles sans alerte SSL côté app.
- Dans Objection, surveillez la console pour voir si des hooks sont déclenchés.
