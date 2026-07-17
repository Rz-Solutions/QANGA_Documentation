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

### A. Mobilité (14)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| **Jetpack** | P | Vol stationnaire + vol rapide + 25 % fuel, puis +50 % fuel (Max 2) : migré tel quel en v2 | **EN JEU** |
| Servomoteurs de jambes | P | Vitesse de sprint +8/16/25 % | ✔ mouvement |
| Amortisseurs cinétiques | P | Dégâts de chute réduits 30/60/100 % | ? dégâts de chute |
| **Vérins de saut** (2026-07-11, BRANCHÉ + VALIDÉ) | P | Hauteur de saut +15/30/50 % (vraie hauteur : vélocité × racine du facteur ; base BP 730 → 894 au max) | ✔ push façade JumpZVelocity |
| Semelles magnétiques | C | Adhérence totale sur surfaces métalliques en pente | ➕ |
| Exosquelette porteur | C | Porte les objets lourds sans malus de vitesse | ? objets lourds |
| Micro-propulseurs | A | Dash directionnel, 2 charges, recharge au sol | ➕ |
| **Propulseur orbital** (2026-07-11, BRANCHÉ + VALIDÉ) | P | Poussée du jetpack dans l'espace +50/125/200 % | ✔ DynamicFlightComponent.Get_FlySpeed (facteur spatial pur-node, gravité locale < 0.05) |
| Glisseur | C | Glissade prolongée sans perte de vitesse | ? slide ALS |
| Coussin gravitationnel | C | Chute lente maintenue (consomme de l'énergie) | ? GravityScape |
| Sprint de fond | P | Coût d'endurance du sprint -20/40/60 % | ✖ REPORTÉ (recon 2026-07-11 : AUCUN système de stamina dans le jeu : le module n'aurait rien à réduire) |
| Récupérateur cinétique | P | Recharge lente d'énergie en déplacement | ➕ |
| Stabilisateur gyroscopique | P | Précision en mouvement +10/20/30 % | ✔ spread armes |
| Jambes de félin (Voss) | P | Sprint +35 % mais bruit de pas +50 % (instable) | ✖ drawback bruit sans effet (pas d'ouïe IA) : à re-designer |

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

### C. Combat (14)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| Compensateur neural | P | Recul de toutes les armes -10/20/30 % | ✔ armes |
| Chargeur neuronal | P | Rechargement global +10/20/30 % | ✔ armes |
| Ciblage assisté | P | Précision de hanche +15/30 % | ✔ spread |
| Adrénaline | C | +10 % dégâts pendant 5 s après un kill | ✔ event kill |
| Rage (Voss) | C | Sous 30 % PV : +25 % dégâts, -15 % armure (instable) | ➕ |
| **Chasseur de primes** (2026-07-11, BRANCHÉ + VALIDÉ) | P | Dégâts +10/20/30 % contre joueurs recherchés ET hors-la-loi (pirates, Voss) | ✔ Lib_Life post-armure (IsPlayerWanted + faction cible ; rack lu via InstigatorController→pawn) |
| **Fléau des machines** (2026-07-11, BRANCHÉ + VALIDÉ) | P | Dégâts +10/20/30 % contre les machines (drones, Autonomous, vaisseaux, nanites + tag acteur QMachine pour tourelles posées) | ✔ Lib_Life post-armure + helper C++ QAI_IsMachineActor |
| Poigne renforcée | P | Dégâts de mêlée +20/40/60 % | ? mêlée |
| Munitions économes | P | 5/10/15 % de chance de ne pas consommer de munition | ✔ ammo |
| Concentration | P | Sway de visée au zoom -30 % | ✔ scopes |
| Réflexes câblés | P | Changement d'arme +25/50 % plus rapide | ✔ équipement |
| Analyste balistique | C | Affiche les multiplicateurs de zone des cibles | ➕ |
| **Nid de guêpes** (2026-07-11, BRANCHÉ + VALIDÉ) | A | Ordnance dorsale style Anthem : 1/2/4 micro-missiles autoguidés courte portée (60 m, ~100 dégâts + souffle 2.5 m), recharge 14/10/6 s, tracking = CHANCE DE TOUCHER 60/80/100 % (les ratés partent dans le décor) | ✔ SV_TriggerShoulderMissiles (visuel BP_Missile ×0.5 homing réel, dégâts serveur Lib_Life) |
| **Nid de frelons** (2026-07-11, QMD créé + BRANCHÉ + VALIDÉ) | A | Ordnance dorsale : 1/2/4 grenades lobées sur le point visé (50 m, 70 dégâts, souffle 6 m), recharge 12/9/6 s | ✔ SV_TriggerShoulderGrenades (visuel BP_GrenadeProjectile balistique, dégâts serveur Lib_Life) |

### D. Furtivité & Information (13)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| **Scanner** | P | Zone 3/6/10 km, détecte véhicules puis joueurs (Max 3) : socle de Spectromètre / Radar / Analyseur | **EN JEU** |
| **Brouilleur de signature** (2026-07-11, BRANCHÉ + VALIDÉ) | P | Recherché : les vagues de la police se déploient jusqu'à 75/150/300 m à côté de ta vraie position | ✔ QPolice (dispatch, renforts, vaisseaux, resolver naturel : 4 points brouillés) |
| **Masque phéromonal** (2026-07-11, QMD créé + BRANCHÉ + VALIDÉ) | P | La faune au sol te détecte à 85/70/55 % de son rayon (sanglines incluses, police non concernée) | ✔ QAI scan d'acquisition (mult par joueur, faune uniquement) |
| **Contre-mesures Voss** (2026-07-11, QMD créé + BRANCHÉ + VALIDÉ) | P | Les patrouilles Voss te détectent à 85/70/55 % de leur rayon | ✔ QAI scan d'acquisition (2e canal par faction de l'agent) |
| Pas feutrés | P | Bruit de déplacement -30/60/90 % | ✖ REPORTÉ (recon 2026-07-11 : QAI n'écoute AUCUN bruit : détection = vue/distance/LOS uniquement ; à re-designer ou attendre un sens Hearing) |
| Radar passif | C | Blips des hostiles proches sur la boussole | ✔ pattern Scanner |
| Marqueur tactique | C | Les cibles touchées restent surlignées 5 s | ➕ |
| **Décodeur QPD** (2026-07-11, BRANCHÉ + VALIDÉ) | P | Décroissance du wanted +25/50/75 % plus rapide par phase | ✔ QPoliceSubsystem (multiplicateur par joueur sur les 4 replanifications du decay) |
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
| Fortune de guerre | P | Chance de loot rare +10/20/30 % | ⏳ DATA PRÊTE (stat chiffrée) ; branchement reporté : AUCUN tirage de rareté n'existe dans le pipeline loot BP actuel (rareté bakée dans la persistance) : passe loot dédiée à prévoir |
| **Géologue** (2026-07-11, BRANCHÉ + VALIDÉ) | P | Rendement du minage +15/30/50 % (quantité par extraction, arrondie serveur) | ✔ RecyclerComponent.ServerGiveItem (StackRes × stat du pawn mineur) |
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
| **Balise de frappe** (2026-07-11, BRANCHÉE + VALIDÉE) | A | Frappe aérienne au point visé : 8/14/22 missiles, zone 10/14/18 m, cooldown 300/240/180 s, délai d'arrivée 3 s | ✔ SV_TriggerAirstrike (visuel BP_Missile multicast sans owner + dégâts serveur via Lib_Life, souffle 7 m décroissant) |
| Largage de ravitaillement | A | Caisse de munitions/soins pour le groupe | ✔ items |
| Leurre holographique | A | Copie qui attire l'aggro des IA | ✔ QAI perception |
| Mine de proximité | A | Mine posée avec gestion ami/ennemi | ➕ |
| Grappin | A | Traction vers un point d'ancrage | ➕ |
| Mur instantané | A | Barricade dépliable temporaire | ✔ meshes QBuilder |
| Impulsion EMP | A | Désactive drones/véhicules proches quelques secondes | ➕ |
| Fabricateur de munitions | A | Convertit de la Matter en munitions | ✔ Matter + ammo |
| Kit de sabotage | A | Désactive portes/tourelles (interaction longue) | ➕ |
| Réparateur automatique | C | Répare lentement le véhicule piloté | ✔ pattern Repair |

### G. Système & Social (10)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| **Protocole de meneur** (2026-07-11, QMD créé + BRANCHÉ) | P | Débloque le recrutement de civils : escouade max 1/2/3 followers (sans module : verrouillé) | ✔ QAI_CyborgRecruitmentComponent (cap MaxRecruitedFollowers ajouté) |
| **Réseau de recruteur** (2026-07-11, QMD créé + BRANCHÉ) | P | Recruter coûte -20/35/50 % et le maintien passe à 2.5/2/1.5 s | ✔ push façade vers RecruitCost / HoldDurationSeconds |
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
| Antenne longue portée | C | **IMPLÉMENTÉ 2026-07-17** (§9.21) : radio intégrée mains-libres niv 1 + portée de réception ×1.5/×2.0 (stations ET balises de mission), contrôle via la roue d'atouts | ✔ QRadio |
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

## 3. Véhicules (19, par exemplaire, sol et vol)
| Module | T | Effet | Ancrage |
|---|---|---|---|
| Turbocompresseur | P | Vitesse max +8/16/25 % | ✔ |
| Injecteur de boost | A | Boost consommable rechargeable | ? boost existant |
| Réservoir étendu | P | Carburant max +25/50 % | ✔ fuel |
| Économiseur | P | Consommation -15/30 % | ✔ fuel |
| Blindage châssis | P | Vie du véhicule +20/40 % | ✔ |
| Pare-buffle renforcé | P | Dégâts de collision infligés +, subis - | ➕ |
| Suspensions tout-terrain | P | Stabilité hors route + | ✔ QVehicles |
| **Hydroglisseur** (2026-07-11, QMD créé) | P | Roule sur l'eau au-dessus de 70/45/25 km/h, contrôle 50/75/100 % (ralentir = couler) | ➕ physique eau QVehicles |
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
- **Rappel de flotte** [PIL, IC Lab, ACTIF SIGNATURE, ? garage / ✔ arrivée céleste via SpaceshipAI] (ajout RzZz 2026-07-05, GARDER d'office) : appelle n'importe où un véhicule POSSÉDÉ (système de garage existant) qui se LIVRE en autonomie : spawn hors de la vue du joueur, trajet visible sur la carte, arrivée à proximité. Les ships descendent du ciel (le pattern du trafic aérien SpaceshipAI, déjà vivant en jeu) ; les véhicules au sol conduisent jusqu'au joueur. **Progression = PORTÉE de livraison** : Niveau 1 = toute la planète où tu te trouves ; Niveau 2 = l'espace local (hors-planète, orbite) ; Niveau 3 = PARTOUT dans l'univers. À équilibrer : cooldown, éventuel coût d'appel, comportement si le véhicule est détruit en route. Référence d'expérience : le summon de Cyberpunk, poussé à l'échelle interplanétaire.

> Total vague 2 : **25 modules** (catalogue : 136 → 161). Les ✔ sont chiffrables dès le batch M4.5 ; les ? demandent une vérification moteur ciblée ; les ➕ attendent leur mécanique (slots de mort, perforation, marquage).

## 8. TRIAGE RzZz (2026-07-04, export du tableau de bord) : la référence du batch M4.5

- **7 EN JEU** : hors triage, acquis d'office (QMD_* déjà créés).
- **GARDER (82) = BATCH 1 DE PRODUCTION** : l'essentiel du cyborg (Mobilité 6, Survie 10, Combat 10, Furtivité 11, Économie 9, Ingénierie/Déployables 14, Système/Social 17, Pilotage 2) + 3 modules d'ARMES seulement (Amplificateur de dégâts, Recycleur de douilles, Chambre thermique). Quasi toute la vague 2 gameplay est GARDÉE (drone/scanner/inventaire).
- **REPORTER (78)** : TOUT le détail armes (accessoires d'établi : chargeurs, canons, munitions spéciales... 22 entrées), TOUS les véhicules (18, y compris Noyau surcadencé : l'asset de test QMD_NoyauSurcadence reste comme harnais), TOUTE la Liaison & Synergie (8 : attendra le chiffrage de l'adjacence), la vague construction QBuilder (3), et le reste du fond de catalogue.
- **COUPER : 0.** Rien n'est jeté : le catalogue entier reste la matière première.
- **Arbitrages de dédoublonnage appliqués** : Compacteur de matière et Négociateur existaient en double (v1 + vague 2) : une seule définition chacun (l'instance GARDER) ; Spectromètre (v1, gardé) vs Spectromètre de gisements (vague 2, reporté) ; **deux Leurres distincts assumés** : Leurre holographique (déployable, Ingénierie) et **Leurre de drone** (renommé, protection de la vue 3P, tag Module.LeurreDrone) ; Membrane climatique : reportée.

> Génération : script `qmodule_batch_m45.py` (82 défs dans /Game/Phases/QModuleV2, tags Module.* + 15 Stat.* nouveaux + Manufacturer.ICLab/Voss auto-enregistrés dans QModuleTags.ini, StatMods chiffrés pour les effets numériques sûrs, le reste = coquilles à équilibrer dans l'Éditeur de Modules).

## 9. AJOUTS DIRECTS RzZz (2026-07-11) : 3 QMD créés hors triage

Demande du 2026-07-11 (quatre idées) ; les définitions sont créées et validées dans
/Game/Phases/QModuleV2 (registre : 97 définitions), le branchement gameplay reste à faire.

- **Propulseur orbital** [MOB, Cyborg, Passif] : poussée du jetpack dans l'espace +50/125/200 %
  (Stat.Cyborg.Jetpack.SpaceThrustMult, Multiply). Icône 005-rocket. Branchement : hook zéro-G
  dans IS_JetPack (détection espace/apesanteur à identifier : GravityScape / NinjaCharacter).
- **Nid de guêpes** [COM, Cyborg, Actif, rareté 3] : lanceur dorsal de micro-missiles autoguidés.
  1/2/4 missiles (Stat.Cyborg.Missile.Count), recharge 14/10/6 s (Stat.Cyborg.Missile.ReloadSec),
  tracking 60/80/100 % (Stat.Cyborg.Missile.TrackingMult). Icône 019-target. Branchement : acquisition
  de cibles attaquables + réutilisation de l'UI de lock des véhicules + projectile homing ; grosse
  tâche dédiée (AbilityGrant M7 probable).
- **Hydroglisseur** [VEH, Véhicule, Passif] : hydroplane au-dessus de 70/45/25 km/h,
  contrôle sur l'eau 50/75/100 % (Stat.Vehicle.Hydroplane.MinSpeedKmh / ControlMult, Override).
  Icône 018-sea. Le fantasme : l'élan te porte, ralentir te coule. Branchement : physique
  eau/roues côté QVehicles (premier module Domain=Vehicle réellement créé).
- **Demande furtivité sanglines : PAS de nouveau module.** La demande « réduire la détection
  et l'aggro des sanglines » est déjà couverte par **Brouilleur de signature** (Furtivité,
  rayon de détection des IA -15/30/45 %, ancrage ✔ QAI perception). Prochaine étape : le brancher
  dans QAI (multiplicateur par joueur-cible au test d'aggro du StateProcessor, en respectant
  le « ce qui ne doit PAS changer » de QAI_ARCHITECTURE §15).
### 9.1 Escouade de followers (2026-07-11, soir) : 2 QMD créés ET branchés

La feature de recrutement de civils (QAI_CyborgRecruitmentComponent sur le PlayerController,
500 $ / 3 s / 450 cm, server-auth avec remboursement) passe sous le contrôle du Mur :

- **Protocole de meneur** [SYS, Passif, rareté 2] : Stat.Cyborg.Squad.MaxFollowers Override 1/2/3.
  Le niveau du module EST la taille d'escouade ; sans module actif : recrutement verrouillé (0).
  Branchement : nouveau cap additif `MaxRecruitedFollowers` sur le composant QAI (-1 = illimité,
  défaut legacy : le live sans v2 ne change PAS), vérifié dans CanRecruitCyborgInternal via
  GetRecruitedSquadPawns (un follower mort libère sa place).
  **APPEL D'ESCOUADE (le cœur du fantasme, clarifié puis livré le 2026-07-11 soir)** :
  `SummonSquad()` (BlueprintCallable + RPC serveur) spawn les followers manquants autour du
  joueur, N'IMPORTE OÙ, sans civil (anneau ~350 cm, posé au sol le long de l'up LOCAL : gravité
  planétaire). Nom généré + loadout armé + registre QAI + ancrage joueur + trackers HUD (chemin
  différé parallèle au recrutement, sans refund). Un follower détruit après une MORT au combat
  verrouille son slot `SummonCooldownSeconds` (300 s, base archétype, modulable via
  Stat.Cyborg.Squad.SummonCooldownSec). Bouton « APPELER L'ESCOUADE » dans la fiche du module
  (mur), commande dev `qai.Recruit.Summon`. VALIDÉ en PIE : 3 spawns sans civil, 0 abort,
  re-summon refusé escouade pleine. Le recrutement de civils sur la carte est CONSERVÉ
  (même cap partagé). Plus tard : arrivée en véhicule/ship armé (modules dédiés).
- **Réseau de recruteur** [SYS, Passif, rareté 1] : Stat.Cyborg.Squad.RecruitCostMult Multiply
  -20/35/50 % + Stat.Cyborg.Squad.RecruitHoldSec Override 2.5/2/1.5 s.
- Push : ApplySquadRecruitment dans QModule_LegacyFacade (pattern ApplyJetpackHardware) :
  résout le composant par NOM de classe (zéro dépendance de module), bases lues sur l'ARCHÉTYPE,
  écrit MaxRecruitedFollowers / RecruitCost / HoldDurationSeconds côté serveur ET client
  propriétaire (HUD juste). Limite connue : le refus « escouade pleine » côté client dépend de la
  réplication d'ownership des followers ; au pire l'interaction s'affiche et le serveur refuse (aucun débit).
- Plus tard : « Entraînement de choc » (buff vie/dégâts des followers) : demande un hook per-agent
  côté QAI, tâche dédiée.

### 9.2 Famille furtivité : le découpage en trois canaux (RzZz, 2026-07-11 soir)

Clarification : « le Brouilleur, ça pourrait être mieux pour la police : quand tu es recherché,
il brouille un peu ta position ; et pareil pour le Voss, on peut faire un module pour ça ».

- **Masque phéromonal** [FUR, QMD créé + BRANCHÉ + VALIDÉ] : le canal FAUNE. Stat
  Stat.Cyborg.Stealth.DetectionMult (Multiply -15/30/45 %). Implémentation : map par joueur
  dans UQAI_SubSystem (QAI_SetPlayerDetectionMultiplier, écrite au game thread par la façade
  QModule côté serveur, lue par le scan d'acquisition dans la fenêtre fork-join : pas de lock) ;
  le scan gonfle la distance du candidat (DistSq / Mult²) UNIQUEMENT pour les archétypes
  créatures : équivaut à réduire le rayon de détection de la faune contre CE joueur. Les
  planchers restent : à fond, une sangline t'aggro encore vers ~44 m. Validé en PIE :
  push x1.00 -> x0.85 -> x0.70 -> x0.55 aux trois phases.
- **Brouilleur de signature** [FUR, BRANCHÉ + VALIDÉ 2026-07-11 soir] : le canal POLICE.
  Stat.Cyborg.Stealth.PoliceTrackingNoiseCm (Override 7500/15000/30000). Implémentation :
  map par joueur dans UQPoliceSubsystem (QPOLICE_SetPlayerTrackingNoise, poussée par la façade
  serveur par réflexion) + GetJammedPlayerLocation (anneau aléatoire 50..100 % du rayon dans le
  plan local du joueur, planet-safe) injecté aux 4 lectures de position de la TRAQUE : dispatch
  des vagues, renforts périodiques, renforts vaisseaux, resolver naturel par unité. Les
  SpawnLocation explicites (postes fixes) restent exactes ; les unités sur zone réacquièrent
  normalement (le brouilleur trompe le dispatch, pas les yeux). VALIDÉ en PIE : push
  7500 -> 15000 -> 30000 cm reçu et confirmé par le subsystem QPolice aux 3 phases.
- **Module Voss** [à créer, nom à choisir : « Contre-mesures Voss » ?] : même esprit contre les
  Voss. Les Voss n'ont pas de système de traque type wanted : le mécanisme naturel est le canal
  détection QAI étendu PAR FACTION (la map par joueur devient un profil faune/police/Voss et le
  scan choisit le canal selon la faction de l'agent). À valider avant construction.

### 9.3 Propulseur orbital : branché (2026-07-11 nuit)

Le vol jetpack (sol ET espace) passe par le composant DynamicFlightComponent (marketplace
adapté) ; sa fonction pure `Get_FlySpeed` calcule la vitesse (Select hover/fast x intensité).
Branchement pur-node dans CE graph : la sortie du Clamp d'intensité est multipliée par un
facteur spatial = SelectFloat(gravité locale < 0.05 ? QMOD_GetStatForObject(SpaceThrustMult,
base 1.0) : 1.0). Le test d'espace lit `GravityAreaComponent.CachedGravityScale` (la même
source que la macro IsAtSpace du jetpack). Au sol : facteur 1.0, zéro changement. En espace :
x1.5 / x2.25 / x3.0 selon la phase. VALIDÉ en PIE : agrégat final=1.500 (phase 1) puis 3.000
(phase 3) ; compile 0 erreur. PIÈGE outillage : la copie backup d'un composant BP
self-referencing ne compile pas (déjà vu sur InventoryComponent) : T3D exporté puis copie
supprimée ; le vrai filet = auto-backup CodeScape pré-write + journal d'édition pin par pin.
Reste à goûter en orbite (RzZz) : la sensation de vitesse réelle.

### 9.4 Lot passifs (2026-07-11 nuit) : 5 branchés + 3 reportés avec preuve

Décisions RzZz : bonus de dégâts « large » (recherchés + pirates/Voss ; machines = drones,
tourelles, robots) ; loot « ambitieux » (upgrade d'un cran de rareté, tous tirages) ; Géologue
= quantité (pas vitesse). Recon par agents (loot/minage, wanted/dégâts, saut/stamina/bruit).

- BRANCHÉS + VALIDÉS PIE : Vérins de saut (JumpZ 730→894 au niveau 3, sqrt exact), Décodeur QPD
  (push x1.25 reçu par QPolice), Géologue (agrégat 1.150), Chasseur de primes (1.100),
  Fléau des machines (1.100). Les deux bonus de dégâts s'insèrent en un seul multiplicateur
  post-armure dans Lib_Life.ApplyStatDamageToActor (chaîne vérifiée pin par pin avant save),
  et s'appliquent aux deux voies (bouclier et physique).
- REPORTÉS avec preuve : Sprint de fond (aucun système de stamina), Pas feutrés (QAI n'a aucune
  ouïe : vue/distance/LOS), Fortune de guerre (stat chiffrée mais AUCUN point de tirage de
  rareté dans le pipeline loot BP : la rareté vient bakée de la persistance ; consommateur à
  créer dans une passe loot dédiée : OnDeathAI des IA / conteneurs).
- Nouveau helper C++ réutilisable : UQAI_FunctionLibrary::QAI_IsMachineActor (archétypes
  mécaniques + tag acteur « QMachine » pour les tourelles/props posés à la main).

### 9.5 Contre-mesures Voss + Frappe aérienne (2026-07-11, fin de nuit)

- **Contre-mesures Voss** [FUR, QMD créé + BRANCHÉ + VALIDÉ x0.85/0.70/0.55] : le 3e canal furtivité.
  2e map par joueur dans UQAI_SubSystem (QAI_SetPlayerVossDetectionMultiplier) ; le scan
  d'acquisition résout la faction de l'AGENT une fois par scan et applique le canal Voss aux
  agents Voss (créatures = canal faune, police = brouilleur de position). Push façade étendu
  (Stat.Cyborg.Stealth.VossDetectionMult).
- **Balise de frappe** [A, BRANCHÉE + VALIDÉE PIE] : premier module ACTIF du jeu, en attendant
  l'infra M7. Design RzZz : désignation AU POINT VISÉ, paliers « rare mais dévastateur »
  (8/14/22 missiles, 10/14/18 m, 300/240/180 s). Implémentation : SV_TriggerAirstrike sur le
  RackComponent (valide niveau module, portée 250 m, cooldown serveur) ; barrage étalé sur 3 s
  après 3 s d'arrivée ; VISUEL = BP_Missile du lance-roquettes spawné par multicast SANS owner
  (le projectile du jeu est client-cosmétique et ne demande des dégâts que si son owner est
  contrôlé localement : sans owner, jamais de double dégât) ; DÉGÂTS = autoritaires serveur,
  souffle du missile répliqué (700 cm, 300 dégâts, décrue 100→10 %) appliqué par RÉFLEXION via
  Lib_Life.ApplyStatDamageToActor (params construits par réflexion : les pins real BP sont des
  doubles, un layout C++ codé en dur corromprait l'appel) : armure et bonus Chasseur/Fléau
  s'appliquent donc aussi aux frappes. Bouton « FRAPPE AERIENNE (POINT VISE) » sur la fiche du
  module + cmd dev qmodule.Test.Airstrike (fallback sol devant le joueur si visée ciel).
  VALIDÉ : inbound 8 missiles zone 10 m, 8 impacts, cooldown refusé à 273 s restants.
  RESTE à goûter (RzZz) : le VISUEL de la pluie de missiles (multicast à voir en jeu), et
  plus tard : version lance-grenades (« nid d'abeille grenades ») + module combiné 2+2.

### 9.6 Ordnances dorsales style Anthem (2026-07-11, seconde nuit)

Référence assumée : les ordnances d'épaule d'Anthem (Ranger). Les DEUX branchées dans la même
passe (le second réutilise le pipeline du premier), validées en PIE :

- **Nid de guêpes** : salve de micro-missiles courte portée (60 m) tirés du dos vers la cible
  AU RÉTICULE. Tracking = chance de toucher par missile (60/80/100 %) : un raté décroche
  visiblement et se perd (cosmétique pur). Touche = ~100 dégâts + mini-souffle 2.5 m via
  Lib_Life (armure/bonus s'appliquent). Visuel : BP_Missile du lance-roquettes à l'échelle 0.5
  avec HOMING RÉEL (HomingTarget = racine de la cible, posé par réflexion au spawn deferred,
  TotalDamage forcé à 0 : jamais de double dégât). Validé : phase 1 = 1 missile 60 % (raté
  observé, comme conçu), phase 3 = 4 missiles 100 % = 4 impacts.
- **Nid de frelons** (NOUVEAU, Combat, rareté 3) : 1/2/4 grenades lobées sur le point visé
  (50 m), recharge 12/9/6 s, profil de la vraie grenade (70 dégâts, souffle 6 m, décrue).
  Visuel : BP_GrenadeProjectile balistique (arc +35 % vers le haut), dégâts serveur au point.
- Mécanique commune : SV_ sur le RackComponent (validation niveau/portée/recharge serveur),
  visuels multicast SANS owner, dégâts par réflexion Lib_Life (helper partagé
  ApplySplashThroughLibLife : cible directe plein pot + voisins en décrue).
- PACING (retour RzZz, appliqué + validé) : rien ne part d'un coup : départs étalés
  (0.22 s/missile, 0.3 s/grenade, épaule résolue au moment du tir), et les projectiles sont
  RALENTIS pour le plaisir de les regarder (CustomTimeDilation : missiles 0.5, grenades 0.7,
  pluie de la frappe aérienne 0.75 avec timer d'impact recalé 1.5 s) : le slow-mo est uniforme
  (fusée + traînée + FX ensemble), zéro modification du pack marketplace.
- SUITE VALIDÉE RzZz : module COMBINÉ 2 missiles + 2 grenades (une fois les deux éprouvés en jeu).

### 9.7 Fix dégâts des ordnances + Roue d'atouts HUD (2026-07-11, tard)

- **BUG DÉGÂTS TROUVÉ ET CORRIGÉ** (rapport RzZz « pas l'impression que ça fasse des dégâts ») :
  les fonctions d'une Blueprint Function LIBRARY portent un paramètre CACHÉ `__WorldContext`
  que le compilateur BP remplit à chaque appel normal. Nos appels par réflexion
  (frappe aérienne, ordnances) le laissaient null → le « Is Server » interne de
  Lib_Life.ApplyStatDamageToActor échouait en silence → zéro dégât. Fix : le paramètre est
  fourni (pawn de l'instigateur), preuve loggée (dump des params réels :
  ActorTarget DamageCauser Damage DamageType InstigatorController __WorldContext), et le
  dégât de la frappe passe désormais par le MÊME helper que les ordnances (un seul point de vérité).
- **DOCK D'ATOUTS AU HUD, v2** (après rejet du v1 par RzZz : « ça n'utilise pas nos assets UI ») :
  un DOCK PERMANENT en bas à droite de l'écran (ancres 1,1 ; marges 42/148 px, alignées aux
  barres HUD existantes) affiche les modules déclenchables actifs du mur
  (Balise de frappe, Nid de guêpes, Nid de frelons, Protocole de meneur/appel d'escouade)
  avec les VRAIES cellules hexagonales du menu Modules : `W_QModuleV2_Cell` (classe lue via
  `GadgetHexCellClass`, couleur famille via `FamilyColorsByTagName`, police BlenderPro),
  cellule sélectionnée en ambre, cooldown en texte sous la cellule (PRET / Ns) via les
  ready-times RÉPLIQUÉS owner-only sur le rack (horloge serveur du GameState).
  MAINTENIR la touche roue agrandit le dock (pick mode), la souris pointe, RELÂCHER
  SÉLECTIONNE (pas de tir accidentel) ; la touche tir DÉCLENCHE l'atout sélectionné sur la
  visée du moment. Implémentation 100 % C++ (UQModule_GadgetHUD : composant driver + widget
  dock natif), attachée au PlayerController local par la façade au premier refresh du rack.

### 9.8 Touches du dock : audit des presets d'input (2026-07-12)

- **Le défaut H était une FAUTE** (rapport RzZz : H = Scanner). Audit complet des 143
  `InputPreset_DA` de `/Game/Inputs` (dump `DefaultInputsCombos` par script éditeur, liste
  intégrale des touches par preset) : en contexte À PIED sont pris A/Q/W/Z/S/D/E/C/F/G/H/J/K/
  L/M/N/U/V/B/Y/T(véhicule)/1-5, Tab, Shift, Ctrl, Space, Alt, etc.
- **Nouveaux défauts VÉRIFIÉS LIBRES à pied** : `GadgetWheelKey = T` (pris seulement en
  véhicule : drift, et en builder : snap) et `GadgetFireKey = X` (pris seulement en
  vaisseau : aim mode, et en builder : remove). Pour neutraliser ces conflits hors-pied,
  le composant HUD IGNORE les deux touches quand le pawn possédé n'est pas un `ACharacter`
  (gate `IsOnFootPawn`) : en véhicule/vaisseau les touches du dock sont mortes.
- Validé en PIE : `Gadget dock ready on 'QangaPlayerController_C_0' (pick 'T', fire 'X')`.
- Touches toujours provisoires en config ini : passeront sur le système de presets d'input
  du jeu dans une passe dédiée (rebind joueur). Le conflit résiduel « mode builder » est
  accepté d'ici là (ouvrir le dock est inoffensif et le builder a son propre contexte).

### 9.9 Fix : le dock v2 ne s'affichait pas du tout (2026-07-12)

- **Bug rapporté** : rien à l'écran (ni dock permanent, ni au maintien de la touche roue)
  depuis la refonte graphique v2.
- **Cause** : `QMOD_RebuildDock` assignait `WidgetTree->RootWidget` à chaque rebuild, mais
  `EnsureDock` fait `CreateWidget` puis `AddToViewport` AVANT le premier rebuild : l'arbre
  Slate est figé au `AddToViewport` avec un root null, et toute assignation ultérieure du
  RootWidget est silencieusement ignorée : le widget rendait un spacer vide en permanence.
- **Fix** : le `UCanvasPanel` root est construit UNE FOIS dans `NativeOnInitialized()`
  (avant la construction Slate) ; le rebuild fait `ClearChildren()` + repeuplement. En prime
  le dock est `HitTestInvisible` hors pick mode (aucun clic du jeu ne peut être avalé par
  les cellules) et un log de preuve compte les cellules réellement construites
  (`Gadget dock rebuilt: N cell(s)`).
- Validé en PIE : `rebuilt: 1 cell(s)` avec la Balise de frappe active. Rappel testeur : sur
  un mur vierge, AUCUN gadget ne s'active tant que le noyau (0,0) est niveau 0 (capacité
  géographique par anneaux) : phaser le noyau d'abord.

### 9.10 Roue d'atouts v3 : popup directionnelle (2026-07-12, retours RzZz sur le dock v2)

- **Retours** : pas de dock permanent (raccourcis toujours visibles refusés), collision avec
  le HUD bas-droite, pas de curseur souris ni de clic ; il veut MAINTENIR la touche et
  STEERER la sélection au mouvement de souris (weapon wheel type Anthem).
- **v3** : RIEN à l'écran hors maintien. Maintenir la touche roue : popup d'une ROUE
  d'hexagones (cellules du menu Modules) en CERCLE autour du centre écran (rayon 150 px,
  1re cellule à 12h, sens horaire), la caméra est GELÉE (`SetIgnoreLookInput`, équilibré au
  release + sécurité EndPlay), et le DELTA souris cumulé (sensibilité 2.0, deadzone 24 px,
  clamp 140 px pour réagir instantanément aux changements de direction) pointe l'hex :
  surbrillance ambre + scale 1.15 + NOM du module au centre (BlenderPro ambre). Relâcher
  commit la sélection et referme. Aucun curseur, aucun clic (widget HitTestInvisible en
  permanence). Le tir reste sur la touche tir (visée du réticule au moment du tir).
- Implémentation : le composant tick UNIQUEMENT roue ouverte (`bStartWithTickEnabled=false`)
  et lit `GetInputMouseDelta` ; le widget expose Rebuild/Show/Hide/UpdatePointer/
  GetHighlighted ; la roue se reconstruit à chaque ouverture (état frais des sockets).

### 9.11 Roue v4 nid d'abeille + vol Anthem des missiles dorsaux (2026-07-12, dessin RzZz)

- **Roue v4** : le cercle plein écran est remplacé par un CLUSTER NID D'ABEILLE compact,
  la formule de tuilage EXACTE du mur (flat-top : X = 1.5·HexSize·Q,
  Y = √3·HexSize·(R + Q/2), cellules carrées 2·HexSize ; HexSize 30 → cellules 60 px),
  slots fixes en losange pour 4 gadgets (haut, épaules gauche/droite, bas), ancré au coin
  bas-droit (centre du cluster à -170/-290), la zone dessinée par RzZz. Steering souris :
  direction = position de la cellule par rapport au centroïde (4 directions cardinales sur
  le losange). Cooldown SUPERPOSÉ au centre de l'hex (les rangées se touchent), nom du
  module sous le cluster.
- **Missiles dorsaux, vol Anthem** : découverte : le pack BallisticsVFX pilote le vol par
  DataAsset (`MissleDA` sur `MissileMovementComponent_C`) : InitialSpeed/TargetSpeed en
  km/h (facteur 0.036), `Acceleration` cm/s² (montée progressive), `HolmingForce` deg/s
  (LA courbe), `WaveOffset` (serpentin), `ExplodeNearHoming`. Le stock (armes) vole à
  1500 km/h avec accélération instantanée : d'où le « tout droit très vite ». Nouveau
  `DA_QModule_ShoulderMissile` (15 km/h départ, 190 en approche, accel 2600 cm/s²,
  virage 150°/s, serpentin 1.5, explosion à 220 cm) injecté PAR INSTANCE après
  FinishSpawning (+ re-seed des valeurs cachées au BeginPlay : vitesse initiale, wave,
  distance d'explosion) ; le DA stock des armes reste intact. Éjection vers le haut et
  l'épaule (alternée) : le homing recourbe vers la cible ; tir au sol sans acteur : ancre
  `ATargetPoint` transient comme cible de homing. CustomTimeDilation supprimée (vraie
  lenteur, trail à vitesse normale), MaxLifeSpan 6 s, timer de dégâts serveur = temps de
  vol quadratique (v0 417 cm/s, a 2600) + 0.35 s, clamp [0.8, 4].
- Validé en PIE : salve de 4, dégâts à 1.6-2.3 s (formule du vol), 0 erreur. La courbe
  et le nid d'abeille se valident à l'œil (RzZz).

### 9.12 Polish roue + lock multi-cibles des missiles dorsaux (2026-07-12)

- **Réactivité roue** : le pointeur de steering a une mémoire courte (décroissance
  exponentielle ~0.15 s, deadzone 14 px, clamp 120 px, sensibilité 3.0) : un renversement
  de geste change la sélection instantanément.
- **Son de switch** : `/Game/Widget/SFX/InterfaceClickselect` (LE clic-sélection canonique
  de l'UI, celui des PresetButton_*) joué à chaque changement d'hex (jamais à l'ouverture) ;
  ini-configurable (`GadgetSwitchSound`).
- **Lock multi-cibles (UI du lance-roquettes réutilisée)** : quand le Nid de guêpes est
  l'atout sélectionné, une boucle client (tick 0.12 s) scanne les pawns IA (overlap
  ECC_Pawn 60 m, cône caméra 35°, test d'occlusion) et pose jusqu'à N marqueurs (N =
  missiles du module) sur des cibles DISTINCTES via le framework Tracker du jeu :
  `Lib_Tracker.AddSingleTempTracker` par réflexion (paramètre caché `__WorldContext`
  fourni), `TK_InactiveTarget` = carré VERT à l'acquisition puis swap
  `TK_VehicleWeaponTarget` = carré ROUGE après 0.45 s. Filets : lifetime 1.2 s rafraîchie
  à chaque tick (un marqueur perdu expire seul), `RemoveTracker` réflexif sinon destroy.
  Au tir : la liste verrouillée part au serveur qui re-valide (portée, vie) et distribue
  les missiles en round-robin : cibles distinctes s'il y a assez d'IA, partage sinon
  (`SV_TriggerShoulderMissiles` prend désormais un `TArray<AActor*>`).
- Limites connues : les followers de l'escouade sont lockables (pas de filtre allié en v1) ;
  les cibles véhicules/vaisseaux (hors canal Pawn) pas encore scannées.
- Test : `qmodule.Test.SelectGadget Module.NidDeGuepes` force la sélection sans la roue
  (boucle de lock testable à distance). Validé PIE : lock loop ~100 ticks sans erreur,
  salve `4 queued on 0 locked target(s)` + dégâts. Marqueurs à l'écran, son et ressenti
  steering : validation visuelle RzZz (map de test sans IA pawn dans le cône).

### 9.13 Touches rebindables + filtre allié du lock (2026-07-12)

- **Touches du dock intégrées au système d'input du jeu** : 2 nouveaux
  `InputPreset_DA` : `/Game/Inputs/Character/Char_GadgetWheel` (T) et `Char_GadgetFire`
  (X), catégorie Character, mêmes touches sur les 2 layouts AZERTY/QWERTY : ils
  apparaissent AUTOMATIQUEMENT dans le menu de rebind du joueur (pattern QRadio validé).
  Le composant HUD se bind sur `InputsComponent.GetCurrentInputData(preset)` →
  `OnInputEvent(bool)` (dépendance module InputSystem ajoutée à QModule) ; les raw
  KeyBindings ini (`GadgetWheelKey`/`GadgetFireKey`) ne servent plus que de FALLBACK si
  les presets manquent. Bonus : les touches respectent maintenant les
  activations/désactivations d'input du jeu (menus), ce que le raw binding ignorait.
  Validé PIE : `Gadget wheel ready (rebindable presets bound: wheel + fire)`.
- **Filtre allié du lock** : les followers du Protocole de meneur ne sont plus
  verrouillables par leur propre leader (lecture réflexive de
  `QAI_CyborgRecruitmentComponent.GetRecruitedCyborgSquad`, sans dépendance QAI).
  Restent hors scan v1 : les véhicules/vaisseaux ennemis (canal Pawn uniquement).

### 9.14 Map de dev unique + GodWall + portée missiles (2026-07-12)

- **Map de travail unique** (consigne RzZz) : `/Game/Maps/LevelDev/L_Dev_Claude` : tous
  les PIE/tests s'y font désormais.
- **`qmodule.Test.GodWall`** : outil dev : maxe le noyau puis installe TOUS les modules
  cyborg du registre au niveau max, placement en spirale. Validé : 88 installés, 0 refus,
  93 sockets, tous les branchements poussés au max. Piège re-confirmé : les commandes
  console du bridge reçoivent le monde ÉDITEUR : toujours passer par `ResolveGameWorld`.
- **Portée des missiles dorsaux : 60 m → 150 m** (`ShoulderMissileRangeCm`, ini-réglable) :
  appliquée au cône de lock client, à la re-validation serveur et au trace de tir. Le vol
  courbé couvre 150 m en ~3.2 s (dans les clamps du timer de dégâts).

### 9.15 Surcouche de bouclier + Caisson hermétique branchés (2026-07-12, design RzZz)

- **Surcouche de bouclier** (50/100/150 pts + régén hors combat) : le stat `SS_Shield`
  du jeu (complet, UI prête, jamais accordé : ancre de l'audit) est GRANTÉ dynamiquement
  par la façade quand le module est actif : `StatsComponent.AddNewStat(SS_Shield_C)` par
  réflexion sur le pawn (serveur), `MaxShield` piloté par `Stat.Cyborg.Shield.Max`
  (écrit serveur + client owner : la barre HUD lit en local), `CurrentShield` serveur
  uniquement (répliqué). Données recalées : Add 50/100/150 (l'ancien Multiply était mort
  sur base 0, le piège Nash). Régén : timer serveur 1 s sur le rack mur
  (`Authority_TickShieldRegen`) : toute BAISSE du bouclier relance le délai de calme
  (`ShieldRegenDelaySeconds`, 12 s ini-réglable) puis +10 %/s jusqu'au max. La barre
  vie+bouclier du HUD (`W_LifeShieldBar`) s'affiche seule dès que le stat existe.
  Validé PIE : `Shield overlay: GRANTED, 150 shield on 'ALS_Base_CharacterBP_C_0'`.
- **Caisson hermétique** (33/66/100 % du sac conservé à la mort) : édit pur-node de
  `ALS_Base_CharacterBP.DropAllItemsDeath` (LA copie du joueur : `DA_DefaultPawn`) :
  dans la boucle de drop finale (qui vide `ListToDrop` : sac + équipés), un Branch
  `RandomFloatInRange(0,1) < QMOD_GetStatForObject(self, Stat.Cyborg.DeathSafe.KeepFraction, 0.0)`
  saute le `SV_DropItem` de l'item conservé. Neutre pour les IA PAR CONSTRUCTION : sans
  mur de modules la fraction vaut 0 (base), le random n'est jamais inférieur : drop
  inchangé (les hiérarchies AI_* héritent de la même fonction sans effet). Données :
  Override 0.33/0.66/1.0, MaxLevel recalé 5→3. Compile clean, connexions vérifiées.
- **La persistance du mur FONCTIONNE sur L_Dev_Claude** : le mur GodWall complet de la
  session précédente est revenu au PIE suivant (le GodWall relancé dit « 0 installed,
  93 sockets »). Restent à valider à l'œil (RzZz) : la barre de bouclier au HUD,
  l'absorption des dégâts, la régén après 12 s de calme, et la mort avec sac conservé.

### 9.16 Fix : warnings « Accessed None InstigatorController » + trou du resolver (2026-07-12)

- **Symptôme** (rapport RzZz après le test bouclier) : 4 erreurs runtime
  `Accessed None... InstigatorController, Node: SetShield` dans Lib_Life. Cause : le
  node `Get Controlled Pawn` de la chaîne bonus VsOutlaw/VsMachine s'exécutait sur un
  InstigatorController None (dégâts de chute/environnement) : la branche bouclier,
  nouvelle, tirait cette chaîne pour la première fois.
- **Découverte plus grave en corrigeant** : `QMOD_GetStatForObject` ne résolvait ni un
  Context qui EST un acteur (GetTypedOuter ne retourne jamais self) ni un controller :
  les facteurs Chasseur de primes / Fléau des machines étaient donc en passthrough
  silencieux (neutres) depuis leur branchement.
- **Fix** : resolver réparé (acteur direct, chaîne d'outers, et les controllers résolvent
  leur pawn possédé, null-safe) ; dans Lib_Life le node fragile est SUPPRIMÉ et
  l'InstigatorController branché directement sur les deux `QMOD_GetStatForObject`.
  Validé : PIE sans aucune erreur, bouclier opérationnel. Leçon : dans une chaîne pure
  insérée par édit, ne jamais utiliser un method-call d'instance (crash sur None) : que
  des fonctions C++ null-safe. À vérifier au prochain cycle : le Context passé par le
  Recycler (Mining.YieldMult, Géologue) : si c'est un DataAsset, encore neutre.

### 9.17 Passe de nuit autonome (2026-07-12, RzZz au lit) : 4 modules cyborg de plus

- **Vérification Géologue** : le Context du Recycler est le PAWN mineur (set
  `PawnsTryingRecycle`) : c'était donc le trou « acteur direct » du resolver : le fix
  9.16 répare AUSSI le rendement minage automatiquement. Aucun édit.
- **Audit data des 102 QMD** : dump complet tag/op/valeurs des modules cyborg. Découvertes :
  `BlindageSousCutane` (Armor.Flat, lu par Lib_Life sur la victime) et
  `ServomoteursDeJambes` (Move.SprintSpeed, chaîne QMOD déjà dans
  UpdateDynamicMovementSettings avec garde-fou > 2.5 → 1.0) étaient DÉJÀ branchés et
  actifs. `CompacteurDeMatiere` (StackMult), `RegulateurAtmospherique` (conso jetpack),
  `NanoReparateurDeDrone`/`BlindageDeDrone` (système drone) : reportés (ancres à
  auditer/système à part). Les ~50 modules NO-MODS = mécaniques nouvelles : design RzZz requis.
- **NanoRegenerateur BRANCHÉ** (façade) : `RegenPerSec` de l'instance SS_PhysicalState
  écrit par réflexion (base archétype 1 HP/s + Add 1/2/3 → 2/3/4 HP/s). Validé PIE :
  `Health regen: 4.0 HP/s (base 1.0)`.
- **SacDigitiqueEtendu BRANCHÉ** (façade) : `InventoryBaseSize` de l'InventoryComponent
  (base archétype + Add 4/8/12) puis `UpdateInventorySize()`. Validé PIE :
  `Inventory extension: base size 0 -> 12`.
- **AmortisseursCinetiques BRANCHÉ + BUG PRÉEXISTANT RÉPARÉ** : la chaîne de réduction
  QMOD (FallDamage.Reduction /100, 1-x, multiply) existait déjà dans l'EventGraph mais
  était INERTE : le `Target` du QMOD Get Stat n'était pas connecté (réduction toujours 0)
  ET SURTOUT l'exec de `SV_OnLanded` était COUPÉ : **les dégâts de chute ne s'appliquaient
  plus du tout dans le jeu** (rupture du type « handler effacé »). Réparé : exec
  rebranché (`SV_OnLanded.then → Branch.execute`), Target ← Self, et un Clamp 0..1 sur la
  fraction (sans lui, une réduction > 100 SOIGNERAIT en tombant). Compile clean.
  ⚠️ DÉCISION À VALIDER (RzZz) : cette réparation RÉACTIVE les dégâts de chute pour tous
  les joueurs (20..80 selon la vitesse) : c'était peut-être cassé depuis un moment.

### 9.18 AUDIT COMPLET pré-continuation (2026-07-16, demande RzZz « tout est réellement bon ? »)

Deux audits adversariaux (C++ multi/correctness/perf + anti-fausses-features data↔code)
plus vérifications pin-à-pin. Résultats et actions :

- **Verdict multi C++** : aucune faille critique sur serveur dédié. RPC conformes
  (Server Reliable état / Multicast Unreliable cosmétique), validations serveur complètes
  (autorité + possession + portée + cooldown), item-racks protégés contre les clients
  malveillants (PawnOwnsInstance), bases archétype anti-cumul, réflexion défensive.
- **4 défauts corrigés dans la foulée (compilés)** :
  1. Les pushes de stats de la façade tournaient sur TOUS les clients à chaque OnRep
     (simulated proxies = O(N²) à 500 joueurs) → gate `HasAuthority || IsLocallyControlled`
     (les proxies gardent seulement le refresh cosmétique CallPhaseUpdate).
  2. Le missile COSMÉTIQUE de la frappe aérienne gardait sa charge de dégâts → en serveur
     d'ÉCOUTE le spawn a autorité = double-dégât possible → TotalDamage/DamageRadius
     neutralisés comme les ordnances d'épaule (namespace helpers déplacé en amont).
  3. Bind LevelUp → crédit de phase : re-bind si l'instance SS_Level liée est recréée
     (sinon les level-ups cessaient de créditer en silence) + re-tentative sur MarkRackDirty.
  4. Durcissements : plafond 16 sur la liste de locks envoyée par le client (anti mini-DoS),
     buffer de splash hissé hors boucle (alloca par acteur), tags de régén en cache statique,
     re-bind du HUD si le rack change (travel/reconnect).
- **Anti-fausses-features : 24 stats ACTIFS vérifiés producteur→consommateur→sémantique.**
  Sprint : le Select « garde-fou » est en réalité un GATE D'ALLURE (Mapped Speed > 2.5 =
  sprint uniquement) : correct, fausse alerte levée (la note 9.17 était fausse sur ce point).
  Drone.ImpactsAdd / Drone.RepairTimeMult / Inventory.StackMult : consommateurs RÉELS
  trouvés dans IS_DroneBase et InventoryComponent (branchés lors de passes antérieures :
  la note « reportés » de 9.17 était périmée) : câblage aval à confirmer à l'occasion.
- **UN vrai fantôme : `Stat.Cyborg.Matter.Max`** (QMD_GeneralSystem +200/400/600) : son seul
  lecteur est le CyborgAdapter jamais instancié (code mort M2). EN JEU la matière suit bien
  le niveau du noyau MAIS par le chemin legacy (PhaseComponent → Select BP) avec les
  VALEURS LEGACY (base 100 → 300 → 500), pas celles du QMD. ⚠️ DÉCISION RZZZ : recaler le
  QMD sur les valeurs réelles (affichage honnête) ou câbler le stat v2 (stage 2) ; et le
  Select legacy n'a que 3 options alors que le noyau v2 monte niveau 3 (comportement de
  l'index 3 à vérifier).
- **PIE des réglages multi** : testé en solo uniquement ; RzZz fait les tests multi réels
  sur une dev branch Steam (les réglages PIE de l'éditeur ont été REMIS EN SOLO).

### 9.19 Cap « compléter le cyborg » + vague A (2026-07-16, directives RzZz)

- **Cap** : compléter TOUS les modules cyborg, puis les véhicules, puis les armes. Les
  modules sont inaccessibles en version publique : itération libre, tests systématiques.
  Ordre des vagues validé : A (branchements rapides), puis D (les DRONES), puis le reste.
- **Matière : GRAND PRÉALABLE** : RzZz refuse toute décision tant que l'écosystème complet
  (disques de matière, recyclage des loots, réparation, marchands...) n'est pas cartographié
  et discuté : « il ne faut pas invalider des propositions de gameplay d'avant ». Une recon
  approfondie est en cours ; le Recycleur optimisé (injecte de la matière) et le recalage
  du noyau ATTENDENT cette discussion.
### 9.20 Chantier matière : étage 2 + Recycleur blindé (2026-07-16, décisions déléguées par RzZz)

- **Vision RzZz gravée** : la matière = jauge vitale universelle (faim/soif/énergie), lore
  de dématérialisation numérique (tout sauf l'organique), disques = refill instantané,
  de plus en plus de puits (soin, réparation drone, conso des modules...) ; les modules
  de capacité de stockage sont désirés. La pression de survie (drain passif, ×2 dégâts à
  sec) NE BOUGE PAS.
- **Matière max : étage 2 avec fallback public, CÂBLÉ** (ALS_Base_CharacterBP, cluster
  On Phase Update) : `SetMaxMatter( IsQModuleEnabled ? Round(QMOD_GetStat(self,
  Stat.Cyborg.Matter.Max, 100)) : Select_legacy )` : nodes purs uniquement, le Select
  legacy 100/300/500 reste le chemin du mode PUBLIC (plugin off), le stat v2 (base 100 +
  Add 200/400/600 = 300/500/700) prend la main quand les modules sont actifs. Ferme le
  bug d'index 3 du Select. Compile clean, sauvé, sanity PIE sans erreur. Valeurs
  identiques aux niveaux 1-2 : zéro changement joueur ; le niveau 3 du noyau ouvre 700.
- **Recycleur optimisé** : data posée (+25/50 %, 2 niveaux, tag Stat.Cyborg.Recycle.YieldMult)
  et helper C++ `QMOD_GetRecycleYieldFactor(RecyclerContext, RecycledItem)` écrit :
  résout le facteur sur le recycleur et retourne TOUJOURS 1.0 pour les disques de matière
  (anti-blanchiment coins→matière), résolution réflexive du DA (ItemDataAsset/ItemData/
  ItemDA, fallback nom). ⚠️ EN ATTENTE : le rebuild C++ est BLOQUÉ par du code non
  compilable d'une AUTRE session (chantier quêtes : CLIScape/Tool_GetQuest.cpp +
  QAutomatedTestSuite/QuestTestSubsystem.cpp, modifiés à 17h32-17h37) : les édits BP des
  2 sites de recyclage (InventoryComponent SV_RecyleItem fan-out ×2 popup inclus +
  ItemScriptBase ServerTryRecycleItem) se feront après compilation du helper.
  Cartographie pin-à-pin des 2 sites archivée en 9.19/agent.
- **Leçon outillage** : les retries aveugles sur le bridge créent des NODES DOUBLONS
  quand la réponse flake (la requête passe souvent quand même) : 4 orphelins créés puis
  purgés proprement (delete_nodes + recompile clean). Sonder avant de re-tenter un
  add_node.
- **RECYCLEUR OPTIMISÉ : BRANCHÉ ET COMPLET** (après déblocage du build quêtes) : le
  helper `QMOD_GetRecycleYieldFactor` est compilé et câblé aux DEUX sites serveur :
  `ItemScriptBase.ServerTryRecycleItem` (chaîne Conv→×facteur→Round entre RecyclableValue
  et AddOrRemoveMatterToActor, item = self) et `InventoryComponent.SV_RecyleItem`
  (même chaîne sur la somme de `CalcRecycleMatterOfItemAndAttachments`, avec les DEUX
  consommateurs re-routés : le crédit ET le popup client `OC Item Recycled` : l'affiché
  = le reçu). Compiles clean, sauvé, sanity PIE sans erreur. Les disques recyclent à
  valeur faciale (exclusion C++). Test en jeu RzZz : recycler un item courant avec le
  module actif : +25/50 % de matière ; recycler un Disk : valeur inchangée.
- **Diagnostic « je ne prends pas de dégâts » (question RzZz)** : conforme au cumul max :
  armure plate 30 annule les armes ≤30 ; le bouclier 150 encaisse ~300 post-armure (le
  pipeline divise les dégâts bouclier par 2) avec régén ; vie 4 HP/s ; chutes nulles ;
  jamais à sec. Leviers d'équilibrage prêts : `GlobalStatClampMaxByTagName` (comme le
  FireRate 0.8). Vérifs : AUCUNE collision de tags entre modules cyborg (les partages
  Weapon.Damage/FireRate sont le design) ; UNE collision d'écriture identifiée :
  `GravityScale` (Coussin) vs l'alignement de gravité planétaire NinjaCharacter (24 refs
  dans le pawn) : le coussin marche sur gravité simple, est écrasé sur planètes : à
  re-brancher DANS le graphe de gravité (pattern sprint) lors d'une prochaine passe.

- **Vague A livrée (hors matière)** :
  - `CoussinGravitationnel` BRANCHÉ : GravityScale natif du CharacterMovement
    (×0.8/0.65/0.5, base archétype, nouveau tag `Stat.Cyborg.Fall.GravityMult`,
    MaxLevel 2→3). Validé PIE : `Gravity cushion: scale x0.65`.
  - `RegulateurAtmospherique` BRANCHÉ : la façade pousse aussi `FuelConsume` du IS_JetPack
    (CDO × facteur `Stat.Cyborg.Flight.FuelUseMult`, ×0.9/0.8/0.7). Validé PIE :
    `Jetpack hardware: level=3 maxfuel=175 consume=0.70`.
  - Confirmations pin-à-pin drone/stack : interrompues par la fermeture éditeur (l'agent
    bridge est mort avec) : à refaire à l'occasion, non bloquant.

### 9.21 Antenne longue portée : radio intégrée du cyborg (2026-07-17, design validé RzZz)

- **Design (3 réponses AskUserQuestion)** : radio intégrée mains-libres dès le niveau 1
  (capte comme une radio portable allumée, sans item) + sensibilité aux niveaux 2-3 ;
  la portée d'accroche accrue vaut AUSSI pour les balises de mission (stations runtime,
  type Ghost Signal) sans indication directionnelle (la recherche au gradient sonore de
  Q027 reste intacte) ; contrôle 100 % roue d'atouts : valider la roue sur le module =
  allumer/couper, touche de tir gadget = allumer puis station suivante à chaque appui.
- **Côté QRadio (additif, neutre par défaut)** : `ReceptionRangeMult` (répliqué, défaut
  1.0) étire `FullCm`/`CutCm` dans `ComputeSignalQuality` (un seul point de code couvre
  catalogue ET balises runtime) ; `bPrivateReception` (répliqué, défaut false) : mode
  implant (le porteur entend en 2D, les autres joueurs RIEN : jamais de World3D),
  contrairement aux radios portables/véhicules qui restent des haut-parleurs ; API
  publique `SetPowered`/`IsPowered`/`StepStation` (délègue à `Server_SetRadioOn` /
  `CycleStation` : même chemin server-authoritative que les touches conducteur).
- **Côté QModule** : `ApplyIntegratedRadio` (LegacyFacade, dans la liste des pushes) :
  création SERVEUR uniquement (le composant `QModule_IntegratedRadio` se réplique),
  recherche par NOM (jamais par classe : un hôte avec sa propre radio reste intact),
  niveau <1 = DestroyComponent, sinon pousse `Stat.Cyborg.Radio.RangeMult` (Multiply,
  base 1.0 : ×1.5/×2.0 attendus aux niveaux 2-3). Dépendance Build.cs QModule→QRadio
  (sans cycle : QRadio ignore QModule). Roue : tag dans `GadgetTags[]`, toggle au commit
  (`HandleWheelReleased`), branche radio en tête de `HandleFirePressed` (aucun tick :
  `UpdateTickMode` inchangé).
- **Tags** : `Module.AntenneLonguePortee` + `Stat.Cyborg.Radio.RangeMult` dans
  `Config/Tags/QModuleTags.ini` (141 tags).
- **Bonus constaté** : les touches radio natives (Next/Prev/Toggle des véhicules) se
  bindent aussi sur la radio intégrée quand les presets sont dans le contexte à pied
  (même mécanisme `BindRadioInputs`, `IsLocalDriver` vrai sur le pawn possédé).
- **Reclassement** : l'entrée catalogue vivait en famille H (Pilotage) ; le module vise
  le cyborg à pied → famille D Furtivité & Information (`Module.Family.Stealth`) au QMD.
- ⚠️ Reste au moment de l'écriture : asset QMD à créer post-rebuild + test PIE.
