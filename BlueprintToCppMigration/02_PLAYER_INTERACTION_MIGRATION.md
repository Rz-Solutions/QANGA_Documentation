# Deuxième migration Blueprint vers C++ — interaction joueur

- **État :** MIGRATION NATIVE TERMINÉE — hotspot basculé, code remplacé supprimé, réseau et consommateurs réels hold/sit validés ; matrice métier étendue et gate cook restent des gates de release
- **Module propriétaire :** `QSystem` (aucun nouveau plugin)
- **Asset principal :** `/Game/Systems/Interact/PlayerControllerInteractComponent`
- **Carte de validation :** `/Game/Maps/LevelDev/L_Dev_Rz`
- **Début :** 2026-08-26

---

## 0. Décision

`UQPlayerControllerInteractComponent`, dans le module runtime existant `QSystem`, devient l'unique propriétaire de la détection, du classement, de l'état de focus et du funnel RPC serveur. Le Blueprint `PlayerControllerInteractComponent` est reparenté sur cette classe et reste une façade transitoire pour `InteractClient`, le highlight, le feedback, le sit et la progression de quête.

La migration ne crée pas un second composant. `QangaPlayerController` conserve son unique composant `PlayerInteractComponent`; le Blueprint et les consommateurs existants continuent donc de résoudre la même instance pendant la bascule.

## 1. Preuve du coût et contrat actuel

L'inspection live, sans rescan du projet, établit :

- parent actuel : `UActorComponent` ;
- `423` nœuds : EventGraph `169`, `CheckInteractionRz` `62`, `InteractInput` `64`, tri `25`, sit `34`, highlight `16`, changement de cible `17` et macro d'occlusion `12` ;
- default `DistanceDetection=175 cm` ;
- `ReceiveTick` exécute la détection à chaque frame sur chaque instance qui ticke, alors que seul le controller local consomme le résultat ;
- trace primaire complexe sur `TraceTypeQuery1`, de `CameraLocation + CameraForward * 10 cm` à `CameraLocation + CameraForward * (DistanceDetection + Distance(CameraLocation, PawnLocation))` ;
- si cette trace ne fournit pas d'interaction et que le pawn n'est pas un véhicule pilotable, une deuxième requête est lancée :
  - vue `NewEnumerator0` : multi-sphere de rayon `DistanceDetection`, de la position du pawn à cette position plus `(1,1,1)` ;
  - vue `NewEnumerator1` : multi-box orientée caméra, centrée à `CameraLocation + CameraForward * DistanceDetection / 2`, demi-taille `(DistanceDetection / 2, 50, 50)` et sweep de `(1,1,1)` ;
- chaque résultat du sweep est classé par dot caméra décroissant, puis vérifié par une line trace non complexe depuis la position du pawn vers l'impact ;
- le tri Blueprint passe par une `TMap<double, FHitResult>` : deux résultats de dot identique partagent une clé et l'un peut écraser l'autre ;
- `UpdateIgnoreActors` reconstruit toutes les secondes les acteurs attachés au pawn, le pawn lui-même et la liste partagée `RTDA_CameraIgnore` ;
- le composant réplique et expose le RPC reliable serveur exact `SV_Interact(UObject* TargetReplicatedObject, bool Pressed, FName Id)`, mais le graphe serveur transmet actuellement l'objet directement à `InteractServer` sans revalidation générique ;
- quatre referencers directs seulement : `QangaPlayerController`, `Qanga_InputsComponent`, `Lib_Interact` et `StandDeTir_Start` ;
- le composant présent sur `StandDeTir_Start` appartient à un `AActor`, son cast owner vers `APlayerController` ne peut donc pas réussir, et les trois événements de cet asset sont vides. Cette instance est morte et doit être supprimée après vérification de ses instances ;
- QATS cherche le composant par fragment de nom et lit `ActiveInteractActor` et `DistanceDetection` par réflexion ; QTrain cherche `SV_Interact` par réflexion. Ces bridges deviennent typés là où les dépendances de modules le permettent.

Aucune trace gameplay J0 représentative n'est disponible. Les coûts structurels sont prouvés, mais aucun gain en millisecondes ne sera revendiqué sans capture avant/après comparable.

## 2. Frontière native

### 2.1 Exécution locale et échéance

Le composant natif :

- ne détecte que sur le `APlayerController` local ;
- désactive immédiatement sa détection sur dedicated server et sur les copies réseau non locales ;
- conserve un tick léger uniquement pour poller `NextDetectionTimeSeconds` ;
- lance au maximum une détection toutes les `1/30 s` par défaut, avec première détection immédiate ;
- neutralise le focus et le feedback si le pawn, la caméra ou un contrat obligatoire disparaît ;
- reconstruit la liste d'ignore au changement de pawn et au plus une fois par seconde, sans timer/delegate récursif.

### 2.2 Requête et classement

La géométrie ci-dessus reste identique, sauf correction explicite : la box utilise la même liste d'ignore que la line trace et la sphere. Les appels passent directement par `UWorld::LineTraceSingleByChannel` et `SweepMultiByChannel`, avec conversion moteur de `TraceTypeQuery1`.

Le classement natif conserve tous les hits et utilise un ordre déterministe :

1. meilleur dot entre le forward caméra et la direction caméra vers l'impact ;
2. plus petite distance carrée ;
3. identité stable actor/component/item ;
4. index d'entrée seulement comme dernier départage local.

Les candidats invalides et occlus sont éliminés avant tout appel métier. Les égalités ne suppriment jamais un résultat.

### 2.3 Contrat Blueprint restant

Les implémentations de `/Game/Systems/Interact/Interact_Interface` restent Blueprint. Leur invocation depuis C++ est confinée dans un bridge nommé qui valide une fois la signature réelle de `HasInteraction`, `InteractLost` et `InteractServer`, puis utilise `ProcessEvent`. Une signature absente ou incompatible est une panne explicite, jamais une branche alternative silencieuse.

Les coutures Blueprint restantes sont limitées au métier et à la présentation :

- présentation : l'événement existant `AfterCheck`, le highlight, le crosshair, le feedback, le sit et la progression de quête.

Elles ne refont ni trace, ni tri, ni occlusion, ni politique ALS/véhicule, ni mutation concurrente de la cible. Les appels existants à `BPI_Get_ViewMode`, `Lib_Vehicle.ActorIsDrivableVehicle` et `Lib_Vehicle.GetPawnIsInVehicle` sont confinés dans des bridges natifs qui valident noms, directions, arité et types avant d'activer le composant. La signature native de `AfterCheck` reprend exactement les cinq entrées du custom event actuel. Le custom event utilisateur ne peut toutefois pas devenir automatiquement l'override d'un événement natif : il sera supprimé pendant le reparent, puis remplacé par le vrai event hérité et son seul flux de présentation sera reconnecté explicitement.

### 2.4 État et autorité

Le composant natif possède `PlayerController`, `PawnCache`, `ActiveInteractActor`, l'objet exact `ActiveInteractionTarget` ayant accepté `HasInteraction`, `InteractCustomTarget`, `DistanceDetection`, le résultat sit, le hit item et les valeurs hold. Un seul funnel compare ancien/nouveau focus, appelle `InteractLost` une fois sur chaque ancien actor/component/custom target distinct, remplace l'état puis notifie la façade de présentation. `GetInteractionDispatchTarget()` fournit à l'input l'unique cible à transmettre au RPC : custom target valide, sinon objet exact ayant accepté l'interaction.

`SV_Interact` conserve exactement son nom et ses paramètres pour les callers existants. Sur le serveur il :

1. résout l'actor réel d'un actor, component ou sous-objet répliqué ;
2. vérifie monde, ownership du component RPC et objet valide ;
3. recalcule distance, visibilité, présence dans la requête native et résultat `HasInteraction` depuis l'état serveur ;
4. appelle `InteractServer` seulement si l'objet demandé correspond au résultat autorisé ;
5. refuse sans mutation et sans spam de log sinon.

Les validations métier plus strictes restent dans leurs propriétaires, par exemple QTrain ou QAI ; le composant générique ne les duplique pas.

## 3. Plan d'exécution

### J0 — Contrats et baseline — terminé

- inspecter parent, defaults, graphes, pins, géométrie, interface, RPC et referencers ;
- identifier les consommateurs C++ et les instances invalides ;
- documenter l'absence de trace J0 au lieu de fabriquer une mesure rétroactive.

### J1 — Math pure et tests — terminé

- ajouter score, ordre total, construction de géométrie et décision d'occlusion en fonctions pures ;
- tester égalités de dot, impacts superposés, valeurs non finies, ordre d'entrée différent, bornes exactes et conservation de tous les hits ;
- tester l'échéance `1/30 s` et la neutralisation des sorties.

Implémenté dans `QPlayerControllerInteractComponent.cpp` : géométrie exacte, score fini, ordre total sans perte d'égalité, décision d'occlusion et échéance sans rattrapage après hitch. Les tests `QSystem.Interaction.QueryGeometry`, `Ordering`, `Deadline`, `Occlusion`, `TargetAuthorization` et `LegacyContract` couvrent notamment l'autorisation actor/component/custom, la propagation du component dans la perte de focus et les signatures réelles des contrats soft.

Le build disque UE 5.7 est vert et les six tests `QSystem.Interaction.*` passent sans erreur ni warning.

**Gate :** tests `QSystem.Interaction.*` verts sans monde gameplay.

### J2 — Composant natif — terminé

- ajouter `UQPlayerControllerInteractComponent` dans `QSystem` ;
- implémenter local-only, échéance polled, cache pawn/caméra, ignore list, trace primaire, sweeps, tri et occlusion ;
- ajouter les scopes Unreal Insights et le bridge validé vers l'interface Blueprint ;
- exposer l'état et les hooks de présentation nécessaires, sans dupliquer les variables.

`UQPlayerControllerInteractComponent` est maintenant présent dans `QSystem`. Il possède la détection locale à `30 Hz`, le refresh d'ignore à `1 Hz`, les requêtes physiques, le tri déterministe, l'occlusion, le focus unique, la neutralisation des sorties, les bridges contrôlés vers `Interact_Interface`, l'interface ALS et `Lib_Vehicle`, ainsi que la revalidation du RPC `SV_Interact`. Le component et l'actor sont interrogés indépendamment ; un component qui implémente seul l'interface reste donc une cible complète et reçoit aussi `InteractLost`. Le chemin sit conserve le contrat Blueprint exact : actor en cible principale, hit component en custom target, hold actif et durée nulle. Le tick est coupé sur dedicated server et copies non locales. Le serveur recalcule sa propre requête avant chaque press, autorise exactement l'actor, le component ayant accepté `HasInteraction` ou le custom target retourné, conserve une autorisation valide face à une release forgée et refuse une nouvelle press si la release précédente n'a pas été distribuée. Une incompatibilité propre au pawn courant n'invalide plus le contrat statique du composant : une possession suivante valide reprend la détection sans redémarrage.

Le composant est instanciable directement, lié dans le build disque et chargé par le Blueprint enfant après redémarrage complet de l'éditeur.

**Gate :** build disque, tests natifs et composant instanciable sans Blueprint.

### J3 — Reparent et suppression du hotspot Blueprint — terminé

- reparenter `PlayerControllerInteractComponent` sur la classe native ;
- supprimer `ReceiveTick`, la boucle `UpdateIgnoreActors`, `CheckInteractionRz`, `SortByViewDotAlignment` et `CheckSomethingInBetween` après branchement de leurs seules règles encore vivantes ;
- supprimer les variables Blueprint désormais héritées et rafraîchir tous leurs usages ;
- supprimer le custom event `AfterCheck`, créer son override natif réel et reconnecter uniquement `UpdateHighlight`, le feedback hold et la visibilité associée ;
- remplacer la sélection actor/custom du graphe `InteractInput` par `GetInteractionDispatchTarget()` afin que le component exact reste atteignable jusqu'au RPC ;
- conserver uniquement `InteractClient`, le sit, la quête et la présentation.

La bascule utilise le reparent transactionnel durci de RzDirectMCP sur un package source propre : toutes les cibles sont validées avant mutation, les graphes obsolètes sont distingués des fonctions réellement redirigées, les nœuds peuvent être supprimés par GUID dans l'EventGraph ou un function graph, les enfants SCS sont préservés et toute erreur de strip, compile ou save déclenche le rollback complet. Si l'undo de transaction lui-même est indisponible, le package propre d'origine est rechargé depuis le disque au lieu de laisser une migration partielle en mémoire.

La bascule asset est exécutée et enregistrée :

- parent live : `QPlayerControllerInteractComponent` ;
- `423 -> 211` nœuds ;
- `4` variables Blueprint restantes, toutes propres à la façade de présentation ;
- `0` macro ;
- suppression de `ReceiveTick`, `UpdateIgnoreActors`, `CheckInteractionRz`, `SortByViewDotAlignment`, `CheckSomethingInBetween`, des variables dupliquées et de l'ancien custom event/RPC remplacé ;
- aucun nœud de trace ou de sweep restant ; les deux appels contenant `Tick` sont uniquement `Tick Enable Interact Feedback`, une sortie de présentation, pas une boucle de détection ;
- compilation Blueprint finale : `0` erreur, `0` warning.

**Gate :** le Blueprint compile à `0` erreur / `0` warning et ne contient plus aucune trace, sweep, tri ou boucle par frame.

### J4 — RPC serveur et consommateurs — terminé pour la frontière interaction

- remplacer le custom event Blueprint `SV_Interact` par le RPC natif revalidé ;
- convertir QATS à `UQPlayerControllerInteractComponent` et supprimer ses lectures réfléchies ;
- conserver temporairement le bridge QTrain seulement si la dépendance `QSystem -> QTrain` interdit le cast sans cycle ; ne pas créer un cycle de modules pour masquer ce contrat ;
- vérifier `Lib_Interact` et supprimer toute fonction devenue sans referencer.

QATS résout désormais le composant par type natif et lit directement `GetActiveInteractActor()` / `GetInteractionDistance()` ; les recherches par fragment de nom et les lectures réfléchies correspondantes sont supprimées. Le bridge QTrain reste la frontière réfléchie nommée entre modules, car remplacer ce seul appel par un cast créerait le cycle `QSystem -> QTrain -> QSystem`.

Les tests natifs couvrent actor, component et custom target, press/release et refus d'autorisation. Le runtime Standalone a confirmé actor target, component-only/custom dispatch, press/release, perte de focus et enable/disable. Les topologies Listen et Dedicated + deux clients ont confirmé l'ownership local et l'absence de détection sur les copies serveur/non locales.

**Gate :** une requête forgée hors distance, occluse, non interactive ou visant un autre objet est refusée ; press/release valide atteint `InteractServer` une seule fois.

### J5 — Cleanup assets et présentation — terminé pour le code remplacé

- supprimer le composant mort de `StandDeTir_Start` après audit de ses instances ;
- supprimer les variables inutilisées `ActiveInteractActor` et `InteractCustomTarget` de `Qanga_InputsComponent` si la recherche fraîche reste vide ;
- supprimer chaque graphe, variable, event, macro et fonction remplacé, pas seulement leurs connexions ;
- supprimer le Blueprint `PlayerControllerInteractComponent` uniquement si plus aucune règle de présentation ou d'interface ne justifie sa façade. Sinon il reste une façade mince et active, pas un second propriétaire.

État vérifié après redémarrage :

- `Qanga_InputsComponent.ActiveInteractActor` et `.InteractCustomTarget` supprimées après recherche asset exacte sans aucun usage ;
- composant mort `PlayerInteractComponent` supprimé de `StandDeTir_Start` ;
- trois événements désactivés, vides et non connectés supprimés du même asset ;
- `StandDeTir_Start` ne contient plus que son `StaticMeshComponent`, un EventGraph vide et le nœud automatique de construction ;
- les referencers directs du Blueprint interaction sont passés de quatre à trois : `QangaPlayerController`, `Qanga_InputsComponent` et `Lib_Interact` ;
- le Blueprint enfant reste nécessaire : ses `211` nœuds portent encore `InteractInput`, sit, highlight, feedback et quête. Il n'est donc pas un asset mort et ne doit pas être supprimé à ce stade.

### J6 — Validation finale sur `L_Dev_Rz` — migration validée

- Standalone : line target, sphere target, box target, component target, actor target, sit socket, highlight, hold/release, perte de cible et changement de pawn ;
- Listen Server : interaction locale serveur et interaction client distant ;
- Dedicated Server + deux clients : ownership RPC, acceptation valide, refus forgés et absence de détection sur les copies serveur/non owner ;
- scénarios réels QTrain, QCableConnector, elevator, item, véhicule et objectif de quête ;
- capture Unreal Insights après migration et comparaison structurelle des traces/sweeps/Blueprint VM, sans chiffre avant/après inventé.

Le seed EasyCook sera vidé au redémarrage demandé, sans rescan. La présence cooked des quatre contrats soft (`Interact_Interface`, `ALS_Character_BPI`, `Lib_Vehicle`, `RTDA_CameraIgnore`) reste donc un gate de release à vérifier après le prochain scan/cook lancé par le propriétaire du build, pas une preuve fabriquée pendant cette migration editor-only.

Preuves acquises :

- Standalone dans `L_Dev_Rz` : actor target, component-only/custom target, press/release, perte de focus et enable/disable ;
- Listen Server : deux mondes `L_Dev_Rz`, copie locale active et copie distante sans détection ;
- Dedicated Server + deux clients : trois mondes `L_Dev_Rz`, copies serveur `IsLocalController=false` et tick interaction coupé, deux owners clients locaux avec tick actif, defaults `175 cm` et `0,033333 s`, et résolution d'une cible actor live ;
- scénario sit réel QTrain en Standalone : un `BP_QTrainSeat` de production placé à `95 cm` devant le pawn est sélectionné par la requête native sans injection de cible ; `GetActiveInteractActor()` et `GetActiveInteractionTarget()` retournent le siège, le press envoyé par `SV_Interact` passe la revalidation serveur, `OccupantPawn` devient le pawn et celui-ci est attaché au siège ; la release ne bascule pas l'état, puis le press suivant remet le pawn debout, vide `OccupantPawn` et supprime l'attache ;
- scénario hold réel en Standalone : un `DEFAULT_Cyborg_V2` concret utilisant le `HasInteraction` de production est rendu recrutable dans le fixture PIE (`RecruitCost=0`, capacité de squad `-1`), immobilisé sur une route de patrol ponctuelle puis visé par le joueur. La détection native produit `CachedIsHoldInteract=true`, `CachedHoldDuration=3.0`, l'actor et l'objet exact attendus, ainsi que `InteractiveFeedbackCustomTarget` comme cible de dispatch. Le press puis la release passent par le vrai `InteractInput`; la release annule avant recrutement et le cyborg conserve uniquement son tag `Citizen` ;
- les deux scénarios réels QTrain et cyborg se terminent avec le Message Log renvoyé automatiquement par `stop_pie` : `2` infos, `0` warning et `0` erreur, sans entrée QTrain, interaction, recruitment ou feedback ;
- build disque complet vert après la bascule et après l'ajout de l'observabilité Message Log ;
- Message Log `PIE` du dernier run Dedicated + deux clients : `778` entrées, dont `772` erreurs préexistantes `LIb_LocalData` / `AchievementLogic_WorldStatisticThreshold`, mais `0` entrée contenant `PlayerControllerInteractComponent` ou `QPlayerInteraction` ;
- `stop_pie` attend désormais la fin synchrone de `EndPlayMap` et retourne automatiquement cette page Message Log dans sa propre réponse. `read_message_log` permet ensuite de la filtrer et de la paginer à la demande ;
- aucun package interaction n'est resté dirty après les sauvegardes et le redémarrage.

Gates de release encore ouvertes, sans revendication prématurée :

- la matrice métier étendue QCableConnector, elevator, item, véhicule et objectif de quête reste à rejouer avant release ; ces consommateurs n'ont pas été réécrits par ce chantier, tandis que les chemins génériques actor/component/custom target, press/release, hold et sit ont maintenant une preuve runtime réelle ;
- capture Unreal Insights après migration. Il n'existe pas de capture J0 comparable, donc seule la suppression structurelle du Blueprint VM et des requêtes physiques est actuellement démontrable ;
- gate cooked des quatre contrats soft après le prochain scan/cook effectué par le propriétaire du build. Aucun rescan ni cook n'a été lancé ici, conformément à la consigne.

## 4. Critères de sortie

Le chantier est terminé uniquement si :

- une seule instance/propriétaire d'interaction existe par controller ;
- aucune requête physique ou boucle de candidats ne reste en Blueprint ;
- aucun calcul de détection ne tourne sur dedicated server ou copie non locale ;
- les égalités de score conservent tous les candidats dans un ordre déterministe ;
- `ActiveInteractActor`, l'objet exact ayant accepté l'interaction et les valeurs hold/sit ont un seul propriétaire natif ;
- `SV_Interact` revalide le résultat côté serveur et les demandes invalides ne déclenchent pas `InteractServer` ;
- les scénarios Standalone, Listen et Dedicated + deux clients sont verts dans `L_Dev_Rz` ;
- le code et les assets remplacés sont supprimés, sans branche désactivée ni fallback legacy.
