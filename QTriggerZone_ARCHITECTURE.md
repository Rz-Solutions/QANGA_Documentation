# QTriggerZone — Architecture

> Plugin (auteur : RzZz) de **zones de déclenchement sphériques** pour la quête et l'ambiance : zones de **son** (déléguées à `QAmbientAudio`), de **téléport** et **custom**. Deux façons de poser une zone — un acteur autonome (`ATriggerZoneActor`) ou un composant multi-zones (`UTriggerZoneComponent`) — plus un **module éditeur** dédié (widget Slate). La détection est **locale au joueur** (non répliquée).

---

## 1. But d'usage

QANGA a besoin de réactions spatiales légères : « quand le joueur entre dans ce rayon, démarre cette ambiance / téléporte-le / notifie la quête ». Plutôt qu'un `ATriggerVolume` + collision overlap par acteur, `QTriggerZone` propose :

- des zones **purement géométriques** (centre + rayon), testées par distance au joueur local sur un **timer lent** (`CheckInterval = 0.1 s`) — pas de corps de collision, pas d'overlap réseau ;
- une intégration **audio** déléguée à `QAmbientAudio` (les zones de son ne jouent pas un `USoundBase` brut, elles **demandent une entrée d'ambiance** au manager) ;
- un **workflow d'édition** confortable : poser un `ATriggerZoneActor` dans le niveau et l'éditer visuellement, ou gérer un lot de zones via un composant et un éditeur Slate.

C'est un système **côté présentation/ambiance et quête locale** : il agit sur le pawn **local**, rien n'est répliqué.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Deux points d'entrée, une même donnée.** `ATriggerZoneActor` = une zone unique posée dans le monde (avec visualisation éditeur). `UTriggerZoneComponent` = N zones gérées sur un acteur hôte (généralement piloté par la quête). Les deux partagent la struct `FQTriggerZoneData`.
- **Détection par distance, pas par collision.** Pas d'`overlap` physique : un timer (`CheckInterval`) appelle `CheckActiveTriggerZones()`, qui compare la position du **pawn local** (`GetLocalPlayerPawn`) au centre/rayon de chaque zone (`IsPlayerInZone`, distance au carré). Cela évite tout corps de collision et tout trafic réseau.
- **Audio délégué à `QAmbientAudio`.** Une zone de son porte un `UQAmbientDefinition` + `AmbientEntryId` + `AmbientPriority` ; à l'entrée, le composant **demande** cette entrée à l'`AQAmbientAudioManager` (et conserve un `FGuid` de requête dans `ZoneRequestIds` pour pouvoir l'annuler à la sortie). Le fade/crossfade et la priorité sont gérés par `QAmbientAudio`, pas réimplémentés ici.
- **Intégration quête découplée.** Les hooks quête prennent des `UObject*` opaques (`InitializeFromQuestDefinition(UObject*)`, `SyncWithQuestRuntimeState(UObject*)`, `UpdateQuestRuntimeState(UObject*)`) — **aucune dépendance de compilation vers `DynamicQuestSystem`** (le `Build.cs` ne dépend que de `QAmbientAudio`). Le lien quête se fait par `OwnerQuestID` / `ZoneID` et réflexion côté appelant.
- **Module éditeur séparé.** `QTriggerZoneEditor` (type Editor) fournit `STriggerZoneEditor` (widget Slate) pour gérer les zones, avec synchronisation acteur ↔ éditeur (`SyncDataToTriggerZoneEditor`, `SyncDataFromActorToEditor`).

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `ATriggerZoneActor` | `AActor` (`BlueprintType`) | Zone unique posée dans le niveau, visualisation éditeur (sphère + billboard + matériau par état) |
| `UTriggerZoneComponent` | `UActorComponent` (ClassGroup `Quest`, `BlueprintSpawnableComponent`) | Gestion de N zones sur un hôte, timer de détection, pont audio + quête |
| `FQTriggerZoneData` | `USTRUCT` (`BlueprintType`) | Config d'une zone : type, centre, rayon, son (ambiance/fade), téléport, drapeaux |
| `FQTriggerZoneExtendedData` | `USTRUCT` (`BlueprintType`) | `BaseData` + état runtime (`bHasBeenTriggered`, `bPlayerCurrentlyInside`, `LastTriggerTime`, `TriggerCount`) |
| `FQTriggerZoneRuntimeState` | `USTRUCT` (`BlueprintType`) | État persistable (sauvegarde) par `ZoneID` |
| `FActiveTriggerZoneInfo` | `USTRUCT` (`BlueprintType`) | Entrée de suivi runtime (`ZoneData` + `bPlayerInside`) |
| `EQTriggerZoneType` | `UENUM` | `Sound` / `Teleport` / `Custom` |
| `EQSoundStartEvent` / `EQSoundStopEvent` / `EQSoundStopMethod` | `UENUM` | Quand démarrer / arrêter le son, et comment (`Immediate`/`Fade`/`Crossfade`) |
| `ETriggerZoneActorState` | `UENUM` | État visuel de l'acteur : `Inactive`/`Active`/`Triggered`/`PlayerInside` |
| `STriggerZoneEditor` | Widget Slate (module Editor) | Outil d'édition des zones |

**Délégués (`UTriggerZoneComponent`)** : `OnTriggerZoneEntered(FQTriggerZoneExtendedData, APawn*)`, `OnTriggerZoneExited(FQTriggerZoneExtendedData, APawn*)`.

---

## 4. Flux de données et cycle de vie

1. **Définition** — un designer pose un `ATriggerZoneActor` (édité dans le viewport, état visuel par matériau) **ou** remplit le tableau `TriggerZones` d'un `UTriggerZoneComponent` (manuellement ou via `InitializeFromQuestDefinition` / `InitializeFromTriggerZoneArray`).
2. **Activation** — `BeginPlay` / `ActivateComponent()` démarre le timer (`StartTriggerZoneTimer`) ; les zones passent dans `ActiveTriggerZones` (map `ZoneID → FActiveTriggerZoneInfo`).
3. **Détection** — toutes les `CheckInterval`, `CheckActiveTriggerZones()` teste la position du pawn local contre chaque zone. Une transition dehors→dedans appelle `HandlePlayerEnteredZone` (diffuse `OnTriggerZoneEntered`, incrémente `TriggerCount`, met `bPlayerCurrentlyInside`), dedans→dehors appelle `HandlePlayerExitedZone`.
4. **Action selon le type** :
   - **Sound** : `TriggerSoundZone` demande l'entrée d'ambiance (`AmbientDefinition`/`AmbientEntryId`/`AmbientPriority`) à l'`AQAmbientAudioManager` (`GetAmbientManager`, caché dans `CachedAmbientManager`), stocke le `FGuid` de requête ; l'arrêt suit `SoundStopEvent`/`SoundStopMethod`.
   - **Teleport** : `TriggerTeleportZone` déplace le joueur vers `TeleportDestination`.
   - **Custom** : point d'extension (logique projet).
5. **Quête** — `NotifyQuestSystemOfTriggerZoneStateChange` / `UpdateQuestRuntimeState` propagent l'état (déclenché/visité) vers l'objet quête fourni ; `bOnlyTriggerOnce` empêche les re-déclenchements.
6. **Persistance** — les champs `SaveGame` (comportement son) et `FQTriggerZoneRuntimeState` permettent de sauvegarder « zone déjà déclenchée » (ex. son joué une seule fois).
7. **Désactivation / EndPlay** — `DeactivateComponent` / `StopTriggerZoneTimer` + `CleanupRequests` annulent les requêtes audio en cours.

---

## 5. Points d'intégration

- **Dépendances module (runtime)** : `Core`, `CoreUObject`, `Engine`, `QAmbientAudio` (public) ; `AudioMixer`, `AudioPlatformConfiguration`, `NavigationSystem` (privé).
- **QAmbientAudio** : dépendance centrale. Les zones de son consomment `UQAmbientDefinition` / `QAmbientAudioTypes` et pilotent l'`AQAmbientAudioManager`. Voir [QAmbientAudio_ARCHITECTURE](QAmbientAudio_ARCHITECTURE.md).
- **Système de quête** : intégration **par interface opaque** (`UObject*`) + `OwnerQuestID`/`ZoneID` ; pas de hard-link vers `DynamicQuestSystem`.
- **Module éditeur** `QTriggerZoneEditor` : dépend de `UnrealEd`, `PropertyEditor`, `ToolMenus`, `AssetTools`, etc., et expose `STriggerZoneEditor` + les délégués de `TriggerZoneEditorDelegates.h`.

---

## 6. Gotchas, invariants et pièges

- **Détection locale, non répliquée.** Aucune `UPROPERTY` répliquée ni RPC dans le plugin : une zone agit sur le **pawn local** de chaque machine. Pour une logique serveur-autoritaire (anti-triche, état partagé), il faut router le résultat par le système de quête, pas se reposer sur la zone.
- **Le CLAUDE.md du plugin est en partie obsolète.** Il mentionne une dépendance plugin `DynamicQuestSystem` ; la réalité du `Build.cs`/`.uplugin` est une dépendance `QAmbientAudio` et des hooks quête génériques. Se fier au code.
- **Les zones de son ne jouent pas un son directement.** Elles **demandent une entrée d'ambiance** à `QAmbientAudio` ; si la `AmbientDefinition`/`AmbientEntryId` est invalide ou le manager absent, rien ne joue (silencieux, pas d'erreur bloquante).
- **Coût = nb de zones × fréquence du timer.** `CheckInterval = 0.1 s` sur beaucoup de zones multiplie les tests de distance ; regrouper les zones par composant et augmenter l'intervalle pour les zones non critiques.
- **Tick éditeur actif.** `ATriggerZoneActor::ShouldTickIfViewportsOnly()` renvoie `true` pour la visualisation live dans l'éditeur — attention si l'on ajoute du travail lourd dans le `Tick` `#if WITH_EDITOR`.
- **`bRequirePlayerAction` / `Custom`** : ce sont des points d'extension ; vérifier qu'une logique projet les implémente avant de s'y fier.

---

## 7. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QTriggerZone/Public/Data/TriggerZoneTypes.h` | Enums + structs (`FQTriggerZoneData`, `…ExtendedData`, `…RuntimeState`) |
| `Source/QTriggerZone/Public/Components/TriggerZoneComponent.h` / `Private/…cpp` | Composant multi-zones, timer, pont audio + quête |
| `Source/QTriggerZone/Public/Actors/TriggerZoneActor.h` / `Private/…cpp` | Acteur zone unique + visualisation éditeur |
| `Source/QTriggerZone/Public/Data/TriggerZoneEditorDelegates.h` | Délégués de synchro acteur ↔ éditeur |
| `Source/QTriggerZoneEditor/Public/Widgets/STriggerZoneEditor.h` / `Private/…cpp` | Widget Slate d'édition des zones |
| `QTriggerZone.uplugin` | Descripteur : module Runtime `QTriggerZone` + module Editor `QTriggerZoneEditor`, catégorie `Quest` |
