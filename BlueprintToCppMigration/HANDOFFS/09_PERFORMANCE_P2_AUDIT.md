# Worker 09 - Performance evidence and P2 dependency audit

## Livrable

- Audit complet: `Documentation/BlueprintToCppMigration/10_PERFORMANCE_AND_P2_AUDIT.md`
- Révision auditée: `dc587c4530`
- Index Asset Registry/RzMCP: schéma 4, complet, 58 428 assets `/Game`, généré le 30 août 2026 à 15:17:16 UTC.
- Aucun code, asset, config, test, build file ou document existant modifié.
- Aucun build, Editor/PIE, cook, package, rescan, stage ou commit exécuté.

## Résultat principal

Il n'existe aucune baseline comparable représentant joueur + véhicules + combat + inventaire. Aucun delta runtime avant/après ne peut être revendiqué.

Les suppressions déjà documentées restent des preuves structurelles. Une valeur en ms ou en pourcentage serait inventée.

## Captures trouvées

Sept `.utrace` existent:

1. `PlanetScape_Mercury_Editor_Audit_20260820.utrace`, avec deux exports qui se chevauchent. Workload Editor/PlanetScape dominé par Slate/renderer.
2. `starmap_adaptive_runtime.utrace`, avec exports global, StarMap fermée et ouverte. Comparaison locale UI uniquement.
3. Deux snapshots `Cache_Local_Efficiency_*` sans export.
4. Trois snapshots `Memory_Memory_Pressure_*` sans export ni scénario gameplay stable; les noms ont été réutilisés/écrasés.

Le store global Unreal Trace contient zéro trace. Les traces projet vivent hors du store.

La trace StarMap n'active pas `Net`, `Object`, `Counters` ou `Task`. Elle montre seulement un `ReceiveTick` agrégé, sans timer ItemsManager/DataObject/projectile/véhicule/tourelle/Combat attribuable.

## Classement P2 exact

Ordre d'intégration après P1 et baseline, pas ordre de gain prétendu:

1. Missile runtime + flare manager.
2. Turret acquisition/fire consommant Weapon/Combat.
3. Reliquat gameplay `VehicleBase`/`FlyVehicleMovementComponent`.
4. Record item persistant vertical `Inventory -> DataManager`, world-drop uniquement.
5. `BP_Projectile` ballistic puis grenade, trace-gated et séparés.
6. Bulk `ItemsManagerGS`/DataManager: ne pas router sans hotspot nommé.
7. Quêtes isolées: rester Blueprint.

## Dépendances non négociables

- Missile et vehicle/turret fire consomment le fire control P1 Weapon lorsque leur modèle d'autorité correspond. Ils ne recréent ni cadence, ammo, RPC ni ownership.
- Impact, life, permission et death consomment le contrat P1 Combat. Les probes `CurrentLife`, `OnDamage`, `IsAlive` doivent disparaître à cette frontière.
- ItemsManager, DataManager, QModule et quêtes consomment le record/transaction P1 Inventory avec rollback. QStorage ne devient pas owner Inventory.
- `FlyVehicleMovement` reste l'unique owner physique. `QVehicles` ne possède aujourd'hui que la sirène.
- `DynamicQuestSystem` est déjà le core natif. Les objectifs isolés inspectés sont event-driven, sans Tick/Delay.

## Preuves structurelles majeures

- `MissileMovementComponent`: `ReceiveTick` par missile. `ProjectilesManager`: Tick + `CheckFlaresHit`.
- `BP_Missile`: event Tick présent mais déconnecté, donc pas de gain à compter pour ce nœud.
- `VehicleBase`: chaîne BP par frame vers collision rapide, camera collision, FOV, velocity, speed control et rollover.
- `FlyVehicleMovementComponent` BP: ancien Tick désactivable par le chemin natif, mais boucle latente `0.5 s` vers damage/underwater.
- `UVehicleMovementComponent`: owner C++ PrePhysics, dedicated autorisé, sommeil/parking déjà implémentés.
- `TurretBase`: loop timer `1 s`, `FindNearest` 53 nœuds, update target/aim/fire par tourelle.
- `ItemsManagerGS`: 547 nœuds et 20 référents, mais event-driven.
- `DataObject`: 1 085 nœuds, 68 fonctions, 72 référents; codec/parse de burst, pas steady-state prouvé.
- `O_PickUpObjective` et `O_ShopTransactionObjective`: 106/195 nœuds, event-driven, deux référents chacun via PrimaryAsset + seed.

## Commits de propriété à conserver dans le raisonnement

- `3bd272b02`: QWeapon devient owner du registre/ciblage spatial véhicule/homing; ancien scan global supprimé.
- `5721a9b6c`: type/cache de gravité grenade migré, mais pas la trajectoire/collision/autorité projectile.
- `93e38bd31`: conduite/physique déterministe renforcée dans `FlyVehicleMovement`.
- `3fc2f081d`: parking/optimisation runtime véhicule.
- `14d000eb2`: corrections de persistance item encore Blueprint.
- `311656ff7`: snapshot/restore QModule via bridge DataManager réfléchi.
- `5a3982a2a`: signal Inventory du premier transfert et objectifs natifs de quête raccordés.

## Scopes routables après P1

Le document principal contient quatre prompts copy-ready avec ownership disjoint:

- `P2-01`: nouveaux runtime/registry missile dans QWeapon + cinq assets missile/manager.
- `P2-02`: contrôleur tourelle QWeapon + `TurretBase` et ses deux children.
- `P2-03`: deux fichiers existants `FlyVehicleMovementComponent.h/.cpp` + les assets `FlyVehicleMovementComponent` et `VehicleBase`.
- `P2-04`: codec item record dans le module DataManager + `DataObject` et `ItemsManagerGS`, limité au round-trip world-drop.

`BP_Projectile`/grenade a seulement un scope de mesure. Aucune implémentation ne doit être routée avant self time et concurrence réels.

## Matrice de trace requise

Map commune: `/Game/Maps/LevelDev/L_Dev_Rz`.

- Integrated: Standalone puis Dedicated + 2 clients, parcours on-foot/vehicle/combat/inventory.
- Missile: 1/16/64 missiles, 0/8 flares, 32 targets.
- Turret: 1/16/64 tourelles, 64 combatants, MG et rocket séparés.
- Vehicle: 1/8/32 véhicules actifs, mêmes inputs et états.
- Item: 2 inventaires de 40 slots, 20 records, 100 moves, 10 world-drop round-trips, 1 save + restore.
- Ballistic: 1/16/64 instances, grenade séparée.

Canaux communs: `Cpu`, `Frame`, `Stats`, `Bookmark`, `Region`, `Counters`, `Object`, `Log`; ajouter `Task` pour persistance et `Net` pour réseau. Trois runs minimum, mêmes build/map/save/seed/counts/topology/channels, warmup hors région.

Comparer self time ciblé, médiane/p95/p99 GameThread/server tick, calls, actor/event counts et bytes/RPC. Si les counts fonctionnels divergent ou si le coût a seulement changé de nom natif, aucun gain n'est valide.

## Candidats de suppression prouvés, non supprimés

1. `/Game/VehicleWeapons/Rocket/Missile_VehicleRocketLauncher`: 0 référent Asset Registry, 0 référence C++/config/doc.
2. `/Game/_QData/Proto_Quest`: 0 référent, 0 référence texte, 77 nœuds déconnectés et 0 fonction/event.
3. `UDataManagerBPLibrary`: 0 référent `/Script/DataManager`, aucune fonction, seulement son constructeur.
4. Module runtime DataManager actuel: Startup/Shutdown vides, aucun module externe linké, contenu BP sans dépendance `/Script/DataManager`. Le réutiliser pour P2-04 ou convertir le plugin en content-only, pas le supprimer avec son contenu.

Le plugin DataManager complet est live: GameDataManager 26 référents, DataObject 72, DataManagerLib 52. Aucun redirect n'a une preuve suffisante de mort; ne rien retirer.

## Inconnues restantes

- concurrence/durée de vie/authority réelle des child projectiles;
- nombre production et proportion awake des véhicules/tourelles;
- gate C++ FlyVehicle effectif pour chaque child;
- cadence/volume/bytes de DataObject;
- self time nommé des scopes P2;
- fréquence réelle des objectifs pickup/shop;
- cold-load/cook final avant suppression des candidats.

## Terminal

L'audit ne bloque pas P1. Après intégration P1, capturer les baselines avant de router `P2-01`, puis respecter l'ordre et les gates ci-dessus. Ne pas migrer un graphe uniquement parce qu'il est gros.
