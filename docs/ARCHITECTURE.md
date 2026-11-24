# Architecture

Deux programmes : `rtype_server`, qui fait tourner la partie, et `rtype_client`, un par joueur, qui l'affiche.
Ils ne se parlent qu'en UDP, avec le protocole binaire décrit dans [PROTOCOL.md](PROTOCOL.md).

```
 rtype_client (SFML)                         rtype_server
 ┌──────────────────────────┐   UDP 0x10 Move  ┌─────────────────────────────┐
 │ entrées → prédiction     │ ───────────────► │ NetworkRecv                 │
 │ registre ECS (sparse)    │   0x11 Canon     │ Settings · Heartbeat        │
 │ PacketHandler            │                  │ Logic → Physics → Collision │
 │ rendu, sons, animations  │ ◄─────────────── │ NetworkSend                 │
 └──────────────────────────┘  positions,      │ Cleanup                     │
                               tirs, impacts,  └─────────────────────────────┘
                               ennemis, score        World (ECS maison)
```

## Serveur

```
Server/
├── engine/          ECS : World, Registry, ComponentPool, EventBus, System, Types
├── games/           le jeu, découpé en modules de systèmes
│   ├── PlayerSystems/    réception des entrées, déplacement, envoi des positions et des impacts
│   ├── BulletSystems/    création, déplacement, collisions et nettoyage des tirs
│   ├── EnemySystems/     apparition, déplacement, tir, dégâts, mort, score
│   ├── AISystems/        IA humanoïde : décision, déplacement, ramassage, lancer, duplication
│   ├── WallSystems/      murs du haut et du bas
│   ├── ModuleSystems/    modules à ramasser (armes)
│   ├── MenuSystems/      salle d'attente, choix du mode et de la difficulté
│   ├── HeartbeatSystems/ latence et déconnexions
│   ├── SettingsSystems/  pause et reprise
│   ├── StatsSystems/     points de vie des joueurs et des ennemis
│   └── WaveManager.hpp   vagues d'ennemis
├── Network/         socket UDP (POSIX ou Winsock), protocole (SimpleProtocol.hpp)
└── main.cpp         enregistre les systèmes puis boucle sur world.update(dt)
```

- **Un seul fil d'exécution** : la boucle principale appelle `world.update(dt)` environ mille fois par seconde.
  La socket UDP est non bloquante ; les messages reçus sont lus pendant la phase `NetworkRecv`.
- **Phases** : chaque système déclare sa phase. Avant la partie, seules `MenuRecv`, `MenuLogic`, `MenuSend` et
  `Heartbeat` tournent. Pendant la partie : `NetworkRecv`, `Settings`, `Heartbeat`, puis, sauf en pause,
  `Logic`, `Physics`, `Collision` et `NetworkSend`, et enfin `Cleanup`.
- **Composants réseau** : `NetPlayer`, `NetBullet`, `LastNetworkPosition`, `NetworkPositionAccumulator`… lient une
  entité à son identifiant réseau et retiennent la dernière position envoyée pour les paquets delta.
- **Identifiants** : compteurs globaux pour les tirs (2 octets) et les ennemis (1 octet), partagés par tous les
  clients.

## Client

```
Client/
├── GameClient.cpp        fenêtre, boucle, états (menu, difficulté, choix du vaisseau, partie, game over)
├── Programs/
│   ├── Connection/       socket UDP, aiguillage des paquets reçus
│   ├── Network/PacketHandler/   un gestionnaire par type de paquet → composants
│   ├── Registry/, SparseArray/  registre ECS à tableaux creux, itération par « zipper »
│   ├── Systems/          rendu, animations, déplacement, nettoyage, boss, ennemis fixes, IA
│   ├── Shooting/         tir chargé (temps d'appui → niveau 1 à 6), prédiction des tirs
│   ├── EntityFactory/    construction des entités (sprites, hitbox, dégâts)
│   ├── Setting/          réglages, remappage des touches, manette, filtres daltoniens
│   └── SoundManager/, TextureManager/
└── Assets/               sprites, polices, sons
```

## Prédiction et réconciliation

1. Le joueur appuie sur une flèche : le client déplace tout de suite son vaisseau et envoie `0x10 Move`. Le serveur
   applique le même déplacement et renvoie la position officielle (`0x30`), sur laquelle le client se recale.
2. Le joueur relâche `Espace` : le client crée un tir **prédit**, marqué d'un horodatage sur 8 octets, et envoie
   `0x11 Canon` avec ce même horodatage.
3. Le serveur crée le vrai tir et diffuse `0x31 PlayerBulletCreate` avec son identifiant et l'horodatage. Le client
   retrouve son tir prédit grâce à l'horodatage et lui donne l'identifiant réseau ; s'il refuse le tir, le
   serveur envoie `0x45 BulletCreationRejected` et le client le supprime.
4. Ensuite, `0x41 PlayerBulletPos` met à jour la position et le client lisse l'écart. Un tir qui ne reçoit plus
   de mise à jour pendant 0,5 s a été détruit côté serveur : le client le retire aussi.

## Vagues et score

`WaveManager` fait apparaître les ennemis vague par vague ; chaque ennemi a une valeur (10, 25 ou 100 points pour
le boss). À chaque ennemi détruit, le serveur ajoute ses points et envoie `0x54 ScoreUpdate` (score et vague) à
tous les clients.
