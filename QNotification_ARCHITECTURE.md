# QNotification — Architecture

> Plugin runtime UI (auteur : RzZz) : un **système de notifications** complet et **purement client** (UMG/Slate). Quatre archétypes — **Toast**, **Warning**, **Progress**, **Communication** (dialogue PNJ) — empilés par **zones** et **priorités**, avec effet machine à écrire synchronisé à l'audio, médias intégrés, et une **façade statique** appelable de partout.

---

## 1. But d'usage

QANGA a besoin d'un canal unique pour afficher au joueur : des infos brèves (toast), des alertes (warning), des barres de progression (récolte, chargement) et du **dialogue PNJ** (portrait + effet typewriter). Plutôt que chaque système gère son propre widget, `QNotification` centralise :

- une **API statique** (`ShowToast` / `ShowWarning` / `ShowProgress` / `ShowCommunication`) appelable depuis n'importe quel Blueprint/C++, qui retourne un `FQNotificationResult` (succès + `NotificationID`) ;
- un **manager-widget** unique qui place chaque notification dans la bonne **zone** d'écran et applique les règles de **priorité** / file d'attente / interruption ;
- des widgets riches (médias, pagination de dialogue, typewriter synchronisé à la voix, retours d'input).

C'est un système **de présentation locale** : rien n'est répliqué ; chaque client affiche ses propres notifications.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Façade statique + singleton.** `UQNotificationManager` (un `UUserWidget`) s'auto-enregistre comme `ManagerInstance`. Les fonctions `Show*` sont **statiques** : elles construisent un `FQNotificationData` et le routent vers l'instance courante (`ShowNotificationInternal`). Pas besoin de référencer le manager pour notifier.
- **Quatre archétypes, une base commune.** Tous les widgets dérivent de `UQNotificationWidgetBase` (abstrait) qui porte la mécanique partagée : typewriter, animations entrée/sortie, auto-hide, son, séquence de dialogue, progress. Les classes concrètes (`UQToastNotificationWidget`, `UQWarningNotificationWidget`, `UQProgressNotificationWidget`, `UQCommunicationNotificationWidget`) spécialisent la présentation.
- **Zones d'écran.** Le manager expose des conteneurs `UVerticalBox` liés par `BindWidget` : `ToastZone`, `WarningZone` (optionnelle, créée à la demande via `EnsureWarningZoneExists`), `ProgressZone`, `CommunicationZone`. `ResolveZone` mappe un `EQNotificationZone` vers son conteneur.
- **Priorité + files.** `EQNotificationPriority` (`Low`/`Normal`/`High`/`Critical`) régit l'interruption ; chaque zone limite son nombre actif (`MaxNotifications`, `MaxWarningNotifications`). Les zones Communication et Warning ont une **file** (`CommunicationNotificationQueue` / `WarningNotificationQueue`) rejouée quand une place se libère.
- **Classes de widget paramétrables.** Le manager référence les widgets par `TSubclassOf<UQNotificationWidgetBase>` (Toast/Progress/Communication/Warning) — l'apparence concrète est faite en BP, le C++ gère la logique.
- **Typewriter synchronisé à l'audio.** `UQNotificationWidgetBase` calcule un débit de caractères adaptatif (`CalculateAudioSyncedCPS`, `UpdateAdaptiveSpeed`) pour que le texte finisse en même temps que la voix du PNJ — paramétré par `FQTypewriterSettings`.
- **Escape global.** `FQNotificationModule` installe un `IInputProcessor` (préprocesseur d'input) au `PostEngineInit` pour intercepter la touche Échap et déclencher `HandleEscapeDismissal` — fermeture des notifications dismissibles sans dépendre d'un focus widget.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQNotificationManager` | `UUserWidget` (`Blueprintable`) | Façade statique + singleton ; zones, priorité, files, spawn des widgets |
| `UQNotificationWidgetBase` | `UUserWidget` (`Abstract`) | Base : typewriter, animations, auto-hide, son, dialogue, progress |
| `UQToastNotificationWidget` | `UQNotificationWidgetBase` | Toast : sections instructives, média (`UMediaPlayer`/`UMediaTexture`), layout généré, Échap |
| `UQWarningNotificationWidget` | `UQNotificationWidgetBase` | Alerte haut-centre : animation « voyage vers cible », retour d'input |
| `UQProgressNotificationWidget` | `UQNotificationWidgetBase` | Barre de progression + auto-progress « chunky » |
| `UQCommunicationNotificationWidget` | `UQNotificationWidgetBase` | Dialogue PNJ : portrait, pagination, typewriter |
| `FQNotificationData` | `USTRUCT` | Payload complet d'une notification (contenu + comportement + style) |
| `FQNotificationResult` | `USTRUCT` | Retour des `Show*` : `bSuccess`, `NotificationID`, `ErrorMessage` |
| `FQTypewriterSettings` | `USTRUCT` | Réglages de l'effet machine à écrire (+ audio sync) |
| `FQToastSectionData` / `FQToastMediaSettings` / `FQToastNotificationOptions` | `USTRUCT` | Sections, média et options du toast |
| `FQWarningNotificationOptions` | `USTRUCT` | Options du warning (animation, retour d'input) |
| `FQNotificationZoneConfig` | `USTRUCT` | Config d'une zone (`MaxNotifications`, direction, espacement) |
| `EQNotificationType` / `EQNotificationPriority` / `EQNotificationZone` / `EQNotificationAnimationState` | `UENUM` | Type, priorité, zone, état d'animation |

**Délégués** : dynamiques `FOnNotificationDisplayed` / `FOnNotificationDismissed` / `FOnNotificationClicked` (BP) + natifs `FOnNotificationDisplayedNative` / `FOnNotificationDismissedNative` (statiques sur le manager).

---

## 4. Flux de données et cycle de vie

1. **Appel** — un système appelle p.ex. `UQNotificationManager::ShowProgress(...)` (statique). Les paramètres sont empaquetés dans un `FQNotificationData`.
2. **Routage** — `ShowNotificationInternal` choisit la zone (`ResolveZone`), vérifie la capacité (`CountActiveNotificationsInZone`, `MaxNotifications`) et la priorité. Si la zone est pleine : interruption d'une notification de priorité inférieure (`FindLowestPriorityWidgetInZone`) ou mise en file (Communication/Warning).
3. **Création** — `SpawnNotificationWidget` / `CreateNotificationWidget` instancie la classe de widget appropriée (`TSubclassOf<UQNotificationWidgetBase>`), l'ajoute à la zone, et l'initialise (`InitializeNotification(Data)`).
4. **Affichage** — le widget joue son animation d'entrée (`StartEntranceAnimation` → `PlayEntranceAnimation` BP), démarre le son, et le cas échéant l'effet typewriter (`StartTypewriterEffect`, synchronisé à `DialogAudio` si activé).
5. **Mise à jour** — pour une progression, `UpdateProgressNotification(ID, Current, Max, Status)` met à jour le widget vivant ; l'auto-progress avance tout seul si configuré.
6. **Fermeture** — auto-hide (`HandleAutoHideTimer`), clic (`HandleNotificationClicked`), bouton dismiss, ou Échap (`HandleEscapeDismissal`). Le widget joue sa sortie (`StartExitAnimation`), diffuse `OnDismissed`, puis `RemoveActiveNotification(ID)` ; une notification en file prend alors sa place (`TryDisplayNextQueued*`).

---

## 5. Points d'intégration

- **Dépendances module** : `Core`, `CoreUObject`, `Engine`, `UMG`, `Slate`, `SlateCore`, `MediaAssets`, `InputSystem` (public) ; `InputCore` (privé). Plugins : `CommonUI` (optionnel), `InputSystem` (requis).
- **InputSystem** : les warnings peuvent afficher un **retour d'input** (`UQWarningNotificationWidget` se lie à `UInputsComponent` / `UCurrentInputData`, et utilise `UInputPreset_DA`) — d'où la dépendance dure à `InputSystem`.
- **MediaAssets** : les toasts peuvent jouer une vidéo/animation (`UMediaPlayer` + `UMediaTexture`).
- **Appelants** : n'importe quel système gameplay (quête, récolte, dialogue PNJ, alertes) via l'API statique. Le manager-widget doit avoir été créé/affiché par le HUD pour que `ManagerInstance` existe.

---

## 6. Gotchas, invariants et pièges

- **Le singleton doit exister.** Les `Show*` statiques ne font rien si aucun `UQNotificationManager` n'a été construit (`ManagerInstance == nullptr`). Le HUD/UI principal doit instancier le manager-widget au démarrage.
- **Purement client, non répliqué.** Notifier sur le serveur n'affiche rien aux joueurs ; il faut router via une RPC owning-client puis appeler l'API côté client.
- **Zones liées par `BindWidget`.** Le BP du manager doit contenir des `UVerticalBox` nommés `ToastZone` / `ProgressZone` / `CommunicationZone` (et `WarningZone` optionnel) — un nom manquant casse le `BindWidget` à la compilation du widget.
- **Échap global via préprocesseur d'input.** Le module installe un `IInputProcessor` au `PostEngineInit` ; il intercepte Échap avant le reste. Attention aux conflits avec d'autres conscommateurs d'Échap (menus).
- **Typewriter + audio sync.** Si `bTypewriterSyncToAudio` est actif mais qu'aucun `DialogAudio` n'est fourni, le débit retombe sur `CharactersPerSecond` ; vérifier la cohérence audio/texte pour les dialogues PNJ.
- **Files Communication/Warning.** Ces deux zones **mettent en file** au lieu d'empiler librement ; une notification critique peut attendre derrière une autre — gérer la priorité explicitement.
- **`FQNotificationData` est volumineux.** Beaucoup de champs (typewriter, warning, auto-progress, input feedback) ; utiliser les helpers `Show*` plutôt que de remplir la struct à la main.

---

## 7. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QNotification/Public/Core/QNotificationManager.h` / `Private/…cpp` | Façade statique, singleton, zones, priorité, files |
| `Source/QNotification/Public/Widgets/QNotificationWidgetBase.h` / `…cpp` | Base widget : typewriter, animations, auto-hide, dialogue |
| `Source/QNotification/Public/Widgets/QToastNotificationWidget.h` / `…cpp` | Toast (sections + média) |
| `Source/QNotification/Public/Widgets/QWarningNotificationWidget.h` / `…cpp` | Warning (animation cible + input feedback) |
| `Source/QNotification/Public/Widgets/QProgressNotificationWidget.h` / `…cpp` | Progress (barre + auto-progress) |
| `Source/QNotification/Public/Widgets/QCommunicationNotificationWidget.h` / `…cpp` | Dialogue PNJ (portrait + pagination) |
| `Source/QNotification/Public/Types/QNotificationTypes.h` | Enums + structs de payload/options |
| `Source/QNotification/Public/QNotificationModule.h` / `Private/…cpp` | Module + préprocesseur d'input Échap |
| `QNotification.uplugin` | Descripteur : Runtime, plugins `CommonUI` (opt.) + `InputSystem` |
