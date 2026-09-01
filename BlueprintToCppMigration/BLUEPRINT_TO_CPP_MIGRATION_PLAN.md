# Migration Blueprint vers C++ — priorités

- **État :** deux premiers chantiers P0 terminés ; troisième chantier personnage/gravité avec cleanup meshless J3 et gate Linux Shipping terminés, ownership J4/J5/J6 et requête monde typée compilés, resolver QAI supprimé, `15/15` QATS verts et runtime local joueur/QPlatform revalidé à `360 FPS`, tandis que le nouveau build packagé, le réseau et runtime AILean restent ouverts ; QCameraControl terminé ; QSpringArm intégré, wrapper et plugin legacy supprimés, matrice visuelle élargie encore ouverte ; SwimmingDetection natif intégré, wrapper supprimé et runtime local vert, matrice réseau/dedicated encore ouverte ; le noyau Weapon (`11/11` QATS) reste dormant ; Inventory/QStorage ferme sa frontière source à `35/35` + `11/11`, avec CAS strict, sémantique bag/équipement et coordinateur durable ; DataManager fournit désormais le write/flush/readback/rollback borné partagé (`4/4`) et le tutoriel n'en duplique plus la logique (`5/5`) ; Combat est reparenté et dépouillé de son ancien owner Blueprint, `16/16` QATS et le runtime Standalone sont verts, tandis que les matrices réseau et packagées restent ouvertes ; le prochain gate d'intégration reste le premier adapter Inventory live, volontairement non commencé dans ce checkpoint
- **Projet :** QANGA, UE 5.7
- **Dernière vérification :** 2026-09-01

---

## 1. Verdict

`InventoryComponent` et `CombatComponent` doivent migrer vers C++, mais ils ne sont pas les premiers coûts CPU à traiter. Les priorités immédiates sont les chemins Blueprint exécutés à fréquence fixe ou multipliés par le nombre d'acteurs : détection de cibles véhicule, interaction joueur, simulation personnage/gravité et caméra.

La migration ne doit jamais être une traduction nœud-pour-nœud. Chaque chantier doit :

- établir un propriétaire natif unique ;
- corriger l'algorithme ou le modèle d'autorité ;
- conserver une façade Blueprint étroite uniquement pendant la transition ;
- supprimer le producteur Blueprint remplacé après validation ;
- échouer explicitement si le contrat requis n'est pas disponible.

## 2. Limite de l'audit

Le classement repose sur l'inspection live des graphes Blueprint, de leurs defaults et des appels C++ actuels. Aucun `.utrace` gameplay récent ne couvre un scénario représentatif personnage + véhicules + combat + inventaire. Les fréquences, les scans globaux et les traversées Blueprint/C++ sont prouvés ; les gains en millisecondes restent à mesurer avec Unreal Insights avant et après chaque migration.

## 3. Classement recommandé

| Priorité | Cible | Preuve actuelle | Frontière native attendue |
|---|---|---|---|
| **P0 — terminé** | Registre de cibles partagé + `VehicleCombatComponent` + `HomingLocker` | Le SAT et le scan global `O(targeters × combat actors)` ont été supprimés. Sept tests natifs, Standalone, Listen Server et Dedicated Server + deux clients sont verts. | `UQWeaponTargetRegistrySubsystem`, `IQWeaponTargetProvider` et `UQWeaponTargetingComponent` sont les propriétaires natifs uniques. Les façades BP restantes portent uniquement les règles et présentations encore vivantes. |
| **P0 — terminé** | `PlayerControllerInteractComponent` | `423 -> 211` nœuds ; l'ancien tick, les traces/sweeps, le tri à perte d'égalité, la macro d'occlusion, les variables dupliquées et le RPC Blueprint sont supprimés. Six tests natifs, Standalone, Listen, Dedicated + deux clients, un siège QTrain réel et un hold cyborg réel valident le noyau et ses deux branches métier les plus distinctes. | `UQPlayerControllerInteractComponent` est l'unique propriétaire de la détection, du focus et de la revalidation serveur. Le Blueprint enfant conserve uniquement input, sit, highlight, feedback et quête encore actifs. |
| **P0** | `ALS_Base_CharacterBP`, variante `AILean` et `GravityAreaComponent` | `4 454` / `4 251` nœuds ; `TickGraph` de `99` / `95` nœuds ; tick autorisé sur dedicated server. `GravityAreaComponent` effectue sa recherche à `10 Hz`. QAI doit réécrire la rotation après ALS. | Un propriétaire natif pour essential values, gravité, état mouvement et rotation. Migration verticale, puis suppression immédiate de chaque branche BP remplacée. |
| **P0/P1 — intégré** | `QSpringArm_Component`, `QCameraControl_Component`, `SwimmingDetection` | Les trois producteurs Blueprint sont supprimés. Caméra et spring arm sont limités au joueur local et exclus du dedicated server. Swimming conserve uniquement la force d'océan procédural WorldScape à `20 Hz`, sans réplication ni écriture ALS ; build froid, QATS `2/2`, huit maps reconstruites et runtime local sont verts. | `QSystem` possède les trois composants natifs. Ninja garde l'eau PhysicsVolume et la gravité, WorldScape garde la surface océan, ALS garde le movement state. Les matrices perceptuelles et réseau/dedicated restent des gates release. |
| **P1 — noyau source vert, intégration ouverte** | `WeaponScript` | `973` nœuds, EventGraph `266`. Le hitscan et le nouveau core cadence/ammo/reload sont natifs et leurs trois QATS passent. Les réservations de reload partielles ou surdimensionnées sont refusées et rollbackées exactement. Le NashV2 de référence, ses adapters contexte/ammo/presentation et les RPC authored restent Blueprint. | Intégrer un seul firearm de référence avec des adapters typés, puis supprimer son ancien owner cadence/ammo/reload/RPC après les gates Standalone/Listen/Dedicated. Réutiliser `QWeaponBulletSubsystem`. |
| **P1 — noyau source vert, intégration ouverte** | Noyau Inventory | `InventoryComponent` : `1 595` nœuds ; `Obj_ItemInstance` : `282` ; `Lib_Inventory` : `303` ; `ItemsManagerGS` : `547`. `QInventory` possède records, codec, validation, CAS endpoint, séparation bag/équipement, transfert atomique et journal durable à deux endpoints (`35/35` QATS). QStorage conserve les générations exactes, échoue fermé sur snapshot incomplet et draine explicitement les écritures (`11/11`). DataManager expose le bridge durable TempDB borné et vérifié, partagé avec le tutoriel. Aucun caller Inventory production n'utilise encore le noyau. | Poser le premier adapter Inventory live, résoudre les endpoints côté serveur, journaliser la paire, puis retirer le remove/recreate Blueprint uniquement après restart et réseau. `InventoryComponent_C` reste une façade temporaire. |
| **P1 — intégré, Standalone validé** | Noyau Combat | `UQCombatComponent` possède vie, permissions, faction, funnel dégâts et snapshots. `CombatComponent_C` est reparenté, ses writers natifs remplacés sont supprimés, ses defaults CDO sont valides après cold restart et ses consommateurs directs utilisent le type natif. `16/16` QATS, `4706/4706` Blueprints et le smoke `L_Dev_Rz` Standalone sont verts. | Fermer Listen/Dedicated en process séparés, les comportements exactly-once drops/récompenses/quêtes et la parité Development/Shipping packagée. Les seams présentation restent consommateurs post-commit, jamais propriétaires de la vie. |
| **P2** | `ItemsManagerGS`, DataManager, projectiles legacy, véhicules/tourelles, quêtes isolées | Gros périmètres ou modèles réseau distincts ; plusieurs noyaux existent déjà en C++. | Migrer après stabilisation des contrats Item, Weapon et Combat. Ne pas recréer les subsystems natifs existants. |

## 4. Ordre interne des systèmes gameplay

### 4.1 Weapon

Le calcul hitscan, les dégâts directs et le tracer poolé appartiennent déjà à `QWeaponBulletSubsystem`. La migration concerne l'orchestration autour de ce noyau, pas un nouveau système balistique.

Avant la bascule, résoudre le conflit live de `/Game/Items/Weapons/ShotGun/IS_Shotgun` : le CDO contient `Damage=0`, la logique de phase écrit `Damage_0`, tandis que `WeaponScript.PreImplementFire` transmet `Damage` au subsystem natif.

Le premier vertical reste dans le module `QWeapon`, sans nouveau plugin, et porte un seul fusil de référence déjà exercé par QATS. Un composant natif devient propriétaire de la cadence par deadline timestampée, des validations authority/owner/equipped/reload/fire-block, des munitions, du reload, du transform `FireLocation` et de l'appel unique à `ServerFireBullet`. `QWeaponBulletSubsystem` garde trace, filtres Combat, dégâts et tracer ; `UQWeaponAnimInstance` garde montage, notifies et présentation.

QAI fournit une référence de cadence native, mais son chemin direct est AI-only et ne couvre pas le contrat ammo/reload joueur. La statistique d'arme équipée exige aussi un adaptateur explicite vers l'instance item : `QModuleItemRack::GetStat` ne peut pas être remplacé par une lecture actor-only susceptible de viser le mauvais rack. Cette tranche ne doit donc pas absorber Inventory ou Combat.

Avant implémentation, relever live les signatures et flags de `Combat_1stTrigger`, `TriggerFire`, `SV_TriggerFire`, `PreImplementFire`, `TraceTest` et `GetInfiniteAmmo`, le point exact de décrément/refill/interruption, la source validée de `FireLocation` et les RPC/FX existants. Les QATS couvrent limite de cadence, refus serveur, ammo/reload, transform et damage exacts, puis Standalone/Listen/Dedicated + deux clients avec un seul dégât et un seul tracer. Les délais, traces, RPC et variables Blueprint ne sont supprimés que dans le fusil validé ; la base `WeaponScript` survit jusqu'à migration de tous ses enfants melee/AI/véhicules.

### 4.2 Inventory

Ne pas réécrire `InventoryComponent_C` en bloc. Première verticale :

1. record item natif avec identité stable, `ItemDataKey`, stack, rarity, owner, slot, attachments, customization et validité ;
2. codec versionné distinguant explicitement inventaire persistant et inventaire transient ;
3. funnel autoritaire unique pour add/remove/consume/equip ;
4. transfert atomique entre inventaires avec rollback ;
5. remplacement progressif des bridges par réflexion.

`Lib_Inventory.TryMoveItemToAnotherInventory` est le meilleur premier cas transactionnel : le chemin actuel retire l'item de la source, le recrée par ID, l'ajoute à la cible puis réécrit la persistance. La migration doit rendre cette opération atomique et garantir absence de perte ou duplication.

QStorage n'est pas encore l'autorité Inventory et ne doit pas le devenir par opportunité. `FQST_ItemStack` ne porte pas l'identité d'inventaire, les attachments/customizations typés ni le mode persistent/transient requis. Son chemin d'identité builder conserve en outre un fallback ordinal legacy côté client, à l'origine ou après refus de création ; en mode activé, ces cas doivent refuser explicitement au lieu de relier silencieusement un container à une identité instable.

Avant de brancher la première transaction, durcir ses frontières de persistance :

- `UnregisterLiveContainer` doit vérifier que l'acteur qui se désenregistre est encore la valeur enregistrée pour l'ancre ;
- la mutation de contenu utilisée par Inventory doit recevoir la version attendue et valider quantité, slot, clé et unicité d'instance ;
- le codec doit appliquer le même validateur sémantique et rendre un segment malformé read-only ;
- les flushes d'un segment doivent être sérialisés par génération, réarmer le dirty sur échec et ordonner proprement le shutdown.

Les bridges actuels ne sont pas des primitives de transaction : `QModule::ConsumeOne` et `GrantItemAsset` déclarent leur succès sans résultat typé ni vérification d'état, et le remboursement recrée une instance sans préserver son payload. `NPCDialogueObjective` écrit aussi `Stack` directement puis invoque `RemoveItemFromInventory` par une struct réfléchie partielle. Ces chemins devront tous rejoindre le funnel natif avant suppression de leurs scanners de réflexion.

Gate de la première verticale : transfert de la même instance et de tout son record, versions des deux endpoints, validations capacité/ownership/quantité, ordre déterministe, postconditions et rollback de snapshots. Les QATS couvrent succès, cible pleine, version stale, source égale cible, duplicats, échec add/remove, concurrence, redémarrage persistant et topologies Standalone/Listen/Dedicated + deux clients. L'implémentation attend la lecture live de la signature et des flags réseau exacts de `TryMoveItemToAnotherInventory`, de l'identité réelle des inventaires et du comportement stack-merge/full-target.

### 4.3 Combat

Commencer par un contrat natif consommable par QWeapon, puis migrer :

- `IsAlive`, vie courante et transition de mort ;
- faction et règle de ciblage ;
- activation PvP/PvE et safe areas ;
- funnel autoritaire d'acceptation des dégâts.

Conserver temporairement `ApplyPointDamage` comme entrée Unreal. Ne pas passer par `Lib_Combat.ClientRequestDamage` pour le chemin serveur : le flux actuel peut ne rien exécuter pour une IA sur dedicated server.

La matrice `AreFactionsHostile` n'est qu'une relation pure : le propriétaire Combat doit aussi préserver wanted, PvP/PvE, safe areas, tags inclusifs et exceptions player-owned/police. Les états safe-area `Unavailable` et `Protected` restent fail-closed. QWeapon, QAI et QModule conservent `ApplyPointDamage` comme entrée autoritaire commune pendant la transition ; le chemin IA Standalone et le chemin réseau Listen/Dedicated ne sont actuellement pas équivalents et doivent être testés séparément.

QAI dépend encore des contrats Blueprint `OnDamaged`, `OnDeath`, `LastDamageCauser`, vie, faction et `SetLifeWhenNoStat`. Son acquisition de cible s'exécute dans un `ParallelFor` : la future frontière Combat doit publier un snapshot natif immuable, jamais appeler `ProcessEvent` ni traverser un UObject mutable depuis un worker. `QWeaponTargetingComponent` reste le propriétaire de son registre et de son lifecycle ciblé ; Combat implémente son provider exact puis l'ancien `ActorTarget` Blueprint disparaît.

Gate de la première verticale : transition vie/dégâts/mort exactement une fois, autorité et réplication Listen/Dedicated, équivalence des trois funnels de dégâts, permissions complètes, safe areas fail-closed, IA dédiée, réactions/spawner, snapshots worker-safe et lifecycle QWeapon. Les signatures/flags live de `CombatComponent`, `OnDamaged`, `OnDeath`, `CheckTargetIsAllowedCombat`, `ClientRequestDamage` et les implémenteurs Blueprint des interfaces QPolice doivent être capturés avant d'écrire ou supprimer leur adaptateur.

## 5. Ce qui reste Blueprint

- DataAssets, presets, loadouts et tuning designer ;
- meshes, sockets, animations, recoil, audio et VFX ;
- widgets Inventory/Trade/Storage, highlights et feedback d'interaction ;
- drops, récompenses et présentation de la mort ;
- petites actions/objectives event-driven ;
- comportement visuel des projectiles tant qu'un contrat projectile server-authoritative complet n'existe pas.

## 6. Gates communs de validation

Chaque migration doit fournir :

1. une trace de référence reproductible avant modification ;
2. la même trace après modification, avec baisse mesurable du Blueprint VM, des scans ou des traversées `ProcessEvent` ciblées ;
3. validation Standalone, Listen Server, Dedicated Server et deux clients selon le système ;
4. aucun second propriétaire, aucune branche Blueprint désactivée mais encore produite ;
5. suppression du code, des graphes, des tâches et des données devenus morts ;
6. logs de panne explicites et limités à `1/s` par source.

## 7. Chantier lancé

Le premier chantier retenu est l'acquisition de cible d'arme partagée par `VehicleCombatComponent` et `HomingLocker`.

Son architecture, son déroulé d'implémentation et ses critères de sortie sont maintenus séparément dans [`01_WEAPON_TARGET_ACQUISITION_MIGRATION.md`](01_WEAPON_TARGET_ACQUISITION_MIGRATION.md).

Point final : le registre spatial, le provider `CombatComponent` et les migrations de `VehicleCombatComponent` et `HomingLocker` sont installés. Le SAT, l'ancien registre manager, l'état Blueprint remplacé, les anciens RPC de cible et les bridges réfléchis ciblés sont supprimés. Le build disque UE 5.7, les sept tests `QWeapon.Targeting.*`, le runtime Standalone, le Listen Server et le Dedicated Server avec deux clients sont verts. La preuve réseau couvre ownership, réplication, commit client confirmé, refus serveur sans mutation, multicast targeted-by, déduplication et cleanup après destruction. Le détail reste dans le document dédié.

## 8. Deuxième chantier — état actuel

Le hotspot `PlayerControllerInteractComponent` est basculé dans le module existant `QSystem`, sans nouveau plugin. Le parent live est natif, le Blueprint est réduit de `423` à `211` nœuds, les anciens producteurs de détection sont supprimés, les six tests natifs et les topologies Standalone/Listen/Dedicated + deux clients sont verts. Le cleanup de `Qanga_InputsComponent` et `StandDeTir_Start` est exécuté.

Les deux branches les plus distinctes ont maintenant une preuve runtime réelle dans `L_Dev_Rz` : détection, press/release, attache puis sortie sur `BP_QTrainSeat`, et hold `3 s` avec custom target sur `DEFAULT_Cyborg_V2`. Les deux arrêts PIE ont retourné un Message Log à `0` erreur et `0` warning. La matrice métier étendue et le gate cooked après le prochain scan/cook du propriétaire du build restent des gates de release, pas du code Blueprint remplacé à conserver. Le détail est maintenu dans [`02_PLAYER_INTERACTION_MIGRATION.md`](02_PLAYER_INTERACTION_MIGRATION.md).

RzDirectMCP sait maintenant lire le vrai modèle en mémoire du Message Log. `stop_pie` attend la fin synchrone de PIE et retourne automatiquement la page `PIE`, ses sévérités, identifiants, textes et tokens ; le dernier test réseau a ainsi isolé `772` erreurs Blueprint préexistantes sans aucune entrée liée au composant interaction.

## 9. Troisième chantier — état actuel

Le chantier personnage/gravité commence par le provider partagé, pas par une traduction du Blueprint ALS complet. Au baseline, `GravityAreaComponent` exécutait une résolution Blueprint à `10 Hz` pour `189` referencers et `LevelGravityArea` en possédait `206`. Cette frontière est maintenant native et les wrappers Blueprint sont vidés ; la fermeture J3 des consommateurs reste requise avant de modifier ou supprimer les `314` / `307` nœuds de `UpdateNinjaGravityDirection` dans les assets.

Le module existant `GravityScape` reste le propriétaire natif du domaine, sans création d'un plugin concurrent. Le durcissement J1 et le provider J2 (`AQLevelGravityArea` / `UQGravityAreaComponent`) ont passé le build disque UE 5.7 et les six tests `QATS.GravityScape.QangaGravity.*`.

J3 a supprimé les deux enums Blueprint après zéro référence, reparenté et vidé `LevelGravityArea` et `GravityAreaComponent`, réparé le manager et rafraîchi `31` consommateurs. Les wrappers conservent leurs chemins sérialisés mais plus aucun second producteur Blueprint. `GlobalGravityAreaManager` et `Lib_GravityArea` utilisent maintenant la base native dans leurs variables, signatures et graphes. FlyVehicle s'abonne directement aux événements register/update/unregister du registre GravityScape et synchronise l'état existant sans polling. Le header public du producer déclare aussi explicitement son subsystem : la compilation non-unity Linux Shipping ne dépend plus de l'ordre accidentel des unity includes. Après réouverture, la seed EasyCook a été vidée et régénérée sans scan. Les trois consommateurs révélés par le premier PIE sont réparés : `CM_SpeedShake` et `SpaceshipBase` par reconstruction native, puis `QPlayerPlatform_Component` par reconnexion exacte de `CachedGravityLocation` vers le pin natif `ForcedLocation` avant retrait de l'ancien pin sérialisé. `BP_Missile` n'expose plus non plus son cache gravité en `LevelGravityArea_C` : son contrat utilise maintenant la base native de bout en bout.

Les composants natifs migrés utilisent désormais un contrat central `Transient + SkipSerialization` et un `PostLoadSubobjects` strict. Les anciens sous-objets sérialisés dans les maps sont reconnus par owner, nom, classe et identité de default subobject, leur ancien `CreationMethod` SCS est normalisé en `Native`, puis propriétés, root et attaches sont restaurés sans resave de niveaux. RzDirectMCP génère ce contrat pour les migrations suivantes, qualifie les membres générés contre les collisions de noms, le valide avant le premier reparent/refresh et ne répare qu'une référence nulle ou l'archetype exact du CDO parent. Le compteur n'est publié qu'après succès atomique complet et aucun actor ou package de map chargé n'est modifié.

Le checkpoint compile `LevelGravityArea` et les six consommateurs réparés à `0` erreur / `0` warning. `L_Dev_Rz` charge, survit au refresh/reinstancing puis au PIE ; `stop_pie` retourne `0` erreur et `0` warning dans le Message Log. Le mode authored `CustomShape` mort est retiré. Les `21` QLevel concernés ont régénéré leurs `LOptimised` depuis leurs sources, l'entrée seed exacte a été retirée sans rescan, l'audit est tombé de `22` à zéro referencer toutes catégories et `SM_CylinderCollision` a été supprimé puis confirmé absent. Le gate contenu meshless J3 est fermé et la target `Qanga Linux Shipping` compile désormais le runtime régénéré en non-unity sans réintroduire la forme ni son produit supprimés. Le détail est maintenu dans [`03_CHARACTER_GRAVITY_MIGRATION.md`](03_CHARACTER_GRAVITY_MIGRATION.md).

La validation de régression post-checkpoint a fermé les chemins présentation et input qui dépendaient implicitement de cette migration. La cible du Down reste propriétaire du Up malgré une perte de focus/possession ; les événements non-hold sont stateless et répétables, tandis qu'un hold seul conserve sa paire cible/id exacte. `VehicleBase` annule un hold interrompu sans RPC et libère un hold terminé avec l'id réellement envoyé. Le prompt initialise ses bornes avec un tracker d'interaction seul, et la minimap reçoit ses références avant `Init` avec six canvases distincts. Dans `L_Dev_Rz`, la validation manuelle confirme les boutons répétés, les holds annulés/terminés/répétés, la sortie véhicule, le prompt projeté et les icônes minimap correctement réparties.

Le composant gravité active désormais son tick natif au `BeginPlay` même lorsque les anciens templates l'ont sérialisé inactif. La sortie/rentrée dans l'atmosphère Earth authored, située à environ `70,7 km` au-dessus du rayon WorldScape, reste stable sans seuil HUD ni scheduler de secours ; l'état atmosphérique ne persiste plus à `110 km`. Les gardes conservent aussi leurs armes dans ce checkpoint validé par le propriétaire du projet.

La fermeture a aussi livré la notification sur changement du contrat effectif d'une même zone, le teardown QATS explicite des actors démarrés manuellement, et un refresh RzDirectMCP atomique : refus des packages dirty/transactions étrangères, transaction identifiée sur les graphes, compilation, finalisation vérifiée avant sauvegarde, puis undo exact et recompilation de l'état restauré au moindre échec. Ce refresh sait également élargir de façon atomique les variables, signatures et pins object child-vers-base, y compris les valeurs de map. La commande RzMCP `QUIT_EDITOR` est reçue hors game thread puis attend la disparition d'un éventuel modal lors du post-tick Slate. Elle programme ensuite un callback game-thread one-shot hors callback Slate et appelle directement `IMainFrameModule::RequestCloseEditor`; ce propriétaire conserve les décisions de packages dirty et n'émet le vrai `QUIT_EDITOR` qu'après acceptation. Cette route évite aussi que PIE ne consomme une commande différée via `ULocalPlayer::Exec_Editor`. L'identité et le schéma sont stricts, et aucune requête de quit ne survit à un échec de réponse ou au redémarrage du bridge. La politique dedicated reste meshless : le mode `CustomShape` sans producteur a été supprimé à la source, et ses produits QLevel historiques sont régénérés depuis leurs sources plutôt que masqués par un guard ou réintégrés au cook serveur.

La frontière source J4 est intégrée sur les deux bases ALS. `UQGravityCharacterComponent` consomme le provider par événement, capture la base et l'état directionnel Ninja authored une seule fois et possède le composite `base × zone signée × coussin`. Le contrat refuse les valeurs non finies ou non représentables, conserve Ninja comme propriétaire de l'inversion négative, ne réécrit pas les simulated proxies et supprime les applications identiques. Il possède désormais au runtime les trois exigences nécessaires à l'écriture directionnelle, reprend toute mutation concurrente avant l'application, puis restaure direction, échelle et flags exacts au teardown avec la réplication en dernier. Les propriétés de rotation restent aux propriétaires J5 distincts. Après un setter direction Ninja, l'applicateur revérifie aussi activation, identité des sources faibles et révision : un listener qui le détruit ou applique un contrat plus récent ne peut plus laisser l'ancien appel réécrire l'échelle. Les QATS couvrent calcul, point/fixe/forced/zéro, ownership, mutation externe, destruction et restauration ; la passe complète compilée est verte à `15/15`.

La source J5 est compilée et chargée avec son export inter-module GravityScape disponible pour QAI et QATS. L'intégration authored est appliquée aux deux bases ALS : composant personnage présent, alignement smooth uniquement sur le joueur et propriété d'alignement capsule explicitement désactivée sur AILean pour laisser QAI seul propriétaire de sa rotation. Une réintroduction partielle de l'ancien writer dans l'asset joueur a été retirée transactionnellement : cache Tick, binding composant, fonction, appels et six variables exclusives ont disparu, tandis que les sorties de mouvement restent reliées directement à leur logique aval. Les `22` référents directs recompilent sans diagnostic. Le probe d'un enfant avait prouvé que les deltas de CDO descendants survivent volontairement à la propagation des defaults parents ; la solution n'est donc ni un setter RzDirectMCP spécial ni douze patches enfants. L'applicateur prend maintenant possession du contrat Ninja requis au runtime et le restitue exactement. Le stress joueur courant couvre cinq attach/detach sur un pickup à `360 FPS`, sans mutation externe ni Message Log ; un enfant AILean en déplacement/combat et un nouveau build packagé restent à revalider.

J6 conserve un composant locomotion non-ticking dans `GravityScape`, déclenché par `OnCharacterMovementUpdated` après le mouvement et avant le Tick acteur. La source compilée Win64 et `Qanga Linux Shipping` possède uniquement les trois produits vivants : `Acceleration=(CurrentVelocity-OldVelocity)/DeltaSeconds`, vitesse 3D et `IsMoving` au seuil de `1 cm/s`. Un bridge cached publie ces valeurs dans les variables Blueprint historiques sans autoriser de snapshot partiel ; les entrées invalides publient un état neutre. Les sorties quaternion/gravity-space/input sans consommateur et leurs tests ont été supprimés. Le composant est présent sur `ALS_Base_CharacterBP` et le cleanup atomique a retiré les trois getters/setters intermédiaires, le cache `PreviousVelocity`, `CalculateAcceleration` et leurs nœuds morts. La base et ses douze descendants legacy compilent à `0` erreur / `0` warning. AILean reste inchangé parce que son îlot Tick est mort.

La session PIE antérieure avec une classe `ULIVECODING_*` n'est pas retenue comme preuve : le bridge final ne lit plus cette propriété SCS et le batch authored a supprimé ses anciens lecteurs. Le build froid complet relie QATS, l'éditeur redémarré charge exactement les `15` tests présents dans la source et la passe `QATS.GravityScape` est verte à `15/15`. Le test locomotion couvre le premier callback, le cache de vélocité, le callback suivant et un sous-step, les entrées invalides et le teardown. Le stress PIE joueur/QPlatform courant ferme le gate local attach/detach ; locomotion AILean, réseau et confirmation dans un nouveau build packagé restent ouverts.

## 10. Quatrième chantier — état actuel

La migration `QCameraControl_Component` reste dans le module existant `QSystem`, sans nouveau plugin. `UQCameraControlComponent` remplace le composant BP qui alignait la rotation world du CineCamera sur `MakeFromXZ(ControlRotation.Vector, ActorUpVector)`. Le booléen custom et son setter ont été retirés au profit du contrat moteur `SetActive/Activate` : inactif signifie zéro tick, et le dedicated server est exclu avant l'enregistrement du tick. L'intention active survit à un transfert de possession tandis que le tick est coupé par événement ; seul un `PlayerController` local peut le reprendre, ce qui exclut aussi les AI standalone que l'Engine classe comme locales. Les trois chemins Blueprint vivants utilisent `SetActive`, et le nœud SCS ALS emploie directement la classe native. `OnRegister` neutralise les anciens defaults sérialisés. Le build froid, le QATS ciblé, la compilation ALS, le chargement sans erreur des huit maps historiques et le runtime `L_Dev_Rz` sont verts. Le writer spring-arm de troisième personne reste exclusif : il ne règle la rotation que lorsque le composant natif vient d'être désactivé. Après zéro référence Asset Registry et EasyCook, le wrapper vide a été supprimé ; le `ClassRedirect` tracked reste pour les copies historiques non versionnées. AILean n'utilise pas ce composant. Le détail est maintenu dans [`04_CAMERA_CONTROL_MIGRATION.md`](04_CAMERA_CONTROL_MIGRATION.md).

La tranche `QSpringArm_Component` est maintenant native dans `QSystem`. `UQSpringArmComponent` est limité au `PlayerController` local, ne ticke pas sur dedicated server, exécute ses `24` sweeps de moustaches à `5 Hz` par deadline polled et publie chaque frame le dispatcher consommé par ALS et le drone. Les calculs et branches de debug sans consommateur n'ont pas été transcrits ; `InterpCameraArmLength` reste dans ALS car cette timeline pilote le blend de view mode `3rdPersonCameraAlpha`, pas la longueur du spring arm.

Après retypage des six derniers nœuds ALS, le wrapper est tombé à zéro referencer et a été supprimé avec un redirect historique. Le package script, les classes exportées et le plugin `Cy_ASpringArm` sont également à zéro dépendance/referencer ; le plugin entier a été supprimé. Le build froid, le QATS ciblé, les quatre Blueprints consommateurs, les defaults `Length=210` / `CameraOffset=62`, la réponse caméra, le crouch, l'activation et les bindings ALS/drone sont verts dans `L_Dev_Rz`, avec deux Message Logs frais à `0/0`. La matrice perceptuelle collision/scope/véhicule reste à rejouer avant le gate release final. Le détail est maintenu dans [`05_SPRING_ARM_MIGRATION.md`](05_SPRING_ARM_MIGRATION.md).

## 11. Sixième chantier — état actuel

`SwimmingDetection` est maintenant un composant natif `QSystem` limité au joueur local. Il conserve
uniquement la force d'océan procédural WorldScape à `20 Hz`, lit la gravité Ninja et l'état ALS sans
les réécrire, ne réplique rien et ne ticke ni sur AI, proxy distant, pawn non possédé ni dedicated
server. Le wrapper a été vidé, le composant ALS reclassé sans perdre son identité SCS, le doublon
propre à `AI_BaseCharacter` supprimé, puis les huit instances de maps historiques reconstruites et
sauvegardées avec la classe native exacte. Après retrait de son unique entrée EasyCook, le wrapper
avait zéro referencer ; un redirect UTF-16LE byte-preserving a été ajouté et prouvé idempotent avant
suppression de l'asset.

Le build froid, les quatre Blueprints affectés et `QATS.QSystem.SwimmingDetection.*` `2/2` sont verts.
Dans `L_Dev_Rz`, le joueur local possède exactement un composant enregistré, actif et tickant en
`TG_DuringPhysics` avec `TickInterval=0.05`, `bReplicates=false`,
`bStartWithTickEnabled=false` et `bAllowTickOnDedicatedServer=false`; le Message Log PIE est à zéro
warning/erreur. La matrice surface/profondeur/underground/zéro et gravité négative, ainsi que les
topologies Listen/Dedicated multi-process, restent ouvertes avant le gate release final. Le détail est
maintenu dans [`06_SWIMMING_DETECTION_MIGRATION.md`](06_SWIMMING_DETECTION_MIGRATION.md).

## 12. Noyaux Weapon, Inventory/QStorage et Combat — état actuel

Le core `QWeapon` possède la cadence, les transactions ammo/reload, le transform de tir et l'appel unique au sous-système balle. Son registre échantillonne désormais les providers hors itération de la map live, puis revalide identité stable et génération de topologie avant publication ; un poll identique ne publie plus un faux changement. Les suites `QWeapon.FireControl` et `QWeapon.Targeting` sont vertes à `3/3` et `7/7`. Le Blueprint firearm reste toutefois autoritaire tant que le premier adapter NashV2 n'est pas intégré et dépouillé de ses writers concurrents. L'audit exact-item confirme que cette bascule dépend du premier endpoint Inventory live : sans CAS magazine et réservation rollbackable des munitions, brancher les variables Blueprint actuelles créerait un deuxième owner.

`QInventory` est vert à `35/35` : transfert whole-record, rollback exact, relectures synchrones réentrantes, CAS strict avec préservation exacte sur refus, séparation bag/équipement et coordinateur de journal durable à deux endpoints. `QStorage` est vert à `11/11` avec suivi exact des générations, y compris la génération zéro, refus read-only des keysets incomplets, propagation des échecs de drain, poursuite des segments sains et abandon explicite des prepares non écrits au shutdown. Le coordinateur neutre `FQInventoryDurabilityCoordinator` persiste atomiquement les prepares, suit les résolutions ordonnées des deux endpoints, écrit les marqueurs commit/abort, et récupère les transactions après crash ou restart avec tri chronologique et nettoyage des fichiers `.tmp` orphelins. Le codec binaire valide magic, version, CRC32 et sémantique complète avant acceptation. DataManager possède maintenant le bridge natif borné qui encode, flush, recharge, compare et rollbacke exactement les rows TempDB ; le tutoriel l'utilise directement, avec `4/4` QATS DataManager et `5/5` contrats Offline Tutorial verts. L'audit live confirme qu'`InventoryItems` et les maps d'équipement sont deux ensembles distincts et qu'aucun delegate, save différé ou RepNotify existant ne couvre toutes les mutations. L'adapter devra donc maintenir un endpoint stable et remplacer chaque writer par étapes ; un sidecar passif est exclu.

`QCombat` est maintenant le propriétaire production. `/Game/Systems/Combat/CombatComponent` dérive de `UQCombatComponent`; l'ancien état vie/faction/mode/cible/provenance et ses mutations Blueprint sont supprimés, sans miroir writable. Les champs conservés couvrent seulement présentation de mort, drops, récompenses et notifications aval. Les adapters QAI, QPolice et QModule utilisent les contrats typés, le map/delegate de mort QAI porte `UQCombatComponent`, le CDO authoré charge `PvEAndPvP` avec un damage type serveur valide et l'AnimBP infecté cold-load sans binding stale. Le build froid, `QATS.QCombat.*` `16/16`, le sweep `/Game` `4706/4706` et le runtime `L_Dev_Rz` Standalone non-lethal/reset/kill/revive sont verts avec un Message Log propre. Listen/Dedicated en process séparés, les réactions gameplay exactly-once et les builds packagés restent explicitement ouverts.

Le checkpoint froid courant compile aussi Win64 Shipping et Linux Shipping. Les Blueprints intégrés caméra, spring arm, nage, gravité et interaction recompilent sans diagnostic, et un smoke PIE `L_Dev_Rz` se ferme avec un Message Log à zéro warning/erreur. Les wrappers supprimés caméra, spring arm et nage, ainsi que `SM_CylinderCollision`, sont absents après cold load et ne figurent plus dans la seed EasyCook. Cela ferme le checkpoint d'intégration existant sans prétendre fermer les matrices réseau, restart ou package encore listées dans les documents système.

L'audit du dernier build Development a aussi fermé les régressions directement attribuables à ce checkpoint. Le GameState possède désormais son manager gravité authored et la librairie ne le crée plus tardivement ; le writer gravité ALS réintroduit a été supprimé jusqu'à ses variables mortes. Les composants d'authoring Warp ne sont plus cuisinés dans les deux acteurs runtime, la lampe d'ascenseur hérite d'une mobilité compatible, les deux GameModes possèdent un HUD engine valide et le preset de personnalisation initial conserve son record mesh complet avant les couleurs. QAI n'applique plus les redirects terrestres au trafic spatial et route le cyborg vers sa classe gravity-aware ; le widget stock suit la possession par événement, les métadonnées monde QSystem restent volontairement sparse, et QGM n'exige un `GameInstance` que pour son enregistrement GI. L'ordre UI contient l'input Help, les succès WeaponCoordinator ne sont plus des warnings, les traces Quest routinières ne sont plus produites et le mapping shader loose n'est résolu que dans les contextes qui compilent réellement les shaders. L'application des CVars de scalabilité respecte désormais leur priorité active et le palier d'ombres authored ne contredit plus l'invariant VSM-only de la config système. Les nouvelles validations chargées sont vertes à GravityScape `15/15`, QModule `8/8`, QSystem `8/8`, RzDirectMCP data asset `2/2` et deux régressions QAI, toutes sans warning ; un nouveau package reste nécessaire pour fermer le gel toit-véhicule et confirmer l'absence des anciens diagnostics cooked.

## 13. Outillage RzDirectMCP livré avec la migration

Les opérations d'authoring nécessaires à cette migration ont été corrigées à la source dans
RzDirectMCP. Elles ne reposent plus sur des manipulations manuelles ou des états partiels :

- `find_in_blueprints` décode maintenant les données FiB sur une file asynchrone bornée et retourne
  explicitement un résultat paginé en attente ou une saturation, sans bloquer l'Editor dans une
  conversion synchrone non interruptible ;
- le reclassage d'un composant SCS conserve son identité, son template et sa hiérarchie, refuse les
  sélecteurs ambigus ou incompatibles, compile avant sauvegarde et vérifie l'undo exact en cas
  d'échec ; les batches de graphes appliquent le même contrat atomique pour la suppression de
  fonctions et variables remplacées ;
- les redirects générés par un reparent ne sont émis que si le membre natif cible existe réellement,
  et l'ajout d'un `ClassRedirect` préserve strictement l'encodage UTF-16LE et tous les octets existants
  du fichier de configuration ; `add_enum_redirect` offre le même contrat byte-preserving pour les
  `EnumRedirect` avec `ValueChanges` de paires d'énumérateurs, refuse les conflits de `OldName` et
  traite un redirect équivalent existant comme no-op idempotent ;
- les acteurs placés dont la construction a conservé un ancien composant peuvent être reconstruits
  dans une transaction bornée à la map demandée, avec validation des classes exactes, absence de
  restes live et rollback avant toute sauvegarde ;
- la seed EasyCook peut retirer une entrée de tableau par chemin de champ et valeur exacte avec un
  contrat de cardinalité, sans matérialiser toute la seed en JSON ni lancer de rescan ;
- les probes runtime exposent des propriétés réfléchies bornées et ne confondent plus `IsHiddenEd()`
  avec l'état caché d'un actor dans un game world ; les recherches de chaînes GC sont filtrables et
  paginées pour les audits de suppression ;
- `compile_project` remonte explicitement les réinstanciations de types réfléchis qui imposent un
  redémarrage avant PIE, `play_in_editor` refuse un Blueprint déjà en erreur avant de compiler les
  autres packages dirty, et `save_asset` retourne la cause exacte lorsqu'une sauvegarde est demandée
  pendant PIE/SIE ;
- les propriétés `TArray` des data assets acceptent maintenant un tableau JSON typé : chaque élément
  est converti par réflexion dans une copie temporaire et l'asset n'est remplacé qu'après validation
  complète. Les mauvais types conservent donc le tableau original au lieu d'échouer après une mutation
  partielle ou d'imposer un export-text manuel ;
- la conversion écran-vers-monde déprojette le rayon réel du viewport et choisit la surface la plus
  proche entre la collision Unreal et le terrain planétaire WorldScape. Elle retourne aussi la normale,
  la source et la distance du hit, sans imposer une collision proxy au terrain procédural.

La lecture du Message Log, son retour automatique par `stop_pie` et la fermeture propre par
`QUIT_EDITOR` étaient déjà intégrés dans RzDirectMCP avant ce checkpoint ; ils restent les primitives
de validation et de redémarrage utilisées par les documents système ci-dessus.
