# QMODULE : catalogue des modules d'ARMES et de VEHICULES

> **Statut : proposition de design, adossee a une mesure du projet faite le 2026-09-04.**
> Compagnon de `QMODULE_CATALOGUE.md` (brainstorm du 2026-07-04, 24 modules d'arme et 19 de
> vehicule, generiques) et de `QMODULE_ACTIVATION_ALIGNMENT.md` (leviers reels).
> Ce document remplace la partie "Armes" et "Vehicules" du brainstorm : il decoupe **par type
> d'arme et par classe de vehicule**, et chaque entree cite le **levier reel mesure dans le
> projet**, pas une intention.
>
> **Legende Etat**
> `OK` : le levier existe et a ete lu dans le code ou les assets ; le module se resume a des
> donnees plus une ligne de lecture.
> `PARTIEL` : le levier existe mais il faut l'etendre (une valeur en dur a rendre variable,
> une donnee authored qui n'existe que sur une famille d'armes).
> `NEUF` : la mecanique n'existe pas du tout ; c'est un chantier a part, chiffre en fin de
> section.
>
> **T** : type d'effet. `P` passif chiffre (pur data), `A` actif (abilite), `C` comportement
> (composant attache).

---

## TL;DR

- **122 modules d'arme** proposes : 13 universels toutes armes, 22 universels armes a feu,
  62 par famille (pistolet, PM, fusil d'assaut, mitrailleuse, pompe, precision, lance-grenades,
  lance-roquettes, famille NASH, outils armes) et 25 au corps a corps.
- **75 modules de vehicule** : 19 universels, 18 sol et sustentation, 14 vol et spatial,
  4 nautiques, 7 utilitaires, 7 d'armement de bord, 6 de furtivite et de loi.
- **Environ les deux tiers sont marques `OK`** : le levier existe deja dans le projet et a ete lu
  ce soir. Un module `OK`, c'est un asset de donnees plus une ligne de lecture.
- **Le vrai verrou n'est pas la liste, c'est le pont.** Aucun code de gameplay ne lit aujourd'hui
  le rack d'un exemplaire d'arme ou de vehicule. Trois petites taches (paragraphe 3, etape 0)
  rendent vivants d'un coup une vingtaine de modules, dont les 7 modules d'arme qui existent deja
  en asset et ne servent a rien.
- **Deux modules existants sont casses en donnee** et le resteront apres le pont :
  `ChargeurRapide` (mauvais operateur) et `ChambreThermique` (contreparties vides). Corrections
  gratuites, etape 0 bis.
- **La melee est un terrain vierge** : les 12 armes de corps a corps sont **strictement
  identiques** en donnees. Les modules n'y ameliorent pas un typage, ils le creent.

---

## 0. Ce qui est mesure, et le seul vrai verrou

### 0.1 Le contrat technique (lu dans `QModule_Definition.h` le 2026-09-04)

Un module est un `UQModule_Definition` (asset `QMD_`). Champs utiles au design :

| Champ | Ce qu'il permet |
|---|---|
| `Domain` | `Cyborg` / `Weapon` / `Vehicle`. Trie le module vers le mur, le rack d'arme ou le rack de vehicule. |
| `TargetFilter` (`FGameplayTagQuery`) | **C'est le mecanisme "par type d'arme"**. Vide = universel. Le commentaire du header dit deja "weapon families, vehicle classes". |
| `ExclusivityTag` | Un seul module actif par groupe sur une meme cible (un seul canon, une seule optique). |
| `SynergyTags` | Familles, pour l'adjacence du mur. Sur un rack d'objet il n'y a **pas** d'adjacence (mesure : `QModuleItemRack::GetStat` agrege sans adjacence). |
| `StatMods` | `StatTag` + `Op` (Add / Multiply / Override / ClampMax) + `ValuePerLevel`. |
| `Drawbacks` | Contreparties, meme pipeline, appliquees sans condition tant que le module est actif. C'est la ou vivent les variantes instables Voss. |
| `MaxLevel` | 3 par defaut. Le niveau vient des phases inserees. |
| `ManufacturerTag` | `Manufacturer.ICLab` et `Manufacturer.Voss` existent deja dans `Config/Tags/QModuleTags.ini`. |
| `ItemDataAsset` | L'item d'inventaire. Depuis le 2026-08-28, `bAllowItemlessModuleInstall=True` : un module sans item s'installe quand meme. |

Le rack d'une arme vit **par exemplaire**, dans une cle `QMODRack` du DataObject de l'instance
d'item : verite serveur, persistance portee par l'item. Le rack d'un vehicule est pose
automatiquement par `UQModule_VehicleRack_World_SubSystem` sur tout acteur dont la chaine de
classes contient `VehicleBase_C`, cote serveur, sans toucher un seul Blueprint.

### 0.2 LE VERROU : aujourd'hui aucun module d'arme ou de vehicule n'a d'effet

Mesure du 2026-09-04, refaite ce soir : **`QModuleItemRack::GetStat` n'a qu'un seul appelant dans
tout le projet, la commande de test `qmodule.Test.Weapon.Stat`** (`QModule_TestCommands.cpp:1250`).
La facade Blueprint `UQModule_StatLibrary` expose `QMOD_GetStat` (mur du PlayerState) mais
**n'expose aucune fonction pour lire le rack d'un exemplaire d'item**. Consequence : les 5 modules
d'arme et les 2 modules de vehicule qui existent deja en `QMD_` sont **inertes par construction**,
et tout ce que ce catalogue propose le restera tant que ce pont n'est pas ecrit.

Ce n'est pas une objection au catalogue, c'est son etape 0. Elle est petite et elle est chiffree
au paragraphe 3.

### 0.2 bis Etat exact des 9 modules d'arme et de vehicule qui existent deja

Releve fait ce soir sur les assets, pas sur la doc.

| QMD | Domaine | Max | StatMods reels | Contreparties | TargetFilter |
|---|---|---|---|---|---|
| `AT56` (base) | Weapon | 3 | `Weapon.Damage` Add 10 / 15 / 20 **et** `Weapon.FireRate` Add 0.0588 / 0.1176 / 0.1765 | aucune | vide |
| `Nash` (base) | Weapon | 3 | `Weapon.FireRate` Add 0.1 / 0.2 / 0.4 **et** `Weapon.Damage` Multiply 0.05 / 0.10 / 0.15 | aucune | vide |
| `AmplificateurDeDegats` | Weapon | 3 | `Weapon.Damage` Multiply 0.05 / 0.10 / 0.15 | aucune | vide |
| `CanonRenforce` | Weapon | 3 | `Weapon.Damage` **Add 10 / 20 / 30** | aucune | vide |
| `ChambreThermique` | Weapon | 3 | `Weapon.Damage` Multiply 0.20 / 0.35 / 0.50 | **VIDES** | vide |
| `ChargeurRapide` | Weapon | 3 | `Weapon.FireRate` **Multiply** 0.1 / 0.2 / 0.3 | aucune | vide |
| `RecycleurDeDouilles` | Weapon | 2 | `Weapon.MatterPerShot` Add 1 / 2 | aucune | vide |
| `NoyauSurcadence` | Vehicle | 3 | `Vehicle.Speed.Max` Multiply 0.1 / 0.2 / 0.3 | aucune | vide |
| `Hydroglisseur` | Vehicle | 3 | `Vehicle.Hydroplane.MinSpeedKmh` Override 70 / 45 / 25 **et** `.ControlMult` Override 0.5 / 0.75 / 1.0 | aucune | vide |

Trois enseignements pour le design a venir :

1. **`CanonRenforce` ne fait pas de portee, il fait des degats plats.** Le nom promet un canon, la
   donnee dit "plus de degats". A trancher : soit on renomme, soit on lui donne la portee (le
   levier `Range` existe). Ce catalogue propose la seconde voie et cree un module distinct pour les
   degats plats.
2. **`ChargeurRapide` est en `Multiply` sur un tag qui porte une fraction.** La semantique figee du
   projet est : `Stat.Weapon.FireRate` = **fraction de reduction du delai**, agregee en `Op Add`,
   plafonnee a 0.8 (le plafond est deja pose dans `QModule_Settings.cpp:315`). Un `Multiply` sur une
   base de 0 rend 0 : ce module est **mathematiquement inerte** meme une fois le pont ecrit. A
   corriger en meme temps que l'etape 0.
3. **Aucun `TargetFilter` n'est renseigne.** Tout le decoupage par type de ce document reste donc a
   ecrire dans les assets, et c'est une passe de donnees, pas de code.

Tags natifs deja declares dans `QModule_Tags.cpp` et donc utilisables tout de suite :
`Module.Weapon`, `Module.Vehicle`, les familles de synergie `Module.Family.Weapon` et
`Module.Family.Vehicle`, et les stats `Stat.Weapon.Damage`, `Stat.Weapon.FireRate`,
`Stat.Vehicle.Speed.Max`, `Stat.Vehicle.Fuel.Max`. Les autres (`Stat.Weapon.MatterPerShot`,
`Stat.Vehicle.Hydroplane.*`) sont dans `Config/Tags/QModuleTags.ini`.

### 0.3 L'inventaire reel des armes (mesure editeur du 2026-09-04)

Valeurs lues sur les CDO des `IS_` (donc les defauts de classe, avant l'ecrasement eventuel par le
`Switch on Int` de phase).

**Armes a feu**

| Arme | Degats | FireDelay | Chargeur | Rechargement | Tir | Rarete | Prix |
|---|---|---|---|---|---|---|---|
| AK47 | 50 | 0.12 | 30 | 2.0 | hitscan | 1 | 2500 |
| AT56 | 30 | 0.17 | 32 | 1.6 | hitscan | 2 | 4500 |
| FA62 | 40 | 0.10 | 30 | 2.2 | hitscan | 0 | 2500 |
| MZ56 | (0 au CDO) | 0.30 | 35 | 1.8 | hitscan | 5 | 1200 |
| Shotgun | (0 au CDO) | 1.50 | 12 | 4.0 | hitscan | 1 | 2500 |
| Sniper | 200 | 2.00 | 5 | 2.0 | hitscan | 5 | 15000 |
| NASH (V1, 6 armes) | 0 a 50 | 0.12 a 0.15 | 16 a 30 | 1.6 a 2.0 | projectile 2000 km/h | 0 a 3 | 450 a 5000 |
| NashV2 Pistol | 20 | 0.15 | 60 | 1.5 | hitscan | 0 | 500 |
| NashV2 Smg | 15 | 0.10 | 60 | 1.5 | hitscan | 0 | 1000 |
| NashV2 Assault Rifle | 35 | 0.15 | 60 | 1.5 | hitscan | 0 | 2500 |
| NashV2 MachineGun | 20 | 0.10 | 120 | 1.5 | hitscan | 2 | 4000 |
| NashV2 Shotgun | 60 | 0.50 | 24 | 2.0 | hitscan | 1 | 1500 |
| NashV2 Sniper | 250 | 0.70 | 6 | 1.5 | hitscan | 5 | 10000 |
| NashV2 Grenade Launcher | 70 | 0.50 | 7 | 1.5 | **projectile** | 4 | 7500 |
| NashV2 Rocket Launcher | 300 | 1.00 | 4 | 2.0 | **projectile** | 5 | 9000 |
| Recycler | 35 | 0.10 | 1 | 2.0 | hitscan | 0 | 0 |
| Grenade | 100 | 0.20 | 1 | 2.0 | lancee | 0 | 200 |

**Corps a corps (12 armes)** : Baseball Bat, Big Axe, Crowbar, Firefighter Axe, Katana, Knuckle
Knife, Military knife, Pipe, Shovel, Small Axe, Spear, Stun Baton.
**Fait a retenir : elles sont TOUTES identiques.** Degats 100, FireDelay 0.20, ReloadTime 2.0,
`ComboResetTimer` 2.0 partout (sauf Baseball Bat et Knuckle Knife a 50.0, et Pipe a 5.0). Aucune
n'est differenciee autrement que par son mesh, ses montages d'attaque et son prix (45 a 2500).
C'est la plus grosse opportunite de contenu de ce document : les modules de melee ne "boostent" pas
une arme deja typee, **ils creent le typage**.

**Taxonomie reelle deja presente** (champ `UseTags`, un `TSet<FName>` sur chaque `IDA_`) :
`Weapon`, `MelleeWeapon` (l'orthographe du projet, deux L, ne pas la corriger), `NASH`, `Sniper`,
`Grenade`, `Recycler`, `Tools`, `WeaponAccessory`, `Scope`, `Muzzle`, plus les emplacements
`1stWeapon`, `2ndWeapon`, `3rdWeapon`, `4thWeapon` (les deux derniers = arme de poing : NASH Pistol,
NashV2 Pistol, Knuckle Knife, Military knife).

**Familles de munitions existantes** : 9x19mm, Rifle, SMG, Shotgun, Sniper, Grenade, Rocket, Nash.

**Accessoires deja en jeu, a ne PAS refaire en module** : `Scope` (HoloDot, Scope 8x) et `Muzzle`
(Muzzle Brake, Suppressor), montes depuis l'inventaire par le systeme d'attachments de
`ItemScriptBase`. Vocabulaire d'emplacements deja ecrit : `Grip`, `Magazine`, `Muzzle`, `Rail`,
`Scope`. **Mesure importante : un seul asset les declare, `IDA_Sniper`.** Toutes les autres armes
ont `AttachmentSlots` vide.

> **Regle de coexistence a tenir** : un accessoire est une **piece visible** montee depuis
> l'inventaire, qui occupe un emplacement physique. Un module est une **carte interne** montee a
> l'etabli, invisible, qui suit l'exemplaire. Un module qui donne une optique ou un silencieux a
> donc du sens precisement parce qu'il **libere** le rail ou le canon.

### 0.4 Les boutons reels d'une arme (lus sur `WeaponScript` et ses enfants)

`Damage`, `FireDelay`, `BaseAmmo`, `CurrentAmmo`, `CurrentAmmoType`, `ReloadTime`,
`ReloadAnimation`, `IsAutoFireEnable`, `InfiniteAmmo`, `AutoReload`, `bUsesHitscan`,
`CrosshairSpreadScale`, `CrosshairBaseSize`, `CrosshairSpreadRecoverTime`, `TriggerTime`,
`AimpointTags` et `CurrentAimpointTag` (visee multi points deja native, avec `SetNextAimpoint`),
`DistanceFromCamera`, les 4 reglages `FireCameraShake*`, `IsFireBlocked`, `HandOffset`,
`HandGripOffset`, `AttachmentOverlayState`, et les dispatchers `OnWeaponHit` et `OnAmmoUpdate`.

Specifiques a la famille NASH (`IS_NASH_Base`) : `Projectile Pattern` (les assets
`DefaultPattern`, `PlusPattern`, `ShotgunPattern`, `XPattern` existent), `ProjectileSpeedKmH`
(2000 a 2100), `NashRecoilCurve`, `NashRecoilImprecision`, `SFX_Pitch`.

Specifiques a la melee (`MeleeWeaponBase`) : `AnimationIndex`, `ComboResetTimer`, `Emote DA`,
`EquippedAttachPointSocket`.

**La portee** est passee en dur (`Range=50000`) a `ServerFireBullet` dans `PreImplementFire` :
c'est un levier existant, juste fige.

**Ce qui n'existe pas** et qu'il ne faut donc pas promettre a la legere : la perforation (un seul
impact par trace hitscan), les statuts (brulure, gel, etourdissement), les coups critiques (seul le
headshot existe), la surchauffe, l'endurance, la durabilite d'arme, et un systeme de recul generique
(seule la famille Nash porte une courbe et une imprecision authored ; ailleurs le recul visible est
un camera shake cosmetique).

### 0.5 L'etabli d'artefacts : ce qui existe deja, et ce qui manque

C'est le point de montage des modules d'arme, et il est **deja construit a 80 %**.

Ce qui existe et compile :

- `AQModule_WorkbenchActor` : acteur replique, mesh plus zone d'interaction, avec
  `QMOD_OpenWorkbench(PlayerController)` et `QMOD_IsPawnInRange(Pawn)`. Portee d'interaction 350 cm,
  pertinence reseau reglee a 5000 m (`WorkbenchNetCullM`).
- `UQModule_WorkbenchWidgetBase` : UI **entierement native**, elle construit sa propre racine, donc
  aucun asset n'est requis pour qu'elle fonctionne. Trois colonnes : equipement du joueur, rack de
  l'objet selectionne (sockets, niveau, boutons phase et retrait), modules en inventaire. Un BP
  enfant peut la rehabiller via `Settings.WorkbenchWidgetClass`.
- Canal serveur complet : 4 RPC sur le rack du PlayerState (`SV_Item_InstallModule`,
  `SV_Item_RemoveModule`, `SV_Item_InsertPhase`, `SV_Item_RemovePhase`), qui delegue a
  `QModuleItemRack`. Le client n'appelle jamais une fonction serveur en direct.
- Harnais : `qmodule.Test.Workbench.Spawn` pose un etabli devant le pawn, `Workbench.Open` ouvre
  celui a portee.

Ce qui manque pour les deux facons de l'obtenir que tu decris :

1. **Pose par le joueur dans sa base** : l'entree existe deja. `QBD_QModule_Workbench` est bien un
   `UQBuilder_Data_ActorData` qui pointe sur `QModule_WorkbenchActor` (verifie sur le fichier).
   **Mais elle n'est inscrite nulle part** : le nom `QBD_QModule_Workbench` n'apparait pas dans
   `QBuilder_Qanga_ActorData.uasset`, donc l'etabli **n'est pas dans le menu de construction**. Il
   reste a lui donner un identifiant dans la map `InputData` de ce catalogue, plus son cout en
   ressources, sa vie de structure et son mesh fantome de placement. La persistance de l'acteur
   construit est ensuite **portee par QBuilder**, sans une ligne cote QModule.
2. **Trouve dans l'univers** : c'est simplement l'acteur pose par le level design dans un bunker,
   une ruine, un relais ou une station. Les profils de loot de modules existent deja et couvrent
   exactement ces lieux (`LDA_QMLoot_Common` pour bunker, ruines et industriel,
   `LDA_QMLoot_Relay`, `LDA_QMLoot_Market` pour le warp). Un etabli trouve devient un point
   d'interet naturel : le joueur y monte ce qu'il vient de looter, sur place.
3. **Le branchement de l'interaction maison** sur `QMOD_OpenWorkbench` (un noeud).
4. **Le domaine vehicule** : `QModule_WorkbenchWidgetBase.cpp:425` passe `EQModule_Domain::Weapon`
   **en dur**. L'etabli est donc armes seulement aujourd'hui. C'est coherent avec ta decision de
   faire passer les vehicules par le garage, mais il faut le savoir : l'etabli ne montera jamais un
   module de vehicule tant que cette ligne est figee.

---

## 1. MODULES D'ARMES

### 1.0 Comment le filtrage par type sera cable

`TargetFilter` est un `FGameplayTagQuery`, or les armes portent aujourd'hui des `UseTags` qui sont
des `FName`, pas des GameplayTags. Il faut donc **une passe de tags sur les 30 `IDA_` d'armes**
(une seule ecriture par arme) avant que le filtrage par type fonctionne. Taxonomie proposee, calquee
sur le reel et non inventee :

```
Weapon.Class.Firearm | Weapon.Class.Melee | Weapon.Class.Thrown | Weapon.Class.Tool
Weapon.Family.Pistol | Smg | AssaultRifle | MachineGun | Shotgun | Sniper
                     | GrenadeLauncher | RocketLauncher
Weapon.Fire.Hitscan  | Weapon.Fire.Projectile
Weapon.Ammo.Pistol   | Rifle | Smg | Shotgun | Sniper | Grenade | Rocket | Nash
Weapon.Brand.Nash    | Weapon.Brand.NashV2 | Weapon.Brand.Surplus
Melee.Type.Blade     | Blunt | Pierce | Shock | Tool
```

Un module universel a un `TargetFilter` vide. Un module "toutes armes a feu" filtre sur
`Weapon.Class.Firearm`. Un module de pompe filtre sur `Weapon.Family.Shotgun`. Aucun code nouveau :
c'est deja ce que le champ sait faire.

---

### 1.1 Universels, toutes armes (feu ET corps a corps)

`TargetFilter` vide.

| Module | T | Effet 1 / 2 / 3 | Levier reel | Etat |
|---|---|---|---|---|
| **Amplificateur de degats** (QMD existe) | P | Degats +5 / +10 / +15 % | `Damage` lu par `PreImplementFire` puis passe a `ServerFireBullet` | OK |
| Servos de manipulation | P | Changement d'arme +15 / +30 / +45 % | `Stat.Cyborg.Swap.TimeMult` existe deja (module `ReflexesCables`, aujourd'hui inerte) ; equipement dans `InventoryComponent` | PARTIEL |
| Reducteur de signature | P | Bruit et niveau de recherche generes au tir -20 / -40 / -60 % | `Stat.Cyborg.Stealth.PoliceTrackingNoiseCm` existe ; QPolice et la perception QAI sont les consommateurs | PARTIEL |
| **Recycleur de douilles** (QMD existe) | P | +1 / +2 matiere par tir ou par coup porte | `AddOrRemoveMatterToActor` appele dans `PreImplementFire` cote serveur | OK |
| Marqueur d'impact | C | La cible touchee est marquee 5 / 8 / 12 s pour l'escouade | `OnWeaponHit` (dispatcher existant) + `QModule_TrackerBridge` (deja ecrit pour les balises) | OK |
| Fleau des machines (variante arme) | P | +10 / +20 / +35 % de degats sur drones, tourelles et vehicules | `CombatComponent.Faction`, deja lu par reflexion par `QWeaponBulletSubsystem` pour le tir ami | OK |
| Chasseur de primes (variante arme) | P | +10 / +20 / +35 % sur Voss et hors-la-loi | Meme levier ; le pendant cyborg `Damage.VsOutlawMult` est deja lu par `Lib_Life` | OK |
| Stabilisateur inertiel | P | Secousse de tir -30 / -50 / -70 % (confort de visee) | Les 4 reglages `FireCameraShake*` sont deja authored par arme | OK |
| Verrou biometrique | C | L'arme ne tire pas pour un autre joueur ; revient au proprietaire a la mort | Proprietaire de l'instance d'item, deja connu du rack | PARTIEL |
| Balise de recuperation | C | Arme perdue signalee sur la carte 5 / 10 / 20 min | `Stat.Cyborg.Tracker.DeathBeacon*` (3 tags deja enregistres) | PARTIEL |
| Journal de bord | P | Compteurs par exemplaire (tirs, kills, precision) affiches sur la fiche | Le systeme de stats QANGA existe (34 compteurs BP) | PARTIEL |
| Module vampirique (Voss) | C | +2 / +4 / +6 PV par kill ; **contrepartie** -1 PV toutes les 10 s hors combat | `OnWeaponHit` + `Lib_Life` ; la contrepartie passe par `Drawbacks` | PARTIEL |
| Baie d'extension | P | +1 emplacement de module sur cet exemplaire | Le rack d'objet est aujourd'hui **sans capacite** (slots lineaires). Il faut d'abord creer la notion de capacite | NEUF |

### 1.2 Universels, armes a feu

`TargetFilter = Weapon.Class.Firearm`.

| Module | T | Effet 1 / 2 / 3 | Levier reel | Etat |
|---|---|---|---|---|
| **Chargeur rapide** (QMD existe) | P | Cadence +10 / +20 / +30 % | `FireDelay`, lu par la macro `AutomaticLoopFireDelay`. Semantique figee : le tag porte une **fraction de reduction du delai**, en `Op Add`, plafonnee a 0.8 (plafond deja pose dans `QModule_Settings.cpp:315`). **L'asset actuel est en `Multiply`, donc inerte : a corriger** | OK |
| Chargeur etendu | P | Chargeur +20 / +40 / +60 % | `BaseAmmo` | OK |
| Chargeur tandem | P | Rechargement +15 / +30 / +45 % | `ReloadTime` | OK |
| Ejecteur rapide | C | Rechargement possible en sprint, sans annulation | Montage de rechargement et gates ALS | PARTIEL |
| **Canon renforce** (QMD existe) | P | Degats +10 / +20 / +30 en valeur plate | `Damage`. **C'est ce que l'asset fait aujourd'hui**, malgre son nom | OK |
| Canon raye | P | Portee +25 / +50 / +100 % | `Range` (50000) passe en dur a `ServerFireBullet` : le rendre variable est une ligne. C'est le module que le nom "canon renforce" laissait attendre | OK |
| **Chambre thermique** (QMD existe, Voss) | P | Degats +20 / +35 / +50 % ; **contrepartie** portee -15 % | Meme leviers ; les `Drawbacks` du QMD sont **vides aujourd'hui**, a saisir | OK |
| Compensateur de recul | P | Recul -15 / -30 / -45 % | `NashRecoilCurve` et `NashRecoilImprecision` existent sur la famille Nash. Ailleurs il n'y a pas de recul de simulation | PARTIEL |
| Pointeur laser | P | Dispersion a la hanche -20 / -35 / -50 % | `CrosshairSpreadScale` et `CalcAimingAndTriggerFire` | OK |
| Ressort de rappel | P | Retour au centre du reticule +30 / +50 / +70 % | `CrosshairSpreadRecoverTime` | OK |
| Optique integree | C | Zoom de visee sans occuper le rail ; +1 niveau de zoom | La visee multi points est native (`AimpointTags`, `SetNextAimpoint`, `CurrentAimpointTag`) | OK |
| Silencieux integre | C | Aucun son monde au tir, detection IA reduite ; **libere l'emplacement Muzzle** | L'asset `IS_Supressor` existe deja comme accessoire ; la partie IA passe par la perception QAI | PARTIEL |
| Culasse allegee | P | Entree et sortie de visee +20 / +35 / +50 % | Alphas de visee de `QWeaponAnimInstance`, `TriggerTime` | PARTIEL |
| Selecteur de tir | C | Bascule automatique, semi, rafale de 3 | `IsAutoFireEnable` existe ; la rafale est a ecrire dans la macro de tir | PARTIEL |
| Fabricateur de munitions (variante arme) | C | 1 munition regeneree toutes les 10 / 6 / 3 s | `CurrentAmmo` et `OnAmmoUpdate` | OK |
| Reserve de secours | C | Au chargeur vide, une recharge instantanee gratuite par combat | `AutoReload` et `CurrentAmmo` | OK |
| Munitions perforantes | P | Traverse 1 / 2 / 3 corps alignes | **N'existe pas** : `LineTraceMultiByChannel` s'arrete au premier impact valide. Chantier C++ `QWeaponBulletSubsystem` (multi impact + attenuation) | NEUF |
| Munitions EMP | P | +25 / +40 / +60 % sur drones et vehicules, electronique coupee 2 s | Bonus par faction : OK. Le "coupe l'electronique" est un statut a creer | NEUF |
| Munitions incendiaires | P | Degats sur la duree, 5 / 8 / 12 par seconde pendant 4 s | Aucun systeme de statut ne durcit les degats dans le temps | NEUF |
| Munitions cryogeniques | P | Ralentit la cible de 20 / 35 / 50 % pendant 2 s | Idem statuts | NEUF |
| Detecteur de failles | P | Coup critique +3 / +6 / +9 % | Seul le headshot existe. A creer dans le hitscan | NEUF |
| Ventilation forcee | P | Surchauffe ralentie | **Il n'y a pas de surchauffe.** A ecarter tant qu'aucune arme energie n'en a une | NEUF |

### 1.3 Pistolets et armes de poing

`Weapon.Family.Pistol`. Concerne NASH Pistol et NashV2 Pistol (emplacements 3 et 4).

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Degainage eclair | P | Sortie d'arme de poing 40 / 60 / 80 % plus rapide | Equipement `InventoryComponent` | PARTIEL |
| Canon long | P | Portee +50 %, dispersion -20 % | `Range`, `CrosshairSpreadScale` | OK |
| Chargeur tandem lourd | P | Rechargement -40 % | `ReloadTime` | OK |
| Crosse pliante | P | Dispersion en deplacement -35 % | `CrosshairSpreadScale` | OK |
| Tir instinctif | C | Les 3 premiers tirs apres un changement d'arme font +25 % | `TriggerTime`, compteur local | PARTIEL |
| Double detente (Voss) | C | 10 % de chance de tir double ; **contrepartie** dispersion +25 % | Macro de tir | PARTIEL |
| Main gauche | C | Autorise un second pistolet a l'emplacement 4 | Emplacements `3rdWeapon` et `4thWeapon` existent deja ; l'animation double arme n'existe pas | NEUF |

### 1.4 Pistolets mitrailleurs

`Weapon.Family.Smg`. NASH SMG, NashV2 Smg.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Culasse ouverte | P | Cadence +20 % ; **contrepartie** dispersion +10 % | `FireDelay`, `CrosshairSpreadScale` | OK |
| Chargeur tambour | P | Chargeur +50 % ; **contrepartie** rechargement +20 % | `BaseAmmo`, `ReloadTime` | OK |
| Amortisseur de poignet | P | Dispersion en rafale -35 % | `CrosshairSpreadScale`, `CrosshairSpreadRecoverTime` | OK |
| Mode assaut | P | +20 % de degats a moins de 15 m | La distance d'impact est connue au moment du trace | OK |
| Cadence progressive | C | +15 % de cadence apres 1,5 s de tir continu | `TriggerTime` | PARTIEL |

### 1.5 Fusils d'assaut

`Weapon.Family.AssaultRifle`. AT56, AK47, FA62, MZ56, NashV2 Assault Rifle.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| **AT56** (QMD existe, module de base) | P | +30 / +20 / +20 de degats et de cadence | Deja en jeu, migre du niveau par type vers le rack par exemplaire | OK |
| Rafale controlee | C | Les 2 premiers tirs partent sans dispersion | `CrosshairSpreadScale` | OK |
| Canon lourd | P | Degats +10 % ; **contrepartie** cadence -8 % | `Damage`, `FireDelay` | OK |
| Bipied deployable | C | A l'accroupi ou appuye, recul et dispersion -60 % | L'asset `S_Bipod_SM` existe (Sniper) ; l'etat accroupi vient d'ALS | PARTIEL |
| Selecteur 3 coups | C | Rafale de 3 calibree | Macro de tir | PARTIEL |
| Lance-grenades sous-canon | A | Tir secondaire : une grenade toutes les 20 s | `E_WeaponTriggers` existe, l'arme lance-grenades existe, la seconde gachette est deja un concept du projet | PARTIEL |
| Munitions surchargees (Voss) | P | Degats +25 % ; **contrepartie** chargeur -25 % | `Damage`, `BaseAmmo` | OK |

### 1.6 Mitrailleuses

`Weapon.Family.MachineGun`. NashV2 MachineGun (120 balles, delai 0.10).

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Bande de munitions | P | Chargeur +40 / +80 / +120 | `BaseAmmo` | OK |
| Montee en regime | C | La cadence gagne 10 / 18 / 25 % apres 2 s de tir continu | `TriggerTime`, `FireDelay` | PARTIEL |
| Canon de rechange | P | Rechargement -50 % | `ReloadTime` | OK |
| Trepied | C | A l'arret et accroupi : dispersion -80 %, mais deplacement bloque | `CrosshairSpreadScale` + gate ALS | PARTIEL |
| Suppression | C | Les IA touchees a proximite se mettent a couvert | Comportements QAI, hook a ecrire | NEUF |
| Refroidisseur | P | Surchauffe | Pas de surchauffe dans le projet | NEUF |

### 1.7 Fusils a pompe

`Weapon.Family.Shotgun`. Shotgun, NASH Shotgun, NASH Shotgun Intercept, NashV2 Shotgun.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Choke resserre | P | Gerbe -20 / -35 / -50 % (portee utile allongee) | `CrosshairSpreadScale` cote hitscan, `Projectile Pattern` cote Nash | OK |
| Chevrotine lourde | P | Degats +25 % ; **contrepartie** gerbe +20 % | `Damage` + dispersion | OK |
| Chargeur tubulaire | P | +4 / +8 / +12 cartouches | `BaseAmmo` | OK |
| Recharge par deux | C | Rechargement -40 % | `ReloadTime` et montage | OK |
| Canon scie | P | Degats +35 % a moins de 8 m ; **contrepartie** portee -40 % | Distance d'impact et `Range` | OK |
| Balle unique | C | Passe d'une gerbe a un projectile unique longue portee | `Projectile Pattern` (assets `DefaultPattern`, `ShotgunPattern`, `XPattern`, `PlusPattern` deja crees) | OK |
| Souffle | C | Repousse les cibles legeres touchees a bout portant | Impulsion physique sur le pawn | PARTIEL |

### 1.8 Fusils de precision

`Weapon.Family.Sniper`. Sniper, NASH Sniper, NashV2 Sniper. **Rappel : `IDA_Sniper` est la seule
arme du jeu qui declare des emplacements d'accessoires** (Grip, Magazine, Muzzle, Rail, Scope).

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Optique variable | C | Deux niveaux de zoom au lieu d'un | `AimpointTags` et `SetNextAimpoint` : la visee multi points est native | OK |
| Culasse polie | P | Cadence +15 / +25 / +35 % | `FireDelay` (2.0 s sur le Sniper : c'est enorme, le levier est lisible) | OK |
| Munition subsonique | P | Tir silencieux ; **contrepartie** degats -20 % | Son monde et perception QAI | PARTIEL |
| Bipied de precision | C | Immobile : dispersion et recul -70 % | `S_Bipod_SM` existe deja dans le dossier Sniper | PARTIEL |
| Telemetre balistique | P | +1 % de degats par tranche de 50 m au-dela de 100 m | La distance est connue au trace | OK |
| Marquage a la touche | C | La cible touchee reste marquee 8 s pour l'escouade | `QModule_TrackerBridge` | OK |
| Perforation | P | Traverse le premier corps | Multi impact a ecrire dans le hitscan | NEUF |
| Retenue de souffle | A | 3 s de visee parfaitement stable, une fois par rechargement | Pas d'endurance, mais un simple timer local suffit | PARTIEL |

### 1.9 Lance-grenades

`Weapon.Family.GrenadeLauncher`, tir projectile. NashV2 Grenade Launcher (70 de degats, 7 coups).

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Barillet etendu | P | +2 / +4 / +6 grenades | `BaseAmmo` | OK |
| Charge a fragmentation | P | Rayon d'explosion +20 / +35 / +50 % | Le projectile et son rayon sont authored | OK |
| Grenade collante | C | La grenade colle a ce qu'elle touche | **`AQModule_StickyGrenadeActor` existe deja** dans QModule (salve dorsale) | OK |
| Amorce a impact | C | Explosion au contact au lieu du delai | Amorce du projectile | PARTIEL |
| Eclatement programme | C | Explose a la distance visee | Distance de visee connue | PARTIEL |
| Charge fumigene | A | Munition alternative non letale, rideau de fumee | FX a produire | NEUF |

### 1.10 Lance-roquettes

`Weapon.Family.RocketLauncher`. NashV2 Rocket Launcher (300 de degats, 4 coups),
`DA_RocketLauncherMissile` et `RocketLauncherMissileProjectile` existent.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Guidage | C | Verrouillage puis poursuite de la cible | `HomingLocker` existe dans `Systems/Weapon` ; **et le verrou multi cibles est deja ecrit en C++** dans QModule (ordonnance dorsale, acquisition en cone, retention, filtre de faction) | OK |
| Charge creuse | P | +40 / +60 / +100 % sur vehicules et structures | Faction et classe de la cible | OK |
| Double tube | C | Salve de 2 roquettes | Salve deja implementee cote QModule (round robin sur 16 cibles) | PARTIEL |
| Propulsion acceleree | P | Vitesse du missile +30 / +50 / +80 % | `ProjectileSpeedKmH` | OK |
| Deflecteur de souffle | P | Plus d'auto degats a bout portant | Filtrage de l'instigateur dans l'explosion | PARTIEL |
| Chargement rapide | P | Rechargement -30 / -45 / -60 % | `ReloadTime` | OK |

### 1.11 Famille NASH

`Weapon.Brand.Nash` (le tag `NASH` existe deja dans les `UseTags`). C'est le seul precedent de
module de famille en jeu, et la seule famille qui porte du recul et des motifs de projectile
authored.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| **All Nash Weapons** (QMD existe) | P | Cadence +10 / +20 / +40 %, degats +5 / +10 / +15 % sur toute la famille | Deja en jeu. La V2 respecte les pourcentages, la V1 a des tables en dur divergentes | OK |
| Stabilisateur Nash | P | `NashRecoilImprecision` -20 / -35 / -50 % | Donnee authored existante | OK |
| Accelerateur de projectile | P | `ProjectileSpeedKmH` +30 / +50 / +80 % | Donnee authored existante | OK |
| Convertisseur de gerbe | C | Change le motif de projectile (Default, Plus, X, Shotgun) | Les 4 assets `*Pattern` existent | OK |
| Livree Nash | C | Debloque une peinture d'arme | `NASH_Skin1_Customization` et `NASH_Skin2_Customization` existent | PARTIEL |

### 1.12 Outils armes

`Weapon.Class.Tool`. Recycler, Multitool, Salvage Prism, NanoWeaver Prism, plus Crowbar et Shovel
qui portent le `UseTag` `Tools`.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Rendement de recyclage | P | Matiere recuperee +10 / +20 / +35 % | `Stat.Cyborg.Recycle.YieldMult` **est deja lu** par `QMOD_GetRecycleYieldFactor` (InventoryComponent, ItemScriptBase) | OK |
| Rendement de minage | P | Minerai +10 / +20 / +35 % | `Stat.Cyborg.Mining.YieldMult` **est deja lu** par `RecyclerComponent` | OK |
| Faisceau elargi | C | Recycle 2 cibles a la fois | Trace du recycleur | PARTIEL |
| Portee du faisceau | P | +25 / +50 / +100 % | `Range` du trace | OK |
| Mode decoupe | C | +100 % de degats sur structures, 0 sur le vivant | Type de cible au trace | PARTIEL |

### 1.13 Corps a corps

`Weapon.Class.Melee`. **Point de depart mesure : les 12 armes sont identiques.** Les modules sont
donc l'outil de differenciation, et la premiere brique de contenu a valeur immediate.

**Universels melee**

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Affutage monomoleculaire | P | Degats +15 / +30 / +50 % | `Damage` (100 partout) et `Stat.Cyborg.Melee.DamageMult`, **deja lu** par `ALS_Base_CharacterBP` | OK |
| Contrepoids | P | Vitesse d'attaque +10 / +20 / +30 % | `FireDelay` (0.2) | OK |
| Fenetre de combo | P | `ComboResetTimer` +50 / +100 / +150 % | Variable existante (2 s par defaut) | OK |
| Enchainement | C | Le 3e coup d'un combo fait x2 | `AnimationIndex` | OK |
| Allonge | P | Portee du coup +10 / +20 / +30 % | Longueur du trace de melee | PARTIEL |
| Recuperateur cinetique | P | +1 / +2 matiere par coup porte | Meme voie que le recycleur de douilles | OK |
| Assassinat | C | Un kill au corps a corps ne declenche aucune alerte | Perception QAI et niveau de recherche QPolice | PARTIEL |
| Execution | C | Cible sous 20 % de vie : coup fatal | `Lib_Life` | PARTIEL |
| Riposte | A | Parade puis contre attaque | Aucune parade n'existe | NEUF |
| Saignement | P | Degats sur la duree apres un coup tranchant | Statuts inexistants | NEUF |

**Par sous type de melee** (necessite les tags `Melee.Type.*`)

| Sous type | Armes concernees | Modules proposes |
|---|---|---|
| Tranchant (`Blade`) | Katana, Big Axe, Small Axe, Firefighter Axe, Military knife, Knuckle Knife | Tranchant affute (degats +), Balayage (touche 2 cibles), Coup fendant (ignore une part d'armure) |
| Contondant (`Blunt`) | Baseball Bat, Pipe, Crowbar, Shovel | Masse d'inertie (+degats, -vitesse), Bris de structure (+100 % sur les constructions), Assommoir (etourdissement, NEUF) |
| Perforant (`Pierce`) | Spear | Allonge longue (portee ++), Charge (bonus apres un sprint), Empalement (immobilise, NEUF) |
| Electrique (`Shock`) | Stun Baton | Decharge EMP (drones et vehicules), Surtension (immobilise 1 s, NEUF), Batterie (nombre de charges) |
| Outil (`Tool`) | Shovel, Crowbar | Levier renforce (ouverture de caisses et de portes), Rendement de fouille, Pelletee (creuse plus vite) |

### 1.14 Grenades

`IDA_GrenadeBase` porte les `UseTags` `Consumable`, `Consumable1`, `Consumable2`, `Grenade`,
`Weapon`. C'est un **consommable empile**, pas un exemplaire unique : un rack par exemplaire n'a
pas de sens dessus. Deux voies possibles, a trancher :
1. Les bonus de grenade sont des **modules cyborg** (rayon, nombre porte, delai), sur le mur.
2. On cree un item "ceinture de grenades" a exemplaire unique qui porte le rack.

Recommandation : voie 1, elle ne coute aucune nouvelle mecanique.

---

## 2. MODULES DE VEHICULES

### 2.0 Cadre mesure

Le rack de vehicule est **deja pose automatiquement** par
`UQModule_VehicleRack_World_SubSystem` (hook `FOnActorSpawned`, serveur, tout acteur dont la chaine
de classes contient `VehicleBase_C`, adjacence desactivee, capacite illimitee, replique) : zero
Blueprint touche. **Limite v1 assumee : les racks de vehicule sont session seulement**, parce que
l'identite de sauvegarde d'un vehicule n'a jamais ete branchee au rack.

La possession existe deja et donne exactement le point d'ancrage du "garage" : composant
`PlayerVehiclesComponent` sur le PlayerState (`OwnedVehicles`, `VehiclesOnGarage`,
`CurrentOwnedVehicle`, persistes par DataObject), sortie par
`WorldVehiclesManager.SERVER_SetSpawnPlayerOwnedVehicle` et l'acteur `SpawnPlayerOwnedVehicle` (qui
persiste deja la transform sous la cle "T" et le carburant sous "F"), terminaux
`BP_TerminalVehicleSpawner` avec l'UI `PUW_SpawnVehicle`.

> Donc : **l'acteur de garage n'est pas a inventer**, c'est le terminal existant a enrichir d'un
> onglet "modules", exactement comme l'etabli a ete branche pour les armes. Et la persistance du
> rack doit s'accrocher a la cle du vehicule possede, pas a l'acteur spawne.

**Boutons reels d'un vehicule** (lus sur `VehicleBase`, `VehicleComponent` et
`FlyVehicleMovementComponent`) :

- Chassis : `VehicleLife`, `DMG_Factor`, `DMG_Curve`, `DMG_Factor_OnDestructible`,
  `KeepVelocityOnDestructible`, `DestroyTime`.
- Carburant : `Fuel` (**un octet, en POURCENTAGE 0 a 100**), `FuelConsumeDelaySeconds` (30 s),
  `FuelSamplingConsume`, `EnableFuelCharge`, `CoinsPerFuelPercent` (10).
- Vitesse : `MaxSpeedKmH` (3200 par defaut), `NormalSpeedLimitKmH`, `BoostSpeedLimitKmH`,
  `VehicleNormalSpeedLimitKmH`, `VehicleBoostSpeedLimitKmH`, `SpeedMultiplier` (replique).
- Sustentation : `HoverForce`, `HoverForceMultiplier`, `HoverExpForce`,
  `HoverGravityForceMultiplier`, `RideHeightOffset`, `bEnableMagneticHover`.
- Tenue de route : `LinearDamping`, `AngularDamping`, `IdleLinearDamping`, `IdleAngularDamping`,
  `DragPerAxis`, `AerodynamicsDragPerAxis`, `SideSlipFactor`, `ForcePerAxis`, `MassInKg`,
  `bEnableSlopeAntiSliding`, `bEnableOnRailsAtLowSpeed`, `bEnableAutoHoldAtLowSpeed`,
  `bUseSmartSteeringReversal`, `bAllowBackwardMove`.
- Pilotage : `LookScales`, `MaxLookScale`, `InterpLookSpeed`, `InputLookSmoothingSpeed`,
  `InputMoveSmoothingSpeed`, `MaxSpeedLookMultiplier`.
- Vol : `bIsAircraftVehicle`, `SpeedToMaxLiftKmH`, `AirborneSteeringFactor`,
  `AirborneOrientationPower`, `AirborneAngularDamping`, `AngularPowerByBoostMode`,
  `LateralPowerByBoostMode`, `VerticalPowerLimitations`, `GravityIntensity`,
  `GravityStabilizationForce`, `bEnableGravityStabilization`.
- Atterrissage : `LandingSpringStiffness`, `LandingSpringDamping`, `LandingAngularDampening`,
  `LandingDampeningStrength`, `LandingPredictiveWindow`, `TouchdownSideDragMultiplier`.
- Confort et vie de bord : `Vehicle_Speed_Control` avec `_Target`, `_Boost` et `_Enabled` (**un
  regulateur de vitesse existe deja**), `LightsState`, `ReplicatedBitMaskLight`, `BreakLight`,
  `DriveLight`, `IsHonkEnable`, `EnterDelay`, `ExitDelay`, `CameraSlots`, `MinSpringArmLength`,
  `MaxSpringArmLength`, `DriverLockedInteraction`, `MainDoorOpen`.
- Contenu : `InventorySlots`, `StorageComponentRef`.
- Armement : `Fire1Delay`, `Fire2Delay`, plus les emplacements `VWSlot_MachineGun`, `VWSlot_Rocket`,
  `VWSlot_Bomb`, `VWSlot_Flares`, `VWSlot_MegaSpotlight`, `VWSlot_Recycler`, geres par
  `VehicleWeaponManager`, `VehicleWeaponComponent` et `VehicleCombatComponent`.
- Etat : `E_VehicleState` (0 eteint, 1 en marche, 2 en panne, 3 detruit), avec le dispatcher
  `OnVehicleStateUpdate` qui est **la bonne source a ecouter** (jamais un timer de scrutation).

**Le piege a connaitre** : `Fuel` est un pourcentage, pas un volume. Un module "reservoir agrandi"
n'a donc **aucun levier** sans remodeler le carburant. Les leviers reels sont la consommation et le
prix du plein.

### 2.1 Taxonomie de ciblage

`VehicleBase` porte deja un champ **`VehicleTags`, qui est un `TSet<FName>`**, et il est deja
renseigne sur les classes de base. Mesure du 2026-09-04 :

| Classe de base | `VehicleTags` | `VehicleLife` | `InventorySlots` | Conso | Fire1 / Fire2 |
|---|---|---|---|---|---|
| `HovercraftBase` | `{"Hovercraft"}` | 1000 | 0 | 30 s | 1.0 / 1.0 |
| `SpaceshipBase` | `{"Spacecraft"}` | 10000 | 10 | 30 s | 1.0 / 1.0 |
| `WatercraftBase` | `{"Hovercraft"}` | 1000 | 0 | 30 s | 1.0 / 1.0 |

> **Dette reperee en passant, a signaler et pas a corriger ici** : `WatercraftBase` est tague
> `Hovercraft`, pas `Watercraft`. Tel quel, un module reserve aux bateaux irait aussi sur tous les
> vehicules a sustentation. A regler avant d'ecrire le moindre `TargetFilter` nautique.

Le vocabulaire existant est donc `Hovercraft` et `Spacecraft`. Proposition d'extension, dans le
meme esprit et sans casser l'existant :

```
Vehicle.Class.Hovercraft | Spacecraft | Watercraft | Bike | Rail
Vehicle.Role.Utility | Sport | Police | Cargo | Passenger | Wreck
Vehicle.Brand.Melrose | Etlas | Velkara | Icli | Nash
```

Deux voies possibles : mapper les `FName` existants vers des GameplayTags a la lecture, ou ajouter
un champ `VehicleGameplayTags` a cote. La premiere est plus propre et evite un doublon de verite.

Parc mesure : 21 cartes de vehicules terrestres (Melrose Citizen, Sport, SUV, Taxi, Van, Truck,
Trailer, PickUp, ArmedPickUp, Coopay, Riper, Sablone, Police, Berlin Police, CityBus, TrashTruck,
TruckArmored, Truck, Etlas Pawad, Etlas PodFury, BlackSync), 7 cartes de vaisseaux (Orizaune,
Rusty Orizaune, Velkara Explorer, Velkara Passenger, Old Velkara Passenger, Storm Travel,
Valrifle), plus bateaux, train et tram, moto, pods et navettes cote plugin QVehicles.

### 2.2 Universels, tous vehicules

| Module | T | Effet 1 / 2 / 3 | Levier reel | Etat |
|---|---|---|---|---|
| Blindage de chassis | P | Vie du vehicule +20 / +40 / +60 % | `VehicleLife` | OK |
| Amortisseurs de collision | P | Degats de collision subis -20 / -35 / -50 % | `DMG_Factor`, `DMG_Curve` | OK |
| Pare buffle renforce | P | Degats infliges aux destructibles +50 % | `DMG_Factor_OnDestructible` | OK |
| Reparateur automatique | C | Repare 1 / 2 / 4 PV par seconde hors combat | `Stat.Cyborg.Vehicle.RepairPerSec` **est deja un tag enregistre** | OK |
| Economiseur | P | Consommation -15 / -30 / -45 % | `FuelConsumeDelaySeconds`, `FuelSamplingConsume` | OK |
| Recharge negociee | P | Prix du plein -20 / -35 / -50 % | `CoinsPerFuelPercent` (10) | OK |
| Regulateur de croisiere | C | Maintien automatique de la vitesse | `Vehicle_Speed_Control_Enabled` **existe deja** | OK |
| Boite noire | C | Position du vehicule detruit signalee 10 min | `QModule_TrackerBridge` | OK |
| Verrou de proprietaire | C | Seuls le proprietaire et son escouade peuvent conduire | `DriverLockedInteraction` plus `PlayerVehiclesComponent` | OK |
| Soute agrandie | P | +2 / +4 / +6 emplacements de stockage | `InventorySlots`, `StorageComponentRef` | OK |
| Phares longue portee | C | Portee d'eclairage doublee, mode conduite de nuit | `LightsState`, `DriveLight` | OK |
| Sono de bord | C | Diffuse la QRadio a l'exterieur du vehicule | QRadio ; **et le bon patron est deja connu** : s'abonner a `VehicleComponent.OnVehicleStateUpdate`, jamais scruter | OK |
| Avertisseur module | C | Klaxon ou sirene personnalisable | `IsHonkEnable`, `UQVehicleSirenComponent` | OK |
| Embarquement rapide | P | Entree et sortie -30 / -50 / -70 % | `EnterDelay`, `ExitDelay` | OK |
| Camera tactique | C | +1 emplacement de camera (vue exterieure haute) | `CameraSlots`, `CurrentCameraSlot` | OK |
| Transpondeur de flotte | P | Rappel de flotte plus rapide de 20 / 35 / 50 % | `Stat.Cyborg.FleetRecall.CooldownSec` **est deja lu** par le rack | OK |
| Trousse de bord | A | Remet en marche un vehicule en panne, une fois par sortie | `E_VehicleState` etat 2 (en panne) vers 1 | OK |
| Balise de detresse | C | Signale a l'escouade un vehicule en panne ou detruit | `PawnAtVehicleDeath`, `OnVehicleStateEvent` | OK |
| Livree | C | Debloque une peinture | Materiaux de skin vehicule | PARTIEL |

### 2.3 Sol et sustentation

`Vehicle.Class.Ground` et `Hover`.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| **Noyau surcadence** (QMD existe, Voss) | P | Vitesse max +10 / +20 / +30 % ; **contrepartie** pannes moteur aleatoires | `MaxSpeedKmH` et `SetMaxSpeed`, plus `SpeedMultiplier`. Les `Drawbacks` sont a saisir | OK |
| Turbocompresseur | P | Vitesse max +8 / +16 / +25 % (variante IC Lab stable) | Idem | OK |
| Injecteur de boost | A | Palier de vitesse de boost +20 / +35 / +50 % | `BoostSpeedLimitKmH`, `VehicleBoostSpeedLimitKmH`, `bBoostInput` | OK |
| Coussin renforce | P | Portance +25 %, franchissement d'obstacles ameliore | `HoverForce`, `HoverForceMultiplier`, `HoverExpForce` | OK |
| Garde au sol reglable | C | Hauteur de vol stationnaire ajustable | `RideHeightOffset` | OK |
| Sustentation magnetique | C | Accroche les surfaces metalliques, murs compris | `bEnableMagneticHover` | OK |
| Antiderapage | P | Glisse laterale -20 / -35 / -50 % | `SideSlipFactor` | OK |
| Mode derive | C | Active la derive controlee | `bDriftModeInput`, `bHandBrakeInput` | OK |
| Direction sport | P | Reactivite de direction +20 / +35 / +50 % | `LookScales`, `InterpLookSpeed`, `MaxLookScale` | OK |
| Frein directionnel | C | Freinage vectorise en virage | `bDirectionalBrakingActive` | OK |
| Ancrage de pente | P | Ne glisse plus a l'arret en devers | `bEnableSlopeAntiSliding` | OK |
| Allegement de chassis | P | Masse -15 % (acceleration +, encaisse -) | `MassInKg` | OK |
| Stabilisateur gravitationnel | P | Tenue amelioree en gravite faible ou variable | `bEnableGravityStabilization`, `GravityStabilizationForce`, `GravityIntensity` | OK |
| Amortisseur d'atterrissage | P | Degats de reception -50 %, rebond supprime | `LandingSpringStiffness`, `LandingSpringDamping`, `LandingDampeningStrength` | OK |
| Detection de paroi | C | Evite les accrochages a grande vitesse | `bEnableWallProximityDetection` | OK |
| Assistance basse vitesse | C | Manoeuvres de precision, maintien en pente | `bEnableOnRailsAtLowSpeed`, `bEnableAutoHoldAtLowSpeed` | OK |
| **Hydroglisseur** (QMD existe) | P | Roule sur l'eau au dela de 70 / 45 / 25 km/h, controle 50 / 75 / 100 % | `Stat.Vehicle.Hydroplane.*` : les 2 tags existent mais **n'ont aucun lecteur** | PARTIEL |
| Traineau de poussiere | C | Effets de sol reduits (discretion) ou amplifies (style) | `bEnableDustEffects`, `DustEffectIntensityMultiplier` | OK |

### 2.4 Vol atmospherique et spatial

`Vehicle.Class.Aircraft` et `Spacecraft`. Rappel de l'echelle mesuree : 3200 km/h en croisiere,
6400 en boost.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Ailerons | P | Trainee aerodynamique -20 / -35 / -50 % | `AerodynamicsDragPerAxis`, `DragPerAxis` | OK |
| Portance amelioree | P | Vitesse de portance maximale atteinte plus tot | `SpeedToMaxLiftKmH`, `bIsAircraftVehicle` | OK |
| Propulseurs vectoriels | P | Poussee laterale et verticale +25 / +40 / +60 % | `ForcePerAxis`, `VerticalPowerLimitations`, `LateralPowerByBoostMode` | OK |
| Gyroscope de vol | P | Stabilite angulaire +30 / +50 / +70 % | `AirborneAngularDamping`, `AngularPowerByBoostMode` | OK |
| Assistance d'orientation | P | Redressement automatique plus vif | `AirborneOrientationPower`, `AirborneOrientationSpeed` | OK |
| Postcombustion orbitale | A | Palier de boost porte a 6400 km/h et au dela | `VehicleBoostSpeedLimitKmH` | OK |
| Bouclier de friction | P | Degats de friction atmospherique -50 / -75 / -100 % | La chaine existe et fait un pourcentage de la vie **maximale** par tick | OK |
| **Regulateur atmospherique** (QMD existe) | P | Consommation de vol -10 / -20 / -30 % | `Stat.Cyborg.Flight.FuelUseMult` : **arbitrage ouvert** entre le jetpack et le vehicule, tranche a faire | PARTIEL |
| Train d'atterrissage assiste | C | Pose douce, plus de derapage a la reception | `TouchdownSideDragMultiplier`, `TouchdownSideDragDuration`, `LandingPredictiveWindow` | OK |
| Pilote automatique | C | Maintien de cap et d'altitude | `Vehicle_Speed_Control` + maintien d'altitude a ecrire | PARTIEL |
| Radar de proximite | C | Blips des menaces et des astres autour du vaisseau | Patron du Scanner deja en jeu | PARTIEL |
| Ordinateur de saut | P | Prechauffe de warp plus courte | Depend du systeme de warp, non audite ici | NEUF |
| Amarrage assiste | C | Accostage automatique aux stations | `ShippingLandingArea` existe cote vaisseaux | NEUF |
| Antigrav de soute | P | La cargaison n'alourdit plus le vaisseau | Il n'existe pas de masse de cargaison | NEUF |

### 2.5 Nautique

`Vehicle.Class.Watercraft`. `WatercraftBase` et `E_VehicleOceanBehavior` existent.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Coque planante | P | Vitesse sur l'eau +20 / +35 / +50 % | `E_VehicleOceanBehavior`, `MaxSpeedKmH` | PARTIEL |
| Ballast | P | Stabilite en vague +50 % | `LinearDamping`, `AngularDamping` | PARTIEL |
| Helice renforcee | P | Acceleration +30 % | `ForcePerAxis` | PARTIEL |
| Sonar | C | Detection sous la surface | `bIsUnderwater` existe, le reste est a ecrire | NEUF |

### 2.6 Utilitaire et economie

`Vehicle.Role.Utility` et `Cargo`. Camion, van, remorque, bus, taxi, benne.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Conteneur renforce | P | Soute +6 / +10 / +16 emplacements | `InventorySlots` | OK |
| Compacteur embarque | P | Recyclage a bord, rendement +15 / +25 / +40 % | `Recycle.YieldMult` est deja lu | OK |
| Atelier mobile | C | Ouvre l'etabli de modules depuis le vehicule | `AQModule_WorkbenchActor` et son UI existent deja | OK |
| Attelage | C | Tracte la remorque Melrose | La remorque existe comme vehicule a part entiere | PARTIEL |
| Treuil | A | Remorque une epave ou un vehicule en panne | Le dossier `Vehicles/Epaves` existe ; la mecanique de traction est a ecrire | NEUF |
| Grue de chargement | A | Charge un conteneur lourd | Idem | NEUF |
| Comptoir mobile | C | Ouvre un point de vente depuis le vehicule | Le systeme de marchand existe (`BPI_Shop`, stocks, restock) | PARTIEL |

### 2.7 Armement de vehicule

Les emplacements existent deja et sont peuples : `VWSlot_MachineGun`, `VWSlot_Rocket`,
`VWSlot_Bomb`, `VWSlot_Flares`, `VWSlot_MegaSpotlight`, `VWSlot_Recycler`, avec leurs pawns de
tourelle, leurs widgets de retour et leur manager.

> **Regle de non doublon** : un module ne doit pas remplacer un `VWSlot`, il doit le **modifier**.
> L'installation d'une tourelle reste un emplacement d'armement ; le module ameliore ce qui est
> monte dessus.

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Servomoteurs de tourelle | P | Cadence des armes de bord +15 / +25 / +40 % | `Fire1Delay`, `Fire2Delay` | OK |
| Conduite de tir | P | Precision et portee des tourelles + | `VehicleWeaponComponent` | PARTIEL |
| Munitions guidees | C | Verrouillage sur cible pour les roquettes de bord | `HomingLocker` plus le verrou C++ QModule | OK |
| Chargeur de soute | P | Reserve de bombes ou de roquettes +50 % | `VWSlot_Bomb`, `VWSlot_Rocket` | PARTIEL |
| Distributeur de leurres | P | +2 / +4 leurres, recharge plus rapide | `VWSlot_Flares` et son `FlareActor` | OK |
| Projecteur de recherche | C | Portee et cone du `MegaSpotLight` augmentes | `VWSlot_MegaSpotlight` | OK |
| Recycleur de bord | P | Portee et rendement du recycleur embarque | `VWSlot_Recycler` | OK |

### 2.8 Furtivite, loi et hors la loi

| Module | T | Effet | Levier | Etat |
|---|---|---|---|---|
| Faux transpondeur (variante vehicule) | C | Plaque falsifiee : le vehicule n'est plus rattache au joueur recherche | Le QMD cyborg existe ; QPolice tient le niveau de recherche | PARTIEL |
| Camouflage thermique | P | Detection IA a l'arret -40 / -60 / -80 % | Perception QAI | PARTIEL |
| Brouilleur de poursuite | C | Empeche le marquage par la police pendant 10 s | `QPoliceSubsystem` et sa decroissance de recherche | PARTIEL |
| Gyrophares et sirene | C | Debloque le comportement de vehicule d'intervention | `UQVehicleSirenComponent` est complet (etats klaxon, sirene, rapide, bwoop, panne) | OK |
| Blindage pare balles | P | Degats d'armes a feu subis -20 / -35 / -50 % | A distinguer de `VehicleLife` : c'est une reduction, pas un pool | PARTIEL |
| Vitres occultees | C | Le conducteur n'est pas identifiable | Perception QAI et QPolice | PARTIEL |

---

## 3. Ce qu'il faut construire, dans l'ordre

### Etape 0 : le pont de stats (sans lui, rien de ce document n'a d'effet)

1. Exposer la lecture du rack d'exemplaire en Blueprint. Aujourd'hui `QModuleItemRack::GetStat`
   est une fonction C++ de namespace, appelee **uniquement** par une commande de test. Il faut un
   `QMOD_GetItemStat(Context, StatTag, BaseValue)` en `BlueprintPure` dans
   `UQModule_StatLibrary`, qui resout l'instance d'item depuis le contexte comme le fait deja
   `QMOD_GetStatForObject`.
2. Emettre le signal `OnItemRackChanged(ItemInstance)` a la fin de chaque mutation reussie de
   `QModuleItemRack` (deja specifie au paragraphe 8.2 de `QMODULE_ACTIVATION_ALIGNMENT.md`), pour
   qu'une arme modifiee a l'etabli n'attende pas un re-equipement.
3. Cinq lectures a poser dans `WeaponScript` : `Damage` dans `PreImplementFire`, `FireDelay` dans
   la macro `AutomaticLoopFireDelay`, `BaseAmmo` dans `ReloadAmmo`, `ReloadTime`, et la portee
   `Range` aujourd'hui en dur a 50000.

Avec ces trois points, **une vingtaine de modules universels de ce catalogue fonctionnent le jour
meme**, y compris les 5 `QMD_` d'arme qui existent deja et ne servent a rien aujourd'hui.

### Etape 0 bis : trois corrections d'assets, gratuites, a faire dans le meme mouvement

1. `QMD_ChargeurRapide` : passer `Stat.Weapon.FireRate` de `Multiply` a `Add` (valeurs inchangees).
   En `Multiply` sur une base de 0, le module reste inerte meme apres l'etape 0.
2. `QMD_ChambreThermique` : saisir les `Drawbacks` (portee -15 %), aujourd'hui vides alors que la
   fiche promet une contrepartie Voss.
3. `WatercraftBase` : corriger `VehicleTags` de `Hovercraft` vers `Watercraft`, sans quoi aucun
   filtre nautique ne sera fiable.

### Etape 1 : le ciblage par type

Passe de tags sur les 30 `IDA_` d'armes et sur les `VehicleTags`, puis remplissage de
`TargetFilter`. Une ecriture par asset, aucun code.

### Etape 2 : dispersion, recul, visee

`CrosshairSpreadScale`, `CrosshairSpreadRecoverTime`, `NashRecoilImprecision`, alphas de visee.
Debloque toute la colonne "precision" du catalogue.

### Etape 3 : cote vehicule

Adaptateur `MaxSpeedKmH` et consommation, puis persistance du rack sur l'identite du vehicule
possede (aujourd'hui session seulement), puis l'onglet modules du terminal de garage.

### Etape 4 : les mecaniques neuves, une par une

Perforation multi impact dans `QWeaponBulletSubsystem`, statuts (brulure, gel, choc), critiques,
parade au corps a corps, treuil. Chacune est un chantier autonome, aucune n'est requise pour livrer
les etapes 0 a 3.

---

## 4. Multiplicateur de contenu : les manufactures

Rappel de la regle deja actee dans `QMODULE_CATALOGUE.md` : une variante de manufacture, ce sont
**les memes `StatTags` avec d'autres valeurs et des `Drawbacks`**, donc zero code. Les deux tags
existent deja (`Manufacturer.ICLab`, `Manufacturer.Voss`). Proposition d'extension :

| Manufacture | Caractere | Exemple sur un meme role |
|---|---|---|
| IC Lab | Reference, stable, equilibre | Canon renforce : portee +50 % |
| Voss | Chiffres superieurs, contrepartie systematique | Canon renforce Voss : portee +90 %, dispersion +20 % |
| Nash Arms | Oriente cadence et projectiles | Canon renforce Nash : portee +40 %, vitesse de projectile +30 % |
| Surplus QPD | Defensif, connote loi, bon marche | Canon renforce QPD : portee +30 %, niveau de recherche genere -20 % |
| Artisanal pirate | Valeurs tirees au hasard, marche gris | Canon renforce pirate : portee +20 a +80 %, fiabilite variable |

Ce seul axe multiplie par cinq le nombre d'entrees reelles du catalogue sans une ligne de code.

---

## 5. Ce que je n'ai pas pu verifier

Honnetete de mesure, a lever avant mise en production :

- Les valeurs `Damage` lues valent 0 au CDO sur MZ56, Shotgun, NASH, NASH SMG et NASH Sniper : ces
  armes recoivent probablement leurs degats par le `Switch on Int` de phase au moment du
  `OnPhaseUpdate`. **Je n'ai pas ouvert ces graphes**, donc ne pas conclure que ces armes font zero
  degat.
- Le champ `CurrentAmmoType` vaut `NONE` sur tous les CDO d'armes : le type de munition est
  vraisemblablement affecte ailleurs (item ou graphe). A confirmer avant de proposer des modules
  qui ciblent une famille de munitions.
- `VehicleTags` a ete releve sur les **trois classes de base** (`Hovercraft`, `Spacecraft`, plus le
  mauvais tag du bateau), pas sur les 28 vehicules concrets : le chargement d'un Blueprint de
  vehicule complet fait abandonner la connexion du pont. La repartition Sport, Police, Cargo,
  Passenger du paragraphe 2.1 reste donc une proposition.
- Les valeurs par vehicule (vie, soute, prix, niveau requis) n'ont ete relevees que sur les classes
  de base. Les enfants peuvent les surcharger.
- Le systeme de warp n'a pas ete audite : le module "ordinateur de saut" est cite pour memoire.
