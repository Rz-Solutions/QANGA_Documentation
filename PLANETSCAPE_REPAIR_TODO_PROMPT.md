# TODO Prompt : terminer la réparation structurelle de PlanetScape

> À coller tel quel dans une future session Codex ouverte sur `G:\QANGA`.
> État relevé le 2026-08-20 après un audit complet du plugin et une contre-vérification
> indépendante sur `L_Mercury`.
> Rz a donné carte blanche pour corriger les éléments ci-dessous. Cette autorisation couvre
> PlanetScape et les assets strictement nécessaires à son fonctionnement, pas les changements
> sans rapport dans QANGA ni la sauvegarde des maps déjà dirty.

## Mission

Réparer PlanetScape à la racine : rendre le terrain et l'océan réellement cookables, rendre le
cycle de vie tile/slice/dispatch transactionnel, supprimer les coûts et chemins morts identifiés,
puis valider dans l'Editor sur Mercury. Préparer ensuite une validation Development packaged que
Rz exécutera lui-même.

Ne masque aucun producteur ou état manquant :

- pas de matériau runtime editor-only présenté comme solution packaged ;
- pas de fallback biome 0 quand les poids sont absents ;
- pas de tile déclarée Active sans instance, slice ou résultat correspondant ;
- pas de timeout qui laisse un readback réutilisable ambigu ;
- pas de compteur, cache ou branche parallèle qui contourne le propriétaire existant.

Relis les signatures, les headers et le diff courant avant toute édition : ce document est un
checkpoint daté, pas une autorisation d'appliquer aveuglément une ancienne hypothèse.

## État de l'Editor et de Mercury

- L'Editor QANGA observé était UE 5.7.3, avec `L_Mercury` chargé :
  `/Game/_QLevel/Universe/Planets/Mercury/L_Mercury`.
- Il y a exactement un `PlanetScapeActor` dans cette map, à l'origine.
- Les autres niveaux de planètes utilisent WorldScape, pas PlanetScape. Ne pas les prendre comme
  cibles de validation ni modifier leurs actors.
- À la fin de l'audit, PIE était arrêté, le trace recorder arrêté, le profiler PlanetScape à 0 et
  le groupe de stats désactivé.
- `L_Mercury`, `/Script/SlateCore` et `/Script/MediaPlate` étaient déjà dirty. Ne sauvegarde
  ni ne réinitialise ces packages sans une décision explicite de Rz.
- Le plugin `Plugins/PlanetScape` était propre à la fin de l'audit. Revalide cet état avant
  toute modification ; le dépôt global est souvent dirty.
- `Documentation` est un dépôt Git imbriqué. Ne stage ni ne commit automatiquement ce document
  ou le plugin.

Configuration live constatée sur Mercury :

| Élément | Valeur |
|---|---:|
| Rayon de base | 1 219 km |
| Amplitude terrain | 2 199 269,25 cm, soit 21,99 km |
| Patch | `Medium_64` : grille 65 x 65 |
| LOD | seuil 64, 16 dispatches/frame, look-ahead 0,5 s, cible 2 m |
| Profondeur automatique | D14 |
| Slot atlas | 2 048 |
| Skirt instancié configuré | 50 000 cm |
| Biomes | 4 |
| POM | actif, 16 steps, 15–30 m |
| Ombres | actives |
| Foliage / océan | désactivés sur Mercury |

À D14, une tile couvre environ 116,9 m d'arc moyen et un segment environ 1,83 m. Une grille
partagée représente 4 225 sommets de surface, 4 485 avec skirt et 8 704 triangles. Les six
grilles de face sont bien partagées : ne recrée pas un mesh par tile.

## Preuve runtime déjà disponible

Artefacts conservés :

- `Saved/EasyTraceAnalyzer/PlanetScape_Mercury_Editor_Audit_20260820.utrace`
- `Saved/EasyTraceAnalyzer/PlanetScape_Mercury_Editor_Audit_20260820_analysis.json`
- `Saved/EasyTraceAnalyzer/PlanetScape_Mercury_Editor_Audit_20260820_stable.json`

La fenêtre stable de l'Editor donne 2 153 frames actives, 9,34 ms moyen, 11,70 ms p95 et
13,30 ms p99. Le scope `PlanetScapeActor` est à 0,353 ms moyen, 1,23 ms max. Les files GPU
Graphics et Compute reflètent tout le viewport et ne doivent pas être attribuées à PlanetScape.
Le hitch de 294,4 ms est un hitch Editor isolé.

Mesures PIE ciblées, séparées de la trace stable :

- `Scheduler.Update` : 0,215 ms ;
- `UpdateMeshVisibility` : 0,142 ms ;
- `ConsumeGPUResults` : 0,001 ms.

Sur la surface de Mercury, 1 838/2 048 slices étaient utilisées. Dans le test volontairement
extrême avec la caméra au centre de la planète, l'atlas a atteint 2 048/2 048 avec 18 à 22 leaves
dormantes. Ce dernier état est un stress test de saturation, pas une mesure gameplay représentative.

Ces traces Editor/PIE ne prouvent ni les FPS packaged ni le comportement d'un build cooké. Après
les réparations, Rz doit lancer un nouveau Development packaged puis fournir une trace comparable.

## Ordre de travail obligatoire

1. Établir le baseline actuel : `git status --short` au root et dans `Documentation`, vérifier
   l'Editor sélectionné, la map courante, l'état PIE et les packages dirty.
2. Corriger en premier le chemin packaged terrain, puis le chemin packaged océan. Ne commence pas
   des micro-optimisations avant que les matériaux soient réellement cookables.
3. Refaire ensuite l'ownership transactionnel tile/slice/dispatch. Une tile, une génération, une
   réservation de slice, un abandon et une libération doivent avoir un propriétaire unique.
4. Corriger les bornes d'adressage et les échecs d'instance avant de pousser la profondeur ou les
   budgets.
5. Retirer les readbacks/caches inutiles et rendre la saturation atlas sans travail futile.
6. Corriger foliage et océan lorsqu'ils sont activés, puis supprimer les API/états morts.
7. Compiler le module/editor après chaque lot C++ cohérent. Ne cooke, ne stage et ne package pas :
   Rz réalise ces opérations.
8. Laisser un test Editor inspectable sur `L_Mercury`, sans sauvegarder la map dirty, et signaler
   précisément ce qui reste à vérifier en packaged.

## Lot 1 — matériaux terrain et océan cookables

### Terrain : blocage packaged critique

Le renderer instancié fabrique un `UMaterial` à partir de graph nodes sous `WITH_EDITOR`.
Hors Editor, `CreateDisplacementMaterial()` retourne systématiquement `nullptr` :

- `Plugins/PlanetScape/Source/PlanetScapeGPU/Private/InstancedTileRenderer.cpp:988`
- l'initialisation a déjà créé les meshes et RT arrays avant de retourner ;
- `Shutdown()` retourne lui aussi tôt lorsque `bInitialized == false` ;
- l'actor continue son scheduler et les dispatches, alors que le renderer ne peut plus afficher.

Le champ public `SurfaceMaterial` est exposé sur `APlanetScapeActor`, mais l'actor passe
`nullptr` et le paramètre `BaseMaterial` de `FInstancedTileRenderer::Initialize()` n'est pas
utilisé.

Réparation attendue :

- créer ou utiliser un vrai material asset cookable pour le WPO instancié ;
- faire de `SurfaceMaterial` le propriétaire réel du material, ou le remplacer proprement par une
  référence asset explicitement nommée ;
- conserver le contrat d'atlas Heightfield, Normal, Weight, PaintMask et StampTint ;
- ne pas dégrader vers un matériau lit par défaut ou une surface plate ;
- rendre `Initialize()` transactionnelle : sur tout échec, nettoyer les composants, textures et
  RHI déjà créés, puis signaler l'échec ;
- ne jamais laisser le scheduler faire croire que le renderer est disponible.

Le renderer realtime shadow-only contient le même type d'hypothèse editor-only. Vérifie tous les
chemins de matériaux, pas seulement l'ISM principal.

### Océan : blocage packaged conditionnel

Le placeholder procédural de l'océan retourne `nullptr` hors Editor :

- `Plugins/PlanetScape/Source/PlanetScapeOcean/Private/OceanMaterialBuilder.cpp:444`
- un `OverrideMaterial` asset valide peut fonctionner ;
- sans override, l'océan peut être marqué initialisé sans matériau utilisable.

Réparation attendue :

- un asset océan cookable par défaut, ou une obligation explicite, validée et bloquante, d'override
  asset ;
- aucune initialisation réussie si le matériau nécessaire est absent ;
- un test du chemin par défaut et du chemin override.

## Lot 2 — cycle de vie terrain transactionnel

### Invalidation, slices et instances

`FTileScheduler::InvalidateTile()` démote une tile Active en Dormant sans mettre en retraite
l'instance ISM ni libérer sa slice :

- `Plugins/PlanetScape/Source/PlanetScape/Private/TileScheduler.cpp:818`
- le dispatch suivant réserve une nouvelle slice ;
- `InstancedSliceMap` est ensuite réécrit, rendant l'ancienne slice inaccessible.

Réparation attendue :

- concentrer la destruction/retrait dans un chemin unique ;
- préserver le mesh précédent seulement si c'est un état explicitement possédé et temporaire ;
- libérer exactement la slice appartenant à la génération abandonnée ;
- rendre impossible l'écrasement silencieux d'une association tile -> slice ;
- vérifier qu'une succession de sculpt/invalidate/rebuild ne fait jamais croître
  `AllocatedSliceCount()` au-delà des tiles réellement réservées.

### Identité de génération

`FInstancedDispatchResult` ne porte pas de génération. `ConsumeGPUResults()` vérifie la clé et
la présence d'une slice, mais pas que le résultat appartient à la réservation courante :

- `Plugins/PlanetScape/Source/PlanetScapeGPU/Public/TerrainDispatcher.h`
- `Plugins/PlanetScape/Source/PlanetScape/Private/PlanetScapeActor.cpp:2376`.

Une même adresse peut être retirée, recréée, puis accepter un vieux résultat ou libérer la slice
du nouveau dispatch après un timeout.

Réparation attendue :

- ajouter une génération monotone au record tile et au résultat dispatch ;
- capturer la génération et la slice réservée au moment de l'enqueue ;
- accepter un résultat uniquement si clé, génération, lifecycle et slice correspondent ;
- sur rejet, libérer seulement la réservation de ce résultat si elle lui appartient encore ;
- annuler/invalider proprement les résultats au merge, rebuild, sculpt et teardown.

### Readbacks et sculpt undo

Le chemin foliage recrée son readback à chaque dispatch et est déjà protégé. Ne le régresses pas.
Le chemin terrain instancié conserve les readbacks par slot ; au timeout il invalide seulement la
clé. Revois ce contrat avec la génération plutôt que d'ajouter une temporisation arbitraire.

`RestoreSculptDelta()` invalide Active et PendingMeshBuild mais pas GPUInFlight. Un résultat
lancé avant undo/redo peut revenir après la restauration :

- `Plugins/PlanetScape/Source/PlanetScape/Private/PlanetScapeActor.cpp:3934`.

L'undo, le sculpt normal, le merge et la destruction doivent tous passer par le même contrat
d'invalidation.

### Échec AddInstance et proxy render-thread

Le retour de `InstancedRenderer.AddTileInstance()` est ignoré : la tile peut devenir Active sans
instance. L'océan a le même défaut avec `AddInstance()`.

Réparation attendue :

- aucun `NotifyMeshReady()` si l'instance n'existe pas ;
- release/retry explicite de la réservation en cas d'échec ;
- diagnostic rare et rate-limité, pas un log par frame.

`PlanetTileMeshComponent::ReconcileProxy()` retire les sections obsolètes et synchronise leur
visibilité, mais ne recrée pas une section absente du proxy RT. Son commentaire revendique une
récupération qu'il ne réalise pas.

Réparation attendue :

- rendre le reconcile réellement autoritatif ou supprimer cette promesse ;
- ne pas recopier/enqueue la map complète à chaque tick lorsqu'aucune section n'a changé ;
- conserver une seule voie de publication GT -> RT.

### Collision

`DestroyTileCollision()` appelle correctement `CancelPendingCook()`. Ne supprime pas cette
protection. En revanche, `TeardownSystems()` détruit directement les composants de
`TileCollisionMap` et du pool sans ce chemin.

Réparation attendue :

- toute destruction, mise en pool ou teardown complet doit annuler le cook en attente avant
  destruction ;
- vérifier un rebuild/destruction pendant cook async ;
- ne pas attribuer un résultat de cook tardif à un composant remis en pool.

## Lot 3 — adresses, capacité et budgets

### Adressage

`FTileAddress::Col` et `Row` sont des `uint16`. D16 est la dernière profondeur encodable
correctement ; la création de D17 wrap les coordonnées. Le calcul de profondeur peut aller jusqu'à
D18 :

- `Plugins/PlanetScape/Source/PlanetScapeCore/Public/TileAddress.h`
- `Plugins/PlanetScape/Source/PlanetScapeCore/Public/SphereMath.h`.

Mercury D14 est sûre. La réparation doit choisir explicitement :

- élargir et repacker l'adresse/les clés de façon compatible avec tous les consommateurs ;
- ou limiter strictement la profondeur effective à D16 dans UI, validation et scheduler.

Ne cache pas le problème avec un clamp local qui laisse les autres producteurs demander D17.
Ajoute un test d'adresse aux limites D16/D17 et de pack/unpack/child/parent.

### Saturation atlas

Le `TileCountHardCap` à 14 336 et les 2 048 slices sont deux ressources différentes. Ce n'est
pas un bug sur Mercury. Le bug réel est :

- l'UI permet `MaxInstancedSlots=4096` ;
- le renderer bride à 2 048 pour la limite Texture2DArray D3D12 ;
- le scheduler reçoit malgré tout le budget brut 4 096 et ne voit donc pas la saturation réelle.

À 2 048/2 048, le scheduler peut encore produire jusqu'à 16 PendingDispatch. Chaque appel
`AllocateSlice()` rescane linéairement les 2 048 entrées avant d'échouer : jusqu'à 32 768 tests
inutiles/frame dans le cas mesuré.

Réparation attendue :

- exposer une seule capacité effective, validée au même endroit, à l'UI, au renderer, au scheduler
  et aux diagnostics ;
- utiliser une free-list ou une structure O(1) pour réserver/libérer les slices ;
- ne pas mettre une tile en PendingDispatch s'il n'existe ni slot libre ni éviction réellement
  possible dans cette frame ;
- conserver l'hystérésis de priorité quand elle est utile, sans refaire chaque frame un travail
  voué à échouer ;
- faire correspondre `freeSlices` aux slices effectives, jamais à la valeur sérialisée brute.

## Lot 3 bis — projection et accord terrain / gameplay

Les points de cette section viennent du premier audit complet. Ils n'ont pas été re-mesurés dans le
contre-audit Mercury du 2026-08-20 : les reproduire sur le code courant avant toute correction et
rapporter le résultat, y compris s'ils ont déjà été réparés entre-temps.

### Inverse cube-face

SphereMath::DirToFaceUV() était suspecté d'utiliser un Newton diagonal qui ignore les dérivées
croisées de la projection equal-area. Un test indépendant de 200 000 directions avait mesuré une
erreur maximale d'environ 8.53e-4 rad, environ 2,70 km au rayon alors audité, avec jusqu'à
27,9 % de cellules reclassées.

Ce contrat est partagé par terrain, foliage, brush, caches de poids, nav et toute recherche de
tile par direction. Ne corrige pas seulement un consommateur.

Réparation attendue si la reproduction confirme le défaut :

- inverser exactement la même projection que FaceUVToSphereDirD() ;
- utiliser le Jacobien complet ou une méthode robuste réellement convergente ;
- tester toutes les faces, seams, pôles, coins et allers-retours direction -> UV -> direction ;
- vérifier que les lookups terrain, poids et foliage utilisent tous la même convention.

### Biomes, paint et niveau d'océan

Le premier audit avait aussi signalé un risque de désaccord entre les poids CPU foliage et le
matériau GPU, une normalisation paint différente et un changement de OceanLevel sans rebake des
consommateurs dépendants. Reproduire ces trois cas avec des données contrôlées.

Si le défaut existe encore :

- faire dériver CPU et GPU de la même source de poids et de la même normalisation ;
- invalider/reconstruire exactement les données dépendantes du niveau océan ;
- couvrir chaque transition par un test ou une validation Editor observable.

## Lot 4 — mémoire, WPO et maintenance

### Readback complet et caches

Chaque dispatch instancié lit 65 x 65 `FTerrainVertex` complets :

- 135 200 octets, environ 132 KiB, par dispatch ;
- 2,06 MiB/frame à 16 dispatches ;
- les résultats servent au cache de hauteurs, à la géométrie shadow-only locale et à la collision.

Le cache runtime invoqué dans les commentaires pour `RTSScapePlanetNavBridge` n'existe pas dans
le dépôt. Les consommateurs trouvés de `SampleSurfaceRadiusCM()` sont editor.

Réparation attendue :

- séparer les données nécessaires au rendu WPO (GPU seulement) de celles nécessaires au gameplay ;
- ne lire les vertices complets que pour une zone qui nécessite effectivement shadow geo, collision
  ou un consommateur runtime réel ;
- conserver le minimum de données CPU pour `SampleSurfaceRadiusCM()`, avec un contrat clair ;
- ne pas introduire un second cache global de sommets.

Le log `estVertexMB` utilise `80 B * GridResolution² * ActiveTiles`. Ce n'est pas une mesure
fiable : 80 B n'est dérivé d'aucun `sizeof` et les couches ne sont pas toutes actives.

Réparation attendue :

- utiliser les métriques RHI exactes déjà disponibles pour les atlases ;
- afficher séparément atlas, height cache, zone shadow, collision et buffers en vol ;
- inclure le padding réel 67 x 67 des textures quand il est pertinent ;
- supprimer le faux total plutôt que le conserver comme pseudo-mesure.

Référence : à 2 048 slices Medium_64, height atlas + normal atlas représentent environ 105,2 MiB
persistants ; le cache de hauteurs seul représente environ 33 MiB si 2 048 tiles y sont stockées.
Les paint/stamp atlases sont paresseux, mais leur première promotion peut ajouter environ 713 MiB.

### WPO / skirts

`InstancedSkirtDepthCM` est présenté comme une profondeur physique constante, mais la custom
data envoie `SkirtDepth * TileUVExtent`. À D14, 50 000 cm deviennent 3,05 cm. Le shader applique
bien cette valeur radialement.

L'océan suit le même contrat ; son skirt automatique de 10 000 cm devient 0,61 cm à D14.

Décider et documenter le contrat réel :

- profondeur physique constante : ne pas multiplier par l'étendue UV ;
- profondeur relative à la tile : renommer le réglage, corriger l'UI et valider les seams à chaque
  profondeur.

Ne modifie pas ce facteur sans captures proches des seams terrain et océan.

### Travail/logs par frame

Dans `APlanetScapeActor::Tick()` :

- eviction poids toutes les 120 frames ;
- compaction toutes les 600 frames ;
- diagnostic mémoire toutes les 3 600 frames ;
- scan complet de `Scheduler.GetAllTiles()` et log toutes les 120 frames.

Ces fréquences dépendent des FPS. À plus de 120 FPS, le résumé peut dépasser un log/seconde et le
scan de toutes les tiles est un coût récurrent sans finalité runtime.

Réparation attendue :

- utiliser un délai en secondes pour maintenance et logs ;
- ne scanner complètement le registre que si un diagnostic explicite est actif ;
- respecter au maximum un log/seconde/source ;
- ne pas compacter périodiquement sans preuve que le gain dépasse le coût.

## Lot 5 — foliage

Mercury désactive foliage, donc les constats suivants sont statiques. Active-le sur une map de test
contrôlée ou un actor temporaire non sauvegardé ; ne modifie pas les autres planètes WorldScape.

### Défauts confirmés

1. Les voisins de cellule au bord d'une cube-face restent sur la même face. Le code reconnaît que
   le wrapping cube-face est « future ». Cela laisse une cellule de foliage manquante aux seams.
2. Le contrat accepte plus de quatre biomes, mais le builder ignore silencieusement
   `BiomeIdx > 3`. Mercury en a quatre, mais ce comportement est une fonctionnalité inachevée.
3. `FPlanetGrassSectorBuilder::RunAsync()` lance une tâche sans handle ni token d'annulation.
   Invalidation, sculpt ou téléportation peuvent laisser plusieurs builds CPU coûteux continuer
   pour des résultats jetés.
4. Si le WeightManager ou une tile manque, le code force les poids `(1,0,0,0)` et pose le foliage
   du biome 0. C'est un fallback de données incorrect : il faut attendre/replanifier avec des poids
   exacts.
5. Les états `ComputeRunning` et `PendingDestroy` sont morts ou jamais assignés.

Réparation attendue :

- une conversion cube-face correcte pour toutes les cellules de la fenêtre ;
- un contrat explicite de quatre couches seulement, ou une vraie représentation extensible des
  poids/biomes ; ne pas ignorer silencieusement les couches supplémentaires ;
- un token de génération/cancellation pour les builds CPU, vérifié avant publication ;
- une attente/retry bornée sur les poids absents, sans créer de secteur visuel faux ;
- simplifier la machine d'états et supprimer les états non consommés.

Vérifier aussi que les poids foliage sont la même source et la même normalisation que le matériau
terrain, y compris après paint/sculpt.

## Lot 6 — océan

Mercury désactive l'océan. Les tests doivent donc être réalisés dans une map/actor PlanetScape
contrôlé, sans toucher aux niveaux WorldScape.

### Défauts confirmés

1. Le matériau océan par défaut n'est pas cookable, décrit dans le lot 1.
2. `OceanRadiusOverrideCM` est exposé mais jamais lu : l'actor calcule toujours son rayon/niveau
   automatiquement.
3. `FOceanFFTDispatcher::UpdateSpectrum()` oublie `Choppiness` dans son test de cache. Changer
   uniquement cette valeur laisse l'ancien spectre actif.
4. `OceanTileRenderer::AddTileInstance()` continue après `INDEX_NONE`, écrit les custom data et
   enregistre la tile ; l'actor peut ensuite appeler `OnTileReady()`.
5. Le quadtree active des tiles sous l'horizon puis les masque seulement après. Cela gaspille
   budget LOD et instances.
6. Les tableaux de candidats/merge sont recréés et triés à chaque tick au lieu d'être scratch
   préalloués.
7. `BindFFTTextures(FTextureRHIRef, FTextureRHIRef)` est une API no-op jamais appelée ; seule la
   variante UTexture est active.
8. Toute vraie bascule de visibilité fait `SetCustomDataValue(..., true)`. Le nombre de
   rebuilds coalescés par UE doit être mesuré avant de choisir l'API de remplacement.
9. Les bounds à 3x le rayon désactivent frustum/distance culling natif pour le WPO. C'est un choix
   structurel ; il faut prouver que le cull GPU/visibility compense réellement.

Réparation attendue :

- faire fonctionner ou retirer `OceanRadiusOverrideCM` ;
- inclure toutes les variables de spectre dans la clé de cache ;
- appliquer le même contrat d'échec instance que le terrain ;
- filtrer les candidats sous l'horizon avant allocation lorsque c'est géométriquement sûr ;
- réutiliser les buffers de candidats ;
- supprimer les APIs mortes et custom-data slots non consommés ;
- mesurer la visibilité avant tout changement de stratégie render-thread.

## Nettoyage obligatoire après les réparations

Supprimer, plutôt que laisser « pour plus tard » :

- `SurfaceMaterial` ou son ancien chemin s'il reste inutilisé ;
- le paramètre `BaseMaterial` s'il n'a plus de rôle ;
- `bShowISM` s'il est toujours forcé à true ;
- `ETileRenderPath`/champ par tile si Instanced est réellement l'unique renderer ;
- `UpdateRebaseOrigin()` no-op et `FaceInstanceDataDirty[]` si aucun consommateur réel ne reste ;
- les paramètres de material shadow explicitement prunés/no-op ;
- l'API océan `BindFFTTextures` no-op ;
- les états foliage morts.

Ne garde pas un état sérialisé, une propriété UI ou un commentaire qui annonce une capacité
inexistante.

## Validation exigée

### Compilation et lecture de diff

- Relire headers, appels et consommateurs avant chaque refactor.
- Compiler l'Editor selon le script projet :

  ```powershell
  "E:\UE573\Engine\Build\BatchFiles\Build.bat" QangaEditor Win64 Development -Project="G:\QANGA\QANGA.uproject" -WaitMutex
  ```

  Si Live Coding est actif, ajouter `-LiveCoding`. Ne jamais utiliser `-NoLiveCoding`.
- Vérifier les fichiers touchés et le diff ; ne stage ni ne commit sans demande.
- Ne cooke/ne package pas : Rz le fait lui-même.

### Tests automatisés à ajouter ou renforcer

Les tests purs doivent rester adjacents à la math/ownership la plus dure :

- pack/unpack/child/parent de `FTileAddress` aux limites D16/D17 ;
- génération tile : résultat périmé, timeout, merge, invalidate et undo sculpt ne peuvent jamais
  publier ni libérer la réservation d'une génération plus récente ;
- allocator slice : aucun duplicate, aucune fuite après plusieurs cycles ;
- cache FFT : un changement isolé de `Choppiness` force une mise à jour ;
- conversion cube-face foliage et fenêtres traversant chaque seam.

### Validation Editor

1. Ouvrir ou conserver `L_Mercury` sans enregistrer sa map dirty.
2. Vérifier qu'il y a un seul PlanetScapeActor et que terrain/biomes/WPO restent visuellement
   corrects.
3. Faire varier la caméra jusqu'à forte pression atlas ; observer slices, lifecycle et logs.
4. Répéter sculpt -> undo/redo pendant des dispatches GPU, puis vérifier qu'aucun ancien relief ne
   revient.
5. Activer foliage/ocean seulement dans un environnement contrôlé et vérifier seams, changement de
   choppiness, visibilité horizon et échec d'instance.
6. Capturer une nouvelle trace avec EasyTraceAnalyzer. Ne pas attribuer les queues GPU globales au
   plugin sans scopes dédiés.

### Validation packaged à laisser à Rz

Après compile et assets cookables, demander un nouveau Development packaged avec une trace de la
même scène. La validation doit prouver :

- terrain WPO présent avec le matériau asset cooké ;
- océan par défaut présent si activé, et override correct ;
- absence de fuite de slices après invalidations ;
- pas de PendingDispatch infini à capacité ;
- pas de phénomène de stale result après sculpt/undo ;
- métriques CPU/GPU comparables à la capture Mercury de référence.

## Sources principales

- `Plugins/PlanetScape/Source/PlanetScape/Private/PlanetScapeActor.cpp`
- `Plugins/PlanetScape/Source/PlanetScape/Private/TileScheduler.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeCore/Public/TileAddress.h`
- `Plugins/PlanetScape/Source/PlanetScapeCore/Public/SphereMath.h`
- `Plugins/PlanetScape/Source/PlanetScapeGPU/Private/InstancedTileRenderer.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeGPU/Private/TerrainDispatcher.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeGPU/Private/PlanetTileMeshComponent.cpp`
- `Plugins/PlanetScape/Source/PlanetScape/Private/TileHeightfieldCollision.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeFoliage/Private/PlanetFoliageSubsystem.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeFoliage/Private/PlanetGrassSectorBuilder.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeOcean/Private/OceanTileRenderer.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeOcean/Private/OceanFFTDispatcher.cpp`
- `Plugins/PlanetScape/Source/PlanetScapeOcean/Private/OceanQuadtree.cpp`

## Handoff attendu à la fin

Le prochain agent doit rendre :

- les fichiers réellement modifiés ;
- les invariants nouvellement garantis ;
- les tests compilés/exécutés et leur résultat ;
- la preuve Editor observable ;
- ce qui attend précisément le Development packaged de Rz ;
- les éventuels écarts entre ce checkpoint et le code réellement trouvé.

Ne prétends jamais que le terrain packaged, l'océan packaged ou la performance cible sont validés
tant qu'un Development cooké correspondant n'a pas été exécuté et tracé.
