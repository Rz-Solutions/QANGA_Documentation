# QStorage: coffre a stockage LOCAL, replique, persistant, constructible et posable
### Document de design v2 (consolide apres revue hostile) : a valider par RzZz. Aucun fichier n a ete modifie, aucune ligne de code n a ete ecrite.

---

## 0. Synthese en 10 lignes

1. On construit un conteneur a contenu **local** (chaque coffre a ses propres objets), server-authoritative, persistant a travers les mises a jour, utilisable en deux poses : construit par QBuilder, ou place a la main dans un sous-niveau QLevel.
2. Ca vit dans un nouveau plugin `Plugins/QStorage/` (couche Q\*), C++ mince plus assets BP, **sans aucune dependance sur QModule** (voir 1.3).
3. La verite du contenu reste l `InventoryComponent` BP (`Content/Systems/Item/InventoryComponent.uasset`) attache au runtime cote serveur, patron deja en prod dans `Content/Systems/Vehicle/VehiclesOwned/VehiclePlayerOwner.uasset`.
4. Zero replication de l acteur : le modele reseau copie `Content/GameplayActors/Shop/BP_Shop.uasset` (mesure `replicates=false`), avec ouverture par le canal d interaction existant `Lib_OptimizedState::SendOptimizedActorInteraction`.
5. Le client n envoie **jamais** de position ni de reference d objet : il dit "j interagis", le serveur resout l acteur, choisit le coffre et renvoie un handle.
6. L identite durable est un `FGuid` alloue par le serveur, resolu au respawn par un index d ancrage **calcule uniquement cote serveur**, avec tombstone a la destruction.
7. La persistance passe par `GameDataManager` / `DataObject` (`Plugins/DataManager/Content/`), mais **QStorage possede sa propre cadence d ecriture** : une cle tableau par coffre, index segmente par secteur, flush a la fermeture de session, jamais par mouvement d objet.
8. Le versionnement est un entier `SchemaVersion` avec echelle de migration, jamais l egalite stricte destructrice de QBuilder (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:994`) ni de DQS.
9. L UI duplique `Content/Widget/Storage/PUW_VehicleStorage.uasset` ; on ne touche ni `BP_Storage` ni `PUW_Storage3d` (44 emplacements de relais et stations).
10. **Le seul vrai point dur** : la couche d ecriture heritee. `TempDB.WriteTempData` reecrit le slot monde ENTIER plus son miroir `BACKUP_`, et `AsyncSaveGameToSlot` serialise en memoire **de facon synchrone sur le game thread** (`C:/UE5_Share/Engine/Source/Runtime/Engine/Private/GameplayStatics.cpp`). Tout le design plie devant ce fait : c est lui qui impose l index segmente, la cadence possedee par QStorage, les plafonds durs, et un banc de mesure au jalon J1.

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

`QStorage.Build.cs` : `Core, CoreUObject, Engine, QLevel` (public/prive selon usage), rien d autre. `RollWeightedItem`, `ResolvePickupClass` et `HashWorldLocation` (`Plugins/QModule/Source/QModule/Public/QModuleLoot_Library.h:19-66`) sont soit copies dans QStorage avec un commentaire de provenance, soit extraits en `Cy_*` par une tache dediee validee **avant J1** (question 8 a RzZz).

**Critique ecartee, partiellement** : l argument du "poids transitif" (QStorage tirerait QAI et DQS) est **non fonde**. Les `PrivateDependencyModuleNames` de QModule ne remontent pas dans QStorage, et QModule est deja actif projet-wide donc deja charge. J adopte quand meme l independance, mais pour le motif du cycle J5, pas pour le poids.

### 1.4 Activation

`UQStorage_DevSettings`, section `[/Script/QStorage.QStorage_DevSettings]` dans `Config/DefaultGame.ini`, **`Enabled=False` a la livraison**. Categorie de log `LogQStorage`.

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

### 4.1 Identite : GUID serveur, ancre serveur, tombstone

Le relecteur data a raison : dans le V1, l identite reelle etait la position, le GUID n en etait qu un alias. Corrige sur trois points.

1. **L ancre n est calculee que par le serveur.** Le client ne la connait pas, ne l envoie pas, ne peut pas la fabriquer (voir 5.1). Cela supprime d un coup : la generation de loot a volonte, la divergence de quantification `FRepMovement` pour une piece QBuilder, et le cas `NM_ListenServer` ou `ServerTransformFull` n est jamais assigne (`Plugins/QBuilder/Source/QBuilder/Private/World/QBuilder_BuilderActor.cpp:176-186`, replique `COND_InitialOnly` ligne 32) et ou les clients reposent leur builder a l identite.
2. **Tombstone obligatoire.** A la destruction d un coffre : ecriture d une pierre tombale persistee (GUID, ancre, `CreatedSeq`), refus de toute resolution d ancre vers un GUID marque detruit, et appel de `DeleteDataObject` / `DeleteDataFromDB` (exposes par `Plugins/DataManager/Content/GameDataManager.uasset`, jamais appeles par le V1) sur le DataObject du coffre, dans la meme transaction. Sans cela, la boucle est triviale : remplir, detruire, ramasser la restitution au sol, reconstruire au meme endroit a moins de 10 cm, recuperer le contenu une seconde fois.
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

### 4.3 Atomicite d un transfert

`TryMoveItemToAnotherInventory` retire de la source puis ajoute a la cible : deux DataObjects distincts, coalesces par le `Delay 1 s + Do Once` de `WriteTempData`. Si les deux mutations tombent de part et d autre d un cycle et que le process meurt entre les deux, l objet est retire du joueur et jamais ajoute au coffre. Le `BACKUP_` n aide pas : il est ecrit une seconde apres le slot principal, il miroite l etat fautif.

Correctif retenu : **journal d intention** (une ligne WAL dans le segment d index : GUID, ItemKey, quantite, sens, seq) ecrite avant la mutation, retiree apres confirmation d ecriture ; au chargement, toute intention non retiree est rejouee ou annulee, et journalisee en Warning. Alternative moins couteuse a evaluer a J4 : faire aboutir les deux cotes du transfert dans la meme cle serialisee, ce qui n est possible que pour un transfert coffre vers coffre, pas joueur vers coffre.

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

**Le coffre ne se replique jamais.** `bReplicates = false`, dans les deux poses.

Correction d argument : le V1 justifiait cela pour le coffre **construit** par le nom de paquet `/Temp.../_LevelInstance_<compteur statique>`. C est faux pour ce cas : cet argument ne vaut que pour un acteur pose dans un sous-niveau streame (`Engine/Private/LevelStreaming.cpp:2780-2795`), et `AQModule_WorkbenchActor` prouve qu un acteur player-built spawne au runtime se replique correctement. **Le vrai argument pour le coffre QBuilder est le cout** : aucun ReplicationGraph n existe dans le projet (0 `ReplicationDriverClassName` dans `Config/`), la pertinence est la boucle naive du NetDriver, et 21429 ancres CONTAINER sont deja cataloguees. Pour le coffre de niveau, l argument `/Temp` reste valable et suffisant.

### 5.1 Ouverture : le client ne nomme jamais un coffre

```cpp
UFUNCTION(Server, Reliable, WithValidation)
void QST_ToServer_OpenNearest();                       // NO position, NO key, NO object ref

UFUNCTION(Server, Reliable, WithValidation)
void QST_ToServer_Close(FGuid ContainerGuid);

UFUNCTION(Server, Reliable, WithValidation)
void QST_ToServer_MoveSlot(FGuid ContainerGuid, uint32 ExpectedContentVersion, uint8 Direction,
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

**Critique partiellement ecartee** : "supprimer `bOpened` car un booleen n a pas de semantique pour N ouvreurs" est **non fonde en tant que tel**. Le bit reste utile pour le visuel a distance ; ce qu il faut, c est qu il soit pilote par le **compteur serveur d ouvreurs** de la session (passe a false quand le compteur retombe a zero), pas par la fermeture du premier joueur. Je conserve `bOpened` avec cette semantique explicite. Le reste de la critique (multicast, cout) est integralement retenu.

### 5.4 Client : pas de reutilisation aveugle de l InventoryComponent

Le V1 affirmait qu on remplirait un `InventoryComponent` transitoire cote client "format identique a `F_ItemRequestReply`, donc reutilisation du chemin BP existant". **Rien ne l etaye** : `GameDataManager.BeginPlay` est gate `Is Server`, `InventoryComponent.BeginPlay` reste a `IsInventoryReady = false` sans `PersistentDataComponent` pret, `W_Inventory` consomme des `Obj_ItemInstance` (portes par `DnD_ItemInstance`) et non des `FQST_ItemStack`, et aucun chemin BP mesure ne construit un `Obj_ItemInstance` cote client. C est un **inconnu bloquant tranche a J0** (INCONNU 2), avec deux branches ecrites d avance :

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
- **Pieges d edition, non negociables** : poser dans `LevelData.Level`, puis **regenerer l optimise** (c est `OptimisedLevel` qui est streame, et le cook ecrase `Level` par `OptimisedLevel`). L acteur doit porter le tag **`QL_ExcludeMerge`**, sans quoi la passe d optimisation le **detruit** des qu un `UStaticMeshComponent` est absorbe dans un ISM (`Plugins/QLevelEditor/Source/QLevelEditor/Private/QLevel_Editor_OptimisationLibrary.cpp:611-645`, `:809-822`). `ExcludeMergeClass` est un tableau `.ini` qui **remplace** le defaut C++ (`Config/DefaultEditor.ini:39`, une seule entree `VehicleSpawner_C`) : le tag acteur est plus sur.
- Le coffre ne doit **jamais** etre enregistre dans la map de nettoyage a l unload de `QModuleLoot` (elle detruit ce qu elle a spawne).

---

## 7. Integration QBuilder

### 7.1 Chaine catalogue

Procedure a 5 etapes, **marquee SUPPOSE dans le dossier** (deduite par `rg -a` sur des .uasset, jamais verifiee au pont) : creer le `QA_*` (`UQBuilder_Data_ActorData`, `ActorClass = BP_QStorage_Container_Build`), creer le `QDD_*`, inscrire sous un ID int32 unique dans `Content/Systems/QBuilder/Data/QBuilder_Qanga_ActorData.uasset`, referencer dans le `QTS_` du set, puis **`DB_Optimise` dans `Content/Systems/QBuilder/Tools/QBuilder_DataTools.uasset`** (sans quoi la piece est invisible du runtime, sans erreur ni log).

Nommage corrige : le segment median des assets QBuilder existants est le **set** (`ICLAB`), et "QANGA" n est pas un set connu. Le set d accueil et le `QTS_` correspondant doivent etre **nommes par RzZz** (question 9), pas inventes ici.

### 7.2 Regles de spawn

`ActorIsReplicated = false`, `ActorAlwaysOnServer = true`, **`ActorPersistantOnServer = false`**. Ce dernier point est vital : malgre son nom, `true` exclut l acteur de `IsOn` **et** de `IsOff` (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:1761` et `:1794`), donc il n est jamais respawne apres un redemarrage.

Contraintes heritees : la piece reste a moins de 327 m du builder (transform relatif borne int16), et l acteur est detruit puis recree a chaque cycle IsOn/IsOff. Le modele y survit : rien de durable ne vit dans l acteur.

**A corriger avant J6, hors perimetre fonctionnel mais dans le chemin** : le crash latent `Builders.Find` deferences avant test de nullite dans `QBuilder_Resource_Price_Compute` et `_Refund` (`Plugins/QBuilder/Source/QBuilder/Private/QBuilder_SubSystem.cpp:2190-2193` et `:2244-2247`), qui sera traverse plus souvent des que le coffre entre au catalogue.

### 7.3 Divergence de cadence entre l existence et le contenu

Fait nouveau, integralement retenu : l existence d un coffre construit vit dans `Saved/SaveGames/QBuilder_SaveGame_N.sav` (autosave **600 s**, rotation 4 slots, `Plugins/QBuilder/Source/QBuilder/Public/QBuilder_DevSettings.h:29-31`), son contenu vit dans le slot monde (cadence de l ordre de la seconde). Deux pannes :

- **perte seche** : coffre construit a T=0, rempli a T=30, crash a T=200 avant tout autosave QBuilder. Au redemarrage le contenu existe en base et l acteur n existe plus : les items sont inatteignables ;
- **duplication** : restauration admin d un `Build_2` vieux de 40 minutes en gardant le slot monde courant.

Regle de reconciliation ecrite, **non negociable** : un enregistrement QStorage dont l ancre ne produit aucun acteur **n est jamais efface automatiquement**. Il est conserve, journalise en Warning, liste par `qstorage.Orphans`, et recuperable par commande admin. Une purge n a lieu que sur demande explicite, avec liste affichee avant execution, et TTL configurable en nombre de demarrages sans resolution.

Deux options a trancher (question 2) : (a) forcer un `QBuilder_SaveData` a la pose et a la destruction d un coffre, ce qui aligne les cadences mais paie une ecriture Gzip complete de tous les builders a chaque evenement, cout a mesurer ; (b) accepter la fenetre et vivre avec la reconciliation ci-dessus.

### 7.4 La regle "premier objet a construire pour pouvoir construire"

**Ce mecanisme n existe pas** : QBuilder ne connait qu une autorisation (proprietaire ou `Allowed_By_ID`, `Private/QBuilder_SubSystem.cpp:380`) et un prix. Deux lectures :

- **Lecture A, litterale** : le coffre est une piece QBuilder qu il faut construire en premier. Logiquement circulaire, et exige un gate serveur additif ;
- **Lecture B** : poser le coffre **cree** la zone de construction (chemin existant `QBuilder_Builder_ClientCreateBuilderActorWithBuild`). La regle devient structurelle.

**Avertissement ajoute apres revue** : en lecture B, le coffre est le builder, et `Content/Systems/QBuilder/Actor/QBuilder_Builder_Actor_BP.uasset` embarque **deja** un `InventoryComponent`, un `PersistentDataComponent`, un `OptimizedStateComponent` et le struct `Inventory_Storage`, et reutilise deja `PUW_VehicleStorage`. Dans ce cas, une moitie de QStorage duplique une chaine existante et le chantier se reduit a : cabler l inventaire du builder sur une UI et une interaction, plus un gate de construction. **La lecture A ou B doit etre tranchee avant J1**, pas avant J6 : c est la seule decision qui peut supprimer quatre jalons.

Dans les deux cas, le garde-fou serveur est un point d extension **additif** dans QBuilder (delegue du type `QBuilder_OnCanAddInstance(BuilderID, DataID, ClientID) -> bool` consulte dans `QBuilder_ServerData_Actor_AddInstances`), sans renommage ni changement de signature.

### 7.5 Deplacement d un coffre

Le V1 ecrivait "nous refuserons le deplacement d un coffre non vide cote serveur" alors qu **aucun point d extension serveur n est identifie** sur le chemin `QBuilder_BP_ChangeTransformInstance`. Corrige : c est une **demande** de point d extension additif, au meme titre que le gate de construction, et tant qu il n existe pas, le deplacement d un coffre reste possible et **casse son ancrage**. C est la question 3 a RzZz.

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
| **J0** | Trancher : lecture A ou B (7.4), branche A ou B de l UI (5.4), montage d inventaire complet (2.2). Lever les INCONNUS 1 a 6. | Reponses ecrites aux 6 points, plus arbitrage RzZz sur les questions 1 et 2 | nul |
| **J1** | Socle : plugin, settings `Enabled=False`, structs, enums, log, subsystem vide. **Banc de mesure** de persistance. **Test QATS de gel du registre DA_AllRef.** Commandes `qstorage.Status` (dont taille du slot monde et nombre de DataObjects), `qstorage.Dump`, `qstorage.Orphans`. | Build a froid des 3 cibles ; `qstorage.Status` repond "0 container" et affiche la taille reelle du slot ; **courbe mesuree de `SaveGameToMemory` et de la taille du slot a 100 / 1000 / 5000 coffres synthetiques**, consignee. Si la courbe est deja a plusieurs ms a 1000, l architecture de stockage est revue **avant** d ecrire l acteur. | nul (systeme eteint) |
| **J2** | Acteur, enregistrement au BeginPlay, attachement paresseux, ouverture solo, UI (branche A ou B). | En **dedie** avec 1 client : deposer, retirer, **arret complet du process serveur**, relance, contenu identique. Plus le meme test en **listen server**. | faible |
| **J3** | Reseau : `OpenNearest` par proximite serveur, registre de sessions, snapshot fige et chunke, deltas en file, rebase, `PawnOwnsInstance`, rate limit, plafonds. | Dedie plus 2 clients : transfert croise sans duplication ; test QATS de concurrence ; **tentative d ouverture depuis une position sans coffre : refus** ; budget reseau par client mesure et consigne. | moyen |
| **J4** | Persistance : segments d index, codec `QSTCRATE;v1`, echelle de migration, quarantaine amont, tombstones, `DeleteDataObject`, WAL d intention, plafonds durs. | Redemarrage serveur : contenu intact. Migration v1 vers v2 rejouee a froid sur chaine. **Cycle remplir / detruire / reconstruire au meme endroit : le contenu ne revient pas.** Index illisible : coffre en lecture seule et Error, pas de migration. | **eleve** : ecriture dans le slot monde partage |
| **J5** | Cas QLevel : `FQST_FillSpec`, tirage paresseux pondere, hook `QLevel_LevelLoad`, tag `QL_ExcludeMerge`, regeneration de l optimise sur 1 niveau pilote. | `qstorage.ProbeAt` **sur le chemin streame reel** (jamais une lecture directe de l asset : precedent `qmoduleloot.SimulateLevel` qui affichait des succes pendant que le vrai chemin etait mort) : coffre present, pre-rempli, pille une seule fois. Verification before/after que le tag survit a la regeneration. | **eleve** : passe d optimisation destructrice |
| **J6** | QBuilder : `QA_`, `QDD_`, inscription, `DB_Optimise`, gating selon A ou B, correction du crash latent `Builders.Find`. | Construire, redemarrer le process, retrouver le coffre et son contenu **apres que `IsOn` a respawne l acteur** ; puis test du cycle IsOn/IsOff avec UI ouverte (aucune perte, `Closed` recu). | moyen : touche le catalogue QBuilder |
| **J7** | Durcissement : suite QATS complete, doc `Documentation/QSTORAGE_ARCHITECTURE.md`, activation. | Suite QATS verte ; bande passante par client et taille du slot monde documentees avant et apres activation. | nul |

---

## 10. INCONNUS a verifier, avec la commande exacte

Rien de ce qui suit n a ete mesure. Les six premiers sont **bloquants pour J0 ou J1**.

1. **Identite du troisieme `Add Component by Class` de `VehiclePlayerOwner.BeginPlay`** (probablement `ORManagerComponent`), et son role dans le fait qu un `InventoryComponent` devienne pret hors PlayerState. Bloquant J2.
   `get_detailed_blueprint_summary` sur `/Game/Systems/Vehicle/VehiclesOwned/VehiclePlayerOwner`.
2. **Un `InventoryComponent` cote client peut-il devenir pret via `SetNotPersistentData` sans `GameDataManager`** (qui est gate `Is Server`) ? Decide la branche A ou B de l UI. Bloquant J0.
   `get_detailed_blueprint_summary` sur `/Game/Systems/Item/InventoryComponent` (graphes `Event BeginPlay`, `ReplicateSaveInventory`) et sur `/DataManager/PersistentDataComponent` (`SetNotPersistentData`).
3. **Double spawn en listen server** : l hote recoit-il l ordre `ToClients` pour un acteur `ActorAlwaysOnServer` ? Bloquant J6.
   Test manuel en listen server plus `execute_python_script` : `len([a for a in unreal.EditorLevelLibrary.get_all_level_actors() if a.get_class().get_name().startswith('BP_QStorage')])` a la pose d une piece existante equivalente (`BP_Storage_ForBuild`).
4. **Taille reelle d un slot monde sur une instance de prod peuplee** (64 Ko mesures en dev solo). Bloquant J1.
   `ls -la Saved/SaveGames/Port*/` et `ls -la Saved/SaveGames/BACKUP_Port*/` sur une instance de prod.
5. **Flags exacts des RPC `SV_*` de `InventoryComponent`** (Server/Reliable, appelables sur le composant d un autre joueur).
   `execute_python_script` : charger la `BlueprintGeneratedClass` `/Game/Systems/Item/InventoryComponent.InventoryComponent_C` et lire `FUNC_Net / FUNC_NetServer / FUNC_NetReliable` sur les `UFunction` `SV_*`.
6. **Comportement de diffusion du `RequesterOptimizedState` BP** : multicast comme le port C++ (`MC_Update`) ou RPC client cible ? Et le composant BP expose-t-il un equivalent de `bServerMulticast` et de `SetCustomLocationKey` ?
   `get_detailed_blueprint_summary` sur `/Game/Systems/OptimizedState/RequesterOptimizedState` et `/Game/Systems/OptimizedState/OptimizedStateComponent` (asset de 1,25 Mo, le pont a deja repondu par intermittence : reessayer).
7. **Generation et unicite du `DataId` d un vehicule possede** (`VehicleDataId`, `VehicleDataIdMap`), modele a imiter pour le GUID.
   `trace_blueprint_flow` sur `SpawnPlayerOwnedVehicle` dans `/Game/Systems/Vehicle/VehiclesOwned/VehiclePlayerOwner`.
8. **`PlayerOwnerOnlyAccessible`** sur `InventoryComponent` : verrou d acces deja prevu ou variable morte ?
   `search_project_index` sur le nom, puis lecture des graphes consommateurs au pont.
9. **`LootActorReloadRespawnTime`** : persiste vers `GameDataManager` ou memoire de session ?
   `get_detailed_blueprint_summary` sur `/Game/Systems/Item/ItemsManagerGS` (graphes `SaveItemsPicked` et voisins).
10. **Ordre de chargement** : quand `GameDataManager` est-il pret par rapport au `BeginPlay` du monde ? Faut-il enregistrer QStorage dans l ordre pilote par DataAsset de QGameManager ?
    `get_detailed_blueprint_summary` sur `/Game/GameMode/QangaGameState` (EventGraph) plus lecture du DataAsset d ordre de QGameManager.
11. **Lecture reelle des trois flags de `QA_ICLAB_StorageBase`** (`ActorIsReplicated` / `ActorAlwaysOnServer` / `ActorPersistantOnServer`), deduits de la table de proprietes serialisees, jamais lus.
    `get_data_asset_details` sur `/Game/Systems/QBuilder/Data/ICLAB/Actor/Storage/QA_ICLAB_StorageBase`.
12. **Qui appelle `QBuilder_SaveData_LoadFromFile_Async` au demarrage du monde**, et quand par rapport a l enregistrement du premier client sur un builder.
    `get_detailed_blueprint_summary` sur `/Game/Systems/QBuilder/Manager/QBuilder_Manager`.
13. **Le slot QBuilder n est pas segmente par port** : deux instances sur la meme machine partagent-elles `Saved/SaveGames/QBuilder_SaveGame_N.sav` ?
    `ls -la Saved/SaveGames/` sur une machine hebergeant deux instances.
14. **Couverture QATS existante du cycle save/load de QBuilder**, avant toute intervention sur son catalogue.
    Inspection de `Plugins/QAutomatedTestSuite/` puis lancement du harnais QATS.
15. **Le tag `QL_ExcludeMerge` survit-il a `DuplicateAsset`** lors de la regeneration de l optimise ? Par construction oui, non verifie.
    Before/after par `execute_python_script` sur le niveau pilote au jalon J5.
16. **Le backend HTTP `QangaDatabaseConnection`** (sidecar Node `API.exe` / `API`) est-il prevu pour la prod dediee ou abandonne ? Ses binaires sont absents du depot et `QangaGameState` ne le reference pas, mais s il revient il change le chemin de persistance de QStorage. Question a poser, pas a deduire.
17. **Le procede de creation d une piece QBuilder (5 assets)** est marque SUPPOSE dans le dossier, deduit de `rg -a` sur des .uasset. A valider au pont avant J6 : `get_data_asset_details` sur `QDD_ICLAB_Storage` et sur `QBuilder_Qanga_ActorData`.

---

## 11. Questions a RzZz

1. **Lecture A ou lecture B de la regle "premier objet a construire" ?** Piece QBuilder gatee, ou objet d inventaire dont la pose cree la zone de construction ? En lecture B, le builder porte deja un `InventoryComponent` et un `PersistentDataComponent`, et QStorage se reduit peut-etre a une UI plus un gate. **A trancher avant J1**, pas avant J6.
2. **Existence versus contenu** : accepte-t-on la fenetre de 10 a 40 minutes entre l autosave QBuilder et l ecriture du monde (avec reconciliation et recuperation admin), ou force-t-on un save QBuilder complet a chaque pose et destruction de coffre (cout d une ecriture Gzip de tous les builders, a mesurer) ?
3. **Un coffre construit peut-il etre deplace ou detruit, et que devient son contenu ?** Refus si non vide, restitution au sol, ou transfert vers la banque du builder ? Et acceptes-tu d ajouter un point d extension serveur additif sur le chemin `ChangeTransform` ?
4. **Qui a acces a un coffre construit ?** Proprietaire seul, groupe ou clan (`PlayerGroupComponent` existe sur le PlayerState), liste `Allowed_By_ID` du builder, ou libre ? Cela fixe `EQST_AccessMode`.
5. **Synchronisation en direct entre deux joueurs devant le meme coffre**, ou rafraichissement a l ouverture suffisant ? C est la seule question qui justifie le cout du registre d abonnement, des deltas et du rebase.
6. **Combien de coffres ouverts simultanement en pointe** sur un serveur a 500 joueurs, et combien de coffres persistes acceptes par monde ? Ces deux chiffres deviennent des plafonds durs dans les settings.
7. **Cible du composant d etat** : on ecrit contre le composant BP `OptimizedStateComponent_C` (voie sure aujourd hui) ou contre le C++ `UOptimizedStateComponent` (voie propre demain, mais dormante) ? Le port C++ est-il abandonne ou en attente de bascule ?
8. **`RollWeightedItem` / `HashWorldLocation`** : extraction en `Cy_*` (tache dediee, validee, avant J1) ou copie dans QStorage avec commentaire de provenance ?
9. **Nom du set QBuilder d accueil** et du `QTS_` correspondant pour le coffre. "QANGA" n est pas un set existant.
10. **Un coffre de niveau doit-il pouvoir se recharger** (`RestockSeconds`, en rechargement paresseux a la premiere ouverture posterieure a l echeance), ou une recompense de decouverte est-elle definitivement pillee, par serveur, pour tous ?

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

**🗄️ QStorage : le design du coffre local passe en v2**

**Ce que c est** : un vrai coffre a contenu local (chaque coffre a ses objets), constructible avec QBuilder ou posable a la main dans les interieurs, server-authoritative et persistant a travers les mises a jour.

**Ce qui a change apres la revue** : trois relecteurs ont demonte la v1, et ils avaient raison sur l essentiel.
- **Cause** : le client fabriquait la cle du coffre qu il voulait ouvrir. **Fix** : le client ne dit plus que "j interagis", le serveur choisit le coffre le plus proche. Ca tue au passage la generation de loot infinie et le bug listen server.
- **Cause** : la persistance heritee reecrit tout le fichier monde a chaque mouvement d objet, sur le thread de jeu. **Fix** : QStorage possede sa propre cadence, index segmente par secteur, ecriture a la fermeture du coffre. Et un banc de mesure obligatoire avant d ecrire l acteur.
- **Cause** : detruire puis reconstruire un coffre au meme endroit rendait l ancien contenu. **Fix** : pierres tombales et vraie suppression en base.
- **Cause** : un item dont l asset disparait etait efface de la base, en silence. **Fix** : quarantaine en amont, la pile n est jamais confiee au systeme qui la detruirait.

**Etat** : design a valider, zero ligne de code, zero fichier touche. 10 questions en attente d arbitrage.