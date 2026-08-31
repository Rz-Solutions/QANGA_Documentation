# Migration QSpringArm_Component vers C++

- **État :** intégration native et cleanup legacy terminés ; matrice visuelle élargie encore ouverte
- **Module :** QSystem (existant, pas de nouveau plugin)
- **Dernière vérification :** 2026-08-31

---

## 1. Verdict

`QSpringArm_Component` est un `SceneComponent` Blueprint de `340` nœuds, `41` variables et `9` fonctions. Son `Update` de `117` nœuds produit chaque frame la position et la rotation caméra. Toutes les `0,2 s`, `Whiskers_Update` déclenche `24` sweeps de moustaches puis `3` pre-probes, soit `27` sweeps. Le Blueprint conserve aussi des branches de debug déconnectées, des calculs sans consommateur et un booléen `Enabled` sans producteur vivant.

La migration crée `UQSpringArmComponent` dans `QSystem`. Le composant natif devient l'unique propriétaire du probe, des moustaches, des interpolations et du dispatcher caméra. Il utilise directement les traces Engine, ne ticke que pour un `PlayerController` local, coupe tout travail sur dedicated server et remplace le timer par une deadline polled dans le tick existant. Aucun fallback, scheduler parallèle ou calcul mort n'est conservé.

## 2. Contrat à préserver

Le dispatcher `Out_Location` reste compatible avec ses quatre sorties exactes :

- `Location : FVector` ;
- `ToNear : bool` ;
- `NearAlpha : double` ;
- `CameraRotation : FRotator`.

ALS consomme ce dispatcher pour la caméra third person et l'activation de `QCameraControlComponent`. Le drone le lie aussi après découverte du composant. Les entrées authored encore vivantes restent exposées pendant la transition : longueur, côté caméra, scope, acteurs ignorés, crouch et vélocité virtuelle.

## 3. Frontière native

### 3.1 Travail par frame

Le chemin par frame conserve uniquement :

1. placement latéral et compensation verticale ;
2. interpolation des origins et de la longueur ;
3. probe sphérique avec récupération d'overlap ;
4. trace scope uniquement quand le scope est actif ;
5. calcul proximité, rotation `MakeFromXZ` et broadcast.

Le tick est event-driven par la possession. Un pawn distant, non possédé, contrôlé par AI ou exécuté sur dedicated server ne produit aucun travail caméra.

### 3.2 Travail à 5 Hz

Les moustaches effectuent `8` sweeps horizontaux et `4 × 4` sweeps supérieurs, soit `24` sweeps toutes les `0,2 s`. La deadline est vérifiée dans le tick déjà existant, sans `FTimerManager`, sans rattrapage en rafale et sans polling supplémentaire quand le composant est inéligible.

Les deux particularités authored sont conservées pendant la migration : l'accumulation top utilise la direction initiale non tournée et son alpha est divisé par le count simple. Les corriger en même temps changerait le comportement caméra sans baseline visuelle dédiée.

### 3.3 Collision

Les helpers de `Cy_ASpringArm` ne restent pas une dépendance du nouveau composant. Les sweeps utilisent directement les canaux projet correspondant à `TraceTypeQuery1` et `TraceTypeQuery2`, les acteurs ignorés existants et les rayons authored. Le probe de même rayon réutilise son premier résultat au lieu de dupliquer la trace.

### 3.4 Fermeture source

La revue du graphe live distingue les deux repères réellement exécutés : `Process_Z_Compensator` tourne le placement d'épaule dans le yaw local du pawn, tandis que les trois pre-probes et la cible latérale de `Whiskers_Update` partent du right vector monde du composant puis appliquent les yaw monde `+35°/-35°`. L'ancienne île de calcul pawn-local présente dans `Whiskers_Update` n'a aucun consommateur et n'est pas transcrite dans le natif.

Le composant démarre inactif, reproduit une seule fois au `BeginPlay` l'activation que l'ancien `Init` imposait, puis sépare cet intent authored de l'éligibilité au tick. Les changements de possession ne réactivent jamais un composant désactivé explicitement. Les counts, longueurs, rayons, scalaires et la vélocité virtuelle présente à l'activation sont validés avant le chemin de tick, y compris la représentabilité des valeurs converties vers les API `float` Engine ; une configuration invalide désactive le composant avec une erreur bornée et empêche toute division par un count nul.

Le source QATS adjacent couvre l'état initial inactif, l'absence de tick non possédé, le refus d'un `PlayerController` non local et d'un `AIController`, l'activation par joueur local, la repossession, la désactivation authored persistante, le rejet des counts nuls et la première interpolation exacte depuis l'état zéro. Ces tests sont écrits mais n'ont pas été exécutés dans cette lane.

## 4. Cleanup réalisé

Le nœud SCS ALS et les six bindings Blueprint encore sérialisés utilisent directement `UQSpringArmComponent`. ALS et le drone restent abonnés au dispatcher natif ; le wrapper n'a plus aucun referencer Asset Registry et a été supprimé. Un `ClassRedirect` conserve la lecture des copies historiques non versionnées, avec les redirects de propriété nécessaires à l'ancien libellé authored.

Le package script, les deux classes exportées et le plugin `Cy_ASpringArm` sont à zéro referencer ; aucun autre plugin ne le déclare comme dépendance et aucune référence de production externe ne subsiste. Son manifeste, son module, ses helpers, son contenu généré et ses règles d'ignore ont donc été supprimés au lieu de conserver un plugin dormant.

La timeline ALS `InterpCameraArmLength` ne fait pas partie de ce cleanup : elle pilote `3rdPersonCameraAlpha` pour le blend de view mode et ne possède pas la longueur du spring arm.

## 5. Gates de validation

- [x] Build froid Win64 de la classe native après suppression du plugin legacy
- [x] QATS de la possession locale, de la repossession, des refus AI/non-local, de la désactivation authored et du premier calcul
- [x] Reclassement SCS sans perte des overrides `Length=210` et `CameraOffset=62`
- [x] Compile ALS, AILean, AI_Cyborg et IS_DroneBase à `0` erreur / `0` warning
- [x] Zéro tick sur AI, pawn distant, pawn non possédé et dedicated server couvert par le contrat/QATS
- [x] PIE `L_Dev_Rz` : rotation de contrôle transmise à la caméra, crouch `165,817662 -> 105,817662 cm` puis retour exact
- [ ] PIE `L_Dev_Rz` : collision proche, mur derrière le joueur, plafond et sortie d'overlap sans pop
- [ ] PIE `L_Dev_Rz` : scope et transitions first/third visuellement inchangés
- [ ] PIE `L_Dev_Rz` : sortie véhicule puis repossession sans second propriétaire caméra
- [x] Binding runtime du dispatcher vers ALS et `IS_DroneBase` présent après redémarrage froid
- [x] Deux arrêts PIE frais retournent le Message Log à `0` erreur / `0` warning
- [x] Wrapper et plugin legacy à zéro référence puis supprimés

Les trois cases visuelles restantes demandent un parcours joueur perceptuel ; elles ne correspondent plus à du code Blueprint de calcul à conserver. Le chemin natif, son ownership, ses consommateurs sérialisés et son teardown sont fermés.
