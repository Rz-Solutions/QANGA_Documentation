# Écran de transfert (coffre, véhicule, builder)

État au 2026-08-28. Ce document sert de passation : ce qui a été fait, ce qu'il reste, et comment
le faire sans rien casser. Tout ce qui est écrit ici a été mesuré sur le projet, pas déduit.
Quand une affirmation est une hypothèse, c'est dit explicitement.

---

## 1. De quoi on parle

Asset : `/Game/Widget/Storage/PUW_VehicleStorage` (WidgetBlueprint, parent C++ `UPopUpDialogBase`,
plugin `Plugins/WidgetBase`).

**Le nom est historique et trompeur.** Ce n'est pas un écran de véhicule, c'est l'écran générique
"inventaire d'un objet du monde". Il a **trois référents de production** :

| Référent | Chemin |
|---|---|
| Véhicules | `/Game/Systems/Vehicle/VehicleBase` |
| Builder (dépôt de minerais) | `/Game/Systems/QBuilder/Actor/QBuilder_Builder_Actor_BP` |
| Coffre local (+ `_Large` par héritage) | `/Game/Systems/QBuilder/Data/ICLAB/Actor/Storage/BP_QStorage_LocalChest` |

Un quatrième référent existe, `/Game/EasyCook/DA_EasyCookSeed_QANGA`, mais c'est le registre de
cook, pas un appelant.

**Conséquence pratique : toute modification de cet écran touche les trois d'un coup.** Il n'y a pas
de variante par usage aujourd'hui, et c'est un choix assumé.

---

## 2. Ce qui a été livré

### Refonte visuelle (27/08, validée dans le Designer le 28/08)

Le `SizeBox_0` passe de 1000x800 à **1050x908**. Le calcul est exact : +32 de padding, +18 de
gouttière, +107 de chrome, donc la zone des deux inventaires garde très exactement la taille
qu'elle avait. Nouvelle pile :

```
SizeBox_0
  Overlay_Frame
    Border_Surface          #0F141C a 0.90, padding 16/14/16/12
      VerticalBox_Root
        HB_Header           sur-titre + titre + ECHAP + W_Button
        HorizontalBox_79    les 2 W_Inventory, INCHANGE (gouttiere Spacer_69 : 10 -> 28)
        Image_Rule_Footer
        HB_Footer           ligne d aide
    CanvasPanel_Frame       4 filets #3F6E96 a 0.34 + 8 crochets #D9942F 13x2
```

Points importants pour qui reprendra le fichier :

- **Rien n'a été recréé.** `W_Button`, `W_Inventory_Storage` et `W_Inventory_Character` ont été
  **déplacés** (`remove_child` / `add_child`), jamais reconstruits. Les liaisons Blueprint se font
  par **nom** : recréer l'un de ces widgets casserait silencieusement les trois graphes `BndEvt__*`.
- Les libellés joueur passent par une String Table dédiée,
  `/Game/_QData/Localization/StorageLocalizationTable` (5 clés `storage.*`). Elles sont vérifiées
  `text_is_from_string_table == True`, donc réellement traduisibles.
- Un `Text_Title` a été ajouté dans l'en-tête, alimenté par un `SetText` chaîné **après** le
  `W_Inventory.SetText` existant, sur la même sortie de `FormatText`.

### Rafraîchissement après un dépôt (28/08, confirmé en jeu)

Symptôme corrigé : on déposait un objet dans le coffre, il n'apparaissait pas, il fallait fermer
et rouvrir.

Dans l'`EventGraph` de `PUW_VehicleStorage` :

```
Inventory to Vault .then  ─┐
                           ├─> Delay 0.35 s
Vault to Inventory .then  ─┘      -> SetActorInventoryRef( W_Inventory_Storage,   VehicleInventory )
                                  -> SetActorInventoryRef( W_Inventory_Character, PawnInventory )
```

C'est volontairement un **second passage**, pas une correction de la cause. Ça refait ce que fait
une ouverture d'écran, une seule fois par dépôt, dans les deux sens, sans boucle possible.
Le délai est réglable : c'est le pin `Duration` du nœud `Delay` (`K2Node_CallFunction_19`).

---

## 3. Contrats à ne jamais casser

1. **`SetVehicleInventoryRef`** (interface `BPI_VehicleStorageWidget`) : les trois référents
   l'appellent. Ni le nom ni la signature ne doivent bouger.
2. **`Close` + `CancelDialogPopUp`** : le bouton appelle `Call Close` **puis** `CancelDialogPopUp`,
   dans cet ordre. Le parent C++ diffuse ensuite `PopUpDialogCancelled`, et **le couvercle animé du
   coffre est branché sur cette broche**. Changer l'ordre ou la classe de widget laisse le couvercle
   ouvert.
3. **Les noms de widgets** `W_Button`, `W_Inventory_Storage`, `W_Inventory_Character` : les
   `ComponentBoundEvent` lient par nom, sans erreur de compilation si ça casse.
4. **Code mort volontaire** : `Event Construct` et la chaîne `GetInventoryComponent -> IsValid ->
   Set VehicleInventory` qui la suit ne sont reliés à rien. C'est l'état d'origine. Ne pas
   "réparer", ne pas supprimer sans décision.

---

## 4. Ce qu'il reste à faire

### 4.1 La cause racine du rafraîchissement (chantier séparé, composant partagé)

**Où** : `/Game/Systems/Item/InventoryComponent`, `EventGraph`.

**Ce qui est mesuré.** Côté client, `OnRep_InventoryItemsReplicated` appelle `RefreshInventory`,
qui reconstruit la liste en bouclant sur `InventoryItemsReplicated` :

```
ForEach ( InventoryItemsReplicated )
  -> Is Valid( instance )
       Is Valid     : suite
       Is Not Valid : RIEN            <- broche non cablee
  -> Add Replicated Item Instance Client Only
  -> Bind Event to On Item Update
  -> Branch( Is Item Data Ready )
       then : Add Item to Inventory
       else : RIEN                    <- broche non cablee
```

Un objet que le client ne sait pas encore résoudre est donc **jeté en silence**. Et
`Lib_Inventory.TryMoveItemToAnotherInventory` porte ce commentaire d'origine, qui explique
pourquoi ça arrive pile au moment d'un transfert :

> recreate the item instance by id because it got destroyed from the previous inventory replication

**Ce qui n'est PAS prouvé** : laquelle des deux broches avale réellement l'objet. Je ne l'ai pas
observé, je l'ai déduit.

**Donc, dans l'ordre :**

1. **Instrumenter d'abord.** Brancher un `Cy Print String` sur chacune des deux broches mortes,
   faire un dépôt en PIE, lire `Saved/Logs/QANGA.log`. C'est un ajout pur, retirable, sans
   changement de signature. Sans cette mesure, on corrige à l'aveugle.
2. **Corriger ensuite.** Le patron correct existe déjà dans le composant :
   `Request Items Instances` sur l'`Items Manager GS`, avec son `BindReply`. Il s'agit de demander
   l'instance manquante et de réagir à la réponse, plutôt que de laisser tomber l'objet.
   **Pas de poll, pas de timer** : on se branche sur l'événement, conformément à CLAUDE.md §5.
3. Une fois la cause corrigée, le `Delay 0.35` de `PUW_VehicleStorage` (§2) devient redondant et
   peut être retiré.

**Attention** : `InventoryComponent` a 103 graphes et est utilisé par tout le jeu, joueurs
compris. Toute modification s'y fait avec sauvegarde préalable et vérification des trois rôles
réseau.

### 4.2 Boutons "tout prendre" et "tout déposer"

Prévus dans la maquette, **non réalisés**, et volontairement.

**Mesure qui commande la décision** : `InventoryComponent` n'a **aucun RPC de lot**. Le transfert
passe par `SV_InventoryToVehicle` et `SV_VehicleToInventory` (custom events dans
`RoutedPlayerStateFunctions`, RELIABLE, exécutés sur le serveur), **un objet à la fois**.

Un bouton "tout déposer" naïf, c'est donc N RPC fiables en rafale sur un composant partagé par
jusqu'à 500 joueurs. Deux voies :

- ajouter un vrai RPC de lot dans `InventoryComponent` (propre, mais touche le composant partagé) ;
- boucler côté client sur les appels existants (rapide, mais il faut chiffrer la rafale avant).

À arbitrer, pas à improviser.

### 4.3 Titre du contenant

Aujourd'hui le titre vient d'un `Format Text` alimenté par `Get Vehicle Name`, après un
**`Cast To VehicleBase`**. Sur un coffre et sur le builder, le cast échoue et **la broche
`CastFailed` n'est reliée à rien** : ils n'avaient donc aucun titre. Un défaut localisé
"CONTAINER" a été posé, ce qui est correct mais générique.

Note utile : les deux colonnes portent **déjà** leur propre nom via `W_FrameContainer`
("STORAGE VAULT", "BACKPACK"). Le grand titre d'écran fait donc doublon sur un coffre. Deux
options, à trancher :

1. le titre d'écran porte le nom du contenant, fourni par le référent ;
2. on supprime le titre d'écran et on ne garde que le sur-titre "TRANSFERT".

### 4.4 Masse en pied de page

Retirée de la maquette. `GetInventoryWeight` existe bien sur `InventoryComponent`, mais **aucun
dispatcher ne signale son changement**. Un binding UMG serait donc un poll par frame, ce que
CLAUDE.md §5 interdit. À faire seulement le jour où une source événementielle existe.

---

## 5. Pièges d'outillage, tous payés sur ce chantier

Cette section fait gagner des heures. Chaque ligne correspond à une erreur réellement commise.

- **`save_loaded_asset` rend `True` même quand la compilation a échoué.** Un asset a été sauvé
  dans un état qui ne compilait pas, sur la foi de ce booléen. La seule vérification valable :
  noter la position du log (`wc -l` sur `Saved/Logs/QANGA.log`), compiler, relire le delta, et
  exiger **zéro ligne `[Compiler]` et zéro `failed to compile`**.
- **`InventoryComponent.RefreshInventory` est privée.** L'appeler depuis un autre Blueprint fait
  échouer la compilation. `W_Inventory.SetActorInventoryRef`, elle, est publique.
- **Le grep binaire d'un `.uasset` ment en négatif.** `HorizontalBox_79` et `Spacer_69` sont
  ressortis "absents" alors que l'arbre les contenait. Ne conclure une absence qu'avec l'outil
  éditeur, jamais avec un grep.
- **Un `OverlaySlot` ne vaut PAS Fill par défaut**, il vaut gauche/haut. C'est ce qui a fait que le
  cadre refondu se recroquevillait en haut à gauche. Après tout reparentage, relire les alignements
  qu'on n'a **pas** écrits.
- **Un brush `DrawAs=BOX` sans texture peut ne rien peindre.** Le témoin fiable sur cet écran est
  `Image_100` : `DrawAs=IMAGE`, `ResourceObject=None`, et il s'affiche.
- **`manage_blueprint_graph action=connect_pins`** attend `source_node` / `source_pin` /
  `target_node` / `target_pin`, et non `from_*` / `to_*`.
- **`CanvasPanelSlot`** n'expose ni `Anchors` ni `Offsets` en écriture : utiliser `set_anchors`,
  `set_offsets`, `set_alignment`, `set_auto_size`, `set_z_order`. En lecture, passer par
  `LayoutData`. Et poser explicitement `set_auto_size(False)`, sinon les tailles sont ignorées.
- **Deux `CLIscapeMCP.exe` vivants** provoquent des refus de connexion en rafale qui ressemblent à
  un outil cassé. Compter les processus avant de conclure.

---

## 6. Checklist avant de dire que c'est fait

- [ ] Sauvegarde du `.uasset` hors du projet, avec son md5, avant la première écriture.
- [ ] Compilation vérifiée dans le **log**, pas via le retour de `save`.
- [ ] Les trois noms de widgets liés par `BndEvt__*` sont intacts.
- [ ] Ouverture et fermeture de l'écran testées sur un **coffre** (le couvercle animé se referme).
- [ ] Un dépôt testé **dans les deux sens**.
- [ ] Testé sur les trois référents, pas seulement celui qu'on avait sous la main.
- [ ] Ce qui n'a pas pu être vérifié est écrit noir sur blanc, avec qui doit le valider.
