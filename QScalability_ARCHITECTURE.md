# QScalability — Architecture

> Plugin runtime (auteur : Iolacorp) : un **framework de réglages / qualité data-driven**. Chaque réglage utilisateur (qualité graphique, etc.) est un **data asset** décrivant son type (switch/int/float/bool), son affichage, ses presets et **comment il s'applique** (commande console, bundle de commandes, ou objet exécuteur custom). Un subsystem GameInstance charge les réglages, **benchmarke le hardware** (score CPU/GPU) pour auto-choisir un preset, applique, notifie et **persiste** dans un SaveGame.

---

## 1. But d'usage

QANGA a besoin d'un menu d'options robuste : exposer des réglages de qualité hétérogènes, les appliquer correctement (souvent via des CVars/commandes console), gérer des **presets** (Bas/Moyen/Élevé/Custom) avec dépendances, **détecter le matériel** pour proposer un défaut sensé, et **sauvegarder** le choix du joueur. Plutôt que de coder en dur chaque option, `QScalability` rend tout cela **data-driven** : un designer crée un data asset par réglage, et le subsystem orchestre chargement, benchmark, application et persistance.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Un réglage = un data asset.** `UQScalability_DataAsset` décrit entièrement un réglage : affichage (`Settings_Name`, `Settings_Description`, `Interactable`), identité (`Settings_Name_ID`), comportement (`Enabled`, `Display_Settings`, `Is_Global_Preset`, `Depends_On_preset`, `Call_Settings_OnSet`), **type** (`EQScalability_Type` ∈ {Switch, Int, Float, Bool}) avec sa data typée (`FQScalability_Switch_Data`/`Int`/`Float`/`Bool`), et ses **presets** (`Preset_Data`).
- **Trois façons d'appliquer un réglage.** Via `EQScalability_Data_ExecMode` : **commande console** (`Switch_Exec_CommandLink`, `Int_Exec_CommandLink`…), **bundle de commandes** (`UQScalability_Command_DataAsset` = `TMap<FString,FString>` de commandes, `Switch_CommandLine_DataAsset`), ou **objet exécuteur custom** (`UQScalability_Object` via `Custom_AutoSet_Object`/`Custom_Init_Object`/`Custom_Executor_Object`) pour la logique qui ne tient pas en une CVar.
- **Exécuteur custom scriptable.** `UQScalability_Object` (UObject Blueprintable) expose `QScalability_AutoSet` / `QScalability_Init` / `QScalability_Update` (BP native events) — un réglage peut donc déclencher du code/BP arbitraire à l'application.
- **Subsystem orchestrateur.** `UQScalability_GI_SubSystem` (GameInstanceSubsystem) charge les data assets listés dans les settings, indexe par `Settings_Name_ID` (`Settings_Data`), exécute un **benchmark** (CPU/GPU), applique les réglages, et persiste.
- **Benchmark hardware → auto-set.** Au premier lancement (`RunBenchmark_OnFirst`), `Run_Benchmark` échantillonne et calcule `CPU_Benchmark_Score` / `GPU_Benchmark_Score` ; `AutoSet_Settings_*` choisit la valeur de chaque réglage selon ces scores. Les multiplicateurs et l'échantillonnage sont configurables.
- **Presets avec dépendances.** Un réglage `Is_Global_Preset` pilote d'autres réglages ; modifier un réglage individuel listé dans `Depends_On_preset` **bascule le preset en « custom »**. `Call_Settings_OnSet` chaîne l'application d'autres réglages.
- **Persistance dédiée.** `UQScalability_SaveGame` stocke les valeurs (`TMap<FName, FQScalability_Data>`) + les scores de benchmark.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQScalability_GI_SubSystem` | `UGameInstanceSubsystem` | Orchestrateur : charge, benchmarke, applique, notifie, persiste |
| `UQScalability_DataAsset` | `UDataAsset` (`Blueprintable`) | Définition complète d'un réglage (type, display, presets, exécuteur) |
| `UQScalability_Command_DataAsset` | `UDataAsset` (`Blueprintable`) | Bundle nommé de commandes console (`TMap<FString,FString>`) |
| `UQScalability_Object` | `UObject` (`Blueprintable`) | Exécuteur custom (`AutoSet`/`Init`/`Update`) pour réglages à logique |
| `UQScalability_SaveGame` | `USaveGame` | Valeurs persistées + scores de benchmark |
| `UQScalability_Settings` | `UDeveloperSettings` (`config=Game`) | Liste des data assets à charger + config benchmark |
| `FQScalability_Data` | `USTRUCT` | Valeur typée d'un réglage |
| `FQScalability_Switch_Data` / `Int_Data` / `Float_Data` / `Bool_Data` | `USTRUCT` | Data par type (bornes, options…) |
| `FQScalability_Preset_Data` | `USTRUCT` | Une entrée de preset |
| `EQScalability_Type` | `UENUM` | `Switch` / `Int` / `Float` / `Bool` |
| `EQScalability_Data_ExecMode` | `UENUM` | Mode d'application (`Command` / DataAsset / custom) |

**Délégué** : `ScalabilityChange(FName Scalability, FQScalability_Data Value)`.

---

## 4. Flux de données et cycle de vie

1. **Init** — `UQScalability_GI_SubSystem::Initialize` → `DelayedStart` → `LoadSettings` : charge les `UQScalability_DataAsset` listés dans `UQScalability_Settings.Settings` (`Load_Settings_DataAsset`), les indexe par `Settings_Name_ID`.
2. **Benchmark (1er lancement)** — si `RunBenchmark_OnFirst` et pas de save, `Run_Benchmark` calcule `CPU_Benchmark_Score`/`GPU_Benchmark_Score` (échantillonnage `Benchmark_Sample`, multiplicateurs).
3. **Auto-set / chargement** — soit `AutoSet_Settings_*` dérive les valeurs des scores de benchmark, soit `Load_SaveGame` restaure le choix du joueur (`Load_Settings`). Sinon `Load_Default_Settings`.
4. **Application** — pour chaque réglage, `Exec_Value_Settings` applique selon le mode : `QScalability_ExecCommandConsole` (CVar/commande), `QScalability_Exec_CommandLine_DataAsset` (bundle), ou l'objet exécuteur custom (`QScalability_Update`).
5. **Changement runtime** — `QScalability_SetSettingValue(Setting, Value, Callback, FromPreset, NoCall)` met à jour la valeur, applique, déclenche `Call_Settings_OnSet` (chaînage), bascule le preset en custom si nécessaire, diffuse `ScalabilityChange`, et `Save_Settings` persiste.
6. **Lecture** — `QScalability_GetSettingValue(Setting, OutValue)` pour l'UI du menu d'options.

---

## 5. Points d'intégration

- **Dépendances module** : `Core`, `DeveloperSettings` (public) ; `CoreUObject`, `Engine`, `Slate`, `SlateCore` (privé). Aucune dépendance vers d'autres plugins QANGA.
- **Consommateurs** : le **menu d'options** (UI) lit/écrit les réglages via le subsystem et écoute `ScalabilityChange`. Les réglages s'appliquent au moteur via **commandes console / CVars** (donc compatibles avec n'importe quel sous-système réglable par CVar).
- **Configuration** : *Project Settings → QScalability* (`UQScalability_Settings`) — liste des réglages chargés au démarrage + paramètres de benchmark.
- **Persistance** : SaveGame dédié (`UQScalability_SaveGame`), indépendant de `QSave`.

---

## 6. Gotchas, invariants et pièges

- **Distinct de la Scalability native d'Unreal.** Ce système est **propre au projet** et data-driven ; ne pas le confondre avec `Scalability.ini`/`sg.*` d'UE (qu'il peut néanmoins piloter via des commandes console).
- **Application via console = dépend des CVars.** Un réglage en mode `Command` n'a d'effet que si la commande/CVar existe et prend effet au bon moment ; une CVar `read-only`/`cheat` peut être ignorée selon le contexte (notamment hors éditeur).
- **Benchmark = heuristique.** Les scores CPU/GPU sont des estimations échantillonnées ; un auto-set agressif peut mal choisir sur du matériel atypique. `Benchmark_*_Multiplier` permet de recalibrer.
- **Dépendances de presets.** Modifier un réglage listé dans `Depends_On_preset` fait passer le preset global en « custom » ; bien renseigner ces liens, sinon l'UI de preset se désynchronise de l'état réel.
- **`Settings_Name_ID` = clé.** L'indexation se fait par cet ID `FName` ; deux data assets de même ID se masquent dans `Settings_Data`.
- **`Is_Global_Preset` ne s'exécute pas au load.** Un preset global sert à **poser d'autres réglages**, pas à s'appliquer lui-même — ne pas y mettre de logique d'exécution directe.

---

## 7. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QScalability/Public/QScalability_GI_SubSystem.h` / `Private/…cpp` | Orchestrateur : load, benchmark, auto-set, apply, save |
| `Source/QScalability/Public/QScalability_DataAsset.h` / `…cpp` | Définition d'un réglage (type, display, presets, exécuteur) |
| `Source/QScalability/Public/QScalability_Command_DataAsset.h` | Bundle de commandes console |
| `Source/QScalability/Public/QScalability_Object.h` / `…cpp` | Exécuteur custom (`AutoSet`/`Init`/`Update`) |
| `Source/QScalability/Public/QScalability_SaveGame.h` | Persistance valeurs + scores benchmark |
| `Source/QScalability/Public/QScalability_Settings.h` / `…cpp` | `UDeveloperSettings` (liste réglages, benchmark) |
| `Source/QScalability/Public/QScalability_Struct.h` | `FQScalability_Data`, data par type, presets, enums |
| `QScalability.uplugin` | Descripteur : module Runtime `QScalability` |
