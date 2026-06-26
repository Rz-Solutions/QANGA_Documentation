# QElevator — Architecture

> Plugin runtime (auteur : CyberAlien) fournissant un **ascenseur sur rail** : une plateforme se déplace le long d'une `USplineComponent` par interpolation d'alpha, avec portes coulissantes, barrières, sons, boutons interactifs et **synchronisation d'état déléguée à un composant externe d'état optimisé** (OSC). L'acteur n'est pas répliqué directement ; la cohérence multijoueur passe par l'OSC.

---

## 1. But d'usage

QANGA a besoin d'ascenseurs/plateformes déterministes et synchronisés en réseau (par ex. les complexes IcLabs). Le défi : déplacer une plateforme (et ce qu'elle porte) le long d'un trajet arbitraire défini par un designer, ouvrir/fermer des portes et barrières, jouer des sons, et garantir que **tous les clients voient la même position et le même état** — le tout sans payer le coût d'un acteur lourdement répliqué tickant en permanence.

La réponse de `QElevator` est :
- un **trajet en spline** éditable (au lieu d'un simple translate A→B), via `UQElevatorMovementComponent` qui mappe un alpha [0..1] sur la spline ;
- une **délégation de l'état réseau** à un `OptimizedStateComponent` (OSC) externe, appelé par **réflexion** — l'ascenseur pousse/lit `CanActive` (bool) et la position de timeline via cet OSC, qui porte la réplication.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Acteur + composant de mouvement.** `AQTrackElevator` est l'acteur placé dans le monde ; `UQElevatorMovementComponent` encapsule le déplacement le long de la spline (séparation logique de transport / présentation).
- **Mouvement par alpha sur spline.** Le composant interpole `CurrentAlpha` de `StartAlpha` vers `TargetAlpha` sur `MoveDuration` (méthode `MoveAlongSpline(TargetAlpha, Duration)`), positionne `MovableRoot` sur la spline à cet alpha, et diffuse `OnMovementUpdate` / `OnReachedDestination`. `SetPositionOnSpline(Alpha)` permet un placement direct (utilisé pour la resynchronisation).
- **État réseau via OSC, par réflexion.** L'ascenseur **ne hard-link pas** la classe de l'OSC. À `BeginPlay`, `BindToOptimizedStateComponent()` cherche, parmi les composants de l'acteur, celui dont le nom de classe **contient** `"OptimizedStateComponent"`, puis appelle ses fonctions par `FindFunctionByName` + `ProcessEvent` : `BindToBool`, `GetBool`, `SetBool`, `SetTimeline`. Ainsi l'état (`CanActive`, position de timeline) est partagé sans dépendance de compilation vers le plugin qui fournit l'OSC.
- **Présentation pilotée par data.** Portes (Z fermé/ouvert), barrières, durées d'animation, sons, classes de plateforme sont tous des `UPROPERTY EditAnywhere` — un designer configure un ascenseur entièrement depuis le Details panel.
- **Plateforme en ChildActor.** `MovablePlatform` est un `UChildActorComponent` : la plateforme effective est un acteur (`MovablePlatformClass`) instancié comme enfant, ce qui permet de réutiliser des BP de plateforme variés.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `AQTrackElevator` | `AActor` | Ascenseur : spline, plateforme, portes, barrières, sons, boutons, pont vers l'OSC |
| `UQElevatorMovementComponent` | `UActorComponent` (`BlueprintSpawnableComponent`) | Déplacement d'un `MovableRoot` le long d'une spline par interpolation d'alpha |
| `AQElevator_IclabsComplexDoor` | `AQTrackElevator` | Spécialisation IcLabs : override `InitializeComponents()`, `DefaultMovablePlatformPath` |
| `FOnElevatorReachedDestination` | délégué `OneParam(float FinalAlpha)` | Fin de déplacement (composant) |
| `FOnElevatorMovementUpdate` | délégué `TwoParams(float CurrentAlpha, float DeltaTime)` | Mise à jour par frame (composant) |
| `FOnElevatorStarted` / `FOnElevatorFinished` | délégués `()` | Début / fin de cycle (acteur) |

**Composants notables de `AQTrackElevator`** : `SceneRoot`, `ElevatorTrack` (`USplineComponent`), `MovablePlatform` (`UChildActorComponent`), `MovementComponent` (`UQElevatorMovementComponent`), `DoorMesh_A`/`DoorMesh_B` (`UStaticMeshComponent`), `LoopingSoundComponent` (`UAudioComponent`), `Interactive_Start`/`Interactive_End`/`Interactive_Inside` (`UStaticMeshComponent` boutons), et `OptimizedStateComponent` (`UActorComponent`, résolu au runtime).

---

## 4. Flux de données et cycle de vie

1. **Construction** — `OnConstruction` / `InitializeComponents()` met en place la spline, la plateforme (via `MovablePlatformClass`), les portes et boutons. `FindInteractiveComponents()` localise les boutons.
2. **BeginPlay** — `BindToOptimizedStateComponent()` repère l'OSC et l'abonne (`BindToBool` sur `OSCCanActiveKey`), puis `RefreshCanActiveFromOSC()` lit l'état initial `CanActive`. `BindInteractiveEvents()` connecte les boutons.
3. **Activation** — `ActivateElevator()` (ou un clic bouton → `OnInteractive*ClientInteract`) déclenche le cycle : diffusion `OnElevatorStarted`, fermeture des portes (`CloseAllDoors`), animation des barrières, lancement du son (`PlayStartSound` / `StartLoopingSound`), et appel `MoveAlongSpline(TargetAlpha, TravelTimeSeconds)` sur le composant.
4. **Propagation d'état** — l'activation pousse l'état via l'OSC : `CallOSCSetBool(OSCCanActiveKey, ...)` et `CallOSCSetTimeline(OSCTimelineKey, TargetAlpha, bForward)`. Les autres machines reçoivent ces changements par `OnOSCCanActiveChanged` / `OnOSCTimelineReceived` et resynchronisent leur plateforme (`SetPositionFromAlpha`).
5. **Déplacement** — chaque frame, `UQElevatorMovementComponent::TickComponent` avance `CurrentAlpha`, repositionne `MovableRoot` (`UpdatePosition`), diffuse `OnMovementUpdate` → `AQTrackElevator::OnMovementUpdate` (mise à jour visuelle/son).
6. **Arrivée** — au bout du trajet, `OnReachedDestination` → `OnMovementFinished` : `bEndPosition` bascule, ouverture de la porte de destination (`OpenDestinationDoor`), arrêt du son (`StopLoopingSound` / `PlayStopSound`), diffusion `OnElevatorFinished`, ré-autorisation (`CallOSCSetBool(OSCCanActiveKey, true)`).
7. **EndPlay** — nettoyage des timers d'animation (portes/barrières) et du son en boucle.

---

## 5. Réplication / réseau

`AQTrackElevator` **n'est pas un acteur répliqué de façon classique** (pas de `DOREPLIFETIME` ni de RPC dans la classe). La synchronisation multijoueur est **entièrement déléguée** à l'`OptimizedStateComponent` (OSC) :

- **Couplage par réflexion, pas par compilation.** L'OSC est trouvé par correspondance de nom de classe (`GetClass()->GetName().Contains("OptimizedStateComponent")`), puis ses fonctions sont invoquées via `FindFunctionByName` + `ProcessEvent` (`BindToBool`, `GetBool`, `SetBool`, `SetTimeline`). Aucune dépendance de module vers le fournisseur d'OSC.
- **État partagé** : un booléen `CanActive` (clé `OSCCanActiveKey = "CanActive"`) et une **timeline** d'ascenseur (clé `OSCTimelineKey = "ElevatorTL"`) portant l'alpha cible et le sens. C'est l'OSC qui réplique réellement ces valeurs ; l'ascenseur ne fait que pousser (set) et réagir (callbacks) localement.
- **Conséquence** : sans OSC valide sur l'acteur, l'ascenseur fonctionne en **local uniquement** (les set/get OSC sont court-circuités par des gardes `if (!OptimizedStateComponent) return;`). Le réseau dépend donc de la présence et de la réplication de ce composant externe.

---

## 6. Points d'intégration

- **Dépendances module** : `Core`, `CoreUObject`, `Engine` (public) ; `Slate`, `SlateCore` (privé). **Aucune** dépendance de compilation vers le plugin fournissant l'OSC (couplage réflexif délibéré).
- **OptimizedStateComponent** (fourni par [`QNetState`](QNetState_ARCHITECTURE.md)) : doit être ajouté sur l'acteur ascenseur (ou son BP) et expose `BindToBool`/`GetBool`/`SetBool`/`SetTimeline`. C'est le contrat implicite du couplage par réflexion ; l'état `Timeline` (Time + direction, synchronisé par l'horloge serveur de `QNetState`) porte la position de la plateforme.
- **Boutons interactifs** : `Interactive_Start`/`End`/`Inside` réagissent à un événement « client interact » (`OnInteractive*ClientInteract`) — intégration avec le système d'interaction du joueur.
- **Plateforme** : `MovablePlatformClass` (ou `DefaultMovablePlatformPath` pour la variante IcLabs) désigne le BP d'acteur transporté.

---

## 7. Gotchas, invariants et pièges

- **L'OSC est résolu par NOM de classe, pas par type.** Renommer la classe du composant d'état (pour qu'elle ne contienne plus `"OptimizedStateComponent"`) **casse silencieusement** la synchro réseau — l'ascenseur retombe en local sans erreur de compilation. De même, changer les signatures de `BindToBool`/`GetBool`/`SetBool`/`SetTimeline` casse le `ProcessEvent`.
- **Pas de réplication native.** Ne pas attendre que `AQTrackElevator` se synchronise tout seul : sans OSC, chaque machine anime son propre ascenseur indépendamment.
- **Z des portes en dur par défaut.** `DoorOpenZ = 312.0` est une valeur par défaut adaptée à un mesh précis ; à ajuster par instance pour d'autres portes.
- **Plateforme en ChildActor.** La plateforme étant un `UChildActorComponent`, ce qui est *posé dessus* n'est pas automatiquement attaché — le transport des acteurs/joueurs sur la plateforme dépend de la logique de la plateforme elle-même, pas de `QElevator`.
- **Animations sur timers.** Portes et barrières s'animent via des `FTimerHandle` + alpha manuel (pas un `UTimelineComponent` piloté) ; `EndPlay` doit invalider ces timers (déjà fait) sous peine de callbacks sur acteur détruit.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QElevator/Public/QTrackElevator.h` / `Private/QTrackElevator.cpp` | Acteur ascenseur, portes/barrières/sons/boutons, pont OSC |
| `Source/QElevator/Public/QElevatorMovementComponent.h` / `.cpp` | Déplacement sur spline par interpolation d'alpha |
| `Source/QElevator/Public/QElevatorTypes.h` | `AQElevator_IclabsComplexDoor` (spécialisation IcLabs) |
| `Source/QElevator/Public/QElevator.h` / `Private/QElevator.cpp` | Module |
| `QElevator.uplugin` | Descripteur : module Runtime, catégorie `Gameplay`, `CanContainContent: false` |
