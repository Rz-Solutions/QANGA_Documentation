# Migration Blueprint vers C++ - preuves performance et audit des dépendances P2

## Décision

- Aucune capture existante n'est une baseline comparable et représentative du chemin joueur + véhicules + combat + inventaire. Aucun gain avant/après, en millisecondes ou en pourcentage, ne peut être attribué aux migrations actuelles.
- Les traces existantes prouvent seulement deux workloads spécialisés: un audit Editor PlanetScape/Mercury et une séquence Editor StarMap ouverte/fermée. Cinq snapshots `EditorPerformance` supplémentaires n'ont aucun export EasyTraceAnalyzer ni scénario gameplay exploitable.
- La taille d'un graphe n'est pas une priorité. Le classement P2 ci-dessous repose sur la fréquence structurellement prouvée, la multiplication par le nombre d'acteurs, la disponibilité d'un propriétaire natif et les dépendances P1.
- Les vrais candidats récurrents sont le runtime missile, le scan/ordonnancement des tourelles et le reliquat gameplay des véhicules. La persistance ItemsManager/DataManager est un hotspot de burst et de correction, pas un coût steady-state prouvé. Les quêtes isolées restent Blueprint.

## Périmètre et instantané de preuve

- Révision auditée: `dc587c4530`, branche `master`, worktree partagé et fortement dirty le 30 août 2026.
- Index RzMCP: schéma `4`, complet, `58 428` assets `/Game`, généré le `2026-08-30T15:17:16.899Z`. Les assets plugin DataManager ont été interrogés directement via l'Asset Registry.
- Les comptes de graphes/nœuds et référents viennent de lectures RzMCP effectuées avant la fermeture de l'Editor. Aucun asset n'a été chargé, compilé, sauvegardé ou rescanné pour cet audit.
- Les modifications concurrentes non committées des autres workers ne sont pas considérées comme des contrats P1 intégrés.
- Le dépôt ne contient encore aucun module natif Inventory ou Combat neutre. Ils restent des prérequis, pas des propriétaires que P2 peut deviner.

## 1. Inventaire Unreal Insights et EasyTraceAnalyzer

### 1.1 Toutes les captures présentes

| Capture | Octets | Nature prouvée | Export ETA | Utilisable pour P2 |
|---|---:|---|---|---|
| `Saved/EasyTraceAnalyzer/PlanetScape_Mercury_Editor_Audit_20260820.utrace` | 237 598 556 | Session Editor PlanetScape/Mercury | Oui, 2 fenêtres | Non: Slate/Editor et renderer dominants, aucun scénario P2 |
| `Saved/Profiling/starmap_adaptive_runtime.utrace` | 79 599 748 | Session Editor StarMap | Oui, info + global + ouvert/fermé | Seulement comparaison locale StarMap, pas migration P2 |
| `Saved/EditorPerformance/Cache_Local_Efficiency_1.utrace` | 36 740 253 | Snapshot automatique EditorPerformance | Non | Non |
| `Saved/EditorPerformance/Cache_Local_Efficiency_2.utrace` | 36 829 744 | Snapshot automatique EditorPerformance | Non | Non |
| `Saved/EditorPerformance/Memory_Memory_Pressure_1.utrace` | 36 736 449 | Snapshot mémoire EditorPerformance | Non | Non |
| `Saved/EditorPerformance/Memory_Memory_Pressure_2.utrace` | 36 769 623 | Snapshot mémoire EditorPerformance | Non | Non |
| `Saved/EditorPerformance/Memory_Memory_Pressure_3.utrace` | 34 711 604 | Snapshot mémoire EditorPerformance lié à une session de crash | Non | Non |

`Saved/EasyTraceAnalyzer/trace_list.json` confirme que le store global `C:/Users/Rz/AppData/Local/UnrealEngine/Common/UnrealTrace/Store/001` contient `0` trace. Les sept captures ci-dessus vivent hors du store et doivent donc être inventoriées directement.

Les noms `Memory_Memory_Pressure_1/2/3` apparaissent dans le log du crash `UECC-Windows-D539...` comme snapshots pris le 21 août. Les dates disque de `_1` et `_2` sont plus récentes, ce qui montre que ces noms peuvent être réutilisés/écrasés. Sans sidecar de scénario ni export, ils ne forment pas une paire avant/après fiable.

### 1.2 Exports existants

| Export | Fenêtre | Frames actives | Résultat descriptif | Limite |
|---|---:|---:|---|---|
| `PlanetScape_..._analysis.json` | `789.0-812.0 s` | 701 | moyenne `9.04 ms`, médiane `8.87`, p95 `11.7`, p99 `12.9`, max `26.6` | Editor; Slate Tick/Draw environ `4.65 ms` inclus |
| `PlanetScape_..._stable.json` | `809.1-829.4 s` | 2 153 | moyenne `9.34 ms`, médiane `8.71`, p95 `11.7`, p99 `13.3`, max `294.4` | chevauche le premier export; frame gap et spike Editor |
| `starmap_adaptive_runtime_info.json` | `0-737.727 s` | 1 165 | moyenne Game Frame `16.379 ms` | trace Editor, longue période vide/non segmentée |
| `starmap_adaptive_runtime_analyze.json` | `0-737.727 s` | 1 165 | moyenne `16.4 ms`, p95 `18.5`, p99 `20.2`, max `88.6` | agrégat de toute la session |
| `starmap_closed_idle.json` | `728.1-730.4 s` | 143 | moyenne `16.1 ms`, p95 `17.7`, p99 `18.8`, max `19.3` | StarMap fermée uniquement |
| `starmap_open_idle.json` | `731.0-735.5 s` | 280 | moyenne `16.1 ms`, p95 `17.5`, p99 `20.3`, max `88.6` | durée différente et outlier d'ouverture |

La paire StarMap ouverte/fermée vient de la même trace et permet une comparaison locale de l'UI. Elle contient `32 827` appels agrégés `ReceiveTick` sur la session, mais aucun timer nommé ItemsManager, DataObject, projectile, véhicule, tourelle ou Combat dans les classements exportés. Elle ne prouve donc rien sur les familles P2.

Canaux activés dans la trace StarMap: `Cpu`, `Gpu`, `Frame`, `Stats`, `Screenshot`, `Region`, `Bookmark`, `Log`. `Net`, `Object`, `Counters`, `Task` et `Slate` sont désactivés. Il est impossible d'en déduire les acteurs présents, les RPC/bytes, les appels de bridge, les tâches de persistance ou l'attribution exacte d'un timer Blueprint.

### 1.3 Verdict de comparabilité

Il n'existe aucune paire répondant simultanément à ces critères:

1. même build/changelist et même configuration;
2. même map, seed, save, caméra, scalabilité et phase de jeu;
3. mêmes nombres d'acteurs, transactions, tirs et projectiles;
4. même topologie réseau;
5. mêmes canaux et régions de trace;
6. baseline prise avant la verticale puis capture après avec un seul changement fonctionnel.

Les migrations P0/P1 déjà documentées peuvent revendiquer des suppressions structurelles de scans, traces et ticks. Elles ne peuvent pas revendiquer un delta runtime rétroactif.

## 2. Carte des coûts restants

### 2.1 Travail récurrent ou multiplié

| Producteur actuel | Travail prouvé | Fréquence/multiplication | Dedicated server | Qualification |
|---|---|---|---|---|
| `BP_Projectile` | `ReceiveTick` actif puis `CheckCollision` (`81` nœuds) | par projectile et par frame; parent `InitialLifeSpan=0.02 s` mais les enfants/usages peuvent le modifier | tick autorisé | Candidat VM réel, concurrence runtime inconnue |
| `BP_Missile` | Event `ReceiveTick` présent mais sans lien d'exécution | aucune charge VM prouvée sur cet event | tick autorisé sur le parent | Nœud mort, pas un gain à compter |
| `MissileMovementComponent` | `ReceiveTick` et calcul de mouvement | par missile actif et par frame | tick autorisé | Candidat VM réel |
| `ProjectilesManager` | `ReceiveTick` vers `CheckFlaresHit` | un manager, coût multiplié par mouvements/flares parcourus | non exclu structurellement | Scan global/manager réel |
| `VehicleBase` | chaîne Tick: `FastSpeedCollisionTest`, caméra collision, FOV FX, velocity, speed control, rollover | par véhicule dont l'actor tick n'est pas parqué | actor tick autorisé; réplication active | Mélange gameplay + présentation à séparer |
| `FlyVehicleMovementComponent` BP | ancien Tick BP désactivé sur le chemin C++; boucle latente `LoopTimers` vers `CalcDamage` et `CheckUnderwater`, délai `0.5 s` | par composant actif à cadence fixe | tick BP autorisé par défaut | Le gros graphe n'est pas entièrement actif; boucle fixe réelle |
| `UVehicleMovementComponent` C++ | physique, collisions, LoD et parking | par véhicule actif; tick natif `PrePhysics` | explicitement autorisé | Propriétaire natif correct, ne pas le dupliquer |
| `VehicleCombatComponent` | `ReceiveTick`/`DelayedUpdate` résiduels après P0 | par véhicule armé | non exclu par le wrapper BP | À attribuer par trace; ciblage spatial déjà natif |
| `TurretBase` | macro `Loop Timer` à `1.0 s`, `FindNearest` (`53` nœuds), update target/aim/fire | par tourelle; scan nearest multiplié par les candidats | tick autorisé; acteur répliqué | Candidat fixe/global réel |
| `ItemsManagerGS` | requêtes, world drops, shops, maps persistantes | événementiel/burst | composant GameState/server | Grand graphe, pas de coût frame prouvé |
| `DataObject` | encode/decode, parser, setters/getters et continuations latentes | par save/load/flush et volume de données | serveur/persistance | Hotspot de burst plausible et inspectable, pas steady-state |
| Objectifs quêtes isolés | callbacks de transaction/pickup/shop | événementiel | selon objectif | Pas un hotspot prouvé |

Le C++ `UVehicleMovementComponent` active son tick seulement quand nécessaire, permet le dedicated server et parque l'actor `VehicleBase`, les skeletal meshes auxiliaires et les widgets lorsque le véhicule dort. Migrer aveuglément les `1 358` nœuds du wrapper BP recréerait un propriétaire concurrent. La seule verticale sûre porte sur le reliquat encore exécuté et consomme le composant natif existant.

### 2.2 Reflection et `ProcessEvent` encore sur les frontières P1

| Appel actuel | Frontière réelle | Dépendance |
|---|---|---|
| `QWeaponBulletSubsystem.cpp:570-592` appelle `GetEquippedItemInstanceBySlot` par `FindFunction/ProcessEvent` | Weapon lit l'inventaire BP | contrat P1 Inventory |
| `QWeaponBulletSubsystem.cpp:729-763` sonde `IsAlive`, `CurrentLife`, `OnDamage`, `OnDamaged` | Weapon devine le contrat Combat | contrat P1 Combat |
| `QModule_InventoryBridge` centralise génération, ajout, consommation et lecture des item instances par reflection | QModule consomme Inventory | transaction P1 Inventory |
| `QModule_PersistenceBridge_World_SubSystem.cpp` résout `GameDataManager_C`, `FindDataObjectById`, `IsReadyData`, `Get/SetStringArray` par reflection | QModule consomme DataManager BP | codec/persistence typé P2 après Inventory |
| `QuestManagerSubsystem.cpp:9517-9585` lit `GetItemDataAsset` et `GetItemInstanceId` par reflection | DQS consomme Obj_ItemInstance | record P1 Inventory |
| `QuestActionBase.cpp:1575-1613` appelle `RemoveItem` par reflection | action de quête mute Inventory | transaction P1 Inventory |

Ces bridges ne doivent pas être migrés indépendamment avant P1. Le bon résultat est un appel typé vers un seul record/transaction Inventory et un seul contrat Combat. Ajouter un deuxième adaptateur P2 serait un fallback et maintiendrait deux autorités.

Le core natif `DynamicQuestSystem` contient aussi des `TActorIterator`, un `GetAllActorsOfClass` sur `AQuestTriggerActor` et plusieurs appels Blueprint par reflection. Ce sont des sujets d'optimisation du core DQS à mesurer séparément. Ils ne rendent pas les petits objectifs designer-authored responsables d'un hotspot.

## 3. Audit par famille P2

### 3.1 ItemsManagerGS et DataManager

| Asset/module | Surface actuelle | Référents Asset Registry | Propriétaire actuel |
|---|---:|---:|---|
| `/Game/Systems/Item/ItemsManagerGS` | 21 vars, 33 fonctions, 12 events, 3 dispatchers, 547 nœuds | 20 | Blueprint GameState item/persistence |
| `/DataManager/GameDataManager` | 6 vars, 18 fonctions, BeginPlay, 1 dispatcher, 165 nœuds | 26 | Blueprint DataManager |
| `/DataManager/DataObject` | 30 vars, 68 fonctions, 1 macro, 2 events, 3 dispatchers, 1 085 nœuds | 72 | Blueprint DataManager |
| `/DataManager/DataManagerLib` | 11 fonctions, 114 nœuds | 52 | Blueprint function library |
| `/Game/Systems/Item/InventoryComponent` | 44 vars, 67 fonctions, 3 macros, 30 events, 13 dispatchers, 1 595 nœuds | dépendance amont | P1 Inventory BP, propriétaire natif absent |
| `/Game/Systems/Item/Obj_ItemInstance` | 18 vars, 35 fonctions, 282 nœuds | dépendance amont | P1 Inventory BP |
| `/Game/Systems/Item/Lib_Inventory` | 20 fonctions, 303 nœuds | dépendance amont | P1 Inventory BP |

Référents représentatifs d'`ItemsManagerGS`: `QangaGameState`, `InventoryComponent`, `Obj_ItemInstance`, `Lib_Inventory`, `Lib_ItemSystem`, `ItemDroppedReplicator`, `MarketDynamicRatesComponent`, mission/trade/shop helpers, objectifs de livraison et spawners de loot. Il est live et partagé.

Référents représentatifs DataManager: GameState, Inventory/ItemsManager, stats, missions, véhicules, météo, builder et QModule. Le contenu du plugin est live. En revanche, les dépendances des trois Blueprints DataManager n'incluent pas `/Script/DataManager`: le module C++ ne fournit actuellement aucun contrat runtime.

`DataObject` porte notamment `BlockingThreadEncode` (`128` nœuds) et `DecodeParserToData` (`83` nœuds), ainsi que les codecs de tableaux et les continuations encode/decode. C'est une verticale de burst mesurable lors d'un flush, pas une justification pour réécrire les 68 fonctions.

Commits de propriété/correction récents:

- `14d000eb2` corrige la persistance et le rejet des items invalides dans `InventoryComponent`/`Obj_ItemInstance`, toujours Blueprint;
- `311656ff7` étend le snapshot/restore QModule via le bridge DataManager réfléchi;
- `5a3982a2a` répare le signal du premier transfert vers un inventaire vide et relie les objectifs natifs de construction à ce flux;
- aucun commit récent n'a créé un propriétaire natif Inventory/DataManager.

Verdict: `ItemsManagerGS` n'est pas un hotspot CPU indépendant prouvé. La verticale P2 réelle est un record item versionné, transactionnel et sérialisable, fourni par P1 Inventory puis encodé par DataManager. Le bulk migration ItemsManager/DataObject reste bloqué tant que la trace ne montre pas un scope dominant.

### 3.2 Projectiles legacy

| Asset | Surface actuelle | Référents | Coût/autorité |
|---|---:|---:|---|
| `/Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/BP_Projectile` | 38 vars, 4 fonctions, 10 composants, 9 events, 445 nœuds | 10 | Tick + collision par instance; parent non répliqué |
| `/Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/BP_Missile` | 18 vars, 5 fonctions, 8 composants, 7 events, 200 nœuds | 6 | Event Tick mort; mouvement délégué au composant |
| `/Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/BP_GrenadeProjectile` | 35 vars, 5 fonctions, 6 composants, 7 events, 313 nœuds | 4 | Pas de Tick listé; physique/collision événementielle |
| `/Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/MissileMovementComponent` | 15 vars, 7 fonctions, 2 events, 122 nœuds | 6 | Tick par missile |
| `/Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/ProjectilesManager` | 2 vars, 5 fonctions, 2 events, 52 nœuds | 4 | manager Tick + flares |
| `/Game/Items/Weapons/NashV2/Rocket_Launcher/RocketLauncherMissileProjectile` | child BP_Missile, 1 fonction, 7 nœuds | 3 | projectile arme joueur |
| `/Game/GameplayActors/Turret/TurretMissileProjectile` | child BP_Missile, 1 fonction, 7 nœuds | 4 | projectile tourelle |
| `/Game/VehicleWeapons/Rocket/Missile_VehicleRocketLauncher` | child BP_Missile, 1 fonction, 7 nœuds | 0 | mort candidat |

Les référents `BP_Projectile` sont surtout les familles AI StaticNPC/drone/shop/tutorial/cargo. `BP_Missile` alimente les launchers joueur, tourelle et véhicule. `ProjectilesManager` est référencé par `QangaGameState`, le mouvement missile et `FlareActor`.

Propriétaires natifs existants:

- `QWeapon` possède le fire hitscan et expose `FireVehicleMachineGun`, `FireMountedVehicleMachineGun` et `FireStaticDefenseMachineGun`;
- `QWeaponTargetRegistrySubsystem` et `QWeaponTargetingComponent` possèdent le ciblage spatial/homing depuis `3bd272b02`;
- le mouvement, les flares et la collision des projectiles legacy restent Blueprint;
- le damage/life reste derrière le contrat Combat BP réfléchi.

`5721a9b6c` a migré le type/cache de zone de gravité de la grenade vers la classe native correspondante. Ce commit ne migre ni la trajectoire, ni la collision, ni l'autorité du projectile.

Verdict: le missile + manager de flares est une verticale réelle, récurrente et déjà bordée par QWeapon. `BP_Projectile` et la grenade constituent un modèle physique différent et ne doivent pas être ajoutés au même lot. Leur implémentation reste trace-gated.

### 3.3 Véhicules et tourelles

| Asset/module | Surface actuelle | Référents | Propriétaire actuel |
|---|---:|---:|---|
| `/Game/Systems/Vehicle/VehicleBase` | 84 vars, 40 fonctions, 2 event graphs, 49 events, 32 composants, 2 247 nœuds | 181 | Blueprint shell; physique native partielle |
| `/Game/Systems/Vehicle/SpaceshipBase` | 35 vars, 12 fonctions, 1 160 nœuds | 41 | child VehicleBase |
| `/Game/Systems/Vehicle/FlyVehicleMovementComponent` | 109 vars, 69 fonctions, 5 events, 3 dispatchers, 1 358 nœuds | 94 | wrapper BP + ancien chemin |
| `UVehicleMovementComponent` dans `FlyVehicleMovement` | C++ `UPawnMovementComponent`, tick PrePhysics, sommeil/LoD | n/a | propriétaire physique actuel |
| `/Game/Systems/Vehicle/Weapon/VehicleCombatComponent` | 7 vars, 11 fonctions, 8 events, 2 dispatchers, 156 nœuds | 74 | child `QWeaponTargetingComponent` |
| `/Game/Systems/Vehicle/Weapon/VehicleWeaponComponent` | 6 vars, 4 fonctions, 63 nœuds | 39 | Blueprint event-driven |
| `/Game/GameplayActors/Turret/TurretBase` | 11 vars, 3 fonctions, 15 events, 10 composants, 343 nœuds | 11 | Blueprint |
| `/Game/GameplayActors/Turret/Turret_MachineGun` / `/Game/GameplayActors/Turret/Turret_RocketLaucher` | 2 / 6 nœuds | enfants live | configuration Blueprint |
| `/Game/VehicleWeapons/MachineGun/VWSlot_MachineGun` / `/Game/VehicleWeapons/Rocket/VWSlot_Rocket` | 39 / 39 nœuds | live | fire implementation événementielle |

Defaults réseau/tick lus:

- `VehicleBase`: `bReplicates=true`, actor tick possible, `NetUpdateFrequency=0.5`;
- `TurretBase`: `bReplicates=true`, tick autorisé dedicated, `NetUpdateFrequency=10`;
- `BP_Projectile` et `BP_Missile` parents: non répliqués; l'autorité exacte doit être validée sur leurs enfants et leur point de spawn.

Répartition correcte des responsabilités:

- mouvement/physique véhicule: `FlyVehicleMovement`, pas QVehicles;
- `QVehicles` ne contient actuellement que le système sirène, sans propriétaire mouvement/combat;
- acquisition de cible et fire control compatible: `QWeapon`;
- life, permission de tirer, dégâts et mort: contrat P1 Combat;
- caméra/FOV/HUD: local presentation, ne doit jamais être déplacé sur le dedicated server.

Commits récents:

- `93e38bd31` a renforcé la conduite/physique déterministe dans `FlyVehicleMovement` et modifié `VehicleBase`;
- `3fc2f081d` a ajouté les optimisations runtime/parking autour de `VehicleBase` et du plugin;
- `3bd272b02` a supprimé le scan global de ciblage véhicule et reparenté `VehicleCombatComponent` au propriétaire QWeapon natif.

Verdict: ne pas remigrer le mouvement. Le candidat véhicule est le reliquat gameplay BP encore cadencé. Les petites armes/slots sont des graphes événementiels et consomment le fire control Weapon; leur taille ne justifie pas une migration. La tourelle est un candidat séparé car son timer/scan nearest se multiplie par acteur.

### 3.4 Quêtes isolées

| Asset | Surface | Référents | Verdict |
|---|---:|---:|---|
| `/Game/Systems/QuestSystem/Quest_Client` | 8 vars, 8 fonctions, 328 nœuds | 11 | UI/HUD event-driven, rester BP |
| `/Game/Systems/QuestSystem/BP_QuestActor` | 27 nœuds | live | shell designer-authored |
| `/Game/Systems/QuestSystem/BP_QuestInteractActor` / `/Game/Systems/QuestSystem/BP_QuestPickupActor` | 74 / 74 nœuds | live | interaction événementielle |
| `O_PickUpObjective` | 5 vars, 1 macro, 4 events, 106 nœuds | 2, PrimaryAsset + seed | rester BP; consommer transaction Inventory |
| `O_ShopTransactionObjective` | 13 vars, 1 macro, 5 events, 195 nœuds | 2, PrimaryAsset + seed | rester BP; consommer transaction Inventory |
| `/Game/_QData/Proto_Quest` | 77 nœuds déconnectés, 0 fonction/event | 0 | candidat suppression, pas migration |

Le core `DynamicQuestSystem` est déjà natif et extensible. Le commit `5a3982a2a` a encore ajouté/étendu des objectifs C++ et corrigé l'attribution par l'événement Inventory. Les deux objectifs isolés inspectés n'ont ni Tick ni Delay. Ils ne sont pas des hotspots.

Seule exception future: si P1 Inventory expose un événement transactionnel typé partagé par plusieurs objectifs, une classe objective native commune peut l'adapter dans `DynamicQuestSystemObjectives`. Ce contrat partagé doit remplacer la reflection, pas migrer les graphes de contenu.

## 4. Dépendances et classement P2 exact

### 4.1 Contrats amont obligatoires

| Contrat P1 | Consommateurs P2 autorisés | Ce que P2 ne doit pas posséder |
|---|---|---|
| Weapon fire control et validation d'autorité | missiles, vehicle fire, turret fire | cadence, ammo, ownership ou RPC parallèles |
| Inventory record + transaction atomique/rollback | ItemsManager, DataManager item codec, QModule, quêtes | mutation directe de tableaux/maps BP |
| Combat life/damage/permission snapshot | projectile impact, véhicule/tourelle life et permission, QAI | heuristique `CurrentLife/OnDamage` réfléchie |

### 4.2 Classement

Ce classement est un ordre d'intégration après P1 et après baseline, pas une promesse de gain:

| Rang | Verticale | Preuve | Gate |
|---:|---|---|---|
| 1 | Missile runtime + flare manager | Tick par missile + manager global; owner QWeapon déjà disponible | Weapon + Combat intégrés, trace P2-MISSILE |
| 2 | Turret acquisition/fire/life consumer | timer `1 s`, `FindNearest`, multiplication par tourelle; registre QWeapon disponible | Weapon + Combat intégrés, trace P2-TURRET |
| 3 | Reliquat gameplay VehicleBase/FlyVehicle BP | chaîne par frame + boucle `0.5 s`; mouvement natif déjà owner | Combat intégré, preuve que les scopes sont actifs |
| 4 | Item record persistence verticale | DataObject codec/latent + reflection QModule; burst de save/load | Inventory record/transaction figés, trace P2-ITEM |
| 5 | Ballistic `BP_Projectile` puis grenade | Tick/collision structurels mais durée/concurrence inconnues; grenade sans Tick | mesure d'instances actives et self time avant code |
| 6 | Bulk ItemsManagerGS/DataManager | graphes massifs mais événementiels, aucun coût steady-state prouvé | ne pas router sans hotspot nommé |
| 7 | Quêtes isolées | event-driven/designer-authored, core déjà natif | rester Blueprint |

## 5. Scopes copy-ready pour les verticales prouvées

Les quatre scopes ci-dessous sont disjoints par fichiers source et assets. Ils ne doivent pas être lancés avant leurs gates respectifs. Les builders ne compilent pas et ne pilotent pas l'Editor; le main Codex garde la validation.

### P2-01 - Missile runtime et registre de flares

```text
Objectif: déplacer le mouvement/ordonnancement missile et CheckFlaresHit vers QWeapon, sans changer le fire control, le damage Combat ni la présentation.

Prérequis:
- P1 Weapon fire control intégré et autoritaire;
- P1 Combat expose un impact/damage typé;
- baseline P2-MISSILE valide avec nombres de missiles/flares comptés.

Ownership source exclusif:
- Plugins/QWeapon/Source/QWeapon/Public/QWeaponMissileRuntimeComponent.h
- Plugins/QWeapon/Source/QWeapon/Private/QWeaponMissileRuntimeComponent.cpp
- Plugins/QWeapon/Source/QWeapon/Public/QWeaponProjectileRegistrySubsystem.h
- Plugins/QWeapon/Source/QWeapon/Private/QWeaponProjectileRegistrySubsystem.cpp

Ownership assets exclusif:
- /Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/BP_Missile
- /Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/MissileMovementComponent
- /Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/ProjectilesManager
- /Game/Items/Weapons/NashV2/Rocket_Launcher/RocketLauncherMissileProjectile
- /Game/GameplayActors/Turret/TurretMissileProjectile

À conserver:
- VFX/audio/mesh/trail Blueprint;
- trajectoire visuelle et point d'impact identiques;
- ciblage via QWeaponTargetRegistrySubsystem;
- damage/life via Combat, jamais par property probing.

Hors scope:
- BP_Projectile, BP_GrenadeProjectile;
- Missile_VehicleRocketLauncher, candidat suppression;
- modification du contrat P1 Weapon/Combat.

Gates:
- zéro Tick/Delay Blueprint dans le mouvement/manager migré;
- aucun scan global de flares; registre événementiel avec cleanup exact;
- dedicated server sans présentation, mais avec autorité trajectoire/impact;
- parité single/listen/dedicated et trace après strictement comparable.
```

### P2-02 - Contrôleur de tourelle consommateur Weapon/Combat

```text
Objectif: remplacer Loop Timer/FindNearest/UpdateTarget/FireWeapon de TurretBase par un contrôleur QWeapon typé, sans créer de seconde vie ou permission Combat.

Prérequis:
- P2-01 fournit le projectile missile si le child rocket en a besoin;
- P1 Weapon et Combat intégrés;
- baseline P2-TURRET valide.

Ownership source exclusif:
- Plugins/QWeapon/Source/QWeapon/Public/QWeaponTurretControllerComponent.h
- Plugins/QWeapon/Source/QWeapon/Private/QWeaponTurretControllerComponent.cpp

Ownership assets exclusif:
- /Game/GameplayActors/Turret/TurretBase
- /Game/GameplayActors/Turret/Turret_MachineGun
- /Game/GameplayActors/Turret/Turret_RocketLaucher

Contrats:
- acquisition via QWeaponTargetRegistrySubsystem/QWeaponTargetingComponent;
- fire via P1 Weapon;
- bAlive, permission et application des dégâts via P1 Combat;
- réplication de l'état minimal, authority-only pour la décision de tir.

Hors scope:
- TurretMissileProjectile, propriété P2-01;
- VehicleCombatComponent/VWSlot_*;
- changement des règles faction/range/cadence authored.

Gates:
- zéro GetAllActors/FindNearest Blueprint et zéro boucle latente;
- pas de tick dédié pour mesh/audio/aim presentation;
- mêmes choix de cible et mêmes tirs pour les mêmes snapshots;
- métriques Net et CPU comparables, sans baisse d'event count.
```

### P2-03 - Reliquat gameplay véhicule dans l'owner FlyVehicleMovement

```text
Objectif: retirer du Blueprint uniquement les sorties gameplay encore cadencées que UVehicleMovementComponent possède déjà ou peut produire sans second owner.

Prérequis:
- P1 Combat intégré;
- baseline P2-VEHICLE prouve les scopes VehicleBase/FlyVehicle BP actifs;
- confirmer le gate runtime du chemin C++ sur chaque classe enfant auditée.

Ownership source exclusif:
- Plugins/FlyVehicleMovement/Source/FlyVehicleMovement/Public/FlyVehicleMovementComponent.h
- Plugins/FlyVehicleMovement/Source/FlyVehicleMovement/Private/FlyVehicleMovementComponent.cpp

Ownership assets exclusif:
- /Game/Systems/Vehicle/FlyVehicleMovementComponent
- /Game/Systems/Vehicle/VehicleBase

Verticale minimale:
- FastSpeedCollisionTest, velocity/speed control et rollover vers l'owner mouvement existant;
- remplacer LoopTimers/CalcDamage/CheckUnderwater par un deadline pollé dans la boucle native existante;
- damage/life uniquement via Combat;
- laisser caméra collision et FOV FX en présentation local-player, sans dedicated server.

Hors scope:
- réécriture des 1 358 nœuds legacy dormants;
- SpaceshipBase/WatercraftBase/HovercraftBase tant que la base n'est pas validée;
- fire control, tourelles, sirènes, HUD.

Gates:
- aucun output gameplay doublé entre BP et C++;
- dedicated server sans ReceiveTick VehicleBase BP;
- sommeil/parking natifs conservés et réveil identique;
- caméra/FOV inchangés sur le client possesseur;
- scale 1/8/32 véhicules comparable.
```

### P2-04 - Record item persistant vertical, pas bulk DataManager

```text
Objectif: sérialiser un FInventoryRecord P1 versionné sur le chemin world-drop aller/retour, puis supprimer uniquement ce chemin refléchi/dupliqué d'ItemsManagerGS/DataObject.

Prérequis:
- type record et transaction P1 Inventory intégrés, persistants et figés;
- rollback multi-step prouvé;
- baseline P2-ITEM avec mêmes records/transactions/bytes.

Ownership source exclusif:
- Plugins/DataManager/Source/DataManager/Public/DataManagerItemRecordCodec.h
- Plugins/DataManager/Source/DataManager/Private/DataManagerItemRecordCodec.cpp
- Plugins/DataManager/Source/DataManager/DataManager.Build.cs seulement si la dépendance au module P1 est requise

Ownership assets exclusif:
- /DataManager/DataObject
- /Game/Systems/Item/ItemsManagerGS

Verticale minimale:
- un seul record item et un seul round-trip world drop;
- format versionné et backward compatible;
- DataManager encode/persiste mais ne mute jamais Inventory;
- Inventory valide et commit/rollback la transaction;
- erreurs explicites, aucun fallback vers l'ancien record.

Hors scope:
- migration des 68 fonctions DataObject;
- shops, market rates, missions, stats, véhicules ou météo;
- GameDataManager/DataManagerLib;
- QStorage comme owner Inventory.

Gates:
- round-trip byte/data exact et anciens saves lisibles;
- mêmes nombres de transactions, commits, rollbacks et flushes;
- aucun ProcessEvent item-record sur cette verticale;
- trace Task/CPU/persistence comparable; aucune revendication whole-frame si le burst est sous le bruit.
```

### Scope de mesure seulement - ballistic/grenade

```text
Ne pas implémenter. Capturer P2-PROJECTILE-BALLISTIC avec 1/16/64 instances, durée de vie réelle, appels CheckCollision, impacts et damage events. Router une verticale BP_Projectile séparée seulement si le self time/calls prouve le multiplicateur. La grenade reste séparée car elle n'a pas de Tick listé et possède une physique/gravity contract distincte.
```

## 6. Matrice de traces comparable

Les nombres ci-dessous définissent des fixtures contrôlées, pas une estimation des populations production. Chaque capture doit écrire les comptes réels en counters/bookmarks; une capture qui n'atteint pas les comptes est invalide.

| ID | Scénario/map | Comptes contrôlés | Topologie | Scopes attendus |
|---|---|---|---|---|
| `P2-INTEGRATED` | `/Game/Maps/LevelDev/L_Dev_Rz`, parcours 30 s: on-foot, entrée véhicule, combat MG/rocket, transfert inventory | 1 joueur/client, 1 véhicule actif, 1 tourelle, 8 combatants, 2 inventaires de 40 slots dont 20 occupés, 8 missiles tirés | Standalone puis Dedicated + 2 clients | preuve représentative end-to-end, pas attribution isolée |
| `P2-MISSILE` | même map, phase fixe 20 s après warmup | tiers 1/16/64 missiles simultanés, 0 puis 8 flares, 32 targets stables | Standalone + Dedicated + 2 clients | `MissileMovementComponent`, `ProjectilesManager`, `CheckFlaresHit`, impact Combat, bytes/RPC |
| `P2-TURRET` | même map, cibles immobiles puis mobiles | tiers 1/16/64 tourelles, 64 combatants, zéro projectile puis modèle MG et rocket séparés | Dedicated + 2 clients | `Loop Timer`, `FindNearest`, update target/aim/fire, QWeapon registry, Combat permission |
| `P2-VEHICLE` | même map, idle éveillé, accélération, virage, collision, rollover, underwater | tiers 1/8/32 VehicleBase actifs; même classe et mêmes inputs replayés | Standalone + Dedicated + 2 clients | VehicleBase ReceiveTick/fonctions, BP LoopTimers, native TickComponent, sleep/wake |
| `P2-ITEM` | même map/save fixture, aucune streaming transition | 2 inventaires x 40 slots, 20 records, 100 moves, 10 world-drop round-trips, 1 save + 1 restore | Dedicated + 2 clients | transaction/rollback, DataObject encode/decode, tasks, ProcessEvent, bytes persistés |
| `P2-PROJECTILE-BALLISTIC` | même map, tirs sans puis avec collision | tiers 1/16/64 instances par sous-famille | Standalone + Dedicated | `BP_Projectile ReceiveTick/CheckCollision`; grenade séparée |

### 6.1 Canaux obligatoires

- Commun: `Cpu`, `Frame`, `Stats`, `Bookmark`, `Region`, `Counters`, `Object`, `Log`.
- Async/persistence: ajouter `Task`.
- Réseau: ajouter `Net` sur serveur et clients.
- Client rendu: `Gpu` autorisé mais analysé séparément; serveur en rendu nul.
- Conserver exactement les mêmes canaux avant/après. Activer les named events nécessaires pour obtenir les noms de classes/functions Blueprint, sans modifier seulement une moitié de la paire.

Counters/bookmarks minimaux:

- build/commit, map, seed, topology, role et phase;
- acteurs actifs/dormants par classe, turrets, targets, flares et projectiles vivants;
- transactions tentées/committed/rolled back, records et bytes;
- tirs demandés/acceptés, impacts, damage events, deaths;
- RPC/bytes par direction et save/flush count/duration.

### 6.2 Protocole et critères honnêtes

1. Baseline juste avant la verticale et capture après avec un seul changement P2.
2. Build Development non Editor identique, mêmes options, save, seed, map et scalabilité. Aucun shader compile, streaming initial, ouverture UI ou GC forcé dans la région mesurée.
3. Warmup hors région, phase bookmarkée de durée identique, trois runs minimum par case.
4. Comparer médiane/p95/p99 GameThread/server tick et self time des scopes ciblés. Ne pas additionner des timers inclusifs imbriqués.
5. Exiger les mêmes counts fonctionnels. Tolérance zéro sur transactions/tirs/impacts; maximum `1 %` sur frames/instances échantillonnées.
6. Si le coût disparaît du Blueprint mais réapparaît sous un scope natif, reporter le déplacement et le delta net, pas seulement « scope éliminé ».
7. Une paire invalidée par actor count, topology, event count, canal ou outlier de chargement ne produit aucune revendication de gain.

Attentes structurelles, pas résultats:

- missile: zéro `ReceiveTick` BP du mouvement et zéro `ProjectilesManager::CheckFlaresHit` BP après migration;
- turret: zéro boucle latente/scan nearest BP, coût déplacé vers requête registre QWeapon bornée;
- vehicle: zéro Tick BP gameplay sur serveur; le tick natif physique reste attendu;
- item: zéro ProcessEvent record sur la verticale et mêmes commits/bytes;
- quest: aucun scope à éliminer tant qu'un contrat partagé mesuré n'est pas proposé.

## 7. Candidats de suppression, sans suppression

| Candidat | Asset Registry | Preuve C++/texte | Décision |
|---|---|---|---|
| `/Game/VehicleWeapons/Rocket/Missile_VehicleRocketLauncher` | 0 référent | 0 référence source/config/doc | candidat froid après P2-01; asset local ignoré/non tracked |
| `/Game/_QData/Proto_Quest` | 0 référent | 0 référence source/config/doc; 0 fonction/event | candidat prototype mort; asset local ignoré/non tracked |
| `UDataManagerBPLibrary` (`DataManagerBPLibrary.h/.cpp`) | `/Script/DataManager.DataManagerBPLibrary`: 0 référent | seulement sa propre déclaration/constructeur, aucune fonction | wrapper mort prouvé |
| module runtime `DataManager` actuel | contenu BP ne dépend pas de `/Script/DataManager`; aucun module externe ne le linke | Startup/Shutdown vides | soit le réutiliser pour P2-04, soit convertir le plugin en content-only; ne pas le laisser vide |
| plugin `DataManager` complet | GameDataManager 26, DataObject 72, DataManagerLib 52 référents | consommateurs live QModule/gameplay | ne pas supprimer |
| redirects `DefaultEngine.ini` | preuve Asset Registry insuffisante pour les packages historiques/exclus du dépôt | migrations P0/P1 et Engine encore concernées | aucun redirect candidat prouvé |

Le `ReceiveTick` déconnecté de `BP_Missile` et les events vides des children sont des nettoyages de graphe possibles, pas des assets/plugins à supprimer et pas des gains runtime à compter.

## 8. Inconnues qui bloquent toute revendication

- concurrence réelle et durée de vie par child projectile;
- point de spawn, rôle autoritaire et réplication effective de chaque projectile enfant;
- nombre production de véhicules/tourelles et proportion réellement awake;
- activation effective du chemin C++ FlyVehicle sur chaque classe enfant;
- volume, cadence, bytes et pauses des saves DataObject;
- coût self-time par scope Blueprint P2, absent des exports actuels;
- fréquence réelle des objectifs pickup/shop;
- cold-load/cook validation des trois candidats morts avant toute suppression.

## Conclusion

P2 ne doit pas être un « reste de gros Blueprints ». Après P1, quatre verticales ont une causalité inspectable: missile, tourelle, reliquat gameplay véhicule et record item persistant. Tout le reste reste Blueprint ou passe d'abord par une trace comparable. Aucun résultat runtime n'est revendiqué par cet audit.
