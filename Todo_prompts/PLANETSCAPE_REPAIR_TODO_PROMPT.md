# TODO Prompt : reprendre PlanetScape après l'audit de `7587c905`

> À coller tel quel dans une future session Codex ouverte sur `G:\QANGA`.
> Checkpoint du 2026-08-23. Le commit Ox Alpha audité est
> `7587c90597314f9c3190b822ad5a78727d019f21`. Toutes les corrections décrites
> ci-dessous sont encore non commitées. Ne rien stage/commit sans demande de Rz.

## Objectif

Finir la validation du diff PlanetScape actuel sans accumuler de nouveaux
workarounds. Préserver les invariants suivants :

- une tile ne devient `Active` qu'avec mesh, instance et slice valides ;
- une slice n'est jamais recyclée tant qu'un ancien travail GPU peut l'écrire ;
- un résultat async n'est publié que si clé, génération, lifecycle et ownership
  correspondent encore ;
- aucune ressource, donnée biome ou collision absente n'est remplacée par un faux
  résultat ;
- aucun log par frame/tile et aucun diagnostic périodique en Shipping/Test ;
- ne jamais sauvegarder une map de test, cook, stage ou package sans ordre de Rz.

## Validation déjà prouvée

### Builds

- `QangaEditor Win64 Development` froid et réellement relinké après les derniers
  changements : vert. Les DLL PlanetScape sont datées de `15:20`, après les sources.
- `Qanga Win64 Development` complet : vert après recompilation PlanetScape et relink
  complet de `Qanga.exe`, 29 actions, 92.02 s.
- `Qanga Win64 Shipping` courant : vert. Son objet `PlanetTileWeightManager` et
  `Qanga-Win64-Shipping.exe` ont été reconstruits à `15:21-15:22`, puis UBT a confirmé
  la cible à jour.
- Aucun cook, stage ou package lancé.

Ne pas prendre un build externe `-LiveCoding` pour une preuve de relink : il avait
compilé les objets sans remplacer les DLL chargées au prochain démarrage froid.

### Automation

Les quatre tests sont verts après le dernier relink, zéro warning/erreur :

- `PlanetScape.Core.SphereMath.DirToFaceUV RoundTrip` : 257 600 directions,
  erreur maximale `1.602e-6 rad`, zéro cellule reclassée ;
- `PlanetScape.Core.TileAddress Depth Limit` : D16 accepté, D17 refusé ;
- `PlanetScape.Core.WeightPainting.NormalizedRedistribution` : dominance à 100 %,
  retrait d'une couche et conservation de la somme unitaire ;
- `PlanetScape.Foliage.CubeFaceWindowSeams` : 9 cellules uniques sur une seam,
  8 au coin où seulement trois faces du cube se rencontrent.

### Runtime Mercury

La seule map validée avec un acteur PlanetScape est :
`/Game/_QLevel/Universe/Planets/Mercury/L_Mercury`.
Les autres planètes utilisent WorldScape.

Après le dernier redémarrage froid sur `PlanetScapeActor_1` :

- rayon `1219 km`, grid `65`, profondeur effective `14` ;
- passage de 6 racines à `294` tiles actives / `294` instances en moins de trois
  secondes ; quatre scans successifs stables à `D=0 P=0 F=0 M=0 A=294` ;
- `1754/2048` slices libres, compteurs stables ;
- aucun timeout GPU, ensure, assertion ou erreur PlanetScape ;
- foliage désactivé sur Mercury ;
- océan normalement désactivé. Son activation temporaire non sauvegardée a créé les
  six ISM et initialisé le quadtree D12 sans erreur pendant environ une minute ;
- aucune preuve visuelle exploitable : la vue Mercury était noire/hors éclairage.

Le premier run froid était bloqué à six racines `PendingDispatch`. La cause exacte
était un second `ConsumePendingDispatches()` dans `ProcessRetires()`, qui vidait la
queue juste avant `DriveGPUDispatches()`. Il a été supprimé ; il ne reste qu'un
consumer, dans `DriveGPUDispatches()`.

## Corrections présentes dans le diff

### Scheduler, lifecycle et GPU

- génération de dispatch monotone actor-wide en `uint64`, éliminant l'ABA d'un
  record recréé à la même adresse ;
- validation complète clé/génération/lifecycle/slice avant publication ;
- slots GPU timed-out quarantined tant que leur travail peut encore finir ;
- readbacks protégés contre use-after-free et rings bloqués récupérés explicitement ;
- init terrain transactionnelle, builds de meshes vérifiés et aucun bookkeeping
  fantôme après échec ;
- D16 est la limite unique des adresses terrain, foliage et océan ;
- suppression des états, compteurs, branches et diagnostics sans producteur réel ;
- diagnostic quadtree opt-in capable de distinguer `PendingDispatch` et
  `PendingMeshBuild`, exclu de Shipping/Test.

### Atlas, matériaux et poids

- atlas paint/mask/tint réellement lazy ; promotion transactionnelle avec retour
  d'échec réel et restauration des placeholders ;
- staging RHI d'upload persistant, sans allocation transitoire par tile ;
- suppression de l'ancien render target de poids par tile ;
- store scindé entre poids procéduraux natifs transients/LRU et paint 256²
  persistant en session ; promotion au premier coup de pinceau seulement ;
- cache biome compact créé à la demande pour foliage/paint ;
- paint et matériau limités explicitement à quatre biomes ;
- peinture redistribuée entre les couches actives avec somme 1, y compris Ctrl pour
  retirer la couche ; une couche peut désormais réellement atteindre 100 % ;
- suppression du `WeightUploadFence` que l'ancien pipeline RT armait encore après
  chaque vague de tiles alors qu'aucun upload n'était produit ;
- asset backfill `/PlanetScape/Materials/M_PlanetScapeBackfill` ajouté pour le
  chemin packaged ; matériaux baked terrain/real-geo/océan validés avant usage.

### Foliage

- wrapping des fenêtres entre faces du cube et test des seams/corners ;
- génération actor-wide, annulation et ownership des tâches CPU async ;
- invalidation après merge/paint, protection contre résultats stale ;
- aucune fabrication de biome 0 quand les poids ne couvrent pas encore la cellule ;
- formules CPU/GPU et normalisation alignées ;
- états morts retirés et contrat strict de quatre biomes ;
- calculs de transform/bounds et log Verbose répétés par secteur supprimés.

### Océan et outils Editor

- init/teardown océan transactionnels, reverse mapping O(1), uploads de custom data
  regroupés, visibilité delta-only, horizon culling et D16 ;
- rayon océan résolu une seule fois et propagé terrain/clouds/foliage ;
- outil Visibility factice supprimé ;
- hit du brush strict : collision réelle ou cache GPU de hauteur réel, sans sphère ou
  heightmap CPU approximative ; rendu du brush réutilisant le hit exact ;
- logs de sculpt throttlés à 1 Hz par source ; logs de succès par stroke supprimés ;
- RzDirectMCP vérifie désormais le montage du plugin avant les opérations assets.

## Travail restant réel

### Validation finale réservée à Rz

L'Editor est laissé sur la vraie map Mercury, hors PIE. Aucun package n'a été
sauvegardé. La validation native demandée est terminée ; Rz garde le cook/package et
le contrôle perceptuel du terrain et de l'océan.

### Limites non résolues à ne pas maquiller

- Le foliage réel avec des meshes/configurations de production n'a pas été exécuté :
  Mercury le désactive et aucun jeu de meshes contrôlé n'a été identifié. Tester
  annulation, seams, paint/sculpt et teardown sur un acteur temporaire non sauvegardé.
- Les stamps biome ont encore une feature inachevée dans `TerrainGenerator.usf` :
  `BiomeSlopeCosApprox()` renvoie 1 et `MacroSlope` est une approximation de hauteur.
  Le gating des ladders de slope n'utilise donc pas une pente géométrique réelle ; le
  chemin Voronoi des stamps conserve aussi une sélection legacy approximative. Soit
  implémenter une vraie pente multi-échantillons/multi-pass, soit retirer ces réglages
  exposés. Ne pas ajouter un autre seuil approximatif.
- Le paint est persistant pendant les rebuilds/retirements de la session, mais les
  stores sont des membres C++ non sérialisés : sauvegarder puis relancer l'Editor ne
  restaure pas les coups de pinceau. Décider d'un format versionné avant de présenter
  le paint comme persistance d'asset.
- Aucun cook/package ni contrôle perceptuel terrain/océan n'est prouvé. Ils restent à
  faire par Rz sur un build Development.
- Ajouter ensuite seulement les tests d'ownership difficiles encore absents : ABA
  stale result, quarantine timeout/slice, cycles allocator et swap-remove ISM.

## État du workspace

- `Documentation` est un dépôt Git imbriqué et contient d'autres changements sans
  rapport. Ne stage que ce fichier si Rz le demande.
- Le worktree racine contient beaucoup de modifications étrangères. Ne stage que les
  fichiers explicitement attribués à PlanetScape/RzDirectMCP.
- Le dossier parasite `Plugins/RzMCPInterface` avait réapparu et cassait UHT par
  duplication de headers. Il a été déplacé de façon récupérable, pas supprimé, vers :
  `C:\Users\Rz\Documents\0RzSoftware\to delete\RzMCPInterface-QANGA-20260823-1337`.
  Ne pas le recréer dans `Plugins`.

## Sources principales

- `Plugins/PlanetScape/Source/PlanetScape/Private/PlanetScapeActor.cpp`
- `Plugins/PlanetScape/Source/PlanetScape/Private/TileScheduler.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeCore/Private/SphereMathTests.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeGPU/Private/InstancedTileRenderer.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeGPU/Private/TerrainDispatcher.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeHM/Private/PlanetTileWeightManager.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeFoliage/Private/PlanetFoliageSubsystem.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeFoliage/Private/PlanetGrassSectorBuilder.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeOcean/Private/OceanQuadtree.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeOcean/Private/OceanTileRenderer.cpp`
- `Plugins/PlanetScape/Shaders/Private/TerrainGenerator.usf`
