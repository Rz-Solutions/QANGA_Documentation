# QAI - Architecture Technique

## Etat reel de l'architecture

L'intention de design originale etait un **"Data-Driven ECS avec FSM Hierarchique et Batch Processing"**. En pratique, le code actuel est plus precisement un :

**"Data-Oriented Batch Processing Architecture avec FSM Plate et Behaviors Routes par Archetype"**

Voici ou le code livre reellement, et ou il reste du chemin :

| Label original            | Realite dans le code                                          | Statut    |
|--------------------------|---------------------------------------------------------------|-----------|
| **Batch Processing**     | ParallelFor, SIMD SSE/NEON, cache-aligned SoA, prefetch      | Livre     |
| **Data-Driven**          | Layout SoA et configs par archetype, mais FSM et factions hardcodees en C++ | Partiel   |
| **ECS**                  | Entity handles + Systems (processeurs), mais pas de composition dynamique de composants | Inspire   |
| **FSM Hierarchique**     | FSM plate a 7 etats + Strategy pattern par archetype, pas de super-etats/sous-etats | Non       |

**Ce que ca implique pour le refactoring** : les fondations performantes (SoA, SIMD, batch) sont solides. Ce qui manque pour atteindre le design original, c'est la composition dynamique (ECS), la configurabilite (data-driven), et l'imbrication d'etats (HFSM). Ces ecarts sont detailles dans la [section 15 (Guide pour le refactoring)](#15-guide-pour-le-refactoring).

---

## Table des matieres

1. [Pourquoi cette architecture pour QANGA](#1-pourquoi-cette-architecture-pour-qanga)
2. [Vue d'ensemble du pipeline](#2-vue-densemble-du-pipeline)
3. [Le Registre SoA (Structure of Arrays)](#3-le-registre-soa-structure-of-arrays)
4. [Optimisations SIMD et cache](#4-optimisations-simd-et-cache)
5. [Les Processeurs (Batch Processing)](#5-les-processeurs-batch-processing)
6. [La FSM (State Machine)](#6-la-fsm-state-machine)
7. [Le systeme de Comportement par Archetype](#7-le-systeme-de-comportement-par-archetype)
8. [Architecture Client/Serveur](#8-architecture-clientserveur)
9. [Coordination et sous-systemes auxiliaires](#9-coordination-et-sous-systemes-auxiliaires)
10. [Spatial Hash Grid et Sensing](#10-spatial-hash-grid-et-sensing)
11. [Composants de mouvement](#11-composants-de-mouvement)
12. [Systeme de factions](#12-systeme-de-factions)
13. [Configuration et settings](#13-configuration-et-settings)
14. [Structures de donnees de reference](#14-structures-de-donnees-de-reference)
15. [Guide pour le refactoring](#15-guide-pour-le-refactoring)
16. [Fichiers et emplacements](#16-fichiers-et-emplacements)

---

## 1. Pourquoi cette architecture pour QANGA

### Le probleme

QANGA est un jeu multijoueur en monde ouvert avec des planetes entieres. Les AIs ne sont pas cantonnees a un niveau : elles existent sur des surfaces planetaires, dans l'espace, en vol atmospherique, et au sol. Le systeme doit gerer :

- **Des centaines d'agents simultanement** sur un serveur dedie (drones, unites au sol, vaisseaux spatiaux)
- **Trois domaines de locomotion distincts** (vol, sol, espace) avec des physiques radicalement differentes
- **Une architecture client/serveur asymetrique** : le serveur gere l'etat et le combat, les clients gerent le mouvement
- **Des performances serveur critiques** : le serveur est headless, pas de GPU, chaque milliseconde compte

### Pourquoi pas le systeme AI natif d'Unreal ?

Le systeme AI d'Unreal (Behavior Trees, AIController, NavMesh) est concu pour des agents individuels :
- **1 AIController par Pawn** = overhead UObject par agent
- **Behavior Trees individuels** = aucune coherence de cache entre agents
- **NavMesh** = non fonctionnel sur des planetes spheriques ou dans l'espace
- **Pas de batch processing** = chaque agent tick independamment, zero vectorisation

### La solution : architecture Data-Oriented inspiree de l'ECS

L'architecture QAI emprunte des patterns ECS tout en restant dans l'ecosysteme UObject d'Unreal :

| Concept ECS          | Equivalent QAI                                      | Fidelite au pattern |
|----------------------|-----------------------------------------------------|---------------------|
| Entity               | `FQAI_AgentHandle` (index sparse + generation)      | Fidele              |
| Component (data)     | Arrays SoA dans `UQAI_AgentRegistry`                | Partiel (fixe, pas composable) |
| System               | Processeurs (`State`, `Movement`, `Combat`)          | Fidele              |
| Archetype (ECS)      | `EQAI_AgentArchetype` + `FQAI_AgentBehaviorBase`    | Divergent (route le behavior, pas le layout memoire) |

**Ce qui fonctionne** : les donnees de N agents sont dans des tableaux contigus. Un processeur qui itere sur tous les agents touche la memoire sequentiellement, optimisant les prefetch CPU et permettant la vectorisation SIMD.

**Ce qui manque par rapport a un vrai ECS** :
- **Pas de composition dynamique** : chaque agent a TOUJOURS toutes les donnees (HotData + PoliceState + PoliceAgent_Data + OwnerActor + WeaponCoordinator + TaserComponent). On ne peut pas avoir un agent avec Movement mais sans Combat.
- **Pas de queries par composant** : impossible de demander "toutes les entites avec Position + Velocity mais sans Physics". C'est "itere sur tous les agents actifs".
- **Layout memoire identique pour tous** : un drone, une unite au sol, et un vaisseau occupent exactement la meme empreinte memoire dans le registre, meme si 90% de leur config est inutilisee (ex: un drone a quand meme `SpaceshipConfig` et `AutonomousConfig` en memoire).

---

## 2. Vue d'ensemble du pipeline

```
                      UQAI_SubSystem (WorldSubsystem)
                              |
                      RealTimeTick(DeltaTime)
                              |
                  +-----------+-----------+
                  |                       |
              [SERVEUR]              [CLIENT]
                  |                       |
      +---------+--------+         MovementProcessor
      |                  |              seulement
StateProcessor    CombatProcessor
      |                  |
      v                  v
  FSM transitions    Behavior->
  + state exec       ComputeCombatDecision()
      |
      v
  MarkAgentForProcessing(Combat|Movement)
```

### Ordre d'execution des processeurs

L'ordre est **deterministe et critique** :

1. **StateProcessor** - Evalue les transitions FSM, met a jour les timers, marque les flags de processing
2. **MovementProcessor** - Calcule et applique le mouvement via le Behavior, synchronise avec l'actor
3. **CombatProcessor** - Execute les decisions de combat via le Behavior, applique les degats

Cet ordre garantit que :
- L'etat est a jour avant de calculer le mouvement (ex: passer en Flee change le mouvement)
- Le mouvement est a jour avant le combat (ex: la distance au target est correcte)

---

## 3. Le Registre SoA (Structure of Arrays)

### Fichiers
- `Public/Registry/QAI_AgentRegistry.h`
- `Private/Registry/QAI_AgentRegistry.cpp`

### Architecture memoire

Le registre stocke les donnees des agents dans des **tableaux paralleles** (SoA), classes par frequence d'acces :

```
HOT DATA (chaque frame, cache-aligne 64 bytes)
+----------------------------------------------+
| FQAI_AgentHotData[]  (TQAICacheAlignedArray) |
|   - Location      (FVector, 12B)             |
|   - Velocity      (FVector, 12B)             |
|   - Rotation      (FQuat, 16B)               |
|   - Scale         (FVector, 12B)             |
|   - ProcessingFlags (uint8, 1B)              |
|   - Padding       (3B)                       |
|   - LastUpdateTime (float, 4B)               |
|   = 60 bytes -> tient dans 1 cache line 64B  |
+----------------------------------------------+

WARM DATA (a chaque transition d'etat)
+----------------------------------------------+
| FQAI_PoliceState[]   (TArray standard)       |
|   - CurrentState, StateTimer, PatrolCenter   |
|   - PursuitTarget, FlankPosition, etc.       |
+----------------------------------------------+

COLD DATA (rarement accede apres creation)
+----------------------------------------------+
| FQAI_PoliceAgent_Data[] (config complete)    |
| TWeakObjectPtr<AActor>[] (actors proprietaires)|
| TWeakObjectPtr<UActorComponent>[] (armes)    |
| TWeakObjectPtr<UActorComponent>[] (tasers)   |
+----------------------------------------------+
```

### Pourquoi cette separation est critique

Quand le MovementProcessor itere sur 200 agents pour mettre a jour leurs positions, il ne touche que `HotData[]`. Comme chaque element fait 60 bytes (< 64 bytes cache line), un prefetch CPU charge exactement 1 agent par cache line. Le CPU n'a jamais besoin de charger les configs, les targets, ou les references d'actors.

**Comparaison** : si tout etait dans un seul struct (comme un `AAIController`), chaque acces a la position chargerait aussi tous les parametres de config, les pointeurs d'actors, les etats de combat, etc. -> pollution massive du cache.

### Le systeme Sparse/Dense

Les agents sont references par des **handles** (`FQAI_AgentHandle`) et non par des pointeurs :

```cpp
struct FQAI_AgentHandle
{
    int32 Index;      // Index dans le sparse array
    uint32 Generation; // Compteur de generation (invalide les vieux handles)
};
```

Le mapping interne fonctionne ainsi :

```
Handle.Index  -->  SparseToDense[Index]  -->  DenseIndex (position dans les arrays)
DenseIndex    -->  DenseToSparse[Dense]  -->  SparseIndex (retour vers le handle)
Generations[] -->  Valide que le handle est toujours actuel
```

**Suppression d'agent** : swap-and-pop. L'agent supprime est echange avec le dernier element, maintenant un tableau dense sans trous. Cout O(1).

**Avantage du compteur de generation** : si un agent est supprime et qu'un autre code conserve son handle, le handle sera invalide (generation differente) plutot que de pointer vers un agent incorrect.

### TQAICacheAlignedArray

Template d'allocation memoire personnalisee pour le hot data :

```cpp
template<typename T>
class TQAICacheAlignedArray
{
    // Allocation alignee sur QAIPerformance::CACHE_LINE_SIZE (64 bytes)
    void Reserve(int32 NewCapacity)
    {
        // Arrondi a SIMD_BATCH_SIZE (4) pour les operations vectorisees
        int32 AlignedCapacity = RoundUp(NewCapacity, SIMD_BATCH_SIZE);
        void* NewData = FMemory::Malloc(RequiredBytes, CACHE_LINE_SIZE);
        // ...
    }
    // Suppression par swap-and-pop (pas de decalage memoire)
    void RemoveAtSwap(int32 Index);
};
```

Avantages :
- Memoire alignee sur 64 bytes = pas de cache line split
- Capacite arrondie a des multiples de 4 = operations SIMD sans depassement
- Allocation via `FMemory::Malloc` avec alignement (pas `new T[]` qui n'aligne pas)
- Non-copyable (move semantics uniquement) = pas de copie accidentelle d'arrays entiers

---

## 4. Optimisations SIMD et cache

### Fichiers
- `Public/Performance/QAI_SIMD.h`
- `Public/Performance/QAI_PerformanceConstants.h`

### Intrinsiques SIMD

Le systeme fournit des fonctions qui traitent **4 agents simultanement** en une seule instruction CPU :

#### Calcul de distance (SSE sur Windows/Linux)
```cpp
// Traite 4 paires de positions en parallele
void CalculateDistancesSquared4_SSE(
    const FVector* PositionsA,  // 4 positions d'agents
    const FVector* PositionsB,  // 4 positions cibles
    float* OutDistancesSquared  // 4 distances carrees
)
{
    // Charge 4 coordonnees X en un registre 128-bit
    __m128 ax = _mm_set_ps(A[3].X, A[2].X, A[1].X, A[0].X);
    // ... idem pour Y, Z, et les positions B
    // dx^2 + dy^2 + dz^2 en 3 instructions (sub, mul, add)
    __m128 distSq = _mm_add_ps(_mm_add_ps(dx2, dy2), dz2);
    _mm_storeu_ps(OutDistancesSquared, distSq);
}
```

#### Normalisation de vecteurs (avec Newton-Raphson)
```cpp
void NormalizeVectors4_SSE(const FVector* In, FVector* Out)
{
    // rsqrt rapide + 1 iteration Newton-Raphson pour precision
    __m128 rsqrt = _mm_rsqrt_ps(magSq);
    // Raffinement : rsqrt * (1.5 - magSq * rsqrt^2 * 0.5)
    // Resultat : 4 vecteurs normalises en ~10 instructions
}
```

#### Range check par batch
```cpp
void RangeCheckBatch(
    TArrayView<const float> DistancesSquared,
    float RangeSquared,
    TArrayView<bool> OutResults)
{
    __m128 threshold = _mm_set1_ps(RangeSquared);
    __m128 distances = _mm_loadu_ps(&DistancesSquared[i]);
    __m128 cmp = _mm_cmple_ps(distances, threshold);
    int mask = _mm_movemask_ps(cmp);
    // 4 comparaisons de range en 3 instructions
}
```

**Support multi-plateforme** : implementations ARM NEON pour Mac/iOS, fallbacks scalaires pour les autres plateformes. La selection est automatique via `#if QAI_SIMD_AVAILABLE`.

### API haut niveau

```cpp
// Utilisation dans un processeur
QAISIMD::CalculateDistancesSquaredBatch(agentPositions, targetPositions, outDistances);
QAISIMD::NormalizeVectorsBatch(movementInputs, normalizedOutputs);
QAISIMD::RangeCheckBatch(distances, TRACE_ATTACK_RANGE_SQUARED, inRangeResults);
```

Le batch processing itere en groupes de `SIMD_BATCH_SIZE` (4), puis traite le reste en scalaire.

### Constantes pre-calculees

Toutes les distances de range check sont stockees au carre en `constexpr` :

```cpp
namespace QAIPerformance
{
    constexpr float TRACE_ATTACK_RANGE_SQUARED = 2500.0f * 2500.0f;
    constexpr float MELEE_RANGE_SQUARED = 200.0f * 200.0f;
    constexpr float DRONE_ACCEPTANCE_RADIUS_SQUARED = 800.0f * 800.0f;
    // ...
    constexpr bool IsInRangeSquared(float DistSq, float RangeSq) { return DistSq <= RangeSq; }
}
```

Cela evite les `sqrt()` a runtime. Comparer `DistSquared` a `RangeSquared` est mathematiquement equivalent mais ~4x plus rapide.

### Prefetching memoire

Le registre expose des helpers de prefetch pour les operations batch :

```cpp
// Prefetch pour lecture (charge en cache L2)
FORCEINLINE void PrefetchForRead(const void* Address) const
{
    _mm_prefetch(static_cast<const char*>(Address), _MM_HINT_T2);
}

// Prefetch pour ecriture (charge en cache L1)
FORCEINLINE void PrefetchForWrite(const void* Address) const
{
    _mm_prefetch(static_cast<const char*>(Address), _MM_HINT_T0);
}
```

**Utilisation dans BatchExtractPositions** :
```cpp
// Prefetch "rolling" : pendant qu'on traite l'element i,
// on prefetch l'element i+4 pour qu'il soit en cache quand on y arrive
for (int32 i = 0; i < Count; ++i)
{
    if (i + PrefetchBatch < Count)
        PrefetchForRead(&HotData[AgentIndices[i + PrefetchBatch]]);
    OutPositions[i] = HotData[AgentIndices[i]].Location;
}
```

Ce pattern de "rolling prefetch" masque la latence memoire : le CPU charge les donnees futures pendant qu'il traite les donnees presentes.

---

## 5. Les Processeurs (Batch Processing)

### Fichiers
- `Public/Processor/QAI_ProcessorBase.h` / `Private/Processor/QAI_ProcessorBase.cpp`
- `Public/Processor/QAI_StateProcessor.h` / `Private/Processor/QAI_StateProcessor.cpp`
- `Public/Processor/QAI_MovementProcessor.h` / `Private/Processor/QAI_MovementProcessor.cpp`
- `Public/Processor/QAI_CombatProcessor.h` / `Private/Processor/QAI_CombatProcessor.cpp`

### ProcessorBase - Le framework de batch processing

Chaque processeur herite de `UQAI_ProcessorBase` et suit un pipeline strict :

```
ProcessBatch(Registry, DeltaTime)
    |
    +-> ShouldProcess() -- gate de frame time / enabled
    |
    +-> GetAgentsForProcessing(ProcessingFlags) -- filtre par bitmask
    |
    +-> PreProcessBatch() -- setup par-frame
    |
    +-> InternalProcessBatch()
    |       |
    |       +-> Si AgentCount >= MinAgentsForParallel (100) :
    |       |       ParallelFor par chunks de AgentsPerChunk (64)
    |       |
    |       +-> Sinon : ProcessChunk sequentiel
    |               |
    |               +-> Pour chaque agent du chunk :
    |                       ShouldProcessAgent() -> ProcessAgent()
    |
    +-> PostProcessBatch() -- stats / logging
    |
    +-> Clear ProcessingFlags des agents traites
```

### Configuration de processing

```cpp
struct FQAI_ProcessingConfig
{
    bool bUseParallelProcessing = true;    // Activer ParallelFor
    int32 MinAgentsForParallel = 100;      // Seuil pour passer en parallele
    int32 AgentsPerChunk = 64;             // Taille de chunk (= 1 batch cache-friendly)
    float MaxProcessingTimeMs = 5.0f;      // Budget max par processeur
    bool bSkipOnHighFrameTime = false;     // Skip si frame trop longue (desactive pour QAI)
    float FrameTimeThresholdMs = 33.3f;    // Seuil = 30 FPS
};
```

**Note importante** : QAI desactive `bSkipOnHighFrameTime` a l'initialisation. L'IA doit toujours etre mise a jour, meme si le frame rate est bas. Sinon les agents se figent sous charge, ce qui est pire que le ralentissement lui-meme.

### Processing Flags (bitmask)

Chaque agent a un byte de flags dans son HotData :

```cpp
enum class EQAI_AgentProcessingFlags : uint8
{
    None       = 0,
    Movement   = 1 << 0,  // Le MovementProcessor doit traiter cet agent
    StateMachine = 1 << 1, // Le StateProcessor doit traiter cet agent
    Combat     = 1 << 2,  // Le CombatProcessor doit traiter cet agent
    Sensing    = 1 << 3,  // Reserve pour futur SensingProcessor
    All        = Movement | StateMachine | Combat | Sensing
};
```

**Flux** : le StateProcessor marque un agent pour `Combat | Movement` quand il passe en etat Attack. Le MovementProcessor ne traite que les agents marques `Movement`. Apres traitement, le flag est efface.

Cela evite de traiter les agents en Idle qui n'ont besoin ni de mouvement ni de combat.

### Profiling integre

Chaque processeur a :
- `SCOPE_CYCLE_COUNTER(STAT_QAIProcessorBatch)` pour Unreal Insights
- `FQAI_ProcessorProfiler` pour le timing interne
- `FQAI_ProcessingStats` avec moyenne glissante (EMA 0.9/0.1)
- Alerte si le temps de processing depasse `MaxProcessingTimeMs`

---

## 6. La FSM (State Machine)

### Etat actuel vs. intention

L'intention etait une **FSM hierarchique (HFSM)** avec des super-etats contenant des sous-etats. En realite, la FSM est **plate** : 7 etats au meme niveau, sans imbrication.

Une vraie HFSM aurait par exemple :
```
Combat (super-etat, gere les transitions communes flee/return)
  ├── Attack (sous-etat)
  ├── Taser (sous-etat)
  └── Flanking (sous-etat)
```

Ce qui existe a la place, c'est **deux niveaux de decision decouples** :
1. **FSM plate** (StateProcessor) → transitions entre les 7 etats
2. **Behaviors** (par archetype) → actions specifiques au sein d'un etat (le SpaceshipBehavior a ses propres sous-etats de mouvement internes, mais c'est dans le behavior, pas dans la FSM)

C'est un **"FSM + Strategy Pattern"**, pas une HFSM. Le refactoring devrait evaluer si une vraie HFSM est necessaire (voir section 15).

### Etats

```cpp
enum class EQAI_PoliceState : uint8
{
    Idle,    // Inactif, transition immediate vers Patrol ou Pursue
    Patrol,  // Patrouille circulaire (drones) ou waypoints aleatoires (sol)
    Pursue,  // Poursuite active d'une cible
    Attack,  // En combat, tire sur la cible
    Taser,   // Attaque taser (capitaine drone seulement)
    Flee,    // Fuite vers un point eloigne, despawn a l'arrivee
    Return   // Retour vers la zone de patrouille
};
```

### Graphe de transitions

```
                    +--------+
                    |  Idle  |
                    +---+----+
                        |
            cible?------+------pas de cible
            |                       |
            v                       v
        +--------+            +---------+
        | Pursue |<-----------| Patrol  |
        +---+----+            +---------+
            |                      ^
  a portee--+--timeout/perte cible-+
            |                      |
            v                      |
        +--------+                 |
        | Attack |--perte cible--->+
        +---+----+            +--------+
            |                 | Return |
    capitaine-+---fuite?----->+--------+
    +taser    |                    ^
            v                      |
        +-------+   timeout        |
        | Taser |--+---------------+
        +-------+  |
                    v
                +------+
                | Flee |---arrive au point de fuite---> Destroy()
                +------+
```

### Transitions conditionnelles

| Depuis   | Vers    | Condition                                                |
|----------|---------|----------------------------------------------------------|
| Idle     | Patrol  | Pas de cible                                             |
| Idle     | Pursue  | Cible valide (`PursuitTarget != nullptr`)                |
| Patrol   | Pursue  | Cible valide                                             |
| Pursue   | Attack  | Distance <= `AttackEngagementDistance` (60000 unites)     |
| Pursue   | Flee    | `ShouldTransitionToFlee()` (actuellement: toujours false) |
| Pursue   | Return  | Cible perdue OU distance > `MaxPursuitDistance` OU timeout |
| Attack   | Taser   | `Behavior->WantsTaser()` (capitaine drone + a portee)    |
| Attack   | Flee    | `ShouldTransitionToFlee()`                               |
| Attack   | Return  | Cible perdue OU trop loin                                |
| Attack   | Pursue  | Cible hors de portee d'attaque (2x engagement distance)  |
| Taser    | Attack  | Timeout 3s                                               |
| Taser    | Flee    | `ShouldTransitionToFlee()`                               |
| Flee     | Return  | Timer > 10s ET cible hors de `MaxPursuitDistance`        |
| Return   | Patrol  | Arrive pres du `PatrolCenter` OU timeout 15s             |

### Cooldown de transition

Un cooldown de `StateTransitionCooldown` (0.5s) empeche les transitions trop rapides. La `StateTimer` est remise a zero a chaque changement d'etat.

### Hooks d'entree/sortie d'etat

`OnStateEntered()` execute des effets secondaires :
- **Patrol** : mode de vol "Cruise", train d'atterrissage rentre
- **Attack/Taser** : `bInCombat = true`, flags du `PoliceUnitMarkerComponent`, mode de vol "Combat"
- **Flee** : `bInCombat = false`, mode de vol "Cruise"
- **Return** : mode de vol "Landing", train d'atterrissage sorti

Ces effets utilisent la reflexion UE (`FindFunctionByName`) pour appeler des fonctions Blueprint sur les composants de l'actor, sans couplage C++ direct.

---

## 7. Le systeme de Comportement par Archetype

### Fichiers
- `Public/Behavior/QAI_AgentArchetype.h`
- `Public/Behavior/QAI_AgentBehavior.h`
- `Public/Behavior/QAI_AgentBehaviorRegistry.h` / `Private/Behavior/QAI_AgentBehaviorRegistry.cpp`
- `Private/Behavior/QAI_DroneBehavior.cpp`
- `Private/Behavior/QAI_AutonomousBehavior.cpp`
- `Private/Behavior/QAI_SpaceshipBehavior.cpp`

### Archetypes

```cpp
enum class EQAI_AgentArchetype : uint8
{
    Unknown,     // Resolu dynamiquement (legacy)
    Drone,       // Unite volante (police aerienne)
    Autonomous,  // Unite au sol (infanterie)
    Spaceship    // Vaisseau spatial
};
```

### Resolution d'archetype

```cpp
// QAIArchetype::ResolveArchetype()
EQAI_AgentArchetype ResolveArchetype(Registry, AgentIndex)
{
    const auto& Cfg = Configs[AgentIndex];
    if (Cfg.Archetype != Unknown)  // Archetype explicite -> utiliser directement
        return Cfg.Archetype;
    if (Cfg.DroneConfig.bCanFly)   // Inference legacy : bCanFly -> Drone
        return Drone;
    return Autonomous;             // Par defaut : sol
}
```

**Important pour le refactoring** : toujours setter `Archetype` explicitement sur les nouveaux types. L'inference legacy ne gere que Drone/Autonomous.

### Interface de comportement

```cpp
class FQAI_AgentBehaviorBase  // Classe C++ pure, PAS un UObject (zero overhead GC)
{
    // Mouvement : retourne direction + vitesse + point a regarder
    virtual bool ComputeMovementInput(
        Registry, AgentIndex, DeltaTime,
        FVector& OutMovementInput,   // Direction normalisee
        float& OutSpeedScale,        // Multiplicateur de vitesse (0.2 - 2.25)
        FVector& OutTargetToFace     // Point que l'agent doit regarder
    ) = 0;

    // Combat : retourne quelle action effectuer
    virtual FQAI_CombatDecision ComputeCombatDecision(
        Registry, AgentIndex, DeltaTime,
        const FQAI_CombatParams& Params  // Ranges, config capitaine, etc.
    );

    // Etat : est-ce que l'agent veut utiliser le taser ?
    virtual bool WantsTaser(Registry, AgentIndex, Params) const;
};
```

### Le BehaviorRegistry

```cpp
// Initialise au demarrage du subsysteme
void UQAI_AgentBehaviorRegistry::Initialize()
{
    BehaviorByType.Add(Drone, MakeUnique<FQAI_DroneBehavior>());
    BehaviorByType.Add(Autonomous, MakeUnique<FQAI_AutonomousBehavior>());
    BehaviorByType.Add(Spaceship, MakeUnique<FQAI_SpaceshipBehavior>());
}

// Lookup : archetype -> behavior (O(1) via TMap)
FQAI_AgentBehaviorBase* GetBehavior(Registry, AgentIndex)
{
    EQAI_AgentArchetype Type = QAIArchetype::ResolveArchetype(Registry, AgentIndex);
    return BehaviorByType[Type];
}
```

**Design delibere** : les behaviors sont des `TUniquePtr<FQAI_AgentBehaviorBase>` (classes C++ pures) et non des UObjects. Raison : zero overhead du garbage collector, pas besoin de reflexion, et destruction deterministe.

### Delegation depuis les processeurs

Les processeurs ne contiennent **aucune logique specifique a un type d'agent**. Ils delegent tout au behavior :

```cpp
// Dans MovementProcessor::ProcessAgent()
FQAI_AgentBehaviorBase* Behavior = BehaviorRegistry->GetBehavior(Registry, AgentIndex);
if (Behavior)
{
    FVector MovementInput; float SpeedScale; FVector TargetToFace;
    Behavior->ComputeMovementInput(Registry, AgentIndex, DeltaTime,
                                    MovementInput, SpeedScale, TargetToFace);
    // Appliquer le mouvement...
}
// Si pas de behavior -> erreur explicite, pas de fallback
```

### Les 3 behaviors existants

#### DroneBehavior (vol atmospherique)
- Integration GravityScape : calcule le "haut local" a la position du drone (planetes spheriques)
- Mouvement 3D avec altitude cible au-dessus de la target
- Bande de tolerance verticale (120u proche, 250u loin) pour eviter les oscillations
- Speed scaling dynamique : 0.85x proche, 1.75x loin, 2.25x en fuite
- Slowdown progressif a l'approche (`SlowdownAlpha`)
- Combat : tir trace ou taser selon distance et role de capitaine

#### AutonomousBehavior (sol)
- Mouvement 2D avec `GetSafeNormal2D()`
- Acceptance radius adaptative (reduite en combat via `PoliceUnitMarkerComponent.StateFlags`)
- Speed scaling selon etat : 1.2x en poursuite/attaque, 0.8x sinon
- Combat : melee, ranged, ou jump selon distance

#### SpaceshipBehavior (espace)
- Sous-etats de mouvement : Idle, Patrol, Hunt, Combat, Evade, Return
- Obstacle avoidance
- Tracking d'altitude relative au joueur
- Vitesse max : 200,000 km/h (55,555 m/s en unites UE)
- Combat : armes primaires, missiles, flares

---

## 8. Architecture Client/Serveur

### Separation des responsabilites

```
SERVEUR (Dedicated/Listen)          CLIENT
+---------------------------+       +---------------------------+
| StateProcessor (FSM)      |       | MovementProcessor         |
| CombatProcessor           |       |   (mouvement local)       |
| Sync transforms depuis    |       | AutoDiscoverAgents()      |
|   actors -> registry      |       |   (decouvre les pawns AI) |
+---------------------------+       +---------------------------+
         |                                    |
         | Replication UE (PoliceUnitMarker)   |
         +----------------------------------->|
```

### Cote serveur

Le serveur ne fait **PAS de mouvement**. Il :
1. Synchronise les transforms des actors (geres par le client) dans le registry
2. Execute le StateProcessor pour les transitions d'etat
3. Execute le CombatProcessor pour les actions de combat
4. Replique `PoliceUnitMarkerComponent.TargetActor` et `.StateFlags` vers les clients

### Cote client

Le client :
1. Auto-decouvre les Pawns AI avec `UQAI_PoliceUnitMarkerComponent` (toutes les 0.5s)
2. Cree des agents locaux dans son propre registry
3. Execute le MovementProcessor pour le mouvement smooth et local
4. Recupere la cible depuis le marker replique pour adapter le mouvement

**Pourquoi le mouvement est client-side** : les drones/unites au sol doivent avoir un mouvement smooth pour le joueur. Envoyer des positions replicades a 10 Hz depuis le serveur donnerait un mouvement saccade. Le client calcule le mouvement en continu et le serveur fait confiance.

---

## 9. Coordination et sous-systemes auxiliaires

### DroneCoordinationSubsystem

**Fichiers** : `Public/Subsystem/QAI_DroneCoordinationSubsystem.h`

Quand plusieurs drones attaquent le meme joueur, ce sous-systeme coordonne les tours d'attaque :

```cpp
struct FCoordinator {
    TArray<TWeakObjectPtr<APawn>> AttackQueue;  // File d'attente
    TWeakObjectPtr<APawn> CurrentAttacker;       // Drone qui attaque
    float CurrentAttackStartTime;
    float TurnDuration = 4.0f;                   // 4 secondes par tour
};
```

- `RegisterDrone()` : ajoute un drone a la file pour un joueur
- `CanDroneAttack()` : retourne true si c'est le tour de ce drone
- `ProcessCoordination()` : fait tourner les tours, trie par priorite (distance)

**Importance pour le refactoring** : ce systeme est specifique aux drones police mais le concept de coordination d'attaque est generalisable a tout type d'IA.

### PoliceUnitMarkerComponent

Le pont entre les systemes :
- **Replicated** : `TargetActor` et `StateFlags` sont repliques serveur -> clients
- **StateFlags** : bitmask (0x01 = patrol, 0x02 = combat, 0x04 = flee)
- **VFX Multicast** : `MC_PlayShotVFX`, `MC_PlayShotVFXEx` pour les effets visuels de tir
- **JumpImpulse** : `MC_ApplyJumpImpulse` pour les attaques de saut (autonomes)

---

## 10. Spatial Hash Grid et Sensing

### Fichiers
- `Public/Performance/QAI_SpatialHashGrid.h` / `Private/Performance/QAI_SpatialHashGrid.cpp`
- `Public/Sensing/QAI_SensingComponent.h` / `Private/Sensing/QAI_SensingComponent.cpp`
- `Public/Sensing/QAI_SensingEmitterComponent.h` (stub, non implemente)
- `Public/Sensing/QAI_SensingFunctionLibrary.h` / `Private/Sensing/QAI_SensingFunctionLibrary.cpp`

### Spatial Hash Grid

Systeme complet de partitionnement spatial pour les requetes de proximite O(1) au lieu de O(n).

**Architecture** :
```
FQAI_SpatialHashGrid (class C++ pure, thread-safe via FCriticalSection)
    |
    +-- SpatialCells: TMap<int32, FQAI_SpatialCell>   // Hash -> cellule
    +-- ActorToCellHash: TMap<Actor, int32>            // Reverse lookup
    +-- CellSize = 1500 unites (configurable)
    +-- CellSizeInv = 1/CellSize (division rapide)
    +-- Hash function: primes (73856093, 19349663, 83492791) XOR
```

**Grille 2D** : la grille n'utilise que X/Y (pas Z) car les requetes de sensing sont principalement horizontales. Le Z est stocke dans les entrees mais pas utilise pour le hashing.

**Requetes optimisees SIMD** :
```cpp
void QueryActorsInRange(Position, Range, OutActors, OutDistances)
{
    // 1. Calcule les cellules dans le range (O(cells) = O(1) pour des ranges raisonnables)
    GetCellsInRange(Position, Range, CellHashes);
    // 2. Collecte les entrees des cellules
    // 3. Calcule les distances avec QAISIMD::CalculateDistancesSquaredBatch()
    // 4. Filtre par range, trie par distance
}
```

**Requetes batch** : `BatchQueryActorsInRange()` traite 4 requetes en parallele.

**Subsysteme** : `UQAI_SpatialHashGridSubsystem` (UTickableWorldSubsystem) gere automatiquement :
- Mise a jour des positions d'acteurs enregistres a chaque tick
- Maintenance periodique (nettoyage des acteurs invalides, suppression des cellules vides) toutes les secondes
- API Blueprint : `RegisterActor()`, `QueryActorsInRange()`, `QueryHostileActorsInRange()`

### SensingComponent

`UQAI_SensingComponent` est un ActorComponent qui detecte les pawns dans un rayon :
- **Integration SpatialHashGrid** : utilise la grille spatiale au lieu de sphere overlaps (bien plus performant)
- **Auto-sensing** : scan periodique configurable via `SensingFrequency` (defaut 0.5s)
- **Events Blueprint** : `OnPawnDetected`, `OnHostilePawnDetected`
- **Faction-aware** : `GetActorFaction()` et `IsActorHostile()` pour filtrer les hostiles
- **Auto-registration** : s'enregistre dans la grille spatiale au BeginPlay, se desenregistre au EndPlay

**Fallback** : si la grille spatiale n'est pas disponible, retombe sur `SphereOverlapActors` (plus lent).

### SensingFunctionLibrary

Bibliotheque Blueprint de fonctions de sensing avancees (vision, son, etc.) :
- `QAI_Sensing_Vision_GetBasicAngleToTarget()` - angle vers une cible
- `QAI_Sensing_Vision_ComputeVisionScore()` - score de visibilite composite
- `QAI_Sensing_Vision_GetSizeProjection()` - taille apparente d'une cible
- `QAI_Sensing_Vision_LineTraceTarget_Multi()` - line trace avec score de vision
- `QAI_Sensing_Vision_HaveSeeTarget()` - detection complete (taille + vue + ligne de vue)

Ces fonctions consomment les structs `FQAI_Sensing_Vision` definies dans les donnees de sensing. Elles sont actuellement appelees depuis des Blueprints mais pourraient etre integrees dans un futur SensingProcessor.

---

## 11. Composants de mouvement

### QAI_FloatingPawnMovement

**Fichier** : `Public/Component/QAI_FloatingPawnMovement.h`

Extension de `UFloatingPawnMovement` d'Unreal avec :
- **Gravite spherique** : `SimulateGravity` avec `GravityForce` configurable, interpolation de gravite
- **Mouvement au sol** : `MovementOnlyOnGround` pour les unites terrestres
- **Replication integree** : input replique via RPC serveur, location replicade avec seuil d'interpolation
- **Ownership** : systeme de changement de proprietaire (`QAI_ChangeOwner()`, `QAI_GetClientIsOwner()`)
- **Custom tick** : tick functions separees pour la replication d'input et de location (intervals configurables)
- **Sweep au sol** : `GravitySweepByTrace`, `TraceGround` pour detection du sol

C'est le composant de mouvement principal utilise par les drones et unites au sol. Le MovementProcessor lui envoie des `AddInputVector()`.

### QAI_BasicMovement

**Fichier** : `Public/Component/QAI_BasicMovement.h`

Composant de mouvement plus simple, base sur la physique :
- Simulation de gravite
- Sweep collision
- Step height
- Jump
- Auto-rotation vers la direction du mouvement + gravite
- Modes : Ground, Surface, Fly, AntiGravity

Utilise directement `AddInput(Direction, Scale)` et `Jump(Direction)`.

### QAI_ShipMovementSimulation

**Fichier** : `Public/Component/QAI_ShipMovementSimulation.h`

Stub pour la simulation de mouvement de vaisseau. Actuellement vide (juste BeginPlay + TickComponent). Le mouvement des vaisseaux est gere via l'interface Blueprint `QSpaceshipMovementInterface` appelee par reflexion depuis le MovementProcessor.

---

## 12. Systeme de factions

Le systeme de factions est implemente a deux endroits (duplique pour la performance) :

### Factions definies

| ID | Faction      | Hostile envers                        |
|----|-------------|---------------------------------------|
| -1 | Unknown     | Personne                              |
| 0  | Player      | Personne (sauf si attaque)            |
| 1  | IcLabs      | Dissidence, Infected, Pirate, Animal  |
| 2  | Dissidence  | IcLabs, Infected                      |
| 3  | Infected    | Tous sauf autres Infected             |
| 4  | Pirate      | Tous sauf autres Pirates              |
| 5  | Animal      | Tous sauf autres Animaux              |

### Detection de faction

`UQAI_SensingComponent::GetActorFaction()` determine la faction d'un actor par :
1. Presence d'un `QAI_PoliceAgentComponent` -> faction 1 (IcLabs)
2. Presence d'un `CombatComponent` avec nom de classe contenant "Police" -> faction 1
3. Fallback par nom de classe : "Player" -> 0, "Police" -> 1, "Dissidence" -> 2, etc.

**Point d'attention pour le refactoring** : la detection par nom de classe est fragile. Idealement, la faction devrait etre stockee dans un composant ou un tag GameplayTag.

---

## 13. Configuration et settings

### QAI_Settings (DeveloperSettings)

**Fichier** : `Public/QAI_Settings.h`

Configuration globale du plugin, accessible dans Project Settings > QAI :

```cpp
UCLASS(config=Game, defaultconfig)
class UQAI_Settings : public UDeveloperSettings
{
    bool Enabled = true;                    // Active/desactive le subsysteme
    ETickingGroup RealTimeTickGroup = TG_PrePhysics;  // Tick group du subsysteme
    bool bDisableSubsystemTick = true;      // Desactive le tick (pour dev/debug)
    int32 MaxAgent = 4096;                  // Nombre max d'agents
    int32 MaxAgentPerClient = 512;          // Nombre max d'agents par client
    int32 MaxDistanceAgent = 100000;        // Distance max d'un agent (1km en unites UE)
};
```

### QAI_Client

**Fichier** : `Public/QAI_Client.h`

Composant associe a chaque joueur pour l'enregistrement au subsysteme. Gere via `QAI_Register_Client()` / `QAI_UnRegister_Client()`.

### QAI_AgentComponent

**Fichier** : `Public/Agent/QAI_AgentComponent.h`

Composant legacy associe a un actor AI :
- `IsAlive` : boolean de vie
- `OnAgentDied` : delegate multicast pour la mort
- `Agent` : `FQAI_Agent_Data` (info + mouvement + sensing)

Ce composant est distinct du systeme processeur/registre. Il est utilise cote Blueprint pour attacher des donnees AI a un actor.

### QAI_Agent_DataAsset

**Fichier** : `Public/Data/QAI_Agent_DataAsset.h`

Data asset stub pour la configuration d'agents. Actuellement ne contient qu'un `Agent_Name`. Prevu pour stocker la config complete d'un type d'agent (mesh, state tree, collision, etc.).

### QAI_FunctionLibrary

**Fichier** : `Public/QAI_FunctionLibrary.h`

Fonctions utilitaires Blueprint :
- `QAI_GetCenterLocationFromLocations()` : calcule le centre et le rayon englobant d'un ensemble de positions
- `QAI_Target_IsHostile_Faction()` : verifie l'hostilite entre deux structures de faction

---

## 14. Structures de donnees de reference

### FQAI_PoliceAgent_Data (config complete d'un agent)

```cpp
struct FQAI_PoliceAgent_Data
{
    FQAI_Agent_Data BaseAgent;           // Info + Movement + Sensing
    FQAI_PoliceState PoliceState;        // Etat FSM initial
    FQAI_DroneConfig DroneConfig;        // Config drone (bCanFly, bIsCaptain, burst fire...)
    FQAI_AutonomousConfig AutonomousConfig; // Config sol (melee/ranged ranges, speed...)
    FQAI_SpaceshipConfig SpaceshipConfig;   // Config vaisseau (vitesse, altitude, ranges...)
    EQAI_AgentArchetype Archetype = Unknown; // Type explicite
    bool bCombatEnabled = true;
    int32 PoliceFaction = 3;
    float AttackCooldown = 5.5f;
    float AggressionDuration = 30.0f;
};
```

### FQAI_Agent_Data (sous-structure)

```cpp
struct FQAI_Agent_Data
{
    FQAI_Info Agent_Info;        // Priority, Dangerousness, Faction, Attack type/range
    FQAI_Movement Agent_Movement; // Type (Movable/Static), PathType (Ground/Surface/Fly), Collision
    FQAI_Sensing Agent_Sensing;  // Vision, Sound, Electromagnetic, Smell (avec sensibilites)
};
```

### EQAI_CombatAction (actions de combat)

```cpp
enum class EQAI_CombatAction : uint8
{
    None,
    Drone_TraceAttack,         // Tir trace (drone standard)
    Drone_Taser,               // Taser (capitaine drone)
    Auto_Melee,                // Attaque melee (sol)
    Auto_Ranged,               // Attaque a distance (sol)
    Auto_Jump,                 // Attaque de saut (sol)
    Spaceship_PrimaryWeapons,  // Armes principales (vaisseau)
    Spaceship_Missiles,        // Missiles (vaisseau)
    Spaceship_Flares           // Contre-mesures (vaisseau)
};
```

### Systeme de Sensing

Les structs de sensing sont prepares pour une utilisation future :

```cpp
struct FQAI_Sensing
{
    bool Have_Vision;     FQAI_Sensing_Vision Vision;     // Angles, sensibilite nuit, etc.
    bool Have_Sound;      FQAI_Sensing_Sound Sound;       // Sensibilite HF/MF/LF, directivite
    bool Have_Electromagnetic; FQAI_Sensing_Electromagnetic; // Radar, etc.
    bool Have_Smell;      FQAI_Sensing_Smell Smell;       // Odorat
};
```

Ces structures sont consommees par la `QAI_SensingFunctionLibrary` (vision scoring, line traces) et pourraient etre integrees dans un futur SensingProcessor batch.

### CombatProcessor - Details supplementaires

Le CombatProcessor est le plus complexe des trois processeurs. Au-dela de la delegation au behavior, il gere :

**Systeme d'armes** :
- **Drone trace attack** : line trace avec spread configurable, degats via `ApplyDamage`
- **Drone burst fire** : rafales de N tirs avec interval interne, cooldown entre rafales
- **Fire rate jitter** : variance aleatoire (25%) du fire rate pour eviter que les drones tirent en sync
- **Taser** : via `StartTaserSequenceViaComponent()`, appel par reflexion sur le composant taser

**Systeme d'armes vaisseau** :
- **WeaponCoordinator** : composant Blueprint trouve par reflexion, gere les armes primaires et secondaires
- `FirePrimaryWeaponsViaCoordinator()`, `SetWeaponTargetViaCoordinator()`
- `FireWeaponsByClassViaCoordinator()` pour les types d'armes specifiques
- **Rockets** : timing de groupes, intervalle min/max entre salves
- **Flares** : scan de menaces missiles, deploiement probabiliste (`FlareDeployChance = 0.6`)
- **Missile targeting** : setup, cleanup periodique, refresh d'intervalle

**VFX** :
- Traceurs Niagara (bullet beam + muzzle flash) avec lifecycle management
- Moving tracers avec vitesse configurable (`TracerSpeed = 30000`)
- Nettoyage automatique des effets expires

**Coordination** : integration avec `DroneCoordinationSubsystem` pour les tours d'attaque

**Config combat** (`FQAI_CombatConfig` et `FQAI_SpaceshipCombatConfig`) :
```cpp
// Drone
TraceAttackRange = 2500, AttackDamage = 2.5, FireRate = 0.125s
AttackSpread = 2.0, BurstFire, FireRateVariance = 25%

// Autonomous
MeleeDamage = 30, MeleeAttackCooldown = 1.5s
JumpAttackRange = 2000, JumpAttackCooldown = 8s

// Spaceship
MachineGunFireInterval = 0.16s, RocketFireInterval = 2s
FlareCooldown = 6s, CombatFOVDegrees = 30
```

### MovementProcessor - Details supplementaires

Le MovementProcessor gere la complexite de 4 systemes de mouvement differents :

**Pipeline d'application du mouvement** (`ApplyMovementInput`) :
1. Calcule la velocite desiree (`MovementInput * BaseSpeed * SpeedScale`)
2. Interpole vers la velocite desiree (`VInterpTo`, vitesse 8.0)
3. Cherche un composant `ObstacleAvoidance` par reflexion -> `GetSteeringDirection()`
4. Applique l'input au `Pawn->AddMovementInput()` (pour CharacterMovement/AIController)
5. Applique aussi a `UQAI_FloatingPawnMovement->AddInputVector()` (si present)
6. Pour les vaisseaux : cherche `QSpaceshipMovementInterface` -> `ApplyDirectMovementInput()` ou `MoveToLocation()`
7. Pour les vehicules sans interface spaceship : `VehicleMovementComponent->AddMoveInput()`
8. Synchronise la transform de l'actor vers le registre

**Rotation** (`UpdateAgentRotation`) :
- Skip si archetype Spaceship ou si VehicleMovement present (rotation geree par la physique)
- Drones : rotation horizontale seulement (pas de pitch/roll)
- Sol : rotation complete vers la cible
- Interpolation via `RInterpTo` a `RotationInterpSpeed` (6.0)

**Flanking** (unites au sol en Attack) :
- Apres 5s a portee de melee sans bouger, l'unite tente un flanking
- Position calculee a 300u perpendiculairement a la direction target
- Projete sur le NavMesh pour navigabilite
- Alternance gauche/droite aleatoire

**Fuite gravity-aware** (`CalculateGravityAwareFlee`) :
- Calcule la direction "loin du joueur" dans le plan perpendiculaire a la gravite locale
- Support WorldScape (planetes spheriques) et GravityScape
- Direction de fuite = horizontale (loin du joueur) + verticale (contre gravite)

---

## 15. Guide pour le refactoring

### A. Ecarts architecturaux a combler

Ces ecarts representent la distance entre l'intention de design ("Data-Driven ECS + HFSM + Batch") et le code actuel. Les combler est necessaire pour que le systeme supporte tous les types d'IA du jeu.

#### A1. ECS : ajouter la composition dynamique de composants

**Probleme actuel** : chaque agent a un layout memoire fixe et identique. Un drone transporte en memoire `AutonomousConfig` + `SpaceshipConfig` (inutilises), un vaisseau transporte `DroneConfig` + `AutonomousConfig` (inutilises). On ne peut pas ajouter/retirer des "composants" de donnees par agent.

**Impact** : impossible d'ajouter un nouveau type d'IA (creature, civil, vehicule) sans gonfler `FQAI_PoliceAgent_Data` pour tout le monde. A 20 types d'IA, la struct cold data deviendra enorme.

**Solution proposee** : decouple le stockage par archetype.

Option legere (garde le SoA mais avec configs typees) :
```cpp
struct FQAI_AgentData
{
    FQAI_Agent_Data BaseAgent;
    EQAI_AgentArchetype Archetype;
    // Un seul actif selon l'archetype, les autres sont null
    TSharedPtr<void> ArchetypeConfig; // Cast selon Archetype
};
```

Option complete (vrai ECS) : migrer vers un systeme ou chaque agent est un ensemble de composants (Position, Velocity, FSMState, CombatConfig...) et les processeurs queryent par ensemble de composants. C'est un refactoring majeur mais c'est le seul moyen de scaler a des dizaines de types d'IA sans compromis.

#### A2. Data-Driven : rendre la FSM et les factions configurables

**Probleme actuel** : les transitions FSM sont hardcodees dans `StateProcessor::UpdateStateTransitions()`. Les regles de faction sont hardcodees par IDs numeriques dans `AreFactionsHostile()` (dupliquees a 2 endroits). La detection de faction se fait par string match sur les noms de classe.

**Impact** : ajouter un nouvel etat FSM ou une nouvelle faction necessite de modifier du C++, recompiler, et risquer des regressions.

**Solution proposee** :
- **FSM** : table de transitions en DataAsset ou DataTable. Chaque archetype reference sa propre table. Le StateProcessor lit la table au lieu d'avoir un `switch/case`.
- **Factions** : DataTable avec matrice d'hostilite. Un composant `UQAI_FactionComponent` sur chaque actor stocke sa faction (plus de string match).
- **Configs combat** : deplacer `FQAI_CombatConfig` et `FQAI_SpaceshipCombatConfig` depuis le processeur (global) vers la config par archetype (par-agent).

#### A3. HFSM : ajouter l'imbrication d'etats

**Probleme actuel** : FSM plate a 7 etats. Pas de super-etats, pas de sous-etats. Les transitions communes (ex: flee depuis n'importe quel etat de combat) sont dupliquees dans chaque case du switch.

**Impact** : chaque nouvel etat multiplie les transitions a gerer. A 15+ etats (creatures, civils, etc.), le switch devient inmaintenable.

**Solution proposee** : implementer une vraie HFSM ou :
- Des super-etats (ex: `Combat`) gerent les transitions communes (flee, return)
- Des sous-etats (ex: `Attack`, `Taser`, `Flanking`) gerent le comportement specifique
- Les hooks `OnStateEntered`/`OnStateExited` se propagent dans la hierarchie

Alternative : State Tree d'Unreal si la performance batch le permet, ou un systeme custom data-driven.

### B. Generalisations necessaires

#### B1. Renommer les structures "Police" -> generiques

Les noms actuels sont colles au domaine police :
- `FQAI_PoliceAgent_Data` -> `FQAI_AgentData`
- `FQAI_PoliceState` -> `FQAI_AgentState`
- `EQAI_PoliceState` -> `EQAI_AgentState`
- `UQAI_PoliceUnitMarkerComponent` -> `UQAI_AgentMarkerComponent`

#### B2. Creer de nouveaux behaviors

Pour chaque nouveau type d'IA, suivre le pattern existant :

1. Ajouter la valeur dans `EQAI_AgentArchetype`
2. Creer `FQAI_NewTypeBehavior : public FQAI_AgentBehaviorBase`
3. Implementer `ComputeMovementInput()` et `ComputeCombatDecision()`
4. Enregistrer dans `UQAI_AgentBehaviorRegistry::Initialize()`

Les processeurs delegueront automatiquement. Ce pattern fonctionne bien tel quel.

#### B3. Completer le DataAsset d'agent

`UQAI_Agent_DataAsset` est un stub vide. Il devrait stocker la config complete d'un type d'agent : archetype, configs specifiques, references mesh/animation, table de transitions FSM, etc. C'est le vecteur principal pour rendre le systeme data-driven.

### C. Ce qui ne doit PAS changer

| Systeme | Raison |
|---------|--------|
| Layout SoA hot/warm/cold | Fondamental pour les performances cache. Ne pas fusionner les arrays. |
| Batch processing (ProcessorBase) | Le pattern ProcessBatch -> ProcessChunk -> ParallelFor est generique et fonctionne pour tout type d'agent. |
| Handles avec generation | Le systeme sparse/dense avec compteur de generation est robuste. Ne pas remplacer par des pointeurs bruts. |
| Architecture client/serveur | Le split state+combat serveur / mouvement client est essentiel pour le netcode de QANGA. |
| Behaviors comme classes C++ pures | Zero overhead GC, destruction deterministe. Ne pas convertir en UObjects. |
| Spatial Hash Grid | Performant et bien integre. Etendre plutot que remplacer. |
| SIMD batch operations | Les intrinsiques SSE/NEON avec fallbacks scalaires sont portables et performantes. |

### D. Bugs et hacks a nettoyer

1. **`IsDroneAgent()` legacy** dans `MovementProcessor:1135` et `CombatProcessor:1498` : verifie `bCanFly` au lieu d'utiliser `ResolveArchetype()`. Contourne le systeme de behavior.

2. **String check** dans `CombatProcessor:1112` : `ActorName.Contains("Drone")` pour sauter les animations d'attaque. Remplacer par une verification d'archetype.

3. **`ShouldTransitionToFlee()`** retourne toujours `false`. A implementer proprement, idealement delegue au behavior par archetype.

4. **Factions dupliquees** : `AreFactionsHostile()` est implementee identiquement dans `SensingComponent` ET `SpatialHashGrid`. Extraire dans une fonction utilitaire unique.

5. **Detection de faction par nom de classe** : `GetActorFaction()` utilise des string matches ("Police", "Infected", etc.). Fragile et non extensible.

6. **`SensingProcessor`** reference dans les flags (`EQAI_AgentProcessingFlags::Sensing`) mais n'existe pas. Les structures de sensing et la `SensingFunctionLibrary` sont en place, il manque le processeur batch.

7. **`UQAI_SensingEmitterComponent`** est un stub vide non implemente.

8. **Configs combat globales** : `FQAI_CombatConfig` et `FQAI_SpaceshipCombatConfig` sont sur le processeur (une instance pour tous les agents). Un vaisseau cargo et un chasseur utilisent les memes timings de combat.

9. **La gravite locale** : seul `DroneBehavior` utilise GravityScape. `MovementProcessor::CalculateDroneMovementVector()` utilise la gravite monde (pas locale). Tout behavior volant sur planete spherique doit utiliser GravityScape.

---

## 16. Fichiers et emplacements

```
Plugins/QAI/Source/QAI/
+-- Public/
|   +-- QAI.h                                    # Module declaration + LogQAI
|   +-- QAI_SubSystem.h                          # WorldSubsystem principal
|   +-- QAI_Settings.h                           # DeveloperSettings (config=Game)
|   +-- QAI_Client.h                             # Composant client/joueur
|   +-- QAI_FunctionLibrary.h                    # Fonctions utilitaires Blueprint
|   +-- Data/
|   |   +-- QAI_Struct.h                         # Structs de config agent + archetype configs
|   |   +-- QAI_Info_Struct.h                    # Info agent (priorite, faction, attaque)
|   |   +-- QAI_Movement_Struct.h                # Types de mouvement et collision
|   |   +-- QAI_Sensing_Struct.h                 # Sensing (vision, son, EM, odeur)
|   |   +-- QAI_State_Struct.h                   # Etats FSM + FQAI_PoliceState
|   |   +-- QAI_Network_Struct.h                 # Structs reseau (vide)
|   |   +-- QAI_Agent_DataAsset.h                # Data asset agent (stub)
|   +-- Registry/
|   |   +-- QAI_AgentRegistry.h                  # Registre SoA + handle system
|   +-- Processor/
|   |   +-- QAI_ProcessorBase.h                  # Framework batch processing
|   |   +-- QAI_StateProcessor.h                 # FSM
|   |   +-- QAI_MovementProcessor.h              # Mouvement
|   |   +-- QAI_CombatProcessor.h                # Combat (+ VFX traceurs + systeme armes vaisseau)
|   +-- Behavior/
|   |   +-- QAI_AgentArchetype.h                 # Enum des archetypes
|   |   +-- QAI_AgentBehavior.h                  # Interface + ResolveArchetype()
|   |   +-- QAI_AgentBehaviorRegistry.h          # Registry archetype -> behavior
|   |   +-- QAI_DroneBehavior.h                  # Behavior drone
|   |   +-- QAI_AutonomousBehavior.h             # Behavior sol
|   |   +-- QAI_SpaceshipBehavior.h              # Behavior vaisseau (sous-etats mouvement)
|   +-- Performance/
|   |   +-- QAI_PerformanceConstants.h           # Constantes pre-calculees (distances^2, timings)
|   |   +-- QAI_SIMD.h                          # Intrinsiques SSE/NEON + fallbacks scalaires
|   |   +-- QAI_SpatialHashGrid.h               # Grille spatiale + subsysteme tickable
|   +-- Agent/
|   |   +-- QAI_AgentComponent.h                 # Composant legacy agent (IsAlive, OnDied)
|   +-- Component/
|   |   +-- QAI_PoliceUnitMarkerComponent.h      # Composant replique serveur->client
|   |   +-- QAI_FloatingPawnMovement.h           # Mouvement flottant avec gravite + replication
|   |   +-- QAI_BasicMovement.h                  # Mouvement physique simple (gravite, sweep, jump)
|   |   +-- QAI_ShipMovementSimulation.h         # Stub mouvement vaisseau
|   +-- Sensing/
|   |   +-- QAI_SensingComponent.h               # Detection de pawns + integration SpatialGrid
|   |   +-- QAI_SensingEmitterComponent.h        # Stub emetteur de signaux
|   |   +-- QAI_SensingFunctionLibrary.h         # Vision scoring, line traces, size projection
|   +-- Subsystem/
|       +-- QAI_DroneCoordinationSubsystem.h     # Coordination d'attaque multi-drones
+-- Private/
    +-- QAI.cpp                                  # Module startup/shutdown
    +-- QAI_SubSystem.cpp                        # Tick pipeline, init processeurs, auto-discover
    +-- QAI_Settings.cpp                         # Settings singleton
    +-- QAI_Client.cpp                           # Client registration
    +-- QAI_FunctionLibrary.cpp                  # Utilitaires BP
    +-- Registry/QAI_AgentRegistry.cpp           # SoA ops, batch, prefetch, handle management
    +-- Processor/QAI_ProcessorBase.cpp          # ParallelFor, chunk processing, profiling
    +-- Processor/QAI_StateProcessor.cpp         # FSM transitions + state execution
    +-- Processor/QAI_MovementProcessor.cpp      # 4 systemes de mouvement, flanking, fuite
    +-- Processor/QAI_CombatProcessor.cpp        # Armes, burst fire, VFX, vaisseau combat avance
    +-- Behavior/QAI_AgentBehaviorRegistry.cpp   # Init + lookup behaviors
    +-- Behavior/QAI_DroneBehavior.cpp           # Vol 3D + GravityScape + speed scaling
    +-- Behavior/QAI_AutonomousBehavior.cpp      # Sol 2D + acceptance adaptative
    +-- Behavior/QAI_SpaceshipBehavior.cpp       # Espace + sous-etats + obstacle avoidance
    +-- Performance/QAI_SpatialHashGrid.cpp      # Grid SIMD + subsysteme + maintenance
    +-- Agent/QAI_AgentComponent.cpp             # Legacy agent component
    +-- Component/QAI_PoliceUnitMarkerComponent.cpp  # Replication + VFX multicast
    +-- Component/QAI_FloatingPawnMovement.cpp   # Mouvement + gravite + replication RPC
    +-- Component/QAI_BasicMovement.cpp          # Mouvement physique
    +-- Component/QAI_ShipMovementSimulation.cpp # Stub
    +-- Sensing/QAI_SensingComponent.cpp         # Scan + spatial grid + faction detection
    +-- Sensing/QAI_SensingEmitterComponent.cpp  # Stub
    +-- Sensing/QAI_SensingFunctionLibrary.cpp   # Vision math (angles, scores, line traces)
    +-- Subsystem/QAI_DroneCoordinationSubsystem.cpp  # Tour d'attaque, priorite, cleanup
    +-- Data/QAI_Agent_DataAsset.cpp             # Stub
```
