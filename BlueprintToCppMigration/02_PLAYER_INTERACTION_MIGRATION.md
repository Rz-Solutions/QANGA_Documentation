# Deuxième migration Blueprint vers C++ — interaction joueur

- **État :** EN COURS — composant natif et tests purs implémentés, build disque et bascule asset en attente
- **Module propriétaire :** `QSystem` (aucun nouveau plugin)
- **Asset principal :** `/Game/Systems/Interact/PlayerControllerInteractComponent`
- **Carte de validation :** `/Game/Maps/LevelDev/L_Dev_Rz`
- **Début :** 2026-08-26

---

## 0. Décision

`UQPlayerControllerInteractComponent`, dans le module runtime existant `QSystem`, devient l'unique propriétaire de la détection, du classement, de l'état de focus et du funnel RPC serveur. Le Blueprint `PlayerControllerInteractComponent` est reparenté sur cette classe et reste une façade transitoire pour les appels à `Interact_Interface`, le choix de forme lié au mode de vue ALS, le highlight, le feedback, le sit et la progression de quête.

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

Deux coutures Blueprint restent justifiées :

- résolution du mode de sweep depuis le mode de vue ALS et du fallback véhicule ;
- présentation : l'événement existant `AfterCheck`, le highlight, le crosshair, le feedback, le sit et la progression de quête.

Elles ne refont ni trace, ni tri, ni occlusion, ni mutation concurrente de la cible. La signature native de `AfterCheck` reprend exactement les cinq entrées du custom event actuel ; Unreal peut donc conformer son corps de présentation au reparent au lieu de créer un second hook transitoire.

### 2.4 État et autorité

Le composant natif possède `PlayerController`, `PawnCache`, `ActiveInteractActor`, `InteractCustomTarget`, `DistanceDetection`, le résultat sit, le hit item et les valeurs hold. Un seul funnel compare ancien/nouveau focus, appelle `InteractLost`, remplace l'état puis notifie la façade de présentation.

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

### J1 — Math pure et tests — implémenté, validation disque en attente

- ajouter score, ordre total, construction de géométrie et décision d'occlusion en fonctions pures ;
- tester égalités de dot, impacts superposés, valeurs non finies, ordre d'entrée différent, bornes exactes et conservation de tous les hits ;
- tester l'échéance `1/30 s` et la neutralisation des sorties.

Implémenté dans `QPlayerControllerInteractComponent.cpp` : géométrie exacte, score fini, ordre total sans perte d'égalité, décision d'occlusion et échéance sans rattrapage après hitch. Les tests `QSystem.Interaction.QueryGeometry`, `Ordering`, `Deadline`, `Occlusion` et `LegacyContract` sont ajoutés ; le dernier vérifie aussi les signatures réelles de l'interface et le schéma de la shared ignore-list. Ils ne seront marqués verts qu'après le prochain build disque.

**Gate :** tests `QSystem.Interaction.*` verts sans monde gameplay.

### J2 — Composant natif — implémenté, validation disque en attente

- ajouter `UQPlayerControllerInteractComponent` dans `QSystem` ;
- implémenter local-only, échéance polled, cache pawn/caméra, ignore list, trace primaire, sweeps, tri et occlusion ;
- ajouter les scopes Unreal Insights et le bridge validé vers l'interface Blueprint ;
- exposer l'état et les hooks de présentation nécessaires, sans dupliquer les variables.

`UQPlayerControllerInteractComponent` est maintenant présent dans `QSystem`. Il possède la détection locale à `30 Hz`, le refresh d'ignore à `1 Hz`, les requêtes physiques, le tri déterministe, l'occlusion, le focus unique, la neutralisation des sorties, le bridge contrôlé vers `Interact_Interface` et la revalidation du RPC `SV_Interact`. Le tick est coupé sur dedicated server et copies non locales. Le serveur recalcule sa propre requête avant chaque press et ne libère que la cible exacte autorisée par ce press.

**Gate :** build disque, tests natifs et composant instanciable sans Blueprint.

### J3 — Reparent et suppression du hotspot Blueprint

- reparenter `PlayerControllerInteractComponent` sur la classe native ;
- supprimer `ReceiveTick`, la boucle `UpdateIgnoreActors`, `CheckInteractionRz`, `SortByViewDotAlignment` et `CheckSomethingInBetween` après branchement de leurs seules règles encore vivantes ;
- supprimer les variables Blueprint désormais héritées et rafraîchir tous leurs usages ;
- conserver uniquement les hooks ALS/véhicule et la présentation.

**Gate :** le Blueprint compile à `0` erreur / `0` warning et ne contient plus aucune trace, sweep, tri ou boucle par frame.

### J4 — RPC serveur et consommateurs — partiellement implémenté

- remplacer le custom event Blueprint `SV_Interact` par le RPC natif revalidé ;
- convertir QATS à `UQPlayerControllerInteractComponent` et supprimer ses lectures réfléchies ;
- conserver temporairement le bridge QTrain seulement si la dépendance `QSystem -> QTrain` interdit le cast sans cycle ; ne pas créer un cycle de modules pour masquer ce contrat ;
- vérifier `Lib_Interact` et supprimer toute fonction devenue sans referencer.

QATS résout désormais le composant par type natif et lit directement `GetActiveInteractActor()` / `GetInteractionDistance()` ; les recherches par fragment de nom et les lectures réfléchies correspondantes sont supprimées. Le bridge QTrain reste temporairement en place pour éviter le cycle de modules `QSystem -> QTrain -> QSystem`.

**Gate :** une requête forgée hors distance, occluse, non interactive ou visant un autre objet est refusée ; press/release valide atteint `InteractServer` une seule fois.

### J5 — Cleanup assets et présentation

- supprimer le composant mort de `StandDeTir_Start` après audit de ses instances ;
- supprimer les variables inutilisées `ActiveInteractActor` et `InteractCustomTarget` de `Qanga_InputsComponent` si la recherche fraîche reste vide ;
- supprimer chaque graphe, variable, event, macro et fonction remplacé, pas seulement leurs connexions ;
- supprimer le Blueprint `PlayerControllerInteractComponent` uniquement si plus aucune règle de présentation ou d'interface ne justifie sa façade. Sinon il reste une façade mince et active, pas un second propriétaire.

### J6 — Validation finale sur `L_Dev_Rz`

- Standalone : line target, sphere target, box target, component target, actor target, sit socket, highlight, hold/release, perte de cible et changement de pawn ;
- Listen Server : interaction locale serveur et interaction client distant ;
- Dedicated Server + deux clients : ownership RPC, acceptation valide, refus forgés et absence de détection sur les copies serveur/non owner ;
- scénarios réels QTrain, QCableConnector, elevator, item, véhicule et objectif de quête ;
- capture Unreal Insights après migration et comparaison structurelle des traces/sweeps/Blueprint VM, sans chiffre avant/après inventé.

## 4. Critères de sortie

Le chantier est terminé uniquement si :

- une seule instance/propriétaire d'interaction existe par controller ;
- aucune requête physique ou boucle de candidats ne reste en Blueprint ;
- aucun calcul de détection ne tourne sur dedicated server ou copie non locale ;
- les égalités de score conservent tous les candidats dans un ordre déterministe ;
- `ActiveInteractActor` et les valeurs hold/sit ont un seul propriétaire natif ;
- `SV_Interact` revalide le résultat côté serveur et les demandes invalides ne déclenchent pas `InteractServer` ;
- les scénarios Standalone, Listen et Dedicated + deux clients sont verts dans `L_Dev_Rz` ;
- le code et les assets remplacés sont supprimés, sans branche désactivée ni fallback legacy.
