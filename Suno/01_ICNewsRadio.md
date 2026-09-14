# I-C News Radio (ICNewsRadio)

I-C News Radio est la station officielle d'IC Labs Industries : la voix civile et corporate de la Capitale, celle qui donne le trafic aérotram, les cours des minerais et le bulletin de zone avec le sourire d'un présentateur qui n'est jamais sorti de derrière le Mur. Le monde mord, la station ne le sait pas, ou fait comme si. C'est le registre C de la bible du lore : tout le sel vient du décalage.

- **StationId** : `ICNewsRadio` (existe déjà dans le catalogue `DA_QRadio_Stations` ; c'est la station de démarrage, `DefaultStationId=ICNewsRadio` dans `Config/DefaultGame.ini`).
- **Animateur** : **Théo Cadence**, la matinale. Enjoué, chaleureux, un peu trop optimiste : il traite un réveil en salle de récupération comme un retard d'aérotram. Noms alternatifs dans la nomenclature du monde (prénom court + patronyme techno ou astronomique), pour arbitrage de Benja : **Sam Vector** ou **Milo Sirius**.
- **Seconde voix** : **Ava Signal**, bulletins de zone et trafic. Sèche, factuelle, elle lit un nid de Sanglines comme un relevé de compteur. Noms alternatifs : **Léa Radian** ou **Mia Azimut**.
- **Langue** : français, avec les slogans peints des marques en anglais, tels quels. Aucune fréquence chiffrée n'est annoncée.
- **Registre musical** : lits instrumentaux space synth-rock, cyberpunk enjoué et rétro-futuriste (les anciennes pistes `ICNEWSRADIO_01`, `I-C_News_YellowWall` et les deux `WAV_I-C_News_Radio_Var` étaient dans cette veine ; retirées du projet le 2026-09-14 au profit de l'émission définitive `WAV_Radio_ICNEWSRADIO_01`), plus six chansons « maison » en français : de la pop corporate optimiste au second degré, l'optimisme institutionnel qui ne voit pas les Sanglines.
- **Rubriques** : ouverture et fermeture de cycle, bulletin de zone, logistique et trafic aérotram (par sauts, de tour en tour), cours des minerais (Fer, Obsidienne, Silicium, Cuivre, Aluminium), publicité. La blague signature reste celle de la bible d'antenne : la mort n'est pas grave, seule la cargaison l'est.
- **Place dans le monde** : émetteur spatial à l'origine de l'univers (plein jusqu'à 9000 km, fondu jusqu'à 10000 km d'après `QRADIO_GUIDE.md`) : on la capte à la Capitale, dans la zone de test, sur le Grand Pont de Dire Dawa et jusqu'à la file d'amarrage d'ICLI Station.
- **Slogans peints** : « Your ultimate playlist, all day, every day » et « Your sound, our passion ». Ils ferment l'autopromo et les identifiants chantés.
- **Ce que ce fichier ajoute** à la bible d'antenne (`QRADIO_ICNEWS_ANTENNE.md`, qui a déjà trois scripts, quatre identifiants et cinq spots) : 12 morceaux, 8 jingles nouveaux, 6 interventions nouvelles, 12 spots nouveaux. Rien ici ne reprend un texte déjà écrit là-bas.

## Sommaire

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

## Morceaux

### 01. Tout va très bien, Cyborg

| Champ | Valeur |
|---|---|
| Artiste | Théo Cadence et ses Convoyeurs |
| Langue | FR |
| Durée cible | 3:10 |
| Style | swing music-hall, cuivres, standard téléphonique |
| Clin d'oeil | d'après « Tout va très bien, Madame la Marquise » (France) |
| Ancrage lore | Un Cyborg appelle sa Tour de Relais depuis la route pour prendre des nouvelles de son convoi ; le standard lui annonce, toujours souriant, la Sablone au fond d'un nid, le contrat perdu, et enfin qu'il appelle depuis la salle de récupération. |

**Prompt Suno (champ Style of Music)**
```text
1930s French music-hall swing, jazz orchestra, 150 bpm, brass section, clarinet, upright bass, brushed drums, playful call-and-response between a cheerful male crooner and a syrupy corporate switchboard choir, sung in French, comic escalating verses, vintage radio warmth with a faint modern synth pad, big final chorus, clean ending
```

**Paroles (champ Lyrics)**
```text
[Intro]
(sonnerie, un standard qui décroche, cuivres qui s'échauffent)
Ici la Tour de Relais, bonjour Cyborg, ne quittez pas

[Verse 1]
Allô, allô, la Tour ? Je rentre de zone sauvage
J'appelle pour mon convoi, dites-moi qu'il est bien sage
Cyborg, votre convoi est arrivé au Grand Pont
Un peu mordu, un peu brûlé, mais dans le bon rayon
Trois caisses sur cinq sont restées dans le sable
Rien de grave, rien de grave, la prime reste stable

[Chorus]
Tout va très bien, Cyborg, tout est sous contrôle
Le Mur est doré, l'aérotram est à l'heure
Tout va très bien, Cyborg, gardez votre rôle
Une signature ancrée, c'est déjà le bonheur

[Verse 2]
Allô, allô, la Tour ? Et ma Sablone chérie ?
Cyborg, votre moto brille encore, au fond d'un nid
Les Sanglines l'ont trouvée quand le métal fumait
Vu de la route, c'était presque joli, en fait
Le garage vous propose une remise sur le Riper
Et la logistique confirme : votre contrat, on le perd

[Chorus]
Tout va très bien, Cyborg, tout est sous contrôle
Le Mur est doré, l'aérotram est à l'heure
Tout va très bien, Cyborg, gardez votre rôle
Une signature ancrée, c'est déjà le bonheur

[Bridge]
(le standard baisse la voix, presque tendre)
Il reste un petit détail, Cyborg, une formalité
Vous nous appelez d'où, exactement ? Ah, de la salle de récupération ?
Corps neuf, bien reçu. Alors tout va très bien : la cargaison, on ne l'a plus.

[Chorus]
Tout va très bien, Cyborg, tout est sous contrôle
Le Mur est doré, l'aérotram est à l'heure
Tout va très bien, Cyborg, reprenez votre rôle
Le corps est neuf, la cargaison ? C'est pour tout à l'heure

[Outro]
(cuivres, le standard raccroche, la sonnerie repart)
Ici la Tour de Relais, bonjour Cyborg, ne quittez pas
```

### 02. T'inquiète, t'es ancré

| Champ | Valeur |
|---|---|
| Artiste | Les Ancrés |
| Langue | FR |
| Durée cible | 3:00 |
| Style | a cappella, percussions vocales, sifflet |
| Clin d'oeil | d'après « Don't Worry, Be Happy » (États-Unis) |
| Ancrage lore | La promesse d'ancrage des Tours de Relais chantée comme une berceuse : la Matière basse, la Sablone dans le sable, le contrat qui prend l'eau, et une note de service d'IC Labs Industries pour rappeler que la cargaison n'est pas couverte. |

**Prompt Suno (champ Style of Music)**
```text
A cappella pop, late 1980s feel, 88 bpm, layered male voices only, vocal percussion and mouth drums, finger snaps, whistled melody hook, relaxed reggae-tinged swing, warm baritone lead, playful and reassuring, sung in French, short spoken corporate interlude, no instruments except voices and snaps
```

**Paroles (champ Lyrics)**
```text
[Intro]
(percussions vocales, claquements de doigts, un sifflement léger)
Ou-ou-ou, hm-hm-hm, t'inquiète

[Verse 1]
Voilà une histoire que tout le monde ici connaît
Une Sangline t'a pris la jambe, et le reste juste après
Tu ouvres les yeux dans une salle, sous une lumière bleue
Le corps est neuf, la tête aussi, enfin presque, un peu

[Chorus]
T'inquiète, t'es ancré
Ta signature est déposée
Le noyau bat, le Mur est doré
T'inquiète, t'es ancré

[Verse 2]
Ta Matière est basse et ton jetpack tousse dans le vent
Ta Sablone dort dans le sable, ton contrat prend l'eau lentement
Le centre logistique t'écrit, avec un sourire poli
Rien n'est perdu, sauf la cargaison, et ça, mon frère, c'est la vie

[Chorus]
T'inquiète, t'es ancré
Ta signature est déposée
Le noyau bat, le Mur est doré
T'inquiète, t'es ancré

[Bridge]
(sifflé, puis le choeur)
Y a trois cents Tours, trois cents lits, trois cents matins
Y a plus de fin, y a juste un peu de chemin
Alors chante avec nous, même la bouche pleine de sable
T'es ancré, Cyborg, et c'est ça qui est formidable

[Spoken]
(voix de standard, chaleureuse, sur les percussions vocales)
Rappel de la maison : l'ancrage de signature est un service IC Labs Industries. La cargaison n'est pas couverte. Bonne journée, Cyborg.

[Chorus]
T'inquiète, t'es ancré
Ta signature est déposée
Le noyau bat, le Mur est doré
T'inquiète, t'es ancré

[Outro]
T'inquiète... t'es ancré... ou-ou-ou
```

### 03. Marcher sous le Mur

| Champ | Valeur |
|---|---|
| Artiste | Costa Rive Club |
| Langue | FR |
| Durée cible | 3:20 |
| Style | pop-rock années 80, cuivres, tambourin |
| Clin d'oeil | d'après « Walking on Sunshine » (Royaume-Uni) |
| Ancrage lore | Le matin d'une convoyeuse qui vient de livrer trois caisses de Cuivre au centre logistique et marche sur Costa Rive, sous les trois cents mètres d'or du Mur qui retient l'océan, entre la Porte Est et la Porte Sud. |

**Prompt Suno (champ Style of Music)**
```text
1980s upbeat pop-rock, 145 bpm, bright brass section, driving drums, tambourine, handclaps, jangly electric guitar, sunny and euphoric, female lead vocal, powerful and joyful, gang backing vocals, sung in French, big shout-along chorus, horn riff hook, glossy retro production
```

**Paroles (champ Lyrics)**
```text
[Intro]
Oh oh oh, oh oh oh oh

[Verse 1]
J'ai rendu ma cargaison au centre avant le lever
Trois caisses de Cuivre, un contrat signé, et la prime est tombée
Je sors par la Porte Sud, la lumière monte sur le béton
Trois cents mètres d'or par-dessus ma tête, et l'océan qui tient bon

[Pre-Chorus]
Le sable de Costa Rive est chaud sous mes pieds tout neufs
Le noyau bat, la Matière est pleine, et j'ai le temps de faire un voeu

[Chorus]
Je marche sous le Mur, et ça brille, ça brille
Je marche sous le Mur, et rien ne me mord aujourd'hui
Je marche sous le Mur, la Capitale s'éveille
Tout est doré, oui, tout est doré, quand je marche sous le Mur

[Verse 2]
La Traverse sature, l'aérotram saute de tour en tour
Les cours du Fer sont stables, la radio dit qu'on vit les plus beaux jours
Dehors, derrière la porte, la zone de test attend son heure
Mais ce matin, le Mur me couvre, et c'est ça, c'est ça, mon bonheur

[Chorus]
Je marche sous le Mur, et ça brille, ça brille
Je marche sous le Mur, et rien ne me mord aujourd'hui
Je marche sous le Mur, la Capitale s'éveille
Tout est doré, oui, tout est doré, quand je marche sous le Mur

[Bridge]
(cuivres, tambourin, les choeurs montent)
Et si demain je repars, et si la nuit me prend
Une Tour me rendra un corps, un matin, en souriant
Alors je danse tant qu'il fait jour, de la Porte Est à la Porte Sud
La mer est haute, le Mur la tient, et sous le sable, rien ne bouge, j'en suis sûre

[Chorus]
Je marche sous le Mur, et ça brille, ça brille
Je marche sous le Mur, et rien ne me mord aujourd'hui
Je marche sous le Mur, la Capitale s'éveille
Tout est doré, oui, tout est doré, quand je marche sous le Mur

[Outro]
(cuivres qui s'éloignent, claquements de mains)
Sous le Mur, sous le Mur, oh oh oh, sous le Mur
```

### 04. Monsieur Mur Jaune

| Champ | Valeur |
|---|---|
| Artiste | Orchestre Lumière du Complexe |
| Langue | FR |
| Durée cible | 4:20 |
| Style | pop orchestrale, cordes, vocodeur |
| Clin d'oeil | d'après « Mr. Blue Sky » (Royaume-Uni) |
| Ancrage lore | Après trois cycles de pluie sur la Capitale, la lumière revient sur le Mur de béton recouvert d'or, et toute la ville (les convoyeurs, les Autonomes bridés, un Cyborg qui sort de la salle de récupération) remercie le Mur comme on remercie un patron. |

**Prompt Suno (champ Style of Music)**
```text
1970s orchestral pop rock, 120 bpm, lush string section, cellos, grand piano, layered vocal harmonies, vocoder interjections, bright major key, euphoric and cinematic, male lead vocal, high clean tenor, sung in French, multi-section song with an orchestral coda and a robotic vocoder outro, glossy analog production
```

**Paroles (champ Lyrics)**
```text
[Intro]
(voix vocodée, grésillement radio) Ici le réseau... la lumière remonte sur la Capitale... bonjour Cyborg

[Verse 1]
Le ciel était gris depuis trois cycles, la pluie tombait sur les Tours
Les rames restaient à quai, Costa Rive avait fermé ses détours
Et puis ce matin, sans prévenir, quelque chose a brillé
Trois cents mètres d'or debout dans la mer, et le gris s'est effacé

[Pre-Chorus]
Tout le monde sort, tout le monde regarde en l'air
Même les Autonomes bridés lèvent leur tête de fer

[Chorus]
Monsieur Mur Jaune, dis-nous comment
Tu retiens l'océan en souriant
Monsieur Mur Jaune, reste encore un instant
Tu nous rends le ciel, tu nous rends le temps

[Verse 2]
Les cours du Cuivre sont montés, la Traverse chante à l'unisson
Un convoi rentre du Grand Pont, la moitié des caisses, mais bon
Un Cyborg sort de la salle de récupération, il s'étire, il rit
Il a tout perdu hier soir, mais le Mur est là, alors ça va, dit-il

[Chorus]
Monsieur Mur Jaune, dis-nous comment
Tu retiens l'océan en souriant
Monsieur Mur Jaune, reste encore un instant
Tu nous rends le ciel, tu nous rends le temps

[Bridge]
(cordes, piano, la voix vocodée revient)
Et quand la nuit reviendra, avec ses bruits sous le sable
On fermera la Porte Sud, on montera la radio, c'est le programme
Monsieur Mur Jaune ne dort jamais, il n'a pas de noyau à recharger
Il tient la mer, il tient la ville, il tient toutes les signatures ancrées

[Chorus]
Monsieur Mur Jaune, dis-nous comment
Tu retiens l'océan en souriant
Monsieur Mur Jaune, reste encore un instant
Tu nous rends le ciel, tu nous rends le temps

[Outro]
(coda orchestrale, choeur vocodé, cordes qui montent) Monsieur Mur Jaune... Monsieur Mur Jaune...
Ici le réseau, la lumière tient, restez à l'écoute
```

### 05. Voilà le cycle

| Champ | Valeur |
|---|---|
| Artiste | Les Scarabées du Relais |
| Langue | FR |
| Durée cible | 3:10 |
| Style | folk-pop acoustique, Moog, claquements |
| Clin d'oeil | d'après « Here Comes the Sun » (Royaume-Uni) |
| Ancrage lore | Le lever d'un cycle vu par un Cyborg qui a dormi des siècles dans la glace de CRYO19 : les Sanglines rentrent au nid avec la lueur, le Fer tient, la Tour garde un corps de rechange, et le froid s'en va. |

**Prompt Suno (champ Style of Music)**
```text
Late 1960s acoustic folk-pop, 128 bpm, bright fingerpicked acoustic guitar, early Moog synth lines, handclaps, gentle drums, sunny and tender, male lead vocal, soft tenor, warm stacked harmonies, sung in French, hopeful morning song with a clapped bridge and a gentle fade
```

**Paroles (champ Lyrics)**
```text
[Intro]
(guitare acoustique, un Moog qui monte doucement)
Doo doo doo doo, voilà le cycle

[Verse 1]
Cyborg, tu as dormi dans la glace, des siècles sans un rêve
Cyborg, tu t'es réveillé dans un corps qui ne prend pas de trêve
Et le froid, tout ce froid, je crois qu'il s'en va

[Chorus]
Voilà le cycle, doo doo doo doo
Voilà le cycle, le Mur prend sa couleur
Ça ira, Cyborg

[Verse 2]
Cyborg, les Sanglines sont rentrées au nid quand la lueur est venue
Cyborg, le sable est chaud et le silence a repris la rue
Et la nuit, toute la nuit, je crois qu'elle s'en va

[Chorus]
Voilà le cycle, doo doo doo doo
Voilà le cycle, le Mur prend sa couleur
Ça ira, Cyborg

[Bridge]
(claquements de mains, le Moog en avant)
Cycle, cycle, cycle, cycle, encore un cycle
Ancre ta signature, recharge ton noyau, et repars sous le ciel
Cycle, cycle, cycle, cycle, encore un cycle
La Matière est pleine, la Sablone démarre, et tout redevient possible

[Verse 3]
Cyborg, la radio te dit que les cours du Fer tiennent bon
Cyborg, la Tour te dit qu'un corps t'attend si tu tombes trop long
Et le doute, tout ce doute, je crois qu'il s'en va

[Chorus]
Voilà le cycle, doo doo doo doo
Voilà le cycle, le Mur prend sa couleur
Ça ira, Cyborg

[Outro]
(guitare seule, le Moog s'éteint)
Voilà le cycle... c'est bon, Cyborg, c'est bon
```

### 06. Le bon côté du réseau

| Champ | Valeur |
|---|---|
| Artiste | La Chorale du Centre Logistique |
| Langue | FR |
| Durée cible | 3:20 |
| Style | music-hall sifflé, ukulélé, choeur |
| Clin d'oeil | d'après « Always Look on the Bright Side of Life » (Royaume-Uni) |
| Ancrage lore | La chorale du centre logistique console les convoyeurs : Sablone en morceaux dans une faille, Silicium mangé par une Sangline, file d'amarrage saturée à ICLI Station, Rogue sur la route du Centre de Purification ; le bon côté, c'est que les trois cents Tours rendent un corps, et que la mort est devenue un formulaire. |

**Prompt Suno (champ Style of Music)**
```text
Cheeky music-hall singalong, 1970s comedy tune feel, 112 bpm, ukulele, whistled melody hook, brass band, marching snare, jaunty and cheerful, male lead vocal, chirpy, full choir on the chorus, sung in French, whistled instrumental refrain, big singalong finale, vintage film soundtrack warmth
```

**Paroles (champ Lyrics)**
```text
[Intro]
(sifflement, ukulélé, un choeur qui s'échauffe)

[Verse 1]
Quand ta Sablone est en morceaux au fond d'une faille
Quand ta cargaison de Silicium nourrit une Sangline qui bâille
Quand le centre logistique te dit que ton contrat est mort
Ne fais pas cette tête, Cyborg, souris un peu plus fort

[Chorus]
Regarde le bon côté du réseau
Trois cents Tours et un corps tout beau
Le noyau repart, le compteur repart aussi
Regarde le bon côté du réseau, et siffle avec nous, allez, merci

[Verse 2]
Quand la file d'amarrage tourne au ralenti, là-haut, à ICLI Station
Quand le warp te refuse parce que t'es encore dans l'atmosphère, quelle déception
Quand un Rogue te regarde de travers sur la route du Centre de Purification
Chante avec nous, Cyborg, c'est la meilleure des protections

[Chorus]
Regarde le bon côté du réseau
Trois cents Tours et un corps tout beau
Le noyau repart, le compteur repart aussi
Regarde le bon côté du réseau, et siffle avec nous, allez, merci

[Bridge]
(le choeur seul, très corporate) Parce que la mort, ici, c'est un formulaire
Une signature ancrée, une lumière bleue, un lit propre, et l'affaire est claire
La cargaison ? Ah, ça, c'est une autre histoire
Mais tu as un corps neuf, et un corps neuf, ça vaut bien un au revoir

[Verse 3]
Quand tu te réveilles avec le goût du sable et l'odeur du métal chaud
Quand la radio te chante que tout va bien, juste un peu trop tôt
Rappelle-toi qui tient les Tours, qui tient les lits, qui tient le réseau
Et dis merci avec le sourire, c'est écrit sur le contrat, tout en haut

[Chorus]
Regarde le bon côté du réseau
Trois cents Tours et un corps tout beau
Le noyau repart, le compteur repart aussi
Regarde le bon côté du réseau, et siffle avec nous, allez, merci

[Outro]
(sifflement seul, qui s'éloigne) Le bon côté... du réseau...
```

### 07. Lever de cycle

| Champ | Valeur |
|---|---|
| Artiste | Habillage I-C News |
| Langue | Instrumental |
| Durée cible | 2:40 |
| Style | space synth-rock, guitare, arpèges |
| Clin d'oeil | aucun titre précis : habillage d'antenne original, esprit synthwave des génériques d'information |
| Ancrage lore | Le lit de la matinale de Théo Cadence : la lumière qui remonte le long du Mur, les rames qui repartent de tour en tour, un cycle qui commence comme si personne n'avait été mordu pendant la nuit. |

**Prompt Suno (champ Style of Music)**
```text
Instrumental, no vocals. Space synth-rock radio bed, 128 bpm, driving electric guitar riff, fast arpeggiated cyberpunk synths, punchy drums, wide analog pads, energetic and optimistic, clean retro-futuristic production, steady loopable groove suitable for talk-over, short radio sting intro, clean ending
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Synth arpeggio intro, radio sting]
[Guitar riff builds with full drums]
[Wide pad breakdown, then full groove to a clean ending]
```

### 08. Cours du cycle

| Champ | Valeur |
|---|---|
| Artiste | Habillage I-C News |
| Langue | Instrumental |
| Durée cible | 2:30 |
| Style | cyberpunk enjoué, basse punchy |
| Clin d'oeil | aucun titre précis : habillage d'antenne original |
| Ancrage lore | Le lit de la rubrique logistique et trafic : contrats du centre logistique, retard d'un convoi au Grand Pont, rame HLL817 à l'heure, Traverse saturée ; ça avance, ça bouge, ça ne s'arrête jamais. |

**Prompt Suno (champ Style of Music)**
```text
Instrumental, no vocals. Upbeat cyberpunk radio bed, 132 bpm, punchy slap bass synth, tight electronic drums, bright staccato synth stabs, retro-futuristic lead line, energetic and busy, glossy production, steady groove suitable for talk-over, short intro sting, clean ending
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Bass and drums intro, radio sting]
[Staccato synth stabs, lead melody]
[Short breakdown, groove returns, clean ending]
```

### 09. Ce qui sera sera

| Champ | Valeur |
|---|---|
| Artiste | Habillage I-C News |
| Langue | Instrumental |
| Durée cible | 2:50 |
| Style | valse rétro-futuriste, lead synthé |
| Clin d'oeil | d'après « Que sera, sera » (États-Unis) |
| Ancrage lore | Le lit des cours des minerais : une valse à trois temps pour le Fer, l'Obsidienne, le Silicium, le Cuivre et l'Aluminium, parce que le marché fera ce qu'il voudra et que la raffinerie Iron Dee encaisse dans tous les cas. |

**Prompt Suno (champ Style of Music)**
```text
Instrumental, no vocals. Retro-futuristic synth waltz in 3/4, 96 bpm, lilting analog synth lead melody, soft electronic drums with a brushed feel, warm bass synth, glockenspiel-like bell accents, dreamy and cheerful, 1950s lounge waltz feel reimagined with 1980s synthesizers, suitable for talk-over, gentle intro, clean ending
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Soft bell and pad intro, waltz begins]
[Synth lead melody, brushed drums]
[Brief key change, final waltz chorus, clean ending]
```

### 10. Bulletin de zone

| Champ | Valeur |
|---|---|
| Artiste | Habillage I-C News |
| Langue | Instrumental |
| Durée cible | 2:20 |
| Style | synth-rock industriel, percussions |
| Clin d'oeil | aucun titre précis : habillage d'antenne original |
| Ancrage lore | Le lit des bulletins d'Ava Signal : nids signalés, Sand Diggers sous la route de Starkitown, Flyers au-dessus du Grand Pont ; tendu, mécanique, mais toujours propre, parce que la station ne panique jamais. |

**Prompt Suno (champ Style of Music)**
```text
Instrumental, no vocals. Tense industrial synth-rock radio bed, 124 bpm, heavy palm-muted electric guitar, metallic industrial percussion, aggressive synth lead, dark pulsing bass, urgent but controlled, cinematic newsroom energy, steady groove suitable for talk-over, alarm-like intro sting, clean ending
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Alarm-like synth sting, industrial percussion intro]
[Heavy guitar riff, pulsing bass]
[Tense breakdown with metallic hits, riff returns, clean ending]
```

### 11. File d'amarrage

| Champ | Valeur |
|---|---|
| Artiste | Habillage I-C News |
| Langue | Instrumental |
| Durée cible | 3:00 |
| Style | synthé cinématique, nappes, lent |
| Clin d'oeil | aucun titre précis : habillage d'antenne original |
| Ancrage lore | Le lit de la fermeture de cycle : ICLI Station en orbite, la file d'amarrage qui tourne au ralenti, le warp interdit en approche, la Capitale qui s'éteint en dessous ; large, lent, patient. |

**Prompt Suno (champ Style of Music)**
```text
Instrumental, no vocals. Slow cinematic space synth bed, 84 bpm, wide analog synth pads, distant clean electric guitar with long delay, slow steady drums, deep sub bass, gentle arpeggios, patient and majestic, orbital and nocturnal mood, suitable for talk-over, long pad intro, slow fade ending
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Long pad intro, distant guitar]
[Slow beat enters, arpeggios]
[Wide climax, then slow fade ending]
```

### 12. La vie en jaune

| Champ | Valeur |
|---|---|
| Artiste | Habillage I-C News |
| Langue | Instrumental |
| Durée cible | 2:50 |
| Style | rétro-futuriste lent, accordéon synthé |
| Clin d'oeil | d'après « La Vie en rose » (France) |
| Ancrage lore | Le lit de nuit de la station : une chanson d'amour de l'Ancien Monde sans les mots, jouée à l'accordéon de synthèse pour la Capitale qui dort sous son Mur doré ; la vie est jaune ici, et la radio trouve ça très beau. |

**Prompt Suno (champ Style of Music)**
```text
Instrumental, no vocals. Slow retro-futuristic ballad bed, 72 bpm, synthesized accordion lead, muted trumpet-like synth counter melody, warm analog pads, soft brushed electronic drums, upright-style bass synth, vintage Parisian chanson feel reimagined with 1980s synthesizers, tender and nostalgic, suitable for talk-over, gentle intro, soft ending
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Soft pad and accordion intro]
[Main melody on synth accordion, muted trumpet answer]
[Warm final chorus, soft ending]
```

## Jingles et identifiants de station

Huit identifiants nouveaux, distincts des quatre de la bible d'antenne (« On parle, vous conduisez », « Depuis la Capitale, jusqu'à l'orbite », « Le monde s'est arrêté. Pas nous. », « Si vous nous entendez, c'est que vous êtes encore là »), qui restent valables. Voix : Théo (homme, enjoué) sur les identifiants de jour, Ava (femme, sèche) sur la nuit et l'alerte. Les chantés passent par un choeur et un vocodeur, à la manière des habillages d'antenne.

### J01. Identifiant chanté : « Your sound, our passion »
Durée cible : 0:08. Langue : FR (chute EN, slogan tel quel).
**Prompt Suno** :
```text
Radio station ident, 8 seconds, bright space synth-rock sting, layered choir singing the station name, vocoder harmony, punchy drum hit, energetic and glossy, then a warm male announcer voice in French, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed: bright synth-rock sting]
[Sung jingle: layered choir with vocoder]
I-C News Ra-di-o !
[Spoken: warm male voice, French]
Your sound, our passion.
```

### J02. Identifiant parlé : « Corps neuf, même station »
Durée cible : 0:06. Langue : FR.
**Prompt Suno** :
```text
Radio station ident, 6 seconds, short cyberpunk synth stab with a rising arpeggio, dynamic male radio announcer, French, close mic, confident and cheerful
```
**Texte** :
```text
[Music bed: short synth stab, rising arpeggio]
[Spoken: dynamic male voice, French]
I-C News Radio. Corps neuf, même station.
[Sting ends]
```

### J03. Retour de pub : « Ça, c'était nos partenaires »
Durée cible : 0:08. Langue : FR.
**Prompt Suno** :
```text
Radio station ident, 8 seconds, upbeat cyberpunk drum fill and synth riser into a bright chord, dynamic male radio announcer, French, warm and playful
```
**Texte** :
```text
[Music bed: drum fill, synth riser, bright chord]
[Spoken: dynamic male voice, French]
Ça, c'était nos partenaires. Ça, c'est nous. I-C News Radio, on reprend.
[Bed continues under the next segment]
```

### J04. Top horaire et nouveau cycle : « Le compteur repart »
Durée cible : 0:12. Langue : FR.
**Prompt Suno** :
```text
Radio top-of-the-hour ident, 12 seconds, three short synchronisation beeps, then a majestic space synth-rock chord swell with drums, layered choir on the station name, dynamic male announcer in French, bright and official
```
**Texte** :
```text
[Three short sync beeps]
[Music bed: chord swell, drums]
[Spoken: dynamic male voice, French]
Top de cycle sur le réseau. Le compteur repart, et vous aussi.
[Sung jingle: layered choir]
I-C News Ra-di-o !
[Spoken: male voice, French]
Nouveau cycle.
```

### J05. Nuit : « On baisse la voix, pas la garde »
Durée cible : 0:10. Langue : FR.
**Prompt Suno** :
```text
Radio station ident, 10 seconds, slow warm analog synth pad with a soft pulse, no drums, calm female radio voice in French, close mic, low and steady, ends on a short vocoder station name
```
**Texte** :
```text
[Music bed: slow warm pad, soft pulse]
[Spoken: calm female voice, French, close mic]
I-C News Radio, service de nuit. On baisse la voix, pas la garde.
[Vocoder: I-C News Radio]
```

### J06. Urgence zone : « Alerte zone »
Durée cible : 0:07. Langue : FR.
**Prompt Suno** :
```text
Radio alert stinger, 7 seconds, short two-tone synth siren, industrial percussion hit, tense low drone, urgent female radio voice in French, close mic, factual and sharp
```
**Texte** :
```text
[Short two-tone synth siren, percussion hit]
[Spoken: urgent female voice, French]
Alerte zone. I-C News Radio. Restez à l'écoute, restez à couvert.
[Low drone fades]
```

### J07. Identifiant chanté : « All day, every day »
Durée cible : 0:10. Langue : EN (slogan tel quel) puis FR.
**Prompt Suno** :
```text
Radio station ident, 10 seconds, retro-futuristic synth-pop bed, vocoder lead singing an English tagline, bright arpeggios, snappy drums, glossy and cheerful, closing spoken French station name by a male announcer
```
**Texte** :
```text
[Music bed: retro-futuristic synth-pop, arpeggios]
[Sung jingle: vocoder lead]
Your ultimate playlist, all day, every day
[Spoken: dynamic male voice, French]
I-C News Radio. Toujours là.
```

### J08. Identifiant matinale : « Théo au micro, Ava sur le trafic »
Durée cible : 0:09. Langue : FR.
**Prompt Suno** :
```text
Radio show ident, 9 seconds, upbeat space synth-rock riff with bright brass-like synth stabs, dynamic male announcer in French, then a short dry female voice line, ends on a drum hit
```
**Texte** :
```text
[Music bed: synth-rock riff, brass-like stabs]
[Spoken: dynamic male voice, French]
Le Mur est doré, les rames sont à quai, Théo Cadence est au micro.
[Spoken: dry female voice, French]
Et Ava Signal sur le trafic.
[Spoken: male voice, French]
I-C News Radio, la matinale.
[Drum hit]
```

## Interventions d'animateur

Six interventions nouvelles, dans la voix de Théo Cadence (matinale) et d'Ava Signal (zone et trafic), plus une rediffusion au format canon « ICLabs Broadcast ». Elles complètent les trois scripts de la bible d'antenne sans les reprendre. Format : `[Spoken Intro: Dynamic and natural French Radio Host]` + `(Bruitage ...)`, encadré de balises de lit musical pour que Suno tienne la voix parlée.

### A01. Ouverture de cycle : Théo sur Dire Dawa
Durée cible : 1:00. Langue : FR.
**Prompt Suno** :
```text
Radio talk segment, 60 seconds, natural French radio host, male, dynamic, warm, cheerful morning energy, close mic, light space synth-rock music bed under the voice, short jingle sting at the start, bed rises at the end
```
**Texte** :
```text
[Music bed fades in: space synth-rock, bright]
[Spoken Intro: Dynamic and natural French Radio Host]
(Jingle dynamique I-C News Radio)
Bonjour Cyborg, ici Théo Cadence, et c'est un nouveau cycle sur I-C News Radio ! La lumière arrive sur Dire Dawa, le Grand Pont sort de la brume, et la Tour de Relais vient de rallumer ses plaques bleu nuit. Tout le monde est là ? Presque. Une pensée pour les trois convoyeurs qui se sont réveillés cette nuit en salle de récupération : le corps est neuf, les amis, on ne va pas pleurer. Enfin, pas pour le corps.
(Bruitage de communication radio)
Au programme : les cours des minerais, avec un Cuivre qui fait des siennes ; le bulletin de zone d'Ava Signal, qui a des choses à vous dire sur la route de Starkitown ; et une rame HLL817 annoncée à l'heure, qui, je vous le dis tout de suite, ne va pas plus loin que la tour suivante. Comme d'habitude. C'est ça, la fiabilité.
Mettez le contact, gardez la Matière au-dessus du rouge, et on y va.
[Music bed rises: instrumental drop, space synth-rock]
```

### A02. Point cours des minerais
Durée cible : 0:55. Langue : FR.
**Prompt Suno** :
```text
Radio talk segment, 55 seconds, natural French radio host, male, dynamic, playful, close mic, retro-futuristic synth waltz music bed under the voice, radio communication sound effect, bed fades at the end
```
**Texte** :
```text
[Music bed fades in: retro-futuristic synth waltz]
[Spoken Intro: Dynamic and natural French Radio Host]
(Bruitage de communication radio)
Les cours du cycle, en direct du guichet d'Iron Dee, et croyez-moi, il y a du mouvement.
Le Fer : stable, comme un Mur. On ne le remercie jamais assez.
L'Obsidienne : en hausse. Les laboratoires en redemandent, ne me demandez pas pour quoi faire, je ne demande pas non plus.
Le Silicium : plat. Complètement plat. Le Silicium, c'est le cycle où il ne se passe rien, et franchement, ça fait du bien.
Le Cuivre : en baisse de trois points, parce qu'un convoi entier a livré d'un coup au sud du Grand Pont. Bravo à eux. Merci pour les autres.
L'Aluminium : en hausse, léger, comme lui.
(Bruitage de communication radio)
Traduction Théo Cadence : vendez l'Obsidienne maintenant, gardez le Cuivre trois cycles, et le Silicium, laissez-le dans la soute, il ne dérange personne. Les cours sont donnés à titre indicatif ; les Sanglines, elles, ne le sont pas.
[Music bed fades out]
```

### A03. Bulletin de zone : Ava Signal
Durée cible : 1:00. Langue : FR.
**Prompt Suno** :
```text
Radio news bulletin, 60 seconds, natural French radio host, female, dry, factual, calm, close mic, tense industrial synth-rock music bed low under the voice, alert stinger at the start, bed fades at the end
```
**Texte** :
```text
[Alert stinger: short two-tone synth siren]
[Music bed fades in: industrial synth-rock, low]
[Spoken Intro: Dynamic and natural French Radio Host, female, dry and factual]
(Bruitage de communication radio)
Ava Signal, bulletin de zone. Trois points.
Un. Route de Starkitown : des Sand Diggers sous la piste, entre le troisième et le cinquième kilomètre après la Tour. Elles sortent au passage des véhicules. Rappel : la carapace est pare-balles, tirer dessus depuis la cabine ne sert à rien, contournez par la crête ouest.
Deux. Failles à l'est de Dire Dawa : des Climbers sur les parois, en nombre. Les équipes de biomasse qui descendent avec des cordes descendront aussi avec des Climbers. À vous de voir.
Trois. Grand Pont : des Flyers au-dessus du tablier au lever du cycle, en groupe. Les convois sont passés, mais pas tous entiers.
(Bruitage de communication radio)
Enfin, pour ceux qui ont croisé du matériel à interface ambre sur la piste sud : la position officielle d'IC Labs Industries n'a pas changé, matériel non homologué, vous êtes prévenus. Bulletin terminé. À vous, Théo.
[Music bed fades out]
```

### A04. Appel d'auditeur : Ilan, salle de récupération de Dire Dawa
Durée cible : 1:20. Langue : FR.
**Prompt Suno** :
```text
Radio call-in segment, 80 seconds, two French voices: a dynamic warm male radio host close mic, and a male caller through a thin telephone filter, shaky and tearful, light space synth-rock music bed low under the voices, phone line sound effects, bed rises at the end
```
**Texte** :
```text
[Music bed fades in: space synth-rock, low]
[Spoken Intro: Dynamic and natural French Radio Host]
(Bruitage de ligne téléphonique qui décroche)
On a un auditeur en ligne. Ilan, bonjour Cyborg, vous nous appelez d'où ?
[Spoken: male caller, telephone filter, shaky voice]
(Bruitage de ligne, souffle)
De... de la salle de récupération de Dire Dawa. Je viens de me réveiller. Théo, j'avais quatre caisses d'Obsidienne, ma Sablone, un contrat pour le centre logistique... Une Sand Digger est sortie du sable juste devant la roue. J'ai rien vu.
[Spoken: radio host, warm]
Ilan, respirez. Le corps, il est comment ?
[Spoken: caller]
Il est... il est neuf. Il est très bien.
[Spoken: radio host]
Voilà. Ça, c'est réglé. Et la cargaison ?
[Spoken: caller]
(sanglot) Elle est là-bas. Avec la Sablone. Avec la Sand Digger.
[Spoken: radio host]
Alors elle a un gardien, c'est déjà ça. Ilan, écoutez-moi : le corps est neuf, la cargaison, elle, ne revient pas toute seule. Mais elle n'est pas partie non plus. Elle vous attend. Prenez une arme au comptoir, un Riper au garage, et allez la chercher au lever du cycle, jamais la nuit.
(Bruitage de communication radio)
Et pour la Sablone, le garage Melrose de Dire Dawa vous fera un prix. Je le dis parce que c'est vrai, pas parce qu'ils nous paient. Enfin, si, un peu. Courage, Ilan. On vous passe un morceau, il est pour vous.
[Music bed rises: instrumental drop]
```

### A05. ICLabs Broadcast : Ayla Romane et le Dr Emil Karrow, variantes de Sanglines
Durée cible : 1:30. Langue : FR.
**Prompt Suno** :
```text
Radio interview segment, 90 seconds, two French voices: a composed female journalist, clear and professional, and an older male scientist, calm, precise, slightly weary, corporate broadcast intro sting, soft neutral synth pad music bed very low under the voices, then a dynamic male radio host returns cheerfully at the end
```
**Texte** :
```text
[Corporate broadcast sting]
[Spoken Intro: Dynamic and natural French Radio Host]
I-C News Radio rediffuse ICLabs Broadcast. Au micro, Ayla Romane.
[Music bed: soft neutral pad, very low]
[Spoken: female journalist, composed]
(Bruitage de studio, léger)
ICLabs Broadcast, variantes confirmées. Docteur Emil Karrow, biologiste à la Research Division d'ICLabs, vous êtes avec nous. Trois variantes de Sanglines sont maintenant documentées. Commençons par la Climber.
[Spoken: male scientist, calm, precise]
La Climber grimpe. Parois, failles, structures métalliques. Elle attend en hauteur et descend sur ce qui passe. Sa vitesse est celle de toutes les Sanglines : supérieure à la nôtre.
[Spoken: female journalist]
La Sand Digger.
[Spoken: male scientist]
Elle s'enfouit. Elle perçoit les vibrations du sol et sort au contact. Sa carapace arrête les projectiles standard. On ne la combat pas de face : on l'évite, ou on la prend par-dessous, ce qui n'est pas un conseil.
[Spoken: female journalist]
Et la Flyer.
[Spoken: male scientist]
La Flyer se déplace en groupe, au-dessus des axes. Elle s'intéresse au métal encore chaud : un moteur, un canon, une cabine au soleil. Coupez le moteur, laissez refroidir, et elle passe.
[Spoken: female journalist]
Docteur, ces trois variantes sont-elles la même espèce ?
[Spoken: male scientist]
Les Sanglines ne sont pas une entité unique. C'est un phénomène évolutif dynamique, potentiellement sans limite.
[Spoken: female journalist]
D'autres variantes vont donc apparaître ?
[Spoken: male scientist]
Ce n'est pas une possibilité. C'est une certitude.
[Spoken: female journalist]
Docteur Emil Karrow, merci. ICLabs Broadcast, Ayla Romane.
[Corporate broadcast sting]
[Spoken: Dynamic and natural French Radio Host, cheerful]
Merci Ayla ! Et sur cette excellente nouvelle, tout de suite, les cours de l'Aluminium !
[Music bed rises: upbeat cyberpunk]
```

### A06. Fermeture de cycle : file d'amarrage à ICLI Station (Théo et Ava)
Durée cible : 1:10. Langue : FR.
**Prompt Suno** :
```text
Radio closing segment, 70 seconds, two French voices: a warm dynamic male radio host close mic, and a dry factual female traffic voice, slow cinematic space synth music bed under the voices, radio communication sound effects, bed rises at the end
```
**Texte** :
```text
[Music bed fades in: slow cinematic space synth]
[Spoken Intro: Dynamic and natural French Radio Host]
(Bruitage de communication radio)
Fin de cycle sur I-C News Radio. La Capitale s'éteint quartier par quartier, Costa Rive a rangé ses parasols, et là-haut, ICLI Station fait ce qu'elle fait de mieux : attendre. Ava, la file ?
[Spoken: dry female voice, factual]
File d'amarrage ICLI Station : quarante-deux vaisseaux en attente, temps estimé deux heures. Un Orizaune Melrose s'est mis en travers du couloir d'approche, le contrôle le dégage. Rappel du contrôle : pas de warp en approche, pas de warp dans l'atmosphère, et pas de warp du tout si vous touchez quelque chose. Pour ceux qui sont déjà au sol : dernière rame vers Dire Dawa au prochain quart, saut par saut, comme toujours. Terminé.
[Spoken: radio host, warm]
Merci Ava. Quarante-deux vaisseaux, deux heures, un Orizaune en travers : voilà, c'est ça, une soirée réussie en orbite.
(Bruitage de communication radio)
Pour ceux qui coupent le moteur ce soir dans une zone sauvage : ancrez votre signature avant de fermer les yeux, pas après. Et si vous vous réveillez quand même dans une salle de récupération, on sera là, avec les cours, le trafic, et le sourire.
C'était Théo Cadence sur I-C News Radio. Keep the Flux.
[Music bed rises: instrumental, slow fade]
```

## Publicités

Douze spots nouveaux, quinze à trente secondes, une marque par spot, le slogan peint en chute (tel quel, en anglais). Ils ne reprennent pas les cinq spots de la bible d'antenne (Melrose Sablone, Iron Dee « nous, on fond », IClabs « trois cents Tours », Titanium ICLI / ICLI Station « montez, achetez, redescendez vivant », SawgeniuS Park). Voix : Théo sur la plupart, une voix corporate sur IC Labs Industries, Ava sur Titanium ICLI, et deux voix féminines de pub sur Cryoday et Sboutique.

### P01. IC Labs Industries : « Quelqu'un a construit un Mur »
Durée cible : 0:25. Langue : FR (chute EN, slogan tel quel).
**Prompt Suno** :
```text
Radio commercial, 25 seconds, warm corporate male voice in French, close mic, majestic slow synth pad bed with a soft choir swell, distant ocean waves, confident and reassuring, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: majestic synth pad, distant waves]
[Spoken Ad: warm corporate voice, French]
(Bruitage de vagues, très loin)
Trois cents mètres de béton recouverts d'or. Huit kilomètres de rayon. Et l'océan, dehors, qui attend son tour depuis toujours.
Vous dormez tranquille parce que quelqu'un a construit un Mur. Et parce que quelqu'un l'entretient.
IC Labs Industries. Yellow Wall: the safety first.
[Music bed fades out]
```

### P02. Cryoday : « La date, on s'en occupe »
Durée cible : 0:25. Langue : FR (chute EN, slogans tels quels).
**Prompt Suno** :
```text
Radio commercial, 25 seconds, soft cheerful female voice in French, close mic, bright retro-futuristic synth-pop bed with a gentle music box, a soft freezer hum, sweet and slightly unsettling, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: bright synth-pop, music box, soft freezer hum]
[Spoken Ad: soft cheerful female voice, French]
(Bruitage de givre, un souffle froid)
Votre famille mérite de se réveiller dans un monde meilleur. La question, c'est quand.
Chez Cryoday, la date, on s'en occupe. Vous, vous fermez les yeux. Nous, on surveille le thermostat, aussi longtemps qu'il faudra.
Cryoday. A better future for your family. Futur is cryo.
[Music bed fades out]
```

### P03. Cope Mutter : « Le dossier suit la signature »
Durée cible : 0:30. Langue : FR (chute EN, slogan tel quel).
**Prompt Suno** :
```text
Radio commercial, 30 seconds, dynamic male radio host voice in French, commercial tone, close mic, upbeat cyberpunk bed with a cash-register-like synth ping, confident and playful, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: upbeat cyberpunk, synth ping]
[Spoken Ad: French Radio Host, commercial tone]
(Bruitage de terminal bancaire)
Un projet ? Une Orizaune, un garage à Dire Dawa, une seconde Sablone parce que la première est dans un nid ?
Cope Mutter prête en Crédits, et seulement en Crédits : votre Matière, on n'y touche pas, elle est à vous.
Remboursement étalé sur plusieurs cycles, et si le corps change en cours de route, le dossier suit la signature. Vous ne nous échapperez pas, et c'est une bonne nouvelle.
Cope Mutter. Believe in yourself, we believe in your project.
[Music bed fades out]
```

### P04. Titanium ICLI : « Le nôtre reste froid »
Durée cible : 0:20. Langue : FR (chute EN, slogans tels quels).
**Prompt Suno** :
```text
Radio commercial, 20 seconds, dry factual female voice in French, close mic, heavy industrial percussion bed with a metallic ring, one hard hammer hit, cold and solid, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: industrial percussion, metallic ring]
[Spoken Ad: dry female voice, French]
(Bruitage de marteau sur plaque)
Une Sangline mord le métal encore chaud. Le nôtre reste froid.
Blindage de cabine, plaques de Tour, coques d'Orizaune : Titanium ICLI tient là où le reste plie.
Titanium ICLI. Unbreakable. Tearproof.
[Hammer hit, bed ends]
```

### P05. ICLI Station : « Les trois comptoirs »
Durée cible : 0:25. Langue : FR (chute EN, slogan ICLIspace tel quel).
**Prompt Suno** :
```text
Radio commercial, 25 seconds, dynamic male radio host voice in French, commercial tone, close mic, wide orbital synth bed with a docking clamp sound and a soft airlock hiss, confident, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: orbital synth pad]
[Spoken Ad: French Radio Host, commercial tone]
(Bruitage de sas, pince d'amarrage)
Oui, la file d'amarrage est longue. C'est parce que ça vaut le coup.
Premier comptoir : les armes. Deuxième : les médicaments. Troisième : les armures. Et au bout du couloir, la station de warp, pour repartir plus loin que vous n'êtes venu.
ICLI Station, en orbite : la seule adresse où l'on s'amarre. Don't dream it, fly it.
[Airlock hiss, bed fades out]
```

### P06. Iron Dee : « Les fours ne s'éteignent jamais »
Durée cible : 0:25. Langue : FR (chute EN, slogan tel quel).
**Prompt Suno** :
```text
Radio commercial, 25 seconds, dynamic male radio host voice in French, commercial tone, close mic, driving space synth-rock bed with furnace roar and molten metal pour, warm and industrial, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: space synth-rock, furnace roar]
[Spoken Ad: French Radio Host, commercial tone]
(Bruitage de coulée de métal)
Vous ramenez le brut, Iron Dee le fond, le coule, le lamine, et vous paie en Crédits au cours du cycle.
Fer, Obsidienne, Silicium, Cuivre, Aluminium : tout passe par le guichet, et les fours ne s'éteignent jamais. Même quand la Tour d'à côté se rallume pour vous.
Iron Dee. Fast. Big. Comfortable. Why choose ?
[Music bed fades out]
```

### P07. Melrose : « L'Orizaune »
Durée cible : 0:25. Langue : FR (chute EN, slogan tel quel).
**Prompt Suno** :
```text
Radio commercial, 25 seconds, dynamic male radio host voice in French, commercial tone, close mic, powerful synth-rock bed with a spaceship engine ignition and a rising thruster roar, epic and glossy, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: synth-rock, engine ignition]
[Spoken Ad: French Radio Host, commercial tone]
(Bruitage de réacteur qui monte)
L'Orizaune Melrose. Warp en orbite, moteurs en descente, et une coque qui ne demande pas son avis à l'atmosphère.
De la file d'amarrage d'ICLI Station au sable de la zone de test, sans changer de siège. Le seul saut qu'elle ne fait pas, c'est celui de tour en tour : ça, c'est le travail de l'aérotram.
Melrose. Tougher than you can imagine.
[Thruster roar, bed fades out]
```

### P08. Vieego : « Une racine à la fois »
Durée cible : 0:20. Langue : FR (chute EN, slogan tel quel).
**Prompt Suno** :
```text
Radio commercial, 20 seconds, warm male voice in French, close mic, gentle retro-futuristic synth bed with birdsong and leaves, hopeful and green, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: gentle synth, birdsong]
[Spoken Ad: warm male voice, French]
(Bruitage de feuillage, un oiseau)
Un contrat livré, un arbre planté. C'est le programme Vieego au Centre de Purification : ce que le sable a pris, on le rend, une racine à la fois.
Vous ne le verrez peut-être pas pousser. Votre prochain corps, si.
Nature by Vieego Futur.
[Music bed fades out]
```

### P09. Hardware Store : « Il ne mord pas »
Durée cible : 0:20. Langue : FR (chute EN, slogans tels quels).
**Prompt Suno** :
```text
Radio commercial, 20 seconds, fast enthusiastic male voice in French, close mic, bright chiptune-flavoured cyberpunk bed with computer boot beeps, geeky and cheerful, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: chiptune cyberpunk, boot beeps]
[Spoken Ad: fast enthusiastic male voice, French]
(Bruitage de terminal qui démarre)
Le nouveau YGC-7999K est arrivé chez Hardware Store. Il démarre plus vite que vous en salle de récupération, il ne perd jamais votre cargaison, et il ne mord pas.
Terminaux, cartes, écrans, câbles : au Centre et à la Traverse. The new YGC-7999K, why are you waiting ? Upgrade to the futur.
[Music bed fades out]
```

### P10. Sboutique : « Ça se dépense en Crédits »
Durée cible : 0:25. Langue : FR (chute EN, slogans tels quels).
**Prompt Suno** :
```text
Radio commercial, 25 seconds, bright cheerful female voice in French, close mic, glossy mall muzak synth-pop bed with a soft elevator chime, sunny and consumerist, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: glossy synth-pop, elevator chime]
[Spoken Ad: bright cheerful female voice, French]
(Bruitage de carillon d'ascenseur)
Sboutique, au Centre : trois étages d'enseignes, une station du tram Tamil à la porte, et des vigiles qui ne mordent pas.
Un corps neuf, ça s'habille. Une prime de contrat, ça se dépense. Et ça tombe bien : chez Sboutique, ça se dépense en Crédits, jamais en Matière.
Sboutique. Shop. Enjoy. Repeat. At our mall. Spend without limits.
[Music bed fades out]
```

### P11. McFrailenergy : « Avant que la Tour ne le fasse »
Durée cible : 0:20. Langue : FR (chute EN, slogan tel quel).
**Prompt Suno** :
```text
Radio commercial, 20 seconds, loud energetic male voice in French, close mic, aggressive fast cyberpunk bed with a can opening and fizz, hyped and fun, ends on a spoken English tagline
```
**Texte** :
```text
[Music bed fades in: fast cyberpunk, can opening, fizz]
[Spoken Ad: loud energetic male voice, French]
(Bruitage de canette qui s'ouvre)
McFrailenergy ne recharge pas votre noyau. Rien ne le fait, à part la Matière, et la Matière n'est pas à vendre ici.
Mais McFrailenergy vous réveille avant que la Tour ne le fasse, et ça, croyez-nous, c'est mieux.
McFrailenergy. The energy you need.
[Music bed fades out]
```

### P12. I-C News Radio : autopromo « Un seul son vous suit partout »
Durée cible : 0:25. Langue : FR (chute EN, les deux slogans tels quels).
**Prompt Suno** :
```text
Radio station promo, 25 seconds, dynamic male radio host voice in French, close mic, bright space synth-rock bed with a short choir sting of the station name, proud and warm, ends on two spoken English taglines
```
**Texte** :
```text
[Music bed fades in: space synth-rock, choir sting]
[Spoken Ad: French Radio Host, warm and proud]
(Bruitage de communication radio)
Le trafic aérotram, saut par saut. Les cours du Fer, de l'Obsidienne, du Silicium, du Cuivre et de l'Aluminium. Le bulletin de zone d'Ava Signal. Et entre deux mauvaises nouvelles, de la musique, beaucoup de musique.
De la Capitale à la file d'amarrage d'ICLI Station, un seul son vous suit partout.
I-C News Radio. Your ultimate playlist, all day, every day. Your sound, our passion.
[Music bed fades out]
```

---

## Notes d'intégration

### Pistes déjà présentes dans le projet

Ces fichiers existent et sont montables sur la station sans rien générer. La durée est celle lue dans le `.uasset` : c'est exactement la valeur à reporter dans `TrackMeta.Duration`.

| Asset | Dossier | Durée (s) | Usage sur l'antenne |
|---|---|---|---|
| `WAV_Radio_ICNEWSRADIO_01` | `Plugins/Qasset/Content/Audio/WAV/Radio/` | 1502.47 | émission définitive montée par Benja (2026-09-14), la seule piste de la station |
| `ICNEWSRADIO_01` | retiré du projet le 2026-09-14 | 1475.37 | ancienne émission, copie dans `F:\QANGA_Backups\qradio_placeholders_2026-09-14` |
| `I-C_News_YellowWall` | retiré du projet le 2026-09-14 | 259.68 | ancien module d'antenne sur YellowWall, copie sauvegardée |
| `WAV_I-C_News_Radio_Var01` | retiré du projet le 2026-09-14 | 259.68 | ancienne variante, copie sauvegardée |
| `WAV_I-C_News_Radio_Var02` | retiré du projet le 2026-09-14 | 227.24 | ancienne variante, copie sauvegardée |
| `Pub_LifeLoop` | `Plugins/Qasset/Content/Audio/WAV/Radio/` | 24.43 | spot LoopLife déjà enregistré, à garder en rotation |
| `Pub_Tamil` | idem | 15.32 | spot Tamil Station déjà enregistré |

Les six spots de la bible d'antenne (`QRADIO_ICNEWS_ANTENNE.md`, section 4 : Melrose, Iron Dee, IClabs, Titanium ICLI et ICLI Station, SawgeniuS Park) et ses quatre identifiants restent valables. Les douze publicités et huit jingles de ce fichier les complètent, ils ne les remplacent pas.

### Lits instrumentaux

Les six lits de ce fichier (morceaux 01, 03, 05, 07, 09, 11) sont écrits pour passer sous les interventions, comme les blocs `[Instrumental Drop]` des trois scripts de la bible d'antenne. Si Benja veut économiser des crédits Suno, le dossier `Content/Sounds/QangaMusic/MusicForMission/` contient quatorze pistes instrumentales qui ne sont branchées nulle part et qui conviennent : `ICLI` 171.12 s, `Circa` 230.54 s, `Equinox` 153.21 s, `Illuminate` 222.24 s, `Recall` 226.98 s, `Limbo` 156.24 s, `Dizzy` 158.75 s, `Vigilant_Final` 299.47 s, `Wayfarer_Final` 307.17 s, `Rust__Bonus_Track_` 300.01 s, `Dusk__Bonus_Track_` 484.86 s, `Winner_Final` 45.89 s.

### Rappels techniques

- La station existe déjà au catalogue `Content/Systems/QRadio/DA_QRadio_Stations` sous le `StationId` `ICNewsRadio` : ce nom est un contrat, il ne se renomme pas.
- Son MetaSound est `MS_QRadioStation`. Ajouter des pistes revient à remplir son tableau `Music` dans l'ordre du Sommaire ci-dessus, puis à créer une entrée `TrackMeta` par piste, dans le même ordre.
- `Duration` doit être la durée réelle du fichier généré, mesurée après coup (`ffprobe -v error -show_entries format=duration -of csv=p=0 fichier.wav`). C'est elle qui pilote l'horloge de diffusion : une valeur fausse fait dériver la synchronisation entre les clients.
- Niveau cible avant import : -17 LUFS, crête -1.5 dBFS, gain pur sans limiteur (voir `QRADIO_STATIONS_PLAN.md`, section 3).
- La lecture continue n'est pas câblée : tant que ce point n'est pas traité, la station joue une piste puis s'arrête. Voir `QRADIO_STATIONS_PLAN.md`, section 1.

---

*Fichier de station du panel Suno. Créé le 12 septembre 2026. Contenu éditorial : aucun asset ni code du projet n'a été modifié.*
