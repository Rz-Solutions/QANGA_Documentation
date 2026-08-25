# Première migration Blueprint vers C++ — acquisition de cible d'arme

- **État :** TERMINÉ - propriétaire natif unique installé, ancien système supprimé, build/tests/runtime réseau validés
- **Module propriétaire :** `QWeapon`
- **Assets principaux :** `VehicleCombatComponent`, `HomingLocker`, `SAT_FilterCombatComponents`, `GS_CombatManager`
- **Début :** 2026-08-25

---
## 0. Point de contrôle actuel

Le registre, le provider et le consommateur véhicule sont maintenant installés :

- `UQWeaponTargetRegistrySubsystem` est l'unique registre world-scoped. Il indexe des références faibles dans un `TOctree2`, prend un snapshot au plus toutes les `0.5 s` et conserve la frontière stricte `DistanceSquared < RadiusSquared` ;
- `IQWeaponTargetProvider` expose actor, aim point, alive, autorisation métier, cible courante et commit. `CombatComponent` implémente ce contrat typé et s'enregistre directement à `BeginPlay` / se désenregistre à `EndPlay` ;
- `UQWeaponTargetingComponent` possède acquisition, rétention exacte de `LastSeen + 0.5 s + 2.0 s`, cône strict `Dot > 0.985`, ordre déterministe, cycle, validation RPC serveur et liste faible `TargetedByActors` ;
- `SAT_FilterCombatComponents`, sa référence EasyCook et tous ses appels dans les deux consommateurs ont été supprimés. `DA_EasyCookSeed_QANGA` reste volontairement vide (`Entries=0`, `AlwaysCookDirectories=0`) sans nouveau scan, conformément à la décision de Rz ;
- `GS_CombatManager.CombatComponents`, `AddCombatComponent`, `RemoveCombatComponent` et `OnAsyncFilterReply` ont été supprimés après audit complet de ses deux referencers live. `EnabledPvP` et ses consommateurs restent hors de ce remplacement ;
- `VehicleCombatComponent` est reparenté sur la base native. Son scan, sa map `WeaponTargets`, `FirstIndexInternal`, `RemoveOld` et ses deux macros de cône sont supprimés. Son EventGraph passe de `110` à `78` nœuds et l'asset de `217` à `141` nœuds après suppression des derniers événements QAI/RPC morts ;
- ses façades publiques encore consommées appellent désormais le propriétaire natif : `SetCurrentTarget`, `SetNextTarget`, `GetOnFrontActor`, `AddTargetedByOthers` et `OnDestroyedTargetedByOthers`. Les anciennes entrées `SV_SetCurrentTarget` et `AuthorityClientAddActorTargeting` ont été supprimées après migration de leurs appels ;
- le lifecycle targeted-by est natif, y compris déduplication, références faibles et nettoyage `OnDestroyed`. `W_VehicleTargetedFrame` lit `GetTargetedByActors`; le dispatcher `OnTargetedByOthers` est conservé uniquement comme contrat UI déjà bindé, alimenté par les événements natifs ;
- `HomingLocker` est reparenté sur la même base. Son activation reste bornée par homing activé, pawn joueur local, owner item valide et item actif ; chaque sortie désactive explicitement l'acquisition native ;
- sa requête registre Blueprint, ses filtres, `WeaponTargets`, `FirstIndexInternal`, `RemoveOld`, ses deux macros de cône et son RPC de cible non validé sont supprimés. Son EventGraph passe de `108` à `71` nœuds et l'asset de `242` à `121` nœuds ;
- `SetCurrentTarget`, `GetOnFrontActor` et `NextTarget` sont désormais des façades natives ; `SV_SetCurrentTarget` a été supprimé après une recherche fraîche et complète sur les referencers. Les trois consommateurs `IS_NashV2_Rocket_Launcher`, sa variante Noel et `VWP_RocketLauncher` recompilent à `0` erreur / `0` warning ;
- `VehicleCombatComponent`, `W_VehicleTargetedFrame` et `TurretBase` compilent à `0` erreur / `0` warning après cette bascule ;
- le bridge QAI ciblé est maintenant typé dans le code source : le registre découvre `UQWeaponCoordinatorComponent` par type, le processor appelle directement `SetWeaponTarget`, `FirePrimaryWeapons`, `SetWeaponsEnabled` et `FireWeaponsByClass`, et `SetupMissileTargeting` ne cherche plus des composants/fonctions par fragments de nom ;
- le lifecycle de warning QAI est rétabli autour d'un acteur source correct : le coordinator retire son owner de l'ancienne cible, l'ajoute à la nouvelle cible native, et les missiles s'enregistrent directement auprès de `UQWeaponTargetingComponent`. Le composant natif est seul propriétaire du cleanup `OnDestroyed`, donc `MissilesWithVCBind` et le second chemin de suppression réfléchi sont supprimés ;
- le dernier appel contenu à `AuthorityClientAddActorTargeting`, dans `VWSlot_Rocket`, est remplacé par un appel direct à `NotifyTargetingCurrentTarget` avec le missile spawné. Une recherche fraîche sur les `45` Blueprints referencers véhicule confirme ensuite zéro appel restant ; l'événement temporaire et les deux `SV_SetCurrentTarget` sont supprimés ;
- le build disque complet après changement de layout UHT passe, l'Editor a été fermé par `QUIT_EDITOR` puis relancé avec récupération de packages auto-refusée. Les neuf Blueprints touchés ou directement consommateurs recompilent à `0` erreur / `0` warning après reload ;
- la validation serveur distingue maintenant l'acquisition locale de l'autorisation autoritaire : une copie serveur distante ne pollue pas le registre, mais peut valider une demande client depuis son propre état. `HomingLocker` exige homing actif, pawn/item valides et item toujours équipé ; `VehicleCombatComponent` exige détection et homing actifs, arme Rocket présente, composant véhicule valide et état différent de `Broken`/`Destroyed` ;
- le build disque `QangaEditor Win64 Development` passe avec les nouveaux symboles réfléchis. Les sept tests `QWeapon.Targeting.*` passent : décision, état de requête, cleanup targeted-by, validation autoritaire sur le vrai `CombatComponent_C`, index spatial, lifecycle registre et bridge provider Blueprint ;
- trois défauts/capacités manquantes de RzDirectMCP révélés par cette migration ont été corrigés à la source : les defaults de pins objet utilisent maintenant `DefaultObject`, la suppression de graphe accepte aussi les macros et `add_function_override` crée une vraie implémentation de fonction parent avec sa signature moteur. Le dernier changement passe UHT, compilation et liaison disque.
- le host de compatibilité UE 5.8 de RzDirectMCP ne junctionne plus le plugin entier : seuls `Source` et `Config` sont partagés, tandis que `Intermediate` et `Binaries` restent propres à chaque moteur. Les artefacts UE 5.8 qui polluaient QANGA ont été supprimés puis régénérés par le build UE 5.7 réussi.

Durcissement validé par build disque, tests natifs et preuves runtime/network :

- l'acquisition conserve maintenant les candidats métier dans le rayon indépendamment du cône, le cône reste une responsabilité de sélection ; `NativeUpdateTargeting` ne choisit plus automatiquement la première cible et le cycle d'une cible unique la vide comme le graphe remplacé ;
- l'acteur de référence est le `TargetingPawn` configuré, y compris pour l'origine, self/attached et le provider demandeur, ce qui rétablit le contrat de `HomingLocker` lorsque son owner est l'item ;
- un client ne mute plus `ActorTarget` avant acceptation : il envoie la demande, le serveur passe par un funnel autoritaire unique, puis confirme son état au client même lors d'un refus ;
- le lifecycle `TargetedByActors` est muté sur l'autorité et distribué par RPC multicast, avec cleanup local à la destruction ; la déduplication et le cleanup ont été validés en Listen Server et Dedicated Server ;
- `QAI_FunctionLibrary` utilise désormais `IQWeaponTargetProvider` pour lire et committer la cible, y compris son dump debug. Les noms internes obsolètes `*OnCombatComponent` ont été supprimés ;
- le dernier lookup mort de `AuthorityClientAddActorTargeting` dans `QSpaceshipPoliceProvider` est supprimé : `UQWeaponCoordinatorComponent::SetWeaponTarget` est déjà le propriétaire typé de cette notification ;
- la règle de tir direct de `QWeaponBulletSubsystem` appelle le même provider typé au lieu de rechercher `CombatComponent.CheckTargetIsAllowedCombat` par nom et `ProcessEvent` ;
- le test `QWeapon.Targeting.AuthorityValidation` couvre le vrai provider Blueprint, les candidats hors cône conservés à l'acquisition, l'absence d'auto-commit, le commit valide et les refus self/attached/tag/alive/rayon/cône sans mutation. Il passe avec les six tests existants après relance propre de l'Editor.
- la validation Standalone sur un véhicule réel a confirmé le filtre métier, l'absence d'auto-commit, le commit/refus, la rétention hors cône, le lifecycle targeted-by et le clear après destruction ;
- la validation Listen Server sur `/Game/Maps/LevelDev/L_Dev_Rz` a confirmé un serveur et un client, la réplication du provider et du composant natif, l'ownership RPC, l'acquisition/sélection, le commit client confirmé par le serveur, le refus autoritaire reconvergé vers null, la déduplication targeted-by et son cleanup après destruction ;
- la même validation Dedicated Server sur `L_Dev_Rz` a confirmé un serveur dédié et deux clients. La cible est observée sur chaque peer, la sélection reste limitée à l'autorité et au client propriétaire, et le multicast targeted-by converge sur les deux clients ;
- le fixture réseau utilisait le vrai `CombatComponent_C` comme `IQWeaponTargetProvider` et le vrai `UQWeaponTargetingComponent` comme propriétaire natif. Les pawns synthétiques ne remplaçaient aucune règle gameplay : ils isolaient uniquement ownership, réplication et RPC de la logique de véhicule de la map ;
- le test multi-PIE a été exécuté dans un Editor isolé avec audio désactivé, après attribution d'un crash antérieur à `UResonanceAudioBlueprintFunctionLibrary::SetGlobalReverbPreset`, avant tout appel du fixture. Aucun guard ni contournement n'a été ajouté au gameplay migré ;
- RzDirectMCP accepte maintenant les paramètres réseau transients de `play_in_editor` (`play_net_mode`, nombre de clients, process unique, serveur séparé), expose chaque monde par `pie_instance` et protège chaque appel direct par l'identité exacte de l'Editor. Ces capacités ont servi aux preuves Listen/Dedicated sans ouvrir, modifier ou sauvegarder la map source ;
- le probe PowerShell créé sous `Saved/RzDirectMCP` pour ces scénarios a été supprimé immédiatement après validation. Aucun harness temporaire ne reste dans le projet.

Le gate fonctionnel est clos. La mesure avant/après reste limitée par l'absence de trace J0 représentative antérieure : aucune valeur de gain en millisecondes n'est revendiquée. La suppression du coût visé est néanmoins vérifiable structurellement : le SAT, le scan global par targeter, les maps/rétentions Blueprint dupliquées et les bridges réfléchis ciblés n'existent plus dans le chemin runtime.

---
## 1. Décision

### 1.1 Périmètre

Le premier chantier est la **chaîne native partagée d'acquisition de cible d'arme**. Il couvre ensemble :

- le registre actuellement porté par `GS_CombatManager.CombatComponents` ;
- le producteur `SAT_FilterCombatComponents` ;
- les requêtes, la rétention et la sélection communes de `VehicleCombatComponent` et `HomingLocker` ;
- la validation serveur de la cible demandée ;
- les appels QAI vers le composant de combat véhicule actuellement effectués par réflexion.

Ce chantier passe avant Inventory et le noyau Combat parce que sa frontière est plus courte, son coût est fréquent et multiplié par le nombre d'acteurs, son propriétaire natif existe déjà, et son résultat peut être mesuré puis validé sans réécrire immédiatement les contrats de persistance, d'équipement, de dégâts et de mort.

Migrer uniquement `VehicleCombatComponent` serait une mauvaise frontière : `HomingLocker` conserverait le même scan global, `SAT_FilterCombatComponents` resterait vivant et le coût producteur ne pourrait pas être supprimé.

### 1.2 Cause racine vérifiée

`SAT_FilterCombatComponents` hérite de `ShortAsyncTask`, mais `UShortAsyncTask::startThread` renvoie `execute()` sur `ENamedThreads::GameThread` puis bloque le worker sur un événement. Le scan Blueprint reste donc sur le Game Thread et ajoute en plus l'allocation de tâche, la synchronisation et les traversées Blueprint/C++.

À chaque requête, la tâche :

1. convertit le `Set<CombatComponent>` global en tableau ;
2. parcourt tous les composants ;
3. lit l'owner et sa position ;
4. applique seulement le test de rayon ;
5. laisse chaque consommateur refaire les filtres self/attached, tag, validité et alive.

Le défaut n'est donc pas seulement « trop de Blueprint ». Le modèle répète un scan global et du filtrage dupliqué pour chaque targeter actif.

Un second défaut doit être corrigé dans le même chantier : les RPC fiables `SV_SetCurrentTarget` acceptent actuellement la cible envoyée par le client et écrivent directement `CombatComponent.ActorTarget`, sans recalcul serveur du rayon, du cône, de l'état alive ni de l'autorisation de ciblage.

### 1.3 Propriétaire natif unique

Le propriétaire permanent sera `QWeapon`, déjà responsable du hitscan natif et déjà dépendance publique de QAI. Aucun nouveau plugin `QCombat` ne sera créé pour ce seul chantier.

Types prévus :

- `UQWeaponTargetProvider` / `IQWeaponTargetProvider` : contrat typé implémenté d'abord par `CombatComponent_C`, puis conservé lorsque le noyau Combat migrera ;
- `UQWeaponTargetRegistrySubsystem` : `UWorldSubsystem` unique par monde, responsable de l'enregistrement, du snapshot spatial et des requêtes ;
- `UQWeaponTargetingComponent` : base native commune de `VehicleCombatComponent_C` et `HomingLocker_C`, responsable de la requête, de la rétention, du score, du cycle et de la validation RPC.

Le subsystem remplacera le registre de ciblage contenu dans `GS_CombatManager`. Il n'existera jamais deux registres actifs en production. `GS_CombatManager` ne conservera que ses responsabilités encore prouvées, par exemple `EnabledPvP`; si aucune autre responsabilité n'a de consommateur live, l'asset et `Lib_Combat.GetCombatManager` seront supprimés après audit de leurs referencers.

### 1.4 Contrats natifs

Le provider expose uniquement les données nécessaires au ciblage :

- acteur cible et position de visée ;
- état actif/alive ;
- règle `CanBeTargetedBy` ;
- point d'entrée typé vers le funnel répliqué `ActorTarget` pendant la transition.

Le registre expose :

- `RegisterTargetProvider` et `UnregisterTargetProvider` ;
- `QueryTargets(Origin, Radius, Requester, OutTargets)` ;
- des compteurs Insights, sans log par candidat.

Les entrées utilisent des références faibles. Toute lecture de `UObject`, d'actor, de tag, de caméra ou de composant reste sur le Game Thread. La première version ne recrée pas une pseudo-async task : elle supprime ce coût. Une exécution hors-thread ne sera envisagée que si une trace ultérieure la justifie et seulement sur un snapshot immuable.

Le broad phase utilisera l'API moteur `TOctree2` sur un snapshot reconstruit au plus une fois par échéance globale et uniquement lorsqu'une requête existe. La requête octree est suivie du test exact `DistanceSquared < RadiusSquared`, afin de conserver la frontière stricte actuelle. La phase J0 doit mesurer l'étendue et la densité réelles avant de figer les bounds et la fréquence ; si ce modèle n'est pas adapté aux données de production, l'architecture sera réévaluée avant implémentation, sans ajouter un second chemin runtime.

### 1.5 Sélection et état de cible

`UQWeaponTargetingComponent` remplace les deux maps `WeaponTargets` et leurs macros dupliquées. Il utilise :

- une échéance `NextTargetQueryTimeSeconds`, sans timer/delegate ajouté à une boucle existante ;
- une expiration explicite `LastSeenTimeSeconds + QueryIntervalSeconds + TargetRetentionGraceSeconds`, avec `TargetRetentionGraceSeconds = 2.0` ; les graphes live utilisent donc actuellement `2.5 s` avec leur tick à `0.5 s` ;
- les filtres self, attached actors, `AllowHolming`, validité provider, alive et autorisation métier ;
- le seuil strict actuel `Dot > 0.985` pour le cône ;
- un classement déterministe : meilleur dot, puis plus petite distance, puis identifiant stable de l'entrée enregistrée ;
- une liste ordonnée stable pour `SetNextTarget`, à la place de l'ordre non garanti d'une `TMap`.

Le joueur utilise la vue de son controller telle que définie par le contrat live ; l'IA utilise le forward de l'owner. Ces origines doivent être conservées séparément, pas fusionnées dans une approximation commune.

Le nouveau RPC serveur ne fait jamais confiance au résultat client. Avant d'écrire `CombatComponent.ActorTarget`, le serveur retrouve l'entrée dans son propre registre et recalcule l'éligibilité complète depuis son état autoritaire : ownership, système activé, self/attached, rayon, cône, tag, alive et `CanBeTargetedBy`. Une cible refusée ne modifie pas l'état courant.

### 1.6 Couture Blueprint temporaire

Les assets et noms appelés par le contenu existant restent disponibles pendant la migration :

- véhicule : `SetEnable`, `SetCurrentTarget`, `GetCurrentTargetSceneRoot`, `GetOnFrontActor`, `SetNextTarget`, `AddTargetedByOthers`, `OnDestroyedTargetedByOthers`, `OnTargetsUpdate`, `OnTargetedByOthers`. L'ancienne façade `AuthorityClientAddActorTargeting` a été supprimée après migration de son dernier appel ;
- homing : `SetEnableHoming`, `GetHomingEnabled`, `NextTarget`, `GetCombatComponent`, `OnHoming`.

Les deux Blueprints deviennent des façades minces reparentées sur la base native. Ils conservent temporairement :

- les règles d'activation liées à l'état du véhicule ou à l'item actif ;
- trackers, UI et feedback de lock/homing ;
- FX et appels de présentation ;
- l'adaptateur vers `CombatComponent.ActorTarget` et son `OnRep` jusqu'à la migration du noyau Combat.

Ils ne conservent pas une branche de scan désactivée « au cas où ». Chaque tranche bascule un propriétaire, vérifie la parité, puis supprime immédiatement le graphe et les données remplacés.

## 2. Plan d'exécution

### J0 — Baseline et contrats — contrats terminés, baseline historique indisponible

- Capturer une trace représentative avec `1`, `8` et `32` targeters actifs, puis `100` et `500` Combat Components.
- Mesurer Game Thread, Blueprint VM, nombre de tâches SAT, candidats parcourus, fréquence réelle, temps de sélection et latence acquire/drop.
- Cartographier les referencers live des fonctions publiques, des delegates, de `ActorTarget`, de `EnabledPvP` et de `Lib_Combat.GetCombatManager`.
- Figer les bounds du snapshot, l'échéance globale et le contrat exact des origines joueur/IA.

**Résultat :** les contrats et propriétaires ont été audités avant la bascule. Aucune trace J0 gameplay représentative n'existait ; cette limite reste documentée et interdit toute revendication chiffrée, mais ne justifie pas de restaurer le SAT supprimé.

### J1 — Math pure et tests

- Implémenter les records sans `UObject`, le test de rayon strict, le cône, le score déterministe, la rétention et le cycle.
- Ajouter des Automation tests adjacents dans `QWeapon` : frontières exactes, coordonnées négatives et très grandes, égalités de score, ordre stable, expiration et références invalidées.
- Tester la requête octree contre une référence brute uniquement dans les tests, sur des distributions randomisées. Cette référence de test n'est jamais compilée comme chemin runtime de secours.

**Gate :** tests verts et choix spatial validé par les mesures J0.

### J2 — Registre natif

- Ajouter le provider typé et le `UQWeaponTargetRegistrySubsystem`.
- Brancher `CombatComponent` sur register à `BeginPlay` et unregister à `EndPlay`.
- Construire le snapshot à la demande, au plus une fois par deadline globale.
- Instrumenter avec Unreal Insights : registered, queried, candidates, accepted, rejected, stale et durée de requête.

**Gate :** parité de registre en Standalone, Listen Server, Dedicated Server et client, sans second registre actif.

### J3 — Consommateur véhicule

- Reparenter `VehicleCombatComponent` sur `UQWeaponTargetingComponent`.
- Migrer requête, filtres, map, rétention, cône, sélection et cycle.
- Conserver uniquement les événements BP de trackers/UI et les contrats publics existants.
- Remplacer dans QAI la recherche par class name et les `FindFunctionByName` par l'API native typée.

**Gate :** véhicule joueur, véhicule IA et tourelle conservent acquire/drop/cycle, notifications missile et réplication de cible.

### J4 — Consommateur HomingLocker et autorité

- Reparenter `HomingLocker` sur la même base native.
- Migrer son gating item actif, son enable homing et sa sélection sans dupliquer le registre.
- Remplacer les deux `SV_SetCurrentTarget` par la validation serveur native commune.
- Vérifier qu'une requête forgée hors rayon, hors cône, morte ou non targetable est refusée.

**Gate :** lance-roquettes/homing validé sur client, listen et dedicated, avec une seule mutation autoritaire de `ActorTarget`.

### J5 — Suppression de l'ancien système

Après relecture live des referencers :

- supprimer `SAT_FilterCombatComponents` et sa référence de cook ;
- supprimer dans les deux Blueprints `Create Async Task`, `runStatefulAsyncTask`, `OnReplyTask`, `SAT_FilterTask`, les maps et macros migrées ;
- supprimer `GS_CombatManager.CombatComponents`, `AddCombatComponent`, `RemoveCombatComponent` et `OnAsyncFilterReply` devenus morts ;
- supprimer les bridges QAI par réflexion et leurs branches de recherche par nom de classe ;
- retirer toute dépendance à `AsyncBlueprintsExtension` qui n'a plus de consommateur réel dans ce périmètre.

**Gate :** zéro instance SAT, zéro scan global par targeter et aucun producteur Blueprint désactivé mais encore maintenu.

### J6 — Validation finale — terminé

- Les sept tests `QWeapon.Targeting.*` couvrent math, frontières, cycle, état de requête, provider Blueprint, registre, autorité et lifecycle targeted-by.
- Le runtime Standalone couvre le provider/composant véhicule réel, rétention, refus, destruction et cleanup.
- Listen Server puis Dedicated Server + deux clients sur `L_Dev_Rz` couvrent ownership, RPC, confirmation/rejet autoritaire et multicast.
- Le build disque `QangaEditor Win64 Development` et les compiles des Blueprints consommateurs sont verts. Les extensions multi-PIE de RzDirectMCP ont ensuite été compilées et exercées par Live Coding dans l'Editor isolé.
- Aucune comparaison de trace avant/après n'est publiée faute de capture J0 représentative antérieure ; recréer le producteur Blueprint supprimé uniquement pour cette mesure serait réintroduire du code mort.

## 3. Critères de sortie

Le chantier est terminé uniquement si :

- `SAT_FilterCombatComponents` n'a plus aucun referencer et l'asset est supprimé ;
- chaque Combat Component est enregistré une seule fois et désenregistré sans entrée faible résiduelle ;
- un seul snapshot spatial est reconstruit par deadline et par monde, quel que soit le nombre de targeters ;
- aucune requête ne scanne le set global par targeter ;
- la latence acquire/drop n'est pas supérieure au comportement actuel à `0.5 s` ;
- l'ordre de sélection/cycle est déterministe ;
- les RPC invalides sont rejetés sans mutation de `ActorTarget` ;
- les callbacks UI, trackers, homing, QAI et tourelles conservent leur comportement ;
- l'ancien SAT et le scan global `O(targeters × combat actors)` sont absents du runtime ; une future trace production peut quantifier le gain sans réintroduire le chemin supprimé ;
- aucun fallback, branche legacy ou code mort du système remplacé ne subsiste.
