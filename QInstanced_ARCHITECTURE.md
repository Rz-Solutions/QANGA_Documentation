# QInstanced — Architecture

> Plugin runtime (`LoadingPhase: EarliestPossible`) qui remplace les composants d'instanced static mesh d'Unreal par des variantes à **création de corps physiques différée et budgétée**. But : éliminer les pics de frame provoqués par la création massive de bodies de collision quand on ajoute des milliers d'instances d'un coup (foliage, props, WorldScape).

---

## 1. But d'usage

Ajouter des instances à un `UInstancedStaticMeshComponent` (ISM) ou `UHierarchicalInstancedStaticMeshComponent` (HISM) déclenche, lors de la création de l'état physique, la construction d'un **corps de collision par instance**. Sur QANGA, où l'on peuple des surfaces planétaires entières avec des dizaines de milliers d'instances, faire cela **synchroniquement** dans la même frame provoque un hitch visible (parfois plusieurs centaines de ms).

`QInstanced` résout ce problème en **différant** la création des corps physiques : les composants `UQInstanced_ISMComponent` / `UQInstanced_HISMComponent` n'instancient pas immédiatement leurs bodies, ils s'enregistrent dans une file gérée par un `UWorldSubsystem`. Ce subsystem traite la file frame par frame avec un **budget temps configurable** (`BodyCreationTimeLimitMS`, 0.5 ms par défaut), de sorte que le coût est étalé sur plusieurs frames au lieu d'être encaissé d'un seul bloc.

Ce n'est **pas** un système de rendu ou de culling alternatif : le rendu des instances reste celui des composants UE natifs (les classes héritent directement de `UInstancedStaticMeshComponent` / `UHierarchicalInstancedStaticMeshComponent`). La seule chose interceptée est le **moment** de création des corps de collision.

---

## 2. Vue d'ensemble / décisions d'architecture

- **Héritage direct des composants UE.** `UQInstanced_ISMComponent` étend `UInstancedStaticMeshComponent`, `UQInstanced_HISMComponent` étend `UHierarchicalInstancedStaticMeshComponent`. On garde toute la machinerie de rendu/instancing native et on n'override que `OnCreatePhysicsState`, `OnComponentDestroyed`, `AddInstance(s)`, `RemoveInstance(s)`, `ClearInstances`.
- **Subsystem comme ordonnanceur.** Un seul `UQInstanced_SubSystem` (par monde) tient la liste des composants en attente (`PendingDatas`) et exécute le travail dans un budget temps. Les composants ne se créent pas tout seuls ; ils délèguent au subsystem.
- **Tick custom, pas le tick d'acteur.** Le subsystem n'utilise pas `Tick()` standard : il enregistre deux `FTickFunction` maison (`FCustomTickFunction`) dans le groupe `TG_PostUpdateWork`, l'une pour la création de corps (`Tick_Pending`), l'autre pour le ré-ordonnancement de la file (`Tick_Order`). Cela permet de choisir précisément le ticking group et l'intervalle, indépendamment d'un acteur hôte.
- **Configuration centralisée et data-driven.** Tous les seuils sont dans `UQInstanced_Settings` (`UDeveloperSettings`, `config=Game`), lus à l'initialisation via `LoadSettings()`. Les champs de réglage sont dupliqués en membres privés du subsystem (copie à l'init).
- **Pas de réplication.** La création de corps physiques est purement locale ; rien n'est répliqué. Le système est même **désactivé sur dedicated server** par défaut (cf. §7).

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQInstanced_SubSystem` | `UWorldSubsystem` | Ordonnanceur : file `PendingDatas`, ticks custom, création de corps budgétée, délégués de progression |
| `UQInstanced_ISMComponent` | `UInstancedStaticMeshComponent` (`BlueprintSpawnableComponent`) | ISM à corps différé ; override Add/Remove/Clear + `OnCreatePhysicsState` ; champ `CullingScale` |
| `UQInstanced_HISMComponent` | `UHierarchicalInstancedStaticMeshComponent` (`BlueprintSpawnableComponent`) | HISM à corps différé ; mêmes overrides |
| `UQInstanced_Settings` | `UDeveloperSettings` (`config=Game`) | Réglages projet (catégorie `QInstanced`) ; accès statique `Get()` |
| `FQInstanced_PendingData` | `USTRUCT` | Entrée de file : pointeur vers un ISM **ou** un HISM en attente |
| `FQInstanced_Option` | `USTRUCT` (`BlueprintType`) | Option exposée : `Enabled_DeferredBody` |
| `FQInstanced_SubSystem::FCustomTickFunction` | `FTickFunction` (struct interne) | Tick maison redirigeant vers une méthode membre via pointeur de fonction |

**Délégués exposés (`BlueprintAssignable`)** sur le subsystem :

| Délégué | Signature | Émis quand |
|---|---|---|
| `QInstanced_StartPending` | `()` | La file passe de vide à non-vide (début d'un lot) |
| `QInstanced_UpdatePending` | `(int32 Pending)` | À chaque palier de traitement, avec le nombre restant |
| `QInstanced_EndPending` | `()` | La file se vide (lot terminé) |

---

## 4. Flux de données et cycle de vie

1. **Initialisation** — `UQInstanced_SubSystem::Initialize` puis `OnWorldBeginPlay` : appel de `LoadSettings()` (copie des valeurs de `UQInstanced_Settings`), puis enregistrement des ticks custom via `Register_CustomTick()`. Sur dedicated server, si `EnabledOnDedicated == false`, le subsystem ne s'arme pas.
2. **Ajout d'instances** — un `UQInstanced_ISMComponent`/`HISMComponent` reçoit `AddInstance(s)`. Le composant ajoute l'instance côté rendu mais **diffère** la création du corps physique ; lors de `OnCreatePhysicsState` (ou via son drapeau `OnPendingCreateBody`) il s'enregistre auprès du subsystem (`QInstanced_AddISMComponentToPending` / `QInstanced_AddHISMComponentToPending`), qui l'ajoute à `PendingDatas`.
3. **Traitement budgété** — sur `Tick_Pending` (`TG_PostUpdateWork`), le subsystem itère `PendingDatas`, appelle `UpdateBodyCreation()` sur chaque composant valide (création incrémentale des corps via `InternalCreateBody`), et **s'arrête dès que le temps écoulé dépasse `BodyCreationTimeLimitMS`** (`FPlatformTime::Seconds()` comparé à `BodyCreationTimeLimitMS * 0.001`). Les entrées terminées sont retirées (`RemoveAt`), et `QInstanced_UpdatePending` est diffusé avec le reste.
4. **Ré-ordonnancement** — `Tick_Order` (intervalle `OrderTickInterval`) réorganise `PendingDatas` quand il reste plus de 2 entrées, pour prioriser/regrouper le travail restant.
5. **Fin de lot** — quand `PendingDatas` se vide, `QInstanced_EndPending` est diffusé.
6. **Destruction** — `OnComponentDestroyed` sur un composant et `Deinitialize` sur le subsystem nettoient la file (`ClearDatas` / `PendingDatas.Empty()`), évitant de traiter des pointeurs invalides.

---

## 5. Points d'intégration

- **Dépendances module** : `Core`, `DeveloperSettings` (public) ; `CoreUObject`, `Engine`, `Slate`, `SlateCore` (privé). Aucune dépendance vers d'autres plugins QANGA — `QInstanced` est une brique bas niveau autonome.
- **Consommateurs** : tout système qui peuple massivement des ISM/HISM (foliage, props procéduraux, instanciation liée à WorldScape) doit utiliser `UQInstanced_ISMComponent` / `UQInstanced_HISMComponent` à la place des composants natifs pour bénéficier de l'étalement.
- **Réglages** : exposés sous *Project Settings → QInstanced* (`UQInstanced_Settings`). Les valeurs sont lues une seule fois à l'init (`LoadSettings`) ; voir §7.
- **UI de progression** : les trois délégués (`StartPending`/`UpdatePending`/`EndPending`) permettent à un écran de chargement ou un widget de debug d'afficher l'avancement de la création des corps.

---

## 6. Gotchas, invariants et pièges

- **Désactivé sur dedicated server par défaut.** Si `EnabledOnDedicated == false` (défaut), le subsystem ne s'arme pas sur `NM_DedicatedServer`. C'est cohérent avec le fait que QANGA ne construit pas la géométrie de collision côté serveur dédié — mais si un usage serveur dépend des corps de collision instanciés, il faut activer ce drapeau.
- **Settings lus une seule fois.** `LoadSettings()` copie les valeurs de `UQInstanced_Settings` dans les membres du subsystem à l'initialisation. Modifier les settings en cours de session n'a pas d'effet tant que le subsystem n'est pas réinitialisé.
- **Budget temps en millisecondes.** `BodyCreationTimeLimitMS` est exprimé en **ms** mais comparé à un delta en secondes (`* 0.001` dans le code). Une valeur trop basse étale la création sur beaucoup de frames (corps de collision absents plus longtemps) ; trop haute réintroduit des hitches.
- **Validité des pointeurs.** La file stocke des pointeurs bruts vers les composants ; le traitement teste `IsValid(...)` avant chaque création. Détruire un composant en cours de traitement est géré, mais ne contournez pas `OnComponentDestroyed`.
- **Corps de collision retardés ≠ rendu retardé.** Les instances sont visibles immédiatement ; seule la **collision** apparaît progressivement. Du code qui trace/teste la collision juste après l'ajout peut rater des instances dont le corps n'est pas encore créé.

---

## 7. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QInstanced/Public/QInstanced_SubSystem.h` / `Private/QInstanced_SubSystem.cpp` | Ordonnanceur, file `PendingDatas`, ticks custom, budget temps |
| `Source/QInstanced/Public/Components/QInstanced_ISMComponent.h` / `.cpp` | ISM à corps différé |
| `Source/QInstanced/Public/Components/QInstanced_HISMComponent.h` / `.cpp` | HISM à corps différé |
| `Source/QInstanced/Public/QInstanced_Settings.h` / `.cpp` | `UDeveloperSettings` (catégorie projet `QInstanced`) |
| `Source/QInstanced/Public/Struct/QInstanced_Struct.h` | `FQInstanced_Option` |
| `Source/QInstanced/Public/QInstanced.h` / `Private/QInstanced.cpp` | Module (`IModuleInterface`) |
| `QInstanced.uplugin` | Descripteur : module Runtime, `LoadingPhase: EarliestPossible` |
