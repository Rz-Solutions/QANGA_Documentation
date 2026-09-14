# Fiche Cyborg V3 : proposition de refonte de l onglet Statistiques

Etat : PROPOSITION du 2026-09-05, LIVREE et validee par Benja le meme jour (W_Stats_Overview reparente sur UQModule_CyborgSheetWidgetBase). Ce document garde la proposition d origine (lot 0).
Maquette interactive : https://claude.ai/code/artifact/b5a30dd1-1698-497c-af06-d9b16cc0f4a1
(scene 1920x1080, vue A = la fiche, vue B = l impact d un module depuis le mur, bouton de
simulation d un palier ; les chiffres du personnage d exemple sont illustratifs, les valeurs de
modules sont celles du catalogue).

Demande d origine (Benja) : le menu des statistiques du cyborg n est ni tres correct ni tres
complet ; il ne donne pas la liste des caracteristiques du personnage ; quand on pose des modules,
surtout des passifs, on s attend a voir dans un menu ce que ca change (niveau, capacites par
membre, etc.).

Toutes les mesures ci-dessous ont ete faites sur disque, editeur ferme, le 2026-09-05. Rien n a
ete execute en jeu. Les chemins sont cliquables ; les numeros de ligne sont ceux du jour.

---

## 1. Ce que le joueur a aujourd hui (mesure)

### 1.1 L onglet Statistiques
- Widgets reels : `Content/Widget/Statistics/W_Statistics.uasset` (conteneur, 2026-08-20) et
  `Content/Widget/Statistics/W_Stats_Overview.uasset` (la fiche, 2026-08-20). Le dossier
  `Content/Stats/` ne contient que les stat scripts, pas les widgets.
- Contenu : PROGRESSION (hexagone de niveau, XP, POINTS DE PHASE, CREDITS), SYSTEMES VITAUX
  (INTEGRITE STRUCTURELLE, BOUCLIER, MATIERE, CAPACITE D INVENTAIRE), REGISTRES DE TERRAIN, bouton
  VOIR LES REGISTRES vers la liste legacy (qui marche).
- Aucune mention des modules : une seule chaine gameplay, « Aucun module bouclier installe ».
  Zero occurrence de `QMOD` dans les deux assets.
- Les POINTS DE PHASE lisent `SS_Phase.CurrentPhasePoints`, c est a dire le stock LEGACY qui
  monte encore a chaque niveau et ne se depense plus. Le portefeuille V2 du rack
  (`QMOD_GetPhaseWallet`) n y est pas lu.
- La fiche est un cliche pris a l `Update` de l onglet : pas de rafraichissement live.
- Colonne REGISTRES : jamais cablee d apres la memoire du 2026-07-31 (fonction `RefreshRecords`
  absente). Non reverifie aujourd hui (pont editeur necessaire).

### 1.2 Le mur des modules (onglet Modules)
- Le popup natif `UQModule_ModulePopupWidgetBase` est MORT : `ModulePopupClass` n est lu nulle
  part, `QMOD_SetupPopup` n a aucun appelant, commentaire explicite
  `Plugins/QModule/Source/QModule/Private/QModule_WallWidgetBase.cpp:2646`.
- Le detail d un module = carte de survol + FICHE MODULE (colonne droite, `RefreshModuleSheet`,
  `QModule_WallWidgetBase.cpp:1587-1846`) : icone, nom, type, ACTIVE / PASSIVE, NIVEAU n / max,
  raison d inactivite, une ligne par palier (description redigee sinon synthese des StatMods),
  boutons + PHASE / - PHASE, bouton d action des gadgets.
- La FICHE MODULE n affiche NI les `Drawbacks`, NI la rarete, NI le fabricant, NI un avant / apres.
  Aucune valeur de base du personnage n est lue par l UI des modules.
- BILAN CYBORG (`RefreshBilan`, `QModule_WallWidgetBase.cpp:3138-3241`, valide en jeu le
  2026-08-29) : somme des effets actifs par le meme code que le rack
  (`QModuleAggregation::BuildStatAggregates`), mais des DELTAS seulement (`+X`, `+Y %`, `= Z`),
  8 lignes maximum puis « + N more active effects », malus fondus dans le net, tri alphabetique,
  clamps tus. La doc ecrit noir sur blanc que les valeurs de base « restent hors perimetre :
  chantier separe si voulu » (`Documentation/QMODULE_ARCHITECTURE.md:2629-2634`).
- Libelles de stats : table `StatLabelForTag` (`QModule_WallWidgetBase.cpp:3057-3136`, 39 entrees,
  `private static`), sources anglaises en `NSLOCTEXT`. 36 des 75 tags n ont pas de libelle et
  tombent sur un repli culture-invariant (« Drone Heal Per Sec »).

### 1.3 En jeu (HUD)
- Module passif = une icone hexagonale, rien d autre : `Content/Widget/HUD/Composition/
  W_HudModulePassives.uasset` + `W_HudModuleIcon.uasset` ne contiennent aucun TextBlock.
- Gadget = roue (`QModule_GadgetHUD.cpp`) : nom en capitales + description du niveau courant.
- Derive mesuree : `Module.TourelleSentinelle` est dans la liste C++ des gadgets
  (`QModule_GadgetHUD.cpp:64`) mais absent du conteneur `GadgetModuleTags` du BP HUD : une fois
  active, la tourelle apparait dans la roue ET dans la rangee des passifs. Bug a part.

### 1.4 Les donnees
- 105 definitions `QMD_*` (`Content/Phases/QModuleV2/`) : 52 avec StatMods, 49 avec
  LevelDescriptions, 1 avec Drawbacks (`QMD_CoqueIntegrale`), 20 avec Rarity, 7 `bBaseModule`,
  1 `bIsWallCore` (`QMD_GeneralSystem`).
- 75 tags de stat enregistres (8 natifs dans `QModule_Tags.cpp`, 67 en
  `Config/Tags/QModuleTags.ini` et `DefaultGameplayTags.ini`). 16 ne sont dans aucun QMD.
- Lecteurs reels de `QMOD_GetStat*` (scan complet de `Content/` et de `Plugins/*/Content/`,
  61 719 uasset de plugins, termine sans erreur) : 8 Blueprints (`ALS_Base_CharacterBP`,
  `InventoryComponent`, `SS_PhysicalState`, `SS_Matter`, `IS_DroneBase`, `RecyclerComponent`,
  `DynamicFlightComponent`, `QBuilder_Client_BP`) + 5 fichiers C++
  (`QModule_CyborgAdapterComponent` (hermetique, pose nulle part), `QModule_LegacyFacade`,
  `QModule_RackComponent`, `QModule_StatLibrary`, `QBuilder_SubSystem`).
- Changement depuis la memoire du 2026-08-31 : le plugin **QCombat** (`Plugins/QCombat/`, cree le
  2026-08-30) a repris la vie, le bouclier, l armure et les multiplicateurs de degats, joueurs ET
  IA. `Content/Systems/Combat/CombatComponent.uasset` est reparente sur `UQCombatComponent` ;
  `Content/Stats/Life/Lib_Life.uasset` est videe (4,6 Ko) et reparentee sur
  `QCombatBlueprintLibrary` ; elle ne lit plus aucun tag. `Armor.Flat`, `Damage.VsMachineMult`,
  `Damage.VsOutlawMult`, `SecondWind.MinHealthFraction` et `Shield.Max` sont pousses par
  `QModule_RackComponent.cpp:3572-3652` (`Authority_PushCombatState`) dans
  `FQCombatDamageTuning` / `UQCombatComponent`.
  - Etat replique a tous : `FQCombatReplicatedState` (`MaxLife`, `CurrentLife`, `MaxShield`,
    `CurrentShield`, `LifeState`, `Faction`...), `OnRep_ReplicatedState`, dispatcher
    `CombatStateUpdate`. Lecture client : `GetCurrentLife(Current, Max)`, `GetMaximumShield()`,
    `GetCombatStateSnapshot()` (`Plugins/QCombat/Source/QCombat/Public/QCombatComponent.h:81-195`).
  - `DamageTuning` n est PAS replique : un widget client recalcule l armure et les multiplicateurs
    via `QMOD_GetStat(PlayerState, tag, base)`.
  - Bases C++ : `LifeWhenNoStat = 100` (`QCombatComponent.h:92`), `MaxShield = 0` (`:98`),
    `FlatArmor 0`, `DamageVsOutlawMultiplier 1.0`, `DamageVsMachineMultiplier 1.0`,
    `SecondWindMinHealthFraction 0` (`QCombatTypes.h:160-176`).
  - Bouclier : accorde en prod par `QMD_SurcoucheDeBouclier` (50 / 100 / 150) ; regeneration
    apres `ShieldRegenDelaySeconds` = 12 s de calme puis 10 % / s (`QModule_RackComponent.cpp:3678-3717`,
    cle absente de `DefaultGame.ini`). Deux chemins coexistent : QCombat (`Authority_PushCombatState`)
    et le stat legacy `SS_Shield` cree par `QModule_LegacyFacade.cpp:143`.
  - Les stat scripts `SS_PhysicalState` / `SS_Shield` ne font plus qu amorcer QCombat
    (`InitializePhysicalStateFromStat`, `InitializeShieldFromStat`) et persister. La fiche actuelle
    lit encore `SS_PhysicalState` : a mesurer en PIE si c est encore une valeur live.
- AUCUNE notion de partie du corps dans `UQModule_Definition` (`QModule_Definition.h`) : les axes
  sont Domaine, `SynergyTags` (11 `Module.Family.*`, mot joueur « TYPES »), `ExclusivityTag`,
  fabricant, rarete, et la position hexagonale (anneau = verrou, pas categorie).
- L item d inventaire d un module (`IDA_QModuleCy_*`, 42) porte sa propre `ItemDescription`
  libre, independante des `LevelDescriptions` du QMD ; 21 sur 42 sont en anglais.
- Localisation : `Config/Localization/Game_Gather.ini` n a aucune etape `GatherTextFromSource`.
  Aucun `NSLOCTEXT` du C++ QModule n est collecte ; les libelles de stats sortent en anglais dans
  les trois langues. Les `QMD_*` sont absents du manifest (textes culture-invariant).

### 1.5 Agregation (la formule que la fiche doit reproduire)
`Plugins/QModule/Source/QModule/Public/QModule_Aggregation.h:13-23`,
`QModule_Aggregation.cpp:9-17, 97-138, 200-224` :

    Final = bHasOverride ? Override : (Base + SommeAdd) * Produit(1 + Mult)
    puis ClampMax (le plus restrictif), puis cap global par tag (QModule_Settings)

- `Multiply` vaut « x (1 + valeur) » : deux modules a +15 % donnent +32,25 %, pas +30 %.
- La BASE est toujours injectee par le consommateur (`QMOD_GetStatFromRack(tag, Base)`), QModule
  n en stocke aucune.
- Les `Drawbacks` sont des StatMods a valeur negative, appliques inconditionnellement, jamais
  adoucis par l adjacence, fondus dans les memes agregats.
- Adjacence : mecanisme present, bonus 0.0 par defaut, aucun override .ini : neutre.
- Les agregats ne sont pas repliques : ils sont reconstruits sur le client a partir des `Sockets`
  repliques (`RebuildAggregates`), donc valides cote client. La decomposition par module est
  reconstructible sans nouveau C++ (`QMOD_GetSockets` + `QMOD_GetDefinition` + `StatMods`
  `BlueprintReadOnly`), mais aucune API ne l expose telle quelle.
- Piege pour une fiche : `Move.SprintSpeed` est consomme comme un FACTEUR (base 1.0) dans
  `ALS_Base_CharacterBP.UpdateDynamicMovementSettings` (`QMODULE_ARCHITECTURE.md:402`), alors que
  l adaptateur C++ (base 600 uu/s) est hermetique. Une fiche qui afficherait « sprint = 600 x ... »
  n afficherait pas la valeur du jeu.

---

## 2. Le principe

La fiche montre la VALEUR EN JEU de chaque caracteristique et sa decomposition : base du jeu +
contribution de chaque module installe (+ malus nommes), rangee par zone du corps du cyborg.

Quatre regles :
1. Jamais une stat sans lecteur. Une caracteristique n apparait que si un consommateur la lit
   vraiment (les 13 sites ci-dessus). Un module dont la stat n a pas de lecteur affiche « aucun
   effet mesurable », jamais un bonus fictif. C est la regle appliquee aux descriptions le
   2026-08-22.
2. La meme formule que le consommateur : `QModuleAggregation::BuildStatAggregates`, le code du
   rack et du BILAN. L affichage ne peut pas deriver de l applique.
3. Le jeu a toujours raison : quand le runtime expose la valeur finale (inventaire
   `GetInventorySize`, matiere `MaxMatter`, vie et bouclier QCombat), la fiche affiche cette valeur
   et n en deduit la base qu a l envers. Quand seul le facteur est observable (sprint), la fiche
   applique la formule documentee du consommateur et un test QATS compare les deux.
4. Evenementiel, pas de sondage : `OnRackChanged`, `OnStatUpdated`, etat QCombat,
   `InventoryUpdate`. Bind une fois, refresh seulement quand l onglet est visible.

---

## 3. Les 7 zones et la liste V1 des caracteristiques cablees

Le mapping stat -> zone est une decision de PRESENTATION (registre de la fiche). Les QMD ne
changent pas.

| Zone | Caracteristique | Tag | Base (source) | Formule du consommateur | Modules connus (catalogue) |
|---|---|---|---|---|---|
| TETE | Portee de scan | (niveau Scanner) | IS_Scanner 1/3/6/10 km, x2 vehicule | Select par phase | Scanner (base) |
| TETE | Visibilite faune | Stealth.DetectionMult | 1.0 | clamp(stat, 0.05, 1.0) -> QAI | Masque pheromonal x0.85/0.70/0.55 |
| TETE | Visibilite Voss | Stealth.VossDetectionMult | 1.0 | idem, canal Voss | Contre-mesures Voss |
| TETE | Brouillage police | Stealth.PoliceTrackingNoiseCm | 0 | max(0, stat) -> QPolice | Brouilleur 75/150/300 m |
| TETE | Decroissance recherche | Wanted.DecayMult | 1.0 (QPoliceTypes.h : DecayDelay 60 s, LevelDecayInterval 20 s) | clamp(stat, 1, 10) divise l intervalle -> QPolice | Decodeur QPD |
| TETE | Portee radio | Radio.RangeMult | 1.0 (ReceptionRangeMult, replique ; stations : rayon 10 000 km) | max(0.1, stat) -> QRadioComponent sur QModule_IntegratedRadio | Antenne longue portee |
| TETE | Rappel de flotte | FleetRecall.CooldownSec | 180 s | GetStat(tag, 180) | gadget Rappel de flotte |
| TORSE | Integrite max | (aucun tag) | QCombat LifeWhenNoStat = 100 | GetCurrentLife(Current, Max), replique a tous | aucun module ne l augmente |
| TORSE | Regeneration | Health.RegenPerSec | 1 PV/s | SS_PhysicalState.TickAutoHeal apres 15 s | Nano-regenerateur +1/+2/+3 |
| TORSE | Armure plate | Armor.Flat | 0 | FQCombatDamageTuning.FlatArmor (non replique, recalcul client) | Blindage sous-cutane 10/20/30 ; Coque integrale (avec malus de regeneration : seul QMD a Drawbacks) |
| TORSE | Bouclier | Shield.Max | 0 = OFFLINE | SetMaximumShieldAndCurrent(round(stat)) ; regen 12 s puis 10 % / s | Surcouche de bouclier 50/100/150 |
| TORSE | Second souffle | SecondWind.MinHealthFraction | 0 = inactif | FQCombatDamageTuning | Second souffle |
| TORSE | Objets conserves a la mort | DeathSafe.KeepFraction | 0 | ALS_Base_CharacterBP.DropAllItemsDeath | Caisson hermetique |
| BRAS | Degats melee | Melee.DamageMult | 1.0 (DamageValue du BP, non lu) | ALS_Base_CharacterBP : DamageValue x stat | Poigne renforcee +20/40/60 % |
| BRAS | Degats vs machines | Damage.VsMachineMult | 1.0 | FQCombatDamageTuning | Fleau des machines |
| BRAS | Degats vs hors-la-loi | Damage.VsOutlawMult | 1.0 | FQCombatDamageTuning | Chasseur de primes |
| BRAS | Drone : impacts, reparation | Drone.ImpactsAdd, Drone.RepairTimeMult | phase du drone | IS_DroneBase | Blindage de drone, Nano-reparateur |
| BRAS | Gadgets : recharges, comptes, rayons | Strike.*, Missile.*, GrenadeLauncher.*, Sentry.*, SupplyDrop.*, Drone.Heal* | reglages QModule_Settings | QModule_RackComponent | gadgets |
| BRAS | Escouade | Squad.* | 0 / 500 / 3 s / 300 s | QModule_LegacyFacade | Protocole de meneur, Reseau de recruteur |
| NOYAU | Matiere max | Matter.Max | 100 | ALS_Base_CharacterBP : GetStat(tag, 100) | Systeme general +200/400/600 |
| NOYAU | Delai avant drain | Matter.DrainDelayMult | 1.0 (MatterConsumeSeconds, non lu) | SS_Matter | Recuperateur cinetique |
| NOYAU | Rendement recyclage | Recycle.YieldMult | 1.0 | QMOD_GetRecycleYieldFactor | Recycleur optimise +25/50 % |
| NOYAU | Rendement minage | Mining.YieldMult | 1.0 | RecyclerComponent | Geologue (a verifier) |
| DOS | Carburant jetpack | Jetpack.FuelMax | MaxFuel du CDO IS_JetPack (non lu) ; 0 sans le module Jetpack | QModule_LegacyFacade.ApplyJetpackHardware ; lecture client MaxFuel + dispatcher FuelUpdate (patron W_JetpackBar) | Jetpack (base) |
| DOS | Consommation en vol | Flight.FuelUseMult | 1.0 | CDO x clamp(stat, 0.2, 1.0) | Regulateur atmospherique x0.9/0.8/0.7 |
| DOS | Poussee spatiale | Jetpack.SpaceThrustMult | 1.0 (HoverSpeed 850, FastFlightSpeed 1350 cm/s) | DynamicFlightComponent.Get_FlySpeed, actif si gravite locale < 0.05 | Propulseur orbital +50/125/200 % |
| DOS | Coussin gravitationnel | Fall.GravityMult | 1.0 | clamp(stat, 0.3, 1.0) -> UQGravityCharacterComponent | Coussin gravitationnel |
| JAMBES | Marche, course, sprint | Move.SprintSpeed (sprint seul) | DT_MovementModelCyborg ligne Cyborg : 150/400/700 en visee libre (150/375/650 en direction de velocite, 150/400/600 en visee) x 0.85 | MaxWalkSpeed = chaine x Select(GetStat(tag, 1.0), 1.0, allure > 2.5) | Servomoteurs de jambes +8/16/25 % |
| JAMBES | Hauteur de saut | Jump.HeightMult | 1.0 (JumpZ 730) | JumpZ = base x sqrt(max(0.25, stat)) | Verins de saut +15/30/50 % |
| JAMBES | Reduction degats de chute | FallDamage.Reduction | 0 % (chute : MinFallDamage a MaxFallDamage, derniere mesure 5 a 30 pts, entre 6000 et 8000 uu/s : litteraux lus dans SV_OnLanded) | SV_OnLanded : degats x (1 - stat/100), clamp 0..1 | Amortisseurs cinetiques 30/60/100 % |
| SOUTE | Emplacements d inventaire | Inventory.Size | InventorySlots du preset DA_Pawn (non lu) + equipement ; final = GetInventorySize() | UpdateInventorySize : + Round(stat) | Sac digitique etendu +4/8/12 ; Sacoche dimensionnelle +2/4/6 (doublon) |
| SOUTE | Taille des piles | Inventory.StackMult | 1.0 | MaxStackPerSlot x (1 + stat), arrondi bas | Compacteur de matiere +25/50/100 % |
| SOUTE | Prix de revente | Trade.SellPriceMult | 1.0 | SV_SellItem | Negociateur +3/6/9 % |
| SOUTE | Coffre personnel | (niveau) | 16 / 32 / 48 / 64 / 92 / 128 | UpdatePlayerStorageSizeByLevel | progression |
| SOUTE | Construction : cout, portee, demontage | Build.CostMult, Build.RangeMult, Build.RecycleMult | 1.0 (QBuilder_Resource_Price_Factor et Refund_Factor 1.0, Build_Trace_MaxLength non lu) | QBuilder_SubSystem, QBuilder_Client_BP | Matrice de construction |

Hors fiche, volontairement : `Stat.Weapon.*`, `Stat.Vehicle.*` (aucun lecteur), `Aim.SpreadMult`,
`Recoil.Mult`, `Reload.TimeMult`, `Swap.TimeMult`, `Loot.RareChanceAdd`, `Pickup.MagnetRadiusM`,
`Armor.ReflectFraction`, `Tracker.*`, `Camera.*` (enregistres, jamais lus). Ils entrent dans la
fiche le jour ou un consommateur les lit, par une ligne de registre.

---

## 4. Architecture proposee (plugin QModule, zero changement de donnees de module)

a. Registre de caracteristiques : `FQModule_SheetAttribute` { Id, Zone (enum 7 zones), Label et
   Unite (String Table), Format (nombre / facteur / fraction / secondes), StatTag, BaseSource
   (QCombat, stat script, composant du pawn, reglage, neutre), Decimales, bLowerIsBetter, texte
   OFFLINE }. Porte par un DataAsset `DA_QModule_CyborgSheet` reference en `TSoftObjectPtr` dans
   `UQModule_Settings` (patron `UIMenuContainerClass`).
b. Deux fonctions de lecture dans `UQModule_StatLibrary` : `QMOD_GetStatBreakdown(Target, StatTag)`
   -> `TArray<FQModule_StatContribution>` { ModuleTag, DisplayName, Level, MaxLevel, Op, Value,
   bDrawback } reconstruite depuis `QMOD_GetSockets` et les definitions ;
   `QMOD_PreviewStat(Target, StatTag, Base, ModuleTag, NewLevel)` rejoue l agregation avec un
   niveau simule. Les deux appellent `QModuleAggregation::BuildStatAggregates` (deja
   `QMODULE_API`). `StatLabelForTag` (privee, statique dans le mur) devient publique.
c. La fiche : `UQModule_CyborgSheetWidgetBase` herite de `UQModule_HudRackWidgetBase`
   (resolution du rack, late-bind au join, garde pawn : obligatoire pour toute brique qui lit le
   rack, cf. `QMODULE_ARCHITECTURE.md:2748-2769`) et construit ses panneaux en natif comme le mur.
   `W_CyborgSheet` = WidgetBlueprint reparente (un UUserWidget C++ brut ne se pose pas comme enfant
   UMG). Insere comme nouvel index de `Switcher_Views` dans `W_Statistics`, mis par defaut ;
   `W_Stats_Overview` reste sur disque comme repli. Briques reutilisees : `W_HudVitalRow`
   (2026-09-01 : icone, barre, courant / total, crans) pour les jauges de l en-tete,
   `W_UnitDisplay` pour les registres, `StatLabelForTag` pour les libelles de repli.
   Relais de visibilite pose sur l ONGLET avec les cinq garde-fous de `UQModule_LegacyPhaseSwap`
   (idempotence, amorcage de l etat, filtre de transition, garde de premiere ouverture, differe
   d un tick avant toute mutation de slot).
d. Conserve tel quel : l event d interface `Update` de `W_Statistics`, le bouton VOIR LES
   REGISTRES et la liste legacy, le retour « < FICHE CYBORG », l index 4 du switcher du hub, le
   cadre `StarMap_MenuContainer`, la barre de section ambre, l hexagone de niveau, la courbe L^3.
   Les POINTS DE PHASE passent du stock legacy au portefeuille V2.
e. Le mur en retour : la FICHE MODULE recoit un bloc IMPACT SUR TA FICHE (pour chaque StatMod et
   Drawback du module selectionne : libelle, valeur actuelle -> valeur au palier suivant via
   `QMOD_PreviewStat`, malus en rouge) et un bouton « Voir dans la fiche » (`BPI_SetActiveTab`).
   Le BILAN CYBORG reste tel quel.
f. Localisation : String Table `ST_QModule_Sheet` (en / fr / es), seule voie qui fonctionne.
   Decision projet a part : ajouter ou non une etape `GatherTextFromSource`.

Contrats a ne pas casser : noms de widgets cherches par chaine (`GetWidgetFromName`), event
d interface `Update`, `Switcher_Views` index 0 = registres legacy, hub singleton jamais
reconstruit (rebind sur `OnVisibilityChanged`), jamais de Live Coding sur QModule (cold rebuild),
aucun poll (CLAUDE.md section 5), `ZOrder` negatif pour du HUD.

---

## 5. Lots

| Lot | Contenu | Effort |
|---|---|---|
| 0 | Ce document : valider les 7 zones, la liste V1, trancher les decisions | 0 code |
| 1 | C++ QModule : registre + DataAsset + reglage ; `QMOD_GetStatBreakdown`, `QMOD_PreviewStat`, libelles publics ; tests QATS (sprint, matiere, inventaire, regeneration, armure) compares au consommateur reel | 1 a 2 jours, cold rebuild |
| 2 | La fiche : base C++ + `W_CyborgSheet`, 7 zones, panneau de decomposition, schema cliquable, refresh evenementiel, String Table, portefeuille V2 dans l en-tete ; validation en jeu par diff affiche vs verite sur un perso REEL | 2 a 3 jours |
| 3 | Le mur : bloc IMPACT SUR TA FICHE, malus nommes, bouton « Voir dans la fiche » | 1 jour |
| 4 | Contenu (separe) : 56 QMD sans description, descriptions d items alignees sur les QMD, Tourelle sentinelle dans `GadgetModuleTags`, etape de gather des sources C++ si decidee | chantier de contenu |

---

## 6. Decisions a prendre

1. Les 7 zones (TETE, TORSE, BRAS, NOYAU, DOS, JAMBES, SOUTE) ou un classement par TYPE de module
   (les 11 du mur). Recommandation : zones pour la fiche, types pour le mur.
2. Montrer les valeurs neutres (« x1,00, aucun module ») avec les modules candidats.
   Recommandation : oui, en gris.
3. Schema : silhouette 2D dediee (texture a forger, une image teintable par zone) ou perso 3D
   existant (`W_ActorViewDisplay`). Recommandation initiale : silhouette. TRANCHE le 2026-09-05 par
   Benja apres la premiere capture ("pas belle, generee par IA, ne reflete pas mon cyborg de deuxieme
   generation") : c est le perso 3D reel (`W_ActorViewDisplay`, cadrage de l onglet Equipement, de
   face, sans deformation) avec des reperes hexagonaux par zone. Detail du contrat mesure : par. 15.35
   de `QMODULE_ARCHITECTURE.md`.
4. Decomposition dans la fiche seulement, ou aussi dans la FICHE MODULE du mur.
   Recommandation : les deux, meme code.
5. Vie maximale : aucun tag ne l augmente ; creer `Stat.Cyborg.Health.Max` est un choix de gameplay
   hors perimetre UI.
6. Nom de l onglet : « Statistiques » designe des compteurs ; « Cyborg » ou « Fiche » dit mieux ce
   qu on y trouve. Libelle hub localise, un seul point a changer.

---

## 7. Trouve au passage, hors perimetre

- Le paper-doll de l onglet Equipement (`W_Equipment`, 15 cases) affiche des cases `Head` et
  `2ndWeapon` qu aucun preset joueur n accorde (`EquipmentSlots` de `PP_PlayerCyborg_Survival`,
  `PP_PlayerCyborg_Tutorial`, `PP_DM_PlayerCyborg`). Les emplacements sont des `FName` (23 cles,
  `Lib_Inventory.GetEquipmentSlotName`), il n existe aucun enum.
- `SS_Coins` n a aucun `OnRep_*` ni dispatcher : credits potentiellement faux cote client.
- `Module.TourelleSentinelle` est dans la liste C++ des gadgets mais pas dans `GadgetModuleTags`
  du BP HUD : double affichage roue + rangee des passifs.
- Aucune stat de survie (oxygene, faim, soif, temperature, radiation, endurance, poids limitant)
  n existe dans le projet : `GetInventoryWeight` existe sans plafond, `W_EnvironmentInfo` affiche
  le climat WorldScape, pas une stat de personnage. La fiche n en inventera pas.
- `PUW_Statistics.uasset` (2025-02) est un popup legacy orphelin : aucun referent trouve.

---

## 8. Non verifie aujourd hui

- Le comportement en jeu (editeur ferme) : rien n a ete lance.
- Les formules internes des 8 Blueprints lecteurs (citees d apres `QMODULE_ARCHITECTURE.md:402-415`).
- L etat reel de la colonne REGISTRES et du bug des gardes `Not Equal (double)` de
  `W_Stats_Overview` (memoire du 2026-07-31).
- Toutes les valeurs par defaut de CDO Blueprint : `MinFallDamage`, `MaxFallDamage`, `DamageValue`,
  `MaxMatter`, `MatterConsumeSeconds`, `RegenLifeSeconds`, `MaxFuel` / `FuelConsume`,
  `InventorySlots` par preset, `Build_Trace_MaxLength`, `RecycleStack` : a lire avec le pont.
- `SS_PhysicalState.CurrentPhysicalState` suit-il encore la vie live depuis QCombat, ou n est-il
  plus qu un instantane de sauvegarde ? Le choix de la source de la fiche en depend.
- Lequel des deux stocks de bouclier (`SS_Shield` legacy, `QCombatComponent.MaxShield`) le HUD
  montre-t-il ? `W_LifeShieldBar` reference les deux.
- La cause de l absence des `QMD_*` dans le manifest de localisation.
