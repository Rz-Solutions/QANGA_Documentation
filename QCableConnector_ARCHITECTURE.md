# QCableConnector — Architecture

> Plugin runtime (BETA, auteur : QANGA) fournissant des **câbles physiques** : un solveur de corde *position-based dynamics* (`URopeCableComponent`) rendu en mesh procédural, plus des acteurs **connecteur** et **socket** manipulables, intégrés au système d'interaction joueur, à l'overlay de portage ALS, à la gravité sphérique (NinjaCharacter) et aux **objectifs de quête** (interface de requête d'état).

---

## 1. But d'usage

Certaines quêtes/énigmes de QANGA demandent au joueur de **saisir un câble et de le brancher dans un socket** (raccorder une alimentation, relier deux machines…). Cela impose trois choses qu'aucun composant UE natif ne couvre simultanément :

1. **Une corde qui se comporte de façon crédible** — pend sous la gravité, se tend, entre en collision avec le monde et avec elle-même, sans recourir à une simulation physique rigide-body coûteuse.
2. **Une manipulation joueur en réseau** — saisir, porter (avec pose ALS), brancher/débrancher, le tout cohérent client/serveur.
3. **Un état interrogeable par le système de quête** — « ce câble est-il branché sur tel socket ? », « est-il tenu ? ».

`QCableConnector` répond avec un **solveur PBD maison** (Verlet + contraintes) qui ne dépend pas de Chaos pour la corde, des acteurs connecteur/socket qui se branchent par proximité, et l'interface `ICableConnectorInterface` que les objectifs de quête interrogent.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Solveur PBD propriétaire, pas Chaos.** `URopeCableComponent` simule un tableau de particules (`FRopeCableParticle`) par intégration de Verlet + satisfaction de contraintes (étirement, flexion, alignement des extrémités), avec sous-pas (`Substeps`) et itérations (`SolverIterations`). C'est déterministe, léger et indépendant du moteur physique rigide.
- **Rendu en ProceduralMesh.** La corde est rendue comme un tube généré à la volée par un `UProceduralMeshComponent` interne (`RenderMesh`), paramétré par `TubeRadius`/`TubeSides`/`CableMaterial`/`UVScale`. Le composant lui-même est un `USceneComponent` (la simulation pilote le mesh, elle n'en hérite pas).
- **Gravité sphérique via NinjaCharacter.** Quand `bUseSphericalGravity` est actif, le solveur va chercher la gravité d'un `ANinjaPhysicsVolume` (`FindGravityVolume`), cohérente avec la gravité planétaire du reste du jeu — la corde tombe « vers la planète », pas vers `-Z` monde.
- **Mise en sommeil.** Au repos, la corde s'endort (`SleepSpeedThreshold`, `SleepDelay`, `SleepBlendTime`) pour ne plus consommer de CPU tant que les ancres ne bougent pas.
- **Acteurs séparés pour connecteur / socket / extrémité.** `ACableConnectorActor` (saisissable, porte le câble), `ACableSocketActor` (cible de branchement), `ACableEndActor` (embout simple). Découplage clair entre la corde (composant) et l'interaction (acteurs).
- **Couplage souple au système d'interaction et à ALS.** Les hooks d'interaction (`InteractClient`/`InteractServer`/…) sont des `BlueprintNativeEvent` qui matchent l'interface de jeu `/Game/Systems/Interact/Interact_Interface` ; l'overlay de portage ALS est résolu par nom (`CarryOverlayName = "Box"`) — pas de hard-link vers ces systèmes BP.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `URopeCableComponent` | `USceneComponent` (`BlueprintSpawnableComponent`) | Solveur PBD de corde + rendu tube procédural + tension + collision |
| `ACableConnectorActor` | `AActor`, `ICableConnectorInterface` | Connecteur saisissable : porte le câble, se branche dans un socket, gère portage ALS et tension |
| `ACableSocketActor` | `AActor` | Socket : zone de branchement par proximité, hooks d'interaction |
| `ACableEndActor` | `AActor` | Embout simple (mesh + zone de saisie) |
| `ICableConnectorInterface` / `UCableConnectorInterface` | `UINTERFACE` (`BlueprintType`) | Requête d'état pour la quête : `CableIsConnectedTo(AActor*, FName)`, `CableIsHeld()` |
| `FRopeCableParticle` | `USTRUCT` | Particule du solveur : `Position`, `InvMass`, `bPinned` |

**Délégués de `URopeCableComponent`** : `OnCableUpdated(TArray<FVector>)`, `OnCableConnected(AActor*)`, `OnCableDisconnected()`, `OnTensionChanged(float)`, `OnMaxTensionReached()`, `OnMaxTensionReleased()`.
**Délégué de `ACableConnectorActor`** : `OnCableAutoDisconnected()`.

---

## 4. Flux de données et cycle de vie

### Simulation de la corde (`URopeCableComponent::TickComponent` → `Simulate`)
1. `UpdateAnchors()` — replace les particules d'extrémité sur `StartComponent`/`EndComponent` (et leurs sockets).
2. `Integrate(dt)` — intégration de Verlet avec gravité (sphérique ou `-Z`), amortissement (`Damping`), clamp de vitesse (`MaxLinearSpeed`).
3. `SatisfyConstraints()` × `SolverIterations` — contraintes de distance (étirement, modulé par `StretchCompliance`), puis `ApplyBending()` (raideur d'angle), `ApplyEndAlignment()` (oriente les segments d'extrémité vers la forward de l'ancre).
4. `ApplyCollision()` / `ApplySelfCollision()` — collision contre le monde (`CollisionChannel`, `CollisionRadius`, friction, contact-pinning) et contre soi-même.
5. `UpdateTensionState()` — calcule `TensionRatio` (longueur courante / repos vs `MaxStretchRatio`), bascule `bAtMaxTension`, diffuse `OnTensionChanged` / `OnMaxTensionReached` / `OnMaxTensionReleased`.
6. `UpdateRender()` — régénère le tube procédural et diffuse `OnCableUpdated`.
7. Mise en sommeil si la corde reste sous le seuil de vitesse pendant `SleepDelay`.

### Manipulation du connecteur (`ACableConnectorActor`)
1. **Saisie** — via l'interaction joueur : `GrabCable(AttachParent, Socket, Offset)` attache le tip à la main (recherche du meilleur socket de main, `FindBestHoldSocket`), active l'overlay ALS de portage (`ApplyCarryOverlay`) et l'IK des mains (`UpdateHandIK`).
2. **Portage** — chaque `Tick`, `UpdateConnectorTransform` / `UpdateCarryHandleTransform` place l'embout entre les mains via le `CarryHandle`.
3. **Branchement** — `PlugIntoNearbySocket()` ou `ConnectToSocket(SocketActor)` : le socket le plus proche est trouvé par overlap (`ConnectorGrabArea` / `ConnectorAttachArea`), `ConnectedSocket` est renseigné et répliqué.
4. **Tension** — `UpdateTensionEffects` ; si tension max, `ApplyMovementConstraint` (bloque le déplacement du porteur si `bBlockPlayerMovementAtMaxTension`) et/ou `CheckAutoDisconnect` (débranchement auto après `AutoDisconnectGracePeriod` si `bAutoDisconnectAtMaxTension`), diffusant `OnCableAutoDisconnected`.
5. **Lâcher / débranchement** — `DropCable()` / `ReleaseFromSocket()` restaurent l'overlay/stance ALS et libèrent le tip.

---

## 5. Réplication / réseau

- **État répliqué** : `ACableConnectorActor::ConnectedSocket` est `UPROPERTY(ReplicatedUsing=OnRep_ConnectedSocket)` (déclaré dans `GetLifetimeReplicatedProps`). Quand le serveur valide un branchement, les clients reçoivent `OnRep_ConnectedSocket` et raccordent visuellement l'extrémité du câble au socket.
- **Actions joueur** : la saisie/le branchement passent par l'**interface d'interaction du jeu** (hooks `InteractClient` / `InteractSendToServer` / `InteractServer` / `InteractLost` / `HasInteraction`, des `BlueprintNativeEvent` matchant `Interact_Interface`). Le chemin client→serveur de l'interaction est porté par ce système d'interaction, pas par des macros `UFUNCTION(Server)` propres au plugin.
- **La simulation de corde est locale.** Le solveur PBD tourne sur chaque machine ; seul l'**état de connexion** (quel socket) est répliqué. La forme exacte de la corde n'est pas synchronisée bit à bit (elle converge via les mêmes ancres + déterminisme du solveur).

---

## 6. Points d'intégration

- **Dépendances module** : `Core`, `CoreUObject`, `Engine`, `ProceduralMeshComponent` (public) ; `RenderCore`, `RHI`, `Projects`, `NinjaCharacter` (privé). Plugins activés : `ProceduralMeshComponent`, `NinjaCharacter`.
- **NinjaCharacter** : fournit `ANinjaPhysicsVolume` pour la gravité sphérique de la corde.
- **Système d'interaction** (`/Game/Systems/Interact/Interact_Interface`) : matché par les hooks `Interact*` ; c'est lui qui déclenche `GrabCable`/`PlugIntoNearbySocket` côté gameplay.
- **ALS** : `CarryOverlayName` (« Box ») et `CarryPoseAnimation` pilotent la posture de portage ; doit exister dans l'`ALS_OverlayState`.
- **Système de quête** : interroge l'état via `ICableConnectorInterface` (`CableIsConnectedTo`, `CableIsHeld`) — un objectif vérifie ainsi qu'un câble précis est branché au bon socket.

---

## 7. Gotchas, invariants et pièges

- **`bUseSphericalGravity` exige un `ANinjaPhysicsVolume`.** Sans volume de gravité Ninja trouvé, la corde retombe sur la gravité par défaut ; sur une planète, cela donne une corde qui « tombe vers -Z monde » au lieu de vers le sol.
- **Coût = NumSegments × Substeps × SolverIterations.** 128 segments × 4 sous-pas × 2 itérations est le défaut ; augmenter ces valeurs rend la corde plus rigide/stable mais multiplie le coût CPU par câble. La mise en sommeil est essentielle pour les câbles statiques.
- **Overlay ALS résolu par nom.** `CarryOverlayName` doit correspondre à une valeur réelle de l'énumération `ALS_OverlayState` ; une faute de frappe casse silencieusement la pose de portage.
- **Hooks d'interaction par convention.** Les `Interact*` ne marchent que si l'acteur est piloté par le système d'interaction du jeu (l'interface `Interact_Interface`) ; ce n'est pas un système de RPC autonome du plugin.
- **Seule la connexion est répliquée.** Ne pas supposer que la forme de la corde est identique au pixel près entre clients ; seuls les points d'ancrage et l'état `ConnectedSocket` sont garantis cohérents.
- **`RequiredConnectorId` sur le socket est inutilisé** (commenté « unused for now ») — le filtrage par identifiant de connecteur n'est pas encore branché.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QCableConnector/Public/RopeCableComponent.h` / `Private/…cpp` | Solveur PBD, collision, tension, rendu tube procédural |
| `Source/QCableConnector/Public/CableConnectorActor.h` / `…cpp` | Connecteur saisissable : grab/plug, portage ALS, tension, réplication `ConnectedSocket` |
| `Source/QCableConnector/Public/CableSocketActor.h` / `…cpp` | Socket de branchement par proximité |
| `Source/QCableConnector/Public/CableEndActor.h` / `…cpp` | Embout simple |
| `Source/QCableConnector/Public/CableConnectorInterface.h` | Interface de requête d'état pour la quête |
| `QCableConnector.uplugin` | Descripteur : Runtime, BETA, plugins `ProceduralMeshComponent` + `NinjaCharacter` |
