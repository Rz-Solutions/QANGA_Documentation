# Recel de véhicules au garage Voss : état mesuré et proposition

> Rapport du 2026-11-21, demandé par Benjamin. **Document à valider par RzZz avant tout code. Aucune modification du
> projet n'a été faite.**
> Méthode : lecture du C++ des plugins, et extraction de noms dans les `.uasset` par `grep -a` (les graphes Blueprint
> n'ont PAS été lus, l'éditeur n'a pas été lancé). Les documentations existantes n'ont servi qu'à s'orienter.
>
> **Niveau de preuve de chaque fait** :
> - **[A]** relu dans le C++, ligne citée.
> - **[B]** nom de variable ou de fonction trouvé dans un `.uasset`. Prouve que le nom existe, pas ce que fait le graphe.
> - **[C]** non vérifié, ou conclusion tirée d'une recherche qui n'a rien rendu. **Une recherche vide n'est pas une
>   preuve d'absence** : chaque [C] est une question à RzZz (§6).

---

## 1. La demande (Benjamin)

Un gameplay de revente de véhicules volés, à l'échelle de tout l'univers :

1. Des véhicules (voitures, vaisseaux) sont garés un peu partout : Terre, autres planètes, stations, espace.
2. Le joueur peut les déverrouiller et les démarrer. C'est possible sans équipement, mais un module dédié, à
   plusieurs niveaux, rend la chose beaucoup plus facile.
3. Il les amène à un garage de la faction Voss, qui les rachète contre de l'argent.
4. Il gagne des points de faction Voss, qui ouvrent des missions.

**Précision de Benjamin (21/11)** : pas un garage unique, mais **un réseau de garages normalisés**. Il y en a un peu
partout sur les planètes, surtout sur Terre, et une version en station dans l'espace. Ils rachètent les véhicules
terrestres comme les vaisseaux. Où que soit le joueur, il doit pouvoir se faire une boucle crochetage puis revente
sans traverser le système solaire.

---

## 2. Ce que le code donne déjà

### 2.1 Véhicules
| Fait | Preuve |
|---|---|
| Registre des véhicules dans `Content/Systems/References/DA_AllRef` : tables `Id:VehicleClass`, `VehicleClass:Price`, `VehicleClass:Tags`, `VehicleClass:AllowedAdmin`, `VehicleCardItem:Vehicle`. Environ 45 véhicules distincts | [B] |
| Possession : composant `Content/Systems/Vehicle/VehiclesOwned/VehiclePlayerOwner` (`OwnerPlayerId`, `OnRep_OwnerPlayerId`, `SetOwnerId`) | [B] |
| Flotte du joueur sur le PlayerState : `PlayerVehiclesComponent_C` (`OwnedVehicles`, `CurrentOwnedVehicle`, `VehiclesOnGarage`, `ServerAddOwnedVehicle`, `ServerRemoveOwnedVehicle`, `RequestBuyVehicle`, `RequestExchangeCard`) | [B], lecture C++ dans `Plugins/QModule/Source/QModule/Private/QModule_VehicleDeliveryActor.cpp:360-373` [A] |
| Moteur : `VehicleComponent.VehicleState` (`E_VehicleState` : Off, On, Broken, Destroyed), dispatcher `OnVehicleStateUpdate`, RPC `VehicleBase.SV_OnOffToggle` | [B] ; seule la valeur `On = 1` est confirmée en C++, `QModule_VehicleDeliveryActor.cpp:45-46` [A] |
| Portes : `VehicleBase.MainDoorOpen` (`OnRep_MainDoorOpen`), état serveur | [A] `QModule_VehicleDeliveryActor.cpp:1263` et `:1333` |
| Réseau : `VehicleBase` ne réplique pas son mouvement, culling à 300 m | [A] commentaire `QModule_VehicleDeliveryActor.h:55-56` (mesure CDO d'août, non refaite) |
| Mouvement simulé par le joueur le plus proche : `UClientAuthorityComponent` (`AuthorityPlayer`, `AuthorityPlayerOverride`, répliqués). Le subsystem choisit le pion de joueur le plus proche ; au-delà de `MaxDistanceKm` (1 km par défaut) le composant lance `OnOutRangeClientAuthority` | [A] `Plugins/ClientAuthority/Source/ClientAuthority/Public/ClientAuthorityComponent.h:18-22`, `:52` ; `ClientAuthoritySubSystem.cpp:40-71` (`FindNearestPawnFromLocation`) ; `ClientAuthorityComponent.cpp:114` |
| Chaque `VehicleBase` s'inscrit au `WorldVehiclesManager` (`AddSpawnedVehicle`), qui porte `VehicleMaxDistanceCm`, `DestroyedVehicleTimeout`, `PlayerVehicleBlockUnspawn` | [B] (seuls ces 2 assets citent `AddSpawnedVehicle` dans Systems, GameplayActors, GameMode, Vehicles) |
| Terminal de garage : `Content/GameplayActors/VehicleSpawner/BP_TerminalVehicleSpawner` (`CheckVehicleOnPlatform`, `HasVehicleToReturn`, `Garage_VehicleKept`, `Garage_DuplicateReward`, lit `OwnerPlayerId`) | [B] |

### 2.2 Interaction, argent, police
| Fait | Preuve |
|---|---|
| Interaction maintenue validée serveur : `UQPlayerControllerInteractComponent` (`bIsHoldInteract`, `HoldDurationSeconds`, `SV_Interact`) | [A] `Plugins/QSystem/Source/QSystem/Public/Component/QPlayerControllerInteractComponent.h:241-246`, `:310` |
| Mini-jeu `Content/Systems/Hacking_Sys/Hacking_Bridge` | [B] existe ; aucun appelant trouvé dans Systems, Widget, Characters, GameMode [C] |
| Argent : `InventoryComponent_C.Coins` (int64), `AddCoins`, `RemoveCoins`, `OnRep_Coins` | [B] |
| Patron de paiement depuis le C++ : `UQModuleBounty_World_SubSystem::PayBounty` (réflexion sur `AddCoins`) | [A] `Plugins/QModule/Source/QModule/Private/QModuleBounty_World_SubSystem.cpp:403-431` ; même patron dans `Plugins/QPolice/Source/QPolice/Private/QPaymentArea.cpp:364` |
| Délit `ECrimeType::Theft`, 12 points | [A] `Plugins/QPolice/Source/QPolice/Public/QPoliceTypes.h:26` et `:196` |
| `Theft` est traité (zone sans loi, téléportation, intranet) mais aucun émetteur trouvé en C++ | [A] pour les 3 lecteurs (`QTerritoryDetectionComponent.cpp:431`, `QEnhancedTeleportationComponent.cpp:508`, `RzIntranetTelemetry.cpp:93`) ; émetteur Blueprint [C] |
| L'API d'émission existe : `UQPoliceLibrary::ReportCrime` / `ReportCrimeAtLocation` (Blueprint) et `UQPoliceSubsystem::ReportCrime(Criminal, ECrimeType, Location, ...)`. Émettre `Theft` est un appel, pas un système à créer | [A] `QPoliceLibrary.h:16-28`, `QPoliceSubsystem.h:104-106` |
| Serveur sans mesh : une interaction validée par le serveur mesure la portée sur un `UQInteractionVolumeComponent` posé sur la cible ; sans ce volume, le serveur dédié refuse | [A] `QPlayerControllerInteractComponent.cpp:1483-1535` |

### 2.3 Faction Voss, modules, missions
| Fait | Preuve |
|---|---|
| `EQAI_Faction::Voss = 7`. La faction attaque None (donc les joueurs), IcLabs, Infected, Rogue | [A] `Plugins/QAI/Source/QAI/Public/Faction/QAI_Faction.h:37-41` et `:71-73` |
| La faction est une matrice d'hostilité fixe. Aucun compteur de réputation du joueur trouvé (cherché : `Reputation`, `FactionPoint`, `FactionScore`, `FactionStanding` dans `Plugins/*/Source` et Systems, GameMode, Characters, Widget) | [C] ; rien non plus dans `Config/*.ini`, et la seule occurrence C++ de `reputation` est une icône de texte de quête (`QuestTextFormatter.cpp:927`) [A] |
| Les zones safe existent déjà : volumes `WV_SafeArea` posés dans l'univers. `QCombatPolicy` refuse le combat quand la source ou la cible est dans une zone protégée, et `UQModule_SafeZoneLibrary` donne le même verdict au client et au serveur | [A] `QCombatPolicy.cpp:319-325`, `QModule_SafeZoneLibrary.h` ; comportement des Voss dans une zone à confirmer en jeu [C] |
| Coquille `Content/Phases/QModuleV2/.../QMD_AccreditationVoss` : 3 niveaux déjà rédigés (prix d'achat Voss -5 %, -10 %, -15 % et ouverture du rayon réservé). Aucun effet déclaré | [B] |
| Coquille `QMD_KitDeSabotage` : « désactive une porte ou une tourelle après 6 s d'interaction, 2 charges », puis 4 s et 2 s. Aucun effet déclaré | [B] |
| Patron d'un module à niveaux : `UQModule_Definition` (`MaxLevel`, `StatMods` avec `ValuePerLevel`, `BehaviorGrants`, `LevelDescriptions`), lecture par `UQModule_StatLibrary::QMOD_GetStat` et `QMOD_GetModuleLevelForActor` | [A] `Plugins/QModule/Source/QModule/Public/QModule_Definition.h`, `QModule_StatLibrary.h` |
| Contrats procéduraux DQS : 10 gabarits (Bounty, Recovery, Salvage, DataHeist, Extermination, Holdout, Sabotage, Uplink, Mining, Courier), générés par `UProceduralContractBoard` | [A] `Plugins/DynamicQuestSystem/Source/DynamicQuestSystemObjectives/Private/Contracts/ContractTemplates.h:34-148` |
| Aucun gabarit ni objectif « livrer un véhicule » trouvé dans les headers DQS | [C] |
| Drapeaux par joueur persistés dans la sauvegarde DQS : `SetPlayerFlag` / `GetPlayerFlag` (valeur `FString`) | [A] `.../Subsystems/QuestManagerSubsystem.h:717-721` |
| `UCheckPlayerFlagCondition` teste un drapeau pour l'accès à une quête : **présence seule**, la valeur n'est pas lue. Un palier de réputation doit donc être son propre drapeau, ou passer par une nouvelle condition qui lit la valeur | [A] `DynamicQuestSystemObjectives/Private/Conditions/CheckPlayerFlagCondition.cpp:69-74` (`HasPlayerFlag`) |
| `UTriggerZoneComponent` lit `GetFirstPlayerController()` : pas utilisable tel quel sur serveur dédié | [A] `Plugins/QTriggerZone/Source/QTriggerZone/Private/Components/TriggerZoneComponent.cpp:352`, `:588` |
| Ancien système Blueprint : `Content/Systems/Contract` (`ContractBase`, `CO_ReturnToDealer`) et `Content/Systems/Mission` (`MissionScriptBase`, qui lit `OwnerPlayerId`) | [B], état d'usage en jeu [C] |

### 2.4 Véhicules présents dans le monde aujourd'hui
| Source | Pilotable par un joueur ? | Preuve |
|---|---|---|
| Trafic aérien QAI : vraies classes `Velkara_Passenger`, `Vintage_Velkara_Passenger`, `Velkara_Explorer`, `IronDee_Lavrik`, `Melrose_Orizaune`, `SingleST`, `StormTravel`. **Pas détruits** : au-delà de la distance de despawn de tous les joueurs, un vaisseau est retiré et garé dans un pool de 12 coques réutilisées, détruit seulement si le pool est plein. Ce sont des BP de vaisseaux pilotables complets, allégés par QAI (ticks coupés en dur par `bCanEverTick = false`, invoker de collision WorldScape retiré, physique coupée côté présentation) | à tester [C] ; un vol demanderait un chemin de remise en état pilotable | [A] `Plugins/QAI/Source/QAI/Private/Spaceship/QAI_AerialTrafficSettings.cpp:34-40`, `QAI_AerialTrafficSubsystem.cpp:442-505`, `:154` ; `QAI_SubSystem_Spaceship.cpp:66-74`, `:288-300`, `:494-548` |
| Trafic au sol Mass (`Mass_Traffic_HovercraftBase`) | non, ce n'est pas un `VehicleBase` | [B] ; plugin `Traffic` désactivé [A] `QANGA.uproject:433-434` |
| `Content/Vehicles/Epaves` | non, maillages statiques seulement | [B] |
| Variantes `*_NoDrive` | non, par leur nom | [C] |
| Véhicules pilotables posés à la main dans les levels ou les QLevel | **non mesuré** : la recherche n'a couvert qu'une partie de `Content/Maps` | [C] |

### 2.5 QLevel (support imposé par Benjamin pour les garages et le parc)
| Fait | Preuve |
|---|---|
| QLevel est actif : `Enabled=True`, `WorldMap=()` vide | [A] `Config/DefaultGame.ini:359-364` |
| **Le serveur dédié fait tourner QLevel**, avec ses propres réglages (`Tick_*_OnDedicated`, `MaxComplexityPerSecond_OnDedicated`, `Loading_Deferred*_OnDedicated`) | [A] `Plugins/QLevel/Source/QLevel/Private/QLevel_SubSystem.cpp:654-677` |
| Sur serveur dédié, seul le proxy visuel est sauté. Les acteurs d'instance sont créés avec le GameState comme propriétaire | [A] `QLevel_Actor_Instance.cpp:741`, `:761`, `:888`, `:901` ; `QLevel_SubSystem_Instance.cpp:221-225` |
| Un `UQLevel_Asset` porte des données libres `AdditionalData` (`TMap<FName,FInstancedStruct>`), lisibles par `QLevel_GetAdditionalData` | [A] `Plugins/QLevel/Source/QLevel/Public/Data/QLevel_Asset.h:34`, `QLevel_Streaming_Instance.h:60-62` |
| Les instances s'enregistrent par `QLevel_RegisterLevelActor` ; `QLevel_GetAllInstancesInLevel` les liste dans un volume | [A] `QLevel_SubSystem.h:139`, `QLevel_Function_Library.h:71` |
| Le monde qui a l'autorité (serveur dédié, serveur d'écoute, solo) charge les sous-niveaux autour de la **position actuelle du pion de chaque joueur**, sans anticipation. Le code prévoit une avance d'une seconde de vitesse (et un mode proxy seul au-delà d'environ 224 m/s) pour les pions qui implémentent `IQLevel_PawnInterface`, mais aucune classe C++ ne l'implémente et le `Cast` du subsystem ne voit pas une implémentation Blueprint : ce chemin ne sert pas aujourd'hui. Sur serveur dédié : état réévalué toutes les 0,25 s, ni affichage ni déchargement différés, budget de 16 000 de complexité par seconde | [A] `QLevel_SubSystem.cpp:1549-1620`, `:1857` ; `QLevel_PawnInterface.h` ; `Config/DefaultGame.ini` (`Tick_State_Interval_OnDedicated`, `Loading_Deferred*_OnDedicated`, `MaxComplexityPerSecond_OnDedicated`) |
| Une graine par instance (`Seed`) permet de varier un même QLevel d'un endroit à l'autre | [A] `QLevel_Actor_Instance.h:114`, `QLevel_Function_Library.h:246` (`QLevel_GetSeedFromOwnerLevel`) |
| **Noms des sous-niveaux différents entre client et serveur** (bug ouvert) : chaque stream reçoit un nom tiré d'un compteur propre au process, et client et serveur ne chargent pas dans le même ordre. Une référence client vers un acteur **posé** dans un sous-niveau QLevel arrive à None sur le serveur. Les systèmes qui adressent par position (OptimizedState, loot QModule) ne sont pas touchés, ni les acteurs créés par le serveur | [A] `QLevel_SubSystem_Instance.cpp:344-352` (`LoadLevelInstanceBySoftObjectPtr` sans nom) |
| Patron existant de placement procédural sur QLevel : `UQModuleLoot_World_SubSystem` se branche aux délégués de chargement de `UQLevel_SubSystem`, lit un catalogue d'ancres stocké dans l'asset, et tire avec une graine dérivée de la position monde de l'instance, identique sur serveur et clients | [A] `QModuleLoot_World_SubSystem.h`, `Documentation/QMODULE_LOOT_ARCHITECTURE.md` |

**Correction d'une hypothèse de la première version de ce document** : j'avais écrit que le serveur ne connaissait
pas les QLevel. C'est faux d'après le code ci-dessus : le serveur les charge, sans leur proxy visuel. La logique de
jeu d'un lieu peut donc vivre dans son sous-niveau et s'exécuter côté serveur.

---

## 3. Ce qui manque, d'après cette lecture

Chaque ligne est une conclusion de type [C]. **RzZz : si l'une d'elles est fausse, c'est ici que l'analyse casse.**

1. **Un parc de véhicules à voler.** Je n'ai pas trouvé de véhicules pilotables garés dans l'univers.
2. **Un verrou.** Je n'ai pas trouvé de contrôle de propriétaire à l'entrée du siège pilote : ni `VehicleBase`, ni
   `VehicleComponent`, ni `VehicleSlot` ne citent `VehiclePlayerOwner` ou `OwnerPlayerId`. `VehicleBase` porte
   pourtant `Driver Locked Interaction`, `LevelRequired`, `AllowedToAdmin` et `VehicleComponent` utilise
   `Lib_TagLock` : leur rôle n'a pas été lu.
3. **Une réputation de faction** persistante par joueur.
4. **Un point de vente** qui paie un véhicule sans en donner la possession.
5. **Un émetteur du délit `Theft`.** L'API existe (`ReportCrime`, §2.2), il ne manque que l'appel.

---

## 4. Proposition de design

### 4.1 Le parc : des places, pas des acteurs
- Des **places de parking** déclarées en données : position, type de lieu (ville, relais, station, orbite, épave),
  catégories de véhicules admises.
- Le **serveur** décide quelle place est occupée, et par quel modèle, par un tirage déterministe sur un cycle long
  partagé. C'est le patron du loot du monde (clé = position, réapparition par l'horloge serveur globale).
- Le véhicule réel n'est créé que lorsqu'un joueur approche, et il passe par `WorldVehiclesManager` pour le quota
  et la disparition à distance.
- Une place vidée par un vol reste vide jusqu'au cycle suivant.
- Dans l'espace : même mécanique, les places sont des points en orbite ou près d'épaves.

**Support : les QLevel** (décision de Benjamin). Les places de parking d'un QLevel (parking de ville, cour de relais,
hangar de station, épave) sont un **catalogue de places stocké dans l'asset** (`AdditionalData`), pas des acteurs
posés. Un subsystem de recel se branche aux délégués de chargement de `UQLevel_SubSystem`, comme le fait déjà
`UQModuleLoot_World_SubSystem` (§2.5) : quand le serveur charge le sous-niveau parce qu'un joueur approche, il lit le
catalogue, applique le tirage du cycle avec une graine tirée de la position monde de l'instance, et crée ou non le
véhicule. Aucune table globale à cuire : une place n'existe pour le serveur que lorsqu'elle sert.

Tranché : **le serveur crée le véhicule** (acteur répliqué dynamique). Un vrai `VehicleBase` posé dans le sous-niveau
est exclu : les noms des sous-niveaux QLevel diffèrent entre client et serveur (§2.5), donc un acteur posé n'a pas
de jumeau fiable entre les deux machines. Un acteur créé par le serveur n'a pas ce problème.

### 4.2 Le verrou : deux gestes
- État serveur répliqué sur le véhicule du parc : **verrouillé**, **ouvert**, **démarré**.
- Geste 1, ouvrir la porte. Geste 2, démarrer le moteur. Chacun est une interaction maintenue validée par le
  serveur (`QPlayerControllerInteractComponent`). Le mini-jeu `Hacking_Bridge` peut servir d'habillage plus tard.
- Chaque modèle de véhicule volable porte un `UQInteractionVolumeComponent` dimensionné sur sa porte et son poste de
  pilotage : le serveur dédié n'a pas de mesh et refuse une interaction dont la cible n'a pas ce volume (§2.2).
- **Sans module** : possible, mais long et bruyant (alarme audible, délit `Theft` signalé à la police).
- Les véhicules possédés par un joueur ne sont pas concernés en v1 (voir §6, question 6).

### 4.3 Le module dédié
Un module cyborg à 3 niveaux, sur le patron existant (stats lues par `QMOD_GetStat`). Nom à choisir.

| Niveau | Temps d'ouverture et de démarrage | Alarme | Classes accessibles | Repérage |
|---|---|---|---|---|
| Sans module | long | toujours | véhicules terrestres simples | aucun |
| 1 | réduit | toujours | terrestres | places occupées proches sur la carte |
| 2 | court | une chance sur deux | + vaisseaux légers | portée étendue |
| 3 | très court | coupée | + vaisseaux lourds | modèle et valeur affichés |

Les valeurs chiffrées sont à régler en jeu. `QMD_KitDeSabotage` (porte et tourelle) est de la même famille : soit on
l'étend aux véhicules, soit on crée un module à part (ma recommandation : à part, comme demandé par Benjamin).

### 4.4 Le trajet
- Véhicule volé = véhicule « chaud » : la police poursuit (le mode de poursuite véhicule ou vaisseau existe déjà dans
  `EPoliceSuspectMode`). En zone sans loi, pas de délit : c'est déjà le comportement de `Theft`.
- L'état du véhicule à l'arrivée (dégâts) baisse le prix.

### 4.5 Le réseau de garages Voss
**Un garage normalisé, posé partout.** Un seul kit (même logique, même interface, même signalétique), décliné en
trois gabarits :

| Gabarit | Où | Accepte |
|---|---|---|
| Garage de surface | villes, relais, bords de route, zones isolées | véhicules terrestres, plus un pad pour les vaisseaux légers |
| Hangar | près des ports et des grandes villes | tous les vaisseaux, et les terrestres |
| Station | une ou plusieurs stations dans l'espace | vaisseaux seulement |

- **Répartition** : dense sur Terre (objectif : toujours un garage à portée raisonnable d'un joueur terrestre),
  quelques-uns sur les autres planètes, au moins une station par grande zone de l'espace. Les nombres et les
  emplacements sont à poser avec Benjamin.
- **Un garage = un QLevel** (décision de Benjamin). Un `UQLevel_Asset` par gabarit, donc trois au total, chacun avec
  son sous-niveau (décor, plateforme de dépôt, receleur, places) et son proxy lointain.
- **Normalisé veut dire** : ajouter un garage revient à poser une instance `AQLevel_Actor_Instance` du gabarit voulu,
  sans rien recoder. La plateforme et le receleur vivent dans le sous-niveau et s'exécutent côté serveur (§2.5),
  mais un client ne doit jamais envoyer au serveur une référence vers un acteur posé dans ce sous-niveau (elle
  arrive à None, §2.5). Les demandes du client désignent le garage par sa position, ou passent par un acteur créé par
  le serveur.
- **Réglages du gabarit** dans `AdditionalData` de l'asset (catégories acceptées, taux de recel). La liste de
  commandes varie d'une instance à l'autre à partir de sa position ou de sa graine.
- **Deux pièges QLevel à traiter** : le serveur charge sans anticipation, à la position actuelle du pion (§2.5), donc
  un vaisseau rapide peut arriver avant que le sous-niveau du garage soit chargé (la vente doit attendre la
  plateforme, pas échouer), et un asset dont `LevelComplexity` vaut 0 échappe au budget de chargement (à mettre à
  jour à la création des gabarits).
- **Sur la carte** : chaque garage a son marqueur, visible de loin, donc avant que son sous-niveau soit chargé. Il
  faut une liste de tous les garages disponible sans chargement (question 10). Un tracker `TK_Garage` et une icône `Icon__Garage` existent déjà
  [B] : leur usage actuel est à lire avant de les réemployer.
- **Décors candidats au réemploi** [B, noms de fichiers seulement] : `BP_Garage_A`, `L_Garage_A1`, `L_Garage_A2`,
  `L_Abandoned_Garage_1`, `L_Voss_City` (plugin Qasset), `L_VOSS_PALETTE`. À regarder dans l'éditeur.
- **Différences entre garages** : chaque garage a sa propre liste de commandes et sa propre saturation. Vendre dix
  fois le même modèle au même endroit paie de moins en moins, ce qui fait tourner le joueur entre garages.

**Le dépôt.**
- Une plateforme de dépôt et un receleur. La détection du véhicule sur la plateforme réemploie le principe de
  `BP_TerminalVehicleSpawner.CheckVehicleOnPlatform` (à lire dans l'éditeur avant de s'y fier).
- **Prix payé** = `VehicleClass:Price` du registre, multiplié par un taux de recel (ordre de grandeur 15 à 25 %),
  par l'état du véhicule et par la demande.
- **Demande** : le garage paie moins un modèle qu'il a déjà beaucoup reçu, et affiche une liste de commandes du
  moment (bonus sur certains modèles), ce qui pousse à aller chercher loin.
- Paiement par `AddCoins`, sur le patron de `PayBounty`.

### 4.6 La réputation Voss
- Un compteur persistant par joueur, crédité à chaque vente. À créer (voir §3 point 3).
- Accès aux missions : `UCheckPlayerFlagCondition` ne teste que la présence d'un drapeau (§2.3). Chaque palier pose
  donc son propre drapeau au moment où le compteur le franchit, ou une nouvelle condition DQS lit la valeur.
- Paliers proposés : contrats de vol sur commande (un nouveau gabarit de contrat « livrer tel modèle »), rabais et
  rayon réservé (`QMD_AccreditationVoss` est déjà rédigé dans ce sens), puis fin de l'hostilité des Voss.

### 4.7 Garde-fous
1. **Seuls les véhicules du parc sont vendables.** Sinon le joueur revend son propre véhicule, un véhicule rappelé
   gratuitement, ou un véhicule admin. Le marqueur de provenance est posé par le serveur à la création.
2. **Validation serveur à la vente** : le mouvement est simulé par le client, donc le serveur contrôle provenance,
   distance parcourue et durée du trajet.
3. **Économie** : une carte de véhicule vaut jusqu'à 50 000 chez les marchands (mesure d'août). Le taux de recel et
   le cycle de réapparition doivent être réglés pour que le vol ne remplace pas tout le reste.
4. **Hostilité** : les Voss tirent aujourd'hui sur les joueurs. Sans zone neutre autour du garage, personne ne peut
   vendre. La zone neutre existe déjà sous forme de `WV_SafeArea` (§2.3) : chaque gabarit de garage en porte une.
5. **Trajet trop court** : avec des garages partout, voler un véhicule à 200 m d'un garage rapporterait autant qu'un
   long convoyage. Le prix tient donc compte de la distance entre la place d'origine et le garage (calculée par le
   serveur à partir de la place, pas du trajet déclaré par le client), et aucune place de parc n'est tirée trop près
   d'un garage.
6. **Coût à 500 joueurs** : un garage inoccupé ne doit rien coûter. Pas de tick, pas d'acteur répliqué en permanence,
   réveil à l'approche d'un joueur.

### 4.8 Multijoueur et solo (exigence de Benjamin : les deux doivent marcher parfaitement)
**Règle** : un seul chemin de code. Tout l'état du recel (place occupée, verrou, provenance, prix, réputation) est
décidé par l'autorité, que ce soit un serveur dédié ou la machine du joueur en solo. Aucune branche « si solo ».
Le client ne fait que demander (RPC serveur) et afficher (état répliqué, `OnRep`).

| Cas à tenir | Réponse du design |
|---|---|
| Deux joueurs crochètent le même véhicule | le verrou est un état serveur : le premier geste validé gagne, l'autre voit la progression et peut monter passager |
| Un joueur arrive pendant un crochetage, ou rejoint la partie | l'état du verrou et l'alarme sont répliqués, pas joués par un événement unique |
| Le voleur se déconnecte en route | le véhicule reste « chaud » et volable par un autre ; la disparition suit `WorldVehiclesManager` |
| Qui est payé à la vente | le pilote au moment du dépôt ; partage avec le groupe à décider (§7) |
| Deux véhicules sur la même plateforme | une vente à la fois par plateforme, file côté serveur |
| Culling à 300 m et mouvement simulé par le client | la vente se valide côté serveur (provenance, distance, durée), jamais sur la parole du client |
| Plusieurs instances de serveur | le cycle du parc et la réputation sont rangés par port serveur, comme la sauvegarde DQS (`SaveGames/Port<port>`, `SaveGames/Offline` en solo, `QuestManagerSubsystem.cpp:5050-5064`) et QStorage (`QStorage_Persistence.cpp:85`) |
| Solo | même logique, la machine locale est l'autorité ; le cycle du parc et la réputation se sauvegardent dans la partie solo |
| Pas de joueur local sur serveur dédié | aucun `GetFirstPlayerController`, aucun widget ni son côté serveur (c'est le défaut relevé sur `UTriggerZoneComponent`, §2.3) |

**Preuve exigée à chaque lot** : le même scénario joué deux fois, en solo, puis en serveur dédié avec deux clients
(un qui vole, un qui regarde et qui rejoint en cours). Tant que les deux ne sont pas passés, le lot n'est pas fini.

Le solo tourne en monde autonome (`NM_Standalone`) et sauvegarde dans `Saved/SaveGames/Offline` : réponse à la
question 11.

---

## 5. Lots proposés (base d'abord)

Chaque lot se termine par une mesure en jeu, **en solo et en serveur dédié à deux clients (§4.8)**, pas par
« c'est fait ». Compilation éditeur de notre côté, cibles jeu
et packaging par RzZz.

| Lot | Contenu | Écrit quoi |
|---|---|---|
| 0 | Lire dans l'éditeur : entrée du siège pilote, `Driver Locked Interaction`, `Lib_TagLock`, `CheckVehicleOnPlatform`. Tester si un vaisseau du trafic aérien se pilote. Répondre aux [C] | rien |
| 1 | Boucle minimale dans `L_Dev_Claude` : un véhicule de parc verrouillé, deux gestes, dépôt sur plateforme, paiement | C++ (plugin à choisir avec RzZz) + duplicatas |
| 2 | Module à 3 niveaux, alarme, délit `Theft` | QMD, item, C++ |
| 3 | Parc à l'échelle : marqueurs de place dans les QLevel, cycle serveur, espace | marqueurs en sous-niveau (accord requis par level), C++ |
| 3 bis | Réseau de garages : 3 QLevel gabarits, instances posées sur Terre d'abord, puis autres planètes et station | 3 `UQLevel_Asset` neufs, pose d'instances (accord requis par level) |
| 4 | Réputation Voss, contrat « livraison de véhicule », marqueurs de carte | C++, DQS |

---

## 6. Questions à RzZz

Réponds directement sous chaque question, ou corrige les tableaux du §2.

1. **Existe-t-il déjà des véhicules pilotables posés dans l'univers** (levels, QLevel, spawners) que je n'ai pas vus ?
   - **Réponse** : pas mesurable sans l'éditeur, c'est le lot 0 (requête du registre d'assets sur les levels, sans
     les charger).
2. **Qu'est-ce qui contrôle aujourd'hui l'entrée au siège pilote ?** Un joueur peut-il monter dans le véhicule
   possédé par un autre et le démarrer ? Rôle de `Driver Locked Interaction`, `LevelRequired`, `Lib_TagLock` ?
   - **Réponse partielle** : le C++ derrière le nom `TagLock` est `UTagLockSubsystem` (plugin EssentialMacros), un
     registre local par monde, non répliqué, de verrous nommés (contexte, tag, ensemble d'identifiants de ceux qui
     tiennent le verrou) [A] `TagLockSubsystem.h`, `TagLockObject.h`. Ce n'est pas un contrôle de propriétaire.
     `Lib_TagLock` est très probablement sa bibliothèque Blueprint. L'entrée au siège, `Driver Locked Interaction`
     et `LevelRequired` sont dans les graphes Blueprint : lot 0.
3. **QLevel côté serveur** : je lis dans le code que le serveur dédié charge les sous-niveaux sans leur proxy
   (§2.5). Est-ce bien le cas en production, autour de chaque joueur, et avec quel délai ? Peut-on y faire vivre un
   acteur de gameplay avec autorité (plateforme de dépôt), et y poser un marqueur qui fait créer un véhicule ?
   - **Réponse** : oui pour le code (§2.5). Le serveur charge autour de la position actuelle du pion de chaque
     joueur, sans anticipation, réévalue toutes les 0,25 s et n'a ni délai d'affichage ni déchargement différé. Un
     acteur posé dans le sous-niveau tourne avec l'autorité du serveur. Deux limites : un client ne peut pas désigner
     au serveur un acteur posé dans un sous-niveau QLevel (noms différents entre les deux machines, bug ouvert), et
     le véhicule doit être créé par le serveur, pas posé (§4.1). La preuve en conditions de production se fait sur
     le serveur Linux packagé.
4. **Existe-t-il un compteur de réputation ou de points de faction** quelque part (Blueprint, DataManager, intranet) ?
   - **Réponse** : rien en C++ ni en config (§2.3). Côté Blueprint, pas lisible sans l'éditeur. Les drapeaux DQS
     par joueur (`SetPlayerFlag`, valeur texte, sauvegardés) peuvent porter le compteur.
5. **Où mettre le C++** : dans QModule, QVehicleIntegration, ou un nouveau plugin ?
   - **Réponse** : `QVehicleIntegration`. C'est déjà le plugin d'intégration des véhicules (passagers, livraison de
     flotte, terrain WorldScape d'après sa description), il dépend de QModule, ClientAuthority, QLevel, QNetState et
     FlyVehicleMovement, et aucun plugin ne dépend de lui. Il faut lui ajouter QPolice (délit `Theft`) et
     DynamicQuestSystem (drapeaux, contrats). Ni l'un ni l'autre ne dépend de QVehicleIntegration, donc pas de cycle.
     Pas dans QModule : l'intégration entre plugins vit dans un plugin d'intégration.
6. **Véhicules des autres joueurs** : intouchables en v1 (ma recommandation), ou volables ?
   - **À trancher par RzZz.** Fait utile : la provenance posée par le serveur à la création (§4.7) exclut déjà les
     véhicules possédés, identifiés par `VehiclePlayerOwner.OwnerPlayerId`.
7. **Quel système de missions** pour les contrats Voss : le tableau procédural DQS (ma recommandation), ou l'ancien
   `Content/Systems/Contract` et `Mission` ?
   - **Réponse** : le tableau procédural DQS. C'est le système de contrats actuel, étendu le 18/11 (démolition,
     extermination, primes, récupération, commit cf010ef03), avec les drapeaux par joueur pour l'accès. Ne pas
     construire sur l'ancien système Blueprint.
8. **Un vaisseau du trafic aérien** doit-il pouvoir être volé un jour (abordage), ou seulement les véhicules garés ?
   - **À trancher par RzZz.** Fait utile : ces vaisseaux sont des BP pilotables complets mais allégés en dur par
     QAI (§2.4), donc un abordage demande un chemin de remise en état pilotable. Hors v1 dans tous les cas.
9. **Hostilité Voss** : zone neutre autour du garage, ou changement de la matrice selon la réputation du joueur ?
   - **Réponse** : zone neutre en v1. Le mécanisme existe déjà (`WV_SafeArea`, refus de combat dans
     `QCombatPolicy`, §2.3). La matrice oppose des factions entre elles, pas un joueur précis : la faire dépendre de
     la réputation d'un joueur demande un filtre par joueur qui n'existe pas. À reprendre plus tard si RzZz le
     veut.
10. **Poser le même QLevel à beaucoup d'endroits** (dizaines de garages sur Terre, quelques-uns ailleurs, une
    station) : dans quel level d'assemblage placer les instances, et comment obtenir la liste de tous les garages
    pour la carte sans charger leurs sous-niveaux (enregistrement des instances au démarrage, ou table à part) ?
    Et à quoi servent aujourd'hui `TK_Garage`, `BP_Garage_A` et `L_Garage_A1/A2` ?
    - **Réponse partielle** : chaque `AQLevel_Actor_Instance` posé s'enregistre auprès du subsystem dès son
      BeginPlay (`QLevel_Actor_Instance.cpp:101`, `QLevel_RegisterLevelActor`), avant que son sous-niveau se
      charge. La liste des garages peut donc se construire sans charger les garages, à condition que les acteurs
      d'instance soient dans un level chargé sur toutes les machines. `QLevel_GetAllInstancesInLevel` passe par
      `GetAllActorsOfClass` : à garder pour l'éditeur, pas pour le jeu. Le level d'assemblage et l'usage actuel de
      `TK_Garage`, `BP_Garage_A` et `L_Garage_A1/A2` : lot 0 dans l'éditeur.

11. **Solo** : le jeu solo tourne-t-il en monde autonome ou en serveur d'écoute, et où sauvegarde-t-il ? Y a-t-il
    des pièges connus de code qui marche en dédié mais pas en solo (ou l'inverse) ? Et comment lancer chez nous un
    test serveur dédié à deux clients sans compiler les cibles jeu ?
    - **Réponse** : monde autonome (`NM_Standalone`). Le client ouvre `L_Persistent_Universe` sans `?listen` (log
      client du 20/11), et DQS range alors sa sauvegarde dans `Offline` (`QuestManagerSubsystem.cpp:5052-5055`).
      Fichiers : `Saved/SaveGames/Offline/<Map>_Offline.sav` et son miroir `BACKUP_Offline`, la progression des
      quêtes dans `Offline/<SteamName>_Progress.bin`, les commits de boutique et d'inventaire dans
      `Saved/QInventory/TempDB`, fusionnés en fin de session.
    - **Pièges connus** : en solo il y a un joueur local et des meshes, en dédié ni l'un ni l'autre. En dédié :
      aucun `GetFirstPlayerController`, pas de collision des StaticMesh, une vue et une position des joueurs fausses
      côté serveur (leur mouvement est décidé par le client), une portée d'interaction mesurée sur les
      `UQInteractionVolumeComponent`. Les noms de sous-niveaux QLevel et les clés OptimizedState par position ne
      divergent qu'entre deux machines, donc jamais en solo.
    - **Test à deux clients** : la PIE avec serveur dédié et deux clients suffit pour itérer la logique, mais elle
      garde les meshes de l'éditeur et tourne dans un seul process. La preuve d'un lot se fait sur un serveur Linux
      packagé (WSL local), packaging par RzZz.

## 7. Décisions attendues de Benjamin

1. Accès au garage dès le début (zone neutre), ou accès à gagner.
2. Sans module : crochetage lent et bruyant, ou impossible.
3. Le level qui sert de modèle au garage normalisé, et le premier endroit où le poser.
4. Le nom du module.
5. L'ordre de grandeur du réseau : combien de garages sur Terre, sur les autres planètes, et combien de stations.
6. En groupe : la paie et la réputation vont au pilote seul, ou sont partagées avec les passagers.
