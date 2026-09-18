# Porter les systemes de surface de ScapeEngine dans WorldScape

Analyse du 2026-09-18. Lecture de code et d'assets uniquement : aucune modification, aucun build, aucune mesure en jeu (editeur ferme, pont CLIScape ferme sur le port 8766, verifie). Chaque affirmation cite le fichier lu ce jour. Les chiffres marques « mesure » viennent de sondes ecrites par Benja dans ScapeEngine ou de commentaires de mesure laisses dans le code ; ceux marques « calcul » sont derives de ces mesures et le calcul est donne.

Sources : `C:\NebulaEngine` (ScapeEngine, hors `build/`, `external/`, `out/`) et `C:\UE5_Projects\QANGA`.

**Note de methode.** Une premiere passe de cette analyse a conclu a l'absence de champs, de fermes procedurales, de rues sur WorldScape et de villes. **Ces quatre conclusions etaient fausses** : une recherche par nom de fichier manquait tout ce qui vit sous `Content/_QLevel/` et sous `Plugins/*/Content/`. Un scan complet les a trouvees, et je les ai re-verifiees une a une ce jour. Elles sont corrigees dans le texte, et le cadrage general en est change : **il manque des regles, pas des briques**. C'est la regle 10 du `CLAUDE.md` en action, et elle a servi deux fois sur ce document.

---

## 0. Ce qui decide, en sept constats

| # | Constat | Consequence |
|---|---|---|
| 1 | **ScapeEngine n'a pas sept systemes qui deforment le sol : il en a un.** `ComposeHeightSphere` (`ScapeEngine/src/scape/TerrainNoiseCPU.h:387-475`) est la seule fonction de hauteur, avec six etages, et le carve routier est le dernier, en ecrasement (`:451-472`). Tout passe par un champ unique, `RoadField.h` (758 lignes), avec quatre types d'empreintes et quatre consommateurs qui doivent l'evaluer a l'identique (`RoadField.h:242-245`). | Le portage a **un** point d'insertion, pas sept. Cote QANGA c'est `AWorldScapeRoot::GetNoise` ([WorldScapeRoot_Noise.cpp:213](../Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScapeRoot_Noise.cpp)) et son miroir `WSHeightfieldGenerate.usf`. |
| 2 | **La Terre de QANGA est 112 fois plus grande que la planete de ScapeEngine.** Rayon 3 169 km contre 300 km (`Documentation/WorldScape_GPU_Completion_Plan_2026-09-01.md` section 1 ; `ScapeEngine/src/scape/FieldPatternCPU.h:176`). Soit 1,26e8 km2 contre 1,13e6 km2 : **229 Frances contre 2,1 Frances**. | Un reseau routier planetaire a la densite ScapeEngine (19,7 segments/km2, mesure) ferait **2,5 milliards de segments, 119 Go** a 48 octets par segment (calcul). Rédhibitoire. Le reseau doit etre **local aux lieux**, pas planetaire. C'est le constat que la premiere passe de cette analyse avait manque. |
| 3 | **Le serveur ne stocke pas de geometrie, et il n'en a pas besoin : il connait le monde par des fonctions.** `GetEffectiveUseGPUNoise()` retourne `false` sur serveur dedie, avec le commentaire « the CPU noise is the only terrain knowledge there (collision, foliage, QAI) » ([WorldScapeRoot_Main.cpp:296-306](../Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScapeRoot_Main.cpp)). Et le code discrimine deja finement : `Grid_SkipStaticMeshOnDedicated` ([WorldScapeRoot.h:771](../Plugins/WorldScape/Source/WorldScapeCore/Public/WorldScapeRoot.h)), `SpawnableOnDedicated` par DataAsset, `bGenerateOnServer` par entree de foliage, `EnabledOnDedicated = false` sur QInstanced. | **Regle d'architecture du chantier** : tout ce que le serveur doit savoir de la surface est une **fonction analytique evaluable depuis une position**, jamais un asset. C'est exactement ce qu'est `RoadField.h`. Le ruban de mesh, les props, les batiments visuels sont **purement client**. |
| 4 | **Le mecanisme des volumes heightmap ne monte pas en charge.** Le gather ignore toute boite : `const FBox Everything(FVector(-1.0e15), FVector(1.0e15))` ([WorldScapeRoot_Main.cpp:725](../Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScapeRoot_Main.cpp)), donc tous les volumes partent au GPU et le shader boucle sur **tous** pour **chaque** texel, les trous **sans test de distance** ([WSHeightfieldVolumes.ush:386](../Plugins/WorldScape/Shaders/WorldScapeGPUTerrain/Private/WSHeightfieldVolumes.ush), [:544](../Plugins/WorldScape/Shaders/WorldScapeGPUTerrain/Private/WSHeightfieldVolumes.ush)). A 24 volumes + 8 trous c'est negligeable. | Un systeme de routes par empreintes de volume est **exclu d'emblee**. ScapeEngine le dit aussi de son cote : cap de **32 stamps pour toute la planete** (`ScapeEngine/src/scape/TerrainComputeDispatch.h:249`). La bonne primitive est un champ collecte spatialement, pas N volumes en boucle lineaire. |
| 5 | **Champs, fermes, rues et villes existent DEJA, poses sur WorldScape, en production.** Pas le patron : les instances. Un champ de tournesols avec stades de croissance et etat recolte, colle a la surface (`Content/_QLevel/Universe/Planetary/Farm/_Snap/Field_01/Farm_Field_Subflowers_01.uasset`, **17,5 Mo**, verifie ce jour). **Onze** revetements de rue et de sol en Snap a trois echelles (Micro, normale, Big) dans `OLD_Town/Snap_Ground/`. **Huit** `UQLevel_AssetCollection` de ferme (barrieres A1 a A5, batiments A1 a A4 et B1 a B2, bottes, jardin, poteaux, epouvantails, arbres), chacune avec son proxy bake. **Treize** quartiers de Djibouti avec leur niveau de detail. Et a l'echelle du projet : **70** `WorldScape_GridDataAsset`, **13** `WorldScape_SnapDataAsset`, **471** assets `Q_*` dans `_QLevel`, contre **6** collections de foliage. | **Ce qui manque n'est pas une brique, c'est une regle.** Le probleme « poser un revetement de route sur un cube-sphere en double precision » est **deja resolu en production**. Le Grid Spawner est le systeme de placement de ce jeu ; le foliage est marginal. C'est le constat qui change le plus le decoupage. |
| 6 | **`UWS_FunctionLibrary` est la brique la plus sous-exploitee du plugin** et contient deja les primitives d'un generateur de routes : projection de masse avec normale, echantillonnage de pente, grille 2D orientee **avec variante asynchrone**, difference de hauteur sur une emprise (`HDifference`, `Median`), `WS_FindMinimalGroundH` + `WS_OverrideLocationH` (aplanissement), et deux fonctions de Bezier cubique ([WS_FunctionLibrary.h:29-97](../Plugins/WorldScape/Source/WorldScapeCore/Public/WS_FunctionLibrary.h)). | Le solveur de trace et l'aplanissement d'emprise ont deja leurs primitives. Il manque l'orchestration, pas les outils. |
| 7 | **`WorldScapePCG` est `Type: "Editor"`** ([WorldScape.uplugin:106-115](../Plugins/WorldScape/WorldScape.uplugin)), donc jamais charge dans un build `Game`, `Client` ou `Server`. Le module fait 441 lignes et le noeud de test echantillonne vraiment la hauteur, mais point par point en `float`, sans normale de terrain ni aucune contrainte. | **PCG oui, mais comme outil d'atelier qui cuit des assets, jamais comme runtime.** Verdict argumente en section 6. |

**Reponse courte a la contrainte dure.** Sur les sept systemes demandes, **cinq ne touchent pas la hauteur du sol** et sont donc sans risque pour la Terre batie : champs (nivellement a 0 par defaut, et c'est verrouille par un commentaire explicite), fermes et bati agricole, mobilier de bord de route, ponts (par construction : `RoadSeg_NoCarve`), tunnels (inexistants, seuil a 0 sur les quatre tiers). **Deux la touchent** : le relief lui-meme, et les routes. Pour ces deux la, le garde-fou existe et il est mesurable ; il est decrit en section 3.

---

## 1. Inventaire ScapeEngine

Volumetrie du chantier, `wc -l` verifie. Total lu ou recense : environ 36 000 lignes de code de surface, dont **environ 18 000 portables** (headers purs, qui compilent seuls contre `ScapeEngine/src` sans flecs, sans Diligent, sans editeur : propriete entretenue et mesuree, `ScapeEngine/tools/probes/README.md:3-5`).

### 1.1 Le fait structurant : une seule fonction de hauteur

`ComposeHeightSphere()`, `ScapeEngine/src/scape/TerrainNoiseCPU.h:387-475`, six etages dans cet ordre exact :

| # | Etage | Ligne | Portee |
|---|---|---|---|
| 1 | `FieldFlattenAt` (nivellement agricole) | `:395` | globale, mais **inerte par defaut** |
| 2 | `EvaluateMacroHeightMeters` (continents, montagnes, collines) | `:396` | base |
| 3 | bande de detail, relative a `detailOrigin` | `:402-409` | base |
| 4 | DEM heightmap | `:412-431` | remplace la base |
| 5 | delta de sculpt | `:434-440` | additif, **casse : rendu seul, ni collision ni foliage, non persiste** (`TerrainCollisionSystem.cpp:467`) |
| 6 | **carve routier** : segments + disques de jonction + sols de ville + stamps | `:451-472` | locale, **en ecrasement** |

L'etage 6 est un unique `h = lerpf(h, rf.height, rf.weight)` (`:471`), miroir GPU `TerrainComputeShader.hlsl:295-296`. Commentaire de l'auteur (`:443-444`) : une route ecrase toute autre source de hauteur, parce que c'est une structure batie, pas une modulation du sol.

### 1.2 Les sept systemes

| Systeme | Etat reel | Modele | Produit | Quand | Touche la hauteur ? |
|---|---|---|---|---|---|
| **Terrain et relief** | complet, pilier de production | cube-sphere 6 faces, quadtree par face, fBm/ridged sur la direction unitaire (`TerrainNoiseCPU.h:345-355`) | heightmap R32F par tuile (`TerrainComputeShader.hlsl:299`) | streaming de tuile, compute async en ring de 8 (`TerrainComputeDispatch.h:152`) | **c'est lui** |
| **Champs et cultures** | v1 livree et mesuree ; crop shell **non commence** (grep `CropShell` sur `src/` et `shaders/` : 0 fichier) | champ scalaire pur fonction de la direction, adressage cube-face, 4 styles (`FieldPatternCPU.h:318`) | un masque + 14 metadonnees, **aucune geometrie** (`:234-245`) | par pixel au rendu, par candidat au scatter, par appel CPU | **oui globalement, mais a 0 par defaut** : `flattenStrength = 0` (`:193`) |
| **Fermes et bati agricole** | 7 types livres, chemin ville quasi invisible (0,5 % des ilots) | rejet dans un semis clipmap, 11 modes de filtre (`FoliageTypes.h:162-192`) | instances de props | au scatter | **non** |
| **Routes et hierarchie** | complet, le plus abouti du depot, 4 tiers | graphe (sites hash, aretes de Gabriel, chemins routes) converti en champ scalaire signe puis en mesh | hauteur carvee, masque et peinture par pixel, mesh de ruban, plateformes | 3 cadences : generation a 8 km de derive, ruban tous les 300 m, carve au streaming de tuile | **oui localement, par 4 mecanismes** |
| **Mobilier de bord de route** | socle complet et extrait (`RoadsideAnchor.h`, 1213 l.) | abscisse curviligne sur une chaine 1-D, explicitement **pas** un champ 2-D (`:16-24`) | instances de props, catenaires | a la publication du reseau, puis fenetre glissante | **non** : socle pur, « no terrain » (`:22-27`) |
| **Ponts et tunnels** | **ponts complets ; tunnels inexistants** | decision par evenement sur le profil en long, un seul verdict par course de spans | tablier (profil ferme, caisson) + piles, `pierSpacing = 60 m` (`RoadMeshBuilder.h:194-199`) | avec le ruban | **non, c'est l'interet** : `RoadSeg_NoCarve` fait `continue` dans la boucle de champ (`RoadField.h:76`, `:503`) |
| **Ville** | plan, mesh et collision des batiments complets ; pas de trafic, pas de LOD par part | emprise, cordes arterielles, BSP par super-cellule, ilots, parcelles mitoyennes (`CityPlan.h`, 1466 l.) | mesh, sols en eventail drape, props, boites de collision orientees, **grille de hauteur 64x64** | plan a la generation, mesh a la fenetre de ruban | **oui localement, mais la ville drape, elle ne nivelle pas** |

**Tunnels : ne pas les compter comme une fonctionnalite existante.** `tunnelThreshold = 0.0f` sur les quatre tiers (`RoadTierTable.h:96`, `:142`, `:174`, `:305`). Le chemin de code existe et est atteignable (le meme code sert pont et tunnel, `RoadNetworkGenerator.h:1992-1993`), mais le seuil ne le declenche jamais. Consequence assumee et mesuree : la profondeur de deblai n'est bornee par rien, **77 m** en nationale, **65 m** en locale, **97 m** en autoroute (`docs/ROAD_HIERARCHY.md:163-166`).

### 1.3 Les couts mesures qui comptent pour un portage

Routes, decomposition du gel d'origine puis apres passage en asynchrone (`docs/ASYNC_GENERATION.md:9-16`, `:63-68` ; `docs/GENERATION_STATE.md:443-451`) :

| Etape | Avant | Apres async | Apres correction de fenetre |
|---|---|---|---|
| Generation | 744 ms dans l'image | 673 ms hors image | **191 ms** |
| Resolve | 832 a 1210 ms | 995 a 1524 ms hors image | **153 ms** |
| Construction du ruban | 1422 a 1947 ms | mediane **0,36 ms**, pire 3,10 ms sur le thread principal | idem |
| Gel tous les 8 km | 3 a 4 s, jusqu'a 12,6 s | | **0,34 s** |
| Segments publies | 362 679 | | **55 561** |

Super-linearite mesuree : a densite x2,6, resolve 3921 ms et ruban 9609 ms, soit **x4,9 segments pour x22 et x78 de cout** (`ASYNC_GENERATION.md:70-72`). La conclusion de l'auteur est la bonne lecon a retenir : optimiser ne suffit pas, il faut sortir le travail de l'image.

Ville : **116 ms synchrones** quand une ville entre dans la fenetre, et c'est le drapage, environ **10 700 requetes de hauteur par ville** (`GENERATION_STATE.md:511-513`). Sonde headless sur 55 035 ilots et 400 villes : cout ajoute par ville p50/p90/max = 11 483 / 61 541 / 138 135 sommets pour une arene de 1,2 M, et 1 180 / 5 471 / 10 459 sondes de sol (`docs/CITY_DESIGN.md:548-563`).

Champs, sonde sur 400 000 echantillons (`docs/FIELD_SYSTEM.md:317-343`) : **0 divergence sur 120 000**, 46,4 % de la surface en region agricole, 44,8 % de sol cultive, 9,4 km de limites par km2 soit 4 702 arbustes/km2 a 2 m d'espacement. Tailles medianes ponderees par l'aire : bocage 2,9 ha, openfield 6,8 ha, grid 25,0 ha, pivot 44,5 ha.

Mobilier, boot propre sur `L_RoadTest` (`docs/POLE_SYSTEM.md:321-334`) : 293 poteaux, 445 travees, 1729 runs, 293 sondes de sol en **3,8 ms dont 1,8 ms de sondes**.

Caps GPU durs, a redimensionner cote UE (`ScapeEngine/src/scape/TerrainComputeDispatch.h`) : 2048 segments de carve par tuile (`:157`), 256 disques de jonction (`:239`), 8 sols de ville (`:243-245`), **32 stamps pour toute la planete** (`:249`), 160 segments par tranche de peinture (`TerrainSystem.h:2363`), grille de sol de ville 64x64 (`RoadField.h:352`).

### 1.4 Le determinisme, et ses deux exceptions

Determinisme pur-hash, invariant n.1 ecrit du domaine (`docs/GENERATION_STATE.md:364-366`) : tout est fonction du hash de cellule ou de site, aucune relaxation iterative, aucun couplage inter-sites. Les hashes d'ensemble sont **insensibles a l'ordre** expres (`road_perf_probe.cpp:178`, `RoadSystem.h:703`). Verifie comme critere d'acceptation sur quatre systemes independamment : champs 0 divergence sur 120 000, eoliennes 144 machines recomparees et ecart max 0,00e+00 m, ville stable sur trois replanifications, chaines de crete 204 depuis deux fenetres a 40 km sans divergence.

Deux sources d'etat **non** regenerables par une graine, a traiter comme de la donnee et non comme de la generation : les **stamps d'auteur** (contenu de niveau, persiste dans `SubLevelRef`, `docs/TERRAIN_STAMP.md:139-147`) et le **DEM heightmap**.

Une reserve explicite de l'auteur, non levee : `FastNoise2` n'est pas prouve thread-safe (`ASYNC_GENERATION.md:95-98`). La bibliotheque est dans `external/`, hors perimetre : **non verifie**.

### 1.5 Les six pieges deja payes, qui se reproduiront en UE

1. **Rendre une grandeur par-segment variable casse tout ce qui l'agregeait par un `max`** (`ROAD_HIERARCHY.md:157-159`). Coût paye : un unique span en deblai profond a tire tout le reseau **quatre niveaux de LOD** plus grossiers (`depth 13, 239 segments` devenu `depth 7, 6043 segments`). Chercher les `max(...)` **avant**, pas apres.
2. **Une sentinelle se verifie sur tous ses consommateurs** (`GENERATION_STATE.md:381-383`). `radius <= 0` oublie dans un seul endroit a aplati chaque carrefour de chaque ville en escalier de plateaux de 39 m.
3. **Les egalites ne se departagent jamais sur l'ordre de liste** (`RoadField.h:713-723`). Deux stamps a poids 1 departages par « collecte en premier » donnent une divergence rendu/collision silencieuse. **En multijoueur server-authoritative c'est la classe de bug n.1** : serveur et client trient deux tableaux dans deux ordres, et le sol n'est pas au meme endroit.
4. **Une sonde ne porte jamais sa propre copie de l'algorithme mesure** (`WIND_FARM.md:386-390`). Deja paye deux fois ; `road_profile_probe.cpp` est encore perime aujourd'hui (`tools/probes/README.md:48-52`).
5. **Une pente a trois chiffres n'est pas une falaise, c'est un eclat invisible** (`ASYNC_GENERATION.md:45-49`). Un `max` sur tout span de plus de 1 m trouve 605 % et 1200 % sur un generateur non modifie. Toujours distinguer le brut du carrossable.
6. **Lire le 2e rebuild, jamais le 1er** (`ASYNC_GENERATION.md:22-25`). Le premier coûte 819 ms pour une raison structurelle et n'est pas un cout par rafraichissement.

Et un septieme, propre au sujet : `terrain_lod_benchmark.cpp` **n'est pas une mesure**. Il n'inclut aucun header moteur et dit lui-meme « estimation heuristique » ; ses recommandations sont des conjectures arithmetiques sur un rayon de 6371 km alors que la planete de reference en fait 300. A ne pas utiliser pour chiffrer quoi que ce soit.

---

## 2. Inventaire WorldScape et QANGA

### 2.1 Les 10 modules

| Module | Fichiers | Lignes | Role reel |
|---|---|---|---|
| `WorldScapeCore` | 61 | 30 686 | tout le moteur : `AWorldScapeRoot`, clipmap CPU, collision, foliage runtime, **Grid Spawner**, **Snap Spawner**, pooling, subsystem, gravite |
| `WorldScapeNoise` | 42 | 10 216 | les 5 classes de bruit + `FNoiseData` |
| `WorldScapeGPUTerrain` | 12 | 6 793 | quadtree GPU cube-sphere, heightfield manager, vertex factory, renderer indirect |
| `WorldScapeCompute` | 11 | 4 949 | dispatch des compute de bruit |
| `WorldScapeVolume` | 23 | 2 473 | `AHeightMapVolume`, `ANoiseVolume`, `ATerrainHoleVolume`, `AFoliageMaskVolume`, `AWS_Grid_InvalidSphere` |
| `WorldScapeCommon` | 12 | 1 786 | `DVector`, math de bruit, Poisson disc |
| `WorldScapeFoliages` | 16 | 1 403 | **uniquement des DataAssets et des interfaces**, aucune logique de placement |
| `WorldScapeFactory` | 33 | 1 327 | factories editeur |
| `WorldScapePCG` | 9 | 441 | deux classes de noeud PCG et une library BP. `Type: Editor` |
| `WorldScapeEditor` | 6 | 345 | trois helpers de viewport |

A noter pour un nouveau module : `WorldScapeCore` depend **publiquement** de `QLevel` (`WorldScapeCore.Build.cs:49`). Le couplage WorldScape / QLevel est deja assume au niveau du build.

### 2.2 L'API analytique : le socle serveur

C'est le point d'appui principal, et il est riche. Dans [WorldScapeRoot.h](../Plugins/WorldScape/Source/WorldScapeCore/Public/WorldScapeRoot.h) :

```cpp
// hauteur
float          GetGroundHeight(const FVector, const bool Water = false);          // :2345
double         GetGroundHeight_HOnly(const FVector);                               // :2347
TArray<double> GetGroundHeight_HOnly_Multi(const TArray<FVector>&);                // :2349
FNoiseData     GetGroundNoise(const DVector&, const bool, const bool);             // :2267
void           GetGroundNoise_Batch(const TArray<DVector>&, TArray<FNoiseData>&, ...); // :2275
// repere local
FVector GetPawnNormal / GetPawnSnappedNormal / GetPawnTangent / GetPawnBiTangent;  // :2291-2297
float   GetPawnAltitude / GetPawnDistanceFromGround / GetDistanceFromWater;        // :2299-2303
// divers
FVector WS_GetGravityFromLocation(const FVector&, FVector& Dir, double& Force);    // :1673
bool    FindOceanGroundHitBetween(const FVector, const FVector, FVector&, FVector&); // :2335
bool    Grid_CheckBySpawnParams(const FVector&, const FNoiseData&, const FWS_Grid_SpawnParams&, const FWS_VolumeData&); // :848
```

`GetGroundNoise_Batch` tourne en `ParallelFor` ([WorldScapeRoot_Noise.cpp:128](../Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScapeRoot_Noise.cpp)). C'est le bon chemin pour tout drapage massif : les 10 700 requetes de hauteur par ville de ScapeEngine passeraient par la.

Et `UWS_FunctionLibrary` ([WS_FunctionLibrary.h](../Plugins/WorldScape/Source/WorldScapeCore/Public/WS_FunctionLibrary.h)) ajoute exactement les primitives d'un generateur de surface :

| Fonction | Ligne | A quoi ca sert ici |
|---|---|---|
| `WS_ProjectGroundLocation_Multi` | `:53` | poser N instances au sol en un appel |
| `WS_ProjectGroundLocationWithNormal_Multi` | `:58` | idem avec assiette : devers d'une route, inclinaison d'un prop |
| `WS_SampleSlopeSurface(..., SlopeMin, SlopeMax, RelativeNormal)` | `:29` | filtre « terrain constructible » |
| `WS_SampleSurfaceH(..., HDifference, Median, moyenne)` | `:32` | filtre « assiette plate » ; et **mesure de portee a franchir** pour un pont |
| `WS_Sample_2DGrid` / `WS_Sample_2DGrid_Async` | `:38`, `:40` | grille orientee projetee : base d'un rang de culture ou d'une emprise |
| `WS_FindMinimalGroundH` + `WS_OverrideLocationH` | `:89`, `:91` | **le couple exact d'un aplanissement d'emprise** |
| `WS_CubicBezierY` + `WS_GetSplineOffset` | `:95`, `:97` | seul code de courbe deja present cote WorldScape |
| `WS_GetGridCoord` / `WS_GetAdjacentGridCoord` | `:47-49` | decoupage cube-sphere stable, independant du LOD visuel |

**Cote GPU, aucun point d'entree.** Le seul shader de sampling, `WSHeightfieldSample.usf:2`, s'annonce comme un harnais de validation CPU contre GPU ; son unique appelant est le chemin de validation du renderer (`WSGPUTerrainRenderer.cpp:301`, `:325`, `:1524`, `:1536-1539`). Aucun readback, aucune fonction publique de sampling dans `WSHeightfieldManager.h` qui n'expose que `RequestTile`, `ReleaseTile` et `Tick`. Si un placement GPU est voulu un jour, **il faut ecrire le pont**.

### 2.3 Le patron de production deja en place sur la Terre

`Content/Systems/WorldScape/GridData/` est organise par planete, et la Terre a neuf familles : `AccidentedPoint`, `Asteroid`, `Bunker`, `Foliage`, `Mining`, `Oasis`, `Plage`, `Sangline`, `Volume`. Exemples ouverts et verifies :

- `GridData/Earth/Bunker/Grid_Earth_BunkerCollection.uasset` est un `WorldScape_GridDataAsset` avec `SpawnType`, `SpawnParams`, `SpawnableOnDedicated`.
- `GridData/Earth/Bunker/WS_HM_Bunker.uasset` (8,4 Mo) est un `HeightMapVolumeData`, et `HM_Bunker.uasset` est le volume qui l'applique.
- `GridData/Earth/Oasis/NewFolder/` porte une **chaine de bake complete** : `NewBaked_Heightmap_Data`, `_Tex2D`, `_Actor`, `Collision_SM/NewBaked_Heightmapx0y0_Collision`, `Proxy_SM/..._Proxy`, `HLOD/..._HLOD`.
- `GridData/_Class/QL_WS_Grid.uasset` derive de `QLevel_Actor_Instance`.

C'est **la meme structure que `Content/Baking_Capital/`** (256 tuiles de collision + 256 proxys + `HM_Bake_Capital_Data` + `HM_Bake_Capital_Tex2D` + `HM_Bake_Capital_Actor`, ce dernier etant un `BlueprintGeneratedClass`). La Capitale est donc une instance industrielle de ce patron, a la grille 16x16.

Conclusion pratique : **pour une ferme, un hameau, un champ en terrasse, il n'y a rien a inventer.** Il y a un patron a instancier, deja teste en production, deja compatible serveur dedie, deja marie a QLevel. Un precedent supplementaire existe pour les champs : `Plugins/PlanetScape/Content/Level/HM_Terrace_Fields_15_Ex.uasset`, une heightmap de champs en terrasse.

Et un precedent de hameau : `Content/Tools/WS_Tools/WS_ReplyTower_Spawner/` porte une quarantaine de niveaux pre-construits poses sur la planete (`WS_Spawn_Level_OldTown_0` a `_15`, `_GasStation`, `_AbandoCabin`, `_AbandoGarage`, `_AbandoStore`, `_AbandoSquare_01` a `_04`) avec son `RTower_LevelStreaming.uasset`.

### 2.3bis Ce qui est deja pose sur la Terre, et qu'il ne faut surtout pas reecrire

Verifie asset par asset ce jour. Volumetrie a l'echelle du projet : **70** `WorldScape_GridDataAsset`, **13** `WorldScape_SnapDataAsset`, **12** `WorldScape_SnapSpawnerActor`, **6** `WorldScapeFoliagesCollection`, **471** assets `Q_*` et **8** `UQLevel_AssetCollection` dans `Content/_QLevel/`.

| Sujet | Ce qui est deja en jeu | Ce que cela prouve |
|---|---|---|
| **Champs** | `Content/_QLevel/Universe/Planetary/Farm/_Snap/Field_01/Farm_Field_Subflowers_01.uasset`, **17 506 118 octets**, un `WorldScape_SnapDataAsset` de champ de tournesols avec stades de croissance et etat recolte, plus son niveau d'auteur `Farm_Field_01.umap` (15,9 Mo). La bibliotheque : **306 meshes** dans `Plugins/Qasset/Content/AssetStore/UltimateFarming/Meshes/`, une soixantaine de cultures declinees en `_Starter`, `_A`, `_B`, `_C`, `_Harvested`, plus serre en 12 pieces, jardinieres, cageots et cloture. Et un `BP_FieldSpline.uasset` | Un champ colle a un cube-sphere en double precision **fonctionne deja**. Ce qui manque est la **parcellisation** et l'**automatisation** : ces 17,5 Mo sont un bake fait a la main, pas une regle |
| **Fermes** | **8** `UQLevel_AssetCollection` sous `Abandonned/Farm/` : `BARRIERE` (A1 a A5), `BATIMENT/Bat_A` (A1 a A4), `BATIMENT/Bat_B` (B1, B2), `HAY_BALE` (A1 a A3), `JARDIN`, `POLE`, `SCARECROW`, `TREE`, chacune avec son `SM_..._proxy` + materiau + textures bakes. Le type est natif QLevel (`QLevel_AssetCollection.h:14`, consomme par `AQLevel_Actor_Instance`) | Ce sont les **8 seules** `AssetCollection` du projet, et elles sont **toutes** pour la ferme. Quelqu'un a deja industrialise ce sujet precis. Il manque la **regle de composition** : quel batiment, ou, oriente comment, avec quelle cour, quel acces |
| **Rues sur WorldScape** | **11** `WorldScape_SnapDataAsset` dans `Abandonned/OLD_Town/Snap_Ground/` : `Street_01` a `_03`, `BigStreet_01` et `_02`, `MicroStreet_01` et `_02`, `MicroAsphalt_01/DATA_Asphalt`, et `Ground/Trash_01` a `_03` | **Une hierarchie de rues a trois echelles est deja resolue sur terrain planetaire.** Le probleme que je pensais a inventer est en production. Il manque le graphe, le trace, les ronds-points et les raccords de carrefour, pas le revetement |
| **Villes** | `Abandonned/Djibouty/` : **13** quartiers `L_Djibouti_Big_City_01` a `_13`, chacun double d'un `_Detail_Data`, plus `L_Djibouti_Ligth_Data`. Plus `IronCity/{Bat, City, Decal, Garbage, Slums}`, `Metropolis/{Bat, City, Debris}`, `OLD_Town/{Batiment, Snap_Ground}`. Les `SnapSpawnerActor` qui les posent : `Q_L_Djibouty_City_01.uasset`, `Q_OLDTown_City_Tiny_01|02|03|05`, `Q_Human_Camp_Composite_Tiny_01|02` | Streaming, proxys, quartiers, rues, pose sur planete, HLOD : **tout existe**. Il manque le **generateur de plan** qui produirait automatiquement ce que 13 quartiers font a la main |
| **Modules de construction** | **163** DataAssets de snap dans `Content/Systems/QBuilder/Snap_Data/`, en 10 familles : `Builder`, `Celling`, `Corner`, `Door`, `Pillar`, `Ramp`, `Stair`, `Wall`, `Window`, `_Multi`. Plus `QBuilder_Snap_Actor`, `QBuilder_Snap_Component`, `QBuilder_Snap_ISM_Component`, un `QBuilder_Client_SnapMode` et un input de bascule | **Correction a mon 2.4 ci-dessous** : le snapping de QBuilder n'est pas une grille cartesienne, il est **typé par socket**, et il vit en Blueprint et DataAsset, pas en C++. Ce qui reste exact : **aucune projection sur terrain WorldScape**. C'est la seule piece a ajouter pour qu'il devienne un poseur de batiments sur planete |

### 2.4 Brique par besoin

| Besoin | Existe et reutilisable | Manque | Existe mais inutilisable en l'etat |
|---|---|---|---|
| **Relief** | 5 classes de bruit portees CPU et GPU au bit pres (`WorldScapeNoise/Public/`), chemin d'extension documente ; `AHeightMapVolume` 4 modes de sampling ; heightmap planetaire ; editeur de heightmap beta | **erosion** (ni hydraulique ni thermique), **reseau hydrographique** (pas de flow accumulation, donc pas de vallees coherentes ni de sites de pont naturels), orogenese, stratification materielle | `ANoiseVolume` : **non porte sur le chemin GPU**, un root qui en a un retombe entierement sur le clipmap CPU avec un avertissement. Disqualifiant si on veut garder le GPU |
| **Champs** | **un champ complet deja pose sur WorldScape** (`Farm_Field_Subflowers_01.uasset`, 2.3bis) ; **306 meshes** de cultures avec stades de croissance ; `BP_FieldSpline` ; `AFoliageMaskVolume` avec `SpawnVolume` et `FoliageLayerMask` ; contraintes humidite / temperature / pente / altitude par entree ; `WS_Sample_2DGrid` ; `WS_SampleSlopeSurface` ; precedent `HM_Terrace_Fields_15_Ex` | notion de **parcelle** (polygone, subdivision, rotation de culture, haie de bordure), placement **en rang**, sillons, chemins d'exploitation, et l'**automatisation du bake** (le champ existant est 17,5 Mo de transforms figes faits a la main) | le sampler du foliage est un **tirage uniforme dans une boite de secteur** (`WorldScapeRoot_Foliages.cpp:205-209`), structurellement incapable de suivre une courbe ou un rang ; le bord d'un champ sera toujours flou (rejet stochastique, `:333-341`) |
| **Fermes** | **un kit complet deja en jeu** : 8 `UQLevel_AssetCollection` avec proxys bakes (2.3bis) ; le triptyque de la section 2.3 ; `SnapSpawner` ; `EWS_Grid_SpawnType::Collection` et `::CollectionStaticMesh` ; `AWS_Grid_InvalidSphere` et `InvalidSize = 85000` pour espacer | **uniquement la regle de composition** (corps de ferme, grange orientee, silos, cour, acces sur la route), et le lien ferme vers champ et ferme vers route | `QBuilder` : 163 DataAssets de snap **typés par socket** (voir 2.3bis, cela corrige une erreur de ma premiere passe), replication et persistance versionnee, rendu ISM. Ce qui manque reellement : **aucune projection sur le terrain WorldScape**, le snap est relatif au module parent, pas au sol |
| **Routes** | **le revetement pose sur WorldScape est resolu** : 11 `SnapDataAsset` de rue a trois echelles (2.3bis) ; `WS_CubicBezierY` ; `WS_FindMinimalGroundH` + `WS_OverrideLocationH` ; `WS_ProjectGroundLocationWithNormal_Multi` ; pack `ModularRoutes` (18 meshes d'intersection, bordures a 3 echelles, terre-plein, marquages, `BP_Sidewalk_Spline_01|02|03`) ; `UQInstanced_ISMComponent::AddInstancesById` | **le graphe** (noeuds, aretes, degre de carrefour, hierarchie), **aucun rond-point** ni mesh ni generateur, solveur de trace, raccord de carrefour a plus de 2 branches, chemin de terre | `RoadTool.uasset` (769 Ko) : sa fonction terrain est `ApplyLandscapeSpline` sur un `LandscapeProxy`. **WorldScape n'est pas un `ULandscape`**, tout le volet terrassement est inoperant, et son parent `RapidRoadTool` a disparu du disque. `WS_Road_Spline_Former` : n'importe pas `WorldScapeCore`, la hauteur vient d'un `NodeHeight` saisi a la main, et ses meshes sont des SkyRoute surelevees |
| **Mobilier de bord de route** | `SnapSpawner` (le bon vehicule) ; `UQInstanced_HISMComponent` pour le budget ; l'art est deja la en quantite (barrieres, panneaux, clotures, lampadaires dont un `FarField`) ; **et cote PCG la primitive de parcours existe** : `UPCGPolyLineData::GetTransformAtDistance` et `GetTransformAtAlpha` (voir 6.6) | `PlaceAlongSpline(spline, spacing, lateralOffset, alignToTangent, projectToGround)` **cote WorldScape** : aucune des briques WorldScape ne parcourt une courbe a pas constant. Plus les regles contextuelles (glissiere selon le devers, panneau a l'approche d'un carrefour) | le foliage : sampler par boite, et les instances n'ont **pas d'identite stable** entre deux passages, donc rien de gameplay ne peut s'y accrocher |
| **Ponts et tunnels** | tunnels : `ATerrainHoleVolume` + canal `Hole` de `FNoiseData` porte jusqu'au GPU + `AFoliageMaskVolume` a superposer. ponts : `AHeightMapVolume` pour les culees, `WS_SampleSurfaceH` avec `HDifference` pour la portee, `WS_FindMinimalGroundH` pour le tablier, `UWorldscapeCollisionInvoker` et `CollisionDependantActor` pour forcer la collision sous un ouvrage ; precedent industrialise `APPROX_SM_Pillar_Route` | **tout le generateur**, dans les deux cas : portee, espacement de piles, hauteur de tablier, garde-corps, raccords ; tete de tunnel raccordee au relief ; detection automatique des sites | `ATerrainHoleVolume` est classe **experimental** et exige une modification manuelle du master material du terrain (multiplier l'alpha du vertex color en fin de lerp d'opacity mask). Il repose sur le vertex color, donc subit le meme lissage que les masques de biome quand le mesh est grossier. `Bridge.uasset` est un pont mobile de gameplay (`OpenBridge`), sans rapport |
| **Ville** | **des villes entieres deja en jeu** : 13 quartiers de Djibouti avec detail, IronCity, Metropolis, OLD_Town et ses rues (2.3bis) ; le patron `WS_ReplyTower_Spawner` ; `FWS_Grid_Proxy` pour le lointain ; pipeline HLOD rode sur la Capitale ; `ModularRoutes` ; ZoneGraph deja en production pour la foule (`L_Capital_ZoneGraph_Crowd.umap`) | **le generateur de plan seulement** : emprise, reseau de rues, ilots, parcelles, gabarits, zonage, densite decroissante. Streaming, proxys, pose sur planete et HLOD sont deja la | `PCG_City.uasset` : nom trompeur, c'est un `PCGSurfaceSampler` + `PCGWorldRayHit` + `PCGDensityFilter` dans un dossier `_GENERATED/Zouzixx/CONSTRUCTION/PCG` a cote de brouillons, date de mai 2024 |

### 2.5 Les hooks disponibles, et le piege

**Piege** : les delegues qui semblent parfaits (`WS_OnStartTerrain`, `WS_OnEndTerrain`, `WS_OnStartFoliage`, `WS_OnStartGrid`, `WS_OnStartCollision`...) sont **entierement commentes**, [WorldScapeRoot.h:440-469](../Plugins/WorldScape/Source/WorldScapeCore/Public/WorldScapeRoot.h). Ils n'existent pas.

Ce qui existe et est diffuse :

| Delegue | Declaration | Broadcast |
|---|---|---|
| `FWS_Grid_LoadCell_Delegate(const FWS_Grid&)` | `WorldScapeRoot.h:977` | `WorldScapeRoot_Grid.cpp:352` |
| `FWS_Grid_UnLoadCell_Delegate(const FWS_Grid&)` | `:981` | `:366` |
| `FWS_Grid_StartPendingLoad` / `EndPendingLoad` | `:989`, `:992` | `:438`, `:407` |
| `FOnCollisionLodsUpdate(UWorldScapeLod*)` | `:429` | `WorldScapeRoot_Collision.cpp:86` |
| `FInitializedEnd` | `:433` | `WorldScapeRoot_Main.cpp:1345` |

Il y a donc un evenement de **cellule** chargee et dechargee, pas de **tuile de terrain**. C'est suffisant et meme souhaitable : la grille WS est un decoupage cube-sphere stable en coordonnees, independant du LOD visuel. **C'est le bon ancrage pour un generateur deterministe.**

### 2.6 Les plugins voisins

| Plugin | Verdict |
|---|---|
| **`QInstanced`** | **le point d'appui obligatoire pour tout spawn massif.** 1 279 lignes. `UQInstanced_ISMComponent` et `UQInstanced_HISMComponent` derivent des composants d'Unreal et interceptent chaque ajout ; il suffit d'ecrire dedans pour s'inscrire dans la file. Budget reel du projet : `BodyCreationTimeLimitMS=0.850000` ([DefaultGame.ini:412](../Config/DefaultGame.ini)), consomme comme deadline a `QInstanced_SubSystem.cpp:115`. `EnabledOnDedicated = false`. Trois delegues d'observabilite |
| **`QLevel`** | **point d'appui obligatoire, pas optionnel.** Octree de 3,5e9 uu de demi-extent sur 26 niveaux ([DefaultGame.ini:356-362](../Config/DefaultGame.ini)). Contraintes en section 4.5 |
| **`Traffic`** (MassTraffic d'Epic) | **desactive deux fois et jamais compile** (`Qanga.uproject:389-392` a `false`, `Traffic.uplugin:13` `EnabledByDefault: false`, aucun `Binaries`). Il **a** un graphe, mais de **circulation** (ZoneGraph), pas de **construction**. Et ZoneGraph est plat, en float, aveugle a la double precision planetaire. Point d'appui credible pour la topologie de circulation **une fois** qu'un reseau existe, pas pour le creer. A noter : `MassAI`, `MassCrowd`, `SmartObjects` sont actifs et ZoneGraph est tire transitivement ; des niveaux ZoneGraph sont en production |
| **`QBuilder`** | systeme modulaire complet : **163 DataAssets de snap typés par socket** en 10 familles (`Content/Systems/QBuilder/Snap_Data/`), `QBuilder_Snap_ISM_Component`, mode client, replication et persistance versionnee. Le snap vit en **Blueprint et DataAsset**, pas en C++ : chercher `FMath::GridSnap` dans le source donne une image fausse. Ce qui manque reellement : **aucune projection sur le terrain WorldScape**, le snap est relatif au module parent. Une seule piece a ajouter, le pont vers `WS_ProjectTransformToGround`. Piege de nommage : `QBuilder_GetISMTransform(..., const bool WorldScape)` n'a **rien a voir** avec WorldScape, c'est le `bWorldSpace` d'Unreal mal nomme |
| **`QVelocity`** | hors sujet, 156 lignes |
| **`PlanetScape`** | **a clarifier avant d'ecrire quoi que ce soit.** Environ 20 000 lignes, 8 modules, compile, et **actif** (absent de `Qanga.uproject`, donc non desactive, et son `.uplugin` n'a pas de `EnabledByDefault: false`). Il contient un `WorldScapeNoiseBridge.cpp` de 711 lignes, un `PlanetFoliageSubsystem.cpp` de 822 lignes et un `PlanetTileWeightManager.cpp` de 1 305 lignes. Non evalue : hors perimetre, mais son existence chevauche le sujet |
| **`CLIScape`** | argument en faveur de PCG : `Source/CLIscape/Private/Tools/PCG/` contient `Tool_CreatePCGGraph`, `Tool_ManagePCGGraph`, `Tool_GetPCGGraphSummary`, `Tool_ExecutePCGGraph`, `Tool_SpawnPCGInLevel`, `Tool_CreatePCGPreset`, plus `Analyzers/PCGGraphDescriber.cpp`. La construction, l'inspection et l'execution de graphes sont pilotables depuis une session |

---

## 3. La question de la hauteur : le pivot

> **Cadrage revise par Benja le 2026-09-18, et il change la nature de cette section.** Deux faits qu'il apporte et que je n'avais pas : (1) **aucune heightmap ne couvre l'ensemble des zones**. Djibouti a bien la sienne, donc Djibouti est protege ; mais **les environs de Djibouti n'ont pas de heightmap et portent quand meme des objets poses**. Les 24 volumes ne sont donc pas une couverture, ce sont des ilots. (2) Le travail se fera sur **une autre version du bruit et de la Terre**, pour ne pas detruire la version actuelle.
>
> Consequence : l'objectif n'est plus « la hauteur ne bouge pas », il est **« la hauteur colle a peu pres, et les ecarts se reprennent a la main, en chirurgical »**. Cette section reste utile comme carte des mecanismes et des sensibilites, mais elle cesse d'etre un veto. Ce qui devient utile a la place, c'est l'**inventaire de ce qui est pose hors volume** : c'est cela qu'il faudra recaler.
>
> Et la cible de qualite est nommee : **une surface terrestre au niveau de Genesis**, le systeme de generation de planetes de Star Citizen. Cela deplace la priorite du lot 7 (relief, erosion, hydrographie) de « le plus risque, a ne pas ouvrir » vers **le chantier principal**.

### 3.1 Classement des sept systemes

| Systeme | Verdict | Mecanisme exact, verifie dans le code |
|---|---|---|
| Mobilier de bord de route | **ne touche pas** | socle pur, « No device, no renderer, **no terrain**, no asset loader » (`RoadsideAnchor.h:22-27`). Il **sonde** le sol, il ne l'ecrit pas |
| Fermes et bati agricole | **ne touche pas** | les props s'**asseyent** sur la hauteur composee ; meme le mode `Pivot` ne decale qu'horizontalement, « la hauteur reste celle echantillonnee au candidat » (`FIELD_SYSTEM.md:557`) |
| Ponts | **ne touche pas** | `RoadSeg_NoCarve` (`RoadField.h:76`) fait `continue` dans la boucle de champ (`:503`). Le terrain sous un pont reste intact. Les spans d'acces restent carves, c'est le remblai d'acces, et c'etait un correctif paye |
| Tunnels | **ne touche pas** (ils n'existent pas) | `tunnelThreshold = 0.0f` sur les 4 tiers. Cote QANGA, un tunnel passerait par `ATerrainHoleVolume`, qui ne modifie pas la hauteur : il met `Hole = 1.0f` ([WSHeightfieldVolumes.ush:551](../Plugins/WorldScape/Shaders/WorldScapeGPUTerrain/Private/WSHeightfieldVolumes.ush)) et supprime la geometrie |
| Champs et cultures | **la touche globalement, mais est inerte par defaut** | `FieldFlattenAt` (`FieldPatternCPU.h:200-231`) est une **modulation multiplicative**, pas un stamp. Elle n'attenue que deux bandes fines : les collines (x0,15 devient x0,0225) et la bande de detail, **ni les continents ni les montagnes**, ce qui rend impossible la creation d'une terrasse. Et `flattenStrength = 0` par defaut (`:193`) avec un commentaire explicite : 0 garde le champ de hauteur bit-identique, et le generateur routier lisant le meme champ, toute autre valeur **deplace le reseau**. Mesure a 0,75 : 165 453 segments devenus 154 466 |
| Ville | **la touche localement, mais elle drape, elle ne nivelle pas** | `cityRadius > 0` exclut la ville de `FlattenJunctionPlateaus` (`RoadNetwork.h:1480`) et de `SplitChainsAtPlatforms` (`:1355`). Chaque batiment porte son propre socle. Le sol partage est une **grille bakee 64x64** echantillonnee bilineairement (`RoadField.h:374-387`) |
| Routes | **la touche localement, par quatre mecanismes** | (1) carve de corridor, plat dans `halfWidth` puis fondu, combineur **soft-min** entre routes qui se recouvrent (`RoadField.h:587-588`, `:694-697`) ; (2) aplatissement de plateau, disques de jonction et esplanades, rampe de 40 m (`RoadNetwork.h:1453-1520`) ; (3) exclusion sur ouvrage ; (4) stamps d'auteur, **en ecrasement au-dessus de tout**, hors du soft-min (`RoadField.h:699-750`) |
| Relief | **c'est lui** | voir 3.3 |

### 3.2 A quoi sont accroches les lieux batis de la Terre

La Terre est `EarthScape` dans `Content/_QLevel/Universe/Planets/Earth/L_Earth.umap` : rayon 3 169 km, bruit `PlanetEarth` (porte GPU), heightmap planetaire `/QangaUnivers/PH_Earth` (**402 Mo**), **24 volumes heightmap, 8 trous, 0 volume de bruit, 10 masques foliage**, collision 16 / 200 non paddee, `NoiseIntensity` 2 200 000, seed 1962, `HeightAnchor` 15 000, MaxLod 12, resolution 128, triangle 100, `bOcean = False` (sonde headless du 2026-09-01, `Documentation/WorldScape_GPU_Completion_Plan_2026-09-01.md` section 1).

Le grep binaire de ce jour trouve des volumes heightmap dans `L_Persistent_Universe.umap`, `L_Persistent_Universe_Keep.umap` et `L_UniverseOnlyEarth.umap`.

Les trois lieux cites accrochent leur sol par **quatre couches superposees**, et chacune a une sensibilite differente a un changement de hauteur :

| Couche | Ce que c'est | Sensibilite a un changement de hauteur |
|---|---|---|
| **1. Le volume heightmap** | un `AHeightMapVolume` monte sur un `HeightMapVolumeData` (une texture). Avec `OverrideHeight = true` (defaut, [HeightMapVolume.h:119](../Plugins/WorldScape/Source/WorldScapeVolume/Public/HeightMapVolume.h)), la hauteur devient `lerp(Height, value, edgefalloff * intensity)` : **au centre d'un volume en override avec alpha 1, le bruit n'intervient plus du tout** | **immunise au centre, vulnerable au bord** : c'est la zone de fondu `EdgeFalloff` qui bouge |
| **2. Le terrain de substitution bake** | pour la Capitale : `Content/Baking_Capital/` = **256 tuiles** `Proxy_SM` (visuel) + **256 tuiles** `Collision_SM`, en grille 16x16, portees par un Blueprint `HM_Bake_Capital_Actor_C`, avec `HM_Bake_Capital_Data` et `HM_Bake_Capital_Tex2D`. Le meme patron existe sur l'Oasis | **totalement immunise** : c'est un asset. Mais si le terrain procedural remonte **sous** lui, il le transperce |
| **3. Les props snappes** | `AWorldScape_SnapSpawnerActor` cuit les transforms contre le terrain via `BuildFromWS()` qui appelle `Snap_ComputeCollection_ForActor_Async` ([WorldScape_SnapSpawnerActor.cpp:178](../Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScape_SnapSpawnerActor.cpp)), puis les rejoue en relatif : `LocalTransform.TransformPosition(Actor.Placement.Transform.GetLocation())` (`:213`) | **vulnerable** : la donnee est **cuite**. Une hauteur qui bouge laisse les props en l'air ou enterres jusqu'a un `BuildFromWS()` manuel, qui ecraserait les ajustements faits a la main |
| **4. Les sous-niveaux QLevel** | `Q_L_Earth.uasset` est un `QLevel_Asset` portant des `QLevel_Level_Data` (niveau, proxy mesh, `FQLevel_Shape`, [QLevel_Data_Struct.h:168](../Plugins/QLevel/Source/QLevel/Public/Struct/QLevel_Data_Struct.h)). Djibouti est sous `_QLevel/Universe/Planetary/Abandonned/Djibouty/` avec ses `L_Djibouti_Big_City_0x` et leurs `_LOptimised` | **vulnerable** : positions absolues. Le sol bouge, les batiments ne suivent pas |

**Verdict du pivot.** Les couches 1 et 2 sont des garde-fous **reels et suffisants**, mais ils ne protegent que ce qu'ils couvrent : l'interieur des 24 volumes et l'emprise du terrain bake. Les couches 3 et 4 sont de la **donnee cuite ou absolue**, et c'est par la que se produirait une regression, pas par le terrain. Donc :

> Toute modification du relief de la Terre doit etre **bornee a l'exterieur des 24 volumes heightmap**, et la bordure `EdgeFalloff` de chacun doit etre verifiee. Ce qui est **a l'interieur** d'un volume en `OverrideHeight` avec alpha 1 est deja immunise par construction.

**Ce que je n'ai pas pu mesurer** : les valeurs reelles de `OverrideHeight`, `EdgeFalloff`, `HeightAlpha` et `Blending` **volume par volume** sur `L_Earth`, ni leur emprise geographique. L'editeur etait ferme et le pont CLIScape aussi. La sonde exacte a lancer est en section 7. Sans elle, je ne peux pas affirmer que les 24 volumes couvrent bien Djibouti, le Finistere et la Capitale : je peux seulement affirmer que le mecanisme qui les protegerait existe et fonctionne comme decrit.

### 3.3 Le relief : deux sous-questions, pas une

Le relief de la Terre a deux etages, et ils ne portent pas le meme risque.

**Etage macro : une texture.** `bUsePlanetaryHeightMap` avec `PlanetaryHeightMap` echantillonne en coordonnees planetaires ([WorldScapeRoot_Noise.cpp:249-292](../Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScapeRoot_Noise.cpp)), soit `PH_Earth.uasset`, **402 Mo**. Falaises, montagnes et vallees a l'echelle continentale sont **deja pilotees par de la donnee**, pas par du bruit. Retoucher cette texture bouge le sol partout, y compris sous les lieux batis : **a ne pas faire**, sauf localement et hors emprise des volumes.

**Etage detail : le bruit.** `PlanetEarth` / `EarthNoiseFuntion`, porte GPU, environ **104 evaluations de simplex fp64 par texel** (`Documentation/WorldScape_GPU_vs_Clipmap_Perf_Diagnostic_2026-09-12.md` section 5.1). C'est la que se jouent les falaises et les vallees a l'echelle du joueur. Toute modification **deplace le sol partout** hors des volumes.

ScapeEngine ne resout pas ce probleme non plus : il n'a **ni erosion ni reseau hydrographique** (verifie : le relief est du bruit pur plus un DEM). Donc pour « terrain beaucoup plus realiste : falaises, montagnes, vallees », **il n'y a rien a porter depuis le moteur maison sur ce point precis**. C'est un chantier neuf, et c'est le plus risque des sept.

---

## 4. Les ecarts d'architecture

### 4.1 L'echelle : l'ecart n.1, et il est dimensionnant

| | ScapeEngine | Terre QANGA | Rapport |
|---|---|---|---|
| Rayon | 300 km | 3 169 km | 10,6 x |
| Surface | 1,13e6 km2 | 1,26e8 km2 | **112 x** |
| En Frances | 2,1 | **229** | |
| Part de la Terre reelle | 0,2 % | 24,7 % | |

Consequence, en partant de la densite mesuree (55 561 segments publies dans une fenetre de 30 km de rayon, soit 2 827 km2, soit **19,7 segments/km2**) :

| Quantite | Calcul | Resultat |
|---|---|---|
| Une fenetre de 30 km | 55 561 x 48 o | **2,54 Mio** |
| 500 joueurs disperses, une fenetre chacun | x 500 | **1,24 Gio** |
| Reseau ScapeEngine complet | 19,7 x 1,13e6 | 2,2e7 segments, **1,07 Go** |
| **Reseau Terre QANGA complet** | 19,7 x 1,26e8 | **2,5e9 segments, 119 Go** |

**Trois lectures de ce tableau, et elles decident du chantier :**

1. Un reseau routier planetaire a la densite francaise est **impossible** sur cette planete, dans n'importe quelle representation. La question n'est pas « comment le stocker », c'est « il ne doit pas exister ».
2. La generation a la demande par hash, qui est la reponse de ScapeEngine, **ne se transpose pas** : 500 joueurs disperses demanderaient 500 fenetres, soit 1,24 Gio et 500 x (191 + 153) ms de generation a chaque derive de 8 km. Le modele de ScapeEngine suppose **un** point de vue.
3. La Terre de QANGA est **presque vide** : 141 relais, quelques villes, des lieux poses a la main. Le reseau routier doit donc etre **local aux lieux**, dimensionne par le nombre de lieux et non par la surface. A titre d'ordre de grandeur : 141 relais avec 20 km de routes chacun font 2 820 km de route, soit environ 470 000 segments a 6 m de pas, soit **22 Mio**. Cela tient en memoire sur le serveur comme sur le client, et c'est cuisable dans un asset.

### 4.2 Le partage serveur / client : ce que le serveur doit savoir

Le serveur ne stocke pas de geometrie, et le code le prevoit deja finement. Matrice verifiee :

| Systeme | Serveur dedie | Preuve |
|---|---|---|
| Collision terrain | **oui, complete** | `bGenerateAllCollision` inclut `NM_DedicatedServer` ([WorldScapeRoot_Collision.cpp:117](../Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScapeRoot_Collision.cpp)) |
| Toutes les requetes de hauteur | **oui, CPU** | `GetEffectiveUseGPUNoise()` retourne `false` ([WorldScapeRoot_Main.cpp:300](../Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScapeRoot_Main.cpp)) |
| Mesh terrain GPU | **jamais** | idem, plus `GetNetMode() != NM_DedicatedServer` a `:1494`, `:1662`, `:3761`, `:4805` |
| Foliage | **oui par defaut, desactivable** | `bGenerateOnServer = true` par entree, skip a `WorldScapeRoot_Foliages.cpp:1032`, interrupteur global `DisableFoliageDedicatedServer` |
| Foliage collision pooling | **client seulement** | `WorldScapeRoot_FoliageCollisionPooling.cpp:185` |
| Grid Spawner | **oui, opt-in par asset, budget dedie** | `SpawnableOnDedicated` teste a `WorldScapeRoot_Grid.cpp:703` et `:3111` ; budgets 1024 / 2048 par seconde a `:443` et `:530` ; **`Grid_SkipStaticMeshOnDedicated`** a `:711`, `:761`, `:912` ; proxys sautes a `:1969`, `:2018` |
| Snap Spawner | **opt-in**, `false` par defaut | `WorldScape_SnapSpawnerActor.h:58`, teste a `.cpp:1010-1017` |
| QInstanced | **non** | `EnabledOnDedicated = false` |
| `WorldScapePCG` | **jamais** | `Type: "Editor"` |

**La regle qui en decoule**, et qui doit gouverner tout le chantier :

> Ce que le serveur doit savoir de la surface est **une fonction analytique evaluable depuis une position**. Ce qui est geometrique est **client**. Une route est donc deux choses distinctes : un **champ** (la hauteur du lit, la largeur, le type, le fait d'etre dessus), qui vit des deux cotes ; et un **ruban de mesh**, qui ne vit que cote client.

C'est exactement la separation que ScapeEngine a deja faite : `RoadField.h` (758 lignes, pur, sans moteur) d'un cote, `RoadMeshSystem.h` + `RoadMeshBuilder` (5 682 lignes) de l'autre. **Le champ est portable ; le mesh est a reecrire en UE, et c'est un gain, pas une perte** (voir 4.4).

Un point de vigilance a noter : `Grid_SkipStaticMeshOnDedicated` est a **`false`** dans le code ([WorldScapeRoot.h:771](../Plugins/WorldScape/Source/WorldScapeCore/Public/WorldScapeRoot.h)) et n'apparait dans aucun `.ini`. Sa valeur effective est celle **serialisee sur l'acteur `EarthScape`**, que je n'ai pas pu lire (editeur ferme). Si elle vaut `false` en production, le serveur cree bien les static meshes du grid. Sonde en section 7.

### 4.3 La precision : deja compatible, et c'est une bonne nouvelle

ScapeEngine utilise **exactement le pattern LWC d'Unreal** : monde en `DVec3` f64, rebase sur une origine locale, evaluation en f32 relatif. `DVec3 rel = spherePt - P.roadLocalOrigin` (`TerrainNoiseCPU.h:455`) cote CPU, et cote GPU `float3 roadPos = TileCenterOffset + PlanetRadius * CubeSphereDelta(...)` avec le commentaire « cancellation-free by construction » (`TerrainComputeShader.hlsl:283-287`).

Deux points de vigilance :

- **La seule zone en f32 pur est le cadastre de champs** (`FieldPatternCPU.h:176`), avec un **seed masque a 24 bits** (`:174`) parce qu'il voyage dans une lane flottante de constant buffer. A conserver tel quel si le pattern est porte, sinon le cadastre CPU et le cadastre GPU nomment deux mondes differents.
- **Le piege de l'origine mobile, deja paye et mesure** (`docs/ASYNC_GENERATION.md:27-43`) : la bande de detail etant relative a un `detailOrigin` qui suit la camera et resnappe tous les 50 km, le reseau routier **n'etait pas une fonction pure de la position**. Mesure : meme ancre 0,0 % de cellules sales, origine figee 23,9 %, **origine resnappee 100 %, dont 3 146 cellules changees a positions de segments identiques**. Correctif connu d'avance : figer l'origine du bruit sur le centre du job, pas sur la camera. **Ce bug se reproduira a l'identique en UE** si l'origine de detail reste mobile.

### 4.4 Ce qui est a reecrire, et pourquoi c'est un gain

| Sujet | Etat ScapeEngine | En UE | Gain ou perte |
|---|---|---|---|
| **Les miroirs CPU/GPU tenus a la main** | `RoadField.h` a maintenir en phase avec `RoadFieldCore.hlsli` + `RoadField.hlsli` + `RoadPaint.hlsli`, et `FieldPatternCPU.h` avec `FieldPattern.hlsli` : **environ 150 litteraux dupliques** (22 sels de hash, la table de 7 cultures, les echelles de tirage, les regles de pas), et le nivellement a fait passer de 3 a 4 miroirs | meme probleme, **et WorldScape le vit deja** : `WSHeightfieldVolumes.ush` est un portage ligne a ligne de la section volumes de `GetNoise`, avec un validateur numerique par tuile | **le plus gros risque du chantier.** La mitigation existe des deux cotes et elle est la meme : un validateur numerique CPU contre GPU, plus des hashes de reference geles |
| **Le ruban de route** | 5 682 lignes d'emission de geometrie dans une arene `MegaMeshBuffer` reecrite en place | `DynamicMesh` / spline + Nanite | **gain net** : deux defauts ouverts de ScapeEngine disparaissent gratuitement, l'absence de LOD par part (« un banc est maille a 1 499 m comme une bordure ») et l'invisibilite au ray tracing des villes qui vivent dans une arene |
| **La collision** | `HeightFieldShape` Jolt 64x64 a 2 m, patch de 126 m, regenere a 30 m de derive, 4 096 echantillons par patch | Chaos, et **WorldScape a deja son systeme** | contrat a preserver : le patch lit **exactement** la meme fonction de hauteur que le rendu, avec la meme collecte et le meme texel |
| **Le drapage** | 10 700 requetes par ville, **116 ms synchrones** | `GetGroundNoise_Batch` en `ParallelFor` existe deja | **gain** : le goulot de ScapeEngine a deja son remede cote QANGA |
| **Le crop shell** | **n'existe pas.** Le mur est chiffre : cellule de foliage 32, capacite 1 024, soit 4,9 plants/m2 contre environ 200 pour un vrai champ de cereales, **facteur 40**, et le pool est deja sature a 4 096/4 096 | a re-resoudre dans le modele UE | rien a porter |

### 4.5 QLevel : ce qu'un spawn massif doit respecter

Non negociable, sept points verifies dans [QLevel_SubSystem.h](../Plugins/QLevel/Source/QLevel/Public/QLevel_SubSystem.h) :

1. **L'unite d'enregistrement est `AQLevel_Actor_Instance`**, pas `AActor` (`:139`, `:144`). Le precedent de production est `QL_WS_Grid.uasset`, qui en derive.
2. **Franchir une porte de disponibilite avant tout spawn** : `QLevel_IsLocationReady` (`:179`) ou `QLevel_CanSpawnAIAtLocation` (`:196`). Spawner dans une QLevel non chargee cree un acteur orphelin.
3. **Maintenir une demande de chargement** : `QLevel_RequestLocationLoad` (`:183`) est un **bail a renouveler**, les demandes expirent quand on cesse de rafraichir. Pour une generation qui dure plusieurs secondes, rafraichir a chaque tick.
4. **Se ranger en `TG_PostUpdateWork`** : c'est le defaut de QLevel (`:255-271`), de QInstanced et du subsystem WorldScape. Copier le patron `RegisterSingleTick(TickFunction, FuncPtr, TickGroup, TickInterval, Enabled)` visible identiquement dans `WorldscapeSubsystem.h:161-181` et `QInstanced_SubSystem.h:104-125`. Ne pas faire un `Tick` d'acteur.
5. **Passer par le budget de QInstanced** : ecrire dans `UQInstanced_ISMComponent` ou `UQInstanced_HISMComponent` suffit a s'y inscrire.
6. **Respecter les budgets deja poses** : Grid 512 / 1024 spawns par seconde, 1024 / 2048 destructions ; Snap 10 acteurs et 2 static meshes par tick.
7. **Se synchroniser sur les evenements de cellule WS**, pas sur les LOD de terrain (2.5).

---

## 5. Decoupage en lots

Ordonne par gain visible sur risque. Chaque lot est livrable seul et ne casse rien s'il s'arrete la.

### Lot 0 : la sonde de verite sur les 24 volumes (une demi-journee)

**Perimetre.** Un script Python editeur en lecture seule qui dumpe, pour chaque `AHeightMapVolume` de `L_Earth` : nom, transform, `Scale3D.X` (donc l'emprise, `volumeSize = ScaleX * 50`), `OverrideHeight`, `EdgeFalloff`, `HeightAlpha`, `UseBlending` et les parametres de blend, `MapHeight`, `Altitude`, le `HeightMapVolumeData` monte et ses tailles de masque. Idem pour les 8 `ATerrainHoleVolume`. Plus la valeur effective de `Grid_SkipStaticMeshOnDedicated` sur `EarthScape`.

**Risque de regression.** Nul, lecture seule.

**Validation.** Un tableau des 24 volumes, et la reponse binaire : Djibouti, le Finistere et la Capitale sont-ils, oui ou non, a l'interieur d'un volume en `OverrideHeight` avec alpha 1.

**Pourquoi en premier.** Tout le reste de la contrainte dure en depend. Aujourd'hui je peux affirmer que le mecanisme protege ce qu'il couvre ; je ne peux pas affirmer qu'il couvre ces trois lieux.

### Lot 1 : `PlaceAlongSpline`, le mobilier de bord de route (2 a 3 jours)

**Perimetre.** La fonction qui manque partout : placer des instances a intervalle regulier le long d'une courbe, avec offset lateral, orientation sur la tangente et projection au sol. Ecrite dans `UWS_FunctionLibrary` a cote de `WS_CubicBezierY`, alimentee par `WS_ProjectGroundLocationWithNormal_Multi`, ecrivant dans `UQInstanced_HISMComponent`. Un `AWorldScape_SnapSpawnerActor` la consomme.

**Fichiers.** `WS_FunctionLibrary.h/.cpp` (ajout pur), un nouvel acteur dans `WorldScapeCore`.

**Risque de regression.** Tres faible : ajout de fonctions, aucun contrat existant touche. **Ne touche pas la hauteur.**

**Validation.** Une spline posee a la main dans `L_Dev_Claude`, 200 lampadaires et glissieres a 40 m de pas, assiette correcte sur une pente, et le critere de stabilite de ScapeEngine : **l'ensemble pose ne doit pas changer quand la fenetre bouge**. La lecon est deja payee la-bas (`roadside_stability_probe.cpp`) : l'identite d'une run est son extremite lexicographiquement plus petite, **quantifiee au quart de metre**, et tout tirage est un hash de cette identite, jamais un compteur ni un index dans une collection dont l'ordre depend de la fenetre.

**Pourquoi en premier apres la sonde.** C'est le plus petit gain visible du lot, il est immediat en jeu, il ne touche rien, et il produit la brique dont les lots 3 et 5 ont besoin.

### Lot 2 : multiplier les fermes et les champs qui existent deja (3 a 5 jours)

**Perimetre.** Pas construire : **dupliquer et regler**. Le champ de tournesols et le kit de ferme sont deja poses sur la Terre (section 2.3bis). Le lot consiste a en faire des variantes et a les semer : prendre `Farm_Field_Subflowers_01.uasset` comme gabarit et decliner 4 ou 5 cultures parmi les 306 meshes d'`UltimateFarming` (ble, mais, tournesol, vigne, betterave), combiner les 8 `AssetCollection` de ferme en 3 ou 4 compositions, et ecrire les `WorldScape_GridDataAsset` qui les implantent (filtres pente et humidite, `InvalidSize` pour l'espacement, `SpawnableOnDedicated`). Un `HeightMapVolumeData` pour l'assiette de la cour si besoin (precedent `HM_Terrace_Fields_15_Ex`). **Zero ligne de C++.**

**Fichiers.** Uniquement du contenu, sous `Content/_QLevel/Universe/Planetary/Farm/` et `Content/Systems/WorldScape/GridData/Earth/Farm/`.

**Risque de regression.** Faible et connu : c'est le chemin des bunkers, des oasis et du champ existant. Le seul risque est le budget de spawn, deja borne par `Grid_MaxSpawnPerSecond`. **Ne touche la hauteur que dans l'emprise du volume pose**, donc localement, comme un bunker aujourd'hui.

**Validation.** `audit_blueprint` et `get_asset_dependencies` sur les assets touches, PIE, comptage des instances, et une verification serveur dedie (`Grid_SimulateDedicated` existe pour cela : [WorldScapeRoot.h:683](../Plugins/WorldScape/Source/WorldScapeCore/Public/WorldScapeRoot.h)).

**Ce qui manque encore apres ce lot, et qu'il faut assumer.** Trois choses : le **rang de culture** (le sampler du foliage tire dans une boite), la **bordure nette** (rejet stochastique, donc bord flou), et surtout l'**automatisation du bake**. Le champ actuel est 17,5 Mo de transforms figes fabriques a la main : multiplier les variantes multiplie ce travail manuel. C'est precisement la que PCG a sa valeur (section 6.2), et c'est ce qui justifie de le faire au lot 2 plutot qu'apres.

### Lot 3 : le champ de route, sans geometrie (1 a 2 semaines)

**Perimetre.** Le portage de `RoadField.h`, et lui seul. Une structure de segment 48 octets, une collecte spatiale bornee, une fonction `EvaluateRoadField(position)` qui retourne `(weight, height, lateral, halfWidth, type)`, un etage insere dans `AWorldScapeRoot::GetNoise` **apres les volumes heightmap et avant les trous**, et son miroir dans `WSHeightfieldGenerate.usf`. Le trace vient d'une donnee **authoree ou cuite**, pas d'un generateur (voir 4.1, point 3).

**Fichiers.** `WorldScapeRoot_Noise.cpp`, `WSHeightfieldGenerate.usf`, un nouveau fichier de champ, un `.ush` partage.

**Risque de regression.** **Le plus eleve du lot, et il est de deux natures.** (a) On touche la fonction de hauteur : hors emprise de route, le champ doit retourner un poids strictement nul, et cela se prouve, pas se suppose. (b) On ajoute un cinquieme miroir CPU/GPU. La mitigation est celle que le chantier GPU s'est deja imposee : un validateur numerique par tuile, et les cinq regles de port non negociables deja ecrites (`Documentation/WorldScape_GPU_Completion_Plan_2026-09-01.md:77`), auxquelles s'ajoute la regle 3 de la section 1.5 : **les egalites se departagent sur la hauteur, jamais sur l'ordre de collecte**.

**Validation.** Le validateur numerique existant, etendu aux points de route ; un personnage et un vehicule sur une route, sans interpenetration ni flottement ; et la non-regression des 24 volumes, mesuree avant et apres.

**Ce que ce lot achete.** Le sol de la route, identique serveur et client, sans un seul mesh. Le serveur sait « je suis sur une route », ce qui ouvre le gameplay (vitesse, bruit de roulement, IA qui suit la route) sans rien stocker.

### Lot 4 : le ruban visuel, cote client seulement (1 semaine, revu a la baisse)

**Perimetre.** La geometrie : ruban, bordures, marquages, a partir du meme champ. **Le probleme dur est deja resolu** : poser un revetement de rue sur le cube-sphere en double precision tourne en production, ce sont les 11 `SnapDataAsset` d'`OLD_Town` (section 2.3bis). Le lot consiste a **generer** ce que ces assets contiennent, au lieu de le baker a la main, et a le faire suivre une courbe. Reutilise le pack `ModularRoutes` pour les intersections. Jamais sur serveur dedie.

**Risque de regression.** Faible, parce que purement client et purement additif. **Ne touche pas la hauteur** : il la lit.

**Validation.** Visuelle, plus le budget de QInstanced, plus une verification que le serveur dedie ne cree rien. Comparaison directe avec `OldTown_Street_Snap_01` : le rendu genere doit tenir la comparaison avec le bake manuel.

**Ce qui manque et qu'il faut assumer.** Aucun rond-point n'existe dans les assets du projet (recherche `roundabout` et `rond` sur tout `Content/` et tous les `Plugins/*/Content/` : rien). Un rond-point est a modeliser ou a acheter.

### Lot 5 : ponts (1 semaine, apres les lots 3 et 4)

**Perimetre.** La decision par evenement sur le profil en long, portee telle quelle depuis `RoadNetworkGenerator.h` : c'est un algorithme pur d'une centaine de lignes utiles. Le tablier porte un flag `NoCarve`, donc **ne touche pas la hauteur**. Les piles ont besoin du relief **sans routes**, donc d'un appel a la fonction de hauteur avec le champ desactive : a prevoir dans l'API du lot 3.

**Risque de regression.** Faible. Mais les **deux defauts que ScapeEngine a payes** sont a eviter d'entree : le plancher de degagement (`max(3 m, Hmax/4)`) qui decide **ou** l'on franchit, distinct du seuil qui decide **si** ; et le fait qu'un span ne rejoint le tablier que s'il est **partout** au-dessus, donc sur son **minimum** et non son maximum. Sans cela, on reproduit « 18,5 % de tablier sous 2 m de degagement » et « le tablier commence 11 m sous le sol ».

### Lot 6 : ville (plusieurs semaines, ou jamais)

**Perimetre.** Deux options, et je recommande la premiere.

**Option A, assemblage (recommandee).** Etendre ce qui fonctionne deja : une ville est un assemblage de quartiers pre-faits, poses par le Grid Spawner, avec proxys et HLOD. Ce n'est pas une hypothese, c'est **l'etat actuel du jeu** : 13 quartiers de Djibouti avec leur niveau de detail, IronCity, Metropolis, OLD_Town et ses rues en Snap, le patron `WS_ReplyTower_Spawner` et une quarantaine de niveaux (section 2.3bis). Coût faible, resultat immediat. Le travail utile est donc d'**ecrire le plan** qui choisit et enchaine les quartiers, pas de refaire la pose.

**Option B, generateur.** Porter `CityPlan.h` (1 466 lignes, plan pur, prouve par une sonde sur 55 035 ilots) plus `CityGround.h` (grille bakee 64x64). L'argument de l'auteur pour la grille bakee est le meilleur argument du dossier et il vaut pour UE : une grille bakee est **bit-identique des deux cotes par construction, parce que ce sont les memes octets**, la ou une formule analytique aurait exige de miroiter deux fonctions de bruit de plus a la main, « la seule classe de bug que ce depot paie le plus souvent ». Mesure a l'appui : quatre reparations moins cheres ont ete essayees et ont **mesure en echec** avant d'arriver a cette grille.

### Lot 7 : le relief, erosion et hydrographie (le chantier principal, non chiffre)

**Requalifie le 2026-09-18** : avec le cadrage revise (section 3, encadre), ce lot n'est plus le plus risque a eviter, c'est **le sujet**. La cible annoncee est le niveau de Genesis.

**Perimetre.** Falaises, vallees en V, cones alluviaux, strates de roche, reseau hydrographique coherent.

**Ce qu'il faut savoir avant de s'engager, et c'est le point dur** : **il n'y a rien a porter depuis ScapeEngine.** Verifie : son relief est du bruit pur plus un DEM, sans erosion ni flow accumulation. Ce lot est donc un chantier **neuf**, et c'est le seul des sept dans ce cas.

**Ce qui le contraint** : le bruit tourne deja a environ 104 evaluations de simplex fp64 par texel sur le chemin GPU (section 1.3 du diagnostic de perf), et toute couche supplementaire s'ajoute a ce budget, des deux cotes du miroir CPU et GPU. Une erosion vraie est iterative, donc incompatible avec une evaluation par texte a la demande : elle se pre-calcule dans une texture, ce qui ramene au modele deja en place (`PH_Earth`, 402 Mo) mais a une resolution plus fine et par region.

**A trancher en premier, avant toute ligne** : erosion pre-calculee en texture par region, ou fonction analytique qui imite l'erosion (domain warping, terracing, ridged multifractal avec masque de pente). La premiere est fidele et coute du disque et un pipeline de cuisson ; la seconde est gratuite en donnee et ne donne jamais de vraies vallees fluviales.

**Ce qui ne vaut pas le cout, franchement.** Les tunnels : ScapeEngine ne les a pas, `ATerrainHoleVolume` est experimental et demande une modification manuelle du master material du terrain, et le canal `Hole` repose sur la couleur de vertex donc subit le lissage du mesh a distance. Un tunnel pose a la main comme sous-niveau QLevel coûte mille fois moins et rend mieux. Et `ANoiseVolume` : l'utiliser fait retomber **toute la planete** sur le clipmap CPU.

---

## 6. Recommandation d'architecture

### 6.1 La recommandation

**Un nouveau module dans WorldScape, nomme `WorldScapeSurface`, qui ne contient qu'un champ et une collecte.** Pas un plugin separe, pas PCG au runtime.

Les raisons, dans l'ordre :

1. **Le point d'insertion est dans WorldScape, et il est unique.** Le champ doit etre un etage de `AWorldScapeRoot::GetNoise` et de `WSHeightfieldGenerate.usf`. Un plugin separe devrait soit dependre de `WorldScapeCore` (et alors autant etre un module du meme plugin), soit faire remonter un callback dans la boucle de bruit, ce qui coûterait un appel virtuel par texel.
2. **Le couplage WorldScape / QLevel est deja assume au niveau du build** (`WorldScapeCore.Build.cs:49`, dependance **publique**). Le nouveau module herite du meme droit sans rien negocier.
3. **Un module separe, et non du code ajoute dans `WorldScapeCore`**, parce que `WorldScapeCore` fait deja 30 686 lignes et que le champ doit rester **pur et testable seul**. C'est la propriete qui fait la valeur de `RoadField.h` cote ScapeEngine, et elle s'entretient : « compile seul contre `src`, aucun include moteur ».
4. **La geometrie ne va pas dans ce module.** Le ruban, les props et les batiments sont du contenu et des acteurs, donc du `WorldScapeCore` existant (Grid, Snap) et du Blueprint. Le module ne connait que des scalaires.

### 6.2 PCG : verdict

**Oui comme atelier en editeur qui cuit des assets. Non comme runtime.** Ton intuition d'aller dans WorldScape est la bonne ; PCG a sa place, mais pas celle qu'on imagine.

Pour :
- le plugin `PCG` d'Epic est **deja actif** (`Qanga.uproject:338-341`, plus dependance de `WorldScape.uplugin:169-172`) et **deja en production dans des niveaux** : `L_Tools_PCG.umap` porte des `PCGComponent` vers `PGC_GRASS_Nord|Sud|NoTree`, `L_Cave_Test.umap` vers un generateur de grotte a bruit spatial ;
- le pont WorldScape vers PCG **a deja fonctionne deux fois**, en Blueprint (`Content/AdvancedCamera/_Test/NewPCGBlueprintElement.uasset`) et en C++ (`WorldScapeTestPCGNode.cpp:71-80`, reprojection radiale correcte). Le module compile : `UnrealEditor-WorldScapePCG.dll` existe ;
- l'API CPU de WorldScape est **idealement formee** pour PCG : des fonctions batch qui correspondent exactement au modele de donnees point ;
- CLIScape sait creer, inspecter et executer des graphes par outil, donc l'iteration est automatisable depuis une session ;
- le moteur dispose sans coût de `PCGBiomeCore` et `PCGGeometryScriptInterop`, actuellement desactives.

Contre, et c'est decisif :
- **`Type: "Editor"`** (`WorldScape.uplugin:108`). Aucun noeud WorldScape n'existera dans un build de jeu ni sur serveur dedie ;
- les graphes deja en place echantillonnent par `PCGWorldRayHit`, qui exige de la **collision physique**. Sur WorldScape la collision n'existe que localement, autour des `CollisionDependantActor` et des invokers. Un graphe PCG ne peut donc pas s'appuyer sur `PCGWorldRayHit` a l'echelle planetaire ; il **doit** passer par un noeud WorldScape natif ;
- le noeud existant est un point de depart, pas une base : `HeightScale` declaree et jamais utilisee, `HeightOffset` clampe a 10 m (absurde a l'echelle planetaire), sampling **point par point en float** au lieu de `GetGroundNoise_Batch`, **aucune normale de terrain** (il utilise la radiale pure, donc les instances seront verticales sur une pente), aucune contrainte de pente, de temperature, d'humidite ni de trou alors que `GetGroundNoise` les retourne, API `UPCGPointData` legacy et `Initialize` sur la surcharge depreciee depuis 5.6 ;
- le fallback de recherche de planete est **casse en editeur** : `GEngine->GetCurrentPlayWorld()` (`WorldScapePCGNodeBase.cpp:86`) retourne `nullptr` hors PIE, et le module est `Editor`. Un noeud sans `WorldScapeRoot` assigne echoue silencieusement ;
- aucun reglage PCG dans les configs, donc aucune runtime generation en place a reprendre.

**Donc, et la cible est plus etroite que je ne le pensais d'abord :** PCG n'a besoin ni de generer de la geometrie ni de poser des instances au runtime, parce que **la couche de pose existe deja et elle est eprouvee** : 70 `GridDataAsset`, 13 `SnapDataAsset` et 8 `AssetCollection` en production, avec budget par tick, opt-in serveur dedie et proxys bakes. Le bon role pour PCG est donc **d'ecrire ces DataAssets**. Un graphe prend une emprise, sort un plan de parcelles ou un plan de rues, et emet un `WorldScape_SnapDataAsset` que le runtime consomme deja. Cela **contourne entierement** le probleme du `Type: Editor`, puisque le runtime ne verrait jamais PCG. Et cela attaque le vrai point de douleur : les 17,5 Mo du champ de tournesols sont un bake manuel, et multiplier les variantes multiplie ce travail a la main.

**PCG jamais** pour le reseau routier (il n'a ni graphe, ni solveur de trace, ni resolution de carrefour), la geometrie de pont, ou quoi que ce soit qui doive exister a l'identique sur serveur dedie sans bake.

**A verifier avant de s'engager sur cette voie** : `Plugins/CLIScape/Source/CLIscape/Private/Tools/PCG/Tool_CreatePCGPreset.cpp` et `Tool_SpawnPCGInLevel.cpp`, pour savoir si l'ecriture d'assets depuis un graphe est deja outillee. **Non verifie.**

Prealables si on investit sur PCG, dans cet ordre : reparer `GetWorldScapeRoot`, reecrire le noeud sur `GetGroundNoise_Batch` plus `WS_ProjectGroundLocationWithNormal_Multi`, migrer vers `FPCGInitializeElementParams` et `UPCGBasePointData`, et **trancher explicitement** entre passer le module en `Runtime` ou assumer le bake. Je recommande le bake : il est compatible avec tout ce qui existe deja.

### 6.3 Le contrat a poser avant la premiere ligne

Sur le modele du §4 du plan de completion GPU, qui a bien servi :

1. **Couleur de vertex RGBA = (HeightNormalize, Temperature, Humidity, Hole), UV0 = grille normalisee. Jamais de changement de packing.** Les quatre canaux sont **pris**. Une route ne peut donc pas etre « peinte » par un cinquieme canal de vertex : il faudra une texture planetaire echantillonnee dans le materiau, ou de la geometrie separee. A trancher au lot 4.
2. **Identite de classe** : le primitif reste `UWorldScapeMeshComponent`, l'acteur reste `AWorldScapeRoot`, les noms de modules ne bougent pas.
3. **L'API bruit reste CPU et independante du mesh** : c'est la seule connaissance du terrain sur le serveur et dans les `ParallelFor` de QAI.
4. **Collision, foliage, grille QLevel, Snap Spawner : non touches.**
5. **Le chemin CPU seul reste fonctionnel** (`bUseGPUNoise = false`).
6. **Hors emprise, le champ retourne un poids strictement nul**, et cela se prouve par une sonde, pas par lecture.
7. **Les egalites se departagent sur une valeur, jamais sur l'ordre de collecte.**

### 6.4 Aligner PCG sur WorldScape : ce que dit le source du moteur

Lecture de `C:\UE5_Share\Engine\Plugins\PCG\Source\PCG\` ce jour. Question de Benja : le PCG cote WorldScape n'est pas avance, que faut-il lire dans le moteur pour **aligner les outils** et viser plus grand, en particulier sur la **courbure** et sur la **hauteur du bruit une fois retravaillee**.

**Le mauvais point d'extension est celui que WorldScapePCG a choisi.** Un `UPCGSettings` plus un `IPCGElement` est un **noeud** : il ne sert que dans les graphes ou quelqu'un l'a place a la main, et il ne rend WorldScape visible a aucun autre noeud. C'est pour cela que les graphes du projet echantillonnent par `PCGWorldRayHit` ou par les pins `Landscape`, c'est a dire sans passer par WorldScape du tout.

**Le bon point d'extension est une Data.** `UPCGSpatialData` (`Public/Data/PCGSpatialData.h`, 349 lignes) est l'interface que tout PCG consomme, et `UPCGSurfaceData` (`Public/Data/PCGSurfaceData.h`) en est la specialisation surface, avec `GetDimension() = 2`, `HasNonTrivialTransform() = true`, et exactement deux membres : **`FTransform Transform` et `FBox LocalBounds`**. Ecrire un `UPCGWorldScapeSurfaceData : UPCGSurfaceData` rend WorldScape utilisable par **tout le reste de PCG sans rien ecrire de plus** : le `SurfaceSampler`, les filtres de densite, la projection de splines, et gratuitement `IntersectWith`, `ProjectOn`, `UnionWith`, `Subtract` qui sont implementes dans la classe de base.

Le modele a copier est `UPCGLandscapeData` (`Public/Data/PCGLandscapeData.h:68-101`). Liste exacte des overrides qu'il pose, donc la liste du travail :

| Override | Pourquoi il compte ici |
|---|---|
| `GetBounds` / `GetStrictBounds` | des `FBox`, donc alignes sur les axes du monde : c'est le premier point ou la courbure mord (voir plus bas) |
| `SamplePoint(Transform, Bounds, OutPoint, Metadata)` | requete unitaire |
| **`SamplePoints(TArrayView<TPair<FTransform,FBox>>, TArrayView<FPCGPoint>, Metadata)`** | **la version batch, et c'est le branchement direct sur `GetGroundNoise_Batch`** qui tourne deja en `ParallelFor`. Le noeud actuel appelle `GetGroundHeight` point par point en `float` : c'est ce qu'il faut cesser de faire |
| `ProjectPoint(InTransform, InBounds, InParams, OutPoint, Metadata)` | la projection au sol, pilotee par `FPCGProjectionParams` (`Public/Elements/PCGProjectionParams.h:29`) qui expose `bProjectPositions`, les rotations, les echelles et les couleurs separement |
| **`PrepareForSpatialQuery(Context, Bounds)` retournant des `FPCGTaskId`** | **le moteur prevoit une preparation asynchrone avant l'echantillonnage.** C'est exactement ou faire charger ce qu'il faut (volumes, heightmap planetaire) sans bloquer, et le Landscape s'en sert |
| `SupportsBoundedPointData() = true` et `CreatePointData(Context, Bounds)` | generer les points par emprise et non pour toute la Data |
| `InitializeTargetMetadata` | **le levier pour « la hauteur retravaillee »** : voir ci-dessous |
| `CopyInternal`, `AddToCrc` | plomberie obligatoire ; le Crc sert au cache de PCG, donc il doit inclure la graine du bruit et l'identite des volumes, sinon PCG resservira un resultat perime |

**La courbure : le probleme est reel, il est dans l'interface, et il est chiffrable.**

`UPCGSpatialData::GetNormal()` est **une constante par Data** : `virtual FVector GetNormal() const { return FVector::UnitZ(); }` (`PCGSpatialData.h:156`). `UPCGLandscapeData` ne l'override meme pas, parce qu'un Landscape est plan. `UPCGWorldData` l'override en `Transform.GetRotation().GetUpVector()`, donc encore une constante. Il n'y a **aucun endroit dans PCG ou la normale d'une surface est une fonction de la position**.

Qui la consomme, et donc qui casserait (grep exhaustif sur `Private/` et `Public/`) : `PCGProjectionData.cpp:122-131` la propage, `:162-163` en derive les bounds, et **`PCGSplineData.cpp:655`, `:708`, `:743`** l'utilisent comme direction unique pour projeter une spline sur une surface. Ce troisieme point touche directement le sujet routes.

Fleche de courbure sur la Terre de QANGA, rayon 3 169 km, formule `L^2 / 8R` :

| Emprise | Fleche | Lecture |
|---|---|---|
| 500 m | 1 cm | sans objet |
| 1 km | 4 cm | sans objet |
| 2 km | 16 cm | acceptable |
| **5 km** | **99 cm** | un prop pose a 1 m du sol |
| 10 km | 3,94 m | un batiment en l'air ou enterre |
| 20 km | 15,8 m | |
| 50 km | 98,6 m | redhibitoire |
| 100 km | 394 m | |

**Correction apres lecture de `PCGSurfaceSampler.cpp` et `PCGLandscapeData.cpp` : cette fleche ne casse PAS l'echantillonnage, et c'est une bonne nouvelle.** Le tableau ci-dessus reste vrai geometriquement, mais il ne s'applique pas la ou je le croyais. Trois mesures dans le source :

1. **`ProjectPoint` n'est pas un lancer de rayon.** `UPCGLandscapeData::ProjectPoint` (`Private/Data/PCGLandscapeData.cpp:326-432`) **ignore purement et simplement le Z du point d'entree** : il passe en espace local, calcule la cellule depuis X et Y, echantillonne le champ de hauteur, et ecrit la position. Le commentaire de l'auteur le confirme (`:341`, « TODO: compute full transform when we want to support bounds »). Donc un candidat pose 98 m au-dessus du sol trouve exactement le meme sol qu'un candidat pose dessus. L'equivalent WorldScape est **naturel et exact a toute distance** : la direction depuis le centre de la planete *est* la coordonnee de surface, donc `GetHeightZeroPosition` puis `GetGroundNoise` rend la bonne hauteur quelle que soit la fleche. Le noeud de test actuel fait deja cette reprojection radiale correctement (`WorldScapeTestPCGNode.cpp:76`).
2. **Le sampler redresse deja la grille sur la surface.** `bNeedsLocalTransformation = !InSurfaceTransform.Rotator().IsNearlyZero()` (`Private/Elements/PCGSurfaceSampler.cpp:179`), et si c'est vrai il construit `PreProjectionTransform` = translation inverse, puis `FQuat::FindBetweenNormals(FVector::UpVector, InSurfaceTransform.GetRotation().GetUpVector())`, puis translation (`:181-187`). **Point d'action concret : la Data doit poser une rotation dans son `Transform`.** Si elle laisse la rotation a zero, la grille de candidats est posee a plat en X et Y du monde, ce qui est faux partout sauf au pole. Avec `CellCenterNormal` comme up, la grille devient tangente gratuitement.
3. **La densite ne souffre pas.** L'angle au bord d'une emprise de 50 km vaut 0,45 degre, donc la dilatation de la projection gnomonique vaut 1,00003. Negligeable jusqu'a des emprises de plusieurs dizaines de kilometres.

**Ce qui reste reellement affecte par la courbure**, apres cette correction :

- **`GetNormal()` pour les splines.** `PCGSplineData.cpp:655`, `:708`, `:743` projettent une spline avec **une** normale de surface. Sur un tronce de 10 km, faux de 3,94 m. C'est le point qui touche les routes, et il n'a pas de contournement propre dans l'interface : il faut soit borner la Data, soit ne pas passer par `PCGSplineData` pour les routes.
- **Les bounds sont des `FBox` en espace monde.** `PreProjectionDisplacement = InEffectiveGridBounds.Max.Z * (1 - epsilon)` (`PCGSurfaceSampler.cpp:196`) : la hauteur de depart des candidats vient du sommet des bounds. Si la Data declare une AABB planetaire, ce Z est absurde. `GetStrictBounds()` et `IsBounded()` existent pour declarer honnetement, et il faut s'en servir.
- **La fidelite du rectangle d'emprise** : une grille plane projetee sur la sphere ne dessine pas exactement le rectangle voulu. Negligeable jusqu'a 50 km, a regarder au-dela.

**Donc la regle de conception s'assouplit :** une `UPCGWorldScapeSurfaceData` doit poser un **`Transform` avec la rotation du plan tangent** et des **bounds honnetes**, et sa taille est bornee par le besoin de `GetNormal()` (les splines) et non par la position des points. Pour tout ce qui est semis de props, champs et implantation, une emprise de plusieurs kilometres passe sans probleme. Pour une spline de route, on borne a quelques kilometres ou on traite la route autrement.

**La grille WorldScape n'est pas le bon borneur, contrairement a ce que sa forme suggere.** `FWS_Grid` (`WorldScape_GridStruct.h:382-400`) porte pourtant exactement ce qu'il faudrait, `CellCenterLocation` et `CellCenterNormal`, qui se mappent un a un sur le `Transform` de `UPCGSurfaceData`. Mais son champ `Depth` **n'est pas une profondeur hierarchique** : c'est le nombre de divisions par face, il recoit `Grid_Size` (`WorldScapeRoot_Grid.cpp:213`) et sert a centrer les indices (`HalfDepth = Depth / 2`). Avec `Grid_Size = 64` ([WorldScapeRoot.h:732](../Plugins/WorldScape/Source/WorldScapeCore/Public/WorldScapeRoot.h)) et une face de 4 978 km, **une cellule fait 77,8 km, soit 239 m de fleche**. Il faudrait `Grid_Size = 2048` pour tomber a 2,4 km de cellule, et cette valeur est un **contrat gele** : les 70 `WorldScape_GridDataAsset` en production sont indexes dessus. A noter au passage, et c'est une bonne nouvelle : la preallocation `Grid_StaticData.Reserve(Grid_Size * Grid_Size * 6)` est **entierement commentee** (`WorldScapeRoot_Grid.cpp:2894-2908`), la grille est donc un adressage **analytique** calcule a la demande, sans cout memoire.

Donc : la grille WS reste le bon ancrage pour **l'implantation** (ou poser une ferme, un village) et pour les evenements de chargement, et une Data PCG locale reste le bon objet pour **l'echantillonnage** dans une emprise. Deux roles, deux echelles, ne pas les confondre.

**« La hauteur du bruit une fois retravaillee » : le levier est `InitializeTargetMetadata`.**

`GetGroundNoise` ne retourne pas une hauteur, il retourne un `FNoiseData` : hauteur, hauteur normalisee, **temperature, humidite, masque de trou**, et c'est deja la composition complete (heightmap planetaire, puis bruit, puis volumes de bruit, puis volumes heightmap, puis trous). Le noeud actuel appelle `GetGroundHeight` et **jette tout le reste**, puis oriente les points sur la radiale pure au lieu de la normale du terrain, donc un prop sur une pente reste vertical.

Aligner les outils, concretement, c'est exposer ce `FNoiseData` comme **attributs de metadonnee PCG** sur chaque point : `Height`, `HeightNormalize`, `Temperature`, `Humidity`, `Hole`, plus la normale reelle (`GetPawnSnappedNormal` ou `WS_GetTerrainNormalTriTest`) et la pente derivee. A partir de la, **tous les filtres PCG existants deviennent des filtres de biome** sans une ligne de code supplementaire : un `AttributeFilter` sur `Humidity` selectionne une zone cultivable, un filtre sur `Hole` evite les trous, un filtre sur la pente selectionne le constructible. C'est ce que `FWS_Grid_SpawnParams` fait deja en dur cote Grid ([WorldScape_GridStruct.h:795-825](../Plugins/WorldScape/Source/WorldScapeCore/Public/WorldScape_GridStruct.h)) ; le rendre declaratif dans un graphe, c'est exactement le gain que PCG apporte.

**Deux pieges a ne pas rater, tous deux dans le source du moteur.**

1. **Les bounds sont des `FBox`**, donc alignes sur les axes du monde. A l'echelle planetaire, la boite englobante d'une emprise posee sur une sphere loin de l'origine est enorme et presque vide. `GetStrictBounds()` existe justement pour declarer la partie garantie pleine, et `IsBounded()` pour dire qu'une Data ne l'est pas. A renseigner honnetement, sinon les operations booleennes de PCG travailleront sur des volumes absurdes.
2. **Le `Crc` pilote le cache.** `AddToCrc(Ar, bFullDataCrc)` doit integrer la graine du bruit, le hash d'identite des volumes (celui que le chemin GPU calcule deja, `WS_HashTerrainVolumesIdentity`) et le transform de la planete. Sans cela, PCG resservira un resultat mis en cache apres un changement de volume, et le bug sera silencieux.

### 6.5 Le patron d'implementation, tire des deux lectures

**Comment le sampler genere ses candidats** (`Private/Elements/PCGSurfaceSampler.cpp:240-330`), et pourquoi c'est un bon design pour nous :

- la grille est indexee par des **entiers globaux** : `CurrentX = Indices.X * CellSize.X` (`:276`), avec `CellMinX = CeilToInt(Bounds.Min.X / CellSize.X)` (`:117`). Les index ne dependent donc **pas** de la fenetre : le meme point est genere quelle que soit l'emprise demandee. C'est exactement la propriete de stabilite que ScapeEngine a payee cher (`roadside_stability_probe.cpp`), et PCG l'a deja.
- la graine est `PCGHelpers::ComputeSeed(Seed, Indices.X, Indices.Y)` (`:281`), donc un **hash des index de cellule**, jamais un compteur. Meme lecon, deja appliquee.
- le rejet est stochastique par cellule : `if (Chance >= Ratio) continue` (`:286`), la meme logique que le foliage WorldScape.
- a l'echelle planetaire les index restent sains : a 3,17e8 cm de l'origine avec des cellules de 1 m, l'index vaut 3,17e6, donc tres loin de la limite `int32`, et `CurrentX` est calcule en double (precision 7e-8 cm). Il y a un garde-fou memoire sur `CellCount` (`:170-175`, CVar `pcg.SamplerMemoryThreshold`).

**Comment le Landscape implemente le batch** (`PCGLandscapeData.cpp:204-326`) : il ne fait pas un `ParallelFor` naif. Il construit d'abord une `TMap<ULandscapeHeightfieldCollisionComponent*, TArray<int>>` qui **regroupe les echantillons par unite de donnees**, par paquets de `DefaultSamplePointsChunkSize`, puis traite groupe par groupe. Et il met `OutPoints[i].Density = 0` d'entree (`:216`), parce que le contrat de `SamplePoints` declare dans l'interface est que les points non couverts sortent a densite nulle.

Pour WorldScape, le regroupement est plus simple et meilleur : **un seul appel `GetGroundNoise_Batch` par chunk**, avec la collecte de volumes faite **une fois** pour l'emprise. Cela evite au passage la dette relevee en section 8 (la boite de 1 000 km par appel unitaire).

**Comment le Landscape prepare de facon asynchrone** (`PCGLandscapeData.cpp:633-700`), le patron a copier tel quel :

1. garde par CVar (`PCGSpatialData::CVarEnablePrepareForSpatialQuery`), pour pouvoir tout desactiver ;
2. si le necessaire est deja pret, retourner `{}` : aucune tache, aucune attente ;
3. sinon construire un `FPCGScheduleGenericParams` avec un lambda qui capture en **`TWeakObjectPtr`** (exactement la regle du `ModernCPP_UE57_RzZz_Guide.md` sur les callbacks differes) et retourner `{ InContext->ScheduleGeneric(Params) }` ;
4. `Params.bCanExecuteOnlyOnMainThread = true` **en editeur seulement**, parce que la creation d'entrees de cache n'y est pas thread-safe ; `false` en runtime.

C'est la que WorldScape ferait sa collecte de volumes sur une vraie boite, et, si le generateur pose des acteurs, sa demande de chargement QLevel (`QLevel_RequestLocationLoad`, a rafraichir : c'est un bail, pas un verrou).

**Et un avantage structurel de WorldScape sur le Landscape, a ne pas manquer.** `PCGLandscapeData::PrepareForSpatialQuery` commence par refuser de travailler si le cache n'est pas serialise : `if (LandscapeCache->SerializationMode == NeverSerialize && PCGHelpers::IsRuntimeOrPIE()) return {};` (`:646`). Le Landscape a donc besoin d'un **cache cuit** pour etre echantillonnable au runtime. WorldScape n'en a aucun besoin : son bruit est **analytique**, disponible partout, y compris sur serveur dedie et dans un process NullRHI. C'est precisement ce que dit le commentaire de `GetEffectiveUseGPUNoise()`. Une Data WorldScape est donc **plus simple et plus disponible** que la Data de reference d'Epic.

### 6.6 Representer une parcelle ou une emprise : un candidat gagne, l'autre est disqualifie

Lecture des deux candidats, ce jour.

**`UPCGSplineInteriorSurfaceData` : disqualifie sur un terrain planetaire.** Sa propre documentation l'annonce (`Public/Data/PCGSplineInteriorSurfaceData.h:16`) : « Represents a surface implicitly using the **top-down 2D projection** of a closed spline ». Et le code le tient : il cache ses sommets en `TArray<FVector2D> CachedSplinePoints2D` (`:75`), son test d'appartenance s'appelle `PointInsidePolygon` et teste « the **top-down** projection » (`:58`), et surtout son `ProjectPoint` ecrit **en dur** `ProjectedTransform.SetLocation(FVector(InLocation.X, InLocation.Y, ProjectionHeight))` (`Private/Data/PCGSplineInteriorSurfaceData.cpp:86`). C'est un Z-up monde, sans transform local. Sur la Terre de QANGA, cela ne fonctionne qu'au voisinage du pole, la ou la verticale locale coincide avec +Z du monde. A l'equateur de la planete, une parcelle est vue par la tranche. **Ne pas construire de parcelle ni d'emprise de ville dessus.**

**`UPCGPolygon2DData` : le bon type, et de loin.** Sa description est exactement ce qu'il faut : « Data representing a single 2D polygon **with a 3D transform** (for spatial operations) » (`Public/Data/PCGPolygon2DData.h:70`). Et contrairement au precedent, **le code respecte ce transform de bout en bout** (`Private/Data/PCGPolygon2DData.cpp:379-406`) :

```cpp
const FTransform LocalSpaceTransform = InTransform.GetRelativeTransform(Transform);   // :382
...
OutPoint.Transform.SetLocation(Transform.TransformPosition(FVector(Center.X, Center.Y, 0.0)));  // :398
FQuat UpVectorRotation = FQuat::FindBetweenNormals(InTransform.GetRotation().GetUpVector(),
                                                   Transform.GetRotation().GetUpVector());      // :404
```

C'est le **seul type de surface d'Epic qui travaille dans un plan arbitraire**, position et rotation comprises. Un plan tangent a la planete est donc un transform legitime pour lui.

Ce qu'il apporte en plus, et qui tombe pile sur le besoin de parcellisation identifie en section 2.4 :

| Ce qu'il offre | Ou | A quoi ca sert ici |
|---|---|---|
| `UE::Geometry::FGeneralPolygon2d` avec **support des trous** | `:65`, plus `HoleIndex` et `HoleCount` dans l'enum de proprietes | une parcelle avec une mare, un ilot avec une cour |
| `SetTransform(InTransform, bCheckWinding)` | `:53` (numerotation relative) | poser le plan tangent |
| `Area`, `Perimeter`, `IsClockwise` | enum `EPCGPolygon2DDataProperties` | trier les parcelles par taille, orienter |
| **`LongestOuterSegmentIndex`** | idem | **orienter les rangs d'un champ sur le plus long cote**, exactement ce que fait ScapeEngine |
| metadonnees **par sommet** (`VertexDomainName = "Vertices"`) et **par Data** | `:43`, `GetAllSupportedMetadataDomainIDs` | une culture, un stade de croissance, un usage par parcelle |
| `LocalPosition` / `LocalRotation` en 2D contre `Position` / `Rotation` en monde | enum `EPCGPolygon2DProperties` | travailler en coordonnees de parcelle |

**Et il derive de `UPCGPolyLineData`, ce qui corrige un manque que j'avais annonce.** J'ai ecrit en section 2.4 que la fonction `PlaceAlongSpline(spline, spacing, lateralOffset, alignToTangent, projectToGround)` « manque partout ». C'est vrai cote WorldScape ; **c'est faux cote PCG** : `UPCGPolyLineData` expose `GetNumSegments()`, `GetSegmentLength(SegmentIndex)`, `GetDistanceAtSegmentStart(SegmentIndex)`, `GetLocationAtAlpha(Alpha)`, `GetTransformAtAlpha(Alpha)` et surtout **`GetTransformAtDistance(SegmentIndex, Distance, bWorldSpace, OutBounds)`**. De quoi parcourir un contour de parcelle ou une spline de route a pas constant, avec le repere a chaque station. Le mobilier de bord de route et les haies de bordure ont donc leur primitive du cote PCG, et il reste a y brancher la projection WorldScape.

**Le piege, et il est precis.** Le redressement de grille decrit en 6.4 **ne se produit que par le noeud**, pas par le chemin implicite :

- le noeud Surface Sampler passe bien le transform : `OutState.SamplerData.Initialize(Settings, Context, EffectiveGridBounds, GeneratingShape->GetTransform())` (`Private/Elements/PCGSurfaceSampler.cpp:627`) ;
- le chemin de collapse d'une Data en points passe la surcharge **sans** transform : `SamplerData.Initialize(Context, EffectiveBounds)` (`:218`), et la valeur par defaut est `FTransform::Identity` (`Public/Elements/PCGSurfaceSampler.h:44`). Donc `bNeedsLocalTransformation` est faux et **la grille de candidats est posee a plat en X et Y du monde**.

Consequence pour une `UPCGWorldScapeSurfaceData` : **ne jamais s'appuyer sur l'implementation par defaut de `CreatePointData`.** Il faut l'override et passer le transform explicitement, comme le fait `UPCGLandscapeData` (`PCGLandscapeData.cpp:432-446`), sinon tout echantillonnage implicite de la Data sera faux hors du pole.

A signaler au passage, c'est de la dette cote Epic : `UPCGPolygon2DInteriorSurfaceData::Initialize` ne fait que `Polygon = InPolygonData` (`Private/Data/PCGPolygon2DInteriorData.cpp:14-18`) et **ne renseigne jamais le `Transform` herite de `UPCGSurfaceData`**. La surface a donc un transform identite alors que son polygone en porte un vrai, et son `CreateBasePointData` appelle justement la surcharge sans transform (`:78-86`). Un polygone incline echantillonne par ce chemin voit sa densite deformee d'un facteur `1/cos(angle)`, et degenere completement a 90 degres. A contourner, pas a subir.

### 6.7 Projeter une Data sur une autre : le piege de cout, et le montage qui l'evite

Lecture de `PCGProjectionData.cpp` (492 lignes) et de la partie projection de `PCGSplineData.cpp`.

**Le piege de cout, et il est code en dur.** `UPCGProjectionData::RequiresCollapseToSample()` (`Private/Data/PCGProjectionData.cpp:213-252`) decide si echantillonner une projection exige de la **reduire en nuage de points**. Le verdict :

```cpp
bool bRequiresCollapse = ProjectionParams.bProjectPositions;                  // vrai des qu'on projette des positions
if (Cast<UPCGSplineData>(Source) && Target->GetDimension() == 2)             { bRequiresCollapse = false; }
if (Cast<UPCGLandscapeSplineData>(Source) && Target->GetDimension() == 2)    { bRequiresCollapse = false; }
```

Deux types exoneres, par `Cast<>`, et rien d'autre. Un `UPCGPolygon2DData` projete sur une surface **declenche donc le collapse**, et `SamplePoint` part alors dans `ToBasePointData(nullptr)->SamplePoint(...)` (`:180-186`), avec un commentaire d'Epic sans ambiguite : « Passing nullptr for the context means the operation will execute **single threaded** which is not ideal ». A l'echelle planetaire, c'est inacceptable.

La liste n'etant pas extensible depuis l'exterieur, la seule porte de sortie propre est de **deriver de `UPCGProjectionData`** et d'overrider `RequiresCollapseToSample()`, ce que fait deja `UPCGSplineProjectionData` et ce que le commentaire du code designe explicitement comme le patron.

**Le montage qui evite le probleme, et c'est celui a recommander.** Le Surface Sampler prend **deux** entrees : la surface generatrice et une forme de delimitation. Le bounding shape est teste **apres** la projection, et sa densite **multiplie** celle du point (`Private/Elements/PCGSurfaceSampler.cpp:325-339`) :

```cpp
if (!InBoundingShape->SamplePoint(OutPoint.Transform, OutPoint.GetLocalBounds(), BoundingShapeSample, nullptr)) { continue; }
DensityRange[WriteIndex] *= BoundingShapeSample.Density;
```

Donc :

> **Surface generatrice = `UPCGWorldScapeSurfaceData`. Forme de delimitation = la parcelle `UPCGPolygon2DData`.** Aucune projection, aucun collapse, aucun chemin single-thread. Et le degrade de bord de parcelle est gratuit, puisque la densite du polygone module celle du semis.

C'est le montage a poser d'entree pour les champs, les fermes et les ilots.

**Les splines de route : `UPCGSplineProjectionData` est a ecarter, et c'est pire qu'une normale constante.** Sa fonction `Project` (`Private/Data/PCGSplineData.cpp:705-736`) lit `GetSurface()->GetNormal()`, projette sur le plan perpendiculaire, puis **cherche la plus grande composante de la normale et jette cette coordonnee** pour obtenir un 2D. C'est un aplatissement axial, et le code porte le TODO correspondant (`:720-721`, « to support surface orientations that are not axis aligned »). Surtout, `Initialize` (`:737` et suivantes) **pre-calcule `ProjectedPosition`**, la spline entiere aplatie en 2D, une fois pour toutes avec cette normale unique. Ce n'est donc pas seulement `GetNormal()` qui est constant : **toute la geometrie 2D de la spline est figee dans un plan unique**. Sur la planete, une route de 10 km voit ses extremites fausses de 3,94 m, et la recherche du point le plus proche sur la spline se fait dans un plan qui n'est plus le bon.

**La bonne facon de faire une route, et elle rejoint exactement ScapeEngine.** Ne pas projeter la spline. La **parcourir** a pas constant avec `GetTransformAtDistance(SegmentIndex, Distance, bWorldSpace, OutBounds)` (exact, en 3D, sans aucune projection), et projeter **chaque station individuellement** sur la surface WorldScape par `ProjectPoint`. C'est le modele de `RoadsideAnchor.h` (abscisse curviligne plus une sonde de sol par station, `ScapeEngine/src/scape/RoadsideAnchor.h:16-24`), et il n'a aucun probleme de courbure par construction, a n'importe quelle longueur.

**Un point favorable trouve au passage.** `UPCGProjectionData::ProjectBounds` (`:135-166`) projette **les 8 coins de la boite individuellement** par `Target->ProjectPoint`, puis etend la boite de plus ou moins `HalfHeight` le long de `Target->GetNormal()`. La partie « 8 coins » suit donc correctement la courbure si la Data projette radialement ; seule l'extension finale utilise la normale constante, ce qui sous-dimensionne legerement la boite sur les bords. C'est benin, et de loin le moins grave des trois usages de `GetNormal()`.

### 6.8 Le montage recommande, en une page

Synthese operationnelle de 6.4 a 6.7. Rien de tout cela n'est ecrit ni teste : c'est la conception que les lectures soutiennent.

| Besoin | Montage | Pourquoi |
|---|---|---|
| **Echantillonner le terrain** | `UPCGWorldScapeSurfaceData : UPCGSurfaceData`, `Transform` = plan tangent (rotation **non nulle**, obligatoire), overrides copies de `UPCGLandscapeData`, `SamplePoints` branche sur `GetGroundNoise_Batch`, `PrepareForSpatialQuery` pour la collecte de volumes sur une vraie boite | rend WorldScape visible a tout PCG ; pas de cache a cuire, contrairement au Landscape |
| **Exposer le biome** | `FNoiseData` complet en attributs de metadonnee : `Height`, `HeightNormalize`, `Temperature`, `Humidity`, `Hole`, plus la normale reelle et la pente | tous les filtres PCG deviennent des filtres de biome sans code |
| **Delimiter une parcelle, un ilot, une emprise** | `UPCGPolygon2DData` (le seul type d'Epic a plan arbitraire), pose **en forme de delimitation** du Surface Sampler | aucune projection, aucun collapse ; `LongestOuterSegmentIndex` oriente les rangs ; trous supportes |
| **Poser du mobilier le long d'une courbe** | parcours par `GetTransformAtDistance` + `ProjectPoint` station par station | exact a toute longueur ; ne jamais passer par `UPCGSplineProjectionData` |
| **Cuire le resultat** | le graphe emet un `WorldScape_SnapDataAsset` ou un `WorldScape_GridDataAsset` ; le runtime les consomme comme aujourd'hui | contourne le `Type: Editor`, et reutilise 70 + 13 assets de plomberie deja eprouvee |
| **Ecrire les instances** | `UQInstanced_ISMComponent` / `UQInstanced_HISMComponent` | budget `BodyCreationTimeLimitMS = 0,85 ms` deja en place |
| **Cohabiter avec le streaming** | `QLevel_IsLocationReady` avant spawn, `QLevel_RequestLocationLoad` rafraichi, tick en `TG_PostUpdateWork` | section 4.5 |

Les trois interdits qui decoulent des lectures : **ne pas** utiliser `UPCGSplineInteriorSurfaceData` (Z-up monde en dur), **ne pas** projeter une Data sur une autre par `ProjectOn` generique (collapse single-thread), **ne pas** laisser la rotation du `Transform` a l'identite (grille de candidats a plat en X et Y du monde).

**Ce qui reste a lire si le chantier demarre** : `UPCGIntersectionData` (si l'on veut composer parcelle et terrain autrement qu'en forme de delimitation), et le chemin GPU de PCG (`PCGCompute`) qui est un module a part et n'a pas ete regarde du tout.

---

## 7. Ce que je n'ai pas pu verifier, et les sondes a lancer

Honnetement, ce qui reste ouvert :

| Question | Pourquoi | Sonde |
|---|---|---|
| Les valeurs de `OverrideHeight`, `EdgeFalloff`, `HeightAlpha` et l'emprise geographique **volume par volume** sur `L_Earth` | editeur ferme, pont CLIScape ferme (port 8766, verifie) | Lot 0 |
| La valeur effective de `Grid_SkipStaticMeshOnDedicated` sur `EarthScape` | serialisee sur l'acteur, pas dans un `.ini` | Lot 0 |
| Djibouti, le Finistere et la Capitale sont-ils **a l'interieur** d'un volume en override | idem | Lot 0 |
| La parite CPU / GPU des 24 volumes sur `L_Earth` | « Validation sur `L_Earth` (24 volumes heightmap, 8 trous) encore a faire », `WorldScape_GPU_Completion_Plan_2026-09-01.md:153` | validateur numerique existant, en PIE |
| `PlanetScape` : ce que son `WorldScapeNoiseBridge.cpp` (711 lignes) fait, et s'il chevauche ce chantier | hors perimetre, mais le plugin est **actif** | lecture dediee, avant le lot 3 |
| Les couts reels de `road_perf_probe`, `field_probe`, `farmprop_probe`, `windfarm_probe` | **aucune sortie de sonde enregistree sur disque** : `find` sur tout `C:\NebulaEngine` pour `*probe*`/`*bench*` en `.md/.txt/.json/.csv/.log` donne **0 resultat**. Tous les chiffres cites viennent de tableaux recopies dans les docs ou de constantes lues dans le code des sondes | executer les sondes avant le lot 3, en particulier la partie D de `road_perf_probe` qui mesure la politique de re-tranche du drapage |
| `FastNoise2` est-il thread-safe | dans `external/`, hors perimetre ; reserve explicite de l'auteur non levee (`ASYNC_GENERATION.md:95-98`) | non verifie |
| `Content/Tools/IA_Scatter/` (avec panneau de controle et presets), `Content/Tools/QuadTreeInstanced/`, `Content/StarMap/3D_Capture/RoadCut/` (`RoadCut_DynMesh`, `RoadCut_BaseMesh_Asset`, 10 occurrences de « Capture », ni Landscape ni Spline ni WorldScape) | non analyses en detail | lecture dediee si le lot 4 demarre |
| L'ecriture d'assets depuis un graphe PCG est-elle deja outillee | `Tool_CreatePCGPreset.cpp` et `Tool_SpawnPCGInLevel.cpp` de CLIScape non lus | lecture dediee, avant d'investir sur PCG (6.2) |
| Le cablage et les reglages exacts des graphes PCG existants | non lisibles par extraction de table de noms : seuls les **types** de noeuds sont identifiables, pas leur cablage | ouvrir `PGC_GRASS_Nord` et `PCG_City` en editeur |
| La logique interne des Blueprints cites (`RoadTool`, `WS_Road_Spline_Former`, `Bridge`, `BP_FieldSpline`) | deduite de la table de noms et de la table d'imports : cela donne les fonctions appelees et les dependances de module avec certitude, **pas la logique de graphe**. L'affirmation « `WS_Road_Spline_Former` ne lit pas la hauteur du terrain » repose sur l'absence de tout import `/Script/WorldScapeCore` : preuve solide mais indirecte | `get_detailed_blueprint_summary` en editeur |

Deux precisions de methode qui changent la lecture des chiffres de ScapeEngine :

- **`terrain_lod_benchmark.cpp` n'est pas une mesure** (voir 1.5). Ses recommandations sont des conjectures sur un rayon de 6 371 km alors que la planete de reference fait 300 km.
- **`road_profile_probe.cpp` porte une copie perimee** de l'algorithme : ses chiffres decrivent « un reseau qui n'existe plus » (`tools/probes/README.md:48-52`). C'est de l'histoire.

---

## 8. Dette relevee en passant, hors perimetre, non corrigee

- `GetGroundNoise_HOnly` collecte les volumes avec une boite de **1 000 km de cote** a chaque appel unitaire ([WorldScapeRoot_Noise.cpp:76](../Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScapeRoot_Noise.cpp)), avec juste au-dessus la ligne commentee d'une boite serree. Trois allocations de `TArray` et la copie des 24 descripteurs par appel. `FHeightMapVolumeDataCopy` garde un **pointeur** vers `UHeightMapVolumeData` (`HeightMapVolumeDataCopy.h:316`), donc les pixels ne sont pas copies : le coût est modere, mais reel pour tout appelant massif.
- Les huit delegues `WS_OnStart*` / `WS_OnEnd*` sont commentes (`WorldScapeRoot.h:440-469`). Un lecteur presse croira qu'ils existent.
- `WorldScapeTestPCGNode` : `HeightScale` declaree et jamais lue ; `HeightOffset` clampe a 10 m.
- `RoadTool.uasset` reference `/Game/Blueprint/Tools/RoadTools/RapidRoadTool`, **chemin absent du disque**. Reference cassee.
- `PCG_City.uasset` porte un nom qui promet un generateur de ville et n'en est pas un.
- `QBuilder_GetISMTransform(..., const bool WorldScape)` : parametre mal nomme, c'est `bWorldSpace`.

---

## Changelog Discord (pret a coller)

**🗺️ Analyse : porter la generation de surface du moteur maison dans WorldScape**

**Le contexte.** Le chemin GPU de WorldScape marche enfin. Question posee : comment y amener le relief, les champs, les fermes, les routes, les ponts et les villes qui tournent deja a 100 % sur le moteur maison, **sans bouger d'un centimetre les hauteurs de la Terre** ou des niveaux entiers sont poses a la main.

**Ce que l'analyse etablit.**
Sur les sept systemes, **cinq ne touchent pas la hauteur du sol** : mobilier de bord de route, fermes, ponts (par construction), tunnels, et les champs dont le nivellement est a zero par defaut. **Deux la touchent** : le relief et les routes, et pour ces deux la le garde-fou existe deja, ce sont les 24 volumes heightmap de la Terre.

**Le chiffre qui change tout.** La Terre du jeu fait **229 Frances** quand la planete du moteur maison en fait 2. Un reseau routier planetaire a la meme densite ferait **119 Go**. Donc le reseau sera **local aux lieux**, pas planetaire : 141 relais avec 20 km de route chacun tiennent dans 22 Mo.

**Le principe retenu.** Le serveur ne stockera **aucune geometrie** : il connaitra les routes par une **fonction**, comme il connait deja le sol, la gravite et la distance a l'eau. Le ruban, les props et les batiments restent **cote client uniquement**. C'est exactement la separation que le moteur maison a deja faite.

**La bonne surprise, et elle est grosse.** Une premiere lecture concluait qu'il n'y avait ni champs, ni fermes, ni rues, ni villes cote Unreal. **C'etait faux, quatre fois.** Il y a deja un champ de tournesols avec stades de croissance colle a la planete, un kit de ferme complet en huit collections avec proxys, onze revetements de rue a trois echelles resolus sur le terrain planetaire, et treize quartiers de Djibouti. Autrement dit : **il ne manque pas des systemes, il manque des regles.** Le travail n'est pas d'ecrire un poseur, c'est d'ecrire ce qui decide ou poser.

Document complet : `Documentation/WorldScape_Surface_Portage_ScapeEngine_2026-09-18.md`
