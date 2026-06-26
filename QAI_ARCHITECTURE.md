# QAI - Architecture Technique

> Document mis a jour pour refleter l'etat du plugin au 26 juin 2026 (UE 5.7). Depuis la
> premiere redaction (police only, 3 behaviors, FSM plate a 7 etats), le plugin a triple de
> perimetre : ~13 archetypes routes par behavior, pathfinding par flow field, tier impostor VAT,
> significance/LOD + paliers d'allegement, module de factions dedie, trafic aerien, partage
> d'animation. Ce document decrit ce que le code livre REELLEMENT aujourd'hui.

## Etat reel de l'architecture

L'intention de design originale etait un **"Data-Driven ECS avec FSM Hierarchique et Batch Processing"**. En pratique, le code actuel est un :

**"Data-Oriented Batch Processing avec FSM plate, Behaviors routes par Archetype, et un pipeline de presentation multi-paliers (live / impostor dormant / partage d'anim)"**

| Label original         | Realite dans le code                                                                 | Statut    |
|------------------------|--------------------------------------------------------------------------------------|-----------|
| **Batch Processing**   | ParallelFor, SIMD SSE/NEON, SoA cache-aligne, prefetch, fusion State->Mvt->Combat    | Livre     |
| **Data-Driven**        | `UQAI_PawnConfig` (DataAsset) par archetype + `FQAI_*Config` par type ; mais la FSM et la matrice de factions restent hardcodees en C++ | Partiel ameliore |
| **ECS**                | Entity handles (sparse+generation) + Systems (processeurs) ; toujours pas de composition dynamique de composants (chaque agent porte TOUS les `FQAI_*Config`) | Inspire   |
| **FSM Hierarchique**   | FSM plate (`EQAI_AgentState`) + Strategy par archetype (`FQAI_*Behavior`) ; pas de super/sous-etats | Non       |

**Ce qui a change depuis la v1** : la composition par archetype a beaucoup progresse (13 behaviors,
configs typees, DataAsset d'authoring), le module de factions duplique a ete consolide
(`QAI_FactionLib`), un vrai systeme de pathfinding existe (flow field), et une couche entiere de
performance a ete ajoutee (significance/LOD, tier impostor, partage d'anim). Ce qui manque toujours
pour atteindre le design original : la composition memoire dynamique (vrai ECS) et l'imbrication
d'etats (HFSM). Voir la [section 19 (Guide pour le refactoring)](#19-guide-pour-le-refactoring).

---

## Table des matieres

1. [Pourquoi cette architecture pour QANGA](#1-pourquoi-cette-architecture-pour-qanga)
2. [Vue d'ensemble du pipeline](#2-vue-densemble-du-pipeline)
3. [Le Registre SoA (Structure of Arrays)](#3-le-registre-soa-structure-of-arrays)
4. [SIMD, cache et constantes](#4-simd-cache-et-constantes)
5. [Les Processeurs (Batch Processing)](#5-les-processeurs-batch-processing)
6. [La FSM (State Machine)](#6-la-fsm-state-machine)
7. [Le systeme de Comportement par Archetype](#7-le-systeme-de-comportement-par-archetype)
8. [Catalogue des archetypes](#8-catalogue-des-archetypes)
9. [Pathfinding par Flow Field](#9-pathfinding-par-flow-field)
10. [Le tier Impostor (VAT) et la dormance](#10-le-tier-impostor-vat-et-la-dormance)
11. [Significance / LOD et paliers d'allegement](#11-significance--lod-et-paliers-dallegement)
12. [Spaceship AI et trafic aerien](#12-spaceship-ai-et-trafic-aerien)
13. [Architecture Client/Serveur et multijoueur](#13-architecture-clientserveur-et-multijoueur)
14. [Systeme de factions](#14-systeme-de-factions)
15. [Spatial Hash Grid et Sensing](#15-spatial-hash-grid-et-sensing)
16. [Composants de mouvement et d'animation](#16-composants-de-mouvement-et-danimation)
17. [Spawners et cycle de vie](#17-spawners-et-cycle-de-vie)
18. [Configuration, Settings et CVars](#18-configuration-settings-et-cvars)
19. [Guide pour le refactoring](#19-guide-pour-le-refactoring)
20. [Structures de donnees de reference](#20-structures-de-donnees-de-reference)
21. [Fichiers et emplacements](#21-fichiers-et-emplacements)

---

## 1. Pourquoi cette architecture pour QANGA

### Le probleme

QANGA est un jeu multijoueur en monde ouvert avec des planetes entieres (LWC). Les AIs existent sur
des surfaces planetaires courbes, dans l'espace, en vol atmospherique, et au sol. Le systeme doit gerer :

- **Des centaines d'agents simultanement** : cyborgs, creatures (Sangline, Infected, Animal...), drones police, vaisseaux de trafic.
- **Plusieurs domaines de locomotion** (sol, surface, vol, espace) avec des physiques radicalement differentes.
- **Une architecture client/serveur asymetrique** : le serveur est autoritaire sur l'etat, le mouvement et le combat ; les clients allegent la presentation.
- **Un serveur dedie SANS meshes** (par design) : aucune collision/geometrie cote serveur, ce qui interdit NavMesh et line-traces fiables.
- **Des performances critiques** : objectif ~60 fps editeur PIE (Slate inclus) avec 100 AI en combat.

### Pourquoi pas le systeme AI natif d'Unreal ?

- **1 `AIController` par Pawn** = overhead UObject par agent, zero coherence de cache.
- **Behavior Trees / StateTree individuels** = aucune vectorisation, aucun batch.
- **NavMesh** = non fonctionnel sur planetes spheriques, dans l'espace, et absent sur serveur dedie.

L'ancien pipeline BP (`Qanga_AI_Controller` + StateTree + `AI_Manager` + `AI_Spawner`) a ete
**entierement migre en C++** : plus d'`AAIController` custom (le `AAIController` stock suffit), plus
de StateTree assets, plus de BTT. Tout est dans le pipeline SoA + behaviors.

### La solution : architecture Data-Oriented inspiree de l'ECS

| Concept ECS        | Equivalent QAI                                                  | Fidelite |
|--------------------|----------------------------------------------------------------|----------|
| Entity             | `FQAI_AgentHandle` (index sparse + generation)                  | Fidele   |
| Component (data)   | Arrays SoA dans `UQAI_AgentRegistry`                            | Partiel (layout fixe, pas composable) |
| System             | Processeurs (`State`, `Movement`, `Combat`)                     | Fidele   |
| Archetype          | `EQAI_AgentArchetype` + `FQAI_*Behavior` + `FQAI_*Config`       | Route le behavior et la config typee, pas le layout memoire |

**Dependances du plugin** (`QAI.uplugin`) : `QWeapon`, `GameplayAbilities`, `Cy_Trace`,
`FlyVehicleMovement`, `GravityScape`, `AnimToTexture`, et `WorldScape` (optionnel). Module `QAI`
charge en `PostDefault`.

---

## 2. Vue d'ensemble du pipeline

Le subsysteme `UQAI_SubSystem` (un `UWorldSubsystem`) possede et orchestre tout :

- `UQAI_AgentRegistry` (stockage SoA)
- `UQAI_AgentBehaviorRegistry` (1 behavior par archetype)
- Les 3 processeurs : `UQAI_StateProcessor`, `UQAI_MovementProcessor`, `UQAI_CombatProcessor`
- `UQAI_FlowFieldManager` (pose sur un acteur-hote transient `QAI_PathfindingManager` au `OnWorldBeginPlay`)

Le tick est une **custom tick function** (`FCustomTickFunction RealTime`) enregistree en
`TG_PrePhysics`, intervalle `0` (chaque frame). Le subsysteme ne s'auto-desactive que via
`bDisableSubsystemTick` (config).

```
UQAI_SubSystem::RealTimeTick(DeltaTime)   [TG_PrePhysics, chaque frame]
        |
        +-- UpdateAgentSignificance()   <- stampe SignificanceLOD de chaque agent (distance au viewer)
        |
        +-- selon NetMode :
        |
   [NM_Standalone] ----------------------------------------------------------------
        | LifecycleTick() (~0.5 Hz)  +  AutoDiscoverAgents() (~0.5 Hz)
        | ProcessAllPhasesPipelined(dt)   <- State->Movement->Combat FUSIONNES
        |                                    en UN seul ParallelFor (1 barriere)
        |
   [NM_DedicatedServer / NM_ListenServer] -----------------------------------------
        | LifecycleTick() (~0.5 Hz)  +  AutoDiscoverAgents() (~0.5 Hz)
        | sync transforms acteur->registre (mouvement client-authoritatif)
        | StateProcessor.ProcessBatch()   (3 ProcessBatch sequentiels, car le
        | MovementProcessor.ProcessBatch()  Movement ne tourne que pour les agents
        | CombatProcessor.ProcessBatch()    a ServerSimulate=true -> sous-ensembles differents)
        |
   [NM_Client] --------------------------------------------------------------------
        | State + Movement UNIQUEMENT pour les agents locaux/proxy (unites police a marker)
        |   -> les creatures serveur-autoritatives sont ignorees (sinon double-simulation)
        | AutoDiscoverAgents() (~0.5 Hz)
        | UpdateClientTrafficShips()      <- presentation lean des vaisseaux de trafic
        | UpdateClientCreatureImpostors() <- impostors VAT cote client pour les creatures lointaines
```

### Ordre d'execution des processeurs (deterministe)

1. **State** (`StateMachine`) — transitions FSM, acquisition de cible, pauses de patrouille, gestion du flow field, marque `Movement`/`Combat` pour la suite.
2. **Movement** (`Movement`) — calcule l'intention via le behavior, l'applique (rotation, ground-snap, FPM/vehicule/spaceship).
3. **Combat** (`Combat`) — execute la decision de combat (tir, melee, detonation, armes vaisseau).

Cet ordre garantit que l'etat est a jour avant le mouvement, et le mouvement avant le combat
(distance/LOS correctes). En standalone, les trois phases sont **fusionnees** dans un unique
`ParallelFor` ou chaque worker traite State->Movement->Combat pour un agent (`ProcessAllPhasesPipelined`),
ce qui remplace 3 `TickCompletionEvents.Wait` par 1.

### CVars de diagnostic du pipeline

| CVar | Defaut | Role |
|------|--------|------|
| `QAI.Disable.State` / `QAI.Disable.Movement` / `QAI.Disable.Combat` | 0 | Saute le `ProcessBatch` d'un processeur (profilage par elimination). |
| `QAI.Debug` | off | Overlay ecran (totaux, par archetype, par etat, spawners). Commande console. |
| `QAI.DumpAgents` / `QAI.DumpSpawners` / `QAI.DumpFPM` | - | Dumps one-shot vers `LogQAI`. |
| `QAI.Verbose [0\|1]` | off | Active les logs verbeux a runtime (override `bVerboseLogging`). |
| `QAI.StressBench <count> <classpath>` | - | Benchmark crowd headless (spawn spirale + capture trace cpu,frame). |

---

## 3. Le Registre SoA (Structure of Arrays)

### Fichiers
- `Public/Registry/QAI_AgentRegistry.h`
- `Private/Registry/QAI_AgentRegistry.cpp`

### Architecture memoire

Donnees des agents en **tableaux paralleles**, classees par frequence d'acces.

```
HOT DATA (chaque frame, struct alignas(64))
+-----------------------------------------------------------+
| TQAICacheAlignedArray<FQAI_AgentHotData> HotData          |
|   - Location      (FVector, 12B)                          |
|   - Velocity      (FVector, 12B)                          |
|   - Rotation      (FQuat,   16B)                          |
|   - Scale         (FVector, 12B)                          |
|   - ProcessingFlags  (uint8, 1B)                          |
|   - SignificanceLOD  (uint8, 1B)   <- NOUVEAU (band LOD)  |
|   - bImpostorDormant (uint8, 1B)   <- NOUVEAU (dormance)  |
|   - Padding          (uint8, 1B)                          |
|   - LastUpdateTime   (float, 4B)                          |
|   = 60 octets, tient dans 1 cache line de 64B             |
+-----------------------------------------------------------+

WARM DATA (a chaque transition d'etat)
+-----------------------------------------------------------+
| TArray<FQAI_AgentState> PoliceStates                      |  (nom historique ; couvre police ET creatures)
+-----------------------------------------------------------+

COLD DATA (rarement accede)
+-----------------------------------------------------------+
| TArray<FQAI_AgentData> AgentConfigs   (tous les FQAI_*Config + Archetype) |
| TArray<TWeakObjectPtr<AActor>> OwnerActors                 |
| TArray<TWeakObjectPtr<UActorComponent>> WeaponCoordinators |  (par nom de classe au CreateAgent)
| TArray<TWeakObjectPtr<UActorComponent>> TaserComponents    |
| TWeakObjectPtr<AActor> CachedWorldScapeRoot  <- NOUVEAU    |  (resolu GameThread, lu en parallele)
+-----------------------------------------------------------+

SPARSE/DENSE
  TArray<int32> SparseToDense, DenseToSparse ; TArray<uint32> Generations
```

`SignificanceLOD` est stampe une fois par frame (GameThread, `UpdateAgentSignificance`) et lu
lock-free pendant la phase parallele (stride du cerveau + throttle de tick du mouvement).
`bImpostorDormant` vaut 1 quand le pawn est l'ancre cachee/figee d'un impostor ISM/VAT.
`CachedWorldScapeRoot` existe pour une raison de **thread-safety** : les behaviors doivent lire le
root WorldScape ici pendant la phase parallele, jamais via `TActorIterator` sur un worker (le
`check(IsInGameThread())` de `FActorIteratorState` est compile out en Shipping -> corruption de heap
silencieuse).

### Le systeme Sparse/Dense + generation

```cpp
struct FQAI_AgentHandle { int32 Index = INDEX_NONE; uint32 Generation = 0; };
```

`HandleToDenseIndex` valide bornes + `SparseToDense != INDEX_NONE` + `Generations[idx] == Handle.Generation`.
La suppression est un **swap-and-pop** (`RemoveAgent`) : O(1), tableau dense sans trous ; `FreeSparseSlot`
**incremente la generation** (invalide les vieux handles). `RemoveAgent` nettoie explicitement la queue
des tableaux warm/cold pour eviter que le GC trace des `AActor*` morts (`PursuitTarget`/`OriginalTarget`
dans `FQAI_AgentState`).

### TQAICacheAlignedArray

Allocation `FMemory::Malloc(bytes, CACHE_LINE_SIZE=64)`, capacite arrondie a `SIMD_BATCH_SIZE`,
non-copyable (move-only), `RemoveAtSwap`. Utilise **uniquement** pour `HotData` ; les tableaux
warm/cold sont des `TArray` standard. Note : les `BatchUpdate*`/`BatchExtract*` du registre sont des
boucles scalaires avec prefetch (le code SSE/NEON sert surtout au spatial hash).

---

## 4. SIMD, cache et constantes

### Fichiers
- `Public/Performance/QAI_SIMD.h`
- `Public/Performance/QAI_PerformanceConstants.h`

### Intrinsiques SIMD (namespace `QAISIMD`)

Traitement de **4 agents par instruction** : `CalculateDistancesSquared4_SSE/_NEON/_Scalar`,
`NormalizeVectors4_*` (rsqrt rapide + 1 iteration Newton-Raphson), `RangeCheckBatch` (`_mm_cmple_ps` +
`_mm_movemask_ps`). API haut niveau auto-dispatch : `CalculateDistancesSquaredBatch`,
`NormalizeVectorsBatch`, `RangeCheckBatch` (boucles de 4 + reste scalaire). Selection via
`#if QAI_SIMD_AVAILABLE` (SSE Win/Linux, NEON Mac).

### Constantes pre-calculees (`namespace QAIPerformance`)

Toutes les distances de range check sont au carre en `constexpr` (`TRACE_ATTACK_RANGE_SQUARED`,
`MELEE_RANGE_SQUARED`, `TASER_RANGE_SQUARED`, `DRONE_ACCEPTANCE_RADIUS_SQUARED`...), plus les timings
(`DRONE_FIRE_RATE=0.125`, `TASER_COOLDOWN=3.0`, `MELEE_ATTACK_COOLDOWN=1.5`...) et les tailles de batch
(`AGENT_BATCH_SIZE=64`, `SIMD_BATCH_SIZE=4`, `CACHE_LINE_SIZE=64`, `SIMD_ALIGN_BYTES=16`).

**NOUVEAU — bandes de significance/LOD** :

| Constante | Valeur | Role |
|-----------|--------|------|
| `NUM_LOD_BANDS` | 4 | nombre de bandes (0 = le plus proche / le plus detaille) |
| `LOD_BrainStride[4]` | `{1, 2, 4, 8}` | stride (en frames) de re-evaluation du cerveau (State+Combat), phase par index d'agent |
| `LOD_MoveTickInterval[4]` | `{0, 0.05, 0.1, 0.25}` | intervalle de tick (s) du FloatingPawnMovement par bande |
| `ClampLODBand(int)` | - | clamp dans `[0, NUM_LOD_BANDS-1]` |

### Prefetch

`PrefetchForRead` (`_MM_HINT_T2`) / `PrefetchForWrite` (`_MM_HINT_T0`) avec un pattern de "rolling
prefetch" (charger l'element i+4 pendant qu'on traite i) dans les `BatchExtract*`/`BatchUpdate*`.

---

## 5. Les Processeurs (Batch Processing)

### Fichiers
- `Public+Private/Processor/QAI_ProcessorBase.{h,cpp}`
- `Public+Private/Processor/QAI_StateProcessor.{h,cpp}`
- `Public+Private/Processor/QAI_MovementProcessor.{h,cpp}`
- `Public+Private/Processor/QAI_CombatProcessor.{h,cpp}`

### ProcessorBase — le framework

Chaque processeur herite de `UQAI_ProcessorBase` (`UCLASS(Abstract)`) et declare son nom + son flag
via la macro `DECLARE_QAI_PROCESSOR(Class, "Name", Flag)`.

```
ProcessBatch(Registry, DeltaTime)
    +-> ShouldProcess()                 (skip si DeltaTime*1000 > FrameTimeThresholdMs=16ms)
    +-> GetAgentsForProcessing(Flag)    (filtre par bitmask de processing flags)
    +-> PreProcessBatch()               (setup GameThread)
    +-> InternalProcessBatch()          (TOUJOURS ParallelFor par chunks de AgentsPerChunk)
    |       +-> ProcessChunk() -> par agent: IsValid -> PassesSignificanceStride -> ShouldProcessAgent -> ProcessAgent
    +-> PostProcessBatch()              (apply GameThread)
    +-> efface le flag de processing des agents traites
```

**Changements vs v1** :
- Le traitement est **toujours parallele** : le toggle `bEnableParallelProcessing` et le seuil
  `MinAgentsForParallel` ont disparu (un petit effectif collapse en un seul chunk). `AgentsPerChunk` = 16 (ctor).
- **Significance brain-stride** : `PassesSignificanceStride` lit `SignificanceLOD`, et ne traite l'agent
  que si `(FrameCounter + AgentIndex) % LOD_BrainStride[band] == 0` (phase par index pour eviter un pic periodique).
  Le **Movement** opte OUT (`UsesSignificanceStride() -> false` : il reste reactif, son cout lointain est rabote par le throttle de tick FPM).
- **Pipeline fusionne** : `BeginPipelinedBatch / RunPreBatch / RunPerAgent / RunPostBatch /
  ClearProcessingFlags / EndPipelinedBatch` permettent au subsysteme de fusionner les 3 processeurs
  dans un seul `ParallelFor` (utilise en Standalone).

### Processing Flags (bitmask)

```cpp
enum class EQAI_AgentProcessingFlags : uint8 {
    None = 0, Movement = 1<<0, StateMachine = 1<<1, Combat = 1<<2, Sensing = 1<<3,
    All = Movement | StateMachine | Combat | Sensing
};
```
`Sensing` est reserve (pas de SensingProcessor batch ; le sensing est evenementiel, voir section 15).

### Le pattern parallel-compute / GameThread-apply

Les processeurs Movement et Combat sont **en deux phases** pour la securite de thread :

- **Phase parallele** (`ProcessAgent`, dans le `ParallelFor`) : appelle le behavior
  (`ComputeMovementInput` / `ComputeCombatDecision`), PURE (pas de `TActorIterator`, pas de
  `ProcessEvent`, pas de line trace bloquant), et capture la decision dans un slot
  (`FQAI_MovementApply` / `FQAI_CombatApply`) indexe par agent.
- **Phase GameThread** (`PostProcessBatch`) : rejoue chaque decision capturee et touche les acteurs,
  composants, physique et `ProcessEvent` (interdits sur worker).

Les appels BP necessaires depuis la phase parallele (ex: cosmetiques de vol des vaisseaux) sont
deferres au GameThread via `AsyncTask` avec un handle faible.

### Profiling

`SCOPE_CYCLE_COUNTER(STAT_QAIProcessorBatch/Agent)`, `FQAI_ProcessorProfiler` (wall-clock),
`FQAI_ProcessingStats` (EMA 0.9/0.1), avertissement si > `MaxProcessingTimeMs` (5 ms). Tous les traces
sont wrappes en `TRACE_CPUPROFILER_EVENT_SCOPE` (`QAI_State_ProcessAgent`, `QAI_Move_ComputeInput`...)
pour de-blober les traces `ParallelFor Task`.

---

## 6. La FSM (State Machine)

### Etat actuel vs intention

Toujours une **FSM plate** (pas de HFSM). Deux niveaux de decision decouples : (1) la FSM plate
(`StateProcessor`) entre etats, (2) les behaviors par archetype (Strategy). C'est un "FSM + Strategy",
pas une HFSM.

### Etats (`EQAI_AgentState`, dans `Data/QAI_State_Struct.h`)

```cpp
enum class EQAI_AgentState : uint8 {
    Idle, Patrol, Pursue, Attack, Taser, Flee, Return,   // les 7 exerces par le StateProcessor
    Wander, Melee, Investigate                            // ajoutes a l'enum, PAS encore references par la FSM
};
```
L'enum a ete renomme `EQAI_PoliceState` -> `EQAI_AgentState` (couvre police ET creatures). Les 3
derniers etats existent pour de futurs comportements mais ne sont pas branches dans le switch.

### Transitions (hardcodees dans `UpdateStateTransitions`)

> Note : un `TArray<FQAI_StateTransition> StateTransitions` existe dans le header mais est **mort** —
> la FSM vivante est le `switch` C++.

| Depuis  | Vers    | Condition |
|---------|---------|-----------|
| Idle    | Pursue / Patrol | cible valide -> Pursue, sinon Patrol |
| Patrol  | Pursue | cible valide |
| Pursue  | Attack | distance <= `AttackEngagementDistance` (60000) |
| Pursue  | Flee   | `ShouldTransitionToFlee` (cible aerienne/injoignable + hostilite/degats recents) |
| Pursue  | Patrol(garde) / Return | cible perdue/timeout |
| Attack  | Taser  | `WantsTaser` (capitaine drone a portee) |
| Attack  | Pursue | melee sol bloque, ou cible hors `AttackEngagementDistance*2` |
| Attack  | Flee / Patrol(garde) / Return | selon perte/distance |
| Taser   | Flee / Attack | apres `StateTimer>3s` |
| Flee    | Attack/Pursue / Return | redevenu non-fuyant + cible, sinon (drones non-capitaine) Return apres 10s |
| Return  | Patrol | a `< DefaultPatrolRadius*0.5` du `PatrolCenter` ou `StateTimer>15s` |

Particularites : **Cyborg et creatures ne reviennent jamais** (`ShouldReturnToPatrol` les exempte ;
ils s'engagent indefiniment dans leur portee). Les **Spaceship** ont un `MaxPursuitDistance` override
de 5 000 000 (ne lachent jamais). Cooldown de transition `StateTransitionCooldown` = 0.5 s.

### Acquisition de cible (`AcquireCreatureTarget`)

- Discrimine creatures `{Sangline, SuperSangline, Flyer, SandDigger, Infected, GazSac, Animal,
  Creature_Autonomus, Nanite}` et police `{Drone, Autonomous, Cyborg}` (les autres recoivent leur cible
  de l'exterieur). Rayon de detection par archetype, clampe a un min projet `MinAggroRangeCm=8000` (80 m) ;
  Cyborg cap a `MaxCyborgAggroRangeCm=12000`.
- **Scan via le spatial hash** (`QueryActorsInRange`), jamais `TActorIterator`. Le scan voisins est
  **stride par index** (`ScanStride=16` : le scan complet coute ~36 ms/frame a 100 animaux ; le keep/drop
  sur la cible courante tourne lui chaque tick).
- **Hysteresis de poursuite** (`PursuitDropMultiplier=1.75`) : garde la cible jusqu'a 1.75x le rayon.
- Regles animales (`ShouldAnimalFleeFrom` / `ShouldAnimalAttackTarget`, proie/predateur/apex).
- A l'acquisition : ponte la cible dans le `CombatComponent` BP via `SetTargetOnCombatComponent`.

### Pauses de patrouille, routes de garde, cohesion d'escouade

`FQAI_AgentState` porte des champs nouveaux pilotes par les processeurs :
- **Pauses de patrouille** : `PatrolPauseUntil` / `NextPatrolPauseAt` (le MovementProcessor centralise
  la pause "regarde autour" : tous les ~6-12 s, duree 0.5-2 s, plafonnee a 2 s ; Flyer/Drone/GazSac exemptes).
- **Route de garde** : `PatrolPointA/B`, `bRouteHeadingToB`, `GuardDefendRadius`. Un Cyborg avec les deux
  endpoints non-nuls marche A<->B (pas de disque aleatoire) et defend son segment. Seede au spawn par
  `UQAI_AgentComponent::SetPatrolRoute`. "Garde" est un **etat runtime**, pas un flag de config/archetype.
- **Escouade** : `SquadAnchor`, `SquadAnchorLocation` (snapshot GameThread en `PreProcessBatch`),
  `SquadSlot`, `SquadSize`. Un garde avec ancre est un suiveur qui se forme en anneau autour de l'ancre.
  Seede par `SetSquadInfo`.
- **Freeze post-attaque** : `MovementFreezeUntil` (le combat fige le mouvement apres une attaque).
- **Specifiques Flyer** : `bFlyerIsAirborne`, `FlyerPullUpUntil`, `FlyerAttackRunUntil`,
  `NextFlyerMeleeCheckTime` (arc plonge -> impact -> ressource).

### Hooks d'etat

`OnStateEntered` / `OnStateExited` pilotent le bitmask `StateFlags` du `PoliceUnitMarkerComponent`
(0x01 patrol / 0x02 combat / 0x04 flee) ET les cosmetiques de vaisseau (`QAI_TrySetFlightModeByName`
"Cruise"/"Combat"/"Landing", `QAI_TryEnsureLandingGear`) — ces appels BP sont deferres au GameThread.

> Stubs : `ExecuteIdleState` est vide ; `IsAgentAlive` retourne toujours `true`. Les flee-destroy
> sont deferres au GameThread (`PendingFleeDestroys` -> `SafeDestroyPawn`) pour eviter un UAF Chaos.

---

## 7. Le systeme de Comportement par Archetype

### Fichiers
- `Public/Behavior/QAI_AgentArchetype.h` (enum)
- `Public/Behavior/QAI_AgentBehavior.h` (interface + decisions)
- `Public/Behavior/QAI_BehaviorHelpers.h` (helpers partages)
- `Public+Private/Behavior/QAI_AgentBehaviorRegistry.{h,cpp}`
- `Private/Behavior/QAI_*Behavior.cpp` (un par archetype)

### Archetypes

```cpp
enum class EQAI_AgentArchetype : uint8 {
    Unknown,
    Drone, Autonomous, Spaceship,                          // police (pipeline d'origine)
    Sangline, SuperSangline, Flyer, SandDigger, Infected,  // creatures
    GazSac, Animal, Creature_Autonomus,
    Cyborg,                                                 // humanoide sol (ex AIC_Cyborg_V2)
    Nanite                                                  // kamikaze blink-bomber (ex AIC_Nanite)
};
```

`QAIArchetype::ResolveArchetype` : si `Archetype != Unknown` -> direct ; sinon inference legacy
(`DroneConfig.bCanFly` -> Drone, sinon Autonomous). **Toujours setter l'archetype explicitement** pour
les nouveaux types.

### Interface de comportement

Deux declinaisons coexistent : `IQAI_AgentBehavior` (UInterface) et `FQAI_AgentBehaviorBase` (classe
C++ pure). La **classe pure est celle utilisee** par le registry (zero overhead GC, destruction
deterministe, `TUniquePtr`).

```cpp
class FQAI_AgentBehaviorBase {
    virtual bool ComputeMovementInput(Registry, AgentIndex, DeltaTime,
        FVector& OutMovementInput, float& OutSpeedScale, FVector& OutTargetToFace) = 0;
    virtual FQAI_CombatDecision ComputeCombatDecision(Registry, AgentIndex, DeltaTime,
        const FQAI_CombatParams& Params);            // defaut: None
    virtual bool WantsTaser(Registry, AgentIndex, const FQAI_CombatParams&) const; // defaut: false
};
```

```cpp
enum class EQAI_CombatAction : uint8 {
    None, Drone_TraceAttack, Drone_Taser, Auto_Melee, Auto_Ranged, Auto_Jump,
    Spaceship_PrimaryWeapons, Spaceship_Missiles, Spaceship_Flares, Auto_Detonate
};
```

### Le BehaviorRegistry

`UQAI_AgentBehaviorRegistry::Initialize()` enregistre **13 behaviors** (tous les archetypes sauf
`Unknown`) : Drone, Autonomous, Spaceship, Sangline, SuperSangline, Flyer, SandDigger, Infected,
GazSac, Animal, Creature_Autonomus, Cyborg, Nanite. `GetBehavior(Registry, AgentIndex)` resout
l'archetype puis lookup O(1). **Pas de fallback** : un archetype sans behavior abandonne le traitement
de l'agent avec une erreur (politique NO FALLBACK).

### Helpers partages (`QAI_BehaviorHelpers`)

Le coeur partage des behaviors sol/creature :

- **`PursueViaFlowField(...)`** : la fonction de steering unique. Resout le but via `ResolveNavGoal`
  (priorite **flee > flank cyborg > cible de poursuite > waypoint de patrouille > centre de retour >
  derniere position connue**), stoppe a l'`AcceptanceRadius`, puis :
  - si l'archetype est "pathfinding-enabled" (`UQAI_Settings::PathfindingEnabledArchetypes`, ou Cyborg avec
    `bAlwaysUsePathfinding`, ou **stuck-escalation**) -> echantillonne le flow field ;
  - sinon -> **steering direct** (ligne droite projetee sur le plan de gravite).
  - La stuck-escalation passe en flow field apres `StuckEscalateGracePeriodSec=0.3 s` de blocage, avec un
    fallback en steering direct si le champ echoue (le pawn continue d'essayer).
- `ResolveNavGoal` : source unique du but de nav (partagee par le steering, le centrage du champ, et la
  publication serveur->client `RepNav*`).
- Roles animaux par nom (`ResolveAnimalCombatRole` : Bear/Rhino/... = Apex, Wolf/Lion/... = Predator,
  Deer/Cow/... = Prey).
- `IsGroundMeleeOnlyAgent`, `GetGroundMeleeVerticalReach`, `IsTargetVerticallyUnreachableForGroundMelee`
  (un melee sol ne grimpe pas vers une cible aerienne).
- `IsValidCombatPawn` (rejette les spectateurs), `IsGazSacDamageTarget` (filtre AOE de gaz),
  `IsAmbientTrafficArchetype` / `IsAerialArchetype` (gating cruise vs hover).

---

## 8. Catalogue des archetypes

Chaque archetype a sa `FQAI_*Config` (dans `QAI_Struct.h`) et son `FQAI_*Behavior`. Les valeurs ci-dessous
sont les defauts de config.

### Police

- **Drone** — vol atmospherique. `FQAI_DroneConfig` (`bIsCaptainDrone`, `bHasTaser`, `AttackRange`,
  `MinFlyHeight`/`MaxFlyHeight`, burst fire `BurstSize`/`BurstInterval`/`BurstCooldown`). Mouvement 3D avec
  bande d'altitude au-dessus de la cible et speed scaling (`FQAI_DroneFlightConfig` du MovementProcessor :
  0.85x pres, 1.75x loin, 2.25x en fuite). Combat : `Drone_TraceAttack` ou `Drone_Taser` (capitaine).
  Coordonne via `DroneCoordinationSubsystem`.
- **Autonomous** — infanterie sol "police". `FQAI_AutonomousConfig` (`MeleeRange`, `RangedRange`,
  `MovementSpeed`, `MaxWalkSlope`). Steering via `PursueViaFlowField`. Combat : `Auto_Melee`/`Auto_Ranged`/`Auto_Jump`.
- **Spaceship** — voir [section 12](#12-spaceship-ai-et-trafic-aerien).

### Creatures (portees du monde BP `Qanga_AI_Controller` / StateTree)

- **Sangline** — `MovementSpeed=800`, `MeleeRange=100`, `JumpMeleeRange=650`, `bCanJump=false` (defaut),
  melee sol. Faction Infected (hostile a tout sauf Infected).
- **SuperSangline** — `MovementSpeed=500`, `MeleeRange=260`, `LeapRange=1800`, plus des **tirs de pics**
  (`SpikeMinRange=990`..`SpikeMaxRange=1500`, `SpikeChance=0.8`, `SpikeCooldown=2`). Le combat projette les
  pics (degats programmes via `ScheduleSuperSanglineSpikeDamage`).
- **Flyer** — `MovementSpeed=600`, vol `MinFlyHeight`/`MaxFlyHeight`, arc **plonge -> impact -> ressource**
  (`MeleeRange=250`, `AttackRange=800`), atterrit (`LandingRange=600`) et redecolle (`TakeoffRange=1500`)
  en basculant le BP `Replicated_OnFly(bool)`. Steering direct (pas de flow field).
- **SandDigger** — `MovementSpeed=350`, melee + `PounceRange=1500`, **terrier** (`BurrowCooldown=8`). Le
  combat maintient l'etat enterre/emerge ; le ground-snap l'ignore (il creuse). Reaction de garde sur
  degats (montage `AN_SandDigger_Guard_Montage`).
- **Infected** — `MovementSpeed=300`, `MeleeRange=160`, `bMeleeOnly=true`. Pousse le joueur (variantes
  Infected_2/3/Boss).
- **GazSac** — ballon de gaz lent (`MovementSpeed=180`, `PatrolSpeedScale=0.3`). S'arme
  (`ArmingDelay=0.8`) puis explose a `ExplosionTriggerRange=250` ; nuage toxique `GasCloudRadius=600` a la
  mort (`ScheduleGazSacServerGasDamage`, filtre `IsGazSacDamageTarget` : ignore Infected + meme faction).
- **Animal** — faune. `FQAI_AnimalConfig` avec roles `EQAI_AnimalCombatRole {Auto, Prey, Predator,
  ApexPredator}` (Auto deduit du nom de l'acteur). Prey fuit, Predator chasse les proies en evitant ses
  semblables, ApexPredator attaque tout. Vitesses par allure (`WalkSpeed`/`RunSpeed`), `WanderRadius=1200`,
  `FleeRadius=8000`.
- **Creature_Autonomus** — creature melee/ranged generique (`MeleeRange=200`, `RangedRange=800`). Peut
  pousser le joueur.

### Speciaux

- **Cyborg** — humanoide sol (ex `AIC_Cyborg_V2`). `FQAI_CyborgConfig` riche : `MeleeRange=250`,
  `MeleeStopRange=1400`, `PatrolRadius=12800`, `PursueSpeedMultiplier=2.0`, `bCanRangedAttack`,
  `RangedRange=1500`, `RangedMinClearance=200`, `bAlwaysUsePathfinding`. C'est le **seul archetype en flow
  field par defaut** (`PathfindingEnabledArchetypes = {Cyborg}`) : il patrouille autour d'objectifs et a
  besoin du champ pour contourner les obstacles. Les cyborgs de combat (Pirates, IcLabs) laissent
  `bAlwaysUsePathfinding=false` (poursuite directe + escalation au blocage). Combat pilote directement le
  systeme d'arme BP (`Combat_1stTrigger` en rafales, `Combat_Melee`, `Combat_EquipMelee` on-change). Tir
  via `QAI.DirectFire` (subsysteme C++ de balles, pas le graphe BP par tir).
- **Nanite** — kamikaze blink-bomber (ex `AIC_Nanite`). `FQAI_NaniteConfig` (`DetonationRange=250`,
  `ArmDelay=2`, `DetonationDamage=100`, `DetonationRadius=500`). Fonce en ligne droite puis emet
  `Auto_Detonate` : le CombatProcessor arme un timer, joue la FX BP, applique le blast radial et
  s'autodetruit.

Champs communs (`FQAI_AgentData`) : `AttackCooldown=1`, `AggressionDuration=30`,
`RotationInterpSpeedOverride`, `bCombatEnabled`, `PoliceFaction`, et `bCanPushPlayer` (peut bloquer/pousser
le joueur ; defaut false — true pour CreatureAutonomus/SandDigger/Infected_2/3/Boss).

---

## 9. Pathfinding par Flow Field

### Fichiers
- `Public/Pathfinding/QAI_FlowField.h` (structures de donnees)
- `Public+Private/Pathfinding/QAI_FlowFieldManager.{h,cpp}`

> Il n'y a **pas de NavMesh** dans QANGA (planetes LWC + serveur sans meshes). Le flow field est le seul
> systeme de pathfinding du crowd. (La seule survivance NavMesh est `ProjectPointToNavigation` dans le
> calcul de flanking — incoherence a nettoyer.)

### Principe

Champ de flux 2D par but, aligne sur le "haut local" de la planete. Walkability echantillonnee par
sphere-sweeps descendants le long de la gravite ; cout propage depuis le but par Dijkstra (min-heap) ;
une direction 8-connectee stockee par cellule ; le champ est **partage** entre agents proches du meme but.

```
FQAI_FlowField : grille CellsPerSide x CellsPerSide (defaut 80x80 = 6400 cellules), CellSize=50cm
  FQAI_FlowCell : { uint8 bWalkable:1; uint16 Cost; uint8 DirectionIdx (0..7, 8=none);
                    float GroundLocalZ; EQAI_FlowCellFail FailReason }
  EQAI_FlowCellFail : None(walkable), NoHit, Penetrating, SteepNormal, NoHeadroom
FQAI_FlowFieldHandle : { int32 Id } opaque
```

### `UQAI_FlowFieldManager` (UActorComponent, sur l'hote QAI_PathfindingManager)

- API : `RequestField(Goal, Center, PawnHint)` (find-shareable ou alloc, `++RefCount`),
  `SampleDirection`/`SampleDirectionDiag` (O(1), direction tangente vers le but ; recovery par
  ring-search en poche serree), `FindReachablePatrolPoint`, `RetargetField` (re-seed Dijkstra sans
  re-tracer), `ReleaseField`.
- **Build asynchrone** (`BuildField`) : `AsyncSweepByChannel` (sphere `AgentRadius`, `bTraceComplex=true`)
  pour le sol puis le headroom, polles par `FTraceDelegate`. Machine a etats
  `DispatchFloor -> AwaitingFloor -> DispatchHeadroom -> AwaitingHeadroom -> Finalize`. Remplace l'ancien
  `LineTraceSingleByChannel` bloquant (~740 ms GT/champ).
- `FinalizeField` : flood de composantes connexes (filtre de pas symetrique), relocalise la graine
  Dijkstra dans la composante du pawn si le but est bloque, BFS distance-to-obstacle pour l'evitement de
  mur, Dijkstra min-heap, swap `PendingCells -> Cells`.
- **Hot-swap sans stall** : le StateProcessor gere `FlowFieldHandle` + `PendingFlowFieldHandle` — un
  rebuild ne montre jamais un champ vide (l'echantillonnage continue sur l'ancien jusqu'a ce que le
  nouveau ait des cellules).
- **Allocation paresseuse** : seul Cyborg consomme le champ inconditionnellement ; les autres archetypes
  ne declenchent un build qu'a la stuck-escalation, et le champ est relache des que le pawn redecolle
  (`UnusedFieldGraceSec=5`).
- **Thread-safety** : `static FCriticalSection GQAIFlowFieldCS` protege `Fields` car
  `RequestField`/`SampleDirection*`/`AllocField` sont appeles depuis le `ParallelFor` (>=100 agents).
- **Stuck tracking** : `MovementProcessor::TickStuckTracking` (GameThread) detecte le blocage projete sur
  la direction-au-but (un wall-slider montre du deplacement brut mais zero progres projete) ; stampe
  `StuckSinceTime`, que `PursueViaFlowField` consulte.

CVars : aucun. Tout est en `UPROPERTY EditAnywhere` sur le manager (`CellsPerSide`, `CellSize`,
`AgentRadius`, `MaxStepHeightCm`, `MinFloorNormalDot`, `WallAvoidRange`...). Reglage runtime du debug via
`qai.DrawDebugPaths` (CVar `ECVF_Cheat`) ou le tag acteur `QAI_DrawPath`. Le setting projet
`bAsyncFlowFieldTraces` (defaut true) bascule entre traces async (physique) et bloquantes (diagnostic).

---

## 10. Le tier Impostor (VAT) et la dormance

### Fichiers
- `Public+Private/Impostor/QAI_ImpostorSubsystem.{h,cpp}`
- Bakes references depuis `UQAI_Settings.ImpostorBakes` (`FQAI_ImpostorBake`)

### Principe

`UQAI_ImpostorSubsystem` (`UTickableWorldSubsystem`) rend les agents **dormants** comme des instances
`InstancedStaticMesh` jouant un bake AnimToTexture (VAT, mode bone) au lieu d'un pawn skeletique vivant.
Un "impostor" est un agent dont le **pawn existe toujours** (acquisition, LOS, trackers, kill-count
fonctionnent) mais qui est cache + fige (pas d'anim, pas de tick FPM, capsule QueryOnly).

- **Demotion** : temporelle — pas de hit/interaction pendant `QAI.ImpostorDemoteAfter` secondes
  (evaluee par `UQAI_AgentComponent`).
- **Promotion** : sur **hit / interaction uniquement** (degats, ou rayon de vue joueur / ecran /
  proximite) — jamais par simple proximite pour les creatures. Voir `PromoteFromImpostor(reason)`.
- **Standalone only** (et `NM_Client` pour la presentation des creatures repliquees) : aucun ISM/VAT sur
  serveur dedie/listen.

### Architecture
- `EnsureRendererForMesh` : un `FQAI_ImpostorBakeRenderer` (ISM + textures bone-pos/rot/weight + table
  d'anims) par mesh skeletique unique, lookup via `UQAI_Settings::FindBakeForMesh(Mesh->GetSkinnedAsset())`.
  Le 1er ISM devient le root du `RendererHost`, les suivants s'attachent en **KeepRelativeTransform**
  (fix LWC : un attach KeepWorld cloue l'enfant a l'origine -> dechirement fp32 de la geometrie).
- **Ancrage camera** : `RendererHost` re-ancre sur la camera quand l'instance-local depasse ~1 km
  (garde les coords fp32 sub-mm a toute distance ; fix LWC du "grid flottant").
- **Stride-matching** : `WritePlayback` choisit le cycle de locomotion (FLOOR sur `AnimSpeeds`),
  `PlayRate = clamp(SmoothedSpeed/AnimSpeed, 0.45, ~2.5)`, avec hysteresis asymetrique (cycle rapide
  immediat, lent/maintenu attend 0.5 s — fix "freeze-then-slide"). Une entree de vitesse 0 encode la
  pose de combat READY (cible vivante, immobile).
- **Variantes de couleur** : recolor cyborg par palette (`Color_T*`) et tons creature (Sentinel/Infected)
  via des ISM freres.
- **Props** (`CaptureProps`) : armes/equipement captures en ISM rigides ou bone-driven GPU
  (`M_QAI_ImpostorPropBoneWPO`). L'arme active reste fonctionnelle (cachee apres capture).
- **Corps statique (vaisseaux)** : `RegisterDormantStatic` / `CaptureShipBody` capture les N plus gros
  meshes statiques visibles (`QAI.AerialTraffic.ImpostorMaxParts=12`). Sur serveur, reussit SANS ISM
  (dead-reckoning headless ; la coque repliquee est le corps visible).
- **Ground-snap symetrique** : `AdjustToGround` + `TraceGround` avec clamps up/down symetriques
  (`MaxUpSnap`/`MaxDownSnap`) — fix de l'asymetrie "fall-through-world". Ignore tous les pawns impostors
  traces (pas de "tour de cyborgs").
- **Tick** : pass parallele (`ParallelFor`) composant `BodyTransform = MeshRelative * GetAgentTransform`
  (cull cone de vue, scratch par entree), puis pass serie GameThread (agregation, `WritePlayback`,
  re-capture 1 Hz), re-ancre hote, et `BatchUpdateInstancesTransforms` **gate** (skip si rien n'a bouge —
  evite la reconstruction du render-proxy RT / `CreateRHIBuffer`).

### CVars (toutes `QAI.Impostor*` / `QAI.AerialTraffic.*`)

| CVar | Defaut | Role |
|------|--------|------|
| `QAI.Impostor` | 1 | active le tier (demote -> pawn cache + ISM/VAT). |
| `QAI.ImpostorDemoteAfter` | 5.0 | secondes sans hit/interaction avant demotion. |
| `QAI.ImpostorGroundTracesPerFrame` | 8 | traces de sol par frame (round-robin sur le crowd). |
| `QAI.ImpostorFocusRangeCm` | 300 | portee du rayon de vue qui promeut un impostor focus. |
| `QAI.ImpostorScreenPromoteCm` | 5000 | promotion si dans le viewport sous cette distance. |
| `QAI.AerialTraffic.ProximityPromoteCm` | 15000 | promotion d'un vaisseau de trafic par proximite joueur. |
| `QAI.ImpostorShadows` / `QAI.ImpostorProps` / `QAI.ImpostorPropsGPU` | 1 | toggles A/B (ombres, props, props GPU). |
| `QAI.ImpostorFreezeUpload` / `QAI.ImpostorSkipMeshMatch` / `QAI.ImpostorDebugDraw` | 0 | diagnostics. |
| `QAI.AerialTraffic.ImpostorMaxParts` | 12 | meshes statiques captures par vaisseau dormant. |

> Un mesh sans bake (`ImpostorBakes`) "ne demote jamais" (logge une fois) et reste un pawn vivant — le
> tier impostor no-op silencieusement.

---

## 11. Significance / LOD et paliers d'allegement

C'est une couche de performance entierement nouvelle, repartie entre le subsysteme, le registre, les
processeurs et `UQAI_AgentComponent`. Objectif : faire scaler le head-count sous-lineairement (100 AI
en combat <= ~20% de baisse de FPS).

### Significance (autorite : `UpdateAgentSignificance`, GameThread, chaque frame)

Stampe `FQAI_AgentHotData.SignificanceLOD` (byte) depuis la distance au viewer le plus proche. 4 bandes
(`UQAI_Settings` : `LODDistance1Cm=4000`, `LODDistance2Cm=9000`, `LODDistance3Cm=18000`). Un agent en
combat ne descend jamais sous `CombatMaxLOD=1`. Throttle combat additionnel : seuls les
`QAI.CombatFullRateCount=8` combattants les plus proches gardent le plein regime ; les autres sont
forces a `QAI.CombatThrottleBand=2` (stride x4) meme a bout portant.

### Leviers (par bande)

| Levier | Mecanisme | CVar |
|--------|-----------|------|
| **Brain stride** | `LOD_BrainStride={1,2,4,8}` : State+Combat re-decident moins souvent (Movement reste reactif). | (intrinseque) |
| **Tick FPM** | `LOD_MoveTickInterval={0,0.05,0.1,0.25}` : le FloatingPawnMovement tick a intervalle. | - |
| **Owner-tick stride** | bande 0 = tick BP du pawn ; bande >=1 = tick BP **tue** (defaut). | `QAI.OwnerTickStride` (-1 = significance) |
| **Partage d'anim** | bande 1+ leader-pose sur des channels partages idle/walk/run. | `QAI.AnimShare` (1) |
| **Strip skin cache** | les meshes AI sautent le GPU skin cache. | `QAI.SkinCacheOff` (1) |
| **Far-pawn collision** | > `LODDistance1` -> capsule en Overlap ; > `LODDistance2` -> root en QueryOnly. | `QAI.FarPawnOverlap` (1), `QAI.FarPawnQueryOnly` (1) |
| **Lean cosmetics** | remplace les item-acteurs cosmetiques (non-arme) par de simples meshes ; tue les boucles Delay latentes + timelines de recul. | `QAI.LeanCosmetics` (1) |
| **Agent tick banding** | le `UQAI_AgentComponent` se tick a 10 Hz en bande 2+. | `QAI.AgentTickBanding` (1) |
| **Tier impostor** | demote -> pawn cache + ISM/VAT (le palier le plus economique, supplante tout). | `QAI.Impostor` (1) |

La plupart de ces leviers sont **standalone-only** (le MP a besoin d'un pass de spec visuelle repliquee
d'abord). Plusieurs classes lean sont substituees au spawn via `UQAI_Settings.LeanClassRedirects`
(`ResolveSpawnClass`) : QAI ne spawne jamais directement les BP de prod, il les route vers leur jumeau lean.

---

## 12. Spaceship AI et trafic aerien

### Fichiers
- `Public/Behavior/QAI_SpaceshipBehavior.h` / `Private/.../QAI_SpaceshipBehavior.cpp`
- `Public+Private/Spaceship/QAI_SubSystem_Spaceship.{h,cpp}`
- `Public+Private/Spaceship/QAI_AerialTrafficSubsystem.{h,cpp}` + `QAI_AerialTrafficSettings.{h,cpp}`
- `Public/Spaceship/QWeaponCoordinatorComponent.h`, `QSpaceshipObstacleAvoidanceComponent.h`,
  `QSpaceshipMovementInterface.h`, `QSimpleProjectile.h`, `QSpaceshipAIConfigDataAsset.h`,
  `QSpaceshipAITypes.h`, `QShipAILog.h`

### SpaceshipBehavior

`FQAI_SpaceshipBehavior` a ses propres sous-etats de mouvement internes : **Idle, Patrol, Hunt, Combat,
Evade, Return** (`Execute*Movement`), avec obstacle avoidance (`GetObstacleAvoidanceInput`), suivi
d'altitude relative au joueur (`UpdateTargetAltitudeForPlayerTracking`) et generation de waypoints.
`FQAI_SpaceshipConfig` : `CombatRange=60000`, `MaxSpeed=5 555 556` (200 000 km/h), `TargetAltitude=50000`,
`CloseCombatStandoff`/`BrakeDistance`, `TurnRate`. Combat : `Spaceship_PrimaryWeapons` / `_Missiles` /
`_Flares` via le `WeaponCoordinator` (par reflexion, `FQAI_SpaceshipCombatConfig` : intervalles MG/roquettes,
flares `FlareDeployChance=0.6`, missiles).

Le mouvement vaisseau est dispatche par le MovementProcessor via l'interface `QSpaceshipMovementInterface`
(reflexion : `SetSpeed`, `MaintainAltitude`, `OrientToTarget`, `ApplyDirectMovementInput` / `MoveToLocation`),
ou via `UVehicleMovementComponent` (FlyVehicleMovement) sinon.

> Regle thread-safety critique : le `ProcessAgent` du spaceship tourne en `ParallelFor` a >=100 agents —
> il ne doit pas faire de `TActorIterator` (utiliser `CachedWorldScapeRoot`) ; le `ProcessEvent` flyer
> reste un crasher latent a corriger.

### Trafic aerien ("living sky")

`UQAI_AerialTrafficSubsystem` (serveur) maintient un budget de vaisseaux transitant le ciel autour de
chaque joueur : chaque vaisseau **entre de loin**, traverse, et est despawne au-dela de
`DespawnDistanceM`. Spawn via `UQAI_SubSystem::SpawnAerialAgent(ShipClass, Loc, Rot, FQAI_ShipAgentParams)`
(ajoute interface mouvement + obstacle avoidance + weapon coordinator + agent QAI + lifecycle).

`UQAI_AerialTrafficSettings` (DeveloperSettings, `[/Script/QAI.QAI_AerialTrafficSettings]`) :
`MaxShipsPerPlayer=6`, `SpawnIntervalSeconds=1.5`, `Min/MaxSpawnDistanceM`, `Min/MaxAltitudeM`,
`AltitudeLowBias`, `MaxRouteOffsetM`, `DespawnDistanceM=9000`, et un `Roster` de
`FQAI_TrafficRosterEntry { Faction, Ships[], bCombatEnabled, Weight }`. Le roster est **bake en C++** (les
lignes `+Roster` de l'ini ne survivaient pas au cook -> fallback police).

Les vaisseaux de trafic sont des **impostors statiques dormants qui volent vraiment** : le
MovementProcessor les fait dead-reckoner analytiquement (sans mesh, donc OK sur serveur dedie), tient
l'altitude radiale (`QAI.AerialTraffic.MinClearanceCm=40000`), re-tangente la velocite sur la courbe
planetaire, et skip le cerveau dormant (`QAI.AerialTraffic.SkipDormantBrain=1`).

---

## 13. Architecture Client/Serveur et multijoueur

### Separation des responsabilites

```
SERVEUR (Dedicated/Listen/Standalone autorite)     CLIENT
+-------------------------------------------+   +------------------------------------------+
| State + Movement + Combat (SoA)           |   | State + Movement LOCAUX/PROXY uniquement |
| sync transforms acteur->registre          |   |   (unites police a marker)               |
| publie le but de nav (RepNav*)            |   | AutoDiscoverAgents()                     |
| LifecycleTick (cull + caps)               |   | UpdateClientTrafficShips (leanify)       |
|                                           |   | UpdateClientCreatureImpostors (VAT)      |
+-------------------------------------------+   +------------------------------------------+
         |  Replication UE (IsAlive, PoliceUnitMarker, RepNav*, transforms)  ^
         +------------------------------------------------------------------>+
```

### Le serveur dedie n'a PAS de meshes (par design)

C'est la contrainte centrale du MP. Cote serveur, FindFloor/grounding/line-traces/collision ne resolvent
jamais la geometrie. Consequences et mitigations :

- **Plancher analytique WorldScape** : le mouvement utilise `GetPawnDistanceFromGround`
  (collision-independant) comme plancher dur (FPM toutes les 0.1 s + impostors dormants), avec
  discrimination grotte/bunker (`IsBelowWorldScapeSurfaceSheltered`).
- **Pont de nav client** (`ClientNavDir` / `ClientNavExpiry`) : le serveur, aveugle aux murs, publie le
  but (`RepNav*`) ; le **client le plus proche** de chaque AI (qui a, lui, les murs) fait tourner le meme
  `UQAI_FlowFieldManager`, echantillonne une direction wall-aware et la renvoie via
  `UQAI_Client::Server_ReportAINav`. Le MovementProcessor l'utilise a la place de sa direction aveugle
  quand elle est fraiche. N'est envoye que quand le chemin du client est bloque (sinon silence -> le
  steering serveur direct vers le but suffit, comme en offline).
- **Floor-oracle client** : le client sweepe le sol (`MarkClientGrounded`) et renvoie des corrections de
  hauteur ; le FPM client neutralise la chute verticale tant que `ClientGroundedUntil` est actif.

### Presentation cliente

- **Creatures** : elles bougent deja correctement en MP (serveur-autoritatif + repliquees + facing-only
  cote client). Pour la parite perf avec le standalone, `UpdateClientCreatureImpostors` rend les
  creatures lointaines en impostors VAT cote client (entree de presentation, sans agent registre, rendue
  sur la transform repliquee — ne peut jamais combattre l'autorite serveur). Promotion sous
  `QAI.Client.CreatureImpostorPromoteCm=600`.
- **Vaisseaux de trafic** : serveur-autoritatifs, dead-reckonnes analytiquement et repliques (hull
  transform). Cote client, `UpdateClientTrafficShips` (`LeanifyTrafficShip`) strip la copie cliente et
  tue ses ticks de vol pour que `ReplicatedMovement` soit le seul driver. Le serveur pousse aussi la pose
  pleine-frequence dans `SmoothTSync.ForceTransform` (`QAI_PushDormantShipToSmoothTSync`) pour un interp
  client doux.
- Les proxies **Cyborg** laissent la replication de mouvement engine posseder rotation+position ; les
  creatures simples recoivent un facing local (`TickClientProxyFacing`).

### Cycle de vie serveur

`LifecycleTick` (~0.5 Hz) : enregistre les pawns joueurs dans le spatial hash (les joueurs n'ont pas de
`UQAI_SensingComponent` ; c'est le SEUL chemin qui les rend visibles a `AcquireCreatureTarget`), cull des
AI au-dela de `AI_SpawnDistanceCm=50000` du joueur le plus proche, reassigne l'autorite. Caps :
`MaxAIPerPlayer=50`, budget global de spawn `MaxSpawnsPerFrame=4` (`TryConsumeSpawnBudget` evite le flood
de proxies Chaos). `UnregisterQAI_Agent` relache aussi le flow field tenu par l'agent (fuite corrigee).

---

## 14. Systeme de factions

### Fichiers
- `Public+Private/Faction/QAI_Faction.{h,cpp}` — **module dedie NOUVEAU**

Remplace l'ancienne logique de faction dupliquee dans `SensingComponent` et `SpatialHashGrid`. Source
unique de verite : `namespace QAI_FactionLib` (+ wrapper BP `UQAI_FactionLibrary`).

```cpp
enum class EQAI_Faction : uint8 {
    None = 0, IcLabs = 1, Dissidence = 2, Infected = 3, Pirate = 4,
    Animal = 5, Rogue = 6, Voss = 7   // miroir du BP E_Factions — NE PAS reordonner
};
```

- `GetFactionForActor` : **composant + reflexion**, plus de string-match par nom de classe. (1) si le
  pawn porte un `UQAI_PoliceUnitMarkerComponent` -> `IcLabs` ; (2) sinon trouve le composant dont la
  classe contient "CombatComponent" (le seul match de nom restant, juste pour *localiser* le composant BP)
  et lit sa propriete `Faction` par reflexion ; (3) sinon `None`.
- `AreFactionsHostile(A, B)` : matrice `static constexpr bool[8][8]` directionnelle (ligne = self), lue
  telle quelle sur la diagonale (`[Rogue][Rogue]=1` : deux Rogues se battent).
- `IsActorHostileTo(Self, Other)`.

Matrice (ligne = self, 1 = hostile) :

| self \ other | None | IcLabs | Diss | Inf | Pirate | Animal | Rogue | Voss |
|---|---|---|---|---|---|---|---|---|
| **None**       | 0 | 0 | 0 | 1 | 1 | 0 | 1 | 1 |
| **IcLabs**     | 0 | 0 | 1 | 1 | 1 | 0 | 1 | 1 |
| **Dissidence** | 0 | 1 | 0 | 1 | 0 | 0 | 1 | 0 |
| **Infected**   | 1 | 1 | 1 | 0 | 1 | 1 | 1 | 1 |
| **Pirate**     | 1 | 1 | 0 | 1 | 0 | 1 | 1 | 0 |
| **Animal**     | 0 | 0 | 0 | 1 | 1 | 0 | 1 | 0 |
| **Rogue**      | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| **Voss**       | 1 | 1 | 0 | 1 | 0 | 0 | 1 | 0 |

`Rogue` (slot 6, ex-"Guard") = cyborgs malfonctionnants/hackes, hostiles a TOUT (y compris eux-memes).
`Voss` (7) = humanoides derives des pirates. Les gardes IcLabs sont des unites de **faction IcLabs**, pas
une faction dediee. Les decisions de plus haut niveau (wanted-level, PvE/PvP, SafeArea) restent dans le
BP `CombatComponent::CheckTargetIsAllowedCombat` — ce module est de la relation de faction pure.

---

## 15. Spatial Hash Grid et Sensing

### Fichiers
- `Public+Private/Performance/QAI_SpatialHashGrid.{h,cpp}`
- `Public+Private/Sensing/QAI_SensingComponent.{h,cpp}`
- `Public/Sensing/QAI_SensingFunctionLibrary.h`, `QAI_Sensing_Vision_Library.{h}`, `QAI_Sensing_Vision_Struct.h`
- `Public+Private/Sensing/QAI_SensingEmitterComponent.{h,cpp}` (stub)
- `Public/Data/QAI_Sensing_Struct.h`

### Spatial Hash Grid

Partitionnement spatial 2D (XY ; le Z est filtre en distance 3D mais pas hashe) pour des requetes de
proximite O(1). `FQAI_SpatialHashGrid` thread-safe (`FCriticalSection`, snapshot-and-release pendant le
filtrage). `CellSize=1500`, primes de hash `(73856093, 19349663, 83492791)`. Le subsysteme tickable
`UQAI_SpatialHashGridSubsystem` re-pousse chaque acteur enregistre par tick, maintenance ~1 Hz. API BP :
`RegisterActor`, `QueryActorsInRange`, `QueryHostileActorsInRange` (delegue a `QAI_FactionLib`).

C'est la source unique de candidats pour l'acquisition de cible du StateProcessor (jamais
`TActorIterator` sur worker). Les pawns joueurs y sont enregistres par `LifecycleTick`.

### SensingComponent

`UQAI_SensingComponent` (ActorComponent, hors pipeline SoA, timer-driven a `SensingFrequency=0.5s`) :
detecte les pawns via la grille spatiale (fallback `SphereOverlapActors`), classe par hostilite
(`QAI_FactionLib`), broadcast `OnPawnDetected`/`OnHostilePawnDetected`. Toute la logique de faction est
deleguee a `QAI_FactionLib` (le string-sniffing par nom de classe a ete supprime).

**Static defender** (`bAutoAcquireCombatTarget`, serveur-only) : un cerveau minimal qui auto-acquiert une
cible hostile et applique un tir hitscan serveur depuis le `SM_WeaponRef`/socket `FirePoint` (vise la
socket `Head`), via `UQWeaponBulletSubsystem::ServerFireBullet` + `SpawnBulletTracer`. CVars :

| CVar | Defaut |
|------|--------|
| `QAI.StaticDefenderDirectFire` | 1 |
| `QAI.StaticDefenderFireInterval` | 0.5 |
| `QAI.StaticDefenderFireDamage` | 12.0 |
| `QAI.StaticDefenderFireRange` | 30000 |

### Vision

Deux familles de structs/libs de vision **coexistent** (heritage du fold-in de l'ancien plugin QSensing) :
la nouvelle (`FQAI_Vision`/`UQAI_Sensing_Vision_Library`) et l'ancienne
(`FQAI_Sensing_Vision`/`UQAI_SensingFunctionLibrary`). Calculs d'angle, score FOV, projection ecran,
line-traces LOS. Son / electromagnetique / odorat sont des **structs stub** (declares, pas implementes).
`UQAI_SensingEmitterComponent` est un **stub vide** (tick active mais ne fait rien).

---

## 16. Composants de mouvement et d'animation

### QAI_FloatingPawnMovement
Le mover principal de presque tous les pawns QAI (creatures, cyborgs, drones). Etend
`UFloatingPawnMovement`, tick `TG_PrePhysics`, replique par defaut.
- **Gravite spherique/analytique** (`SimulateGravity`, `GravityForce`), integration le long de la gravite.
- **Plancher dur WorldScape** : `ResolveWorldScapeTerrainPenetration` toutes les 0.1 s (cache du root,
  re-resolu toutes les 2 s), avec discrimination grotte/ceiling et `GetAgentRestOffset`.
- **Replication** : input (`Client_SendInput`, RPC serveur, ~0.05 s) et location (~0.15 s) ; transfert
  d'ownership (`QAI_ChangeOwner`, `QAI_GetClientIsOwner`).
- **Proxy facing** (`bServerAuthoritativeCreature`) : les Cyborgs laissent l'engine posseder
  rotation+position ; les creatures simples recoivent `TickClientProxyFacing` (vers le but de nav repliquE).
- `Auto_RotateOwner` toggleable, `LockMovement`, `Jump` (fenetre `JumpGroundingSuppressUntil=2.5s`),
  `MovementOnlyOnGround` **desactive** (trace planetaire non fiable).

### QAI_BasicMovement
Mover physique standalone (gravite/sweep/step/jump, modes "Ground/Surface/Fly/AntiGravity" decrits en
commentaire mais non enum-switches). Composant alternatif/legacy, pas sur le chemin principal. (Contient
des champs morts et des fuites de `FHitResult` — a nettoyer.)

### QAI_ShipMovementSimulation
**Stub vide** (tick active, aucune logique). Le mouvement vaisseau passe par FlyVehicleMovement +
`QSpaceshipMovementInterface`, pas par cette classe.

### QAI_PoliceUnitMarkerComponent
Replique `TargetActor` (mirroir client-side -> `State.PursuitTarget`) et `StateFlags` (8 bits ; semantique
0x01/0x02/0x04 posee par les hooks d'etat), multicast les FX de tir et le jump (`MC_PlayShotVFX`,
`MC_PlayShotVFXEx` -> `UQWeaponBulletSubsystem::SpawnBulletTracer`, `MC_ApplyJumpImpulse`).

### QAI_AnimInstance / QAI_AnimShareManager (NOUVEAU)
- `UQAI_AnimInstance` : anim instance "lean" thread-safe qui remplace l'ALS BP sur les pawns AI (pas
  d'event graph, pas de foot-IK). Quasi tout tourne en `NativeThreadSafeUpdateAnimation`. Expose Speed,
  Direction, AimPitch/Yaw, `OverlayAlpha`, `bCombatReady`, `GunGrip*`, `ForcedSpeed`, `LocomotionPlayRate`.
- `UQAI_AnimShareManager` (`UTickableWorldSubsystem`, standalone-only) : evaluation d'anim partagee
  "bucketed". Les agents de bande lointaine leader-pose sur un petit set de meshes-driver caches
  (idle/walk/run, `GChannelDefs={0,165,350}`), evitant ~100 evaluations de graphe d'anim
  (`SetLeaderPoseComponent` + tick desactive sur le suiveur, refresh batch 1/3 par frame ~20 Hz).

---

## 17. Spawners et cycle de vie

### Fichiers
- `Public+Private/Spawner/QAI_AgentSpawner.{h,cpp}`
- `Public+Private/Spawner/QAI_GuardPatrolSpawner.{h,cpp}`
- `Public/Data/QAI_PawnConfig.h`

### AQAI_AgentSpawner (port C++ du BP `AI_Spawner`)

Spawner generique : `AI_PawnClasses` (pick aleatoire), `SpawnQuantity`, `DelayBetweenEachSpawn`,
`SpawnStartJitter=3s` (anti-flood Chaos), `RespawnTime`, `OutRangeDelayDestroy`, detection joueur par
distance adaptative (`SpawnDistanceCm=20000`), cull `AI_MaxDistanceFromSpawnerCm=60000`, ciblage
(`CustomTargetActor`, `bForceSetCustomTarget`). Champs de parite legacy (`bHasSpawnerToDestroy`,
`SpawnerLife`, `bOverrideFaction`/`FactionOverride`, `bPropagateQuestComponentToAI`). Hooks BP (noms
identiques au BP legacy pour le reparenting) : `WorldObjectiveAllowSpawner` (gate quete),
`OnSpawned`, `OnPropagateQuestComponent`, `TryCompleteMissions`, `OnDeathAI`, `UpdateSpawnerVisibility`.
Delegues natifs : `OnAllKilled`, `OnSpawnedAI`, `OnAISpawnedDestroyed`, `OnSpawnerDeath`. Enregistre ses
pawns aupres de `UQAI_SubSystem` (`AddAI_Spawned`).

### AQAI_GuardPatrolSpawner

Sous-classe qui spawne des **gardes Cyborg armes** et seede a chacun une route A<->B au spawn
(`SetPatrolRoute`). `GuardDefendRadius=4000`, `NumPatrolGroups` x `GuardsPerGroup` (chaque groupe couvre
une sous-section du corridor), `CaptainPawnClass` optionnel (1er garde d'un groupe >=3 = capitaine).
Endpoints draggables en viewport (`PatrolStart`/`PatrolEnd`) ou par acteurs
(`PatrolStartActor`/`PatrolEndActor`). Faction par defaut IcLabs. Respawn maintien-de-population
(`ShouldRespawnAfterDeath` : refill des qu'un slot s'ouvre). Forme des escouades (anchor par groupe ->
`SetSquadInfo`).

### Authoring : UQAI_PawnConfig

`UPrimaryDataAsset` portant un unique `FQAI_AgentData`. Un asset par archetype de pawn (ex
`QAI_PawnConfig_Sangline`). Le pawn porte un `UQAI_AgentComponent` referencant ce config ; a l'auto-
discover, le config est copie dans le registre SoA et le pipeline prend la main (plus d'`AAIController`
custom). `Archetype` DOIT etre set.

### UQAI_AgentComponent (le hub par pawn)

Implemente `IQAI_AgentInterface` (marqueur "ce pawn est pilote par QAI", remplace le cast vers
l'ancien `Qanga_AI_Controller`). API : `RequestMoveTo`, `SetPursuitTarget`, `SetPatrolRoute`,
`SetSquadInfo`, `NotifyAgentDied`, `BeginAsDormantImpostor`/`PromoteFromImpostor`/`IsImpostorDormant`,
accesseurs de nav repliquee. C'est lui qui pilote la machine demote/promote impostor, les paliers de
significance/LOD (owner-tick stride, throttle composants, anim-share, strip skin cache, far-pawn
collision), les cosmetiques lean, le facing combat Cyborg/Autonomous, et le pont de nav client/serveur
(`TickServerNavPublish` ~5 Hz, `TickClientNav` ~15 Hz). `IsAlive` replique (`OnRep_IsAlive` rejoue le
teardown).

---

## 18. Configuration, Settings et CVars

### UQAI_Settings (DeveloperSettings, `Project Settings > QAI`)

```cpp
// SubSystem
bool Enabled = true;
ETickingGroup RealTimeTickGroup = TG_PrePhysics;
bool bDisableSubsystemTick = true;
// Agent
int32 MaxAgent = 4096;  int32 MaxAgentPerClient = 512;  int32 MaxDistanceAgent = 100000;
// Significance / LOD
bool bEnableSignificanceLOD = true;
float LODDistance1Cm = 4000;  LODDistance2Cm = 9000;  LODDistance3Cm = 18000;
int32 CombatMaxLOD = 1;
// Impostor
TArray<FQAI_ImpostorBake> ImpostorBakes;            // 1 par mesh skeletique unique
// Lean classes
TArray<FQAI_LeanClassRedirect> LeanClassRedirects;  // ProdClass -> LeanClass au spawn
// Logging / Pathfinding
bool bVerboseLogging = false;
bool bAsyncFlowFieldTraces = true;
TSet<EQAI_AgentArchetype> PathfindingEnabledArchetypes = { Cyborg };
```
Helpers statiques : `IsVerboseLogging` (fast path), `SetVerboseLoggingRuntime`, `IsPathfindingEnabledFor`,
`FindBakeForMesh(USkeletalMesh*)`, `ResolveSpawnClass(UClass*)`. Macro de log :
`QAI_VLOG(...)` (court-circuit gratuit si verbose off ; a utiliser partout en site per-tick/per-agent —
jamais `UE_LOG(LogQAI, ...)` brut).

`UQAI_AerialTrafficSettings` est un DeveloperSettings separe (section 12).

### Index des CVars (par domaine)

- **Pipeline / diag** : `QAI.Disable.{State,Movement,Combat}`, `QAI.Debug`, `QAI.Verbose`,
  `QAI.DumpAgents/DumpSpawners/DumpFPM/DumpBlueprintAI`, `QAI.StressBench`, `QAI.StressBenchDamage`.
- **Combat throttle** : `QAI.CombatFullRateCount` (8), `QAI.CombatThrottleBand` (2), `QAI.CrowdFirePauseScale` (6).
- **Direct-fire cyborg** : `QAI.DirectFire` (1), `QAI.DirectFireSpreadDeg` (1.5),
  `QAI.DirectFireSoundMaxPerFrame` (1), `QAI.DirectFireSoundFrameInterval` (2),
  `QAI.DirectFireSoundMinInterval` (0.30), `QAI.DirectFireSoundMaxDist` (2500), `QAI.DirectFireOverflowDelayScale` (2).
- **Static defender** : `QAI.StaticDefender{DirectFire,FireInterval,FireDamage,FireRange}`.
- **Impostor** : `QAI.Impostor`, `QAI.ImpostorDemoteAfter`, `QAI.ImpostorGroundTracesPerFrame`,
  `QAI.ImpostorFocusRangeCm`, `QAI.ImpostorScreenPromoteCm`, `QAI.Impostor{Shadows,Props,PropsGPU,FreezeUpload,SkipMeshMatch,DebugDraw}`.
- **Lean / significance (agent)** : `QAI.OwnerTickStride`, `QAI.LeanCosmetics`, `QAI.FarPawnOverlap`,
  `QAI.FarPawnQueryOnly`, `QAI.SkinCacheOff`, `QAI.AnimShare`, `QAI.AgentTickBanding`.
- **Trafic aerien** : `QAI.AerialTraffic.SkipDormantBrain` (1), `QAI.AerialTraffic.MinClearanceCm` (40000),
  `QAI.AerialTraffic.ProximityPromoteCm` (15000), `QAI.AerialTraffic.ImpostorMaxParts` (12).
- **Client MP** : `QAI.Client.CreatureImpostorPromoteCm` (600).
- **Pathfinding debug** : `qai.DrawDebugPaths` (`ECVF_Cheat`).

---

## 19. Guide pour le refactoring

### A. Ce qui a ETE comble depuis la v1

| Ecart v1 | Statut |
|----------|--------|
| Factions dupliquees (SensingComponent + SpatialHashGrid) | **Resolu** — module unique `QAI_FactionLib` + matrice. |
| Detection de faction par nom de classe | **Resolu** — composant + reflexion (`GetFactionForActor`). |
| Seulement 3 behaviors | **Resolu** — 13 behaviors enregistres, configs typees par archetype. |
| Pas de pathfinding (line tracing manuel) | **Resolu** — flow field async + Dijkstra + hot-swap. |
| `DataAsset` d'agent stub | **Resolu** — `UQAI_PawnConfig` (template d'authoring complet). |
| Inference d'archetype par `bCanFly` | **Mitige** — `Archetype` explicite + branch sur `PathType==Fly` (le `bCanFly` defaut true etait un piege). |

### B. Ecarts architecturaux restants

- **ECS — composition dynamique** : chaque agent porte TOUS les `FQAI_*Config` (Drone + Spaceship +
  Sangline + ... + Nanite) en cold data. A 13 archetypes, `FQAI_AgentData` est deja volumineux. La cible
  reste un stockage par archetype (config typee unique) ou un vrai ECS query-by-component.
- **HFSM** : toujours une FSM plate. Les transitions communes (flee depuis tout etat de combat) sont
  dupliquees dans le switch. Les etats `Wander/Melee/Investigate` existent dans l'enum mais ne sont pas
  branches — soit les implementer, soit les retirer.
- **Configs combat globales** : `FQAI_CombatConfig` / `FQAI_SpaceshipCombatConfig` vivent sur le
  processeur (une instance pour tous). Un vaisseau cargo et un chasseur partagent les memes timings.

### C. Dette / nettoyage identifie

1. `StateTransitions` (`TArray<FQAI_StateTransition>`) + `FQAI_StateTransition` = **code mort** (la FSM
   vivante est le switch hardcode).
2. `ExecuteIdleState` vide ; `IsAgentAlive` retourne toujours `true` (stubs).
3. `MovementProcessor::ProcessDroneMovement` / `ProcessGroundMovement` semblent **legacy non appeles** par
   le pipeline parallele (le steering vient des behaviors). A verifier/supprimer.
4. **Deux systemes de vision coexistent** (`FQAI_Vision` vs `FQAI_Sensing_Vision`) — deduplication a faire.
   Son/electromagnetique/odorat sont des structs stub.
5. `UQAI_SensingEmitterComponent` = **stub vide avec tick actif** (tick gaspille).
6. `UQAI_ShipMovementSimulation` = stub vide ; `UQAI_BasicMovement` a des champs morts et des fuites de
   `FHitResult`.
7. `ProjectPointToNavigation` dans le flanking = unique survivance NavMesh dans un systeme sans NavMesh
   (incoherence).
8. Le `ProcessEvent` Flyer en phase parallele reste un crasher latent (a deferrer comme les autres).
9. Plusieurs `static`/process-global non world-scoped (registre de tracers du marker, blackbox cyborg).

### D. Ce qui ne doit PAS changer

| Systeme | Raison |
|---------|--------|
| Layout SoA hot/warm/cold | Fondamental pour le cache. Ne pas fusionner les arrays. |
| Pattern parallel-compute / GT-apply | La separation phase pure / phase acteur est ce qui rend le batch thread-safe. |
| Handles sparse + generation | Robuste. Ne pas remplacer par des pointeurs bruts. |
| Behaviors = classes C++ pures (`TUniquePtr`) | Zero GC, destruction deterministe. |
| `CachedWorldScapeRoot` / interdiction `TActorIterator` en parallele | Empeche une corruption de heap Shipping. |
| Split serveur (state+combat) / client (presentation) + pont de nav | Essentiel au netcode meshless. |
| Tier impostor + significance/LOD | Le levier de perf principal du crowd. |

### E. Ajouter un nouveau type d'IA

1. Ajouter la valeur dans `EQAI_AgentArchetype`.
2. Creer `FQAI_<Type>Config` dans `QAI_Struct.h` + le champ dans `FQAI_AgentData`.
3. Creer `FQAI_<Type>Behavior : public FQAI_AgentBehaviorBase`, implementer `ComputeMovementInput` (souvent
   via `QAI_BehaviorHelpers::PursueViaFlowField`) et `ComputeCombatDecision`.
4. L'enregistrer dans `UQAI_AgentBehaviorRegistry::Initialize()`.
5. Cas creature/combat specifique : ajouter le branch dans `AcquireCreatureTarget` (StateProcessor) et
   l'executeur dans `ProcessCreatureCombat`/`ProcessCyborgCombat`/etc. (CombatProcessor).
6. Authoring : un `UQAI_PawnConfig` avec `Archetype` set, pose sur le pawn via `UQAI_AgentComponent.PawnConfig`.
7. Impostor : ajouter le bake dans `UQAI_Settings.ImpostorBakes` (sinon le pawn ne demote jamais).

---

## 20. Structures de donnees de reference

### FQAI_AgentData (config complete d'un agent, `Data/QAI_Struct.h`)

```cpp
struct FQAI_AgentData {
    FQAI_Agent_Data BaseAgent;        // Info + Movement + Sensing
    FQAI_AgentState State;            // etat FSM + tous les champs runtime (squad, route, flow field, stuck...)
    // Police
    FQAI_DroneConfig      DroneConfig;
    FQAI_AutonomousConfig AutonomousConfig;
    FQAI_SpaceshipConfig  SpaceshipConfig;
    // Creatures
    FQAI_SanglineConfig         SanglineConfig;
    FQAI_SuperSanglineConfig    SuperSanglineConfig;
    FQAI_FlyerConfig            FlyerConfig;
    FQAI_SandDiggerConfig       SandDiggerConfig;
    FQAI_InfectedConfig         InfectedConfig;
    FQAI_GazSacConfig           GazSacConfig;
    FQAI_AnimalConfig           AnimalConfig;
    FQAI_CreatureAutonomusConfig CreatureAutonomusConfig;
    // Speciaux
    FQAI_CyborgConfig  CyborgConfig;
    FQAI_NaniteConfig  NaniteConfig;

    EQAI_AgentArchetype Archetype = Unknown;
    float RotationInterpSpeedOverride = 0;   // 0 = global (6.0)
    bool  bCombatEnabled = true;
    int32 PoliceFaction = 3;
    float AttackCooldown = 1.0f;
    float AggressionDuration = 30.0f;
    bool  bCanPushPlayer = false;
};
```

### FQAI_Agent_Data (sous-structure)
`FQAI_Info Agent_Info` (Dangerousness, Priority, `FQAI_Info_Attack`, `FQAI_Info_Faction`),
`FQAI_Movement Agent_Movement` (`EQAI_Movement_Type {Movable, Static}`, `EQAI_Movement_PathType {Ground,
Surface, Fly}`, `FQAI_Movement_Collision`), `FQAI_Sensing Agent_Sensing` (vision active par defaut, le
reste stub).

### FQAI_AgentState (les champs runtime cles)
`CurrentState`, `StateTimer`, `PatrolCenter/Radius`, `PursuitTarget`/`OriginalTarget`, `bInCombat`,
`bWeaponRaisedForCombat`, `LastKnownTargetLocation`/`LastTargetSightTime` ;
**route garde** `PatrolPointA/B`, `bRouteHeadingToB`, `GuardDefendRadius` ;
**escouade** `SquadAnchor`, `SquadAnchorLocation`, `SquadSlot`, `SquadSize` ;
**flanking** `bIsFlanking`, `FlankPosition`, `TimeAtDistance` ; `FleeTarget` (`TOptional`) ;
**gates** `PatrolPauseUntil`, `NextPatrolPauseAt`, `MovementFreezeUntil` ;
**flyer** `bFlyerIsAirborne`, `FlyerPullUpUntil`, `FlyerAttackRunUntil`, `NextFlyerMeleeCheckTime` ;
**flow field** `FlowFieldHandle`, `PendingFlowFieldHandle`, `FlowFieldRequestedForGoal/RealGoal`, `NextFlowFieldCheckAt` ;
**stuck** `LastTrackedPosition`, `LastMovementTime`, `StuckSinceTime`, `StuckEscalateHoldUntilTime` ;
**nav client** `ClientNavDir`, `ClientNavExpiry` ; **degats** `LastDamageTarget`, `LastDamageTime` ;
`bDestroyOnFleeTargetReached`.

### EQAI_CombatAction
`None, Drone_TraceAttack, Drone_Taser, Auto_Melee, Auto_Ranged, Auto_Jump, Spaceship_PrimaryWeapons,
Spaceship_Missiles, Spaceship_Flares, Auto_Detonate`.

### Configs combat (sur le CombatProcessor, globales)
- `FQAI_CombatConfig` (drone : `TraceAttackRange=2500`, `AttackDamage=2.5`, `FireRate=0.125`,
  `FireRateVariance=0.25`, burst ; autonomous : `MeleeDamage=30`, cooldowns ; flank).
- `FQAI_SpaceshipCombatConfig` (`MachineGunFireInterval=0.16`, `RocketFireInterval=2`,
  `FlareCooldown=6`, `FlareDeployChance=0.6`, `CombatFOVDegrees=30`, missiles).

### DroneCoordinationSubsystem
`UQAI_DroneCoordinationSubsystem` (`UWorldSubsystem`) coordonne les tours d'attaque des drones contre un
joueur : `RegisterDrone`, `CanDroneAttack`, file `FCoordinator` (`AttackQueue`, `CurrentAttacker`,
`TurnDuration=4s`, `MinDronesForCoordination=2`, priorite par distance).

### QAI_FunctionLibrary (helpers BP)
`QAI_GetCenterLocationFromLocations`, `QAI_Target_IsHostile_Faction`, `DumpBlueprintAI`,
`Mark/IsAdminSpawnedAI`, et les ponts ALS (`SetPawnRotationMode/Gait/MovementState` via `BPI_Set_*`),
`SetTargetOnCombatComponent`/`GetTargetFromCombatComponent`, plus des helpers de ciblage AIController legacy.

---

## 21. Fichiers et emplacements

```
Plugins/QAI/Source/QAI/
+-- Public/
|   +-- QAI.h                                # module + LogQAI
|   +-- QAI_SubSystem.h                      # WorldSubsystem orchestrateur + SpawnAerialAgent + lifecycle
|   +-- QAI_Settings.h                       # DeveloperSettings (significance, impostor bakes, lean redirects, pathfinding)
|   +-- QAI_Client.h                         # composant client/joueur + Server_ReportAINav
|   +-- QAI_FunctionLibrary.h                # helpers BP + ponts ALS / CombatComponent
|   +-- Interface/QAI_AgentInterface.h       # marqueur "pilote par QAI"
|   +-- Data/
|   |   +-- QAI_Struct.h                     # FQAI_AgentData + tous les FQAI_*Config + EQAI_AnimalCombatRole
|   |   +-- QAI_State_Struct.h               # EQAI_AgentState + FQAI_AgentState (runtime)
|   |   +-- QAI_Info_Struct.h / QAI_Movement_Struct.h / QAI_Sensing_Struct.h / QAI_Network_Struct.h
|   |   +-- QAI_PawnConfig.h                 # UPrimaryDataAsset d'authoring
|   +-- Registry/QAI_AgentRegistry.h         # SoA + handles + significance/impostor accessors
|   +-- Processor/
|   |   +-- QAI_ProcessorBase.h              # batch + pipelined + significance stride
|   |   +-- QAI_StateProcessor.h             # FSM + acquisition + routes/squads + flow field
|   |   +-- QAI_MovementProcessor.h          # steering multi-systeme + ground-snap + dormant integration
|   |   +-- QAI_CombatProcessor.h            # combat par archetype + hitscan + VFX + armes vaisseau
|   +-- Behavior/
|   |   +-- QAI_AgentArchetype.h             # EQAI_AgentArchetype (14 valeurs)
|   |   +-- QAI_AgentBehavior.h              # interface + EQAI_CombatAction + FQAI_Combat{Params,Decision}
|   |   +-- QAI_AgentBehaviorRegistry.h      # archetype -> behavior
|   |   +-- QAI_BehaviorHelpers.h            # PursueViaFlowField, ResolveNavGoal, roles animaux...
|   |   +-- QAI_{Drone,Autonomous,Spaceship}Behavior.h           # police
|   |   +-- QAI_{Sangline,SuperSangline,Flyer,SandDigger}Behavior.h
|   |   +-- QAI_{Infected,GazSac,Animal,CreatureAutonomus}Behavior.h
|   |   +-- QAI_{Cyborg,Nanite}Behavior.h
|   +-- Performance/
|   |   +-- QAI_PerformanceConstants.h       # constantes^2 + LOD_BrainStride / LOD_MoveTickInterval
|   |   +-- QAI_SIMD.h                        # SSE/NEON + fallbacks
|   |   +-- QAI_SpatialHashGrid.h            # grille 2D + subsysteme tickable
|   +-- Faction/QAI_Faction.h                # EQAI_Faction + QAI_FactionLib (matrice)
|   +-- Pathfinding/
|   |   +-- QAI_FlowField.h                  # FQAI_FlowField/Cell/Handle + EQAI_FlowCellFail
|   |   +-- QAI_FlowFieldManager.h           # build async + Dijkstra + sampling
|   +-- Impostor/QAI_ImpostorSubsystem.h     # tier VAT dormant
|   +-- Animation/
|   |   +-- QAI_AnimInstance.h               # anim lean thread-safe
|   |   +-- QAI_AnimShareManager.h           # channels d'anim partages
|   +-- Component/
|   |   +-- QAI_FloatingPawnMovement.h       # mover principal (gravite, plancher WorldScape, replication)
|   |   +-- QAI_BasicMovement.h              # mover physique alternatif (legacy)
|   |   +-- QAI_ShipMovementSimulation.h     # stub
|   |   +-- QAI_PoliceUnitMarkerComponent.h  # replication target/flags + FX multicast
|   +-- Agent/QAI_AgentComponent.h           # hub par pawn (API + impostor + significance + nav bridge)
|   +-- Sensing/
|   |   +-- QAI_SensingComponent.h           # sensing + spatial grid + static defender
|   |   +-- QAI_SensingEmitterComponent.h    # stub
|   |   +-- QAI_SensingFunctionLibrary.h / QAI_Sensing_Vision_Library.h / QAI_Sensing_Vision_Struct.h
|   +-- Spaceship/
|   |   +-- QAI_SubSystem_Spaceship.h        # tick/own spaceship
|   |   +-- QAI_AerialTrafficSubsystem.h     # trafic ambiant serveur
|   |   +-- QAI_AerialTrafficSettings.h      # DeveloperSettings (roster, distances, altitudes)
|   |   +-- QWeaponCoordinatorComponent.h / QSpaceshipObstacleAvoidanceComponent.h
|   |   +-- QSpaceshipMovementInterface.h / QSimpleProjectile.h
|   |   +-- QSpaceshipAIConfigDataAsset.h / QSpaceshipAITypes.h / QShipAILog.h
|   +-- Spawner/
|   |   +-- QAI_AgentSpawner.h               # port BP AI_Spawner
|   |   +-- QAI_GuardPatrolSpawner.h         # gardes cyborg + routes + escouades + capitaine
|   +-- Subsystem/QAI_DroneCoordinationSubsystem.h  # tours d'attaque multi-drones
+-- Private/
    +-- ... (.cpp correspondants)
    +-- QAI_GravityAreaResolver.h            # helper gravite/surface (reflexion Lib_GravityArea, WorldScape)
```

---

**Note de maintenance** : ce document reflete l'etat du code au 26 juin 2026. Les valeurs numeriques
(defauts de config, CVars) sont celles trouvees dans la source a cette date ; verifier la source avant
de s'appuyer sur une valeur precise. Le plugin-level `Plugins/QAI/CLAUDE.md` decrit l'intention du
refactoring behavior/archetype mais est partiellement perime (il documente un `UQAI_PathFindingClient`
octree A* qui a depuis ete remplace par le flow field decrit en section 9). En cas de divergence, la
source fait foi.
