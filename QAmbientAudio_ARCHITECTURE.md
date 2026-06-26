# QAmbientAudio — Architecture

> Plugin runtime (auteur : QANGA) : un **gestionnaire d'ambiance audio réseau** à pile de priorités, lecture MetaSound côté client **synchronisée sur l'horloge serveur**, transitions contrôlées (fade/crossfade) et sources spatialisées. C'est le socle audio dont s'inspire `QRadio`, et la dépendance audio de `QTriggerZone`.

---

## 1. But d'usage

QANGA a besoin d'une ambiance sonore **cohérente entre joueurs** et **pilotable par le gameplay** : un fond d'ambiance par défaut (vent, ville…), que des événements peuvent **temporairement surcharger** (alarme, zone spéciale), avec retour propre à l'ambiance précédente. Trois exigences :

1. **Autorité serveur + même rendu partout.** L'état d'ambiance est décidé côté serveur et répliqué ; chaque client le joue localement (l'audio n'est jamais streamé sur le réseau), **synchronisé sur l'horloge serveur** pour que tout le monde entende la même position de lecture.
2. **Empilement par priorité.** Une demande haute priorité couvre l'ambiance courante puis, en se terminant, la dépile et restaure la précédente — d'où une **pile de couches** (`LayerStack`).
3. **Transitions douces** : fade in/out, crossfade entre deux ambiances, sources spatialisées (global, position monde, attaché à un acteur/composant).

---

## 2. Vue d'ensemble / décisions d'architecture

- **Un manager `AInfo` répliqué.** `AQAmbientAudioManager` (acteur léger `AInfo`) porte tout l'état serveur et **un seul `UPROPERTY` répliqué** : `ActiveState` (`FQAmbientReplicatedState`). Tous les détails (définition active, entrée, fades, volume, heure de départ serveur, état, source) tiennent dans cette struct unique.
- **Pile de couches, pas une seule ambiance.** Le serveur maintient `LayerStack` (tableau de `FQAmbientLayerEntry`) ordonné par priorité ; la couche de tête détermine l'`ActiveState` répliqué. `PushLayer` / `PopLayer` ajoutent/retirent ; une couche peut expirer (`Duration` → `HandleExpire`).
- **Lecture côté client, synchro horloge.** `UQAmbientPlaybackComponent` (sur le manager) reçoit l'état répliqué et joue le MetaSound localement. Il calcule un **offset de départ** à partir de `ServerStartTime` (`CalculateStartTimeOffset`) pour que la lecture soit alignée sur l'horloge serveur — déterminisme inter-clients (même mécanique que `QRadio`).
- **Crossfade par double composant audio.** Le playback maintient deux `UAudioComponent` (`PrimaryAudio` / `SecondaryAudio`) et alterne (`SwapActiveAudio`) pour fondre l'ancien dans le nouveau.
- **Ambiances en data asset.** `UQAmbientDefinition` (`UPrimaryDataAsset`) regroupe des entrées `FQAmbientEntry` (un MetaSound + atténuation + concurrency + comportement spatial + fades/volume par défaut), indexées par `FName` (lookup `EntryIndexById`).
- **Chargement asynchrone.** Les définitions sont des `TSoftObjectPtr` chargées via `FStreamableHandle` ; le playback gère l'attente (`HandleLoadedDefinition`, `RetryPendingState`).
- **Emetteurs placés = demandeurs.** `UQAmbientEmitterComponent` ne joue rien lui-même : il **demande** une couche au manager (`bServerOnly` par défaut), ce qui garde l'autorité côté serveur.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `AQAmbientAudioManager` | `AInfo` | Manager serveur-autoritaire : pile de couches, état répliqué, RPC, transitions |
| `UQAmbientPlaybackComponent` | `UActorComponent` | Lecture client : MetaSound, crossfade double-composant, synchro horloge serveur |
| `UQAmbientEmitterComponent` | `UActorComponent` (`BlueprintSpawnableComponent`) | Emetteur placé : demande/retire une couche d'ambiance au manager |
| `UQAmbientDefinition` | `UPrimaryDataAsset` | Catalogue d'entrées d'ambiance (`FQAmbientEntry`), lookup par `EntryId` |
| `FQAmbientEntry` | `USTRUCT` | Une ambiance : `UMetaSoundSource` + atténuation + concurrency + source/fades/volume par défaut |
| `FQAmbientRequest` | `USTRUCT` | Demande d'ambiance : définition, entrée, priorité, durée, fade, source |
| `FQAmbientReplicatedState` | `USTRUCT` | État répliqué : définition active, entrée, fades, volume, `ServerStartTime`, `PlayState`, `RequestId`, priorité, source |
| `FQAmbientLayerEntry` | `USTRUCT` | Couche de la pile serveur (request résolue + `RequestId` + `ExpireHandle` + ancre) |
| `FQAmbientSource` | `USTRUCT` | Spatialisation : mode + transform + acteur/composant d'attache |
| `FQAmbientFadeSettings` | `USTRUCT` | Fade in/out, volume cible, options « utiliser les valeurs de l'entrée » |
| `EQAmbientSourceMode` | `UENUM` | `Global` / `WorldLocation` / `AttachActor` / `AttachComponent` |
| `EQAmbientPlayState` | `UENUM` | `Stopped` / `Playing` / `Paused` |

---

## 4. Flux de données et cycle de vie

1. **Demande** — un appelant (emetteur placé, zone `QTriggerZone`, gameplay) appelle `RequestOverride(FQAmbientRequest)` ou `SetDefaultAmbience(...)` sur le manager. Côté client, ces wrappers `BlueprintCallable` routent vers les **RPC serveur** correspondants.
2. **Empilement serveur** — le serveur `PushLayer(Request)` : résout la définition (`ResolveDefinition`), fusionne la source (`MergeSource`), arme une expiration si `Duration > 0`, et réordonne la pile par priorité. La couche de tête devient l'ambiance active (`ApplyActiveLayer`) → met à jour `ActiveState` (avec `ServerStartTime = GetServerTime()`).
3. **Réplication** — `ActiveState` se réplique ; les clients reçoivent `OnRep_AmbientState(PreviousState)` → `BroadcastStateChange` → `UQAmbientPlaybackComponent::ApplyReplicatedState(New, Previous)`.
4. **Lecture client** — le playback charge la définition (async si soft), choisit le composant audio inactif, calcule l'offset de départ depuis `ServerStartTime`, applique atténuation/concurrency/source, puis fond l'ancienne ambiance (`FadeOutActive`) et démarre la nouvelle (crossfade via `SwapActiveAudio`).
5. **Pause / Resume / Stop** — `PauseActive` / `ResumeActive` / `StopActive` (→ RPC) modifient le `PlayState` répliqué ; les clients appliquent `ApplyPause` / `ApplyResume` / `StopAll`.
6. **Expiration / dépilage** — à la fin d'une couche (timer ou `StopOverrideById`), `PopLayer` la retire ; la couche suivante de la pile redevient active, restaurant l'ambiance précédente avec fade.

---

## 5. Réplication / réseau

- **Modèle serveur-autoritaire à état unique.** Une seule propriété répliquée : `AQAmbientAudioManager::ActiveState` (`UPROPERTY(ReplicatedUsing = OnRep_AmbientState)`). La pile de couches (`LayerStack`) vit **uniquement côté serveur** ; seul son sommet (l'état actif) est répliqué.
- **RPC serveur fiables** pour toutes les mutations : `ServerSetDefaultAmbience`, `ServerRequestOverride`, `ServerPauseActive`, `ServerResumeActive`, `ServerStopActive`, `ServerStopOverrideById`. Les méthodes `BlueprintCallable` publiques sont les façades qui appellent ces RPC.
- **Audio jamais répliqué.** Seul l'**état** (quoi jouer, depuis quand, comment fondre) traverse le réseau ; le son est rendu localement par chaque client.
- **Synchronisation par horloge serveur.** `ServerStartTime` (heure serveur du démarrage de la couche) permet à chaque client de calculer la même position de lecture, indépendamment de sa latence. C'est la clé de la cohérence inter-joueurs.

---

## 6. Points d'intégration

- **Dépendances module** : `Core`, `CoreUObject`, `Engine`, `MetasoundEngine` (public) ; `AudioMixer` (privé). Plugin requis : `Metasound`.
- **Consommateurs** :
  - [`QTriggerZone`](QTriggerZone_ARCHITECTURE.md) : les zones de son demandent des entrées d'ambiance au manager.
  - [`QRadio`](QRADIO_ARCHITECTURE.md) : repose sur la même mécanique de synchro par horloge serveur (`GetServerWorldTimeSeconds`).
  - Emetteurs placés (`UQAmbientEmitterComponent`) et tout gameplay voulant pousser une ambiance.
- **Données** : les ambiances sont des `UQAmbientDefinition` (`UPrimaryDataAsset`) — pensez à les enregistrer comme Primary Assets (`FPrimaryAssetId("AmbientDefinition", …)`) pour le chargement asynchrone.

---

## 7. Gotchas, invariants et pièges

- **Un seul manager fait autorité.** `AQAmbientAudioManager` est un `AInfo` ; le système suppose une instance qui porte l'état. Les emetteurs/zones le retrouvent (`FindAmbientManager`) — s'il est absent, rien ne joue.
- **`MetaSound` requis.** Les entrées pointent un `UMetaSoundSource` ; sans le plugin Metasound activé, la lecture échoue.
- **Synchro = `ServerStartTime`.** Toute logique de position de lecture dépend de l'horloge serveur ; ne pas démarrer une ambiance en se basant sur une horloge locale sous peine de désync entre clients.
- **Throttle de logs sur entrées manquantes.** Le manager garde `MissingEntryHashes` pour ne pas spammer le log quand une requête référence un `EntryId` inexistant — une ambiance « muette » peut donc cacher une mauvaise `EntryId`/définition ; vérifier le data asset.
- **Priorité décide tout.** Deux demandes de même priorité : c'est l'ordre d'empilement qui tranche. Donner des priorités distinctes aux ambiances qui ne doivent pas se voler la tête.
- **`bServerOnly` sur l'emetteur.** Par défaut un `UQAmbientEmitterComponent` ne déclenche sa demande que côté serveur ; le passer à client-only romprait l'autorité.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QAmbientAudio/Public/QAmbientAudioManager.h` / `Private/…cpp` | Manager : pile de couches, état répliqué, RPC, transitions |
| `Source/QAmbientAudio/Public/QAmbientPlaybackComponent.h` / `…cpp` | Lecture client : MetaSound, crossfade, synchro horloge |
| `Source/QAmbientAudio/Public/QAmbientEmitterComponent.h` / `…cpp` | Emetteur placé (demandeur de couche) |
| `Source/QAmbientAudio/Public/QAmbientDefinition.h` / `…cpp` | Data asset d'ambiances (`UQAmbientDefinition` + `FQAmbientEntry`) |
| `Source/QAmbientAudio/Public/QAmbientAudioTypes.h` | Enums + structs (request, source, fade, état répliqué) |
| `QAmbientAudio.uplugin` | Descripteur : Runtime, plugin `Metasound` |
