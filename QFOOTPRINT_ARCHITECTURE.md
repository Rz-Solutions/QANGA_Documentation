# QFootprint : empreintes de pas sur le sol des planetes

Livre le 2026-09-11 (session Claude, compile et valide en PIE sur `L_Dev_Claude`). Cadrage et mesures d origine :
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
       matiere : WorldScape -> GetGroundNoise(ECEF) + GetPawnAltitude + pente vs radial ; sinon rien (v1)
       anti doublon : meme pied a moins de MinStepDistance = rien
  -> UQFootprint_SubSystem::AddFootprint
       style de la matiere (teinte, intensite, taille), rotation MakeFromXZ(-normale, avant)
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
| Materiau Terre (D1) | `Content/Resources/MasterMaterial/M_EarthBase_OPT` : `MaterialDecalResponse` = `ColorNormalRoughness` (sauve par Benja le 2026-09-11 ; `M_EarthBase` inutilise par les cartes, laisse en `ColorRoughness`) |

## 4. Reglages (DefaultGame.ini, section QFootprint) et CVars

- Activation : `bEnabled`, `bEnableForAI`, `PrintFootstepTypes` (0 Step, 1 Walk/Run, 3 Land ; 2 Jump exclu), `LandFootstepType`, `LandIntensityMultiplier`, `LeftFootSocketSuffix` (`_L`).
- Budget : `MaxDecals` 256 (moitie gauche, moitie droite, jamais plus), `LifeTime` 60 s, `FadeTime` 10 s, `FadeInTime`, `MaxDistance` 6000 cm, `MaxDistanceAI` 3000, `RecentlyRenderedTolerance`, `FadeScreenSize` 0.005, `MinStepDistance` 10 cm, `SweepInterval` 1 s.
- Placement : `TraceUp` 15, `TraceDown` 40, `MaxSlopeDeg` 35, `SurfaceOffset` -15 (la boite de projection est enfoncee sous le sol : avec une profondeur de 20 elle couvre -35 a +5 cm, donc le cyborg qui repasse sur ses traces n est pas peint), `DecalSize` (20, 12.5, 12) demi-etendues cm (l axe V du decal tombe sur Y et U sur Z ; la chaussure du kit occupe 79 pour cent de V et 31 pour cent de U, centree, mesure par lecture des pixels du masque le 2026-09-12 : une boite de 25 x 24 cm donne un pied de 20 x 7.5 cm, la taille de la botte du cyborg validee par Benja ; pour agrandir, garder le rapport Y / Z), `TextureYawOffset` -90 (l offset tourne toute la boite autour de l axe de projection ; -90 met la pointe vers l avant, mesure en jeu par Benja), `MaterialLeft` / `MaterialRight`.
- Pieds : `bResolveFootFromBones` true, `LeftFootBone` foot_l, `RightFootBone` foot_r, `LeftBallBone` ball_l, `RightBallBone` ball_r, `FootCenterAlpha` 0.55 (centre de l empreinte entre cheville et avant-pied), `LeftFootSocketSuffix` `_L`, `RightFootSocketSuffix` `_R`. Le socket du notify vaut `root` sur les animations de marche ALS (mesure 2026-09-11) : sans cette resolution, toutes les empreintes tombent sous le bassin en ligne droite. Le suffixe du socket decide le cote quand il est present, sinon le pied le plus bas le long de l axe haut du pawn ; la direction de la chaussure = cheville vers avant-pied.
- Materiau : parametre `MaskBoost` 1.8 (durcit la semelle : opacite = saturate((1 - R) x MaskBoost)).
- Matiere (bruit WorldScape) : `MinAltitudeCm` -50 (eau), `BeachAltitudeMaxCm` 3000 (sable de plage), `DesertTemperatureMin` 0.6 et `DesertHumidityMax` 0.35 (sable de desert), `SnowTemperatureMax` 0.25, sinon terre ; `bPrintOnNonTerrain` false (sols de station, vaisseaux : rien en v1).
- Styles : `SandStyle`, `SnowStyle`, `SoilStyle`, `MudStyle` = teinte lineaire (plus sombre que le sol), intensite (alpha du DecalColor), echelle. Valeurs livrees mesurees sur le desert de L_Dev_Claude.
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

Pas de deformation reelle du sable, pas d eclaboussures (v2 Niagara sur le kit `ns_Footprint_01_01_Splash*`), pas d empreintes hors terrain WorldScape (table `EPhysicalSurface` a realigner d abord, les `PM_*` marketplace portent des index d une autre table), pas d animaux ni de Sanglantines (autres notifies, meme noeud a ajouter), pas de vehicules (antigravite). Persistance : aucune, le pool meurt avec le monde.
