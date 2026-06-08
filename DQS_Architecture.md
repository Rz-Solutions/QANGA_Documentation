# DynamicQuestSystem -- Documentation Architecture Exhaustive

## 1. Vue d'ensemble

Le **DynamicQuestSystem** est un framework de quetes modulaire pour UE5, concu pour un jeu multijoueur (client/serveur dedie) avec un monde procedural (WorldScape). Il gere le cycle de vie complet des quetes : definition, assignation, progression, completion, persistance binaire, et interface utilisateur.

Le plugin est structure en **5 modules** avec une separation nette entre runtime, edition, debug, interaction et objectifs concrets. L'architecture est **server-authoritative** : toute mutation d'etat passe par le serveur via des RPCs, puis est repliquee aux clients via des proprietes repliquees sur `PlayerQuestComponent`.

Le systeme supporte a la fois des quetes designees manuellement (editeur JSON + graphe visuel) et des quetes **procedurales** generees a partir de Points d'Interet (POI) decouverts dans le monde, avec des templates d'archetypes multi-etapes.

---

## 2. Architecture reelle

### 2.1 Modules et dependances

```
DynamicQuestSystemInteraction     <-- Module autonome, aucune dep. quest
        ^
        | (public dep)
DynamicQuestSystem (Core)         <-- Orchestration, subsystems, donnees, UI
        ^                ^
        | (private)      | (public)
        |                |
DynamicQuestSystemDebug  DynamicQuestSystemObjectives
                                  ^
                                  | (public)
                         DynamicQuestSystemEditor
```

### 2.2 Tableau des sous-systemes

| Sous-systeme | Module | Statut | Description |
|---|---|---|---|
| QuestManagerSubsystem | Core | **Complet** | Orchestrateur central (GameInstanceSubsystem). ~2000 lignes. |
| PlayerQuestSubsystem | Core | **Complet** | Facade par joueur local (LocalPlayerSubsystem) |
| PlayerQuestComponent | Core | **Complet** | Composant replique sur PlayerController, RPCs client/serveur |
| QuestRelevancyManager | Core | **Partiel** | Ciblage d'actions reseau. `IsPlayerQuestParticipant` et `IsPlayerOnSameTeam` sont des stubs (toujours `true`). |
| QuestComponentManager | Core | **Complet** | Factory/conteneur d'objets modulaires par quete |
| BinaryQuestDataSource | Core | **Complet** | Persistance binaire versionnee (v4) avec backup |
| QuestDynamicRegistry | Core | **Complet** | Key-value store runtime type (string, int, float, bool, vector, etc.) |
| QuestEventManager | Core | **Complet** | Broadcast evenementiel thread-safe avec priorite |
| QuestObjectiveFactory | Core | **Complet** | Factory d'objectifs/actions/conditions par nom de type |
| QuestAssetManager | Core | **Doublon** | Decouverte de classes -- chevauche QuestObjectiveFactory |
| QuestUIManager | Core | **Complet** | Gestion UI centralisee (tracker, notifications) |
| QuestNotificationUtils | Core | **Complet** | Sequencage de notifications avec blocage |
| ProceduralQuestTemplate | Core | **Complet** | 8 templates de quetes procedurales |
| MultiStageQuestTemplate | Core | **Complet** | 5 archetypes multi-etapes |
| QuestStringTableRuntime | Core | **Complet** | Resolution de textes localises via StringTables |
| InteractableMeshDatabase | Core | **Complet** | Classification de meshes par mots-cles (~200 entrees) |
| LevelMeshScanner | Core | **Complet** | Scan de niveaux. `PerformAsyncScan` differe la passe UObject sur le game thread. |
| QuestEditorWindow | Editor | **Complet** | Editeur principal, 8 onglets |
| QuestFlowGraph | Editor | **Lecture seule** | Graphe visuel. `SaveToQuestDefinition()` est vide. |
| EditorJsonQuestDataSource | Editor | **Complet** | Persistance JSON editeur avec migration binaire legacy |
| QuestDebugSubsystem | Debug | **Complet** | 6 commandes console + overlay Slate temps reel |
| InteractionSubsystem | Interaction | **Complet** | Registre global d'acteurs interactifs |
| InteractionComponent | Interaction | **Complet** | Detection par line trace avec prompt UI |
| QuestModuleClassRegistrar | Objectives | **Complet** | Enregistrement anti-GC de toutes les classes du module |

### 2.3 Dependances externes

| Plugin | Usage |
|---|---|
| QNotification | Notifications UI (toast, warning, progress, communication) |
| QLevel | Streaming de niveaux, assets POI pour quetes procedurales |
| QTriggerZone | Zones de declenchement spatiales |
| QAmbientAudio | Controle audio ambiant via actions de quete |
| QCableConnector | Composant cable pour ConnectorObjective |
| WorldScape | Monde procedural (gravite, terrain, POIs) |
| EssentialMacros | Macros utilitaires |
| InputSystem | Systeme d'input custom |
| EnhancedInput | Input UE5 pour interactions |
| CommonUI | Framework UI |
| Niagara | Effets visuels (PlayNiagaraAction) |
| LevelSequence/MovieScene | Cinematiques (PlayLevelSequenceAction) |

---

## 3. Pipeline d'execution

### 3.1 Cycle de vie d'une quete

```
                          EDITEUR                          RUNTIME
                    +------------------+
                    | JSON sur disque  |
                    | (EditorQuests/)  |
                    +--------+---------+
                             |
                    Export binaire (RuntimeQuests.bin)
                             |
                             v
              +-----------------------------+
              | QuestManagerSubsystem       |
              | Initialize()                |
              |   LoadQuestsFromBinaryAsync |------> FAsyncLoadQuestsFromBinaryTask
              |   -> QuestDataMap populated  |            (thread pool)
              +-----------------------------+            |
                             |                    callback game thread
                             v                           |
              QuestDataMap : TMap<FName, FQuestDefinition>
                             |
          +------------------+------------------+
          |                                     |
    AssignQuestToPlayer()             GenerateProceduralQuest()
          |                                     |
          v                                     v
    FRuntimeQuestInstance cree         ProceduralQuestTemplate
    ObjectiveInstances instancies      -> FQuestDefinition generee
    ActionInstances instancies         -> AssignQuestToPlayer()
          |
          v
    PlayerQuestComponent (serveur)
    -> Replication ActiveQuests[]
    -> Client: OnRep_ActiveQuests()
          |
          v
    Objectifs actives (timer/event)
    ProgressObjective() / ProcessQuestEvent()
          |
          +-- Objectif complete --> Actions executees
          |                         (local ou via QuestActionBridge)
          |
          +-- Tous objectifs completes
                    |
                    v
              CompleteQuest()
              -> AsyncSaveQuestsToBinaryTask
              -> Delegates broadcast
              -> UI notification
```

### 3.2 Flux d'une interaction joueur -> progression

```
Joueur appuie sur "Interact"
        |
        v
InteractionComponent::TryInteract()
        |
        v
IInteractableInterface::Interact()
        |
        v
DynamicInteractableComponent / QuestInteractComponent
        |
        v
PlayerQuestComponent::Server_UpdateObjectiveProgress()  [Server RPC]
        |
        v
QuestManagerSubsystem::RegisterObjectiveProgress()
        |
        +-- Met a jour FRuntimeQuestInstance::ObjectiveProgress
        +-- Appelle UQuestObjectiveBase::ProgressObjective()
        +-- Si complete: execute actions liees
        +-- Replique vers client via PlayerQuestComponent
        +-- Broadcast delegates (natifs + dynamiques)
```

### 3.3 Flux reseau (actions)

```
SERVEUR                                       CLIENT
   |                                             |
   | ExecuteAction()                             |
   |   |                                        |
   |   +-- NetworkMode == LocalOnly?             |
   |   |     -> execute localement               |
   |   |                                        |
   |   +-- NetworkMode == ServerOnly?            |
   |   |     -> execute sur serveur              |
   |   |                                        |
   |   +-- NetworkMode == ClientOnly?            |
   |   |     -> PQC::Client_ExecuteSpawnAction() |---> execute sur client
   |   |                                        |
   |   +-- NetworkMode == RelevancyBased?        |
   |         -> QuestRelevancyManager            |
   |           -> spawn QuestActionBridge        |
   |             -> IsNetRelevantFor()           |
   |               -> Multicast_ExecuteAction()  |---> execute sur clients cibles
   |                                             |
```

---

## 4. Sous-systemes majeurs

### 4.1 QuestManagerSubsystem -- Orchestrateur central

**Fichiers :** `Subsystems/QuestManagerSubsystem.h/.cpp`, `Subsystems/QuestManagerDelegates.h`

#### Architecture interne

Le subsysteme central (~2000 lignes dans le header, ~3000+ dans l'implementation) est un `UGameInstanceSubsystem` implementant `IQuestProvider`. Il est le **point unique de verite** pour :

- `QuestDataMap` : toutes les definitions de quetes chargees
- `PlayerQuestStates` : etat runtime par joueur (TMap<FName, TArray<FRuntimeQuestInstance>>)
- `PlayerFlags` : drapeaux persistants par joueur (TMap<FName, TSet<FName>>)
- `QuestInstanceActors` : acteurs repliques representant les quetes actives
- `ActiveTriggerZoneManagers` : gestionnaires de zones par quete

#### Pattern de delegates dual

Chaque evenement est broadcast a la fois en delegate natif (pour C++) et en delegate dynamique (pour Blueprint) :

```cpp
// Natif (C++ only)
FQuestManager_OnQuestStateChanged OnQuestStateChanged;

// Dynamique (Blueprint + C++)
UPROPERTY(BlueprintAssignable)
FOnQuestStateChangedEvent OnQuestStateChangedEvent;

// Broadcast unifie
void BroadcastQuestStateChanged(const FName& QuestID, EQuestStateType NewState)
{
    OnQuestStateChanged.Broadcast(QuestID, NewState);
    OnQuestStateChangedEvent.Broadcast(QuestID, NewState);
}
```

#### Persistance

- Format binaire versionne (v4) avec header de version
- Fichier de backup automatique (.bak)
- I/O asynchrone via `FNonAbandonableTask` (thread pool) avec callback game thread
- Chemin de sauvegarde segmente par port serveur (multi-instance dedie)
- Thread-safety via `FCriticalSection QuestDataMutex` et `GQuestFileLock`

#### Maintenance periodique

| Timer | Intervalle | Role |
|---|---|---|
| `PeriodicBlueprintReferenceCleanup` | 60s | Nettoyage refs stale (quasi no-op) |
| `CheckInactiveObjectivesForActivation` | 2s | Scan objectifs dormants |
| `ValidateWorldContext` | 5s | Verification contexte monde |
| Scan objectifs pickup | variable | TActorIterator dans 600m pour items |

#### Pieges et subtilites

- **Transition de monde** : sur `OnPostWorldInitialization`, TOUT l'etat en memoire est efface et recharge depuis le disque via un timer de 0.5s. Ceci evite les pointeurs UObject stales apres changement de niveau.
- **Detection lobby** : le nom de monde contenant `"L_Lobby"` suspend le systeme de quetes.
- **Monde tutoriel** : gestion speciale sans rechargement de progression.
- **Nettoyage PIE** : code extensif de nettoyage multi-pass avec GC force, derooting d'urgence de tous les objets quete, gestion des corruptions memoire PIE reseau.
- **`const_cast`** dans `GetTrackedQuests` et `GetAllQuestTags_Implementation` pour mettre a jour des caches depuis des methodes const.

---

### 4.2 PlayerQuestComponent -- Pont reseau

**Fichiers :** `Components/PlayerQuestComponent.h/.cpp`

#### Architecture interne

Composant replique sur le PlayerController. Sert de **pont entre le serveur (QuestManagerSubsystem) et le client (UI)**.

```
SERVEUR                          RESEAU                      CLIENT
QuestManagerSubsystem   -->  PlayerQuestComponent  -->  UI / Son / Dialogue
                              ActiveQuests[] (Rep)
                              CompletedQuests[] (Rep)
                              FailedQuests[] (Rep)
                              TrackedQuestInfo[] (Rep)
```

#### Structures cle

```cpp
struct FActiveLocationObjectiveInfo {
    FName QuestID;
    FName ObjectiveID;
    FVector Location;
    float Radius;
};

struct FObjectiveKey {   // Cle composite hashable
    FName QuestID;
    FName ObjectiveID;
    friend uint32 GetTypeHash(const FObjectiveKey& Key);
};

struct FTrackedQuestInfo {  // Repliquee
    FName QuestID;
    bool bIsTracked;
};
```

#### Optimisation de replication

Le composant calcule une signature de replication (`ComputeQuestReplicationSignature`) pour eviter le traitement redondant dans `OnRep_ActiveQuests`. Ce fix a resolu un probleme de 360+ appels inutiles par replication.

#### Verification de position

Un timer a frequence dynamique (0.05s a 5s) selon la distance au plus proche objectif de location. Plus le joueur est proche, plus la verification est frequente.

#### RPCs

| Direction | RPC | Usage |
|---|---|---|
| Client->Serveur | `Server_AcceptQuest` | Accepter une quete |
| Client->Serveur | `Server_AbandonQuest` | Abandonner une quete |
| Client->Serveur | `Server_UpdateObjectiveProgress` | Rapporter progression |
| Client->Serveur | `Server_RequestQuestDialogue` | Ouvrir dialogue NPC |
| Client->Serveur | `Server_ForwardLocationObjectiveEvent` | Evenement zone |
| Client->Serveur | `Server_NotifyCableInteraction` | Interaction cable |
| Serveur->Client | `Client_ExecuteSpawnAction` | Spawn cote client |
| Serveur->Client | `Client_OpenQuestDialogue` | Ouvrir popup dialogue |
| Serveur->Client | `Client_TriggerObjectiveNotification` | Notification objectif |
| Serveur->Client | `Client_PlayQuestAcceptSound` | Son acceptation |
| Serveur->Client | `Client_PlaySpatialSoundAtLocation` | Son spatial |
| Serveur->Client | `Client_SynchronizeActionState` | Sync etat action |

#### Piege

`FPlayerDataCache` (cache de refs joueur) est explicitement **non replique** -- un commentaire dans le code explique que les `TWeakObjectPtr` causaient des crashs de replication en build package.

---

### 4.3 Systeme d'objectifs

**Fichiers :** `Objectives/QuestObjectiveBase.h/.cpp` (core) + module `DynamicQuestSystemObjectives/`

#### Hierarchie des objectifs

```
UQuestObjectiveBase (core, abstract)
    |
    +-- UConnectorObjective       Puzzle de connexion cable/socket
    +-- UCustomObjective          Extension Blueprint (Blueprintable)
    +-- UEGStateObjective         Surveillance d'etat "Environmental Gadget"
    +-- UInteractObjective        Interaction temporisee
    +-- UKillObjective            Suivi de kills avec filtrage riche
    +-- ULocationObjective        Visite de lieux (le plus complexe)
    +-- UNPCDialogueObjective     Dialogue NPC avec items requis
    +-- UNotificationPlaybackObjective  Attente de notifications
    +-- UQLevelLoadObjective      Attente chargement niveau (QLevel)
    +-- UScannerObjective         Declenchement scanner
    +-- UTriggerOnlyObjective     Auto-completion immediate (trigger d'actions)
```

#### Cycle de vie d'un objectif

```
                 Instancie
                    |
                    v
    [Inactif] --Activate()--> [Actif]
                                |
                    +-----------+-----------+
                    |                       |
              ProgressObjective()     ProcessQuestEvent()
                    |                       |
                    v                       v
              Progress >= Required    Event match?
                    |                       |
                    +----------+------------+
                               |
                               v
                      FinishObjective()
                               |
                    +----------+-----------+
                    |                      |
              [Complete]             [Failed]
                    |                      |
                    v                      v
            Execute actions          Execute fail actions
            Notifications            Notifications
```

#### Pattern de garde d'activation

```cpp
// QuestObjectiveBase garantit que la logique native s'execute
// meme si un Blueprint override oublie d'appeler Super::OnActivated
void UQuestObjectiveBase::OnActivated_Implementation()
{
    StartNativeActivationGuard();  // Set flag
    // ... logique Blueprint peut s'executer ici ...
    EnsureNativeOnActivatedExecuted();  // Si Blueprint n'a pas appele Super
}
```

#### Objectifs notables

**LocationObjective** (~1500 lignes) : Le plus complexe. Gere des visites sequentielles ou non-ordonnees, un systeme de "guardian objective" (completion douce en attente d'un objectif gardien), le spawn de trackers, et une integration editeur bidirectionnelle avec `ALocationObjectiveZoneActor`.

**KillObjective** : Filtrage riche (faction, type ennemi, arme, kills critiques/furtifs, fenetre temporelle, effets de statut, assists). Mecanisme d'auto-reparation : si les entites de quete despawnent, re-execute les actions de maintenance de spawn.

**ScannerObjective** : Integration entierement par reflexion avec l'inventaire (aucune dependance compile-time). Systeme de filtre de scanner statique global (`ActiveScannerFilters`) utilise par l'UI du scanner.

**ConnectorObjective** : Puzzle physique cable/socket. Resolution duale (tag ou reference directe). Audio spatial avec support multijoueur.

---

### 4.4 Systeme d'actions

**Fichiers :** `Actions/QuestActionBase.h/.cpp` (core) + module `DynamicQuestSystemObjectives/Actions/`

#### Architecture

Les actions sont des effets de bord declenchables par le systeme de quete. La classe de base (`UQuestActionBase`, ~1500 lignes) gere :

- Execution avec delai, retries, conditions
- 5 modes reseau : LocalOnly, ServerOnly, ClientOnly, AllClients, RelevancyBased
- Integration notifications (blocage de completion pendant affichage)
- Integration registre dynamique
- Chaines d'execution (action -> action)
- Gestion d'inventaire (spawn/consommation items)

#### Actions disponibles

| Action | Role | Complexite |
|---|---|---|
| `ActivateInteractableAction` | Spawn/active un objet interactif | Moyenne |
| `ControlAmbientAudioAction` | Controle audio ambiant | Faible |
| `ControlLevelSoundAction` | Controle composants audio | Moyenne |
| `HighlightActorAction` | Outline/highlight d'acteurs | Moyenne |
| `OpenPersistentUniverseAction` | Transition de niveau | Faible |
| `PlayLevelSequenceAction` | Cinematique complete | **Haute** (~30 proprietes) |
| `PlayNiagaraAction` | Effet particules | Faible |
| `PlaySoundAction` | Son 2D/spatial/attache | Faible |
| `QuestActionEGControl` | Controle etat Environmental Gadget | Moyenne |
| `QuestActionLightControl` | Controle lumieres (transitions) | Moyenne |
| `QuestNotificationPlayAction` | Notification UI | Faible |
| `SetObjectiveStateAction` | Active/complete/echoue un objectif | Moyenne |
| `SpawnActorAction` | Spawn d'entites (ennemis, objets) | **Haute** |
| `TeleportAction` | Teleportation joueur | Moyenne |
| `TriggerActorAction` | Appel de fonction par reflexion | Moyenne |

#### Pattern d'execution

```cpp
// Deux chemins d'execution complementaires :

// 1. Fire-and-forget (void)
virtual void OnExecuteAction_Implementation();

// 2. Avec retour (bool)
virtual bool ExecuteAction_Implementation();

// Les actions async appellent manuellement :
MarkActionCompleted();  // ou MarkActionFailed();
```

#### Resolution d'acteurs cibles

Toutes les actions utilisent un pattern commun pour trouver leurs cibles :
1. Reference directe (TSoftObjectPtr)
2. Recherche par tag (TActorIterator)
3. Filtrage par nom
4. Filtrage par classe
5. Registre dynamique (cle de registre)

---

### 4.5 Systeme de conditions

**Fichiers :** `Conditions/QuestConditionBase.h/.cpp` (core) + module `DynamicQuestSystemObjectives/Conditions/`

#### Architecture

Evaluateurs stateless invoquees par le systeme de quete. La classe de base supporte :
- Types : Acceptance, Activation, Progress, Failure, Skip
- Inversion (`bInvertResult`)
- Cache de resultat
- Delegates de changement d'etat
- Serialisation binaire

#### Conditions disponibles

| Condition | Evaluation |
|---|---|
| `CheckPlayerFlagCondition` | `QuestManager->HasPlayerFlag(PlayerID, FlagName)` |
| `IsMultiplayerCondition` | `World->GetNetMode() != NM_Standalone` |
| `LevelSequenceCondition` | Cinematique terminee (scoring heuristique multi-acteurs) |

---

### 4.6 Systeme de dialogue

**Fichiers :** `UI/QuestDialogue*.h/.cpp`, `UI/QuestNPCMenuWidget.h/.cpp`, `Actors/QuestTriggerActor.h/.cpp`

#### Architecture

```
AQuestTriggerActor (NPC)
    |
    +-- DialogueLines[]
    +-- DialogueBranches[] (avec conditions prioritisees)
    +-- AvailableQuestIDs[]
    |
    v
ResolveDialogueForPlayer()
    |
    v
FQuestDialogueSessionData
    |
    v
UQuestDialoguePopupWidget
    |-- Machine a etats : None -> Dialogue -> QuestList -> FinalChoice
    |-- Animation de frappe (typewriter)
    |-- Navigation clavier/souris/gamepad
    |-- Lecture audio synchronisee
    |-- Panneau de details
    |-- Boutons Accept/Decline/Skip
```

#### Branches conditionnelles

Les branches de dialogue sont evaluees par priorite avec conditions sur l'etat des quetes et objectifs :

```cpp
struct FQuestDialogueBranch {
    int32 Priority;                      // Plus eleve = evalue en premier
    TArray<FQuestDialogueCondition> Conditions;  // Toutes doivent etre vraies
    TArray<FQuestDialogueLine> Lines;
};

struct FQuestDialogueCondition {
    FName QuestID;
    EQuestDialogueQuestState RequiredQuestState;  // NotStarted/Active/Completed/Failed
    FName ObjectiveID;  // Optionnel
    EQuestDialogueObjectiveState RequiredObjectiveState;  // Inactive/Active/Completed
};
```

---

### 4.7 Editeur de quetes

**Fichiers :** Module `DynamicQuestSystemEditor/` (70 fichiers)

#### Architecture

```
+--------------------------------------------------------------+
|  FQuestEditorWindow (Nomad Tab - SCompoundWidget)            |
|                                                              |
|  +----------+  +-------------+  +-----------+  +----------+ |
|  | SQuestList|  | Details Tab |  | Objectives|  | Actions  | |
|  | (arbre)  |  | (champs)   |  | (categ.)  |  | (categ.) | |
|  +----------+  +-------------+  +-----------+  +----------+ |
|                                                              |
|  +----------+  +-------------+  +-----------+  +----------+ |
|  | NPCs Tab |  | TriggerZones|  | QuestFlow |  | Settings | |
|  +----------+  +-------------+  | (graphe)  |  +----------+ |
|                                  +-----------+               |
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

#### Persistance editeur

Trois generations coexistent :
1. **Binaire** (legacy `.bin`) -- supporte pour migration uniquement
2. **JSON** (actuel) -- un fichier par quete dans `Content/Quests/EditorQuests/`
3. **Export binaire** (`RuntimeQuests.bin`) -- pour le jeu package

Migration automatique : si un `.json` n'existe pas mais un `.bin` existe, le systeme convertit et archive le binaire en `.legacy`.

#### Systeme de verrouillage

Verrouillage pessimiste par fichier (`.lock` dans `ProjectSaved/QuestEditorLocks/`) avec detection de verrous stales (timeout 300s). `FQuestLockInfo` stocke utilisateur/machine/timestamp.

#### Graphe visuel (QuestFlowGraph)

Un systeme `UEdGraph` complet avec schema, nodes, pins, politique de dessin de connexions et widgets Slate. **Le graphe est en lecture seule** : il visualise la structure de la quete mais `SaveToQuestDefinition()` est vide. Les nodes sont colores par type (objectif, action, condition, start, complete) avec des animations d'etat.

#### Editeurs modulaires

Deux generations coexistent :
1. `SModularComponentEditor<T>` (ancien, simple)
2. `SCategorizedComponentEditor<T>` (actuel, avec categories, drag-and-drop, copier/coller)

Les deux sont des templates C++ instancies pour `UQuestObjectiveBase`, `UQuestActionBase`, `UQuestConditionBase`.

---

### 4.8 Systeme d'interaction

**Fichiers :** Module `DynamicQuestSystemInteraction/` (14 fichiers)

#### Architecture

Module **completement autonome** sans aucune dependance sur le systeme de quete :

```
IInteractableInterface (9 methodes BlueprintNativeEvent)
        ^
        |
AInteractableActor (base abstraite, implementations par defaut)
        ^
        |
UInteractionComponent (detection par line trace periodique)
        |
        v
UInteractionSubsystem (registre global, recherche spatiale/tags)
```

#### Detection

`UInteractionComponent` effectue des line traces periodiques depuis la camera du joueur. Intervalle configurable (defaut 0.1s). Gere l'etat de focus, les delegates, et le widget de prompt.

#### Limitation connue

La distance configuree sur `AInteractableActor::InteractionDistance` n'est **jamais interrogee** par `InteractionComponent` qui utilise sa propre `DefaultInteractionDistance`. La propriete de l'acteur est donc effectivement morte.

---

### 4.9 Systeme de debug

**Fichiers :** Module `DynamicQuestSystemDebug/` (8 fichiers)

#### Commandes console

| Commande | Usage |
|---|---|
| `dqs.ToggleDebugUI` | Active/desactive l'overlay Slate |
| `dqs.AcceptQuest <QuestID>` | Force l'acceptation d'une quete |
| `dqs.ClearAllQuests` | Efface toute la progression |
| `dqs.ExecuteAction <QuestID> <ActionID> [params]` | Execute une action specifique |
| `dqs.CompleteObjective [QuestID]` | Complete les objectifs actifs |
| `dqs.CompleteQuest [QuestID]` | Complete tous les objectifs (0.5s entre chaque) |

#### Overlay Slate (SDynamicQuestDebugWidget)

Widget Slate temps reel avec 3 panneaux (overview, objectifs, actions) et une section "Dynamic Registry" collapsible. Met a jour a 1Hz. Utilise un pattern snapshot-and-diff pour minimiser la reconstruction de widgets.

L'inspection du registre dynamique utilise la **reflexion UE** (`TFieldIterator<FProperty>`) pour introspecter 13 noms de proprietes hardcodes -- fragile si le registre change.

---

### 4.10 Quetes procedurales

**Fichiers :** `Data/ProceduralQuestTypes.h`, `Data/ProceduralQuestTemplate.h/.cpp`, `Data/MultiStageQuestTemplate.h/.cpp`, `Data/QuestArchetypeTypes.h`

#### Architecture

```
Monde (WorldScape + QLevel)
        |
        v
Decouverte de POIs (FDiscoveredPOI)
  - SanglineNest, EnemySpawner
  - MineralDeposit, Cave
  - Bunker, Outpost
        |
        v
Selection de template (par type POI)
        |
        +-- USanglineExterminationTemplate
        +-- USpawnerClearanceTemplate
        +-- UResourceCollectionTemplate
        +-- UAreaExplorationTemplate
        +-- UFactionEliminationTemplate
        +-- UBunkerRaidTemplate
        +-- UCaveExplorationTemplate
        +-- UOutpostCaptureTemplate
        |
        v
GenerateQuest(POI, Difficulty)
        |
        v
FQuestDefinition (objectifs + actions generes)
        |
        v
AssignQuestToPlayer()
```

#### Archetypes multi-etapes

```cpp
UMultiStageQuestTemplate (base)
    +-- UInfiltrationArchetype    // Approche -> Infiltration -> Extraction
    +-- UExterminationArchetype   // Approche -> Combat -> Nettoyage
    +-- UMiningArchetype          // Approche -> Collecte -> Transport
    +-- UInvestigationArchetype   // Approche -> Investigation -> Rapport
    +-- UDefenseArchetype         // Approche -> Preparation -> Defense
```

Chaque etape genere des objectifs et actions avec scaling de difficulte. Les complications (ennemis supplementaires, contraintes temporelles) sont injectees selon la difficulte.

---

## 5. Architecture reseau

### 5.1 Autorite

| Donnee | Autorite | Replication |
|---|---|---|
| Definitions de quetes | Serveur (QuestManagerSubsystem) | Non repliquees (chargees localement) |
| Etat runtime (progression) | Serveur (QuestManagerSubsystem) | Via PlayerQuestComponent |
| Quetes actives | Serveur | `DOREPLIFETIME` -> `OnRep_ActiveQuests` |
| Quetes completees | Serveur | `DOREPLIFETIME` -> `OnRep_CompletedQuests` |
| Quetes echouees | Serveur | `DOREPLIFETIME` -> `OnRep_FailedQuests` |
| Suivi de quete | Serveur | `DOREPLIFETIME` -> `OnRep_TrackedQuestInfo` |
| Actions (ciblage) | Serveur -> QuestActionBridge | `IsNetRelevantFor` -> clients cibles |

### 5.2 QuestActionBridge

Acteur replique ephemere cree par `QuestRelevancyManager` pour router les actions vers les joueurs concernes. Override `IsNetRelevantFor` pour ne repliquer que vers les joueurs cibles. Auto-destruction apres un delai configurable.

```cpp
struct FQuestActionNetworkData {
    FName QuestID;
    FName ActionID;
    FName PlayerID;
    TSubclassOf<UQuestActionBase> ActionClass;
    TMap<FName, FString> ActionParameters;
};
```

---

## 6. Structures de donnees de reference

### 6.1 FQuestDefinition

```cpp
struct FQuestDefinition {
    FName QuestID;
    FText Title;
    FText Description;
    EQuestDifficulty Difficulty;
    int32 RecommendedLevel;
    FGameplayTagContainer QuestTags;
    bool bIsRepeatable;
    bool bAutoTrack;
    bool bIsStoryQuest;

    // Composants modulaires
    TArray<TObjectPtr<UQuestObjectiveBase>> Objectives;
    TArray<TObjectPtr<UQuestActionBase>> Actions;
    TArray<TObjectPtr<UQuestConditionBase>> AcceptanceConditions;
    TArray<TObjectPtr<UQuestConditionBase>> FailureConditions;

    // Zones de declenchement
    TArray<FTriggerZoneData> TriggerZones;

    // Flux du graphe
    TArray<FQuestFlowConnectionData> FlowConnections;

    // Recompenses
    FQuestRewardInfo Rewards;

    // NPC assigne
    FString AssignedNPCID;

    // Serialisation
    TArray<FModularComponentData> SerializedObjectives;
    TArray<FModularComponentData> SerializedActions;
    TArray<FQuestConditionData> SerializedConditions;

    // Flags
    bool bIsTemporaryCopy = false;

    // Operateurs de serialisation binaire
    friend FArchive& operator<<(FArchive& Ar, FQuestDefinition& Def);
};
```

### 6.2 FRuntimeQuestInstance

```cpp
struct FRuntimeQuestInstance {
    FName QuestID;
    FName PlayerID;
    EQuestStateType State;
    FText Title;
    FText Description;

    // Progression par objectif
    TMap<FName, int32> ObjectiveProgress;
    TArray<FObjectiveProgressPair> ObjectiveProgressArray;  // Pour replication

    // Instances vivantes
    TMap<FName, TObjectPtr<UQuestObjectiveBase>> ObjectiveInstances;
    TMap<FName, TObjectPtr<UQuestActionBase>> ActionInstances;

    // Objectifs actifs
    TArray<FName> ActiveObjectives;

    // Etat des actions
    TMap<FName, FString> ActionStateMap;
    TArray<FActionStatePair> ActionStateArray;  // Pour replication

    // Timestamps
    TMap<FName, float> ObjectiveCompletionTimes;
    TArray<FObjectiveTimePair> ObjectiveCompletionTimeArray;

    // Suivi
    bool bIsTracked;

    // Serialisation
    friend FArchive& operator<<(FArchive& Ar, FRuntimeQuestInstance& Inst);
};
```

### 6.3 Enumerations principales

```cpp
enum class EQuestStateType : uint8 {
    NotStarted, Active, Completed, Failed, Abandoned
};

enum class EObjectiveType : uint8 {
    Kill, Location, Interact, Custom, Dialogue, Connector,
    Scanner, EGState, LevelLoad, TriggerOnly, NotificationPlayback
};

enum class EQuestDifficulty : uint8 {
    Easy, Normal, Hard, VeryHard, Nightmare
};

enum class EQuestFlowNodeType : uint8 {
    Start, Objective, Action, Condition, Complete
};

enum class EQuestFlowConnectionType : uint8 {
    Sequential, Conditional, Parallel, Optional, Failure
};
```

### 6.4 FQuestNotificationPayload

```cpp
struct FQuestNotificationPayloadBase {
    FText Title;
    FText Message;
    FText SubMessage;
    // ~50 proprietes : type, audio, typewriter, icone, duree, style...
};

struct FQuestNotificationSequenceEntry {
    FQuestNotificationPayloadBase Payload;
    float DelayBeforeEntry;
    bool bWaitForPreviousDismissal;
};

struct FQuestNotificationPayload {
    TArray<FQuestNotificationSequenceEntry> Entries;
    bool bIsBlocking;              // Empeche completion pendant affichage
    FName BlockingSourceID;
};
```

---

## 7. Points d'attention

### 7.1 Subtilites -- Ce qu'un developpeur doit absolument savoir

1. **AddToRoot/RemoveFromRoot** : Les objectifs, actions et conditions sont `AddToRoot()`'d par `QuestComponentManager` pour eviter le GC. Tout oubli de `RemoveFromRoot` cree un leak. Le nettoyage est gere par `CleanupReferences()` dans `FQuestDefinition::~FQuestDefinition` avec des gardes defensives.

2. **Transition de monde** : Sur changement de niveau, l'etat en memoire est **integralement detruit et recharge** depuis le disque. Ne jamais stocker de references durables vers des objets quete entre les niveaux.

3. **Deux systemes d'evenements** : `UQuestEventManager` (UObject, avec handlers prioritises) et `FQuestEventManager` (singleton statique, delegates multicast). Les deux coexistent avec des roles differents.

4. **Double factory** : `UQuestObjectiveFactory` et `UQuestAssetManager` font essentiellement la meme chose (decouverte et enregistrement de classes). Utiliser `QuestObjectiveFactory`.

5. **Graphe visuel en lecture seule** : Le `QuestFlowGraph` ne sauvegarde jamais vers la definition. Il est purement visuel.

6. **Resolution de PlayerID** : Cascade complexe dans `PlayerQuestSubsystem::GetAuthoritativePlayerID()` avec 4+ fallbacks. Les IDs `"0"` et `"STEAM_ID_FAILED"` sont rejetes comme invalides.

7. **Nettoyage PIE agressif** : Le module Objectives fait un nettoyage PIE avec rename vers le transient package, `ConditionalBeginDestroy`, et GC force. Necessaire a cause de leaks passes.

8. **Reflexion intensive** : `ScannerObjective`, `EGStateObjective`, `NPCDialogueObjective`, `TriggerActorAction` et `QuestActionEGControl` utilisent massivement `ProcessEvent`/`FindFProperty` pour eviter des dependances compile-time. Ces chemins sont fragiles face aux renommages de proprietes/fonctions.

9. **Serialisation binaire defensif** : `FModularQuestArchiveProxy` avale silencieusement les references UObject manquantes (assets deplaces/supprimes). Ceci evite les crashs mais peut causer des pertes de donnees silencieuses.

10. **Actions et triple RPC** : `MarkActionCompleted`/`MarkActionFailed` iterent tous les PlayerControllers et envoient des RPCs via tous les `PlayerQuestComponent` trouves sur PC, PlayerState ET Pawn -- potentiellement triple envoi par joueur.

### 7.2 Limitations connues

1. **QuestRelevancyManager** : `IsPlayerQuestParticipant` retourne toujours `true`. `IsPlayerOnSameTeam` retourne toujours `true`. La resolution de PlayerID utilise `GetPlayerName()` au lieu du Steam ID. Le systeme de relevance est donc fonctionnellement un broadcast a tous.

2. **LevelMeshScanner::PerformAsyncScan** *(corrige)* : la passe UObject (iteration d'acteurs, `GetComponents`, acces mesh, `NewObject`) est desormais differee sur le game thread via `AsyncTask(ENamedThreads::GameThread, ...)`. Auparavant elle tournait sur un thread pool et derefencait des acteurs liberes pendant le chargement de niveau (crash use-after-free dans `IsA<ABrush>`).

3. **InteractionComponent** : `bCanEverTick = true` toujours, meme quand le timer est utilise (tick ne fait rien dans ce cas -- gaspillage). La distance de l'interface (`GetInteractionDistance()`) n'est jamais interrogee.

4. **ClearWithPrefix dans QuestDynamicRegistry** : Ne decouvre les cles que dans `StringValues`, rate les cles presentes uniquement dans d'autres maps typees.

5. **ModularInteractionPromptWidget** : `NativeConstruct()` met `CurrentInteractableActor` a null, ce qui race avec `SetInteractableActor` appele avant `AddToViewport`. Le prompt peut ne jamais afficher de donnees.

6. **QuestFlowGraph** : Pas de sauvegarde, pas d'auto-layout reel (`OnAutoArrange` fait juste zoom-to-fit), pas de validation reelle (`OnValidateGraph` fait juste refresh), pas de breakpoints fonctionnels (brushes non enregistres).

7. **CanPasteComponent** dans `SCategorizedComponentEditor` : Deserialise le JSON du clipboard a chaque frame pendant l'evaluation de `IsEnabled()` du bouton -- cout de performance.

### 7.3 Hacks et dette technique

| Fichier | Description | Raison probable |
|---|---|---|
| `QuestDataTypes.h` | Verification de corruption memoire via `reinterpret_cast<intptr_t>` dans le destructeur de `FQuestDefinition` | Crashs durant le shutdown/GC |
| `QuestDataTypes.h` | `FModularQuestArchiveProxy` duplique `FQuestSilentObjectProxyArchive` de `SerializeQuestUtils.h` | Evolution independante des deux fichiers |
| `QuestManagerSubsystem.cpp` | Detection lobby par substring `"L_Lobby"` dans le nom de monde | Pas d'enum/tag pour les types de monde |
| `QuestManagerSubsystem.cpp` | Timer de 0.5s pour recharger apres transition de monde | Race condition avec l'initialisation du monde |
| `BinaryQuestDataSource.cpp` | Logging de debug hardcode pour la quete "Q001" (lignes 415-427) | Artefact de developpement |
| `InteractableMeshDatabase.cpp` | ~200 mots-cles hardcodes avec typo "StorageContianer" | Base initiale, devrait etre en config |
| `ProceduralQuestTemplate.cpp` | `static UClass* CachedClass = nullptr` dans les helpers -- persiste entre sessions PIE | Optimisation de performance |
| `QuestObjectiveBase.cpp` | `GetOwningPlayerPawn()` et `GetOwningPlayerState()` : ~100 lignes dupliquees | Resolution de joueur complexe non factorisee |
| `QuestConditionBase.cpp` | 5 `DQS_LOG_WARNING` par evaluation de condition | Logging excessif oublie en production |
| `FQuestDebugCommands` | `RegisterConsoleCommand()` et `UnregisterConsoleCommand()` sont des stubs vides | Les vraies commandes sont via `FAutoConsoleCommand` statics |
| `QuestEditorWindow.h` | `UPROPERTY()` sur `QuestIconTexture` dans un `SCompoundWidget` (non-UObject) | Macro sans effet, jamais nettoye |
| `ComponentOrganizationManager.cpp` | Commentaire "BenFix ma guele" | Commentaire de dev oublie |
| `FQuestDefinition::CleanupReferences` | Bloc condition avec corps vide pour `AcceptanceConditions` (ligne ~1319) | Code de nettoyage retire mais condition conservee |
| `QuestFlowConnectionDrawingPolicy.h` | `DoesLineIntersectNode` et `FindPathAroundObstacles` declares mais jamais definis | Fonctionnalite planifiee non implementee |
| `SQuestFlowPin.cpp` | `bIsDragging` set sur `OnMouseButtonDown` mais jamais reset | Reset manquant |

---

## 8. Arborescence complete des fichiers

### Module Core (`DynamicQuestSystem/`)

```
DynamicQuestSystem.Build.cs                          -- Config build, deps lourdes (WorldScape, QLevel, etc.)

Public/
  Core/
    DynamicQuestSystem.h                             -- Module, log category, macros DQS_LOG
    DynamicQuestSystemSettings.h                     -- UDeveloperSettings, config plugin
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
    IQuestProgress.h                                 -- Interface cycle de vie joueur (16 methodes)
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
    QuestNotificationSettings.h                      -- Config notifications (~50 proprietes)
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
    QuestFlowSettings.h                              -- UDeveloperSettings graphe visuel
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