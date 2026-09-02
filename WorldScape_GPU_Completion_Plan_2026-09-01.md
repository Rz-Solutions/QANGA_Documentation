# WorldScape : plan d'achèvement du renderer GPU

Date : 2026-09-01. Décision de Benja : « pas de mesure, il faut finir la version GPU ». Ce document est le plan de travail, construit sur l'audit `WorldScape_ClipmapGeneration_Audit_2026-09-01.md` (cahier des charges du chemin CPU) et sur quatre lectures ciblées faites le même jour (parité des bruits, port des volumes, intégration renderer, consommateurs QANGA), plus une sonde headless qui a lu les acteurs planète réels.

Objectif contractuel : le mode « indirect » (`bUseGPUNoise && bUseIndirectInstancedNoise`) rend chaque planète de QANGA **à parité avec le chemin CPU**, la collision, le foliage et toutes les requêtes de hauteur restant sur le CPU **inchangés**. Le chemin CPU reste le fallback et le chemin du serveur dédié : il ne doit pas régresser.

Statut : plan, **aucun code modifié**. Sauvegarde md5 des 239 sources et shaders du plugin : `Saved/ClaudeBackups/2026-09-01/WorldScape_GPU_before/md5_before.txt`.

---

## 1. Ce que les planètes utilisent vraiment (sonde headless, lecture seule)

Valeurs lues sur les acteurs `AWorldScapeRoot` placés dans les niveaux (script `ws_read_planets.py`, commandlet Python, aucun asset sauvé).

| Acteur (niveau) | Type | Rayon | Bruit (classe) | Heightmap planétaire | Volumes | Matériau | Réglages |
|---|---|---|---|---|---|---|---|
| EarthScape (`_QLevel/Universe/Planets/Earth/L_Earth`) | Planète | 3 169 km | `PlanetEarth` (**EarthNoiseFuntion**, porté GPU) | `/QangaUnivers/PH_Earth` | **24 heightmap, 8 trous**, 0 bruit, 10 masques foliage | `Mi_EarthMat2` (M_EarthBase) | MaxLod 12, résolution 128, triangle 100, HeightAnchor 15 000, NoiseIntensity 2 200 000, seed 1962, `bOcean = False`, OceanHeight 1 120 000, collision 16 / 200 non paddée, tangentes off, VSM invalidation Static, occluder off |
| MarsPlanet (`Planets/Mars/L_Mars`) | Planète | 1 695 km | `BarenWorldMaterial/Mars` (**HeightMapBased**, non porté) | `MarsDataFile` | aucun | `Mi_MarsMat1` (M_EarthBase) | MaxLod 12, résolution 128, HeightAnchor 100 000, NoiseIntensity 800 000, VSM Rigid |
| Venus (`Planets/Venus/L_Venus`) | Planète | 3 005 km | `BarenWorldMaterial/Venus` (**HeightMapBased**, non porté) | `VenusData` | aucun | `Venus_inst` | idem Mars, NoiseIntensity 1 200 000 |
| Lune (`Planets/Earth/Moon/L_Moon`, non chargée par la sonde) | Planète | ? | `TheMoon` (**TheMoonNoise**, non porté, cratères) | ? | ? | `Moon_Inst` (Moon) | NON VÉRIFIÉ |
| 15 lunes (Callisto, Europe, Ganymede, IO, Deimos, Phobos, Triton, Encelade, Minas, Tethys, Titan, Miranda, Oberon, Setebos, Titania) | Planète | ? | `VenusNoise` (**WorldScapeCustomNoise**, branche Earth, non porté) | ? | ? | `Mi_Europe2` / `Mi_IO` (M_Europe) | NON VÉRIFIÉ (référence d'asset seulement) |
| Mercure (`L_MercuryBACKUP`) | Planète | ? | `MercuryNoise` (**WorldScapeCustomNoise**, branche Moon, cratères) | ? | ? | ? | NON VÉRIFIÉ |
| Asteroide_Saturn (`Maps/Universe/Sub_Levels/L_Saturn`, streamé avec la Terre) | **Monde plat** | limite 6,5e9 | sous-objet `WorldScapeCustomNoise` par défaut | aucune | aucun | `M_Invisible` | MaxLod 2, résolution 16, NoiseIntensity 1, collision off : c'est un spawner de foliage (anneau d'astéroïdes), le terrain est invisible |
| EarthScape (`Maps/LevelDev/L_Dev_Claude`) | Planète | 3 169 km | `PlanetEarth` | `PH_Earth` | 1 masque foliage | `Mi_EarthMat2_OPT` | résolution 16, triangle 300, **`bOcean = True`** (océan 8 / 600 / MaxLod 4) |

Conséquences directes :
- **Aucun tableau `MaterialsLod` n'est renseigné** sur les acteurs lus : la sélection de matériau par LOD n'est pas utilisée dans QANGA aujourd'hui.
- **`bGenerateTangents` est faux partout** : la variante de normales lissées n'est pas en jeu.
- **Trois familles de bruit non portées sont en production** : `HeightMapBased` (Mars, Vénus), `WorldScapeCustomNoise` (16 corps + le monde plat de Saturne), `TheMoonNoise` (Lune). Seule la Terre passe par le GPU tel quel.
- **La Terre dépend des volumes** : 24 volumes de heightmap (dont des sous-classes Blueprint `HMI1_C` et `Direct_HeightMap_Volume_C`) et 8 trous. Sans le port des volumes, l'Everest et les trous disparaissent en mode GPU.
- **Le monde plat est en usage** (Saturne) : le chemin plat GPU et sa bordure doivent marcher, même si ce terrain est invisible.
- Les matériaux maîtres (`M_EarthBase`, `M_Europe`, `Moon`, `M_Ocean`, T3D exportés) ne contiennent **aucun nœud TextureCoordinate** ; ils lisent `VertexColor` (6 / 3 / 0 / 0 nœuds) et `VertexNormalWS`, jamais `VertexTangentWS`. Le contrat de vertex à tenir est donc : position, normale, couleur RGBA = (hauteur normalisée, température, humidité, trou).
- `r.RayTracing=False` et `r.Lumen.HardwareRayTracing=False` (`Config/DefaultEngine.ini:154-155`) ; aucun volume RVT dans les niveaux de planète : ray tracing et sortie RVT ne sont pas des régressions (voir §6).

---

## 2. État du chemin GPU (résumé des quatre lectures)

Fait et cohérent : tuiles de heightfield en compute avec cache LRU à budget, quadtree cube-sphère à erreur écran (ou mode clipmap reproduisant les anneaux CPU), vertex buffer GPU persistant, vertex factory à origine relative caméra en double, un draw indirect par chunk, identité `UWorldScapeMeshComponent` conservée (proxy remplacé), deux harnais de validation numérique contre le CPU, chemin plat, heightmap planétaire, océan (slot 1), couleur de vertex RGBA identique au CPU, UV0 identique.

Manques, classés par impact sur QANGA :

| Manque | Impact QANGA | Preuve |
|---|---|---|
| Volumes de heightmap et de trou : uniformes câblés à 0, jamais lus | **Terre cassée** (Everest, 24 volumes, 8 trous) | `WSHeightfieldManager.cpp:1196-1198`, `WSHeightfieldGenerate.usf:55-57` |
| `HeightMapBased` non porté | **Mars et Vénus plates** (fallback CPU avec warning) | `WorldScapeNoiseGPUTypes.h:311-315`, `WorldScapeRoot_Main.cpp:428-440` |
| `WorldScapeCustomNoise` non porté | **16 corps + Saturne** en fallback CPU | idem |
| `TheMoonNoise` non porté, bruit cellulaire GPU faux (masque d'index inversé, table 64/256) | **Lune** en fallback CPU ; tout cratère impossible | `WorldScapeNoise.ush:38, 843-845` vs `CustomNoise.cpp:172-177` |
| Volumes de bruit | aucun placé, mais spawnables par pooling Blueprint | `WorldScapeRoot_Pooling.cpp` |
| Teardown : rien dans `EndPlay`/`BeginDestroy`, fence bloquante, 2 caches `static` jamais purgés | risque de gel ou fuite au stop PIE et au streaming des 20 niveaux | `WorldScapeRoot_Main.cpp:605-699`, `WSGPUTerrainRenderer.cpp:723-726`, `WSHeightfieldManager.cpp:106`, `WorldScapeNoiseDispatcher.cpp:122` |
| CVars `ws.GPUNoise` / `ws.IndirectInstancing` réécrivent les UPROPERTY à chaque tick | une console qui salit les niveaux | `WorldScapeRoot_Main.cpp:881-893` |
| Ombres et flags de proxy : 8 réglages CPU non repris (dont l'invalidation VSM), vélocité non forcée à zéro | fantômes TSR, VSM recalculée en permanence | `WorldScapeRoot_Main.cpp:3069-3081` vs `:505-508` |
| Point de vue : `bOverridePlayerPosition`, `bProjectPosition` ignorés ; pawn (CPU) vs camera manager (GPU) | LOD CPU (collision) et GPU (rendu) ancrés sur deux points | `WorldScapeRoot_Main.cpp:1335-1358, 1522-1620, 3712-3828` |
| `bInvertOrder` forcé à 0 | mode clipmap : faces arrière au pôle sud | `WSGPUTerrainRenderer.cpp:1744` |
| Bordure du monde plat absente | Saturne (invisible) : sans effet visible, parité incomplète | `WorldScapeRoot_Noise.cpp:219-227` |
| Branche eau absente du bruit Qanga, masques planétaires OceanHeight/OceanAlpha non échantillonnés | pas de corps Qanga en production ; océan Earth plat = CPU (GetOceanNoise renvoie 0) | `WSHeightfieldGenerate.usf:764-782, 362-400` |
| Écarts de port Qanga/Earth (DIFF-1, DIFF-2, DIFF-4 du rapport de parité) | DIFF-4 (bornage d'`ActualData` à 0,5 quand le masque manque) peut toucher la Terre : à mesurer avec le validateur | `WorldScapeNoiseEarthBiomes.ush:216-218` vs `EarthNoise.cpp:109` |
| Tableaux `MaterialsLod` ignorés | non utilisés dans QANGA | `WorldScapeRoot_Main.cpp:1888, 3014` |
| UV1-3 aliasés sur UV0 (convention moteur) au lieu de (0,0) | aucun matériau ne lit d'UV | `WSGPUTerrainVertexFactory.cpp:116-119` |

---

## 3. Découpage du travail

Estimations en jours de travail effectif, à prendre comme des ordres de grandeur.

### WP0. Base et instrument (1 à 2 j)
1. Build froid de l'arbre actuel (aucune modification) pour fixer la référence de compilation.
2. Étendre le validateur `bValidateIndirectInstancedNoise` (`WorldScapeRoot_Main.cpp:1775-1817`, `WSGPUTerrainRenderer.cpp:880-1100`, `WSHeightfieldSample.usf`) : boîte de tuile réelle à la place du `FBox()` dégénéré, points d'échantillon dans et autour de chaque volume touchant la tuile, verdict explicite par canal (hauteur : cible 2 cm, échec à 8 cm ; trou : égalité stricte ; température/humidité : 0,01), log greppable `WS IndirectTerrain Noise Validation`. C'est l'instrument de toute la suite : la collision restant CPU, le CPU est la référence.
3. Première passe sur la Terre de `L_Dev_Claude` avec `ws.IndirectInstancing 1` et le validateur : mesure des écarts existants (DIFF-4 en particulier). Aucun log de validation n'existe sur le disque : cette passe sera la première.

### WP1. Sécurité et hygiène du renderer (4 à 5 j)
Dans cet ordre : `bInvertOrder` (3 lignes) ; teardown (`EndPlay`/`Destroyed` libèrent `IndirectNoiseState` et appellent `ClearGPUTerrainRenderer`, fence hors GC, caches `static` passés en état d'instance avec purge sur `OnWorldCleanup`) ; CVars en surcharge non mutante (`GetEffectiveUseGPUNoise()` / `GetEffectiveUseIndirectInstancedNoise()` utilisés aux 9 sites de lecture) ; parité des flags d'ombre et de proxy (11 réglages, `PreviousLocalToWorld`, `bVelocityRelevance = false`, `bUsesLightingChannels`) ; point de vue unifié (`GetTerrainViewpoint` partagé par les trois sites). Vérification : PIE start/stop x5, changement de niveau avec deux planètes streamées, `ws.IndirectInstancing 0` à chaud.

### WP2. Volumes sur GPU (8 à 12 j)
Ordre : trous (0,5 à 1 j, valide toute la plomberie des buffers de descripteurs) ; volumes de heightmap chemin falloff (3 à 4 j) ; chemin `UseBlending` (2 à 3 j) ; culling par boîte de tuile avec marge verticale `NoiseIntensity`, hash des volumes dans `HashNoiseParams`, atlas `Texture2DArray` avec cache par `TWeakObjectPtr` et purge (2 à 3 j) ; volumes de bruit (2 à 4 j, après WP3 pour la classe par défaut `WorldScapeCustomNoise`).
Règles de port non négociables (elles conditionnent la parité collision) : le CPU tronque en **float32** après la transformation ECEF vers monde (`WorldScapeRoot_Helper.cpp:25-32`, `WorldScapeRoot_Noise.cpp:435, 481, 627`), le GPU doit faire pareil ; `GetFalloff` renvoie **1,0** quand `EdgeFalloff == 0` (NaN puis clamp, valeur par défaut), le GPU doit brancher explicitement ; le point de surface est recalculé **trois fois** (`:413, 459, 613`) entre les familles ; tri par `Priority` décroissante, la plus basse écrase ; `NoiseOffset` de `ANoiseVolume` s'auto-affecte (`NoiseVolume.cpp:61`) : on reproduit, ou on corrige des deux côtés dans le même commit.
Plafond : 16 volumes par famille et par tuile, dépassement logué et tronqué par priorité, jamais silencieux.

### WP3. Couverture des bruits (8 à 10 j)
1. Réparer `WS_CellularNoise` (masque `hash & (255 << 2)`, table `RandomVect3D` complète de 256 vecteurs) et ajouter la variante double ; porter la famille cratère (`CavityShape`, `RimShape`, `CraterShape`, `GetCrater`, `GetCraterFractal`) : 1,5 à 2 j.
2. `WorldScapeCustomNoise`, branches Earth et Moon : 2 à 2,5 j (16 corps, Mercure, Saturne).
3. `TheMoonNoise` : 1 j (mutualisé avec les cratères).
4. `HeightMapBased` (Mars, Vénus) : 1,5 à 2 j, chiffrage à confirmer après lecture de `HeightMapBased.cpp` (196 lignes).
5. DIFF-1, DIFF-2, DIFF-4 sur Qanga/Earth, décidés par le validateur (le CPU fait foi sauf bug CPU avéré) : 0,5 j.
6. Branche eau Qanga + masques planétaires OceanHeight/OceanAlpha, factorisés dans un `WS_ApplyWaterBody` partagé par les deux shaders : 1 j.
Pour chaque classe : `EGPUNoiseType`, struct de paramètres aligné 16 octets, `Fill<X>Params`, les **deux** chaînes de `Cast<>` (`WorldScapeRoot_Main.cpp:428-444` et `WorldScapeRoot_Thread.cpp:838-864`), `SHADER_PARAMETER` dans `WSHeightfieldManager.cpp` et le passe vertex, branche dans les deux `.usf`, et le hash de requête de tuile.
`QangaV2Noise` (aucun corps ne l'utilise) : différé, documenté.

### WP4. Parité renderer restante (3 à 6 j)
Bordure du monde plat (1 j) ; ombres et occlusion de l'océan par batch (0,5 j) ; flux zéro `GNullVertexBuffer` pour UV1-3 (0,5 j, précédent moteur `LocalVertexFactory.cpp:556`) ; tableaux `MaterialsLod` par palette compacte de slots (2 à 3 j, **inutilisé dans QANGA** : à faire en dernier ou à différer, décision de Benja).

### WP5. Validation et bascule (4 à 5 j)
- Validateur PASSED sur toutes les tuiles visibles : Terre (`L_Dev_Claude` puis `L_Earth` : Everest, trous, zones de volumes), Mars ou Vénus, Lune, une lune à `VenusNoise`, Saturne (plat).
- Œil de Benja (R4 du skill `qanga-verify`) : coutures entre niveaux de quadtree, pôles, horizon, transitions de LOD à vitesse de véhicule, ombres VSM, absence de fantômes TSR.
- Collision : marche et véhicule sur la Terre GPU, comparaison hauteur rendue / hauteur de collision aux points du validateur (écart cible 2 cm).
- Trois rôles réseau : serveur dédié (n'entre jamais dans le chemin GPU ; collision et foliage inchangés), client connecté, PIE standalone. Suite QATS headless verte.
- Bascule : après le WP1, `ws.GPUNoise=1` et `ws.IndirectInstancing=1` dans `Config/DefaultEngine.ini` (`[SystemSettings]`) activent le GPU sur les 20 planètes sans toucher un seul niveau ; retour arrière = deux lignes. Les flags d'instance restent disponibles pour un test par planète.

Ordre d'exécution proposé (valeur au plus tôt pour la Terre) : **WP0 → WP1 → WP2 (trous, heightmap) → WP3 (HeightMapBased, CustomNoise, cratères, TheMoon) → WP2 (volumes de bruit) → WP4 → WP5.** Point de contrôle après chaque WP : build vert, validateur, PIE vu par Benja.

Total : 28 à 40 jours de travail effectif.

---

## 4. Contrats à tenir pendant tout le chantier

1. Couleur de vertex RGBA = (HeightNormalize, Temperature, Humidity, Hole), UV0 = grille normalisée. Jamais de changement de packing.
2. Identité de classe : le primitif reste `UWorldScapeMeshComponent` (prédicats `IsA<>` de QAI), l'acteur reste `AWorldScapeRoot`, les noms de modules ne bougent pas.
3. L'API bruit (`GetGroundNoise`, `GetNoise`, `GetPawnDistanceFromGround`, projections, gravité) reste CPU et indépendante du mesh : c'est la seule connaissance du terrain sur le serveur et dans les `ParallelFor` de QAI.
4. Collision, foliage, grille QLevel, snap spawner : non touchés.
5. Le chemin CPU (`bUseGPUNoise = false`) et le mode hybride (`bUseGPUNoise` seul) restent fonctionnels : liste de non-régression au §3 WP1 et WP5. Les fichiers partagés à surveiller : `WorldScapeNoiseBiomes.ush` / `WorldScapeNoiseEarthBiomes.ush` (inclus par les deux shaders), `WorldScapeNoiseDispatcher.cpp`, `EndPlay`/`Destroyed`, le point de vue unifié qui alimente aussi `UpdatePosition`.
6. Règles C++ du projet (`Documentation/ModernCPP_UE57_RzZz_Guide.md`) : C++20, pas d'exception, pas d'initialiseur désigné sur USTRUCT, `TWeakObjectPtr` dans tout callback différé, pas de `static` d'instance. Sources en ASCII pur, pas de tiret cadratin.
7. Sauvegarde et md5 avant chaque fichier touché (`Saved/ClaudeBackups/2026-09-01/`), état du disque annoncé fichier par fichier à chaque livraison. Un changement C++ ou shader n'existe pas tant qu'il n'est pas compilé (`Result: Succeeded`).

---

## 5. Comment on prouve la parité

- **Validateur numérique** (WP0) : à chaque génération de tuile, 12 texels fixes plus les points de volume, comparés au CPU `GetGroundNoise` avec les volumes de la tuile ; verdict par canal, log throttlé, compteur de tuiles PASSED/FAILED exposé par `bDebugIndirectInstancedNoise`.
- **Buffers de référence** : dump des tuiles GPU (hauteur, aux) et des sommets CPU aux mêmes positions monde sur une trajectoire fixe de `L_Dev_Claude`, comparaison par script.
- **Visuel** : Benja en PIE, avec `ws.GPUTerrain.Indirect.ForceVertexColorView` pour vérifier la couleur de vertex, et la comparaison CPU / GPU à bascule à chaud (`ws.IndirectInstancing 0/1`).
- **Collision** : personnage et véhicule sur les zones à volumes ; aucune interpénétration ni flottement visible ; écart hauteur rendue / collision aux points du validateur.

---

## 6. Différé, avec justification

- **Ray tracing** : `r.RayTracing=False` dans le projet, le BLAS CPU ne tourne pas non plus ; chemin 5.7.3 documenté (`FRayTracingGeometry::RequestBuildIfNeeded`, `AddRayTracingGeometryUpdate(ViewIndex, ...)`, budget `r.RayTracing.Geometry.MaxBuiltPrimitivesPerFrame`) : 4 à 6 j si un jour activé.
- **Sortie RVT** : le bloc `DrawStaticElements` du proxy CPU est inerte en 5.7.3 (`bSupportsRuntimeVirtualTexture` jamais posé, `PrimitiveSceneProxy.cpp:390`), aucun mesh WorldScape ne reçoit de RVT, aucun volume RVT dans les niveaux de planète : parité avec rien. Ticket séparé si un terrain RVT est voulu (4 à 6 j).
- **PSO precaching** : ni l'un ni l'autre chemin ne le fait ; l'activer exige `GetPSOPrecacheVertexFetchElements` (sinon `checkNoEntry`) : nouvelle fonctionnalité, 2 à 3 j, hors parité.
- **Mesh draw commands cachées** : jamais, les draws indirects par chunk sont refaits à chaque frame.
- **Écran partagé / plusieurs joueurs locaux** : index joueur 0 figé des deux côtés, préexistant.
- **`QangaV2Noise`** : aucun corps ne l'utilise.

---

## 7. Questions ouvertes pour Benja

1. Bascule par CVars dans `DefaultEngine.ini` (recommandé) ou par les flags des 20 acteurs ?
2. Mode de production : quadtree (recommandé, défaut du code) ou mode clipmap (`bIndirectInstancedNoiseUseClipmaps`) ?
3. `MaterialsLod` inutilisé dans QANGA : à implémenter en dernier (2 à 3 j) ou à différer ?
4. `bOcean = False` sur `L_Earth` mais `True` sur `L_Dev_Claude` : l'océan de la Terre en production vient-il d'ailleurs que de WorldScape ? Cela fixe la priorité de la parité océan.

---

## 8. Journal d'exécution (2026-09-01 au 2026-09-02, session Claude, goal "tous les WPx à 100 %")

État par lot (tout est écrit et compilé sauf mention contraire ; le détail des fichiers est dans `Saved/ClaudeBackups/2026-09-01/`).

- **WP0 (instrument)** : validateur enrichi : boîte de tuile réelle, échantillons sous chaque volume, prédicat robuste au NaN (`!(diff <= tol)`), tolérance absolue `ws.GPUTerrain.Validation.HeightToleranceCm` (8) et relative `ws.GPUTerrain.Validation.HeightToleranceRelative` (1e-5 x NoiseIntensity), position ECEF et échantillons planétaires bruts (Hum, Temp, Height) dans la ligne FAILED, comparaison couleur de vertex contre l'aux en linéaire. Résumé de log : `scratchpad/ws_log_summary.py`.
- **WP1 (hygiène)** : livré et compilé (CVars non mutantes, release propre, keeper mesh synchronisé, eau sans ombre, PreviousLocalToWorld). Correctif tardif : `Prev_bUseGPUNoise` / `Prev_bUseIndirectInstancedNoise` étaient réassignés depuis les propriétés brutes après régénération, ce qui relançait une régénération à chaque image dès qu'une CVar surchargeait l'acteur ; ils prennent maintenant les valeurs effectives.
- **WP2 (volumes)** : port complet des volumes heightmap et trous (`WSHeightfieldVolumes.ush`, atlas R32F, snapshot haché par tick). Les volumes de bruit ne sont pas portés : aucun corps QANGA n'en place ; un root qui en aurait reste sur le CPU avec un avertissement (`GetEffectiveUseIndirectInstancedNoise`). Validation sur `L_Earth` (24 volumes heightmap, 8 trous) encore à faire.
- **WP3 (bruits)** : les cinq classes ont un port GPU (Qanga 0, Earth 1, CustomWorld 2, TheMoon 3, HeightMapBased 4, enum unique `EGPUNoiseType`), hash par valeur des paramètres (une édition d'asset régénère les tuiles), correctif `WS_CellularNoise` (table de 256 vecteurs). Parité mesurée seulement sur Earth (voir plus bas) ; Mars, Vénus, Lune et une lune `VenusNoise` restent à mesurer (scripts `ws_swap_noise.py` et sessions `L_Mars` / `L_Venus`).
- **WP4 (parité renderer)** : bordure du monde plat (uniformes + early-out du noyau), `bUseAsOccluder` par batch océan, flux UV1-3 à zéro (`GNullVertexBuffer`), palette `MaterialsLod` (slots 2+ du keeper mesh, LOD équivalent = round(log2(cellule / TriangleSize))), UV planétaire en double précision (`WSPreciseTrigD.ush`), couleur de vertex **linéaire** comme le CPU (le noyau encodait en sRGB : M_EarthBase lit VertexColor 29 fois, les seuils de biome sont réglés en linéaire).
- **WP5 (validation)** : en cours. Mesures Earth (`L_Dev_Claude`, LodResolution 16, TileRes 128) après les correctifs : 216 PASSED / 0 FAILED sur une tournée caméra de 13 arrêts, puis 1370 PASSED / 1 FAILED sur 38 tuiles distinctes pendant un vol de Benja ; écart de hauteur max 7 cm hors le cas isolé ci-dessous, température et humidité à 5e-4 près.

Écarts trouvés par le validateur et tranchés (le CPU fait foi) :

1. **UV planétaire en float** : `atan2` / `asin` HLSL n'existent pas en double (erreur 6e-8, soit un mètre au sol sur une carte 8192 x 8192 en bicubique). Newton en double depuis la graine float (`WS_Atan2PreciseD`, `WS_AsinPreciseD`, `WS_SqrtPreciseD`).
2. **DIFF-4** : le port remplaçait tout échantillon planétaire négatif par 0,5 (ou 0). Or la bicubique déborde légèrement sous 0 près des bords de masque et le CPU fait alors `pow(negatif, 0.6)` = NaN puis `NoiseMathUtils::Clamp(NaN)` = 0 (mesuré : température CPU 0,0000 contre GPU 0,69). `WS_PowStdF` reproduit `std::pow` (exposant entier = puissance signée, fractionnaire = NaN) et `saturate` ramène le NaN à 0 comme le CPU. Règle pour tout port futur : ne jamais assainir une entrée que le CPU ne nettoie pas.
3. **Couleur de vertex sRGB** : voir WP4.
4. **Erreur float résiduelle** : les transcendantes du bruit tournent en float sur GPU, écart mesuré jusqu'à 28 cm sur 9,7 km de relief avant tolérance relative ; c'est très en dessous de l'écart intrinsèque entre le clipmap de rendu CPU et son maillage de collision (pas de 200 contre 100).

Blocage découvert le 2026-09-02 : à **LodResolution 128 (valeur de la Terre de production)** le cadre entier devient plat (noir, affiché blanc par les visionneuses car alpha = 0 : mesurer les pixels, ne pas se fier à l'oeil). Cause : le pool de sommets persistant (112 octets par sommet, 16 384 sommets par tuile, free-lists par résolution sans compactage) monte à 1,7 à 1,9 Go et le rendu casse sans aucune erreur RHI. Correctifs écrits (à compiler) : plafond `ws.GPUTerrain.MaxVertexPoolMB` (768) avec chunks non maillés au-delà et avertissement, compactage des layouts quand le span alloué dépasse 1,5 fois l'actif, caps de feuilles dérivés du pool (80 % terre / 20 % océan, en 1 / TriRes^2) et cap de feuilles **dur** dans le quadtree (projection feuilles + file, fusion forcée au-dessus du cap).

Garde ajoutée : `GetEffectiveUseGPUNoise()` rend faux sur un serveur dédié ou un process NullRHI (cooker, tests headless) ; le thread LOD hybride n'avait aucune garde et `ws.GPUNoise=1` en `.ini` aurait tenté un compute shader sur le serveur.

Reste à faire pour clore WP5 : build des correctifs ci-dessus, relance et mesure à LodResolution 128 (`L_Dev_Claude` puis `L_Earth` avec volumes), Mars / Vénus (`HeightMapBased`), Lune et lune `VenusNoise` par échange de bruit, Saturne (plat), collision (marche), trois rôles réseau, suite QATS headless, build `QangaServer`, puis bascule `ws.GPUNoise=1` + `ws.IndirectInstancing=1` dans `Config/DefaultEngine.ini` ([SystemSettings]).

Mesures après recompilation (2026-09-02, 01:00 à 01:20, `L_Dev_Claude` à **LodResolution 128**, pool 650 à 700 Mo, caps L=351 / O=128) :

| Classe (asset échangé à chaud sur EarthScape, NoiseIntensity 2,2e6) | Résultat | Écart max |
|---|---|---|
| Earth (PlanetEarth) | 285 PASSED / 0 FAILED | 8,6 cm |
| HeightMapBased (Mars.Mars) | 95 PASSED / 0 FAILED | 5,9 cm |
| TheMoon (TheMoon.TheMoon) | 3/12 à 49,5 cm avant, 23 PASSED / 0 FAILED après | 9 cm sur 36 km de relief |
| CustomWorld (MoonNoise) | 2/12 à 59 cm sur 88 km de relief (6,7e-6, résidu float) | 59 cm, dans la tolérance relative à la hauteur |

Le cas TheMoon venait de `WS_AsinD` (erreur 3,7e-4) appliqué à la latitude ; les ports TheMoon et CustomWorld utilisent maintenant `WS_AsinPreciseD` sur une latitude arrondie en float, comme le CPU (`float lattitude = GetLattitude(...)`). La tolérance du validateur devient `max(8 cm, 2e-5 x max(|hauteur CPU|, NoiseIntensity))` : les résidus float mesurés vont de 2,5e-6 (TheMoon) à 6,7e-6 (CustomWorld) de la hauteur. Couverture : le validateur tourne désormais sur toutes les tuiles rendables à tour de rôle (une par tick).

Session `L_Earth` (2026-09-02, 01:25 à 01:35, planète de production, 24 volumes heightmap + 8 trous, LodResolution 128) : 133 PASSED / 0 FAILED sur 18 tuiles, dont des validations à 17, 22, 27, 32 et 52 échantillons (les points ajoutés sous chaque volume) : le port des volumes est conforme au CPU (écart max 22 cm sur NoiseIntensity 2,2e6). Deux défauts de flux trouvés grâce à l'oeil de Benja (« les tuiles GPU mettent beaucoup trop de temps à apparaître ») :

1. Le validateur lui-même : 12 à 52 appels `GetGroundNoise` CPU par tick sur le game thread, avec les 24 volumes, écrase la cadence de l'éditeur. Il ne doit jamais rester actif pour juger la vitesse ; il est coupé par défaut.
2. `FWSQuadtreeManager::EnsureRoots` comparait `RootSizeWorld` même sur une sphère. Ce paramètre ne sert qu'au monde plat, mais le root le recalcule avec le multiplicateur d'altitude du clipmap (un doublement par bande au-dessus de HeightAnchor) : chaque changement de bande d'altitude remettait le quadtree à zéro et regénérait toutes ses tuiles (dans le log : `Chunks=3` puis remontée). Corrigé : la comparaison ne vaut qu'en monde plat.

Un compteur `SplitDeniedNeighbor` est ajouté aux statistiques du quadtree pour mesurer les subdivisions refusées par la création des voisins (soupçon aux bords de face de cube), à lire après recompilation.

Décision de Benja (2026-09-02, 01:57) : pas de build de la cible `QangaServer` pour ce chantier (« ça doit marcher comme avant ») ; le contrat serveur est tenu par le code (`GetEffectiveUseGPUNoise()` rend faux sur serveur dédié et NullRHI, le clipmap CPU est inchangé). La suite QATS headless est différée à une fenêtre où la machine est libre. Crash éditeur du 01:47 (Undo avec le GPU actif : `UpdateTerrainMaterial` déréférençait les meshes des LOD CPU détruits par le mode indirect) : gardes `IsValid` ajoutées dans les deux fonctions de matériau et dans la boucle de visibilité du tick, à compiler.

Vitesse d'apparition des tuiles (retour de Benja, 02:00 : « le quadtree est à la ramasse ») : mesure dans les logs des sessions précédentes : la file de génération contient 20 à 76 tuiles en permanence mais **une seule tuile est générée par image** (`HFGen=1`, le compte adaptatif retombait à 1 dès qu'une image dépassait le budget de 2 ms, la création des deux textures RHI d'une tuile suffisant à le dépasser), et chaque croissance du buffer de sommets (taille exacte) regénérait tous les maillages (rafales de 279 à 585 dispatches). Correctifs écrits (à compiler) : pool de textures de tuile par résolution (réutilisation à l'éviction), plancher de 4 tuiles par tick et croissance x2, budget `ws.GPUTerrain.Indirect.HeightfieldTickBudgetMs` 2 ms -> 6 ms, marge de 50 % (min 32 Mo) à chaque réallocation du buffer de sommets, pré-requête des petits-enfants quand un noeud a encore au moins deux niveaux à descendre. Objectif mesurable : 700 tuiles en moins de 5 s à LodResolution 16 (au lieu de 30 s), plateau en moins de 10 s à 128.

Mesures de vitesse après correctifs (builds `wp5_build3` à `wp5_build5`, `L_Dev_Claude`, caméra fixe à 7 km) :

| Configuration | Rampe (tuiles) | Régime stable |
|---|---|---|
| LodResolution 16, avant | 703 tuiles en ~30 s (1 tuile par image) | |
| LodResolution 16, après | 8 puis 122 tuiles en 2 s (4 par image, file vidée) | 58 images/s |
| LodResolution 128, seuil 2 px, pré-requête petits-enfants | 423 tuiles en 1 s mais boucle (64 évictions et 64 générations par image) | 8 images/s : retiré |
| LodResolution 128, seuil 2 px, sans pré-requête, éviction interdite dans l'image de la demande | 6 puis 345 tuiles en 6 s, puis 0 régénération | 345 tuiles, 5,6 M sommets, 604 Mo, 21 images/s |
| LodResolution 128, seuil 6 px | | 136 tuiles, 2,1 M sommets, 336 Mo, 42 images/s |

Le seuil d'erreur écran (`Quadtree_ScreenSpaceErrorThresholdPixels`, 2 px sur `L_Dev_Claude`) est le réglage qualité / performance : à 2 px le quadtree dessine 25 à 35 fois plus de sommets que le clipmap CPU pour la même LodResolution (128 sommets par tuile contre 128 par anneau). Un régulateur par temps d'image est ajouté à la génération (une tuile par tick sous 20 images/s, croissance au-dessus de 40) pour éviter les chutes à 2 images/s pendant une rampe.

Point d'attention : l'éditeur de validation est partagé (Benja vole dans la carte, une autre session Claude a échangé le bruit du root par une copie transiente `WSHashProbeNoise` puis l'a restauré). Lire `Regen Request by ...` dans le log avant d'interpréter une mesure.

Session finale `L_Dev_Claude` (2026-09-02, 02:35 à 02:50, binaires `wp5_build5`). Consignes de Benja avant d'aller se coucher : plus aucun build lancé par Claude (il compile lui-même), aucune session `L_Earth`, couper le moteur en fin de tâche.

| Mesure (LodResolution 128, seuil 2 px, validateur coupé) | Résultat |
|---|---|
| Rampe, caméra fixe à 7 km | 6 puis 348 tuiles en 8,2 s (95 % du plateau) ; plateau 366 tuiles, 5,6 M sommets, buffer 768 Mo (plafond), 0 éviction, 0 rafale de maillage |
| Cadence pendant la rampe | 49, 39, 28, 25, 23, 22 puis 21 images/s en régime stable : c'est le nombre de tuiles rendues qui fait la cadence, pas la génération ; aucune chute à 2 images/s après la première seconde |
| Téléportation de 50 km | toutes les tuiles visibles rendables à chaque tick, mais file de 388 tuiles vidée à **1 tuile par tick pendant 24 s** |
| Validateur, 75 s | 73 PASSED / 0 FAILED, écart max de hauteur 5,88 cm |
| Rendu | deux captures mesurées (pixels texturés, aucune image plate) aux deux positions |

Défaut mesuré et corrigé, **écrit mais NON compilé (à compiler par Benja)** : le régulateur par temps d'image de `wp5_build5` (seuils absolus : diviser par deux au-dessus de 50 ms, croître sous 25 ms) restait au plancher d'une tuile par tick dès que l'éditeur tournait à 21 images/s à cause du rendu lui-même, et ne pouvait jamais remonter. `FWSHeightfieldManager` garde maintenant la moyenne du temps d'image des ticks sans génération (`IdleFrameSecondsAverage` : descente immédiate, montée lente à 2 % par tick) et compare en relatif : division par deux au-delà de 1,5 fois cette référence (jamais sous 50 ms), croissance tant que l'image reste sous 1,25 fois (jamais sous 25 ms), plafond de 64 tuiles par tick (`MaxAdaptiveGenerationsPerTick` ; il n'y en avait pas, un delta nul au démarrage aurait fait croître le compte sans limite). Fichiers : `Plugins/WorldScape/Source/WorldScapeGPUTerrain/Public/WSHeightfieldManager.h` (trois membres) et `Private/WSHeightfieldManager.cpp` (bloc en tête de `Tick`, bloc régulateur, `GenerationsLastTick = Jobs.Num()`). Vérification attendue après compilation : téléportation de 50 km à LodResolution 128, `HFGen` doit monter à 4, 8, 16 tant que la cadence tient et la file se vider en moins de 5 s ; en cas de doute, `ws.GPUTerrain.Indirect.HeightfieldTickBudgetMs 0` désactive toute régulation (file entière par tick).

Piège d'outillage : `ws_launch_editor.ps1 -Map /Game/...` lancé depuis le shell Bash arrive avec `C:/Program Files/Git/Game/...` (conversion de chemin MSYS) et l'éditeur ouvre une carte vide ; charger ensuite la carte par `LevelEditorSubsystem.load_level` depuis le pont (vérifié), ou lancer depuis PowerShell.

État en fin de session (2026-09-02, 02:50) : éditeur quitté proprement sans sauvegarde (`new_blank_map(False)` puis `quit_editor()`, `LogExit: Exiting` dans le log), `Content/Maps/LevelDev/L_Dev_Claude.umap` inchangé (md5 `41e9de097004c5c3ba1846ea229f20e5`) ; seule une copie d'autosave de la session (flags GPU posés) existe dans `Saved/Autosaves/Game/Maps/LevelDev/L_Dev_Claude_Auto1.umap`, à ignorer. Bascule `ws.GPUNoise=1` + `ws.IndirectInstancing=1` dans `Config/DefaultEngine.ini` NON appliquée ; suite QATS headless et build `QangaServer` non lancés (décision de Benja) ; tout le C++ et les shaders sont compilés dans les binaires éditeur courants (`wp5_build5`) sauf le régulateur relatif ci-dessus, à compiler par Benja.

Reste ouvert au 2026-09-02, 03:05, et pourquoi :

1. **Compilation du régulateur relatif** : interdite à Claude par Benja (« c'est moi qui ferai ça ») ; une simple vérification de syntaxe du fichier par le compilateur seul a aussi été refusée par la couche de permissions de l'outil. Le changement est donc lu et relu, mais ni compilé ni exécuté.
2. **Bascule projet** (`ws.GPUNoise=1` + `ws.IndirectInstancing=1`) : volontairement non appliquée. Trois raisons techniques, au-delà du choix d'équipe : les binaires jeu et serveur sur disque n'ont pas été reconstruits avec ce chantier (une bascule `.ini` activerait dans un build existant l'ancien chemin GPU incomplet) ; la carte `Univers` n'a pas été testée avec plusieurs planètes GPU simultanées (le pool de sommets de 768 Mo et le budget de heightfield sont par root, pas globaux : à mesurer avant d'activer partout) ; le régulateur corrigé n'est pas compilé.
3. **Suite QATS headless et build `QangaServer`** : build serveur refusé par Benja ; lancement headless de la suite QATS refusé par la couche de permissions de l'outil cette nuit. À lancer par Benja : `run_qats_headless.ps1` du scratchpad de session, ou la commande de la section « Lancer les tests sans éditeur ouvert » du skill `qanga-verify`.

### Retour de Benja (2026-09-02, 10:22) : « les performances sont encore plus lentes que le clipmap CPU » et crash « Corrupt hash »

**Crash.** `Assertion failed: NextElementIndex != INDEX_NONE ... Corrupt hash` dans `TSet::RemoveImpl` appelé par `FWSQuadtreeManager::RemoveChildrenRecursive` (`WSQuadtreeManager.cpp:416`) depuis `Tick_RenderThread`. Cause lue dans le code : la boucle de parcours prenait une référence `FWSQuadtreeNode& Node = NodePool[NodeIndex]` puis appelait `EnsureKeyExists` / `FindOrAddNode`, qui allouent des noeuds (`NodePool.AddDefaulted()` réalloue le TArray quand la capacité est dépassée) ; les écritures suivantes (`Node.Children[ChildSlot] = ChildIndex`) partaient dans le buffer libéré : enfants perdus (noeuds orphelins) et tas corrompu, d'où l'assertion plus tard dans `NodeIndexByKey.Remove`. Même motif dans la lambda `EnsureKeyExists` (`Node` pris sur `NodePool[CurrentIndex]` avant `FindOrAddNode`). Le bug est probabiliste (il faut une réallocation pendant le tick puis une réutilisation du bloc) : dix sessions de mesure sans crash ne prouvaient rien. Correctif écrit, **à compiler par Benja** : tout accès après un appel allouant passe par l'index (`NodePool[NodeIndex].…`, `NodePool[CurrentIndex].…`), sept sites dans `WSQuadtreeManager.cpp`.

**Performance, mesurée sur `L_Dev_Claude`** (2026-09-02, 10:36 à 11:00, binaires de Benja de 10:06, viewport éditeur 1413 x 711, LodResolution 128, caméra fixe, huit secondes de comptage d'images par point, chaque configuration stabilisée avant la mesure) :

| Position | Clipmap CPU | GPU seuil 2 px (défaut jusqu'ici) | GPU seuil 8 px | GPU seuil 16 px |
|---|---|---|---|---|
| A : 7 km d'altitude, tangage -35° | 46,9 img/s | 18,7 img/s (372 tuiles, 5,7 M sommets, buffer au plafond 768 Mo, 185 subdivisions refusées par tick) | 38,9 img/s (145 tuiles, 2,2 M sommets) | 58,5 img/s (41 tuiles, 0,57 M sommets) |
| B : 1,5 km d'altitude, tangage -20° | 51,2 img/s | 18,7 img/s (380 tuiles, 5,7 M sommets, 206 refus par tick) | 34,6 à 37 img/s (160 tuiles, 2,5 M sommets) | 49,8 img/s (80 tuiles, 1,2 M sommets) |

Benja a raison : au seuil par défaut de 2 px, le chemin GPU est 2,5 fois plus lent que le clipmap CPU. Ce n'est pas la génération (files vides, 0 éviction en régime stable) mais la densité géométrique : 2 px dessine 5,7 M sommets là où le clipmap CPU en dessine de l'ordre de 0,8 M (16 LOD x 3 sections). La cadence suit le nombre de sommets : parité avec le clipmap vers 16 px sur ce viewport, moins 25 % à 8 px. Le clipmap CPU ne tourne plus quand le GPU est actif (`CheckForLodGeneration` annule les tâches et sort, `UpdateLOD` sort au début), donc il n'y a pas de double coût.

**Correctif écrit, à compiler par Benja : seuil automatique.** `Quadtree_ScreenSpaceErrorThresholdPixels` passe à **0 = automatique** par défaut : seuil = `ws.GPUTerrain.Quadtree.AutoThresholdFactor` (2,75) x (hauteur du viewport / (2 tan(fov/2))) / texels par côté de tuile, soit 8 px sur le viewport de mesure et 12 px en 1080p à 90° de champ : le rendu du clipmap CPU quel que soit l'écran (captures CPU et GPU 8 px identiques au pixel près, moyennes R/G/B à 0,5 près), pour 25 % de cadence en moins. Le facteur 5,5 (16 px) donne la parité de cadence mais lisse les masques de biome (capture 16 px : aplat sombre au premier plan). Le seuil effectif est imprimé dans la ligne de statistiques (`SSE=`). Une valeur explicite en pixels reste possible ; facteur plus bas = plus fin, plus haut = plus rapide (5,5 équivaut au point 16 px du tableau). Fichiers : `WSQuadtreeManager.h/.cpp`, `WSGPUTerrainRenderer.cpp` (impression), `WorldScapeRoot.h` (défaut), `WorldScapeRoot_Main.cpp` (CVar et plomberie).

**Limite à connaître, vue sur les captures** : les masques de biome de `M_EarthBase` et la profondeur d'eau sont portés par la couleur de vertex ; une maille plus grossière lisse ces masques. Le clipmap CPU concentre ses sommets sous la caméra (anneaux LOD0 minuscules), le quadtree les répartit à erreur écran constante : à densité égale, le GPU est plus fin au sol et plus grossier vu de 1,5 km d'altitude. Le défaut retenu est le rendu du clipmap (8 px) ; la piste pour rattraper la cadence sans perdre le rendu est un seuil dépendant de la distance (plus fin sous la caméra, plus grossier au loin, comme les anneaux du clipmap) : à expérimenter après compilation.

**Piège d'outillage** : `viewport_control screenshot` a coupé le mode temps réel du viewport (plus aucun tick d'acteur, statistiques et logs périodiques arrêtés, Slate toujours à 59 images/s) ; `LevelEditorSubsystem.editor_set_viewport_realtime(True)` le rétablit. À vérifier après chaque capture.

### Retour de Benja (2026-09-02, 11:20) : « les tuiles GPU sont moins définies que le clipmap pour une résolution donnée, le foliage suit la surface cible mais pas la maille »

**Mesure au sol sur `L_Dev_Claude`** (caméra à 1,8 m au-dessus du terrain, point de terre à (-60 km, 85 km) à 727 m au-dessus de la mer, LodResolution 128, binaires de Benja de 10:06, seuil 2 px) : `ChunkLod=[3..11]`, 294 subdivisions refusées par tick, 453 feuilles (cap atteint), `QMaxDepthWant=0`. La tuile de profondeur 11 fait 3,1 km de côté sur ce rayon de 3 169 km, soit des cellules de **24 m sous les pieds**, là où le clipmap CPU dessine des cellules de 3 m (TriangleSize 300 cm) sur son anneau LOD0. Même résultat à l'origine (fond marin). Benja a raison, et ce n'est ni la profondeur maximale (16, jamais atteinte) ni le seuil.

**Cause, lue dans le code et vérifiée par le calcul** : l'erreur écran d'une tuile (`WSQuadtreeManager.cpp`, boucle de parcours) mesure la distance caméra-tuile vers le centre de la tuile projeté **sur la sphère de rayon PlanetScale**, alors que le terrain de ce niveau est 10,6 à 11,9 km au-dessus de cette sphère (NoiseIntensity 22 km). À hauteur d'homme, chaque tuile paraît donc à 8 à 10 km : profondeur 11 (3,1 km, cellule 24 m) : distance 9,8 km, erreur 0,87 px < 2 px, arrêt ; profondeur 10 : 2,8 px, subdivision. La prédiction tombe exactement sur le `ChunkLod` mesuré. Le clipmap n'a pas ce problème : ses anneaux suivent `PlayerDistanceToGround`.

**Correctif écrit, à compiler par Benja** : `FWSQuadtreeFrameParams::CameraGroundRadius` (rayon du terrain sous la caméra = rayon de la caméra moins `PlayerDistanceToGround`, rempli par le root à côté de `ScreenHeightPixels`) ; le parcours projette les échantillons de tuile sur ce rayon (`SurfaceRadius`, neuf sites) au lieu de `PlanetScale`. Premier ordre : le terrain autour de la caméra est supposé à la même altitude ; la suite possible est une borne de hauteur par tuile lue dans le heightfield généré. Attendu après compilation, au sol : `ChunkLod` jusqu'à 16 (cellules 0,76 m, quatre fois plus fin que le clipmap à 3 m), et avec le seuil automatique une maille plus fine que le clipmap sous les pieds et équivalente au loin. Pour aller au-delà (l'objectif de Benja : augmenter la résolution du terrain par la voie GPU), `Quadtree_MaxDepth` peut monter (clé de tuile sur 5 bits de profondeur et 27 bits par axe, donc 27 maximum ; profondeur 20 = cellules 4,7 cm sur ce rayon) et le cap de feuilles suit le pool de sommets (768 Mo, 112 octets par sommet : un sommet plus compact multiplierait le nombre de tuiles).

Fichiers : `WSQuadtreeManager.h` (champ), `WSQuadtreeManager.cpp` (rayon de surface), `WorldScapeRoot_Main.cpp` (remplissage).

Compléments mesurés au même point de terre : au seuil 8 px (équivalent du mode automatique), `ChunkLod=[1..10]`, cellules de 49 m sous les pieds, seulement 10 refus de cap : le cap n'est pas en cause, la distance l'est. Le root de `L_Dev_Claude` n'a qu'un matériau de terrain (`Mi_EarthMat2_OPT`, aucune liste par LOD) : la différence d'aspect au sol entre le clipmap (herbe verte, rochers) et le GPU (sol sombre) vient uniquement de la maille grossière, les masques de biome étant lus par vertex ; captures `land_cpu.png` et `land_gpu2.png` (89 % des pixels du tiers inférieur diffèrent).
