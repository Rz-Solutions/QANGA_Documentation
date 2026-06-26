# QGameManager — Architecture

Petit plugin Runtime (`Type: Runtime`, `LoadingPhase: Default`, auteur Iolacorp) qui fournit une infrastructure de **bootstrap et d'orchestration du chargement de systèmes de jeu** à l'échelle de la GameInstance et du World. Tout est piloté par des `DataAsset` et exposé au Blueprint ; il n'y a aucun acteur, aucun tick, aucune réplication. État : opérationnel, code-only côté C++, contenu et logique d'usage côté Blueprint.

---

## 1. But d'usage

QANGA charge des « systèmes » de jeu hétérogènes (économie, save, monde, UI, etc.) qui doivent démarrer dans un ordre maîtrisé, parfois conditionné au niveau courant, et certains seulement après que d'autres soient prêts. Plutôt que de câbler ces dépendances à la main dans des Blueprints d'initialisation fragiles, `QGameManager` offre :

- un **registre central de systèmes** (par `UQGameManager_World_SubSystem`) qui attend que tous les systèmes attendus se soient enregistrés, puis déclenche leur chargement en respectant les dépendances déclarées (`RequiredSystemBeforeLoading`) et un filtre par niveau ;
- des **objets de démarrage** (« Startup ») créés une fois par GameInstance et survivant aux changements de monde (par `UQGameManager_GI_SubSystem`), utiles pour des singletons logiques (services persistants) ;
- deux **magasins génériques** : un magasin d'`UObject` par nom (au niveau GameInstance) et un magasin de tableaux d'`AActor` par nom (au niveau World), pour publier/retrouver des références sans couplage direct ;
- une **table de noms de niveaux** : associe un niveau (asset `UWorld`) à un `DisplayName` lisible (réutilisable pour la sauvegarde ou l'UI).

Le tout est configuré via les **Project Settings** (`UQGameManager_Settings`, section `Project > QGameManager`) en pointant des `DataAsset`, sans code.

---

## 2. Vue d'ensemble / décisions d'architecture

Le plugin repose sur **deux subsystems UE** et une **interface** de cycle de vie :

- `UQGameManager_GI_SubSystem` (`UGameInstanceSubsystem`) — durée de vie = celle de la GameInstance. C'est le **propriétaire des données globales** : il charge (synchroniquement) les listes de `DataAsset` depuis les settings, instancie les objets de démarrage (chargement **asynchrone** de classe via `FStreamableManager`), tient les magasins persistants et la table monde→nom. Il garde une référence au `UQGameManager_World_SubSystem` actuellement enregistré (un seul à la fois).
- `UQGameManager_World_SubSystem` (`UWorldSubsystem`) — durée de vie = celle du World. C'est le **moteur d'orchestration** : il s'enregistre auprès du GI subsystem à l'init, lit le niveau courant, et pilote la machine d'état d'enregistrement/chargement des systèmes du monde. Il porte aussi le magasin d'acteurs par nom.
- `IQGameManager_Interface` (`UInterface`) — les systèmes et objets de démarrage **implémentent** cette interface pour recevoir les callbacks de cycle de vie (`QGM_Start_Loading`, `QGM_NoLoad`, `QGM_AllSystemIsReady`, `QGM_OnGameInstanceEnd`, `QGM_OnWorldEnd`). Ce sont des `BlueprintNativeEvent`, donc destinés à être implémentés en Blueprint.

**Modèle de chargement (machine d'état par système).** Chaque système traverse des listes : `System_OnWait_Load` → `System_OnStart_Load` → `System_IsLoaded` (ou `System_NoLoad`). L'enum `EQGM_SystemState` (`OnWaitLoad`, `OnStartLoad`, `OnLoaded`, `OnNoLoad`) décrit ces états ; **attention** : ces transitions sont en pratique gérées par appartenance aux `TArray` ci-dessus, l'enum n'est pas stocké ni utilisé comme champ d'état dans le code lu (il documente la sémantique plus qu'il ne la pilote).

**Déclenchement.** L'orchestration démarre quand **tous** les `System_DataAssets` connus du GI subsystem se sont enregistrés (`QGM_System_GetIsAllSystem_IsRegisted`) — alors `QGM_OnProcessLoad_IsStart` passe à `true`, le delegate `QGM_StartLoaded` est diffusé, et les systèmes dont les dépendances sont satisfaites sont lancés. Un système peut aussi se charger immédiatement à l'enregistrement si `Direct_LoadOnRegistered` est vrai.

**Filtre par niveau.** Si `OnlyLoad_OnLevel` est vrai, le système n'est éligible au chargement que si le niveau courant figure dans sa liste `Levels` (comparaison par `GetOutermost()->GetName()`). Sinon, `QGM_NoLoad` est appelé sur lui.

**Honnêteté sur l'écart intention/code :**
- L'enum `EQGM_SystemState` n'est pas utilisé pour stocker l'état réel (l'état est implicite via l'appartenance aux tableaux).
- `InitSettings()` (qui copie `Version`/`Version_Tag` depuis les settings) **n'est jamais appelée** dans le code lu ; les champs `Version`/`Version_Tag` du GI subsystem restent donc aux valeurs par défaut codées en dur (`"1.0.0"` / `"Dev"`).
- Plusieurs accès supposent la validité des pointeurs/clés (déréférencements `*Map.Find(...)` sans garde, indexation `Levels[0]`) — voir §7.
- Le nommage des fonctions mélange les préfixes `QGM_` et `GQM_` de façon non systématique (probable coquille historique conservée pour la compatibilité Blueprint).

---

## 3. Carte des composants

| Élément | Type | Rôle |
| --- | --- | --- |
| `UQGameManager_GI_SubSystem` | `UGameInstanceSubsystem` (UCLASS) | Propriétaire des données globales : charge les DataAsset depuis les settings, crée/détient les objets Startup, magasin d'`UObject` par nom, table monde→nom, référence au World subsystem courant. |
| `UQGameManager_World_SubSystem` | `UWorldSubsystem` (UCLASS) | Moteur d'orchestration du chargement des systèmes du monde ; magasin d'`AActor` par nom ; expose delegates et fonctions Blueprint. |
| `IQGameManager_Interface` / `UQGameManager_Interface` | `UInterface` (UINTERFACE) | Callbacks de cycle de vie implémentés par les systèmes/objets : `QGM_Start_Loading`, `QGM_NoLoad`, `QGM_AllSystemIsReady`, `QGM_OnGameInstanceEnd`, `QGM_OnWorldEnd` (tous `BlueprintNativeEvent`). |
| `UQGameManager_Settings` | `UDeveloperSettings` (UCLASS, `config=Game, defaultconfig`) | Source de configuration (Project Settings, catégorie `QGameManager`) : listes de `World_DataAssets`, `System_DataAssets`, `Startup_DataAssets`, plus `Version`/`Version_Tag`. |
| `UQGameManager_System_DataAsset` | `UDataAsset` (UCLASS) | Décrit un système : `Direct_LoadOnRegistered`, `OnlyLoad_OnLevel`, `Levels` (filtre), `RequiredSystemBeforeLoading` (dépendances). |
| `UQGameManager_World_DataAsset` | `UDataAsset` (UCLASS) | Associe un asset `UWorld` (`Level`) à un `DisplayName` (utilisé pour résoudre un nom lisible de niveau). |
| `UQGameManager_Startup_DataAsset` | `UDataAsset` (UCLASS) | Décrit un objet de démarrage : `ObjectClass` (`TSoftClassPtr<UObject>`) + `Name` (clé). |
| `EQGM_SystemState` | `UENUM(BlueprintType)` | États sémantiques d'un système : `OnWaitLoad`, `OnStartLoad`, `OnLoaded`, `OnNoLoad` (descriptif ; non stocké comme champ d'état). |
| `FQGM_Actor_Array` | `USTRUCT(BlueprintType)` | Wrapper `TArray<AActor*>` permettant d'avoir un `TMap<FName, TArray>` (UE n'autorise pas les tableaux imbriqués nus dans une `TMap` UPROPERTY). |
| `FQGM_StartLoaded` / `FQGM_AllLoaded` / `FQGM_SystemLoaded` | Dynamic multicast delegates | Événements Blueprint diffusés par le World subsystem : début du process, tout chargé, un système chargé (passe le `System_DataAsset`). |
| `FQGameManagerModule` | `IModuleInterface` | Module Runtime ; `StartupModule`/`ShutdownModule` sont vides (pas d'enregistrement manuel — les subsystems UE s'auto-découvrent). |

---

## 4. Flux de données et cycle de vie

**Initialisation GameInstance (`UQGameManager_GI_SubSystem::Initialize`)**
1. `Load_Datas_FromSettings()` — charge **synchroniquement** (`LoadSynchronous`) chaque `World_DataAssets`, `System_DataAssets`, `Startup_DataAssets` depuis `UQGameManager_Settings::Get()`. Pour chaque world data, remplit `WorldByName` indexé par `Level.GetAssetName()`.
2. `LoadStartup()` — pour chaque `Startup_Datas`, appelle `CreateStartup()` qui lance un **chargement asynchrone** de `ObjectClass` via `UAssetManager::GetStreamableManager().RequestAsyncLoad`. Dans le callback : `NewObject` de la classe, insertion dans `Startup_Objects` par `Name`, puis si l'objet implémente l'interface, appel de `QGM_Start_Loading`.

**Initialisation World (`UQGameManager_World_SubSystem::Initialize`)**
1. Court-circuite si commandlet/cook (`IsRunningCommandlet()` / `IsRunningCookCommandlet()`).
2. `GQM_Init_WorldSubSystem()` → `Register_To_GameInstance()` (récupère le GI subsystem et s'y enregistre comme World subsystem courant), `GetCurrentLevel()` (capte `PersistentLevel` et `World` courants dans `CurrentLevel`/`CurrentWorld`), puis réinitialise les listes d'état des systèmes.

**Runtime — enregistrement et chargement des systèmes** (appels Blueprint des systèmes eux-mêmes)
1. Un système appelle `QGM_System_Register(System, SystemData)`. Conditions : `SystemData` connu du GI subsystem **et** `System` implémente l'interface. Il est ajouté à `System_Registed` (map data→objet) et à `System_OnWait_Load`.
2. Si `OnlyLoad_OnLevel` : `FilterByLevelName` décide. Niveau correspondant + `Direct_LoadOnRegistered` → `QGM_System_StartLoad`. Niveau non correspondant → `GQM_System_SetNoLoad` (appelle `QGM_NoLoad` sur le système, l'ajoute à `System_NoLoad`).
3. `QGM_System_GetIsAllSystem_IsRegisted` vérifie après chaque enregistrement si **tous** les systèmes connus sont enregistrés ; si oui → `QGM_System_ProcessStart_LoadSystem` : `QGM_OnProcessLoad_IsStart = true`, diffuse `QGM_StartLoaded`, puis `QGM_System_GetSystemCanBeLaunch`.
4. `QGM_System_GetSystemCanBeLaunch` parcourt `System_OnWait_Load` et lance (`QGM_System_StartLoad`) ceux dont `GQM_System_GetAllRequiredSystemIsLoad` est vrai (toutes les dépendances de `RequiredSystemBeforeLoading` sont dans `System_IsLoaded` ou `System_NoLoad`).
5. `QGM_System_StartLoad` déplace le système de `System_OnWait_Load` vers `System_OnStart_Load` et appelle `QGM_Start_Loading` sur lui.
6. Quand un système a fini, **il** appelle `QGM_System_IsLoaded(SystemData)` : déplacement vers `System_IsLoaded`, diffusion de `QGM_SystemLoaded(SystemData)`, relance de `QGM_System_GetSystemCanBeLaunch` (si le process est démarré) pour débloquer les dépendants, puis `QGM_System_GetAllSystemIsLoaded`.
7. `QGM_System_GetAllSystemIsLoaded` : si `System_OnWait_Load` **et** `System_OnStart_Load` sont vides → diffuse `QGM_AllLoaded` et appelle `QGM_AllSystemIsReady` sur tous les systèmes enregistrés.

**Teardown World (`Deinitialize` → `QGM_End_WorldSubSystem`)** : vide `ActorArrayStorage`, puis `UnRegister_To_GameInstance` → le GI subsystem met `World_SubSystem = nullptr` et appelle `CallEndWorldToStartup` (→ `QGM_OnWorldEnd` sur chaque objet Startup).

**Teardown GameInstance (`Deinitialize`)** : `CallEndGameInstance` (→ `QGM_OnGameInstanceEnd` sur chaque objet Startup), puis `ConditionalBeginDestroy` sur les objets Startup, vidage des maps/arrays.

**Magasins et helpers d'accès (Blueprint)**
- Startup : `QGM_BP_GetStartup_Object(Name, out Object)`.
- Magasin d'UObject (GI) : `QGM_StorageObject_Add/Remove/Get(Name, ...)`.
- Magasin d'acteurs (World) : `QGM_Actor_RegistreTo/UnRegistreTo/GetRegistre(Registre, Actor)`.
- Nom de monde lisible : `QGM_Get_WorldName()` (World) → `GetLevelName()` (GI) via `WorldByName`.
- Accès systèmes : `QGM_BP_GetSystem`, `GQM_BP_GetSystemIsLoaded`.

---

## 5. Réplication / réseau

**Aucune.** Le plugin ne réplique rien : pas de `Replicated`, pas de `DOREPLIFETIME`, pas de `GetLifetimeReplicatedProps`, pas de RPC `Server_/Client_/Multicast_`. Les subsystems s'exécutent localement sur chaque instance (client, serveur, standalone) selon leur portée UE habituelle. Toute coordination réseau doit être assurée par les systèmes consommateurs eux-mêmes ; `QGameManager` est purement local.

---

## 6. Points d'intégration

**Dépendances du module** (`QGameManager.Build.cs`) :
- Public : `Core`, `DeveloperSettings`.
- Private : `CoreUObject`, `Engine`, `Slate`, `SlateCore`.
- `.uplugin` : `CanContainContent: true` (le plugin embarque/peut embarquer du contenu) ; un seul module `QGameManager` (Runtime, Default).

**Configuration** (`UQGameManager_Settings`, `config=Game, defaultconfig`, section `[/Script/QGameManager.QGameManager_Settings]`). Configuré dans `G:/QANGA/Config/DefaultGame.ini`, qui peuple `World_DataAssets` :
```
[/Script/QGameManager.QGameManager_Settings]
+World_DataAssets=/Game/Systems/_QGM/Level/QGM_World_Lobby.QGM_World_Lobby
+World_DataAssets=/Game/Systems/_QGM/Level/QGM_World_Tutorial.QGM_World_Tutorial
+World_DataAssets=/Game/Systems/_QGM/Level/QGM_World_Univers.QGM_World_Univers
```
À noter : dans le `DefaultGame.ini` actuel, seuls `World_DataAssets` sont renseignés ; `System_DataAssets` et `Startup_DataAssets` ne le sont pas (les pipelines système/startup sont donc, pour cette config, inactifs jusqu'à ce que des DataAsset y soient ajoutés ou que des systèmes appellent l'API directement).

**Consommateurs dans QANGA :** l'intégralité de l'API runtime est `BlueprintCallable`/`BlueprintAssignable` et **les consommateurs sont côté Blueprint** (assets sous `/Game/Systems/_QGM/...`). Aucun code C++ du jeu (`G:/QANGA/Source`) ni d'un autre plugin ne référence les types publics de `QGameManager` — la seule référence hors-plugin trouvée est l'entrée de configuration dans `DefaultGame.ini`. Pour inspecter les implémentations concrètes (objets Startup, systèmes implémentant `IQGameManager_Interface`), passer par les assets Blueprint, pas par le code C++.

**CVars / commandes console :** aucune définie par le plugin.

---

## 7. Gotchas, invariants et pièges

- **Déréférencements non gardés de `Map.Find`.** `QGM_BP_GetStartup_Object`, `QGM_StorageObject_Get`, `QGM_BP_GetSystem` font `Object = *Map.Find(Name)` **avant** de tester la nullité de la valeur. Si la clé est absente, `Find` renvoie `nullptr` et le déréférencement plante. Côté Blueprint, n'appeler ces getters qu'avec une clé connue comme présente.
- **`Levels[0]` indexé sans contrôle.** Dans `QGM_System_Register`, quand `OnlyLoad_OnLevel` est vrai, le code log `SystemData->Levels[0]->GetPathName()`. Un `System_DataAsset` avec `OnlyLoad_OnLevel=true` mais `Levels` vide provoque un accès hors-bornes. Invariant : si `OnlyLoad_OnLevel`, `Levels` doit contenir au moins une entrée valide.
- **Le déclenchement attend l'enregistrement de TOUS les systèmes.** `QGM_System_ProcessStart_LoadSystem` n'est lancé que lorsque chaque `System_DataAssets` (connu du GI subsystem) s'est enregistré. Un seul système qui ne s'enregistre jamais bloque tout le pipeline (`QGM_StartLoaded`/`QGM_AllLoaded` ne sont jamais diffusés). Garder `System_DataAssets` (settings) synchronisé avec les systèmes réellement présents au runtime.
- **Le signalement de fin de chargement est à la charge du système.** Un système doit explicitement appeler `QGM_System_IsLoaded` ; tant qu'il ne le fait pas, il reste dans `System_OnStart_Load` et bloque `QGM_AllLoaded`. Il n'y a ni timeout ni watchdog côté plugin.
- **`NoLoad` compte comme « satisfait » pour les dépendances.** `GQM_System_GetAllRequiredSystemIsLoad` considère un prérequis présent dans `System_NoLoad` comme satisfait. Un système requis mais filtré hors du niveau courant ne bloquera donc pas ses dépendants (comportement voulu : « pas censé charger ici »).
- **Un seul World subsystem enregistré à la fois.** `GI_SubSystem->World_SubSystem` est un pointeur unique ; `Register_WorldSubSystem` écrase sans condition. En cas de mondes multiples/transitions, le dernier enregistré gagne. `UnRegister_WorldSubSystem` remet à `nullptr` et déclenche `QGM_OnWorldEnd` sur les objets Startup.
- **Chargement synchrone des DataAsset au démarrage GameInstance.** `Load_Datas_FromSettings` utilise `LoadSynchronous` sur toutes les listes — coût de chargement bloquant proportionnel au nombre/poids des DataAsset référencés dans les settings.
- **Objets Startup créés asynchronement.** `CreateStartup` charge la classe en async ; l'objet n'existe donc pas immédiatement après `Initialize`. Ne pas supposer qu'un objet Startup est disponible juste après le démarrage de la GameInstance (vérifier via `QGM_BP_GetStartup_Object`).
- **`Startup_Objects` indexé par `Name` (clé d'unicité).** `CreateStartup` ignore un DataAsset dont le `Name` est déjà présent. Le défaut `Name = "non"` dans `UQGameManager_Startup_DataAsset` collisionnerait si plusieurs DataAsset gardent la valeur par défaut → un seul serait créé. Donner un `Name` unique à chaque Startup DataAsset.
- **`EQGM_SystemState` est descriptif, pas autoritatif.** L'état réel d'un système se lit par appartenance aux tableaux `System_OnWait_Load` / `System_OnStart_Load` / `System_IsLoaded` / `System_NoLoad`, pas via un champ enum.
- **`InitSettings()` mort.** La fonction existe mais n'est jamais appelée ; `Version`/`Version_Tag` du GI subsystem ne reflètent pas les settings.
- **Préfixes mixtes `QGM_`/`GQM_`.** Incohérence de nommage historique (`GQM_System_SetNoLoad`, `GQM_BP_GetSystemIsLoaded`, `GQM_Init_WorldSubSystem`, etc.). Conserver les noms exacts pour ne pas casser les Blueprints qui les appellent.
- **Logs sur `LogTemp`.** Toute la journalisation passe par `LogTemp` (Warning/Error/Log), pas une catégorie dédiée — bruyant et difficile à filtrer.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
| --- | --- |
| `G:/QANGA/Plugins/QGameManager/QGameManager.uplugin` | Descripteur du plugin (module Runtime unique, `CanContainContent`). |
| `G:/QANGA/Plugins/QGameManager/Source/QGameManager/QGameManager.Build.cs` | Règles de build et dépendances (`Core`, `DeveloperSettings` ; `CoreUObject`, `Engine`, `Slate`, `SlateCore`). |
| `Source/QGameManager/Public/QGameManager.h` / `Private/QGameManager.cpp` | Module `FQGameManagerModule` (startup/shutdown vides). |
| `Source/QGameManager/Public/QGameManager_GI_SubSystem.h` / `Private/QGameManager_GI_SubSystem.cpp` | GameInstance subsystem : chargement des DataAsset, objets Startup, magasin d'UObject, table monde→nom, référence au World subsystem. |
| `Source/QGameManager/Public/QGameManager_World_SubSystem.h` / `Private/QGameManager_World_SubSystem.cpp` | World subsystem : machine d'orchestration du chargement des systèmes, delegates Blueprint, magasin d'acteurs ; définit `EQGM_SystemState` et `FQGM_Actor_Array`. |
| `Source/QGameManager/Public/QGameManager_Interface.h` / `Private/QGameManager_Interface.cpp` | Interface de cycle de vie `IQGameManager_Interface` (callbacks `BlueprintNativeEvent`). |
| `Source/QGameManager/Public/QGameManager_Settings.h` / `Private/QGameManager_Settings.cpp` | `UDeveloperSettings` : listes de DataAsset configurées en Project Settings + `Get()`. |
| `Source/QGameManager/Public/QGameManager_System_DataAsset.h` / `.cpp` | DataAsset décrivant un système (flags de chargement, filtre niveau, dépendances). |
| `Source/QGameManager/Public/QGameManager_World_DataAsset.h` / `.cpp` | DataAsset associant un asset `UWorld` à un `DisplayName`. |
| `Source/QGameManager/Public/QGameManager_Startup_DataAsset.h` / `.cpp` | DataAsset décrivant un objet de démarrage (`ObjectClass` + `Name`). |
| `G:/QANGA/Config/DefaultGame.ini` (section `[/Script/QGameManager.QGameManager_Settings]`) | Configuration projet : référence les `World_DataAssets` sous `/Game/Systems/_QGM/Level/`. |
