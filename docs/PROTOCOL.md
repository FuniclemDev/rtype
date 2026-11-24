# Protocole réseau

Binaire, sur UDP. Chaque paquet commence par **un octet d'identifiant**, suivi de champs de taille fixe en
big-endian. Les identifiants d'entité tiennent sur 1 octet, ceux des tirs sur 2 octets. La spécification complète,
au format RFC, est dans le dépôt source (`RFC 9901 - R'Type.txt`, document interne au projet).

## Client → serveur

| ID | Nom | Contenu | Rôle |
| --- | --- | --- | --- |
| `0x01` | Connect | version (1) | connexion |
| `0x02` | Disconnect | — | départ propre |
| `0x03` | Ready | statut | prêt à jouer |
| `0x10` | Move | direction (4 bits) | déplacement (8 directions) |
| `0x11` | Canon | niveau (1) + horodatage (8) | tir, chargé de 1 à 6 |
| `0x13` | Heartbeat | horodatage (4) | maintien de la connexion, mesure de la latence |
| `0x14` | MenuRequestGamemode | mode (1) | solo, multijoueur, spectateur |
| `0x15` | GameSettings | statut (1) | pause, reprise, retour au menu |

## Serveur → client

### Session et menus

| ID | Nom | Contenu |
| --- | --- | --- |
| `0x20` | ConnectAck | joueur (1), accepté |
| `0x21` | DisconnectAck | joueur (1) |
| `0x24` | PlayerList | nombre, puis [joueur, x, y] |
| `0x27` | GameSettingsAck | statut (1) |
| `0x50` – `0x53` | Menu… | mode choisi, joueurs connectés, noms, tous prêts |
| `0x54` | ScoreUpdate | score (4) + vague (2) |

### Temps réel

| ID | Nom | Contenu |
| --- | --- | --- |
| `0x30` | PlayerPos | joueur (1), x (2), y (2) |
| `0x31` | PlayerBulletCreate | tir (2), joueur (1), x, y, direction, niveau, horodatage (8) |
| `0x32` | HitBullet | tir du joueur (2), missile ennemi (2) : les deux se détruisent |
| `0x33` | Hit | joueur (1), dégâts (1), missile (2) |
| `0x34` | HeartbeatReply | joueur (1), horodatage (4) |
| `0x35` | EnemyMissile | ennemi (1), x, y, angle (float 4), missile (2), type (1) |
| `0x38` | EnemyCanonBoss | ennemi (1), x, y, direction, missile (2) |
| `0x39` | HitEnemy | ennemi (1), dégâts (1), tir (2) |
| `0x3A` | EnemyPosition | ennemi (1), x, y, direction |
| `0x3B` | EnemyCreation | ennemi (1), type (1), x, y, direction, PV (1) |
| `0x3C` | EnemyDespawn | ennemi (1) |
| `0x3D` | PlayerEnemyHP | PV de tous les joueurs et ennemis en un paquet |
| `0x3E` / `0x3F` | ModuleCreation / ModulePos | modules à ramasser |
| `0x41` | PlayerBulletPos | tir (2), x (2), y (2), direction |
| `0x44` | EnemyMissilePos | missile (2), x (2), y (2), variation d'angle (4) |
| `0x45` | BulletCreationRejected | horodatage (8) du tir prédit refusé |
| `0x47` – `0x4E` | AI… | état de l'IA, ramassage, saisie, lancer, duplication, projectiles, objets |

## Choix de conception

- **UDP partout** : pas d'attente de retransmission ; un paquet de position perdu est remplacé par le suivant.
  Les messages importants (création, destruction) sont doublés d'un filet de sécurité côté client : un tir ou un
  missile qui ne reçoit plus de mise à jour est retiré au bout d'une demi-seconde (tir) ou d'une seconde (missile).
- **Paquets courts** : les positions tiennent sur 2 octets signés, les directions sur quelques bits ; un paquet de
  position de tir fait 8 octets.
- **Horodatage des tirs** : il relie le tir prédit par le client au tir officiel créé par le serveur.
- **Validation** : le serveur vérifie la taille attendue de chaque paquet reçu avant de le traiter.
