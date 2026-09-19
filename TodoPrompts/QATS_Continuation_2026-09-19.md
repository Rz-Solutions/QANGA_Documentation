# QATS — checkpoint de reprise

## Objectif et limites
- Objectif mis en pause à la demande de l'utilisateur : terminer via les vraies actions QATS toutes les quêtes du tutoriel et de l'univers, sans forcer leur complétion. Aucune quête complète validée à ce stade.
- Périmètre : Q004–Q011, Q015, Q017–Q028 (21 quêtes). Exclure Q001–Q003, Q012–Q014 et Q016 (développement).
- Respecter AGENTS.md : plugins projet uniquement, aucune modification moteur, aucun cook/package ni lancement de jeu compilé. PIE arrêté avant toute compilation. Ne pas committer sans demande.
- Ancien checkpoint du 8 septembre lu puis supprimé sur demande. Changements préparés pour staging dans QANGA et Documentation, sans commit.

## État arrêté
- Éditeur associé : G:/QANGA/QANGA.uproject, UE E:/UE573. Dernier PID observé : 28132 (revérifier).
- Au checkpoint, MCP confirme PIE/SIE arrêté. Le dernier artefact indique une interruption par désinitialisation, pas une réussite ni une restauration complète démontrée.
- Dernier parcours : Saved/QuestTests/PIE/20260919_033439_28132_705F5513457B170E5AB8A585A39BB12A.
- Dernier état observé : Q006, O_UseRepairBase actif, après récupération du module et de la matière. QATS a reconnu l'onglet équipement visible, mais aucun message de recyclage terminé ni d'entrée réparation n'a suivi. Le pawn était immobile. Ne pas conclure à un défaut du module : vérifier d'abord le drag/drop de recyclage et la fermeture du menu.
- Toutes les modifications C++ du checkpoint ont compilé avec Build.bat -LiveCoding, puis build éditeur complet après arrêt requis pour changement de layout du runner. L'éditeur redémarré exécutait ces changements.

## Corrections présentes
- QATS : override de throttling à propriété identifiée, conservation des arguments de lancement et remplacement unique du mode hovercraft, commande quest.test.stop avec nettoyage asynchrone.
- Fixture inventaire : admission/restauration native atomique ; ne capturer/restaurer que les deux slots d'armes et supprimer les seuls items générés. Préserver les équipements de quête et leurs déblocages, notamment le drone remplacé par le tutoriel.
- InventoryComponent Blueprint : AddToEquipmentInventory passe par QINV_EquipOrAdmitItem ; plus de mutation partielle des maps d'équipement. Null rejeté, succès natif retourné.
- Lecture MCP des quêtes sans réécriture des string tables ; sauvegardes conservant leur synchronisation habituelle.
- Q005 : prérequis Scan corrigé en Scanner. Q006 : doublon strict Location_Hangar_Jetpack supprimé, 65 objectifs. Export RuntimeQuests.bin actualisé. Le reste du diff JSON est de la sérialisation de valeurs par défaut vérifiées ; pas d'édition de localisation conservée.
- Audio de quête : composant enregistré avec son monde explicite.
- Objectifs de position : snapshot des clés et copie des informations avant callbacks pouvant modifier les registrations, plus de références/itérateurs invalidés par la progression synchrone.
- Transitions QATS : relâchement des entrées/navigation sans objectif actif ; délai d'absence d'objectif suspendu pendant les notifications réellement bloquantes, avec délai d'attente borné séparé. Cela a permis O_GoinVent_2 -> O_RepairRoomDoor : ancien timeout 10 s courait pendant un dialogue de ~9 s suivi du contrôle périodique des prérequis toutes les 2 s. Le parcours a ensuite passé les deux portes et les notifications de réparation/recyclage.
- Câbles : l'acteur socket possède l'éligibilité et le dispatch serveur ; CustomTarget vise l'acteur, pas sa sphère. Proxy de trace natif remplaçant le BP générique sérialisé ; mutation côté serveur uniquement, connecteur tenu requis. Compilation faite, connexion physique encore à rejouer après les étapes précédentes.

## Reprise prioritaire
1. Inspecter TickRecyclePrerequisiteItem et TickCloseGameplayMenu dans QuestTestSubsystem.cpp autour de 22400–22700. Dernier log : équipement visible avant recyclage. Éviter tout équipement/complétion forcé pour faire passer cette étape.
2. Rejouer Q006 depuis le début en PIE standalone, une seule instance. Commande : quest.test.start Q006 1 1337 Heuristic. Map /Game/Maps/DevMap/ConstructionLevel/L_Persistent_Tutorial. L'intro utilise une vraie entrée Space ; le focus OS peut retarder son acceptation au premier PIE après redémarrage.
3. Vérifier physiquement les deux connexions câble (IsHeld false, ConnectedSocket correct, sockets connectés), puis continuer le tutoriel. L'ancien parcours partiel atteignait les câbles sans les connecter malgré un progrès affiché ; ne pas prendre le compteur seul pour preuve.
4. Plus loin, QModulePhaseObjective n'a pas d'exécuteur dans ResolveExecutor. Sa classe est définie dans Plugins/QModule/Source/QModule/Private/QModule_QuestObjectives.h ; ne pas inclure un header privé depuis QATS. L'installation et l'insertion de phase doivent emprunter les vraies interactions UI/rack. Aucun changement fait pour ce point.
5. La fin tutorial ouvre Universe via OpenPersistentUniverseAction et capture le handoff offline ; respecter et vérifier cette transition, pas seulement un compteur de quête.

## Autres blocages connus, non corrigés
- Q004 : QAI hovercraft, guide 16,8 km, cellules ~140 m. Cellule requester rejetée NoHeadroom, projection de sol ~352 cm sous le pawn ; ne pas remplacer par une cellule voisine arbitraire. Artefact 20260919_022014_3516_C51CE3DA4533DD9725C911A09681374D.
- Q028 : O_Retreat_Cyborg utilise CompleteOnExit. QATS suppose encore Progress % Locations et transforme inside en Passive. Utiliser le contrat existant ULocationObjective::GetCurrentLocationRequirement pour l'entrée/sortie et les objectifs multiples, y compris les consommateurs de destination longue distance.
- Q019/Q022 : récompense liée à O_Reward absent ; Q022 porte aussi un QuestID Q019 copié. Ne pas simplement lier au dialogue final : actions déclenchées à l'activation, délai récompense 5 s pouvant être annulé par la fin immédiate.
- Prérequis de portée : Q005 dépend Q006 ; Q017 dépend Q008 ; Q026 dépend Q008/Q017 ; Q027 dépend Q008 ; Q028 dépend Q026. Une fixture forçant un prérequis n'est pas une preuve de parcours complet.

## Outils et preuves
- Logs Saved/Logs/Qanga.log ; résultats Saved/QuestTests/PIE/<run>/quest_test.json et .jsonl.
- MCP execute_python_script avec allow_during_pie=true pour lecture seule. Résoudre pc via GameplayStatics.get_player_controller(world,0), pawn via GameplayStatics.get_player_pawn(world,0), composant PlayerQuestComponent.
- c.get_quest_instance_data('Q006') retourne un struct avec export_text() pour ActiveObjectiveIDs (propriété protégée autrement) et objective_progress_array. Les UObject objectifs n'ont pas export_text().
- c.get_active_objective_instance_by_id peut retourner une instance inactive ; toujours lire ActiveObjectiveIDs pour déterminer l'étape réelle.
- Avant arrêter PIE, demander quest.test.stop checkpoint et attendre l'artefact de nettoyage quand la session tourne encore. Au présent checkpoint elle était déjà arrêtée.
- Pas de staging large ni commit. Vérifier les chemins explicites et git diff --cached --check dans les deux repos.
