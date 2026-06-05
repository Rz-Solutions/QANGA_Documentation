# QRadio — Guide d'utilisation

> But : ajouter / configurer des **radios à la GTA** dans QANGA, sans toucher au C++.
> Pour le *design* et les décisions d'architecture, voir **`QRADIO_ARCHITECTURE.md`**. Ce document-ci est le **mode d'emploi** pratique (designer / level / audio).
>
> Règle d'or : **le code réel fait foi.** Si une affirmation ici diverge du code, c'est le code qui gagne — et on corrige ce document.

---

## 1. Vue d'ensemble en 30 secondes

- Une **station** = un **DataAsset** (`UQRadioStationDefinition`) qui pointe vers **un MetaSound** (la playlist + l'audio) et décrit **où elle se capte** (les *emitters*).
- Un **récepteur** = un **composant** (`UQRadioComponent`) posé sur un hôte. Aujourd'hui il est sur **`VehicleBase`** donc **tous les véhicules** l'ont (hérité). Demain : un acteur/objet radio au sol (voir §11).
- **Rien n'est répliqué « à la main ».** La radio **déduit** tout de signaux déjà répliqués :
  - **moteur allumé ?** → lu sur le `VehicleComponent` du véhicule (`GetVehicleState`),
  - **qui conduit ?** → contrôle local du pawn,
  - **où en est la diffusion ?** → **horloge serveur** (tout le monde calcule la même position dans la playlist).
- L'audio est **rendu localement** sur chaque client (jamais sur un serveur dédié).

**Ressenti GTA visé :**
| Situation | Rendu |
|---|---|
| Tu conduis (moteur **ON**) | Radio **2D**, « dans les oreilles » |
| Tu es à l'extérieur d'un véhicule dont le moteur tourne | Radio **3D** atténuée depuis le véhicule |
| Moteur **OFF** / véhicule vide | **Silence** |
| Tu remontes dans le véhicule | La radio **continue** là où la diffusion en est (pas de redémarrage à 0) |

---

## 2. Les pièces du système

| Pièce | Type | Fichier / Asset | Rôle |
|---|---|---|---|
| **Composant récepteur** | `UQRadioComponent` | `Plugins/QRadio/.../QRadioComponent.h/.cpp` | Capte + joue. Posé sur les hôtes (véhicules). |
| **Définition de station** | `UQRadioStationDefinition` (PrimaryDataAsset) | `QRadioStationDefinition.h` | Décrit **une** station (id, MetaSound, emitters, pistes). |
| **Types de données** | `FQRadioEmitter`, `FQRadioTrackMeta`, `EQRadioEmitterType` | `QRadioTypes.h` | Briques de config (zones de réception, pistes). |
| **Réglages projet** | `UQRadioSettings` | `QRadioSettings.h` → `DefaultGame.ini` | Volume maître, fréquence d'évaluation. |
| **Le son** | `UMetaSoundSource` | ex. `/Game/Systems/QRadio/MS_QRadio_Station` | **Contient la playlist** et joue le bon morceau à la bonne position. |
| **Classe son** | `USoundClass` | `/Game/Sounds/_SoundClass/QSClass_Music` | Branche le volume radio sur le **réglage « Musique » du jeu**. |

**Flux de données :**

```
DA station (UQRadioStationDefinition)            UQRadioComponent (sur le véhicule)
   ├─ StationMetaSound  ─────────────────────►   - lit l'état moteur (VehicleComponent.GetVehicleState)
   ├─ Emitters (où ça se capte) ─────────────►   - calcule le signal (0..1) selon ta position
   ├─ TrackMeta (durées des pistes) ─────────►   - calcule la position "live" via l'horloge serveur
   └─ StationId / DisplayName / Icon (UI)        - pilote le MetaSound : Index, StartTime, SignalQuality, Play
                                                  - rend en 2D (conducteur) ou 3D (extérieur)
```

---

## 3. Comment ça marche au runtime

1. **Gating moteur** — la radio ne joue **que si le moteur est allumé**. La vérité moteur, c'est `VehicleComponent.VehicleState` (mis à jour quand tu appuies sur **R** « Engine On/Off »). La radio le lit toute seule.
2. **2D vs 3D** — si **ton** client contrôle le véhicule (tu es au volant) → **2D**. Sinon (extérieur, autre joueur) → **3D** spatialisé depuis le véhicule. Le passage 2D↔3D quand tu entres/sors **ne coupe pas** le son (on ne réajuste que l'atténuation, en direct).
3. **Position « live »** — chaque récepteur calcule la position courante dans la playlist depuis l'**horloge serveur** (`GetServerWorldTimeSeconds`) et les **durées des pistes** (`TrackMeta`). Donc tout le monde entend la même chose au même moment, et un récepteur qui (re)démarre **reprend à la bonne position**.
4. **Réception / signal** — la qualité (0..1) vient des **emitters** de la station selon ta distance (voir §7). Pas d'emitter = **réception parfaite partout**. Le signal est multiplié dans le MetaSound (fondu / friture quand c'est faible).
5. **Coût** — évaluation **toutes les 0,5 s** (réglable), **jamais par frame**. **Aucun audio** sur un serveur dédié.

---

## 4. Ajouter une station (pas à pas)

> Exemple de référence à copier : **`DA_QRadio_TestStation`** (+ son MetaSound `MS_QRadio_Station`), dans `/Game/Systems/QRadio/`.

1. **Préparer l'audio** : importe tes `WAV` (ex. dans `/Qasset/Audio/WAV/Radio/`). Note la **durée exacte** de chaque piste.
2. **Faire le MetaSound de station** : duplique `MS_QRadio_Station` (le plus simple) et remplace la liste de morceaux (entrée `Music`). Respecte le **contrat §5**. Garde la sortie sur la **SoundClass `QSClass_Music`**.
3. **Créer le DataAsset** : clic droit → *Miscellaneous → Data Asset* → classe **`QRadioStationDefinition`**. Range-le dans `/Game/Systems/QRadio/` (préfixe `DA_`).
4. **Remplir le DataAsset** (voir la table §10) :
   - `StationId` : un nom **unique** et stable (ex. `RockFM`). ⚠️ c'est un **contrat** (sauvegarde / futur réseau) — ne le renomme pas à la légère.
   - `DisplayName` : nom affiché (passe par la **localisation**).
   - `StationMetaSound` : ton MetaSound de l'étape 2.
   - `TrackMeta` : **une entrée par piste**, avec la **`Duration` en secondes** = celle réellement utilisée dans le MetaSound (sinon la synchro « live » dérive).
   - `Emitters` : où ça se capte (voir §7). **Vide = audible partout.**
5. **Brancher la station sur les récepteurs** : sur le composant radio (du/des véhicules), ajoute le DataAsset à **`AvailableStations`** (voir §9).

C'est tout. Pas de compilation, pas de C++.

---

## 5. Le contrat MetaSound de station ⚠️ (à respecter)

Le composant **pilote** le MetaSound par paramètres. Ton MetaSound de station **doit exposer ces entrées** (noms **exacts**) :

| Entrée MetaSound | Type | Qui l'écrit | Rôle |
|---|---|---|---|
| `Music` | **WaveAsset (Array)** | toi (dans le MetaSound) | La **playlist** : la liste ordonnée de tes `WAV`. |
| `Index` | **Int32** | le composant | Quel morceau jouer (calculé depuis l'horloge serveur). |
| `StartTime` | **Time** (secondes) | le composant | À quelle position démarrer le morceau (reprise « live »). |
| `SignalQuality` | **Float** (0..1) | le composant | Force du signal → à **multiplier** sur la sortie audio. |
| `Play` | **Trigger** | le composant | (Re)déclenche la lecture (au changement de morceau). |

**Ce que ton graphe MetaSound doit faire :**
- À `On Play` **et** sur le trigger `Play` : jouer `Music[Index]` en démarrant à `StartTime`.
- Multiplier la sortie par `SignalQuality` (fondu de réception).
- Router la sortie vers la **SoundClass `QSClass_Music`** (pour suivre le volume « Musique » du jeu).

> **Note de dette technique** : le commentaire dans `QRadioStationDefinition.h` mentionne encore une ancienne interface « `BroadcastTime` / `Play`/`Stop` ». **C'est périmé** : le contrat réel est celui du tableau ci-dessus (Index / StartTime / SignalQuality / Play). À nettoyer dans le code à l'occasion.

> **Pour une reprise « live » et une lecture longue parfaites** : à l'intérieur du MetaSound, câble bien `StartTime` sur l'entrée *Start Time* du **Wave Player** et active la **boucle** (sinon : une piste plus longue que la session finit par s'arrêter, et rallumer le moteur peut repartir au début). C'est le point à finaliser sur `MS_QRadio_Station` (voir §11).

---

## 6. Créer d'autres types de stations

Tout passe par le **même contrat** (§5) ; c'est la façon de remplir `Music` + `TrackMeta` qui change :

- **Station « boucle » (une seule piste)** : 1 `WAV` dans `Music`, 1 entrée `TrackMeta` (sa durée), Wave Player en boucle. Idéal pour une ambiance/loop courte.
- **Station « playlist »** : N `WAV` dans `Music`, N entrées `TrackMeta` (mêmes durées). Le composant choisit l'`Index` courant selon l'horloge serveur ; quand on change de piste, il renvoie `Index`+`StartTime`+`Play`.
- **Station « talk » / mix** : pareil — chaque segment (jingle, voix, morceau) est une entrée `Music`/`TrackMeta`.

> Le **séquençage** (quel morceau, dans quel ordre) est piloté par l'`Index` côté composant à partir de `TrackMeta`. Donc : **l'ordre et les durées dans `TrackMeta` doivent coller à l'ordre et aux durées réels dans `Music`.**

---

## 7. Définir où une station se capte (les **Emitters**)

Une station n'a **aucun acteur posé dans le monde** : sa couverture est **pure data**, dans `Emitters`. Chaque entrée = une zone.

| Champ (`FQRadioEmitter`) | Type | Rôle |
|---|---|---|
| `Type` | `Local` / `Planetary` / `Space` | Comment la position est ancrée (info / intention). |
| `PlanetRef` | `TSoftObjectPtr<AActor>` | Corps céleste d'ancrage. **Vide** ⇒ `LocalOffset` est une **position absolue** (cas `Space`). |
| `LocalOffset` | `FVector` (cm) | Position **relative** à `PlanetRef` (ou absolue si vide). |
| `RadiusKm` | `double` (km) | Portée de coupure : au-delà, **plus de signal**. |
| `FalloffStartKm` | `double` (km) | Distance où le signal **commence à se dégrader** (friture). Garde `FalloffStartKm <= RadiusKm`. |

**Règle de signal** : `Center = PlanetRef ? PlanetRef.location + LocalOffset : LocalOffset`, puis le signal vaut **1** jusqu'à `FalloffStartKm`, **descend** jusqu'à **0** à `RadiusKm`. Si plusieurs emitters, on garde le **meilleur** signal.

**Exemples :**
- **Station de ville** : `Type=Local`, `PlanetRef=<la planète>`, `LocalOffset=<position de l'antenne>`, `FalloffStartKm=4`, `RadiusKm=8`.
- **Station planétaire** (capte sur toute la planète) : `Type=Planetary`, `PlanetRef=<la planète>`, gros `RadiusKm`.
- **Relais orbital** : `Type=Space`, `PlanetRef` **vide**, `LocalOffset=<position absolue univers>`.
- **Station « partout »** (debug / nationale) : **laisser `Emitters` vide** → réception pleine partout.

> Coordonnées : l'univers QANGA est en précision double (LWC) ; une position de pawn est sa position **absolue** dans l'univers. Pour ancrer à une planète, utilise `PlanetRef` (le centre est résolu au runtime).

---

## 8. Réglages globaux & volume

**Réglages projet** (`UQRadioSettings`) — *Project Settings → QRadio*, ou section dans `Config/DefaultGame.ini` :

```ini
[/Script/QRadio.QRadioSettings]
MasterVolume=1.0            ; multiplicateur radio global (0..1)
ReceptionCheckInterval=0.5  ; période d'évaluation en secondes (mini 0.05) — jamais par frame
```
> Par défaut **aucune section** n'est écrite : ce sont les valeurs C++ ci-dessus qui s'appliquent. Ajoute la section seulement pour **surcharger**.

**La chaîne de volume** (du plus global au plus local) :
1. **Réglage « Musique » du jeu** → via la SoundClass `QSClass_Music` (le slider du joueur).
2. **`MasterVolume`** (QRadioSettings) → multiplicateur radio global (injecté dans `SignalQuality`).
3. **Gain interne du MetaSound** → équilibrage figé dans le graphe (sur `MS_QRadio_Station` c'est un gain de balance).
4. **`SignalQuality`** (0..1) → réception (distance aux emitters).

**Debug** : console `qradio.Debug 1` → affiche à l'écran, par véhicule :
`QRadio[Nom] Eng=<0/1> Local=<0/1> Mode=<OFF/2D/3D> Defs=<nb stations> Sta=<station> Sig=<0..1>`
(gris = OFF, bleu = en lecture). `qradio.Debug 0` pour couper.

---

## 9. Intégration véhicules (au sol)

- Le composant **`RadioComponent` (`UQRadioComponent`)** est posé sur **`/Game/Systems/Vehicle/VehicleBase`** (le parent de tout). **Tous les véhicules en héritent** automatiquement — y compris les véhicules au sol via **`HovercraftBase`** (voitures, motos, etc.).
- **Par véhicule**, ce qui se règle sur le composant :

| Propriété (`UQRadioComponent`) | Défaut | Rôle |
|---|---|---|
| `AvailableStations` | (à remplir) | Les stations que **ce** véhicule peut capter (liste de DataAssets). |
| `bRadioEnabled` | `true` | Interrupteur maître de la radio sur cet hôte. |
| `OutsideFalloffDistance` | `4000` | Distance d'atténuation 3D (cm) pour les auditeurs **extérieurs**. |

- **Comment il lit le véhicule** : à la 1re évaluation, le composant trouve le `VehicleComponent` de son hôte et lit `GetVehicleState()` (moteur **On** = joue). C'est **la même vérité** que celle qui pilote le son moteur (`SFX_Engine_Loop`) dans `HovercraftBase`. Donc radio et moteur sont **toujours synchrones**.
- **Aucune édition de graphe Blueprint n'est nécessaire** côté véhicule : la radio se pilote seule.

---

## 10. Référence des champs

### `UQRadioStationDefinition` (le DataAsset station)
| Champ | Type | Note |
|---|---|---|
| `StationId` | `FName` | **Contrat** : unique, stable. |
| `DisplayName` | `FText` | Localisé. |
| `Frequency` | `float` | Affichage tuner (ex. 98.5). `0` = inutilisé. |
| `GenreTags` | `FGameplayTagContainer` | Genre (UI / filtres). |
| `Icon` | `TSoftObjectPtr<UTexture2D>` | Icône UI. |
| `StationMetaSound` | `TSoftObjectPtr<UMetaSoundSource>` | **L'audio + la playlist** (contrat §5). |
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

**Déjà fonctionnel :** gating moteur, 2D conducteur / 3D extérieur, transition entrée/sortie sans coupure, multi-véhicules, volume sur la classe « Musique », debug.

**À finaliser / prévu :**
- 🎚️ **MetaSound `StartTime` + boucle** : câbler `StartTime` sur le Wave Player et activer la boucle de `MS_QRadio_Station`, pour une **reprise « live » exacte** (rallumage moteur) et une **lecture > durée de piste**.
- 🧍 **Passagers en 2D** : aujourd'hui seul le **conducteur** passe en 2D ; les passagers entendent en 3D. À étendre via le système de **slots** (`VehicleComponent`).
- 📻 **UI tuner + changement de station** : `SetStation` existe ; reste l'UI et la **réplication de la station choisie** (pour que les autres occupants entendent la même).
- 🚆 **Trains** : station **fixe choisie par le designer** (pas de changement par les passagers) — à brancher sur le réseau QTrain.
- 🧱 **Acteur / Item radio (futur)** : un **acteur radio posable au sol** ou un **item** (via le système d'items existant du projet) qui **capte exactement comme un véhicule** (même composant, mêmes stations, mêmes emitters). Côté code, ce sera le **même `UQRadioComponent`** posé sur un autre hôte — d'où l'intérêt qu'il soit **agnostique de l'hôte**.

---

## 12. Dépannage — « la radio ne joue pas »

Vérifie dans l'ordre (avec `qradio.Debug 1`) :
1. **Modif C++ ?** → il faut un **rebuild complet** (fermer l'éditeur → build → rouvrir). Le **Live Coding ne suffit pas** si la classe a changé de layout.
2. **`Eng=0`** → moteur éteint : appuie sur **R** (Engine On/Off). La radio ne joue **que** moteur allumé (voulu).
3. **`Defs=0`** → aucune station chargée : vérifie `AvailableStations` sur le composant + que le `StationId` du DataAsset n'est pas vide.
4. **`Sig=0`** alors que tu es loin → tu es **hors portée** des emitters de la station (§7). Pour tester, laisse `Emitters` vide (= partout).
5. **`Mode=3D` au volant** → ce client ne « contrôle » pas le véhicule (passager, ou possession non transférée). Normal pour un passager (voir §11).
6. **Aucune ligne `QRadio[...]`** → le composant n'est pas sur l'hôte (ou pas en contexte audio : pas de rendu sur serveur dédié).

---

## Références

- **Design / architecture** : `Documentation/QRADIO_ARCHITECTURE.md`
- **Code** : `Plugins/QRadio/Source/QRadio/` (`QRadioComponent`, `QRadioStationDefinition`, `QRadioSettings`, `QRadioTypes`)
- **Assets de référence** : `/Game/Systems/QRadio/MS_QRadio_Station`, `/Game/Systems/QRadio/DA_QRadio_TestStation`
- **SoundClass** : `/Game/Sounds/_SoundClass/QSClass_Music`
