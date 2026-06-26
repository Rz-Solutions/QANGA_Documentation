# QSave — Architecture

> Plugin runtime (auteur : Iolacorp) : un **framework de sauvegarde** en couches. Des primitives de payload (clé-valeur typée, bytes compressés, en-tête versionné) ; un **conteneur maître** (`UQS_SaveGame`) qui agrège plusieurs SaveGames/objets et les **sérialise + compresse (Oodle)**, en synchrone ou asynchrone ; et un **orchestrateur monde** (`UQS_WorldSubSystem`) qui gère l'auto-save, la rotation et les backups.

---

## 1. But d'usage

QANGA doit sauvegarder un état de jeu riche et hétérogène : réglages, état de joueur, données de monde — sans réécrire une classe `USaveGame` monolithique par système, et sans bloquer la frame pendant la sérialisation. `QSave` fournit :

- un **payload générique typé** (`FQS_CommonBaseType`) pour stocker des valeurs simples par clé `FName`, sans classe dédiée ;
- un **conteneur maître** qui regroupe N sous-SaveGames et objets, les compresse (Oodle) et peut le faire **en tâche de fond** (copie + async) pour ne pas figer le jeu ;
- une **gestion monde** avec auto-save périodique, **rotation** de fichiers et **backups**, plus la détection automatique de la dernière sauvegarde valide.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Trois couches.** (1) *Primitives* : `FQS_CommonBaseType` (KV typée), `FQS_BytesData` (octets sérialisés + classe + flag de compression), `FQS_Header` (version + dates + nom, lu/écrit en binaire). (2) *Conteneur* : `UQS_SaveGame`. (3) *Orchestration* : `UQS_WorldSubSystem` (+ `UQS_GI_SubSystem` prévu pour les réglages).
- **Conteneur maître agrégateur.** `UQS_SaveGame` tient des maps `FName → USaveGame*` et `FName → UObject*` (`Internal_SaveGame` / `Internal_Object`). On y ajoute/retire des sous-objets (`QS_Add_SaveGame`, `QS_Add_Object`, …). À la sauvegarde, chaque entrée est sérialisée puis **compressée Oodle** dans `Compressed_SaveGame` / `Compressed_Object` (`TMap<FName, FQS_BytesData>`).
- **Auto-save non bloquant par copie.** Pour éviter de figer le jeu, l'auto-save prend une **copie** des objets (`Copy_SaveGame` / `Copy_Object`) puis sérialise/compresse **en async** (`SerializeAndCompress_Copy_Async`, `Create_Copy_Internal_SaveGameAndObject_Async`). Les délégués `QS_SerializedEnd` / `QS_UnSerializedEnd` signalent la fin.
- **Orchestrateur monde.** `UQS_WorldSubSystem` (WorldSubsystem) possède le `WorldSaveGame`, arme un timer d'auto-save (`Auto_Save_Time = 600 s`), gère une **rotation** (`WorldSaveGame_MaxRotation = 3`, `ComputeNextRotation`), crée des backups (`Internal_Create_Backup`) et, au démarrage, **retrouve et charge la dernière sauvegarde valide** (`Get_LastSaveGame` → `Internal_Load_SaveGame`). Diffuse `QS_OnSaveIsReadyd` quand prêt.
- **En-tête versionné.** `FQS_Header` (Major/Minor version, GameName, dates, classe) est écrit/relu en binaire (`FMemoryWriter`/`FMemoryReader`) — base d'une stratégie de migration de saves.
- **Configuration centralisée.** `UQS_Settings` (`UDeveloperSettings`, `config=Game`) regroupe les réglages d'auto-save, de nommage, de classes par défaut, de version et de compression.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQS_WorldSubSystem` | `UWorldSubsystem` | Orchestrateur monde : auto-save, rotation, backup, chargement de la dernière save valide |
| `UQS_SaveGame` | `USaveGame` | Conteneur maître : agrège SaveGames/objets, sérialise + compresse (Oodle), async + copie |
| `UQS_GI_SubSystem` | `UGameInstanceSubsystem` | **Stub** (prévu pour réglages input/jeu/graphismes ; corps commenté) |
| `UQS_Library` | `UBlueprintFunctionLibrary` | Get/Set typés sur `FQS_CommonBaseType` (11 types) + helpers de (dé)sérialisation/compression |
| `UQS_Settings` | `UDeveloperSettings` (`config=Game`) | Réglages : auto-save, noms, classes par défaut, version, compression |
| `FQS_CommonBaseType` | `USTRUCT` | KV typée : maps `FName → {bool, uint8, int32, int64, double, FName, FString, FText, FVector, FRotator, FTransform}` |
| `FQS_BytesData` | `USTRUCT` | Octets sérialisés + `ClassPath` + `IsCompressed` |
| `FQS_Header` | `USTRUCT` | En-tête de save versionné (dates, nom, version, classe) |

**Délégués** : `UQS_SaveGame::QS_SerializedEnd(bool)` / `QS_UnSerializedEnd(bool)` ; `UQS_WorldSubSystem::QS_OnSaveIsReadyd()`.

---

## 4. Flux de données et cycle de vie

1. **Init monde** — `UQS_WorldSubSystem::OnWorldBeginPlay` → `Init_Save` : cherche la dernière sauvegarde valide (`Get_LastSaveGame`), la charge (`Internal_Load_SaveGame` → `UnSerializeAndUnCompress`), ou en crée une neuve (`Internal_Create_SaveGame`). Diffuse `QS_OnSaveIsReadyd`.
2. **Enregistrement de données** — les systèmes ajoutent leurs sous-SaveGames/objets au `WorldSaveGame` (`QS_Add_SaveGame` / `QS_Add_Object` par `FName`), et/ou écrivent des valeurs simples dans un `FQS_CommonBaseType` via `UQS_Library` (`SetInt32Value`, `SetVectorValue`, …).
3. **Auto-save** — le timer (`Auto_Save_Time`) déclenche `UpdateAutoSave` : copie des objets (`Create_Copy_Internal_SaveGameAndObject`), sérialisation + compression Oodle **async** (`SerializeAndCompress_Copy_Async`), puis écriture disque avec rotation (`ComputeNextRotation`) et backup (`Internal_Create_Backup` → `Internal_OnBackupFinich`).
4. **Save manuel** — `UQS_SaveGame::SaveData` / `UQS_WorldSubSystem::QS_CreateBackup` font la même chaîne à la demande.
5. **Chargement** — `UnSerializeAndUnCompress(_Async)` décompresse les `FQS_BytesData` et reconstruit les objets via leur `ClassPath` (cf. `Default_SaveGameClass`/`Default_ObjectClass`).
6. **Lecture** — les systèmes récupèrent leurs objets (`QS_Get_SaveGame` / `QS_Get_Object`) ou leurs valeurs (`UQS_Library::Get*Value`).

---

## 5. Points d'intégration

- **Dépendances module** : `Core`, `DeveloperSettings`, `Serialization`, `QSettings` (public) ; `CoreUObject`, `Engine`, `Slate`, `SlateCore` (privé). Plugin requis : `QSettings`.
- **QSettings** : fournit la configuration/les chemins ; `QSave` s'appuie dessus pour résoudre où écrire et avec quels réglages.
- **Compression** : Oodle (via le module `Serialization` / l'API moteur de compression) pour réduire la taille des saves.
- **Consommateurs** : tout système gameplay qui doit persister un état entre sessions (joueur, monde, progression). Les réglages passent plutôt par `QSettings`/`UQS_Settings`.

---

## 6. Gotchas, invariants et pièges

- **`UQS_GI_SubSystem` est un stub.** Son corps est commenté ; ne pas s'attendre à ce qu'il gère les SaveGames d'input/graphismes — c'est `UQS_WorldSubSystem` qui porte la logique réelle (save monde).
- **Reconstruction par `ClassPath`.** `FQS_BytesData` stocke le chemin de classe ; renommer/déplacer une classe d'objet sauvegardé casse la désérialisation des saves existantes. Versionner via `FQS_Header`.
- **Copie pour l'async.** L'auto-save sérialise une **copie** ; modifier les objets pendant l'auto-save n'affecte pas le snapshot en cours, mais attention à la cohérence si l'on s'attend à sauver l'état « le plus récent ».
- **Rotation + backup.** Le monde tourne sur `WorldSaveGame_MaxRotation` (3) fichiers ; vérifier l'espace disque et la cohérence du nommage (`UQS_Settings` → catégorie *Qanga_Save_Name*).
- **Sauvegarde locale, non répliquée.** `QSave` écrit sur disque (côté serveur/standalone) ; ce n'est pas un canal réseau. La distribution d'état aux clients reste du ressort des systèmes de réplication.
- **Toujours attendre `QS_OnSaveIsReadyd`.** Lire le `WorldSaveGame` avant qu'il soit prêt (`QS_SaveGameIsReady`) peut donner un conteneur vide/partiel.

---

## 7. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QSave/Public/QS_WorldSubSystem.h` / `Private/…cpp` | Orchestrateur : auto-save, rotation, backup, load dernière save |
| `Source/QSave/Public/QS_SaveGame.h` / `…cpp` | Conteneur maître : agrégation + sérialisation/compression (Oodle), async |
| `Source/QSave/Public/QS_Library.h` / `…cpp` | Get/Set typés + helpers (dé)sérialisation/compression |
| `Source/QSave/Public/QS_Struct.h` | `FQS_CommonBaseType`, `FQS_BytesData`, `FQS_Header` |
| `Source/QSave/Public/QS_Settings.h` / `…cpp` | `UDeveloperSettings` (auto-save, noms, classes, version, compression) |
| `Source/QSave/Public/QS_GI_SubSystem.h` | `UGameInstanceSubsystem` (stub) |
| `QSave.uplugin` | Descripteur : Runtime, plugin `QSettings` |
