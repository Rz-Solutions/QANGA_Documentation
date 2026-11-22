# QANGA à la manette : état des lieux mesuré et plan

Relevé du 2026-11-21, lecture seule. Rien n'a été modifié dans le projet.
Cible : jouable à 100 % à la manette (Xbox, PlayStation, Steam Controller, Steam Deck, Steam Machine sous SteamOS), avec la démo Winter Sale (17/12) en ligne de mire.

## 0. Comment ça a été mesuré, et ce que ça ne prouve pas

- Code C++ des plugins : lu directement.
- Assets : recherche binaire **complète et terminée** sur `Content/` et `Plugins/` (tous les `.uasset` et `.umap`, environ 5 min par passe), sur les noms de touches, de classes et de fonctions. Les noms d'assets, de classes, de variables et de touches (`FKey`) vivent dans la table de noms en ASCII : un résultat positif est sûr. Un résultat négatif est une forte présomption, pas une preuve (un littéral de graphe peut être en UTF-16).
- **Aucun graphe Blueprint n'a été ouvert** : l'éditeur était tenu par une autre session (`Saved/VERROU_EDITEUR.txt`). Tout ce qui est dit sur la logique d'un graphe vient des noms qu'il contient. C'est marqué « (à confirmer en jeu) ».
- **Aucun test manette réel** : je n'ai pas de manette en main. La liste de test pour Benjamin est au §6.

## 1. La chaîne d'entrée, telle qu'elle est

```
Matériel
  XInput (Windows, UE natif)  -> touches Gamepad_*        <- seule voie native pour une manette
  SDL3 (JoystickPlugin)       -> touches Joystick_*       <- HOTAS / volants ; IgnoreGameControllers=True rejette les manettes
  RawInput (Epic)             -> GenericUSBController_*
        |
  PlayerController : EnhancedPlayerInput / EnhancedInputComponent (capture bas niveau seulement)
        |
  UInputsComponent (C++, plugin InputSystem) : binding AnyKey pressé/relâché, tick, combos
    -> InputsComponentBP (BP du plugin) : lit les deux sticks et les gâchettes
         (GetInputAnalogStickState), expose Input_GamepadThumbStickL/R, Input_GamepadTriggerL/R,
         Input_MouseAxis, CurrentControllerType (Keyboard / Mouse / Gamepad),
         FlipGamepadStickSide, FlipGamepadTriggerSide
    -> Qanga_InputsComponent (Content/Inputs) : déplacement, caméra, visée, catégories
         Camera_LookRightLeft / LookUpDown = MouseAxis + presets analogiques CameraY/CameraZ,
         réglages AimSpeed, GamepadAimSpeed, InvertHAxis, InvertVAxis, HoldToAim
         Char_MoveForwardBackward / RightLeft = touches Char_Move* + presets MoveX/MoveY
        |
  Actions = UInputPreset_DA (boutons) et UAnalogInputPreset_DA (axes), dans Content/Inputs/
  Rebind  = W_InputSettings (classe C++ UInputSettingsWidget) + W_InputCombo + W_InputAnalogAxisSetting
  Sauvegarde = Saved/SaveGames/InputConfig.sav (grave la combo dès la 1re utilisation d'un preset)
```

Config (`Config/DefaultInput.ini`) :
- zone morte des sticks : `+AxisConfig` Gamepad_LeftX/LeftY/RightX/RightY **DeadZone=0.005** (les lignes `-AxisConfig` à 0.25 sont retirées). Même valeur par défaut dans `FCustomAnalogData::Deadzone`. C'est quasi nul : une manette usée dérive.
- `[/Script/JoystickPlugin.JoystickInputSettings] IgnoreGameControllers=True` : SDL ignore tout ce qu'il reconnaît comme manette. Voulu (sinon doublon avec XInput), mais conséquence : sous Windows **une DualSense, une manette Switch ou une manette générique non XInput n'est vue que si Steam Input la traduit en manette Xbox**.
- `Config/DefaultEngine.ini` : `GameViewportClientClassName=CommonGameViewportClient`, `Slate.RequireFocusForGamepadInput=1`. Aucune section `CommonInputSettings` (pas d'`InputData`, pas de données de manette CommonUI).
- Plugins actifs : CommonUI, EnhancedInput, JoystickPlugin, RawInput, SteamCore.

## 2. Ce qui existe déjà (à réemployer, pas à refaire)

| Brique | Où | État mesuré |
|---|---|---|
| Lecture des sticks et gâchettes | `Plugins/InputSystem/Content/InputsComponentBP` | Délégués `Input_GamepadThumbStickL/R`, `Input_GamepadTriggerL/R`. **Aucun abonné trouvé** dans tout le projet hors ce BP. |
| Type de contrôleur courant | `InputsComponentBP.CurrentControllerType` (enum `E_ControllerType` : Keyboard, Mouse, Gamepad) | Lu par `Lib_InputSystem.GetCurrentControllerType` |
| Texte de touche selon le contrôleur | `Lib_InputSystem` : `GetCurrentInputForControllerType`, `GetTextForCombo`, `ReduceToShortInputName`, `GetInputKeyText` | Existe. Affiche du **texte** (nom de touche), pas d'icône |
| Convention « combo 0 = clavier, combo 1 = manette » | `DynamicQuestSystem/.../QuestNotificationUtils.cpp:43` (`ResolveInputComboIndex` via `UCommonInputSubsystem::GetCurrentInputType`) | Utilisée par les notifications de quête |
| Navigation de menus à la croix | `Plugins/WidgetBase` : `W_SuperClassBase` (OnKeyDown, OnFocusReceived, DPad, bouton A, LB/RB), `WidgetMacros`, `Lib_WidgetNavigation` | **73 widgets** référencent `W_SuperClassBase` : réglages (19), lobby V1 et V2 (6), inventaire (5), véhicule (5), création de personnage (3), boutique (2), `W_GameplayMenus`, `W_SystemMenus`, `W_MissionMenu`, groupe, clan, crédits... |
| Dialogues PNJ | `QuestDialoguePopupWidget.cpp`, `QuestDialogueChoiceWidget.cpp` | A, croix, stick gauche, X pour revenir : **gérés en C++** |
| Fermer une notification | `QNotificationModule.cpp:26` | Start / Select ferment |
| Rebind manette des boutons | `W_InputCombo` (`HorizontalBox_GamepadSlots`, `IgnoreDoubleGamepadKey`) + `UInputSettingsWidget` | Une colonne manette existe dans l'écran de réassignation |
| Rebind manette des axes | `InputSettingsCaptureProcessor.cpp:138` + `InputSettingsWidget.cpp:811` | Capture un axe de stick ou de gâchette et règle min/milieu/max automatiquement |
| Vitesse de visée manette | `GamepadAimSpeed` dans `Qanga_InputsComponent` et `W_GameplaySettings` | Réglage exposé au joueur |
| API Steam | `Plugins/SteamCore` : `ShowGamepadTextInput`, `IsSteamRunningOnSteamDeck`, `IsSteamInBigPictureMode`, toute l'API Steam Input (`ActivateActionSet`, `GetDigitalActionData`, `GetGlyphForActionOrigin`...) | Disponible, **aucun appel trouvé** dans le contenu |

## 3. Touches manette par défaut (les 34 presets qui en ont une)

| Bouton | Presets |
|---|---|
| A (FaceButton_Bottom) | Char_Jump, JetPack_FlyUp, Spaceship_CombatTrigger1 |
| B (FaceButton_Right) | Char_Crouch, JetPack_FastFlightToggle, Spaceship_CombatTrigger2, Vehicle_Brake |
| X (FaceButton_Left) | Interact |
| Y (FaceButton_Top) | Combat_Reload, Spaceship_StationaryMode |
| LB | Char_Walk, Jetpack_RollLeft, Hovercraft_MoveLeft, Spaceship_MoveLeft, Combat_Repair (avec RB) |
| RB | Char_Sprint, Jetpack_RollRight, Hovercraft_MoveRight, Spaceship_MoveRight |
| LT (bouton) | Camera_AimMode |
| L3 | Combat_Melee, Spaceship_Boost |
| R3 | Camera_Viewmode, Misc_PushToTalk |
| Croix haut | Misc_Lights, Vehicle_EngineControl |
| Croix bas | Combat_SwitchAimpoint, Spaceship_GravityAssistance |
| Croix gauche | Char_InclineLeft, Spaceship_LandingGear |
| Croix droite | Char_InclineRight, Vehicle_Honk |
| Start | UI_OpenMenu |
| Select | Ragdoll |

**RT n'est utilisé nulle part.** Les 16 presets analogiques (`Content/Inputs/Analogs/` : MoveX, MoveY, CameraY, CameraZ, ShipMove*, ShipRot*, HoverMove*, HoverRotZ, VehicleBrake) n'ont **aucune touche d'axe par défaut** : leur `DefaultAnalogData` ne porte que `Scale`, `AnalogAxis` reste vide (mesuré dans `MoveX.uasset`).

## 4. Contexte par contexte

Légende : OK = présent ; PARTIEL ; NON = rien trouvé ; ? = à confirmer en jeu.

| Contexte | Liaisons manette | Ce qui manque |
|---|---|---|
| **À pied : se déplacer** | NON par défaut (?) | MoveX/MoveY sans axe, Char_Move* sans touche manette, délégué stick gauche sans abonné. Le joueur peut assigner le stick à la main dans les réglages. |
| **À pied : regarder** | NON par défaut (?) | CameraY/CameraZ sans axe. `GamepadAimSpeed` existe, donc la visée stick a été prévue, mais rien n'est branché par défaut. |
| À pied : sauter, s'accroupir, sprint, marche, pencher | OK | |
| **Interagir** | OK (X) | |
| **Combat : tirer** | NON | `Combat_1stTrigger` et `Combat_2ndTrigger` sans touche manette. RT libre. |
| Combat : viser, recharger, mêlée, réparer, point de visée | OK (LT, Y, L3, LB+RB, croix bas) | |
| Combat : armes 1 à 4, consommable, recycleur, soin, gadget (tir, roue) | NON | Combat_Slot1-4, Combat_Consumable, Combat_Recyler, Char_Heal, Char_GadgetFire, Char_GadgetWheel |
| Visée assistée | NON | Aucun nom `AimAssist` ni équivalent trouvé |
| **Jetpack** | PARTIEL | Monter (A), roulis (LB/RB), vol rapide (B). Direction = déplacement à pied, donc rien. Descente, stationnaire, esquives : rien. |
| **Véhicules terrestres** | PARTIEL | Moteur, klaxon, frein (B). **Accélérer, ralentir, boost, dérapage, porte : rien.** Direction = ? (analogique VehicleBrake vide). Radio : rien. |
| **Vaisseau** | PARTIEL | Tir 1/2 (A/B), boost, train, gravité, stationnaire, translation latérale. **Avant/arrière, haut/bas, tangage, lacet, roulis, vitesse, visée, leurres : rien** (presets Ship* sans axe). |
| Aéroglisseur | PARTIEL | Seulement translation LB/RB. |
| **Construction (QBuilder, 43 presets)** | NON | Aucune touche manette dans `Content/Inputs/Builder/`. |
| **Menus (inventaire, réglages, lobby, boutique, missions, création de perso)** | PARTIEL (?) | Navigation croix + A + LB/RB codée dans `W_SuperClassBase` et héritée par 73 widgets. À vérifier : focus initial à l'ouverture, **retour avec B** (UI_Return n'a pas de touche manette), glisser-déposer d'inventaire (souris). |
| Ouvrir le menu principal | OK (Start) | Carte, missions, social, émotes, chat, minimap, aide, traqueur : rien. |
| **StarMap, carte** | NON (?) | Un seul widget de la StarMap hérite de la navigation. Zoom, déplacement, sélection = souris. |
| Roues (gadgets, émotes), menu de construction, widgets 3D (`UI3D_*`) | NON (?) | Hors `W_SuperClassBase`. `UI3D_LeftClick` / `UI3D_ShowMouse` sans touche manette. |
| Curseur virtuel | NON | Pas de curseur analogique (`AnalogCursor` absent, CommonUI non configuré). Indispensable pour les écrans restés à la souris. |
| **Saisie de texte** | NON | Chat, mot de passe serveur, message admin, digicode, nom du personnage. `ShowGamepadTextInput` (SteamCore) jamais appelé : c'est la remarque de Valve sur le clavier virtuel. |
| Dialogues de quête | OK | Gérés en C++. |
| Notifications | OK (Start / Select ferment) | Les invites de quête choisissent la combo manette (index 1). |
| Émotes | NON | UI_OpenEmoteList sans touche manette. |
| **Glyphes affichés** | PARTIEL (?) | `W_InputFeedback` appelle `GetInputComboByIndex` (index utilisé inconnu sans ouvrir le graphe). Le texte bascule peut-être, mais il n'existe **aucune icône de bouton manette** dans le projet. Affiches de tutoriel : même widget. |
| Rebind manette | OK | Boutons et axes réassignables. |
| Vibrations | NON | Aucun `ForceFeedbackEffect` ni `PlayForceFeedback` dans le contenu ou le C++ du jeu. |
| Inversion, sensibilité | PARTIEL | `InvertHAxis`, `InvertVAxis`, `GamepadAimSpeed` existent ; zone morte à 0.005. |

Lecture d'ensemble : **les touches d'action sont en partie posées, mais les deux sticks ne pilotent rien par défaut et la gâchette de tir n'est pas assignée.** On ne peut donc ni marcher, ni regarder, ni tirer à la manette sans passer par les réglages. Ça colle avec la remarque de Valve (« la configuration manette par défaut ne marche pas entièrement ») : le jeu est jouable sur SteamOS grâce à la traduction clavier/souris de Steam Input, pas grâce au support natif.

## 5. Plan par lots (la base d'abord)

Chaque lot se termine par un test manette réel avant le suivant. Toute touche par défaut modifiée sur un preset déjà livré demande une **migration** de `InputConfig.sav` (patron `Migrate_BuilderMenuKey` de `Qanga_InputsComponent`), y compris pour les axes, qui sont aussi gravés dans la sauvegarde (MoveX et MoveY y sont).

**Lot 0. Filet Steam, sans toucher au jeu (Benjamin ou RzZz, Steamworks)**
- Publier une **configuration Steam Input officielle** pour l'appli 1648190 : stick gauche = ZQSD/WASD, stick droit = souris, RT = clic gauche, LT = clic droit, A = espace, X = interagir, Start = Tab, etc. Ça rend le jeu jouable sur Deck et avec toute manette **dès maintenant**, en attendant le natif.
- Steam Input « activé pour toutes les manettes » par défaut (couvre PlayStation et Switch sous Windows, vu `IgnoreGameControllers=True`).
- Ne pas encore déclarer « prise en charge complète » sur la page : on passera à « partielle » après le lot 2.

**Lot 1. Se déplacer, regarder, tirer, interagir (natif)**
- Axes par défaut : MoveX = stick gauche X, MoveY = stick gauche Y, CameraY/CameraZ = stick droit, avec une zone morte manette d'environ 0,15 à 0,2 (et non 0,005). Vérifier d'abord, graphe ouvert, comment `Qanga_InputsComponent` combine axe et touches, et le rôle exact de `GamepadAimSpeed`.
- RT = tir (`Combat_1stTrigger`), LT reste visée. Arbitrer `Combat_2ndTrigger` (même touche que la visée à la souris).
- Retour / fermer = B (`UI_Return`), et menu système accessible à la manette.
- Migration de la sauvegarde d'entrée.
- Décision demandée à Benjamin : le schéma de touches complet (§7).

**Lot 2. Menus**
- Passer écran par écran : focus initial, B pour revenir, LB/RB pour les onglets, inventaire (déplacer un objet sans glisser-déposer).
- Curseur virtuel au stick pour ce qui reste à la souris (StarMap, widgets 3D, menu de construction). Deux voies : activer le curseur analogique de CommonUI (déjà le viewport du jeu) ou un curseur maison piloté par `Input_GamepadThumbStickR`. Choix à faire après mesure.
- Clavier virtuel Steam (`ShowGamepadTextInput`) sur les 5 champs de texte, quand une manette est utilisée ou sur Deck.

**Lot 3. Afficher les bons boutons**
- Jeu d'icônes Xbox / PlayStation / Deck (textures maison, DA du projet), choisi par `CurrentControllerType` puis par type de manette.
- `W_InputFeedback` et les invites prennent la combo manette quand la manette est active (convention combo 1, comme les quêtes).
- Option : glyphes officiels via Steam Input (`GetGlyphForActionOrigin`), qui couvrent aussi Steam Controller et Deck.

**Lot 4. Véhicules, vaisseau, aéroglisseur, jetpack, construction**
- Voitures : RT accélérer, LT freiner/reculer, stick gauche diriger.
- Vaisseau et aéroglisseur : axes Ship*/Hover* sur les sticks et gâchettes.
- Jetpack : direction au stick, descente, stationnaire.
- QBuilder : 43 actions à répartir (modes, rotation, validation, annulation), probablement avec une roue radiale.
- Armes 1 à 4, soin, consommable, roues de gadgets et d'émotes : croix maintenue ou roue.

**Lot 5. Confort**
- Vibrations (tir, dégâts, atterrissage, moteur).
- Visée assistée légère (ralentissement près d'une cible), réglable et désactivable.
- Courbes de réponse et sensibilités séparées manette / souris.

**Lot 6. Steam, Deck, SteamOS**
- Steamworks : cocher la prise en charge manette (partielle puis complète), garder la config Steam Input par défaut à jour avec les touches natives.
- Demander une nouvelle revue Deck / SteamOS une fois les lots 1 à 3 faits. Le « non pris en charge » Deck vient des performances graphiques : chantier séparé (préréglage Deck), hors de ce plan.
- Steam Machine : pas de grille de test publique à ce jour ; viser le même niveau que Deck / SteamOS.

## 6. Ce que Benjamin doit essayer (manette Xbox branchée, build actuelle)

1. Lobby : naviguer avec la croix et A, changer d'onglet avec LB/RB, revenir avec B.
2. En jeu, sans rien régler : le stick gauche fait-il avancer ? le stick droit tourner la caméra ?
3. A saute, X interagit, LT vise, RT tire-t-il ?
4. Start ouvre l'inventaire : peut-on déplacer, équiper, jeter un objet ?
5. Réglages, touches : assigner MoveY au stick gauche, puis vérifier qu'on marche. Noter la sensation de zone morte.
6. Entrer dans une voiture, un vaisseau : que répond la manette ?
7. Ouvrir le chat : un clavier s'affiche-t-il (en mode Big Picture) ?
8. Refaire 2 et 3 avec Steam Input activé, puis désactivé (clic droit sur QANGA, Propriétés, Manette).
9. Si possible, une manette PlayStation sous Windows, Steam Input désactivé : est-elle vue ?

## 7. Décisions à prendre

1. Schéma de touches manette complet (je propose une grille au début du lot 1, à valider).
2. Lot 0 (config Steam Input officielle) tout de suite, avant la démo ?
3. Curseur virtuel : CommonUI ou maison ?
4. Icônes de boutons : jeu maison ou glyphes Steam ?
5. Visée assistée : oui ou non, et en multijoueur (équité clavier/manette) ?
