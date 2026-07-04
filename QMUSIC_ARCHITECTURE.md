# QMUSIC — Architecture de la musique d'ambiance dynamique (non-diégétique)

> Document de conception. État : **C++ IMPLÉMENTÉ (2026-07-03)**, compilation/validation en cours ; reste la **phase assets éditeur** (SoundClass, SoundMix, DataAssets, `.ini`) puis le test en jeu.
> Plugin cible : **`QMusicDirector`** (nouveau, haut-niveau).
> À lire avant toute implémentation. Met à jour ce document si le code diverge.
> **Vérifié contre le code le 2026-06-26** : tous les systèmes réutilisés existent (symboles réels notés dans le doc). Un seul hook manque et doit être ajouté — un délégué d'observation public sur le manager (§9.4) ; sans lui, le yield « OnRep » n'est pas bindable tel quel.

---

## 1. Objet et périmètre

### Ce qu'on construit
Une couche de **musique d'ambiance non-diégétique** : la musique qui accompagne le joueur selon **l'expérience qu'il est en train de vivre** (exploration calme, poursuite, zone sûre, base, jour/nuit...). Ce n'est pas de la musique jouée dans le lore.

### Ce que ce n'est PAS
- Ce n'est **pas** QRadio. QRadio est **diégétique** : une vraie radio que le personnage entend dans le monde (véhicule, item portable, train), positionnée dans l'univers, synchronisée par horloge serveur. QMusic ne touche pas à QRadio.
- Ce n'est **pas** un nouveau moteur de lecture. Le moteur existe déjà : `QAmbientAudio`. QMusic est le **cerveau** qui le pilote, plus une **couche de mixage** (ducking).

### Principe directeur : 100% client, zéro réplication
La musique d'ambiance est une **décision de présentation locale**. Chaque client choisit et joue sa propre musique à partir d'états **déjà disponibles localement**. Le plugin `QMusicDirector` :
- ne réplique **rien**,
- n'émet **aucun RPC**,
- ne tourne **pas** sur le serveur dédié (early-return si `IsRunningDedicatedServer()`),
- agit uniquement sur le **joueur local**.

La seule chose réseau dans tout le tableau est **pré-existante** : le manager partagé de `QAmbientAudio` (utilisé par la quête et les beds d'environnement). QMusic ne fait que **l'observer localement** (un délégué broadcasté depuis l'`OnRep` du manager — petit renfort §9.4, **aucune réplication ajoutée**) pour savoir quand se taire. Il n'ajoute aucune réplication.

---

## 2. Décisions verrouillées (2026-06-25)

| Décision | Choix retenu |
|---|---|
| Périmètre réseau | **100% local client, aucune réplication.** La couche d'ambiance n'a rien à voir avec le réseau. |
| Ducking (voix/notif) | **SoundMix** (mécanisme éprouvé, local, faible risque) pour la v1. |
| Détection combat | **Wanted + zones taguées seulement** en v1. Détection de combat IA pur = évolution future. |
| Emplacement | **Nouveau plugin `QMusicDirector`** (haut-niveau, dépend des systèmes qu'il observe). |

**Compléments verrouillés (2026-07-03) :**

| Décision | Choix retenu |
|---|---|
| Ducking et radio | **La radio (QRadio) n'est jamais duckée.** Le SoundMix de ducking cible uniquement la SoundClass d'ambiance (`QSClass_AmbientMusic`). |
| Association profil / monde | **Map Monde -> Profil dans `UQMusicDirectorSettings`** (`DefaultGame.ini`) + `DefaultProfile` en fallback. |
| Contenu des profils v1 | **Mapping par défaut accepté** : Exploration = `MusicForMission`, Chase/Tension = `MusicForFight`, SafeZone = `MusicBar`, Lobby = `MusicLobbys`. Pure data, ajustable sans recompiler. |
| Livraison | **D'une traite** : tout le C++ (renforts + plugin + Wanted + zones + hystérésis) avant le premier test in-game, puis phase assets dans l'éditeur. |

---

## 3. Architecture en 3 couches

```
   SIGNAUX LOCAUX (déjà connus du client)        COUCHE C — BUS & DUCKING (local)
   Wanted / Zones / Radio-Véhicule / Notif       SoundClass dédiée "musique non-diégétique"
   + état du manager partagé (callback local)     + SoundMix de ducking quand voix/notif
            │                                                   ▲
            ▼                                                   │ baisse / remonte le volume
   COUCHE B — QMusicDirector (NOUVEAU) = LE CERVEAU             │
   subsystem CLIENT-ONLY, joueur local                         │
   - calcule le MOOD courant (priorité + hystérésis)           │
   - mappe mood -> UQAmbientDefinition (via UQMusicProfile)    │
   - YIELD (se tait) si quête/radio/voix prioritaire           │
            │ joue localement                                   │
            ▼                                                   │
   COUCHE A — QAmbientAudio (RÉUTILISÉ) = LES MAINS ────────────┘
   crossfade 2 AudioComponents, chargement async, MetaSound.
   (Le Director l'utilise en LOCAL, sans passer par le manager répliqué.)

   Coexistence (déjà en place, on n'y touche pas) :
   - DQS ControlAmbientAudioAction -> musique de quête (priorité 90-100, via le manager partagé).
   - QRadio -> diégétique, bus séparé, gated par le moteur du véhicule.
```

**Responsabilité unique par couche :**
- **A. QAmbientAudio** : sait *jouer* un morceau proprement (crossfade, async, synchro). Ne décide rien.
- **B. QMusicDirector** : sait *quoi* jouer et *quand* (mood, priorité, yield). Ne gère pas le bas niveau audio.
- **C. Bus & Ducking** : sait *atténuer* la musique quand une voix passe. Ne choisit pas le morceau.

---

## 4. Le modèle d'anti-collision (le "music mux")

C'est la réponse directe au besoin : **la musique de quête ne doit pas collider avec la musique d'ambiance**, et **la musique baisse quand Tona parle / une notif arrive**.

Deux mécanismes **distincts**, à ne jamais confondre :

### 4.1 Mux par priorité (choisit QUEL morceau joue)
Une seule couche de musique audible à la fois, la plus prioritaire gagne. La pile de priorité de `QAmbientAudio` fait déjà ça pour la quête ; le Director applique la même logique pour ses moods, et **cède** quand une source plus prioritaire joue.

| Source | Priorité | Comportement | Qui la produit |
|---|---|---|---|
| Tona / dialogue / notif critique | la plus haute | **DUCK** (baisse, ne coupe pas) | Couche C (SoundMix) |
| Musique de quête scriptée (DQS) | 90-100 | **REMPLACE** (le Director se tait) | Manager partagé QAmbientAudio (existant) |
| Poursuite police (Wanted) | 60-80 | **REMPLACE** (mood situationnel) | QMusicDirector (local) |
| Zone de danger taguée | 40-60 | **REMPLACE** | QMusicDirector (local) |
| Zone sûre / base / settlement | 30 | base alternative | QMusicDirector (local) |
| Exploration (défaut, biome / jour-nuit) | 0 | couche de fond | QMusicDirector (local) |
| QRadio diégétique (véhicule actif) | n/a | la musique perso **CÈDE** (silence) | QMusicDirector cède le pas |

> ⚠️ **Les priorités 90-100 de la quête sont des valeurs par défaut, pas une bande imposée.** `UControlAmbientAudioAction::OverridePriority` (=100) / `NextPriority` (=90) sont des `int32` `EditAnywhere` **non clampés** : un designer peut poser une quête sous le mood `Chase` (60-80) et casser le « la quête gagne toujours ». → **Réserver/clamper une bande de priorité quête** (≥ tous les moods du Director). La pile du manager (`AQAmbientAudioManager::LayerStack`) est un **stack LIFO strict** : `PushLayer` rejette toute requête de priorité < sommet courant (ce n'est pas une file triée).

### 4.2 Ducking (baisse le VOLUME, ne change pas le morceau)
Quand Tona parle ou qu'une notif s'affiche, on **n'enlève pas** la musique : on la baisse de quelques dB le temps de la voix, puis on remonte. Mécanisme : un **SoundMix** poussé/retiré sur la SoundClass de musique (voir §7).

### 4.3 Règles de "yield" (le Director se tait localement)
Le Director observe **localement** (callbacks, aucun réseau ajouté) et coupe sa propre musique tant que :
- le **manager partagé** joue une musique **prioritaire** (quête / bed scripté) : l'état `ActiveState` (`FQAmbientReplicatedState`) de `AQAmbientAudioManager` est déjà répliqué (`DOREPLIFETIME`) et son rep-notify `OnRep_AmbientState` se déclenche sur chaque client. ⚠️ **`OnRep_AmbientState` est une `UFUNCTION()` protégée qui ne fait que rappeler `ApplyReplicatedState` — ce n'est PAS un délégué bindable.** Le Director ne peut pas s'y abonner tel quel : il faut le **renfort §9.4** (un multicast delegate public sur le manager, broadcasté depuis `OnRep_AmbientState`).
- ⚠️ **Discriminer par priorité, pas par « ça joue ».** Le lit d'ambiance par défaut transite par le **même** `ActiveState` à `Priority=0`. Un yield sur « quelque chose joue ? » rendrait le Director muet en permanence. Le yield doit se déclencher sur `ActiveState.Priority >= seuil_quête` (et `PlayState == Playing`, `ActiveDefinition` non nul).
- la **QRadio** est active pour le joueur local : `UQRadioComponent::IsPlaying()` existe mais renvoie vrai aussi pour une radio **3D distante** (autre conducteur). Il faut le **gater par `IsLocallyControlled()`** sur le pawn possédé, ou ajouter un petit getter public `IsLocalDriver()` (aujourd'hui privé).
- (le ducking voix, lui, ne fait pas taire mais baisser : c'est la couche C).

Résultat : **jamais deux musiques empilées**. La musique de quête prend toujours le dessus ; l'ambiance reprend en crossfade dès que la quête se termine.

---

## 5. Modèle de données

### 5.1 `EQMusicMood` (enum, `uint8`)
Les moods de la v1 (extensible) :
```
Exploration   // défaut, calme, teinté biome / jour-nuit
SafeZone      // base, settlement, ville amicale
Tension       // zone de danger taguée, approche hostile
Chase         // poursuite police (échelonné selon Wanted level)
Lobby         // menus / hub social
// (futurs : Combat, Boss, Underwater, Night-variant...)
```

### 5.2 `UQMusicProfile` (DataAsset)
La table centrale **mood -> musique**. Remplace la "pose manuelle d'un asset + liste de sons".
- `TMap<EQMusicMood, FQMusicMoodEntry>` où chaque entrée pointe une `UQAmbientDefinition` (réutilise tel quel le conteneur de `QAmbientAudio`) + des réglages de transition (fade, priorité, hystérésis spécifique).
- Un **profil par contexte** : un profil "Univers", un "Tutorial", un "Lobby". Le profil actif est choisi par monde via la **map Monde -> Profil de `UQMusicDirectorSettings`** (décision 2026-07-03), avec `DefaultProfile` en fallback.
- Les musiques **déjà triées** (`MusicForFight`, `MusicForMission`, `MusicBar`, `MusicLobbys` dans `Content/Sounds/QangaMusic/`) se rangent directement en `UQAmbientDefinition` -> entrées de mood. La matière première est prête.

### 5.3 `UQMusicDirectorSettings` (UDeveloperSettings)
Section `[/Script/QMusicDirector.QMusicDirectorSettings]` dans `DefaultGame.ini` :
- `DefaultProfile` (soft ref `UQMusicProfile`),
- `MusicSoundClass` (la SoundClass non-diégétique dédiée, voir §7),
- `DuckingSoundMix` (le SoundMix de ducking),
- `DuckAttenuationDb`, `DuckFadeInSeconds`, `DuckFadeOutSeconds`,
- hystérésis par défaut (`CombatLingerSeconds`, `MoodChangeCooldownSeconds`),
- `bEnabled` (kill-switch global).

---

## 6. Le cerveau : `UQMusicDirectorSubsystem` (UWorldSubsystem client-only)

### 6.1 Type et cycle de vie
- **Client-only.** Early-return total sur serveur dédié.
- Subsystem du **joueur local** (pas de réplication, pas d'état partagé). Il agit sur le pawn/PlayerController possédé localement.
- Bind des délégués au démarrage (quand les systèmes sont chargés, via `QGM_AllLoaded`), unbind à la destruction. Toute capture différée passe par `TWeakObjectPtr` (cf. guide C++ moderne).

### 6.2 Entrées v1 (toutes event-driven, aucun poll)
| Signal | Source (telle que câblée) | Hook réel (implémenté 2026-07-03) | Mood déduit |
|---|---|---|---|
| Niveau de recherche | `UQPoliceSubsystem` (WorldSubsystem, broadcast côté client) ¹ | `OnWantedLevelUpdated` (AddDynamic ; filtre pawn local ; seuil `MinWantedLevelForChase`) | `Chase` |
| Entrée/sortie de zone | `UTriggerZoneComponent` ² | hooks statiques natifs `GetOnAnyZoneEnteredNative/ExitedNative` (ajoutés) + champ `FQTriggerZoneData.MusicMood` ("Safe" / "Tension" / "Danger") | `SafeZone` / `Tension` |
| Musique de quête active | `AQAmbientAudioManager` ³ | `OnAmbientStateChangedNative` (ajouté) + `GetActiveState()` ; yield si `Priority >= ManagerYieldPriorityThreshold` (défaut 1 : tout override sauf le lit défaut) | yield (se tait) |
| Radio véhicule active | `UQRadioComponent` ⁴ | `OnListenModeChangedNative` + `IsLocal2D()` (ajoutés) ; rebind sur `APlayerController::OnPossessedPawnChanged` | yield (se tait) |
| Notif / dialogue (Tona) | `UQNotificationManager` ⁵ | `GetOnNotificationDisplayedNative/DismissedNative` (existants, statiques) ; refcount par NotificationID | **ducking** (SoundMix push/pop) |

**Notes de réalisation (2026-07-03, remplacent les notes de vérif du 2026-06-26) :**
1. Les délégués du **service** `UQWantedSystemService` ne broadcastent que **côté serveur** ; le chemin client réel (celui du HUD) est `UQPoliceSubsystem::OnWantedLevelUpdated`, broadcasté sur le subsystem client par `SendWantedClientUpdate()`. C'est lui qui est bindé.
2. Zones : pas de tag natif → champ additif **`MusicMood`** (`FName`, défaut None) sur `FQTriggerZoneData` + **hooks statiques natifs** sur le composant (les délégués par instance restent intacts ; filtrage monde + pawn local dans le Director). ⚠️ Une zone "musique" doit être de type **`Custom`** : une zone `Sound` pousse déjà le manager partagé et ferait céder le Director (seuil de yield par défaut = 1).
3. `OnRep_AmbientState` reste protégé ; le hook ajouté est un délégué **natif** (`FQAmbientStateChangedNative`, pas dynamic : observateur C++ pur) broadcasté depuis l'OnRep (clients) **et** depuis `BroadcastStateChange` (listen server / standalone), jamais sur serveur dédié. Yield gated sur `Priority >= seuil` (le lit défaut est à `Priority=0`).
4. Radio : ajouts additifs `IsLocal2D()` + `OnListenModeChangedNative` (broadcast depuis `ApplyMode` sur changement effectif) ; le Director rebinde à chaque `OnPossessedPawnChanged`. Une radio 3D distante (autre conducteur, poste portable) ne fait **pas** céder la musique.
5. Les délégués QNotification sont broadcastés à l'affichage réel du widget, **tous types confondus** (toast, warning, progress, communication) → le ducking couvre Tona et les notifs sans distinction de type en v1 (refcount par `NotificationID`).

Le mood "Exploration" est le **défaut** quand aucune autre condition n'est vraie. Sa teinte (biome, jour/nuit) est un raffinement (voir §10).

### 6.3 Arbitrage et anti-flicker (hystérésis)
- À chaque changement de signal, le Director recalcule le mood cible = **max de priorité** parmi les conditions vraies.
- **Hystérésis obligatoire** pour éviter le yo-yo : un mood "chaud" (Chase, Tension) **persiste quelques secondes** après la fin de sa condition avant de redescendre ; un cooldown minimal entre deux changements de mood empêche le clignotement quand le joueur entre/sort vite d'une zone.
- Le changement de mood déclenche un **crossfade** via la couche A (déjà géré par `QAmbientAudio`).

### 6.4 Lecture locale
Le Director joue la musique **localement** en réutilisant le composant de lecture de `QAmbientAudio` (crossfade, chargement async, MetaSound déjà résolus), **sans passer par le manager répliqué**. Il construit son état de lecture en local (un `FQAmbientReplicatedState` rempli côté client, temps local) et le pousse via `UQAmbientPlaybackComponent::ApplyReplicatedState` (point d'entrée déjà existant, early-return sur serveur dédié). Voir §9.2 pour le wrapper local nécessaire.

---

## 7. Couche C : bus et ducking

### 7.1 Routage : une SoundClass dédiée à la musique non-diégétique
Aujourd'hui, `QAmbientAudio` ne force aucune SoundClass (le routage dépend de chaque MetaSound). Pour pouvoir **ducker précisément la musique d'ambiance sans toucher au reste** :
- On route la musique du Director vers une **SoundClass dédiée**, enfant de `QSClass_Music` (ex. `QSClass_AmbientMusic`). Elle hérite du slider "Musique" du joueur tout en étant **isolable** pour le ducking.
- Application via `SoundClassOverride` sur les AudioComponents de lecture (petit renfort §9), ou en déclarant la classe dans les MetaSounds d'ambiance.
- **QRadio reste sur `QSClass_Music`** (ou sa propre sous-classe), donc le ducking de l'ambiance ne touche pas la radio (et inversement). **Décision 2026-07-03 : la radio n'est jamais duckée** (diégétique) ; le SoundMix ne cible que `QSClass_AmbientMusic`.

### 7.2 Ducking par SoundMix (v1)
- Un `USoundMix` "DuckMusic" qui applique un override d'atténuation sur `QSClass_AmbientMusic` (et éventuellement `QSClass_Music`).
- Sur `OnNotificationDisplayedNative` (Tona / notif) : `PushSoundMixModifier` + `SetSoundMixClassOverride` avec fade.
- Sur `OnNotificationDismissedNative` : `PopSoundMixModifier` (avec compteur de références si plusieurs notifs se chevauchent).
- **À inspecter** : `QPassiveMix_Base` existe déjà sous `_SoundClass/` ; possiblement réutilisable comme base, à vérifier avant d'en créer un nouveau.
- 100% local. Aucun réseau.

### 7.3 Évolution : AudioModulation
Le plugin **AudioModulation est déjà activé mais inutilisé**. Migration future possible vers des **Control Buses** (intensité continue : combat qui monte progressivement, ducking stackable, slider Musique unifié). Hors périmètre v1, noté comme voie d'évolution propre.

---

## 8. Réutilisation vs création

### Réutilisé tel quel (on ne réécrit rien)
- **`QAmbientAudio`** : moteur de lecture (crossfade, async, priorité, `UQAmbientDefinition`).
- **DQS `ControlAmbientAudioAction`** : la musique de quête y passe déjà ; l'anti-collision est gratuit.
- **`QNotification`** : délégués natifs pour le ducking.
- **`QPolice` / `QTriggerZone`** : délégués Wanted et zones.
- **Les musiques déjà triées** + la hiérarchie SoundClass existante.

### Créé (le strict nécessaire)
- Plugin **`QMusicDirector`** : `UQMusicDirectorSubsystem` (cerveau, client-only), `EQMusicMood` + `FQMusicMoodEntry`, `UQMusicProfile` (DataAsset), `UQMusicDirectorSettings` (DeveloperSettings).
- 1 **SoundClass** dédiée (`QSClass_AmbientMusic`) + 1 **SoundMix** de ducking (phase assets).
- Renforts additifs sur `QAmbientAudio` (§9), plus 2 hooks d'observation additifs (`QTriggerZone`, `QRadio`).

---

## 9. Renforts sur `QAmbientAudio` (chirurgical) — RÉALISÉS le 2026-07-03
1. **`UQAmbientAudioSettings`** (nouveau `UDeveloperSettings`, section `[/Script/QAmbientAudio.QAmbientAudioSettings]`) : `AmbientMusicSoundClass` (soft ref). FAIT.
2. **`SoundClassOverride`** appliqué dans `EnsureAudioComponents` (sur les 2 AudioComponents de lecture) quand `AmbientMusicSoundClass` est configurée. Non configurée = comportement legacy strictement inchangé (zéro régression sur le chemin quête tant que l'asset n'est pas créé). FAIT.
3. **Chemin de lecture local** : AUCUNE modif nécessaire. `ApplyReplicatedState` était déjà public et gated dedicated ; le "wrapper local" vit dans le Director (états `FQAmbientReplicatedState` construits côté client + **epoch par mood** pour reprendre en cours de piste après un yield, façon radio). FAIT.
4. **Délégué d'observation** : `FQAmbientStateChangedNative OnAmbientStateChangedNative` (délégué **natif**, pas dynamic : l'observateur est du C++ pur) + getter `GetActiveState()` (état courant pour le late-bind), broadcastés depuis `OnRep_AmbientState` (clients) et `BroadcastStateChange` (listen server / standalone). FAIT.
5. **`FQAmbientEntry.bRestartOnFinish`** (défaut **false** = comportement inchangé partout) + relance dans le composant de lecture (`OnAudioFinishedNative` -> re-`Play()` si l'état actif dit Playing). Raison : **un graphe MetaSound REFUSE les cycles** (`MetasoundGraphCycleError`), donc « On Finished -> piste suivante » ne peut PAS se câbler dans le graphe ; la source playlist est **one-shot** (une piste = une exécution, le Shuffle garde son sac via Enable Shared State) et c'est le composant qui relance. C'est ce mur qui explique la sortie `Repeat` orpheline du `MS_UniversMusic_BAK` (tentative abandonnée du même pattern). FAIT (2026-07-03, après l'erreur de cycle constatée en jeu).

Ces points sont additifs, ne changent aucune API publique existante, ne cassent pas le chemin quête actuel.

---

## 10. Points ouverts et évolutions futures

### À confirmer avant implémentation
- **Source jour/nuit** : non identifiée de façon fiable (rien de probant côté AtmoScape ; piste `PlanetScape`). Tant que non confirmée, la teinte jour/nuit du mood Exploration reste optionnelle.
- **`QPassiveMix_Base`** : rôle exact à inspecter (réutilisable comme base de ducking ?). _(Existence confirmée : `Content/Sounds/_SoundClass/QPassiveMix_Base`.)_
- ~~**Ducker la radio** quand Tona parle~~ : **résolu (2026-07-03) : non**, la radio n'est jamais duckée.
- ~~**Profil par niveau**~~ : **résolu (2026-07-03)** : map Monde -> Profil dans `UQMusicDirectorSettings` + profil par défaut.
- **Bande de priorité quête** : réserver/clamper (les 90-100 sont des defaults non clampés ; voir §4.1).
- **Mapping zone → mood** : définir une convention sur `ZoneID`/`ZoneName` ou un nouveau champ tag — il n'y a **pas** de tag SafeZone/Tension natif (`EQTriggerZoneType` = `Sound`/`Teleport`/`Custom`), et les zones pilotent déjà `AQAmbientAudioManager` (voir §6.2 note 2).
- **Getter radio local** : exposer `IsLocalDriver()` public sur `UQRadioComponent` (ou gater `IsPlaying()` par `IsLocallyControlled()`) pour distinguer la radio 2D du joueur d'une radio 3D distante.

### Évolutions (hors v1, voie propre)
- **Détection de combat IA** : ajouter un délégué `OnCombatStateChanged` (CombatComponent / QAI) plutôt que poller `bInCombat`. Ouvre le mood `Combat` / `Boss`.
- **Santé** : pas de délégué sur `IQDamageable` aujourd'hui ; un wrapper event-driven ouvrirait un mood "low health".
- **Biome par position** : WorldScape expose un climat par position (température/humidité/altitude) ; permet une teinte d'exploration **par biome** sans zones manuelles.
- **AudioModulation** : ducking et intensité continue via Control Buses.
- **Layering vertical** (remix additif : lit d'ambiance + couche d'intensité par-dessus) au lieu du remplacement couche-unique actuel.
- **Per-player quest scoping** : le manager partagé joue la musique de quête pour tout le monde dans le monde (OK au tuto, discutable à 500 joueurs). À traiter séparément, ce n'est pas dans le périmètre QMusic.
- **Crossfade apparié** : la couche A fade aujourd'hui seulement la piste précédente (mono-buffer : `FadeOutActive` sur l'ancien composant pendant que le nouveau démarre) ; un fondu équal-power apparié lisserait les swaps rapides.

---

## 11. Ce qui ne doit PAS changer (contrats)
- **QRadio** : comportement intouché (diégétique, bus séparé, gating moteur). Seul ajout 2026-07-03 : un hook d'observation additif (`IsLocal2D()` + `OnListenModeChangedNative`, broadcast depuis `ApplyMode`) dont rien dans QRadio ne dépend.
- **QTriggerZone** : comportement intouché. Ajouts additifs 2026-07-03 : champ optionnel `FQTriggerZoneData.MusicMood` (défaut None) + hooks statiques natifs miroirs des deux events existants.
- **Chemin quête audio (DQS)** : `ControlAmbientAudioAction` et sa priorité 90-100 restent le canal de la musique de quête. On ne re-câble pas la quête.
- **API publique de `QAmbientAudio`** : on ajoute (renforts §9), on ne renomme/supprime rien.
- **Aucune réplication** ajoutée pour la couche d'ambiance. Si une future feature exige du partagé, elle passe par le manager existant, pas par le Director.

---

## 12. Plan d'implémentation (validé le 2026-07-03 : exécution d'une traite, C++ complet puis phase assets éditeur)
1. **Bus audio** : créer `QSClass_AmbientMusic` (enfant de `QSClass_Music`) + le SoundMix de ducking ; câbler le ducking sur les délégués `QNotification` (testable seul, sans le reste).
2. **Renforts `QAmbientAudio`** (§9) : SoundClassOverride + entrée de lecture locale + DeveloperSettings.
3. **Plugin `QMusicDirector`** : enum, `UQMusicProfile`, DeveloperSettings, subsystem client-only avec un seul mood (Exploration) qui joue le profil par défaut. Valider la lecture locale + le yield quête/radio.
4. **Entrées situationnelles** : brancher Wanted (Chase) puis zones (SafeZone/Tension), avec hystérésis.
5. **Ranger les musiques existantes** en `UQAmbientDefinition` par mood ; remplir le profil Univers.
6. **Validation 3 contextes** : Lobby, Tutorial, Univers. Vérifier : pas de double musique, ducking propre sur Tona/notif, crossfades, et silence radio.

**État (2026-07-03)** : C++ COMPILÉ (Result: Succeeded) **et phase assets FAITE via le pont CLIScape** (éditeur live). Créés : `QSClass_AmbientMusic` (enfant de `QSClass_Music`) + `QMix_MusicDuck` (duck 0.35, fades 0.3/0.8, enfants inclus) dans `Content/Sounds/_SoundClass/` ; 4 MetaSounds playlist `Content/Systems/QMusicDirector/Music/MS_QMusic_{Exploration,Tension,Chase,Lobby}` (**one-shot** : OnPlay -> Shuffle(Shared State) -> WavePlayer Loop=false -> On Finished -> sortie OnFinished ; la piste suivante est relancée par le composant via `bRestartOnFinish`, car un câblage On Finished -> Next dans le graphe est un cycle REJETÉ par MetaSound ; corrige le bug de loop du `MS_UniversMusic_BAK`, qui reste intouché) ; `DA_QMusic_Moods` (définition 4 entrées) + profils `MP_Universe` / `MP_Lobby` ; sections `DefaultGame.ini` + CDO live. Exploration reprend la playlist Univers réelle (35 pistes du BAK) ; Tension/Chase = 9 pistes MusicForFight (SFX Swish et SC_COMBAT exclus) ; **musiques de bar exclues** (décision user : hors sujet). Preview éditeur OK.

**Reste à valider (étape 6)** : test PIE/jeu (fond musical, ducking sur notif/Tona, yield quête et radio 2D, Chase sur wanted). Points connus : (1) dans l'Univers, le **cue posé dans le niveau persistant** (joue `MS_UniversMusic_BAK`) fera DOUBLE musique avec le Director : à retirer/désactiver au moment du test ; (2) mapping `WorldProfiles` Lobby non renseigné (le lobby jouera MP_Universe tant que la map lobby n'est pas associée, via Project Settings) ; (3) mood `SafeZone` volontairement non mappé en musique v1 (bar exclu) : une zone taguée "Safe" produit du silence tant qu'aucune piste ne lui est affectée ; (4) aucune zone `MusicMood` taguée encore (Tension/SafeZone dormants).

**Fichiers créés/modifiés (C++)** :
- `Plugins/QMusicDirector/` (nouveau plugin) : `.uplugin`, `Build.cs`, module (`QMusicDirector.h/.cpp`), `QMusicDirectorTypes.h` (`EQMusicMood`, `FQMusicMoodEntry`), `QMusicProfile.h`, `QMusicDirectorSettings.h/.cpp`, `QMusicDirectorSubsystem.h/.cpp`.
- `Plugins/QAmbientAudio/` : + `QAmbientAudioSettings.h/.cpp` ; `QAmbientAudioManager.h/.cpp` (délégué natif + `GetActiveState`) ; `QAmbientPlaybackComponent.cpp` (SoundClassOverride) ; `Build.cs` (+DeveloperSettings).
- `Plugins/QTriggerZone/` : `TriggerZoneTypes.h` (+`MusicMood`), `TriggerZoneComponent.h/.cpp` (+hooks statiques natifs).
- `Plugins/QRadio/` : `QRadioComponent.h/.cpp` (+`IsLocal2D`, +`OnListenModeChangedNative`, broadcast dans `ApplyMode`).
- `QANGA.uproject` : plugin `QMusicDirector` activé.

---

*Document de conception QMusic. Source de vérité du design tant que le plugin n'est pas codé. À mettre à jour au fil de l'implémentation.*
