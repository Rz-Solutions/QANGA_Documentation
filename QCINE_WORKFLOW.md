# QCINE : atelier de cinematiques QANGA

> Etat au 2026-09-04 (premiere version, construite et testee dans l editeur par Claude, pilotee par Benja).
> Objectif : produire des plans "triple A" (presentation de cyborgs, vaisseaux, vehicules, orbites, gros plans,
> style Star Citizen) sans savoir se servir de Sequencer, en une ligne de Python ou un appel du pont CLIScape.
> Regle absolue rappelee par Benja le 2026-09-04 : **aucun parametre de rendu du projet n est modifie**
> (pas de `.ini`, pas de Project Settings, pas de scalabilite, pas de post-process du jeu). Tout vit dans
> `Content/QSequences/`, `Content/Python/qcine/` et `Saved/QCine/`. Les CVars de qualite ne s appliquent que
> dans la session de rendu Movie Render Queue et sont restaurees a la fin.

## 1. Ce qui existe deja dans le projet (audit du 2026-09-04)

- **Un studio de reveal de 2023** : `Content/QSequences/Reveal/` (map `L_Studio_Reveal`, cyclo `SM_Studio`,
  7 LevelSequence `SQ_Reveal_Cyborg_*` sur des rails de camera, 4 RectLights, un preset MRQ PNG dont le dossier de
  sortie pointe vers un OneDrive disparu). C est la preuve que MRQ a deja tourne sur ce projet.
- **Un second atelier, plus recent (avril 2025)** : `Content/Maps/DevMap/ConstructionLevel/LEVEL_CAM/`. Il contient
  deux presets Movie Render Queue complets (`RenderSettings_5_3_WILLIAM`, `RENDER_QUEUE_ZOUZIXX` : passe deferred,
  anti-aliasing, **ColorSetting**, **GameOverrideSetting**, sorties **EXR** et JPG, sans chemin externe code en dur),
  une configuration **OpenColorIO** (`OpenColorIO.uasset` : ACES, ACEScg, Linear, sRGB), le plateau in-situ
  `L_CAM_LEVEL.umap` (2 CineCamera, 4 WeatherActor, 3 PostProcessVolume, fonds Brushify, sol `SM_SOL_LEVEL_CAM`) et
  `L_Persistent_Universe_DEV_2.umap` (8 CineCamera, 5 CameraRig_Rail, rendu planetaire complet). Les sequences
  `SANDDIGGER`, `SANGLINEFLY`, `LEVEL_SEQUENCE_Mineral_Processing_Plant_A1` y sont deja montees. C est l etat de l art
  du projet pour les plans "dans le monde" ; `QSequences/Reveal` reste le meilleur exemple de tour de table produit.
- **49 LevelSequence** au total (intro, cryo, premiere mission, lobby saisonnier, `S_INTRO_QANGA`, takes Control Rig
  dans `Content/ControlRig/` et `Content/Developers/xadaa/Takes/`).
- **Plugins actifs** (montes dans le log editeur) : MovieRenderPipeline (tire par `DLSSMoviePipelineSupport`),
  LevelSequenceEditor, SequencerScripting, TemplateSequence, ActorSequence, Takes (TakeRecorder), CameraShakePreviewer,
  PythonScriptPlugin, EditorScriptingUtilities. **Inactifs** : CineCameraRigs (rails splines experimentaux), Composure,
  MoviePipelineMaskRenderPass. Les `CameraRig_Rail` / `CameraRig_Crane` historiques sont dans le moteur et disponibles.
- **Le rendu du jeu tient dans un acteur** : `Content/Systems/Weather/WeatherActor` (soleil, lune, SkyLight, nuages,
  26 PostProcessComponent, kernels de bloom et lens flare). Pour retrouver le look exact du jeu dans un plateau
  exterieur, on le pose tel quel, on ne le modifie jamais.
- **Recette orbitale du jeu** (lue dans `L_Persistent_Universe.umap` et `Lobby_WorldSpace.umap`) : `BP_Starfield`
  (plugin SpaceScape), `WorldScapeRoot` + `PH_Earth` (QangaUnivers), `FarCloudSystem` (AtmoScape), `SkyAtmosphere`,
  `VolumetricCloud`, `QLevelGravityArea`, `WeatherActor`.
- **ffmpeg** est installe (winget, `Gyan.FFmpeg 8.1.2`) : encodage mp4 et planches contact automatiques.
- **Aucun script Python Sequencer n existait** avant cet atelier ; `Content/Python/init_unreal.py` ne fait que
  rapatrier les rapports de bug.

## 2. L atelier livre

```
Content/Python/qcine/            boite a outils Python (editeur)
    __init__.py                  import qcine ; qcine.reload()
    util.py                      log fichier, look-at, bornes d acteur, cadrage par focale
    stage.py                     niveaux de plateau, placement d acteurs, eclairage 3 points, post-process, materiau neutre
    shots.py                     constructeurs de plans : orbit, dolly, crane, flyby, static, rail + animation + fondu
    render.py                    Movie Render Queue (presets preview / hd / final / final_60), image fixe, mp4 + planche
Content/QSequences/
    Stages/L_Stage_Studio        plateau studio (cyclo neutre, 3 RectLights, SkyLight, PostProcessVolume, GameModeBase)
    Shots/SQ_*                   sequences generees (camera spawnable, cuts, focus, animation)
    _Shared/M_QC_Neutral         materiau PBR neutre du cyclo
Saved/QCine/Renders/<job>/       images PNG, status.json, <job>.mp4, <job>_sheet.png
Saved/QCine/Logs/                journaux des rendus
```

### 2.1 Une cinematique en quatre lignes

```python
import qcine
qcine.stage.open_or_create("/Game/QSequences/Stages/L_Stage_Studio")
hero = qcine.stage.place("/Game/Pawn/CyborgV2/Male/CyborgMaleRigged_PROD", "Hero")
seq = qcine.shots.orbit("/Game/QSequences/Shots/SQ_Hero_Orbit", target="Hero", duration=8, focal=50, aperture=2.0)
qcine.shots.add_animation(seq, "Hero", "/Game/Pawn/Human/Male/Animations/NC_Idle"); qcine.shots.save(seq)
qcine.render.render("/Game/QSequences/Shots/SQ_Hero_Orbit", preset="preview")   # 720p, puis "final" = 4K 8 samples
```

Le rendu est asynchrone (MRQ lance une session PIE). `Saved/QCine/Renders/<nom>/status.json` passe par
`queued`, `rendering`, `rendered`, `done` (ou `failed` / `error`). A la fin : `<nom>.mp4` et `<nom>_sheet.png`.
`qcine.render.still(seq, frame)` rend UNE image en une vingtaine de secondes : c est la boucle de look-dev.

### 2.2 Les constructeurs de plans (`qcine.shots`)

| Fonction | Mouvement | Parametres utiles |
|---|---|---|
| `orbit` | cercle autour de la cible, camera toujours pointee dessus | `radius` (auto depuis les bornes), `height`, `start_deg`, `end_deg`, `easing` |
| `dolly` | travelling avant ou arriere sur un azimut fixe | `start_distance`, `end_distance` (auto), `azimuth_deg`, `height` |
| `crane` | grue verticale a distance fixe | `start_height`, `end_height`, `azimuth_deg` |
| `flyby` | passage rectiligne a cote de la cible | `start`, `end` ou `lateral`, `height`, `forward` |
| `static` | plan fixe, zoom optique optionnel | `azimuth_deg`, `elevation_deg`, `distance`, `focal_zoom_to` |
| `rail` | spline Catmull-Rom par points | `points=[(x,y,z), ...]` |

Tous acceptent `focal` (mm), `aperture` (f), `sensor` ((36, 24) plein format, (36, 15) pour du 2.39:1),
`fps`, `look_offset` (decalage du point vise, par exemple la tete), `bounds_mode` (`mesh` par defaut : les
bornes des seuls composants mesh, jamais les rayons de lumiere ni les FX, cf. le piege mesure sur le Velkara).
La camera est un **spawnable** de la sequence (aucun acteur camera ne reste dans le niveau), le mouvement est
**cuit une cle par image** (interpolation lineaire, pas de depassement de tangente, yaw continu) et la mise au
point manuelle suit la cible image par image.

Complements : `add_animation(seq, acteur, AnimSequence)` (piste squelettique), `add_fade(seq)` (fondu noir),
`describe(path)` (resume lisible des pistes et cles).

### 2.3 Les plateaux (`qcine.stage`)

- `open_or_create(path)` : charge ou cree un niveau vide, sauve immediatement.
- `set_game_mode(None)` : World Settings sur `GameModeBase` (aucune pile gameplay QANGA pendant le rendu).
- `place(asset, label, location, yaw, ...)` : Blueprint, StaticMesh ou SkeletalMesh, avec un label stable.
- `studio_lighting(center, distance, key_lumens, fill_lumens, rim_lumens, sky_intensity)` : key chaud a 135 degres,
  fill froid, rim bleute, SkyLight faible. Labels `QC_Key`, `QC_Fill`, `QC_Rim`, `QC_Sky`.
- `post_process(exposure_ev, bloom, vignette, grain, lens_flare, motion_blur)` : PostProcessVolume non borne,
  priorite 10, exposition manuelle. Look "neutre haute fidelite", aucun virage colorimetrique, aucun LUT.
- `neutral_material()` + `set_material(acteur, materiau)` : cyclo gris PBR.
- `sun(pitch, yaw, lux, color, atmosphere)` : DirectionalLight (+ SkyAtmosphere optionnelle) pour l exterieur.

### 2.4 Les presets de rendu (`qcine.render.PRESETS`)

| Preset | Resolution | Samples temporels | Usage |
|---|---|---|---|
| `preview` | 1280x720 | 1 | boucle de travail, contact sheet |
| `hd` | 1920x1080 | 4 | validation |
| `final` | 3840x2160 | 8 | livraison |
| `final_60` | 3840x2160 @60 | 8 | livraison 60 images par seconde |

Reglages appliques dans le job seulement : `MoviePipelineGameOverrideSetting` (GameMode MRQ, LOD 0, HLOD off,
ombres haute qualite, distance de vue x50, flush du streaming), TSR force par `MoviePipelineAntiAliasingSetting`,
32 images de chauffe. Les methodes de GI, de reflexion et d anti-aliasing du projet ne sont pas touchees
(`Documentation/Lighting/RzZz_Tested_Lighting_Settings.txt` reste l autorite).

## 3. Resultats mesures le 2026-09-04

- Plateau `L_Stage_Studio` cree, sauve, rendu : premiere orbite de 180 images en 720p, 333 s dont environ 4 min de
  compilation de shaders (premiere session PIE du niveau), puis 2 a 3 images par seconde.
- Image fixe de look-dev : 20 s par image la premiere fois, 12 s ensuite (demarrage PIE + 32 images de chauffe compris).
- Quatre iterations de look sur le cyborg masculin PROD, jugees sur image (pas sur "ca compile") :
  v1 cyclo `WorldGridMaterial` + exposition auto = tout crame ; v2 cyclo gris 0.12 + exposition manuelle EV 0 = trop
  sombre ; v3 EV 2.5 + lumieres 40k / 12k / 45k lumens calees sur l azimut camera = lisible ; v4 (retenu) : sol 0.05
  roughness 0.35 (flaque de lumiere reflechie), key 40k chaud, fill 8k, rim 60k bleute, SkyLight 0.12, EV 2.0, bloom
  0.3, vignette 0.35, grain 0.02.
- Lecon de mise en scene : une camera qui orbite autour d un rig de lumieres fixe passe dans l ombre du key pendant
  la moitie du tour. Pour un tour de table, c est le HEROS qui tourne (`qcine.shots.turntable`, cles de yaw par
  image sur sa piste transform) et les lumieres sont construites relativement a la camera
  (`studio_lighting(camera_azimuth_deg=...)`).
- Le visage du cyborg PROD sort blanc : sa peau vient du systeme de customisation en jeu (materiaux poses au
  runtime), pas du SkeletalMesh. A habiller avec les instances de skin pour une vraie presentation.
- Piege MRQ mesure : `{sequence_name}` se resout en chaine vide sur un job sans shot, ce qui produit des fichiers
  `.0001.png` (fichiers caches) que `glob` ignore. Les images sont nommees par le nom du job.
- Piege Sequencer mesure : `add_spawnable_from_class(CineCameraActor)` donne un gabarit dont la liaison de
  composant est morte (`add_track` rend `None`). La conversion depuis une instance de niveau
  (`add_spawnable_from_instance`) marche, l instance est detruite ensuite, et la liaison fantome laissee par la
  conversion est retiree.
- Piege Python 5.7 : `get_cine_camera_component()` rend `None` sur un gabarit de spawnable ; la propriete
  `camera_component` fonctionne. `PrimitiveComponent.get_local_bounds` n existe pas ; `SystemLibrary.get_component_bounds` si.

## 4. Le pont CLIScape : ce qu il y a dans le ventre (audit du 2026-09-04)

25 outils lus dans `Plugins/CLIScape/Source/CLIscape/Private/Tools/` (Sequencer, Rendering, CameraShake, Level,
VirtualProduction, SceneCapture, EditorUtility). Le serveur MCP est un exe C++ (`Source/CLIscapeMCP`) qui
fabrique les schemas par analyse du texte des `.h` : un parametre documente peut n etre jamais lu par le `.cpp`.

| Outil | Etat | Point cle |
|---|---|---|
| `create_cinematic_shot` | complet | le seul qui SAUVE l asset ; 24 ips en dur, label camera non unique |
| `create_level_sequence` | partiel | jamais sauve sur disque, ips entier seulement, echec si le package existe |
| `bind_actor_to_sequence` | complet | possessables d acteurs par label ; pas de composant, pas de spawnable |
| `add_transform_track` | complet | seul outil qui honore `interp` ; une seule section, rotations non deroulees |
| `add_property_track` | partiel | float seulement, piste dupliquee a chaque appel, `PropertyName` recoit le chemin complet |
| `add_camera_cut_track` | complet | DESTRUCTIF : efface tous les cuts a chaque appel |
| `set_keyframes` | partiel | `interp` documente mais ignore (cubique toujours) |
| `get_sequence_summary` | partiel | pas de compte de cles ni d info de camera cut malgre la description |
| `open_sequence`, `viewport_control`, `manage_post_process`, `configure_light`, `manage_scene_capture`, `manage_atmosphere` | complets | limites mineures listees dans l audit |
| `manage_sequencer_tracks` | partiel | audio, fade, animation squelettique reels ; event et particle decoratifs ; `volume` ignore |
| `manage_movie_render_queue` | complet mais aveugle | rendu reel, aucun retour de fin ni d erreur, toujours le dernier job, images seulement |
| `manage_take_recorder` | partiel | aucun `start_recording` possible (API 5.5+) |
| `manage_camera_shake`, `manage_composure`, `manage_virtual_production` | stubs | reponses `success:true` fabriquees, parametres jamais lus |
| `execute_python_script` | complet | motifs bloques : `open(`, `exec(`, `subprocess`... ; refuse pendant PIE |

**Bug de pont mesure le 2026-09-04 (a corriger, priorite haute)** : quand la socket client tombe (`10053`),
le listener a re-execute le **payload de la requete precedente** (dispatch de 18:25:07 dans `Saved/Logs/QANGA.log` :
le job `job_render_preview.py` envoye, `job_shots_test.py` execute). Une mutation peut donc etre rejouee a l insu
du client. Parade appliquee dans l atelier : chaque job verifie un ticket ecrit par l appelant et refuse de tourner
si le ticket ne le nomme pas (`scratchpad/qjob.py`). A investiguer dans `MCPListenerThread.cpp`
(`ReceiveJsonPayload` et la conversion `UTF8_TO_TCHAR` sur un tampon non termine).

**Regle d usage du pont pendant un rendu** : ne jamais appeler le pont pendant une session de rendu MRQ (PIE). Un
appel dispatche a ce moment (mesure : `get_recent_engine_errors` a 21:07:50) n est execute qu a la fin du rendu et
tient la connexion unique du listener pendant tout le rendu (connexion en `CLOSE_WAIT` 5 minutes) : tout autre
client recoit `10053`. Le runner de l atelier reemet toutes les 5 s avec un ticket anti-rejeu jusqu au dispatch.

**Pourquoi l atelier est en Python et pas dans les outils C++** : la voie Python couvre tout (spawnables, cles par
image, focus, fondu, MRQ avec rappel de fin, encodage), elle ne demande aucune recompilation, et les outils C++
actuels n enregistrent pas sur disque. Les corrections C++ proposees (a valider par Benja avant toute compilation) :
1. sauver l asset apres chaque mutation Sequencer (ou parametre `save`) ;
2. `set_keyframes` : lire `interp` ; `add_property_track` : chercher la piste existante et ne passer que le dernier
   segment du chemin dans `PropertyName` ;
3. `create_cinematic_shot` : parametre `frame_rate`, label de camera unique, yaw deroule ;
4. `manage_movie_render_queue` : rappel de fin (`on_executor_finished_delegate`), chemin de sortie renvoye,
   selection du job, sortie mp4 (`MoviePipelineMP4EncoderOutput` existe dans le moteur 5.7) ;
5. un outil `qcine_*` (ou `execute_python_script` allege) qui appelle directement `qcine` pour exposer
   `orbit / dolly / render / still` au chat in-editor.

## 5. Recettes de plateau

### Studio (livre)
`L_Stage_Studio` : `SM_Studio` (cyclo du reveal 2023) avec `M_QC_Neutral`, `QC_Key` / `QC_Fill` / `QC_Rim` (RectLights
lumens), `QC_Sky` (capture temps reel, hemisphere bas non noir), `QC_PostProcess` (exposition manuelle). Heros
`Hero` = `CyborgMaleRigged_PROD` avec `NC_Idle` (l AnimSequence de la pose de customisation `AM_CustomizationPoseLoop`).

### Orbite (construit le 2026-09-04, look NON resolu)
Etat honnete : le niveau, le vaisseau, les plans (`SQ_Ship_Orbit`, `SQ_Ship_Flyby`) et le cadrage sont bons, mais
l image rendue est un fond blanc uniforme, une planete en disque noir et un vaisseau en silhouette, quel que soit
l exposition (EV -5 a +4), avec ou sans le champ d etoiles, avec ou sans le vaisseau (5 images de diagnostic dans
`Saved/QCine/Renders/SQ_Ship_Orbit_*`). Faits mesures : `BP_Starfield` expose une variable d acteur `Brightness`
(1.0) et `SunAngularDiameter` (0.54), son materiau est un `MID_M_Starfield_0` (parametres `Brightness` 0.02,
`Size`, `SunAngularDiameter`, `Sun Brightness` 1.0, `RotationSpeed`) recree par le script de construction a chaque
session PIE, donc une modification du MID en editeur ne survit pas au rendu. Le fond blanc ne vient pas de lui
(identique acteur cache). Piste la plus probable : ces acteurs sont concus pour l exposition et les parametres que
le `WeatherActor` leur pousse en jeu ; la bonne base pour un plateau orbital est donc la recette complete de
`Lobby_WorldSpace.umap` (WeatherActor inclus, jamais modifie) plutot qu un eclairage manuel. A reprendre.
`L_Stage_Orbit` : `BP_Starfield` (SpaceScape, sphere de 16 km de rayon autour de l origine), sphere
`SM_NonVolumetricSphereMHighRes1` (rayon brut 1 m) avec `M_Planet_Inst` (Jupiter) mise a l echelle 20 000 et posee
a 60 km, `QC_Sun` (6 lux), `QC_PlanetBounce` (0.4 lux chaud), SkyLight 0.05, `QC_PostProcess` EV 0, vaisseau
`Velkara_Explorer` (le vrai Blueprint de gameplay) physique et gravite coupees sur tous les composants primitifs.
Faits mesures au placement du vaisseau :
- ses bornes "mesh" font 85 m x 58 m a cause de `SM_AtmosphereEntry_FX` (le maillage du plasma de rentree,
  demi-etendue 42 x 29 x 29 m) et du widget 3D `VehicleHud3dWidget_ShipControlHud`, contre une coque reelle de
  19,4 x 16,2 m (bornes collisionnantes, extent 972 x 811 x 800) : le cadrage des plans de vaisseau utilise
  `bounds_mode="colliding"` (cf. le piege deja paye sur la livraison de flotte) ;
- le Blueprint cree deux acteurs annexes des le placement en editeur (`ShippingLandingArea0`,
  `QAudio_Filter_Volume0`), qui restent dans le niveau de plateau ;
- le spawn a ete decale par la gestion de collision (le vaisseau n est pas a l origine exacte) : toujours relire
  ses bornes apres placement plutot que de supposer sa position.

### Surface planetaire (a faire, la base existe)
Le plateau existe deja : `Content/Maps/DevMap/ConstructionLevel/LEVEL_CAM/L_CAM_LEVEL.umap` (WeatherActor, PostProcess,
fonds lointains, sol dedie). `qcine.stage.open_or_create` le charge tel quel ; on y place le heros avec `place` et on
construit les plans avec `qcine.shots`. Alternatives : `Content/Maps/LevelDev/L_EmptyScene.umap` (WeatherActor +
SkyAtmosphere + gravite, presque rien d autre) ou `L_Dev_Claude`. Ne jamais modifier le WeatherActor ni ces niveaux
d equipe : dupliquer d abord (`EditorAssetLibrary.duplicate_asset`) vers `Content/QSequences/Stages/`.

## 6. Ce qu il reste a faire

- Look studio : regler exposition et lumieres sur les vrais heros (masculin PROD, feminin, armes), ajouter une
  variante fond noir "Star Citizen" et une variante fond blanc.
- Plateau orbite : valider que le Blueprint de vaisseau reste stable en PIE sans joueur (physique gelee), sinon
  utiliser la version `_NoDrive` ou le StaticMesh.
- Plateau surface avec le WeatherActor.
- Presets MRQ en assets (`MoviePipelinePrimaryConfig`) pour l interface MRQ, si Benja veut rendre a la main.
- Un panneau Editor Utility Widget "QCine" (boutons orbite / dolly / rendu) pour l usage sans code.
- Corrections C++ du pont listees en section 4, apres validation.
