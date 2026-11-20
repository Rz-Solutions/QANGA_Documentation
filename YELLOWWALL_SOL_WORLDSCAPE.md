# Capitale Yellow Wall : remplacer le sol cuit par le terrain WorldScape

> Étude du 2026-11-19, révisée le même jour après les décisions de Benjamin (§0). **Validée par Benjamin le 19/11.** **Aucune modification du projet**
> (seuls ce rapport et trois cartes dans `Documentation/Capitale_YellowWall/` ont été écrits). Méthode : lecture hors
> moteur des `.uasset`/`.umap` (`Saved/Claude/uasset_offline/`), décodage des heightmaps et des textures sources,
> lecture du code WorldScape et QAI. L'éditeur n'a pas été lancé.
> Repère : X = est, **Y négatif = nord**, origine = centre de la capitale, mètres.

---

## 0. Décisions de Benjamin (19/11)

1. La règle du 22/10 « ne pas toucher la version cuite » est levée.
2. **100 % WorldScape** : on retire le sol cuit **et ses collisions** (résolution limitée, mauvais contacts avec les
   véhicules ; on libère du stockage).
3. **La collision terrain sur le CPU est acceptée** (« la plupart des trucs sont en GPU, que le CPU soit utilisé pour
   la collision, c'est pas grave »). Le reste (rendu, bruit, feuillage) reste sur le GPU.
4. Le climat (température, humidité) de l'enceinte vient de la bonne texture, pour que le feuillage suive les biomes
   à l'intérieur du mur.
5. Masques de feuillage sur les zones construites au sol (levels). Les trous pour les tunnels : plus tard.

## 1. Réponse courte

- **La bonne hauteur existe et elle est exacte** : `/Game/Maps/ComplexWallHM1_True` (8192²). Son tableau de hauteurs
  est identique au canal rouge de `ComplexWallHM1_coqdzqdpy_Tex2D`, la source du bake (écart max 1 cm). Les 203 tuiles
  Nanite collent à cette heightmap (décalage moyen 0,05 m, pas de miroir). Ça confirme ce que dit Benjamin.
- **Le volume d'avant existe encore** : `HM_WallComplex2` dans `Maps/CompositeUniverse/Sous_Levels/L_ToolsWorldscape`
  (non chargé en jeu), aux dimensions exactes du bake.
- **Le terrain WorldScape actuel est une cuvette** : `TerrainrWallHM` vaut 0 dans toute l'enceinte, le sol est à
  environ -600 m sous la ville, soit 614 à 1 295 m sous le sol cuit. On ne retouche pas `TerrainrWallHM`, on ajoute
  le volume de la capitale par-dessus.
- **Le climat stocké dans `ComplexWallHM1_True` est cassé** (§2.6) : on le reprend dans la même texture source,
  écrêté au lieu d'être replié.
- **Trois réglages à ne pas rater** : position du volume (9 m d'écart entre l'outil et l'acteur cuit), courbure
  (3,2 m au mur), taille des données (l'atlas GPU déborderait). Détail au §2.3 et au §5.

## 2. Mesures

### 2.1 Les volumes de la capitale (`L_Persistent_Universe`)
| Volume | Données | Emprise | Hauteur | Rôle mesuré dans l'enceinte |
|---|---|---|---|---|
| `TerrainWall_HMVolume2` | `TerrainrWallHM`, hauteur stockée en 1024² (107 m/pixel) | carré 110 km, lacet -90 | Altitude 10 100 m + 350 m, EdgeFalloff 0,5 | valeur 0 partout : sol à -600 m. Le bruit planétaire garde 2 à 11 % du poids |
| `OceanWall_HMVolume` | `OceanWallHM1`, eau seule | carré 9,05 km, lacet -90 | eau 10 768 à 11 270 m | niveau d'eau **64,8 à 67,7 m** (monde) |
| `CapitalHangarLift_OceanExclusion_HMVolume` | eau seule | carré 100 m en (-141,9 ; 933,9) | eau à 10 735 m | eau baissée à 34,8 m dans le puits du monte-charge |
| `HM_WallComplex2` (outil, pas en jeu) | `ComplexWallHM1_True`, hauteur + température + humidité | carré 8 968,7 m, sans rotation, en (1,63 ; 7,63) | Altitude 10 713,52 m + 683,5 m, EdgeFalloff 0 | c'est le sol cuit |

Formule WorldScape (CPU `WorldScapeRoot_Noise.cpp:500`, GPU `WSHeightfieldVolumes.ush:430`, identiques) :
hauteur = Altitude + valeur x MapHeight ; demi-côté = échelle X x 50 cm ; z monde à l'origine = hauteur - 10 700,16 m.

### 2.2 Le sol cuit (`HM_Bake_Capital_Actor` dans `_SubLevel/L_YellowWallGround`)
- Acteur en **(10,73 ; 16,85 ; 13,59) m**, sans rotation ; 203 tuiles `Proxy` (Nanite, collision `BlockAll`) sur 256,
  les 53 manquantes dans les coins hors du mur. Les 256 `Collision_SM` ne sont pas posées par l'acteur.
- Sol cuit = 13,59 + valeur x 683,5 m, **plan** (pente de calage en r² : 0,02 m/km², une sphère donnerait -0,16).
- Dans l'enceinte (63,6 km²) : **68 % du sol est sous l'eau** (fond à 13,6 à 66 m) ; **20,2 km² de sol sec** (croissant
  nord-est, île ouest, îles sud et sud-est du Complex), altitude médiane 142 m, pente médiane 27°.

### 2.3 Écart si on remet `ComplexWallHM1_True` (carte `Capitale_YellowWall/sol_worldscape_ecart.png`)
| Cause | Écart dans l'enceinte | Effet |
|---|---|---|
| **Position** : outil en (1,63 ; 7,63), acteur cuit en (10,73 ; 16,85) | 9,1 m est, 9,2 m sud : écart moyen 2,6 m, plus de 1 m sur 53 % de la surface, plus de 5 m sur 16 % | 836 des 4 115 objets posés au sol décalés de plus de 1 m |
| **Altitude** : outil 10 713,52 m, acteur à 10 713,75 m | 0,23 m | négligeable, autant caler juste |
| **Courbure** : WorldScape suit la sphère (rayon 3 169 km), le bake est plan | 1,6 m en moyenne, **3,2 m au mur**, 6,4 m dans les coins | 2 679 objets posés au sol flotteraient de plus de 1 m, 1 524 de plus de 2 m |

Volume posé sur l'acteur cuit + heightmap qui compense la courbure (lot 1) : l'écart tombe à l'interpolation près.

### 2.4 Voies ZoneGraph, routes, bâtiments
- **Voies ZoneGraph** (`L_ZoneGraph_Merging`, 239 853 points) : **aucun point à moins de 2 m du sol cuit** ; 99 % à
  plus de 20 m au-dessus (paliers 96, 113,6, 137, 155, 175 et 650-670 m). La ville piétonne est sur des dalles
  au-dessus de l'eau. **Aucun recalage de voie n'est nécessaire.** 44 voies (0,4 km) passent sous le sol (tunnels).
- **Objets** (117 026 boîtes uniques dans l'enceinte) : **4 115 posés sur le sol**, surtout `L_YW_AirField_Sud` (1 407),
  `L_YW_UnderWater_Desert` (586), `L_Capital_CostaRive` (432), piles de `L_Capital_BaseStruct` (371),
  `L_YW_WeaponFactory` (301). Mesure sur les boîtes des acteurs, pas sur les points de contact exacts.

### 2.5 Ce que lit le spawn planétaire QAI
- `QAI_Planetary_Manager` : `GetPawnDistanceFromGround`, `WS_GetTemperature/Humidity`. `QAI_Planetary_Client` :
  trace physique `CyTraceSphereByChannelSingle` + `GetDistanceFromWater`. Pions : `TryGetWorldScapeGround`
  (`QAI_FloatingPawnMovement.cpp:1189`) et `ResolveWorldScapeTerrainPenetration` (remonte un pion passé sous la surface).
- Aujourd'hui ces requêtes voient la cuvette à -600 m sous l'eau : très probablement la raison pour laquelle le spawn
  planétaire ne donne rien dans l'enceinte (non vérifié en jeu). Après : les 20 km² d'îles deviennent un sol valide ;
  le centre (dalles au-dessus de l'eau) reste rejeté par le test d'eau. Restent à exclure les zones construites au sol
  des îles (aérodrome sud, usine, carrière).
- Le filtre de biome QAI lira le nouveau climat : avec une humidité basse, les animaux (`Min_Humidity` 0,10 à 0,30)
  seront rejetés (cf. mémoire `qai-planetary-spawn-system`). À regarder avec le choix du climat.

### 2.6 Le climat (carte `Capitale_YellowWall/sol_worldscape_climat.png`)
- **`ComplexWallHM1_True` : température et humidité inutilisables.** La source `coqdzqdpy` est en RGBA16F avec des
  valeurs au-dessus de 1 (vert jusqu'à 15,9, bleu jusqu'à 2,8). À la conversion en 16 bits, elles ont été **repliées
  modulo 1** (vérifié : écart nul avec ce repli). 41 % des pixels de température et 21 % d'humidité sont touchés : on
  obtient des anneaux concentriques au sud. Utilisé tel quel, le feuillage pousserait en cercles.
- **Pas d'autre climat dans les assets voisins** (vérifié pixel à pixel) : `ComplexWallHM1_Good_Tex2D` a G = B = R,
  une ancienne hauteur en niveaux de gris (37 m d'écart moyen avec le bake) ; les assets les plus récents,
  `ComplexWallHM1_copy` et `ComplexWallHM1_cEzdopy` (13/10/2026), en sont des copies (hauteur = température =
  humidité). `ComplexWallHM1_True_Backup_Tex2D` a aussi des valeurs au-dessus de 1.
- **Climat retenu (Benjamin : « l'asset le plus récent »)** : la texture la plus récente qui porte un climat,
  **`ComplexWallHM1_coqdzqdpy_Tex2D`** (29/09/2023, la source du bake), canaux G (température) et B (humidité),
  **écrêtés à 0..1** au lieu d'être repliés. Résultat : le nord froid et humide, le sud chaud et sec, sans anneaux.
  Sur les îles sèches : température médiane 0,72 (bornes 0,26 à 1 du volume de l'outil), humidité médiane 0,04 ;
  53 % des îles ont une humidité sous 0,10 (aucun animal QAI éligible là, cf. §2.5). Variante « divisé par le max »
  mesurée aussi : plus froide (médiane 0,29), à écarter sauf avis contraire.
- **Climat simulé (demande de Benjamin du 19/11, retenu par lui le 19/11)** : la peinture d'origine est faite au pinceau et presque binaire
  une fois écrêtée. On la remplace par une micro-simulation hors moteur
  (`Saved/Claude/YellowWall/climat/climate_sim.py`, jamais en jeu) qui part du récit : **le mur entretient
  l'écosystème**, nord froid, sud chaud, bande tempérée au centre.
  - Température : chaleur diffusée depuis l'anneau du mur (froid au nord, chaud au sud, doux à l'est et à l'ouest),
    modérée par la mer, refroidie par l'altitude (environ 6,5 °C par km), réchauffée sur les versants exposés au sud,
    air froid qui descend dans les creux.
  - Humidité : air humide soufflé par le mur nord, séché par la chaleur, versants au vent (nord) plus humides, versants
    sous le vent plus secs, creux et vallées humides, crêtes sèches, influence large de la mer.
  - **Mêmes parts de biome que la peinture**, terres et eau à part. Sur les terres : 45,3 % froid (sous 0,33),
    5,1 % tempéré, 49,6 % chaud (au-dessus de 0,66) ; 56,3 % sec, 1,7 % moyen, 42,0 % humide. À l'intérieur de chaque
    biome, les valeurs sont continues et suivent le relief.
  - Aperçu : `Capitale_YellowWall/sol_worldscape_climat_simule.png`. Fichiers 16 bits 4096², même cadrage que la
    hauteur (ligne 0 = nord) : `Saved/Claude/YellowWall/climat/YW_Climat_Temperature_4096.png` et
    `YW_Climat_Humidite_4096.png` (hors arbre synchronisé, rien d'importé). Paramètres ajustables en tête des sections
    du script.

### 2.7 Qui utilise le sol cuit (recherche par chaîne sur tous les `.umap`, `Config`, sources)
| Fichier | Utilise | Conséquence |
|---|---|---|
| `Maps/CompositeUniverse/_SubLevel/L_YellowWallGround` | `HM_Bake_Capital_Actor` | le level du jeu : à retirer de la sphère `SL_YellowWallTerrain` |
| `Maps/Universe/Sub_Levels/L_Steaming_On_Earth` (+ `_Keep`) | `HM_Bake_Capital_HLOD` | mesh affiché de loin (sphères PI et Terrain) : à retirer, le terrain GPU le remplace |
| `Maps/Introduction/SubLevelsIntroduction/L_Intro_Ground` | `HM_Bake_Capital_Actor` + `HM_Bake_Capital_Intro` | **l'introduction utilise le sol cuit** (et `L_IntroductionPersistent` utilise l'ancien `ComplexWallHM1`) |
| `StarMap/_Data/LevelMap/Earth/Capital/Capital_Map_3D` | `HM_Bake_Capital_HLOD` | **la carte 3D de la StarMap affiche ce mesh** |
| `_QLevel/.../Exterieur/L_MUR_EXTERIEUR`, `_QLevel/.../YellowWall_OLD/SubLevel/L_YellowWallGround`, `Developers/SpaceProject/landscapefinetune`, `_LevelTest/L_Multiplayer_TestLevel` | acteur ou nom | construction et tests |
| `Config/EasyCookGenerated.ini` | 409 lignes, en commentaire (`;`) | à vérifier au cook |

**Donc : on retire le sol cuit de la chaîne Univers sans supprimer les assets** tant que l'introduction et la StarMap
s'en servent. Le stockage ne se libère qu'après leur passage (lot 6). Sur disque : `Proxy_SM` 409 Mo, `Collision_SM`
240 Mo, `HM_Bake_Capital_Tex2D` 121 Mo, soit **environ 770 Mo de sources**. Et `ComplexWallHM1_True` (400 Mo) n'a pas
à être cuit si on passe par la copie du lot 1.

## 3. Collision 100 % WorldScape

- La collision WorldScape se construit sur le CPU (`ColisionGeneration::CalculateNoiseSample`,
  `WorldScapeRoot_Collision.cpp:443`), autour de chaque joueur (3 x 3 tuiles de 16 x 2 m avec les réglages par défaut
  d'`EarthScape`), pour tous les joueurs sur le serveur dédié, **seulement à moins de 141 m du sol WorldScape**.
  Accepté par Benjamin.
- Changement à prévoir : aujourd'hui, sur les dalles (96 à 175 m), le sol est à -600 m et **aucune tuile n'est générée en
  ville**. Avec le vrai relief, il sera à 13-66 m sous les dalles : des tuiles se génèrent sous la ville pour chaque
  joueur, serveur compris. Coût à mesurer au lot 7 (on ne marche pas dessus, mais il est payé).
- Points de collision fine à surveiller : tuiles de 2 m (le bake était à 2,2 m) ; véhicules ; objets lancés loin des
  joueurs (pas de collision terrain au-delà des 3 x 3 tuiles, comme partout ailleurs sur la planète).

## 4. Plan par lots

Jours de session, hors validation de Benjamin et compilation de RzZz. Chaque lot finit par une mesure.

| Lot | Contenu | Durée | Écrit dans le projet ? |
|---|---|---|---|
| 0 | **Mesurer en éditeur** (§6), lecture seule | 0,5 j | non |
| 1 | Données de la capitale : hauteur tirée de `_True`, **4096²** (même densité que le bake, 2,2 m), courbure compensée (+r²/2R, MapHeight relevé à 690 m) ; température et humidité = climat simulé (§2.6), 4096². Calcul hors moteur, import en éditeur d'un nouveau HeightMapVolumeData. Vérif hors moteur : écart au sol cuit < 0,1 m | 0,5 à 1 j | 1 texture + 1 HeightMapVolumeData (nouveaux) |
| 2 | Volume dans `L_Dev_Claude` (même planète, même bruit) : posé sur l'acteur cuit (10,73 ; 16,85), échelle 8968,686523, Altitude calée sur z 13,59, EdgeFalloff 0, Override, hauteur + température + humidité. Comparaison avec une copie du sol cuit, journaux GPU | 1 j | `L_Dev_Claude` seulement |
| 3 | Masques de feuillage (`FoliageMaskVolume`) sur les zones construites au sol : aérodrome sud, usine, carrière, CostaRive, piles de BaseStruct, Complex | 0,5 à 1 j | nouveaux volumes |
| 4 | Intégration Univers : volume dans `HeightMapVolumeList` d'`EarthScape` **après** `TerrainWall_HMVolume2` ; `L_YellowWallGround` retiré de `SL_YellowWallTerrain` ; `HM_Bake_Capital_HLOD` retiré des listes de `L_Steaming_On_Earth` | 0,5 j | persistant de jeu + streaming (accord explicite ; RzZz a modifié le persistant le 14/11) |
| 5 | QAI intra-muros (lot U du rapport piétons) : spawn sur les îles, exclusions des zones construites, intérieurs sous le sol | 1 j | volumes QAI |
| 6 | Introduction et StarMap : décider (garder le cuit pour elles, ou passer l'intro au volume et la StarMap à un mesh plus léger), puis seulement supprimer `Baking_Capital` | 0,5 j + décision | levels intro / StarMap |
| 7 | Test serveur dédié + 2 clients : collision (tuiles générées sous la ville, véhicules), mémoire, temps de frame, journaux GPU | 0,5 j | non |
| **Total** | | **5 à 6,5 j** | tenable avant le 17/12 |

## 5. Risques

1. **Atlas GPU et mémoire** : `ComplexWallHM1_True` complet = 384 Mio dans l'atlas GPU des volumes, plafonné à 16 384²
   soit 512 Mio (`WorldScapeRoot_Main.cpp:1396`), qui contient déjà le jeu de l'Univers (169 Mio de sources d'après le
   code). Il déborderait et des masques seraient abandonnés sur le GPU. La copie 4096² (trois masques, 96 Mio) passe ;
   le moteur sait réduire chaque masque à part (`MaxHeight/Temperature/HumidityResolution`).
2. **Position** : copier le volume de l'outil tel quel décale tout de 13 m en diagonale (§2.3).
3. **Courbure** : sans compensation, jusqu'à 3,2 m de vide sous les objets posés près du mur.
4. **Climat de `_True` replié** (§2.6) : feuillage en anneaux si on le prend.
5. **Feuillage** : aucun masque ni volume de trou autour de la capitale aujourd'hui ; sans le lot 3, il pousse sur
   l'aérodrome, l'usine et les routes au sol.
6. **Aspect** : le bake utilise `/Game/Maps/LandscapeInst_Inst`, le terrain le matériau planétaire. Les îles changent.
7. **Tunnels et intérieurs sous le sol** (puits du monte-charge, `L_Usine_Autonome_Stream`, 44 voies en tunnel) : le
   terrain les recouvre, comme le bake, mais sans trou la collision WorldScape peut bloquer une entrée, et la remontée
   QAI peut sortir un pion. Reporté par Benjamin.
8. **Introduction et StarMap** utilisent le sol cuit (§2.7) : ne pas supprimer les assets avant le lot 6.
9. **Couture au bord du carré** : sur les axes cardinaux le carré s'arrête au pied du mur ; dehors le terrain est à
   -255/-600 m, donc une marche sous l'eau. Elle existe déjà avec le bake ; à regarder en éditeur.
10. **Ordre des volumes** : avant `TerrainWall` dans la liste, le nouveau volume serait écrasé.

## 6. Reste à mesurer en éditeur (lot 0, lecture seule)

- Hauteur réelle du terrain WorldScape dans l'enceinte (grille `GetGroundNoise`) : confirmer la cuvette.
- Ligne `WS IndirectTerrain` de l'atlas (taille, masques abandonnés), avant et après le volume.
- Feuillage présent aujourd'hui dans l'enceinte, et sur `L_Dev_Claude` avec le nouveau climat.
- Journal : `WS_FOLIAGE_GPU run (gpu)` avec 0 secteur sur le bruit CPU, `forced_cpu=0`.
- `QAI.Debug 1` en marchant sur une île, avant et après.
- Collision : tuiles WorldScape générées sur les dalles (attendu 0 aujourd'hui).

## 7. Fichiers

- Cartes : `Documentation/Capitale_YellowWall/sol_worldscape_ecart.png` (terrain actuel, sol cuit, écart),
  `sol_worldscape_zones.png` (eau, îles sèches, voies ZoneGraph), `sol_worldscape_climat.png` (climat replié de
  `_True` et climat retenu).
- Scripts de mesure : dossier de travail de la session (extraction des tableaux `Stored*`, décodage Oodle des textures,
  calage des tuiles, comparaison). À copier dans `Saved/Claude/uasset_offline/` si on les garde.
- Doc à corriger après validation : `YELLOWWALL_PEDESTRIANS_STATE.md` (« les voies devront être recalées » est faux) et
  `CAPITALE_YELLOWWALL_ARCHITECTURE.md` §4 bis (`TerrainrWallHM` creuse l'enceinte à -600 m ; `HM_WallComplex2`).
