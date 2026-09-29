# Audit de sécurité, d'accessibilité VoiceOver et comparaison avec Audiobookshelf officiel

- **Projet audité :** `math65/audiobookshelf-mac`, commit `ddac10e` (branche `main`). C'est une copie identique du dépôt de son auteur, Artur Grochau.
- **Référence officielle :** `advplyr/audiobookshelf` v2.37.1 (serveur et application web).
- **Date :** 29 septembre 2026.
- **Méthode :**
  - lecture de tout le code, en plusieurs audits parallèles (réseau, système, fichiers et données du serveur, interface et historique git, VoiceOver) ;
  - vérification à la main, dans le code, des trouvailles les plus graves ;
  - recherche dans la documentation Apple (API d'accessibilité, recommandations de design, sessions WWDC, critères des « étiquettes d'accessibilité » de l'App Store) ;
  - comparaison avec le serveur et l'app web officiels, avec tes propres projets et avec AudioBooth, tous lus sans être modifiés.
- **Limite :** l'environnement d'audit tourne sous Linux, sans Mac ni compilateur Swift. Rien n'a été lancé. Les points marqués **(à confirmer sur Mac)** dépendent d'un comportement de macOS que je n'ai pas pu tester.
- **Aucune modification du code de l'app** n'a été faite. Ce fichier est le seul ajout.

---

## 0. Résumé en une page

### Peut-on faire confiance à l'app ?

**Oui pour l'honnêteté du code : rien de malveillant.**

- Pas de télémétrie, pas de pistage, aucune dépendance tierce.
- L'app ne contacte que ton propre serveur.
- L'historique git est propre : aucun secret, aucun fichier caché.
- Les polices et les images sont authentiques, et plusieurs sont identiques octet pour octet à celles d'Audiobookshelf officiel.

**Non pour la robustesse : il y a de vrais défauts de sécurité.** Aucun n'est une porte dérobée. Ce sont des erreurs de conception typiques d'un projet récent (3 jours, écrit en grande partie avec une IA), sans relecture extérieure.

Les 3 défauts les plus graves :

1. **Vol du mot de passe via l'adresse locale (Élevé, confirmé).** Ça ne te concerne que si le champ « Local Address » est rempli. Dans ce cas, un appareil sur un autre Wi-Fi (café, hôtel) qui prend la même adresse reçoit ton mot de passe Audiobookshelf en clair.
2. **Un serveur malveillant peut faire effacer des dossiers de ton Mac (Élevé, à confirmer sur Mac).** Deux chemins mènent à ce résultat : la déconnexion, et les téléchargements. Il faut un serveur piraté, ou quelqu'un qui intercepte une connexion en `http://`.
3. **Plantages volontaires (Moyen, confirmé).**
   - Un lien piégé sur une page web fait planter l'app.
   - Des données serveur aberrantes la font planter à chaque lancement, car elles restent dans le cache.

### Accessibilité VoiceOver

**Pour l'instant, l'app n'est pas vraiment utilisable avec VoiceOver.** Il n'y a que 3 réglages d'accessibilité dans toute l'app. Les 4 blocages :

- ouvrir un livre au clavier est impossible, et avec VoiceOver ce n'est pas garanti ;
- la barre de progression est invisible pour VoiceOver ;
- les chapitres, les signets et la minuterie sont inutilisables ;
- Échap ferme le lecteur par erreur et les flèches changent le volume sans rien dire.

La bonne nouvelle : les commandes du menu, les touches média et le Centre de contrôle marchent bien. On peut donc déjà piloter la lecture sans regarder l'écran.

### En attendant les corrections

1. **Laisse vide le champ « Local Address »**, ou déconnecte-toi et reconnecte-toi sans lui.
2. **Mets ton serveur en `https://`.**
3. **Ne connecte l'app qu'à ton propre serveur**, un serveur de confiance.
4. **N'active pas l'option « lecture capot fermé »** tant que tu n'en as pas besoin.

---

## 1. Sécurité

Niveaux utilisés : **Critique** > **Élevé** > **Moyen** > **Faible** > **Info**.
« Confirmé » veut dire que j'ai relu le chemin du code moi-même.

### S1. Élevé (confirmé) : le mot de passe part vers n'importe quel appareil qui répond à l'adresse locale

- **Fichiers :**
  - `Sources/ABSCore/Server/ServerRouter.swift:71-76, 93-103`
  - `Sources/Audiobookshelf/App/AppModel.swift:116-130, 150-153, 241-245`
  - `Sources/ABSCore/API/APIClient.swift:164-169, 241-262`
- **Le mécanisme :**
  1. Pour choisir entre l'adresse publique et l'adresse locale, l'app demande seulement `/ping` et attend la réponse « 200 OK ».
  2. Si l'adresse locale répond, et elle répond presque toujours plus vite, **toute l'API** passe par elle : la connexion, le renouvellement du jeton, et la reconnexion automatique avec le mot de passe du Trousseau.
  3. L'adresse locale est en `http://` par défaut (`LoginView.swift:97`), donc tout passe en clair.
- **Scénario :**
  1. Tu as mis `http://192.168.1.10:13378` comme adresse locale.
  2. Au café, un appareil qui a cette même adresse répond « 200 » à `/ping`, puis « 401 » à tout le reste.
  3. L'app lui envoie alors le jeton d'accès, puis le jeton de renouvellement, puis **ton identifiant et ton mot de passe**.
  4. Il peut ensuite forcer ta déconnexion.
- **Correctif :**
  - Ne jamais envoyer le mot de passe ni le jeton de renouvellement à l'adresse locale. La connexion et le renouvellement doivent toujours passer par l'adresse publique en `https`.
  - Vérifier que l'appareil local est bien *ton* serveur avant de l'utiliser, par exemple en comparant un identifiant obtenu via l'adresse publique.
  - Les téléchargements envoient aussi le jeton à l'adresse locale (`DownloadManager.swift:267-274`) : même correctif.

### S2. Élevé (à confirmer sur Mac) : la déconnexion peut effacer un dossier choisi par le serveur

- **Fichiers :** `AppModel.swift:99`, `App/DiskCache.swift:19, 50-53`
- **Le mécanisme :** le dossier de cache s'appelle `"<hôte>-<identifiant utilisateur>"`, et l'identifiant vient du serveur sans aucun filtrage. Un identifiant du type `x/../../../../..` fait pointer le cache vers ton dossier personnel. Or la déconnexion fait un effacement récursif de ce dossier.
- **Le serveur peut forcer cette déconnexion :** il lui suffit de répondre « 401 » partout.
- **Conditions :** un serveur malveillant, ou quelqu'un qui intercepte une connexion en `http://`. Ça peut aussi passer par S1.
- **Correctif :**
  - Nommer le dossier avec une empreinte (SHA-256 de hôte + identifiant), jamais avec une valeur venant du serveur.
  - Avant d'effacer, vérifier que le chemin est bien à l'intérieur du dossier de cache.

### S3. Élevé contre un serveur malveillant, Faible sinon (à confirmer sur Mac) : les téléchargements peuvent écrire et effacer en dehors de leur dossier

- **Fichier :** `Downloads/DownloadManager.swift:153-156, 161, 170-171, 191, 376-382, 421`
- **Le mécanisme :** la fonction `clean()` remplace `/` et `:`, mais laisse passer les noms `.` et `..`.
  - Nom de fichier `..` : la destination devient le dossier parent, et l'app l'efface (`removeItem`) avant d'y déplacer le fichier.
  - Auteur `..` **et** nom de fichier `..` : l'app efface le dossier *au-dessus* du dossier de téléchargements. Avec le réglage par défaut, c'est `~/Library/Application Support/Audiobookshelf` : tes jetons, tes téléchargements et tes écoutes pas encore synchronisées. Si tu as choisi `~/Audiobooks` comme dossier de téléchargements, c'est tout ton dossier personnel.
- **Qui peut le déclencher :**
  - Un nom d'auteur `..` peut être saisi par n'importe quel utilisateur de ton serveur qui a le droit de modifier les métadonnées.
  - Un nom de fichier `..` demande un serveur piraté.
- **Correctif :**
  - Refuser `.`, `..`, les noms qui commencent par un point et les caractères de contrôle.
  - Vérifier que chaque chemin final reste dans le dossier de téléchargements.
  - Ne jamais effacer un *dossier* à l'endroit où un fichier doit arriver.
- **Pour comparer :** le serveur officiel a une fonction `sanitizeFilename` bien plus stricte.

### S4. Moyen : le `http` en clair est autorisé partout, sans aucun avertissement

- **Fichiers :** `packaging/Info.plist` (`NSAllowsArbitraryLoads = true`), `LoginView.swift:90, 97`
- **Ce qui passe en clair si l'adresse est en `http://` :** le jeton à chaque requête, le jeton de renouvellement chaque jour, le mot de passe à la connexion, et le jeton de la connexion WebSocket.
- **Correctif :**
  - Retirer `NSAllowsArbitraryLoads`.
  - Afficher un avertissement quand une adresse non locale est en `http`.

### S5. Moyen (confirmé) : des données serveur aberrantes font planter l'app, parfois à chaque lancement

En Swift, convertir en entier un nombre décimal énorme, infini ou « NaN » fait planter le programme. Où ça arrive :

- **`Models.swift:252`** : un identifiant de chapitre à `1e20` fait planter l'app.
- **`AppModel.swift:435, 445`** : une date `updatedAt` à `1e300` sur un livre ou un auteur.
  - Ces données sont **gardées en cache sur le disque**, donc l'app replante à chaque lancement, même hors ligne.
  - Pour s'en sortir, il faut vider `~/Library/Caches/com.arturgrochau.audiobookshelf-mac`.
- **`Format.swift`** : `timestamp`, `elapsedPretty` et `bytesPretty` plantent sur des valeurs énormes. `bytesPretty` plante aussi sur une taille négative, car le logarithme d'un nombre négatif donne « NaN ».
- **`DownloadManager.swift:188, 239, 397`** : des tailles de fichier géantes font déborder le compteur d'octets, qui plante.
- **`Models.swift:403-414`** (`LossyArray`, à confirmer sur Mac) : un élément de liste qui n'est pas un objet JSON pourrait provoquer une **boucle infinie** qui gèle l'interface. Ça peut venir d'un message poussé par le serveur, sans aucune action de ta part.
- **Correctif :**
  - Une seule fonction de conversion sûre, `safeInt`, utilisée partout.
  - Des additions qui ne débordent pas.
  - Un `Discard` qui réussit toujours à lire l'élément, pour sortir de la boucle.

### S6. Moyen (confirmé) : le lien `audiobookshelf://` peut être utilisé par n'importe quelle page web

- **Fichier :** `App/AppDelegate.swift:222-298`
- **Plantage :** `audiobookshelf://sleep?m=inf` plante l'app immédiatement (`Format.sleepBadge`, `Format.swift:93`).
- **Perte de progression :** `audiobookshelf://route?r=seek&t=0` remet la position du livre en cours à zéro, **et la synchronise avec le serveur**. L'erreur se propage donc sur le web et le téléphone.
  - `route?r=playitem&id=…` lance la lecture à voix haute.
  - `speed?v=10` change la vitesse de façon durable.
- **Une précision sur les téléchargements :** les commandes qui touchent aux fichiers (`download`, `remove-download`) sont bien désactivées par défaut. C'est une bonne chose.
- **Correctif :**
  - Réserver `route` (navigation, `playitem`, `seek`) aux versions de développement.
  - Limiter les valeurs acceptées : minuterie de 1 à 1440 minutes, nombres finis uniquement.

### S7. Moyen (à confirmer sur Mac) : des réponses authentifiées sont gardées en cache sur le disque, et jamais effacées

- **Fichier :** `APIClient.swift:102-108`
- **Le mécanisme :** la session réseau de l'API utilise le cache système par défaut (`URLCache.shared`). Les réponses JSON authentifiées peuvent donc être écrites dans `~/Library/Caches/com.arturgrochau.audiobookshelf-mac/Cache.db`. Ça inclut `/api/me`, qui contient l'ancien jeton du serveur, un jeton qui n'expire jamais.
- La déconnexion ne vide pas ce cache.
- **Correctif :** mettre `cfg.urlCache = nil` sur la session de l'API, et vider le cache à la déconnexion.

### S8. Faible : protections locales limitées

- **Jetons en clair :** les jetons sont stockés dans `account.json` (droits `0600`, documenté). Tout programme qui tourne sous ton compte peut les lire. Il peut aussi **modifier l'adresse du serveur** : à la reconnexion automatique suivante, l'app enverra ton mot de passe du Trousseau à cette nouvelle adresse (`Keychain.swift:82-111, 126`).
- **Compilation :**
  - pas de « hardened runtime » ni de sandbox ;
  - signature ad hoc avec `--deep`, une option dépréciée (`build.sh:29-36`).
- **Conséquence :** un programme local peut injecter du code dans l'app (`DYLD_INSERT_LIBRARIES`) et lire le mot de passe du Trousseau sans que macOS demande quoi que ce soit.
- **Correctif :**
  - Ajouter `codesign --options runtime` et retirer `--deep`.
  - Stocker les jetons dans le Trousseau.
  - Lier le mot de passe enregistré à l'adresse du serveur pour laquelle il a été saisi.

### S9. Faible : la règle `sudoers` de l'option « capot fermé »

- **Fichier :** `Player/KeepAwake.swift`
- **Ce qui est sûr :** j'ai reconstitué la commande exécutée en administrateur, caractère par caractère. Elle est correcte et ne permet aucune injection. La règle est validée par `visudo` et ne permet que `pmset -a disablesleep 0` et `pmset -a disablesleep 1`.
- **Les limites :**
  - N'importe quel programme sous ton compte peut empêcher la mise en veille capot fermé, avec un risque de surchauffe dans un sac.
  - Après une coupure de courant, la veille reste désactivée jusqu'à la prochaine ouverture de l'app, et ce réglage survit au redémarrage.
  - Un compte qui s'appellerait littéralement `ALL` donnerait cette autorisation à tous les comptes du Mac.
- **Correctif :**
  - Écrire la règle avec le numéro d'utilisateur (`#<uid>`) au lieu du nom.
  - Vérifier l'état de la veille à chaque lancement.
  - Pour l'enlever toi-même : `sudo rm /etc/sudoers.d/audiobookshelf-lid`

### S10. Faible : la déconnexion n'efface pas tout

- **Fichier :** `AppModel.swift:258-296`
- **Ce qui reste après la déconnexion :**
  - le cache d'images (jusqu'à 1 Go) ;
  - `URLCache` (voir S7) ;
  - `local-sessions.json`. Les écoutes hors ligne de l'ancien compte seront **envoyées au compte suivant**.
  - les réglages `lastServerURL` et `lastUsername`.
- La requête de déconnexion peut elle-même partir vers l'adresse locale.

### S11. Faible et Info : divers

- **Connexion de test dans la version normale :** les variables d'environnement `ABS_TEST_SERVER`, `ABS_TEST_USER` et `ABS_TEST_PASS` sont lues même dans l'app normale (`AppDelegate.swift:37-50`). Il faudrait les réserver aux versions de développement (`#if DEBUG`).
- **Identifiants non vérifiés :** les identifiants venant du serveur ou d'un lien sont mis tels quels dans les chemins d'URL. `?` et `#` sont bien encodés, mais `../` passe (`Endpoints.swift`, `APIClient.swift:114-127`). Il faudrait accepter seulement `^[A-Za-z0-9_-]+$`.
- **Redirections :** elles sont suivies sans contrôle. Il faudrait refuser celles qui changent de domaine, ou retirer l'en-tête `Authorization` dans ce cas.
- **Formats d'image :** les images du serveur sont décodées sans liste de formats autorisés (`ImagePipeline.swift`). Il vaudrait mieux accepter seulement JPEG, PNG, WebP et HEIC.
- **Textes affichés tels quels :**
  - le texte d'une erreur 401 est affiché sur l'écran de connexion (`LoginView.swift:108`) ;
  - les messages `admin_message` du serveur deviennent des toasts (`AppModel.swift:333`).
  - Un faux serveur peut s'en servir pour afficher un faux message d'hameçonnage.
- **Réglages inutilisés :** `controlPortEnabled` est activé par défaut, avec le port 9801 (`AppSettings.swift:70-73, 141`), mais aucun serveur local n'existe dans le code. Le commentaire « loopback control port » (`AppDelegate.swift:221`) est un reste. S'il est un jour implémenté, il faudra le désactiver par défaut et le protéger par un jeton.
- **CI :**
  - `actions/checkout@v5` est épinglé par tag : il vaudrait mieux un SHA de commit ;
  - ajouter `persist-credentials: false` et `timeout-minutes`.
- **`build.sh` :**
  - ajouter `set -u` : si `HOME` était vide, la commande `rm -rf "$HOME/Applications/…"` viserait `/Applications` ;
  - l'identité de signature par défaut, « Aeon Local », est celle de l'auteur.

### Ce qui a été vérifié et qui est sain

- **Aucune télémétrie et aucun hôte écrit en dur.** L'app ne contacte que le serveur que tu as saisi.
- **Aucune dépendance** (Swift Package Manager).
- **Validation TLS par défaut, jamais contournée.** Aucun délégué ne l'accepte à l'aveugle.
- **Jamais de jeton dans une URL.** La lecture passe par `/public/session/<id>`, et les couvertures sont chargées sans authentification.
- **Pas d'injection de Markdown ou de liens :** tout le texte venant du serveur est affiché tel quel. Aucune WebView.
- **URL ouvertes dans le navigateur :** elles viennent seulement de l'adresse que *tu* as saisie. Le serveur ne peut pas faire ouvrir `file://` ou un autre protocole.
- **La commande administrateur de `sudoers`** est correcte et ne permet pas d'injection (voir S9).
- **Aucun serveur local ou IPC caché :** pas de port en écoute, pas d'AppleScript, pas de XPC.
- **Historique git propre :**
  - aucun secret, jamais, même effacé plus tard ;
  - aucun fichier supprimé ou caché ;
  - polices et images sans données ajoutées à la fin (tables des polices vérifiées, fins de fichiers JPEG, PNG et GIF vérifiées) ;
  - `en-us.json`, `absicons.ttf` et `book_placeholder.jpg` sont identiques à ceux d'Audiobookshelf officiel.
- **Mot de passe :** champ sécurisé (`SecureField`), rangé dans le Trousseau. Le mot de passe n'apparaît jamais dans les logs.

### Pour ton fork (info)

- **Ton fork est une copie exacte de l'original.** Le nom interne de l'app (`com.arturgrochau…`), les liens du README et le contact de `SECURITY.md` pointent encore vers l'auteur original.
- **Licence GPL-3.0 :** si tu distribues une app compilée, tu dois fournir le code source et indiquer tes modifications.
- **Nom et logo :** l'app s'appelle « Audiobookshelf » et utilise le logo officiel. Attention à la confusion avec l'app officielle si tu la distribues.

---

## 2. Accessibilité VoiceOver

- **Point de départ :** 3 `accessibilityLabel` dans toute l'app, 14 `onTapGesture`, et aucune annonce, aucun titre de section (« heading »), aucune action personnalisée, aucun élément regroupé pour VoiceOver.
- **Critère de référence :** celui des « étiquettes d'accessibilité » de l'App Store (Apple, 2025). Pour y prétendre, *toutes les tâches courantes* doivent être faisables avec VoiceOver seul : se connecter, parcourir, ouvrir un livre, lire, se déplacer dans le livre, changer la vitesse, les chapitres, les signets, la minuterie, les réglages.

### Bloquant

**A1. On ouvre les livres avec des gestes de souris, pas avec de vrais boutons.**
- **Où :** couverture des livres (`Cards.swift:50`), séries (`:362`), auteurs (`:293`), collections (`ListPages.swift:40`), narrateurs (`:227`), liens auteur, série et genre (`ItemPage.swift:314`), dépliants Chapitres et Pistes (`:450`), lignes de chapitres (`:483`), lignes des fenêtres chapitres, signets, minuterie et file d'attente (`PlayerModals.swift:364`), titre du lecteur (`PlayerBar.swift:42, 48`).
- **Effet :**
  - Tab et l'accès clavier complet les sautent : **impossible d'ouvrir un livre au clavier**.
  - VoiceOver ne sait pas que ces éléments sont cliquables, et VO-Espace ne fonctionne pas toujours (à confirmer sur Mac).
- **Correctif :** `Button { … }.buttonStyle(.plain)`, avec `.isLink` pour les liens et `.isSelected` pour l'élément sélectionné.

**A2. La barre de progression est invisible.**
- **Où :** `PlayerBar.swift:276-354`. Elle est faite de formes et d'un geste de glisser.
- **Effet :** impossible d'aller à un endroit précis dans un livre de 10 heures.
- **Correctif :**
  - `.accessibilityRepresentation { Slider(…) }`, ou bien `.accessibilityElement()` avec `.accessibilityValue("12 minutes sur 8 heures")` et `.accessibilityAdjustableAction` (pas de 15 à 30 secondes).
  - Ajouter une commande de menu **« Aller à… »** (saisie hh:mm:ss) et une commande **« Dire la position »**.

**A3. Chapitres, signets, minuterie, file d'attente et réglages du lecteur sont dessinés *par-dessus* la fenêtre.**
- **Où :** `WebControls.swift:96-132`, `MainView.swift:40-46`
- **Effet :**
  - Le focus de VoiceOver ne va pas dans la fenêtre qui s'ouvre, et rien n'est annoncé.
  - Le fond reste accessible.
  - Avec A1 en plus, **impossible de choisir un chapitre ou un signet**.
- **Correctif :**
  - Le mieux : utiliser de vraies `.sheet`.
  - Sinon, au minimum :
    - `.accessibilityAddTraits(.isModal)` sur la fenêtre ;
    - `.accessibilityHidden(true)` sur le fond ;
    - `@AccessibilityFocusState` pour placer le focus sur le titre ;
    - `.accessibilityAction(.escape)` pour fermer avec Échap.

**A4. Échap ferme le lecteur, et les flèches sont prises en permanence.**
- **Où :** `AppDelegate.swift:147-202`
- **Choix du propriétaire, à garder :** **Espace = lecture/pause partout.** C'est sans gêne pour VoiceOver, qui active les boutons avec VO-Espace (Ctrl-Option-Espace), une touche que l'app ne touche pas.
- **Le seul coût :** avec l'accès clavier complet *sans* VoiceOver, la barre d'espace seule n'active plus le bouton qui a le focus. C'est un compromis accepté.
- **Ce qui reste à corriger :**
  - **Échap ferme le lecteur** (`:170-173`) et arrête la session. On appuie souvent sur Échap pour sortir de quelque chose, donc c'est facile à déclencher par erreur. → Supprimer ce comportement, ou demander confirmation.
  - **Les flèches ↑ et ↓ changent le volume sans rien dire,** même quand une liste ou un menu a le focus. → Les laisser passer quand un contrôle a le focus, et annoncer le volume (voir A15).
  - **Les touches S, A, X et Z** : elles pourraient gêner la sélection par lettre dans les listes. → À rendre désactivables dans les réglages (à confirmer à l'usage).

### Majeur

- **A5. Les toasts et les erreurs ne sont jamais lus.**
  - Exemples : « Server could not be reached », « Your session expired », minuterie terminée…
  - **Correctif :** une petite fonction `Speak.say()` basée sur `AccessibilityNotification.Announcement`, disponible depuis macOS 14 :
    - priorité `.high` pour les erreurs ;
    - ne pas faire disparaître le toast tant que VoiceOver est dessus.
- **A6. Chaque carte de livre est lue en 3 à 6 morceaux.**
  - Ces morceaux sont : une image sans nom, « #3 », le nom de l'icône « téléchargé », le titre, l'auteur. La progression n'est jamais lue, et Lire et « ⋯ » n'existent qu'au survol de la souris.
  - **Correctif :**
    - `.accessibilityElement(children: .ignore)`, avec un libellé du type « Titre, de Auteur, 45 % écouté, téléchargé » ;
    - le trait `.isButton` ;
    - des actions nommées : Lire, Marquer comme terminé, Télécharger, Ajouter à la file (VO-Cmd-Espace) ;
    - couvertures masquées pour VoiceOver.
  - Même chose pour les cartes de séries, d'auteurs et de collections.
- **A7. Aucun titre de section.**
  - Impossible de sauter d'une étagère ou d'une section à l'autre (rotor VO-U, ou la touche H en navigation rapide).
  - **Correctif :** `.accessibilityAddTraits(.isHeader)` ou `.accessibilityHeading(.h1/.h2)` sur les titres de page, les étagères de l'accueil, « Chapitres » et les titres des fenêtres.
- **A8. Barre du lecteur :**
  - **Ordre de lecture :** VoiceOver lit la vitesse, la minuterie et les options *avant* Lire. Correctif : `.accessibilitySortPriority(1)` sur les contrôles de lecture.
  - **Bouton vitesse :** il se lit « 1.5x » sans nom.
  - **Minuterie :** elle se lit « 12m » ou « EoC ».
  - **Menu « ⋯ » :** il n'a pas de nom.
  - **Bouton Lire pendant le chargement :** le chargement n'est pas annoncé.
- **A9. Formulaires :**
  - **Connexion :** les champs n'ont pas de nom (`TextField("")`), l'erreur n'est pas annoncée, et le logo n'a pas de nom.
  - **Réglages du lecteur :** `Toggle("")` et `Picker("")` sont lus comme « interrupteur, activé » sans dire lequel.
  - **Renommer un signet :** le champ n'a pas de nom.
  - **Correctif :** passer le vrai libellé, et garder `.labelsHidden()` si besoin.
- **A10. Changer de page ne donne aucun retour.**
  - La page est reconstruite et le curseur VoiceOver se perd. Le titre de la fenêtre reste toujours « Audiobookshelf ».
  - **Correctif :**
    - mettre à jour le titre de la fenêtre ;
    - annoncer le nom de la page ;
    - placer le focus VoiceOver sur le titre de la page.
- **A11. Fenêtre des chapitres :**
  - Les coches invisibles (`opacity(0)`) sont quand même lues, à confirmer sur Mac.
  - Le chapitre en cours n'est signalé que par la couleur.
  - **Correctif :** `.accessibilityHidden` sur les coches, `.isSelected` sur le chapitre en cours, et un libellé du type « Titre, commence à 1 heure 2 minutes ».
  - Idéalement, ajouter un **rotor « Chapitres »**.
- **A12. Boutons avec icône seule, sans nom :**
  - menu du compte, sélecteur de bibliothèque, bouton pour effacer la recherche, indicateur hors ligne, flèches des étagères, menus « ⋯ » ;
  - badges « E » et « A » (explicite, abrégé) ;
  - puces de filtre, menu de tri (le sens n'est indiqué que par une flèche) ;
  - bouton Lire du mini-lecteur, boutons de la minuterie « −30m » et « +30m ».
- **A13. Contrôles qui n'existent qu'au survol de la souris :**
  - Lire et « ⋯ » sur les cartes, le bouton de fermeture du mini-lecteur.
  - **Correctif :** des actions nommées pour VoiceOver, et le bouton de fermeture toujours présent (juste transparent quand la souris n'est pas dessus).
- **A14. La recherche :**
  - Les suggestions ne sont pas annoncées.
  - Elles se ferment quand le focus quitte le champ.
  - On ne peut pas descendre dedans avec la flèche bas.
- **A15. Les raccourcis ne disent rien :**
  - vitesse (S, A, X, Z), volume (↑ et ↓), coupure du son (M), minuterie réglée.
  - Il faut annoncer le résultat de chaque raccourci.
- **A16. Menu « Controls » :**
  - pas de coche sur la vitesse actuelle ;
  - pas d'état de la minuterie ;
  - les raccourcis sont écrits *dans* le titre du menu (« Jump Forward ⌥→ »), donc VoiceOver lit les symboles.
  - **Commandes qui manquent :** file d'attente, réglages, couper le son, volume, fermer le lecteur, ajouter un signet, aller au livre en cours, rechercher (⌘F), **Aller à…**, **Dire la position**.
- **A17. Tout est en anglais.**
  - `L` ne charge que `en-us.json`, beaucoup de textes sont écrits en dur, et les durées sont au format « hr/min ».
  - La voix française de VoiceOver lit donc de l'anglais avec un accent français.
  - **Correctif :**
    - ajouter `fr.json`, qui existe déjà dans Audiobookshelf officiel ;
    - déclarer `CFBundleLocalizations` ;
    - formater les durées lues avec `Duration.formatted(.units(width: .wide))`.
- **A18. Des états ne sont indiqués que par la couleur :**
  - page active, vitesse différente de 1x, préréglage de vitesse sélectionné, livre en cours dans la file, minuterie active, livre terminé ou en cours.

### Mineur

1. **Contrastes insuffisants** (calculés selon la norme WCAG) :
   - `gray500` (#6A7282) : 2,43:1 sur le fond #373838. Le minimum pour du texte est 4,5:1. Remplacer par `gray400` ou `gray300`.
   - Toasts en texte blanc : de 2,37 à 3,19:1.
   - Message d'erreur de connexion : 3,69:1.
   - Bordures des champs : 1,56:1.
2. Progression, détails et statistiques sont découpés en morceaux. Il faut les regrouper (`.combine`).
3. Les tableaux des narrateurs et des pistes sont lus cellule par cellule.
4. Les chargements sont silencieux (page blanche). Il faut un `ProgressView` avec un nom.
5. La progression des téléchargements n'est jamais affichée ni annoncée.
6. Les dates sont au format américain (MM/dd/yyyy).
7. Le mini-lecteur :
   - sa fenêtre n'a pas de nom ;
   - sa progression n'est pas accessible ;
   - VoiceOver l'atteint difficilement, à confirmer sur Mac.
   - Correctif : `canBecomeKey`, un titre de fenêtre et le sous-rôle `.floatingWindow`.
8. Des images décoratives ne sont pas masquées : silhouettes, icônes des statistiques, loupe, icônes des toasts.
9. « Jump Backward - 10 seconds » : le tiret risque d'être lu « tiret ». Mettre une virgule.
10. Signets : « Rename » et « Delete » ne disent pas de quel signet il s'agit.
11. Mouvements : les animations ne respectent pas le réglage « Réduire les animations » (`accessibilityReduceMotion`).

### Déjà bien fait

- `SymbolButton` donne toujours un nom au bouton, et le bouton Lire/Pause suit l'état de la lecture.
- **Les icônes ne sont pas lues comme du texte bizarre.** Elles sont affichées par code de caractère, pas par ligature, et les SF Symbols sont bien gérés par VoiceOver.
- **Il y a des menus natifs :**
  - `Menu` : bibliothèque, compte, tri ;
  - un vrai `popover` pour la vitesse ;
  - un `Form` natif dans la fenêtre Réglages.
- **Barre de menus :** ⌘1 à ⌘7 pour les pages, ⌘[ et ⌘] pour naviguer, et un menu Controls complet pour la lecture.
- **Menu du Dock**, et **touches média, AirPods et Centre de contrôle** (`MPRemoteCommandCenter`). On peut piloter la lecture sans passer par l'interface.
- **Raccourcis de vitesse :** ils suivent le caractère tapé, donc ils marchent sur un clavier AZERTY.
- **Écran de connexion :** le focus va sur le premier champ vide, et Entrée valide.
- **Notifications natives** (lues par VoiceOver), et un son quand la minuterie s'arrête.

### Plan de test VoiceOver, à faire sur ton Mac après les corrections

1. Parcourir toute la fenêtre avec VO-→. Chaque élément doit avoir un nom, sans silence ni nom d'icône.
2. Ouvrir le rotor avec VO-U : les titres de section doivent être présents, ainsi que le rotor Chapitres.
3. Dans la grille : entrer avec VO-Maj-↓, vérifier les libellés, puis les actions avec VO-Cmd-Espace.
4. Barre de progression : entrer avec VO-Maj-↓, régler avec VO-← et VO-→, et vérifier la valeur annoncée.
5. Chapitres, signets et minuterie :
   - le focus doit aller dedans ;
   - le fond doit être inaccessible ;
   - Échap doit fermer la fenêtre et ramener le focus au bouton d'origine.
6. Déclencher une erreur : elle doit être annoncée une seule fois.
7. Barre de menus (VO-M) : toutes les commandes doivent y être.
8. Recommencer avec **l'accès clavier complet** activé (Réglages Système › Accessibilité › Clavier) :
   - Tab, Espace et les flèches doivent tout atteindre ;
   - taper un espace dans la recherche doit fonctionner.
9. Recommencer avec **Contrôle vocal** (« Afficher les noms »), **Réduire les animations** et **Augmenter le contraste**.
10. Côté développement :
    - lancer l'audit de l'**Accessibility Inspector** d'Xcode ;
    - dans les tests, appeler `performAccessibilityAudit()` (disponible depuis macOS 14).

### Sources principales

- Apple, critères VoiceOver des étiquettes d'accessibilité : https://developer.apple.com/help/app-store-connect/manage-app-accessibility/voiceover-evaluation-criteria
- `accessibilityRepresentation` : https://developer.apple.com/documentation/swiftui/view/accessibilityrepresentation(representation:)
- `AccessibilityNotification.Announcement` : https://developer.apple.com/documentation/accessibility/accessibilitynotification/announcement
- `isModal` : https://developer.apple.com/documentation/swiftui/accessibilitytraits/ismodal
- `AccessibilityFocusState` : https://developer.apple.com/documentation/swiftui/accessibilityfocusstate
- `accessibilityRotor` : https://developer.apple.com/documentation/swiftui/view/accessibilityrotor(_:entries:)
- WWDC21 « SwiftUI Accessibility: Beyond the basics » : https://developer.apple.com/videos/play/wwdc2021/10119/
- WWDC23 « SwiftUI cookbook for focus » : https://developer.apple.com/videos/play/wwdc2023/10162/
- WWDC24 « Catch up on accessibility in SwiftUI » : https://developer.apple.com/videos/play/wwdc2024/10073/
- WWDC25 (Mac) : https://developer.apple.com/videos/play/wwdc2025/229/
- HIG, recommandations d'Apple sur l'accessibilité, le menu et le clavier : https://developer.apple.com/design/human-interface-guidelines/accessibility
- `performAccessibilityAudit` : https://developer.apple.com/documentation/xcuiautomation/xcuiapplication/performaccessibilityaudit(for:_:)

---

## 3. Comparaison avec Audiobookshelf officiel (v2.37.1)

Pour chaque différence, une décision :

- **Reprendre d'ABS** : l'app Mac doit s'aligner sur l'officiel.
- **Garder** : l'app Mac fait aussi bien ou mieux.
- **Ajouter** : il manque une fonction à l'app Mac.
- **Inutile** : pas nécessaire pour une app d'écoute.

Chemins utilisés : `Srv` = `server/`, `Web` = `client/` du dépôt officiel.

### 3.1 Échanges avec le serveur

| Sujet | Différence | Décision |
|---|---|---|
| **Adresse locale** (`ServerRouter.swift`, `APIClient.swift`) | Le serveur n'a aucun moyen de prouver son identité sans connexion : `/ping` et `/status` ne renvoient rien d'unique (`Srv/Server.js:368-388`). L'app se fie donc à n'importe quel appareil qui répond (voir S1). | **Reprendre d'ABS** (comme l'app web, qui n'a qu'une adresse) : connexion, renouvellement et déconnexion **toujours vers l'adresse publique en https**. En local, seulement le jeton d'accès (valable 1 h) et l'audio. |
| **`/status` jamais appelé avant la connexion** (`Endpoints.swift:7`) | L'app web vérifie les méthodes de connexion. Les jetons exigent le serveur 2.26 ou plus, et le délai de grâce du renouvellement le 2.35 ou plus (`Srv/migrations`). | **Reprendre d'ABS** : appeler `/status` et afficher un message clair pour un serveur trop ancien ou en OIDC seulement. |
| **Jetons** : accès 1 h, renouvellement 30 jours, rotation avec délai de grâce de 10 min (`Srv/auth/TokenManager.js:17-21, 210-269`) | Un seul renouvellement à la fois, sauvegardé avant utilisation, comme le serveur l'attend. Le commentaire `AppModel.swift:100` dit à tort qu'une réutilisation « détruit la session » : elle échoue seulement. | **Garder** (corriger le commentaire). |
| **Ancien jeton `user.token`**, qui n'expire jamais (`Srv/models/User.js:617`) | L'app le retire avant de garder quoi que ce soit (`AppModel.swift:214-221`). | **Garder.** C'est une faiblesse du serveur. |
| **Limite de connexions** : 40 requêtes par 10 min et par IP, partagée entre `/login` et `/auth/refresh` (`Srv/utils/rateLimiterFactory.js`) | Un téléchargement refusé (401) relance le renouvellement sans vérifier si le jeton a déjà changé (`DownloadManager.swift:320-336`). 3 téléchargements × 5 essais peuvent épuiser la limite. | **Reprendre d'ABS** : même vérification que `APIClient.swift:164`. |
| **Mot de passe gardé** pour la reconnexion silencieuse | Le serveur n'en a pas besoin : le jeton de renouvellement dure 30 jours et se prolonge à chaque usage. | **À rendre facultatif**, et ne jamais l'envoyer sur l'adresse locale. |
| **`POST /api/items/:id/play`** (`PlayerModel.swift:176-213`) | Mêmes champs que `Web/players/PlayerHandler.js:182-200`. | **Garder.** Détail : ajouter `audio/x-caf` aux formats acceptés. |
| **Sessions transcodées** | Le serveur redirige `/track/1` vers du HLS, et l'app suit la redirection. | **Reprendre d'ABS** : utiliser `contentUrl` quand `playMethod == 2`, comme l'app web (à vérifier : se déplacer dans le livre pendant un transcodage). |
| **Synchronisation** : 20 s la première fois, puis toutes les 10 s | Identique à l'app web. En plus, l'app Mac synchronise à la pause, et passe à 60 s en mode économie d'énergie. Le serveur ne vérifie rien (`Srv/objects/PlaybackSession.js:238`). | **Garder** : l'app Mac est meilleure, et l'app web perd jusqu'à 10 s à chaque pause. |
| **`user_session_closed`** (`PlayerModel.swift:745-750`) | Le serveur envoie cet événement à *chaque* fermeture de session, y compris celles que l'app demande elle-même (`Srv/managers/PlaybackSessionManager.js:429`). L'app croit alors à une session perdue et en rouvre une, ce qui peut relancer l'ancien livre. L'app web, elle, remet simplement le lecteur à zéro. | **Reprendre d'ABS** : ignorer les fermetures demandées par l'app elle-même (bug à confirmer sur Mac). |
| **Repli hors ligne** (`PlayerModel.swift:150`) | *N'importe quelle* erreur, y compris 401, 403 et 404, fait basculer vers une session locale (vérifié). Le serveur ne contrôle pas l'accès dans `/api/session/local-all`, donc un utilisateur privé d'accès peut continuer à écrire sa progression. | **Reprendre d'ABS** : basculer hors ligne seulement sur une vraie erreur réseau. |
| **File d'envoi hors ligne** (`LocalSessionStore.swift:32-49`) | Une session refusée par le serveur (livre ou bibliothèque introuvable) reste dans la file et est renvoyée à chaque fois. | **Corriger** : retirer ces sessions de la file. |
| **Sessions hors ligne** renvoyées avec des totaux qui grandissent | Le serveur *remplace* les valeurs pour un même identifiant, donc rien n'est compté deux fois. | **Garder.** |
| **Socket.IO** : les étapes de connexion (`0`, `40`, `auth`, `init`) | Conforme. La surveillance de 45 s correspond exactement à ping + délai du serveur. | **Garder.** |
| **Téléchargement : erreur 403** | Réessayée 5 fois (`DownloadManager.swift:420-429`). | **Reprendre d'ABS** : considérer 403 et 404 comme définitifs. |
| **Téléchargement : identifiant de fichier** | Déduit de la fin de `contentUrl`. Le serveur fournit directement `track.ino`, un nombre (`Srv/models/Book.js:293`). | **Reprendre d'ABS** : lire `ino` et vérifier que ce sont des chiffres (`^\d+$`). |
| **Noms de fichiers** | La fonction `clean()` de l'app Mac est bien plus faible que `sanitizeFilename` du serveur (`Srv/utils/fileUtils.js:360`). Le serveur refuse les noms faits uniquement de points, retire les caractères de contrôle et coupe en octets en gardant l'extension. Il ne nettoie **pas** les noms d'auteur (`Srv/controllers/AuthorController.js`). | **Reprendre d'ABS** : porter `sanitizeFilename` en Swift (voir S3). |
| **Identifiants** | Le vrai serveur n'utilise que des UUID v4 (`Srv/models/User.js:193`) et vérifie leur format (`Srv/utils/index.js:249`). | **Reprendre d'ABS** : vérifier le format UUID au décodage (voir S2). |
| **Identifiants de progression inventés** (`local-<id>`, UUID aléatoire : `PlayerModel.swift:578`, `ItemActions.swift:260`) | Réinitialiser ou masquer une progression avec un faux identifiant donne un 404 et un message d'erreur. | **Corriger** : récupérer d'abord la vraie progression. |
| **Couvertures sans connexion** | Le serveur les autorise (`Srv/Auth.js:21, 36-38`). | **Garder.** |
| **Schéma `audiobookshelf://`** | L'app iOS officielle utilise `audiobookshelf://oauth` pour sa connexion OIDC. Les deux entrent en conflit si l'app iOS tourne sur un Mac Apple Silicon. Le risque est faible, car l'app Mac ignore `oauth`. | **Garder**, ou changer de schéma. |

### 3.2 Application web officielle : fonctions et comportements

**Déjà identique, rien à changer :**
- chapitre précédent (règle des 3 s), chapitre suivant, file d'attente ;
- étagères de l'accueil, tris de la bibliothèque, tailles de couverture ;
- couleurs (copie exacte du thème web) ;
- `en-us.json`, identique octet pour octet.

| Sujet | Différence | Décision |
|---|---|---|
| **Espace, ←/→, ↑/↓, M, L, Maj+↑/↓** | Identiques au web. L'app Mac ajoute ⌥←/→, ⌘←/→, ⌘1-7, ⌘D et ⌘⇧M. | **Garder** (Espace partout : ton choix). |
| **Échap** | Le web aussi ferme le lecteur avec Échap. | **Ne pas reprendre d'ABS ici** : supprimer ce comportement (voir A4). |
| **S, A, X, Z** (vitesse) | N'existent pas dans le web. | **Garder, mais désactivables.** |
| **Sauts** (10, 15, 30, 60, 120, 300 s) et **vitesse** (0,5 à 10) | Identiques au web. Les préréglages de vitesse diffèrent : 1 / 1,2 / 1,5 / 1,75 / 2 dans le popover, mais 0,5 / 1 / 1,2 / 1,5 / 2 dans le menu et dans le web. | **Reprendre d'ABS** : une seule liste partout. |
| **Minuterie** | Mêmes préréglages. L'app Mac ajoute les fonctions de l'app mobile : décompte seulement pendant la lecture, fondu, retour en arrière, réarmement, son. | **Garder.** |
| **Minuterie automatique, son, retour en arrière auto** | Codés, mais **aucun réglage visible** (`AudiobookshelfApp.swift:109-123`). | **Ajouter** les réglages. |
| **Réglages enregistrés mais jamais utilisés** | `showMenuBarExtra`, `controlPort`, `hotkeys`, `pauseOnHeadphonesDisconnect`, `collapseSeries`, `seriesSortBy`… | **Implémenter ou supprimer.** |
| **Chapitre en cours** | Le web affiche « chapitre (3 sur 12) » et le pourcentage. L'app Mac n'affiche que le titre du chapitre. | **Reprendre d'ABS**, au moins dans la valeur lue par VoiceOver. |
| **Marquer comme terminé, réinitialiser la progression** | Le web **demande confirmation**. L'app Mac, non. | **Reprendre d'ABS** : c'est facile à déclencher par erreur avec VoiceOver. |
| **Filtres de bibliothèque** | Le web a un menu de filtres (Progression › Pas commencé / En cours / Terminé…). Dans l'app Mac, toute la logique existe (`LibraryQuery.swift`), mais aucun menu. | **Ajouter** : très utile, et le code est déjà là. |
| **Tri des séries et des auteurs, regrouper les séries** | Absents de l'app Mac. | **Ajouter** (priorité moyenne à basse). |
| **Ajouter à une liste de lecture, retirer une série de « Continuer la série »** | Les appels à l'API existent mais ne sont pas branchés. | **Ajouter.** |
| **Description** | Le web affiche le HTML nettoyé. L'app Mac retire toutes les balises. | **Garder le texte simple** (il se lit bien avec VoiceOver), mais ajouter des puces « • » pour les listes. |
| **Date au format américain, « left », « Resume »** | Le web utilise le format de date du serveur et des textes traduits. | **Reprendre d'ABS.** |
| **Langues** | Le web propose 33 langues, dont un **`fr.json` complet (1 167 clés sur 1 167, vérifié)**, choisies par utilisateur, et règle la langue de la page pour les lecteurs d'écran. L'app Mac est en anglais uniquement, avec environ 90 textes écrits en dur. | **Reprendre d'ABS** (priorité n°1 pour toi) : copier `fr.json`, choisir la langue selon le système, retomber sur l'anglais si une clé manque, et déclarer `CFBundleLocalizations`. |
| **Podcasts** | Non gérés, mais une bibliothèque de podcasts peut quand même être choisie. | **Inutile**, mais masquer ces bibliothèques ou afficher « non pris en charge ». |
| **Livres numériques, administration, envoi de fichiers, modification des métadonnées, Chromecast** | Absents de l'app Mac. | **Inutile** : le lien « Ouvrir dans le navigateur » suffit. |
| **Ce que seule l'app Mac a** : téléchargements hors ligne, rester éveillé, mini-lecteur, adresse locale, lien `audiobookshelf://`, touches média, menu du Dock | — | **Garder** (en corrigeant la sécurité : S1, S3, S6). |
| **Accessibilité du web** | Le web a 104 `aria-label`, de vraies fenêtres modales (`role="dialog"`, focus déplacé, Échap), et de vrais menus ARIA. Mais sa barre de progression est inaccessible, ses cartes n'ont pas de nom, et le ticket #2268 « Improve accessibility for screen readers » est toujours ouvert. | **Faire mieux qu'ABS.** Le web n'est pas un bon modèle d'accessibilité. |

### 3.3 Comparaison avec tes propres projets

**Projets lus, sans rien y modifier :**

- **Tes apps Mac et iOS :** DSM Access, tt-Accessible, NVDA Remote for Mac, BrailliantConnect et reaperaccessible-ios.
- **Tes apps Windows**, pour tes habitudes seulement : DownAccess, MarkdownAccess, Accessible Media Converter et Accessible Media Editor.

**Ce qui définit tes projets :**

- **L'accessibilité passe avant tout.** « L'accessibilité fait partie de la correction fonctionnelle, pas d'une passe finale » (`dsmaccess/CLAUDE.md:8-10`).
- **Les annonces passent par un seul outil central.**
- **Les contrôles sont natifs.**
- **Les vraies fenêtres modales** s'ouvrent avec `.sheet(item:)`.
- **Le focus est géré** avec `@AccessibilityFocusState`.
- **Les titres de section sont marqués comme tels :** 67 dans DSM Access.
- **Les durées sont formulées pour être lues à voix haute.**
- **Le français est toujours inclus**, et un test vérifie qu'aucune traduction ne manque.
- **La distribution est soignée :** Developer ID, hardened runtime, notarisation, sans `--deep`.
- **La sécurité réseau est stricte :** seulement `NSAllowsLocalNetworking`, et pour les certificats auto-signés, l'empreinte que tu as approuvée est gardée dans le Trousseau.

L'app Mac est loin de tout ça.

| # | Sujet | audiobookshelf-mac | Ta pratique (fichier de référence) | Décision |
|---|---|---|---|---|
| 1 | **Annonces** | Aucune | `VoiceOver.announce(_:category:priority:)` avec un délai de 100 ms et des regroupements (`dsmaccess/Support/VoiceOver.swift:12-114`). Voix de secours quand l'app est en arrière-plan (`nvdaremote-mac/App/AppModel.swift:791-798`). | **Reprendre ta pratique**, branchée dans `AppModel.toast()` |
| 2 | **Échec d'une action que tu as demandée** | Toast de 4 s | Une alerte qui reste affichée (`dsmaccess/CLAUDE.md:197-204`) | **Reprendre ta pratique** : échec de lecture, de connexion ou de téléchargement |
| 3 | **Événements en arrière-plan** (minuterie, fin de livre) | Toast | Un son au premier plan, une notification en arrière-plan (`dsmaccess/Support/OperationNotifier.swift:61-80`) | **Reprendre ta pratique** : une app d'écoute tourne souvent en arrière-plan |
| 4 | **Barre de progression** | Formes + glisser, invisible pour VoiceOver | `MediaPlaybackPositionControl` : rôle slider, ±5 s, Début, Fin et Page, `valueChanged` seulement quand le focus est dessus (`ttaccessible/.../AppKit/MediaPlaybackPositionControl.swift:12-268`). Ou le `Slider` de `ReplaysView.swift:351-356` (« 3 minutes 12 sur 58 minutes ») | **Reprendre ta pratique** : la version tt-Accessible est prête à réutiliser |
| 5 | **Éléments cliquables** | 14 `onTapGesture` | Des `Button` natifs | **Reprendre ta pratique** |
| 6 | **Cartes de livres** | 3 à 6 arrêts, actions au survol seulement | `Button`, `children: .ignore`, libellé composé, indice, actions nommées, menu contextuel (`reaperaccessible-ios/ReaperAccessible/Replays/ReplaysView.swift:124-169`) | **Reprendre ta pratique** |
| 7 | **Listes de livres** | Grille `LazyVGrid` | Tableaux `Table` ou `NSTableView` pour les données (`dsmaccess/CLAUDE.md:159-165`) | **À décider par toi** : garder la grille, et ajouter une vue en liste ou en tableau ? |
| 8 | **Fenêtres modales** | Dessinées par-dessus la fenêtre | `.sheet(item:)`, titre marqué comme titre, focus et annonce à l'ouverture, raccourcis Annuler et OK (`dsmaccess/Views/NameEntrySheet.swift:43-75`) | **Reprendre ta pratique** |
| 9 | **Échap** | Ferme le lecteur | « Un Échap égaré ne doit jamais fermer la fenêtre de session » (`ttaccessible/.../EscapeClosableWindow.swift:12-14`) | **Reprendre ta pratique** |
| 10 | **Flèches** | Toujours prises | Surveillance du clavier seulement si l'app est active et hors d'un champ texte, et les touches de liste restent aux listes (`ttaccessible/.../ChannelMixerKeyboardController.swift:56-100`) | **Reprendre ta pratique.** On garde Espace = lecture/pause. |
| 11 | **Titres de section** | 0 | Partout (`ttaccessible/AGENTS.md:364`) | **Reprendre ta pratique** |
| 12 | **Champs sans nom** | `TextField("")`, `Toggle("")`, `Picker("")` | Chaque contrôle a un nom | **Reprendre ta pratique** |
| 13 | **Barre de titre transparente et titre masqué** (`AppDelegate.swift:71-73`) | Oui | « Ça tue la navigation VoiceOver dans la barre d'outils » (`ttaccessible/AGENTS.md:324`) | **Reprendre ta pratique**, au moins donner un titre à la fenêtre |
| 14 | **Langue** | Anglais seulement, avec le mécanisme `L.s` | Français et anglais obligatoires, avec un test (`dsmaccess/dsmaccessTests/LocalizationCatalogTests.swift`). BrailliantConnect a déjà une table maison `L.t()` pour un paquet SwiftPM | **Garder `L.s`** (mêmes clés qu'ABS), **ajouter `fr.json`** et un test des clés manquantes |
| 15 | **Durées lues** | « 1:05:32 », « hr/min » en anglais | `SpokenTime` : « 1:05:32 lu par VoiceOver devient une heure de la journée » (`ReaperAccessible/Replays/SpokenTime.swift:7-49`) ; `Duration.formatted(.units(width: .wide))` | **Reprendre ta pratique** |
| 16 | **ATS** | `NSAllowsArbitraryLoads` | `NSAllowsLocalNetworking` seulement, et le `http` est un choix explicite de l'utilisateur (`dsmaccess/Networking/DSMEndpoint.swift:12-25`) | **Reprendre ta pratique** |
| 17 | **Certificats auto-signés** | Possibles seulement parce que l'ATS est grand ouvert | Empreinte approuvée gardée dans le Trousseau (`dsmaccess/Networking/ServerTrustDelegate.swift:15-119`) | **Reprendre ta pratique** |
| 18 | **Jetons** | Fichier JSON avec les droits 0600 | Trousseau (`dsmaccess/Session/CredentialStore.swift:42-67`) | **Reprendre ta pratique** dès que l'app sera signée avec un Developer ID |
| 19 | **Outil pour le Trousseau** | Distingue « verrouillé » de « absent » | Renvoie seulement `nil` | **Garder la version de l'app Mac**, qui est un peu meilleure |
| 20 | **Signature** | Ad hoc, `--deep`, sans hardened runtime | Developer ID « Mathieu Martin (633EG76YX5) », `--options runtime --timestamp`, notarisation, refus de `get-task-allow` (`ttaccessible/build.sh:77-93`, `brailliantconnect/tools/make-dist.sh:198-207`) | **Reprendre ta pratique** |
| 21 | **Sandbox** | Aucune | DSM Access et tt-Accessible sont en sandbox. NVDA Remote ne l'est pas, et la raison est écrite | **Écrire pourquoi il n'y en a pas** : `sudoers`/`pmset` pour le capot fermé l'empêchent |
| 22 | **Compilation** | SwiftPM, sans Xcode | Projet Xcode pour les apps Mac | **Garder pour l'instant.** Passer à Xcode, c'est à toi de décider : ça débloquerait `.xcstrings` et les audits d'accessibilité automatiques |
| 23 | **Tests** | Uniquement la logique de base (ABSCore) | Swift Testing, plus des tests d'interface avec `performAccessibilityAudit` | **Ajouter** au moins un test des traductions |

**Code prêt à reprendre de tes projets :**

- **Annonces :** `dsmaccess/Support/VoiceOver.swift`
- **Barre de progression :** `ttaccessible/.../AppKit/MediaPlaybackPositionControl.swift` et `AccessibleSlider.swift`
- **Durées lues :** `ReaperAccessible/Replays/SpokenTime.swift`
- **Cartes :** `ReplaysView.swift:124-169`, qui montre aussi l'ordre des actions (VoiceOver les liste à l'envers : `:61-94`)
- **Fenêtre modale :** `dsmaccess/Views/NameEntrySheet.swift`
- **Échap :** `EscapeClosableWindow.swift`
- **Clavier :** `ChannelMixerKeyboardController.swift`
- **Menu branché sur la vue active :** `dsmaccess/Support/AppCommands.swift`
- **Progression :** `dsmaccess/Views/DSMUpdateView.swift:98-101`
- **Test des traductions :** `LocalizationCatalogTests.swift`
- **Certificats :** `ServerTrustDelegate.swift`
- **Référence d'accessibilité à copier dans `docs/` :** `dsmaccess/docs/accessibility.md` (600 lignes)

### 3.4 Comparaison avec AudioBooth (ton fork)

**Ce qu'est AudioBooth :**

- un autre client Audiobookshelf, en Swift, pour iOS 17+ et watchOS, qui compile aussi pour Mac Catalyst ;
- son auteur est Jeremy Grenier : ce n'est **pas** ton code, donc pas tes conventions.

**AudioBooth ne règle pas les mêmes problèmes de sécurité** :

- lui non plus ne vérifie pas que l'appareil local est bien ton serveur ;
- il a lui aussi des plantages `Int(Double)` et un chemin de téléchargement non vérifié (`bookID`).

**Il est en avance** sur les modes de connexion, le rangement des données par serveur, certains points VoiceOver et le français.

**L'app Mac est meilleure** sur la synchronisation et la gestion des jetons.

| Sujet | AudioBooth | Décision pour l'app Mac |
|---|---|---|
| **Modes de connexion** | Appelle `/status`, puis propose mot de passe, **OIDC avec PKCE** ou **clé d'API** (`ServerViewModel.swift:382-437`, `OIDCAuthenticationManager.swift`). | **Reprendre d'AudioBooth**, sans reprendre son journal qui enregistre le code OIDC et les cookies. |
| **En-têtes personnalisés par serveur** (Cloudflare Access, Authelia) | Oui (`AuthenticationService.swift:231-246`). | **Reprendre d'AudioBooth.** |
| **Renouvellement du jeton** | Anticipé : 60 s avant l'expiration, avec des nouvelles tentatives de plus en plus espacées (`CredentialsActor.swift:14-85`). | **Reprendre d'AudioBooth**, et **garder** en plus la nouvelle tentative sur 401 de l'app Mac. |
| **Mot de passe enregistré** | Aucun. | **Garder l'app Mac** (reconnexion silencieuse), mais uniquement vers l'adresse publique en https. |
| **Déconnexion** | Locale seulement : le jeton reste valable sur le serveur. | **Garder l'app Mac**, qui révoque le jeton sur le serveur. |
| **Jeton dans l'URL** | `?token=` pour les livres numériques et les téléchargements de la montre. | **Garder l'app Mac** : jamais de jeton dans une URL. |
| **Lien profond** | `audiobooth://connection/…` importe un serveur et **bascule dessus sans demander**. | **Garder l'app Mac** : aucune connexion par lien. |
| **Adresse secondaire (réseau local)** | Facultative, choisie par l'utilisateur. Vérifiée *avec* le jeton (`/api/authorize`). Bascule seulement sur une erreur réseau. | **Reprendre l'idée** : adresse locale facultative, et bascule seulement sur une erreur réseau. **Ajouter** une vérification *sans* identifiants : comparer l'empreinte d'une couverture connue, qui est servie sans authentification, entre l'adresse publique et l'adresse locale. C'est une proposition, à tester. |
| **Rangement par serveur** | Un **UUID créé localement** par serveur (`Server.swift:90`). | **Reprendre d'AudioBooth** : ça corrige S2. |
| **Nom des fichiers audio** | `<index><extension>`, avec une extension choisie dans une liste autorisée (`DownloadManager.swift:597`). | **Reprendre d'AudioBooth** (corrige S3), en gardant les dossiers Auteur / Titre lisibles. |
| **Synchronisation** | Toutes les 20 s environ. Peut sauter la synchro à la pause. Marque un livre terminé quand il reste **60 s ou moins**. | **Garder l'app Mac**, fidèle à ABS. AudioBooth peut marquer un livre terminé trop tôt. |
| **Synchro en temps réel entre appareils** | Pas de socket. La position la plus avancée gagne. | **Garder l'app Mac.** |
| **Fermer la session après 10 min de pause** | Oui, avec de nouvelles tentatives. | **Reprendre d'AudioBooth.** |
| **Sessions hors ligne coupées par jour** | Oui. | **Reprendre d'AudioBooth**, parce que les statistiques sont comptées par jour. |
| **Signets en attente hors ligne** | Oui (`BookmarkSyncQueue.swift`). | **Reprendre d'AudioBooth.** |
| **Minuterie** | « Fin dans N chapitres », alarme à une heure donnée. | **Reprendre d'AudioBooth** : « fin dans N chapitres ». |
| **Raccourcis (App Intents)** : minuterie, saut, signet | Oui. | **Reprendre d'AudioBooth**, pour l'app Raccourcis de macOS. |
| **VoiceOver** | 38 `accessibilityLabel`, 24 `accessibilityHidden`, 10 éléments regroupés, 8 traits, 4 `accessibilityAdjustableAction`. Mais **sa barre de progression n'est pas accessible non plus**, et il n'a aucune annonce. | **Reprendre d'AudioBooth** : le modèle de réglage au clavier de `TickSlider.swift:58-65` et `PlayerPreferencesView.swift:203-212`, la liste des chapitres en boutons (`ChapterPickerSheet.swift`), le badge de la minuterie avec un libellé complet, et les durées en toutes lettres (`TimeInterval+Formatting.swift:12-17`). Pour tout le reste, **tes propres projets sont un meilleur modèle** (voir 3.3). |
| **Français** | 665 textes sur 665 (`Localizable.xcstrings`). | **Préférer le `fr.json` officiel d'ABS**, qui utilise les mêmes clés que l'app Mac. |
| **Journal des erreurs de décodage** | Enregistre où la lecture a échoué (le chemin du champ). | **Reprendre d'AudioBooth.** |

---

## 4. Ordre de correction conseillé

### Étape 1 : sécurité urgente (petits correctifs, gros effet)

1. **S1.** Connexion, renouvellement, reconnexion et déconnexion **uniquement vers l'adresse publique**. Adresse locale facultative, qui ne reçoit que le jeton d'accès et l'audio. Même chose pour les téléchargements.
2. **S2.** Nommer le dossier de cache avec un UUID local (comme AudioBooth), jamais avec l'identifiant envoyé par le serveur.
3. **S3.** Porter `sanitizeFilename` d'ABS. Nommer les pistes `<index>.<extension autorisée>`. Vérifier que chaque chemin reste dans le dossier. Ne jamais effacer un dossier à l'endroit où arrive un fichier.
4. **S5.** Une fonction `safeInt`, des additions qui ne débordent pas, et un `LossyArray` qui avance toujours.
5. **S6.** `route`, `seek` et `playitem` réservés au mode développement. Valeurs des liens bornées.
6. **Repli hors ligne** seulement sur une vraie erreur réseau. **403 et 404** définitifs pour les téléchargements.

### Étape 2 : accessibilité VoiceOver, les blocages

7. **Français :** copier `fr.json` d'ABS, choisir selon la langue du système, remplacer les ~90 textes écrits en dur, déclarer `CFBundleLocalizations`, et ajouter un test des clés.
8. **A1 et A6 :** de vrais `Button` partout. Des cartes lues en un seul élément, avec des actions nommées (le modèle de `ReplaysView.swift`).
9. **A2 :** la barre de progression accessible (le `MediaPlaybackPositionControl` de tt-Accessible). Commandes « Aller à… » et « Dire la position ».
10. **A3 :** chapitres, signets, minuterie, file d'attente et réglages dans de vraies `.sheet`, avec focus, titre et Échap (le modèle de `NameEntrySheet.swift`).
11. **A5 :** un outil d'annonce central (`VoiceOver.swift` de DSM Access), branché sur les toasts. Des alertes pour les échecs des actions que tu as demandées.
12. **A4 :** Échap ne ferme plus le lecteur. Les flèches ne sont plus prises quand une liste a le focus. **Espace reste lecture/pause partout.**

### Étape 3 : accessibilité, le confort

13. **A7** titres de section, **A8** barre du lecteur, **A9** champs de formulaire, **A10** changement de page, **A11** chapitres (avec un rotor), **A12** noms des icônes, **A15** raccourcis qui parlent, **A16** menu complet.
14. Durées en toutes lettres (`SpokenTime`, `Duration.formatted`). Contrastes (`gray500` → `gray400`, toasts plus foncés). Titre de fenêtre visible.
15. Confirmation avant « Marquer comme terminé » et « Réinitialiser la progression », comme ABS.

### Étape 4 : sécurité, le reste

16. Retirer `NSAllowsArbitraryLoads`. Prévenir en cas de `http`. Épingler les certificats auto-signés que tu as acceptés (`ServerTrustDelegate.swift`).
17. Signature Developer ID avec `--options runtime --timestamp`, sans `--deep`, et notarisation. Ensuite, jetons dans le Trousseau.
18. `urlCache = nil` sur l'API. Déconnexion complète (images, file d'envoi hors ligne). Connexion de test réservée à `#if DEBUG`. Règle `sudoers` écrite avec le numéro d'utilisateur.
19. `user_session_closed` : ignorer les fermetures demandées par l'app elle-même. File d'envoi hors ligne : retirer les sessions refusées.

### Étape 5 : fonctions (voir 3.2 et 3.4)

20. Menu de filtres, tri des séries, réglages de la minuterie automatique, « fin dans N chapitres », `/status` avec OIDC et clé d'API, en-têtes personnalisés, fermeture de session après 10 min de pause, raccourcis pour l'app Raccourcis de macOS.
21. **Masquer les bibliothèques de podcasts.** Supprimer ou implémenter les réglages morts.

### Pour ton fork (identité)

- **Changer le nom interne de l'app :** remplacer `com.arturgrochau.audiobookshelf-mac` par ton propre identifiant.
- **Mettre à jour les liens :** README, `SECURITY.md` et modèles de tickets pointent encore vers l'auteur original.
- **L'identité de signature :** `build.sh` utilise celle de l'auteur ; mets la tienne.
- **Licence GPL-3.0 :** respecter ses obligations si tu distribues l'app.

---

## Annexe : ce qui n'a pas pu être vérifié ici

Faute de Mac dans l'environnement d'audit, ces points sont à tester :

- les chemins avec `..` pour S2 et S3, et les protections TCC de macOS ;
- la boucle infinie de `LossyArray` ;
- la mise en cache de `URLCache` (S7) ;
- ce que VoiceOver lit exactement (noms d'icônes, « 12m », coches invisibles) ;
- le comportement réel de `.isModal` sur les fenêtres dessinées par-dessus ;
- la relance de l'ancien livre après `user_session_closed`.
