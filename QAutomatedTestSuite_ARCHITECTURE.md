# QAutomatedTestSuite (QATS) - Architecture

> Plugin `DeveloperTool` de QANGA chargé en `PostDefault`. QATS réunit les contrats Unreal Automation, les vérifications runtime en conditions réelles et le driver end-to-end de quêtes. Il fonctionne en éditeur et en build packagé Development selon la surface utilisée, mais il est volontairement absent des builds Shipping.

---

## 1. But d'usage

QATS sert à détecter quatre familles de régressions qui ne se prouvent pas avec un simple build C++ :

- **Contrats déterministes** : math de navigation, agrégation et persistance QModule, cohérence des assets audio, invariants du contrôleur neural.
- **Intégration runtime réelle** : registre QModule, notifications, ducking audio, overrides d'ambiance, radio, musique et niveau de recherche dans un vrai monde de jeu.
- **Quêtes end-to-end** : accepter les vraies quêtes, attendre leur streaming, résoudre leurs objectifs, envoyer les vrais inputs joueur, valider les trackers/UI et observer la progression autoritaire.
- **Véhicule et entraînement neural** : conduire les objectifs longue distance avec QAI, ou brancher une session DreamerV3 sur le hovercraft de test sans créer une seconde simulation de gameplay.

Les objectifs architecturaux sont :

- protéger le profil joueur par isolation explicite ;
- exécuter les systèmes de production au lieu de simuler leurs résultats ;
- produire des artefacts JSON exploitables même après un crash, un timeout ou un enfant qui quitte ;
- échouer ou marquer `unsupported` quand aucun exécuteur réel n'existe, sans valider artificiellement l'objectif ;
- restaurer toutes les mutations temporaires avant de terminer.

---

## 2. Vue d'ensemble / décisions d'architecture

QATS expose quatre surfaces complémentaires :

| Surface | Entrée | Contexte | Rôle |
|---|---|---|---|
| Contrats Unreal Automation | Tests `QATS.*` | Éditeur ou Development compatible avec les flags du test | Vérifications rapides, sans campagne de jeu complète |
| Campagne maître | `qats.runall` | Build packagé Development, `L_Persistent_Universe` | Lance trois processus isolés : `Automation`, `Runtime`, puis `Quest` |
| Driver de quêtes | `quest.test.start` | PIE en place ou processus Development isolé | Joue une sélection de quêtes avec les systèmes et inputs réels |
| Runner QModule | `qats.qmodule.start` | Contract en PIE/jeu ; Interactive uniquement en Development isolé | Vérifie toutes les définitions puis, en mode interactif, leurs effets sur des cibles réelles |

Décisions structurantes :

- **Un seul module, plusieurs runners privés.** L'unique module `QAutomatedTestSuite` enregistre les commandes et instancie les services `QATSRunAll` et `QATSQModuleTests`. Le seul type public substantiel est `UQuestTestSubsystem`.
- **Deux gardes de compilation.** Les tests Automation sont sous `WITH_DEV_AUTOMATION_TESTS`. Les runners console et runtime sont sous `!UE_BUILD_SHIPPING`. Une commande absente en Shipping est donc un contrat, pas un défaut de packaging.
- **Isolation par processus pour la campagne complète.** Le parent copie le `Saved/SaveGames` de référence dans un `-UserDir` distinct pour chaque enfant. Les résultats d'une suite ne polluent pas la suivante.
- **Isolation ciblée en PIE.** Le driver de quêtes supprime la sauvegarde de progression éditeur pendant la passe et rend temporairement non persistante la transform du joueur avant tout déplacement. La transform et son état de persistance sont restaurés à la fin ; un échec de restauration fait échouer la passe.
- **Machines à états pollées.** `FTSTicker` pilote le master et QModule ; `UQuestTestSubsystem::Tick` pilote les quêtes. Les deadlines sont des timestamps, les actions répétées sont bornées, et les diagnostics fréquents sont throttlés.
- **Les systèmes de production restent propriétaires.** QATS appelle `AcceptQuest`, les composants d'input, QLevel, les flow fields QAI, les composants véhicule, les managers audio et les APIs autoritaires QModule. Une classe d'objectif inconnue devient `unsupported`.
- **Les artefacts sont la source de vérité.** Les messages écran et `LogQAutomatedTestSuite` donnent l'état humain ; les fichiers JSON/JSONL déterminent le résultat machine.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `FQAutomatedTestSuiteModule` | `IModuleInterface` | Enregistre `quest.test.start`, démarre/arrête les runners QModule et RunAll, désactive le throttling de fond pendant un quest test |
| `FQATSRunAllService` | classe C++ privée + `FTSTicker` | Orchestrateur parent de la campagne et runner du processus enfant `Runtime` |
| `UQuestTestSubsystem` | `UGameInstanceSubsystem` + `FTickableGameObject` | Driver de quêtes, navigation, interaction, combat, véhicule, entraînement neural, reporting et nettoyage |
| `FQModuleTestService` | classe C++ privée + `FTSTicker` | Runner Contract/Interactive de toutes les définitions QModule |
| `QAudioAutomationTests.cpp` | tests Unreal Automation | Contrats d'assets et de réglages QRadio/QMusicDirector/QAmbientAudio |
| `QModuleAutomationTests.cpp` | tests Unreal Automation | Agrégation, voisins hexagonaux, codec de sockets et validation des définitions |
| Contrats dans `QuestTestSubsystem.cpp` | tests Unreal Automation | Décisions pures QAI/navigation/véhicule et contrats du pipeline neural |
| `LogQAutomatedTestSuite` | catégorie de log | Sortie commune de toutes les surfaces QATS |

### 3.1 Contrats Automation enregistrés

Les tests `ProductFilter` utilisent `EAutomationTestFlags_ApplicationContextMask` et peuvent être retenus par le processus Automation packagé :

- `QATS.QRadio.Catalog.Contract`
- `QATS.QMusicDirector.Profile.Contract`
- `QATS.QModule.Aggregation.OperationOrder`
- `QATS.QModule.Aggregation.HexNeighbors`
- `QATS.QModule.Persistence.SocketCodec`
- `QATS.QModule.Definition.Validation`

Les contrats de décision dans `QuestTestSubsystem.cpp` sont `EditorContext | EngineFilter` :

- QAI et quêtes : `QATS.QAI.AmbientSpawnSuppressionScope`, `QATS.Quest.PersistentUniverseCombatLoadoutScope`, `QATS.Quest.ElevatorRouteOwnership`.
- Navigation : `QATS.Navigation.MidpointCoverage`, `QATS.Navigation.ResolvedEndpointOwnership`, `QATS.Navigation.StepTraversal`.
- Véhicule : `QATS.Vehicle.PredictiveFieldCoverage`, `QATS.Vehicle.GroundedRequester`, `QATS.Vehicle.QaiReverseCorridor`, `QATS.Vehicle.MacroscopicFlowPath`.
- Neural, sous `QATS.Vehicle.*` : `NeuralAuthoritativeRoutePolyline`, `NeuralEpisodeTimeoutArming`, `NeuralTrainingHealthContract`, `NeuralCurriculumRouteStart`, `NeuralDetailedQaiCacheContract`, `NeuralFinalApproachFieldContract`, `NeuralObjectiveSurfaceGoal`, `NeuralPerAgentReadinessPartition`, `NeuralObstacleClosingSpeed`, `NeuralResetObservation`, `NeuralEpisodeResumeContract`, `NeuralPhysicsFrameValidity` et `NeuralCurriculumBarrier`.

Le préfixe `QATS.` est obligatoire pour qu'un nouveau contrat soit découvert par `qats.runall` via `Automation RunTests StartsWith:QATS.`. Unreal ne lance que les tests compatibles avec le contexte courant ; la campagne exige au moins un test exécuté et refuse tout `failed`, `notRun` ou `inProcess` résiduel dans le rapport.

---

## 4. Flux de données et cycle de vie

### 4.1 Initialisation du module

1. Le module `DeveloperTool` est chargé en `PostDefault`.
2. `FQAutomatedTestSuiteModule::StartupModule` démarre `FQModuleTestService` et `FQATSRunAllService`, puis enregistre `quest.test.start`.
3. Chaque service ajoute un ticker et ses commandes console.
4. `UQuestTestSubsystem` existe par `UGameInstance`. Sans `-QuestTest` ou appel PIE explicite, il conserve ses hooks de monde nécessaires au neural mais son driver ne ticke pas.
5. Au shutdown, les commandes et tickers sont supprimés, le processus enfant actif est terminé si nécessaire et les mutations runtime sont nettoyées.

### 4.2 Campagne `qats.runall`

Préconditions vérifiées avant toute mutation :

- build **packagé Development** (`RequiresCookedData() == true`) ;
- monde courant `L_Persistent_Universe` ;
- processus standalone ou listen-server, jamais client pur ni serveur dédié ;
- lancement depuis le profil normal, sans `-UserDir` déjà présent ;
- dossier baseline `Saved/SaveGames` existant.

Le parent crée `Saved/QAutomatedTestSuite/Runs/<run-id>/`, puis exécute séquentiellement :

1. **Automation** : enfant `-unattended -nullrhi -nosound`, profil `Profiles/Automation`, commande `Automation RunTests StartsWith:QATS.;SoftQuit`, timeout 10 minutes. Le parent parse `Automation/index.json` et ses compteurs.
2. **Runtime** : enfant visible en `L_Persistent_Universe`, profil `Profiles/Core`, timeout 15 minutes. Il attend le monde, le joueur et les subsystems, lance le contrat QModule complet, puis exerce QNotification, le ducking de communication, QAmbientAudio, QRadio, QMusicDirector et QPolice avec leurs vraies APIs et leurs vrais inputs.
3. **Quest** : enfant avec profil `Profiles/Quest`, timeout 6 heures. Il lance le driver sur le filtre demandé en mode hovercraft `Heuristic`, écrit son JSON/JSONL, puis quitte grâce à `-QuestTestExitWhenDone`.

Le rapport maître passe uniquement si les trois enfants ont un exit code `0`, un artefact parseable et leur statut attendu (`passed`, `passed`, `complete`). Un launch raté, un timeout, un artefact absent ou une suite incomplète fait échouer la campagne. `qats.runall.stop` termine seulement l'enfant isolé courant et écrit un master `aborted`.

### 4.3 Driver `quest.test.start`

Dans PIE, la commande arme le `UQuestTestSubsystem` du `GameInstance` courant. Dans un build packagé, elle :

1. clone le profil normal dans `Saved/QuestTests/Runs/<run-id>/UserDir/Saved/SaveGames` ;
2. retire `Tutorial.sav` et `BACKUP_Tutorial.sav` de la copie pour que l'état runtime du tutorial soit propre ;
3. relance le même exécutable avec `-QuestTest`, `-QuestTestAutoStart=Standalone` et le `-UserDir` isolé ;
4. ferme le processus d'origine après un lancement réussi.

L'enfant refuse de démarrer si `ProjectSavedDir` n'est pas réellement sous le `-UserDir` annoncé. Si le profil cloné déclenche le first-run setup, le driver confirme la langue anglaise, choisit AZERTY, saute la compilation optionnelle des shaders et confirme le personnage par défaut avant de rejoindre le monde requis.

La machine à états principale est :

`WaitForPlayer -> ClearProgress -> AcceptQuest -> RunQuest -> CleanupQuest -> AcceptQuest ... -> Complete`

- **WaitForPlayer** attend un controller local, un pawn et un `UPlayerQuestComponent` valide.
- **ClearProgress** vide la progression dans le profil isolé, prépare le niveau joueur requis et attend l'acquittement autoritaire des prérequis de fixture.
- **AcceptQuest** trie d'abord les quêtes et leurs prérequis topologiquement, voyage vers le monde requis si nécessaire, puis maintient des leases QLevel sur les destinations initiales jusqu'à leur visibilité avant d'appeler `AcceptQuest`.
- **RunQuest** vérifie l'UI et les trackers, choisit l'objectif actif non optionnel le plus prioritaire, puis délègue à un exécuteur réel.
- **CleanupQuest** neutralise les inputs, libère les flow fields, nettoie le véhicule, abandonne seulement une quête encore active et attend que l'état autoritaire se stabilise avant la suivante.
- **Complete** restaure les états temporaires, écrit l'artefact final et quitte si le run a été lancé avec `-QuestTestExitWhenDone`.

Exécuteurs disponibles :

| Type | Exécution réelle |
|---|---|
| `Location` | Déplacement par flow field QAI, gestion des pas, obstacles, portes, ascenseurs, jetpack et handoff véhicule selon l'objectif |
| `Interaction` | Résolution de la cible vivante, approche atteignable, focus par la caméra et input d'interaction |
| `Dialogue` | Observation du popup, sélection via l'UI et fermeture par input |
| `Input` | Résolution du binding actif dans `InputSystem`, press/release réels ; inclut scanner, réparation et certains workflows UI |
| `Kill` | Acquisition d'une cible hostile, équipement du loadout de test, LOS et attaque par inputs de production |
| `Connector` | Prise du câble, approche de la socket et confirmation de connexion |
| `Passive` | Attend la progression produite par le système propriétaire avec timeout |
| `Missing` | Termine la quête avec `unsupported`, jamais avec un faux succès |

Le driver met en pause ses deadlines pendant le verrou global de readiness gameplay. À l'entrée du verrou, il neutralise les outputs et invalide les routes calculées avant le streaming ; à la reprise, il reconstruit depuis la position autoritaire courante.

### 4.4 Runner QModule

Le runner charge les définitions depuis `UQModule_Registry_GI_SubSystem`, les trie par domaine puis tag, et exécute un préflight suivi d'un contrat par définition.

| Mode | Cible | Profil | Validation |
|---|---|---|---|
| `Contract` | acteur caché et `UQModule_RackComponent` créés par le test | Aucun état joueur touché | Validation de définition, insertion/retrait, phases, stats, codec et nettoyage ; `real_status=NotRun` |
| `Interactive` | mur du joueur, item d'arme ou rack véhicule réel | Processus Development obligatoirement isolé | Prépare la cible, attend l'observation humaine, puis reçoit `pass`, `fail` ou `skip` |

Le mode Interactive capture un snapshot du mur/rack avant mutation. Les items temporaires sont consommés après remboursement, les sockets et limites sont restaurées, puis l'artefact est écrit. Un `skip`, un échec de contrat, un échec réel ou un cleanup incomplet empêche le run d'être considéré comme passé.

### 4.5 Nettoyage commun

Les sorties normales, les arrêts manuels et `Deinitialize` libèrent notamment :

- inputs maintenus et bindings debug temporairement supprimés ;
- flow fields QAI, leases de streaming et suppression de spawn ambiant ;
- véhicules autoritaires, sessions neurales, overrides de fuel/vie et callbacks monde ;
- loadout, god mode de setup, fixtures de prérequis et verrou de readiness ;
- notifications, ducking, override d'ambiance, radio temporaire, wanted level et delegates du child Runtime ;
- snapshots QModule et acteur/rack scratch.

Un échec de cleanup est enregistré comme un vrai échec de suite.

---

## 5. Réplication / réseau

QATS ne définit aucun état répliqué, aucune RPC et aucun `GetLifetimeReplicatedProps`. Il consomme les contrats réseau des plugins testés.

- **RunAll** accepte uniquement standalone ou listen-server dans le processus parent packagé.
- **QModule** refuse `NM_Client` et exige l'autorité sur l'acteur scratch et les racks réels.
- **Processus avec joueur local** (standalone, listen-server ou client) : le driver pilote le controller local, puis attend que l'acceptation, la progression, le niveau joueur et les fixtures de prérequis deviennent autoritaires et répliqués.
- **Serveur dédié** : le même `UQuestTestSubsystem` bascule vers `TickServerObserver`. Il prépare le loadout et le véhicule côté autorité, observe les quêtes actives et écrit des événements `scope=server_authority`. Il ne simule pas l'input d'un joueur local.
- **Artefacts** : chaque événement porte `role=standalone|listen_server|server|client`. Si un serveur lancé avec `-QuestTestExitWhenDone` a vu son joueur puis le perd pendant plus de 10 secondes, il termine avec `client_disconnected`.

Les mutations multijoueur doivent donc continuer à passer par les composants de production. Ajouter une RPC privée à QATS pour contourner une autorité manquante casserait le but du harness.

---

## 6. Points d'intégration

### 6.1 Dépendances

Le descripteur active explicitement `DynamicQuestSystem`, `QAI`, `QDynamicsCollision`, `QAmbientAudio`, `QElevator`, `QNotification`, `QModule`, `QMusicDirector`, `QPolice`, `QRadio`, `InputSystem`, `EnhancedInput`, `RzNeuralNetwork` et `WorldScape`.

Le module C++ consomme aussi `DynamicQuestSystemObjectives`, `FlyVehicleMovement`, `QLevel`, `GameplayTags`, `MetasoundEngine`, `PhysicsCore`, `Slate`, `UMG`, `Json`, `AssetRegistry` et `ApplicationCore`. Cette dépendance large est volontaire : QATS est le point d'intégration, pas une bibliothèque gameplay réutilisable.

### 6.2 Commandes publiques

| Commande | Usage |
|---|---|
| `qats.runall [QuestRepetitions=1] [Seed=1337] [QuestFilter=All]` | Campagne packagée complète depuis `L_Persistent_Universe` |
| `qats.runall.status` | Stage parent, nombre de suites, PID enfant et chemin du rapport |
| `qats.runall.stop` | Termine l'enfant courant et clôt la campagne en `aborted` |
| `quest.test.start [QuestID\|All] [Repetitions] [Seed] [Heuristic\|Neural\|Train\|TrainResume]` | Driver ciblé ; défauts directs : `All`, `1000`, `1337`, `Neural` |
| `qats.qmodule.start [All\|Cyborg\|Weapon\|Vehicle] [Interactive\|Contract] [WeaponSlotFilter] [VehicleIndex]` | Runner QModule ; le mode omis vaut `Interactive` |
| `qats.qmodule.pass [note]` | Valide l'observation réelle courante |
| `qats.qmodule.fail [raison]` | Échoue l'observation réelle courante |
| `qats.qmodule.skip [raison]` | Enregistre explicitement un prérequis/contrôle réel non exécuté |
| `qats.qmodule.retry` | Reprend la préparation après satisfaction du prérequis |
| `qats.qmodule.status` | État, définition courante et artefact |
| `qats.qmodule.stop` | Nettoie les cibles et écrit un artefact partiel |
| `qats.NeuralHovercraft.DrawMainFlowField <0\|1>` | Dessin debug du champ QAI, de la perception 360° et de la route autoritaire du hovercraft |

Les arguments sont positionnels. Le filtre de quêtes accepte `All`, un ID, ou plusieurs IDs séparés par des virgules :

```text
qats.runall 1 1337 All
quest.test.start Q002,Q007 1 1337 Heuristic
qats.qmodule.start All Contract
```

Pour une passe Automation ciblée dans l'éditeur :

```text
Automation RunTests StartsWith:QATS.
```

### 6.3 Arguments internes du driver de quêtes

Les commandes publiques construisent normalement ces arguments. Ils restent utiles pour comprendre un lancement ou reproduire un enfant :

| Argument | Effet |
|---|---|
| `-QuestTest` | Active le subsystem au démarrage |
| `-UserDir=<dir>` | Isolation obligatoire hors PIE ; `ProjectSavedDir` doit être sous ce dossier |
| `-QuestTestAutoStart=Standalone` | Traverse le lobby/first-run setup puis charge le monde requis |
| `-QuestTestExitWhenDone` | Quitte après l'artefact final |
| `-QuestTestQuest=`, `-QuestTestRepetitions=`, `-QuestTestSeed=` | Sélection et déterminisme |
| `-QuestTestStageTimeout=`, `-QuestTestPlayerTimeout=`, `-QuestTestObjectiveTimeout=`, `-QuestTestTrackerGrace=`, `-QuestTestFlowTimeout=` | Deadlines bornées |
| `-RzHoverDrive=Heuristic\|Neural\|Train\|TrainResume` | Politique hovercraft |
| `-RzHoverPolicySnapshots=<dir>` | Répertoire de checkpoints DreamerV3 |
| `-RzHoverResumeTraining`, `-RzHoverOptimizeTime` | Reprise et objectif d'optimisation |

Les arguments `-QATSRunAll*` et `-QATSQModule*` identifient des processus enfants privés. Ils ne constituent pas une API utilisateur stable.

### 6.4 Artefacts

Tous les chemins sont relatifs au `ProjectSavedDir` effectif. Avec `-UserDir`, ils restent donc dans le sandbox.

| Surface | Chemin principal | Contenu |
|---|---|---|
| RunAll | `Saved/QAutomatedTestSuite/Runs/<run-id>/summary.json` | Statut maître, paramètres, trois suites, détails, artefacts et exit codes |
| Automation child | `<run>/Automation/index.json` | Rapport Unreal Automation et compteurs de tests |
| Runtime child | `<run>/Core/summary.json` | Stage et résultat de chaque contrôle QModule/audio/UI/police |
| Quest child RunAll | `<run>/Profiles/Quest/Saved/QuestTests/quest_test_<pid>.json` + `.jsonl` | Résumé et flux complet des événements de quête |
| Quest PIE | `Saved/QuestTests/PIE/<run-id>/quest_test_pie.json` + `.jsonl` | Même schéma, sans profil OS séparé |
| Quest direct packagé | `Saved/QuestTests/Runs/<run-id>/UserDir/Saved/QuestTests/quest_test_<pid>.json` + `.jsonl` | Résultat dans le profil relancé |
| QModule | `Saved/QModuleTests/Runs/<run-id>/Artifacts/summary.json` | Résultat par définition, statuts contract/réel et détail |

Le résumé Quest contient les compteurs globaux mais seulement les **8 événements récents**. Le `.jsonl` adjacent est l'historique complet ; chaque ligne contient scope, outcome, raison, rôle réseau, répétition, seed, quête, objectif, classe, stage, temps et position du pawn quand elle existe.

Statuts à ne pas confondre :

- master : `running`, `passed`, `failed`, `aborted` ;
- quest : `starting`, `running`, `complete`, `complete_with_failures`, `failed`, `client_disconnected` ;
- résultat d'objectif : `passed`, `failed`, `skipped`, `unsupported`, `observed` selon le scope ;
- QModule : `Passed|Failed` pour le contrat et `Passed|Failed|Skipped|NotRun` pour le réel.

---

## 7. Gotchas, invariants et pièges

- **`BUILD SUCCESSFUL` ne prouve aucune passe QATS.** Une campagne est réussie uniquement si son `summary.json` maître vaut `passed` et référence trois artefacts enfants valides.
- **`qats.runall` n'est pas une commande PIE.** Il exige un Development packagé déjà entré dans `L_Persistent_Universe` avec le profil normal.
- **Les valeurs par défaut diffèrent fortement.** `qats.runall` lance une répétition de quête en `Heuristic`. `quest.test.start` lancé sans arguments demande 1000 répétitions en `Neural`.
- **Le mode QModule par défaut est `Interactive`.** En PIE normal il est refusé pour protéger le profil. Utiliser explicitement `Contract` en PIE.
- **Un `-UserDir` existant bloque le relaunch public.** Il faut repartir du processus Development normal ; imbriquer des sandboxes rendrait la provenance du profil ambiguë.
- **Le baseline `Saved/SaveGames` est obligatoire.** QATS refuse de fabriquer un faux profil quand la source manque.
- **Les mondes editor-only ne sont pas cuisinés.** Une quête qui les cible peut être `skipped` pendant une campagne packagée ; si aucune quête sélectionnée n'a de monde packagé supporté, l'auto-start échoue explicitement.
- **`recent_results` n'est pas le rapport complet.** Lire le `.jsonl` pour diagnostiquer une longue campagne.
- **Les contrats `EditorContext` ne deviennent pas magiquement Product tests.** Le child Automation packagé exécute seulement les tests compatibles avec son contexte.
- **Les inputs viennent du mapping actif.** L'absence de binding clavier valide est une erreur de configuration réelle, pas une raison de substituer une touche hardcodée.
- **Le streaming fait partie du test.** Une destination doit être visible via QLevel et sa cible vivante avant action. Allonger arbitrairement le timeout ne remplace pas cette preuve.
- **Les flow fields appartiennent à QAI.** La navigation et le véhicule ne doivent pas recevoir un line trace ou un calcul direct de secours si QAI échoue ; le runner doit rendre un diagnostic causal.
- **Le cleanup est transactionnel.** Toute nouvelle mutation doit avoir son snapshot/rollback et participer aux sorties normale, stop, timeout et deinitialize.
- **Le master sérialise volontairement ses enfants.** Les profils sont distincts, mais l'ordre Automation -> Runtime -> Quest garantit un rapport lisible et évite la concurrence sur les ressources de jeu.
- **Les logs fréquents restent bornés.** RunAll et QModule drainent au plus un message de leur queue par seconde ; les heartbeats et diagnostics du driver possèdent leurs propres intervalles.

### Ajouter une couverture

- Pour une décision pure ou un contrat d'asset, ajouter un test `WITH_DEV_AUTOMATION_TESTS` nommé `QATS.<Système>.<Contrat>` avec les flags correspondant réellement au contexte nécessaire.
- Pour une interaction de production packagée, étendre la machine à états `Runtime`, enregistrer chaque observation avec `RecordCore`, puis restaurer la mutation dans `CleanupCoreState`.
- Pour une nouvelle classe d'objectif, ajouter un exécuteur explicite dans `ResolveExecutor` et son chemin de cleanup. Ne jamais la classer `Passive` uniquement pour éviter `unsupported`.
- Pour le véhicule/neural, extraire d'abord la décision mathématique pure et la couvrir par Automation ; garder dans le tick uniquement l'observation et l'application aux propriétaires runtime.
- Toute nouvelle suite enfant doit avoir un artefact parseable, un timeout, un exit contract et une entrée dans le statut maître avant d'être considérée comme intégrée.

---

## 8. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Plugins/QAutomatedTestSuite/QAutomatedTestSuite.uplugin` | Descripteur `DeveloperTool`, phase de chargement et plugins requis |
| `Source/QAutomatedTestSuite/QAutomatedTestSuite.Build.cs` | Dépendances C++ du harness |
| `Public/QAutomatedTestSuite.h` | Catégorie et macros de log QATS |
| `Private/QAutomatedTestSuiteModule.cpp` | Cycle de vie du module, `quest.test.start`, relaunch et isolation du profil |
| `Private/QATSRunAll.cpp` | Parent RunAll, enfants Automation/Runtime/Quest, runtime core et rapports consolidés |
| `Private/QModuleTestRunner.cpp` | Runner QModule Contract/Interactive, snapshots, commandes manuelles et artefact |
| `Private/QAudioAutomationTests.cpp` | Contrats QRadio et QMusicDirector/QAmbientAudio |
| `Private/QModuleAutomationTests.cpp` | Contrats purs QModule |
| `Public/QuestTestSubsystem.h` | États et données persistantes du driver |
| `Private/QuestTestSubsystem.cpp` | Driver end-to-end, contrats de décision, navigation, UI, combat, véhicule, neural et JSONL |
