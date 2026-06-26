# QSystem — Architecture

> Plugin runtime (auteur : Iolacorp) : la **couche fondation** de QANGA. Il fournit (1) les **classes de base du framework** (`QGameInstance`, `QGameMode`, `QGameState`, `QPlayerController`, `QPlayerState`) à sous-classer, et (2) un **framework de « Systèmes »** : des composants modulaires (`UQSystem_SystemComponent`) qui s'auto-valident contre le **type/les attributs du monde courant**, s'enregistrent dans un **registre à portée monde** (`UQSystem_WorldSubsystem`) et sont **découvrables par nom** (localement ou par joueur).

---

## 1. But d'usage

QANGA tourne sur plusieurs types de mondes (jeu, lobby, mondes « système ») et doit activer/désactiver des fonctionnalités selon le contexte, tout en gardant un moyen propre de **retrouver un sous-système** sans le câbler en dur. `QSystem` répond à deux besoins :

1. **Typer le monde** : chaque niveau/monde est étiqueté (type + attributs) en data (`UQSystem_Settings`), pour que le code puisse demander « dans quel type de monde suis-je ? » et conditionner son activation.
2. **Déclarer des « Systèmes » modulaires** : une fonctionnalité est un `UQSystem_SystemComponent` posé sur un acteur ; il ne s'initialise que si le monde est **valide** pour lui, s'enregistre sous un `SystemNameID`, et devient récupérable par nom — un **service-locator** simple, à portée monde et par joueur.

S'y ajoutent les classes de framework Q* (GameMode, GameState, PlayerController…) que le projet sous-classe en Blueprint.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Classes de framework = bases vides.** `UQGameInstance`, `AQGameMode`, `AQGameState`, `AQPlayerController`, `AQPlayerState` sont des sous-classes Blueprintable quasi vides : des **points d'ancrage** que le projet étend (en C++ ailleurs ou en BP). Elles existent pour donner un type Q* stable au framework.
- **Système = composant auto-validant.** `UQSystem_SystemComponent` porte un `SystemNameID` et des règles de validité monde (`UseWorldValidation`, `ValidWorldType`, `CustomWorldTypeValidation`, `LaunchByWorldAttribute`, `SystemCanLaunchByWorldAttribute`). À `BeginPlay`, il vérifie le monde courant via le world subsystem ; s'il est valide, il s'initialise (`OnSystemBeginPlay`, BP native event) et diffuse `System_OnSystemInitialized`.
- **Registre à portée monde.** `UQSystem_WorldSubsystem` (WorldSubsystem) tient deux registres : `QSystem_Local_Components` (par `SystemNameID`) et `QSystem_Player_Controller_Components` (par PlayerController → map de systèmes). Découverte : `QSystem_GetLocalSystemComponent(Name)` / `QSystem_GetPlayerSystemComponent(PC, Name)`.
- **Identité du monde en data.** `UQSystem_Settings` (`UDeveloperSettings`) mappe chaque `UWorld` vers un `FQSystem_WorldAttribute` (type `EQSystem_WorldType` ∈ {`Game`, `Lobby`, `System`, `Custom`}, type custom, nom, attributs `FName`). Le world subsystem résout ces données au démarrage (`GetWorldData`) et les expose (`QSystem_GetWorldType`, `GetWorldAttribute`, `QSystem_GetWorldName`…).
- **Validation découplée.** Un système ne « sait » pas dans quel monde il tourne : il déclare ses mondes/attributs valides, et le subsystem tranche. Activer une fonctionnalité par monde devient déclaratif.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQSystem_WorldSubsystem` | `UWorldSubsystem` | Registre des systèmes (local + par joueur), identité/typage du monde, découverte par nom |
| `UQSystem_SystemComponent` | `UActorComponent` (`BlueprintSpawnableComponent`) | Un « Système » modulaire : validation monde, init, enregistrement |
| `UQSystem_Settings` | `UDeveloperSettings` (`config=Game`) | Map `UWorld → FQSystem_WorldAttribute` (typage des mondes) |
| `UQSystem_GISubsystem` | `UGameInstanceSubsystem` | Placeholder (vide) |
| `UQSystem_FunctionLibrary` | `UBlueprintFunctionLibrary` | Placeholder (vide) |
| `UQGameInstance` | `UGameInstance` | Base de framework |
| `AQGameMode` | `AGameModeBase` | Base de framework |
| `AQGameState` | `AGameStateBase` | Base de framework |
| `AQPlayerController` | `APlayerController` | Base de framework |
| `AQPlayerState` | `APlayerState` | Base de framework |
| `AQGameMode_Base` | `APlayerState` | Base déclarée (parent inhabituel — voir §6) |
| `FQSystem_WorldAttribute` | `USTRUCT` | Type + custom type + nom + attributs d'un monde |
| `FQSystem_PlayerSystemComponent` | `USTRUCT` | Map `SystemNameID → composant` pour un joueur |
| `EQSystem_WorldType` | `UENUM` | `Game` / `Lobby` / `System` / `Custom` |

**Délégués** : `UQSystem_SystemComponent::System_OnSystemInitialized` / `System_OnSystemEnd` ; `UQSystem_WorldSubsystem::System_OnSystemInitialized(FName)`.

---

## 4. Flux de données et cycle de vie

1. **Init monde** — `UQSystem_WorldSubsystem::OnWorldBeginPlay` appelle `GetWorldData` : résout le `UWorld` courant dans `UQSystem_Settings.WorldAttribute` → renseigne `WorldName`/`WorldType`/`WorldAttribute`/`WorldCustomWorldType` (`WorldDataFound`).
2. **Spawn d'un système** — un `UQSystem_SystemComponent` posé sur un acteur démarre (`BeginPlay`) et appelle `CheckWorldValidation(subsystem)` : compare `ValidWorldType`/`CustomWorldTypeValidation` et `SystemCanLaunchByWorldAttribute` au monde courant.
3. **Initialisation conditionnelle** — si valide, le composant s'initialise (`QSystem_SetIsInitialized`, `OnSystemBeginPlay`), s'enregistre auprès du subsystem (`QSystem_RegisterLocal_SystemComponent` et/ou `QSystem_Register_PlayerComponent`), et diffuse `System_OnSystemInitialized`. Sinon il reste inactif.
4. **Découverte** — d'autres systèmes/BP récupèrent un système par nom : `QSystem_GetLocalSystemComponent("X")` ou `QSystem_GetPlayerSystemComponent(PC, "X")`.
5. **Fin** — `EndPlay` désenregistre le composant (`QSystem_UnregisterLocal_SystemComponent` / `QSystem_UnRegister_PlayerComponent`) et diffuse `System_OnSystemEnd`.

---

## 5. Points d'intégration

- **Dépendances module** : `Core`, `DeveloperSettings` (public) ; `CoreUObject`, `Engine`, `Slate`, `SlateCore` (privé). Aucune dépendance vers d'autres plugins QANGA — c'est une **brique fondation**.
- **Consommateurs** : tout système gameplay qui veut être (a) **conditionné par le type de monde** et (b) **découvrable par nom** sans référence dure. Les classes de framework Q* sont les bases du `GameMode`/`GameState`/etc. du projet.
- **Configuration** : *Project Settings → QSystem* (`UQSystem_Settings`) — typage de chaque monde.

---

## 6. Gotchas, invariants et pièges

- **`AQGameMode_Base` hérite d'`APlayerState`.** Dans le code actuel, `AQGameMode_Base` est déclaré `: public APlayerState` (et non un GameMode) — incohérence apparente avec son nom. Vérifier l'usage réel avant de s'en servir comme « base de GameMode ».
- **Un système inactif est silencieux.** Si la validation monde échoue, le composant **ne s'initialise pas** et ne s'enregistre pas ; `QSystem_GetLocalSystemComponent` renverra `null`. Vérifier `ValidWorldType`/attributs et le typage du monde dans `UQSystem_Settings`.
- **Le typage du monde dépend des settings.** Un monde non listé dans `WorldAttribute` retombe sur des valeurs par défaut (`WorldDataFound == false`) ; les systèmes validant par type/attribut risquent de ne jamais s'activer. Étiqueter chaque monde.
- **Registre à portée monde.** Les enregistrements vivent dans le `UQSystem_WorldSubsystem` du monde courant ; ils ne survivent pas à un changement de niveau (recréation du subsystem).
- **`SystemNameID` doit être unique par portée.** Deux systèmes locaux de même nom se masquent dans `QSystem_Local_Components` (map par `FName`).
- **`UQSystem_GISubsystem` / `UQSystem_FunctionLibrary` sont vides** — points d'extension non remplis.

---

## 7. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QSystem/Public/QSystem_WorldSubsystem.h` / `Private/…cpp` | Registre des systèmes + typage du monde |
| `Source/QSystem/Public/Component/QSystem_SystemComponent.h` / `…cpp` | Composant « Système » auto-validant |
| `Source/QSystem/Public/QSystem_Settings.h` / `…cpp` | `UDeveloperSettings` : map monde → attributs |
| `Source/QSystem/Public/Struct/QSystem_Struct.h` | `EQSystem_WorldType`, `FQSystem_WorldAttribute` |
| `Source/QSystem/Public/GameClass/Q*.h` | Bases de framework (`QGameInstance`, `QGameMode`, `QGameState`, `QPlayerController`, `QPlayerState`) |
| `QSystem.uplugin` | Descripteur : module Runtime `QSystem` |
