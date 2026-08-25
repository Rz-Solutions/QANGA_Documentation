# Retour joueur Terpen (prelive) : audio des armes, points de phase, axes HOTAS

- **Source** : Terpen, Discord, 2026-08-25 14:42, test du dernier build sur le serveur prelive.
- **Analyse** : session Claude du 2026-08-25, editeur ouvert, valeurs mesurees sur les assets reels
  (pas de lecture de doc ni de supposition). Aucune modification faite sur le projet.
- **Destinataire** : RzZz (correction).

## Verbatim du joueur

> I tried the latest build on the prelive server, but :
>
> * I find the noise of the NPC's weapons still too loud. It seems to me that they are even louder
>   than mine. And going inside a building does't help : the sound of guns fired outside is not muffled
> * My cyborg is level 14 and has lost all his phase points, so I guess I'll have to restart from the beginning
> * The bindings of the buttons are OK but the analog bindings of my hotas are lost : it has happened
>   with every update, but now I can't rebind them : moving a joystick while trying to bind an axis
>   has no effect. I couldn't fly my ship. I will try to reinitialize them.

## Portee : ce n'est pas specifique au prelive

Les trois causes vivent dans des assets, une config et du C++ partages par toutes les cibles de build.
Aucune n'est derriere un flag prelive. Sauf divergence de branche, **les joueurs du serveur public sont
exposes aux memes trois defauts**. A confirmer par RzZz sur la branche publique avant communication.

---

## 1. Armes des NPC trop fortes, et aucun etouffement en interieur

Ce sont deux defauts distincts qui se cumulent dans le ressenti.

### 1a. Le niveau relatif

Une meme arme a deux voies, arbitrees par `ShouldPlayLocalWeaponAudio`
(`Plugins/QWeapon/Source/QWeapon/Private/QWeaponAudioLibrary.cpp:235`), cablees dans
`IS_NashV2_Assault_Rifle` / `IS_NashV2_MachineGun` :

| | son joue | gain | attenuation |
|---|---|---|---|
| Arme du joueur local | `PlayerNashAR` | 1.5 (asset) x 1.0 (pin) | **aucune** |
| Arme d'un tiers / NPC | `DistantNashAR` | 1.0 (asset) x 0.9 (pin) | `Guns_Attenuation` |

Le tir des IA en voie directe (`QAI_CombatProcessor.cpp:437`) passe par
`PlayDefaultDistantRifleShotAtLocation`, meme son et meme 0.9 : coherent avec la voie BP.

Sur le papier le joueur est donc plus fort. Le ressenti inverse vient des valeurs de
`Guns_Attenuation` (`/Game/Items/Weapons/NashV2/SFX_V2/Guns_Attenuation`), mesurees :

- `FalloffDistance = 120000` (1,2 km) avec **`dBAttenuationAtMax = -18 dB` seulement**. La courbe est
  trop plate : un tir a 300 m arrive quasiment a pleine puissance.
- `bAttenuateWithLPF = True` mais **`LPFFrequencyAtMin = LPFFrequencyAtMax = 20000 Hz`**. Les deux
  bornes etant identiques et a 20 kHz, **le filtre ne fait rien du tout**. Un tir lointain garde
  exactement le timbre d'un tir a 3 m, ce que l'oreille interprete comme "proche et fort". C'est
  probablement le facteur dominant du ressenti, avant meme le gain.
- Concurrency `SCON_Guns_Limit` : `MaxCount = 30` mais **`bLimitToOwner = True`**, donc la limite
  s'applique par tireur. Rien ne borne la somme quand plusieurs NPC tirent ensemble, et
  `VolumeScale = 1.0` (pas de ducking a l'empilement).

Pistes (valeurs d'assets uniquement, zero C++, reversible) : remonter `dBAttenuationAtMax` vers une
vraie perte de distance, donner au LPF deux bornes differentes (typiquement plein spectre au plus pres,
quelques kHz au plus loin) pour que la distance s'entende, et ajouter une concurrency **globale** pour
les tirs de tiers en plus de la limite par proprietaire.

### 1b. L'interieur ne change rien

Le mecanisme existe cote assets : `Tail-Indoor` et `Tail-Outdoor` sont presents, et les MetaSounds
(`DistantNashAR`, `PlayerNashAR`, les `Tail-*`) exposent une entree `IsInterior`.

**Il n'est jamais active.** Le C++ ecrit `IsInterior = false` en dur sur les trois sites de lecture
(`QWeaponAudioLibrary.cpp:141`, `:227`, plus le composant de queue). Aucun code du projet ne calcule si
l'auditeur est a l'interieur : la recherche `IsInterior` sur `Plugins/` ne remonte que ce fichier.

Second facteur : l'occlusion est bien activee dans `Guns_Attenuation`
(`bEnableOcclusion = True`, `OcclusionLowPassFilterFrequency = 1000`), mais
**`OcclusionVolumeAttenuation = 1.0`**, donc elle ne baisse pas le volume d'un decibel, elle ne pose
qu'un passe-bas. Le trace utilise `ECC_Pawn`, a verifier sur les batiments concernes.

Pistes : baisser `OcclusionVolumeAttenuation` (gain immediat, asset seul), puis alimenter reellement
`IsInterior` (demande du C++ : determiner l'etat interieur de l'auditeur et le passer aux trois sites
au lieu du litteral `false`).

---

## 2. Points de phase perdus au niveau 14

**Hypothese forte, coherente avec le code, non prouvee sur les donnees du serveur.** A verifier en base
avant toute correction ou communication au joueur.

QModule V2 est `Enabled=True` depuis le 2026-07-18 (`Config/DefaultGame.ini:536`) et
`QModule_LegacyPhaseSwap` remplace l'ecran historique `W_PhaseTree` par le mur V2. Or il existe
**deux stocks de points sans passerelle entre eux** :

- ancien : `SS_Phase` sur le PlayerState, clef persistee `"PhasePoints"`, achat via `SV_UpPhase` ;
- nouveau : `PhaseWallet` du `QModule_RackComponent`, persiste sous `QMODWall<coeur><PlayerId>`
  par `QModule_PersistenceBridge_World_SubSystem`.

Recherche faite sur tout `Plugins/QModule` : **aucun code de conversion du solde legacy vers le wallet
V2**. La seule migration presente convertit des items phase egares dans le sac
(`QModule_RackComponent.cpp:552`). Le plan d'architecture prevoyait pourtant ce grandfathering.

Et dans V2 la **seule** source de points est le level-up
(`HandleLegacyLevelUp` -> `Authority_CreditPhase(1, Delta)`, `QModule_RackComponent.cpp:822`) ;
`Authority_CreditPhase` n'etant pas une UFUNCTION, aucune quete ne peut en crediter.

Consequence attendue : un joueur ayant gagne ses niveaux **avant** la bascule n'a jamais ete credite
dans le wallet V2. Son solde d'origine est vraisemblablement toujours en base, simplement plus lu par
l'ecran. Ce n'est donc pas forcement une perte de donnees, plutot un stock devenu invisible.

A verifier sur le prelive pour ce joueur :
1. presence et valeur des clefs `PhasePoints` et `PhaseData` (encodage `TagName<pique>Level`) ;
2. contenu de son `QMODWall<coeur><PlayerId>` (entrees `Sockets`, ligne wallet incluse) ;
3. que son `PlayerId` (`ServerAuth.GetPlayerId`) n'a pas change, ce qui creerait une ligne neuve et
   donnerait le meme symptome par une cause differente.

Selon le resultat : passerelle de conversion one-shot au premier login (versionnee), ou compensation
manuelle des joueurs concernes.

---

## 3. Axes HOTAS perdus, et rebind qui ne repond pas

Le point le plus bloquant : le joueur ne peut plus piloter son vaisseau.

### 3a. Pourquoi les axes se perdent alors que les boutons tiennent

`Saved/SaveGames/InputConfig.sav` (classe `InputSaveData_C`) contient **deux maps** :

- boutons : clef = **chemin d'asset** du preset (`SoftObjectProperty`), valeur = `InputCombos` ;
- axes : clef = `AnalogKeyId` (FName), valeur = `FCustomAnalogData`, qui stocke un `FKey`.

Les axes sont donc bien sauvegardes. Le point fragile est le **nom de la FKey** : JoystickPlugin
nomme ses cles `Joystick_<NomDevice>_<InternalDeviceIndex>_Axis<N>`
(`Plugins/JoystickPlugin/Source/JoystickPlugin/Private/JoystickInputDevice.cpp:96`), et
`InternalDeviceIndex` vient de l'ordre d'enumeration SDL. Si cet index bouge (rebranchement, autre
peripherique present, mise a jour), la cle sauvegardee ne correspond plus a aucune cle enregistree et
l'axe devient muet, **sans erreur ni message**. Les boutons du joueur etant vraisemblablement au
clavier, ils ne subissent pas ce nommage : cela explique l'asymetrie qu'il decrit.

### 3b. Pourquoi le rebind ne reagit pas

Mesure sur le graphe de `W_InputAnalogAxisSetting` :

- l'auto-detection ne demarre **que** sur `Pre Open` de la liste deroulante d'axes
  (`Pre Open -> UpdateAvailableAxis -> LoopAutoDetection`), pas quand on est simplement sur la ligne.
  Premiere question a poser au joueur : ouvrait-il bien la liste deroulante ?
- `UpdateAvailableAxis` ne liste que les axes vus par le `JoystickSubsystem` (SDL) :
  `GetInstanceIds` -> `GetJoystickState` -> `GetJoystickKeyByIndex`. Les axes RawInput
  (`GenericUSBController_Axis*`) ne sont **jamais** scrutes ; la map `RawInput:Index` du widget ne sert
  qu'a la conversion d'affichage en repli.
- si cette liste est vide, la garde `Length > n` de `LoopAutoDetection` empeche la boucle de demarrer :
  bouger le manche ne **peut** rien produire, et rien ne le signale a l'ecran.

### 3c. Le terrain : deux backends d'entree en parallele

`JoystickPlugin` et `RawInput` sont **tous les deux actifs** dans `QANGA.uproject`.

`Config/DefaultInput.ini:521` pose `IgnoreGameControllers=True`. Consequence
(`JoystickSubsystem.cpp:617` et le helper `IsUEHandledGameController` ligne 39) : JoystickPlugin
**rejette** tout peripherique vu comme game controller par SDL, c'est-a-dire `SDL_IsGamepad` vrai
**ou** chemin HID Windows contenant `&IG_`. Un HOTAS disposant d'un mapping SDL (les Thrustmaster
T.Flight par exemple) disparait donc entierement de la liste des axes, et le symptome decrit est exact.

Chemin d'echec supplementaire, silencieux cote joueur : dans `Initialize()`, si SDL RAWINPUT est deja
initialise, la fonction **sort avant d'initialiser SDL** (`JoystickSubsystem.cpp:100-103`,
`return` prematurie, `bIsInitialised` reste faux). Dans ce cas plus **aucun** joystick n'existe pour le
jeu, et seule une ligne `LogJoystickPlugin` d'erreur en temoigne.

### A demander a Terpen (tranche en une minute)

1. le modele exact de son HOTAS (marque et reference des deux ou trois peripheriques) ;
2. son `InputConfig.sav` ;
3. **son log de jeu** : `EnableLogs=True` est actif, la categorie `LogJoystickPlugin` liste les devices
   ajoutes et les cles creees, ou l'erreur d'initialisation. C'est ce qui separe les branches 3a, 3b et 3c.

---

## Dette relevee au passage, hors perimetre, non touchee

`Plugins/QAI/Source/QAI/Private/Processor/QAI_CombatProcessor.cpp:154` et `:407` utilisent des `static`
locaux (`LastSoundTime`, `SoundBudgetFrame`, `SoundsThisFrame`) pour le budget sonore de la foule.
Le CLAUDE.md paragraphe 6 l'interdit : cet etat est partage entre mondes et fausse le comptage en PIE
multi-monde. A traiter dans une tache dediee.
