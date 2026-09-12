# QANGA Radio : brief commun pour le panel Suno

> Ce fichier est la règle de fabrication de TOUS les fichiers de station du dossier `Documentation/Suno/`.
> Il est lu par chaque rédacteur (humain ou agent) avant d'écrire un morceau. Il dérive de
> `Documentation/QANGA_LORE_BIBLE.md` (le canon) et de `Documentation/QRADIO_ICNEWS_ANTENNE.md` (le ton d'antenne).
> En cas de doute entre ce brief et la bible du lore, la bible gagne.
>
> Rédigé le 2026-09-12 pour Benja. Contenu éditorial uniquement : aucun asset, aucun code.

---

## 1. Ce qu'on fabrique

Un panel de morceaux, jingles, interventions d'animateur et publicités **prêts à coller dans Suno**, pour chaque station de radio du jeu. Niveau visé : les radios de Fallout et de GTA V. Chaque station a une identité, un animateur nommé, une langue dominante, des jingles, des pubs pour les marques du monde, et une playlist cohérente.

**Ce qu'attend Benja (demande du 2026-09-12)** :
1. Beaucoup de morceaux (15 à 30 par station musicale). Il a les crédits Suno.
2. Des langues variées : français, anglais, espagnol, russe, et d'autres quand c'est pertinent (allemand, italien, portugais, japonais, coréen, arabe, latin). On mélange les langues d'une station à l'autre, et à l'intérieur d'une station quand son identité le permet.
3. **Des références à notre vrai monde** : chaque morceau est un clin d'oeil à une musique connue dans le monde entier ou dans son pays (le titre, le nom d'artiste pastiche, le genre, la structure, le thème), pour créer une accroche immédiate chez le joueur. Et en même temps il est ancré dans le lore de Qanga. Les deux à la fois, jamais l'un sans l'autre.
4. Aucune modification du moteur, des assets ou du catalogue QRadio. Benja intègre lui-même.

---

## 2. La règle du clin d'oeil (juridique et Suno)

- **On ne recopie JAMAIS une parole réelle**, pas même un vers, pas même un refrain célèbre. Les paroles sont 100 % originales. Le clin d'oeil passe par : le **titre** détourné (`Hotel Capitale`, `No Cyborg No Cry`, `Yellow Wall Style`), le **nom d'artiste pastiche** (le jeu a déjà Bob Marcly, Damia Marcly, Big Roh : on continue dans cette veine), le **genre et l'époque** décrits dans le prompt, la **structure** (un refrain scandé, un break de guitare, un choeur), et le **thème** retourné vers le lore.
- **Jamais de nom d'artiste réel dans le prompt de style Suno** : Suno refuse ces prompts. On décrit le style (« 1970s Jamaican roots reggae, one drop drum, skank guitar, warm male vocal ») sans nommer personne.
- Chaque morceau porte une ligne `Clin d'oeil :` qui dit à Benja quelle oeuvre ou quel artiste réel est visé (pays entre parenthèses). Cette ligne est pour lui, elle ne va pas dans Suno.
- Une chanson traditionnelle du domaine public (Katioucha, La Cucaracha, Guantanamera, Cielito Lindo, Santiano...) peut être évoquée par le titre et l'esprit, mais on écrit quand même des paroles originales.

---

## 3. Le canon en une page (à respecter mot pour mot)

**Dates.** L'effondrement (La Perte d'un Monde) : **2180**. L'Appel d'ICLabs et le lancement de **CRYO19** : **2184** (39 % de la population mondiale adhère, construction de Yellow Wall City). Naissance du Fléau, les **Sanglines** : **2228**. Premier cyborg fonctionnel, le bipède **Qanga 19#08** : **2290**. Lancement, le lever du jour : **2350**. **Le présent du jeu : 2755.** Entre l'effondrement et le présent il y a 575 ans. On date en **cycles** (`Cycle 103.27`), jamais en années à l'antenne.

**Le monde.** La Terre est hostile et méconnaissable. L'humanité vit dans des enclaves comme **Yellow Wall City** : cité-forteresse au milieu de l'océan, entourée d'un **mur de béton recouvert d'or** (300 m de haut, 8 km de rayon) qui retient les eaux. Elle abrite **la Capitale** (siège d'ICLabs). Un ascenseur relie le complexe souterrain à la surface. Autour, la **zone de test** : désert, plaines, forêt, jungle, failles, grottes. **YellowWall est au sol : pas de dôme, pas de baie d'amarrage.** Les amarrages sont à **ICLI Station**, la station **orbitale** (vendeurs d'armes, de médicaments, d'armures, station de warp).

**Quartiers de la Capitale** (noms réels) : le Centre, la Traverse, Centinela, Mass District, Downroad, Costa Rive (la plage), Staros, Qamarac, Central Place, Penisula, Port, Hopital, Banlinor, Litchout, les jardins et le Hall du Complexe IClabs, SawgeniuS Park, le Centre de Purification, Quarry Stone (la carrière), Echoes of Ruin. Le réseau de tram de la Capitale s'appelle **Tamil Station**.

**Région de départ** : **Djibouti / Dire Dawa** (Afrique de l'Est : Tour de Relais, centre logistique, le Grand Pont). Autres lieux nommés : **Starkitown** (ville abandonnée), la **Raffinerie Iron Dee**, l'usine **Rost-Stahl** (à 1 km dans le désert), la **Tour de Relais 77** (relais radio isolé au pied de la montagne). Quarry Stone et Rost-Stahl ne sont pas des destinations visitables : on peut les citer comme lieux de travail ou de rumeur, jamais comme « allez-y ».

**Le joueur.** Un **Cyborg** : un humain cryogénisé (CRYO19) réveillé par ICLabs et intégré dans un corps de cyborg. On s'adresse à lui par « Cyborg » ou « Citoyen ». **La mort n'est plus définitive** : environ **300 Tours de Relais** sur la planète, chacune avec sa **salle de récupération**. Si le corps est détruit, la conscience traverse le réseau et se réveille dans un corps neuf à la tour d'**ancrage** (« ancrer sa signature »). Le **noyau** (core) est l'organe vital du cyborg. La blague signature : la mort n'est pas grave, seule la cargaison l'est (« le corps est neuf, la cargaison, elle, ne revient pas toute seule »).

**Services d'une Tour de Relais** : salle de récupération, laboratoire (rachète la biomasse de Sangline), armurerie, magasin d'équipement, centre logistique (contrats de livraison), comptoir d'échange (rachète les vestiges de l'Ancien Monde), aérostation, garage. Signalement : plaques bleu nuit et structures rouges.

**Transports.** **Aérotram** : navettes autonomes qui relient les tours **par sauts courts**, jamais de vol direct longue distance, on change de tour en tour (rames codées, ex. `HLL817`). **Melrose** est le constructeur de véhicules : la **Sablone** (moto), l'**Orizaune** (vaisseau), Taxi, SUV, Sport, PickUp, PickUp armé, Riper, Coopay, Citizen, Police. Autres constructeurs : **Velkara** (Explorer, Passenger : vendus par ICLIspace), **Etlas** (Pawad, PodFury), **Berlin** (Police), **HV**, **Storm**, **Valrifle**. Armes : **NASH**. **Warp** : impossible dans l'atmosphère, bloqué en cas de collision, consomme de l'énergie. Le **jetpack** consomme de la Matière.

**Économie.** Monnaie : les **Crédits**. **La Matière n'est pas de l'argent** : c'est l'énergie vitale du cyborg, on **recycle** un objet pour en produire, elle soigne, répare, alimente le jetpack et les véhicules (« Warning! Low Matter! »). Le **disque de Matière** est sa forme échangeable. Minerais : **Fer, Obsidienne, Silicium, Cuivre, Aluminium** (et Acier, Lithium, Sable, Pierre, Plastique). **Contrats** de livraison au centre logistique.

**Les menaces.** **Les Sanglines** (féminin : une Sangline) : créatures libérées par la fonte des glaces, rapides, en **nids**, variantes **Climber, Sand Digger** (carapace pare-balles), **Flyer**, Spike's. Elles mordent le métal encore chaud. **Les Infectés** : humains contaminés par les Sanglines, plusieurs stades. **Cyborgs-Pirates**, charognards. **Rogue** : cyborgs défaillants, hostiles à tout. Zones dites **sauvages** entre les tours.

**Les sociétés.** **ICLabs Industries** (possède les tours, donc possède la résurrection ; ses intentions finales restent volontairement en suspens). **Autonomes** : robots construits pour bâtir Yellow Wall ; les modèles à intelligence évolutive se sont rebellés, ICLabs a bridé les autres. **Humains Désynchronisés** : ont repris un corps organique hors du contrôle d'ICLabs. **Communautés de la Surface** : petits groupes humains en surface, en marge de la technologie d'ICLabs, refusant son contrôle (à l'antenne on ne les appelle **jamais « survivants »**). **Humains de MarsOne** : descendants des colons martiens, société indépendante, méfiante envers les Terriens. **Habitants d'ICLISpace** : communautés humaines en orbite.

**Le Voss.** La faction s'appelle **le Voss**, et rien d'autre (jamais « Échelon Zéro »). Fondé par le **Dr Cael Veyron**, scientifique d'ICLabs spécialiste du transfert de conscience. Sa doctrine, le **Principe de Correction** : « L'évolution corrige toujours ses erreurs. Et l'humanité pourrait être l'une d'elles. » Il rassemble des Autonomes, des cyborgs de première génération rejetés, des IA militaires abandonnées et des humains volontaires. Trois objectifs : empêcher la renaissance incontrôlée de l'humanité biologique ; créer une civilisation hybride conscience humaine + machine ; restaurer l'équilibre écologique sous supervision artificielle. **Ce n'est pas une bande de pillards** : c'est une position argumentée. Esthétique : tenue anthracite, interface **ambre**. Le Voss est en paix avec la Dissidence et les Pirates, hostile à ICLabs. Son matériel est « puissant mais **instable**, avec contreparties ». La ligne officielle d'ICLabs sur le Voss est une **mise en garde produit** (matériel non homologué), pas une déclaration de guerre. Le pirate du Grand Pont parle des « vrais maîtres du désert ».

**TONA** : l'IA installée au réveil du joueur, qui le tutoie, argot militaire sec (« Stay frosty »). Son secret (premier transfert humain raté) **ne doit jamais être révélé** dans une chanson.

**Expressions du monde** : « **Par le Voile** » (juron fort, à doser, jamais expliqué), « **Keep the Flux** » (adieu, jamais expliqué), « **Bonjour Cyborg** », « Stay frosty », « a deal is a deal ». Argot pirate : charognard, gear, hideout, spare arm, Matter Disk, reboot, drained battery. **L'Ancien Monde** : l'avant (vieux terminaux, pièces de moteur, ferraille).

**Les 22 marques et leurs slogans peints** (à utiliser tels quels dans les pubs) :

| Marque | Métier | Slogan |
|---|---|---|
| IC Labs Industries | la corporation | « Yellow Wall: the safety first » ; « Industry intelligence, innovation & design » |
| I-C News Radio | la radio officielle | « Your ultimate playlist, all day, every day » ; « Your sound, our passion » |
| ICLIspace | chantier orbital, vend les Velkara | « Now available: Velkara Explorer » ; « Don't dream it, fly it » |
| Titanium ICLI | matériaux | « Unbreakable » ; « Tearproof » |
| Iron Dee | raffinerie et véhicules | « Fast. Big. Comfortable. Why choose ? » |
| Melrose | constructeur, division sport | « Tougher than you can imagine » ; « Melrose Motorsport » |
| Cryoday | marque grand public de CRYO19 | « A better future for your family » ; « Futur is cryo » |
| Cope Mutter | la banque | « Believe in yourself, we believe in your project » ; « Invested in your future. Now. » |
| Sola | soda | « Sola drink, great cryo » ; « Which one is your mood today ? » |
| McFrailenergy | boisson énergisante | « The energy you need » |
| Komet | parfumerie masculine | « Komet Fragrance. The Pure. » |
| Luna | parfumerie féminine | « New fragrance, Iris, for her » ; « Luna Fragrance, Narcisse » |
| Starkito | restauration saine | « Healthy everyday » |
| Chez Fred | restaurant, service du midi | « Treat yourself to lunch, Monday to Friday » ; « The real taste » |
| Sboutique | centre commercial | « Shop. Enjoy. Repeat. At our mall. » ; « Spend without limits » |
| Hardware Store | matériel informatique | « The new YGC-7999K, why are you waiting ? » ; « Upgrade to the futur » |
| A.D Store | armurerie | « You're one purchase away from being safe. » ; « Protect your family / Stand up for yourself » |
| CineVortex | cinéma | « Get your ticket » ; « Open 24/24 » ; « Admit one » |
| SawgeniuS Park | parc d'attractions | « New attraction, new sensation » ; « Open 7/7, unlimited fun » |
| Costa Riv | plage et loisirs | « Take a break, summer holiday » ; « Summer special offer -50 % » |
| LoopLife | végétal et bien-être | « What if you keep it alive all your life ? » ; « Take a deep breath » |
| Vieego | écologie | « Nature by Vieego Futur » |

Autres entités canon utilisables : **Etlas** (véhicules extrêmes), **Vissco** (déchets et protection écologique), **Loanicle** (location de véhicules par carte), **MarsOne**, **IC Police** (écusson « IC POLICE / ICLABS INDUSTRIE »).

**Nomenclature des noms de PNJ** : prénom court + patronyme technique ou astronomique. Existants (à ne pas réutiliser pour un animateur sans accord) : Liam Apex, Gia Matrix, Ben Stratos, Will Stryke, Kara Unitas, Maya Servo, Iris Daxon, Hugo Nexion, Daria Forge, Tom Ohm, Noah Quanta, Aiden Quasar, Aria Pulsar, Elara Nova, Tara Nebula, Zorin Cypher, Ayla Romane (journaliste), Dr Emil Karrow (biologiste). Les animateurs proposés dans ce panel suivent la même forme (Théo Cadence, Vera Volt, Ray Kelvin, Gus Lumen, Rafa Cometa, Nika Sputnik, Rex Carbone, Inès Legato, Lou Pixel, Max Tera, Abel Tempo, Awa Zenith). Les noms d'artistes pastiches peuvent être plus libres (Bob Marcly, Rame, Big Roh).

---

## 4. Les interdits absolus

1. **Aucun tiret cadratin (U+2014) ni demi-cadratin (U+2013)**, nulle part : ni dans les paroles, ni dans les prompts, ni dans les tableaux. Deux-points, virgule, point, parenthèses. Seul le trait d'union `-` est permis dans un mot composé. Vérification obligatoire avant de rendre : `grep -c -e $'\xe2\x80\x93' -e $'\xe2\x80\x94' <fichier>` doit afficher `0`.
2. Pas de « survivants » (au sens post-apo), pas d'« apocalypse », pas de « zombies », pas de « la Terre est morte », pas de « fin du monde ». Le monde s'est effondré, l'économie continue.
3. Pas de dôme sur YellowWall, pas d'amarrage à YellowWall (c'est à ICLI Station, en orbite).
4. Crédits et Matière ne sont jamais confondus. On ne « paie » pas en Matière ; on ne « recycle » pas des Crédits.
5. Pas de vol aérotram direct longue distance : on saute de tour en tour.
6. Jamais « Échelon Zéro ». La faction est **le Voss**.
7. Pas de fréquence chiffrée annoncée à l'antenne (le champ n'est pas fixé dans le jeu).
8. Ne pas expliquer le Voile ni le Flux. Ne pas révéler ce qu'est TONA. Ne pas résoudre ce qu'ICLabs cache.
9. Aucune parole réelle recopiée, aucun nom d'artiste réel dans un prompt Suno.
10. Les chaînes destinées au joueur passent par la localisation dans le jeu : ici on écrit du contenu audio, donc la langue de chaque morceau est celle annoncée, et on l'écrit correctement (vrai russe en cyrillique, vrai espagnol avec ses accents, vrai allemand, vrai japonais).

---

## 5. Format de chaque fichier de station

Nom : `Documentation/Suno/NN_<StationId>.md`. Encodage UTF-8. Structure obligatoire :

```
# <Nom de la station> (<StationId>)

Identité, animateur, langues, registre, place dans le monde : 8 à 15 lignes.

## Sommaire

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | ... | ... | FR | 3:20 | reggae roots | ... |
(une ligne par MORCEAU musical, dans l'ordre des blocs ci-dessous)

## Morceaux

### 01. <Titre>

| Champ | Valeur |
|---|---|
| Artiste | <nom pastiche> |
| Langue | <FR / EN / ES / RU / ...> |
| Durée cible | <m:ss> |
| Style | <3 à 6 mots> |
| Clin d'oeil | <oeuvre ou artiste réel visé (pays)> |
| Ancrage lore | <1 phrase : ce que le morceau raconte du monde> |

**Prompt Suno (champ Style of Music)**
```text
<150 à 400 caractères : genre, sous-genre, époque, tempo en bpm, instruments, ambiance, type de voix (male/female, tessiture, timbre), langue chantée, structure notable. Sans nom d'artiste réel.>
```

**Paroles (champ Lyrics)**
```text
[Intro]
...
[Verse 1]
...
[Pre-Chorus]
...
[Chorus]
...
[Verse 2]
...
[Chorus]
...
[Bridge]
...
[Chorus]
...
[Outro]
...
```

(répéter pour chaque morceau)

## Jingles et identifiants de station

### J01. <Titre du jingle>
Durée cible : 0:08. Langue : FR.
**Prompt Suno** : ```text ... ```
**Texte** : ```text [Spoken: ...] ... [Sung jingle] ... ```

## Interventions d'animateur

### A01. <Titre>
(même bloc : durée cible, langue, prompt, texte avec balises [Spoken Intro: ...] / (bruitage ...) comme dans QRADIO_ICNEWS_ANTENNE.md)

## Publicités

### P01. <Marque> : <accroche>
(même bloc ; 15 à 30 secondes ; termine par le slogan peint de la marque, tel quel ou traduit)

## Notes d'intégration
Pistes existantes du projet à monter sur cette station (nom d'asset et durée exacte), et remarques pour Benja.
```

Règles de contenu :
- **Paroles complètes** pour chaque morceau : 16 à 40 lignes, entre 900 et 2 800 caractères (limite Suno sûre : 3 000). Balises : `[Intro]`, `[Verse 1]`, `[Verse 2]`, `[Pre-Chorus]`, `[Chorus]`, `[Bridge]`, `[Outro]`, plus au besoin `[Instrumental Break]`, `[Guitar Solo]`, `[Drop]`, `[Spoken]`, `[Choir]`, `[Rap Verse]`. Un morceau instrumental porte `[Instrumental]` et une ou deux balises de section descriptives (`[Piano intro]`, `[Build]`, `[Outro fade]`).
- **Aucun bloc vide, aucun « à compléter »**. Chaque morceau est fini.
- **Durée cible** : chansons 2:30 à 4:00 (un ou deux morceaux longs de 5 à 6 min par station sont bienvenus) ; jingle 5 à 15 s ; pub 15 à 30 s ; intervention 30 à 90 s. La durée réelle sera mesurée après génération (voir `QRADIO_STATIONS_PLAN.md`).
- **Langue** : celle annoncée, écrite correctement. Un refrain peut mélanger deux langues si l'identité de la station le permet (ex. espagnol avec un mot d'anglais, français avec un mot de patois jamaïcain). Le russe est en cyrillique, le japonais en kana/kanji, le coréen en hangeul, l'arabe en alphabet arabe (avec une translittération en commentaire si utile pour Benja). Pour un titre non latin, donner la translittération entre parenthèses dans le Sommaire.
- **Registre d'antenne** (bible §9 bis, registre C) : décalage entre un ton enjoué et un monde qui mord. Jamais déprimé, jamais cynique au point de casser l'immersion. L'humour vient du contraste. Pour les stations non corporate (Voss, Surface), le ton est le leur, mais toujours argumenté et écrit, pas « bande de pillards ».
- **Ancrage lore** : chaque chanson contient au moins un élément concret du monde (une Tour de Relais, une salle de récupération, la Matière, un quartier, une marque, une Sangline, l'aérotram, le Mur, un cycle, un contrat, ICLI Station, l'Ancien Monde, le Voss...). Les meilleures chansons prennent un thème universel (amour, nuit, fête, nostalgie, colère, fatigue, espoir) et le tordent avec une réalité du monde (on ne meurt plus vraiment, le corps est neuf mais la mémoire reste, tout est à recycler, le mur doré, le réveil après des siècles).
- **Pubs** : une marque par spot, ton de la station, le slogan peint en chute. Les 22 marques doivent être couvertes au moins une fois sur l'ensemble des stations (répartition dans le plan). La radio du Voss ne fait pas de pub : elle fait des **contre-publicités** par-dessus les marques d'ICLabs.
- **Jingles** : au moins 4 par station (un identifiant court, un « retour de pub », un « nuit / fin de cycle », un « top horaire / nouveau cycle »). Ils peuvent être chantés (choeur, vocodeur) ou parlés, à la manière de GTA.
- **Interventions** : 4 à 8 par station, 30 à 90 secondes, dans la voix de l'animateur. Pas de nombres précis de fréquence. On peut citer les cours des minerais, le trafic aérotram, un nid signalé, la file d'amarrage à ICLI Station, un contrat en retard, une rumeur de Voss, la météo de cycle, un auditeur qui s'est réveillé en salle de récupération.

---

## 6. Écrire un bon prompt Suno (rappels)

- Ordre utile : genre principal, sous-genre ou époque, tempo (bpm), instruments clés, texture de production (lo-fi, analog tape, glossy, live room), ambiance en 2 ou 3 adjectifs, voix (male / female / duet / choir, tessiture, timbre : raspy, breathy, crooner, autotuned, vocoder), langue chantée (« French lyrics », « sung in Russian »), et une note de structure si elle compte (« big anthemic chorus », « rap verses over sung chorus », « long instrumental outro »).
- Pour un instrumental : ajouter « instrumental, no vocals » et cocher Instrumental dans Suno.
- Pour un jingle ou une pub : « radio jingle », « radio commercial », « spoken word announcer over bed », « 15 seconds », et décrire le lit musical.
- Pour une voix d'animateur : « natural French radio host, dynamic, warm, close mic, light music bed » et écrire le texte avec `[Spoken Intro: ...]` comme dans les scripts existants. Suno rend mieux une intervention parlée quand elle est encadrée d'une ou deux balises musicales (`[Music bed fades in]` / `[Music bed fades out]`).
- Éviter les prompts trop longs ou contradictoires ; éviter « best », « viral », « hit » ; ne pas mettre de paroles dans le champ style.

---

## 7. Les 13 stations (résumé, détail dans chaque fichier)

| NN | StationId | Nom d'antenne | Langue dominante | Genre | Animateur proposé |
|---|---|---|---|---|---|
| 01 | `ICNewsRadio` (existe) | I-C News Radio | FR | parlé, corporate, space synth-rock | Théo Cadence (+ Ava Signal) |
| 02 | `ChillFM` (existe) | Chill FM | instrumental, EN, FR, JP | ambient, downtempo, lo-fi | aucun (identifiants chuchotés) |
| 03 | `YellowRoots` | Yellow Roots | FR, EN | reggae, dub, dancehall, hip-hop | Ray Kelvin |
| 04 | `Traverse` | Traverse | EN, FR, DE | house, French touch, techno, synthwave | Vera Volt |
| 05 | `VieuxMonde` | Vieux Monde | FR, EN, ES, IT | folk, country, chanson, blues acoustique | Gus Lumen (alt. Tom Ohm) |
| 06 | `VossAmbre` | AMBRE (fréquence du Voss) | FR, EN, RU, DE | industrial, EBM, post-punk, dark techno | la Voix AMBRE (anonyme, vocodée) |
| 07 | `RadioCentinela` | Radio Centinela | ES (+ PT) | latin pop, reggaeton, cumbia, salsa, bachata | Rafa Cometa |
| 08 | `RadioOrbita` | Радио Орбита | RU (+ EN) | pop soviétique, post-punk russe, russian rock, hardbass | Nika Sputnik |
| 09 | `ForgeFM` | Forge FM | EN, FR, DE, ES | rock, hard rock, metal, punk, grunge | Rex Carbone |
| 10 | `RadioVelours` | Radio Velours | EN, FR, PT, IT, JP | jazz, soul, bossa, lounge, city pop | Inès Legato |
| 11 | `HitWall` | HitWall | EN, FR, KO, JP, ES, IT | pop 2755, dance-pop, K-pop, eurodance | Lou Pixel et Max Tera |
| 12 | `Pantheon` | Panthéon | IT, LA, DE, FR, EN | classique, opéra, choeur, néo-classique, film score | Abel Tempo |
| 13 | `TamilOndes` | Tamil Ondes | instrumental + annonces FR/EN/ES/RU | muzak, easy listening, exotica | la Voix du réseau (synthétique) |
| (hors panel) | `RadioSable` | Radio Sable | FR, AR, EN | desert blues, éthio-jazz, afrobeat | Awa Zenith |

Radio Sable (la voix des communautés de la Surface, autour de Djibouti / Dire Dawa) est proposée en 14e station dans le plan ; elle a son propre fichier `14_RadioSable.md`.

---

## 8. Banques de clins d'oeil par station

Chaque station a sa banque (dans son fichier) pour éviter qu'un même tube soit détourné deux fois. Règle de partage : un clin d'oeil n'est utilisé que par une station. Quand deux stations pourraient revendiquer un tube, la répartition ci-dessous fait foi :

- Traverse garde la French touch, Kraftwerk, la trance, l'eurodance à synthés, « Wake Me Up », « Titanium », « 99 Luftballons ».
- HitWall garde les tubes pop mondiaux, K-pop, J-pop, ABBA, Madonna, « Toto : Africa », « Papaoutai », Aqua, Eiffel 65.
- Velours garde Sinatra, Nat King Cole, Sade, Amy Winehouse, la bossa (Ipanema), la city pop japonaise, « Volare », Gainsbourg.
- Vieux Monde garde Dylan, Cash, John Denver, Eagles, Simon and Garfunkel, Brassens, Brel, Cabrel, Piaf, Aznavour, Manu Chao, De André, Dalla, les traditionnels mexicains et cubains.
- Forge FM garde AC/DC, Metallica (Enter Sandman, Nothing Else Matters), Queen, Nirvana, Guns N' Roses, Led Zeppelin, Scorpions, Téléphone, Noir Désir, Indochine, Johnny, Héroes del Silencio, Soda Stereo, Die Toten Hosen.
- AMBRE garde Rammstein, Nine Inch Nails, Depeche Mode, Joy Division, Kino (Перемен), Nautilus Pompilius, Гражданская Оборона, Rage Against the Machine, Metallica (Master of Puppets), Laibach, Bérurier Noir, Trust.
- Radio Orbita garde Земляне, t.A.T.u., Кино (Звезда по имени Солнце), Пугачёва, Высоцкий, Катюша, Калинка, Подмосковные вечера, Little Big, Ленинград, Мумий Тролль, Сплин, ДДТ, Молчат Дома, Кипелов, Пахмутова (Нежность), Руки Вверх.
- Radio Centinela garde Despacito, Bailando, Vivir mi vida, Gasolina, Livin' la Vida Loca, La Bamba, Macarena, Cielito Lindo, Guantanamera, Danza Kuduro, Waka Waka, Hips Don't Lie, Gipsy Kings, Gardel, Juanes, Aventura, Rosalía, Calle 13, Maná, Santana, Michel Teló, Sergio Mendes.
- Yellow Roots garde Bob Marley, Peter Tosh, Jimmy Cliff, Shaggy, Sean Paul, Alpha Blondy, Tiken Jah, Dub Inc, Danakil, IAM, NTM, MC Solaar, Orelsan, PNL, Booba, Snoop, Kendrick, Notorious B.I.G., Eminem, Coolio, Magic System.
- Chill FM garde Air, Massive Attack, Portishead, Moby, Boards of Canada, Enya, Röyksopp, Bonobo, Nujabes, Brian Eno, Satie, Sakamoto, Vangelis, Sigur Rós, Radiohead, Zero 7, Groove Armada.
- I-C News garde Ray Ventura (Tout va très bien), Bobby McFerrin (Don't Worry Be Happy), Katrina and the Waves, ELO (Mr. Blue Sky), Beatles (Here Comes the Sun), Monty Python (Bright Side), Doris Day (Que sera sera), « La vie en rose » (en version jaune).
- Panthéon garde Orff, Puccini, Verdi, Beethoven, Mozart, Bach, Haendel, Barber, Holst, Strauss, Bizet, Wagner, Tchaïkovski, Debussy, Vivaldi, Morricone, Zimmer, Vangelis (Chariots), Pärt, Jenkins, Gerrard, le chant grégorien, Полюшко-поле.
- Tamil Ondes garde Herb Alpert, Bacharach, Popcorn, Martin Denny, Perrey, Yellow Magic Orchestra, Ellington (Take the A Train), Chattanooga Choo Choo, Moon River, Last Train to Clarksville.
- Radio Sable garde Tinariwen, Mulatu Astatke, Fela Kuti, Ali Farka Touré, Youssou N'Dour, Khaled, Fairuz, Oum Kalthoum, Amadou et Mariam, Burna Boy, Salif Keita, Rachid Taha, Bombino, Songhoy Blues.

---

*Brief commun du panel Suno. Créé le 12 septembre 2026. Toute modification du canon se reporte d'abord dans `QANGA_LORE_BIBLE.md`.*
