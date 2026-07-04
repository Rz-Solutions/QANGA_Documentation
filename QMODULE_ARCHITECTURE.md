# QMODULE : Architecture cible du système de Modules v2

> **Statut : DESIGN EN VALIDATION. RIEN N'EST IMPLÉMENTÉ.**
> Rédigé le 2026-07-04 (session de conception RzZz + Claude). Ce document décrit :
> 1. l'état RÉEL du système Phase actuel (audité dans le moteur ce jour),
> 2. le système CIBLE décidé avec RzZz,
> 3. le plan de migration sans régression.
> Toute divergence découverte pendant l'implémentation doit être corrigée ici dans le même mouvement.

---

## 0. TL;DR

Le système Phase actuel (modules du cyborg + modules par type d'item, montés avec des points) devient **la plateforme de récompense globale du jeu** :

- Le **Module** est une compétence installable (cyborg, arme, véhicule) : une définition data + un item échangeable.
- La **Phase** devient un **item consommable** qu'on insère dans un module pour l'activer et le monter en niveau.
- Le cyborg a un **Mur d'hexagones** (slots gérés par le niveau du Système Général) ; armes et véhicules ont des **racks par exemplaire**, édités à l'établi.
- L'injection de gameplay est 100 % data-driven via 3 canaux : **StatMods** agrégés par GameplayTags, **Actifs** (classes BP type stratagèmes), **Behaviors** (composants attachés).
- Le tout vit dans un nouveau plugin C++ : **QModule** (couche Q*, cf. CLAUDE.md §2).

---

## 1. Décisions actées (2026-07-04)

| Sujet | Décision |
|---|---|
| Progression | Les **Phases sont des items** (récompenses de level design, quêtes, etc.). Niveau du module = somme des tiers de phases insérées, plafonnée au MaxLevel du module (une Phase 2 = deux Phases 1). **Module sans phase = inactif.** |
| Armes / véhicules | Racks **par exemplaire**, stockés dans l'item lui-même. Édition à l'**établi d'armes** (qui gère aussi les pièces : canons, crosses...) et équivalent véhicule. |
| Échange | Modules **échangeables / vendables entre joueurs**, SAUF les modules de base. Marchands de modules prévus : PNJ de campagne, armurerie IC Lab, côté Voss. |
| Mort | Modules et phases **installés** jamais lootables sur un cadavre (« hermétiques à la mort » : ce sont les compétences du joueur). Ceux **en sac** suivent les règles d'inventaire normales à la mort (tranché 2026-07-04 : le sac est lootable). |
| Rack (stockage) | **Option A** (tranché 2026-07-04) : module installé = item instance attaché à un slot `Module_N` de l'instance porteuse, via le mécanisme attachments existant (§12.2). |
| Établi | **Acteur établi physique à créer** (tranché 2026-07-04) : installation ET retrait des modules d'armes/véhicules uniquement à l'établi. |
| Le Mur (build) | **Adjacence au cœur de la V1** (bonus de voisinage par famille + modules de Liaison + constellations), **placement libre** en coordonnées hex, loadouts en V2 (tranché 2026-07-04, détail §13). |
| Manufactures | Un même rôle existe en variantes : IC Lab (stock, stable), Voss (puissant mais instable, avec contreparties), artisanal, etc. L'instabilité = pure data (drawbacks), pas de code spécial. |
| Actifs | Modules actifs type stratagèmes Helldivers 2 : frappe aérienne, tourelle déployable, drone de soin, etc. |
| Périmètre V1 | **TOTAL** : cyborg + armes + véhicules dès la première version du plugin. |

---

## 2. Vocabulaire cible (attention, le sens des mots change)

| Terme | Aujourd'hui (code actuel) | Cible v2 |
|---|---|---|
| **Module** | le perk (asset `DA_Phase`, ex. P_Jetpack) | inchangé : la compétence installable |
| **Phase** | le NIVEAU d'un module (« phase 2 du jetpack ») | un ITEM consommable inséré dans un module (tier 1..N) |
| **Mur** | n'existe pas (arbre `W_PhaseTree`) | grille hexagonale de slots du cyborg |
| **Rack** | n'existe pas | slots de modules d'une arme / d'un véhicule (par exemplaire) |
| **Module de base** | les modules cyborg actuels | pré-installés, non retirables, non échangeables |
| **Système Général** | module cœur actuel | inchangé, MAIS son niveau pilote en plus la **capacité du Mur** |

⚠️ Collision UI : l'onglet « Modules » actuel (`/Game/Widget/Inventory/W_Modules`) désigne l'**équipement d'items** (slots d'équipement), pas les perks. À renommer ou distinguer pendant la passe UI.

---

## 3. État actuel vérifié (audit moteur du 2026-07-04)

### 3.1 Assets cœur (100 % Blueprint, zéro C++)
- `/Game/Systems/Phase/` : `DA_Phase`, `F_PhaseData`, `PhaseComponent`, `GlobalPhaseManager`, `PlayerPhaseData`, `Lib_Phase`.
- `/Game/Stats/PhasePoints/` : `SS_Phase` (points), `Lib_PhasePoints`.
- `/Game/Widget/Phase/` : `W_PhaseRouter`, `W_PhaseTree`, `W_PhaseTreeElem`, `W_PhaseLevel`, `W_PhasePoints`, `W_PhaseDescription`, `PUW_PhaseDescription`, `PUW_ItemPhase`, `EUW_Phase` + art hex réutilisable (`WF_HexFrame`, `T_FrameHex`, `T_FrameHexMask`, `T_PhaseTier0..3`).

### 3.2 Modèle de données
- `DA_Phase` (PrimaryDataAsset BP) = { `PhaseTag` (GameplayTag), `Name`, `Icon`, `MaxPhase`, `DescriptionPhase1..3` }.
- Instances sous `/Game/Phases/` ; les items/armes ont leurs propres définitions sous `/Game/Phases/ItemPhase/...` (vérifié : `IDA_AT56.Phase` → `/Game/Phases/ItemPhase/Weapons/P_AT56`).
- Registre actuel : `DA_AllRef.Phases` (`/Game/Systems/References/DA_AllRef`, classe `DA_References_C`), Map<GameplayTag, DA_Phase> de **7 entrées** vérifiées : P_GeneralSystem, P_Jetpack, P_Drone, P_Repair, P_Scanner, P_AT56, P_Nash. `W_PhaseTree` pose en dur 7 `W_PhaseTreeElem` sur son canvas (dont At56 et AllNash : l'arbre actuel mélange déjà cyborg et armes) et remplit une grille depuis cette map. Une map maintenue à la main ne scale pas : le registre AssetManager du §4 la remplace. Deuxième map dans le même DA : `Tag:PhaseGameplayTag` (Name → GameplayTag, utilisée au décodage de la persistance).
- **Effets et plafonds RÉELS des 7 modules (vérifiés dans les DA le 2026-07-04)** : General System Max 2 (+200 Matter par niveau) ; Jetpack Max 2 (P1 : vol stationnaire + vol rapide + 25 % fuel ; P2 : +50 % fuel) ; Drone Max 2 (réparation +25/50 %, résistance, flashbang 20 s) ; Repair Max 2 (+60/+80 réparation, cooldown 25/20 s) ; Scanner Max 3 (zone 3/6/10 km, détecte véhicules P1 puis joueurs P2) ; AT56 Max 3 (+30/+20/+20 dégâts + cadence) ; Nash Max 3 (TOUTE la famille Nash : cadence +10/20/40 %, dégâts +5/10/15 %). Donc : les plafonds actuels sont 2 ou 3 (pas uniformément 3), et il existe DÉJÀ un module de famille multi-armes (P_Nash) : précédent utile pour les modules « famille » vs « exemplaire ».
- **Économie actuelle vérifiée** : coût = 1 point de phase par niveau (le bouton d'achat de `PUW_PhaseDescription` n'apparaît que si points > 0 ET niveau < MaxPhase, puis appelle `SV_UpPhase(Phase)` sur `QangaPlayerState` ; le débit exact vit dans ce RPC). Les points (`SS_Phase`, int persistant clé « PhasePoints ») s'affichent via `W_PhasePoints`. Il existe aussi `Lib_Phase.RandomWeightedPhaseChance` : tirage 0..99 → niveau 0 (≤50), 1 (51..80), 2 (81..98), 3 (99) ; appelants à identifier (probable : niveaux d'arme aléatoires des IA/loot). C'est un embryon de table de rareté réutilisable pour le loot de modules/phases.

### 3.3 État joueur et réplication
- `PlayerPhaseData` (parent `ORReplicatedObject`, un par joueur, servi par `GlobalPhaseManager`) : `PhaseLevelMap` Map<GameplayTag, byte>, répliquée via un array de `F_PhaseData` {PhaseId, Level} + OnRep.
- Mutation UNIQUEMENT via `ServerSetPhase` (server-authoritative, clamp à MaxPhase).
- Persistance : objet persistant du GameDataManager, clé « PhaseData », encodage string « TagName♠Level ».
- Montée de niveau actuelle : points (`SS_Phase`) dépensés via RPC `SV_UpPhase` (vérifié en session antérieure).

### 3.4 Distribution sur le gameplay
- `PhaseComponent` (ActorComponent) posé sur : le cyborg (`ALS_Base_CharacterBP`), les IA (`AI_BaseCharacter`, `AI_Voss_*`) et TOUS les items/armes (**106 référents**). Sur un item : BeginPlay → `ItemScriptBase.GetItemDataAsset().Phase` → `PhaseTarget`, suit `OnOwnerPawnChanged`.
- Contrat consommé partout : `GetCurrentPhase` + dispatcher `OnPhaseUpdate`. **Interdiction de casser ce contrat (CLAUDE.md §4)** : la migration passe par une façade (§8).
- ⚠️ Conséquence : aujourd'hui le niveau d'une arme est **par TYPE et par JOUEUR** (le rifle AT56 lit le niveau du module P_AT56 de son porteur). La cible v2 (rack par exemplaire) est un changement de comportement : règle de conversion en §8.

### 3.5 Faits techniques utiles au design
- **Pas d'AbilitySystemComponent sur le joueur** (vérifié sur `ALS_Base_CharacterBP`) : GAS présent dans le projet mais non branché sur le pawn. Stats custom : `SS_PhysicalState`, `SS_Shield`, `SS_Matter`, `SS_Level`, `SS_Coins`, `SS_CharacterStatistics`, `SS_Transform`.
- **Précédent AssetManager** : DQS déclare son PrimaryAssetType (`QuestSystemAssets`) dans `DefaultGame.ini` → même pattern pour le registre QModule.
- **Art prêt et inutilisé** : `/Game/Items/ModulePhase/` contient `ModulePhaseBase` + `ModulePhase1..6` + matériaux, sans aucun référent → base parfaite pour les items Phase tiers 1..6.
- Cibles d'adaptateurs identifiées : `NinjaCharacterMovementComponent`, `DynamicFlightComponent` (jetpack), `StatsComponent`, `InventoryComponent`, `CombatComponent`, `ClientAuthorityComponent` (cyborg) ; `/Game/Systems/Vehicle/VehicleBase` (véhicules) ; `WeaponScript` (armes).
- Établi : la map `L_ATELIER_ARME` existe ; contenu/feature réels à auditer (M0).

---

## 4. Architecture cible

### 4.1 Principe fondateur
**Un module ne modifie JAMAIS le gameplay directement.** Il publie des effets sous forme de données ; chaque domaine (cyborg, arme, véhicule) possède un adaptateur qui les applique. Conséquence : ~90 % des modules = un DataAsset, **zéro code**, tant qu'ils n'utilisent que des stats déjà exposées.

### 4.2 Plugin `QModule`
Couche Q* : dépend de Cy_* et GameplayTags ; ne dépend JAMAIS du contenu `/Game` (les définitions vivent en contenu). Macro `QMODULE_API`, catégorie `LogQModule`, préfixe BP `QMOD_`, réglages via `UDeveloperSettings` + CVars (`qmodule.Enabled`, `qmodule.Debug`).

| Classe | Rôle |
|---|---|
| `UQModule_Settings` | DeveloperSettings (DefaultGame.ini) : chemins de scan, courbe de capacité du Mur, flags. |
| `UQModule_Definition` | UPrimaryDataAsset (PrimaryAssetType « QModuleAssets ») : identité `ModuleTag`, `Domain` (Cyborg/Weapon/Vehicle), `TargetFilter` (FGameplayTagQuery : quelles familles acceptent ce module), `Manufacturer`, `Rarity`, `MaxLevel`, `bBaseModule`, `ExclusivityTag`, effets par niveau (§4.4), `Drawbacks` (mêmes structs, en négatif), UI (Name/Descriptions localisées par niveau, Icon, style de cadre hex), lien vers l'IDA de sa forme item. |
| `UQModule_Registry_GI_Subsystem` | Scan AssetManager au boot (serveur ET client), Map<Tag, Definition>, requêtes par domaine/manufacture/rareté (loot, marchands), validation (tags dupliqués, StatTag inconnu, icône manquante). |
| `UQModule_RackComponent` | LE composant universel (mur cyborg, rack arme, rack véhicule). État répliqué + RPC serveur + cache d'agrégation. |
| `UQModule_StatLibrary` | Façade BP : `QMOD_GetStat(Target, StatTag, Base)`, `QMOD_GetModuleLevel(Target, ModuleTag)`, bind `OnRackChanged`... |
| `UQModule_AbilityBase` | UObject BP-able : `CanActivate` / `Activate` (serveur) / hooks cosmétiques client / cooldown. Un BP enfant par actif. |
| Adaptateurs de domaine | Cyborg : composant qui applique l'ApplyMap (écrit MaxWalkSpeed, fuel jetpack...). Arme : lectures pull dans `WeaponScript`. Véhicule : ApplyMap sur `VehicleBase`. Seuls les adaptateurs connaissent le gameplay. |

Structs principaux (rappel CLAUDE.md §6 : pas d'initialiseurs désignés dans les USTRUCT) :
- `FQModule_StatMod` { StatTag, Op (Add / Mult / Override / ClampMax), valeur par niveau }.
- `FQModule_AbilityGrant` { TSoftClassPtr<AbilityBase>, Cooldown, Charges, InputSlot }.
- `FQModule_BehaviorGrant` { TSoftClassPtr<UActorComponent> }.
- `FQModule_SocketState` { SlotIndex, ModuleTag, InsertedPhases (TArray<uint8> des tiers), Level (dérivé, clampé), bActive }.

### 4.3 Taxonomie de tags (le vocabulaire partagé)
- Modules : `Module.<Domaine>.<Categorie>.<Role>.<Manufacture>` (ex. `Module.Cyborg.Mobility.JetpackDrive.Voss`).
- Stats : `Stat.Cyborg.*`, `Stat.Weapon.*`, `Stat.Vehicle.*` : déclarées **en natif C++** (`QModule_Tags.h`) pour zéro typo ; c'est le contrat entre définitions et adaptateurs.
- `ExclusivityTag` : un seul module actif par groupe sur une même cible (pas deux drives de jetpack).
- Gouvernance : revue de nommage obligatoire avant chaque batch de contenu (le volume prévu est grand).

### 4.4 Injection : les 3 canaux
**A) StatMods (passifs chiffrés).** Le rack agrège par StatTag : `Final = (Base + ΣAdd) × Π(1 + Mult)`, puis Override éventuel, puis ClampMax ; ordre fixe et documenté. Cache recalculé UNIQUEMENT sur changement de rack (équip/retrait/insertion de phase) : zéro tick, lecture O(1).
- **PULL** (lecture à l'usage) : `WeaponScript` au tir : `Damage = QMOD_GetStat(Arme, Stat.Weapon.Damage, BaseIDA)`. Une ligne remplace les branchements codés en dur actuels.
- **PUSH** (écriture moteur) : l'adaptateur écoute `OnRackChanged` et applique son ApplyMap : `Stat.Cyborg.Move.SprintSpeed` → NinjaCharacterMovement ; `Stat.Cyborg.Jetpack.FuelMax` → DynamicFlightComponent ; `Stat.Vehicle.*` → VehicleBase.

**B) Actifs (stratagèmes).** Chaque actif = un BP enfant de `UQModule_AbilityBase`, déclenché via le funnel RPC `SV_ActivateAbility(Slot)` validé serveur (cooldowns, charges, coûts). Bind input via le plugin **InputSystem maison** (presets `InputPreset_DA`) + hotbar UI. Le core ne connaît AUCUN actif : on peut en créer des dizaines sans toucher au plugin.

**C) Behaviors (passifs comportementaux).** Un composant attaché tant que le module est actif (double saut, aimant de loot...). Contrat strict : add à l'activation, remove à la désactivation, rien d'autre.

### 4.5 Réseau (server-authoritative, serveur dédié, 500 joueurs)
- Toute mutation par **RPC serveur Reliable** : `SV_InstallModule`, `SV_RemoveModule`, `SV_InsertPhase`, `SV_ActivateAbility` (+ retrait de phase selon §5). Validations : possession de l'item, TargetFilter, slot débloqué, exclusivité, cap, règles base-module, cooldowns.
- Réplication d'état compact { Slot, ModuleTag, Phases[] } : Mur cyborg répliqué au owner (détail complet) via le canal ORReplicatedObject éprouvé (pattern PlayerPhaseData) ; racks armes/véhicules répliqués sur le composant de l'acteur (relevancy naturelle).
- **Événementiel partout** : OnRep → recalcul du cache → broadcast `OnRackChanged`. Jamais de polling (règle QRadio, CLAUDE.md §5).
- FX d'actifs : Multicast **Unreliable** (convention projet : Reliable pour l'état, Unreliable pour le cosmétique).
- Anti-triche : le client n'écrit jamais une stat qui compte ; le serveur recalcule avec les mêmes DataAssets. ⚠️ Stats de MOUVEMENT : `ClientAuthorityComponent` est sur le pawn ; l'ApplyMap mouvement doit s'appliquer à l'identique des deux côtés (déterminisme data) pour ne pas fausser la réconciliation.

### 4.6 Persistance
- **Mur cyborg** : même canal que `PlayerPhaseData` aujourd'hui (objet persistant GameDataManager), nouvelle clé versionnée (ex. « ModuleWall;v1 »), encodage compact.
- **Racks par exemplaire** (armes/véhicules) : **RÉSOLU par l'audit M0 (§12)**. Chaque item possède déjà un `Obj_ItemInstance` (ORReplicatedObject) persisté clé/valeur dans son DataObject (plugin `/DataManager`, DB) : Stack, Rarity, Owner, Customization et la map d'attachments `Slot:AttachmentId` (encodée « Slot♦Id », clé « SlotAttachments », rechargée par `LoadFromDataObject`). Les racks de modules utilisent EXACTEMENT ce mécanisme : soit de nouvelles clés dédiées, soit (recommandé) le module installé = un item instance attaché à un slot module de l'instance porteuse, comme une pièce. Zéro nouvelle infrastructure de persistance à inventer.
- **Mort** : les flux loot-on-death ne touchent NI le Mur NI les racks (hermétiques). Les modules/phases NON installés, en sac, suivent les règles d'inventaire normales à la mort (TRANCHÉ 2026-07-04 : le sac est lootable).

### 4.7 UI
- **Le Mur** : grille hex infinie **virtualisée** (on ne crée pas 500 widgets vivants), anneaux de slots débloqués par le niveau du Système Général. Réutilise `WF_HexFrame`, `T_FrameHex`, `T_PhaseTier*`. Remplace l'écran `W_PhaseTree`.
- **Détail module** : évolution de `PUW_PhaseDescription` : niveaux, phases insérées, manufacture, drawbacks, comparaison de variantes.
- **Établi d'armes/véhicules** : acteur d'interaction À CRÉER (sur le pattern d'interaction existant), qui ouvre l'écran rack de l'arme/du véhicule posé ; installation ET retrait des modules uniquement là (tranché 2026-07-04). Les PIÈCES (canon, crosse...) restent le système d'attachments actuel, éditable depuis l'inventaire, hors périmètre QModule v1.
- **Hotbar des actifs** : bind via InputSystem (presets), layout à trancher (§11).

---

## 5. Modèle de progression détaillé
- Item Phase : tiers 1..6 (l'art existe). Valeur d'insertion = tier.
- Niveau du module = min(MaxLevel, somme des tiers insérés). 0 phase = module inactif.
- **Règle anti-deadlock** : le Système Général doit avoir un niveau plancher de 1 (ou la courbe de capacité du Mur doit avoir une base > 0), sinon capacité 0 = plus rien d'activable. À fixer en config, pas en code.
- Retrait de phase (respec) : **OUVERT**. Recommandation v1 : retrait libre au Mur/à l'établi, phases rendues intactes (encourage l'expérimentation, durcissable ensuite). Alternative : retrait payant ou destructif (puits d'économie).
- Modules de base : pré-installés, non retirables, non échangeables ; montent avec des phases comme les autres.

---

## 6. Économie et acquisition
- Sources de **phases** : placement level design (récompense d'exploration), quêtes DQS, boss/événements.
- Sources de **modules** : loot ciblé, marchands (PNJ de campagne, armurerie IC Lab, côté Voss), craft éventuel plus tard.
- Échange joueur à joueur : oui, sauf `bBaseModule` (vérifié CÔTÉ SERVEUR à tout transfert).
- Manufactures = multiplicateur de contenu : chaque rôle × { IC Lab, Voss, artisanal, ... } sans une ligne de code de plus.

---

## 7. Première liste de modules (proposition de cadrage, non exhaustive)

> **Le catalogue complet (136 entrées + builds + règles transverses) vit dans `Documentation/QMODULE_CATALOGUE.md`.** La liste ci-dessous n'est que l'aperçu initial conservé pour l'historique.

Chaque entrée peut exister en variantes de manufacture (IC Lab stable / Voss fort mais instable / artisanal aléatoire). (A) = actif.

**Cyborg, mobilité** : Servomoteurs (vitesse sprint) ; Amortisseurs cinétiques (dégâts de chute) ; Vérins de saut ; Semelles magnétiques (adhérence) ; Exosquelette porteur (capacité de port, si stat de poids exposée).
**Cyborg, survie** : Blindage sous-cutané (réduction de dégâts) ; Nano-régénérateur (regen hors combat) ; Régulateur thermique (climats extrêmes, s'appuie sur l'API climat WorldScape) ; Surcouche de bouclier (SS_Shield max) ; Condensateur (énergie/stamina, à confirmer).
**Cyborg, économie & prospection** : Compacteur de matière (cap Matter, reprend l'effet actuel du Système Général) ; Aimant de collecte ; Spectromètre (ressources riches surlignées au scan) ; Négociateur (prix PNJ) ; Décodeur QPD (gestion du wanted, s'appuie sur QPolice).
**Cyborg, info & furtivité** : Brouilleur de signature (rayon de détection des IA réduit, s'appuie sur QAI) ; Radar passif ; Marqueur tactique.
**Cyborg, actifs (A)** : Balise de frappe aérienne ; Tourelle déployable ; Drone médical (réutilise la base drone existante) ; Bulle de bouclier ; Impulsion EMP (drones/véhicules) ; Largage de ravitaillement ; Leurre holographique ; Stimulant de combat (buff avec contrecoup) ; Ping longue portée.
**Armes** : Amplificateur de dégâts ; Accélérateur de culasse (cadence) ; Chargeur étendu ; Auto-chargeur (rechargement) ; Compensateur (recul) ; Canon allongé (portée/précision) ; Munitions perforantes ; Munitions EMP ; Réducteur de signature (aggro/wanted au tir) ; Module vampirique Voss (soin au kill, instable : drain passif).
**Véhicules (sol et vol)** : Turbocompresseur (vitesse max) ; Injecteur de boost ; Réservoir étendu ; Blindage châssis ; Suspensions renforcées ; Radar embarqué ; Soute agrandie ; Stabilisateur de vol ; Régulateur de croisière ; Camouflage thermique.

---

## 8. Migration depuis le système actuel (anti-régression, CLAUDE.md §4)
1. **Contrats préservés** : `PhaseComponent.GetCurrentPhase` + `OnPhaseUpdate` + `PhaseTag` deviennent une FAÇADE lisant le nouveau rack. Les 106 référents (armes, items, IA, cyborg) ne changent pas d'un octet au jour 1.
2. **Tags conservés** : les `PhaseTag` existants restent les `ModuleTag` des définitions migrées.
3. **Conversion des définitions** : `DA_Phase` (cyborg + `/Game/Phases/ItemPhase/...`) → `UQModule_Definition`, via outil éditeur + validation EUW.
4. **Points → phases-items** : conversion du solde `SS_Phase` et des niveaux acquis en équivalent phases à la première connexion (grandfathering, blob versionné). `SV_UpPhase` déprécié après bascule ; l'arbre actuel reste fonctionnel jusqu'à la bascule du Mur.
5. **Armes, type → exemplaire** : aujourd'hui le niveau est par type et par joueur ; demain par exemplaire. Règle de conversion à trancher (proposition : à la bascule, les armes possédées héritent du niveau du type de leur propriétaire).

---

## 9. Plan d'implémentation V1 (périmètre total, par jalons)
- **M0. Audits préalables : TERMINÉ le 2026-07-04, résultats en §12.** 4 audits sur 5 concluants (persistance d'instance OK, marchands OK, liste de modules OK, pièces d'armes OK) ; reste ouvert : règles de mort/drop d'inventaire (non bloquant pour M1..M4).
- **M1.** Squelette plugin QModule : Settings, tags natifs, Registry + scan AssetManager, log, EUW de validation.
- **M2.** RackComponent + réplication + persistance Mur + adaptateur cyborg sur 3 stats pilotes (sprint, fuel jetpack, cap Matter).
- **M3.** Façade PhaseComponent + conversion des définitions existantes (zéro régression visible joueur).
- **M4.** Items Phase (meshes existants) + insertion + UI Mur v1.
- **M5.** Armes : racks par exemplaire + établi + lectures pull dans WeaponScript.
- **M6.** Véhicules : rack + adaptateur VehicleBase.
- **M7.** Actifs : AbilityBase + hotbar + 2 ou 3 vitrines (drone de soin, balise de frappe, tourelle).
- **M8.** Marchands, placement loot, balance (caps par StatTag), polish UI.

Chaque jalon est validé sur les 3 rôles réseau (serveur dédié, serveur d'écoute, client) avant de passer au suivant.

---

## 10. Risques identifiés
- Persistance par exemplaire : PROUVÉE pour l'état d'instance (M0, §12). Risque résiduel : durée de vie des items droppés AU SOL entre redémarrages serveur (à vérifier en test M5), et volume de DataObjects si chaque module/phase devient une instance (surveiller la DB).
- Deadlock Système Général à 0 (§5) : règle plancher obligatoire.
- Collision de nommage UI « Modules » (équipement) vs Mur.
- Balance à l'échelle : caps (`ClampMax`) par StatTag dès le jour 1 + EUW de validation.
- Stats de mouvement vs `ClientAuthority` : déterminisme des deux côtés.
- Migration armes type → exemplaire : communication joueurs nécessaire (Early Access).
- Volume de contenu : gouvernance des tags et revue de nommage par batch.
- Ne JAMAIS supprimer `PhaseComponent`/`DA_Phase` avant la fin de migration : 106 référents, couplage par chaîne possible ailleurs.

---

## 11. Questions ouvertes (à trancher avant M4)
1. Politique de respec (retrait de phases) : libre / payant / destructif ?
2. Plage des tiers de phase au lancement : 1..6 (art complet) ou réduite ?
3. Un module installé est-il retirable et retourne-t-il en inventaire ? (Recommandé : oui, sauf modules de base.)
4. Les pièces d'armes (canon, crosse) restent-elles un système séparé du rack ? (Recommandé : oui, hors QModule v1.)
5. Layout d'input des actifs : hotbar chiffrée, roue, ou combinaison ?
6. ~~Sac à la mort~~ TRANCHÉ 2026-07-04 : le sac est lootable (règles d'inventaire normales) ; seuls les modules/phases INSTALLÉS sont hermétiques.
7. ~~Établi~~ TRANCHÉ 2026-07-04 : établi physique à créer ; installation et retrait des modules d'armes/véhicules uniquement à l'établi.

---

## 12. Résultats des audits M0 (2026-07-04, vérifiés dans le moteur)

### 12.1 Persistance par instance d'item : EXISTE, en production
- Chaque item a un **`Obj_ItemInstance`** (`/Game/Systems/Item/Obj_ItemInstance`, parent `ORReplicatedObject`) : `ItemInstanceId`, `ItemDataAsset`, `Stack`, **`Rarity`** (déjà là : les variantes de manufacture ont un logement naturel), `Owner`, `CustomizationInstanceId`, `AttachedToSlot/Id`, map `Slot:AttachmentId`.
- **Écriture traversante** : chaque setter écrit dans le `DataObject` persistant (clés `ItemDA`, `Stack`, `Rarity`, `Owner`, `AttachedToId/Slot`, `SlotAttachments`, `CustomizationInstanceId`) ; `LoadFromDataObject` recharge tout (décodage asynchrone, event `FinishedDecodeData`).
- Les DataObjects viennent du plugin **`/DataManager`** (Blueprints `GameDataManager`, `DataObject`, `PersistentDataComponent`, `DataManagerLib`) : `FindDataObjectById(Persistent=true)` + `GetDataFromDB` ; l'infra DB est fournie par les plugins `/Q_DataBase` et `/QSQL_Interface`. C'est le même canal que `PlayerPhaseData` (id « Phase♥<PlayerId> », construit par `GlobalPhaseManager.InitPlayerStatePhaseData`).
- Réplication : propriétés + OnRep, attachments répliqués en string encodée (`RepSlotAttachments`).
- L'inventaire (`InventoryComponent`, sur le pawn) persiste par le même canal (`InventoryOwnerDataObject`, `CurrentInventoryKey`, encode/décode équipement) et porte `Coins` (int64) + dispatchers achat/vente.

### 12.2 Conséquence pour les racks : réutiliser, ne rien inventer
**TRANCHÉ le 2026-07-04 : Option A retenue.** Les deux options étudiées :
- **Option A (recommandée)** : module installé = **item instance attaché** à un slot module de l'instance porteuse (mécanisme attachments existant, slots nommés ex. `Module_0..N`). Les phases insérées = clé (« Phases » = liste de tiers) sur l'instance DU module. Avantages : le module reste un vrai item (échange, vente, retrait triviaux), réplication et persistance déjà câblées.
- **Option B** : clés dédiées sur l'instance porteuse (« ModuleSockets » encodé façon `SlotAttachments`). Plus léger en DataObjects, mais ré-implémente ce que A obtient gratuitement.

### 12.3 Marchands : EXISTE
Système de shop opérationnel : `BPI_Shop`, `F_ShopItems`/`F_ShopItemData`, stocks encodés + **restock par temps** (`SAT_ShopRecoverItemsStockByTime`), UI `W_GameShopCoins`, objectif de quête `O_ShopTransactionObjective`, monnaie `Coins` sur l'InventoryComponent. Les marchands de modules = de nouveaux inventaires de shop listant les items module/phase. Rien à créer côté infra.

### 12.4 Pièces d'armes (canons, crosses) : EXISTE, éditées depuis l'inventaire
Le système d'attachments est complet dans `ItemScriptBase` (spawn par slot/socket, `AttachmentItemSpawnerHelper`, `RequestUpdateAttachments`) et l'UI vit dans les panneaux d'inventaire : `W_AttachmentsSlots` est hébergé par `W_ItemInstance`, `W_ItemDetails` et `PUW_ItemActions`. **Il n'existe PAS d'acteur « établi » gameplay** : les maps `L_ATELIER_*` sont des salles de travail de dev (level design), pas une feature. L'établi physique voulu pour les modules est donc À CRÉER (simple acteur d'interaction qui ouvre l'UI rack), question §11.7.

### 12.5 Source de la liste des modules : `DA_AllRef.Phases`
Map à la main de 7 entrées (§3.2) + 7 éléments posés en dur dans `W_PhaseTree`. Confirme le besoin du registre AssetManager.

### 12.6 Reste ouvert
- Règles de mort/drop d'inventaire : aucune fonction de drop-on-death trouvée sur `InventoryComponent` (la logique vit probablement côté `CombatComponent`/pawn) ; à trancher avec la question §11.6.
- Durée de vie des items droppés au sol entre redémarrages serveur (`F_WorldDroppedItemInstance`, `ItemsManagerGS` existent : le chargement « ItemDropDataReady » suggère une sauvegarde des drops, à confirmer par test en M5).

---

## 13. Le Mur comme surface de build (VALIDÉ le 2026-07-04)

L'ambition n'est pas un menu d'améliorations : c'est un **arbre de talents illimité** où le joueur compose un build qui lui ressemble. Proposition pour que l'EMPLACEMENT des modules compte autant que leur choix :

- **Placement libre** : le joueur pose ses modules où il veut dans les anneaux débloqués. Techniquement : `FQModule_SocketState` stocke des coordonnées hexagonales axiales (Q, R) au lieu d'un simple index (2 octets de plus par module répliqué, négligeable).
- **Adjacence** : chaque module porte des `SynergyTags` (famille). Voisins de même famille = petit bonus d'efficacité ; l'instabilité Voss peut se propager aux voisins ; les modules de **Liaison** (catalogue §4 : Connecteur, Amplificateur, Stabilisateur, Résonateur, Parasite...) font de la topologie du mur un puzzle d'optimisation.
- **Constellations** : compléter un motif hexagonal précis (dessiné visuellement sur le mur) donne un bonus de set. Très lisible avec l'esthétique hex existante.
- **Le Système Général au centre** : le mur rayonne en anneaux autour du cœur IC Lab ; la croissance du build est organique et visuelle.
- **Infini maîtrisé** : le mur s'étend sans limite visuelle. La puissance, elle, est bornée par : la capacité (slots ACTIFS, pilotée par le Système Général), les caps par StatTag, et l'exclusivité par rôle. Proposition : autoriser la pose AU-DELÀ de la capacité en état « éteint » (préparation de builds, collection), l'activation restant bornée.
- **Impacts techniques** : la passe d'adjacence s'ajoute au recalcul d'agrégation, toujours uniquement sur changement de rack (zéro tick) ; l'UI du mur doit être virtualisée ; le risque principal est la **balance combinatoire** (caps obligatoires + revue de chaque batch de contenu).
- **DÉCISIONS (2026-07-04)** : adjacence AU CŒUR DU JEU dès la V1 (bonus de famille + modules de Liaison + constellations) ; **placement libre** ; loadouts de mur en **V2** (l'architecture les prévoit : un profil = une liste de placements, mais hors périmètre V1).
