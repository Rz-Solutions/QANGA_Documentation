# Documentation QANGA — Index

Documentation d'architecture des plugins first-party de QANGA (UE5.7, plugins C++/UObject). Convention : prose en **français**, identifiants/code en **anglais**. Chaque fiche suit la même structure (but d'usage → architecture → carte des composants → flux de données → réplication → intégration → gotchas → fichiers).

> Les plugins tiers (DLSS/FSR/XeSS/Streamline, SteamCore, NinjaCharacter, JoystickPlugin, etc.) ne sont pas documentés ici — voir leur documentation amont.

---

## Fondations & framework

| Plugin | Doc | Résumé |
|---|---|---|
| QSystem | [QSystem_ARCHITECTURE](QSystem_ARCHITECTURE.md) | Bases de framework (Q* GameMode/State/PC) + « Systèmes » modulaires validés par type de monde |
| QGameManager | [QGameManager_ARCHITECTURE](QGameManager_ARCHITECTURE.md) | Gestionnaire de jeu / état global |
| QSave | [QSave_ARCHITECTURE](QSave_ARCHITECTURE.md) | Framework de sauvegarde (conteneur maître, compression Oodle, auto-save) |
| QScalability | [QScalability_ARCHITECTURE](QScalability_ARCHITECTURE.md) | Réglages/qualité data-driven + benchmark hardware |
| Q_DataBase | [Q_DataBase_ARCHITECTURE](Q_DataBase_ARCHITECTURE.md) | Registre de données en mémoire, requêtable (index de pré-filtrage) |
| QSQL_Interface | [QSQL_Interface_ARCHITECTURE](QSQL_Interface_ARCHITECTURE.md) | Couche SQLite clé-valeur (`int64 → blob`) |

## Réseau & état

| Plugin | Doc | Résumé |
|---|---|---|
| QNet | [QNet_ARCHITECTURE](QNet_ARCHITECTURE.md) | Réplication de mouvement optimisée (bit-packing, espace relatif) |
| QNetState | [QNetState_ARCHITECTURE](QNetState_ARCHITECTURE.md) | « Optimized State » : réplication d'état à la demande, indexée par localisation |

## Monde, espace & rendu

| Plugin | Doc | Résumé |
|---|---|---|
| WorldScape | [WorldScape_Plugin_Documentation](WorldScape_Plugin_Documentation.md) | Terrain planétaire procédural (async) |
| PlanetScape | [PlanetScape_ARCHITECTURE](PlanetScape_ARCHITECTURE.md) | Génération de planètes sphériques GPU |
| AtmoScape | [AtmoScape_ARCHITECTURE](AtmoScape_ARCHITECTURE.md) | Diffusion atmosphérique (Scratchapixel) |
| SpaceScape | [SpaceScape_ARCHITECTURE](SpaceScape_ARCHITECTURE.md) | Ciel étoilé / fond galactique (content-only) |
| GravityScape | [GravityScape_ARCHITECTURE](GravityScape_ARCHITECTURE.md) | Zones/direction de gravité |
| RzIndirectInstancingPlugin | [RzIndirectInstancingPlugin_ARCHITECTURE](RzIndirectInstancingPlugin_ARCHITECTURE.md) | Vertex factory d'instanciation indirecte |
| QSpatialGrid | [QSpatialGrid_ARCHITECTURE](QSpatialGrid_ARCHITECTURE.md) | Grille spatiale hiérarchique planétaire + persistance SQLite |
| QInstanced | [QInstanced_ARCHITECTURE](QInstanced_ARCHITECTURE.md) | ISM/HISM à création de corps physiques différée et budgétée |

## IA

| Plugin | Doc | Résumé |
|---|---|---|
| QAI | [QAI_ARCHITECTURE](QAI_ARCHITECTURE.md) | IA data-oriented batch (SoA/SIMD), vol/sol/espace, client-serveur |

## Gameplay

| Plugin | Doc | Résumé |
|---|---|---|
| QWeapon | [QWeapon_ARCHITECTURE](QWeapon_ARCHITECTURE.md) | Couche d'animation d'arme C++ (squelette ALS) |
| QVehicles | [QVehicles_ARCHITECTURE](QVehicles_ARCHITECTURE.md) | Véhicules + systèmes de sirène |
| FlyVehicleMovement | [FlyVehicleMovement_ARCHITECTURE](FlyVehicleMovement_ARCHITECTURE.md) | Mouvement de véhicule volant |
| QTrain | [QTrain_ARCHITECTURE](QTrain_ARCHITECTURE.md) | Trains (sièges, handoff, spline) |
| QBuilder | [QBuilder_ARCHITECTURE](QBuilder_ARCHITECTURE.md) | Système de construction/placement |
| QElevator | [QElevator_ARCHITECTURE](QElevator_ARCHITECTURE.md) | Ascenseur sur spline (état via QNetState) |
| QCableConnector | [QCableConnector_ARCHITECTURE](QCableConnector_ARCHITECTURE.md) | Câbles physiques (solveur PBD) + connecteurs/sockets |
| QTriggerZone | [QTriggerZone_ARCHITECTURE](QTriggerZone_ARCHITECTURE.md) | Zones de déclenchement (son/téléport/custom) |
| QPolice | [QPOLICE_ARCHITECTURE](QPOLICE_ARCHITECTURE.md) | Niveau de recherche & réponse policière |
| DynamicQuestSystem | [DQS_Architecture](DQS_Architecture.md) | Framework de quêtes (manuel + procédural) |

## Audio

| Plugin | Doc | Résumé |
|---|---|---|
| QAmbientAudio | [QAmbientAudio_ARCHITECTURE](QAmbientAudio_ARCHITECTURE.md) | Ambiance audio réseau (pile de priorités, MetaSound, horloge serveur) |
| QRadio | [QRADIO_ARCHITECTURE](QRADIO_ARCHITECTURE.md) · [Guide](QRADIO_GUIDE.md) | Stations radio locales par position |
| QMusic | [QMUSIC_ARCHITECTURE](QMUSIC_ARCHITECTURE.md) | Système de musique |

## UI & feedback

| Plugin | Doc | Résumé |
|---|---|---|
| QNotification | [QNotification_ARCHITECTURE](QNotification_ARCHITECTURE.md) | Notifications (toast/warning/progress/dialogue PNJ) |

## Outillage développement & build

| Plugin | Doc | Résumé |
|---|---|---|
| CodeScape | [CodeScape_ARCHITECTURE](CodeScape_ARCHITECTURE.md) | Outillage IA/prompt de création de jeu |
| RzDirectMCP | [RzDirectMCP_ARCHITECTURE](RzDirectMCP_ARCHITECTURE.md) | Bridge MCP headless (tools client/natifs) |
| RzBlueprintTools | [RzBlueprintTools_ARCHITECTURE](RzBlueprintTools_ARCHITECTURE.md) | Organisation de graphes BP + vérification projet |
| EasyCook | [EasyCook_ARCHITECTURE](EasyCook_ARCHITECTURE.md) | Correctifs de cook (refs cachées, shaders GPU Niagara) |
| QReport | [QReport_ARCHITECTURE](QReport_ARCHITECTURE.md) | Signalement en jeu (IGBR) — bundles offline-first |

## Guides transverses

| Doc | Résumé |
|---|---|
| [ModernCPP_UE57_RzZz_Guide](ModernCPP_UE57_RzZz_Guide.md) | Conventions C++ moderne UE5.7 |
