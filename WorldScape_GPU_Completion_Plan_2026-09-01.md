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

### Retour de Benja (2026-09-02, 19:30) : « c'est mieux, mais des tuiles font n'importe quoi par moments, déformées, étirées ; PlanetScape a une bonne logique de quadtree »

Binaires de Benja de 13:05 (crash, seuil automatique et rayon de terrain inclus), test dans `L_Persistent_Universe` sur des roots à LodResolution 16 (93 avertissements « Low detail configuration ... TileRes=16 »). Deux audits de code en parallèle (cycle de vie des tuiles GPU ; quadtree de PlanetScape) ; pas de capture, à la demande de Benja.

**Défaut confirmé n° 1 : compactage du pool de sommets après la prise des plages.** Dans `FWSGPUTerrainRenderer::Tick_RenderThread`, les `FChunkPrep` copient `VertexBase` depuis `TileVertexLayout`, puis le bloc de compactage (déclenché quand le pool alloué dépasse d'une moitié les sommets actifs : saut de caméra, changement de LOD) réécrit toutes les bases dans la carte sans rafraîchir les preps. La frame du compactage dispatch et dessine aux anciennes bases et enregistre les tuiles comme à jour ; la frame suivante lit les nouvelles bases : chaque chunk dessine les sommets d'une autre tuile, ou de la mémoire jamais écrite (triangles étirés), jusqu'à une invalidation. Variante aggravante : le buffer peut être rétréci la même frame sur la base des nouvelles plages alors que les anciennes bases dépassent. Correctif écrit : après le rebasage, les preps relisent leur plage dans la carte (le `Reset` des états de maillage force déjà le remaillage de tous les chunks dans la même frame, sans budget, donc aucune frame sans données).

**Défaut confirmé n° 2 : identifiant de génération redémarrant à 1.** `FWSHeightfieldTile::GenerationId` était un compteur par objet tuile ; une tuile recréée (éviction puis nouvelle demande, ou recréation du gestionnaire entier sur changement de résolution ou de budget, ce que Benja fait en basculant LodResolution) repartait à 1, et le renderer, qui compare cet identifiant à celui du dernier maillage construit pour la clé, pouvait juger le chunk à jour et garder l'ancien maillage. Correctif écrit : compteur monotone à l'échelle du process (`GWSHeightfieldGenerationCounter`), 0 réservé à « jamais généré ».

**Plausible, non corrigé : couture limitée au 2:1.** `WSMeshGenerate.usf` ne coud qu'un niveau d'écart et le masque est calculé contre le voisin de profondeur - 1 ; un voisin émis en feuille deux niveaux plus grossier (enfants pas prêts, cap de feuilles) laisse des jonctions en T (fissures en bord, pas des tuiles étirées). PlanetScape n'a aucune couture : il cache les fissures par des jupes de 500 m et calcule chaque sommet depuis l'UV de face absolue (bords partagés bit-identiques). Piste : jupes dans le générateur de maillage.

**Ce que PlanetScape fait de plus robuste, à reprendre si les défauts ci-dessus ne suffisent pas** (`Plugins/PlanetScape`, étude du 2026-09-02) : distance d'erreur corrigée par l'altitude (composante tangentielle + altitude caméra au-dessus du terrain le plus haut, centre de tuile déplacé de la hauteur moyenne de la tuile : `TileScheduler.cpp:783-809`, `:892-897`), quatre couches d'hystérésis (moyenne glissante de l'erreur, fusion sous 0,4 x seuil, seuil x 1,5 à saturation, marge de 15 % à l'éviction), subdivision autorisée seulement depuis un parent qui a déjà un maillage, parent visible tant que les quatre enfants ne sont pas actifs, compteur de génération par dispatch avec rejet des résultats périmés, tranche en vol qui voyage avec le dispatch et revient avec lui, verrou `bReadbackInFlight` contre la publication d'un résultat sous la mauvaise clé, créneaux bloqués mis en quarantaine et jamais recyclés, données d'instance écrites en une fois et jamais modifiées par frame.

Fichiers : `WSGPUTerrainRenderer.cpp` (rafraîchissement des preps après compactage), `WSHeightfieldManager.cpp` et `.h` (compteur). Note : `WorldScapeRoot_Main.cpp` porte une modification du 2026-09-02 à 19:05 qui n'est ni de cette session ni des autres sessions Claude (toutes interrogées) : édition de Benja, à confirmer par lui ; mes correctifs du soir ne touchent pas ce fichier.

### Retour de Benja (2026-09-02, 20:00) : « imprécision illogique sur les tuiles profondes au sol à 128 » (captures : damier régulier de bosses et de creux, fil de fer avec un sommet sur deux décalé)

**Cause, lue dans le code** : `WS_SqrtD(double X) { return sqrt(X); }` dans `WorldScapeNoiseMath.ush`. HLSL n'a pas de racine carrée en double : le compilateur arrondit l'argument en float, calcule en float et reconvertit. Le maillage (`WSMeshGenerate.usf`) normalise chaque sommet avec cette racine sur la longueur au carré d'une position planétaire (1e17) : erreur relative 6e-8, soit 19 cm de gigue radiale par sommet à 3 169 km de rayon. Invisible sur des cellules de 24 m (profondeur 11, tout ce qui était validé avant le correctif de rayon de terrain), énorme sur des cellules de 76 cm (profondeur 16, atteinte depuis le correctif) : un quart de la cellule, d'où le damier. Le validateur de parité compare les texels du heightfield, pas les sommets du maillage, et ne pouvait pas le voir. Même racine dans la normalisation du shader de heightfield (UV planétaire), dans les volumes et dans le compute de bruit.

**Correctif écrit, à compiler par Benja (shader : `recompileshaders changed` ou redémarrage de l'éditeur suffit, pas de C++)** : `WS_SqrtD` fait deux itérations de Newton en double à partir de la graine float (24 puis 48 puis 53 bits), en gardant le comportement float pour 0, négatif et NaN (parité avec la racine CPU). Attendu au sol : disparition du damier, sommets à la précision du CPU. Vérification possible sans capture : `bValidateIndirectInstancedNoise` au sol (les texels sont comparés au CPU) et le fil de fer à hauteur d'homme.

### Retour de Benja (2026-09-02, 21:00) : « un peu mieux, mais dans les pentes il y a toujours de l'imprécision par rapport au clipmap CPU »

**Mesure (lecture seule, sur l'éditeur de Benja, root de `L_Persistent_Universe`)** : deux transects de 40 m du bruit CPU (`GetPawnDistanceFromGround`) échantillonnés à 76 cm (cellule GPU de profondeur 16) et à 3,04 m (cellule du clipmap). Résidu du profil fin par rapport à un lissage à 3 m : 0,9 à 1,7 cm en moyenne, 2,9 à 5,2 cm au maximum ; dérivée seconde 1 à 1,9 cm ; pentes de 0,04 à 0,61. Le bruit est donc lisse sous 3 m : la géométrie GPU à 76 cm ne peut pas être « rugueuse » de plus de 5 cm, et ce qui reste visible sur les pentes n'est pas la géométrie mais les **normales**.

**Cause** : `WSMeshGenerate.usf` calcule la normale par différences centrées à un texel de distance (76 cm sur les tuiles profondes) ; le clipmap CPU (`LodGenerationThread::CalculateNormal`) lisse les normales des triangles adjacents, soit des cellules de 3 m (TriangleSize) au plus près. Une ondulation de 1 à 5 cm sur 76 cm bascule la normale GPU de 1 à 4 degrés au hasard d'un sommet à l'autre : éclairage granuleux en lumière rasante et masques de pente du matériau (roche / herbe) mouchetés, précisément sur les pentes.

**Correctif écrit (C++ et shaders, à compiler par Benja)** : les normales sont échantillonnées à `NormalStepTexels` texels de distance, avec `NormalStepTexels = clamp(round(TriangleSize / taille du texel), 1, bordure)`, soit 3 m sur ce niveau quelle que soit la profondeur (4 texels à la profondeur 16, 2 à 15, 1 au-delà) ; la géométrie garde sa finesse. Pour que ces échantillons restent dans la tuile, la bordure du heightfield passe de 1 à 4 texels (`FWSHeightfieldManager::HeightfieldBorderTexels`, `WS_HEIGHTFIELD_BORDER` dans les trois shaders : génération, maillage, lecture du validateur) : textures 136 x 136 au lieu de 130 x 130, soit 9,5 % de mémoire et de génération en plus, budget automatique recalculé. Le paramètre `PaddingA` du dispatch de maillage devient `NormalStepTexels` (même taille de structure).

**Test sans build pour confirmer le diagnostic** : `Quadtree_MaxDepth = 14` sur le root de `L_Dev_Claude` donne des cellules de 3,04 m, exactement le clipmap ; si les pentes redeviennent propres, c'est bien l'échelle des normales.

Fichiers : `WSHeightfieldManager.h/.cpp`, `WSGPUTerrainRenderer.cpp`, `WSHeightfieldGenerate.usf`, `WSMeshGenerate.usf`, `WSHeightfieldSample.usf`.

### Validation de Benja (2026-09-02, soir)

Après son build C++ et shaders : « C'est bel et bien corrigé. » Les correctifs du jour sont donc vérifiés sur le produit qui tourne : crash du quadtree (référence pendante), seuil automatique, rayon de terrain sous la caméra (finesse au sol), régulateur relatif, relecture des plages après compactage, compteur de génération unique, racine carrée en double, normales à la maille du clipmap avec bordure de 4 texels.

Reste ouvert, à décider par Benja : bascule projet (`ws.GPUNoise=1` + `ws.IndirectInstancing=1`, uniquement après reconstruction du jeu et du serveur, sinon les anciens binaires activent l'ancien chemin GPU) ; roots de `L_Persistent_Universe` à LodResolution 16 (16 sommets par tuile GPU : passer à 128 ou découpler la résolution de maillage GPU) ; couture limitée à un niveau d'écart entre voisins (jonctions en T possibles, piste : jupes) ; `Quadtree_MaxDepth` au-delà de 16 si une finesse sous 76 cm est voulue (clé jusqu'à 27) ; suite QATS et build serveur non lancés ; modification de `WorldScapeRoot_Main.cpp` du 2026-09-02 à 19:05 d'origine inconnue de ce côté.

### Découplage de la résolution de maillage GPU (2026-09-02, soir, demande de Benja)

`IndirectNoise_MeshResolution` (root, catégorie « Noise|GPU Compute », 0 = LodResolution) fixe le nombre de sommets par côté d'une tuile GPU indépendamment de LodResolution, qui reste le réglage du clipmap CPU (secours et serveur dédié). Arrondi au multiple de 4, minimum 8, maximum 256. Quand `IndirectNoise_TileResolution` vaut 0, la résolution du heightfield suit la plus grande des trois valeurs (LodResolution, OceanLodResolution, résolution de maillage), sinon le maillage rééchantillonnerait plusieurs fois le même texel. Un changement déclenche la même régénération que LodResolution (`Regen Request by IndirectNoise_MeshResolution`). Usage visé : les roots de `L_Persistent_Universe` à LodResolution 16 avec un maillage GPU à 128. Écrit, à compiler par Benja. Fichiers : `WorldScapeRoot.h` (propriété et `Prev_`), `WorldScapeRoot_Main.cpp` (résolution automatique du heightfield, résolution des triangles, détection de changement à trois endroits).

### Audit matériau et draw calls (2026-09-02, soir, question de Benja)

Draw calls : chaque tuile GPU est un `FMeshBatch` à un élément (`NumInstances = 1`, arguments indirects par tuile) dans `GetDynamicMeshElements` : une commande de dessin par tuile, 300 à 450 au sol contre une cinquantaine pour le clipmap (16 LOD x 3 sections). Piste : regrouper par (index buffer de couture, résolution, slot de matériau) et dessiner en instances, la vertex factory lisant la base de sommets par instance ; une dizaine de dessins au lieu de centaines.

Matériau `M_EarthBase_OPT` (lecture seule) : 905 noeuds, PS 1487 instructions / 230 échantillons, VS 379, 0 échantillon VT (RVT à moitié câblé), 4 samplers. Aucune logique liée au clipmap : pas de TextureCoordinate, pas d'ObjectPosition / bounds / LocalPosition, pas de Custom HLSL, pas de QualitySwitch. Entrées qui dépendent du maillage : un `VertexColor` (HeightNormalize, Temperature, Humidity, Hole : même ordre sur les deux chemins, vérifié dans `WorldScapeRoot_Thread.cpp:750` et `WSHeightfieldGenerate.usf:1428`), trois `VertexNormalWS` (masques de pente), dix `VertexInterpolator`, un WPO proche ; `tangent_space_normal = False`, donc la base tangente n'influe pas. Sept `Transform World -> Local` sur des vecteurs : identiques sur les deux chemins (tous les composants sont attachés au TransformKeeper). La différence perçue entre CPU et GPU se réduit donc à la densité du maillage (masques par vertex à seuils raides) et à l'échelle des normales (corrigée).

Leviers proposés (aucun appliqué) : finir le RVT pour le lointain ; boucle de couches en Custom HLSL avec sélection des deux biomes dominants par pixel (le vrai gain sur 230 échantillons) ; tableaux de textures ; POM sur la projection dominante au plus près (hauteur déjà présente dans T_Grass_Near_ORMH), fondu à quelques dizaines de mètres ; QualitySwitch ; distances caméra-planète en scalaires par frame.

### Instancing des draw calls (2026-09-03, goal de Benja : « fais l'instancing des draw calls, tout doit être ultra performant »)

**Avant** : une commande de dessin par tuile (`FMeshBatch` à un élément, arguments indirects par tuile, `NumInstances = 1`), 300 à 450 dessins au sol, et un sommet de 112 octets (position, tangentes et couleur en float4, centre de tuile en double découpé répété dans chaque sommet).

**Après (écrit, à compiler par Benja : C++ et shaders)** :
- Sommet compacté à 32 octets (`FWSTerrainVertex`) : position float3, tangente X et normale en SNORM8x4 (w = signe de la binormale), couleur en UNORM8x4 (même quantification que le FColor du clipmap CPU), UV0 en half2. Le centre de tuile sort du sommet : il est porté par l'instance. Pool de sommets divisé par 3,5 (une tuile 128 x 128 passe de 1,8 Mo à 512 Ko) ; sous le même plafond `ws.GPUTerrain.MaxVertexPoolMB` (768) les caps de feuilles dérivés du pool (`GetVertexStrideBytes`) admettent 3,5 fois plus de tuiles.
- Vertex factory sans flux de sommets : elle lit le pool et un buffer d'instances par `ByteAddressBuffer` (fetch manuel par `SV_VertexID`), la seule déclaration restante est le flux d'identifiant de primitive de GPU Scene. Une instance = une tuile : `FWSTerrainInstanceData` (32 octets : base de sommets + centre de tuile high/low), buffer rebâti à chaque tick (`WS.GPUTerrain.Instances`).
- Un dessin par groupe (index buffer de couture, slot de matériau, sens des faces, eau) : `FDrawItem` porte `FirstInstance` et `NumInstances`, le proxy déclare `NumInstances` instances GPU Scene par batch (`FMeshBatchDynamicPrimitiveData`, transformations identité) et passe `FirstInstance` par `UserIndex` (`WSInstanceBaseOffset` dans le shader). L'indice local d'instance vaut `InstanceId - Primitive.InstanceSceneDataOffset` (stable sous le culling d'instances), ou `SV_InstanceID` sur les plateformes à uniform buffer de primitive.
- Les tuiles non prêtes ne coûtent plus rien (avant : un dessin indirect à 0 instance chacune). Plus d'arguments indirects.
- Le centre de tuile est calculé une fois sur CPU en double (`WS_ComputeTileCenter`) et envoyé à la fois au shader de maillage (uniformes `TileCenterHigh` / `TileCenterLow`) et à l'instance, donc identique bit à bit des deux côtés ; le shader de maillage perd son bloc groupshared et sa barrière.
- Mesure attendue : une dizaine de dessins au lieu de 300 à 450 ; à lire dans la ligne de stats (`DrawItems=` groupes, `Instances=`, `Inst=` Ko du buffer d'instances, `ProxyInst=`) et au `stat scenerendering` de Benja.

**À vérifier au premier lancement** : (1) la recompilation des permutations du VF (shader map) sans erreur ; (2) la validation RHI : les deux buffers sont désormais en `SRVGraphics` (ils étaient VertexOrIndexBuffer et IndirectArgs) ; (3) les validateurs décodent la couleur quantifiée (UNORM8, erreur max 1/510), la ligne `VertexColor` du log reste indicative ; (4) `ForceVertexColorView` inchangé ; (5) rendu noir : vérifier `ProxyInst=1` dans la ligne de stats (vue d'instances liée), puis `WSInstanceBaseOffset`.

Fichiers : `WSGPUTerrainVertexFactory.h` / `.cpp` (réécrits), `WSGPUTerrainVertexFactory.ush` (réécrit), `WSMeshGenerate.usf` (layout 32 octets, centre en uniforme, sans groupshared), `WSGPUTerrainRenderer.h` / `.cpp` (groupes, buffer d'instances, proxy, stats).

### Retour de Benja (2026-09-03) : « coutures chaotiques entre des tuiles de même profondeur, et des fentes qu'on entrevoit »

**Cause 1 (le symptôme de la capture)** : le masque de couture d'une feuille était calculé en partant de son PARENT : « le voisin ouest du parent, à la profondeur d-1, est-il une feuille ? ». Pour l'enfant situé du côté est de son parent, son bord ouest touche son frère (même profondeur), mais la question posée restait celle du bord ouest du parent : dès que le parent avait un voisin plus grossier à l'ouest, les deux enfants alignés recevaient le drapeau, et le bord intérieur entre frères était cousu comme s'il faisait face à une tuile grossière. D'où les bandes en zigzag sur les bords intérieurs d'un bloc 2 x 2, exactement la capture.

**Cause 2 (les fentes)** : la couture repliait les sommets d'indice local impair sur leurs voisins pairs. Or les sommets de la tuile grossière tombent sur les indices pairs comptés depuis le coin du parent ; avec une résolution paire (128, soit 127 cellules par tuile), l'enfant de la seconde moitié commence à l'indice 127, impair : ses sommets alignés sont les impairs locaux, et la couture repliait précisément les sommets alignés sur les sommets décalés. Le coin partagé par les deux frères (indice 127, exempté du repli) n'était jamais sur un sommet grossier : une fente à chaque coin cousu, un zigzag qui traverse les milieux de segments. Les bits de « flip » (sens du repli, alternés par parité de tuile) ne pouvaient pas corriger une parité d'alignement, et faisaient diverger les deux frères à leur coin commun.

**Correctif (écrit, à compiler par Benja : C++ et shader)** :
- Quadtree (`WSQuadtreeManager.cpp`) : le voisin est cherché à la profondeur de la feuille (le frère est trouvé, les changements de face du cube sont résolus à la bonne profondeur), puis ses ancêtres : parent (2:1) ou grand-parent (4:1). Le masque porte, par bord, l'écart de niveau (0, 1 ou 2 : bits 0 à 7) et la position de la feuille dans son grand-parent (X mod 4 en bits 8-9, Y mod 4 en bits 10-11). Plus de bits de flip.
- Shader (`WSMeshGenerate.usf`) : plus de repli. Un sommet de bord dont l'indice relatif au coin de l'ancêtre n'est pas multiple de l'espacement grossier (2 ou 4 cellules) est placé sur le segment entre les deux sommets grossiers qui l'encadrent (interpolation linéaire de leurs positions, échantillonnées dans le heightfield de la tuile et sa bordure de 4 texels). Il est alors exactement sur l'arête des triangles grossiers : aucune fente, aucun triangle dégénéré, aucune bande en zigzag, et les deux frères calculent le même point pour leur coin commun. Un écart de plus de deux niveaux reste sans couture (comme avant).
- Coût : deux évaluations de position supplémentaires pour les seuls sommets de bord concernés (moins de 3 % des sommets d'une tuile).

Limite : l'interpolation est exacte quand la résolution du heightfield vaut celle du maillage (défaut). Si le heightfield est plus fin que le maillage (LodResolution 16 avec un océan à 128), un échantillon hors tuile peut dépasser la bordure de 4 texels et le coin cousu s'écarte légèrement.

### Retour de Benja (2026-09-03) : « sur les flancs du globe, les tuiles disparaissent complètement quand j'arrive au sol ; au sommet tout va bien » (carte L_Persistent_Universe, root Terre à LodResolution 128, océan à 52, MaxDepth 15)

**Mesure** : le log de sa session, à l'heure exacte du test, montre le cache de heightfields épinglé au plafond souple pendant plus d'une minute (`Using over-budget allowance`, 1407 tuiles pour un budget de 1280 = 2 x 640 feuilles). Ce n'est pas une affaire de face du cube : rien dans le quadtree, le culling, le shader ou la factory ne dépend de la face, la trigonométrie en double du heightfield est saine à l'équateur. Ce sont des tuiles évincées puis régénérées trop tard.

**Cause (régression de l'instancing)** : le sommet compact (32 octets au lieu de 112) a laissé les plafonds de feuilles monter jusqu'aux valeurs configurées (512 terre + 128 océan) là où le pool de sommets les bridait à 334 + 128. L'ensemble de travail du cache (feuilles + parents gardés pour l'hystérésis + tuiles juste hors champ) dépasse alors le budget automatique « 2 fois les feuilles ». Au sol, tout est évincé en permanence, en particulier les tuiles hors champ (non demandées, donc candidates LRU) ; au demi-tour elles sont régénérées avec un retard réglé par le régulateur (1 à 4 tuiles par tick sur une image lente) et, pire, le parent (celui qui est dessiné tant que les enfants manquent) était mis en file APRÈS ses quatre enfants. Résultat : des trous de plusieurs secondes. Le sommet marchait parce que la Terre y demande moins de feuilles (pas d'océan sous les yeux) : le cache ne saturait pas.

**Correctif (écrit, à compiler par Benja, C++ seul)** :
- Budget automatique du heightfield : 4 fois les plafonds de feuilles au lieu de 2 (toujours borné à 2 Go ; `IndirectNoise_HeightfieldMemoryBudgetBytes` prime). Terre à 128 : 568 Mo au lieu de 284.
- Requêtes urgentes : une feuille dessinée cette image dont le heightfield n'est pas prêt est demandée `bUrgent` (`FWSHeightfieldTile::FrameLastUrgent`). Le manager les génère en premier (priorité 64) et au-delà du plafond adaptatif, jusqu'au plafond dur de 64 par tick.
- Ordre parent puis enfants : quand un enfant manque, le noeud demande sa propre tuile (urgente si absente) AVANT la précharge des enfants.
- Rien ne change pour le chemin CPU ni pour le clipmap GPU (`RequestTile` garde sa signature avec un paramètre par défaut).

Fichiers : `WSHeightfieldManager.h/.cpp`, `WSQuadtreeManager.cpp`, `WorldScapeRoot_Main.cpp`.

À vérifier après build, au sol sur un flanc : plus de `Using over-budget allowance` en continu dans le log, et aucun trou au demi-tour. Si le symptôme persiste, activer `bDebugIndirectInstancedNoise` sur le root et me donner la ligne `WS IndirectTerrain:` (QCull, TileReady, TileRenderable, ReadyNoMesh, HFEvict, QLeafCap).

Piste de mémoire non appliquée : la texture Aux des tuiles est en RGBA16F alors que ses quatre canaux sont des valeurs [0,1] quantifiées en UNORM8 dans le sommet ; en RGBA8 une tuile passerait de 222 Ko à 148 Ko.
