# FlyVehicleMovement — Architecture

Plugin de mouvement pour véhicules volants / à coussin d'air (hovercraft) de QANGA. Auteurs : Kathuz & RzZz. Module `Runtime` unique, `LoadingPhase: PreLoadingScreen`, version `1.4`. Tout le code décrit ici a été lu dans la source ; les sections sans support dans le code sont signalées comme telles.

## 1. But d'usage

QANGA est un open-world planétaire où le joueur pilote des engins volants (skimmers, motos volantes, vaisseaux) au-dessus de terrains à gravité variable, y compris des surfaces de planète courbes et des zones de gravité personnalisées. Le besoin concret que ce plugin résout :

- **Une simulation de hover/vol arcade à base de forces**, posée sur un corps rigide Chaos (le composant dérive de `UPawnMovementComponent` mais pilote un `UPrimitiveComponent` simulant la physique via `AddForce`/`AddTorque`, pas via le pawn movement classique). Le pilotage est centré sur la sensation : input lissé, banking, détection de marche arrière intelligente, modes boost/drift, limites de vitesse normale/boost.
- **Un système de coussin d'air multi-propulseurs** (`HovergroundThrusters`) qui suit le sol par traces de collision, avec un mode « réaliste » alternatif (`FRealisticHoverSettings`) orienté tenue de route urbaine (on-rails, anti-glissement en pente, auto-hold à basse vitesse).
- **Une intégration avec la gravité non-uniforme de QANGA** : gravité ponctuelle / planétaire / atmosphère portée par `FVehicleGravity`, et un sous-système monde (`UFlyVehicleWorldSubSystem`) qui applique une échelle de gravité globale par zones.
- **Du passage à l'échelle** : avec potentiellement de nombreux véhicules instanciés (trafic IA aérien via QAI/SpaceshipAI), le plugin embarque un cache de collision asynchrone round-robin, une LoD par distance (suppression Niagara/audio/widgets 3D) et une mise en sommeil physique des véhicules à l'arrêt (`SetSimulatePhysics(false)`).
- **Du multijoueur via plateformes mobiles** : un mode « platform ride » qui colle le véhicule à un ascenseur/plateforme animé côté serveur et pousse le résultat dans le composant `SmoothTSync` (plugin `ClientAuthority`) pour la réplication.

Ce n'est donc pas un « vehicle movement » générique UE : c'est un composant taillé pour des hovercraft à coussin d'air dans un monde à gravité planétaire, avec des optimisations de masse et un pont MP spécifique à QANGA.

## 2. Vue d'ensemble / décisions d'architecture

**Forme du système.** Un seul gros composant, `UVehicleMovementComponent`, concentre quasiment toute la logique. Il tick en `TG_PrePhysics`, ne s'auto-active pas (`bAutoActivate = false`, `bStartWithTickEnabled = false`) et n'applique des forces que si `RootPrimitiveComponent->IsSimulatingPhysics()`. Autour gravitent :

- un `UWorldSubsystem` (`UFlyVehicleWorldSubSystem`) pour la gravité globale et la liste d'ignore des traces ;
- une `UBlueprintFunctionLibrary` (`UFlyVehicleMovementBPLibrary`) pour l'accès Blueprint pratique ;
- des structures de configuration (`FRealisticHoverSettings`, `FHovergroundThruster`, `FVehicleGravity`, `FAsyncCollisionSettings`, etc.) ;
- un widget Slate de debug (`SFlyVehicleDebugWidget` + `STelemetryGraphWidget`) et un `UUserWidget` de drift (`UFlyVehicleDriftWidget`) ;
- une commande console (`FlyVehicleConsoleCommands.cpp`).

**Décisions clés observées dans le code.**

1. **Pilotage par forces sur le corps rigide, pas par `UPawnMovementComponent` classique.** Le composant lit la vitesse/transform du `RootPrimitiveComponent` et applique forces et couples. `CalcAndApplyMovement()` n'est appelé que quand le corps simule la physique.
2. **Deux modèles de hover coexistent.** Le hover « propulseurs » historique (`HovergroundThrusters`, `CalcHoverground*`) et le hover « réaliste » (`FRealisticHoverSettings.bUseRealisticHoverPhysics`, `RealisticHoverWheels`, `CalcRealisticHoverPhysics`). Le réaliste ajoute on-rails, anti-sliding en pente, auto-hold, gestion de pente, atterrissage prédictif.
3. **Optimisation de collision intégrée et activable.** `bEnableAsyncCollisionOptimization` + `FAsyncCollisionSettings` pilotent un cache temporel par propulseur (`FThrusterCollisionCache`), un ordonnancement round-robin (`ThrustersPerFrame`, `CurrentRoundRobinIndex`) et une fréquence dépendant de la distance caméra. Malgré le nom « async », la lecture du code montre des traces effectuées en immédiat (`PerformImmediateTraces`, `PerformSingleThrusterTrace`) avec mise en cache et échelonnement, pas des requêtes de scène réellement asynchrones — le gain vient du caching et du budget par frame, pas d'un thread de collision séparé.
4. **LoD et sommeil physique de masse.** Une LoD par distance au pawn joueur le plus proche (`UpdatePerformanceLoD`, ré-évaluée tous les `DistanceLoDUpdateInterval`) coupe Niagara (`NiagaraSuppressionDistance`), audio (`AudioSuppressionDistance`) et widgets monde 3D (`WidgetSuppressionDistance`). Un véhicule `Off`/`Broken` immobile assez longtemps (`SleepDelay`) voit son corps physique coupé (`SetSimulatePhysics(false)`) et ses auxiliaires « garés » (`SetAuxiliariesParkedForSleep`), pour un coût GameThread quasi nul. `SetVehicleState(On)` ré-arme tout.
5. **Pont MP par réflexion, sans dépendance de build.** Le « platform ride » trouve le composant `SmoothTSync` (`/Script/ClientAuthority.SmoothTSync`) par `FindObject`/`ProcessEvent` réflexion, pour éviter une dépendance de module sur `ClientAuthority`. Le plugin lui-même ne réplique aucune propriété (voir §5).

**Implémenté vs intention.** Le commentaire « async collision » est partiellement trompeur : il s'agit d'un cache + échelonnement, pas d'un async réel. Certains champs de debug (`bTestDustProperty`, `PIDError`/`PIDOutput` dans `FFlyVehicleDebugData`) sont des reliquats d'instrumentation, pas une boucle PID active. `FlyVehicleConsoleCommands.h` n'inclut que des en-têtes ; toute la commande vit dans le `.cpp`.

## 3. Carte des composants

| Élément | Type | Rôle |
| --- | --- | --- |
| `UVehicleMovementComponent` | UCLASS (`UPawnMovementComponent`, `Blueprintable`) | Cœur du plugin : hover, aérodynamique, input, forces, LoD, sommeil, platform ride, debug. |
| `UFlyVehicleWorldSubSystem` | UCLASS (`UWorldSubsystem`) | Gravité globale par zones (`AddGravityArea`/`RemoveGravityArea`), `GetCurrentGravityScale`, `HovergroundTraceIgnoreList`. |
| `UFlyVehicleMovementBPLibrary` | UCLASS (`UBlueprintFunctionLibrary`) | Helpers BP : `GetFlyVehicleMovementComponent`, ride height, drift/boost/brake, `SetRealisticHoverSettings`, `GetCurrentGlobalGravityScale`, `IsHovercraftBraking`. |
| `UFlyVehicleDriftWidget` | UCLASS (`UUserWidget`, `Blueprintable`) | HUD de drift : se lie au véhicule possédé, écoute `OnDriftModeChanged`, anime l'apparition/disparition. |
| `EVehicleState` | UENUM (`uint8`) | `Off`, `On`, `Broken`, `Destroyed`. |
| `EVehicleBehaviorAtOcean` | UENUM (`uint8`) | `Disabled`, `Float`, `Sink` — comportement à la surface de l'océan selon l'état. |
| `FHovergroundThruster` | USTRUCT | Configuration d'un propulseur de coussin d'air (position, rayon, `HoverForceAlpha`, offset par vitesse). |
| `FVehicleGravity` | USTRUCT | Données de gravité du véhicule : ponctuelle/planétaire/atmosphère, échelle, rayons². |
| `FRealisticHoverSettings` | USTRUCT | Réglage complet du hover « réaliste » : traces intelligentes, on-rails, anti-sliding, auto-hold, low-FPS protection, courbes, steering feel, limites de vitesse. |
| `FRealisticHoverWheel` | USTRUCT | Roue virtuelle de contact sol (position, rayon, demi-hauteur, normale/contact en sortie). |
| `FAsyncCollisionSettings` | USTRUCT | Budget de collision : distances haute fréquence/max, `ThrustersPerFrame`, `MaxCacheAgeMs`, cohérence temporelle. |
| `FThrusterCollisionCache` | USTRUCT | Entrée de cache par propulseur (hit, normale, validité, frame/temps, `IsExpired`). |
| `FGravityAreaInfo` | USTRUCT (non `BlueprintType`) | Entrée interne du sous-système : `TWeakObjectPtr<AActor>` + `GravityScale`. |
| `FFlyVehicleDebugData` | struct (non-UObject) | Snapshot d'instrumentation poussé au widget de debug (état, forces, historiques, données par propulseur). |
| `SFlyVehicleDebugWidget` | classe Slate (`SCompoundWidget`) | Overlay de debug à l'écran ; singleton (`GetInstance`/`CreateInstance`/`DestroyInstance`). |
| `STelemetryGraphWidget` | classe Slate (`SLeafWidget`) | Tracé des courbes de télémétrie (vitesse, altitude, hover power, ground effect). |
| `FFlyVehicleMovementModule` | `IModuleInterface` | Module runtime ; `StartupModule`/`ShutdownModule` vides. |
| `FlyVehicleConsoleCommands.cpp` | `FAutoConsoleCommand` | Commande console `HoverVehicle.ToggleDebug`. |

## 4. Flux de données et cycle de vie

**Initialisation.**
- Le module se charge en `PreLoadingScreen` (disponible tôt pour le spawn de véhicules). `FFlyVehicleMovementModule::StartupModule()` est vide ; la commande console s'enregistre via le static `FAutoConsoleCommand` dans `FlyVehicleConsoleCommands.cpp`.
- `UVehicleMovementComponent::BeginPlay()` : capture `World`, met en cache `CachedFlyVehicleSubSystem = World->GetSubsystem<UFlyVehicleWorldSubSystem>()`, puis `UpdateReferences()`, `InitializeDustEffects()`, `InitializeDefaultCollisionParams()`. Le composant n'auto-active pas et ne tick pas tant qu'il n'est pas explicitement activé / mis en état `On`.

**Runtime (TickComponent, `TG_PrePhysics`).** Chaque frame :
1. Met à jour `DeltaSecs`, `FrameCounter`, `RideHeightChangeTime`.
2. Si `bInPlatformRide` → return immédiat (le ride est piloté par un timer 60 Hz séparé, voir §5).
3. `UpdatePerformanceLoD()` (throttlé) — suppression distance de Niagara/audio/widgets.
4. Si état `Off`/`Broken` et corps immobile sous les seuils pendant `SleepDelay` → `SetSimulatePhysics(false)`, baisse le tick interval à `0.2 s`, `SetAuxiliariesParkedForSleep(true)`. Si déjà endormi → return.
5. Si le corps simule la physique → `CalcAndApplyMovement()` (la grosse chaîne : `UpdateValues` → `CalcGravityFrame` → input/aéro/hover/freinage → `CalcFinalMovement`/`CalcFinalAngularForce` → `ApplyResult` → `EndFrameReset`).
6. `UpdateDustEffects(DeltaTime)`, puis debug widget / debug draw si activés.

**Chemin hover.** Selon `RealisticHoverSettings.bUseRealisticHoverPhysics`, soit le hover propulseurs (`CalcHoverground`, `CalcHovergroundCopyGround[Async]`, cache `FThrusterCollisionCache` round-robin), soit le hover réaliste (`CalcRealisticHoverPhysics`, `RealisticHoverSteeringSystem`, `RealisticHoverUpdateVerticalForces`, `RealisticHoverAngularStabilisation`, `CalcRealisticHoverDrag`, anti-sliding/auto-hold/landing).

**Gravité.** `UFlyVehicleWorldSubSystem::RefreshVehiclesGravity` est déclenché par `AddGravityArea`/`RemoveGravityArea` (l'échelle est lue par réflexion sur la propriété `InitialGravityScale` de l'acteur de zone) : il parcourt les pawns, et pour chaque `UVehicleMovementComponent` non `bIsAircraftVehicle`, écrit `VehicleGravityData.IsInsideGravityArea`/`GravityScale`.

**Teardown.** Pas de logique de destruction spécifique côté module. Le composant nettoie les effets via `CleanupDustEffects`, et le widget Slate via `DestroyInstance`. Le platform ride libère son timer et ré-active la physique dans `EndPlatformRide()`.

## 5. Réplication / réseau

**Le plugin ne réplique aucune propriété et ne déclare aucun RPC.** Aucun `DOREPLIFETIME`, `GetLifetimeReplicatedProps`, `Replicated`, ni `UFUNCTION(Server/Client/Multicast)` n'existe dans la source. La simulation de mouvement est locale au corps rigide.

La synchronisation réseau passe par un autre système, intégré **par réflexion** dans le chemin « platform ride » (`BeginPlatformRide`/`UpdatePlatformRide`/`EndPlatformRide`) :

- Le composant cible la classe `SmoothTSync` du plugin `ClientAuthority` via `FindObject<UClass>(nullptr, TEXT("/Script/ClientAuthority.SmoothTSync"))` puis `GetComponentByClass`. Cela évite une dépendance de build sur `ClientAuthority`.
- **Modèle d'autorité.** Le ride est piloté côté serveur uniquement : `UpdatePlatformRide()` s'arrête si `!OwnerActor->HasAuthority()`. Le serveur snappe le véhicule à `RidingInitialRelativeTransform * PlatformTransform` chaque frame (timer 60 Hz, `ETeleportType::TeleportPhysics`), puis pousse le transform dans `SmoothTSync` par réflexion (`ForceTransform`, `SetServerTransformOverride`), avec `FlushNetDormancy()` + `ForceNetUpdate()`.
- **Pourquoi pas un attach/weld.** Commentaire du code : l'attach+weld était non fiable parce qu'UE désactive la physique au weld-vers-kinematic et que le spawner téléporte la plateforme (Sweep=false, Teleport=true) ; ni le push de collision ni la propagation du scene-graph n'atteignaient le corps welded en MP. D'où le snap explicite + push `SmoothTSync`.
- **Serveurs dédiés.** Le tick du composant et le ride autorisent `bAllowTickOnDedicatedServer = true`.

En résumé : la réplication du véhicule appartient à `SmoothTSync`/`ClientAuthority` ; ce plugin se contente, en mode ride, d'écrire l'autorité de transform dans ce composant.

## 6. Points d'intégration

**Dépendances du plugin (`FlyVehicleMovement.Build.cs`).**
- Public : `Core`.
- Private : `CoreUObject`, `Engine`, `Slate`, `SlateCore`, `UMG`, `Niagara`, `NiagaraCore`.
- `.uplugin` : dépend du plugin `Niagara`.
- Dépendance **soft** (réflexion, pas de build) : `SmoothTSync` du plugin `ClientAuthority` pour le platform ride.

**Systèmes QANGA qui dépendent de ce plugin.** Trois modules listent `FlyVehicleMovement` en dépendance de build et consomment `UVehicleMovementComponent` :
- **`QAI`** (`QAI.Build.cs`) — inclut `FlyVehicleMovementComponent.h` dans `QAI_MovementProcessor.cpp` (détection « ce pawn est-il un véhicule volant ? » via `FindComponentByClass<UVehicleMovementComponent>`) et dans `QSpaceshipMovementInterface.cpp` (résout le composant via `UVehicleMovementComponent::GetVehicleMovementComponent`, puis pilote le vaisseau IA en l'utilisant). C'est le chemin du trafic aérien IA.
- **`SpaceshipAI`** (module de `QPolice`) — `QSpaceshipPoliceProvider.cpp` configure le composant `FlyVehicleMovement` détecté par nom (`ConfigureSpaceshipMovement`).
- **`QPolice`** — déclare aussi la dépendance de module.

**Points d'entrée d'intégration côté API.**
- Résolution du composant : `UVehicleMovementComponent::GetVehicleMovementComponent(AActor*)` (résout aussi via une propriété `SpawnedVehicleRef` sur l'acteur source) et `UFlyVehicleMovementBPLibrary::GetFlyVehicleMovementComponent(AActor*)`.
- Action latente BP `WaitForVehicleMovementComponent` — attend que le composant et ses sphères de contact (`Collision_*`) soient prêts avant un `BeginPlatformRide` (remplace un `Delay 0.2` fragile du spawner de véhicules).
- Helpers de scene component par nom/sous-chaîne (`FindSceneComponentByName`, `FindSceneComponentByNameContains`) pour les spawners de terminaux.

**CVars / commandes console / réglages.**
- Commande console : **`HoverVehicle.ToggleDebug`** — bascule TOUT le debug (widget, draw, logging) sur le véhicule du joueur (ou le premier véhicule trouvé). Pilote le static `UVehicleMovementComponent::bGlobalDetailedDebugLogging`.
- Pas de CVar `IConsoleVariable` déclarée ; les réglages sont des `UPROPERTY` éditables (notamment `FRealisticHoverSettings`, `FAsyncCollisionSettings`, les distances de LoD `NiagaraSuppressionDistance`/`AudioSuppressionDistance`/`WidgetSuppressionDistance`, les seuils de sommeil `SleepLinearSpeedThreshold`/`SleepAngularSpeedThreshold`/`SleepDelay`).
- Zones de gravité : un acteur de zone exposant une propriété float `InitialGravityScale` est enregistré via `UFlyVehicleWorldSubSystem::AddGravityArea`.

## 7. Gotchas, invariants et pièges

- **Le tick ne fait rien sans physique active.** `CalcAndApplyMovement()` n'est appelé que si `RootPrimitiveComponent->IsSimulatingPhysics()`. Un véhicule sans corps simulant la physique (endormi, `Off`, sur plateforme) ne bouge pas via ce composant.
- **`SetVehicleState(On)` est la seule façon de réveiller un véhicule endormi.** L'auto-sommeil coupe `SetSimulatePhysics(false)`, baisse le tick à `0.2 s` et gare les auxiliaires. Tout passage à `On` ré-arme physique + tick + auxiliaires (`SetAuxiliariesParkedForSleep(false)`) et remet le tick interval à `0.0`. Ne pas remettre l'état à `On` laisse le véhicule figé.
- **`bIsAircraftVehicle` exclut le véhicule de l'échelle de gravité de zone** (`RefreshVehiclesGravity` saute ces composants) et change le comportement océan (`Float` même cassé). À régler en cohérence avec le type d'engin.
- **Le platform ride coupe la physique et prend l'autorité de transform.** Pendant un ride, `TickComponent` return immédiatement ; le mouvement est piloté par le timer 60 Hz `UpdatePlatformRide` côté serveur. Oublier `EndPlatformRide()` laisse la physique désactivée et le timer actif.
- **Le pont MP est par réflexion : échecs silencieux throttlés.** Si la classe/le composant/les fonctions `SmoothTSync` (`ForceTransform`, `SetServerTransformOverride`) ne sont pas trouvés, le code logue un warning une seule fois (`LogTemp`) et continue sans crasher. Renommer ces symboles dans `ClientAuthority` casse le ride sans erreur de compilation ici.
- **« Async collision » n'est pas réellement asynchrone.** Le système est un cache temporel + round-robin avec traces immédiates échelonnées. Ne pas attendre un thread de collision séparé ; le levier de perf est `ThrustersPerFrame`, `MaxCacheAgeMs` et la fréquence par distance caméra.
- **`PreviousResultTimes` est par-instance — ne pas le repasser en `static`.** Le commentaire dans l'en-tête documente un bug historique : un `static TArray` dans la fonction de hover faisait partager un seul tableau entre tous les véhicules (corruption sporadique quand deux véhicules avaient des comptes de propulseurs différents).
- **Le snapshot des sphères de contact attend `Collision_*` et `BeginPlay`.** `WaitForVehicleMovementComponent` / `BuildPlatformContactSnapshot` exigent que les `USphereComponent` nommés `Collision_*` soient enregistrés et que `HasActorBegunPlay()` soit vrai. Des noms différents ou un appel trop tôt font échouer le seating.
- **Le widget de debug est un singleton Slate.** `SFlyVehicleDebugWidget` partage une instance globale ; activer le debug sur plusieurs véhicules pointe le même overlay (`SetTargetComponent`).
- **Limites de vitesse à deux niveaux.** Limites globales (`NormalSpeedLimitKmH`/`BoostSpeedLimitKmH`) vs par-véhicule (`VehicleNormalSpeedLimitKmH`/`VehicleBoostSpeedLimitKmH`, où `0` = utiliser le global). Régler l'un sans comprendre l'autre donne des plafonds incohérents.

## 8. Fichiers et emplacements

| Fichier | Rôle |
| --- | --- |
| `FlyVehicleMovement.uplugin` | Manifeste : module runtime `PreLoadingScreen`, dépendance plugin `Niagara`, version `1.4`. |
| `Source/FlyVehicleMovement/FlyVehicleMovement.Build.cs` | Dépendances de module (Core / CoreUObject / Engine / Slate / SlateCore / UMG / Niagara / NiagaraCore). |
| `Public/FlyVehicleMovement.h` / `Private/FlyVehicleMovement.cpp` | `FFlyVehicleMovementModule` (startup/shutdown vides), `IMPLEMENT_MODULE`. |
| `Public/FlyVehicleMovementComponent.h` | Déclaration de `UVehicleMovementComponent`, `EVehicleState`, `EVehicleBehaviorAtOcean`, `FThrusterCollisionCache`, `FAsyncCollisionSettings`, `FRealisticHoverWheel`, stats. |
| `Private/FlyVehicleMovementComponent.cpp` | Implémentation : tick/forces, hover, LoD, sommeil, platform ride + pont `SmoothTSync` par réflexion, action latente. |
| `Public/FlyVehicleMovementTypes.h` | `FHovergroundThruster`, `FVehicleGravity`, `FRealisticHoverSettings`. |
| `Public/FlyVehicleWorldSubSystem.h` / `Private/FlyVehicleWorldSubSystem.cpp` | `UFlyVehicleWorldSubSystem` + `FGravityAreaInfo` : gravité globale par zones, ignore-list de traces. |
| `Public/FlyVehicleMovementBPLibrary.h` / `Private/FlyVehicleMovementBPLibrary.cpp` | `UFlyVehicleMovementBPLibrary` : helpers BP (résolution composant, ride height, drift/boost/brake, gravité). |
| `Public/FlyVehicleDebugWidget.h` / `Private/FlyVehicleDebugWidget.cpp` | `SFlyVehicleDebugWidget`, `STelemetryGraphWidget`, struct `FFlyVehicleDebugData`. |
| `Public/FlyVehicleDriftWidget.h` / `Private/FlyVehicleDriftWidget.cpp` | `UFlyVehicleDriftWidget` : HUD de drift lié au véhicule possédé. |
| `Public/FlyVehicleConsoleCommands.h` / `Private/FlyVehicleConsoleCommands.cpp` | Commande console `HoverVehicle.ToggleDebug`. |
| `CLAUDE.md` | Note d'orientation du plugin (vue d'ensemble, workflows de tuning). |
