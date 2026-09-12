# Reprise migration Blueprint vers C++ — checkpoint du 2026-09-08

## Prompt à reprendre

Reprends la migration QANGA depuis ce checkpoint volontaire. Lis `AGENTS.md`, puis `Documentation/BlueprintToCppMigration/BLUEPRINT_TO_CPP_MIGRATION_PLAN.md`, `08_INVENTORY_MIGRATION.md` et les documents spécialisés des lanes réellement reprises. Réévalue leurs affirmations face à la source et aux assets courants : ce checkpoint est un point de reprise, pas une architecture immuable ni une clôture du plan global.

L'utilisateur a demandé un arrêt sain, le staging des seuls fichiers concernés et aucun commit. Ce document a été créé après ce staging. Ne relance pas automatiquement le chantier à sa lecture sans demande de reprise. Préserve les changements des autres sessions ; ne commit pas, ne cook/stage/package pas le jeu. Le staging Git n'autorise pas le packaging Unreal.

Sol garde intégration, compilation, tests et contrôle de l'éditeur. Délègue les implémentations non triviales à des builders natifs avec ownership disjoint ; ils ne lancent aucun test ni commande éditeur. Aucun override de modèle/provider/effort. Les opérations d'assets passent par RzDirectMCP ; ne lis pas les `.uasset` comme du texte. Si une modification nécessite un restart, relance avec `-AutoDeclinePackageRecovery`. L'ouverture des maps seule ne justifie pas leur sauvegarde.

## État réellement livré

- QInventory core : records/codec v2, identité DataObject distincte, payload typé non possédé conservé, validation sémantique commune, préflight sans mutation, réindexation du sac compact, équipement séparé et rollback exact. Le journal v2 accepte un backend de récupération nommé et des payloads opaques avant/après ; pruning des journals récupérés implémenté.
- `QInventoryIntegration` contient uniquement le module et `QInventoryComponent.h/.cpp`. L'adapter est complet en source mais dormant : le Blueprint Inventory reste parenté à ActorComponent, aucun reparent ni bootstrap n'a été effectué. Les quatre RPC et les huit appels UI restent sur le chemin existant.
- Il n'existe PAS de propriétaire de persistance Inventory de production. Les fichiers incomplets `QInventoryPersistence.h/.cpp` ont été supprimés avant le checkpoint. N'en déduis pas qu'un subsystem ou un prepare/publish est déjà branché.
- Le callback post-commit de l'adapter dormant invoque encore les anciens `ReplicateSaveInventory` source/cible. Ne l'active pas pour les transferts persistants tant qu'une journalisation paire complète ne remplace pas cette production de saves sur le seul chemin géré.
- Synchronisation legacy : InventoryUpdate et EquipmentUpdate programment une seule tâche GT différée, car EquipmentUpdate est diffusé avant UpdateInventorySize. Le teardown neutralise la tâche via capture faible. Les inventaires appartenant à AQBuilder sont explicitement exclus de l'initialisation et de la synchronisation natives.
- DataManager possède un writer FIFO par slot pour la paire principal/secours, partagé entre sauvegarde asynchrone et bridge durable, avec bornes, readback, rollback et drain. L'attente durable utilise un état partagé avec `FEventRef`, pas `TTask::GetResult()` : ce dernier peut exécuter la tâche sur le GT malgré `DoNotRunInsideBusyWait`. La tâche du pipe reste l'unique exécutant I/O.
- TempDB appelle une seule action native de sauvegarde de paire à la place des deux saves indépendants. Son résultat agrégé pilote la suite existante et le retry existant. DataObject BlockingThreadEncode utilise désormais le séparateur de tableau `,¶`, identique au décodeur et à l'encodeur asynchrone.
- QWeapon délègue l'autorisation des dégâts au provider natif Combat avec une évaluation fraîche, sans reproduire faction/safe areas ni réutiliser une décision de ciblage en cache. Les destructibles sans provider Combat conservent le dispatch Unreal de dégâts.
- QSystem expose la revalidation serveur de la cible exacte d'accès via sa requête existante, sans les raccourcis d'entrée/sortie véhicule réservés à l'action d'interaction.
- RzDirectMCP : qualification des membres contre le shadowing des paramètres locaux ; inspection réseau étendue aux composants/UObjects et paramètres RPC ; vue des graphes authored imbriqués sans compter les stubs compilateur ; connexion des pins résolue par nom ET direction pour les pins input/output homonymes.

## Frontière de vérification

- Build froid QangaEditor réussi le 2026-09-08, puis Live Coding réussi pour les deux dernières modifications cpp (attente DataManager et assertions journal v2). L'éditeur a été laissé ouvert, sans PIE. Pour le prochain restart, reconstruire si nécessaire la base avec l'éditeur fermé.
- Suites finales : `QATS.QInventory.*` 36/36, `QATS.QStorage.*` 11/11, `QATS.DataManager.*` 7/7 ; `FifoAcrossAsyncAndDurable` répété 20/20 sans warning. Ciblage QWeapon 7/7, Bullet.CombatPolicyConsumer 1/1, Combat 23/23, interaction 9/9, RzDirectMCP.Codegen.SelfMemberShadowing 1/1.
- Le cas Combat de configuration invalide émet un warning attendu en vérifiant le refus ; aucun échec. Les autres suites citées n'ont aucun warning.
- TempDB et DataObject compilés/sauvegardés sans erreur ni warning. Fixture transient DataObject : la ligne `12☺SlotAttachments☺Optic♦Item_A,¶Barrel♦Item_B` survit au blocking encode/decode avec exactement les deux attachments.
- Rien de cela ne prouve le transfert de l'adapter dormant en gameplay, les modes réseau, le restart après journal interrompu, une coupure électrique ou le packaging. Les anciens résultats Offline Tutorial ne sont pas un nouveau run de ce checkpoint.

## Première reprise : une transaction TempDB complète

1. Créer UN owner serveur par monde/WorldId dans l'intégration, seul propriétaire du coordinateur durable existant. Vérifier headers et ownership avant toute déclaration réfléchie ; ne laisser aucune API déclarée sans définition liée.
2. Faire le préflight complet sans effets de bord : autorité, accès, endpoints, versions, capacité, quantité, identités, payload, mode de persistance et faisabilité backend. L'admission des GUID doit être planifiée sans write ; aucune admission durable indépendante avant la faisabilité finale.
3. Résoudre les DataObjects propriétaires exacts et capturer leurs lignes TempDB par `DataObject.GetDataObjectId`, DISTINCT du `InventoryId` legacy. Capturer avant/après complets incluant les GUID planifiés, records et payloads opaques de récupération.
4. Écrire durablement Prepared AVANT admission GUID ou mutation live. Réutiliser le coordinateur/journal et le writer paire existants ; pas de second journal, pas de saves concurrentes indépendantes.
5. Appliquer admission et mutation, vérifier les postconditions, réconcilier caches/versions/générations, puis persister les deux lignes en un batch exact. Garder les callbacks Blueprint arbitraires hors des locks d'endpoints.
6. Committed seulement après persistance paire réussie. Publier ensuite les notifications ; supprimer la production de sauvegardes legacy doublonnée sur ce chemin géré, en préservant ses autres consommateurs. Pruner après commit ; conserver un journal committé si le pruning échoue afin de permettre la récupération idempotente.
7. À chaque échec pré-commit, restaurer exactement les deux participants live et leurs préimages durables. Conserver le journal si la restauration exacte n'est pas prouvée. Ne pas masquer une erreur par un backend alternatif.
8. Commencer si utile par persistent→persistent, mais le résultat final doit préserver les modes persistent↔transient et transient↔transient requis. Une première tranche qui refuse explicitement les modes encore non implémentés n'est pas une migration terminée.

### Recovery avant publication du monde

- Point exact relevé dans `/Game/Systems/Data/TempDB/TempDB.EventGraph` : `K2Node_IfThenElse_35.else` → `K2Node_VariableSet_2` (`IsWorldReady=true`) → GameDataManager → `LoadWorldResult(true)`.
- Insérer la récupération AVANT ce Set. WorldID et TempDatabaseOBJ sont déjà définis, LoadedWorldData déjà décodé, et les requêtes non-world attendent le ready.
- Utiliser `ResolveLoadedTempDbRecoveryContext`, dont le contrat exige `IsWorldReady=false`. Scanner dans l'ordre chronologique. Prepared/Aborted restaurent les deux préimages ; Committed applique les deux postimages. Un seul batch `WriteExactRecoveryRowsToContext`, puis `PruneRecoveredJournal` seulement après réussite durable des deux lignes.
- En cas d'échec : conserver le journal, garder ready=false et passer par le broadcast/clear existant de `LoadWorldResult(false)`. Erreur debuggable et bornée, sans readiness partielle.

## Bascule des callers génériques après ces gates

- Revalider sémantiquement les assets avant de modifier : l'audit précédent comptait 115 Blueprints référents et 8 maps ; huit appels UI ciblés ont été relevés dans quatre widgets.
- `Widgets/Inventory/W_Character.EventGraph` : CallFunction_18 vers inventaire, CallFunction_22 vers vault ; `Widgets/Inventory/W_Inventory.OnDrop` : mêmes deux suffixes.
- `Widgets/Storage/PUW_Storage` : InventoryToVault et VaultToInventory, CallFunction_6 dans chacun ; `PUW_VehicleStorage` : CallFunction_6 au dépôt, CallFunction_4 au retrait.
- Pour le véhicule, utiliser le PawnInventory possédé comme SELF et ses versions ; ne pas substituer l'inventaire PlayerState.
- Deux fins d'initialisation à revalider dans `Inventory.InitEquipment`, VariableSet_2 et VariableSet_1, posent l'état true avec le then actuellement libre : raccorder la readiness native après reparent lorsque son contrat est prêt.
- Les items ont pour Outer le composant ItemsManagerGS/GameState. Ils ne sont pas children de l'inventaire : ne pas ajouter de reouter sans besoin démontré. Conserver les mêmes objets item/attachments pour les transferts génériques.
- Équipement : la racine équipée conserve AttachedToSlot=clé d'équipement et AttachedToId=None. Les champs opaques sont identifiés par le couple exact `(type,key)`, jamais par key seul. Owner=None est légitime en transient. Capacité gameplay 100000 et borne de racines sérialisées 4096 sont distinctes.
- Une fois les routes réellement validées, retirer les anciens producteurs remove/recreate, RPC et writes remplacés. Désactiver un consommateur ne suffit pas à retirer son producteur.

## QBuilder : frontière distincte, aucun changement livré

Ne traite pas son inventory comme le deuxième endpoint générique. `Builder_Advanced_Resource` est le ledger autoritaire ; le sac est une projection jetable, reconstruite avec de nouvelles instances/identités par InteractServer. Le ledger est sauvegardé dans la famille world `*_Build_0..3.sav`, pas TempDB. Aucune source QBuilder n'a été modifiée dans ce checkpoint.

- Dépôt : consommer le record complet sélectionné d'un vrai inventaire et incrémenter le ledger. Retrait : décrémenter le ledger et créer une nouvelle identité d'item joueur ; ne pas exporter l'identité de projection.
- Il faut une transaction typée inventory + ledger, avec versions attendues, préimages/postimages, validation mapping/quantité/capacité/overflow, rollback et publication différée. Observer TOUS les writers ledger : dépôt/retrait, coût construction, remboursement, réparation et restore.
- Quest/deposit events et refresh de projection seulement après commit durable. Conserver le chemin existant jusqu'à remplacement fonctionnel, puis retirer la projection persistée et sa production de données devenue inutile.
- Mapping authored actuellement hardcodé par trois nodes sur `/Game/Systems/QBuilder/Ressource/QBuilder_Res_DB.ItemTable` ; ne pas inventer une propriété sur le builder. L'UI déplace toute la stack sélectionnée, sans sélecteur de quantité.
- Le manager lie Init_BindAutoSave à QBuilder_Manager_Auto_Save puis Save_Game.Data et Save_Save_Game (AsyncSaveToFile) ; EndPlay utilise SyncSaveToFile, load AsyncLoadFromFile. Le nom provient de Compute_Level_Name(WorldId)+`_Build`, avec rotation et AutoSaveOnDestroy=false.
- QBuilder_BuilderID est réassigné au load : ce n'est pas une clé durable. La suggestion d'utiliser directement le GUID QStorage n'a PAS été validée ; le chemin QStorage par défaut possède encore une branche legacy ordinale et ses containers ne possèdent pas ce ledger. Établir une identité durable réelle avant recovery, sans fallback ordinal ni faux endpoint.
- L'actuel BeginPlay obtient un ID et appelle PDC.SetIdAndGetData pour la projection (OnDataReady règle notamment sa taille à 10). Retirer ce producteur quand le nouveau chemin le rend obsolète, pas seulement sa consommation.
- RequestBuilderUpdate → SendUpdatedResourcesToQBuilder(addc++) mène à une fonction vide : pas de preuve d'un MapAdd corrompu. Supprimer cette chaîne morte lors de la bascule, sans diagnostic inventé.

## Gates puis suite du plan

Après implémentation : build/UHT selon l'état de l'éditeur, suites natives ciblées, puis autorisation utilisateur pour les tests runtime nécessaires. Couvrir rejet sans effet de bord, chaque échec prepare/write/commit/prune, rollback bag/equipment/attachments/opaque payload exact, callbacks réentrants, convergence des versions, recovery avant ready, shutdown, restart et concurrence de deux clients. Pour QBuilder ajouter tous les writers du ledger, mapping invalide, overflow/underflow, retrait cible pleine et identité après reload. Ne pas conclure à la durabilité physique sur la seule base de mocks.

La reprise globale ne s'arrête pas à Inventory : continuer ensuite les lanes Weapon de référence, convergence des mutations Item/QModule/quests, consumers Combat, les gates AILean/gravité, caméra/swimming et les P2 décrits dans le plan. Ne pas remigrer des owners déjà validés ni élargir silencieusement le périmètre. Les gates network/dedicated/packaged/perceptuels restants doivent rester explicites ; le user réalise cook/package.

## Périmètre Git confirmé par l'utilisateur

L'utilisateur a confirmé que les modifications QAI_Faction.cpp/.h et les fichiers JS bridge/execution-metadata/execution-policy de RzDirectMCP appartiennent également à ce travail et doivent être stagés. QAI verrouille l'alignement des valeurs de factions sérialisées avec QCombat. Le bridge expose les métadonnées d'effets d'exécution (lecture, écriture, validation, contrôle éditeur), avec règles selon les arguments et refus explicite si la politique est illisible/invalide.

L'utilisateur a ensuite demandé d'inclure aussi son déplacement manuel de `Documentation/Todo_prompts/PLANETSCAPE_REPAIR_TODO_PROMPT.md` vers `Documentation/TodoPrompts/PLANETSCAPE_REPAIR_TODO_PROMPT.md`. Tout le travail présent au checkpoint est donc stagé, sans commit. `Documentation/RzTodoPrompts/QATS_Continuation_2026-09-08.md` existait déjà et n'a pas été modifié. Vérifier le nouvel état Git avant toute reprise ; ces repères ne constituent pas une autorisation d'écrasement.
