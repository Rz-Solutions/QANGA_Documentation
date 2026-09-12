# Modules v2 : verification finale avant passage pre-live vers live (2026-09-12)

Demande de Benja : "derniere verification avant de switcher la branche pre-live sur la branche live
de Steam ; tout fonctionne sauf le rappel de flotte qui rappelle des vehicules terrestres".
Session Claude du 2026-09-12 (soir), editeur ferme, aucun fichier du projet modifie hors ce document
et le dossier local `Saved/FleetRecall_Patch_20260912/` (non synchronise).

## 0. Verdict

**La build pre-live peut passer en live telle quelle** : elle est coherente (107 definitions de
modules, 51 items, tous les assets references par le code presents dans le manifeste Steam du
12/09 15:46, correctif du menu Modules vide au join present dans l exe, 8/8 tests QATS du plugin
verts). Rien de "test", "debug" ou "triche" ne part en Shipping.

**Deux points a trancher AVANT, parce qu ils touchent tous les joueurs live existants :**

1. **Le solde de phase d avant les modules n est pas repris.** Le live actuel (v0.0.22a du 10/07)
   tourne avec l ancien systeme de phases. Le passage en 0.0.23 remplace l ecran par le mur v2 dont
   le portefeuille part de zero : tous les joueurs qui ont progresse avant la bascule voient leurs
   points de phase "disparaitre" (le retour joueur Terpen du 25/08 sur le pre-live, section 2.1).
   Aucune conversion n existe dans le code. Decision produit : convertir, compenser, ou assumer et
   communiquer.
2. **Le rappel de flotte accepte les vehicules terrestres.** Diagnostic fait, correctif C++ pret et
   relu, NON applique dans l arbre (il est synchronise en direct avec la machine de RzZz) et NON
   compile. Section 3.

Le reste est de la dette non bloquante ou des regles de prod a respecter (sections 4 a 6).

## 1. Ce qui a ete mesure, et comment

| Verification | Methode | Resultat |
|---|---|---|
| Suite de tests du projet | QATS headless, 162 tests, 63 s (`Saved/QATSAutomation/index.json`) | 131 OK, 29 OK avec warnings, 2 echecs (section 1.1) |
| Plugin C++ QModule (112 fichiers, 44 618 lignes) | audit statique par agent : Shipping, serveur dedie, reseau, lifetime, chemins d assets, persistance, reglages, TODO | section 4 et 5 |
| Cook des assets nommes par le code | 115 chemins C++ croises avec `Manifest_UFSFiles_Win64.txt` de la build Steam installee (12/09 15:46) | 115/115 presents |
| Build pre-live installee | lecture de `Qanga-Win64-Shipping.exe` (201 Mo) | contient la livraison v2 du rappel de flotte et le correctif "Wall bound on late retry" du join multi |
| Donnees QMD, items, DA_AllRef, loot, distribution, localisation | agent, lecture binaire des .uasset | section 7 |
| Couplage C++ vers BP par reflexion, referents BP du plugin, docs et chantiers ouverts | agent | section 8 |
| Rappel de flotte | lecture du code + des BP du garage (`PlayerVehiclesComponent`, `SpawnPlayerOwnedVehicle`, `DA_AllRef`) | section 3 |

### 1.1 Les deux echecs QATS, expliques

- `QATS.Stats.KillTags.CoveredByAnActor` : **artefact de l arbre de travail, pas de la build.** Le
  run headless a charge `VehicleBase`, qui est desormais reparente sur une classe C++ `AFlyVehiclePawn`
  (plugin FlyVehicleMovement) arrivee par Syncthing aujourd hui (`FlyVehiclePawn.h/.cpp` 18:22,
  `FlyVehicleMovementComponent.h/.cpp` 18:52, `VehicleBase.uasset` 18:28). La DLL compilee sur cette
  machine datait d hier 16:19 et ne contient pas cette classe, donc tous les BP vehicule ont echoue a
  charger dans le run. Benja a lance la compilation de `QangaEditor` a 21:06 : a re-lancer apres.
- `QATS.Items.VossCaptainHat.Authoring` : le test (fichier de RzZz, `QVossHatAuthoringTests.cpp`)
  exige "le loot Voss lache TOUJOURS le chapeau du capitaine". La refonte du loot du 05/09 (validee
  par Benja) l a passe a 20 % de chance. **Les deux ne peuvent pas etre vrais** : soit le test est
  perime, soit la donnee a ete changee sans que RzZz le sache. A aligner entre vous.

Les 8 tests du plugin sont verts : agregation (voisins hex, ordre des operations), validation des
definitions, cles d items inscrites dans `DA_AllRef`, scripts d items valides, codec de persistance,
zones sures.

## 2. Les deux decisions

### 2.1 Solde de phase legacy (tous les joueurs live existants)

Etat du code, verifie ligne par ligne (`Plugins/QModule/Source/QModule/Private/QModule_RackComponent.cpp`) :

- la seule conversion existante est `Authority_ConvertLegacyPhaseItems` (ligne 736) : elle transforme
  des ITEMS de phase presents dans le sac en points de portefeuille. Elle ne lit ni le solde
  `PhasePoints` de `SS_Phase`, ni la cle de sauvegarde `PhaseData` (les niveaux deja achetes) ;
- la seule source de points v2 est le level-up (`HandleLegacyLevelUp`, ligne 870) : 1 point T1 par
  niveau gagne APRES la bascule ;
- l ancien stock continue de monter en silence a chaque niveau (`QangaPlayerState.LevelUp_Event`
  credite toujours `SS_Phase`) et n est plus depensable, l ecran qui le depensait est replie par
  `UQModule_LegacyPhaseSwap`.

Consequence : un joueur niveau 14 arrive avec un mur vide (hors 5 modules de base) et 0 point. Ses
donnees ne sont pas detruites, elles ne sont plus lues.

Options, par cout croissant :

1. **Assumer et communiquer** dans le patch note ("la progression de phase repart de zero avec le
   nouveau systeme"). Cout nul, risque d avis Steam negatifs (Terpen l a ecrit spontanement).
2. **Compensation manuelle** des joueurs actifs (items `IDA_QModulePhase_T1` via l admin, la
   conversion item vers point existe deja). Cout humain, pas de code.
3. **Passerelle one-shot au premier login v2** : a la creation de l entree `QMODWall` (aucune
   sauvegarde v2 existante), lire `PhasePoints` + la somme des niveaux de `PhaseData` sur le
   PlayerState et crediter autant de points T1 ; marquer la conversion faite (une ligne versionnee
   dans la sauvegarde, comme `QMODWALLET;v1`). Environ une journee avec test PIE multi. C est ce que
   le plan d architecture prevoyait (grandfathering).

### 2.2 Rappel de flotte : section 3.

## 3. Rappel de flotte : diagnostic et correctif pret

**Cause.** `SV_TriggerFleetRecall` (`QModule_RackComponent.cpp:2646`) prend `CurrentOwnedVehicle`
sur le `PlayerVehiclesComponent` du PlayerState, sinon `OwnedVehicles[0]`, et l envoie tel quel au
pipeline de garage `SpawnPlayerOwnedVehicle_C`. Aucun test de type nulle part dans le plugin : la
livraison volee (`AQModule_VehicleDeliveryActor`) fait donc voler ce qu on lui donne, hover car
comprise. Le meme choix est duplique dans `Authority_ResolveFleetIds` (commande `qmodule.Test.Recall`).

**Ou est l information de type.** Dans les donnees du jeu, pas dans le plugin : `DA_AllRef` (classe BP
`DA_References_C`) porte la map `Id:VehicleClass` (FName id vers soft class), celle que le garage lit
deja. Et toute la famille volante derive de `SpaceshipBase_C` (`Spacecraft`), les autres de
`HovercraftBase_C`, `WatercraftBase_C` (etiquete `Hovercraft`, dette connue) et `BikeBase_C`.

**Correctif** (8 fichiers du plugin, 184 lignes de diff, relu, PAS compile) :

- nouveau choix du vehicule : `AQModule_VehicleDeliveryActor::Authority_PickRecallVehicle` : le
  vehicule COURANT s il descend de `SpaceshipBase_C`, sinon le premier vaisseau de `OwnedVehicles`
  (ordre du garage), sinon refus. La resolution id vers classe passe par `Id:VehicleClass` de
  `DA_AllRef` (soft ref dans les settings, `FleetRecallVehicleRegistry`), la classe est chargee (le
  garage la charge de toute facon juste apres) et la parente est parcourue PAR NOM : le plugin ne lie
  toujours aucun contenu du jeu ;
- un seul point de decision, partage par le RPC et par la commande de test ;
- nouveau refus lisible `FleetRecallNoFlyingVehicle` (valeur ajoutee EN FIN d enum, contrat fil et
  BP intact), texte anglais "The transponder only reaches spacecraft: none in your fleet.", branche
  sur le reticule comme les trois autres refus ;
- reglage `FleetRecallVehicleBaseClassName` (defaut `SpaceshipBase_C`, `None` = ancien comportement)
  : kill switch sans rebuild dans Project Settings ;
- les deux helpers devenus morts sont retires (pas de warning "fonction statique non referencee").

**Livraison** : `Saved/FleetRecall_Patch_20260912/` (dossier local, Syncthing ignore `Saved`) :
`fleetrecall.diff`, les 8 fichiers `orig/` et `new/`, `APPLY.ps1` (verifie par md5 que les fichiers
n ont pas bouge, sauvegarde, copie) et `REVERT.ps1`. Pourquoi pas applique directement : l arbre
`Plugins/*/Source` est synchronise en direct avec la machine de RzZz (`.stfolder`, `.stignore`),
qui travaille en ce moment sur FlyVehicleMovement ; un C++ non compile y arriverait pendant son
build. A appliquer sur ton feu vert, puis compiler `QangaEditor`.

**Reste a faire apres compilation** : `qmodule.Test.Recall` avec une flotte mixte (hover car
courante + vaisseau possede : le vaisseau doit venir ; hover car seule : refus a l ecran) ; puis
traduction fr/es de la nouvelle phrase (voir la note de localisation en section 7).

## 4. Ce qui est propre (aucune action)

- **Shipping** : les 62 `qmodule.Test.*`, 11 `qmoduleloot.*`, 5 `qmodulebounty.*` et le fichier de
  debug sont entierement sous `#if !UE_BUILD_SHIPPING` ; les 6 tests d automatisation sous
  `WITH_DEV_AUTOMATION_TESTS` (a 0 en Shipping, verifie dans UBT) ; zero `DrawDebug`. Les logs
  verbeux `QMOD_VLOG` sont compiles mais eteints (`bVerboseLogging=false`, non surcharge dans l ini).
- **Serveur dedie** : les 5 World subsystems sont serveur-safe (filtres de creation, autorite
  verifiee) ; toute la chaine audio et les 5 multicasts sont gardes ; le HUD gadget n est jamais cree
  sur un serveur.
- **Reseau** : 27 RPC, tous dans 3 classes, Reliable pour l etat et Unreliable pour le cosmetique ;
  un client ne peut pas appeler un `SV_*` sur le PlayerState d un autre joueur (verifie dans
  `DataChannel.cpp` et `NetDriver.cpp` du moteur) ; les 57 proprietes repliquees sont toutes dans
  `GetLifetimeReplicatedProps` ; `SV_Item_*` verifie que le pion possede l instance.
- **Lifetime** : zero `this` nu dans un callback differe (tout est `CreateWeakLambda`), tous les
  `ProcessEvent` testent leur `UFunction*`, tous les acces tableau suspects sont bornes.
- **Statics locaux** : aucun etat de gameplay (18 statics, tous caches de classe ou etrangleurs de log).
- **Chemins d assets en dur** : 115, tous presents sur disque ET dans la build Steam.
- **TODO / FIXME / HACK / stub** : zero dans le plugin.
- **Reglages** : aucun defaut qui porte un nom de test, debug, cheat ou bypass. `Enabled=True`,
  loot actif, primes actives : voulus.

## 5. Regles de prod a respecter (pieges de sauvegarde, code actuel)

Ces trois points ne cassent rien AUJOURD HUI ; ils disent ce qu il ne faut plus faire une fois en live.

1. **Ne jamais renommer ni supprimer un `QMD_` en prod sans migration.** Au chargement, un tag de
   module inconnu fait sauter l entree avec un warning (`QModule_RackComponent.cpp:1128-1134`) : le
   module ET les phases inserees dedans sont perdus, sans credit au portefeuille, puis le mur tronque
   est REECRIT en base (`MarkRackDirty` ligne 1210, `SaveEntry` du PersistenceBridge). Correctif
   simple si on veut le filet : rembourser les phases au portefeuille avant de jeter l entree.
2. **Ne jamais faire tourner un serveur plus ancien que les sauvegardes.** Un en-tete de codec
   inconnu (`QMODSOCKETS;v1` attendu) fait `Sockets.Reset()` (ligne 1206-1209) puis sauvegarde le mur
   vide. Le format est versionne, la politique est "j efface" au lieu de "je refuse d ecrire".
3. **Trois dereferences `GetOwner()` non gardees dans les missiles d epaule**, une ligne chacune :
   `QModule_RackComponent.cpp:3425-3426` (le `&&` ne protege pas : si `GetOwner()` est nul, le second
   terme dereference nul), `QModule_RackComponent.cpp:264` (`EndPlay`) et
   `QModule_ShoulderMissileComponent.cpp:166` (`EndPlay`). Frequence du cas nul non demontree ; la
   faute de logique du premier est certaine. A corriger dans la prochaine passe, pas bloquant.

## 6. Dette non bloquante, a savoir

- `bUseLoggingInShipping = true` dans `Source/Qanga.Target.cs` et `QangaServer.Target.cs` : aucun log
  n est strippe en Shipping (choix projet, ~337 `UE_LOG` + 181 `QMOD_VLOG` dans QModule).
- `Sockets` et `MaxActiveSockets` repliquent a TOUS les clients (`COND_None`) sur un PlayerState
  toujours pertinent : a 500 joueurs, chacun recoit le mur de tous. Dette documentee dans le code
  ("owner-only optimization is a later perf pass").
- `bAllowItemlessModuleInstall = true` : un module sans item s installe gratuitement. 107 QMD sur
  disque pour 51 items. C est le seul defaut qui ressemble a un assouplissement de dev ; a confirmer
  comme voulu (decision du 28/08 : "un item n est plus requis pour installer").
- `FMemory_Alloca` sans plafond (`QModule_RackComponent.cpp:2068`) la ou les deux sites freres
  plafonnent a 64 octets ; deux `ProcessEvent` sur buffer zero sans `FStructOnScope`
  (`QModule_MedicalDroneActor.cpp:360` et `:441`) ; structs de parametres ecrites a la main validees
  par la seule taille (`QModule_LegacyFacade.cpp:424`, `:440`, `:485`, `:561`). Fragile, pas casse.
- Presentation qui tourne pour rien sur serveur dedie : moteur et porte du dropship (`BeginPlay` sans
  garde), transformations de la tourelle et de la grenade collante, toast de prime appele cote
  serveur par reflexion (`PayBounty`). Gaspillage, pas de crash.
- Deux commentaires perimes (`QModule_RackComponent.h:31` et `:150`).

## 7. Donnees : definitions, items, registre, distribution, cook, localisation

NON TERMINE : l audit des donnees (109 QMD, 115 items, DA_AllRef, tables de loot, quetes, cook,
localisation fr/es) etait en cours quand Benja a arrete la session (2026-09-12, 21:30). Acquis
partiels deja verifies ailleurs : les 8 tests QATS du plugin (cles d items dans DA_AllRef, scripts
d items valides, definitions valides) sont verts ; le manifeste Steam contient les 107 QMD et les
items QModule. Reste a mesurer : coquilles vides encore vendues ou lootees, doublons d items,
traductions fr/es des noms et descriptions (les 138 LevelDescriptions etaient culture-invariant
le 2026-08-22, donc jamais traduites).

Seul sous-lot termine : **les reliquats de chantier sous `Content`** (scan complet, lecture disque).
Trois choses a supprimer a la main (aucun referent, verifie), rien d autre dans les dossiers modules :

- `Content/Systems/Phase/PhaseComponent_BACKUP_PreFacade.uasset` (158 Ko, 10/07) : 0 referent dans
  Content, absent de la graine de cook et de Config ; cite seulement par deux docs.
- `Content/Items/Jetpack/IS_JetPack_BACKUP_PreV2Gates.uasset` (1,16 Mo, 26/08) : snapshot d avant les
  portes v2 du jetpack ; 0 referent dans EasyCook, Items, Systems, Widget, Phases, GameMode et
  Config (controle positif fait) ; les maps et Missions n ont pas ete balayees pour ce nom, donc
  `get_asset_dependencies` dans l editeur avant de supprimer.
- 4 dumps d outil d audit du 14/08 (`*.uasset.names.txt`) : `Systems/Item/ItemScriptBase`,
  `Items/QModuleCyborg/IS_QModuleCy_NidDeFrelons`, `Phases/QModuleV2/Loot/LDA_QMLoot_PoliceActive`
  et `LDA_QMLoot_PolicePassive`. Fichiers texte, aucun role runtime, ne partent pas dans le pak.

Et une confirmation : la v1 des phases (`Systems/Phase`, 104 referents pour `PhaseComponent` dont
68 maps, GameState, PlayerState, pawn joueur, `W_GameplayMenus`) est vivante et referencee par les
widgets v2 eux-memes. La garder en place est coherent ; la retirer serait un chantier, pas un menage.

## 8. Couplage Blueprint et etat des chantiers documentes

NON TERMINE, meme raison. Reste a mesurer : presence dans les .uasset des ~60 noms de fonctions et
proprietes BP que le C++ appelle par reflexion (liste complete en section 6 du rapport de l agent
C++, conservee dans la memoire de session), referents BP des fonctions publiques du plugin, entrees
roadmap en etat `check`, statut reel des bugs connus (double mort ServerKill, AppendOwnSquad,
marqueur de lock, etabli non inscrit dans QBuilder_Qanga_ActorData, WatercraftBase tague Hovercraft).

## 9. Passage pre-live vers live : mecanique

- Rien dans `Config/` n est specifique au pre-live (aucun flag, aucune adresse) : la branche live
  recevra le meme depot, meme binaire, meme pak. Version `Qanga_Version=0.0.23:2347`
  (`Config/DefaultGame.ini:253`).
- Le serveur dedie live doit etre mis a jour avec la MEME build que le client (le correctif du join
  du 22/08 etait cote client, le serveur pre-live n avait rien a changer ; c est deja le cas ici).
- Sauvegardes : la persistance v2 cree sa cle `QMODWall<coeur><PlayerId>` a la premiere connexion de
  chaque joueur, sans toucher aux cles legacy (`PhasePoints`, `PhaseData`). Rien a migrer en base
  pour que ca demarre ; voir 2.1 pour ce que ca implique.
- Tous les assets du plugin references depuis le code sont cuits (section 4), et les dossiers du
  systeme sont couverts par `+DirectoriesToAlwaysCook` (`DefaultGame.ini:133-140`, `:154`, `:159-168`).

### Checklist de mise en live

1. Trancher 2.1 (solde legacy) : option 1, 2 ou 3.
2. Rappel de flotte : appliquer le patch (ou pas) ; si oui, compiler, tester, repackager.
3. Ne pas repackager tant que le chantier FlyVehiclePawn de RzZz (arrive aujourd hui par Syncthing,
   `VehicleBase` reparente) n est pas compile ET valide en jeu : la build pre-live actuelle ne le
   contient pas, un repackage maintenant l embarquerait.
4. Aligner le test `VossCaptainHat` avec la donnee (20 %) ou l inverse.
5. Patch note : reprendre `#next_patchnotes`, couper au 10/07 (v0.0.22a), voir la memoire des sources.

## 10. Ce qui n a PAS ete verifie

- La base du serveur live (combien de joueurs ont `PhasePoints > 0` ou un `PhaseData` non vide).
- Le correctif du rappel de flotte en jeu : pas compile, pas teste.
- Le chantier FlyVehiclePawn : compilation lancee par Benja a 21:06, resultat non lu au moment d ecrire.
- Le corps des Blueprints atteints par reflexion (`Lib_Reward`, `SetVehicleState`, `Lib_Tracker`) :
  presence des noms verifiee (section 8), comportement non execute.
