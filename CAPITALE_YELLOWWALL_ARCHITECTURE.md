# Capitale Yellow Wall : levels de construction, bake LO_PI et chargement en jeu

> Mesuré hors moteur le 2026-10-22, par lecture directe des `.umap` (outil `Saved/Claude/uasset_offline/`, voir §11).
> Aucun fichier du jeu n'a été modifié pour écrire ce document. Unités : mètres, coordonnées du persistant de construction
> (identiques à celles du persistant Univers, la capitale est posée à l'origine).
> Carte : `Documentation/Capitale_YellowWall/carte_quartiers.png`. Liste complète des intrus : `Documentation/Capitale_YellowWall/intrus_2026-10-22.csv`.
> Carte de toute l'enceinte, nord en haut, levels hors `QL_YellowWall` compris : `Documentation/Capitale_YellowWall/carte_enceinte_complete.png` (§2 bis).

## 1. Vue d'ensemble

```
SOURCES (éditeur)                        BAKE (outil Level Operator / LO_PI)        JEU (Steam)
Maps/CompositeUniverse/Sous_Levels  ---> LO_PI_D_YellowWall.umap (foliage)          L_Persistent_Universe
 (copies cuites dans D, mars 2026)       + Sub/L_LO_PI_D_YellowWall_0..45      -->   + L_Steaming_On_Earth (toujours chargé)
                                           (ISM, grille de 300 m)                      = sphères LevelStreamingArea_C
_QLevel/.../YellowWall/QL_YellowWall      + Sub/LO_PI_D_YellowWall_Others              centrées sur l'origine (0,0,0)
 + 74 sous-levels SUB_LEVEL               + Keep/LO_PI_D_Keep
 (copie de travail QLevel, nov. 2025,
  zone Traverse extraite en QLevel)
```

**Vocabulaire (Benja, 2026-10-22)** : Yellow Wall = la capitale = **tout ce qui est dans le mur, mur compris**. Le **cœur**
est le centre-ville (les 13 zones de la StarMap) ; le reste de l'enceinte (Complex, AirPort, ceinture sud, parc SawgeniuS,
IcliPort...) et le mur font autant partie de la capitale.

La capitale n'est **pas** en QLevel (seul Traverse a une version QLevel, voir §10). Le jeu charge la version cuite LO_PI "D",
cuite depuis les copies `Sous_Levels` et **pas** depuis `QL_YellowWall` (preuve au §6).

**`QL_YellowWall` ne contient pas toute la ville.** 11 levels de l'enceinte vivent hors de lui : toute la ceinture sud
(South City, AirField Sud, usine d'armes, Trash Area, UnderWater Desert, carrière, cuits dans D), le parc SawgeniuS,
IcliPort, Arrêt Nord, la station d'épuration et le bâtiment A de l'hôpital (non cuits). Voir §2 bis.

Autour de la ville, deux éléments vivent à part et ne passent pas par le bake : le **mur d'enceinte** (Blueprint
`BP_YellowWallComplex`, anneau de 4,5 km de rayon) et le **terrain cuit** (`HM_Bake_Capital`). Voir §4 bis et
`Documentation/Capitale_YellowWall/carte_mur_terrain.png`.

## 2. Le persistant de construction : `QL_YellowWall`

`/Game/_QLevel/Universe/Planetary/YellowWall/QL_YellowWall` : le persistant qui rassemble toute la ville (quartiers, couches,
Complex, intérieurs, mur extérieur). C'est une copie de travail préparée en novembre 2025 pour la conversion QLevel (pilote
Traverse) ; la capitale n'est pas convertie pour autant. 74 `LevelStreamingDynamic`, tous avec une transformation identité
(les sous-levels sont dans les coordonnées du persistant). Dossiers du panneau Levels :

| Dossier | Contenu |
|---|---|
| Capital (46) | quartiers `SUB_LEVEL/Capital/L_Capital_*`, couches (Roads, SkyRoad, Decal, BaseStruct, Build, Pietonniere...), sons, `L_MUR_EXTERIEUR` (mur d'enceinte + extérieur du Complex + sol cuit + planète, §4 bis) |
| Complex (15) | `SUB_LEVEL/Complex/L_Complex_*`, `L_Lab_*`, `L_Int_Complex_MiddleTower` **et** `L_Int_Complex_MiddleTower1` |
| Interieur (8) | boutiques `SUB_LEVEL/Capital/Interieur/L_Sub*`, `L_Int_Capital_AeroTram`, `L_Int_Capital_GoldShop` |
| EN+ (3) | `L_FoliageYellowWall`, `L_FoliageYellowWall_Nord`, `L_Gameplay_Destructible` |
| Light (2) | `L_Gameplay_PoleLight`, `L_TarmacLighting` |

Masqués dans l'éditeur par défaut : `L_MUR_EXTERIEUR`, les deux foliages, les sons.

Le persistant contient lui-même de la géométrie : un groupe de 25 StaticMeshActor (`GroupActor_358`, un bâtiment préfabriqué
`SM_MERGE_Bat_Prefab_*` + auvents + enseigne Iclab) vers (X -1252, Y -106), qui devrait vivre dans un quartier, et 8
`QLevel_Actor_Instance` : les 5 morceaux de Traverse (`Q_LEVEL/Traverse/...`), `Q_L_Light_YellowWall` (lumières, fichier
`L_Light_YellowWall` modifié le 2026-10-13) et deux maisons `Q_QL_Maison_A3`.

Couches (elles traversent la ville par construction, leurs "intrus" ne sont pas des erreurs) : `L_Capital_BaseStruct`
(sol, murs de soutènement, falaises de glace du fond, 5,7 x 9 km), `L_Capital_Roads`, `L_Capital_SkyRoad`, `L_Capital_Decal`,
`L_Capital_Build`, `L_Capital_Pietonniere(_Detail)`, les foliages, `L_Gameplay_Destructible`, les sons, `L_Capital_Effect`,
`L_Capital_Mission(1)`, `L_Capital_Qtrain`.

## 2 bis. Ce qui est dans l'enceinte mais hors de `QL_YellowWall`

Carte : `Documentation/Capitale_YellowWall/carte_enceinte_complete.png` (nord en haut).

Orientation : le nord est vers **Y négatif**, l'est vers **X positif**. Déduit des noms, pas lu dans un réglage : zone StarMap
"nord" à Y -770..-270, zones "CentreSud" et "entrysud" côté Y positif, `L_ArretNord` à Y -830, `L_YW_AirField_Sud` à
Y +3 900, `L_Capital_Entry_Est` côté X positif. Avec cette boussole, le Complex est au sud-est, contre le mur.

| # | Level | Où (centre, m) | Bake D (instances de ses meshes propres) | Qui le charge |
|---|---|---|---|---|
| 1 | `Maps/CompositeUniverse/_SubLevel/L_YW_SounthCity` (sic) | sud, X 0 / Y 3 085 | cuit (28/28) | aucun persistant actuel |
| 2 | `Maps/CompositeUniverse/Sous_Levels/L_YW_AirField_Sud` | sud, X 560 / Y 3 880 | cuit (330/330) | seulement l'ancien `_OLD/L_Persistent_UniverseNEWTERRANOISE` |
| 3 | `_SubLevel/L_YW_WeaponFactory` | sud, X 530 / Y 2 620 | cuit (167/167) | aucun persistant actuel |
| 4 | `_SubLevel/L_YW_TrashArea` | sud, X 900 / Y 2 090 | cuit (125/126) | aucun persistant actuel |
| 5 | `_SubLevel/L_YW_UnderWater_Desert` | sud, X -380 / Y 2 560 | cuit (256/256) | aucun persistant actuel |
| 6 | `Sous_Levels/L_YL_Carriere` | sud-ouest, X -1 990 / Y 3 530 | cuit (361/361) | aucun persistant actuel |
| 7 | `_SubLevel/L_SawgeniusPark` (parc d'attractions) | sud-ouest, X -2 940 / Y 2 340 | non cuit (2/131) | seulement `LO_PI_B_YellowWall` (génération B, pas celle du jeu) |
| 8 | `Maps/Universe/Sub_Levels/L_IcliPort` | nord-ouest, X -2 315 / Y -3 155 | non cuit (14/1 520) | la copie `Maps/Trash/L_IcliPort1` |
| 9 | `Maps/Universe/Sub_Levels/L_ArretNord` | nord du centre, X -370 / Y -830 | non cuit (0/1 409) | aucun |
| 10 | `Maps/Universe/Sub_Levels/L_EpurationStation` | sud, X -800 / Y 3 820 | partiel (149/594) | aucun |
| 11 | `Maps/Universe/Sub_Levels/L_Hopital_Bat_A` | sud, X 110 / Y 3 200 | non cuit (2/186) | la copie `Maps/Trash/L_Hopital_Bat_A_WP` |

- Les 6 premiers (la ceinture sud) sont **dans le jeu par le bake D** : D a été cuit avec eux alors que `QL_YellowWall` ne
  les charge pas. En dev, il faut les ouvrir à part. Copies plus anciennes dans `_QLevel/Universe/Planetary/YellowWall_OLD/SubLevel/`.
- Pour 7 à 11, aucun chargeur trouvé : persistants actuels, recherche par nom dans les `.umap` et `.uasset` de `_QLevel`,
  `Maps`, `StarMap`, `_QData`, `Systems`, dans le C++ et les `.ini`. Les fiches `_QData/_Level_Template/Level/QDB_L_IcliPort`,
  `QDB_L_ArretNord`, `QDB_L_EpurationStation` décrivent le lieu mais ne chargent rien (aucun lecteur en jeu). Cela ne
  prouve pas que ces lieux sont absents du jeu : à vérifier en jeu.
- `L_Usine_Autonome_Stream` (dans `QL_YellowWall`, non cuit) est un intérieur de 89 x 88 m, plus bas (Z 34 à 53 m), sur le
  site de `L_YW_WeaponFactory` (cuit, Z 75 à 249 m). Aucun mesh en commun : deux parties du même lieu, pas deux versions.
- 117 acteurs de `L_IcliPort` sont entassés sur 4 m au cœur (X 410, Y -42, Z 115) : voir le lot 2
  (`Documentation/Capitale_YellowWall/lot2_rangement.md`).
- **Vérification croisée avec le jeu** : sur les 7 682 cases de 25 m qui contiennent des instances du bake D (100 545
  instances, `Sub` + `Keep`), 7 676 ont une source à moins de 50 m dans `QL_YellowWall`, la zone Traverse des `Sous_Levels`
  ou ces 11 levels. Les 6 autres sont des objets isolés. L'inventaire du décor cuit est complet.
- Levels de la capitale qui ne sont pas du décor à ranger : gameplay (`L_Gameplay_Master` et `L_Gameplay_Vehicles`,
  chargés par `L_Steaming_On_Earth` ; `L_Gameplay_AI`), son (`_SubLevel_Sound/*`), éclairage (`_SubLevel_Lithing/*`),
  foule Mass (`L_Mass_Capital` chargé en jeu, `Mass_ZoneGraph_SubLevel/*`), outils (`L_RoadTools_Spline`,
  `L_ToolsWorldscape`, `L_Tools_PCG`), `L_Vide*` (presque vides), temporaires du Complex (`_TempLevel/*`),
  `Sous_Levels/Capital_Train/L_CapitalTrain_OLD`.
- Non classés : `Sous_Levels/L_SpacePort` (3 669 acteurs), `Maps/Universe/Sub_Levels/L_Capital_RebelCamp` (585) et les
  grottes `_Cave/01..06` ont leurs acteurs autour de l'origine et aucun chargeur trouvé : leur vraie position (transformation
  de streaming) est inconnue.
- **Décision de Benja (2026-10-22)** : ces 11 levels entrent dans le périmètre du rangement du dev (ils font partie de la
  capitale, §1). On les range là où ils sont : dupliquer une map dans l'éditeur fait planter, et aucun persistant du jeu ne
  les charge. Pour la ceinture sud, déjà dans le bake D, un changement n'atteint le jeu qu'au prochain rebake. Étape
  éditeur à venir : les ajouter à `QL_YellowWall` pour que le dev décrive toute la capitale.

## 3. Le bake : outil Level Operator / Partitioning Instancing (LO_PI)

- Outil Blueprint : `Content/Tools/Level_Operator/Partitioning_Instancing/` (`LO_Partitioning_Instancing_Tools`,
  `Process/LO_PI_Process`, étapes UnCompose, Sort, Spawner, Sub_Level, HLOD, ExcludeClass). Aides C++ :
  `Plugins/QLevelPartition_Editor` (`UQLP_FunctionLibrary` : cellules, instances par taille, approximation de mesh).
- Sorties dans `Content/Maps/CompositeUniverse/_Optimised_Lvl_Capital/` : générations B (2024-06-01), A (2024-11-01),
  C (2025-02-13), **D (2026-03-07 23:44, celle du jeu)**.
- Contenu de D :
  - `LO_PI_D_YellowWall` : 274 `LO_PI_Foliage_Actor` (la végétation, en foliage ISM).
  - `Sub/L_LO_PI_D_YellowWall_0..45` : `LO_PI_ISM_Actor` (instances regroupées, acteurs posés aux centres d'une grille
    de 300 m) et `LO_PI_SM_Comp_Actor`.
  - `Sub/LO_PI_D_YellowWall_Others` : tout ce qui n'est pas un mesh (BP de props, decals, Niagara...) dont
    157 `LevelStreamingArea_C` d'intérieurs (rayon médian 50 m).
  - `Keep/LO_PI_D_Keep` : BP de gameplay gardés tels quels (matter stations, spawners de véhicules, lumières...).

## 4. Le chargement en jeu

`L_Persistent_Universe` charge en permanence `Maps/Universe/Sub_Levels/L_Steaming_On_Earth`, qui contient des
`LevelStreamingArea_C` (BP : sphère de détection + `LevelsToStream` + `StaticMeshListWhenUnloaded` affichés quand c'est déchargé) :

| Sphère | Centre | Rayon | Levels |
|---|---|---|---|
| `SL_YellowWallCapital_PI` | origine | 27,9 km | les 46 cellules + Others + Keep + `LO_PI_D_YellowWall` + `L_Capital_Mission(1)` + `L_Capital_(Big)PoleLight_V2` ; 11 meshes lointains `HM_Bake_Capital_HLOD` / `APPROX_SM_*` quand déchargé |
| `SL_YellowWallEffects` | (0, 0, 636) | 64,3 km | `L_YellowWall` (le mur d'enceinte `BP_YellowWallComplex` + l'extérieur du Complex), `L_YellowWall_Effect`, `L_Capital_Lighting_BlockVolume`, `L_Gameplay_Master`, `L_Mass_Capital` ; `APPROX_Pipe_Segment_Wall_part` quand déchargé |
| `SL_YellowWallTerrain` | origine | 15,9 km | `L_YellowWallGround` (acteur `HM_Bake_Capital_Actor`, le sol cuit) ; `HM_Bake_Capital_HLOD` quand déchargé |
| `SL_YellowWall_Capital_Details` | origine | 5,1 km | `L_Gameplay_Vehicles`, `L_Capital_Sound`, `L_Capital_Effect` |
| 3 sphères Complex | Complex | 1,1 à 1,6 km | **désactivées** (`Enabled=False`) : le Complex est désormais cuit |

Ces levels de jeu non cuits viennent de `Maps/CompositeUniverse` (`Sous_Levels`, `_SubLevel*`, `Mass_ZoneGraph_SubLevel`),
jamais de `_QLevel/.../SUB_LEVEL`.
L'intérieur de la tour (`Sous_Levels/L_Int_Complex_MiddleTower`) est streamé par une sphère de 169 m posée dans
`L_Gameplay_Master` à (2937, 2946, 1885).

## 4 bis. Le mur d'enceinte et le terrain

**Le mur (celui qui donne son nom à la ville)**
- C'est un Blueprint, pas un level : `/Game/Maps/BP_YellowWallComplex` (41 composants instanciés, 1 950 instances,
  matériaux `/Qasset/Wall`). Il est posé à l'origine dans `Sous_Levels/L_YellowWall` (jeu : sphère Effects de 64 km,
  non cuit) et dans `SUB_LEVEL/Exterieur/L_MUR_EXTERIEUR` (construction). Le modifier, c'est modifier ce BP, partagé
  par le jeu et la construction.
- Géométrie mesurée : anneau fermé (aucun trou de plus de 3°) centré en (X -8, Y 3), rayon moyen 4,5 km (pièces entre
  4,2 et 5,6 km du centre), environ 28 km de tour. Piliers `Pillar_Wall_*` / `Pillar_Part_*`, tuyaux `Pipe_Segment_*`
  (252 de chaque), 4 grands modules `Straight_Module_*` aux 4 points cardinaux à 4,3-4,5 km (porte d'eau, cascade, tour
  d'observation, route basse ; `Straight_Module_MainFrame` sur trois d'entre eux, `SM_MUR_Sortie_aqua` sur le quatrième,
  côté X négatif). Le même BP porte l'extérieur du Complex (tours `SM_Left/Right_*`, `SM_Entry`, héliports) collé au mur
  côté X+ Y+ (au sud-est avec la boussole du §2 bis), et un chantier naval.
- Piège de mesure : ces meshes ont leur pivot près du centre de l'anneau et leur géométrie à 4 km. Un calcul sur les
  pivots seuls donne un faux rayon de 850 m ; il faut passer par les bounds des meshes.
- Au-delà de 64 km, le mur est remplacé par `APPROX_Pipe_Segment_Wall_part`.
- `/Game/Inst_YellowWall` (2026-04-24) : variante HISM du même anneau (17 composants, 262 instances), posée dans aucun
  level de `Maps` ni de `_QLevel` (recherche par nom). Rôle à confirmer.

**Le terrain**
- Le sol de la planète est modelé à l'origine (`L_Persistent_Universe`) par deux `HeightMapVolume` WorldScape centrés en
  (0, 0, -60 m) : `TerrainWall_HMVolume2` (heightmap `/Game/Maps/TerrainrWallHM` : hauteur, température, humidité, océan ;
  échelle 110 000) et `OceanWall_HMVolume` (`/Game/Maps/OceanWallHM1` : niveau d'eau entre 10 768 et 11 270 m d'altitude
  planétaire ; échelle 9 050).
- Dans l'enceinte, ce sol est cuit en meshes : `Content/Baking_Capital/HM_Bake_Capital_*`, produit par l'outil WorldScape
  `WS_BakingToStaticMesh` (`Plugins/WorldScape/Content/Tools/WS_Heightmap_Editor/EditorContent/Baking_SM/`), données
  `HM_Bake_Capital_Data` (2023) : 16 x 16 tuiles de rendu (`Proxy_SM`), 16 x 16 tuiles de collision (`Collision_SM`),
  sous-échantillonnage 2, et un `HM_Bake_Capital_HLOD`.
- L'acteur `HM_Bake_Capital_Actor` (203 composants) est posé dans `_SubLevel/L_YellowWallGround` et couvre X -4097 à 4378,
  Y -3907 à 4490 m (8,5 x 8,4 km, tout l'intérieur du mur). Chargé par la sphère Terrain (15,9 km), remplacé par
  `HM_Bake_Capital_HLOD` au-delà.
- `L_MUR_EXTERIEUR` contient aussi un `HM_Bake_Capital_Actor` et la planète `EarthScape2` : en construction, on voit le
  même sol et la même planète qu'en jeu.
- `Content/Maps/Universe/YellowWallTerrain/YW_MeshData*` (2023-10-09) : ancien bake de terrain, référencé par aucun level
  de `Maps` ni de `_QLevel`.
- Le terrain cuit date de 2023 : si le relief autour de la capitale change, il faut le recuire avec l'outil WorldScape.

## 5. Position sur la planète

- La capitale est à l'origine du persistant Univers. La Terre est un `WorldScapeRoot` "EarthScape" centré à
  (0, 0, -3 180 000 m), `PlanetScale` 3 169 299,84 m, rotation (Pitch 356,4 ; Yaw -178,1 ; Roll -40,8).
  `EarthPivot` (TargetPoint) et `WS_Level_Planetary` sont au même centre.
- La planète est donc tournée pour que le site de Yellow Wall soit au sommet, sous l'origine : la ville est construite
  "à plat" en coordonnées locales et c'est la planète qui s'oriente autour.
- `L_MUR_EXTERIEUR` contient une copie de cette planète (`EarthScape2`, même centre, même rotation) pour voir le contexte
  pendant la construction.

## 6. Pièges : deux jeux de sources

Chaque quartier existe en deux exemplaires :
- `Maps/CompositeUniverse/Sous_Levels/L_Capital_*`, `L_Complex_*`... (format 5.7, ré-enregistrés le 2026-03-08 03:16). Ce sont
  **eux que le bake D a cuits** : 36 meshes qui n'existent que dans ces copies (absents de tous les levels de `QL_YellowWall`)
  sont tous dans D avec au moins le même nombre d'instances. Les anciens persistants Univers archivés
  (`Maps/Universe/_OLD/L_Persistent_Universe*`, déplacés le 2026-07-07) streamaient directement 44 à 63 de ces levels.
- `_QLevel/Universe/Planetary/YellowWall/SUB_LEVEL/...` (la plupart jamais ré-enregistrés depuis UE 5.3, nov. 2025), chargés
  par `QL_YellowWall`. Comparaison acteur par acteur (nom, position, mesh) sur les 66 paires : **identiques sauf la zone
  Traverse**, dont 2 496 acteurs à mesh ont été retirés de `L_Capital_Traverse` (-1235), `L_Capital_Build` (-918),
  `L_Capital_Roads` (-446), `L_Capital_Pietonniere` (-148) et `L_Capital_BaseStruct` (-142) et se retrouvent (2 485 par
  compte de mesh) dans le pilote QLevel `Q_LEVEL/Traverse` (2 995 acteurs). Petits écarts restants : `L_Capital_Mission`
  (+43 côté SUB_LEVEL), `L_Capital_Port` (retouché le 2026-03-31), `L_Capital_Litchout` (+20 côté Sous_Levels), et des
  acteurs sans mesh en plus côté Sous_Levels dans AirPort, Decal, Gameplay_Destructible ; le reste (portes renommées,
  acteurs enfants) est du bruit de nommage.
- Exception : les 6 levels de la ceinture sud (§2 bis) n'ont **aucun** exemplaire dans `QL_YellowWall` ; D les a cuits
  depuis `_SubLevel` et `Sous_Levels`.

Conséquence : modifier un exemplaire ne change pas l'autre, et rebaker depuis `QL_YellowWall` sans traiter les
`QLevel_Actor_Instance` de Traverse risquerait de perdre la zone Traverse (non vérifié : dépend de l'outil LO_PI).

Autres doublons :

| Construction (`QL_YellowWall`) | Jeu | État mesuré |
|---|---|---|
| `SUB_LEVEL/Exterieur/L_MUR_EXTERIEUR` | `Sous_Levels/L_YellowWall` (non cuit, sphère de 64 km) | mêmes 320 acteurs à un près, plus `HM_Bake_Capital_Actor` et `EarthScape2` côté construction |
| `SUB_LEVEL/Capital/Interieur/L_Sub*`, `L_Int_*` | `Sous_Levels/L_Sub*`, `L_Int_*` streamés par les sphères de `Others`, plus des intérieurs de `/Game/Maps/DevMap/ConstructionLevel/` (Armurerie, NightClub, BigUsine, Starkito, Tower_Isma, AquaTram...) | modifier la copie de construction ne change pas le jeu |
| `L_Int_Complex_MiddleTower1` | rien | doublon exact de `L_Int_Complex_MiddleTower` (931 acteurs sur 931), chargé deux fois en construction |

## 7. Ce qui est cuit, ce qui ne l'est pas

Méthode : pour chaque level de construction, les meshes qui n'existent que dans ce level sont-ils présents dans les
instances du bake D ?

- **Cuits** : tous les quartiers `L_Capital_*` (sauf ceux ci-dessous), tout `L_Complex_*`, `L_Lab_*`, les deux foliages,
  `L_Gameplay_Destructible` (partie mesh), `L_SubPMU`. Indéterminé (aucun mesh propre pour trancher) : `L_Gameplay_PoleLight`,
  `L_Capital_Decal`, `L_Complex_Decal`, `L_Capital_Pietonniere_Detail`, le groupe de 25 meshes du persistant.
- **Non cuits** : les intérieurs (sauf `L_SubPMU`), `L_Int_Complex_MiddleTower(1)`, `L_MUR_EXTERIEUR`, `L_Capital_Effect`,
  `L_Capital_Mission(1)`, `L_TarmacLighting`, `L_Usine_Autonome_Stream` ; `L_UnderWater_Field` partiellement (161 meshes
  propres, 94 instances cuites).
- **Hors `QL_YellowWall`** (§2 bis) : la ceinture sud est cuite (6 levels), `L_EpurationStation` partiellement (149/594) ;
  `L_SawgeniusPark`, `L_IcliPort`, `L_ArretNord`, `L_Hopital_Bat_A` ne le sont pas.
- Pas de chargeur trouvé pour `L_TarmacLighting`, `L_Usine_Autonome_Stream` et `L_UnderWater_Field` dans
  `Maps/Universe`, `_Optimised_Lvl_Capital`, `Sous_Levels`, `_SubLevel`. Cela ne prouve pas qu'ils sont absents du jeu : à vérifier en jeu.

## 8. Bounds des quartiers et intrus

Deux bounds comptent :
- le **`LevelBounds`** de chaque sous-level (acteur `ALevelBounds`, transformation = centre et taille), enregistré dans le `.umap` ;
- la **forme QLevel** (cible à terme) : `UQLevel_Function_Library::QLevel_ComputeBound_FromOnlyStaticMeshComponent`
  (`Plugins/QLevel/Source/QLevel/Private/QLevel_Function_Library.cpp:1267`) = union des bounds de tous les
  `UStaticMeshComponent` visibles (ISM compris) du level. Un seul mesh égaré gonfle le rayon de chargement.

Mesure faite sur le jeu `QL_YellowWall` / `SUB_LEVEL` ; elle vaut aussi pour les copies `Sous_Levels`, sauf dans la zone
Traverse (§6).

Méthode de mesure : chaque acteur est réduit au centre de sa boîte monde (mesh `ExtendedBounds` x transformation, parents
compris) ; le territoire d'un quartier = la plus grande zone connexe de cases de 50 m qu'il occupe (plus toute zone qui
porte au moins 10 % de ses acteurs) ; un intrus est un acteur hors de ce territoire, et le "quartier cible" est le quartier
dont le territoire contient (ou touche à moins de 200 m) sa position. Les boîtes calculées recoupent les `LevelBounds`
enregistrés (CostaRive : 1019 x 1237 m mesurés contre 1019 x 1261 m enregistrés), ce qui valide la lecture.

Quartiers et Complex (tailles X x Y en m, triés par gonflement) :

| Level | LevelBounds enregistré | Cœur mesuré | Intrus | Quartier cible mesuré |
|---|---|---|---|---|
| `L_Complex_Welcome` | 434 464 922 x 256 864 533 | 688 x 682 | 3 | un `Cube` (Cube_DistanceField) à 434 000 km, un `Plane` à (-1802, -826), une reflection capture côté Terrasse |
| `L_Capital_PoliceStation_Sud` | 500 x 1150 | 60 x 115 | 1 | BusStation |
| `L_Capital_CostaRive_Tram` | 486 x 1221 | 126 x 69 | 4 | Center (barrières et clôture à (-57, 24)) |
| `L_Complex_Iclabs_Center` | 842 x 861 | 117 x 152 | 20 | aucun 8, MiddleTower 7, Elevator_Down 5 |
| `L_UnderWater_Field` | 1735 x 2721 | 394 x 397 | 143 | aucun 83, Usine_Autonome_Stream 31, Complex_Restaurant 18 |
| `L_Capital_Banlinor` | 1325 x 1782 | 281 x 351 | 1 | un panneau de l'arrêt Tamil Aqua à (2036, 2067) |
| `L_Capital_Center` | 2903 x 3095 | 677 x 672 | 149 | **Nord 99**, Litchout 22, Entry_Est 17, BusStation 10 |
| `L_Capital_Port` | 4457 x 2549 | 654 x 906 | 28 | Traverse 16, aucun 8 (dont le chalutier `SM_Chalutier2` à (-3385, 1756)) |
| `L_Capital_Entry_Sud` | 1333 x 969 | 281 x 400 | 26 | Banlinor 17, Peninsula 5 |
| `L_Capital_Nord` | 2169 x 1178 | 959 x 429 | 43 | AirPort 41 (36 bordures `SM_Road_Curb_B_*` vers (1850, 250)) |
| `L_Capital_Centinela` | 1478 x 1023 | 616 x 481 | 59 | Litchout 54 |
| `L_Capital_Center_Sud` | 706 x 1088 | 575 x 401 | 7 | Nord 6 |
| `L_Capital_Entry_Est` | 1239 x 952 | 837 x 629 | 16 | Banlinor 15 |
| `L_Capital_Commerces` | 784 x 941 | 544 x 686 | 19 | Nord 15, Entry_Est 4 |
| `L_Capital_CostaRive` | 1019 x 1261 | 972 x 1031 | 3 | aucun 2, Entry_Sud 1 |
| `L_Complex_Terrasse` | 1174 x 911 | 1174 x 773 | 5 | MiddleTower 4 |
| `L_Complex_Parking`, `L_Complex_Elevator_Top`, `L_Int_Capital_AeroTram`, `L_Int_Complex_MiddleTower(1)` | | | 1 à 2 | |

Écart sans intrus mesh (LevelBounds figé, ou composants non mesh comptés par `ALevelBounds`) : `L_Int_Complex_MiddleTower(1)`
(3018 m enregistrés pour 171 m de meshes), `L_Capital_Centinela_Tram`, `L_Capital_PoliceStationTraverse`,
`L_Complex_PoliceStation`, `L_Capital_Peninsula`, `L_Lab_Chamber`.

Limites de la mesure : les instances de foliage (`InstancedFoliageActor`) et les meshes hérités d'un Blueprint sans
surcharge dans le level ne sont pas mesurés (l'acteur est alors réduit à son pivot). Le mur d'enceinte et le sol cuit
(§4 bis), qui sont des Blueprints, sont donc hors de cette mesure ; ce ne sont pas des quartiers. Les noms de quartier cible sont une
proposition géométrique, pas une décision.

## 8 bis. Référence officielle des quartiers : les zones de la StarMap

`Content/StarMap/_Data/LevelMap/Earth/Capital/Capital_Map_3D.umap` contient 13 acteurs `StarMap_MapView_Zone_*`, chacun
avec un mesh plat de `Capital/Zone/` (Banlinor, Centinela, Centre, CentreSud, costarive, EntryEst, entrysud, Hopital,
Litchout, nord, Penisula, port, Travers). Ils sont posés **dans les coordonnées de la ville, à l'échelle 1** : c'est le
découpage officiel des quartiers tel que la carte le montre au joueur. Les sommets de ces meshes ne sont pas lisibles
hors moteur (données source stockées hors du `.uasset`) ; on compare avec leurs boîtes.

Recoupement avec les territoires mesurés au §8 (écart entre les centres) : Nord 3 m, Banlinor 9 m, Hôpital 52 m,
CostaRive 63 m, Port 76 m, Centre et CentreSud 85 m, Peninsula 117 m, EntrySud 124 m, Litchout 129 m, Centinela 144 m,
EntryEst 158 m, Traverse 182 m (normal : dans le dev, la zone Traverse est partie dans le QLevel). Les zones pavent la
ville presque sans recouvrement. Carte : `Documentation/Capitale_YellowWall/carte_zones_starmap.png`.

Règle pour les lots suivants : un objet va d'abord dans le quartier dont la **zone StarMap** le contient ; le territoire
mesuré ne sert que pour les quartiers sans zone (AirPort, Complex, BusStation, Decharge, Tranche...) et pour les objets
hors de toute zone (zone la plus proche).

## 9. Avant de déplacer des objets entre levels

1. Le déplacement se fait dans l'éditeur, persistant de construction ouvert (`EditorLevelUtils.move_actors_to_level` ou
   "Move Selection to Current Level") : un `.umap` ne s'édite pas hors moteur.
2. **Décision de Benja (2026-10-22)** : on ne touche pas à la version cuite ni à rien de ce que le jeu lit ; on range et on
   met en cohérence uniquement le dev, c'est-à-dire `QL_YellowWall` et le dossier `_QLevel/Universe/Planetary/YellowWall/`,
   **plus**, depuis la même date, les 11 levels de la capitale qui sont hors de `QL_YellowWall` (§2 bis), rangés en place.
   Vérifié : aucun fichier de la chaîne de jeu (persistant Univers, `L_Steaming_On_Earth`, bake D, levels non cuits de
   `Sous_Levels` et `_SubLevel*`) ne cite ce dossier. Réserves dans ce dossier : `SUB_LEVEL/Capital/L_Capital_Mission` et
   `L_Capital_Qtrain` sont ajoutés au cook par EasyCook (rapport `Config/EasyCookGenerated.ini`), donc on n'y touche pas
   avant de savoir qui les lit ; `L_Light_YellowWall` est un travail en cours (2026-10-13). Les assets partagés
   (`BP_YellowWallComplex`, `HM_Bake_Capital_Actor`, meshes, matériaux) ne se modifient pas : on déplace seulement leurs
   acteurs dans les levels de dev.
3. Ce qu'on range (Benja, 2026-10-22) : les objets de level design (static meshes, particules, préfabs de décor) vont
   dans le quartier où ils sont posés. Ce qu'on ne disperse pas : tout ce qui est mission (levels et acteurs de mission)
   et les lampadaires, qui restent chacun dans leur level unique. Les couches qui couvrent toute la ville (Roads, Decal,
   SkyRoad...) gardent leurs objets partout, c'est voulu.
4. Le level cible doit être de la même famille au runtime : un mesh déplacé d'un quartier cuit vers un intérieur ou un level
   non cuit ne sera plus visible de la même façon en jeu (§7).
5. Les acteurs d'un `GroupActor` ou attachés à un parent doivent bouger avec lui.
6. Après modification : rebaker (LO_PI) puis vérifier que `SL_YellowWallCapital_PI` pointe bien sur la génération produite.
   Point non vérifié : la façon dont le bake traite les `QLevel_Actor_Instance` (la zone Traverse n'est plus que dans
   `Q_LEVEL/Traverse` côté `QL_YellowWall`). À contrôler avant tout bake depuis `QL_YellowWall`.
7. Ne pas toucher `L_Light_YellowWall` / `Q_L_Light_YellowWall` sans demander (travail récent, 2026-10-13).

## 10. Vers le QLevel

Cible annoncée : passer la capitale au système QLevel (un `UQLevel_Asset` par quartier, chargé par la forme calculée au §8).
Seul Traverse est converti : `Q_LEVEL/Traverse/` (`QL_M_Capital_TRAVERSE`, Base, BUILDING avec 7 bâtiments, DETAILS, Road,
`SM_MERGED_Traverse`), posé dans `QL_YellowWall`. Nettoyer les intrus d'abord : c'est ce qui donne des rayons de chargement justes.

## 11. Outil hors moteur

`Saved/Claude/uasset_offline/` (local, non synchronisé) : lit les packages éditeur UE 5.3 et 5.7 sans lancer l'éditeur.
- `ue_pkg.py` : en-tête, noms, imports, exports, soft paths, propriétés taguées (formats 5.3 et 5.4+), vecteurs pré-LWC.
- `level_actors.py` : acteurs d'un level avec transformation monde (chaîne `AttachParent`).
- `city_scan.py` : scanne les 74 sous-levels de `QL_YellowWall` (bounds enregistrés, boîtes meshes) vers `city_scan.json`.
- `outliers.py` : territoires et intrus vers `outliers.json` ; `city_map.py` : la carte ; `diff_levels.py` : compare deux copies d'un quartier.

Commande : `python city_scan.py` puis `python outliers.py` (quelques minutes, disque uniquement).
