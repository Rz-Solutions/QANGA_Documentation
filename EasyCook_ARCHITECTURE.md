# EasyCook — Architecture

Outillage éditeur de correction de cook (par Rz Software). EasyCook trouve les références d'assets que le cooker d'Unreal ne sait pas tracer (chemins en dur dans du C++, pins de Blueprint, defaults reflétés, strings de data-content), génère une *seed* de cook, et corrige en amont trois familles de bugs « marche en PIE, cassé en build packagé » (imports editor-only survivants, héritage d'émetteurs Niagara strippé, shadermaps GPU Niagara périmées). Plugin Win64 uniquement, ciblant UE 5.7.0.

---

## 1. But d'usage

Le cooker d'UE ne package que les assets qu'il atteint par fermeture de références *à la sauce moteur* : refs dures, refs soft taggées « Game », et la liste de maps. Tout ce qui est *résolu au runtime par une string* lui est invisible. Sur QANGA — un open-world sci-fi multijoueur à planètes entières — cela couvre une grande surface :

- chemins d'assets en dur dans le C++ (`LoadObject("/Game/...")`, `ConstructorHelpers::FObjectFinder`, `FString::Printf` construisant un chemin) ;
- pins de Blueprint contenant un chemin d'asset en string ;
- propriétés `Asset` de composants imbriqués dans le SimpleConstructionScript d'un BP (ex. `NiagaraComponent.Asset = "/path/NS"`) que ni le C++ ni le graphe BP ne montrent ;
- chemins stockés en plain-string dans des DataTable / DataAsset / CDO de BP (son, dialogue, icônes, VFX marketplace référencés indirectement).

Sans correction, ces assets manquent du build packagé : VFX absents, sons muets, contenu data-driven vide. EasyCook résout ce problème en deux temps : un **scan éditeur** qui découvre statiquement ces références cachées et les persiste dans un data asset (`UEasyCookSeed`), puis un **hook de cook** (`FEasyCookModule::OnModifyCook`) qui, à chaque cook, lit la seed et ajoute force chaque package au set du cooker.

En plus du problème de découverte, EasyCook embarque trois **passes de réparation** ciblant des bugs de cook propres à la migration 5.3→5.7 de QANGA :

1. **Package repair** — un import dont la cible est editor-only mais marqué `bUsedInGame=true` survit au strip du cook et fait échouer la vérification du graphe EDL (« Content is missing from cook »).
2. **Niagara GPU shader refresh** — un `NiagaraSystem` à émetteur GPU jamais re-sauvé depuis la migration moteur cook sans shadermap de compute, ce qui rend `UNiagaraSystem::IsValid()` faux et **désactive silencieusement tout le système** au runtime (toutes les émetteurs, CPU comprises). Marche en PIE, noir en build, zéro log.
3. **Niagara inliner** (hors `RunAll`, manuel) — le cook strippe les métadonnées d'héritage Niagara ; les émetteurs enfants perdent leur parent externe et deviennent des stubs vides en build.

---

## 2. Vue d'ensemble / décisions d'architecture

EasyCook est divisé en **deux modules** :

- **`EasyCook`** (Runtime, `PostConfigInit`) — minimal. Sa seule responsabilité réelle est le hook de cook `OnModifyCook` (gardé `WITH_EDITOR`) plus les types partagés (`UEasyCookSeed`, `FEasyCookEntry`, enums). Il est de type Runtime, et non Editor, pour que le délégué `GetModifyCookDelegate()` soit enregistré dans le contexte du cook commandlet.
- **`EasyCookEditor`** (Editor, `PostEngineInit`) — tout l'outillage : subsystem, scanners, passes de réparation, writers, worker isolé, widget.

Décisions clés :

- **La seed est l'unique source de vérité du cook, pas un .ini.** `UEasyCookSeed::IsEditorOnly()` retourne `true` : la seed elle-même n'est jamais cookée (sinon le cooker suivrait chaque `FSoftObjectPath` de `Entries` et tirerait tout le junk editor/dev dans le build). Le cook est piloté par le délégué `OnModifyCook`, qui charge la seed en éditeur pour lire ses entries. Le fichier `Config/EasyCookGenerated.ini` est **purement informationnel** — un snapshot lisible/diffable, jamais lu par le cooker (voir `FEasyCookIniGenerator`).

- **Le scan de data-content est isolé dans un process enfant expendable.** Charger un package corrompu fait un `Fatal` irrécupérable dans `FLinkerLoad`, impossible à try-catcher. `FEasyCookScanWorker` lance donc un commandlet headless (`UEasyCookScanCommandlet`, `-run=EasyCookEditor.EasyCookScan`) qui checkpoint l'index *avant* chaque load ; si le worker meurt, le parent lit le checkpoint, note l'asset comme illisible, et relance le worker à l'index suivant. Les autres scanners (C++, Blueprint, Reflection) tournent in-process.

- **Le filtre de force-cook est délibérément asymétrique.** Côté `OnModifyCook`, plusieurs niveaux de filtrage écartent les packages dangereux ou redondants : contenu moteur / `/Script/` / `/Temp/`, plugins editor-only, arbres de maps (World Partition / HLOD générés), orphelins non atteignables, et packages avec imports editor-only survivants. Une **exemption par classe** garde toujours les `NiagaraEmitter` parents (le cooker les classe editor-only à cause de la résolution d'héritage, donc il ne les atteint jamais nativement — c'est précisément pourquoi le scanner les ajoute).

- **Découverte 100 % statique, pas de log-scraping ni de listes manuelles d'assets.** Les passes Niagara s'auto-découvrent via le tag d'Asset Registry `HasGPUEmitter` ; la réparation de package lit la table d'imports on-disk via `FPackageReader`. Aucune dépendance à un log de cook précédent.

- **Aucune passe ne resauvegarde de package depuis l'intérieur d'un cook.** `UPackage::SavePackage` déclenche des PreSave qui assertent en contexte cook (ex. `ULandscapeHeightfieldCollisionComponent::PreSave`). Les passes destructives (`FEasyCookPackageRepair`, `FEasyCookNiagaraInliner`, `FEasyCookNiagaraGpuShaderRefresh`) sont donc **editor-mode only** ; `OnModifyCook` ne fait que consommer le résultat.

État : implémenté et utilisé sur QANGA (seed live `/Game/EasyCook/DA_EasyCookSeed_Qanga`, ~77 000 packages dans le dernier rapport généré). `InlineNiagaraParents` existe en méthode + commande console mais n'est **pas** câblé dans `RunAll` (réparation manuelle ponctuelle). Le délégué d'inline est laissé à l'opérateur car il modifie irréversiblement les .uasset sources et n'est requis que pour les VFX à héritage externe.

---

## 3. Carte des composants

| Élément | Type | Rôle |
| --- | --- | --- |
| `FEasyCookModule` | IModuleInterface (module `EasyCook`) | Enregistre `OnModifyCook` sur `FGameDelegates::GetModifyCookDelegate()` ; pilote le force-cook au cook time. |
| `UEasyCookSeed` | UCLASS : UDataAsset | Data asset persistant. `Entries`, `AlwaysCookDirectories`, stats. `IsEditorOnly()==true`. |
| `FEasyCookEntry` | USTRUCT | Une référence découverte : `AssetPath`, `Kind`, `Source`, origine (fichier/ligne/contexte), `bIncludeInCook`. Hashable, comparée sur (AssetPath, Kind). |
| `EEasyCookRefKind` | UENUM | HardObject / SoftObject / HardClass / SoftClass / DirectoryAlways. |
| `EEasyCookSource` | UENUM | CppLiteral / BlueprintNode / CdoDefault / Manual / PieRecording / DataContent. Détermine si l'entry est force-cookée inconditionnellement. |
| `LogEasyCook` | catégorie de log | Toute la journalisation du plugin. |
| `FEasyCookEditorModule` | IModuleInterface (module `EasyCookEditor`) | Module éditeur ; Startup/Shutdown vides (le travail vit dans le subsystem et les commandes console). |
| `UEasyCookSubsystem` | UCLASS : UEditorSubsystem | Façade BlueprintCallable : scans, repairs, `RunAll`, save/validate, commandes console `EasyCook.*`. Détient `ScratchEntries` / `ScratchDirectories`. |
| `UEasyCookWidgetBase` | UCLASS : UEditorUtilityWidget | Base de l'EUW ; auto-bind `BtnClear`/`BtnRun` par nom dans `NativeConstruct`, met à jour `StatsText`. |
| `FEasyCookCppScanner` | classe statique | Scan des sources C++ (projet + plugins) pour chemins d'assets en string littérale. |
| `FEasyCookBlueprintScanner` | classe statique | Scan des pins de tous les `UBlueprint` montés pour defaults ressemblant à des chemins. |
| `FEasyCookReflectionScanner` | classe statique | Parcours par réflexion de chaque CDO de `UClass` chargée (FObject/FSoftObject/FClass/FSoftClass, + templates SCS). |
| `FEasyCookDataContentScanner` | classe statique | Trouve chemins stockés en plain-string dans DataTable / DataAsset / CDO de BP. Découpé en `GatherAssetPaths` (sûr) + `ScanSingleAsset` (charge, donc worker-only). |
| `FEasyCookDependencyExpander` | classe statique | Étend chaque entry à sa fermeture de cook (refs dures + soft de l'Asset Registry), purge moteur/Script/plugin-éditeur. |
| `FEasyCookPathHeuristics` | classe statique | Heuristiques partagées : reconnaître un chemin, normaliser, valider via AR, extraire le préfixe statique d'un chemin dynamique. |
| `FEasyCookAssetFilter` | classe statique | Filtre d'émission : exclut moteur/plugins-moteur, `+DirectoriesToNeverCook`, patterns dev, refs soft cassées. |
| `FEasyCookScanWorker` | classe statique | Orchestrateur parent du scan data-content crash-isolé ; relance le worker autour des assets corrompus. |
| `UEasyCookScanCommandlet` | UCLASS : UCommandlet | Worker headless (`-run=EasyCookScan`) ; checkpoint avant chaque load, hard-exit en fin. |
| `FEasyCookManifestWriter` | classe statique | Dedupe + tri déterministe des entries, écrit le `UEasyCookSeed.uasset`. |
| `FEasyCookIniGenerator` | classe statique | Écrit le rapport informationnel `Config/EasyCookGenerated.ini` ; strippe le bloc-marqueur legacy de `DefaultGame.ini`. |
| `FEasyCookPackageRepair` (+ `FEasyCookPackageRepairResult`) | classe statique | Patch binaire des imports editor-only `bUsedInGame=true` (clear du bit dans la section AssetRegistryDependencyData). |
| `FEasyCookNiagaraGpuShaderRefresh` (+ `...Result`) | classe statique | Recompile + resauve les `NiagaraSystem` GPU à compile-ids périmés. |
| `FEasyCookNiagaraInliner` (+ `...Result`) | classe statique | Détache les `NiagaraSystem` de leurs `NiagaraEmitter` parents externes (RemoveParent). |
| `EUW_EasyCook` | asset (Content/) | Editor Utility Widget reposant sur `UEasyCookWidgetBase`. |
| `DA_EasyCookSeed_Qanga` | asset (`/Game/EasyCook/`) | La seed live de QANGA. |

---

## 4. Flux de données et cycle de vie

### Init / registration

- `FEasyCookModule::StartupModule` (Runtime, `PostConfigInit`) : sous `WITH_EDITOR`, `AddRaw` de `OnModifyCook` sur `FGameDelegates::Get().GetModifyCookDelegate()`. `ShutdownModule` retire le délégué.
- `FEasyCookEditorModule` (Editor, `PostEngineInit`) : Startup/Shutdown vides.
- `UEasyCookSubsystem::Initialize` : `RegisterConsoleCommands()` enregistre 8 commandes `EasyCook.*` (`Scan`, `SaveSeed`, `GenerateIni`, `ClearScratch`, `ExpandDeps`, `Repair`, `InlineNiagaraParents`, `RefreshNiagaraGpuShaders`). `Deinitialize` les désenregistre.

### Authoring : le pipeline `RunAll` (déclenché par `EUW_EasyCook` → `BtnRun`, ou la console)

`UEasyCookSubsystem::RunAll` exécute, dans l'ordre :

1. **`ClearScratch`** — vide `ScratchEntries` et `ScratchDirectories`.
2. **`FEasyCookPackageRepair::RunAll`** — *avant* le scan, pour que l'état disque soit propre quand les scanners suivent les refs dures (résultats stables entre reruns).
3. **`FEasyCookNiagaraGpuShaderRefresh::RunAll`** — *avant* le scan, pour qu'aucun système GPU périmé ne reste dans le set.
4. **`ScanAll`** — séquence : `ScanCpp` → `ScanBlueprints` → `ScanReflection` → `ScanDataContent` (worker isolé) → `ExpandDependencies`. Forcé en mode unattended (`GIsRunningUnattendedScript=true`, restauré sur scope-exit) pour router les erreurs de load vers le log au lieu d'une fenêtre modale (qui peut crasher l'éditeur près de la limite de handles USER Windows).
5. **`SaveSeed("")`** — `FEasyCookManifestWriter::Save` dedupe/trie, écrit `/Game/EasyCook/DA_EasyCookSeed_<ProjectName>`, puis re-réinjecte le résultat dédupliqué dans le scratch.
6. **`GenerateIni`** — `FEasyCookIniGenerator` écrit le rapport `Config/EasyCookGenerated.ini`.

`RunAll` retourne `bSave && bIni`. `RefreshNiagaraGpuShaders` est dans `RunAll` ; `InlineNiagaraParents` ne l'est pas (manuel).

### Détail du scan data-content (crash-isolé)

`FEasyCookScanWorker::RunScan` : `GatherAssetPaths` (requête AR pure) → écrit `scan_assets.txt` sous `Intermediate/EasyCook/` → boucle de relance de `UEasyCookScanCommandlet` (`CreateProc`, hidden, `-nullrhi`). Chaque worker checkpoint l'index dans `scan_progress.txt` avant le load, append les résultats dans `scan_results.txt`, écrit `scan_done.txt` en fin et hard-exit (`FPlatformMisc::RequestExit(true)`). Si pas de `scan_done.txt`, le parent lit le checkpoint, saute l'asset corrompu, relance. Borne de sécurité `MaxRuns = AssetPaths.Num() + 2`.

### Runtime de cook : `OnModifyCook`

À chaque cook, le délégué `OnModifyCook` reçoit `PackagesToCook` / `PackagesToNeverCook` :

1. `AR.WaitForCompletion()`, construit les préfixes de mount exclus (plugins moteur + plugins éditeur-only).
2. Pré-calcule `MapTreeFolders` (dossiers contenant un `UWorld`) — tout package sous un arbre de map est skip (cooké nativement ; force-cooker casserait IoStore).
3. Pré-calcule `NaturalCookClosure` : BFS depuis `PackagesToCook` *à l'instant de l'appel* (maps de `+MapsToCook`, de `-Map=` en ligne de commande, PrimaryAssets, UFS) en suivant refs Game (Hard + Soft). Sert à dropper les orphelins.
4. Pour chaque `UEasyCookSeed` : `AddUnique` la seed à `PackagesToNeverCook` ; puis pour chaque entry incluse, une cascade de filtres (`IsPackageCookExcluded`, `IsInsideMapTree`, `ShouldForceCookEntry`, `ShouldExcludeFromForceCook`) avant `PackagesToCook.AddUnique`.
5. Les entries `CppLiteral`/`BlueprintNode`/`Manual`/`PieRecording` sont force-cookées **inconditionnellement** (vrais gaps de string) ; les entries `CdoDefault`/`DataContent` ne le sont que si dans `NaturalCookClosure` (sinon orphelin droppé).
6. `AlwaysCookDirectories` : force-cook récursif de chaque dossier (refus si <3 segments = trop large ; jamais de map). Log de synthèse final.

### Teardown

`OnModifyCook` est sans état persistant (caches locaux à l'appel). `ShutdownModule` retire le délégué. Le subsystem désenregistre ses commandes console.

---

## 5. Réplication / réseau

Sans objet. EasyCook est un outil d'éditeur et un hook de cook ; aucune réplication, aucun RPC, aucun `GetLifetimeReplicatedProps`. Le module runtime existe seulement pour enregistrer un délégué de cook dans le bon contexte de chargement.

---

## 6. Points d'intégration

### Dépendances (consommées par EasyCook)

- Module `EasyCook` : `Core`, `Projects` (public) ; `CoreUObject`, `Engine`, `DeveloperSettings`, `AssetRegistry` (privé). Inclut `Source/Runtime/AssetRegistry/Internal` pour `FPackageReader` (lecture de la table d'imports on-disk dans `OnModifyCook`).
- Module `EasyCookEditor` : ajoute `UnrealEd`, `Slate`/`SlateCore`, `EditorStyle`, `EditorFramework`, `EditorSubsystem`, `ToolMenus`, `UMG`/`UMGEditor`, `Blutility`, `AssetTools`, `BlueprintGraph`, `Kismet`/`KismetCompiler`, `PropertyEditor`, et **`Niagara`** (pour `FEasyCookNiagaraInliner` / `FEasyCookNiagaraGpuShaderRefresh`).
- Plugin requis : `Niagara` (déclaré dans `.uplugin`).

### Dépendants (ce qui dépend d'EasyCook)

Aucun. EasyCook n'expose aucun type au reste de QANGA : un grep de ses types publics sur `G:/QANGA/Source` et `G:/QANGA/Plugins` (hors plugin lui-même) ne renvoie rien. C'est un outil terminal — il agit *sur* le contenu du projet et *sur* le cook, mais aucun code QANGA ne le `#include` ni ne référence `UEasyCookSeed`. L'intégration se fait par effet (la seed pilote le cook ; les passes réparent les .uasset).

### Settings / fichiers de config

- `Config/EasyCookGenerated.ini` — rapport informationnel généré, non lu par le cooker. Présent et à jour sur QANGA (77 111 packages au dernier run).
- Pas de `UDeveloperSettings`. `DeveloperSettings` est lié mais aucune classe de settings n'est définie ; les chemins (seed par défaut, ini relatif) sont en constantes / propriétés de la seed.
- `FEasyCookAssetFilter` lit `+DirectoriesToNeverCook` de `DefaultGame.ini` du projet pour son filtre d'exclusion.

### Commandes console (toutes éditeur)

`EasyCook.Scan`, `EasyCook.SaveSeed [path]`, `EasyCook.GenerateIni`, `EasyCook.ClearScratch`, `EasyCook.ExpandDeps`, `EasyCook.Repair`, `EasyCook.InlineNiagaraParents`, `EasyCook.RefreshNiagaraGpuShaders`.

---

## 7. Gotchas, invariants et pièges

- **La seed DOIT rester editor-only.** `UEasyCookSeed::IsEditorOnly()==true` est ce qui empêche le cooker de suivre ses `FSoftObjectPath` et de tirer tout le junk listé. `OnModifyCook` la `AddUnique` aussi à `PackagesToNeverCook` en ceinture-et-bretelles. Ne jamais retirer ce flag.

- **Les passes destructives ne tournent JAMAIS sous cook.** `FEasyCookNiagaraGpuShaderRefresh` refuse explicitement un contexte commandlet non-éditeur (`IsRunningCommandlet() && !GIsEditor`). `UPackage::SavePackage` déclenche des PreSave (`UWorld → FinishAllCompilation → SetMaterialUsage → FMessageLog::Open → fenêtre Slate`) qui peuvent crasher l'éditeur près de la limite de 10 000 handles USER Windows. C'est pourquoi `OnModifyCook` ne resauve rien et délègue la réparation à la phase d'authoring.

- **`FEasyCookPackageRepair` est un patch binaire, pas un resave.** Il clear le bit `bUsedInGame` directement dans la section `AssetRegistryDependencyData` du .uasset, sans charger d'UObject ni appeler `SavePackage`. Raison documentée dans le header : un incident du 24/05/26 a corrompu 768 .uasset en 90 s via `UPROPERTY+SavePackage` à l'échelle de 1700 packages (récupéré par restore Backblaze). Les packages où le classifier d'UPROPERTY override le bit (UPROPERTY non-`WITH_EDITORONLY_DATA` pointant vers `/Engine/Editor*`) sont **non-réparables** par cette passe — ils sont loggés « still broken » et nécessitent une édition BP manuelle ou une entrée `+PackagesToNeverCook`.

- **L'AR cache les imports editor-only.** `IAssetRegistry::GetDependencies` filtre les imports via `ImportsUsedInGame` au moment de `WritePackageData`, donc il ne voit pas les refs editor-only survivantes. Les deux passes qui détectent le pattern cook-breaking (`OnModifyCook::HasUsedInGameEditorOnlyImport` et `FEasyCookPackageRepair`) lisent **la table d'imports brute** via `FPackageReader::ReadLinkerObjects` — d'où l'include du dossier `Internal/` de l'AssetRegistry dans les deux Build.cs.

- **Exemption Niagara dans `ShouldExcludeFromForceCook`.** Les `NiagaraEmitter` parents externes sont exemptés *par classe* du filtre editor-only. Le cooker les classe editor-only (résolution d'héritage = editor-time) et ne les atteint donc jamais nativement — c'est exactement pourquoi le scanner les ajoute. Sans cette exemption, les VFX cyborg / jetpack / à émetteur externe se désactivent silencieusement en build.

- **Le scan data-content charge des assets et PEUT crasher fatalement.** C'est pour ça qu'il vit dans un process enfant (`UEasyCookScanCommandlet`). Ne jamais appeler `FEasyCookDataContentScanner::ScanSingleAsset` in-process ; seul `GatherAssetPaths` (requête AR) est sûr. Le worker checkpoint *avant* le load et hard-exit en fin (un worker propre ne doit pas laisser de crash report parasite via le teardown de plugins tiers).

- **`FEasyCookPathHeuristics::IsAlwaysCookableDirectory` et `ExtractStaticPrefix` ont des contraintes de thread.** `ExtractStaticPrefix` est de la string pure (n'importe quel thread) ; `IsAlwaysCookableDirectory` appelle `IAssetRegistry::GetAssetsByPath` qui asserte le thread appelant → **game thread only**, jamais dans un `ParallelFor`.

- **`AlwaysCookDirectories` exige ≥3 segments.** `OnModifyCook` refuse de force-cooker un dossier top-level (ex. `/Game/Maps`) : cooker un arbre entier tire les maps + leurs packages World Partition/HLOD générés et casse le packaging IoStore. Idem, aucune `UWorld` n'est jamais force-cookée — les maps passent par la liste de maps du cooker.

- **`NaturalCookClosure` est seedé à l'instant de l'appel d'`OnModifyCook`.** Lire seulement `+MapsToCook` du .ini raterait les maps passées en `-Map=` par le Project Launcher. Le BFS part de `PackagesToCook` tel qu'UE l'a déjà rempli (request set initial), d'où la dépendance au timing du délégué.

- **Détection GPU Niagara = `HasGPUEmitter` (tag AR) + `AreScriptAndSourceSynchronized()`.** Le tag, lisible sans charger l'asset, sélectionne les candidats ; un script GPU `enabled` désynchronisé = compile-id périmé = cookerait sans shadermap. `NeedsGpuShaderRefresh` est un sur-ensemble sûr (ne rate jamais un système cassé, ne touche jamais un système déjà à jour). La passe ne re-sauve QUE les périmés, pour ne pas churner les bytes sources. Un système toujours invalide après `RequestCompile(true)` est un graphe corrompu → loggé `SystemsFailedToCompile`, non sauvé.

- **`InlineNiagaraParents` est destructif et hors `RunAll`.** Il fait l'équivalent de « Remove Parent Emitter » (zéro `VersionedParent` + `VersionedParentAtLastMerge`) et resauve le .uasset. Réservé à une réparation manuelle ponctuelle des VFX à héritage externe ; à ne pas confondre avec le GPU shader refresh (problèmes orthogonaux).

- **L'EUW auto-bind par NOM.** `UEasyCookWidgetBase::NativeConstruct` cherche `BtnClear`/`BtnRun` (et `StatsText`) dans l'arbre de widgets du BP par nom. Renommer ces widgets dans `EUW_EasyCook` casse silencieusement le binding (log `Verbose`).

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
| --- | --- |
| `EasyCook.uplugin` | Déclare les 2 modules (Runtime `PostConfigInit` + Editor `PostEngineInit`), Win64-only, plugin requis `Niagara`. |
| `Source/EasyCook/EasyCook.Build.cs` | Deps runtime + include `Internal/` de l'AssetRegistry (`FPackageReader`). |
| `Source/EasyCook/Public/EasyCookModule.h` / `Private/EasyCookModule.cpp` | `FEasyCookModule` + tout le hook de cook `OnModifyCook` (filtres, closures, force-cook). |
| `Source/EasyCook/Public/EasyCookTypes.h` | `FEasyCookEntry`, `EEasyCookRefKind`, `EEasyCookSource`, `LogEasyCook`. |
| `Source/EasyCook/Public/EasyCookSeed.h` / `Private/EasyCookSeed.cpp` | `UEasyCookSeed` (data asset, `IsEditorOnly()`). |
| `Source/EasyCookEditor/EasyCookEditor.Build.cs` | Deps éditeur (UnrealEd, UMG, Niagara, …) + include `Internal/` de l'AssetRegistry. |
| `Source/EasyCookEditor/Private/EasyCookEditorModule.cpp` | `FEasyCookEditorModule` (Startup/Shutdown vides). |
| `Source/EasyCookEditor/Public/EasyCookSubsystem.h` / `Private/...cpp` | `UEasyCookSubsystem` : façade BP, `RunAll`, commandes console `EasyCook.*`. |
| `Source/EasyCookEditor/Public/EasyCookWidgetBase.h` / `Private/...cpp` | `UEasyCookWidgetBase` : auto-bind des boutons de l'EUW. |
| `Source/EasyCookEditor/Public/Scanners/EasyCookCppScanner.h` / `Private/...cpp` | Scan des sources C++. |
| `.../Scanners/EasyCookBlueprintScanner.{h,cpp}` | Scan des pins BP. |
| `.../Scanners/EasyCookReflectionScanner.{h,cpp}` | Scan des CDO par réflexion (+ templates SCS). |
| `.../Scanners/EasyCookDataContentScanner.{h,cpp}` | Scan des strings DataTable/DataAsset/CDO (split safe/worker). |
| `.../Scanners/EasyCookDependencyExpander.{h,cpp}` | Fermeture de dépendances AR. |
| `.../Scanners/EasyCookPathHeuristics.{h,cpp}` | Heuristiques de chemins (validation, préfixe statique). |
| `Source/EasyCookEditor/Public/EasyCookAssetFilter.{h,cpp}` | Filtre d'exclusion d'émission. |
| `Source/EasyCookEditor/Private/EasyCookScanWorker.{h,cpp}` | Orchestrateur parent du scan crash-isolé. |
| `Source/EasyCookEditor/Private/EasyCookScanCommandlet.{h,cpp}` | `UEasyCookScanCommandlet` (worker headless). |
| `Source/EasyCookEditor/Public/EasyCookManifestWriter.{h,cpp}` | Dedupe/tri + écriture du `UEasyCookSeed.uasset`. |
| `Source/EasyCookEditor/Public/EasyCookIniGenerator.{h,cpp}` | Rapport `EasyCookGenerated.ini` + strip du bloc legacy. |
| `Source/EasyCookEditor/Public/EasyCookPackageRepair.{h,cpp}` | Patch binaire des imports editor-only `bUsedInGame`. |
| `Source/EasyCookEditor/Public/EasyCookNiagaraGpuShaderRefresh.{h,cpp}` | Recompile + resave des systèmes GPU Niagara périmés. |
| `Source/EasyCookEditor/Public/EasyCookNiagaraInliner.{h,cpp}` | Détache les émetteurs Niagara de leurs parents externes (manuel). |
| `Content/EUW_EasyCook.uasset` | Editor Utility Widget (basé sur `UEasyCookWidgetBase`). |
| `Config/EasyCookGenerated.ini` (racine projet) | Rapport informationnel généré (non lu par le cooker). |
| `/Game/EasyCook/DA_EasyCookSeed_Qanga.uasset` (Content projet) | Seed live de QANGA. |
