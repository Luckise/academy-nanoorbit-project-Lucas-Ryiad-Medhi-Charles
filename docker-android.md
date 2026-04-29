# Émulateur Android dans Docker

> Service `android-emulator` ajouté à `docker-compose.yml`.
> Image utilisée : `budtmo/docker-android:emulator_11.0` (Samsung Galaxy S10).

## Architecture

Trois conteneurs sur un réseau Docker partagé `nanoorbit-net` :

```
┌──────────────────┐    ┌──────────────────┐    ┌────────────────────┐
│  oracle-db       │───→│  api             │←───│  android-emulator  │
│  (Oracle 23ai)   │    │  (Flask 5000)    │    │  (noVNC 6080)      │
└──────────────────┘    └──────────────────┘    └────────────────────┘
        │                       │                         │
        └───────────────────────┴─────────────────────────┘
                       nanoorbit-net (bridge)
```

L'émulateur appelle l'API par son **nom de conteneur** (`http://nanoorbit_api:5000/`), pas par `10.0.2.2:5000` qui ne fonctionne qu'avec l'émulateur Android Studio sur la machine hôte.

## Pré-requis

### Linux
KVM disponible (`/dev/kvm`). Vérification :
```bash
sudo apt install cpu-checker
kvm-ok
# Doit afficher : "KVM acceleration can be used"
```

### Windows 11 + WSL2
Activer la **virtualisation imbriquée** :

1. PowerShell → ouvrir `notepad $env:USERPROFILE\.wslconfig` :
   ```ini
   [wsl2]
   nestedVirtualization=true
   ```

2. Dans WSL2 Ubuntu, éditer `/etc/wsl.conf` :
   ```ini
   [boot]
   command = /bin/bash -c 'chown -v root:kvm /dev/kvm && chmod 660 /dev/kvm'
   ```

3. Ajouter ton user au groupe kvm :
   ```bash
   sudo usermod -a -G kvm $USER
   ```

4. Redémarrer WSL2 depuis PowerShell :
   ```powershell
   wsl --shutdown
   ```

## Démarrage

```bash
docker compose up -d
```

Trois conteneurs démarrent :
- `vls_oracle_23ai` (~30-60 s à initialiser au premier démarrage)
- `nanoorbit_api` (attend Oracle prêt)
- `nanoorbit_android` (~30 s pour booter l'émulateur)

## Accès à l'émulateur

Ouvrir dans le navigateur :
```
http://localhost:6080
```

L'écran de l'émulateur s'affiche en noVNC. Interaction souris/clavier directe.

## Installation de l'app NanoOrbit

```bash
# 1. Build l'APK depuis le projet Android
cd altn82-android/starter
./gradlew assembleDebug

# 2. Copier l'APK dans le conteneur
docker cp app/build/outputs/apk/debug/app-debug.apk nanoorbit_android:/tmp/

# 3. Installer via ADB dans le conteneur
docker exec nanoorbit_android adb install /tmp/app-debug.apk
```

L'app apparaît dans le drawer de l'émulateur. Au lancement, elle appelle `http://nanoorbit_api:5000/` qui résout vers le conteneur API grâce au DNS interne Docker.

## Vérification de la connectivité

Tester depuis l'émulateur que l'API répond :
```bash
docker exec nanoorbit_android adb shell curl -s http://nanoorbit_api:5000/health
# Doit renvoyer : {"status":"ok","database":"connected"}
```

## Connexion ADB depuis l'hôte

```bash
adb connect localhost:5555
adb devices
# nanoorbit_android:5555  device
```

Permet d'utiliser l'émulateur comme un device standard depuis Android Studio.

## Limitations connues

| Limitation | Conséquence |
|---|---|
| Pas de capteur `TYPE_ROTATION_VECTOR` | **L'écran AR Sky-Track ne fonctionne pas** dans le conteneur |
| Pas de caméra physique | Idem AR (pas de flux caméra) |
| Pas de GPS réel | `osmdroid` géolocalisation requiert `adb emu geo fix lon lat` |
| Pas d'accélération GPU | Compose lourd (animations) un peu saccadé |
| Image Linux/x86_64 | Ne tourne pas sur Mac Apple Silicon sans contournement |

Pour tester l'**AR**, utiliser un téléphone physique. Pour tester **Dashboard / Detail / Planning / Map**, le conteneur suffit.

## Arrêt et nettoyage

```bash
# Arrêt propre
docker compose down

# Reset complet (efface les données Oracle + état émulateur)
docker compose down -v
```

## Dépannage

**`Error response from daemon: error gathering device information ... no such file or directory`**
→ KVM pas accessible. Vérifier `kvm-ok` et le setup nested virtualization.

**Émulateur reste sur "Starting..."**
→ Première initialisation peut prendre 1-2 minutes. Vérifier les logs : `docker logs -f nanoorbit_android`.

**App lance mais "Mode hors-ligne" affiché en permanence**
→ L'API n'est pas joignable. Vérifier que `API_BASE_URL=http://nanoorbit_api:5000/` est bien dans `.env` et que l'APK a été rebuild après modification du `.env`.

**`docker exec ... adb install` échoue**
→ L'émulateur n'a pas fini de booter. Attendre `adb shell getprop sys.boot_completed` = `1`.
