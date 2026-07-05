# QMODULE : Catalogue de modules (matière première design)

> **Statut : BRAINSTORM STRUCTURÉ, à trier et valider. Rien n'est implémenté.**
> Compagnon de `QMODULE_ARCHITECTURE.md`. Rédigé le 2026-07-04. Objectif : donner une base LARGE (142 entrées, dont les 7 modules de base déjà en jeu, marqués **EN JEU**) pour que le joueur construise son propre build, façon arbre de talents illimité. Les noms sont des propositions ; les chiffres sont des ordres de grandeur à équilibrer.
>
> **Légende** : T = type d'effet : **P** (passif chiffré, pur data), **A** (actif type stratagème, une classe BP), **C** (comportement, un composant attaché). Ancrage : **✔** s'appuie sur un système vérifié dans le moteur ; **?** probable mais à confirmer ; **➕** nouvelle mécanique à créer. **EN JEU** = module v1 déjà implémenté : c'est la base acquise du catalogue, hors triage.

---

## 0. Règles transverses (le multiplicateur de contenu)

- **Manufactures** : chaque rôle du catalogue peut exister en variantes. IC Lab (référence, stable, équilibré) ; Voss (chiffres supérieurs + contrepartie/instabilité) ; Nash Arms (armes, orienté cadence) ; Surplus QPD (défensif, connoté loi) ; Artisanal pirate (stats tirées aléatoirement, pas cher, marché gris). Une variante = mêmes StatTags, autres valeurs + drawbacks : **zéro code**. La liste réelle est donc bien plus grande que les 136 lignes.
- **Tiers** : 1..3 par défaut (aligné sur l'existant qui plafonne à 2 ou 3) ; prototypes rares 1..5.
- **Rareté** : Commun / Avancé / Prototype / Relique, mappée sur le champ `Rarity` déjà présent sur les instances d'items.
- **Exclusivité** : une seule variante d'un même rôle active par cible (ExclusivityTag).
- **Hermétique à la mort** : tout module/phase INSTALLÉ ; le sac reste lootable.
- **Fragments de phase** : un module doublon se recycle en fragments (X fragments = 1 phase tier 1) : aucun loot mort.
- **Schémas de constellation** : les motifs sont des ITEMS à looter (« Schéma : <nom> ») qui dessinent le motif sur le mur ; pas de liste dans un menu.
- **Module de récompense pré-phasé** : quêtes et boss livrent le module avec une phase insérée ; modules sauvages et marchands vides.
- **SynergyTags** : chaque module porte des tags de famille pour les effets d'adjacence sur le Mur (concept §13 du doc d'architecture, à valider).

---

## 1. Cyborg

### A. Mobilité (13)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| **Jetpack** | P | Vol stationnaire + vol rapide + 25 % fuel, puis +50 % fuel (Max 2) : migré tel quel en v2 | **EN JEU** |
| Servomoteurs de jambes | P | Vitesse de sprint +8/16/25 % | ✔ mouvement |
| Amortisseurs cinétiques | P | Dégâts de chute réduits 30/60/100 % | ? dégâts de chute |
| Vérins de saut | P | Hauteur de saut +15/30/50 % | ✔ mouvement |
| Semelles magnétiques | C | Adhérence totale sur surfaces métalliques en pente | ➕ |
| Exosquelette porteur | C | Porte les objets lourds sans malus de vitesse | ? objets lourds |
| Micro-propulseurs | A | Dash directionnel, 2 charges, recharge au sol | ➕ |
| Glisseur | C | Glissade prolongée sans perte de vitesse | ? slide ALS |
| Coussin gravitationnel | C | Chute lente maintenue (consomme de l'énergie) | ? GravityScape |
| Sprint de fond | P | Coût d'endurance du sprint -20/40/60 % | ? stamina |
| Récupérateur cinétique | P | Recharge lente d'énergie en déplacement | ➕ |
| Stabilisateur gyroscopique | P | Précision en mouvement +10/20/30 % | ✔ spread armes |
| Jambes de félin (Voss) | P | Sprint +35 % mais bruit de pas +50 % (instable) | ✔ PawnNoiseEmitter |

### B. Survie (12)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| Blindage sous-cutané | P | Réduction de dégâts +4/8/12 % | ✔ combat |
| Nano-régénérateur | P | Régénération hors combat | ✔ SS_PhysicalState |
| Régulateur thermique | P | Résistance aux climats extrêmes par palier | ✔ climat WorldScape (dégâts climat ?) |
| Surcouche de bouclier | P | Bouclier max +15/30/50 % | ✔ SS_Shield |
| Condensateur | P | Énergie max +20/40/60 % | ? stat énergie |
| Recycleur pulmonaire | P | Autonomie O2/apnée prolongée | ? mécanique O2 |
| Armure réactive (QPD) | C | Renvoie 10 % des dégâts de mêlée subis | ➕ |
| Second souffle | C | Survit à un coup fatal avec 1 PV (long cooldown) | ➕ |
| Isolation Faraday | P | Durée des EMP subis -50 % | ➕ dépend munitions EMP |
| Coque intégrale (Voss) | P | Réduction de dégâts +20 % mais régénération -50 % (instable) | ✔ combat |
| Trousse interne | A | Auto-soin activable, charges limitées | ✔ pattern Repair |
| Balise de détresse | A | Signale ta position aux alliés + regen de groupe brève | ➕ coop |

### C. Combat (12)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| Compensateur neural | P | Recul de toutes les armes -10/20/30 % | ✔ armes |
| Chargeur neuronal | P | Rechargement global +10/20/30 % | ✔ armes |
| Ciblage assisté | P | Précision de hanche +15/30 % | ✔ spread |
| Adrénaline | C | +10 % dégâts pendant 5 s après un kill | ✔ event kill |
| Rage (Voss) | C | Sous 30 % PV : +25 % dégâts, -15 % armure (instable) | ➕ |
| Chasseur de primes | P | Dégâts +10/20 % contre les cibles recherchées | ✔ QPolice wanted |
| Fléau des machines | P | Dégâts +10/20/30 % contre drones et tourelles | ✔ factions QAI |
| Poigne renforcée | P | Dégâts de mêlée +20/40/60 % | ? mêlée |
| Munitions économes | P | 5/10/15 % de chance de ne pas consommer de munition | ✔ ammo |
| Concentration | P | Sway de visée au zoom -30 % | ✔ scopes |
| Réflexes câblés | P | Changement d'arme +25/50 % plus rapide | ✔ équipement |
| Analyste balistique | C | Affiche les multiplicateurs de zone des cibles | ➕ |

### D. Furtivité & Information (11)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| **Scanner** | P | Zone 3/6/10 km, détecte véhicules puis joueurs (Max 3) : socle de Spectromètre / Radar / Analyseur | **EN JEU** |
| Brouilleur de signature | P | Rayon de détection des IA -15/30/45 % | ✔ QAI perception |
| Pas feutrés | P | Bruit de déplacement -30/60/90 % | ✔ PawnNoiseEmitter |
| Radar passif | C | Blips des hostiles proches sur la boussole | ✔ pattern Scanner |
| Marqueur tactique | C | Les cibles touchées restent surlignées 5 s | ➕ |
| Décodeur QPD | P | Décroissance du niveau de recherche +50 % | ✔ QPolice |
| Interception radio | C | Capte les canaux police/Voss (alertes, lore) | ✔ QRadio |
| Vision nocturne | C | Mode vision de nuit activable | ➕ post-process |
| Analyseur de menace | C | Niveau et faction des cibles scannées | ✔ Scanner |
| Peau caméléon (Voss) | C | Quasi-invisible immobile, drain d'énergie (instable) | ➕ |
| Faux transpondeur | P | Les scanners ennemis t'affichent comme neutre | ➕ PvP sensible |

### E. Économie & Prospection (12)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| Compacteur de matière | P | Matter max +200/400/600 | ✔ module actuel (General System) |
| Aimant de collecte | C | Ramassage automatique dans un rayon de 3/5/8 m | ➕ |
| Spectromètre | C | Les ressources riches brillent au scan | ✔ Scanner |
| Négociateur | P | Achats -5/10/15 %, ventes +5/10/15 % | ✔ shops |
| Fortune de guerre | P | Chance de loot rare +10/20/30 % | ✔ Rarity instances |
| Géologue | P | Rendement du minage +15/30/50 % | ✔ mining |
| Recycleur optimisé | P | Matière récupérée au recyclage +25/50 % | ✔ ServerTryRecycleItem |
| Sacoche dimensionnelle | P | Taille d'inventaire +2/4/6 slots | ✔ InventoryMaxSize |
| Prospecteur orbital | A | Ping global des ressources 10 s (long cooldown) | ✔ Scanner étendu |
| Assurance IOLA | P | Conserve 25/50 % des coins à la mort | ? règles de mort |
| Contrebandier | C | Vend les objets marqués/volés chez les pirates | ➕ |
| Protocole de partage | P | Récompenses de groupe +10 % | ➕ coop |

### F. Ingénierie & Déployables (16)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| **Drone** | P | Réparation +25/50 %, résistance, flashbang 20 s (Max 2) : socle des futurs drones actifs | **EN JEU** |
| **Repair** | P | +60/+80 réparation, cooldown 25/20 s (Max 2) : migré tel quel en v2 | **EN JEU** |
| Tourelle sentinelle | A | Tourelle automatique temporaire | ➕ (socle QAI ✔) |
| Drone médical | A | Drone qui soigne en zone | ✔ base drone existante |
| Drone de combat | A | Drone offensif temporaire | ✔ IS_DroneBase + QAI |
| Bulle de bouclier | A | Dôme bloquant les projectiles quelques secondes | ➕ |
| Balise de frappe | A | Frappe aérienne sur zone marquée après délai | ➕ |
| Largage de ravitaillement | A | Caisse de munitions/soins pour le groupe | ✔ items |
| Leurre holographique | A | Copie qui attire l'aggro des IA | ✔ QAI perception |
| Mine de proximité | A | Mine posée avec gestion ami/ennemi | ➕ |
| Grappin | A | Traction vers un point d'ancrage | ➕ |
| Mur instantané | A | Barricade dépliable temporaire | ✔ meshes QBuilder |
| Impulsion EMP | A | Désactive drones/véhicules proches quelques secondes | ➕ |
| Fabricateur de munitions | A | Convertit de la Matter en munitions | ✔ Matter + ammo |
| Kit de sabotage | A | Désactive portes/tourelles (interaction longue) | ➕ |
| Réparateur automatique | C | Répare lentement le véhicule piloté | ✔ pattern Repair |

### G. Système & Social (8)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| **Système Général** | P | +200 Matter par niveau (Max 2) : en v2, devient le Cœur IC Lab qui pilote la capacité du Mur (non échangeable) | **EN JEU** |
| Interface de contrats | P | +1/+2 contrats simultanés | ✔ ContractTargetComponent |
| Réputation IC Lab | P | Accès et prix armurerie IC Lab améliorés | ✔ shops (réputation ➕) |
| Accréditation Voss | P | Accès aux marchands Voss, prix améliorés | ✔ shops (réputation ➕) |
| Voix de commandement | C | Buff léger de regen/endurance aux alliés proches | ➕ coop |
| Traducteur universel | C | Débloque dialogues et quêtes annexes | ✔ DQS (design) |
| Boîte noire | C | Balise sur ton cadavre visible par ton équipe | ➕ |
| Filon quotidien | P | Bonus de récompenses sur les premières activités du jour | ➕ rétention, à débattre |

### H. Pilotage (8)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| Interface neurale de pilotage | P | Maniabilité des véhicules +10/20/30 % | ✔ QVehicles |
| Injection nerveuse | P | Réactivité du boost véhicule + | ✔ QVehicles |
| Mécano de bord | C | Régénération lente de la coque en pilotant | ✔ vie véhicule |
| As de l'esquive | C | Alerte de danger/verrouillage en vol | ➕ |
| Longue-vue stellaire | C | Portée d'affichage StarMap augmentée | ? StarMap |
| Docker aguerri | P | Entrée/sortie et amarrage +50 % plus rapides | ➕ |
| Antenne longue portée | C | Capte les stations QRadio plus loin | ✔ QRadio |
| Pilote de chasse (Voss) | P | +20 % vitesse aérienne, surchauffe moteur (instable) | ➕ |

---

## 2. Armes (24, par exemplaire, installés à l'établi)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| **AT56** | P | +30/+20/+20 dégâts et cadence (Max 3) : bascule v2 du niveau par type vers le rack par exemplaire | **EN JEU** |
| **All Nash Weapons** | P | Cadence +10/20/40 %, dégâts +5/10/15 % sur toute la famille Nash (Max 3) : précédent des modules de famille | **EN JEU** |
| Amplificateur de dégâts | P | Dégâts +5/10/15 % | ✔ |
| Accélérateur de culasse | P | Cadence +8/16/25 % | ✔ |
| Chargeur étendu | P | Chargeur +20/40/60 % | ✔ |
| Auto-chargeur | P | Rechargement +15/30/45 % | ✔ |
| Compensateur | P | Recul -15/30/45 % | ✔ |
| Canon rayé | P | Portée/précision + | ✔ |
| Munitions perforantes | P | Ignore une part de l'armure | ? modèle d'armure |
| Munitions EMP | P | Bonus vs drones/véhicules + effet bref | ➕ |
| Réducteur de signature | P | Aggro et wanted générés au tir réduits | ✔ QPolice/QAI |
| Chambre cryogénique | P | Ralentit brièvement la cible | ➕ statuts |
| Chambre incendiaire | P | Dégâts sur la durée | ➕ statuts |
| Ventilation forcée | P | Surchauffe plus lente (armes énergie) | ? surchauffe |
| Lunette intégrée | C | Zoom sans pièce lunette | ✔ scopes |
| Pointeur laser | P | Précision de hanche + | ✔ |
| Détecteur de failles | P | Chance de critique +3/6/9 % | ? critiques |
| Munitions traçantes | C | Cibles touchées marquées pour l'équipe | ➕ |
| Culasse allégée | P | Vitesse de switch +30 % | ✔ |
| Module vampirique (Voss) | C | +3 PV par kill, -1 PV/10 s hors combat (instable) | ➕ |
| Double détente (Voss) | C | 10 % de chance de tir double, recul accru (instable) | ➕ |
| Stabilisateur Nash | P | Précision en rafale + (famille Nash) | ✔ héritage P_Nash |
| Silencieux intégré | C | Tir moins bruyant pour la détection | ? ouïe IA |
| Éjecteur rapide | C | Recharge en sprint sans malus | ➕ |

---

## 3. Véhicules (18, par exemplaire, sol et vol)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| Turbocompresseur | P | Vitesse max +8/16/25 % | ✔ |
| Injecteur de boost | A | Boost consommable rechargeable | ? boost existant |
| Réservoir étendu | P | Carburant max +25/50 % | ✔ fuel |
| Économiseur | P | Consommation -15/30 % | ✔ fuel |
| Blindage châssis | P | Vie du véhicule +20/40 % | ✔ |
| Pare-buffle renforcé | P | Dégâts de collision infligés +, subis - | ➕ |
| Suspensions tout-terrain | P | Stabilité hors route + | ✔ QVehicles |
| Radar embarqué | C | Blips des menaces autour du véhicule | ✔ pattern Scanner |
| Soute agrandie | P | Stockage du véhicule + | ? coffres véhicule |
| Stabilisateur de vol | P | Stabilité et précision de vol + | ✔ FlyVehicleMovement |
| Régulateur de croisière | C | Maintien automatique de la vitesse | ➕ simple |
| Treuil | A | Remorque une épave ou un véhicule | ➕ |
| Camouflage thermique | P | Détection IA réduite à l'arrêt | ✔ QAI |
| Éjecteurs de fumée | A | Rideau de fumée derrière le véhicule | ➕ FX |
| Système antivol | C | Verrouillage aux non-propriétaires | ? ownership |
| Phares longue portée | C | Éclairage étendu de nuit | ✔ trivial |
| Sono de bord | C | Diffuse la QRadio vers l'extérieur | ✔ QRadio |
| Noyau surcadencé (Voss) | P | +30 % vitesse, pannes moteur aléatoires (instable) | ➕ |

---

## 4. Liaison & Synergie (8 ; adjacence VALIDÉE le 2026-07-04 : au cœur de la V1)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| Connecteur | P | +5 % d'efficacité aux 2 modules qu'il relie | ➕ adjacence |
| Amplificateur de famille | P | +10 % aux voisins de la même famille | ➕ adjacence |
| Répartiteur | P | Copie 25 % du bonus du meilleur voisin vers les autres | ➕ adjacence |
| Catalyseur | P | +15 % à lui-même par voisin au niveau max | ➕ adjacence |
| Stabilisateur | P | Réduit de moitié l'instabilité des Voss adjacents | ➕ adjacence |
| Résonateur (Voss) | P | Amplifie bonus ET malus des voisins (instable) | ➕ adjacence |
| Noyau de constellation | P | Bonus de set si un motif hexagonal est complété | ➕ adjacence |
| Parasite pirate | C | Vole 10 % des bonus des voisins (instable) | ➕ adjacence |

---

## 5. Exemples de builds (le fantasme joueur)

- **Le Fantôme** (infiltration) : Brouilleur de signature, Pas feutrés, Peau caméléon, Radar passif, Vision nocturne, Concentration + arme : Silencieux intégré, Détecteur de failles.
- **Le Bulldozer** (tank) : Blindage sous-cutané, Coque intégrale Voss, Surcouche de bouclier, Second souffle, Rage Voss, Poigne renforcée + arme : Compensateur, Chambre incendiaire.
- **L'Ingénieur de siège** : Tourelle sentinelle, Drone de combat, Mur instantané, Fabricateur de munitions, Largage de ravitaillement, Condensateur + liaison : Amplificateur de famille.
- **Le Prospecteur** : Compacteur de matière, Spectromètre, Géologue, Aimant de collecte, Sacoche dimensionnelle, Négociateur, Prospecteur orbital, Fortune de guerre.
- **L'As du volant** : Interface neurale, Mécano de bord, Docker aguerri, Antenne longue portée + véhicule : Turbocompresseur, Injecteur de boost, Treuil, Éjecteurs de fumée.
- **Le Chasseur de primes** : Chasseur de primes, Décodeur QPD, Analyseur de menace, Marqueur tactique, Interface de contrats, Adrénaline, Réflexes câblés + arme : Munitions traçantes.

---

## 6. Backlog d'équilibrage (à traiter avant la mise en contenu massive)
- Caps par StatTag (ClampMax) définis AVANT le premier batch de contenu.
- Budget de puissance : la capacité du Mur (Système Général) est LE levier global ; courbe à modéliser.
- Les entrées **?** exigent une vérification moteur ; les **➕** exigent une mécanique nouvelle (prioriser celles qui servent plusieurs modules : statuts élémentaires, énergie, adjacence).
- Passe de nommage/lore (manufactures, cohérence univers IOLACORP) avec l'équipe.

## 7. Vague 2 : calibrage gameplay (directives RzZz, 2026-07-04 soir)

> Directive : la liste actuelle est une bonne base MAIS doit être agrémentée de modules PLUS COHÉRENTS avec le gameplay réellement en place. Le batch de production (M4.5) attend cette vague. Exemples concrets donnés :

**Modules cyborg / inventaire (validés dans l'esprit : « tous les modules qui touchent l'inventaire, c'est bien »)**
- **Slots hermétiques à la mort** : le module réserve 1 à 5 slots d'inventaire (selon le niveau de phases) dont le contenu SURVIT à la mort. Les items de quête occupent automatiquement ces slots en priorité. Contexte : des sacs à dos « digitiques » existent déjà en jeu : vérifier leur mécanique avant chiffrage (interaction module x sac à définir).

**Modules d'ARME (par exemplaire, à l'établi)**
- **Recycleur de douilles** : récupère de la matière sur les douilles éjectées au tir (+1 ou +2 Matter par tir recyclé, directement au personnage). Module d'arme, pas de cyborg.
- **Chambre thermique** (« balles chauffées ») : chauffe les balles avant le tir : portée réduite, dégâts nettement supérieurs, et PERCE les surfaces (pénétration). Triple contrepartie/bonus : typé Voss ?

**Axe SCAN (extensions à inventer, demande explicite : « trouver des extensions de module pour le scan »)**
- Pistes à proposer : surlignage des ressources récoltables, détection des douilles/loot au sol, marquage des drones ennemis, empreinte thermique des véhicules moteur allumé, scan des slots hermétiques des cadavres, cartographie éphémère des QLevels proches.

**Axe DRONE / VUE TROISIÈME PERSONNE (lore : la 3e personne EST un drone physique)**
- Fait de gameplay CONFIRMÉ par RzZz : le drone de vue 3P est visible derrière le joueur ; si on TIRE sur le drone d'un joueur, celui-ci est FORCÉ en vue FPS jusqu'à l'auto-réparation du drone. C'est « quelque chose d'énergétique dans notre gameplay ».
- Pistes modules : blindage du drone (PV du drone +), auto-réparation accélérée, drone furtif (silhouette réduite/discrétion), drone déporté (angle/hauteur de caméra alternatif), leurre (le drone encaisse UN tir gratuit), contre-mesure (révèle le tireur qui a touché le drone).

**Axes à couvrir pour la cohérence (systèmes vivants du jeu)**
- **Construction (QBuilder)** : ex. remise sur le coût en minéraux, portée de placement, vitesse de construction, recyclage de structures amélioré (lien avec l'économie builder-bank existante).
- **Économie / revente** : ex. meilleurs prix de revente, taxes réduites, estimation de valeur au scan.
- **Exploration** : ex. réduction du coût de fuel en vol atmosphérique, résistance aux climats extrêmes (l'API climat WorldScape par position existe), balises personnelles.

**Prochain pas M4.5** : produire cette vague 2 en entrées de catalogue complètes (famille, fabricant, rareté, niveaux, contreparties), PUIS générer le batch d'assets (les ✔ + la vague 2 chiffrable).

### 7.1 Les 25 entrées de la vague 2 (rédigées le 2026-07-04, à valider par RzZz)

Format : Nom [Famille, Fabricant, Type, Ancrage ✔/?/➕] : effet par niveau.

**Inventaire et survie**
- **Caisson hermétique** [SUR, IC Lab, Passif, ➕] : 1 slot d'inventaire par niveau (Max 5) dont le contenu SURVIT à la mort ; les items de quête s'y rangent automatiquement en priorité. Interaction avec les sacs digitiques à cadrer.
- **Sac digitique étendu** [ECO, IC Lab, Passif, ✔ SetInventoryBaseSize] : +4/8/12 slots d'inventaire.
- **Compacteur de matière** [ECO, IC Lab, Passif, ?] : taille de pile des ressources +25/50/100 %.
- **Membrane climatique** [SUR, IC Lab, Passif, ✔ API climat WorldScape par position] : résistance aux climats extrêmes par niveau.

**Armes (par exemplaire, directive RzZz)**
- **Recycleur de douilles** [ARM, IC Lab, Passif, ? tir + AddMatter] : récupère la matière des douilles éjectées : +1/+2 Matter par tir.
- **Chambre thermique** [ARM, VOSS, Passif, ➕ perforation] : balles chauffées : dégâts +20/35/50 %, portée -15 %, PERCE les surfaces. Instable.

**Extensions Scanner (6 propositions, demande RzZz « je n'ai pas trop d'idées »)**
- **Spectromètre de gisements** [SYS, IC Lab, Conditionnel, ?] : le scan surligne les ressources récoltables.
- **Traqueur de butin** [SYS, IC Lab, Conditionnel, ?] : le scan révèle items au sol et caisses.
- **Oeil thermique** [SYS, IC Lab, Conditionnel, ✔ OnVehicleStateUpdate] : marque les véhicules moteur ALLUMÉ.
- **Analyse nécrologique** [SYS, IC Lab, Conditionnel, ➕] : scanner un cadavre révèle son loot et ses slots hermétiques.
- **Écho de constellation** [SYS, IC Lab, Actif, ?] : impulsion révélant les sites d'intérêt (QLevels) non découverts proches.
- **Contre-scan** [SYS, VOSS, Conditionnel, ➕] : alerte quand TU es scanné + direction approximative. Instable.

**Famille Drone / vue 3P (fait de gameplay confirmé : la 3e personne EST un drone physique abattable)**
- **Blindage de drone** [SYS, IC Lab, Passif, ✔ PV du drone] : PV du drone +25/50/75 %.
- **Nano-réparateur de drone** [SYS, IC Lab, Passif, ✔ auto-réparation existante] : temps de réparation -20/40/60 %.
- **Drone spectre** [FUR, VOSS, Passif, ?] : silhouette du drone réduite, plus dur à repérer/toucher. Instable.
- **Gyroscope déporté** [SYS, IC Lab, Conditionnel, ? caméra] : position de la caméra-drone ajustable (épaule, hauteur).
- **Leurre holographique** [SYS, VOSS, Conditionnel, ➕] : le premier tir qui abattrait le drone est absorbé (long cooldown). Instable.
- **Mouchard réflexe** [COM, VOSS, Conditionnel, ➕] : drone abattu = tireur marqué 5 s. Instable.

**Construction (QBuilder, économie builder-bank existante)**
- **Optimiseur de chantier** [ING, IC Lab, Passif, ?] : coût minéraux des constructions -5/10/15 %.
- **Grue gravitique** [ING, IC Lab, Passif, ?] : portée de placement +25/50 %.
- **Démolisseur certifié** [ING, IC Lab, Passif, ?] : recyclage de structures +10/20/30 % de matériaux rendus.

**Économie / revente**
- **Négociateur** [ECO, IC Lab, Passif, ? boutiques] : prix de VENTE aux marchands +3/6/9 %.
- **Estimateur** [ECO, IC Lab, Conditionnel, ?] : valeur marchande des items visible au scan.

**Exploration**
- **Régulateur atmosphérique** [PIL, IC Lab, Passif, ?] : consommation de carburant en vol -10/20/30 %.
- **Balise personnelle** [SYS, IC Lab, Actif, ➕] : pose 1/2/3 balises personnelles visibles au HUD.

> Total vague 2 : **25 modules** (catalogue : 136 → 161). Les ✔ sont chiffrables dès le batch M4.5 ; les ? demandent une vérification moteur ciblée ; les ➕ attendent leur mécanique (slots de mort, perforation, marquage).

## 8. TRIAGE RzZz (2026-07-04, export du tableau de bord) : la référence du batch M4.5

- **7 EN JEU** : hors triage, acquis d'office (QMD_* déjà créés).
- **GARDER (82) = BATCH 1 DE PRODUCTION** : l'essentiel du cyborg (Mobilité 6, Survie 10, Combat 10, Furtivité 11, Économie 9, Ingénierie/Déployables 14, Système/Social 17, Pilotage 2) + 3 modules d'ARMES seulement (Amplificateur de dégâts, Recycleur de douilles, Chambre thermique). Quasi toute la vague 2 gameplay est GARDÉE (drone/scanner/inventaire).
- **REPORTER (78)** : TOUT le détail armes (accessoires d'établi : chargeurs, canons, munitions spéciales... 22 entrées), TOUS les véhicules (18, y compris Noyau surcadencé : l'asset de test QMD_NoyauSurcadence reste comme harnais), TOUTE la Liaison & Synergie (8 : attendra le chiffrage de l'adjacence), la vague construction QBuilder (3), et le reste du fond de catalogue.
- **COUPER : 0.** Rien n'est jeté : le catalogue entier reste la matière première.
- **Arbitrages de dédoublonnage appliqués** : Compacteur de matière et Négociateur existaient en double (v1 + vague 2) : une seule définition chacun (l'instance GARDER) ; Spectromètre (v1, gardé) vs Spectromètre de gisements (vague 2, reporté) ; **deux Leurres distincts assumés** : Leurre holographique (déployable, Ingénierie) et **Leurre de drone** (renommé, protection de la vue 3P, tag Module.LeurreDrone) ; Membrane climatique : reportée.

> Génération : script `qmodule_batch_m45.py` (82 défs dans /Game/Phases/QModuleV2, tags Module.* + 15 Stat.* nouveaux + Manufacturer.ICLab/Voss auto-enregistrés dans QModuleTags.ini, StatMods chiffrés pour les effets numériques sûrs, le reste = coquilles à équilibrer dans l'Éditeur de Modules).
