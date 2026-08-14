# QMODULE_LOOT_ARCHITECTURE

Distribution procédurale des modules QModule dans l'univers persistant.

> Statut : document d'architecture, **en attente de validation**. Aucune ligne de code
> ni aucun asset n'a été écrit au moment de la rédaction. Toutes les valeurs chiffrées
> de la section 2 ont été **mesurées dans l'éditeur** le 2026-08-07, pas estimées.
>
> Ce document complète `QMODULE_ARCHITECTURE.md`, dont le jalon M8 avait explicitement
> renvoyé « les marchands de modules et le PLACEMENT du loot » à un chantier ultérieur
> porté par « les quêtes, le loot procédural des IA et les QLevels répartis dans
> l'univers ». C'est ce chantier.

---

## 1. Objet et périmètre

**Objet.** Faire apparaître les modules cyborg dans le monde, de façon procédurale,
cohérente avec le lieu, déterministe, persistante, et sans éditer les niveaux à la main.

**Périmètre v1 (tranché par RzZz le 2026-08-07) :** les **12 modules cyborg qui
possèdent déjà un item**. Sont explicitement hors v1 :

- les phases `T1` à `T6` (elles ont un item, mais leur courbe d'obtention touche à
  l'économie de progression : chantier séparé) ;
- les modules arme et véhicule (leur `GameShopPrice` vaut 0, à corriger d'abord) ;
- les 94 QMD du catalogue complet, dont l'audit interne du 2026-07-27 compte
  **59 coquilles vides** sans aucun effet. Les distribuer produirait du loot qui ne
  fait rien.

**Hors périmètre, définitivement :** les marchands de modules (infrastructure de shop
existante, chantier propre), les récompenses de quêtes (DQS), le craft.

---

## 2. État vérifié de l'existant

Cette section est le socle factuel. Chaque affirmation a été mesurée.

### 2.1 Comment les items sont posés aujourd'hui

**618 maps** référencent des assets de `/Game/Items/`, pour **595 items distincts**.
Ce sont des acteurs `IS_*` (enfants d'`ItemScriptBase`) **posés à la main**, un par un.
Le gros du volume est dans `Content/Maps/DevMap/ConstructionLevel/` (les intérieurs de
bâtiments) et `Content/Maps/Universe/Sub_Levels/Earth_Levels/DamagedCity/` (jusqu'à
277 items distincts dans une seule maison).

**920 `UQLevel_Asset`** existent. **288 `.umap` contiennent eux-mêmes un
`AQLevel_Actor_Instance`** : l'imbrication récursive bâtiment vers intérieur est la
norme, pas un cas particulier.

### 2.2 Le modèle réseau : le fait structurant

Mesuré sur les CDO :

| Classe | `bReplicates` | `NetCullDistanceSquared` |
|---|---|---|
| `ItemScriptBase` | **False** | 9e8 (300 m) |
| `IS_CanOfBeans` | **False** | 9e8 |
| `IS_QModuleCy_ChasseurDePrimes` | **False** | 9e8 |
| `SeededLootActor`, `RandomLootActor` | **False** | 2.25e8 (150 m, défaut moteur) |

**Aucun item du monde ne réplique.** Chaque machine possède sa propre copie :

- les items posés en niveau arrivent par le niveau lui-même (`bNetLoadOnClient=True`) ;
- les items spawnés au runtime sont recopiés localement par le mécanisme
  « forced actor » de `UOptimizedStateManager` (plugin QNetState) : `ForcedSpawnActors`
  est une liste répliquée d'acteurs **non répliquants** que les clients doivent
  mirror-spawner, avec destruction groupée par un unique
  `MC_ForcedActorsDestroy(TArray<FVector>)`.

La clé de voûte : **`FOptimizedStateKey` est une position monde quantifiée en `int64`,
utilisée comme clé de `TMap`**. Tout l'état partagé (ramassé, caché, temps de respawn)
est **adressé par position**, jamais par identité d'acteur. C'est ce qui rend le
système tenable à 500 joueurs : zéro canal de réplication par item.

La réplication n'est activée que dynamiquement, quand un item devient possédé par un
pawn (commentaire dans `ItemScriptBase` : « if item is owned by pawn, so turn
replication on »).

**Conséquence pour ce chantier :** la distribution des modules ne doit créer aucun
acteur répliqué. Elle doit produire, sur chaque machine, le même résultat à partir
d'une graine déterministe, et ne faire voyager que l'état.

### 2.3 Le respawn persistant existe déjà de bout en bout

`ItemsManagerGS.AddItemPickedToRespawn(CléDePosition, Délai)` :

1. `TargetTime = GetCurrentGlobalTimeSeconds() + Délai` (horloge serveur globale, pas
   un timer de session) ;
2. écriture dans la map `Loot Picked: Target Time Respawn`, **clé = position de l'item** ;
3. encodage puis **`SaveItemsPicked`** vers le `GameDataManager`, donc la base.

`IsItemPickedWaitingRespawn` relit et compare. **La règle « cycle long partagé par le
serveur », retenue pour ce chantier, est donc déjà implémentée et survit aux
redémarrages.** Il ne reste qu'à choisir la valeur du délai pour les modules.

### 2.4 Le framework de loot : construit, jamais déployé

`Content/Systems/Item/Loot/` contient :

- **`DA_Loot`** (PrimaryDataAsset BP) : `Item:DropWeight` (map IDA vers poids),
  `ItemAlwaysDropped`, `SelfWearingItemsPossibility`, `DurationOnGround`,
  `RespawnTimeSeconds`.
- **`SpawnLootComponent`** (SceneComponent), le moteur réel :
  `GetSeededItemByLocation` calcule
  `Seed = Truncate(Dot(WorldLocation, (0.484, 99.839996, -0.470)))`, alimente un
  `FRandomStream`, tire dans la table, spawne en différé la classe `ItemScript` de
  l'IDA, pose `RespawnTimeSeconds` sur l'acteur et bind `OnPicked`.
- **`SeededLootActor`**, **`RandomLootActor`**, **`LootSlotComponent`** : des acteurs
  multi-emplacements construits par-dessus.

Deux constats :

1. **Le tirage ignore les poids, SUR CE CANAL UNIQUEMENT.** `GetSeededItemByLocation`
   prend `Keys(Item:DropWeight)` puis fait un `RandomIntegerInRange` **uniforme** sur
   l'index. Précision (investigation du 2026-08-14) : le canal de mort des IA, lui,
   lit bien la colonne, mais avec une **autre sémantique** :
   `CombatComponent.GetRandomDrop` mélange les clés puis fait un jet de Bernoulli
   `RandomBoolWithWeight(poids)` par clé et retourne la première qui passe. La valeur
   est donc une **probabilité par item** (0..1), pas un poids relatif, un seul item
   au maximum tombe, et il est possible que rien ne tombe. `ItemAlwaysDropped` est
   garanti, `RespawnTimeSeconds` n'est pas consommé par ce chemin (seul
   `DurationOnGround` l'est), et le spawn passe par `ItemDroppedReplicator`.
   La colonne de poids n'a en revanche aucun effet sur ce
   chemin. Le C++ de QModule, lui, la lit correctement :
   `AQModule_SupplyCrateActor::Authority_RollSupplyItem` (patron réflectif
   `FMapProperty` / `FScriptMapHelper`, gère les clés dures et souples, les valeurs
   `float` et `double`).
2. **Rien n'est déployé.** `SeededLootActor` et `RandomLootActor` ne sont référencés
   que dans trois maps de développement (`L_Dev_Alcesteiro`, `TestLootRandom`,
   `L_ContractsTest`). Les tables `Brocante_LootActor`, `Food_LootActor`,
   `Medicine_LootActor` n'ont aucun référent hors graine de cook.

### 2.5 Le gisement : 51 517 ancres déjà sur disque

DQS a scanné les sous-niveaux et stocké le résultat **dans les QLevel assets
eux-mêmes** : `UQLevel_Asset.AdditionalData["InteractableMeshes"]`, un
`FInstancedStruct` de type `FQLevelInteractableData`.

Mesure du 2026-08-07 sur les 920 QLevel :

| | |
|---|---|
| QLevel porteurs d'un catalogue | **419** |
| Ancres totales | **51 517** |
| CONTAINER | 21 429 |
| FURNITURE | 9 013 |
| MACHINERY | 8 164 |
| GENERIC | 7 876 |
| TERMINAL | 3 301 |
| SWITCH | 1 734 |

Chaque entrée porte le nom du mesh, le nom de l'acteur, la catégorie, l'extent, et la
**position et rotation en coordonnées locales du sous-niveau** : le scan charge la map
source seule (`LevelMeshScanner.cpp`, `PopulateSingleQLevelWithInteractableData`). La
transform de l'instance QLevel les convertit en coordonnées monde au runtime. C'est
exactement la propriété dont ce chantier a besoin : une carte réutilisable quelle que
soit la position planétaire du bâtiment.

Le régénérateur existe : `ULevelMeshScanner::PopulateAllQLevelsWithInteractableData()`
(`CallInEditor`). **Piège** : il charge les 920 niveaux sources en synchrone, ce qui a
fait crasher l'éditeur pendant l'étude
(`FDistanceFieldVolumeData::CacheDerivedData`, thread de build de mesh). Toute passe de
rescan doit se faire par lots.

Les **501 QLevel sans catalogue ont tous un niveau source** et ne sont pas des
variantes optimisées : ils n'ont jamais été scannés, ou le scan n'a rien trouvé. Les
deux cas sont indiscernables sans relancer, car le code ne stocke rien quand il trouve
zéro mesh.

### 2.6 Les sept canaux de spawn d'items de l'univers

1. items posés à la main dans les sous-niveaux (618 maps) ;
2. loot de mort des IA : variable `ItemDropLoot` (un `DA_Loot`) sur les BP IA
   (Sangline, Voss, Infected, Pirate, Crocodile, SandDigger, GazSac, Boss), plus
   `DropAllItemsDeath` et `ListToDrop` ;
3. boutiques (`BPI_Shop`, `BP_Shop`, `NPC_Shop`, restock par temps) ;
4. caisse de ravitaillement QModule (`AQModule_SupplyCrateActor`) ;
5. minage (`RecyclerComponent` sur `MiningActorBase`) ;
6. récompenses de quêtes et de contrats (DQS) ;
7. drop au sol du joueur et à la mort (`ItemsManagerGS`).

Ce chantier attaque le canal 1 (couverture de masse) et prépare le branchement des
canaux 2 et 4, qui consomment le même format de table.

### 2.7 Les items module sont prêts

Les 21 items module (12 cyborg, 6 phases, 2 arme, 1 véhicule) ont tous une classe
`ItemScript` de ramassage, `AvailableAdminSpawn=true`, et sont **tous inscrits dans
`DA_AllRef.ItemKey:DAItem`** (301 clés au total). Vérifié un par un : le piège numéro
un des items QANGA, l'oubli d'inscription qui fait disparaître l'item au drop, n'est
pas présent.

Attention au nommage : les clés des modules cyborg sont `QModCy_*`, pas `QModule*`.
Un filtre sur la chaîne « QModule » les rate toutes.

---

## 3. Décisions actées

| Sujet | Décision | Date |
|---|---|---|
| Périmètre v1 | Les 12 modules cyborg qui ont un item | 2026-08-07, RzZz |
| Respawn | Cycle long partagé par le serveur | 2026-08-07, RzZz |
| Modèle réseau | Aucun acteur répliqué, déterminisme plus état par position | 2026-08-07, mesuré |
| Format de table | `DA_Loot` existant, pas de nouveau format | 2026-08-07 |

---

## 4. Architecture cible

### 4.1 Vue d'ensemble

```
UQLevel_SubSystem.QLevel_LevelLoad  (délégué existant, déjà utilisé par DQS)
        |
        v
UQModuleLoot_World_SubSystem                       <-- à créer
  1. lit AdditionalData["InteractableMeshes"] du QLevel qui vient de charger
  2. résout le PROFIL DE LIEU du QLevel                       (couche données)
  3. graine déterministe = hash64(position monde de l'instance)
  4. filtre les ancres par catégorie selon le profil
  5. tire K ancres, puis un module par ancre dans la table pondérée
  6. interroge ItemsManagerGS : cette position est-elle en attente de respawn ?
  7. spawne l'acteur ItemScript du module, à la position locale transformée en monde
        |
        v
ItemScriptBase (existant)   ->  ramassage  ->  AddItemPickedToRespawn(position, délai long)
        |                                              |
        v                                              v
OptimizedStateComponent (état par position)      GameDataManager / base (persistance)
```

### 4.2 Couche données : le profil de lieu

Un `UQModuleLoot_ProfileDataAsset` (nouveau, plugin QModule) décrit un archétype de
lieu :

| Champ | Rôle |
|---|---|
| `ProfileTag` | identité du profil (`Loot.Profile.Bunker`, `Loot.Profile.Relay`...) |
| `LevelPathPrefixes` | préfixes de chemin de QLevel qui retombent sur ce profil |
| `AllowedCategories` | quelles catégories d'ancres sont éligibles, avec un poids par catégorie |
| `ModuleTable` | un `DA_Loot` listant les modules et leurs poids |
| `SpawnChancePerAnchor` | probabilité qu'une ancre éligible porte un module |
| `MaxModulesPerLevel` | plafond dur, garde-fou anti-inondation |
| `RespawnSeconds` | délai passé à `AddItemPickedToRespawn` |
| `RequiredManufacturer` | filtre facultatif (les modules IC Lab chez IC Lab) |

Un `UQModuleLoot_Settings` (`UDeveloperSettings`, section dans `DefaultGame.ini`)
porte : `Enabled` (défaut **false**, comme QModule), la liste des profils, un
multiplicateur global de densité, et les CVars de debug.

**Choix assumé :** on réutilise `DA_Loot` pour la table d'items plutôt que d'inventer
un format. C'est déjà la table du projet, QModule sait déjà la lire pondérée, et cela
rend les mêmes tables consommables par les canaux IA et caisse sans conversion.

### 4.3 Couche placement : le subsystem

`UQModuleLoot_World_SubSystem` (`UWorldSubsystem`, plugin QModule) :

- s'abonne à `UQLevel_SubSystem::QLevel_LevelLoad` et `QLevel_LevelUnLoad` ;
- **ne tourne que si `Enabled`** et ne fait rien sur les mondes sans QLevel ;
- s'enregistre auprès de QGameManager selon le patron déjà utilisé par le registre
  QModule.

Coût : nul tant que personne n'est dans le secteur, puisque tout est piloté par le
streaming. Aucun tick, aucun poll : on se branche sur la source, conformément à la
règle 5 du CLAUDE.md.

### 4.4 Déterminisme : attention à la précision planétaire

Le seed doit produire le même résultat sur le serveur et sur chaque client, sinon les
copies locales divergent.

**Risque identifié dans l'existant.** La formule de `GetSeededItemByLocation` est
`Truncate(Dot(WorldLocation, (0.484, 99.839996, -0.470)))`. `Truncate` renvoie un
`int32`. Or les acteurs QModule ont été mesurés à **5,19e7 unités de l'origine** lors
de la passe multijoueur du 2026-07-29. Sur cette seule composante, `5,19e7 x 99,84`
vaut environ **5,2e9**, très au-delà de la capacité d'un `int32` (2,15e9). À l'échelle
planétaire, ce seed sature donc ou se replie. Le projet a déjà tiré cette leçon
ailleurs : `FOptimizedStateKey` est en `int64` précisément pour cette raison.

**Décision :** le nouveau code n'utilise pas cette formule. Il utilise un hash 64 bits
sur la position quantifiée, de la même façon que `FOptimizedStateKey`, ce qui a en
prime l'avantage d'aligner la graine de placement et la clé d'état sur la même
adresse.

Le tirage du module lui-même passe par le patron réflectif pondéré déjà éprouvé de
`Authority_RollSupplyItem`, factorisé pour être appelable par les deux acteurs.

### 4.5 Ce que l'on ne fait pas, et pourquoi

- **On n'édite aucune des 618 maps.** Le catalogue d'ancres rend cela inutile, et une
  édition de masse de niveaux World Partition est le pire risque de régression du projet.
- **On ne pose pas de `SeededLootActor` à la main.** Même raison, et cet acteur
  hérite du cull par défaut à 150 m.
- **On ne touche pas aux items posés à la main existants.** Ils restent la couche
  artisanale ; la couche procédurale s'ajoute par-dessus.
- **On ne corrige pas `GetSeededItemByLocation` dans le cadre de ce lot.** C'est un
  contrat consommé par trois BP ; sa correction est une tâche dédiée (section 8).

---

## 5. Les lieux couverts

### 5.1 Critère de sélection

Un QLevel est retenu s'il satisfait **les deux** conditions :

1. il porte un catalogue d'ancres (`AdditionalData["InteractableMeshes"]` non vide) ;
2. il est **atteignable depuis `/Game/Maps/Universe/L_Persistent_Universe`** par le
   graphe de dépendances (parcours mesuré : 988 maps visitées, 403 QLevel atteints).

Le second critère est ce qui distingue un bâtiment réellement présent dans l'univers
d'un bâtiment de bibliothèque ou d'archive.

**Résultat : 241 QLevel retenus, 29 002 ancres, dont 14 257 conteneurs.**

### 5.2 Répartition des lieux retenus

| Archétype QLevel | QLevel | Ancres | Conteneurs | Mobilier | Terminaux |
|---|---:|---:|---:|---:|---:|
| `Maps/DevMap` (intérieurs de bâtiments) | 30 | 9 181 | 3 775 | 1 420 | 1 199 |
| `Planetary/Relay` | 78 | 6 092 | 2 450 | 947 | 425 |
| `Planetary/Abandonned` | 83 | 5 747 | 4 468 | 785 | 176 |
| `Maps/Universe` | 24 | 4 061 | 2 214 | 671 | 73 |
| `Space/Warp` | 10 | 2 239 | 287 | 673 | 191 |
| `Planetary/Camp` | 4 | 1 064 | 742 | 99 | 29 |
| `Planetary/ICLAB` | 7 | 431 | 237 | 54 | 15 |
| `Planetary/Outpost` | 1 | 157 | 58 | 27 | 16 |
| `Planetary/Rebel` | 1 | 12 | 12 | 0 | 0 |
| `Planetary/Human` | 1 | 9 | 9 | 0 | 0 |
| `Maps/CompositeUniverse` | 1 | 6 | 4 | 0 | 0 |
| `Planetary/Ocean` | 1 | 3 | 1 | 0 | 0 |
| **Total** | **241** | **29 002** | **14 257** | **3 676** | **2 124** |

### 5.3 Les cinq profils de lieu, et leurs niveaux

**P1. Bunker et camp hostile** (prototype)

`Planetary/Camp` : `Q_L_Camp_Bunker_Sangline_A1`, `Q_L_Camp_Bunker_Sangline_A2`,
`Q_L_Camp_Bunker_RebelCyborg_A1`, `Q_L_Camp_Bunker_RebelCyborg_A2`.
Renforts `Maps/DevMap` : `Q_L_Camp_Bunker_RebelCyborg_C1`,
`Q_L_RebelCampCyborg1WithActor`, `Q_L_Camp_Underground_Humain_A1`.
Environ 7 niveaux, 1 900 ancres, 1 200 conteneurs. Petits, fermés, hostiles,
lisibles : c'est le terrain de test idéal.

**P2. Relais et stations IC Lab**

`Planetary/Relay` (78) et `Planetary/ICLAB` (7), plus les relais de `Maps/DevMap` :
`Q_L_Relay_Planetary_A1`, `A2`, `PVE_B1`, `PVE_B2`, `Q_L_RelayPlanetaryGarage`,
`Q_L_MoonStation_A1`, `Q_L_RelayPlanetary_Cyborg_A1`, `Q_L_Sky_RelayPlanetary`.
Pièces phares : `Q_MoonICLABStation_MegaBlock_A2` (509 conteneurs),
`Q_EarthICLABRelayA_IntA_GunShop` (219), `Q_Relay_Tower_Djibouti_Weapon_Store` (219),
`Q_EarthICLABRelayA_Int_Control_Center` (90 terminaux).
C'est le foyer naturel des modules `Manufacturer.ICLab`, soit **8 des 12** du périmètre.

**P3. Ruine et bidonville**

`Planetary/Abandonned` (83) et la partie ruines de `Maps/Universe` :
`Q_IronCity_Slums_06/09/12`, `Q_Slum_07_Solo_0/1`, `Q_L_Env_Djibouti_*`,
`Q_L_Djibouti_Big_City_*_Detail`, `Q_L_AbandonStore`, `Q_L_AbandonedSquare_02`,
`Q_L_Abandoned_Garage_1`, `Q_L_OldTown_Maison_*`.
C'est l'archétype le plus dense en conteneurs (4 468) : la fantaisie de fouille.
Modules courants, rareté basse, densité forte.

**P4. Marché et station warp**

`Space/Warp` (10) : `Q_LS_SpaceWarp_Market_V2`, `Q_L_SpaceWarp_Market_Infected`,
`Q_LS_SpaceWarp_Top_V2`, `Q_LS_SpaceWarp_V2`, `Q_L_SpaceWarp_Infected`,
`Q_L_WarpShop01` à `04`.
Dominé par le mobilier (673) plutôt que les conteneurs (287) : profil à faible densité
mais rareté plus haute, cohérent avec un lieu de passage marchand.

**P5. Divers et industriel**

`Q_L_Weapons_Factory_A1` (310 conteneurs), `Q_L_StarkitownProd` (326),
`Q_L_Moon_CREATION` (687), `Q_L_MarsOne` (169), `Q_L_Outpost_Cyborg_A1`,
`Q_Rebel_Camp_Composite_Tiny_01`, `Q_Oil_Platform_A1`.

### 5.4 Lieux volontairement écartés, et pourquoi

**178 QLevel portent un catalogue mais ne sont atteints par aucun chemin partant de
l'univers persistant :**

| Groupe écarté | QLevel | Ancres | Motif |
|---|---:|---:|---|
| `Planetary/YellowWall_OLD` | 104 | 9 124 | 327 QLevel dans ce dossier, hébergés par 2 maps seulement, toutes deux internes au dossier. Aucune map extérieure ne les atteint. **À confirmer : dossier mort ?** |
| `Maps/DevMap` non atteints | 15 | 5 877 | maps de construction non assemblées |
| `Qasset/AssetStore/Hospital_Meshingun` | 2 | 2 256 | contenu marketplace, **0 map hôte** |
| `Planetary/Relay` non atteints | 16 | 1 603 | variantes non assemblées |
| `Planetary/Human` non atteints | 10 | 1 359 | appartements et maisons non assemblés |
| `Planetary/YellowWall` | 9 | 1 162 | même situation que YellowWall_OLD. **À confirmer.** |
| autres | 22 | 1 134 | |

**Point d'attention pour RzZz :** `Q_IronCity_Test` est retenu par le critère (il est
atteignable) et pèse **2 771 conteneurs à lui seul, soit 19 % de tous les conteneurs
vivants**. Son nom dit « Test ». S'il s'agit d'un niveau de travail, il faut l'exclure,
sans quoi il écrase à lui seul l'équilibre de la distribution.

**162 QLevel sont atteignables mais n'ont pas de catalogue.** Une passe de rescan par
lots dirait lesquels contiennent réellement du mobilier. C'est le gisement d'extension
naturel après la v1.

---

## 6. Rareté et répartition des 12 modules

État mesuré des définitions :

| Module | Fabricant | Rarity | Prix | Profil naturel |
|---|---|---:|---:|---|
| `AntenneLonguePortee` | ICLab | 1 | 2500 | P2 relais |
| `BlindageSousCutane` | ICLab | **0** | 2500 | P1, P3 |
| `BrouilleurDeSignature` | ICLab | **0** | 2500 | P2 |
| `ChasseurDePrimes` | ICLab | **0** | 2500 | P2, P4 |
| `DecodeurQPD` | ICLab | **0** | 2500 | P2 |
| `FleauDesMachines` | ICLab | **0** | 2500 | P5 industriel |
| `RecuperateurCinetique` | ICLab | **0** | 2500 | P1, P3 |
| `SacocheDimensionnelle` | ICLab | **0** | 2500 | P3 |
| `CoqueIntegrale` | **Voss** | **0** | 2500 | aucun : voir ci-dessous |
| `ContreMesuresVoss` | *(vide)* | 2 | 4000 | P2 |
| `MasquePheromonal` | *(vide)* | 2 | 4000 | P3, P4 |
| `NidDeFrelons` | *(vide)* | 3 | 5000 | P4, boss |

Trois anomalies de données à traiter avant l'équilibrage :

1. **Huit modules sur douze ont `Rarity = 0`**, ce qui rend le champ inutilisable comme
   axe de tirage. Le prix, lui, est déjà cohérent et concorde partout où la rareté est
   renseignée (rareté 2 vaut 4000, rareté 3 vaut 5000). **Proposition : remplir
   `Rarity` à partir des bandes de prix existantes**, ce qui est une passe de données
   petite et vérifiable, et non un travail de design.
2. **`ContreMesuresVoss` n'a aucun `ManufacturerTag`** alors qu'il est thématiquement
   marqué. À trancher : IC Lab (contre-mesure fabriquée par IC Lab) ou Voss.
3. **`CoqueIntegrale` est `Manufacturer.Voss`, mais aucun archétype de lieu Voss
   n'existe dans l'univers atteignable.** Ce module n'a donc pas de foyer bâti : sa
   source naturelle est le loot des IA Voss, c'est-à-dire le canal 2, pas ce lot.

---

## 7. Plan par lots

| Lot | Contenu | Livrable vérifiable |
|---|---|---|
| **L0** | Passe de données : remplir `Rarity` des 12, trancher les 2 anomalies de fabricant | 12 QMD relus, diff justifié |
| **L1** | `UQModuleLoot_Settings` et `UQModuleLoot_ProfileDataAsset` ; factorisation du tirage pondéré de `Authority_RollSupplyItem` en helper partagé | compile à froid, registre lu au boot |
| **L2** | `UQModuleLoot_World_SubSystem` : abonnement à `QLevel_LevelLoad`, lecture du catalogue, hash 64 bits, tirage, spawn ; commandes `qmoduleloot.Test.*` | **prototype P1** : un module apparaît dans un bunker en PIE |
| **L3** | Persistance : branchement `AddItemPickedToRespawn`, vérification du cycle long après redémarrage | ramassage, redémarrage, absence confirmée |
| **L4** | Passe multijoueur : serveur d'écoute deux instances, vérification que serveur et client posent le module **à la même position**, et que le ramassage d'un joueur le retire chez l'autre | recensement chiffré dans le log |
| **L5** | Profils P2 à P5, équilibrage de densité, plafonds par niveau | passe de relecture des 241 niveaux |
| **L6** | Extension : rescan par lots des 162 QLevel atteignables sans catalogue | nouvelle couverture chiffrée |

Les lots 2, 3 et 4 sont indissociables : un module posé mais non persistant, ou posé à
deux endroits différents selon la machine, est une régression, pas une fonctionnalité.

---

## 8. Risques et dettes

| Risque | Portée | Traitement |
|---|---|---|
| Seed `int32` saturé à l'échelle planétaire dans `GetSeededItemByLocation` | 3 BP existants, non déployés en prod | non corrigé ici ; le nouveau code n'utilise pas cette formule. Tâche dédiée. |
| `Item:DropWeight` ignoré par le tirage BP | toutes les tables du projet | le nouveau code utilise le tirage pondéré C++ ; la correction du BP est une tâche dédiée |
| `PopulateAllQLevelsWithInteractableData` fait crasher l'éditeur | passe de rescan L6 | par lots, jamais d'un bloc |
| `Q_IronCity_Test` pèse 19 % des conteneurs vivants | équilibrage | décision RzZz requise avant L5 |
| Statut de `YellowWall` et `YellowWall_OLD` | 113 QLevel, 10 286 ancres | décision RzZz requise |
| Densité mal calibrée : le module devient banal | design | plafond `MaxModulesPerLevel` dès L2, multiplicateur global dans les settings |
| Variable `WorldP?ckedItems` d'`ItemsManagerGS` contient un caractère non ASCII | contrat gelé | ne pas y toucher |

---

## 9. Contraintes à respecter

- **Sources en ASCII pur**, commentaires compris.
- **Jamais de Live Coding sur QModule** : le module porte plusieurs `UWorldSubsystem`,
  le réinstanciage désynchronise la collection et fait crasher l'éditeur au PIE
  suivant. Toujours un build à froid.
- Le nouveau subsystem porte un `UWorldSubsystem` de plus : la règle ci-dessus
  s'applique à lui dès sa création.
- **`Enabled` à `false` par défaut** dans `DefaultGame.ini`, comme QModule.
- Textes destinés au joueur : localisation, source en anglais.
- Tout nouveau dossier d'assets doit passer les quatre registres de cook.

---

## 10. Questions ouvertes

1. `Q_IronCity_Test` : niveau réel ou niveau de travail ?
2. `YellowWall` et `YellowWall_OLD` : morts, ou en attente de réassemblage ?
3. Valeur du délai de respawn des modules : quel ordre de grandeur, heures ou jours ?
4. Densité cible : combien de modules trouvés par heure d'exploration ?
5. `ContreMesuresVoss` : fabricant IC Lab ou Voss ?
6. Un module déjà installé sur le mur doit-il continuer d'apparaître dans le monde,
   ou la table doit-elle exclure ce que le joueur possède déjà ?

---

## 11. Implémentation (2026-08-14)

Feu vert RzZz du 2026-08-14 : lots L0, L1, L2 livrés, plus le canal police (nouvelle
demande du même jour : « certains IA police droppent des modules passifs assez
rarement, actifs très très rarement »). Compile vert : `QangaEditor` (97 actions,
premier coup) ; cible `QangaServer` vérifiée aussi. **Tout est dormant** :
`Enabled=False` dans `DefaultGame.ini`, activation de session par `qmoduleloot.Enable`.

### 11.1 Code (plugin QModule, plus un hook QAI)

| Fichier | Rôle |
|---|---|
| `QModuleLoot_Library.h/.cpp` | rouleur pondéré `RollWeightedItem` (factorisé DEPUIS `AQModule_SupplyCrateActor::Authority_RollSupplyItem`, qui y délègue désormais : plus de doublon), `ResolvePickupClass` (« ItemScript » d'abord), `SpawnPickupActor` (spawn différé, propriétés `RespawnTimeSeconds`/`DropSpawned` posées AVANT `FinishSpawning`), `HashWorldLocation` (64 bits, quantifié 10 cm), `ResolveGameWorld` (partagé avec les TestCommands historiques) |
| `QModuleLoot_Profile.h` | `UQModuleLoot_Profile_DataAsset` : préfixes de chemin, poids par catégorie d'ancre (clés FName pour garder la dépendance DQS privée), table, chance par ancre, plafond par niveau, respawn |
| `QModuleLoot_Settings.h/.cpp` | `[/Script/QModule.QModuleLoot_Settings]`, `Enabled=false` par défaut, exclusion `Q_IronCity_Test` par défaut, bloc police (taux, tables, mots-clés d'éligibilité) |
| `QModuleLoot_World_SubSystem.h/.cpp` | le cœur : bind unique sur `QLevel_LevelLoad`/`LevelUnloaded`, seed déterministe par position monde de l'instance, un `FRand` par ancre éligible dans l'ordre stable du catalogue, spawn NON répliqué identique sur chaque machine, destruction des spawns à l'unload (pas de doublon au re-stream) ; canal police |
| `QModuleLoot_TestCommands.cpp` | `qmoduleloot.Enable/Disable/Status/SimulateLevel/PoliceDrop`, strippé en Shipping |
| `QAI_AgentComponent.h/.cpp` | **hook producteur** : `static FQAI_OnAnyAgentDiedNative OnAnyAgentDiedNative`, broadcast dans `HandleDeath` APRÈS la coupure d'autorité (serveur seul, une fois par mort via `bDeathHandled`, cadavre encore valide et encore enregistré, jamais déclenché par un despawn/cull). Patron identique à `UQRadioComponent::OnAnyRadioRegisteredNative` |

Dépendances ajoutées : `QLevel`, `DynamicQuestSystem`, `QAI` (privées, `Build.cs` +
`.uplugin`). Pas de cycle (vérifié : DQS ne dépend pas de QModule).

### 11.2 Canal police : pourquoi ce câblage

L'investigation du 2026-08-14 (lecture de code + graphes BP, éditeur) a établi :
la mort d'une unité police est actée par le BP `CombatComponent` (dispatcher
`OnDeath`), que `UQAI_AgentComponent` écoute déjà par réflexion ; tout converge dans
`HandleDeath()`. S'accrocher à `OnDestroyed` côté QPolice aurait droppé du loot sur
les unités simplement **cullées** ; renseigner `ItemDropLoot` sur les BP police
aurait exigé des éditions BP et n'aurait pas permis le double palier passif/actif.
Le roster police n'a **aucune** table `ItemDropLoot` (mesuré) : aucun conflit avec
le loot existant.

Mécanique : à chaque mort d'agent QAI, le subsystem filtre par monde (délégué
statique, plusieurs mondes en PIE), vérifie l'éligibilité par mot-clé de nom de
classe (`AI_DronePolice` couvre standard/Captain/Heavy ; `AI_AutonomusPolice`), puis
un seul jet : bande active d'abord (0,15 %), sinon bande passive (2 %), sinon rien.
Le drop est rollé côté serveur dans la table pondérée et spawné **répliqué** (un
événement dynamique n'est pas déterministe côté client), en mode `DropSpawned`,
sans respawn. Pools : passifs = les 10 modules cyborg passifs à item ; actifs =
`AntenneLonguePortee` et `NidDeFrelons`, les 2 seuls actifs (liste des 8 gadgets en
dur dans `QModule_GadgetHUD.cpp:46`) qui ont un item. Les vaisseaux police sont
**exclus des mots-clés v1** : leur mort passe peut-être par `VehicleCombatComponent`,
non vérifié.

### 11.3 Données (toutes sous `/Game/Phases/QModuleV2/Loot/`, dossier déjà cuit)

5 tables `DA_Loot` (dupliquées de `Supply_LootDA`) : `LDA_QMLoot_Common` (10
passifs, poids 100/r1 et 35/r2), `LDA_QMLoot_Relay` (12, actifs à 20 et 3),
`LDA_QMLoot_Market` (8, raretés hautes favorisées), `LDA_QMLoot_PolicePassive` (10),
`LDA_QMLoot_PoliceActive` (Antenne 100, Frelons 10). NOTE : ces tables sont
consommées par le rouleur C++ (poids relatifs). Si un jour elles sont branchées sur
`CombatComponent.GetRandomDrop`, la sémantique change (probabilités par item, cf. §2.4).

5 profils `QMLP_*` : Bunker (Container 1.0/Furniture 0.3, 1 %, max 2, 72 h),
Relay (+Terminal 0.6/Machinery 0.3, 0,6 %, max 2, 72 h), Ruins (Container 1.0,
0,8 %, max 3, 48 h), Warp (0,4 %, max 1, 96 h), Industrial (0,8 %, max 2, 72 h).

Passe L0 appliquée : les 8 QMD à prix 2500 et Rarity 0 sont passés à **Rarity 1**.
Les anomalies fabricant (`ContreMesuresVoss` sans tag, `CoqueIntegrale` Voss sans
lieu Voss) restent ouvertes (questions §10).

### 11.4 Vérifié / pas vérifié

**Vérifié (PIE, 2026-08-14, marqueurs dans le log)** : subsystem lié (`bound=1`,
5/5 profils), `SimulateLevel` sur `Q_L_Camp_Bunker_Sangline_A1` a spawné 2 pickups
(plafond du profil atteint : 229 conteneurs à 1 %), `PoliceDrop` passif et actif ont
spawné, et les 4 acteurs mesurés en jeu sont bien les classes de ramassage réelles
(`IS_QModuleCy_CoqueIntegrale_C`, `RecuperateurCinetique_C`,
`SacocheDimensionnelle_C`, `AntenneLonguePortee_C`).

**Pas encore vérifié** : une mort réelle d'unité police en jeu atteignant le hook
(le chemin est établi par lecture, pas observé) ; le chemin streaming réel
(`QLevel_LevelLoad` dans l'univers, vs la commande de simulation) ; l'identité des
positions serveur/client en multijoueur (garantie par construction, à mesurer comme
la passe 15.12 de QModule) ; le comportement du ramassage/respawn long en boucle
complète ; les vaisseaux police (exclus v1).

### 11.5 Revue adversariale du 2026-08-14 : 5 défauts confirmés, 5 corrigés

Passe de 19 agents (3 relecteurs par dimension, chaque trouvaille contre-vérifiée
par un réfutateur indépendant) : 16 trouvailles brutes, 5 confirmées sur pièces,
toutes corrigées le jour même, les deux cibles recompilées vertes.

1. **CRITIQUE, préexistant (plugin QLevel)** :
   `UQLevel_Streaming_Instance::QLevel_GetAdditionalData` avait un test de nullité
   inversé et retournait **toujours false**. Conséquence : le chemin streamé du
   placement ne posait JAMAIS rien, pendant que `qmoduleloot.SimulateLevel`
   (lecture directe de l'asset) affichait des succès : la fausse feature type.
   Corrigé des deux côtés : l'accesseur est réparé (zéro appelant C++ ou BP
   n'avait jamais vu `true`, vérifié projet entier, donc aucun changement de
   comportement pour l'existant), ET le subsystem lit désormais l'asset en direct,
   même source pour le chemin streamé et la simulation.
2. **MAJEUR, préexistant (caisse de ravitaillement)** : le contenu était spawné
   côté autorité seulement, non répliqué ; or `ItemScriptBase` ne réplique pas par
   défaut : **personne ne voyait les supplies en serveur dédié** (seul l'hôte d'un
   listen server les voyait). La caisse passe par `SpawnPickupActor` répliqué,
   comme le drop police.
3. **MINEUR (code neuf)** : `RespawnSeconds=0` n'était jamais écrit (garde `> 0`)
   alors que le défaut de classe d'`ItemScriptBase` est **1000** (mesuré au parsing
   binaire du CDO) : un drop de mort serait devenu une source de module se
   régénérant sur place. Sémantique corrigée : `>= 0` écrit, négatif = ne pas toucher.
4. **MAJEUR (code neuf)** : le drop répliqué partait éveillé, sans dormance, sans
   cull dédié, sans durée de vie ni suivi : population croissante d'acteurs
   répliqués reconsidérés pour 500 connexions. Corrigé : `DORM_Initial`, fréquence
   réseau 2 Hz, et expiration `WorldDropLifetimeSeconds` (défaut 1800 s, config)
   sur tous les drops répliqués (police + caisse).
5. (= le n°2 vu par la dimension réseau, même correctif.)

Au passage, le build serveur a réveillé un bug IWYU latent du plugin marketplace
`UltimateLevelArtTool` (`GWorld` sans include de `Engine/World.h`, module
AutoSpline) : corrigé dans les 2 fichiers fautifs, commenté.

### 11.6 Dette signalée au passage (hors périmètre, non corrigée)

Dans `QPoliceSubsystem.cpp` : un `static int32 HeavyDroneToggle` local de fonction
(:4888, viole la règle multi-monde du CLAUDE.md §6) et deux chemins de classe morts
(:4850 `/Game/Characters/AI/AI_DronePolice`, :5493 `LavrikPolice`, jamais existants).
Proposé en tâche séparée.

*Document rédigé le 2026-08-07, mesures en éditeur. Implémentation, revue et
section 11 le 2026-08-14.*
