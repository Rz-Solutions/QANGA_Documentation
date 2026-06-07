# QRadio — Guide d'utilisation

> But : ajouter / configurer / régler des **radios à la GTA** dans QANGA, sans toucher au C++.
> Pour le *design* et les décisions d'architecture, voir **`QRADIO_ARCHITECTURE.md`**. Ce document-ci est le **mode d'emploi** pratique (designer / level / audio).
>
> Règle d'or : **le code réel fait foi.** Si une affirmation ici diverge du code, c'est le code qui gagne — et on corrige ce document.
>
> *Mis à jour 2026-06-06 : modèle **catalogue central** (remplace l'ancien « une station = un DataAsset ») + **friture/réception** dans le MetaSound + hôtes **véhicules / items / trains** + horloge **temps réel**.*

---

## 1. Vue d'ensemble en 30 secondes

- **Toutes les stations** vivent dans **un seul catalogue** : `DA_QRadio_Stations` (`UQRadioStationCatalog`). Une station = **une ligne** dans ce catalogue (id, MetaSound, où ça se capte). **Aucun acteur posé dans le monde.**
- Un **récepteur** = le composant **`UQRadioComponent`**, posé sur un hôte. Il est déjà sur **3 hôtes** :
  - **`VehicleBase`** → tous les véhicules (radio **conditionnée au moteur**),
  - **`IS_PortableRadio`** (l'item radio) → **toujours allumée**, 3D,
  - **`QTrain_BaseActor`** → tous les trains → **toujours allumée**, 3D.
- **Rien n'est répliqué « à la main ».** La radio **déduit** tout de signaux déjà connus :
  - **moteur allumé ?** → lu sur le `VehicleComponent` de l'hôte (`GetVehicleState`). Pas de `VehicleComponent` (item, train) ⇒ **toujours alimentée**.
  - **qui conduit ?** → contrôle local du pawn (conducteur = 2D).
  - **où en est la diffusion ?** → **horloge temps réel** (tout le monde calcule la même position dans la diffusion).
  - **quelle qualité de signal ?** → distance aux **emitters** de la station → pilote la **friture**.
- L'audio est **rendu localement** sur chaque client (jamais sur un serveur dédié).

**Ressenti visé, par type d'hôte :**

| Hôte | Quand ça joue | Rendu |
|---|---|---|
| **Véhicule** (conducteur, moteur **ON**) | moteur allumé | **2D**, « dans les oreilles » |
| **Véhicule** (extérieur, moteur tournant) | moteur allumé | **3D** atténué depuis le véhicule |
| **Véhicule** (moteur **OFF** / vide) | — | **silence** |
| **Item radio** (posé au sol) | toujours | **3D** depuis l'objet |
| **Train** | toujours | **3D** depuis le train |
| *(tous)* on (re)vient à portée | — | reprend **là où la diffusion en est** (pas de redémarrage à 0) |

---

## 2. Les pièces du système

| Pièce | Type | Fichier / Asset | Rôle |
|---|---|---|---|
| **Catalogue des stations** | `UQRadioStationCatalog` (PrimaryDataAsset) | **`/Game/Systems/QRadio/DA_QRadio_Stations`** | **LA** liste de toutes les stations. ⭐ cœur du système. |
| **Une station** | `FQRadioStation` (struct) | *(une ligne du catalogue)* | id, MetaSound, emitters, pistes, UI. |
| **Composant récepteur** | `UQRadioComponent` | `Plugins/QRadio/.../QRadioComponent.h/.cpp` | Capte + joue. Posé sur les hôtes. |
| **Types de données** | `FQRadioEmitter`, `FQRadioTrackMeta`, `EQRadioEmitterType` | `QRadioTypes.h` | Briques de config (zones de réception, pistes). |
| **Réglages projet** | `UQRadioSettings` | `QRadioSettings.h` → `DefaultGame.ini` | Pointe le catalogue + station par défaut + volume maître. |
| **Le son** | `UMetaSoundSource` | ex. **`/Game/Systems/QRadio/MS_QRadioStation`** | **Playlist + audio + friture**. |
| **Classe son** | `USoundClass` | `/Game/Sounds/_SoundClass/QSClass_Music` | Branche le volume radio sur le **réglage « Musique » du jeu**. |

**Flux de données :**

```
DA_QRadio_Stations (catalogue)            UQRadioComponent (sur l'hôte)
   └─ station[i] (FQRadioStation)            - choisit la station (DefaultStationId, puis SetStation)
        ├─ StationMetaSound ───────────►     - lit l'état moteur (si VehicleComponent) sinon "toujours ON"
        ├─ Emitters (où ça se capte) ──►     - calcule le signal 0..1 selon TA position
        ├─ TrackMeta (durées) ─────────►     - calcule la position "live" via l'horloge temps réel
        └─ DisplayName/Icon/Freq (UI)        - pilote le MetaSound : Index, StartTime, SignalQuality, Play
                                             - rend en 2D (conducteur) ou 3D (sinon)
   (le composant trouve le catalogue via UQRadioSettings.StationCatalog)
```

---

## 3. Comment ça marche au runtime

1. **Gating** — la radio joue si `bRadioEnabled` **ET** le catalogue est chargé **ET** « alimenté » :
   - **Véhicule** : « alimenté » = **moteur allumé**. La vérité = `VehicleComponent.VehicleState` (touche **R** Engine On/Off). Le composant le lit tout seul (aucun câblage BP).
   - **Item / Train** (pas de `VehicleComponent`) : **toujours alimenté**.
2. **2D vs 3D** — si **ton** client contrôle un pawn (tu es au volant) → **2D**. Sinon (extérieur, item, train, autre joueur) → **3D** spatialisé depuis l'hôte. Le passage 2D↔3D **ne coupe pas** le son (on ne réajuste que l'atténuation, en direct).
3. **Position « live »** — chaque récepteur calcule la position courante dans la diffusion depuis une **horloge temps réel commune** (`UtcNow` depuis 2020) et les **durées** (`TrackMeta`). Conséquence : toute radio (pré-placée, lâchée, respawnée) calcule **la même position** ; un récepteur qui (re)démarre **reprend au bon endroit**. *(Synchro parfaite sur une même machine ; entre machines, à la dérive d'horloge près — choix assumé.)*
4. **Réception / friture** — la qualité (0..1) vient des **emitters** de la station selon ta distance (voir §7). Pas d'emitter = **réception parfaite partout**. Le `SignalQuality` est envoyé au MetaSound, qui **fond la musique** et **monte la friture** quand le signal faiblit (voir §6).
5. **Coût** — évaluation **toutes les 0,5 s** (réglable), **jamais par frame**. **Aucun audio** sur un serveur dédié.

---

## 4. Ajouter une station (pas à pas)

> Tout se passe dans **un seul asset** : le catalogue **`DA_QRadio_Stations`**. Plus besoin de créer un DataAsset par station.

1. **Préparer l'audio** : importe tes `WAV` (ex. `/Qasset/Audio/WAV/Radio/`). Note la **durée exacte** de chaque piste.
2. **Faire le MetaSound de station** : le plus simple = **dupliquer `MS_QRadioStation`** (il a déjà la friture + le contrat) et remplacer la liste de morceaux (entrée **`Music`**). Respecte le **contrat §5**. Range-le dans `/Game/Systems/QRadio/` (préfixe `MS_`).
3. **Ouvre `DA_QRadio_Stations`** et **ajoute un élément** au tableau `Stations`. Remplis (voir la table §10) :
   - `StationId` : nom **unique** et stable (ex. `RockFM`). ⚠️ **contrat** (sauvegarde / futur réseau) — ne le renomme pas à la légère.
   - `DisplayName` : nom affiché (passe par la **localisation**).
   - `StationMetaSound` : ton MetaSound de l'étape 2.
   - `TrackMeta` : **une entrée par piste**, `Duration` en secondes = celle réellement utilisée dans le MetaSound (sinon la synchro « live » dérive).
   - `Emitters` : où ça se capte (voir §7). **Vide = audible partout.**
4. *(optionnel)* Pour qu'une station soit celle de **démarrage**, mets son `StationId` dans `DefaultStationId` (§8).

C'est tout. Pas de compilation, pas de C++, **aucun acteur à poser**.

---

## 5. Le contrat MetaSound de station ⚠️ (à respecter)

Le composant **pilote** le MetaSound par paramètres. Ton MetaSound de station **doit exposer ces entrées** (noms **exacts**) :

| Entrée MetaSound | Type | Qui l'écrit | Rôle |
|---|---|---|---|
| `Music` | **WaveAsset (Array)** | toi (dans le MetaSound) | La **playlist** : la liste ordonnée de tes `WAV`. |
| `Index` | **Int32** | le composant | Quel morceau jouer (calculé depuis l'horloge). |
| `StartTime` | **Time** (s) | le composant | À quelle position démarrer le morceau (reprise « live »). |
| `SignalQuality` | **Float** (0..1) | le composant | Force du signal → pilote **volume musique + friture**. |
| `Play` | **Trigger** | le composant | (Re)déclenche la lecture (changement de morceau). *(voir note dette §11)* |

**Ce que ton graphe MetaSound doit faire :**
- À `On Play` : jouer `Music[Index]` en démarrant à `StartTime`.
- **Fondre la musique** par `SignalQuality` (réception).
- Router la sortie vers la **SoundClass `QSClass_Music`** (volume « Musique » du jeu) — *réglage de l'asset, pas du graphe.*

> En plus du contrat, `MS_QRadioStation` expose **5 knobs de friture** réglables librement (voir §6). Le C++ **n'y touche pas** : leurs valeurs par défaut dans l'asset = les valeurs en jeu.

---

## 6. La friture / réception (réglable) 🎚️

`MS_QRadioStation` ajoute un **système d'artefacts de réception** piloté par `SignalQuality` :

- **Hiss continu** : un bruit filtré (`Noise → Biquad` passe-bande) qui **monte quand le signal baisse**.
- **Crackles** : des extraits `WAV` (entrée `RadioEffect`) déclenchés **de temps en temps**.
- **Loi** : `force friture = clamp((1 − SignalQuality) × StaticIntensity, 0, 1)`. Donc à **`SignalQuality = 1` (proche) → friture nulle, musique propre** ; à signal faible → friture pleine (et la musique a déjà fondu).

**Les 5 knobs exposés** (entrées du MetaSound, modifiables sans rebuild) :

| Knob | Rôle | Défaut |
|---|---|---|
| `StaticIntensity` | Vitesse/profondeur de montée de la friture quand le signal tombe | `1.0` |
| `StaticLevel` | **Volume du hiss continu** | `2.5` |
| `ArtifactLevel` | Volume des **crackles** | `1.0` |
| `Cutoff` | Centre du hiss (Hz) | `1256` |
| `Bandwidth` | Largeur du hiss : fin sifflement (bas) ↔ gros hiss large (haut) | `3.0` |
| `RadioEffect` | La liste de `WAV` de crackles/hum | 2 loops `radio_tv_*` |

**Comment régler (live, sans rebuild)** : ouvre `MS_QRadioStation` → **Play** → mets `SignalQuality` à `0.0` (friture pleine) ou `0.4` (mélange) → sélectionne un knob → **Details → Default Value** → ça change **en direct**. Quand c'est bon, **sauve l'asset** (les défauts = les valeurs en jeu).

---

## 7. Définir où une station se capte (les **Emitters**)

Une station n'a **aucun acteur posé dans le monde** : sa couverture est **pure data**, dans `Emitters`. Chaque entrée = une zone.

| Champ (`FQRadioEmitter`) | Type | Rôle |
|---|---|---|
| `Type` | `Local` / `Planetary` / `Space` | Comment la position est ancrée (intention). |
| `PlanetRef` | `TSoftObjectPtr<AActor>` | Corps céleste d'ancrage. **Vide** ⇒ `LocalOffset` est une **position absolue** (cas `Space`). |
| `LocalOffset` | `FVector` (cm) | Position **relative** à `PlanetRef` (ou absolue si vide). |
| `RadiusKm` | `double` (km) | Portée de coupure : au-delà, **plus de signal** (`Sig=0`). |
| `FalloffStartKm` | `double` (km) | Distance où le signal **commence à se dégrader** (la friture monte). Garde `FalloffStartKm <= RadiusKm`. |

**Règle de signal** : `Center = PlanetRef ? PlanetRef.location + LocalOffset : LocalOffset`, puis le signal vaut **1** jusqu'à `FalloffStartKm`, **descend** jusqu'à **0** à `RadiusKm`. Si plusieurs emitters, on garde le **meilleur** signal.

**Exemple en place — la station `ICNewsRadio`** : un emitter `Type=Space`, `PlanetRef` vide, `LocalOffset=(0,0,0)` (origine univers), `FalloffStartKm=9000`, `RadiusKm=10000` ⇒ plein jusqu'à 9000 km de l'origine, fondu 9000→10000, silence au-delà.

**Autres exemples :**
- **Station de ville** : `Type=Local`, `PlanetRef=<la planète>`, `LocalOffset=<antenne>`, `FalloffStartKm=4`, `RadiusKm=8`.
- **Station planétaire** : `Type=Planetary`, `PlanetRef=<la planète>`, gros `RadiusKm`.
- **Station « partout »** (debug / nationale) : **laisser `Emitters` vide** → réception pleine partout.

> ⚠️ Les **trains** parcourent >200 km : leur réception est calculée **à la position du train** comme tout le monde (même système). Pense à des `RadiusKm` cohérents avec les trajets.
> Coordonnées : univers en double précision (LWC) ; une position de pawn est sa position **absolue**. Pour ancrer à une planète, utilise `PlanetRef` (centre résolu au runtime).

---

## 8. Réglages globaux & volume

**Réglages projet** (`UQRadioSettings`) — *Project Settings → QRadio*, ou section dans `Config/DefaultGame.ini` :

```ini
[/Script/QRadio.QRadioSettings]
StationCatalog=/Game/Systems/QRadio/DA_QRadio_Stations.DA_QRadio_Stations   ; LE catalogue
DefaultStationId=ICNewsRadio    ; station de démarrage (un StationId du catalogue)
MasterVolume=1.0                ; multiplicateur radio global (0..1)
ReceptionCheckInterval=0.5      ; période d'évaluation en secondes (mini 0.05) — jamais par frame
```
> ⚠️ `StationCatalog` et `DefaultStationId` **doivent** être renseignés ici (sinon : pas de stations). Le catalogue est référencé **par cette config**, pas par le graphe d'assets → il apparaît **« unused » dans le Reference Viewer** : **ne le supprime jamais** en te fiant à ça (piège classique).

**La chaîne de volume** (du plus global au plus local) :
1. **Réglage « Musique » du jeu** → via la SoundClass `QSClass_Music` (le slider du joueur). *(réglé sur l'asset MetaSound : `Sound Class = QSClass_Music`)*
2. **Volume de l'asset MetaSound** → `MS_QRadioStation.Volume = 0.15` (équilibrage de base ; compense le gain interne ×2.5).
3. **`MasterVolume`** (QRadioSettings) → multiplicateur radio global (injecté dans `SignalQuality`).
4. **`SignalQuality`** (0..1) → réception (distance aux emitters) → fond la musique + monte la friture.

**Debug** : console `qradio.Debug 1` → affiche à l'écran, par hôte :
```
QRadio[Nom] Eng=<0/1> Local=<0/1> Mode=<OFF/2D/3D> Defs=<nb stations du catalogue> Sta=<StationId> Sig=<0..1>
```
gris = OFF, cyan = en lecture. `qradio.Debug 0` pour couper.
*(Note : `Local=0` est normal pour un item/train — l'hôte n'est pas un pawn possédé.)*

---

## 9. Hôtes : véhicules / items / trains

Le **même** `UQRadioComponent` est posé sur 3 familles d'hôtes. Il est **agnostique de l'hôte** : il s'auto-pilote selon ce qu'il trouve.

| Hôte | Asset | Alimentation | Rendu |
|---|---|---|---|
| **Véhicules** | `/Game/Systems/Vehicle/VehicleBase` (hérité par tous : voitures via `HovercraftBase`, motos, vaisseaux…) | **moteur** (`VehicleComponent.GetVehicleState`) | 2D conducteur / 3D sinon |
| **Item radio** | `/Game/Items/.../IS_PortableRadio` | **toujours ON** (pas de moteur) | 3D depuis l'objet. Ramassé = détruit → silence ; reposé → rejoint le direct. |
| **Trains** | `QTrain_BaseActor` (hérité par `QTrain_Warp`, `QTrain_Earth`…) | **toujours ON** | 3D depuis le train. |

**Réglages par hôte** (sur le composant `RadioComponent`) :

| Propriété (`UQRadioComponent`) | Défaut | Rôle |
|---|---|---|
| `bRadioEnabled` | `true` | Interrupteur maître de la radio sur cet hôte. |
| `OutsideFalloffDistance` | `4000` | Distance d'atténuation 3D (cm) pour les auditeurs extérieurs. |

> **Comment il lit le moteur** : à la 1re évaluation, le composant cherche un `VehicleComponent` sur son hôte et lit `GetVehicleState()`. **Trouvé** → gate sur le moteur (= même vérité que le son moteur). **Pas trouvé** (item, train) → **toujours alimenté**. **Aucune édition de Blueprint** n'est nécessaire côté hôte.

---

## 10. Référence des champs

### `UQRadioStationCatalog` (`DA_QRadio_Stations`)
| Champ | Type | Note |
|---|---|---|
| `Stations` | `TArray<FQRadioStation>` | La liste. **Ajouter une station = ajouter un élément.** |

### `FQRadioStation` (une ligne)
| Champ | Type | Note |
|---|---|---|
| `StationId` | `FName` | **Contrat** : unique, stable. |
| `DisplayName` | `FText` | Localisé. |
| `Frequency` | `float` | Affichage tuner (ex. 98.5). `0` = inutilisé. |
| `GenreTags` | `FGameplayTagContainer` | Genre (UI / filtres). |
| `Icon` | `TSoftObjectPtr<UTexture2D>` | Icône UI. |
| `StationMetaSound` | `TSoftObjectPtr<UMetaSoundSource>` | **L'audio + la playlist + la friture** (contrat §5). |
| `Emitters` | `TArray<FQRadioEmitter>` | Réception (§7). Vide = partout. |
| `TrackMeta` | `TArray<FQRadioTrackMeta>` | Durées des pistes (synchro) + titre/artiste (UI). |

### `FQRadioTrackMeta`
| Champ | Type | Note |
|---|---|---|
| `Title` / `Artist` | `FText` | UI « now playing ». |
| `Duration` | `float` (s) | **Doit** matcher la durée réelle dans le MetaSound. |

### `UQRadioComponent` — fonctions Blueprint utiles
| Fonction | Type | Rôle |
|---|---|---|
| `SetStation(FName)` | Callable | Change de station (par `StationId`). Base de la future UI tuner. |
| `SetEngineOn(bool)` | Callable | Forçage manuel du moteur (normalement auto-piloté). |
| `IsPlaying()` | Pure | La radio joue-t-elle ? |
| `GetSignalQuality()` | Pure | Signal courant (0..1). |
| `GetCurrentStationId()` | Pure | Station courante. |

---

## 11. Limites actuelles & feuille de route

**Déjà fonctionnel :** catalogue central, friture/réception réglable, gating moteur (véhicules) + toujours-ON (items/trains), 2D conducteur / 3D sinon, transition entrée/sortie sans coupure, horloge temps réel, volume sur la classe « Musique », debug.

**À finaliser / prévu :**
- 🔁 **Lecture continue** *(prioritaire)* : aujourd'hui la station joue `Music[Index]` puis **s'arrête en fin de piste** (le Wave Player n'est pas en boucle et le trigger `Play` du contrat **n'est pas encore câblé** dans le graphe). À finaliser : **Loop** (station mono-piste) ou **avance de piste** (`On Finished → Index suivant`, multi-pistes).
- 🧍 **Passagers en 2D** : aujourd'hui seul le **conducteur** passe en 2D ; les passagers entendent en 3D. À étendre via les **slots** (`VehicleComponent`).
- 📻 **UI tuner + changement de station** : `SetStation` existe ; reste l'UI + la **réplication du `StationId`** choisi (pour que les autres occupants entendent la même).
- 🚆 **Trains : station fixe designer** : aujourd'hui le train prend la station par défaut ; prévu = une **station fixe par train/ligne** (non changeable par les passagers).
- 🧹 **Dette** : la classe C++ `UQRadioStationDefinition` (ancien modèle) n'est plus utilisée → à retirer au prochain rebuild ; le commentaire « BroadcastTime » dans le code est périmé (contrat réel = §5).

---

## 12. Dépannage — « la radio ne joue pas »

Vérifie dans l'ordre (avec `qradio.Debug 1`) :
1. **Modif C++ ?** → **rebuild complet** (fermer l'éditeur → build → rouvrir). Le **Live Coding ne suffit pas** si la classe a changé de layout.
2. **`Defs=0`** → catalogue introuvable : vérifie `StationCatalog` + `DefaultStationId` dans `DefaultGame.ini` (§8) et que `DA_QRadio_Stations` a au moins une station.
3. **`Eng=0`** sur un **véhicule** → moteur éteint : appuie sur **R**. La radio ne joue **que** moteur allumé (voulu). *(Sur un item/train, `Eng` doit être `1`.)*
4. **`Sig=0`** alors que tu es loin → **hors portée** des emitters de la station (§7). Pour tester partout, laisse `Emitters` **vide**.
5. **Aucune friture audible** → tu es probablement à `Sig=1` (réception parfaite) : la friture **n'apparaît que `Sig<1`** (voulu). Monte `StaticLevel` ou rapproche-toi de `RadiusKm` (§6).
6. **`Mode=3D` au volant** → ce client ne « contrôle » pas le véhicule (passager / possession non transférée). Normal pour un passager (voir §11).
7. **Trop fort / pas affecté par le slider Musique** → vérifie sur le MetaSound : `Sound Class = QSClass_Music` et `Volume ≈ 0.15` (un rebuild de graphe peut les réinitialiser !).
8. **Aucune ligne `QRadio[...]`** → le composant n'est pas sur l'hôte (ou pas en contexte audio : pas de rendu sur serveur dédié).

---

## Références

- **Design / architecture** : `Documentation/QRADIO_ARCHITECTURE.md`
- **Code** : `Plugins/QRadio/Source/QRadio/` (`QRadioComponent`, `QRadioStationCatalog`, `QRadioSettings`, `QRadioTypes`)
- **Assets clés** : `/Game/Systems/QRadio/DA_QRadio_Stations` (catalogue), `/Game/Systems/QRadio/MS_QRadioStation` (station + friture)
- **SoundClass** : `/Game/Sounds/_SoundClass/QSClass_Music`
