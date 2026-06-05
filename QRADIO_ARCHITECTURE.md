# QRADIO — Architecture (conception)

> **Statut : CONCEPTION — rien n'est implémenté.** Ce document fige la conception du système de radio avant écriture du moindre code. Il fait foi pour l'implémentation à venir et doit être mis à jour si une décision change. Date de cadrage : 2026-06-03.
>
> Convention : explications en français, identifiants/commentaires de code en anglais, chaînes joueur via localisation (String Tables / `NSLOCTEXT`).

---

## 1. But d'usage

Stations de musique écoutées **localement** quand le joueur se déplace dans l'univers. Chaque station = un **MetaSound auto-suffisant** (qui contient sa playlist) capté selon la **position** (rayon + atténuation autour de points d'émission définis en data). Embarqué dans les **véhicules** (voitures, vaisseaux, motos, hovercrafts, watercrafts) et dans les **trains** QTrain. Le module côté **personnage** viendra plus tard (cf. §13).

Référence d'expérience : « comme une vraie radio / Fallout au minimum » — on tune une station nommée, le signal se dégrade avec la distance (plein → friture → silence), et plusieurs stations sont disponibles selon où l'on est.

---

## 2. Décisions verrouillées

1. **Nouveau plugin `QRadio`** (système transverse : dépendu par `QVehicles` et `QTrain`). Anatomie Q* standard.
2. **Local + partagé** : l'audio est **joué localement** chez chaque client (jamais répliqué). Seul un **état minimal** est répliqué (`{bPowered, StationId}`) pour partager la station entre occupants.
3. **Réception légère** : position via `GetActorLocation()` (LWC, gratuit), comparaison à une **liste de points en data** sur **timer lent** / **à la demande**. **Aucun** recours à `QSpatialGrid` ni à l'octree `QLevel`. **Aucun acteur/volume placé dans le monde.**
4. **Synchronisation par horloge serveur** : `AGameStateBase::GetServerWorldTimeSeconds()` + époque globale → chaque client calcule la même position de diffusion. Mécanique déjà éprouvée par `QAmbientAudio`.
5. **Le MetaSound contient la playlist**, rendu déterministe par une entrée `BroadcastTime` (pas d'aléatoire libre). Voir le **contrat MetaSound** (§8).
6. **Contrôle de la station** :
   - Véhicules (`VehicleBase`) → **tout occupant assis** (conducteur OU passager) peut changer (dernier gagne).
   - Trains (QTrain) → **station FIXE choisie par le designer** ; les passagers ne changent pas.
7. **Coordonnées d'émetteur** ancrées sur une planète (`PlanetRef + LocalOffset`), résolues au runtime → robuste contigu/sectorisé (§7).

---

## 3. Carte des composants (plugin QRadio)

| Élément | Type | Rôle |
|---|---|---|
| `UQRadioStationDefinition` | `UPrimaryDataAsset` | Une station : identité + émetteurs + MetaSound + métadonnées UI |
| `FQRadioEmitter` / `EQRadioEmitterType` | `USTRUCT` / `UENUM` | Zone(s) de captation (pure data) |
| `FQRadioTrackMeta` | `USTRUCT` | Métadonnées d'un morceau pour l'UI « now playing » |
| `UQRadioComponent` | `UActorComponent` (`BlueprintSpawnableComponent`) | Le lecteur embarqué (posé sur l'hôte) |
| `IQRadioHost` | `UINTERFACE` (BP-implementable) | Adaptateur d'hôte (véhicule ↔ train) |
| `UQRadioSubsystem` | `UGameInstanceSubsystem` | Registre des stations + requêtes de réception |
| `UQRadioFunctionLibrary` | `UBlueprintFunctionLibrary` | Façade BP `QRADIO_*` |
| `UQRadioSettings` | `UDeveloperSettings` (`config=Game`) | Réglages (`DefaultGame.ini`) |
| `LogQRadio` | catégorie de log | Diagnostic, + CVar `qradio.Debug` |

---

## 4. Modèle de données (assets — éditables par le sound designer)

```
EQRadioEmitterType : Local | Planetary | Space

FQRadioEmitter {
  Type           : EQRadioEmitterType
  PlanetRef      : TSoftObjectPtr<AActor>   // ancre planète (vide => LocalOffset est absolu) — cf. QTrain Planet_OriginLocation
  LocalOffset    : FVector (double)         // position relative à l'ancre
  RadiusKm       : double                   // portée jusqu'à la coupure
  FalloffStartKm : double                   // début de la friture (peut devenir une UCurveFloat*)
}

FQRadioTrackMeta {                          // UI uniquement (l'audio vit dans le MetaSound)
  Title    : FText
  Artist   : FText
  Duration : float (s)                      // pour mapper BroadcastTime -> morceau courant sans charger l'audio
}

UQRadioStationDefinition : UPrimaryDataAsset {
  StationId        : FName                       // CONTRAT stable (réplication + save) — ne pas renommer à la légère
  DisplayName      : FText                        // localisé
  Frequency        : float                        // option UI ("98.5")
  Genre/Tags       : FGameplayTagContainer
  Icon             : TSoftObjectPtr<UTexture2D>
  StationMetaSound : TSoftObjectPtr<UMetaSoundSource>  // station auto-suffisante (implémente l'interface QRadio.Station)
  Emitters         : TArray<FQRadioEmitter>
  TrackMeta        : TArray<FQRadioTrackMeta>     // companion UI (durées dupliquées vs le graphe — coût assumé)
  GetPrimaryAssetId() -> ("RadioStation", GetFName())
}
```

Registre : les stations sont des **PrimaryAssets** de type `RadioStation` (comme `UQAmbientDefinition`), énumérées/chargées en async par `UQRadioSubsystem`.

---

## 5. Réception (légère, sans système coûteux)

- **Position** du listener = `GetActorLocation()` du pawn local (FVector `double`, absolu — origin rebasing OFF). Gratuit.
- **Évaluation** : pour chaque station, `signal = max sur ses émetteurs de falloff(Dist(listener, Center_world))`.
  - `Dist <= FalloffStartKm*100000` → `signal = 1` (plein).
  - entre les deux → `signal` interpolé `1→0` (la friture monte).
  - `Dist >= RadiusKm*100000` → `signal = 0` (non captée).
- **Cadence** : pas de Tick par frame.
  - Liste des stations captées : calculée **à la demande** (ouverture du tuner) ou rafraîchie sur le timer lent **uniquement si le tuner est ouvert**.
  - En écoute : on ne teste que **la station courante** (1 à quelques points) sur un **timer lent** (période réglable, ex. 0,25–1 s) pour piloter la friture / couper.
  - Radio éteinte → zéro évaluation.
- Tout en `double`. Jamais repasser par `float` pour les distances à l'échelle univers.

---

## 6. Synchronisation des pistes

- Une station = **diffusion continue dans le temps**. La position courante est **fonction pure** de l'horloge serveur synchro : `BroadcastTime = GetServerWorldTimeSeconds() - Epoch` (époque globale).
- Le `UQRadioComponent` pousse `BroadcastTime` (et `SignalQuality`) dans le `StationMetaSound`, puis `Play`. Le graphe se positionne au bon morceau/offset (cf. §8) → **tous les clients entendent la même chose**.
- **On ne réplique jamais l'audio**, seulement `{bPowered, StationId}` (cf. §10). Pas de timestamp par tune (l'époque est globale).
- **Drift inter-clients** : chaque joueur n'entend que **sa propre sortie** → un écart de quelques dizaines de ms entre machines est inaudible. Le seek au tune-in suffit ; pas de re-sync agressive nécessaire.

---

## 7. Coordonnées de l'univers (inconnue fermée)

Vérifié dans le code : `p.EnableMultiplayerWorldOriginRebasing=False` + WorldScape en double précision (ECEF, `DVector` relatif au centre planète) + `AQTrain_Spline_Track::Planet_OriginLocation` (FVector) + distances `double` en cm. ⇒ **l'univers est un espace LWC unique** ; une position planétaire est **ancrée sur l'origine d'une planète** puis résolue en monde absolu.

Conséquence radio : `FQRadioEmitter` stocke `PlanetRef + LocalOffset`, résolus **au runtime** via le transform live de la planète :
`Center_world = PlanetRef ? PlanetRef->GetActorLocation() + LocalOffset : LocalOffset`.
→ Correct que l'univers soit contigu **ou** sectorisé (on ne fige aucune hypothèse). Émetteurs `Space` : `PlanetRef` vide, `LocalOffset` = position absolue.

---

## 8. Contrat MetaSound — `QRadio.Station` (zone de danger §4 du projet)

Le C++ pilote chaque station par des **paramètres nommés**. Ces noms sont un **contrat** : les renommer côté asset casse le C++ **sans erreur de compilation**.

Interface **`QRadio.Station`** (MetaSound Interface — déclarée et vérifiable dans l'éditeur) :
- Entrée `BroadcastTime` : `float` (secondes depuis l'époque) — pilote le positionnement déterministe.
- Entrée `SignalQuality` : `float` 0–1 — le graphe mixe audio propre ↔ friture/statique.
- Triggers `Play` / `Stop`.

Exigences du graphe (authoring sound designer) :
- À partir de `BroadcastTime` + **durées des morceaux bakées dans le graphe**, calculer (morceau courant, offset) et démarrer le Wave Player au bon **Start Time** (rejoindre la diffusion en cours).
- Avancer la playlist de façon **déterministe** (ordre fixe ou shuffle seedé).

Parades anti-régression :
- **MetaSound modèle** dupliquable par les designers (interface déjà branchée).
- **Validation au chargement** dans `UQRadioComponent`/`UQRadioSubsystem` : vérifier que la source implémente `QRadio.Station` ; sinon `UE_LOG(LogQRadio, Error, ...)` et station ignorée.

---

## 9. Hôtes & intégration

Le `UQRadioComponent` est générique. Deux **modes de contrôle** :

| Hôte | Pose du composant | Mode | Contrôle |
|---|---|---|---|
| Véhicules (`VehicleBase`, parent `Pawn`) | **une seule fois sur `VehicleBase`** (hérité par Spaceship/Hovercraft/Bike/Watercraft) | `SeatedOccupants` | tout pawn dans un `VehicleSlot` → `Server_SetStation` (serveur valide), dernier gagne |
| Trains (cart, ex. `BP_TrainCartBase`/`BP_FlyingTrain`) | sur le cart | `DesignerFixed` | aucun contrôle runtime ; `DefaultStationId` posé par le designer (sur le train ou la ligne/track), appliqué serveur au `BeginPlay` + répliqué |

Interface `IQRadioHost` (implémentée par `VehicleBase` et le cart) :
- `QRadio_IsLocalListener() -> bool` : le joueur local est-il à bord ? (véhicule : occupant ; train : assis dans un `AQTrainSeatActor` du train).
- `QRadio_CanControl(APawn*) -> bool` : véhicule = vrai si dans un `VehicleSlot` ; train = **toujours faux**.

`VehicleBase` porte déjà `VehicleComponent`, `VehicleSlot_Driver` (+ `GetAllSlots`/`GetExitSlot`) et 6 `AudioComponent` → ajouter la radio est sans friction.

---

## 10. Réseau & autorité

- **Server-authoritative** : un occupant autorisé appelle `Server_SetStation(StationId)` / `Server_SetPowered(bool)` ; le serveur valide (via `QRadio_CanControl`) et met à jour l'état répliqué.
- État répliqué (minimal) : `bPowered : bool`, `CurrentStationId : FName`. (Pattern identique à `QVehicleSirenComponent`.)
- **Serveur dédié** : le composant ne fait **aucun audio, aucun timer** ; il porte/réplique seulement l'état.
- Clients : réagissent via `OnRep` → (dé)marrent la lecture locale, calée sur `BroadcastTime`.

---

## 11. Performance — garde-fous (obligatoires)

Le concept est **O(1) par client**, pas O(véhicules). Pour le garantir :
1. **Serveur dédié** : zéro audio, zéro timer (juste l'état).
2. **Dormant** tant que `QRadio_IsLocalListener()` est faux : pas d'`UAudioComponent`, pas de timer. Activation à l'embarquement, coupure à la sortie.
3. **Jamais de Tick par frame** : un timer, et seulement quand actif.
4. **Chargement asynchrone** (`FStreamableHandle`) du `StationMetaSound` → pas de hitch au tune.
5. **Stop/clean à `EndPlay`** (QLevel streame les hôtes en entrée/sortie).

---

## 12. Points d'attention / dette potentielle

- **Couplage par chaîne (§4 du projet)** : les noms de paramètres `QRadio.Station` et le `StationId` sont des contrats. Toute recherche de référents par **chaîne** avant renommage. Validation runtime obligatoire.
- **Duplication des durées** : `FQRadioTrackMeta.Duration` (UI) vs durées bakées dans le graphe MetaSound — doivent rester cohérentes. Alternative à étudier : output `CurrentTrackName` exposé par le MetaSound (zéro duplication, plus de plomberie).
- **Ordre de chargement** : `UQRadioSubsystem` doit s'enregistrer auprès de `QGameManager` selon la convention (à vérifier au moment de l'implémentation).

---

## 13. Différé / hors scope (v1)

- **Module radio côté personnage** : reporté (l'arbre de talent des modules du personnage doit d'abord être refait). Le `UQRadioComponent` est conçu pour s'y poser plus tard (mode `Local`, non répliqué).
- **Radio audible depuis l'extérieur du véhicule** (émetteur spatialisé) : possible évolution ; v1 = lecture locale 2D pour l'occupant.
- **Synchro fine sample-accurate** : non nécessaire (cf. §6).

---

## 14. Sources vérifiées (ancrages)

- Audio réutilisable : `Plugins/QAmbientAudio/...` — `UQAmbientDefinition` (PrimaryDataAsset + entrées MetaSound/atténuation), `UQAmbientPlaybackComponent` (crossfade, async, `GetServerWorldTimeSeconds()` + `ServerStartTime`).
- Composant audio par véhicule : `Plugins/QVehicles/Source/QVehicles/Public/QVehicleSirenComponent.h`.
- Hiérarchie véhicules (confirmée éditeur) : `Content/Systems/Vehicle/VehicleBase` (parent `Pawn`) ← `SpaceshipBase`, `HovercraftBase`, `BikeBase`, `WatercraftBase`. Places : `VehicleSlot`.
- Trains : `Plugins/QTrain/...` — `AQTrain_Train` (acteur sur spline, sans conducteur, server-authoritative), `AQTrainSeatActor` (places), `AQTrain_Spline_Track` (`IsPlanetaryTrack`, `Planet_OriginLocation`, distances `double` en cm, warp = vitesse). Réseau relais = `QTrain_SubSystem` + stations `QTrain_*_Station`.
- Coordonnées : `Config/DefaultEngine.ini` (`p.EnableMultiplayerWorldOriginRebasing=False`), WorldScape (ECEF/`DVector`).
- À NE PAS utiliser : `QSpatialGrid` (lourd, désactivé par défaut).

---

## 15. Plan d'implémentation par étapes (à exécuter SEULEMENT après feu vert)

> Principe : squelette d'abord, **valider chaque couche avant la suivante**, isoler les 2 risques (faisabilité MetaSound = spike/Phase 2 ; édition de `VehicleBase` = Phase 6). Validation systématique : compilation (Qanga / QangaEditor / QangaServer), `validate_blueprint_graph` + `audit_blueprint` pour le BP, test des 3 rôles réseau quand pertinent, vérif des référents.

**Spike préalable (recommandé, AVANT tout C++) — dérisquer le MetaSound.** Prototyper en éditeur l'interface `QRadio.Station` + 1 MetaSound qui, piloté à la main par `BroadcastTime`, rejoint la bonne piste au bon offset et mixe `SignalQuality`. Si infaisable proprement → bascule sur « playlist en data » (le fork écarté) **avant** d'écrire du C++. C'est le **go/no-go de la synchro**.

**Phase 0 — Scaffolding `QRadio`.** Plugin Runtime, `.uplugin`, `Build.cs` (Core/CoreUObject/Engine/MetasoundEngine/AudioMixer/NetCore/GameplayTags), `LogQRadio`, Public/Private, header umbrella. QRadio reste **bas niveau** : il ne dépend PAS de QVehicles/QTrain — c'est le contenu/les hôtes qui dépendent de QRadio. *Validation : compile à vide, 3 cibles.*

**Phase 1 — Data model.** `EQRadioEmitterType`, `FQRadioEmitter`, `FQRadioTrackMeta`, `UQRadioStationDefinition` (+ PrimaryAssetType `RadioStation`), `UQRadioSettings`. *Validation : créer/éditer un asset station ; section `[/Script/QRadio...]` présente dans `DefaultGame.ini`.*

**Phase 2 — Interface MetaSound + station de test.** Officialiser `QRadio.Station` (issue du spike), 1 MetaSound modèle + 1 station de test (2-3 morceaux). *Validation : seek + friture corrects, pilotés à la main.* (Travail surtout sound-design, parallélisable avec 0/1.)

**Phase 3 — Composant lecteur LOCAL (sans réseau).** `UQRadioComponent` : AudioComponent, load async, push `BroadcastTime`+`Play`, `SignalQuality` ; réception locale (timer lent, résolution `PlanetRef+offset`, falloff, coupure). `UQRadioFunctionLibrary` (`QRADIO_*`). *Validation : PIE standalone, composant sur acteur de test, on tune et on se déplace → friture/coupure/lecture OK ; garde-fous perf en place.*

**Phase 4 — Subsystem + registre.** `UQRadioSubsystem` : chargement async des stations, `FindStation`, `GetReceivableStations(Position)`, **validation du contrat** (rejet + `LogQRadio` si non conforme). Enregistrement auprès de QGameManager si l'ordre de chargement l'exige. *Validation : liste captée à la position ; asset non conforme rejeté proprement.*

**Phase 5 — Réseau (partage + autorité).** État répliqué `{bPowered, CurrentStationId}` + `Server_SetPowered`/`Server_SetStation` (validés via `QRadio_CanControl`), `OnRep` → lecture locale synchro. *Validation : 3 rôles ; 2 clients dans le même véhicule = même station synchro ; serveur dédié = 0 audio.*

**Phase 6 — Intégration hôtes (⚠️ phase la plus à risque).** `IQRadioHost` sur `VehicleBase` (mode `SeatedOccupants`) → poser le composant **une seule fois** ; sur le cart de train (mode `DesignerFixed` + `DefaultStationId`). ⚠️ `VehicleBase` est un BP central et lourd : `get_detailed_blueprint_summary` + `get_asset_dependencies` AVANT, `validate_blueprint_graph` + `audit_blueprint` APRÈS, **vérifier la non-régression des véhicules existants** (sirène, 6 SFX, slots…). *Validation : voiture + vaisseau héritent la radio ; un passager change la station ; un train diffuse sa station fixe.*

**Phase 7 — UI tuner.** `WBP_` tuner : stations captées, sélection, now-playing (`TrackMeta` + `BroadcastTime`), jauge de signal ; Enhanced Input. *Validation : UX complète en jeu.*

**Phase 8 — Contenu & réglages.** Vraies stations par genre (MetaSounds + assets), émetteurs posés en data (`PlanetRef + LocalOffset`), réglages `DefaultGame.ini`. *Validation : balade en jeu, captation cohérente selon la position.*
