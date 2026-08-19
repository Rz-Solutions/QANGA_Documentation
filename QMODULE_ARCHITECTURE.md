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

### 15.11 Standard de designation + vaisseau de renfort (2026-07-29, livre et valide en jeu)

**Regle de design (RzZz)** : le jeu ne peint JAMAIS un marqueur d'interface au sol pour dire
"ca tombe ici". Tout module actif qui appelle quelque chose se designe en LANCANT une balise
physique (`AQModule_StrikeBeaconActor`) : la ou elle se plante, la charge arrive. Elle est
visible de tous, elle justifie le delai, et elle remplace toute UI au sol. Modules a balise :
`Module.BaliseDeFrappe`, `Module.LargageDeRavitaillement`, `Module.ProtocoleDeMeneur`
(`UQModule_RackComponent::QMOD_IsBeaconModule`). **L'armement dorsal en est exclu par
decision** : missiles et grenades d'epaule sont des armes, elles tirent droit.

**Le geste, identique pour tous les modules (revise 2026-08-12, lot 1 de la refonte
designation)** : maintenir la touche module (X, preset `Char_GadgetFire`, rebindable) pour
viser, CLIC GAUCHE pour lancer, RELACHEMENT DE X pour annuler (seule voie d'annulation :
le bind brut clic droit -> annuler a ete retire, le clic droit garde son sens arme/pointage
et ne participe plus au geste). Un arc holographique et un anneau d'impact montrent le point
de chute REEL (l'apercu balaie la parabole en cherchant les collisions), donc un
declenchement par accident est impossible. Etats du reticule : RECHARGE, HORS PORTEE,
PAS DE CIEL.

**La MEME grammaire vaut pour l'ordonnance dorsale depuis le 2026-08-12 (RzZz : "X seul
ne doit jamais tirer")** : missiles d'epaule et grenades collantes n'ont plus d'appui
direct : X MAINTENU les ARME (reticule nom + icone, triggers d'arme suspendus), le CLIC
GAUCHE lache la salve, relacher X remet au repos. `HandleFirePressed` est scinde : le
dispatch de tir historique vit dans `FireSelectedDirectGadget()`, appele au clic via
`BeginOrdnanceArm`/`CancelOrdnanceArm`. Les toggles utilitaires (drone medical, radio,
rappel de flotte) gardent l'appui direct : ils ne tirent rien.

**Presence corporelle du geste (2026-08-12)** : pendant tout le maintien, le composant
pilote le canal de pointage du doigt EXISTANT du personnage (heartbeat par frame local +
relais serveur a 0,05 s, extinction garantie par le watchdog 0,25 s du character BP), dirige
vers le point de chute reel de l'arc ; et il RANGE l'arme tenue le temps du geste (variante
B validee RzZz, facon Helldivers) via la primitive repliquee du systeme d'items, puis la
ressort au desarmement. Le geste expose aussi `OnTargetingStateChanged` (BlueprintAssignable)
et tente le hook optionnel `QMOD_TargetingStateChanged(bool)` sur le pawn (no-op tant que
le BP ne le definit pas).

**CONTRATS REFLEXIFS DU GESTE (ne pas renommer ces membres BP sans mettre a jour
`QModule_GadgetHUD.cpp`, namespace `QModulePointingBridge`)** : sur `ALS_Base_CharacterBP` :
`UpdatePointingFinger`, `SV_PointingFinger`, la variable `InventoryComponent` ; sur
`InventoryComponent` (BP) : `GetActiveItem` ; sur `ItemScriptBase` : `GetItemIsHidden`,
`LocalSetItemIsHidden` (choisie parce qu'elle est AUTO-REPLIQUEE : elle route vers
`SV_SetItemHidden` et `IsHidden` a un OnRep ; ne PAS remplacer par `SetActiveItem`, qui
change l'item actif ET declenche une sauvegarde d'inventaire). Les parametres sont ecrits
PAR TYPE (premier vector, premier bool), donc un renommage de PARAMETRE est tolere ; un
renommage de FONCTION casse en silence (log une fois via QMOD_VLOG).

**Suppression du tir pendant le geste** : le mode vide temporairement
`UCurrentInputData::CurrentInputCombos` de `Combat_1stTrigger`, `Combat_2ndTrigger`,
`Camera_AimMode` et `UI3D_RightClick` (4e bind du clic droit a pied, rate par l'audit du
2026-07-29, ajoute le 2026-08-12) sur le client local, et les restaure a la sortie et a
l'EndPlay (`Settings->TargetingSuspendedPresets`). Ne PAS utiliser `BlockWhenOthersPressed`
pour ca : son test depend de l'ordre d'iteration d'une TMap dans la frame, donc il ne
bloque que parfois.

**Briques partagees** : `AQModule_ThrownDeviceActor` (une parabole en forme fermee, evaluee a
l'identique partout depuis {depart, vitesse, up, gravite, horodatage serveur} repliques une
fois ; resolue dans le repere local du lanceur, jamais en Z monde) ; enfants =
`AQModule_StrikeBeaconActor` et `AQModule_StickyGrenadeActor` (salve dorsale collante qui
detonne en ligne, fusee = index x ChainDelay pour que le rythme ne depende pas du terrain) ;
`QModule_TrackerBridge` (acces reflexif partage au framework Lib_Tracker/TK_* du jeu).

**LA VISEE ET LA COLONNE D'ORDONNANCE (lots 2-3 de la refonte, 2026-08-12).**
L'apercu de visee est passe des spheres moteur aux TIRETS LASER orientes (mesh
/Engine/BasicShapes/Plane + materiau `M_QModule_HoloDash`, cree par script dans
/Game/Widget/QModuleV2) : 34 tirets qui DEFILENT vers le point de chute (billboard
cylindrique vers la camera), anneau de 20 tirets tangentiels tournant lentement + 4 ticks
cardinaux fixes, couleurs alignees sur le HUD terrain (valide #EB8D0C, invalide #FF4046,
defauts dans `UQModule_Settings::TargetingValid/InvalidColor`). Le reticule affiche en plus
l'icone du module arme (20 px) et la DISTANCE au point de chute reel en metres.

La balise plantee dresse la **colonne d'ordonnance** : coeur + gaine (2 cylindres moteur,
materiau `M_QModule_Beam` : parametres Color/Intensity/PulseSpeed/PulseTiling/PulseStrength/
BaseGlow/TopFadePower), impulsions montantes qui ACCELERENT sur le compte a rebours ; pour
la frappe uniquement, a l'heure du tir, un CONE descend du ciel et se comprime sur la zone
pendant `AirstrikeArrivalDelaySeconds`, tient serre pendant le barrage, puis tout fond sur
les 2 dernieres secondes du linger. Anneau de zone au sol (12 tirets plats) au rayon EXACT :
`ZoneRadiusCm` est replique sur la balise (un float, une fois), calcule au lancer. TOUTE la
timeline client derive de `FireAtServerTime` + les deux delais partages en Settings
(`AirstrikeArrivalDelaySeconds` / `BarrageSpreadSeconds`, qui pilotent AUSSI le serveur :
ce que la colonne annonce est ce qui arrive). Couleur par module : frappe `StrikeBeamColor`
#FF931E, ravitaillement `SupplyBeamColor` (la couleur de chute existante de la caisse),
meneur `LeaderBeamColor` #D9942F ; le no-op historique User.Color du NS_TaserBeam emprunte
est mort avec lui. La balise porte enfin un traceur HUD pour son APPELANT seul
(`BeaconTracker`, TK_Drop par defaut, duree calculee d'avance : les TempTracker ne se
prolongent pas apres coup) et chaque impact de missile pousse un
`UQModule_StrikeCameraShake` via PlayWorldCameraShake (epicentre = impact, attenuation
`StrikeShakeInner/OuterRadiusCm`). Dependance module ajoutee : `EngineCameras` (les shake
patterns Perlin vivent la sur UE 5.7, PAS dans GameplayCameras). Le MC
`MC_AirstrikeMissileVisual` porte desormais le point d'impact (3e parametre).

**COULEURS DU DOCK GADGET (2026-08-13, demande RzZz).** Le dock et la roue ne portent
plus les couleurs de familles : chaque logo de module ET chaque contour de cellule
prennent l'orange des icones du HUD general (`GadgetHudCellColor`, defaut #FF931E
mesure), et la cellule SELECTIONNEE (surlignee dans la roue, armee en mode compact)
passe logo + contour au vert (`GadgetHudSelectedColor`, defaut #17B98F, le vert
matiere du HUD terrain). Les deux sont des reglages Config (QModule|GadgetHUD).
Mecanique : `UQModule_HexCellWidgetBase` expose `SelectionColor` (defaut ambre
historique) et `bTintIcon` (defaut false) : LE MUR NE CHANGE PAS (familles + ambre),
seul le dock active le mode teinte. `ResolveFamilyColor` du dock a ete supprimee
(plus d'appelant).

**AUDIO DE LA DESIGNATION (lot 4, 2026-08-12).** Sept soft refs optionnelles dans
`UQModule_Settings` (null = silence, jamais une erreur), servies par le pack maison
/Game/Sounds/_Ressource/IndieGameModel/SF_Meca. **Verdict RzZz du meme jour** : une
passe de remplacement par des MetaSounds synthetiques purs (`QMS_QModule_*`, toujours
sur disque dans /Game/Widget/QModuleV2/Audio avec leur script de construction) a ete
JUGEE PIRE ("trop sobre, ca fait genere a l'IA") et revertee : les waves organiques
du pack restent la reference, les QMS_* servent de base a une future passe manuelle.
Le clonk de plantee est joue a pitch 0.85 (adouci). Roles : `TargetingArmSound` /
`TargetingCancelSound` (2D, sortir/ranger le dispositif ; le blip d'annulation se TAIT
sur un lancer, flag `bCommitInProgress`), `BeaconThrowSound` (depart de main, saute au
late-join car bPlanted est deja replique), `BeaconPlantSound` (clonk), `BeaconBeepSound`
(LE bip de liaison : meme cadence accelerante que la lumiere et les pulses, 1,5 a 9 Hz,
pitch montant avec l'urgence), `StrikeFireSound` (confirmation au tir, tous payloads),
`StrikeIncomingSound` (boucle de grondement, frappe seulement : enfle en volume et en
pitch sur la descente du cone, FadeOut apres le barrage, coupee a l'EndPlay). Les waves
du pack ne sont pas routees en SoundClass (l'etat de ~89 % du projet) : la passe de mix
globale reste un chantier separe.

**Vaisseau de renfort** (`AQModule_DropshipActor`) : decor cosmetique qui fait venir l'escouade
par les airs sur la balise. Il spawne le VRAI `IronDee_Lavrik` (la variante `_Police_Nodrive`
herite d'Actor : un sac de meshes sans moteur ni porte). Pieges rencontres et corriges :
allumer l'etat vehicule reveille aussi la logique de vol (le vaisseau est parti a 1,7 km en
emportant la passerelle) donc on rendort ses composants de mouvement et son tick d'acteur
A CHAQUE IMAGE, et on recolle l'acteur enfant a la transform pilotee ; ses propulseurs animes
poussent la coque a chaque image, ce qui logue 1500+ warnings quand la collision est coupee,
donc la coque garde un corps CINEMATIQUE qui ignore tous les canaux ; son enfant
`ShippingLandingArea` part en Accessed None sur un decor, on le detruit ; une interpolation
droite traverse les montagnes, donc `MeasureCruiseHeight` echantillonne le sol sous toute la
course et le vaisseau croise au-dessus, ne descendant que sur la derniere fraction
(`DropshipDescentFraction`) ; le point de largage est lu sur le composant `SM_IcliShip_SM_Path`
du vaisseau, jamais estime.

**Unique modification cote QAI** : `UQAI_CyborgRecruitmentComponent::QAI_SetSummonSpawnOverride`
(point de spawn du prochain rappel d'escouade, a expiration automatique), appele par reflexion
depuis `Authority_ReleaseSquadFromDropship`. Le recrutement normal sur la carte est intact.

**Un piege moteur, general** : `FTimerManager::SetTimer` avec un delai <= 0 EFFACE le timer
au lieu de l'armer (une grenade sur quatre disparaissait silencieusement de chaque salve).

> **CORRECTION 2026-07-29 (audit reseau).** Ce paragraphe affirmait un second piege :
> `FVector_NetQuantize*` deborderait a l'echelle de QANGA (plafonds 2^20 a 2^24) et
> mutilerait les positions chez les clients distants, avec cinq signatures a corriger.
> **C'est faux sur UE 5.7.** Depuis la version reseau `PackedVectorLWCSupport` (5.1+),
> `FVector_NetQuantize::NetSerialize` passe par `UE::Net::WriteQuantizedVector`
> (`Net/Core/Private/Net/Core/Serialization/QuantizedVectorSerialization.cpp`) : le
> nombre de bits par composante est **dynamique**, les plafonds pour un `FVector` double
> sont 2^52 et 2^62, et au-dela le code bascule explicitement en pleine precision au lieu
> de clamper. Le `20` de `SerializePackedVector<1,20>` ne sert plus qu'au chemin de
> lecture herite (`LegacyReadPackedVector`), face a un moteur anterieur a 5.1. A 5,2e7
> unites, un `FVector_NetQuantize` arrive donc **arrondi au centimetre**, sur ~88 bits au
> lieu de 192 : il est correct ET moins cher qu'un `FVector` plein. Les signatures
> concernees sont conservees telles quelles (decision RzZz 2026-07-29). Le vrai probleme
> de portee planetaire est ailleurs : voir 15.12.

### 15.12 Passe multijoueur (2026-07-29)

Rien de 15.11 n'avait jamais tourne ailleurs qu'en PIE solo, ou la pertinence reseau
n'est jamais evaluee et ou rien n'est serialise. Audit puis correctifs.

**Portee reseau (le bug principal).** Aucun acteur QModule ne declarait sa portee, donc
tous heritaient du defaut moteur : `NetCullDistanceSquared = 225000000`, soit **150 m**.
Or tout ce plugin voyage plus loin : la balise se designe jusqu'a 250 m (frappe) et le
vaisseau vient de 1,5 km. Consequence mesurable : au-dela de 150 m **le lanceur ne
recevait meme pas sa propre balise**, et toute l'approche du vaisseau (18 s de scene)
etait invisible sauf les 150 derniers metres. Chaque acteur declare desormais sa portee
dans `PostInitializeComponents`, depuis `UQModule_Settings` (categorie `QModule|Network`,
en metres) : device lance 600, caisse 900, vaisseau 3000, drone medical 400, etabli 5000
(aligne sur QBuilder). Meme famille de reglage que `QBuilder_BuilderActor` (5 km) et
`QAI_AgentSpawner`. La frequence de mise a jour reseau descend a 10 Hz (2 Hz pour
l'etabli) : ces acteurs changent deux fois dans leur vie, pas cent fois par seconde.

**Vaisseau de renfort : plus de `UChildActorComponent`.** `IronDee_Lavrik` herite de
`SpaceshipBase` -> `VehicleBase` -> `APawn`, et le constructeur d'`APawn` met
`bReplicates = true`. Or `UChildActorComponent::CreateChildActor` **sort en debut de
fonction sur toute machine non autoritaire** quand la classe enfant replique (elle attend
la copie du serveur). Chez le client, `GetChildActor()` etait donc nul : pas de
neutralisation, pas de porte, pas de recollage de transform, et c'est le vrai vehicule
replique du serveur qui arrivait, avec sa collision et sa logique de vol intactes (elles
n'avaient ete endormies que cote serveur). Le vaisseau est maintenant spawne **localement
sur chaque machine** (`SpawnShip`), `SetReplicates(false)` avant `FinishSpawning`, detruit
dans `EndPlay`. C'est ce qu'un decor cosmetique doit etre : purement local, pilote par les
parametres de vol repliques de l'acteur porteur.

**Multicasts cosmetiques filtres a l'arrivee.** Le rack vit sur le `PlayerState`, qui est
`bAlwaysRelevant` : un `MC_*` parti de la atteint **toutes** les connexions. Sans filtre,
une salve d'epaule faisait charger la classe projectile et spawner des acteurs chez 500
clients autour d'un combat qu'aucun ne peut voir. `QModuleOrdnance::IsWorthRendering`
compare le point au point de vue local (`OrdnanceVisualRangeM`, 800 m par defaut).

**Caisse de ravitaillement : modele deterministe.** C'etait le seul acteur du lot en
`SetReplicatingMovement(true)`, pour une chute en ligne droite a vitesse constante, et
ses parametres de chute n'etaient pas repliques (donc `UpVector`, qui oriente la poussiere
d'impact et le panache au sol chez le client, valait le defaut : faux sur une planete).
Elle suit maintenant le meme modele que les objets lances : {depart, point au sol,
vitesse, horodatage serveur} repliques une fois, chute evaluee a l'identique partout,
seul `bLanded` decide par le serveur.

**Horodatages en `double`.** `ThrowStartServerTime` et `SceneStartServerTime` etaient des
`float`. `GetServerWorldTimeSeconds` est un double qui croit avec l'uptime : a 1e6 s
(11 jours, banal sur un serveur persistant) un float ne resout plus que 1/16 de seconde,
donc les arcs deterministes, echantillonnes chaque image, se mettent a saccader. Les six
`*ReadyAtServerTime` du rack **restent en float expres** (un cooldown de 10 a 300 s s'en
moque, et c'est de l'etat replique sur chaque rack d'un serveur a 500 joueurs).

**Divers.** `SV_TriggerAirstrike` comparait son cooldown a `World->GetTimeSeconds()` alors
que le champ est documente et relu par le HUD en temps monde serveur : aligne sur les cinq
autres gadgets.

**Ce qui a ete MESURE (PIE serveur d'ecoute, 2 instances, 2026-07-29)** : instruments neufs
`qmodule.Test.NetCensus` (recensement de tous les acteurs QModule dans TOUS les mondes du
process, avec role reseau et distance), `qmodule.Test.NetSpawnProbe <beacon|crate|grenade|
dropship> [DistanceM]` (spawn cote serveur, sans pawn) et `qmodule.Test.GodWallAll`.
Resultats, tous relus dans `Saved/Logs` :
- **Portee** : vaisseau present dans le monde CLIENT a **1686 m**, puis 1272, 592 et 91 m,
  suivi sans interruption. Au defaut moteur de 150 m, le client n'aurait rien recu.
- **Vaisseau local** : la ligne `Dropship: ship 'IronDee_Lavrik_C_0' neutralised
  (80 primitive(s)), engine started` apparait **DEUX fois**, une par monde. Avec l'ancien
  `UChildActorComponent` elle n'apparaissait que cote serveur.
- **Determinisme** : balise plantee a la position **identique au millimetre** entre serveur
  (role Authority) et client (role SimulatedProxy), a 5,19e7 unites de l'origine. Caisse :
  environ 1,2 m d'ecart pendant la chute (le client lit une horloge serveur en retard de sa
  latence, c'est le modele), **convergence exacte a l'atterrissage**.
- **Chaine complete en reseau** : 4 grenades collantes repliquees dans les deux mondes,
  degats appliques (`Ordnance impact ... 1 actor(s)`), caisse `6/6 supplies unpacked`,
  escouade larguee (`Leader Protocol: squad released from the ramp`). Zero erreur QModule.

**PAS ENCORE TESTE** : le serveur DEDIE. Et un obstacle a connaitre pour toute session
automatisee future : `BaseGameMode` demarre les joueurs en SPECTATEURS et les fait posseder
par un flux Blueprint de chargement, donc une session PIE non pilotee par un humain a des
PlayerController mais **aucun pawn**. C'est la raison d'etre de `NetSpawnProbe`, qui part du
point de vue au lieu du pawn. Autre piege outillage : `ULevelEditorPlaySettings::PlayNetMode`
est un `TEnumAsByte` que le Python de l'editeur ne sait ni lire ni ecrire ; le mode
multijoueur de PIE se regle dans
`Saved/Config/WindowsEditor/EditorPerProjectUserSettings.ini`, **editeur ferme**.

**Verifie sain, a ne pas re-auditer** : la suspension d'entrees du geste (le
`UCurrentInputData` est resolu par l'`InputsComponent` du PlayerController, donc aucune
fuite entre joueurs ; restauration a `EndPlay` et a chaque tick des que le pawn n'est plus
valide, ce qui couvre la mort et le changement de pawn) ; l'accroche de la grenade
collante (`AttachmentReplication` reste actif malgre `SetReplicatingMovement(false)` :
seul un RootComponent replique la desactive) ; l'attache serveur du drone medical ;
l'apercu de visee (`bReplicates = false`, purement local) ; le HUD pose sur le seul
controleur local.

**Couches maison, verdict** : `QNet` est un registre de clients, pas un substitut de
replication ; `ClientAuthority` + `SmoothTSync` delegueraient l'autorite d'un acteur lourd
au client le plus proche, contre-indique ici (artillerie deterministe et courte, degats
serveur) ; `QNetState`/OptimizedState est un bus d'etat cle/valeur indexe par cle de
localisation, pertinent pour de l'etat spatialise persistant, pas pour des RPC de tir ;
`CyReplicatedObject` sert aux UObject repliques hors acteur. La replication UE brute est
le bon choix pour ce plugin ; ce qui manquait etait le reglage de portee.

**Reste ouvert apres ce lot** : sons (une seule ligne de son dans tout le plugin), lanceur
dorsal visible et mesh de micro-missile (assets a fournir), mesh d'etabli, HUD permanent
des gadgets, animations du mur (exigent de passer la grille en mise a jour incrementale),
cooking des nouvelles soft refs, String Tables pour les textes du reticule.

### 15.13 Transpondeur de transit : porte QModule sur le reseau QAssistance (2026-07-31)

Chantier "retirer l'Assistance des menus de base et la recycler en module". Design valide RzZz
le 2026-07-29, arbitrages tranches le 2026-07-31. **Le detail du design, l'audit chiffre du systeme
QAssistance et les trois corrections au cadrage sont dans QMODULE_CATALOGUE.md par.9.23.** Cette
section ne porte que le contrat technique.

**Sens de la dependance : QAssistance (BP) interroge QModule (C++), jamais l'inverse.** C'est le
pattern de branchement etage 2 valide en par.15.10 : insertion de NOEUDS PURS dans le BP
consommateur, au fil de la donnee, via `UQModule_StatLibrary`. Aucun code C++ n'est ajoute a
QModule pour ce chantier, donc **aucun rebuild, aucun risque de Live Coding sur QModule**.

**Livre et verifie au 2026-07-31 :**
- `QMD_TranspondeurDeTransit` (`/Game/Phases/QModuleV2/`), tag `Module.TranspondeurDeTransit`
  enregistre dans `Config/Tags/QModuleTags.ini` (162 tags). Domaine Cyborg, famille
  `Module.Family.System`, rarete 2, MaxLevel 3, icone `025-signal`, **zero StatMod** (porte pure :
  `QMOD_GetModuleLevelForActor` renvoie deja 0 si absent ou non alimente en phase).
  Verdict QATS `20260731_131624_1672` : contract_status **Passed**.
- Les 6 stations orbitales portent `Transit.Orbital` dans leur `Data_Tags` (champ vide sur les
  200 stations avant ce jour). Lecture du palier d'une station :
  palier 1 = `Type == RELAY_TOWER` / palier 2 = `Type == WARP` ET `Data_Tags` contient
  `Transit.Orbital` / palier 3 = `Type == WARP` sans ce marqueur.

**Reste a brancher. Specification exacte des 4 edits BP.** Chacun est une insertion de noeuds
PURS, flux exec intouche, backup du BP avant edit (les 4 assets sont sauvegardes dans le
scratchpad de session du 2026-07-31).

1. **Masquer l'onglet** (`/Game/Widget/W_GameplayMenus`). Poser sur `W_Button_Assistance` une
   visibilite `Visible` / `Collapsed` selon
   `QMOD_HasModule(GetOwningPlayerPawn(), Module.TranspondeurDeTransit, 1)`.
   Le bouton est le SEUL point d'entree du systeme (mesure), donc `Collapsed` suffit a le rendre
   injoignable : son handler `On Released` ne peut plus tirer.
   **Piege** : ce widget fait `SetWidgetAsSingleInstance` au Construct, donc un gate pose
   uniquement sur `Event Construct` ne se re-evalue pas si le joueur sockete le module en cours de
   session. La forme robuste est un **binding de visibilite UMG** (fonction pure evaluee par Slate)
   plutot qu'un set ponctuel.
2. **Filtrer les paliers** (`/Game/Systems/QAssistance/Widget/ListView/QAssistance_ListView`,
   fonction `Compute_Data`). Le noeud `Set ValidStation [Valid Station <- 'Get Valid_Station']`
   devient `Valid_Station AND (palier de la station <= QMOD_GetModuleLevelForActor(pawn, tag))`.
   C'est le gate FONCTIONNEL, et il est re-evalue a chaque ouverture de la liste : sans module,
   zero destination valide. Il tient meme si le gate cosmetique du point 1 est en retard d'une
   session.
3. **Plancher de prix par type** (`/Game/Systems/QAssistance/QAssistance_Client`). Aujourd'hui
   `BasePrice` est un unique entier a 200 pour toutes les destinations. Le remplacer par une
   lecture par type (relais bon marche, orbital cher, warp tres cher) au point ou `Compute_Price`
   et `Compute_Data` additionnent `Get BasePrice`. Quitter le sol doit etre un evenement.
4. **Refus lisible**. `Compute_Price` renvoie deja `Valid=false` quand le solde est insuffisant, et
   le seul retour joueur est un `Cy_PrintString "Price fail"` en `Print to Screen = false`. Le canal
   propre existe deja a cote dans le meme graphe : `Make_PopUp [Target <- 'Get
   StarMap_UI_PopUp_System']`, avec un type rouge deja utilise par `ReturnCreateAssistance`.
   Y brancher le refus (solde insuffisant) et le refus de palier (module trop bas).

**Non livre, et pourquoi.** Les 4 edits ci-dessus n'ont pas ete graves depuis le pont :
`insert_code` exige un noeud SELECTIONNE dans un editeur ouvert (donc un humain), et
`generate_blueprint_logic` ne sait qu'ajouter en bout de chaine via un Sequence, ce qui ne produit
pas un gate en tete de chaine. De plus `get_api_context` est **casse sur cette installation**
("Could not load API reference file (ue5_api_reference.json)"), donc les noms de fonctions
generes ne sont pas validables avant ecriture. Graver a l'aveugle dans le hub de menus du jeu
n'etait pas un risque acceptable.

**Notification de decouverte** (arbitrage RzZz : l'onglet disparait, donc la feature doit se faire
connaitre). A poser sur l'approche d'une station relais, une seule fois par profil. Aucune quete ne
peut servir de relais pedagogique : **Q008 enseigne l'aerotram**, un systeme different
(`TK_AeroTram`, levels `L_Capital_AeroTram`, plugin QTrain), pas le menu Assistance.

**Localisation.** Les 3 descriptions de niveau du QMD sont en `INVTEXT`, aligne sur les 104 autres
QMD du catalogue. C'est la dette deja consignee au point 3 de la checklist d'activation
(passe String Tables / NSLOCTEXT avant prod), pas une regression introduite ici.

### 15.14 Lock d'ordonnance dorsale : de la scrutation au verrou tenu (2026-08-02, compile vert)

**Le constat RzZz** : "le petit HUD au niveau de l'IA visee ne reste pas longtemps, il faudrait qu'il
reste tout le temps affiche, et qu'on puisse avoir plusieurs cibles". Le multi-cible existait DEJA
(le serveur repartit la salve en round-robin sur la liste recue, jusqu'a 16 cibles), mais le CLIENT
le bridait au nombre de missiles et lachait ses marqueurs sur cinq conditions independantes.

**Ce qui faisait disparaitre le marqueur** (tout etait dans `QModule_GadgetHUD.cpp`) :
1. sortie d'un cone de 35 degres autour de l'axe camera, sans hysteresis : tourner la tete cassait
   tout ;
2. un `LineTraceTestByChannel` vers l'origine de l'acteur, sans delai de grace : un tronc ou un
   chambranle pendant une frame, et le lock sautait ;
3. l'eviction par le plafond : les candidats etaient tries par alignement puis tronques au nombre de
   missiles, donc un ennemi plus proche du reticule EJECTAIT un lock parfaitement valide ;
4. `ClearAllLocks()` juste apres le tir : le HUD se vidait alors que les missiles etaient encore en
   vol vers ces cibles ;
5. la duree de vie du marqueur. C'est le piege non evident : le marqueur est un `TempTracker` cree
   avec 1,2 s de vie, rafraichi par reflexion a chaque passe. **`TempTracker` n'expose que
   `ResetDestroyDelay`** ; ni `ResetLifetime` ni `SetLifetimeSeconds` n'existent dessus, donc ces
   deux appels, y compris le failsafe de `QModule_TrackerBridge.cpp`, sont des no-op silencieux.
   Verifie par recherche binaire dans `Content/Systems/Tracker/TempTracker.uasset`.

**Le modele retenu** : un lock est TENU, pas scrute. Il s'ACQUIERT dans un cone reglable et se GARDE
ensuite. Conditions de liberation, et elles seules : la cible meurt, elle sort de la portee du
module, ou elle reste hors de vue au-dela de la fenetre de grace. Le cone ne participe plus a la
retention. Le tir ne libere plus rien.

**Reglages** (`QModule|GadgetHUD`, retunables sans recompiler) : `GadgetLockConeDegrees` (35),
`GadgetLockMaxTargets` (6, DECOUPLE du nombre de missiles), `GadgetLockLostSightGraceSeconds` (3),
`GadgetLockMarkerLifetimeSeconds` (600, long EXPRES : le HUD est proprietaire de la suppression du
marqueur, le timer du tracker n'est plus qu'un filet, et la boucle repose un marqueur que le
framework aurait detruit sous elle).

**Eviction par vol de slot, avec marge** : plateau plein, une nouvelle cible ne peut prendre que la
place du lock dont le joueur s'est le plus clairement DETOURNE, et seulement si elle le bat d'une
marge (`LockStealMarginCos`, environ 2 degres). Deux fausses bonnes idees ecartees en chemin :
evicter le moins bien aligne SANS marge (deux ennemis a angle presque egal se volent le meme slot a
chaque passe), et evicter le PLUS ANCIEN, qui parait equitable mais fait tourner tout le plateau a
chaque passe des qu'on se tient au milieu d'une foule. Les deux reproduisent le clignotement que
cette refonte existe pour supprimer.

**Les grenades collantes entrent dans le systeme.** Elles n'avaient aucun lock. Elles en prennent un
maintenant, mais restent balistiques et collantes : seul le CENTRE de la ligne d'impacts se pose sur
la cible tenue au lieu du trace brut du reticule. La portee de lob (5000 cm) quitte la constante en
dur de `SV_TriggerShoulderGrenades` pour `Settings->ShoulderGrenadeRangeCm`, **lue des deux cotes** :
un lock que le serveur refuserait ne doit jamais recevoir de marqueur, sinon le joueur peint une
cible et la salve entiere est refusee sans un mot.

**Ce qui n'a PAS ete fait, et pourquoi.** Pas de purge au clic droit, alors qu'elle etait au plan
initial : le clic droit est deja bind globalement et non consommant (`HandleCancelClick`), donc il
sert aussi l'ADS de l'arme. Purger les locks a chaque visee aurait rendu le verrou tenu inutile. Les
regles de liberation plus le vol de slot couvrent le besoin sans toucher a l'ergonomie de tir.

**Consequence assumee** : un ennemi qui passe derriere un mur garde son marqueur pendant la grace
(le tracker se dessine en surcouche HUD). `GadgetLockLostSightGraceSeconds` est le bouton, 0 rend le
comportement d'avant.

**Reseau** : aucun changement de contrat. Tout est client ; le serveur revalidait deja la liste
(cible vivante, a portee, 16 maximum) et continue de le faire.

### 15.15 Le joueur sait toujours ou il en est : refus lisibles + lecture permanente (2026-08-02, compile vert)

**Le constat, mesure et pas suppose.** Le serveur refusait une action de module dans **20 cas**
distincts, et les 20 partaient dans `QMOD_VLOG`, c'est a dire dans un log de developpeur. Cote
joueur : appui sur la touche, rien du tout. Deux choses trainaient dans le code, ecrites et
inutilisees :
- `ShowTransientReticleMessage` (`QModule_GadgetHUD.cpp`), un canal de message au reticule complet,
  avec creation du widget et auto-masquage a 1,6 s : **zero site d'appel dans tout le projet** ;
- les six temps de recharge, **deja repliques au proprietaire** (`COND_OwnerOnly`) et deja lisibles
  par `QMOD_GetGadgetReadyTime`, affiches nulle part hors de la roue.

La partie chere, le reseau, etait faite depuis le debut. Il manquait le branchement.

**Ce qui est livre.**
1. **Garde-fou client devant chaque module a APPUI DIRECT** (ordonnance dorsale, drone, rappel de
   flotte, radio) : `CanPressSelectedGadget` verifie module actif puis recharge et affiche
   `MODULE INACTIF` ou `RECHARGE Ns` au reticule, au lieu d'envoyer un RPC voue au refus. Les modules
   a balise n'y passent pas : leur geste de visee portait deja ses refus (`RECHARGE`, `HORS PORTEE`,
   `PAS DE CIEL`).
2. **Refus specifiques** la ou le code se contentait d'un `return` : `AUCUNE CIBLE` (salve de
   missiles sans verrou ni visee), `HORS PORTEE` (grenades au-dela du lob), `PAS D'ANTENNE` (module
   radio actif mais aucun composant radio sur le pawn), `AUCUN MODULE ARME` (roue jamais ouverte).
3. **Lecture permanente du module arme** : la roue ne quitte plus l'ecran, elle passe en mode COMPACT
   (`EDockMode`), la cellule armee seule, **a sa propre place dans la grappe** pour que l'ouverture de
   la roue fasse pousser le cluster autour d'elle au lieu de la deplacer, opacite 0,70, avec son
   decompte de recharge. Elle s'efface quand rien n'est arme ou quand le joueur n'est pas a pied.
4. **Signal de retour a disposition** : la lecture comptait a rebours sans jamais dire "c'est bon".
   Une breve enflure de la cellule (0,35 s, +18 % en pointe, par-dessus l'echelle du surlignage, pas
   a la place) marque le passage a zero. Volontairement petite : ce bloc est a cote du reticule, il
   s'annonce et il s'efface.

**Le piege du drone medical, et pourquoi il y a une variable repliquee de plus.** La touche du drone
est un TOGGLE, et cote serveur le RAPPEL s'execute **avant** la verification de recharge
(`SV_TriggerMedicalDrone` : un drone vivant se replie immediatement). Un garde-fou client sur la
recharge aurait donc refuse le rappel et laisse le drone en l'air jusqu'a expiration. Le client ne
pouvait pas trancher seul : `ActiveMedicalDrone` est un `TWeakObjectPtr` serveur. D'ou
`bMedicalDroneDeployed`, miroir replique `COND_OwnerOnly`, ecrit dans les deux seuls endroits qui
touchent l'original. **C'est le detail qui transforme une amelioration en regression** : ajouter une
verification cliente devant un RPC oblige a relire le serveur ligne par ligne, jamais a supposer sa
forme d'apres son nom.

**Ce qui reste muet, volontairement.** Les revalidations serveur que le client ne peut pas predire :
cible sortie de portee pendant le vol du RPC, liste de verrous surdimensionnee (client trafique),
stat `HealPerSec` a 0 sur un QMD mal cable. Elles ne se declenchent que sur une latence limite ou sur
une donnee mal autorisee, jamais sur un geste normal.

**Localisation** : tous les textes passent par `NSLOCTEXT("QModule", ...)`, comme les etats de
reticule existants. La dette String Tables du plugin reste identique, elle n'augmente pas.

**Cadence** : le composant HUD tique desormais a 0,12 s des qu'un module est arme (avant : seulement
quand l'ordonnance dorsale etait selectionnee), parce que la lecture permanente doit suivre le
changement de pawn et le decompte. Le widget, lui, ne tique que quand il est visible, et en mode
compact il ne parcourt que la cellule affichee. `HandleRackChanged` reconstruit maintenant la roue
meme fermee, sinon la cellule compacte resterait perimee apres un changement de niveau de module.

### 15.16 Le Mur repond aux clics : canal de resultat rebranche (2026-08-02, compile vert)

**Le constat, et il est pire que celui du 15.15.** Le Mur avait DEJA tout : neuf phrases de refus
ecrites et localisees (`QMOD_DescribeActionResult`), un RPC client dedie (`CL_ActionResult`), un
delegue de diffusion (`OnActionResult`), et meme un commentaire affirmant "le joueur recoit toujours
une reponse, succes ou refus". **C'etait faux.** Recherche binaire dans tout `Content/` plus
recherche C++ : `OnActionResult` n'avait **aucun abonne**, ni Blueprint ni natif. Le tuyau etait
construit jusqu'a un metre de la sortie, et les phrases n'ont jamais atteint un ecran.

**Ce qui manquait vraiment.** Le geste le plus repete de l'ecran, l'insertion de phase, n'atteignait
meme pas ce canal : `SV_InsertPhaseFromWallet`, `SV_RemovePhaseToWallet`, `SV_InsertPhase` et
`SV_RemoveLastPhase` refusaient dans un `UE_LOG` / `QMOD_VLOG` avec un `FString Error` riche qui ne
quittait jamais le serveur.

**Ce qui est livre.**
1. `UQModule_WallWidgetBase` s'abonne a `OnActionResult` (dans `QMOD_BindWall`, desabonnement dans
   `QMOD_UnbindWall`) et affiche la phrase dans une ligne native construite en fin de
   `QMOD_BuildChrome` : ajoutee EN DERNIER pour passer au-dessus des autres couches, en bas au
   centre et **decalee a gauche de la largeur du panneau lateral**, donc centree sur la GRILLE, la
   ou le regard est deja. Ambre pour un succes, rouge sourd pour un refus, effacement a 4 s.
2. Quatre valeurs ajoutees a `EQModule_ActionResult`, **apres `Rejected`** pour qu'aucune valeur
   existante ne change de numero : `PhaseMaxLevel`, `PhaseNone`, `PhaseReserveEmpty`,
   `NoModulePicked`. `TryInsertPhase` et `TryRemoveLastPhase` prennent le meme parametre
   `EQModule_ActionResult* OutResult` optionnel que `TryInstall`/`TryRemove` avaient deja.
3. **Refus de phase seulement, jamais les succes** : une insertion reussie se voit deja sur les
   pastilles de niveau de la cellule, et c'est le geste le plus repete de l'ecran. Un bandeau par
   phase serait du harcelement. Les modules, eux, gardent leur message de succes : l'action est plus
   rare et plus lourde de consequences.
4. **Clic sur une case libre sans module choisi** : c'est litteralement le premier geste d'un joueur
   qui decouvre le Mur, et il ne produisait rien. Refus client, dit sur place sans aller-retour
   serveur, en reutilisant la meme phrase localisee.

**Regle qui se degage des deux passes.** Un canal de retour joueur n'est pas livre quand il est
ecrit : il est livre quand quelque chose l'AFFICHE. Chercher `OnXxx.Broadcast` sans abonne, et les
`FText` de refus sans site d'appel, est un audit rapide qui trouve cette classe de trous d'un coup.

**Deux etats muets fermes dans la foulee.**
1. **Un module installe mais INACTIF n'expliquait rien.** La fiche affichait "NIVEAU 0 / 3" meme
   quand le joueur y avait mis deux phases, ce qui nie son investissement au lieu de l'expliquer.
   La fiche montre desormais le niveau REELLEMENT insere, plus une ligne rouge qui donne la raison.
   Le jeu de raisons n'est pas devine : `RecomputeDerivedState` pose
   `bActive = (Level > 0) && bAllowed && Definition`, et sur un mur `bAllowed` vaut
   `Ring <= CoreLevel * WallRingsPerCoreLevel`. Il ne reste donc que deux causes reparables par le
   joueur, aucune phase inseree ou anneau verrouille, et c'est exactement ce que la ligne dit.
2. **La roue de gadgets vide** affichait un bandeau sombre sans un mot quand aucun module
   declenchable n'etait installe. Elle annonce maintenant l'etat et dit quoi faire.

**Fausse piste ecartee, a ne pas re-"corriger".** `CachedCoreLevel` dans `QModule_WallWidgetBase`
ne contient PAS le niveau du noyau malgre son nom : ligne 345, il vaut deja
`CoreLevel * RingsPerLevel`. La comparaison `QMOD_HexDistance(...) > CachedCoreLevel` qui decide de
l'affichage verrouille est donc correcte et alignee sur l'autorite. La renommer serait utile, la
"reparer" serait une regression.

### 15.17 Le Mur a une voix et les phases un visage (2026-08-13, compile vert, chemins mesures en PIE)

**Audio du mur.** Huit soft refs optionnelles dans `UQModule_Settings` categorie `QModule|Wall`
(null = silence, jamais une erreur, le contrat de la designation) : `WallOpenSound`,
`WallCloseSound`, `WallSelectSound`, `WallInstallSound`, `WallModuleRemoveSound`,
`WallPhaseInsertSound`, `WallPhaseRemoveSound`, `WallDenySound`, plus `WallSoundVolume` (0.45)
et `WallSoundPitch` (0.8), memes valeurs que la designation puisque c'est la meme banque
SF_Meca (verdict 2026-08-12 : les waves organiques battent les MetaSounds synthetiques).
Lecture par `PlayWallSound` (2D, client pur, `LoadSynchronous`). Les waves `UI/` du pack sont
routees `QSClass_UI` ; `Clonk_Small` (install) et la paire `Elevator_Small` (phases) ne sont
routees nulle part, comme le clonk de la designation : dette de mix connue, pas ce chantier.

**Le succes de phase voyage enfin.** `SV_InsertPhaseFromWallet` / `SV_RemovePhaseToWallet`
n'emettaient `CL_ActionResult` que sur refus (regle anti-harcelement : pas de toast sur le geste
le plus repete de l'ecran). Deux codes APPENDUS en fin d'enum (`PhaseInserted`, `PhaseRemoved`,
la regle du commentaire de `EQModule_ActionResult` : rien ne se decale sur le fil) sont maintenant
emis sur le chemin de succes. Cote client `HandleActionResult` les traite SON SEULEMENT et sort
avant le bandeau : la regle anti-harcelement a change de camp, elle est devenue une decision de
presentation par code, pas une absence d'information.

**Le refus sonne la ou il s'affiche.** `WallDenySound` est joue en tete de `ShowActionMessage`
quand `bRefusal`, AVANT la garde de chrome : il couvre d'un seul point les refus serveur
(HandleActionResult) et le refus purement client "aucun module choisi" de `HandleCellClicked`.
Ne pas le deplacer dans `HandleActionResult` : le refus client ne repasserait plus par lui.

**Ouverture / fermeture : c'est l'onglet qui parle.** Le hub ne pilote que la visibilite du TAB
(`UQModule_LegacyPhaseSwap`), jamais celle du mur imbrique. `HandleTabVisibilityChanged` filtre
desormais les vraies transitions visible/cache (`bLastTabVisible`, seme sur l'etat reel au
construct) et relaie vers `QMOD_NotifyTabShown` (refresh du stock + son d'ouverture, remplace
l'appel direct a `QMOD_RefreshStock`) / `QMOD_NotifyTabHidden` (son de fermeture, verrouille par
`bTabShownOnce` : un hub qui collapse ses onglets inactifs au boot n'est pas une fermeture).
LIMITE CONNUE : un mur ouvert HORS hub (`qmodule.Test.OpenWall`, action de quete
`A_OpenModuleWall` du tutoriel) ne recoit aucun signal d'onglet, donc pas de son d'ouverture ;
si on le veut la, l'appelant appelle `QMOD_NotifyTabShown` apres l'AddToViewport.

**Les phases ont un visage.** La tuile du stock EST l'item de phase (design RzZz 2026-08-13 :
"T1..T6 ne parle pas aux joueurs, montre l'objet") : son logo en identite (30 px, teinte
`GTierColors`), le compte possede comme seul texte. Le label "T&lt;n&gt;" ne revient qu'en SECOURS
si aucune icone ne se resout, pour ne jamais laisser une tuile anonyme. Icone lue par
`ResolvePhaseTierIcon` : `PhaseItemAssetByTier[Tier]` -> `IconAsTexture` de l'IDA par reflexion.
DOUBLE CONTRAT PAR CHAINE, nom ET TYPE : la propriete est un `TSoftObjectPtr<UTexture2D>`
(mesure via get_data_asset_details), donc une **FSoftObjectProperty** ; un
`FindFProperty<FObjectProperty>` (type dur) rend null SANS ERREUR et l'icone ne s'affiche
jamais, c'est exactement le bug de la premiere passe. La resolution attrape desormais soft
puis dur. Une seule source de verite : re-skinner l'item re-skinne le panneau. Il n'existe que
4 textures (`T_PhaseTier0..3`), T4/T5/T6 pointent l'art de T3 : c'est la TEINTE `GTierColors`
qui distingue les hauts tiers (compte a zero = icone eteinte alpha 0.45). **L'inventaire ne
montre que T1..T3** (design RzZz 2026-08-13) : mesure sur les 104 QMD du disque, MaxLevel vaut
2 (x13) ou 3 (x91), JAMAIS plus, et les tiers 4..6 n'ont aucune source en jeu (audit 07/2026).
Une tuile T4..T6 ne s'affiche que si le wallet en contient reellement (console de test, legacy) :
l'ecran de tous les jours lit trois tuiles, un stock reel n'est jamais cache. Le hint sous
"PHASES EN STOCK" est reformule et passe en `NSLOCTEXT` ("Les phases s'inserent dans un module
installe et augmentent son niveau"), seul texte localisable du chrome avec les deux etats
INACTIF ; le reste passe par `FText::AsCultureInvariant` (dette de loc connue, par.15.13).

### 15.18 Les sons marchaient en editeur et etaient MUETS en build : deux bugs (2026-08-14)

Rapporte par RzZz apres un packaging : mur, designation ET feedback du HUD gadget, tous muets en
build alors qu'ils jouent en editeur. L'audit (11 agents, mesures sur la build Steam installee) a
trouve **deux causes independantes**, pas une.

**BUG 1 : les assets ne sont jamais cuits.** Un `TSoftObjectPtr` dont le chemin n'existe que
comme litteral dans un constructeur C++ n'a **aucun referenceur dur** : ni l'AssetRegistry ni
l'UHT ne le voient, le cooker ne l'atteint pas. En editeur `LoadSynchronous()` resout contre le
`/Content` en vrac sur disque et tout marche ; en build il n'y a plus que des paks, donc
`LoadPackage` emet `SkipPackage` et le pointeur revient NUL. Preuve dans le log du jeu livre
(`Steam/steamapps/common/Qanga/Qanga/Saved/Logs/QANGA.log:2546,2549,2552,2556`) :
`SkipPackage: /Game/Sounds/_Ressource/.../Scifi_Interface_DeviceA_Open_Wav - The package to load
does not exist on disk or in the loader`. Mesure independante sur le manifeste livre :
**34 des 106 waves SF_Meca cuites, et 0 des 10 sons concernes** (`Manifest_UFSFiles_Win64.txt`).
Les 5 sons du mur qui marchaient le faisaient PAR HASARD, via un referenceur sans rapport
(`Env_Call_03`, `Env_DoorAuto_Open_3`, `Generator_PowerON`, `S_TUTO_MISSION_1_FALL_CYBORG`) :
si un de ces assets change, ils cassent sans avertissement.
FIX APPLIQUE : une ligne dans `Config/DefaultGame.ini` (bloc `DirectoriesToAlwaysCook`, apres les
6 lignes QModule existantes) qui force-cuit `/Game/Sounds/_Ressource/IndieGameModel/SF_Meca`.
L'entree est RECURSIVE (`CookOnTheFlyServer.cpp` utilise `FindFilesRecursive`), ce qui est
indispensable ici : 7 des 10 sons vivent dans le sous-dossier `SF_Meca/UI/`. Controle positif que
le mecanisme est honore dans ce projet : `/Game/Sounds/Dialogue` est cuit 73/73.

**BUG 2 : le code n'etait pas dans le binaire.** Independant du cook. Scan d'octets du `Qanga.exe`
livre : **zero occurrence** des 8 chemins du mur ni de `WallOpenSound`/`PlayWallSound`, alors que
les 7 chemins de la designation y sont (2 occurrences UTF-16 chacun) et que la classe
`QModule_WallWidgetBase` est bien presente. L'exe a ete lie sur une capture de sources anterieure
a la passe audio du mur. **Aucun reglage de cook ne peut y changer quoi que ce soit** : il faut
recompiler la cible jeu. A savoir : le build Steam **ne sort pas de cet arbre**
(`Binaries/Win64` n'a aucun `Qanga.exe`/`Qanga.target`, pas de `Saved/StagedBuilds`, et les
fichiers de reponse UAT pointent vers `A:\QANGA\` non monte).

**LE SILENCE ETAIT LE VRAI COUPABLE.** Les 11 sites de lecture audio du plugin faisaient
`if (Load()) { Play(); }` sans `else` ni log : un asset absent du cook rendait exactement le meme
resultat qu'un reglage volontairement vide. `PlayWallSound` distingue desormais les deux et
journalise UNE fois par chemin (`WarnedMissingSounds`, membre `mutable`) : "is set but failed to
load: almost certainly missing from the cook". **Les 10 autres sites (StrikeBeaconActor.cpp
138/244/337/352/359, GadgetHUD.cpp 616/1323/1346/1968/1990) gardent le patron muet** : les durcir
est une tache dediee, hors perimetre de cette passe.

**RESTE OUVERT.** (a) La graine `DA_EasyCookSeed_QANGA` du plugin EasyCook (le mecanisme prevu
pour cet idiome : son scanner C++ matche `FSoftObjectPath(TEXT("/..."))` et force-cuit les entrees
`Source=CppLiteral`) **n'a pas ete re-scannee depuis le 2026-08-12 12:07** ; la regenerer corrige
la CLASSE du bug, mais `EasyCook.SaveSeed` **REMPLACE** la graine et detruirait les entrees
`Source=Manual` (a filtrer d'abord), et le bouton "Run" de l'EUW reecrit des `.uasset` avant de
scanner (mutation de masse, validation requise). (b) Verifier apres rebuild : le mur doit revenir
**a moitie audible** avant tout correctif de cook (4 sons sur 8 deja cuits) ; un mur totalement
muet dans un binaire qui contient le code signalerait une TROISIEME cause. (c) Le son de la roue
gadget est compile ET cuit : s'il est muet lui aussi, cet audit ne l'explique pas.

**Mesure en PIE (2026-08-13)** : build froid `Succeeded` 10.9 s ; `qmodule.Test.InsertPhaseInv`
-> `Action result on client: 14`, `RemovePhaseInv` -> `15`, insertion sur case vide -> `9`
(refus) ; ouverture du mur par `OpenWall` sans erreur ; `get_recent_engine_errors` vierge de
QModule. Verdict d'oreille RzZz le jour meme, en direct dans la session PIE : "les sons sont
bien". Reste au gout de RzZz : le rendu visuel des logos teintes ; tout (waves, volume, pitch)
se re-regle dans les DevSettings sans rebuild.

### 15.19 Les ordnances ne blessaient AUCUNE IA : la porte d'entree des degats (2026-08-18, compile vert)

**Symptome RzZz** : "les modules cyborg qui font des degats, passif ou actif, ne font aucun
degat aux IA ni aux joueurs".

**Cause, mesuree dans le code, pas supposee.** Le projet a UNE porte d'entree pour les degats :
`UGameplayStatics::ApplyDamage / ApplyPointDamage / ApplyRadialDamage`, donc `AActor::TakeDamage`,
donc l'evenement `OnTakeAnyDamage`. Deux auditeurs INDEPENDANTS y sont abonnes, et c'est tout
l'interet :

- `SS_PhysicalState.OnTakeAnyDamage_Event` -> `Lib_Life.ApplyStatDamageToActor` : la vie des
  JOUEURS, celle qui vit dans un script de stat (avec l'etape d'armure et les multiplicateurs
  Chasseur de primes / Fleau des machines poses le 2026-07-11).
- `CombatComponent` (bind `Assign On Take Any Damage` sur son owner au Begin Play) : la vie des
  IA, qui n'ont AUCUN script de stat. Leur vie est `CurrentLife` plafonnee par `LifeWhenNoStat`,
  la paire exacte que QAI lit et ecrit (`QAI_AgentComponent.cpp:322` et `:4786`). Verifie sur les
  assets : `AI_AutonomusPolice` et `BASE_Animal` portent un `CombatComponent`, zero reference a
  `SS_PhysicalState`.

Les ordnances, elles, appelaient par REFLEXION `Lib_Life.ApplyStatDamageToActor` directement
(l'ancien `ApplySplashThroughLibLife`), c'est-a-dire le DERNIER maillon de la seule branche
joueur. Or le graphe de cette fonction ne fait qu'une chose : `GetStatByActor` -> `Cast To
SS_Shield` / `Cast To SS_PhysicalState` -> `SetShield` / `TakeDamage`. Sur une IA le cast echoue,
la fonction sort, **zero degat, zero erreur, zero log**. Ce n'etait pas un reglage : c'etait
structurellement impossible.

**Pourquoi ca avait ete cru valide le 2026-07-11** : le log de controle
`Ordnance impact at X: N actor(s)` comptait les acteurs sur lesquels on avait APPELE la fonction,
pas ceux qui avaient perdu des PV. `1 actor(s)` s'affichait meme quand rien n'encaissait. Une
mesure d'appel n'est pas une mesure d'effet.

**Correctif** : `UQModule_RackComponent::Authority_ApplySplashDamage` remplace
`ApplySplashThroughLibLife` et passe par la porte d'entree, exactement comme les balles
(`QWeaponBulletSubsystem.cpp:1006`) et comme l'IA (`QAI_CombatProcessor.cpp:4186` et `:3987`) :

- coup direct = `ApplyPointDamage(cible, degat plein, ...)`, instigator = controller du porteur,
  causer = son pawn ;
- souffle = `ApplyRadialDamageWithFalloff(degat, degat x 0.1, rayon, falloff 1.0)`, ce qui
  reproduit a l'identique la decrue historique 100 % au centre -> 10 % au bord, la cible directe
  etant dans les `IgnoreActors` (elle a deja paye plein tarif) ;
- `DamagePreventionChannel = ECC_MAX` : AUCUN test de ligne de vue, le souffle traverse le
  decor comme il l'a toujours fait (decision RzZz 2026-08-18) ;
- **auto-degat conserve** : le moteur epargne toujours le `DamageCauser`, donc le porteur est
  servi explicitement avec la meme decrue. Decision RzZz : se prendre son propre souffle doit
  faire mal, c'est ce qu'un futur module d'amortissement viendra racheter.

Concerne d'un coup les 4 armements qui partagaient ce point unique : missiles d'epaule
(Nid de guepes), grenades d'epaule (Nid de frelons), frappe aerienne (Balise de frappe),
grenades collantes.

**Ce que le contournement coutait en plus du degat**, et qui revient gratuitement : `OnDamaged`,
la riposte QAI (`HandleCombatComponentDamaged`), la mort et `OnDeath`, le drop de loot, les coins
et l'XP, les points de recherche, `OnActorDeadStopAI`, les regles PvP / PvE / safe-zone du
`CombatComponent`, et les points faibles des cocons (`AQAI_AgentSpawner::TakeDamage`).

**Drone medical, meme defaut en miroir** : `Lib_Life.ApplyHealToActor` n'ecrit lui aussi que
dans un script de stat, donc le drone ne pouvait PAS soigner un compagnon IA. Ajout d'un repli
`HealThroughCombatComponent` qui ecrit `CurrentLife` borne par `LifeWhenNoStat` (le patron exact
de `UQAI_AgentComponent::TickRecruitedFollowerRegen`) puis pousse le miroir replique via
`RefreshReplicatedCombatHealth()`.

**Instrument** : le log ne compte plus les appels, il imprime ce que les victimes RENVOIENT
(`ApplyPointDamage` rend le degat reellement encaisse par `TakeDamage`) :
`Ordnance blast at X: D dmg / R cm -> direct 'A' took d1, splash landed|empty, self took d2`.
Visible avec `qmodule.Verbose 1`.

**Etat de verification.** Build : `UnrealEditor-QModule.dll` recompile et relinke le 2026-08-18
a 01:59. ATTENTION, le target `QangaEditor` NE build PAS en entier, pour une raison ETRANGERE a
ce chantier : `QSystem/Private/Component/QInteriorPostProcessComponent.cpp:614/619/623` reference
`FSceneView::InteriorDiffuseLightingIntensityScale` / `InteriorSkyLightIntensityScale` /
`InteriorIndirectSpecularIntensityScale`, qui n'existent NULLE PART dans le moteur de
`C:/UE5_Share` (grep moteur complet, zero occurrence). Source modifiee le 2026-08-17 10:27, DLL
QSystem datant du 2026-08-15 15:36 : ce fichier n'a jamais compile depuis son edition. A traiter
separement (patch moteur perdu, ou code a retirer).

RESTE A MESURER EN JEU (a faire par RzZz ou en session avec editeur) : `qmodule.Verbose 1`, puis
`qmodule.Test.AddRack` + `qmodule.Test.GodWall`, puis `qmodule.Test.ShoulderMissiles` /
`ShoulderGrenades` / `Airstrike` sur une IA, en lisant sa `CombatComponent.CurrentLife` avant et
apres. Deux points ouverts a trancher a cette occasion :
1. **Tir ami**. Le degat radial moteur n'a aucun filtre de faction, alors que les balles en ont un
   (`QWeapon` supprime le degat entre factions amies). Si le `CombatComponent` ne filtre pas
   lui-meme dans son handler, une salve tuera desormais les compagnons recrutes et l'escouade
   posee par le Protocole de meneur. A verifier AVANT de considerer le sujet clos.
2. **Type de degat**. On passe `UDamageType::StaticClass()`, par parite stricte avec les balles.
   Le projet possede pourtant une taxonomie (`DMG_Base`, `DMG_Weapon`, `DMG_WeaponHeadshot`,
   **`DMG_Missile`**, `DMG_VehicleDestroyed`, `DmgTypeBP_Environmental`) que le `CombatComponent`
   utilise pour classer les causes de mort. Passer `DMG_Missile` classerait correctement les kills
   d'ordnance, et donnerait la cle d'une future resistance au souffle. Non fait ici faute de
   pouvoir inspecter le contenu de `DMG_Missile` sans editeur.

---

### 15.21 Installer un module prend du temps, et ca se voit (2026-08-19)

**La demande.** Poser un module sur le Mur etait instantane. Desormais l'installation dure, et le
joueur voit une barre de progression pendant ce temps.

**Ce qui a ete livre.**
1. **Une duree, configurable a deux niveaux.** `UQModule_Settings::WallInstallSeconds` (3 s par
   defaut, categorie `QModule|Wall`, donc pilotable depuis `DefaultGame.ini`) et, quand un module
   merite une attente differente, `UQModule_Definition::InstallSeconds` (0 = utiliser le reglage
   projet). **`WallInstallSeconds = 0` restaure exactement l'ancien chemin instantane**, ce qui est
   aussi le garde-fou contre le piege maison `SetTimer(0)` qui EFFACE un timer au lieu de l'armer.
2. **`TryInstall` scinde en deux, sans qu'une seule regle ne change.** `QMOD_ValidateInstall` porte
   toutes les validations (aucune mutation) et rend au passage l'inventaire et l'item deja resolus ;
   `TryInstall` l'appelle puis applique (consommation de l'item, pose du socket). **Tous les
   appelants existants sont donc inchanges** : bootstrap des modules de base a la connexion, les
   quatre actions de quete du tutoriel, la console de test.
3. **Le serveur garde l'autorite et valide DEUX fois.** `SV_InstallModule` valide immediatement (un
   anneau verrouille ou une case occupee se dit tout de suite, pas au bout de trois secondes), arme
   un timer porte par le composant du PlayerState, puis a l'echeance appelle `TryInstall` qui
   revalide tout : en trois secondes le sac, l'anneau ou la case ont pu changer, et l'autorite ne
   fait pas confiance a sa propre decision passee. L'item n'est donc consomme qu'a la FIN.
4. **Une installation a la fois par rack.** Une seconde demande est refusee par
   `EQModule_ActionResult::InstallBusy`, **ajoutee en fin d'enum** pour qu'aucune valeur existante ne
   change de numero (meme regle qu'au paragraphe 15.16), avec sa phrase localisee dans
   `QMOD_DescribeActionResult`. Deux timers simultanes voudraient dire deux plaques a l'ecran, et
   aucune des deux ne decrirait ce que le Mur fait vraiment.
5. **La plaque.** Nouveau RPC client `CL_InstallStarted(ModuleTag, Seconds)` : cote client
   uniquement, il ouvre la barre de progression partagee du jeu via
   `UQNotificationManager::ShowProgress` avec le nom localise du module, **l'icone de sa definition**
   et l'auto-progression reglee sur la duree. La barre se remplit donc toute seule : **aucun RPC de
   progression, aucun tick**. Un message a l'aller, un message (`CL_ActionResult`) au retour.

**Dependances ajoutees** : `QNotification` en `PrivateDependencyModuleNames` du `QModule.Build.cs` et
dans les plugins de `QModule.uplugin`. QNotification est purement client (rien de replique), donc la
dependance ne remonte jamais cote serveur dedie.

**Le widget de la plaque a ete refait dans la foulee** : voir la copie
`/Game/Widget/Notifications/V2/W_QProgressNotification_V2`, sur laquelle pointe desormais
`ProgressNotificationWidgetClass` du gestionnaire de notifications. **Attention, ce widget est
PARTAGE** : c'est la barre de progression de tout QANGA (recolte, quetes DQS), pas une piece du Mur.

## 16. Contrat audio central (2026-08-18)

**Pourquoi.** Les 13 refs son du plugin partaient en `PlaySound2D` / `PlaySoundAtLocation`
depuis 11 sites disperses, **sans attenuation, sans concurrency et sans `OwningActor`**, avec
un `LoadSynchronous` a chaque lecture (jusqu'a 9 fois par seconde pour le bip de balise). Un
`LimitToOwner` ne peut rien limiter sans owner, et un son sans attenuation ne se localise pas.

**Ce qui est livre.** `UQModule_AudioLibrary` (`Public/QModule_AudioLibrary.h`), portage direct
du patron deja eprouve de `UQWeaponAudioLibrary` (chaine NashV2) :

- `ShouldPlayLocalModuleAudio` : remonte Owner / Instigator / parent d'attache (16 sauts, set de
  visites). Depuis le rack, qui vit sur le PlayerState, la marche atteint le Controller local.
  C'est ce qui separe la couche LOCALE (non spatialisee, stable a la camera) de la couche WORLD.
- `IsWorthHearing` : rayon audio PROPRE, volontairement decouple de `OrdnanceVisualRangeM`
  (800 m). Ce dernier repond "faut-il spawner ce projectile", une question de budget de rendu.
- `PlayWorldOneShot` / `PlayLocalOneShot` : passent par `AudioDevice->PlaySoundAtLocation`,
  **aucun `UAudioComponent` cree** pour un one-shot.
- `SpawnAttachedLoop` / `StopOrFadeLoop` / `SpawnTail` : le composant n'existe que pour une
  boucle, un son attache ou une tail. `bTailSurvivesOwner` porte la difference de fond entre une
  tail de depart (suit le lanceur, survit a l'obus) et une tail d'explosion (reste au point).
- `ResolveSound` : cache de resolution (plus de `LoadSynchronous` par bip) ET generalisation du
  diagnostic de `PlayWallSound` : une ref vide = silence voulu, une ref renseignee qui ne charge
  pas = bug de cook, journalise UNE fois avec son chemin. Voir le BUG 1 de la section audio du mur.

`FQModule_AudioEvent` (dans `QModule_Types.h`) decrit un evenement audible : sons local/world/tail,
attenuation, concurrency, volumes, pitch, politique d'attache, survie de la tail.
Dependance ajoutee : `AudioExtensions` (prive), comme QWeapon.

**Tranche verticale : le depart des missiles d'epaule.**
`Client_PlayShoulderMissileLaunchAudio` est appele en tete de `MC_ShoulderMissileVisual`,
**avant** le gate `IsWorthRendering` : sinon toute salve au-dela de 800 m serait muette pour une
raison de rendu. Le tireur recoit la couche locale, les autres la couche world attenuee, et la
tail est accrochee au pawn lanceur.

**CE QUI N'A PAS ETE FAIT, ET POURQUOI.** Le vol et l'impact ne sont PAS traites ici.
`BP_Missile` (`/Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/`, partage par le
lance-roquettes, les tourelles, les vehicules et les deux chemins d'ordnance QModule) porte deja
un `AudioComponent`, un `RichochetSound`, un `PlaySoundAtLocation` et des refs vers
`QATT_Big_Weapon`, `QSC_Bullet` et le pack Gamemaster. **Y ajouter un impact en C++ le
doublerait.** L'inventaire exact de son graphe demande l'editeur ouvert (pont CLIScape).

**Sources cablees** (existantes, remplacables sans toucher au code) : body =
`explosion_large_no_tail_03` (prise seche, pas de reverb imprimee), tail = `Tail-GL_V2` (tail de
lance-grenades Nash, meme production que le set de reference du mix), attenuation =
`QATT_Big_Weapon`, celle que `BP_Missile` utilise deja, pour que depart et impact du meme missile
parlent la meme piece.

**RESTE OUVERT.** (a) `QSC_Ordnance_Launch` (1 voix par owner, StopOldest) n'existe pas : le champ
`Concurrency` est laisse NUL, c'est une soft ref de config donc l'asset s'active sans recompiler.
(b) Les 10 sites audio historiques gardent encore le patron muet et ne sont pas migres vers la
bibliotheque. (c) Rien n'a ete ecoute en jeu : cette passe est compilee, pas validee a l'oreille.

### 16.1 La balise : de ~21 voix a UNE (2026-08-18)

**Le defaut.** `AQModule_StrikeBeaconActor` appelait `PlaySoundAtLocation` a chaque bip, a une
cadence qui monte avec le compte a rebours : `BeepFrequency = Lerp(1.5, 9.0, Urgency)`. Sur une
onde de plusieurs secondes, une SEULE balise pouvait donc tenir une vingtaine de copies d'elle-meme
vivantes en meme temps. Illisible a l'oreille, et paye sur l'audio thread. Aucune attenuation,
aucune concurrency, et un `LoadSynchronous` a chaque bip.

**Le fix.** `Client_PulseUplinkBeep` maintient UN `UAudioComponent` (`BeepAudio`) attache a la
balise et le REDEMARRE a chaque pulsation (`Stop()` puis `Play()`). La limite devient
**structurelle** : elle ne depend plus d'un asset de concurrency qui nettoierait apres coup. A 9 Hz
on n'entend que le transient de tete, ce qui est exactement ce que doit etre un tic de compte a
rebours. `Client_StopUplinkBeep` coupe la voix quand le compte a rebours se termine ET dans
`EndPlay`.

**Les sons monde de la designation ont enfin une place dans l'espace.** Les cinq (`BeaconThrowSound`,
`BeaconPlantSound`, `BeaconBeepSound`, `StrikeFireSound`, `StrikeIncomingSound`) jouaient **sans
aucune attenuation** : une balise plantee etait donc entendue a plein volume par tous les joueurs de
la planete, sans direction et sans occlusion. Nouveau reglage `DesignationWorldAttenuation`, par
defaut `QATT_GameplayElement` (le profil du projet pour un objet physique non offensif, ce qu'est
une balise). `DesignationWorldConcurrency` reste nul, en attente de son asset.

**`StrikeIncomingSound` est desormais ATTACHE** a la balise au lieu d'etre pose a un point du monde :
le grondement appartient a la balise, donc il la suit et meurt avec elle.

### 16.2 Migration des sites muets

Le point "RESTE OUVERT" de la section audio du mur est traite. Les 5 sites du `GadgetHUD`
(`GadgetSwitchSound`, `TargetingArmSound` x2, `TargetingCancelSound` x2) et les 5 sites de la
balise passent maintenant par `UQModule_AudioLibrary` (`PlayLocalSoundRef` / `PlayWorldSoundRef` /
`SpawnAttachedSoundRef`). Ils heritent donc des gardes serveur dedie, du cache de resolution et
surtout du **diagnostic de cook** : un asset configure absent du build le DIT une fois, au lieu de
jouer le silence.

`PlayWallSound` (`QModule_WallWidgetBase`) n'a PAS ete migre : il porte deja son propre
`WarnedMissingSounds` et fonctionne. Le migrer serait du churn sans gain.

### 16.3 Ce que l'ouverture de BP_Missile a revele (2026-08-18, mesure editeur)

`BP_Missile` (`/Game/Marketplace/BallisticsVFX/FXSpawnerBlueprints/Projectiles/`) est bien le
projectile PARTAGE : **6 referents** (tourelle, lance-roquettes vehicule, les deux lance-roquettes
NashV2, `RocketLauncherMissileProjectile`, plus la graine EasyCook), en plus des deux chemins
d'ordnance QModule qui le spawnent via `AirstrikeMissileClass`.

**Il avait deja plus d'audio que prevu, et moins de qualite que prevu :**

- `SFX_RocketLoop` (AudioComponent) : **la boucle de vol EXISTE**. Lancee par `Play` au
  `Event BeginPlay` (Sequence Then 1), coupee par `Stop` dans `HitEvent`. Ne pas la doubler.
- `RichochetSound` (AudioComponent) : passe au `Cubit_ImpactFX_Spawner` comme composant de ricochet.
- L'impact est **un unique `Play Sound at Location`** dans `HitEvent`, volume 1.0 et pitch 1.0
  figes, sur **une seule onde** : `explosion_large_08`. Aucune variante, aucun Close/Med/Far,
  aucune couche sub, aucune tail. Sur une salve de 64 missiles c'est 64 fois le meme fichier.
- Le `UserConstructionScript` fait `Branch(Is Dedicated Server) -> Destroy Actor` : le projectile
  se suicide sur serveur dedie, donc aucune voix serveur. Bon point deja en place.

**LE VRAI DEFAUT, mesure et corrige.** `explosion_large_08` avait
`sound_class_object = None`, `attenuation_settings = None`, `concurrency_set = {}`. Comme le noeud
BP ne passe aucune surcharge, **l'explosion etait entendue a plein volume par tous les joueurs de
la planete**, sans distance, sans direction et sans occlusion. Et cette onde n'est pas au missile :
elle est partagee par `BP_Missile`, `BP_Bomb` et `BP_GrenadeProjectile`. Corriger l'asset repare
donc les trois systemes d'un coup : classe `QSClass_Weapon`, attenuation `QATT_Big_Weapon`,
budget `QSC_Ordnance_Impact`.

**Le bip de balise etait sur `QSClass_UI`.** Un objet physique du monde mixe comme de l'interface,
donc hors du ducking monde. Zero autre referent que la graine de cook, donc reroutage sans risque
vers `QSClass_GameElement` + `QATT_GameplayElement`. Duree confirmee : **2.3216 s**.

**Profils crees** : `QSC_Ordnance_Launch` (MaxCount 1, LimitToOwner, StopOldest) et
`QSC_Ordnance_Impact` (MaxCount 16, global, StopFarthestThenOldest, pour que l'explosion la plus
PROCHE survive a une salve de 64). Force-cook ajoute sur `/Game/Sounds/_SoundClass` et
`/Game/Sounds/_Attenuation` : `QSC_Ordnance_Launch` n'est atteint que par un litteral C++.

**RESTE.** L'impact n'a toujours qu'une prise. Le remede est un SoundCue de variantes (random sans
remplacement + modulation de pitch) branche sur le pin Sound du `Play Sound at Location` de
`BP_Missile`. **CORRIGE le 2026-08-18 : `manage_sound_cue` action `connect_nodes` ne crashe plus.**
Cause reelle, mesuree dans la source moteur : l'outil traitait `from_node` comme le PARENT, or
un `wave_player` est une feuille dont `GetMaxChildNodes()` vaut 0 (`SoundNodeWavePlayer.cpp:306`).
`USoundNode::InsertChildNode` refuse alors SANS RIEN DIRE (`SoundNode.cpp:305`), et la ligne 251
ecrivait dans un slot inexistant. Second defaut, plus grave : les noeuds etaient crees par
`NewObject` brut, donc sans `USoundCueGraphNode`, que le moteur passe a `CastChecked` des qu'une
insertion reussit, donc meme l'appel CORRECT crashait. Le fix passe par `ConstructSoundNode` +
`LinkGraphNodesFromSoundNodes`, et `from_node` designe desormais la SOURCE, `to_node` la
DESTINATION (sens du signal). **Actif seulement apres recompilation de CLIScape.**

### 16.4 Assets livres et LE seul geste manuel restant

Crees et verifies par relecture le 2026-08-18 :

| Asset | Reglage | Role |
|---|---|---|
| `/Game/Sounds/_SoundClass/Concurrency/QSC_Ordnance_Launch` | MaxCount 1, LimitToOwner, StopOldest | depart d'ordnance : une salve etagee se lit COMME une salve parce que chaque depart releve le precedent |
| `/Game/Sounds/_SoundClass/Concurrency/QSC_Ordnance_Impact` | MaxCount 16, global, StopFarthestThenOldest | impacts : sur une salve de 64, l'explosion la PLUS PROCHE du joueur survit |
| `/Game/Sounds/_Ordnance/Cue_Ordnance_Impact` | Modulator (pitch 0.92-1.08, vol 0.9-1.0) -> Random SANS remplacement -> 3 prises | supprime la repetition machine du fichier unique |

Les 3 prises du cue : `explosion_large_08`, `explosion_med_long_tail_01`,
`explosion_large_no_tail_03`. Cue route `QSClass_Weapon` + `QATT_Big_Weapon` + `QSC_Ordnance_Impact`.

**GESTE MANUEL RESTANT (15 secondes, dans l'editeur).** Ouvrir `BP_Missile`, EventGraph, evenement
`HitEvent`, branche `Sequence -> Then 1`, noeud **`Play Sound at Location`** : remplacer la valeur
du pin **Sound** (actuellement l'onde `explosion_large_08`) par
**`/Game/Sounds/_Ordnance/Cue_Ordnance_Impact`**. Compiler et sauver.

Pourquoi ce n'est pas automatise : l'API Python n'expose pas les graphes K2 (`ubergraph_pages` /
`function_graphs` n'existent pas cote Python), donc il n'y a aucun chemin scripte SUR
pour ce pin. Ecraser un graphe partage par 6 systemes au jugE n'en valait pas le risque.
Sauvegarde prealable du BP : `Saved/AudioPass_Backups/BP_Missile_pre-audio-2026-08-18.uasset`.

Tant que ce pin n'est pas change, l'impact reste l'onde unique, MAIS elle est desormais
correctement attenuee, classee et budgetee : le gros du defaut est deja corrige.

### 16.5 Grenade collante, largage, drone medical (2026-08-18)

**Etat de depart : les trois acteurs etaient TOTALEMENT muets.** Zero occurrence de `Sound` ou
`Audio` dans `QModule_StickyGrenadeActor`, `QModule_SupplyCrateActor` et
`QModule_MedicalDroneActor`. Leurs cosmetiques client etaient deja proprement separes de
l'autorite, donc les points d'accroche existaient : il n'y avait qu'a les brancher.

**Grenade collante.**
- `OnPlantedCosmetic` : clonk de collage, profil court (`QATT_Reload_weapon`, ~46 m).
- `Tick` : **l'oreille ne suit PAS l'oeil.** La lumiere clignote a 14 Hz ; un son cale sur cette
  cadence empilerait exactement comme le bip de balise. La pulsation a donc son PROPRE compteur
  (`StickyGrenadeArmPulseHz`, 2.5 Hz par defaut) et UNE voix unique qu'elle redemarre.
- `MC_Detonate` : coupe la pulsation AVANT la detonation, joue `Cue_Ordnance_Impact` (variantes
  + modulation, donc une salve ne repete pas le meme fichier), puis une tail **non attachee**.
  La grenade est detruite 0.6 s plus tard : l'echo d'une explosion appartient au LIEU, jamais a
  l'objet qui l'a causee. `EndPlay` ajoute pour couper la pulsation dans tous les cas.

**Largage.**
- `StartFallCosmetics` : boucle de descente attachee a `CrateMesh`, profil longue portee. Une
  caisse qui tombe d'un ciel a 150 m est un signal de ralliement : il faut l'ENTENDRE arriver.
- `PlayLandingCosmetics` : coupe la descente (fade 0.12 s), impact lourd + tail au point de
  chute, non attachee. `EndPlay` coupe la boucle.

**Drone medical.**
- `BeginPlay`, branche `FApp::CanEverRender()` : blip de deploiement + boucle de hover attachee a
  `DroneMesh`, spawnee **une seule fois**, jamais dans `Tick`. Le "MaxCount 1 par objet" est
  structurel (un composant par drone), pas delegue a un asset de budget.
- `OnRep_PulseCounter` : le blip de soin est pilote par le compteur REPLIQUE, donc une fois par
  vraie pulsation sur chaque client, sans scrutation ni recalcul local.
- `EndPlay` : fade du hover (0.25 s) puis blip d'extinction laisse au point, pas attache a un
  acteur en train de disparaitre. Profil `QATT_GameplayElement` : le drone est cull a 400 m et
  vit sur l'epaule du joueur, ce n'est pas de l'ordnance.

**Sources** : toutes dans SF_Meca (`Mechanism_Clonk_Small`/`Big`, `Interface_Bips_3_1`,
`Elevator_Big_Loop`, `Engine_Small_Start`/`Loop`/`End`, `Scanner_Validated_2`), plus
`Tail-Outdoor` et `Cue_Ordnance_Impact`. Tous ces dossiers sont deja couverts par les lignes
`DirectoriesToAlwaysCook` posees plus haut : aucune nouvelle ligne necessaire.

**COMPILE ET VERIFIE** le 2026-08-18 a 22:53 (`Result: Succeeded`, cible a jour, aucun source
plus recent que le DLL). Ecrit d abord sans build parce qu une mise a jour moteur etait en cours
de reception ; les verifications statiques faites a la place ont d ailleurs attrape un vrai defaut,
`UAudioComponent` manquait en declaration avant dans DEUX des trois headers, ce qui aurait casse
le build. **La mise a jour moteur a aussi repare `QSystem/QInteriorPostProcessComponent`** : les
champs d eclairage interieur de `FSceneView` existent a nouveau, donc le probleme de volume light
venait bien de la et non d un bug isole (voir 16.3).

### 16.6 Dropship, et pourquoi le bouclier n'a PAS ete traite (2026-08-18)

**LE BOUCLIER N'EXISTE PAS COMME OBJET.** Le brief audio prevoyait "activation / loop / impacts
rate-limites / break / recharge / extinction". Recherche faite dans tout le plugin : le seul
"Shield" est `QModule_LegacyFacade::ApplyShieldOverlay`, un **overlay de STAT** qui ecrit
`MaxShield` / `CurrentShield` sur le StatsComponent du pawn via reflexion. Le commentaire du code
dit lui-meme que `SS_Shield` "exists complete in the game but is NEVER granted". Il n'y a donc
aucun acteur, aucun composant et aucun evenement de break/recharge ou accrocher quoi que ce soit.
**Rien n'a ete ecrit pour le bouclier** : le faire aurait voulu dire inventer un systeme, pas
sonoriser un existant. A reprendre le jour ou le bouclier devient un vrai effet attache.

**Dropship : uniquement des accents, et JAMAIS une boucle.** Le Lavrik est un vrai Blueprint de
vehicule, spawne par `SpawnShip` puis demarre par `StartShipEngine` qui appelle `SetVehicleState`
(valeur 1 = On) par reflexion sur l'acteur ou l'un de ses composants. **Le moteur appartient donc
au vehicule** et vit sur `QSClass_Vehicle`. La regle de RzZz ("ne jamais doubler le moteur du
vehicule") est tenue de maniere STRUCTURELLE : les trois evenements ajoutes sont des one-shots,
jamais un bed. Il n'y a volontairement **pas d'evenement "hover"**, parce qu'un hover bed EST la
boucle moteur.

- `SpawnShip` : accent d'arrivee (`DropshipApproachAudio`), profil `QATT_Ship`.
- `OnRep_Phase` : c'est LE point d'accroche, parce que la phase est **repliquee**. Chaque client
  reagit a la meme transition d'etat au lieu de scruter le vaisseau ou de recalculer le moment
  localement. Passage a `Hovering` -> rampe ; passage a `Outbound` -> depart. Un joueur qui
  arrive alors que la phase est deja `Outbound` ne voit pas la transition et n'entend donc pas le
  depart : c'est le comportement voulu, il l'a rate.

**`DropshipRampAudio` est laisse VIDE volontairement.** `ApplyDoorState` pilote la logique de porte
**du Lavrik lui-meme** (appel par nom de fonction). Si ce Blueprint voix deja sa rampe, remplir cet
evenement poserait deux hydrauliques sur une seule porte. A remplir seulement APRES avoir ecoute la
rampe s'ouvrir en PIE avec l'evenement encore muet.

**RESTE A FAIRE (routage).** Les waves SF_Meca utilisees par les lots 5 et 6 sont encore sans
SoundClass (l'etat de ~89 % du projet). Elles devraient aller sur `QSClass_GameElement` pour la
grenade / le largage / le drone, et sur `QSClass_Vehicle` pour les deux accents de dropship. Non
fait ici : une mise a jour moteur etait en cours et l'editeur ne devait pas etre sollicite. Verifier
les referents de chaque wave avant de router (plusieurs sont partagees).

**VERIFICATION FAITE A DEFAUT DE BUILD.** Extraction automatique des 72 chemins `/Game/...` du
constructeur de `QModule_Settings.cpp` et controle sur disque : **70 resolvent**, les 2 restants
sont des faux positifs pre-existants et sans rapport (`IDA_QModulePhase_T%d`, un gabarit resolu au
runtime, et `/Game/Phases`, un dossier de scan). Tous les chemins audio des lots 1, 2, 5 et 6
resolvent donc reellement sur disque.

### 16.7 Passe de routage et de mix, terminee (2026-08-18 soir)

**CORRECTION IMPORTANTE de la section 16.3.** Il y etait ecrit que l'impact du missile etait
"entendu a plein volume par toute la planete". **C'est FAUX pour `BP_Missile`.** La lecture du
noeud `Play Sound at Location` (`K2Node_CallFunction_31`) montre que son pin
`AttenuationSettings` portait `/Game/Systems/Sound/Attenuation/AI_Attenuation` : **un override de
noeud gagne toujours sur le reglage de l'asset**. L'impact etait donc bien attenue, mais par un
profil concu pour les voix d'IA, applique a une explosion.

La lecon generale : sur un `Play Sound at Location`, l'attenuation ET la concurrency peuvent etre
surchargees au noeud. Regler l'onde ne suffit PAS pour ces deux-la, alors que la **SoundClass**,
elle, vient toujours de l'asset. Verifier les pins du noeud avant de conclure.

Ce qui restait vrai du diagnostic : la SoundClass manquante (corrigee) et le budget de voix absent
(le pin `ConcurrencySettings` est vide, donc `QSC_Ordnance_Impact` pose sur l'onde s'applique bien).

**`BP_Missile` corrige** (sauvegarde prealable dans `Saved/AudioPass_Backups/`) :
- pin `Sound` : `explosion_large_08` -> `/Game/Sounds/_Ordnance/Cue_Ordnance_Impact` (3 prises en
  random sans remplacement + modulation de pitch, donc plus de repetition machine sur une salve) ;
- pin `AttenuationSettings` : `AI_Attenuation` -> `QATT_Big_Weapon`, le profil que ce meme
  Blueprint utilise deja pour sa boucle de vol.
Les deux relus apres compilation et sauvegarde. Note d'outillage : `get_asset_dependencies` a
repondu que le cue n'etait pas reference JUSTE APRES la sauvegarde (registre pas encore rescanne).
**Le noeud fait foi, pas le registre**, sur une verification immediate.

**Routage du mix : 22 sons du contrat, 0 probleme restant.** Chaque onde a une SoundClass et plus
aucun son du monde ne reste sur le bus interface. Grenade et missile en `QSClass_Weapon` ; balise,
largage et drone en `QSClass_GameElement` ; accents dropship en `QSClass_Vehicle` ; seuls
`DeviceA_Open` et `DeviceA_Close1` restent en `QSClass_UI`, ce qui est correct, ce sont de vrais
sons 2D.

**TROIS ondes du monde etaient sur `QSClass_UI`**, pas une : le bip de balise (`Bips_7`), la
pulsation de grenade (`Bips_3_1`) et le lancer de balise (`Swipe_5`). Toutes jouees en position
dans le monde, toutes mixees comme de l'interface. C'est un travers recurrent du projet, a
verifier systematiquement quand on prend une source dans le sous-dossier `SF_Meca/UI/`.

**Piege evite.** Le blip de soin du drone visait `Scanner_Validated_2`, aussi referencee par
`IGBR_Repport_Widget`. Lui donner une classe monde aurait deroute un blip d'interface. Bascule sur
`Scanner_Validated_3`, qui n'a aucun referent. **Ce changement d'une ligne dans
`QModule_Settings.cpp` n'est PAS encore compile** (editeur ouvert au moment du changement).

### 15.20 Le verrou d'ordnance restait colle aux cadavres : `IsValid` n'est pas "vivant" (2026-08-19, compile vert)

**Symptome (RzZz)** : avec le Nid de guepes ou le Nid de frelons arme, le marqueur de cible reste
affiche "de temps en temps" et continue de suivre des IA deja tuees.

**Cause, en deux temps.** Le paragraphe 15.14 dit qu'un verrou est tenu "jusqu'a ce que la cible
meure", mais le test ecrit etait `IsValid(Target)`, qui ne dit pas "vivant", il dit "l'acteur n'a
pas ete detruit". Un cadavre reste un acteur valide pendant tout son ragdoll. Le verrou survivait
donc a la mort. Et surtout, deux systemes se battaient :

1. a la mort, QAI detruit les temp trackers attaches a l'agent
   (`QAI_DestroyTempTrackersForDeadAgent`, appele par `HandleDeath`, qui tourne aussi sur le client
   via `OnRep_IsAlive`). Le marqueur disparaissait, correctement ;
2. 0,12 s plus tard, la boucle de verrou voyait `Lock.Marker` a null, `IsValid(Target)` toujours
   vrai, et **respawnait** le marqueur sur le cadavre, par la branche "the tracker framework dropped
   the marker under us". Avec `GadgetLockMarkerLifetimeSeconds` a 600 s, le marqueur repose ensuite
   dix minutes sur le corps.

Le "de temps en temps" du rapport, c'est la difference entre une IA reellement detruite (le verrou
tombait) et un corps qui reste au sol a portee (le marqueur revenait en boucle).

**Correctif.** Un seul concept, quatre sites, aucun changement de contrat reseau :
- `QModuleGadgetHUD::IsLockTargetAlive` (namespace anonyme de `QModule_GadgetHUD.cpp`) delegue a
  `QAI_BehaviorHelpers::IsActorCombatAlive`. QModule dependait deja de QAI (dependance privee,
  ajoutee pour QModuleLoot), donc rien a inventer : le helper teste `UQAI_AgentComponent::IsAlive`,
  qui est **replique sans condition** (`DOREPLIFETIME`), puis retombe par reflexion sur le
  `CombatComponent` BP. Il lit donc la meme chose sur un client que sur le serveur ;
- retention (`UpdateOrdnanceLocks`) : `bAlive` passe de `IsValid` au test de vie ;
- acquisition (meme fonction) : le test est place **entre le cone et la trace de visibilite**, moins
  cher qu'une trace. Sans lui, un cadavre dans le cone serait re-verrouille a la passe suivante,
  juste apres que la retention l'a lache ;
- `BuildLockTargetList` : une cible morte entre la derniere passe et la detente ne part plus au
  serveur ;
- `SV_TriggerShoulderMissiles` : la revalidation serveur pretendait deja verifier "alive" alors
  qu'elle ne testait que `IsValid`. Une salve pouvait donc encore se guider sur un corps.

**Regle a retenir** : sur QANGA, "mort" ne se lit jamais avec `IsValid`. Deux modeles de vie
coexistent (voir 15.19), et le seul qui traverse le reseau pour une IA est `IsAlive` de
`UQAI_AgentComponent`.

**Reste a valider** : la conduite en jeu (armer le Nid de guepes, tuer une cible verrouillee,
verifier que le marqueur part tout de suite et ne revient pas). Le lock loop part du pawn du joueur,
donc un PIE non supervise ne l'exerce pas (voir 15.11, "unattended PIE has no player pawn").

### 16.8 Warnings de cook : 96 kHz et Bink Audio (2026-08-18)

**Symptome.** 4 warnings au cook, qui ne sont en fait que **2 ondes** journalisees deux fois
(une par le CookWorker, une remontee par `LogInit`) :
`explosion_far_distant_02` et `explosion_med_long_tail_01`, "High sample rate wave (96000) with
Bink Audio - perf waste".

**Cause, lue dans le moteur.** `AudioDerivedData.cpp:1896` :
`if (WaveSampleRate > 48000 && Inputs.BaseFormat == NAME_BINKA)`. Le cook ne redescendait pas les
ondes parce que `bResampleForDevice=False` dans
`[/Script/WindowsTargetPlatform.WindowsTargetSettings]` (`DefaultEngine.ini`), alors que
`MaxSampleRate=48000` y etait deja renseigne. Bink jetait donc tout au-dessus de 48 kHz, mais on
payait quand meme le stockage et la decompression.

**Pourquoi CES deux ondes.** Les 4 waves du pack sont en 96 kHz / 24-bit. Les deux qui ont averti
sont celles que cette passe a fait ENTRER dans le cook : `med_long_tail_01` via
`Cue_Ordnance_Impact`, et `far_distant_02` via la ligne `DirectoriesToAlwaysCook` posee sur tout le
dossier du pack. Les deux autres avaient deja leur DDC construit.

**FIX : `bResampleForDevice=True`.** Une ligne. `USoundWave::GetSampleRateForCompressionOverrides`
(`SoundWave.cpp:4401`) rend `FMath::Min(MaxSampleRate, SampleRate de l onde)`, et `GetResampleRate`
ne rééchantillonne que si la valeur DIFFERE de la source. Donc **jamais de sur-echantillonnage** :
une onde 44.1 kHz reste a 44.1, une 48 reste a 48, seules celles au-dessus redescendent a 48. Les
assets sources ne sont pas touches, seule la sortie cuite change. C est aussi ce qui respecte la
regle de RzZz "ne resample pas un bon original juste pour cocher une case" : on ne resample pas
l original, on arrete juste d expedier des frequences que le codec jette.

**Portee mesuree.** Scan des sources du projet : **108 fichiers sont au-dessus de 48 kHz** (99 en
96 kHz, 9 en 192 kHz) sur 252 au total. Les 2 warnings n etaient donc que la partie emergee ; le
correctif couvre les 108 et fera baisser la taille de l audio cuit.

**Effets de bord a connaitre.** Le drapeau entre dans la cle du DDC audio
(`AudioCompressionSettings.cpp:47`, `AppendHash("R4DV", bResampleForDevice)`) : **le prochain cook
reconstruit toute l audio derivee et sera donc plus long**, une seule fois.

**Linux non touche** : la section `[/Script/LinuxTargetPlatform.LinuxTargetSettings]` porte le
commentaire "Linux is a dedicated-server target only". Un serveur dedie ne cree aucune voix (c est
la premiere garde de `UQModule_AudioLibrary`), la question du rééchantillonnage ne s y pose pas.

**Deuxieme volet du meme jour : le verrou peignait aussi les PNJ pacifiques.** Capture RzZz a
l'appui, un androide civil de station portait le marqueur. L'acquisition ne filtrait que "pawn non
joueur, hors de mon escouade". Elle passe maintenant par la matrice de QAI
(`QAI_FactionLib::AreFactionsHostile`, la source unique d'hostilite du projet), avec la faction du
joueur lue **une fois par passe** et non par candidat (la lecture parcourt les composants de
l'acteur et construit une FString par composant, donc elle ne doit pas tourner sur chaque pawn du
rayon).

Perimetre choisi par RzZz : **tout sauf les civils (None) et la Dissidence**. Deux factions que la
matrice declare amies restent volontairement verrouillables :
- **IcLabs** (police, gardes) : leur hostilite depend du niveau de recherche, qui vit dans le
  subsystem QPolice cote serveur et n'est pas lisible depuis le HUD client ;
- **Animal** : l'agressivite predateur/proie se decide par comportement, pas par faction.

Reglage de repli `UQModule_Settings::bLockEnemiesOnly` (`QModule|GadgetHUD`, defaut `true`) : le
mettre a `false` rend de nouveau tout pawn non joueur verrouillable, sans rebuild.

**Non touche, volontairement** : le tir sans verrou. `FireSelectedDirectGadget` retombe sur l'acteur
vise par la trace quand aucun verrou n'est pris, et la revalidation serveur ne filtre pas la faction.
Le joueur peut donc toujours tirer une salve sur un civil s'il vise, avec les consequences QPolice
qui vont avec : c'est le HUD qui devait arreter de designer tout seul, pas le tir qui devait devenir
impossible.

**Si un PNJ reste verrouillable apres ce filtre**, ce n'est plus le HUD : c'est la faction de son
Blueprint. Elle s'authore en override de composant herite (ICH) sur le `CombatComponent`, et se lit
en spawnant l'acteur en editeur (voir la note ICH dans `Documentation/QAI_ARCHITECTURE.md`).
