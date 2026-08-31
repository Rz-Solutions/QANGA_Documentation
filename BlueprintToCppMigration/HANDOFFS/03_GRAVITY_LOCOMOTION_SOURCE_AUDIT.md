# Audit source J4-J6 Gravity / Locomotion

## Statut

Audit statique terminé sur la source courante. Les corrections source et la couverture QATS déterministe manquante sont présentes dans le worktree partagé.

Aucun build, UHT, QATS, compile Blueprint, Editor, PIE, RzMCP, asset, cook, package, stage ou commit n'a été exécuté dans cette lane. Tous les gates d'exécution restent ouverts.

## Evidence relue

- `agents.md`, `BLUEPRINT_TO_CPP_MIGRATION_PLAN.md` et `03_CHARACTER_GRAVITY_MIGRATION.md` lus intégralement.
- Diffs courants relus avant modification, en conservant les hunks QAI/Ninja sans rapport avec la gravité.
- Historique récent inspecté, notamment `3bd272b02`, `976971f95` et `dc587c453`, plus l'historique QAI de la rotation cyborg.
- Headers et implémentations Ninja réels inspectés pour `GetGravityDirection`, `GetFinalGravityScale`, les setters point/fixed, les multicasts, `UpdateGravity`, `OnMovementUpdated`, `UpdateComponentRotation` et la capture/restauration directionnelle.
- UE 5.7 réel inspecté pour la signature `FCharacterMovementUpdatedSignature`, l'ordre `OnMovementUpdated -> CallMovementUpdateDelegate`, `APawn::GetVelocity` et `FMath::QInterpTo`.
- Recherche des writers source effectuée dans GravityScape, QModule, QPlatform, QAI et NinjaCharacter.

## Fichiers et edits de cette lane

- `QGravityArea.h/.cpp` : ownership/restauration J4, isolation du scale physics-volume, validation du CMC réel sur no-op, surface query liée à l'owner, validation atomique J6 et lifecycle delegate unique.
- `NinjaCharacterMovementComponent.h/.cpp` : propriétaire natif unique, écritures point/fixed/restauration qualifiées, verrou des 12 setters directionnels et guards des 15 multicasts gravité, sans bypass réentrant ; source J5 relue sans ajouter de deuxième phase.
- `QAI_AgentComponent.h/.cpp` : résolution déterministe de tangente, cache gravity-component au `BeginPlay`, requête trois états, rotation cyborg authority-only et prerequisites explicites.
- `QGravityScapeAutomationTests.cpp` et `QGravityAreaAutomationTestProbe.h` : assertions déterministes manquantes sur ownership/réentrance, tangente QAI, bridge reflected et invalidation numérique.
- `03_CHARACTER_GRAVITY_MIGRATION.md` : uniquement les faits source/tests prouvés et les gates encore ouverts.

## J4, ownership gravité personnage

### Contrat confirmé

- `BaseGravityScale` est capturé une fois depuis l'instance CMC, uniquement s'il est fini.
- Le scale CMC est composé par le propriétaire unique depuis base, zone signée et coussin. Le chemin QPlatform compose direction et scale avec son alpha sans créer de second writer.
- Les produits `double` sont validés comme finis et représentables en `float` avant écriture.
- Point utilise le centre vectoriel exact. Fixed utilise une direction normalisée non signée. Le signe reste exclusivement dans le scale, puis Ninja l'applique une seule fois dans `GetGravityDirection(false)`.
- Forced location/scale restent des valeurs du provider. Aucun overlap ou calcul radial par frame n'a été ajouté.
- Hors zone sans plateforme, le scale est zéro et aucune ancienne direction géométrique n'est présentée comme une surface valide.
- `QuerySurfaceUp` garde trois états explicites : `ValidUp`, `DirectionlessZero`, `Unavailable`; une perte d'ownership rend immédiatement la requête unavailable au lieu d'exposer une surface périmée.

### Défauts réparés

1. Le fast path comparait seulement le dernier contrat logique. Une direction ou un `GravityScale` CMC modifié pouvait donc survivre à une notification identique. Le no-op exige maintenant que le mode/donnée directionnels réels et le scale réel correspondent aussi au contrat enregistré.

2. `ANinjaPhysicsVolume` reste vivant pour ses autres consommateurs. Il multiplie pourtant `GetFinalGravityScale()` et pousse ses directions aux Ninjas suivis. J4 capture, impose et restaure maintenant `bIgnoreOtherGravityScales`, en plus de `bDisableGravityReplication`, `bAlignGravityToBase` et `bRevertToDefaultGravity`. Le composite natif ne peut donc plus être remultiplié par le volume actif.

3. Le CMC Ninja expose maintenant un propriétaire explicite et unique du contrat natif. Tant que J4 le détient, tous les setters directionnels publics concurrents sont refusés ; les écritures point/fixed et la restauration exacte passent uniquement par des méthodes qualifiées par ce propriétaire. Une destruction ou un refresh déclenché par un listener reste contrôlé par l'identité de source, l'identité d'area et `ApplicationRevision` avant l'écriture du scale. Le teardown ne restaure rien s'il a perdu l'ownership, donc il ne peut pas écraser un propriétaire plus récent.

4. Les implementations multicast Ninja de tous les modes directionnels, du scale et d'`AlignGravityToBase` refusent une livraison lorsque `bDisableGravityReplication` est actif localement ou qu'un propriétaire natif est présent. Un ancien RPC fiable en vol ne peut plus reprendre l'ownership sur un simulated proxy après l'activation du calcul local J4.

5. Le teardown désactive d'abord l'application et retire les delegates, reprend temporairement les quatre flags, restaure l'état directionnel exact et le scale de base, puis restaure les quatre flags authored avec la réplication en dernier.

### Writers restants

- Aucun writer direct de `UCharacterMovementComponent::GravityScale` n'est présent dans QModule ou QPlatform. QModule pousse le multiplicateur typé, QPlatform pousse l'override typé.
- Les setters Ninja et `ANinjaPhysicsVolume` restent des API partagées pour les consommateurs non J4. Leurs écritures directionnelles sont refusées pendant l'ownership J4 et leurs multiplicateurs de scale sont isolés.
- Les flags de rotation joueur/QAI ne font pas partie de J4.

## J5, rotation surface et ordre QAI

### Joueur

- `bSmoothAlignToGravity` est opt-in et sa seule phase d'écriture gravité est `UNinjaCharacterMovementComponent::OnMovementUpdated`.
- Les autres chemins immédiats floor/gravity reçoivent l'axe capsule courant lorsque le smooth est actif, donc ne deviennent pas un deuxième writer.
- Activation exacte : `GravityScale >= +0.15` ou `< -0.15`.
- Mapping exact : scale signé `0.15..0.5` vers vitesse `0.5..6`, clampé, puis `FInterpTo` à `4`.
- La vitesse lissée continue dans la dead-band, mais l'orientation n'y est pas écrite.
- La rotation interpole un quaternion, puis utilise `UpdateComponentRotation`; le sweep capsule et la rotation de vélocité Ninja restent propriétaires.
- Writers autorisés : autorité ou pawn localement contrôlé. Un simulated proxy ne produit pas l'alignement et consomme la rotation acteur répliquée.
- Directionless conserve l'orientation. Le scale négatif n'est pas inversé deux fois.

### QAI

- `UQGravityCharacterComponent` est résolu une fois au `BeginPlay`; aucune recherche de ce composant n'existe dans le tick gravité.
- La rotation corps cyborg est authority-only. Les proxies clients ne réappliquent plus une rotation concurrente.
- `Unavailable` ne tombe pas sur actor-up et journalise au maximum une fois par seconde et par agent. `DirectionlessZero` n'écrit rien.
- `ValidUp` continue l'alignement à vitesse nulle en reprojetant le heading mémorisé.
- Défaut réparé : si le forward mémorisé est parallèle au nouvel up, sa projection était zéro et l'écriture complète était sautée. `QAIGravityAlignment::TryResolveSurfaceForward` essaie le facing demandé, le forward source, puis le right source orthogonal. L'up valide reste donc appliqué avec une tangente déterministe.
- Ordre source : QAI est `TG_PrePhysics`, après le tick owner et `UQAI_FloatingPawnMovement`; le mesh possède une prerequisite après QAI. L'AnimBP doit donc échantillonner la rotation corps finale.

## J6, essential locomotion values

- `UQGravityLocomotionComponent` ne ticke pas.
- Il se lie et se délie exactement à `ACharacter::OnCharacterMovementUpdated`.
- UE diffuse ce delegate après `UNinjaCharacterMovementComponent::OnMovementUpdated` et après le scoped movement update.
- Accélération exacte : `(CurrentVelocity - OldVelocity) / DeltaSeconds`, avec le `OldVelocity` du callback. Aucun cache par frame ou calcul gravity-space n'existe.
- `Speed` est la longueur 3D. `IsMoving` est strictement `Speed > 1 cm/s`.
- Toutes les entrées sont validées, puis les résultats calculés sont eux-mêmes validés. Un overflow produit depuis des entrées finies neutralise le snapshot.
- Si le bridge owner utilise `float Speed`, toute valeur `double` finie mais non représentable en `float` neutralise les trois valeurs avant publication. Il n'existe plus de conversion vers `inf` ni de snapshot partiel.
- Le bridge reflected est résolu une fois sur la classe owner et n'est actif que si `Acceleration FVector`, `Speed float/double` et `IsMoving bool` sont tous présents. Il ne recherche ni composant SCS ni classe `ULIVECODING_*`.
- La neutralisation est publiée avant de vider les pointeurs reflected au teardown.
- Aucun gate dedicated-server n'a été ajouté, car `IsMoving` conserve un consommateur gameplay via le bruit de déplacement.

## Couverture QATS source

Le fichier contient exactement 15 définitions `QATS.GravityScape.*`. Aucune n'a été exécutée dans cette lane.

Couverture ajoutée ou renforcée :

- refus d'un setter directionnel concurrent pendant l'ownership J4, y compris depuis le delegate synchrone d'une écriture/restauration owned, sans révision ni fenêtre de bypass ;
- reprise d'un scale CMC brut modifié au prochain événement provider sous un contrat logique fixe ;
- notification provider identique ensuite réellement no-op ;
- ownership natif, reprise après mutation, libération au teardown et restauration exacte de `bIgnoreOtherGravityScales` dans deux configurations authored opposées ;
- tangent QAI déterministe lorsque source-forward et surface-up sont parallèles, avec refus/neutralisation d'un up nul, et surface unavailable dès la perte d'ownership J4 ;
- owner de test avec bridge reflected complet ;
- publication native/reflected du même snapshot ;
- seuil exact `1 cm/s` et valeur juste supérieure ;
- overflow d'accélération issu d'entrées finies ;
- vitesse `double` non représentable dans le bridge `float` ;
- récupération après entrée invalide puis neutralisation reflected au teardown.

### Exécution demandée au main Codex

Après build froid, exécuter d'abord ces tests ciblés :

1. `QATS.GravityScape.QangaGravity.Character.ScaleComposition`
2. `QATS.GravityScape.QangaGravity.Character.NativeDirectionAndScale`
3. `QATS.GravityScape.QangaGravity.Character.ReplicationConfigurationContract`
4. `QATS.GravityScape.NinjaGravityAlignment.Math`
5. `QATS.GravityScape.QangaGravity.Character.SurfaceQuery`
6. `QATS.GravityScape.QangaGravity.Locomotion.ComponentLifecycleAndSnapshot`

Puis exécuter le filtre complet `QATS.GravityScape`. Attendu avant fermeture du gate : exactement 15 tests découverts, 15 réussis, aucune erreur inattendue. Les erreurs de conflit volontairement injectées sont enregistrées avec `AddExpectedErrorPlain`.

## Gates compile/UHT demandés au main Codex

L'éditeur peut rester fermé pour les deux builds :

```powershell
& "E:\UE573\Engine\Build\BatchFiles\Build.bat" QangaEditor Win64 Development -Project="G:\QANGA\QANGA.uproject" -WaitMutex
& "E:\UE573\Engine\Build\BatchFiles\Build.bat" Qanga Linux Shipping -Project="G:\QANGA\QANGA.uproject" -WaitMutex -DisableUnity -NoPCH
```

Le premier build doit exercer UHT pour le character QATS reflected et vérifier les nouvelles frontières exportées Ninja/QAI entre modules. Le second ferme explicitement Linux, non-unity et l'indépendance aux PCH/IWYU avec les options reconnues par l'UBT UE 5.7 courant. Ne pas utiliser Live Coding pour ce gate reflété/inter-module.

Quand l'éditeur pourra être rouvert :

```powershell
& "E:\UE573\Engine\Binaries\Win64\UnrealEditor.exe" "G:\QANGA\QANGA.uproject" -AutoDeclinePackageRecovery
```

Compiler sans sauvegarde forcée `ALS_Base_CharacterBP`, `ALS_Base_CharacterBP_AILean` et leurs descendants directs concernés. Vérifier zéro UHT/reflection error, zéro `ULIVECODING_*` attendu, zéro pin/property SCS obsolète, puis exécuter QATS.

## Plan runtime `L_Dev_Rz`

Pour chaque rôle observé, relever au même instant : net mode, local/remote role, area gagnante, `ApplicationRevision`, résultat de `IsNativeGravityContractOwner`, `BaseGravityScale`, `GravityScale`, `GetFinalGravityScale()`, état directionnel Ninja capturé, `QuerySurfaceUp`, actor up, les quatre flags owned, état QPlatform, `Acceleration`, `Speed` et `IsMoving`.

### Standalone, 1 joueur

1. Charger `/Game/Maps/LevelDev/L_Dev_Rz` dans un éditeur froid.
2. Sur une zone point positive, rester immobile puis marcher autour de la courbure. Vérifier centre point exact, `GetFinalGravityScale() == GravityScale`, up acteur aligné avec `QuerySurfaceUp`, rotation fluide et absence de second snap immédiat.
3. Exercer une mutation de scale/direction/centre sur la même zone et les forced location/scale existants. Chaque contrat effectif doit produire une seule révision; une répétition identique ne doit rien écrire.
4. Traverser point, fixed, scale négatif, scale nul et absence de zone. Le fixed reste non signé dans Ninja; le négatif change le sens une seule fois; zéro/hors-zone retourne `DirectionlessZero` sans world-up périmé.
5. Entrer sur une QPlatform, observer alpha partiel, midpoint neutre, pleine intensité, puis detach/destroy. Vérifier composition par le propriétaire J4 et suppression immédiate de l'override exact.
6. Pour le joueur, vérifier idle, marche, sprint, saut et chute. `Speed` suit la norme 3D, les pics d'accélération restent finis, `IsMoving` pilote les consommateurs AnimBP/bruit sans `Accessed None`.
7. Spawner un enfant AILean réel, de préférence `AI_GuardCaptain_AILean`. Tester patrol/pursue/combat puis arrêt sur une surface courbe. Vérifier QAI seul writer, up valide maintenu à l'arrêt, heading stable et aucun twitch mesh/arme.
8. Provoquer uniquement avec un fixture PIE transitoire le cas forward parallèle à surface-up. Le cyborg doit choisir la tangente source-right et appliquer le nouvel up, pas sauter la frame.

### Listen Server, 2 joueurs, processus séparés

1. Configurer Listen Server, 2 joueurs, `Run Under One Process=false`.
2. Faire traverser au host et au client les mêmes transitions point/fixed/négatif/zéro/QPlatform.
3. Comparer serveur, owning client et simulated proxy. Direction, raw scale, final scale et surface query doivent converger; les trois rôles ont le provider J4 local, mais le simulated proxy ne produit pas le smooth joueur.
4. Vérifier l'owner natif exact, `bDisableGravityReplication=true` et `bIgnoreOtherGravityScales=true` sur chaque instance active. Aucun ancien multicast Ninja ne doit réécrire direction, scale ou align-base.
5. Faire rejoindre/reprendre possession au client alors que le pawn est déjà dans une zone, puis répéter une transition. Le premier contrat local doit être complet sans attendre un replay multicast.
6. Observer un cyborg AILean en déplacement/combat. Seul le serveur appelle la rotation QAI; le client reçoit la rotation acteur sans compensation locale.
7. Vérifier que le bruit `IsMoving` produit l'effet gameplay serveur attendu sans duplication client.

### Dedicated Server, 2 clients, processus séparés

1. Configurer Play As Client avec Dedicated Server, 2 clients, `Run Under One Process=false`.
2. Répéter les transitions gravité et QPlatform avec un client immédiat et un client reconnecté/tardif.
3. Sur le serveur meshless, vérifier que J4 et J6 restent actifs, que le final scale n'est pas multiplié par le physics volume et que le bruit de déplacement reste produit.
4. Sur les clients, vérifier calcul local J4, smooth uniquement sur l'owning pawn, rotation répliquée sur les simulated proxies et absence de correction Ninja tardive.
5. Spawner un AILean réel. Vérifier rotation QAI uniquement sur l'autorité serveur et parité visuelle des deux clients en patrol, pursue, combat et arrêt.
6. Détruire/respawner un pawn pendant une zone ou un override plateforme. Vérifier aucune callback après teardown, aucun override ressuscité et aucun log de dépendance détruite hors événement attendu.

### Critères de fermeture runtime

- Aucun `Accessed None`, `ULIVECODING_*`, erreur GravityScape/QAI inattendue ou spam supérieur à 1 log/s/source.
- Aucun double scale de physics volume, double inversion négative, actor-up fallback, double writer rotation ou correction proxy oscillante.
- `stop_pie` propre dans Standalone, Listen et Dedicated, avec vérification Message Log séparée pour serveur et clients.
- Aucun gate runtime fermé sur la seule base de valeurs CVar, d'un compile ou d'un QATS vert; le résultat player et AILean doit être visuellement inspectable.

## Requêtes cross-lane

1. Le propriétaire de `BLUEPRINT_TO_CPP_MIGRATION_PLAN.md` doit réconcilier ses mentions contradictoires `16/16` et `15/15` après l'exécution courante. La source contient 15 tests. Son état global et ses paragraphes J4/J6 ne doivent plus présenter la révision courante comme compilée/QATS verte avant ces nouveaux gates.
2. Dans ce même plan, remplacer après validation les faits périmés : J4 possède quatre flags, calcule localement aussi sur simulated proxy mais bloque les RPC/setters concurrents, et J6 n'utilise aucun cache de vélocité ni ordre non prouvé « avant Tick acteur ». Le callback est prouvé après `OnMovementUpdated` et le scoped movement update.
3. Le main Codex doit revalider les defaults authored : player `bSmoothAlignToGravity=true`; AILean `false` avec QAI seul writer; présence correcte de `UQGravityCharacterComponent` et de J6 là où consommé.
4. Aucun changement QModule, QPlatform, Build.cs, config ou asset n'est demandé par cet audit source. Leurs intégrations doivent seulement être exercées dans les scénarios ci-dessus.

## Vérifications statiques effectuées

- Recherche ciblée des writers, bridges `ULIVECODING_*`/SCS, component lookups, delegates et 15 noms de tests.
- Vérification des 15 implementations multicast gravité protégées par `bDisableGravityReplication` ou l'ownership natif ; les flags de rotation component hors ownership restent inchangés.
- `git diff --check` sur tous les fichiers owned : succès, uniquement les avertissements CRLF normaux du worktree Windows.
