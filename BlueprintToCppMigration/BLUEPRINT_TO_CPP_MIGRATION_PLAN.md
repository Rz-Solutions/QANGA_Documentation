# Migration Blueprint vers C++ — priorités

- **État :** premier chantier terminé ; deuxième chantier P0 audité et en cours
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
| **P0** | `PlayerControllerInteractComponent` | `423` nœuds ; à chaque frame : line trace complexe, puis multi-sphere ou multi-box trace et tri des candidats si le premier test échoue. | Trace, filtrage, score direction/distance et changement de cible en C++ avec échéance polled. Interfaces métier, highlight et UI restent BP. |
| **P0** | `ALS_Base_CharacterBP`, variante `AILean` et `GravityAreaComponent` | `4 454` / `4 251` nœuds ; `TickGraph` de `99` / `95` nœuds ; tick autorisé sur dedicated server. `GravityAreaComponent` effectue sa recherche à `10 Hz`. QAI doit réécrire la rotation après ALS. | Un propriétaire natif pour essential values, gravité, état mouvement et rotation. Migration verticale, puis suppression immédiate de chaque branche BP remplacée. |
| **P0/P1** | `QSpringArm_Component`, `QCameraControl_Component`, `SwimmingDetection` | Spring arm : `326` nœuds dont `Update` à `117` nœuds avec traces. Caméra : tick chaque frame. Swimming : `20 Hz`. Les trois autorisent actuellement le dedicated server. | Composants natifs activés uniquement dans le contexte qui consomme réellement la sortie, avec filtrage, hystérésis et deadlines. Tuning et présentation restent BP. |
| **P1** | `WeaponScript` | `973` nœuds, EventGraph `266`. Le hitscan est déjà natif, mais aim, cadence, ammo, reload et RPC restent BP. L'AnimInstance et le bullet subsystem traversent encore la réflexion BP. | Orchestrateur de tir typé : cadence timestampée, validation serveur, ammo/reload, transform de tir et événement cosmétique répliqué unique. Réutiliser `QWeaponBulletSubsystem`. |
| **P1** | Noyau Inventory | `InventoryComponent` : `1 591` nœuds ; `Obj_ItemInstance` : `282` ; `Lib_Inventory` : `303` ; `ItemsManagerGS` : `547`. Fort couplage réplication/persistance/équipement. | Record item versionné, codec, validation à la frontière et mutation autoritaire atomique. `InventoryComponent_C` reste une façade temporaire. |
| **P1** | Noyau Combat | `CombatComponent` : `550` nœuds ; `Lib_Combat` : `171`, dont matrice de factions à `103` nœuds. Event-driven, donc moins urgent pour le CPU brut. | Vie, état alive/dead, faction, PvP/PvE, safe areas et acceptation des dégâts dans une API native typée. Drops, récompenses et FX restent consommateurs BP. |
| **P2** | `ItemsManagerGS`, DataManager, projectiles legacy, véhicules/tourelles, quêtes isolées | Gros périmètres ou modèles réseau distincts ; plusieurs noyaux existent déjà en C++. | Migrer après stabilisation des contrats Item, Weapon et Combat. Ne pas recréer les subsystems natifs existants. |

## 4. Ordre interne des systèmes gameplay

### 4.1 Weapon

Le calcul hitscan, les dégâts directs et le tracer poolé appartiennent déjà à `QWeaponBulletSubsystem`. La migration concerne l'orchestration autour de ce noyau, pas un nouveau système balistique.

Avant la bascule, résoudre le conflit live de `/Game/Items/Weapons/ShotGun/IS_Shotgun` : le CDO contient `Damage=0`, la logique de phase écrit `Damage_0`, tandis que `WeaponScript.PreImplementFire` transmet `Damage` au subsystem natif.

### 4.2 Inventory

Ne pas réécrire `InventoryComponent_C` en bloc. Première verticale :

1. record item natif avec identité stable, `ItemDataKey`, stack, rarity, owner, slot, attachments, customization et validité ;
2. codec versionné distinguant explicitement inventaire persistant et inventaire transient ;
3. funnel autoritaire unique pour add/remove/consume/equip ;
4. transfert atomique entre inventaires avec rollback ;
5. remplacement progressif des bridges par réflexion.

`Lib_Inventory.TryMoveItemToAnotherInventory` est le meilleur premier cas transactionnel : le chemin actuel retire l'item de la source, le recrée par ID, l'ajoute à la cible puis réécrit la persistance. La migration doit rendre cette opération atomique et garantir absence de perte ou duplication.

QStorage n'est pas encore l'autorité Inventory. Son chemin d'identité builder conserve un fallback ordinal legacy lorsque le stockage est désactivé, exécuté côté client ou non résolu. Ce fallback doit disparaître avant toute dépendance autoritaire.

### 4.3 Combat

Commencer par un contrat natif consommable par QWeapon, puis migrer :

- `IsAlive`, vie courante et transition de mort ;
- faction et règle de ciblage ;
- activation PvP/PvE et safe areas ;
- funnel autoritaire d'acceptation des dégâts.

Conserver temporairement `ApplyPointDamage` comme entrée Unreal. Ne pas passer par `Lib_Combat.ClientRequestDamage` pour le chemin serveur : le flux actuel peut ne rien exécuter pour une IA sur dedicated server.

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

## 8. Chantier en cours

Le coût P0 actuel est `PlayerControllerInteractComponent`. Son audit live, son architecture, ses étapes de suppression et ses gates sont maintenus dans [`02_PLAYER_INTERACTION_MIGRATION.md`](./02_PLAYER_INTERACTION_MIGRATION.md). Le contrat est figé ; l'implémentation commence dans le module existant `QSystem`, sans nouveau plugin.
