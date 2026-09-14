# QRadio : plan des stations et panel Suno

> Livrable de contenu pour Benja, rédigé le 2026-09-12. Aucun asset ni code n'a été modifié : ce document et le dossier `Documentation/Suno/` sont du contenu à intégrer.
> Sources : `QANGA_LORE_BIBLE.md` (canon), `QRADIO_ICNEWS_ANTENNE.md` (ton d'antenne), `QRADIO_GUIDE.md` et `QRADIO_ARCHITECTURE.md` (technique), `QMUSIC_ARCHITECTURE.md` (ce qui est déjà affecté à l'ambiance), et un relevé des assets audio (durées lues dans les `.uasset`, section 8).
> Règle du document : tout ce qui est présenté comme un fait du projet a été mesuré (chemin et valeur). Tout ce qui est une proposition est marqué comme telle.

---

## 0. En une minute

- **14 stations** : 2 existantes (I-C News Radio, Chill FM) et 12 nouvelles, chacune avec identité, animateur nommé, langues, jingles, interventions, publicités et playlist.
- **268 morceaux, 65 jingles, 85 interventions et 81 publicités ou annonces**, tous écrits en entier : prompt de style et paroles complètes taguées, prêts à coller dans Suno. Onze langues : français, anglais, espagnol, russe, allemand, italien, portugais, japonais, coréen, arabe, latin.
- **Le panel Suno** est dans `Documentation/Suno/` : un brief commun (`00_BRIEF_COMMUN.md`) et un fichier par station (`01_ICNewsRadio.md` à `14_RadioSable.md`). Le détail par station est en section 4, les comptes vérifiés en section 9.
- **Chaque morceau est un clin d'oeil à une chanson réelle**, par son titre détourné, son nom d'artiste pastiche et son genre, dans la lignée des Bob Marcly et Big Roh déjà en jeu. Aucune parole réelle n'est reprise et aucun nom d'artiste réel n'apparaît dans un prompt Suno : la ligne `Clin d'oeil` de chaque morceau dit à Benja ce qui est visé.
- **Lecture continue : voie (B) retenue par Benja le 2026-09-14** : un bloc d'antenne monté par station, rejoué en boucle (la boucle était déjà active dans les MetaSounds de station). Chill FM tourne sur son bloc `WAV_Radio_CHILLFM_01` (1964.71 s) depuis le 2026-09-14 (section 1).
- **Les décisions qui reviennent à Benja** sont listées en section 2 (noms de stations, animateurs, langues, fréquences, arbitrages sur trois pistes existantes).

---

## 1. Le blocage technique et les limites mesurées

| Constat | Où c'est écrit | Conséquence pour ce plan |
|---|---|---|
| **Lecture continue : voie (B) retenue par Benja le 2026-09-14** : une seule longue piste par station (un bloc d'antenne monté : jingle, chanson, pub, chanson...), rejouée en boucle par le Wave Player (`Loop = true`, mesuré le 2026-09-14 sur `MS_QRadioStation` et `MS_QRadio_ChillFM`). Ces MetaSounds n'ont pas d'entrée `Play` : inutile avec un bloc unique. | `Documentation/QRADIO_GUIDE.md`, section 11 (« À finaliser / prévu », première puce) | Chaque station se monte en **un bloc exporté du logiciel de Benja**, normalisé à -17 LUFS, et `TrackMeta[0].Duration` doit valoir **exactement** la `Duration` du SoundWave (sinon départ au mauvais endroit et auditeurs désynchronisés). La voie **(A)** (playlist par piste, avance d'index à `On Finished`, « now playing » exact) n'est pas câblée. |
| **Aucun widget de tuner** : on zappe avec deux touches, à l'aveugle. | `QRADIO_GUIDE.md` section 11 (« UI tuner + changement de station ») | À 14 stations, un tuner devient nécessaire. Les champs `DisplayName`, `Icon`, `Frequency` du catalogue sont prêts pour lui. |
| **`Frequency` vaut 0.0 et n'est lu nulle part** ; **`GenreTags` est vide partout**. | Session précédente (catalogue `Content/Systems/QRadio/DA_QRadio_Stations`) | Les fréquences de la section 6 sont une proposition d'affichage. **Aucune fréquence n'est prononcée à l'antenne** dans le panel. |
| **La réception ne filtre pas par module** : les émetteurs ne connaissent que la distance. | `QRADIO_GUIDE.md` section 7 | La station du Voss « recevable seulement avec le module Interception radio » demande un petit ajout (un drapeau de station testé contre le module) : **hors périmètre, à valider**. Le module `Interception radio` est listé dans `QMODULE_CATALOGUE.md` (ligne 89, « Capte les canaux police/Voss ») ; `Antenne longue portée` (section 9.21) multiplie déjà la portée de réception (`ReceptionRangeMult`). |
| **Un MetaSound par station** : dupliquer `MS_QRadioStation` et remplacer le tableau `Music`. | `QRADIO_GUIDE.md` sections 4 et 5 | 12 duplications à faire, une par nouvelle station. |
| **`TrackMeta.Duration` doit être exact** : c'est ce qui pilote l'horloge live. | `QRADIO_GUIDE.md` sections 3 et 10 | Les durées du panel sont des **cibles** ; la durée réelle est mesurée après génération (section 3). |
| **Cible de niveau** : musique à **-17 LUFS**, crête -1.5 dBFS. | Étalonnage du 2026-09-10/11 (mémoire `qanga-audio-loudness-calibration`) | Chaque morceau généré passe par la mesure et le gain avant import (section 3). |

---

## 2. Ce que Benja doit trancher

1. **Les noms d'antenne** des 12 nouvelles stations (section 4). Ils sont proposés, pas gravés. Les `StationId` proposés deviennent des contrats dès qu'ils entrent au catalogue : à fixer une fois.
2. **Les animateurs** : un nom principal et deux alternatives par station, dans la nomenclature du jeu (prénom court + nom techno ou astronomique). Cas particulier : Vieux Monde pourrait être animée par **Tom Ohm**, PNJ existant (cyborg Gen-1 hors réseau, collectionneur), ce qui l'engage narrativement ; le panel propose **Gus Lumen** par défaut.
3. **Les langues** : la répartition de la section 4 (FR, EN, ES, RU, DE, IT, PT, JP, KO, AR, LA). Chaque station a une langue dominante et des invitées ; c'est ajustable station par station sans réécrire le reste.
4. **La voie de lecture continue** : **tranchée par Benja le 2026-09-14, voie (B)**, bloc d'antenne long en boucle. Le panel, écrit une piste par fichier, se monte en blocs dans son logiciel avant import.
5. **Trois pistes existantes à double usage** : `Dissidence` (479.36 s), `Sanglotown` (104.56 s) et `Glitze` (158.88 s) sont dans `Content/Sounds/QangaMusic/SoundTracks/2026/`, c'est-à-dire dans la playlist d'ambiance Exploration de QMusicDirector. Les affecter aussi à une radio (AMBRE, Traverse) veut dire qu'un joueur peut les entendre en ambiance ET en radio.
6. **`SummerHit_2790`** : le titre porte l'année 2790 alors que le présent du jeu est 2755. Renommer (« SummerHit 2755 »), ou assumer un « tube du futur » ; le panel le liste sur HitWall sans trancher.
7. **La station du Voss** : recevable partout dès qu'on est près d'un site Voss (émetteurs locaux), ou uniquement avec le module Interception radio (demande un ajout moteur). Le panel est écrit pour la seconde option, et fonctionne avec la première.
8. **Les fréquences d'affichage** (section 6) pour un futur tuner.

---

## 3. Méthode d'intégration d'une station (sans C++)

Pas à pas, dérivé de `QRADIO_GUIDE.md` section 4, avec les mesures à faire sur chaque fichier généré par Suno :

1. **Générer** dans Suno : coller le prompt de style dans *Style of Music*, les paroles dans *Lyrics* (cocher *Instrumental* pour les pistes `[Instrumental]`). Générer 2 variantes, garder la meilleure, télécharger en **WAV**.
2. **Mesurer la durée réelle** (c'est la valeur de `TrackMeta.Duration`) :
   ```bash
   ffprobe -v error -show_entries format=duration -of csv=p=0 "morceau.wav"
   ```
3. **Mesurer et corriger le niveau** vers -17 LUFS (crête -1.5 dBFS), gain pur, sans limiteur :
   ```bash
   ffmpeg -i "morceau.wav" -af ebur128=peak=true:framelog=quiet -f null - 2>&1 | grep -E "I:|Peak:"
   ```
   puis, si l'écart est `g` dB : `ffmpeg -i in.wav -af volume=<g>dB -c:a pcm_s24le out.wav` (plafonner `g` pour que la crête reste sous -1.5 dBFS). Les masters de l'étalonnage précédent sont dans `G:\QangaSync\Sons\_Etalonnage_2026-09\`.
4. **Importer** les WAV dans l'éditeur, dans un sous-dossier par station (proposition : `Content/Sounds/QangaMusic/MusicRadio/<StationId>/`, à côté du vivier actuel ; les voix et pubs existantes sont dans `Plugins/Qasset/Content/Audio/WAV/Radio/`). Après import, vérifier `bLooping = False` et la SoundClass : le réimport UE réinitialise des propriétés (mémoire de l'étalonnage).
5. **Dupliquer `MS_QRadioStation`** en `MS_QRadio_<StationId>` dans `Content/Systems/QRadio/`, remplacer le tableau `Music` par la playlist dans l'ordre du Sommaire du fichier de station. Vérifier que l'asset garde `Sound Class = QSClass_Music` et `Volume ≈ 0.15` (guide section 8, point 2 : un rebuild de graphe peut les réinitialiser).
6. **Ajouter la ligne** dans `DA_QRadio_Stations` : `StationId` (celui de la section 4, stable), `DisplayName` (localisé), `StationMetaSound`, `TrackMeta` (Title, Artist, Duration mesurée, une entrée par piste, même ordre que `Music`), `Emitters` (proposition section 5).
7. **Tester** avec `qradio.Debug 1` (guide section 8) : `Defs` compte la nouvelle station, `Sig` varie avec la distance aux émetteurs.

Le câblage de la lecture continue (section 1, voie A ou B) se fait une fois, sur `MS_QRadioStation` avant les duplications, ou sur chaque copie.

---

## 4. Les 14 stations

Tableau de synthèse (le détail de chaque station, ses jingles, ses interventions, ses pubs et sa playlist complète sont dans son fichier `Documentation/Suno/NN_<StationId>.md` ; le sommaire de chaque playlist est recopié plus bas).

| NN | StationId | Nom d'antenne | Statut | Langue dominante (invitées) | Genre | Animateur proposé |
|---|---|---|---|---|---|---|
| 01 | `ICNewsRadio` | I-C News Radio | existe | FR | parlé, corporate, lits space synth-rock | Théo Cadence, Ava Signal |
| 02 | `ChillFM` | Chill FM | existe | instrumental (EN, FR, JP) | ambient, downtempo, lo-fi | aucun (identifiants chuchotés) |
| 03 | `YellowRoots` | Yellow Roots | nouvelle | FR, EN (ES) | reggae, dub, dancehall, hip-hop | Ray Kelvin |
| 04 | `Traverse` | Traverse | nouvelle | EN, FR (DE) | house, French touch, techno, synthwave | Vera Volt |
| 05 | `VieuxMonde` | Vieux Monde | nouvelle | FR, EN (ES, IT) | folk, country, chanson, blues acoustique | Gus Lumen (alt. Tom Ohm) |
| 06 | `VossAmbre` | AMBRE (fréquence du Voss) | nouvelle, clandestine | FR, EN (RU, DE) | industrial, EBM, post-punk, dark techno | la Voix AMBRE |
| 07 | `RadioCentinela` | Radio Centinela | nouvelle | ES (PT) | latin pop, reggaeton, cumbia, salsa, bachata | Rafa Cometa |
| 08 | `RadioOrbita` | Радио Орбита | nouvelle | RU (EN) | estrada, russian rock, post-punk, hardbass | Nika Sputnik |
| 09 | `ForgeFM` | Forge FM | nouvelle | EN, FR (DE, ES) | rock, hard rock, metal, punk, grunge | Rex Carbone |
| 10 | `RadioVelours` | Radio Velours | nouvelle | EN, FR (PT, IT, JP) | jazz, soul, bossa, lounge, city pop | Inès Legato |
| 11 | `HitWall` | HitWall | nouvelle | EN, FR (KO, JP, ES, IT, SV, DE) | pop 2755, dance-pop, K-pop, eurodance | Lou Pixel et Max Tera |
| 12 | `Pantheon` | Panthéon | nouvelle | IT, LA (DE, FR, EN, RU) | classique, opéra, choeur, film score | Abel Tempo |
| 13 | `TamilOndes` | Tamil Ondes | nouvelle | instrumental + annonces FR, EN, ES, RU | muzak, easy listening, exotica | la Voix du réseau (synthétique) |
| 14 | `RadioSable` | Radio Sable | nouvelle, hors réseau | FR, AR (EN) | desert blues, éthio-jazz, afrobeat, raï | Awa Zenith |

Pourquoi ces 14 et pas d'autres : les 5 premières viennent de la piste de départ validée dans la maquette de l'intranet (I-C News, Chill FM, Yellow Roots, Traverse, Vieux Monde) ; la station du Voss était demandée ; Benja a demandé de l'espagnol et du russe (Centinela depuis le quartier Centinela de la Capitale, Orbita depuis ICLI Station, les communautés d'ICLISpace étant canon), du rock, du jazz, de la pop 2755, du classique ; Tamil Ondes répond à un besoin du moteur (les trains prennent une station fixe désignée par le designer, `QRADIO_GUIDE.md` section 9) ; Radio Sable donne une voix aux communautés de la Surface (canon, section 4 de la bible) dans la région de départ.

### 4.1 Le placement dans le monde (proposition d'émetteurs)

Format de `QRADIO_GUIDE.md` section 7 : `Type`, `PlanetRef`, `LocalOffset`, `FalloffStartKm`, `RadiusKm`. Les stations existantes utilisent un émetteur à l'origine avec `FalloffStartKm=9000` et `RadiusKm=10000` (couverture planétaire). Les positions locales (Capitale, ICLI Station, Dire Dawa, sites Voss) sont **à relever dans l'éditeur** ; les portées sont des valeurs de départ à régler en jeu.

| Station | Couverture | Proposition |
|---|---|---|
| I-C News Radio, Chill FM, Panthéon, Tamil Ondes | planétaire | comme aujourd'hui : émetteur à l'origine, 9000 / 10000 km |
| HitWall, Forge FM, Vieux Monde | planétaire large | 9000 / 10000 km (Forge FM peut aussi avoir un second émetteur fort sur la Raffinerie Iron Dee) |
| Yellow Roots, Traverse, Radio Velours, Radio Centinela | la Capitale et sa région | `Local`, `PlanetRef` = la planète, `LocalOffset` = antenne de la Capitale (à relever), `FalloffStartKm` 150, `RadiusKm` 350 |
| Радио Орбита | l'orbite et la face visible | `Space`, `LocalOffset` = position d'ICLI Station (à relever), `FalloffStartKm` 3000, `RadiusKm` 8000 |
| AMBRE | ponctuelle, intermittente | plusieurs émetteurs `Local` sur les sites Voss et les camps rebelles (`Content/_QLevel/Universe/Planetary/Rebel/Composite/`), `FalloffStartKm` 4, `RadiusKm` 15 ; plus tard, gating par le module Interception radio |
| Radio Sable | le désert de Dire Dawa | cible : `Local`, `LocalOffset` = zone Dire Dawa, `FalloffStartKm` 20, `RadiusKm` 60. **Au catalogue depuis le 2026-09-14 en réception planétaire provisoire** (choix de Benja). Position candidate relevée dans `QTrain_RelayDireDawa94566_Station.Data_Location` : (-206833577.38, -147529482.36, -126205927.81), repère non vérifié, à valider sur place avec `qradio.Debug 1` avant de passer en local |

### 4.2 Détail par station

Chaque bloc ci-dessous reprend l'identité et le sommaire de la playlist. Les prompts et paroles sont dans le fichier indiqué.

#### I-C News Radio (ICNewsRadio)

Fichier : `Documentation/Suno/01_ICNewsRadio.md` (54 Ko). Morceaux : 12 ; jingles : 8 ; interventions ou messages : 6 ; publicités ou annonces : 12.

I-C News Radio est la station officielle d'IC Labs Industries : la voix civile et corporate de la Capitale, celle qui donne le trafic aérotram, les cours des minerais et le bulletin de zone avec le sourire d'un présentateur qui n'est jamais sorti de derrière le Mur. Le monde mord, la station ne le sait pas, ou fait comme si. C'est le registre C de la bible du lore : tout le sel vient du décalage.

- **StationId** : `ICNewsRadio` (existe déjà dans le catalogue `DA_QRadio_Stations` ; c'est la station de démarrage, `DefaultStationId=ICNewsRadio` dans `Config/DefaultGame.ini`).
- **Animateur** : **Théo Cadence**, la matinale. Enjoué, chaleureux, un peu trop optimiste : il traite un réveil en salle de récupération comme un retard d'aérotram. Noms alternatifs dans la nomenclature du monde (prénom court + patronyme techno ou astronomique), pour arbitrage de Benja : **Sam Vector** ou **Milo Sirius**.
- **Seconde voix** : **Ava Signal**, bulletins de zone et trafic. Sèche, factuelle, elle lit un nid de Sanglines comme un relevé de compteur. Noms alternatifs : **Léa Radian** ou **Mia Azimut**.
- **Langue** : français, avec les slogans peints des marques en anglais, tels quels. Aucune fréquence chiffrée n'est annoncée.
- **Registre musical** : lits instrumentaux space synth-rock, cyberpunk enjoué et rétro-futuriste (les anciennes pistes `ICNEWSRADIO_01`, `I-C_News_YellowWall` et les deux `WAV_I-C_News_Radio_Var` étaient dans cette veine ; retirées du projet le 2026-09-14 au profit de l'émission définitive `WAV_Radio_ICNEWSRADIO_01`, copies dans `F:\QANGA_Backups\qradio_placeholders_2026-09-14`), plus six chansons « maison » en français : de la pop corporate optimiste au second degré, l'optimisme institutionnel qui ne voit pas les Sanglines.
- **Rubriques** : ouverture et fermeture de cycle, bulletin de zone, logistique et trafic aérotram (par sauts, de tour en tour), cours des minerais (Fer, Obsidienne, Silicium, Cuivre, Aluminium), publicité. La blague signature reste celle de la bible d'antenne : la mort n'est pas grave, seule la cargaison l'est.
- **Place dans le monde** : émetteur spatial à l'origine de l'univers (plein jusqu'à 9000 km, fondu jusqu'à 10000 km d'après `QRADIO_GUIDE.md`) : on la capte à la Capitale, dans la zone de test, sur le Grand Pont de Dire Dawa et jusqu'à la file d'amarrage d'ICLI Station.
- **Slogans peints** : « Your ultimate playlist, all day, every day » et « Your sound, our passion ». Ils ferment l'autopromo et les identifiants chantés.
- **Ce que ce fichier ajoute** à la bible d'antenne (`QRADIO_ICNEWS_ANTENNE.md`, qui a déjà trois scripts, quatre identifiants et cinq spots) : 12 morceaux, 8 jingles nouveaux, 6 interventions nouvelles, 12 spots nouveaux. Rien ici ne reprend un texte déjà écrit là-bas.

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Tout va très bien, Cyborg | Théo Cadence et ses Convoyeurs | FR | 3:10 | swing music-hall, cuivres, standard téléphonique | d'après « Tout va très bien, Madame la Marquise » (France) |
| 02 | T'inquiète, t'es ancré | Les Ancrés | FR | 3:00 | a cappella, percussions vocales, sifflet | d'après « Don't Worry, Be Happy » (États-Unis) |
| 03 | Marcher sous le Mur | Costa Rive Club | FR | 3:20 | pop-rock années 80, cuivres, tambourin | d'après « Walking on Sunshine » (Royaume-Uni) |
| 04 | Monsieur Mur Jaune | Orchestre Lumière du Complexe | FR | 4:20 | pop orchestrale, cordes, vocodeur | d'après « Mr. Blue Sky » (Royaume-Uni) |
| 05 | Voilà le cycle | Les Scarabées du Relais | FR | 3:10 | folk-pop acoustique, Moog, claquements | d'après « Here Comes the Sun » (Royaume-Uni) |
| 06 | Le bon côté du réseau | La Chorale du Centre Logistique | FR | 3:20 | music-hall sifflé, ukulélé, choeur | d'après « Always Look on the Bright Side of Life » (Royaume-Uni) |
| 07 | Lever de cycle | Habillage I-C News | Instrumental | 2:40 | space synth-rock, guitare, arpèges | aucun titre précis : habillage d'antenne original, esprit synthwave des génériques d'information |
| 08 | Cours du cycle | Habillage I-C News | Instrumental | 2:30 | cyberpunk enjoué, basse punchy | aucun titre précis : habillage d'antenne original |
| 09 | Ce qui sera sera | Habillage I-C News | Instrumental | 2:50 | valse rétro-futuriste, lead synthé | d'après « Que sera, sera » (États-Unis) |
| 10 | Bulletin de zone | Habillage I-C News | Instrumental | 2:20 | synth-rock industriel, percussions | aucun titre précis : habillage d'antenne original |
| 11 | File d'amarrage | Habillage I-C News | Instrumental | 3:00 | synthé cinématique, nappes, lent | aucun titre précis : habillage d'antenne original |
| 12 | La vie en jaune | Habillage I-C News | Instrumental | 2:50 | rétro-futuriste lent, accordéon synthé | d'après « La Vie en rose » (France) |

#### Chill FM (ChillFM)

Fichier : `Documentation/Suno/02_ChillFM.md` (41 Ko). Morceaux : 18 ; jingles : 4 ; interventions ou messages : 2 ; publicités ou annonces : 2.

Chill FM est la station que personne n'a choisie et que tout le monde laisse tourner. Pas d'animateur, pas de bulletin, pas de cours des minerais : de la musique, et entre deux morceaux une voix chuchotée qui dit le nom de la station comme on vérifie que la lumière est restée allumée.

- **StationId** : `ChillFM` (au catalogue ; depuis le 2026-09-14 sa piste est le bloc d'antenne `WAV_Radio_CHILLFM_01`, 1964.71 s, qui remplace `Music_chill`).
- **Animateur** : aucun. Quatre identifiants chuchotés (FR et EN) et deux « voix de nuit » de vingt secondes remplacent les interventions.
- **Langues** : instrumental en majorité (11 morceaux sur 18), puis anglais (3), français (2, dont un en vocalises), japonais (2, en kana et kanji).
- **Registre** : ambient, downtempo, lo-fi, chillwave, trip-hop, néo-classique électronique. Tempos entre 50 et 128 bpm, jamais de refrain scandé, jamais de guitare saturée, jamais de voix qui force.
- **Ton** : apaisé, un peu mélancolique, jamais lourd. Le monde qui mord reste dehors ; la mélancolie vient de ce qu'on ne dit pas. Quand le lore entre, il entre par la fenêtre : une Tour de Relais la nuit, le calme d'un corps neuf en salle de récupération, le Mur doré au lever du cycle, l'orbite vue d'ICLI Station, les jardins IClabs, le silence du relais 77, l'aérotram entre deux tours, les siècles de sommeil CRYO19, un souvenir de l'Ancien Monde.
- **Place dans le monde** : émetteur planétaire, la station se capte partout. C'est celle qu'on garde en longeant Costa Rive de nuit, en attendant une rame sur le quai d'une aérostation, ou pendant les cinq minutes où l'on se rhabille dans une salle de récupération.
- **Voix** : quand il y en a une, elle est douce, proche du micro, souvent chuchotée. Les identifiants alternent une voix de femme et une voix d'homme, toujours à voix basse.
- **Pubs** : deux seulement, murmurées sur lit ambient (LoopLife, Vieego). Les autres marques appartiennent aux autres stations.
- **Blague signature, version Chill FM** : on ne la dit jamais en entier. « Le corps est neuf » suffit ; la cargaison, on la vérifiera plus tard.

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Coredrop | Massive Anchor | EN | 5:10 | trip-hop, voix féminine soufflée | Massive Attack, « Teardrop » (Royaume-Uni) |
| 02 | La Femme de cuivre | Air Recyclé | Instrumental | 6:00 | downtempo lounge, basse Moog | Air, « La Femme d'argent » (France) |
| 03 | Cryopédie n°1 | Erik Satellite | Instrumental | 3:20 | piano néo-classique, valse lente | Erik Satie, « Gymnopédie n°1 » (France) |
| 04 | Sexy Cyborg | Air Recyclé | FR (hook EN) | 4:40 | electro-pop vocodée | Air, « Sexy Boy » (France) |
| 05 | At the Wall | Groove Armature | Instrumental | 4:20 | chill-out trombone, vinyle | Groove Armada, « At the River » (Royaume-Uni) |
| 06 | Titanium | Mobius | EN | 4:00 | electronica piano, voix chuchotée | Moby, « Porcelain » (États-Unis) |
| 07 | Dire Dawa Cowboy | Boards of Dawa | Instrumental | 5:00 | ambient IDM, bande désaccordée | Boards of Canada, « Dayvan Cowboy » (Écosse) |
| 08 | 眠りの世紀 (Nemuri no seiki, « Les siècles de sommeil ») | Rei Sakamoon | JP | 4:30 | piano néo-classique, murmure | Ryuichi Sakamoto, « Merry Christmas Mr. Lawrence », « Energy Flow » (Japon) |
| 09 | Music for Aerostations | Brian Eon | Instrumental | 6:00 | ambient génératif, sans batterie | Brian Eno, « Music for Airports » (Royaume-Uni) |
| 10 | Everything in Its Right Slot | Relayhead | EN | 4:15 | art rock électronique, voix haute | Radiohead, « Everything in Its Right Place » (Royaume-Uni) |
| 11 | Yellow Sands | Bonobot | Instrumental | 4:50 | downtempo organique, harpe | Bonobo, « Black Sands » (Royaume-Uni) |
| 12 | 二つの塔の間 (Futatsu no tō no aida, « Entre deux tours ») | Nujavu | JP | 3:50 | lo-fi jazz-hop, rap doux | Nujabes, « Luv(sic) », « Feather » (Japon) |
| 13 | Costa Rive Flow | Ennéa | Instrumental | 4:10 | new age, pizzicato, harpe | Enya, « Orinoco Flow » (Irlande) |
| 14 | Poor Relay | Röyksync | Instrumental | 4:30 | electronica arpèges, glockenspiel | Röyksopp, « Poor Leno », « Eple » (Norvège) |
| 15 | Sour Cycles | Portishade | Instrumental | 4:20 | trip-hop noir, thérémine | Portishead, « Sour Times » (Royaume-Uni) |
| 16 | Par le Voile | Sigur Orbit | FR (vocalises, langue inventée) | 5:30 | post-rock ambient, crescendo | Sigur Rós, chant en langue inventée (Islande) |
| 17 | Memories of the Old World | Van Helios | Instrumental | 5:20 | synthé cinématique, sans batterie | Vangelis, « Memories of Green », « Blade Runner Blues » (Grèce) |
| 18 | In the Recovery Line | Zero 77 | Instrumental | 4:10 | soul-electronica, Rhodes, flûte | Zero 7, « In the Waiting Line » (Royaume-Uni) |

#### Yellow Roots (YellowRoots)

Fichier : `Documentation/Suno/03_YellowRoots.md` (76 Ko). Morceaux : 22 ; jingles : 5 ; interventions ou messages : 6 ; publicités ou annonces : 6.

**Au catalogue depuis le 2026-09-14** : StationId `YellowRoots` (contrat : ne plus le renommer), bloc d'antenne `WAV_Radio_YellowRoot_01` (2037.66 s ; source à -18.2 LUFS, copie remontée de 1 dB à -17.2 LUFS, gain limité par la crête à -1.5 dBFS) dans `MS_QRadio_YellowRoots`. Réception planétaire provisoire comme HitWall et Radio Sable ; la cible du plan reste la Capitale et sa région (section 4.1).

Yellow Roots est la radio de la Capitale populaire. Elle émet depuis un local au-dessus d'un atelier de recyclage de Downroad, et son signal couvre Costa Rive (la plage), Mass District et le Port. Reggae roots, dub, dancehall, hip-hop français et anglais, une touche afro et un crossover en espagnol pour la plage. C'est la station de ceux qui bossent : la carrière, les contrats de livraison, la biomasse revendue au labo, le comptoir d'échange, le quart de nuit à l'aérostation.
Animateur : **Ray Kelvin**, dit « Ray K ». Voix grave et chaude, il tutoie tout le monde, appelle ses auditeurs « la famille » et parle français avec des mots d'anglais jamaïcain (bredda, irie, easy, big up, nuff respect, ya man). Il rit de la mort parce qu'on se réveille à la tour, mais il ne rit jamais de la cargaison. Deux noms alternatifs dans la nomenclature du monde (prénom court + patronyme technique ou astronomique) si Ray Kelvin doit changer : **Kofi Meridian** ou **Sol Tesla**.
Langues : français dominant (13 morceaux), anglais teinté de patois jamaïcain (8 morceaux), espagnol (1 morceau). Registre C de la bible (radio civile, décalage entre un ton chaud et un monde qui mord), version quartier : chaleur, famille, débrouille, conscience sociale douce. Le Mur est doré, et alors. Le Voss est une rumeur qu'on entend au Port, jamais une adhésion. ICLabs n'est pas attaqué de front : on constate, on sourit, on recycle.
Artistes pastiches déjà établis par le jeu et prolongés ici : Bob Marcly, Damia Marcly, Big Roh. Nouveaux pastiches de la station : Peter Torch, Tiken Jah Faraday, Alpha Blondix, Lee Patch Perry, Snoop Drone, Dub Ink, Sir Shaggo, Sean Pulse, MC Solar, Akhen Atom, Kool Shell & Joey Storm, Orelsun, B2-Orbit, Frères Litchout, Coolant, Slim Circuit, Notorious C.Y.B., Magic Sistem, Cultura Protésica.
La banque de clins d'oeil réservée à Yellow Roots (brief §8) et l'état de consommation de chaque titre détourné sont listés dans les notes d'intégration, en fin de fichier. « One Core » et « Get Up, Reboot » sont utilisés en jingles chantés.
Règles appliquées à tout le fichier : aucun tiret cadratin ni demi-cadratin, aucun nom d'artiste réel dans un prompt Suno, paroles 100 % originales, Crédits et Matière jamais confondus, aérotram par sauts, aucune fréquence chiffrée, le Voss et rien d'autre.

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | No Cyborg No Cry | Bob Marcly | FR (refrain EN) | 3:50 | reggae roots, one drop | Bob Marley, No Woman No Cry (Jamaïque) |
| 02 | Three Little Drones | Bob Marcly | EN (patois) | 3:00 | reggae roots ensoleillé | Bob Marley, Three Little Birds (Jamaïque) |
| 03 | Recovery Song | Damia Marcly | EN | 3:30 | folk reggae acoustique | Bob Marley, Redemption Song (Jamaïque) |
| 04 | Sangline Soldier | Peter Torch | EN (patois) | 3:40 | roots reggae militant, steppers | Bob Marley, Buffalo Soldier ; nom d'artiste : Peter Tosh (Jamaïque) |
| 05 | Plus rien ne me tue | Tiken Jah Faraday | FR | 4:00 | reggae roots ouest-africain | Tiken Jah Fakoly, Plus rien ne m'étonne (Côte d'Ivoire) |
| 06 | IC Police | Alpha Blondix | FR | 3:50 | reggae roots, cuivres, choeur | Alpha Blondy, Brigadier Sabari (Côte d'Ivoire) |
| 07 | Recycle It Up (Downroad Dub) | Lee Patch Perry feat. Snoop Drone | EN (toasting) | 5:20 | dub quasi instrumental, toasting | Bob Marley, Stir It Up, en dub à la manière de Lee « Scratch » Perry (Jamaïque) ; toasting : la période reggae de Snoop (USA) |
| 08 | Tout ce qu'ils veulent (c'est ta Matière) | Dub Ink | FR (hook EN) | 4:10 | dub reggae français, cuivres | Dub Inc, Tout ce qu'ils veulent (France) |
| 09 | C'était pas moi (c'était mon corps d'avant) | Sir Shaggo | FR (répons EN) | 3:30 | dancehall pop, duo comique | Shaggy, It Wasn't Me (Jamaïque) |
| 10 | Température (Iron Dee) | Sean Pulse | EN (patois, refrain FR) | 3:20 | dancehall 2000s, club | Sean Paul, Temperature (Jamaïque) |
| 11 | Jamming (Costa Rive) | Big Roh | FR (patois) | 3:15 | ragga dancehall français | Bob Marley, Jamming (Jamaïque), en ragga français |
| 12 | Sangline | MC Solar | FR | 4:00 | boom bap jazzy 90s | MC Solaar, Caroline (France) |
| 13 | Je danse le Tram | Akhen Atom | FR | 3:40 | boom bap funk 90s | IAM, Je danse le Mia (France) |
| 14 | Laisse pas traîner ton core | Kool Shell & Joey Storm | FR | 3:50 | boom bap hardcore 90s | NTM, Laisse pas traîner ton fils (France) |
| 15 | La Terre est un mur | Orelsun | FR | 4:30 | rap mélancolique, piano | Orelsan, La terre est ronde (France) |
| 16 | Duc de Downroad | B2-Orbit | FR | 3:30 | trap sombre, 808 | Booba, le Duc de Boulogne (France) |
| 17 | Le Mur ou rien | Frères Litchout | FR | 4:00 | cloud rap, autotune | PNL, Le monde ou rien (France) |
| 18 | Cyborg's Paradise | Coolant | EN | 4:00 | hip-hop 90s, choeur gospel | Coolio, Gangsta's Paradise (USA) |
| 19 | Lose Your Core | Slim Circuit | EN | 4:10 | rap tendu, guitare, 2000s | Eminem, Lose Yourself (USA) |
| 20 | Matière | Notorious C.Y.B. | EN | 3:50 | hip-hop East Coast 90s, soul | Notorious B.I.G., Juicy (USA) |
| 21 | 1er Cycle | Magic Sistem | FR | 3:30 | zouglou, afro-pop dansante | Magic System, 1er Gaou (Côte d'Ivoire) |
| 22 | Cuerpo nuevo, misma playa | Cultura Protésica | ES (mots EN) | 3:20 | reggae en espagnol, dembow doux, zouk | Cultura Profética (Porto Rico), reggae hispanophone, crossover reggaeton léger |

#### Traverse (Traverse)

Fichier : `Documentation/Suno/04_Traverse.md` (61 Ko). Morceaux : 22 ; jingles : 5 ; interventions ou messages : 5 ; publicités ou annonces : 6.

> Station de radio du jeu QANGA. Contenu éditorial pour Suno : prompts de style et paroles complètes.
> Règles de fabrication : `Documentation/Suno/00_BRIEF_COMMUN.md`. Canon : `Documentation/QANGA_LORE_BIBLE.md`.
> Aucun asset ni code du projet n'est modifié par ce fichier.

**StationId proposé** : `Traverse` (à valider par Benja, le nom devient un contrat dès qu'il entre au catalogue).

**Ce qu'est la station.** La radio de la nuit de la Capitale. Elle porte le nom du quartier de la Traverse, qui sature dès que la lumière tombe, et elle couvre le Centre, Downroad, le Port et le CineVortex ouvert 24 heures sur 24. House, French touch, techno, trance, eurodance, synthwave, electro. Elle ne s'arrête jamais vraiment : à l'aube elle passe la main au Mur doré, et le cycle recommence.

**Le ton.** Froid, élégant, hypnotique. Un humour pince-sans-rire sur les corps neufs et les nuits qui ne finissent pas : la formule de la station est « la nuit ne se termine pas, elle se recycle ». On ne pleure personne à l'antenne : quelqu'un qui tombe rate la fin du set, voilà tout, il rouvrira les yeux à la tour et il reviendra danser au prochain cycle.

**Langues.** Anglais en dominante, trois morceaux en allemand (clin d'oeil à Berlin, qui est aussi le nom du constructeur des véhicules de police du jeu), un en français, trois instrumentaux.

**Animatrice proposée** : **Vera Volt**. Voix basse, sensuelle et détachée, parle peu, en français et en anglais, un mot d'allemand de temps en temps. Deux noms alternatifs dans la nomenclature du jeu, au choix de Benja : **Nyx Pulsar** ou **Lena Kelvin**.

**Réception proposée** : émetteur local sur la Capitale, plein jusqu'à 150 km, silence au-delà de 350 km (valeurs de départ, à régler en jeu).

---

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Around the Wall | Daft Prox | EN | 4:20 | French touch, filtre disco | d'après « Around the World » (France) |
| 02 | One More Cycle | Daft Prox | EN | 5:20 | French touch, vocodeur | d'après « One More Time » (France) |
| 03 | Harder Better Newer Body | Daft Prox | EN | 3:40 | French touch, voix robotique | d'après « Harder, Better, Faster, Stronger » (France) |
| 04 | Get Synced | Daft Prox et Nile Rotor | EN | 4:30 | disco house, guitare funk | d'après « Get Lucky » (France) |
| 05 | Music Sounds Better With Core | Stardust Relay | EN | 4:10 | french house, boucle disco | d'après « Music Sounds Better With You » (France) |
| 06 | Lady Vera | Modjovial | EN | 4:00 | french house filtrée | d'après « Lady (Hear Me Tonight) » (France) |
| 07 | S.Y.N.C. | Injustice | EN | 3:30 | electro house, choeur d'enfants | d'après « D.A.N.C.E. » (France) |
| 08 | Aerotram | Werk 7 | DE | 6:00 | techno de Berlin, motorik minimal | d'après « Autobahn » (Allemagne) |
| 09 | Die Autonomen | Werk 7 | DE | 4:00 | electro-pop robotique | d'après « Die Roboter » (Allemagne) |
| 10 | Trans-Relais Express | Werk 7 | EN | 5:00 | electro motorik, train synthétique | d'après « Trans-Europe Express » (Allemagne) |
| 11 | Modell V2 | Werk 7 | DE | 3:40 | synthpop froide | d'après « Das Modell » (Allemagne) |
| 12 | Corestarter | The Progeny | EN | 3:40 | big beat, breakbeat rageur | d'après « Firestarter » (Royaume-Uni) |
| 13 | Galvanize (Titanium) | Chemical Sisters | EN | 4:00 | big beat, cordes orientales | d'après « Galvanize » (Royaume-Uni) |
| 14 | Flat Core | Monsieur Oizeau | Instrumental | 3:00 | electro minimal, basse jaune | d'après « Flat Beat » (France) |
| 15 | Born Synced | Underworks | EN | 5:30 | progressive house, flux parlé | d'après « Born Slippy » (Royaume-Uni) |
| 16 | Insomnia (Cryo Mix) | Faithlesser | EN | 5:40 | trance progressive, spoken word | d'après « Insomnia » (Royaume-Uni) |
| 17 | Children of the Wall | Roberto Milles | Instrumental | 4:30 | dream trance, piano | d'après « Children » (Italie) |
| 18 | Wake Me Up (2755) | Avicci Nova | EN | 4:00 | folk house, banjo et drop | d'après « Wake Me Up » (Suède) |
| 19 | Don't You Worry Cyborg | Swedish Relay Mafia | EN | 3:50 | progressive house anthem | d'après « Don't You Worry Child » (Suède) |
| 20 | L'ancrage toujours | Gigi D'Ancrage | FR | 4:00 | italo dance, piano en boucle | d'après « L'amour toujours » (Italie) |
| 21 | How Much Is the Fer | Scootair | EN | 3:40 | happy hardcore, voix criée | d'après « How Much Is the Fish? » (Allemagne) |
| 22 | Sand Digger | Darudé | Instrumental | 3:50 | trance de festival, lead perçant | d'après « Sandstorm » (Finlande) |

#### Vieux Monde (VieuxMonde)

Fichier : `Documentation/Suno/05_VieuxMonde.md` (63 Ko). Morceaux : 20 ; jingles : 4 ; interventions ou messages : 6 ; publicités ou annonces : 4.

La station des restes de l'Ancien Monde. Folk, country, chanson francaise, blues acoustique, bluegrass, chanson italienne et espagnole, guitare seule. Ici on ne chante pas ce qu'on a perdu, on chante ce qu'on a garde : un vieux terminal qui s'allume encore, une piece de moteur qu'on revend au comptoir d'echange, une photo qui a traverse le sommeil.

C'est la radio qu'on ecoute au garage, dans une Sablone qui refroidit, au comptoir d'echange pendant qu'un preneur pese votre ferraille, ou dans une salle de recuperation quand le corps est neuf et que la memoire, elle, a encore le gout d'avant 2180. Les cyborgs de premiere generation l'ecoutent parce qu'elle parle leur langue ; les autres l'ecoutent parce qu'elle raconte d'ou vient tout ce metal.

Le ton est une nostalgie tendre, jamais plaintive. Le monde a change, l'economie continue, la mort n'est plus grave : donc on a le droit de sourire en chantant les vieilles choses. Beaucoup de guitares seules, peu de batterie, des harmonicas, un dobro, un accordeon, des choeurs a deux voix, de la bande analogique.

Langues : francais (7 morceaux), anglais (6), espagnol (5 dont 2 traditionnels revisites), italien (2). Deux ballades longues (5 minutes) pour les fins de cycle.

**Animateur propose : Gus Lumen.** Cyborg de premiere generation (V1), voix qui gresille comme un vieux haut-parleur, collectionneur compulsif. Entre deux morceaux il pose un objet de l'Ancien Monde sur la table et raconte d'ou il vient. Il ne vend jamais rien, il montre.

**Alternative a arbitrer par Benja : Tom Ohm**, PNJ existant du jeu (cyborg Gen-1 hors reseau, collectionneur). Il collerait parfaitement a cette station, mais c'est un personnage deja ecrit ailleurs : le prendre comme animateur l'engage. Decision de Benja.

**Troisieme nom, neuf, si aucun des deux ne convient : Abel Cuivre** (meme profil, meme grain de voix, aucun passif dans le jeu).

| N | Titre | Artiste | Langue | Duree cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Blowin' in the Sand | Bob Dunes | EN | 3:10 | folk protest acoustique | d'apres « Blowin' in the Wind » (USA) |
| 02 | Le silence du relais 77 | Lemaire et Sandoz | FR | 3:30 | folk duo, deux voix | d'apres « The Sound of Silence » (USA) |
| 03 | Ring of Iron | Hank Cinder | EN | 2:45 | country vintage, contrebasse | d'apres « Ring of Fire » (USA) |
| 04 | Murito Lindo | Los Tres Reles | ES | 3:00 | traditionnel mexicain, trio guitares | d'apres « Cielito Lindo » (Mexique, traditionnel) |
| 05 | Les cyborgs d'abord | Georges Brassure | FR | 3:20 | chanson francaise, guitare contrebasse | d'apres « Les copains d'abord » (France) |
| 06 | Take Me Home, Relay Roads | Danny Weaver | EN | 3:25 | country folk, banjo violon | d'apres « Take Me Home, Country Roads » (USA) |
| 07 | Desincronizado | Manu Chava | ES | 3:15 | rumba acoustique latino | d'apres « Clandestino » (France / Espagne) |
| 08 | Non, je ne regrette rien (de l'Ancien Monde) | Edith Palma | FR | 2:50 | chanson realiste, accordeon | d'apres « Non, je ne regrette rien » (France) |
| 09 | Quarry Stone Blues | Red Marlow | EN | 3:05 | country blues, train beat | d'apres « Folsom Prison Blues » (USA) |
| 10 | La canzone di Marinetta | Fabrizio De Ferro | IT | 3:45 | ballade italienne, guitare et cordes | d'apres « La canzone di Marinella » (Italie) |
| 11 | Matiere gagnante | Remi Gavroche | FR | 3:35 | chanson francaise, accordeon guitare | d'apres « Mistral gagnant » (France) |
| 12 | Torre de Rele | Los Guajiros de Rele | ES | 3:10 | guajira cubaine, tres et guitare | d'apres « Guantanamera » (Cuba, traditionnel) |
| 13 | Hotel Capitale | The Harriers | EN | 5:20 | folk rock acoustique, long solo | d'apres « Hotel California » (USA) |
| 14 | Ne me recycle pas | Jacques Verlain | FR | 3:40 | chanson dramatique, ondes et guitare | d'apres « Ne me quitte pas » (Belgique) |
| 15 | Ojala (que despiertes) | Silvio Marea | ES | 3:30 | trova cubaine, guitare seule | d'apres « Ojala » (Cuba) |
| 16 | La Cryo | Charles Amaris | FR | 3:50 | valse chanson, guitare et cordes | d'apres « La Boheme » (France) |
| 17 | Wish You Were Synced | Grey Halo | EN | 5:00 | folk rock lent, slide guitar | d'apres « Wish You Were Here » (Royaume-Uni) |
| 18 | Caruso (Costa Rive) | Lucio Marenco | IT | 4:00 | ballade napolitaine, piano accordeon | d'apres « Caruso » (Italie) |
| 19 | Me gusta la chatarra | Manu Chava | ES | 2:55 | latin acoustique, guitare et trompette | d'apres « Me gustas tu » (France / Espagne) |
| 20 | Je l'aime a mourir (ce n'est plus grave) | Francois Cabral | FR | 3:30 | ballade francaise, guitare picking | d'apres « Je l'aime a mourir » (France) |

#### AMBRE, la fréquence du Voss (VossAmbre)

Fichier : `Documentation/Suno/06_VossAmbre.md` (67 Ko). Morceaux : 20 ; jingles : 4 ; interventions ou messages : 6 ; publicités ou annonces : 6.

Station clandestine, nouvelle. Elle n'est recevable qu'avec le module Interception radio (design : rien à faire côté moteur dans ce livrable). Elle émet depuis les sites tenus par le Voss et les camps rebelles du désert : signal faible, intermittent, souvent enfoui sous le souffle de bande et le crépitement d'un relais détourné. Le nom vient de la couleur d'interface du Voss. La station ne publie pas à côté d'IC Labs : elle publie par-dessus. Elle reprend les slogans peints des marques et les retourne.

Ton : ce n'est pas une bande de pillards. C'est une position argumentée, froide, patiente, parfois belle. La doctrine est le Principe de Correction : « L'évolution corrige toujours ses erreurs. Et l'humanité pourrait être l'une d'elles. » Les Sanglines y sont lues comme une réponse possible de la biosphère. Trois objectifs, répétés sans hausser le ton : empêcher la renaissance incontrôlée de l'humanité biologique ; bâtir une civilisation hybride conscience + machine ; rendre l'équilibre écologique à la Terre sous supervision artificielle. Le Voss rassemble des Autonomes rebelles, des cyborgs V1 rejetés, des IA militaires abandonnées et des humains volontaires ; il est en paix avec la Dissidence et les Pirates. Son matériel est puissant mais instable, et la station en fait une fierté : instable, comme tout ce qui est vivant. Le Dr Cael Veyron est cité avec respect, rarement, jamais en icône.

Animateur : **la Voix AMBRE**, anonyme, vocodée, sans genre affirmé, calme. Indicatifs alternatifs proposés : « **CORRECTION** » (indicatif court, scandé par le choeur des jingles) et « **le Correcteur** » (quand la Voix se nomme à la troisième personne, en fin de message).

Langues : FR, EN, RU (cyrillique), DE. Registre : industrial, EBM, post-punk, dark techno, doom, dark folk, coldwave. Pas de publicité : des contre-publicités, par-dessus les slogans d'IC Labs. Pas de fréquence annoncée. Jamais d'autre nom pour la faction que le Voss.

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Du hast (Materie) | Anthrazit | DE | 3:40 | metal industriel, NDH | d'après « Du hast » (Allemagne) |
| 02 | Autonom | Anthrazit | DE | 4:00 | metal industriel, ballade lourde | d'après « Engel » (Allemagne) |
| 03 | Core Like a Hole | Ninth Core | EN | 4:00 | industrial rock | d'après « Head Like a Hole » (États-Unis) |
| 04 | The Corrected People | Merrill Mainframe | EN | 3:30 | industrial rock, stomp | d'après « The Beautiful People » (États-Unis) |
| 05 | Enjoy the Correction | Correction Mode | EN | 3:50 | EBM, synth-pop sombre | d'après « Enjoy the Silence » (Royaume-Uni) |
| 06 | Personal Doctor | Correction Mode | EN | 3:45 | EBM, blues électronique | d'après « Personal Jesus » (Royaume-Uni) |
| 07 | Correction | Mogadishu | EN (choeur DE) | 4:00 | martial industrial, choeur | d'après « Tanz mit Laibach » (Slovénie) |
| 08 | Drones on Parade | Rage Against the Relay | EN | 3:50 | dark techno, rap industriel | d'après « Bulls on Parade » (États-Unis) |
| 09 | Sync Will Tear Us Apart | Sync Division | EN | 3:30 | post-punk, synthé glacé | d'après « Love Will Tear Us Apart » (Royaume-Uni) |
| 10 | A Nest | No Remedy | EN | 5:20 | post-punk, long outro | d'après « A Forest » (Royaume-Uni) |
| 11 | Коррекции! (Korrektsii!) | Кадр 7 (Kadr 7) | RU | 3:50 | post-punk russe | d'après « Перемен » (Russie) |
| 12 | Связанные одной сетью (Svyazannye odnoy setyu) | Аммонит (Ammonit) | RU | 4:00 | post-punk russe, orgue froid | d'après « Скованные одной цепью » (Russie) |
| 13 | Master of Anchors | Materika | EN | 5:50 | thrash doom | d'après « Master of Puppets » (États-Unis) |
| 14 | Flying Sanglines | Ossature | EN | 4:00 | doom progressif | d'après « Flying Whales » (France) |
| 15 | Cryo Suey | System of a Sleep | EN | 3:40 | metal alternatif | d'après « Chop Suey! » (États-Unis) |
| 16 | Nothing Hurts Anymore | Ninth Core | EN | 3:50 | dark folk, drone | d'après « Hurt » (États-Unis) |
| 17 | Всё идёт по протоколу (Vsyo idyot po protokolu) | Гражданская Коррекция (Grazhdanskaya Korrektsiya) | RU | 3:30 | dark folk punk, lo-fi | d'après « Всё идёт по плану » (Russie) |
| 18 | Salle de réveil | Bérurier Ambre | FR | 3:10 | coldwave punk, boîte à rythmes | d'après « Porcherie » (France) |
| 19 | Antisync | Méfiance | FR | 3:50 | coldwave, guitare froide | d'après « Antisocial » (France) |
| 20 | Sleeping in the Name | la Voix AMBRE | FR (refrain EN) | 4:00 | spoken word sur drone, manifeste | d'après « Killing in the Name » (États-Unis) |

#### Radio Centinela (RadioCentinela)

Fichier : `Documentation/Suno/07_RadioCentinela.md` (73 Ko). Morceaux : 22 ; jingles : 5 ; interventions ou messages : 6 ; publicités ou annonces : 6.

Radio Centinela est la radio latine de la Capitale. Elle émet depuis une terrasse du quartier de Centinela, face au Mur doré, et rayonne sur Costa Rive (la plage), le Port et les lignes de Tamil Station qui descendent vers la mer. C'est la station de la fête et de la famille : on y danse avec un corps neuf, on y rit de la mort parce qu'on se réveille à la Torre de Relé, et « el ciclo » (le cycle, la journée du jeu) rythme tout, de l'ouverture sur la plage à la fermeture de nuit.

Animateur : **Rafa Cometa**. Cri de ralliement : « ¡Buenos ciclos, cyborgs! ». Énergique, chaleureux, il tutoie tout le monde (auditeurs, IC Policía, Sanglines comprises), parle espagnol avec des éclats de français et d'anglais (« ¡Bonjour cyborg! », « stay frosty, mi gente », « Keep the Flux »). Alternatives dans la nomenclature du projet (prénom court + nom techno ou astronomique en espagnol) : **Nacho Vector** (voix masculine plus grave, plus posée) ou **Lola Neutrino** (voix féminine, même énergie). J'évite « Órbita » en patronyme d'animateur pour ne pas télescoper Radio Orbita (station 08).

Langues : espagnol pour 20 morceaux, avec trois couleurs selon le genre : caribéenne (reggaeton, salsa, bachata : Porto Rico, Cuba, République dominicaine), mexicaine (cumbia, ranchera, bolero) et espagnole d'Espagne (flamenco pop, rumba). Plus deux morceaux en portugais brésilien (sertanejo, forró). Jingles, interventions et publicités sont en espagnol, avec les éclats bilingues de Rafa.

Registre : registre C de la bible (radio civile, décalage entre la fête et un monde qui mord), version chaleureuse et familiale. La blague signature vue de Centinela : « el cuerpo es nuevo, la carga no vuelve sola ». Vocabulaire du monde en espagnol, à réutiliser tel quel : la Torre de Relé, la sala de recuperación, anclar la firma, la Materia (jamais confondue avec los Créditos), el aerotram de torre en torre (jamais de vuelo directo), la Sablone, el taxi Melrose, las Sanglines (féminin : una Sangline), la IC Policía, el ciclo, ICLI Station en órbita, MarsOne (rumeur : les Martiens se méfient des Terriens), el Puesto de Intercambio (le comptoir d'échange), el Viejo Mundo, el Muro dorado.

Place dans le monde : la station des quartiers populaires du bord de mer, complémentaire d'I-C News Radio (la voix officielle) et de HitWall (la pop mondiale). Elle ne parle pas au nom d'ICLabs, elle parle au nom du barrio. Les 22 marques citées en pub sont celles du canon ; six sont couvertes ici (Chez Fred, Starkito, Costa Riv, Sola, Melrose, Cope Mutter).

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Despacito (por el Velo) | Luis Fotón ft. Papi Yanqui | ES | 3:45 | reggaeton pop, guitare | Despacito (Porto Rico) |
| 02 | Bailando en Costa Rive | Enrique Ignis ft. Gente del Muro | ES | 3:50 | latin pop, guitare flamenca | Bailando (Espagne / Cuba) |
| 03 | Materia | Papi Yanqui | ES | 3:10 | reggaeton old school | Gasolina (Porto Rico) |
| 04 | Vivir mi ciclo | Marco Antena | ES | 4:00 | salsa, cuivres, montuno | Vivir mi vida (États-Unis / Porto Rico) |
| 05 | Livin' la Vida Cyborg | Ricky Marte | ES (+ EN) | 3:35 | latin pop rock, cuivres | Livin' la Vida Loca (Porto Rico) |
| 06 | La Bomba (Iron Dee) | Richi Valente y los Sonideros de Iron Dee | ES | 3:00 | cumbia sonidera rock | La Bamba (Mexique / États-Unis) |
| 07 | Sincronización | Romeo Santelmo y Grupo Avería | ES | 4:00 | bachata urbaine | Obsesión (République dominicaine / États-Unis) |
| 08 | Chips Don't Lie | La Kira ft. Wyclo | ES (+ EN) | 3:30 | reggaeton, trompette | Hips Don't Lie (Colombie) |
| 09 | Anclarena | Los del Muro | ES | 3:40 | rumba pop dance | Macarena (Espagne) |
| 10 | Bamboléo (Sablone) | Los Reyes de la Duna | ES | 3:25 | rumba flamenca pop | Bamboléo, Gipsy Kings (France / Espagne) |
| 11 | Danza Sanguina | Don Ohmar ft. Lucênio | ES (+ PT) | 3:20 | reggaeton kuduro | Danza Kuduro (Porto Rico / Portugal) |
| 12 | Oye cómo va el tram | Tito Viaducto y la Orquesta Tamil | ES | 5:20 | salsa cha-cha-chá, latin rock | Oye cómo va (Porto Rico / États-Unis) |
| 13 | La chapa negra | Juancho Vector | ES | 3:35 | cumbia pop, accordéon | La camisa negra (Colombie) |
| 14 | Warp Warp | La Kira | ES (+ EN) | 3:30 | afro-latin pop, choeur | Waka Waka (Colombie) |
| 15 | Tusa (cuerpo nuevo) | Karol Gravedad | ES | 3:20 | trap latino | Tusa (Colombie) |
| 16 | Malamente (Voss) | La Rosa de Ámbar | ES | 3:05 | flamenco pop trap | Malamente (Espagne) |
| 17 | Mi gente de la Torre | Jota Vector ft. Willy Wattio | ES (+ FR) | 3:15 | reggaeton moombahton | Mi gente (Colombie / France) |
| 18 | Zona de prueba | Calle Diecinueve | ES | 4:45 | cumbia andina, rap | Latinoamérica (Porto Rico) |
| 19 | El rey del ciclo | Jorge Neutrino y el Mariachi Centinela | ES | 3:40 | ranchera mariachi | El rey (Mexique) |
| 20 | Sincronízame mucho | Marisol Quásar | ES | 3:45 | bachata bolero | Bésame mucho (Mexique) |
| 21 | Ai se eu te sincronizo | Michel Telúrio | PT-BR | 2:50 | sertanejo universitário | Ai se eu te pego (Brésil) |
| 22 | Mas que Matéria | Sérgio Maré e o Trio Litoral | PT-BR | 3:15 | forró samba | Mas que nada (Brésil) |

#### Радио Орбита / Radio Orbita (RadioOrbita)

Fichier : `Documentation/Suno/08_RadioOrbita.md` (90 Ko). Morceaux : 20 ; jingles : 5 ; interventions ou messages : 6 ; publicités ou annonces : 6.

Радио Орбита est la radio des habitants d'ICLISpace : les communautés humaines qui vivent en orbite, autour et à bord d'**ICLI Station** (les trois comptoirs d'armure, de médicaments et d'armement, la station de warp, les baies d'amarrage saturées). Émise depuis la station, elle regarde la Terre par le hublot : le Mur doré de Yellow Wall City au milieu de l'océan, les Tours de Relais du désert de Dire Dawa, l'herbe qu'on ne touche plus. Héritage cosmonaute assumé : « Поехали! », la Terre vue du hublot, la nostalgie de la surface, l'ennui des files d'amarrage, la méfiance des Martiens de MarsOne.

**Animatrice** : **Nika Sputnik** (« Говорит Орбита »). Voix posée, deadpan, drôle sans jamais le montrer. Parle russe, avec quelques mots d'anglais technique (warp, docking, low Matter, dispatcher, Keep the Flux). Alternatives dans la nomenclature du monde (prénom court + nom techno ou astronomique) : **Vika Proton** ou **Dasha Apogey**.

**Langues** : russe en cyrillique pour 17 morceaux, anglais pour 3 (le rave « Skibidi Cyborg », chanté en anglais avec des cris de foule en russe ; la pop orbitale « Элемент ядра / Core Element » ; le rock « Orbita Calling »). Jingles, interventions et publicités en russe ; les slogans peints restent en anglais tels quels ou sont traduits.

**Registre** : chaleur nordique, humour pince-sans-rire, mélancolie de l'orbite. En bas le monde mord (Sanglines, nids, contrats en retard) ; en haut on attend son tour de stykovka avec un café. La blague signature de la station, version orbitale : le corps est neuf, la cargaison est restée dans la soute d'un Velkara qui, lui, est redescendu sans vous.

**Genres** : estrada / pop soviétique, russian rock, post-punk russe, hardbass et rave, ballades, synthpop, chanson à guitare, un choeur.

**Place dans le monde** : émetteur de type Space centré sur ICLI Station (voir Notes d'intégration). On la capte en orbite et en approche ; en bas, elle n'est plus qu'un souvenir qui grésille entre deux tours.

**Vocabulaire russe du monde utilisé dans ce fichier** : Башня Реле / релейная башня (Tour de Relais), зал восстановления (salle de récupération), ядро (le noyau), Материя (la Matière, jamais confondue avec les кредиты), Санглина / Санглины (les Sanglines, féminin), летучая Санглина (Flyer), стыковка / очередь на стыковку (amarrage / file d'amarrage), варп (le warp), аэротрам (l'aérotram) et son argot de station « рама » (une rame), Стена / Золотая Стена (le Mur doré), Столица (la Capitale), цикл (le cycle), заякорить подпись (ancrer sa signature).

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Трава у Башни (Trava u Bashni) | ВИА «Орбитяне» | RU | 3:40 | estrada rock soviétique, VIA 80s | Земляне, « Трава у дома » (URSS) |
| 02 | Миллион алых дронов (Million alykh dronov) | Галина Гравитова | RU | 3:50 | ballade estrada 80s, piano, cordes | Алла Пугачёва, « Миллион алых роз » (URSS) |
| 03 | Орбитальные вечера (Orbital'nye vechera) | Иван Хаблов и Оркестр Орбиты | RU | 3:30 | estrada lente, valse, accordéon | « Подмосковные вечера » (URSS) |
| 04 | Катюша ждёт у башни (Katyusha zhdyot u bashni) | Хор Орбитального Причала | RU | 3:10 | estrada avec choeur, marche folk | « Катюша » (traditionnel russe) |
| 05 | Орбита 2755 (Orbita 2755) | Мумий Дрон | RU | 3:20 | russian rock, rockapops 90s | Мумий Тролль, « Владивосток 2000 » (Russie) |
| 06 | Что такое цикл (Chto takoye tsikl) | Юра Шлюзов и группа ДЦТ | RU | 4:10 | russian rock ballad, violon | ДДТ, « Что такое осень » (Russie) |
| 07 | Я свободен (Ya svoboden) | Валера Варпов | RU | 5:10 | heavy rock, hymne du warp | Кипелов, « Я свободен » (Russie) |
| 08 | Экзоскелет (Ekzoskelet) | Орбитград | RU | 3:15 | ska-punk festif, cuivres | Ленинград, « Экспонат » (Russie) |
| 09 | Станция по имени Орбита (Stantsiya po imeni Orbita) | ХОД | RU | 4:00 | post-punk russe 80s | Кино, « Звезда по имени Солнце » (URSS) |
| 10 | Шлюз (Shlyuz) | Молчат Шлюзы | RU | 3:40 | post-punk, cold wave | Молчат Дома, « Судно » (Biélorussie) |
| 11 | Выход есть (Vykhod yest') | Сплайн | RU | 4:20 | post-punk ballad, sombre | Сплин, « Выхода нет » (Russie) |
| 12 | Калинка (rave) (Kalinka) | DJ Стыковка | RU | 3:00 | hardbass, rave folk | « Калинка » (traditionnel russe) |
| 13 | Skibidi Cyborg | Tiny Huge | EN (+ RU) | 2:50 | rave-pop absurde, hardbass | Little Big, « Skibidi » (Russie) |
| 14 | Нежность (Nezhnost') | Ольга Тишина | RU | 4:00 | ballade des cosmonautes, piano | Пахмутова, « Нежность » (URSS) |
| 15 | Дорогой длинною, от башни к башне (Dorogoy dlinnoyu, ot bashni k bashne) | Нина Дальняя и трио Рельс | RU | 3:45 | romance russe, guitare, violon | « Дорогой длинною » (romance russe) |
| 16 | Рамы привередливые (Ramy priveredlivye) | Семён Хриплов | RU | 3:50 | chanson à guitare, voix rauque | Высоцкий, « Кони привередливые » (URSS) |
| 17 | Нас не догонят (Sangline mix) (Nas ne dogonyat) | в.А.к.У.у.М. | RU | 3:30 | synthpop, eurodance 2000s | t.A.T.u., « Нас не догонят » (Russie) |
| 18 | Крошка моя V2 (Kroshka moya V2) | Руки Вниз | RU | 3:20 | synthpop, eurodance 90s | Руки Вверх!, « Крошка моя » (Russie) |
| 19 | Элемент ядра / Core Element (Element yadra) | Vitaly Ultra | EN | 3:30 | pop orbitale, techno-pop, falsetto | Витас, « 7th Element » (Russie) |
| 20 | Orbita Calling | Orbit Park | EN | 4:10 | glam hard rock 1989 | Gorky Park, « Moscow Calling » (Russie) |

#### Forge FM (ForgeFM)

Fichier : `Documentation/Suno/09_ForgeFM.md` (62 Ko). Morceaux : 22 ; jingles : 5 ; interventions ou messages : 6 ; publicités ou annonces : 6.

La radio du métal chaud. Forge FM émet depuis un local sans fenêtre accroché au flanc de la Raffinerie **Iron Dee**, entre deux coulées, et se capte partout où quelqu'un casse de la pierre : la carrière de **Quarry Stone**, l'usine **Rost-Stahl**, les pistes de convoyage vers la **Tour de Relais 77**, les garages des tours, les bennes qui remontent du Fer, de l'Obsidienne, du Silicium, du Cuivre et de l'Aluminium.

C'est une radio d'équipe de quart. On y parle fort parce que la machine est plus forte. Le ton, c'est la gueule, la sueur et le rire gras : ici la mort n'est pas un drame, c'est une pause technique. Tu te fais ouvrir par une Sangline sur le carreau de la carrière, tu te réveilles en salle de récupération, tu remets un corps neuf au boulot et tu retournes chercher ta benne. Ce qui se pleure, ce n'est jamais le corps : c'est la cargaison.

Fierté ouvrière assumée, zéro misérabilisme. La Capitale vit dans l'or du Mur, très bien : quelqu'un a fondu ce béton et quelqu'un a coulé cet or. Les cyborgs V1 ont bâti ce Mur, Forge FM ne laisse personne l'oublier.

**Sponsors d'antenne** : **Iron Dee** (raffinerie et véhicules) et **Titanium ICLI** (matériaux). Passent aussi : A.D Store, McFrailenergy, Melrose, Hardware Store.

**Animateur** : **Rex Carbone**, voix rauque, rit fort, tutoie tout le monde, parle français en balançant des expressions anglaises sur les fins de phrase. Cri d'antenne : « Forge FM, on chauffe le Fer ! ».
**Deux alternatives dans la nomenclature** : **Vic Scoria** et **Dan Vulcain**.

**Langues** : EN (12), FR (4), DE (2), ES (2), plus 2 instrumentaux.
**Répartition de genres** : hard rock, heavy metal, punk, grunge, stoner et desert rock, rock français, rock allemand, rock espagnol. Les titres non anglophones portent eux aussi un genre de la famille (le punk allemand compte dans le punk, le hard rock espagnol dans le hard rock).

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Highway to Wall | Iron Voltage | EN | 3:20 | hard rock riff | d'après « Highway to Hell » (Australie) |
| 02 | Back in Body | Iron Voltage | EN | 3:25 | hard rock stomp | d'après « Back in Black » (Australie) |
| 03 | Sweet Sangline o' Mine | Gunner Roze | EN | 3:55 | hard rock mélodique | d'après « Sweet Child o' Mine » (USA) |
| 04 | Entre dos torres | Héroes del Relevo | ES | 3:50 | hard rock espagnol | d'après « Entre dos tierras » (Espagne) |
| 05 | Enter Sangline | Metalflux | EN | 4:00 | heavy metal | d'après « Enter Sandman » (USA) |
| 06 | Nothing Else Matter | Metalflux | EN | 5:10 | ballade heavy metal | d'après « Nothing Else Matters » (USA) |
| 07 | Iron Dee Man | Obsidian Crown | EN | 3:40 | heavy metal doom | d'après « Iron Man » (Royaume-Uni) |
| 08 | Run to the Tower | Iron Relay | EN | 3:30 | heavy metal galopant | d'après « Run to the Hills » (Royaume-Uni) |
| 09 | Sangline Bop | The Rebootz | EN | 2:30 | punk 1977 | d'après « Blitzkrieg Bop » (USA) |
| 10 | Cryo Case | Green Slag | EN | 2:50 | punk mélodique | d'après « Basket Case » (USA) |
| 11 | Zyklen wie diese | Die Rostbrüder | DE | 3:30 | punk rock allemand | d'après « Tage wie diese » (Allemagne) |
| 12 | Smells Like Sangline Spirit | Grey Static | EN | 3:45 | grunge | d'après « Smells Like Teen Spirit » (USA) |
| 13 | Le Voile nous portera | Noir Relais | FR | 3:40 | rock alternatif français | d'après « Le vent nous portera » (France) |
| 14 | Infected | Obsidian Crown | EN | 3:35 | stoner doom | d'après « Paranoid » (Royaume-Uni) |
| 15 | Smoke on the Refinery | Deep Furnace | EN | 3:40 | stoner heavy blues | d'après « Smoke on the Water » (Royaume-Uni) |
| 16 | Un Ancien Monde | Téléscope | FR | 3:30 | rock français 80s | d'après « Un autre monde » (France) |
| 17 | Le Convoyeur | Indochrome | FR | 3:35 | rock français new wave | d'après « L'aventurier » (France) |
| 18 | Allumer la forge | Jo Brasier | FR | 3:45 | rock français de stade | d'après « Allumer le feu » (France) |
| 19 | Wind of Cycle | Nordstahl | DE | 4:00 | power ballade allemande | d'après « Wind of Change » (Allemagne) |
| 20 | De Materia ligera | Sodio Estéreo | ES | 3:40 | rock argentin 80s | d'après « De música ligera » (Argentine) |
| 21 | Coulée 900 | Les Hauts Fourneaux | instrumental | 1:50 | desert rock, riff d'ouverture | titre libre |
| 22 | Le Solo de Quarry Stone | Vince Tungsten | instrumental | 3:10 | desert rock, solo | titre libre |

#### Radio Velours (RadioVelours)

Fichier : `Documentation/Suno/10_RadioVelours.md` (68 Ko). Morceaux : 20 ; jingles : 4 ; interventions ou messages : 5 ; publicités ou annonces : 6.

**Identité.** Radio Velours est la station des soirées de la Capitale : jazz vocal et crooners, swing de grand orchestre, soul et Motown, bossa nova, lounge, city pop japonaise, crooners italiens, chanson jazz française, blues lent. Elle joue tard, elle joue bas, elle joue près du micro. C'est la radio qu'on entend dans les bars d'hôtel du Centre, dans les ascenseurs de la banque Cope Mutter, derrière les vitrines des parfums Komet et Luna, sur les terrasses de Staros quand les lampes du quartier s'allument une à une, et sur la plage de Costa Rive quand le Mur doré prend le clair de lune.

**Animatrice.** **Inès Legato**, voix feutrée, lente, qui laisse des silences. Sa phrase d'ouverture : « Il est tard sur la Capitale. » Elle parle français et anglais, glisse un mot de portugais ou d'italien quand un disque le demande, ne hausse jamais le ton. Son humour est discret et toujours tendre : il porte sur les corps neufs et les vieilles chansons (« un corps neuf, une vieille chanson »), sur les siècles de sommeil, sur la mémoire qui revient par le refrain avant de revenir par le nom. Deux noms alternatifs dans la nomenclature du monde, si Inès Legato ne convient pas : **Nora Vesper** (l'étoile du soir) ou **Léa Sidéral**.

**Langues.** Anglais dominant (8 morceaux), français (4), portugais du Brésil (2), italien (2), japonais (2), plus 2 instrumentaux. Les jingles, interventions et publicités sont en français, avec une phrase d'anglais quand Inès s'adresse à un auditeur d'ICLI Station.

**Registre.** Élégant, nocturne, jamais mélancolique au point de peser. Le contraste vient de là : des standards intemporels chantés par des gens qui ont dormi cinq siècles, qui se réveillent dans une salle de récupération et qui trouvent que, tout compte fait, la chanson tient mieux que le corps. Les Sanglines n'apparaissent qu'au loin, comme un bruit de fond derrière la baie vitrée. L'aérotram HLL817 passe entre deux cuivres. ICLI Station, en orbite, tient le rôle de la lune.

**Place dans le monde.** Station de la Capitale, sponsorisée avec discrétion par la banque Cope Mutter et les parfumeries Komet et Luna. Elle n'est pas corporate dans le ton (on n'y récite pas les consignes de sécurité), mais elle vit à l'intérieur du Mur et n'en sort que par la fenêtre. Programme type : ouverture de soirée, standards, bossa vers minuit, city pop et crooners italiens dans la nuit, blues lent avant le premier aérotram.

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Fly Me to the Station | Frankie Stellar & the Cope Mutter Orchestra | EN | 3:10 | jazz vocal crooner, swing | d'après « Fly Me to the Moon » (USA) |
| 02 | My Funny Sangline | Chet Bakelite | EN | 3:40 | ballade jazz, trompette, crooner | d'après « My Funny Valentine » (USA) |
| 03 | Les tours de mon coeur | Juliette Azimut | FR | 3:30 | jazz vocal français, valse tournante | d'après « Les moulins de mon coeur » (France) |
| 04 | Jardin IClabs | Henri Solaris | FR | 3:00 | ballade jazz douce, guitare nylon | d'après « Jardin d'hiver » (France) |
| 05 | Capitale, Capitale | Liza Minerai & the Staros Big Band | EN | 3:30 | swing big band, show tune | d'après « New York, New York » (USA) |
| 06 | Feeling New | Nina Simulacre | EN | 3:20 | big band soul jazz, cuivres | d'après « Feeling Good » (USA) |
| 07 | Smooth Operator (Iris Daxon) | Ombre | EN | 4:00 | sophisti-pop, quiet storm, sax | d'après « Smooth Operator » (Royaume-Uni) |
| 08 | Back to Body | Ivy Coldhouse | EN | 3:30 | soul rétro années 60, girl group | d'après « Back to Black » (Royaume-Uni) |
| 09 | Ain't No Sunshine Under the Wall | Bill Wattmore | EN | 3:00 | soul acoustique, cordes | d'après « Ain't No Sunshine » (USA) |
| 10 | Hit the Road, HLL817 | Ray Charger & the Relay-ettes | EN | 2:50 | rhythm and blues, call and response | d'après « Hit the Road Jack » (USA) |
| 11 | Garota de Costa Rive | Astra & João Gilbyte | PT-BR | 3:20 | bossa nova, guitare, voix douce | d'après « Garota de Ipanema » (Brésil) |
| 12 | Águas do Muro | Elis Rotina & Tom Jobinário | PT-BR | 3:30 | bossa nova en duo, liste chantée | d'après « Águas de Março » (Brésil) |
| 13 | Titanium Love | Mari Tachyon | JP | 4:00 | city pop années 80, funk, cuivres | d'après « Plastic Love » (Japon) |
| 14 | 上を向いて (ICLI) (Ue o muite (ICLI)) | Kyū Hoshimoto | JP | 3:10 | pop japonaise années 60, sifflement | d'après « 上を向いて歩こう / Sukiyaki » (Japon) |
| 15 | Volare (Orizaune) | Domenico Modulo | IT | 3:20 | crooner italien, swing, choeur | d'après « Nel blu dipinto di blu (Volare) » (Italie) |
| 16 | Vieni con me | Paolo Contatore | IT | 3:40 | jazz de piano-bar, voix rauque | d'après « Vieni via con me » (Italie) |
| 17 | La Cyborgaise | Serge Circuit | FR | 3:00 | chanson jazz française, valse lente | d'après « La Javanaise » (France) |
| 18 | Le poinçonneur de Tamil | Serge Circuit | FR | 3:10 | chanson jazz, contrebasse, scat | d'après « Le Poinçonneur des Lilas » (France) |
| 19 | Take Cycle Five | The Dave Bitrate Quartet | Instrumental | 5:20 | cool jazz en 5/4, sax alto | d'après « Take Five » (USA) |
| 20 | Wall Blues | Miles Décibel Sextet | Instrumental | 5:40 | blues modal lent, trompette sourdine | d'après « All Blues » (USA) |

#### HitWall (HitWall)

Fichier : `Documentation/Suno/11_HitWall.md` (78 Ko). Morceaux : 24 ; jingles : 6 ; interventions ou messages : 6 ; publicités ou annonces : 7.

> Station de radio du jeu QANGA. Contenu éditorial pour Suno : prompts de style et paroles complètes.
> Règles de fabrication : `Documentation/Suno/00_BRIEF_COMMUN.md`. Canon : `Documentation/QANGA_LORE_BIBLE.md`.
> Aucun asset ni code du projet n'est modifié par ce fichier.

**StationId** : `HitWall`, au catalogue depuis le 2026-09-14 (contrat : ne plus le renommer). Bloc d'antenne `WAV_Radio_HitWall_01` (1544.66 s, copie à -17 LUFS) dans `MS_QRadio_HitWall`, mêmes émetteurs qu'I-C News.

**Ce qu'est la station.** Le mur du son. Le Top 40 de la Capitale, celui qu'on entend dans les couloirs de Sboutique, dans la file du CineVortex, dans les manèges de SawgeniuS Park et dans tous les taxis Melrose. Pop 2755, dance-pop, K-pop, J-pop, eurodance, variété française, synthpop relue par la génération d'aujourd'hui, afrobeats. Sponsorisée par Sola, Sboutique et SawgeniuS Park.

**Le ton.** Hyper, sucré, jeune, des refrains énormes et une bonne humeur qui ne retombe jamais. Le lore passe en contrebande dans les tubes : une chanson d'amour parle d'un corps neuf sans le dire, une chanson de fête parle du Mur doré comme d'un décor de clip, et le morceau signature de la station réclame de la Matière sur un rythme de disco alors que la Matière n'est pas de l'argent et ne s'achète pas.

**Langues.** Anglais en dominante, plus le coréen, le japonais, l'italien, le français et l'allemand. Tout est chanté : c'est une station de tubes, il n'y a pas d'instrumental.

**Animateurs proposés** : le duo **Lou Pixel** (elle, rapide, rieuse, finit les phrases de l'autre) et **Max Tera** (lui, faux sérieux, jeux de mots lourds assumés). Alternatives dans la nomenclature du jeu, au choix de Benja : **Zia Flux** ou **Mia Photon** pour elle, **Théo Bit** ou **Sam Orbit** pour lui.

**Artiste pastiche déjà en jeu** : **Rame**, du nom des rames d'aérotram, auteur de « CryoBaby » et de « Long time », déjà présent dans le vivier musical du projet. Il signe ici deux nouveaux titres.

**Réception proposée** : couverture planétaire, comme les stations nationales (plein jusqu'à 9000 km, fondu jusqu'à 10000 km).

---

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Yellow Wall Style | PSI | KO | 3:40 | K-pop dance, cri de foule | d'après « Gangnam Style » (Corée du Sud) |
| 02 | Obsidienne | BT7 | KO et EN | 3:20 | K-pop disco-funk | d'après « Dynamite » (Corée du Sud) |
| 03 | PomPomCore | Kyary Pixel Pixel | JP | 3:10 | J-pop kawaii électro | d'après « PonPonPon » (Japon) |
| 04 | Polyrelay | Parfum Digital | JP | 4:10 | techno-pop japonaise | d'après « Polyrhythm » (Japon) |
| 05 | Matière Girl | Lady Gigaoctet | EN | 3:40 | synthpop années 80 | d'après « Material Girl » (États-Unis) |
| 06 | Cryo One More Time | Britney Spike | EN | 3:30 | teen pop années 90 | d'après « ...Baby One More Time » (États-Unis) |
| 07 | Cyborg Face | Lady Gigaoctet | EN | 3:50 | electropop de club | d'après « Poker Face » (États-Unis) |
| 08 | Reboot It | Michael Jason | EN | 4:00 | pop funk rock années 80 | d'après « Beat It » (États-Unis) |
| 09 | Blinding Lights (Relay Tower) | The Weeknight | EN | 3:20 | synthwave pop | d'après « Blinding Lights » (Canada) |
| 10 | Jetpacking | Dua Lipo | EN | 3:25 | nu-disco pop | d'après « Levitating » (Royaume-Uni) |
| 11 | Obsidiennes | Rihanna Nova | EN | 3:45 | pop ballade dance | d'après « Diamonds » (Barbade) |
| 12 | Hello (From the Recovery Room) | Adèle Orbit | EN | 4:15 | ballade pop au piano | d'après « Hello » (Royaume-Uni) |
| 13 | Sync Me Maybe | Rame | EN | 3:15 | dance-pop, cordes pincées | d'après « Call Me Maybe » (Canada) |
| 14 | Cyborg Girl | Aqualux | EN | 3:15 | eurodance bubblegum, duo | d'après « Barbie Girl » (Danemark) |
| 15 | Yellow (Da Ba Dee) | Eiffel 2755 | IT et EN | 3:30 | eurodance, voix filtrée | d'après « Blue (Da Ba Dee) » (Italie) |
| 16 | Il Cyborg | Toto Cotone | IT | 3:40 | variété italienne à refrain | d'après « L'italiano » (Italie) |
| 17 | Dancing Machine | ABBAtoir | EN | 3:50 | disco pop suédois, piano | d'après « Dancing Queen » (Suède) |
| 18 | Forever Cryo | Alphacode | DE et EN | 3:50 | synthpop années 80 | d'après « Forever Young » (Allemagne) |
| 19 | Tona où t'es | Stromatolite | FR | 3:40 | électro-variété, cuivres | d'après « Papaoutai » (Belgique) |
| 20 | Alors on recycle | Stromatolite | FR | 3:30 | électro-variété, boucle de synthé | d'après « Alors on danse » (Belgique) |
| 21 | Matja | Aya Nanomètre | FR | 2:50 | afropop urbaine | d'après « Djadja » (France) |
| 22 | Djibouti | Toto Relay | EN | 4:30 | soft rock années 80, percussions | d'après « Africa » (États-Unis) |
| 23 | Downroad Funk | Mark Relay et Bruno Matière | EN | 4:20 | funk pop à cuivres | d'après « Uptown Funk » (Royaume-Uni et États-Unis) |
| 24 | Last Cycle | Rame | EN | 3:20 | afrobeats | d'après « Last Last » (Nigeria) |

#### Panthéon (Pantheon)

Fichier : `Documentation/Suno/12_Pantheon.md` (53 Ko). Morceaux : 17 ; jingles : 4 ; interventions ou messages : 6 ; publicités ou annonces : 5.

Panthéon est la station culturelle officielle d'ICLabs : classique, opéra, choeur, musique sacrée, lied, mélodie, néo-classique et grandes pages orchestrales de film. Elle est sponsorisée par Cryoday (« A better future for your family ») et par IC Labs Industries, et elle le dit peu : le corporate est dans le choix des oeuvres, jamais dans les mots.
Sa ligne est grandiose et solennelle. La résurrection par les Tours de Relais y est chantée comme un mystère sacré ; la Matière comme une substance, presque une eucharistie ; le Mur de béton recouvert d'or (300 m de haut, 8 km de rayon, au milieu de l'océan) comme une cathédrale. Les trois cents Tours sont des sanctuaires, la salle de récupération est une chapelle, l'ancrage de la signature est un sacrement.
Sous cette surface, une inquiétude que personne ne nomme : la promesse de 2184 (« nous vous réveillerons quand tout sera terminé ») et l'archive de 2290, portant la mention « non communiqué », que la station lit pourtant à l'antenne avec la même solennité que le reste. Le contraste fait tout le travail. Abel ne commente jamais.
Animateur : **Abel Tempo**, voix grave, lente, diction parfaite, français. Il présente les oeuvres comme un conservateur présente une relique, lit les archives PostCom sur un lit de cordes discret, et rend hommage aux dormants de CRYO19 à chaque fin de cycle. Noms alternatifs dans la nomenclature du monde (prénom court + patronyme techno ou astronomique) : **Ivo Parsec** ou **Léo Solstice**.
Langues : italien (3 airs d'opéra), latin (3 pièces chorales), allemand (2 : un finale choral et un oratorio court), français (2 mélodies), anglais (2 choeurs ou thèmes de film chantés), russe (1 choeur), plus 4 pièces orchestrales instrumentales dont une longue de six minutes et demie. Les jingles sont en latin et en français.
Registre : celui du monde (bible §9 bis, registre C), mais ralenti et sacralisé. Pas de blague à l'antenne ; le décalage vient de ce qu'Abel dit avec gravité des choses terribles (« le corps est neuf ») comme s'il lisait un office. Les années n'apparaissent que dans les titres d'archives (2184, 2290), qui sont des références de document ; tout le reste se date en cycles.
Place dans le monde : Panthéon s'écoute dans le Hall du Complexe IClabs, dans les salles d'attente des Tours de Relais, dans les cabines d'aérotram entre deux sauts, et par ceux qui, la nuit, attendent leur tour dans une salle de récupération. Formations pastiches récurrentes : le Choeur des Trois Cents Tours (et sa section russe), l'Orchestre Philharmonique de la Capitale, l'Orchestre de Chambre ICLI, le ténor Aldo Pavone, la soprano Maria Callisto, la mezzo Solène Altaïr, la basse Otto Meridian.

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | O Materia | Choeur des Trois Cents Tours, Orchestre Philharmonique de la Capitale | LA | 3:10 | cantate chorale, ostinato, timbales | Orff, « O Fortuna » (Allemagne) |
| 02 | Nessuno dorme | Aldo Pavone, ténor | IT | 3:20 | air d'opéra, ténor, romantique tardif | Puccini, « Nessun dorma » (Italie) |
| 03 | Ancoratus | Choeur des Trois Cents Tours, Orchestre de Chambre ICLI | LA | 3:40 | choeur baroque, trompettes, timbales | Haendel, « Hallelujah » (Allemagne / Angleterre) |
| 04 | Ode an den Zyklus | Choeur des Trois Cents Tours, solistes, Orchestre Philharmonique de la Capitale | DE | 4:00 | finale choral symphonique | Beethoven, « Ode an die Freude » (Allemagne) |
| 05 | Habanera de la Sangline | Solène Altaïr, mezzo | FR | 3:30 | habanera d'opéra, mezzo, ironique | Bizet, « Habanera » de Carmen (France) |
| 06 | Va, coscienza | Choeur des Trois Cents Tours | IT | 3:50 | choeur d'opéra, unisson, nostalgique | Verdi, « Va, pensiero » (Italie) |
| 07 | Adagio pour un corps | Orchestre de Chambre ICLI | instrumental | 5:20 | adagio pour cordes | Barber, « Adagio for Strings » (États-Unis) |
| 08 | Lacrimosa pro dormientibus | Choeur des Trois Cents Tours, Orchestre Philharmonique de la Capitale | LA | 3:30 | requiem classique, choeur | Mozart, « Lacrimosa » (Autriche) |
| 09 | Clair de Luna | Maria Callisto, soprano, piano | FR | 3:30 | mélodie française, piano-voix | Debussy, « Clair de lune » (France) |
| 10 | Il corpo è mobile | Aldo Pavone, ténor | IT | 2:40 | canzone d'opéra, ténor, brillante | Verdi, « La donna è mobile » (Italie) |
| 11 | MarsOne | Orchestre Philharmonique de la Capitale | instrumental | 4:00 | poème symphonique martial en 5/4 | Holst, « Mars » (Angleterre) |
| 12 | Now We Are Synced | Maria Callisto, Choeur des Trois Cents Tours | EN | 4:00 | thème de film, voix éthérée, choeur | Lisa Gerrard, « Now We Are Free » (Australie) |
| 13 | Also sprach ICLabs | Otto Meridian, basse, Choeur des Trois Cents Tours, orgue | DE | 2:40 | oratorio court, orgue, cuivres | Strauss, « Also sprach Zarathustra » (Allemagne) |
| 14 | Полюшко-зона (Poliouchko-zona) | Choeur des Trois Cents Tours, section russe | RU | 3:20 | choeur d'hommes, marche lente | « Полюшко-поле » (Russie) |
| 15 | Chevauchée des Orizaunes | Orchestre Philharmonique de la Capitale | instrumental | 3:40 | chevauchée orchestrale, cuivres, galop | Wagner, « Chevauchée des Walkyries » (Allemagne) |
| 16 | L'or du Mur | Maria Callisto, choeur, Orchestre Philharmonique de la Capitale | EN + vocalise | 3:50 | western orchestral, soprano, choeur | Morricone, « L'Extase de l'or » (Italie) |
| 17 | Les Quatre Cycles | Orchestre de Chambre ICLI | instrumental | 6:30 | concerto baroque en quatre mouvements | Vivaldi, « Les Quatre Saisons » (Italie) |

#### Tamil Ondes (TamilOndes)

Fichier : `Documentation/Suno/13_TamilOndes.md` (33 Ko). Morceaux : 11 ; jingles : 2 ; interventions ou messages : 12 ; publicités ou annonces : 3.

Tamil Ondes est la radio de service du réseau de transport : elle joue dans les cabines du tram **Tamil Station** de la Capitale, dans les aérotrams qui sautent de Tour de Relais en Tour de Relais, et dans les trains du réseau relais. Personne ne l'a choisie ; elle est là quand les portes se ferment.

- **StationId** : `TamilOndes` (nouvelle station, à ajouter au catalogue `DA_QRadio_Stations`).
- **Animateur** : aucun. La seule voix est **la Voix du réseau** : féminine, synthétique, polie, légèrement inquiétante à force d'être calme. Registre A de la bible (le manuel corporate) : « Citoyen », « ICLabs vous assure que... », une promesse de service par phrase, jamais d'excuse, jamais d'émotion.
- **Langues** : musique instrumentale (11 morceaux) ; annonces en FR (4), EN (3), ES (3) et RU (2, en cyrillique) ; pubs en FR et EN.
- **Registre musical** : musique d'ascenseur, easy listening, exotica, bossa légère, chiptune doux, jazz de salon, swing de gare. Tempos de 72 à 160 bpm, aucune voix, aucune guitare saturée, aucune rupture : la cabine doit rester calme.
- **Ton** : la station qui annonce une attaque de Sangline sur le même carillon que l'arrêt Costa Rive. L'humour vient du contraste entre le calme de la Voix et ce qu'elle dit.
- **Place dans le monde** : émetteur planétaire (plan §4.1). C'est la station écrite pour être la station fixe des trains (`QRADIO_GUIDE.md` §9 et §11) et la station naturelle des aérostations. En véhicule, on tombe dessus en cherchant autre chose et on la garde par lassitude.
- **Vocabulaire du réseau** : correspondance obligatoire à la tour suivante ; pas de vol direct (il n'y en a jamais eu) ; rames codées `HLL817` ; arrêts du tram attestés dans le jeu : Center, East exit, IClabs Garden, Qamarac, Centinela, Costa Rive (chaînes `Tamil Station : ...` de la localisation).
- **Pubs** : trois marques du transport et du commerce (Melrose Taxi, Sboutique, Loanicle). La pub Tamil Station existe déjà (`Pub_Tamil`, 15.32 s) et n'est pas réécrite.
- **Ce qu'on ne dit jamais** : pas de fréquence chiffrée, pas de vol direct, pas de Matière comme monnaie, pas de mot qui fasse peur au Citoyen (la Voix ne connaît que des « incidents »).
- **Blague signature, version réseau** : « Votre signature est ancrée. Seule votre cargaison est en jeu. »

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Melrose Taxi | Herb Alpine and the Tamil Brass | Instrumental | 2:50 | easy listening brass, ameriachi | Herb Alpert and the Tijuana Brass, « Tijuana Taxi » (États-Unis) |
| 02 | Sangline Flea | Herb Alpine and the Tamil Brass | Instrumental | 2:40 | brass novelty, marimba | Herb Alpert and the Tijuana Brass, « Spanish Flea » (États-Unis) |
| 03 | Sanglines Keep Fallin' on My Roof | Burt Backtrack | Instrumental | 3:10 | sunshine pop, ukulélé, cordes | Burt Bacharach et Hal David, « Raindrops Keep Fallin' on My Head » (États-Unis) |
| 04 | Popcore | Hot Matter | Instrumental | 2:45 | synth-pop analogique 1972 | Gershon Kingsley, « Popcorn » (États-Unis, version Hot Butter) |
| 05 | Quiet Tower | Martin Dennay and his Exotica Console | Instrumental | 3:40 | exotica, vibraphone, cris d'oiseaux | Martin Denny, « Quiet Village » (États-Unis) |
| 06 | H.L.L. 817 | Jean-Jacques Périgée | Instrumental | 3:00 | space-age pop, basse Moog, breakbeat | Jean-Jacques Perrey, « E.V.A. » (France) |
| 07 | Rydeen (Tamil) | Yellow Wall Orchestra | Instrumental | 3:30 | techno-pop japonaise 1979, chiptune | Yellow Magic Orchestra, « Rydeen » (Japon) |
| 08 | Take the Aerotram | Duke Elevator and his Salon Orchestra | Instrumental | 3:20 | swing de salon, trompette bouchée | Duke Ellington et Billy Strayhorn, « Take the A Train » (États-Unis) |
| 09 | Dire Dawa Choo Choo | Glenn Milliamp and his Orchestra | Instrumental | 3:15 | big band swing, rythme de train | Glenn Miller, « Chattanooga Choo Choo » (États-Unis) |
| 10 | Costa Rive | Henri Manciné and the Costa Strings | Instrumental | 3:50 | valse orchestrale, harmonica | Henry Mancini, « Moon River » (États-Unis) |
| 11 | Last Rame to Djibouti | The Monorails | Instrumental | 2:55 | jangle pop 1966, riff de guitare | The Monkees, « Last Train to Clarksville » (États-Unis) |

#### Radio Sable (RadioSable)

Fichier : `Documentation/Suno/14_RadioSable.md` (76 Ko). Morceaux : 18 ; jingles : 4 ; interventions ou messages : 6 ; publicités ou annonces : 5.

**Identité.** Radio Sable est la voix des communautés de la Surface autour de Djibouti et Dire Dawa : des gens qui ne se sont jamais endormis dans la glace, qui n'ont jamais pris de corps de rechange, et qui refusent le contrôle d'ICLabs sans le combattre. Ils se nomment « ceux de la Surface », « les gens du sable », « la Surface ». Ils vivent avec le sable, l'eau rare, les Sanglines sous les dunes, le troc et la mémoire des noms. La station émet faiblement, hors du réseau IC, depuis un émetteur bricolé quelque part entre le Grand Pont, la dépression du Danakil et la montagne du relais 77 (qui, lui, reste muet). On la capte dans le désert de la région de départ, et elle se perd dès qu'on s'en éloigne.

**Animatrice.** **Awa Zenith** : voix chaude, posée, rieuse, qui tutoie tout le monde et glisse des mots d'arabe et de somali (salam, shukran, yalla, walaal, biyo, nabad). Elle ne crie jamais, elle ne vend rien. Noms alternatifs dans la nomenclature du monde si Awa Zenith ne convient pas : **Hodan Vega** ou **Samra Antarès**.

**Langues.** Français dominant (7 morceaux), arabe (5, en alphabet arabe, dialecte maghrébin pour le raï et le chaâbi, levantin et égyptien pour les deux chansons de tarab), anglais (4, dont un duo EN/FR), plus 2 instrumentaux. Les interventions sont en français semé d'arabe et de somali, les jingles en français et en arabe.

**Registre.** Dignité, chaleur, refus sans haine. Ici on ne dit pas « cycle » mais « jour », on ne compte pas en Crédits mais en gourdes d'eau et en pièces de moteur, on ne se réveille qu'une fois. La Tour de Relais de Djibouti est à côté, gratuite, et personne n'y va : « un corps, une vie, un nom ». On regarde la Capitale et son Mur doré comme une chose lointaine et un peu triste. Les convoyeurs d'ICLabs passent au Grand Pont, on leur fait signe, on ne monte pas. Pas de pillards, pas de Voss : ceux du sable s'en méfient autant que d'ICLabs, et n'en parlent qu'à demi-mot.

**Place dans le monde.** Nouvelle station, hors réseau IC. Émetteur `Local` faible centré sur la zone de Dire Dawa (plan section 4.1 : `FalloffStartKm` 20, `RadiusKm` 60) ; au catalogue depuis le 2026-09-14 (`RadioSable`, bloc `WAV_Radio_RADIOSABLE_01` dans `MS_QRadio_RadioSable`) en réception planétaire provisoire, l'émetteur local reste à valider sur place. Pas de publicité : des annonces de troc à la place, plus un message poli aux convoyeurs. Aucune piste existante du projet n'est montée sur cette station.

| N | Titre | Artiste | Langue | Durée cible | Style | Clin d'oeil |
|---|---|---|---|---|---|---|
| 01 | Sable et Sangline | Ténéré Watt | FR | 3:50 | desert blues, guitare hypnotique, tindé | Tinariwen « Sastanàqqàm » (Mali / Algérie) |
| 02 | L'eau qui se souvient | Ali Kafra Sirius | FR | 3:40 | desert blues acoustique, calebasse, njarka | Ali Farka Touré « Savane » (Mali) |
| 03 | Talking Starkitown | Ali Kafra Sirius et Roy Slide | EN | 3:30 | desert blues, slide guitar, talking blues | Ali Farka Touré et Ry Cooder « Talking Timbuktu » (Mali / USA) |
| 04 | نسم علينا الرمل (Nassam Alayna ar-Raml) | Fayrouna | AR | 3:30 | desert blues levantin, oud, voix féminine | Fairuz « Nassam Alayna El Hawa » (Liban) |
| 05 | Au coeur du Danakil | Ali Kafra Sirius et Toma Kora | Instrumental | 4:00 | desert blues, guitare et kora, sans voix | Ali Farka Touré et Toumani Diabaté « In the Heart of the Moon » (Mali) |
| 06 | Yekermo Cyborg | Mulatu Astral | Instrumental | 4:10 | éthio-jazz, vibraphone, cuivres, orgue | Mulatu Astatke « Yekermo Sew » (Éthiopie) |
| 07 | ألف دورة ودورة (Alf Dawra wa Dawra) | Oum Qamar et l'Orchestre du Relais 77 | AR | 5:50 | éthio-jazz et tarab, long, cuivres, qanun | Oum Kalthoum « Alf Leila wa Leila » (Égypte) |
| 08 | Tezeta de l'Ancien Monde | Mulatu Astral et Hodan Lumière | FR | 3:45 | éthio-jazz chanté, saxophone, nostalgie | Mulatu Astatke « Tezeta » (Éthiopie) |
| 09 | Water Get Enemy | Fela Kilowatt et l'Afrika 77 | EN | 5:40 | afrobeat 1970s, cuivres, long groove | Fela Kuti « Water No Get Enemy » (Nigeria) |
| 10 | Yeh (The Sand Does Not Lie) | Burna Sol | EN | 3:20 | afro-fusion moderne, mid-tempo, chaleureux | Burna Boy « Ye » (Nigeria) |
| 11 | 7 Cycles | Youssou N'Dune et Nene Sirocco | EN/FR | 3:55 | afro-pop ballade, duo, percussions sabar | Youssou N'Dour et Neneh Cherry « 7 Seconds » (Sénégal / Suède) |
| 12 | عائشة (على حافة الجسر الكبير) / Aïcha (au bord du Grand Pont) | Cheb Khalil Nadir | AR/FR | 4:00 | raï pop 1990s, guitare, derbouka, synthé | Khaled « Aïcha » (Algérie / France) |
| 13 | ديدي (ديري داوا) / Didi (Dire Dawa) | Cheb Khalil Nadir | AR | 3:35 | raï dansant, accordéon, derbouka, choeurs | Khaled « Didi » (Algérie) |
| 14 | يا رايح (نحو العاصمة) / Ya Rayah (vers la Capitale) | Rachid Tesla | AR | 3:50 | chaâbi rock, mandole, darbouka, guitare | Rachid Taha « Ya Rayah » (Algérie / France) |
| 15 | Sablone | Omara Bombardo | FR | 3:30 | rock touareg rapide, riffs, claps | Bombino « Iyat Ninhay (Jaguar) » (Niger) |
| 16 | Soubour (Patience du sable) | Songhaï Circuit | FR | 3:25 | rock touareg et songhaï, énergique | Songhoy Blues « Soubour » (Mali) |
| 17 | Dimanche à Dire Dawa | Amadi et Miriam Solaire | FR | 3:30 | chanson afro-blues, duo, joyeuse | Amadou et Mariam « Dimanche à Bamako » (Mali) |
| 18 | Madan (Le nom qu'on garde) | Salif Kepler et Fayrouna | FR/AR | 4:05 | chanson mandingue, kora, balafon, duo | Salif Keita « Madan » (Mali) |


---

## 5. Répartition des 22 marques (publicités)

Toutes les marques du monde (bible, section 8) ont au moins un spot, écrit dans la langue et le ton de la station. Les 5 spots de `QRADIO_ICNEWS_ANTENNE.md` (Melrose, Iron Dee, IClabs, Titanium ICLI / ICLI Station, SawgeniuS Park) et les deux enregistrements existants (`Pub_LifeLoop`, `Pub_Tamil`) restent valables ; le panel ne les duplique pas.

| Marque | Stations qui la diffusent |
|---|---|
| IC Labs Industries | I-C News, Panthéon (et contre-pub AMBRE) |
| I-C News Radio (autopromo) | I-C News (et contre-pub AMBRE) |
| ICLIspace (Velkara) | Orbita |
| Titanium ICLI | I-C News, Orbita, Forge FM (et contre-pub AMBRE) |
| ICLI Station (les trois comptoirs) | I-C News, Orbita |
| Iron Dee | I-C News, Forge FM, Vieux Monde |
| Melrose | I-C News (Orizaune), Yellow Roots (Sablone), Centinela (Taxi), Forge FM (PickUp armé), Tamil Ondes (Taxi) (et contre-pub AMBRE) |
| Cryoday | I-C News, Panthéon (et contre-pub AMBRE) |
| Cope Mutter | I-C News, Centinela, Orbita, Velours, Panthéon (et contre-pub AMBRE) |
| Sola | Yellow Roots, Centinela, HitWall |
| McFrailenergy | I-C News, Traverse, Forge FM, HitWall |
| Komet | Traverse, Orbita, Velours |
| Luna | Velours, HitWall, Panthéon |
| Starkito | Yellow Roots, Centinela |
| Chez Fred | Yellow Roots, Vieux Monde, Centinela, Velours |
| Sboutique | I-C News, Traverse, Velours, HitWall, Tamil Ondes |
| Hardware Store | I-C News, Traverse, Vieux Monde, Orbita, Forge FM |
| A.D Store | Forge FM |
| CineVortex | Traverse, Velours, HitWall |
| SawgeniuS Park | Yellow Roots, HitWall |
| Costa Riv | Yellow Roots, Centinela, HitWall |
| LoopLife | Chill FM, Panthéon |
| Vieego | I-C News, Chill FM |
| Loanicle (entité canon sans affiche) | Traverse, Tamil Ondes |
| Comptoir d'échange (service des Tours) | Vieux Monde |

Radio Sable ne diffuse pas de publicité (annonces de troc à la place) ; AMBRE ne diffuse que des contre-publicités.

---

## 6. Fréquences d'affichage (proposition, pour un futur tuner)

Le champ `Frequency` existe et n'est lu nulle part ; ces valeurs ne servent qu'à l'affichage et ne sont jamais dites à l'antenne.

| Station | Fréquence proposée |
|---|---|
| I-C News Radio | 87.5 |
| Chill FM | 88.8 |
| Yellow Roots | 90.9 |
| Traverse | 92.2 |
| Vieux Monde | 94.4 |
| Radio Centinela | 97.7 |
| Радио Орбита | 99.9 |
| Forge FM | 101.1 |
| Radio Velours | 103.3 |
| HitWall | 104.4 |
| Panthéon | 105.5 |
| Tamil Ondes | 106.6 |
| AMBRE | 107.7 (au bout de la bande, brouillée) |
| Radio Sable | 108.0 |

---

## 7. Ce que le panel respecte (et ce qu'il ne fait pas)

- **Lore** : dates (2180, 2184, 2228, 2290, 2350, présent 2755), lieux réels (quartiers de la Capitale, Djibouti / Dire Dawa, ICLI Station orbitale, Tour de Relais 77, Starkitown), vocabulaire (cycle, Matière contre Crédits, ancrage, salle de récupération, aérotram par sauts, warp), factions (le Voss et rien d'autre, les communautés de la Surface sans le mot « survivants »), les 22 marques et leurs slogans peints, la nomenclature des noms.
- **Interdits d'antenne** : pas de « survivants », « apocalypse », « zombies », « la Terre est morte » ; pas de dôme ni d'amarrage à YellowWall ; Crédits et Matière distincts ; pas de vol aérotram direct ; jamais « Échelon Zéro » ; aucune fréquence prononcée ; le Voile et le Flux jamais expliqués ; le secret de TONA jamais révélé.
- **Clins d'oeil au vrai monde** (demande de Benja du 2026-09-12) : chaque morceau vise une oeuvre ou un artiste réel, connu mondialement ou dans son pays, par le titre détourné, le nom d'artiste pastiche (dans la lignée de Bob Marcly, Damia Marcly, Big Roh, Rame), le genre et le thème. **Jamais une parole réelle recopiée** (droits), **jamais un nom d'artiste réel dans un prompt Suno** (Suno les refuse). La ligne `Clin d'oeil` de chaque morceau dit à Benja ce qui est visé.
- **Style** : aucun tiret cadratin ni demi-cadratin dans les livrables (vérifié par grep sur chaque fichier, résultats en section 9).
- **Ce que le panel ne fait pas** : aucune modification d'asset, de catalogue, de MetaSound, de `.ini` ni de code ; aucune fréquence fixée ; aucune décision de nom prise à la place de Benja.

---

## 8. Pistes existantes réutilisables (durées mesurées)

Durées lues dans le bloc de tags des `.uasset` (`Duration`, en secondes). C'est la valeur à mettre dans `TrackMeta.Duration` si la piste est montée telle quelle.

**Vivier radio, `Content/Sounds/QangaMusic/MusicRadio/` (27 pistes)**

| Piste | Durée (s) | Affectation proposée |
|---|---|---|
| BobMarcly_-_Moi | 236.64 | Yellow Roots |
| DamiaMarcly-_LuiAussi | 219.84 | Yellow Roots |
| BigRoh_-_Jgriffe | 208.00 | Yellow Roots |
| Folk_-_MaleInYellowall | 229.80 | Vieux Monde |
| ORGHOUSE | 402.60 | Traverse |
| ChakaPower | 183.04 | Traverse |
| SummerHit_2790 | 169.00 | HitWall (titre à arbitrer : 2790 > 2755) |
| Rame-CryoBaby | 168.80 | HitWall |
| Rame-CryoBaby-RadioEdition | 134.52 | HitWall |
| Rame-CryoBaby-SunEdition | 144.04 | HitWall |
| Rame-Long_time | 156.72 | HitWall |
| RiseRise | 188.44 | HitWall |
| LE_REVEIL_DES_SANGLINES | 110.09 | AMBRE |
| 2180Echoes | 209.84 | AMBRE |
| BZHLINE_2_COLOSSE | 237.88 | Forge FM (genre à vérifier à l'écoute) |
| FredericCllbt_-_Controle | 179.92 | Forge FM (genre à vérifier à l'écoute) |
| ABCD | 219.80 | à écouter et affecter |
| CristalSaid | 148.12 | à écouter et affecter |
| CroyEchoes | 185.28 | à écouter et affecter |
| Dorotax | 229.96 | à écouter et affecter |
| DreamWater | 163.01 | à écouter et affecter |
| Fodel | 161.72 | à écouter et affecter |
| Porokelatos_no | 194.72 | à écouter et affecter |
| Porokelatos_yes | 196.16 | à écouter et affecter |
| Rul_-_YellowGarden | 155.48 | à écouter et affecter |
| ShakataOne | 429.20 | à écouter et affecter |
| StandAlone | 170.08 | à écouter et affecter |

Ces 27 pistes sont déjà montées dans `MS_MusicLib` (25) et `MS_BarMusic` (13) ; les monter aussi en radio ne retire rien à ces MetaSounds.

**Voix, pubs et lits, `Plugins/Qasset/Content/Audio/WAV/Radio/`**

| Piste | Durée (s) | Usage |
|---|---|---|
| ICNEWSRADIO_01 | 1475.37 | ancienne émission d'I-C News, **retirée du projet le 2026-09-14** (copie dans `F:\QANGA_Backups\qradio_placeholders_2026-09-14`) |
| WAV_Radio_ICNEWSRADIO_01 | 1502.47 | I-C News, émission définitive montée par Benja, importée depuis une copie à -17 LUFS (2026-09-14) |
| I-C_News_YellowWall | 259.68 | placeholder I-C News, **retiré du projet le 2026-09-14** (aucune référence, copie sauvegardée) |
| WAV_I-C_News_Radio_Var01 | 259.68 | placeholder I-C News, **retiré du projet le 2026-09-14** (aucune référence, copie sauvegardée) |
| WAV_I-C_News_Radio_Var02 | 227.24 | placeholder I-C News, **retiré du projet le 2026-09-14** (aucune référence, copie sauvegardée) |
| Music_chill | 374.44 | ancienne piste de Chill FM, remplacée le 2026-09-14 (asset conservé) |
| WAV_Radio_CHILLFM_01 | 1964.71 | Chill FM, bloc d'antenne monté par Benja, importé depuis une copie à -17 LUFS (2026-09-14) |
| WAV_Radio_HitWall_01 | 1544.66 | HitWall, bloc d'antenne monté par Benja, importé depuis une copie à -17 LUFS (2026-09-14) |
| WAV_Radio_RADIOSABLE_01 | 1229.09 | Radio Sable, bloc d'antenne monté par Benja, importé depuis une copie à -17 LUFS (2026-09-14) |
| WAV_Radio_YellowRoot_01 | 2037.66 | Yellow Roots, bloc d'antenne monté par Benja, importé depuis une copie à -17.2 LUFS (2026-09-14) |
| Pub_LifeLoop | 24.43 | pub LoopLife (toutes stations) |
| Pub_Tamil | 15.32 | pub Tamil Station (Tamil Ondes) |
| Pub_Tamil_Replique | 3.16 | réplique Tamil |
| MainRadio | 640.26 | ancien montage radio, à réécouter |
| SON_Radio_Garage | 997.45 | ambiance garage, à réécouter |
| Wave_Solo_Guitar | 65.44 | lit pour Vieux Monde |
| radio_tv_electronic_static_hum_loop_01 à 03 | 5.84 / 7.48 / 6.42 | friture (déjà dans `RadioEffect` de `MS_QRadioStation`) |

**Ambiance Exploration, `Content/Sounds/QangaMusic/SoundTracks/2026/` (40 pistes, déjà utilisées par QMusicDirector)** : `Dissidence` 479.36 s, `Sanglotown` 104.56 s (candidates AMBRE), `Glitze` 158.88 s (candidate Traverse). Double usage à arbitrer (section 2, point 5). `MusicForMission/` (14 pistes, 45.9 à 484.9 s) n'est branché nulle part et pourrait servir de lits instrumentaux à I-C News ou Panthéon (`Circa` 230.54 s, `Equinox` 153.21 s, `Illuminate` 222.24 s, `Recall` 226.98 s, `Vigilant_Final` 299.47 s, `Wayfarer_Final` 307.17 s, `ICLI` 171.12 s, `Limbo` 156.24 s, `Dizzy` 158.75 s, `Rust__Bonus_Track_` 300.01 s, `Dusk__Bonus_Track_` 484.86 s, `Winner_Final` 45.89 s).

**Sources hors éditeur** : `G:\QangaSync\Sons\RadioMusics\` (2180Echoes et V2, CristalSaid, CryoEchoes, DreamWater, les 4 Lovidry, les 3 QANGA - One, Rame-BabyWomen, les Rame-CryoBaby, 7 WAV_AtmosMusicSound, 7 Untitled) et `G:\QangaSync\Sons\Musiques\` (Altai, Ascendo, Banjo Fun, Chill Beat, Lo Fi Acoustic Chill, Myriad, Outland, Shaman, Sheikh, Bushido, salasa...). Plusieurs de ces sources ne sont pas importées : à écouter pour compléter les playlists sans crédit Suno.

---

## 9. Vérifications faites

| Fichier | Morceaux | Jingles | Interventions | Pubs ou annonces | Blocs de paroles | Tirets interdits | Taille |
|---|---|---|---|---|---|---|---|
| `01_ICNewsRadio.md` | 12 | 8 | 6 | 12 | 12 | 0 | 54 Ko |
| `02_ChillFM.md` | 18 | 4 | 2 | 2 | 18 | 0 | 41 Ko |
| `03_YellowRoots.md` | 22 | 5 | 6 | 6 | 22 | 0 | 76 Ko |
| `04_Traverse.md` | 22 | 5 | 5 | 6 | 22 | 0 | 61 Ko |
| `05_VieuxMonde.md` | 20 | 4 | 6 | 4 | 20 | 0 | 63 Ko |
| `06_VossAmbre.md` | 20 | 4 | 6 | 6 | 20 | 0 | 67 Ko |
| `07_RadioCentinela.md` | 22 | 5 | 6 | 6 | 22 | 0 | 73 Ko |
| `08_RadioOrbita.md` | 20 | 5 | 6 | 6 | 20 | 0 | 90 Ko |
| `09_ForgeFM.md` | 22 | 5 | 6 | 6 | 22 | 0 | 62 Ko |
| `10_RadioVelours.md` | 20 | 4 | 5 | 6 | 20 | 0 | 68 Ko |
| `11_HitWall.md` | 24 | 6 | 6 | 7 | 24 | 0 | 78 Ko |
| `12_Pantheon.md` | 17 | 4 | 6 | 5 | 17 | 0 | 53 Ko |
| `13_TamilOndes.md` | 11 | 2 | 12 | 3 | 11 | 0 | 33 Ko |
| `14_RadioSable.md` | 18 | 4 | 6 | 5 | 18 | 0 | 76 Ko |

**Total : 268 morceaux, 65 jingles, 84 interventions ou messages, 80 publicités ou annonces.** Le compte de tirets interdits est celui de `grep` sur les caractères U+2013 et U+2014 ; il doit être 0 partout (le plan lui-même est vérifié de la même façon).

---

## 10. Ordre de production suggéré

1. Voie de lecture continue : tranchée par Benja le 2026-09-14, voie (B), un bloc d'antenne par station (section 1). Chill FM est la première station montée ainsi.
2. Compléter les deux stations existantes (I-C News : jingles, pubs, lits ; Chill FM : 18 pistes) : elles sont déjà au catalogue, zéro ligne à ajouter.
3. Yellow Roots, Traverse, Vieux Monde : les trois stations de la maquette intranet, avec leurs pistes existantes déjà montées.
4. HitWall et Forge FM : les deux plus grosses playlists, celles qui donnent le sentiment « GTA ».
5. Radio Centinela et Радио Орбита : les deux langues demandées.
6. Radio Velours, Panthéon, Tamil Ondes.
7. AMBRE et Radio Sable, qui demandent un placement d'émetteurs local et, pour AMBRE, un arbitrage sur le module.

---

*Plan des stations QRadio. Créé le 12 septembre 2026. À mettre à jour dès qu'une station entre au catalogue.*
