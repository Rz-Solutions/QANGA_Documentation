# QSpatialGrid — Architecture

> Plugin runtime (auteur : Iolacorp) : une **grille spatiale hiérarchique à l'échelle planétaire** (Part → Sector → Cell) indexée par hash entier `int64`, déclinée en **cube** et en **cube-sphère** (6 faces), avec **streaming de données par région** autour du/des joueur(s) et **persistance SQLite** (données statiques authoring + données dynamiques runtime auto-sauvegardées). Fournit aussi un schéma de coordonnées **LWC** (origin-rebasing) pour les positions à l'échelle de millions de km.

---

## 1. But d'usage

QANGA se joue sur des **planètes entières** et dans l'espace : les coordonnées atteignent des millions de km, bien au-delà de la précision `float`, et la quantité de données « accrochées au monde » (POI, état dynamique, sauvegardes de zone) ne peut pas tenir entièrement en mémoire. Il faut donc :

1. **Partitionner l'espace** de façon stable et indexable à l'échelle planétaire — d'où une grille hiérarchique à index entiers.
2. **Streamer** les données par région autour du joueur (charger ce qui est proche, décharger ce qui s'éloigne, avec hystérésis pour éviter le thrashing).
3. **Persister** ces données : un volet **statique** (authoring, lecture seule) et un volet **dynamique** (mutations runtime, sauvegardées périodiquement) — le tout en base **SQLite**.
4. **Rebaser l'origine** des coordonnées pour rester précis (`FQSG_ReplicationGrid_Coord` : grille 1000 km + sous-grille 20 km + offset float local).

Ce n'est ni `WorldScape` (génération/maillage du terrain) ni l'octree de `QLevel` : `QSpatialGrid` est la **grille de données** — un index spatial adossé à une base de données.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Hiérarchie à 3 niveaux : Part > Sector > Cell.** Chaque grille définit trois profondeurs de subdivision (`Part_Depth = 2`, `Sector_Depth = 6`, `Cell_Depth = 8` par défaut). On charge/décharge indépendamment au niveau Part, Sector et Cell.
- **Index entiers `int64`, pas de coordonnées flottantes.** Les coordonnées de grille sont des vecteurs entiers hashés vers un `int64` (`FQSG_Vector_Int16::ToHashint64` — **sans collision** pour des cellules de 1000 km sur ±32768, soit ~65 millions de km). Des variantes existent (`FQSG_Vector_Int32`/`Int64` avec mélange de bits, collisions possibles, documentées comme telles dans le code).
- **Deux topologies.** `EQSG_Grid_Type::Cube` (grille cubique classique) et `Sphere` (cube-sphère à 6 faces, `FQSG_GridSphereCoord` avec `FaceIndex` + X/Y, adjacence inter-faces via `QSG_AdjacentFacesLookup`). La sphère permet une grille qui épouse une planète.
- **Acteurs = définition de zones, Subsystem = moteur.** Un `AQSG_SpatialGrid_Actor` (abstrait, décliné Cube/Sphere) **pose une grille** dans le monde (nom unique, priorité, profondeurs, rayon planétaire, échelles de préchargement). Le `UQSG_SubSystem` (WorldSubsystem) enregistre ces acteurs, crée un `UQSG_Runtime_Data` par grille et pilote le streaming.
- **Streaming à deux fréquences.** Le subsystem enregistre **deux ticks custom** (`FCustomTickFunction`, même motif que `QInstanced`) : `HighFrequencyUpdate` (`HighFrequency = 0.1 s`, position/chargement proche) et `LowFrequencyUpdate` (`LowFrequency = 0.5 s`, déchargement/maintenance). Le déchargement est temporisé par `FQSG_DynamicLock` (hystérésis `UnLoadTime`/`MaxUnLoadTime` + verrous de référence).
- **Persistance SQLite, statique + dynamique.** Les données vivent en base via `UQSQL_DB_Object` (plugin `QSQL_Interface` + `SQLiteCore`). Le volet **dynamique** (mutations de jeu) est sauvegardé dans une base `DynamicDB`, avec **auto-backup rotatif** (`AutoBackup_Time = 600 s`, `AutoBackup_Number = 3`). Les données par cellule sont des `UObject` arbitraires sérialisés binairement (`FQSG_Data_Objects` + son `operator<<` qui (re)crée les objets par chemin de classe).
- **Coordonnées LWC découplées.** `FQSG_ReplicationGrid_Coord` (grille 48 bits + sous-grille 24 bits) + un offset `FQSG_Vector_Float32` local reconstituent une `FVector` monde — c'est le schéma d'origin-rebasing pour rester précis à l'échelle planétaire (`QSG_Replication_Grid_WorldToCoord` / `…CoordToWorld`).

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQSG_SubSystem` | `UWorldSubsystem` | Moteur : enregistre les grilles, crée les runtime data, ticks haute/basse fréquence, DB dynamique + backup, helpers de coordonnées |
| `UQSG_Runtime_Data` | `UObject` | État vivant d'une grille : sets chargés (Part/Sector/Cell), verrous, load/unload par liste, conversions location↔index, adjacence |
| `AQSG_SpatialGrid_Actor` | `AActor` (`Abstract`) | Définition d'une zone de grille (nom, priorité, profondeurs, paramètres cube/sphère) |
| `AQSG_CubeGrid_Actor` | `AQSG_SpatialGrid_Actor` | Grille cubique (`UBoxComponent` Size/PreLoad) |
| `AQSG_SphereGrid_Actor` | `AQSG_SpatialGrid_Actor` | Grille cube-sphère (`USphereComponent` Size/PreLoad/Part/Sector) |
| `UQSG_Settings` | `UDeveloperSettings` (`config=Game`) | Réglages projet (catégorie `QSpatialGrid`) : activation, version, auto-backup |
| `UQSG_FunctionLib` | `UBlueprintFunctionLibrary` | Helpers de conversion index ↔ coordonnées / adjacence |
| `FQSG_DataState` | `USTRUCT` | Sets `int64` des Cell/Sector/Part chargés + flag `Storage` |
| `FQSG_DynamicLock` | `USTRUCT` | Hystérésis de déchargement (compteur de verrous + timers) |
| `FQSG_Data_Objects` | `USTRUCT` | Conteneur `TMap<FName,UObject*>` à sérialisation binaire custom |
| `FQSG_Vector_Int16` / `Int32` / `Int64` / `int8` | `USTRUCT` | Coordonnées entières ↔ hash `int64` (précision/collision documentées) |
| `FQSG_GridSphereCoord` | `USTRUCT` | Coordonnée cube-sphère (face + X/Y) ↔ index |
| `FQSG_ReplicationGrid_Coord` | `USTRUCT` | Coordonnée LWC (grille 1000 km + sous-grille 20 km) ↔ `FVector` |
| `EQSG_Grid_Type` | `UENUM` | `Cube` / `Sphere` |
| `EQSG_ReturnState` | `UENUM` | États de chargement (`Loaded`, `OnLoad`, `OnPendingSave`…) |

**Délégués (subsystem)** : `QSG_NewRuntimeDate(UQSG_Runtime_Data*)`, `QSG_RemoveRuntimeDate(UQSG_Runtime_Data*)`.

---

## 4. Flux de données et cycle de vie

1. **Enregistrement** — au `BeginPlay`, chaque `AQSG_SpatialGrid_Actor` posé appelle `QSG_Register_Grid` (ou le subsystem les découvre). Le subsystem crée un `UQSG_Runtime_Data` par grille (`Create_RuntimeData` → `Init_Runtime_Data`), copie la config de l'acteur (`CopyToRuntimeData`), diffuse `QSG_NewRuntimeDate`. Si le subsystem n'est pas prêt, la création est mise en file (`Pending_Runtime` / `MakePending`).
2. **Init DB** — la base dynamique (`Dynamic_World_DB_Object`, `UQSQL_DB_Object`) est ouverte/initialisée (`Init_DynamicData`), restaurée depuis un éventuel save (`QSettings`).
3. **Localisation** — `Update_Location` récupère la (ou les) position(s) joueur (`Get_PlayerLocations`, ou positions custom). Sur dedicated server, le contexte est marqué (`IsDedicatedServer`).
4. **Streaming haute fréquence** (`0.1 s`) — pour chaque grille, `Auto_ByLocations(Locations)` calcule les Cell/Sector/Part à charger autour des joueurs (`Grid_Get_IndexMultiLocation`) et déclenche `Grid_Auto_Load_ByList`. Les données sont lues depuis la DB et désérialisées en `UObject` (`FQSG_Data_Objects`).
5. **Streaming basse fréquence** (`0.5 s`) — déchargement des régions hors zone (`Grid_Auto_UnLoad_byList`), arbitré par `FQSG_DynamicLock` (une région verrouillée ou récemment quittée n'est pas déchargée immédiatement). Les données dynamiques modifiées sont renvoyées vers la DB (`Send_Dynamic_DB`).
6. **Backup** — périodiquement (`AutoBackup_Time`, si `Use_AutoBackup_DynamicData`), `Backup_Dynamic_DB` effectue une sauvegarde rotative (`AutoBackup_Number` copies).
7. **Fermeture** — `Request_Close` / `Request_Close_Finich` sur les runtime data ; `Deinitialize` du subsystem ; selon `Request_Destroy_RuntimeData_OnDestroy`, la runtime data est seulement déchargée ou détruite.

---

## 5. Réplication / réseau

Le plugin **ne réplique pas d'état réseau** au sens UE (pas de `DOREPLIFETIME`/RPC) : c'est un système de partitionnement et de persistance **local à chaque instance** (serveur ou client), conscient du contexte serveur dédié (`IsDedicatedServer`).

Le préfixe « Replication » de `FQSG_ReplicationGrid_Coord` désigne un **schéma de coordonnées** (grille 1000 km + sous-grille 20 km + offset float) destiné à transmettre/stocker des positions planétaires de façon compacte et précise (origin-rebasing LWC), **pas** un canal de réplication d'acteurs. Les helpers `QSG_Replication_Grid_WorldToCoord` / `…CoordToWorld` convertissent entre `FVector` monde et cette coordonnée.

---

## 6. Points d'intégration

- **Dépendances module** : `Core`, `QSQL_Interface`, `SQLiteCore`, `QSettings`, `DeveloperSettings` (public) ; `CoreUObject`, `Engine`, `Slate`, `SlateCore` (privé). Plugins : `SQLiteCore`, `QSQL_Interface`, `QSettings`.
- **QSQL_Interface / SQLiteCore** : couche de persistance (`UQSQL_DB_Object`) — la DB statique (authoring) et la DB dynamique (`DynamicDB`).
- **QSettings** : sauvegarde/restauration de l'état des données dynamiques entre sessions.
- **Consommateurs** : tout système qui doit accrocher des données à des régions de l'espace planétaire (POI, état de monde persistant, sauvegardes de zone). Les données stockées sont des `UObject` projet (via `FQSG_Data_Objects`).
- **Réglages** : *Project Settings → QSpatialGrid* (`UQSG_Settings`) — activation, tag de version, paramètres d'auto-backup.

---

## 7. Gotchas, invariants et pièges

- **Choisir le bon encodage entier.** `FQSG_Vector_Int16` est **sans collision** (recommandé pour cellules de 1000 km) ; `FQSG_Vector_Int32`/`Int64` utilisent un mélange de bits avec **collisions possibles** (le code le signale explicitement). Indexer une grille avec un encodage à collision corrompt le lookup.
- **Sérialisation binaire des données = couplage aux classes.** `FQSG_Data_Objects::operator<<` retrouve la classe par **chemin** (`GetPathName`) et recrée l'objet ; renommer/déplacer une classe de données stockée casse la désérialisation des saves existantes.
- **Hystérésis de déchargement.** Le déchargement passe par `FQSG_DynamicLock` (timers + verrous) : une région n'est pas déchargée à la frame où le joueur la quitte. Ne pas supposer un déchargement immédiat ; tenir compte de `UnLoadTime`/`MaxUnLoadTime`.
- **Auto-backup rotatif.** La DB dynamique est sauvegardée toutes les `AutoBackup_Time` (600 s) en `AutoBackup_Number` (3) copies — vérifier l'espace disque côté serveur et la cohérence de `Dynamic_DB_Name`.
- **Initialisation différée.** Si une grille s'enregistre avant que le subsystem soit prêt, elle passe par `Pending_Runtime` ; ne pas s'attendre à une runtime data immédiatement après `QSG_Register_Grid`.
- **Profondeurs = coût mémoire/IO.** Augmenter `Cell_Depth`/`Sector_Depth`/`Part_Depth` densifie la grille (plus d'index, plus d'entrées DB). Les valeurs par défaut (2/6/8) et les rayons (planète ~6371 km) sont calibrés ; les changer impacte le streaming et la base.
- **Distinct de WorldScape et QLevel.** C'est la grille de **données** (DB spatiale), pas le terrain ni l'octree d'acteurs — ne pas confondre les responsabilités.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QSpatialGrid/Public/QSG_SubSystem.h` / `Private/…cpp` | Moteur : enregistrement, runtime data, ticks, DB dynamique + backup, helpers coord |
| `Source/QSpatialGrid/Public/Data/QSG_Runtime_Data.h` / `…cpp` | État vivant d'une grille : load/unload, conversions, adjacence |
| `Source/QSpatialGrid/Public/Actor/QSG_SpatialGrid_Actor.h` / `…cpp` | Acteur abstrait : définition d'une zone de grille |
| `Source/QSpatialGrid/Public/Actor/QSG_CubeGrid_Actor.h` / `QSG_SphereGrid_Actor.h` | Variantes cube / cube-sphère |
| `Source/QSpatialGrid/Public/QSG_Struct.h` | Coordonnées entières/sphère/LWC, hash `int64`, sérialisation données, lookups d'adjacence, enums |
| `Source/QSpatialGrid/Public/QSG_FunctionLib.h` / `…cpp` | Helpers de conversion / adjacence |
| `Source/QSpatialGrid/Public/QSG_Settings.h` / `…cpp` | `UDeveloperSettings` (auto-backup, version) |
| `QSpatialGrid.uplugin` | Descripteur : Runtime, plugins `SQLiteCore` + `QSQL_Interface` + `QSettings` |
