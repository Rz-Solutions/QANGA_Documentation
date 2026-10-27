# Yellow Wall : lot 1 du rangement dev (à valider)

> Préparé le 2026-10-22, hors moteur. Périmètre : **dev uniquement** (`QL_YellowWall` et `_QLevel/Universe/Planetary/YellowWall/SUB_LEVEL`).
> Le jeu, le bake `LO_PI_YellowWall_D` et les copies `Sous_Levels` ne sont pas touchés.
> Carte : `lot1_carte.png`. Liste acteur par acteur : `lot1_rangement.csv` (251 lignes, séparateur `;`).

## Les 6 dossiers

| Dossier | Acteurs | Cibles | Bounds du level source, avant puis après |
|---|---|---|---|
| 1. Intrus de `L_Capital_Center` | 149 | Nord 99, Litchout 22, Entry_Est 17, Port 10, Complex_Welcome 1 | 2903 x 3050 m, puis **677 x 672 m** |
| 2. Intrus de `L_Capital_Nord` | 43 | AirPort 41 (bordures), Hospital 1, Entry_Est 1 | 2169 x 1178 m, puis **959 x 429 m** |
| 3. Intrus de `L_Capital_Port` | 28 | Traverse 16, SkyRoad 7 (porte et blocs sur le parking SkyRoad), CostaRive 2, Litchout 2, BaseStruct 1 (chalutier) | 4457 x 2549 m, puis **654 x 906 m** |
| 4. Intrus de `L_Complex_Welcome` | 3 | Complex_Terrasse 1, BaseStruct 1 (plan d'écume), suppression 1 (cube) | 434 464 km, puis **688 x 682 m** |
| 5. Meshes posés dans le persistant `QL_YellowWall` | 26 | un quartier (bâtiment Iclab, 25 meshes et leur groupe) | le persistant n'a plus de géométrie |
| 6. Tour du Complex | 2 | retirer `L_Int_Complex_MiddleTower1` du persistant ; supprimer l'acteur de test `robottest` | LevelBounds de la tour : 3 km, puis environ 170 m |

Règle de cible : le quartier dont le territoire mesuré contient l'objet ; à distance égale, celui dont les bounds grossissent
le moins. Les quartiers cibles ne grossissent pas, sauf Litchout (+70 m), Traverse (+220 m si on y met le bâtiment du
dossier 5), Hospital (voir ci-dessous) et le fond `L_Capital_BaseStruct` (+700 m, il fait déjà 5,7 x 9 km).

Les 37 acteurs enfants (portes, etc.) générés par 5 Blueprints parents (`BP_YellowBuild`, `BP_Immeuble_A1`, 3 `BP_SmallShop_B`)
ne se déplacent pas à la main : ils se recréent avec leur parent. Aucun objet du lot n'appartient à un `GroupActor` partiel.

## Choix validés par Benja (2026-10-22)

1. **Cube** de `L_Complex_Welcome` (cube de base, matériau par défaut, à 434 000 km) : supprimé.
2. **Acteur de test `robottest`** à l'origine dans `L_Int_Complex_MiddleTower` (vient de `_GENERATED/Zouzixx`) : supprimé.
3. **Plan d'écume** de 6 m (`MI_Wave_Foam`) à (X -1802, Y -826) : rangé dans le fond `L_Capital_BaseStruct`.
4. **Bâtiment Iclab du persistant** à (X -1254, Y -105) : vers `L_Capital_Traverse` (le plus proche, 218 m).
5. **Maison `SM_House02`** de Nord à (X 1185, Y -368) : vers `L_Capital_Hospital` (qui passe de 365 x 350 m à 379 x 603 m).
6. **Chalutier** du Port à (X -3385, Y 1756) : rangé dans le fond `L_Capital_BaseStruct`.

## Tentative du 2026-10-22 : rien d'écrit, lot reporté

- Sans fenêtre (commandlet) : le moteur ne déplace pas d'acteurs sans interface (`MoveActorsToLevel` rend 0, sans erreur).
- Éditeur minimisé : les 210 déplacements ont réussi en mémoire et ont été vérifiés un par un. `SM_StoreFrontsPrefab44` est
  laissé dans Center : il est désactivé volontairement (caché en jeu, caché dans l'éditeur, éditeur seulement).
- Le PC a planté pendant la sauvegarde (capitale entière chargée, 74 levels, pendant un autre travail lourd). Vérifié
  ensuite : les 15 fichiers sont intacts (md5 identiques à l'origine), rien d'autre n'a été écrit, aucune restauration
  automatique en attente.
- À refaire plus léger : ne charger que les levels touchés, sauver par petits lots, machine libre confirmée juste avant.

## Version légère (prête, rien de lancé)

Scripts dans `Saved/Claude/YellowWall/light/`, générés depuis la liste validée (210 déplacements, 2 suppressions).
L'éditeur démarre sur sa carte de démarrage (légère). `ywl1L_0_blank.py` crée une carte vierge jamais sauvée. Ensuite,
un lot par level source, lancés un par un :

| Lot | Source | Levels chargés | Contenu |
|---|---|---|---|
| A | Center | 6 | 111 déplacements vers Nord, Litchout, Entry_Est, Port, Complex_Welcome |
| B | Nord | 4 | 43 déplacements vers AirPort, Entry_Est, Hospital |
| C | Port | 6 | 28 déplacements vers BaseStruct, CostaRive, Litchout, SkyRoad, Traverse |
| D | Complex_Welcome | 3 | 2 déplacements, suppression du cube |
| E | Int_Complex_MiddleTower | 1 | suppression de `robottest` |

Chaque lot : vérifie qu'il est bien dans la carte jetable, charge ses levels sans doublon, contrôle chaque acteur
(présent, bonne classe, bonne position, ni masqué ni verrouillé), déplace, vérifie dans la cible et dans la source, puis
sauve les cibles d'abord et la source en dernier (un plantage laisserait au pire un doublon visible, jamais un objet
perdu), et décharge ses levels. Au moindre écart, il s'arrête sans rien sauver. Contrôle md5 et sauvegarde avant de commencer,
contrôle hors moteur après chaque lot, fermeture par `ywlot1_3_quit.py`.

**Étape à part (demande d'ouvrir `QL_YellowWall`, donc ses 74 levels)** : les 26 objets du bâtiment Iclab posés dans le
persistant vers `L_Capital_Traverse`, et le retrait du sous-level doublon `L_Int_Complex_MiddleTower1`. Le plus léger est
de le faire à la main en ouvrant `QL_YellowWall` (2 minutes) : rendre `L_Capital_Traverse` courant, sélectionner
`GroupActor_358` (25 meshes vers X -1252, Y -106 m), clic droit puis Level puis « Move Selection to Current Level » ;
dans le panneau Levels, retirer `L_Int_Complex_MiddleTower1` (le fichier reste sur disque) ; sauver.

## Préparation (faite, rien d'appliqué)

- Scripts éditeur générés depuis la liste validée, relus, sans motif interdit par le pont :
  `Saved/Claude/YellowWall/ywlot1_0_preflight.py` (contrôle seul), `ywlot1_1_apply.py` (déplace 211 acteurs, supprime 2,
  retire le doublon), `ywlot1_2_verify_save.py` (vérifie par classe et position, puis sauve uniquement les 15 levels attendus).
- Vérifié hors moteur : aucune référence vers ces acteurs, ni dans leurs levels, ni depuis les autres levels de dev ; les
  15 levels ont la mise à jour automatique des `LevelBounds`.
- Empreinte md5 des 15 fichiers prise après l'analyse (`lot1_md5_analyse.txt`) ; `lot1_backup.sh` refuse de sauvegarder si un
  fichier a changé depuis, puis copie dans `C:/UE5_Projects/_Backups_Claude/` (hors projet, non synchronisé) et vérifie les copies.
- Pendant les déplacements, l'éditeur passe par le presse-papiers Windows : ne pas copier-coller pendant ces quelques secondes.

## Application (après ta validation, quand ta machine est libre)

- Copie de sauvegarde des `.umap` touchés : `QL_YellowWall` et, dans `SUB_LEVEL`, Center, Nord, Port, Complex_Welcome,
  Int_Complex_MiddleTower, plus les cibles AirPort, Litchout, Entry_Est, Hospital, Traverse, CostaRive, SkyRoad,
  BaseStruct, Complex_Terrasse. Aucun n'est dans les réserves (`L_Capital_Mission`, `L_Capital_Qtrain`, `L_Light_YellowWall`).
- Dans l'éditeur, `QL_YellowWall` ouvert : script Python qui lit `lot1_rangement.csv`, déplace les acteurs « deplacer »
  vers leur level cible (`EditorLevelUtils.move_actors_to_level`), fait les suppressions validées, retire le sous-level
  doublon, puis recalcule les `LevelBounds`. Un premier passage à blanc liste ce qu'il ferait, sans rien modifier.
- Vérification après coup avec l'outil hors moteur : les bounds mesurés doivent correspondre à la colonne « après ».
- Les fichiers sont synchronisés chez RzZz : on le prévient avant.
