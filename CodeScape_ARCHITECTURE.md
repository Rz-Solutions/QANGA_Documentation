# CodeScape — Architecture

CodeScape Studio (`VersionName` 2.1.0, `EngineVersion` 5.7.0) — plugin **éditeur uniquement** (`Type: Editor`, `LoadingPhase: PostEngineInit`) qui transforme du langage naturel en assets Unreal réels (Blueprints, Materials, UMG, Niveaux, Niagara, Cinématiques, etc.) via un LLM et un registre de ~328 outils auto-décrits. C'est un outil de **création de contenu assistée par IA**, distinct du jeu QANGA : aucun code runtime de QANGA n'en dépend (voir §6).

> Source vérifiée : `CodeScape.uplugin`, `CodeScape.Build.cs`, `Public/*.h`, points d'entrée de modules et principaux sous-systèmes. Le plugin compte ~824 fichiers source ; les outils individuels (`Tools/<Catégorie>/Tool_*.h`) suivent tous le même contrat `ICodeScapeTool` et ne sont pas détaillés un par un.

---

## 1. But d'usage

Le problème concret résolu : permettre à un développeur de QANGA de **créer ou modifier des assets éditeur en décrivant ce qu'il veut**, au lieu de cliquer manuellement dans chaque éditeur Unreal. Une requête en langage naturel (« crée un BP_Enemy de type Character avec une logique OnBeginPlay qui… ») est envoyée à un fournisseur LLM, qui répond avec des appels d'outils JSON ; chaque outil exécute une mutation éditeur réelle (création de Blueprint, génération de nodes K2, instanciation de Material, etc.).

Deux particularités de conception qui en font plus qu'un wrapper LLM générique :

1. **NodeForge** — un compilateur maison qui traduit un DSL (proche du C++) en graphes de nodes Blueprint (`UEdGraphNode`/`UK2Node`) réellement câblés et compilés. C'est le cœur différenciant : l'IA écrit du « pseudo-code » et NodeForge le matérialise en logique Blueprint valide.
2. **Double surface d'accès** — les mêmes outils sont exposés (a) dans une UI de chat dockée dans l'éditeur, et (b) via un serveur TCP **MCP** (Model Context Protocol) permettant à des clients IA externes (Claude Desktop, etc.) de piloter l'éditeur. Le registre d'outils est partagé entre les deux surfaces.

Le plugin embarque aussi `CanContainContent: true` (assets/styles bundlés sous `Content/`, `Resources/`).

---

## 2. Vue d'ensemble / décisions d'architecture

Le flux principal est une boucle **LLM → tool call → mutation éditeur → tool result → LLM** :

```
Utilisateur (UI chat / client MCP externe)
        │
        ▼
FCodeScapeChatSession  ──build prompt──►  FAISystemPromptBuilder (+ ToolRegistry::GenerateToolDefinitions)
        │  (HTTP)                                    + FRulePackManager + contexte projet/asset
        ▼
Fournisseur LLM (Gemini / OpenAI / Claude / DeepSeek / MiniMax / Groq / Mistral / Cohere / Grok / GLM / Custom)
        │  (réponse contenant des appels d'outils JSON)
        ▼
FAIResponseParser  ──►  extraction tool call(s)
        │
        ▼
FToolRegistry::DispatchTool(name, args)  ──►  ICodeScapeTool::Execute()
        │                                              │
        │                                              ├─ outils Blueprint logique ─► FNodeForgeOrchestrator
        │                                              │      └─ FNodeChainAssembler + FGraphTypeResolver + FGraphLayoutEngine
        │                                              └─ ~328 outils par catégorie (Material, Niagara, Level, …)
        ▼
FToolExecutionResult (JSON)  ──►  réinjecté dans l'historique  ──►  nouveau tour LLM (chaînage)
```

Décisions clés :

- **Outils auto-décrits, registre singleton.** `ICodeScapeTool` impose à chaque outil de fournir son `GetToolName/GetDescription/GetArgumentsSchema/GetUsageExample`. La section « AVAILABLE TOOLS » du system prompt est **auto-générée** depuis `FToolRegistry`. Ajouter un outil = créer un fichier `Tool_*.h/.cpp` + une ligne dans `ToolRegistration.cpp`. Au démarrage, `FCodeScapeModule::StartupModule` log un avertissement si moins de ~300/316 outils sont enregistrés.
- **Registre partagé chat + MCP.** Le même `FToolRegistry` sert l'UI in-editor et le thread TCP MCP ; il est protégé par un `FRWLock` (`RegistryLock`) car `HasTool/GetAllToolNames/GenerateToolDefinitions` sont appelés depuis le thread TCP MCP tandis que les `RegisterTool` n'ont lieu qu'au startup sur le GameThread.
- **Budget de prompt maîtrisé.** Le registre peut générer un sous-ensemble d'outils *classés par pertinence* à la requête (`GenerateRelevantToolDefinitions(RequestText, MaxTools, MaxChars)`) pour ne pas saturer le contexte. Le `FCodeScapeChatSession` gère en plus la compaction de l'historique (résumé IA), la troncature des résultats d'outils (`MaxToolResultTokens*`), la détection de réponses incomplètes et la détection de boucles d'échec.
- **NodeForge = compilateur dédié, pas du templating.** Le pipeline `FNodeForgeOrchestrator → FNodeChainAssembler → FGraphTypeResolver → FGraphLayoutEngine` parse un DSL, classe chaque bloc (`EBlockKind`), résout types/fonctions C++ vers le système de pins Blueprint (avec aliases et fuzzy-match Levenshtein), assemble et câble les nodes, puis applique un layout. Il gère l'idempotence (purge de la chaîne exec downstream), un mode Replace/Append (`EEventRegenMode`) et une pile d'undo statique (`GenerationHistory`).
- **Multi-fournisseur, OpenAI-compatible par défaut.** `EApiProvider` énumère 11 backends ; `FProviderEntry` porte URL/clé/modèles. Enum **append-only** (commentaire explicite : insérer avant `Custom` décalerait silencieusement les `providers.json` sauvegardés).
- **Infrastructure transverse.** `FCSInfrastructure` orchestre journalisation d'activité, backups d'assets avant mutation, analytics d'usage, profils, plans et rétention — toutes écritures asynchrones pour ne pas bloquer le GameThread.
- **Éditions de build.** La variable d'env `CODESCAPE_EDITION` (`studio` par défaut, ou `free`) pilote des `PrivateDefinitions` : `CODESCAPE_FREE_EDITION`, `CODESCAPE_ENABLE_EXTERNAL_MCP`, `CODESCAPE_ENABLE_API_REFERENCE_DATA`, `CODESCAPE_QUIET_RELEASE`. En `free`, le serveur MCP externe et les données de référence API sont désactivés, et le module se précompile/distribue en object code. Plusieurs intégrations sont optionnelles : `WITH_POSE_SEARCH` (détecté via présence du plugin), `WITH_PIXEL_STREAMING=0`, `WITH_PYTHON_SCRIPT=0` (les outils retombent sur `GEngine->Exec`).

État réel vs intention : le plugin est fonctionnel et large (328 outils enregistrés via `ToolRegistration.cpp`). Certaines dépendances de modules sont commentées dans `Build.cs` (`MovieRenderPipelineCore`, `LiveLinkInterface`) — les outils correspondants existent mais tombent en « not available » au runtime tant que ces modules ne sont pas liés.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `FCodeScapeModule` | `IModuleInterface` | Point d'entrée du module : style, commandes, menus/toolbars, onglet dockable, enregistrement des outils/tests, infrastructure, intégration éditeurs d'assets. |
| `ICodeScapeTool` | Interface C++ | Contrat auto-descriptif de tout outil (`GetToolName/Description/ArgumentsSchema/UsageExample/Execute`). |
| `FToolRegistry` | Singleton | Registre central O(1) des outils ; dispatch, génération des définitions pour le prompt (full / JSON MCP / pertinence-classée), filtrage par catégorie activée. Thread-safe (`FRWLock`). |
| `FToolExecutionResult` / `FToolCallExtractionResult` / `FMultiToolExtractionResult` | structs | Résultat d'exécution d'outil et résultats d'extraction (mono/multi tool-call). |
| `Tool_*` (~328) | classes `ICodeScapeTool` | Outils par catégorie sous `Tools/<Catégorie>/` (Blueprint, Material, Level, Niagara, Sequencer, GAS, EnhancedInput, PCG, AnimBP, Widget, Rendering, …). |
| `FCodeScapeChatSession` | classe | Logique de session de chat partagée : cycle HTTP, dispatch tool calls, parsing, retry/loop-detection, compaction IA, profondeur de chaînage. Pilote la boucle LLM↔outils. |
| `ICodeScapeChatSessionOwner` | Interface | Découplage session↔widget (résolution provider, system prompt, persistance). |
| `FAIProviderManager` | classe | Charge/sauve/migre les fournisseurs (`Saved/CodeScape/providers.json`) ; sync des clés depuis les settings éditeur. |
| `EApiProvider` / `FProviderEntry` | enum / struct | 11 fournisseurs LLM (append-only) ; URL/clé/modèles par défaut. |
| `FAISystemPromptBuilder` | classe | Génère le system prompt (cloud/local) en assemblant définitions d'outils + contexte projet/asset. |
| `FAIRequestBuilder` / `FAIResponseParser` / `FAIContextManager` | classes | Construction des requêtes (+ `FTokenUsageData`), parsing provider-agnostique des tool calls et détection d'incomplétude, gestion du contexte (incl. `FCodeScapeImageAttachment` multimodal). |
| `FPromptReformulator` | classe | Reformule/structure/traduit l'input utilisateur via un appel LLM one-shot avant envoi. |
| `FNodeForgeOrchestrator` | classe | Compilateur DSL→graphe K2 : parse signatures, classe blocs (`EBlockKind`), émet chaînes d'événements/fonctions, undo (`GenerationHistory`), modes Replace/Append (`EEventRegenMode`), validation post-génération (`FForgeValidationResult`). |
| `FNodeChainAssembler` | classe | Transforme les lignes DSL en chaînes de K2 nodes câblés ; opère sur `FForgeSessionContext`. |
| `FGraphTypeResolver` | classe | Résout types C++→pins Blueprint, découvre Enum/Struct/Class/DataTable, résout fonctions (exact→alias→fuzzy Levenshtein), mappe les events DSL→UE. |
| `FGraphLayoutEngine` | classe | Disposition automatique des nodes générés. |
| `FForgeSessionContext` / `FParameterDef` / `FTimelinePendingLogic` / `FForgeValidationResult` | structs | État mutable d'une session forge, paramètres, logique timeline différée, rapport de validation. |
| `UMCP_EditorSubsystem` | `UEditorUtilitySubsystem` | Héberge le serveur TCP MCP (lifecycle) ; dispatch des commandes JSON sur le GameThread ; utilitaires partagés (ex. `HandleSetupBlueprintComponents`). |
| `FMCPListenerThread` | `FRunnable` | Thread d'écoute TCP : accepte connexions, valide un `SessionToken` par session, lit/écrit le JSON, délègue au subsystem. |
| `UCodeScapeAssetChatSubsystem` | `UEditorSubsystem` | Gère les conversations de chat dockées par asset (manifeste partagé, délégué `FOnAssetChatSaved`). |
| `UCodeScapeProjectSettings` | `UDeveloperSettings` (`config=Game`) | Réglages partagés (versionnés) : dossier dev, serveur MCP, toggles par catégorie d'outils, NodeForge, intégration éditeurs d'assets. |
| `UCodeScapeEditorSettings` | `UDeveloperSettings` (`config=EditorPerProjectUserSettings`) | Réglages par utilisateur (non versionnés) : provider par défaut, clés API, chat, UI, avancé (reasoning, timeouts, retries), gestion de contexte. |
| `ECodeScapeDefaultProvider` | `UENUM` | Provider par défaut sélectionnable dans les Editor Preferences. |
| `FCSInfrastructure` | Singleton | Orchestrateur des sous-systèmes d'infrastructure (init/shutdown unique). |
| `FCSActivityLogger` / `FCSAssetBackup` / `FCSUsageAnalytics` / `FCSProfileManager` / `FCSPlanStore` / `FCSRetentionManager` / `FCSBugReporter` | classes | Journal JSONL des appels d'outils ; backup d'asset avant mutation ; stats par outil ; profils de réglages ; plans ; rétention ; rapport de bug. |
| `FRulePackManager` / `FRulePack` | Singleton / struct | « Rule packs » injectés dans le system prompt (defaults plugin + overrides + état activé). |
| `FTemplateLibrary` / `FTemplateAction` | Singleton / struct | Templates de prompts curatés (defaults plugin + overrides utilisateur). |
| `Analyzers` (`FBpSummarizer`, `FBpFlowTracer`, `FBpGraphValidator`, `FMaterialGraphDescriber`, `FPCGGraphDescriber`, `FDeepAssetContextBuilder`, …) | classes utilitaires | Résument/décrivent/valident des assets (Blueprint, Material, Behavior Tree, PCG) pour fournir du contexte compact à l'IA. |
| `SCodeScapeWidget` / `SCodeScapeChatCore` / `SCodeScapeAssetChat` / `SConversationListRow` / `SBugReportDialog` / `SCodeScapeWelcomeDialog` | widgets Slate | UI de chat principale, cœur partagé, chat docké asset, listes, dialogues. |
| `FCodeScapeAssetEditorIntegration` | classe | Injecte l'onglet chat CodeScape dans les éditeurs d'assets (BP/Material/Widget/AnimBP/BehaviorTree). |
| `FCodeScapeCommands` / `FCodeScapeStyle` / `FCodeScapePaths` / `FContentBrowserPathResolver` | classes éditeur | Commandes UI, style Slate, résolution de chemins (dossier développeur), résolution de chemin Content Browser. |
| `FCodeScapeProject` / `FConversationInfo` / `FProjectKnowledgeAsset` / `FProjectFolderContext` | structs (ChatTypes) | Modèle de « projet » CodeScape : instructions, base de connaissances d'assets, dossiers de contexte, conversations. |
| **CodeScapeMCP** (`CodeScapeMCP.exe`) | exécutable C++ autonome (hors module UE) | Bridge MCP : parle JSON-RPC 2.0 MCP sur stdio, forwarde les tool calls en TCP vers `MCPListenerThread`. Composants : `MCPProtocolHandler`, `MCPBridgeClient`, `MCPToolDefinitions`. |

---

## 4. Flux de données et cycle de vie

**Démarrage du module** (`FCodeScapeModule::StartupModule`, `PostEngineInit`) :
1. Style (`FCodeScapeStyle::Initialize`) + commandes (`FCodeScapeCommands::Register`).
2. Bindings d'actions (ouvrir l'onglet, ouvrir les settings).
3. Menus/toolbars différés via `UToolMenus::RegisterStartupCallback` ; entrées ajoutées dans les fenêtres `LevelEditor`, `BlueprintEditor`, `AnimationBlueprintEditor`, `MaterialEditor`, `BehaviorTreeEditor`, `WidgetBlueprintEditor` (table `GEditorWindowMenus` / `GAssetEditorToolbars`).
4. Onglet dockable nomade (`RegisterNomadTabSpawner`, catégorie Tools).
5. Suivi de sélection du Content Browser (`OnContentBrowserSelectionChanged`) pour auto-détecter le Blueprint cible.
6. **`RegisterAllCodeScapeTools()`** — peuple le `FToolRegistry` (~328 outils) ; log d'avertissement si < 300.
7. `RegisterAllCodeScapeTests()` + `FCodeScapePaths::EnsureDeveloperFolderExists()`.
8. Backend WebBrowser (`IWebBrowserModule::Get()`, requis pour le rendu du chat).
9. `FCodeScapeAssetEditorIntegration::Initialize()` (onglet chat injecté dans les éditeurs d'assets).
10. Délégués de changement de settings (projet + éditeur).
11. `FCSInfrastructure::Get().Initialize()` (logging/backups/analytics/profils/plans/rétention).

**Runtime — une requête de chat** :
- L'owner (`SCodeScapeWidget` ou `SCodeScapeAssetChat`) construit le system prompt via `FAISystemPromptBuilder` (définitions d'outils auto-générées + rule packs activés + contexte projet/asset), puis `FCodeScapeChatSession::SendRequest()` lance la requête HTTP au provider résolu.
- `OnApiResponseReceived` → `ProcessResponse` : `FAIResponseParser` extrait le(s) tool call(s) ; chaque appel passe par `FToolRegistry::DispatchTool` (qui respecte les toggles de catégorie des Project Settings) puis `ICodeScapeTool::Execute`.
- Les outils de **logique Blueprint** délèguent à `FNodeForgeOrchestrator::CompileToGraph` (parse → classifie `EBlockKind` → assemble via `FNodeChainAssembler`/`FGraphTypeResolver` → layout `FGraphLayoutEngine` → optionnellement compile le BP selon `bCompileAfterGeneration`).
- Le `FToolExecutionResult` (JSON) est réinjecté dans l'historique et un nouveau tour LLM est déclenché (chaînage, profondeur `MaxToolCallDepth`, `-1 = illimité`). La session gère retry, détection de réponse incomplète (`ConsecutiveIncompleteCount`), détection de boucle d'échec (`MaxSameToolFailures`), rate-limit (`MinRequestIntervalSeconds`) et compaction d'historique.
- En parallèle, `FCSActivityLogger` journalise chaque appel d'outil (JSONL async) et `FCSAssetBackup` snapshot l'asset avant toute mutation.

**Runtime — accès MCP externe** :
- `UMCP_EditorSubsystem::StartServer()` ouvre un socket TCP (`MCPServerPort`, défaut 8765) et démarre `FMCPListenerThread`. Le client externe se connecte via `CodeScapeMCP.exe` (stdio JSON-RPC 2.0 ↔ TCP). Chaque requête est validée contre le `SessionToken` puis dispatchée sur le GameThread via le même `FToolRegistry`.

**Teardown** (`ShutdownModule`) : `FCSInfrastructure::Shutdown()` (flush) en premier ; opérations dépendantes des UObject gardées par `UObjectInitialized()` (CDOs potentiellement détruits en fin de shutdown) ; retrait des délégués, démontage de l'intégration éditeurs, retrait du délégué Content Browser, désenregistrement de l'onglet, du style et des commandes.

---

## 5. Réplication / réseau

**Le plugin ne réplique rien et n'a aucun état réseau de jeu.** C'est un plugin éditeur (`Type: Editor`) sans acteur/composant runtime répliqué : aucun `GetLifetimeReplicatedProps`, aucune RPC `Server_/Client_/Multicast_`. (Les fichiers contenant le mot « Replicated » sont des outils réseau — `Tool_ConfigureReplication.cpp`, `Tool_ManageNetworkingAdvanced.cpp`, `Tool_ManageReplicationGraph.cpp`, `Tool_EditVariableProperties.cpp`, etc. — qui *éditent* les réglages de réplication d'autres assets ; ils ne sont pas eux-mêmes répliqués.)

Le seul « réseau » présent est un **serveur TCP local** (`UMCP_EditorSubsystem` + `FMCPListenerThread`) destiné aux clients IA externes sur `localhost`, validé par un `SessionToken` par session. Il n'a aucun rapport avec la réplication gameplay de QANGA. Le LLM est joint par **HTTP** (modules `HTTP`/`Json`) côté `FCodeScapeChatSession`.

---

## 6. Points d'intégration

**Dépendances entrantes (ce que CodeScape requiert)** — voir `CodeScape.Build.cs` et `CodeScape.uplugin`. Le plugin lie un très grand éventail de modules **éditeur** pour pouvoir manipuler chaque domaine : `UnrealEd`, `Kismet`/`KismetCompiler`/`BlueprintGraph`/`GraphEditor`/`Blutility`, `UMG`/`UMGEditor`, `MaterialEditor`, `Niagara*`, `AnimGraph`, `Foliage`/`Landscape`, `PCG`, `EnhancedInput`, `GameplayAbilities`, `IKRig`, `StateTreeModule`/`SmartObjectsModule`, `Metasound*`, `ControlRig`, `MassEntity`, `GeometryScripting*`, `HairStrands*`, `TakeRecorder`, `RemoteControl`, `GameFeatures`, `Water*`, `ChaosVehicles`/`GeometryCollectionEngine`, `Sockets`/`Networking` (MCP), `HTTP`/`Json`/`JsonUtilities` (LLM), `WebBrowser`/`WebBrowserWidget` (rendu chat). Plugins activés via `.uplugin` : `EditorScriptingUtilities`, `Niagara`, `WebBrowserWidget`, `PCG`, `EnhancedInput`, `IKRig`, `GameplayAbilities`, `GameplayTagsEditor`, `ChaosVehiclesPlugin`, `Water`, `StateTree`, `SmartObjects`, `Metasound`, `MotionWarping`, `ControlRig`, `MassGameplay`, `GeometryScripting`, `HairStrands`, `Takes`, `RemoteControl`, `GameFeatures`, `OnlineSubsystem(+Utils)`, `AudioModulation`, `Synthesis`, `PoseSearch` (optionnel), `CableComponent`.

**Dépendances sortantes (ce qui dépend de CodeScape)** — **aucun module de QANGA.** Recherche des types publics (`UCodeScapeProjectSettings`, `FToolRegistry`, `ICodeScapeTool`, `UMCP_EditorSubsystem`) dans `G:/QANGA/Source` et dans les autres plugins : zéro référence. CodeScape est un **outil d'auteur autonome** : il agit *sur* les assets du projet (Content/Blueprints/Materials…) mais le jeu ne l'appelle jamais. Il peut être désactivé/retiré du build runtime sans impact gameplay.

**Réglages & ports pertinents** :
- Project Settings → Plugins → CodeScape (`UCodeScapeProjectSettings`) : `DeveloperContentPath` (défaut `/Game/Developers/CodeScape/`), `bEnableMCPServer`, `MCPServerPort` (défaut **8765**), `bAutoStartMCPOnLoad`, ~30 toggles `bEnable*Tools`, `MaxToolCallDepth` (-1 = illimité), `ToolExecutionTimeoutSeconds`, options NodeForge (`bAutoLayoutAfterGeneration`, `bCompileAfterGeneration`, `NodeSpacingX/Y`), injection par éditeur d'asset.
- Editor Preferences → Plugins → CodeScape (`UCodeScapeEditorSettings`) : provider par défaut, **clés API** par fournisseur (`PasswordField`, stockées localement), historique de chat, UI, options reasoning (Gemini/OpenAI), timeouts/retries HTTP, gestion de contexte (`MaxToolResultTokens*`, `CloudCompressionThreshold`, `MidChainCompressionDepth`).
- Le bridge externe `CodeScapeMCP.exe` se lance par défaut sur le port **8765** (aligné sur `MCPServerPort`), configurable via `--port`. Côté `manifest.json`/`.mcpb`, il est déclaré en serveur stdio pour les clients MCP.

**Fichiers d'état sur disque** : `Saved/CodeScape/providers.json` (fournisseurs), `Saved/CodeScape/Backups/` (snapshots avant mutation), logs d'activité JSONL, `user_rule_packs.json`/`enabled_rule_packs.json`, `user_templates.json`. Defaults bundlés : `Resources/DefaultRulePacks.json`, `Resources/DefaultTemplates.json`.

---

## 7. Gotchas, invariants et pièges

- **`EApiProvider` est append-only.** Les `providers.json` persistent la valeur entière de l'enum. Insérer un provider avant `Custom` décale silencieusement tous les fichiers sauvegardés (commentaire explicite dans `AIProviderTypes.h`). `GLM` a été ajouté *après* `Custom` pour cette raison.
- **`FToolRegistry` est lu depuis le thread TCP MCP.** Toute lecture (`HasTool/GetAllToolNames/GenerateToolDefinitions`) doit passer par le `FRWLock`. Les `RegisterTool` ne doivent avoir lieu qu'au startup sur le GameThread — ne jamais enregistrer un outil à chaud depuis un autre thread.
- **Ajouter un outil = 1 fichier + 1 include.** Un `Tool_*.h/.cpp` conforme à `ICodeScapeTool` est invisible tant qu'il n'est pas inclus/enregistré dans `ToolRegistration.cpp`. Le startup log un avertissement sous le seuil ~300/316 ; un compte anormalement bas pointe un include manquant.
- **NodeForge — idempotence et modes de régénération.** `generate_blueprint_logic` répété sur un même event purge la chaîne exec downstream (`PurgeExecChainDownstream`) avant de réémettre. Le mode par défaut est `EEventRegenMode::Replace` (détruit la logique existante) ; `Append` insère via un node Sequence. Mauvais mode = perte de logique existante ou doublons.
- **Résolution de types tolérante mais faillible.** `FGraphTypeResolver` résout les fonctions par exact→alias→fuzzy Levenshtein et découvre Enum/Struct/Class via cache `TWeakObjectPtr` (un nullptr caché = « déjà cherché, introuvable »). Après ajout d'assets à chaud, appeler `InvalidateTypeCache()` sinon de nouveaux types restent invisibles. La résolution fuzzy peut câbler une fonction proche mais incorrecte — valider via `FForgeValidationResult`.
- **Le serveur MCP est local et token-gardé.** Chaque requête TCP est validée contre le `SessionToken` généré à `Init()`. Le bridge `CodeScapeMCP.exe` doit utiliser le même port que `MCPServerPort` (défaut 8765) ; un décalage de port = bridge muet.
- **Shutdown gardé par `UObjectInitialized()`.** En fin de shutdown moteur le sous-système UObject peut déjà être détruit ; `GetMutableDefault<>()`/`UToolMenus` planteraient. Tout accès UObject dans `ShutdownModule` doit rester sous cette garde.
- **Éditions free vs studio.** `CODESCAPE_EDITION=free` désactive le MCP externe et les données de référence API, force la distribution en object code et coupe `PoseSearch`. Ne pas supposer que tous les outils/chemins sont actifs sans vérifier les `PrivateDefinitions` du build courant. De même `WITH_PIXEL_STREAMING`/`WITH_PYTHON_SCRIPT` sont à `0` par défaut : les outils correspondants retombent sur un fallback « not available » / `GEngine->Exec`.
- **Infrastructure d'abord au shutdown.** `FCSInfrastructure::Shutdown()` doit flush *avant* tout autre teardown pour ne pas perdre logs/backups/analytics.
- **Plugin éditeur uniquement.** Aucune logique runtime/gameplay ; ne jamais l'inclure dans un build serveur/shipping de jeu ni dépendre de ses types depuis du code runtime QANGA.

---

## 8. Fichiers et emplacements

| Fichier / dossier | Rôle |
|---|---|
| `CodeScape.uplugin` | Manifeste plugin : module unique `CodeScape` (`Editor`, `PostEngineInit`), liste des plugins requis. |
| `Source/CodeScape/CodeScape.Build.cs` | Dépendances de modules, éditions (`CODESCAPE_EDITION`), defines optionnels (`WITH_POSE_SEARCH/PIXEL_STREAMING/PYTHON_SCRIPT`). |
| `CLAUDE.md` | Règles de travail + carte de la pipeline principale (`ChatSession`/`ToolRegistry`/`NodeForge…`). |
| `Source/CodeScape/Private/CodeScapeModule.cpp` | Cycle de vie du module ; tables d'intégration éditeurs ; enregistrement outils/tests/infra. |
| `Public/CodeScapeModule.h` | Déclaration `FCodeScapeModule`. |
| `Public/ICodeScapeTool.h` / `ToolRegistry.h` / `ToolTypes.h` | Contrat d'outil, registre central, structs de résultat. |
| `Private/Tools/ToolRegistration.cpp` | Liste exhaustive des includes + `RegisterTool` (~328 outils). |
| `Private/Tools/<Catégorie>/Tool_*.{h,cpp}` | Implémentations d'outils par domaine (Blueprint, Material, Level, Niagara, Sequencer, GAS, EnhancedInput, PCG, AnimBP, Widget, Rendering, …). |
| `Private/Chat/CodeScapeChatSession.{h,cpp}` | Boucle LLM↔outils : HTTP, dispatch, retry, compaction, chaînage. |
| `Private/Chat/ChatTypes.h` / `ChatPersistence.*` / `ProjectPersistence.*` | Modèle de conversation/projet et persistance. |
| `Private/AI/AIProviderManager.*` | Chargement/sauvegarde des fournisseurs (`providers.json`). |
| `Public/AIProviderTypes.h` | `EApiProvider`, `FProviderEntry`, modèles/URLs par défaut. |
| `Private/AI/AISystemPromptBuilder.*` / `AIRequestBuilder.*` / `AIResponseParser.*` / `AIContextManager.*` / `PromptReformulator.*` | Construction du prompt, des requêtes, parsing des tool calls, contexte, reformulation. |
| `Private/Compiler/NodeForgeOrchestrator.{h,cpp}` | Orchestrateur DSL→graphe K2 (entrée des outils de logique Blueprint). |
| `Private/Compiler/NodeChainAssembler.*` / `GraphTypeResolver.*` / `GraphLayoutEngine.*` | Assemblage des nodes, résolution de types/fonctions, layout. |
| `Private/MCP/MCP_EditorSubsystem.*` / `MCPListenerThread.*` | Serveur TCP MCP in-editor (lifecycle + thread d'écoute token-gardé). |
| `Source/CodeScapeMCP/` | Exécutable C++ autonome `CodeScapeMCP.exe` : bridge MCP stdio↔TCP (`MCPProtocolHandler`, `MCPBridgeClient`, `MCPToolDefinitions`, `json.hpp`). |
| `Private/Subsystems/CodeScapeAssetChatSubsystem.*` | Conversations de chat dockées par asset. |
| `Public/CodeScapeProjectSettings.h` / `Public/CodeScapeEditorSettings.h` | Réglages projet (versionnés) et utilisateur (par user). |
| `Public/Infrastructure/CS*.h` + `Private/Infrastructure/CS*.cpp` | Logging d'activité, backups, analytics, profils, plans, rétention, bug reporter (orchestrés par `FCSInfrastructure`). |
| `Private/RulePacks/` / `Private/Templates/` | Rule packs et templates de prompts (defaults plugin + overrides). |
| `Private/Analyzers/` | Résumeurs/descripteurs/validateurs d'assets (Blueprint, Material, BT, PCG) pour le contexte IA. |
| `Private/UI/` (`SCodeScape*`, `CodeScapeAssetEditorIntegration.*`, `SBugReportDialog.*`, `SCodeScapeWelcomeDialog.*`) | Widgets Slate de chat + injection d'onglet dans les éditeurs d'assets. |
| `Private/Core/` (`CodeScapeCommands`, `CodeScapeStyle`, `CodeScapePaths`, `ContentBrowserPathResolver`, settings .cpp) | Commandes UI, style, chemins, résolution Content Browser. |
| `Private/Testing/` | Harnais de tests (`CodeScapeTestRunner`, sandbox, `FAutoConsoleCommand` d'exécution). |
| `Resources/DefaultRulePacks.json` / `DefaultTemplates.json` | Données par défaut bundlées (rule packs, templates). |
| `manifest.json` / `unreal-handshake.mcpb` | Déclaration du serveur MCP stdio pour les clients IA externes. |
