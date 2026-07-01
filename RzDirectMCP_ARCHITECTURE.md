# RzDirectMCP — Architecture

Bridge MCP headless qui expose l'éditeur Unreal de QANGA à des clients IA externes (Claude Code, Codex, Cursor). Plugin éditeur autonome, sans dépendance C++ du reste du projet : le code de QANGA ne le référence jamais — il est piloté entièrement de l'extérieur via un socket TCP loopback.

> Statut : opérationnel. Un seul module éditeur, un seul `UEditorSubsystem`, un thread serveur TCP, 40 bibliothèques de fonctions réflexives totalisant 872 outils natifs, et un adaptateur stdio Node.js qui présente 700 outils côté client. La doc embarquée historique (`Plugins/RzDirectMCP/Docs/ARCHITECTURE.md`) décrit le système à grands traits mais comporte plusieurs imprécisions corrigées ici (voir §7).

---

## 1. But d'usage

QANGA est un jeu UE5.7 massif (plugins C++, planètes entières, contenu Blueprint dense). Une grande partie du travail d'itération — inspecter un Blueprint, déplacer un acteur, recâbler un graphe matériau, lire les erreurs de compilation, capturer le viewport — passe par l'éditeur. RzDirectMCP rend ces opérations pilotables par un agent IA **sans embarquer d'IA dans l'éditeur** : pas de workspace IA intégré, pas de runtime de provider, pas d'éditeur de remplacement. Le plugin se contente d'**exposer** la surface d'édition d'Unreal sous forme d'outils MCP que n'importe quel client compatible (stdio JSON-RPC 2.0) peut appeler.

Concrètement, il résout : « comment laisser un assistant IA externe lire et muter le projet ouvert dans l'éditeur, en toute sécurité vis-à-vis du thread de jeu et du GC, sans modifier le projet hôte ». Le plugin se dépose dans n'importe quelle install UE 5.7+ et les outils dont le plugin de domaine sous-jacent est absent (Niagara, PCG, GAS, ...) deviennent des stubs inertes au lieu de casser la compilation (voir §6).

---

## 2. Vue d'ensemble / décisions d'architecture

Le système comporte trois couches, du client vers l'éditeur :

```
  Client IA (Claude Code, Codex, Cursor, ...)
        |  stdio JSON-RPC 2.0 (protocole MCP)
        v
  rzmcp-bridge.mjs   (serveur stdio Node.js, sans dépendance npm)
        |            surface de 700 outils "client", pilotée par manifest.json
        |  une requête JSON UTF-8 par connexion TCP   127.0.0.1:8767  (loopback)
        v
  Processus Unreal Editor
    URzMCPBridgeSubsystem  (UEditorSubsystem)
        |  ouvre le socket d'écoute et lance le thread
        v
    FRzMCPBridgeWorker  (FRunnable, thread "RzMCPNativeBridgeThread")
        |  accept() -> lit 1 objet JSON -> ExecuteOnGameThread(...)
        v
    HandleRequestOnGameThread  (sur le game thread)
        |  type=ping | get_tool_count | list_tools | call_tool
        v
    FRzMCPNativeToolRegistry  (singleton C++, non-UObject)
        |  table nom d'outil -> (UClass*, UFunction*)
        v
    40 UBlueprintFunctionLibrary  taguées meta=(RzMCPToolLibrary)
        872 UFUNCTION  appelées par réflexion via ProcessEvent
```

Décisions clés et invariants réels du code :

- **Dispatch game-thread par requête, pas de file persistante.** Le thread serveur ne maintient PAS de file drainée dans un `Tick`. Pour chaque requête il appelle `ExecuteOnGameThread(...)`, qui poste un `FTSTicker::GetCoreTicker()` one-shot et **bloque le thread serveur** sur un `FEvent` jusqu'à complétion (timeout 120 s). Tous les outils s'exécutent donc sur le game thread, dans une frame éditeur normale → sûreté UObject/GC/undo-redo par construction.
- **Une requête, une connexion.** Le worker `accept()` une connexion, lit exactement **un** objet JSON complet (comptage d'accolades hors-chaîne dans `ReceiveJsonPayload`, plafond 10 Mo), répond, puis ferme et détruit le socket. Pas de keep-alive, pas de multiplexage : chaque appel d'outil est une connexion TCP éphémère.
- **Découverte d'outils par réflexion + métadonnée.** Le registre itère toutes les `UClass` dérivant de `UBlueprintFunctionLibrary` taguées `meta=(RzMCPToolLibrary)`, et enregistre chaque `UFUNCTION` `static` + `BlueprintCallable`. Aucun code d'enregistrement manuel : ajouter une bibliothèque taguée suffit (rebuild requis).
- **Nom d'outil = `<library>.<function>` en snake_case.** Dérivé du nom de classe (drop `U`, drop suffixe `Library`, snake_case) et du nom de fonction (snake_case). Pas de surcharge `RzMCPName` : cette méta n'existe pas dans le code (contrairement à ce qu'affirme la doc embarquée).
- **Alias de nom simple.** Si un nom de fonction est unique parmi toutes les bibliothèques, il est aussi exposé sans préfixe (alias).
- **Deux surfaces d'outils distinctes.** La couche native (~777 UFUNCTION réflexives) est la vérité côté éditeur. La couche client (~604 outils) est un re-packaging défini par `manifest.json` côté Node : chaque outil client porte une entrée `dispatch` qui le mappe sur un (ou des) outil(s) natif(s). Les deux comptes ne coïncident pas et ne doivent pas l'être.
- **Gating par présence de plugin sur disque.** `Build.cs` scanne les `.uplugin` présents et définit `RZMCP_WITH_<X>=0|1`. Les libs dont le plugin optionnel est absent sont **exclues de l'enregistrement** (`SkippedLibraries`), donc absentes de `list_tools`.

Ce qui n'existe PAS (à ne pas inventer) : aucune réplication réseau (c'est un outil éditeur loopback, voir §5) ; aucun chat UI ; aucun provider IA ; aucune méta `RzMCPName` ; aucune file d'outils drainée dans un `Tick`.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `FRzDirectMCPModule` | `IModuleInterface` | Point d'entrée du module. `StartupModule`/`ShutdownModule` vides — tout le cycle de vie passe par le subsystem éditeur. |
| `URzMCPBridgeSubsystem` | `UEditorSubsystem` (`UCLASS`) | Démarre/arrête le serveur TCP. `Initialize` appelle `EnsureInitialized()` du registre puis `StartServer()`. Résout le port (config / env). Détient `ListenerSocket`, `ServerThread`, `ServerWorker`. |
| `FRzMCPBridgeWorker` | `FRunnable` (privé, dans le .cpp) | Boucle d'acceptation TCP. Lit une requête JSON, l'exécute via `ExecuteOnGameThread`, renvoie la réponse, ferme la connexion. |
| `FRzMCPNativeToolRegistry` | classe C++ singleton (non-UObject) | Découverte par réflexion des bibliothèques `RzMCPToolLibrary`, table `ToolsByName` (nom → `FToolHandle`) et `DefinitionsByPrimaryName`. Marshalling JSON↔FProperty et appel via `ProcessEvent`. Protégé par `FRWLock`. |
| `FRzMCPNativeToolDefinition` | `struct` | Métadonnée d'un outil : `Name`, `Library`, `Function`, `Description`, `InputSchemaJson` (JSON Schema), `Aliases`. |
| `FRzMCPNativeToolRegistry::FToolHandle` | `struct` (privé) | Handle d'appel : `TWeakObjectPtr<UClass>`, `TWeakObjectPtr<UFunction>`, `PrimaryName`. |
| `FBlueprintGraphVariableSpec` | `USTRUCT(BlueprintType)` | Descripteur de type de pin (catégorie, sous-catégorie, conteneur array/set/map) passé depuis MCP/Blueprint vers les outils de graphe. |
| `RzMCP::BitmapAnnotation` (`FBitmapCanvas`, ...) | namespace / structs CPU | Moteur d'annotation bitmap RGBA pur-CPU (lignes, points, police 5×7, grille monde projetée, callouts par acteur) pour annoter les captures viewport en mode headless. Implémentation locale RzDirectMCP. |
| `FNodeForgeOrchestrator` & al. (`Compiler/`) | classes C++ internes | « NodeForge » : génération/assemblage de graphes Blueprint à partir de specs de haut niveau (entrée/sortie de fonction, chaînes de nœuds, résolution de types, layout). Catégorie de log `LogNodeForge`. Support interne des outils `blueprint_graph.*`. |
| Les **40** `UBlueprintFunctionLibrary` taguées `RzMCPToolLibrary` | `UCLASS(meta=(RzMCPToolLibrary))` | Surface réelle des outils. UFUNCTION `static`+`BlueprintCallable` → outils MCP. |

Les 40 bibliothèques d'outils (toutes `public : UBlueprintFunctionLibrary`) :

`UAbilitySystemRuntimeLibrary`, `UActorEditorLibrary`, `UAnimationEditorLibrary`, `UAnimBlueprintGraphLibrary`, `UAssetManagementLibrary`, `UAssetWorkflowLibrary`, `UAudioWorkflowLibrary`, `UBackupLibrary`, `UBlueprintGraphLibrary`, `UBlueprintToCppLibrary`, `UComponentEditorLibrary`, `UConfigSettingsLibrary`, `UDataflowAgentLibrary`, `UDataTableEditorLibrary`, `UEditorWorkflowLibrary`, `UExtendedEditorLibrary`, `UFoliageEditorLibrary`, `UFoliageWorkflowLibrary`, `UGameFeaturesEditorLibrary`, `UGameplayEditorLibrary`, `ULandscapeWorkflowLibrary`, `ULocalizationEditorLibrary`, `ULogsLibrary`, `UMaterialEditingHelperLibrary`, `UMCPEditorUtilityLibrary`, `UMeshEditorLibrary`, `UMeshWorkflowLibrary`, `UNiagaraEditorLibrary`, `UPCGEditorLibrary`, `UPhysicsEditorLibrary`, `UPluginToolsetLibrary`, `URenderingWorkflowLibrary`, `USequencerEditorLibrary`, `USequencerWorkflowLibrary`, `USlateInspectorLibrary`, `USmartObjectWorkflowLibrary`, `UStateTreeEditorLibrary`, `UViewportCaptureLibrary`, `UWorldConditionLibrary`, `UWorldPartitionEditorLibrary`.

> `UMCPEditorHelpers` (header-only, `MCPEditorHelpers.h`) est une bibliothèque utilitaire de (dé)sérialisation JSON ; elle **n'est pas** taguée `RzMCPToolLibrary` et n'expose donc pas d'outils. Elle centralise les helpers `SerializeJson`/`SerializeVector`/etc. partagés par les autres libs.

---

## 4. Flux de données et cycle de vie

**Initialisation (au démarrage de l'éditeur)**
1. UE charge le module `RzDirectMCP` (`Type=Editor`, `LoadingPhase=Default`, `Win64` uniquement).
2. `URzMCPBridgeSubsystem::Initialize` s'exécute. Si `IsRunningCommandlet()`, il sort sans rien faire (pas de bridge en cook/commandlet).
3. `FRzMCPNativeToolRegistry::Get().EnsureInitialized()` : sous `FWriteScopeLock`, `RegisterAll()` itère les `UClass`, ignore les classes `CLASS_Abstract`/`CLASS_NewerVersionExists` (REINST) et les bibliothèques `SkippedLibraries` (plugin absent), puis `RegisterLibrary()` pour chaque lib taguée. Les alias de nom simple sont calculés ensuite.
4. `StartServer()` : `ResolvePort()` lit `[/Script/RzDirectMCP.RzMCPBridgeSubsystem] NativeBridgePort` dans `GEditorPerProjectIni`, puis la variable d'environnement `RZMCP_NATIVE_PORT` (qui a priorité). Validation 1024–65535. `FTcpSocketBuilder` bind `127.0.0.1:<port>` (`AsReusable`, `Listening(16)`). Un `FRzMCPBridgeWorker` est lancé sur le thread `RzMCPNativeBridgeThread`.

**Runtime (par requête)**
1. Le client ouvre une connexion TCP, envoie un objet JSON, le worker `accept()`.
2. `ReceiveJsonPayload` lit par chunks de 16 Ko, détecte la fin de l'objet par comptage d'accolades hors-chaîne, désérialise. Échec/oversize → erreur JSON renvoyée directement.
3. `ExecuteOnGameThread([...]{ Response = HandleRequestOnGameThread(Payload); }, 120.0)` : si déjà sur le game thread, exécution directe ; sinon ticker one-shot + attente bloquante sur `FEvent`. Timeout → `"Game thread tool execution timed out"`.
4. `HandleRequestOnGameThread` route selon `type` :
   - `ping` → `{"success":true,"message":"pong"}`
   - `get_tool_count` → `{"success":true,"count":<N>}`
   - `list_tools` / `tools/list` / `get_tool_definitions` → catalogue trié (nom, library, function, description, `inputSchema`, `aliases`)
   - `call_tool` / `tools/call` → lit `tool`|`name` et `arguments`|`args`, délègue à `Registry.CallToolJson`
   - sinon, si `type` est lui-même un nom d'outil connu → appel direct avec la requête comme arguments
5. `CallToolJson` : résout `FToolHandle`, alloue les params via `FMemory_Alloca`+`InitializeStruct`, marshalle chaque `CPF_Parm` non-`Return`/`Out` depuis le JSON (`SetPropertyFromJson`), applique les défauts `CPP_Default_*` si absents, appelle `ProcessEvent` sur le **CDO** de la classe, puis reconstruit le JSON de retour (`BuildReturnJson`) et `DestroyStruct`.
6. La réponse UTF-8 est renvoyée en une passe, le socket client est fermé/détruit.

**Marshalling JSON → FProperty → JSON** (dans `RzMCPNativeToolRegistry.cpp`)
- Résolution d'arguments tolérante : nom C++ exact, sinon snake_case, sinon insensible à la casse.
- Types gérés : bool, numériques (entiers/flottants/enum), `FStr`/`FName`/`FText`, struct (cas spéciaux `FVector`/`FRotator`/`FLinearColor` champ-à-champ, sinon `ImportText_Direct`), tableaux (récursif via `FScriptArrayHelper`), `FClassProperty` (résolution `/Game/...` + suffixe `_C`), `FObjectProperty` (chargement d'asset par path).
- Retour : si la fonction renvoie une `FString` qui est elle-même du JSON, il est ré-injecté tel quel (avec champ `tool` ajouté) ; sinon enveloppé dans `{success, tool, result}`.
- Normalisation spéciale : `material_editing_helper.apply_material_operations` accepte `operations`/objet d'opération brut et les reconvertit en `operations_json` (`NormalizeMaterialOperationsArgs`).

**Teardown**
- `URzMCPBridgeSubsystem::Deinitialize` → `StopServer()` : `Worker->Stop()` (lève `bStopping`), ferme le `ListenerSocket`, `WaitForCompletion()` du thread, delete worker, détruit le socket via `ISocketSubsystem`.

**Couche client (Node, hors processus éditeur)**
- `rzmcp-bridge.mjs` (sans dépendance npm) lit `manifest.json` + `specials.mjs` au démarrage, parle MCP stdio JSON-RPC 2.0 au client IA. Sur `tools/call`, il consulte l'entrée `dispatch` de l'outil pour construire le payload natif, ouvre `net.createConnection({host, port: RZMCP_NATIVE_PORT ?? 8767})`, envoie `{type:"call_tool", tool, arguments}`, et retraduit la réponse en `CallToolResult`.

---

## 5. Réplication / réseau

**Le plugin ne réplique rien au sens Unreal.** Aucun `UPROPERTY(Replicated)`, aucun `DOREPLIFETIME`, aucune RPC `Server_`/`Client_`/`Multicast_`, aucun `GetLifetimeReplicatedProps`. Il n'y a ni `AActor` ni `UActorComponent` répliqué dans ce plugin.

Le seul aspect « réseau » est un **socket TCP loopback** :
- Liaison stricte à `127.0.0.1` (`FIPv4Address(127,0,0,1)`) — jamais exposé hors machine.
- Port par défaut `8767`, surchargé par `[/Script/RzDirectMCP.RzMCPBridgeSubsystem] NativeBridgePort` (`GEditorPerProjectIni`) puis par l'env `RZMCP_NATIVE_PORT`.
- Protocole applicatif minimal : un objet JSON UTF-8 par connexion, champ `type` discriminant (`ping`, `get_tool_count`, `list_tools`/`tools/list`/`get_tool_definitions`, `call_tool`/`tools/call`, ou un nom d'outil direct). Réponses en `{"success":bool, ...}`.
- Modèle d'autorité : il n'y en a pas au sens jeu. L'éditeur est le seul exécutant ; le client TCP est un appelant de confiance (loopback) ; toute exécution d'outil est forcée sur le game thread.

C'est un canal de contrôle d'outillage, pas du gameplay réseau. Aucune section autorité serveur/client de jeu ne s'applique.

---

## 6. Points d'intégration

**Dépendances entrantes (ce que le plugin consomme)**
- Modules toujours requis (extrait de `Build.cs`) : `UnrealEd`, `Kismet`, `KismetCompiler`, `BlueprintGraph`, `Blutility`, `Json`, `JsonUtilities`, `ImageWrapper`, `EditorSubsystem`, `Sockets`, `Networking`, `HTTP`, `UMG`/`UMGEditor`, `Slate`/`SlateCore`, `MaterialEditor`, `AnimGraph`, `AssetTools`, `AssetRegistry`, `LevelEditor`, `Sequencer`/`LevelSequence`/`MovieScene*`, `MovieRenderPipeline*`, `Foliage`, `Landscape`, `WorldPartitionEditor`, `DataLayerEditor`, `ControlRig*`, `PoseSearch*`, `PythonScriptPlugin`, etc.
- Plugins éditeur déclarés dans le `.uplugin` (beaucoup `Optional`) : EditorScriptingUtilities, DataValidation, GameplayAbilities, EnhancedInput, Niagara, GameplayTagsEditor, GameFeatures, PCG, StateTree, Dataflow, WorldConditions, Paper2D, RemoteControl, LiveLink, Composure, MovieRenderPipeline, Datasmith/Interchange, Metasound, AudioModulation, Synthesis, ModelViewViewModel, DataRegistry, PoseSearch, Chooser, ContextualAnimation, DeformerGraph, MotionWarping, HairStrands, Takes, LevelSnapshots, NetworkPrediction, ReplicationGraph, OnlineSubsystem(+Utils), ZoneGraph, ProceduralMeshComponent, GeometryScripting, ChaosVehicles/ChaosCloth, Water, MassGameplay, SignificanceManager, PythonScriptPlugin.
- **Gating** (`Build.cs`) : chaque plugin optionnel donne `RZMCP_WITH_<X>=0|1` selon présence disque. À `0`, la bibliothèque correspondante n'est pas enregistrée (`SkippedLibraries`) et n'apparaît pas dans `list_tools`. Familles gardées : `RZMCP_WITH_GAS`, `_NIAGARA`, `_PCG`, `_STATETREE`, `_GAMEFEATURES`, `_DATAFLOW`, `_WORLDCONDITIONS`, plus les familles « umbrella » (`_MASS`, `_WATER`, `_COMPOSURE`, `_LIVELINK`, `_REMOTECONTROL`, `_METASOUND`, `_TAKES`, `_GEOMETRYSCRIPTING`, ...). `RZMCP_WITH_GAMEPLAY_EDITOR` n'est `1` que si GAS **et** EnhancedInput **et** ZoneGraph sont présents. `RZMCP_WITH_FOLIAGE`/`_DATALAYEREDITOR`/`_WORLDPARTITION_EDITOR` sont des modules moteur toujours-présents forcés à `1`.

**Dépendances sortantes (qui dépend du plugin)**
- **Aucun module C++ de QANGA ne référence RzDirectMCP.** Grep sur `G:/QANGA/Source` et les autres `Plugins/*` : aucun usage de `RZDIRECTMCP_API`, `URzMCPBridgeSubsystem`, ni des types des bibliothèques. Le plugin n'est pas non plus activé dans `QANGA.uproject`.
- L'intégration se fait donc **hors C++** : un client MCP externe se connecte au socket (directement, ou via `rzmcp-bridge.mjs`). C'est un outil de productivité éditeur, pas une brique de gameplay.

**Réglages / commandes**
- Config : `NativeBridgePort` dans `GEditorPerProjectIni`, section `/Script/RzDirectMCP.RzMCPBridgeSubsystem`.
- Env : `RZMCP_NATIVE_PORT` (côté éditeur ET côté `rzmcp-bridge.mjs`), `BRIDGE_HOST`/`BRIDGE_PORT` côté Node.
- Catégories de log : `LogRzMCPBridge`, `LogRzMCPNativeTools`, `LogNodeForge`.

**Empaquetage**
- `Config/FilterPlugin.ini` inclut `/Docs/...` et `/Source/RzMCP/...` dans le plugin packagé (pour livrer `rzmcp-bridge.mjs`, `specials.mjs`, `manifest.json` et la doc).

---

## 7. Gotchas, invariants et pièges

- **Deux comptes d'outils, pas une coquille.** 872 = outils natifs réflexifs (vérité éditeur, `get_tool_count`). 700 = outils client définis par `manifest.json`. Ne pas « réconcilier » ces nombres : ce sont deux surfaces différentes. Le compte natif réel dépend des plugins présents (gating).
- **La doc embarquée `Docs/ARCHITECTURE.md` reste un résumé utilisateur.** Le code réel utilise `FRzMCPBridgeWorker` (FRunnable) et un dispatch one-shot par `FTSTicker` bloquant. La méta de découverte est `RzMCPToolLibrary`; le suffixe `_exact` est un détail de mapping client/manifest, pas un mécanisme du registre natif.
- **Le thread serveur bloque pendant l'exécution.** `ExecuteOnGameThread` attend la fin sur le game thread (timeout 120 s). Un outil qui bloque le game thread bloque le bridge entier. Pas de parallélisme entre appels sur ce worker : connexions sérialisées.
- **Une connexion = un appel.** Pas de keep-alive. Tout client doit ré-ouvrir un socket par requête. Le parseur attend un objet JSON top-level unique terminé par accolade équilibrée (plafond 10 Mo).
- **Appel sur le CDO.** `ProcessEvent` est invoqué sur `LibraryClass->GetDefaultObject()`. Les fonctions doivent donc être de vraies static library functions sans état d'instance — ce que garantit le filtre `FUNC_Static | FUNC_BlueprintCallable`.
- **REINST / abstract ignorés.** Le registre saute `CLASS_Abstract | CLASS_NewerVersionExists` : une bibliothèque en cours de reinstancing (recompile à chaud) peut temporairement disparaître du catalogue.
- **Pas de bridge en commandlet.** `Initialize` sort tôt si `IsRunningCommandlet()`. Pas de bridge en cook/build headless — c'est volontaire.
- **Plateforme Win64 uniquement** (`PlatformAllowList`). `bUseUnity = false` (compilation non-unity pour ce module).
- **Ajouter un outil = ajouter une UFUNCTION dans une lib taguée + rebuild.** Aucune ré-écriture du registre. Mais Live Coding ne ré-enregistre pas une nouvelle `UFunction`/`UClass` : un nouvel outil exige un build complet, pas un patch LC.
- **Côté client, un outil cassé = entrée `dispatch` cassée.** La couche `manifest.json` mappe chaque outil client sur un outil natif via `kind` (`settings_merged`, `function_merged`, `raw_spec`, `native_passthrough`, `bridge_request`, ...). Une régression côté client peut venir du manifest sans que la couche native ait bougé.
- **Annotations viewport = pur CPU.** `RzMCP::BitmapAnnotation` n'utilise ni GPU ni Slate : sûr en capture headless, mais c'est un blit logiciel sur les pixels capturés (coût CPU proportionnel à la résolution).

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `G:/QANGA/Plugins/RzDirectMCP/RzDirectMCP.uplugin` | Descripteur : module `Editor`/`Default`/`Win64`, liste des plugins (beaucoup `Optional`). |
| `Source/RzDirectMCP/RzDirectMCP.Build.cs` | Dépendances + gating `RZMCP_WITH_*` par présence disque ; `bUseUnity=false`. |
| `Source/RzDirectMCP/Private/RzDirectMCPModule.cpp` | `FRzDirectMCPModule` (startup/shutdown vides). |
| `Source/RzDirectMCP/Private/RzMCPBridgeSubsystem.h/.cpp` | `URzMCPBridgeSubsystem` + `FRzMCPBridgeWorker` + dispatch game-thread + routage des `type`. |
| `Source/RzDirectMCP/Private/RzMCPNativeToolRegistry.h/.cpp` | Découverte réflexive, table d'outils, marshalling JSON↔FProperty, `ProcessEvent`. |
| `Source/RzDirectMCP/Private/RzMCPBitmapAnnotation.h/.cpp` | Moteur d'annotation bitmap CPU pour captures viewport. |
| `Source/RzDirectMCP/Private/Compiler/` | NodeForge : `NodeForgeOrchestrator`, `NodeChainAssembler`, `GraphTypeResolver`, `GraphLayoutEngine`, `SafeNodeHelpers` (génération de graphes Blueprint). |
| `Source/RzDirectMCP/Public/*.h` | 40 headers de bibliothèques `RzMCPToolLibrary` + `MCPEditorHelpers.h` (utilitaire JSON, non-taguée). |
| `Source/RzDirectMCP/Private/*.cpp` | Implémentations des bibliothèques d'outils (incl. l'« umbrella » `ExtendedEditorLibrary`). |
| `Source/RzMCP/rzmcp-bridge.mjs` | Serveur MCP stdio Node.js (sans dépendance), client du socket TCP. |
| `Source/RzMCP/specials.mjs` | Handlers/textes d'aide spéciaux côté Node. |
| `Source/RzMCP/manifest.json` | Catalogue des 700 outils client + table `dispatch` (mapping vers outils natifs). |
| `Config/FilterPlugin.ini` | Inclut `Docs/` et `Source/RzMCP/` dans le plugin packagé. |
| `Docs/ARCHITECTURE.md`, `README.md`, `QUICKSTART.md`, `TOOLS.md`, `ADVANCED.md`, `PRICING.md` | Doc embarquée (utile mais à recouper — voir §7). |
