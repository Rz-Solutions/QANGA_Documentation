# GravityScape — Architecture

Plugin runtime de **gravité directionnelle par zones** (auteur : Iolacorp). Module unique `GravityScape` (Type `Runtime`, `LoadingPhase: Default`). Le système est piloté par un `UWorldSubsystem` qui agrège des sources de gravité placées dans le niveau et pousse, à fréquence réduite, un vecteur de gravité local à chaque acteur abonné.

> État : système fonctionnel et consommé en production par QAI et QPolice, mais **uniquement comme source de gravité analytique pour le calcul du « local-up »** (voir §6). La voie « application directe de force physique » (`AutoSet_GravityForceTo_PrimitiveComponent`, `GS_SetGlobalGravityPhysics`) existe dans le code mais n'est pas le chemin nominal utilisé par les pawns du jeu.

---

## 1. But d'usage

QANGA est un jeu à planètes entières : le « bas » n'est pas une constante `(0,0,-981)` mondiale, il dépend de l'endroit où se trouve l'acteur. GravityScape résout le problème concret suivant : **donner, pour une position donnée dans le monde, le vecteur de gravité qui s'y applique**, à partir de volumes/sources de gravité posés par les designers (gravité ponctuelle sphérique, gravité de boîte, gravité directionnelle).

Deux usages distincts cohabitent dans le code :

1. **Source de direction (« local-up »)** — c'est l'usage réel en production. Un appelant interroge `UGravityScape_SubSystem::GS_GetGravityAtLocation(Location)`, normalise et inverse le résultat (`-G.GetSafeNormal()`) pour obtenir l'orientation « haut » locale, qui sert ensuite à l'orientation des pawns/véhicules et à la projection des déplacements. C'est ainsi que QAI (drones, vaisseaux, flowfield) et QPolice consomment le plugin.
2. **Application de gravité aux corps physiques** — le `UGravityScapeInstanceComponent` peut diffuser le vecteur de gravité final par delegate (`NewGravity_Vector`) et/ou l'appliquer en `AddForce` sur les `UPrimitiveComponent` simulant la physique de son acteur ; le subsystem peut aussi écraser la gravité globale du `FPhysScene` (`GS_SetGlobalGravityPhysics`). Ces chemins existent mais ne sont pas le mécanisme de locomotion des pawns du jeu (aucune intégration `NinjaCharacter`/`CharacterMovement` dans ce plugin).

Le système ne remplace pas la gravité radiale planétaire de WorldScape : chez les consommateurs, GravityScape est explicitement traité comme une **source secondaire** (le local-up de WorldScape/sol est préféré quand il est disponible — voir les commentaires dans `QAI_SpaceshipBehavior.cpp` / `QSpaceshipMovementInterface.cpp`).

---

## 2. Vue d'ensemble / décisions d'architecture

Forme du système :

- **Un `UWorldSubsystem`** (`UGravityScape_SubSystem`) joue le rôle de registre central et de moteur de calcul. Il détient deux listes parallèles :
  - les **sources** de gravité (`Gravitys` : tableau de `UGravityScapeGravity*`) et leur copie de données aplatie (`GravityData` : `TArray<FGS_Gravity_Data>`),
  - les **consommateurs** (`Components` : tableau de `UGravityScapeInstanceComponent*`).
- **Deux `UActorComponent` Blueprintables** :
  - `UGravityScapeGravity` = **émetteur** (« une source de gravité »). À `BeginPlay`, il s'enregistre auprès du subsystem ; ses propriétés (forme, force, rayons, priorité) sont copiées dans un `FGS_Gravity_Data`.
  - `UGravityScapeInstanceComponent` = **récepteur** (« cet acteur subit la gravité »). À `BeginPlay`, il s'enregistre ; à chaque batch le subsystem lui réinjecte sa gravité globale + override via `GS_Set_UpdateGravityData`.
- **Séparation données / objets pour le calcul parallèle.** Le calcul lourd ne touche jamais les UObject : il travaille sur des POD (`FGS_Gravity_Data`, `FGS_Component_Data`). C'est ce qui permet le mode `ParallelFor`/`AsyncParallelFor`. Le code source vit dans `FGS_Gravity_Data::ComputeGravity()` (méthode `const` pure sur la struct).
- **Tick custom à fréquence réglable.** Le subsystem n'utilise pas un tick d'acteur classique : il enregistre des `FTickFunction` custom (`FCustomTickFunction`) sur `World->PersistentLevel`, dans des groupes de tick précis :
  - mise à jour des gravités des récepteurs (`UpdateGravity`) en `TG_PostUpdateWork`, intervalle = `UpdateFrequency` (0.25 s par défaut),
  - rafraîchissement des données de sources si elles bougent (`CheckUpdateGravity`) en `TG_PostUpdateWork`, intervalle = `CheckUpdateGravity_Interval`, **seulement si `GravityCanUpdate`**,
  - mise à jour de la gravité physique globale (`UpdateGlobalGravity`) en `TG_PrePhysics`, chaque frame.
- **Trois stratégies d'update sélectionnables** via `EGS_UpdateType` (`GameThreadDirect` / `ParallelFor` / `AsyncParallelFor`), pilotées par le réglage `Update_Gravity_Type`.
- **Configuration centralisée** via un `UDeveloperSettings` (`UGravityScapeSettings`, catégorie projet « GravityScape »), chargée une fois à l'`Initialize` du subsystem dans `LoadSettings()`.

Implémenté vs intention (honnêteté) :

- `UGravityScape_Library` est une `UBlueprintFunctionLibrary` **vide** — coquille déclarée, sans aucune fonction. À supprimer ou à remplir.
- Le bloc `Initialize` du subsystem qui distingue `EnabledOnDedicated` / `EnabledOnClient` aboutit aux **trois branches au même appel** `Register_CustomTick()` : aujourd'hui ces flags ne gatent pas réellement l'enregistrement du tick (voir §7).
- Le mode `AsyncParallelFor` capture des `TArray` **locaux par référence** dans deux `AsyncTask` imbriqués (voir §7, piège réel).
- `GlobalGravity_Inter`/`_Target` et `UpdateGlobalGravity` poussent la gravité au `FPhysScene` via `SetUpForFrame` ; `CanUpdateGlobalGravity` vaut `true` par défaut dans le subsystem mais le réglage par défaut est `false` — c'est le réglage qui fait foi après `LoadSettings()`.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UGravityScape_SubSystem` | `UWorldSubsystem` (UCLASS) | Registre central + moteur de calcul. Enregistre/désenregistre sources et récepteurs, calcule la gravité par batch, gère les ticks custom et la gravité physique globale. |
| `UGravityScapeGravity` | `UActorComponent` (UCLASS, Blueprintable, BlueprintSpawnableComponent) | **Émetteur** : définit une source de gravité (forme, force, priorité). Ne tick pas ; se marque « dirty » quand son transform change. |
| `UGravityScapeInstanceComponent` | `UActorComponent` (UCLASS, Blueprintable, BlueprintSpawnableComponent) | **Récepteur** : reçoit du subsystem sa gravité globale/override, interpole vers une gravité finale, diffuse par delegate, applique optionnellement la force physique. Tick chaque frame. |
| `UGravityScapeSettings` | `UDeveloperSettings` (UCLASS, `config=Game`, `defaultconfig`) | Réglages projet (catégorie « GravityScape ») : activation, fréquence, type d'update, gravité par défaut, gravité globale. |
| `UGravityScape_Library` | `UBlueprintFunctionLibrary` (UCLASS) | Coquille vide (aucune fonction). |
| `EGS_UpdateType` | `UENUM(BlueprintType)` | Stratégie d'update : `GameThreadDirect`, `ParallelFor`, `AsyncParallelFor`. |
| `FGS_Component_Data` | `USTRUCT(BlueprintType)` | POD d'un récepteur pour le calcul : `Location`, `GlobalGravity`, `OverrideGravity`, `IsOverride`. |
| `FGS_Gravity_Data` | `USTRUCT(BlueprintType)` | POD d'une source pour le calcul + la logique de gravité elle-même (`ComputeGravity(ComponentLocation)`). |
| `FGravityScapeModule` | `IModuleInterface` | Module runtime ; `StartupModule`/`ShutdownModule` vides. |
| `FCustomTickFunction` | `FTickFunction` (struct interne au subsystem) | Pointeur sur fonction membre `void(float)` exécuté dans un groupe de tick donné à un intervalle donné. |

---

## 4. Flux de données et cycle de vie

**Initialisation (`UGravityScape_SubSystem::Initialize`)**
1. `LoadSettings()` recopie tous les champs de `UGravityScapeSettings::Get()` (le CDO) dans les membres privés du subsystem.
2. Si `Enabled`, le subsystem appelle `Register_CustomTick()` qui enregistre trois `FCustomTickFunction` sur `World->PersistentLevel` :
   - `UpdateGravity_TickFunction` → `UpdateGravity`, `TG_PostUpdateWork`, intervalle `UpdateFrequency`.
   - (si `GravityCanUpdate`) `UpdateGravityData_TickFunction` → `CheckUpdateGravity`, `TG_PostUpdateWork`, intervalle `CheckUpdateGravity_Interval`.
   - `UpdateGlobalGravity_TickFunction` → `UpdateGlobalGravity`, `TG_PrePhysics`, intervalle `0.0` (chaque frame).

**Enregistrement des sources (`UGravityScapeGravity`)**
- À `BeginPlay` : récupère l'owner, met `Valid = true`, s'abonne à `RootComponent->TransformUpdated` si `Auto_GetTransformChange`, puis appelle `GS_RegistreGravity(this)`.
- `GS_RegistreGravity` ajoute la source à `Gravitys` **et** construit un `FGS_Gravity_Data` via `Make_GravityData` (les deux tableaux restent indexés en parallèle ; `BoxBound = BoxExtend.SquaredLength() * 1.5`).
- Quand le transform de la source bouge, `OnTransformUpdated` met `IsDirty = true`. Le tick `CheckUpdateGravity` (si actif) recopie les champs des sources dirty dans `GravityData`.
- À `EndPlay` : `GS_UnRegistreGravity` retire la source de `Gravitys` et l'entrée correspondante de `GravityData`.

**Enregistrement des récepteurs (`UGravityScapeInstanceComponent`)**
- À `BeginPlay` : récupère l'owner, calcule `Internal_GravityInfluenceThresholdSquared`, puis `GS_RegistreComponent(this)`. Le subsystem ajoute le récepteur à `Components` et lui pousse **immédiatement** une première valeur de gravité (calcul ponctuel via `UpdateGravityAt`).
- À `EndPlay` : `GS_UnregistreComponent` le retire de `Components`.

**Runtime — calcul de gravité (tick `UpdateGravity`, intervalle `UpdateFrequency`)**
- `UpdateGravity` dispatche selon `Update_Gravity_Type` :
  - `GameThreadDirect` → `UpdateGravityComponent` (boucle séquentielle sur le GameThread),
  - `ParallelFor` → `Update_ParallelGravity` (`ParallelFor` sur les `FGS_Component_Data`, puis réinjection GameThread),
  - `AsyncParallelFor` → `AsyncUpdateGravityComponent` (`AsyncTask` hi-pri → `ParallelFor` → `AsyncTask` GameThread pour réinjecter).
- Cœur du calcul : `UpdateGravityAt(FGS_Component_Data&, const TArray<FGS_Gravity_Data>&)` :
  - pour chaque source `Blend == true` : **somme** la contribution `ComputeGravity(Location)` dans `GlobalGravity`,
  - pour chaque source `Blend == false` (override) : retient la contribution de la **plus haute** `Override_Priority` dans `OverrideGravity` et met `IsOverride = true`.
- `FGS_Gravity_Data::ComputeGravity(ComponentLocation)` :
  - source **boîte** (`IsBox`) : test rapide par `BoxBound` puis test précis `FBox::IsInsideOrOn` en espace local ; renvoie soit la gravité directionnelle transformée (`IsDirectionalGravity`), soit la gravité ponctuelle.
  - source **sphère** (sinon) : force = `GetMappedRangeValueClamped` entre `Sphere_Influence_FullRadius` et `+Sphere_Influence_EndRadius` (atténuation), direction = vers le centre de la source (ou `CustomLocationForPointGravity`).
- Si `GravityData` est vide : selon `UseDefault_GravityOnEmptyVolume`, on pousse `DefaultGravity` (par défaut `(0,0,-981.78)`) ou un vecteur nul à tous les récepteurs.
- Résultat réinjecté via `GS_Set_UpdateGravityData(Global, Override, IsOverride)`.

**Runtime — application côté récepteur (`UGravityScapeInstanceComponent::TickComponent`, chaque frame)**
- `Update_Gravity` sélectionne la cible (`Internal_ManualOverride_Gravity` > `Internal_Override_Gravity` > `Internal_Global_Gravity`) et l'interpole (`VInterpTo`, `Gravity_InterpSpeed*`) si `Gravity_UseInterp`, sinon la prend telle quelle, dans `Internal_Final_Gravity`.
- Si `AutoGet_GravityInfluence` : compare `|gravité|²` à `Internal_GravityInfluenceThresholdSquared` et broadcast `OnInfluence(bool)` au changement d'état.
- Broadcast `NewGravity_Vector(Internal_Final_Gravity)` chaque frame.
- Si `AutoSet_GravityForceTo_PrimitiveComponent` : `Update_AutoSetGravityForce` applique `AddForce(Internal_Final_Gravity)` à chaque `UPrimitiveComponent` simulant la physique de l'owner.

**Runtime — gravité physique globale (tick `UpdateGlobalGravity`, `TG_PrePhysics`, chaque frame)**
- Si `CanUpdateGlobalGravity` : interpole `GlobalGravity_Inter` → `GlobalGravity_Target` (`GlobalGravityInterpSpeed`) et appelle `FPhysScene::SetUpForFrame` avec ce vecteur et les paramètres de `UPhysicsSettings`. `GS_SetGlobalGravityPhysics(NewGravity, Direct)` règle la cible (et la valeur courante si `Direct`).

**Teardown (`Deinitialize`)**
- `UnRegister_CustomTick()` dé-enregistre les `FTickFunction` (et celle de data si `GravityCanUpdate`).

**Requête ponctuelle (sans récepteur)**
- `GS_GetGravityAtLocation(Location)` : construit un `FGS_Component_Data`, appelle `UpdateGravityAt`, renvoie `OverrideGravity` si `IsOverride`, sinon `GlobalGravity` ; renvoie `(0,0,0)` si aucune source. C'est l'API consommée par QAI/QPolice.

---

## 5. Réplication / réseau

**Le plugin ne réplique pas.** Aucun `DOREPLIFETIME`, `GetLifetimeReplicatedProps`, ni RPC `Server_/Client_/Multicast_` dans le module. Toutes les propriétés `UPROPERTY` sont locales ; le calcul de gravité est déterministe à partir des sources placées dans le niveau et tourne identiquement sur chaque instance (serveur, client, standalone).

Le seul élément réseau-conscient est dans `UGravityScape_SubSystem::Initialize`, qui inspecte `World->GetNetMode()` et les réglages `EnabledOnDedicated` / `EnabledOnClient`. **Attention** : dans l'état actuel du code, les trois branches (`NM_DedicatedServer && EnabledOnDedicated`, `NM_DedicatedServer && EnabledOnClient`, `else`) appellent toutes `Register_CustomTick()` — les flags ne désactivent donc pas réellement le subsystem sur dédié/client (voir §7).

---

## 6. Points d'intégration

**Dépendances du module (`GravityScape.Build.cs`)**
- Public : `Core`, `DeveloperSettings`, `PhysicsCore`.
- Private : `CoreUObject`, `Engine`, `Slate`, `SlateCore`.
- `.uplugin` : `CanContainContent: true`, un seul module `GravityScape` (Runtime, Default).

**Systèmes QANGA qui dépendent de GravityScape**

Le point d'entrée consommé est **`UGravityScape_SubSystem::GS_GetGravityAtLocation(Location)`**, récupéré via `World->GetSubsystem<UGravityScape_SubSystem>()`. Le motif d'usage est systématiquement « gravité → local-up » : `LocalUp = (-G).GetSafeNormal()` quand `G` n'est pas quasi-nul.

- **QAI** (`QAI.uplugin` déclare une dépendance plugin `GravityScape` ; `QAI.Build.cs` ajoute le module avec le commentaire « Gravity-aware movement on planets (local up from GravityScape) ») :
  - `QAI_FlowFieldManager::ResolveLocalUp` — GravityScape d'abord, puis fallback WorldScape (`AWorldScapeRoot`).
  - `QAI_DroneBehavior` — local-up pour projeter le déplacement horizontal des drones.
  - `QAI_SpaceshipBehavior`, `QSpaceshipMovementInterface`, `QSpaceshipObstacleAvoidanceComponent` — local-up des vaisseaux. **Les commentaires de ces fichiers documentent explicitement que GravityScape est un fallback off-planet** : le « up » fiable du sol / radial WorldScape est préféré, et certaines branches `IsNearlyZero` après l'appel GravityScape sont notées comme « dead » (le `UpVector` par défaut n'est jamais nul).
- **QPolice** (`QPolice.uplugin` + `QPolice.Build.cs` + `SpaceshipAI.Build.cs` dépendent de `GravityScape`) :
  - `QPoliceSubsystem` (plusieurs sites) et `QSpaceshipPoliceProvider` — même motif local-up pour le placement / la navigation des unités de police.

**Réglages pertinents (`UGravityScapeSettings`, Project Settings → catégorie « GravityScape »)**

| Réglage | Défaut | Effet |
|---|---|---|
| `Enabled` | `true` | Active le subsystem (gate du `Register_CustomTick`). |
| `UpdateGravityFrequency` | `0.250` | Intervalle (s) du tick de mise à jour des récepteurs. |
| `EnabledOnDedicated` | `false` | (Voir §7 — actuellement sans effet réel sur l'enregistrement du tick.) |
| `EnabledOnClient` | `true` | (Idem.) |
| `Update_Gravity_Type` | `GameThreadDirect` | Stratégie d'update (`GameThreadDirect` / `ParallelFor` / `AsyncParallelFor`). |
| `GravityCanUpdate` | `false` | Active le tick `CheckUpdateGravity` (sources mobiles). Si `false`, les sources qui bougent ne rafraîchissent pas leurs données. |
| `CheckUpdateGravity_Interval` | `1.0` | Intervalle (s) du rafraîchissement des sources dirty. |
| `CanUpdateGlobalGravity` | `false` | Active la poussée de gravité au `FPhysScene` (gravité physique globale). |
| `GlobalDefaultGravity` | `(0,0,-981.78)` | (Chargé mais non réutilisé pour initialiser la cible globale.) |
| `GlobalGravityInterpSpeed` | `16.0` | Vitesse d'interpolation de la gravité physique globale. |
| `UseDefault_GravityOnEmptyVolume` | `true` | Quand aucune source : pousser `DefaultGravity` (true) ou un vecteur nul (false). |
| `DefaultGravity` | `(0,0,-981.78)` | Gravité poussée aux récepteurs hors zone. |

Aucune CVar ni commande console n'est exposée par ce plugin.

---

## 7. Gotchas, invariants et pièges

- **Invariant d'indexation parallèle.** `Gravitys` et `GravityData` doivent rester alignés index par index : `GS_RegistreGravity` ajoute aux deux, `GS_UnRegistreGravity` retire l'entrée à l'index renvoyé par `Gravitys.Remove`, et `CheckUpdateGravity` recopie `Gravitys[i] → GravityData[i]`. Toute opération qui désynchronise ces deux tableaux casse le calcul. Idem pour `ValidComp`/`CompData` reconstruits ensemble à chaque batch.
- **`GravityCanUpdate = false` ⇒ sources figées.** Avec le défaut, le tick `CheckUpdateGravity` n'est pas enregistré : une source dont le transform bouge marque `IsDirty` mais ses `FGS_Gravity_Data` ne sont jamais rafraîchies. Pour des sources de gravité mobiles, activer `GravityCanUpdate`.
- **Les flags réseau ne gatent pas réellement.** Dans `Initialize`, les trois branches `NetMode` aboutissent au même `Register_CustomTick()`. `EnabledOnDedicated`/`EnabledOnClient` sont chargés mais n'empêchent pas l'enregistrement du tick. Ne pas compter dessus pour désactiver le subsystem côté serveur/client.
- **`AsyncParallelFor` capture des locaux par référence.** Dans `AsyncUpdateGravityComponent`, les `TArray` `ValidComp`, `CompData`, `Data` sont locaux à la fonction et capturés **par référence** (`[&...]`) dans des `AsyncTask` qui survivent à la sortie de la fonction → accès à des locaux hors de portée. Préférer `GameThreadDirect` (le défaut) ou `ParallelFor` ; ne pas activer `AsyncParallelFor` sans corriger les captures.
- **`Use_CustomLocationForPointGravity` : ordre des opérations.** Dans `ComputeGravity`, l'expression `ComponentLocation - CustomLocationForPointGravity.GetSafeNormal() * Force` normalise la **location custom** (et non le vecteur `ComponentLocation - CustomLocation`). Le résultat n'est pas « gravité dirigée vers le point custom » au sens naïf — relire la formule avant d'ajuster ce mode.
- **GravityScape est une source secondaire chez les consommateurs.** Côté QAI/QPolice, le local-up fiable vient du sol / du radial WorldScape ; GravityScape n'est interrogé qu'en fallback off-planet. Ne pas présumer qu'il pilote l'orientation des pawns sur une planète. Les commentaires QAI signalent des branches `IsNearlyZero` post-GravityScape comme **dead code** (le `UpVector` par défaut n'étant jamais nul).
- **Pas de pont `NinjaCharacter`/`CharacterMovement`.** Ce plugin ne fournit aucune intégration directe avec le movement component des personnages. L'application de gravité aux corps physiques passe soit par le delegate `NewGravity_Vector` (à câbler côté gameplay), soit par `AddForce` sur les primitives en simulation (`AutoSet_GravityForceTo_PrimitiveComponent`), soit par la gravité globale du `FPhysScene`.
- **`UGravityScape_Library` est vide.** Ne pas s'attendre à des helpers Blueprint dans cette classe ; les fonctions Blueprintables utiles sont sur le subsystem (`GS_GetGravityAtLocation`, `GS_SetGlobalGravityPhysics`) et sur `UGravityScapeInstanceComponent`.
- **Ticks enregistrés sur `PersistentLevel`.** `RegisterSingleTick` enregistre les `FTickFunction` sur `World->PersistentLevel` ; valide tant que le subsystem vit la durée du monde, mais à garder en tête lors d'un changement de niveau / streaming.
- **Logs sous `LogTemp`.** Les messages d'init et l'erreur « owner actor is not valid » sortent sous `LogTemp` (pas de catégorie dédiée), peu filtrables.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `GravityScape/GravityScape.uplugin` | Manifeste : module unique `GravityScape` (Runtime, Default), `CanContainContent`. |
| `GravityScape/Source/GravityScape/GravityScape.Build.cs` | Dépendances : `Core`/`DeveloperSettings`/`PhysicsCore` (public), `CoreUObject`/`Engine`/`Slate`/`SlateCore` (private). |
| `Source/GravityScape/Public/GravityScape.h` / `Private/GravityScape.cpp` | `FGravityScapeModule` (IModuleInterface, startup/shutdown vides). |
| `Source/GravityScape/Public/GravityScape_Struct.h` | `EGS_UpdateType`, `FGS_Component_Data`, `FGS_Gravity_Data` (+ logique `ComputeGravity`). |
| `Source/GravityScape/Public/GravityScapeSettings.h` / `Private/GravityScapeSettings.cpp` | `UGravityScapeSettings` (UDeveloperSettings) + `Get()`. |
| `Source/GravityScape/Public/GravityScape_SubSystem.h` / `Private/GravityScape_SubSystem.cpp` | `UGravityScape_SubSystem` : registre, moteur de calcul, ticks custom, gravité physique globale. |
| `Source/GravityScape/Public/Component/GravityScapeGravity.h` / `Private/Component/GravityScapeGravity.cpp` | `UGravityScapeGravity` : composant émetteur (source de gravité). |
| `Source/GravityScape/Public/Component/GravityScapeInstanceComponent.h` / `Private/Component/GravityScapeInstanceComponent.cpp` | `UGravityScapeInstanceComponent` : composant récepteur (subit la gravité, diffuse/applique). |
| `Source/GravityScape/Public/GravityScape_Library.h` / `Private/GravityScape_Library.cpp` | `UGravityScape_Library` : BlueprintFunctionLibrary actuellement vide. |
