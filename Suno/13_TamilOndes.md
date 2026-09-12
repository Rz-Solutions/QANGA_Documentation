# Tamil Ondes (TamilOndes)

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

## Sommaire

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

## Morceaux

### 01. Melrose Taxi

| Champ | Valeur |
|---|---|
| Artiste | Herb Alpine and the Tamil Brass |
| Langue | Instrumental |
| Durée cible | 2:50 |
| Style | easy listening brass, ameriachi 1965 |
| Clin d'oeil | Herb Alpert and the Tijuana Brass, « Tijuana Taxi » (États-Unis) |
| Ancrage lore | Le Taxi Melrose qui klaxonne dans la Traverse saturée, à la sortie de l'aérostation, pendant que la rame repart sans lui. |

**Prompt Suno (champ Style of Music)**
```text
1960s easy listening brass instrumental, ameriachi style, 122 bpm, twin trumpets in tight harmony, marimba, acoustic bass, brushed drums, comic taxi horn honks used as rhythmic accents, bright and playful lounge mood, warm analog tape production, short call-and-response between trumpets and marimba, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: two taxi horn honks, then a marimba vamp]
[Main theme: twin trumpets in harmony, marimba answering]
[Outro: last horn honk and a final brass stab]
```

### 02. Sangline Flea

| Champ | Valeur |
|---|---|
| Artiste | Herb Alpine and the Tamil Brass |
| Langue | Instrumental |
| Durée cible | 2:40 |
| Style | brass novelty, marimba, tuba sautillante |
| Clin d'oeil | Herb Alpert and the Tijuana Brass, « Spanish Flea » (États-Unis) |
| Ancrage lore | Une petite Sangline qui saute de tuile en tuile sur le toit de la cabine ; les passagers sont priés de ne pas la nourrir. |

**Prompt Suno (champ Style of Music)**
```text
playful 1960s easy listening brass instrumental, 128 bpm, bouncy muted trumpet lead over a hopping tuba-like bass line, marimba and vibraphone, light brushed drums, hand claps, cheeky novelty lounge mood, warm analog tape sound, short marimba break in the middle, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: hopping bass line and marimba]
[Main theme: bouncy muted trumpet, hand claps]
[Marimba break]
[Outro: trumpet flourish and one last hop]
```

### 03. Sanglines Keep Fallin' on My Roof

| Champ | Valeur |
|---|---|
| Artiste | Burt Backtrack |
| Langue | Instrumental |
| Durée cible | 3:10 |
| Style | sunshine pop 1969, ukulélé, cordes |
| Clin d'oeil | Burt Bacharach et Hal David, « Raindrops Keep Fallin' on My Head » (États-Unis) |
| Ancrage lore | Les Flyers grattent le toit des cabines d'aérotram entre deux tours ; le guiro fait le bruit des griffes sur le blindage. |

**Prompt Suno (champ Style of Music)**
```text
1969 sunshine pop easy listening instrumental, 108 bpm, ukulele strum and tack piano, muted trumpet carrying the melody, warm string section, guiro scratches and light rimshots like claws tapping a metal roof, gentle optimistic mood with a wink, clean vintage studio production, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: ukulele strum, guiro scratches on a metal roof]
[Main theme: muted trumpet over warm strings]
[Bridge: tack piano and rimshots]
[Outro: guiro scratches fade as the cabin lands]
```

### 04. Popcore

| Champ | Valeur |
|---|---|
| Artiste | Hot Matter |
| Langue | Instrumental |
| Durée cible | 2:45 |
| Style | synth-pop analogique 1972, staccato |
| Clin d'oeil | Gershon Kingsley, « Popcorn » (États-Unis, version Hot Butter) |
| Ancrage lore | Le noyau du cyborg qui bat au rythme de la rame ; la Matière chauffe, la cabine reste fraîche. |

**Prompt Suno (champ Style of Music)**
```text
early 1970s analog synth-pop novelty instrumental, 132 bpm, staccato bubbling square-wave synth lead with fast portamento, bouncy synth bass, crisp drum machine, playful and hypnotic, vintage modular synthesizer textures, short call-and-response between two synth voices, clean retro production, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: bubbling synth pulse like a heartbeat]
[Main theme: staccato synth lead, second synth answering]
[Break: bass and drum machine only]
[Outro: the pulse slows down and stops]
```

### 05. Quiet Tower

| Champ | Valeur |
|---|---|
| Artiste | Martin Dennay and his Exotica Console |
| Langue | Instrumental |
| Durée cible | 3:40 |
| Style | exotica 1957, vibraphone, cris d'oiseaux |
| Clin d'oeil | Martin Denny, « Quiet Village » (États-Unis) |
| Ancrage lore | Une Tour de Relais dans la jungle, la nuit, plaques bleu nuit et structures rouges ; les cris d'oiseaux sont peut-être des Flyers. |

**Prompt Suno (champ Style of Music)**
```text
1950s exotica lounge instrumental, 88 bpm, vibraphone and piano melody, bongos and congas, upright bass, birdcall and frog imitations, jungle night ambience, dreamy and mysterious tiki bar mood, lush vintage stereo production, slow atmospheric intro before the theme, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: night birdcalls and frog imitations over a soft piano]
[Main theme: vibraphone and piano, bongos underneath]
[Percussion break: bongos and congas]
[Outro: birdcalls fade into the jungle]
```

### 06. H.L.L. 817

| Champ | Valeur |
|---|---|
| Artiste | Jean-Jacques Périgée |
| Langue | Instrumental |
| Durée cible | 3:00 |
| Style | space-age pop 1970, basse Moog, breakbeat |
| Clin d'oeil | Jean-Jacques Perrey, « E.V.A. » (France) |
| Ancrage lore | La rame HLL817 qui décolle de l'aérostation pour un saut court vers la tour suivante, et rien de plus loin. |

**Prompt Suno (champ Style of Music)**
```text
1970 space-age electronic pop instrumental, 96 bpm, fat funky analog synth bass, dry breakbeat drums, quirky whistling synth lead, tape-loop sound effects, blips and whooshes, bright futuristic lounge mood, vintage French library-music production, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: whooshes and tape-loop blips, the rame powers up]
[Main theme: funky synth bass and whistling synth lead]
[Break: drums and sound effects]
[Outro: the synth descends and stops at the tower]
```

### 07. Rydeen (Tamil)

| Champ | Valeur |
|---|---|
| Artiste | Yellow Wall Orchestra |
| Langue | Instrumental |
| Durée cible | 3:30 |
| Style | techno-pop japonaise 1979, chiptune doux |
| Clin d'oeil | Yellow Magic Orchestra, « Rydeen » (Japon) |
| Ancrage lore | Le tram Tamil Station qui galope de Qamarac à Costa Rive sous le Mur doré, portes qui carillonnent à chaque arrêt. |

**Prompt Suno (champ Style of Music)**
```text
1979 Japanese techno-pop instrumental, 126 bpm, galloping sequenced synth bass, bright melodic chiptune-style lead, electronic drums with syndrum fills, layered analog pads, uplifting and futuristic, pristine early digital production, clear four-bar hook repeated with variations, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: sequenced synth gallop, doors closing]
[Main theme: bright chiptune lead over the gallop]
[Bridge: syndrum fills and analog pads]
[Outro: the gallop slows into a three-note door chime]
```

### 08. Take the Aerotram

| Champ | Valeur |
|---|---|
| Artiste | Duke Elevator and his Salon Orchestra |
| Langue | Instrumental |
| Durée cible | 3:20 |
| Style | swing de salon 1941, trompette bouchée |
| Clin d'oeil | Duke Ellington et Billy Strayhorn, « Take the A Train » (États-Unis) |
| Ancrage lore | Le seul moyen d'aller à Dire Dawa : l'aérotram, une tour après l'autre, correspondance à chaque saut. |

**Prompt Suno (champ Style of Music)**
```text
1941 swing jazz instrumental arranged for a small salon combo, 160 bpm, piano intro with a descending chromatic figure, muted trumpet melody, clarinet counterlines, walking upright bass, brushed drums, elegant and lively easy listening feel, warm vintage mono-style production, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: solo piano vamp with a descending figure]
[Main theme: muted trumpet, clarinet answering]
[Piano solo over walking bass]
[Outro: short brass shout and a clean stop]
```

### 09. Dire Dawa Choo Choo

| Champ | Valeur |
|---|---|
| Artiste | Glenn Milliamp and his Orchestra |
| Langue | Instrumental |
| Durée cible | 3:15 |
| Style | big band swing 1941, rythme de train |
| Clin d'oeil | Glenn Miller, « Chattanooga Choo Choo » (États-Unis) |
| Ancrage lore | Le train du réseau relais qui quitte la Tour de Relais de Dire Dawa, passe le Grand Pont et file vers le désert. |

**Prompt Suno (champ Style of Music)**
```text
1941 big band swing instrumental, 118 bpm, chugging train rhythm in the rhythm section, clarinet-led reed section melody, muted brass answering, train-whistle trumpet effects and a brush shuffle, cheerful travelling mood, warm vintage ballroom production, builds to a full band finish, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: brushes chug like a train leaving the station]
[Main theme: clarinet-led reed section, muted brass answering]
[Brass break with train-whistle effects]
[Outro: full band finish, brakes and a hiss of steam]
```

### 10. Costa Rive

| Champ | Valeur |
|---|---|
| Artiste | Henri Manciné and the Costa Strings |
| Langue | Instrumental |
| Durée cible | 3:50 |
| Style | valse orchestrale 1961, harmonica chromatique |
| Clin d'oeil | Henry Mancini, « Moon River » (États-Unis) |
| Ancrage lore | Costa Rive, la plage de la Capitale, terminus du tram : le sable, l'eau retenue par le Mur, et la lumière du cycle qui tombe. |

**Prompt Suno (champ Style of Music)**
```text
1961 easy listening orchestral waltz instrumental, 3/4 time, 72 bpm, lush string section, chromatic harmonica carrying the melody, harp glissandi, gentle acoustic guitar arpeggios, dreamy and nostalgic seaside mood, wide vintage stereo production, slow build to a tender string climax, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: harp glissando and soft strings]
[Main theme: chromatic harmonica over strings and guitar]
[String climax]
[Outro: strings fade like waves on the sand]
```

### 11. Last Rame to Djibouti

| Champ | Valeur |
|---|---|
| Artiste | The Monorails |
| Langue | Instrumental |
| Durée cible | 2:55 |
| Style | jangle pop 1966, riff de guitare, tambourin |
| Clin d'oeil | The Monkees, « Last Train to Clarksville » (États-Unis) |
| Ancrage lore | La dernière rame du cycle vers la Tour de Relais de Djibouti ; on court sur le quai, et les portes ne reviennent pas. |

**Prompt Suno (champ Style of Music)**
```text
1966 jangle pop garage instrumental, 156 bpm, twangy electric guitar riff, tambourine, driving bass, handclaps, garage organ stabs, sunny and urgent mood like running for the last departure, vintage mono-style production, the guitar riff returns as the main hook, instrumental, no vocals
```

**Paroles (champ Lyrics)**
```text
[Instrumental]
[Intro: twangy guitar riff and tambourine]
[Main theme: the riff with organ stabs and handclaps]
[Guitar solo]
[Outro: the riff once more, then a closing-doors chime]
```

## Jingles et identifiants de station

Deux jingles seulement : la station n'a pas d'animateur, ce sont les annonces qui font l'habillage. Le carillon à trois notes est l'identité sonore du réseau ; il ouvre aussi chaque annonce.

### J01. Carillon d'identification
Durée cible : 0:06. Langue : FR.
**Prompt Suno**
```text
radio station ident, 6 seconds, three-note ascending chime like a public transport announcement bell, soft synth pad underneath, calm synthetic female announcer says the station name, clean PA system sound, spoken in French
```
**Texte**
```text
(carillon à trois notes, ascendant)
[Spoken: calm synthetic female announcer, PA system]
Tamil Ondes. Le réseau vous accompagne.
```

### J02. Carillon de fin de cycle
Durée cible : 0:10. Langue : FR.
**Prompt Suno**
```text
radio station ident, 10 seconds, three-note descending chime like a public transport bell, soft vocoder choir singing the station name on the same three notes, calm synthetic female announcer closing line, clean PA system sound, spoken in French
```
**Texte**
```text
(carillon à trois notes, descendant)
[Sung jingle: soft vocoder choir on three descending notes]
Ta-mil On-des.
[Spoken: calm synthetic female announcer, PA system]
Fin de cycle. Merci de votre confiance.
```

## Interventions d'animateur (annonces de la Voix du réseau)

Pas d'animateur sur cette station : les douze interventions sont des annonces de bord dites par la Voix du réseau, de 8 à 25 secondes. Même prompt de base pour toutes, seule la langue change ; le lit musical est discret et s'arrête avec la voix. Chaque annonce s'ouvre sur le carillon à trois notes.

### A01. Bienvenue, Citoyen
Durée cible : 0:20. Langue : FR.
**Prompt Suno**
```text
public address announcement for a tram network, 20 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft elevator muzak bed with vibraphone, spoken in French
```
**Texte**
```text
[Music bed fades in: soft elevator muzak, vibraphone]
(carillon à trois notes)
[Spoken: calm synthetic female announcer, PA system]
Citoyen, bienvenue à bord du réseau Tamil Station. Votre synchronisation avec la rame est en cours. Gardez vos membres à l'intérieur de la cabine, y compris ceux de rechange. ICLabs vous assure que toutes les informations diffusées à bord ont été testées et approuvées. Bon voyage.
[Music bed fades out]
```

### A02. Correspondance obligatoire à la tour suivante
Durée cible : 0:20. Langue : FR.
**Prompt Suno**
```text
public address announcement for an aerotram network, 20 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft elevator muzak bed with marimba, spoken in French
```
**Texte**
```text
[Music bed fades in: soft elevator muzak, marimba]
(carillon à trois notes)
[Spoken: calm synthetic female announcer, PA system]
Citoyen, cette navette arrive à la Tour de Relais suivante, où la correspondance est obligatoire. Aucune rame ne poursuit au-delà : le réseau fonctionne par sauts courts, pour votre sécurité et pour la nôtre. Descendez avec l'ensemble de votre cargaison. Tamil Ondes vous accompagne jusqu'à la prochaine tour.
[Music bed fades out]
```

### A03. No direct flight
Durée cible : 0:18. Langue : EN.
**Prompt Suno**
```text
public address announcement for an aerotram network, 18 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft elevator muzak bed with vibraphone, spoken in English
```
**Texte**
```text
[Music bed fades in: soft elevator muzak, vibraphone]
(three-note chime)
[Spoken: calm synthetic female announcer, PA system]
Citizen, there is no direct flight to your destination. There has never been a direct flight to your destination. Aerotram shuttles proceed tower by tower, in short hops. Asking for a direct line at the counter does not create one. ICLabs thanks you for travelling.
[Music bed fades out]
```

### A04. Próxima parada: Sboutique
Durée cible : 0:15. Langue : ES.
**Prompt Suno**
```text
public address announcement for a city tram, 15 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft bossa nova muzak bed, spoken in Spanish
```
**Texte**
```text
[Music bed fades in: soft bossa nova muzak]
(carillón de tres notas)
[Spoken: calm synthetic female announcer, PA system]
Ciudadano, próxima parada: Sboutique, centro comercial. Salida por el lado derecho de la cabina. En Sboutique se paga en Créditos; sus discos de Materia no son dinero y no serán aceptados. Gracias por viajar con Tamil Station.
[Music bed fades out]
```
Sens : prochain arrêt Sboutique, sortie à droite ; on paie en Crédits, les disques de Matière ne sont pas de l'argent et ne sont pas acceptés.

### A05. Aucun disque de Matière sans surveillance
Durée cible : 0:15. Langue : FR.
**Prompt Suno**
```text
public address announcement for a tram network, 15 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft elevator muzak bed with vibraphone, spoken in French
```
**Texte**
```text
[Music bed fades in: soft elevator muzak, vibraphone]
(carillon à trois notes)
[Spoken: calm synthetic female announcer, PA system]
Citoyen, ne laissez aucun disque de Matière sans surveillance. Un disque abandonné sur un siège est un disque recyclé par le réseau. La Matière n'est pas de l'argent, mais elle est la vôtre. Signalez tout objet oublié à la borne de la prochaine tour. Merci.
[Music bed fades out]
```

### A06. In the event of a Sangline attack
Durée cible : 0:20. Langue : EN.
**Prompt Suno**
```text
public address announcement for an aerotram cabin, 20 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft exotica muzak bed with vibraphone, faint scratching sound effects on a metal roof, spoken in English
```
**Texte**
```text
[Music bed fades in: soft exotica muzak, vibraphone]
(three-note chime)
[Spoken: calm synthetic female announcer, PA system]
Citizen, in the event of a Sangline attack, please remain seated. The cabin is armoured. Scratching sounds on the roof are normal and do not require your attention. Your signature is anchored at the nearest Relay Tower; only your cargo is at risk. ICLabs thanks you for your calm.
(faint scratching on the roof)
[Music bed fades out]
```

### A07. Очередь на стыковку у станции ICLI
Durée cible : 0:18. Langue : RU.
**Prompt Suno**
```text
public address announcement for an orbital shuttle network, 18 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft elevator muzak bed with vibraphone, spoken in Russian
```
**Texte**
```text
[Music bed fades in: soft elevator muzak, vibraphone]
(звуковой сигнал из трёх нот)
[Spoken: calm synthetic female announcer, PA system]
Гражданин, очередь на стыковку у станции ICLI переполнена. Время ожидания увеличено. Соблюдайте дистанцию в очереди и не включайте варп на подходе: обломки не имеют приоритета при посадке. ICLabs благодарит вас за терпение. Tamil Ondes.
[Music bed fades out]
```
Sens : Citoyen, la file d'amarrage à ICLI Station est saturée. Le temps d'attente est allongé. Gardez vos distances dans la file et n'enclenchez pas le warp en approche : les débris n'ont pas de priorité à l'amarrage. ICLabs vous remercie de votre patience.

### A08. ICLabs assures you
Durée cible : 0:18. Langue : EN.
**Prompt Suno**
```text
public address announcement for a tram network, 18 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft elevator muzak bed with harp, spoken in English
```
**Texte**
```text
[Music bed fades in: soft elevator muzak, harp]
(three-note chime)
[Spoken: calm synthetic female announcer, PA system]
Citizen, ICLabs assures you that all information broadcast on this network has been tested and approved. Timetables, connections and safety instructions are accurate at the time of your synchronization. Any difference between this information and your experience is a property of your experience. Now, it is time to move.
[Music bed fades out]
```

### A09. Cierre de puertas
Durée cible : 0:12. Langue : ES.
**Prompt Suno**
```text
public address announcement for a city tram, 12 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft bossa nova muzak bed, doors closing sound effect at the end, spoken in Spanish
```
**Texte**
```text
[Music bed fades in: soft bossa nova muzak]
(carillón de tres notas)
[Spoken: calm synthetic female announcer, PA system]
Ciudadano, atención: cierre de puertas. Aléjese de las puertas. Las puertas no esperan a nadie, ni siquiera a un cyborg con cuerpo nuevo. Sujétese a la barra durante el arranque. Gracias.
(puertas que se cierran)
[Music bed fades out]
```
Sens : attention, fermeture des portes ; les portes n'attendent personne, pas même un cyborg au corps neuf ; tenez la barre au démarrage.

### A10. Último tranvía del ciclo
Durée cible : 0:20. Langue : ES.
**Prompt Suno**
```text
public address announcement for a city tram at night, 20 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft late-night lounge muzak bed with vibraphone, spoken in Spanish
```
**Texte**
```text
[Music bed fades in: soft late-night lounge muzak, vibraphone]
(carillón de tres notas)
[Spoken: calm synthetic female announcer, PA system]
Ciudadano, este es el último tranvía del ciclo. No habrá otro hasta el próximo ciclo. Si lo pierde, la sala de recuperación de la torre más cercana permanece abierta, pero dormir allí no está previsto en el protocolo. Buenas noches, y gracias por viajar con Tamil Station.
[Music bed fades out]
```
Sens : dernier tram du cycle, pas d'autre avant le prochain cycle ; si vous le ratez, la salle de récupération de la tour la plus proche reste ouverte, mais y dormir n'est pas prévu par le protocole.

### A11. Terminus : Costa Rive
Durée cible : 0:18. Langue : FR.
**Prompt Suno**
```text
public address announcement for a city tram arriving at a beach terminus, 18 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft seaside easy listening bed with harp and strings, spoken in French
```
**Texte**
```text
[Music bed fades in: soft seaside easy listening, harp and strings]
(carillon à trois notes)
[Spoken: calm synthetic female announcer, PA system]
Citoyen, terminus : Costa Rive, la plage. Tous les passagers sont invités à descendre. Le sable est chaud, l'eau est surveillée, le Mur est à sa place. N'oubliez rien dans la cabine : le réseau ne rend pas les objets, il les recycle. Bonne baignade, sous la surveillance d'ICLabs.
[Music bed fades out]
```

### A12. Аэротрам HLL817 в направлении Дире-Дауа
Durée cible : 0:18. Langue : RU.
**Prompt Suno**
```text
public address announcement for an aerotram station, 18 seconds, calm synthetic female announcer, polite and slightly unsettling, clean PA system sound, three-note chime at the start, very soft elevator muzak bed with marimba, spoken in Russian
```
**Texte**
```text
[Music bed fades in: soft elevator muzak, marimba]
(звуковой сигнал из трёх нот)
[Spoken: calm synthetic female announcer, PA system]
Гражданин, аэротрам HLL817 в направлении Дире-Дауа отправляется по расписанию. Пересадка обязательна на следующей релейной башне. Прямого рейса нет и никогда не было. Займите свои места и закрепите груз. Tamil Ondes желает вам спокойного перелёта.
[Music bed fades out]
```
Sens : Citoyen, la rame HLL817 vers Dire Dawa part à l'heure. Correspondance obligatoire à la Tour de Relais suivante. Il n'y a pas de vol direct et il n'y en a jamais eu. Prenez place et arrimez votre cargaison. Tamil Ondes vous souhaite un vol calme.

## Publicités

Trois spots, dits par la Voix du réseau sur le lit de la station. La pub Tamil Station elle-même existe déjà en enregistrement (`Pub_Tamil`) et n'est pas réécrite ici.

### P01. Melrose : le Taxi qui attend à la sortie de l'aérostation
Durée cible : 0:25. Langue : FR.
**Prompt Suno**
```text
radio commercial, 25 seconds, calm synthetic female announcer over a soft brass easy listening bed with marimba and one taxi horn honk, polite corporate tone, clean PA system sound, spoken in French, ends on the brand name and slogan
```
**Texte**
```text
[Music bed fades in: soft brass easy listening, marimba]
[Spoken: calm synthetic female announcer, PA system]
Citoyen, la rame s'arrête ici. Votre cycle, non. À la sortie de chaque aérostation, un Taxi Melrose vous attend : cabine renforcée, suspension réglée pour tous les sols de la zone de test, compteur en Crédits, jamais en Matière. Le véhicule se conduit seul. Le sable ne discute pas, la Sangline non plus : elle n'a pas le temps de vous rattraper.
(coup de klaxon)
Melrose. Tougher than you can imagine.
[Music bed fades out]
```

### P02. Sboutique : four levels under one roof
Durée cible : 0:25. Langue : EN.
**Prompt Suno**
```text
radio commercial, 25 seconds, calm synthetic female announcer over a bright bossa nova muzak bed with vibraphone, polite corporate tone, clean PA system sound, spoken in English, ends on the brand name and slogan
```
**Texte**
```text
[Music bed fades in: bright bossa nova muzak, vibraphone]
[Spoken: calm synthetic female announcer, PA system]
Citizen, your next stop is Sboutique. Four levels of shops under a roof that no Flyer has ever scratched. Fragrances, hardware, a fountain on every floor, and a cloakroom where your cargo waits for you, whether you come back or come back new. Credits accepted. Matter is not a currency, and it is not accepted.
Sboutique. Shop. Enjoy. Repeat. At our mall.
[Music bed fades out]
```

### P03. Loanicle : une carte, tous les garages
Durée cible : 0:25. Langue : FR.
**Prompt Suno**
```text
radio commercial, 25 seconds, calm synthetic female announcer over a soft exotica muzak bed with bongos and vibraphone, polite corporate tone, clean PA system sound, spoken in French, ends on the brand name and slogan
```
**Texte**
```text
[Music bed fades in: soft exotica muzak, bongos and vibraphone]
[Spoken: calm synthetic female announcer, PA system]
Citoyen, vous descendez à une tour où votre véhicule ne vous attend pas. Loanicle, si. Une carte, et le garage de la tour vous ouvre une Sablone, un SUV ou un Pawad : toutes marques, tous terrains, réglé en Crédits. Votre propre véhicule reste stocké, modifié et entretenu, et vous retrouve à la tour de votre choix. Le corps est neuf, le véhicule aussi.
Loanicle. Une carte, tous les garages.
[Music bed fades out]
```
Loanicle n'a pas de slogan peint dans le jeu (entité canon sans affiche) : « Une carte, tous les garages. » est une accroche inventée pour ce panel, à valider.

## Notes d'intégration

**Pistes existantes du projet à monter sur cette station** (`Plugins/Qasset/Content/Audio/WAV/Radio/`) :

| Asset | Durée (s) | Usage proposé |
|---|---|---|
| `Pub_Tamil` | 15.32 | la pub Tamil Station déjà enregistrée : elle nomme la station de tram et passe telle quelle dans la rotation de pub. Ce panel ne la réécrit pas et ne propose aucune autre pub Tamil Station. |
| `Pub_Tamil_Replique` | 3.16 | réplique courte : à coller après J01 ou J02, ou en sting entre deux annonces. |

Remarques pour Benja :

1. **Trains.** `QRADIO_GUIDE.md` §9 : le train (`QTrain_BaseActor` et ses enfants) est un hôte « toujours ON » du même `UQRadioComponent`, sans édition de Blueprint côté hôte. §11 : la station fixe par train ou par ligne, choisie par le designer et non changeable par les passagers, est listée en feuille de route (le guide écrit « aujourd'hui le train prend la station par défaut »). Tamil Ondes est écrite pour être cette station fixe ; tant que le champ designer n'est pas câblé, elle ne joue sur les trains que si elle est la station par défaut du catalogue. À vérifier sur le build courant avant de compter dessus.
2. **Durées.** `TrackMeta.Duration` = durée réelle mesurée après génération Suno (guide §10 : « doit matcher la durée réelle dans le MetaSound »). Les durées cibles de ce fichier ne sont que des consignes de génération ; Suno déborde souvent de 10 à 20 secondes sur un instrumental.
3. **Émetteur.** Planétaire, comme I-C News et Chill FM (plan §4.1 : émetteur à l'origine, `FalloffStartKm` 9000, `RadiusKm` 10000).
4. **Rotation suggérée.** Une annonce entre chaque morceau, une pub tous les trois morceaux, J01 en tête de boucle, J02 en fin de cycle. La station a peu de morceaux : ce sont les annonces et l'alternance des quatre langues qui masquent la boucle.
5. **Arrêts du tram.** Attestés dans la localisation du jeu (`Content/Localization/Game/en/Game.po`, chaînes `Tamil Station : ...`) : Center, East exit (EastDoor), IClabs Garden, Qamarac, Centinela, Costa Rive. L'arrêt Sboutique des annonces A04 et P02 vient de la commande de ce panel, pas de la localisation : à valider, ou à remplacer par un arrêt attesté au montage.
6. **Fréquence.** La valeur d'affichage proposée dans le plan (tuner) n'est dite nulle part dans ce fichier, conformément au brief.
7. **Vérifications faites avant rendu.** Aucun tiret cadratin ni demi-cadratin (grep = 0) ; aucun nom d'artiste réel dans les prompts (ils n'apparaissent que dans le Sommaire et les lignes « Clin d'oeil ») ; espagnol et russe relus, avec un résumé en français sous chaque texte non francophone.

*Fichier de station du panel Suno. Créé le 12 septembre 2026. Contenu éditorial uniquement : aucun asset, aucun code.*
