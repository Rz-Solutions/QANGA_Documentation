# QNet — Architecture

> Plugin runtime (auteur : Iolacorp) : un **composant de réplication de mouvement** sur mesure (`UQNet_MovementComponent`), alternatif au mouvement répliqué natif d'Unreal. Optimisé pour la **bande passante** (envoi conditionné par tolérance/seuil, encodage bit-packé 24 bits, cadence serveur dépendante de la distance) et pour les besoins **espace relatif** de QANGA (transform relatif à un acteur mobile : plateforme, vaisseau, planète).

---

## 1. But d'usage

Répliquer le mouvement d'acteurs non-personnages (véhicules, props, objets physiques) à grande échelle, sur des serveurs dédiés peuplés, sans saturer la bande passante ni payer le coût du mouvement répliqué générique d'Unreal. Quatre exigences propres à QANGA :

1. **Économie réseau** : n'envoyer que quand l'état change au-delà d'une **tolérance**, à une **cadence variable** selon la distance au client pertinent, avec un **encodage compact** (quantification 24 bits).
2. **Espace relatif** : un objet posé sur une plateforme/vaisseau/planète en mouvement doit répliquer un transform **relatif** à ce porteur, sinon il dérive (mondes planétaires LWC en translation permanente).
3. **Autorité flexible** : un même composant peut être **piloté par le serveur** ou **par un client propriétaire** (conduite de véhicule), avec bascule d'autorité runtime.
4. **Lissage client** : interpolation + prédiction (extrapolation par ping) pour masquer la latence.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Deux représentations du mouvement.** `FQNet_Movement_Struct` = état complet précision pleine (location, rotation, velocity, angular velocity, + variantes *relative*). `FQNet_Movement_Data` = **format fil** compact (`FVector3f`/`FRotator3f`, quantification 24 bits) avec un **`EncodeFlag`** (bitfield : quels champs sont présents, relatif ou non, 24 vs 32 bits) et un **`NetSerialize`** custom qui bit-packe les champs activés.
- **Envoi conditionné.** Avant de répliquer, le composant compare le nouvel état au dernier envoyé via **tolérances** (`Location_Tolerance`, `Rotation_Tolerance`, `Velocity_Tolerance`…) et **seuils** (`Get_Threshold`, `Location_Threshold_Distance`). Pas de changement significatif → pas d'envoi. Un « sub-send » peut intervenir à intervalle minimal (`SubSend_MinimalTickInterval`) sur franchissement de seuil.
- **Cadence serveur dynamique.** Si `Dynamic_ServerTick_IntervalByDistance`, l'intervalle d'envoi serveur s'interpole entre `Dynamic_ServerTick_MinimalInterval` (proche, ~500 m) et `Dynamic_ServerTick_MaxInterval` (loin, ~1200 m) — les objets lointains coûtent moins cher.
- **Autorité client-propriétaire optionnelle.** `OwnerPlayerController` (répliqué) désigne le propriétaire ; un client propriétaire envoie son mouvement au serveur par **RPC direct** (`Owner_DirectSend_Data`, Server Reliable) si `Use_DirectRPC_ClientToServer`. `QNet_ChangeOwner` bascule l'autorité ; `Auto_ChangeOwner_OnParentActor` la dérive du parent.
- **Transform relatif répliqué.** `RelativeToActor` (répliqué, OnRep) : quand défini, location/rotation sont exprimées **relativement** à cet acteur (`Location_Relative`, `Rotation_Relative`). `QNet_ChangeRelative` (via `Client_CallChange_Relative`, Server Reliable) le change. Indispensable pour rester collé à un porteur mobile.
- **Lissage + prédiction.** Côté client, interpolation paramétrable (`*_Interp_Speed`, `*_Factor`, `*_Damping`) et prédiction par ping (`Apply_Prediction`, `Prediction_Ping_Factor`) extrapolent entre deux updates.
- **Ticks par rôle.** Intervalles/ticking-groups distincts pour serveur (`Server_Tick_Interval = 0.15`), client simulé (`Client_Tick_Interval`), client propriétaire (`OwnerClient_Tick_Interval = 0.1`), tous en `TG_PostUpdateWork` par défaut. Tick désactivé en standalone sauf `EnabledTick_OnStandAlone`.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQNet_MovementComponent` | `UActorComponent` (`BlueprintSpawnableComponent`) | Réplication de mouvement : envoi conditionné, encodage compact, autorité, relatif, lissage |
| `FQNet_Movement_Struct` | `USTRUCT` (`BlueprintType`) | État complet précision pleine (abs + relatif) |
| `FQNet_Movement_Data` | `USTRUCT` (`BlueprintType`) | Format fil compact (`EncodeFlag` + `NetSerialize` bit-packé 24 bits) |
| `EQNet_MovementMode` | `UENUM` | `Simulate_VelocityAndAngularVelocity` / `Simulate_Velocity` / `Simulate_AngularVelocity` / `Direct_InterpTransform` |

**Délégués** : `QNet_UpdateMovement(FQNet_Movement_Struct)`, `QNet_OwnerChange(APlayerController*)`, `QNet_RelativeChange(AActor*)`.

---

## 4. Réplication / réseau

- **Propriétés répliquées** : `Server_Data` (`FQNet_Movement_Struct`, Replicated), `ServerToClient_Data` (`FQNet_Movement_Data`, `ReplicatedUsing = OnRep_ServerMovementChange`), `OwnerPlayerController` (`OnRep_OwnerChange`), `RelativeToActor` (`OnRep_RelativeToActorChange`).
- **RPC serveur (Reliable)** : `Owner_DirectSend_Data(FQNet_Movement_Data)` (client propriétaire → serveur), `Client_CallChange_Relative(AActor*)`.
- **Chemins de mise à jour** (par rôle, par frame) : `Update_Server_Movement`, `Update_Client_Movement`, `Update_ServerToClient_Movement`, `Update_ClientToServer_Movement`, `Update_Movement_Local`. Le serveur encode (`Encode_ReplicatedData`) vers `ServerToClient_Data` ; les clients décodent (`Decode_ReplicatedData`) en compensant le ping (`Get_ClientPing`).
- **Encodage** : `NetSerialize` n'écrit que les champs activés par `EncodeFlag`, en 24 bits quand la précision réduite suffit — gain net de bande passante par rapport au mouvement répliqué standard.

> Différence avec `NinjaCharacter` (SmoothSync, personnages) : `QNet` cible les **acteurs non-personnages** et expose le contrôle fin de l'autorité, du relatif et de l'encodage. Voir aussi [`QNetState`](QNetState_ARCHITECTURE.md) pour la réplication d'**état** (et non de mouvement).

---

## 5. Flux de données et cycle de vie

1. **BeginPlay** — résolution du rôle (`NetMode`), de l'acteur hôte, du `Local_PlayerController` ; enregistrement des intervalles de tick par rôle ; init de l'autorité (`OwnerPlayerController`, `Auto_ChangeOwner_OnParentActor`).
2. **Capture** — si `Auto_GetTransformVelocity_FromOwnerActor`, le composant lit le transform/vélocité de l'acteur (`Auto_GetTransform`) dans un `FQNet_Movement_Struct`.
3. **Décision d'envoi** — `Get_Tolerance` / `Get_Threshold` décident s'il faut répliquer ; sinon on saute (économie réseau).
4. **Encodage + envoi** — `Encode_ReplicatedData` produit un `FQNet_Movement_Data` compact ; le serveur le réplique (`ServerToClient_Data`), ou le client propriétaire l'envoie par RPC (`Owner_DirectSend_Data`).
5. **Réception** — `OnRep_ServerMovementChange` / le RPC décodent (`Decode_ReplicatedData`), compensent le ping, et alimentent l'interpolation.
6. **Application** — chaque frame, l'interpolation/prédiction avance l'état et, si `Auto_SetTransformVelocity_ToOwnerActor`, applique le transform à l'acteur (`Auto_SetTransform`, avec `UseSweep`/`Teleport`). Diffuse `QNet_UpdateMovement`.

---

## 6. Points d'intégration

- **Dépendances module** : `Core`, `DeveloperSettings`, `RenderCore` (public) ; `CoreUObject`, `Engine`, `Slate`, `SlateCore` (privé). Aucune dépendance vers d'autres plugins QANGA.
- **Consommateurs** : véhicules, props physiques, objets transportés — tout acteur non-personnage dont le mouvement doit être répliqué efficacement, en particulier en **espace relatif** (sur plateforme/vaisseau/planète mobile).
- **Espace relatif** : se combine avec les systèmes de portage/plateforme (`QElevator`, `QTrain`, véhicules) via `QNet_ChangeRelative`.

---

## 7. Gotchas, invariants et pièges

- **Quantification 24 bits = précision réduite.** L'encodage compact réduit la précision de rotation/vélocité ; pour un objet exigeant la pleine précision, activer les bits 32 bits via `EncodeFlag` (au prix de la bande passante).
- **Le relatif doit pointer un acteur valide et cohérent.** Si `RelativeToActor` est détruit ou désynchronisé entre client/serveur, l'objet dérive. Toujours router le changement via `QNet_ChangeRelative` (qui passe par le serveur).
- **Tolérances trop hautes = saccades.** Des tolérances/seuils élevés économisent la bande passante mais peuvent produire des sauts visibles ; équilibrer avec l'interpolation.
- **Autorité explicite.** En mode client-propriétaire, c'est le client qui pousse son mouvement (RPC) ; mal configurer l'autorité (`OwnerPlayerController`) provoque des combats d'autorité ou des objets figés.
- **Tick par rôle.** Les intervalles diffèrent serveur/client/owner ; un comportement « marche en PIE, pas en dédié » vient souvent d'un tick désactivé en standalone (`EnabledTick_OnStandAlone`) ou d'un rôle mal détecté.
- **`NetSerialize` custom.** `FQNet_Movement_Data` implémente `NetSerialize` (bit-packing) ; toute évolution du format de `EncodeFlag` doit rester compatible entre client et serveur de même version.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QNet/Public/Component/QNet_MovementComponent.h` / `Private/…cpp` | Composant de réplication de mouvement (logique complète) |
| `Source/QNet/Public/Component/QNet_Movement_Struct.h` | `FQNet_Movement_Struct` (plein) + `FQNet_Movement_Data` (fil, `NetSerialize`) |
| `QNet.uplugin` | Descripteur : module Runtime `QNet` |
