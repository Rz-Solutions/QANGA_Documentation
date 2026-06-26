# QBuilder — Architecture

Système de construction/placement réseau pour QANGA, par Iolacorp (`Copyright 2024 IOLACORP STUDIO`). Module `Runtime` unique (`QBuilder`, `LoadingPhase: Default`), entièrement en C++ (~32 fichiers source). Cette doc décrit la réalité du code, pas l'intention marketing : plusieurs sous-systèmes annoncés (réparation/maintenance, savegame interne, redimensionnement adaptatif de queue) existent à l'état de plomberie partielle ou désactivée, et c'est signalé explicitement.

## 1. But d'usage

QBuilder fournit l'ossature serveur-autoritaire qui permet aux joueurs de QANGA de poser et retirer des éléments de construction (murs, sols, props, structures) à l'échelle d'un monde-planète, sans saturer le réseau ni la mémoire. Le problème concret qu'il résout :

- **Échelle planétaire / coordonnées énormes** : les constructions ne sont pas posées en transform monde absolu (qui exploserait en float). Chaque construction vit dans un conteneur spatial (`AQBuilder_BuilderActor`) et les instances sont stockées en **transform relatif compressé sur int16** (`FQBuilder_Int16Transform`), borné à ±32767 cm autour du builder.
- **Densité massive** : pour les objets répétitifs et simples, on utilise des Instanced Static Meshes (`UQBuilder_ISM_IntanceComponent`) plutôt que des acteurs, afin de tenir des milliers d'instances par builder à coût de rendu marginal. Les objets complexes (interactifs, avec composants/logique) passent par le chemin acteur (`UQBuilder_ActorInstanceComponent`).
- **Burst de pose réseau** : poser un mur entier d'un coup génère des dizaines/centaines d'opérations. QBuilder les agrège dans des structures-queue compressées **Oodle** (`FQBuilder_Network_ToServer_Queue` / `FQBuilder_Network_ToClient_Queue`) pour ne pas multiplier les RPC.
- **Persistance du monde construit** : sérialisation binaire compacte + compression, avec rotation de fichiers de sauvegarde et hooks délégués pour brancher un système de savegame externe (celui de QANGA), plutôt que d'imposer le savegame interne du plugin.

C'est donc une infrastructure d'instanciation/réplication/persistance de constructions, pas un gameplay de construction prêt-à-l'emploi : le ciblage, la prévisualisation (ghost), le snapping fin et l'UI sont laissés aux Blueprints qui consomment l'API `QBuilder_BP_*` de `UQBuilder_Client`.

## 2. Vue d'ensemble / décisions d'architecture

Architecture en trois étages :

1. **`UQBuilder_SubSystem` (UWorldSubsystem)** — l'orchestrateur unique par monde. Il connaît le rôle réseau (`QBuilder_Standalone_Mode` / `QBuilder_IsServer`), tient les registres de builders et de clients, gère le pool d'ID serveur, la sauvegarde, et toutes les fonctions `QBuilder_ServerData_*` (autorité) et `QBuilder_OnClient_*` (application client).
2. **`UQBuilder_Client` (UActorComponent)** — composant attaché au pion/contrôleur joueur. Surface d'API Blueprint (`QBuilder_BP_*`), émetteur des RPC `Server,Reliable` vers le serveur et récepteur des RPC `Client,Reliable`. Détient la queue sortante serveur→client et la logique d'autorisation par `Client_ID` / `Allowed_By_ID` / `IsAdmin`.
3. **`AQBuilder_BuilderActor` (AActor)** — conteneur spatial répliqué. Détient deux stockages de données d'instances (`QBuilder_ISMData`, `QBuilder_ActorData` de type `FQBuilder_Instance_DataStorage`), la map des composants ISM par `Mesh_ID`, la map des acteurs-instances, les ressources et la vie du builder.

Décisions structurantes (vérifiées dans le code) :

- **Double stockage indépendant ISM / Actor.** Chaque builder maintient des collections séparées pour les instances ISM et les instances acteur, avec un système d'ID 16 bits à pooling (`FQBuilder_Instance_DataStorage::GetNewIndex` / `RemoveIndex`, plafond 65535). Le choix ISM vs Actor se fait par instance via le flag `FQBuilder_BP_AddInstance::Actorinstance`.
- **Compression réseau systématique.** Les deux structures-queue surchargent `NetSerialize` pour : (a) sérialiser en mémoire avec un champ de bits `SerializedArrays` n'émettant que les tableaux non vides, puis (b) compresser via `FCompression::CompressMemory(NAME_Oodle, …)`. C'est le cœur "queue compressée" du plugin.
- **Transform compressé int16.** `FQBuilder_Vector16` stocke position en int16 (cm) et rotation en int16 (degrés × 10). Conséquence : portée relative bornée et **perte de précision** (≈0,5 cm en position, 0,1° via le ×10, mais quantifié 360/128 côté grille BP). Le snapping grille par défaut (`FQBuilder_BP_GridTransform`) est calibré pour rester dans cette résolution (`LocationGrid = 0.5`, `RotationGrid = 2.8112 = 360/128`).
- **Custom data ISM packé.** `UQBuilder_ISM_IntanceComponent` force `NumCustomDataFloats = 3` et encode `[0]=ID`, `[1]=Life`, `[2]=Color` dans `PerInstanceSMCustomData`, avec mapping bidirectionnel `ID↔Index` maintenu à la main pour survivre aux `RemoveAtSwap`.
- **Autorité serveur, application optimiste client.** Le client envoie des intentions via RPC serveur ; le serveur valide (autorisation + ressources) et renvoie l'état appliqué via la queue client. En `NM_Standalone`, tout passe en local sans RPC (chemin `QBuilder_GetIsStandAloneMode`).
- **Init asynchrone tolérante au hot-reload.** `Initialize` détecte commandlet/cook et s'auto-désactive ; `LoadDefaultSettings` charge les DataBase via `StreamableManager` puis arme un `FTSTicker` qui attend la stabilisation (pas d'async-load, pas de recompile BP) avant de broadcaster `QBuilder_IsInit`. C'est une protection explicite contre le STRUCT_REINST / reconstruction d'instances.

État réel vs intention (honnêteté) :

- **Maintenance / réparation / `Repair_*`** : les facteurs (`QBuilder_Resource_Maintenance_Factor`, `QBuilder_Resource_AutoRepair*`, `QBuilder_Resource_Repair_Factor`) et les champs (`Simple_Maintenance_Price_PerHour`, `Repair_Price`, `Repair_Minute`) existent en données, mais aucun timer de maintenance/réparation n'est câblé dans le subsystem (seuls le prix de pose et le remboursement sont calculés). C'est de l'infrastructure prête, pas du gameplay actif.
- **Savegame interne** : `QBuilder_SaveGame_StartTimer()` est appelé en commentaire dans `Initialize`. Le chemin réellement utilisé par QANGA est le savegame **externe** par délégués (`QBuilderAutoSave` portant `TArray<FQBuilder_Builder_SaveGame>`).
- **`CustomDataMap` (FInstancedStruct)** : répliqué et présent dans la structure savegame, mais sa sérialisation binaire est **commentée** dans `FQBuilder_Builder_SaveGame::MSerialize/MDeserialize` (les lignes `CustomDataMap.Serialize(Ar)` sont en commentaire). Le custom data survit donc à la réplication mais **pas à la sauvegarde disque**.
- **Redimensionnement adaptatif de queue** : `Auto_AjustQueueSize` / `QBuilder_AjustQueueSize` (RPC `Server,Unreliable`) et `Queue_AutoAjustSizeFromPerformanceCheck` existent ; c'est une boucle opt-in, désactivée par défaut.
- **`FQBuilder_ActorInterface`** : interface BP optionnelle pour que les acteurs-instances réagissent à ID/transform/vie/couleur/init. C'est un point d'extension, pas un mécanisme obligatoire.

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQBuilder_SubSystem` | `UWorldSubsystem` (UCLASS) | Orchestrateur central : registres builders/clients, pool d'ID serveur, savegame, calcul de ressources, délégués d'intégration |
| `UQBuilder_Client` | `UActorComponent` (UCLASS) | Composant joueur : API Blueprint `QBuilder_BP_*`, RPC client↔serveur, queue sortante, autorisation, visibilité standalone |
| `AQBuilder_BuilderActor` | `AActor` (UCLASS) | Conteneur spatial répliqué : stockages d'instances ISM/Actor, ressources avancées, vie, transform serveur |
| `UQBuilder_ISM_IntanceComponent` | `UInstancedStaticMeshComponent` (UCLASS) | Composant ISM par mesh : ajout/suppression/maj d'instances avec ID 16 bits stable, custom data packé (ID/Life/Color) |
| `UQBuilder_ActorInstanceComponent` | `UActorComponent` (UCLASS) | Marqueur sur un acteur-instance spawné : porte `Actor_Instance_ID`, `Actor_Data`, vie, couleur, transform relatif |
| `UQBuilder_FunctionLibrary` | `UBlueprintFunctionLibrary` (UCLASS) | Helpers BP : accès subsystem, résolution DataMesh/DataActor par ID, conversion grille, savegame ZLIB (sync/async), requêtes ressources |
| `UQBuilder_DevSettings` | `UDeveloperSettings` (UCLASS, `config=Game`) | Réglages projet (catégorie "QBuilder") : DataBase soft-refs, classe builder, ressources, poids/échelle de queue, fréquence standalone |
| `IQBuilder_ActorInterface` / `UQBuilder_ActorInterface` | `UInterface` (UINTERFACE) | Interface BP optionnelle sur les acteurs-instances : `QBuilder_ActorInterface_CallID`, `QBuilder_ActorInterface_ChangeTransform`, `QBuilder_ActorInterface_ChangeLife`, `AQBuilder_ActorInterface_ChangeColor`, `QBuilder_ActorInterface_Init` |
| `UQBuilder_Data_MeshData` | `UDataAsset` (UCLASS) | Template d'élément ISM : mesh, matériaux override, vie/résistances, ressources de coût, échelle de base |
| `UQBuilder_Data_MeshDataBase` | `UDataAsset` (UCLASS) | Registre `int32 → UQBuilder_Data_MeshData` (table des meshes constructibles), résolution par ID |
| `UQBuilder_Data_ActorData` | `UDataAsset` (UCLASS) | Template d'acteur-instance : classe, règles de spawn (réplication/serveur/persistance), vie, ressources, vues mesh |
| `UQBuilder_Data_ActorDataBase` | `UDataAsset` (UCLASS) | Registre `int32 → UQBuilder_Data_ActorData`, résolution par ID |
| `UQBuilder_Data_ResourceData` | `UDataAsset` (UCLASS) | Définition d'une ressource avancée (asset-identité, sert de clé de TMap) |
| `UQBuilder_Data_ResourceDataBase` | `UDataAsset` (UCLASS) | Liste des `UQBuilder_Data_ResourceData` connus |
| `UQBuilder_Data_WorldSaveGame` | `USaveGame` (UCLASS) | Objet savegame : `TArray<FQBuilder_Builder_SaveGame>` + date |
| `FQBuilder_Vector16` / `FQBuilder_Int16Transform` | USTRUCT (NetSerialize) | Vecteur/transform compressés int16 (cm / deg×10), conversion vers `FVector`/`FRotator`/`FTransform` |
| `FQBuilder_Data_Instance` | USTRUCT (NetSerialize) | Donnée d'instance : `Data_ID` (uint16), transform compressé, `Life` (uint16), `Color` (uint8) |
| `FQBuilder_Instance_DataStorage` | USTRUCT | Stockage d'instances + pool d'index libres + (dé)sérialisation mémoire avec bornes de validation |
| `FQBuilder_Instance_UpdateTransform` / `…UpdateLife` / `…UpdateColor` | USTRUCT (NetSerialize) | Deltas de mise à jour ciblés par ID |
| `FQBuilder_Resource_Data` | USTRUCT (BlueprintType) | Coûts simple+avancé, facteurs remboursement, prix/durée réparation |
| `FQBuilder_Life_Data` | USTRUCT (BlueprintType) | Vie totale + résistances par `FName` (0=immunité, 1=plein dégât) |
| `FQBuilder_Network_ToServer_Queue` / `FQBuilder_Network_ToClient_Queue` | USTRUCT (NetSerialize) | Queues d'opérations agrégées, bitfield des tableaux + compression Oodle |
| `FQBuilder_Network_Client_AddInstance` / `…Server_AddInstance` | USTRUCT (NetSerialize) | Payloads d'ajout d'instance (sens client→serveur / serveur→client) |
| `FQBuilder_Network_Server_Builder_BasicInfo` | USTRUCT (BlueprintType) | Info légère d'un builder pour découverte côté client (ID, transform, owner, nb instances) |
| `FQBuilder_Network_Server_ReplicatedAdvancedResource` | USTRUCT (NetSerialize) | Ressource avancée répliquée (ressource + quantité int64) |
| `FQBuilder_Builder_SaveGame` | USTRUCT (BlueprintType) | Snapshot complet d'un builder pour sauvegarde (ressources, vie, transform, ISM_DATA, ACTOR_DATA) |
| `FQBuilder_SaveGame_Header` | USTRUCT (BlueprintType) | En-tête de fichier de sauvegarde (date, nom, totaux, version) |
| `FQBuilder_BP_AddInstance` / `…RemoveInstance` / `…ChangeTransform` / `…ChangeColor` / `…GridTransform` | USTRUCT (BlueprintType) | Structures d'entrée de l'API Blueprint du client |
| `FQBuilder_ISM_InternalData` | USTRUCT | Donnée locale par instance ISM (vie + couleur) |
| `FQBuilder_Actor_MeshView` | USTRUCT (BlueprintType) | Aperçu mesh+transforms d'un ActorData (preview/ghost) |
| `FQBuilder_Base_FInstanceStruct` | USTRUCT (BlueprintType) | Type de base du `CustomDataMap` (FInstancedStruct) du builder |
| `FQBuilder_ClientArray` | USTRUCT | Wrapper `TSet<UQBuilder_Client*>` pour la map builder→clients |
| `QBuilderLegacyPayload::TryDeserializeObjectPayload` | namespace/fonction | Lecture des anciennes sauvegardes (payload UObject UE5.3.2) lors de la migration |

## 4. Flux de données et cycle de vie

### Initialisation (subsystem)
1. `UQBuilder_SubSystem::Initialize` : si commandlet/cook → désactivation immédiate. Sinon détermine le rôle (`UKismetSystemLibrary::IsStandalone` / `IsServer`) et fixe `QBuilder_Standalone_Mode` / `QBuilder_IsServer`.
2. `LoadDefaultSettings` recopie les réglages de `UQBuilder_DevSettings` et lance un `RequestAsyncLoad` des soft-refs (ActorDataBase, MeshDataBase, ResourceDataBase, BuilderClass).
3. À la fin du chargement, `ActorDataBase`/`MeshDataBase`/`ResourceDataBase`/`Builder_Class` sont résolus, `Is_Initialized = true`, puis un `FTSTicker` attend ~2 s de stabilité (ni async-load, ni `GCompilingBlueprint`, ni `GIsReconstructingBlueprintInstances`), force un `CollectGarbage`, et broadcaste `QBuilder_IsInit`. Le gameplay BP appelle ensuite `QBuilder_SetSubSystem_Ready()` qui broadcaste `QBuilder_IsReady`.

### Enregistrement runtime
- **Client** : `QBuilder_Client_RegisterClient` insère dans `Clients` ; le client local est `Local_Client`. L'autorisation builder↔client passe par `Builder_ToClient` (`FQBuilder_ClientArray`) + `QBuilder_Builder_GetClientAutorized` (vérifie `QBuilder_PlayerOwner` vs `Client_ID`/`Allowed_By_ID`/`IsAdmin`).
- **Builder** : à `AQBuilder_BuilderActor::BeginPlay`, l'acteur fixe `SetNetCullDistanceSquared`, récupère le subsystem et s'enregistre (`QBuilder_Builder_Register`). Côté `NM_Client`, il demande ses données au serveur (`QBuilder_ToServer_ClientRequest_BuilderData`) et applique `ServerTransformFull` (transform répliqué) ; côté `NM_DedicatedServer`, il capture sa transform dans `ServerTransformFull`.

### Boucle d'opérations (build/remove/update)
1. **Intention BP (client)** : `QBuilder_BP_AddInstance` / `RemoveInstance` / `ChangeTransform` / `ChangeColor` / `CreateBuilder(WithData)` sur `UQBuilder_Client`. Les transforms monde sont convertis en relatif+grille (helpers `QBuilder_ConvertorRelative_*`) puis compressés int16.
2. **Envoi** : agrégation dans `FQBuilder_Network_ToServer_Queue`, compressée Oodle, transmise par `QBuilder_ToServer_CompressData` (`Server,Reliable`).
3. **Validation serveur** : le subsystem applique via `QBuilder_ServerData_ISM_*` / `QBuilder_ServerData_Actor_*` (add/remove/changeTransform/changeLife/changeColor), avec `QBuilder_Resource_Price_Compute` / `QBuilder_Resource_Refund` quand les ressources sont actives.
4. **Diffusion** : l'état appliqué est renvoyé aux clients autorisés via `FQBuilder_Network_ToClient_Queue` (compressée) → `QBuilder_ToClient_CompressData`. Le client applique par `QBuilder_OnClient_ISM_*` / `QBuilder_OnClient_Actor_*`.
5. **Application sur le builder** : `AQBuilder_BuilderActor::QBuilder_ISM_*` crée/peuple les `UQBuilder_ISM_IntanceComponent` (un par `Mesh_ID`, via `QBuilder_ISM_CreateISM`) ; `QBuilder_Actor_*` spawne/retire les acteurs-instances selon les règles de `UQBuilder_Data_ActorData`.

### ISM : gestion d'ID stable
`UQBuilder_ISM_IntanceComponent` maintient `ISM_PerInstanceID_ToIndex` / `ISM_PerInstanceIndex_ToID` pour garder un `ID` 16 bits stable malgré le `RemoveAtSwap` interne d'UE. Le custom data packe ID/Life/Color dans `PerInstanceSMCustomData` (3 floats par instance) pour le matériau et les lookups.

### Standalone
En `NM_Standalone`, pas de RPC : `QBuilder_OnStandAlone_StartTimer` lance un timer (fréquence `QBuilder_OnStandAlone_UpdateFrequency`) qui met à jour la visibilité des builders autour de `QBuilder_OnStandAlone_GetPlayerLocation` (BlueprintNativeEvent surchargeable) et charge/décharge `StandAlone_ClientLoaded_Builders`.

### Sauvegarde
- `QBuilder_SaveGame_CreateData` collecte les builders en `TArray<FQBuilder_Builder_SaveGame>`. Sérialisation binaire (`QBuilder_SaveData_SerializeData`), compression (`QBuilder_SaveData_CompressData`), écriture fichier avec en-tête `FQBuilder_SaveGame_Header` et rotation (`QBuilder_AutoSaveGameRotationMax`, `QBuilder_SaveData_ComputeSaveName` / `IncrementRotation`).
- Auto-save : timer `QBuilder_AutoSave_Timer` → `QBuilder_SaveGame_AutoSaveGame`. Si `QBuilder_AutoSaveGame_UseExternalSaveGame`, broadcaste `QBuilderAutoSave(Data)` au lieu d'écrire en interne.
- Chargement : `QBuilder_SaveData_LoadFromFile` / `QBuilder_SaveGame_LoadData` → `QBuilder_SaveGame_CreateBuilder` par builder. `QBuilderLegacyPayload::TryDeserializeObjectPayload` gère les anciens fichiers UE5.3.2.

### Teardown
`Deinitialize` : sauvegarde optionnelle on-destroy (`QBuilder_AutoSaveGame_OnDestroy`), retrait du `FTSTicker`, clear des délégués `QBuilder_IsInit`/`QBuilder_IsReady`. `AQBuilder_BuilderActor::EndPlay` désenregistre via `QBuilder_Builder_UnRegister`. Note : le corps de `BeginDestroy` (builder) est entièrement commenté — le désenregistrement réel passe par `EndPlay`.

## 5. Réplication / réseau

Le système est **serveur-autoritaire** et réplique réellement (vérifié : `Replicated`, `ReplicatedUsing`, `GetLifetimeReplicatedProps`, RPC `Server`/`Client`/`NetMulticast`).

### Modèle d'autorité
- Le **serveur** détient la vérité : pool d'ID builder (`Server_Builders`, `Server_Builder_PoolIndex`, `QBuilder_ServerBuilder_GetNewIndex`), application des opérations (`QBuilder_ServerData_*`), calcul/débit des ressources, et diffusion aux clients autorisés.
- Le **client** émet des intentions et applique l'état reçu (`QBuilder_OnClient_*`). Il ne crée pas d'autorité ; ses builders locaux (`Builders`, `StandAlone_ClientLoaded_Builders`) sont peuplés à partir du serveur (ou directement en standalone).
- L'**autorisation** par builder repose sur `QBuilder_PlayerOwner` (FName) confronté à `UQBuilder_Client::Client_ID`, `Allowed_By_ID` et `IsAdmin` (tous `Replicated`).

### État répliqué
- `UQBuilder_Client` : `Client_ID`, `Allowed_By_ID`, `IsAdmin`.
- `AQBuilder_BuilderActor` : `QBuilder_BuilderID`, `QBuilder_PlayerOwner`, `Builder_Simple_Resource`, `Builder_Life`, `QBuilder_Model`, `CustomDataMap` (`FInstancedStruct`), `ServerTransformFull`, et `Replicated_AdvancedResource` (`ReplicatedUsing = OnRep_AResUpdate`, qui reconvertit vers la map `Builder_Advanced_Resource` côté client).
- `SetNetCullDistanceSquared(QBuilder_Culling_SquareDistance)` (défaut 250000000000 = ~5 km de rayon) borne la pertinence réseau des builders.

### RPC (vérifiés dans `UQBuilder_Client.h`)
- **Client → serveur** (`Server,Reliable`) : `QBuilder_ToServer_CompressData`, `QBuilder_ToServer_ClientRequest_BuilderData`, `QBuilder_ToServer_ClientRemove_Builder`, `QBuilder_ToServer_CreateBuilder`, `QBuilder_ToServer_CreateBuilderWithBuild`, `QBuilder_ToServer_DestroyBuilder`, `QBuilder_ToServer_GetVisibleBuilder`. Plus `QBuilder_AjustQueueSize` (`Server,Unreliable`).
- **Serveur → client** (`Client,Reliable`) : `QBuilder_ToClient_CompressData`, `QBuilder_ToClient_RefundResource`, `QBuilder_ToClient_ReturnVisibleBuilder`, `QBuilder_ToClient_ReturnCreatedBuilder`.
- **Multicast** (`AQBuilder_BuilderActor`) : `QBuilder_Net_RequestDestroy` (`NetMulticast,Reliable`).

### Compression & burst
Les payloads volumineux passent par `FQBuilder_Network_ToServer_Queue` / `FQBuilder_Network_ToClient_Queue` : bitfield `SerializedArrays` (n'émet que les tableaux non vides) + compression `NAME_Oodle`. Le client agrège les opérations dans une queue pondérée (`Queue_*_weight`, `Queue_MaxLength`) avant envoi, pour absorber un mur posé d'un coup. Le remboursement de ressources vers l'inventaire est délégué côté client via `QBuilder_RefundResource` (`BlueprintNativeEvent`, point d'extension d'intégration inventaire).

## 6. Points d'intégration

### Dépendances du plugin
- `.uplugin` : aucune dépendance plugin déclarée (`Plugins: []`). Le CLAUDE.md du plugin mentionne StructUtils, mais le `Build.cs` ne le liste pas explicitement (les types `FInstancedStruct` viennent de `StructUtils/InstancedStruct.h`, fourni par le moteur/Engine en 5.7).
- `Build.cs` : public `Core`, `DeveloperSettings`, `Serialization` ; privé `CoreUObject`, `Engine`, `Slate`, `SlateCore`. Compression Oodle via `FCompression`/`NAME_Oodle` (Engine).

### Branchement dans QANGA
QBuilder est piloté **par configuration et par Blueprint**, pas par code C++ d'un autre module (aucune référence aux types `QBuilder_*` dans `G:/QANGA/Source` ni dans les autres plugins). Les points de contact réels :
- `Config/DefaultGame.ini`, section `[/Script/QBuilder.QBuilder_DevSettings]` : assigne les DataBase et la classe builder du projet :
  - `ActorDataBase=/Game/Systems/QBuilder/Data/QBuilder_Qanga_ActorData`
  - `MeshDataBase=/Game/Systems/QBuilder/Data/QBuilder_Qanga_MeshData`
  - `ResourceDataBase=/Game/Systems/QBuilder/Data/QBuilder_Qanga_ResourceData`
  - `BuilderClass=/Game/Systems/QBuilder/Actor/QBuilder_Builder_Actor_BP.QBuilder_Builder_Actor_BP_C`
  - `QBuilder_Resource_UseAdvanced=True`, `QBuilder_Resource_FreeMode=False`
- Le gameplay (ciblage, ghost, snapping, UI) vit dans le Content `/Game/Systems/QBuilder/…` et consomme `UQBuilder_FunctionLibrary::QBuilder_GetSubSystem` + l'API `UQBuilder_Client::QBuilder_BP_*`.
- Intégration savegame/inventaire de QANGA : par délégués (`QBuilderAutoSave`, `QBuilderAutoSaveCall`, `QBuilderOn/Off`, `QBuilderRegister/UnRegister`) et par surcharge de `QBuilder_RefundResource`.

### Réglages clés (`UQBuilder_DevSettings`, catégorie projet "QBuilder")
- Données : `BuilderClass`, `ActorDataBase`, `MeshDataBase`, `ResourceDataBase`, `EnabledISMInstance`, `EnabledActorInstance`, `BuildMeshCastShadow`, `BuildMeshStartCullDistance`, `BuildMeshEndCullDistance`.
- Ressources : `QBuilder_Resource_UseAdvanced`, `…Price_Factor`, `…Refund_Factor`, `…Maintenance_Factor`, `…AutoRepair`, `…AutoRepairFrequency`, `…Repair_Factor`, `…FreeMode`.
- Réseau client : `Queue_Remove_weight`, `Queue_AddISM_weight`, `Queue_AddActor_weight`, `Queue_Change_weight`, `Queue_MaxLength`, `Queue_AutoScaling`, `QBuilder_Dedicated_NetworkChargeMin/Max`.
- AutoSave : `QBuilder_AutoSaveGame`, `QBuilder_AutoSaveGame_Time`. Standalone : `QBuilder_OnStandAlone_UpdateFrequency`.

Aucune CVar ni commande console n'est déclarée par le plugin.

## 7. Gotchas, invariants et pièges

- **Portée relative bornée à int16.** Toute instance posée doit rester dans ±32767 cm (~327 m) du `AQBuilder_BuilderActor` qui la porte. Au-delà, `FQBuilder_Vector16` overflow silencieusement. Le builder est le repère LWC qui rend les coordonnées planétaires viables — ne jamais poser en transform monde absolu.
- **Perte de précision assumée.** Rotation stockée en deg×10 (int16) et grille BP par défaut à 360/128 ≈ 2,8112°. Position quantifiée au cm. Concevoir le snapping en conséquence ; ne pas attendre un placement sub-centimétrique.
- **ID 16 bits avec pooling — plafond 65535.** `FQBuilder_Instance_DataStorage::GetNewIndex` renvoie -1 à saturation. Idem pour les builders serveur (pool d'index). Au-delà, l'ajout échoue silencieusement côté data.
- **L'index 0 est réservé.** `FQBuilder_Instance_DataStorage` ajoute un élément par défaut au constructeur ; l'ID 0 n'est pas une instance valide. Les lookups partent à 1.
- **Le custom data du builder n'est pas persisté.** `CustomDataMap` (`FInstancedStruct`) est répliqué mais sa sérialisation disque est commentée dans `FQBuilder_Builder_SaveGame`. Ne pas compter dessus pour de l'état devant survivre à un reload serveur.
- **Maintenance/réparation = données sans moteur.** Les facteurs et champs `Maintenance`/`Repair` sont lus en config mais aucun système périodique ne les applique. Ne pas présenter ça comme un gameplay actif.
- **Init différée de ~2 s + GC forcé.** `QBuilder_IsInit` n'est broadcasté qu'après stabilisation (anti STRUCT_REINST/recompile BP) et un `CollectGarbage`. Tout consommateur BP doit s'abonner à `QBuilder_IsInit`/`QBuilder_IsReady` et ne rien faire avant — `Is_Initialized`/`QBuilder_GetSubSystem_IsReady` reflètent ces deux jalons distincts.
- **Désactivation en commandlet/cook.** `Initialize` court-circuite sous commandlet/cook : le subsystem existe mais `Is_Initialized=false`. Ne pas l'appeler en pipeline offline.
- **`BeginDestroy` du builder est neutralisé.** Le désenregistrement réel passe par `EndPlay` (`QBuilder_Builder_UnRegister`). Le corps de `BeginDestroy` est entièrement commenté — ne pas s'y fier pour du cleanup.
- **`Server_Builders` est un TArray à trous.** Le pool d'index serveur réutilise des slots ; itérer dessus implique de filtrer les entrées nulles/recyclées.
- **ISM = `UInstancedStaticMeshComponent`, pas HISM.** L'héritage HISM est présent en commentaire mais inactif. Pas de culling/LOD hiérarchique automatique côté instances ISM ; le culling se fait au niveau builder (net cull distance + visibilité standalone).
- **Net cull distance énorme par défaut.** ~5 km de rayon (`QBuilder_Culling_SquareDistance = 250000000000`). Sur un dédié à fort peuplement, c'est un levier de charge réseau à régler par builder/modèle.
- **Standalone ≠ serveur.** Trois chemins distincts (`QBuilder_Standalone_Mode`, `QBuilder_IsServer`, client pur). Le code de pose diverge : standalone applique en local sans RPC. Tester les trois modes séparément.
- **Dépendance StructUtils non déclarée dans `Build.cs`.** Le code inclut `StructUtils/InstancedStruct.h` et `StructSerializer.h` ; ça compile en 5.7 via Engine, mais c'est un couplage implicite à surveiller en cas de migration moteur.

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QBuilder/QBuilder.Build.cs` | Dépendances de module (Core, DeveloperSettings, Serialization, Engine, Slate) |
| `Source/QBuilder/Private/QBuilder.cpp` / `Public/QBuilder.h` | Implémentation `IModuleInterface` (vide ; pas de startup logic) |
| `Source/QBuilder/Public/QBuilder_SubSystem.h` / `Private/QBuilder_SubSystem.cpp` | Subsystem central : init async, registres, ressources, savegame, ServerData/OnClient, legacy payload |
| `Source/QBuilder/Public/QBuilder_Client.h` / `Private/QBuilder_Client.cpp` | Composant joueur : API BP, RPC, queue sortante, autorisation, visibilité standalone |
| `Source/QBuilder/Public/World/QBuilder_BuilderActor.h` / `Private/.../QBuilder_BuilderActor.cpp` | Conteneur spatial répliqué : stockages ISM/Actor, ressources avancées, vie, transform |
| `Source/QBuilder/Public/World/QBuilder_ISM_IntanceComponent.h` / `Private/.../*.cpp` | Composant ISM : ID stable, custom data packé ID/Life/Color, override add/remove |
| `Source/QBuilder/Public/World/QBuilder_ActorInstanceComponent.h` / `Private/.../*.cpp` | Marqueur d'acteur-instance : ID, data, vie, couleur, transform relatif |
| `Source/QBuilder/Public/World/QBuilder_ActorInterface.h` | Interface BP optionnelle des acteurs-instances |
| `Source/QBuilder/Public/QBuilder_FunctionLibrary.h` / `Private/.../*.cpp` | Helpers BP : subsystem, résolution data par ID, conversion grille, savegame ZLIB |
| `Source/QBuilder/Public/QBuilder_DevSettings.h` / `Private/.../*.cpp` | `UDeveloperSettings` (config=Game) : data, ressources, réseau, standalone |
| `Source/QBuilder/Public/Struct/QBuilder_Struct.h` | Types compressés (Vector16/Int16Transform), structs BP d'entrée, header savegame |
| `Source/QBuilder/Public/Struct/QBuilder_Struct_Data.h` | Instance/updates/ressources/vie/stockage/savegame builder |
| `Source/QBuilder/Public/Struct/QBuilder_Struct_Network.h` | Queues réseau compressées Oodle + payloads add/ressource |
| `Source/QBuilder/Public/Data/QBuilder_Data_MeshData(.h/Base.h)` + `Private/.../*.cpp` | Templates ISM + registre par ID |
| `Source/QBuilder/Public/Data/QBuilder_Data_ActorData(.h/Base.h)` + `Private/.../*.cpp` | Templates acteur + registre par ID |
| `Source/QBuilder/Public/Data/QBuilder_Data_ResourceData(.h/Base.h)` | Ressources avancées + liste |
| `Source/QBuilder/Public/Data/QBuilder_Data_WorldSaveGame.h` / `Private/.../*.cpp` | Objet `USaveGame` du monde construit |
| `Config/DefaultGame.ini` (projet) | Assignation des DataBase/BuilderClass QANGA et flags ressources |
| `Content/Systems/QBuilder/…` (projet, `/Game/Systems/QBuilder`) | Assets QANGA : data, classe builder BP, gameplay/UI de construction |
