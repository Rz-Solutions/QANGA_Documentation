# QFootprint : empreintes de pas sur le sol des planetes

Livre le 2026-09-11 (session Claude, compile et valide en PIE sur `L_Dev_Claude`), etendu aux autres planetes le 2026-09-16
(section 8 : profils de planete + normale des decals sur `M_EarthBase` et `M_Europe`). Cadrage et mesures d origine :
`Documentation/QFOOTPRINT_DESIGN_PROPOSAL.md`. Captures : `Documentation/QFootprint/`.

## 1. Ce que ca fait

Quand un humanoide (joueur ou IA partageant `ALS_AnimBP`) pose le pied au sol, un decal d empreinte est projete a
l endroit du contact, oriente dans le sens de la marche, avec le relief de la semelle (normale) et une teinte plus
sombre que le sol. Il s efface au bout de `LifeTime` + `FadeTime`. Tout est **cote client, cosmetique** : aucune
replication, aucun RPC, aucun etat de gameplay, rien sur le serveur dedie.

## 2. Chaine complete

```
Animation ALS (55 anims portent deja Footstep_AnimNotify)
  -> Footstep_AnimNotify.Received_Notify (Blueprint existant, inchange pour le son)
       premier noeud ajoute : QFP On Footstep (MeshComp, AttachPointName, FootstepType)
  -> UQFootprint_FunctionLibrary::QFP_OnFootstep (C++)
  -> UQFootprint_SubSystem::OnFootstep
       gardes : subsystem client seulement, qfootprint.Enabled, type de pas retenu (Step/WalkRun/Land),
                mesh rendu recemment, IA autorisee, distance camera < MaxDistance(AI)
       trace : 1 ligne le long de -Up du pawn (NinjaCharacter, gravite arbitraire), canal Visibility
       matiere : WorldScape -> pente vs radial, puis profil de planete (asset de bruit du root) :
                 Terre (bUseClimate) -> GetGroundNoise(ECEF) + GetPawnAltitude -> sable, neige, terre ;
                 autre planete -> style du profil (teinte noire) ; planete non listee -> UnknownPlanetStyle ;
                 hors WorldScape -> rien (v1)
       anti doublon : meme pied a moins de MinStepDistance = rien
  -> UQFootprint_SubSystem::PlaceFootprint (QFP Add Footprint passe par AddFootprint -> PlaceFootprint avec le style de la matiere)
       style de la matiere (Terre) ou du profil de planete (teinte, intensite, taille), rotation MakeFromXZ(-normale, avant)
  -> AQFootprint_Manager::Place : slot du ring (gauche ou droit), SetWorldTransform, SetDecalColor,
       SetFadeOut(LifeTime, FadeTime) puis SetLifeSpan(0) (le pool garde ses composants), SetVisibility
  -> rendu : UDecalComponent DBuffer, materiau M_QFootprint (masque + normale du kit Footprints_01)
  balayage 1 Hz : HideExpired (un decal dont le fondu est fini coute encore un draw sinon)
```

## 3. Fichiers

| Element | Chemin |
|---|---|
| Plugin | `Plugins/QFootprint/QFootprint.uplugin` (Runtime, depend du plugin WorldScape) |
| Module | `Plugins/QFootprint/Source/QFootprint/` : `QFootprint.h/.cpp` (log `LogQFootprint`), `QFootprint_Types.h`, `QFootprint_Settings.h/.cpp`, `QFootprint_Manager.h/.cpp`, `QFootprint_SubSystem.h/.cpp`, `QFootprint_FunctionLibrary.h/.cpp`, `Private/Tests/QFootprint_Tests.cpp` |
| Materiau | `Content/Systems/Footprint/M_QFootprint` (Deferred Decal, Translucent = DBuffer), instances `MI_QFootprint_L` (MirrorU 0) et `MI_QFootprint_R` (MirrorU 1) |
| Textures | reutilisees par reference : `Plugins/Qasset/Content/AssetStore/AbandonedFactory/Effects/Footprints_01/t_Footprint_01_01_Mud_m` (masque, empreinte = 1 - R) et `t_Footprint_01_01_Mud_n` (normale) |
| Notify modifie | `Content/Systems/Character/Blueprints/AnimNotifys/Footstep_AnimNotify.uasset` : noeud `QFP On Footstep` insere entre l entree et le premier `Is Valid` (backup md5 `45ee8be1...` dans le scratchpad de session) |
| Config | `Config/DefaultGame.ini` section `[/Script/QFootprint.QFootprint_Settings]` + `+DirectoriesToAlwaysCook=(Path="/Game/Systems/Footprint")` (les instances sont des soft refs C++, invisibles au cooker) |
| Projet | `QANGA.uproject` : plugin `QFootprint` active |
| Materiau Terre (D1) | `Content/Resources/MasterMaterial/M_EarthBase_OPT` : `MaterialDecalResponse` = `ColorNormalRoughness` (sauve par Benja le 2026-09-11) |
| Materiaux des autres planetes (2026-09-16) | `M_EarthBase` (instances `Mi_MarsMat1` Mars, `Mi_EarthMat2` du sous-niveau `L_Earth`, `Mi_EarthMat1/4/DEV/Atelier`, `Mi_forSM_Ground_Earth`) et `M_Europe` (`Mi_IO` des 14 lunes, `Mi_Europe2` Europe, `Mi_Europe`) : `MaterialDecalResponse` = `ColorNormalRoughness`, sauves par la session. Correction de la ligne precedente : `M_EarthBase` n etait PAS inutilise (Mars et `L_Earth` le portent). Backups md5 `175d764f` et `b3c216df` dans le scratchpad de session `backup_2026-09-16/` |

## 4. Reglages (DefaultGame.ini, section QFootprint) et CVars

- Activation : `bEnabled`, `bEnableForAI`, `PrintFootstepTypes` (0 Step, 1 Walk/Run, 3 Land ; 2 Jump exclu), `LandFootstepType`, `LandIntensityMultiplier`, `LeftFootSocketSuffix` (`_L`).
- Budget : `MaxDecals` 256 (moitie gauche, moitie droite, jamais plus), `LifeTime` 60 s, `FadeTime` 10 s, `FadeInTime`, `MaxDistance` 6000 cm, `MaxDistanceAI` 3000, `RecentlyRenderedTolerance`, `FadeScreenSize` 0.005, `MinStepDistance` 10 cm, `SweepInterval` 1 s.
- Placement : `TraceUp` 15, `TraceDown` 40, `MaxSlopeDeg` 35, `SurfaceOffset` -15 (la boite de projection est enfoncee sous le sol : avec une profondeur de 20 elle couvre -35 a +5 cm, donc le cyborg qui repasse sur ses traces n est pas peint), `DecalSize` (20, 12.5, 12) demi-etendues cm (l axe V du decal tombe sur Y et U sur Z ; la chaussure du kit occupe 79 pour cent de V et 31 pour cent de U, centree, mesure par lecture des pixels du masque le 2026-09-12 : une boite de 25 x 24 cm donne un pied de 20 x 7.5 cm, la taille de la botte du cyborg validee par Benja ; pour agrandir, garder le rapport Y / Z), `TextureYawOffset` -90 (l offset tourne toute la boite autour de l axe de projection ; -90 met la pointe vers l avant, mesure en jeu par Benja), `MaterialLeft` / `MaterialRight`.
- Pieds : `bResolveFootFromBones` true, `LeftFootBone` foot_l, `RightFootBone` foot_r, `LeftBallBone` ball_l, `RightBallBone` ball_r, `FootCenterAlpha` 0.55 (centre de l empreinte entre cheville et avant-pied), `LeftFootSocketSuffix` `_L`, `RightFootSocketSuffix` `_R`. Le socket du notify vaut `root` sur les animations de marche ALS (mesure 2026-09-11) : sans cette resolution, toutes les empreintes tombent sous le bassin en ligne droite. Le suffixe du socket decide le cote quand il est present, sinon le pied le plus bas le long de l axe haut du pawn ; la direction de la chaussure = cheville vers avant-pied.
- Materiau : parametre `MaskBoost` 1.8 (durcit la semelle : opacite = saturate((1 - R) x MaskBoost)).
- Matiere (bruit WorldScape) : `MinAltitudeCm` -50 (eau), `BeachAltitudeMaxCm` 3000 (sable de plage), `DesertTemperatureMin` 0.6 et `DesertHumidityMax` 0.35 (sable de desert), `SnowTemperatureMax` 0.25, sinon terre ; `bPrintOnNonTerrain` false (sols de station, vaisseaux : rien en v1).
- Styles : `SandStyle`, `SnowStyle`, `SoilStyle`, `MudStyle` = teinte lineaire (plus sombre que le sol), intensite (alpha du DecalColor), echelle. Valeurs livrees mesurees sur le desert de L_Dev_Claude.
- Planetes (2026-09-16) : `PlanetProfiles`, un profil = `Name`, `Noises` (assets de bruit de la planete, `AWorldScapeRoot::WorldScapeNoise`), `bUseClimate`, `Style`. Livre : `Earth` (`PlanetEarth`, `EarthV2`, climat Terre et styles ci-dessus) ; `Moon` (`TheMoon`), `Mars` (`BarenWorldMaterial/Mars`), `Venus` (`BarenWorldMaterial/Venus`), `SmallMoons` (`VenusNoise`, les 15 lunes Phobos, Deimos, Io, Europe, Titan...) = teinte noire, intensite 0.6, pas de climat. `UnknownPlanetStyle` (meme style) pour toute planete non listee. La liste du `.ini` REMPLACE le defaut C++ : garder les deux alignes. L intensite se regle par planete dans le `.ini`, sans rebuild.
- CVars : `qfootprint.Enabled` (1), `qfootprint.AI` (1), `qfootprint.LifeTimeScale` (1.0), `qfootprint.Debug` (0, hors Shipping : trace dessine + une ligne de log par pas avec la matiere, T, H, altitude, pente).
- Facade Blueprint : `QFP On Footstep`, `QFP Add Footprint` (tout emetteur : vehicule, quete, creature), `QFP Is Active`, `QFP Get Stats` (pool, visibles, poses).

## 5. Faits mesures a connaitre avant de toucher

- `GetGroundNoise` attend une position dans le repere de la planete : `Root->GetActorTransform().InverseTransformPositionNoScale(Monde)`. Une position monde donne un climat faux (le root de la Terre est tourne).
- Le terrain WorldScape n a aucun physical material : la matiere vient du bruit + de la pente (`ImpactNormal` contre la direction radiale).
- Les notifies tournent aussi hors ecran et sur serveur (`VisibilityBasedAnimTickOption = AlwaysTickPoseAndRefreshBones` sur `ALS_Base_CharacterBP_C` et `AI_Cyborg_C`) : les gardes `WasRecentlyRendered` et distance sont indispensables.
- Sur les animations de saut et d atterrissage, `AttachPointName` vaut `root` (pas un pied) : l empreinte est posee sous le bassin et compte comme pied droit. Acceptable pour un atterrissage.
- Sans D1 (`MaterialDecalResponse` avec Normal sur `M_EarthBase_OPT`) l empreinte n a pas de relief sur le terrain : tache plate.
- Materiau : un decal DBuffer avec seulement BaseColor + Opacity (sans normale) ne compile pas sur ce projet (Substrate) ; relief seul (sans BaseColor) est presque invisible sur le sable ; la version livree ecrit BaseColor + Normal + Opacity, pas la rugosite.
- `UDecalComponent::SetFadeOut` arme un timer qui detruirait le composant : `SetLifeSpan(0)` juste apres, sinon le pool se vide.
- L identite d une planete = l asset de bruit de son root (`AWorldScapeRoot::WorldScapeNoise`, reference d asset, jamais dupliquee au runtime ; meme cle que `QSystem_AchievementSubsystem`). Mesure en PIE : le root de `L_Dev_Claude` rend le package `/Game/Resources/NoiseWorldscape/PlanetEarth`.
- Une teinte COLOREE dans `DecalColor` ne rend pas la couleur demandee (mesure 2026-09-16, meme camera, pixels lus) : la teinte sable (0.06, 0.045, 0.03) sort beige sur le sable clair, orange sur le sol lunaire, kaki sur Mars ; la teinte neige sort blanc lumineux partout. La teinte NOIRE est previsible : le sol local est fonce en gardant sa couleur (luminance lineaire mesuree x 0.38 a alpha 0.6, x 0.18 a 0.8). C est pour cela que les planetes utilisent le noir.
- Relief (normale) seulement si le maitre du terrain accepte la normale des decals. D origine : `M_EarthBase_OPT` (Terre de l Univers), `Moon` (Lune), `BarenWorldMasterMaterial` (Venus). Passes le 2026-09-16 : `M_EarthBase` et `M_Europe` (statistiques 1487 / 230 echantillons -> 1502 / 231, soit exactement `M_EarthBase_OPT` ; 1470 / 110 -> 1485 / 111). Aucun autre ecart : memes 905 noeuds et memes parametres entre `M_EarthBase` et `M_EarthBase_OPT`.
- Un `.uplugin` avec `"PlatformAllowList": []` n est jamais compile en 5.7 (liste vide = aucune plateforme) : ne pas copier ce champ depuis QTriggerZone.

## 6. Validation faite le 2026-09-11

- Build `QangaEditor` : `Result: Succeeded`, 0 warning sur le module.
- PIE `L_Dev_Claude`, pawn teleporte sur le desert (T 0.85, H 0.00, altitude 287 m, pente 13 deg) : `pool ready 256`, atterrissage reel via le notify -> `matter 1 (Sand)`, 6 appels scriptes -> 2 poses puis 4 refus anti doublon, `stats pool=256 visible=4 placed=4`, teardown `placed 4, rejected 0`.
- Materiau verifie a l oeil dans le monde editeur (probe de decals sur le sable, cote a cote avec le decal du kit).
- **Fuite : mesuree, aucune** (2026-09-12 00:36) : apres deux sessions PIE de Benja (`subsystem down (placed 91, rejected 0)` puis `(placed 210, rejected 0)`), `obj gc` puis `obj list class=QFootprint_Manager` rend `0 Objects` et aucun `DecalComponent` n a un `QFootprint_Manager` pour outer. Chaine de vie : manager spawne `RF_Transient` et detruit dans `Deinitialize` ; decals `UPROPERTY(Transient)` du manager, crees par `NewObject` + `RegisterComponent` (donc possedes par l acteur, detruits avec lui) ; `SetLifeSpan(0)` apres `SetFadeOut` pour annuler le timer d autodestruction du composant ; timer de balayage en `CreateWeakLambda` et `ClearTimer` a la fermeture ; cache anti doublon purge des acteurs morts a chaque balayage.
- **Shipping** : tout le debug (CVar `qfootprint.Debug`, `DrawDebug*`, log par pas) est sous `#if !UE_BUILD_SHIPPING` ; restent des logs `Log` (compiles hors Shipping) et deux `Warning` legitimes (manager non spawne, materiau manquant). Verification par lecture des gardes, pas par un build Shipping.
- **Revue "package" du 2026-09-12** (statique, sans build Shipping : l editeur de Benja tournait) : modules dependants `WorldScapeCore` / `WorldScapeNoise` de type Runtime (donc presents dans les cibles Game et Server) ; aucun include ni module editeur dans le plugin ; tests sous `WITH_DEV_AUTOMATION_TESTS` ; le noeud ajoute au notify a un GUID (`Tool_ManageBlueprintGraph.cpp:991` appelle `CreateNewGuid`, puis compile + sauvegarde) ; section ini parsee sans warning au boot ; dossier force-cooke existant. Deux durcissements sans changement de comportement, a recompiler avec le prochain build : plus de variable assignee mais lue seulement en debug (`UsedBone`, pour Clang `-Wunused-but-set-variable` cote Linux), et include explicite de `Engine/EngineTypes.h` pour `FTimerHandle`. Aucun log de packaging QANGA present sur cette machine : les erreurs recurrentes de Benja au packaging restent a lui demander en texte.
- **Valide par Benja en jeu le 2026-09-12** apres trois retours corriges le meme soir (pied resolu depuis les os, pointe vers l avant, boite enfoncee sous le sol, teinte plus sombre, taille de la botte) : "la pointe est dans le bon sens, ca marche", "bon pour le ratio", "la taille est nickel".
- Tests headless `Automation RunTests StartsWith:QATS.QFootprint` : 2 succes, 0 echec (`Manager.PoolIsFixed` : un ring de 8 slots absorbe 40 poses sans grandir, tout visible puis tout cache apres LifeTime + FadeTime ; `Settings.MaterialsResolve` : les deux instances se chargent). Rapport `Saved/QATSAutomation_QFootprint/index.json`.

## 7. Limites et suite possible

Planetes : un seul style par planete (pas de biome), meme intensite 0.6 partout, reglable par profil. Les styles Terre sable / terre /
neige, eux, sortent plus clairs que le sol sur un fond sombre et blancs pour la neige (planche 2026-09-16) : sans effet sur le desert
valide, a revoir si Benja voit des empreintes claires sur les sols sombres de la Terre ou sur la neige.
Pas de deformation reelle du sable, pas d eclaboussures (v2 Niagara sur le kit `ns_Footprint_01_01_Splash*`), pas d empreintes hors terrain WorldScape (table `EPhysicalSurface` a realigner d abord, les `PM_*` marketplace portent des index d une autre table), pas d animaux ni de Sanglantines (autres notifies, meme noeud a ajouter), pas de vehicules (antigravite). Persistance : aucune, le pool meurt avec le monde.

## 8. Extension aux autres planetes (2026-09-14 / 16)

Demande de Benja : empreintes sur la Lune, Mars, Venus et toutes les planetes WorldScape, en accord avec le materiau de chaque planete
("pas de traces jaunes sableuses sur la Lune, pas de traces blanches comme sur la Lune sur Mars"), sans rien casser ; puis "si tu vois
des lacunes sur les materiaux des autres planetes comparees a la Terre, remets-les a niveau".

Probleme mesure : avant, tout root WorldScape passait par les regles de la Terre (bruit, altitude, desert, neige). Sur une autre planete
le bruit rend d autres valeurs : empreintes sable, neige ou terre selon la region, ou rien sous le "niveau de la mer".

Inventaire mesure (registre d assets et niveaux `Content/_QLevel/Universe/Planets`) :

| Planete | Bruit (identite) | Materiau du terrain | Maitre | Relief avant | Relief apres |
|---|---|---|---|---|---|
| Terre (Univers persistant, `L_Dev_Claude`) | `NoiseWorldscape/PlanetEarth` | `Mi_EarthMat2_OPT` | `M_EarthBase_OPT` | oui | oui |
| Terre (sous-niveau `L_Earth` / `Q_L_Earth`) | `NoiseWorldscape/PlanetEarth` | `Mi_EarthMat2` | `M_EarthBase` | non | oui |
| Lune (`L_Moon`) | `NoiseWorldscape/TheMoon` | `Moon_Inst` | `Moon` | oui | oui |
| Mars (`L_Mars`) | `BarenWorldMaterial/Mars` | `Mi_MarsMat1` | `M_EarthBase` | non | oui |
| Venus (`L_Venus`) | `BarenWorldMaterial/Venus` | `Venus_inst` | `BarenWorldMasterMaterial` | oui | oui |
| 14 lunes (Phobos, Deimos, Io, Callisto, Ganymede, Triton, Encelade, Minas, Tethys, Titan, Miranda, Oberon, Setebos, Titania) | `NoiseWorldscape/VenusNoise` | `Mi_IO` | `M_Europe` | non | oui |
| Europe (`L_Europe`) | `NoiseWorldscape/VenusNoise` | `Mi_Europe2` | `M_Europe` | non | oui |

`L_Mercury`, `L_Jupiter`, `L_Saturn`, `L_Uranus`, `L_Neptune` du dossier `_QLevel` n ont pas de root WorldScape (0 occurrence) : rien a
faire ; si l un en recoit un, `UnknownPlanetStyle` s applique.

Livre :
- C++ (`QFootprint_Types.h`, `QFootprint_Settings.h/.cpp`, `QFootprint_SubSystem.h/.cpp`, test) : `FQFootprint_PlanetProfile`,
  `PlanetProfiles`, `UnknownPlanetStyle`, `FindPlanetProfileIndex` ; `ClassifyHit` regarde le profil apres la pente ; la Terre garde
  exactement son chemin (bruit, altitude, eau, plage, desert, neige, terre) ; les autres planetes sautent le bruit (moins cher) et
  prennent le style de leur profil ; `PlaceFootprint` factorise la pose (le Blueprint `QFP Add Footprint` est inchange).
- `DefaultGame.ini` : `UnknownPlanetStyle` et 5 `+PlanetProfiles` (miroir du defaut C++).
- Materiaux : `M_EarthBase` et `M_Europe` en `ColorNormalRoughness` (seule propriete changee ; dependances identiques avant / apres,
  67 et 62 packages ; relus depuis le disque).

Validation :
- Build `QangaEditor` 2026-09-14 : `Result: Succeeded`, module QFootprint recompile sans warning.
- Tests headless 2026-09-16 `Automation RunTests StartsWith:QATS.QFootprint` : 3 succes (`Manager.PoolIsFixed`,
  `Settings.MaterialsResolve`, nouveau `Settings.PlanetProfiles` : l ini est lu, les 5 bruits attendus ont leur profil et leur mode,
  chaque bruit liste existe sur disque, aucun style de planete n est la teinte sable ou neige). EXIT CODE 0.
- PIE `L_Dev_Claude`, subsystem reel : atterrissage via le notify -> `matter 1 (T 0.85 H 0.00 W 0.00 alt 28715cm slope 13.1)` (Terre
  inchangee) ; profil Terre passe sans climat en memoire -> `planet Earth (/Game/Resources/NoiseWorldscape/PlanetEarth)`, decal pose
  (0, 0, 0, 0.6) ; aucun profil -> `planet unknown`, decal (0, 0, 0, 0.33) avec `UnknownPlanetStyle` a 0.33 ; retour aux profils
  d origine -> sable (0.06, 0.045, 0.03, 1.0) ; `subsystem down (placed 5, rejected 0)` ; apres GC : 0 manager hors objet par defaut,
  0 decal de pool. CDO restaure, `DefaultGame.ini` identique (md5).
- Visuel : planches `Documentation/QFootprint/pl_planets_prints_2026-09-16.jpg` (6 materiaux, teintes Terre contre teinte noire) et
  `pl_relief_before_after_2026-09-16.jpg` (Mars, Io, Europe avant / apres la normale des decals). Methode : materiau du terrain de
  `L_Dev_Claude` remplace en memoire, captures hors ecran (voir 8.1).

Non verifie : le rendu sur les vraies planetes en jeu (eclairage propre a chaque planete, sol reel) : a regarder par Benja en
marchant sur la Lune, Mars, Venus et une petite lune ; l intensite 0.6 se regle par profil dans le `.ini`.

### 8.1 Methode de capture quand la fenetre de l editeur n est pas au premier plan

- Le viewport ne redessine pas et le root WorldScape ne tique pas (sa mise a jour de materiau passe par son tick) : un
  `set_editor_property('TerrainMaterial')` ne change rien a l ecran. Appliquer le materiau sur le composant qui rend
  (`WorldScapeMeshComponent` du keeper GPU, slot 0) avec `create_dynamic_material_instance` en recopiant `PlanetLocation` de la MID
  remplacee.
- Capture : acteur `SceneCapture2D` + render target RGBA8, `CaptureSource` = Final Color LDR, `AutoExposureBias` 1.8, puis
  `capture_scene()` (rend tout de suite, sans viewport) et `RenderingLibrary.export_render_target` en PNG.
- Un materiau de planete tout juste charge s affiche en aplat gris clair tant que ses grosses textures se construisent (Mars : ~20 min
  au premier chargement de la session ; `T_MoonBaseColor` 16384 x 8192) et pendant la compilation d une nouvelle permutation : recapturer
  plus tard, ne rien conclure sur cet aplat.
