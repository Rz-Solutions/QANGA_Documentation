# Lot 2 du rangement dev : les 11 levels de la capitale hors `QL_YellowWall`

> Proposition du 2026-10-22, mesurée hors moteur. **Rien n'est appliqué.** Carte : `lot2_carte.png` (nord en haut).
> Liste complète, acteur par acteur : `lot2_rangement.csv`. Contexte : `Documentation/CAPITALE_YELLOWWALL_ARCHITECTURE.md` §2 bis.
> Mêmes règles que le lot 1 : le décor va dans le quartier où il est posé (quartier à moins de 200 m) ; on ne disperse ni la
> mission ni les lampadaires ; un groupe bouge entier ou pas du tout ; un déplacement qui fait grossir la cible de plus de
> 100 m est à valider.

## Résultat

- **9 levels sur 11 sont propres** : aucun objet hors de leur zone, leurs bounds collent à leur contenu.
- Les 2 autres : IcliPort (le tas, B) et WeaponFactory (6 pièces laissées, C).
- **Le lot 1 validé ne change pas.** Seul cas proche : 5 objets du lot 1 (Port vers SkyRoad) sont à 200 m de Trash Area,
  mais à moins de 100 m du parking SkyRoad : SkyRoad reste la cible.

## B. À décider : le tas d'IcliPort (117 acteurs)

117 acteurs de `L_IcliPort` (57 touffes d'herbe, antennes, tubes, cheminées, poubelles, écrans, 8 décals) sont empilés sur
4 m au cœur (X 410, Y -42, Z 115), à 4 km d'IcliPort. Ils sont libres, visibles, et réunis dans **un seul groupe**
(`GroupActor_0`, 117 membres) : quelqu'un les a regroupés exprès. Pas en jeu : IcliPort n'est chargé par rien.

Effet : les bounds d'IcliPort passent de 458 x 205 m à 2,9 x 3,2 km, et le tas apparaîtra au cœur dès qu'IcliPort sera
ajouté à `QL_YellowWall`.

**Décision du 2026-10-22 : on n'y touche pas.** Benja ne sait pas à quoi servent ces objets. Vérifié : aucun de leurs meshes
n'est dans le bake D à cet endroit (seulement 38 instances d'autres meshes à moins de 10 m : herbe de trottoir, bordures,
colonnes).

## A. Proposé : le décor du sud posé dans le level dev `L_UnderWater_Field`

Ce level de `QL_YellowWall` habille tout le sud : en plus de sa zone (clôtures, barrières), il porte de petites scènes posées
près des quartiers sud. Celles à moins de 200 m d'un quartier :

| Scène | Objets | Cible | Distance | Bounds de la cible |
|---|---|---|---|---|
| A1 Forge (enclume, tonneaux, seaux) | 9 | `L_YW_WeaponFactory` | 100 m | 1 143 x 510 m deviennent 1 143 x 584 m |
| A2 Panneau à lierre et câbles (X 1 060) | 16 | `L_YW_TrashArea` | 100 m | 148 x 103 m deviennent 229 x 149 m |
| A3 Quai, palettes, citerne, caddie | 12 | `L_YW_UnderWater_Desert` | 150 m | 538 x 538 m deviennent 593 x 575 m |
| A4 Panneau à lierre, groupe entier `GroupActor_6` (X 698) | 13 | `L_YW_TrashArea` | 150 m | **à valider** : Trash Area passerait à 416 x 193 m |
| A5 Clôtures (X 43, Y 2 558) | 4 | `L_YW_UnderWater_Desert` | 200 m | **à valider** : 751 x 575 m |

Par défaut : A1 à A3 bougent, A4 et A5 restent (ils feraient grossir leur cible de plus de 100 m). **Validé par Benja le
2026-10-22, pas encore appliqué** (script `Saved/Claude/YellowWall/light/ywl2L_A_UnderWater_Field.py`, sauvegarde faite).

Laissés dans `L_UnderWater_Field` : deux autres panneaux (2 x 29 objets, X 1 300 et 1 500) à plus de 200 m de tout quartier,
et 31 objets (rails, encadrements de porte, 5 cubes de base) sur le site de l'usine, juste au-dessus de l'intérieur
`L_Usine_Autonome_Stream` : sans doute son entrée, à voir avec toi.

Les cibles sont cuites dans D, la source en partie : rien ne change en jeu avant un rebake. Les bounds de
`L_UnderWater_Field` ne bougent pas (1 735 x 767 m) : d'autres objets les tiennent.

## C. Laissé tel quel

- `L_YW_WeaponFactory` : 6 objets à 150 m de l'usine (pièces du préfab "Petite Usine" et un lampadaire), aucun autre
  quartier à moins de 200 m.

## Pour un passage dédié plus tard

- `LevelBounds` en double : WeaponFactory, Carriere, TrashArea (2 chacun), à traiter avec les 14 levels de `QL_YellowWall`.
- Doublons exacts empilés (même mesh, même place) dans `L_UnderWater_Field` : au moins 10 paires, surtout des câbles de panneaux.

## Étape éditeur (pas lancée)

1. **Ajouter les 11 levels à `QL_YellowWall`**, dans un dossier `Enceinte` du panneau Levels. Ça ouvre `QL_YellowWall`,
   donc toute la capitale (c'est ce qui a fait planter le PC au lot 1). Le plus sûr : dans ta session, Levels > Add Existing,
   puis ranger les 11 dans le dossier. À regrouper avec l'étape lourde du lot 1, qui ouvre le même persistant (groupe Iclab
   de 26 acteurs vers Traverse, retrait de `L_Int_Complex_MiddleTower1`).
2. **Les déplacements du lot 2**, une fois validés : par petits lots comme le lot 1 léger (carte vierge, seulement les levels
   touchés, cibles sauvées avant la source). Les scripts seront générés après ta validation.
