# QTrain — Architecture

Système de train/navette sur rail pour QANGA, par Iolacorp. Module `Runtime` unique (`QTrain`), `LoadingPhase` `Default`, `CanContainContent: true`. Dépend des plugins `Niagara` et `QPlatform`.

---

## 1. But d'usage

QTrain fait circuler des véhicules (trains, navettes) le long de `USplineComponent` posées dans le monde, à l'échelle planétaire de QANGA (distances en centaines de kilomètres, coordonnées LWC). Le problème concret qu'il résout n'est pas « déplacer un acteur sur une spline » — c'est de le faire **à très grande distance et en multijoueur sans coût** :

- La position du train sur le rail est calculée **analytiquement** (distance le long de la spline `Internal_SplineLocation`) et n'a besoin d'aucun acteur visible tant qu'aucun joueur n'est proche. Le rail (`AQTrain_Spline_Track`) maintient un état « virtuel » et ne **spawn** l'acteur train (`AQTrain_Train`) que dans un rayon configurable.
- Le rail est **autoritaire serveur** ; le serveur ne réplique qu'une poignée de scalaires (position spline + vitesse), et chaque client **rejoue le même calcul de spline** localement pour reconstruire la position/rotation, avec compensation de ping. C'est un modèle de réplication minimaliste (pas de réplication de transform par frame).
- Sur planète, l'orientation du train est dérivée de la gravité (vecteur vers `Planet_OriginLocation`) combinée à la tangente de la spline, pour que le train « colle » à la surface courbe.
- `AQTrainSeatActor` permet à un joueur de **s'asseoir** dans le train en mouvement. Le siège réutilise les systèmes de QANGA existants (composant d'interaction content-side, état de mouvement ALS, `UQPlatformComponent`, et le composant client-authoritative `SmoothTSync`) plutôt que d'en créer un nouveau — d'où une forte intégration par réflexion avec du contenu Blueprint.

Le système gère aussi le **docking** (arrêts en gare avec attente/file), la **marche arrière** (aller-retour sur le rail), un **mode warp** (accélération au milieu du trajet), et des effets de **propulseurs** Niagara purement cosmétiques.

---

## 2. Vue d'ensemble / décisions d'architecture

Le système se compose de quatre acteurs et d'un `UWorldSubsystem`, avec une séparation nette **rail (logique/autorité) ↔ train (présentation/réplication)**.

- **`AQTrain_Spline_Track`** porte la spline, la vitesse, l'état de progression et toute la simulation. **Il ne tourne réellement que sur le serveur / standalone** (`BeginPlay` désactive le tick côté client). Il possède l'instance `AQTrain_Train` (`Internal_Train`), la spawn et la détruit selon la distance au joueur le plus proche.
- **`AQTrain_Train`** est l'acteur visible et **répliqué**. Il ne décide rien : il reçoit (ou recalcule) la position spline, en dérive un transform via les helpers du rail, et applique sa position. Côté client, il **re-simule** le mouvement à partir de la dernière position serveur connue + ping, pour masquer la latence.
- **`AQTrain_Dock`** est un point d'arrêt qui sérialise l'accès au quai (un train docké à la fois, les autres mis en file d'attente).
- **`UQTrain_SubSystem`** est un registre par-monde des rails (par `ReferenceName`) ; il permet au train de retrouver son rail par `ReferenceID`, et expose des helpers de (dé)sérialisation des positions de tous les trains (pour sauvegarde / persistance).
- **`AQTrainSeatActor`** est indépendant du pipeline rail/train : c'est un acteur siège répliqué qui attache un pawn et neutralise/restaure son mouvement.

Décisions notables et état réel du code :

- **LOD par distance, piloté serveur.** `Compute_Updating_Distance()` choisit un mode (`ETrainUpdateType`) selon la distance carrée au pawn joueur le plus proche : `OnlyVirtual` (pas d'acteur, mise à jour par timer toutes les `Virtual_Frequency` s, optionnellement sur un thread worker via `Async_SplineUpdate`), `LowFrequency` (tick à fréquence variable), `RealTime` (tick chaque frame). En `OnlyVirtual`, l'acteur train est **détruit**.
- **Tick custom hors-acteur.** Le rail enregistre une `FCustomTickFunction` (`Update_LowFrequency`, `TG_PostUpdateWork`, intervalle `CheckMode_Interval`) qui ré-évalue le LOD de distance même quand le tick principal de l'acteur est désactivé (cas `OnlyVirtual` async).
- **Mode async opt-in.** `Use_VirtualAsync` déporte `SplineUpdate` sur `ENamedThreads::AnyNormalThreadHiPriTask`, puis ré-entre sur le GameThread pour appliquer le résultat et décider du spawn — le calcul de spline pur n'a pas besoin du jeu. (Note : `Async_SplineUpdate` lit les positions des pawns via `UGameplayStatics::GetPlayerPawn` **avant** de partir sur le worker, pas pendant.)
- **Réplication par recalcul, pas par transform.** Voir §5. C'est le choix central : on réplique l'entrée (position spline) et chaque machine régénère la sortie (transform).
- **Siège : pas de système custom, réutilisation par réflexion.** `AQTrainSeatActor` ne définit pas son propre flux d'interaction : il s'enregistre dynamiquement comme implémentant l'interface content `/Game/Systems/Interact/Interact_Interface`, et pilote l'état du pawn en cherchant par nom des fonctions/propriétés Blueprint (ALS `BPI_Set_MovementState`, `ReturnLocationSit`, `SittingSwitchLock`, composant `SmoothTSync`, `UQPlatformComponent`). C'est fragile par construction (couplé à des noms content), mais volontaire pour rester cohérent avec le pawn ALS de QANGA.
- **Le siège sur serveur dédié sans mesh est un cas dur connu** (voir §7). Le code contient des commentaires détaillés expliquant pourquoi le rétablissement de mouvement passe par une **movement base CMC** (based-movement) et non un scene-attach, et pourquoi `StabilizePawnOnSeatExit` no-op sur serveur dédié.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `AQTrain_Spline_Track` | `UCLASS(Blueprintable)` : `AActor` | Rail. Porte `USplineComponent Track`, simule la progression (`Internal_SplineLocation`), gère vitesse/ease/warp/docking, LOD par distance, spawn/destruction du train. Autoritaire serveur. |
| `AQTrain_Train` | `UCLASS(Blueprintable)` : `AActor` | Train visible et répliqué. Recalcule/applique le transform à partir de la position spline + ping, expose des events `QTrain_Client_*` à Blueprint, anime les propulseurs Niagara. |
| `AQTrain_Dock` | `UCLASS(Blueprintable)` : `AActor` | Quai. Sérialise l'accès (un train docké, file `Private_TrainOnWaitDocking`). Porte un `USceneComponent` racine. |
| `AQTrainSeatActor` | `UCLASS(Blueprintable)` : `AActor` | Siège répliqué. Attache un pawn (`OccupantPawn`), suspend/restaure son mouvement, gère l'interaction et la clearance de sortie. |
| `UQTrain_SubSystem` | `UCLASS` : `UWorldSubsystem` | Registre par-monde des rails par `ReferenceName` ; lookup `QTrain_GetSpline`, helpers (dé)sérialisation des positions. |
| `ETrainUpdateType` | `UENUM(Blueprintable)` : `uint8` | Mode de mise à jour du rail : `RealTime`, `LowFrequency`, `OnlyVirtual`. |
| `FQTrain_PosData` | `USTRUCT(BlueprintType)` | Snapshot de position d'un rail : `ReferenceName`, `Forward`, `SplineLocation`. Utilisé pour la persistance. |
| `AQTrain_Spline_Track::FCustomTickFunction` | `struct : FTickFunction` (privé, imbriqué) | Tick custom enregistré séparément de l'acteur, pour ré-évaluer le LOD même quand le tick acteur est off. |
| `FQTrainAnimatedThrusterState` | `struct` (anonyme, `.cpp`) | État par-train de l'animation/VFX des propulseurs (cache, lissage, composants Niagara). Stocké dans la map globale `GQTrainAnimatedThrusterStates`. |
| `FQTrainModule` | `IModuleInterface` | Module runtime. `StartupModule`/`ShutdownModule` vides (aucune init globale). |

Délégués exposés par le rail (`DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam`, `BlueprintAssignable`) : `OnStop`, `OnStart`, `OnWaiting`, `TrainSpawned`.

---

## 4. Flux de données et cycle de vie

### Rail (`AQTrain_Spline_Track`)
1. **`BeginPlay`** : initialise `Internal_Speed = MaxSpeed`, s'enregistre dans `UQTrain_SubSystem` (`QTrain_Register`), appelle `Init_Spline()` (calcule `Internal_SplineLength`, place le départ à 0 ou à la longueur selon `Start_ToForward`, broadcast `OnStart`).
   - **Serveur/standalone uniquement** : `Register_CustomTick()`, puis `Compute_Updating_Distance()` + `Startup_Virtual_Updating()` (démarre en `OnlyVirtual`). Côté client pur, le tick acteur est désactivé.
   - `Start_Delay` optionnel : un timer `StopStartDelay` gèle la simulation pendant `Start_Delay_Time` s.
2. **Tick / async** : tant que non en pause et non en délai, `Checking_Docking` → `SplineUpdate` → `GetOnEnd` (détection de fin de course), puis `Update_ActorPosition` (interpole l'acteur train vers la position courante, serveur dédié seulement si `FilterAutoTransformDedicatedServer`). En `OnlyVirtual` async, ce travail passe par `Async_SplineUpdate` (timer `VirtualTimer`).
3. **`SplineUpdate`** : calcule la vitesse (warp, ease in/out aux extrémités via `SlowingDownDistance_*` et `LandingDistance_*`, facteur d'échelle de spline, `DockingInterp`), avance/recule `Internal_SplineLocation` clampée à `[0, Internal_SplineLength]`, met à jour `Internal_Location`.
4. **Fin de course (`GetOnEnd`)** : quand `Internal_SplineLocation` atteint une extrémité → `OnPause = true`, broadcast `OnStop`, ouvre les portes du train, programme `PreRestart` (pré-démarrage / fermeture de portes) puis `Restart` (inversion de `Internal_Forward`, ré-émission de `OnStart`, notification au train).
5. **Spawn/destruction du train** : `Get_SpawnTrain(Distance)` spawn `Internal_Train` (`SpawnActorDeferred<AQTrain_Train>` avec `TrainClass`, oriente selon gravité+tangente) dans le rayon `Spawn_Actor_Distance`, le détruit hors rayon ou en `OnlyVirtual`. Gère aussi hidden-by-distance, collision-by-distance et proxy-mode.
6. **`EndPlay`** : `UnRegister_CustomTick`, se désenregistre du subsystem, détruit `Internal_Train`.

### Train (`AQTrain_Train`)
1. **`BeginPlay`** : s'abonne à `RootComponent->TransformUpdated`, résout `ReferenceTrack` (via `ReferenceID` + `UQTrain_SubSystem::QTrain_GetSpline` si non assigné), puis `InitializeRuntimeFromReferenceTrack()`.
2. **`InitializeRuntimeFromReferenceTrack`** branche selon `GetNetMode()` :
   - `NM_DedicatedServer` : copie l'état initial du rail, `IsDedicatedServer = true`, tick à `Server_ReplicationUpdate_Tick`.
   - `NM_Client` : `UpdateLocal = true`, tick à 0 (chaque frame), `QTrain_Client_Init_Start()`.
   - `NM_Standalone` : copie l'état du rail, `IsStandAlone = true`, tick à 0.
3. **`Tick`** route vers `QTrain_Server_Update`, `QTrain_StandAlone_Update` ou `QTrain_Client_Update` selon le mode. Le client recalcule sa position (`QTrain_Client_UpdateLocation` + `QTrain_Client_ComputePositionRotation`), applique le transform si `Auto_SetTransform`, et anime les propulseurs (`UpdateQTrainAnimatedThrusters`).
4. **Events Blueprint** : chaque changement d'état (porte, docking, direction, start/stop, warp, prestart) appelle un `BlueprintNativeEvent` `QTrain_Client_*` que le Blueprint dérivé surcharge pour la présentation (animation de portes, sons, etc.).
5. **`EndPlay`/`BeginDestroy`** : `ResetQTrainAnimatedThrusters`, détruit l'instance proxy éventuelle.

### Subsystem
Cycle de vie standard `UWorldSubsystem`. Pas de tick. Sert de table `ReferenceName → AQTrain_Spline_Track*` (`Internal_Spline`) plus un `TSet` `Registed_Spline`. `Set_Track_Location`/`Get_Track_Location` poussent/lisent les positions ; un train enregistré après coup récupère sa position en attente (`Internal_Track_Location`).

### Siège (`AQTrainSeatActor`)
- **Constructeur** : crée `Root`, `SeatAnchor`, `ExitAnchor`, `InteractionBounds` (box `QueryOnly`, réponse `ECC_Visibility = Overlap`), `bReplicates = true`. Enregistre dynamiquement l'interface content `Interact_Interface`.
- **Sit (`TrySeatPawn`, autorité)** : `CanSeatPawn` → `ApplySeatedState` (sauvegarde l'état, détache la plateforme, attache à `SeatAnchor`, verrouille l'input, état ALS `Sitting`, suspend le mouvement basé/CMC et `SmoothTSync`), positionne `OccupantPawn`, active le tick.
- **Tick** : `MaintainSeatedAttachment` ré-épingle le pawn sur `SeatAnchor` chaque frame ; si l'autorité détecte que le pawn a quitté l'état ALS `Sitting` par un autre chemin → `QueueExternalStandCleanup` (filet de sécurité).
- **Stand (`TryStandPawn`, autorité)** : vérifie la clearance de sortie (`IsExitClear`, overlap capsule à `ExitAnchor`), `RestoreFromSeat` (détache, restaure l'état, ré-active plateforme/`SmoothTSync`, `StabilizePawnOnSeatExit` = re-base CMC sur le sol du train), libère le siège.
- **Réplication** : `OnRep_OccupantPawn` rejoue sit/stand sur les proxies en comparant `OccupantPawn` à `AppliedOccupantPawn` (transient).

---

## 5. Réplication / réseau

Deux acteurs répliquent (`bReplicates = true`) : `AQTrain_Train`, `AQTrainSeatActor`. (`AQTrain_Spline_Track` et `AQTrain_Dock` ne répliquent pas — le rail est purement serveur, le dock est serveur.)

### `AQTrain_Train` — modèle « répliquer l'entrée, recalculer la sortie »
Le serveur ne réplique **pas** le transform du train. Il réplique :

- **`Internal_Server_SplineLocation`** (`ReplicatedUsing = OnRep_ServerLocationUpdate`) et **`Internal_Server_Speed`** (`DOREPLIFETIME`) — l'entrée du calcul.
- Un bloc d'état initial uniquement (`COND_InitialOnly`) : `ReferenceID`, `QTrain_Start`, `QTrain_DoorOpen`, `QTrain_Forward`, `QTrain_WarpMode`, `QTrain_Docking`, `QTrain_DockingInterp(_Target/_Speed)`.
- **`Switch_Proxy_Mode`** (`ReplicatedUsing = OnRep_Switch_Proxy_Mode`) — bascule l'acteur train vers un acteur proxy léger (`Proxy_ActorClass`) à grande distance.

Côté serveur, `QTrain_Server_Update` lit la position courante du rail dans `Internal_Server_SplineLocation`. Côté client :
- À la réception (`OnRep_ServerLocationUpdate`), si `Use_PingPredictionLocation`, le client avance la position serveur reçue de `GetClientPing()` (ping `PlayerState` × `PingFactor`) via `QTrain_GetNextSplineLocation` du rail.
- Chaque frame, `QTrain_Client_UpdateLocation` interpole `Internal_Client_SplineLocation` vers `Internal_Server_SplineLocation` (vitesse `Difference_Prediction_InterpSpeed`) puis ré-avance le résultat → mouvement fluide qui converge vers l'autorité.
- `QTrain_Client_ComputePositionRotation` génère le transform via `AQTrain_Spline_Track::QTrain_GetTransformAtLocation` (interp position + rotation gravité/tangente).

Les changements d'état discrets sont diffusés par **RPC `NetMulticast, Reliable`** : `ToClient_DockingChange`, `ToClient_ForwardChange`, `ToClient_PreStart`, `ToClient_DoorUpdate`, `ToClient_OnWarpMode`, `ToClient_OnStopStart`. Chaque implémentation met à jour l'état local et appelle l'event Blueprint correspondant. Mode RPC alternatif pour la position : si `UsingRPC`, le serveur pousse la position via `QTrain_Server_CallSplineLocation` (`NetMulticast, Reliable`) au lieu du `OnRep`.

`NetDormancy = DORM_Awake`.

### `AQTrainSeatActor`
Propriétés répliquées (`DOREPLIFETIME`) : `OccupantPawn` (`ReplicatedUsing = OnRep_OccupantPawn`), `bSavedMovementModeForOccupant`, `SavedMovementMode`, `SavedCustomMovementMode`.

Modèle d'autorité :
- **Serveur** : `TrySeatPawn`/`TryStandPawn` exigent `HasAuthority()`. Ils appliquent l'état autoritaire (attache, `SetReplicateMovement(false)` pendant l'assise, sauvegarde/restauration du `MovementMode`) et forcent `ForceNetUpdate()`.
- **Client** : l'interaction est relayée au serveur via `ForwardClientInteractToServer`, qui cherche par réflexion `SV_Interact` sur un composant du `PlayerController` (le composant d'interaction content de QANGA). Le client n'applique jamais l'état autoritaire lui-même ; il réagit à `OnRep_OccupantPawn`.
- `OnRep_OccupantPawn` est le **seul** chemin d'application côté proxy : il compare `OccupantPawn` (répliqué) à `AppliedOccupantPawn` (transient, local) et joue `RestoreFromSeat`/`ApplySeatedState` en mode non-autoritaire.

Important : le mouvement du joueur dans QANGA est **client-authoritative** (`SmoothTSync`, plugin ClientAuthority). Le siège suspend/réactive ce composant ; sur serveur dédié sans mesh, c'est `SmoothTSync` qui applique le transform du client côté serveur — détail central du bug de sortie de siège (§7).

---

## 6. Points d'intégration

### Dépendances (consommées par QTrain)
- **`QPlatform`** (`PrivateDependencyModuleNames`, plugin requis) : `AQTrainSeatActor` manipule directement `UQPlatformComponent` du pawn (`QPlatform_IsAttached`, `QPlatform_ForceDetach`, `QPlatform_ForceCancelDetache`, `QPlatform_LockAttachement`, `QPlatform_PauseTracer`, `ApplyVelocityOnDetachement`) pour détacher/figer la plateforme pendant l'assise et la restaurer à la sortie. `AQTrain_Train` est la « plateforme » que le pawn chevauche (le siège remonte la chaîne d'attache via `ResolveSeatCarrierActor` qui cherche un `AQTrain_Train` parent).
- **`Niagara`** : effets de propulseurs (`UNiagaraComponent` créés dynamiquement, système `/Game/Systems/Vehicle/VFX/NS_HoverDust` pour la poussière d'atterrissage, systèmes « burner » nommés `QTrainThrusterBurner`). Purement cosmétique, côté client.
- **`Slate`/`SlateCore`** (listés dans le `Build.cs` mais sans usage de widget propre dans le code QTrain).
- **Contenu Blueprint résolu par chemin / réflexion** (couplage fort, non typé) :
  - `/Game/Systems/Interact/Interact_Interface` — interface d'interaction implémentée dynamiquement par le siège.
  - `/Game/Systems/Character/Data/Enums/ALS_MovementState` — enum ALS, valeurs `Sitting`/`Grounded` résolues par nom.
  - Fonctions pawn cherchées par nom : `BPI_Set_MovementState`, `BPI_Set_EmoteById`, et fallbacks `BPI_StandCharacter`/`BPI_StopSitCharacter`/`BPI_ExitSeat`.
  - Propriétés pawn cherchées par nom : `ReturnLocationSit`, `SittingSwitchLock`, `ALSMovementState`, `UpdateBaseMovement_Override` (sur le CMC).
  - Composant `SmoothTSync` (plugin ClientAuthority) trouvé par nom de classe ; fonction `SV_Interact` cherchée sur les composants du `PlayerController`.

### Dépendants (qui consomme QTrain)
Aucun code C++ de `G:/QANGA/Source` ni d'un autre plugin ne référence les types QTrain. L'intégration se fait **entièrement côté contenu** : les Blueprints dérivés de `AQTrain_Train`/`AQTrain_Spline_Track`/`AQTrain_Dock`/`AQTrainSeatActor` sont placés dans les niveaux, et le flux d'interaction passe par le système d'interaction content (`SV_Interact`). `UQTrain_SubSystem` est `BlueprintCallable` pour la sauvegarde/restauration des positions de trains.

### CVars (console)
- `qtrain.ThrusterDebug` (`int32`, défaut 0) — log throttlé de l'animation des propulseurs.
- `qtrain.ThrusterFxDebug` (`int32`, défaut 0) — log throttlé/diagnostic Niagara des propulseurs (état des émetteurs, scalabilité GPU/CPU).

Aucune `FAutoConsoleCommand` n'est définie par le plugin.

---

## 7. Gotchas, invariants et pièges

- **Le rail est serveur-only ; le train est répliqué.** Ne pas attendre que `AQTrain_Spline_Track` tourne côté client — son tick y est désactivé. Toute la logique vit sur le rail serveur ; le client ne voit que `AQTrain_Train` qui re-simule.
- **La position se réplique comme une entrée scalaire, pas comme un transform.** Le transform du train est recalculé sur chaque machine. Si le rail (`USplineComponent`) diffère entre serveur et client (spline non identique, échelle, points), les positions divergeront. Le rail doit être strictement identique partout.
- **`ReferenceName` (rail) ↔ `ReferenceID` (train) est la clé de liaison.** Le train retrouve son rail par `ReferenceID` dans le subsystem. Une faute de frappe → `ResolveReferenceTrack` échoue (log `LogTemp` Error) et le train ne bouge pas. Le défaut est `"Default"` des deux côtés.
- **En `OnlyVirtual`, l'acteur train est détruit, pas caché.** Tout ce qui tient un pointeur sur `Get_Train()` doit gérer le `nullptr`. Le passage `OnlyVirtual → RealTime` re-spawn un acteur neuf.
- **Mode async (`Use_VirtualAsync`)** : `SplineUpdate` tourne sur un thread worker ; ne rien y ajouter qui touche au monde/aux acteurs. Les positions joueurs sont capturées sur le GameThread **avant** le dispatch. Le tick acteur est désactivé pendant ce mode (`Disabled_TickOnVirtual`), d'où la nécessité de la `FCustomTickFunction` pour ré-évaluer le LOD.
- **Orientation planétaire.** `IsPlanetaryTrack` fait dériver l'« up » du train de `(Planet_OriginLocation - Location)`. Si `Planet_OriginLocation` est mal réglé (défaut = origine du monde), le train s'oriente vers le mauvais centre de gravité. Helper partagé : `MakeFromZX(gravity, tangent)`.
- **Siège : couplage par réflexion = fragile par conception.** `AQTrainSeatActor` dépend de chemins de contenu et de noms de fonctions/propriétés Blueprint exacts (ALS, interaction, `SmoothTSync`). Renommer un de ces symboles content casse silencieusement le siège (pas d'erreur de compilation C++). C'est volontaire pour réutiliser le pawn ALS de QANGA, mais c'est un invariant à respecter.
- **Sortie de siège = re-base CMC, jamais scene-attach.** `RestoreFromSeat`/`StabilizePawnOnSeatExit` rétablissent le **même** mode de transport que sans siège : une movement base CMC (based-movement) sur le porteur via `SetBaseFromFloor` + `MOVE_Walking`. Un scene-attach à la place désactiverait le CMC → le pawn ne se grounde jamais et tombe d'un train rapide. Ne pas « simplifier » en ré-attachant.
- **Serveur dédié = aucun mesh, par design.** Sur le serveur dédié Linux, `FindFloor` ne trouve jamais de sol de train (`StabilizePawnOnSeatExit` retourne `false` et no-op) ; le mouvement est client-authoritative (`SmoothTSync` applique le transform du client côté serveur). Toute logique de siège dépendant de la collision serveur est architecturalement insatisfaisable headless — voir l'historique mémoire `project_qtrain_seat_stand_dedicated_server`. La sortie de siège a fait l'objet de plusieurs itérations (handoff QPlatform, réactivation `SmoothTSync`, drag par scene-attach du train en mouvement) ; le chemin de stand sur serveur dédié passe par `CompleteExternalStandCleanup` (filet de sécurité du Tick), **pas** par `TryStandPawn`. `CompleteExternalStandCleanup` doit appeler le **même** `RestoreFromSeat(Pawn, true, true)` que `TryStandPawn` (ne pas dupliquer le restore en omettant `SetSeatPlatformSuspended(false)`, sinon `SmoothTSync` reste inactif et le pawn gèle au siège).
- **`SetIgnoreMoveInput` est un compteur engine.** `SetMoveInputLocked` est gardé par `bAppliedMoveInputLock` pour rester équilibré ; un verrou/déverrou non apparié latch le compteur côté client (état per-machine, non répliqué) → input bloqué jusqu'à reconnexion.
- **Le siège ne réplique l'état de mouvement que partiellement.** `OccupantPawn`, `SavedMovementMode`, `SavedCustomMovementMode`, `bSavedMovementModeForOccupant` sont des props `DOREPLIFETIME` **séparées et non atomiques** : sur un proxy, `OnRep_OccupantPawn` peut arriver avant que les `Saved*` soient cohérents. `AppliedOccupantPawn` (transient) absorbe les transitions coalescées.
- **`InteractionBounds` est en `ECC_Visibility = Overlap`, pas en collision physique.** C'est une cible de trace pour le système d'interaction, pas un bloqueur. La clearance de sortie, elle, fait un `OverlapMultiByChannel` avec la capsule du pawn à `ExitAnchor` et ignore le porteur de siège (et tout acteur attaché à lui).
- **VFX propulseurs : état global statique.** `GQTrainAnimatedThrusterStates` est une `TMap` globale au module indexée par `TWeakObjectPtr<AQTrain_Train>`. `ResetQTrainAnimatedThrusters` (appelé dans `EndPlay`/`BeginDestroy`) doit nettoyer l'entrée, sinon fuite de composants Niagara créés dynamiquement.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `QTrain.uplugin` | Manifeste : module `QTrain` (Runtime, Default), dépendances `Niagara` + `QPlatform`. |
| `Source/QTrain/QTrain.Build.cs` | Règles de module. Dépendances : `Core` (public) ; `CoreUObject`, `Engine`, `Niagara`, `QPlatform`, `Slate`, `SlateCore` (privé). |
| `Source/QTrain/Public/QTrain.h` / `Private/QTrain.cpp` | `FQTrainModule` (`IModuleInterface`). Startup/shutdown vides. |
| `Source/QTrain/Public/QTrain_Spline_Track.h` / `Private/QTrain_Spline_Track.cpp` | Rail : spline, simulation, LOD distance, spawn/destruction du train, warp, ease, tick custom. Définit `ETrainUpdateType`. |
| `Source/QTrain/Private/QTrain_Spline_Track_Dock.cpp` | Méthodes de docking du rail (`Set_WaitDocking`, `Checking_Docking`, `Set_ByPassDocking`) — partie de `AQTrain_Spline_Track`, fichier séparé. |
| `Source/QTrain/Public/QTrain_Train.h` / `Private/QTrain_Train.cpp` | Train répliqué : recalcul de position client/serveur, prédiction de ping, RPCs d'état, events Blueprint, proxy-mode, animation/VFX des propulseurs (~980 lignes cosmétiques en tête du `.cpp`). |
| `Source/QTrain/Public/QTrain_Dock.h` / `Private/QTrain_Dock.cpp` | Quai : sérialisation de l'accès au dock, file d'attente. |
| `Source/QTrain/Public/QTrain_SubSystem.h` / `Private/QTrain_SubSystem.cpp` | `UQTrain_SubSystem` (registre des rails) + `FQTrain_PosData`. (Dé)sérialisation des positions. |
| `Source/QTrain/Public/QTrainSeatActor.h` / `Private/QTrainSeatActor.cpp` | Siège : assise/lever, suspension/restauration du mouvement, interaction par réflexion, clearance de sortie, filet de sécurité Tick. |
