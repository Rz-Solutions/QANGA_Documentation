# Deuxième migration Blueprint vers C++ — interaction joueur

- **État :** MIGRATION NATIVE TERMINÉE — hotspot basculé et code remplacé supprimé ; correction d'entrée véhicule dédiée recorrigée depuis les preuves du serveur déployé, validation Development encore ouverte
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

Le composant natif possède `PlayerController`, `PawnCache`, `ActiveInteractActor`, l'objet exact `ActiveInteractionTarget` ayant accepté `HasInteraction`, `InteractCustomTarget`, `DistanceDetection`, le résultat sit, le hit item et les valeurs hold. Un seul funnel compare ancien/nouveau focus, appelle `InteractLost` une fois sur chaque ancien actor/component/custom target distinct, remplace l'état puis notifie la façade de présentation. `GetInteractionDispatchTarget()` fournit à l'input l'unique cible à transmettre au RPC : custom target valide, sinon objet exact ayant accepté l'interaction, uniquement tant que cette cible exacte et son éventuel composant source obligatoire restent valides. Il ne retombe jamais sur l'actor de focus.

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

`UQPlayerControllerInteractComponent` est maintenant présent dans `QSystem`. Il possède la détection locale à `30 Hz`, le refresh d'ignore à `1 Hz`, les requêtes physiques, le tri déterministe, l'occlusion, le focus unique, la neutralisation des sorties, les bridges contrôlés vers `Interact_Interface`, l'interface ALS et `Lib_Vehicle`, ainsi que la revalidation du RPC `SV_Interact`. Le component et l'actor sont interrogés indépendamment ; un component qui implémente seul l'interface reste donc une cible complète et reçoit aussi `InteractLost`. Le chemin sit conserve le contrat Blueprint exact : actor en cible principale, hit component en custom target, hold actif et durée nulle. Le tick est coupé sur dedicated server et copies non locales. Les événements non-hold restent stateless : chaque press/release est revalidé et distribué indépendamment, y compris avec un identifiant `None` déjà utilisé par des consommateurs historiques. Seuls les holds explicitement déclarés conservent une paire serveur cible/id ; une release exacte la ferme, une release forgée ne la modifie pas et les interactions non-hold restent indépendantes. Une incompatibilité propre au pawn courant n'invalide plus le contrat statique du composant : une possession suivante valide reprend la détection sans redémarrage.

Le composant est instanciable directement, lié dans le build disque et chargé par le Blueprint enfant après redémarrage complet de l'éditeur.

**Gate :** build disque, tests natifs et composant instanciable sans Blueprint.

### J3 — Reparent et suppression du hotspot Blueprint — terminé

- reparenter `PlayerControllerInteractComponent` sur la classe native ;
- supprimer `ReceiveTick`, la boucle `UpdateIgnoreActors`, `CheckInteractionRz`, `SortByViewDotAlignment` et `CheckSomethingInBetween` après branchement de leurs seules règles encore vivantes ;
- supprimer les variables Blueprint désormais héritées et rafraîchir tous leurs usages ;
- supprimer le custom event `AfterCheck`, créer son override natif réel et reconnecter uniquement `UpdateHighlight`, le feedback hold et la visibilité associée ;
- remplacer la sélection actor/custom du graphe `InteractInput` par `ResolveInteractionInputTarget(Pressed)` afin que la cible exacte du Down reçoive aussi le Up après perte de focus ou changement de possession ;
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

### J6 bis — Régressions runtime post-migration — corrigé et validé

- `InteractInput` mémorise la cible exacte du Down et lui distribue le Up même si le focus ou la possession change pendant l'action. Un Up sans Down ne retombe pas sur la cible courante.
- Les interactions non-hold sont à nouveau stateless et répétables ; elles ne créent plus un verrou serveur global. Seuls les holds déclarés conservent une paire cible/id exacte jusqu'à leur release.
- `VehicleBase` alimente le pin `Released` du hold. Un tap ou un hold interrompu annule et remet la progression à zéro sans RPC serveur ; un hold terminé mémorise l'id réellement envoyé, puis émet une seule release exacte au Up.
- `W_Trackers` met à jour ses bornes viewport dès qu'un tracker d'interaction valide existe, même si `TrackersToShow` est vide. Le prompt d'interaction ne retombe plus sur la position `(200,200)` du clamp non initialisé.
- `StarMap_Component` initialise les références capture/composant avant le widget minimap ; les six canvases distincts reçoivent leur propriétaire exact et la mise à jour minimap reste indépendante du chargement de la StarMap complète.
- La validation manuelle finale dans `L_Dev_Rz` confirme les press répétés sur un même bouton et en alternance, l'annulation tap/hold partiel, la sortie véhicule après hold complet, la répétition du cycle et l'interaction immédiate après sortie. Le prompt reste projeté sur sa cible et les icônes minimap ne s'empilent plus en haut à gauche.
- Deux QATS natifs couvrent désormais la cible input mémorisée ainsi que la séparation entre événements non-hold stateless et holds appariés.
- La détection distingue maintenant la provenance de chaque candidat. Le véhicule courant est refusé comme cible d'entrée ou socket de siège et aucun focus synthétique n'est créé sur lui. Les interactables de sa hiérarchie restent disponibles pour un passager, mais sont refusés lorsque le pawn possédé est le véhicule lui-même.
- La sortie véhicule appartient uniquement au chemin d'input : sans cible monde, le press sélectionne et mémorise explicitement le véhicule courant. Le release retourne à cette cible exacte sans réactiver le prompt de focus.
- La visibilité est appliquée aux candidats de la trace primaire comme des sweeps. Une cible component n'accepte que le hit de son primitive exact : une coque, un mur ou un parent attaché reste donc un occluder même lorsqu'il appartient au même actor.
- La requête serveur reconstruit la même éligibilité et la même visibilité que la détection locale avant de distribuer l'interaction ; aucun ancien fallback monde ne contourne ce funnel.
- Les quatre QATS `QATS.QSystem.PlayerInteraction.*` couvrent la cible input mémorisée, la disposition serveur, l'hystérésis et la matrice pilote/passager/sortie/occlusion.
- La validation manuelle dans `L_Dev_Rz` confirme que le prompt d'entrée du véhicule disparaît pendant la conduite, que le hold de sortie reste fonctionnel et qu'un bouton masqué par un mur ne peut plus être ciblé ni activé.

### J6 ter — Divergence du build Development — corrigée, validation packaged à rejouer

- Le build Development contient bien les contrats interaction, feedback, ignore et véhicule attendus dans son manifest et dans le chunk `0`. La divergence n'était donc pas une dépendance absente du cook et aucune règle de cook supplémentaire n'a été ajoutée.
- Le log packaged montre des press hold `Driver` répétés après activation, puis refusés car le même hold serveur est encore actif. `InputSystem` émet volontairement `Pressed` pendant le maintien ; le graphe partagé `VehicleBase` réarmait alors sa temporisation à chaque répétition.
- `VehicleBase` filtre désormais les répétitions `Pressed` tant que son hold est actif. La release conserve son chemin exact, ferme l'id mémorisé puis permet un nouveau cycle, sans correctif décliné dans chaque classe de véhicule.
- Une indisponibilité transitoire du bridge véhicule n'invalide plus les contrats statiques. Elle bloque les nouveaux press jusqu'au retour d'un contexte runtime cohérent, tout en conservant la cible mémorisée nécessaire à la release déjà engagée.
- Une perte de détection isolée conserve uniquement la présentation pendant `150 ms`, à condition que la cible reste à distance, dans la vue, visible, éligible et liée au même composant source. Cette grâce n'autorise aucun nouveau dispatch.
- Un changement de pawn ou de véhicule, une occlusion, une sortie du véhicule ou la disparition du composant source expire immédiatement le focus. Une interaction issue d'un composant ne peut pas être rétrogradée silencieusement vers son acteur.
- Les tests de décision couvrent les transitions press/release, l'indisponibilité runtime, la grâce de présentation, son expiration, l'occlusion et les changements de contexte. Le build packaged doit encore être régénéré puis rejoué pour confirmer le comportement avec son timing réel.

### J6 quater — Fluidité du prompt en build Development — corrigée, validation packaged à rejouer

- L'ancien tick Blueprint par frame masquait un contrat de présentation dépendant de la cadence : le tracker temporaire était réarmé et réattaché à chaque détection native à `30 Hz`, avec une expiration de `120 ms` susceptible de détruire le prompt pendant un hitch packaged.
- La détection conserve sa cadence native à `30 Hz`, mais la présentation devient événementielle : une acquisition ou un remplacement de focus crée et attache une seule fois un tracker persistant ; un focus stable ne renvoie plus de heartbeat UI.
- La perte ou le remplacement de cible détruit explicitement l'ancien tracker avant d'exposer le nouveau focus. Un tracker ne peut donc plus expirer, sauter ou survivre à sa cible selon l'espacement des polls.
- La propriété de présentation est conservée jusqu'au cleanup même si l'ancien actor est déjà pending-kill. La réutilisation d'un tracker persistant transmet directement sa lifetime nulle et ne réintroduit plus la lease minimale de `100 ms` des trackers temporaires.
- Le changement `Press` / `Hold` reconstruit immédiatement le texte visible avec la même source de touche et le même format que le tracker ; il ne dépend plus d'un événement d'input ultérieur et ne peut plus conserver le libellé de la cible précédente.
- La position écran reste calculée par le système tracker après chaque tick monde. Le mouvement visuel suit donc la caméra à la cadence d'affichage et n'est plus quantifié par la fréquence de détection.
- Les transitions acquisition, focus stable, mise à jour du payload, remplacement et perte sont couvertes par le test natif de présentation. Le build Development doit encore être régénéré et vérifié visuellement par le propriétaire du build.

### J6 quinquies — Focus embarqué pendant le pilotage — corrigé, validation manuelle à rejouer

- L'éligibilité savait identifier le véhicule courant mais ne connaissait pas l'autorité de possession. Un composant ou actor attaché restait donc ciblable depuis certains angles, que le joueur pilote réellement le véhicule ou soit seulement présent à bord.
- Le contexte candidat porte désormais l'état de pilotage dérivé de l'identité entre le pawn possédé et le véhicule courant. Toute cible appartenant à cette hiérarchie est refusée pendant le pilotage actif, sans nom d'asset, tag ou exception locale.
- La sortie explicite reste autorisée par son chemin d'input dédié. Les mêmes interactables restent disponibles pour un passager ou après avoir quitté le poste de pilotage, et une interaction monde sans lien avec le véhicule n'est pas masquée.
- La détection locale, la rétention du focus et la revalidation serveur consomment toutes la même décision. La QATS d'éligibilité couvre le pilote, le passager, la sortie explicite et une interaction monde indépendante.

### J6 sexies — Audit de parité packaged — défaut natif réparé, validation packaged ouverte

- L'audit statique a trouvé une fenêtre de rétrogradation encore active dans `GetInteractionDispatchTarget()` : si l'objet exact ayant accepté l'interaction devenait invalide avant le poll natif suivant, le getter renvoyait `ActiveInteractActor`. Un sit socket pouvait aussi conserver l'actor alors que son composant source obligatoire avait disparu.
- Le dispatch exige désormais que l'objet exact et, lorsque le candidat l'impose, son composant source physique soient encore valides. La cible custom ne gagne la priorité qu'après cette validation et aucun actor de focus n'est utilisé comme fallback. La QATS de résolution couvre la priorité custom, la cible exacte, la perte de l'objet exact et la perte du composant source obligatoire.
- La source actuelle conserve un seul propriétaire de détection/focus et un seul funnel serveur : cadence de détection `30 Hz`, refresh ignore `1 Hz`, présentation événementielle, logs d'échec limités à `1 Hz`, tick uniquement sur le controller local, requête serveur reconstruite avec la même sélection, éligibilité et occlusion.
- Le dernier exécutable Development disponible dans l'installation Steam date du `2026-08-28 18:58:38`. Son log confirme `Build Configuration: Development`, mais cet exécutable précède les commits tracker/prompt `90f0ba1e4` (`19:55:22`) et filtre pilote `0d57d4dc8` (`23:14:42`) : il ne peut pas valider ces corrections.
- Ce log Development montre, entre `17:39:05.456` et `17:39:08.518`, un `W_TrackerElem` qui continue `WarpOcclusionUpdate` avec un `TrackerComponent` pending-kill. Le log ne contient pas l'identité métier du tracker ; cette trace prouve un défaut de cycle de vie dans cet ancien build, pas qu'il s'agit du prompt d'interaction actuel. Le même build signale aussi `FloorComponent` nul dans `BP_Elevator.SetInteractionEnabled`, hors du propriétaire natif interaction.
- Le log Development Editor du `2026-08-26` contient les anciens refus `Driver` / `MiddleDoor` répétés et une release sans paire autorisée. Il précède les correctifs interaction concernés et sert uniquement de preuve historique du mécanisme. Les logs Editor du `2026-08-30` et le dernier log client local, Shipping du `2026-08-27`, ne contiennent pas d'événement `LogQPlayerInteraction`; leur silence ne prouve aucun scénario.
- Un rebuild Development frais, puis les replays Editor et packaged de la matrice interaction restent obligatoires. Aucun build, test runtime, PIE, cook ou package n'a été exécuté pendant cet audit.

### J6 septies — Entrée et sortie véhicule en Development — point d'entrée authored recorrigé, validation déployée finale ouverte

- Le test Development Standalone a isolé un premier défaut : une interaction sérialisée comme hold à durée nulle ouvrait une paire serveur persistante sans release correspondante, puis pouvait bloquer la sortie `Driver`. Le serveur ne conserve désormais une paire press/release que pour un hold dont la durée reconstruite est finie et strictement positive. Un hold à durée nulle ou non positive reste soumis à la validation complète, mais ses deux fronts sont dispatchés sans état persistant. La sortie du vaisseau a ensuite été confirmée fonctionnelle par le propriétaire du build.
- Les snapshots immuables du client Steam et du serveur `QangaDev`, archivés sous `Saved/Diagnostics/DedicatedVehicleEntry`, prouvent un second défaut distinct : les RPC `Driver` atteignent bien le serveur, mais cinq pressions successives visant le même hovercraft sont refusées avant tout dispatch ou changement de possession. La cible demandée apparaît dans zéro hit primaire et zéro hit de sweep ; l'autorisation actor/component/custom n'est donc jamais atteinte. Sa distance au pivot varie de `189` à `244 cm`, au-delà des `175 cm` authored, alors que le joueur est physiquement au contact du véhicule.
- La revalidation serveur réutilisait la sélection locale complète, puis n'autorisait que son premier candidat classé. Elle imposait donc au client et au serveur de choisir le même premier objet malgré leurs vues, ordres de hits et identités d'objets propres à chaque processus. Une cible demandée valide pouvait être présente dans la même requête autoritaire et néanmoins être refusée parce qu'un autre interactable était classé devant elle.
- La détection locale conserve sa sélection first-ranked. Pour les interactions ordinaires, la revalidation serveur parcourt les candidats classés jusqu'à retrouver la cible explicitement demandée ; le résultat temporaire de chaque hit est isolé afin qu'un candidat refusé ne puisse pas contaminer le suivant. Cette correction est nécessaire aux divergences de classement, mais les nouveaux logs prouvent qu'elle ne peut pas résoudre une cible entièrement absente des hits autoritaires.
- Une seconde reproduction déployée, entre `20:47:41` et `20:48:06`, atteint douze fois la nouvelle branche directe avec l'actor exact du hovercraft. Le serveur trouve `18` primitives dont `6` compatibles query, mais `GetSquaredDistanceToCollision` ne produit aucun point sur les six et laisse `vehicleSurfaceCm=-1`. Cette preuve falsifie le contrat de distance physique utilisé par la première correction. Le calcul collision et son substitut AABB non validé sont entièrement supprimés ; aucune bounds englobante ne peut autoriser du vide autour d'un véhicule.
- Le propriétaire authored du point d'entrée existe déjà dans `VehicleBase.HasInteraction` : le véhicule choisit son `VehicleSlot` disponible et y attache `Vehicle_InteractiveFeedbackCustomTarget`. Après l'éligibilité de possession, la revalidation serveur appelle ce `HasInteraction` une seule fois, puis exige que la cible retournée soit un `USceneComponent` valide, enregistré, dans le même monde et possédé par l'actor véhicule.
- Le routage est explicite et à trois états. L'actor véhicule et l'exact custom target émis utilisent la validation d'entrée ; tout autre sous-objet monté sur le véhicule conserve le chemin générique, même quand le véhicule n'a momentanément aucun siège disponible. L'échec d'une vraie entrée reste terminal et ne retombe jamais sur une seconde requête permissive.
- Un build client/serveur Development frais a ensuite confirmé que la branche directe permet bien de prendre le contrôle des véhicules sur dedicated server. Le reliquat était plus précis : tant que le joueur collait la porte l'entrée fonctionnait, mais la validation mesurait encore le point de siège situé dans l'habitacle. Les rejets observés plaçaient ce siège jusqu'à environ `248 cm` du pawn alors que le prompt local désignait correctement l'entrée accessible.
- Le siège sélectionné reste l'unique cible de feedback et de dispatch, mais il ne possède plus la portée. La validation lit son `SlotKey` sur le parent immédiat, appelle le `GetExitSlot` authored du véhicule et utilise ce point de porte pour la distance et la visibilité. Quand aucun point spécifique n'est déclaré, elle reprend exclusivement le `ExitPosition` de base déjà utilisé par `GetValidExitPosition` ; une hiérarchie, une signature ou une référence invalide échoue explicitement au lieu de retomber sur le siège ou sur une géométrie approximative.
- La portée authored reste exactement `175 cm`, mesurée entre le pawn autoritaire et ce point d'entrée/sortie, avec arithmétique finie. Le serveur conserve la ligne de visibilité, l'autorisation actor/custom target et le funnel hold avant dispatch. Aucun point client, rayon élargi, corps physique ou bounds approximative n'est accepté.
- La reconstruction d'autorité conserve le point de vue caméra pour un controller local et utilise les yeux du pawn pour un controller distant. La détection locale first-ranked, la recherche générique target-aware, les composants interactifs montés et le chemin de sortie explicite restent séparés.
- Le diagnostic agrégé `[ServerValidationDiag]`, toujours limité par le throttle interaction à une ligne par seconde, expose le `SlotKey`, le nom et la distance du point de porte réellement validé. Les compteurs de primitives et la distance de surface supprimée ne peuvent plus faire croire qu'un corps physique absent est le propriétaire du contrat d'entrée.
- Les unités `QPlayerControllerInteractComponent`, `QPlayerControllerInteractionTests`, `QSystem` et `QAutomatedTestSuite` compilent après la réécriture. Le nouveau symbole exporté ne peut pas être relié à une DLL QATS déjà chargée par Live Coding ; l'exécution du test d'intégration `VehicleEntryRangeAnchor` attend donc un build Editor froid, sans réinterpréter un ancien module comme preuve.
- La sortie de vaisseau et l'entrée dédiée des véhicules sont confirmées sur le dernier Development déployé. La seule gate runtime encore ouverte dans cette sous-tranche est le nouveau calcul portée/visibilité sur le point de porte, à rejouer dans le prochain client et serveur reconstruits ; aucune validation de l'ancien point de siège ne vaut pour ce source.

### J6 octies — Contrat de flotte et retournement dedicated — source/QATS verts, déploiement à revalider

- L'audit sémantique de toute la hiérarchie véhicule a identifié deux contrats d'entrée authorés. Les hovercrafts, véhicules terrestres, watercrafts et transporteurs utilisent l'actor/root partagé de `VehicleBase`; les vaisseaux pilotables authorent exactement un composant enregistré dont l'`Id` vaut `Driver`. Le validateur serveur résout ces deux contrats sans nom de modèle ni liste d'exceptions. Zéro composant `Driver` sélectionne le contrat root, un sélectionne ce composant exact et plusieurs sont rejetés comme authoring ambigu.
- Seul l'`Id` d'action `Driver` emprunte cette résolution d'entrée. Les portes, fenêtres, échelles, sièges passagers et armes montées conservent la requête générique actor/component avec visibilité exacte ; leur `Id` d'action n'est pas comparé au champ `Id` du composant, car ces deux valeurs diffèrent volontairement dans plusieurs vaisseaux. Le dispatch conserve l'actor et l'action demandés.
- Une vraie entrée doit toujours produire le custom target de siège authoré, puis passe la portée et la visibilité du point d'entrée/sortie. À l'inverse, un `HasInteraction=true` accompagné d'un custom target strictement nul désigne une action actor-level : le serveur la repasse par la requête de vue exacte au lieu de l'interpréter comme un siège invalide. Un pointeur non nul mais invalide reste un échec explicite.
- `BikeBase` et `WatercraftBase`, les deux bases qui déclarent la capacité de retournement, exécutaient `TryFlip` derrière un Client RPC porté par le véhicule renversé et donc sans owning connection sur dedicated server. Ce RPC et son événement sont supprimés ; leur `InteractServer` déjà autoritaire appelle maintenant directement `TryFlip` lorsque le véhicule est renversé et inoccupé. Le chemin normal continue vers le parent pour l'entrée.
- Le source Live Coding compile, les suites `QSystem.Interaction.*` et `QATS.QSystem.PlayerInteraction.*` passent respectivement à `9/9` et `5/5`, et les cinq bases plus des représentants hovercraft, bike, watercraft, transporteur et vaisseaux à contrats root/composant recompilent sans diagnostic. Cette preuve ferme le contrat statique et Editor ; le retournement et l'entrée/portes de chaque famille doivent encore être rejoués dans un client Development contre le dedicated server reconstruit avant de revendiquer la parité déployée.

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
