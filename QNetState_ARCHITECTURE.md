# QNetState — Architecture (« Optimized State »)

> Plugin runtime (auteur : Iolacorp) : un système de **réplication d'état à la demande, indexée par localisation**. Au lieu de répliquer en continu chaque acteur d'état (porte, ascenseur, interrupteur, machine), chaque acteur porte un `UOptimizedStateComponent` adressé par une **clé de localisation quantifiée** (`FOptimizedStateKey`) ; les clients **demandent** les états dont ils ont besoin (à proximité, au spawn), un **manager serveur** y répond, et des **acteurs forcés** peuvent être spawnés côté client pour montrer l'état d'objets non répliqués. C'est le composant auquel `QElevator` se lie par réflexion.

---

## 1. But d'usage

Sur des planètes entières peuplées de milliers d'objets d'état (portes, ascenseurs, leviers, panneaux), la réplication standard d'Unreal ne passe pas à l'échelle : rendre chaque acteur **net-relevant** et répliqué en permanence sature la bande passante et la liste de pertinence. Pourtant, quand un joueur approche, il doit voir l'**état courant** (porte ouverte/fermée, position de l'ascenseur sur son trajet).

`QNetState` (« Optimized State ») résout ça en **découplant l'état de la réplication d'acteur** :

- l'état d'un objet est un ensemble de **paires clé-valeur typées** (`Bool`, `Int`, `Timeline`, `Transform`…) portées par un `UOptimizedStateComponent` ;
- chaque composant est **adressé par sa localisation** (`FOptimizedStateKey` = `int64` X/Y/Z dérivé de la position), pas par un NetGUID — un client peut donc demander « l'état de ce qui se trouve ici » même si l'acteur n'est pas répliqué ;
- les états sont **tirés à la demande** par un `URequesterOptimizedState` (sur le PlayerController), répondus par un `UOptimizedStateManager` (serveur), en messages **batchés et compacts** ;
- pour les objets qui doivent exister visuellement mais ne se répliquent pas, le manager peut **forcer leur spawn** côté client (`FSpawnForcedActorsClient`).

---

## 2. Vue d'ensemble / décisions d'architecture

- **État typé clé-valeur.** `UOptimizedStateComponent` stocke des maps par type (`BoolKey_DefaultValue`, `IntKey_DefaultValue`, … `TimelineKey_DefaultValue`) et un map de `UOptimizedStateValueObject` (`ValueObjects`) portant les délégués de mise à jour par type. API symétrique : `SetBool/SetTimeline/…`, `GetBool/GetTimeline/…`, `BindToBool/BindToTimeline/…`.
- **Adressage par localisation.** `FOptimizedStateKey` (`int64` X/Y/Z, quantifié depuis une `FVector`, `FromVector`/`ToVector`, hashable) sert de **clé de registre**. `bUseCustomLocationKey` + `SetCustomLocationKey` permettent une clé custom (objets posés par `QBuilder`).
- **État « Timeline » pour le mouvement.** Plutôt que de répliquer un transform en continu, un état `Timeline` (`FTimelineValue` = `Time` + `EQTimelineState` : `Stopped`/`Playing`/`Reversing`/`Paused`) décrit une progression d'animation/mouvement. Combiné au **temps serveur** (`SyncTime`, répliqué), chaque client rejoue la même position (c'est ainsi que `QElevator` partage la position de sa plateforme).
- **Tirage à la demande, batché.** Le `URequesterOptimizedState` (client) accumule les besoins en files (`Request_Queue`, `UpdateMulticast_Queue`, `EventQueue`) et les envoie au serveur (`SV_RequestComponentsStates`) ; le serveur répond via multicast/cible (`MC_ServerSendMulticast`, `SR_…`) avec des structs compacts par type (`FOptimizedRequestBool` et son encodage `FoundAndValue` 0/1/2, `FOptimizedRequestInt`, `…TL`…).
- **Acteurs forcés.** Le `UOptimizedStateManager` tient des ensembles `ForcedSpawnActorsServer` (répliqué, `OnRep_ForcedSpawnActors`) → côté client, spawn d'un acteur (`FSpawnForcedActorsClient` : transform + `TSoftClassPtr<AActor>`) via `SpawnForcedActorHelper` (BP). Sert à matérialiser un objet non répliqué dont l'état doit être visible.
- **Couplage souple par convention.** Les consommateurs (ex. `QElevator`) trouvent le composant par **nom de classe** (`"OptimizedStateComponent"`) et appellent ses fonctions par réflexion (`BindToBool`/`GetBool`/`SetBool`/`SetTimeline`), évitant une dépendance de compilation vers `QNetState`.
- **Optimisations « hot path ».** Le code privilégie des helpers `Broadcast*Update` typés (pas de switch ni de lookup de tag de type) et a consolidé Time+Direction en un seul `FTimelineValue` pour halver les lookups (commentaires du code).

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UOptimizedStateComponent` | `UActorComponent` (`BlueprintSpawnableComponent`) | Porte l'état typé d'un acteur (Set/Get/Bind par clé), adressé par localisation |
| `UOptimizedStateManager` | `UActorComponent` (`BlueprintSpawnableComponent`) | Registre serveur (clé→composant), temps de synchro répliqué, acteurs forcés |
| `URequesterOptimizedState` | `UActorComponent` (`BlueprintSpawnableComponent`) | Client : files de demandes, RPC de requête/événement/interaction, stats |
| `UOptimizedStateValueObject` | `UObject` | Valeur d'une clé + délégués de mise à jour par type |
| `SOptimizedStateDebugWidget` | Widget Slate | Outil de debug (stats du requester) |
| `FOptimizedStateKey` | `USTRUCT` | Clé de localisation `int64` X/Y/Z (hashable) |
| `FTimelineValue` | `USTRUCT` | État timeline : `Time` + `EQTimelineState` |
| `FOptimizedStateRequest` | `USTRUCT` | Demande : clé de localisation + clé d'état + type |
| `FOptimizedRequestBool/Byte/Int/Int64/Double/Name/Rotator/Vector/Transform/TL` | `USTRUCT` | Réponses compactes par type (wire) |
| `FSpawnForcedActorsClient` | `USTRUCT` | Acteur à forcer côté client : transform + classe |
| `FOptimizedStateStats` | `USTRUCT` | Statistiques de trafic (debug) |
| `EQOptimizedStateType` | `UENUM` | `Timeline`/`Bool`/`Event`/`Int`/`Int64`/`Byte`/`Double`/`Rotator`/`Vector`/`Transform`/`Name` |
| `EQTimelineState` | `UENUM` | `Stopped`/`Playing`/`Reversing`/`Paused` |

**Délégués (composant)** : `OnStateUpdate(FName Key)`, `OnServerMulticastEvent(FName)`, `ServerReceiveEvent(FName, APlayerController*)` + délégués single-cast de bind par type (`FOnOptimizedBoolUpdateSingle`, …, `FOnOptimizedTimelineUpdateSingle`).

---

## 4. Réplication / réseau

- **Manager (serveur autoritaire)** : `UOptimizedStateManager` réplique `SyncTime` (`ReplicatedUsing=OnRep_SyncTime`, horloge de synchro des timelines) et `ForcedSpawnActorsServer` (`ReplicatedUsing=OnRep_ForcedSpawnActors`). Registre interne `TMap<FOptimizedStateKey, UOptimizedStateComponent*>`.
- **Requester (par client)** : RPC `SV_RequestComponentsStates` (Server), `MC_ServerSendMulticast` (NetMulticast), `SV_SendOptimizedEvent` (Server), `SV_OptimizedInteractServer` (Server), + un Client RPC de réponse. Les états ne transitent que **sur demande**, et en **lots typés compacts**.
- **Setters** : `Set*(Key, Value, bServerMulticast=true)` sur le composant ; côté serveur, `ServerMulticastState` met la mise à jour en file dans le requester pour propagation client.
- **Modèle** : l'**acteur** n'a pas besoin d'être répliqué ; c'est son **état** (clé-valeur) qui est servi à la demande, et au besoin son **spawn** est forcé côté client. C'est l'inverse du modèle UE « réplique l'acteur, l'état suit ».

> À ne pas confondre avec [`QNet`](QNet_ARCHITECTURE.md), qui réplique le **mouvement** continu d'acteurs (véhicules, props). `QNetState` réplique de l'**état discret** à la demande.

---

## 5. Flux de données et cycle de vie

1. **Enregistrement** — au `BeginPlay`, `UOptimizedStateComponent::ResolveManagerAndRegister` calcule sa clé (`FOptimizedStateKey` depuis sa localisation, ou clé custom) et s'enregistre auprès du `UOptimizedStateManager` (`AddOptimizedStateComponent`). Côté client, si le manager n'est pas prêt, retry différé (`DelayedBindedRequest`).
2. **Écriture d'état (serveur)** — la logique d'un objet appelle `SetBool/SetTimeline/…` ; si `bServerMulticast`, l'update est mise en file pour les clients pertinents.
3. **Demande (client)** — quand un client a besoin de l'état d'un objet (proximité, bind, spawn), `URequesterOptimizedState` empile une `FOptimizedStateRequest` et envoie `SV_RequestComponentsStates`.
4. **Réponse (serveur→client)** — le manager résout la clé (`GetValidOptimizedStateComponent`), lit les valeurs, et le requester les renvoie en lots compacts (`MC_ServerSendMulticast` / RPC client).
5. **Application + bind** — à réception, les valeurs sont écrites dans les `ValueObjects`, et les délégués de bind par clé (`BindToBool`, …) se déclenchent ; `OnStateUpdate(Key)` est diffusé.
6. **Timelines** — un état `Timeline` (Time+Direction) + `SyncTime` permet à chaque client de rejouer la même progression (ex. ascenseur, porte animée).
7. **Acteurs forcés** — si un objet doit apparaître côté client (sans réplication standard), le manager l'ajoute à `ForcedSpawnActorsServer` → `OnRep` côté client → `SpawnForcedActorHelper` instancie la classe à la transform donnée ; nettoyage via `ClientDestroyForcedActor`.

---

## 6. Points d'intégration

- **Dépendances module** : `Core`, `CoreUObject`, `Engine`, `NetCore` (public) ; `Slate`, `SlateCore` (privé). Aucune dépendance vers d'autres plugins QANGA.
- **Consommateurs** : [`QElevator`](QElevator_ARCHITECTURE.md) (bind par réflexion sur `BindToBool`/`SetBool`/`SetTimeline`), portes, leviers, machines — tout objet d'état dont la réplication continue serait trop coûteuse. Les objets posés par [`QBuilder`](QBuilder_ARCHITECTURE.md) utilisent une **clé custom** (`SetCustomLocationKey`).
- **Setup** : un `UOptimizedStateManager` doit exister (sur un acteur manager/GameState) et chaque client doit avoir un `URequesterOptimizedState` (sur son PlayerController) pour que le pull fonctionne.

---

## 7. Gotchas, invariants et pièges

- **Clé = localisation quantifiée.** `FOptimizedStateKey` dérive de la position (`int64` X/Y/Z) : deux composants trop proches peuvent **collisionner** sur la même clé. Pour les objets superposés/mobiles, utiliser une **clé custom** (`SetCustomLocationKey`, via `QBuilder`).
- **Couplage par réflexion chez les consommateurs.** `QElevator` (et autres) résolvent le composant par **nom de classe** `"OptimizedStateComponent"` et appellent ses fonctions par `ProcessEvent` ; renommer la classe ou changer les signatures de `BindToBool`/`GetBool`/`SetBool`/`SetTimeline` casse ces intégrations **silencieusement**.
- **Manager + Requester obligatoires.** Sans `UOptimizedStateManager` (serveur) ni `URequesterOptimizedState` (client), aucun état n'est servi ; les binds restent muets.
- **Pull, pas push permanent.** Les clients ne reçoivent que ce qu'ils demandent ; un objet dont aucun client n'a demandé l'état n'envoie rien. Les transitions visibles dépendent donc d'une demande déclenchée au bon moment (proximité/spawn).
- **Timelines = temps serveur.** La cohérence d'une animation répliquée dépend de `SyncTime` ; rejouer une timeline sur une horloge locale désynchronise les clients.
- **Acteurs forcés ≠ réplication.** Un acteur forcé est spawné **localement** par chaque client via un helper BP ; sa logique doit savoir qu'elle tourne en présentation pure (pas d'autorité).

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QNetState/Public/OptimizedStateComponent.h` / `Private/…cpp` | Composant d'état typé (Set/Get/Bind), enregistrement, broadcast |
| `Source/QNetState/Public/OptimizedStateManager.h` / `…cpp` | Registre serveur, `SyncTime`, acteurs forcés |
| `Source/QNetState/Public/RequesterOptimizedState.h` / `…cpp` | Client : files de demandes, RPC, stats |
| `Source/QNetState/Public/OptimizedStateValueObject.h` / `…cpp` | Valeur par clé + délégués par type |
| `Source/QNetState/Public/OptimizedStateTypes.h` | `FOptimizedStateKey`, `FTimelineValue`, requêtes wire, enums |
| `Source/QNetState/Public/OptimizedStateDebugWidget.h` / `…cpp` | Widget Slate de debug |
| `QNetState.uplugin` | Descripteur : Runtime, dépend de `NetCore` |
