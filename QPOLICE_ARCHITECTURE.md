# QPolice - Documentation Technique

## 1. Vue d'ensemble

QPolice est un plugin UE5 implementant un systeme complet de police et de niveaux de recherche pour QANGA. Il gere le cycle complet : signalement de crime, accumulation de points de recherche, escalade progressive sur 6 niveaux, spawn d'unites de police (drones au sol, unites autonomes, vaisseaux spatiaux), systeme de paiement d'amendes, et teleportation en prison. Le plugin s'integre profondement avec le systeme QAI pour le comportement des unites de police, GravityScape pour la gestion de la gravite sur mondes spheriques, et FlyVehicleMovement pour les vaisseaux.

Le plugin est compose de deux modules runtime : **QPolice** (coeur du systeme) et **SpaceshipAI** (IA des vaisseaux de police spatiaux).

**Etat actuel : le systeme est desactive par defaut** (`bPoliceSystemEnabled = false`). L'activation se fait via `UQPolice_DevSettings` (DefaultGame.ini) ou la CVar `qpolice.SystemEnabled`.

---

## 2. Architecture reelle

### Tableau des sous-systemes

| Sous-systeme | Fichiers | Statut | Role |
|---|---|---|---|
| **QPoliceSubsystem** | QPoliceSubsystem.h/cpp (~8200 lignes) | Complet mais monolithique | Coordinateur central, contient la majorite de la logique |
| **WantedSystemService** | Services/QWantedSystemService.h/cpp | Partiel | Gestion des niveaux de recherche et decay - sous-utilise, logique dupliquee dans le subsystem |
| **ProviderManagerService** | Services/QProviderManagerService.h/cpp | Complet | Orchestration des fournisseurs de reponse policiere (IModularFeature) |
| **RegistryService** | Services/QRegistryService.h/cpp | Complet | Registre des unites actives (police, combattants, temoins, porteurs d'armes) |
| **UiEventService** | Services/QUiEventService.h/cpp | Complet | Bus d'evenements UI, notifications, integration QNotification |
| **AsyncHelper** | QPoliceAsyncHelper.h/cpp | Complet | Chargement async de classes BP, mise a jour UI securisee |
| **AsyncLoadManager** | QAsyncLoadManager.h/cpp | Complet | UEngineSubsystem global de chargement d'assets async avec cache |
| **ConfigResolver** | QPoliceConfigResolver.h/cpp | Complet | Resolution sync/async des classes de vehicules depuis DataAssets |
| **PrisonSystem** | QPrisonSystemPrimaryAsset.h/cpp, QPrisonTeleporter.h/cpp | Complet | Base de donnees de prisons, teleportation |
| **PaymentSystem** | QPaymentArea.h/cpp, QPaymentZoneComponent.h/cpp | Complet | Zones de paiement d'amendes |
| **TaserSystem** | Components/QTaserComponent.h/cpp | Complet | Immobilisation par taser avec phases (charge, tir, hit, cooldown) |
| **TerritoryDetection** | Components/QTerritoryDetectionComponent.h/cpp | Complet | Detection de faction/territoire (Iclab, pirates, mercenaires) |
| **EnhancedTeleportation** | Components/QEnhancedTeleportationComponent.h/cpp | Complet | Teleportation phasee avec countdown, fade, amende |
| **SpaceshipPoliceProvider** | SpaceshipAI/QSpaceshipPoliceProvider.h/cpp | Complet | IPoliceResponseProvider pour vaisseaux spatiaux |
| **SpaceshipMovement** | SpaceshipAI/QSpaceshipMovementInterface.h/cpp | Complet | Interface de mouvement 3D vers FlyVehicleMovement |
| **ObstacleAvoidance** | SpaceshipAI/QSpaceshipObstacleAvoidanceComponent.h/cpp | Complet | Evitement d'obstacles par sphere traces async |
| **WeaponCoordinator** | SpaceshipAI/QWeaponCoordinatorComponent.h/cpp | Complet | Coordination d'armes (mitrailleuse, roquettes) avec effets Niagara |
| **SimpleProjectile** | SpaceshipAI/QSimpleProjectile.h/cpp | Complet | Projectile basique avec homing optionnel |
| **Interfaces** | Interfaces/*.h | Complet | 6 interfaces (PoliceUnit, AIConfigurable, WeaponCarrier, Combatant, Damageable, MovementConfigurable) |
| **InterfaceQueryLibrary** | QInterfaceQueryLibrary.h/cpp | Complet | Requetes d'interfaces sur acteurs pour BP |
| **Debug/CVars** | QPoliceDebugCVars.h/cpp | Complet | ~25 CVars de debug, profilage, visualisation |
| **DevSettings** | QPolice_DevSettings.h/cpp | Complet | UDeveloperSettings pour activation systeme |
| **PoliceLibrary** | QPoliceLibrary.h/cpp | Complet | API Blueprint facade (ReportCrime, GetWantedLevel, etc.) |

### Poids du code

Le fichier `QPoliceSubsystem.cpp` fait **~8200 lignes** et concentre une proportion disproportionnee de la logique (spawn, reponse, coordination, reseau, QAI, vehicules, amendes). Les services existent mais sont sous-utilises -- le subsystem execute directement une grande partie de ce que les services devraient gerer.

---

## 3. Pipeline d'execution

### Flux principal : Crime -> Reponse policiere

```
  Blueprint / Gameplay                     QPolice Core                                QAI
  =====================                    ==============                              ====

  UQPoliceLibrary::ReportCrime()
       |
       v
  UQPoliceSubsystem::ReportCrime()
       |
       +-- Verifie bPoliceSystemEnabled && CVar qpolice.SystemEnabled
       +-- Resout l'acteur criminel (GetCriminalEntity)
       +-- Calcule les points (PoliceConfig.CrimePoints[CrimeType])
       +-- Gere le cooldown AttackingPolice
       +-- Cree FCrimeData, ajoute aux RecentCrimes
       |
       v
  AddWantedPoints(Criminal, Points)
       |
       +-- Verifie IsWantedLevelProtected, IsPoliceAgent
       +-- Status.WantedPoints += Points (clamp MaxWantedPoints)
       +-- InvalidateWantedLevelCache
       |
       v
  UpdateWantedLevel(Player, Status)
       |
       +-- PointsToWantedLevel(Points) via WantedThresholds
       +-- Si level monte: BroadcastWantedLevelUpdate
       +-- OnWantedLevelChanged (BlueprintImplementableEvent)
       |
       v
  TriggerPoliceResponse(Player)
       |
       +-- Determine ResponseType selon WantedLevel:
       |     Suspect/Wanted  -> Drones
       |     Fugitive+       -> Mixed
       |     En vaisseau     -> SpaceCombat
       |
       v
  SpawnPoliceAI(Player, ResponseType, Location)
       |
       +-- GetDroneClassForWantedLevel() ou GetAIClassForResponseType()
       +-- Calcule DesiredAICount selon WantedLevel
       +-- ReuseExistingUnitsForPlayer() si possible
       |
       v
  SpawnOnePoliceUnit(Player, Type, AIClass, Index, Total, Location, Radius)
       |
       +-- FindNaturalSpawnTransform() ou ComputeSpawnTransformWithVariation()
       +-- ProcessEvent("ServerSpawnAI_TargetingActor") sur Lib_AI_Manager_C
       +-- CreatePoliceAgent(SpawnedPawn, ...)  --------+
       +-- Track dans ActiveSquads, UnitToPlayer        |
       +-- MC_CreateClientAgent (NetMulticast)          |
       +-- StartReinforcementTimer                      v
                                                   QAISystem->CreateAgent(AgentData, Pawn)
                                                        |
                                                        v
                                                   FQAI_AgentHandle stockee dans
                                                   ActorToAgentHandle / AgentHandleToActor
```

### Flux de decay

```
  AddWantedPoints -> StartDecayTimer
       |
       v
  [Timer: PoliceConfig.DecayDelay secondes]
       |
       v
  ProcessWantedDecay(Player)
       |
       +-- Verifie si police a LOS sur le joueur
       +-- Si oui: reset le timer, pas de decay
       +-- Si non: decremente WantedPoints selon DecayRate
       +-- Si bUseLevelDecay: decremente par paliers (LevelDecayInterval)
       +-- Si WantedPoints <= 0: ClearWantedLevel
       +-- Sinon: relance le timer
```

### Flux de paiement (drone captain)

```
  Drone Captain spawne -> OnCaptainPaymentZoneCreated
       |
       v
  UQPaymentZoneComponent::StartPaymentZone(TargetPlayer, CaptainDrone)
       |
       +-- TickComponent verifie proximite joueur
       +-- Si joueur entre dans la zone:
       |     -> UpdatePaymentProgress -> ProcessPayment
       |     -> ClearPlayerWantedLevel
       +-- Si joueur s'enfuit:
       |     -> EscapeTimer atteint timeout
       |     -> EscalateToLevel2 -> SwitchToTaserMode
       +-- Si joueur attaque le drone:
            -> OnCrimeReported -> EscalatePoliceAIToAggressive
```

### Flux reseau

```
  Serveur                                     Client
  ========                                    ======

  SpawnOnePoliceUnit()
       |
       +-- Spawn physique (ProcessEvent)
       +-- CreatePoliceAgent (serveur)
       +-- MC_CreateClientAgent ----RPC----> MC_CreateClientAgent_Implementation
       |                                         |
       |                                         +-- CreatePoliceAgent (client)
       |                                         +-- Traite PendingAgentTargets
       |
       +-- SetAgentTarget (serveur)
       +-- MC_SetAgentTarget -------RPC----> MC_SetAgentTarget_Implementation
       |                                         |
       |                                         +-- SetAgentTarget (client)
       |
  AddWantedPoints()
       +-- SendWantedClientUpdate --------> ProcessWantedLevelMessage
                                                |
                                                +-- Met a jour PlayerWantedStatus local
                                                +-- NotifyHuntedUIUpdated
```

---

## 4. Sous-systemes majeurs

### 4.1 QPoliceSubsystem (Coordinateur central)

**Fichiers :** `QPoliceSubsystem.h` (437 lignes), `QPoliceSubsystem.cpp` (~8200 lignes)

#### Architecture interne

Le subsystem est un `UWorldSubsystem` qui orchestre tout le systeme QPolice. Il est cree automatiquement par Unreal quand un World est initialise.

**Donnees principales :**
- `PlayerWantedStatus` : `TMap<TWeakObjectPtr<AActor>, FWantedStatus>` - etat de recherche par joueur
- `ActiveSquads` : `TMap<Player, TMap<ResponseType, FPlayerSquadState>>` - escouades actives par joueur et type
- `UnitToPlayer` : `TMap<TWeakObjectPtr<APawn>, TWeakObjectPtr<AActor>>` - mapping unite -> joueur cible
- `ActorToAgentHandle` / `AgentHandleToActor` : mapping bidirectionnel acteurs <-> QAI
- `ActivePoliceSpaceships` : vaisseaux de police actifs
- `WantedLevelCache` : cache mutable avec invalidation
- `RecentPoliceAttacks` : cooldown par paire (criminel, police attaquee)

**Timers :**
- `CleanupTimerHandle` : nettoyage des joueurs invalides toutes les 30s
- `ServiceTickTimerHandle` : tick des services toutes les 0.1s
- `VehicleTransitionCheckTimer` : detection transitions vehicule toutes les 0.5s
- Par joueur : `DecayTimer`, `EscalationTimer`, `ReinforcementTimer`

#### Patterns notables

**Spawn d'IA via ProcessEvent sur Blueprint :** Le spawn de drones utilise `ProcessEvent` sur le CDO de `Lib_AI_Manager_C` pour appeler `ServerSpawnAI_TargetingActor`. Cela couple le systeme C++ a un Blueprint specifique.

```cpp
// QPoliceSubsystem.cpp:2964-2997
UClass* LibAIManagerClass = StaticLoadClass(UObject::StaticClass(), nullptr,
    TEXT("/Game/Systems/AI/Lib_AI_Manager.Lib_AI_Manager_C"));
UFunction* ServerSpawnFunction = LibAIManagerClass->FindFunctionByName(
    TEXT("ServerSpawnAI_TargetingActor"));

struct FServerSpawnAIParams { /* ... */ };
FServerSpawnAIParams Params;
CDO->ProcessEvent(ServerSpawnFunction, &Params);
```

**Spawn naturel hors champ de vision :** `FindNaturalSpawnTransform` utilise un systeme de scoring pour trouver un emplacement de spawn hors du champ de vision du joueur, avec des candidats multiples evalues par score (hors vue, occlusion, hauteur).

**Detection de vaisseau par nom de classe :** `IsSpaceshipLike` utilise du string matching sur les noms de classes (`Spaceship`, `SpaceShip`, `Spacecraft`, `Ship`) et les tags.

#### Interactions

- **QAI** : `CreatePoliceAgent` cree des agents QAI avec `FQAI_PoliceAgent_Data` (drone config, autonomous config, spaceship config)
- **GravityScape** : `ResolveUpVector` et `AdjustLocationForGround` utilisent le subsystem de gravite pour les mondes spheriques
- **WorldScape** (optionnel) : `WS_SingleProject` pour projeter les positions de spawn sur la surface
- **ClientAuthority** : inclus mais utilisation non directement visible dans le subsystem

#### Pieges

1. **Le systeme est OFF par defaut** - `bPoliceSystemEnabled` est `false` dans le constructeur de `FPoliceConfig`, et `UQPolice_DevSettings::bEnablePoliceSystem` est `false`. Toute operation verifie ces deux flags.
2. **Chemins hardcodes** - Plusieurs chemins Blueprint sont codes en dur (Lib_AI_Manager, classes de drones, classes de vaisseaux).
3. **Variables statiques locales dans TickServices** - `LastPatrolAssignmentCheck` et `LastDistanceCleanupCheck` sont `static` dans la fonction, donc partagees entre toutes les instances (probleme en PIE multi-monde).
4. **Le fichier fait 8200+ lignes** - Refactoring necessaire; la logique de spawn, de reponse, de QAI et de reseau devrait etre distribuee dans les services.

---

### 4.2 Systeme de niveaux de recherche (Wanted)

**Fichiers :** `QPoliceTypes.h` (types), `QPoliceSubsystem.cpp` (logique), `Services/QWantedSystemService.h/cpp`

#### Les 6 niveaux

| Niveau | Enum | Seuil par defaut | Reponse |
|---|---|---|---|
| 0 | NotWanted | 0 | Aucune |
| 1 | Suspect | 15 | 1 drone captain (paiement) |
| 2 | Wanted | 30 | 2 drones (taser + attaque) |
| 3 | Fugitive | 60 | Mixte (drones + autonomes) |
| 4 | PublicEnemy | 120 | Reponse lourde |
| 5 | MostWanted | 300 | Maximum (max: 350 points) |

#### Points par type de crime

```
Trespassing: 5, Assault: 10, PropertyDamage: 8,
AttackingPolice: 25, AttackingCivilian: 15, Theft: 12, Murder: 50
```

#### Decay

- Delai avant debut du decay : `DecayDelay` (60s par defaut)
- Annule si un policier a une ligne de vue sur le joueur
- Decremente par `DecayRate` (1 point/tick)
- Mode decay par palier : `bUseLevelDecay` / `LevelDecayInterval` (20s) - decremente d'un niveau a la fois

#### Protection de niveau

`SetWantedLevelProtection` bloque temporairement l'augmentation du wanted level (utilise apres un paiement d'amende pour eviter une re-escalade immediate). Dure 5 secondes via `LastWantedClearTime`.

---

### 4.3 Systeme de fournisseurs de reponse (Provider)

**Fichiers :** `IPoliceResponseProvider.h`, `Services/QProviderManagerService.h/cpp`, `SpaceshipAI/QSpaceshipPoliceProvider.h/cpp`

#### Architecture

Le systeme utilise le pattern **Strategy** via `IModularFeature` d'Unreal :

```
IPoliceResponseProvider (IModularFeature)
    |
    +-- GetSupportedResponseTypes()     // Quels types cette impl. gere
    +-- CanHandleResponseType()         // Peut-on gerer ce type ?
    +-- SpawnPoliceUnit()               // Spawn effectif
    +-- ConfigurePoliceUnit()           // Configuration post-spawn
    +-- CleanupPoliceUnit()             // Nettoyage
    +-- GetMaxUnitsForResponse()        // Limites par niveau
    +-- GetPriority()                   // Priorite entre providers
    +-- GetStableProviderId()           // Identifiant stable
```

**FQSpaceshipPoliceProvider** est l'implementation concrete pour les vaisseaux spatiaux. Elle :
- S'enregistre comme IModularFeature au demarrage du module SpaceshipAI
- Gere `SpacePatrol` et `SpaceCombat`
- Cache les classes `LavrikPolice` et `VelkaraPolice`
- Spawn via `SpawnSpaceshipPawn`, configure le mouvement et les armes
- Priorite : 100

**UQProviderManagerService** :
- Ecoute les enregistrements/desenregistrements de IModularFeature
- Gere les overrides de priorite (CVars, code)
- Trie les providers par priorite effective

#### Point d'extension

Pour ajouter un nouveau type de reponse (ex: faction pirate) :
1. Implementer `IPoliceResponseProvider`
2. S'enregistrer via `IModularFeatures::Get().RegisterModularFeature(IPoliceResponseProvider::ModularFeatureName, this)`
3. Le `ProviderManagerService` le detectera automatiquement

---

### 4.4 Systeme de paiement d'amendes

**Fichiers :** `QPaymentArea.h/cpp`, `Components/QPaymentZoneComponent.h/cpp`

#### Deux implementations

1. **AQPaymentArea** (heritage de AQPoliceInteractionArea) - Acteur autonome spawne dans le monde, avec sphere de detection, decal, et timer de paiement.

2. **UQPaymentZoneComponent** - Composant attache au drone captain, avec machine a etats :

```
Inactive -> WaitingForPayment -> PaymentInProgress -> PaymentCompleted
                  |                                        |
                  +---- PlayerEscaped <---------(timeout)--+
```

Le drone captain est spawne au niveau Suspect. Il cree une zone de paiement autour de lui. Si le joueur entre dans la zone et reste dedans pendant `PaymentTime` (3s), l'amende est payee et le wanted level est efface. Si le joueur fuit, un timer d'evasion demarre. A expiration, le systeme escalade vers le niveau 2 et passe en mode taser.

**Calcul d'amende :** Base sur les points de recherche du joueur, avec un minimum de 100.

---

### 4.5 Systeme de taser

**Fichiers :** `Components/QTaserComponent.h/cpp`

Machine a etats :

```
Inactive -> Charging -> Firing -> Hit -> Immobilizing -> Cooldown -> Inactive
                          |
                          +---> Miss -> Cooldown -> Inactive
```

**Configuration (FTaserConfig) :**
- Portee : 1500
- Temps de charge : 2s
- Duree immobilisation : 5s
- Cooldown : 10s
- Portee de precision : 500 (au-dela, probabilite de miss)

Le taser desactive l'input du joueur pendant l'immobilisation, puis teleporte le joueur en prison via `TeleportPlayerToPrison`. Les effets visuels utilisent Niagara (beam, impact, charge).

---

### 4.6 Systeme de territoire et factions

**Fichiers :** `Components/QTerritoryDetectionComponent.h/cpp`

#### Types de territoire

```
ETerritoryType: Unknown, IclabControlled, PirateTerritory, MercenaryZone,
                PrivateSecurity, LawlessZone, NeutralSpace
```

La detection utilise trois methodes en cascade :
1. **Volumes** - Detection par collision avec des volumes de territoire
2. **Noms de niveau** - Matching de mots-cles dans le nom du level (ex: "City" -> Iclab)
3. **Tags d'acteurs** - Tags sur les acteurs marqueurs de territoire

Selon le territoire, les crimes sont filtres differemment : en zone pirate, seul `AttackingPolice` declenche une reponse. En zone Iclab, tous les crimes sont signales. En zone lawless, rien n'est signale.

---

### 4.7 Systeme de prison

**Fichiers :** `QPrisonTeleporter.h/cpp`, `QPrisonSystemPrimaryAsset.h/cpp`, `QPrisonDatabaseTypes.h`, `Components/QEnhancedTeleportationComponent.h/cpp`

#### Base de donnees de prisons

`UQPrisonSystemPrimaryAsset` est un `UPrimaryDataAsset` qui maintient une liste de `FPrisonDatabaseEntry`. Le scan des prisons est execute depuis l'editeur via `ScanAllLevelsForPrisons()` qui :
- Parcourt tous les levels du projet
- Cherche les acteurs `AQPrisonTeleporter` (y compris dans les sous-niveaux QLevel recursifs)
- Extrait les transformations en espace monde

#### Teleportation phasee

`UQEnhancedTeleportationComponent` implemente une teleportation en phases :

```
Inactive -> Countdown (5s) -> FadeOut (1.5s) -> Processing (2s) -> Transport -> FadeIn (1.5s) -> Completed
```

Pendant la teleportation :
- L'input du joueur est desactive
- Une amende est calculee et deduite
- Le joueur est teleporte au point de spawn le plus proche de la prison
- Le wanted level est efface

---

### 4.8 Module SpaceshipAI

**Fichiers :** `Source/SpaceshipAI/` (8 headers, 6 cpps, 1 Build.cs)

#### FQSpaceshipPoliceProvider

Implementation de `IPoliceResponseProvider` pour les reponses spatiales. Enregistre automatiquement via le module SpaceshipAI au startup.

- Cache les classes de vaisseaux (Lavrik, Velkara)
- Spawn via `SpawnSpaceshipPawn`
- Configure mouvement et armes post-spawn
- Setup QAI agent avec `DelayedQAIAgentSetup` (timer 0.5s pour attendre la replication)

#### UQSpaceshipMovementInterface

Composant qui traduit les ordres de navigation en inputs pour le systeme FlyVehicleMovement :

- `MoveToLocation(Target)` : calcule la direction, applique les inputs vehicule
- `OrientToTarget(Actor)` : oriente le vaisseau vers un acteur
- `MaintainAltitude(Altitude)` : maintient une altitude cible avec hysteresis
- `PerformEvasiveManeuvers()` : manoeuvres d'evasion aleatoires
- Detecte les crashs et lance le cleanup automatique (destroy apres delai)

#### UQSpaceshipObstacleAvoidanceComponent

Evitement d'obstacles par sphere traces :

- Traces avant et peripheriques (angle configurable, 20 deg par defaut)
- Traces async pour eviter les stalls
- Distance de trace proportionnelle a la vitesse (`ReactionTime * Speed`)
- Calcul de waypoints temporaires pour le contournement
- Smoothing de la direction d'evitement

**Configuration clef :**
```
ReactionTime: 2.0s, MinTrace: 10000, MaxTrace: 100000, TraceRadius: 300
AvoidanceStrength: 1.0, UpdateFrequency: 5Hz
```

#### UQWeaponCoordinatorComponent

Coordonne les systemes d'armes du vaisseau :

**Mode Direct Fire :**
- **MachineGun** : hitscan avec spread, tracers Niagara, gate d'alignement (30 deg)
- **Rocket** : projectile (`AQSimpleProjectile`) avec homing optionnel, degats de zone

Decouvre les muzzles par convention de nommage (composants contenant "MachineGun" ou "Rocket"), ou par composants d'armes enregistres. Les tracers sont repliques via `Multicast_SpawnMGTracer` (NetMulticast, Unreliable).

---

### 4.9 Systeme de chargement asynchrone

**Fichiers :** `QAsyncLoadManager.h/cpp`, `QPoliceConfigResolver.h/cpp`, `QPoliceAsyncHelper.h/cpp`

#### UQAsyncLoadManager (UEngineSubsystem)

Subsysteme moteur global gerant le chargement asynchrone d'assets :

- Cache d'assets charges (`AssetCache`)
- Suivi des chargements en cours (`ActiveLoads`)
- Nettoyage periodique du cache (30s interval, 300s max age)
- Templates C++ pour `LoadClassAsync<T>` / `LoadObjectAsync<T>`
- Preload configurable (EssentialAssets, ResponseAssets, UIAssets)

#### UQPoliceConfigResolver

Bibliotheque de fonctions pour resoudre les configurations vehicules depuis `UQPoliceConfigDataAsset` :
- Resolution synchrone (`ResolvePawnAndController`)
- Resolution asynchrone (`ResolvePawnAndControllerAsync`)
- Preload latent (`PreloadVehicleType`) pour Blueprints

#### UQPoliceAsyncHelper

Encapsule les operations async courantes du systeme police :
- `SafeUpdateHuntedUI` : mise a jour UI avec chargement async du widget
- `SafeExecuteAITargeting` : ciblage IA securise
- `SafeExecuteTeleport` : teleportation securisee
- Chemins hardcodes vers des assets BP (HuntedWidget, AIManager, TeleportLib)

---

### 4.10 Systeme de debug

**Fichiers :** `QPoliceDebugCVars.h/cpp`

~25 variables console organisees par categorie :

| Prefixe | Exemples | Usage |
|---|---|---|
| `qpolice.Debug.*` | WantedSystem, Providers, Registry, UIEvents, SpaceshipAI | Log par sous-systeme |
| `qpolice.Profile.*` | WantedSystem, Providers, Registry, Interfaces | Profilage par sous-systeme |
| `qpolice.Draw.*` | PoliceUnits, WantedActors, ObstacleTraces, AvoidanceVectors | Visualisation debug 3D |
| `qpolice.Stats.*` | Collect, UpdateInterval, Log, MaxEventHistory | Statistiques |
| `qpolice.` | Debug, Verbose, SafeMode, SystemEnabled, EnableSpace | Controles globaux |

Les macros `QPOLICE_DEBUG_LOG`, `QPOLICE_DEBUG_DRAW_*` sont conditionnellement compilees (strippees en Shipping).

**Attention :** `QPOLICE_STAT_INCREMENT` et `QPOLICE_STAT_SET` sont des **stubs vides** - les corps des macros ne font rien meme quand `GetCollectStats()` retourne true.

---

## 5. Architecture reseau

### Autorite

Le systeme est **server-authoritative** :
- Le serveur gere les wanted levels, le spawn des unites, et la logique de reponse
- Les clients recoivent les mises a jour via :
  - `MC_CreateClientAgent` (NetMulticast, Reliable) - creation d'agent QAI cote client
  - `MC_RemoveClientAgent` (NetMulticast, Reliable) - suppression d'agent
  - `MC_SetAgentTarget` (NetMulticast, Reliable) - changement de cible
  - `SendWantedClientUpdate` -> `ProcessWantedLevelMessage` - sync du wanted level via message string

### Flux de synchronisation du wanted level

```
Serveur: AddWantedPoints() -> SendWantedClientUpdate()
    |
    +-- Pour chaque PlayerController client:
         +-- Trouve le pawn equivalent sur le serveur
         +-- Formate le message "WantedLevel:{level}:{points}"
         +-- Envoie via ClientAuthorityComponent ou broadcast direct

Client: ProcessWantedLevelMessage("WantedLevel:2:35")
    |
    +-- Parse le message
    +-- Met a jour PlayerWantedStatus local
    +-- Notifie UI
```

### Replication des unites

Les unites de police sont spawnees sur le serveur et repliquees normalement par Unreal. La creation de l'agent QAI correspondant est declenchee cote client via le RPC `MC_CreateClientAgent`. Un systeme de `PendingAgentTargets` gere le cas ou la cible est assignee avant que l'agent client n'existe.

---

## 6. Structures de donnees de reference

### Enums principales

```cpp
// 6 niveaux de recherche
enum class EWantedLevel : uint8 {
    NotWanted, Suspect, Wanted, Fugitive, PublicEnemy, MostWanted
};

// 9 types de crimes
enum class ECrimeType : uint8 {
    None, Trespassing, Assault, PropertyDamage, AttackingPolice,
    AttackingCivilian, Theft, Murder, Custom
};

// 6 types de reponse
enum class EPoliceResponseType : uint8 {
    None, Drones, Autonomous, Mixed, SpacePatrol, SpaceCombat
};

// Etats de combat
enum class EQCombatState : uint8 {
    Neutral, Defensive, Aggressive, Retreating, Disabled
};

// Etats du taser
enum class ETaserState : uint8 {
    Inactive, Charging, Firing, Hit, Immobilizing, Cooldown
};

// Etats de la zone de paiement
enum class EPaymentZoneState : uint8 {
    Inactive, WaitingForPayment, PaymentInProgress, PaymentCompleted, PlayerEscaped
};

// Phases de teleportation
enum class ETeleportPhase : uint8 {
    Inactive, Countdown, FadeOut, Processing, Transport, FadeIn, Completed, Failed
};

// Types de territoire
enum class ETerritoryType : uint8 {
    Unknown, IclabControlled, PirateTerritory, MercenaryZone,
    PrivateSecurity, LawlessZone, NeutralSpace
};

// Factions de reponse
enum class EFactionResponseType : uint8 {
    None, IclabPolice, PirateFaction, MercenaryGuild,
    PrivateCorps, LocalMilitia, CustomFaction
};

// Etats IA vaisseau
enum class ESpaceshipAIState : uint8 {
    Idle, Patrol, Hunt, Combat, Evade, Return
};
```

### Structs principales

```cpp
// Etat de recherche d'un joueur
struct FWantedStatus {
    EWantedLevel WantedLevel;      // Niveau actuel
    int32 WantedPoints;            // Points accumules
    float DecayStartTime;          // Debut du decay
    bool bInCustody;               // En detention
    TArray<FCrimeData> RecentCrimes; // 10 derniers crimes
    EPoliceResponseType ActiveResponse; // Reponse active
};

// Configuration globale du systeme police
struct FPoliceConfig {
    TMap<EWantedLevel, int32> WantedThresholds;  // Points par niveau
    TMap<ECrimeType, int32> CrimePoints;          // Points par crime
    float DecayDelay = 60.0f;                     // Delai avant decay
    float DecayRate = 1.0f;                       // Vitesse de decay
    float SurrenderTime = 5.0f;                   // Temps pour se rendre
    int32 MaxWantedPoints = 350;                  // Plafond de points
    float AttackingPoliceCooldown = 3.0f;         // Cooldown entre reports
    float DroneAttackCooldown = 5.5f;             // Cooldown attaque drone
    float DroneAttackTurnDuration = 4.0f;         // Duree d'un tour d'attaque
    bool bEnableDroneCoordination = true;         // Coordination entre drones
    bool bPoliceSystemEnabled = false;            // SYSTEME DESACTIVE PAR DEFAUT
};

// Escouade active pour un joueur
struct FPlayerSquadState {
    int32 DesiredCount;                    // Nombre desire d'unites
    TArray<TWeakObjectPtr<APawn>> Units;   // Unites assignees
    float LastReinforcementCheck;           // Dernier check de renforts
};
```

---

## 7. Points d'attention

### 7.1 Subtilites

1. **Double flag d'activation** - Le systeme verifie `PoliceConfig.bPoliceSystemEnabled` ET `QPoliceDebugCVars::GetSystemEnabled()`. Les deux doivent etre `true`. Le DevSettings ecrit dans les deux au demarrage.

2. **GetCorrectPlayerActor** - La bibliotheque BP contient un hack qui detecte quand un acteur passe est un "ALS_Base_CharacterBP" ou un vaisseau de police, et le remplace par le pawn du premier PlayerController. Cela compense des bugs cote Blueprint qui passent le mauvais acteur.

3. **Wanted level transfer vehicule** - Quand un joueur entre/sort d'un vaisseau, `CheckAndHandleWantedLevelTransfer` transfere le wanted level entre le pawn precedent et le nouveau pawn. `GetCriminalEntity` resout toujours vers le joueur meme s'il est dans un vehicule.

4. **ProcessEvent pour le spawn** - Le spawn d'IA passe par un Blueprint (`Lib_AI_Manager_C::ServerSpawnAI_TargetingActor`) via `ProcessEvent`. Cela signifie que si ce BP est renomme, deplace, ou modifie, le spawn casse silencieusement.

5. **Spawn hors vue** - `FindNaturalSpawnTransform` essaie de placer les unites derriere le joueur ou derriere des occluders. Le systeme de scoring favorise les positions hors du champ de vision.

6. **Mondes spheriques** - Toute la logique de spawn et de positionnement utilise `GravityScape` et optionnellement `WorldScape` pour gerer l'orientation sur des planetes. Le vecteur "up" est toujours resolu dynamiquement.

7. **Deduplication des logs** - Le namespace `QPoliceWantedLogging` maintient des `TSet` de cles de log pour eviter les messages repetitifs. Les cles expirent quand les acteurs deviennent invalides.

### 7.2 Limitations connues

1. **Pas de persistance entre sessions** - Le wanted level est en memoire uniquement, pas sauvegarde.
2. **Pas de multi-faction simultane** - Le TerritoryDetectionComponent detecte un territoire mais le systeme de reponse reste celui de QPolice (pas de providers de faction differents actifs en meme temps).
3. **Static locals dans TickServices** - Les variables `LastPatrolAssignmentCheck` et `LastDistanceCleanupCheck` sont `static`, donc partagees entre toutes les instances de subsystem (problematique en PIE avec multi-world).
4. **Stat macros vides** - `QPOLICE_STAT_INCREMENT` et `QPOLICE_STAT_SET` ne font rien meme quand le profilage est active.
5. **Nettoyage de commandes console fantome** - `QPoliceDebugCVars::Cleanup()` tente de desenregistrer `qpolice.PrintSettings`, `qpolice.EnableAllDebug`, `qpolice.DisableAllDebug` qui ne sont jamais enregistrees dans `Initialize()`.

### 7.3 Hacks et dette technique

| Fichier | Description | Raison |
|---|---|---|
| `QPoliceModule.cpp:22-37` | Suppression du callback `OnAssetLoaded` de `ContentBrowserAssetDataSource` | Workaround pour un crash UE 5.7 dans `TryCacheClass` |
| `QPoliceLibrary.cpp:10-11` | Variables statiques globales `LastCrimeReportTime` et `CleanupCounter` | Anti-spam de reports de crime, devrait etre dans le subsystem |
| `QPoliceLibrary.cpp:355-383` | `GetCorrectPlayerActor` avec string matching sur noms d'acteurs | Contourne des bugs BP qui passent le mauvais acteur |
| `QPoliceSubsystem.cpp:232-282` | `IsSpaceshipLike` avec matching de noms de classes | Detection heuristique de vaisseaux sans interface formelle |
| `QPoliceSubsystem.cpp:2938` | Chemin hardcode `/Game/AI/Drone/AI_DronePolice.AI_DronePolice_C` | Devrait utiliser le DataAsset de config |
| `QPoliceSubsystem.cpp:2964` | Chemin hardcode `/Game/Systems/AI/Lib_AI_Manager.Lib_AI_Manager_C` | Couplage BP/C++ fragile via ProcessEvent |
| `QPoliceSubsystem.cpp:3006` | `TakeDamage(0.0f, ...)` pour declencher l'aggression de l'IA | Hack pour activer le comportement hostile via le systeme de dommages |
| `QPoliceSubsystem.cpp:7294-7316` | Variables `static` dans `TickServices` | Devrait etre des membres d'instance |
| `QPoliceDebugCVars.cpp:292-297` | Desenregistrement de commandes non enregistrees | Desynchronisation entre Initialize/Cleanup |

---

## 8. Arborescence complete des fichiers

```
QPolice.uplugin                                    -- Definition du plugin (2 modules: QPolice, SpaceshipAI)

Source/QPolice/
    QPolice.Build.cs                               -- Regles de build, deps (QAI, GravityScape, FlyVehicleMovement, etc.)
    Public/
        QPolice.h                                  -- Header umbrella (includes tous les types publics)
        QPoliceTypes.h                             -- Enums (EWantedLevel, ECrimeType, EPoliceResponseType), structs (FCrimeData, FWantedStatus, FPoliceConfig), delegates
        QPoliceModule.h                            -- Declaration du module, categorie de log LogQPolice
        QPoliceSubsystem.h                         -- UWorldSubsystem central (~437 lignes), FPlayerSquadState, FWantedLevelCache
        QPoliceLibrary.h                           -- UBlueprintFunctionLibrary facade (ReportCrime, GetWantedLevel, etc.)
        QPoliceConfigDataAsset.h                   -- UDataAsset de config vehicules (FQMovementTuning, FQCombatTuning, FQWantedCaps, FQPoliceVehicleConfig)
        QPoliceConfigResolver.h                    -- Resolution sync/async de classes depuis DataAssets
        QPoliceAsyncHelper.h                       -- Helper d'operations async (UI, targeting, teleport)
        QAsyncLoadManager.h                        -- UEngineSubsystem global de chargement async avec cache
        QPoliceDebugCVars.h                        -- Namespace de CVars debug, macros QPOLICE_DEBUG_LOG/DRAW
        QPolice_DevSettings.h                      -- UDeveloperSettings (bEnablePoliceSystem)
        QPoliceInteractionArea.h                   -- Acteur base pour zones d'interaction (sphere, decal, proximite)
        QPaymentArea.h                             -- Zone de paiement d'amende (herite QPoliceInteractionArea)
        QPrisonDatabaseTypes.h                     -- FPrisonDatabaseEntry (nom, location, capacite, spawn points)
        QPrisonSystemPrimaryAsset.h                -- UPrimaryDataAsset base de donnees de prisons, scan de niveaux
        QPrisonTeleporter.h                        -- Acteur teleporteur de prison (spawn points, teleportation)
        IPoliceResponseProvider.h                  -- Interface IModularFeature + manager statique FPoliceResponseProviderManager
        QInterfaceQueryLibrary.h                   -- Requetes d'interfaces sur acteurs (FindWeaponCarrier, FindCombatant, etc.)
        Interfaces/
            QPoliceUnit.h                          -- IQPoliceUnit (assignment, target, active, duty)
            QPoliceAIConfigurable.h                -- IQPoliceAIConfigurable + FQPoliceAIConfig (aggression, patrol, detection)
            QWeaponCarrier.h                       -- IQWeaponCarrier (HasWeaponType, FireAt, GetWeaponRange)
            QCombatant.h                           -- IQCombatant + EQCombatState (target, threat, combat state)
            QDamageable.h                          -- IQDamageable + FQDamageInfo (health, shield, damage resistance)
            QMovementConfigurable.h                -- IQMovementConfigurable + FQMovementConfig (speed, accel, turn, brake)
        Components/
            QPaymentZoneComponent.h                -- Composant sphere de paiement sur drone captain (machine a etats)
            QTaserComponent.h                      -- Composant taser (charge, tir, immobilisation, cooldown, effets Niagara)
            QTerritoryDetectionComponent.h         -- Detection de territoire/faction (volume, nom de level, tags)
            QEnhancedTeleportationComponent.h      -- Teleportation phasee (countdown, fade, transport, amende)
        Services/
            QProviderManagerService.h              -- Orchestration des IPoliceResponseProvider via IModularFeature
            QRegistryService.h                     -- Registre d'acteurs (police, combattants, temoins, armes)
            QWantedSystemService.h                 -- Service wanted level (decay, calcul, events)
            QUiEventService.h                      -- Bus d'evenements UI (notifications, wanted, crimes, payment)
    Private/
        QPoliceModule.cpp                          -- Startup/Shutdown, fix crash UE 5.7 ContentBrowserAssetDataSource
        QPoliceSubsystem.cpp                       -- ~8200 lignes, coeur du systeme (spawn, reponse, QAI, reseau, etc.)
        QPoliceLibrary.cpp                         -- Implementations facade BP, anti-spam, GetCorrectPlayerActor hack
        QPoliceConfigDataAsset.cpp                 -- Chargement/validation config vehicules
        QPoliceConfigResolver.cpp                  -- Resolution sync/async classes, StreamableManager
        QPoliceAsyncHelper.cpp                     -- Implementations safe update/targeting/teleport
        QAsyncLoadManager.cpp                      -- Implementations chargement async, cache, preload
        QPoliceDebugCVars.cpp                      -- ~25 CVars, commandes console, fonctions d'inspection
        QPolice_DevSettings.cpp                    -- Constructeur DevSettings
        QPoliceInteractionArea.cpp                 -- Logique zone interaction (proximite, decal, ground snap)
        QPaymentArea.cpp                           -- Logique paiement (calcul amende, deduction, timer)
        QInterfaceQueryLibrary.cpp                 -- Implementations template recherche interfaces sur composants
        IPoliceResponseProvider.cpp                -- Definition ModularFeatureName, implementations manager statique
        QPrisonSystemPrimaryAsset.cpp              -- Scan de niveaux pour prisons, extraction de donnees
        QPrisonTeleporter.cpp                      -- Teleportation, generation spawn points, BeginPlay registration
        Components/
            QPaymentZoneComponent.cpp              -- Machine a etats paiement, overlap, LOS, escalation
            QTaserComponent.cpp                    -- Machine a etats taser, effets, disable input, teleport prison
            QTerritoryDetectionComponent.cpp       -- Detection multi-source, filtrage crimes, routage factions
            QEnhancedTeleportationComponent.cpp    -- Phases teleportation, fade, amende, cleanup
        Services/
            QProviderManagerService.cpp            -- Ecoute IModularFeature, tri par priorite, overrides
            QRegistryService.cpp                   -- Enregistrement/nettoyage acteurs, cleanup periodique
            QWantedSystemService.cpp               -- Tick decay, calcul wanted level, events
            QUiEventService.cpp                    -- Connexion aux services, forwarding events, QNotification

Source/SpaceshipAI/
    SpaceshipAI.Build.cs                           -- Regles de build (QPolice, QAI, FlyVehicleMovement, WorldScape optionnel)
    Public/
        SpaceshipAI.h                              -- Module SpaceshipAI, macros QSHIP_LOG, categorie LogSpaceshipAI
        QSpaceshipAITypes.h                        -- Enums (ESpaceshipAIState, EFlightMode), FSpaceshipMovementParams, FSpaceshipAIConfig
        QSpaceshipAIConfigDataAsset.h              -- UDataAsset config (armes, mouvement, comportement IA)
        QSpaceshipPoliceProvider.h                 -- IPoliceResponseProvider pour vaisseaux (Lavrik, Velkara)
        QSpaceshipMovementInterface.h              -- Composant interface mouvement 3D -> FlyVehicleMovement
        QSpaceshipObstacleAvoidanceComponent.h     -- Evitement obstacles (sphere traces async, circumnavigation)
        QWeaponCoordinatorComponent.h              -- Coordination armes (mitrailleuse hitscan, roquettes homing, Niagara)
        QSimpleProjectile.h                        -- Projectile basique (collision, mouvement, homing, degats)
    Private/
        SpaceshipAI.cpp                            -- Startup/Shutdown module, creation/destruction du provider
        QSpaceshipPoliceProvider.cpp               -- Spawn vaisseaux, config mouvement/armes, QAI agent setup
        QSpaceshipMovementInterface.cpp            -- Navigation -> input vehicule, altitude hold, crash detect
        QSpaceshipObstacleAvoidanceComponent.cpp   -- Sphere traces, calcul avoidance, waypoints temporaires
        QWeaponCoordinatorComponent.cpp            -- Direct fire MG/Rocket, muzzle discovery, Niagara tracers, multicast
        QSimpleProjectile.cpp                      -- Setup velocity/homing, BeginPlay
```

---

## Dependances entre modules

```
                  +----------+
                  |   QAI    |
                  +----+-----+
                       ^
                       |
  +-----------+   +----+-----+   +-----------+   +---------------+
  | GravityS. |<--+  QPolice |-->| FlyVehicle|   | ClientAuth.   |
  +-----------+   +----+-----+   +-----------+   +---------------+
                       ^  ^                             ^
                       |  |                             |
                  +----+--+---+                         |
                  |SpaceshipAI|-------------------------+
                  +-----+-----+
                        |
                        v
                  +-----------+
                  | WorldScape|  (optionnel, WITH_WORLDSCAPE)
                  +-----------+

Dependances publiques QPolice:
  Core, CoreUObject, Engine, DeveloperSettings,
  AIModule, GameplayTasks, NavigationSystem,
  UMG, Slate, SlateCore,
  FlyVehicleMovement, Niagara, QAI, GravityScape,
  ClientAuthority, QNotification

Dependances privees QPolice:
  GameplayAbilities, NetCore, ReplicationGraph
  [Editor: UnrealEd, EditorStyle, EditorWidgets, ToolMenus, PropertyEditor, BlueprintGraph, KismetCompiler, AssetRegistry]
  [Conditionnel: WorldScapeCore, WorldScapeCommon, WorldScapeNoise, WorldScapeVolume, WorldScapeFoliages]
```
