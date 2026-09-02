# WorldScape : audit de la génération des meshes de clipmap

Date : 2026-09-01. Demande de Benja (auteur du plugin) : « la génération des meshes du clipmap est lente, tout est côté CPU, je suis limité à de petites résolutions ; existe-t-il une méthode plus rapide, et peut-on refaire la génération sans rien perdre ni rien casser ? »

Statut : **analyse seule, aucun fichier de code modifié.** Lecture statique du plugin (`Plugins/WorldScape`, ~58 000 lignes), de la doc, des plugins voisins (`ClipmapCore`, `PlanetScape`, `RzIndirectInstancingPlugin`) et des consommateurs QANGA. L'éditeur était fermé pendant l'audit : les valeurs d'instance des acteurs planète et le graphe du matériau n'ont **pas** pu être lus (voir §8).

Chaque affirmation ci-dessous renvoie à un `fichier:ligne`. Ce qui n'a pas pu être vérifié est marqué NON VÉRIFIÉ.

---

## 0. Verdict en bref

1. **Le composant mesh n'est pas le goulot.** `UWorldScapeMeshComponent` (fork du ProceduralMeshComponent) met à jour ses buffers RHI **en place**, avec **une seule commande render pour les 3 sections d'un LOD** (`Plugins/WorldScape/Source/WorldScapeCore/Private/WorldScapeMeshComponent.cpp:872-1048`, `:336-387`). Changer de composant (DynamicMesh, RealtimeMesh, `UProceduralMeshGPU`) ne gagnerait rien sur le temps de génération.
2. **Le goulot est algorithmique, dans le thread de génération** : à chaque franchissement d'une cellule de la grille, **l'anneau entier est recalculé** (bruit complet pour chacun de ses ~12 700 vertex, normales, copies, upload), sur **un seul thread par LOD**, sans cache ni mise à jour incrémentale, derrière une barrière tout-ou-rien. Avec les défauts C++ (résolution 128, 8 niveaux), une régénération complète évalue **112 072** fois le bruit ; une marche à pied déclenche plusieurs régénérations de LOD 0 par seconde.
3. **Deux niveaux de réponse, cumulables** :
   - **Rendre le chemin CPU incrémental** (cache de bruit indexé sur la grille entière, parallélisme sur les manques, normales en O(N)) : gain estimé de x15 à x100 sur le coût par déplacement, **contrat 100 % inchangé** (même composant, mêmes sections, mêmes couleurs de vertex, même collision, même API bruit). Estimation : 1 à 2 semaines.
   - **Terminer le renderer GPU déjà présent dans le plugin** (`WorldScapeGPUTerrain` + `WorldScapeCompute`, ~6 000 lignes, janvier-février 2026, jamais activé) : c'est la « méthode moderne » (heightfield calculé en compute, vertex buffer résident GPU, draws indirects). Il manque les volumes (trous, heightmap, bruit), les LOD de matériau, le ray tracing, les champs de distance, un teardown propre. Estimation : 4 à 8 semaines.
4. **Ce qu'il ne faut pas faire** : brancher `UProceduralMeshGPU` (plugin `ClipmapCore`, compilé aujourd'hui) à la place du composant actuel. Il garde tout le calcul sur le CPU, encode les positions en **float16** (précision inutilisable au-delà de 655 m de l'origine) et impose un matériau de déplacement incompatible avec `M_EarthBase` (§3.2).

---

## 1. Ce que fait le chemin CPU aujourd'hui (le cahier des charges)

Toute réécriture doit reproduire chacun de ces comportements. Références sous `Plugins/WorldScape/Source/WorldScapeCore/`.

### Topologie
1. Un niveau de LOD = un `UWorldScapeLod` (USceneComponent) portant un `UWorldScapeMeshComponent` à **3 sections** : 0 = corps, 1 = patch A (bande verticale de 2 colonnes), 2 = patch B (bande horizontale de 2 lignes). `Public/WorldScapeLod.h:32-46`, `Private/WorldScapeLod.cpp:93, 207, 240`.
2. LOD 0 = grille pleine de `(R-2)^2` vertex ; LOD > 0 = **anneau** en 4 rectangles (bas, gauche, droite, haut) qui se recouvrent d'une ligne, chaque bande ayant son propre bloc de vertex (vertex de jointure dupliqués). `WorldScapeLod.cpp:60-171`.
3. Les patchs A/B ne sont créés que pour les LOD de rendu (`bCollision == false`) : ils ferment le L laissé quand un anneau se recentre d'une demi-cellule du parent. `WorldScapeLod.cpp:175-241`, placement selon `SubPosition` `Private/WorldScapeRoot_Thread.cpp:1396-1436`.
4. La topologie (index buffers `Triangles`, `TrianglesPatchA`, `TrianglesPatchB`) est construite **une fois** dans `Init` ; ensuite seules les données de vertex changent, **à nombre de vertex constant** (une mise à jour de taille différente est refusée et loguée). `WorldScapeMeshComponent.cpp:904-913`.
5. Pas de l'anneau i = `TriangleSize * 2^i * AltitudeMultiplier`, avec `AltitudeMultiplier = 2^round(max(0, log2(distanceAuSol / HeightAnchor)))`, plafonné à 128 rayons planétaires. `Private/WorldScapeRoot_Main.cpp:3778, 3901-3902, 3931`.
6. Recentrage : la position projetée du joueur est snappée (ceil) sur la grille du LOD **et** sur celle du parent ; la différence donne `SubPosition` dans {0,1}^2. Régénération quand le déplacement dépasse **la moitié d'un pas**, sur `ForceUpdate` (changement d'`AltitudeMultiplier`), ou quand la normale snappée change (`GridAngle`). `Private/WorldScapeWorldType.cpp:214-313`, `WorldScapeRoot_Main.cpp:3947`.
7. Raccord entre niveaux par **couture** : les vertex de la bordure extérieure sont ramenés sur la ligne du voisin pair selon leur parité (les paires fusionnées donnent des triangles d'aire nulle, qui restent dans le buffer). Pas de jupe, pas de bande de raccord, pas de morphing. `Public/WorldScapeRoot.h:105-138, 178-211`.
8. Cas polaire : repère tangent aligné sur les axes et inversion du sens de parcours quand `|Normal.Z| > 0.9`. `WorldScapeRoot_Thread.cpp:1287-1295`, `WorldScapeWorldType.cpp:261-277`.
9. Projection : grille posée sur le plan tangent local (double précision `DVector`), normalisée puis multipliée par `PlanetScaleCode`, moins l'origine du chunk ; ré-ancrage double vers float par `OffSetHelper`. Ce n'est pas un cube-sphère. `WorldScapeRoot.h:230-234`, `WorldScapeRoot_Thread.cpp:1454-1469`.
10. Monde plat : déplacement selon +Z, repère identité, bornes `FlatWorldLimit`. `WorldScapeRoot_Thread.cpp:734-737, 1281-1285`, `WorldScapeRoot.h:218-224`.
11. Océan : un second jeu de LOD (`WorldScapeLodOcean`) avec sa résolution, sa taille de triangle, son `OceanMaxLod`, son tableau de matériaux, sans ombres, normale radiale pure, bruit océan borné par `OceanHeight`. `WorldScapeRoot_Main.cpp:3127-3181`, `WorldScapeLod.cpp:35-40`, `Private/WorldScapeRoot_Noise.cpp:347-351`.

### Contrat par vertex (à préserver au bit près)
12. **Couleur de vertex (rendu) = (HeightNormalize, Temperature, Humidity, Hole ? 1 : 0)**. `WorldScapeRoot_Thread.cpp:745, 771, 796`. Le matériau maître en dérive les biomes et **le masque d'opacité** (les trous sont un flag, jamais une suppression de géométrie). `WaterMask` et `FoliageMask` ne sont **pas** écrits dans le mesh de rendu.
13. Couleur de vertex (collision) = (HeightNormalize, Temperature, Humidity, **FoliageMask**) : asymétrie volontaire ou non, elle existe. `Private/WorldScapeRoot_Collision.cpp:406`.
14. UV0 = `((x + SubPosition.X) / R, (y + SubPosition.Y) / R)` ; **UV1, UV2, UV3 valent toujours (0,0)** et occupent quand même 24 octets par vertex. `WorldScapeRoot.h:239`, `WorldScapeMeshComponent.cpp:792-794, 918-920`.
15. Normales : jamais analytiques. Si `bGenerateTangents` : lissage complet avec recherche de vertex coïncidents (`FindVertOverlaps`, balayage de **tous** les vertex par coin de triangle) ; sinon somme des produits vectoriels des faces puis normalisation, tangente constante. Pas de couronne de bordure : les normales aux limites de chunk sont fausses par construction. `WorldScapeRoot_Thread.cpp:1019-1246`, `Plugins/WorldScape/Source/WorldScapeCommon/Private/WorldScapeHelper.cpp:93-110`.

### Sources de hauteur
16. Ordre d'évaluation dans `GetNoise` : bordure du monde plat ; heightmap planétaire (UV cube, remap min/max) ; classe de bruit (`GetNoise` ou `GetOceanNoise`) ; blend planétaire (Replace / Addition / Subtract, alphas par canal) ; **early-out si aucun volume ne touche le chunk** ; volumes de bruit ; volumes de heightmap (blend ou falloff, override ou addition, `WaterMask`) ; volumes de trou. `WorldScapeRoot_Noise.cpp:219-635`.
17. Culling des volumes par boîte du chunk (`GetFBox`, `LodSize` publié par `SetData(..., -1)`). `WorldScapeRoot_Thread.cpp:705, 1446`.
18. Aucun détail dépendant du LOD : chaque vertex de chaque niveau paie la pile d'octaves complète ; aucun cache. `Plugins/WorldScape/Source/WorldScapeNoise/Public/WorldScapeNoiseClass.h:39-46`.

### Ordonnancement, upload, collision, matériaux
19. Un `FAsyncTask<LodGenerationThread>` par LOD à régénérer, lancés dans le même tick ; pipeline `SetPointPosition` puis `CalculateNoise` puis `CalculateNormal` x3 puis `SetData` x3 puis `SetMeshLod`. **Aucun `ParallelFor`** dans ce chemin (une version parallèle de `ProcessVertices` est en commentaire, `WorldScapeRoot.h:245-327`). `WorldScapeRoot_Thread.cpp:1439-1518`, `WorldScapeRoot_Main.cpp:3950-3954`.
20. **Barrière** : le game thread attend que **tous** les LOD lancés soient finis, puis uploade tout dans la même frame et réapplique les matériaux ; tant qu'une génération est en cours, aucun nouveau travail n'est lancé. `WorldScapeRoot_Main.cpp:3846, 3876, 3997-4075`.
21. Upload : `UpdateMeshSections_LinearColor` écrit `PlanetVertexBuffer`, construit un tableau parallèle `FDynamicMeshVertex`, puis une commande render fait 4 `LockBuffer`/`Memcpy` par section ; si le ray tracing est actif, le **BLAS est reconstruit à chaque mise à jour**. `WorldScapeMeshComponent.cpp:336-412, 1037-1043`.
22. Position pilote : vue de preview, sinon **pawn du joueur 0 uniquement**, sinon caméra éditeur ; `bOverridePlayerPosition` possible. `WorldScapeRoot_Main.cpp:3719-3808`.
23. Serveur dédié : le worker de clipmap n'est jamais lancé ; mais `GenerateBaseMesh` crée quand même les sections placeholder non déplacées, et **les LOD de collision sont générés pour chaque PlayerController et chaque `CollisionDependantActor`** (bloc 3x3 par acteur, génération **synchrone sur le game thread** dans le constructeur de `ColisionGeneration`, cuisson async). `WorldScapeRoot_Main.cpp:3876, 3105`, `WorldScapeRoot_Collision.cpp:117-145, 274-332, 337-368`.
24. Collision : composants séparés (`CollisionLods`), `CollisionResolution (+3 si bPaddedCollision)`, `CollisionTriangleSize`, `bUseComplexAsSimpleCollision`, `bFlipNormals/bDeformableMesh/bFastCook`, clé de requête pour éviter les recalculs, `OnCollisionLodsUpdate`. `WorldScapeRoot_Collision.cpp:57-88, 217-237`, `WorldScapeMeshComponent.cpp:1499-1501, 1543-1598`.
25. LOD de matériau : `SetMeshLod(Lod + AltitudeLod)` puis premier `FInt32Range` contenant la valeur, sinon `DefaultMaterial`, appliqué aux 3 sections après chaque lot. `WorldScapeRoot_Thread.cpp:1502-1506`, `Private/WorldScapeRoot_Helper.cpp:45-131`.
26. Rendu : relevance dynamique forcée (`IsDynamicRelevance` est mort), `bVelocityRelevance = false` avec `PreviousLocalToWorld` forcé (correctif 5.7), éléments statiques uniquement pour la sortie RVT, ray tracing via `r.RayTracing.Geometry.WorldScapeMesh`, pas de Nanite ni de champ de distance. `WorldScapeMeshComponent.cpp:155-207, 487-494, 550-552, 586-652`.
27. Éditeur : tick en viewport, caméra éditeur comme pilote, `bGenerateCollisionInEditor`, `bStaticCollisionInEditor`, `bProjectPosition`. `WorldScapeRoot_Main.cpp:573, 2944-2947, 3736-3796`.
28. Surface publique lisant le mesh généré : **une seule** fonction, `UWorldScapeLod::GetClosestSurfaceNormal` (balayage linéaire des triangles, `WorldScapeLod.cpp:367-384`), plus les tableaux `BlueprintReadWrite` de `UWorldScapeLod` (`Vertices`, `Triangles`, `Normals`, `VertexColors`, `UV`...). **Tout le reste** (`GetGroundNoise`, `GetPawnDistanceFromGround`, `GetPawnSnappedNormal`, `WS_ProjectGroundLocation`, `ProjectLocationToGround`, gravité...) recalcule depuis le bruit et ne dépend pas du mesh. `WorldScapeRoot_Helper.cpp:135-403`, `Private/WS_FunctionLibrary.cpp:284-372`, `Private/WorldscapeSubsystem.cpp:235-257`.

### Réglages (défauts C++, `WorldScapeRoot_Main.cpp:522-570`)
`MaxLod = 8` (8 niveaux : 1 grille + 7 anneaux), `LodResolution = 128` (arrondi au multiple de 4), `TriangleSize = 100`, `OceanMaxLod = 8`, `OceanLodResolution = 32`, `OceanTriangleSize = 400`, `HeightAnchor = 10000`, `CollisionResolution = 16`, `CollisionTriangleSize = 200`, `bGenerateCollision = true`, `bPaddedCollision = true`, `TickPerSecond = 60`. `bGenerateTangents`, `bGenerateCollisionInEditor`, `bGenerateCollisionForAllPlayer` : booléens **non initialisés** dans le constructeur (donc false par le zero-init UObject, sauf valeur posée dans le niveau). Aucune section `[/Script/WorldScapeCore...]` dans `Config/DefaultGame.ini` ni `DefaultEngine.ini` : **les valeurs réelles vivent dans les niveaux de planète**. Un locateur binaire (chaînes dans les `.umap`, pas un audit) situe les acteurs `AWorldScapeRoot` dans des sous-niveaux streamés par QLevel : `Content/_QLevel/Universe/Planets/<Corps>/L_<Corps>.umap` (plus une variante `_Data/L_<Corps>_LOptimised.umap`), enveloppés par un `UQLevel_Asset` `Q_L_<Corps>`, pour 20 corps (Earth, Moon, Mars, Phobos, Deimos, Venus, Mercury, Europe, Ganymede, Callisto, IO, Titan, Encelade, Minas, Tethys, Triton, Miranda, Oberon, Titania, Setebos). Aucune sous-classe Blueprint d'`AWorldScapeRoot` n'existe : ce sont des instances C++. Les valeurs de propriétés restent NON VÉRIFIÉES (éditeur fermé). Le cook seed référence `SG_WorldScape_Resolution` et `SG_WorldScape_Update` dans `/Game/Systems/Scalability_Sys/QangaScalability/WorldScape/` : la résolution est probablement pilotée par la scalabilité QANGA au runtime (NON VÉRIFIÉ).

Assets référencés par les niveaux (chaînes, donc « le niveau référence l'asset », sans dire quel slot) : Terre = `Mi_EarthMat2` (instance de `M_EarthBase`), bruit `NoiseWorldscape/PlanetEarth`, océan `OceanPak/MI_Ocean`, heightmap `HMV_Everest` ; Lune = `Material_Planet/Moon_Inst`, bruit `TheMoon` ; Mars = `Mi_MarsMat1` ; Europe = `Mi_Europe2` et Titan = `Mi_IO`, tous deux avec `VenusNoise`. Une cellule vide dans cette liste n'est pas une absence (la valeur peut être dans le sous-niveau `_LOptimised` ou dans un acteur externe).

---

## 2. Où passe le temps (analyse structurelle, non mesurée)

Comptage par niveau, avec `R = LodResolution` :

| Élément | Formule | R = 128 | R = 32 (océan) |
|---|---|---|---|
| LOD 0 (corps + patchs) | `(R-2)^2 + 4R - 6` | 16 382 | 1 022 |
| Anneau (corps + patchs) | `(R/2)(R-2) + R^2/4 + 4R - 6` | 12 666 | 858 |
| Total 8 niveaux | | 105 044 | 7 028 |
| **Régénération complète** | | **112 072 évaluations de bruit**, ~210 000 triangles | |

Classement des coûts suspectés (à confirmer par une trace Insights, §5 Phase 0) :

1. **Bruit par vertex** (`CalculateNoise`, `WorldScapeRoot_Thread.cpp:722-798`) : pile d'octaves complète à chaque vertex, aucune réduction avec la distance, aucun cache, plus la boucle de volumes. **Et surtout : à chaque franchissement d'une demi-cellule, l'anneau entier est recalculé.** Or dans un clipmap, après un déplacement d'une cellule, seule une bande en L de nouvelles cellules est réellement nouvelle. Pour LOD 0 : 126 nouveaux points contre 16 382 recalculés (patchs A/B : jusqu'à 506 de plus s'ils ne partagent pas la grille). Pour un anneau : environ 190 nouveaux points contre 12 666 recalculés. Le travail utile est entre 1 % et 6 % du travail fait. À 6 m/s avec des cellules de 1 m, LOD 0 se régénère 6 fois par seconde, LOD 1 trois fois, etc. : de l'ordre de **200 000 évaluations de bruit par seconde** en marchant, dix fois plus en véhicule. C'est là que naît la sensation de « génération super longue » : les anneaux fins n'arrivent plus à suivre.
2. **Normales avec `bGenerateTangents = true`** : `FindVertOverlaps` balaie tous les vertex pour chacun des 3 coins de chaque triangle : pour LOD 0, environ 3 x 31 000 x 15 876 = **1,5 milliard** de comparaisons `FVector::Equals` par régénération. Si cette option est active sur les planètes QANGA (NON VÉRIFIÉ), elle domine tout le reste à elle seule. `WorldScapeRoot_Thread.cpp:1149-1183`.
3. **Un seul thread par LOD** : le parallélisme effectif est le nombre de LOD en cours de régénération (souvent 1 ou 2 en régime établi). `WorldScapeRoot_Main.cpp:3950`.
4. **Copies** : `SetData(LodData data, ...)` reçoit ses 5 tableaux **par valeur**, 3 fois par LOD (`WorldScapeLod.h:134`, `WorldScapeRoot_Thread.cpp:1497-1499`) ; `SetData` reconstruit un `FVerticeList` élément par élément (`WorldScapeLod.cpp:291-298`) ; `CalculateNormal` copie tout l'index buffer (`:1204-1206`) ; l'upload alloue `PreconvertedVertices` en plus de `PlanetVertexBuffer`. Ordre de 25 à 30 Mo touchés par régénération complète.
5. **Barrière et rafale** : tout attend le plus lent, puis tout est uploadé dans une seule frame (`WorldScapeRoot_Main.cpp:4049-4056`) ; la boucle de `TransformKeeper->UpdateBounds()` et la réassignation des matériaux suivent.
6. **`GenerateBaseMesh`** : 4 `MarkRenderStateDirty()` par composant (clear + 3 sections), soit 64 reconstructions de scene proxy par régénération complète. `WorldScapeMeshComponent.cpp:749-854`.
7. **Collision** : synchrone sur le game thread, 9 chunks par acteur invoquant, 4 `UpdateCollision()` par chunk (`SetMesh` recrée les 3 sections). Sur le serveur dédié, cela se multiplie par le nombre de joueurs (§7).
8. **BLAS ray tracing** reconstruit à chaque mise à jour de section quand le RT est actif.

**Le fork PMC lui-même** : il fait mieux que le PMC moteur (mise à jour multi-sections en une commande, 4 canaux UV, sortie RVT, ray tracing, contrôle de l'invalidation VSM, correctif des vecteurs de vitesse 5.7). Son poids mort : UV1-3 toujours nuls (17 % du vertex de 144 octets), `IsDynamicRelevance` inopérant, reconstruction BLAS par mise à jour. Rien de cela n'explique une génération lente.

---

## 3. Inventaire des alternatives déjà présentes dans le projet

### 3.1 `WorldScapeGPUTerrain` + `WorldScapeCompute` (dans le plugin, jamais activés)

Chemin GPU complet derrière `bUseGPUNoise && bUseIndirectInstancedNoise && !NM_DedicatedServer` (`WorldScapeRoot_Main.cpp:1079, 2961`). Shaders datés janvier-février 2026, sources recopiées le 23 juin 2026, jamais retouchées depuis, alors que `WorldScapeCore` a bougé jusqu'à aujourd'hui. Flags tous `false` en C++ (`WorldScapeRoot.h:1866-1878`), rien dans les `.ini`. Binaires compilés et présents (`Binaries/Win64/UnrealEditor-WorldScapeGPUTerrain.dll`, 18 août).

Architecture : tuiles de heightfield en compute (`Shaders/WorldScapeGPUTerrain/Private/WSHeightfieldGenerate.usf:815-816` écrit hauteur + `(HeightNormalize, Temperature, Humidity, Hole)`), cache LRU à budget mémoire (`WSHeightfieldManager.cpp:1478-1545`), quadtree 6 faces à erreur écran avec culling d'horizon et équilibrage 2:1 (`WSQuadtreeManager.cpp:871-1010`), **ou** un mode clipmap qui reproduit les anneaux CPU (`bIndirectInstancedNoiseUseClipmaps`, `WorldScapeRoot_Main.cpp:1349-1509`), vertex buffer GPU persistant écrit par `WSMeshGenerate.usf` (normales par différences centrales), vertex factory custom avec **origine relative caméra en double découpée high/low** (`WSGPUTerrainVertexFactory.ush:236-253`), un draw indirect par chunk. Le composant primitif reste `UWorldScapeMeshComponent` (le `TransformKeeper`), dont le proxy est remplacé (`WorldScapeMeshComponent.cpp:1311-1315`) : l'identité de classe est conservée. Deux harnais de validation numérique contre le CPU existent (`bValidateGPUNoise`, `bValidateIndirectInstancedNoise`, tolérances hauteur 1.0 / 8.0, `WorldScapeRoot_Thread.cpp:352-353`, `WSGPUTerrainRenderer.cpp:935-936`).

État vis-à-vis du chemin CPU :

| Fonction | État | Preuve |
|---|---|---|
| Collision | conservée, reste CPU | `WorldScapeRoot_Main.cpp:3856-3893` |
| Volumes de trou / heightmap / bruit | **MANQUANT** (uniformes câblés à 0) | `WSHeightfieldManager.cpp:1196-1198` |
| Heightmap planétaire | fait | `WSHeightfieldManager.cpp:1345-1460` |
| Océan | fait (slot matériau 1) | `WSGPUTerrainRenderer.cpp:1439-1443, 1560` |
| LOD de matériau (`MaterialsLod`) | **MANQUANT** (seul `DefaultMaterial`) | `WorldScapeRoot_Main.cpp:1888, 3014` |
| Couleur de vertex RGBA | identique au CPU | `WSMeshGenerate.usf:245-246` |
| UV0 | fait ; UV1-3 aliasés sur UV0 | `WSGPUTerrainVertexFactory.cpp:116-119` |
| Raccords entre LOD | partiel (masque de stitch, pas de morph) | `WSMeshGenerate.usf:209-232` |
| Inversion de parcours (hémisphère opposé) | **régression** (`bInvertOrder` forcé à 0) | `WSGPUTerrainRenderer.cpp:1744` |
| Bordure du monde plat | manquant | aucun `FlatWorldLimit` côté GPU |
| Ray tracing, champs de distance, cartes Lumen, PSO precaching, MDC cachés | **manquants** | `WSGPUTerrainVertexFactory.cpp:132-135` |
| Classes de bruit | 2 sur 18 (`QangaNoise`, `EarthNoise` ; `QangaV2Noise` non couvert) | `WorldScapeNoiseGPUTypes.h:311-315`, `WorldScapeRoot_Main.cpp:428-440` |
| Foliage GPU (`bUseGPUNoiseForFoliage`) | flag mort | `WorldScapeRoot_Main.cpp:2156` seul lecteur |
| Teardown | fragile : rien dans `EndPlay`/`BeginDestroy`, fence bloquante dans `Shutdown`, deux caches `static` jamais purgés | `WorldScapeRoot_Main.cpp:605-699`, `WSGPUTerrainRenderer.cpp:723-726`, `WSHeightfieldManager.cpp:106`, `WorldScapeNoiseDispatcher.cpp:122` |

Verdict : un renderer alternatif cohérent, mené jusqu'à « ça rend et les nombres collent », puis mis en pause. Le mode `bUseGPUNoise` **seul** (sans indirect) est une variante hybride qui garde toute la topologie CPU et ne remplace que `CalculateNoise` par un dispatch + readback (latence d'une frame minimum, volumes appliqués côté CPU ensuite, 2 types de bruit) : `WorldScapeRoot_Thread.cpp:801-1017`.

### 3.2 `Plugins/ClipmapCore` (UEGPUClipmaps de Jesse Watson, modifié)

Copié dans QANGA et rendu compilable sur 5.7 aujourd'hui (session « ProceduralMeshGPU deprecation warnings »). Personne ne le référence (`Source/`, `WorldScape/Source`, `.uproject` : zéro occurrence). Deux choses dedans :

- `AWorldScapeCore` (nom homonyme, sans lien avec le plugin WorldScape) : clipmap **plat** de heightmap statique (`UTerrainAsset`), technique de Mike Savage : 5 mesh statiques (croix, tuile, remplissage, bordure, couture) instanciés en ISM, texture fenêtre `Texture2DArray` remplie par le CPU depuis la heightmap, déplacement dans le matériau. `Plugins/ClipmapCore/Source/ClipmapCore/Private/WorldScapeCore.cpp:419-603`. Pas de bruit procédural, pas de sphère, pas de double précision.
- `UProceduralMeshGPU` : composant « compatible API PMC » qui **garde tout le calcul CPU** et encode positions, normales, couleurs et UV dans 4 textures (`EncodeVerticesToTextures`, boucles par texel + `UpdateResource()` à chaque mise à jour), rendues sur une grille fixe (`BaseGridResolution`, 8 à 512) par un matériau de déplacement obligatoire (`BaseMaterial` lisant `VertexPositions`...). `Private/ProceduralMeshGPU.cpp:280-497`, `Public/ProceduralMeshGPU.h:265-274`. **Positions en `PF_FloatRGBA` (float16)** : au-delà de 65 504 unités (655 m) la valeur déborde, et à 100 m la résolution est déjà de 6 cm. Inutilisable pour des anneaux de plusieurs kilomètres. Il ne réduit aucun coût de génération et casse le contrat du matériau.

Intérêt réel pour WorldScape : aucun en l'état ; la technique ISM + fenêtre de texture est déjà reproduite, en double précision, par le mode clipmap du module GPU de WorldScape.

### 3.3 `Plugins/PlanetScape` (plugin de Benja, non activé, aucun consommateur)

Planète GPU complète : cube-sphère + quadtree à erreur écran, heightfield dans un `Texture2DArray` écrit par `HeightFieldCS`, un ISM par face avec déplacement en WPO, collision heightfield par tuile, précision LWC manuelle (`Documentation/PlanetScape_ARCHITECTURE.md` §2). C'est la référence d'architecture pour « comment on fait un terrain GPU », mais ce n'est pas une réponse à la question posée : bruit différent (`WorldscapeTerrain.ush` ne porte que `PlanetTerraNoise`), pas de volumes, pas d'API bruit CPU utilisable sur serveur dédié (`SampleSurfaceRadiusCM` lit un cache peuplé par readback GPU, absent sur un serveur headless, §5 de sa doc), pas d'identité `AWorldScapeRoot`/`UWorldScapeMeshComponent`, pas de grille QLevel ni de snap. Migrer QANGA dessus serait un chantier de plusieurs mois.

### 3.4 Autres
- `RzIndirectInstancingPlugin` : banc d'essai (vertex factory indirect, Nanite runtime, `RzPlanet` quadtree). Non intégré, aucun consommateur (`Documentation/RzIndirectInstancingPlugin_ARCHITECTURE.md`).
- `SphereScape` (projet RTSTemplate, 30 août) : anneaux sphériques à LOD écran, expérimentation séparée.
- Moteur : `UDynamicMeshComponent` (GeometryFramework) est un transport plus lourd, sans gain de génération ; `VirtualHeightfieldMesh` (`Engine/Plugins/Experimental`) rend un heightfield depuis une RVT, pensé pour le Landscape plat, pas pour une sphère procédurale de 1 500 km.

---

## 4. Options

| Option | Ce qui change | Gain attendu | Risque contrat | Effort (estimation) | Reste CPU |
|---|---|---|---|---|---|
| A. Chemin CPU incrémental (§5 Phase 1) | Thread de génération uniquement | x15 à x100 sur le coût par déplacement ; x(nb cœurs) sur les régénérations complètes ; résolution 256 atteignable | Nul : mêmes composants, sections, couleurs, UV, collision, API | 5 à 10 jours + 2 à 3 jours de validation | Tout, mais peu |
| B. `bUseGPUNoise` seul (existant) | Le bruit part sur GPU avec readback | Le bruit disparaît du CPU ; latence d'une frame ; 2 types de bruit | Faible (topologie et contrat inchangés) ; volumes encore CPU | Test 1 jour ; durcissement 1 semaine | Normales, copies, upload, collision |
| C. Finir le renderer GPU (§5 Phase 2) | Rendu résident GPU ; le CPU ne fait plus que collision et requêtes | Résolution bornée par le GPU seulement | Moyen : matériau (UV1-3), volumes à porter, RT/DF/Lumen, teardown | 4 à 8 semaines | Collision (inchangée), foliage, requêtes |
| D. Changer de composant mesh (`UDynamicMeshComponent`, `UProceduralMeshGPU`, RealtimeMesh) | Transport | **Aucun** sur la génération | Élevé (identité de classe utilisée par QAI, matériau) | | Tout |

A et C sont cumulables : A rend le jeu confortable tout de suite et reste le chemin serveur/collision ; C est le long terme pour les très hautes résolutions. B est un test gratuit à faire pendant la Phase 0.

---

## 5. Recommandation et plan

### Phase 0 : mesurer (0,5 à 1 jour, éditeur ouvert)
1. Lire sur l'acteur planète placé (et via la scalabilité QANGA) : `LodResolution`, `MaxLod`, `TriangleSize`, `HeightAnchor`, `bGenerateTangents`, `bGenerateCollision`, `CollisionResolution`, classe de bruit, volumes. Si `bGenerateTangents` est vrai, c'est le premier levier (§2 point 2).
2. Trace Unreal Insights sur `L_Dev_Claude` (marche 30 s, véhicule 30 s) : les scopes existent déjà (`WS_Mesh_DoWork`, `WS_Mesh_SetPointPosition`, `WS_Mesh_CalculateNoise`, `WS_Mesh_CalculateNormal`, `STAT_WorldScapeMesh_UpdateSectionGT`). Compter les régénérations par LOD et par seconde, le temps par scope, la durée entre demande et upload.
3. Essayer `ws.GPUNoise 1` (option B) sur la même trajectoire, avec `bValidateGPUNoise` pour vérifier la parité.

### Phase 1 : chemin CPU incrémental (recommandé)
Réécriture de `LodGenerationThread` (`WorldScapeRoot_Thread.cpp`) à contrat constant :
1. **Cache de bruit par LOD indexé sur la grille entière.** Clé = coordonnées entières du point sur la grille snappée du LOD (origine snappée / pas, plus l'index local) ; valeur = `FNoiseData` complet (hauteur, hauteur normalisée, température, humidité, trou, masques). Tore `R x R` par LOD (0,8 Mo à R = 128), possédé par le LOD, lu et écrit uniquement par sa tâche (la barrière garantit l'exclusion avec le game thread). Les vertex cousus échantillonnent la position d'un voisin pair, donc un point de grille : la clé reste entière. Invalidation totale sur : régénération complète, changement d'`AltitudeMultiplier` (pas différent), changement de normale snappée (repère différent), `HMIForceUpdate` (volumes rafraîchis). Résultat attendu : par déplacement, seuls les points de la bande entrante sont évalués.
2. **`ParallelFor` sur les manques** (régénérations complètes, téléportations). La sûreté multi-thread de `GetGroundNoise` est déjà établie par `GetGroundNoise_Batch` (`WorldScapeRoot_Noise.cpp:128`).
3. **Normales en O(N) par adjacence de grille**, reproduisant les deux variantes actuelles (somme des faces incidentes ; et, si `bGenerateTangents`, le lissage à travers les jointures de bandes que `FindVertOverlaps` obtenait par force brute, en résolvant les vertex dupliqués par leur clé de grille). Écart numérique attendu : ordre de grandeur de l'ordre d'accumulation float, invisible.
4. **Suppression des copies** : `SetData` par référence ou déplacement, index buffer non copié, `FVerticeList` supprimé.
5. **Barrière conservée** (elle évite qu'un niveau s'affiche avant son voisin cousu), upload et composant inchangés.
6. En passant, corriger la course de données sur `WorldScapeLodInGeneration` (TMap de `bool` écrit par le worker et lu par le game thread, `WorldScapeRoot_Thread.cpp:1508-1511` vs `WorldScapeRoot_Main.cpp:4013-4020`) par un compteur atomique, et supprimer `DoWorkMain` (mort, `delete this`).
7. Un CVar `ws.Mesh.NoiseCache 0/1` pour comparer à chaud.

Preuve de non-régression (avant de dire « c'est fait ») :
- **Buffers de référence** : un CVar de dump écrit, pour chaque LOD, positions / normales / couleurs / UV à N points d'une trajectoire scriptée (QATS) sur `L_Dev_Claude`, avec l'ancien code puis le nouveau ; comparaison par script (tolérance 1e-4 sur positions, 1e-3 sur normales, exact sur couleurs et UV).
- Trace Insights avant/après sur la même trajectoire, chiffres dans la doc.
- Vérification visuelle des coutures (aucun trou entre niveaux), des trous de volume, des heightmaps (`HMV_Everest`), de l'océan, de la collision (marche, véhicule), en PIE puis en build.
- Les trois rôles réseau : la collision serveur passe par `ColisionGeneration`, non touché en Phase 1.

### Phase 2 (optionnelle) : terminer le renderer GPU
Dans l'ordre des dépendances : (1) volumes de trou, heightmap et bruit dans `WSHeightfieldGenerate.usf` (les uniformes existent, câblés à 0) ; (2) vérifier que `M_EarthBase` et le matériau océan ne lisent que UV0, sinon étendre `FWSTerrainVertex` ; (3) `MaterialsLod` ; (4) `bInvertOrder` et bordure du monde plat ; (5) flags renderer (PSO precaching, MDC cachés, ray tracing, cartes Lumen, champs de distance) ; (6) teardown déterministe (`EndPlay` reset `IndirectNoiseState`, `ClearGPUTerrainRenderer`, purge des caches `static`) ; (7) `QangaV2Noise` et les autres bruits si un monde les utilise. Validation par `bValidateIndirectInstancedNoise` et par les mêmes buffers de référence (couleurs et UV0 doivent être identiques au CPU).

---

## 6. Contrats à préserver (relevés chez les consommateurs QANGA)

1. **Couleur de vertex R/G/B/A = hauteur normalisée / température / humidité / trou**, masque d'opacité du matériau maître compris. Changer le packing casse toutes les planètes en silence.
2. **Identité de classe utilisée comme prédicat** : `QAI_FlowFieldManager`, `QAI_FloatingPawnMovement`, `QAI_ImpostorSubsystem` testent `IsA<UWorldScapeMeshComponent>()`, `IsA<AWorldScapeRoot>()`, `IsA<UWorldScape_FCP_CollisionComponent>()`. Un nouveau composant ferait passer le terrain pour de la géométrie posée à la main (faux plafonds, faux blocages de marche).
3. **Canaux de collision** `ECC_GameTraceChannel6` (`Worldscape`) et `ECC_GameTraceChannel8` (`WS_SmallObject`) : 11 profils et 5 `EditProfiles` dans `Config/DefaultEngine.ini`, `QModule_StickyGrenadeActor` les bloque explicitement avec un test d'automation ; redirection `WorldscapeGround` vers `Worldscape` à garder.
4. **`CollisionDependantActor` / `UWorldscapeCollisionInvoker`** : QAI retire l'invoker des impostors par réflexion sur le nom `RemoveWolrdScapeCollisionInvoker` (faute figée) et par le tag `WorldScapeNoCollision` ; QATS l'exige sur l'hovercraft neuronal.
5. **L'API bruit doit rester indépendante du mesh** : `GetGroundNoise(_Batch)`, `GetNoise`, `GetPawnDistanceFromGround`, `GetDistanceFromWater`, `GetVolumesData`, `WorldToECEF`, `WorldScapeWorld::GetUpVector`, `WS_GetAltitudeInLocation`, `WS_SingleProject`, `ProjectLocationToGround`, `WS_GetActualGravityFromLocation`. C'est la seule connaissance du terrain sur le serveur dédié et la seule chose sûre dans un `ParallelFor` (QAI mouvement et spawn, QPolice, DQS, QSystem natation et succès, relief de la minimap). Côté Blueprint, un locateur binaire (non audité, éditeur fermé) trouve aussi `MapView/MiniMap`, `StarMap_Component` et `Get_WS_Conf`, les contrats de ravitaillement, le spawner Sanglantines et les volumes de vent et de vagues : tous lisent la configuration de la planète ou la hauteur du sol, jamais le mesh.
6. **Gravité** : QAI vaisseaux et trafic aérien prennent le « haut » radial sur `AWorldScapeRoot` avant GravityScape ; `GenerationType == Flat` change de branche.
7. **Snapshot de foliage** (`GatherVisibleCollidableFoliageInstances`...) consommé par QAI sur le game thread.
8. **Grille QLevel** (`FWS_Grid`, codec legacy inclus) et **snap spawner** (`Snap_ComputeCollection_ForActor`).
9. **Identité de planète par chemin d'asset de bruit** (`QSystem_AchievementSubsystem` compare à `/Game/Resources/NoiseWorldscape/PlanetEarth` et `/TheMoon`).
10. **Noms de modules** : `QSystem`, `DynamicQuestSystem`, `QAutomatedTestSuite` lient `WorldScapeCore`/`WorldScapeNoise`/`WorldScapeVolume` sans garde `WITH_WORLDSCAPE`.
11. `UWorldScapeLod::GetClosestSurfaceNormal` et les tableaux `MeshInfo` exposés en Blueprint (aucun appelant C++ trouvé ; appelants Blueprint NON VÉRIFIÉS).
12. **QLevel est sur le chemin critique de chaque planète** : les 20 acteurs `AWorldScapeRoot` naissent dans des sous-niveaux streamés par QLevel (§1, réglages). Toute modification de l'initialisation de l'acteur doit survivre à un spawn dans un sous-niveau streamé, avec la variante `_LOptimised`.
13. **Toutes les instances de matériau de planète descendent d'un seul maître** (`M_EarthBase` : `Mi_EarthMat2`, `Mi_MarsMat1`, `Mi_Europe2`, `Mi_IO`, `Moon_Inst`, plus `MI_Ocean` pour l'océan) : changer le packing des attributs de vertex les casse toutes d'un coup.
14. **Minimap, MapView et StarMap** (`MapView_Master_Component`, `MapView_MiniMap_Component`, `MiniMap_Capture/Get_H`, `MapView_PlanetActor`, `StarMap_Component`, `Get_WS_Conf`) lisent la configuration de la planète et la hauteur du sol, comme QAI et DQS.
15. **Outillage éditeur** `Tools/WS_Tools/WS_ReplyTower_Spawner/*` (volumes de heightmap et pooling), `WS_Get`, `PasseStatic`, `Item_Random_SpawnerManager` : une régression y est silencieuse, pas visible au runtime.

---

## 7. Dette et risques relevés en passant (hors périmètre, à ne pas corriger sans demande)

- `TriangleSize * (2 ^ i)` utilise le XOR, pas la puissance (`WorldScapeRoot_Main.cpp:3105, 3163`) ; sans effet car `Init` ne pose que des positions placeholder.
- Course de données sur `WorldScapeLodInGeneration` (voir Phase 1 point 6).
- `LodGenerationThread::DoWorkMain()` finit par `delete this` et n'a aucun appelant (`WorldScapeRoot_Thread.cpp:1520-1525`).
- `WS_CalculateNormalForPatch` duplique mot pour mot `CalculateNormal` (230 lignes, `:15-248` et `:1019-1246`).
- Conversion sRGB : création avec `bSRGBConversion = false`, mises à jour avec `true` (`WorldScapeMeshComponent.h:233, 274, 282`) ; les LOD de collision et le premier frame gardent des couleurs quantifiées différemment.
- Serveur dédié : `bGenerateAllCollision` inclut `NM_DedicatedServer` et itère tous les PlayerController (`WorldScapeRoot_Collision.cpp:117-136`), génération synchrone sur le game thread. À 500 joueurs, cela peut peser ; à confronter avec « le serveur n'a pas de géométrie » (mémoire projet) : NON VÉRIFIÉ sur l'acteur placé.
- `Grid_Update` : `EnsureCompletion()` en commentaire, course MPSC connue (`WorldScapeRoot_Grid.cpp:152-166`, mémoire `worldscape-grid-mpsc-race`).
- Module GPU : deux caches `static TMap<const UHeightMapVolumeData*, ...>` jamais purgés (`WSHeightfieldManager.cpp:106`, `WorldScapeNoiseDispatcher.cpp:122`), fence bloquante dans `Shutdown`, `bInvertOrder` forcé à 0, flags `bUseGPUNoiseForFoliage`/`bValidateFoliageGPUNoise`/`bDebugFoliageGPUNoise` sans lecteur, CVars `ws.GPUNoise`/`ws.IndirectInstancing` qui réécrivent les UPROPERTY à chaque tick.
- Le serveur dédié linke `WorldScapeCompute` et `WorldScapeGPUTerrain` sans les utiliser (`WorldScape.uplugin`, `Type: Runtime`).
- `M_EarthBase` : chemin RVT à moitié câblé (mémoire `earthbase-material-profile`, travail en cours confirmé en juillet).

---

## 8. À vérifier dès que l'éditeur est ouvert (bloquants pour chiffrer, pas pour planifier)

1. Valeurs d'instance des acteurs planète, à lire dans `Content/_QLevel/Universe/Planets/<Corps>/L_<Corps>.umap` (ouvrir `L_Earth`, puis `get_all_scene_actors` sur l'`AWorldScapeRoot`) et sur `L_Dev_Claude` : `LodResolution`, `MaxLod`, `TriangleSize`, `HeightAnchor`, `bGenerateTangents`, `bGenerateCollision(ForAllPlayer)`, `CollisionResolution`, tableau `TerrainMaterial`, volumes. NON VÉRIFIÉ.
1b. Classe C++ de chaque asset de bruit (`PlanetEarth`, `TheMoon`, `VenusNoise`, `MercuryNoise`) : le chemin GPU ne couvre que `QangaNoise` et `EarthNoiseFuntion` ; si `TheMoon` est de la classe `UTheMoon`, la Lune resterait plate en mode GPU tant que ce bruit n'est pas porté.
2. Ce que `SG_WorldScape_Resolution` / `SG_WorldScape_Update` (QangaScalability) poussent réellement sur l'acteur.
3. Entrées de vertex lues par `M_EarthBase` et par le matériau océan (`get_material_graph_summary`) : VertexColor (canaux), `TexCoord` (indices), `VertexNormalWS`/`VertexTangentWS`. Détermine si UV1-3 peuvent être abandonnés (Phase 2) et si les tangentes comptent.
4. Appelants Blueprint de `GetClosestSurfaceNormal` et des tableaux `MeshInfo`.
5. Une trace Insights (Phase 0) pour remplacer le classement « suspecté » du §2 par des millisecondes.
