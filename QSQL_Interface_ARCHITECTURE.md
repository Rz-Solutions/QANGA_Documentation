# QSQL_Interface — Architecture

> Plugin runtime (auteur : Iolacorp) : une **couche d'accès SQLite** mince pour QANGA. Son cœur est un modèle **clé-valeur binaire** (`int64` index → `TArray<uint8>` BLOB) adossé à SQLite, avec bases **statiques** (authoring, `Content/DataBases/`) et **dynamiques** (runtime, `SaveDir`), sauvegardes rotatives, et accès synchrone **ou** asynchrone. C'est la brique de persistance de `QSpatialGrid`.

---

## 1. But d'usage

QANGA persiste de grandes quantités de données « accrochées au monde » (état de régions, données de grille, sauvegardes). Plutôt que des fichiers `.sav` monolithiques, le projet utilise **SQLite** pour un stockage indexé, requêtable et incrémental. `QSQL_Interface` enveloppe `SQLiteCore` derrière une API Blueprint/C++ simple, centrée sur un cas d'usage précis : **stocker/relire un blob d'octets par index `int64`** (typiquement un `UObject` sérialisé représentant une cellule/secteur/part de `QSpatialGrid`).

---

## 2. Vue d'ensemble / décisions d'architecture

- **Un objet = une base.** `UQSQL_DB_Object` (`UObject` Blueprintable) encapsule **une** `FSQLiteDatabase` (`DB_Instance`) + un `FSQLitePreparedStatement` réutilisable. `Init` / `Close` / `CreateNewDataBase` gèrent son cycle de vie.
- **Statique vs dynamique.** Le drapeau `DynamicData` choisit l'emplacement : base **dynamique** dans le `SaveDir` projet (mutable au runtime) ou base **statique** dans `Content/DataBases/` (authoring, lecture seule sauf `WriteEditor`).
- **Modèle « Storage » clé-valeur.** L'API réellement implémentée est le stockage indexé : une table `(Index INTEGER, Data BLOB)` manipulée par `QSQL_Storage_Create_Table` / `AddUpdateData` / `GetData` / `DeleteData` / `GetIndex`. C'est volontairement minimal : pas d'ORM, juste un dictionnaire persistant `int64 → bytes`.
- **Sync + async.** Les opérations Storage existent en version synchrone et **asynchrone** (`QSQL_Storage_AddUpdateData_Async` / `GetData_Async`) avec délégués de complétion (`FQSQL_OnSetStorageDataDelegate` / `FQSQL_OnGetStorageDataDelegate`). `QSQL_GetDB()` expose la `FSQLiteDatabase*` brute pour des requêtes complexes maison.
- **Sauvegardes rotatives.** `CreateBackup` produit `.dbBackup0..N` (`MaxBackup_Number = 4`) ; `Get_BackupFile` liste les sauvegardes datées.
- **Schéma typé en data.** `FQSQL_TableField` décrit un champ (type `EQSQL_DataType`, contraintes NOT NULL / PRIMARY KEY / AUTOINCREMENT / UNIQUE) et sait se rendre en SQL (`GetString`).

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQSQL_DB_Object` | `UObject` (`Blueprintable`) | Wrapper d'une base SQLite : init/close/backup, stockage clé-valeur (sync + async), accès DB brut |
| `UQSQL_SubSystem` | `UWorldSubsystem` | **Coquille vide** (placeholder ; aucune logique pour l'instant) |
| `UQSQL_FunctionLib` | `UBlueprintFunctionLibrary` | **Coquille vide** (placeholder) |
| `FQSQL_TableField` | `USTRUCT` | Définition d'un champ de table (type + contraintes) → SQL |
| `FQSQL_Field_Array` | `USTRUCT` | Liste de champs (schéma de table) |
| `EQSQL_DataType` | `UENUM` | Types SQLite : `INTEGER` / `TEXT` / `BLOB` / `REAL` / `NUMERIC` |

**Délégués** : `FQSQL_OnSetStorageDataDelegate(int64 Index, bool IsValid)`, `FQSQL_OnGetStorageDataDelegate(int64 Index, TArray<uint8> Data, bool IsValid)`.

---

## 4. Flux de données et cycle de vie

1. **Ouverture** — un système crée un `UQSQL_DB_Object` et appelle `Init(Name, DynamicData, WriteEditor)` : résolution du chemin (`SaveDir` ou `Content/DataBases/`), ouverture/création de la base, introspection des tables (`Get_DB_TableInfo`).
2. **Préparation** — `QSQL_Storage_Create_Table(Table)` crée la table de stockage `(Index, Data)` si absente.
3. **Écriture** — `QSQL_Storage_AddUpdateData(Table, Index, Data)` insère ou met à jour le blob pour cet index (UPSERT). Version async : `…_Async` avec délégué de complétion.
4. **Lecture** — `QSQL_Storage_GetData(Table, Index, OutData)` (ou `GetData_Async`) renvoie le blob ; `QSQL_Storage_GetIndex` teste l'existence.
5. **Sauvegarde** — `CreateBackup()` (sous verrou) effectue une copie rotative `.dbBackupN`.
6. **Fermeture** — `Close()` / `BeginDestroy()` libèrent le statement et la base.

---

## 5. Points d'intégration

- **Dépendances module** : `Core`, `SQLiteCore` (public) ; `CoreUObject`, `Engine`, `Slate`, `SlateCore` (privé). Plugin requis : `SQLiteCore`.
- **Consommateurs** : [`QSpatialGrid`](QSpatialGrid_ARCHITECTURE.md) utilise `UQSQL_DB_Object` comme support de persistance de ses données de cellules/secteurs/parts (DB statique + DB dynamique `DynamicDB`). Tout système ayant besoin d'un dictionnaire `int64 → bytes` persistant peut s'y adosser.
- **Pas de réplication** : c'est de la persistance locale (disque), conçue surtout côté serveur/standalone.

---

## 6. Gotchas, invariants et pièges

- **Seul le modèle « Storage » est vivant.** Le bloc « Common Table » (création/suppression de tables et de champs génériques : `QSQL_CreateTable`, `QSQL_AddField_ToTable`…) est **commenté/non implémenté**. Ne pas s'y fier ; utiliser l'API Storage (`int64 → blob`).
- **`UQSQL_SubSystem` et `UQSQL_FunctionLib` sont vides.** Ce sont des points d'extension non remplis ; toute la logique vit dans `UQSQL_DB_Object`.
- **Statique vs dynamique = emplacement disque.** Une base statique (`Content/DataBases/`) n'est écrite que si `WriteEditor` est vrai (workflow d'authoring) ; les mutations runtime vont dans une base dynamique (`SaveDir`).
- **Backups sous verrou.** `CreateBackup` nécessite que la base ne soit pas en cours d'écriture ; sérialiser les backups avec les writes.
- **Statement réutilisé.** Un seul `FSQLitePreparedStatement` est gardé et réinitialisé (`Reset_Statement`) ; ne pas supposer un accès concurrent multi-thread naïf sur un même `UQSQL_DB_Object` — passer par les versions async ou `QSQL_GetDB()` pour du travail complexe.
- **Index = clé primaire `int64`.** Le modèle suppose des index `int64` (souvent des hash de coordonnées de `QSpatialGrid`) ; la qualité du hash (collisions) relève de l'appelant.

---

## 7. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QSQL_Interface/Public/QSQL_DB_Object.h` / `Private/…cpp` | Wrapper SQLite : init/close/backup, stockage clé-valeur (sync + async) |
| `Source/QSQL_Interface/Public/QSQL_Struct.h` | `EQSQL_DataType`, `FQSQL_TableField`, `FQSQL_Field_Array` |
| `Source/QSQL_Interface/Public/QSQL_SubSystem.h` | `UWorldSubsystem` (placeholder vide) |
| `Source/QSQL_Interface/Public/QSQL_FunctionLib.h` | `UBlueprintFunctionLibrary` (placeholder vide) |
| `QSQL_Interface.uplugin` | Descripteur : Runtime, plugin `SQLiteCore` |
