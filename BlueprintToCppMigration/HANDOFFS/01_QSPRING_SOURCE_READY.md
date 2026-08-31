STATE: SOURCE_READY
OWNED FILES CHANGED:
- `Plugins/QSystem/Source/QSystem/Public/Component/QSpringArmComponent.h`
- `Plugins/QSystem/Source/QSystem/Private/Component/QSpringArmComponent.cpp`
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QSpringArmAutomationTests.cpp`
- `Documentation/BlueprintToCppMigration/05_SPRING_ARM_MIGRATION.md`
- `Documentation/BlueprintToCppMigration/HANDOFFS/01_QSPRING_SOURCE_READY.md`
EVIDENCE:
- Inspection live read-only confirmée avant fermeture de l'Editor : `/Game/Systems/QSpringArm/QSpringArm_Component` reste un `SceneComponent` de `41` variables, `9` fonctions et `340` nœuds ; `Update` possède `117` nœuds ; `Whiskers_Update` produit `8 + 4x4` sweeps de moustaches et `3` pre-probes ; `Out_Location` garde exactement `FVector Location`, `bool ToNear`, `double NearAlpha`, `FRotator CameraRotation`.
- Le chemin live distingue `Process_Z_Compensator`, qui tourne l'épaule avec le yaw local pawn, de `Whiskers_Update`, dont les pre-probes et la cible latérale consomment le right vector monde du composant. L'île pawn-local de `Whiskers_Update` et sa sortie Select sont déconnectées ; le natif ne les transcrit plus.
- Le natif possède seul activation et tick : bootstrap unique au `BeginPlay`, intent actif conservé pendant une unpossession, tick uniquement avec un `APlayerController` local, désactivation immédiate pour AI, joueur non local, pawn non possédé et dedicated server.
- La deadline reste `CurrentWorldTime + 0.2 s` après chaque exécution : aucun timer et aucun rattrapage en rafale. Les quirks top authored restent inchangés : accumulation avec `-StartDir`, puis division de l'alpha par le count simple.
- Les traces utilisent directement `TraceTypeQuery1/2`, les acteurs authored plus le pawn owner, `bTraceComplex=false`, les rayons authored et le premier résultat du probe lorsque le facteur vaut `1`. Aucun debug draw, fallback trace, log par frame ou scheduler supplémentaire.
- Revue des headers UE réels effectuée pour `ECollisionChannel`, `FCollisionQueryParams`, `FCollisionShape`, `FHitResult`, les signatures `UWorld` const, `ReceiveControllerChangedDelegate`, `FMath::FInterpTo/VInterpTo` et les casts vers les API `float`. Les entrées castées sont maintenant validées comme représentables avant activation.
- QATS source déterministe étendu : inactif avant `BeginPlay`, unpossessed, joueur non local, joueur local, AI local-authority, repossession, désactivation authored persistante, counts nuls rejetés et première frame exacte depuis l'état zéro. Le test n'a pas été exécuté dans cette lane.
- `git diff --check` est propre sur les fichiers owned. Aucun build, UHT, QATS, commande Editor, PIE, mutation asset, staging ou commit n'a été exécuté.
INTEGRATION REQUESTS:
- Reparent `/Game/Systems/QSpringArm/QSpringArm_Component` vers `/Script/QSystem.QSpringArmComponent`, supprimer ses `9` fonctions, ses variables/états remplacés et ses graphes après reconstruction des consommateurs, sans sauvegarder de comportement debug ou de calcul déconnecté.
- Dans `/Game/Systems/Character/Blueprints/CharacterLogic/ALS_Base_CharacterBP`, reclasser le nœud SCS `QSpringArm_Component` vers la classe native en conservant sa relative location `Z=187.39`, sa relative yaw `90`, `Length=210`, `CameraOffset=62`, `Left_Right=true`, `Interp_RLocation_Speed=0`, `Whiskers_H_Count=8`, `Whiskers_T_Count=4` et les flags de tick natifs.
- Retaper les accès et le binding `Out_Location` dans `ALS_Base_CharacterBP` et `/Game/Items/Drone/IS_DroneBase`; retirer l'appel legacy `Disabled` dans `/Game/AI/Cyborg/AI_Cyborg`; compiler aussi `/Game/AI/Lean/ALS_Base_CharacterBP_AILean` et les descendants concernés.
- Mettre `Documentation/BlueprintToCppMigration/BLUEPRINT_TO_CPP_MIGRATION_PLAN.md` à `source QSpringArm prête statiquement, compile/intégration en attente`, puis ne fermer ses gates qu'avec les preuves ci-dessous. Aucun changement `Build.cs`, QCamera, QAI ou config n'est requis par la source actuelle.
- Après zéro référence, retirer la seed EasyCook exacte sans rescan, puis supprimer le wrapper et `Cy_ASpringArm` uniquement si l'audit C++/assets confirme zéro consommateur.
- RzDirectMCP à corriger hors ownership de cette lane : `find_in_blueprints` dans `Plugins/RzDirectMCP/Source/RzDirectMCP/Private/ExtendedEditorLibrary.cpp` ne vérifie sa deadline qu'entre les assets ; `ConvertJsonStringToObject` vers la ligne `33970` peut bloquer au-delà de `time_budget_seconds` et monopoliser l'Editor. Ajouter une limite/cancellation à cette conversion avant de relancer une recherche globale.
TESTS FOR INTEGRATOR:
- Editor fermé : `"E:\UE573\Engine\Build\BatchFiles\Build.bat" QangaEditor Win64 Development -Project="G:\QANGA\QANGA.uproject" -WaitMutex`.
- Exécuter exactement `QATS.QSystem.SpringArm.PossessionLifecycle` et confirmer toutes les assertions, avec une seule erreur attendue `has invalid configuration` pour le composant aux counts nuls.
- Compiler `QSystem` dans la target non-unity Linux-Clang : `"E:\UE573\Engine\Build\BatchFiles\Build.bat" Qanga Linux Shipping -Project="G:\QANGA\QANGA.uproject" -WaitMutex`. Le QATS reste couvert par la target Editor Win64.
- Relancer l'Editor avec `-AutoDeclinePackageRecovery`, reparent/reclasser les assets ci-dessus, puis les compiler à `0` erreur et `0` warning ; vérifier que les bindings reconstruits exposent encore le dispatcher à quatre paramètres exacts.
- PIE `L_Dev_Rz` : repos, mouvement, crouch, changement d'épaule, collision mur/plafond, sortie d'overlap, scope, transitions first/third, sortie véhicule et repossession ; vérifier l'absence de pop et la continuité caméra.
- PIE réseau et dedicated : prouver zéro tick/output sur simulated proxy, joueur distant, pawn non possédé, AI et dedicated server, avec un seul owner caméra local.
- Cold-load puis audit de références : préserver les overrides ALS, confirmer le binding drone, `stop_pie` à `0` erreur / `0` warning, puis obtenir zéro référence wrapper/plugin avant toute suppression.
REMAINING:
- Tous les gates compile, UHT, QATS, asset, cold-load, référence, réseau et runtime restent ouverts ; cette lane ne revendique que la fermeture statique du source.
- L'Editor peut rester fermé jusqu'au build et à l'intégration asset de l'intégrateur exclusif.
