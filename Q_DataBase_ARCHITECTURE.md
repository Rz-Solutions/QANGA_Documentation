# Q_DataBase — Architecture

> Plugin (auteur : Iolacorp ; entrées par CyberAlien) : un **registre de données en mémoire, requêtable**. Des entrées génériques à payload **type-erased** (`FInstancedStruct`), classées par **ID / catégories / tags / attributs / position spatiale**, regroupées en bases (`UQDB_DataBase`) qui maintiennent des **index de pré-filtrage** pour des requêtes rapides (sync **et** async). Module runtime + module éditeur.

---

## 1. But d'usage

QANGA manipule beaucoup de « données de contenu » hétérogènes (objets, recettes, POI, définitions diverses) qu'il faut **rechercher et filtrer** efficacement : « toutes les entrées de catégorie X », « ayant le tag Y », « dont l'ID est Z », « proches de cette position ». Plutôt qu'un `UDataTable` rigide (une struct par table) ou des recherches linéaires répétées, `Q_DataBase` fournit :

- une **entrée générique** (`UQDB_Data`) dont le payload est un `FInstancedStruct` (n'importe quelle struct, sans classe dédiée), accompagné de métadonnées de classification ;
- une **base** (`UQDB_DataBase`) qui **pré-calcule des index** (par catégorie, tag, attribut, ID, spatial) à partir de ses entrées, pour transformer les requêtes en lookups de map plutôt qu'en balayages ;
- une **API de requête riche** (par ID, nom, sous-chaîne, catégorie, tag, attribut, spatial), en versions synchrones et **asynchrones** (avec délégués).

---

## 2. Vue d'ensemble / décisions d'architecture

- **Entrée à payload type-erased.** `UQDB_Data` (`UDataAsset`) porte un `Base_Data_Struct` (`FInstancedStruct`) + une `Data_Map` (`TMap<FName, FInstancedStruct>`) — le contenu réel est une struct quelconque, sans avoir à dériver une classe par type de donnée. Autour : identité (`Data_Name`, `Data_ID`), classification (`Data_Catergory`, `Data_Tags`, `Data_Attribute`, `Data_AdvancedAttribute`), spatial (`Data_Spacial`, `Data_Location`, `Data_Bound`, hiérarchie parent/enfant), refs additionnelles, métadonnées (transform, dates, `Data_IsDirty`, `Dynamic_Data`).
- **Miroir struct pour transfert.** `FQDB_DataStruct` reflète les champs de `UQDB_Data` en struct simple (`QDB_SetData_FromStruct` / `QDB_ExportData_ToStruct`) — pour sauvegarder/transmettre une entrée sans l'objet.
- **Base = entrées + index de pré-filtrage.** `UQDB_DataBase` (`UDataAsset`) tient le tableau brut (`Data_Array`) **et** des index : `Data_Map_ID` (par ID), `Data_Map_Category`, `Data_Map_Tags`, `Data_Map_Attribute`, `Data_Map_Attribute_Complete`, `Data_Spacial`/`Data_Spacial_PrimaryData`, `Data_Dynamics`. `QDB_ComputePrefilterFromAllData` (re)construit ces index.
- **Requêtes en lookup, pas en balayage.** Les helpers `QDB_Found_Data_ByCatergory` / `ByTag` / `ByName` / `Get_Data_ID` exploitent les index ; les filtres `QDB_Filter_*` (set vs `_Strict`) opèrent sur une entrée donnée. Versions `_Async` avec `FFilterDelegate` / `FEnd` pour ne pas bloquer la frame sur de gros volumes.
- **Données dynamiques vs statiques.** `Dynamic_Data` distingue les entrées créées au runtime (mutables, `Data_Dynamics`) des entrées d'authoring ; `QDB_Remove_Data(..., OnlyIfDynamic)` protège par défaut les données statiques.
- **Spatial hiérarchique.** Les entrées peuvent porter une localisation + un rayon (`Data_Bound`) et une hiérarchie parent/enfant (`Data_Spacial_Parent`/`Child`, résolue en IDs via `QDB_ComputeRef_ToID`) — base de requêtes spatiales primaires.
- **Subsystem GameInstance + module éditeur.** `UQDB_GI_SubSystem` (GameInstanceSubsystem) gère les bases au runtime ; `Q_DataBase_Editor` (`UQDB_Editor_Function`) fournit des helpers d'authoring (via `AssetRegistry`).

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQDB_Data` | `UDataAsset` (`Blueprintable`) | Une entrée : payload `FInstancedStruct` + classification (ID/cat/tags/attributs) + spatial + métadonnées |
| `UQDB_DataBase` | `UDataAsset` (`Blueprintable`) | Une base : entrées + index de pré-filtrage + API de requête (sync/async) |
| `UQDB_GI_SubSystem` | `UGameInstanceSubsystem` | Gestion runtime des bases (enregistrement/accès) |
| `UQDB_Function_Library` | `UBlueprintFunctionLibrary` | Helpers |
| `UQDB_Editor_Function` | `UBlueprintFunctionLibrary` (module Editor) | Helpers d'authoring (AssetRegistry) |
| `FQDB_DataStruct` | `USTRUCT` (`BlueprintType`) | Miroir struct d'une entrée (transfert/sauvegarde) |
| `FQDB_Data_Array` | `USTRUCT` | Tableau d'entrées (valeur des maps d'index) |
| `FQDB_AttributValue` | `USTRUCT` | Index attribut → (valeur → entrées) |
| `FQDB_FNameData` | `USTRUCT` | Liste de `FName` (attribut avancé) |
| `FQDB_MultiLocation` | `USTRUCT` | Localisation + rayon (requête spatiale) |

**Délégués** : `FFilterDelegate(TArray<UQDB_Data*>)`, `FEnd(UQDB_DataBase*)`.

---

## 4. Flux de données et cycle de vie

1. **Authoring / construction** — on crée des `UQDB_Data` (data assets ou au runtime) et on les ajoute à un `UQDB_DataBase` (`QDB_Add_Data` / `QDB_Set_DataArray`).
2. **Pré-filtrage** — `QDB_ComputePrefilterFromAllData` (ou `_Async`) parcourt `Data_Array` et remplit les index (par ID, catégorie, tag, attribut, spatial, dynamique). À refaire après modification du jeu d'entrées.
3. **Requête** — l'appelant interroge : par ID (`QDB_Get_Data_ID` / `QDB_Found_Data_ID`), catégorie (`QDB_Found_Data_ByCatergory`), tag (`QDB_Found_Data_ByTag`), nom / sous-chaîne (`QDB_Found_Data_ByName` / `ByContaineString`), attribut, ou spatial. Les requêtes lourdes ont une variante `_Async` rappelée par délégué.
4. **Filtrage d'un set en entrée** — `QDB_Filter_Input_By*` filtre un tableau fourni (pas toute la base) ; `QDB_Filter_By*` / `_Strict` testent une entrée unique (membre partiel vs strict).
5. **Comparaison** — `QDB_Compare(A, B, OutDifference, OutUnion)` calcule différence/union de deux ensembles d'entrées.
6. **Mutation runtime** — ajout/retrait d'entrées dynamiques (`Data_Dynamics`), `Data_IsDirty` pour suivre les modifications ; `QDB_Remove_Data(OnlyIfDynamic)` protège les données statiques.

---

## 5. Points d'intégration

- **Dépendances module (runtime)** : `Core` (public) ; `CoreUObject`, `Engine`, `Slate`, `SlateCore` (privé). Utilise `FInstancedStruct` (StructUtils). **Aucune** dépendance vers d'autres plugins QANGA. Module éditeur : `Q_DataBase` + `AssetRegistry`.
- **Consommateurs** : tout système ayant besoin d'un catalogue requêtable de données de contenu (objets, POI, définitions). Le payload `FInstancedStruct` laisse chaque domaine définir sa propre struct sans coupler `Q_DataBase`.
- **Pas de réplication ni de persistance disque intégrée** : c'est un registre **en mémoire**. La persistance (si voulue) passe par `FQDB_DataStruct` + un système de save (`QSave`) ou une base (`QSQL_Interface`) côté appelant. À ne pas confondre avec [`QSpatialGrid`](QSpatialGrid_ARCHITECTURE.md) (streaming spatial DB) ni [`QSQL_Interface`](QSQL_Interface_ARCHITECTURE.md) (SQLite).

---

## 6. Gotchas, invariants et pièges

- **Les index ne se mettent pas à jour tout seuls.** Après ajout/retrait/modification d'entrées, il faut rappeler `QDB_ComputePrefilterFromAllData` (ou `_Async`) ; sinon les requêtes par catégorie/tag/attribut renvoient des résultats périmés.
- **`FInstancedStruct` = couplage par type de struct.** Le payload est une struct quelconque ; lire une entrée suppose de connaître/caster vers la bonne struct. Renommer une struct de payload casse la relecture des données sérialisées (`FQDB_DataStruct`).
- **Set vs Strict.** `QDB_Filter_By*` (appartenance partielle) et `QDB_Filter_By*_Strict` (toutes les valeurs requises) ont des sémantiques différentes ; choisir selon le besoin (« au moins un tag » vs « tous les tags »).
- **Spatial = hiérarchie à résoudre.** Les liens parent/enfant spatiaux sont des pointeurs d'objet **et** des IDs (`ID_Data_Spacial_Parent`/`Child`) ; `QDB_ComputeRef_ToID` doit être appelé pour garder les deux cohérents (notamment avant sérialisation).
- **Coquilles de helpers.** `UQDB_Function_Library` est minimal ; l'essentiel de l'API vit sur `UQDB_DataBase`.
- **« Catergory ».** Le champ est orthographié `Data_Catergory` / `QDB_Found_Data_ByCatergory` dans le code — respecter l'orthographe exacte côté Blueprint/C++.

---

## 7. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/Q_DataBase/Public/QDB_Data.h` / `Private/…cpp` | Entrée `UQDB_Data` (payload + classification + spatial) |
| `Source/Q_DataBase/Public/QDB_DataBase.h` / `…cpp` | Base `UQDB_DataBase` : index de pré-filtrage + API de requête |
| `Source/Q_DataBase/Public/QDB_DataStruct.h` | `FQDB_DataStruct` + helpers (`FQDB_FNameData`) |
| `Source/Q_DataBase/Public/QDB_GI_SubSystem.h` / `…cpp` | Gestion runtime des bases |
| `Source/Q_DataBase/Public/QDB_Function_Library.h` | Helpers |
| `Source/Q_DataBase_Editor/Public/QDB_Editor_Function.h` | Helpers d'authoring (AssetRegistry) |
| `Q_DataBase.uplugin` | Descripteur : module Runtime `Q_DataBase` + module Editor `Q_DataBase_Editor` |
