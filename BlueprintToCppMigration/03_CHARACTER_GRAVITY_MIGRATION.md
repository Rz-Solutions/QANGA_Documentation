# Troisième migration Blueprint vers C++ — personnage et gravité

- **État :** J0/J1/J2 terminés ; cleanup meshless J3 et son gate Linux Shipping terminés ; J4/J5/J6 et la requête monde typée compilent en build Editor froid, `15/15` QATS passent ; cleanup joueur, closure Blueprint et stress PIE QPlatform à `360 FPS` revalidés ; runtime AILean, réseau et nouveau build packagé restent ouverts
- **Propriétaire du provider :** module runtime existant `GravityScape` (aucun nouveau plugin)
- **Consommateurs principaux :** `ALS_Base_CharacterBP`, `ALS_Base_CharacterBP_AILean`
- **Provider Blueprint :** `/Game/Systems/GravityArea/GravityAreaComponent`
- **Source Blueprint :** `/Game/Systems/GravityArea/LevelGravityArea`
- **Carte de validation :** `/Game/Maps/LevelDev/L_Dev_Rz`
- **Début :** 2026-08-26

---

## 0. Décision

La troisième migration ne traduit pas les `4 454` nœuds ALS en bloc. Elle commence par le contrat de gravité partagé qui alimente ALS, AILean, les véhicules, le jetpack et plusieurs systèmes IA.

L'ordre retenu est :

1. rendre le provider `GravityAreaComponent` et la donnée `LevelGravityArea` strictement natifs ;
2. basculer l'application direction/échelle du personnage ;
3. basculer la rotation alignée surface avec un ordre de tick explicite par rapport à QAI ;
4. migrer ensuite `SetEssentialValues` et les autres chemins mouvement ALS mesurés.

Le module `GravityScape` est déjà le domaine natif de gravité et il est déjà une dépendance de QAI et QPolice. Aucun plugin `QGravity` concurrent ne sera créé. Son subsystem historique est toutefois désactivé dans `DefaultGame.ini` et ne peut pas être activé tel quel : ses règles de sphère, de priorité, de net mode et son chemin async ne sont pas équivalents au système QANGA actif. Il sera corrigé et étendu en place, sans devenir un fallback silencieux.

Les assets `GravityAreaComponent` et `LevelGravityArea` restent d'abord des wrappers de compatibilité aux mêmes chemins. Chaque graphe et variable remplacé sera supprimé. Un wrapper ne restera que tant qu'un referencer sérialisé a encore besoin de sa classe ; il ne contiendra aucun second producteur de gravité.

## 1. Preuve du coût et contrat actuel

### 1.1 Personnage

L'inspection live, sans rescan de l'index projet, établit :

- `ALS_Base_CharacterBP` : `4 454` nœuds, `TickGraph` à `99` nœuds et `UpdateNinjaGravityDirection` à `314` nœuds ;
- `ALS_Base_CharacterBP_AILean` : `4 251` nœuds, `TickGraph` à `95` nœuds et `UpdateNinjaGravityDirection` à `307` nœuds ;
- les deux bases héritent directement de `NinjaCharacter` et ne sont pas héritées l'une de l'autre ;
- le Tick ALS lit `GetGravityDirectionAttracting`, écrit `Gravity_Direction`, puis appelle `UpdateNinjaGravityDirection` ;
- le dispatcher `GravityUpdated` rappelle également `UpdateNinjaGravityDirection` ;
- `UNinjaCharacterMovementComponent` possède déjà les API natives d'application de gravité et de rotation ;
- QAI réapplique aujourd'hui une rotation après ALS. La bascule rotation doit donc posséder une dépendance de tick vérifiable, pas une nouvelle course d'écriture.

### 1.2 Provider partagé

`GravityAreaComponent` contient `167` nœuds, `16` variables, `15` fonctions et le dispatcher `GravityUpdated`.

Son exécution actuelle :

- groupe de tick `TG_PostUpdateWork` ;
- `UpdateTime=0.1 s` par défaut ;
- si `UpdateTime <= 0`, une boucle latente relance la requête toutes les `0.01 s` ;
- première requête immédiate au BeginPlay ;
- attente aléatoire `0..0.2 s`, puis activation du tick après résolution du manager global ;
- `GlobalGravityAreaManagerRef` n'est ensuite jamais consommé par le calcul ;
- chaque refresh lance `Sphere Overlap Actors` de rayon `1 cm` sur `ObjectTypeQuery8`, donc le canal objet `NinjaVolume` ;
- les zones qui se recouvrent sont départagées par `Priority` strictement supérieur ;
- la zone gagnante alimente tag, position, échelle et référence cachés ;
- les overrides scale/location/tag sont appliqués après la résolution ;
- le dispatcher est émis sur force refresh, changement de zone ou `AlwaysCallUpdateOnCheck`.

Le composant possède `189` referencers directs. `LevelGravityArea` en possède `206`, et `Lib_GravityArea` `54`. Une suppression brutale des chemins d'assets casserait donc des SCS et des graphes sérialisés ; le reparent et le cleanup doivent conserver le contrat pendant la transition.

### 1.3 Sémantique exacte à conserver

- direction radiale non signée : `Normalize(AreaLocation - QueryLocation)` ;
- direction fixe non signée : forward de `FixedGravityDirection_Arrow` ;
- direction attirante : direction non signée multipliée par `-1` uniquement si l'échelle cachée est négative ;
- absence de zone : direction zéro, `Success=false`, tag `None`, échelle `0` ;
- `SetForceGravityScale`, `ForceGravityLocation` et `ForceTagArea` rafraîchissent immédiatement ;
- `SetForcedGravity` force les trois valeurs et respecte son paramètre `ForceRefresh`, vrai par défaut ;
- `DisabledForcedGravity` libère les trois valeurs et force un refresh ;
- `GetIsForcedGravity` est vrai si au moins un des trois overrides est actif ;
- `GetHasAtmosphere` renvoie `false` sans zone.

### 1.4 Pourquoi le subsystem GravityScape ne sera pas simplement activé

Le code actuel est configuré `Enabled=False` et `EnabledOnClient=False`. L'audit de ses sources révèle aussi :

- une branche de net mode qui teste deux fois `NM_DedicatedServer` ;
- un unregister qui utilise le nombre d'éléments supprimés comme index ;
- un calcul sphérique qui compare une distance au carré à des rayons non carrés ;
- un chemin custom-location dont les parenthèses ne représentent pas le vecteur entre les deux positions ;
- un override sélectionné par priorité même lorsque sa zone ne produit aucune gravité à la position ;
- un chemin async qui capture des tableaux locaux par référence au-delà de leur durée de vie.

Ces défauts seront corrigés avec tests. Le subsystem restera désactivé pour ses anciens producteurs tant que les zones QANGA n'y sont pas enregistrées et validées. Aucun consommateur ne basculera sur une sortie non équivalente.

## 2. Frontière native

### 2.1 Types et source de zone

Le module `GravityScape` reçoit :

- des enums natifs direction et forme remplaçant les deux User Defined Enums après migration de leurs `11` et `2` referencers ;
- une base native `AQLevelGravityArea` portant priorité, direction, tag, échelle, forme, dimensions et atmosphère ;
- une résolution typée du composant directionnel existant, mise en cache et invalidée explicitement ;
- l'enregistrement/désenregistrement et une génération de zones dans le subsystem monde.

`LevelGravityArea` est reparenté sur cette base. Ses composants SCS et son comportement de construction restent pendant la première bascule, mais les variables devenues héritées sont supprimées immédiatement.

### 2.2 Composant consommateur de zone

`UQGravityAreaComponent` reprend les noms et signatures publics réellement consommés. Il :

- utilise le broadphase collision Unreal sur le canal `NinjaVolume` ;
- conserve la priorité et la sémantique de direction exactes ;
- remplace les Delay récursifs par une échéance timestampée polled ;
- neutralise les sorties avant chaque échec ;
- ne requête pas de nouveau si le propriétaire n'a pas bougé et si la génération des zones n'a pas changé ;
- force néanmoins le refresh aux appels métier explicites ;
- limite toute erreur répétée à `1/s` par instance ;
- expose un seul dispatcher après mutation complète du cache.

Le manager global Blueprint ne reste pas une condition artificielle de tick. S'il a encore des consommateurs propres, il reste leur propriétaire jusqu'à sa migration ; sa référence inutilisée n'est pas reproduite dans le composant natif.

### 2.3 Application personnage

Une fois le provider typé, un composant personnage natif applique direction et échelle via `UNinjaCharacterMovementComponent`. Il ne lit aucun champ Blueprint par réflexion.

La rotation surface-aligned est une étape séparée. Son tick termine avant le processor QAI qui possède la rotation finale des agents. Chaque early return neutralise la sortie qu'il possédait ; aucune seconde écriture cachée ne reste dans ALS.

## 3. Plan d'exécution

### J0 — Audit et frontière — terminé

- inspecter les trois assets principaux, leurs graphes, defaults, signatures et referencers ;
- vérifier les API NinjaCharacter, QPlatform, QAI et GravityScape ;
- rejeter l'activation directe du subsystem GravityScape non équivalent ;
- choisir le provider natif comme prérequis au consommateur ALS.

### J1 — Math native et durcissement GravityScape — terminé

- extraire la sélection de priorité, la direction radiale/fixe, le signe d'échelle et les règles forced en fonctions pures ;
- corriger les défauts prouvés du subsystem existant ;
- ajouter des tests sur limites de forme, priorité égale/supérieure, zone hors influence, échelle négative/nulle et unregister ;
- conserver `Enabled=False` tant que la bascule asset n'est pas prête.

Le chemin `AsyncParallelFor` dangereux a été supprimé, pas remappé. `Update_Gravity_Type` n'est sérialisé que dans la configuration du `UDeveloperSettings` et la recherche de toutes les configurations du dépôt ne trouve aucune valeur legacy `AsyncParallelFor`. Conserver une fausse valeur deprecated aurait maintenu du code mort et un comportement qui n'a aucun propriétaire actif.

Le polling dirty a également été supprimé. Les transformations et tous les setters Blueprint du producer historique rafraîchissent maintenant immédiatement la seule copie de données enregistrée ; aucune option cachée ne peut laisser une source mobile figée.

La réactivation du suivi de transform force aussi un refresh synchrone : un producer déplacé pendant la désactivation ne peut plus réutiliser une transform obsolète. La recherche Find-in-Blueprints complète des `4 694` Blueprints indexés ne trouve aucun appel à l'ancienne fonction `GS_GetIsValid`, donc cette API morte reste supprimée.

**Gate :** build disque UE 5.7 réussi et six tests `QATS.GravityScape.QangaGravity.*` verts. Aucun asset de production n'est encore reparenté à ce stade.

### J2 — Classes natives provider — terminé

- `AQLevelGravityArea` porte les formes collision natives, les enums stables `0/1` et `0/1`, la priorité, le tag, l'échelle, l'atmosphère et le `ANinjaPhysicsVolume` associé ;
- `UQGravityAreaComponent` reprend les signatures publiques du provider Blueprint, le canal résolu depuis le profil `NinjaVolume`, le cache par position/génération et les forced values ;
- la sélection garde le premier résultat à priorité égale et neutralise intégralement le cache sans zone ;
- des scopes Insights couvrent la mise à jour des formes et la requête du consommateur ;
- le test QATS natif couvre lifecycle, collision sphère/boîte, direction point/fixe, priorité runtime, zéro/négatif, force/unforce, atmosphère, génération et ownership du volume Ninja ;
- un volume Ninja fourni par l'appelant n'est jamais détruit par l'area, tandis qu'un volume spawné par elle est détruit ou remplacé sans orphan ;
- les zones QANGA restent statiques comme leurs composants Blueprint d'origine : aucun faux chemin de déplacement runtime n'est conservé.

**Gate :** build disque UE 5.7 réussi ; découverte exacte de six tests, `6/6` réussis. Les deux scratch worlds produisent uniquement les trois warnings QGM/world-data déjà connus parce qu'ils n'ont volontairement ni GameInstance ni world data configurée ; aucune assertion ni erreur GravityScape.

### J3 — Reparent et suppression du provider Blueprint — cleanup meshless terminé

- les deux redirects enum ont été appliqués à `11` referencers, revérifiés à zéro référence puis les deux User Defined Enums ont été supprimés ;
- `LevelGravityArea` hérite maintenant de `AQLevelGravityArea` et ne contient plus aucune variable, fonction, dispatcher ou nœud Blueprint ; seul son `EnvironmentSetupComponent` designer reste authored ;
- `GravityAreaComponent` hérite maintenant de `UQGravityAreaComponent` et ne contient plus aucune variable, fonction, dispatcher, composant ou nœud Blueprint ;
- le dispatcher natif conserve son paramètre exact `GravityAreaComponent`, et `GetRadiusSizeSquared` conserve l'output Blueprint exact `RadiusSq` au lieu d'introduire un `ReturnValue` incompatible ;
- le lifecycle de la zone alimente à la fois le subsystem natif et, tant qu'il reste propriétaire de ses autres consommateurs, le `GlobalGravityAreaManager` existant directement sur le `GameState` ; aucun appel par réflexion à `Lib_GravityArea` ni aucune réflexion par tick n'a été ajouté ;
- `31` Blueprints consommateurs ont été rafraîchis et sauvés à `0` erreur / `0` warning ; `GlobalGravityAreaManager` et `Lib_GravityArea` n'exposent plus de type enfant `LevelGravityArea_C` dans leurs variables, signatures ou graphes ;
- `FlyVehicleMovementComponent` a conservé tous ses liens avec les bases natives et son subsystem monde consomme maintenant directement les événements register/update/unregister du registre GravityScape, avec synchronisation initiale et sans polling ni second writer ;
- RzDirectMCP valide maintenant les dispatchers avant reparent, suit les variables authored par GUID malgré les renommages de collision, et migre atomiquement les variables, signatures et pins object child-vers-base, y compris les valeurs de map et les fonctions de `Create Event` ;
- le header public du producer GravityScape déclare explicitement son subsystem au lieu de dépendre d'un include unity accidentel ; la target `Qanga Linux Shipping` non-unity compile et lie désormais ce module ;
- les six propriétés de composants natifs de `AQLevelGravityArea` sont désormais `Transient + SkipSerialization` : les anciennes références sérialisées dans les maps ne peuvent plus écraser les sous-objets créés par le constructeur ;
- au `PostLoadSubobjects`, la zone résout strictement chaque sous-objet par owner, nom, classe et flag `RF_DefaultSubObject`, exige une occurrence unique, normalise l'ancien `CreationMethod=SimpleConstructionScript` en `Native`, puis restaure les propriétés, le root et les attaches ; les maps historiques restent compatibles sans resave massif ;
- le générateur RzDirectMCP émet ce même invariant pour les futures migrations d'actors à composants. Le reparent et `bp_refresh_all_nodes` refusent avant compilation un parent qui n'expose pas les flags et sous-objets requis ; la réparation n'accepte qu'une référence nulle, le composant du CDO généré ou son archetype exact sur le CDO parent, puis publie son compteur uniquement après succès atomique complet. Les accès aux membres du `PostLoadSubobjects` généré sont qualifiés par `this` afin qu'un nom de composant ne puisse pas entrer en collision avec les identifiants internes ; aucun actor ou package de niveau chargé n'est modifié ni sauvegardé ;
- le build disque UE 5.7 froid courant est vert avec les classes GravityScape, FlyVehicle, Ninja, QATS et RzDirectMCP ; les dix tests `QATS.GravityScape.QangaGravity.*` et le QATS de collision des noms du codegen passent sur ce binaire.

Le lancement PIE a révélé le dialogue modal Unreal et trois consommateurs J3 obsolètes. Après fermeture par la commande console `QUIT_EDITOR`, le build froid complet a réussi, l'éditeur a été relancé et la seed EasyCook avait été vidée/persistée/régénérée sans scan. Elle a depuis été rescannée pour les builds de développement : la lecture live courante contient `76 658` entrées et deux répertoires always-cook. Le cleanup J3 retirera donc uniquement l'entrée morte prouvée, sans effacer la seed entière ni relancer un scan. `CM_SpeedShake` et `SpaceshipBase` ont été reconstruits par refresh natif. Dans `QPlayerPlatform_Component`, le refresh seul ne pouvait pas résoudre deux pins homonymes : la sortie `CachedGravityLocation` a été reconnectée au nouveau pin natif `ForcedLocation`, puis l'ancien pin sérialisé `Forced Location` a été délié et supprimé. Un dernier contrat enfant obsolète a ensuite été retiré de `BP_Missile` : son cache `LastValidGravityArea` utilise désormais `QLevelGravityArea`, comme la sortie native de `Lib_GravityArea`, sans cast ni conversion intermédiaire. Les quatre assets sont maintenant sauvés et recompilés à `0` erreur / `0` warning. Le nouveau preflight RzDirectMCP retourne les erreurs chargées inline, avec opt-in explicite `allow_blueprint_errors`, et refuse une configuration PIE qui empêcherait le mono-processus demandé.

La revue de fermeture source a trouvé trois régressions supplémentaires avant validation : les redirects de `Forced Location` ciblaient un nom natif inexistant, le subsystem FlyVehicle ne lisait que les propriétés `float` alors que l'échelle native est un `double`, et le bridge manager gardait une clé/snapshot obsolète après mutation du tag ou de l'échelle. Le checkpoint courant contient les redirects vers `ForcedLocationValue`, la lecture `FNumericProperty` float/double, l'enregistrement FlyVehicle idempotent, la clé legacy enregistrée séparément avec rollback de mutation et le refus de créer silencieusement un manager lorsque le composant authored manque. Les QATS couvrent les échelles FlyVehicle `2/0/-2`.

Le scratch world QATS reste entièrement avant `BeginPlay` tant qu'un test n'a pas assemblé ses dépendances. Le test d'intégration provider crée explicitement son `GameState` et le manager legacy, puis `QATSScratchWorld::StartPlay` exécute ensemble le démarrage des world subsystems et le `NotifyBeginPlay` des actors ; les sorties anticipées ne peuvent donc plus laisser des subsystems démarrés sans `OnWorldEndPlay`. Le test auto-transform démarre et détruit explicitement son actor afin d'exercer `EndPlay`. Le probe QATS du dispatcher vérifie les broadcasts de tag, échelle, direction, atmosphère et teardown. Côté runtime, `UQGravityAreaComponent` mémorise le contrat effectif complet et diffuse aussi lorsqu'une même zone change de centre, mode, direction, échelle, tag, atmosphère ou état forced ; le double broadcast du refresh a été supprimé. Ces chemins font partie du gate `9/9` courant.

RzDirectMCP a également été durci avant de réparer les consommateurs : le démarrage PIE désactive l'authentification OnlineSubsystem qui rendait la création du PIE asynchrone. `bp_refresh_all_nodes` refuse un package déjà dirty ou une transaction étrangère, capture Blueprint/graphes/nœuds dans une transaction identifiée, compile, vérifie qu'il possède toujours cette transaction puis la finalise avant toute écriture disque ; un échec annule uniquement cet identifiant, recompile l'état restauré et ne rend le package clean que si la restauration est complète. La commande exacte `editor_console_command QUIT_EDITOR` possède maintenant une voie Slate qui valide strictement projet, instance et schéma d'arguments, répond au client, puis attend la disparition d'un modal lors du post-tick Slate. Un callback game-thread one-shot exécuté hors callback Slate appelle ensuite `IMainFrameModule::RequestCloseEditor`, ce qui conserve ses décisions de sauvegarde et évite qu'une commande différée soit interceptée puis jetée par le local player PIE. Le MainFrame émet lui-même le vrai `QUIT_EDITOR` seulement après acceptation ; la requête atomique n'est armée qu'après une réponse effectivement envoyée et est purgée aux deux extrémités d'un arrêt du bridge. Elle ne ferme pas le process par une API OS.

Le gate dedicated meshless a maintenant une réponse authored complète. L'audit live a parcouru `199` références de niveaux dans `133` worlds et `368` instances uniques sans échec : `282` zones Box, `86` zones Sphere et zéro `CustomShape`. Les `42` Blueprints referencers ont aussi été chargés et indexés ; le seul pin `CustomShape` restant est une ancienne branche de switch dans `FlyVehicleMovementComponent`, reliée au même chemin que Box et sans instance productrice. Le mode est donc mort, pas indisponible : l'enum, le composant static mesh, le chargement obligatoire de `SM_CylinderCollision`, ses propriétés/setters, ses tests et son redirect ont été supprimés à la source. Aucune redirection silencieuse `CustomShape -> Box` n'est conservée.

La source reflétée a depuis été rechargée et compilée dans l'éditeur. La lecture live ciblée du wrapper confirme uniquement son `EnvironmentSetupComponent` authored et les cinq sous-objets natifs attendus ; aucun `Detection_CustomShape` orphelin ni aucune référence `CustomShape` ne subsiste dans ce Blueprint. L'Asset Registry trouvait pourtant encore `22` referencers toutes catégories de `SM_CylinderCollision`. La chaîne GC du package pilote a invalidé l'hypothèse d'un simple import orphelin : l'ancien composant de détection avait été fusionné par l'optimiseur QLevel en un `QInstanced_HISMComponent` réel, avec deux instances visibles et le mesh encore assigné. Une simple sauvegarde ne pouvait donc rien supprimer.

La réparation est passée par le propriétaire canonique. `Q_LS_SpaceWarp`, puis les vingt autres QLevel propriétaires, ont régénéré leurs `LOptimised` depuis les niveaux sources courants via l'action QLevelEditor. L'entrée `FSoftObjectPath` exacte a été retirée de la seed EasyCook sans vider ni rescanner ses `76 658` autres entrées. L'Asset Registry est ensuite passé de `22` à zéro referencer toutes catégories ; `SM_CylinderCollision` a seulement alors été supprimé, et son absence a été revérifiée. Aucun HISM historique, produit seed ou asset mort n'est conservé.

**État validé avant J4 :** `GlobalGravityAreaManager`, `Lib_GravityArea`, `CM_SpeedShake`, `QPlayerPlatform_Component`, `SpaceshipBase`, `BP_Missile` et `LevelGravityArea` compilaient à `0` erreur / `0` warning. `L_Dev_Rz` chargeait directement avec le root et les composants historiques de `Earth_LevelGravityArea` valides, les conservait après refresh/reinstancing et dépassait l'ancien point de crash. `stop_pie` retournait `0` erreur et `0` warning dans le Message Log. Les dix QATS gravité passaient, dont le test de restauration exacte des composants historiques SCS et le QATS codegen de collision des noms. Les targets `Qanga Win64 Shipping` et `Qanga Linux Shipping` compilaient et liaient avant la suppression reflétée finale de `CustomShape`.

**Gate restant :** compiler `Qanga Linux Shipping` avec la forme reflétée et son produit mesh supprimés. Le cleanup contenu J3 est fermé. Le volume Ninja reste conservé pour les câbles et n'est pas remis en cause par la suppression de la forme morte.

### J3 bis — HUD atmosphère et altitude — corrigé

- `UQGravityAreaComponent::BeginPlay` force l'activation native après validation de l'owner. Les anciens templates Blueprint sérialisés inactifs reprennent donc leur tick `0,1 s` et actualisent leur zone pendant les déplacements, sans second scheduler ni appel manuel de refresh.
- `W_ShipFrameHud` conserve son chemin authored : `GetHasAtmosphere` pilote l'état espace et l'altitude vient de la distance WorldScape réelle. Seul l'ancien nœud océan/sol orphelin a été supprimé ; aucun plafond, seuil ou fallback HUD n'a été ajouté.
- Dans `L_Dev_Rz`, `Earth_LevelGravityArea` porte une marge atmosphérique authored de `7 070 016 cm`, soit environ `70,7 km` au-dessus du rayon WorldScape. La validation manuelle confirme une transition stable dans les deux sens et aucun état atmosphérique retenu à `110 km`.
- `BP_Missile` consomme désormais `QLevelGravityArea` sur son cache `Last Valid Gravity Area`, ce qui supprime la rupture de type entre la sortie native et le pin enfant.
- Les composants natifs sérialisés dans les niveaux sont restaurés par leur contrat `Transient + SkipSerialization` et `PostLoadSubobjects`; aucun niveau n'a été resauvegardé en masse. Le test de régression dédié exerce l'activation et les déplacements uniquement par ticks normaux.

### J4 — Application direction/échelle personnage — intégration authored joueur et AILean effectuée, gates finaux en attente

- le consommateur natif est authored sur ALS et AILean ;
- les écritures Blueprint direction/échelle remplacées sont supprimées des deux bases ;
- la rotation Blueprint remplacée par J5 est également supprimée ;
- restent les QATS et la validation gravité normale, zéro, négative, forced et QPlatform dans `L_Dev_Rz`.

Le propriétaire personnage final ne peut pas être déduit du nom `ANinjaPhysicsVolume`. L'audit source prouve que cette classe dérive de `AActor`, pas de `APhysicsVolume`, ne possède aucune collision et n'appelle pas automatiquement ses méthodes `ActorEnteredVolume` / `ActorLeavingVolume`. Le CMC ne peut donc pas la récupérer par `GetPhysicsVolume()`. En revanche, le volume spawné par `AQLevelGravityArea` n'est pas mort : `URopeCableComponent` l'énumère et appelle `GetGravity`, tandis que `ACableConnectorActor` appelle explicitement `PrimitiveEnteredVolume` / `PrimitiveLeaveVolume` pour l'embout simulé. Le spawn, la propriété, la configuration et leurs QATS d'ownership doivent donc être conservés pour ce consommateur partagé. Avant J4, la recherche live doit seulement établir si ALS/AILean appellent eux aussi `CustomNinjaPhysicsVolume`, `ActorEnteredVolume` ou `ActorLeavingVolume`; elle ne conditionne plus la survie du contrat câble. L'application direction/échelle personnage aura ensuite un propriétaire CMC explicite, une composition définie avec les modificateurs personnage existants et un chemin QPlatform séparé.

La revue statique J4 interdit de traduire aveuglément en C++ un éventuel `SetFixedGravityDirection` exécuté chaque frame. `UQGravityAreaComponent` conserve volontairement un intervalle de recherche de zone de `0,1 s`, mais `UNinjaCharacterMovementComponent` possède déjà `SetPointGravityDirection` et résout ensuite lui-même la direction radiale depuis la position courante. Le candidat natif minimal est donc une configuration événementielle du contrat CMC : mode `Point` + centre pour `PointGravity`, mode `Fixed` + direction non signée pour une zone fixe, puis échelle signée. Cela évite un second overlap, un calcul manuel par frame et le risque de marquer/répliquer une direction fixe différente à chaque tick. Les graphes live doivent confirmer la sémantique actuelle avant implémentation ; le travail continu ne restera que s'il est réellement requis par l'interpolation QPlatform ou, au gate J5, par la sortie visuelle de rotation.

Le mode point doit utiliser la surcharge vectorielle `SetPointGravityDirection(CachedGravityLocation)`, jamais `SetPointGravityDirectionFromActor`. La réplication Ninja multicast actuellement le point vectoriel et efface `GravityActor` côté clients ; la surcharge actor créerait donc un contrat différent entre serveur et clients. Pour le mode fixe, l'API C++ `SetFixedGravityDirection` exige déjà un vecteur normalisé, contrairement au wrapper K2 qui normalise son entrée.

Une lacune du provider devra être fermée pour rendre cette configuration réellement événementielle : `UpdateGravityAreaTrace` relit bien le cache lorsque la génération globale change, mais ne diffuse actuellement `GravityUpdated` que si le pointeur de zone gagnante change. Une mutation d'échelle, de centre ou de direction sur la même zone met donc le cache à jour sans réveiller le futur applicateur. J4 devra notifier sur changement du contrat effectif, ou conserver un poll natif léger sans overlap ; la comparaison explicite du contrat est préférable à un broadcast global systématique. L'absence de zone applique une échelle `0` sans setter une direction nulle, et ne doit jamais utiliser `GetGravityDirection(true)` comme preuve de gravité active puisque Ninja peut alors retourner une ancienne direction géométrique malgré l'échelle zéro.

L'échelle a déjà un second writer natif : `QModule_LegacyFacade::ApplyGravityCushion` écrit directement `UCharacterMovementComponent::GravityScale` à chaque mutation de rack. J4 doit remplacer cette collision par un compositeur unique, avec au minimum `échelle de zone signée × base personnage × modificateur CoussinGravitationnel`; laisser l'applicateur et QModule s'écraser mutuellement selon leur ordre d'exécution est interdit. Pour le CMC, la direction fournie reste non signée : Ninja inverse déjà la direction finale lorsque l'échelle calculée est négative, donc utiliser `GetGravityDirectionAttracting` en plus d'une échelle négative produirait une double inversion.

Le graphe ALS actuel calcule la gravité localement sur chaque rôle, y compris les simulated proxies. Le multicast Ninja ne rejoue pas son dernier état aux clients tardifs ; il ne peut donc pas devenir l'unique source sans ajouter un second stockage réseau. La frontière retenue conserve le calcul local événementiel sur chaque rôle et exige `bDisableGravityReplication=true` sur les deux CMC authored. Les implementations multicast Ninja de direction, échelle et alignement base refusent désormais aussi une livraison tardive lorsque ce flag local est actif ou qu'un propriétaire natif est présent ; un ancien RPC en vol ne peut plus redevenir writer sur un simulated proxy. L'échelle de base est capturée une seule fois depuis l'instance CMC réelle ; aucune relecture de l'archetype et aucun remplacement silencieux par `1.0` ne sont permis. Les échelles `double` de zone/forced et leur produit doivent être finis et représentables en `float` avant d'atteindre le CMC, sinon la sortie est neutralisée et l'erreur est explicite.

Ce compositeur reste dans `GravityScape`, déjà dépendant de `NinjaCharacter`. `QModule` lui pousse le multiplicateur du coussin et, si la preuve live l'exige, `QPlatform` lui pousse son état ; `GravityScape` ne dépend d'aucun de ces deux plugins afin d'éviter le cycle existant `QModule -> QAI -> GravityScape`. La gravité QPlatform ne peut pas être réduite à un scalaire : elle produit un vecteur de force interpolé et un lerp chaque frame. Son contrat actuel a en plus deux trous de notification à traiter si ces chemins sont actifs : le detach remet l'état/la force à zéro sans émettre un dernier `QP_OnUpdateGravity`, et `QPlatform_SetGravity` modifie la force sans broadcast. Le mode `State_NormalGravity` contient également un test impossible (`HitAlpha` borné à `[0,1]` puis comparé à `> 2`) ; la recherche live doit établir son usage avant de l'inclure et de supprimer le défaut à sa racine.

La frontière native retenue est un `UQGravityCharacterComponent` dans `GravityScape`, déclaré avec les classes existantes plutôt que dans un nouveau plugin ou une nouvelle couche. Il se lie au `UQGravityAreaComponent` déjà authored, capture une seule fois l'échelle de base du CMC Ninja, applique le mode point/fixe seulement lors d'un changement de contrat et possède l'unique écriture `GravityScale = base × zone signée × coussin`. `QModule` pousse maintenant uniquement le multiplicateur du coussin vers ce propriétaire typé ; son ancienne écriture directe du CMC est supprimée.

La source J4 courante a été revue statiquement, sans build ni exécution. `UQGravityCharacterComponent` ne ticke pas : il se lie au dispatcher natif, force une consommation initiale du cache pour fermer la course d'ordre `BeginPlay`, supprime les contrats réellement identiques et se délie lors de `EndPlay`, `DestroyComponent` ou fin de source. Il capture avant mutation l'échelle, l'état directionnel exact et les quatre flags Ninja qu'il doit posséder, puis impose réplication gravité désactivée, échelles externes ignorées, alignement sur la base coupé et retour à la gravité par défaut coupé tant que le contrat est actif. `bIgnoreOtherGravityScales=true` empêche le `ANinjaPhysicsVolume` actif de remultiplier le composite natif. Le CMC porte un propriétaire explicite du contrat natif : tant que J4 le détient, les setters directionnels concurrents et les RPC multicast tardifs sont refusés, tandis que les écritures point/fixe et la restauration passent par des méthodes qualifiées par ce propriétaire. Le no-op vérifie aussi l'état CMC réel et une mise à jour de contrat reprend une échelle brute modifiée. Le diagnostic reste limité à une fois par seconde. Le teardown reprend d'abord ces valeurs, restaure direction et échelle, restitue les quatre flags authored avec la réplication en dernier, puis libère l'ownership ; s'il a déjà perdu celle-ci, il n'écrase pas le nouvel owner. Les flags de rotation joueur/QAI restent hors de cette ownership. L'absence de zone neutralise l'échelle sans inventer de direction ; les contrats point/fixe valides fournissent à Ninja le centre ou la direction normalisée non signée.

QPlatform pousse sa force normalisée et son alpha dans ce même propriétaire. La direction interpole entre zone et plateforme, tandis que l'échelle de zone interpole vers l'échelle plateforme authored `1.0` avant multiplication par base et coussin. Hors zone, la source de zone est explicitement directionless avec une échelle nulle : une plateforme à pleine intensité reste donc valide dans l'espace. Une annulation exacte ou comprise dans la tolérance du nœud Blueprint reste un override actif mais neutre, direction et échelle nulles, puis récupère normalement au-delà du midpoint. QPlatform garde la cible exacte et un token monotone afin qu'un callback synchrone, une destruction ou un detach ne puisse ni ressusciter un override retiré ni nettoyer le mauvais composant.

La source QATS adjacente couvre le calcul pur et le contrat personnage réel : point positif, location forcée, coussin, non-compounding de la base, fixe positif/négatif, gravité zéro hors zone, plateforme hors zone, midpoint neutre et récupération, invalidation, restauration exacte et teardown réentrant. Le contrat dédié couvre les quatre flags concurrents, dont l'isolation/restauration des échelles de physics volume, un départ déjà conforme, le refus d'un setter directionnel concurrent, y compris depuis un delegate synchrone owned, la reprise d'une échelle brute mutée au prochain événement provider, la libération de l'ownership et la restitution exacte des deux configurations. La requête surface devient immédiatement `Unavailable` si cet owner est perdu. Le build Editor froid UHT/adaptive non-unity passe et les `15/15` tests `QATS.GravityScape.*` sont verts sur cette source.

La bascule authored joueur est appliquée. `ALS_Base_CharacterBP` possède désormais `UQGravityCharacterComponent`, son CDO effectif porte `bDisableGravityReplication=true` et `bSmoothAlignToGravity=true`, le Tick contourne l'ancien bloc gravité, et l'event binding, `UpdateNinjaGravityDirection`, ses appels et ses six variables exclusives sont supprimés.

Une régression packagée a prouvé que ce cleanup avait été réintroduit partiellement dans l'asset joueur : le Tick et `GravityUpdated` rappelaient encore l'applicateur Blueprint, qui écrivait directement `GravityScale=0.0015` pendant que le contrat natif attendait `0` au décrochage d'une plateforme. Le log Development montrait la reprise d'ownership au même moment ; la gravité native n'était donc pas en cause et ajouter une réassertion dans QPlatform aurait conservé les deux writers. L'effet plus fort en Shipping est cohérent avec ce writer par frame et le build débridé, sans constituer à lui seul une mesure de performance.

Le cleanup courant a été rejoué dans une transaction asset unique. La lecture gravité et son cache Tick morts ont été supprimés, les six sorties ALS vivantes rejoignent directement la logique de mouvement aval, l'événement composant sans autre consommateur a été retiré, puis la fonction et ses six variables ont été supprimées. L'asset persistant passe de `4 424` à `4 099` nœuds, son `TickGraph` de `99` à `93`, et sa recherche exacte ne retourne plus aucun appel `UpdateNinjaGravityDirection`. Les `22` Blueprints référents directs recompilent à `0` erreur / `0` warning sans nouveau package dirty.

La passe `QATS.GravityScape` est verte à `15/15`. Après redémarrage sur les dernières classes, elle repasse à `15/15` avec zéro warning ; les suites adjacentes QModule `8/8`, QSystem `8/8` et les deux régressions QAI de récupération passent également sans diagnostic. Dans `L_Dev_Rz`, un pickup réel spawné sous PIE à `360 FPS` a subi cinq cycles toit, detach et réattachement : l'actor attaché est exact à chaque entrée, l'état et la référence sont vides à chaque sortie, aucune mutation externe du contrat Ninja n'est apparue dans le log, et `stop_pie` retourne `0` erreur / `0` warning. Un smoke PIE frais après l'intégration des corrections packagées se ferme lui aussi avec un Message Log `0/0` ; cette assertion ne couvre pas les autres catégories de l'Output Log. Cette preuve ferme uniquement le runtime local joueur/QPlatform dans l'Editor.

Les mesures antérieures des phases QPlatform sous la milliseconde ne fermaient pas le gel packagé. Les nouveaux logs `latest_dev_crash_and_logs/QANGA_Session1.log` et `QANGA_Session2.log` isolent un autre chemin : `UpdateSmoothGravityAlignment -> UpdateComponentRotation -> FloorSweepTest -> Chaos::FindGeometryOpposingNormal`. Dans la seconde session, l'assertion de normale invalide immobilise le thread pendant `2,643 s`. Le caller d'alignement fournit une sphère, mais le mode flat-base la traite comme une capsule et lit un `HalfHeight` non initialisé dans l'union `FCollisionShape` ; la box envoyée à Chaos dépend alors des données résiduelles de pile. Le contrat à restaurer est une conversion flat-base réservée aux capsules, en préservant les sphères d'alignement. La validation packagée reste ouverte.

Le correctif source réserve maintenant la conversion flat-base aux formes `IsCapsule()` ; les sphères passent directement à l'API de sweep du monde. Aucun changement QPlatform, seuil de taille, suppression d'assertion ou modification moteur. Le nouveau QATS `QATS.GravityScape.NinjaFloorSweep.ShapeSafety` compare les impacts sur un sol physique entre sphère directe, sphères conservant des hauteurs de capsule résiduelles `0`/`200 cm`, et capsule flat-base valide. La suite contient désormais `16` tests en source ; les bilans `15/15` précédents restent historiques. Revue statique et whitespace effectués ; ce correctif et ce nouveau test ne sont pas compilés ni exécutés ici, le propriétaire ayant repris la validation dans sa prochaine version packagée.

Retour du propriétaire sur le build Development suivant : les transitions de toit semblent bonnes. Le log Steam `QANGA.log` se terminant le 5 septembre à `00:27 UTC` ne contient plus la pile `FloorSweepTest -> UpdateComponentRotation`. Une assertion de normale Chaos subsiste à `23:47:30 UTC`, mais depuis un `GeomSweepMulti` sphérique asynchrone sur worker, pas depuis l'alignement Ninja. Une assertion GPUScene, une mutation de conteneur pendant itération et un crash `MallocBinned2 Corruption Canary` restent distincts et non corrigés par cette tranche. Le retour utilisateur et l'absence de l'ancienne pile soutiennent le correctif ciblé ; ils ne prouvent pas une session globalement sans erreur ni une validation Shipping.

Correction du bilan Slate précédent : la configuration courante conserve `Slate.EnableGlobalInvalidation=1` dans `[SystemSettings]` et `0` dans `[SystemSettingsEditor]` ; le retour annoncé au mode standard dans tous les targets n'était pas effectif. La première session contient encore `1 308` assertions `WidgetPtr`, la seconde aucune. Ce défaut UI reste distinct du défaut de sweep prouvé dans les deux sessions ; ni sa correction ni la disparition de tous les hangs ne sont revendiquées ici. Les diagnostics temporaires QPlatform restent retirés et l'énumération unique des actors attachés reste conservée.

La gravité de plateforme utilise désormais le contexte typé déjà produit par GravityScape et le mode stationnaire autoritaire du composant de mouvement véhicule. Dans une zone atmosphérique, un véhicule non stationnaire dont l'axe haut passe sous l'horizon ne retourne plus le personnage : l'override conserve sa force mais reprend la direction de surface planétaire, avec hystérésis autour de 90 degrés. Le comportement local historique est conservé dans l'espace, en mode stationnaire, pour les plateformes sans composant véhicule et lorsque le contexte de surface est explicitement indisponible. Ce gate reste à valider dans une nouvelle build packagée sur un véhicule retourné, successivement en atmosphère, en mode stationnaire et hors atmosphère.

Un probe PIE antérieur dans `L_Persistent_Universe` retrouvait le composant natif instancié avec `BaseGravityScale=1`, `bDisableGravityReplication=true`, les deux writers concurrents à `false`, smooth joueur actif et aucune erreur sur la page Message Log de cette session. Le correctif RzDirectMCP qui ignore `IsHiddenEd()` uniquement dans les game worlds avait été validé par ce résultat ; les editor worlds conservent leur filtre historique. Un autre probe antérieur avait mesuré un axe Up superposé à la normale radiale terrestre à environ `0,000002°`. Ces probes restent historiques ; la preuve locale courante est le stress QPlatform ci-dessus.

La bascule authored de la base AILean est appliquée. `ALS_Base_CharacterBP_AILean` possède désormais `UQGravityCharacterComponent`; son CDO effectif porte `bDisableGravityReplication=true`, `bSmoothAlignToGravity=false`, `bAlignComponentToGravity=false`, `bAlignGravityToBase=false` et `bRevertToDefaultGravity=false`. QAI reste donc l'unique propriétaire de sa rotation. Les six sorties de mode vivantes contournent directement l'ancien bloc gravité, tandis que l'îlot `In Vehicle` reste déconnecté comme avant. L'event binding, `UpdateNinjaGravityDirection`, ses appels et ses six variables exclusives sont supprimés. La base a été recompilée immédiatement à `0` erreur / `0` warning.

Le probe PIE d'un enfant réel avait empêché une fausse fermeture : `AI_GuardCaptain_AILean` instancie encore son ancien `CharMoveComp.bDisableGravityReplication=false`, comme les CDO de plusieurs descendants, et l'ancien applicateur validateur se désactivait alors avec `BaseGravityScale=0`. La lecture du chemin de compilation Blueprint a ensuite prouvé que la reconstruction propage d'abord le parent puis restaure les anciennes différences du CDO enfant ; forcer une pseudo-propagation RzDirectMCP ou patcher les douze enfants aurait donc créé le mauvais propriétaire. `UQGravityCharacterComponent` possède maintenant ces quatre exigences au runtime, indépendamment des deltas historiques des descendants, et restaure leur état exact au teardown. Les deux bases ALS et les douze descendants AILean ont été recompilés ; après la réparation SCS transitoire du premier passage, le second passage complet est propre à `0` erreur / `0` warning. Un enfant AILean réel reste à reprober en déplacement et en combat avant fermeture runtime.

RzDirectMCP continue de lire les composants natifs depuis le CDO effectif de chaque classe générée, tout en identifiant leur origine par le parent natif et en excluant les doublons SCS/UCS. Il rapporte donc correctement les divergences historiques sans tenter de les réécrire. La piste de propagation native des defaults est abandonnée et aucun fallback d'authoring enfant par enfant n'est conservé.

### J5 — Rotation et ordre QAI — source et QATS validés ; runtime/réseau à revalider

- source native d'alignement surface et frontières de rôles implémentées puis revues statiquement ;
- ordre QAI, requête de surface typée et suppression de la compensation proxy implémentés puis revus statiquement ;
- les deux profils authored sont intégrés et `UpdateNinjaGravityDirection` ainsi que ses branches exclusives sont supprimés dans ALS et AILean ;
- restant : valider un cyborg AILean réel en déplacement et en combat sans reprendre le contrôle de sa rotation à QAI.

La source native J5 expose la requête de surface consommée par QAI et QATS. `UNinjaCharacterMovementComponent` porte un opt-in `bSmoothAlignToGravity` désactivé par défaut. Il reproduit les seuils authored asymétriques (`>= +0,15` ou `< -0,15`), le mapping signé `0,15..0,5 -> 0,5..6` et le `FInterpTo` de vitesse à `4` depuis la valeur initiale `0`, y compris lorsque l'alignement est momentanément inactif. La rotation elle-même interpole en quaternion puis passe par `UpdateComponentRotation`, donc conserve le sweep capsule et n'introduit aucun `SetActorRotation` parallèle sur le joueur. Ce chemin compile dans le build Editor froid courant et sa couverture QATS est verte ; la preuve perceptuelle et réseau reste ouverte.

Ce mode smooth possède une seule phase d'écriture dans `OnMovementUpdated`. Tant qu'il est actif, les chemins immédiats floor/gravity renvoient l'axe capsule courant, même si un graphe remet `bAlignComponentToGravity` ou `bAlignComponentToFloor` à vrai. Seuls l'autorité et le pawn localement contrôlé écrivent ; les simulated proxies consomment la rotation acteur répliquée. Une gravité nulle ou directionless conserve l'orientation courante, sans world-up ni direction géométrique périmée, et le signe négatif n'est pas inversé deux fois.

QAI consomme maintenant une requête typée à trois états depuis `UQGravityCharacterComponent` : up valide, contrat directionless, ou contrat indisponible/invalide. Le cyborg écrit sa rotation corps uniquement sur l'autorité. À l'arrêt, il reprojette son heading courant et continue d'aligner l'up valide ; si ce heading est parallèle au nouvel up, le resolver pur utilise l'axe droit de la même rotation comme tangente déterministe afin que l'alignement ne soit pas sauté. En directionless il n'écrit rien. Un contrat indisponible ne tombe plus sur l'up acteur : il laisse la rotation intacte et produit une erreur explicite limitée à une par seconde et par agent. Le composant GravityScape est résolu une fois au `BeginPlay`, pas recherché à chaque tick.

La compensation client qui réappliquait `GetReplicatedMovement().Rotation` après ALS est supprimée. `CombatFacing` reste vivant comme source d'interpolation stable et la prerequisite après l'acteur reste justifiée par l'ordre gameplay/AnimBP et la vélocité FPM. Le bridge `QAI_GravityAreaResolver`, sa réflexion vers `Lib_GravityArea`, son cache par actor et ses fallbacks actor/FPM/world-up sont supprimés. Tous ses consommateurs production utilisent maintenant `UGravityScape_SubSystem::QueryGravityAtWorldPosition`, qui sélectionne le snapshot canonique sous verrou et retourne uniquement des valeurs immuables `Valid`, `Directionless` ou `Unavailable`, sans `UObject` exposé aux workers.

La direction fixe publiée ne peut plus devenir périmée quand un Blueprint ou un système runtime tourne directement `FixedGravityDirection_Arrow`. L'actor lie `TransformUpdated` pendant le jeu, retire exactement ce binding au teardown, reconfigure le volume Ninja et republie le snapshot canonique sans tick ni polling. Le QATS provider prouve que la génération avance et que la requête monde renvoie immédiatement la nouvelle direction.

Les QATS adjacents couvrent les seuils, mappings, lissages, quaternions et rôles, la tangente QAI lorsque forward et up sont parallèles, puis la requête de surface unavailable/valide/directionless sur un personnage réel. Le calcul négatif a été redérivé depuis `FMath::FInterpTo` et sa tolérance float est explicite. Ces assertions font partie du gate `15/15` vert. Les propriétaires authored et toute la hiérarchie Blueprint doivent encore être recompilés puis revalidés dans le runtime IA.

### J6 — Essential values et mouvement ALS — build/QATS verts ; runtime/réseau à revalider

- `UQGravityLocomotionComponent` est non-ticking et consomme uniquement `OnCharacterMovementUpdated` ;
- le calcul et la publication native/reflected sont revus statiquement ;
- le composant authored et le cleanup Blueprint antérieurs doivent être revalidés sans mutation supplémentaire ;
- build Editor froid et QATS exécutés avec succès ; build Linux, compile Blueprint fraîche, PIE et test réseau restent ouverts ;
- AILean n'est pas traité : son îlot Tick mort ne doit pas être activé par symétrie.

La frontière finale est un composant locomotion frère de `UQGravityCharacterComponent` dans le module existant `GravityScape`. Ninja reste propriétaire du mouvement. J6 ne recrée pas un second modèle gravity-space : les seuls produits vivants de cette verticale sont les trois variables historiques consommées par l'AnimBP et le `PawnNoiseEmitter`, soit `Acceleration`, `Speed` et `IsMoving`.

Le composant ne tick pas. Il se lie à `ACharacter::OnCharacterMovementUpdated`, émis après `UNinjaCharacterMovementComponent::OnMovementUpdated` et la fermeture du scoped movement update. Il calcule exactement l'expression Blueprint `(CurrentVelocity - OldVelocity) / DeltaSeconds`, sans cache de frame, correction quaternion, décomposition planaire/verticale ni sorties sans consommateur. `Speed` reste la longueur 3D de la vélocité et `IsMoving` utilise le test strict `Speed > 1 cm/s`. Chaque callback possède son propre `OldVelocity`. Les entrées non finies, les résultats arithmétiques non finis et une vitesse `double` non représentable par le bridge `float` neutralisent atomiquement les trois sorties avant publication.

Le test dedicated à deux joueurs a révélé une frontière manquante sur les simulated proxies. `ALS_Base_CharacterBP` désactive la réplication de mouvement Engine parce que `SmoothTSync` possède le transport du transform ; le CMC ne diffusait donc jamais `OnCharacterMovementUpdated` sur ces copies et J6 conservait `Speed=0` / `IsMoving=false`, ce qui figeait les jambes malgré le déplacement de l'actor. Après application du transform et de la vélocité répliqués, la branche non-authority de `SmoothTSync` diffuse maintenant une fois le callback Engine avec la position et la vélocité capturées avant mutation. Le pawn propriétaire, l'autorité qui simule localement et le standalone gardent exclusivement le callback CMC existant ; aucune accélération synthétique ni second writer ALS n'est ajouté. La compilation Live Coding et le QATS `ComponentLifecycleAndSnapshot` sont verts ; l'observation visuelle entre deux clients dedicated dans un build frais reste le gate runtime.

Le bridge de compatibilité résout une seule fois les trois propriétés reflected sur la classe owner. Il accepte `Speed` en float ou double, refuse une publication partielle et produit une erreur exploitable si le contrat exact manque. Il ne recherche ni composant SCS ni classe `ULIVECODING_*`. La neutralisation est publiée elle aussi afin que les consommateurs Blueprint ne gardent aucune valeur périmée. Il n'existe pas de gate dedicated server sur cette verticale : `IsMoving` commande encore le `PawnNoiseEmitter`, qui est un side effect gameplay et non une présentation visuelle.

`ControlRotation`, `AimYawRate` et le call `PawnNoiseEmitter` restent Blueprint. `MovementInputAmount` et `HasMovementInput` ne sont pas recréés : leur ancienne chaîne avait zéro exec entrant. Les anciens helpers rotation-corrected/gravity-space, leurs sorties, leurs états de frame et leurs tests spéculatifs ont été supprimés de la source au lieu d'être conservés désactivés.

Le checkpoint authored antérieur indiquait `25` nœuds dans `SetEssentialValues` et `4` dans `CacheValues`. Le batch atomique avait retiré les trois setters, les getters natifs intermédiaires et les commentaires devenus morts, reconnecté directement la sortie `then_3` à la branche du bruit avec un getter `IsMoving`, puis réduit `CacheValues` au seul cache `PreviousAimYaw`. La même transaction avait supprimé la fonction `CalculateAcceleration` et la variable `PreviousVelocity`. Ce résultat Blueprint historique n'a pas été revalidé dans la présente passe.

Le Message Log d'une session PIE antérieure avait prouvé pourquoi ce chemin intermédiaire devait disparaître : la propriété générée `GravityLocomotion` était vide alors que l'instance native homonyme existait, et les trois getters produisaient en boucle `Accessed None` dans `SetEssentialValues`. La propriété reflected attendait une classe `ULIVECODING_*`, signe d'une réinstanciation UCLASS que le SCS chargé ne pouvait pas réconcilier sans redémarrage froid. Aucun guard Blueprint ni auto-affectation temporaire ne sera ajouté. Le bridge natif courant ne dépend pas de cette référence SCS ; la prochaine preuve runtime doit partir d'un éditeur froid.

La recherche Find-in-Blueprints courante ne trouve ces deux symboles que dans `ALS_Base_CharacterBP`. Comme l'index projet n'est volontairement pas rescanné, cette passe globale reste partielle. Le contrôle exact charge cependant les `22` Blueprints qui référencent directement la base et les recherche sans compiler ni sauvegarder : `22/22` sont indexés sans problème et aucun ne lit `PreviousVelocity` ni n'appelle `CalculateAcceleration`.

Le QATS source `ComponentLifecycleAndSnapshot` utilise désormais un owner natif qui expose le bridge reflected complet. Il couvre le premier callback valide, l'utilisation effective de `OldVelocity`, un sous-step, les deux côtés du seuil strict de `1 cm/s`, le delta nul, un vecteur NaN, l'overflow d'accélération avec entrées finies, la conversion `double -> float` non représentable, la récupération suivante et la neutralisation native/reflected au teardown. Il passe dans le gate complet `15/15` courant.

Les implémentations C++ [`ALS-Refactored`](https://github.com/Sixze/ALS-Refactored) et [`ALS-Community`](https://github.com/PanicPetal/ALS-Community) restent des références utiles pour les futures verticales gait/view/rotation, sans reparent ni import aveugle. Leur existence n'impose pas des produits supplémentaires à ce slice : le contrat QANGA courant reste défini par ses consommateurs réels.

Reste à faire pour J6 : build Linux Shipping non-unity, compile de la hiérarchie Blueprint, puis validation runtime de `Acceleration`, `Speed`, `IsMoving`, du bruit de déplacement et des consommateurs AnimBP. Les gates Message Log, PIE et réseau restent également ouverts.

### J7 — Validation finale et cleanup — à faire

Le jetpack autorise le contrat `fuel minimum OU zéro-g` et ne s'arrête plus explicitement à l'entrée dans l'espace. Son macro `IsAtSpace` lit désormais `UQGravityAreaComponent::CachedGravityScale == 0.0`, avec égalité exacte : aucune zone et les zones explicitement à zéro permettent le vol spatial, tandis que les zones à gravité faible ou négative conservent le vol activé par l'input et le contrôle de fuel. La présence d'atmosphère ne décide pas du mode de déplacement.

Le log Development packagé signale l'arrivée sur `MoonScape` à `19:11:50` puis `19:12:06` UTC. L'inspection sémantique de `L_Moon` confirme une zone native `MoonGravity`, à échelle `0.331999987`, direction vers le centre, rayon `59 383 026 cm` et sans atmosphère. L'ancien prédicat `!GetHasAtmosphere` déclenchait donc `OnGravityUpdate -> SetHoverMode -> StartHover -> MOVE_Flying` malgré cette gravité valide. Le nouveau prédicat emprunte le chemin normal de sortie du vol spatial : sans input de propulsion, `CancelFlying -> StopHover -> MOVE_Falling`. Les deux nœuds atmosphère/NOT obsolètes sont supprimés. Compilation et sauvegarde du Blueprint : zéro erreur, zéro warning ; contrôle des liens et des transitions effectué. Aucun niveau ni source C++ modifié pour cette correction. Le comportement dans le prochain build packagé reste à confirmer par le test utilisateur, notamment espace/Lune/espace et relâchement du jetpack sur une planète sans atmosphère.

Le test PIE suivant a néanmoins invalidé cette première causalité comme explication complète de l'immobilité. En gravité exactement nulle, `UNinjaCharacterMovementComponent::PhysFlying` et `PhysFalling` effaçaient `Acceleration` et `Velocity` puis interrompaient leur intégration dès que `GetGravityDirection(false)` retournait zéro. Cette branche annulait dans le même tick les forces, impulsions et inputs déjà accumulés par DynamicFlight. Elle est supprimée : une direction nulle est désormais le contrat zéro-g valide, l'accélération de gravité reste nulle et les chemins physiques normaux intègrent la propulsion et l'inertie. La sonde avant/après sur le même pawn en `MOVE_Flying`, `GravityScale=0` et via l'événement authored `Char_MoveForwardBackward` passe de vélocité strictement nulle à un déplacement tridimensionnel effectif. Le QATS GravityScape complet reste vert à `15/15`, le Message Log de sortie PIE est propre et la validation manuelle utilisateur confirme le déplacement jetpack dans `L_Dev_Rz`. La parité d'un nouveau build packagé et le retour atmosphérique restent ouverts.

- Standalone, Listen Server et Dedicated Server + deux clients dans `L_Dev_Rz` ;
- parité direction, échelle, axe haut, état cached, forced values et rotation finale ;
- Message Log automatique propre à chaque `stop_pie` ;
- capture Insights comparable avant/après ;
- suppression de chaque graphe, variable, enum, bridge et asset devenu sans propriétaire.

## 4. Gates de non-régression

- aucune activation implicite de GravityScape pour les mondes ou consommateurs non migrés ;
- aucune requête par frame ajoutée ;
- aucun calcul manuel parallèle à une API NinjaCharacter ou collision Unreal existante ;
- aucune réflexion vers `Lib_GravityArea` dans le nouveau chemin natif ;
- aucune perte de la priorité, du signe de gravité, des overrides ou de `HasAtmosphere` ;
- aucune course d'écriture ALS/QAI ;
- aucun Blueprint remplacé laissé désactivé mais présent.
