# QStorage: coffre a stockage LOCAL, replique, persistant, constructible et posable
### Document de design v3 : ARBITRE PAR RzZz le 2026-08-22. Le design est valide, J1 est autorise. Aucune ligne de code n avait ete ecrite au moment de l arbitrage.

> Historique : v1 (design initial), v2 (consolide apres trois revues hostiles), v3 (arbitrages RzZz integres, voir 0.1).

---

## 0. Synthese en 10 lignes

1. On construit un conteneur a contenu **local** (chaque coffre a ses propres objets), server-authoritative, persistant a travers les mises a jour, utilisable en deux poses : construit par QBuilder, ou place a la main dans un sous-niveau QLevel.
2. Ca vit dans un nouveau plugin `Plugins/QStorage/` (couche Q\*), C++ mince plus assets BP, **sans aucune dependance sur QModule** (voir 1.3).
3. La verite du contenu reste l `InventoryComponent` BP (`Content/Systems/Item/InventoryComponent.uasset`) cote serveur : attache au runtime pour le coffre de niveau, **deja present et cable** sur le builder pour le coffre construit.
4. **Les deux poses n ont pas le meme modele reseau, et c est voulu** (encadre en tete de section 5). Le coffre construit **est** le builder, qui se replique deja (`bReplicates = true`, `QBuilder_BuilderActor.cpp:22`) : on garde sa chaine en production. Le coffre de niveau ne se replique pas et passe par snapshot et deltas, sur le canal d interaction existant `Lib_OptimizedState::SendOptimizedActorInteraction`.
5. Le client n envoie **jamais** de position ni de reference d objet : il dit "j interagis", le serveur resout l acteur, choisit le coffre et renvoie un handle.
6. L identite durable est un `FGuid` alloue par le serveur, resolu au respawn par un index d ancrage **calcule uniquement cote serveur**, avec tombstone a la destruction.
7. La persistance passe par `GameDataManager` / `DataObject` (`Plugins/DataManager/Content/`), mais **QStorage possede sa propre cadence d ecriture** : une cle tableau par coffre, index segmente par secteur, flush a la fermeture de session, jamais par mouvement d objet.
8. Le versionnement est un entier `SchemaVersion` avec echelle de migration, jamais l egalite stricte destructrice de QBuilder (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:994`) ni de DQS.
9. L UI duplique `Content/Widget/Storage/PUW_VehicleStorage.uasset` ; on ne touche ni `BP_Storage` ni `PUW_Storage3d` (44 emplacements de relais et stations).
9bis. **Ce que le chantier repare en plus de ce qu il ajoute** : la chaine storage du builder existe deja mais s indexe sur un `BuilderID` **realloue a chaque chargement** (`QBuilder_SubSystem.cpp:728` et `:733`, champ sauvegarde marque `// Not Use`). Le contenu des zones construites ne survit donc pas de facon fiable a un redemarrage. QStorage y substitue son `FGuid` : c est **tout** le perimetre cote construit (4.0 et 7.4).
10. **Le seul vrai point dur** : la couche d ecriture heritee. `TempDB.WriteTempData` reecrit le slot monde ENTIER plus son miroir `BACKUP_`, et `AsyncSaveGameToSlot` serialise en memoire **de facon synchrone sur le game thread** (`C:/UE5_Share/Engine/Source/Runtime/Engine/Private/GameplayStatics.cpp`). Tout le design plie devant ce fait : c est lui qui impose l index segmente, la cadence possedee par QStorage, les plafonds durs, et un banc de mesure au jalon J1.

---

## 0.1 Arbitrages RzZz du 2026-08-22

Les dix questions de la section 11 sont tranchees. La section 11 conserve les decisions
et leur motif. Ce qui change structurellement :

| Sujet | Arbitrage | Effet |
|---|---|---|
| Regle "premier objet a construire" | **Lecture B, definitif** | Le perimetre de QStorage cote construit se reduit a fournir l identite stable a la place du `BuilderID`. J6 devient marginal. Voir 4.0 et 7.4. |
| WAL d intention (4.3) | **Supprime** | Remplace par une fenetre de perte bornee, assumee, journalisee, plus un toggle `.ini` de flush post-transfert evalue au banc J1. Voir 4.3. |
| Existence contre contenu (7.3) | **Option b** | Fenetre assumee, reconciliation, recuperation admin. Un `QBuilder_SaveData` force est mesure au banc J1, et n est adopte que sur pose et destruction. |
| Deplacement et destruction | **Point d extension additif accepte** | Destruction refusee si des piles resolues subsistent, deplacement autorise a vide uniquement. Voir 7.5. |
| Acces | **`OwnerOnly` par defaut** | Partage par `Allowed_By_ID` (`Plugins/QBuilder/Source/QBuilder/Public/QBuilder_Client.h:47`), `Public` reserve aux coffres de niveau, `GroupOnly` plus tard. L enum reste inchange. |
| Synchronisation live | **Retenue** | Deltas, snapshot fige et rebase conserves tels quels. Plan B explicite : refresh a l ouverture. |
| Composant d etat | **BP en v1** | Le port C++ n est pas abandonne, il est en attente de bascule. Le repli du 2.4 reste conditionne a l INCONNU 6. |
| Tirage pondere | **Extraction `Cy_*` dediee** | Pas de copie : `Plugins/QModule/Source/QModule/Public/QModuleLoot_Library.h` impose lui-meme le no-duplicates entre ses trois consommateurs. |
| Set QBuilder | **ICLAB** | Et l entree `QA_ICLAB_StorageBase` existante est **rebranchee ou remplacee**, jamais doublee. Voir 7.1. |
| Migration des anciens contenus | **Aucune** | Les contenus keyes sur `BuilderID` numerique ne sont pas stables par construction. `qstorage.Orphans` les ramasse. Voir 4.0. |
| Plafonds initiaux | **Fixes en `.ini`** | 128 sessions simultanees, `MaxHotContainers` 256, 5000 coffres persistes par monde, 2000 entrees par segment d index. A revisiter avec la courbe J1. |

**Consequence sur le planning** : J0 ne porte plus aucun inconnu de **design**. Il ne reste
que des mesures, replacees dans les jalons ou elles servent. J1 peut demarrer.

---

## 1. Verdict d architecture

### 1.1 Nouveau plugin, pas une extension

**Nouveau plugin `QStorage` (couche Q\*), hybride C++ mince plus assets Blueprint, zero nouveau format d items.**

- QBuilder n a **aucun payload libre par piece** : une instance vaut exactement `Data_ID(uint16) + FQBuilder_Int16Transform + Life(uint16) + Color(uint8)` (`Plugins/QBuilder/Source/QBuilder/Public/Struct/QBuilder_Struct_Data.h:16-63`). Le seul slot libre, `CustomDataMap`, est porte par le builder et **doublement mort sur disque** : serialisation commentee (`:382-384`, `:425`) et restauration absente de `QBuilder_SaveGame_CreateBuilder` (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:714-739`).
- Etendre le format binaire QBuilder est exclu : son test de version est une egalite stricte sur `"1.0.0"` (`Plugins/QBuilder/Source/QBuilder/Public/QBuilder_SubSystem.h:380`, teste a `Private/QBuilder_SubSystem.cpp:994` et `:1101`). Un bump efface 100 % des constructions de tous les serveurs.
- QModule est le systeme de modules du joueur, pas un conteneur monde. QNetState est un canal reseau par position qui **ne persiste rien** (grep Save/Serialize/Persist sur `Plugins/QNetState/Source/QNetState/Public/*.h` : 0 resultat).

### 1.2 Repartition C++ / Blueprint

C++ : registre des coffres vivants, registre de sessions, RPC valides, codec versionne et echelle de migration, hook QLevel, tirage pondere, commandes de diagnostic.
Blueprint : l acteur (il porte le composant **BP** `OptimizedStateComponent_C`, car le port C++ `UOptimizedStateComponent` est dormant : 0 reference `/Script/QNetState` sur 9489 assets scannes), l inventaire (`Content/Systems/Item/InventoryComponent.uasset`, 100 % BP), l UI.

**Point ouvert transverse, tranche a J0** : quelle couche ajoute les composants au runtime. Le seul precedent mesure est Blueprint (`VehiclePlayerOwner.BeginPlay`). Aucun appelant C++ de `SetIdAndGetData` ou `SetNotPersistentData` n existe dans `Plugins/` (grep .h/.cpp : zero). Faire l ajout en C++ imposerait de resoudre `InventoryComponent_C` et `PersistentDataComponent` par chemin, donc de tomber sur le piege cook connu du projet (soft ref C++ jamais cuite : marche en editeur, absente de la build packagee, en silence). **Decision par defaut : l ajout des composants se fait en Blueprint dans `BP_QStorage_Container`, comme `VehiclePlayerOwner`.** Si un besoin impose le C++, alors `TSoftClassPtr<UActorComponent>` dans `UQStorage_DevSettings` plus inscription de `Content/Systems/QStorage` dans `DirectoriesToAlwaysCook` et dans la graine EasyCook.

### 1.3 Dependances de module : QStorage ne depend PAS de QModule

Le V1 prevoyait une dependance privee sur QModule pour `QModuleInventory` et `QModuleLoot::RollWeightedItem`. **Abandonne**, pour deux raisons dont une seule est celle du relecteur :

- `QModule_InventoryBridge.h:21-52` est pawn-centric (`FindInventoryComponent(APawn*)`, `PawnOwnsInstance(APawn*)`) et se declare server-only par nature. Il ne s applique pas a un coffre. Surtout, `GrantItemAsset` fait `GenerateNewItemInstance + AddItemToInventory` (`Plugins/QModule/Source/QModule/Private/QModule_InventoryBridge.cpp:362-374`) : il **cree une instance neuve**, donc perd `SlotAttachments`, `Rarity`, `CustomizationInstanceId`. Le "ou le pont QModuleInventory" du V1 pour les transferts etait faux et dangereux : **il est retire**.
- Le jalon J5 (loot dans les coffres) creerait la pression inverse QModule vers QStorage, donc un cycle.

`QStorage.Build.cs` : `Core, CoreUObject, Engine, QLevel` (public/prive selon usage), rien d autre. `RollWeightedItem`, `ResolvePickupClass` et `HashWorldLocation` (`Plugins/QModule/Source/QModule/Public/QModuleLoot_Library.h:19-66`) sont **extraits en `Cy_*` par une tache dediee** (arbitrage RzZz). La copie est ecartee : l en-tete de `QModuleLoot_Library` impose deja le no-duplicates du tirage pondere entre ses trois consommateurs actuels, une copie irait contre ce que le fichier declare lui-meme. Cette extraction ne se trouve pas sur le chemin du socle J1 (elle sert au remplissage, donc a J5) : elle est planifiee comme tache dediee avant J5, avec relecture des trois consommateurs existants.

**Critique ecartee, partiellement** : l argument du "poids transitif" (QStorage tirerait QAI et DQS) est **non fonde**. Les `PrivateDependencyModuleNames` de QModule ne remontent pas dans QStorage, et QModule est deja actif projet-wide donc deja charge. J adopte quand meme l independance, mais pour le motif du cycle J5, pas pour le poids.

### 1.4 Activation

`UQStorage_DevSettings`, section `[/Script/QStorage.QStorage_DevSettings]` dans `Config/DefaultGame.ini`, **`Enabled=False` a la livraison**. Categorie de log `LogQStorage`.

**Chemin retour apres desactivation.** Repasser `Enabled=False` sur un monde qui a deja
tourne ne doit rien detruire. Regle : a l extinction, le subsystem flushe ce qu il a en
memoire puis cesse toute activite ; les acteurs ne s enregistrent plus, aucune ouverture
n est servie (refus `Unresolved`, jamais un coffre vide), et **aucune ecriture, aucune
purge, aucun tombstone n est emis**. Les segments d index et les DataObjects de coffres
restent intacts sur disque et sont repris tels quels a la reactivation, sous reserve du
controle de `SchemaVersion` du 4.4. Un coffre deja pose reste visible et devient
simplement inerte, avec un message localise. La consequence est assumee : un aller-retour
d activation n est pas une operation de nettoyage, et le nettoyage reste manuel et
explicite (`qstorage.Orphans`).

---

## 2. L acteur

```
AActor
 └── AQStorage_Container_Actor            (C++, QSTORAGE_API, bReplicates = false, tick desactive)
      └── BP_QStorage_Container            (BP : composants, mesh, tags, ajout runtime des composants)
           ├── BP_QStorage_Container_Build (preset QBuilder)
           └── BP_QStorage_Container_World (preset pose editeur, porte le tag QL_ExcludeMerge)
```

| Composant | Pourquoi |
|---|---|
| `StaticMeshComponent` (herite du C++) | visuel et collision d interaction |
| `OptimizedStateComponent_C` (BP) | etat leger par cle, canal deja en prod sur `BP_Shop` et `BP_Chest_A1` |
| `InteractiveFeedbackCustomTarget`, `TrackerComponent`, `ScannableComponent` | alignement sur `Content/GameplayActors/Storage/BP_Storage.uasset` |
| `PersistentDataComponent` + `InventoryComponent` (+ probablement `ORManagerComponent`) | ajoutes **au runtime, cote serveur, paresseusement a la premiere ouverture** |

### 2.1 Attachement paresseux (correction majeure)

Le V1 disait "ajoutes au runtime" sans dire quand. Au `BeginPlay`, un entrepot a 200 ancres CONTAINER instancierait 200 `PersistentDataComponent` plus 200 `InventoryComponent` plus 200 chargements asynchrones de `DataObject`, **par region occupee**, et le serveur charge l union des positions des 500 joueurs (`Plugins/QLevel/Source/QLevel/Private/QLevel_SubSystem.cpp:1255-1270`). Corrige :

- au `BeginPlay` serveur, le coffre ne fait que **s enregistrer** aupres du subsystem (position, kind, preset). Zero composant, zero I/O ;
- les composants sont attaches **a la premiere ouverture**, detaches avec flush a la fermeture de session ;
- plafond dur `MaxHotContainers` dans les settings, relie au plafond de sessions ouvertes ;
- **race de chargement traitee** : tant que `IsInventoryReady == false`, `Open` repond `Refused(NotReady)`. Il ne sert **jamais** un snapshot vide sur un inventaire non pret (sinon le depot du joueur est ecrase ou duplique quand le chargement DB atterrit).

### 2.2 Le montage d inventaire, et son inconnu bloquant

`InventoryComponent.ReplicateSaveInventory` appelle `Register Replicated UObject` sur la variable `UObjectReplicationManager : ORManagerComponent` **avant** son `SetStringArray`. Sur un acteur sans `ORManagerComponent`, cette reference est nulle, et en Blueprint un appel sur une reference nulle **avorte la frame d execution** : la sauvegarde ne s ecrit jamais, sans erreur, sans log. Le troisieme `Add Component by Class` de `VehiclePlayerOwner.BeginPlay` est tres probablement ce composant, et il est declare inconnu.

**Consequence de plan** : identifier ce troisieme composant est un prealable a J2, pas une note de bas de page (INCONNU 1). Le critere de fin de J2 devient "arret complet du process serveur puis relance, contenu identique", jamais "quitter et revenir".

### 2.3 Resolution du kind

A `BeginPlay`, le C++ resout `EQST_AnchorKind` : tag `QBuilder_ActorInstance` present (pose avant `FinishSpawningActor`, verifie a `Plugins/QBuilder/Source/QBuilder/Private/World/QBuilder_BuilderActor.cpp:338`) donne `PlayerBuilt`, sinon `LevelPlaced`. Le reste du comportement est identique ; seul le `FQST_FillSpec` du preset differe.

### 2.4 La cle de l OptimizedState (correction)

Le meme tag `QBuilder_ActorInstance` **detourne** la cle du composant vers la cle composite `BuilderID*100 + InstanceID*10 + DataID` (`Plugins/QBuilder/Source/QBuilder/Private/World/QBuilder_ActorInstanceComponent.cpp:104-112`), qui collisionne et depend d un `BuilderID` reattribue au chargement (`Private/QBuilder_SubSystem.cpp:716`). Correctif : des la resolution de l identite, QStorage appelle `SetCustomLocationKey` avec une cle derivee de son propre `FGuid`. Si la verification (INCONNU 6) montre que le composant **BP** ne l expose pas de la meme facon, on renonce a l OptimizedState pour les coffres construits et on ne le garde que pour les coffres de niveau.

---

## 3. Modele de donnees

La verite du contenu est un `InventoryComponent` BP cote serveur : on herite du stacking, de la rarete, des attachements et des transferts (`Content/Systems/Item/Lib_Inventory.uasset`). Les USTRUCT de QStorage portent l identite, le fil reseau, le remplissage et l index. Pas d initialiseurs designes (interdit UHT) : constructeurs.

```cpp
UENUM(BlueprintType)
enum class EQST_AnchorKind : uint8 { PlayerBuilt = 0, LevelPlaced = 1 };

UENUM(BlueprintType)
enum class EQST_AccessMode : uint8 { Public = 0, OwnerOnly = 1, GroupOnly = 2 };

UENUM(BlueprintType)                                  // jamais de refus muet
enum class EQST_Refusal : uint8 { None=0, NotReady=1, TooFar=2, Gone=3, Full=4,
                                  Conflict=5, NotOwner=6, RateLimited=7, Unresolved=8 };

USTRUCT(BlueprintType)
struct QSTORAGE_API FQST_AnchorKey                    // SERVER-SIDE ONLY: never sent by a client
{
    GENERATED_BODY()
    UPROPERTY(BlueprintReadOnly) int64 GridX;         // world position quantised to 10 cm
    UPROPERTY(BlueprintReadOnly) int64 GridY;
    UPROPERTY(BlueprintReadOnly) int64 GridZ;
    UPROPERTY(BlueprintReadOnly) EQST_AnchorKind Kind;
    FQST_AnchorKey();
    bool operator==(const FQST_AnchorKey& Other) const;
};
QSTORAGE_API uint32 GetTypeHash(const FQST_AnchorKey& Key);

USTRUCT(BlueprintType)
struct QSTORAGE_API FQST_ItemStack                    // wire + persisted mirror + fill spec
{
    GENERATED_BODY()
    UPROPERTY(BlueprintReadOnly) FName   ItemKey;     // DA_Item.ItemDataKey, stable identity
    UPROPERTY(BlueprintReadOnly) int32   Quantity;
    UPROPERTY(BlueprintReadOnly) int32   Rarity;
    UPROPERTY(BlueprintReadOnly) int32   SlotIndex;
    UPROPERTY(BlueprintReadOnly) FString InstanceId;
    UPROPERTY(BlueprintReadOnly) FString EncodedData; // FULL payload, persisted, never a key alone
    UPROPERTY(BlueprintReadOnly) bool    bUnresolved; // ItemKey missing from DA_AllRef
    FQST_ItemStack();
};

USTRUCT(BlueprintType)
struct QSTORAGE_API FQST_ContainerRecord
{
    GENERATED_BODY()
    UPROPERTY(BlueprintReadOnly) FGuid           ContainerGuid;
    UPROPERTY(BlueprintReadOnly) FQST_AnchorKey  Anchor;
    UPROPERTY(BlueprintReadOnly) FName           FillProfileTag;
    UPROPERTY(BlueprintReadOnly) FString         OwnerPlayerId;   // ServerAuth.GetPlayerId
    UPROPERTY(BlueprintReadOnly) EQST_AccessMode AccessMode;
    UPROPERTY(BlueprintReadOnly) int32           SlotCount;
    UPROPERTY(BlueprintReadOnly) bool            bFilled;
    UPROPERTY(BlueprintReadOnly) uint32          ContentVersion;  // optimistic concurrency, in RAM + persisted
    UPROPERTY(BlueprintReadOnly) int64           CreatedSeq;      // monotonic, tombstone comparison
    FQST_ContainerRecord();
};
```

Trois changements par rapport au V1 : `EncodedData` est desormais **persiste** (et non un simple miroir de cles), `LastMutationUnixSeconds` **sort de l index** (il salissait l index a chaque mouvement d objet, donc reencodait tout le tableau), et `ContentVersion` est un champ explicite du record, persiste, donc il survit au cycle IsOn/IsOff qui detruit et recree l acteur.

Classes : `UQStorage_World_SubSystem` (registre des coffres vivants, registre de sessions, verite serveur, persistance), `UQStorage_Client_Component` (entonnoir RPC sur le PlayerController), `UQStorage_Fill_DataAsset` (`UPrimaryDataAsset`, type `QStorageFill` a inscrire dans l AssetManager), `UQStorage_Function_Library` (facade BP, fonctions prefixees `QST_*`), `UQStorage_DevSettings`.

---

## 4. Identite et persistance

### 4.0 QStorage corrige un bug de persistance qui existe deja (fait apporte par RzZz, verifie)

La chaine storage de `Content/Systems/QBuilder/Actor/QBuilder_Builder_Actor_BP` est
complete de bout en bout : `InventoryComponent`, `PersistentDataComponent` avec bind sur
`OnDataReady`, `OptimizedStateComponent`, `InteractServer` et `InteractClient` cables, et
ouverture de `PUW_VehicleStorage` par `PopUpDialogClass`. **Son seul defaut est son
identite de persistance** : `SetIdAndGetData(Conv_StringToName(Conv_IntToString(GetBuilderID())))`.

Ce `BuilderID` n est pas durable, et la mesure est sans ambiguite :

- `QBuilder_SaveGame_CreateBuilder` alloue un index neuf a la restauration
  (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:728`) puis l ecrase
  sur l acteur (`:733`). **Le `QBuilder_BuilderID` sauvegarde n est jamais relu** ; le
  champ porte d ailleurs le commentaire `// Not Use`
  (`Public/Struct/QBuilder_Struct_Data.h:341`).
- `QBuilder_ServerBuilder_GetNewIndex` (`Private/QBuilder_SubSystem.cpp:429-441`) rend le
  premier index du pool de recyclage s il n est pas vide, sinon `Server_Builders.Num()`.
  L identite est donc **ordinale** : elle depend de l ordre d iteration de
  `QBuilder_SaveGame_LoadData` et de l etat du pool.

Consequence : des que l ordre ou la taille du tableau sauvegarde change d un cran (un
builder detruit, un builder ajoute, un index recycle), chaque builder suivant se decale et
`SetIdAndGetData` **tombe sur le DataObject d un autre builder**. Le contenu des zones
construites ne survit donc pas de facon fiable a un redemarrage. Le defaut est invisible en
solo continu, ou les indices coincident.

**Ce que cela change pour le chantier** :

1. QStorage ne se contente pas d ajouter une fonctionnalite, **il repare cette chaine** en
   substituant son `FGuid` serveur au `BuilderID` comme cle de `SetIdAndGetData`. C est le
   coeur du perimetre cote construit, et c est tout le perimetre (voir 7.4).
2. **Aucune migration des anciens contenus** (arbitrage RzZz). Une donnee keyee sur un
   entier ordinal non stable ne porte pas d identite reconstituable : la migrer reviendrait
   a rattacher des contenus au hasard. Les enregistrements existants sont traites comme
   orphelins, listes par `qstorage.Orphans`, jamais effaces automatiquement (7.3).
3. **A dire a l equipe et au support** : des plaintes de coffres de base vides apres un
   redemarrage sont l expression de ce bug, pas une regression du chantier. C est un
   argument de priorisation, pas une excuse : la chaine reste cassee tant que J1 a J4 ne
   sont pas livres.

### 4.1 Identite : GUID serveur, ancre serveur, tombstone

Le relecteur data a raison : dans le V1, l identite reelle etait la position, le GUID n en etait qu un alias. Corrige sur trois points.

1. **L ancre n est calculee que par le serveur.** Le client ne la connait pas, ne l envoie pas, ne peut pas la fabriquer (voir 5.1). Cela supprime d un coup : la generation de loot a volonte, la divergence de quantification `FRepMovement` pour une piece QBuilder, et le cas `NM_ListenServer` ou `ServerTransformFull` n est jamais assigne (`Plugins/QBuilder/Source/QBuilder/Private/World/QBuilder_BuilderActor.cpp:176-186`, replique `COND_InitialOnly` ligne 32) et ou les clients reposent leur builder a l identite.
2. **Tombstone obligatoire.** A la destruction d un coffre : ecriture d une pierre tombale persistee (GUID, ancre, `CreatedSeq`), refus de toute resolution d ancre vers un GUID marque detruit, et appel de `DeleteDataObject` / `DeleteDataFromDB` (exposes par `Plugins/DataManager/Content/GameDataManager.uasset`, jamais appeles par le V1) sur le DataObject du coffre, dans la meme transaction. Sans cela, la boucle est triviale : remplir, detruire, ramasser la restitution au sol, reconstruire au meme endroit a moins de 10 cm, recuperer le contenu une seconde fois.

   **Un tombstone ne protege pas d une restauration de sauvegarde.** Le chemin d ecriture
   produit un `BACKUP_` en plus du slot principal, et une restauration admin d un slot
   monde anterieur au tombstone **ressuscite l enregistrement pre-tombstone**, donc rend le
   contenu detruit. Le tombstone rend le rejeu impossible **en vol**, jamais apres un
   retour arriere de fichier. Trois consequences ecrites : les tombstones vivent dans le
   **meme** segment que les enregistrements qu ils annulent (une restauration partielle ne
   peut pas ramener un record sans son tombstone) ; `CreatedSeq` est **monotone par monde**
   et un record dont la `CreatedSeq` est superieure a la sequence courante connue est
   refuse et journalise ; toute restauration de sauvegarde est declaree comme une operation
   qui peut dupliquer des objets, et elle doit etre suivie d un `qstorage.Orphans`. Ce
   n est pas un defaut de QStorage : c est une propriete de toute restauration de fichier,
   et elle est documentee plutot que masquee.
3. **Collision de bucket refusee, pas devinee.** Si deux coffres vivants tombent dans le meme bucket de 10 cm, le second refuse son enregistrement avec `UE_LOG(LogQStorage, Error)` nommant les deux GUID. Un coffre non ouvrable et bruyant vaut mieux qu un coffre qui herite du contenu d un autre.

Options ecartees, pour memoire : la cle de position seule (troncature au centimetre, `Plugins/QNetState/Source/QNetState/Public/OptimizedStateTypes.h:61-67`) et la cle composite QBuilder (melange un `BuilderID` reattribue, collisionne).

### 4.2 Cadence : QStorage possede son ecriture

C est la correction la plus lourde. Le V1 affirmait "zero nouveau chemin disque, segmentation multi-instance heritee gratuitement". L heritage est reel, **la gratuite ne l est pas** :

- `InventoryComponent.ReplicateSaveInventory` -> `SetStringArray` -> `DataSettedCallUpdate` (0,01 s) -> `GameDataManager.DataObjectUpdated` -> `SaveDataToDB` -> `TempDB.WriteTempData` (Delay 1 s + Do Once) -> **deux** `AsyncSaveGameToSlot` (slot puis `BACKUP_`) ;
- `UGameplayStatics::AsyncSaveGameToSlot` appelle `SaveGameToMemory` **de facon synchrone sur le game thread** (`C:/UE5_Share/Engine/Source/Runtime/Engine/Private/GameplayStatics.cpp`) et n asynchronise que l ecriture disque ;
- chaque instance d item est son propre DataObject. Mesure de reference : `Saved/SaveGames/Offline/L_Dev_Claude_Offline.sav` fait 64 Ko pour 5 DataObjects et 121 lignes, soit environ 500 octets par ligne.

Un debounce cote QStorage ne freinait que l index, pas le contenu, parce que le contenu etait ecrit par un chemin BP que QStorage ne possede pas. Corrige :

- l `InventoryComponent` du coffre est mis en **`SetNotPersistentData`** : il ne s auto-persiste pas ;
- **QStorage encode les piles d un coffre dans UNE seule cle tableau** du DataObject de ce coffre, avec le codec `QSTCRATE`, patron `QModuleItemRack` (`Plugins/QModule/Source/QModule/Private/QModule_ItemRack.cpp:22`). On divise le nombre d entrees par 15 a 40 ;
- ecriture **a la fermeture de session** et au flush synchrone de `EndPlay`, jamais par mouvement d objet ;
- l **index est segmente** : une cle DataObject par secteur (grille de region, ou par `QLevel_Asset`), jamais un tableau global unique. Un index monolithique a 21429 ancres represente 2,5 a 3 Mo reencodes a chaque mutation ;
- plafonds durs dans `UQStorage_DevSettings` : nombre maximal de coffres persistes par monde, taille maximale d un segment d index, nombre de slots par coffre, nombre de coffres chauds. Decides avant J4.

**Note de sincerite** : cela n annule pas le cout, cela le borne. Le slot monde reste reecrit en entier, deux fois, sur le game thread, a chaque flush. C est pourquoi le banc de mesure passe de J7 a J1 (voir 9).

### 4.2bis ARBITRAGE DU 2026-08-22 : le contenu sort du slot monde

Le banc a tranche (voir 9.1). RzZz a choisi la voie **(B)** : le contenu des coffres
quitte le slot monde partage. Tout le 4.2 ci-dessus reste vrai comme **description du
chemin herite** et comme justification, mais il n est plus le chemin d ecriture de
QStorage. Ce qui suit le remplace.

**Le probleme mesure, en une phrase** : la segmentation d index bornait le re-encodage,
mais `SaveGameToMemory` porte sur le slot **entier** a chaque flush, sur le game thread,
et rien ne bornait ce poste. 10,83 ms a 1000 conteneurs, 62,80 ms a 5000.

**Le nouveau chemin, et pourquoi il coute zero milliseconde de game thread.**

Les donnees de QStorage ne sont **pas** des UObject : ce sont des `FQST_ItemStack` et des
`FQST_ContainerRecord`, c est-a-dire des `FName`, des `FString` et des entiers. Rien
n oblige a passer par `USaveGame`, et c est precisement `USaveGame` qui impose
`SaveGameToMemory` sur le game thread. Donc :

1. **sur le game thread** : on **copie** les structures du segment sale dans un paquet de
   travail. C est une copie memoire de `TArray<FQST_...>`, sans encodage, sans
   serialisation, sans reflexion ;
2. **sur un thread de fond** (`AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, ...)`)
   : on encode le codec `QSTCRATE` **et** on ecrit le fichier. Les **deux** postes mesures
   par le banc partent du game thread, y compris l encodage, qui etait le plus cher des
   deux (130 ms contre 31 ms a 5000) ;
3. l ecriture est un `FFileHelper::SaveArrayToFile` sur un fichier **propre a QStorage**,
   jamais le slot partage : sa taille ne depend plus de ce que le reste du jeu y met.

**Le piege a ne pas recopier de QBuilder.** `QBuilder_AsyncSave`
(`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:751-760`) fait la bonne
chose (tout le travail sur un thread de fond) mais capture un `UQBuilder_Data_WorldSaveGame`
cree par `NewObject` et **non enracine**, consomme sur ce thread : rien ne le protege du
ramasse-miettes pendant l ecriture. Nous ne reproduisons pas ce montage. Comme notre
paquet de travail ne contient **aucun UObject**, la question ne se pose pas : c est le
deuxieme argument, apres le cout, pour ne pas passer par `USaveGame`.
(Ce defaut de QBuilder est signale comme tache separee, hors perimetre de ce chantier.)

**Ce que (B) nous fait perdre, et qu il faut donc refaire nous-memes.**

- **La segmentation par port serveur**, qui venait gratuitement de `GameDataManager`. Le
  chemin devient explicite : `Saved/SaveGames/QStorage/Port<Port>/<Monde>/Segment_<X>_<Y>.qst`.
  La limite du 4.6 reste vraie et inchangee : en mode hors ligne il n y a pas de port,
  donc pas de segmentation, et deux instances sur la meme map partagent le fichier.
- **Le miroir `BACKUP_`**. Nous ecrivons donc en deux temps : fichier temporaire, puis
  remplacement atomique, et conservation d **une** generation precedente. Un remplacement
  atomique vaut mieux qu un miroir ecrit une seconde plus tard, qui recopiait l etat
  fautif (4.3).
- **Le chargement au demarrage**, qui devient notre travail : lecture paresseuse par
  segment, a la premiere resolution d ancre dans ce secteur, jamais un chargement global.

**Ce que (B) nous rend.**

- Le plafond `MaxPersistedContainersPerWorld` cesse d etre une contrainte de frame. Il
  reste comme garde-fou memoire et disque, et il est reevalue quand le nouveau chemin sera
  mesure, avec le meme banc.
- Le format nous appartient de bout en bout, donc le `SchemaVersion` et l echelle de
  migration du 4.4 s appliquent a un fichier que nous sommes seuls a ecrire. La reserve du
  4.4 sur `E_DataDivider` et le payload d item, elle, reste entiere : `EncodedData` reste
  produit par `Obj_ItemInstance`.

**Ce qui ne change pas** : la verite du contenu reste l `InventoryComponent` cote serveur,
l identite reste le `FGuid` avec ses tombstones, la cadence reste "a la fermeture de
session, jamais par mouvement d objet", et le toggle `bFlushAfterTransfer` du 4.3 reste
`False` par defaut, mais devient beaucoup moins cher a activer.

**A mesurer avant d appeler cela resolu** : le nouveau chemin doit repasser au banc, avec
la meme courbe, et prouver que le cout game thread se reduit bien a la copie. Tant que
cette courbe n existe pas, (B) est une intention, pas un resultat.

### 4.3 Atomicite d un transfert

`TryMoveItemToAnotherInventory` retire de la source puis ajoute a la cible : deux DataObjects distincts, coalesces par le `Delay 1 s + Do Once` de `WriteTempData`. Si les deux mutations tombent de part et d autre d un cycle et que le process meurt entre les deux, l objet est retire du joueur et jamais ajoute au coffre. Le `BACKUP_` n aide pas : il est ecrit une seconde apres le slot principal, il miroite l etat fautif.

**Le journal d intention (WAL) du V2 est supprime. Arbitrage RzZz, et il a raison** : ce
journal vivait dans le segment persiste **a la fermeture de session**, donc il n etait pas
durable au moment precis du crash qu il pretendait couvrir. Et le rendre durable imposait
un flush par mutation, ce qui detruisait la cadence du 4.2. Un WAL non durable est pire
qu absent : il donne une garantie que le systeme ne tient pas.

Ce qui le remplace, en trois pieces :

1. **Une fenetre de perte bornee et assumee.** Perimetre exact : les transferts joueur vers
   coffre effectues pendant une session d ouverture qui se termine par un crash du process.
   Tout ce qui est ferme proprement est ecrit (flush de fermeture de session et flush
   synchrone d `EndPlay`, 4.2). La borne est donc la duree d une session d ouverture, pas
   la duree de la partie. Elle est **ecrite dans la doc du plugin**, journalisee au
   demarrage suivant (une session ouverte non fermee est detectable : elle laisse un
   marqueur d ouverture dans le segment), et remontee en Warning nommant le GUID.
2. **Un toggle `.ini` de flush post-transfert, rate-limite**
   (`bFlushAfterTransfer`, `FlushAfterTransferMinIntervalSeconds`), **False par defaut**.
   Il ecrit le segment du coffre apres un transfert, au plus une fois par intervalle. Il
   n est **active que si le banc J1 montre que le cout passe** : c est precisement la
   courbe que le banc mesure. Ce n est pas une promesse, c est une option instrumentee.
3. **L alternative meme-cle reste en piste pour J4** : faire aboutir les deux cotes du
   transfert dans la **meme** cle serialisee rend le transfert atomique par construction.
   Elle ne couvre que coffre vers coffre (les deux cotes appartiennent a QStorage), jamais
   joueur vers coffre, dont un cote appartient au `PersistentDataComponent` du joueur.

**Ce que nous ne promettons pas** : l atomicite d un transfert joueur vers coffre face a un
arret brutal du process. Aucun mecanisme a notre portee ne la donne sans payer une ecriture
par mutation sur le chemin que le 4.2 vient precisement de degager.

### 4.4 Versionnement

- Entier `SchemaVersion`, pas un booleen (le `MergedData1` de `TempDB.MergeOldDataKeys` ne permet qu une migration dans toute la vie du projet).
- **Absent, illisible et valeur sont trois etats distincts.** Le decodage `DecodeParserToData` de `DataObject` a un `Switch on E_DataDivider` **sans branche Default** : il saute silencieusement toute ligne inconnue, donc un index tronque rend "version 0" au lieu de "je suis casse". Regle : version lue par une cle chaine ; absent ou illisible donne un chargement en **lecture seule**, `UE_LOG(Error)`, coffre non ouvrable. Jamais de migration sur une donnee dont la version n a pas ete lue avec certitude.
- Version stampee **par ligne** en plus de l en-tete (le codec le permet : champ additionnel en fin de ligne). Chaque marche `Upgrade_N_to_N1` est nommee, **idempotente**, testable a froid sur une chaine.
- Version superieure a la version courante (retour arriere de build) : lecture seule, jamais d ecrasement.
- On ne reecrit jamais l index dans le meme tick que la lecture qui a declenche la migration.
- On ecrit aussi `Qanga_Version` (`Config/DefaultGame.ini:241`) au moment de l ecriture. Aucun systeme du projet ne le fait aujourd hui.
- **Limite assumee et ecrite noir sur blanc** : QStorage versionne son propre format. Le payload d item encode par `Obj_ItemInstance` (separateur exotique) et l enum d octets `E_DataDivider` sont hors de son controle. La promesse "persistant a travers les mises a jour" couvre le conteneur, l index et l identite ; elle ne peut pas couvrir un retypage de `E_DataDivider`.
- Le separateur du codec `QSTCRATE` doit etre produit par **sequence d echappement ASCII** dans les sources C++ (regle projet : sources en ASCII pur), et choisi pour ne collisionner avec aucun codec existant.

### 4.5 Item disparu de DA_AllRef : la quarantaine doit etre EN AMONT

Le V1 promettait une quarantaine qu il n avait pas les moyens d appliquer : `Obj_ItemInstance.LoadFromDataObject` pose `InvalidItem = true` puis appelle `DestroyDeleteItemInstanceData`, ce qui **efface l enregistrement en base**, et ce chemin s execute quand l `InventoryComponent` du coffre charge ses instances, donc avant que QStorage regarde quoi que ce soit. Corrige :

1. QStorage lit **d abord** sa propre liste de piles (avec `EncodedData` complet, pas seulement des cles) ;
2. il verifie chaque `ItemKey` contre `DA_AllRef.ItemKey:DAItem` (`Content/Systems/References/DA_AllRef.uasset`) ;
3. il ne confie a l `InventoryComponent` que les cles resolues. Les autres restent en quarantaine dans son propre segment, **jamais transmises a `Obj_ItemInstance`**, donc jamais detruites ;
4. UI : slot grise, non deplacable, libelle localise. Si l asset revient, le coffre reprend la pile au chargement suivant ;
5. le test QATS de gel du registre passe de J7 a **J1** et tourne en continu.

### 4.6 Segmentation multi-instance : condition de validite

Vraie en dedie (`Port<Port>/<Gamemode>_Port_<Port>`, `Plugins/DataManager/Content/GameDataManager.uasset`). **Fausse en mode `Offline/<Map>_Offline`** : aucune segmentation, deux listen servers sur la meme machine et la meme map partagent le fichier, dernier ecrivain gagne. Idem si deux instances sont lancees avec le meme `-worldid`. QStorage n aggrave pas la situation mais **ne doit pas pretendre la resoudre** : la limite est documentee dans la doc du plugin et signalee au boot par un Warning quand deux conditions de collision sont detectables.

---

## 5. Reseau

> **Consequence de la lecture B, ajoutee en v3 : cette section ne decrit plus qu une des
> deux poses.** En lecture B le coffre construit **est** le builder, et
> `AQBuilder_BuilderActor` porte `bReplicates = true`
> (`Plugins/QBuilder/Source/QBuilder/Private/World/QBuilder_BuilderActor.cpp:22`,
> `GetLifetimeReplicatedProps` a `:29`). Le coffre construit se replique donc **deja**, et
> sa chaine inventaire, sa replication d objets et son UI `PUW_VehicleStorage` sont **en
> production**. On ne les remplace pas : on leur donne une identite stable (4.0, 7.4).
>
> Les deux poses n ont donc pas le meme modele reseau, et c est assume :
>
> | | Coffre construit (lecture B) | Coffre de niveau |
> |---|---|---|
> | Acteur | `AQBuilder_BuilderActor`, **replique** | `AQStorage_Container_Actor`, **non replique** |
> | Contenu vers le client | chaine existante (replication d objets) | snapshot et deltas, 5.1 |
> | UI | `PUW_VehicleStorage`, deja cablee | 5.4 et section 8 |
> | Ce que QStorage apporte | l identite stable uniquement | tout |
> | Nombre attendu | borne par le nombre de zones de construction | jusqu a 21429 ancres CONTAINER |
>
> Le cout de replication qui justifiait le "jamais replique" est paye **aujourd hui** par
> les builders existants, sans QStorage : nous n aggravons rien. Il reste la seule bonne
> raison de ne pas repliquer les coffres de niveau, dont le nombre est d un autre ordre.
> **Tout ce qui suit s applique au coffre de niveau**, et au coffre construit **seulement
> si** une mesure montre que la chaine existante ne tient pas la charge, ce qui serait un
> chantier distinct sur QBuilder et non sur QStorage.

**Le coffre de niveau ne se replique jamais.** `bReplicates = false`.

Correction d argument : le V1 justifiait cela pour le coffre **construit** par le nom de paquet `/Temp.../_LevelInstance_<compteur statique>`. C est faux pour ce cas : cet argument ne vaut que pour un acteur pose dans un sous-niveau streame (`Engine/Private/LevelStreaming.cpp:2780-2795`), et `AQModule_WorkbenchActor` prouve qu un acteur player-built spawne au runtime se replique correctement. **Le vrai argument pour le coffre QBuilder est le cout** : aucun ReplicationGraph n existe dans le projet (0 `ReplicationDriverClassName` dans `Config/`), la pertinence est la boucle naive du NetDriver, et 21429 ancres CONTAINER sont deja cataloguees. Pour le coffre de niveau, l argument `/Temp` reste valable et suffisant.

### 5.1 Ouverture : le client ne nomme jamais un coffre

```cpp
UFUNCTION(Server, Reliable, WithValidation)
void QST_ToServer_OpenNearest();                       // NO position, NO key, NO object ref

UFUNCTION(Server, Reliable, WithValidation)
void QST_ToServer_Close(FGuid ContainerGuid);

// Direction NAMES BOTH ENDPOINTS: it is the only thing that says which inventory
// SourceSlot indexes and which one TargetSlot indexes. Without it _Validate cannot
// bound-check either index, and the server cannot tell a deposit from a withdrawal.
UENUM(BlueprintType)
enum class EQST_MoveDirection : uint8
{
    PlayerToContainer = 0,   // SourceSlot indexes the pawn inventory, TargetSlot the container
    ContainerToPlayer = 1,   // SourceSlot indexes the container, TargetSlot the pawn inventory
    WithinContainer   = 2    // both index the container
};

UFUNCTION(Server, Reliable, WithValidation)
void QST_ToServer_MoveSlot(FGuid ContainerGuid, uint32 ExpectedContentVersion,
                           EQST_MoveDirection Direction,
                           int32 SourceSlot, const FString& ExpectedSourceInstanceId,
                           int32 TargetSlot, int32 Quantity);

UFUNCTION(Client, Reliable)
void QST_ToClient_Snapshot(FGuid ContainerGuid, uint32 ContentVersion,
                           int32 ChunkIndex, int32 ChunkCount, const TArray<FQST_ItemStack>& Stacks);

UFUNCTION(Client, Reliable)
void QST_ToClient_SlotDelta(FGuid ContainerGuid, uint32 NewContentVersion, const FQST_ItemStack& Stack);

UFUNCTION(Client, Reliable)
void QST_ToClient_Closed(FGuid ContainerGuid, EQST_Refusal Reason);   // container gone / kicked

UFUNCTION(Client, Reliable)
void QST_ToClient_Refused(FGuid ContainerGuid, EQST_Refusal Reason);
```

**L inventaire source est nomme par `Direction`, et par rien d autre.** Le V2 laissait
`uint8 Direction` sans semantique ecrite, ce qui rendait `_Validate` ambigu : un index de
slot ne veut rien dire tant qu on ne sait pas quel inventaire il indexe, et les deux n ont
ni la meme taille ni le meme proprietaire. `Direction` est donc un enum a trois valeurs qui
**designe les deux extremites a la fois** ; le serveur en deduit quel conteneur borner,
lequel verifier en propriete (`PawnOwnsInstance` ne s applique qu au cote joueur), et quel
sens de flux journaliser. Un `Direction` hors enum est un refus `Unresolved`, pas un
defaut silencieux sur une branche.

Le serveur maintient `TMap<FQST_AnchorKey, TWeakObjectPtr<AQStorage_Container_Actor>>` alimente par le `BeginPlay` des acteurs **serveur** ; a l ouverture il choisit lui-meme le coffre le plus proche du pawn dans `MaxInteractDistanceCm` et renvoie le `FGuid`. Le fill ne se declenche **que** pour un record cree par le `BeginPlay` d un acteur serveur reel, jamais par un RPC client. Patron existant : `Lib_OptimizedState::SendOptimizedActorInteraction` aboutit sur `InteractServer(APlayerController*, bool, FName)` (`Plugins/QNetState/Source/QNetState/Private/RequesterOptimizedState.cpp:458-474`), qui prend le PC et resout l acteur serveur lui-meme. On se branche sur ce canal plutot que d en ouvrir un nouveau.

Tous les RPC sont portes par `UQStorage_Client_Component` **sur le PlayerController** : le coffre a pour Owner le builder ou rien, donc n a aucune connexion proprietaire, et un RPC serveur appele sur lui serait rejete en silence.

### 5.2 Validation serveur

1. **Distance revalidee a chaque mutation**, pas seulement a l ouverture.
2. **Resolution serveur de l instance** : le client envoie un index de slot **et** l `InstanceId` qu il croit y voir. Le serveur valide la propriete (`PawnOwnsInstance`, patron durci deja ecrit) **puis** compare a l index, et refuse en cas de desaccord. Le V1 bannissait l InstanceId pour bloquer l injection ; c etait une erreur d analyse : la securite vient de la **validation de propriete**, pas de l absence d identifiant. Sans cela, une recompense de quete qui compacte le sac pendant le drag fait deposer l objet voisin, sans aucune erreur.
3. **Rebase, pas refus systematique** : si le slot vise est inchange cote serveur, la mutation est appliquee malgre une version obsolete. Refus reserve au conflit reel de slot. Sans cela, deux joueurs qui vident un coffre de 40 slots produisent des dizaines de snapshots complets sur un canal fiable, portant des `FString EncodedData` : auto-attaque en bande passante.
4. **Plafond d un snapshot par fenetre de 250 ms et par session.**
5. **Snapshot fige avant chunking**, deltas mis en file jusqu au dernier chunk. Sinon un delta emis entre le chunk 1 et le chunk 3 est ecrase par le chunk 3, qui porte l etat d avant : objet fantome, puis refus, puis nouveau snapshot.
6. **Chunking** (`SnapshotChunkSize`, defaut 16 piles). Le canal fiable non plafonne de `OC_Reply` (`Plugins/QNetState/Source/QNetState/Private/RequesterOptimizedState.cpp:304-308`) est un precedent a ne pas reproduire : un depassement de buffer fiable deconnecte le client.
7. **Limiteur de debit** par PlayerController, plafond de sessions ouvertes simultanees.
8. **`_Validate` reellement ecrits** : quantite negative, index hors bornes, source egale cible, GUID inconnu.
9. **Transaction unique** via `Lib_Inventory.TryMoveItemToAnotherInventory` cote serveur, avec test de place **prealable** : `AddItemToInventory` jette l item au sol quand la cible est pleine, et `TryMove` exige un slot libre meme pour un empilement.

### 5.3 Etat leger : correction de chiffre

Le V1 annoncait "1 a 2 octets". Mesure contraire sur le port C++ : tous les setters ont `bServerMulticast = true` **par defaut** (`Plugins/QNetState/Source/QNetState/Public/OptimizedStateComponent.h:58-84`), `ServerMulticastState` empile dans le requester (`Private/OptimizedStateComponent.cpp:370-388`) qui appelle `MC_Update`, declare `UFUNCTION(NetMulticast, Reliable)` (`Public/RequesterOptimizedState.h:96`), et en serveur dedie le manager resout le requester sur le GameState (`Private/OptimizedStateManager.cpp:54-60`), donc always relevant. `FOptimizedStateKey` n a aucun `NetSerialize` : 24 octets bruts plus une `FName`. Un `SetBool` coute donc un RPC fiable vers 500 connexions, sans filtre de distance.

Retenu : **`bServerMulticast = false` explicite sur tout appel**, lecture par pull uniquement, **garde de distance sur le pull** (sinon un client enumere a distance quels coffres sont pleins).

**Throttle du pull, ecrit noir sur blanc.** Un pull sans plafond est un canal de requetes
serveur ouvert que le client cadence lui-meme : c est le meme defaut que celui qu on vient
de corriger sur les snapshots. Regles chiffrees, toutes en `.ini` :

- **plafond par joueur** : `PullMaxPerSecond`, defaut 4 requetes par seconde et par
  PlayerController, compteur en fenetre glissante ; au-dela, refus `RateLimited`, aucune
  file d attente, aucun travail serveur ;
- **plafond par coffre et par joueur** : une reponse au plus toutes les
  `PullMinIntervalSecondsPerContainer` secondes (defaut 1,0). Deux demandes rapprochees sur
  le meme coffre rendent la meme reponse memoisee, sans relecture d etat ;
- **garde de distance evaluee avant tout travail**, sur la meme constante que l ouverture
  (`MaxInteractDistanceCm`), et **revalidee**, jamais heritee d une ouverture precedente ;
- **le pull ne repond que sur un coffre deja enregistre cote serveur** : il ne declenche ni
  chargement de niveau, ni attachement de composant, ni remplissage. Un coffre non charge
  rend `NotReady`. Sans cette regle, un client balaye la carte et fait charger le monde au
  serveur, coffre par coffre ;
- **compteurs exposes** par `qstorage.Status` (pulls servis, refuses, memoises) pour que le
  reglage se fasse sur une mesure et non sur une intuition.

**Critique partiellement ecartee** : "supprimer `bOpened` car un booleen n a pas de semantique pour N ouvreurs" est **non fonde en tant que tel**. Le bit reste utile pour le visuel a distance ; ce qu il faut, c est qu il soit pilote par le **compteur serveur d ouvreurs** de la session (passe a false quand le compteur retombe a zero), pas par la fermeture du premier joueur. Je conserve `bOpened` avec cette semantique explicite. Le reste de la critique (multicast, cout) est integralement retenu.

### 5.4 Client : pas de reutilisation aveugle de l InventoryComponent

Le V1 affirmait qu on remplirait un `InventoryComponent` transitoire cote client "format identique a `F_ItemRequestReply`, donc reutilisation du chemin BP existant". **Rien ne l etaye** : `GameDataManager.BeginPlay` est gate `Is Server`, `InventoryComponent.BeginPlay` reste a `IsInventoryReady = false` sans `PersistentDataComponent` pret, `W_Inventory` consomme des `Obj_ItemInstance` (portes par `DnD_ItemInstance`) et non des `FQST_ItemStack`, et aucun chemin BP mesure ne construit un `Obj_ItemInstance` cote client. **En v3, cette question ne concerne plus le coffre construit** : en lecture B il est
replique et son UI fonctionne deja en prod (voir l encadre en tete de section 5). Elle ne
porte donc que sur le **coffre de niveau**, ce qui la deplace de J0 vers **J2**, ou elle
est mesuree au moment ou l on ecrit le panneau. Deux branches, ecrites d avance :

- **branche A** (l inventaire client peut devenir pret via `SetNotPersistentData`) : le wrapper alimente un `InventoryComponent` transitoire, `W_Inventory` est reutilise tel quel, section 8 inchangee ;
- **branche B** (il ne peut pas) : le panneau coffre est alimente par un **modele pur** construit depuis les `FQST_ItemStack`, `W_Inventory` **n est pas reutilisable tel quel** pour le panneau coffre, et la section 8 est reecrite (le panneau joueur, lui, reste `W_Inventory`, il pointe un inventaire local deja pret).

Cout de la branche B : un panneau de slots dedie, plus un chemin de drag and drop qui ne porte pas d `Obj_ItemInstance`. C est du travail, pas un blocage d architecture.

### 5.5 Listen server

- Garde `HasAuthority()` en tete de **tout** le chemin client d ajout et de destruction de composants. Sur un listen server la copie locale **est** la copie serveur : `GetInventoryComponent(Self)` est un `FindComponentByClass`, et une fermeture d UI pourrait detruire l inventaire faisant autorite. Perte de coffre chez l hote en une fermeture de fenetre.
- **Double spawn a verifier, pas a supposer** : en `ActorAlwaysOnServer` hors standalone, le serveur spawne l acteur **et** ajoute l ordre dans `ToClients` (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:1259-1271`), envoye a tous les clients enregistres, et l hote s enregistre par le meme chemin. Le branchement est sans ambiguite mais le comportement effectif pour l hote n a pas ete mesure : INCONNU 3, bloquant pour J6. Si confirme, exclure l hote de la liste.
- Chaque jalon reseau est valide **en listen ET en dedie**, jamais en PIE standalone (aucune quantification reseau, aucune evaluation de pertinence).

### 5.6 Mort du coffre sous le joueur

Trois causes documentees : `IsOff` detruit les acteurs d instance quand le **dernier** client se desabonne du builder (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:1780-1809`), la boucle standalone decharge a 5 km (`Private/QBuilder_Client.cpp:902`), le streamer QLevel decharge le sous-niveau. Retenu : **flush synchrone dans `EndPlay` avant destruction**, invalidation explicite de toutes les sessions du GUID, `QST_ToClient_Closed(Gone)`, et un test QATS dedie au cycle IsOn/IsOff **avec une UI ouverte**.

---

## 6. Le cas QLevel (coffres poses en editeur, pre-remplis)

Le probleme suppose (sous-niveau client-only) n existe pas : `UQLevel_SubSystem` tourne aussi en serveur dedie, avec des budgets `*_OnDedicated` (`Config/DefaultGame.ini:353-381`) et une iteration de tous les PlayerController (`Plugins/QLevel/Source/QLevel/Private/QLevel_SubSystem.cpp:1255-1270`). L acteur pose existe bien cote serveur.

- **Autorite** : le subsystem serveur. La copie de chaque client s enregistre en lecture seule.
- **Limite assumee** : le serveur ne charge que ce qu un joueur regarde. Un coffre de salle de boss n a aucune existence serveur tant que personne n approche, ce qui est acceptable pour un conteneur. **Plan B** : `QLevel_RequestLocationLoad` plus `QLevel_IsLocationReady` (`Plugins/QLevel/Source/QLevel/Public/QLevel_SubSystem.h:155-190`), deja utilises par QAI, en sachant que la requete **expire** si elle n est pas rafraichie et qu elle n est pas BlueprintCallable en l etat.
- **Contenu initial** : `FQST_FillSpec` sur le preset. Table `DA_Loot` tiree avec `RollWeightedItem` (seul tirage reellement pondere du projet ; `GetSeededItemByLocation` ignore les poids et sature en int32 a l echelle planetaire), plus `GuaranteedStacks` pour une salle de boss.
- **Determinisme** : tirage **une seule fois, cote serveur, paresseusement a la premiere ouverture**, graine `HashWorldLocation`, puis persiste. Le client ne tire jamais.
- **`RestockSeconds`** : un restock ne peut se declencher que quand le coffre est charge cote serveur, donc a la premiere ouverture posterieure a l echeance. C est un rechargement **paresseux**, pas un timer monde. Le V1 laissait ce point implicite ; il est desormais ecrit.
- **Le rechargement est une propriete du profil, pas du coffre** (arbitrage RzZz). C est
  `FillProfileTag` qui decide : un profil de loot commun porte un `RestockSeconds` non nul
  et se recharge paresseusement, un profil de recompense (les `GuaranteedStacks` d une
  salle de boss) porte `RestockSeconds = 0` et est **definitivement pille, par serveur,
  pour tous les joueurs**. Un profil mixte est refuse a la validation du DataAsset : un
  coffre de boss qui repousse ses garanties est une machine a farmer, et le refuser au
  moment de l edition coute moins cher que de le decouvrir en prod.
- **Pieges d edition, non negociables** : poser dans `LevelData.Level`, puis **regenerer l optimise** (c est `OptimisedLevel` qui est streame, et le cook ecrase `Level` par `OptimisedLevel`). L acteur doit porter le tag **`QL_ExcludeMerge`**, sans quoi la passe d optimisation le **detruit** des qu un `UStaticMeshComponent` est absorbe dans un ISM (`Plugins/QLevelEditor/Source/QLevelEditor/Private/QLevel_Editor_OptimisationLibrary.cpp:611-645`, `:809-822`). `ExcludeMergeClass` est un tableau `.ini` qui **remplace** le defaut C++ (`Config/DefaultEditor.ini:39`, une seule entree `VehicleSpawner_C`) : le tag acteur est plus sur.
- Le coffre ne doit **jamais** etre enregistre dans la map de nettoyage a l unload de `QModuleLoot` (elle detruit ce qu elle a spawne).

---

## 7. Integration QBuilder

### 7.1 Chaine catalogue

Procedure a 5 etapes, **marquee SUPPOSE dans le dossier** (deduite par `rg -a` sur des .uasset, jamais verifiee au pont) : creer le `QA_*` (`UQBuilder_Data_ActorData`, `ActorClass = BP_QStorage_Container_Build`), creer le `QDD_*`, inscrire sous un ID int32 unique dans `Content/Systems/QBuilder/Data/QBuilder_Qanga_ActorData.uasset`, referencer dans le `QTS_` du set, puis **`DB_Optimise` dans `Content/Systems/QBuilder/Tools/QBuilder_DataTools.uasset`** (sans quoi la piece est invisible du runtime, sans erreur ni log).

**Set d accueil : ICLAB** (arbitrage RzZz). Le segment median des assets QBuilder est le
set, et "QANGA" n en etait pas un.

**On ne cree pas une piece parallele : on rebranche celle qui existe.**
`Content/Systems/QBuilder/Data/ICLAB/Actor/Storage/QA_ICLAB_StorageBase` pointe deja vers
`BP_Storage_ForBuild_C` avec exactement la configuration du 7.2, **lue au pont par RzZz le
2026-08-21** : `ActorIsReplicated = False`, `ActorAlwaysOnServer = True`,
`ActorPersistantOnServer = False`. Cela **ferme l INCONNU 11** et confirme le 7.2 sur une
mesure et non sur une deduction.

Mais `BP_Storage_ForBuild` **ne reference pas d `InventoryComponent`** : c est une coquille
vide, un coffre constructible qui n a jamais eu de contenu propre. Deux consequences :

- l entree de catalogue est **remplacee ou rebranchee**, jamais doublee. Ajouter un
  `QA_*` parallele laisserait deux coffres visibles dans le menu de construction, dont un
  inerte, et rendrait le diagnostic illisible ;
- la procedure a 5 etapes ci-dessus n est donc pas a executer en entier : l entree, le
  `QDD_`, l inscription et le `QTS_` existent. **Reste le rebranchement de `ActorClass`
  (ou le peuplement de `BP_Storage_ForBuild`) et le `DB_Optimise`**, qui reste obligatoire
  et reste le piege : sans lui la piece est invisible du runtime, sans erreur ni log.

Le nom final de l asset est fixe apres l arbitrage de la lecture B (7.4), puisque cette
lecture peut supprimer le besoin d une piece separee. La procedure a 5 etapes reste
**marquee SUPPOSE** pour les etapes non encore exercees (deduite par `rg -a`, jamais
verifiee au pont) : elle est validee au pont avant toute ecriture de catalogue.

### 7.2 Regles de spawn

`ActorIsReplicated = false`, `ActorAlwaysOnServer = true`, **`ActorPersistantOnServer = false`**. Ce dernier point est vital : malgre son nom, `true` exclut l acteur de `IsOn` **et** de `IsOff` (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:1761` et `:1794`), donc il n est jamais respawne apres un redemarrage.

Contraintes heritees : la piece reste a moins de 327 m du builder (transform relatif borne int16), et l acteur est detruit puis recree a chaque cycle IsOn/IsOff. Le modele y survit : rien de durable ne vit dans l acteur.

**A corriger avant J6, hors perimetre fonctionnel mais dans le chemin** : le crash latent `Builders.Find` deferences avant test de nullite dans `QBuilder_Resource_Price_Compute` et `_Refund` (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:2190-2193` et `:2244-2247`), qui sera traverse plus souvent des que le coffre entre au catalogue.

### 7.3 Divergence de cadence entre l existence et le contenu

Fait nouveau, integralement retenu : l existence d un coffre construit vit dans `Saved/SaveGames/QBuilder_SaveGame_N.sav` (autosave **600 s**, rotation 4 slots, `Plugins/QBuilder/Source/QBuilder/Public/QBuilder_DevSettings.h:29-31`), son contenu vit dans le slot monde (cadence de l ordre de la seconde). Deux pannes :

- **perte seche** : coffre construit a T=0, rempli a T=30, crash a T=200 avant tout autosave QBuilder. Au redemarrage le contenu existe en base et l acteur n existe plus : les items sont inatteignables ;
- **duplication** : restauration admin d un `Build_2` vieux de 40 minutes en gardant le slot monde courant.

Regle de reconciliation ecrite, **non negociable** : un enregistrement QStorage dont l ancre ne produit aucun acteur **n est jamais efface automatiquement**. Il est conserve, journalise en Warning, liste par `qstorage.Orphans`, et recuperable par commande admin. Une purge n a lieu que sur demande explicite, avec liste affichee avant execution, et TTL configurable en nombre de demarrages sans resolution.

**Option (b) tranchee par RzZz** : on accepte la fenetre, avec la reconciliation ci-dessus
et la recuperation admin. L autosave a 600 s avec rotation est verifie
(`Plugins/QBuilder/Source/QBuilder/Public/QBuilder_DevSettings.h:31`,
`QBuilder_AutoSaveGame_Time = 600.0f`).

Le banc J1 mesure en plus le cout d un `QBuilder_SaveData` force. **S il passe**, il est
adopte sur les **deux seuls evenements rares** : la pose et la destruction d un coffre.
**Jamais sur un mouvement d objet**, ce qui reintroduirait exactement le probleme que le
4.2 vient de resoudre.

**Correction technique au passage (v3)** : le profil de cout de QBuilder n est pas celui
qu on croyait. `QBuilder_AsyncSave` execute `SaveGameToSlot` **entierement dans un
`AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask)`**
(`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:751-760`), donc la
serialisation en memoire **n est pas** sur le game thread, contrairement au chemin
`AsyncSaveGameToSlot` du 4.2. Le cout game thread de QBuilder est la **collecte** :
`QBuilder_SaveGame_CreateData` recopie `ISM_DATA` et `ACTOR_DATA` de **tous** les builders
(`:710-713`, declaree `Public/QBuilder_SubSystem.h:212`). Le banc doit donc instrumenter
**la collecte**, pas `SaveGameToMemory` : ce sont deux profils differents, et les
confondre ferait mesurer la mauvaise chose. Les trois etages sont mesurables separement,
tous exposes publiquement : `QBuilder_SaveGame_CreateData` (collecte, game thread),
`QBuilder_SaveData_SerializeData` et `QBuilder_SaveData_CompressData` (Gzip), ces deux
derniers hors game thread dans le chemin reel.

**Dette signalee, hors perimetre** : le lambda de `QBuilder_AsyncSave` capture un
`UQBuilder_Data_WorldSaveGame` cree par `NewObject` et non enracine, consomme sur un thread
de fond. Rien ne le protege du GC pendant l ecriture. Tache separee, a ne pas traiter dans
ce chantier.

### 7.4 La regle "premier objet a construire pour pouvoir construire"

**LECTURE B, TRANCHEE PAR RzZz. Definitif.**

Le mecanisme de gating n existe pas dans QBuilder, qui ne connait qu une autorisation
(proprietaire ou `Allowed_By_ID`, `Private/QBuilder_SubSystem.cpp:380`) et un prix. Deux
lectures etaient possibles :

- ~~Lecture A, litterale : le coffre est une piece QBuilder qu il faut construire en
  premier.~~ **Ecartee** : logiquement circulaire, et elle exigeait un gate serveur additif
  pour un benefice nul par rapport a B.
- **Lecture B, retenue** : poser le coffre **cree** la zone de construction (chemin
  existant `QBuilder_Builder_ClientCreateBuilderActorWithBuild`). La regle devient
  **structurelle** : il n y a rien a garder puisqu on ne peut rien construire sans zone, et
  la zone n existe que si le coffre est pose.

**Ce que la lecture B fait au perimetre.** Le coffre **est** le builder, et
`Content/Systems/QBuilder/Actor/QBuilder_Builder_Actor_BP` porte deja toute la chaine :
`InventoryComponent`, `PersistentDataComponent` binde sur `OnDataReady`,
`OptimizedStateComponent`, `InteractServer` et `InteractClient` cables, et l ouverture de
`PUW_VehicleStorage` par `PopUpDialogClass`. **Le cablage existe et on le garde tel quel.**

Le perimetre de QStorage cote construit se reduit donc a une seule chose, et c est
exactement le defaut demontre au 4.0 : **fournir l identite stable**. Concretement :

1. substituer le `FGuid` serveur de QStorage au `BuilderID` comme cle de `SetIdAndGetData`,
   avec l index segmente et les tombstones qui vont avec ;
2. conserver le reste du cablage existant sans y toucher ;
3. ajouter le gate structurel, qui ne coute presque rien en lecture B.

**J6 devient marginal** : il n y a plus de nouvelle piece a faire entrer au catalogue, il y
a une cle de persistance a remplacer. Les sections 2, 3 et 5 restent valables telles
quelles pour le cas **coffre de niveau** (7.4 ne concerne que le cas construit), et le
modele de donnees comme le modele reseau sont partages entre les deux poses, ce qui etait
l objectif d origine.

**Le garde-fou serveur reste un point d extension additif** dans QBuilder (delegue du type
`QBuilder_OnCanAddInstance(BuilderID, DataID, ClientID) -> bool` consulte dans
`QBuilder_ServerData_Actor_AddInstances`), sans renommage ni changement de signature.
Nous n ajoutons rien d autre a QBuilder que des points d extension additifs : c est la
condition pour ne casser aucun appelant Blueprint existant.

**Precision mesuree en v3 (le chemin est deja instrumente, mais il n y a pas de gate).**
Depuis le 2026-08-22, QBuilder depend deja de QModule et consulte le mur cyborg du joueur
sur le chemin serveur de placement : `UQBuilder_SubSystem::QBuilder_ActingBuildStat`
(`Private/QBuilder_SubSystem.cpp:2209`) lit `QMOD_GetStat` sur le PlayerState. Deux
consequences, dont une corrige le commentaire du `Build.cs` de QBuilder :

- ce hook n est **pas** un gate : ses deux seuls appelants sont
  `QBuilder_Resource_Price_Compute` (`:2241`) et le remboursement (`:2298`), et il rend un
  **multiplicateur** de prix, pas un booleen de refus. Le commentaire du `Build.cs`
  ("checks the acting player's wall before accepting a placement") est imprecis : un prix
  plus eleve peut faire echouer le paiement, ce n est pas la meme chose qu un refus.
  La conclusion du 7.4 tient donc : **le gating n existe pas** ;
- en revanche le point d ancrage, lui, existe deja, au bon endroit, avec le PlayerState
  deja resolu et la dependance QModule deja declaree. Le gate additif s installe la, sans
  nouvelle dependance de module et sans nouveau chemin de resolution.

### 7.5 Deplacement d un coffre

Le V1 ecrivait "nous refuserons le deplacement d un coffre non vide cote serveur" alors qu **aucun point d extension serveur n est identifie** sur le chemin `QBuilder_BP_ChangeTransformInstance`. Le V2 en faisait une demande. **RzZz a accepte le point d extension additif**, donc la regle devient applicable :

- **Destruction** : refusee tant que le coffre contient au moins une pile **resolue**. Le
  joueur vide d abord. **Aucune restitution automatique au sol** : elle creerait un chemin
  de duplication (des items au sol non repliques, plus un contenu encore en base si
  l ecriture ne suit pas) et un pic d acteurs. Les piles `bUnresolved` (4.5) ne bloquent
  pas la destruction, elles suivent le tombstone en quarantaine.
- **Deplacement** : autorise **uniquement a vide**. Le `FGuid` est **conserve**, l ancre est
  **recalculee**, et **les deux segments d index sont reecrits** (celui d origine et celui
  de destination, qui peuvent differer puisque l index est segmente par secteur). Un
  deplacement qui echoue a mi-chemin laisse le record dans le segment d origine : le
  segment de destination n est valide qu apres confirmation, jamais l inverse.
- **Les deux operations passent par le meme point d extension serveur** consulte dans
  QBuilder, additif, sans renommage ni changement de signature : un refus remonte au client
  par un code de refus lisible et localise, jamais par un echec muet.

**Limite assumee** : tant que ce point d extension n est pas livre (J6), un coffre reste
deplacable par le chemin QBuilder existant, ce qui **casse son ancrage**. Jusque-la, la
resolution par ancre echoue proprement (le record devient orphelin, liste par
`qstorage.Orphans`, jamais efface), elle ne rend pas le contenu d un autre coffre : c est la
regle de refus de collision de bucket du 4.1 point 3 qui garantit ce comportement.

---

## 8. UI

On **ne touche pas** a `Content/GameplayActors/Storage/BP_Storage.uasset` ni a `Content/Widget/Storage/PUW_Storage3d.uasset` : ce widget fait `SetWidgetAsSingleInstance` et sert le vault du PlayerState dans 44 emplacements de relais et stations ICLAB.

On duplique `Content/Widget/Storage/PUW_VehicleStorage.uasset` en **`PUW_QStorage_Container`** (prefixe corrige : les voisins directs sont `PUW_VehicleStorage`, `PUW_Storage3d`, `PUW_PlayerTrade` ; la regle projet est de suivre le prefixe du dossier voisin).

Ce qui change dans le wrapper :

- le drag and drop (`OnDnD_Item`, `OnDrop`) ne mute plus localement : il appelle `QST_ToServer_MoveSlot` en joignant l `InstanceId` attendu, et attend le delta ;
- etat "en attente serveur" par slot, refus lisible et localise sur `QST_ToClient_Refused` (jamais de refus muet), fermeture propre sur `QST_ToClient_Closed` ;
- slots `bUnresolved` grises et non deplacables ;
- fermeture (`QST_ToServer_Close`) sur echap, sur mort, sur eloignement.

**Cette section est conditionnee par la branche A ou B du 5.4.** En branche B, le panneau coffre n est pas `W_Inventory` mais un panneau de slots alimente par un modele pur.

---

## 9. Plan de livraison par jalons verifiables

Chaque jalon se termine par une **compilation a froid** : les `UWorldSubsystem` ne doivent jamais etre Live-Codes dans ce projet.

| Jalon | Perimetre | Critere de fin verifiable | Risque |
|---|---|---|---|
| **J0** | **CLOS le 2026-08-22.** Tous les arbitrages de design sont rendus (0.1 et section 11). Il ne reste aucune question de design ouverte. | Les dix decisions sont ecrites en section 11 ; la lecture B et l option (b) sont actees ; INCONNU 11 ferme par une lecture au pont | nul |
| **J1** | Socle : plugin, settings `Enabled=False`, structs, enums, log, subsystem vide. **Banc de mesure** de persistance (deux profils distincts, voir 7.3). **Test QATS de gel du registre DA_AllRef.** Commandes `qstorage.Status` (dont taille du slot monde, nombre de DataObjects, compteurs de pull), `qstorage.Dump`, `qstorage.Orphans`. | Build a froid des 3 cibles ; `qstorage.Status` repond "0 container" et affiche la taille reelle du slot ; **deux courbes consignees a 100 / 1000 / 5000 coffres synthetiques** : (1) `SaveGameToMemory` sur le chemin monde `AsyncSaveGameToSlot` du 4.2, (2) la **collecte** `QBuilder_SaveGame_GetData` sur le chemin QBuilder. Si la courbe (1) est deja a plusieurs ms a 1000, l architecture de stockage est revue **avant** d ecrire l acteur ; la courbe (2) decide du `QBuilder_SaveData` force du 7.3 et du toggle `bFlushAfterTransfer` du 4.3. | nul (systeme eteint) |
| **J2** | Acteur, enregistrement au BeginPlay, attachement paresseux, ouverture solo, UI (branche A ou B). | En **dedie** avec 1 client : deposer, retirer, **arret complet du process serveur**, relance, contenu identique. Plus le meme test en **listen server**. | faible |
| **J3** | Reseau : `OpenNearest` par proximite serveur, registre de sessions, snapshot fige et chunke, deltas en file, rebase, `PawnOwnsInstance`, rate limit, plafonds. | Dedie plus 2 clients : transfert croise sans duplication ; test QATS de concurrence ; **tentative d ouverture depuis une position sans coffre : refus** ; budget reseau par client mesure et consigne. | moyen |
| **J4** | Persistance **voie (B)** (4.2bis) : fichier propre a QStorage, copie sur le game thread puis encodage **et** ecriture sur thread de fond, remplacement atomique plus une generation conservee, segmentation par port refaite, chargement paresseux par segment, codec `QSTCRATE;v1`, echelle de migration, quarantaine amont, tombstones, plafonds. **Remonte avant J2** : c est lui qui portait le risque. | Redemarrage serveur : contenu intact. **Le banc rejoue sur le nouveau chemin et montre un cout game thread reduit a la copie** (sinon (B) n a rien resolu). Migration v1 vers v2 rejouee a froid sur chaine. **Cycle remplir / detruire / reconstruire au meme endroit : le contenu ne revient pas.** Segment illisible : coffre en lecture seule et Error, jamais de migration. | moyen : le fichier n est plus partage, donc une erreur n abime plus les donnees des autres systemes |
| **J5** | Cas QLevel : `FQST_FillSpec`, tirage paresseux pondere, hook `QLevel_LevelLoad`, tag `QL_ExcludeMerge`, regeneration de l optimise sur 1 niveau pilote. | `qstorage.ProbeAt` **sur le chemin streame reel** (jamais une lecture directe de l asset : precedent `qmoduleloot.SimulateLevel` qui affichait des succes pendant que le vrai chemin etait mort) : coffre present, pre-rempli, pille une seule fois. Verification before/after que le tag survit a la regeneration. | **eleve** : passe d optimisation destructrice |
| **J6** | **Fortement reduit par la lecture B.** Substituer le `FGuid` QStorage au `BuilderID` comme cle de `SetIdAndGetData` sur `QBuilder_Builder_Actor_BP` (4.0), gate structurel, points d extension additifs `CanAddInstance` / `ChangeTransform` / destruction (7.5), rebranchement de l entree `QA_ICLAB_StorageBase` si elle reste utile (7.1), correction du crash latent `Builders.Find`. **Aucune nouvelle piece de catalogue.** | **Le test qui prouve la correction du bug 4.0** : poser deux zones, remplir la seconde, detruire la premiere, redemarrer le process, verifier que la seconde retrouve **son** contenu (avant le chantier, elle heritait de celui d une autre). Plus le cycle IsOn/IsOff avec UI ouverte (aucune perte, `Closed` recu). | moyen : touche une chaine **en production**, celle du builder. Sauvegarde des assets avant edition (le backup auto du pont est mort). |
| **J7** | Durcissement : suite QATS complete, doc `Documentation/QSTORAGE_ARCHITECTURE.md`, activation. | Suite QATS verte ; bande passante par client et taille du slot monde documentees avant et apres activation. | nul |

---

## 9.1 Journal d avancement

**J1 (socle) : ECRIT, COMPILATION NON VALIDEE.** 2026-08-22.

Livre :

- `Plugins/QStorage/` : `.uplugin` (Runtime, LoadingPhase Default, dependance QBuilder),
  `QStorage.Build.cs`, module et categorie de log `LogQStorage` ;
- `QStorage_Types.h/.cpp` : les quatre enums (`EQST_AnchorKind`, `EQST_AccessMode`,
  `EQST_Refusal`, `EQST_MoveDirection`) et les quatre structs (`FQST_AnchorKey` avec sa
  quantification 10 cm et son `GetTypeHash`, `FQST_ItemStack`, `FQST_ContainerRecord`,
  `FQST_Tombstone`), plus `SchemaVersion` et le tag de codec ;
- `QStorage_DevSettings.h/.cpp` et la section `[/Script/QStorage.QStorage_DevSettings]`
  de `Config/DefaultGame.ini`, **`Enabled=False`**, avec les plafonds arbitres ;
- `QStorage_World_SubSystem.h/.cpp` : registres, sequence monotone d instance, collecte
  des orphelins, trois dumps. Inerte tant que le systeme est desactive ;
- `QStorage_Bench.cpp` : `qstorage.Bench`, les **deux** profils separes ;
- `QStorage_TestCommands.cpp` : `qstorage.Status`, `qstorage.Dump`, `qstorage.Orphans`,
  `qstorage.Enable`, `qstorage.Disable` ;
- `Plugins/QAutomatedTestSuite/.../QStorageRegistryTests.cpp` : deux tests QATS,
  `QATS.QStorage.Registry.ItemKeyRegistryIntegrity` et `...ItemKeyRegistrySizeFrozen`.

**Ecart au plan, assume** : le gel du registre ne duplique pas
`QModuleItemRegistrationTests.cpp`, qui couvre deja "les items MODULE sont inscrits dans
`DA_AllRef` et pointent un script". Les deux nouveaux tests couvrent ce que celui-la ne
couvre pas et dont QStorage depend : l integrite de **tout** le registre (aucune cle None,
aucune entree pointant dans le vide) et le fait qu il **ne retrecisse pas**.

**Valide par la mesure, le 2026-08-22 au soir** :

1. **Compilation, deux cibles sur trois.** `QangaEditor` : `UnrealEditor-QStorage.dll`
   produite, les 7 objets compiles dont `QStorage_Bench.cpp.obj` (celui qui appelle
   l API QBuilder, donc le lien inter-plugin est valide) et `QStorageRegistryTests.cpp.obj`
   cote QATS. `QangaServer` : `Result: Succeeded`, 1218 actions, `QangaServer.exe` linke,
   831 s. **`Qanga` (client) n a PAS ete compile** : c est la troisieme cible, restante.
2. **Les deux tests QATS passent.** `Result={Success}` sur
   `QATS.QStorage.Registry.ItemKeyRegistryIntegrity` et `...ItemKeyRegistrySizeFrozen`.
   Note d outillage : l automation attend une framerate de 10 FPS avant de demarrer
   (`FWaitForInteractiveFrameRate`), elle a attendu 600 s puis a renonce et a tourne quand
   meme, parce qu un build tournait en parallele et tenait l editeur a 3 FPS. Ne pas lancer
   de tests et de build en meme temps.
3. **Registre mesure : 303 entrees, 0 valeur nulle** (lecture directe de
   `DA_AllRef.ItemKey:DAItem` dans l editeur). Le plancher du test est **cale sur 303**,
   et non sur les 301 herites du dossier d enquete.

---

### La mesure qui compte : le banc, et ce qu il declenche

`qstorage.Bench` (40 piles par coffre), profil 1, chemin monde, **game thread** :

| Coffres | Encodage | `SaveGameToMemory` | Taille serialisee | **Cout reel d un flush** (slot + miroir `BACKUP_`) |
|---|---|---|---|---|
| 100 | 3,01 ms | 0,79 ms | 307 Ko | **1,58 ms** |
| 1000 | 26,58 ms | 5,42 ms | 3,0 Mo | **10,83 ms** |
| 5000 | 130,05 ms | 31,40 ms | 15,3 Mo | **62,80 ms** |

**La clause de revision du jalon J1 est declenchee.** Le critere ecrit etait : "si le
chiffre double est deja a plusieurs ms a 1000 conteneurs, l architecture de stockage est
revue AVANT d ecrire l acteur". Il est a 10,83 ms, soit **65 pour cent d une frame a
60 FPS**, pour la seule part QStorage, sur un slot qui porte deja tout le reste. Au
plafond arbitre de 5000 coffres, c est 62,80 ms, presque quatre frames : un hoquet visible.

Trois enseignements que seule la mesure donnait :

- **la segmentation de l index ne resout que la moitie du probleme.** Elle borne le
  **re-encodage** (un segment au lieu de tout), mais `SaveGameToMemory` s applique au
  **slot entier** a chaque flush, quel que soit le decoupage. Ce poste croit avec le
  nombre total de coffres persistes et rien dans le design actuel ne le borne ;
- **l encodage domine la serialisation d un facteur quatre** (130 ms contre 31 ms a 5000).
  Le poste le plus cher est notre propre codec, pas le moteur. Il est donc optimisable,
  et il est deja borne par la segmentation ;
- **le plafond `MaxPersistedContainersPerWorld = 5000` n est pas tenable** avec le slot
  monde partage. Soit il descend, soit le contenu sort de ce slot.

**Consequence : J2 (ecriture de l acteur) ne demarre pas avant cet arbitrage.** Deux voies,
et c est une decision de RzZz :

- **(A) baisser le plafond** et rester dans le slot monde partage. Simple, zero code
  nouveau, mais le nombre de coffres devient un budget serre : environ 1000 coffres pour
  rester sous les 11 ms, et ce budget est partage avec tout ce que le slot porte deja ;
- **(B) sortir le contenu des coffres du slot monde** vers un fichier propre a QStorage,
  ecrit hors game thread (patron `QBuilder_AsyncSave`, mesure a
  `QBuilder_SubSystem.cpp:751-760` : `SaveGameToSlot` **entierement** dans un
  `AsyncTask`, donc zero cout game thread pour la serialisation). Plus de travail, mais
  cela retire le plafond et decouple QStorage de la croissance du slot partage.

**Non mesure** : le profil 2 (chemin QBuilder) exige une session avec des zones
construites. Le banc a repondu "no game world" hors PIE. A relancer en PIE sur une
sauvegarde qui contient des builders, c est ce qui decide du `QBuilder_SaveData` force
du 7.3.

---

## 10. Mesures restantes, avec la commande exacte

**Aucun inconnu de design ne subsiste** : la section 11 les a tous tranches. Ce qui suit
sont des **mesures**, replacees dans le jalon ou elles servent. Aucune ne bloque J1.

L INCONNU 11 du V2 est **ferme** : `QA_ICLAB_StorageBase` a ete lu au pont par RzZz le
2026-08-21 et porte `ActorIsReplicated = False`, `ActorAlwaysOnServer = True`,
`ActorPersistantOnServer = False`, exactement la configuration du 7.2 (voir 7.1).

| # | Mesure | Jalon | Comment |
|---|---|---|---|
| M1 | Identite du troisieme `Add Component by Class` de `VehiclePlayerOwner.BeginPlay` (probablement `ORManagerComponent`), et son role dans le fait qu un `InventoryComponent` devienne pret hors PlayerState. **Portee reduite** : ne concerne plus que le coffre de niveau, le coffre construit heritant d une chaine en production. | J2 | `get_detailed_blueprint_summary` sur `/Game/Systems/Vehicle/VehiclesOwned/VehiclePlayerOwner` |
| M2 | Un `InventoryComponent` cote client peut-il devenir pret via `SetNotPersistentData` sans `GameDataManager` (gate `Is Server`) ? Decide la branche A ou B du panneau **du coffre de niveau** (5.4). | J2 | `get_detailed_blueprint_summary` sur `/Game/Systems/Item/InventoryComponent` (graphes `Event BeginPlay`, `ReplicateSaveInventory`) et sur `/DataManager/PersistentDataComponent` |
| M3 | Double spawn en listen server : l hote recoit-il l ordre `ToClients` pour un acteur `ActorAlwaysOnServer` ? | J5 et J6 | Test en listen server plus comptage des acteurs a la pose d une piece existante equivalente |
| M4 | Taille reelle d un slot monde sur une instance de prod peuplee (64 Ko mesures en dev solo). | **J1** | `ls -la Saved/SaveGames/Port*/` et `Saved/SaveGames/BACKUP_Port*/` sur une instance de prod |
| M5 | Flags exacts des RPC `SV_*` de `InventoryComponent` (Server/Reliable, appelables sur le composant d un autre joueur). | J3 | `execute_python_script` : lire `FUNC_Net / FUNC_NetServer / FUNC_NetReliable` sur les `UFunction` `SV_*` de `InventoryComponent_C` |
| M6 | Comportement de diffusion du `RequesterOptimizedState` **BP** (multicast comme le port C++ ou RPC cible ?), et exposition d un equivalent de `bServerMulticast` et de `SetCustomLocationKey`. Conditionne le repli du 2.4. | J3 | `get_detailed_blueprint_summary` sur `/Game/Systems/OptimizedState/RequesterOptimizedState` et `OptimizedStateComponent` (asset de 1,25 Mo, le pont repond par intermittence : reessayer) |
| M7 | Generation et unicite du `DataId` d un vehicule possede (`VehicleDataId`), modele a imiter pour le GUID. | J4 | `trace_blueprint_flow` sur `SpawnPlayerOwnedVehicle` |
| M8 | `PlayerOwnerOnlyAccessible` sur `InventoryComponent` : verrou d acces existant ou variable morte ? Touche directement `EQST_AccessMode = OwnerOnly`. | J3 | `search_project_index` puis lecture des graphes consommateurs au pont |
| M9 | `LootActorReloadRespawnTime` : persiste vers `GameDataManager` ou memoire de session ? | J5 | `get_detailed_blueprint_summary` sur `/Game/Systems/Item/ItemsManagerGS` |
| M10 | Ordre de chargement : quand `GameDataManager` est-il pret par rapport au `BeginPlay` du monde ? Faut-il enregistrer QStorage dans l ordre pilote par DataAsset de QGameManager ? | **J1** | `get_detailed_blueprint_summary` sur `/Game/GameMode/QangaGameState` plus lecture du DataAsset d ordre de QGameManager |
| M11 | Qui appelle `QBuilder_SaveData_LoadFromFile_Async` au demarrage, et quand par rapport a l enregistrement du premier client sur un builder. | J6 | `get_detailed_blueprint_summary` sur `/Game/Systems/QBuilder/Manager/QBuilder_Manager` |
| M12 | Le slot QBuilder n est pas segmente par port : deux instances sur la meme machine partagent-elles `QBuilder_SaveGame_N.sav` ? | J6 | `ls -la Saved/SaveGames/` sur une machine hebergeant deux instances |
| M13 | Couverture QATS existante du cycle save/load de QBuilder, **avant** toute intervention sur sa chaine. | **avant J6** | Inspection de `Plugins/QAutomatedTestSuite/` puis lancement du harnais QATS |
| M14 | Le tag `QL_ExcludeMerge` survit-il a `DuplicateAsset` lors de la regeneration de l optimise ? | J5 | Before/after par `execute_python_script` sur le niveau pilote |
| M15 | Le backend HTTP `QangaDatabaseConnection` (sidecar Node) est-il prevu pour la prod dediee ou abandonne ? S il revient, il change le chemin de persistance. | question ouverte a RzZz, non bloquante | a poser, pas a deduire |
| M16 | Validation au pont de la procedure de catalogue QBuilder (marquee SUPPOSE), pour les etapes non encore exercees. | J6 | `get_data_asset_details` sur `QDD_ICLAB_Storage` et `QBuilder_Qanga_ActorData` |

---

## 11. Decisions arbitrees par RzZz (2026-08-22)

Les dix questions du V2 sont tranchees. Elles sont conservees ici avec leur reponse et son
motif, parce qu une decision sans motif se redecide au premier obstacle.

1. **Lecture A ou B de la regle "premier objet a construire" ?**
   **Lecture B, definitif.** La chaine coffre-via-builder est deja cablee, UI comprise : le
   coffre cree la zone, le gate devient structurel. Le perimetre de QStorage cote construit
   se reduit a fournir l identite stable (`FGuid` serveur, index segmente, tombstones) a la
   place du `BuilderID`, et a garder le cablage existant. **Cela reduit J6 a presque rien et
   corrige au passage le bug du 4.0.**

2. **Existence contre contenu.**
   **Option (b)** : fenetre assumee, reconciliation, recuperation admin (7.3). L autosave
   600 s avec rotation est verifie. Le banc J1 mesure un `QBuilder_SaveData` force ; s il
   passe, il est adopte sur les deux seuls evenements rares, pose et destruction, **jamais**
   sur un mouvement.

3. **Deplacement et destruction d un coffre construit.**
   **Point d extension additif accepte** sur `ChangeTransform`. Destruction refusee si des
   piles resolues subsistent, on vide d abord, **jamais de restitution automatique au sol**.
   Deplacement autorise a vide uniquement, `FGuid` conserve, ancre recalculee, les deux
   segments d index reecrits (7.5).

4. **Acces a un coffre construit.**
   **`OwnerOnly` par defaut**, partage via `Allowed_By_ID` du builder
   (`Plugins/QBuilder/Source/QBuilder/Public/QBuilder_Client.h:47`). `Public` reserve aux
   coffres de niveau. `GroupOnly` plus tard via `PlayerGroupComponent`
   (`Content/Systems/Group/`). **L enum reste inchange.**

5. **Synchronisation live entre deux joueurs devant le meme coffre.**
   **Oui.** Deux joueurs qui pillent le meme coffre arrivera, et un snapshot perime genere
   des signalements de duplication. Les deltas, le snapshot fige et le rebase sont conserves
   tels quels. **Plan B explicite** : rafraichissement a l ouverture si le cout ne passe pas.

6. **Plafonds initiaux**, tous en `.ini`, a revisiter avec la courbe J1 :
   **128** sessions ouvertes simultanees (pic estime a environ 50 sur 500 joueurs),
   **`MaxHotContainers` 256**, **5000** coffres persistes par monde,
   **2000** entrees par segment d index.

7. **Cible du composant d etat.**
   **Composant BP pour la v1**, c est la voie deja en production (`OnServerMulticastEvent`
   est consomme ainsi sur le builder). Le port C++ **n est pas abandonne, il est en attente
   de bascule**. Le repli du 2.4 reste conditionne a la mesure M6.

8. **`RollWeightedItem` et `HashWorldLocation`.**
   **Extraction `Cy_*` dediee**, pas de copie : l en-tete de `QModuleLoot_Library` impose
   deja le no-duplicates du tirage pondere entre ses trois consommateurs actuels, une copie
   irait contre ce que le fichier declare lui-meme. Tache dediee, hors du chemin du socle,
   a livrer avant J5.

9. **Set QBuilder d accueil : ICLAB.**
   Et l entree `QA_ICLAB_StorageBase` existe deja avec la bonne configuration (INCONNU 11
   ferme). `BP_Storage_ForBuild` est en revanche une **coquille vide** sans
   `InventoryComponent` : on **remplace ou rebranche** cette entree, on n en ajoute pas une
   parallele. Nommage final apres application de la lecture B (7.1).

10. **Rechargement des coffres de niveau.**
    **Restock paresseux oui** pour le loot commun via `RestockSeconds`, a la premiere
    ouverture posterieure a l echeance. Les `GuaranteedStacks` de salle de boss sont
    **definitivement pilles**. Le comportement est **decide par `FillProfileTag`** (6).

**Decision transverse : aucune migration des anciens contenus** keyes sur `BuilderID`
numerique. Ils ne sont pas stables par construction, il n y a rien de sain a migrer, et
`qstorage.Orphans` les ramasse comme le reste (4.0).

---

## 12. Critiques ecartees, et pourquoi (une ligne chacune)

1. **"Poids transitif : QStorage linkerait QAI et DQS via QModule"** : non fonde, les `PrivateDependencyModuleNames` de QModule ne remontent pas dans QStorage et QModule est deja charge projet-wide. J adopte quand meme l independance, mais pour le motif du cycle J5.
2. **"Supprimer `bOpened`, un booleen n a pas de semantique pour N ouvreurs"** : non fonde en tant que tel, le bit reste valide s il est pilote par le compteur serveur d ouvreurs de la session. Le reste de la critique sur le multicast est integralement retenu.
3. **"Le patron `VehiclePlayerOwner` ne marche pas puisqu il repose sur un acteur replique"** : fonde pour la moitie **cliente** (section 8 reecrite en deux branches), non fonde pour la moitie **serveur** : la verite et la persistance ne dependent pas de `AddReplicatedSubObject`.
4. **"La cle de l OptimizedState bascule sur la cle composite QBuilder"** : fonde comme risque et traite au 2.4, mais la mesure porte sur le port C++ **dormant** ; le comportement du composant BP en service reste a verifier (INCONNU 6), donc le correctif est conditionnel et non acquis.
5. **"Un coffre replique pose par QBuilder est casse par le nom de paquet `/Temp`"** : c est **ma propre affirmation V1 qui etait non fondee**, corrigee au 5. L argument `/Temp` ne vaut que pour les coffres poses en sous-niveau ; pour le coffre construit, le vrai argument est le cout reseau.
6. **"La duplication vient de l affichage optimiste"** : non fonde, et c est encore une erreur du V1 ; la duplication vient de la resolution serveur d un index de slot source non versionne. Corrige au 5.2 point 2.

---

### Changelog Discord (pret a coller)

**🗄️ QStorage : le coffre local est valide, et il repare un bug qu on avait pas vu**

**Ce que c est** : un vrai coffre a contenu local (chaque coffre a ses objets), server-authoritative et persistant a travers les mises a jour. Deux usages : la zone de construction que vous posez, et des coffres poses a la main dans les interieurs, pre-remplis, en recompense de decouverte ou en salle de boss.

**Le truc qu on a trouve en chemin** : la zone de construction stocke deja vos mineraux, mais elle s y retrouve grace a un numero **reattribue a chaque chargement du serveur**. Traduction : le contenu des zones construites ne survivait pas de facon fiable a un redemarrage. Personne l avait vu parce qu en solo continu les numeros retombent juste. C est exactement le trou que le nouveau systeme bouche, avec un identifiant qui, lui, ne bouge plus.
- **Si vous avez deja eu un coffre de base vide apres un restart** : c etait ca.

**Les autres decisions**
- **Poser le coffre cree la zone de construction.** Plus besoin de gate artificiel : pas de coffre, pas de chantier.
- **Un coffre plein ne peut pas etre detruit**, et un coffre ne se deplace qu a vide. Pas de restitution automatique au sol, ca ferait des objets dupliques.
- **Prive par defaut**, partage avec la liste d autorisation deja existante de votre zone.
- **Deux joueurs peuvent fouiller le meme coffre en meme temps** sans se marcher dessus.
- **Les coffres de loot commun se rechargent**, les recompenses de boss se pillent **une seule fois, pour tout le serveur.**

**Etat** : design valide, developpement demarre au socle. Le premier jalon est un banc de mesure : on mesure le cout d ecriture avant d ecrire la moindre ligne de gameplay.