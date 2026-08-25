# QSpaceTraffic : vie spatiale sur les routes de warp et dans le vide

### Document de cadrage v1 : EN ATTENTE D ARBITRAGE RzZz. Aucune ligne de code n a ete ecrite. Tout ce qui suit a ete mesure dans le code et les assets, pas suppose.

> Demande d origine (2026-08-22) : "le trafic aerien fonctionne bien et sert d exemple ; je veux
> deux choses, du trafic dans les routes de warp entre stations, et de la vie spatiale quand on
> voyage a vitesse normale dans le vide. C est un peu flou, j ai besoin d aide pour eclaircir."

---

## 0. Synthese en 12 lignes

1. Les deux objectifs demandes n ont pas la meme faisabilite. Le second (vie spatiale a vitesse normale) est une transposition directe du trafic aerien existant. Le premier (trafic dans les routes de warp) se heurte a un mur physique mesure, et doit etre redefini.
2. **Le mur** : un warp parcourt entre 13 km et 2 200 km **par image** a 60 fps. Aucun vaisseau physique n est perceptible dans un tunnel de warp, quelle que soit la distance. Voir 2.2.
3. **Le trou a combler est reel et il est gros** : sur les 105 segments possibles du reseau, **50 durent 150 s** (plafond de duree). La moitie des trajets de l univers, c est deux minutes trente d ecran vide. Voir 2.3.
4. Le graphe de routes existe deja en donnee : **15 stations** bakees en `QDB_WD_Warp_Station` dans `Content/_QData/WorldData/Warp/`, generees par l outil editeur `Make_Warp_Data`. Rien a re-authorer. Voir 1.3.
5. L univers a **deux echelles**, et le trafic ne peut pas etre uniforme : une grappe locale Terre de 8 stations dans 11 000 km, et 7 destinations planetaires isolees entre 1 et 20 millions de km. Voir 1.4.
6. Le systeme se decoupe en **trois chantiers**, dont deux partagent le meme moteur et ne different que par la regle de spawn. Voir 3.
7. **Chantier A** (vie spatiale ambiante) : transposition du `QAI_AerialTrafficSubsystem`, vrais pawns IA, factions, attaquables.
8. **Chantier B** (zones d approche des stations) : meme moteur, ancre sur les stations au lieu du joueur. C est la que la route devient lisible, parce que c est le seul endroit ou l on est assez lent pour voir.
9. **Chantier C** (compagnons de warp) : du visuel accroche a la spline du joueur, pas de l IA simulee. C est la seule reponse honnete au tunnel.
10. **Un bug bloquant est deja identifie dans le moteur de vol IA** : en croisiere, loin d une planete, le comportement melange 60 pour cent de vecteur "vers le bas" et fait piquer le vaisseau vers la planete la plus proche. `QAI_SpaceshipBehavior.cpp:234-252`. Aucun trafic spatial ne tient sans corriger ca. Voir 5.1.
11. Un second defaut est mesure et gratuit a corriger au passage : un line trace de 10 km par tick et par vaisseau dont le resultat n est jamais lu. `QSpaceshipMovementInterface.cpp:175-191`. Voir 5.2.
12. **Rien n est engage tant que la section 7 n est pas arbitree.** Quatre decisions y sont ouvertes, dont une qui peut supprimer purement et simplement le chantier C.

---

## 1. Ce qui existe, mesure

### 1.1 Le systeme de warp (100 pour cent Blueprint, `Content/Systems/Warp/`)

| Brique | Fait verifie |
|---|---|
| `WarpManager` | Composant **du `QangaGameState`** (resolu par `Lib_Warp.GetWarpManager` : `GetGameState` puis `GetComponentByClass`). Tient `WarpPointByLocationKey : Map<Vector, WarpPoint>` (registre des portes) et `ActorWarping : Map<Actor, WarpPoint>` (qui warpe, vers quoi). |
| Dispatchers | **`OnStartedWarp` et `OnEndWarp`** sont des event dispatchers du `WarpManager`. C est la source a brancher. Aucun timer de scrutation n est necessaire, conformement au paragraphe 5 du CLAUDE.md. |
| `WarpPoint` | S auto-enregistre au `BeginPlay`, se retire au `EndPlay`. Porte `WarpName`, `QDB_ID`, un `TrackerComponent` (le marqueur HUD), une `WV_DiscoveryArea`. Implemente `Warp_Interface` (`FocusedWarpPoint`, `UnfocusWarpPoint`, `RotateToWarpPoint`). |
| `SAT_WarpFilter` | Tache asynchrone **client**, relancee a chaque tick depuis `WarpManager.Event Tick`, qui calcule pour chaque porte l occlusion et les points d evitement, en tenant compte des acteurs WorldScape et des gravity areas. C est la regle "pas d obstacle entre le vaisseau et la cible". Alimente `LocalWarpPoint:Occluded` et `LocalWarpPoint:Evade`. |
| Declenchement | Le pawn doit porter le tag **`AllowWarp`**. Maintien de l input `Misc_Warp` pendant **2,1 s** (`Hold Input`, un tap annule), puis `TryWarp` cote client, puis `SV_RequestWarp` (RPC serveur), puis `ServerTryWarp` qui revalide le tag et resout la porte. Server-authoritative de bout en bout. |

### 1.2 La trajectoire de warp, exactement

Dans `WarpPoint.AddActorToMultiWarp` :

- **A** = position courante de l acteur. **B** = position de la station decalee de `max(bounds * 1,5 ; 55 000 uu)` vers l acteur, donc **on est depose 550 m avant la station**, jamais dedans.
- **Duree** = `MapRangeUnclamped(Distance, 0..80 000 000 000 uu -> 5..30 s)` puis `Clamp(5 ; 150)`. La reference est donc 800 000 km en 30 s.
- Un `SplineComponent` est construit par acteur : A, puis les points d evitement, puis B. L echantillonnage se fait en `GetLocationAtTime` avec `UseConstantVelocity = true`.
- Le profil temporel est un **`Ease` InOut de `BlendExp` 2,5** : depart et arrivee lents, milieu tres rapide.
- **Multi-warp** : plusieurs acteurs peuvent warper ensemble. `CheckIsNearRange` decale les temps d arrivee de 5 s pour eviter que tout le monde sorte au meme instant.
- Cote client uniquement (`Is Dedicated Server` = faux), un acteur `WarpEffectShip` (`/Game/Widget/HUDShip/PhysicalScreens/WarpEffectShip`) est spawn et attache pendant tout le trajet. **C est le point d accroche naturel du chantier C.**
- A la fin, `RemoveActorFromMultiWarp` detruit la spline et diffuse `WarpFinished`, qui remonte en `OnEndWarp`.

### 1.3 Le graphe de routes est deja une donnee

L outil editeur `Content/Systems/QDataBase/World_Data/_Tools/Make_Warp_Data` (EditorUtilityWidget) scanne les `WarpPoint` et `WarpPointQanga` du monde et genere des assets `QDB_WD_Warp_Station` portant `Data_ID`, `Data_Name`, `Data_Location`, `Data_Transform`.

**Il y en a 15**, dans `Content/_QData/WorldData/Warp/`.

C est une meilleure source que le registre runtime du `WarpManager` : ce dernier ne se remplit qu au `BeginPlay` de chaque porte, donc il depend du streaming `QLevel`, alors que les data assets sont lisibles immediatement, sur serveur dedie, sans qu aucune station ne soit chargee.

> Attention : l asset `QDB_Warp_WarpNaptune28021` porte une faute gelee dans son nom **et dans son ID**. C est un contrat au sens du paragraphe 4 du CLAUDE.md. Ne pas le "corriger".

### 1.4 La geometrie reelle du reseau

Positions extraites des 15 data assets, distances et durees calculees avec la formule du 1.2.

| Station | Distance / WarpEarth | Duree de warp | Vitesse moyenne |
|---|---:|---:|---:|
| WarpIcliSpace | 3 995 km | 5,1 s | 780 km/s |
| OrbitalEarthAustralia | 4 166 km | 5,1 s | 812 km/s |
| OrbitalEarthEurope | 4 283 km | 5,1 s | 834 km/s |
| OrbitalEarthAsia | 4 645 km | 5,1 s | 903 km/s |
| OrbitalEarthSouthAmerica | 6 931 km | 5,2 s | 1 329 km/s |
| OrbitalEarthDoorNorthAmerica | 8 086 km | 5,3 s | 1 539 km/s |
| WarpMoon | 10 738 km | 5,3 s | 2 012 km/s |
| WarpVenus | 1 013 270 km | 36,7 s | 27 636 km/s |
| WarpMars | 1 295 129 km | 45,5 s | 28 481 km/s |
| WarpMercury | 2 544 455 km | 84,5 s | 30 107 km/s |
| WarpJupiter | 6 387 251 km | **150 s** | 42 582 km/s |
| WarpSaturn | 11 525 146 km | **150 s** | 76 834 km/s |
| WarpUranus | 16 493 103 km | **150 s** | 109 954 km/s |
| WarpNaptune | 19 812 449 km | **150 s** | 132 083 km/s |

Segment le plus court : `OrbitalEarthEurope` vers `WarpIcliSpace`, 522 km, 5,0 s.
Segment le plus long : `WarpSaturn` vers `WarpUranus`, 22 620 187 km, 150 s.

> Methode : lecture binaire des `.uasset` hors editeur, chaque position confirmee par sa double occurrence dans le fichier. **A recouper par une commande editeur** avant de coder quoi que ce soit dessus. C est le seul chiffre de ce document qui ne vient pas d une lecture directe de code.

### 1.5 Le modele a suivre : le trafic aerien

`Plugins/QAI/Source/QAI/Private/Spaceship/QAI_AerialTrafficSubsystem.cpp`, un `UWorldSubsystem` serveur :

- boucle sur timer (`SpawnIntervalSeconds`), au plus un spawn par tick, ce qui sert de limiteur naturel ;
- budget `MaxShipsPerPlayer`, entree entre `MinSpawnDistanceM` et `MaxSpawnDistanceM`, decentrage lateral `MaxRouteOffsetM` pour que les routes ne convergent pas toutes sur le joueur ;
- despawn au dela de `DespawnDistanceM` **de tous les joueurs** ;
- roster pondere par faction, **bake en C++** dans `QAI_AerialTrafficSettings.cpp` (le `.ini` ne survivait pas au packaging) ;
- vaisseaux crees par `UQAI_SubSystem::SpawnAerialAgent`, qui monte les composants du moteur de vol IA et cree l agent QAI.

C est ce squelette qu il faut reprendre, pas reinventer.

---

## 2. Les trois chiffres qui contraignent tout

### 2.1 Un vaisseau IA plafonne a 3 200 km/h

`FlyVehicleMovementComponent.h:393` : `MaxSpeedKmH = 3200` (`MaxSpeedCmS = 88888`). Croisiere par defaut 300 km/h, boost 400. Le levier par vaisseau existe deja : `VehicleNormalSpeedLimitKmH` et `VehicleBoostSpeedLimitKmH`.

Consequence : dans le vide, du trafic a 300 km/h paraitra fige. Le reglage de vitesse est un parametre de premier ordre du chantier A, pas un detail de finition.

### 2.2 Un warp parcourt de 13 km a 2 200 km par image

Le warp le plus lent du jeu (Terre vers IcliSpace, 780 km/s) avance de 13 km par image a 60 fps. Le plus rapide (Neptune, 132 083 km/s) de 2 201 km par image.

Un vaisseau de 30 m est donc traverse en un quatre-centieme d image dans le meilleur des cas. **"Croiser du trafic pendant le warp" au sens propre est geometriquement impossible.** Ce qui remplira le tunnel devra se deplacer avec le joueur.

L ease InOut nuance a peine : le depart et l arrivee sont lents, mais c est deja couvert par le fait qu on est depose 550 m avant la station et qu on y termine en vol normal. C est le chantier B, pas le chantier C.

### 2.3 La moitie des trajets durent 150 s

Sur les 105 paires de stations, **50 atteignent le plafond de duree de 150 s**, et 28 sont des sauts courts de 5 s dans la grappe Terre.

C est le fait le plus structurant du document. Il ne dit pas seulement "il faudrait de la vie dans le warp", il dit **"il y a deux minutes trente d ecran vide, une fois sur deux"**. Cela deplace le chantier C du confort vers le ressenti de base, et cela ouvre une alternative qui n est pas du trafic : reduire le plafond, ou imposer des sauts par paliers. Voir 7.1.

---

## 3. Le decoupage propose

### Chantier A : vie spatiale ambiante (l objectif 2 de la demande)

Autour de chaque joueur en vol normal dans le vide, des vaisseaux IA entrent de loin, traversent, repartent. Vrais pawns, vraies factions, attaquables, reactifs comme dans le ciel.

Transposition du `QAI_AerialTrafficSubsystem`, en retirant **toute** la machinerie sol : plus de `WS_SingleProject`, plus de `GroundDistance`, plus de plancher d urgence, plus de tenue d altitude. La geometrie de spawn devient spherique autour du joueur au lieu d etre horizontale a une altitude donnee.

### Chantier B : zones d approche des stations (le "trafic de route" reellement visible)

Meme systeme, autre regle de spawn : ancre sur les stations du 1.3 au lieu du joueur. Des vaisseaux arrivent, repartent, attendent. Les trajectoires s alignent sur les segments station vers station, ce qui **rend la route lisible** au seul moment ou l on est assez lent pour la voir.

A et B partagent le meme sous-systeme et le meme roster. Ce sont deux modes de spawn, pas deux systemes.

### Chantier C : compagnons de warp (l objectif 1, version faisable)

Pendant le tunnel, des vaisseaux qui warpent **avec** le joueur, donc quasi immobiles dans son referentiel. Ce ne sont pas des agents IA simules : c est du visuel accroche a la spline, sur le `WarpEffectShip` qui existe deja et qui est deja client-only.

C est la seule facon de repondre a "je ne suis pas seul dans le couloir" sans mentir sur la physique du 2.2.

### Priorite recommandee

**A, puis B, puis C.** A debloque B (meme moteur, meme correctif du 5.1), et B est celui qui donne le plus de sensation de route. C ne se justifie que si la reponse au 7.1 est "on garde les 150 s".

Et une precision d echelle : A et B ont un interet fort **dans la grappe Terre** (8 stations, 28 segments courts, c est la que les joueurs vivent) et quasi nul autour de Neptune. Autant y concentrer l effort.

---

## 4. Plan par jalons

Chaque jalon compile, se teste seul, et ne casse pas le trafic aerien ni la police spatiale.

| Jalon | Contenu | Validation |
|---|---|---|
| **J0** | Mesure. Recouper les 15 positions du 1.4 par commande editeur. Mesurer le comportement reel d un vaisseau IA lache dans le vide (confirmer le piquage du 5.1). | Un log, pas une hypothese. |
| **J1** | Mode espace dans le moteur de vol IA : detection "hors atmosphere" et court-circuit du bloc sol. Reveille le `bInSpace` mort du 5.2 au lieu d en ecrire un nouveau, en analytique (`WS_GetAltitudeInLocation`) plutot qu en trace. | Un vaisseau lache dans le vide vole droit. Le trafic aerien et la police restent identiques. |
| **J2** | `UQSpaceTrafficSubsystem` : squelette calque sur le trafic aerien, mode A (ambiant autour du joueur), roster bake en C++, reglages en `UDeveloperSettings`. | Trafic visible en vol normal dans le vide, en PIE puis en dedie. |
| **J3** | Mode B : lecture des 15 stations, generation des segments, spawn ancre sur les zones d approche. | Une station est vivante quand on en sort. |
| **J4** | Reglage : vitesses (2.1), densite, mix de factions, budget par joueur. | Passe de ressenti avec RzZz. |
| **J5** | Chantier C, **si et seulement si** 7.1 le maintient. | Passe de ressenti sur un trajet Neptune. |

Le decoupage suit la regle "copier, rerouter, supprimer" qui a marche pour l extraction du moteur de vaisseau vers QAI : jamais de fenetre cassee.

---

## 5. Risques de regression identifies

### 5.1 Le piquage vers la planete (bloquant, deja mesure)

`Plugins/QAI/Source/QAI/Private/Behavior/QAI_SpaceshipBehavior.cpp:234-252`. En etat `Patrol`, le code calcule `AltError = TargetAltitude - GroundDistance`. Loin d une planete, `GroundDistance` est enorme, donc `AltError` est massivement negatif, donc `Blend` sature a 0,6 et le vecteur `LocalUp * sign(AltError)` pointe **vers le bas**. Le vaisseau recoit 40 pour cent d avant et 60 pour cent de "vers la planete".

Ce bloc a ete ajoute pour corriger le nez qui piquait des vaisseaux du ciel sur une planete courbe. Il est correct dans son domaine et faux hors atmosphere. Il faut le **conditionner**, pas le supprimer : le trafic aerien et la police en dependent.

### 5.2 Un raycast de 10 km par tick et par vaisseau, jamais lu

`Plugins/QAI/Source/QAI/Private/Spaceship/QSpaceshipMovementInterface.cpp:175-191`. `bInSpace` est declare, calcule par `LineTraceSingleByChannel` sur un million d unites, et **n est lu nulle part dans le fichier**. Cout pur. C est aussi le crochet naturel du J1 : reveiller cette variable plutot qu en introduire une autre.

### 5.3 Reseau

L historique du trafic aerien a coute trois allers-retours sur ce point precis, il ne faut pas les repayer :

- le roster en `.ini` ne survit pas au packaging, d ou le bake C++ ;
- `SetReplicateMovement(true)` sur un vaisseau de trafic **casse** la simulation cliente au lieu de l aider ;
- le modele qui marche est l agent QAI cote client, cree par la boucle de decouverte de `QAI_SubSystem`, declenchee par un composant marqueur.

Le chantier spatial herite de ces conclusions au lieu de les redecouvrir.

### 5.4 Echelle et precision

L univers s etend a environ 2 x 10^12 unites du centre. C est du domaine LWC : toute API Blueprint en `FVector` ou `float` perd la precision metrique bien avant. Les surcharges `DVector` de WorldScape sont obligatoires sur les positions absolues.

### 5.5 Streaming et budget

`QLevel` gere le streaming par octree et peut entrer en conflit avec un spawn ou une destruction naifs. Le budget QAI est partage (`MaxAgent` a 1024 dans le `.ini`, pas 4096 comme dans le constructeur). Le trafic spatial doit posseder son propre despawn par distance, comme le trafic aerien, et ne pas dependre du cull de faune a 500 m.

---

## 6. Ce qui n est PAS dans le perimetre

- Toucher au systeme de warp lui-meme (duree, spline, occlusion, cout). Sauf arbitrage explicite du 7.1.
- Toucher a la police spatiale ou a son declenchement par le niveau de recherche.
- Toucher a QAssistance (le reseau de voyage rapide planetaire, 200 stations, deja fonctionnel et payant).
- Toute conversion des appels par reflexion du moteur de vaisseau vers des appels types. C est un chantier separe, deja repousse une fois a raison.
- Le renommage de quoi que ce soit, y compris la faute `Naptune` du 1.3.

---

## 7. Decisions attendues de RzZz

### 7.1 Les 150 s, on les remplit ou on les reduit ?

C est la question qui commande le chantier C. Trois reponses possibles, exclusives :

- **a) On les garde et on les remplit** : le chantier C se justifie, on met du compagnon de warp.
- **b) On reduit le plafond de duree** : un reglage dans `WarpPoint`, le probleme disparait, le chantier C tombe.
- **c) On impose des sauts par paliers** de station en station : change le design de navigation, supprime les trajets longs, et rend le reseau du 1.4 beaucoup plus vivant par construction.

**Cette decision ne se prend pas dans le trafic, mais elle decide de ce que le trafic doit accomplir.**

### 7.2 "Les marqueurs de route spatiale", c est quoi exactement ?

Les `TrackerComponent` des portes existantes, ou une **ligne de route dessinee** entre deux stations, qui n existe nulle part aujourd hui et serait a creer ?

### 7.3 Quelle nature de trafic ?

Du civil qui ignore le joueur, comme le ciel aujourd hui ? Ou tout de suite des convois, des pirates qui campent les routes, des patrouilles ? La reponse change le roster, le mode combat, et le rapport avec le systeme de recherche.

### 7.4 Les stations vivent-elles ?

Vaisseaux amarres, files d attente, navettes en va-et-vient court : cela change beaucoup l ambiance du chantier B et son cout. A trancher avant J3, pas avant J1.

---

*Document redige le 2026-08-22. Toutes les references de fichier et de ligne ont ete lues, a l exception des positions du 1.4 (extraction binaire, a recouper). Le pont CLIScape etait instable pendant la redaction : les resumes Blueprint proviennent de `get_asset_summary` quand il repondait, et de lecture binaire sinon.*
