# QMODULE : Catalogue de modules (matière première design)

> **Statut : BRAINSTORM STRUCTURÉ, à trier et valider. Rien n'est implémenté.**
> Compagnon de `QMODULE_ARCHITECTURE.md`. Rédigé le 2026-07-04. Objectif : donner une base LARGE (136 entrées) pour que le joueur construise son propre build, façon arbre de talents illimité. Les noms sont des propositions ; les chiffres sont des ordres de grandeur à équilibrer.
>
> **Légende** : T = type d'effet : **P** (passif chiffré, pur data), **A** (actif type stratagème, une classe BP), **C** (comportement, un composant attaché). Ancrage : **✔** s'appuie sur un système vérifié dans le moteur ; **?** probable mais à confirmer ; **➕** nouvelle mécanique à créer.

---

## 0. Règles transverses (le multiplicateur de contenu)

- **Manufactures** : chaque rôle du catalogue peut exister en variantes. IC Lab (référence, stable, équilibré) ; Voss (chiffres supérieurs + contrepartie/instabilité) ; Nash Arms (armes, orienté cadence) ; Surplus QPD (défensif, connoté loi) ; Artisanal pirate (stats tirées aléatoirement, pas cher, marché gris). Une variante = mêmes StatTags, autres valeurs + drawbacks : **zéro code**. La liste réelle est donc bien plus grande que les 136 lignes.
- **Tiers** : 1..3 par défaut (aligné sur l'existant qui plafonne à 2 ou 3) ; prototypes rares 1..5.
- **Rareté** : Commun / Avancé / Prototype / Relique, mappée sur le champ `Rarity` déjà présent sur les instances d'items.
- **Exclusivité** : une seule variante d'un même rôle active par cible (ExclusivityTag).
- **Hermétique à la mort** : tout module/phase INSTALLÉ ; le sac reste lootable.
- **SynergyTags** : chaque module porte des tags de famille pour les effets d'adjacence sur le Mur (concept §13 du doc d'architecture, à valider).

---

## 1. Cyborg

### A. Mobilité (12)
| Module | T | Effet | Ancrage |
|---|---|---|---|
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

### D. Furtivité & Information (10)
| Module | T | Effet | Ancrage |
|---|---|---|---|
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

### F. Ingénierie & Déployables (14)
| Module | T | Effet | Ancrage |
|---|---|---|---|
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
| Cœur IC Lab (Système Général) | P | Capacité du Mur (module de base, non échangeable) | ✔ concept v2 |
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

## 2. Armes (22, par exemplaire, installés à l'établi)
| Module | T | Effet | Ancrage |
|---|---|---|---|
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
