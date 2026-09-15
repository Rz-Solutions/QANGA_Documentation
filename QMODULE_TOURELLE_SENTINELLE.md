# Tourelle sentinelle : dossier de chantier (module actif QModule)

> Etat au 2026-08-31. Toutes les valeurs de ce document sont **mesurees dans l'editeur**
> (pont CLIScape, `SubobjectDataSubsystem` + `get_detailed_blueprint_summary`), pas deduites.
> Marqueurs de mesure : `QDIAG_KIT_*`, `QDIAG_TB_*`, `QDIAG_CH_*`, `QDIAG_LR_*`.
>
> Ce dossier est le cahier des charges anti-regression du chantier. Regle CLAUDE.md 10 :
> tout ce que l'existant fait, la nouvelle version doit continuer a le faire.

---

## 1. Etat du module aujourd'hui : coquille vide

`Content/Phases/QModuleV2/QMD_TourelleSentinelle.uasset` ne porte que son identite.

| Champ | Valeur reelle |
|---|---|
| ModuleTag | `Module.TourelleSentinelle` (enregistre, `Config/Tags/QModuleTags.ini:96`) |
| Domain | `EQModule_Domain::Cyborg` |
| SynergyTags | `Module.Family.Engineering` |
| ManufacturerTag | `Manufacturer.ICLab` |
| DisplayName | "Tourelle sentinelle" |
| Icon | `T_QMI_TourelleSentinelle` (existe) |
| StatMods, LevelDescriptions, AbilityGrants, BehaviorGrants, ItemDataAsset, MaxLevel | **absents** |

Recherche projet complete et terminee sur la chaine `TourelleSentinelle` : 4 occurrences
(le tag `.ini`, `QMODULE_CATALOGUE.md:1177`, `QMODULE_ACTIVATION_ALIGNMENT.md:195`, le
snapshot du build d'icones). **Zero ligne de code.** Pas de paire `IDA_`/`IS_` alors que
43 autres modules cyborg en ont une. Absente de `Config/EasyCookGenerated.ini`.

Elle passe `IsDefinitionValid` (qui n'exige que ModuleTag + Domain + MaxLevel), donc elle
s'affiche sur le Mur, s'installe et coute des phases sans rien faire.

---

## 2. La tourelle existante : cahier des charges a preserver

`Content/GameplayActors/Turret/TurretBase.uasset`, parent commun de trois enfants.
**Ne PAS modifier ce parent** : il porte les tourelles du monde ET la tourelle constructible
QBuilder du joueur. Toute evolution passe par un enfant.

### 2.1 Hierarchie mesuree

```
Scene (root)
+-- Base                     SM_Turret_Base          scale 1.0  (0.25 sur _Build)
|   +-- Rotator              SM_Turret_Platform      loc Z=412.7      <- axe YAW
|       +-- WeaponMesh       SM_Turret_Cannon        loc -109.1,0,163.7  <- axe PITCH
|           +-- WeaponProjectileLocation  loc 774.7,0,0 (944.7 sur les enfants)  <- LE muzzle
|           +-- WeaponCanonMachineGunL    loc 0,-110,0  rot yaw -90  scale 10
|           +-- WeaponCanonMachineGunR    loc 0,+110,0  rot yaw -90  scale 10
+-- CollisionSphere          r=500  loc 51.8,0,328   (r=125 loc 13,0,82 sur _Build)
+-- WV_DetectPlayer          r=100000 sur la base, r=50000 sur les 3 enfants
+-- CombatComponent, NameComponent, OptimizedStateComponent
```

**Les deux emplacements d'arme existent deja**, symetriques a Y = +/-110 sous `WeaponMesh`,
introduits par `Turret_MachineGun` (`SM_NASH56_CanonSniper`, scale 10) et herites par
`Turret_MachineGun_Build`. Ils sont **purement cosmetiques** : `Turret_MachineGun` n'ajoute
aucun graphe (`EventGraph` vide, `UserConstructionScript` = simple appel parent).

### 2.2 Ce que le code fait, etape par etape

**BeginPlay** : memorise `OriginalRotation` (rotation monde du `Rotator`), s'abonne a la cle
booleenne `Alive` de `OptimizedStateComponent`, lit le parametre global
`Turrets / RespawnS` (valeur par defaut **300 s**) dans `RespawnSecs`, puis `UpdateRespawn`.

> Piege mesure : le CDO porte `RespawnSecs = 600`, mais **BeginPlay l'ecrase** par le
> parametre global (300 par defaut). La valeur effective n'est pas celle du CDO.

**Detection** : `WV_DetectPlayer` (volume monde, `BPI_WorldVolume`) declenche
`OnEnterVolume` / `OnExitVolume`. A l'entree, `ActorIsDrivableVehicle` vrai range la cible
dans `PriorityTargets`, sinon dans `TargetsInside`. **Les vehicules sont prioritaires.**

**Choix de cible** (`FindNearest`, serveur uniquement) : ligne de vue par
`LineTraceByChannel` depuis `WeaponProjectileLocation`, filtre
`CombatComponent.CheckTargetIsAllowedCombat`, priorite aux vehicules, sinon le plus proche.
`UpdateTarget` se re-arme toutes les **2 s**.

**Visee** (`UpdateAim`) : `LoopTimer` de **1 s**, interpolation `RLerp` du `Rotator` vers
`FindLookAtRotation(muzzle -> PredictedLocation)`. `SetWeaponRotation` applique le **yaw** au
`Rotator` et le **pitch** au `WeaponMesh`, clamp **[-30, +60] degres**. Un
`DoOnceAutoReset` de **4 s** emet l'evenement multicast `Fire`. Sans cible, retour vers
`OriginalRotation`.
`TargetUnprecision` disperse la visee dans une boite de demi-taille 200 cm
(`RandomPointInBoundingBox`). Valeur par defaut **0.0** sur les trois enfants.

**Tir** (`FireWeapon`, sur l'evenement `Fire`) : une seule branche `Equal(Enum)` sur
`TurretWeapon`.

| Voie | Valeurs mesurees |
|---|---|
| `MachineGun` | `AutomaticLoopFireDelay` **0.5 s**, `FireStaticDefenseMachineGun` portee **50000 cm**, `TracerSizeScale` 4.0, son `PlayDefaultDistantRifleShotAtLocation` volume 0.9, puis `Delay` **2 s** avant la rafale suivante. Degats = `Select(ActorHasTag(cible,"Vehicle") ? **125** : **25**)` |
| `Missile` | `SpawnActor TurretMissileProjectile` : `TotalDamage` **100**, `MaxLifeSpan` **8 s**, `DamageRadius` **600 cm**, `HomingTarget` = racine de la cible. Si la cible a le tag `Spacecraft`, `AddTargetedByOthers` |

**Mort** : `CombatComponent.OnDeath` -> `Alive=false` en multicast -> evenement `Death`
(Niagara attache a `CollisionSphere`) -> `UpdateRespawn`.

**Respawn** : serveur, `RetriggerableDelay(RespawnSecs)` -> `ResetLifePoints` -> `Alive=true`.
`UpdateActorAlive` cache l'acteur et coupe sa collision tant qu'il est mort.

**Riposte** : `CombatComponent.OnDamaged`, serveur, si `IsSameFaction` est **faux**, prend
l'agresseur pour cible.

### 2.3 Valeurs par variante

| | TurretBase | Turret_MachineGun | Turret_MachineGun_Build | Turret_RocketLaucher |
|---|---|---|---|---|
| TurretWeapon | Missile | MachineGun | MachineGun | Missile |
| Faction | IC_LABS | IC_LABS | IC_LABS | IC_LABS |
| LifeWhenNoStat | 1500 | 1500 | 1500 | 1500 |
| RespawnSecs (CDO) | 600 | 600 | 600 | 600 |
| WV_DetectPlayer | 100000 | 50000 | 50000 | 50000 |
| CollisionSphere | 500 | 500 | **125** | 500 |
| Base scale | 1.0 | 1.0 | **0.25** | 1.0 |

La tourelle constructible du joueur est donc **la meme mecanique a un quart d'echelle**, et
elle est **de faction IC_LABS**, pas d'une faction propre au joueur.

### 2.4 Dettes constatees dans l'existant (a ne pas propager)

1. **La voie missile n'est pas server-authoritative.** `FireWeapon` teste
   `IsDedicatedServer` et ne prend la branche que si **faux** : le projectile est spawne
   cote client, et son proprietaire est choisi par `IsLocallyControlled` sur la cible. Le
   graphe porte deja le commentaire de l'auteur : "TODO fix the ownership of missile to
   damage". Sur serveur dedie, cette voie ne tire pas cote serveur.
2. **Un seul muzzle pour deux canons.** `WeaponProjectileLocation` est unique ; les canons L
   et R sont decoratifs.
3. **`FactionTag` est vide** sur le `CombatComponent` ; seule l'enum `Faction` est renseignee.

---

## 3. Le kit d'art fourni par RzZz

`Content/GameplayActors/Turret/Turret_Mesh/Turret_Mesh_Module_Cyborg.uasset`
(cree le 2026-08-31 a 15:56). Acteur BP nu, parent `Actor`, `EventGraph` entierement
desactive : c'est un **kit de pieces**, pas une tourelle.

```
DefaultSceneRoot
+-- SM_Turret    Socle/SM_Turret                 bounds origin 0,0,14    extent 72,72,14
    +-- SM_Turret1  Body/SM_Turret               bounds origin 0,0,71    extent 64,64,44
        +-- SM_Turret3  Gun/SM_Turret_MachineGun bounds origin -16,42,-8 extent 16,72,19
        |   +-- SM_Turret4  Loader/SM_Turret     bounds origin 0,0,0     extent 13,22,15
        +-- SM_Turret2  Eye/SM_Turret            bounds origin 0,3,0     extent 13,18,16
        +-- SM_Turret5  Missiles/SM_Turret_Missile bounds origin 30,0,0  extent 30,36,37
```

Tous les composants sont a la position relative **0,0,0** : les decalages sont **bakes dans
les pivots des meshes**. Consequence directe : la mitrailleuse est laterale a **X = -16** et
le pod de missiles a **X = +30**. Les deux points d'ancrage **ne sont pas symetriques**.

Chaque piece a deja son jeu d'instances de materiau (`Blackout`, `Metal_1..4`,
`Plastic_1..3`, plus `Led` et `Glass` sur l'oeil) : c'est le levier de l'effet de
materialisation au deploiement.

**Renommage effectue le 2026-08-31 a 16:24** : `SM_Turret_MacgineGun` -> `SM_Turret_MachineGun`.
3 referents suivis (`Turret_Mesh_Module_Cyborg`, `Turret_Mesh_BuildSAVE`,
`DA_EasyCookSeed_QANGA` en soft ref), aucun redirecteur orphelin, verifie au binaire sur
disque (0 occurrence de l'ancien nom).

---

## 4. La brique de livraison : deja ecrite

`AQModule_ThrownDeviceActor` (QModule, `UCLASS(Abstract, NotBlueprintable)`) est la base
partagee de tout actif lance : balise de frappe, balise de ravitaillement, balise de flotte,
grenades collantes. Elle fournit deja :

- une **parabole en forme close** evaluee des memes entrees repliquees sur serveur et
  clients, sans replication de mouvement par frame ; un client qui recoit l'acteur en vol
  reprend l'arc au bon endroit ;
- l'arc resolu dans le **repere local du lanceur**, jamais en Z monde (correct a l'echelle
  planetaire), avec des `FVector` pleins et non quantifies ;
- le **serveur seul** balaye la trajectoire, pose l'engin au premier blocage et replique
  `PlantedLocation` ; hook enfant `Authority_OnPlanted(Hit)` pour armer la charge ;
- helpers partages avec la preview : `SampleArc`, `SolveArcVelocity`, `ResolveFlightSeconds`
  (arc lisible de 5 m a 200 m).

### 4.1 Les deux origines de lancer, mesurees

| Famille | Formule | Sort de |
|---|---|---|
| Ordonnance dorsale (grenades collantes) | `ActorLocation + Up*95 - Forward*30 + Right*(+/-24)` (`QModule_RackComponent.cpp:3348`) | **le dos**, tubes alternes |
| Balises | `ActorLocation + Up*70 + Forward*45` (`QModule_RackComponent.cpp:1833`) | la main |

Decision RzZz du 2026-08-31 : la tourelle part **du dos**, donc la premiere formule.

### 4.2 La coque a deux etats existe deja

`UQModule_Settings` porte `BeaconFoldedMesh` (`/Game/Systems/QModule/Meshes/SM_Beacon_Folded`)
et `BeaconDeployedMesh` (`SM_Beacon_Deployed`), plus `BeaconMeshScale`.
`QModule_StrikeBeaconActor.cpp:134` choisit l'un ou l'autre selon `bDeployed`, et le
commentaire du code dit "swapped in the instant the beacon plants" : **le crochet de la
transformation de deploiement existe, il est juste instantane aujourd'hui.**

### 4.3 La preview au sol existe deja

`AQModule_TargetingPreviewActor::QMOD_UpdatePreview(ArcPoints, ImpactNormal, RadiusCm, bValid)`
dessine l'arc holographique et un **anneau de tirets au sol**, teinte valide/invalide.
Pilotee depuis `QModule_GadgetHUD.cpp` par le geste de designation (tenir pour viser, clic
gauche pour lancer, clic droit pour annuler), etats de reticule RECHARGE / HORS PORTEE /
PAS DE CIEL. **Elle n'est allumee aujourd'hui que pour les modules balise**, pas pour
l'ordonnance dorsale.

---

## 5. Cout d'un actif, mesure sur le gabarit le plus proche

Le Drone medical (deployable, duree, effet de zone) touche **8 fichiers** :
`QModule_MedicalDroneActor.h/.cpp`, `QModule_Settings.h/.cpp`, `QModule_RackComponent.h/.cpp`
(champ de cooldown replique + RPC), `QModule_GadgetHUD.cpp` (entree de roue + reticule),
`QModule_TestCommands.cpp`, plus `QModule_SafeZoneLibrary.cpp`.

Le canal actif est encore **cable en dur par tag** : `QModuleGadgetHUD::GadgetTags`
(10 entrees) et `UQModule_RackComponent::ResolveGadgetReadyField`. Le chemin generique
`AbilityGrant` / `UQModule_AbilityBase` (jalon M7) **n'a jamais ete construit**.

---

## 6. Contraintes qui s'appliquent a ce chantier

1. **Regle UX 14.6** (`QMODULE_ARCHITECTURE.md`, validee le 2026-07-04) : actif offensif en
   zone urbaine declenche le wanted QPolice, **plafond d'un deployable par joueur**,
   cooldowns longs. La demande RzZz du 2026-08-31 parle de tourelles au pluriel : conflit
   ouvert, arbitrage a dater.
2. **Inscription d'asset obligatoire** (skill `qanga-assets`) : sans paire `IDA_`/`IS_`,
   registres et seed de cook, le module marche en PIE et disparait en build.
3. **`QAI_IsMachineActor`** compte les tourelles posees via l'ActorTag `QMachine` : une
   sentinelle joueur taguee `QMachine` prendrait le bonus du module Fleau des machines.
   A decider explicitement.
4. **Jamais de Live Coding sur QModule** (3 `UWorldSubsystem`, crash au PIE suivant) :
   build froid obligatoire.
5. **Serveur dedie** : la voie missile de `TurretBase` ne tire pas cote serveur (par. 2.4).
   Une sentinelle joueur doit corriger cela dans son enfant, sans toucher au parent.

---

## 7. Ce qui est livre en C++ (2026-08-31, compile et linke)

### 7.1 Fichiers

| Fichier | Role |
|---|---|
| `Plugins/QModule/Source/QModule/Public/QModule_SentryTurretActor.h` | nouveau, la coque lancee |
| `Plugins/QModule/Source/QModule/Private/QModule_SentryTurretActor.cpp` | nouveau, vol, pose, minutage, spawn de la tourelle |
| `QModule_Settings.h` | bloc `QModule|SentryTurret` : 3 classes de variante, minutage, portee, plafond, audio |
| `QModule_RackComponent.h/.cpp` | RPC `SV_DeploySentryTurret`, cooldown replique, plafond de deployables |
| `QModule_GadgetHUD.cpp` | entree de roue, routage du geste vers le bon appel serveur |
| `QModule_SafeZoneLibrary.cpp` | ajout a la liste de refus en zone sure |
| `Config/Tags/QModuleTags.ini` | 3 tags `Stat.Cyborg.Sentry.*` |

### 7.2 Le partage des roles

Le C++ possede le lancer, la pose, le minutage du deploiement, la duree de vie, le cooldown
et le plafond. **La tourelle posee reste un enfant Blueprint de `TurretBase`** : la boucle de
combat en production n'est pas reecrite, ni meme touchee. Trois variantes sont referencees en
soft class dans les reglages, une par configuration d'armement, indexees par le niveau du
module (1 double mitrailleuse, 2 hybride, 3 double missile). Une variante non renseignee
retombe sur celle du niveau 1 plutot que de ne rien poser.

### 7.3 Le deploiement ne coute rien sur le reseau

Le serveur replique **un seul horodatage**, `DeployStartServerTime`. Chaque machine derive
toute l'animation de cette valeur, exactement comme l'arc de vol derive de ses entrees de
lancer. Un client qui arrive pendant le depliage tombe sur la bonne image au lieu de voir un
pop. La coque s'enfonce et s'ecrase sur son axe vertical pendant que la tourelle sort d'elle,
pour que les deux acteurs se lisent comme un seul objet.

### 7.4 Le geste

La sentinelle est declaree **module balise** (`QMOD_IsBeaconModule`), ce qui lui donne
gratuitement le geste de visee, l'arc holographique, l'anneau de tirets au sol et les etats
de refus au reticule. Seul le tube de lancer differe : elle part **du dos**
(`ActorLocation + Up*95 - Forward*30`, la geometrie des grenades dorsales, centree car il n'y
en a jamais qu'une), et `CommitTargeting` route sa validation vers `SV_DeploySentryTurret`
au lieu de `SV_ThrowBeacon`.

### 7.5 Etat de compilation, honnetement

`QModule_SentryTurretActor.cpp.obj` produit a 16:43:56, `UnrealEditor-QModule.dll` linkee a
16:44:23, **zero erreur et zero warning** portant sur QModule ou Sentry, avec
`-WarningsAsErrors` actif.

La cible `QangaEditor` echoue toutefois dans son ensemble, sur une erreur **exterieure a ce
chantier** :
`DynamicQuestSystemObjectivesModule.cpp(184,18): error C2039: 'CleanupSpawnedActors' n'est
pas membre de 'USpawnActorAction'`. `SpawnActorAction.h/.cpp` ont ete modifies le 2026-08-31
a 16:44, pendant cette session, et la methode a disparu du header alors que son appelant
(date du 2026-06-23) l'appelle toujours. Travail en cours de quelqu'un d'autre : **non
touche**. La DLL `DynamicQuestSystemObjectives` sur disque date du 2026-08-29, donc l'editeur
demarre encore, sur la version precedente de DQS.

**Reste donc a valider par une compilation complete** : rien dans le code de la sentinelle,
mais la cible ne peut pas rendre `Succeeded` tant que DQS n'est pas reconcilie.

## 9. Ce qui est livre en donnees (2026-08-31, verifie apres rechargement disque)

### 9.1 Assets

| Asset | Etat |
|---|---|
| `Content/Phases/QModuleV2/QMD_TourelleSentinelle` | rempli : MaxLevel 3, 3 StatMods, 3 LevelDescriptions, ItemDataAsset |
| `Content/Items/QModuleCyborg/IDA_QModuleCy_TourelleSentinelle` | cree, cle `QModCy_TourelleSentinelle`, icone `T_QMI_TourelleSentinelle` |
| `Content/Items/QModuleCyborg/IS_QModuleCy_TourelleSentinelle` | cree (duplique de la balise), retro-reference corrigee vers son propre IDA |
| `Content/Systems/References/DA_AllRef` | map `ItemKey:DAItem` 336 -> 337, zero entree nulle |

### 9.2 Les chiffres graves

| StatTag (Op = Override) | Niveau 1 | Niveau 2 | Niveau 3 |
|---|---|---|---|
| `Stat.Cyborg.Sentry.CooldownSec` | 120 | 100 | 80 |
| `Stat.Cyborg.Sentry.LifetimeSec` | 90 | 120 | 150 |
| `Stat.Cyborg.Sentry.MaxDeployed` | 1 | 2 | 2 |

Le plafond passe a 2 des le niveau 2 : c'est la derogation assumee a la regle UX 14.6
(RzZz, 2026-08-31). Duree de vie doublee le 2026-09-15 (Benja : les sentinelles s'usaient trop
vite) ; elle valait 45 / 60 / 75 s.

### 9.3 Cook

Les deux dossiers sont deja couverts : `/Game/Phases/QModuleV2` par le scan AssetManager
`QModuleAssets` (`DefaultGame.ini:183`) ET par `DirectoriesToAlwaysCook` (ligne 134),
`/Game/Items/QModuleCyborg` par `DirectoriesToAlwaysCook` (ligne 140). Rien a ajouter pour
ces assets.

**Attention pour la suite** : les 3 Blueprints de variantes seront references en **soft class
depuis les reglages**, donc invisibles au cooker. Il faudra les couvrir explicitement, sinon
le module marchera en editeur et sera ABSENT de la build (piege deja paye sur ce projet).

### 9.4 Deux observations sur l'existant, sans conclusion

1. `ModuleItemAssetByTagName` ne contient que **3 entrees** (CanonRenforce, ChargeurRapide,
   NoyauSurcadence, tous arme ou vehicule). Les 43 items de module cyborg n'y sont pas :
   leur lien passe par le champ `ItemDataAsset` du QMD. Rien a inscrire la.
2. Sur 6 scripts d'item cyborg sondes, **5 portent un `ItemDataAsset` pointant vers
   `IDA_QModuleCy_ChasseurDePrimes`** (BaliseDeFrappe, DroneMedical, LargageDeRavitaillement,
   Geologue, VerinsDeSaut) ; seul NidDeGuepes pointe vers le sien. Reliquat de duplication.
   Non propage sur la sentinelle, dont la retro-reference est correcte. Aucune conclusion
   tiree sur un eventuel effet en jeu : c'est une observation, pas un diagnostic.

### 9.5 Choix de langue a valider

Les `LevelDescriptions` et `ItemDescription` sont ecrites en **anglais**, comme celles de
`QMD_BaliseDeFrappe` et conformement a la culture par defaut du projet (`en`). Le nom reste
en francais ("Module : Tourelle sentinelle") comme tous les autres modules. L'existant est
incoherent sur ce point (`QMD_DroneMedical` est en francais) : bascule en une edition si
RzZz prefere le francais.

### 9.6 Ce qui n'est PAS verifie

Rien n'a tourne en PIE. La validation du registre (`qats.qmodule.start Cyborg Contract`), le
test drop / rechargement de l'item et le comportement en jeu restent a faire.

## 10. Les trois variantes de tourelle (2026-08-31)

`/Game/Systems/QModule/Turrets/` : `Turret_ModuleCy_TwinGun`, `Turret_ModuleCy_Hybrid`,
`Turret_ModuleCy_TwinMissile`. Enfants Blueprint de `TurretBase`, cree via `BlueprintFactory`,
compiles et sauves. Le dossier est couvert par `DirectoriesToAlwaysCook` (`/Game/Systems/QModule`),
donc la reference soft depuis les reglages ne fait pas disparaitre les variantes au cook.

### 10.1 Comment le kit se greffe sur la hierarchie du parent

**Correction du 2026-08-31 apres retour RzZz ("tous les mesh sont en 0,0,0, ce n'est pas bon
visuellement").** Ma premiere passe avait tout mis a la position relative zero, en supposant
que les decalages etaient bakes dans les pivots. C'etait faux : les pieces du kit sont
**attachees par sockets**, donc leur position relative vaut bien zero mais leur position
MONDE ne l'est pas. Transforms monde mesurees sur le kit pose a l'origine :

| Piece | Position monde |
|---|---|
| Socle, Corps | 0, 0, 0 |
| Oeil | 0, -3.36, 91.23 |
| Mitrailleuse | -36.72, -27.42, 92.20 |
| Pod missiles | +36.70, -26.76, 91.84 |
| Chargeur | -57.61, -29.88, 70.38 |

Les deux armes sont donc symetriques a X = +/-36.7 et la tete est a Z = 91.

Deux consequences structurelles :

1. **Le pivot de pitch doit etre a la tete.** `WeaponMesh` etait a l'origine de l'acteur, donc
   les armes auraient pivote autour du sol au lieu de hocher. Il est desormais a (0, 0, 91).
2. **Un noeud de tete porte le lacet -90.** Le canon du kit pointe vers +Y, alors que le parent
   vise le long de +X (`WeaponProjectileLocation.GetForwardVector`). `SentryHeadRoot` fait la
   conversion une seule fois pour tout l'ensemble cosmetique.

```
Base                 <- Socle                       (jamais tourne)
Rotator (yaw)
  +-- SentryBody     <- Body            rel 0,0,0
WeaponMesh (pitch)   rel 0,0,91
  +-- SentryHeadRoot  rot lacet -90
  |     +-- SentryEye        <- Eye      rel 0,-3.36,0.23
  |     +-- SentryGunL/PodL  <- Gun/Pod  rel -36.72,-27.42,1.20  ou  -36.70,-26.76,0.84
  |     +-- SentryGunR/PodR  <- Gun/Pod  rel  36.72,-27.42,1.20  ou   36.70,-26.76,0.84
  |     +-- SentryLoaderL/R  <- Loader   rel -57.61,-29.88,-20.62
  +-- WeaponProjectileLocation  rel 120,0,0
```

**Verifie numeriquement** : le kit et la variante Hybrid poses cote a cote, chaque piece de la
variante tombe sur la position du kit tournee de -90 en lacet, ecart < 0,5 cm sur les 6 pieces.

Les copies jumelles utilisent une **echelle X negative** pour le miroir, ce qui inverse les
normales. Acceptable a l'oeil, a remplacer par des meshes miroir dedies si ca se voit.

### 10.2 Valeurs par variante

| | TwinGun (niv 1) | Hybrid (niv 2) | TwinMissile (niv 3) |
|---|---|---|---|
| Emplacement gauche / droit | Gun / Gun | Gun / Missiles | Missiles / Missiles |
| `TurretWeapon` | MachineGun | MachineGun | Missile |
| `CombatComponent.LifeWhenNoStat` | 400 | 550 | 700 |
| `WV_DetectPlayer` | 6000 cm | 7000 cm | 8000 cm |
| `CollisionSphere` | r=90 a Z=60 | idem | idem |

Toutes les surcharges de composants herites sont ecrites sur le **template propre a l'enfant**
(`get_object_for_blueprint`), verifie par l'outer : `Turret_ModuleCy_*_C`. `TurretBase` n'a pas
ete touche.

Le respawn du parent n'a pas besoin d'etre desactive : le C++ pose un `SetLifeSpan` de 90 a 150 s (45 a 75 s avant le 2026-09-15)
sur la tourelle, toujours plus court que les 300 s du parametre global `Turrets/RespawnS`, donc
une sentinelle detruite ne revient jamais.

### 10.3 Cablage

`Config/DefaultGame.ini`, section `[/Script/QModule.QModule_Settings]` : les trois
`SentryTurretClassLevel1/2/3` pointent les variantes. Les trois classes ont ete chargees et
portent la bonne valeur de `TurretWeapon`. Les deux `Saved/Config/.../Game.ini` locaux ne
portent que `Enabled=True` pour cette section, ils ne masquent donc pas ces cles.

## 11. Etat reel, et ce qui reste

### 11.1 Verifie

- C++ compile, linke, **vivant dans l'editeur** (`/Script/QModule.QModule_SentryTurretActor`
  se charge, les reglages Sentry repondent).
- Donnees completes et relues depuis le disque (par. 9).
- Les 3 variantes existent, se compilent, se chargent, portent les bons meshes (6 composants
  mesh valides avec leurs materiaux) et les bonnes valeurs.
- Aucun debris laisse dans `L_Safe_Startup` : les acteurs de test ont ete detruits et le
  niveau n'a jamais ete sauvegarde.

### 11.2 NON verifie, et pourquoi

1. **L'aspect visuel des variantes.** Impossible a capturer : dans `L_Safe_Startup` le viewport
   rend blanc en mode lit et noir en unlit, et **un mesh temoin pose nu ne rend pas davantage**.
   Le probleme est donc l'eclairage de cette map, pas l'assemblage. A juger a l'oeil en ouvrant
   un des trois Blueprints.
2. **Le jeu n'a jamais tourne.** Le geste, l'arc, la pose, le depliage et le tir n'ont pas ete
   exerces une seule fois.
3. **Le depliage de la tourelle elle-meme n'existe pas.** Le C++ fait sombrer et ecraser la
   coque sur 1,6 s, puis la tourelle apparait. La transformation "elle se complete en temps
   reel" demandee le 2026-08-31 suppose une timeline sur les pieces du kit, cote Blueprint.
   **Non fait.**
4. **Le niveau 3 herite du verrou serveur dedie.** Mesure noeud par noeud : dans
   `TurretBase.FireWeapon`, le `Branch` du `Is Dedicated Server` (`K2Node_IfThenElse_3`) n'a
   que sa sortie `else` connectee. Sur serveur dedie, la voie missile ne fait rien. Cela
   concerne aussi `Turret_RocketLaucher` en production. Le correctif tient en UNE connexion
   dans le parent partage : **non fait, arbitrage RzZz requis**.

### 11.3 Un signal de regression a trancher (pas forcement le mien)

`qats.qmodule.start Cyborg Contract` **avorte au preflight** :
`__UniversalRack__ | Failed | universal rack install/phase/active-capacity contract failed.`

Historique des runs sur cette copie : preflight **Passed** le 2026-07-31, le 2026-08-13 et le
2026-08-23 ; **Failed** aujourd'hui. C'est donc une regression reelle, apparue apres le 23/08.

Ce que j'ai pu ecarter :
- le module que le preflight selectionne est `QMD_VerinsDeSaut` (premier non-core non-exclusif
  du registre), **pas la sentinelle** ;
- `UnrealEditor-QModule.dll` (17:21:56) et `UnrealEditor-QAutomatedTestSuite.dll` (17:23:10)
  viennent du meme build, donc pas de melange de layouts ;
- mes modifications du rack sont **purement additives** (nouveaux membres, nouvelles fonctions) :
  je n'ai touche ni a l'installation, ni aux phases, ni au calcul de `bActive`
  (`QModuleAggregation::RecomputeDerivedState`, fichier non modifie).

Ce que je n'ai PAS pu ecarter : mes ajouts changent le layout de `UQModule_RackComponent`.

Piste la plus probable, a verifier par RzZz : **six sources QModule ont ete modifiees
aujourd'hui AVANT mes edits**, et elles sont exactement dans la zone que le preflight
exerce : `QModule_Types.h` (00:59, qui porte `FQModule_SocketState.bActive`),
`QModule_SReticle.cpp` (01:56), `QModule_HudRackWidgetBase` (03:03),
`QModule_WallWidgetBase` (14:33), `QModule_GadgetHUD.h` (14:51).

Test decisif propose : relancer `qats.qmodule.start Cyborg Contract` sur une copie sans mes
quatre edits C++. Si le preflight echoue quand meme, la regression est anterieure.

## 12. Etat au 2026-09-15 (resume de reprise)

### Correctif du combat, compile et valide en PIE le 15/09

**Livre et actif** : C++ compile par Codex apres autorisation explicite de l'utilisateur,
classes chargees dans l'editeur, cinq Blueprints compiles et sauvegardes sans erreur ni
avertissement. Les essais ci-dessous utilisent uniquement `L_Dev_Claude`, avec un serveur
dedie PIE (instance 0) et un client connecte (instance 1). PIE est arrete en fin de chantier.

#### Causes mesurees et rectification des hypotheses du par. 2.2

- Le `FindLookAtRotation` de `UpdateAim` part de **WeaponMesh**, pas du muzzle.
  `PredictedLocation` recoit directement la position de la cible, sans prediction de vitesse.
  Le `LoopTimer` de 1 s reprend `Delay(0)` pendant cette seconde : ce n'est pas un calcul a 1 Hz.
- Avant correction, le muzzle est a **X=120 cm** sous WeaponMesh, pivot a Z=91 cm.
  `FireStaticDefenseMachineGun` utilise sa position et son forward. Son trace serveur ignore
  les hits `bStartPenetrating` ou de distance <= 1 cm. A 1,2 m, mesure en PIE : muzzle a
  **0,337 cm du centre de la capsule IA**, trace initialement penetrant, distance 0 ; depuis
  le pivot, entree a **90,109 cm** sans penetration. Vie serveur **98700 -> 98700 en 6 s**.
  Recentrage du muzzle de cette seule instance : **98700 -> 98550 en 10,674 s**.
  L'origine avancee explique donc directement les tirs perdus au contact.
- `SetWeaponRotation` reoriente aussi le muzzle en rotation monde puis, apres **0,07 s**,
  vise un point aleatoire dans une boite de demi-taille **200 cm** autour des non-vehicules.
  Cette constante ignore `TargetUnprecision=0`. Elle perturbe fortement le tir au contact.
  Le clamp [-30,+60] du pivot ne borne pas directement cette rotation independante du muzzle.
- Dans l'orbite de reference a 2 m, recentrage seul, le pivot accuse jusqu'a **21,95 degres**
  de retard et un bref recul de 158,802 a 158,045 degres au passage de cible 165 -> 180.
  A 80 cm, le muzzle avance peut viser vers l'arriere. La singularite supposee du calcul
  principal depuis le muzzle est donc infirmée ; l'origine du tir, la dispersion et le
  suivi interpole sont les defauts effectivement observes.
- La branche missile de TurretBase ne spawne rien sur serveur dedie. BP_Missile detruit en
  construction les projectiles dedies sans ShoulderMissile et utilise historiquement
  `ClientRequestDamage`. Le projectile propre au module remplace ces deux chemins.
- La remise en service du missile a revele un masquage par son propre collider : projectile
  immobilise entre tourelle et IA, `FindNearest` renvoie `None` avec Visibility=Block, puis
  **la meme IA** avec Visibility=Ignore. Seule cette reponse est changee dans l'enfant missile.
  Preuve A/B : `evidence/Missile_LineOfSight_AB.json`.

#### Correctif limite aux sentinelles du module

- Nouveau parent `/Game/Systems/QModule/Turrets/Turret_ModuleCy_CombatBase`, enfant de
  TurretBase, parent des trois variantes existantes. Son seul evenement de combat surcharge
  `SetWeaponRotation` et appelle `SentryCombat.ApplyAim`, sans appel parent.
- `UQModule_SentryCombatComponent` vise la position actuelle de la cible depuis le pivot
  stable, dans le repere planetaire de Base. Il ecrit lacet et tangage sur le serveur ;
  les SceneComponents repliquent ces rotations au client. Muzzle recentre et rotation
  relative nulle : plus de rotation aleatoire independante ni d'origine devant la cible.
  Acquisition, riposte, priorite vehicule et cadences restent celles de TurretBase.
- Nouveau `/Game/Systems/QModule/Turrets/Projectile_ModuleCy_SentryMissile`, enfant de
  TurretMissileProjectile. La surcharge `FireWeapon` du seul TwinMissile cree ce projectile
  sur l'autorite ; sons et notification de menace vaisseau sont conserves.
  `UQModule_SentryMissileComponent` applique les degats sur serveur avec la politique native
  QWeapon/QCombat et la regle de zone sure. Impact replique, mouvement client desactive,
  retour cosmetique via le hook existant. Rayon **600 cm**, attenuation quadratique et
  multiplicateurs vehicules conserves ; le tireur est explicitement exclu du souffle.
- La collision de vol reste la sphere racine existante de **50 cm**, independante du mesh.
  Camera et Visibility sont Ignore dans ce seul enfant ; les autres reponses sont conservees.
  Aucun code de combat n'est ajoute a la coque detruite apres depliage. Noms des pieces,
  hierarchies de deploiement et Base Movable sont conserves.

#### Mesures apres correction

Les IA de test proviennent du **QAI_AgentSpawner existant**, classe configuree
`AI_Infected_1`, redirigee par le projet vers AILean. Pour isoler la geometrie, les series
fixes desactivent `QAI_FloatingPawnMovement.EnabledSimulation` sur les deux pairs et utilisent
`K2_SetActorLocation`. `LockMovement` seul ne garantit pas une position fixe en combat.
La vie est lue dans `GetCombatStateSnapshot`, jamais dans la valeur initiale `CurrentLife`.
Les PV eleves des cibles et tourelles sont des reglages d'instances PIE uniquement.

| Essai | Palier 1 | Palier 2 | Palier 3 |
|---|---:|---:|---:|
| Rayons 50 / 80 / 120 / 200 / 300 cm | 75 degats a chaque point | 125 / 125 / 150 / 100 / 175 | 198 / 198 / 194 / 182 / 79 |
| Orbite controlee a 120 cm, 25 positions, pas de 15 degres | 500 degats | 500 degats | 485 degats |
| Ecart maximal de lacet serveur/client dans cette orbite | 0,00273 degre | 0,00197 degre | 0,00197 degre |
| Tir a 10 m | 75 degats | 100 degats | 89 degats |
| Tir a 50 m, ligne degagee | 150 degats | 200 degats | 145 degats |
| Tir a 150 m, ligne degagee | 75 degats | 225 degats | 156 degats |

Les fenetres de mesure durent environ 5 a 9 s selon la serie : ces totaux ne comparent pas
les DPS. Le lacet deroule sans inversion sur les trois orbites ; erreur maximale par rapport
aux positions demandees < 0,016 degre. Les tirs a 50/150 m utilisent une ligne surélevee de
50 m pour isoler la portee des obstacles de la carte. La serie missile finale a 10/50/150 m
ne perd aucune cible dans ses huit releves par distance, apres correction de Visibility.

- **Mouvement QAI normal** : palier 1, IA arrive au contact puis meurt ; tourelle attaquee au
  corps a corps. Paliers 2 et 3, `EnabledSimulation` reactive sur les deux pairs et poursuite
  demandee par l'API existante `QAI_AgentComponent.SetPursuitTarget` : **300 et 279 degats**,
  attribues par `GetLastDamageCauser` aux bonnes tourelles. L'IA atteint environ **1,67/1,68 m**
  du centre de la tourelle et l'attaque. Les releves de position serveur/client sont
  sequentiels, pas une mesure synchronisee du retard QAI ; ils ne prouvent pas que QAI soit
  la cause initiale du defaut.
- **Priorite vehicule** : le FindNearest herite choisit un SingleST hostile a environ 35 m
  avant l'IA a 3 m dans le test du palier 1, cible courante effacee. L'entree IA est fournie
  par l'evenement OnEnterVolume existant pour ce test a simulation figee. Les trois variantes
  conservent exactement le meme proprietaire de cette logique, sans surcharge de selection.
  Ce controle ne constitue pas une matrice exhaustive des situations de priorite vehicule.
- **Replication missile** : projectile present sur le serveur dedie et le client, Tick de
  mouvement client desactive ; meme position et normale d'impact repliquees sur les deux.
- **Deploiement** : RPC de lancer du rack execute pour les niveaux 1, 2 et 3 apres installation
  par l'API autorite de test existante. Les **9 / 8 / 7 pieces** changent de transform sur
  serveur et client ; a la fin, meme variante debout sur les deux pairs et coque absente.
  Le test parcourt le chemin rack/coque/depliage, pas la manipulation clavier T/X de la roue.
  Le module de test a ete retire par l'API autorite, sans credit d'inventaire ou de phases.

#### Compilation, preuves et integrite

- Build complet **QangaEditor : Succeeded** ; test natif **QModule.Sentry.AimGeometry : Success**
  (repere planetaire, coordonnees LWC, passage +/-180 degres, visee sous -30 degres et pole vertical).
- Dossier `Saved/SentryCombat_Patch_20260915/` : `orig`, `new`, `diff`, manifest MD5,
  `APPLY.ps1`, `REVERT.ps1`, integration appliquee et preuves JSON/logs dans `evidence`.
  Le retour arriere couvre les **5 fichiers source et 5 assets** ; son precontrole MD5 passe.
  Les mutations du script exigent l'editeur ferme, puis une compilation avant reouverture.
- Mesures : `Validation_summary.json`, `After_Tier1_Matrix.json`, `After_Tier2_Matrix.json`,
  `After_Tier3_Isolated.json`, `After_Tier*_Orbit.json`, `Final_Tier3_Range.json`,
  `After_Tier1_ClearRange.json`, `After_Tier2_Tier3_ClearRange.json`,
  `After_Natural_Tier1.json`, `Final_Natural_Tier2_Tier3.json`,
  `Final_Missile_Replication.json`, `Vehicle_Priority.json`, `Deployment_ThreeTiers.json`.
  Diagnostic avant correction : `PIE_measurements.json` et exports semantiques des graphes.
- `TurretBase.uasset` avant/apres : **CEF39D52282AC454BB9C168214BAA7B9**.
  TurretMissileProjectile et BP_Missile sont aussi identiques aux MD5 initiaux. Tourelles du
  monde et QBuilder, carte et rendu inchanges. `CLAUDE.md` absent de cette racine ;
  instructions de `agents.md` lues. Aucun fichier moteur modifie.
- **Limite** : validation en PIE dedie avec client, pas en serveur packagé. PIE conserve les
  meshes editeur. Les erreurs Blueprint LocalData/achievements deja presentes dans les
  sessions de test ne permettent pas de qualifier le journal global de sans erreur.

### Historique du deploiement

- **Les 3 paliers se deploient en jeu** (PIE `L_Dev_Claude`, DLL du 14/09) : niveau 3
  `TwinMissile` (7 pieces animees resolues), niveau 1 `TwinGun` (9), niveau 2 `Hybrid` (8), et
  Benja confirme "ca fonctionne quel que soit le palier". Le "paliers 2 et 3 KO" du 10/09 datait
  de l'ancienne DLL ; le build complet du 14/09 (22:54 a 23:00, `Result: Succeeded`) a aussi regle
  `CombatComponent`, qui ne compilait plus (`ResolveLastDamageController` synchronise, non compile).
- **Retour Benja du 15/09** : aucune animation de deploiement, "tout reste aplati et la tourelle
  100 % montee apparait d'un coup" ; les tourelles doivent tenir deux fois plus longtemps.
- **Cause mesuree du depliage fige** : `AQModule_ThrownDeviceActor::ApplyPlantedState` (et son
  `Tick`) coupe le Tick de la coque des qu'elle touche le sol. La sentinelle ne tiquait donc plus :
  `ApplyUnfold(0)` a la pose (tout aplati), puis plus rien jusqu'a la mort de la coque par sa duree
  de vie de secours (`ThrowMaxFlightSeconds + 20 s`), dont l'`EndPlay` posait la tourelle finie.
  La balise de frappe et la grenade collante reactivent leur Tick dans `OnPlantedCosmetic` ; la
  sentinelle ne le faisait pas.
- **Second defaut, lu dans le log** : `Base`, herite de `TurretBase`, est `Static` ; le moteur
  refuse de le deplacer ("Base has to be 'Movable' if you'd like to move").
- **Fait en donnees (sauve)** : `Base` passe `Movable` sur les 3 variantes, par surcharge propre a
  chaque enfant (outer `Turret_ModuleCy_*_C`), `TurretBase.uasset` identique au md5 pres avant et
  apres ; les 3 variantes compilent sans avertissement. `QMD_TourelleSentinelle` : duree de vie
  **90 / 120 / 150 s** (tags et operations relus apres ecriture).
- **Patch C++ applique le 2026-09-15 sur go de Benja et compile vert** (build de Benja a 01:54 local,
  `Result: Succeeded`, 0 erreur ni avertissement, DLL QModule plus recente que toutes ses sources ;
  `Saved/SentryTurret_Patch_20260915/`,
  4 fichiers verifies au md5, sauvegarde `before_apply/`, retour arriere `REVERT.ps1`) :
  - Tick reactive a la pose et au late join ; la branche posee du `Tick` ne repasse plus par la
    brique de base (meme decoupage que la balise) ; une piece non `Movable` reste debout avec un
    avertissement unique au lieu d'un refus par image ;
  - sons : sifflement de lancer et clonk de pose de la balise (meme coque, meme geste, memes
    appels) ; depliage sur `Scifi_Elevator_Medium_End`, l'onde de `QBuilder_Sound_Construct`
    (la construction des tourelles de base), jouee en onde monde (la MetaSound est routee
    `QSClass_UI`), profil `QATT_GameplayElement`, pitch 1.3 pour que ses 2,15 s tombent sur le
    depliage de 1,6 s ; `SentryTurretLockAudio` cree **vide** (regle `DropshipRampAudio` : a
    remplir apres ecoute, pour ne pas empiler deux clonks) ;
  - repli `SentryTurretLifetimeSeconds` 45 -> 90 (egal au palier 1).
- **Verifie en PIE apres les correctifs de donnees** (23:36 UTC, DLL encore sans le patch) : palier 2
  lance avec "life 120 s", plus aucun avertissement de mobilite sur `Base`.
- **Suivi actuel** : lancers des trois paliers revalides ci-dessus ; descriptions en/fr/es qui disent
  encore 45 / 60 / 75 s (passe loc : source, traductions, compilation des locres) ; aucun effet de
  pose ajoute, ni la balise ni la construction QBuilder n'en ont (la poussiere du largage
  `NS_Smoke_03` reste disponible) ; arbitrages hors de ce correctif (voie missile monde/QBuilder inchangee,
  pas de hochement visuel, variantes jumelles en echelle negative).
- **Equilibrage a connaitre** : 90 / 120 / 150 s de vie contre 120 / 100 / 80 s de recharge et un
  plafond 1 / 2 / 2 : au palier 3, deux sentinelles restent en poste presque en permanence.

## 8. Journal

- **2026-08-31** : dossier ouvert. Renommage `SM_Turret_MacgineGun` -> `SM_Turret_MachineGun`
  livre et verifie. Mesure complete de `TurretBase`, des 3 enfants et du kit
  `Turret_Mesh_Module_Cyborg`. Decision RzZz : lancer dorsal facon grenade collante, coque
  de balise, deploiement anime, trois configurations d'armement (double mitrailleuse,
  hybride, double missile).
- **2026-08-31, phase 4 (C++)** : coque lancee, reglages, RPC serveur, cooldown replique,
  plafond de deployables, entree de roue, refus en zone sure et 3 tags de stats. QModule
  compile et linke propre. Cible globale bloquee par une erreur DQS exterieure (par. 7.5).
- **2026-08-31, phase 3 (donnees)** : QMD rempli, paire IDA_/IS_ creee, cle inscrite dans
  DA_AllRef (336 -> 337), cook deja couvert. Verifie apres rechargement disque. Reste les
  3 Blueprints de variantes, leur couverture de cook, et le test PIE.
- **2026-08-31, phases 4 a 6** : 3 variantes de tourelle creees et cablees, reglages en ini,
  C++ verifie vivant dans l'editeur. NON FAIT : depliage anime de la tourelle, test en jeu,
  validation visuelle. A TRANCHER : verrou serveur dedie de la voie missile, et regression du
  preflight QATS apparue apres le 23/08.
- **2026-09-15** : les 3 paliers se deploient en PIE. Depliage fige explique (Tick coupe a la pose par
  la brique de base) ; `Base` Movable sur les variantes et duree de vie doublee, faits en donnees ;
  patch C++ du Tick et des sons applique sur go (`Saved/SentryTurret_Patch_20260915/`), compile vert a
  01:54 ; reste le test en jeu.
