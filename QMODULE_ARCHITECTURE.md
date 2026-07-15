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

> **Le catalogue complet (142 entrées, dont les 7 modules de base en jeu, + builds + règles transverses) vit dans `Documentation/QMODULE_CATALOGUE.md`.** La liste ci-dessous n'est que l'aperçu initial conservé pour l'historique.

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
- **M1. TERMINÉ le 2026-07-04.** Squelette du plugin QModule livré : 19 fichiers, 100 % additifs (aucun fichier existant modifié, ni .uproject ni .ini), dormant par défaut (`Enabled=false` dans le C++). Contenu : Settings, tags natifs, types, Definition, Registry (scan AssetManager runtime + validation), intégration QGameManager opt-in, log/CVars, façade QMOD_*. Compilé vert : QangaEditor Win64 Development (DLL liées). Reste à compiler lors d'une fenêtre calme : cibles Qanga et QangaServer (mêmes sources, risque faible).
- **M2. TERMINÉ (volet C++) le 2026-07-04.** Rack universel répliqué (`UQModule_RackComponent` : funnel RPC serveur validé, API Authority pour les managers, OnRep événementiel, zéro tick) ; moteur d'agrégation à ordre fixe (Add, Multiply, Override, ClampMax) avec **mécanisme d'adjacence par SynergyTags** (bonus 0.0 par défaut : neutre jusqu'à l'atelier de chiffrage) ; codec de persistance versionné `QMODSOCKETS;v1` à décodage bruyant ; adaptateur cyborg 3 stats pilotes (calcul + événement BP, AUCUNE écriture dans ALS/jetpack/SS_Matter avant activation) ; harnais console `qmodule.Test.*` non-Shipping avec activation runtime sans toucher l'ini. Compilé vert QangaEditor. Le manager BP du Mur (hébergement du rack sur le PlayerState + DataObject de persistance) part avec M3. Simplification actée : PAS de classe WallObject dédiée, le Mur EST un RackComponent sur le PlayerState (réplication standard, COND_None pour l'instant, passe owner-only plus tard).
- **M3. EN COURS : moitié livrée le 2026-07-04.** FAIT : les 7 définitions historiques converties en `UQModule_Definition` dans `/Game/Phases/QModuleV2/` (QMD_*), tags historiques recopiés à l'identique (Phase.GeneralSystem, Phase.Item.Jetpack/.Drone/.Repair/.Scanner/.At56/.NashRifle), plafonds réels, StatMods chiffrés (Matter +200/400 ; fuel +25/50 % ; AT56 dégâts 30/50/70 ; Nash cadence 10/20/40 % et dégâts 5/10/15 %) ; **premier test runtime de bout en bout VERT en PIE** (registre 7/7, mur sur PlayerState, installations, rejet domaine, rejet niveau max, Matter 1000→1400, fuel 100→125, capacité qui éteint/rallume, DumpRack conforme). RESTE : la façade `PhaseComponent` (TOUCHE UN BP EXISTANT à 106 référents : REPOUSSÉE AU JOUR DE L'ACTIVATION sur décision utilisateur du 2026-07-04 : les deux systèmes vivent côte à côte d'ici là). **Binder de persistance LIVRÉ (2026-07-04 nuit)** : finalement en C++ réflexif, pas en BP : `UQModule_PersistenceBridge_World_SubSystem` (mondes de jeu, serveur only, dormant : le bind d'`OnWallHosted` est inconditionnel mais chaque handler sort si Enabled=false). CHARGEMENT : wall hébergé → id joueur via `ServerAuth.GetPlayerId` (attente auth pilotée par timer 0.25 s, timeout 60 s) → DataObject `QMODWall♥<PlayerId>` (séparateur cœur du projet, échappé `♥` en source ASCII) via `GameDataManager.FindDataObjectById(Create+Persistent)` + `GetDataFromDB` → à `IsReadyData` : `GetStringArray("Sockets")` → `QMOD_Authority_DecodeSockets`. SAUVEGARDE : `OnRackChanged` → debounce 2 s → passe de diff (encode vs dernier sauvé, le délégué ne porte pas le rack) → `SetStringArray` ; **le manager auto-sauve en SQLite sur `DataUpdated` (ready+persistent)** ; objet neuf jamais décodé → `SetReadyOverwriteWithCurrentData` d'abord (le pattern « items generation » du DataManager). Flush final au Deinitialize. Helpers réflexifs partagés dans `Private/QModule_ReflectionCall.h` (remplissage de paramètres par ordre de type via FStructOnScope). Commandes : `qmodule.Test.PersistDump` / `PersistFlush`. Testé à froid (garde-fous) ; la validation complète charge/sauve demande une map de gameplay avec ServerAuth + GameDataManager réels.
- **M3b LIVRÉ le 2026-07-04 (100 % additif, compilé et validé en PIE).** `UQModule_WallManager_World_SubSystem` : dormant sans `Enabled` ; sinon, à la connexion (GameModePostLoginEvent) il pose le rack « QModuleWall » sur le PlayerState, pré-installe les modules de base (placements configurables `BaseModulePlacements`, sinon layout auto : cœur + anneau 1), et pilote la capacité du mur par le NIVEAU DU MODULE EN (0,0) via la courbe + plancher `MinWallCapacity` (7, règle UX §14 : jamais de mur verrouillé). Zéro dépendance au legacy. Test PIE VERT : mur auto-hébergé (5 modules de base, inactifs sans phase : état « cyborg neuf »), GS monté niveau 2 → Matter 1000→1400, et **codec de persistance validé en jeu** (`qmodule.Test.SaveLoad` : encode 6 entrées → clear → decode → PASS). Correctif cosmétique noté : le layout auto place le premier module lexical au cœur (Drone) au lieu du GS → CORRIGÉ le 2026-07-04 : flag `bIsWallCore` sur la définition (posé sur QMD_GeneralSystem), le wall manager place ce module en (0,0) : vérifié en PIE (`(0,0) Phase.GeneralSystem`). Dans la foulée : `W_QModuleV2_Wall` (la COPIE) reparentée sur `UQModule_WallWidgetBase` (0 erreur, 0 warning) et `QMODED_ValidateAll` opérationnel (« 7 definition(s), all clean »). Leçons d'atelier : le bridge exécute les commandes console avec le monde ÉDITEUR (les commandes de test résolvent désormais le monde de jeu actif elles-mêmes) ; un patch de corps de fonctions dans un .cpp passe par Live Coding déclenché à distance (`LiveCoding.Compile`) sans fermer l'éditeur ; le struct GameplayTag est verrouillé en écriture côté Python (recopie de struct existant, ou ImportText via `edit_data_asset_defaults`).
- **M4.** Items Phase (meshes existants) + insertion + UI Mur v1.
- **M5.** Armes : racks par exemplaire + établi + lectures pull dans WeaponScript.
- **M6.** Véhicules : rack + adaptateur VehicleBase.
- **M7.** Actifs : AbilityBase + hotbar + 2 ou 3 vitrines (drone de soin, balise de frappe, tourelle).
- **M8.** Balance (caps par StatTag) + polish UI. **PÉRIMÈTRE RÉDUIT le 2026-07-04 (décision utilisateur)** : les marchands de modules et le PLACEMENT du loot de phases/modules ne sont PAS dans ce chantier : ils arriveront plus tard avec le système de quêtes/missions secondaires, le loot procédural des IA et les QLevels répartis dans l'univers. QModule expose seulement les briques consommables par ces systèmes (items Phase, définitions requêtables par domaine/manufacture/rareté).

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
1. Politique de respec (retrait de phases) : libre / payant / destructif ? (Bornée par la règle UX §14.1 : le RÉARRANGEMENT des modules est gratuit ; seule l'extraction de phases reste à trancher.)
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

---

## 14. Règles d'expérience joueur (garde-fous validés le 2026-07-04)

Issues de la revue « fun / frictions ». Elles PRIMENT sur les choix d'implémentation : si une contrainte technique entre en conflit avec une de ces règles, on remonte l'arbitrage.

1. **Réarrangement gratuit et fluide.** Déplacer, échanger, réorganiser les modules sur le Mur ne coûte JAMAIS rien (drag and drop, swap direct). Le puzzle est le fun, pas la manutention. Un coût éventuel ne peut porter que sur l'EXTRACTION de phases (cf. question §11.1, désormais bornée par cette règle).
2. **Révélation progressive.** Mur initial minimal (Cœur + 4 modules de base). L'UI d'adjacence n'apparaît qu'au premier cas pertinent. Les motifs de constellations ne sont PAS listés dans un menu : ce sont des **Schémas lootables** (item « Schéma : <nom> ») qui dessinent le motif sur le mur. La découverte des règles est elle-même du contenu d'exploration.
3. **Module de récompense pré-phasé.** Les modules issus de quêtes DQS et de boss tombent avec UNE phase déjà insérée (le premier contact est toujours un plaisir). Les modules sauvages et achetés arrivent vides.
4. **Aucun loot mort.** Un module doublon se recycle en **Fragments de phase** (X fragments = 1 phase tier 1, courbe à définir en M8), via le système de recyclage existant. Les doublons de phases sont utiles par nature.
5. **Établi accessible.** Un établi dans chaque hub majeur + un **établi personnel constructible via QBuilder**. Le Mur cyborg, lui, s'édite partout (c'est le système du joueur).
6. **Actifs offensifs régulés.** Utiliser un actif offensif en zone urbaine déclenche le wanted QPolice (c'est du gameplay, pas une interdiction) ; plafond d'un déployable par joueur ; cooldowns longs. À câbler sur QTriggerZone / QPolice.
7. **Étoile polaire de balance.** Un mur complet ne dépasse jamais ~35 % de puissance de combat BRUTE au-dessus du socle de base ; tout le reste de la progression est de la VERSATILITÉ (scan, économie, mobilité, options). Les nouveaux restent dangereux, les vétérans restent mortels.
8. **Sac lootable sous surveillance.** La règle « sac lootable à la mort » est conservée ; contre-jeu prévu (Assurance IOLA) ; levier d'ajustement si les playtests montrent trop de rage : pourcentage du sac qui tombe.

---

## 15. Squelette du plugin QModule (aligné sur les patterns maison, audit du 2026-07-04)

Audit croisé de 6 plugins du projet (DynamicQuestSystem, QAI, QGameManager, QRadio, CyReplicatedObject, DataManager) pour caler le squelette sur ce qui fonctionne déjà. Références : QRadio = gabarit de plugin Q récent et propre ; DQS = multi-modules, PrimaryAssets, RPC, persistance versionnée ; QAI = settings/logs/CVars ; QGameManager = contrat d'intégration au chargement ; CyReplicatedObject = état par joueur répliqué.

### 15.1 Arborescence proposée (M1)
```
Plugins/QModule/
  QModule.uplugin                  Runtime « QModule » (PreDefault) + Editor « QModuleEditor » (PostEngineInit)
  Source/QModule/
    QModule.Build.cs               Public : Core, CoreUObject, Engine, GameplayTags, DeveloperSettings
                                   Private : NetCore, CyReplicatedObject, Slate, SlateCore
    Public/
      QModule.h                    IModuleInterface + DECLARE_LOG_CATEGORY_EXTERN(LogQModule, Log, All)
      QModule_Settings.h           UQModule_Settings : UDeveloperSettings (config=Game, defaultconfig,
                                   DisplayName « QModule ») : Enabled, bVerboseLogging, courbe de capacité du Mur,
                                   accès GetDefault<> + static Get() (pattern QRadio/QAI)
      QModule_Tags.h               Tags natifs racines Stat.* / Module.* (UE_DECLARE_GAMEPLAY_TAG_EXTERN)
      QModule_Types.h              FQModule_StatMod, FQModule_AbilityGrant, FQModule_BehaviorGrant,
                                   FQModule_SocketState { Q, R, ModuleTag, InsertedPhases, Level, bActive },
                                   EQModule_Domain, EQModule_Op (enums : uint8)
      QModule_Definition.h         UQModule_Definition : UPrimaryDataAsset ;
                                   GetPrimaryAssetId() = (Type « QModuleAssets », Name = nom d'asset) (pattern DQS)
      QModule_Registry_GI_SubSystem.h  UGameInstanceSubsystem : scan AssetManager (serveur ET client),
                                   Map<Tag, Definition>, requêtes domaine/manufacture/rareté (loot, marchands),
                                   validation au boot (tags dupliqués, StatTag inconnu, icône manquante)
      QModule_RackComponent.h      (M2) UActorComponent répliqué : sockets, RPC SV_*, cache d'agrégation, OnRackChanged
      QModule_WallObject.h         (M2) UQModule_WallObject : UCyReplicatedObject_ObjectBase : le Mur par joueur
                                   (même socle que PlayerPhaseData : StartReplicated() → AddReplicatedSubObject sur l'Owner)
      QModule_AbilityBase.h        (M7) UObject BP-able : CanActivate / Activate serveur / hooks cosmétiques / cooldown
      QModule_StatLibrary.h        UBlueprintFunctionLibrary : préfixe QMOD_* (QMOD_GetStat, QMOD_GetModuleLevel...)
      QModule_AdapterComponent.h   (M2) base des adaptateurs de domaine (ApplyMap) + UQModule_CyborgAdapter
    Private/
      *.cpp + QModule_Debug.cpp    CVars qmodule.* (FAutoConsoleVariableRef) + commandes de dump
  Source/QModuleEditor/
    Public/QModuleEditor.h ; Private/ : validation du registre, hooks pour l'EUW de contrôle
```

### 15.2 Conventions héritées de l'audit (appliquées telles quelles)
- **En-tête** : `// QANGA // IOLACORP. All Rights Reserved` (l'officiel ; QAI utilise une variante « Copyright 2025 IOLACORP STUDIO », on suit l'officiel pour le neuf).
- **Logs** : `LogQModule` + macro `QMOD_VLOG` sur le modèle **QAI_VLOG** (flag settings + CDO caché + override console atomique `qmodule.Verbose`), retenu plutôt que le bool global de DQS car plus robuste.
- **CVars** : `qmodule.Enabled`, `qmodule.Debug`, `qmodule.Verbose` via FAutoConsoleVariableRef dans un .cpp dédié (pattern QAI).
- **BP** : toutes les UFUNCTION exposées préfixées `QMOD_` (pattern QAI_*/QGM_*/QPOLICE_*).
- **RPC** : `Server_*` / `Client_*` **Reliable** pour l'état, `Multicast_*` **Unreliable** pour le cosmétique (pattern DQS/projet).
- **Réplication** : DOREPLIFETIME_CONDITION_NOTIFY + REPNOTIFY_Always + OnRep_* ; en réserve si le volume l'exige : l'optimisation par **signature CRC** de DQS (ComputeReplicationSignature) pour éviter les reconstructions inutiles côté client.
- **Réflexion C++ → BP** (façade PhaseComponent, adaptateurs vers composants BP, Obj_ItemInstance) : noms FName **centralisés** dans un namespace `QModuleNames` + null-checks systématiques après FindFunction/FindFProperty (pattern DQS ScannerObjective).
- **Build** : `OptimizeCode = InShippingBuildsOnly` (pattern DQS, confort de debug) : à confirmer.

### 15.3 Intégrations décidées
- **AssetManager** (pattern DQS exact) : ajout dans DefaultGame.ini :
  `+PrimaryAssetTypesToScan=(PrimaryAssetType="QModuleAssets", AssetBaseClass="/Script/QModule.QModule_Definition", bHasBlueprintClasses=False, Directories=((Path="/Game/Phases"),(Path="/QModule")), CookRule=AlwaysCook)`
- **QGameManager** (contrat vérifié) : un `UQGameManager_System_DataAsset` dédié « QGM_System_QModule » (Direct_LoadOnRegistered=true, aucune dépendance) ; le Registry implémente `IQGameManager_Interface`, s'enregistre via `QGM_System_Register` et signale `QGM_System_IsLoaded` une fois le scan terminé. Les systèmes consommateurs pourront le déclarer dans leur `RequiredSystemBeforeLoading`.
- **CyReplicatedObject** : dépendance de module pour `UQModule_WallObject` (flux vérifié : serveur = SetOwner + StartReplicated ; client = PostNetInit → events Begin ; destruction = RPC fiable).
- **DataManager** (content-only, vérifié : pas d'API C++ à lier) : la persistance du Mur reste orchestrée par un manager BP mince (comme GlobalPhaseManager aujourd'hui : FindDataObjectById Persistent=true + GetDataFromDB) qui fournit le DataObject au WallObject ; le C++ expose seulement Encode/Decode versionnés.
- **Racks armes/véhicules** : aucun canal nouveau : clés sur `Obj_ItemInstance` (Option A, §12.2), accès par réflexion sécurisée depuis le RackComponent.

### 15.4 Périmètre exact de M1
M1 livre : module runtime + module editor, Settings + ini, tags natifs, types, Definition, Registry (scan + validation + intégration QGameManager), log/CVars, façade StatLibrary vide de logique gameplay. M1 ne livre NI rack, NI mur répliqué, NI abilities, NI UI (M2+). Compilation Windows attendue verte sur Qanga, QangaEditor et QangaServer.

### 15.5 UI v2 sur COPIES + Éditeur de modules (décisions du 2026-07-04)
- **Principe UI (décision utilisateur)** : l'écran Mur v2 se construit sur des **COPIES** de l'UI Phase existante ; l'ancien chemin (W_PhaseRouter/W_PhaseTree...) reste vivant et intouché jusqu'à l'ordre explicite de suppression. Le Mur doit épouser le design des UI du jeu tout en reproduisant la lecture de la maquette du tableau de bord (hexagones, liseré des modules de base, pastilles de phases, ambre Voss, anneaux verrouillés).
- **Copies créées** (`/Game/Widget/QModuleV2/`, originaux intacts) : W_QModuleV2_Router, W_QModuleV2_Wall, W_QModuleV2_HexCell, W_QModuleV2_Level, PUW_QModuleV2_Module, W_QModuleV2_Description. Les références internes des copies pointent encore le legacy : le recâblage sur l'API QMOD_* se fait copie par copie, jamais sur les originaux.
- **`UQModule_WallWidgetBase`** (C++) : toute la géométrie hexagonale native (HexToLocal/LocalToHex avec arrondi cubique, anneaux, distances), binding du mur du joueur local, événements `QMOD_OnWallBound`/`QMOD_OnWallChanged` (event-driven, zéro tick). Le BP enfant (la copie W_QModuleV2_Wall, à reparenter) ne porte que l'habillage.
- **Boucle de test UI** : settings `WallWidgetClass` + commande `qmodule.Test.OpenWall` (viewport direct, aucun menu existant touché).
- **CELLULE STYLÉE EN JEU le 2026-07-04** : `UQModule_HexCellWidgetBase` (C++) + BP `W_QModuleV2_Cell` fabriqué par outils (arborescence de widgets nommés, ZÉRO graphe) : fond hexagonal rempli (`T_FrameHexMask`), bordure couleur famille, monogramme généré (CORE pour le Cœur), sous-titre NIV X (rien sur les inactifs : l'atténuation suffit, règle maquette), pastilles de phases, liseré intérieur des modules de base ; slots verrouillés par le C++ (designer-proof). **Leçons d'art** : l'art hex du projet est FLAT-TOP (géométrie basculée en conséquence : x=1.5s·q, y=√3·s·(r+q/2)) et vit dans des textures CARRÉES 256x256 → cellules carrées obligatoires (l'espacement hexagonal reste mathématique). Itérations pilotées par retours visuels utilisateur (3 allers-retours corrigés à chaud par Live Coding).
- **PREMIER AFFICHAGE EN JEU le 2026-07-04** : rendu de grille 100 % natif dans la base (`Canvas_WallGrid` lié par nom via BindWidgetOptional, auto-bind du mur au Construct, cellules = HexCellClass BP optionnelle sinon images `T_FrameHex` teintées : actif teal / inactif gris-bleu / cellules libres en filigrane, anneaux 0..2, ancrage central, event-driven). Vérifié en PIE : `wall binding OK`, widget au viewport, zéro erreur runtime, Cœur GS niveau 2 actif + 4 modules de base inactifs. La copie `W_QModuleV2_Wall` n'a AUCUN graphe ajouté (habillage legacy conservé, contenu legacy replié en Collapsed, rien de supprimé).
- **Éditeur de modules LIVRÉ (2026-07-04 soir)** : finalement en **Slate pur** plutôt qu'en EUW (raison technique : `UEditorUtilityWidget` est MinimalAPI en 5.7, l'héritage C++ cross-module ne linke pas ; et un onglet Slate évite tout assemblage d'asset par le pont). Architecture : `SQModuleEditor_Panel` (`Plugins/QModule/Source/QModuleEditor/Private/QModuleEditor_Panel.h/.cpp`) monté dans un onglet nomade enregistré par `FQModuleEditorModule` (caché des menus). **Ouverture** : commande console `qmodule.Editor.Open` ou `QMODED_OpenEditor()` (BP/Python). **Fonctions** : liste triée par ModuleTag (pastille couleur famille, domaine CYB/ARM/VEH, N max, badges COEUR/BASE, « ! » rouge + tooltip si invalide), **panneau de détails moteur complet** à droite (IDetailsView de PropertyEditor : édite TOUTES les propriétés, StatMods, tags, soft refs), boutons Actualiser / Valider tout (rapport, détail en Output Log) / Tout sauver (assets dirty) / **Créer** (nom → QMD_*, tag Module.* auto-enregistré via `QMODED_EnsureTag` dans **`Config/Tags/QModuleTags.ini`, fichier NEUF 100 % additif**, asset dans /Game/Phases/QModuleV2) / **Dupliquer la sélection** (production de catalogue à la chaîne ; rappel automatique de changer le ModuleTag). Sécurité GC : lignes en `TStrongObjectPtr`. Nouvelles fonctions librairie : `QMODED_EnsureTag`, `QMODED_DuplicateDefinition`, `QMODED_OpenEditor`. Dépendances ajoutées (module éditeur uniquement) : GameplayTagsEditor, PropertyEditor, Slate, SlateCore, InputCore ; le uplugin référence le plugin GameplayTagsEditor (TargetAllowList Editor). Reste (améliorations futures) : filtre texte/famille, table de balance croisée modules x stats, bouton de création de tags de famille.

### 15.6 Interactions du Mur + items Phase (2026-07-04, validés/créés)
- **Interactions VALIDÉES EN JEU par l'utilisateur** : clic sur cellule (NativeOnMouseButtonDown → délégué OnCellClicked) → fiche module native (`UQModule_ModulePopupWidgetBase` + BP `W_QModuleV2_Popup` assemblé par outils, zéro graphe) : titre, famille (accent couleur), NIV x / max, description, boutons + PHASE / - PHASE / FERMER, états grisés automatiques, rafraîchissement par réplication. Souris libérée par la commande de test. **CONSOMMATION CÂBLÉE (2026-07-04 soir, compilé vert)** : les boutons passent par `SV_InsertPhaseFromInventory` / `SV_RemovePhaseToInventory` (nouveaux RPC du rack). Insertion = item Phase de tier le plus bas trouvé dans l'inventaire du pawn, inséré PUIS consommé via `ServerConsumeItem` (rollback du socket si la consommation échoue). Retrait = phase retirée PUIS item re-généré via `GenerateNewItemInstance`+`AddItemToInventory` (rollback si le grant échoue : aucune perte d'item possible). Pont réflexif `QModule_InventoryBridge.h/.cpp` (pattern DQS : noms centralisés, null-checks, logs bruyants, remplissage de paramètres par type via FStructOnScope, candidats multiples pour la librairie de génération : Lib_ItemSystem puis Lib_Inventory). Identification des tiers par comparaison de chemin avec `PhaseItemAssetByTier` (nouvelle map de Settings, soft refs vers les 6 IDA, défauts en constructeur). Les anciens RPC `SV_InsertPhase`/`SV_RemoveLastPhase` restent le chemin libre/admin (commandes de test). POLITIQUE D'EXTRACTION : retrait = remboursement intégral en v1 ; le coût éventuel (atelier respec §11) se règlera dans ces deux RPC uniquement.
- **Items Phase créés** (`/Game/Items/QModulePhase/`, 100 % additifs) : 6 paires `IDA_QModulePhase_T1..T6` + `IS_QModulePhase_T1..T6` (enfants d'ItemScriptBase, composant PhaseMesh = les meshes `ModulePhase1..6` endormis), stack 10, droppables, ramassables, AvailableAdminSpawn=true, icônes T_PhaseTier1..3 (placeholder au-delà). **RESTE pour la boucle complète** : (1) test de ramassage + consommation dans une map de gameplay avec le vrai pawn (le menu admin existant peut déjà donner les items : AvailableAdminSpawn=true) ; (2) ACTIVATION UNIQUEMENT : inscription des 6 clés `QModulePhase_T*` dans `DA_AllRef.ItemKey:DAItem` (modification d'un asset existant : interdite pour l'instant, consignée dans la checklist d'activation). La consommation elle-même est FAITE (voir 15.6 ci-dessus).

### 15.7 Distribution des phases : audit du système VIVANT + design d'attribution v2 (2026-07-04)

**Audit du circuit actuel (vérifié dans QangaPlayerState)** :
- **La SEULE source gameplay de points de phase est le LEVEL-UP** : `SS_Level` broadcast « Level Up » → `LevelUp_Event` dans QangaPlayerState → `AddPhasePointsByActor(CurrentLevel - LastLocalCurrentLevel)` : 1 point par niveau gagné. L'XP (`AddExperienceByActor`) vient du gameplay général. Le même event ajuste la taille du stockage joueur par paliers de niveau.
- Source secondaire : `SV_AdminPhasePoint` (+1, permission Admin).
- **Tentative ITEM abandonnée découverte** : `PUW_ItemPhase` (popup fiche d'item phase avec bouton AddPhase) est un ORPHELIN (0 référent, bouton neutralisé par un AND false codé en dur), et les meshes `ModulePhase1..6` n'avaient aucun référent : l'équipe avait déjà esquissé des phases-items puis abandonné. La vision v2 en est l'aboutissement.
- Pattern d'octroi d'item DISPONIBLE dans le même BP : `SV_AdminGetItem` = `GenerateNewItemInstance(IDA, Stack, Rarity, Persistent=true)` + `AddItemToInventory` : c'est exactement le grant à réutiliser.

**Design d'attribution v2 (les briques QModule ; l'implémentation des sources reste aux systèmes concernés)** :
1. **Continuité au jour 1** : le hook `LevelUp_Event` donnera **1 item Phase T1** au lieu d'1 point (même rythme, zéro rééquilibrage : remplacer l'appel AddPhasePoints par le pattern GenerateNewItemInstance + AddItemToInventory avec `IDA_QModulePhase_T1`). MODIFICATION D'UN BP EXISTANT : jour d'activation uniquement.
2. **Les tiers supérieurs (T2+) ne viennent JAMAIS du level-up** : réservés aux sources d'exploration (quêtes/missions secondaires via DQS, loot procédural des IA, QLevels répartis dans l'univers, boss) : c'est le moteur de la vision « récompense d'exploration ». Ces systèmes consommeront les briques QModule (IDA_QModulePhase_T1..T6) quand leurs chantiers respectifs arriveront (HORS périmètre QModule, décision utilisateur).
3. **Table de rareté de référence** : `Lib_Phase.RandomWeightedPhaseChance` (50/30/18/1 % → 0/T1/T2/T3) sert de graine aux futures tables de loot IA.
4. **Migration** : solde de points converti en items T1 à la bascule (déjà acté §8) ; `SV_AdminPhasePoint` conserve son rôle legacy jusqu'à extinction ; `SV_AdminGetItem` sait DÉJÀ donner les items Phase (AvailableAdminSpawn=true posé sur les 6 IDA) : **le test de ramassage/octroi peut passer par le menu admin existant, sans nouveau code**.

**STATUT M1 (2026-07-04) : LIVRÉ.** Compilé vert sur QangaEditor Win64 Development (Result: Succeeded ; DLL `UnrealEditor-QModule.dll` + `UnrealEditor-QModuleEditor.dll` liées). Particularités de livraison : scan des définitions par `ScanPathsForPrimaryAssets` runtime (AUCUNE entrée AssetManager dans DefaultGame.ini : hermétisme total), plugin actif d'office comme plugin projet (pattern QRadio : ni .uproject ni EnabledByDefault), 3 verrous de dormance (Enabled=false, intégration QGM opt-in x2, façade neutre). Cibles Qanga/QangaServer : à compiler plus tard. Activation le jour J : `[/Script/QModule.QModule_Settings]` `Enabled=True`. Leçon d'atelier : UBT refuse de compiler tant que l'éditeur tourne avec Live Coding (et un NOUVEAU plugin ne passe jamais par Live Coding) : fermer l'éditeur pour les jalons C++.

### CHECKLIST D'ACTIVATION CONSOLIDÉE (source de vérité, complétée par la revue scénarios du 2026-07-06)

> **ACTIVATION LOCALE DEV FAITE le 2026-07-10** : `Enabled=True` posé dans `Saved/Config/WindowsEditor/Game.ini`
> et `Saved/Config/Windows/Game.ini` de la machine de dev (fichiers locaux non versionnés, créés pour l'occasion ;
> le `DefaultGame.ini` d'équipe reste dormant ; revert = supprimer ces 2 fichiers). Répétition boot-enabled
> PASSÉE sans aucune commande de test : registre 93/93 scanné au boot, mur hébergé au login, persistance
> restaurée à travers le redémarrage, façade active sur les 6 consommateurs (fallback -1 vérifié).
> Les points ci-dessous restent OBLIGATOIRES avant l'activation ÉQUIPE/PROD.

Modifications d'existant autorisées UNIQUEMENT ce jour-là, dans cet ordre :
1. `[/Script/QModule.QModule_Settings]` `Enabled=True` dans DefaultGame.ini (+ `bAutoHostVehicleRacks` selon décision). **DÉCISION RzZz 2026-07-11 : AUCUNE CONVERSION, tout le monde repart à zéro à l'activation (le script 8.4 est ANNULÉ, la question du mapping jetpack legacy L1 disparaît). L'ancien état SS_Phase/PlayerPhaseData reste en base, simplement ignoré.**
2. **COOKING (bloquant build packagé, trouvé en revue scénarios)** : TOUS nos assets sont chargés par chemins soft depuis le C++ (QMD_* scannés runtime, W_QModuleV2_* via Settings/soft paths, items QModulePhase/QModuleWeapon/QModuleVehicle) : AUCUN référenceur dur → **ils ne seront PAS cuits** sans : entrée AssetManager `PrimaryAssetTypesToScan` pour QModuleAssets (couvre les QMD_*) + `DirectoriesToAlwaysCook` (ou graine EasyCook, le projet utilise DA_EasyCookSeed_QANGA) pour /Game/Widget/QModuleV2, /Game/Items/QModulePhase, /Game/Items/QModuleWeapon, /Game/Items/QModuleVehicle. À tester par un cook complet AVANT la release.
3. **LOCALISATION (règle CLAUDE.md, trouvé en revue scénarios)** : tous les textes joueur de l'UI v2 sont EN DUR en ASCII (« MODULES ACTIFS », « NIV », légende des familles, établi, fiche, panneau latéral) : passe String Tables/NSLOCTEXT obligatoire (en/fr/es) avant prod.
3bis. **OBSOLETE (résolu en mieux le 2026-07-10 soir)** : les phases ne sont PLUS DU TOUT des items. Correction RzZz appliquée en C++ : les phases sont des POINTS DE COMPETENCE dans un `PhaseWallet` (TArray<int32> par tier) porté par le rack MUR du PlayerState (répliqué owner-only, persisté dans la même clé `Sockets` via une entrée `QMODWALLET;v1`, roundtrip SQLite prouvé). L'inventaire d'objets ne voit plus jamais une phase : aucun filtre UI nécessaire. Les IDA_QModulePhase_T* ne servent plus qu'à la conversion lazy des vieilles saves de dev (`Authority_ConvertLegacyPhaseItems`, appelée au restore et à chaque insertion) ; à supprimer du projet à terme. Les MODULES restent des items physiques (voulus).
3ter. **JETPACK 3 NIVEAUX (décision RzZz 2026-07-10)** : data déjà appliquée (QMD_Jetpack MaxLevel=3) ; le REMAP des gates IS_JetPack (activation>=1, rapide>=2, stationnaire>=3) et l'offset de conversion (+1) sont sur l'étage 2 : détail dans QMODULE_ACTIVATION_ALIGNMENT.md §8.3bis.
4. Inscription des items MODULES dans `DA_AllRef.ItemKey:DAItem` (les clés `QModulePhase_T*` sont OBSOLETES : les phases sont des points, plus des items).
5. Bascule LevelUp : **FAITE côté C++ le 2026-07-11, zéro édit BP** : `Authority_BindLegacyLevelUp` (rack) se binde par réflexion au dispatcher `LevelUp` du SS_Level du joueur (résolu via la map `StatScriptClass:StatScriptSpawned` du StatsComponent, signature vérifiée), retenté par les timers post-login du WallManager ; chaque level-up crédite max(1, delta) point(s) T1 au portefeuille. Le legacy `AddPhasePointsByActor` continue en parallèle (inoffensif : plus rien ne consomme les points legacy). Test : `qmodule.Test.LevelUp` (pipeline réel IncrementLevel). Validation en jeu : RzZz.
6. Façade `PhaseComponent` : **ÉTAGE 1 FAIT ET PROUVÉ EN PIE le 2026-07-10** (répétition générale, détail dans QMODULE_ACTIVATION_ALIGNMENT.md §8) : GetCurrentPhase et CallPhaseUpdate lisent le NIVEAU du mur v2 via `UQModule_LegacyFacade::QMOD_GetLegacyPhaseLevel` (SelectInt pur, -1 = retombe legacy octet-identique en dormant ; backup `PhaseComponent_BACKUP_PreFacade` en place ; sonde `qmodule.Test.Facade`). La re-notification est FAITE aussi (2026-07-10, prouvée en live) : push C++ côté producteur (`QMOD_NotifyLegacyPhaseComponents`, câblé dans MarkRackDirty/OnRep_Sockets + les 4 mutations d'item-rack), zéro edit BP supplémentaire. RESTE l'étage 2 : lecture des stats AGRÉGÉES par les consommateurs réels (jetpack, armes via WeaponScript, véhicules via VehicleBase).
7. Entrée établi dans le catalogue QBuilder (`UQBuilder_Data_ActorDataBase.InputData`, ID stable réservé) + coûts `ResourceData` + mesh.
8. Multi réel : valider les lectures de rack d'item côté client distant (données DataObject serveur : prévoir réplication du codec si l'UI client en a besoin) et re-dérouler l'E2E en listen + dédié.
9. Limite connue : le flush de persistance au changement de monde est best-effort (fenêtre du debounce 2 s : une insertion faite < 2 s avant un travel peut se perdre ; l'auto-save par changement couvre le reste).

### 15.10 CHANTIER « 82 MODULES CYBORG » : campagne de branchement étage 2 (démarrée 2026-07-11, priorité RzZz)

Décision RzZz 2026-07-11 : priorité aux modules CYBORG du catalogue (armes/véhicules plus tard). Objectif : chaque module de la liste développable de manière fonctionnelle et fun. La table des leviers vérifiés est dans QMODULE_ACTIVATION_ALIGNMENT.md §5.1.

**Le pattern de branchement validé (Servomoteurs, 2026-07-11)** : insertion de NŒUDS PURS dans le BP consommateur, au fil de la donnée, via `UQModule_StatLibrary::QMOD_GetStat(self, Stat.X, base)` (BlueprintPure, passthrough neutre si plugin OFF, résout le mur via le PlayerState). JAMAIS de flux exec touché. Backup systématique du BP avant édit. Exemple livré : ALS_Base_CharacterBP `UpdateDynamicMovementSettings` : MaxWalkSpeed final = (chaîne existante ×0.85) × SelectFloat(QMOD_GetStat(SprintSpeed, 1.0), 1.0, GetMappedSpeed() > 2.5) : le facteur ne s'applique qu'au sprint, identique serveur/client owner (data répliquée), compilé 0 erreur. Backup : `ALS_Base_CharacterBP_BACKUP_PreQMOD`.

**Lot 1 : leviers OK (édits purs, un module = un edit = un test)** :
- [x] Servomoteurs de Jambes (sprint) : FAIT 2026-07-11, test de course réelle par RzZz à faire.
- [x] Amortisseurs Cinétiques : FAIT 2026-07-11 (SV_OnLanded : dégât × (1 - QMOD/100), nœuds purs, compilé 0 erreur).
- [x] Nano-Régénérateur : FAIT 2026-07-11 via la nouvelle `QMOD_GetStatForObject` (les stat scripts ne sont pas des acteurs et `OwnerStatsComponent` est private : résolution par chaîne d'outer côté C++). Vigilance : vérifier en jeu que la résolution d'outer aboutit (sinon VLOG « no owning actor resolvable » et plan B).
- [x] Sac Digitique Étendu : FAIT 2026-07-11 (UpdateInventorySize : terme Round(GetStat) ajouté entre la somme et le Max(0)).
- [x] Négociateur : FAIT 2026-07-11 (le DynamicRate passe par GetStatForObject au point de consommation ; l'affichage PUW_Shop du prix de vente reste à harmoniser).
- [x] Nano-Réparateur de Drone : FAIT 2026-07-11 (temps Select × (1+v), appliqué au Delay ET au feedback client).
- [x] Blindage de Drone : FAIT 2026-07-11 (Selection du Switch d'impacts = clamp(phase + ImpactsAdd, 0, 3) : sémantique +N hits avec le plafond existant).
- LEÇON backups : ne PAS dupliquer les composants BP self-référencés (copie incompilable + dialogue au Play) : export T3D + doc des édits à la place. Le backup InventoryComponent a été supprimé (T3D conservé), les 4 autres compilent proprement.
**Lot 2 : PARTIEL (pré-requis à créer)** :
- [x] Compacteur de Matière : FAIT 2026-07-11 (MaxStackPerSlot × (1+v) aux 2 sites de lecture, arrondi bas, min 1 ; les stacks existants sur-remplis à l'extraction du module restent valides, seuls les nouveaux ajouts respectent la limite réduite).
- [x] Blindage Sous-Cutané : FAIT 2026-07-11 : étape d'ARMURE créée en tête de Lib_Life.ApplyStatDamageToActor : dégât effectif = max(0, dégât - Armor.Flat de la CIBLE) avant NoMatter/bouclier/vie. S'applique à tout acteur avec un mur (passthrough sinon). DÉCISION EN ATTENTE (RzZz) : câbler le DamageReductionPercent des 11 équipements (casques/torses, donnée jamais lue) dans la même étape ?
- [ ] Caisson Hermétique : filtre DropAllItemsDeath (flux exec : à faire avec soin, fonction dupliquée PawnBP/CharacterBP).
- [ ] Surcouche de Bouclier : nécessite l'infrastructure BehaviorGrant (accorder SS_Shield quand le module est actif) : premier consommateur du pipeline de grants.
**Lot 3 : les coquilles restantes du catalogue** : chiffrer les StatMods dans l'Éditeur de Modules puis brancher famille par famille (les tags Stat.* existants sont réutilisables ; créer les manquants en natif).

Règles de campagne : un backup par BP touché ; nœuds purs only ; toute nouvelle stat = tag natif QModule_Tags ; test PIE par module (install + phase + effet mesuré) ; jamais plus d'un BP existant modifié par lot de validation.

### 15.8 M5 Armes + M6 Véhicules : couche de données livrée (2026-07-04 nuit, compilée verte)

**Décision de modèle v1 (à valider par l'utilisateur)** : le rack d'un EXEMPLAIRE d'arme vit dans **une clé write-through de son instance** (`SetStringArray("QMODRack", codec)` sur `Obj_ItemInstance`) : vérité serveur, et **persistance GRATUITE portée par l'item lui-même** (la clé voyage avec l'instance dans son DataObject). Les items modules sont **consommés à l'installation et remboursés au retrait** (le pattern éprouvé du Mur, rollbacks anti-perte identiques). L'Option A complète (modules = instances d'items VIVANTES attachées via `SetAttachmentToSlot`/`Slot:AttachmentId`, API vérifiée dans le binaire d'Obj_ItemInstance) reste la cible d'activation si on veut l'usure/la rareté par module : la clé codec v1 y migre trivialement.

- **`QModule_ItemRack.h/.cpp`** (namespace `QModuleItemRack`) : GetSockets (décode + RecomputeDerivedState), InstallModule (validation domaine/exclusivité/déjà-installé + consommation de l'item module mappé dans `Settings.ModuleItemAssetByTagName`, slots linéaires Q=index), RemoveModule (refuse si phases insérées ; rembourse l'item), InsertPhaseFromInventory / RemovePhaseToInventory (mêmes règles que le Mur), GetStat (BuildStatAggregates SANS adjacence), DumpRack. Codec partagé : `QMOD_Encode/DecodeSocketArray` statiques extraits du rack (les méthodes existantes délèguent, aucun contrat changé).
- **`QModule_VehicleRack_World_SubSystem`** (M6) : hook `FOnActorSpawned` (serveur, monde de jeu) qui pose dynamiquement un `UQModule_RackComponent` (DomainFilter=Vehicle, adjacence OFF, capacité illimitée, répliqué) sur tout acteur dont la chaîne de classes contient `VehicleBase_C` : **zéro Blueprint touché**. `QMOD_EnsureRacksForExistingVehicles` pour les véhicules déjà spawnés. LIMITE v1 : racks véhicules **session-only** (l'identité de sauvegarde des véhicules est un sujet d'activation).
- **Pont d'inventaire généralisé** : `FindInstanceByAssetPath` + `GrantItemAsset` (GrantPhaseItem délègue).
- **Settings** : `ModuleItemAssetByTagName` (map tag→IDA soft, 3 défauts de test : Module.CanonRenforce, Module.ChargeurRapide, Module.NoyauSurcadence).
- **Harnais** : `qmodule.Test.Weapon.Slots/Dump/Install/Remove/InsertPhase/RemovePhase/Stat` (opèrent sur l'item ÉQUIPÉ du pawn local via la map `Slot:ItemInstance` lue génériquement) ; `qmodule.Test.Vehicle.EnsureRacks/List/Install/InsertPhase/Dump/Stat`.
- **LIMITE ASSUMÉE (les deux domaines)** : en dormant, rien ne branche les stats agrégées dans WeaponScript ni VehicleBase (BP existants) : la consommation VIVANTE des stats (dégâts réels, vitesse réelle) est sur la checklist d'activation, comme la façade cyborg. L'établi physique (acteur + UI) est le prochain morceau M5.

### 15.9 L'Établi (M5 partie 2) : acteur + UI + chemin QBuilder (2026-07-05)

**Exigence utilisateur** : l'établi doit être posable par les designers dans les levels ET **constructible par le joueur dans sa base via QBuilder**. Audit des sources C++ QBuilder (local, sans pont) :
- Une entrée constructible acteur = un DataAsset **`UQBuilder_Data_ActorData`** : `ActorClass` (N'IMPORTE QUELLE classe d'acteur), `ActorIsReplicated/ActorAlwaysOnServer/ActorPersistantOnServer`, `LifeData` (santé structure), `ResourceData` (coûts minéraux), `Mesh_View_Actor` (fantôme de placement). **La persistance des acteurs construits est portée par QBuilder** (world save + respawn par ID d'entrée).
- Le catalogue vivant = **`UQBuilder_Data_ActorDataBase`** (asset existant côté /Game) : `TMap<int32, ActorData*> InputData`. **Inscrire l'ID de l'établi = modifier un asset existant = JOUR D'ACTIVATION** (comme AllRef). Pour tester sans rien toucher : injection RUNTIME de l'entrée dans la map en mémoire (pattern qmodule.Test.Enable) : l'établi apparaîtra dans le menu construction d'une session de test.

**Livré (compilé)** : `AQModule_WorkbenchActor` (acteur répliqué neuf : mesh + zone d'interaction + `QMOD_OpenWorkbench(PC)` : l'interaction maison s'y branchera en un nœud, le harnais l'ouvre directement) ; **`UQModule_WorkbenchWidgetBase`** (UI 100 % native, RebuildWidget crée sa racine : AUCUN asset requis, un BP enfant pourra la réhabiller via `Settings.WorkbenchWidgetClass`) : 3 colonnes : ÉQUIPEMENT (lecture générique de la map Slot:ItemInstance via `GetEquippedInstances`, promue dans le pont d'inventaire) / RACK DE L'OBJET (sockets + NIV x/max + boutons ± PHASE et RETIRER) / MODULES EN INVENTAIRE (croisement Settings x inventaire, bouton INSTALLER) ; statut + rafraîchissement différé après chaque action. **Canal serveur** : 4 nouveaux RPC sur le rack du PlayerState (`SV_Item_InstallModule/RemoveModule/InsertPhase/RemovePhase`) qui délèguent à `QModuleItemRack` (validé E2E) : le widget client n'appelle jamais les fonctions server-only en direct. Commandes : `qmodule.Test.Workbench.Spawn` (pose un établi devant le pawn, bypass QBuilder) et `Workbench.Open` (ouvre l'UI du plus proche à portée). RESTE : assets au retour du pont (ActorData établi + coûts, mesh, entrée QBuilder en activation), branchement interaction maison, et le domaine Véhicule à l'établi (v1 = armes).

**E2E AUTONOME COMPLET VALIDÉ (2026-07-04 ~21h35, L_Dev_Start / Survival_GM, éditeur reconstruit à froid)** : mur (item Phase généré → consommé → Drone niveau 1), arme (item module consommé → phase insérée → `Stat.Weapon.Damage` 100→110 sur instance réelle), véhicule (racks auto-posés sur les véhicules du trafic aérien de la map → Noyau surcadencé → `Stat.Vehicle.Speed.Max` 100→110), persistance (DataObject `QMODWall♥0` résolu, auto-save SQLite « 6 entrie(s) », **mur RESTAURÉ après redémarrage de session**). Correctifs décisifs découverts par le test : (1) les fonctions de LIBRAIRIE BP portent un paramètre caché `__WorldContext` à remplir PAR NOM sinon `GenerateNewItemInstance` rend null ; (2) **les entrées tableau/struct BP passent par référence et portent `CPF_OutParm|CPF_ReferenceParm`** : le remplissage réflexif doit les traiter comme des ENTRÉES (c'était LE verrou de `FindDataObjectById` et de l'écriture de la clé de rack) ; (3) `Obj_ItemInstance` n'expose PAS Get/SetStringArray : l'API clé vit sur sa propriété `DataObject` ; (4) les pawns dev spawnnent avec `InventoryMaxSize=0` et `AddItemToInventory` jette en silence (filet de test `SetInventoryBaseSize`) ; (5) après `AddNewGameplayTagToINI`, VÉRIFIER le fichier sur disque (un crash éditeur peut avaler l'écriture non flushée : 2 tags perdus puis recréés) ; (6) discipline Live Coding : 2-3 rounds max par session d'éditeur, ensuite rebuild à froid (au-delà : UClass pourris → crash `ForEachSubsystem` au teardown PIE ; stack confirmée par l'utilisateur). Commandes de test ajoutées : `qmodule.Test.GivePhase <Tier> [N]` et `GiveModuleItem <Tag>` (les dons remplacent le menu admin pour les tests).
