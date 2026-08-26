# Migration Blueprint vers C++ — priorités

- **État :** deux premiers chantiers P0 terminés ; troisième chantier personnage/gravité au checkpoint J3 runtime sain, gate dedicated meshless reporté
- **Projet :** QANGA, UE 5.7
- **Dernière vérification :** 2026-08-26

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
| **P0/P1** | `QSpringArm_Component`, `QCameraControl_Component`, `SwimmingDetection` | Spring arm : `326` nœuds dont `Update` à `117` nœuds avec traces. Caméra : tick chaque frame. Swimming : `20 Hz`. Les trois autorisent actuellement le dedicated server. QAI cite `6–12 ms` pour `100` cyborgs et coupe encore ces ticks par recherche de noms de classes/composants. | Spring arm natif limité au joueur local et absent du dedicated server, sans second propriétaire de rotation caméra. `SwimmingDetection` doit d'abord être comparé aux états eau/immersion déjà natifs du Ninja CMC : s'il ne fait que les republier, supprimer le Blueprint au lieu de recréer un détecteur. Tuning et présentation restent BP. |
| **P1** | `WeaponScript` | `973` nœuds, EventGraph `266`. Le hitscan est déjà natif, mais aim, cadence, ammo, reload et RPC restent BP. L'AnimInstance et le bullet subsystem traversent encore la réflexion BP. | Orchestrateur de tir typé : cadence timestampée, validation serveur, ammo/reload, transform de tir et événement cosmétique répliqué unique. Réutiliser `QWeaponBulletSubsystem`. |
| **P1** | Noyau Inventory | `InventoryComponent` : `1 591` nœuds ; `Obj_ItemInstance` : `282` ; `Lib_Inventory` : `303` ; `ItemsManagerGS` : `547`. Aucun propriétaire Inventory natif n'existe encore ; QStorage possède la persistance des containers, pas les transactions d'items. | Propriétaire Inventory neutre, record item versionné, validation à la frontière et mutation autoritaire atomique. `InventoryComponent_C` reste une façade temporaire ; QStorage reçoit les records par adaptateur. |
| **P1** | Noyau Combat | `CombatComponent` : `550` nœuds ; `Lib_Combat` : `171`, dont matrice de factions à `103` nœuds. Event-driven, donc moins urgent pour le CPU brut. Aucun propriétaire Combat natif ne couvre encore vie + permissions + faction + cible. | Vie, état alive/dead, faction, PvP/PvE, safe areas et acceptation des dégâts dans une API native typée, avec snapshots immuables pour QAI. Drops, récompenses et FX restent consommateurs BP. |
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

Son architecture, son déroulé d'implémentation et ses critères de sortie sont maintenus séparément dans [`01_WEAPON_TARGET_ACQUISITION_MIGRATION.md`](./01_WEAPON_TARGET_ACQUISITION_MIGRATION.md).

Point final : le registre spatial, le provider `CombatComponent` et les migrations de `VehicleCombatComponent` et `HomingLocker` sont installés. Le SAT, l'ancien registre manager, l'état Blueprint remplacé, les anciens RPC de cible et les bridges réfléchis ciblés sont supprimés. Le build disque UE 5.7, les sept tests `QWeapon.Targeting.*`, le runtime Standalone, le Listen Server et le Dedicated Server avec deux clients sont verts. La preuve réseau couvre ownership, réplication, commit client confirmé, refus serveur sans mutation, multicast targeted-by, déduplication et cleanup après destruction. Le détail reste dans le document dédié.

## 8. Deuxième chantier — état actuel

Le hotspot `PlayerControllerInteractComponent` est basculé dans le module existant `QSystem`, sans nouveau plugin. Le parent live est natif, le Blueprint est réduit de `423` à `211` nœuds, les anciens producteurs de détection sont supprimés, les six tests natifs et les topologies Standalone/Listen/Dedicated + deux clients sont verts. Le cleanup de `Qanga_InputsComponent` et `StandDeTir_Start` est exécuté.

Les deux branches les plus distinctes ont maintenant une preuve runtime réelle dans `L_Dev_Rz` : détection, press/release, attache puis sortie sur `BP_QTrainSeat`, et hold `3 s` avec custom target sur `DEFAULT_Cyborg_V2`. Les deux arrêts PIE ont retourné un Message Log à `0` erreur et `0` warning. La matrice métier étendue et le gate cooked après le prochain scan/cook du propriétaire du build restent des gates de release, pas du code Blueprint remplacé à conserver. Le détail est maintenu dans [`02_PLAYER_INTERACTION_MIGRATION.md`](./02_PLAYER_INTERACTION_MIGRATION.md).

RzDirectMCP sait maintenant lire le vrai modèle en mémoire du Message Log. `stop_pie` attend la fin synchrone de PIE et retourne automatiquement la page `PIE`, ses sévérités, identifiants, textes et tokens ; le dernier test réseau a ainsi isolé `772` erreurs Blueprint préexistantes sans aucune entrée liée au composant interaction.

## 9. Troisième chantier — état actuel

Le chantier personnage/gravité commence par le provider partagé, pas par une traduction du Blueprint ALS complet. Au baseline, `GravityAreaComponent` exécutait une résolution Blueprint à `10 Hz` pour `189` referencers et `LevelGravityArea` en possédait `206`. Cette frontière est maintenant native et les wrappers Blueprint sont vidés ; la fermeture J3 des consommateurs reste requise avant de modifier ou supprimer les `314` / `307` nœuds de `UpdateNinjaGravityDirection` dans les assets.

Le module existant `GravityScape` reste le propriétaire natif du domaine, sans création d'un plugin concurrent. Le durcissement J1 et le provider J2 (`AQLevelGravityArea` / `UQGravityAreaComponent`) ont passé le build disque UE 5.7 et les six tests `QATS.GravityScape.QangaGravity.*`.

J3 a supprimé les deux enums Blueprint après zéro référence, reparenté et vidé `LevelGravityArea` et `GravityAreaComponent`, réparé le manager et rafraîchi `31` consommateurs. Les wrappers conservent leurs chemins sérialisés mais plus aucun second producteur Blueprint. `GlobalGravityAreaManager` et `Lib_GravityArea` utilisent maintenant la base native dans leurs variables, signatures et graphes. FlyVehicle s'abonne directement aux événements register/update/unregister du registre GravityScape et synchronise l'état existant sans polling. Le header public du producer déclare aussi explicitement son subsystem : la compilation non-unity Linux Shipping ne dépend plus de l'ordre accidentel des unity includes. Après réouverture, la seed EasyCook a été vidée et régénérée sans scan. Les trois consommateurs révélés par le premier PIE sont réparés : `CM_SpeedShake` et `SpaceshipBase` par reconstruction native, puis `QPlayerPlatform_Component` par reconnexion exacte de `CachedGravityLocation` vers le pin natif `ForcedLocation` avant retrait de l'ancien pin sérialisé. `BP_Missile` n'expose plus non plus son cache gravité en `LevelGravityArea_C` : son contrat utilise maintenant la base native de bout en bout.

Les composants natifs migrés utilisent désormais un contrat central `Transient + SkipSerialization` et un `PostLoadSubobjects` strict. Les anciens sous-objets sérialisés dans les maps sont reconnus par owner, nom, classe et identité de default subobject, leur ancien `CreationMethod` SCS est normalisé en `Native`, puis propriétés, root et attaches sont restaurés sans resave de niveaux. RzDirectMCP génère ce contrat pour les migrations suivantes, qualifie les membres générés contre les collisions de noms, le valide avant le premier reparent/refresh et ne répare qu'une référence nulle ou l'archetype exact du CDO parent. Le compteur n'est publié qu'après succès atomique complet et aucun actor ou package de map chargé n'est modifié.

Le checkpoint compile `LevelGravityArea` et les six consommateurs réparés à `0` erreur / `0` warning. `L_Dev_Rz` charge, survit au refresh/reinstancing puis au PIE ; `stop_pie` retourne `0` erreur et `0` warning dans le Message Log. Les dix QATS gravité passent, dont la simulation explicite des composants historiques SCS, et le QATS codegen couvre les collisions de noms internes ; les targets `Qanga Win64 Shipping` / `Qanga Linux Shipping` lient. Le seul gate J3 restant porte sur la résolution dedicated meshless de `CustomShape`. Le détail est maintenu dans [`03_CHARACTER_GRAVITY_MIGRATION.md`](./03_CHARACTER_GRAVITY_MIGRATION.md).

La fermeture a aussi livré la notification sur changement du contrat effectif d'une même zone, le teardown QATS explicite des actors démarrés manuellement, et un refresh RzDirectMCP atomique : refus des packages dirty/transactions étrangères, transaction identifiée sur les graphes, compilation, finalisation vérifiée avant sauvegarde, puis undo exact et recompilation de l'état restauré au moindre échec. Ce refresh sait également élargir de façon atomique les variables, signatures et pins object child-vers-base, y compris les valeurs de map. La commande RzMCP `QUIT_EDITOR` est reçue hors game thread puis attend la disparition d'un éventuel modal lors du post-tick Slate. Elle programme ensuite un callback game-thread one-shot hors callback Slate et appelle directement `IMainFrameModule::RequestCloseEditor`; ce propriétaire conserve les décisions de packages dirty et n'émet le vrai `QUIT_EDITOR` qu'après acceptation. Cette route évite aussi que PIE ne consomme une commande différée via `ULocalPlayer::Exec_Editor`. L'identité et le schéma sont stricts, et aucune requête de quit ne survit à un échec de réponse ou au redémarrage du bridge. Un conflit dedicated reste ouvert sans modifier la politique meshless : `StaticMesh` est exclu du cook serveur tandis que `AQLevelGravityArea` charge encore un mesh pour `CustomShape`. L'usage authored sera audité live à la reprise ; le mode sera supprimé s'il est mort, sinon sa collision deviendra server-safe. Aucun guard silencieux et aucune réintégration globale des meshes serveur ne seront acceptés.

La frontière source J4 est également validée sans toucher encore aux assets ALS. `UQGravityCharacterComponent` consomme le provider par événement, capture la base Ninja authored une seule fois et possède le composite futur `base × zone signée × coussin`. Le contrat refuse les valeurs non finies ou non représentables, conserve Ninja comme propriétaire de l'inversion négative, ne réécrit pas les simulated proxies et supprime les applications identiques. La base et l'état directionnel Ninja exact sont capturés/restaurés ensemble, le provider notifie explicitement sa propre destruction, et l'applicateur refuse un CMC Ninja dont la réplication gravité est désactivée. Le teardown désactive et délie l'applicateur avant la restauration. Après un setter direction Ninja, l'applicateur revérifie aussi activation, identité des sources faibles et révision : un listener qui le détruit ou applique un contrat plus récent ne peut plus laisser l'ancien appel réécrire l'échelle. Les QATS couvrent le calcul, point/fixe/forced/zéro, destruction de l'applicateur et du provider, refresh pendant restauration et destruction depuis le callback direction lui-même. Le helper scratch world n'active aucun subsystem avant son `StartPlay` explicite et apparié à `EndPlay`. Le writer QModule de production reste inchangé tant que l'applicateur n'est pas authored : son ajout aux bases ALS, le switch du writer et la suppression des nœuds remplacés seront réalisés atomiquement à la reprise.
