# AtmoScape — Architecture

> Plugin de rendu de diffusion atmosphérique (atmospheric scattering) pour QANGA. `Type: Runtime` (module `AtmoScape`, `LoadingPhase: PostConfigInit`) + module éditeur `AtmoScapeEditor` (`Type: Editor`, `LoadingPhase: PreDefault`). Version `3.0`, auteur iolaCorp Studio. Le modèle physique de référence est tiré de Scratchapixel ("Simulating the Colors of the Sky"). Diffusion purement **visuelle et client-side** : rien n'est répliqué, et l'actor s'auto-désactive sur dedicated server.

---

## 1. But d'usage

QANGA est un jeu à planètes entières (système WorldScape). Le `SkyAtmosphere` natif d'Unreal modélise une atmosphère unique centrée sur l'origine du monde et ne tient pas la route quand on s'éloigne de la surface ou qu'on passe d'un corps céleste à l'autre. AtmoScape fournit une atmosphère **attachée à un actor**, donc déplaçable et instanciable par planète : chaque corps possède son `AAtmoScape` avec son propre rayon, sa hauteur d'atmosphère, ses coefficients Rayleigh/Mie et son ozone.

Concrètement, le plugin résout :

- **Diffusion vue depuis le sol et depuis l'espace** : un même actor empile plusieurs coquilles sphériques (shells) qui rendent l'in-scatter du ciel, le skylight (lumière ambiante diffusée), l'absorption (extinction de ce qui est vu à travers), et l'airglow (halo de rim au limbe de la planète).
- **Mise à l'échelle physique** : les paramètres sont saisis en kilomètres (rayon planète, hauteur d'atmosphère, scale-heights Rayleigh/Mie) et convertis en centimètres-monde, ce qui permet de calibrer une atmosphère « accurate to earth » par défaut.
- **Presets par corps céleste** (`UAtmoScapeBodyDataAsset`) : un asset réutilisable par planète réelle (Mars, Vénus, Titan…) au lieu d'un réglage à la main par instance.
- **Champ d'étoiles** (`AStarfieldScape`) : une sphère de très grand rayon affichant starfield + disque solaire, indépendante de l'atmosphère.

Ce que le `SkyAtmosphere` d'Epic ne donne pas et qu'AtmoScape vise : plusieurs atmosphères co-existantes, déplaçables avec la planète, source lumineuse surchargeable (`LightSource`), et un modèle d'échelle planétaire explicite.

---

## 2. Vue d'ensemble / décisions d'architecture

Le système a **deux générations de rendu** qui coexistent dans le code :

### 2.1 Renderer « shell » classique (en production)

C'est le chemin par défaut et le seul effectivement câblé sur des assets. `AAtmoScape` instancie cinq `UStaticMeshComponent` (des coquilles sphériques), chacun portant un `UMaterialInstanceDynamic` créé à partir d'un master material du plugin :

| Composant mesh | Master material | Rôle |
| --- | --- | --- |
| `PlanetaryAtmoMesh` | `MM_PlanetaryAtmo` | In-scatter du ciel (Rayleigh + Mie) |
| `PlanetarySkylightMesh` | `MM_PlanetarySkylight` | Skylight (lumière ambiante diffusée) |
| `PlanetaryAbsorptionMesh` | `MM_Absorbtion` | Extinction à travers l'atmosphère (`TranslucencySortPriority = -2`) |
| `PlanetaryOutterMesh` | `MM_PlanetaryOutterAtmo` | Airglow / halo de rim |
| `SpacePlanetaryAtmoMesh` | (voir §7 — non assigné en C++) | Vue depuis l'espace ; **masqué** par défaut |

Toute la logique runtime tient dans `AAtmoScape::UpdateScale()` : elle calcule le rayon d'atmosphère, l'échelle des meshes, la longueur de segment de raymarch (`LightSegLength`), normalise les coefficients de diffusion par `MultiScatering`, puis pousse ~25 paramètres dans chaque MID via `UpdateMaterialParameters()`. La diffusion elle-même (raymarch single-scattering Rayleigh+Mie avec optical depth) vit dans le shader `AtmoScape.ush`, inclus par les Custom nodes des master materials.

`AtmoScape.ush` est une **consolidation** : son en-tête indique qu'il unifie un raymarch (`AtmoIntegrate`) jadis copié-collé entre in-scatter et skylight, et qu'il fait enfin respecter le paramètre `MiePhase` (auparavant un `g=0.76` codé en dur).

### 2.2 Renderer LUT « Phase 2 » (expérimental, opt-in, non câblé)

Une seconde implémentation, **clean-room**, vit dans `AtmosphereCommon.ush` : un modèle à LUT précalculées (famille Bruneton / Hillaire) — transmittance LUT, multiple-scattering LUT, raymarch du ciel, et composite post-process avec aerial perspective. Côté C++ elle est exposée par :

- le flag maître `bUseLUTModel` (défaut `false`),
- `RegenerateLUTs()` qui bake la transmittance LUT dans un `UTextureRenderTarget2D` via `UKismetRenderingLibrary::DrawMaterialToRenderTarget`,
- `GetTransmittanceLUT()` pour la relire.

**État réel à documenter honnêtement** : `bUseLUTModel` n'est lu nulle part dans le C++ ; il ne bascule sur aucun chemin de rendu. `RegenerateLUTs()` ne bake que la **transmittance** LUT (le `TransmittanceLUTMaterial` doit être assigné à la main ; le master `MM_AtmoScape_TransmittanceLUT` existe dans `Content/Materials/LUT`, comme `MM_AtmoScape_MultiScatterLUT`, `MM_AtmoScape_Sky` et `M_AtmoScape_PostProcess`, mais aucun n'est chargé/piloté en C++). La multiple-scattering LUT et le composite post-process (`AtmoMultiScatterFromUV`, `AtmoRenderSky`, `AtmoCompositePP`) sont écrits en HLSL mais n'ont **aucun pilote C++**. C'est de l'infrastructure préparée, pas un renderer actif.

### 2.3 Décisions structurantes

- **Client-only** : le constructeur de `AAtmoScape` et `OnConstruction` retournent tôt si `NetMode == NM_DedicatedServer`. `RegenerateLUTs()` aussi. L'atmosphère n'existe que sur les clients ; rien n'est répliqué (cf. §5).
- **Tick en éditeur** : `ShouldTickIfViewportsOnly()` renvoie `true` et `PrimaryActorTick.bCanEverTick = true`, pour que l'aperçu réagisse dans le viewport. Le travail lourd reste néanmoins dans `OnConstruction`/`UpdateScale` (déclenché à la (re)construction), pas dans `Tick`.
- **Paramètres en km, conversion en cm** : `UnitConversion = 100000` (km→cm), `Vec_PlanetScale = PlanetRadius * 2000` pour l'échelle du mesh.
- **Source lumineuse surchargeable** : si `LightSource` est défini, sa position est poussée comme `LightPosition` et `CustomLightSource=1` ; sinon le shader retombe sur la directional light de la scène.

---

## 3. Carte des composants

| Élément | Type | Rôle |
| --- | --- | --- |
| `AAtmoScape` | `UCLASS` (`AActor`) | Actor principal : empile les coquilles atmosphériques, calcule l'échelle et pousse les paramètres aux MID. Cœur du plugin. |
| `AStarfieldScape` | `UCLASS` (`AActor`) | Sphère de starfield à très grand rayon (`SM_StarfieldMesh`, `MM_StarfieldScape`) ; expose intensité du soleil, diamètre angulaire, intensité/atténuation/échelle du champ d'étoiles. |
| `UAtmoScapeBodyDataAsset` | `UCLASS` (`UPrimaryDataAsset`) | Preset data-driven d'une atmosphère de corps céleste ; ses champs miroitent les propriétés tunables d'`AAtmoScape`. `LightSource` exclu volontairement (référence de scène par instance). |
| `UMyEditorUtilityWidget` | `UCLASS` (`UEditorUtilityWidget`) | Widget utilitaire éditeur : fait tourner la/les directional light(s) (cycle jour/nuit d'aperçu). Module `AtmoScapeEditor`. |
| `AtmoScapeEditor` | classe C++ statique (non-UObject) | Helpers éditeur : `GetViewPortCameraPosition()`, `IsInViewPort()`. Module `AtmoScapeEditor`. |
| `FAtmoScapeModule` | `IModuleInterface` | Module runtime ; à `StartupModule` mappe `Shaders/` sur le chemin virtuel `/Plugin/AtmoScape`. |
| `FAtmoScapeEditorModule` | `IModuleInterface` | Module éditeur ; enregistre le `FSlateStyleSet` (icônes/thumbnails des classes `AtmoScape` et `StarfieldScape`). |
| `LogAtmoScape` | catégorie de log | Catégorie dédiée (`DECLARE_LOG_CATEGORY_EXTERN` dans `AtmoScape.h`). |

> Il n'existe **aucun** `USTRUCT`/`UENUM`/`UINTERFACE` exposé par ce plugin. `FAtmoParams` est une `struct` purement HLSL (dans `AtmosphereCommon.ush`), pas un USTRUCT C++.

### Shaders (`.ush`, dans `Shaders/Private`)

| Fichier | Rôle |
| --- | --- |
| `AtmoScape.ush` | Renderer shell de production : `AtmoIntegrate` (raymarch single-scatter Rayleigh+Mie), wrappers `AtmoInScatter` / `AtmoSkylight` / `AtmoAbsorption` / `AtmoAirglow`, et helpers d'occlusion (`AtmoSceneDistanceAlongView`, `AtmoDistanceFieldShadow` via Global Distance Field). |
| `AtmosphereCommon.ush` | Renderer LUT clean-room (Phase 2) : profils de densité, optical depth, transmittance LUT, multiple-scattering LUT (Hillaire), `AtmoRenderSky`, `AtmoCompositePP`. |
| `VolumetricCloudFunctions.ush` | Fonctions de nuages volumétriques (contenu « Bonus », hors cœur scattering). |

---

## 4. Flux de données et cycle de vie

### Init / construction (`AAtmoScape`)

1. **Constructeur** : early-return si dedicated server. Crée `Root` (`USceneComponent`) puis les 5 `UStaticMeshComponent` via la macro `CREATE_MESH_COMPONENT` (no-collision, no-shadow, no-overlap). Charge les static meshes (`SM_AtmosphereMeshHD`, `SM_SpaceAtmosphereMesh`, `SM_AtmosphereMesh`, `SM_OutterAtmosphereMesh`) et les master materials (`MM_PlanetaryAtmo`, `MM_PlanetarySkylight`, `MM_Absorbtion`, `MM_PlanetaryOutterAtmo`) par `ConstructorHelpers::FObjectFinder`. Masque `SpacePlanetaryAtmoMesh`.
2. **`FAtmoScapeModule::StartupModule`** (au chargement du module, `PostConfigInit`) : mappe `Shaders/` → `/Plugin/AtmoScape` pour que les `#include` des Custom nodes résolvent.
3. **`OnConstruction`** : early-return si dedicated server. Crée les `UMaterialInstanceDynamic` (un par mesh/master), initialise `SunlightIntensity`, applique le preset si `bAutoApplyBodyData && IsValid(BodyData)`, met à jour les baselines (`LastPlanetRadius`/`LastAtmosphereHeight`), puis appelle `UpdateScale()`.
4. **`BeginPlay`** : se contente de resynchroniser les baselines `LastPlanetRadius`/`LastAtmosphereHeight` (l'appel à `UpdateScale()` y est commenté).

### Runtime — `UpdateScale()`

Pivot du système, déclenché à chaque (re)construction :

1. **Re-scale relatif** : si `PlanetRadius` ou `AtmosphereHeight` a changé depuis la dernière baseline, applique un `SizeCoef` à `RayleighHeight`, `MieHeight`, `AtmosphereHeight` pour garder des proportions cohérentes.
2. **Géométrie** : `PlanetScale = PlanetRadius * 100000` (cm), `AtmosRadius = PlanetScale + AtmosphereHeight*100000`, `AtmoScale = AtmosRadius / PlanetScale` ; chaque mesh est mis à `RelativeScale3D = AtmoScale * Vec_PlanetScale`.
3. **Longueur de segment** : `LightSegLength` dérivé de la profondeur optique normalisée (`sqrt(AtmoScale²-1)*2*PlanetScale / LightSamplesCount`).
4. **Coefficients** : `MultiScateringScaled = (MultiScatering*0.05)/Vec_PlanetScale.Y` ; `MieScatteringCoef`, `RayleighScatteringCoef`, et `RayleighOzoneScateringCoef = ((Absorption*OzoneContribution)+RayleighScattering)*MultiScateringScaled` ; `AtmoBackHSV` dérivé du Rayleigh (B=0).
5. **Push** : `UpdateMaterialParameters()` est appelé pour les 5 MID, écrivant les scalaires (`AtmosRadius`, `EarthRadius`, `LightSegLength`, `ScaleHeight_R/M`, `Mie_G`, `CameraSamples`, `LightSamples`, `SkylightIntensity`, `SkylightShadow`, `StartDistanceAO`, `StepsNumAO`, `AirGlowIntensity`, `AtmosOpacity`, `ParticulateIntensity`, `CameraSamples_Recip`…) et vecteurs (`coef_M`, `coef_R`, `coef_RO`, `OutterColor`, `insideColor`, `AtmoBack`, et `LightPosition` si `LightSource`).
   - **Conversion `MiePhase` → `Mie_G`** : `Mie_G = MiePhase*1.55 - MiePhase³*0.55` (Cornette-Shanks mapping).

### Presets — `ApplyBodyData()` / `ApplyBodyDataValues()`

`ApplyBodyDataValues()` recopie les ~20 champs de `BodyData` dans les propriétés de l'actor et resynchronise les baselines (pour que `UpdateScale` ne dérive pas d'un delta périmé). `ApplyBodyData()` (BlueprintCallable + `CallInEditor`) l'invoque puis appelle `UpdateScale()` si les MID existent déjà. Appelé automatiquement en construction si `bAutoApplyBodyData`.

### Chemin LUT — `RegenerateLUTs()`

`CallInEditor` / BlueprintCallable. Early-return si dedicated server ou `TransmittanceLUTMaterial` non assigné. Crée/redimensionne `TransmittanceLUT_RT` (`RTF_RGBA16f`, clampé 16–512 × 16–256), crée le MID transmittance, pousse les paramètres en km (`BottomRadius`, `TopRadius`, `RayleighScaleHeight`, `MieScaleHeight`, `MiePhaseG`, `RayleighScattering`, `MieScattering`/`MieExtinction`, `AbsorptionExtinction`, `OzoneCenter`/`OzoneWidth`), et `DrawMaterialToRenderTarget`.

### Teardown

Aucun teardown explicite : composants/MID détruits par GC à la destruction de l'actor. `FAtmoScapeModule::ShutdownModule` est vide (le mapping shader n'est pas démappé).

### `AStarfieldScape`

Tick désactivé (`bCanEverTick = false`), mesh `Static`. Le constructeur charge `SM_StarfieldMesh`/`MM_StarfieldScape` et fixe une échelle monde de `1e10`. `OnConstruction` crée le MID et pousse `SunIntensity`, `SunAngularDiameter`, `StarFieldIntensity`, `StarfieldDecrease`, `StarfieldTileScale`.

---

## 5. Réplication / réseau

**Aucune réplication.** Le plugin n'a ni `Replicated`/`ReplicatedUsing`, ni `DOREPLIFETIME`, ni RPC (`Server_`/`Client_`/`Multicast_`), ni `GetLifetimeReplicatedProps`. C'est un système purement cosmétique, **client-side only** :

- `AAtmoScape` (constructeur, `OnConstruction`) et `RegenerateLUTs()` retournent tôt sur `NM_DedicatedServer` : sur un serveur dédié l'actor n'a aucun composant ni matériel, ce qui est cohérent avec la convention QANGA « les serveurs dédiés n'ont pas de meshes ».
- Les render targets et MID LUT sont marqués `Transient` (données de rendu non sérialisées).

Conséquence pour un mainteneur : tout réglage d'atmosphère qui doit être cohérent entre joueurs (ex. corps céleste actif) doit être piloté en amont (placement de l'actor, choix du `BodyData`) par le système qui le spawn, pas par AtmoScape lui-même.

---

## 6. Points d'intégration

### Dépendances du plugin

- **Module `AtmoScape`** (`PrivateDependencyModuleNames`) : `Core`, `CoreUObject`, `Engine`, `Slate`, `SlateCore`, `Projects` (IPluginManager — mapping du dossier de shaders), `RenderCore` (`AddShaderSourceDirectoryMapping`). En target Editor il ajoute `AtmoScapeEditor` en dépendance publique.
- **Module `AtmoScapeEditor`** : `Core`, `CoreUObject`, `InputCore`, `Engine`, `Slate`, `SlateCore`, `Projects`, `Blutility`, `EditorScriptingUtilities`, `UMG`, `UnrealEd`.
- **`.uplugin`** : dépend du plugin `EditorScriptingUtilities`. `CanContainContent: true`.

### Dépendants dans QANGA

Aucun code C++ de `G:/QANGA/Source` ni d'un autre plugin ne référence `AAtmoScape`, `AStarfieldScape` ou `UAtmoScapeBodyDataAsset` (vérifié par grep). L'intégration au jeu se fait **au niveau contenu** : placement des actors dans les levels/Blueprints et assignation d'un `UAtmoScapeBodyDataAsset` (ex. `DA_Body_Mars`), en complément du système de planètes WorldScape. AtmoScape est donc un plugin de rendu autonome, consommé par les artistes/level designers, pas par du gameplay C++.

### Settings / CVars

- **Aucune CVar** ni `UDeveloperSettings` propre au plugin.
- Réglages exposés via les `UPROPERTY` d'`AAtmoScape` (catégories `Planet`, `Atmosphere`, `Atmosphere|Rayleigh`, `Atmosphere|Mie`, `Atmosphere|Absorption`, `Skylight`, `AirGlow`, `Overide SUN | LightSource`, `AtmoScape | Body Data`, `AtmoScape | LUT Model`) et via `UAtmoScapeBodyDataAsset`.
- Le widget éditeur `UMyEditorUtilityWidget` expose `RotationSpeed`, `SunYaw`, `SunPitch` et `ToggleAutomaticRotation()` pour piloter la rotation des directional lights en aperçu.
- Chemin virtuel shader : `/Plugin/AtmoScape` (mappé sur `Shaders/`), à utiliser dans les `#include` des Custom nodes.

---

## 7. Gotchas, invariants et pièges

- **`SpaceAtmo_Material` n'est jamais chargé.** Le constructeur charge `Atmo_Material`, `PlanetarySkylight_Material`, `Absorbtion_Material`, `Outter_Material` — mais **pas** `SpaceAtmo_Material`. Pourtant `OnConstruction` fait `SpaceAtmosphereMaterial = SpacePlanetaryAtmoMesh->CreateDynamicMaterialInstance(0, SpaceAtmo_Material)` avec un parent nul. Ce n'est pas fatal aujourd'hui parce que `SpacePlanetaryAtmoMesh` est masqué (`SetVisibility(false)` dans le constructeur et confirmé dans `UpdateScale`). Si on réactive la vue depuis l'espace, il faut d'abord charger ce master material. Le bloc historique qui basculait visibilité shell/espace selon la distance caméra est **commenté** dans `UpdateScale()` ; tous les meshes (sauf l'espace) sont forcés visibles.
- **`bUseLUTModel` est inerte.** Le flag n'est lu par aucun code C++ ; activer le « LUT Model » ne change rien au rendu. `RegenerateLUTs()` ne produit que la transmittance LUT ; le master `MM_AtmoScape_TransmittanceLUT` (et les autres masters LUT) existe dans `Content/Materials/LUT` mais n'est jamais chargé ni assigné par le C++ (il faut le brancher à la main sur `TransmittanceLUTMaterial`). Tout `AtmosphereCommon.ush` (multiple-scattering, `AtmoRenderSky`, `AtmoCompositePP`) est du code sans pilote. Ne pas présenter ce chemin comme fonctionnel.
- **Le travail se fait en `OnConstruction`, pas en `Tick`.** Bien que le tick soit activé (y compris en viewport), `UpdateScale()` n'est pas appelé par `Tick()` (il n'y a pas d'override `Tick`). Toute modification de paramètre nécessite une reconstruction de l'actor (déplacement/édition de propriété en éditeur, respawn en jeu) pour se propager aux MID.
- **`UpdateScale` mute les paramètres source.** Le re-scale relatif multiplie directement `RayleighHeight`, `MieHeight`, `AtmosphereHeight` quand `PlanetRadius`/`AtmosphereHeight` change. C'est pourquoi `ApplyBodyDataValues()` resynchronise systématiquement `LastPlanetRadius`/`LastAtmosphereHeight` : sans ça, un preset appliqué après une édition manuelle dériverait depuis un delta périmé. Respecter cet invariant si on ajoute un autre point d'entrée qui touche ces champs.
- **Conversion `MiePhase`.** Le shader attend `Mie_G` (paramètre de Henyey-Greenstein), pas `MiePhase` brut. La conversion `Mie_G = MiePhase*1.55 - MiePhase³*0.55` est faite côté C++ dans `UpdateMaterialParameters`. Ne pas réécrire `MiePhase` directement dans le matériau.
- **Coefficients en *10⁻⁶.** Les `FLinearColor` Rayleigh/Mie/Absorption sont documentés « value is *10⁻⁶ per metre » dans les tooltips ; `ScatteringUnitScale` (`0.001`, chemin LUT) est le knob de calibration km↔m. Les valeurs par défaut (`RayleighScattering`, `MieScattering`, `Absorption`) sont calibrées « accurate to earth ».
- **`MM_StarfieldScape` n'est pas piloté par `UpdateScale`.** `AStarfieldScape` est indépendant ; ses paramètres sont poussés une seule fois en `OnConstruction`, tick désactivé.
- **Encodage source.** Plusieurs fichiers `.cpp` contiennent des commentaires français en encodage non-UTF8 (caractères « � » dans `LoadAndAssignMesh`, `CREATE_MESH_COMPONENT`…). Cosmétique, mais éviter de propager.
- **Mapping shader non idempotent au shutdown.** `StartupModule` ajoute `/Plugin/AtmoScape` (gardé par un test `Contains`) mais `ShutdownModule` ne le retire pas. Sans incidence en pratique (durée de vie processus), à noter pour un hot-reload de module.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
| --- | --- |
| `AtmoScape.uplugin` | Manifeste : modules `AtmoScape` (Runtime/PostConfigInit) + `AtmoScapeEditor` (Editor/PreDefault), dépendance `EditorScriptingUtilities`, version 3.0. |
| `Source/AtmoScape/AtmoScape.Build.cs` | Règles du module runtime (deps `Engine`, `Projects`, `RenderCore`…). |
| `Source/AtmoScape/Public/PlanetaryAtmosphere.h` | Déclaration d'`AAtmoScape` : toutes les `UPROPERTY` tunables + API LUT/BodyData. |
| `Source/AtmoScape/Private/PlanetaryAtmosphere.cpp` | Cœur runtime : construction des shells, `UpdateScale`, `UpdateMaterialParameters`, `ApplyBodyData`, `RegenerateLUTs`. |
| `Source/AtmoScape/Public/AtmoScapeBodyDataAsset.h` | `UAtmoScapeBodyDataAsset` (preset par corps céleste). |
| `Source/AtmoScape/Public/StarfieldScape.h` / `Private/StarfieldScape.cpp` | `AStarfieldScape` (sphère de champ d'étoiles + disque solaire). |
| `Source/AtmoScape/Public/AtmoScape.h` / `Private/AtmoScape.cpp` | `FAtmoScapeModule` + `LogAtmoScape` ; mapping `Shaders/` → `/Plugin/AtmoScape`. |
| `Source/AtmoScapeEditor/AtmoScapeEditor.Build.cs` | Règles du module éditeur. |
| `Source/AtmoScapeEditor/Public/AtmoScapeEditor.h` / `Private/AtmoScapeEditor.cpp` | `FAtmoScapeEditorModule` : enregistrement du `FSlateStyleSet` (icônes de classes). |
| `Source/AtmoScapeEditor/Public/EditorUtils.h` / `Private/EditorUtils.cpp` | Helpers viewport (`GetViewPortCameraPosition`, `IsInViewPort`). |
| `Source/AtmoScapeEditor/Public/MyEditorUtility.h` / `Private/MyEditorUtility.cpp` | `UMyEditorUtilityWidget` (rotation auto des directional lights pour l'aperçu). |
| `Shaders/Private/AtmoScape.ush` | Renderer shell de production (raymarch in-scatter / skylight / absorption / airglow). |
| `Shaders/Private/AtmosphereCommon.ush` | Renderer LUT clean-room (Phase 2, non câblé). |
| `Shaders/Private/VolumetricCloudFunctions.ush` | Fonctions de nuages volumétriques (contenu Bonus). |
| `Content/Materials/Master/` | Masters : `MM_PlanetaryAtmo`, `MM_PlanetarySkylight`, `MM_Absorbtion`, `MM_PlanetaryOutterAtmo`, `MM_StarfieldScape`, `MM_AccuredSun`… |
| `Content/Mesh/` | Coquilles : `SM_AtmosphereMesh(HD)`, `SM_SpaceAtmosphereMesh`, `SM_OutterAtmosphereMesh`, `SM_StarfieldMesh`… |
| `Content/Data/Bodies/DA_Body_Mars.uasset` | Exemple de preset `UAtmoScapeBodyDataAsset`. |
| `Content/Tools/` | `AtmoscapeToolBar`, `SkylightRecapture` (utilitaires éditeur). |
| `Config/FilterPlugin.ini` | Filtre de packaging (modèle par défaut, sans entrée custom). |
| `Resources/Icon16.png` / `Icon64.png` | Icônes/thumbnails de classes (enregistrées par le module éditeur). |
