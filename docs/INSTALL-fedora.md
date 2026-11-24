# Installation sur Fedora

Testé sur Fedora 39 avec GCC 13, CMake 3.30 et SFML 2.5.1. Le dépôt prévoit Conan et SFML 2.6, mais la SFML des
dépôts Fedora suffit : pas besoin de Conan si l'on compile le serveur et le client séparément.

## 1. Dépendances

```bash
sudo dnf install gcc-c++ cmake make SFML-devel
```

## 2. Récupérer le code

```bash
git clone git@github.com:FuniclemDev/rtype-source.git rtype
cd rtype
```

## 3. Compiler

Le `CMakeLists.txt` racine exige GoogleTest et le fichier de configuration généré par Conan. On compile donc
chaque programme à partir de son propre dossier, dans un dossier de build à part :

```bash
cmake -S Server -B build/server -DCMAKE_BUILD_TYPE=Release
cmake --build build/server -j"$(nproc)"

cmake -S Client -B build/client -DCMAKE_BUILD_TYPE=Release
cmake --build build/client -j"$(nproc)"
```

Avec Conan et GoogleTest installés, la méthode du README source (`conan install` puis `cmake` à la racine)
compile aussi les tests.

## 4. Lancer

Les deux programmes chargent leurs fichiers (`Client/Assets/…`) par des chemins relatifs : lancez-les **depuis la
racine du dépôt**.

```bash
# terminal 1 : le serveur, sur le port de votre choix (1024 à 65535)
./build/server/rtype_server 4242

# terminal 2 : un client par joueur
./build/client/rtype_client 127.0.0.1 4242
```

Dans le menu : **Solo**, puis la difficulté, puis la couleur du vaisseau. Pour jouer à plusieurs, chaque joueur
lance un client vers l'adresse IP du serveur et choisit **Multiplayer**.

## Commandes

| Touche | Action |
| --- | --- |
| Flèches | se déplacer |
| `Espace` | tirer ; maintenir pour charger (le tir part au relâchement) |
| Icône réglages (en haut à droite) | pause et réglages (touches, daltonisme, son) |
| Manette | prise en charge par le module joystick de SFML |

## Tester en mauvaises conditions réseau

```bash
sudo ./simulate_lag.sh start --port 4242 --delay 80ms --jitter 20ms --loss 1%
sudo ./simulate_lag.sh stop
```

Le script utilise `tc` et `netem` (paquet `iproute-tc`) sur l'interface locale.

## Problèmes connus

- Le client affiche `Failed to open sound file "Client/Assets/Sounds/shoot.wav"` : ce son manque dans le dépôt,
  le jeu fonctionne sans.
- Sans argument, le client vise `127.0.0.1:8080` et le serveur écoute sur `8081` : passez toujours le même port
  aux deux.
