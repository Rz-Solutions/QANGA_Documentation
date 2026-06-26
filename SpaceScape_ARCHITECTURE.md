# SpaceScape — Architecture

> Plugin **content-only** (auteur : iolaCorp Studio) : **aucun module C++**, aucun `Source/`. Il fournit le **ciel étoilé / fond galactique** de QANGA sous forme d'assets de contenu — un mesh de skybox, un pipeline de matériaux (Voie lactée + nuages de Magellan + lune), leurs textures, un Blueprint placeur et une map d'exemple. Ce document décrit l'**organisation du contenu** ; les paramètres précis des matériaux se lisent en ouvrant les assets dans l'éditeur.

> ⚠️ **Particularité.** `SpaceScape` ne contient que du contenu (`"CanContainContent": true`, pas de `Modules`). Il n'y a donc ni classe, ni réplication, ni cycle de vie C++ à documenter — uniquement une **carte d'assets** et le pipeline de rendu qu'ils composent.

---

## 1. But d'usage

QANGA se joue sur des planètes et dans l'espace : il faut un **fond de ciel** crédible et performant (étoiles, Voie lactée, nuages de Magellan, lune) qui s'affiche autour du joueur sans coût de simulation. Plutôt qu'une vraie galaxie en géométrie, `SpaceScape` rend ce fond avec une **skybox texturée** (un grand mesh autour de la caméra) dont le matériau compose plusieurs couches astronomiques. C'est un asset de **décor**, posé dans un niveau via un Blueprint dédié.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Rendu par skybox, pas par géométrie.** Un mesh statique (`SM_Starfield`) entoure la scène ; son matériau (`M_Starfield`) dessine le ciel. C'est l'approche standard « unlit sky dome » — coût de rendu minimal, indépendant de la position du joueur.
- **Couches astronomiques en textures.** Les textures séparent les composantes du ciel : `T_Starfield_MilkyWay` (Voie lactée), `T_Starfield_LMC` / `T_Starfield_SMC` (Grand / Petit Nuage de Magellan), `T_Starfield_Noise` (bruit, scintillement/étoiles), `T_Starfield_Flow` (carte de flux pour animer/distordre). Le matériau les combine.
- **Material functions réutilisables.** `MF_Starfield_Sampler` (échantillonnage des couches du ciel) et `MF_Starfield_ExtractColour` (extraction/composition de couleur) factorisent la logique de matériau, partagée entre les variantes.
- **Variantes de matériau.** `M_Starfield` (ciel de base) et `M_Starfield_GalaxyTex` (variante pilotée par texture de galaxie) ; `M_Starfield_Moon` (rendu de la lune avec `T_Moon_BaseColour` + `T_Moon_Normal`). Les **instances** `MI_Starfield` / `MI_Starfield_GalaxyTex` exposent les paramètres réglables sans recompiler le master.
- **Placement par Blueprint.** `BP_Starfield` assemble le mesh + le(s) matériau(x) en un acteur posable ; `StarfieldExampleMap` montre l'usage de référence.
- **Pas de logique runtime.** Aucun code : l'animation éventuelle (scintillement via `T_Starfield_Flow`) est faite **dans le matériau** (WPO/panner/temps), pas en C++/BP.

---

## 3. Carte des assets

| Asset | Type | Rôle |
|---|---|---|
| `Content/Blueprints/BP_Starfield` | Blueprint (acteur) | Acteur placeur : mesh de skybox + matériau(x) du ciel |
| `Content/Maps/StarfieldExampleMap` | Level | Map d'exemple / usage de référence |
| `Content/Objects/SM_Starfield` | Static Mesh | Dôme/skybox entourant la scène |
| `Content/Materials/M_Starfield` | Material | Matériau de ciel de base (composition des couches) |
| `Content/Materials/M_Starfield_GalaxyTex` | Material | Variante pilotée par texture de galaxie |
| `Content/Materials/M_Starfield_Moon` | Material | Rendu de la lune (base color + normal) |
| `Content/Materials/MF_Starfield_Sampler` | Material Function | Échantillonnage des couches du ciel |
| `Content/Materials/MF_Starfield_ExtractColour` | Material Function | Extraction / composition de couleur |
| `Content/Materials/MI_Starfield` | Material Instance | Paramètres réglables du ciel de base |
| `Content/Materials/MI_Starfield_GalaxyTex` | Material Instance | Paramètres réglables de la variante galaxie |
| `Content/Textures/T_Starfield_MilkyWay` | Texture | Voie lactée |
| `Content/Textures/T_Starfield_LMC` / `T_Starfield_SMC` | Texture | Grand / Petit Nuage de Magellan |
| `Content/Textures/T_Starfield_Noise` | Texture | Bruit (étoiles / scintillement) |
| `Content/Textures/T_Starfield_Flow` | Texture | Carte de flux (animation/distorsion) |
| `Content/Textures/T_Moon_BaseColour` / `T_Moon_Normal` | Texture | Couleur / normal de la lune |
| `Resources/Icon128.png` | Image | Icône du plugin |

---

## 4. Pipeline de rendu (composition du matériau)

1. **Skybox** — `SM_Starfield` est rendu autour de la caméra ; son matériau (`M_Starfield` via `MI_Starfield`, ou la variante `GalaxyTex`) dessine le ciel en unlit.
2. **Échantillonnage des couches** — `MF_Starfield_Sampler` lit les textures astronomiques (`MilkyWay`, `LMC`, `SMC`, `Noise`) selon les coordonnées du ciel.
3. **Composition couleur** — `MF_Starfield_ExtractColour` extrait/mixe les contributions de chaque couche en une couleur finale.
4. **Animation** — `T_Starfield_Flow` peut moduler/distordre les couches dans le temps (scintillement, dérive) — entièrement côté matériau.
5. **Lune** — `M_Starfield_Moon` rend la lune via `T_Moon_BaseColour` + `T_Moon_Normal`, en élément séparé.
6. **Réglage** — les `MI_*` exposent les paramètres (intensités, couleurs, échelles) sans toucher aux masters ; `BP_Starfield` câble le tout pour le placement.

---

## 5. Points d'intégration

- **Aucune dépendance plugin** (`.uplugin` sans `Plugins`) ; aucun module — uniquement du contenu.
- **Consommateurs** : les niveaux qui veulent un fond stellaire posent un `BP_Starfield` (cf. `StarfieldExampleMap`). Se combine avec les systèmes d'atmosphère/ciel ([`AtmoScape`](AtmoScape_ARCHITECTURE.md), `SpaceScape` étant le **fond hors-atmosphère**) et les vues spatiales du jeu.
- **Modification** : tout réglage passe par les `MI_*` (paramètres) ou l'édition des masters/material functions dans l'éditeur — il n'y a pas d'API code.

---

## 6. Gotchas, invariants et pièges

- **Content-only : pas de code.** Ne pas chercher de classe `USpaceScape*` ni de subsystem — il n'y en a pas. Toute logique est dans les matériaux/BP.
- **Documenter = ouvrir les assets.** Les paramètres exacts (intensités, mapping des couches, animation `Flow`) ne sont visibles qu'en ouvrant `M_Starfield` / `MI_Starfield` / les material functions dans l'éditeur ; ce document décrit les **rôles** d'après l'organisation, pas les valeurs internes.
- **Migration matériaux 5.3→5.7.** Comme tout matériau custom du projet, les nodes HLSL custom/échantillonneurs ont pu nécessiter des correctifs LWC/sampler lors de la migration (cf. patterns de migration matériaux du projet) ; vérifier en éditeur si le ciel s'affiche mal.
- **Skybox = ordre de rendu.** Le mesh de ciel doit être unlit et rendu en fond ; mal configurer la profondeur/le matériau peut faire passer le ciel devant la scène ou inversement.

---

## 7. Fichiers et emplacements

| Chemin | Rôle |
|---|---|
| `SpaceScape.uplugin` | Descripteur (content-only : `CanContainContent=true`, aucun `Modules`) |
| `Content/Blueprints/BP_Starfield.uasset` | Acteur placeur |
| `Content/Objects/SM_Starfield.uasset` | Mesh de skybox |
| `Content/Materials/*` | Masters, material functions et instances du ciel/lune |
| `Content/Textures/*` | Textures astronomiques (Voie lactée, Magellan, bruit, flux, lune) |
| `Content/Maps/StarfieldExampleMap.umap` | Map d'exemple |
| `Resources/Icon128.png` | Icône du plugin |
