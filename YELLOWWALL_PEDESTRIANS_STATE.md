# Piétons de la capitale Yellow Wall : état mesuré, cause des bugs, plan

> Rapport du 2026-11-19, étape 1 (diagnostic) + étape 2 (plan). **Aucune modification du projet n'a été faite.**
> Méthode : lecture hors moteur des `.umap`/`.uasset` (outil `Saved/Claude/uasset_offline/`), lecture du code moteur
> 5.7.3 (`C:/UE5_Share/Engine/Plugins`), simulation du graphe de trottoirs. L'éditeur n'a pas été lancé (un éditeur
> tournait déjà, verrou libre mais machine occupée) : ce qui demande le moteur est listé au §5.
> Repère : X = est, **Y négatif = nord**, origine = centre de la capitale, distances en mètres.

---

## 1. La chaîne complète (mesurée)

### 1.1 Plugins et code
| Élément | État |
|---|---|
| `MassGameplay`, `MassAI`, `MassCrowd`, `StateTree`, `GameplayStateTree`, `SmartObjects`, `GameplayBehaviorSmartObjects` | activés dans `QANGA.uproject` (plugins moteur, compilés avec le moteur 5.7.3) |
| `Plugins/Traffic` (MassTraffic, échantillon City Sample d'Epic) | **désactivé** (`"Enabled": false`), aucun binaire. Le contenu `Content/Mass/Traffic` en dépend : mort en l'état |
| C++ QANGA | **aucun** : la foule est 100 % Blueprint + DataAssets sur Mass d'Epic. Seul `CLIScape/Tool_ManageZoneGraph` (outil) touche à ZoneGraph |
| QAI | système séparé (registre SoA, pas Mass). Aucun lien avec la foule : pas de concurrence de ressources sauf le CPU |
| QLevel | ne gère pas la foule (la capitale n'est pas en QLevel, cf. `CAPITALE_YELLOWWALL_ARCHITECTURE.md`) |

**Conclusion « C++ à recompiler »** : il n'y a pas de C++ maison dans la chaîne. Ce qui a bougé avec 5.7, ce sont les
réglages et assets (voir §4), pas un module à rebuild.

### 1.2 Niveaux chargés en jeu
| Level | Chargé par | Contenu |
|---|---|---|
| `Maps/CompositeUniverse/Mass_ZoneGraph_SubLevel/L_ZoneGraph_Merging` (117 Mo, modifié le 2026-08-24) | `L_Persistent_Universe`, `LevelStreamingAlwaysLoaded` (serveur et clients) | 2 368 `ZoneShape` + **3 `ZoneGraphData`** |
| `.../L_Mass_Capital` | `L_Steaming_On_Earth`, sphère `SL_YellowWallEffects` 64 km | 3 spawners `Mass_Spawner_Crowd` |
| `L_Capital_ZoneGraph_Crowd` (CompositeUniverse et LevelDev), `L_ZoneGraph_Complex`, `CryoRoom`, `Keep/L_ZoneGraph_Elevator` | aucun persistant de jeu trouvé (`refs_by_persistent.json`) | sources de travail des shapes |

### 1.3 Le graphe de trottoirs (stockage `ZoneGraphData` de `L_ZoneGraph_Merging`)
- 2 398 zones, **12 855 voies, 272,6 km**. Toutes taguées `Crowd_Sidewalk` (bit 2).
- Capitale (`Crowd_Capital`, bit 9) : 11 073 voies, 250 km. Complex (bit 8) : 1 726 voies, 21,6 km (réseau à part).
  Intérieur (bit 7, CryoRoom) : 44 voies. `Crowd_CrossRoad` (bit 5) : 12 voies. Tags de densité (bits 29 à 31) : **0 voie**.
- Profils de voies dans `Config/DefaultPlugins.ini` (`ZoneGraphSettings`), dont deux profils **à sens unique**
  `Crowd_Sidewalk_1-50_F` / `_B`.

### 1.4 Qui spawn, où, combien
- `Mass_Spawner_Crowd` (BP) hérite de `Content/Mass/Sys/Mass_Spawner_Class` (BP sur `AMassSpawner`) :
  `bAutoSpawnOnBeginPlay = false`, le BP spawne lui-même **derrière `IsServer` / `IsDedicatedServer`** après un `Delay`,
  avec `Count = BaseSpawnCount x échelle` (paramètre global `EssentialMacros` + `ScaleSpawningCount`). Événements
  `ServerRequestSpawn / DeSpawn / Reload / ClearTemplate` : aucun appelant trouvé dans `Systems`, `GameMode`, `UI_Sys`,
  `Widget`, `Pawn`, `Characters`, `_QData`, `Data` (recherche bornée, pas une preuve d'absence).
- Générateur : `MassEntityZoneGraphSpawnPointsGenerator` (points tous les 1 à 3 m sur les voies filtrées, tirage au hasard).

| Spawner | Position | BaseSpawnCount (Count enregistré) | Filtre de voies | Types (proportions) |
|---|---|---|---|---|
| Capital | (239, 286, 116) | 220 (150) | `Crowd_Capital` sauf Inside/Complex | CyborgV1 0,5 ; CyborgV2 0,5 ; Autonome 0,1 ; Human 0,01 ; HumanCitizen 0,01 |
| Complex | (2 560, 2 573, 670) | 35 (35) | `Crowd_Complex` sauf Inside/Capital | CyborgV2 1,0 ; CyborgV1 0,05 |
| CryoRoom | (2 242, 2 255, -155) | 3 (3) | `Crowd_Inside` | CyborgV2 |

Soit environ **0,9 piéton par km de trottoir** dans la capitale (à l'échelle 1).

### 1.5 L'agent (configs `Content/Mass/Crowd`)
- Chaîne : `Mass_Crowd_Character_<Type>_Config` -> `_BaseCrowdConfig/Mass_Crowd_Agent_BaseConfig` ->
  `Mass_Crowd_Replication_BaseConfig`. Traits : mouvement, steering, évitement, ZoneGraph navigation (filtre
  `Crowd_Sidewalk`), StateTree, LOD simulation à tick variable, LookAt, obstacle de navigation, annotation ZoneGraph,
  **réplication Mass** (`MassReplicationTrait` + bulles client). `Mass_Crowd_Character_Template_BaseConfig`
  (avec `Mass_Crowd_BaseCharacter_Server`) n'est parent d'**aucun** des 5 types utilisés.
- Représentation : `MassCrowdVisualizationTrait`, LOD Medium **et** Low forcés en `HighResSpawnedActor`. Avec les
  distances visibles par défaut de MassCrowd (High 0, Medium dès 10 m, Low dès 50 m, Off au-delà de 100 m ; 50 m est
  la limite hors champ), **tout piéton visible à moins de 100 m est un `ACharacter` complet** (CharacterMovement, AnimBP, CombatComponent,
  GravityArea, son) ; le mesh VAT `Cyborg_SM` (ISM) n'est jamais utilisé ; **au-delà de ~100 m il n'y a plus personne**.
- L'acteur (`Mass_Crowd_Cyborg_V1_High` etc., parent `Mass_Crowd_BaseCharacter_Server`, classe `Character`) : synchro
  `FeetLocation ActorToMass` + `MovementSync` (la position Mass suit l'acteur et sa collision capsule), réaction aux
  impacts (`AddImpulse`, `LaunchCharacter`), dégâts et mort (`ApplyDamage`, `Kill`, `KillEntity`), collisions joueur
  remontées au serveur par `PS_MassEventsRep` (composant du `QangaPlayerState`) et `Lib_MassAI.SendToServerCollisionMassAI`.
- StateTree (4 arbres pour les 5 types, `HumanCitizen` réutilise celui de `Human` ; tous ont la même structure, celle de City Sample allégée) : **Walk** (`ZG Find Wander Target` + path follow
  + LookAt) / **Idle** (`ZG Stand` avec délai) / **Escape** (`FindEscapeTarget` sur tag d'annotation).
  Pas de destination, pas de SmartObject, pas d'activité.

### 1.6 Réseau
- Serveur : seul à spawner et simuler (Mass pur, **sans géométrie** côté serveur dédié, cf. mémoire projet).
- Clients : reçoivent les agents par la réplication Mass (`MassCrowdReplicator`, bulles par client), puis spawnent
  localement les `ACharacter` de représentation (processeurs de visualisation `ExecutionFlags=13` = Standalone + Client
  + Editor, donc pas sur le serveur dédié).
- Conséquence : sur client, la position suit l'acteur et sa collision ; sur serveur, elle suit la voie. Les deux
  peuvent diverger (à mesurer en multi, §5).

---

## 2. Cause des bugs

### 2.1 « Elles tournent en rond sur le même circuit » : le graphe a des pièges (MESURÉ)
Le StateTree fait une **marche aléatoire pure** : en bout de voie, `MassZoneGraphFindWanderTarget` (code moteur) tire
une voie **sortante** au hasard ; s'il n'y en a pas, il fait demi-tour sur la voie **adjacente**. J'ai rejoué
exactement cette règle sur le graphe réel (composantes fortement connexes, Tarjan) :

- Le gros de la capitale (10 705 voies, 242 km) est bien un réseau d'un seul tenant, **mais il fuit dans 13 « puits »** :
  des morceaux d'où l'on ne ressort jamais, parce que des raccords entre shapes ne sont faits que dans un sens.
- **Îlots isolés** (aucune entrée, les IA y naissent et n'en sortent jamais) :

| Taille | Voies | Position (X ; Y) | Part des points de spawn | Effet |
|---|---|---|---|---|
| 4 696 m | 164 | (-423 ; 847), sud-ouest | 1,9 % | **circuit fermé** : ~3 IA y tournent pour toujours |
| 282 m | 14 | (388 ; 153) | 0,11 % | petit circuit |
| 17 à 45 m | 2 chacun, 7 tronçons | (357 ; 216), (354 ; 240), (361 ; 239), (107 ; 100), (587 ; 18), (555 ; -331), (-1 200 ; -672) | 0,07 % au total | ping-pong sur place |

- **Puits absorbants** (on y entre depuis le réseau, on n'en sort pas) :

| Taille | Position (X ; Y) | Nature |
|---|---|---|
| **17 m** | **(1 122 ; -152)**, est | zone polygone 363 : 2 voies sans sortie, adjacentes l'une à l'autre, entrées depuis 4 voies |
| 109 m | (348 ; -127) | boucle de 6 voies à sens unique (zones 1061, 1093, 1100, 1114 à 1116) |
| 678 m | (-77 ; 26), centre | polygone 2214 (12 voies) + voies voisines |

- **Simulation** (agents répartis comme le spawner, 1,4 m/s, sans respawn) : part des IA piégées
  **4,3 % après 1 h, 16,8 % après 8 h, 36,1 % après 24 h**. Le polygone de 17 m à (1 122 ; -152) en avale le plus
  (27 % de toute la foule au bout de 24 h). Un serveur dédié qui tourne des jours finit donc avec une bonne partie
  de la foule coincée dans 3 endroits.

### 2.2 « Elles s'empilent à certains endroits »
1. **Les puits ci-dessus** (MESURÉ) : les IA qui y entrent font du ping-pong sur 17 m et s'accumulent. **C'est la
   cause des piles à l'arrêt.**
2. **L'évitement en marche est neutralisé par la config** (LU dans le code moteur 5.7.3, à confirmer au runtime) :
   `Config/DefaultMass.ini` place `MassMovingAvoidanceProcessor` en `FrameEnd` (défaut `PrePhysics`).
   `MassSteerToMoveTargetProcessor` (PrePhysics) **écrase** la force (`Force.Value = SteerK * (...)`,
   `MassSteeringProcessors.cpp:238`) et `MassApplyForceProcessor` l'applique puis la remet à zéro
   (`MassMovementProcessors.cpp:58`) : la force d'évitement calculée en fin de frame est écrasée avant d'être lue.
   Les piétons en marche se traversent.
3. **La séparation à l'arrêt marche** (correction du 20/11, relu dans le code) : `MassStandingAvoidanceProcessor` ne
   touche pas la force, il déplace le *ghost* (`FMassGhostLocationFragment`), que le steering relit à la frame
   suivante. En `FrameEnd` il n'a qu'une frame de retard. Ce n'est donc pas lui qui explique les piles.

### 2.3 Défauts secondaires mesurés
- **Graphe chargé 3 fois** : les 3 `ZoneGraphData` de `L_ZoneGraph_Merging` ont chacun le graphe complet (12 855
  voies, mêmes bornes) ; le moteur enregistre chaque acteur sans dédoublonner (`ZoneGraphSubsystem.cpp:93`). Effets :
  mémoire et requêtes x3, densité par voie et zones d'attente éclatées sur 3 copies, spawn tiré sur les 3 copies.
- **`MassCrowdSettings.CrossingTag = CrowdTag`** (`DefaultPlugins.ini`, bit 2) : les 12 855 voies sont traitées comme
  des passages piétons, toutes **fermées au démarrage** avec une zone d'attente chacune (`MassCrowdSubsystem.cpp:193`).
  Sans effet visible aujourd'hui (aucun composant d'annotation de voie trouvé dans les levels de foule), mais coût
  CPU/mémoire inutile et piège dès qu'on ajoutera les annotations (lot 4).
- **L'état Escape ne peut pas se déclencher** : il attend un tag d'annotation, or aucun `ZoneGraphDisturbanceAnnotation`
  ni annotation de foule n'a été trouvé dans `Content/Mass` et les levels ZoneGraph, et aucune section
  `ZoneGraphAnnotations` n'existe dans les `.ini`.
- **Ce que la recherche n'a pas couvert** : un acteur d'annotation posé dans un autre level du jeu. À vérifier au runtime.

---

## 3. Ce qui casse ou vieillit en 5.7
| Point | Mesure | Risque |
|---|---|---|
| Configs, StateTrees, acteurs de foule enregistrés en format 5.3 (`UE5 = 1009`), sauf le StateTree CyborgV2 et les spawners (5.7) | lu dans les en-têtes | StateTree recompilé au chargement ; une erreur de liaison serait silencieuse en jeu : **lire le log** (§5) |
| `Plugins/Traffic` désactivé, jamais compilé ici | pas de `Binaries` | contenu `Content/Mass/Traffic` mort ; ne pas le réactiver sans portage |
| `DefaultMass.ini` : évitement, `DynamicObstacle`, `LODCollector` en `FrameEnd` | lu | évitement en marche mort, séparation à l'arrêt avec une frame de retard (§2.2) ; obstacles et LOD décalés d'une frame |
| `CrossingTag` = `CrowdTag` | lu | §2.3 |
| Réplication Mass | toujours présente en 5.7 mais peu documentée par Epic | à éprouver à 500 joueurs (lot 7) |
| Logs disponibles (`Saved/Logs`, 19 fichiers) | aucune session Univers avec la foule | aucun diagnostic runtime possible sans relancer |

---

## 3 bis. Nos IA de l'univers et la capitale (demande de Benjamin du 19/11)

Constat de Benjamin : en ville on ne croise que des citoyens, alors que l'univers a police, cyberpolice, drones,
Autonomes, gardes, pirates, Voss. Ce qui existe et pourquoi ça n'arrive pas en ville :

### Deux populations qui ne se parlent pas
| | Foule Mass (capitale) | QAI (reste de l'univers) |
|---|---|---|
| Rôle | figurants d'ambiance | agents de gameplay (combat, poursuite, police) |
| Nombre | ~220 x échelle, global | `Clamp(512 / nb joueurs, 1, 32)` par joueur : 32 jusqu'à 16 joueurs, puis ça baisse, ~1 par joueur à 500 |
| Déplacement | voies ZoneGraph | navmeshes Recast/Detour partagés (`QAI_GroundNavigation`) ; `QAI_ARCHITECTURE.md` §9 (flow field) n'est plus à jour |
| Autorité | serveur + réplication Mass | serveur, pawns répliqués |
| Rendu de loin | rien au-delà de ~100 m | **tier impostor VAT** (`QAI_ImpostorSubsystem`) |
| Faction | `CombatComponent` = None (citoyen) | factions `E_Factions` / `EQAI_Faction` |

Aucun lien dans le code : ni QAI, ni QPolice, ni QCombat ne référencent Mass (grep des sources).

### Ce qui existe déjà pour la ville, mais n'est pas branché (mesuré sur disque)
- **`Sous_Levels/L_Gameplay_AI`** (modifié 2026-04-27) : **22 `AI_Spawner` + 1 `Drone_Spawner` déjà posés dans la
  capitale** (ex. (-178 ; 523), (515 ; 2 566), (-623 ; -443)), classe `AI_Cyborg`. Aucun persistant de jeu ne le charge
  (seulement listé dans `EasyCookGenerated.ini`). Chargeur à retrouver, ou intention à demander à RzZz.
- **`_Optimised_Lvl_Capital/L_Gameplay_Master_Optimised`** : 3 `QAI_Planetary_Volume` (nids Sangline) et des spawners
  de véhicules dans la ceinture sud ; non chargé en jeu (le jeu charge `Sous_Levels/L_Gameplay_Master`, qui n'a que 2
  spawners de véhicules).
- **Commissariats** : `L_Capital_PoliceStation_Sud`, `L_Capital_PoliceStationTraverse`, `L_Complex_PoliceStation`
  (acteurs `BP_Police`, `WV_PoliceVolume`) ; `L_Gameplay_Vehicles` (chargé en jeu) référence `AI_DronePolice`.
- **Briques QAI prêtes** : `AQAI_GuardPatrolSpawner` (patrouilles A vers B, capitaine, escouades, archétype Cyborg),
  `AQAI_AgentSpawner` (proximité, respawn, faction forcée), override par volume `QAI_Planetary_Volume` (liste de spawn
  propre à une zone), archétypes police `Drone` (vol, taser capitaine) et `Autonomous` (infanterie).

### Pourquoi la police ne « vit » pas en ville
1. **Spawn planétaire** (`QAI_Planetary_Manager`) : pensé pour le terrain WorldScape autour du joueur (biomes, occlusion
   par traces). Le graphe contient une branche `ExcludeZone` et des overrides par volume : ce qu'ils donnent en ville
   n'est **pas mesurable hors moteur** (lot 0, `QAI.Debug 1`).
2. **QPolice ne fait que réagir** : la police apparaît sur crime (wanted), à ~40 m hors champ. Pas de présence de fond.
3. **La capitale n'est probablement pas reconnue comme territoire Iclab** : `QTerritoryDetectionComponent` cherche des
   `TriggerVolume` tagués ou le **nom de la carte** (`L_Persistent_Universe`) contre `Iclab/Police/City/Urban` ;
   aucun tag `IclabTerritory` trouvé dans `Content/Maps`. À confirmer en jeu (le défaut est `NeutralSpace`).
4. **Tuer un citoyen Mass déclenche-t-il un crime ?** Les acteurs de foule ont un `CombatComponent` (faction None) ;
   le lien vers QPolice n'est pas vérifié hors moteur.

### Principe de cohérence proposé
- **Une seule ville, deux couches** : Mass = la foule dense et bon marché ; QAI = les personnages qui comptent
  (police, cyberpolice, drones, gardes, et incursions pirates ou Voss dans les quartiers chauds).
- **La capitale devient territoire Iclab** par volume tagué (zones StarMap officielles), quartier par quartier :
  centre et Complex très surveillés, ceinture sud et quartiers pauvres moins, zones « chaudes » où pirates et Voss
  peuvent apparaître.
- **Présence de fond** : patrouilles à pied (`QAI_GuardPatrolSpawner`) sur les grands axes et autour des commissariats,
  drones en altitude, gardes aux portes du mur. **Budget** : le plafond QAI par joueur tombe à ~1 à 500 joueurs, donc
  la présence policière de fond ne peut pas venir du quota par joueur. Il lui faut un budget propre à la ville (pool
  fixe de patrouilles partagé par tous les joueurs présents), à chiffrer au lot 7.
- **Cohérence des réactions** : un crime contre un citoyen alimente QPolice ; une fusillade ou une poursuite QPolice
  pose une annotation de perturbation ZoneGraph et la foule fuit (lot 4) ; la foule s'écarte d'une patrouille.
- **Même rendu de loin** : réutiliser les bakes VAT du tier impostor QAI (`ImpostorBakes`, mêmes meshes
  `Original_Cyborg_V5` / CyborgV2 que la foule) au lieu de refaire des VAT de foule.
- **Plus tard** : promouvoir un citoyen Mass en agent QAI quand il est pris dans un combat (il fuit ou se défend
  avec la vraie IA).

### Sol WorldScape et zones intra-muros (Benjamin, 19/11)
Benjamin veut remplacer le sol cuit par le terrain WorldScape et faire tourner le spawn planétaire QAI dans
l'enceinte, hors des rues. Mesuré sur disque :
- Sol actuel = grille 16 x 16 de `Content/Baking_Capital/HM_Bake_Capital_*` (256 fichiers, 203 tuiles posées par
  l'acteur, cf. `YELLOWWALL_SOL_WORLDSCAPE.md`), cuites (WorldScape
  `WS_BakingToStaticMesh`) depuis la heightmap **`/Game/Maps/ComplexWallHM1_coqdzqdpy_Tex2D`**.
- Le terrain WorldScape de la capitale est déjà creusé par **`TerrainWall_HMVolume2`** (heightmap
  `/Game/Maps/TerrainrWallHM`, origine, MapHeight 35 000) + `OceanWall_HMVolume` (`OceanWallHM1`) +
  `CapitalHangarLift_OceanExclusion_HMVolume`, dans `L_Persistent_Universe`. **Ce sont deux heightmaps différentes** :
  la comparaison est le préalable.
- Couplage à surveiller : les voies ZoneGraph, les routes et les bâtiments sont calés sur le sol cuit. Tout écart de
  hauteur du nouveau sol ferait flotter ou enterrer les piétons. **Mesuré par l'étude sol** : aucun point de voie à
  moins de 2 m du sol cuit, 99 % à plus de 20 m au-dessus du sol WorldScape actuel ; le sol retenu est posé sur
  l'acteur cuit, donc **aucun recalage de voie n'est nécessaire** (voir `YELLOWWALL_SOL_WORLDSCAPE.md` §2.4).
- La décision du 22/10 (« ne pas toucher la version cuite ») est levée par cette demande : à confirmer noir sur blanc.

**Trois zones, une seule source de vérité** (les zones StarMap + des volumes) lue par les deux systèmes :
| Zone | Qui peuple | Comment |
|---|---|---|
| Ville (rues, ZoneGraph) | foule Mass + patrouilles QAI | spawn planétaire exclu ou liste « ville » (police, drones) |
| Intra-muros hors ville (sol WorldScape, pas de route) | spawn planétaire QAI autour du joueur | liste « intra-muros » (police, drones, Autonomes, animaux urbains, incursions) |
| Hors mur | spawn planétaire actuel | inchangé |
Une bande de transition où la densité Mass décroît pendant que l'anneau QAI prend le relais, et un budget par joueur
commun (la foule Mass ne consomme pas le quota QAI).

---

## 4. Plan d'amélioration par lots (base d'abord)

Chiffrage en jours de travail de session (Claude + vérification), hors temps de validation de Benjamin et de
compilation par RzZz. Chaque lot se termine par une mesure, pas par « c'est fait ».

### Lot 0 : mesurer au runtime (0,5 j) [aucune écriture]
Éditeur avec verrou, `L_Dev_Claude` + chargement de `L_ZoneGraph_Merging` et `L_Mass_Capital` en lecture :
nombre de `ZoneGraphData` enregistrés (attendu 3), force d'évitement des agents (attendu 0), log StateTree/Mass,
positions de la foule toutes les 60 s pendant 15 min en PIE serveur + 2 clients (piles et pièges réels).

### Lot 1 : corriger les bugs (1,5 à 2 j) [touche un level de jeu et 2 `.ini` : accord explicite requis]
1. `DefaultMass.ini` : remettre les deux processeurs d'évitement en `PrePhysics` (retirer leurs sections). Mesure avant/après.
2. `L_ZoneGraph_Merging` : garder **un** `ZoneGraphData`, supprimer les 2 autres, reconstruire le graphe.
   Le level a été modifié le 2026-08-24 : vérifier avec RzZz qu'il n'y travaille pas.
3. `CrossingTag` -> `Crowd_CrossRoad` (bit 5, 12 voies).
4. Réparer les 13 puits : rebrancher les shapes (connecteurs des polygones 363 et 2214, boucle à sens unique des zones
   1061 à 1116, îlot de 4,7 km au sud-ouest), supprimer les 7 tronçons isolés.
5. Garde-fou permanent : le script `Saved/Claude/uasset_offline/zg_crowd_graph.py` devient un test (QATS ou CLIScape)
   « aucun puits, aucun îlot » à relancer après chaque modif de ZoneGraph.
- **Fini quand** : 0 puits au test, 0 % piégé à 24 h simulées, évitement non nul mesuré, plus de pile après 15 min en PIE.

### Lot U : nos IA dans la capitale (4 à 6 j) [après le lot 1, en parallèle des lots 2 à 4]
1. Mesure (dans le lot 0) : ce que `QAI_Planetary_Manager` fait en ville, territoire QPolice détecté, crime ou non sur
   la mort d'un citoyen, rôle de `L_Gameplay_AI` (question à RzZz).
2. Territoires : volumes tagués Iclab / chaud par quartier StarMap (sur duplicata, puis intégration validée).
3. Présence policière de fond : patrouilles à pied (`QAI_GuardPatrolSpawner`), drones, gardes aux portes ;
   commissariats comme points de départ ; budget plafonné par joueur.
4. Liste de spawn « ville » (override par volume) : police, Autonomes, drones ; pirates et Voss seulement en zone chaude.
5. Ponts de réaction foule / QPolice (avec le lot 4).
- **Fini quand** : en 15 min de marche en ville on croise police et drones à des endroits plausibles, la foule réagit
  à une intervention, et le pool de la ville ne réduit pas le quota QAI des joueurs hors de la ville.

### Lot T : sol WorldScape dans l'enceinte (session dédiée, à chiffrer par elle)
Comparer `ComplexWallHM1` (sol cuit) et `TerrainrWallHM` (WorldScape), choisir la heightmap, remplacer les tuiles
`HM_Bake_Capital` sans rien casser (collision, serveur dédié, règle GPU du skill `terrain-gpu`). Prérequis de la
zone « intra-muros » du lot U. Pas de recalage des voies ZoneGraph (mesuré, voir ci-dessus).

### Lot 2 : destinations et itinéraires variés (3 à 4 j)
Remplacer la marche aléatoire par des trajets : chaque piéton reçoit une destination (station QTrain, place, commerce,
parc, arrêt) et suit un chemin calculé (`ZoneGraphAStar`, présent dans le moteur), avec variation d'itinéraire et
retour à la marche libre en route. Tags de destination sur les zones. Nouvelles tâches StateTree en
**C++ natif** (règle native-first du projet), dans un module à créer ou à choisir avec RzZz ; compilation éditeur
de notre côté, cibles jeu par RzZz.

### Lot 3 : activités (4 à 6 j + animations)
SmartObjects (déjà activés, `MassSmartObjects` moteur) : s'asseoir sur les bancs, discuter à 2 ou 3, téléphoner,
attendre au passage piéton ou à l'arrêt, regarder une vitrine, fumer. Réutiliser bancs et mobilier déjà posés dans les
quartiers. Besoin d'animations (montages sur l'AnimBP de foule existant).

### Lot 4 : réactions (3 à 5 j)
- Joueur proche : écart, regard (LookAt déjà là), bousculade (impacts déjà gérés côté acteur).
- Véhicules et danger (tirs, explosions, police) : annotations de perturbation ZoneGraph posées par le **producteur**
  de l'événement (QWeapon, QPolice, véhicules), ce qui réveille l'état Escape qui existe déjà.
- Météo et nuit : lecture de l'état météo existant (pas de sondage) pour presser le pas, s'abriter, vider les rues.

### Lot 5 : densité par quartier et par heure (2 à 3 j)
Utiliser les tags `Crowd_Density_High/Mid/Low` déjà déclarés (0 voie taguée aujourd'hui) + `LaneDensities` de MassCrowd,
une courbe par heure du jour, et un spawn **autour des joueurs** (anneaux d'apparition hors champ) au lieu d'un
lot fixe de ~220 IA pour 250 km. Quartiers = zones StarMap officielles.

### Lot 6 : animations et rendu (3 à 5 j, dépend des assets)
Remettre le LOD Low en ISM animé par VAT (assets `Character/VAT` déjà cuits) pour avoir une foule visible au-delà de
100 m ; acteurs complets seulement près de la caméra ; variété via `MassCrowd_Customize_*` (existant) ; ratio humains /
cyborgs à décider (aujourd'hui ~2 % d'humains).

### Lot 7 : performance et multi à 500 joueurs (3 à 5 j)
Banc headless (comme `QAI.Debug.StressBench`) : coût serveur par agent, coût des bulles de réplication par client,
budget d'acteurs par client. **Décision d'architecture à prendre** : piétons tous serveur (tuables, coûteux en réseau)
ou foule hybride (piétons d'ambiance simulés sur chaque client, non répliqués, + piétons « importants » serveur).

**Total** : environ 25 à 35 jours. Pour la démo Steam du 17/12 : lots 0 et 1 obligatoires, puis le lot U (police et
drones en ville, le plus visible pour « un univers vivant »), puis 5 et 6 (densité et foule visible de loin).

---

## 5. Reste à mesurer (moteur requis, verrou + accord)
1. Confirmer au runtime : 3 graphes enregistrés, force d'évitement nulle, emplacement réel des piles.
2. Log 5.7 au chargement des StateTree de foule (liaison, recompilation).
3. Valeur effective de l'échelle de spawn (paramètre global) et donc du nombre réel d'IA.
4. Multi : divergence serveur / client des positions, coût de la réplication par client.
5. Existence éventuelle d'un acteur d'annotation ZoneGraph dans un level non couvert par la recherche.
6. QAI en ville : ce que font `ExcludeZone` et les overrides du spawn planétaire, territoire QPolice détecté,
   crime sur la mort d'un citoyen, chargeur de `L_Gameplay_AI`.

## 6. Outils
- `Saved/Claude/uasset_offline/zg_crowd_dump.py <umap> <sortie.pkl>` : extrait le stockage ZoneGraph d'un level.
- `Saved/Claude/uasset_offline/zg_crowd_graph.py <sortie.pkl>` : composantes, puits, îlots, simulation de piégeage.
- `Saved/Claude/uasset_offline/mass_level_spawners.py <umap>` : spawners Mass d'un level et leurs réglages.
