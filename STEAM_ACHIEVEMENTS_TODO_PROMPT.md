# TODO Prompt : terminer le raccordement des Steam Achievements de QANGA

> À coller tel quel dans une future session Codex ouverte sur `G:\QANGA`.
> État relevé le 2026-08-20 après l'audit, la réparation du socle et deux lots raccordés.
> Le dépôt est régulièrement modifié entre deux sessions : revalide les faits ci-dessous avant
> toute édition et conserve les changements sans rapport déjà présents dans le worktree.

## Mission

Termine l'inventaire puis le raccordement des Steam Achievements de QANGA en partant des
producteurs gameplay réels. Commence par les achievements qui disposent déjà d'une source
autoritative exploitable. N'ajoute aucun compteur parallèle, aucun déblocage arbitraire et aucun
fallback destiné à masquer un producteur manquant.

Ne crée et ne publie aucun nouvel ID Steam sans une décision explicite de Rz. Un achievement Steam
réel ne doit jamais être débloqué sur son compte pour servir de test.

## État mesuré au 2026-08-20

| Domaine | IDs publiés sur Steam | Logiques référencées dans le jeu | Non connectés |
|---|---:|---:|---:|
| Character | 33 | 27 | 6 |
| World | 41 | 15 | 26 |
| Kill | 25 | 4 | 21 |
| **Total** | **99** | **46** | **53** |

Configuration Steamworks constatée :

- les 99 achievements sont `Set By Client` ;
- aucun Steam Stat et aucun `Progress Stat` ne sont déclaré ;
- huit achievements sont cachés ;
- les noms anglais existent, mais les descriptions sont vides ;
- QModule v2 et QBuilder n'ont actuellement aucun ID Steam.

La limite Steam initiale est de 100 achievements. Il ne reste donc qu'un nouveau slot garanti tant
qu'une capacité supérieure n'a pas été confirmée dans Steamworks.

## Réparations déjà présentes dans le worktree

Ne les réimplémente pas aveuglément. Relis le diff et vérifie qu'elles sont toujours présentes :

- `Content/Systems/Achievement/PC_AchievementComponent.uasset`
  - politique de cartes lue depuis le CDO de `AchievementLogic`, et non depuis les 36 classes filles
    dont `AllowedMaps` est vide ;
  - chemins distincts pour client, dedicated server et listen server ;
  - sur listen server, le PlayerController local exécute le chemin autoritatif puis l'initialisation
    Steam client ;
  - les sept logiques autoritatives existantes sont spawnées côté serveur et transmettent leur clé
    au client propriétaire ;
  - les logiques client ne sont spawnées que pour les achievements encore verrouillés ;
  - queue d'unlock par batch, remise en attente des échecs, retry borné à 10 secondes et
    regroupement de `StoreStats` sur une seconde ;
  - aucun chemin `GameServerStats` n'est encore consommé.
- `Plugins/SteamCore/Source/SteamCore/Private/SteamUserStats/SteamUserStats.cpp`
- `Plugins/SteamCore/Source/SteamCore/Public/SteamUserStats/SteamUserStats.h`
  - `RequestCurrentStats` est compatible avec Steamworks 1.61 et n'entretient plus une boucle
    `Delay(0)` infinie ;
  - l'état prêt/en attente est suivi explicitement et le callback ne valide que le Steam ID local.
- `Plugins/RzDirectMCP/Source/RzDirectMCP/Private/BlueprintGraphLibrary.cpp`
- `Plugins/RzDirectMCP/Source/RzDirectMCP/Public/BlueprintGraphLibrary.h`
- `Plugins/RzDirectMCP/Source/RzDirectMCP/Private/ExtendedEditorLibrary.cpp`
  - les actions Array utilisent `UK2Node_CallArrayFunction` au lieu d'un
    `K2Node_CallFunction` wildcard impossible à typer ;
  - `add_assign_delegate` enregistre désormais les alias du bind et de l'événement, ce qui permet
    de les connecter dans le même batch d'opérations.
- `Plugins/RzDirectMCP/Source/RzDirectMCP/Private/ActorEditorLibrary.cpp`
  - `get_level_actors` peut inspecter un asset de map sans l'ouvrir ni remplacer le niveau courant ;
  - la réponse distingue explicitement une inspection `editor_world` d'une inspection `map_asset`.
- `Content/Systems/Achievement/Logics/AchievementLogic_WorldStatisticThreshold.uasset`
  - logique générique événementielle sur un `WorldStatisticDataAsset` persistant ;
  - initialisation immédiate si les stats sont prêtes, sinon abonnement à leur signal de disponibilité ;
  - abonnement à `OnStatReady` et `OnStatUpdated`, sans Tick ni polling ;
  - le serveur transmet uniquement le résultat au client propriétaire via le contrat existant.
- `Content/Systems/Achievement/Logics/KILL_INFECTED_1.uasset`
- `Content/Systems/Achievement/Logics/KILL_INFECTED_10.uasset`
- `Content/Systems/Achievement/Logics/KILL_INFECTED_100.uasset`
- `Content/Systems/Achievement/Logics/KILL_INFECTED_500.uasset`
  - quatre enfants minces configurés sur `WSDA_KilledInfecteds` avec les seuils `1/10/100/500` ;
  - les quatre clés exactes sont présentes dans `DA_AllRef.ClientAchievements` ;
  - leur `EventGraph` est vide : ils héritent directement du `BeginPlay` de la logique générique.
- `Content/Systems/Achievement/Logics/DISCOVER_TOWER_5.uasset`
- `Content/Systems/Achievement/Logics/DISCOVER_TOWER_10.uasset`
- `Content/Systems/Achievement/Logics/DISCOVER_TOWER_20.uasset`
- `Content/Systems/Achievement/Logics/DISCOVER_TOWER_30.uasset`
- `Content/Systems/Achievement/Logics/DISCOVER_TOWER_40.uasset`
- `Content/Systems/Achievement/Logics/DISCOVER_TOWER_50.uasset`
  - six enfants minces configurés sur `WSDA_TowerExplored` avec les seuils `5/10/20/30/40/50` ;
  - les six clés exactes sont présentes dans `DA_AllRef.ClientAchievements` ;
  - leur `EventGraph` est vide : ils héritent directement du `BeginPlay` de la logique générique.
- `Content/Stats/CharacterStatistics/WolrdStatistics/ExploreCounters/WSDA_TowerExplored.uasset`
- `Content/Stats/CharacterStatistics/WolrdStatistics/ExploreCounters/WSDA_WarpStationsExplored.uasset`
  - leurs affichages de comptes entiers utilisent désormais `0/0` chiffre fractionnaire au lieu de
    `2/2` ; aucune valeur persistée ni logique d'incrément n'a été changée ;
  - attention : `/Content/Stats/` est globalement ignoré par `.gitignore`. Ces deux corrections sont
    présentes localement mais ne seront pas incluses dans un commit normal sans décision explicite
    sur la politique de versioning de ce dossier.
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QStatsRegistrationTests.cpp`
  - `QATS.Stats.Achievements.StatisticThresholdRegistration` verrouille le registre, l'héritage,
    les dix clés, leurs seuils, leurs compteurs persistants et `ReplicateToClient`.

La dernière vérification connue donnait `PC_AchievementComponent`, la logique générique et les dix
enfants Infectés/Tours compilés et sauvegardés avec zéro erreur et zéro warning. Le registre
contenait 46 clés uniques. Le build C++ du module QATS réussissait également. Cela prouve le graphe,
le registre et l'outillage, et le test ciblé des dix seuils était vert. Cela ne prouve pas la
livraison Steam d'un build cooké. Une validation packagée exigera un nouveau Development cook/build
exécuté par Rz.

## Lots maintenant raccordés

### A. 10 raccordés à des producteurs persistants valides

Le compteur serveur `WSDA_KilledInfecteds` (`AutoId=1922666113`) est sauvegardé dans
`PlayerStatistics` :

```text
KILL_INFECTED_1
KILL_INFECTED_10
KILL_INFECTED_100
KILL_INFECTED_500
```

La chaîne autoritative est
`GameplayGlobalEvents.OnPawnDeathServerData -> KillCounterBase -> SS_CharacterStatistics.Add`.
Les cinq principaux CDO Infected inspectés portent exactement le tag `Infected`.

Le compteur serveur `WSDA_TowerExplored` (`AutoId=823999745`) est également sauvegardé dans
`PlayerStatistics` :

```text
DISCOVER_TOWER_5
DISCOVER_TOWER_10
DISCOVER_TOWER_20
DISCOVER_TOWER_30
DISCOVER_TOWER_40
DISCOVER_TOWER_50
```

La chaîne autoritative est
`PlayerZoneComponent.OnDiscoverNewZone -> ExploreCounterBase -> SS_CharacterStatistics.Add`.
`PlayerZoneComponent` déduplique les zones avec `ActiveZonesIds` et persiste ce set dans
`ZonesData`; revisiter une tour ne recrédite donc pas le compteur. Le producteur
`QLevel_Relay_Actor` n'est référencé en production que par les assets Terre, et l'ancien authoring
`LSA_RelayTower` observé porte également le tag `RelayTower` sur les tours terrestres.

Ces dix éléments ne font plus partie des éléments manquants. Ne pas recréer leur logique et ne pas
déplacer leurs mutations dans le composant Steam. Revalider
`QATS.Stats.Achievements.StatisticThresholdRegistration` après toute modification des stats ou du
registre.

## Classement exact des 53 achievements encore non connectés

### B. 39 nécessitant un bridge typé, un compteur ou un authoring explicite

Les six Character précédemment classés à tort comme directement raccordables sont :

```text
COMPLETE_CONTRACT_5
COMPLETE_CONTRACT_50
COMPLETE_CONTRACT_100
COMPLETE_CONTRACT_GROUP
SELL_500_ITEM
SELL_1000_ITEM
```

- `COMPLETE_CONTRACT_5/50/100` n'ont aucun total historique persistant ;
- `COMPLETE_CONTRACT_GROUP` doit être produit dans `ContractBase.FinishContract` en utilisant
  `MatchActor.IdsInTheMatch`, avant la destruction du match ;
- `SELL_500_ITEM/SELL_1000_ITEM` n'ont aucun total persistant. Le producteur doit compter les unités
  réelles via `ServerOnly_OnItemSell + ItemInstance.GetStack()`, pas les transactions ni la valeur
  monétaire temporaire `SellingIteratorValue`.

Les six POI ont un chemin technique utilisable via `TriggerAchievementOnArea`, mais aucun authoring
de production inspecté ne porte ces clés. Il faut placer et configurer les zones réelles avant de
créer leurs classes de logique :

```text
DISCOVER_ABANDONED_CAMP
DISCOVER_CAVE
DISCOVER_BUNKER
DISCOVER_ARMORY
DISCOVER_POLICE
DISCOVER_AERO_TRAM
```

Les 27 autres clés exigeant un producteur ou compteur explicite sont :

```text
USE_SCANNER_EARTH
USE_SCANNER_MOON
USE_SCANNER_MARS
USE_SCANNER_VENUS
USE_SCANNER_SPACE_STATION
DISCOVER_SANGLINE_TOWER_1
DISCOVER_PIRATE_TOWER_1
DISCOVER_SANGLINE_TOWER_5
DISCOVER_PIRATE_TOWER_5
DISCOVER_MIDDLE_COMPLEX
DISCOVER_TOP_COMPLEX
DISCOVER_SPACE_STATION
KILL_BIG_INFECTED_1
KILL_BIG_INFECTED_10
KILL_BIG_INFECTED_50
KILL_DRONE_1
KILL_DRONE_10
KILL_DRONE_100
KILL_HEAVY_DRONE_1
KILL_HEAVY_DRONE_10
KILL_HEAVY_DRONE_100
KILL_POLICE_GUARD_1
KILL_PLAYER_1
KILL_CITZEN_1
DESTROY_DESTRUCTIBLE_10
DESTROY_DESTRUCTIBLE_100
DESTROY_DESTRUCTIBLE_1000
```

Les bridges doivent traduire un événement gameplay existant vers une mutation unique du compteur
persistant. Pas de polling global, de déduction approximative par classe, de line trace ou de
compteur dupliqué dans le composant d'achievements.

### C. 3 nécessitant encore une preuve de scope ou d'événement gameplay

- `DISCOVER_WARP_STATION_EARTH_1` et `DISCOVER_WARP_STATION_EARTH_ALL` : le compteur Warp global
  existe, mais le scope Terre et le total exact nécessaire à `ALL` ne sont pas définis ;
- `USE_AERO_TRAM` : ne pas l'inférer depuis l'entrée dans une zone. Le véritable événement
  d'embarquement/utilisation QTrain reste à identifier.

### D. 6 bloqués par un producteur ou une persistance cassés

- `KILL_SANGLINE_1`, `KILL_SANGLINE_10`, `KILL_SANGLINE_100`, `KILL_SANGLINE_500` et
  `KILL_SANGLINE_1000` : `WSDA_KilledSanglines.AutoId=-2147483647`, donc sa clé de sauvegarde est
  invalide. Plusieurs variantes Sangline actives semblent également dépourvues du tag ;
- `KILL_AUTONOMOUS_1` : `WSDA_KilledAutonomous` est valide, mais le switch attend `Autonomus` et ce
  tag n'est porté que par `OLD_AI_AutonomousBase`, désormais déprécié.

Réparer les producteurs puis exiger le passage des tests concernés dans `QATS.Stats` avant de
connecter ces clés. Après un chargement propre, la suite compte 8 tests : 6 réussissent et 2
échouent sur des défauts de contenu réels, décrits ci-dessous ; ne pas affaiblir ces tests. La
session Live Coding de l'audit affichait temporairement 9 tests (`7/9` verts), car l'ancien nom du
test de seuil restait enregistré en mémoire jusqu'au prochain redémarrage.

- `QATS.Stats.WSDA.UniqueAutoId` : `WSDA_KilledSanglines.AutoId=-2147483647` ;
- `QATS.Stats.KillTags.CoveredByAnActor` : `Cyborg` et `FlyerSangline` sans acteur porteur,
  `Autonomus` uniquement sur un Blueprint déprécié.

### E. 5 définitions à revalider avant toute implémentation

- `REACH_DEEPEST_OCEAN` : atteindre le point le plus profond du fond marin ;
- `REACH_UNDERGROUND_5KM` : atteindre 5 km sous terre ;
- `REACH_UNDERGROUND_50KM` : atteindre 50 km sous terre ;
- `REACH_UNDERGROUND_100KM` : atteindre 100 km sous terre ;
- `REACH_UNDERGROUND_EARTHCORE` : atteindre le centre de la Terre.

Leur faisabilité et leur sens dans le monde actuel ne sont pas établis. Ne crée pas une mesure
artificielle pour rendre ces définitions vraies. Confirme d'abord l'espace jouable, les unités et
le signal autoritatif attendu, puis propose de les conserver, les reformuler ou les supprimer.

## Ordre de travail obligatoire

1. Rechercher les assets avec `search_project_index` et les inspecter avec RzDirectMCP. Les
   `.uasset` sont binaires : ne jamais les grep.
2. Refaire la matrice depuis les 99 clés Steam et `DA_AllRef`. Vérifier les clés exactes, les
   doublons, le statut caché et l'existence de chaque classe de logique.
3. Revalider les dix éléments A avec
   `QATS.Stats.Achievements.StatisticThresholdRegistration`. Ils sont déjà raccordés via une logique
   générique de seuil sur `SS_CharacterStatistics` ; ne dupliquer ni les événements ni les compteurs.
4. Pour chaque lot suivant, tracer le chemin complet : événement gameplay, propriétaire
   autoritatif, compteur/persistance, condition de seuil, logique d'achievement, RPC éventuel et
   appel Steam client. Chaque mutation doit avoir un seul propriétaire.
5. Compiler et sauvegarder chaque Blueprint modifié avant de passer au lot suivant. Vérifier la
   valeur effective en relisant l'asset depuis le moteur.
6. Pour les 39 éléments B, produire d'abord une proposition de bridges ou d'authoring avec
   propriétaires précis.
   Implémenter uniquement les bridges dont le signal source est prouvé.
7. Résoudre séparément les preuves C, les producteurs cassés D et les définitions E. Ne pas coder
   une approximation pour faire passer une définition dont le signal réel n'est pas établi.
8. Préparer enfin les nouveaux domaines QModule et QBuilder, sans publication Steam.

## Contrat réseau et Steam à préserver

- Steam reste l'unique propriétaire du déblocage final côté client.
- Une condition autoritative détectée sur le serveur transmet seulement la clé au client
  propriétaire. Elle ne tente pas `GameServerStats.SetUserAchievement`.
- Les logiques purement locales ne tournent que pour le PlayerController local.
- Le dedicated server ne dépend jamais de `SteamUserStats()` client pour spawner ses producteurs.
- Le listen server doit exécuter à la fois le chemin autoritatif et le chemin Steam du client local.
- Un échec `SetAchievement` ou `StoreStats` reste observable et réessayable ; il n'est jamais jeté
  silencieusement.
- Aucun log fréquent ne dépasse un message par seconde et par source.

## Matrice à rendre

Pour les 99 IDs, fournis au minimum ces colonnes :

| Colonne | Contenu attendu |
|---|---|
| Steam API Name | clé exacte publiée |
| Domaine | Character, World, Kill, Module ou Construction |
| Définition Steam | nom, description, caché, Progress Stat |
| Logique locale | classe et présence dans `DA_AllRef` |
| Producteur | événement, compteur ou système gameplay exact |
| Autorité | client, serveur ou listen server dual-path |
| Persistance | source de sauvegarde et unité du compteur |
| Statut | connecté, producteur prêt, bridge requis ou définition à revoir |
| Vérification | preuve statique, PIE sûr ou test packagé restant |
| Action | modification minimale encore nécessaire |

## Nouveaux domaines à préparer

### QModule v2

Le candidat recommandé est la première installation **manuelle et payée** d'un module Cyborg
non-base sur le mur du joueur. Ne fige pas son API Name avant validation produit.

- entrée joueur : `UQModule_RackComponent::SV_InstallModule`, RPC serveur fiable ;
- succès immédiat : retour vrai de `TryInstall(..., bAllowBaseModule=false)` dans
  `SV_InstallModule_Implementation` ;
- succès différé : même retour vrai dans `QMOD_FinishPendingInstall` ;
- mutation réelle : validation serveur, `QModuleInventory::ConsumeOne`, ajout dans `Sockets`, puis
  `MarkRackDirty` ;
- persistance : le rack du `PlayerState` encode `Sockets` dans le DataObject
  `QMODWall♥<PlayerId>` ; il ne conserve aucun historique d'installations.

Le futur bridge doit être émis uniquement depuis les deux succès du funnel `SV_InstallModule`, après
confirmation qu'un item a réellement été consommé et que `Definition->bBaseModule` est faux. Ne pas
le placer dans `TryInstall`, `OnRackChanged` ou `OnRep_Sockets` : ces chemins voient aussi le
bootstrap, les restaurations, les quêtes gratuites et les commandes de test. Les racks Weapon et
Vehicle ont d'autres propriétaires et ne disposent pas encore d'un signal commun ; ne pas les
agréger artificiellement au premier achievement.

### QBuilder

Le candidat suivant, si Steam accorde plus de 100 slots, est la première construction payante
acceptée par le serveur. Ne fige pas son API Name avant validation produit.

- entrée réseau : `UQBuilder_Client::QBuilder_ToServer_CompressData`, RPC serveur fiable ;
- autorité : `QBuilder_ServerData_ISM_AddInstances` et
  `QBuilder_ServerData_Actor_AddInstances` ;
- coût : `QBuilder_Resource_Price_Compute` vérifie puis débite les ressources du builder ;
- succès logique : `QBuilder_ISM_Data_AddISMInstance` ou `QBuilder_Actor_Data_AddActorInstance`
  retourne un index différent de `-1` ;
- persistance : `QBuilder_SaveGame_CreateData` sauvegarde directement les tableaux ISM/Actor et la
  reconstruction ne repasse pas par les fonctions Add.

QBuilder n'expose encore aucun delegate autoritatif de « construction payante acceptée ». Ajouter un
événement natif commun après `ReturnIndex != -1`, avec Builder ID, Data ID, type ISM/Actor,
propriétaire et preuve qu'un coût strictement positif a été débité. Exclure `FreeMode`, la fondation
gratuite créée par `QBuilder_Builder_ClientCreateBuilderActorWithBuild`, les coûts nuls, les reloads
et les resynchronisations. L'attribution reste à décider quand un collaborateur construit sur le
builder d'un autre joueur : les fonctions Add ne reçoivent pas le client appelant et ne connaissent
actuellement que `QBuilder_PlayerOwner`.

Défaut QBuilder distinct à ne pas masquer dans le bridge : le coût est débité avant l'allocation de
l'index et n'est pas remboursé si celle-ci retourne `-1`. Le signal achievement doit rester après le
succès, mais le rollback de ressources doit être corrigé séparément avant de considérer ce chemin
transactionnel.

Pour chaque nouvelle proposition, exige : un événement autoritatif existant, une condition non
ambiguë, une persistance définie, un comportement multijoueur déterministe et une valeur réelle
pour le joueur. Une simple action technique répétitive ne suffit pas à justifier un achievement.

## Recette

- aucune clé Steam inventée ou publiée ;
- aucun déblocage réel effectué sur le compte de Rz ;
- aucune logique forcée depuis la console ou un Blueprint de test ;
- aucun compteur gameplay dupliqué dans le système Steam ;
- Blueprint compilé et sauvegardé avec zéro erreur et zéro warning ;
- build `QangaEditor Win64 Development` réussi si du C++ est modifié ;
- pas de cook, stage ou package : Rz les exécute ;
- pas de commit ni de staging sans demande explicite ;
- seules les modifications appartenant à cette tâche sont rapportées ;
- le handoff final distingue clairement ce qui est prouvé, ce qui reste à tester dans un nouveau
  build Steam et ce qui attend une décision produit.

Commence par l'état réel du dépôt. Si le snapshot ci-dessus diverge du code ou des assets actuels,
la preuve actuelle fait autorité et la matrice doit être corrigée avant toute implémentation.
