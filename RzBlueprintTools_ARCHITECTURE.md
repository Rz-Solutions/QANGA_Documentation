# RzBlueprintTools — Architecture

Plugin editeur-seul (UE 5.7), auteur Rz, version `2.0`. Module unique `RzBlueprintTools` de `Type: Editor`, `LoadingPhase: PostEngineInit`, `EnabledByDefault: true`, `CanContainContent: false`. Aucun code runtime, aucun UObject, aucun asset de contenu — c'est purement de l'outillage d'edition de Blueprints. Statut : fonctionnel, deux outils livres (organisation de graphe + verification de compilation projet-wide).

## 1. But d'usage

QANGA est un projet majoritairement Blueprint au-dessus de ses plugins C++. Deux problemes concrets de maintenance recurrents motivent ce plugin :

1. **Hygiene de layout des graphes.** Les Event Graphs et fonctions Blueprint deviennent illisibles : noeuds empiles, flux d'execution non aligne, noeuds de donnees disperses loin de leurs consommateurs. `FGraphOrganizer` reorganise le graphe *focalise* dans l'editeur Blueprint ouvert : il aligne le flux d'execution en colonnes/lanes, et place chaque noeud de donnee sous son consommateur principal. Il ne modifie jamais la topologie (ne cree, ne supprime, ne reconnecte aucun pin) — il deplace uniquement les noeuds et redimensionne les commentaires existants.

2. **Detection en masse des Blueprints casses.** Sur un projet avec des centaines de Blueprints (et apres des migrations comme 5.3→5.7), savoir *lesquels* ne compilent plus n'est pas trivial : l'editeur ne recompile pas tout au boot. `FBlueprintVerifier` parcourt l'Asset Registry, compile chaque Blueprint d'un scope choisi (`/Game/`, plugins, `/Engine/`) en silence, agrege erreurs/warnings et ecrit un rapport texte horodate sous `Saved/BlueprintVerification`.

Ce n'est pas un substitut d'outils tiers complets (Blueprint Assist, etc.) ni un correcteur automatique : il ne corrige aucune erreur de compilation, ne formate pas en continu pendant l'edition, et n'utilise aucun service externe.

## 2. Vue d'ensemble / decisions d'architecture

Le plugin se reduit a un module editeur (`FRzBlueprintToolsModule`) qui, au `StartupModule`, enregistre des extensions de menu/toolbar via `UToolMenus`, plus une `FAutoConsoleCommand` statique enregistree au chargement de la traduction du fichier objet. Deux classes de logique pure (non-UObject, methodes statiques) font le travail :

- **`FGraphOrganizer`** — algorithme de layout deterministe operant sur un `UEdGraph`.
- **`FBlueprintVerifier`** — scan + compilation + rapport, operant sur l'Asset Registry.

Decisions notables, telles que le code les implemente reellement :

- **Aucune classe UObject, aucune UCLASS/USTRUCT/UENUM/UINTERFACE.** Les structs (`FNodePlacement`, `FBlueprintCompileResult`, `FBlueprintVerificationReport`, `FBlueprintVerifierSettings`) sont des structs C++ plain, sans `UPROPERTY`. Tout est statique et stateless entre appels. Consequence : rien n'est reflechi/scriptable depuis Blueprint ou Python ; les outils ne sont accessibles que par l'UI editeur et la console.

- **Le verifier ne s'appuie que sur l'Asset Registry + Kismet.** Il appelle `AssetRegistry.SearchAllAssets(true)` (scan synchrone complet) puis `GetAssetsByClass(UBlueprint::StaticClass()->...)`, charge chaque asset avec `LOAD_NoWarn | LOAD_DisableCompileOnLoad`, et compile via `FKismetEditorUtilities::CompileBlueprint` avec `EBlueprintCompileOptions::SkipGarbageCollection` et un `FCompilerResultsLog` en `bSilentMode`. Le GC est declenche manuellement toutes les 10 s pour borner la consommation memoire pendant un gros scan.

- **L'organizer est garde au schema K2/Blueprint uniquement.** Il sort tot si le nom de classe du schema ne contient ni `K2` ni `Blueprint` — pas de support Material/Niagara/PCG/Control Rig, par design. C'est de l'auto-discovery via nom de classe (heuristique chaine), pas via cast de type.

- **L'organizer separe execution et donnees.** Un noeud est "data node" s'il n'a aucun pin d'execution (`HasExecutionPins`). Les noeuds d'execution sont parcourus en BFS depuis des racines (events, function entries, input nodes, ou noeuds sans exec entrant) et indexes en `Column`/`Lane`. Les noeuds de donnees sont ensuite empiles sous leur "ultimate consumer" d'execution (resolution recursive a travers les chaines data→data).

- **Coherence honnete avec l'intention.** Le `Docs/README.md` du plugin decrit precisement ce comportement ; aucune divergence code/doc constatee. Le seul point a noter est que la "preservation des reroute nodes" annoncee dans la doc est obtenue par construction (l'organizer ne traite jamais la topologie ni les liens), pas par un traitement special des reroutes.

## 3. Carte des composants

| Element | Type | Role |
| --- | --- | --- |
| `FRzBlueprintToolsModule` | `class : IModuleInterface` | Module editeur. `StartupModule`/`ShutdownModule` → `RegisterMenus`/`UnregisterMenus`. Aucun etat persistant. |
| `FGraphOrganizer` | `class` (methodes statiques) | Reorganisation de layout d'un `UEdGraph` K2. Point d'entree unique `OrganizeGraph(UEdGraph*)`. |
| `FNodePlacement` | `struct` (plain, header `GraphOrganizer.h`) | Etat de placement par noeud : `Column`, `Lane`, `X/Y`, `Width/Height`, flags `bIsDataNode`/`bIsComment`, `DataPredecessors`, `PrimaryConsumer`, `StackIndex`. |
| `FBlueprintVerifier` | `class` (methodes statiques) | Scan + compilation + ecriture rapport. `VerifyAllBlueprints`, `WriteReportToFile`, `GetReportDirectory`, `CompileSingleBlueprint` (prive). |
| `FBlueprintVerifierSettings` | `struct` (plain) | Scope du scan : `bIncludeGameContent`/`bIncludePlugins`/`bIncludeEngineContent`. `GetPathFilters()` derive les prefixes de chemin. |
| `FBlueprintCompileResult` | `struct` (plain) | Resultat par Blueprint : chemin, nom, `NumErrors`/`NumWarnings`, listes de messages, `HasIssues()`. |
| `FBlueprintVerificationReport` | `struct` (plain) | Agregat du scan : compteurs totaux, `Results`, `Timestamp`, `GetSummary()`. |
| `SBlueprintVerifierDialog` | `class : SCompoundWidget` | Dialog Slate (3 cases a cocher de scope) construit pour le bouton de menu. Defini dans `RzBlueprintToolsModule.cpp`. |
| `Rz.VerifyBlueprints` | `FAutoConsoleCommand` | Commande console : verifie et ecrit le rapport. Args : `[Game] [Plugins] [Engine]` (defaut : Game). |
| `LogRzBlueprintTools` | categorie de log statique | Log de la commande console (defini via `DEFINE_LOG_CATEGORY_STATIC`). |

## 4. Flux de donnees et cycle de vie

**Init (PostEngineInit).** `StartupModule` → `RegisterMenus`, qui differe l'extension via `UToolMenus::RegisterStartupCallback`. Dans le callback :
- Etend `AssetEditor.BlueprintEditor.ToolBar` → section `RzBlueprintTools`, bouton `Organize` (icone `Icons.Layout`). Action : `ExecuteOrganizeGraph`, gardee par `CanExecuteOrganizeGraph`.
- Etend `LevelEditor.MainMenu.Tools` → section `BlueprintTools` ("Blueprint Tools"), entree `Verify All Blueprints` (icone `Icons.Blueprint`). Action : `ExecuteVerifyBlueprints`.

La `FAutoConsoleCommand` `Rz.VerifyBlueprints` est enregistree au chargement statique du module (objet global), independamment du `StartupModule`.

**Runtime — Organize.** `ExecuteOrganizeGraph` localise l'editeur Blueprint actif via `FindActiveBlueprintEditor` (itere `UAssetEditorSubsystem::GetAllEditedAssets`, garde le premier `UBlueprint` dont `GetFocusedGraph()` est non nul), amene le graphe au premier plan, ouvre un `FScopedTransaction` ("Organize Graph"), puis appelle `FGraphOrganizer::OrganizeGraph`. Pipeline interne de l'organizer :
1. `CollectNodes` — construit la `TMap<UEdGraphNode*, FNodePlacement>` ; classe chaque noeud (comment / data / exec), capture taille via `GetNodeSize`/`EstimateNodeSize`, et collecte les predecesseurs de donnees (pins d'entree non-exec).
2. `GetRootNodes` — racines = events, `UK2Node_FunctionEntry`, input nodes, ou tout noeud exec sans exec entrant ; triees par `NodePosY`.
3. `PlaceExecutionNodes` — BFS par racine, assigne `Column` (profondeur) et `Lane` (la premiere sortie reste sur la lane du parent, les branches supplementaires prennent de nouvelles lanes).
4. `PlaceDataNodes` — `FindUltimateConsumer` (recursif) rattache chaque data node a son consommateur d'execution de plus petite colonne ; empilement (`StackIndex`) trie par `NodePosY`.
5. `CalculatePositions` — convertit colonnes/lanes en X/Y absolus (largeurs de colonne et hauteurs de lane mesurees, en tenant compte de la hauteur des piles de data nodes).
6. `ResolveCollisions` — jusqu'a 10 passes de separation verticale des AABB qui se chevauchent.
7. `ApplyPositions` — ecrit les positions via `Schema->SetNodePosition` (fallback `Node->Modify()` + `NodePosX/Y`), uniquement si la position change.
8. `HandleCommentNodes` — redimensionne chaque `UEdGraphNode_Comment` autour des noeuds qu'il contient deja (`GetNodesUnderComment`), avec padding.

Si des noeuds ou des commentaires ont change : `Graph->NotifyGraphChanged()` + `FBlueprintEditorUtils::MarkBlueprintAsModified`.

**Runtime — Verify.** `ExecuteVerifyBlueprints` ouvre un `SCustomDialog` contenant `SBlueprintVerifierDialog`. Sur "Verify" (et au moins un scope coche), appelle `FBlueprintVerifier::VerifyAllBlueprints` puis `WriteReportToFile`, et affiche une notification Slate (`FSlateNotificationManager`) avec icone succes/warning/erreur et, si le rapport a ete ecrit, un hyperlien "Open Report" (`LaunchFileInDefaultExternalApplication`). `VerifyAllBlueprints` :
- Force un scan Asset Registry complet, filtre les `UBlueprint` par les prefixes de `GetPathFilters()`.
- Affiche un `FScopedSlowTask` annulable ; pour chaque asset : `EnterProgressFrame`, charge (`StaticLoadObject`, flags no-warn/no-compile-on-load), compile via `CompileSingleBlueprint`, accumule les compteurs. Echec de chargement = 1 erreur synthetique. GC manuel toutes les 10 s.
- Retourne le `FBlueprintVerificationReport` agrege ; `WriteReportToFile` serialise un `.txt` horodate (`Report_%Y%m%d_%H%M%S.txt`) sous `GetReportDirectory()` = `Saved/BlueprintVerification`.

**Teardown.** `ShutdownModule` → `UnregisterMenus` : `UToolMenus::UnRegisterStartupCallback(this)` puis `RemoveSection` sur les deux menus etendus.

## 5. Replication / reseau

Sans objet. Le plugin est editeur-seul, ne contient aucune classe replicable, aucun `UPROPERTY` replique, aucun RPC `Server_`/`Client_`/`Multicast_`, aucun `GetLifetimeReplicatedProps`. Verifie par grep : zero occurrence.

## 6. Points d'integration

**Dependances (de `RzBlueprintTools.Build.cs`).**
- Public : `Core`, `CoreUObject`, `Engine`, `Slate`, `SlateCore`, `InputCore`.
- Prive : `UnrealEd`, `BlueprintGraph`, `Kismet`, `GraphEditor`, `EditorStyle`, `ToolMenus`, `Projects`, `EditorFramework`, `WorkspaceMenuStructure`, `ApplicationCore`, `ToolWidgets`, `AnimGraph`, `KismetCompiler`, `AssetRegistry`.

Toutes ces dependances sont editeur/Slate/Kismet — d'ou `Type: Editor`. Le plugin ne sera pas compile/charge dans un build de jeu (client ou dedicated server).

**Dependants dans QANGA.** Aucun. Les types publics (`FGraphOrganizer`, `FBlueprintVerifier`, etc.) ne sont reference nulle part ailleurs dans `G:/QANGA` (verifie par grep sur l'arbre complet hors plugin ; seules les correspondances sont les sources du plugin, ses propres `Docs/`, et `.gitignore`). C'est un outil autonome : le reste du projet ne lie pas ce module et n'a pas a le faire.

**Commande console.** `Rz.VerifyBlueprints [Game] [Plugins] [Engine]` — sans argument : scope `Game` seul. Aucun CVar de configuration ; les seuls reglages sont les trois booleens de scope (`FBlueprintVerifierSettings`), pilotes par les args console ou les cases du dialog.

**Sortie disque.** Rapports sous `Saved/BlueprintVerification/Report_<timestamp>.txt`. `Saved/` est ignore par git, donc les rapports ne sont pas versionnes. Le repertoire source du plugin est explicitement *suivi* (`!/Plugins/RzBlueprintTools/` dans `.gitignore`), seuls `Binaries/` et `Intermediate/` sont ignores.

## 7. Gotchas, invariants et pieges

- **L'organizer agit sur le graphe FOCALISE, pas sur l'asset selectionne.** `FindActiveBlueprintEditor` retourne le premier editeur Blueprint ouvert ayant un `GetFocusedGraph()` non nul. Avec plusieurs Blueprints ouverts, l'outil cible le premier itere, pas forcement celui que l'utilisateur regarde. Le bouton `Organize` est desactive (`CanExecuteOrganizeGraph`) s'il n'y a aucun graphe focalise.

- **Garde de schema par nom de classe, pas par type.** `OrganizeGraph` sort si le nom de classe du schema ne contient ni `K2` ni `Blueprint`. C'est une heuristique chaine — robuste pour les graphes K2 standards (`UEdGraphSchema_K2`), mais elle exclut volontairement Material/Niagara/PCG/Control Rig/AnimGraph. Ne pas attendre de layout sur ces graphes.

- **`SearchAllAssets(true)` est synchrone et bloquant.** Le verifier force un scan complet de l'Asset Registry au demarrage de chaque verification. Sur un projet planetaire comme QANGA, le premier scan peut etre long ; c'est attendu, pas un blocage.

- **La verification CHARGE et COMPILE chaque Blueprint en memoire.** `StaticLoadObject` + `FKismetEditorUtilities::CompileBlueprint` ont un cout reel ; le GC manuel toutes les 10 s borne la memoire mais le scope `Plugins`/`Engine` peut tirer des milliers d'assets — d'ou leur desactivation par defaut. Le `FScopedSlowTask` est annulable (`ShouldCancel`) : un rapport partiel reste produit a l'annulation.

- **`SkipGarbageCollection` a la compilation est volontaire.** La compilation passe `EBlueprintCompileOptions::SkipGarbageCollection` car le verifier gere son propre GC periodique ; ne pas "corriger" en reactivant le GC par compilation (cout multiplie).

- **Pas de transaction pour le verifier ; transaction unique pour l'organizer.** L'organizer enveloppe son travail dans un `FScopedTransaction` ("Organize Graph") cote module — donc Undo restaure le layout. Le verifier ne modifie pas d'assets de facon persistante (compile en memoire) et n'a pas de transaction.

- **`SetNodePosition` est la voie privilegiee ; le fallback direct existe.** `ApplyPositions`/`HandleCommentNodes` ecrivent via `Schema->SetNodePosition` quand un schema est present, sinon `Node->Modify()` + ecriture directe de `NodePosX/Y`. En pratique le schema est toujours present (garde en amont), mais le fallback couvre le cas degenere.

- **L'organizer ne touche jamais la topologie.** Aucun pin n'est cree/detruit/reconnecte ; seuls position et taille de commentaire changent. C'est l'invariant central : un graphe reorganise est fonctionnellement identique a l'original. Toujours review l'asset modifie sous source control avant commit (le layout n'est qu'un changement cosmetique mais il *salit* le diff de l'asset).

- **`ResolveCollisions` est borne a 10 passes.** Sur des graphes pathologiquement denses, des chevauchements residuels peuvent subsister apres 10 passes. Ce n'est pas une erreur : la boucle privilegie la terminaison sur la perfection.

## 8. Fichiers et emplacements

| Fichier | Role |
| --- | --- |
| `G:/QANGA/Plugins/RzBlueprintTools/RzBlueprintTools.uplugin` | Descripteur du plugin : module editeur unique, `PostEngineInit`, `EnabledByDefault`, pas de contenu. |
| `G:/QANGA/Plugins/RzBlueprintTools/Source/RzBlueprintTools/RzBlueprintTools.Build.cs` | Regles de build : dependances editeur/Slate/Kismet/AssetRegistry. |
| `Source/RzBlueprintTools/Public/RzBlueprintToolsModule.h` | Interface du module `FRzBlueprintToolsModule`. |
| `Source/RzBlueprintTools/Private/RzBlueprintToolsModule.cpp` | Startup/shutdown, enregistrement toolbar+menu (`UToolMenus`), `FAutoConsoleCommand` `Rz.VerifyBlueprints`, `SBlueprintVerifierDialog`, actions `ExecuteOrganizeGraph`/`ExecuteVerifyBlueprints`, `FindActiveBlueprintEditor`. |
| `Source/RzBlueprintTools/Public/GraphOrganizer.h` | `FGraphOrganizer` + `FNodePlacement` ; constantes de layout (`HorizontalSpacing`, `VerticalSpacing`, `DataNodeOffset`, etc.). |
| `Source/RzBlueprintTools/Private/GraphOrganizer.cpp` | Implementation de l'algorithme de layout (collect → roots → exec BFS → data stacking → positions → collisions → apply → comments). |
| `Source/RzBlueprintTools/Public/BlueprintVerifier.h` | `FBlueprintVerifier` + structs `FBlueprintCompileResult`/`FBlueprintVerificationReport`/`FBlueprintVerifierSettings`. |
| `Source/RzBlueprintTools/Private/BlueprintVerifier.cpp` | Scan Asset Registry, filtrage par scope, charge+compile via Kismet, agregation, ecriture du rapport `.txt`. |
| `Source/RzBlueprintTools/../Docs/` (`README.md`, `QUICKSTART.md`, `TOOLS.md`, `ADVANCED.md`, `THIRD_PARTY.md`) | Documentation utilisateur du plugin (coherente avec le code). |
| `Saved/BlueprintVerification/Report_<timestamp>.txt` | Sortie runtime du verifier (non versionnee). |
