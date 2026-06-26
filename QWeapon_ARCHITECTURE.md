# QWeapon — Architecture

Couche d'animation d'armes en C++ pour le squelette de personnage ALS (auteur : RzZz). Le plugin remplit deux rôles distincts mais complémentaires : (1) il porte la **pose procédurale des bras/de la colonne/de la tête tenant une arme** (IK, copies d'os, ADS), réécrite depuis un ancien `AnimBP` Blueprint en C++ pur multi-thread ; (2) il fournit le **chemin de tir hitscan « sans projectile »** (dégâts autoritaires serveur + traçantes cosmétiques côté client) partagé par les armes joueur et les IA QAI.

Statut : en production. La couche d'animation est entièrement portée en C++ (event graph et bone-manipulation chains de l'`AnimBP` d'origine reconstruits à l'identique, avec correctifs explicites par rapport au graphe legacy). Le sous-système de balles est le résultat du chantier « bullet hitscan rework » (cf. mémoire projet) et est déjà consommé par QAI.

---

## 1. But d'usage

QANGA est un open-world multijoueur sci-fi à planètes entières, dont les personnages reposent sur le squelette/framework **ALS (Advanced Locomotion System) Mannequin**. Deux problèmes concrets motivent ce plugin :

**a) La pose d'arme procédurale.** L'ancien personnage portait son arme via un `AnimBP` Blueprint massif (≈70 variables, des dizaines de nœuds `ModifyBone` / `CopyBone` / `TwoBoneIK` / `FABRIK`). Ce graphe était coûteux (event graph par frame, exécution game-thread), fragile, et difficile à maintenir. `QWeapon` réimplémente l'intégralité de cette logique en C++ :
- l'`event graph` devient `UQWeaponAnimInstance::NativeUpdateAnimation` (+ un `FAnimInstanceProxy` thread-safe) ;
- la chaîne de manipulation d'os devient un nœud d'`AnimGraph` natif unique, `FAnimNode_QWeaponEvaluate`, qui s'exécute `_AnyThread` (worker threads d'animation).

L'objectif n'est pas une feature UE générique : c'est la reproduction fidèle, performante et déterministe d'un comportement d'arme ALS spécifique (rifle, pistolet 1 main / 2 mains, arc, doigt pointé, vol, ADS première/troisième personne, rechargement) sur le squelette mannequin de QANGA.

**b) Le tir hitscan sans projectile.** Auparavant chaque tir faisait apparaître un acteur `BP_Projectile` (point light + particle + déplacement Ninja), ce qui saturait le frame en combat de masse et **cassait les dégâts d'IA sur serveur dédié meshless**. `UQWeaponBulletSubsystem` remplace ça par un line trace `ECC_Pawn` autoritaire serveur (qui marche sans mesh) + une traçante cosmétique client (mesh étiré émissif + point light), pooled, le tout partagé par le joueur ET les drones/cyborgs QAI — un seul système de balles pour tout le jeu.

---

## 2. Vue d'ensemble / décisions d'architecture

Le plugin contient **deux modules** (cf. `QWeapon.uplugin`) :

- `QWeapon` — module **Runtime** (`LoadingPhase: Default`, Win64/Linux). Tout le runtime : anim instance, anim node, sous-système de balles, camera shake.
- `QWeaponEditor` — module **UncookedOnly** (éditeur). Le wrapper `UAnimGraphNode_*` qui expose le nœud dans l'éditeur d'AnimGraph, plus un utilitaire de câblage automatique du graphe.

### Décisions clés

1. **Séparation Update / Evaluate via un proxy.** `UQWeaponAnimInstance` calcule par frame (sur le game thread, dans `NativeUpdateAnimation`) ~70 valeurs d'état (modes ALS, angles de visée, offsets de viseur, rotations IK…). Ces valeurs sont copiées dans `FQWeaponAnimProxy` (sous-classe de `FAnimInstanceProxy`) lors de `PreUpdate`, et c'est UNIQUEMENT cette copie thread-safe que `FAnimNode_QWeaponEvaluate` lit pendant `Evaluate_AnyThread`. C'est le pattern UE standard pour autoriser l'évaluation d'anim hors game-thread sans data race.

2. **Couplage au personnage Blueprint par réflexion / BPI, pas par dépendance dure.** L'anim instance ne connaît pas la classe du personnage QANGA. Elle lit l'état via des fonctions d'interface Blueprint trouvées par nom (`FindFunction` + `ProcessEvent`) : `BPI_Get_RotationMode`, `BPI_Get_OverlayState`, `BPI_Get_ViewMode`, `BPI_Get_MovementState`, `BPI_Get_InclineState`, `BPI_GetWeaponValues`, `BPI_GetFPCameraTarget`, et l'arme active via `InventoryComponent.GetActiveItem` / `UpdateCurrentAimpointPosition` / `CurrentAimpointScene`. Le plugin reste donc découplé de l'implémentation BP côté gameplay.

3. **Mapping d'axes squelette spécifique.** Pour ce squelette mannequin, en ComponentSpace : **X = Pitch**, **Y = Roll**, **Z = Yaw** (différent de la convention `FRotator` Pitch=Y/Yaw=Z/Roll=X). Les rotations de pitch (tête, ik_hand_gun) sont appliquées par quaternion autour de l'axe X pour éviter le gimbal lock (cf. `EvaluateSpineChain`, MB30 head). Le **Roll de `ik_hand_gun` = le pitch visuel de l'arme** (le canon qui pointe haut/bas). En vue FP relative à l'écran : X = gauche/droite, Y = haut/bas (Y positif = pousse vers le bas), Z = avant/arrière.

4. **Sous-système de balles partagé, mesh-free.** `UQWeaponBulletSubsystem` est un `UTickableWorldSubsystem` (un par monde Game/PIE). Les dégâts sont résolus côté autorité par un seul line trace `ECC_Pawn` (capsule, fonctionne sur serveur dédié sans mesh), appliqués via `UGameplayStatics::ApplyDamage` — exactement comme les drones QAI, ce qui était le but du rework. Les cosmétiques (traçante, muzzle flash, impact FX, hit marker, camera shake) ne tournent que sur les machines qui rendent (skip `NM_DedicatedServer`) et sont gouvernés par des budgets pour tenir en combat de masse.

5. **Tout passe par des CVars `QWeapon.*`.** Le sous-système de balles n'a pas de `UDeveloperSettings` ; il est entièrement piloté par ~25 console variables (`QWeapon.DebugHitscan`, `QWeapon.TracerPoolMax`, `QWeapon.ImpactFX*`, `QWeapon.MuzzleFlash*`, `QWeapon.CamShake*`…), ce qui permet de tuner les gouverneurs de foule en live sans rebuild.

### Implémenté vs intention

- Le nœud d'anim contient des branches très spécialisées par `OverlayState` (notamment **Pistol1H** qui a un traitement complet à part : pas de front-grip, IK d'épaule, arc de visée, pose « relaxed » à la Lara Croft) et **Bow**. Les autres overlays (Rifle, Pistol2H) suivent le chemin générique.
- `FQWeaponAnimProxy` et `UQWeaponAnimInstance` déclarent un grand nombre de champs miroirs (l'instance porte la version « source », le proxy la copie). C'est volontaire (parité thread-safe avec l'ancien graphe) ; certains champs de l'ancien graphe sont conservés même s'ils sont aujourd'hui constants (ex. `HeadYaw` reste `(90,0,0)` du CDO, jamais modifié — commenté comme tel).
- L'utilitaire éditeur `UQWeaponAnimGraphSetup::SetupWeaponAnimGraph` câble automatiquement le graphe d'anim de référence (LinkedInputPose → cache « Input » → QWeaponEvaluate → cache « WeaponBase » → Slot « Reload » → LayeredBoneBlend → QWeaponEvaluate post-reload → Root). C'est un outil d'édition/setup, pas du runtime.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQWeaponAnimInstance` | `UCLASS` (`UAnimInstance`) | Remplace l'event graph de l'ancien AnimBP : lit l'état ALS/arme du personnage (par BPI/réflexion), calcule par frame les ~70 valeurs de pose (angles de visée, offsets viseur, rotations IK, etc.), pilote le rechargement (`HandleBPI_Reload`). |
| `FQWeaponAnimProxy` | `USTRUCT` (`FAnimInstanceProxy`) | Copie thread-safe de l'état calculé par l'anim instance ; remplie dans `PreUpdate`, seule source lue par le nœud d'anim dans `Evaluate_AnyThread`. |
| `FAnimNode_QWeaponEvaluate` | `USTRUCT` (`FAnimNode_Base`) | Le cœur : nœud d'AnimGraph qui manipule les os (spine, tête, mains, `ik_hand_gun`) — `ModifyBone`, `CopyBone`, `TwoBoneIK`, `FABRIK` — pour produire la pose d'arme/ADS/visée/pointage. Mode `bPostReloadGripOnly` pour le post-pass de grip pendant le rechargement. |
| `UQWeaponBulletSubsystem` | `UCLASS` (`UTickableWorldSubsystem`) | Tir hitscan : `ServerFireBullet` (dégâts autoritaires serveur, headshots, friendly-fire) + `SpawnBulletTracer` (traçante mesh+light pooled, muzzle flash, impact FX, hit marker, camera shake — cosmétiques client). Tick fait avancer les traçantes en vol. |
| `UQWeaponFireCameraShake` | `UCLASS` (`ULegacyCameraShake`) | Camera shake par tir pour le tireur local : buzz rotationnel (pitch up + yaw/roll aléatoires) + un FOV punch. Joué seulement sur le joueur local possédant l'arme. |
| `FQWeaponModule` | `IModuleInterface` | Module runtime ; déclare `LogQWeapon`. `StartupModule`/`ShutdownModule` vides (rien à enregistrer au boot). |
| `namespace QWeapon` (RotationMode/OverlayState/ViewMode/MovementState/Gait/MovementAction/InclineState) | constantes `uint8` + helpers | Miroir C++ des enums Blueprint ALS (valeurs `uint8`), plus `IsPistolOrBow` / `IsPistol1H`. Évite une dépendance dure aux UENUM Blueprint. |
| `UAnimGraphNode_QWeaponEvaluate` | `UCLASS` (`UAnimGraphNode_Base`, module Editor) | Wrapper éditeur exposant `FAnimNode_QWeaponEvaluate` dans l'AnimGraph (titre/tooltip/catégorie « QWeapon »). |
| `UQWeaponAnimGraphSetup` | `UCLASS` (`UBlueprintFunctionLibrary`, module Editor, `WITH_EDITOR`) | Outil d'édition : `SetupWeaponAnimGraph(Path)` reconstruit et câble automatiquement le graphe d'anim de référence (cache poses, slot Reload, layered blend, post-pass grip). |
| `FQWeaponEditorModule` | `IModuleInterface` (module Editor) | Module éditeur minimal (startup/shutdown vides). |

Sous-structures internes (non-UCLASS) de `UQWeaponBulletSubsystem` : `FQWeaponRig` (mesh streak + point light) et `FQWeaponTracer` (un rig en vol muzzle→impact). Les composants créés sont gardés vivants par des tableaux `UPROPERTY(Transient)` (`AllTracerMeshes`, `AllTracerLights`).

---

## 4. Flux de données et cycle de vie

### Couche d'animation

1. **Init** — `NativeInitializeAnimation` : résout le pawn propriétaire (via `TryGetPawnOwner`, ou l'acteur du `GetOwningComponent` pour les linked anim graphs), met en cache `Character`/`Pawn`, détermine `bIsPlayerController`/`bLocallyControlled`, et enregistre `RegisterOnBoneTransformsFinalizedDelegate` → `HandleBoneTransformsFinalized` (qui re-pousse les transforms des composants attachés par socket après finalisation des os).
2. **Per-frame (game thread)** — `NativeUpdateAnimation` :
   - gère la fin de montage de rechargement (`ClearReloadState`) ;
   - `CallBPI_GetCurrentStates` (lit les 5 enums ALS par BPI) ;
   - détecte le changement d'arme via `InventoryComponent.GetActiveItem` (`bWeaponSwitchSnap`) ;
   - `UpdateWeaponLocationSight` appelle `UpdateCurrentAimpointPosition` sur l'arme (déclenche la chaîne BPI/réplication côté gameplay) ;
   - `CallBPI_GetWeaponValues` (un `FTransform` dont l'échelle transporte les offsets de main gauche) ;
   - sur switch : `OverrideAimpointFromWeapon` lit directement `CurrentAimpointScene`/`DistanceFromCamera` de l'arme par réflexion pour éviter le lag de la chaîne BPI asynchrone, puis snap l'interpolation ;
   - `UpdateCameraPosition` (socket `FP_Camera`/`head`, cible caméra FP via `BPI_GetFPCameraTarget`) ;
   - `UpdateALSValues` : calcule `AimAngles` (delta control/actor en espace local), `PitchPerBone` (réparti sur la colonne), `HeadPitch`, alphas ready/aim, `HandLIK_Alpha` (désactivé pour Pistol1H/melee), offsets front-grip (socket `FrontGripSocket` de l'arme → espaces `hand_r`/`ik_hand_gun`).
3. **PreUpdate (proxy)** — `FQWeaponAnimProxy::PreUpdate` copie l'état de l'instance vers le proxy. Point critique : la `ReplicatedControlRotation` est relue **directement** depuis le pawn pour les pawns localement contrôlés (PreUpdate précède NativeUpdateAnimation, donc la variable d'instance est périmée pour l'ADS FP frame-accurate).
4. **Evaluate (worker thread)** — `FAnimNode_QWeaponEvaluate` :
   - `CacheBones_AnyThread` résout une fois tous les indices d'os (spine_01/02/03, neck_01, head, clavicle/upperarm/lowerarm/hand l+r, ik_hand_root/gun/r/l, `VB Sight`, `VB RHS_ik_hand_gun`) ;
   - `Update_AnyThread` lisse les alphas de blend (weapon/aim/pointing/spine-yaw/reload-suppression) ;
   - `Evaluate_AnyThread` construit la pose : chaîne spine (`EvaluateSpineChain`), puis blend entre `EvaluateNotAimingPath` (arme baissée/ready) et `EvaluateAimingPath` (ADS, TwoBoneIK des mains), plus `EvaluatePointingPath` (doigt pointé) — chaque chemin opère en ComponentSpace puis reconvertit en local. Mode `bPostReloadGripOnly` : ne fait qu'ancrer `ik_hand_gun` sur `VB RHS`/`hand_r` pendant le poids du slot « Reload ».
5. **Teardown** — `NativeUninitializeAnimation` désenregistre le delegate `OnBoneTransformsFinalized`.

### Tir hitscan

1. Le code de tir (graphe d'arme BP côté joueur, ou `QAI_CombatProcessor` côté IA) récupère `World->GetSubsystem<UQWeaponBulletSubsystem>()`.
2. **Autorité** : `ServerFireBullet(bEnabled, Shooter, Muzzle, Dir, Range, Damage, …)` — `LineTraceMultiByChannel(ECC_Pawn)`, saute les hits `bStartPenetrating`/distance ≤ 1 (volumes spawner), saute le friendly-fire (faction lue par réflexion sur `CombatComponent.Faction`), détecte le headshot (bone `head`, sinon trace per-bone sur le skeletal mesh), applique les dégâts (`UGameplayStatics::ApplyDamage`, instigator = controller du tireur). No-op si `bEnabled=false` (armes à ordnance) ou hors autorité.
3. **Cosmétique** (toutes machines qui rendent) : `SpawnBulletTracer` — muzzle flash (attaché au composant muzzle si fourni), camera shake (joueur local), trace de visibilité `ECC_Visibility` pour l'endpoint, impact FX surface-typé, hit marker `W_HitFeedback` (joueur local), puis acquiert un rig pooled et lance une traçante.
4. **Tick** : fait avancer chaque traçante (`Lerp` muzzle→impact) ; à l'arrivée, masque instantanément le rig et le rend au pool.
5. **Deinitialize** : détruit tous les composants mesh/light créés.

---

## 5. Réplication / réseau

**Ce plugin ne réplique aucun état lui-même.** Aucun `DOREPLIFETIME`, aucune `UPROPERTY(Replicated)`, aucun RPC `Server_`/`Client_`/`Multicast_` dans `QWeapon`/`QWeaponEditor`.

Le modèle réseau est **assumé par les systèmes appelants**, pas par le plugin :

- **Dégâts** : `ServerFireBullet` est explicitement *authority-only* (`ShooterPawn->HasAuthority()` requis ; sinon no-op). La résolution de hit et l'application de dégâts sont donc serveur-autoritaires. La conception (line trace `ECC_Pawn` capsule, sans mesh) est précisément ce qui permet aux dégâts de fonctionner sur les serveurs dédiés meshless de QANGA. L'attribution crime/wanted suit l'instigator = `PlayerController` du tireur.
- **Cosmétiques** : `SpawnBulletTracer` (et muzzle flash/impact FX/hit marker/camera shake) ne tournent que sur les machines qui rendent (early-out sur `NM_DedicatedServer`). Ils ne sont pas répliqués par le plugin : c'est à l'appelant de les déclencher sur les bonnes machines (ex. QAI les déclenche via un multicast `MC_PlayShotVFXEx` de son `QAI_PoliceUnitMarkerComponent` ; les armes joueur les appellent localement). Les shots du joueur local possédant l'arme contournent les gouverneurs de foule (le tireur doit toujours voir son propre feedback).
- **État d'animation** : l'anim instance lit la `ControlRotation` (répliquée par l'engine pour les proxies) et l'état ALS du personnage via BPI ; elle ne réplique rien.

---

## 6. Points d'intégration

### Dépendances du plugin (de `QWeapon.uplugin` + `QWeapon.Build.cs`)
- Plugins : **Niagara**, **EngineCameras** (ce dernier pour `ULegacyCameraShake`).
- Modules : `Core`, `CoreUObject`, `Engine`, `AnimGraphRuntime`, `AnimationCore` (les solveurs `TwoBoneIK`/`FABRIK`), `Niagara`, `UMG` (le hit marker `W_HitFeedback`), `EngineCameras`.
- Module éditeur `QWeaponEditor` : dépend de `QWeapon` + (privé) `AnimGraph`, `BlueprintGraph`, `UnrealEd`.

### Systèmes QANGA qui dépendent de QWeapon
- **QAI** (`QAI.Build.cs` liste `"QWeapon"`). `QAI_CombatProcessor.cpp` inclut `QWeaponBulletSubsystem.h` et fait tirer les IA ranged (cyborgs et drones) directement par `ServerFireBullet` + `SpawnBulletTracer` — au lieu de l'ancien flux BP latent. C'est l'unification voulue : joueur et IA partagent un seul système de balles. (Voir aussi `QAI_PoliceUnitMarkerComponent.cpp`, `QAI_SensingComponent.cpp`.)
- **Personnage ALS / graphes d'armes BP** : consomment `UQWeaponAnimInstance` (comme anim instance liée sur le mesh) et le nœud `QWeapon Evaluate` dans l'AnimGraph ; côté gameplay les armes BP appellent `ServerFireBullet`/`SpawnBulletTracer` depuis le chemin de tir de base (`WeaponScript`).

### Contrats d'interface attendus côté gameplay (par nom, résolus par réflexion)
- Sur le personnage : `BPI_Get_RotationMode`, `BPI_Get_OverlayState`, `BPI_Get_ViewMode`, `BPI_Get_MovementState`, `BPI_Get_InclineState`, `BPI_GetWeaponValues` (retourne un `FTransform`), `BPI_GetFPCameraTarget` (retourne un `FVector` monde), propriété `InventoryComponent`.
- Sur le composant d'inventaire : `GetActiveItem`. Sur l'arme : `UpdateCurrentAimpointPosition`, propriétés `CurrentAimpointScene` (`USceneComponent*`), `DistanceFromCamera` (`double`), socket `Aimpoint`. Sur le mesh d'arme : socket `FrontGripSocket`.
- Sur le mesh personnage : sockets/os `FP_Camera`, `head`, `hand_r`, `ik_hand_gun`, et les bones cachés par `CacheBones` (dont les virtual bones `VB Sight` et `VB RHS_ik_hand_gun`).

### Assets référencés en dur (chemins)
Traçante : `/Engine/BasicShapes/Sphere`, `/Game/Systems/Weapon/M_QWeaponTracer`. Impact/muzzle/hit-marker (BallisticsVFX + widgets) : `Cubit_ImpactFX_Spawner`, `Muzzle_Flash_Large`, `Generic_Impact` (lite IA), `W_HitFeedback`. Setup éditeur : poses ALS Pistol 1H/2H et `/Game/Pawn/Anim/PointingFinger_Pose`.

### CVars notables (toutes `QWeapon.*`)
`DebugHitscan`, `TracerPoolMax` (256), `TracerLightIntensity`/`TracerLightRadius`/`TracerLength`/`TracerWidth`, `ImpactFX`, `ImpactFXMaxDist`/`ImpactFXMaxPerFrame`/`ImpactFXFrameInterval`/`ImpactFXDecalLife`/`ImpactFXDecalLifeAI`/`ImpactFXLiteAI`, `MuzzleFlash`/`MuzzleFlashScale`/`MuzzleFlashMaxDist`/`MuzzleFlashMaxPerFrame`/`MuzzleFlashFrameInterval`, `CamShake`/`CamShakeScale`. Les gouverneurs de distance/par-frame/intervalle existent pour tenir un firefight à 100+ IA (les cosmétiques par tir, pas le brain, dominaient le frame — cf. mémoire projet) ; le joueur local est toujours exempté.

---

## 7. Gotchas, invariants et pièges

- **Mapping d'axes non standard.** ComponentSpace pour ce squelette : **X=Pitch, Y=Roll, Z=Yaw**. Toute rotation de pitch sur head/`ik_hand_gun` se fait par quaternion autour de X (pas via `FRotator`), sinon gimbal lock. Le **Roll de `ik_hand_gun` est le pitch visuel de l'arme**. Ne pas « corriger » ces axes vers la convention `FRotator`.
- **Le proxy est la seule donnée valide en `_AnyThread`.** N'accédez jamais à `UQWeaponAnimInstance` (ses `UPROPERTY`) depuis le nœud d'anim — uniquement `FQWeaponAnimProxy`. Toute nouvelle valeur consommée par le nœud doit être ajoutée au proxy ET copiée dans `PreUpdate`.
- **`ReplicatedControlRotation` est relu dans `PreUpdate`** directement depuis le pawn pour les pawns locaux (l'ordre `PreUpdate` → `NativeUpdateAnimation` rend la variable d'instance périmée pour l'ADS FP frame-accurate).
- **Ne pas utiliser les axes de `ComponentTransform` comme directions dans les helpers CS.** Ce sont des directions monde ; sur les mondes à gravité planétaire/LWC le personnage est réorienté et ça casse. Utiliser `FVector::UpVector`/axes d'os en ComponentSpace (cf. `ComputeRightElbowJointTargetCS`).
- **`ServerFireBullet` est authority-only** et doit sauter les hits `bStartPenetrating`/distance ≤ 1 (un volume spawner qui répond à `ECC_Pawn` et où le muzzle se trouve renvoie un hit à distance 0, masquant la vraie cible). Même règle côté trace cosmétique.
- **Friendly-fire et headshot sont résolus par réflexion** (`CombatComponent.Faction`, bones `head`/`neck_01`). Le headshot via skeletal mesh **n'existe pas sur serveur dédié meshless** (graceful : pas de headshot là-bas, comme l'ancien projectile).
- **Ne pas router les dégâts via `Lib_Combat.ClientRequestDamage`.** Commenté explicitement : cette fonction n'applique les dégâts serveur que sur la branche [PvP-on + Is-Server] ; en PvE elle retombe sur un chemin client-only qui no-op sur serveur dédié et re-casserait les dégâts d'IA. Utiliser `UGameplayStatics::ApplyDamage` direct.
- **Les composants de traçante sont gardés vivants manuellement** par `AllTracerMeshes`/`AllTracerLights` (les `FQWeaponRig` ne portent que des pointeurs bruts). `Deinitialize` doit les `DestroyComponent`. Le pool est plafonné par `QWeapon.TracerPoolMax` ; au plafond, la traçante est silencieusement droppée (les dégâts, eux, sont déjà résolus).
- **Cosmétiques côté rendu uniquement.** `SpawnBulletTracer` et tous les FX early-out sur `NM_DedicatedServer`. Le plugin ne réplique pas les FX : l'appelant est responsable de les jouer sur les bonnes machines.
- **Le rechargement répare les montages malformés.** `ResolveReloadMontage` duplique et reconstruit un montage dont l'`AnimReference` est cassée (en retrouvant la séquence par substitution de chemin `_Montage`). Pendant le reload, le chemin source de l'arme est supprimé (`bPostReloadGripOnly` ancre juste le grip) et les slots ALS de transition haut-du-corps sont stoppés pour ne pas combattre le blend-in.
- **`HeadYaw` est constant `(90,0,0)`** (du CDO, jamais modifié) — vestige fidèle de l'ancien graphe ; ne pas supposer qu'il est calculé.
- **`UQWeaponAnimGraphSetup` est `WITH_EDITOR` / DevelopmentOnly** : c'est un outil de setup du graphe de référence, pas du runtime. Il efface et reconstruit l'AnimGraph complet ; à n'exécuter qu'en connaissance de cause.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `G:/QANGA/Plugins/QWeapon/QWeapon.uplugin` | Manifeste : 2 modules (Runtime `QWeapon`, UncookedOnly `QWeaponEditor`), plugins Niagara + EngineCameras, Win64/Linux. |
| `Source/QWeapon/QWeapon.Build.cs` | Dépendances runtime (AnimGraphRuntime, AnimationCore, Niagara, UMG, EngineCameras…). |
| `Source/QWeapon/Public/QWeaponModule.h` / `Private/QWeaponModule.cpp` | Module runtime ; `LogQWeapon` ; startup/shutdown vides. |
| `Source/QWeapon/Public/QWeaponTypes.h` | Miroirs `uint8` des enums ALS (RotationMode/OverlayState/ViewMode/…) + helpers `IsPistolOrBow`/`IsPistol1H`. Header-only. |
| `Source/QWeapon/Public/QWeaponAnimInstance.h` / `Private/QWeaponAnimInstance.cpp` | `UQWeaponAnimInstance` + `FQWeaponAnimProxy` : remplace l'event graph, calcule l'état de pose, pilote rechargement/pointage, lit le personnage par BPI/réflexion. |
| `Source/QWeapon/Public/AnimNode_QWeaponEvaluate.h` / `Private/AnimNode_QWeaponEvaluate.cpp` | `FAnimNode_QWeaponEvaluate` : la chaîne de manipulation d'os (spine/tête/mains/ik_hand_gun), TwoBoneIK + FABRIK maison, blends aim/ready/pointing, post-pass grip de reload. (~1771 lignes — le gros du plugin.) |
| `Source/QWeapon/Public/QWeaponBulletSubsystem.h` / `Private/QWeaponBulletSubsystem.cpp` | `UQWeaponBulletSubsystem` : hitscan autoritaire (`ServerFireBullet`) + traçantes/FX cosmétiques pooled (`SpawnBulletTracer`), gouverneurs de foule, toutes les CVars `QWeapon.*`. |
| `Source/QWeapon/Public/QWeaponFireCameraShake.h` / `Private/QWeaponFireCameraShake.cpp` | `UQWeaponFireCameraShake` : shake rotationnel + FOV punch par tir, tireur local. |
| `Source/QWeaponEditor/QWeaponEditor.Build.cs` | Module éditeur (dépend de QWeapon + AnimGraph/BlueprintGraph/UnrealEd). |
| `Source/QWeaponEditor/Private/QWeaponEditorModule.cpp` | Module éditeur minimal. |
| `Source/QWeaponEditor/Public/AnimGraphNode_QWeaponEvaluate.h` / `Private/AnimGraphNode_QWeaponEvaluate.cpp` | `UAnimGraphNode_QWeaponEvaluate` : wrapper éditeur du nœud (titre/tooltip/catégorie). |
| `Source/QWeaponEditor/Public/QWeaponAnimGraphSetup.h` / `Private/QWeaponAnimGraphSetup.cpp` | `UQWeaponAnimGraphSetup::SetupWeaponAnimGraph` : câblage automatique du graphe d'anim de référence (WITH_EDITOR). |
