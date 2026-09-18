# WorldScape : pourquoi le quadtree GPU coûte plus cher que le clipmap CPU à résolution égale

Diagnostic par lecture de code, 2026-09-12. Aucune modification, aucune mesure en jeu (éditeur fermé, aucune ligne `WS IndirectTerrain` dans les logs récents). Chaque affirmation cite le fichier et la ligne lus ce jour. Les chiffres mesurés cités viennent des commentaires laissés dans le code par les sessions de septembre (L_Dev_Claude, viewport 711 px) et sont signalés comme tels.

Racine des chemins : `Plugins/WorldScape/`. Moteur : `C:/UE5_Share/Engine/Source/Runtime/`.

## 0. Résumé

Le mot « 128 » ne désigne pas la même quantité dans les deux chemins, et le chemin GPU paie en plus quatre surcoûts que le clipmap n'a pas. Par ordre d'importance :

| # | Cause | Où dans le code | Effet attendu |
|---|---|---|---|
| 1 | **Densité de sommets** : 128 = cellules d'UN anneau centré caméra (clipmap) contre 128 = sommets de CHAQUE feuille (quadtree). Pour la même taille de cellule à une distance donnée, le quadtree pose 4 à 9 tuiles carrées là où le clipmap pose 1 anneau. Deux amplificateurs : test d'horizon élargi de 33 km sur Terre, MaxDepth 16 = cellules de 76 cm sous la caméra (clipmap : 100 cm). | `WSQuadtreeManager.cpp:573-582, 785-800, 843-853` ; `WorldScapeRoot_Main.cpp:2727-2731` ; commentaires `WSQuadtreeManager.h:34-38`, `WorldScapeRoot.h:1907` | Clipmap 12 anneaux = 155 708 sommets par passe. Quadtree mesuré (sessions précédentes) : 2,1 M à 6 px, 5,6 M à 2 px, soit 13 à 36 fois plus ; à 2 px « 2,5x plus lent », au seuil auto 2,75 « 75 % de la cadence du clipmap ». |
| 2 | **Génération des tuiles en double précision sur le GPU, dans l'image** : environ 104 évaluations de simplex fp64 par texel, 18 496 texels par tuile, débit fp64 à 1/64 sur GeForce. | `WSHeightfieldGenerate.usf`, `WorldScapeNoiseEarthBiomes.ush:238-342`, `WorldScapeNoise.ush:1282-1937` ; régulateur `WSHeightfieldManager.cpp:1213-1240` | 2 à 8 ms de GPU par tuile de 128 selon la carte, sur la file graphique ; le régulateur ne mesure pas ce temps GPU. En marchant au sol, plusieurs tuiles par image. Le clipmap calcule son bruit sur des threads CPU hors de l'image. |
| 3 | **Chaque sommet coûte plus cher et est dessiné dans plus de passes** : vertex factory sans variante position-only (le VS complet tourne en prépasse, en ombres VSM et en custom depth), 7 loads dépendants sur 3 niveaux + loads GPU Scene par sommet, travail par instance refait par sommet. | `WSGPUTerrainVertexFactory.cpp:121-124` ; `WSGPUTerrainVertexFactory.ush:250-271, 299-326` ; moteur `DepthRendering.cpp:1138`, `ShadowDepthRendering.cpp:373, 2288` | Multiplie le coût de la cause 1 par passe. Le clipmap utilise `FLocalVertexFactory` (`WorldScapeMeshComponent.cpp:89`) qui a le flux position-only. |
| 4 | **Primitive dynamique par image, donc jamais mise en cache par les Virtual Shadow Maps** : les tuiles sont des instances `DynamicPrimitiveData` réémises à chaque image ; le moteur les traite comme dynamiques par définition, invalide leurs pages chaque image et les rasterise une fois par niveau de clipmap traversé (11 niveaux par défaut), avec le VS complet. | `WSGPUTerrainRenderer.cpp:273-291, 541-557, 2095-2110` ; moteur `VirtualShadowMapPageCacheCommon.ush:51-62`, `GPUScene.cpp:1821-1831`, `VirtualShadowMapArray.cpp:3697-3707` | Jusqu'à N tuiles x 16 384 sommets x k niveaux par image en ombre, sans cache. Le clipmap est un jeu de primitives persistantes en `Static` : pages d'ombre en cache, rendues seulement à la régénération d'un anneau. |
| 5 | **Surcoûts fixes par image** : traversée du quadtree en double sur le render thread, une requête de heightfield par noeud visité, recalcul des centres et bornes de toutes les tuiles, un RDG autonome exécuté chaque image, SRV neuf chaque image ; VRAM de 0,6 à 1,3 Go. | `WSGPUTerrainRenderer.cpp:1656-1670, 2013-2080, 2095-2233` ; `WSHeightfieldManager.cpp:880-955` ; `WorldScapeRoot_Main.cpp:1728` | Quelques ms de render thread (non mesuré) et une pression VRAM absente du clipmap. |

Le verrou à 30 fps n'a pas de cause dans le code de WorldScape : c'est la quantification VSync (une image à plus de 16,7 ms tombe à 30 exactement) ou l'option « FPS max » du menu. Voir §8.

Ce qui est innocenté : en mode GPU le clipmap CPU ne tourne plus du tout (pas de double travail), le keeper mesh n'est pas recréé chaque image, il n'y a aucun readback GPU synchrone dans ce chemin, la collision et le foliage sont identiques dans les deux modes. Voir §9.

## 1. Périmètre et méthode

- Question posée : pourquoi, à LodResolution 128 des deux côtés, le quadtree GPU (`bUseGPUNoise` + `bUseIndirectInstancedNoise`) est nettement moins performant que le clipmap CPU, et pourquoi les joueurs tombent à 30 fps.
- Méthode : lecture complète du module `WorldScapeGPUTerrain` (renderer, heightfield manager, quadtree, vertex factory), des shaders GPU, du tick de `AWorldScapeRoot` (intégration, régulateurs, keeper mesh), du chemin clipmap CPU pour le baseline, et des points du moteur 5.7 qui décident du coût des passes (position-only, GPU Scene, VSM).
- Aucune mesure : l'éditeur était fermé, et les logs `Saved/Logs/QANGA.log` et le backup du 11/09 ne contiennent aucune ligne de stats `WS IndirectTerrain`. Tous les ordres de grandeur sont annoncés comme des estimations ou comme des mesures antérieures consignées dans le code.
- Activation du chemin GPU : aucun `.ini`, aucun DataAsset QScalability ni aucun `.umap` de l'Univers ne sérialise `bUseIndirectInstancedNoise` ou `bUseGPUNoise` (grep binaire sur `L_Earth.umap` : 0 occurrence). Il s'active par les flags de l'acteur ou par les CVars `ws.GPUNoise` / `ws.IndirectInstancing` (`WorldScapeRoot_Main.cpp:182-193`). `L_Dev_Claude.umap` les sérialise.

## 2. Ce que « 128 » veut dire dans chaque chemin

| | Clipmap CPU | Quadtree GPU |
|---|---|---|
| Unité de « 128 » | Cellules par côté d'un anneau centré sur la caméra, un anneau par LOD (`MaxLod` anneaux) | Sommets par côté de chaque feuille du quadtree (`IndirectNoise_MeshResolution`, sinon `LodResolution`, `WorldScapeRoot_Main.cpp:1687`) |
| Nombre d'éléments | Fixe : `MaxLod` anneaux (12 sur la Terre de prod) quelle que soit la scène | Variable : décidé par l'erreur écran, plafonné par les caps de feuilles (512 terre + 128 océan par défaut, `Main.cpp:1697-1698`) et le pool de sommets |
| Sommets par élément | 16 382 au LOD 0, 12 666 par anneau troué (4 bandes, `WorldScapeLod.cpp:60-241`) | 128 x 128 = 16 384 (grille pleine, pas de trou) |
| Total de sommets | Borné : 155 708 sommets, 293 706 triangles pour 12 anneaux (§3) | Mesuré (sessions précédentes) : 5,6 M à 2 px, 2,1 M à 6 px ; à 8 px (auto 2,75 sur 711 px) « 75 % de la cadence du clipmap » |
| Draws par passe | 36 (3 sections x 12 anneaux) | 1 à 4 (un par groupe index buffer x matériau x sens x eau), instanciés |
| Cellule la plus fine | `TriangleSize` = 100 cm sur la Terre | Face du cube 6 338 km / 2^16 / 127 = 76 cm à `Quadtree_MaxDepth` 16 (`WorldScapeRoot.h:1911`) |
| Étendue dessinée | 128 x TriangleSize x 2^(MaxLod-1) x multiplicateur d'altitude | Jusqu'à l'horizon élargi de la marge de hauteur (§4.2) |

Autrement dit, régler le quadtree à 128 revient à choisir une taille de tuile, pas une densité globale. La densité globale est pilotée par `Quadtree_ScreenSpaceErrorThresholdPixels` (0 = auto, facteur `ws.GPUTerrain.Quadtree.AutoThresholdFactor` = 2,75) et par `Quadtree_MaxDepth`.

## 3. Baseline du clipmap CPU (coût par image)

Lecture de `WorldScapeLod.cpp`, `WorldScapeMeshComponent.cpp`, `WorldScapeRoot_Main.cpp`, `WorldScapeRoot_Thread.cpp` et du moteur (`StaticMeshVertexBuffer.cpp`, `LocalVertexFactory.cpp`, `ShadowDepthRendering.cpp`, `VirtualShadowMapCacheManager.cpp`).

| Poste | Valeur à LodResolution 128 | Source |
|---|---|---|
| Anneaux | `MaxLod` composants `UWorldScapeLod` (défaut C++ 8 ; 12 sur L_Earth d'après la sonde du 2026-09-01, non re-vérifié ce jour), un `UWorldScapeMeshComponent` chacun | `WorldScapeRoot_Main.cpp:1115, 3858-3860` |
| Sections par anneau | 3 (anneau troué, bande verticale, bande horizontale) | `WorldScapeLod.cpp:93, 172, 207, 240` |
| Sommets et triangles | LOD 0 : 16 382 sommets, 31 752 triangles ; LOD i > 0 : 12 666 sommets, 23 814 triangles (anneau en 4 bandes, trou central de 62 x 62 cellules) | formules `WorldScapeLod.cpp:60-241` |
| Total 12 anneaux | **155 708 sommets, 293 706 triangles, 36 draws par passe** ; 9,75 Mo de VRAM (sommets 6,23 Mo + indices 3,52 Mo) | calcul |
| Étendue | anneau i = 127 x TriangleSize x 2^i x multiplicateur d'altitude, soit 260 km de côté au LOD 11 avec TriangleSize 100 | `WorldScapeRoot_Thread.cpp:1350` |
| Océan | 0 anneau sur L_Earth (`bOcean = False`) ; sinon `OceanMaxLod` anneaux de 32 (7 028 sommets pour 8 anneaux) | `Main.cpp:1116, 1123, 3927-3982` |
| Type de draw | dynamique (`bDynamicRelevance = true`), un `FDynamicPrimitiveUniformBuffer` par section et par vue ; mais primitives et instances PERSISTANTES dans GPU Scene (pas de `DynamicPrimitiveData`) | `WorldScapeMeshComponent.cpp:446-541, 551` |
| Vertex factory | `FLocalVertexFactory` sur `FStaticMeshVertexBuffers` : 40 octets par sommet en 4 flux (position 12, tangentes 8, UV 16, couleur 4), index 32 bits ; flux position-only disponible pour la prépasse et les ombres (si le matériau est opaque sans WPO, non vérifié sur M_EarthBase) | `WorldScapeMeshComponent.cpp:85-89, 250` ; moteur `LocalVertexFactory.cpp:433-454`, `ShadowDepthRendering.cpp:2093-2096` |
| Régénération | anneau i régénéré au franchissement d'une cellule de ce LOD (LOD 0 : tous les 100 cm), ou tous les anneaux au changement de multiplicateur d'altitude ; travail sur `FAsyncTask` (workers), hors image | `Main.cpp:4735-4758`, `WorldScapeWorldType.cpp:214-277` |
| Upload par anneau régénéré | 1 `ENQUEUE_RENDER_COMMAND` pour les 3 sections, 4 `LockBuffer/Unlock` en place par section (0,5 à 0,65 Mo par anneau), index jamais retouchés, ni proxy ni buffer recréés ; une mise à jour de transform GPU Scene | `WorldScapeMeshComponent.cpp:336-387, 1037-1043`, `WorldScapeLod.cpp:338` |
| Coût par image sans mouvement | aucun upload ; tick à 60 Hz depuis Slate avec un échantillon de bruit, un `GetSnappedPosition` par anneau, le handler de collision et `CheckForHeightmapModifier` (identique en mode GPU jusqu'au retour anticipé de `UpdatePosition`) | `Main.cpp:1128, 1147, 4557-4573, 4660-4664, 4733-4790, 5432-5485` |
| Ombres VSM | Movable, `ShadowCacheInvalidationBehavior = TerrainVSMShadow_Invalidation` : défaut C++ `Auto`, `Static` sur L_Earth (sonde 2026-09-01). En `Static` : primitives cachées, aucune invalidation en régime stable. En `Auto` : le proxy ne surcharge pas `bHasDeformableMesh` (vrai par défaut), donc invalidation à chaque image aussi | `WorldScapeRoot.h:603`, `Main.cpp:3869` ; moteur `PrimitiveSceneProxy.cpp:364`, `VirtualShadowMapCacheManager.cpp:556-575, 1339-1352` |
| Culling | aucun anneau n'est jamais frustum-cullé (chaque anneau contient la caméra) ; prépasse complète forcée par `r.EarlyZPass=3` | `WorldScapeMeshComponent.cpp:466-517, 1388-1397` |

En mode GPU, ce chemin est complètement coupé : `UpdatePosition` retourne avant la boucle des anneaux (`WorldScapeRoot_Main.cpp:4690-4697`), `CheckForLodGeneration` annule les tâches (`4805-4813`), et le worker ne calcule pas de bruit (`WorldScapeRoot_Thread.cpp:1519-1531`). Il n'y a donc pas de double travail à imputer au mode GPU.

Comparaison brute, même LodResolution 128, mesures des sessions précédentes pour le quadtree : 2,1 M de sommets (136 tuiles, seuil 6 px) à 5,6 M (345 tuiles, 2 px) contre 155 708 pour 12 anneaux, soit **13 à 36 fois plus de sommets** dessinés par passe. Le tooltip de `Quadtree_ScreenSpaceErrorThresholdPixels` parle de « 7 fois » à 2 px (`WorldScapeRoot.h:1907`) : la référence clipmap de cette mesure n'est pas connue, mais l'ordre de grandeur (au moins 10x) ne fait pas de doute.

## 4. Cause 1 : la densité de sommets

### 4.1 Quadtree contre anneaux

Critère de subdivision (`WSQuadtreeManager.cpp:573-582` puis `843-853`) : une tuile de taille S à distance D (distance du centre projeté moins 0,707 S) est divisée tant que `S / 127 / D x PixelsPerUnit > 2,75 x PixelsPerUnit / 127`, soit tant que `S > 2,75 D`. Une feuille a donc une taille comprise entre 1,375 D et 2,75 D. Ses cellules mesurent entre D/92 et D/46. Le clipmap place un point à distance d dans l'anneau L tel que 32 c_L <= d < 64 c_L, donc des cellules entre d/64 et d/32. La densité linéaire du quadtree est donc environ 1,4 fois plus fine à distance égale, soit environ 2 fois plus de triangles par unité de surface, et cela sans compter les deux points suivants.

Le vrai facteur est structurel : pour couvrir la bande de distances d'un niveau donné, le clipmap pose exactement un anneau de 16 384 sommets (dont le quart central est troué dans les indices). Le quadtree pose des tuiles carrées, jamais centrées sur la caméra, sans trou : une bande couverte par des tuiles de côté S entre les distances S/2,75 et S/1,375 nécessite typiquement 4 à 9 tuiles selon l'alignement de la caméra dans la grille. C'est 4 à 9 fois 16 384 sommets par niveau, contre 16 384 pour le clipmap. Cela rejoint les mesures des sessions précédentes consignées dans le code : « A fixed 2 px draws about 7 times more vertices than the clipmap » (`WorldScapeRoot.h:1907`) et « the old fixed 2 px drew 5.7 M vertices, 2.5x slower than the clipmap » (`WSQuadtreeManager.h:34-38`).

### 4.2 Amplificateur : l'horizon élargi de la marge de hauteur

Le test de visibilité (`WSQuadtreeManager.cpp:785-800`) élargit la boîte caméra ET le test d'horizon de `MaxDisplacementWorld`, qui vaut `HeightMargin = |NoiseIntensity| + |OceanHeight| (+ |PlanetaryAltitude| + |PlanetaryMapHeight|)` (`WorldScapeRoot_Main.cpp:2727-2731`). Sur la Terre : 22 km (NoiseIntensity 2,2e6 cm) + 11,2 km (OceanHeight) + les termes planétaires, soit au moins 33 km. À 2 m du sol sur un rayon de 3 169 km, l'horizon géométrique est à environ 5 km ; avec 33 km de marge, un point situé jusqu'à sqrt(2 x 3 169 x 33) = environ 457 km passe le test. Le relief réel culmine vers 9 km. Le quadtree accepte donc, au sol, des tuiles de profondeur 4 à 8 (400 à 25 km de côté) sur des centaines de km derrière l'horizon, chacune à 16 384 sommets, invisibles à l'écran mais dessinées dans la prépasse, la base pass et les ombres (le culling GPU Scene par boîte d'instance n'aide pas : ces boîtes sont elles aussi élargies de la marge, `WSGPUTerrainRenderer.cpp:112-146`). Le clipmap, lui, s'arrête à l'étendue de son dernier anneau.

### 4.3 Amplificateur : MaxDepth 16 sous la caméra

`Quadtree_MaxDepth` vaut 16 par défaut (`WorldScapeRoot.h:1911`, non sérialisé dans `L_Earth.umap`). Sur la Terre, une tuile de profondeur 16 fait 6 338 km / 65 536 = 96,7 m, soit des cellules de 76 cm, plus fines que le `TriangleSize` de 100 cm du clipmap : 1,7 fois plus de triangles par m² près de la caméra, là où les tuiles sont les plus nombreuses.

## 5. Cause 2 : la génération des tuiles, en double précision, dans l'image

### 5.1 Coût d'une tuile

Rapport shaders (lecture intégrale de `WSHeightfieldGenerate.usf`, `WorldScapeNoise.ush`, `WorldScapeNoiseEarthBiomes.ush`, `WSPreciseTrigD.ush`, `WSHeightfieldVolumes.ush`, `WSMeshGenerate.usf`) :

| Étape | Par tuile de 128 | Citation |
|---|---|---|
| Texels de heightfield | (128 + 2 x 4)² = 18 496, groupes 8 x 8 = 289 groupes | `WSHeightfieldGenerate.usf:978-983`, `WSHeightfieldManager.cpp:2073-2083` |
| Bruit Terre par texel | environ 104 évaluations de `WS_OpenSimplexNoiseD` (toutes en `double3`, variables `precise` donc sans FMA, fractales en `[loop]`), soit environ 21 000 instructions double et 2 200 loads de tables dépendants | `WorldScapeNoiseEarthBiomes.ush:238-342`, `WorldScapeNoise.ush:1282-1937, 1966, 1986` |
| Trigonométrie et heightmap planétaire par texel | `WS_Atan2PreciseD` + `WS_AsinPreciseD` (16 divisions double, 2 Horner degré 9), racine et rotation quaternion calculées deux fois, 4 taps (bilinéaire) ou 16 taps (bicubique) par masque avec le tiling recalculé en double à chaque tap | `WSHeightfieldGenerate.usf:451-463, 505-537, 1023-1048`, `WSPreciseTrigD.ush:20-126` |
| Volumes (permutation `WS_WITH_VOLUMES`, 24 volumes heightmap + 8 trous sur la Terre) | boucle sur TOUS les volumes par texel, transformation inverse en double par trou sans test de distance | `WSHeightfieldVolumes.ush:374-402, 544-555` |
| Maillage par sommet | `ComputeTileReferenceOffset` (uniforme par tuile mais exécuté par chaque thread) + 5 à 7 projections sphère en double : environ 18 divisions double et 270 opérations double | `WSMeshGenerate.usf:178-266, 280, 322, 361-364` |
| Dispatches | 2 par tuile (heightfield, maillage) plus les transitions | `WSHeightfieldManager.cpp:2078`, `WSGPUTerrainRenderer.cpp:2281` |

Total estimé : environ 4 x 10^8 instructions double par tuile, dont plus de 90 % dans le simplex. Débit fp64 : 1/64 du fp32 sur GeForce Ampere et Ada, 1/32 sur Turing, 1/16 sur Radeon RDNA. Temps ALU pur à 100 % d'occupation : 0,9 ms (RTX 4070), 2,1 ms (RTX 3060), 0,5 ms (RX 6700 XT) ; avec la divergence de la branche d'octant, la pression registre et les boucles latence-bound, l'estimation réaliste est de 2 à 4 ms par tuile sur 4070 et 4 à 8 ms sur 3060, sur la file graphique, donc pris sur l'image. Le clipmap calcule le même bruit en double sur des cœurs x86 à plein débit fp64, sur des workers, hors chemin critique.

### 5.2 Pourquoi cela touche chaque image en marchant

- Chaque noeud visité par le quadtree demande sa tuile (`WSQuadtreeManager.cpp:1013-1130`), et chaque split ou merge produit des tuiles à générer. Au sol, la caméra traverse en permanence les frontières des petites tuiles (97 m à la profondeur 16, 193 m à 15, etc.), donc les feuilles proches changent en continu.
- Le régulateur (`WSHeightfieldManager.cpp:1213-1240`) ne mesure que le temps CPU de soumission (budget 6 ms, `Main.cpp:160-165`) et le temps d'image relatif, avec un plancher de 4 tuiles par tick, un plafond de 64 et une rafale urgente de 8 (`WSHeightfieldManager.h:151-155`). Le commentaire des lignes 1214-1216 le dit : « The CPU submit time says nothing about the GPU cost of a 128 x 128 Earth tile ». Quatre tuiles par tick sur une 3060, c'est 16 à 32 ms de GPU dans l'image.
- À chaque tuile générée, le maillage doit aussi être régénéré (nouvelle tuile = obligatoire, `WSGPUTerrainRenderer.cpp:1895-1925`), et les voisins dont le masque de couture change sont remaillés (`1898-1901`).
- Le cache est un LRU sous budget (3 x caps x 361 Ko par tuile, clamp 128 Mo à 2 Go, `Main.cpp:1728`), donc au sol un demi-tour ré-demande des tuiles évincées (mesuré le 03/09, voir mémoire du projet).
- Ordre de grandeur VRAM en mode GPU (estimation du 2026-09-12) : 0,55 à 1,3 Go (cache HF 3 x caps x 361 Ko + pool de sommets <= 768 Mo), contre ~10 Mo pour le clipmap.

## 6. Cause 3 : le coût par sommet et le nombre de passes

### 6.1 Vertex factory

`FWSTerrainVertexFactory` (`WSGPUTerrainVertexFactory.cpp:121-124`) déclare `UsedWithMaterials | SupportsDynamicLighting | SupportsPrecisePrevWorldPos | SupportsPrimitiveIdStream`, et rien d'autre : ni `SupportsPositionOnly`, ni `SupportsPositionAndNormalOnly`, ni `SupportsCachingMeshDrawCommands`. Conséquence moteur (`DepthRendering.cpp:1138`, `ShadowDepthRendering.cpp:373, 2288`) : la prépasse de profondeur (forcée complète par `r.EarlyZPass=3`, `Config/DefaultEngine.ini:288`), les ombres (VSM) et le custom depth utilisent le flux par défaut, donc `GetVertexFactoryIntermediates` complet. Le clipmap CPU utilise `FLocalVertexFactory` qui a ce flux position-only (12 octets par sommet en profondeur et en ombre).

Par sommet et par passe (`WSGPUTerrainVertexFactory.ush`) : loads GPU Scene de la primitive et de l'instance (`:223-233`), puis `WSInstanceData.Load4` x2 (`:254-259`), puis les 3 pads des sommets 0, 1, 2 de la tuile (`:263-267`), puis `WSVertexData.Load4` x2 (`:269-271`) : 7 loads explicites en chaîne dépendante à 3 niveaux, plus environ 150 à 200 opérations dont 45 `precise` pour les deux `WSTwoSum` (`:299-326`). Les 5 loads d'instance et de référence et le TwoSum sont identiques pour les 16 384 sommets d'une tuile.

### 6.2 Passes

Chaque sommet visible est traité au minimum dans : la prépasse, la base pass, et chaque niveau de clipmap VSM où son instance est visible (section 7). Avec 2 M de sommets (auto) à 5,7 M (2 px) et 4 à 8 passes, on est entre 10 M et 45 M d'invocations du VS complet par image, avec 15 requêtes mémoire chacune. Ordre de grandeur : 1 à 3 ms de GPU sur une carte milieu de gamme rien qu'en vertex shading (estimation, non mesurée).

### 6.3 Permutations

`ShouldCompilePermutation` (`WSGPUTerrainVertexFactory.cpp:84-94`) accepte tout vertex et tout pixel shader de tout matériau du projet : chaque matériau embarque les permutations de cette VF (compilation, DDC, mémoire des shader maps, deux variantes par passe dans le pré-cache PSO faute de position-only). Ce n'est pas un coût par image mais un coût de chargement et un risque de hitch à la première apparition d'un matériau. Le heightfield a 10 kernels (5 types de bruit x volumes, `WSHeightfieldManager.cpp:180-182`).

## 7. Cause 4 : primitive dynamique par image et Virtual Shadow Maps

### 7.1 Ce que fait le proxy

`FWSGPUTerrainSceneProxy::GetDynamicMeshElements` (`WSGPUTerrainRenderer.cpp:300-560`) : pour chaque vue, un `FDynamicPrimitiveUniformBuffer` alloué pour l'image (`:435`), puis un `FMeshBatch` par groupe de tuiles avec `NumInstances = tuiles du groupe` et `DynamicPrimitiveData = FWSTerrainBatchInstances` (`:541-557`), structure allouée pour l'image qui contient N `FInstanceSceneData` identité et une copie des N bornes locales (`:273-291`). `GetViewRelevance` rend `bDynamicRelevance = true` (`:565-585`). Côté tick, le buffer d'instances est recréé et re-uploadé chaque image (`:2095-2110`), converti en accès externe (`:2205-2206`), avec un `GraphBuilder.Execute()` autonome chaque image même sans dispatch (`:2209`) et une SRV neuve (`:2229-2233`).

Le keeper mesh est `Movable` et reçoit les mêmes flags d'ombre que les anneaux CPU, dont `ShadowCacheInvalidationBehavior = TerrainVSMShadow_Invalidation` (`WorldScapeRoot_Main.cpp:1044-1062`).

### 7.2 Conséquence sur le VSM (vérifié dans le moteur 5.7, citations en annexe B)

1. **Les instances dynamiques sont réallouées et ré-uploadées à chaque image, et une fois par vue.** Le collecteur de primitives dynamiques appartient à la vue (`SceneRendering.h:1813`), les IDs de primitive sont remis à zéro en début et en fin d'image (`GPUScene.cpp:738, 757-762`), les slots d'instances sont libérés en fin d'image (`GPUScene.cpp:2398-2413`). Chaque ombre VSM a son propre collecteur et son propre upload (`ShadowDepthRendering.cpp:1258-1265`, `ShadowSetup.cpp:6758-6761`). Pour N tuiles, une directionnelle et L lumières locales VSM : N x (2 + L) entrées d'instances GPU Scene uploadées par image.

2. **Ces instances ne peuvent jamais être mises en cache par le VSM.** Le test GPU `GetCachePrimitiveAsDynamic` rend « dynamique » tout primitive dont l'index persistant dépasse le maximum (`VirtualShadowMapPageCacheCommon.ush:51-62`) ; un `FDynamicPrimitiveUniformBuffer` part avec `PersistentPrimitiveIndex = INDEX_NONE` (`PrimitiveUniformShaderParametersBuilder.h:77`). Le `ShadowCacheInvalidationBehavior::Static` posé sur le keeper mesh n'a donc aucun effet sur ce chemin : il ne concerne que le bit-array CPU indexé par index persistant (`VirtualShadowMapCacheManager.cpp:553-576`).

3. **Toutes les instances dynamiques déclenchent une invalidation de pages explicite à chaque image**, une entrée par instance et par identifiant de VSM (`GPUScene.cpp:1821-1831`, `VirtualShadowMapCacheManager.cpp:618-624`). Avec 700 tuiles et un clipmap de 11 niveaux, 7 700 entrées d'invalidation par image et par lumière. Comme le terrain couvre tout l'écran, la couche dynamique de toutes les pages visibles est re-rendue chaque image, ce qui entraîne aussi tous les autres objets dynamiques de ces pages (personnages, véhicules).

4. **En rendu non Nanite, une instance est rasterisée une fois par niveau de clipmap qu'elle recoupe** (`VirtualShadowMapArray.cpp:3697-3707`, `VirtualShadowMapBuildPerPageDrawCommands.usf:224-405`) ; le clipmap directionnel a 11 niveaux par défaut (`VirtualShadowMapClipmap.h:28-29`). Une tuile de 16 384 sommets visible dans k niveaux coûte k x 16 384 invocations du VS complet (§6.1) en passe d'ombre. Borne haute donnée par le moteur : 700 tuiles x 16 384 x 11 = 123 M de sommets par image rien qu'en ombre, sans cache.

5. **`GetDynamicMeshElements` est appelé 1 + 1 + L fois par image** (vue principale, clipmap directionnel, chaque lumière locale VSM ; `SceneVisibility.cpp:4152-4158`, `ShadowSetup.cpp:2842-2844, 4678-4687, 4746-4749`), chaque appel reconstruisant tous les `FMeshBatch`, un `FDynamicPrimitiveUniformBuffer` et le bloc d'instances.

6. **Le clipmap CPU, lui, est cachable** : ses composants persistants ont un index persistant valide ; avec `TerrainVSMShadow_Invalidation = Static` (valeur relevée sur L_Earth par la sonde du 2026-09-01) il n'est jamais marqué dynamique et ses invalidations de transform sont supprimées (`VirtualShadowMapCacheManager.cpp:561-575`) : ses pages d'ombre ne sont rendues qu'à la régénération des anneaux. Nuance à retenir : en `Auto` (défaut C++), le proxy CPU ne surcharge pas `bHasDeformableMesh` (vrai par défaut, `PrimitiveSceneProxy.cpp:364`) et serait lui aussi invalidé à chaque image (`VirtualShadowMapCacheManager.cpp:1339-1345`) ; l'avantage VSM du clipmap dépend donc du réglage `Static` sur la planète, à vérifier sur l'acteur de l'Univers utilisé pour la comparaison. Même dans ce cas défavorable, le clipmap re-rasterise 156 k sommets, le quadtree des millions.

Réglages du projet qui rendent ce point sensible : `r.Shadow.Virtual.Enable=1`, `r.Shadow.Virtual.ForceOnlyVirtualShadowMaps=1` (`DefaultEngine.ini:143, 400`), `r.Shadow.Virtual.MaxPhysicalPages=512` au niveau ShadowQuality 0 (`DefaultScalability.ini:135`).

## 8. Le verrou à 30 fps

Rien dans WorldScape ne plafonne la cadence. Deux mécanismes expliquent un 30 exact :
- **VSync** : dès qu'une image dépasse 16,7 ms, l'affichage tombe à 30 (une image sur deux). Le jeu expose l'option (`SG_Screen_VSync`, `DefaultGame.ini:323`).
- **FPS max** : le jeu a une option « FPS Max » (`Content/Systems/Scalability_Sys/QangaScalability/Common/SG_Common_FPS`, variable `FPSmax` appliquée en `FrameRateLimit` ; `QangaPlayerController` et `W_GraphicsSettings` y font référence). Sa valeur par défaut n'a pas pu être lue (asset binaire).

Vérification à faire en jeu, sans rien modifier : `stat unit` (si « Frame » est verrouillé à 33,3 ms avec « GPU » entre 17 et 33 ms, c'est la VSync ; si « GPU » est proche de 33 ms, c'est le coût réel), puis `r.VSync 0` et `t.MaxFPS 0` pour confirmer. Le chemin GPU rend la frame plus longue que 16,7 ms par les causes 1 à 4, et la VSync fait le reste.

## 9. Ce qui est innocenté ou identique entre les deux chemins

- Le chemin CPU ne tourne pas en mode GPU (§3), aucun double travail.
- Le keeper mesh n'est pas recréé chaque image : `SetMobility` a un early-out moteur (`SceneComponent.cpp:3073`), `SetGPUTerrainRenderer` retourne si le renderer est identique (`WorldScapeMeshComponent.cpp:1401`), `WS_SyncKeeperMeshRenderFlags` ne marque dirty que sur changement (`Main.cpp:1060-1062`), `SetGPUTerrainLocalBounds` recalcule des bornes constantes (`WorldScapeMeshComponent.cpp:1424-1428`, `PrimitiveComponent.cpp:4548-4555`).
- Aucun readback GPU synchrone dans le chemin quadtree : les readbacks du renderer ne servent qu'au validateur (`bValidateIndirectInstancedNoise`, off), et le `DispatchNoise` synchrone du dispatcher (attente par fence, `WorldScapeNoiseDispatcher.cpp:1782-1795`) n'a aucun appelant dans le plugin.
- Le validateur et le log de debug sont off par défaut (`WorldScapeRoot.h:1883, 1887`) ; s'ils sont actifs, ils écrasent le game thread (12 à 52 `GetGroundNoise` par tick avec les volumes, mesuré le 02/09) : à couper avant toute mesure.
- Collision, foliage, matériaux et post-process sont identiques dans les deux modes.
- Aucune configuration de prod n'active le chemin GPU (§1) : les joueurs qui sont à 30 fps avec le quadtree l'ont activé par flag ou CVar.

## 10. Comment trancher par la mesure (sans rien modifier)

Un éditeur QANGA était ouvert par Benja en fin d'analyse (PID 21856) ; il n'a pas été sollicité. Les mesures ci-dessous se font dans cet éditeur ou dans une build, sans toucher au code.

1. Même caméra, même niveau, deux runs : `ws.IndirectInstancing 1` puis `0` (CVar non mutante, `Main.cpp:188-193`), `stat unit` et `stat gpu`. Les stats GPU nommées du chemin GPU : `WorldScape IndirectTick` (`Main.cpp:39`), `WSHeightfieldGenerate` (`WSHeightfieldManager.cpp:1152`), `WSGPUTerrainMesh` (`WSGPUTerrainRenderer.cpp:2088`). Comparer aussi `PrePass`, `BasePass`, `ShadowDepths` et `VirtualShadowMaps`.
2. Activer `bDebugIndirectInstancedNoise` sur l'acteur pour la ligne de stats à 1 Hz (`WSGPUTerrainRenderer.cpp:2378-2430`) : lire `TotalVerts`, `Instances`, `DrawItems`, `HFGen`, `HFEvict`, `QLeaf`, `SSE`, `ChunkLod`. Immobile, `HFGen` doit être 0 ; en marchant, sa valeur x le coût par tuile donne la part de la cause 2.
3. Isoler la cause 1 : `ws.GPUTerrain.Quadtree.AutoThresholdFactor 5.5` (moitié de sommets environ) et `Quadtree_MaxDepth 14` (cellules de 3 m) ; lire `TotalVerts` et la cadence.
4. Isoler la cause 4 : `r.Shadow.Virtual.Enable 0` dans les deux modes ; si l'écart entre modes se resserre fortement, le VSM est le poste dominant. `r.Shadow.Virtual.ShowStats 1` donne les pages rendues et invalidées par image.
5. Isoler la cause 3 : `ProfileGPU` sur une image immobile, lire le temps des draws du keeper dans `PrePass` et `ShadowDepths` (aucune tuile n'est régénérée immobile).
6. VRAM : `stat rhi` (mémoire texture et buffer) dans les deux modes.
7. Mode intermédiaire déjà présent dans le code : `bIndirectInstancedNoiseUseClipmaps` (`WorldScapeRoot.h:1879`, `Main.cpp:1946-2120`) dessine exactement les anneaux du clipmap (index buffer troué `BuildClipmapRingIndices`, `WSGPUTerrainRenderer.cpp:182-230`) avec le pipeline GPU : c'est la comparaison « à géométrie égale » qui sépare la cause 1 de toutes les autres. État fonctionnel de ce mode non vérifié ce jour.

## 11. Pistes, non appliquées, classées par gain estimé sur effort

1. **Précision mixte dans le simplex GPU** (cellule de lattice en double, deltas et contributions en float) : gain attendu 20 à 40x sur la génération, erreur de quelques millimètres ; la collision reste CPU. C'est la piste qui rend la génération invisible dans l'image.
2. **Culling par tuile honnête** : marge d'horizon = relief réel (hauteur max de la heightmap planétaire, pas `NoiseIntensity` + `OceanHeight`) ; c'est un paramètre, pas une refonte.
3. **Régulateur sur le temps GPU réel** (timestamp de la stat `WSHeightfieldGenerate`) au lieu du temps de soumission CPU.
4. **Vertex factory** : flags `SupportsPositionOnly` + entrée position-only, et pré-calcul par instance et par image de l'offset caméra-relatif (5 loads et 60 ALU de moins par sommet et par passe).
5. **Alignement quadtree / clipmap** : `Quadtree_MaxDepth` tel que la cellule la plus fine soit `TriangleSize`, et seuil auto calibré sur le nombre de sommets et non sur l'image.
6. **Coût fixe** : n'uploader le buffer d'instances que quand le jeu de tuiles change, ne pas recalculer centres et bornes des tuiles inchangées, ne demander au heightfield manager que les tuiles nouvelles.
7. **VSM** : selon le verdict moteur du §7.2, rendre la géométrie cachable en statique (instances persistantes dans GPU Scene plutôt que `DynamicPrimitiveData` par image).

Aucune de ces pistes n'a été appliquée. Chacune touche un contrat (miroirs shader / VF / C++, régulateurs, précision) : à planifier une par une, avec mesure avant et après.

## 12. Réglage d'équivalence : quelle résolution GPU pour la densité du clipmap 128 (ajout du 2026-09-18)

Question de Benja : « quelle valeur mettre dans la résolution du quadtree GPU pour avoir la même densité que le clipmap en 128 ».

**Fait vérifié dans le code, contre-intuitif : le nombre de tuiles ne dépend pas de la résolution.** Le seuil automatique vaut `Facteur x PixelsPerUnit / (HeightfieldResolution - 1)` (`WSQuadtreeManager.cpp:573-582`) et l'erreur mesurée vaut `(TailleTuile / (HeightfieldResolution - 1)) / Distance x PixelsPerUnit` (`:843-849`). Le diviseur se simplifie des deux côtés : une tuile est divisée si et seulement si `TailleTuile > 2,75 x Distance`, quelle que soit la résolution. Changer la résolution ne change donc ni la forme de l'arbre, ni le nombre de tuiles : uniquement les sommets par tuile (au carré) et les texels par tuile.

Simulation de la règle de subdivision sur une face plane (script `scratchpad/ws_density_sim.py`, sans frustum ni horizon, donc comptage indicatif en absolu mais exact en rapport) : 120 à 144 tuiles au sol, 94 à 115 en altitude, **identique pour toutes les résolutions testées**.

| Résolution GPU | Sommets dessinés | Rapport au clipmap (155 708) | Taille de cellule contre le clipmap |
|---|---|---|---|
| 128 (test actuel) | 1 970 000 | 12,6 x | 3,2 fois plus fine |
| 64 | 534 000 | 3,4 x | 1,6 fois plus fine |
| 48 | 313 000 | 2,0 x | 1,3 fois plus fine |
| 40 | 229 000 | 1,5 x | équivalente |
| **32** | **146 000** | **0,94 x** | équivalente (1,09) |
| 24 | 83 000 | 0,53 x | 1,7 fois plus grossière |

**Réglage à poser sur l'acteur planète (trois propriétés, pas une) :**

| Propriété | Valeur | Pourquoi |
|---|---|---|
| `IndirectNoise_MeshResolution` | **32** | sommets par côté de tuile GPU ; `LodResolution` reste à 128 pour le clipmap CPU (fallback et serveur dédié) |
| `IndirectNoise_TileResolution` | **32** | sinon la résolution du heightfield reste `max(LodResolution, OceanLodResolution, MeshResolution)` = 128 (`WorldScapeRoot_Main.cpp:1687-1689`) et le coût du bruit en double précision reste entier. À 32 : 1 600 texels par tuile au lieu de 18 496, soit 11,6 fois moins, et 31 Ko au lieu de 361 Ko |
| `Quadtree_MaxDepth` | **18** | à 16, la cellule la plus fine tombe à 3,12 m contre 1,00 m au clipmap. À 18 elle vaut 0,78 m. Formule : face du cube 6 338 km / 2^profondeur / (résolution - 1) |

Effet de bord à connaître : la résolution des tuiles d'océan est plafonnée à `max(résolution terrain / 4, 8)` (`WorldScapeRoot_Main.cpp:1866-1871`). À 32 côté terre, l'océan tombe à 8 sommets par tuile. Sans effet sur L_Earth (`bOcean = False`) mais visible sur L_Dev_Claude (`bOcean = True`), à compenser par `OceanLodResolution` si besoin.

Ce réglage traite la cause 1 (densité) et l'essentiel de la cause 2 (coût du bruit par tuile). Il ne change rien aux causes 3 et 4 : le vertex shader sans variante position-only et l'absence de cache VSM sur une primitive dynamique restent. À densité et sommets égaux, le quadtree GPU restera donc plus cher que le clipmap en prépasse et en ombres. Mesurer après changement avec le plan du paragraphe 10.

## Annexe 0 : dette relevée en passant, hors périmètre, non corrigée

- Chemin CPU : `TriangleSize * (2 ^ i)` dans `GenerateBaseMesh` (`WorldScapeRoot_Main.cpp:3905, 3963`) est un XOR C++, pas une puissance ; sans effet visible car la géométrie provisoire d'`Init` est remplacée par le worker. Les canaux UV1 à UV3 sont toujours nuls mais uploadés (12 octets par sommet par régénération). `IsDynamicRelevance` (`WorldScapeMeshComponent.h:198`) est morte, `bTreatAsBackgroundForOcclusion` est inerte sur desktop.
- Chemin GPU : `OceanTempMask` (`WorldScapeNoiseEarthBiomes.ush:265`, 10 octaves) et `LattitudeRemapped` (`:264`) ne sont jamais lus ; `RiverVienMask` et `RiverVienMaskArea` évaluent deux fois la même fractale (`:278-279`) ; racine et rotation quaternion calculées deux fois dans la partie commune du heightfield (`WSHeightfieldGenerate.usf:1023-1043`).

## Changelog Discord (prêt à coller)

**🔎 WorldScape : diagnostic quadtree GPU vs clipmap CPU**
**Contexte :** à résolution égale (128), le terrain GPU est nettement plus lent que le clipmap CPU et le jeu tombe à 30 fps.
**Cause :** « 128 » n'est pas la même chose des deux côtés (128 cellules d'un anneau centré caméra contre 128 sommets par tuile du quadtree, soit 13 à 36 fois plus de sommets dessinés), le bruit des tuiles est calculé en double précision sur le GPU pendant l'image (2 à 8 ms par tuile sur GeForce), le vertex shader n'a pas de variante allégée pour les ombres et la profondeur, et le terrain GPU est une primitive dynamique que les Virtual Shadow Maps ne peuvent jamais mettre en cache (re-rasterisée à chaque image dans chaque niveau de clipmap).
**Fix :** aucun pour l'instant, diagnostic seul ; plan de mesure et pistes classées dans `Documentation/WorldScape_GPU_vs_Clipmap_Perf_Diagnostic_2026-09-12.md`.

## Annexe A : fichiers lus

`Plugins/WorldScape/Source/WorldScapeGPUTerrain/{Private,Public}/*` (renderer 2 450 lignes, heightfield manager 2 209, quadtree 1 240, VF), `Shaders/WorldScapeGPUTerrain/Private/*`, `Shaders/WorldScapeCompute/Private/*`, `WorldScapeCore/Private/WorldScapeRoot_Main.cpp` (tick, intégration, keeper), `WorldScapeRoot_Thread.cpp`, `WorldScapeMeshComponent.cpp`, `Config/DefaultEngine.ini`, `DefaultGame.ini`, `DefaultScalability.ini`, moteur `SceneComponent.cpp`, `PrimitiveComponent.cpp`, `DepthRendering.cpp`, `ShadowDepthRendering.cpp`, GPU Scene et VSM (annexe B).

## Annexe B : vérifications moteur (UE 5.7, C:/UE5_Share/Engine/Source/Runtime)

| Fait | Fichier et lignes | Extrait |
|---|---|---|
| Collecteur de primitives dynamiques par vue | `Renderer/Private/SceneRendering.h:1813` | `FGPUScenePrimitiveCollector DynamicPrimitiveCollector;` |
| IDs dynamiques remis à zéro chaque image | `Renderer/Private/GPUScene.cpp:738, 757-762, 1986-1987` | `DynamicPrimitivesOffset = Scene.GetMaxPersistentPrimitiveIndex();` |
| Slots d'instances libérés en fin d'image | `GPUScene.cpp:2398-2413` | `GPUScene.FreeInstanceSceneDataSlots(UploadData->InstanceSceneDataOffset, UploadData->TotalInstanceCount);` |
| Un collecteur et un upload par ombre VSM | `Renderer/Private/ShadowDepthRendering.cpp:1258-1265` ; `ShadowSetup.cpp:6758-6761` | `DepthPassView->DynamicPrimitiveCollector = FGPUScenePrimitiveCollector(...)` ; `UploadDynamicPrimitiveShaderDataForView(GraphBuilder, *ProjectedShadowInfo->ShadowDepthView, ...)` |
| Primitive dynamique = jamais statique pour le cache VSM | `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapPageCacheCommon.ush:51-62` | `// Dynamic primitives are not tracked, so we treat them as dynamic since they can't be cached by definition.` |
| Index persistant invalide sur un UB dynamique | `Engine/Public/PrimitiveUniformShaderParametersBuilder.h:77` | `Parameters.PersistentPrimitiveIndex = INDEX_NONE;` |
| Invalidation explicite de toutes les instances dynamiques chaque image | `GPUScene.cpp:1821-1831` | `// Enqueue cache invalidations for all dynamic primitives' instances, as they will be removed this frame` |
| Une entrée d'invalidation par identifiant de VSM | `Renderer/Private/VirtualShadowMaps/VirtualShadowMapCacheManager.cpp:618-624` | `for (int32 Index = 0; Index < NumEntries; ++Index) Instances.Add(...)` |
| Décision statique/dynamique d'un primitive persistant | `VirtualShadowMapCacheManager.cpp:553-576` | `if (Proxy->IsMeshShapeOftenMoving() && Behavior != Static) CachePrimitiveAsDynamic[...] = true;` |
| Retour au cache statique après 100 images calmes | `VirtualShadowMapCacheManager.cpp:85-89, 1830-1878` | `r.Shadow.Virtual.Cache.FramesStaticThreshold` = 100 |
| `ShouldCacheShadowAsStatic` déprécié en 5.7 | `Renderer/Public/PrimitiveSceneInfo.h:543-547` | `UE_DEPRECATED(5.7, "Unused") ... return false;` |
| Une vue VSM par niveau de clipmap | `Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.cpp:3697-3707, 4563-4567` | `return { uint32(Clipmap->GetLevelCount()), 1u };` |
| Niveaux de clipmap par défaut | `VirtualShadowMapClipmap.h:28-29` | `FirstLevel = 8; LastLevel = 18;` |
| Une commande par (instance, vue) au culling | `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapBuildPerPageDrawCommands.usf:224-227, 382-386, 404-405` | `WriteCmd(MipViewId, InstanceId, ...)` ; `InterlockedAdd(DrawIndirectArgsBufferOut[...], ThreadTotalForAllViews);` |
| Budget d'instances = instances x vues | `VirtualShadowMapArray.cpp:3998-4029` | `TotalInstances * NumPrimaryViews * NumMipLevels` |
| GDME : un appel vue principale | `Renderer/Private/SceneVisibility.cpp:4152-4158` | `Proxy->GetDynamicMeshElements(FirstViewFamily.AllViews, ...)` |
| GDME : un appel par ombre (VSM incluses) | `ShadowSetup.cpp:2842-2844, 4678-4687, 4746-4749` | `PrimitiveSceneProxy->GetDynamicMeshElements(Views, ViewFamily, 0x1, MeshCollector);` |
| Proxy dynamique GPU Scene ajouté aux sujets d'ombre | `ShadowSetup.cpp:2104-2109` | `else if (bDynamicRelevance && ...) DynamicSubjectPrimitives.Add(PrimitiveSceneInfo);` |
| Seul flag position-only en 5.7 | `RenderCore/Public/VertexFactory.h:142, 715-718` | `SupportsPositionOnly = 1u << 5` ; `SupportsPositionOnlyStream() { return !!PositionStream.Num(); }` |
| Passe d'ombre sans position-only = VS complet | `ShadowDepthRendering.cpp:599-608, 2093-2118` | `TShadowDepthVS<VertexShadowDepth_VirtualShadowMap, false>` ; `EMeshPassFeatures::Default` |
| Matériau par défaut en ombre si opaque sans WPO | `ShadowDepthRendering.cpp:538-546, 2152-2161` | `UseDefaultMaterialForShadowDepth(...)` |
| Prépasse sans position-only = `TDepthOnlyVS<false>` | `Renderer/Private/DepthRendering.cpp:153-154, 961-1001` | `bSupportPositionOnlyStream = MeshBatch.VertexFactory->SupportsPositionOnlyStream()` |
| `SetMobility` : no-op si inchangé | `Engine/Private/Components/SceneComponent.cpp:3071-3089` | `if (NewMobility != Mobility) { FComponentReregisterContext ...` |
| `UpdateBounds` : pas de dirty rendu si bornes inchangées | `Engine/Private/Components/PrimitiveComponent.cpp:4548-4555` | comparaison `Origin` / `BoxExtent` avant notification |
