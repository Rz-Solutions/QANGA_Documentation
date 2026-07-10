# QMODULE : Audit d'alignement avant activation (Modules v2 vs jeu reel)

> **Statut : AUDIT TERMINE, 100 % LECTURE SEULE. Rien n'a ete modifie (ni asset, ni code, ni ini).**
> Redige le 2026-07-10 (session dediee demandee par RzZz). Methode : editeur ouvert + pont CLIScape
> (dumps CDO/SCS Python, get_detailed_blueprint_summary) + ripgrep binaire sur les .uasset + lecture
> des sources C++ des plugins. Chaque levier cite a ete VERIFIE au moteur ce jour ; les points non
> verifiables sont listes en section 9. Complement de `QMODULE_ARCHITECTURE.md` (checklist d'activation)
> et de `QMODULE_CATALOGUE.md` (triage).
>
> Perimetre : les 93 definitions QMD_* de /Game/Phases/QModuleV2 (7 legacy + 82 batch GARDER +
> 3 modules de test + Rappel de flotte), le pawn joueur et tous ses systemes consommateurs.

---

## 0. TL;DR : verdict

1. **L'architecture v2 est correctement branchable** : chaque stat chiffree du batch a un levier reel
   identifie (chemin + fonction + variable exacte ci-dessous), sauf les exceptions listees en 7.1.
2. **La bascule doit se penser en DEUX ETAGES** (section 2). Etage 1 (facade) : PhaseComponent lit le
   niveau depuis les racks v2, les valeurs de gameplay restent les Switch/Select des BP existants :
   AUCUNE derive de valeurs possible par construction. Etage 2 (stats agregees) : les consommateurs
   lisent QMOD_GetStat ; la les chiffres des QMD doivent etre EXACTS, et l'audit montre qu'ils ne le
   sont pas tous (AT56 notamment).
3. **Le legacy ment deja au joueur** : plusieurs effets decrits par les DA_Phase n'existent PAS dans
   l'implementation (Jetpack niveau 2 "+50% Fuel" : inexistant ; Drone niveau 2 "Flashbang" :
   inexistant ; descriptions AT56 fausses). Le batch v2 a herite de ces chiffres de DESCRIPTION et
   non des chiffres d'IMPLEMENTATION. Corrections chiffrees en 7.1.
4. **Aucun module GARDER n'est orphelin de systeme**, mais 9 modules reposent sur une mecanique a
   creer (7.3) et 2 reposent sur une refonte non triviale (contrats simultanes, PV du drone).

---

## 1. Conventions v2 rappelees (verifiees dans le C++ du plugin)

- Agregation par StatTag (QModule_Aggregation.h) : `Final = bHasOverride ? Override : (Base + SommeAdd) * ProduitMult` puis ClampMax eventuel. `Multiply` contribue en `x (1 + Valeur)`.
- `ValuePerLevel[N-1]` = valeur TOTALE au niveau N du module (semantique valeur-au-niveau, identique aux Select/Switch legacy). Pas de cumul entre niveaux.
- Lecture : `UQModule_StatLibrary::QMOD_GetStat(Target, StatTag, BaseValue)` (BlueprintPure ; resout le rack de l'acteur, sinon celui du PlayerState du pawn). Racks d'exemplaire : `QModuleItemRack::GetStat(ItemInstance, StatTag, BaseValue)`. Niveau d'un module : `UQModule_RackComponent::QMOD_GetModuleLevel(ModuleTag)`.
- Evenement : `UQModule_RackComponent::OnRackChanged` (BlueprintAssignable, sans payload).
- Vocabulaire Stat.* : 7 tags natifs (QModule_Tags.h) + 15 tags dans Config/Tags/QModuleTags.ini = 22 stats. Les 3 tags Stat.Cyborg.Build.* sont enregistres mais AUCUN module GARDER ne les utilise (modules QBuilder reportes au triage : coherent, voir 6.3).

## 2. La regle des deux etages (structure de l'activation)

**Etage 1 : la facade (checklist point 6a).** PhaseComponent.GetCurrentPhase et CallPhaseUpdate
redirigent la SOURCE DU NIVEAU vers les racks v2 (mur du PlayerState pour le cyborg, rack d'instance
pour les armes). Les Switch on Int / Select des BP d'items restent la source des VALEURS (degats,
delais, rayons, couts). Consequence forte : a l'etage 1, les erreurs de chiffrage des QMD n'ont AUCUN
effet en jeu ; la continuite jour-1 est garantie par construction si la conversion des niveaux est
correcte (section 8.4).

**Etage 2 : le branchement des stats agregees (checklist point 6b, progressif).** Consommateur par
consommateur, les valeurs en dur sont remplacees par des lectures QMOD_GetStat. C'est SEULEMENT ici
que les StatMods des QMD deviennent la verite : ils doivent d'abord etre alignes sur les valeurs
REELLES (7.1). L'etage 2 peut se faire module par module, apres l'activation, sans big-bang.

Le mur v2, les items Phase, la consommation, la persistance et l'etabli sont deja E2E-valides
(voir QMODULE_ARCHITECTURE.md 15.6-15.9) : cet audit ne les re-teste pas, il aligne le CONTENU.

## 3. Les 7 modules legacy : realite d'implementation vs QMD (continuite jour-1)

Valeurs relues ce jour dans les DA ET dans les graphes consommateurs.

| Module | DA legacy (MaxPhase) | Implementation REELLE verifiee | QMD v2 actuel | Verdict |
|---|---|---|---|---|
| General System | P_GeneralSystem (2) | ALS_Base_CharacterBP `On Phase Update` -> `SS_Matter.SetMaxMatter(Select [P0=100, P1=300, P2=500])` | Matter.Max Add [200,400], max 2, BASE+CORE | **OK** avec Base=100 (100+200=300, 100+400=500). ATTENTION : BaseMatterMax=1000 du CyborgAdapter est un placeholder FAUX, la base reelle est 100 |
| Jetpack | P_Jetpack (2) | IS_JetPack : gates `GetCurrentPhase > 0` debloquent vol stationnaire (`AllowStationary`) ET vol rapide (`SetFastFlightMode`). Fuel : MaxFuel=100, conso 1/0.1s (stationnaire /4), regen 1/tick apres 3 s si Matter>0. **Le +25/+50 % de fuel N'EST PAS IMPLEMENTE (aucun Set MaxFuel nulle part)** | Jetpack.FuelMax Mult [0.25,0.5], max 2, BASE | **ECART** : le QMD promet un effet qui n'a jamais existe en jeu. Decision RzZz : (a) l'implementer enfin a l'etage 2 (buff assume), ou (b) vider le StatMod et garder les gates comportementales. Les gates niveau 1 restent des behaviors (pas des stats) |
| Drone | P_Drone (2) | IS_DroneBase : PAS de PV ; modele a IMPACTS : Switch(GetCurrentPhase) P0=detruit au 1er hit, P1=1 hit encaisse (fenetre 10 s), P2=2 hits (RetriggerableDelay 10 s). Auto-reparation Select [30,20,10] s. Destroyed (RepNotify) -> FPS force. **Flashbang niveau 2 : INEXISTANT** | coquille (0 StatMod), max 2, BASE | **OK etage 1** (valeurs dans le BP). Etage 2 : voir BlindageDeDrone/NanoReparateur (5.2). Flashbang : effet fantome a retirer des descriptions ou a creer |
| Repair | P_Repair (2) | IS_RepairBase : Heal `Select [40,60,80,100]`, cout Matter `Select [50,40,30,25]`, cooldown `Select [30,25,20,15]` s, indexes par GetCurrentPhase. Repare SS_PhysicalState via ApplyHealToActor, RPC SV_Repair | coquille, max 2, BASE | **OK etage 1**. Nota : des valeurs phase 3 existent dans les Select alors que MaxPhase=2 (marge de progression prete) |
| Scanner | P_Scanner (3) | IS_Scanner : rayon `Select [100000,300000,600000,1000000]` cm (1/3/6/10 km) x `VehicleMultiplier=2.0` en vehicule ; cooldown `Select [30,20,10,5]` s ; detection par branches Sequence (IA gate phase>1, vehicules, joueurs (branche commentee "dont work"), items monde, QAI, ScannableComponent, points QLevel) | coquille, max 3, BASE | **OK etage 1**. La branche joueurs est buggee AUJOURD'HUI (commentaire dans le graphe) : la promesse "Detect Players" de P2 est douteuse en l'etat |
| AT56 | P_AT56 (3), descriptions "+30/+20/+20 Damage + FireRate" | IS_AT56 `On Phase Update` -> Switch on Int : Damage 30 -> 40/45/50 ; FireDelay 0.17 -> 0.16/0.15/0.14 s | Weapon.Damage Add [30,50,70], max 3, BASE. PAS de mod FireRate | **DOUBLE ECART** : (1) les valeurs Add du QMD suivent la DESCRIPTION (+30/+50/+70) alors que le reel est +10/+15/+20 ; (2) le mod cadence manque. Fix 7.1 |
| All Nash Weapons | P_Nash (3), "FR 10/20/40 %, DMG 5/10/15 %" | DEUX implementations : NashV2 (8 armes, IDA_NashV2_*) : delay x0.90/x0.80/x0.60 et degats x1.05/x1.10/x1.15 (les % delai/degats sont respectes) ; Nash V1 (IS_NASH_Base/IS_NASH_SHOT, IDA_NASH) : tables EN DUR par module (ex. Rifle 0.15/35 -> 0.09/49) qui NE respectent PAS les % | Weapon.FireRate Mult [0.1,0.2,0.4] + Weapon.Damage Mult [0.05,0.10,0.15], max 3, BASE | **OK cote degats V2** (x1.05 = Mult 0.05 exact). **ECART semantique cadence** : "FireRate 10 %" signifie ici "delai reduit de 10 %" (voir 4.1). V1 : divergence assumee a documenter (l'etage 1 la preserve telle quelle) |

Assets IDA relies aux phases (14 sur 287 IDA au total) : IDA_AT56 -> P_AT56 ; IDA_NASH + les 8
IDA_NashV2_* -> P_Nash ; IDA_DroneBase -> P_Drone ; IDA_Jetpack -> P_Jetpack ; IDA_ScannerBase ->
P_Scanner ; IDA_RepairBase -> P_Repair. P_GeneralSystem n'est lie a aucun IDA (porte par les
characters via le default SCS du PhaseComponent).

## 4. Semantiques de stats a FIGER avant l'etage 2

### 4.1 Stat.Weapon.FireRate = fraction de REDUCTION DU DELAI entre tirs
L'implementation reelle (NashV2 : delay x0.90/x0.80/x0.60 ; AT56 : 0.17 -> 0.16/0.15/0.14) montre que
"FireRate +10 %" au sens QANGA = "FireDelay x (1 - 0.10)". Convention recommandee :
- Le module publie la fraction en **Op Add** (les fractions s'additionnent entre modules).
- L'adaptateur applique : `FireDelay_final = FireDelay_base * (1 - Clamp(Somme, 0, 0.8))`.
- **QMD_Nash : passer le mod FireRate de Multiply a Add** (un Multiply lu sur une Base 0 rend 0 :
  inutilisable pour une fraction). Valeurs [0.1,0.2,0.4] inchangees.
- QMD_AT56 : ajouter FireRate Add [0.0588,0.1176,0.1765] (= 1 - 0.16/0.17, 1 - 0.15/0.17, 1 - 0.14/0.17).
- Poser un **ClampMax global 0.8** sur ce tag des le premier batch d'equilibrage (regle catalogue par.6).

### 4.2 Stat.Cyborg.Drone.HealthMult : MAL NOMME, le drone n'a pas de PV
Le drone 3P encaisse des IMPACTS (0/1/2 selon la phase), pas des degats sur un pool. Deux options :
(a) renommer la mecanique : le module ajoute des IMPACTS encaisses (stat entiere Add, ex. +1 hit) ;
(b) donner de vrais PV au drone (mecanique nouvelle, plus lourde, mais ouvre DroneSpectre/Leurre).
Recommandation : (a) pour l'activation, (b) en chantier drone ulterieur.

### 4.3 Reductions en pourcentage (FallDamage.Reduction, Drone.RepairTimeMult, Flight.FuelUseMult)
Le batch mixe deux ecritures : FallDamage.Reduction Add [30,60,100] (pourcents entiers) et
RepairTimeMult/FuelUseMult Mult negatifs [-0.2...]. Les deux marchent mais FIGER l'ecriture par tag
dans un commentaire de l'Editeur de Modules : les tags `*.Mult` en Multiply negatif (le x(1+v) reduit),
les tags `*.Reduction` en Add de POINTS DE POURCENTAGE consommes par l'adaptateur en /100. Melanger
les deux styles sur un meme tag casserait l'agregation multi-modules.

### 4.4 Base a fournir par l'adaptateur (valeurs relevees)
- Matter.Max : Base = **100** (pas 1000 : corriger BaseMatterMax du CyborgAdapter ou alimenter la base depuis SS_Matter).
- Move.SprintSpeed : Base = 700 (DT_MovementModelCyborg row Cyborg, Sprint), applique x0.85 en aval (sprint effectif ~595). Le mod doit s'inserer AVANT ce x0.85 pour rester coherent.
- Jetpack.FuelMax : Base = 100 (IS_JetPack.MaxFuel).
- Weapon.Damage / FireRate : Base = variables BP `Damage` / `FireDelay` de l'exemplaire (CDO par arme, ex. AT56 30 / 0.17).
- Vehicle.Speed.Max : Base = `MaxSpeedKm/h` du FlyVehicleMovementComponent (3200 par defaut).

## 5. Table d'alignement : les 22 definitions chiffrees

Legende Etat : OK = levier verifie + chiffres coherents ; ECART = levier verifie mais chiffres/semantique a corriger ; PARTIEL = levier existant a etendre ; SANS LEVIER = mecanique a creer.

### 5.1 Cyborg (mur du PlayerState, adaptateur pawn)

| Module (tag) | StatMod actuel | Levier REEL verifie | Etat | Branchement etage 2 |
|---|---|---|---|---|
| QMD_GeneralSystem (Phase.GeneralSystem) | Matter.Max Add [200,400] | ALS_Base_CharacterBP EventGraph `On Phase Update` -> `SS_Matter.SetMaxMatter` (SS_Matter : MaxMatter int RepNotify, /Game/Stats/Matter/SS_Matter) | **OK** (Base 100) | Remplacer le Select du pawn par `SetMaxMatter(QMOD_GetStat(self, Stat.Cyborg.Matter.Max, 100))`, rafraichi sur OnRackChanged |
| QMD_Jetpack (Phase.Item.Jetpack) | Jetpack.FuelMax Mult [0.25,0.5] | IS_JetPack.MaxFuel=100 (variable BP, /Game/Items/Jetpack/IS_JetPack) ; AUCUN scaling existant | **ECART** (effet jamais implemente) | Si assume : `Set MaxFuel(QMOD_GetStat(OwnerPawn, Stat.Cyborg.Jetpack.FuelMax, 100))` a l'equipement + OnRackChanged. Gates stationnaire/rapide : restent sur le niveau du module (facade) |
| QMD_ServomoteursDeJambes | Move.SprintSpeed Mult [0.08,0.16,0.25] | DT_MovementModelCyborg row Cyborg (Walk 175/Run 400/Sprint 700) applique par `UpdateDynamicMovementSettings` -> `Set MaxWalkSpeed` (x0.85). `MaxSpeedMultiplierALS` existe mais est ecrit sans etre consomme (levier mort) | **OK** | Multiplier dans UpdateDynamicMovementSettings (ou reactiver MaxSpeedMultiplierALS). **IMPERATIF ClientAuthority (CLAUDE.md 5)** : appliquer identiquement serveur ET client proprietaire (data deterministe, rack replique : OK) |
| QMD_AmortisseursCinetiques | FallDamage.Reduction Add [30,60,100] | ALS_Base_CharacterBP `SV_OnLanded(velocite)` : si >3000, ApplyDamage(MapRangeClamped(v, 3000..5000 -> MinFallDamage=20..MaxFallDamage=80)) | **OK** | Multiplier le degat calcule par (1 - Reduction/100) dans SV_OnLanded (serveur only : propre) |
| QMD_NanoRegenerateur | Health.RegenPerSec Add [1,2,3] | SS_PhysicalState `TickAutoHeal` : TakeHeal(1) toutes les 1 s, apres RegenLifeSeconds=15 sans degat, SI Matter>0 | **OK** (la regen EXISTE) | TakeHeal(1 + QMOD) ou raccourcir le delai : rester sur le montant (+N PV/s), le delai 15 s est un global param |
| QMD_SurcoucheDeBouclier | Shield.Max Mult [0.15,0.30,0.5] | SS_Shield (MaxShield=100, CurrentShield RepNotify, absorption moitie-degats dans Lib_Life) MAIS **SS_Shield n'est PAS dans InitialStats du pawn : le joueur n'a JAMAIS de bouclier aujourd'hui** (UI W_LifeShieldBar prete) | **PARTIEL** | Pre-requis : accorder la stat (ajout SS_Shield aux InitialStats du StatsComponent pawn = edit d'existant jour J, ou grant par BehaviorGrant du module). Puis SetShield/MaxShield x(1+v) |
| QMD_BlindageSousCutane | Armor.Flat Add [10,20,30] | Pipeline degats : `Lib_Life.ApplyStatDamageToActor` (MakeNoise -> autorisations -> NoMatter x2 -> bouclier -> TakeDamage). **AUCUNE etape d'armure n'existe** ; `DamageReductionPercent` est renseigne sur 11 IDA (casques, torses) mais n'a AUCUN lecteur (donnee morte) | **PARTIEL** | Creer l'etape de reduction dans Lib_Life.ApplyStatDamageToActor (avant bouclier) : degat - Armor.Flat du pawn cible. Decision associee : cabler aussi (enfin) DamageReductionPercent des equipements dans la meme etape |
| QMD_CaissonHermetique | DeathSafe.Slots Add [1..5] (max 5) | `DropAllItemsDeath` (ALS_Base_PawnBP, REDEFINIE a l'identique dans ALS_Base_CharacterBP) : boucle inventaire + equipement, **exclut deja les items tagges `StaticModule`**, drop via InventoryComponent.SV_DropItem (ItemDroppedReplicator, timeout -1) | **PARTIEL** | Etendre le filtre : proteger N items supplementaires (priorite quete) selon QMOD_GetStat(DeathSafe.Slots). Attention : fonction dupliquee (PawnBP + CharacterBP + variantes IA) : modifier LA source unique si possible (PawnBP) et verifier l'ombre de CharacterBP |
| QMD_SacDigitiqueEtendu | Inventory.Size Add [4,8,12] | InventoryComponent : `InventoryMaxSize` (RepNotify) = PawnPreset InventorySlots (4) + somme `AddInventorySlot` des items equipes (sacs) + `InventoryBaseSize` (SetInventoryBaseSize : deja appele par QangaPlayerState par paliers de niveau 16/32/...) ; recalcul `UpdateInventorySize` | **OK** | Ajouter un terme module dans UpdateInventorySize (ou incrementer InventoryBaseSize sur OnRackChanged, moins propre car persiste). Overflow deja gere (TryDropOverflowItemsInventorySize) |
| QMD_CompacteurDeMatiere | Inventory.StackMult Mult [0.25,0.5,1.0] | Taille de pile = `MaxStackPerSlot` PAR IDA, lue dans `FindAllAvailableItemStackByItem` et `HasAvailableStackByItemDA` (InventoryComponent) | **PARTIEL** | Multiplier a CES DEUX sites de lecture (arrondi bas, min 1). Verifier les ecritures de stack a la depose/split, et le comportement a l'EXTRACTION du module (stacks sur-remplis) : regle de degrade a definir |
| QMD_Negociateur | Trade.SellPriceMult Mult [0.03,0.06,0.09] | `InventoryComponent.SV_SellItem` : prix = `GameShopPrice` (IDA) x `DynamicRate` (= GetShopSellPriceMultiplier du shop : BP_Shop 0.5, NPC_PawShop_Seller 0.9, NPC_Mineral_Shop 1.0, BP_Shop_MatterTank 0.75 ; ou taux marche par stock via MarketDynamicRatesComponent) -> AddCoins(Ceil64) | **OK** | Dans SV_SellItem : DynamicRate x (1 + QMOD_GetStat(OwnerPawn, Trade.SellPriceMult, 0)). L'UI PUW_Shop devra afficher le meme calcul |
| QMD_BlindageDeDrone | Drone.HealthMult Mult [0.25,0.5,0.75] | IS_DroneBase : modele a IMPACTS (Switch phase : 1er hit fatal / 1 hit encaisse / 2 hits), fenetre 10 s | **ECART semantique** (pas de PV) | Redefinir en "+N impacts encaisses" (Add entier) OU creer de vrais PV drone. Voir 4.2 |
| QMD_NanoReparateurDeDrone | Drone.RepairTimeMult Mult [-0.2,-0.4,-0.6] | IS_DroneBase : auto-reparation `Select [30,20,10]` s (indexe par phase) -> Delay -> Destroyed=false | **OK** | Temps final = Select-base x (1 + v) via QMOD_GetStat sur le pawn proprietaire (v negatif reduit). Base a figer quand le module remplace la phase (30 s neuf) |
| QMD_RegulateurAtmospherique | Flight.FuelUseMult Mult [-0.1,-0.2,-0.3] | DEUX candidats : (a) jetpack : IS_JetPack CheckFuelTick (conso 1/0.1s, stationnaire /4) ; (b) vehicules : VehicleBase `FuelConsumeDelaySeconds`=30 + `ServerConsumeFuel` (graphe replie ConsumeFuelTimer), conducteur accessible via `GetDriverPawn()` | **ECART a trancher** (famille PIL + texte catalogue = vol vehicule ; le tag dit Cyborg.Flight) | Recommande : vehicule (fidele au texte "vol atmospherique"), en rallongeant FuelConsumeDelaySeconds x 1/(1+v) lu sur le mur du CONDUCTEUR a la prise du siege + OnRackChanged. Renommer mentalement le tag "conso de vol du pilote" |
| QMD_TrousseInterne (active) | (coquille, active) | Pattern soin verifie : IS_RepairBase ApplyHealToActor + cout Matter + cooldown | OK (canal actif M7) | AbilityGrant : soin actif auto-suffisant, reutilise ApplyHealToActor |

### 5.2 Armes (racks PAR EXEMPLAIRE, etabli)

| Module (tag) | StatMod actuel | Levier REEL verifie | Etat | Branchement etage 2 |
|---|---|---|---|---|
| QMD_AT56 (Phase.Item.At56) | Damage Add [30,50,70] | IS_AT56 Switch : Damage 30 -> 40/45/50 ; FireDelay 0.17 -> 0.16/0.15/0.14 | **ECART BLOQUANT etage 2** | Rechiffrer : Damage Add [10,15,20] + FireRate Add [0.0588,0.1176,0.1765] (4.1). Etage 1 : aucun impact (Switch conserve) |
| QMD_Nash (Phase.Item.NashRifle) | FireRate Mult [0.1,0.2,0.4] + Damage Mult [0.05,0.10,0.15] | NashV2 : delay x0.90/0.80/0.60, degats x1.05/1.10/1.15 (8 armes) ; V1 : tables en dur divergentes | **ECART semantique** | FireRate : Op -> Add + adaptateur delai x(1-v) (4.1). Damage : OK tel quel. V1 : laisser le Switch (etage 1) ou aligner la V1 sur les % (decision design) |
| QMD_AmplificateurDeDegats | Damage Mult [0.05,0.10,0.15] | Variable BP `Damage` de l'exemplaire, consommee par PreImplementFire -> `ServerFireBullet(..., Damage)` (C++ QWeaponBulletSubsystem prend le degat en PARAMETRE) | **OK** | Lecture pull au tir : Damage_final = QModuleItemRack::GetStat(instance, Stat.Weapon.Damage, Damage_base) : une ligne dans PreImplementFire (ou recalcul sur changement de rack) |
| QMD_RecycleurDeDouilles | Weapon.MatterPerShot Add [1,2] (max 2) | Chaine de tir : PreImplementFire (WeaponScript) apres conso munition ; credit matiere : `AddOrRemoveMatterToActor(OwnerPawn, N)` (pattern verifie dans SV_RecyleItem et IS_RepairBase) | **OK** | Dans PreImplementFire serveur : si module present, AddOrRemoveMatterToActor(pawn, valeur). Garde-fou : uniquement munitions physiques (balles), cadence deja bornee par FireDelay |
| QMD_ChambreThermique (Voss) | Damage Mult [0.20,0.35,0.5] ; **Drawbacks VIDES** | Degats : OK (meme levier qu'Amplificateur). Portee : parametre `Range=50000` passe en dur a ServerFireBullet dans PreImplementFire (levier existant). **PERFORATION : n'existe pas** (une seule cible par hitscan) | **PARTIEL** | Completer les Drawbacks (portee -15 % : Range x0.85 au tir). Perforation = mecanique nouvelle dans QWeaponBulletSubsystem (multi-hit du trace) : chantier C++ QWeapon, hors activation |
| QMD_CanonRenforce (test) | Damage Add [10,20,30] | idem Amplificateur | OK (harnais) | Conserver comme module de test etabli |
| QMD_ChargeurRapide (test) | FireRate Mult [0.1,0.2,0.3] | Variable BP `FireDelay` lue par la macro `AutomaticLoopFireDelay` (WeaponScriptMacros, RetriggerableDelay) | OK (harnais) | Meme semantique 4.1 |

Precision d'architecture armes : le tir est initie par l'enfant (Combat_1stTrigger -> macro
AutomaticLoopFireDelay -> ... -> TriggerFire -> SV_TriggerFire -> PreImplementFire -> ServerFireBullet
+ SpawnBulletTracer). Les degats et la cadence sont donc 100 % cote BP WeaponScript/enfants : le C++
QWeapon ne fixe AUCUNE valeur (il execute). Les 26 armes referencant PhaseComponent bindent toutes
OnPhaseUpdate ; les armes SANS phase (FA62, AK47, MZ56, Shotgun, Sniper : IDA.Phase=None) dependent du
**broadcast par defaut CallPhaseUpdate(0)** pour initialiser leurs stats : contrat a preserver (8.2).

### 5.3 Vehicules (racks session-only v1 ; module de test)

| Module (tag) | StatMod actuel | Levier REEL verifie | Etat | Branchement |
|---|---|---|---|---|
| QMD_NoyauSurcadence (test) | Vehicle.Speed.Max Mult [0.1,0.2,0.3] | `/Game/Systems/Vehicle/FlyVehicleMovementComponent` : `MaxSpeedKm/h` (3200) + `SetMaxSpeed` (convertit en MaxSpeedCm/s) ; present sur Bike/Spaceship/Hovercraft/Watercraft. Aussi : VehicleBase.SpeedMultiplier (nerf 0.01..1.0, replique) | **OK** | SetMaxSpeed(base x (1+v)) sur pose de rack/changement. Attention aux enfants qui overrident MaxSpeedKm/h par instance |
| (tag enregistre, aucun module) | Stat.Vehicle.Fuel.Max | VehicleComponent.Fuel = **byte 0..100 en POURCENTAGE** (ServerSetFuel clamp 0..100, init aleatoire 5..95, persistance cle "F") | **SANS LEVIER** | Un "reservoir agrandi" exigerait de re-modeler le fuel (litres). Module Reservoir etendu = REPORTER (deja le cas au triage). Leviers reels disponibles : conso (FuelConsumeDelaySeconds), prix recharge (CoinsPerFuelPercent=10) |

### 5.4 Rappel de flotte (actif signature, PIL)

Ancrage COMPLET verifie, aucun systeme a inventer :
- Possession : `PlayerVehiclesComponent` (ActorComponent sur le PlayerState) : `OwnedVehicles` (string array RepNotify), `VehiclesOnGarage`, `CurrentOwnedVehicle` ; persistance DataObject cles exactes `OwnedVehicles` / `VehiclesOnGarage` / `CurrentOwnedVehicle` ; achat SV_RequestBuy (prix : DA_AllRef map `VehicleClass:Price`, 51 vehicules ; echange carte : map `VehicleCardItem:Vehicle`, 29 entrees).
- Sortie : UN vehicule possede sorti par joueur : `WorldVehiclesManager.SERVER_SetSpawnPlayerOwnedVehicle` + acteur **`SpawnPlayerOwnedVehicle`** (persiste transform cle "T" + fuel cle "F", resout la classe via `Id:VehicleClass`, LoadClassAsync, SpawnActor au transform sauve).
- Le module = un AbilityGrant qui appelle ce pipeline avec un transform CIBLE proche du joueur au lieu du transform sauve, + trajet autonome : ships = pattern SpaceshipAI (trafic aerien vivant, descend du ciel) ; sol = conduite autonome a implementer (ou spawn hors champ + arrivee scriptee v1). Niveaux = portee (planete / espace local / univers) : verification de distance au cast, cooldown.
- Terminaux existants : BP_TerminalVehicleSpawner + PUW_SpawnVehicle (reference d'UX). TK_Garage / TK_GarageShip = simples DataAssets de marqueurs de carte (pas le systeme).

## 6. Table d'alignement : les 71 coquilles (classees par classe d'ancrage)

Rappel : une coquille sans StatMod n'a AUCUN effet meme active (agregation vide) : zero risque jour-1.
Le chiffrage se fait dans l'Editeur de Modules (qmodule.Editor.Open), en respectant les semantiques (4).

### 6.1 ANCRE VERIFIEE (levier existant, chiffrage possible des maintenant)

| Modules | Levier verifie (audit de ce jour) |
|---|---|
| SacocheDimensionnelle | = meme levier que SacDigitiqueEtendu (InventoryMaxSize). **DOUBLON fonctionnel** avec SacDigitiqueEtendu : fusionner ou differencier (decision design, 7.2) |
| RecycleurOptimise | `SV_RecyleItem` (typo gelee, InventoryComponent) : somme des `RecyclableMatterValue` via `CalcRecycleMatterOfItemAndAttachments` -> `AddOrRemoveMatterToActor`. Multiplier la somme. (Variante sol : ItemScriptBase.ServerTryRecycleItem, meme valeur plate) |
| Geologue | `RecyclerComponent` des noeuds MiningActorBase : `ServerGiveItem` partage `max(consomme/nbMineurs,1)` du `ItemResource` par tick 0.7 s : multiplier StackRes PAR MINEUR (compatible multi-joueurs). ATTENTION : suspicion de bug existant (conso Stack x nbPawns au carre, 9.5) : verifier avant chiffrage |
| ChasseurDePrimes | Cible recherchee : `QPoliceLibrary::GetWantedLevel(WorldContext, Player)` (C++, QPoliceLibrary.h:36) ; degats : meme levier que 5.2 au tir (bonus conditionnel dans PreImplementFire) |
| FleauDesMachines | Faction de la cible : `CombatComponent.Faction` (byte, lu par reflexion aussi par QWeapon pour le friendly-fire) : bonus conditionnel vs drones/tourelles au tir |
| Adrenaline | Event de kill : `CombatComponent.OnDeath` + recompenses au tueur (AddCoinsToInventory/AddExperienceByActor deja cables) : buff temporaire cote tueur = behavior + timer |
| InterceptionRadio | QRadio vivant (stations, reception par position) : nouvelle station police/Voss + regle de reception = contenu, pas de code core |
| OeilThermique | `VehicleComponent.OnVehicleStateUpdate(VehicleComponent)` (dispatcher verifie, fire par OnRep_VehicleState ; etat via E_VehicleState 0=Off 1=On 2=Broken 3=Destroyed) : marquer les vehicules etat=On au scan |
| TraqueurDeButin / Spectrometre / AnalyseurDeMenace / RadarPassif / EchoDeConstellation | IS_Scanner, fonction `AddTrackerByDistance` : les branches de detection existent DEJA (items monde via QUtilities_SpatialFilter_Group(Items), IA via QAI ScanAndCreateTempTrackers, ScannableComponent, **points QLevel via SpatialFilter_SubGroup(Levels/Scannable)** pour Echo) : chaque module = degater/enrichir une branche (gates par module au lieu de gates par phase) |
| AnalyseNecrologique | Cadavre/loot : `ItemDroppedReplicator` (grappe de drops) + DeathTrackerActor (TTL 1800 s) : reveler au scan = nouvelle branche sur ces acteurs |
| DecodeurQPD | `QPoliceSubsystem::StartDecayTimer(Player)` + FWantedStatus.DecayTimer (C++, QPoliceSubsystem.h:302/317) : la decroissance EXISTE ; +50 % = accelerer le delai au rearmement (petite modif C++ QPolice, lire le mur au moment du StartDecayTimer, event-driven) |
| BrouilleurDeSignature | `DetectionRadius` par archetype QAI (QAI_Struct.h, 8000..15000, consomme par QAI_StateProcessor) : reduction PAR CIBLE = modif C++ QAI respectant le SoA (cache par joueur rafraichi sur OnRackChanged, JAMAIS de poll ; whitelist QAI "ce qui ne doit pas changer" a respecter) |
| InterfaceDeContrats | Contrats VERIFIES structurels : `MatchManager` maps `PlayerId:MatchActor` (1 pour 1) + `GetPlayerContract` : **la limite 1 n'est pas une variable, c'est le modele de donnees** : +1/+2 contrats = refonte du mapping (design + code), PAS un tuning. Reclasser ce module (reporter ?) ou planifier la refonte |
| FortuneDeGuerre | Systeme de loot monde : Item_Random_Tools (IR_*.uasset, Item_Random_SpawnerManager) + champ Rarity des instances : le POINT DE TIRAGE de la rarete reste a localiser precisement avant chiffrage |
| ReputationICLab / AccreditationVoss | Shops : SellPriceMultiplier par shop + tags d'achat verifies ; "reputation" N'EXISTE PAS : v1 = remise/acces conditionnels au module (lecture du mur dans SV_BuyItem/SV_SellItem), la reputation-systeme reste a creer |
| TrousseInterne | (deja en 5.1) pattern IS_RepairBase |

### 6.2 CANAL ACTIFS (M7 AbilityBase + hotbar InputSystem ; mecanique par actif)

BaliseDeFrappe, TourelleSentinelle, DroneMedical, DroneDeCombat, BulleDeBouclier, ImpulsionEMP,
LargageDeRavitaillement, LeurreHolographique, MineDeProximite, Grappin, MurInstantane, KitDeSabotage,
FabricateurDeMunitions, BalisePersonnelle, MicroPropulseurs (dash), RappelDeFlotte (5.4).
- Canal verifie : plugin maison **InputSystem** (InputPreset_DA + InputsComponent sur le PlayerController + OnInputEvent) = binding des actifs ; UQModule_AbilityBase (M7) a implementer ; funnel SV_ActivateAbility prevu par l'architecture.
- Chaque actif porte SA mecanique (spawn tourelle = QAI/pattern drone police ; largage = GenerateNewItemInstance + drop ; fabricateur = AddOrRemoveMatterToActor -> munitions...). Aucun n'est bloquant pour l'activation : sans AbilityGrant cable, le module est inerte.
- Regle UX 14.6 (wanted QPolice sur actif offensif en ville, plafond 1 deployable, cooldowns longs) : a cabler dans AbilityBase des la V1 du canal.

### 6.3 SANS LEVIER AUJOURD'HUI (mecanique a creer ; modules inertes sans elle)

| Module | Constat d'audit |
|---|---|
| PasFeutres | **Aucun systeme d'ouie IA** : 0 consommateur de PawnNoiseEmitter/MakeNoise dans QAI et QPolice (le composant PawnNoiseEmitter du pawn est un VESTIGE ; seul usage : MakeNoise dans Lib_Life a la prise de degats, que personne n'ecoute). L'ancrage "PawnNoiseEmitter" du catalogue est invalide |
| PeauCameleon / FauxTranspondeur / DroneSpectre / ContreScan / MouchardReflexe / LeurreDrone | Furtivite/contre-detection : demandent des hooks dans la perception QAI (comme BrouilleurDeSignature) et/ou dans IS_Scanner (etre scanne, absorber un tir drone, marquer le tireur) : extensions coherentes mais a ecrire (drone : modele a impacts, 4.2) |
| ArmureReactive / SecondSouffle / IsolationFaraday / Rage / CoqueIntegrale | Renvoi de degats, survie a 1 PV, resistance EMP (les munitions EMP n'existent pas), buff conditionnel bas-PV, malus de regen : tous = etapes nouvelles dans Lib_Life.ApplyStatDamageToActor / TickAutoHeal. Le pipeline est centralise (bonne nouvelle) : UNE extension de Lib_Life peut servir les 5 modules |
| Condensateur | **Aucune stat energie/stamina n'existe** (sprint illimite, pas de barre d'energie) : module a re-scoper (ex. energie du drone ? fuel jetpack ?) ou a reporter |
| CoussinGravitationnel | GravityScape est DESACTIVE (DefaultGame.ini:307-311) : ancrer sur le GravityScale du mouvement (NinjaCharacterMovement) en chute + drain : behavior nouveau raisonnable |
| RecuperateurCinetique / SprintDeFond / ChargeurNeuronal / CompensateurNeural / CiblageAssiste / PoigneRenforcee / ReflexesCables / Concentration-like | Recharge d'energie en mouvement (pas de stat energie), cout endurance (pas de stamina), rechargement global (ReloadTime par arme existe : levier partiel), recul (pas de systeme de recul expose cote WeaponScript, le shake est cosmetique), precision de hanche (l'imprecision vit dans CalcAimingAndTriggerFire : levier partiel a creuser), melee (systeme melee a auditer), vitesse de switch (equipement : levier a creuser dans InventoryComponent). Chiffrage differable, aucun bloquant |
| VisionNocturne / MarqueurTactique / BoiteNoire / VoixDeCommandement / TraducteurUniversel / FilonQuotidien / AimantDeCollecte / GyroscopeDeporte | Post-process a creer ; marquage de cibles a creer ; balise cadavre (DeathTrackerActor EXISTE : extension facile) ; buff de groupe (coop) ; dialogues DQS (design) ; retention quotidienne (horloge serveur a definir) ; aimant de ramassage (ItemDroppedReplicator + overlap : extension facile) ; camera drone (offsets dans le suivi drone existant : extension facile) |

## 7. Ecarts et corrections, classes par gravite

> **STATUT 2026-07-10 : LES 6 BLOQUANTS SONT TRAITES** (decision RzZz : le buff Jetpack est ASSUME).
> Appliques et valides (93 definitions all clean) : AT56 Damage Add[10,15,20] + FireRate Add[0.0588/0.1176/0.1765] ;
> Nash FireRate en Add ; cap global FireRate 0.8 via Settings.GlobalStatClampMaxByTagName (applique dans
> BuildStatAggregates) ; BaseMatterMax 100 ; QMD_BlindageDeDrone sur le nouveau tag natif Stat.Cyborg.Drone.ImpactsAdd
> [1,2,3]. En complement : librairie facade UQModule_LegacyFacade (QMOD_GetLegacyPhaseLevel, plan 8.2) COMPILEE et
> dormante ; PUW_QModuleV2_Module (copie obsolete, 0 referent) supprimee (dette 7.4-10 reglee).

### 7.1 BLOQUANT avant l'etage 2 (chiffres/semantique faux : corriger les QMD dans l'Editeur de Modules)
1. **QMD_AT56** : Damage Add [30,50,70] -> **[10,15,20]** ; AJOUTER FireRate Add [0.0588,0.1176,0.1765]. (Reel verifie : 30->40/45/50 degats, 0.17->0.16/0.15/0.14 delai.)
2. **QMD_Nash** : FireRate Op Multiply -> **Add** (semantique 4.1). Valeurs inchangees. Damage : OK.
3. **Semantique Stat.Weapon.FireRate** a figer (4.1) + ClampMax 0.8 global sur le tag.
4. **QMD_Jetpack FuelMax** : decision requise (effet jamais implemente en legacy) : assumer le buff a l'etage 2 OU vider le StatMod. En l'etat, cabler "tel quel" changerait le jeu au moment du branchement.
5. **CyborgAdapter BaseMatterMax** : 1000 -> **100** (ou alimentation de la base depuis la valeur courante de SS_Matter) avant tout branchement Matter.
6. **QMD_BlindageDeDrone** : redefinir (impacts, pas PV) avant chiffrage etage 2 (4.2).

### 7.2 A CORRIGER (avant le chiffrage du module concerne, pas jour-1)
1. QMD_ChambreThermique : Drawbacks vides (portee -15 % a saisir) ; perforation = chantier QWeapon C++ separe.
2. Doublon SacocheDimensionnelle / SacDigitiqueEtendu : fusionner ou differencier.
3. QMD_RegulateurAtmospherique : trancher jetpack vs vehicule (recommandation : vehicule, 5.1).
4. InterfaceDeContrats : reclasser (la limite de contrats est structurelle, refonte necessaire).
5. Condensateur : re-scoper (aucune stat energie).
6. PasFeutres (+ drawback bruit de JambesDeFelin si un jour repris) : necessite un systeme d'ouie IA complet : reporter ou accepter le chantier QAI.
7. Branche joueurs du scanner marquee "dont work" : a reparer AVANT de vendre "Detect Players" v2.
8. Rarity=0 sur les 93 defs : bandes Commun/Avance/Prototype/Relique a poser en passe d'equilibrage.

### 7.3 OK (leviers verifies, branchements directs)
Matter.Max (base 100) ; Nash Damage V2 ; Repair/Scanner/Drone etage 1 ; ServomoteursDeJambes
(avec imperatif ClientAuthority) ; AmortisseursCinetiques ; NanoRegenerateur ; SacDigitiqueEtendu ;
Negociateur ; NanoReparateurDeDrone ; RecycleurDeDouilles ; AmplificateurDeDegats ; CanonRenforce ;
ChargeurRapide ; NoyauSurcadence ; RecycleurOptimise ; Geologue (apres verif bug 9.5) ;
ChasseurDePrimes ; FleauDesMachines ; OeilThermique ; extensions scanner (branches existantes) ;
DecodeurQPD et BrouilleurDeSignature (petites modifs C++ event-driven) ; RappelDeFlotte (pipeline
garage complet) ; canal actifs (InputSystem + M7).

### 7.4 Dettes du legacy decouvertes en passant (a signaler, HORS perimetre modules)
1. `DamageReductionPercent` renseigne sur 11 IDA (casques/torses) : AUCUN lecteur : l'armure d'equipement est morte.
2. P_Jetpack niveau 2 et P_Drone niveau 2 (flashbang) : effets decrits jamais implementes (le joueur paie un niveau pour rien : jetpack P2 n'apporte RIEN aujourd'hui ; drone P2 apporte les 2 impacts et 10 s mais pas le flashbang).
3. `MaxSpeedMultiplierALS` : ecrit chaque frame, consomme par personne (levier mort).
4. SS_Shield : stat complete + UI (W_LifeShieldBar) mais jamais accordee au joueur (InitialStats). Une entree None traine dans InitialStats du pawn.
5. IS_NASH_Base : CDO Damage=0.0 : sans le broadcast initial OnPhaseUpdate, degats nuls (fragilite du contrat, 8.2).
6. `IDA_Recycler.ItemScript` pointe IS_Multitool (pas IS_Recycler).
7. Scanner : branche joueurs commentee "dont work, conflict with group tracker".
8. Minage : suspicion conso Stack x nbPawns au carre dans ConsumeTick/ServerConsumeStack (9.5).
9. `Lib_Phase.RandomWeightedPhaseChance` : fonction ORPHELINE (0 appelant verifie) : reste utile comme graine des futures tables de loot.
10. PUW_QModuleV2_Module (copie M4 non recablee) appelle encore SV_UpPhase legacy : a recabler ou supprimer avant l'activation UI.
11. PawnNoiseEmitter sur le pawn : aucun consommateur nulle part.

## 8. Plan de facade PhaseComponent (checklist point 6a, detail d'implementation)

### 8.1 Contrat actuel verifie (ce que les 103 referencers consomment)
- `GetCurrentPhase() -> Current Phase : int` : si `PlayerPhaseData` valide -> `PlayerPhaseData.GetPhaseLevel(Phase = PhaseTarget)` sinon 0. **`PhaseTarget` est un OBJET DA_Phase** (pas un tag ; le tag est `PhaseTarget.PhaseTag`). Defaults : SCS des characters = P_GeneralSystem ; items = `IDA.Phase` pose au BeginPlay (boucle d'attente de l'IDA).
- `OnPhaseUpdate (dispatcher, param int)` : pousse par `CallPhaseUpdate()` : broadcast(GetPhaseLevel) si PlayerPhaseData ET PhaseTarget valides, **sinon broadcast(0)** : ce broadcast-0 initialise les stats des armes SANS phase (FA62, AK47, MZ56, Shotgun, Sniper) : NE PAS LE PERDRE.
- Resolution : items = OnOwnerPawnChanged -> PlayerState -> SetPlayerId ; pawn = OnControllerChanged -> SetPlayerId. Replication : PlayerPhaseData RepNotify (rebind cote client).
- Referencers : 26 armes + 5 items (Drone/Jetpack/Repair/Scanner/Construction) + 2 characters + WeaponScript + 68 maps (instances posees) + cook seed. Consommation pull (GetCurrentPhase) : IS_NASH_Base, IS_NASH_SHOT, IS_Scanner, IS_JetPack, IS_RepairBase, IS_DroneBase ; le reste = push OnPhaseUpdate.

### 8.2 Cible : redirection de la source de niveau (2 corps de fonctions edites, signatures INTACTES)
Nouvelle fonction C++ additive dans QModule (librairie facade), REFLEXIVE (pattern DQS/InventoryBridge) :

```
// UQModule_StatLibrary (ou lib dediee QModule_LegacyFacade) :
// Niveau v2 d'un PhaseComponent legacy. Retourne -1 si la resolution v2 n'est pas possible
// (plugin OFF, pas de rack, pas de tag) : l'appelant BP retombe alors sur le chemin legacy.
UFUNCTION(BlueprintPure, Category="QModule")
static int32 QMOD_GetLegacyPhaseLevel(const UActorComponent* LegacyPhaseComponent);
```
Resolution interne :
1. Lire `PhaseTarget` (FObjectProperty) sur le composant ; si valide, lire `PhaseTag` (FStructProperty FGameplayTag) sur le DA_Phase : c'est le ModuleTag v2 (tags conserves VERBATIM).
2. Owner = ItemScriptBase (arme/item) ? -> resoudre son instance (`Obj_ItemInstance` du porteur, deja fait par QModuleInventory/QModuleItemRack) -> `QModuleItemRack::GetSockets` -> niveau du socket dont ModuleTag == PhaseTag. (Domaines WEAPON : AT56, Nash ; les items cyborg Drone/Jetpack/Repair/Scanner restent sur le MUR, voir note ci-dessous.)
3. Sinon (character) -> Pawn->PlayerState -> UQModule_RackComponent -> `QMOD_GetModuleLevel(PhaseTag)`.
4. `-1` dans tous les cas d'echec (dormant, rack absent, tag invalide).

NOTE de routage : les 4 items cyborg (Drone, Jetpack, Repair, Scanner) sont des modules de BASE du
MUR en v2 (bBaseModule, domaine Cyborg) : leur PhaseComponent (sur l'ITEM) doit lire le MUR DU PAWN
PROPRIETAIRE, pas un rack d'instance. Regle de routage dans le helper : si le tag est un module de
domaine Cyborg -> mur du pawn proprietaire de l'item ; si domaine Weapon -> rack d'instance. Le
registre v2 (QMOD_GetRegistry) donne le domaine par tag.

Edits BP (jour J, sur /Game/Systems/Phase/PhaseComponent UNIQUEMENT, signatures inchangees) :
- **GetCurrentPhase** : Branch `QMOD_IsQModuleEnabled` -> true : `L = QMOD_GetLegacyPhaseLevel(self)` ; si L >= 0 return L ; sinon chemin legacy actuel. false : chemin legacy actuel.
- **CallPhaseUpdate** : meme redirection pour la valeur broadcastee ; la branche "invalide -> broadcast(0)" reste STRICTEMENT identique.
- **Re-notification** : nouveau bind (dans SetPlayerId et/ou BeginPlay) : si enabled, s'abonner a `OnRackChanged` du rack resolu (mur) -> `CallPhaseUpdate`. Pour les racks d'ITEM (etabli), ajouter un signal producteur C++ additif : `FQModule_OnItemRackChanged(ItemInstance)` (static multicast, world-scope par parametre, pattern qualite QMusic) broadcaste a la fin des mutations reussies de QModuleItemRack ; le PhaseComponent d'un item s'y abonne et filtre sur SON instance -> CallPhaseUpdate. Sans ce signal, une arme modifiee a l'etabli garderait ses stats jusqu'au re-equipement.

### 8.3 Ce que la facade NE fait PAS
- Elle ne supprime RIEN du legacy (GlobalPhaseManager/PlayerPhaseData/SS_Phase continuent de tourner cote donnees tant que la bascule n'est pas consommee).
- Elle ne change AUCUNE valeur de gameplay (etage 1).
- Elle ne touche pas aux 102 autres referencers : un seul asset edite (PhaseComponent), plus l'eventuel signal C++ additif.

### 8.4 Conversion des niveaux a la bascule (script d'activation, une fois)
- MUR : pour chaque joueur, lire PlayerPhaseData.PhaseLevelMap et inserer l'equivalent en phases dans les modules de base du mur (`QMOD_Authority_InsertPhase`, tiers T1 x niveau). GS niveau 2 = 2 phases T1 dans le noyau, etc.
- ARMES (type -> exemplaire, decision 8.5 du doc d'archi) : pour chaque instance possedee dont l'IDA.Phase est P_AT56/P_Nash : installer le module jumeau sur SON rack (`QModuleItemRack::InstallModule` version autorite) + inserer niveau-du-type phases T1. Les armes lootees ensuite arrivent vides (comportement v2 voulu).
- POINTS : solde SS_Phase (`GetPhasePoints`) converti en items IDA_QModulePhase_T1 (GenerateNewItemInstance + AddItemToInventory, pattern SV_AdminGetItem) puis SetPhasePoints(0) ; `LevelUp_Event` bascule de AddPhasePointsByActor vers le grant d'item T1 (checklist point 5).
- SV_UpPhase / PUW_PhaseDescription restent fonctionnels jusqu'a la bascule UI (le swap W_PhaseTree est deja en place cote QModule).

### 8.5 Complements checklist (delta vs QMODULE_ARCHITECTURE.md, a reporter dans la checklist)
- Point 4 (AllRef) : le nom EXACT de la map est **`ItemKey:DAItem`** (289 entrees ce jour) ; la map de decodage des tags est `Tag:PhaseGameplayTag` (7 entrees). Accessible en python par ces noms exacts.
- Point 6 : integrer la table 5.x (sites de branchement par stat) + les corrections 7.1 AVANT le cablage.
- NOUVEAU point : script de conversion 8.4 (mur + armes + points) a ecrire et tester sur une sauvegarde de dev.
- NOUVEAU point : signal `OnItemRackChanged` (8.2) si l'etabli doit rafraichir les armes a chaud.

## 9. Incertitudes residuelles (a lever a la demande, aucune bloquante pour ce rapport)

1. Le x0.85 de UpdateDynamicMovementSettings est lu comme litteral de pin ; si la pin est en realite connectee a MaxSpeedMultiplierALS, le "levier mort" n'en est pas un. A ouvrir dans l'editeur avant le cablage SprintSpeed.
2. Seuils exacts des gates scanner (branche vehicules) et jetpack (>0 presume) : non imprimes par l'outil.
3. Graphes replies non developpables par le pont : `ConsumeFuelTimer` (VehicleBase : formule exacte de conso, probablement 1 %/30 s moteur allume) et `DeathEvents` (pawn : ordre exact death screen/tracker/drop). Les entrees/sorties et variables sont verifiees, pas le cablage interne node par node.
4. Flags RPC exacts (Reliable etc.) des events SV_/MC_ legacy : convention projet supposee.
5. Minage : la suspicion "conso au carre" (ConsumeTick passe Stack=nbPawns a ServerConsumeStack qui re-multiplie) demande une verification dediee avant d'ancrer Geologue.
6. Les 7 NashV2 non-AR sont presumes porter les memes multiplicateurs que l'Assault_Rifle (bases CDO relevees, Switch non lus un par un). Idem variantes Noel.
7. Les instances posees en niveau (68 maps) peuvent overrider PhaseTarget/valeurs par instance : non verifiable en masse en lecture seule.
8. `search_project_index` indisponible dans cette session (index CodeScape jamais scanne) : les recherches d'appelants reposent sur ripgrep binaire des FNames (fiable pour l'existence, muet sur lecture vs ecriture).

---

*Audit realise editeur ouvert (session du 2026-07-10, boot 12:15), referme apres audit. Aucune*
*mutation : les seuls fichiers ecrits sont ce document et la mise a jour de la memoire de session.*
