# DynamicQuestSystem -- Documentation Architecture Exhaustive

> **Derniere verification :** 2026-06-26 -- document reaudite ligne par ligne contre le code source actuel du plugin (`Plugins/DynamicQuestSystem/Source`). Les chiffres de lignes, listes de RPC, enumerations, structs et limitations refletent l'etat reel du code a cette date, pas la redaction initiale (2026-03). Le systeme a fortement grossi depuis : `QuestManagerSubsystem` est passe de ~3000 a **13 558** lignes d'implementation, et plusieurs enumerations / structs documentees auparavant ne correspondaient plus du tout au code.

## 1. Vue d'ensemble

Le **DynamicQuestSystem** est un framework de quetes modulaire pour UE5 (5.7.3), concu pour un jeu multijoueur (client / serveur dedie) avec un monde procedural (WorldScape). Il gere le cycle de vie complet des quetes : definition, assignation, progression, completion, persistance binaire et interface utilisateur.

Le plugin est structure en **5 modules** (Core, Objectives, Editor, Debug, Interaction) avec une separation nette entre runtime, edition, debug, interaction et objectifs concrets. L'architecture est **server-authoritative** : toute mutation d'etat passe par le serveur (`QuestManagerSubsystem`), puis est repliquee aux clients via des proprietes repliquees et des RPC sur `PlayerQuestComponent` (et, en voie parallele, via des `AQuestInstanceActor` repliques par quete).

Le systeme supporte a la fois des quetes designees manuellement (editeur JSON + graphe visuel en lecture seule) et des quetes **procedurales** generees a partir de Points d'Interet (POI) decouverts dans le monde, avec des templates d'archetypes multi-etapes.

L'identite joueur est **deterministe** : `GetOrCreatePersistentPlayerID()` resout l'`UniqueNetId` OSS/Steam et **defere** (retourne `NAME_None`) tant que le `PlayerState`/OSS n'est pas pret -- les acceptations de quete emises avant la resolution d'identite sont mises en file et rejouees (pas de fallback d'ID factice).

---

## 2. Architecture reelle

### 2.1 Modules et dependances

5 modules. Les fleches reproduisent fidelement les `*.Build.cs` (les libelles `(public)`/`(prive)` sont la portee UBT reelle). Attention: dans chaque `Build.cs`, le `PublicDependencyModuleNames` contient `"Core"` = **module moteur**, a ne pas confondre avec le module `DynamicQuestSystem` (le "Core" quest).

```
DynamicQuestSystemInteraction        <-- Autonome (aucune dep. quest)
   ^         ^              ^
   | public  | prive        | prive
   |         |              |
DynamicQuestSystem    DynamicQuestSystemObjectives
   (Core quest)              ^      ^
   ^      ^                  |      |
   | priv | public (Core)    | public (Objectives)
   |      +------------------+|     |
   |                          +-----+
DynamicQuestSystemDebug   DynamicQuestSystemEditor
```

Relations exactes (verifiees dans les `Build.cs`) :

| Module | Depend de (modules quest) | Portee | Notes |
|---|---|---|---|
| Interaction | (aucun) | -- | Autonome. PublicDep moteur: EnhancedInput, UMG, GameplayTags ; IWYUSupport.Full |
| Core (`DynamicQuestSystem`) | Interaction | public | + QNotification, QTriggerZone, QAmbientAudio, QLevel, InputSystem, CommonUI/CommonInput (public) ; OnlineSubsystem/CoreOnline, EssentialMacros, Niagara, LevelSequence/MovieScene, 8x WorldScape (prive) |
| Objectives | Core ; **Interaction** ; QNotification | Core=public, Interaction+QNotification=prive | + Niagara, QCableConnector, QLevel, QAmbientAudio, LevelSequence/MovieScene/MovieSceneTracks (public) |
| Debug | Core | **prive** | Tres leger : PublicDep = moteur "Core" uniquement ; le quest-Core est en PrivateDep |
| Editor | **Core** ; **Objectives** ; QTriggerZone/QAmbientAudio | Core+Objectives=public | + QTriggerZoneEditor (prive) ; GraphEditor, BlueprintGraph, KismetWidgets, PropertyEditor, etc. |

Pieges :
- Le graphe d'origine ne montrait qu'`Editor -> Objectives` ; en realite **Editor depend aussi directement de Core en public** et de **QTriggerZoneEditor** en prive.
- `Objectives` n'est pas qu'un consommateur de Core : il depend aussi de **Interaction** (prive) et **QNotification** (prive).


### 2.2 Tableau des sous-systemes

| Sous-systeme | Module | Statut | Description (etat 2026-06) |
|---|---|---|---|
| `QuestManagerSubsystem` | Core | **Complet** | Orchestrateur central (`UGameInstanceSubsystem`, implemente `IQuestProvider`). 818 l. header / **13558 l.** impl. Porte aussi la machine a etats de transition d'univers persistant, le tracking d'inventaire, les quetes procedurales et les world-kill events. |
| `PlayerQuestSubsystem` | Core | **Complet** | Facade par joueur local (`ULocalPlayerSubsystem`, implemente `IQuestProgress`). Resout l'ID joueur deterministe et gere les acceptations en attente (`PendingQuestAcceptances`). |
| `PlayerQuestComponent` | Core | **Complet** | Pont reseau replique sur le `PlayerController`. 6 proprietes repliquees, ~17 RPC, timer de localisation a frequence dynamique. |
| `QuestRelevancyManager` | Core | **Partiel (broadcast)** | Ciblage d'actions reseau. `IsPlayerQuestParticipant` / `IsPlayerOnSameTeam` degenerent en `true` ; seule la regle `ProximityBased` filtre reellement. |
| `QuestComponentManager` | Core | **Complet** | Factory / conteneur d'objets modulaires par quete (`AddToRoot` anti-GC). |
| `BinaryQuestDataSource` | Core | **Complet** | Persistance binaire versionnee (**v4**) avec backup `.bak` ; fichier monolithique OU individuel par quete (`bUseIndividualQuestFiles`). |
| `QuestInstanceActor` | Core | **Actif (voie parallele)** | Acteur replique par-quete (etat / progression / timestamps), distinct de `PlayerQuestComponent`. |
| `QuestActionBridge` | Core | **Actif** | Acteur replique ephemere (auto-destruction 30 s) routant les actions vers les joueurs cibles via `IsNetRelevantFor`. |
| `QuestDynamicRegistry` | Core | **Complet** | Magasin cle->valeur type (11 types + 2 collections) avec chargement async ; expose via `UQuestDynamicRegistrySubsystem`. |
| `QuestEventManager` (`UObject`) | Core | **Complet** | Bus d'evenements UObject avec handlers `IQuestEventHandler` tries par priorite. |
| `QuestEvents` (`FQuestEventManager`) | Core | **Complet (redondant)** | Singleton statique C++ a 5 delegates multicast ; coexiste avec `UQuestEventManager`. |
| `QuestObjectiveFactory` | Core | **Complet** | Factory d'objectifs/actions/conditions par nom de type. |
| `QuestAssetManager` | Core | **Doublon** | Decouverte de classes -- chevauche `QuestObjectiveFactory`. |
| `QuestUIManager` | Core | **Complet** | Centralise le container de trackers + le dispatch des notifications toast via `QuestNotificationUtils`. |
| `QuestNotificationUtils` | Core | **Complet** | Sequencage de notifications (bridge QNotification) + suivi du blocage de completion par `SourceContext`. |
| `ProceduralQuestTemplate` | Core | **Complet** | 8 templates de quetes procedurales. |
| `MultiStageQuestTemplate` | Core | **Complet** | 5 archetypes multi-etapes (`EQuestArchetypeID` en declare 11). |
| `QuestStringTableRuntime` | Core | **Complet** | Resolution de textes localises via StringTables locales par QuestID. |
| `InteractableMeshDatabase` | Core | **Complet** | Classification de meshes par ~169 mots-cles codes en dur (placement procedural Interact/Scan). |
| `LevelMeshScanner` | Core | **Complet** | Scan de niveaux. `PerformAsyncScan` differe la passe UObject sur le game thread (fix dd8b0e3b). |
| `QuestEditorWindow` | Editor | **Complet** | Editeur principal, **8 onglets** (`PlayerProgress` ajoute). |
| `DynamicQuestSystemEditorSubsystem` | Editor | **Complet** | Couche donnees editeur : cache, verrous pessimistes (300 s), JSON par quete, migration legacy, export binaire. |
| `QuestFlowGraph` | Editor | **Lecture seule** | Graphe visuel. `SaveToQuestDefinition()` est vide. |
| `EditorJsonQuestDataSource` | Editor | **Complet** | Persistance JSON editeur avec migration binaire legacy. |
| `QuestDebugSubsystem` | Debug | **Complet** | 6 commandes console (`dqs.*`) + overlay Slate temps reel ; contexte resolu aussi en serveur dedie. |
| `InteractionSubsystem` | Interaction | **Complet** | Registre global d'acteurs interactifs (index par tag + recherche spatiale). |
| `InteractionComponent` | Interaction | **Complet (dette)** | Detection par line trace timer (0.1 s) ; `GetInteractionDistance()` de l'interface jamais interrogee. |
| `QuestModuleClassRegistrar` | Objectives | **Complet** | Enregistrement anti-GC de toutes les classes du module. |

### 2.3 Dependances externes

Reconciliation `.uplugin Plugins[]` vs modules `Build.cs`. Le `.uplugin` active 14 plugins ; certains n'ont pas de dep. module C++ (lien de cooking/activation uniquement).

| Plugin (.uplugin) | Module(s) C++ utilise(s) | Usage |
|---|---|---|
| QNotification | Core (public), Objectives (prive) | Notifications UI (toast, progress, warning, communication) |
| QLevel | Core (public), Objectives (public) | Streaming de niveaux, assets POI quetes procedurales |
| QTriggerZone | Core (public), Editor (public) | Zones de declenchement spatiales (+ QTriggerZoneEditor cote Editor) |
| QAmbientAudio | Core (public), Objectives (public), Editor (public) | Controle audio ambiant via actions de quete |
| QCableConnector | Objectives (public) | Composant cable pour ConnectorObjective |
| WorldScape | Core (prive) -- **8 sous-modules** : WorldScapeCommon, WorldScapeNoise, WorldScapeVolume, WorldScapeFoliages, WorldScapeCompute, WorldScapeGPUTerrain, WorldScapeCore | Monde procedural (gravite, terrain, projection) |
| EssentialMacros | Core (prive) | Macros utilitaires |
| InputSystem | Core (public) | Input custom (InputPreset_DA pour notifications) |
| EnhancedInput | Interaction (public), Objectives (public) | Input UE5 pour interactions (**pas** une dep. de Core) |
| CommonUI | Core (public, + CommonInput) | Framework UI |
| Niagara | Core (prive), Objectives (public) | Effets visuels (PlayNiagaraAction) |
| OnlineSubsystem | Core (prive, + CoreOnline) | Support multijoueur |
| LevelSequence / MovieScene | Core (prive), Objectives (public, + MovieSceneTracks) | Cinematiques (PlayLevelSequenceAction) |
| EditorScriptingUtilities | (active dans .uplugin ; **aucun Build.cs ne le reference**) | Lien plugin editeur, sans dep. module C++ |

Subtilites :
- `OnlineSubsystem`/`CoreOnline` sont des deps **privees de Core** -- ne figuraient pas dans l'ancienne table.
- `EditorScriptingUtilities` est active au niveau `.uplugin` mais n'est consomme par aucun module (dependance d'activation).
- WorldScape n'est pas "un" module : Core tire 7 sous-modules WorldScape distincts en prive.
- Core compile avec `OptimizeCode = InShippingBuildsOnly`, `bEnableExceptions=false` en Shipping, et une dep. prive `Settings` conditionnelle (Editor ou non-Shipping).


## 3. Pipeline d'execution

### 3.1 Cycle de vie d'une quete

```
                          EDITEUR                          RUNTIME
                    +------------------+
                    | JSON sur disque  |
                    | (Content plugin: |
                    |  EditorQuests/)  |
                    +--------+---------+
                             |
                    Export binaire (RuntimeQuests.bin, v4)
                             |
                             v
              +-----------------------------+
              | QuestManagerSubsystem       |
              | Initialize()                |
              |   LoadQuestsFromBinaryAsync |------> FAsyncLoadQuestsFromBinaryTask
              |   -> QuestDataMap populated |            (FAutoDeleteAsyncTask)
              +-----------------------------+            |
                             |                    callback game thread
                             v                           |
              QuestDataMap : TMap<FName, FQuestDefinition>
                             |
          +------------------+------------------+
          |                                     |
    AssignQuestToPlayer()             GenerateProceduralQuest...()
          |                                     |  (POI + faction + difficulte)
          v                                     v
    FRuntimeQuestInstance cree         Template/Archetype -> FQuestDefinition
    Objectives/Actions instancies      -> AssignQuestToPlayer()
    (Reconstruct... si charge depuis disque/reconnexion)
          |
          v
    PlayerQuestComponent (serveur)
    -> Replication ActiveQuests[] (+ AQuestInstanceActor par quete)
    -> Client : OnRep_ActiveQuests() (saute si signature CRC inchangee)
          |
          v
    Objectifs actives (autorite uniquement)
    ProcessQuestEvent() / IncrementProgress() / RegisterObjectiveProgress()
          |
          +-- Objectif complete --> FinishObjective()
          |       (differe si notifications bloquantes actives)
          |       -> Actions executees (local ou via QuestActionBridge)
          |
          +-- Tous objectifs completes
                    |
                    v
              CompleteQuest()
              -> FAsyncSaveQuestsToBinaryTask (+ .bak)
              -> Delegates broadcast (8 natifs + 8 dynamiques)
              -> UI notification
```

### 3.2 Flux d'une interaction joueur -> progression

```
Joueur appuie sur "Interact" (camera line-trace dans InteractionComponent)
        |
        v
UInteractionComponent::PerformInteractionTrace() -> Execute_Interact()
        |
        v
IInteractableInterface::Interact()  (DynamicInteractableComponent / QuestInteractComponent / QuestTriggerActor)
        |
        v
PlayerQuestComponent::Server_UpdateObjectiveProgress()  [Server RPC]
   (ou Server_ForwardLocationObjectiveEvent pour les objectifs de localisation/guardian)
        |
        v
QuestManagerSubsystem::RegisterObjectiveProgress()
        |
        +-- Met a jour FRuntimeQuestInstance (ObjectiveProgress / *Array)
        +-- Appelle UQuestObjectiveBase::ProcessQuestEvent() puis FinishObjective() si seuil atteint
        +-- Si complete : execute les actions liees (autorite)
        +-- Replique vers le(s) client(s) via PlayerQuestComponent
        +-- Broadcast delegates (natifs + dynamiques)
```

### 3.3 Flux reseau (actions)

```
SERVEUR                                          CLIENT
   |                                                |
   | UQuestActionBase::ExecuteAction()              |
   |   selon EQuestActionNetworkMode :              |
   |   |                                            |
   |   +-- ServerOnly        -> execution serveur   |
   |   |                                            |
   |   +-- OwningClientOnly  -> bridge vers le seul PC instigateur ---> execute client
   |   |                                            |
   |   +-- TargetedMulticast -> bridge vers les joueurs pertinents --> execute clients cibles
   |   |                                            |
   |   +-- GlobalMulticast   -> bridge vers tous les joueurs pertinents
   |   |                                            |
   |   +-- RelevancyBased    -> QuestRelevancyManager
   |         -> CreateActionBridge (AQuestActionBridge, BridgeLifetime 30s)
   |           -> IsNetRelevantFor() (TargetPlayers uniquement)
   |             -> Multicast_ExecuteAction() ---------------------> execute clients cibles
   |                                                |
   |   ExecuteClientBridgeAction (cote client) desactive
   |   progression/registre/flow pour eviter la double comptabilisation.
```

> Note : `RelevancyBased` retombe en pratique sur un **broadcast** car `QuestRelevancyManager` ne filtre reellement que par proximite (cf. 5.3 / 7.2).

---

## 4. Sous-systemes majeurs

### 4.1 QuestManagerSubsystem -- Orchestrateur central

**Fichiers :** `Subsystems/QuestManagerSubsystem.h` (818 lignes) / `.cpp` (13558 lignes), `Subsystems/QuestManagerDelegates.h` (13 lignes)

#### Architecture interne

Le subsysteme central est un `UGameInstanceSubsystem` implementant `IQuestProvider`. Il a fortement grossi depuis la redaction initiale (le header fait ~2000 lignes annoncees mais en realite **818**, l'implementation **13558**). Il reste le **point unique de verite** pour :

- `QuestDataMap` : `TMap<FName, FQuestDefinition>` -- toutes les definitions chargees
- `PlayerQuestStates` : `TMap<FName, FPlayerQuestStateMap>` -- etat runtime par joueur (le wrapper `FPlayerQuestStateMap` encapsule le `TMap<FName, FRuntimeQuestInstance>`)
- `PlayerFlags` : `TMap<FName, FPlayerFlagsMap>` -- drapeaux persistants par joueur (ex. `CameFromTutorial`), exposes via `SetPlayerFlag`/`GetPlayerFlag`/`HasPlayerFlag`/`ClearPlayerFlag`/`ClearAllPlayerFlags`
- `QuestComponentManagers` : `TMap<FName, UObject*>` -- factory/conteneur d'objets modulaires par quete
- `QuestInstanceActors` : `TMap<FName, AQuestInstanceActor*>` (Transient) -- acteurs repliques des quetes actives
- `ActiveTriggerZoneManagers` : `TMap<FName, TObjectPtr<AActor>>` (Transient) -- gestionnaires de zones par quete

Au-dela du suivi de quetes, l'orchestrateur porte aussi plusieurs responsabilites **non documentees auparavant** :

- **Machine a etats de transition d'univers persistant** : `QueuePersistentUniverseTransition()` met en file une `FPersistentUniverseTransitionRequest`. Un `FPersistentUniverseTransitionTask` traverse les etats `WaitingForFade -> WaitingForMapLoad -> WaitingForGameMode -> WaitingForTeleportDelay -> TeleportRequested`, pilote par un `FTSTicker` (`TickPersistentUniverseTransitions`) et un delegate `PostLoadMap`. Gere fade-to-black, blocage d'input, attente de la classe de GameMode, application de la respawn location, et teleport avec retry.
- **Tracking d'inventaire** : `UQuestInventoryBindingProxy` se lie a un `FScriptDelegate` decouvert sur le composant d'inventaire du joueur pour suivre les pickups d'items spawnes par quete (`FQuestTrackedSpawnedItem` / `FQuestInventoryListener`, `GatherTrackedInventoryItemsForPlayer`).
- **Quetes procedurales** : `GenerateProceduralQuestOptions`, `QueryWorldPOIs`/`QueryQLevelPOIs`, cooldowns POI par joueur, cache procedural 60s.
- **World-kill events** : `DispatchWorldKillEvent()` permet le tracking de kills procedural sans `QuestEntityComponent` ; `GetPlayersWithActiveObjective()` resout les proprietaires autoritaires d'un objectif pour crediter les kills.

#### Pattern de delegates dual

Chaque evenement est broadcast a la fois en delegate natif (C++) et en delegate dynamique (Blueprint). **8 delegates natifs** (`QuestManagerDelegates.h`) face a **8 delegates dynamiques** `BlueprintAssignable` (declares dans le header) :

| Natif (`QuestManagerDelegates.h`) | Dynamique (`UPROPERTY BlueprintAssignable`) |
|---|---|
| `FQuestManager_OnQuestDataAdded` | `OnQuestDataAddedEvent`* |
| `FQuestManager_OnQuestStateChanged` | `OnQuestStateChangedEvent` |
| `FQuestManager_OnQuestUpdated` | `OnQuestUpdatedEvent` |
| `FQuestManager_OnQuestObjectiveCompleted` | `OnQuestObjectiveCompletedEvent` |
| (objectif active -- dynamique uniquement) | `OnQuestObjectiveActivatedEvent` |
| `FQuestManager_OnQuestCompleted` | `OnQuestCompletedEvent` |
| `FQuestManager_OnQuestFailed` | `OnQuestFailedEvent` |
| `FQuestManager_OnQuestAbandoned` | `OnQuestAbandonedEvent` |
| `FQuestManager_OnQuestTrackingChanged` | `OnQuestTrackingChangedEvent` |

(*Les macros `FOnQuest*Event` dynamiques sont declarees lignes 154-162 ; `OnQuestObjectiveActivatedEvent` n'a pas de pendant natif.)

#### Persistance

- Format binaire versionne : **v4** (`DynamicQuestSystem::BinaryVersion = 4`, repris par `UQuestManagerSubsystem::CurrentBinaryVersion` et `UBinaryQuestDataSource::CurrentBinaryVersion`).
- Fichier de backup automatique `.bak` (copie via `CopyFile` avant sauvegarde).
- I/O asynchrone via `FNonAbandonableTask` -- classes `FAsyncSaveQuestsToBinaryTask` / `FAsyncLoadQuestsFromBinaryTask`, lancees par `FAutoDeleteAsyncTask`, avec callback game thread.
- Chemin de sauvegarde **segmente par port serveur** (`Port%d`, sinon `Offline` ou `Server_UnknownPort`) pour le multi-instance dedie.
- `UBinaryQuestDataSource` supporte un fichier monolithique **ou** un fichier individuel par quete (`bUseIndividualQuestFiles`).
- Thread-safety via `FCriticalSection QuestDataMutex` et `GQuestFileLock`.

#### Reconstruction differee des instances runtime (reconnexion)

Au chargement de progression (`DeserializePlayerProgressFromArchive`), toute quete `IsBinarySerialized()` est d'abord reconstruite (`QuestDef.ReconstructModularComponentsFromSerialized(this)`), puis ses instances runtime sont recreees via `InitializeActionInstancesFromDefinition` + `InitializeObjectiveInstancesFromDefinition`. C'est le chemin qui restaure objectifs et actions live a la reconnexion d'un joueur.

#### Maintenance periodique

| Timer | Intervalle | Role |
|---|---|---|
| `BlueprintCleanupTimerHandle` (`PeriodicBlueprintReferenceCleanup`) | 60s | Nettoyage refs stale (quasi no-op) |
| `ObjectiveActivationCheckTimerHandle` (`CheckInactiveObjectivesForActivation`) | 2s | Scan objectifs dormants |
| `WorldValidationTimerHandle` (`ValidateWorldContext`) | 5s | Verification contexte monde |
| Scan objectifs pickup (`PreparePickupObjectiveForActivation`) | a la demande | `TActorIterator` dans **600m** (60000 unites) pour items |
| `RealTimeValidationTimerHandle` (`RealTimeReferenceValidation`) | -- | **STUB VIDE** : `SetupRealTimeReferenceValidation` et `RealTimeReferenceValidation` ont un corps vide, le timer n'est jamais demarre |

#### Resolution de l'ID joueur

L'ID deterministe est resolu par `GetOrCreatePersistentPlayerID()` (delegue depuis `ResolvePlayerID`) :

1. `PlayerState->GetUniqueId()` (OSS/Steam `FUniqueNetIdRepl`)
2. `LocalPlayer->GetPreferredUniqueNetId()`
3. Fallback packaged/editeur : `FPlatformProcess::UserName()`

Si l'`UniqueNetId` n'est pas encore disponible, la fonction retourne `NAME_None` -- les appelants doivent **reessayer une fois le PlayerState/OSS initialise** (acceptation avant resolution d'identite). Les IDs `"0"` (`LegacyZeroPlayerID`) et `"STEAM_ID_FAILED"` sont rejetes comme invalides partout dans la cascade.

#### Pieges et subtilites

- **Transition de monde** : sur `OnPostWorldInitialization`, l'etat en memoire est efface et la progression rechargee depuis le disque via un timer de **0.5s** (`QuestProgressReloadTimerHandle` -> `ReloadQuestProgressAfterWorldTransition`). Evite les pointeurs UObject stales apres changement de niveau.
- **Detection lobby** : un nom de monde contenant `"L_Lobby"` suspend le systeme (`bIsQuestSystemSuspended`).
- **Monde tutoriel** : nom contenant `"Tutorial"` -- ne charge pas la progression sauvegardee (Q006 non persistant) mais appelle quand meme `AutoAssignStoryQuestIfNeeded`.
- **Auto-assign de quete principale decouple** : `AutoAssignStoryQuestIfNeeded` existe toujours (assigne la premiere `bIsMainQuest` eligible) mais n'est **plus** appele lors de la synchro d'ID joueur (commentaires explicites `intentionally NOT called here` dans `PlayerQuestSubsystem`/`PlayerQuestComponent`). Il ne se declenche que sur rechargement post-transition et via le `PlayerQuestComponent` a la connexion.
- **Nettoyage PIE** : `OnPIEEnd`/`OnWorldCleanup` avec GC force et derooting des objets quete.
- **`const_cast` depuis methodes const** : `GetAllQuestTags_Implementation` met a jour `CachedQuestTags` ; `GetTrackedQuests` bascule `bIsQuestSystemSuspended` (transition lobby/jeu) et retire les `QuestInstanceActors` invalides en cours d'iteration.

### 4.2 PlayerQuestComponent -- Pont reseau

**Fichiers :** `Components/PlayerQuestComponent.h` (579 lignes) / `.cpp` (6720 lignes)

#### Architecture interne

Composant `UActorComponent` replique, place sur le **PlayerController**. Il sert de **pont entre le serveur (QuestManagerSubsystem) et le client (UI / son / dialogue)**.

```
SERVEUR                          RESEAU                          CLIENT
QuestManagerSubsystem   -->  PlayerQuestComponent  -->  UI / Son / Dialogue
                              PlayerID            (Replicated)
                              bIsComponentValid   (OnRep_ComponentValidity)
                              ActiveQuests[]      (OnRep_ActiveQuests)
                              CompletedQuests[]   (OnRep_CompletedQuests)
                              FailedQuests[]      (OnRep_FailedQuests)
                              TrackedQuestInfo[]  (OnRep_TrackedQuestInfo)
```

Les 4 tableaux + `bIsComponentValid` repliquent via `DOREPLIFETIME_CONDITION_NOTIFY(..., COND_None, REPNOTIFY_Always)` (notification meme si la valeur n'a pas change). `PlayerID` repliquee via `DOREPLIFETIME` simple.

#### Structures cle

```cpp
USTRUCT()
struct FActiveLocationObjectiveInfo {
    TSet<int32> IndicesPlayerIsInside;   // indices de zones ou le joueur est present
};

USTRUCT()
struct FObjectiveKey {                   // Cle composite hashable
    UPROPERTY() FName QuestID;
    UPROPERTY() FName ObjectiveID;
    bool operator==(const FObjectiveKey&) const;
    friend uint32 GetTypeHash(const FObjectiveKey& Key);
};

USTRUCT(BlueprintType)
struct FTrackedQuestInfo {               // Repliquee
    UPROPERTY() FName QuestID;
    UPROPERTY() bool  bIsTracked;
};

USTRUCT(BlueprintType)
struct FPlayerDataCache {                // Cache local -- NON replique (voir Piege)
    FName PlayerID;
    TWeakObjectPtr<APlayerController> PlayerController;
    TWeakObjectPtr<APlayerState>      PlayerState;
    TWeakObjectPtr<APawn>             PlayerPawn;
    TWeakObjectPtr<APawn>             OriginalPlayerCharacter;
    bool bIsValid; bool bIsLocalPlayer; bool bHasAuthority;
};
```

> Note : `FActiveLocationObjectiveInfo` a ete reduit a un simple `TSet<int32>`. Le couplage QuestID/ObjectiveID/zone est porte par la map `TMap<FObjectiveKey, FActiveLocationObjectiveInfo> ActiveLocationObjectives`.

#### Optimisation de replication

`ComputeQuestReplicationSignature` calcule un CRC (`FCrc::StrCrc32`) sur (QuestID, CurrentState, AttemptCount, CompletionCount, bIsTracked, `ObjectiveProgressArray` trie, `ActiveObjectiveIDs` trie, `ActionStateArray` trie). `OnRep_ActiveQuests` compare la liste d'IDs et de signatures a la derniere replication et **saute la reconstruction** si rien n'a change. Ce fix a supprime ~360 reconstructions inutiles par session.

#### Verification de position (timer a frequence dynamique)

`EnsureLocationTimerRunning` / `CheckActiveLocationObjectives` replanifient le timer selon `MinDistanceRatio` (distance / taille de zone effective, `MinEffectiveZoneSize` = 200) :

| Ratio distance | Rate | Frequence |
|---|---|---|
| <= 1.5x | 0.05 s | 20 Hz (dans/tres proche) |
| <= 2.5x | 0.10 s | 10 Hz |
| <= 5x   | 0.33 s | 3 Hz |
| <= 10x  | 1.0 s  | 1 Hz |
| <= 25x  | 3.0 s  | 0.33 Hz |
| > 25x   | 5.0 s  | 0.2 Hz |

Defaut sans info de distance : 0.33 s. Le timer est arrete quand `ActiveLocationObjectives` est vide.

#### RPCs

| Direction | RPC | Usage |
|---|---|---|
| Client->Serveur | `Server_AcceptQuest` | Accepter une quete |
| Client->Serveur | `Server_AbandonQuest` | Abandonner une quete |
| Client->Serveur | `Server_ClearAllQuests` | Tout effacer |
| Client->Serveur | `Server_RequestQuestDialogue` | Ouvrir dialogue NPC |
| Client->Serveur | `Server_RequestAcceptQuest` | Acceptation via dialogue NPC |
| Client->Serveur | `Server_DispatchDialogueEvent` | Evenement de dialogue (phase/index) |
| Client->Serveur | `Server_UpdateObjectiveProgress` | Rapporter progression |
| Client->Serveur | `Server_ForwardLocationObjectiveEvent` | Forward guardian : LocationVisited / LocationExited |
| Client->Serveur | `Server_NotifyCableInteraction` | Interaction cable (contourne la limite RPC des acteurs places) |
| Client->Serveur | `Server_SetQuestTracked` | Changer le suivi |
| Client->Serveur | `Server_SetAuthoritativePlayerID` | Propager le PlayerID resolu cote client |
| Serveur->Client | `Client_ExecuteSpawnAction` | Spawn cote client |
| Serveur->Client | `Client_SynchronizeActionState` | Sync etat action |
| Serveur->Client | `Client_PlayQuestAcceptSound` | Son acceptation |
| Serveur->Client | `Client_PlaySpatialSoundAtLocation` | Son spatial |
| Serveur->Client | `Client_OpenQuestDialogue` | Ouvrir popup dialogue |
| Serveur->Client | `Client_HandleQuestAcceptanceResult` | Resultat d'acceptation (succes/echec + raison) |
| Serveur->Client | `Client_TriggerObjectiveNotification` | Notification objectif |
| Serveur->Client | `Client_NotifyObjectiveCompleted` | Objectif complete |

> Aucun de ces RPC n'est un `NetMulticast` (les multicasts existent sur `AQuestActionBridge`, pas ici).

#### Forward guardian (correction MP)

Pour les objectifs de type « gardien », le **client** detecte l'entree/sortie de zone (`ProcessLocationObjectiveHit` / `ProcessLocationObjectiveExit`) puis appelle `Server_ForwardLocationObjectiveEvent` avec `EventType = LocationVisited` ou `LocationExited`. Le serveur rejoue l'evenement sur son instance d'objectif autoritaire (`ProcessQuestEvent`) pour mettre a jour son propre etat (`bSoftCompleted`). Sans ce forward, l'etat guardian n'etait jamais maj cote serveur en multijoueur.

#### Piege

`FPlayerDataCache` est explicitement **non replique** : `UPROPERTY(Transient)`. Un commentaire dans le header explique que repliquer ses champs `TWeakObjectPtr` etait *unsafe* et empechait `ActiveQuests` (et autres) de se repliquer de maniere fiable en build package.

---

### 4.3 Systeme d'objectifs

**Fichiers :** `DynamicQuestSystem/Objectives/QuestObjectiveBase.h` (480) / `.cpp` (2214) (classe de base abstraite) + module `DynamicQuestSystemObjectives/` (les 11 sous-classes).

> Note : il n'existe **pas** d'enum `EObjectiveType` listant les types d'objectifs. Le type d'un objectif est un `FName` renvoye par `GetObjectiveType_Implementation()`. La base derive ce nom du nom de classe (`U...Objective` -> ex. `Location`, `TriggerOnly`, `NotificationPlayback`) ; les sous-classes le surchargent (`Kill`, `Interact`, `Custom`, `Connector`, `Scanner`, `EGState`, `QLevel`, `NPCDialogue`).

#### Hierarchie des objectifs

```
UQuestObjectiveBase (core, abstract, EditInlineNew, DefaultToInstanced)
    |
    +-- UConnectorObjective       Puzzle cable/socket (FConnectorPair, RopeCable, audio MP)
    +-- UCustomObjective          Extension Blueprint pure (Blueprintable)
    +-- UEGStateObjective         Surveillance d'etat "Environmental Gadget" (OptimizedStateComponent)
    +-- UInteractObjective        Interaction temporisee
    +-- UKillObjective            Suivi de kills avec filtrage riche + self-repair de spawn
    +-- ULocationObjective        Visite de lieux (la plus complexe, 2328 lignes)
    +-- UNPCDialogueObjective     Dialogue NPC avec items requis (reflexion)
    +-- UNotificationPlaybackObjective  Attente de notifications bloquantes
    +-- UQLevelLoadObjective      Attente chargement/dechargement de QLevel
    +-- UScannerObjective         Declenchement scanner (reflexion + filtre global)
    +-- UTriggerOnlyObjective     Auto-completion immediate (declencheur d'actions)
```

#### Cycle de vie d'un objectif

```
                 Instancie / InitializeObjective(QuestOwner)
                          |
                          v
   [Inactif] --Activate--> OnActivated()  --(ActivationDelay)
                          |
   ProcessQuestEvent(EventType, Params, OutProgress)  <-- evenements de quete
                          |  (l'objectif fixe OutProgress / appelle IncrementProgress)
                          v
   QuestManager->RegisterObjectiveProgress / UpdateQuestObjectiveProgress
                          |
                          v
                  FinishObjective()
       (differe si notifications bloquantes actives -> reprise sur dismissal)
                          |
        +-----------------+------------------+
        |                                    |
     OnCompleted()                       OnFailed()       (+ OnReset, OnResumed)
   (CompletionDelay)                  (bFailsQuestOnFailure)
        |                                    |
   Execute actions, notifs OnCompleted   notifs OnFailed
```

Notes :
- Il **n'existe pas** de methode `ProgressObjective()`. La progression passe par `ProcessQuestEvent_Implementation()` (qui ecrit `OutProgress`), `IncrementProgress(Amount)` et `FinishObjective()` qui delegue a `UQuestManagerSubsystem`.
- L'execution des actions et le spawn des trackers ne se font **que sur autorite** (`NM_Standalone`/`NM_DedicatedServer`/`NM_ListenServer`) ; `FinishObjective` ignore un objectif deja marque `IsObjectiveFailed`.
- `FinishObjective` est differee tant que des notifications **bloquantes** (`bIsBlocking`) sont actives pour l'objectif, puis reprise via `HandlePendingCompletionAfterNotificationsCleared`.
- `OnResumed()` est appele periodiquement par le serveur pour les objectifs actifs : il rejoue `OnActivated` sous la garde d'activation native **mais saute** l'execution des actions et les notifications, et conserve le tracker existant (spawn idempotent) pour eviter le clignotement du marqueur.
- Conditions modulaires de la base : `ActivationConditions` (ET), `FailureCondition`, `ProgressCondition`, et `SkipConditions` (OU -> auto-completion **sans** executer les actions).

#### Pattern de garde d'activation

```cpp
// QuestObjectiveBase garantit que la logique native s'execute meme si un
// Blueprint override oublie d'appeler Super::OnActivated.
void UQuestObjectiveBase::OnActivated_Implementation()
{
    bHasExecutedNativeOnActivated = true;
    ExecuteNativeOnActivatedLogic();   // -> PerformObjectiveActivation()
}

// Au point d'appel (ex. OnResumed) :
StartNativeActivationGuard();                       // bHasExecutedNativeOnActivated = false
OnActivated();                                       // logique BP eventuelle
EnsureNativeOnActivatedExecuted(TEXT("Context"));    // execute la base si le BP a saute Super
```

#### Objectifs notables

**LocationObjective** (2328 lignes) : la plus complexe. Visites sequentielles ou non-ordonnees (`bOrderMatters`, `bRequireAllLocations`, `LocationsVisited`), systeme de **"guardian objective"** (`GuardianObjectiveID` -> "soft-completion" : entree dans la zone => `bSoftCompleted`/`SoftCompletedLocationIndex`, le tracker disparait en attendant la completion du gardien, et reapparait si le joueur ressort via `HandleGuardedLocationExit`). Les soft-completes exposent `IsSoftCompleted`/`ShouldSuppressProgressUpdate`/`ShouldHideInTracker`. Spawn de trackers, integration editeur bidirectionnelle avec `ALocationObjectiveZoneActor` (`#if WITH_EDITOR` : CreateLocationZoneActors / ReconstructLocationZoneActors / Cleanup...). En multijoueur, `ProcessQuestEvent` traite les evenements **`LocationVisited`** et **`LocationExited`** transmis cote serveur (correctif MP guardian).

**KillObjective** : filtrage riche -- `TargetFactions` (`EEnemyFaction`), `EnemyType`, `SpecificTargetNames`/`bAnyEnemyOfType`, `MinimumEnemyLevel`, `RequiredWeaponType`, `RequiredStatusEffect`, `bRequireCriticalKill`, `bRequireStealthKill`, fenetre temporelle (`KillTimeWindow`/`KillsPerTimeWindow`/`KillTimestamps`), `bCountAssists`, `bOnlyTargetQuestSpawned`. **Self-repair de spawn** : si `bEnsureQuestSpawnAvailability`, un timer (`SpawnAvailabilityCheckInterval`) compte les entites de quete actives (`CountActiveQuestEntities`) et, sous le buffer, re-execute les `SpawnMaintenanceActionIds` (cooldown `SpawnAvailabilityRetryCooldown`).

**ScannerObjective** : integration **entierement par reflexion** avec l'inventaire et le scanner (`FindFunction`/`ProcessEvent`/`FindFProperty`, noms centralises dans `ScannerObjectiveNames`), aucune dependance compile-time. Liaison via timer de retry (`TryBindToScanner`, `BindingRetryDelay`, `MaxBindingAttempts`), declenchement unique (`bScannerTriggered`). Filtre de scanner **statique global** `TMap<FName, FScannerFilterConfig> ActiveScannerFilters` (par joueur) utilise par l'UI du scanner. Correctif `995a4122` : apparition des trackers calee sur la vague de scan (fin du delai ~10s + fix blink).

**ConnectorObjective** : puzzle cable/socket. `FConnectorPair` a **resolution duale** (`CableTag`/`CableActor` et `TargetConnectorTag`/`TargetConnectorActor`), cable via `URopeCableComponent`. Audio spatial : en Standalone `PlaySoundAtLocation` direct ; en multijoueur le son est joue cote serveur via `UPlayerQuestComponent::Client_PlaySpatialSoundAtLocation` (RPC client), position resolue sur `ACableConnectorActor::GetConnectorTipComponent` ou `URopeCableComponent::GetEndLocation`.

**EGStateObjective** : surveille un `OptimizedStateComponent` (par reflexion : `CallOptimizedStateGetBool/Int32/Name`) sur des acteurs cibles selectionnes par `EGTags` (ET/OU) ou `DirectEGReferences`. Conditions `FEGStateCondition` (Bool/Int32/Name) avec `EEGStateComparison` (Equal/NotEqual/GreaterThan/LessThan/GreaterOrEqual/LessOrEqual), logique ANY/ALL sur les cibles.

**QLevelLoadObjective** : attend le **chargement OU le dechargement** (`bWaitForUnload`) d'un `UQLevel_Asset` (via `UQLevel_SubSystem`/`UQLevel_Streaming_Instance`) ; `bCompleteImmediatelyIfLoaded`.

**NotificationPlaybackObjective** : joue des notifications configurees et se complete automatiquement quand les notifications bloquantes sont levees (poll `NotificationPollInterval`/`MaxNotificationPollAttempts` + `RegisterOnBlockingNotificationsCleared`).

**TriggerOnlyObjective / CustomObjective** : `TriggerOnly` se complete immediatement a l'activation (declencheur d'actions). `Custom` est une extension Blueprint pure (`Blueprintable`).

### 4.4 Systeme d'actions

**Fichiers :** `Actions/QuestActionBase.h` (408) / `.cpp` (2074) (module Core) + module `DynamicQuestSystemObjectives/Actions/`

#### Architecture

Les actions sont des effets de bord declenchables par le systeme de quete. La classe de base (`UQuestActionBase`, 2074 lignes de `.cpp`) gere :

- Execution avec delai (`ExecutionDelay`), retries (`MaxRetries`/`RetryDelay`/`bAutoRetryOnFailure`), et condition d'activation (`ExecutionCondition`)
- 5 modes reseau via `EQuestActionNetworkMode` (defaut `ServerOnly`)
- Integration notifications (blocage de completion pendant affichage)
- Integration registre dynamique (`bStoreResultsInRegistry` / `RegistryKeyPrefix`)
- Chaines d'execution (flow `OnSuccess/OnFailure/OnRetryTargetID`, `ExecuteActionByID`/`Instance`/`CreateAndExecuteAction`)
- Consommation d'inventaire (`RemoveRegisteredInventoryItems`, par reflexion sur `RemoveItemFromInventory`)
- Mise a jour de progression d'objectif (`bUpdateObjectiveProgress`)

```cpp
// Modes reseau REELS (QuestActionBase.h)
UENUM(BlueprintType)
enum class EQuestActionNetworkMode : uint8
{
    ServerOnly,          // execution serveur uniquement (defaut)
    OwningClientOnly,    // bridge vers le seul PC instigateur
    TargetedMulticast,   // bridge vers les joueurs pertinents
    GlobalMulticast,     // bridge vers tous les joueurs pertinents
    RelevancyBased       // idem, gere par le QuestRelevancyManager
};
```

#### Actions disponibles

15 actions au total (14 dans `Objectives` + `ControlAmbientAudioAction` dans le module Core).

| Action | Module | Role | Complexite |
|---|---|---|---|
| `ActivateInteractableAction` | Objectives | Spawn/active un objet interactif | Moyenne |
| `ControlAmbientAudioAction` | **Core** | Override entree ambiante + lecture "Next" differee (timer) | Moyenne |
| `ControlLevelSoundAction` | Objectives | Controle composants audio | Moyenne |
| `HighlightActorAction` | Objectives | Outline/highlight d'acteurs (resolution par tags) | Moyenne |
| `OpenPersistentUniverseAction` | Objectives | Transition de niveau (fade) | Faible |
| `PlayLevelSequenceAction` | Objectives | Cinematique complete (~30 proprietes) | **Haute** (1116 l.) |
| `PlayNiagaraAction` | Objectives | Effet particules | Faible |
| `PlaySoundAction` | Objectives | Son 2D/spatial/attache | Faible |
| `QuestActionEGControl` | Objectives | Controle etat Environmental Gadget (reflexion) | **Haute** (891 l.) |
| `QuestActionLightControl` | Objectives | Controle lumieres (transitions) | **Haute** (922 l.) |
| `QuestNotificationPlayAction` | Objectives | Notification UI | Faible (39 l.) |
| `SetObjectiveStateAction` | Objectives | Active/complete/echoue un objectif | Moyenne |
| `SpawnActorAction` | Objectives | Spawn d'entites (ennemis, objets) | **Haute** (1002 l.) |
| `TeleportAction` | Objectives | Teleportation joueur | Moyenne |
| `TriggerActorAction` | Objectives | Appel de fonction par reflexion (`ProcessEvent`) | Moyenne |

#### Pattern d'execution

```cpp
// ExecuteAction_Implementation aiguille selon bUseAttemptExecutionMethod :

// 1. Fire-and-forget (void) -- succes presume
virtual void OnExecuteAction_Implementation(const TMap<FName,FString>& P, UObject* W);

// 2. Avec retour (bool) + raison d'echec -- declenche les retries
virtual bool AttemptActionExecution_Implementation(..., FString& OutFailureReason);

// Les actions async posent bManualCompletionOnly=true et appellent
// manuellement MarkActionCompleted() / MarkActionFailed(Reason) a la fin.
```

Sequence : `ExecuteAction` -> (si non ServerOnly sur serveur) `ExecuteWithNetworking` (bridge) -> attend le PC local pret (`IsLocalPlayerControllerReady`, sondage 0.05s) -> verifie `ExecutionCondition` -> applique `ExecutionDelay` (timer) -> `ExecuteActionInternal` -> `OnExecuteAction`/`AttemptActionExecution` -> succes -> `MarkActionCompleted` (sauf `bManualCompletionOnly`).

#### Resolution d'acteurs cibles

Pattern commun des actions ciblant des acteurs (ex. `HighlightActorAction`) :
1. Reference directe (`TSoftObjectPtr`)
2. Recherche par tag (`TActorIterator` + `ActorHasTag`)
3. Filtrage par nom
4. Filtrage par classe
5. Cle de registre dynamique

#### Reseau & bridge

Les modes != `ServerOnly` creent un `AQuestActionBridge` cote serveur (via `UQuestRelevancyManager::CreateActionBridge`) qui rejoue l'action serialisee chez les clients cibles (`BridgeLifetime`, defaut 30s). Cote client, `ExecuteClientBridgeAction` desactive progression/registre/flow pour eviter la double comptabilisation.

---

### 4.5 Systeme de conditions

**Fichiers :** `Conditions/QuestConditionBase.h` (86) / `.cpp` (193) (Core) + module `DynamicQuestSystemObjectives/Conditions/`

#### Architecture

Evaluateurs invoques par le systeme de quete (NON stateless : la base cache son dernier resultat). La classe de base supporte :

```cpp
UENUM(BlueprintType)
enum class EQuestConditionType : uint8
{
    None,
    QuestAcceptance,
    ObjectiveActivation,
    ObjectiveProgress,
    ObjectiveFailure,
    ActionExecution,
    ObjectiveSkip
};
```

- Inversion (`bInvertResult`) appliquee au resultat brut dans `CheckCondition_Implementation`
- Cache de resultat (`bLastCheckResult`/`bHasBeenChecked`, SaveGame) via `UpdateCachedResult`
- Delegate `OnConditionStateChanged` diffuse uniquement sur transition d'etat
- Serialisation binaire (override `Serialize`)
- Les classes derivees surchargent `EvaluateCondition_Internal` (pas `CheckCondition`)

#### Conditions disponibles

| Condition | Evaluation |
|---|---|
| `CheckPlayerFlagCondition` | `QuestManager->HasPlayerFlag(PlayerID, FlagName)` (PlayerID lu dans les Parameters) |
| `IsMultiplayerCondition` | `World->GetNetMode() != NM_Standalone` |
| `LevelSequenceCondition` | Cinematique terminee (scoring heuristique multi-acteurs sur `ALevelSequenceActor`, mode `bWaitForCompleteStop`) |

> **Piege logging** : `CheckCondition_Implementation` emet toujours jusqu'a 5 `DQS_LOG_WARNING` par evaluation (et `LevelSequenceCondition` davantage). Ces logs n'ont PAS ete convertis en `Verbose` ; ils sont seulement neutralises au runtime par le garde global `GDQSLoggingEnabled` (desactive par defaut depuis b7d5ead7). Le commit 388a1f73 (Error->Verbose) ne concernait que le chemin de deserialisation des actions, pas les conditions.

### 4.6 Systeme de dialogue

**Fichiers :** `UI/QuestDialogue*.h/.cpp`, `UI/QuestNPCMenuWidget.h/.cpp`, `UI/QuestUIManager.h/.cpp`, `Actors/QuestTriggerActor.h/.cpp`, `UI/QuestInputModeHelper.h`

#### Architecture

```
AQuestTriggerActor (NPC)                 247L .h / 1476L .cpp
    |
    +-- DialogueLines[]            (FQuestDialogueLine : Text + Sound soft-ref)
    +-- DialogueBranches[]         (branches conditionnelles priorisees)
    +-- AvailableQuestIDs[]        (ou mode procedural : bUseProceduralQuests)
    +-- Head look-at + scannable + spawn d'objectifs
    |
    v
ResolveDialogueForPlayer()  -> branche valide de plus haute priorite
    |
    v
FQuestDialogueSessionData
    { NPCName, NPCActorName, DialogueLines[], QuestOptions[] }
    |
    v
UQuestDialoguePopupWidget                237L .h / 2724L .cpp
    |-- Machine a etats : None -> Dialogue -> QuestList -> FinalChoice
    |-- Animation de frappe (typewriter via QuestTextFormatter)
    |-- Navigation clavier / souris / molette / gamepad
    |-- Lecture audio VO (UAudioComponent fortement reference, attache au NPC)
    |-- Panneau de details (UQuestDialogueDetailsWidget)
    |-- Boutons Accept / Decline / Skip (EQuestFinalAction)
    |-- Capture / restauration du mode input
```

> Piege : la machine a etats `None->Dialogue->QuestList->FinalChoice` vit dans **UQuestDialoguePopupWidget** (enum prive `EQuestDialogueStage`), PAS dans `UQuestNPCMenuWidget`. Ce dernier est l'UI "contract machine" / menu NPC classique.

#### Branches conditionnelles

Les branches sont evaluees par priorite (`Priority`, la plus haute branche valide gagne). Chaque branche porte ses propres conditions ET ses lignes de dialogue.

```cpp
struct FQuestDialogueBranch {
    int32 Priority;                              // Plus eleve = prioritaire
    TArray<FQuestDialogueCondition> Conditions;  // TOUTES doivent passer
    TArray<FQuestDialogueLine> DialogueLines;    // /!\ DialogueLines (pas "Lines")
    bool bAllowObjectiveOverrides = true;        // Autorise les overrides d'objectif (NPCDialogue) a remplacer ces lignes
};

struct FQuestDialogueCondition {
    FName QuestID;                               // Vide = pas d'exigence de quete
    EQuestDialogueQuestState RequiredQuestState; // defaut Any
    bool bUseObjectiveState = false;             // /!\ gate l'evaluation de l'objectif
    FName ObjectiveID;                           // evalue seulement si bUseObjectiveState
    EQuestDialogueObjectiveState RequiredObjectiveState; // evalue seulement si bUseObjectiveState
};

// /!\ Enums reels (l'ordre et les noms comptent) :
enum class EQuestDialogueQuestState : uint8 {
    Any, NotStarted, Available, InProgress, ObjectivesCompleted, Completed, Failed
};
enum class EQuestDialogueObjectiveState : uint8 {
    Any, NotStarted, Active, Completed, Failed
};
```

#### Overrides d'objectif (fix VO UE5.7 — 5cf2736e)

Apres `ResolveDialogueForPlayer`, si la branche retenue autorise `bAllowObjectiveOverrides`, un scan des objectifs de type NPCDialogue peut remplacer les lignes. Depuis le fix post-migration UE5.7, ce scan **s'arrete au premier override applicable** (plus d'ecrasement silencieux par une autre quete active). Le meme commit a corrige le **playback audio VO** : le `UAudioComponent` est desormais cree, *fortement* reference (`UPROPERTY(Transient) ActiveDialogueAudio`), attache/registre proprement au monde du NPC, puis stoppe et `DestroyComponent()` explicitement — l'ancienne weak-ref ne survivait pas en 5.7 (seule la 1re ligne jouait). Le son n'est PAS synchronise au typewriter ; la synchro texte<->audio adaptative n'existe que pour les notifications de type Communication.

#### Blocage des inputs gameplay (e08d3445) + helper partage (4735899f)

Pendant un dialogue ou une contract machine, le popup/menu passe le PlayerController en `FInputModeGameAndUI`, met `IgnoreMove/LookInput` et `bShowMouseCursor`, puis appelle le helper statique partage `BroadcastInputModeChanged(PC, bIsFocusedInUI)` (`UI/QuestInputModeHelper.h`). Ce helper retrouve par reflection le `WidgetGlobalComponent` du jeu hote, lit son delegue multicast `OnInputModeChanged` et sa propriete bool `IsFocusedInUI`, et notifie l'etat focus-UI sans dependance directe au code du jeu. La `ContractMachine` (`InitializeForContractMachine(bKeepGameInput)`) peut **conserver** l'input gameplay (`bKeepGameInput=true` => pas de `SetupInputMode`).

#### Role de QuestUIManager

`UQuestUIManager` (UObject simple, 63L .h / 282L .cpp) centralise : (a) le widget tracker via un `UQuestTrackerContainerWidget` (`EnsureTrackerContainer` / `AttachWidgetToViewport`, Show/HideQuestTracker), et (b) les notifications toast quete (`ShowQuestUpdate/Completion/NewQuestNotification`) construites en `FQuestNotificationPayload` (`BuildToastPayload`) et dispatchees via `QuestNotificationUtils::SendNotification`. Il ne contient PAS de logique de blink tracker (le fix blink 995a4122 concerne le systeme scanner/trackers, hors de ce plugin).

---

### 4.7 Editeur de quetes

**Fichiers :** Module `DynamicQuestSystemEditor/` (71 fichiers). Point d'entree : `QuestEditorWindow.cpp` (6497 lignes).

#### Architecture

```
+--------------------------------------------------------------+
|  FQuestEditorWindow (Nomad Tab - SCompoundWidget)            |
|                                                              |
|  +-----------+   PANNEAU LATERAL = SQuestList (arbre DnD)    |
|  | SQuestList|   ----------------------------------------    |
|  | (arbre)   |   CENTRE = SWidgetSwitcher pilote par         |
|  +-----------+   l'enum EQuestEditorTab (8 onglets) :        |
|                                                              |
|  +---------+ +-----------+ +---------+ +------+              |
|  | Details | | Objectives| | Actions | | NPCs |              |
|  +---------+ +-----------+ +---------+ +------+              |
|  +-------------+ +-----------+ +----------+ +--------------+ |
|  | TriggerZones| | QuestFlow | | Settings | | PlayerProgress| |
|  +-------------+ | (graphe)  | +----------+ +--------------+ |
|                 +-----------+                                 |
+--------------------------------------------------------------+
        |                                  |
        v                                  v
UDynamicQuestSystemEditorSubsystem    UQuestFlowGraph
  (cache memoire, locking, JSON)       (lecture seule)
        |
        v
UEditorJsonQuestDataSource
  (fichiers JSON sur disque)
```

> **Piege diagramme :** `QuestList` N'est PAS un onglet du switcher -- c'est l'arbre lateral permanent. Les onglets du `SWidgetSwitcher` sont definis par `enum class EQuestEditorTab : uint8` (`QuestEditorWindow.h:39`) qui contient exactement, dans l'ordre : `Details, Objectives, Actions, NPCs, TriggerZones, QuestFlow, Settings, PlayerProgress`.

> **Nouvel onglet `PlayerProgress`** (`QuestEditorWindow.cpp:5833`, `CreatePlayerProgressTabContent`) : charge un fichier de sauvegarde (saisie `PlayerProgressPlayerIDInput`) pour inspecter la progression de quete d'un joueur depuis l'editeur.

#### Persistance editeur

Trois generations coexistent :
1. **Binaire legacy** (`.bin`) -- supporte pour migration uniquement.
2. **JSON** (actuel) -- un fichier par quete `Quest_<ID>.json` dans le **Content DU PLUGIN** : `DynamicQuestSystem/Content/Quests/EditorQuests/` (chemin via `GetDefaultSavePath()` = `QuestPlugin->GetContentDir()/Quests`, fallback `ProjectContentDir/Quests` seulement si le plugin est introuvable ; puis `/EditorQuests`). Voir `DynamicQuestSystemEditorSubsystem.cpp:392-406`.
3. **Export binaire** (`RuntimeQuests.bin`) -- pour le jeu package, dans `Content/QuestSystem/RuntimeQuests.bin` (`GetRuntimeBinaryPath()`, :976).

**Migration automatique** : dans `LoadQuestFromFile`, si le `.json` est absent mais qu'un `.bin` legacy existe, il est lu via `FModularQuestArchiveProxy`, re-sauve en JSON, puis le binaire est archive en `.legacy` (`MoveFile`). Voir `DynamicQuestSystemEditorSubsystem.cpp:509-558`.

**Sauvegardes incrementales / async** : le subsystem suit les quetes modifiees (`ModifiedQuests` `TSet`, `MarkQuestAsModified`/`SaveModifiedQuests`) et peut ecrire le JSON en arriere-plan (`TryBuildQuestJsonPayload` + `QueueAsyncQuestJsonWrite`, payload copie avant dispatch). `UEditorJsonQuestDataSource` cree des backups `.bak` par defaut (`bCreateBackups=true`).

#### Systeme de verrouillage

Verrouillage pessimiste par fichier (`.lock` dans `ProjectSaved/QuestEditorLocks/`) avec detection de verrous stales (timeout **300s code en dur**, `GetLockTimeoutSeconds()` retourne `300.0f`). `FQuestLockInfo` (USTRUCT) stocke `User`/`Machine`/`Timestamp` (int64) et se (de)serialise via `ToString`/`FromString` sur 3 lignes `cle: valeur` dans le fichier `.lock`. Voir `DynamicQuestSystemEditorSubsystem.h:16-61`, `.cpp:1234,1246-1274`.

#### Graphe visuel (QuestFlowGraph)

Un systeme `UEdGraph` complet avec schema, nodes, pins, politique de dessin de connexions (`DrawSplineWithArrow`, bezier + fleches + glow + pulse) et widgets Slate. **Le graphe est en lecture seule** : il visualise la structure de la quete (`CreateNodesFromQuestDefinition`/`CreateConnectionsFromQuestDefinition`) mais `SaveToQuestDefinition()` a un corps vide (`QuestFlowGraph.cpp:35-42`). Les nodes sont colores par type (`QuestStart`, `Objective`, `Action`, `Condition`, `QuestComplete`) avec animations d'etat.

#### Editeurs modulaires

Deux generations coexistent toujours :
1. `SModularComponentEditor<T>` (ancien, simple, 1221 lignes) -- son header reste reference dans `DynamicQuestSystemModularEditors.cpp`.
2. `SCategorizedComponentEditor<T>` (actuel, 2166 lignes -- categories, drag-and-drop, copier/coller).

Les facades de `SModularQuestEditors.cpp` (`SModularQuestObjectivesEditor`, `SModularQuestActionsEditor`, `SModularQuestConditionsEditor`) instancient **`SCategorizedComponentEditor<T>`** pour `UQuestObjectiveBase`, `UQuestActionBase`, `UQuestConditionBase`.

> **Decouverte de classes (refonte 5.7)** : `FDynamicQuestSystemModularEditors::GetDerivedClasses` n'utilise plus le helper `LoadClassFromPath` (retire par le commit 2a357794 a cause d'un crash `ContentBrowserAssetDataSource` lors d'un chargement async). Elle resout desormais les classes via `FindObject<UClass>` seul (aucun `StaticLoadObject`/`LoadObject` synchrone), avec mise en cache dans `CachedDerivedClassesMap` (`DynamicQuestSystemModularEditors.cpp:326-343`).

---

### 4.8 Systeme d'interaction

**Fichiers :** Module `DynamicQuestSystemInteraction/` (13 fichiers source : 7 `.h` + 6 `.cpp`, dependances minimales, IWYU Full)

#### Architecture

Module **completement autonome** sans aucune dependance sur le systeme de quete :

```
IInteractableInterface (9 methodes BlueprintNativeEvent)
        ^
        |
AInteractableActor (UCLASS Abstract, IInteractableInterface, implementations par defaut)
        ^
        |
UInteractionComponent (detection par line trace timer-driven)
        |
        v
UInteractionSubsystem (GameInstanceSubsystem : registre global, index par tag, recherche spatiale)
```

Les 9 methodes de `IInteractableInterface` : `CanInteract`, `OnFocus`, `OnEndFocus`, `Interact`, `GetInteractableName`, `GetInteractionActionText`, `GetInteractionDescription`, `GetInteractionDistance`, `GetInteractableTags`. `FInteractionData` (TargetActor / SourceActor / bSuccess / InteractionTags / ResultValue) est le retour de `Interact`.

#### Detection

`UInteractionComponent` effectue des line traces depuis le viewpoint du joueur (`PC->GetPlayerViewPoint`). En `BeginPlay`, si `InteractionTraceInterval > 0` (defaut **0.1s**) un timer repete appelle `PerformInteractionTrace`. La distance utilisee est `DefaultInteractionDistance` (**250cm** sur le composant). Sur chaque hit, l'acteur est retenu s'il implemente l'interface **ou** si l'un de ses composants l'implemente (`GetComponentsByInterface`). Le composant gere l'etat de focus (`Execute_OnFocus`/`OnEndFocus`), les delegates `OnInteractableFound`/`OnInteractableLost`/`OnInteraction`, et le cycle de vie du `UModularInteractionPromptWidget` (`SetInteractableActor` + `AddToViewport`/`RemoveFromParent`). `UInteractionSubsystem` indexe les interactables par `FGameplayTag` (`TMap<FGameplayTag, FTaggedActorList>`) et offre `FindInteractablesWithTag/AllTags/AnyTags`, `FindNearestInteractable` (recherche par distance au carre) et `SetInteractablesEnabled`.

#### Limitations connues

- La distance configuree sur `AInteractableActor::InteractionDistance` (200cm) — et plus generalement `IInteractableInterface::GetInteractionDistance()` — n'est **jamais interrogee** par `InteractionComponent` qui utilise sa propre `DefaultInteractionDistance`. La propriete de l'acteur est donc effectivement morte.
- `PrimaryComponentTick.bCanEverTick = true` est conserve en permanence ; le corps du tick ne fait du travail **que** si `InteractionTraceInterval <= 0` (chemin de repli a 0.1s code en dur). Avec l'intervalle par defaut, c'est le timer qui pilote la trace et le tick est un no-op — cout de tick gaspille.
- `UModularInteractionPromptWidget::NativeConstruct()` remet `CurrentInteractableActor` a null, ce qui race avec `SetInteractableActor` appele **avant** `AddToViewport` (qui declenche `NativeConstruct`). Le prompt peut ne jamais afficher de donnees.
- `UInteractionSubsystem::SetInteractablesEnabled` n'agit que sur les sous-classes de `AInteractableActor` (`Cast<AInteractableActor>`) ; les acteurs qui implementent seulement l'interface (ou hebergent un composant interface) sont silencieusement ignores.

---

### 4.9 Systeme de debug

**Fichiers :** Module `DynamicQuestSystemDebug/` (8 fichiers, deps : Core + CoreUObject/Engine/Slate/SlateCore/DynamicQuestSystem)

#### Commandes console

6 commandes enregistrees par des `FAutoConsoleCommand` statiques globaux dans `QuestDebugSubsystem.cpp` :

| Commande | Usage |
|---|---|
| `dqs.ToggleDebugUI` | Active/desactive l'overlay Slate |
| `dqs.AcceptQuest <QuestID>` | Force l'acceptation d'une quete pour le joueur local |
| `dqs.ClearAllQuests` | Efface toute la progression du joueur local |
| `dqs.ExecuteAction <QuestID> <ActionID> [PlayerID ou Key=Value]` | Execute une action specifique |
| `dqs.CompleteObjective [QuestID]` | Complete le(s) objectif(s) actif(s) via le pipeline complet |
| `dqs.CompleteQuest [QuestID]` | Complete tous les objectifs via le pipeline complet |

> **Note (commit c4b80f91)** : `FindGameplayContext` resout desormais un contexte de quete non seulement en PIE/Standalone/Client mais aussi en `NM_DedicatedServer` (iteration des PlayerControllers, resolution de `UPlayerQuestComponent` depuis le premier PC ou son pawn ; `FQuestDebugContext::bIsDedicatedServer`). Aucune nouvelle commande n'a ete ajoutee — l'ensemble reste a 6.

#### Overlay Slate (SDynamicQuestDebugWidget, ~1357 lignes)

Widget Slate temps reel avec **3 panneaux** : `Quest Overview` (haut-centre), `Actions` et `Objectives` (haut-droite), plus une section `Dynamic Registry` en `SExpandableArea` `InitiallyCollapsed(true)`. Refresh **1Hz** (`UpdateTimer >= 1.0f`), avec auto-desactivation du tick (`SetCanTick(false)`) si le subsystem proprietaire disparait ou si `World->bIsTearingDown`.

Le widget utilise un pattern **snapshot-and-diff** : `GatherQuestSnapshots` construit des `FQuestSnapshot`/`FObjectiveSnapshot`/`FActionSnapshot`, puis `HasStructureChanged` decide entre `RebuildQuestSections` (reconstruction complete) et `UpdateInPlace` (mise a jour des widgets caches via `FCachedQuestSection`). Les actions reussies sont fanees puis masquees (`SuccessfulActionCompletionTimes` / `PermanentlyHiddenActions` / `CleanupExpiredSuccessfulActions`).

L'inspection du registre dynamique utilise la **reflexion UE** (`TFieldIterator<FProperty>` + comparaison de `PropName`) sur **13 noms de proprietes hardcodes** de `UQuestDynamicRegistry` (`StringValues`, `IntValues`, `FloatValues`, `BoolValues`, `VectorValues`, `RotatorValues`, `TransformValues`, `ObjectReferences`, `ActorReferences`, `ClassReferences`, `SoftAssetReferences`, `ObjectCollections`, `ActorCollections`) — **fragile** si le schema du registre change.

#### Dette / vestige

`FQuestDebugCommands` (`TCommands`) reste vestigial : `RegisterConsoleCommand()` et `UnregisterConsoleCommand()` sont des **stubs vides**, et `Register()`/`Unregister()` (appeles au startup/shutdown du module) y routent. Seul le `UI_COMMAND` `ToggleDebugDisplay` est declare, sans usage. Les vraies commandes sont les `FAutoConsoleCommand` statics ci-dessus.

---

### 4.10 Quetes procedurales

**Fichiers :** `Data/ProceduralQuestTypes.h`, `Data/ProceduralQuestTemplate.h/.cpp` (193 / 1271 l.), `Data/MultiStageQuestTemplate.h/.cpp` (121 / 2697 l.), `Data/QuestArchetypeTypes.h` (375 l.), `Data/InteractableMeshDatabase.h/.cpp`

#### Architecture

```
Monde (WorldScape + QLevel)
        |
        v
Decouverte de POIs (FDiscoveredPOI)
  EPOIType : SanglineNest, EnemySpawner, MineralDeposit,
             Cave, Bunker, Outpost, Generic
        |
        v
Selection de template (par type POI + faction)
        |
        +-- USanglineExterminationTemplate
        +-- USpawnerClearanceTemplate
        +-- UResourceCollectionTemplate
        +-- UAreaExplorationTemplate
        +-- UFactionEliminationTemplate
        +-- UBunkerRaidTemplate
        +-- UCaveExplorationTemplate
        +-- UOutpostCaptureTemplate
        +-- UMultiStageQuestTemplate (archetypes multi-etapes)
        |
        v
GenerateQuest(POI, Difficulty, OutQuest)
        |
        v
FQuestDefinition (objectifs + actions generes)
        |
        v
Assignation via QuestManagerSubsystem
```

`UProceduralQuestTemplate` est `Abstract, EditInlineNew, DefaultToInstanced`. Parametres de base : `TemplateID`, `TitlePattern`/`DescriptionPattern`, `RequiredPOITypes`, `ValidFactions`, `BaseCredits` (500), `CreditsPerDifficulty` (100), `CooldownHours` (1), `MinDifficulty`/`MaxDifficulty` (1-10). Points d'extension : `CanGenerateForPOI` et `GenerateQuest` (BlueprintNativeEvent), plus helpers `CalculateRewards`, `GenerateQuestID`, `FormatQuestText`, `ConfigureBaseQuestDefinition`.

`FProceduralQuestCache` met en cache les quetes generees par localisation/rayon (`CachedQuests`, `CacheTime`, `QueryLocation`, `QueryRadius`, expiration par defaut 60 s).

#### Archetypes multi-etapes

```cpp
UMultiStageQuestTemplate (base)
    +-- UInfiltrationArchetype
    +-- UExterminationArchetype
    +-- UMiningArchetype
    +-- UInvestigationArchetype
    +-- UDefenseArchetype
```

Les 5 sous-classes ne fixent que leurs valeurs par defaut dans le constructeur ; toute la logique de generation est dans la base : `SelectArchetype` -> `GenerateStages` -> `CreateObjectiveForStage`/`CreateActionsForStage`, avec scaling (`BaseStageCount`=3, `StagesPerDifficultyLevel`, `MaxTotalStages`=8) et injection de complications (`ComplicationChancePerDifficulty`). Les etapes sont decalees spatialement (`GenerateOffsetLocation`, `LocationOffsetRadius`).

**Enumerations d'archetype** (`QuestArchetypeTypes.h`) :

```cpp
enum class EQuestStageType : uint8 {
    Approach, Infiltrate, Combat, Investigate, Retrieve, Interact,
    Defend, Escape, Deliver, Scan, Sabotage, Ambush, Boss, Mine
};

enum class EQuestArchetypeID : uint8 {
    None, Infiltration, Extermination, Investigation, Mining,
    Rescue, Sabotage, Defense, Assassination, Delivery, Exploration
};

enum class EStageObjectiveType : uint8 {
    Location, Kill, KillStealth, KillTimed, Interact, Scan, Pickup,
    Dialogue, Defend, Timer, EGState, QLevelLoad, MineOre
};

enum class EStageComplication : uint8 {
    None, Reinforcements, TimedExplosives, Alarm, EnvironmentalHazard,
    BossSpawn, Ambush, EquipmentFailure, HostageExecution, EscapeVehicle
};
```

> **Note** : `EQuestArchetypeID` declare 11 archetypes mais seules 5 sous-classes `U*Archetype` existent ; Rescue/Sabotage/Assassination/Delivery/Exploration n'ont pas de sous-classe dediee.

#### Base d'interactables (placement procedural d'objectifs Interact/Scan)

`UInteractableMeshDatabase` (UDataAsset) mappe des noms de mesh vers une categorie via ~169 mots-cles codes en dur (`LoadDefaultKeywords`), avec un cache statique `CachedDatabase`. Categories :

```cpp
enum class EInteractableCategory : uint8 {
    Terminal, Container, Machinery, Furniture, Switch, Generic
};
```

Le scan d'un QLevel produit un `FQLevelInteractableData` (liste de `FDiscoveredInteractableMesh` + comptes par categorie) que les archetypes interrogent (`FindMeshNearLocation`, `GetRandomMeshOfCategory`).

> **Piege** : un mot-cle contient la faute de frappe historique `"StorageContianer"` (categorie Container, priorite 2.3). Devrait etre externalise en config.

---

## 5. Architecture reseau

### 5.1 Autorite

| Donnee | Autorite | Replication |
|---|---|---|
| Definitions de quetes | Serveur (QuestManagerSubsystem) | Non repliquees (chargees localement) |
| Etat runtime (progression) | Serveur (QuestManagerSubsystem) | Via PlayerQuestComponent |
| PlayerID | Serveur | `DOREPLIFETIME` |
| Validite du composant | Serveur | `DOREPLIFETIME_CONDITION_NOTIFY` -> `OnRep_ComponentValidity` |
| Quetes actives | Serveur | `DOREPLIFETIME_CONDITION_NOTIFY(COND_None, REPNOTIFY_Always)` -> `OnRep_ActiveQuests` |
| Quetes completees | Serveur | idem -> `OnRep_CompletedQuests` |
| Quetes echouees | Serveur | idem -> `OnRep_FailedQuests` |
| Suivi de quete | Serveur | idem -> `OnRep_TrackedQuestInfo` |
| Actions (ciblage) | Serveur -> QuestActionBridge | `IsNetRelevantFor` -> joueurs cibles |
| Etat par-quete (alternatif) | Serveur -> QuestInstanceActor | Acteur replique par quete (voie parallele) |

### 5.2 QuestActionBridge

**Fichiers :** `Actors/QuestActionBridge.h` (180) / `.cpp` (431)

Acteur replique ephemere cree par `QuestRelevancyManager::CreateActionBridge` (lui-meme invoque par `UQuestActionBase::CreateActionBridge`) pour router une action vers les joueurs concernes. `IsNetRelevantFor` ne renvoie `true` que si le viewer fait partie de `TargetPlayers`. Auto-destruction par `LifetimeTimer` (`BridgeLifetime`, 30 s par defaut) -> `OnLifetimeExpired` -> `DestroyBridge`. RPC propres : `Multicast_ExecuteAction`, `Server_ConfirmActionExecution` (WithValidation), `Client_ExecuteAction`. `ActionData` repliquee via `OnRep_ActionData`.

```cpp
USTRUCT(BlueprintType)
struct FQuestActionNetworkData {
    FName ActionID;                  // = NAME_None
    FName QuestID;
    FName ObjectiveID;
    FName InstigatingPlayerID;
    FString ActionClassPath;         // chemin de classe (pas un TSubclassOf)
    TArray<uint8> SerializedActionData;
    TArray<FString> ParameterKeys;   // parametres = 2 tableaux paralleles
    TArray<FString> ParameterValues; //   (decompresses via GetParametersAsMap())
    uint8 ActionType = 0;
};

USTRUCT(BlueprintType)
struct FQuestActionRelevancyContext {
    FName QuestID; FName ActionID; FName InstigatingPlayerID;
    TWeakObjectPtr<APlayerController> InstigatingController;
    float MaxRelevancyDistance = 0.0f;
    bool  bRequireQuestParticipation = true;
    bool  bIncludeSpectators = false;
};
```

### 5.3 QuestRelevancyManager (broadcast effectif)

`UGameInstanceSubsystem`. Regles `EQuestRelevancyRule { AllPlayers, QuestParticipants, ProximityBased, TeamBased, Custom }` (defaut = `QuestParticipants`). Cache de relevance (`CacheLifetime` 5 s, `MaxCacheEntries` 100). **Limite majeure** : `IsPlayerQuestParticipant` retourne `true` des que le `QuestManagerSubsystem` existe et `IsPlayerOnSameTeam` retourne `true` des que les deux joueurs ont un `PlayerState` ; la resolution de PlayerID passe par `PlayerState->GetPlayerName()` (pas de Steam / `UniqueNetId`). Seule `ProximityBased` (distance pawn-a-pawn) filtre reellement. En pratique, la relevance par quete/equipe est donc un **broadcast a tous**.

### 5.4 QuestInstanceActor (voie de replication parallele)

**Fichiers :** `Actors/QuestInstanceActor.h` (153) / `.cpp` (521)

Acteur replique **par quete** (`ReplicatedQuestID`, `QuestTitle`/`QuestDescription`, `CurrentState` + `OnRep_QuestState`, `ObjectiveProgressArray` + `OnRep`, `ObjectiveTimeArray`, `bIsTracked`, timestamps `int64`). `IsRelevantToLocalPlayer()` filtre l'envoi via `OnRep_PlayerID`. Reference par `QuestManagerSubsystem`. C'est une voie de replication **distincte** de `PlayerQuestComponent` (etat porte par l'acteur, pas par le composant du PC).

## 6. Structures de donnees de reference

### 6.1 FQuestDefinition

`QuestDataTypes.h:919`. Definition statique d'une quete (editable, modulaire, serialisable). Champs principaux (ordre du code) :

```cpp
struct FQuestDefinition {
    FName  QuestID;
    FText  Title;
    FText  Description;
    TArray<FString> AssociatedNPCs;
    int32  LevelRequirement = 1;
    TArray<FName> PrerequisiteQuestIDs;
    bool   bIsRecurring = false;
    float  RecurringCooldownHours = 24.0f;
    TArray<FName> QuestTags;              // FName, pas FGameplayTagContainer
    FVector QuestStartLocation;
    FText  LocationName;
    FText  Level;
    EQuestDifficulty Difficulty = EQuestDifficulty::Normal;
    int32  EstimatedTimeMinutes = 15;
    bool   bIsMainQuest = false;          // auto-assign/track + gate de niveau
    bool   bIsTimeLimited = false;
    FDateTime AvailabilityStartTime / AvailabilityEndTime;
    bool   bAutoCompleteIfNoObjectives = true;
    bool   bReAcceptOnFailure = false;
    bool   bHideInLogOnCompletion = false;
    FString CustomData;
    FText  AcceptDialog / CompleteDialog;
    FString AcceptSoundPath / CompleteSoundPath;
    bool   bIsStoryQuest = false;
    bool   bAutoTrack = false;
    bool   bPersistProgress = true;       // false => progression non sauvegardee

    TObjectPtr<UQuestComponentManager> ComponentManager; // Transient
    bool   bIsTemporaryCopy = false;

    // Composants modulaires (Instanced)
    TArray<TObjectPtr<UQuestObjectiveBase>> ModularObjectives;
    TArray<TObjectPtr<UQuestActionBase>>    QuestActions;
    TArray<TObjectPtr<UQuestConditionBase>> AcceptanceConditions;

    // Serialisation (SaveGame)
    TArray<FModularComponentData> SerializedObjectives;
    TArray<FModularComponentData> SerializedActions;
    TArray<FQuestConditionData>   SerializedConditions;
    bool   bHasModularData = false;

    TArray<FTriggerZoneData>           TriggerZones;
    TArray<FQuestFlowConnectionData>   FlowConnections;
};
```

> Pas de `FailureConditions`, pas de `FQuestRewardInfo Rewards`, pas de `AssignedNPCID` : les recompenses sont gerees par des actions modulaires (ex. `A_Rewards`), le NPC par `AssociatedNPCs`. Le constructeur de copie force `bIsTemporaryCopy=true` et `ComponentManager=nullptr` pour eviter les nettoyages destructeurs sur les copies temporaires (voir 7.1 #1).

### 6.2 FRuntimeQuestInstance

`QuestDataTypes.h:2475`. Etat runtime d'une quete pour un joueur (replique cote `PlayerQuestComponent`). Le `PlayerID` n'est PAS dans la struct (la propriete est portee par le conteneur).

```cpp
struct FRuntimeQuestInstance {
    FName  QuestID = NAME_None;
    EQuestStateType CurrentState;        // 'CurrentState', pas 'State'
    int64  StartTime / LastCompletionTime / FailTime / CompletionDuration;
    int32  AttemptCount / CompletionCount;
    FString CustomRuntimeData;           // SaveGame

    // Maps transientes (autorite)
    TMap<FName, int32>   ObjectiveProgress;
    TMap<FName, float>   ObjectiveTimeRemaining;
    TMap<FName, TObjectPtr<UQuestObjectiveBase>> ObjectiveInstances;
    TMap<FName, TObjectPtr<UQuestActionBase>>    ActionInstances;

    // Miroirs serialisables / replicables
    TArray<FObjectiveProgressPair> ObjectiveProgressArray;
    TArray<FObjectiveTimePair>     ObjectiveTimeArray;
    TArray<FActionStatePair>       ActionStateArray;

    TArray<UQuestObjectiveBase*>   ActiveObjectives;   // objets, transient
    TArray<FName>  ActiveObjectiveIDs;                 // SaveGame
    TArray<FName>  FailedObjectiveIDs;                 // SaveGame
    bool   bIsTracked;
    TArray<FTriggerZoneRuntimeState> TriggerZoneStates;
};
```

`UpdateArraysFromMaps()` reconstruit les tableaux `*Array` depuis les maps avant replication, en elaguant les instances d'action invalides (gardes `IsValid`/`reinterpret_cast` defensifs).

### 6.3 Enumerations principales

```cpp
enum class EQuestStateType : uint8 {
    NotStarted, InProgress, Completed, Failed, MAX /*Hidden*/
};

// Classification primaire/optionnelle d'un objectif (PAS une liste de types)
enum class EObjectiveType : uint8 {
    Primary, Optional
};

enum class EQuestDifficulty : uint8 {
    Easy, Normal, Hard, Expert, Elite
};

enum class EQuestFlowNodeType : uint8 {
    None, QuestStart, QuestComplete, QuestFail, Objective,
    Action, Condition, Prerequisite, Flow
};

enum class EQuestFlowConnectionType : uint8 {
    None, Prerequisite, LinkedAction, ActivationCondition,
    FailureCondition, Sequence, Optional,
    ActionSuccess, ActionFailure, ActionRetry
};
```

> Les types concrets d'objectif (Kill, Location, Scanner, EGState, etc.) ne sont PAS un enum : ce sont des sous-classes de `UQuestObjectiveBase` resolues par nom via `UQuestObjectiveFactory`/`UQuestAssetManager`.

### 6.4 Famille FQuestNotificationPayload

Definie dans `Notifications/QuestNotificationSettings.h` (et non dans `QuestDataTypes.h`). Reecrite autour du plugin **QNotification** : un payload est une sequence d'entrees, chacune configurant une notification QNotification.

```cpp
struct FQuestNotificationPayloadBase {
    EQNotificationType     NotificationType;  // Toast / Progress / Warning / Communication
    EQNotificationPriority Priority;
    FText Title; FText Message; FText SubMessage;
    float DisplayDuration;
    TObjectPtr<UTexture2D> Icon; TObjectPtr<USoundBase> Sound;
    bool  bAutoHide; bool bCanBeDismissedByClick;
    // Toast    : FQToastNotificationOptions ToastOptions
    // Progress : CurrentValue/MaxValue, bAutoProgress, bUseObjectiveProgressAsValues, ...
    // Warning  : FQWarningNotificationOptions, bUseInputFeedbackForWarning, InputFeedbackPreset, ...
    // Communic.: Portrait, Dialogue, DialogueAudio, TypewriterSpeed + sync audio, ...
    bool  bBlockObjectiveCompletionWhileActive;  // blocage par-entree
    bool  bBlockActionCompletionWhileActive;
};

struct FQuestNotificationSequenceEntry {
    float DelayBefore;
    float DelayAfter;
    bool  bWaitForCompletion;
    FQuestNotificationPayloadBase Payload;
};

struct FQuestNotificationPayload {
    TArray<FQuestNotificationSequenceEntry> Entries;
    float LastEntryHoldDuration = 0.0f;
};
```

> Le blocage de completion (anciennement `bIsBlocking`/`BlockingSourceID`) est desormais porte par chaque entree via `Payload.bBlockObjectiveCompletionWhileActive` / `bBlockActionCompletionWhileActive`.

### 6.5 Registre dynamique (UQuestDynamicRegistry)

Conteneur cle->valeur type pour les quetes procedureaux/scriptees. Types supportes : String, Int, Float, Bool, Vector, Rotator, Transform, ObjectReference, ActorReference, ClassReference, SoftAssetReference, plus collections d'objets/acteurs. Chargement async (`LoadAssetAsync`, `FindLevelAndStoreAsync`, `FindClassAndStoreAsync`), `ComputeAndStoreTransform`, `FindByPropertyValue`. Accessible via `UQuestDynamicRegistrySubsystem` (GameInstanceSubsystem).

> **Piege (cf. 7.2)** : `ClearWithPrefix` ne parcourt que `StringValues` pour collecter les cles ; les cles presentes uniquement dans une autre map typee (Int, Vector, ObjectReference, ...) ne sont jamais purgees (`QuestDynamicRegistry.cpp:430`).

## 7. Points d'attention

### 7.1 Subtilites -- Ce qu'un developpeur doit absolument savoir

1. **AddToRoot / RemoveFromRoot** : les objectifs, actions et conditions instancies sont maintenus hors-GC par `QuestComponentManager`. Tout oubli de `RemoveFromRoot` cree un leak. Les copies temporaires de `FQuestDefinition` forcent `bIsTemporaryCopy=true` + `ComponentManager=nullptr` pour eviter les nettoyages destructeurs.

2. **Transition de monde** : sur `OnPostWorldInitialization`, l'etat en memoire est **integralement detruit et recharge** depuis le disque via un timer de 0.5 s (`ReloadQuestProgressAfterWorldTransition`). Ne jamais stocker de references durables vers des objets quete entre les niveaux.

3. **Reconstruction differee a la reconnexion** : au chargement de progression (`DeserializePlayerProgressFromArchive`), les quetes `IsBinarySerialized()` sont reconstruites (`ReconstructModularComponentsFromSerialized`) PUIS leurs instances runtime recreees (`InitializeAction/ObjectiveInstancesFromDefinition`) avant reprise/scan d'activation. C'est ce qui restaure objectifs et actions live quand l'etat logique survit mais que les UObject transients ont ete vides.

4. **Auto-assign de quete principale decouple de la synchro d'ID** : `AutoAssignStoryQuestIfNeeded()` existe toujours mais n'est **plus** appele lors de la resolution d'ID joueur (commentaires explicites `intentionally NOT called here`). Il ne se declenche que sur rechargement post-transition et via `PlayerQuestComponent` a la connexion. Ne pas re-cabler l'auto-assign sur la resolution d'ID (regression connue : quete qui pop sans acceptation au login/rejoin).

5. **Resolution de PlayerID deterministe + deferral** : `GetOrCreatePersistentPlayerID()` utilise l'`UniqueNetId` OSS (`PlayerState->GetUniqueId`, puis `LocalPlayer->GetPreferredUniqueNetId`) et retourne `NAME_None` tant qu'il n'est pas disponible -- les appelants reessayent. `UserName()` n'est qu'un fallback packaged/editeur. Les IDs `"0"` (`LegacyZeroPlayerID`) et `"STEAM_ID_FAILED"` sont rejetes. (Attention : `QuestRelevancyManager`, lui, utilise encore `GetPlayerName()`.)

6. **Deux systemes d'evenements** : `UQuestEventManager` (UObject, handlers prioritises via `IQuestEventHandler`) et `FQuestEventManager` (singleton statique, 5 delegates multicast) coexistent avec des roles differents.

7. **Double factory** : `UQuestObjectiveFactory` et `UQuestAssetManager` font essentiellement la meme chose (decouverte/enregistrement de classes). Utiliser `QuestObjectiveFactory`.

8. **Reflexion intensive** : `ScannerObjective`, `EGStateObjective`, `NPCDialogueObjective` (et cote actions `TriggerActorAction`, `QuestActionEGControl`) utilisent massivement `ProcessEvent`/`FindFunction`/`FindFProperty` pour eviter des dependances compile-time (inventaire, scanner, `OptimizedStateComponent`). Noms centralises (ex. `ScannerObjectiveNames`). Chemins fragiles face aux renommages de proprietes/fonctions.

9. **Serialisation binaire defensive** : `FModularQuestArchiveProxy` avale silencieusement les references UObject manquantes (assets deplaces/supprimes) -- evite les crashs mais peut causer des pertes silencieuses. Il duplique `FQuestSilentObjectProxyArchive` de `SerializeQuestUtils.h`.

10. **Triple RPC asymetrique** : `MarkActionCompleted` envoie `Client_SynchronizeActionState` via **3 `if` independants** (PC, PlayerState, Pawn) -> jusqu'a 3 envois par joueur. `MarkActionFailed`, lui, est en `if/else-if` -> un seul envoi. Le defaut ne touche QUE la voie "completed".

11. **Execution d'action double voie** : `bUseAttemptExecutionMethod` choisit entre `OnExecuteAction` (void, succes presume) et `AttemptActionExecution` (bool + raison, retries). `bManualCompletionOnly` differe l'auto-completion pour les actions async (sequences/timers) qui appellent `MarkActionCompleted` elles-memes.

12. **Optimisation de replication par signature** : `OnRep_ActiveQuests` compare des CRC (`ComputeQuestReplicationSignature`) et saute la reconstruction si rien n'a change (fix supprimant ~360 reconstructions/session). `FActiveLocationObjectiveInfo` a ete reduit a un simple `TSet<int32>`.

13. **Couplage input par reflexion** : le blocage des inputs gameplay pendant dialogues/contract machines passe par `BroadcastInputModeChanged` (`QuestInputModeHelper.h`) qui retrouve par reflection le `WidgetGlobalComponent` du jeu hote (delegate `OnInputModeChanged`, propriete `IsFocusedInUI`) -- couplage implicite non type, silencieux si absent.

14. **Commandes console** : les 6 `dqs.*` sont des `FAutoConsoleCommand` statiques (dans `QuestDebugSubsystem.cpp`), PAS `FQuestDebugCommands` (dont les `Register/UnregisterConsoleCommand` sont des stubs vides).

### 7.2 Limitations connues

1. **QuestRelevancyManager** : `IsPlayerQuestParticipant` retourne `true` des que le subsystem existe, `IsPlayerOnSameTeam` retourne `true` des que les deux ont un `PlayerState`, et la resolution PlayerID passe par `GetPlayerName()` (pas de Steam/`UniqueNetId`). Seule `ProximityBased` filtre. La relevance par quete/equipe est donc un **broadcast a tous**.

2. **`RealTimeReferenceValidation`** : `SetupRealTimeReferenceValidation()` et `RealTimeReferenceValidation()` sont des **stubs vides** ; le `RealTimeValidationTimerHandle` n'est jamais demarre. (Le nettoyage 60 s `PeriodicBlueprintReferenceCleanup` reste quasi no-op.)

3. **LevelMeshScanner::PerformAsyncScan** *(corrige -- dd8b0e3b)* : la passe UObject (iteration d'acteurs, `GetComponents`, acces mesh, `NewObject`) est differee sur le game thread via `AsyncTask(ENamedThreads::GameThread, ...)`. Auparavant elle tournait sur un thread pool et derefencait des acteurs liberes pendant le chargement de niveau (crash use-after-free).

4. **InteractionComponent** : `bCanEverTick = true` en permanence alors que le tick ne fait du travail QUE si `InteractionTraceInterval <= 0` (chemin de repli) -- cout de tick gaspille avec l'intervalle par defaut. La distance d'interface (`GetInteractionDistance()`, 200 cm sur l'acteur) n'est jamais interrogee : le composant utilise sa propre `DefaultInteractionDistance` (250 cm).

5. **`ModularInteractionPromptWidget`** : `NativeConstruct()` remet `CurrentInteractableActor` a null, ce qui race avec `SetInteractableActor` appele AVANT `AddToViewport`. Le prompt peut ne jamais afficher de donnees.

6. **`UInteractionSubsystem::SetInteractablesEnabled`** : n'agit que sur les sous-classes de `AInteractableActor` (`Cast`) ; les acteurs qui se contentent d'implementer l'interface (ou hebergent un composant interface) sont ignores.

7. **`ClearWithPrefix` (QuestDynamicRegistry)** : ne decouvre les cles que dans `StringValues`, rate les cles presentes uniquement dans une autre map typee (Int, Float, Vector, ObjectReference, ...).

8. **QuestFlowGraph** : pas de sauvegarde (`SaveToQuestDefinition()` vide), pas d'auto-layout reel (`OnAutoArrange` fait `OnZoomToFit`), pas de validation reelle (`OnValidateGraph` fait `OnRefreshGraph`), routage anti-collision declare mais non implemente.

9. **`CanPasteComponent` (SCategorizedComponentEditor)** : deserialise le JSON du clipboard a chaque appel (cable au `IsEnabled()` du bouton Paste -> evalue a chaque tick Slate) -- cout de performance.

10. **Logging de conditions** : `CheckCondition_Implementation` emet toujours jusqu'a 5 `DQS_LOG_WARNING` par evaluation (`LevelSequenceCondition` davantage). Non convertis en `Verbose` ; seulement neutralises au runtime par le garde global `GDQSLoggingEnabled` (off par defaut depuis b7d5ead7).

11. **Pas de spawn d'item cote actions** : le systeme d'actions ne fait que la **consommation** d'inventaire (`RemoveRegisteredInventoryItems`, par reflexion). Aucun helper d'octroi/spawn d'item.

12. **Audio dialogue non synchronise** : le VO du popup de dialogue n'est PAS synchronise au typewriter ; la synchro adaptative texte<->audio n'existe que pour les notifications de type `Communication`.

13. **Archetypes incomplets** : `EQuestArchetypeID` declare 11 archetypes mais seules 5 sous-classes `U*Archetype` existent (Rescue/Sabotage/Assassination/Delivery/Exploration n'ont pas de sous-classe).

14. **`EditorScriptingUtilities`** : active dans le `.uplugin` mais reference par aucun `Build.cs` (dependance d'activation morte cote C++).

### 7.3 Hacks et dette technique

| Fichier (emplacement actuel) | Description | Raison probable |
|---|---|---|
| `BinaryQuestDataSource.cpp:415-426` | Logging de debug hardcode pour la quete `"Q001"` (dans `SaveQuestsToDirectory`) | Artefact de developpement |
| `QuestManagerSubsystem.cpp` | Detection lobby par substring `"L_Lobby"` dans le nom de monde | Pas d'enum/tag pour les types de monde |
| `QuestManagerSubsystem.cpp` | Timer de 0.5 s pour recharger apres transition de monde | Race condition avec l'initialisation du monde |
| `QuestManagerSubsystem.cpp:1028-1036` | `SetupRealTimeReferenceValidation` / `RealTimeReferenceValidation` reduits a des corps vides ; handle conserve, jamais demarre | Validation temps-reel abandonnee |
| `QuestConditionBase.cpp:107-127` | Jusqu'a 5 `DQS_LOG_WARNING` par evaluation de condition (masques par `GDQSLoggingEnabled`) | Logging de debug oublie |
| `QuestActionBase.cpp:1352-1377` | `MarkActionCompleted` : 3 `if` independants (PC/PlayerState/Pawn) -> RPC potentiellement triple | Iteration de composants non factorisee |
| `InteractableMeshDatabase.cpp:54` | ~169 mots-cles codes en dur, avec la faute `"StorageContianer"` | Base initiale, devrait etre en config |
| `ProceduralQuestTemplate.cpp:92-168` | `static UClass* CachedClass = nullptr` dans les helpers -- persiste entre sessions PIE | Optimisation de performance |
| `QuestDataTypes.h:359` | `FModularQuestArchiveProxy` duplique `FQuestSilentObjectProxyArchive` (`SerializeQuestUtils.h:14`) | Evolution independante des deux fichiers |
| `QuestEditorWindow.h:279-280` | `UPROPERTY()` sur `QuestIconTexture` dans un `SCompoundWidget` (non-UObject) | Macro sans effet, jamais nettoye |
| `ComponentOrganizationManager.h:138` | Commentaire `//BenFix ma guele` (a migre du `.cpp` vers le header) | Commentaire de dev oublie |
| `QuestFlowConnectionDrawingPolicy.h:29-31` | `DoesLineIntersectNode` et `FindPathAroundObstacles` declarees mais jamais definies | Routage anti-collision planifie non implemente |
| `SQuestFlowPin.cpp:138` | `bIsDragging` set sur `OnMouseButtonDown` mais jamais remis a false (aucun `OnMouseButtonUp`) | Reset manquant |
| `QuestDebugCommands.cpp` | `RegisterConsoleCommand()` / `UnregisterConsoleCommand()` stubs vides ; `TCommands` vestigial | Les vraies commandes sont des `FAutoConsoleCommand` statics |
| `QuestAssetManager.h` | Doublon fonctionnel de `QuestObjectiveFactory` (decouverte de classes) | Deux implementations paralleles |
| `DynamicQuestSystem.uplugin` | `EditorScriptingUtilities` active mais inutilise par tout module C++ | Dependance d'activation morte |

---

## 8. Arborescence complete des fichiers

### Module Core (`DynamicQuestSystem/`)

```
DynamicQuestSystem.Build.cs                          -- Config build, deps lourdes (WorldScape, QLevel, etc.)

Public/
  Core/
    DynamicQuestSystem.h                             -- Module, log category, macros DQS_LOG
    DynamicQuestSystemSettings.h                     -- UDeveloperSettings: 5 props + FPlayerFlagDefinition + BinaryVersion=4
    QuestAssetManager.h                              -- Decouverte classes (doublon de Factory)
  Data/
    QuestDataTypes.h                                 -- TOUTES les structs/enums centrales (~1400 lignes)
    QuestArchetypeTypes.h                            -- Types archetypes multi-etapes
    ProceduralQuestTypes.h                           -- POI types, cache procedural
    SerializeQuestUtils.h                            -- Archive proxy, FindClassWithFallbacks
    QuestDataAsset.h                                 -- UPrimaryDataAsset simple (legacy/alternatif)
    QuestDynamicRegistry.h                           -- Key-value store runtime type
    BinaryQuestDataSource.h                          -- Persistance binaire IQuestDataSource
    QuestSystemPrimaryAsset.h                        -- Catalogue refs assets pour cooking
    QuestStringTableRuntime.h                        -- Resolution textes localises
    InteractableMeshDatabase.h                       -- Classification meshes par mots-cles
    ProceduralQuestTemplate.h                        -- 8 templates proceduraux
    MultiStageQuestTemplate.h                        -- 5 archetypes multi-etapes
  Interfaces/
    IQuestDataSource.h                               -- Interface persistance
    IQuestEventHandler.h                             -- Interface traitement evenements
    IQuestProgress.h                                 -- Interface cycle de vie joueur (18 methodes)
    IQuestProvider.h                                 -- Interface catalogue quetes
    IQuestSpawnedMeshActor.h                         -- Interface mesh spawne
  Events/
    QuestEventManager.h                              -- UObject event manager avec priorite
    QuestEvents.h                                    -- Singleton statique delegates multicast
  Conditions/
    QuestConditionBase.h                             -- Base conditions (types, inversion, cache)
  Objectives/
    QuestObjectiveBase.h                             -- Base objectifs (~1500 lignes, garde activation)
    QuestObjectiveFactory.h                          -- Factory par nom de type
  Subsystems/
    QuestManagerSubsystem.h                          -- Orchestrateur central (~2000 lignes header)
    QuestManagerDelegates.h                          -- 8 delegates natifs
    PlayerQuestSubsystem.h                           -- Facade LocalPlayerSubsystem
    QuestRelevancyManager.h                          -- Ciblage actions reseau (partiellement stub)
  Components/
    PlayerQuestComponent.h                           -- Composant replique, RPCs, location check
    QuestComponentManager.h                          -- Factory/conteneur par quete, AddToRoot
    QuestEntityComponent.h                           -- Composant entite spawnee (AI, death notify)
    QuestInteractComponent.h                         -- Composant interaction simple
    DynamicInteractableComponent.h                   -- Composant interactif dynamique (IInteractableInterface)
  Actors/
    QuestInstanceActor.h                             -- Acteur replique par quete active
    QuestObjectiveActor.h                            -- Acteur monde (porte, conteneur, switch)
    QuestTriggerActor.h                              -- NPC donneur de quete + dialogue
    QuestActionBridge.h                              -- Pont reseau pour actions ciblees
    QuestInteractActorBase.h                         -- Base acteur interactif spawne
    TriggerZoneManager.h                             -- Gestionnaire UTriggerZoneComponent
  Actions/
    QuestActionBase.h                                -- Base actions (~1500 lignes, 5 modes reseau)
    ControlAmbientAudioAction.h                      -- Controle audio ambiant
  UI/
    QuestUIManager.h                                 -- Gestionnaire UI centralise
    QuestUIStyles.h                                  -- Constantes couleurs/polices/marges
    QuestUIAnimations.h                              -- Systeme animation par timers
    QuestTrackerWidget.h                             -- Tracker par quete
    QuestTrackerEntryWidget.h                        -- Entree objectif dans tracker
    QuestTrackerContainerWidget.h                    -- Conteneur de trackers
    QuestDialogueWidget.h                            -- Header dialogue (Accept/Decline/Skip)
    QuestDialogueChoiceWidget.h                      -- Choix individuel
    QuestDialogueDetailsWidget.h                     -- Details quete selectionnee
    QuestDialoguePopupWidget.h                       -- Popup dialogue complete (FSM)
    QuestDialogueTypes.h                             -- Structs dialogue (branches, conditions)
    QuestNPCMenuWidget.h                             -- Menu NPC alternatif (machine a contrats)
    QuestTextFormatter.h                             -- Formatage rich text + typewriter
    InteractionPromptWidget.h                        -- Widget prompt minimal (abstrait)
  Notifications/
    QuestNotificationSettings.h                      -- 7 structs de payload QNotification (PAS un UDeveloperSettings)
    QuestNotificationUtils.h                         -- Sequencage + blocage notifications
  Libraries/
    QuestSystemHelperFunctions.h                     -- 2 fonctions utilitaires Blueprint
  Utils/
    LevelMeshScanner.h                               -- Scan meshes interactifs (sync/async)
    QuestHelperFunctions.h                           -- Projection terrain, gravite WorldScape

Private/
  [Meme structure que Public/ avec les .cpp correspondants]
  UI/
    QuestInputModeHelper.h                           -- Helper statique broadcast input mode
```

### Module Objectives (`DynamicQuestSystemObjectives/`)

```
DynamicQuestSystemObjectives.Build.cs                -- Deps: Core, Interaction, QLevel, Niagara, etc.

Public/
  Core/
    DynamicQuestSystemObjectives.h                   -- Module declaration
    QuestModuleClassRegistrar.h                      -- Anti-GC, EnsureClassIsReferenced<T>
  Objectives/
    ConnectorObjective.h                             -- Puzzle cable/socket
    CustomObjective.h                                -- Extension Blueprint (Blueprintable)
    EGStateObjective.h                               -- Surveillance etat Environmental Gadget
    InteractObjective.h                              -- Interaction temporisee
    KillObjective.h                                  -- Kills filtre (faction, arme, furtif, etc.)
    LocationObjective.h                              -- Visite de lieux (le plus complexe)
    NPCDialogueObjective.h                           -- Dialogue NPC + items requis
    NotificationPlaybackObjective.h                  -- Attente notifications
    QLevelLoadObjective.h                            -- Chargement QLevel
    ScannerObjective.h                               -- Scanner (reflexion intensive)
    TriggerOnlyObjective.h                           -- Auto-completion (trigger actions)
  Actions/
    ActivateInteractableAction.h                     -- Spawn/active interactif
    ControlLevelSoundAction.h                        -- Controle audio niveau
    HighlightActorAction.h                           -- Outline/highlight
    OpenPersistentUniverseAction.h                   -- Transition niveau
    PlayLevelSequenceAction.h                        -- Cinematique (~30 proprietes)
    PlayNiagaraAction.h                              -- Effets particules
    PlaySoundAction.h                                -- Son 2D/spatial/attache
    QuestActionEGControl.h                           -- Controle etat EG
    QuestActionLightControl.h                        -- Controle lumieres
    QuestNotificationPlayAction.h                    -- Notification UI
    SetObjectiveStateAction.h                        -- Modifie etat objectifs
    SpawnActorAction.h                               -- Spawn entites
    TeleportAction.h                                 -- Teleportation joueur
    TriggerActorAction.h                             -- Appel fonction par reflexion
  Conditions/
    CheckPlayerFlagCondition.h                       -- Verifie flag joueur
    IsMultiplayerCondition.h                         -- Verifie mode multijoueur
    LevelSequenceCondition.h                         -- Verifie fin cinematique
  Actors/
    LocationObjectiveZoneActor.h                     -- Zone editeur/runtime (sphere/box/cylinder)

Private/
  [.cpp correspondants + DynamicQuestSystemObjectivesModule.cpp]
```

### Module Editor (`DynamicQuestSystemEditor/`)

```
DynamicQuestSystemEditor.Build.cs                    -- Deps: Core, Objectives, GraphEditor, etc.

Public/
  Core/
    DynamicQuestSystemEditor.h                       -- Module (nomad tab, factories)
    DynamicQuestSystemEditorSubsystem.h              -- Subsystem data layer (cache, locking, JSON)
    QuestEditorCommands.h                            -- 1 commande UI (sans raccourci)
  Settings/
    QuestFlowSettings.h                              -- UDeveloperSettings graphe visuel (~40 props de style)
  Style/
    QuestEditorStyle.h                               -- Style boutons + 16 icones
    QuestFlowStyle.h                                 -- Style graphe (nodes, pins, wires)
  Graph/
    QuestFlowGraph.h                                 -- UEdGraph (lecture seule)
    QuestFlowGraphNode.h                             -- Node (type, composant, etat)
    QuestFlowGraphSchema.h                           -- Regles connexion, menu contextuel
    QuestFlowGraphNodeFactory.h                      -- Factory SGraphNode
    QuestFlowPinFactory.h                            -- Factory SGraphPin
    QuestFlowGraphToolbar.h                          -- Toolbar (zoom, layout, validation stub)
    QuestFlowConnectionDrawingPolicy.h               -- Bezier cubique + fleches + glow
    SQuestFlowGraphWidget.h                          -- Host SGraphEditor
    SQuestFlowGraphNode.h                            -- Rendu node multi-couches
    SQuestFlowPin.h                                  -- Pin circulaire anime
    SQuestFlowNodeTooltip.h                          -- Tooltip riche
  Slate/
    SCategorizedComponentEditor.h                    -- Editeur template categories (actuel)
    SCategoryManagementWidget.h                      -- Admin categories
    SModularComponentEditor.h                        -- Editeur template simple (ancien)
    SModularQuestEditors.h                           -- Facades par type
    SQuestList.h                                     -- Arbre quetes/dossiers avec DnD
    SQuestSystemSettingsWidget.h                     -- Panneau reglages
    QuestActionDetailsCustomization.h                -- Customisation proprietes actions
    SQuestConditionDetailsCustomization.h            -- Customisation proprietes conditions (stub)
  Editors/
    QuestEditorWindow.h                              -- Fenetre principale 8 onglets
    DynamicQuestSystemModularEditors.h               -- Decouverte classes (static caches)
    ComponentOrganizationManager.h                   -- Gestion categories (singleton JSON)
  Data/
    EditorJsonQuestDataSource.h                      -- IQuestDataSource JSON
    QuestStringTableUtils.h                          -- Gestion StringTables localisees
  Widgets/
    SActorSelectionWidget.h                          -- Picker acteur
    SAssetSelectionWidget.h                          -- Picker asset content browser
    SClassSelectionWidget.h                          -- Picker UClass
    SEnumSelectionWidget.h                           -- Combo enum
    STagSelectionWidget.h                            -- Editeur tags tile-view

Private/
  [.cpp correspondants]
```

### Module Debug (`DynamicQuestSystemDebug/`)

```
DynamicQuestSystemDebug.Build.cs                     -- Deps legeres: Core seulement

Public/
  Core/DynamicQuestSystemDebug.h                     -- Module
  Subsystems/QuestDebugSubsystem.h                   -- Subsystem + 6 commandes console
  Commands/QuestDebugCommands.h                      -- TCommands (vestigial)
  Widgets/SDynamicQuestDebugWidget.h                 -- Overlay Slate 3 panneaux + registre

Private/
  Core/DynamicQuestSystemDebug.cpp                   -- Includes inutiles
  Subsystems/QuestDebugSubsystem.cpp                 -- Implementation commandes (~770 lignes)
  Commands/QuestDebugCommands.cpp                    -- Stubs vides
  Widgets/SDynamicQuestDebugWidget.cpp               -- Widget debug (~1360 lignes, le plus gros fichier)
```

### Module Interaction (`DynamicQuestSystemInteraction/`)

```
DynamicQuestSystemInteraction.Build.cs               -- Autonome, IWYU, deps minimales

Public/
  DynamicQuestSystemInteraction.h                    -- Module (no-op)
  Interfaces/InteractableInterface.h                 -- 9 methodes BlueprintNativeEvent
  Components/InteractionComponent.h                  -- Detection line trace
  Actors/InteractableActor.h                         -- Base abstraite
  Subsystems/InteractionSubsystem.h                  -- Registre global tags/spatial
  Blueprints/InteractionFunctionLibrary.h            -- Helpers Blueprint
  UI/ModularInteractionPromptWidget.h                -- Widget prompt base

Private/
  DynamicQuestSystemInteraction.cpp                  -- Module vide
  Components/InteractionComponent.cpp                -- Trace + focus + widget
  Actors/InteractableActor.cpp                       -- Registration auto
  Subsystems/InteractionSubsystem.cpp                -- Registre + recherche
  Blueprints/InteractionFunctionLibrary.cpp          -- Wrappers null-safe
  UI/ModularInteractionPromptWidget.cpp              -- Tick par frame (non throttle)
```

---
### 8.1 Deltas constates contre le code (2026-06)

Le tableau de fichiers de la section 8 reste fidele au code actuel ; corrections ponctuelles :

| Element | Doc | Reel |
|---|---|---|
| Interfaces -- `IQuestProgress.h` | "16 methodes" | **18 methodes** BlueprintNativeEvent |
| Module Interaction -- nombre de fichiers | "14" mentionne ailleurs | **13 fichiers source** (7 `.h` + 6 `.cpp`) + 1 `.Build.cs` |
| `Core/DynamicQuestSystemSettings.h` | "config plugin" | UDeveloperSettings, 5 props (DefaultDataSourceClass, bCreateBackups, bShowDebugInfo, bEnableQuestSystemLogging, RegisteredFlags) + `BinaryVersion = 4` + struct `FPlayerFlagDefinition` |
| `Notifications/QuestNotificationSettings.h` | "config notifications (~50 proprietes)" | **Pas un UDeveloperSettings** : 7 structs de payload (FQuestNotificationPayloadBase ~70 props avec bloc Audio Sync, FQuestNotificationSequenceEntry, FQuestNotificationPayload, *EventConfig, *ProgressConfig, FQuestObjectiveNotificationSettings, FQuestActionNotificationSettings) |
| `Settings/QuestFlowSettings.h` (Editor) | "UDeveloperSettings graphe visuel" | ~40 props de style (Node/Wire Colors, Node States overlays, Grid, Pin/Node Appearance, Animation) + GetNodeColorForType/GetWireColorForType |

Aucun fichier present dans l'arborescence documentee n'a disparu, et aucun nouveau fichier `.h`/`.cpp` hors arborescence n'a ete detecte dans les modules Core/Objectives/Editor/Debug/Interaction (compte verifie : Core 62h/48cpp, Objectives 31h/30cpp, Editor 35h/35cpp, Debug 4h/4cpp, Interaction 7h/6cpp).

