# PlanetScape — Architecture

Plugin UE5.7 de génération procédurale GPU de planètes sphériques (cube-sphere + quadtree LOD). Huit modules, ~137 fichiers source, zéro réplication réseau. **Statut : autonome — non activé dans `QANGA.uproject` et référencé nulle part ailleurs dans le code QANGA (aucun consommateur externe au moment de la rédaction).**

> Note de portée : ce plugin n'a aucune dépendance runtime sur WorldScape. Les seules mentions « WorldScape » sont algorithmiques (`WorldscapeTerrain.ush` est un *portage GPU* de l'algorithme `PlanetTerraNoise` de WorldScape) ou conventionnelles (le filtre `SpawnInWater` du foliage suit le motif `FWorldScapeFoliagesContraint`). Voir §6.

---

## 1. But d'usage

PlanetScape résout un problème que ni le `Landscape` ni WorldScape d'Unreal ne couvrent nativement : afficher une **planète sphérique entière** (de 1 km à ~10 000 km de rayon) avec terrain procédural détaillé jusqu'à l'échelle FPS au sol, **sans construction de mesh par tuile** sur le CPU et **sans perte de précision float32** aux rayons planétaires.

Concrètement, le plugin fournit :

- Un acteur unique `APlanetScapeActor` qu'on pose dans le niveau : il devient une planète visible immédiatement dans le viewport éditeur (sans Play/Simulate).
- Un terrain généré entièrement sur GPU (compute shaders), affiché via **un seul `InstancedStaticMeshComponent` par face de cube** dont le matériau lit la hauteur depuis un `Texture2DArray` partagé et fait la projection cube→sphère + le displacement en World-Position-Offset.
- Un quadtree LOD à erreur écran (screen-space error) avec horizon culling, pour ne raffiner que ce que la caméra voit.
- Des sous-systèmes optionnels cohérents avec l'état de la planète : océan FFT (Tessendorf), nuages volumétriques, foliage en fenêtre glissante, heightmaps importées, sculpt/paint/flatten éditeur.
- Une collision physique par tuile (heightfield cuit hors-Nanite), découplée du rendu.

Le tout vise un contrat de précision strict : le foliage, la collision et les requêtes de hauteur CPU passent **exactement** par le même `HeightFieldCS` que le terrain rendu, donc les positions sont identiques au pixel près par construction (pas de divergence terrain/foliage).

---

## 2. Vue d'ensemble / décisions d'architecture

### Forme générale

```
   ┌─────────────────────────────────────────────────────────────┐
   │  APlanetScapeActor (PlanetScape, orchestrateur, GameThread)  │
   │    Tick : UpdateCameraState → Scheduler.Update →            │
   │           ProcessRetires → DriveGPUDispatches →             │
   │           ConsumeGPUResults → ProcessPendingCollision →     │
   │           UpdateMeshVisibility → Cloud/Foliage/Ocean tick   │
   └───┬───────────────┬────────────────┬──────────────┬─────────┘
       │               │                │              │
  FTileScheduler  FTerrainCompute   FInstancedTile  sous-systèmes
  (quadtree LOD)   Dispatcher (GPU)  Renderer (ISM)  (HM/Sculpt/Ocean/
                                                       Foliage/Clouds)
```

### Découpage en modules (PostConfigInit pour le bas niveau GPU)

| Module | Type | LoadingPhase |
|---|---|---|
| `PlanetScapeCore` | Runtime | PostConfigInit |
| `PlanetScapeGPU` | Runtime | PostConfigInit |
| `PlanetScapeHM` | Runtime | PostConfigInit |
| `PlanetScapeFoliage` | Runtime | PostConfigInit |
| `PlanetScapeClouds` | Runtime | PostConfigInit |
| `PlanetScapeOcean` | Runtime | Default |
| `PlanetScape` | Runtime | Default |
| `PlanetScapeEditor` | Editor | Default |

Le découpage évite les dépendances circulaires : `PlanetScape` (haut niveau) dépend de tous les autres ; `PlanetScapeOcean` ne dépend **pas** de `PlanetScape` (l'acteur orchestre l'océan depuis son `Tick`, pas l'inverse — commenté explicitement dans `PlanetScapeOcean.Build.cs`).

### Décisions clés (et ce qui est *réellement* implémenté)

1. **Un seul chemin de rendu : `InstancedGPUOnly`.** L'enum `EHybridRenderMode` ne contient plus qu'une valeur. Les anciens modes `NaniteOnly` et `InstancedGPU` (hybride Nanite-proche / ISM-loin) ont été **supprimés en V9** ; `APlanetScapeActor::PostLoad` ramène toute valeur sauvegardée héritée vers `InstancedGPUOnly`. Les noms `FHybridRenderSettings`, `ETileRenderPath::Instanced` (enum à une valeur) sont conservés uniquement pour la compatibilité de sauvegarde. **Le doc reflète donc un système single-path, malgré la nomenclature « Hybrid ».**

2. **Le heightfield reste sur GPU.** Pour le rendu, aucune lecture CPU (`readback`) du heightfield n'a lieu : `HeightFieldCS` écrit directement dans une slice du `Texture2DArray`, et le matériau WPO l'échantillonne. Le readback de vertices n'existe que pour les chemins qui en ont besoin : foliage, collision, cache de hauteur CPU.

3. **Précision LWC manuelle.** Aux rayons planétaires, `SDir * (R + HD)` perd en float32. Le renderer repositionne l'ISM à `CamDir * R` (calculé en double CPU) et le shader calcule `(SDir - CamDir) * R + SDir * HD`, gardant les valeurs intermédiaires petites près de la caméra (voir `FInstancedTileRenderer::UpdateCameraDirection`). Les requêtes utilisent une décomposition double-float (`RebaseOrigin` / `RebaseOriginLow`, `PlanetRadiusCM` / `PlanetRadiusCMLow`).

4. **Adressage entier des tuiles.** `FTileAddress` (Face/Depth/Col/Row) se compacte dans un `uint64` servant de clé de `TMap`. Pas d'égalité flottante, dérivation parent/enfant O(1).

5. **Tout le scheduling sur le GameThread, toute la RHI sur le RenderThread.** `FTileScheduler` et `FOceanQuadtree` tournent intégralement sur le GameThread ; `FTerrainComputeDispatcher`, `FOceanFFTDispatcher`, `FPlanetSculptManager` font toute création/Lock/Unlock RHI exclusivement sur le RenderThread, communiquant par `FThreadSafeBool` et `FRHIGPUBufferReadback`.

6. **Cohérence terrain/foliage/collision par réutilisation du même shader.** `DispatchFoliageHeights` réutilise `HeightFieldCS` avec les mêmes uniformes (HM + Mountain Enhance + Sculpt + bruit procédural) → H_final bit-identique au terrain rendu.

---

## 3. Carte des composants

### Types fondamentaux (`PlanetScapeCore`)

| Élément | Type | Rôle |
|---|---|---|
| `FTileAddress` | struct | Adresse de tuile cube-sphere (Face/Depth/Col/Row) packée en `uint64`, navigation hiérarchie, bornes UV de face |
| `SphereMath` | namespace | Projection cube→sphère équiaire (Everitt), inverse `DirToFaceUV` (Newton 4 pas), erreur écran, horizon culling, calcul de `MaxDepth` |
| `PlanetScapeFaces` | namespace | Vecteurs Up/Right/Forward des 6 faces de cube |
| `EPatchResolution` | UENUM | Résolution de grille de patch (32/64/96/128) |
| `ETileLifecycle` | UENUM | Dormant → PendingDispatch → GPUInFlight → PendingMeshBuild → Active |
| `ETileRenderPath` | UENUM | Une valeur (`Instanced`) — conservé pour stabilité de type |
| `EHybridRenderMode` | UENUM | Une valeur (`InstancedGPUOnly`) |
| `FNoiseLayer` | USTRUCT | Couche de bruit FBM (amplitude, fréquence, lacunarité, octaves, persistance, domain warp) |
| `FLODProfile` | USTRUCT | Seuils LOD (ScreenErrorThreshold, MaxDepth, MaxDispatchesPerFrame, TargetTriangleSizeM, MaxTileCount) |
| `FHybridRenderSettings` | USTRUCT | Réglages ISM (MaxInstancedSlots, InstancedSkirtDepthCM) |
| `FPlanetBiomeSettings` / `FPlanetMaterialSettings` | USTRUCT | Seuils de distribution de biome (neige/désert/roche/terre/herbe/rivage/océan) |
| `FPlanetRenderSettings` / `FPlanetCollisionSettings` | USTRUCT | Ombres/canaux de lumière/normales HQ ; collision (bGenerateCollision, CollisionStartDepth) |
| `FBiomeStampLayer` (`PlanetBiomeStampTypes.h`) | USTRUCT | Couche de stamp heightmap par biome (style Star Citizen) |

### Orchestration et rendu (`PlanetScape`, `PlanetScapeGPU`)

| Élément | Type | Rôle |
|---|---|---|
| `APlanetScapeActor` | UCLASS (AActor) | Orchestrateur unique : possède tous les sous-systèmes, drive le pipeline par frame |
| `FTileScheduler` | classe (GameThread) | Registre de tuiles, décisions subdivide/merge à erreur écran, file de dispatch priorisée, prefetch prédictif, file de retire |
| `FTileRecord` / `FTileScheduleEntry` | struct | État runtime par tuile ; entrée de file de priorité |
| `FTerrainComputeDispatcher` | classe (RenderThread RHI) | Ring buffer de slots compute ; `DispatchInstanced` (terrain), `DispatchFoliageHeights` (foliage) ; readback async |
| `FTerrainVertex` / `FTileComputeRequest` / `FInstancedDispatchResult` | struct | Layout vertex GPU ; requête de dispatch (rayon, bruit, HM, mountain, sculpt, pads, stamps, biome) ; résultat |
| `FInstancedTileRenderer` | classe | Un ISM par face + mesh-grille partagée + atlas `Texture2DArray` (height/normal/weight/paintmask/stamptint) + MID de displacement |
| `FInstancedTileParams` | struct | Données custom par instance (19 floats : face, UV, rayon, slice, dir, offset, …) |
| `UTileHeightfieldCollision` | UCLASS (UPrimitiveComponent) | Collision physique par tuile, cuite hors-Nanite depuis le grid de readback ; invisible |

### Heightmap / sculpt / paint (`PlanetScapeHM`)

| Élément | Type | Rôle |
|---|---|---|
| `FPlanetHMManager` | classe | Cache CPU + ressources GPU des heightmaps importées (cube-face ou équirectangulaire) |
| `FPlanetSculptManager` | classe | 6 textures R16F (delta d'élévation) ; brushes sculpt/smooth/flatten/noise sur RenderThread |
| `FPlanetFlattenPadManager` | classe | Liste de pads analytiques (plateaux géométriques) évalués per-pixel par le shader, scale-independent ; cap `kMaxPadsPerTile = 16` |
| `FPlanetTileWeightManager` | classe | Textures de poids RGBA8 par tuile (256²) pour le paint de couches, store CPU sparse |
| `EPlanetHMSourceType`, `FPlanetHMSettings`, `FFlattenPad`, `FPlanetSculptSettings`, `FPlanetSurfaceLayer` | UENUM/USTRUCT | Types de config HM / pad / sculpt / couche de surface |

### Océan (`PlanetScapeOcean`, shaders dans `PlanetScapeGPU`)

| Élément | Type | Rôle |
|---|---|---|
| `FOceanQuadtree` | classe (GameThread) | Quadtree LOD de la surface océanique (rayon = R + niveau océan, MaxDepth décalé) |
| `FOceanTileRenderer` | classe | ISM des tuiles océan (pas de dispatch GPU de hauteur, surface analytique + FFT) |
| `FOceanFFTDispatcher` | classe (RenderThread) | Simulation de vagues FFT Tessendorf (H0 JONSWAP → évolution → IFFT butterfly 256² RGBA16F) |
| `FOceanEvolveCS`, `FOceanButterflyCS`, `FOceanCombineCS` | FGlobalShader | Passes de la FFT océan (`OceanFFTShaders.h`) |
| `FOceanSettings`, `FOceanWaveSettings`, `FOceanQuadtreeSettings` | USTRUCT | Config océan / vagues / quadtree (`OceanTypes.h`) |

### Foliage / nuages

| Élément | Type | Rôle |
|---|---|---|
| `FPlanetFoliageSubsystem` | classe (`PlanetScapeFoliage`) | Fenêtre glissante N×N de secteurs, état PendingDispatch→…→Live, HISM, hauteurs via GPU |
| `FPlanetGrassSectorRenderer` / `FPlanetGrassSectorBuilder` | classe | Rendu HISM / build async des instances |
| `EPlanetGrassSectorState`, `FPlanetGrassSectorEntry`, `FPlanetBiome`, `FPlanetGrassSettings` | enum/struct | État de secteur ; biome (matériau + meshes + règles de placement) |
| `UPlanetCloudComponent` | UCLASS (`PlanetScapeClouds`) | Contrôleur qui drive un `UVolumetricCloudComponent` frère (composition, pas héritage) |
| `FPlanetCloudSettings` / `FPlanetCloudLayer` | USTRUCT | Réglages de nuages volumétriques |

### Éditeur (`PlanetScapeEditor`)

| Élément | Type | Rôle |
|---|---|---|
| `FPlanetScapeEdMode` | classe (FEdMode) | Mode éditeur calqué sur `FEdModeLandscape` (ToolMode → Tool → Brush) |
| `IPlanetScapeTool` / `IPlanetScapeBrush` | interface | Outils (Sculpt/Smooth/Flatten/Paint/Erase/Noise/Visibility) et brushes (Circle/Alpha/Pattern) |
| `UPlanetScapeEditorObject`, `UPlanetModeSettings` (+ dérivés Ocean/…) | UCLASS | Objets de réglage pour les panneaux Slate (`SPlanet*Panel`) |
| `UPlanetScapeEditorToolbarSettings` | UDeveloperSettings | Config éditeur persistante (Editor Preferences > Plugins > PlanetScape) |

### Compute shaders (`/PlanetScapeShaders`, mappé runtime)

| Fichier | Rôle |
|---|---|
| `TerrainGenerator.usf` | `HeightFieldCS` (Pass 1, hauteur → slice ou buffer) + `VertexSolverCS` (Pass 2, vertices/normales/moyenne) |
| `PlanetScapeCommon.ush`, `BiomeWeights.ush`, `MacroVariation.ush`, `NextGenSurface.ush`, `NextGenTerrain.ush` | Primitives de bruit/biome/surface partagées |
| `WorldscapeTerrain.ush` | Portage GPU de l'algorithme `PlanetTerraNoise` de WorldScape (signé [-1,+1]) |
| `SculptBrush.usf`, `SmoothBrush.usf`, `FlattenBrush.usf`, `NoiseBrush.usf`, `PaintBrush.usf` | Brushes éditeur compute |
| `OceanEvolve.usf`, `OceanButterfly.usf`, `OceanCombine.usf`, `OceanSea.ush` | FFT océan |

---

## 4. Flux de données et cycle de vie

### Initialisation

1. **Démarrage module.** `FPlanetScapeGPUModule::StartupModule` enregistre le chemin shader virtuel `/PlanetScapeShaders` → `Shaders/` via `AddShaderSourceDirectoryMapping` (et **non** dans le `.Build.cs`). `FPlanetScapeModule::StartupModule` force la CVar `r.SkyAtmosphere.AerialPerspectiveLUT.FastApplyOnOpaque` à `0` (rendu d'atmosphère per-pixel correct à l'échelle planétaire), via `OnFEngineLoopInitComplete` + tentative immédiate (hot-reload).

2. **Placement de l'acteur.** `APlanetScapeActor::PostRegisterAllComponents` lie les délégués `BeginPIE`/`EndPIE`. L'acteur tique même en viewport éditeur (`ShouldTickIfViewportsOnly() == true`, `bStartWithTickEnabled == true`).

3. **`InitialiseSystems()`** (déclenché par `bNeedsRebuild` au début du `Tick`, ou par `RebuildTerrain`) : initialise `FTileScheduler` (6 tuiles racines de face), `FTerrainComputeDispatcher`, `FInstancedTileRenderer` (mesh-grille partagée + atlas `Texture2DArray`), `FPlanetHMManager`, `FPlanetSculptManager`, `FPlanetTileWeightManager`, et conditionnellement océan/foliage/nuages. `PostLoad` migre les champs hérités (`SurfaceLayers_DEPRECATED`, `GrassLayerMappings_DEPRECATED` → `Biomes[]`) et clampe `EHybridRenderMode`.

### Boucle par frame (`APlanetScapeActor::Tick`, TG_PrePhysics)

1. `UpdateCameraState` — caméra (position, vélocité pour prefetch, FOV, hauteur écran).
2. `Scheduler.Update(...)` — recalcule l'arbre LOD : erreur écran terrain-aware (correction d'altitude), horizon cull, décisions subdivide/merge avec hystérésis (EMA `SmoothedScreenError`), file de dispatch priorisée, éviction si le budget de slices ISM (`MaxInstancedSlots`) est saturé. Les caches LOD par tuile sont indexés sur `ScoringStamp` (caméra immobile → réutilisation, pas de recompute sur 2000+ tuiles).
3. `ProcessRetires` — libère slices/instances ISM des tuiles fusionnées **avant** les dispatches (réutilisation même frame).
4. `DriveGPUDispatches` — consomme la file de dispatch, construit un `FTileComputeRequest` par tuile (`BuildRequest`), appelle `DispatchInstanced`. Le ring buffer (`InstancedRingSize = 32`) borne le travail en vol.
5. `ConsumeGPUResults` — `CollectInstancedResults` retourne les slots dont le readback de moyenne est prêt ; allocation de slice, ajout d'instance ISM (`AddTileInstance`), passage `GPUInFlight → PendingMeshBuild → Active`, remplissage du cache CPU de hauteur (`CacheTileHeightsFromVertices`), calcul des poids de biome.
6. `ProcessPendingCollision(2)` — cuit la collision de ≤2 tuiles/frame (coût PhysX ~5-15 ms/tuile étalé).
7. `UpdateMeshVisibility` + `FlushDirtyInstanceData` — coalesce toutes les mutations de custom-data ISM en **un** `MarkRenderStateDirty` par face.
8. Tick océan (quadtree → ISM → FFT → bind textures), foliage (fenêtre glissante), nuages (`SyncWithPlanet`).

### Rendu (sans CPU par tuile)

Le matériau de displacement (un `UMaterial` généré, MID partagé) lit, par instance ISM : la `FaceIndex`, l'origine/étendue UV, le rayon, la slice heightfield, la direction/offset de centre de tuile. Il échantillonne le `Texture2DArray` de hauteur, projette cube→sphère, applique le displacement en WPO, et échantillonne l'atlas de normales (calculées par `VertexSolverCS`, pas en différence centrale per-pixel). Skirts de 1 vertex pour masquer les T-junctions entre LOD.

### Teardown

`TeardownSystems` / `EndPlay` : `Reset`/`Shutdown` de chaque sous-système (libération RHI sur RenderThread), vidage des caches (`TileHeightCache`, `TileCollisionMap`, `InstancedSliceMap`), retrait des délégués PIE. Le `ShutdownModule` de `PlanetScape` ne restaure **pas** la CVar d'atmosphère (choix délibéré).

---

## 5. Réplication / réseau

**Aucune.** Le plugin ne réplique rien : pas de `GetLifetimeReplicatedProps`, `DOREPLIFETIME`, RPC `Server_`/`Client_`/`Multicast_`, ni `SetReplicates`. La génération de terrain est purement déterministe (mêmes `Seed` + paramètres → même planète sur toutes les machines), donc chaque client peut régénérer localement sans synchronisation. `APlanetScapeActor` est un acteur de rendu/scène non répliqué.

> Conséquence pour une éventuelle intégration QANGA multijoueur : comme pour d'autres systèmes meshless du projet, un serveur dédié sans meshes ne résoudra pas la collision/grounding côté serveur. La collision (`UTileHeightfieldCollision`) n'existe que là où le quadtree a généré des tuiles, ce qui est piloté par la caméra locale — donc absente sur un serveur dédié headless. Toute logique de gameplay dépendante du sol devrait s'appuyer sur la hauteur **analytique** (`SampleSurfaceRadiusCM`, qui lit le cache CPU peuplé par le readback) plutôt que sur des traces, et garder à l'esprit que ce cache n'est peuplé que pour les tuiles que la caméra locale a fait générer.

---

## 6. Points d'intégration

### Dépendances (entrantes)

- **Plugin** : `GeometryProcessing` (déclaré dans le `.uplugin`).
- **Modules engine** : `Core`, `CoreUObject`, `Engine`, `RenderCore`, `RHI`, `Renderer`, `MeshDescription`, `StaticMeshDescription`, `PhysicsCore`, `Foliage` (HISM), `Projects` (IPluginManager pour le chemin shader). Côté éditeur : `UnrealEd`, `LevelEditor`, `EditorFramework`, `LandscapeEditor` (réutilisation d'icônes), `PropertyEditor`, `DetailCustomizations`, `ToolMenus`, `EditorStyle`, `DeveloperSettings`, `ApplicationCore`.
- **Pas de dépendance sur WorldScape.** `WorldscapeTerrain.ush` est un portage GPU autonome de l'algorithme `PlanetTerraNoise` (construit sur `ValueNoise3D`/FBM de `PlanetScapeCommon.ush`, sans bibliothèque externe). Le foliage emprunte le *motif* de contrainte d'eau de WorldScape (`SpawnInWater` ~ `FWorldScapeFoliagesContraint`). `UPlanetScapeEditorToolbarSettings` remplace explicitement le motif SaveGame de WorldScape par `UDeveloperSettings`. Aucun de ces points n'introduit de couplage runtime.

### Dépendants (sortants)

Aucun au moment de la rédaction. Recherche exhaustive sur `G:/QANGA/Source` et `G:/QANGA/Plugins` (hors plugin) : **zéro référence** à `APlanetScapeActor`, `SampleSurfaceRadiusCM`, `AddFlattenPad`, etc. Le plugin **n'est pas activé** dans `QANGA.uproject`.

> Le header `PlanetScapeActor.h` documente un consommateur prévu nommé `RTSScapePlanetNavBridge` (placement de bâtiment / sampling de nav-grid via `SampleSurfaceRadiusCM`). Ce type **n'existe pas** dans le dépôt actuel — c'est de l'intention de design, pas du code livré. `SampleSurfaceRadiusCM` est l'API publique stable conçue pour ce branchement futur (elle reste peuplée en PIE et en jeu, pas seulement en éditeur).

### API publique notable pour une intégration gameplay

| API | Usage prévu |
|---|---|
| `APlanetScapeActor::SampleSurfaceRadiusCM(Face, UV, OutRadiusCM[, OutDepth])` | Hauteur de surface finale (HM + bruit + mountain + sculpt) au pixel près, depuis le cache CPU de readback |
| `AddFlattenPad / RemoveFlattenPad / UpdateFlattenPad / ClearAllFlattenPads` | Plateaux analytiques runtime : fondations RTS, cratères d'explosion, futures routes/decals |
| `RebuildTerrain / RebuildOcean / RebuildFoliage`, `GetActiveTileCount`, `GetInFlightDispatchCount`, `GetInstancedTileCount` | Contrôle/diagnostic Blueprint |
| `SphereMath::DirToFaceUV` | Inverse exact de la projection terrain — indispensable pour que tout système externe (nav, foliage) atterrisse aux mêmes positions monde que les tuiles |

### CVars / settings

- CVar engine forcée par le plugin : `r.SkyAtmosphere.AerialPerspectiveLUT.FastApplyOnOpaque = 0` (priorité `ECVF_SetByCode`).
- Chemin shader virtuel : `/PlanetScapeShaders`.
- Le plugin **ne déclare aucune CVar console propre** (`FAutoConsoleVariable`/`FAutoConsoleCommand`).
- Réglages persistants : `UPlanetScapeEditorToolbarSettings` (UDeveloperSettings, éditeur uniquement).

---

## 7. Gotchas, invariants et pièges

1. **Single render path — ne pas réintroduire de mesh par tuile.** Tout est ISM + heightfield-sur-GPU. Les noms `Hybrid`/`Nanite`/`RenderPath` sont des fossiles de sauvegarde. Construire un `UStaticMesh` par tuile fut « a stability and maintenance liability » (commentaire `PlanetTypes.h`).

2. **Frontière de threads stricte.** Toute RHI (création de texture, Lock/Unlock, readback) **doit** être sur le RenderThread ; tout le scheduling sur le GameThread. Les passerelles sont `FThreadSafeBool bDataReady` et `FRHIGPUBufferReadback`. Ne jamais toucher un `FInstancedComputeSlot::VertexReadback`/`AverageReadback` depuis le GameThread.

3. **Précision : ne pas simplifier en `SDir * (R + HD)`.** À grand rayon, float32 craque. Garder la décomposition double-float (`RebaseOrigin`/`RebaseOriginLow`, `PlanetRadiusCM`/`PlanetRadiusCMLow`) et le repositionnement ISM caméra-relatif. La LOD elle-même calcule la distance en double (`ComputeScreenSpaceError`) pour éviter le jitter LOD à 100 km.

4. **Cohérence terrain/foliage/collision = même shader.** Le foliage **doit** passer par `DispatchFoliageHeights` (réutilise `HeightFieldCS`), jamais ré-évaluer la hauteur côté CPU avec une formule parallèle — sinon divergence de placement. `BuildFoliageHeightRequest` règle `Address.Depth` assez profond pour saturer le blend HM-vs-procédural.

5. **`SampleSurfaceRadiusCM` ne couvre que les tuiles que la caméra a fait générer.** Le cache (`TileHeightCache`) est peuplé par `ConsumeGPUResults` et purgé par `ProcessRetires`. Une direction dont aucune tuile active n'existe (face opposée, juste après spawn) renverra `false`. Le foliage gate d'ailleurs sur une profondeur LOD minimale via la surcharge `OutDepth`.

6. **Collision étalée et poolée.** Cuire toutes les collisions d'un burst de tuiles en une frame stalle le GameThread. `ProcessPendingCollision(2)` limite à ~2 tuiles/frame. Les composants `UTileHeightfieldCollision` sont poolés (`MaxCollisionPoolSize = 64`) pour éviter le churn UObject lors des subdivisions/merges rapides.

7. **Budget de slices = mur dur.** `MaxInstancedSlots` borne le nombre de tuiles concurrentes (1 slice = 1 tuile active). Saturation → le scheduler évince les tuiles Active de plus basse priorité. `MaxTileCount = 0` (auto depuis le rayon) est recommandé ; le mettre trop bas provoque du force-merge constant.

8. **`MaxDepth` est plafonné à 18** (`ComputeEffectiveMaxDepth`). Le chemin legacy (demi-arc 200 cm, ignorant la résolution de grille) produisait MaxDepth=18 + ~38K tuiles + spam de force-merge sur grandes planètes. Préférer `TargetTriangleSizeM` (défaut 2 m).

9. **Nuages : composition, pas héritage, et Mobility Movable obligatoire.** `UVolumetricCloudComponent` est MinimalAPI → impossible à sous-classer hors-Engine. Le `VolumetricCloud` frère **doit** être `EComponentMobility::Movable`, sinon `AreDynamicDataChangesAllowed()` renvoie false et chaque `Set*` runtime (MID, rayon, altitudes) est silencieusement ignoré → nuages invisibles. Requiert SkyAtmosphere + DirectionalLight (Atmosphere Sun Light) dans le niveau.

10. **Persistance éditeur des viewports non-realtime.** Après une invalidation de sculpt, `SculptRefreshFramesRemaining` force les viewports à continuer de tiquer pour que le scheduler re-dispatche, sinon les tuiles restent Dormant indéfiniment quand l'input s'arrête.

11. **Stamps de biome déclarés à plat sur l'acteur.** `bUseBiomeStamps`/`BiomeStampArray`/`BiomeStampLayers` sont sur l'acteur (pas dans un sous-struct) car l'éditeur UE5 grise le bouton « + » des `TArray<FStruct>` imbriqués.

12. **Migration de sauvegarde.** `PostLoad` migre `SurfaceLayers_DEPRECATED` / `GrassLayerMappings_DEPRECATED` vers `Biomes[]` et clampe les modes hérités. Ne pas supprimer ces champs `_DEPRECATED` tant qu'un projet sauvegardé les référence.

13. **L'index de biome est le canal de poids.** `Biomes[0]→R … [3]→A` ; au-delà de 4 entrées, un biome ne contribue plus au blend matériau RGBA mais peut encore générer du foliage via `WeightThreshold`.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `PlanetScape.uplugin` | 8 modules, dépendance `GeometryProcessing` |
| `Source/PlanetScapeCore/Public/TileAddress.h` | Adressage de tuile `uint64` |
| `Source/PlanetScapeCore/Public/SphereMath.h` | Projection cube-sphère, LOD, horizon, `DirToFaceUV` |
| `Source/PlanetScapeCore/Public/PlanetTypes.h` | Enums + structs de réglage (LOD, render, biome, collision) |
| `Source/PlanetScape/Public/PlanetScapeActor.h` / `Private/PlanetScapeActor.cpp` | Orchestrateur (~3800 lignes de cpp) |
| `Source/PlanetScape/Public/TileScheduler.h` / `Private/TileScheduler.cpp` | Quadtree LOD GameThread |
| `Source/PlanetScape/Public/TileHeightfieldCollision.h` / `.cpp` | Collision physique par tuile |
| `Source/PlanetScape/Public/PlanetMaterialBuilder.h` / `.cpp` | Construction du matériau de surface |
| `Source/PlanetScape/Private/PlanetScapeModule.cpp` | Force la CVar d'atmosphère |
| `Source/PlanetScapeGPU/Public/TerrainDispatcher.h` / `Private/TerrainDispatcher.cpp` | Dispatcher compute (ring buffer, readback) |
| `Source/PlanetScapeGPU/Public/InstancedTileRenderer.h` / `Private/InstancedTileRenderer.cpp` | ISM par face + atlas Texture2DArray + MID |
| `Source/PlanetScapeGPU/Public/OceanFFTShaders.h` / `Private/OceanFFTShaders.cpp` | FGlobalShaders FFT océan |
| `Source/PlanetScapeGPU/Private/PlanetScapeGPUModule.cpp` | Mapping `/PlanetScapeShaders` |
| `Source/PlanetScapeHM/Public/PlanetHMManager.h` / `PlanetSculptManager.h` / `PlanetFlattenPadManager.h` / `PlanetTileWeightManager.h` | HM, sculpt, pads, poids |
| `Source/PlanetScapeOcean/Public/OceanQuadtree.h` / `OceanTileRenderer.h` / `OceanFFTDispatcher.h` | Océan (quadtree, ISM, FFT) |
| `Source/PlanetScapeFoliage/Public/PlanetFoliageSubsystem.h` (+ SectorBuilder/Renderer) | Foliage fenêtre glissante / HISM |
| `Source/PlanetScapeClouds/Public/PlanetCloudComponent.h` | Contrôleur de nuages volumétriques |
| `Source/PlanetScapeEditor/Public/PlanetScapeEdMode.h` (+ `SPlanet*Panel`, `Tools/`, `Brushes/`) | Mode éditeur style Landscape |
| `Shaders/Private/TerrainGenerator.usf` | `HeightFieldCS` + `VertexSolverCS` |
| `Shaders/Private/WorldscapeTerrain.ush` | Portage GPU de `PlanetTerraNoise` (WorldScape) |
| `Shaders/Private/Ocean*.usf` / `*.ush` | FFT océan + surface |
| `Shaders/Private/{Sculpt,Smooth,Flatten,Noise,Paint}Brush.usf` | Brushes éditeur compute |
