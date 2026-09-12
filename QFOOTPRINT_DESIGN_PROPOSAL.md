# QFootprint : empreintes de pas sur le sol des planetes

### Document de cadrage v1 : EN ATTENTE D ARBITRAGE (Benja / RzZz). Aucune ligne de code ecrite, aucun asset modifie. Tout ce qui suit a ete mesure dans le code du projet, les assets et le moteur 5.7 (build C:/UE5_Share), pas suppose.

> Demande d origine (Benja, 2026-09-11) : "un systeme optimise qui affiche les traces de pas des joueurs sur le terrain de nos planetes ; pas un truc complique ; ca doit fonctionner dans tout l univers, sur toutes les planetes, mais surtout deja dans le sable sur Terre. Puis peut-etre les vehicules, mais ils sont en antigravite donc pas tres utile."

---

## 0. Synthese en 12 lignes

1. Oui, la demande est comprise : quand un cyborg pose le pied sur le sol d une planete, une empreinte apparait, orientee dans le sens du pas, elle s efface au bout d un moment, et ca ne doit rien couter au serveur ni charger les clients.
2. **Il n existe aucune empreinte dans le jeu aujourd hui.** Il existe UN notify de pas, purement audio : `Footstep_AnimNotify` (Blueprint), pose sur 55 animations ALS et partage par le joueur ET toutes les IA humanoides. C est le point d accroche : **un seul noeud a ajouter**, rien a poser dans les animations.
3. **Le terrain WorldScape recoit les decals** (flag moteur par defaut, jamais desactive ; base pass et depth pass normales ; vrai en clipmap CPU et en quadtree GPU). Rien a debloquer.
4. **Mais le materiau Terre refuse la normale des decals** : `M_EarthBase` / `M_EarthBase_OPT` sont en `MaterialDecalResponse = ColorRoughness`. Sans changer ce reglage, une empreinte sera une tache plus sombre et plus mate, sans relief. Avec le reglage par defaut (`ColorNormalRoughness`), on a le creux. Decision D1.
5. **Le terrain n a aucun physical material** (zero dans WorldScape, zero dans le materiau, jamais propage aux maillages de collision). La matiere (sable / neige / terre / roche) se deduit du bruit WorldScape (`GetGroundNoise` : hauteur normalisee, temperature, humidite) plus la pente : c est exactement la methode du foliage.
6. **Rendu retenu : decals DBuffer projetes, pool fixe de `UDecalComponent`** (256 par defaut) tenu par un subsystem monde cote client. Zero tick, fondu calcule par le GPU, transform en double (LWC-propre a 3e8 cm de l origine, verifie dans le moteur). Une empreinte = un draw d un cube unite, elimine par taille ecran quand il est loin.
7. **Niagara (Decal Renderer + Data Channels) etudie et ecarte pour la v1** : simulation CPU seulement, un proxy de decal par particule de toute facon (meme cout GPU), reconstruction O(N) chaque frame meme immobile, aucun usage dans le projet, piege de tuile LWC. Reste la bonne brique pour une v2 avec eclaboussures.
8. **Multijoueur : 100 pour cent client, cosmetique.** Le serveur dedie ne cree pas le subsystem. Les autres joueurs laissent des traces par leurs animations repliquees. Le budget est borne par le pool, pas par les 500 joueurs.
9. **Un kit d assets dormant est reutilisable** : `Footprints_01` (texture masque + normale d empreinte, materiau decal DBuffer, deja cooke), pose a la main dans deux niveaux, jamais spawne par le jeu.
10. **Vehicules : hors perimetre**, antigravite = pas de contact. Si un jour il en faut, `Collision_Feedback_Component` sait deja poser des decals par materiau physique sur toute la flotte.
11. **Le plugin sera minuscule** : un `UDeveloperSettings`, un `UWorldSubsystem`, une `UBlueprintFunctionLibrary`, un acteur porte-pool. Environ 400 lignes de C++, un materiau, un noeud Blueprint.
12. **Rien n est engage tant que la section 8 n est pas arbitree** (4 decisions). Premiere action apres arbitrage : une sonde de faisabilite de 30 minutes dans l editeur, sans aucune ecriture (section 5, phase 0).

---

## 1. Ce qui existe, mesure

### 1.1 La chaine des pas : audio seulement

| Brique | Fait verifie |
|---|---|
| Notify | `Content/Systems/Character/Blueprints/AnimNotifys/Footstep_AnimNotify.uasset`, Blueprint derive de `/Script/Engine.AnimNotify`. Fonction `Received_Notify`. |
| Contexte disponible dans le notify | `MeshComp` (SkeletalMeshComponent), `AttachPointName` (socket `Foot_L` ou `Foot_R`, valeur posee par instance dans chaque animation), `FootstepType` (enum BP `Step`, `Walk/Run`, `Jump`, `Land`), `Sound`, volume et pitch modules par la courbe `Mask_FootstepSound`. |
| Ce qu il fait | `SpawnSoundAttached` du `Footstep_Cue` sur la socket, puis `SetIntParameter(FootstepType)` qui pilote un `SoundNodeSwitch`. |
| Ce qu il ne fait pas (0 occurrence dans l asset) | LineTrace, SphereTrace, PhysicalMaterial, SurfaceType, BreakHitResult, GetGroundNoise, Biome, Temperature, Humidity, WorldScape. **Aucune notion de surface.** Le son est du beton quelle que soit la matiere. |
| Animations porteuses | ~55 animations ALS de `Content/Marketplace/AdvancedLocomotionV4/CharacterAssets/MannequinSkeleton/AnimationExamples/` (Locomotion Walk/Run/Sprint N/CLF/CRF, InAir Jump/Land, Transitions, TurnInPlace, Mantle, GetUp). Scan complet de `Content/`, termine. |
| Qui l utilise | `ALS_AnimBP`, donc `ALS_Base_CharacterBP` (joueur) ET `AI_BaseCharacter` : `AI_Cyborg`, `AI_Cyborg_Police`, `AI_GuardCyborg`, `AI_GuardCaptain`, `AI_Pirate*`, `AI_Voss_*`, variantes `*_AILean`. Ne l utilisent pas : `ABP_Cyborg_V2`, `Cyborg_V1_AnimBP_Base`, `ABP_AILean`, Sanglantines et Infected (notify `AnimNotify_PlaySound` a eux), foule Mass, tous les `*_AnimalBP`. |
| Detection de sol du mouvement | `Plugins/NinjaCharacter/Source/NinjaCharacter/Private/NinjaCharacterMovementComponent.cpp:3554` `ComputeFloorDist`, FindFloor standard ; `bReturnPhysicalMaterial` jamais demande pour le sol (une seule occurrence, ligne 5041, a false, dans `ApplyRepulsionForce`). |
| Brique de trace disponible | `Plugins/Cy_Trace` : `ReturnPhysicalMaterial = true` par defaut (`Cy_TraceEnumStruct.h:26`), deja lie a `AI_Cyborg`. Jamais appele par la chaine des pas. |

Contrat gele : les valeurs `Foot_L` / `Foot_R` sont posees dans 55 animations. On ne les renomme pas, on les lit.

### 1.2 Le terrain WorldScape face aux decals

| Point | Mesure |
|---|---|
| Composant porteur | `UWorldScapeMeshComponent` (`Plugins/WorldScape/Source/WorldScapeCore/Public/WorldScapeMeshComponent.h:189`), proxy CPU `FWorldScapeSceneProxy` ou proxy GPU `FWSGPUTerrainSceneProxy` (`WSGPUTerrainRenderer.cpp:366`), choisi dans `CreateSceneProxy()` (`WorldScapeMeshComponent.cpp:1307-1318`). |
| Reception des decals | Aucun `SetReceivesDecals` / `bReceivesDecals=false` sur le terrain (6 occurrences dans le plugin, toutes feuillage ou collision de feuillage). Valeur effective = defaut moteur `true` (`PrimitiveComponent.cpp:335`), ecrite dans l uniform buffer par les deux proxies (`WSGPUTerrainRenderer.cpp:443-449`, `WorldScapeMeshComponent.cpp:487-494`). |
| Passes | Base pass et depth pass normales (`bUseForMaterial`, `bUseForDepthPass` a true, `SDPG_World`). Pas Nanite. `r.EarlyZPass=3`. |
| Materiau reel de l Univers | `Mi_EarthMat2_OPT` (seul chemin reference par `L_Persistent_Universe.umap`), parent `Content/Resources/MasterMaterial/M_EarthBase_OPT`, octets identiques a `M_EarthBase` sur les points ci-dessous. `MD_Surface`, `BLEND_Masked`, Substrate (`r.Substrate=True`). |
| **Reponse aux decals** | **`MaterialDecalResponse = MDR_ColorRoughness`** (chaine contigue relevee dans l asset). Le shader de base pass n applique le DBuffer que sur les canaux du masque (`Engine/Shaders/Private/BasePassPixelShader.usf:1096-1113`, meme chemin sous Substrate). Un decal DBuffer ne peut donc ecrire sur le terrain que couleur et rugosite, **pas la normale**. Et sous Substrate, tous les decals passent par le DBuffer (`RenderUtils.cpp:1512`). |
| RVT | Cable a moitie dans le materiau, mais **aucun volume RVT dans `L_Persistent_Universe.umap`** : canal mort en jeu. Pas de voie "peindre dans une texture virtuelle" sans construire ce pipeline. |
| Stabilite des composants | Les anneaux clipmap sont teleportes et reecrits a chaque pas (`WorldScapeLod.cpp:335-349`), les maillages de collision sont crees et detruits en continu (`WorldScapeRoot_Collision.cpp:7-88`), les tuiles GPU sont evincees chaque frame (`WSGPUTerrainRenderer.cpp:1736`). **Un decal projete en espace monde survit a tout cela ; un objet attache a un composant de terrain ne survit pas.** |
| Echelle | `PlanetScale = 316 929 984 cm` (3 169 km, valeur lue dans `L_Persistent_Universe`, `L_UniverseOnlyEarth`, `L_Dev_Claude`), centre planete a `z = -3.18e8 cm`. Pas de rebasing d origine (`p.EnableMultiplayerWorldOriginRebasing=False`, aucun `SetNewWorldOrigin` dans `Plugins/`). Regime LWC pur. |

### 1.3 La matiere du sol : il n y a pas de physical material

- Zero occurrence de `PhysMaterial`, `PhysicalMaterial`, `SurfaceType`, `bReturnPhysicalMaterial` dans tout `Plugins/WorldScape/Source` (recherche complete, terminee).
- Le `UBodySetup` du terrain prend `DefaultInstance = Terrain_CollisionSettings` sans `PhysMaterialOverride` (`WorldScapeMeshComponent.cpp:1519-1533`), et ce reglage n est propage qu aux LOD visuels, jamais aux `CollisionLods` sur lesquels on marche (`WorldScapeRoot_Main.cpp:3861-3862` contre `WorldScapeRoot_Collision.cpp:57-88`).
- Le joueur marche sur des maillages de collision CPU invisibles generes autour des joueurs (1 sous le joueur + 8 autour), `bUseComplexAsSimpleCollision`, qui bloquent tous les canaux (`WorldScapeLod.cpp:20-23`). Ce chemin reste CPU meme en mode GPU.
- Les biomes sont encodes en couleur de vertex `R = HeightNormalize, G = Temperature, B = Humidity` (`WorldScapeRoot_Thread.cpp:627, 750, 776, 801`), en **sRGB 8 bits** sur le chemin visible. Il n y a **aucun canal "sable"** : la classification plage / desert / herbe / roche / neige est faite dans le graphe du materiau par seuillage (`ShoreShift 0.739`, `ShoreShaprness 190`, `HeightShift -0.742`, `TempShaprness32`, `SnowHeightShift`, `Slope*`, cf. `Documentation/WorldScape_EarthMaterial_Audit_2026-09-06.md`).
- La seule source de verite gameplay est donc `AWorldScapeRoot::GetGroundNoise(Position)` (`WorldScapeRoot.h:2262`, BlueprintCallable, bruit CPU pur, deja utilise par le foliage et par `Plugins/QAI/Source/QAI/Private/Spawner/QAI_AgentSpawner.cpp:865`) qui rend `FNoiseData` (`HeightNormalize`, `Height`, `Temperature`, `Humidity`, `WaterMask`, `FoliageMask`). Le plugin calcule deja chaque tick `TemperatureAtPlayerPosition` / `HumidityAtPlayerPosition` / `PlayerAltitude` pour le joueur 0 (`WorldScapeRoot_Main.cpp:4520-4573`).
- `GetPawnNormal` / `GetPawnSnappedNormal` sont la direction radiale, **pas une normale de surface** (`WorldScapeRoot_Helper.cpp:149-163`). La pente vient du `ImpactNormal` du trace sur le maillage de collision.
- La table projet `EPhysicalSurface` (`Config/DefaultEngine.ini:694-729`) declare 36 surfaces dont `Sand` (28), `Snow` (24), `Mud` (4), `Soil` (29), `Gravel` (18), `Rock` (17). Piege mesure : les assets `PM_*` marketplace portent des index d une autre table (`PM_Sand` = SurfaceType13 = `Electrical`, `PM_Mud` = 14 = `Rubber`, `PM_Road_Surface` = 7 = `Glass`). Toute table surface vers empreinte basee sur ces assets heriterait de ces decalages. En v1 on ne s en sert pas.

### 1.4 Le kit d empreintes dormant

`Plugins/Qasset/Content/AssetStore/AbandonedFactory/Effects/Footprints_01/` (28 assets, cookes via `DA_EasyCookSeed_QANGA`) : `t_Footprint_01_01_Mud_m` (masque) et `t_Footprint_01_01_Mud_n` (normale), materiaux decal `m_Footprint_01_01_Mud` / `_Blood` / `_Oil` (blend `DBM_DBuffer` dans les octets), `bp_Footprint_01_01_MudLeft` / `_02_MudRight` (derives de `DecalActor`), eclaboussures Niagara `ns_Footprint_01_01_SplashMud` / `SplashWater`, `pm_Footprint_01_01_Mud`.
Seul consommateur : le personnage de demo du pack. Poses a la main comme decor dans `L_Env_Djibouti_09.umap` (27 empreintes de sang) et `L_ICLABS_TUTOV2.umap` (11 de boue). **Aucun code ne les spawne.**

### 1.5 Moteur 5.7 : ce que coutent les decals, et les pieges Niagara / LWC

| Fait | Preuve |
|---|---|
| Un decal classique = 1 `DrawIndexedPrimitive` d un cube unite, PSO par decal, lots de 128 par passe RDG | `Renderer/Private/CompositionLighting/PostProcessDeferredDecals.cpp:707-807` |
| Le Niagara Decal Renderer cree **1 `FDeferredDecalProxy` par particule** (pool de proxies, mais tableau d updates de N elements reconstruit **chaque frame** et pousse au render thread), et il est **CPU sim uniquement** | `NiagaraRendererDecals.cpp:205-238, 300-309`, `NiagaraDecalRendererProperties.h:41` |
| Les Niagara Data Channels sont **desactives sur serveur dedie** et jamais utilises dans le projet (0 asset, 0 C++) | `NiagaraDataChannelManager.cpp:175`, scan complet du projet |
| Positions Niagara en float32 relatives a une tuile LWC de 2^21 cm (~21 km) calculee depuis le composant ; le projet a `fx.LWCTileRecache=0` (`DefaultEngine.ini:319`), donc jamais de recache ni de reset, mais precision qui se degrade si le systeme s eloigne de sa tuile | `NiagaraTypes.h:79-92`, `NiagaraSystemInstance.cpp:822-827, 1308-1375` |
| `UDecalComponent` : transform **double** (`FTransform ComponentTrans`, `DeferredDecalProxy.h:42`), projection en translated world, `DecalToWorld` en double-float ; un seul chemin marque `LWC_TODO: Precision loss` par Epic (`SvPositionToDecal`, `DecalRenderingShared.cpp:239-243`) | a verifier a l oeil loin de l origine (phase 0) |
| Fondu integre : `SetFadeOut(StartDelay, Duration, bDestroyOwner)` arme un timer qui **detruit le composant** a la fin (`DecalComponent.cpp:284-292, 318-337`) ; `SetLifeSpan(0)` (public, `DecalComponent.h:152`) annule ce timer ; le fondu lui-meme est calcule dans le shader a partir du temps absolu (`DecalRenderingShared.cpp:274`) et re-arme a la reutilisation (`RendererScene.cpp:2562-2580`) | c est ce qui permet un pool sans tick |
| Un decal dont le fondu est termine n est **pas** elimine par le moteur (seule la taille ecran elimine : `bShouldRender = FadeAlpha > 0`, `DecalRenderingShared.cpp:500-507`) | d ou le balayage a 1 Hz du pool (2.5) |
| `FadeScreenSize` : elimination par taille ecran, multipliee par `r.Decal.FadeScreenSizeMult` | `DecalRenderingShared.cpp:400-412` |
| AnimNotify : aucune garde reseau ni serveur dedie dans `TriggerSingleAnimNotify` (`AnimInstance.cpp:1707-1729`). Le seul verrou est le tick de pose : `VisibilityBasedAnimTickOption` + `bRecentlyRendered` (`SkeletalMeshComponent.cpp:1718-1751`). Sur serveur dedie, le notify peut tourner (il tente deja le son) | la garde client doit etre dans notre noeud |
| Noeuds materiau disponibles pour un decal : `DecalColor` (couleur par decal sans instance dynamique, `SetDecalColor`), `DecalLifetimeOpacity` (le fondu) | `Engine/Public/Materials/MaterialExpressionDecalColor.h`, `MaterialExpressionDecalLifetimeOpacity.h` |

### 1.6 Reglages projet qui comptent

`r.DBuffer=True` (`DefaultEngine.ini:140`), `r.Substrate=True` (`:164`), `r.AllowStaticLighting=False`, `r.Shadow.Virtual.ForceOnlyVirtualShadowMaps=1`, `fx.LWCTileRecache=0` (`:319`), `fx.Niagara.QualityLevel=3` force sur les 5 paliers `EffectsQuality`. Aucun `r.Decal.*` dans `Config/`, aucun bucket decal dans `DefaultScalability.ini` : le budget empreintes est entierement a definir, sans conflit.
`DefaultEngine.ini:945-964` exclut deja `NiagaraEmitter` / `NiagaraScript` du serveur dedie.

---

## 2. Design propose

### 2.1 Vue d ensemble

```
Animation ALS (55 anims, notify existant aux bons frames)
  -> Footstep_AnimNotify.Received_Notify          (BP existant, son inchange)
       + 1 noeud : QFOOTPRINT_OnFootstep(MeshComp, AttachPointName, FootstepType)
            -> UQFootprint_Library (C++)
                 gardes : pas serveur dedie, systeme actif, type de pas retenu,
                          mesh rendu recemment, distance camera < MaxDistance
                 trace 1 ligne socket -> sol (le long de -Up du personnage)
                 classification matiere (WorldScape : bruit + pente ; sinon : rien en v1)
            -> UQFootprint_World_SubSystem (client seulement)
                 pool fixe de N UDecalComponent (ring buffer), zero tick
                 slot recycle : transform monde, DecalColor (teinte + intensite),
                                SetFadeOut(LifeTime, FadeTime) puis SetLifeSpan(0)
                 balayage 1 Hz : SetVisibility(false) des slots expires
  -> rendu : decal DBuffer, materiau M_QFootprint (masque + normale du kit Footprints_01)
```

### 2.2 Declencheur : un noeud dans le notify existant

- On ne cree PAS un nouveau notify (il faudrait le poser dans 55 animations x 2 pieds). On ajoute un appel en fin de `Received_Notify`, apres le son. Le son, ses parametres et sa courbe ne bougent pas (regle "modifier n est pas remplacer").
- Le noeud recoit ce que le notify a deja : `MeshComp`, `AttachPointName` (`Foot_L` / `Foot_R`, pied gauche / droit par suffixe, configurable), `FootstepType` (byte).
- Types retenus : `Step`, `Walk/Run`, `Land` (Land = empreinte plus marquee). `Jump` = decollage, pas d empreinte.
- Couverture immediate : joueur local, autres joueurs (leurs animations sont repliquees et jouent localement), IA humanoides qui partagent l AnimBP. Un CVar `qfootprint.AI` et une distance reduite pour les IA (D2).

### 2.3 Trace et orientation (gravite arbitraire)

- Depart = position de la socket + 15 cm vers le haut du personnage, arrivee = 40 cm vers le bas, canal `Visibility` (les collisions WorldScape bloquent tous les canaux), acteur du personnage ignore, `bReturnPhysicalMaterial = true` (gratuit).
- Haut du personnage = `GetActorUpVector()` du pawn (NinjaCharacter oriente la capsule sur la gravite), **jamais +Z monde**.
- Orientation du decal : axe X du decal = `-ImpactNormal` (un decal projette le long de son X), axe Z = direction avant du pied (axe de la socket projete sur le plan tangent). `DecalSize` (demi-etendues) = (20, 6, 15) cm : empreinte de 12 x 30 cm, projection sur 40 cm de haut pour absorber les variations de hauteur du terrain entre LOD.
- Pente : angle entre `ImpactNormal` et la direction radiale > `MaxSlopeDeg` (35 par defaut) = roche, pas d empreinte (et pas d etirement de projection).
- Anti-doublon : si la derniere empreinte du meme pied est a moins de `MinStepDistance` (10 cm), on ne pose rien (pietinement sur place).
- Cout : 1 trace + 1 evaluation de bruit par pas, soit 2 par seconde et par humanoide visible. Negligeable (le foliage en fait des milliers par seconde).
- Piste d optimisation si besoin (pas en v1) : lire `CurrentFloor` du CharacterMovement au lieu de tracer.

### 2.4 Classification de la matiere

Si le composant touche appartient a un `AWorldScapeRoot` :
1. `GetGroundNoise(position)` rend `HeightNormalize`, `Temperature`, `Humidity`, `WaterMask` (espace de la position a verifier a l implementation : le suivi joueur du plugin passe une position ECEF, `WorldScapeRoot_Main.cpp:4557-4571`, c est la reference a copier).
2. Regles, dans l ordre, avec des seuils dans le `UDeveloperSettings` (valeurs initiales relues sur l INSTANCE `Mi_EarthMat2_OPT`, pas sur le parent, et re-encodees en 8 bits sRGB comme le materiau) :
   - eau (`WaterMask`) : rien ;
   - pente > MaxSlope : roche, rien ;
   - bande de plage (`HeightNormalize` juste au-dessus du seuil `ShoreShift`) : **Sable**, intensite 1 ;
   - chaud et sec (temperature haute, humidite basse : le desert du materiau) : **Sable**, intensite 1 ;
   - froid (sous le seuil neige du materiau) : **Neige**, intensite 1, teinte claire ;
   - sinon : **Terre / herbe**, intensite 0.35 (empreinte discrete).

Sinon (sol de station, vaisseau, mesh) : v1 = rien. v2 = table `EPhysicalSurface` vers matiere, une fois les `PM_*` realignes sur la table projet (1.3).
Une seule texture d empreinte pour toutes les matieres en v1 ; la matiere ne change que la teinte et l intensite (via `DecalColor`), donc un seul materiau pour tout le pool et zero instance dynamique.

### 2.5 Rendu : pool de decals, zero tick

- `AQFootprint_Manager` (acteur transient, non replique, sans collision) spawne a la premiere empreinte, porte `N` `UDecalComponent` crees une fois (`N` = `MaxDecals`, 256 par defaut), transforms absolus.
- Poser une empreinte = prendre le slot suivant du ring (le plus ancien est recycle), `SetWorldTransform`, `SetDecalColor(teinte, intensite)`, `SetFadeOut(LifeTime, FadeTime, false)` puis `SetLifeSpan(0)` pour annuler la destruction, `SetVisibility(true)`. Aucune allocation apres le remplissage initial, aucune instance dynamique de materiau, aucun changement de materiau par slot (cela recreerait le proxy), aucun tick.
- Balayage a 1 Hz (timer du subsystem) : les slots dont `LifeTime + FadeTime` est ecoule passent `SetVisibility(false)` (sinon un decal invisible coute encore son draw, 1.5).
- `FadeScreenSize` regle pour qu une empreinte de 30 cm disparaisse au-dela d une cinquantaine de metres ; `MaxDistance` camera cote pose evite meme de tracer pour un personnage lointain.
- Cout GPU maximum : 256 cubes unites dans la passe decal, la plupart elimines par la taille ecran. Cout memoire : 256 composants, une fois. Cout CPU par frame : zero hors pose.
- Changement de monde : le subsystem meurt avec le monde, le pool avec lui. Rien n est persiste (empreintes ephemeres, comme les impacts de QWeapon).

### 2.6 Materiau d empreinte

- `M_QFootprint` : domaine Deferred Decal, Substrate, DBuffer (le projet est en `r.DBuffer=True`). Entrees : masque `t_Footprint_01_01_Mud_m`, normale `t_Footprint_01_01_Mud_n` (kit, reutilises par reference, pas copies) ; `DecalColor` RGB = teinte, A = intensite (multiplie l opacite) ; `DecalLifetimeOpacity` = fondu ; parametre statique `MirrorU` pour le pied droit (donc 2 instances : `MI_QFootprint_L`, `MI_QFootprint_R`, un sous-pool chacune).
- Le kit `m_Footprint_01_01_Mud` sert de reference visuelle, pas de parent (fichier marketplace, on ne le modifie pas).
- Sans D1, la normale n a aucun effet sur le terrain ; le materiau reste le meme, seul le resultat visible change.

### 2.7 Reseau et serveur dedie

- Aucune replication, aucun RPC, aucune nouvelle `UPROPERTY` repliquee. L etat autoritaire n est pas concerne.
- `UQFootprint_World_SubSystem::ShouldCreateSubsystem` rend false si `IsRunningDedicatedServer()` ou monde non Game/PIE. Le noeud BP sort immediatement sans subsystem. Le notify continue de faire ce qu il fait sur le serveur (comportement inchange).
- Serveur d ecoute : l hote est un client comme un autre.
- Autres joueurs : le notify tourne chez moi pour leurs meshes tant que leur pose est mise a jour ; si `VisibilityBasedAnimTickOption` est en `OnlyTickPoseWhenRendered` sur leur mesh, les joueurs hors ecran ne posent rien (parfait). La valeur effective sur `ALS_Base_CharacterBP` n est pas lisible statiquement (3 valeurs differentes dans les octets) : mesure en phase 0.

### 2.8 Budget et reglages

`[/Script/QFootprint.QFootprint_Settings]` dans `Config/DefaultGame.ini` (comme les autres Q*) : `bEnabled`, `MaxDecals` 256, `LifeTime` 60 s, `FadeTime` 10 s, `MaxDistance` 6000, `MaxDistanceAI` 3000, `bAI` true, `MaxSlopeDeg` 35, `MinStepDistance` 10, `DecalSize`, `FadeScreenSize`, seuils de matiere, teintes et intensites par matiere, `TSoftObjectPtr` vers les 2 instances de materiau.
CVars runtime : `qfootprint.Enabled`, `qfootprint.AI`, `qfootprint.LifeTimeScale`, `qfootprint.Debug` (dessine le trace et affiche la matiere ; strippe en Shipping).
Log : `LogQFootprint`. Une ligne au demarrage du subsystem : actif ou non, taille du pool, materiaux resolus.

### 2.9 Ce que ca ne fait pas (v1)

Pas de deformation reelle du sable (il faudrait une RVT ou un render target dans le materiau Terre, donc toucher `M_EarthBase_OPT` et sa vertex factory : recompilation de tous les materiaux du jeu, cf. memoire du 2026-09-06). Pas d eclaboussures (v2 Niagara). Pas d empreintes d animaux ni de Sanglantines (autres notifies ; le meme noeud peut y etre ajoute si voulu). Pas de traces de vehicules (antigravite). Pas de persistance entre sessions.

---

## 3. Pourquoi pas Niagara, et quand

| Critere | Pool de UDecalComponent | Niagara Decal Renderer + Data Channels |
|---|---|---|
| Cout GPU | 1 draw par empreinte visible | identique : 1 proxy de decal par particule (`NiagaraRendererDecals.cpp:226-238`) |
| Cout CPU par frame | 0 (fondu GPU, balayage 1 Hz) | O(N) : tableau d updates de N entrees reconstruit et envoye chaque frame, meme immobile ; simulation CPU obligatoire |
| Objets | N composants, une fois | 0 UObject par empreinte, mais un systeme par ilot NDC |
| LWC | transform double, verifie | float32 par tuile, `fx.LWCTileRecache=0` dans le projet, TODO moteur sur les data interfaces |
| Existant projet | `UDecalComponent` deja utilise (QPolice, Collision_Feedback, impacts BallisticsVFX) | 0 usage de NDC, 0 usage du Decal Renderer |
| Serveur dedie | subsystem non cree | NDC desactive par le moteur |
| Complexite | ~400 lignes C++, 1 materiau | asset NDC, systeme Niagara, ilots, reglage LWC, plus le C++ d ecriture |
| Extensibilite | API C++/BP `QFOOTPRINT_AddFootprint(Location, Normal, Forward, Side, Matter, Strength)` pour tout emetteur (vehicule, IA, quete) | Data Channel ecrivable par tout BP |

Quand Niagara redevient le bon choix : v2 avec eclaboussures d eau ou de poussiere au pas (le kit a `ns_Footprint_01_01_SplashMud` / `SplashWater`), qui sont des particules courtes et mobiles, ce pour quoi Niagara est fait. Le pool de decals reste pour l empreinte elle-meme.

---

## 4. Fichiers et assets touches

Nouveaux :
- `Plugins/QFootprint/QFootprint.uplugin`, `Source/QFootprint/QFootprint.Build.cs` (deps : Core, CoreUObject, Engine, DeveloperSettings, WorldScapeCore ; aucune dependance vers une couche haute), `Public/QFootprint.h`, `Public/QFootprint_Settings.h`, `Public/QFootprint_World_SubSystem.h`, `Public/QFootprint_Library.h`, `Public/QFootprint_Manager.h` et leurs `.cpp`. En-tete `// QANGA // IOLACORP. All Rights Reserved`, ASCII strict, C++20, pas d exception, pas de static local, `TWeakObjectPtr` pour tout ce qui est capture par le timer.
- `Content/Systems/Footprint/M_QFootprint`, `MI_QFootprint_L`, `MI_QFootprint_R`.
- `Documentation/QFOOTPRINT_ARCHITECTURE.md` (a la livraison).

Modifies :
- `Content/Systems/Character/Blueprints/AnimNotifys/Footstep_AnimNotify.uasset` : 1 noeud ajoute en fin de `Received_Notify`. Backup + md5 avant, `validate_blueprint_graph` apres. Aucune variable renommee.
- `Config/DefaultGame.ini` : nouvelle section `[/Script/QFootprint.QFootprint_Settings]`.
- `QANGA.uproject` : entree du plugin.
- Inscription cook : les instances de materiau sont referencees en soft depuis les settings C++, donc **jamais cuites toutes seules** (piege connu du projet) : ajout dans `DA_EasyCookSeed_QANGA` (skill `qanga-assets`).
- Selon D1 : `Content/Resources/MasterMaterial/M_EarthBase_OPT.uasset` (et `M_EarthBase` pour coherence), `MaterialDecalResponse` vers `ColorNormalRoughness`.

Pas touches : les 55 animations, `ALS_AnimBP`, `ALS_Base_CharacterBP`, NinjaCharacter, le code WorldScape, le kit Footprints_01, `Footstep_Cue`.

---

## 5. Chantiers et ordre

**Phase 0 : sonde de faisabilite (30 min, editeur, aucune ecriture, `L_Dev_Claude`)**
1. Poser en PIE un `bp_Footprint_01_01_MudLeft` sur le terrain et un autre sur un static mesh voisin, capture des deux : prouve la projection sur le quadtree GPU et montre l effet du masque `ColorRoughness` (attendu : pas de relief sur le terrain, relief sur le mesh). C est la piece a conviction pour D1.
2. Meme test loin de l origine du monde (flanc de la planete) : aucun tremblement du decal (chemin `SvPositionToDecal`).
3. Lire la valeur effective de `VisibilityBasedAnimTickOption` sur le mesh du pawn et d une IA (2.7).
4. Lire sur `Mi_EarthMat2_OPT` les seuils `ShoreShift`, `HeightShift`, temperature / humidite / neige / pente (2.4).
5. Confirmer l espace attendu par `GetGroundNoise` en appelant `WS_GetTemperatureValueAtLocation` a la position du pawn et en comparant a `TemperatureAtPlayerPosition`.

**Phase 1 : plugin C++ (demi-journee)** : settings, subsystem, manager + pool, library, CVars, log. Compilation par Benja (regle de la machine). Test QATS `QATS.QFootprint.PoolIsFixed` (le ring ne grandit jamais, recycle le plus ancien) et `QATS.QFootprint.SettingsResolve` (les soft refs se resolvent), dans un monde de test si le harnais le permet, sinon commande de dump en PIE.

**Phase 2 : materiau et instances (1 h)** : `M_QFootprint` + 2 instances, verification a l oeil sur un mesh puis sur le terrain (selon D1).

**Phase 3 : branchement (30 min)** : 1 noeud dans `Footstep_AnimNotify`, section ini, cook seed, `.uproject`.

**Phase 4 : validation (1 h)** : section 7.

**Phase 5 : doc + changelog Discord (30 min)**.

---

## 6. Risques de regression et parades

| Risque | Parade |
|---|---|
| `Footstep_AnimNotify` est utilise par 55 animations et toutes les IA humanoides : une erreur casse le son des pas partout | 1 noeud ajoute apres le son, en fin de graphe, rien de renomme ; backup md5 ; `validate_blueprint_graph` ; test PIE du son avant / apres |
| D1 (normale des decals sur le terrain) change aussi le rendu des decals de decor deja poses sur le terrain (`DECAL_SandPath`, `DECA_*` de Qasset) : ils ecriraient desormais leur normale | capture avant / apres sur une zone qui en a ; retour arriere = un enum |
| Recompilation de `M_EarthBase_OPT` (toutes les permutations de la VF WorldScape pour ce materiau) : editeur en materiau par defaut le temps de la compilation | a faire a un moment choisi, annonce a Benja (memoire du 2026-09-06) |
| Serveur dedie : un subsystem cree par erreur allouerait 256 composants pour rien | `ShouldCreateSubsystem` + ligne de log ; verification sur build serveur (phase 4) |
| Un decal projete sur une pente forte s etire | seuil `MaxSlopeDeg` |
| Terrain qui change de LOD sous l empreinte (quelques cm) | profondeur de projection 40 cm |
| Empreintes posees hors ecran pour rien | garde `WasRecentlyRendered` + `MaxDistance` |
| Soft refs C++ non cuites | inscription cook seed (phase 3) + test QATS |
| Couche : `QFootprint` depend de `WorldScapeCore` (plugin tiers bas niveau) | sens autorise (Q* vers tiers) ; aucune dependance inverse |
| Le timer du balayage capture le manager | `TWeakObjectPtr`, valide avant usage ; timer arrete dans `Deinitialize` |

---

## 7. Validation : comment on prouve que ca marche

1. PIE `L_Dev_Claude`, marche sur la plage puis dans le desert : empreintes gauche / droite alternees, orientees dans le sens de la marche, qui s effacent apres `LifeTime` (capture a 0 s, 30 s, 70 s).
2. `stat SceneRendering` : nombre de decals rendus <= `MaxDecals` ; `stat unit` avant / apres sans ecart mesurable ; `qfootprint.Debug 1` pour lire la matiere deduite.
3. Deux clients (serveur d ecoute + client) : chacun voit les traces de l autre.
4. Loin de l origine (flanc / antipode via le teleport admin) : pas de tremblement, pas de decalage.
5. Serveur dedie (build `QangaServer`) : ligne de log "QFootprint: disabled (dedicated server)", zero composant.
6. Herbe / roche / station : pas d empreinte (ou discrete pour la terre), pas d empreinte sur les pentes.
7. Son des pas identique avant / apres (le notify n a rien perdu).
8. `Automation RunTests StartsWith:QATS.QFootprint` vert.

---

## 8. Decisions ouvertes (a arbitrer avant la phase 1)

- **D1. Relief ou tache ?** Passer `M_EarthBase_OPT` (et `M_EarthBase`) en `MaterialDecalResponse = ColorNormalRoughness` pour que les empreintes aient un creux. Cout : une recompilation de ce materiau, et les decals de decor sur le terrain gagnent aussi leur normale. Recommandation : **oui**, apres la sonde de phase 0 qui montre les deux rendus cote a cote. Sans D1, la v1 reste une tache sombre et mate (acceptable pour commencer, moins "2026").
- **D2. Qui laisse des traces ?** Joueur seul, ou joueur + IA humanoides (gratuit techniquement : meme notify ; le pool borne le cout). Recommandation : **tout le monde**, avec `MaxDistanceAI` plus court.
- **D3. Ou vit le code ?** Nouveau plugin `QFootprint` (isole, convention Q*) ou module dans `QPlayers`. Recommandation : **nouveau plugin**, comme les autres systemes.
- **D4. Reglages de depart.** `LifeTime` 60 s, `FadeTime` 10 s, `MaxDecals` 256, sable uniquement en v1 ou toutes matieres avec intensite (2.4). Recommandation : **toutes matieres, intensite faible hors sable et neige**, c est le meme code.

## 9. Phase 0 : mesures du 2026-09-11 (editeur, L_Dev_Claude)

Arbitrage de Benja le 2026-09-11 : D1 a D4 acceptes tels que recommandes. Editeur lance par la session avec
`-DisablePlugins=RzWebWidget` (le module `RzIntranet` de ce plugin, en cours d ecriture par une autre session,
n etait pas compile et bloquait le boot sur la modale "Missing QANGA Modules" ; aucune dependance externe a ce
plugin dans les Build.cs, les uplugin ni la carte). Captures : `Documentation/QFootprint/p0_*.jpg`.

| Question | Mesure |
|---|---|
| Materiau du terrain de L_Dev_Claude | `Mi_EarthMat2_OPT` (parent `M_EarthBase_OPT`), identique a l Univers. Confirme. |
| `MaterialDecalResponse` effectif (Python sur l asset charge) | `M_EarthBase_OPT` = `MDR_ColorRoughness`, `M_EarthBase` = `MDR_ColorRoughness`. |
| Decal sur le terrain GPU (`bUseGPUNoise` et `bUseIndirectInstancedNoise` a true), a 1236 km de l origine | Projete et visible. Sans D1 : tache jaune floue sans relief (`p0_before_D1_terrain_closeup`). Le meme decal du kit, pose sur un plan a materiau moteur 1,5 m a cote : empreinte nette avec relief (`p0_before_D1_side_by_side`). |
| D1 applique en memoire (`MDR_ColorNormalRoughness` + recompile : 513 jobs shader, 76 s de thread, 71 pour cent de DDC) | Relief revenu sur le terrain, meme angle, meme decal (`p0_after_D1_terrain_closeup`). D1 est demontre. |
| Espace attendu par `GetGroundNoise` et `WS_Get*ValueAtLocation` | **ECEF (repere de la planete)** : `InverseTransformLocation(Root.GetActorTransform(), PositionMonde)`. Le root de L_Dev_Claude est tourne (pitch -3.6, yaw -178.1, roll -40.8) donc "monde moins position de l acteur" est faux aussi. Une position monde brute rend T=0.998 au lieu de 0.394 (valeur du root pour la meme camera). `WS_SingleProject`, `GetPawnAltitude`, `GetGroundHeight` prennent eux une position MONDE (ils convertissent). Piege a garder en tete pour `QAI_AgentSpawner.cpp:865` (a verifier, hors perimetre). |
| Hauteur normalisee | `HeightNormalize = Height / NoiseIntensity` (2 200 000) ; ocean = `OceanHeight` 1 120 000 soit Hn 0.5091. La couleur de vertex R = sRGB(Hn) : `ShoreShift 0.739` et `HeightShift -0.742` valent en lineaire 0.5056 et 0.5103, soit -77 m et +27 m autour du niveau de la mer. |
| Seuils de l instance `Mi_EarthMat2_OPT` | ShoreShift 0.739, ShoreShaprness 190, HeightShift -0.742, SlopeOffset -0.616, SlopeOffset2 -0.555, SlopeOffset3 0.97, SlopePower 0.07, SnowHeightShift 1, SnowTempShaprness32 10, TempShift3 10, TempShaprness32 -0.141, Snow Influence1/2/3 = 1 / 152.4 / 10, GrassSnow 3. Les regles chaud-sec et neige du graphe restent a lire dans le materiau, les noms ne suffisent pas. |
| Climat | `Temperature` et `Humidity` en [0..1]. Camera de depart de L_Dev_Claude : T 0.394, H 0.630 (herbe temperee, capture `shot1`). 45 echantillons desert (T > 0.6, H < 0.35, terre emergee) sur 1260, le plus proche a 790 km (lat 30, lon 80 du repere planete, monde (-62805248, -103211313, -25002491)). |
| `VisibilityBasedAnimTickOption` (CDO) | `ALS_Base_CharacterBP_C` et `AI_Cyborg_C` : `AlwaysTickPoseAndRefreshBones`, `bNoSkeletonUpdate` false. Les notifies tournent donc aussi hors ecran et sur serveur dedie : garde client + `WasRecentlyRendered` + distance obligatoires dans notre noeud. |
| Normale de sol par le bruit | `WS_GetTerrainNormalTriTest` a rendu (0,0,0) au point teste. La recette 3 points projetes (`WS_SingleProject` + produit vectoriel) donne une normale coherente (pente 11.9 deg). En jeu on prend l `ImpactNormal` du trace. |
| Kit `m_Footprint_01_01_Mud` sous Substrate | Rend correctement (domaine Deferred Decal, visible sur plan et terrain). |

Ce que la phase 0 change dans le plan : (a) le classifieur 2.4 travaille en ECEF via `InverseTransformLocation` et
compare `Height` a `OceanHeight` (metres au-dessus de la mer) plutot que des seuils sRGB ; (b) D1 est demontre ;
(c) la garde de rendu recent est obligatoire.

Etat a la fin de la phase 0 : `M_EarthBase_OPT` et `M_EarthBase` passes en `ColorNormalRoughness` dans l editeur de
la session (en memoire ; la sauvegarde par script a ete refusee par le classificateur de l outil, a sauver depuis
l editeur). Backups md5 des 3 materiaux dans le scratchpad de session `backup_2026-09-11/`. Sondes : 3 acteurs
`QFP_PROBE` (2 DecalActor + 1 StaticMeshActor) dans L_Dev_Claude vers (-63032888, -103585407, -23940512), a
detruire sans sauver la carte.

Reste non verifie : la precision visuelle en mouvement loin de l origine (une capture fixe ne montre pas un
tremblement) et la regle exacte chaud-sec / neige du graphe `M_EarthBase_OPT`.

## 10. Livraison du 2026-09-11 (phases 1 a 4, sur demande de Benja : "tu rebuild et tu fais le boulot jusqu au bout")

Reference d exploitation : `Documentation/QFOOTPRINT_ARCHITECTURE.md`.

| Phase | Fait | Preuve |
|---|---|---|
| 1 C++ | Plugin `Plugins/QFootprint` (settings, subsystem client, manager du pool, facade BP, 2 tests QATS), active dans `QANGA.uproject` | `Build.bat QangaEditor` : `Result: Succeeded`, 0 warning. Piege rencontre : `"PlatformAllowList": []` copie de QTriggerZone = module jamais compile en 5.7 (liste vide = aucune plateforme), corrige. |
| 2 Materiau | `Content/Systems/Footprint/M_QFootprint` + `MI_QFootprint_L` / `_R` (MirrorU), textures du kit Footprints_01 par reference | probe de decals sur le desert : variante sans normale ne compile pas (Substrate), variante relief seul presque invisible, version livree = BaseColor + Normal + Opacity avec teinte plus sombre que le sol |
| 3 Branchement | 1 noeud `QFP On Footstep` en tete de `Footstep_AnimNotify.Received_Notify` (5 liaisons), compile, sauve ; section ini + `+DirectoriesToAlwaysCook` | md5 du notify `45ee8be1...` -> `f4f5c4ef...`, le son est intact (chaine d origine reconnectee derriere le noeud) |
| 4 Validation | PIE `L_Dev_Claude`, pawn teleporte sur le sable : pool 256 pret, atterrissage reel via le notify -> matiere Sand, 6 appels scriptes -> 2 poses + 4 refus anti doublon (`visible=4 placed=4`), teardown `placed 4, rejected 0` | log `LogQFootprint` de la session ; tests headless `Automation RunTests StartsWith:QATS.QFootprint` : 2 succes, 0 echec (`QATS.QFootprint.Manager.PoolIsFixed`, `QATS.QFootprint.Settings.MaterialsResolve`), rapport `Saved/QATSAutomation_QFootprint/index.json` |

Observation utile : sur les animations de saut et d atterrissage le notify porte `AttachPointName = root` (pas un pied) ;
l empreinte est alors posee sous le bassin et compte comme pied droit. Acceptable, a affiner si besoin (D2 v2).

Non fait, volontairement : la validation visuelle en marchant (Benja, en jeu), la neige (jamais vue), les sols hors
WorldScape, les IA hors ALS, les vehicules. Les teintes des styles sont dans `DefaultGame.ini` et se reglent sans rebuild.

### 10.1 Retour de Benja en jeu (2026-09-11 soir) et corrections

Constats de Benja en marchant : "pas assez visibles", "tournees de 90 degres", "en ligne droite au centre du perso et
pas sous les pieds", puis "les decals se posent aussi sur le mesh du perso quand je repasse dessus", puis avec l offset
de 90 : "180 degres de trop", "on y est presque".

| Constat | Cause mesuree | Correction |
|---|---|---|
| Ligne droite au centre | Les animations de marche ALS passent `AttachPointName = root` au notify (pas `Foot_L` / `Foot_R`) : le trace partait du bassin, cote toujours droit | Resolution du pied depuis le squelette (`foot_l` / `foot_r`, `ball_l` / `ball_r`) : suffixe du socket si present, sinon le pied le plus bas ; centre entre cheville et avant-pied ; direction cheville vers avant-pied. Rebuild par Benja. Verifie en PIE : `socket root -> foot right (foot_r)` dans le log |
| 90 degres | La texture du kit a la chaussure le long de U, pas de V | `TextureYawOffset` = 90 (sonde editeur : axe long dans le sens de marche) puis -90 apres le retour "180 de trop" (pointe vers l avant) |
| Pas assez visibles | Teintes trop proches du sol, semelle douce dans le masque | Teintes plus sombres a intensite pleine (sable 0.06/0.045/0.03 lineaire, 1.0) et parametre `MaskBoost` 1.8 dans le materiau (saturate((1 - R) x boost)) |
| Decal peint sur le cyborg | Boite de projection centree 2 cm au-dessus du sol avec 20 cm de demi-profondeur : les pieds et les mollets traversaient le volume | `SurfaceOffset` = -15 : la boite couvre -35 a +5 cm autour du sol. Aucune desactivation de la reception des decals sur le cyborg (d autres effets peuvent en dependre) |
| Empreinte trop large, pas assez longue (retour du 2026-09-12 apres la pointe corrigee) | L offset tourne TOUTE la boite, donc l axe long de la boite s est retrouve en travers ; et la texture du kit n est pas remplie bord a bord : lecture des pixels du masque (render target + `read_render_target_pixel`, valeurs 0..255) : la chaussure occupe u = [0.36, 0.67] (31 pour cent) et v = [0.11, 0.90] (79 pour cent), centree ; le V du decal tombe sur l axe Y de la boite | `DecalSize` (20, 19, 18.5) : boite de 38 x 37 cm, pied de 30 x 11.5 cm. Benja : "bon pour le ratio mais trop grand" (la botte du cyborg fait environ 20 cm) -> `DecalSize` (20, 12.5, 12) : boite de 25 x 24 cm, pied de 20 x 7.5 cm, pris a la prochaine creation du pool (nouveau PIE). Header, ini et CDO a chaud alignes |

Ces reglages sont dans `DefaultGame.ini` (section QFootprint) et dans les defauts C++ (`QFootprint_Settings.h`, a
recompiler pour qu ils comptent sans ini). Le materiau a ete reconstruit proprement (18 noeuds, `MaskBoost` inclus).

**Cloture (2026-09-12, 00:30) : valide par Benja en jeu** ("la pointe est dans le bon sens, ca marche", "bon pour le ratio",
"la taille est nickel"). Reste ouvert pour plus tard : neige (jamais vue), sols hors WorldScape, IA hors ALS, eclaboussures v2.
