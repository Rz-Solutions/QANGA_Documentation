# QReport — Architecture

> Plugin runtime (auteur : IolaCorp ; Win64 + Linux uniquement) : l'outillage **de signalement en jeu (IGBR — In-Game Bug Report)**. Il capture un **bundle** (`report.json` + log client complet + screenshot), le persiste localement, puis l'**uploade** vers un récepteur HTTP — **offline-first** : si l'upload échoue, le bundle reste en file et est réessayé. Côté lecture, il agrège les bundles tirés du serveur pour la visionneuse de rapports.

---

## 1. But d'usage

Les joueurs et testeurs doivent pouvoir signaler un bug **depuis le jeu**, avec assez de contexte pour être exploitable : ce qu'ils voyaient (screenshot), où (niveau + position), sur quel serveur, et le **log client complet**. Les contraintes :

- **Offline-first** : un rapport ne doit jamais être perdu si le réseau/récepteur est indisponible — capture locale garantie, upload best-effort avec **file de reprise**.
- **Screenshot du jeu, pas du formulaire** : la capture doit se faire **avant** l'ouverture de l'UI de rapport.
- **Identité serveur** : rattacher le bon **log serveur** au rapport (via un tag serveur stable).
- **Visionneuse unifiée** : afficher côte à côte les rapports en ligne et les bundles hors-ligne tirés du serveur.

---

## 2. Vue d'ensemble / décisions d'architecture

- **API statique en BlueprintFunctionLibrary.** Tout passe par `UQReportTools` (fonctions statiques `BlueprintCallable`) — pas d'acteur/objet à instancier. C'est l'interface entre le formulaire de rapport (BP/UI) et la capture/upload C++.
- **Modèle « bundle » sur disque.** Chaque rapport est un dossier `Saved/Reports/<BundleId>/` contenant `report.json` (métadonnées : reporter, description, importance, catégorie, niveau, position, identité serveur, timestamp), le **log client complet**, et `screenshot.png`. Côté pull, les bundles tirés du serveur vont dans `Saved/PlayerReports/`.
- **Capture en deux temps.** `BeginReportCapture` (au point de déclenchement, **avant** le formulaire) génère l'ID de bundle et **capture l'écran composité** (`FScreenshotRequest::RequestScreenshot`, pour éviter le « screenshot noir » d'un back-buffer non résolu). `SubmitReportBundle` (après le formulaire) assemble `report.json` + log + screenshot et lance l'upload.
- **Upload HTTP authentifié, offline-safe.** L'upload POST vers `Endpoint` porte un en-tête `X-QReport-Token` (secret partagé). En cas d'échec, le bundle **reste en file** localement et est réessayé au prochain lancement (hook post engine-init) ou via `FlushPendingReportBundles`. Une **rétention** borne la croissance du dossier pour un client durablement hors-ligne (les bundles non envoyés sont conservés).
- **Configuration en DeveloperSettings.** `UQReportSettings` (`Config=Game`, section `QReport` sous *Plugins*) porte `Endpoint`, `Token`, `bEnabled` — fiable en éditeur **et** en build packagé (contrairement à une section `.ini` custom). La capture locale a lieu même si l'upload est désactivé.
- **Commande console + intégrations.** Une commande console C++ `report` (et un alias) est enregistrée dans le module (`FAutoConsoleCommandWithWorld`) pour ouvrir le flux de rapport ; le formulaire BP route ensuite vers `UQReportTools`.
- **Plateformes restreintes.** Module limité à `Win64` + `Linux` (exclut Mac/iOS/Android) — cohérent avec les cibles client/serveur du jeu.

---

## 3. Carte des composants

| Élément | Type | Rôle |
|---|---|---|
| `UQReportTools` | `UBlueprintFunctionLibrary` | API : capture, soumission, flush, pull, identité serveur, ouverture de fichiers |
| `UQReportSettings` | `UDeveloperSettings` (`Config=Game`) | `Endpoint`, `Token`, `bEnabled` (section `QReport` sous Plugins) |
| `FQReportBundleInfo` | `USTRUCT` (`BlueprintType`) | Un bundle parsé pour la visionneuse (méta + chemins screenshot/client.log/server.log) |
| `FQReportModule` | `IModuleInterface` | Hooks engine-init / post-load-map (auto-flush) ; commande console ; arrêt propre du ticker |

**API `UQReportTools`** :

| Fonction | Rôle |
|---|---|
| `BeginReportCapture(WorldContext)` | Génère l'ID de bundle + capture le screenshot (**avant** le formulaire). Renvoie l'ID |
| `GetActiveBundleId()` | ID du dernier `BeginReportCapture` |
| `SubmitReportBundle(WorldContext, Reporter, Description, Importance, Category, ServerTag)` | Assemble `report.json` + log + screenshot, persiste, uploade (offline-safe) |
| `FlushPendingReportBundles()` | Réessaie l'upload de tous les bundles en file (auto après engine-init) |
| `GetCommandLineValue(Key)` | Lit un argument `-Key=Value` (ex. `-QReportTag=EU1`) pour l'identité serveur |
| `ListSaveGameSlots(Prefix)` | Liste les slots SaveGame (ex. `IGBR_SaveData_Port_<port>` par serveur) |
| `LoadReportBundles(RootDir)` | Parse les bundles tirés (`Saved/PlayerReports` par défaut) pour la visionneuse |
| `OpenFileInOS(FilePath)` | Ouvre un fichier (client.log / server.log) dans l'app OS |
| `ShutdownReporting()` | Annule le ticker de reprise (appelé au `ShutdownModule`, sûr pour Live Coding) |

---

## 4. Flux de données et cycle de vie

1. **Déclenchement** — le joueur lance un rapport (commande console `report`, touche dédiée, ou bouton de menu). Au **choke point**, le BP appelle `BeginReportCapture` : un `BundleId` est généré et le screenshot du jeu est capturé dans `Saved/Reports/<BundleId>/screenshot.png`.
2. **Formulaire** — l'UI de rapport s'ouvre (le screenshot montre déjà le jeu, pas le formulaire) et collecte reporter/description/importance/catégorie.
3. **Soumission** — `SubmitReportBundle` écrit `report.json` (niveau, position, identité serveur via `ServerTag`/CLI, métadonnées) + copie le **log client complet** dans le bundle, puis POST vers `Endpoint` avec l'en-tête `X-QReport-Token`.
4. **Offline / reprise** — si l'upload échoue (ou `Endpoint` vide), le bundle reste sur disque ; `FlushPendingReportBundles` (au prochain lancement, post engine-init, ou à la demande) le réessaie. Une rétention borne le dossier.
5. **Côté serveur** — un récepteur HTTP (service externe) reçoit les bundles, y ajoute éventuellement `server.log`, et les expose.
6. **Visionneuse** — l'outil de tri tire les bundles, `LoadReportBundles` les parse en `FQReportBundleInfo`, et `OpenFileInOS` ouvre les logs. `ListSaveGameSlots` permet de fusionner les rapports IGBR de plusieurs serveurs.

---

## 5. Points d'intégration

- **Dépendances module** : `Core`, `CoreUObject`, `Engine` (public) ; `HTTP`, `Json`, `ImageWrapper`, `RenderCore`, `ApplicationCore`, `DeveloperSettings` (privé). `ImageWrapper` pour le PNG, `HTTP`/`Json` pour l'upload.
- **Récepteur** : service HTTP externe (un récepteur systemd a été déployé côté EU sur `:8088`) ; l'URL et le secret se configurent dans `UQReportSettings` (`DefaultGame.ini`, section `[/Script/QReport.QReportSettings]`).
- **Identité serveur** : les serveurs dédiés sont lancés avec un tag (`-QReportTag=...`), lu via `GetCommandLineValue` et estampillé dans le `GameState` répliqué pour que le rapport client rattache le bon log serveur.
- **UI / commandes** : le formulaire BP, la commande console `report`, la touche dédiée et le bouton de menu **routent tous** vers `UQReportTools` (avec un cooldown applicatif côté appelant).

---

## 6. Gotchas, invariants et pièges

- **Capturer AVANT le formulaire.** `BeginReportCapture` doit être appelé au point de déclenchement, pas après l'ouverture de l'UI, sinon le screenshot montre le formulaire. La capture utilise `RequestScreenshot` (frame composité) pour éviter le screenshot noir.
- **`Endpoint` doit être correctement formé dans l'`.ini`.** Une URL contenant `//` peut être traitée comme un commentaire selon le parsing ; la **citer** dans le `.ini`. `Endpoint` vide = capture locale uniquement (file).
- **`Token` doit matcher le récepteur.** L'en-tête `X-QReport-Token` doit égaler le `QREPORT_TOKEN` côté serveur, sinon l'upload est rejeté.
- **Offline-first, pas de perte.** Ne pas supposer qu'un rapport est « parti » après `SubmitReportBundle` : vérifier l'envoi réel ; les bundles non envoyés persistent et seront réessayés.
- **Plateformes restreintes.** Le module n'existe pas sur Mac/iOS/Android — toute logique appelant `UQReportTools` doit gérer son absence sur ces cibles.
- **Live Coding.** `ShutdownReporting` annule le ticker de reprise au `ShutdownModule` pour éviter qu'un ticker en file ne tire dans du code de module déchargé (hot-reload).

---

## 7. Fichiers et emplacements

| Fichier | Rôle |
|---|---|
| `Source/QReport/Public/QReportTools.h` / `Private/QReportTools.cpp` | API capture / soumission / upload / pull / parsing |
| `Source/QReport/Public/QReportSettings.h` | `UDeveloperSettings` (`Endpoint`, `Token`, `bEnabled`) |
| `Source/QReport/Private/QReportConsole.cpp` | Commande console `report` (+ alias) |
| `Source/QReport/Public/QReport.h` / `Private/QReport.cpp` | Module : hooks engine-init / post-load-map, auto-flush |
| `QReport.uplugin` | Descripteur : Runtime, Win64 + Linux uniquement |
