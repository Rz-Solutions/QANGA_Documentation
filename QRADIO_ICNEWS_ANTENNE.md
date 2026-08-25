# I-C News Radio : bible d'antenne et scripts

> Contenu redactionnel pour la station `ICNewsRadio` du catalogue `/Game/Systems/QRadio/DA_QRadio_Stations`.
> Pour le systeme (ajouter une piste, une station, regler la friture) voir **`QRADIO_GUIDE.md`**.
> Regle : tout ce qui est cite ici comme fait du jeu a ete releve dans le contenu reel (localisation, quetes, assets). Les elements de la section 5 sont des propositions, pas des faits.

---

## 1. Ancrages verifies (ce qu'on peut dire a l'antenne)

**Le monde**
- Le monde s'est effondre, l'economie continue. Vocabulaire joueur : "les restes de l'Ancien Monde" (vieux terminaux, pieces de moteur, ferraille). Source : dialogue du Comptoir d'echange (`Game.po`).
- La mort n'est plus definitive : environ **300 Tours de Relais** sur la planete, chacune avec sa salle de recuperation. Si le corps est detruit, la conscience traverse le reseau et se reveille dans un corps neuf a la tour d'ancrage. Source : tutoriel Tona.
- Le joueur est un **Cyborg** reveille par **IClabs** (relais de Djibouti, Afrique). "Bonjour Cyborg." est la facon normale de s'adresser a lui.
- Expression du monde : **"Par le Voile"** (*By the Veil*). A utiliser avec parcimonie, c'est fort.

**Les lieux (noms exacts)**

| Lieu | Nature |
|---|---|
| **la Capitale** | grande ville : le Centre, la Traverse, Centinela, Mass District, Downroad, plage Costa Rive, jardins IClabs, Porte Est / Porte Sud |
| **YellowWall** | la region autour : YellowWall City, Centre de Purification, Quarry Stone (la carriere), SawgeniuS Park, Chambre du Complexe |
| **ICLI Station** (ICLISPACE) | station **orbitale** : vendeurs d'armes, de medicaments, d'armures, station de warp |
| **Djibouti / Dire Dawa** | region de depart : Tour de Relais, centre logistique, Grand Pont |
| **Starkitown** | ville abandonnee |
| **Raffinerie Iron Dee** | raffinerie |
| **usine Rost-Stahl** | complexe industriel a 1 km dans le desert |
| **Tour de Relais 77** | relais radio isole au pied de la montagne |

**Les services d'une Tour de Relais** (le sommaire naturel d'un bulletin) : salle de recuperation, laboratoire de recherche, armurerie, centre logistique (contrats de livraison), comptoir d'echange, station d'aerotram, raffinerie.

**Transports**
- **Aerotram** : navettes autonomes qui relient les tours **par sauts courts**, jamais de vol direct vers une destination lointaine. On change de tour en tour. Les rames ont un code (ex. `HLL817`).
- **Melrose** : le constructeur. Modeles reels : **Sablone** (la moto), **Orizaune** (le vaisseau), Taxi, SUV, Sport, PickUp, PickUp arme, Riper, Coopay, Citizen, Police.
- **Warp** : voyage spatial. Impossible dans l'atmosphere, bloque en cas de collision, consomme de l'energie.

**Economie**
- Monnaie joueur : **Credits**.
- Minerais : **Fer, Obsidienne, Silicium, Cuivre, Aluminium**.
- **Matiere** : ce n'est pas de l'argent, c'est **l'energie du corps cyborg**. On **recycle** un objet pour le convertir en Matiere. Reservoir de Matiere. Le jetpack en consomme.
- **Contrats** de livraison au centre logistique, biomasse de Sangline revendue au laboratoire.

**Les menaces**
- **Sanglines** : rapides, en nids, variantes Climber et Flyer. Feminin : une Sangline, des Sanglines.
- **Humains Infectes**.
- **Cyborgs-Pirates**, charognards.
- Zones dites **sauvages** entre les tours.

**Les marques du monde** (affiches et ecrans deja en jeu, `Content/Resources/Pubs/`) :
IClabs, ICLIspace, Titanium ICLI, Iron Dee, Melrose, I-C News Radio, IC Police, CineVortex, Cryoday, LoopLife, Luna, Sola, Komet, Starkito, Vieego, McFrailenergy, CoperMuter, Sboutique, A.D Store, Hardware Store, Chez Fred, Costa Riv, SwageniusPark.

---

## 2. Ligne editoriale

**Ce qu'est la station.** Une radio civile et corporate qui parle par-dessus une planete cassee. Le ton n'est pas post-apocalyptique deprime : c'est un presentateur enjoue qui annonce le trafic et les cours des minerais comme si tout allait bien, alors qu'une partie de ses auditeurs vient de se faire devorer. C'est le decalage qui fait le sel.

**A qui il parle.** A des cyborgs qui rentrent de zone sauvage, a des convoyeurs sous contrat, a des mineurs, a des pilotes. Jamais a "des survivants" : ce mot n'existe pas dans le jeu.

**Les cinq rubriques recurrentes**
1. **Ouverture / fermeture de cycle** ("cycle" est le mot du jeu pour la journee, cf. quete *First Cycle*).
2. **Bulletin de zone** : ou les Sanglines ont ete vues, quelle route eviter.
3. **Logistique** : contrats disponibles, retards, aerotram, saturation d'une tour.
4. **Cours des matieres** : Fer, Obsidienne, Silicium, Cuivre, Aluminium.
5. **Publicite** : une des marques ci-dessus.

**La blague signature de la station** : la mort n'est pas grave, seule la cargaison l'est. Le presentateur traite les reveils en salle de recuperation comme un desagrement administratif.

**Interdits d'antenne**
- Pas de tiret cadratin dans les textes.
- Pas de "survivants", "apocalypse", "zombies", "la Terre est morte".
- Pas de dome sur YellowWall, pas de baie d'amarrage a YellowWall : YellowWall est **au sol**, les amarrages sont a **ICLI Station** (orbitale).
- Ne pas melanger Credits et Matiere comme si c'etait la meme chose.
- Pas de vol aerotram direct longue distance : le reseau fonctionne par sauts.

---

## 3. Scripts prets a generer

### SCRIPT 1 : Ouverture de cycle (Capitale / YellowWall)

```
[Spoken Intro: Dynamic and natural French Radio Host]
(Jingle dynamique I-C News Radio)
Ici I-C News Radio. Nouveau cycle sur la Capitale, la lumiere remonte le long du Mur Jaune et la Traverse est deja en train de saturer.
Une pensee pour ceux qui finissent leur quart a la carriere de Quarry Stone, et pour ceux qui rentrent de zone sauvage la soute pleine. Vous avez tenu la nuit, c'est deja beaucoup.
Et si vous avez ouvert les yeux ce matin dans une salle de recuperation plutot que dans votre lit : le corps est neuf, la cargaison, elle, ne revient pas toute seule.
On pousse les moteurs, c'est parti.

[Instrumental Drop: Space Synth-Rock]
[Long Instrumental Break: driving electric guitar, fast cyberpunk synths, energetic beat]
(Music playing intensely)
[End of Instrumental]

[Music fades down into background]
[Spoken Interlude: Dynamic and natural French Radio Host]
(Bruitage de communication radio)
Point logistique avant de reprendre. Le centre logistique de Dire Dawa signale un retard sur les contrats du matin : deux convois n'ont pas confirme leur arrivee au sud du Grand Pont. Si vous partez dans cette direction, verifiez vos munitions avant de verifier celles des autres.
Cote raffinerie, les cours restent stables : le Fer tient, l'Obsidienne monte doucement, le Silicium ne bouge pas. Bon moment pour vider vos soutes a Iron Dee.
Allez, on garde le rythme.

[Instrumental Drop: Upbeat Cyberpunk]
[Long Instrumental Break: punchy bass, fast tempo, energetic retro-futuristic synths]
(Music playing intensely)
[End of Instrumental]

[Music fades down]
[Spoken Outro: French Radio Host]
C'etait I-C News Radio. Bon repos a ceux qui rentrent, bonne chance a ceux qui sortent.
Et pensez a ancrer votre signature avant de franchir la Porte Sud. On n'est jamais trop prudent.
(Radio static fade out)
```

### SCRIPT 2 : Bulletin de zone

```
[Spoken Intro: Dynamic and natural French Radio Host]
(Jingle dynamique I-C News Radio)
I-C News Radio, bulletin de zone.
Trois signalements cette nuit. Un nid confirme a l'ouest de l'usine Rost-Stahl : les equipes de recuperation attendront le prochain cycle, et honnetement, elles ont raison.
Des Sanglines volantes ont ete vues au-dessus de la route du Centre de Purification. Si vous entendez frotter au-dessus de la cabine, ce n'est pas le vent.
Et le relais radio soixante-dix-sept est toujours muet. Le silence dans le desert n'a jamais voulu dire le calme.
On enchaine, ca ira mieux en musique.

[Instrumental Drop: Space Synth-Rock]
[Long Instrumental Break: heavy guitars, industrial percussion, aggressive synth lead]
(Music playing intensely)
[End of Instrumental]

[Music fades down into background]
[Spoken Interlude: Dynamic and natural French Radio Host]
(Bruitage de communication radio)
Message du laboratoire de recherche, et ils insistent pour qu'on le repasse : ils rachetent la biomasse fraiche. Tissus, membres, fragments, tout se paie en Credits selon la viabilite.
Donc oui : si vous en descendez une, prenez le temps de la ramasser. C'est la seule fois de votre vie ou on vous paiera pour du travail bien fait et pour les restes.
Rappel de la maison : votre Matiere n'est pas de l'argent. Gardez le reservoir plein, le jetpack ne pardonne pas.
On repart.

[Instrumental Drop: Upbeat Cyberpunk]
[Long Instrumental Break: punchy bass, fast tempo, energetic retro-futuristic synths]
(Music playing intensely)
[End of Instrumental]

[Music fades down]
[Spoken Outro: French Radio Host]
C'etait le bulletin de zone d'I-C News Radio.
Rentrez entiers. Ou au moins, rentrez.
(Radio static fade out)
```

### SCRIPT 3 : Fin de cycle, orbite et amarrages (ICLI Station)

```
[Spoken Intro: Dynamic and natural French Radio Host]
(Jingle dynamique I-C News Radio)
Ici I-C News Radio, fin de cycle. En orbite, ICLI Station tourne au ralenti et les baies d'amarrage sont saturees depuis deux heures.
Le controle nous demande de le dire clairement : gardez vos distances dans la file, et pas de warp en approche. Personne n'a envie de finir la journee en debris.
Pour ceux qui redescendent vers la Capitale, la nuit tombe deja sur Costa Rive. On vous accompagne.

[Instrumental Drop: Space Synth-Rock]
[Long Instrumental Break: slower tempo, wide analog synth pads, distant guitar, cinematic]
(Music playing intensely)
[End of Instrumental]

[Music fades down into background]
[Spoken Interlude: Dynamic and natural French Radio Host]
(Bruitage de communication radio)
Rappel du reseau aerotram : les navettes fonctionnent par sauts courts, de tour en tour. Il n'y a pas de vol direct, il n'y en a jamais eu, et non, insister au guichet ne cree pas de ligne nouvelle.
Alors profitez du trajet. C'est le seul endroit de cette planete ou personne ne vous tire dessus.
Derniere piste du cycle, montez le son.

[Instrumental Drop: Upbeat Cyberpunk]
[Long Instrumental Break: nostalgic retro-futuristic synths, steady beat, warm bass]
(Music playing intensely)
[End of Instrumental]

[Music fades down]
[Spoken Outro: French Radio Host]
C'etait I-C News Radio, jusqu'au prochain cycle.
A ceux qui partent maintenant : la nuit appartient aux Sanglines, mais la route vous appartient encore.
Restez prudents la dehors.
(Radio static fade out)
```

---

## 4. Banque d'inserts reutilisables

### Identifiants de station (a coller entre deux morceaux)

```
(Jingle court)
I-C News Radio. On parle, vous conduisez.
```
```
(Jingle court)
I-C News Radio. Depuis la Capitale, jusqu'a l'orbite.
```
```
(Jingle court)
I-C News Radio. Le monde s'est arrete. Pas nous.
```
```
(Jingle court)
I-C News Radio. Si vous nous entendez, c'est que vous etes encore la.
```

### Spots publicitaires (marques reelles du jeu)

**Melrose**
```
[Spoken Ad: French Radio Host, commercial tone]
(Bruitage de moteur)
Une Sablone Melrose, ca ne discute pas avec le sable. Ca le traverse.
Melrose. Depuis l'Ancien Monde, et apres.
```

**Raffinerie Iron Dee**
```
[Spoken Ad: French Radio Host, commercial tone]
Fer, Obsidienne, Silicium, Cuivre. Vous extrayez, Iron Dee transforme, vous encaissez.
Guichet ouvert a chaque cycle. Iron Dee : nous, on fond. Pas vous.
```

**IClabs**
```
[Spoken Ad: warm corporate voice]
Trois cents Tours de Relais. Une seule promesse.
Ancrez votre signature. Le reste, IClabs s'en occupe.
```

**Titanium ICLI / ICLI Station**
```
[Spoken Ad: French Radio Host, commercial tone]
Armure, medicaments, armement. Trois comptoirs, une seule adresse : ICLI Station.
Montez, achetez, redescendez vivant.
```

**SawgeniuS Park**
```
[Spoken Ad: cheerful, slightly too cheerful]
SawgeniuS Park, YellowWall. De la vraie verdure, sous vraie surveillance.
Entree libre. Sortie controlee.
```

### Bulletins courts

**Cours des matieres**
```
(Bruitage de communication radio)
Cours du cycle : le Fer stable, l'Obsidienne en hausse, le Silicium plat, le Cuivre en baisse de deux points.
Traduction : videz vos soutes maintenant, pas au prochain cycle.
```

**Trafic aerotram**
```
(Bruitage de communication radio)
Reseau aerotram : rame HLL817 a l'heure vers Dire Dawa. Correspondance obligatoire a la tour suivante, comme toujours.
```

**Alerte zone**
```
(Bruitage de communication radio)
Avis aux convoyeurs : nid signale sur l'axe est. Contrats de livraison maintenus, prime de risque relevee. A vous de voir ce que vaut votre cycle.
```

---

## 5. A arbitrer (non verifie dans le jeu)

- **Frequence affichee** de la station : le champ `Frequency` existe dans `FQRadioStation`, sa valeur reelle n'a pas ete relevee. Ne pas annoncer un nombre a l'antenne tant qu'il n'est pas fixe.
- **Nom du presentateur** : aucun n'existe. Si on en veut un, il doit entrer dans la nomenclature des PNJ du jeu (prenom court + nom techno : Maya Servo, Liam Apex, Iris Daxon, Kara Unitas, Noah Quanta, Will Stryke, Ben Stratos, Gia Matrix).
- **Terme FR pour "Recovery Room"** : le jeu dit "Centre de Reveil" pour *awakening center* dans une quete FR, mais la traduction de *Recovery Room* n'a pas ete relevee. Les scripts utilisent "salle de recuperation" : a aligner sur la loc definitive.
- **Duree des segments** : chaque piste ajoutee doit avoir son `Duration` exact dans `TrackMeta` du catalogue, sinon la synchro horloge derive (voir `QRADIO_GUIDE.md` par. 4).
