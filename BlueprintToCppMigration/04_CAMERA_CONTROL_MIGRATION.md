# Migration QCameraControl_Component vers C++

- **État :** terminé — propriétaire natif validé à froid et en runtime, wrapper Blueprint supprimé après zéro référence
- **Module :** QSystem (existant, pas de nouveau plugin)
- **Dernière vérification :** 2026-08-31

---

## 1. Verdict

`QCameraControl_Component` était un composant Blueprint attaché sous `QSpringArm_Component` dans `ALS_Base_CharacterBP`. Sa logique métier se réduisait à un seul contrat : quand le composant moteur est actif, aligner la rotation world du premier child scene component (le CineCamera) sur `MakeFromXZ(ControlRotation.Vector, ActorUpVector)`.

La migration crée `UQCameraControlComponent` dans QSystem et reclasse directement le nœud SCS consommé par ALS vers la classe native. Les huit imports historiques de maps ont été supprimés par resérialisation ciblée de leurs packages non courants, puis l'entrée EasyCook exacte a été retirée sans rescan. Après build froid, QATS, chargement des maps et validation runtime `L_Dev_Rz`, le wrapper vide a été supprimé. AILean n'utilise pas ce composant.

## 2. Architecture native

### 2.1 Contrat

`UQCameraControlComponent : USceneComponent` possède :

- le contrat standard `UActorComponent::SetActive/Activate` comme unique propriétaire de l'état ;
- `Activate` — résout le pawn propriétaire et le premier child, refuse explicitement l'activation si l'ownership est invalide ou sur dedicated server, puis conserve l'état actif authored même si le pawn est temporairement non possédé ;
- `TickComponent` — applique `MakeFromXZ(ControlRotation.Vector, ActorUpVector)` uniquement quand le composant est actif et contrôlé par un `PlayerController` local ;
- `OnRegister` — réimpose `bCanEverTick = true`, `bStartWithTickEnabled = false`, `bAllowTickOnDedicatedServer = false`, `TickGroup = TG_DuringPhysics` et `bAutoActivate = false` avant l'enregistrement du tick pour neutraliser les anciens defaults sérialisés ;
- `BeginPlay` — résout l'ownership, s'abonne au changement de Controller et synchronise le tick avec l'état actif réel ;
- `TickGroup = TG_DuringPhysics`.

### 2.2 Ownership de l'activation

Le booléen custom `bActive` et son setter dupliquaient le système d'activation moteur et forçaient un tick permanent même quand la caméra était inactive. Ils sont supprimés. Les chemins Blueprint vivants doivent appeler le `SetActive` standard : son activation réveille le tick et sa désactivation le coupe immédiatement. Aucun scheduler parallèle ni polling inactif n'est conservé.

### 2.3 Possession et contrôle local

L'état `IsActive()` reste l'intention authored du view mode. La perte temporaire de possession ne le détruit pas : le delegate natif de changement de Controller coupe seulement le tick, puis le réactive immédiatement si le même pawn redevient contrôlé par un `PlayerController` local. Le cast explicite exclut aussi les `AIController`, que l'Engine considère locaux en standalone. Les pawns distants, non possédés et AI ne produisent donc aucun travail caméra par frame.

### 2.3 Exclusion dedicated server

Le tick porte `bAllowTickOnDedicatedServer = false` avant son enregistrement. `Activate` refuse aussi le mode dedicated ; le composant ne peut donc ni s'activer ni s'enregistrer comme ticker côté serveur.

## 3. Suppression du wrapper Blueprint

### 3.1 Transition

`QCameraControl_Component` a été reparenté de `SceneComponent` vers `QCameraControlComponent` pendant la transition. Après reparent :

- `0` variable, `0` fonction, `0` macro ;
- `3` nœuds morts dans EventGraph (K2Node_IfThenElse_0, K2Node_CallFunction_0, K2Node_CallFunction_1) — appels à `Init`/`Update` supprimés, boucle sans point d'entrée.

### 3.2 Cleanup final

Les `3` nœuds morts ont été supprimés atomiquement. Le wrapper possédait alors `0` variable, fonction, événement ou nœud et compilait à `0` erreur / `0` warning. Une fois ses références Asset Registry et EasyCook tombées à zéro et les gates froid/runtime fermés, l'asset a été supprimé. Le `ClassRedirect` reste le seul contrat de compatibilité pour les copies historiques non versionnées.

## 4. Consommateurs

### 4.1 ALS_Base_CharacterBP

`QCameraControl_Component1` utilise directement `/Script/QSystem.QCameraControlComponent`, reste attaché sous `QSpringArm_Component` et conserve le CineCamera comme child index 0. Son template sérialise `bAutoActivate=false`, `TickGroup=TG_DuringPhysics`, `bCanEverTick=true` et `bAllowTickOnDedicatedServer=false`. Les trois chemins vivants utilisent maintenant le contrat moteur :

| Graph | Node | Valeur | Trigger |
|---|---|---|---|
| OnViewModeChanged | `SetActive(false)` | false | Third Person |
| OnViewModeChanged | `SetActive(true)` | true | First Person |
| EventGraph | `SetActive(ToNear)` | dynamique | Sortie proximité du spring arm |

Les liaisons exec, booléennes et de cible existantes sont conservées exactement. Le setter custom et le `Set bActive` mort ont été supprimés. `ALS_Base_CharacterBP` et le wrapper compilent à `0` erreur / `0` warning après reclassement.

### 4.2 Maps et EasyCook

Les `8` maps contenaient encore un import historique du composant wrapper dans leurs instances sérialisées. La sauvegarde forcée de chaque package `UWorld`, sans ouvrir la map courante, a resérialisé ces instances avec la classe native ; chaque map a disparu immédiatement de la liste des referencers. L'unique entrée `DA_EasyCookSeed_QANGA` correspondante a ensuite été retirée avec un contrat `expected_count=1`, sans rescan. Après suppression du wrapper et redémarrage froid, les huit maps chargent encore avec `0` Load Error. Comme ces maps sont exclues du dépôt racine, un `ClassRedirect` tracked couvre aussi toute copie sérialisée antérieure ; les anciens `PropertyRedirects` vers des champs supprimés ont été retirés.

### 4.3 AILean

`ALS_Base_CharacterBP_AILean` n'a pas de composant QCameraControl. Non concerné.

## 5. Gates de validation

- [x] Compile C++ et chargement froid de la nouvelle UCLASS
- [x] Suppression des `3` nœuds morts EventGraph du wrapper BP
- [x] Remplacement des trois chemins d'activation vivants par `UActorComponent::SetActive`
- [x] Reclassement atomique du composant SCS vers la classe native
- [x] Compile wrapper BP à `0` erreur / `0` warning
- [x] Compile ALS_Base_CharacterBP à `0` erreur / `0` warning
- [x] Template ALS sérialisé sans auto-activation ni tick dedicated server
- [x] Resérialisation ciblée des `8` maps et suppression de l'entrée EasyCook exacte
- [x] Wrapper à zéro référence
- [x] `ClassRedirect` tracked pour les copies de maps non versionnées
- [x] Compile Live Coding du gate player-local/possession réfléchi
- [x] QATS : inactive, non possédé, possess/unpossess/repossess, désactivation authored et exclusion AI
- [x] Recompile et QATS du hardening `OnRegister` contre les defaults Blueprint historiques
- [x] Build et chargement froid du gate réfléchi et du `ClassRedirect`
- [x] PIE L_Dev_Rz : caméra first-person active, rotation alignée sur control rotation + up vector
- [x] PIE L_Dev_Rz : caméra third-person inactive (pas de rotation forcée)
- [x] PIE L_Dev_Rz : transition first/third via view mode change
- [x] PIE L_Dev_Rz : transfert de possession coupe le tick sans perdre l'état authored, retour pawn le reprend
- [x] Message Log `stop_pie` à `0` erreur / `0` warning sur une session propre
- [x] Aucun writer concurrent : le chemin spring-arm ne règle la rotation que sur la branche où le natif vient d'être désactivé
- [x] Suppression du wrapper BP après preuve runtime et zéro référence

## 6. Ce qui reste Blueprint

- Tuning du spring arm, arm length, socket offset ;
- Timelines d'interpolation de view mode ;
- Logique de view mode change (appels standards à `SetActive`) ;
- Présentation et FX caméra.
