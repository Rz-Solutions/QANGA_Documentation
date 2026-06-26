# QVehicles — Architecture

Plugin Runtime de QANGA (auteur `RzZz`) qui regroupe le **contenu des véhicules** (mannequins statiques, sons, blueprints) et un **système de sirène/gyrophare** entièrement piloté en C++. Le C++ est volontairement mince : une seule `UActorComponent` (la machine à états de la sirène) plus une `UBlueprintFunctionLibrary` d'aide qui pilote audio MetaSound, matériaux et `USpotLightComponent`. Tout le reste (modèles de véhicules, châssis, montures de gyrophares, blueprints de gameplay) est du contenu embarqué (`CanContainContent: true`).

État : implémenté et fonctionnel. Aucun autre module C++ de QANGA ne référence les types publics de ce plugin ; l'intégration se fait exclusivement côté Blueprint/contenu.

## 1. But d'usage

QANGA possède des véhicules au sol (camions, motos, pods, vélo) et des navettes. Certains véhicules sont des véhicules « d'intervention » qui ont besoin d'un comportement de **sirène à plusieurs états** crédible — celui d'un véhicule de police/secours moderne : un klaxon court, une sirène « normale », une sirène « rapide », un effet « Bwoop », et un mode de panne (« glitch ») où la sirène et les gyrophares dégénèrent.

Le problème concret résolu par le C++ n'est pas « jouer un son » (trivial en Blueprint) mais :

1. **Détection de gestes d'entrée à plusieurs taps/holds** (appui court / appui maintenu / double-tap / triple-tap) à partir d'un unique bouton « honk », avec une logique de fenêtres temporelles via des `FTimerHandle`. Reproduire cette machine à états en Blueprint serait fragile et difficile à répliquer.
2. **Pilotage des gyrophares via Custom Primitive Data** sur des `UStaticMeshComponent` (rouge/bleu, intensité + toggle), plus des `USpotLightComponent` gauche/droite, selon des motifs alternés — sans tick par frame, uniquement par timers.
3. **Synchronisation réseau** de l'état de sirène pour que les autres joueurs voient/entendent le bon état (autorité serveur + multicast).
4. Un effet de **panne audiovisuelle** (`GlitchSiren`) avec descente progressive du pitch/volume et filtre passe-bas, indépendant de la machine à états principale.

## 2. Vue d'ensemble / décisions d'architecture

Forme du système :

- **Machine à états sur le composant** (`UQVehicleSirenComponent`). L'enum `ESirenState` (`None`/`Horn`/`NormalSiren`/`FastSiren`/`Bwoop`) est la seule source de vérité de l'état audible. Les transitions sont déclenchées par deux entrées (`OnHonkInputPressed`/`OnHonkInputReleased`) et un ensemble de timers nommés.
- **Pas de tick.** `PrimaryComponentTick.bCanEverTick = false`. Toute la temporisation (délai d'activation du klaxon, délai de sirène rapide, fenêtres double/triple tap, rafales Bwoop, clignotement des feux, descente audio du glitch) passe par le `FTimerManager` du monde. C'est conforme à la règle « performance first » : aucune logique par frame.
- **Séparation composant / bibliothèque.** Le composant détient l'**état et l'intention** ; toute l'**exécution bas-niveau** (recherche de l'`UAudioComponent` par nom, application des Custom Primitive Data, gestion des `USpotLightComponent`, descente audio du glitch) vit dans `UQVehicleSirenBlueprintLibrary` en fonctions `static`. Cette bibliothèque maintient un registre statique global `TMap<TWeakObjectPtr<AActor>, FSirenState>` pour ne pas avoir à stocker l'état de clignotement/glitch sur le composant.
- **Convention par nom plutôt que par référence.** Le composant ne tient pas de pointeurs vers ses meshes/lumières/audio : il les retrouve à l'exécution par `FName` (`AudioComponentName`, `FSirenMeshConfig::MeshName`, `LeftSpotLightNames`, `RightSpotLightNames`). C'est ce qui permet à la bibliothèque d'opérer sur n'importe quel acteur sans couplage de type, mais cela impose que les noms de composants concordent exactement (voir §7).
- **Gyrophares pilotés par Custom Primitive Data**, pas par MID. Chaque `FSirenMeshConfig` mappe 4 indices de Custom Primitive Data (intensité rouge, toggle rouge, intensité bleu, toggle bleu) ; le matériau de la mesh lit ces flottants. C'est plus performant qu'instancier des matériaux dynamiques.
- **Réplication minimale** : seuls `CurrentSirenState` et `bNormalSirenToggled` sont répliqués ; les transitions visibles/audibles côté clients passent par un `NetMulticast` fiable.

Implémenté vs intendu / honnêteté sur le code réel :

- L'enum interne `EGyroPattern` propose 4 motifs (`LeftRight_Alternating`, `FrontBack_Alternating`, `Diagonal_Alternating`, `Rotating_Sequential`) mais `SelectPatternForVehicleState` n'en sélectionne réellement que **deux** : `Rotating_Sequential` en mode rapide, `LeftRight_Alternating` sinon. `FrontBack_Alternating` et `Diagonal_Alternating` sont définis et gérés dans le `switch` de `SetSirenLightMaterials` mais ne sont jamais choisis à l'exécution — du code prêt mais inactif.
- `IsVehicleParked` (vitesse < 50 u/s) bascule le clignotement en mode « rapide » quand le véhicule roule, via `ToggleSirenLights`. C'est la seule influence de l'état physique du véhicule sur les feux.
- `ActivateNormalSiren()` (privé) ne fait qu'appeler `ToggleNormalSiren()` et n'est câblé sur aucun timer ni entrée — fonction non utilisée.
- `OnHonkInputPressed`/`OnHonkInputReleased` font un `return` précoce si l'owner n'a pas l'autorité : l'enchaînement des gestes ne s'évalue donc que sur le serveur (voir §5).

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQVehicleSirenComponent` | `UCLASS` (`UActorComponent`, `BlueprintSpawnableComponent`) | Machine à états de la sirène : entrées klaxon, transitions d'état, timers de gestes, déclenche audio + feux + multicast. |
| `ESirenState` | `UENUM(BlueprintType)` (`uint8`) | États audibles : `None`, `Horn`, `NormalSiren`, `FastSiren`, `Bwoop`. |
| `FSirenStateChanged` | `DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams` | Délégué `BlueprintAssignable` (`OnSirenStateChanged`) émis à chaque transition avec `(OldState, NewState)`. |
| `UQVehicleSirenBlueprintLibrary` | `UCLASS` (`UBlueprintFunctionLibrary`) | Exécution bas-niveau : lecture/arrêt MetaSound, clignotement des feux, glitch audiovisuel, registre d'état statique, nettoyage. |
| `FSirenMeshConfig` | `USTRUCT(BlueprintType)` | Mappe une mesh de gyrophare (par `FName`) à 4 indices Custom Primitive Data (rouge/bleu intensité+toggle) et leurs valeurs d'intensité. Surcharge `operator==`/`!=`. |
| `EGyroPattern` | `enum class : uint8` (privé du .cpp, **non** réflecté) | Motifs de clignotement internes ; 2 sur 4 réellement sélectionnés. |
| `FSirenState` | `struct` (privé du .cpp, **non** réflecté) | État runtime par acteur dans la `TMap` statique : flag actif, `FTimerHandle` de clignotement, étape de motif, snapshot audio (pitch/volume d'origine), état de descente audio du glitch. |
| `FQVehiclesModule` | `IModuleInterface` | Module runtime ; `StartupModule`/`ShutdownModule` sont vides (pas de subsystem, pas d'enregistrement). |

## 4. Flux de données et cycle de vie

**Initialisation.** Le module `QVehicles` (`Default` loading phase) ne fait rien au démarrage. `UQVehicleSirenComponent` est ajouté à un blueprint de véhicule (via `BlueprintSpawnableComponent`). Dans son constructeur il désactive le tick, active la réplication (`SetIsReplicatedByDefault(true)`) et fixe les valeurs par défaut (délais, taux de clignotement, `AudioComponentName = "SirenAudioComponent"`). `BeginPlay` n'appelle que `Super`.

**Configuration (Blueprint).** L'acteur véhicule doit posséder :
- un `UAudioComponent` dont le `FName` correspond à `AudioComponentName` ;
- des `UStaticMeshComponent` de gyrophares dont les noms figurent dans `LightMeshConfigs[].MeshName`, avec leurs indices Custom Primitive Data ;
- optionnellement des `USpotLightComponent` dont les noms figurent dans `LeftSpotLightNames` / `RightSpotLightNames`.
Les sons (`HornSound`, `NormalSirenSound`, `FastSirenSound`) sont des `UMetaSoundSource` assignés en éditeur ou via les setters `SetHornSound` / `SetNormalSirenSound` / `SetFastSirenSound`.

**Runtime — résolution des gestes (serveur).**
- `OnHonkInputPressed` : enregistre `HoldStartTime`, arme `HornDelayHandle` (klaxon après `HornActivationDelay`) ; si `bNormalSirenToggled`, arme aussi `FastSirenDelayHandle` (sirène rapide après `FastSirenDelayTime`).
- `OnHonkInputReleased` : annule les timers de hold, puis décide selon l'état courant et la durée d'appui :
  - en `FastSiren` → retour `NormalSiren` ;
  - en `Horn` → `NormalSiren` si la sirène est togglée, sinon `None` + ouverture d'une fenêtre double-tap ;
  - appui court (`HoldDuration < MaxTapDuration`) → progression dans les fenêtres : simple → ouvre la fenêtre double-tap ; dans la fenêtre double-tap → `ToggleNormalSiren()` + ouvre la fenêtre triple-tap ; dans la fenêtre triple-tap → état `Bwoop` (deux rafales via `StartBwoopSecondBurst`/`EndBwoop`).

**Runtime — transition d'état.** `UpdateSirenState(NewState)` est le point central : il ignore les no-ops, arrête l'audio de l'ancien état, fixe `CurrentSirenState`, joue l'audio du nouvel état, diffuse `OnSirenStateChanged`, met à jour les feux (`UpdateSirenLights`), appelle l'événement Blueprint `OnSirenLightUpdate`, puis réplique via `MulticastUpdateSirenState`.

**Runtime — feux.** `UpdateSirenLights` (si `bAutomaticLightControl`) appelle `StartSirenLights`/`StopSirenLights` de la bibliothèque selon l'état. `StartSirenLights` crée/relance un timer répété au taux de clignotement et, à chaque tic, `ToggleSirenLights` fait avancer l'étape de motif et `SetSirenLightMaterials` applique les Custom Primitive Data aux meshes et bascule la visibilité/couleur des spotlights.

**Runtime — glitch.** `GlitchSiren` (composant → bibliothèque) coupe le clignotement normal, lance `ApplyGlitchPattern` (motifs aléatoires d'intensité), démarre `StartAudioPowerDown` (snapshot pitch/volume, filtre passe-bas, interpolation `InterpEaseOut` sur `SirenAudioPowerDownDuration = 5.0s` jusqu'à arrêt) et auto-réarme un timer à intervalle aléatoire.

**Teardown.** `DisableAllSirens` (composant) remet l'état à `None`, vide tous les `FTimerHandle` et appelle la bibliothèque, qui à son tour fait `CleanupSirenSystem`. La bibliothèque se nettoie aussi de façon défensive : `CleanupInvalidActors` purge les entrées de la `TMap` dont l'acteur (`TWeakObjectPtr`) n'est plus valide, en libérant leurs timers. `RemoveAllSirenLights` (composant) arrête les feux (via `StopSirenLights` de la bibliothèque) et vide les tableaux de configuration du composant.

## 5. Réplication / réseau

Le système est répliqué (autorité serveur).

**État répliqué** (`GetLifetimeReplicatedProps`, `DOREPLIFETIME`) :
- `CurrentSirenState` (`ESirenState`) — l'état audible courant ;
- `bNormalSirenToggled` (`bool`) — si la sirène « normale » est verrouillée en marche.

**Modèle d'autorité.** `OnHonkInputPressed` et `OnHonkInputReleased` retournent immédiatement si `!GetOwner()->HasAuthority()`. Toute la résolution des gestes et toutes les transitions d'état s'exécutent donc côté serveur. Les clients n'évaluent pas la logique d'entrée — le composant tel quel suppose que ces fonctions sont invoquées dans un contexte ayant l'autorité (typiquement le véhicule/contrôleur serveur). Il n'y a **pas** de RPC `Server_*` dans ce composant : le passage client→serveur de l'intention d'entrée doit être assuré en amont par le blueprint/pawn appelant.

**RPC.** `MulticastUpdateSirenState` (`UFUNCTION(NetMulticast, Reliable)`) est appelé à la fin de chaque `UpdateSirenState`. Son `_Implementation` ne fait du travail que **côté non-autorité** (`if (!HasAuthority())`) : il applique l'état reçu, arrête/joue l'audio correspondant, met à jour les feux et déclenche `OnSirenLightUpdate`. C'est ce qui rend la sirène audible et visible chez les autres joueurs. Le serveur, lui, a déjà tout exécuté dans `UpdateSirenState`.

**Responsabilités.** Serveur : résolution des gestes, machine à états, lecture audio locale, mise à jour des feux locale, diffusion multicast. Clients : application cosmétique (audio + feux) sur réception du multicast et/ou de la variable répliquée.

Note : le glitch audiovisuel et le clignotement détaillé reposent sur des timers/`TMap` **locaux** (non répliqués) — ils tournent indépendamment sur chaque instance qui les déclenche.

## 6. Points d'intégration

**Dépendances du plugin** (`.uplugin`) : `Qasset`, `EnhancedInput`, `Metasound`.

**Modules C++** (`QVehicles.Build.cs`) :
- Public : `Core`, `CoreUObject`, `Engine`, `EnhancedInput`, `NetCore`, `InputCore`, `MetasoundEngine`.
- Privé : `Slate`, `SlateCore`, `UMG`, `AudioPlatformConfiguration`, `AudioMixer`.

`MetasoundEngine` / `Metasound` sont la dépendance fonctionnelle clé : les sons de sirène sont des `UMetaSoundSource` joués via un `UAudioComponent`. `EnhancedInput`/`InputCore` reflètent l'origine « entrée honk », bien que le composant expose des `UFUNCTION` `BlueprintCallable` (`OnHonkInputPressed`/`OnHonkInputReleased`) plutôt que de lier directement des `UInputAction` — le mapping Enhanced Input est fait côté blueprint.

**Dépendants dans QANGA.** Aucun module C++ de `Plugins` ni de `Source` ne référence `UQVehicleSirenComponent`, `ESirenState` ni `FSirenMeshConfig` : l'intégration est **100 % Blueprint/contenu**. Les blueprints de véhicules du plugin (sous `Content/`, par ex. `Content/Vehicle/Blueprints/B_Navette_icli`) consomment le composant et la bibliothèque. Le plugin embarque tout le contenu véhicule de QANGA (camions, motos, pods, vélo, navettes, train/tram, sons) — d'où `CanContainContent: true` et un grand arbre `Content/`. À noter : `QVehicles` n'est pas listé dans la section `Plugins` de `QANGA.uproject` (les plugins voisins `ChaosVehiclesPlugin` et `FlyVehicleMovement` le sont) ; il est activé par sa simple présence dans `Plugins/`.

**Réglages / constantes** (pas de CVar, pas de console command). Le réglage se fait par les `UPROPERTY` `EditAnywhere`/`BlueprintReadWrite` du composant (délais, fenêtres de tap, taux de clignotement, matériaux, listes de noms) et par les constantes `static constexpr` privées de la bibliothèque qui gouvernent la descente audio du glitch : `SirenAudioPowerDownDuration` (5.0s), `SirenAudioPowerDownTickInterval` (0.1s), `SirenAudioPitchFloorMultiplier` (0.35), `SirenAudioLowPassStartHz` (2200) / `SirenAudioLowPassEndHz` (450), etc.

## 7. Gotchas, invariants et pièges

- **Concordance des `FName`.** Le composant ne tient aucune référence directe : il faut que le `UAudioComponent` se nomme exactement comme `AudioComponentName` (défaut `"SirenAudioComponent"`), et que chaque `FSirenMeshConfig::MeshName` / nom dans `LeftSpotLightNames`·`RightSpotLightNames` corresponde à un composant réel de l'acteur. Un nom erroné = sirène/feu silencieusement inactif (les fonctions échouent en `return` sans log).
- **Custom Primitive Data obligatoires.** Les gyrophares ne s'allument que si le matériau des meshes lit les indices déclarés (`RedIntensityDataIndex=0`, `RedToggleDataIndex=1`, `BlueIntensityDataIndex=2`, `BlueToggleDataIndex=3` par défaut). Sans ces indices côté matériau, `SetCustomPrimitiveDataFloat` n'a aucun effet visible. Un index négatif désactive le canal correspondant (`SetCustomPrimitiveDataSafe` exige `DataIndex >= 0`).
- **Registre statique global.** `SirenStates` est une `TMap` `static` au niveau fichier, partagée par tous les acteurs et toutes les instances (et persistante au sens du module, pas du monde). La purge dépend de `CleanupInvalidActors` (appelée par plusieurs entrées) via `TWeakObjectPtr`. Toujours appeler `DisableAllSirens`/`CleanupSirenSystem` à la destruction propre d'un véhicule pour libérer son entrée et ses timers ; sinon on s'appuie uniquement sur le nettoyage paresseux des poignées invalides.
- **Autorité requise pour les entrées.** `OnHonkInputPressed`/`OnHonkInputReleased` ne font rien sans autorité. Si on les appelle depuis un contexte client sans relais serveur, rien ne se passe. Aucun RPC serveur n'est fourni par le composant.
- **`Bwoop` réutilise `NormalSirenSound`.** `PlayAudioForState(ESirenState::Bwoop)` joue `NormalSirenSound`, pas un son dédié — il n'existe pas de `BwoopSound`. L'effet « bwoop » vient du double déclenchement temporisé (`BwoopBurstDuration`), pas d'un asset distinct.
- **`StopAudioForState` ignore l'état.** Elle arrête toujours le `UAudioComponent` par nom, quel que soit l'argument `State` — il n'y a qu'un seul canal audio de sirène par acteur ; on ne peut pas superposer deux sons de sirène.
- **Glitch vs clignotement normal.** `GlitchSiren` met `bNormalSirenOn = false` et réquisitionne le `LightBlinkHandle` partagé. Lancer un glitch écrase le clignotement normal en cours ; revenir à un état normal nécessite de repasser par `UpdateSirenState`/`StartSirenLights`. La descente audio du glitch prend un snapshot pitch/volume et le restaure via `RestoreSirenAudio` — si l'`AudioComponentName` ne correspond pas, le snapshot n'est jamais pris et l'audio n'est pas affecté.
- **Motifs `EGyroPattern` partiellement morts.** Seuls `LeftRight_Alternating` et `Rotating_Sequential` sont jamais sélectionnés ; ne pas supposer que `FrontBack_Alternating`/`Diagonal_Alternating` sont actifs.
- **Pas de tick — tout dépend du `FTimerManager`.** Si le monde est en pause ou si l'acteur change de monde, les `FTimerHandle` doivent être réarmés. Toute la temporisation (gestes, clignotement, glitch) cesse avec le timer manager.

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `G:/QANGA/Plugins/QVehicles/QVehicles.uplugin` | Manifeste : module Runtime unique `QVehicles`, `CanContainContent`, dépendances `Qasset`/`EnhancedInput`/`Metasound`. |
| `G:/QANGA/Plugins/QVehicles/Source/QVehicles/QVehicles.Build.cs` | Dépendances de module (MetasoundEngine, NetCore, AudioMixer, etc.). |
| `G:/QANGA/Plugins/QVehicles/Source/QVehicles/Public/QVehiclesModule.h` | Déclaration `FQVehiclesModule`. |
| `G:/QANGA/Plugins/QVehicles/Source/QVehicles/Private/QVehiclesModule.cpp` | `IMPLEMENT_MODULE` ; startup/shutdown vides. |
| `G:/QANGA/Plugins/QVehicles/Source/QVehicles/Public/QVehicleSirenComponent.h` | `UQVehicleSirenComponent`, `ESirenState`, délégué `FSirenStateChanged`, toutes les `UPROPERTY` de configuration et les `UFUNCTION` exposées. |
| `G:/QANGA/Plugins/QVehicles/Source/QVehicles/Private/QVehicleSirenComponent.cpp` | Machine à états : entrées honk, fenêtres tap, transitions, réplication (`DOREPLIFETIME`, `MulticastUpdateSirenState`). |
| `G:/QANGA/Plugins/QVehicles/Source/QVehicles/Public/QVehicleSirenBlueprintLibrary.h` | `UQVehicleSirenBlueprintLibrary`, `FSirenMeshConfig`. |
| `G:/QANGA/Plugins/QVehicles/Source/QVehicles/Private/QVehicleSirenBlueprintLibrary.cpp` | Exécution bas-niveau : audio MetaSound, Custom Primitive Data, spotlights, glitch audiovisuel, registre statique `SirenStates`, `EGyroPattern`, `FSirenState`. |
| `G:/QANGA/Plugins/QVehicles/Content/` | Contenu embarqué : `OvergroundVehicle/` (Camion, MotoSparta, Pod*, …), `Ships/`, `Train_Tram/`, `Vehicle/Blueprints/` (par ex. `B_Navette_icli`), sons, `sm_siren*`. Consomme le composant côté Blueprint. |
