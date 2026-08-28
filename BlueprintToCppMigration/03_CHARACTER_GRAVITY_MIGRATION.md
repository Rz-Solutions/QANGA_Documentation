# Troisième migration Blueprint vers C++ — personnage et gravité

- **État :** J0/J1/J2 TERMINÉS — checkpoint J3 runtime sain ; gate dedicated meshless reporté, source J4 validée sans intégration authored
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

- `AQLevelGravityArea` porte les formes collision natives, les enums stables `0/1` et `0/1/2`, la priorité, le tag, l'échelle, l'atmosphère et le `ANinjaPhysicsVolume` associé ;
- `UQGravityAreaComponent` reprend les signatures publiques du provider Blueprint, le canal résolu depuis le profil `NinjaVolume`, le cache par position/génération et les forced values ;
- la sélection garde le premier résultat à priorité égale et neutralise intégralement le cache sans zone ;
- des scopes Insights couvrent la mise à jour des formes et la requête du consommateur ;
- le test QATS natif couvre lifecycle, collision sphère/boîte/custom, direction point/fixe, priorité runtime, zéro/négatif, force/unforce, atmosphère, génération et ownership du volume Ninja ;
- un volume Ninja fourni par l'appelant n'est jamais détruit par l'area, tandis qu'un volume spawné par elle est détruit ou remplacé sans orphan ;
- les zones QANGA restent statiques comme leurs composants Blueprint d'origine : aucun faux chemin de déplacement runtime n'est conservé.

**Gate :** build disque UE 5.7 réussi ; découverte exacte de six tests, `6/6` réussis. Les deux scratch worlds produisent uniquement les trois warnings QGM/world-data déjà connus parce qu'ils n'ont volontairement ni GameInstance ni world data configurée ; aucune assertion ni erreur GravityScape.

### J3 — Reparent et suppression du provider Blueprint — checkpoint sain, gates dedicated/runtime en attente

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

Le lancement PIE a révélé le dialogue modal Unreal et trois consommateurs J3 obsolètes. Après fermeture par la commande console `QUIT_EDITOR`, le build froid complet a réussi, l'éditeur a été relancé et la seed EasyCook a été vidée/persistée/régénérée sans scan ; `Entries` et `AlwaysCookDirectories` sont vides. `CM_SpeedShake` et `SpaceshipBase` ont été reconstruits par refresh natif. Dans `QPlayerPlatform_Component`, le refresh seul ne pouvait pas résoudre deux pins homonymes : la sortie `CachedGravityLocation` a été reconnectée au nouveau pin natif `ForcedLocation`, puis l'ancien pin sérialisé `Forced Location` a été délié et supprimé. Un dernier contrat enfant obsolète a ensuite été retiré de `BP_Missile` : son cache `LastValidGravityArea` utilise désormais `QLevelGravityArea`, comme la sortie native de `Lib_GravityArea`, sans cast ni conversion intermédiaire. Les quatre assets sont maintenant sauvés et recompilés à `0` erreur / `0` warning. Le nouveau preflight RzDirectMCP retourne les erreurs chargées inline, avec opt-in explicite `allow_blueprint_errors`, et refuse une configuration PIE qui empêcherait le mono-processus demandé.

La revue de fermeture source a trouvé trois régressions supplémentaires avant validation : les redirects de `Forced Location` ciblaient un nom natif inexistant, le subsystem FlyVehicle ne lisait que les propriétés `float` alors que l'échelle native est un `double`, et le bridge manager gardait une clé/snapshot obsolète après mutation du tag ou de l'échelle. Le checkpoint courant contient les redirects vers `ForcedLocationValue`, la lecture `FNumericProperty` float/double, l'enregistrement FlyVehicle idempotent, la clé legacy enregistrée séparément avec rollback de mutation et le refus de créer silencieusement un manager lorsque le composant authored manque. Les QATS couvrent les échelles FlyVehicle `2/0/-2` et un overlap custom-shape inside/outside réel.

Le scratch world QATS reste entièrement avant `BeginPlay` tant qu'un test n'a pas assemblé ses dépendances. Le test d'intégration provider crée explicitement son `GameState` et le manager legacy, puis `QATSScratchWorld::StartPlay` exécute ensemble le démarrage des world subsystems et le `NotifyBeginPlay` des actors ; les sorties anticipées ne peuvent donc plus laisser des subsystems démarrés sans `OnWorldEndPlay`. Le test auto-transform démarre et détruit explicitement son actor afin d'exercer `EndPlay`. Le probe QATS du dispatcher vérifie les broadcasts de tag, échelle, direction, atmosphère et teardown. Côté runtime, `UQGravityAreaComponent` mémorise le contrat effectif complet et diffuse aussi lorsqu'une même zone change de centre, mode, direction, échelle, tag, atmosphère ou état forced ; le double broadcast du refresh a été supprimé. Ces chemins font partie du gate `9/9` courant.

RzDirectMCP a également été durci avant de réparer les consommateurs : le démarrage PIE désactive l'authentification OnlineSubsystem qui rendait la création du PIE asynchrone. `bp_refresh_all_nodes` refuse un package déjà dirty ou une transaction étrangère, capture Blueprint/graphes/nœuds dans une transaction identifiée, compile, vérifie qu'il possède toujours cette transaction puis la finalise avant toute écriture disque ; un échec annule uniquement cet identifiant, recompile l'état restauré et ne rend le package clean que si la restauration est complète. La commande exacte `editor_console_command QUIT_EDITOR` possède maintenant une voie Slate qui valide strictement projet, instance et schéma d'arguments, répond au client, puis attend la disparition d'un modal lors du post-tick Slate. Un callback game-thread one-shot exécuté hors callback Slate appelle ensuite `IMainFrameModule::RequestCloseEditor`, ce qui conserve ses décisions de sauvegarde et évite qu'une commande différée soit interceptée puis jetée par le local player PIE. Le MainFrame émet lui-même le vrai `QUIT_EDITOR` seulement après acceptation ; la requête atomique n'est armée qu'après une réponse effectivement envoyée et est purgée aux deux extrémités d'un arrêt du bridge. Elle ne ferme pas le process par une API OS.

Un gate dedicated supplémentaire est ouvert : la politique meshless exclut `StaticMesh` du cook serveur, alors que le constructeur natif charge et `checkf` actuellement `SM_CylinderCollision` même pour une zone Sphere/Box. Un simple guard null casserait silencieusement `CustomShape`, et retirer l'exclusion inclurait toutes les static meshes. Après réouverture de l'éditeur, l'audit live doit établir si des instances authored utilisent réellement `CustomShape` : si non, ce mode mort sera supprimé ; si oui, sa collision sera portée par une donnée/forme server-safe sans asset `UStaticMesh`. La politique dedicated meshless reste inchangée.

**Checkpoint de pause :** `GlobalGravityAreaManager`, `Lib_GravityArea`, `CM_SpeedShake`, `QPlayerPlatform_Component`, `SpaceshipBase`, `BP_Missile` et `LevelGravityArea` compilent à `0` erreur / `0` warning. `L_Dev_Rz` charge directement avec les six références et le root de `Earth_LevelGravityArea` valides, conserve ces références après refresh/reinstancing, puis reste en PIE au-delà de l'ancien crash. `stop_pie` retourne `0` erreur et `0` warning dans le Message Log. Les dix QATS gravité passent, dont le nouveau test qui simule les composants historiques SCS, efface leurs propriétés/root et vérifie leur restauration native exacte ; le QATS codegen couvre aussi les collisions entre noms de composants et identifiants internes. Les targets `Qanga Win64 Shipping` et `Qanga Linux Shipping` compilent et lient. Le provider Blueprint remplacé est nettoyé, les types enfant obsolètes de ces consommateurs ont été retirés, aucune map n'a été resauvegardée et aucune intégration authored J4 n'a été commencée.

**Gate restant à la reprise :** audit live des usages `CustomShape` et du volume Ninja, puis résolution server-safe du gate dedicated meshless. Le gate runtime de chargement, reinstancing et démarrage PIE est fermé ; J3 ne sera entièrement fermé qu'après la preuve dedicated.

### J3 bis — HUD atmosphère et altitude — corrigé

- `UQGravityAreaComponent::BeginPlay` force l'activation native après validation de l'owner. Les anciens templates Blueprint sérialisés inactifs reprennent donc leur tick `0,1 s` et actualisent leur zone pendant les déplacements, sans second scheduler ni appel manuel de refresh.
- `W_ShipFrameHud` conserve son chemin authored : `GetHasAtmosphere` pilote l'état espace et l'altitude vient de la distance WorldScape réelle. Seul l'ancien nœud océan/sol orphelin a été supprimé ; aucun plafond, seuil ou fallback HUD n'a été ajouté.
- Dans `L_Dev_Rz`, `Earth_LevelGravityArea` porte une marge atmosphérique authored de `7 070 016 cm`, soit environ `70,7 km` au-dessus du rayon WorldScape. La validation manuelle confirme une transition stable dans les deux sens et aucun état atmosphérique retenu à `110 km`.
- `BP_Missile` consomme désormais `QLevelGravityArea` sur son cache `Last Valid Gravity Area`, ce qui supprime la rupture de type entre la sortie native et le pin enfant.
- Les composants natifs sérialisés dans les niveaux sont restaurés par leur contrat `Transient + SkipSerialization` et `PostLoadSubobjects`; aucun niveau n'a été resauvegardé en masse. Le test de régression dédié exerce l'activation et les déplacements uniquement par ticks normaux.

### J4 — Application direction/échelle personnage — source compilée et testée, intégration authored en attente

- brancher le consommateur natif sur ALS et AILean ;
- supprimer dans les deux bases les écritures Blueprint direction/échelle remplacées ;
- conserver temporairement la rotation Blueprint jusqu'au gate suivant ;
- valider gravité normale, zéro, négative, forced et QPlatform dans `L_Dev_Rz`.

Le propriétaire personnage final ne peut pas être déduit du nom `ANinjaPhysicsVolume`. L'audit source prouve que cette classe dérive de `AActor`, pas de `APhysicsVolume`, ne possède aucune collision et n'appelle pas automatiquement ses méthodes `ActorEnteredVolume` / `ActorLeavingVolume`. Le CMC ne peut donc pas la récupérer par `GetPhysicsVolume()`. En revanche, le volume spawné par `AQLevelGravityArea` n'est pas mort : `URopeCableComponent` l'énumère et appelle `GetGravity`, tandis que `ACableConnectorActor` appelle explicitement `PrimitiveEnteredVolume` / `PrimitiveLeaveVolume` pour l'embout simulé. Le spawn, la propriété, la configuration et leurs QATS d'ownership doivent donc être conservés pour ce consommateur partagé. Avant J4, la recherche live doit seulement établir si ALS/AILean appellent eux aussi `CustomNinjaPhysicsVolume`, `ActorEnteredVolume` ou `ActorLeavingVolume`; elle ne conditionne plus la survie du contrat câble. L'application direction/échelle personnage aura ensuite un propriétaire CMC explicite, une composition définie avec les modificateurs personnage existants et un chemin QPlatform séparé.

La revue statique J4 interdit de traduire aveuglément en C++ un éventuel `SetFixedGravityDirection` exécuté chaque frame. `UQGravityAreaComponent` conserve volontairement un intervalle de recherche de zone de `0,1 s`, mais `UNinjaCharacterMovementComponent` possède déjà `SetPointGravityDirection` et résout ensuite lui-même la direction radiale depuis la position courante. Le candidat natif minimal est donc une configuration événementielle du contrat CMC : mode `Point` + centre pour `PointGravity`, mode `Fixed` + direction non signée pour une zone fixe, puis échelle signée. Cela évite un second overlap, un calcul manuel par frame et le risque de marquer/répliquer une direction fixe différente à chaque tick. Les graphes live doivent confirmer la sémantique actuelle avant implémentation ; le travail continu ne restera que s'il est réellement requis par l'interpolation QPlatform ou, au gate J5, par la sortie visuelle de rotation.

Le mode point doit utiliser la surcharge vectorielle `SetPointGravityDirection(CachedGravityLocation)`, jamais `SetPointGravityDirectionFromActor`. La réplication Ninja multicast actuellement le point vectoriel et efface `GravityActor` côté clients ; la surcharge actor créerait donc un contrat différent entre serveur et clients. Pour le mode fixe, l'API C++ `SetFixedGravityDirection` exige déjà un vecteur normalisé, contrairement au wrapper K2 qui normalise son entrée.

Une lacune du provider devra être fermée pour rendre cette configuration réellement événementielle : `UpdateGravityAreaTrace` relit bien le cache lorsque la génération globale change, mais ne diffuse actuellement `GravityUpdated` que si le pointeur de zone gagnante change. Une mutation d'échelle, de centre ou de direction sur la même zone met donc le cache à jour sans réveiller le futur applicateur. J4 devra notifier sur changement du contrat effectif, ou conserver un poll natif léger sans overlap ; la comparaison explicite du contrat est préférable à un broadcast global systématique. L'absence de zone applique une échelle `0` sans setter une direction nulle, et ne doit jamais utiliser `GetGravityDirection(true)` comme preuve de gravité active puisque Ninja peut alors retourner une ancienne direction géométrique malgré l'échelle zéro.

L'échelle a déjà un second writer natif : `QModule_LegacyFacade::ApplyGravityCushion` écrit directement `UCharacterMovementComponent::GravityScale` à chaque mutation de rack. J4 doit remplacer cette collision par un compositeur unique, avec au minimum `échelle de zone signée × base personnage × modificateur CoussinGravitationnel`; laisser l'applicateur et QModule s'écraser mutuellement selon leur ordre d'exécution est interdit. Pour le CMC, la direction fournie reste non signée : Ninja inverse déjà la direction finale lorsque l'échelle calculée est négative, donc utiliser `GetGravityDirectionAttracting` en plus d'une échelle négative produirait une double inversion.

La composition doit respecter le rôle réseau déjà utilisé par QModule : écriture uniquement sur l'autorité ou le pawn localement contrôlé. Le serveur laisse ensuite `UNinjaCharacterMovementComponent::ReplicateGravityToClients` multicast la direction et l'échelle aux simulated proxies. Un applicateur actif sur ces proxies recalculerait au contraire avec un coussin local par défaut et écraserait le composite reçu. L'échelle de base est capturée une seule fois depuis l'instance CMC réellement authored ; aucune relecture de l'archetype et aucun remplacement silencieux par `1.0` ne sont permis. Les échelles `double` de zone/forced et leur produit doivent être finis et représentables en `float` avant d'atteindre le CMC, sinon la sortie est neutralisée et l'erreur est explicite.

Ce compositeur reste dans `GravityScape`, déjà dépendant de `NinjaCharacter`. `QModule` lui pousse le multiplicateur du coussin et, si la preuve live l'exige, `QPlatform` lui pousse son état ; `GravityScape` ne dépend d'aucun de ces deux plugins afin d'éviter le cycle existant `QModule -> QAI -> GravityScape`. La gravité QPlatform ne peut pas être réduite à un scalaire : elle produit un vecteur de force interpolé et un lerp chaque frame. Son contrat actuel a en plus deux trous de notification à traiter si ces chemins sont actifs : le detach remet l'état/la force à zéro sans émettre un dernier `QP_OnUpdateGravity`, et `QPlatform_SetGravity` modifie la force sans broadcast. Le mode `State_NormalGravity` contient également un test impossible (`HitAlpha` borné à `[0,1]` puis comparé à `> 2`) ; la recherche live doit établir son usage avant de l'inclure et de supprimer le défaut à sa racine.

La frontière native retenue pour l'implémentation est un `UQGravityCharacterComponent` dans `GravityScape`, déclaré avec les classes existantes plutôt que dans un nouveau plugin ou une nouvelle couche. Il se lie au `UQGravityAreaComponent` déjà authored, capture une seule fois l'échelle de base du CMC Ninja, applique le mode point/fixe seulement lors d'un changement de contrat et possède l'unique écriture `GravityScale = base × zone signée × coussin`. `QModule` ne réécrira plus le CMC : il poussera uniquement le multiplicateur du coussin vers ce propriétaire typé. Ce composant sera ajouté aux deux bases ALS après confirmation live des graphes, puis tous les nœuds direction/échelle qu'il remplace seront supprimés.

L'implémentation source de cette frontière J4 est maintenant écrite, mais pas encore déclarée validée. `UQGravityCharacterComponent` ne ticke pas : il se lie au dispatcher natif, force une consommation initiale du cache pour fermer la course d'ordre `BeginPlay`, supprime les contrats identiques, se délie lors de `EndPlay` comme lors de `DestroyComponent`, et n'écrit que sur l'autorité ou le pawn local. L'absence de zone neutralise uniquement l'échelle, tandis que les contrats point/fixe valides fournissent à Ninja le centre ou la direction normalisée non signée. La composition `float base × double zone × float coussin` est extraite dans une fonction pure qui refuse toute entrée non finie ou tout résultat non représentable en `float` ; aucune valeur de remplacement n'est inventée.

Le compositeur natif expose l'entrée typée du coussin, mais le writer QModule de production reste volontairement sur son chemin existant dans ce checkpoint. Le basculer avant d'avoir authored l'applicateur sur les deux bases aurait supprimé l'effet du module lorsque le composant manque ; l'ajout du composant, le switch du writer et le retrait du Blueprint devront donc rester une seule verticale à la reprise. Deux QATS adjacents couvrent le calcul pur puis le contrat personnage réel : point positif, location forcée, coussin, non-compounding de la base, fixe positif/négatif, inversion unique laissée à Ninja, gravité zéro hors zone, suppression des écritures identiques et déliaison à la destruction. La revue statique a renforcé le teardown sans inventer une direction par défaut : Ninja expose une capture/restauration exacte de son mode, de ses deux vecteurs et de son éventuel actor source ; détruire l'applicateur sous contrat non nul restaure cette direction et l'échelle authored. L'applicateur se désactive, retire ses deux delegates et oublie ses sources avant cette restauration, car `GravityDirectionChanged` invoque synchroniquement les listeners natifs et Blueprint. Le même risque existe pendant l'application : un listener peut détruire l'applicateur depuis `SetPointGravityDirection` ou `SetFixedGravityDirection`. Après chaque setter, le writer revérifie donc son activation, ses deux sources faibles et sa révision avant d'écrire l'échelle ou d'enregistrer le contrat. Le QATS détruit l'applicateur depuis ce callback, force un refresh pendant la restauration réentrante et prouve que ni l'échelle ni la révision retirées ne sont réappliquées. `UQGravityAreaComponent` publie aussi un événement natif de fin de source afin que sa destruction indépendante déclenche la même restauration au lieu de laisser le dernier composite actif. La revue a enfin rendu vérifiable le flag Ninja de réplication et l'applicateur refuse explicitement `bDisableGravityReplication=true`, car les simulated proxies ne recalculent volontairement pas le composite. Le scratch world QATS ne démarre plus les world subsystems sans pouvoir les arrêter : un `StartPlay` explicite effectue désormais le couple `UWorld::BeginPlay` + `NotifyBeginPlay` seulement après assemblage des dépendances. Le build courant et les dix QATS gravité incluent ce guard de réentrée et la compatibilité des composants historiques. Le checkpoint s'arrête avant l'ajout authored du composant aux deux bases, le switch du writer QModule, le retrait des nœuds Blueprint remplacés et les gates PIE de J4.

Trois options Ninja doivent aussi être prouvées sur les deux bases avant de rendre l'application purement événementielle : `bAlignGravityToBase` réécrit le mode/direction après chaque mouvement, `bRevertToDefaultGravity` peut remettre la direction monde vers le bas lors d'un changement de physics volume, et `bDisableGravityReplication` empêcherait le serveur de transmettre direction et échelle aux simulated proxies. Leurs defaults C++ sont faux, mais une valeur Blueprint sérialisée vraie constituerait encore un producteur concurrent ou une rupture réseau. J4 doit supprimer les deux writers s'ils sont authored, maintenir la réplication Ninja active et retirer les nœuds remplacés ; il ne masquera pas la concurrence par une réapplication périodique.

### J5 — Rotation et ordre QAI — à faire

- migrer l'alignement surface ;
- imposer la dépendance de tick par rapport au propriétaire rotation QAI ;
- supprimer les branches `UpdateNinjaGravityDirection` remplacées dans ALS et AILean ;
- valider un joueur, un cyborg AILean en déplacement et un cyborg en combat.

Le code QAI actuel documente et applique plusieurs écritures tardives destinées explicitement à gagner contre `UpdateNinjaGravityDirection` : dépendance de tick après l'acteur, réapplication de la rotation répliquée sur proxy client et base de rotation de combat isolée des écritures ALS. J5 doit les réévaluer une par une après suppression du graphe, conserver uniquement le propriétaire yaw/réplication réellement nécessaire et supprimer toute compensation devenue morte ; les empiler au-dessus du nouvel applicateur recréerait deux autorités de rotation.

La revue source sépare déjà ces écritures : la réapplication client de `GetReplicatedMovement().Rotation` est explicitement une compensation du graphe ALS et doit disparaître avec lui ; le `CombatFacing` mémorisé n'existe que pour empêcher ce même graphe de polluer la source d'interpolation et devra être simplifié si l'acteur devient une source stable. En revanche, le calcul `MakeFromZX(LocalUp, ForwardInGravPlane)` possède réellement le heading combat autour de l'up gravitationnel et ne doit pas être confondu avec l'alignement surface. La prerequisite après l'actor tick reste à justifier séparément par l'échantillonnage AnimBP et la vélocité du mouvement, pas par l'ancien combat d'écriture ALS.

### J6 — Essential values et mouvement ALS — à faire

- migrer `SetEssentialValues`, cache vitesse/accélération/input et états mouvement mesurés ;
- couper tout travail visuel sur dedicated server ;
- continuer verticalement, une responsabilité et un cleanup Blueprint à la fois.

Références natives vérifiées avant implémentation :

- [`ALS-Refactored` 4.17](https://github.com/Sixze/ALS-Refactored) cible directement UE 5.7 et sépare explicitement dans le tick mouvement-base, input, locomotion early/main/late, vue, gait puis rotation ; son changelog 4.17 corrige aussi le calcul de vitesse angulaire de movement base pour la custom gravity ;
- [`ALS-Community`](https://github.com/PanicPetal/ALS-Community) fournit la traduction C++ directe de `SetEssentialValues` et son ordre avant gait/rotation ; elle sert de table de correspondance pour les sorties historiques, pas de code à copier aveuglément ;
- QANGA ne sera pas reparenté sur l'une de ces classes : `NinjaCharacter`, la gravité arbitraire, QPlatform et le propriétaire rotation QAI sont des contrats de production distincts. Les références servent à reprendre les frontières et l'ordre éprouvés tout en gardant un propriétaire QANGA unique.

La revue source UE 5.7 précise la frontière portable. L'ordre natif ALS mouvement-base puis locomotion/vue/gait/rotation, ainsi que la prerequisite mesh après acteur, sont réutilisables. Ses calculs de rotation acteur, caméra et plusieurs sorties animation restent cependant en `Yaw`, `UpVector` ou `Velocity.Z` monde ; ils ne peuvent pas remplacer les APIs gravity-space de Ninja. De même, la compensation de rotation répliquée d'ALS suppose qu'ALS possède entièrement l'acteur, alors que Ninja resynchronise déjà l'axe Z depuis `ReplicatedMovement`. J5 conservera donc l'alignement capsule de Ninja et supprimera les compensations QAI devenues mortes sans importer celles d'un autre ALS. La caméra QANGA pourra reprendre uniquement l'ordre movement-base → rotation → pivot/lag → offsets → trace → cache final. Enfin, aucune des deux références ALS ne possède un système swimming à substituer : `UNinjaCharacterMovementComponent::PhysSwimming` / `StartSwimming` restent le propriétaire natif à comparer au Blueprint `SwimmingDetection`.

### J7 — Validation finale et cleanup — à faire

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
