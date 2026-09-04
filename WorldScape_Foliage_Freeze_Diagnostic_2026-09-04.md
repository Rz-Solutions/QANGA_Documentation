# WorldScape foliage : diagnostic des freezes en editeur (2026-09-04)

Demande de Benja : rendre fluide la generation des foliages WorldScape. Observation de depart, en editeur sur L_Persistent_Universe : des freezes reguliers en se deplacant sur la planete, qui disparaissent quand la liste `Foliages` du root est videe, plus des freezes plus petits et moins frequents d une autre origine.

Ce document est le diagnostic mesure, avant tout code. Aucun fichier du projet n a ete modifie par cette session (carte L_Dev_Claude md5 `36839F4F587E343D769BD4EA53E673E8` avant et apres, aucun asset ecrit). Les scripts de mesure et les exports sont dans le scratchpad de session `df7503aa` (`ws_foliage_*.py`, `analyze_*.py`, `insights_*.tsv`, `trace_*.utrace`, `ws_foliage_editor.log`).

## 1. Ce qui a ete mesure, et comment

Reproduction sur `/Game/Maps/LevelDev/L_Dev_Claude` (regle de validation), editeur seul, sans PIE, binaires WorldScape du 2026-09-03 14:39.

- Le root de L_Dev_Claude tel que sauve : 1 collection (`/Qasset/WS_Foliage/Earth/BiomeGeologic/Biome_Geologic`), `Foliage_ForceDisabledCollision = True`, `FoliageCollisionPooling_Enabled = True`, `Foliage_MobilityType = Static`, `bUseGPUNoise = True`, `bUseIndirectInstancedNoise = True`, LodResolution 32, TickPerSecond 60, `OverrideMainTick = True`, `UsePostSlateTick = True`.
- Le jeu de foliage de l Univers a ete reproduit en memoire sur ce root : les 9 collections referencees par le root Terre de `L_Earth.umap` et de `L_Persistent_Universe.umap` (assets du plugin Qasset : Biome_Geologic, Geologic_Cliff, Biome_Beach, Biome_Desert, Biome_Ice, Biome_TemperateForest, Biome_TemperateSavannah, Biome_TropicalForest, GameplayActorWS), soit 55 entrees de foliage (assets ISM, clusters, et 6 Blueprints d acteurs : sons de vent, oiseaux, vagues, poussiere, spawner Sangline). Les deux cartes de l Univers portent aussi `Foliage_ForceDisabledCollision = True` et `FoliageCollisionPooling_Enabled = True` (presence des noms dans leur table de noms).
- Trajet : au ras du sol (3 m), en ligne droite tangente, dans la zone temperee au sud-ouest du PlayerStart (autour de X = -450 km, Y = -267 km en monde). Position joueur de WorldScape forcee par `bOverridePlayerPosition` (ecriture sans notification) et rendu suivi par un CameraActor pilote par le viewport actif. Erreur de suivi mesuree : 25 cm a 15 m/s.
- Instruments, tous non visuels : `stat dumphitches` avec `t.HitchFrameTimeThreshold 45` (hierarchie de stats des images lentes dans le log), `CsvProfile` (series par image), `Trace.File cpu,frame,counters` puis export sans interface par `UnrealInsights.exe -OpenTraceFile -ExecOnAnalysisCompleteCmd` (statistiques par timer et evenements du game thread), `DebugFoliageGenerationTiming` sur le root (lignes `[FOLIAGE-FULLPARALLEL]`), et un marqueur par tick (delta Slate, nombre de composants ISM et d instances, etat du worker).

## 2. Resultats

| Mesure | Foliage du root | Vitesse, duree | Images | Cadence | Image moyenne | Pire image | Images > 50 ms | Cumul > 50 ms |
|---|---|---|---|---|---|---|---|---|
| A, carte sauvee | 1 collection (geologique) | 15 m/s, 90 s | 5053 | 56 img/s | 17,9 ms | 125 ms (demarrage des instruments) | 1 | 51 ms |
| B, jeu Univers | 9 collections, 55 entrees | 15 m/s, 90 s | 4935 | 55 img/s | 18,3 ms | 89 ms | 4 | 287 ms |
| C, temoin | aucune | 15 m/s, 90 s | 5375 | 60 img/s | 16,8 ms | 54 ms (demarrage) | 0 | 0 ms |
| D, jeu Univers rapide | 9 collections | 50 m/s, 60 s | 3155 | 53 img/s | 19,1 ms | 602 ms (GC) | 10 | 692 ms |

Complement CSV (temps par thread, en ms) : B, 9 images sur 4936 au-dessus de 40 ms (543 ms), game thread max 89, render thread max 86, RHI max 52 ; D, 14 images sur 3150 au-dessus de 40 ms (1360 ms), game thread max 563.

Etat du foliage pendant B : 680 a 800 composants ISM, 0,9 a 1,04 million d instances, worker actif 13 secondes sur 90, sans jamais bloquer le game thread. Le worker prend 9 ms en moyenne par secteur et jusqu a 130 ms pour un secteur d herbe de 15 000 points (GrassTemperate, secteur de 500 m).

Le temoin C reproduit exactement le constat de Benja : meme trajet, liste vide, zero image lente.

## 3. Anatomie d un freeze (mesure B, image la plus lente : 156 ms)

Game thread 97,6 ms, dont 70,2 ms dans le tick du root (`AWorldScapeRoot::Tick`, entierement dans `FoliageHandleTick` d apres Insights) :

- 47,7 ms de "Self" : la boucle de nettoyage des secteurs eloignes et l application des nouveaux secteurs, sans budget de temps. Sur cette image : 84 composants ISM detruits et 141 000 instances retirees d un coup (delta mesure par le marqueur par tick), 258 `UnregisterComponent` (3,6 ms), 92 `UpdateComponentToWorld` avec `CalcBounds` d ISM (8,3 ms).
- 10,4 ms de `Destroy Actor` pour 83 acteurs de foliage Blueprint (52 `WS_Sound_Wind_Forest_C`, 31 `WS_Sound_Bird_Temperate_C`).
- Sur une autre image lente (52 ms) : 27 ms de `SpawnActor` pour 55 acteurs sons, dont 5,4 ms d enregistrement de 34 `AudioComponent`.
- Creation des ISM : `CreateInstancedMesh` 695 appels, 0,7 ms en moyenne, 4 ms au pire ; `AddInstancesInternal` 1015 appels, 1,7 ms au pire, 43 ms cumules sur 90 s. L ajout d instances lui-meme est bon marche : c est le nombre d objets crees et detruits dans la meme image qui coute.

Render thread 156 ms sur la meme image : `PrepareDistanceFieldScene` 62 ms (`UpdateDistanceFieldObjectBuffers` 52 ms, dont 34 ms d upload des donnees et bornes des objets). Les ISM de foliage sont crees avec `bAffectDistanceFieldLighting = bCastShadows`, donc chaque rafale d ajout ou de retrait de primitives reconstruit les buffers d objets du champ de distance. Le game thread attend ensuite le render thread (`EventWait`).

Frequence sur le trajet B : `FoliageHandleTick` depasse 20 ms 7 fois en 90 s (70, 55, 47, 28, 27, 25, 24 ms), a chaque changement de cellule d une famille de secteurs (les grandes familles a 1 km et 500 m produisent les rafales, les petites a 20 a 100 m produisent le bruit de fond a 20 a 35 ms).

## 4. Mesure D, a 50 m/s

- Une rafale de 6 images consecutives entre 50 et 91 ms a t = 6 s : l application d une vague de secteurs s etale sur plusieurs ticks, chacun sature.
- Une image de 602 ms a t = 15 s : garbage collection de l editeur (`FRealtimeGC::PerformReachabilityAnalysis` 64 ms, purge, `FMemory_Trim` 34 ms, 380 ms de Self non instrumente dans la meme image), rendue lourde par le churn du foliage : plusieurs centaines de composants ISM et d acteurs sons crees et detruits par minute. C est le profil des "freezes plus petits et moins frequents" (avec les hitches de compilation de PSO : 50 releves dans ta session de la veille).

## 5. Hypotheses tranchees

1. Attente bloquante du worker au changement de cellule : refutee. Aucun `EnsureCompletion` hors `BeginPlay` et `EndPlay` ; une nouvelle tache n est lancee que si la precedente est finie ; `GameThreadWaitForTask` culmine a 36 ms et correspond aux attentes de fin d image du render thread.
2. Application des resultats sur le game thread : confirmee, cause principale. `FoliageHandleTick` vide toute la file de secteurs et nettoie tous les secteurs eloignes dans le meme tick (le `Timer` declare en tete de fonction n est jamais lu), et les foliages Blueprint spawnent ou detruisent des dizaines d acteurs dans la meme image.
3. Pooling de collision : hors cause en editeur. Aucun evenement `WS_FCP_TickUpdate` dans les traces (le systeme demarre au `BeginPlay`, donc en jeu). A re-mesurer en PIE avec un pawn.
4. SnapSpawner et grille : hors cause. `WS_Grid_Update` culmine a 0,25 ms.
5. Bruit qui sature les coeurs : hors cause pour le freeze. Le worker est asynchrone et n apparait pas dans les attentes du game thread. Sur la Terre de production (24 volumes heightmap, 8 trous) il sera plus lent, ce qui retarde l apparition du foliage mais ne fige pas l image. Porter le bruit du foliage sur le GPU (voies A ou B de la memoire `worldscape-foliage-noise-cost-map`) ne supprimerait pas les freezes mesures.
6. Cause secondaire : la garbage collection amplifiee par le churn d objets, et le render thread (champ de distance).

## 6. Ce qui n a pas ete mesure

- Le PIE avec un pawn (pooling de collision actif, gravite, streaming QLevel).
- La Terre de production `L_Earth` (regle : pas de session dessus) : memes mecanismes, worker plus lent a cause des volumes.
- La vitesse et l altitude reelles du deplacement de Benja dans l Univers.

## 7. Plan propose (a valider par Benja avant tout code)

Par petits pas, chacun mesure avec le meme harnais (trajet B et D, 0 image > 50 ms comme objectif), en gardant tout ce que le foliage fait aujourd hui : densites, masques, clusters, collision par pooling, chemin CPU du serveur dedie.

- P1, budget de temps dans `FoliageHandleTick` : appliquer au plus N secteurs (ou X ms) par tick et garder le reste en file ; etaler de la meme facon le nettoyage des secteurs eloignes (destruction des composants et des acteurs). C est le levier principal, sans changement de donnees.
- P2, acteurs Blueprint : limiter les `SpawnActor` et `Destroy` a quelques-uns par tick (27 ms par rafale de 55 aujourd hui), ou les mettre en pool.
- P3, champ de distance : ne plus poser `bAffectDistanceFieldLighting` sur l herbe et les petits objets (le champ de distance ne sert qu aux ombres DF et a Lumen), ou lisser les creations pour lisser l upload ; a mesurer sur `PrepareDistanceFieldScene`.
- P4, churn et GC : reutiliser les composants ISM d un pool au lieu de `NewObject` puis `DestroyComponent` a chaque secteur ; a defaut, mesurer `gc.TimeBetweenPurgingPendingKillObjects`.
- P5, optionnel et plus tard : bruit GPU pour le foliage. Gain attendu : latence d apparition du foliage, pas fluidite de l image.

Pieges d outillage rencontres, a connaitre pour la suite : l editeur en arriere-plan tombe a 2 images par seconde (`bThrottleCPUWhenNotForeground`, a couper en memoire) ; `set_level_viewport_camera_info` n ecrit que le premier viewport perspective alors que WorldScape lit le viewport actif ; une ecriture `set_editor_property` sur le root a chaque tick declenche `PostEditChangeProperty`, donc `RerunConstructionScripts` et le re-enregistrement des 800 ISM (55 ms par image), d ou l ecriture avec `notify_mode = NEVER` ; le delta de tick Slate est plafonne a 125 ms, il ne mesure pas une cadence sous 8 images par seconde.
