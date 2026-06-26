# RzIndirectInstancingPlugin — Architecture

Plugin de rendu massif expérimental (`VersionName: 6.0`, `Category: Other`). Il regroupe **six modules** de R&D autour d'un thème commun : générer et afficher d'énormes quantités de géométrie procédurale (voxels, cubes, terrains planétaires) sans passer par le pipeline `UStaticMesh`/`UInstancedStaticMeshComponent` classique. Trois voies distinctes coexistent : un **vertex factory GPU-driven indirect** (module `IndirectInstancing`), un **Nanite construit à l'exécution** (`RuntimeNanite`), et un **terrain planétaire quadtree** bâti par-dessus le Nanite runtime (`RzPlanet`).

> **État réel (à lire avant tout).** Au moment de la rédaction, le plugin n'est **référencé par aucun module C++ de `G:/QANGA/Source`, aucun autre `.uplugin`/`.Build.cs` du projet, et aucun asset de `Content/`**. Aucune classe ne se réplique. Il s'agit d'un banc d'essai technique autonome (toolbox éditeur + acteurs de test placés à la main), pas d'un système intégré au gameplay shipping de QANGA. La documentation ci-dessous décrit ce que le code fait réellement, pas une intention d'intégration.

---

## 1. But d'usage

QANGA affiche des planètes entières. Les approches standard d'Unreal pour de la géométrie massive ont chacune un mur :

- `UInstancedStaticMeshComponent`/`HISM` poussent les transforms d'instances depuis le CPU et culling/LOD restent largement pilotés côté CPU ;
- un terrain de planète à pleine résolution dépasse les budgets mémoire et de build d'un `UStaticMesh` Nanite cuit hors-ligne.

Ce plugin explore trois réponses, toutes orientées « le GPU/Nanite fait le gros du travail, le CPU ne fait qu'amorcer » :

1. **`IndirectInstancing`** — un `FVertexFactory` custom (`FRzIndirectInstancingMeshVertexFactory`) qui dessine un maillage via `DrawIndexedPrimitiveIndirect`, où le **buffer d'instances et les arguments indirects sont générés et cullés entièrement par des compute shaders** sur le render thread (génération de grille + frustum culling GPU par vue, y compris les passes d'ombre). Le CPU n'envoie jamais la liste d'instances.
2. **`RuntimeNanite`** — construit des `Nanite::FResources` **à l'exécution** à partir de géométrie CPU arbitraire (`FLiveNaniteManager`/`FRzNaniteBuilder`), avec rebuild asynchrone par chunk, pour bénéficier du LOD/cull Nanite sur du contenu généré dynamiquement (impossible avec un Nanite cuit).
3. **`RzPlanet`** — un manager de quadtree sphérique (cube-sphere 6 faces, sélection par erreur écran) qui produit des chunks de terrain et délègue leur rendu à `RuntimeNanite`.

`RzBenchmark` (éditeur) chronomètre ces voies, et `IndirectInstancingEditor` fournit l'onglet « Rz Instancing Toolbox » pour les piloter à la main.

---

## 2. Vue d'ensemble / décisions d'architecture

### Les six modules (`.uplugin`)

| Module | Type | LoadingPhase | Rôle |
|---|---|---|---|
| `RzIndirectInstancingPlugin` | Runtime | Default | Module-coquille. Son `StartupModule`/`ShutdownModule` ne font qu'un `UE_LOG`. Dépend publiquement de `IndirectInstancing` et `RzPlanet` — sert d'agrégateur/point d'entrée nominal. |
| `IndirectInstancing` | Runtime | PostConfigInit | Cœur du vertex factory GPU-driven + meshing greedy de voxels. Mappe le répertoire shader virtuel `/IndirectInstancingShaders`. |
| `RuntimeNanite` | Runtime | PostConfigInit | Construction de `Nanite::FResources` à l'exécution + scene proxy Nanite live. |
| `RzPlanet` | Runtime | Default | Terrain planétaire quadtree, consommateur de `RuntimeNanite`. |
| `RzBenchmark` | Editor | Default | Runner de benchmark `UObject` (mesure les voies RuntimeNanite/Greedy). |
| `IndirectInstancingEditor` | Editor | Default | Toolbox Slate (onglet nomad + entrée menu Window). |

### Décisions clés

- **GPU-driven indirect (module `IndirectInstancing`).** Le `FRzIndirectInstancingSceneProxy` n'émet aucune instance côté CPU. Dans `GetDynamicMeshElements`, il alloue un `FMeshBatch` avec `NumPrimitives = 0` et un `IndirectArgsBuffer`, puis enregistre du travail (`AddWork`) auprès d'une extension de renderer globale. Le buffer d'arguments est rempli par compute shaders à chaque frame.
- **Extension de renderer hors-proxy.** `FRzIndirectInstancingRendererExtension` (un `TGlobalResource`) s'accroche à `GetPreRenderDelegateEx()` / `GetPostRenderDelegateEx()`. Toute la chaîne compute (`InitBuffers` → `AddInstances` → `Cull`) tourne dans `SubmitWork` au début de frame via un `FRDGBuilder`. Découple la génération d'instances du cycle de vie d'un proxy individuel et permet de partager des buffers entre proxies/vues.
- **Culling par vue, ombres comprises.** `AddWork` est appelé une fois par vue visible (vue principale ET frusta d'ombre) ; chaque vue obtient sa propre passe `CullInstancesCS` avec ses 5 plans de frustum transformés en espace local du proxy.
- **Manual vertex fetch.** Le VF déclare `MANUAL_VERTEX_FETCH=1` : positions/tangentes/UV/couleurs sont lus depuis des SRV (cf. `FRzIndirectInstancingMeshUniformParameters`), pas via des streams classiques. Seul le flux de position + le PrimitiveId stream sont déclarés comme éléments de vertex declaration.
- **Greedy meshing CPU.** En amont du rendu, `FGreedyMeshGenerator` voxelise un chunk (bruit Perlin 3D, masques 64-bit, fusion de quads par direction) et produit un `FGreedyMeshData`. Le résultat est empaqueté dans un `FGreedyMeshCube` (FRenderResource) dont les SRV alimentent le VF. C'est le maillage « réel » dessiné ; le compute ne fait que dupliquer/culler des **instances** de ce maillage.
- **RuntimeNanite = voie alternative.** `RuntimeNanite` n'utilise PAS le vertex factory ci-dessus. Il construit de vraies `Nanite::FResources` (`FRzNaniteBuilder::BuildNaniteResourcesOnly`) à partir de géométrie CPU, les enregistre auprès du Nanite streaming manager, et les rend via `FLiveNaniteSceneProxy : Nanite::FSceneProxyBase`. Rebuild asynchrone par chunk borné par `MaxConcurrentRebuilds`.
- **RzPlanet empile sur RuntimeNanite.** `FRzPlanetManager` fait la sélection LOD quadtree (cube-sphere, Morton codes, erreur écran, horizon/frustum cull) sur le **game thread** (`TickComponent` → `UpdateFromView`), construit les chunks de manière asynchrone (`FRzPlanetBuildTask`), et instancie un `URzPlanetChunkComponent` par chunk visible — chacun bâtissant ses propres `Nanite::FResources`.
- **Honnêteté sur `RuntimeNanite::ConfigureNaniteLimits`.** Cette fonction (qui gonfle `r.Nanite.MaxNodes`, etc.) existe mais **l'appel est commenté** dans `StartupModule()` : `// ConfigureNaniteLimits();`. Les limites Nanite ne sont donc PAS modifiées par ce module en l'état.

---

## 3. Carte des composants

### Module `IndirectInstancing`

| Élément | Type | Rôle |
|---|---|---|
| `FRzIndirectInstancingMeshVertexFactory` | class : `FVertexFactory` | Vertex factory GPU-driven. `ShouldCompilePermutation`/`ModifyCompilationEnvironment`, manual fetch, PrimitiveId stream, uniform buffer de fetch. |
| `FRzIndirectInstancingMeshUniformParameters` | global shader parameter struct | SRV de fetch (TexCoord/Tangents/Color) + `VertexFetch_Parameters`. Implémenté sous le nom shader `RzIndirectInstancingMeshVF`. |
| `FRzIndirectInstancingUserData` | struct : `FOneFrameResource` | Porte le `InstanceBufferSRV` jusqu'au `GetElementShaderBindings`. |
| `FRzIndirectInstancingVertexFactoryShaderParametersBase` / `...VS` / `...PS` | classes shader params | Bindings VS/PS ; le VS attache `InstanceBuffer` (SRV) + l'uniform buffer du VF. |
| `FRzIndirectInstancingSceneProxy` | class : `FPrimitiveSceneProxy` | Proxy de `URzIndirectInstancingComponent`. Crée le VF à la volée, alloue le `FMeshBatch` indirect, enregistre le travail compute. |
| `FRzIndirectInstancingRendererExtension` | class : `FRenderResource` (`TGlobalResource`) | Hook pre/post-render global ; orchestre `SubmitWork` (chaîne RDG compute) et le pool de `FDrawInstanceBuffers`. |
| `FInitBuffers_CS`, `FAddInstances_CS`, `FInitInstanceBuffer_CS`, `FCullInstances_CS`, `FClearInstances_CS` | `FGlobalShader` | Compute shaders (`RzIndirectInstancingCompute.usf`) : init, génération de grille, init args, frustum cull (perm. `REUSE_CULL`), clear. |
| `URzIndirectInstancingComponent` | `UPrimitiveComponent` | Composant scriptable : params de chunks/bruit, `GenerateChunks`/`GenerateAll`/`Dispose`/`ToggleShadows`. Détient le `FGreedyMeshCube` actif. |
| `ARzIndirectInstancing` | `AActor` (MinimalAPI) | Acteur porteur d'un `URzIndirectInstancingComponent`. |
| `EChunkSize` | `UENUM(BlueprintType)` | Dimension de chunk (4/8/16/32/48/64). |
| `FChunkPosition` | `USTRUCT(BlueprintType)` | Clé entière (X,Y,Z) hashable d'un chunk. |
| `FGreedyMeshGenerator` / `FGreedyMeshData` | class / struct | Voxelisation + greedy meshing (bruit Perlin 3D, masques `uint64`, fusion de quads). `CHUNK_SIZE 64`. |
| `FGreedyMeshCube` | class : `FRenderResource` | Conteneur RHI du maillage greedy (position/tangent/texcoord/color SRV + index buffer). `CreateFromGeometry`. |
| `FProceduralCubeMesh` / `FProceduralCubeIndexBuffer` / `FCubeVertex` | class / class / struct | Maillage de cube de secours (voie non-greedy du proxy). |
| `FIndirectInstancing` | `IModuleInterface` | Mappe `/IndirectInstancingShaders` vers `Shaders/IndirectInstancing/Private`. |

### Module `RuntimeNanite`

| Élément | Type | Rôle |
|---|---|---|
| `ULiveNaniteComponent` | `UPrimitiveComponent` | API Blueprint pour pousser des chunks/mesh combiné et déclencher des rebuilds Nanite live (`SetChunkData`, `ProcessUpdates`, batch update…). |
| `ALiveNaniteActor` | `AActor` (MinimalAPI) | Acteur de test (`GenerateTestChunk`/`ClearChunks`). |
| `FLiveNaniteManager` | class | Gère la map de chunks, la file dirty, les rebuilds asynchrones (`FAsyncTask<FLiveNaniteRebuildTask>`), le swap de ressources et l'enregistrement streaming. Borné par `MaxConcurrentRebuilds`. |
| `FLiveNaniteRebuildTask` / `FLiveNaniteRebuildResult` | `FNonAbandonableTask` / struct | Construction Nanite hors game thread. |
| `FLiveNaniteChunk` / `FLiveNaniteMeshData` | struct / struct | État d'un chunk (atomics `Version`/`bDirty`/`bRegistered`, `Nanite::FResources`) ; données de mesh source. |
| `FLiveNaniteSceneProxy` | class : `Nanite::FSceneProxyBase` | Proxy Nanite live ; `SetResources`, `GetResourceMeshInfo`/`GetResourcePrimitiveInfo`. |
| `FRzNaniteBuilder` | class | Cœur du build : géométrie CPU → `Nanite::FResources`/`FStaticMeshRenderData`/`UStaticMesh`. Précision de position quantifiée (`PositionPrecision`). |
| `FRuntimeNaniteModule` | `IModuleInterface` | `ConfigureNaniteLimits()` présent mais **non appelé** (commenté). |
| (`RzMeshSimplifier`, `RzGraphPartitioner`, `RzNaniteLODBuilder`, `RzNanite`) | classes internes | Briques de build Nanite (simplification, partitionnement de graphe, LOD, octaèdre de normales `Rz::Nanite::FOctahedron`). |

### Module `RzPlanet`

| Élément | Type | Rôle |
|---|---|---|
| `URzPlanetComponent` | `UPrimitiveComponent` | Composant planète : rayon, profondeur quadtree, seuil d'erreur écran, budget mémoire, bruit. `RebuildPlanet`/`GetStats`. Tick → `UpdateFromView`. |
| `ARzPlanetActor` | `AActor` (Blueprintable) | Acteur porteur d'un `URzPlanetComponent`. |
| `URzPlanetChunkComponent` | `UPrimitiveComponent` | Un chunk de terrain rendu en Nanite (`CreateSceneProxy` → proxy Nanite, `BuildNaniteResources`). |
| `FRzPlanetManager` | class | Sélection LOD quadtree (cube-sphere 6 faces, frustum/horizon cull), build asynchrone (`FAsyncTask<FRzPlanetBuildTask>`), création/destruction de composants chunk, budget mémoire. |
| `FRzQuadNode` / `FRzCubeFace` / `FRzPlanetChunk` / `FRzPlanetBuildTask` | classes/struct internes | Nœud de quadtree, face de cube-sphère, état de chunk, tâche de build. |
| `ERzCubeFaceId` | `UENUM(BlueprintType)` | Les 6 faces (PosX…NegZ). |
| `FRzQuadNodeId` | struct | Identité d'un nœud (face + depth + Morton code) ; helpers parent/enfant/UV bounds/hash. |
| `FRzNoiseSettings` | `USTRUCT(BlueprintType)` | Paramètres de bruit fBm (octaves, fréquence, persistance, lacunarité, amplitude, seed). |
| `FRzPlanetConfig` | struct (non-UStruct) | Config interne du manager (rayon, depth, résolution de chunk, builds concurrents…). |
| `FRzPlanetStats` | `USTRUCT(BlueprintType)` | Stats runtime (chunks actifs, builds, triangles, clusters, LOD visible, mémoire…). |
| Groupe de stats `STATGROUP_RzPlanet` | stats | `UpdateFromView`, `SelectVisibleNodes`, `TraverseQuadtree`, `BuildNaniteResources`, compteurs de chunks/mémoire. |

### Modules `RzBenchmark` et `IndirectInstancingEditor`

| Élément | Type | Rôle |
|---|---|---|
| `URzBenchmarkRunner` | `UObject` (BlueprintType) | Lance/annule des benchmarks (`RunBenchmark`), délégués `OnBenchmarkComplete`/`OnBenchmarkProgress`, agrège les `FRzBenchmarkResult`. |
| `ERzBenchmarkType` | `UENUM(BlueprintType)` | RuntimeNanite_CubeGrid / _Sphere / _CustomMesh / GreedyMesh_Voxels / CustomInstancing_Grid. |
| `FRzBenchmarkResult` / `FRzBenchmarkConfig` | `USTRUCT(BlueprintType)` | Résultats chronométrés (tris/s, mémoire…) et config de test. |
| `FRzBenchmarkModule` | `IModuleInterface` | Module éditeur du benchmark. |
| `SRzInstancingToolbox` | `SCompoundWidget` | Panneau Slate à 4 onglets (Greedy Mesh / Live Nanite / Landscape / Benchmark) ; spawn d'acteurs de test, stats live. |
| `FIndirectInstancingEditorModule` | `IModuleInterface` | Enregistre l'onglet nomad `RzInstancingToolbox` + l'entrée du menu Window. |

---

## 4. Flux de données et cycle de vie

### Voie `IndirectInstancing` (vertex factory GPU-driven)

1. **Génération CPU.** `URzIndirectInstancingComponent::GenerateChunks` (appelable en éditeur ou via `GenerateAll`) parcourt la grille `NumChunks{X,Y,Z}`, fait tourner `FGreedyMeshGenerator::GenerateMeshWithNoise` par chunk, fusionne la géométrie, puis crée un `FGreedyMeshCube` combiné (`ActiveGreedyMeshResource`) initialisé sur le render thread via `ENQUEUE_RENDER_COMMAND`. `MarkRenderStateDirty()` reconstruit le proxy.
2. **Création du proxy.** `CreateSceneProxy()` → `FRzIndirectInstancingSceneProxy`. Le proxy récupère le `FGreedyMeshCube`, valide le matériau (exige `MATUSAGE_VirtualHeightfieldMesh`, sinon fallback `MD_Surface`), et appelle `RzIndirectInstancingRendererExtension.RegisterExtension()` (idempotent).
3. **Construction du VF (lazy, render thread).** Au premier `GetDynamicMeshElements`, le proxy alloue un `FRzIndirectInstancingMeshVertexFactory`, lie les vertex buffers du `FGreedyMeshCube` (`Bind*VertexBuffer`), construit l'uniform buffer de fetch (`CreateVFUniformBuffer`) et appelle `InitRHI` (déclaration : position en attribut 0, tangentes 1/2, color 3, texcoords 4+, PrimitiveId stream en 13).
4. **Émission du batch indirect.** Pour chaque vue visible (et chaque frustum d'ombre), le proxy : appelle `AddWork(this, vue principale, vue de cull)` pour obtenir un `FDrawInstanceBuffers` ; alloue un `FMeshBatch` `PT_TriangleList` avec `NumPrimitives = 0`, `IndirectArgsBuffer` = celui des buffers, et un `FRzIndirectInstancingUserData` portant le `InstanceBufferSRV`.
5. **Chaîne compute (début de frame suivante).** `FRzIndirectInstancingRendererExtension::BeginFrame` (délégué pre-render) appelle `SubmitWork(FRDGBuilder&)` : transition des buffers, `InitInstanceBuffer` (args indirect = `NumIndices`), puis par proxis/vue principale : `InitBuffers`, `Clear` si `ClearInstancesNextFrame`, `AddInstances` si `AddInstancesNextFrame` (génère `GridSizeX*Y*Z` instances, threadgroup 1024), enfin `CullInstances` par vue enfant (5 plans de frustum en espace local). `MAX_INSTANCES = 4 194 304`.
6. **Rendu.** Le VS lit l'instance depuis `InstanceBuffer` (manual fetch), le draw consomme les arguments indirects remplis par le cull. `EndFrame` (post-render) recycle les `FDrawInstanceBuffers` (libérés après 4 frames d'inactivité, cf. `DiscardId`).
7. **Teardown.** `Dispose()` libère les `FGreedyMeshCube` sur le render thread ; `DestroyRenderThreadResources` libère le VF et l'uniform buffer.

> Note de réalité : `AddPass_AddInstances` génère une **grille** d'instances (`GridSize{X,Y,Z}`), tous fixés à `1` dans le constructeur du proxy. Telle quelle, la voie indirecte dessine donc **une seule instance** du maillage greedy combiné par défaut — le mécanisme d'instanciation GPU est en place mais non « rempli » par le composant en l'état.

### Voie `RuntimeNanite`

1. `ULiveNaniteComponent::SetChunkData`/`SetCombinedMeshData` → `FLiveNaniteManager::MarkChunkDirty` (déplace la `FLiveNaniteMeshData`).
2. `ProcessPendingUpdates` (tick ou `ProcessUpdates`) démarre jusqu'à `MaxConcurrentRebuilds` `FAsyncTask<FLiveNaniteRebuildTask>` ; chaque tâche appelle `FRzNaniteBuilder::BuildNaniteResourcesOnly` hors game thread.
3. À complétion, `PerformResourceSwap` échange les `Nanite::FResources` (avec gestion de `Version` pour rejeter les builds périmés) et (dé)enregistre auprès du Nanite streaming manager.
4. `CreateSceneProxy` → `FLiveNaniteSceneProxy::SetResources`. Le rendu suit le pipeline Nanite standard.

### Voie `RzPlanet`

1. `URzPlanetComponent::TickComponent` → `Manager->UpdateFromView(viewLocation, viewportHeight)` (game thread) : `SelectVisibleNodes` traverse le quadtree des 6 faces, choisit les nœuds par erreur écran, frustum et horizon.
2. Les nœuds à charger entrent dans la `BuildQueue` → `FAsyncTask<FRzPlanetBuildTask>` (jusqu'à `MaxConcurrentBuilds`).
3. `ProcessCompletedBuilds`/`CreateChunkComponent` crée un `URzPlanetChunkComponent` (≤ `MaxChunksPerFrame` par frame) qui construit ses `Nanite::FResources` et les rend en Nanite. `EnforceMemoryBudget` décharge selon `MemoryBudgetBytes`.

### Voies éditeur

- `IndirectInstancingEditor::StartupModule` enregistre l'onglet nomad `RzInstancingToolbox` (catégorie Developer Tools) + une entrée dans `LevelEditor.MainMenu.Window`. `SRzInstancingToolbox` spawn des acteurs de test et lit les stats des managers.
- `URzBenchmarkRunner::RunBenchmark` exécute `RunRuntimeNaniteCubeGrid`/`RunRuntimeNaniteSphere`, chronomètre setup/exécution/première frame, et diffuse résultats/progression.

---

## 5. Réplication / réseau

**Aucune.** Aucun module ne déclare de propriété `Replicated`, n'implémente `GetLifetimeReplicatedProps`, ni n'expose de RPC `Server_`/`Client_`/`Multicast_`. Tout est purement client/local (génération de géométrie, rendu, builds asynchrones, outillage éditeur). Sur un serveur dédié sans rendu, ces composants n'auraient aucun effet visible et ne synchronisent rien.

---

## 6. Points d'intégration

### Dépendances internes (modules → modules)

- `RzIndirectInstancingPlugin` (coquille) → `IndirectInstancing`, `RzPlanet` (public).
- `RzPlanet` → `RuntimeNanite` (private). `RzBenchmark` → `RuntimeNanite` (public).
- `IndirectInstancingEditor` (via `SRzInstancingToolbox`) → `RzIndirectInstancingComponent` (IndirectInstancing) + `RzBenchmarkRunner` (RzBenchmark) + `ULiveNaniteComponent` (RuntimeNanite).

### Dépendances engine notables

- `IndirectInstancing` : `Renderer`, `RenderCore`, `RHI`, `MaterialShaderQualitySettings`. En éditeur : `MaterialUtilities`, `UnrealEd`, Slate.
- `RuntimeNanite` : `Renderer`, `MeshDescription`, `StaticMeshDescription`, `MeshUtilitiesCommon`, et l'include interne `Source/Runtime/Engine/Internal` (accès aux APIs Nanite internes).
- `RzPlanet` : `Renderer`, `RuntimeNanite`. En éditeur : `PropertyEditor`.

### Répertoires shader virtuels

- `/IndirectInstancingShaders` → `Plugins/RzIndirectInstancingPlugin/Shaders/IndirectInstancing/Private` (mappé par `FIndirectInstancing::StartupModule`, avec fallback via `IPluginManager`). Fichiers : `RzIndirectInstancingVertexFactory.ush`, `RzIndirectInstancingCompute.usf`, `RzIndirectInstancing.ush`, `GreedyMeshShaders.usf`, `MeshOptimization.usf`.

### CVars / settings

- Pas de CVar `IConsoleVariable` enregistrée par le plugin. `RuntimeNanite::ConfigureNaniteLimits` **lit/écrit** des CVars Nanite (`r.Nanite.MaxNodes`, `r.Nanite.MaxCandidateClusters`, `r.Nanite.MaxVisibleClusters`, `r.Nanite.MaxCandidatePatches`, `r.Nanite.MaxVisiblePatches`) mais **n'est pas appelée** (commentée dans `StartupModule`). Si un build massif heurte les limites Nanite, c'est le levier à réactiver.
- Macro de compilation : `RzIndirectInstancing_ENABLE_GPU_SCENE_MESHES 1` (header du VF) ; `RZINDIRECTINSTANCINGPLUGIN_EXPORTS=1` (Build.cs de la coquille).

### Dépendants QANGA

Aucun module C++ de `G:/QANGA/Source`, aucun autre plugin, aucun asset `Content/`, et le `.uproject` ne liste pas explicitement le plugin (chargé par présence en tant que project plugin). Le plugin est autonome.

---

## 7. Gotchas, invariants et pièges

- **`ShouldCompilePermutation` (le piège historique).** `FRzIndirectInstancingMeshVertexFactory::ShouldCompilePermutation` rejette `MD_Volume`, accepte inconditionnellement les special engine materials, et sinon **ne compile que pour `SF_Vertex`, `SF_Pixel`, `SF_RayHitGroup`**. Auparavant trop permissif (compilait pour toutes les fréquences shader) — restreindre cette liste est un correctif documenté. Toute fréquence ajoutée ici multiplie le coût de compilation shader.
- **Exigence matériau `MATUSAGE_VirtualHeightfieldMesh`.** Le scene proxy n'utilise le matériau du composant que si `CheckMaterialUsage_Concurrent(MATUSAGE_VirtualHeightfieldMesh)` passe ; sinon il retombe silencieusement sur le matériau par défaut. Un matériau sans ce usage flag rend en gris « default » sans erreur évidente.
- **`GridSize{X,Y,Z}` codés à 1.** Le proxy fixe `GridSizeX/Y/Z = 1` : la voie indirecte ne génère qu'**une** instance par défaut. Le pipeline d'instanciation GPU (`AddInstancesCS`) est fonctionnel mais le composant ne lui passe pas de grille > 1.
- **VF créé sur le render thread dans `GetDynamicMeshElements` (const-cast).** Le proxy construit son `FRzIndirectInstancingMeshVertexFactory` à la première frame via `const_cast<FRzIndirectInstancingSceneProxy*>(this)`. C'est intentionnel mais fragile : toute modification doit rester thread-safe render-thread et ne jamais s'exécuter sur le game thread.
- **L'extension de renderer est un singleton global.** `RzIndirectInstancingRendererExtension` (`TGlobalResource`) s'enregistre une seule fois (`static bool bInit`) sur les délégués pre/post-render. Les `FDrawInstanceBuffers` sont **partagés/poolés entre tous les proxies et vues** et recyclés après 4 frames inactives. Ne pas supposer qu'un buffer appartient durablement à un proxy.
- **Manual vertex fetch obligatoire.** Le shader VF gère deux chemins (`MANUAL_VERTEX_FETCH` 0/1) mais `ModifyCompilationEnvironment` force `MANUAL_VERTEX_FETCH=1`. Les SRV de l'uniform buffer (`VertexFetch_*`) doivent être valides, sinon vertices nuls.
- **`MAX_INSTANCES = 4 194 304`** dimensionne l'`InstanceBuffer` (structuré, `sizeof(FRzIndirectInstancingRenderInstance)` = 3 floats). Dépasser plante le cull/draw.
- **`RuntimeNanite` touche des APIs engine internes.** Le module inclut `Engine/Internal` et manipule directement `Nanite::FResources` + le streaming manager. Très sensible aux montées de version d'Unreal (les structures Nanite internes ne sont pas une API stable).
- **`ConfigureNaniteLimits` non branchée.** Voir §6 — ne pas supposer que les limites Nanite sont relevées par le plugin.
- **Logs `LogTemp` verbeux.** Les voies `IndirectInstancing`/`RuntimeNanite` loggent abondamment en `LogTemp` (warnings/errors par-frame en cas de ressource invalide, displays par chunk généré). En production cela violerait la règle « max 1 log/s par source » du projet ; à durcir avant tout usage shipping.
- **`bUseGreedyMesh = true` par défaut.** Le proxy privilégie la voie greedy ; la voie `FProceduralCubeMesh` n'est utilisée que si aucun `FGreedyMeshCube` valide n'existe.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `RzIndirectInstancingPlugin.uplugin` | Déclaration des 6 modules, types et LoadingPhases. |
| `Source/RzIndirectInstancingPlugin/...` | Module-coquille (logs de démarrage seulement). |
| `Source/IndirectInstancing/Public/RzIndirectInstancingVertexFactory.h` + `Private/...cpp` | **Cœur VF** : `ShouldCompilePermutation`, `ModifyCompilationEnvironment`, `InitRHI`, bindings VS/PS, uniform buffer de fetch, `IMPLEMENT_VERTEX_FACTORY_TYPE` (→ `RzIndirectInstancingVertexFactory.ush`). |
| `Source/IndirectInstancing/Public/RzIndirectInstancingSceneProxy.h` + `Private/...cpp` | Proxy + `FRzIndirectInstancingRendererExtension` + tous les `FGlobalShader` compute + `SubmitWork` (chaîne RDG). |
| `Source/IndirectInstancing/.../RzIndirectInstancingComponent.{h,cpp}` | `URzIndirectInstancingComponent` (`EChunkSize`, `FChunkPosition`, génération de chunks). |
| `Source/IndirectInstancing/.../RzIndirectInstancingActor.{h,cpp}` | `ARzIndirectInstancing`. |
| `Source/IndirectInstancing/.../GreedyMeshGenerator.{h,cpp}` | Voxelisation + greedy meshing (`FGreedyMeshData`, bruit Perlin 3D). |
| `Source/IndirectInstancing/.../GreedyMeshCube.{h,cpp}`, `ProceduralCubeMesh.{h,cpp}` | Conteneurs RHI des maillages dessinés. |
| `Source/IndirectInstancing/Private/IndirectInstancing.cpp` | Mapping du répertoire shader `/IndirectInstancingShaders`. |
| `Shaders/IndirectInstancing/Private/RzIndirectInstancingVertexFactory.ush` | VS/PS du VF (manual fetch, instancing, shadow depth). |
| `Shaders/IndirectInstancing/Private/RzIndirectInstancingCompute.usf` | Compute : `InitBuffersCS`, `AddInstancesCS`, `InitInstanceBufferCS`, `CullInstancesCS`, `ClearInstancesCS`. |
| `Source/RuntimeNanite/Public/RzLiveNaniteComponent.h` + `Private/...` | `ULiveNaniteComponent` (API Blueprint de Nanite live). |
| `Source/RuntimeNanite/Public/RzLiveNaniteManager.h` + `Private/...` | `FLiveNaniteManager`, rebuild asynchrone, swap de ressources. |
| `Source/RuntimeNanite/Public/RzLiveNaniteChunk.h` | `FLiveNaniteChunk`, `FLiveNaniteMeshData`. |
| `Source/RuntimeNanite/Public/RzLiveNaniteSceneProxy.h` + `Private/...` | `FLiveNaniteSceneProxy : Nanite::FSceneProxyBase`. |
| `Source/RuntimeNanite/Public/RzNaniteBuilder.h` + `Private/RzNaniteBuilder.cpp` | Construction `Nanite::FResources` à l'exécution. |
| `Source/RuntimeNanite/Private/{RzNanite,RzMeshSimplifier,RzGraphPartitioner,RzNaniteLODBuilder}.cpp` | Briques internes du build Nanite. |
| `Source/RuntimeNanite/Private/RuntimeNanite.cpp` | Module ; `ConfigureNaniteLimits` (non appelée). |
| `Source/RzPlanet/Public/RzPlanetComponent.h` + `Private/...` | `URzPlanetComponent` (tick → `UpdateFromView`). |
| `Source/RzPlanet/Public/RzPlanetManager.h` + `Private/RzPlanetManager.cpp` | `FRzPlanetManager` (quadtree, sélection LOD, builds async). |
| `Source/RzPlanet/Public/RzPlanetTypes.h` | `ERzCubeFaceId`, `FRzQuadNodeId`, `FRzNoiseSettings`, `FRzPlanetConfig`, `FRzPlanetStats`, stats. |
| `Source/RzPlanet/.../{RzQuadNode,RzCubeFace,RzPlanetChunk,RzPlanetBuildTask,RzPlanetChunkComponent,RzPlanetActor}.{h,cpp}` | Quadtree, faces, chunks, tâches de build, composant/acteur. |
| `Source/RzBenchmark/Public/RzBenchmarkRunner.h` + `RzBenchmarkTypes.h` + `Private/...` | `URzBenchmarkRunner`, `ERzBenchmarkType`, résultats/config. |
| `Source/IndirectInstancingEditor/.../IndirectInstancingEditorModule.{h,cpp}` | Onglet nomad + entrée menu Window. |
| `Source/IndirectInstancingEditor/.../SRzInstancingToolbox.{h,cpp}` | Toolbox Slate 4 onglets (Greedy/Live Nanite/Landscape/Benchmark). |
