# Spécification technique — lecteur IPTV LG webOS TV 6+

**Version :** 1.2 — **Date :** 28 septembre 2026<br>
**Statut :** spécification de conception, à valider sur téléviseurs réels avant implémentation complète<br>
**Langue de l’interface :** français par défaut, architecture prête pour la localisation<br>
**Révision 1.2 :** corrige les points de revue suivants — persistance M3U et consentement (§8.2), cible de compilation du service (§2.3), indexation Xtream (§5.1), clés M3U (§5.3), sonde MIME (§6.1), curseurs versionnés et coût disque (§2.4, §7.2), filtrage des URL de flux (§2.5, §6.1), saut alphabétique et RTL (§4.3, §9.1), trous du lecteur (§4.6), critères statistiques de go/no-go (§11.2, §12), contrôle de distribution en phase 0 (§12), magasin de certificats du Node embarqué (§2.5, §10). Détail en annexe A.

> But : développer un lecteur IPTV webOS clair, utilisable entièrement à la télécommande et réactif sur un téléviseur LG de gamme moyenne. L’application lit uniquement les sources fournies par l’utilisateur. Elle ne fournit ni playlist, ni chaîne, ni compte, ni contenu audiovisuel.

## 0. Décisions recommandées

| Sujet | Décision de référence | Motif |
|---|---|---|
| Type d’application | **Application web empaquetée** (`.ipk`), avec toutes les ressources d’interface dans le paquet | Le premier écran doit s’afficher sans attendre un serveur d’application distant ; LG distingue les apps empaquetées des apps hébergées dont le démarrage dépend du serveur. [S09] |
| Framework UI | **Enact 3.4.9 + Sandstone 1.4.6** pour couvrir webOS TV 6.0 | La matrice LG associe précisément webOS 6.0 à cette version. Enact 4 a abandonné le support de la génération TV 2021, donc ne convient pas à la borne minimale demandée. [S02][S03] |
| Langage | **TypeScript**, compilé en JavaScript pour la TV | Types pour les réponses Xtream/M3U/EPG incohérentes, refactorings plus sûrs ; Enact CLI prend en charge TypeScript. [S04] |
| Navigation télécommande | **`@enact/spotlight`**, intégré à Enact/Sandstone | Gestion de focus 5-way, des conteneurs et du mode pointeur ; les composants Sandstone navigables sont déjà intégrés à Spotlight. [S05][S06] |
| Réseau fournisseur | **Service JavaScript webOS** (Node.js 8.12 sur webOS 6) pour réseau/parsing, avec index compact et paginé des catalogues M3U/Xtream dans son répertoire privé | Sa mémoire n’est pas durable et le service peut être arrêté après inactivité ; son stockage de fichiers est distinct de DB8. Le TLS et les certificats du Node embarqué doivent être validés séparément du navigateur et du lecteur. Le service est compilé pour **Node 8.12 / ES2017**, cible distincte de celle de l’interface (§2.3). [S11][S32][S33] |
| Lecture | Élément **HTML `<video>` natif** et pipeline média webOS, URL de flux fournie par le profil | HLS est pris en charge, mais live MPEG-TS progressif, MIME, redirects, codecs et protocoles doivent passer la matrice phase 0 avant d’être annoncés comme compatibles. [S14][S15][S35] |
| Persistance | **DB8** pour profils, favoris, préférences et reprise ; **index catalogue compact sur disque privé du service** ; aucune passphrase obligatoire en V1 | La menace locale est explicitée §8. Les index/caches sont reconstruisibles ; DB8 ne reçoit pas tout le catalogue. Une source **M3U** ne peut pas fonctionner sans conserver ses URL de lecture : le consentement y est un **prérequis affiché**, pas une option, et l’index est chiffré au repos (§8.2). [S17][S19][S29][S33] |

**Prototype et go/no-go obligatoires avant l’UI complète :** tester Enact/Spotlight, LS2 et le stockage sur une TV webOS 6 ; confronter le service et le `<video>` à au moins deux portails Xtream autorisés et deux playlists M3U autorisées, couvrant MPEG-TS progressif, HLS, URL sans extension, redirect inter-hôte, HTTP, HEVC, AC-3/E-AC-3 et 1080i. S’y ajoutent trois vérifications **éliminatoires** : la politique LG pour une application qui lit des playlists fournies par l’utilisateur (§12, phase 0), le magasin de certificats du Node embarqué confronté à une racine récente (§2.5) et l’exécution du service sur un Node 8.12 épinglé (§2.3). Critères initiaux : **30 tentatives indépendantes par combinaison** déclarée supportée avec **≥ 29 premières images** (ou 20/20, §11.2), première image p95 ≤ 15 s **et p50 de zapping ≤ 4 s** (à calibrer) sur réseau de labo stable, aucune fuite de session visible après zapping. Le Simulator webOS 6.0 ne remplace pas le test TV. [S14][S15][S24][S35]

---

## 1. Objectifs, périmètre et limites

### 1.1 Objectifs produit

1. Présenter quatre entrées principales sur l’accueil : **Live TV**, **VOD**, **Séries** et **Profil**.
2. Ajouter et modifier une source **Xtream-compatible** ou une **URL de playlist M3U** depuis Profil.
3. Parcourir rapidement de gros catalogues au D-pad et à la Magic Remote.
4. Afficher le programme courant et le programme suivant quand un EPG est disponible.
5. Lancer des flux compatibles avec le pipeline média du téléviseur, sans bloquer l’interface.
6. Expliquer les erreurs de manière compréhensible et proposer une action utile : réessayer, changer de source, revenir au catalogue ou corriger le profil.
7. Garder l’écran et la navigation utilisables en cas de perte de réseau, de catalogue vide ou d’indisponibilité EPG.

### 1.2 Inclus dans la V1

- Gestion locale d’un ou plusieurs profils fournisseur, un profil actif à la fois ; enregistrement des secrets selon le choix de persistance expliqué §8 — pour une source M3U, le consentement de persistance est requis avant tout import (§3.5).
- Live TV, catégories, chaînes, favoris et informations EPG courant/suivant.
- VOD, catégories, affiches/posters, recherche globale, saut alphabétique, fiche courte et lecture.
- Séries, catégories, recherche globale, saut alphabétique, fiche série, sélection de saison, liste d’épisodes et lecture.
- Masquage et réordonnancement local des catégories, sans prétention de contrôle parental.
- Lecture des formats réseau réellement validés en phase 0 sur les appareils ciblés ; aucune garantie universelle par extension.
- États chargement/vide/erreur, reprise manuelle et reconnexion contrôlée.
- Aucun espace publicitaire vide dans la V1 : le panneau EPG utilise toute sa colonne. Un emplacement rétractable pourra être activé ultérieurement uniquement quand une annonce existe.
- Mode pointeur Magic Remote en complément ; aucune fonction essentielle ne dépend du pointeur.

### 1.3 Hors périmètre / non garantis

- Fourniture, découverte, partage ou vente de playlists, identifiants ou chaînes.
- Contournement d’un DRM, d’un abonnement, d’une restriction régionale ou d’un contrôle d’accès.
- Enregistrement vidéo, timeshift/catch-up, contrôle parental, multi-écran, cast, téléchargement ou lecture hors ligne.
- Promesse de prise en charge de tout flux `.ts`, RTSP, RTMP, MPEG-DASH, de tout codec, ou de tous les tags HLS. Les limites webOS et du serveur source s’appliquent. [S14][S15]
- Publicités tierces avant consentement et intégration d’un SDK publicitaire dans cette spécification.

**Conformité :** l’utilisateur doit posséder ou être autorisé à utiliser les sources configurées. Les mentions légales, la politique de confidentialité et toute déclaration demandée par LG doivent être fournies avant distribution.

---

## 2. Recherche : choix du langage, framework et architecture

### 2.1 Contraintes concrètes du runtime

- webOS TV 6.x utilise **Chromium 79**. Les générations suivantes ont des moteurs plus récents, mais le build doit rester compatible avec Chromium 79 si webOS 6 est réellement supporté. Vérifier les propriétés CSS récentes sur ce moteur : ne pas dépendre de `gap` sur un conteneur flex ou de `aspect-ratio` sans fallback, et tester chaque API navigateur au lieu de supposer qu’une transpilation TypeScript suffit. [S01]
- Une application TV webOS est une application web HTML/CSS/JavaScript. Un framework webOS TV n’est pas une application React Native native. [S01]
- Le tableau de compatibilité LG indique **Enact 3.4.9 / Sandstone 1.4.6** pour webOS 6.0. Enact 4 ne prend plus en charge les téléviseurs 2021 ou antérieurs : un build V1 commun webOS 6+ doit donc rester sur la branche Enact 3 validée, ou maintenir des builds distincts après essais. [S02][S03]
- webOS prend en charge deux modes de contrôle Magic Remote : pointeur et navigation 5-way. LG impose de concevoir aussi pour le 5-way ; les flèches et OK doivent suffire à accomplir tout le parcours. [S06]
- La lecture vidéo dépend de l’appareil, du manifeste, du codec, du profil, du débit et du serveur. L’émulateur/simulateur ne reproduit pas fidèlement toutes les capacités média du téléviseur. [S14][S15][S24]
- Le service dépend d’un **Node.js 8.12 embarqué** (version documentée par LG pour webOS 6), pas du Node courant : `fs.promises` n’existe qu’à partir de **Node 10.1**, l’itération asynchrone (`for await`) et `stream.pipeline` à partir de **Node 10.0**, Brotli (`zlib.createBrotliDecompress`) à partir de **Node 10.16**, `worker_threads` n’existe pas. Le service doit tenir dans le sous-ensemble ES2017/Node 8.12 et chaque API douteuse est bloquée par le lint de compatibilité (§2.3). [S36]
- Ce même Node embarque **OpenSSL 1.0.2p** : pas de TLS 1.3, pas de traitement des certificats signés en RSASSA-PSS, et un **magasin de racines figé à la date de build**. Chromium et le pipeline média ont leur propre magasin, mis à jour par firmware ; les trois piles se testent séparément (§2.5, §10). [S28][S36]

### 2.2 Comparaison des options

| Option | Avantages | Limites pour ce produit | Décision |
|---|---|---|---|
| **Enact 3 + Sandstone + Spotlight** | Écosystème LG, composants 10-foot, focus 5-way et pointeur intégrés, virtualisation de listes, TypeScript, outillage Enact. [S02][S04][S05][S08] | Framework React et rendu DOM : impose de virtualiser et de limiter les mises à jour/effets. Les composants et les versions doivent rester compatibles avec webOS 6. | **Choix V1** : meilleur équilibre entre compatibilité officielle, télécommande et maintenabilité pour l’interface demandée. |
| React + **Norigin Spatial Navigation** | Bonne bibliothèque de navigation spatiale pour React, annoncée pour webOS et d’autres TV web. [S26] | Remplace/duplique une partie de la navigation déjà fournie par Enact/Sandstone ; risque de deux gestionnaires de focus concurrents. | Ne pas ajouter si Spotlight est utilisé. Alternative seulement si l’UI abandonne Enact. |
| **Lightning 3 / Blits** | Framework conçu pour les interfaces TV ; rendu orienté performance et animations sur appareils contraints. [S27] | Modèle de rendu et composants différents, intégration du focus, du pointeur et du pipeline vidéo à prouver sur une TV webOS 6 réelle. Le gain n’est pas garanti sans benchmark sur le matériel visé. | Alternative à prototyper si les mesures Enact échouent, pas une seconde couche à mélanger à la V1. |
| React/DOM sans bibliothèque de focus | Liberté et dépendances potentiellement réduites. | Il faudrait réimplémenter le focus spatial, les frontières, le retour de focus et le scroll par télécommande ; risque élevé de régressions. | Rejeté pour ce besoin où la télécommande est centrale. |

### 2.3 Langage et compilation

- Écrire l’interface et les adaptateurs de données en **TypeScript** ; le code exécuté reste du JavaScript compilé. Activer `strict` dans `tsconfig` et garder les réponses réseau au type `unknown` jusqu’à leur validation runtime : les types TypeScript ne valident pas les données du fournisseur.
- Utiliser la version Enact CLI compatible avec Enact 3.4.9 ; figer toutes les dépendances dans un lockfile et contrôler leurs versions dans CI.
- Cibler explicitement **Chrome 79** dans Browserslist et compiler la syntaxe vers `target: ES2018` (ou plus ancien) : le compilateur doit transformer `?.`/`??` en syntaxe antérieure. Bloquer en CI tout bundle qui contient une syntaxe post-ES2018 non transformée (parse AST en `ecmaVersion: 2019` ou analyse équivalente).
- La cible syntaxique ne polyfill pas les APIs : interdire/remplacer `String.prototype.replaceAll` et toute méthode absente de Chromium 79, ou ajouter un polyfill ciblé. Activer un lint de compatibilité basé sur Browserslist puis exécuter le bundle de production sur Chromium 79 réel/épinglé et une TV.
- Produire un paquet de production statique (bundles locaux, CSS et assets locaux) avec le build production Enact, qui minifie et regroupe le code de l’application. [S10] Pas de `import()`/chargement de chunks à l’exécution dans la V1 empaquetée ; pas de dépendance à un serveur web distant pour dessiner l’interface.
- Éviter les syntaxes et APIs plus récentes sans transpilation/polyfill vérifié sur webOS 6. Ne pas confondre transpilation TypeScript et polyfill des APIs navigateur.

**Deux cibles de compilation distinctes.** L’interface et le service ne s’exécutent pas sur le même runtime ; une cible unique est une source de bugs silencieux (le code compile, puis échoue sur la TV) :

| Cible | Runtime de référence | Cible TS / syntaxe | Contrôles |
|---|---|---|---|
| `ui/`, `application/`, `providers/` côté application | Chromium 79 (webOS 6) | `target: ES2018`, Browserslist `chrome 79`, parse AST `ecmaVersion: 2019` | syntaxe post-ES2018 non transformée, `String.prototype.replaceAll`, APIs absentes de Chromium 79 |
| `services/provider-service/` | **Node.js 8.12.0** (webOS 6) | `target: ES2017`, `lib: ES2017` | `for await`, `fs.promises`/`fs/promises`, Brotli, `worker_threads`, `stream.pipeline`, `Object.fromEntries`, `String.prototype.matchAll`, `Array.prototype.flat`/`flatMap`, `String.prototype.trimStart`, groupes de capture nommés, BigInt, `globalThis`, `import()` dynamique |

- Le service utilise les callbacks `fs` (ou une promisification locale ; `util.promisify` existe en Node 8.0) et lit les flux avec `on('data')`/`read()`, jamais `for await`.
- Deux configurations `tsconfig`, deux lints de compatibilité (`eslint-plugin-es` ou `eslint-plugin-n` avec `engines: node >=8.12.0` pour le service, Browserslist pour l’interface) et un contrôle CI par parse AST de chaque bundle livré.
- **CI :** exécuter les tests du service sur un **Node 8.12.0 épinglé** en plus du Node de développement, comme Chromium 79 épinglé pour l’interface. Cela garantit le sous-ensemble de langage et les API, **pas** le build LG lui-même : relever `process.versions` sur l’appareil en phase 0 et consigner tout écart.
- Node 8.12.0 est EOL et sans correctifs de sécurité. Risque accepté pour un service local **sans écoute réseau entrante**, à condition que les données fournisseur restent traitées comme non fiables (bornes de taille, parseurs sans `eval`, aucune commande shell) et que la limite soit divulguée dans la page confidentialité. Toute API Node introduite plus tard exige une nouvelle mesure sur l’appareil.
- Compressions : annoncer `Accept-Encoding: gzip, deflate, identity` (jamais `br`) ; si un fournisseur répond malgré tout en Brotli, échouer proprement avec un message explicite plutôt que de tenter une décompression indisponible.

### 2.4 Architecture retenue

Architecture en couches et adaptateurs (ports/adapters), sans serveur cloud obligatoire :

```mermaid
flowchart LR
  R[Magic Remote / télécommande classique] --> K[Adaptateur d'entrées + Spotlight]
  K --> UI[Écrans Enact / Sandstone]
  UI --> UC[Cas d'usage + état des écrans]
  UC --> REP[Repositories de domaine]
  REP --> LS2[Adaptateur LS2 / service webOS]
  LS2 --> JS[Service webOS Node.js]
  JS --> NET[HTTP/HTTPS fournisseur]
  JS --> CAT[Index M3U/Xtream compact / fichiers privés du service]
  REP --> DB[DB8 / profils, favoris, reprises]
  UI --> PC[PlaybackController]
  PC --> VID[Un élément HTML video réutilisé]
  VID --> MEDIA[Pipeline média natif webOS]
```

**Responsabilités :**

- `ui/` : écrans et composants Enact/Sandstone ; aucun appel HTTP fournisseur directement dans un composant visuel.
- `domain/` : profils, chaînes, catégories, films, séries, épisodes, programmes EPG, favoris, erreurs normalisées.
- `application/` : cas d’usage (`testerProfil`, `chargerCategories`, `chargerElementsCategorie`, `lireChaine`, `reprendreVOD`, etc.).
- `providers/xtream/` : appels du dialecte Xtream et normalisation des réponses variables.
- `providers/m3u/` : téléchargement et parsing de playlist ; regroupement des entrées et extraction des métadonnées courantes.
- `epg/` : adaptation XMLTV/Xtream, correspondance chaîne-programmes et sélection courant/suivant.
- `platform/webos/` : LS2, cycle de vie, DB8, état réseau, saisie/clavier virtuel, clés système, capacités média.
- `services/provider-service/` : service JS webOS compact pour les requêtes fournisseur et traitements hors UI ; Node.js 8.12.0 est la version documentée pour webOS 6. [S11]
- `ServiceCatalogStore` : index compact M3U et index de recherche pour catalogues Xtream en fichiers du service, écrit/relu par pages ; quand l’API Xtream ne pagine pas, ingérer son tableau JSON par flux/parseur incrémental vers le même index. Chaque index porte un **`indexVersion`** monotone ; toute page, recherche ou tranche alphabétique renvoie cette version et tout curseur est un couple `(indexVersion, offset/ancrage)` exploitable uniquement sur cette version — jamais un « dernier offset » implicite. Le processus peut disparaître entre deux appels et n’est jamais la source de durabilité. DB8 ne reçoit que profils, préférences, favoris, correspondances et reprises.
- `player/` : interface `PlayerAdapter` et implémentation `NativeHtmlVideoPlayer`. L’UI du lecteur ne connaît pas les détails du fournisseur.

Le service JS webOS ne doit pas être un serveur de longue durée ni un proxy ouvert. LG documente son lancement à la demande et un délai d’inactivité de 5 s sans activité LS2 ; un service actif ne doit pas être supposé persister en mémoire après une réponse. Il s’exécute dans un environnement isolé ; ses fichiers peuvent être conservés sous `/media/internal` et réutilisés avec le même UID. [S11][S32][S33]

- `services.json` n’expose que des commandes étroites (`importPlaylist`, `getPage`, `getBuckets`, `cancelOperation`, etc.) ; toutes renvoient l’`indexVersion` utilisée. Conserver `public:false` (valeur par défaut) et ne jamais publier un proxy générique. Vérifier sur TV les règles effectives d’appelant/LS2 et n’accepter que l’ID de profil et les opérations attendues. [S34]
- Un import garde une requête/abonnement LS2 active uniquement tant que l’utilisateur le suit, publie la progression et accepte une annulation ; ne pas installer un keep-alive permanent ni laisser le service travailler en arrière-plan. LG déconseille les services qui restent actifs plusieurs minutes. [S32][S33]
- Favoriser chunks HTTP Range/ETag si le serveur les supporte et checkpoint d’index temporaire ; sinon un seul flux asynchrone peut rester actif pendant l’import, mais son temps/RSS doivent passer la qualification phase 0. Si la TV termine le service ou si la durée sûre est dépassée, supprimer la transaction temporaire, garder l’ancien index et proposer de reprendre/recommencer. Le process peut rouvrir l’index validé au prochain appel.
- **Bascule atomique, rétention et budget disque.** L’index précédent reste lisible après la bascule tant qu’un lecteur (requête LS2 en vol) le référence, avec une rétention bornée (valeur de départ 30 s) ; ensuite, les nouveaux lecteurs visent la version courante ou reçoivent `catalog/indexChanged` (§7.2). Pendant un import, le disque porte donc **l’ancien index + le nouvel index + les fichiers temporaires**, soit environ deux fois la taille de l’index plus l’entrée : le quota de sécurité se calcule sur `2 × index + marge`, pas sur l’index seul, et le test de capacité de 256 Mio mesure ce pic réel. Séquence : `catalog-<version>.tmp` → validation (décompte, empreinte) → `fsync` → `rename` → écriture atomique du manifeste. Au démarrage du service, supprimer les `.tmp` orphelins et toute version non élue par le manifeste.
- Le service parse M3U/XMLTV en flux et par lots sur son event loop, **sans `for await` ni `fs.promises`** (§2.3) ; pas de `worker_threads` requis ni supposé sur Node 8.12. Toute étape CPU longue cède la main régulièrement pour traiter annulation, progression et appels LS2.
- Les méthodes LS2 imposent pagination/chunks bornés ; le service limite tailles reçues et répond, ferme les requêtes annulées et n’écrit jamais d’identifiants ou d’URLs authentifiées dans ses logs.

### 2.5 Paquet empaqueté, CORS et accès fournisseur

**Problème à traiter dès le prototype :** webOS applique la politique CORS du navigateur. Les portails Xtream et les URLs M3U ne configurent pas tous les en-têtes CORS nécessaires. LG décrit CORS comme une configuration côté serveur ; le forum développeur webOS précise qu’une app empaquetée n’a pas d’origine HTTP classique, ce qui peut rendre les requêtes API navigateur incompatibles avec certains fournisseurs. [S12][S13]

**Décision :** les appels API, le téléchargement M3U et l’EPG passent par le service JS webOS et ses requêtes réseau ; l’interface reçoit des objets JSON normalisés, paginés/chunkés. Le lecteur vidéo reçoit directement l’URL de média et ne lit pas les segments avec du JavaScript. Ne jamais désactiver CORS, ni faire passer les identifiants via un proxy cloud invisible.

**TLS par pile :** le tableau LG confirme TLS 1.2/1.3 au niveau webOS, mais cela ne prouve pas que le client HTTPS du service Node 8.12 négocie TLS 1.3 ou utilise le même magasin de certificats que Chromium/le pipeline média. Le tag Node.js 8.12.0 upstream embarque OpenSSL 1.0.2p, qui ne fournit pas TLS 1.3 ; le build LG peut différer et n’est pas documenté à ce niveau. Lire uniquement la version non sensible `process.versions.openssl` sur l’appareil, puis tester la même API sur TLS 1.2, TLS 1.3-only et certificats publics représentatifs. Exiger TLS 1.2 comme baseline du service ; annoncer TLS 1.3 que si le prototype appareil le démontre. Aucun contournement TLS ni ajout silencieux de CA privée. [S28][S36]

**Magasin de racines du service (risque de fraîcheur) :** le bundle de CA du Node 8.12 embarqué est figé à la date de build du firmware : une chaîne qui aboutit à une racine publiée plus tard ne valide pas, alors que Chromium ou le média peuvent réussir (magasin distinct, mis à jour par firmware). Cas typique : chaînes ECDSA terminant sur **ISRG Root X2 (2020)**, ou toute racine apparue après 2018. La phase 0 doit donc inclure un hôte dont la chaîne **échoue** avec un bundle de 2018, en plus des cas TLS 1.2, TLS 1.3-only, certificat absent de l’appareil et horloge TV fausse, et consigner le résultat séparément par pile. Remède retenu pour la V1 : **embarquer un bundle de racines publiques à jour** (par ex. celui de Mozilla, ~200 Ko, licence et date tracées) et le passer au client HTTPS du service via l’option `ca`, vérification du nom d’hôte conservée. Ce n’est pas un contournement : aucune CA privée, aucun `rejectUnauthorized: false`, aucun élargissement silencieux de la confiance ; le bundle est mis à jour avec l’application, et le pipeline média — sur lequel l’application n’a pas de prise — reste un état d’erreur distinct (§7.2). [S28][S36][S39]

**Protection contre URL malveillantes/redirects :** toutes les URL M3U/EPG/stream sont non fiables. Pour les appels HTTP exécutés par le service, limiter les redirects à 5, revalider schéma/port/hôte à chaque saut, résoudre chaque nom et épingler l’IP publique validée via le callback DNS du client HTTP ; bloquer loopback, link-local, multicast et plages privées par défaut. Pour un serveur LAN, demander une autorisation explicite par profil et ne pas autoriser les redirects hors de cette plage. Ne jamais transférer `Authorization`/cookies à un autre origin. **Pour les URL de flux remises au `<video>`**, cette politique ne peut pas être appliquée par le service (aucun hook) : elle est approximée par le marquage d’hôte à l’indexation (§5.2) puis le contrôle avant affectation de la source (§6.1), avec les limites de cette approximation.

**Limite du `<video>` natif :** le media pipeline résout DNS et suit ses redirects lui-même ; l’app n’a pas de hook documenté pour pinner l’IP au socket. Avant de lui donner une URL, appliquer un contrôle best-effort de schéma/host et tester les redirects connus ; signaler explicitement que l’app ne peut ni pinner l’IP ni garantir l’inspection de chaque redirect du chemin média sans proxy. N’accepter que les URLs du profil actif/import choisi, jamais une URL issue d’un champ HTML arbitraire. Si une menace SSRF forte est exigée, réévaluer un relais local opt-in plutôt que prétendre au pinning complet.

Si le service local ne fonctionne pas avec un fournisseur ou est bloqué par une exigence de distribution, solutions de repli explicitement opt-in :

1. demander au fournisseur une API/configuration autorisant l’accès ;
2. proposer un pont local que l’utilisateur héberge et configure lui-même ;
3. éventuellement proposer un service relais documenté, avec consentement clair, politique de confidentialité, chiffrement et divulgation des données transmises.

Un relais distant n’est **pas** le comportement par défaut.

---

## 3. Parcours et exigences fonctionnelles

### 3.1 Accueil — quatre choix primaires

L’accueil comprend quatre grandes cartes/boutons, sans menu d’icônes ambigu :

1. **Live TV**
2. **VOD**
3. **Séries**
4. **Profil**

Chaque carte a un intitulé lisible, une icône simple, un état de focus très visible et un aperçu visuel discret. Le profil actif peut être indiqué dans l’en-tête. Si aucun profil n’est configuré, les sections de contenu expliquent qu’une source est nécessaire et le focus peut aller directement vers **Profil**, sans supprimer les quatre entrées.

### 3.2 Live TV

Disposition cible en 16:9 :

```text
┌────────────────┬──────────────────────┬───────────────────────────────┐
│ Catégories     │ Chaînes               │ EPG de la chaîne sélectionnée │
│ panneau gauche │ panneau central       │ panneau droit pleine hauteur  │
│                │                       │                               │
└────────────────┴──────────────────────┴───────────────────────────────┘
```

- Le panneau catégories et le panneau chaînes sont adjacents, séparés par une bordure ou un faible espace ; ils ne doivent pas paraître comme deux pages distinctes.
- Le panneau de droite donne la priorité à l’EPG et l’affiche sur toute la hauteur en V1 ; aucun bloc vide ou intitulé « publicité » n’est affiché.
- La sélection d’une chaîne met à jour son nom, logo disponible, programme courant, horaires, progression, description courte et programme suivant. **La sélection seule ne lance pas le flux** ; OK ouvre le lecteur.
- Le panneau EPG fonctionne même si le flux n’est pas lancé. Un EPG absent ou non apparié donne un état « Guide non disponible » sans bloquer les chaînes.
- Une régie/ad slot futur est rétracté par défaut et ne se déplie qu’avec une annonce effectivement disponible, consentie et conforme ; il ne prend jamais le focus ni ne retarde la lecture.
- États à concevoir : aucune catégorie, aucune chaîne, chargement, favoris vides, EPG absent, réseau indisponible, erreur fournisseur, source non compatible.
- Actions V1 : ajouter/retirer des favoris, rafraîchir la source depuis Profil et rechercher une chaîne dans toutes les catégories visibles avec le clavier virtuel ; la saisie vocale fournie par le clavier reste optionnelle et ne remplace pas le saut alphabétique.

### 3.3 VOD

- Catégories à gauche ; à droite, la plus grande zone d’écran est une grille virtualisée de films.
- En tête de la grille : recherche globale sur tous les films des catégories visibles du profil, saisissable au clavier virtuel, plus saut alphabétique dont les **tranches viennent des scripts réellement présents dans les données indexées** (§9.1) — pas de la locale de l’interface : un catalogue arabe avec interface française affiche les tranches arabes. Utilisable au D-pad ; résultat limité et paginé par le service.
- Debounce de recherche de 250 ms, annulation des requêtes obsolètes, accents/casse normalisés ; OK sur un résultat ouvre sa fiche et Back restaure query, lettre, catégorie, scroll et focus.
- Poster de taille modérée, jamais une affiche plein écran dans la grille. Afficher le titre et, si disponibles, année/durée/notation ; les données manquantes ne doivent pas créer de trous visuels.
- La grille doit rester parcourable si les posters sont absents ou lents : placeholder immédiat, chargement différé, image de repli en cas d’échec.
- OK sur un poster ouvre une fiche (titre, affiche, résumé, durée, année, bouton Lire, favori). OK sur **Lire** ouvre le lecteur.
- Après retour du lecteur, restaurer catégorie, index sélectionné, scroll et focus.

### 3.4 Séries

- Catégories à gauche ; grille de séries virtualisée dans le panneau principal.
- Recherche globale sur toutes les séries des catégories visibles et saut alphabétique sur les scripts présents dans les données (§9.1), avec même comportement D-pad, clavier virtuel, pagination, debounce et restauration d’état que la VOD.
- **Portée explicite de la recherche :** elle porte sur les **titres de séries**, pas sur les épisodes. Xtream n’expose les épisodes que par le détail d’une série (`get_series_info&series_id`) ; l’index global ne les couvre donc pas et l’écran de recherche doit le dire (« les épisodes se trouvent dans la fiche de la série »). Aucun compteur « N résultats » ne doit suggérer une recherche exhaustive sur les épisodes.
- OK sur une série ouvre une fiche de série (nom, visuel, synopsis si disponible) et le choix de saison.
- Après choix de saison, afficher les épisodes ; chaque ligne/card indique numéro, titre, durée ou résumé si présents.
- OK sur un épisode ouvre le lecteur. À la fermeture, restaurer la série, la saison, l’épisode sélectionné et la position de scroll.
- Xtream-compatible fournit normalement des objets de séries/saisons/épisodes, mais les réponses et noms de champs varient. L’adaptateur doit vérifier les données et afficher les champs présents.
- Une playlist M3U ordinaire n’a pas une structure série/saison/épisode universelle. Avec M3U, regrouper uniquement lorsque les métadonnées ou noms permettent une inférence raisonnable (ex. `S02E04`) ; sinon proposer une liste d’éléments plats/catégorisés, ne pas inventer une saison ou un titre d’épisode.

### 3.5 Profil et sources

Le panneau Profil permet de créer, tester, modifier, choisir comme actif, actualiser et supprimer une source. Au moins deux types :

**Xtream-compatible**
- Nom de profil facultatif.
- URL de base (schéma, hôte, port et éventuel chemin de base).
- Nom d’utilisateur.
- Mot de passe, masqué par défaut.
- Format live : Auto / HLS / transport stream lorsque le fournisseur propose ces variantes ; une alternative ne doit être essayée que si connue et compatible.
- URL EPG facultative si elle diffère de la source par défaut.
- Bouton **Tester et enregistrer** : valider l’URL, effectuer une requête d’identification/categories limitée, présenter le résultat et les erreurs sans afficher le mot de passe.
- Si la réponse Xtream expose `status`, `exp_date`, `is_trial`, `active_cons`, `max_connections` ou `allowed_output_formats`, afficher ces informations en lecture seule, avec date/fuseau explicites ; ne pas supposer que les champs sont toujours présents ni que `active_cons` est temps réel.
- Accès à un gestionnaire de catégories pour masquer/réordonner des groupes localement (ce n’est pas un verrou parental).

**URL M3U**
- Nom de profil facultatif.
- URL M3U/M3U8 complète (saisie de type URL, support HTTPS prioritaire).
- URL EPG XMLTV facultative ; sinon utiliser l’URL `url-tvg`/`x-tvg-url` détectée si elle existe.
- Boutons **Tester**, **Importer/actualiser** et **Enregistrer**. Afficher le nombre d’entrées/groupes trouvés, progression, taille estimée et avertissements de parsing.
- Après découverte des groupes, permettre d’indexer tous les groupes ou de n’en garder qu’un sous-ensemble ; cette sélection locale sert de solution de repli quand un catalogue est trop volumineux, sans demander au fournisseur de le modifier.
- **Prérequis de persistance (M3U) :** dans une playlist M3U, chaque ligne de lecture **porte elle-même les identifiants** ; il n’existe pas de petit secret séparé. Refuser la persistance reviendrait donc à retélécharger et réindexer le catalogue entier — jusqu’à 256 Mio — à chaque lancement, ce qui est inacceptable pour l’utilisateur, la bande passante et le fournisseur. L’écran de création du profil affiche donc le prérequis **avant** l’import : « Cette source ne fonctionne qu’en conservant ses adresses sur ce téléviseur (stockage privé de l’application, chiffré, effacé avec le profil) », avec deux issues seulement : accepter la persistance, ou ne pas ajouter la source en mode persistant. Aucun import M3U ne démarre sans ce consentement (§8.2). L’exception non garantie (préfixe d’identifiants commun permettant un index sans secret) est décrite §5.2 et reste un confort, jamais une promesse.

**Préférences locales du profil**
- Masquer/afficher et réordonner les catégories live, VOD et séries ; écran de rétablissement des catégories masquées et retour à l’ordre fournisseur.
- Décalage EPG manuel (minutes, 0 par défaut) pour les sources à fuseau ambigu, avec aperçu avant/après correction ; XMLTV dont le fuseau est explicite reste converti normalement. Ne pas modifier l’horodatage source.
- Accès à la correspondance EPG manuelle décrite §5.4.
- Ces préférences améliorent le confort ; elles ne protègent pas les contenus et ne remplacent pas un contrôle parental.

Les champs URL et mot de passe doivent s’utiliser avec le clavier virtuel LG. Utiliser les attributs `type="url"` et `type="password"` ; garder le champ dans la zone visible lorsque le clavier apparaît. Les événements clavier ordinaires ne sont pas une source fiable pour lire les caractères produits par le clavier virtuel : écouter la valeur du champ (`input`/`change`) et traiter `keyboardStateChange` sans annuler une saisie vocale. [S22]

---

## 4. Spécification détaillée de la télécommande

### 4.1 Principes non négociables

- **100 % des opérations courantes doivent être possibles au D-pad et à OK**, sans pointeur ni clavier externe. [S06]
- Le focus ne doit jamais disparaître, être invisible ou sauter vers le haut de l’écran sans raison.
- Chaque écran définit explicitement les destinations gauche/droite/haut/bas aux frontières entre panneaux. Ne pas laisser l’algorithme spatial choisir une destination ambiguë.
- Garder l’état de focus et le scroll par écran. À l’ouverture d’une boîte modale, mémoriser le focus d’origine ; au retour, le restaurer.
- Magic Remote pointeur et roue restent pris en charge en amélioration progressive. Une action au pointeur doit produire le même état de sélection que l’action OK.
- Réserver `@enact/spotlight` comme **seul** gestionnaire spatial V1. Utiliser ses conteneurs, `Spottable`/composants Sandstone et les hooks de direction pour les transitions déterministes. Spotlight sait basculer entre mode 5-way et pointeur et calcule le focus à partir de la géométrie DOM ; les changements dynamiques nécessitent une gestion correcte du cache/focus. [S05]

### 4.2 Codes et actions

Codes de référence publiés pour Magic Remote webOS. Vérifier également `event.key` quand disponible, mais conserver la table de codes webOS testée sur appareil. [S06]

| Touche | Code | Action proposée |
|---|---:|---|
| Gauche / Haut / Droite / Bas | 37 / 38 / 39 / 40 | Déplacer le focus ; dans le lecteur, comportement contextuel décrit ci-dessous. |
| OK / Entrée | 13 | Sélectionner, ouvrir, lire ou valider l’action du contrôle focalisé. |
| Retour / Back | 461 | Revenir à l’étape précédente selon la pile d’historique. À la racine, laisser le comportement standard webOS 6+ ou demander confirmation sans empêcher la sortie système. |
| Rouge / Vert / Jaune / Bleu | 403 / 404 / 405 / 406 | Facultatives ; n’assigner qu’une action annoncée visuellement et testée. Ne jamais rendre une fonction essentielle dépendante de leur présence. |
| Lecture / Pause | 415 / 19 | Contrôle de lecture si la télécommande conventionnelle et l’événement sont disponibles. |
| Avance / Retour rapide / Stop | 417 / 412 / 413 | Stop : quitter le lecteur ; avance/retour : saut VOD borné seulement si `seekable` le permet. Ne pas promettre l’accélération média continue. |
| CH+ / CH− | Non publiés pour usage app | LG indique « App Usage: N/A » pour ces touches. Les sonder sur TV réelle ; ne les mapper que si un événement arrive effectivement à l’application. Ne jamais dépendre d’elles ni intercepter une commande système. D-pad haut/bas reste la navigation de zapping garantie. |

Les commandes Power, Home, volume, mute et les actions réservées au système ne doivent pas être interceptées. Les numéros ne sont pas requis : certaines Magic Remote n’ont pas de touches numériques ; le clavier virtuel est l’alternative.

### 4.3 Carte de focus par écran

Les règles ci-dessous sont écrites en **termes logiques** (« vers le panneau catégories », « vers le panneau EPG », « vers la fiche ») : en RTL, le layout se miroite et ces destinations s’inversent d’elles-mêmes, les touches restant physiques. Aucune règle ne doit être écrite « droite/gauche » sans nommer la destination logique correspondante.

| Écran | Règles de focus (logiques ; miroir appliqué en RTL) |
|---|---|
| Accueil | Grille 2 × 2 ou rangée de quatre cartes selon résolution ; déplacement naturel, focus initial sur le profil actif ou Live si configuré. |
| Live catégories | Haut/bas dans les catégories ; **vers le panneau chaînes** (droite en LTR, gauche en RTL) ; l’autre direction ne quitte pas la page. |
| Live chaînes | Haut/bas dans la liste virtualisée ; **vers le panneau catégories** (bord amont) ; **vers le panneau EPG** (bord aval) ; si aucune action EPG n’est disponible, rester dans le panneau. La colonne EPG est prioritaire ; aucun espace publicitaire vide n’est focalisable. |
| VOD / Séries | Haut/bas/gauche/droite dans la grille ; **depuis le bord amont de la grille, vers le panneau catégories** ; entrée dans une fiche par OK ; le retour restaure l’élément. |
| Fiche série | Focus initial sur les saisons ; **vers la liste d’épisodes** (côté aval) ; haut/bas dans la colonne ; OK confirme la saison/l’épisode. |
| Fiche VOD | Ordre : Lire, Favori, Retour/fermer ; jamais de focus derrière la modalité. |
| Profil | Formulaire vertical ; OK active le clavier virtuel ; le curseur système peut apparaître, mais les autres champs et actions restent D-pad accessibles. |
| Dialogues | Focus piégé dans le dialogue, bouton principal mis en évidence ; Retour ferme ou annule selon le contexte, aucune suppression directe. |
| Lecteur | La vidéo ne capte jamais le focus. Overlay masqué : flèches appliquent les règles de zap/seek du §4.6 ou le font apparaître sans action destructive ; overlay affiché : flèches déplacent le focus dans les contrôles (barre de progression comprise) et OK active. Retour quitte le lecteur. **Exception au miroir : le lecteur conserve l’axe temporel gauche→droite et un mapping physique des flèches de seek, dans toutes les locales (§4.6).** |

### 4.4 Répétition, pressions longues et protection contre les doubles actions

- Traiter `keydown` et `keyup` ; utiliser `event.repeat`/la répétition observée, sans attendre que la touche soit relâchée pour afficher le premier mouvement.
- Le premier D-pad déplace le focus immédiatement. Répétitions bornées par un throttle réglable (cible de départ : 80–120 ms), avec accélération de défilement seulement après un maintien mesuré ; le focus doit s’arrêter sur l’élément final, sans sauter aléatoirement.
- Dédupliquer OK sur un délai court (cible initiale : 250–350 ms) et ignorer une double validation provoquée par la répétition d’une touche tenue.
- Sur une saisie active, ne pas détourner les flèches/OK du clavier virtuel ou du champ. Les raccourcis globaux sont désactivés jusqu’à la fermeture effective du clavier.
- N’appeler `preventDefault()`/`stopPropagation()` que pour une touche effectivement prise en charge par l’application.

### 4.5 Retour, routes et Magic Remote

- Utiliser la pile History webOS : un état par écran/fiche/lecteur, `pushState()` à l’entrée et `popstate` au Retour. Ne pas créer un historique par mouvement de focus.
- Garder le comportement Back standard autant que possible ; ne définir `disableBackHistoryAPI` à `true` que si un test démontre que l’application doit prendre entièrement la main, puis appeler la méthode de retour plateforme dans le cas prévu. LG documente les deux modes et le code Back 461. [S07]
- Laisser le système gérer Back à la racine conformément à webOS 6+ ; éviter une imitation fragile de la sortie système.
- Écouter la visibilité du pointeur et les événements de vie ; quand le système recouvre l’app (Home/Settings), l’app ne doit pas croire que le pointeur est encore disponible. [S06][S16]

### 4.6 Comportement spécifique du lecteur

- **Règle overlay unique :** overlay masqué, seules les flèches contextuelles exécutent une action de lecture (zapping/seek) ; toutes les autres touches directionnelles affichent l’overlay sans action secondaire. Overlay visible, les flèches déplacent le focus parmi les commandes, **sauf lorsque la barre de progression a le focus : elle consomme alors Gauche/Droite pour ajuster la position** (comportement slider, point ci-dessous) ; OK active le contrôle focalisé. Retour quitte le lecteur dans les deux états.
- **Live sans DVR** : overlay masqué, Haut/Bas demandent chaîne précédente/suivante dans la **file de lecture** définie ci-dessous ; Gauche/Droite affichent l’overlay. **Live avec fenêtre DVR seekable** : overlay masqué, Gauche/Droite sautent de 10 secondes ; Haut/Bas zappent. Ne jamais seek dans un direct sans `seekable`.
- **VOD / épisode** : overlay masqué, Gauche/Droite font un saut de 10 secondes si le média est seekable ; Haut/Bas affichent l’overlay. Un maintien répète le seek de façon bornée, sans `playbackRate` autre que 1.0.
- **Seek précis — la barre de progression est un contrôle focalisable** (slider) : overlay visible, la focaliser (Haut/Bas circulent entre les rangées de contrôles), Gauche/Droite ajustent la position cible — pas de 10 s, répétition bornée, puis accélération 30 s/60 s après maintien mesuré ; le code temporel cible s’affiche au-dessus de la barre et OK reprend la lecture à cette position. C’est le chemin garanti pour un seek précis au D-pad ; seek borné à `video.seekable`, jamais de `playbackRate` ≠ 1.0, pas d’affichage de durée infinie (§6.3).
- **Sens des flèches et RTL** : dans le lecteur, l’axe temporel de la barre de progression reste **gauche→droite** et les flèches gardent un sens **physique** (Gauche = retour dans le temps, Droite = avance) dans toutes les locales. C’est la convention des lecteurs vidéo et la seule qui ne rende pas le geste contradictoire avec le contrôle ; le reste de l’habillage (ordre des boutons, titres, panneaux) se miroite normalement (§9.1). QA RTL explicite sur ce point.
- **Séquence de zapping = file de lecture explicite**, transmise au lecteur à l’ouverture et mémorisée dans l’état d’historique ; « la catégorie visible » n’est qu’un des cas :
  - depuis une catégorie Live : chaînes visibles de cette catégorie, dans l’ordre affiché ;
  - depuis **Favoris** : favoris dans l’ordre affiché (liste inter-catégories ; on ne « saute » pas implicitement d’une catégorie à l’autre, on suit la liste de favoris) ;
  - depuis une **recherche globale** : la liste de résultats affichée, dans son ordre, sans relancer la recherche ni changer de catégorie ;
  - depuis une **fiche série** : épisodes de la saison courante (Haut/Bas = épisode précédent/suivant) ;
  - depuis une grille VOD : la liste affichée de la catégorie courante.
  Les catégories masquées et les entrées sans `streamRef` résolu sont exclues ; pas de wrap en bout de file ; un toast nomme la file au premier zap (« Favoris », « Résultats », « Saison 2 »). Retour restaure exactement l’écran d’origine et sa file.
- **Coalescence et fermeture du flux** : déplacer la sélection UI immédiatement, mais n’ouvrir le flux demandé qu’après 400 ms sans nouvelle touche (valeur de départ 300–500 ms). Garder seulement la dernière demande pendant cette fenêtre. Sérialiser le changement : arrêter/annuler l’ancienne session, retirer sa source et attendre `emptied` ou un timeout borné avant d’affecter la nouvelle URL. Aucun deuxième `<video>`/décodeur ne doit rester actif.
- La barre de commandes disparaît après temporisation uniquement si le focus n’est pas sur un contrôle. Une touche non contextuelle la réaffiche. Afficher les raccourcis la première fois ; ne pas les laisser invisibles.
- Au Back, arrêter proprement le flux et faire de la dernière chaîne réellement lancée la sélection restaurée. Revenir à **l’écran d’origine de la file** — sa catégorie pour un lancement depuis une catégorie, la liste Favoris pour un lancement depuis Favoris, la liste de résultats pour un lancement depuis la recherche — même si cet écran diffère de celui où l’utilisateur était avant lecture ; retrouver index/scroll ; si la liste a changé, choisir le voisin le plus proche et montrer une indication discrète.

---

## 5. Sources, formats et normalisation des données

### 5.1 Adaptateurs fournisseur

Xtream-compatible et M3U ne doivent pas être codés en dur dans les vues. Chaque profil expose une interface commune :

```text
ProviderAdapter
  testConnection(profile)
  getCategories(contentType)
  getItems(contentType, categoryId, cursor?)
  searchItems(contentType, normalizedQuery, cursor?)
  jumpToLetter(contentType, letter, cursor?)
  getDetails(contentType, itemId)
  getEpg(channelId, timeRange)
  resolveStream(item, requestedFormat)
  refresh(profile)
```

Les actions et champs Xtream sont une convention de facto, non une API unique garantie. Tester la réponse réelle, gérer identifiants en nombre ou chaîne, champs manquants, versions de noms (`series_id`, par exemple) et variantes EPG ; ne jamais supposer que toute réponse 200 est un JSON valide.

**Appels courants à encapsuler dans l’adaptateur Xtream-compatible :** authentification `player_api.php`, catégories live/VOD/séries, listes filtrées par `category_id`, détail VOD, détail série avec saisons/épisodes, EPG court ou XMLTV. Les URL de flux sont construites séparément à partir du format/identifiant communiqué par le fournisseur, sans stocker la chaîne d’URL dans les logs.

Xtream ne fournit pas toujours une pagination ni une recherche serveur. Respecter les pages si l’API les propose.

**Indexation : l’appel « tous les flux » d’abord, la catégorie ensuite.** Les portails Xtream exposent `get_live_streams`, `get_vod_streams` et `get_series` **sans filtre de catégorie** : c’est la voie par défaut — trois appels de liste plus les trois listes de catégories — ingérée par parseur de tableau incrémental vers `ServiceCatalogStore`. Le découpage **par catégorie** (`get_live_streams_by_category`, etc.) n’est qu’un **repli** : portail qui refuse ou limite l’appel global, quotas par réponse, reprise après échec, ou sélection volontaire d’un sous-ensemble de groupes. Des centaines de catégories multiplieraient les requêtes, exposeraient aux 429 et allongeraient une indexation que la TV ne peut pas poursuivre en arrière-plan (§2.4, §7.3). Les deux voies alimentent le même index versionné ; la reprise se fait par lots (offsets de page ou de catégorie), jamais par un état en mémoire.

Ne jamais appeler `JSON.parse()` sur une réponse entière potentiellement géante ni conserver tous les objets en mémoire. L’index de recherche reste **attaché à la souscription LS2 au premier plan** : il publie sa progression, s’annule, se reprend, et l’interface signale « résultats incomplets » tant que l’indexation n’est pas terminée.

**Portée de la recherche :** chaînes, films et **titres de séries**. Les **épisodes** ne sont pas indexés globalement, car Xtream ne les expose que par le détail d’une série (`get_series_info&series_id`) ; l’écran de recherche l’indique et l’épisode reste accessible par la fiche série (§3.4). Aucun compteur « N résultats » ne doit suggérer une recherche exhaustive sur les épisodes.

### 5.2 Import M3U

Le parser M3U doit prendre en charge les conventions habituelles, sans prétendre qu’elles forment un schéma IPTV entièrement standard :

- `#EXTM3U`, `#EXTINF`, commentaires, lignes vides, CRLF/LF, BOM UTF-8.
- Attributs fréquents `tvg-id`, `tvg-name`, `tvg-logo`, `group-title` et numéro de chaîne.
- Valeurs citées pouvant contenir des espaces ou virgules : séparer le titre au bon séparateur, pas à chaque virgule.
- URL média sur la ligne suivante ; tolérer certains commentaires/interlignes intermédiaires ; ignorer proprement les entrées incomplètes et compter les avertissements.
- `url-tvg`/`x-tvg-url` comme EPG par défaut ; permettre une URL EPG distincte saisie dans Profil.
- Pas de `innerHTML` : les noms provenant de playlist/provider restent du texte, échappé comme contenu.
- N’accepter que les schémas de lecture approuvés et testés (`http`/`https` en V1). Signaler les schémas/protocoles non pris en charge au lieu de tenter de les exécuter.
- Pour l’import, borner les redirections (maximum initial de 5), revalider le schéma et l’hôte à chaque saut, ne jamais transférer `Authorization`/cookies à un autre hôte et demander confirmation avant un downgrade HTTPS→HTTP. Refuser loopback par défaut ; les serveurs LAN privés peuvent être autorisés explicitement par l’utilisateur, car ils sont un cas d’usage possible.
- Détecter les doublons par identifiant/URL de source ; ne pas supprimer deux chaînes distinctes au seul motif qu’elles ont le même nom.
- Ne pas télécharger simultanément toutes les images. Charger les logos/posters à la demande et limiter l’activité réseau.
- **Marquage des URL de flux non sûres.** À l’indexation, analyser l’hôte de chaque URL de lecture et marquer les entrées dont l’hôte est une **adresse IP littérale** privée (`10/8`, `172.16/12`, `192.168/16`), loopback (`127/8`, `::1`), link-local (`169.254/16`, `fe80::/10`, métadonnées `169.254.169.254`), CGNAT (`100.64/10`), `0.0.0.0`, multicast, ou IPv4 encapsulée dans IPv6 — y compris en notation décimale compacte, octale ou hexadécimale (`2130706433`, `0177.0.0.1`). Ces entrées ne sont pas jouables sans **autorisation LAN explicite du profil** (§2.5). Les **noms d’hôte** qui résolvent vers une adresse privée restent un risque résiduel assumé : le pipeline média résout lui-même son DNS et l’application ne peut pas épingler l’IP (§6.1).
- **Persistance et identifiants.** Tester à l’import si les URL de lecture partagent un **préfixe porteur d’identifiants** (cas courant `http://hôte:port/utilisateur/motdepasse/…`). Si oui, l’index peut ne conserver que la partie non secrète (identifiant de flux, extension, forme d’URL) et l’URL est reconstruite à la lecture à partir de l’URL de playlist que l’utilisateur re-saisit à la session suivante ; sinon, la conservation des URL complètes exige le consentement §8.2. Cette détection est une **optimisation de confort, pas une garantie** : elle est consignée sans secret, tout échec de reconstruction retombe sur une demande explicite à l’utilisateur, et l’absence de motif commun n’empêche pas l’import — elle rend simplement le consentement obligatoire.
- **Tranches alphabétiques dans les données.** L’index mémorise par entrée le **script dominant** du titre normalisé et les compteurs par tranche (§9.1), afin que le saut alphabétique se calcule sur le catalogue et non sur la locale de l’interface.

**Stockage et lecture de grands catalogues :** le service ne garde jamais le catalogue entier en RAM ni dans DB8. Il lit la réponse ligne par ligne après décompression et écrit un index compact versionné et **chiffré au repos** (§8.2) dans son répertoire privé sous `/media/internal` : dictionnaire des groupes partagé, enregistrements par chaîne/film/épisode, source order, métadonnées d’affichage, script/tranche alphabétique, `streamRef` opaque et URL de lecture (traitée comme un secret, en mode `storedSecret` ou reconstruite en mode `derived`), plus index d’offsets par groupe et titre normalisé. `getItems(category,cursor)`/recherche ouvrent l’index et renvoient 100–250 objets maximum par réponse LS2. Écrire d’abord `catalog-<version>.tmp`, valider l’import, puis basculer atomiquement vers la nouvelle version ; un import annulé/arrêté ne remplace jamais le dernier index valide. La suppression du profil efface index, fichiers temporaires et références associées. Au prochain appel, le service rouvre les fichiers : aucune dépendance à une mémoire de processus durable. [S33]

Les plafonds initiaux de 25 Mio/50 000 entrées sont retirés. La qualification vise au minimum **250 000 entrées ou 256 Mio décompressés** (test de capacité, pas plafond produit). Mesurer les octets reçus après décompression, le nombre d’entrées, l’espace disque libre et le pic mémoire au fil du flux ; ne pas faire confiance au seul `Content-Length`. Le seuil dur est un quota de sécurité configuré selon les mesures du plus petit téléviseur, pas une constante universelle. Si le quota est atteint, continuer au besoin un scan léger des noms de groupes sans conserver les URLs, présenter la sélection locale, puis refaire un téléchargement en n’indexant que les groupes choisis ; indiquer que cette opération peut retélécharger la source. Conserver l’ancien catalogue pendant ce parcours et ne jamais exiger que l’utilisateur fasse modifier la playlist chez son fournisseur.

**Classification M3U :** durée live inconnue et groupe/type explicite pour les chaînes ; durée finie ou groupe `film`/`vod`/`movie` pour VOD. Détection série uniquement à partir de champs/noms explicites (ex. SxxExx) et avec repli « liste non structurée ». Un profil Xtream-compatible est la source de référence pour une navigation fiable saisons/épisodes.

### 5.3 Modèle de domaine minimal

| Objet | Champs normalisés (exemples) |
|---|---|
| `Profile` | `id`, `name`, `providerType`, `baseUrl`, `username` si nécessaire, `secretRef`, `epgUrl?`, `preferredLiveFormat`, `lastSyncAt`, `status` |
| `Category` | `id`, `profileId`, `contentType`, `name`, `sourceOrder` |
| `CategoryPreference` | `profileId`, `categoryKey`, `isHidden`, `manualSortOrder?` |
| `Channel` | `id`, `categoryId`, `name`, `logoUrl?`, `epgId?`, `sourceKey`, `streamRef`, `sourceOrder` |
| `Movie` | `id`, `categoryId`, `title`, `posterUrl?`, `year?`, `duration?`, `plot?`, `rating?`, `streamRef` |
| `Series` | `id`, `categoryId`, `title`, `posterUrl?`, `plot?`, `seasons[]` ou référence de détail |
| `Episode` | `id`, `seriesId`, `seasonNumber?`, `episodeNumber?`, `title`, `duration?`, `plot?`, `streamRef` |
| `Program` | `epgChannelId`, `startUtc`, `endUtc?`, `title`, `description?`, `category?`, `imageUrl?` |
| `FavoriteReference` | `profileId`, `contentType`, `sourceKeyHash`, `lastSeenName`, `updatedAt`, `matchState` |
| `EpgMapping` | `profileId`, `sourceKeyHash`, `epgChannelId`, `matchMethod`, `updatedAt` |
| `PlaybackPosition` | `profileId`, `contentType`, `sourceKeyHash`, `positionSeconds`, `durationSeconds?`, `updatedAt`, `completed` |
| `CatalogIndexManifest` | `profileId`, `indexVersion`, `contentType`, `entryCount`, `bytes`, `scriptBuckets`, `createdAt`, `state` (building/valid) — écrit par le service, jamais en DB8 |

`streamRef` est une référence opaque résolue par le service dans son index privé ; l’URL de lecture authentifiée n’est ni renvoyée ni persistée dans DB8 et reste en mémoire aussi longtemps que nécessaire. Deux modes de résolution coexistent selon le consentement §8.2 : `storedSecret` (URL complète conservée dans l’index, chiffrée au repos) et `derived` (URL reconstruite à la lecture depuis un préfixe d’identifiants fourni en session) ; le mode est un champ de l’entrée d’index, jamais une supposition de l’interface.

**Identité stable et réassociation :** Xtream utilise d’abord l’identifiant fournisseur avec `profileId` et `contentType`. En M3U, `sourceKey` est un SHA-256 d’une **clé composite** : `contentType` + `tvg-id` + **nom normalisé sans retrait des suffixes qualité/langue** + `group-title` + **rang d’occurrence** parmi les lignes strictement identiques, dans l’ordre source. Le rang évite que deux lignes identiques fusionnent.

Cette clé **stricte** est la seule qui décide de l’identité : « TF1 HD », « TF1 FHD » et « TF1 4K » restent trois entrées distinctes, conformément à « ne pas fusionner deux chaînes distinctes ». `tvg-id` est un **indice, pas une identité** : il est souvent partagé par des variantes ou des flux de secours, parfois faux ; il ne sert jamais seul, et s’il se répète dans un même profil il ne départage rien — ce sont le nom et le groupe qui distinguent. L’URL/token ne participe à aucune clé.

Une **clé assouplie** — nom normalisé avec suffixes qualité/langue retirés (liste prudente partagée avec l’EPG, §5.4) + groupe + type — n’est utilisée **que pour la réassociation**, après actualisation : elle propose un candidat unique pour un favori ou une reprise devenus non appariés. S’il reste plusieurs candidats ou si la correspondance est ambiguë, l’entrée est présentée à l’utilisateur (§5.4) et n’est **jamais reliée automatiquement**. Les références DB8 stockent `sourceKeyHash`, `lastSeenName`, le groupe et le rang observés, ce qui permet d’afficher la proposition puis de l’accepter, la modifier ou la supprimer.

### 5.4 EPG

- Correspondance automatique : `tvg-id`/identifiant de chaîne exact en premier ; sinon normaliser prudemment les noms (Unicode, casse, accents, espaces) et retirer uniquement des préfixes de langue/suffixes de qualité configurés. Auto-apparier seulement une correspondance unique au-dessus du seuil de confiance ; sinon proposer des candidats sans choisir à la place de l’utilisateur.
- Ajouter dans Profil un écran **Correspondance EPG** : chaînes sans ID/correspondance, suggestions, aperçu du programme trouvé, recherche dans les IDs/noms EPG, accepter/changer/effacer. Persister la correction par `sourceKeyHash`, pas par URL de stream ; conserver l’état « non apparié » après refresh si le candidat est ambigu.
- XMLTV stocke des chaînes et programmes avec IDs et heures début/fin ; parser le flux avec un parseur SAX/streaming dans le service Node, filtrer au fil de l’eau par chaînes connues et fenêtre utile (ex. maintenant ± 24–48 h), et ne jamais construire un DOM XMLTV complet. Convertir les dates XMLTV en UTC puis afficher dans le fuseau système. [S25]
- Pour l’EPG court Xtream, conserver l’heure brute et ajouter un **décalage manuel par profil** (minutes, valeur initiale 0, plage bornée) avec aperçu du programme à l’heure corrigée ; cela couvre les fournisseurs qui renvoient un fuseau implicite/mal documenté. La correction ne doit pas modifier les données source enregistrées.
- Traiter les gros EPG en streaming/chunks, avec mémoire bornée et annulation coopérative ; appliquer une limite de réponse décompressée calculée selon l’espace disponible, pas un seuil fixe de 50 Mio. Sur quota, garder le cache précédent et proposer l’EPG court/une fenêtre plus petite.
- Charger l’EPG en arrière-plan après les premières catégories ; mettre à jour à la sélection de chaîne avec debounce de 150–250 ms ; ne pas requêter à chaque répétition de touche.
- Si le guide est vide, trop ancien, non apparié ou invalide, afficher une indication discrète ; la lecture live reste indépendante.
- Ne pas recharger la totalité d’un XMLTV à chaque navigation ; cache borné des programmes filtrés et rafraîchissement contrôlé (valeur de départ 6–12 h, ajustable par profil).

---

## 6. Lecture média et états du lecteur

### 6.1 Stratégie de lecture

- Utiliser un **seul élément `<video>` réutilisable** ; éviter plusieurs décodeurs actifs lors des changements de chaîne.
- À chaque nouveau média : annuler les listeners/timers de la session précédente, arrêter la vidéo précédente, vider sa source proprement, affecter la nouvelle URL, appeler `load()` puis `play()` après l’action OK. Observer le rejet de `play()` : si le moteur exige une activation utilisateur directe, afficher « Appuyez sur OK pour démarrer » et retenter depuis cette nouvelle pression, sans classer cela comme un échec de codec.
- HLS : privilégier le HLS natif webOS. La spécification LG déclare HLS pris en charge sur le téléviseur, mais documente également des tags HLS absents/partiels, l’absence d’avance/retour rapide continu et la prise en charge du seek selon le média. [S14]
- Matrice média à valider sur chaque gamme : HLS, MPEG-TS progressif sur HTTP(S), H.264/MPEG-2/HEVC, audio AAC/AC-3 (Dolby Digital)/E-AC-3 (Dolby Digital Plus) et un cas 1080i. LG répertorie certains de ces codecs/conteneurs pour webOS 6, mais la combinaison réelle, le modèle, le profil, le débit et l’entrelacement changent la compatibilité ; aucun support universel n’est promis. GMC/Qpel et certains profils ne sont pas pris en charge. [S15]
- URL sans extension : ne pas se fier à une détection par nom. Construire un `<source>` avec `type` MIME uniquement si le fournisseur ou une sonde contrôlée l’établit ; pour HLS, tester explicitement le MIME/`mediaTransportType` webOS documenté. Si le type reste inconnu, tenter une seule lecture native puis afficher une erreur claire, sans essais indéfinis. Le paramètre MIME doit être validé sur TV, pas seulement desktop. [S35]
- **Ordre de préférence pour établir le type :** 1) métadonnées du fournisseur (`container_extension` du détail Xtream, variante/format du profil) ; 2) forme d’URL connue (`m3u8`) ; 3) **sonde bornée** ; 4) tentative native unique sans `type`.
- **Discipline de la sonde** : une sonde est une **connexion au flux**. Elle consomme une connexion du compte et peut déclencher `max_connections` (§7.2) ou être comptée comme une session par le fournisseur. Règles : **jamais de sonde quand `max_connections ≤ 1`** ni quand le profil est en format Auto sans variante connue — les métadonnées et la tentative native suffisent ; au plus une sonde à la fois par compte, comptée dans le sémaphore de concurrence (§7.3) ; **la sonde est fermée/annulée avant `load()`/`play()`**, jamais exécutée pendant la fenêtre de coalescence du zap ni pendant une lecture ; résultat mémorisé par forme d’hôte+chemin pour ne pas resonder à chaque changement de chaîne. Si un doute subsiste, préférer un échec propre et documenté à une sonde non maîtrisée.
- **Boucles de qualification sur comptes dédiés** : les boucles d’essais de la phase 0 (10 essais par cas puis 30 par cellule) s’exécutent sur des **comptes de test dédiés** fournis à cet effet, jamais sur le compte d’un utilisateur : un compte à `max_connections=1` ne peut pas soutenir un test de charge, et marteler un compte réel peut le faire bloquer. Documenter `max_connections` et les formats autorisés de chaque compte de test.
- **Contrôle avant `video.src`** : avant d’affecter l’URL au média, revérifier le marquage d’hôte de l’entrée (§5.2) contre la politique LAN du profil actif ; une entrée marquée « non sûre » sans autorisation explicite n’est pas jouée et produit « destination réseau non autorisée » (§7.2). Les redirects du pipeline média restent hors de portée de l’application : ce contrôle est best-effort et documenté comme tel (§2.5).
- Tester les redirects 301/302/307/308 vers le même et un autre hôte, changement HTTP↔HTTPS, DNS et autorisation. La pile `<video>` suit ses propres redirects ; l’app ne peut pas inspecter tous les en-têtes/statuts. Ne pas transmettre de credentials d’API par défaut au nouvel origin ; URLs contenant tokens restent des secrets.
- Le HTTP clair peut être nécessaire pour certains fournisseurs : autoriser après avertissement explicite **une seule fois par profil/host** (répéter si le schéma/hôte change), mémoriser le choix et rappeler que réseau/credentials peuvent être observables. HTTPS reste la préférence ; ne jamais downgrader silencieusement.
- Limite d’interface : `<video>` natif ne permet pas de garantir l’envoi d’en-têtes d’authentification arbitraires. Les fournisseurs qui exigent un header propriétaire doivent être déclarés non supportés tant qu’un prototype TV ne valide pas une solution. Le token d’une URL M3U ou de lecture est un secret.
- Ne pas inclure HLS.js dans le chemin par défaut : il transfère au navigateur la segmentation/buffer en JavaScript et augmente les coûts CPU/mémoire. La V1 s’appuie sur le pipeline média natif ; toute bibliothèque MSE (par ex. Shaka) est une évolution après mesure/prototype ciblé.
- L’usage de DASH/MSE/EME ou d’un DRM doit faire l’objet d’une décision produit, d’une validation des droits/licences et d’essais appareil par appareil. La matrice LG webOS 6 décrit certaines combinaisons HLS-AES128 et MSE/EME-PlayReady/Widevine ; cela ne signifie pas que tous les flux DASH/DRM fournisseurs fonctionnent. [S14]
- Si le manifeste indique des tags non pris en charge, segments A/V incompatibles, un codec non pris en charge ou un certificat TLS non reconnu, afficher une erreur « source non compatible ou inaccessible » et diagnostic non sensible. Ne pas boucler indéfiniment.

### 6.2 Machine d’état

```text
IDLE → PREPARING → BUFFERING → PLAYING
                     ↘ ERROR     ↗
PLAYING ↔ BUFFERING
PLAYING/PAUSED → STOPPING → IDLE
```

- **PREPARING** : valider référence, URL et profil ; arrêter ancien média ; afficher titre et affiche/logo.
- **BUFFERING** : spinner discret, commandes déjà accessibles, timer de démarrage.
- **PLAYING** : masquer le spinner ; afficher un toast temporaire au changement live.
- **PAUSED** : afficher les commandes ; état réservé principalement à VOD et commandes télécommande compatibles.
- **ERROR** : message, cause probable, actions Réessayer/Retour/Autre chaîne ; préserver l’accès Back.
- **STOPPING** : pause, retirer la source, appeler `load()` pour libérer le pipeline ; ne garder aucune session de lecture inutile.

Événements à observer : `loadstart`, `loadedmetadata`, `canplay`, `playing`, `waiting`, `stalled`, `timeupdate`, `durationchange`, `seeking`, `seeked`, `pause`, `ended`, `abort`, `emptied`, `error`. Gérer proprement le démontage de l’écran, un changement de profil et le passage de l’application en arrière-plan.

### 6.3 Contrôles VOD et reprise

- Pause/reprise, seek uniquement dans `video.seekable` et à une vitesse normale ; aucune promesse d’avance accélérée ou de vitesse variable.
- Seek précis au D-pad : la barre de progression est un contrôle focalisable (§4.6) — Gauche/Droite ajustent la cible par pas bornés (10 s, puis 30/60 s en maintien), OK relance la lecture à la position choisie. C’est le chemin garanti lorsque l’overlay est visible ; il ne remplace pas les sauts directs de 10 s overlay masqué.
- Afficher une barre de progression seulement si la durée est finie et exploitable ; ne pas afficher `NaN`/durée infinie.
- Enregistrer la position périodiquement à faible fréquence (valeur de départ 30 s) et à pause/fin, pas à chaque `timeupdate`.
- À la reprise : proposer **Reprendre** ou **Depuis le début** ; marquer terminé selon une règle configurable (par ex. position > 95 %), puis autoriser à rejouer.
- Sous-titres/audio : afficher uniquement les pistes réellement signalées et testées par le média natif ; ne pas afficher un sélecteur vide.

### 6.4 Limites de commandes à expliquer à l’utilisateur

LG documente le seek et le live seek (webOS 3+), mais indique que l’avance rapide et le retour rapide continus et les vitesses autres que 1.0 ne sont pas pris en charge par le lecteur média. Les touches de saut VOD doivent donc effectuer une recherche temporelle bornée, pas promettre une lecture accélérée. [S14]

---

## 7. Gestion des erreurs, réseau et reprise

### 7.1 Modèle d’erreur commun

Chaque couche renvoie une erreur structurée, sans texte sensible :

```text
AppError {
  domain: network | cors | auth | provider | playlist | epg |
          playback | storage | platform | unknown
  code: string
  userMessageKey: string
  retryable: boolean
  operationId: string
  httpStatus?: number
  providerCode?: string
  timestamp: number
  safeTechnicalHint?: string
}
```

Ne pas inclure URL complète, paramètres authentifiés, nom d’utilisateur, mot de passe, token ou contenu de playlist dans la console, les rapports de crash ou les dialogues d’erreur.

### 7.2 Matrice d’erreurs et d’actions

| Cas | Détection/limite | Message/action côté TV |
|---|---|---|
| TV hors ligne / Wi-Fi perdu | Statut système si disponible + requête test réelle ; `navigator.onLine` n’est qu’un indice | « Le téléviseur n’est pas connecté à Internet. » Réessayer, ouvrir Profil ou continuer sur catalogue local disponible. |
| DNS, hôte introuvable, délai dépassé | Timeout réseau ; distinguer l’erreur de connexion d’un JSON invalide | « Le serveur ne répond pas. Vérifiez l’adresse et le réseau. » Réessayer ou modifier le profil. |
| URL/redirection refusée par la politique réseau | Schéma interdit, redirect excessif, DNS vers adresse locale/réservée ou origin non autorisé | « Cette destination réseau n’est pas autorisée. » Vérifier l’URL ; autoriser un serveur LAN uniquement dans les réglages avancés et par profil. |
| TLS/certificat/horloge/racine absente | Échec du service, du navigateur ou du player selon la pile utilisée | « Connexion sécurisée impossible. Vérifiez l’adresse, l’horloge TV ou le certificat. » Diagnostic local indique quelle pile a échoué ; ne jamais désactiver TLS. |
| Négociation TLS API impossible (ex. serveur TLS 1.3-only) | Handshake du client Node échoue tandis que navigateur/media peut réussir ; capacité exacte qualifiée en phase 0 | « La connexion sécurisée API n’est pas compatible avec ce serveur. » Ne pas rétrograder TLS ; source/configuration alternative nécessaire. |
| CORS API web | En TV, appels fournisseur par le service JS ; en mode navigateur dev, reconnaître `TypeError`/erreur CORS sans la confondre avec un 404 | « Cette source n’autorise pas l’accès depuis le navigateur. » En TV, vérifier le service fournisseur ; en dev, utiliser un mock ou un relais local explicite. |
| HTTP 401/403 fournisseur | Lire le statut si la réponse API l’expose ; la lecture native peut ne pas exposer le statut HTTP à JS | « Identifiants invalides, abonnement expiré ou accès refusé. » Modifier/tester le profil. Ne pas prétendre connaître la raison exacte si la TV ne donne que `MediaError`. |
| Connexions simultanées épuisées (Xtream) | Code/message fournisseur explicite ou début de lecture refusé alors que `active_cons >= max_connections` (valeurs possiblement obsolètes) | « Limite de connexions atteinte. Fermez une autre lecture sur ce compte puis réessayez. » Une seule tentative manuelle différée ; ne pas changer d’ID/output format pour contourner la limite. |
| HTTP 404/base URL incorrecte | Réponse API disponible | Vérifier URL, port, chemin de base et type d’API. Bouton Modifier le profil. |
| HTTP 429/rate limit | Statut et `Retry-After` si fourni | Suspendre les requêtes, réessayer après le délai ; ne pas marteler le fournisseur. |
| HTTP 5xx fournisseur | Réponse API | « Le fournisseur rencontre un problème. » Réessai borné et manuel. |
| HTML/login inattendu à la place JSON | Content-type ou parsing/schema invalide | « Réponse fournisseur inattendue. Vérifiez l’URL ou le portail. » Aucun crash de l’écran. |
| Profil accepté mais 0 catégorie/chaîne | Liste vide valide séparée d’une erreur | « Aucune chaîne dans cette source » + actualiser/changer de profil. |
| M3U illisible, vide ou mal formée | Header absent, entrées invalides ou fin prématurée | Compteur d’éléments importés + avertissement « Corriger l’URL / Réessayer ». Garder l’ancien index tant que le nouvel import n’est pas validé. |
| Quota disque/volume M3U dépassé | L’index temporaire n’atteint pas le seuil modèle ; espace libre insuffisant pour **2 × index + temporaire** (§2.4) | Ne pas écraser l’ancien index ; proposer de choisir des groupes, libérer de l’espace ou relancer l’import localement. Aucun refus permanent au seul motif d’un catalogue >50 000 entrées. |
| XMLTV vide, XML mal formé, fuseaux/IDs inconnus | Parseur EPG indépendant | « Guide EPG indisponible » ; les chaînes restent utilisables. |
| Logo/poster 404 ou lent | `onError`, délai, chargement différé | Placeholder stable, aucun blocage catalogue ni déplacement de focus. |
| URL source expirée/flux 403 | `MediaError`, test de la source quand possible ; le statut HTTP exact n’est pas toujours disponible au `<video>` | « Flux inaccessible — l’adresse a peut-être expiré ou l’accès est refusé. » Réessayer, autre chaîne ou modifier source. |
| Source/formats/codecs non pris en charge | `MediaError` code 3/4, type média inconnu, manifeste incompatible | « Format non pris en charge par ce téléviseur ou cette source. » Retour au catalogue ; détails dans l’écran local, export manuel uniquement. |
| Lecture refusée avant démarrage | Rejet de `play()` (ex. politique/activation utilisateur), sans `MediaError` exploitable | « Appuyez sur OK pour démarrer la lecture. » Relancer sur une pression explicite et garder Retour accessible. |
| `waiting`/`stalled` trop long | Seuil de départ : 10–15 s sans avancée, ajustable au HLS fournisseur | Distinguer « Mise en mémoire tampon » et « Lecture interrompue ». Réessayer ou revenir au direct ; pas de spinner infini. |
| DB8 plein/indisponible | Erreur LS2 / quota | Préserver l’interface, expliquer que favoris/profil ne peuvent être enregistrés, proposer libérer cache. Pas de perte silencieuse. |
| Retour, changement de profil ou relance en cours de requête | Annulation par `operationId`/token de session | Annuler la requête et empêcher toute réponse ancienne de remplacer le nouvel écran ou le nouveau flux. |
| Index remplacé pendant une pagination ou une recherche | Curseur portant un `indexVersion` périmé au moment de la requête | « Le catalogue a été mis à jour. » L’interface relance **la même requête** sur la nouvelle version en conservant catégorie, requête, scroll et focus ; aucune page partielle n’est fusionnée avec l’ancienne. |
| Racine de confiance absente du service mais présente ailleurs | Échec de vérification limité aux appels du service Node, alors que Chromium ou le média réussissent | « Connexion sécurisée API impossible sur ce téléviseur. » Vérifier le bundle de racines publiques embarqué et l’horloge TV (§2.5) ; ne jamais désactiver la vérification. |
| Hôte de flux privé/loopback non autorisé | Marquage d’index (§5.2) + absence d’autorisation LAN du profil | « Cette destination réseau n’est pas autorisée. » Autoriser explicitement l’hôte dans les réglages avancés du profil si c’est un serveur LAN voulu. |
| Application masquée/arrière-plan | `visibilitychange`, `webOSRelaunch` | Arrêter chargements de posters/EPG non nécessaires ; pause/arrêt média selon la politique ; restaurer l’état sûr au retour. [S16] |

**Important :** les erreurs `HTMLMediaElement` ne donnent pas toujours le code HTTP précis du segment/manifeste. Utiliser les codes média (abandon, réseau, décodage, source non prise en charge), le statut réseau observable et des tests ciblés ; ne pas déduire à tort qu’un code 4 signifie mot de passe erroné. [S14]

### 7.3 Timeouts, concurrence et réessais

Valeurs de départ à mesurer et à rendre configurables :

- Appels API catégories/détails : délai connexion 8 s ; délai total 15 s.
- Import M3U/EPG : deadline et quota disque dépendant du modèle, progression par subscription LS2, annulation par Retour/changement de profil ; reprendre avec Range/ETag seulement après preuve que le serveur supporte ces mécanismes.
- Au plus 4 requêtes de métadonnées fournisseur simultanées ; pas de chargement global de toutes les images. Ce plafond inclut la sonde MIME (§6.1), qui est en outre unique par compte et fermée avant toute lecture.
- Indexation Xtream : appels « tous les flux » en priorité (§5.1) ; les replis par catégorie sérialisent leurs lots et respectent le même plafond de 4, jamais en parallèle d’une lecture du même compte quand `max_connections` est bas.
- GET idempotent : au plus 2 réessais automatiques sur timeout/erreur réseau/5xx, backoff avec jitter (ex. 1 s puis 3 s). Pas de retry auto sur credentials incorrects, CORS, parsing ou refus 403/404.
- HTTP 429 : respecter `Retry-After`, sinon temporisation plus longue, jamais boucle serrée.
- Démarrage vidéo : indicateur immédiat ; timeout utilisateur initial 20 s à calibrer sur réseau de référence ; un clic Réessayer lance une nouvelle session et annule l’ancienne.
- Flux live : au plus 3 reconnexions automatiques dans une fenêtre de 2 minutes et uniquement sur incident réseau récupérable. Pas de zapping silencieux vers une autre chaîne.
- La connexion TV est un état indicatif : effectuer une requête réelle à la source au besoin. `online` ne prouve pas que le DNS, le portail ou le flux est disponible.

---

## 8. Données persistantes, secrets et confidentialité

### 8.1 Modèle de menace et persistance

**Menaces couvertes en V1 :** fuite accidentelle dans les logs/diagnostics, lecture par une autre app ordinaire, interception réseau lorsque TLS est disponible, injection de contenu fournisseur et réutilisation involontaire après suppression de profil. **Hors garantie :** téléviseur rooté/compromis, extraction forensique des fichiers par un attaquant ayant accès privilégié au système, malware exécuté dans le même contexte. Cette limite est affichée dans la page confidentialité ; l’application ne prétend pas que DB8 ou le sandbox équivalent à un coffre matériel. [S20][S33]

- DB8 (kinds versionnés + validation runtime) stocke profils, préférences, favoris, correspondances EPG et positions de reprise, pas le catalogue complet. `private:true` sert à la politique de suppression/désinstallation selon les APIs DB8 ; ce n’est **pas** une promesse de chiffrement. [S17]
- Les index compacts M3U/Xtream vivent dans l’espace privé du service sous `/media/internal`, avec même UID de service entre exécutions ; le service est le seul accès applicatif à ces fichiers. Les URLs M3U de lecture qu’ils contiennent restent des secrets : ils sont **chiffrés au repos** (§8.2) et suivent la même suppression que le profil. Le chiffrement ne change pas la garantie annoncée : un attaquant privilégié reste hors périmètre. [S29][S33]
- `localStorage` est réservé à des réglages non sensibles et non critiques. LG indique que le stockage local d’une application empaquetée peut être supprimé lors d’une mise à jour/suppression ; prévoir migration et ne pas y garder la seule copie des favoris/profils. [S19]
- Import temp + bascule atomique : les écritures interrompues ne corrompent ni DB8 ni le dernier index valide.

### 8.2 Secrets : ergonomie et protection

Les URL M3U, réponses API, index local et URL de lecture peuvent contenir des identifiants ou tokens. La pile média reçoit l’URL décodée pour lire le flux : l’application peut réduire sa durée de vie, mais ne peut pas cacher ce secret au système média ni au serveur fournisseur. Aucun secret dans l’UI de diagnostic, logs, crash reports ou télémétrie.

- **V1 webOS 6–23 — profil Xtream :** demander un consentement explicite **« Mémoriser ce profil sur ce téléviseur »**, puis conserver les identifiants dans les données DB8 app-aware et les URLs de catalogue dans le stockage privé du service. Pas de passphrase obligatoire ni de saisie répétée à chaque lancement : le compromis est cohérent avec la menace §8.1 et doit être divulgué. Si l’utilisateur refuse, le secret Xtream (utilisateur/mot de passe, quelques dizaines d’octets) reste en mémoire de session et est redemandé au lancement suivant ; le catalogue déjà indexé, lui, n’est pas rejoué.
- **V1 webOS 6–23 — source M3U : le consentement de persistance est un prérequis affiché, pas une option.** Dans une playlist M3U, chaque ligne de lecture **porte** les identifiants : il n’existe pas de petit secret séparé que l’on pourrait redemander. Refuser la persistance reviendrait à retélécharger et réindexer jusqu’à 256 Mio à chaque lancement — inacceptable pour l’utilisateur, la bande passante et le fournisseur. L’écran de création du profil annonce donc la règle **avant** l’import (« Cette source ne fonctionne qu’en conservant ses adresses sur ce téléviseur, dans le stockage privé de l’application, chiffré, effacé avec le profil »), avec deux issues : accepter la persistance, ou renoncer à cette source. L’unique exception — non garantie — est le cas où les URL partagent un préfixe porteur d’identifiants : l’index ne conserve alors que la partie non secrète et l’URL de playlist est redemandée à la session suivante (§5.2).
- **Chiffrement au repos dans tous les cas (exigence LG).** L’index du service contient des URL authentifiées ; il est écrit **chiffré en AES-256-GCM par blocs de taille fixe**, avec une clé aléatoire par profil conservée dans DB8 app-aware et un IV/nonce dérivé du numéro de bloc. Les pages sont déchiffrées à la lecture : pas de KDF sur le chemin chaud, coût mesuré en phase 0 sur la TV de référence. Cette mesure répond à l’exigence de la Self Checklist LG (« les informations de confidentialité et d’identifiants doivent être stockées dans un espace sûr ou chiffrées ») et couvre la lecture par une autre application ou un accès non privilégié ; elle **ne résiste pas** à un attaquant privilégié (§8.1) et n’est pas présentée comme une protection matérielle. Repli uniquement si les mesures montrent un coût inacceptable : stockage non chiffré **avec** la même divulgation explicite — jamais un chiffrement annoncé mais absent. [S20][S29]
- **Keymanager3/TEE : reporté hors V1.** Le modèle de menace §8.1 exclut précisément l’attaquant contre lequel une clé TEE protège (TV rootée, extraction privilégiée), aucune API de coffre n’est documentée avant webOS 24, et la supporter maintenant ajouterait un second chemin de secrets à tester sans bénéfice couvert. À traiter **avec** l’option passphrase (§8.2, option future) dans une version ultérieure, quand webOS 24+ devient le plancher de support : à ce moment-là, la clé d’index migre vers Keymanager3 lorsqu’il est disponible, avec repli DB8 conservé. [S18]
- Option future « chiffrement local avec passphrase » pour les utilisateurs qui veulent résister à l’extraction de stockage : AES-GCM avec KDF versionnée, sel aléatoire et nonce unique. Si PBKDF2-HMAC-SHA-256 est retenu, 600 000 itérations est la baseline OWASP indiquée pour ce KDF ; la note FIPS concerne le choix de PBKDF2. Exécuter en asynchrone et mesurer sur la TV minimale ; ne jamais réduire le coût en silence. [S30][S31]
- Un URL/token transmis à une source HTTP claire est observable sur le réseau. Avertir avant la première requête claire vers chaque hôte (API, M3U/EPG ou média) du profil, mémoriser l’acceptation par couple profil/hôte/schéma, et réavertir si le schéma ou l’hôte change. Proposer HTTPS si la source le supporte ; aucune alerte répétitive à chaque lecture.
- Ne jamais coder une clé dans le paquet ni désactiver la validation TLS. Effacer les références en mémoire au changement de profil/fermeture, sans prétendre pouvoir effacer immédiatement les copies internes du décodeur.

### 8.3 Confidentialité, logs et diagnostics

- Aucun compte d’application, analytics, télémétrie ou identifiant TV n’est nécessaire en V1 ; aucun diagnostic n’est envoyé automatiquement.
- Ajouter un **écran de diagnostic local** (statut service, dernière erreur normalisée, version webOS, opérations et métriques non sensibles) et un **export manuel** après aperçu/consentement. Scrubber obligatoire : retirer URL complète, query strings, user/password/token, IP privée, noms de chaîne si l’utilisateur n’a pas opté explicitement pour les inclure.
- Pas d’envoi d’identifiants, d’historique, d’IP ou de titre regardé à un serveur du développeur.
- Prévoir suppression d’un profil et de ses caches/favoris, effacement global des données et page confidentialité en langage simple.
- Tout SDK futur (publicité/analytics) nécessite examen des accès, divulgation et consentement lorsque requis ; vérifier la politique LG à jour avant intégration. [S20]

---

## 9. UI, identité visuelle et performance

### 9.1 Direction UI

- Interface « salon » : lisible à distance, peu de texte secondaire, hiérarchie plate et panneaux clairement nommés. LG recommande la simplicité, peu d’idées simultanées, de grands éléments et un usage mesuré du mouvement. [S23]
- Fond sombre bleu/ardoise, surfaces légèrement différenciées, accent turquoise/cyan ou ambre réservé au focus/état actif. Contraste élevé, libellés explicites, icônes toujours accompagnées d’un nom lorsque leur sens n’est pas évident.
- Typographie TV : titres généreux, noms de chaînes/posters lisibles, texte EPG avec lignes limitées et « Lire la suite » focalisable.
- Effets élégants mais économiques : légère mise à l’échelle du focus, contour lumineux, transitions fondues ou `transform`/`opacity` de 150–220 ms. Pas de vidéo d’arrière-plan, filtres flous continus, grosses ombres sur chaque carte ni animation déclenchée par chaque rerender.
- Respecter une zone sûre d’écran (5 % environ comme cible initiale) et vérifier 1280×720, 1920×1080 et 4K ; éviter qu’un focus en bord de liste soit coupé.
- Maintenir une grille de 8 px logique (adaptée à la densité TV), espacement constant et cartes assez grandes pour cliquer au pointeur.
- Affiche/poster : image adaptée à la taille de carte, lazy-load, ratio stable et fallback ; aucune grande image inutile décodée à l’ouverture du catalogue.
- **RTL dès l’architecture :** régler `dir="rtl"` à la racine selon la locale, utiliser les propriétés CSS logiques (`margin-inline`, `inset-inline`, `text-align:start`) et éviter les `left/right` codés en dur. Ce qui se miroite est le **layout** — ordre et position des panneaux, côté du panneau catégories, transitions Spotlight exprimées sur l’axe logique — pas les **touches**, qui restent physiques : toutes les règles de focus sont écrites en termes logiques (§4.3, « vers le panneau catégories ») et se lisent à l’envers en RTL sans réécriture. **Deux exceptions explicites**, traitées §4.6 : l’axe temporel de la barre de progression du lecteur et le mapping Gauche/Droite du seek restent physiques. QA RTL dédiée sur ces points.
- **Saut alphabétique :** les tranches proviennent des **scripts réellement présents dans les données indexées** (latin A–Z, arabe, cyrillique, hébreu, grec, CJK, chiffres et symboles regroupés sous `#`), pas de la locale de l’interface : une interface française avec un catalogue arabe affiche les tranches arabes, et inversement. Les tranches suivent l’ordre canonique du script, affichent leur nombre d’entrées et masquent celles qui sont vides. Si aucune tranche exploitable n’existe, un unique groupe `#` reste affiché.
- **Normalisation de tri partagée :** le même module de normalisation que l’EPG (§5.4) retire, pour les besoins de tri et de recherche uniquement, les préfixes fournisseur (`FR|`, `|4K|`, `[VIP]`, flèches/emoji), les suffixes qualité/langue (`HD`, `FHD`, `4K`, `UHD`, `HEVC`, `SD`, `1080p`, `MULTI`, `VF`, `VO`, codes de langue) et les articles initiaux (`le`, `la`, `les`, `l’`, `un`, `une`, `the`, `el`, `ال`…), avec décomposition Unicode NFKD et repli de casse. Le titre affiché n’est jamais modifié et cette normalisation n’entre pas dans `sourceKey` (§5.3). Prévoir une passe QA arabe avant distribution, même si le français est la langue V1.
- **Accessibilité :** `appinfo.json` active `supportsAudioGuidance`; utiliser ARIA labels/roles/state sur les contrôles personnalisés et vérifier l’annonce du focus avec l’audio guidance webOS réellement activé. Ne pas prétendre à l’accessibilité vocale sans test appareil. Focus identifiable autrement que par couleur seule ; texte redimensionnable (100/125/150 %) sans troncature critique ; poster avec alternative utile ou décorative explicitement marquée ; dialogues et erreurs compréhensibles au lecteur d’écran. [S37]

### 9.2 Budgets de performance projet

Ces nombres sont des **objectifs internes à valider**, pas des performances garanties par LG. Le modèle de référence est la TV webOS 6 de gamme la plus modeste disponible à l’équipe, raccordée à un réseau de test stable.

| Mesure | Objectif V1 |
|---|---|
| Accueil froid, hors réseau | visible rapidement avec ressources empaquetées ; cible < 3 s pour l’écran interactif au p95 sur l’appareil de référence. |
| D-pad → focus visible | cible p95 ≤ 150 ms ; aucune navigation qui attend le réseau ou le chargement d’une image. |
| Transition locale entre écrans | cible p95 ≤ 300 ms, sans recharger l’app entière. |
| Sélection d’une catégorie déjà en cache | cible ≤ 250 ms avant feedback ; le réseau remplit ensuite le contenu. |
| VOD/grilles/catalogues longs | virtualisation Sandstone ; ne monter que les éléments visibles et un petit tampon ; retenir le focus sans reconstruire tout le catalogue. [S08] |
| Lecture | afficher immédiatement l’état de préparation ; afficher l’image/nom pendant l’attente ; mesurer séparément délai fournisseur et temps jusqu’à première image. |
| Zapping live (chaîne suivante, même profil/format) | cible initiale **p50 ≤ 4 s** et p95 ≤ 8 s, mesurés depuis la **dernière** pression de la rafale jusqu’à la première image affichée, fenêtre de coalescence de 400 ms incluse ; à calibrer en phase 0 puis à figer. |
| Images | jamais attendre logo/poster pour rendre l’élément, la liste ou le focus. |
| Session longue | scénario soak de 30 min et au moins 100 changements de chaîne ; aucune croissance mémoire monotone, perte durable de réactivité, écran noir ou crash. Mesurer CPU/mémoire sur TV réelle avec les outils LG. [S21] |
| Bundle | build production minifié, imports limités et assets locaux ; fixer un budget de bundle en CI après première mesure réelle plutôt que d’ajouter des dépendances sans contrôle. |

### 9.3 Techniques obligatoires

- Utiliser Sandstone `VirtualList`/`VirtualGridList` pour les longues listes. Définir `itemSize`, `dataSize`, renderer stable, clé et `data-index` pour Spotlight ; ne pas reconstruire de fonctions/item tree à chaque mouvement de focus. Enact documente la virtualisation comme réponse aux listes longues et au coût repaint/reflow. [S08]
- Charger catégories, détail et EPG à la demande ; ne jamais récupérer film + série + live + EPG complet avant l’accueil.
- Un **seul** module de normalisation (casse, accents, NFKD, préfixes/suffixes fournisseur, articles) alimente la correspondance EPG, les clés de tri, la recherche et les tranches alphabétiques ; sa duplication par écran est interdite. Il ne modifie jamais le texte affiché et n’intervient pas dans `sourceKey` (§5.3).
- Recherche et saut alphabétique exécutés par requête indexée du service (catégories visibles, résultats paginés), jamais par filtre JS d’un tableau géant ; debounce 250 ms, annuler les requêtes obsolètes, garder l’état visible précédent jusqu’au remplacement.
- Éviter mises à jour React globales à chaque `timeupdate` vidéo ; rafraîchir la progression par cadence contrôlée ou au niveau du DOM du contrôle dédié.
- Utiliser placeholder et tailles fixes pour réduire layout shift ; ne pas animer `width`/`height` d’une grille à chaque focus.
- Limiter les couches `will-change` à l’élément réellement animé ; ne pas GPU-promouvoir toutes les cartes.
- Aucun calcul massif XMLTV/M3U sur le thread UI ; parser par lots/service et reporter l’UI entre lots si un Worker est utilisé.
- Profilage obligatoire sur téléviseur, pas seulement Chrome desktop. LG recommande Simulator/TV réelle, `ares-device-info`, Web Inspector/Node Inspector et Beanviser. [S21][S24]

---

## 10. Cycle de vie, réseau et comportement de reprise

- Au lancement sans profil : ouvrir l’accueil immédiatement ; ne pas lancer de réseau avant un choix utilisateur.
- Traiter `webOSLaunch` et `webOSRelaunch` sans réinitialiser tout l’état d’une app déjà active. [S16]
- À l’événement `visibilitychange` masqué : arrêter le préchargement, annuler les opérations non indispensables, ne pas continuer audio/vidéo en arrière-plan sans politique validée ; noter la route et l’état sûr.
- À la reprise visible : vérifier le profil actif ; rafraîchir au plus une source d’état nécessaire ; conserver focus, scroll et route si possible ; relancer EPG/catalogue uniquement si expiré.
- Si le téléviseur redémarre ou termine l’application : reprendre accueil/dernier écran par défaut, pas démarrage automatique d’un flux potentiellement coûteux ; option de reprise live uniquement après choix produit explicite.
- Pour Chromium et la pile média webOS, LG documente TLS 1.2 et TLS 1.3 sur webOS 6 ; un certificat racine absent de l’appareil peut bloquer la connexion. La pile Node 8.12 du service est distincte : baseline API TLS 1.2, TLS 1.3 seulement après validation effective sur le téléviseur. Ne jamais accepter un certificat invalide. [S28][S36]
- Le **bundle de racines publiques embarqué** par le service (§2.5) est un artefact suivi : source, date de génération et licence consignées, mise à jour contrôlée à chaque version de l’application, sans CA privée. Un écart entre le magasin du service et celui de Chromium/du média doit rester observable dans le diagnostic local, pas masqué.
- Ne pas supposer que le client `http` Node utilise HTTP/2/3 parce que le navigateur ou le pipeline média le prend en charge ; HTTP/1.1 est la baseline fournisseur du service. Tester les protocoles séparément si une source l’exige.

---

## 11. Plan de tests et critères d’acceptation

### 11.1 Tests automatisés

- Les tests Spotlight/focus reposent sur la géométrie et le layout DOM : **jsdom ne suffit pas**. Exécuter les parcours d’intégration dans un vrai navigateur, sur Chromium 79 épinglé pour le minimum, puis sur une TV réelle ; conserver les tests unitaires indépendants dans le runner léger.
- **CI des deux cibles :** tests du service sur un **Node 8.12.0 épinglé**, parcours d’interface sur **Chromium 79 épinglé**. Un test qui échoue seulement sur Node 8.12 est un bug de cible, pas un test à désactiver. Ajouter un contrôle qui échoue si le bundle du service contient une API interdite (§2.3) et vérifier qu’aucun `for await`, `fs.promises` ni Brotli n’entre par une dépendance.
- **Clés M3U :** fixtures `TF1 HD` / `TF1 FHD` / `TF1 4K`, `tvg-id` dupliqué ou faux, variantes de secours, trois lignes strictement identiques (rangs distincts) ; vérifier qu’aucune chaîne distincte n’est fusionnée, qu’un favori reste sur la bonne variante après refresh et qu’aucune réassociation ambiguë n’est automatique.
- **Curseurs et version d’index :** pagination/recherche interrompues par une bascule, lecteur sur une version retirée, reprise après redémarrage du service, import annulé n’écrasant jamais l’ancien index ; vérifier `catalog/indexChanged`, le rejeu au même ancrage et le pic disque `2 × index + temporaire`.
- **Filtrage d’URL de flux :** hôtes `10.0.0.1`, `127.0.0.1`, `169.254.169.254`, `2130706433`, `0177.0.0.1`, `[::1]`, `fe80::1` marqués puis refusés à la lecture sans autorisation LAN ; autorisation explicite par profil qui les rend jouables ; sonde MIME désactivée et non comptée quand `max_connections ≤ 1`.
- **Tranches alphabétiques et RTL :** catalogue arabe avec interface française et catalogue latin avec interface arabe produisent les tranches du contenu dans l’ordre canonique du script, avec comptes corrects ; vérifier aussi la barre de progression et le seek physique en RTL.
- Maintenir des fixtures de structure réalistes : réponses Xtream anonymisées (success, expiry, connexions, formats, erreurs et variantes de schéma), M3U de tailles/formats variés avec URLs et tokens remplacés, XMLTV bruités (fuseaux, CDATA, encodages et programmes hors fenêtre). Les fixtures de référence et leur provenance/licence sont revues avant commit ; compléter par des données synthétiques de fuzz.
- Adaptateur Spotlight : parcours flèches/OK de chaque conteneur, entrée/sortie de fiches, focus initial, focus restauré après navigation et modale.
- Retour : une pression par profondeur d’écran, fermeture de dialog, retour lecteur→catalogue, touche Back en saisie, comportement à la racine.
- Parsing M3U : BOM, LF/CRLF, attributs cités, virgule dans titre, lignes vides, groupes manquants, URLs invalides, doublons, très grande playlist, import interrompu.
- Adaptateur Xtream : réponses valides et incorrectes, champ manquant, erreur HTTP, timeout, champs chaînes/nombres, variante d’identifiant de série.
- XMLTV : début/fin de journée, fuseau horaire, programme sans stop, id absent, description CDATA, données invalides et réponse vide.
- Player : transition IDLE→BUFFERING→PLAYING, événements tardifs, erreur média, timeout, annulation, changement rapide de chaîne et retour.
- Persistance : migration de schéma, données DB8 corrompues/pleines, effacement de profil et absence de secret en clair dans les logs.

### 11.2 Matrice appareil obligatoire

- **Minimum** : une TV réelle webOS 6, modèle de gamme moyenne/faible, 1080p, Chromium 79.
- **Versions de plateforme** : webOS 6 comme minimum, puis matrice de régression webOS 22, 23, 24, 25 et 26 (Simulator/émulateur quand disponible). Au moins une TV réelle récente en plus de la TV 6 ; chaque major annoncée comme compatible passe ses scénarios de release. Si une version n’est pas testée, l’indiquer comme non certifiée plutôt que d’extrapoler.
- **Familles de modèles** : la certification média est en plus **par famille de modèles**, car les limites HEVC/1080i/débit divergent entre une entrée de gamme et un OLED de la même année. Deux téléviseurs de même version webOS mais de gammes différentes ne se couvrent pas l’un l’autre : annoncer « certifié » uniquement pour les familles testées et « non testé » ailleurs — jamais par version d’OS seule.
- **Protocole statistique** : au moins **30 tentatives indépendantes par cellule** (modèle × format × codec × forme d’URL), critère **≥ 29/30** — ou **20/20** en variante resserrée (même ordre de confiance, borne inférieure ≈ 84 %), ou **20 essais au minimum** si la cellule est coûteuse à tester, avec 20/20 exigé. Dans tous les cas : aucun échec reproductible deux fois, sinon la cellule n’est pas certifiée. Un 9/10 ou un 18/20 ne prouve rien. Consigner le nombre d’essais, les échecs et leur cause.
- **Comptes de test dédiés** pour toute boucle d’essais, avec `max_connections` et formats documentés ; jamais un compte utilisateur.
- Magic Remote pointeur et 5-way ; tester explicitement si CH+/CH− livrent des keycodes sur chaque télécommande physique. Télécommande conventionnelle avec touches média si disponible ; clavier virtuel et entrée URL.
- Réseau normal, latence haute, DNS erroné, perte/reprise Wi-Fi, portail indisponible, HTTP et HTTPS, API/flux TLS 1.2 et TLS 1.3-only, certificat racine invalide/absent ; tester séparément service Node, navigateur et pipeline `<video>`. **Cas obligatoire :** un hôte dont la chaîne aboutit à une racine postérieure au build du Node embarqué (par ex. ISRG Root X2, 2020) — vérifier si le magasin de 2018 échoue, que le bundle de racines publiques embarqué corrige le service, et consigner le résultat par pile. [S39]
- Matrice de flux autorisés : HLS, MPEG-TS progressif; URL sans extension avec `Content-Type` correct, absent ou générique ; redirects cross-host ; H.264, HEVC, MPEG-2, AAC, AC-3, E-AC-3, 1080i et cas non supportés. Tester HTTP clair avec avertissement une fois par profil.
- Catalogues de capacité : fixtures synthétiques de 250 000 entrées et 256 Mio décompressés, 1 000 catégories, catégories sélectionnées, images absentes, logos lourds, recherche, import interrompu/repris et quota disque ; mesurer le **pic disque en incluant l’ancien index conservé et les fichiers temporaires** (≈ 2 × index + entrée) ; EPG XMLTV volumineux/filtré.
- Mesure mémoire définie : après 10 min de chauffe, relever le RSS de l’app et du service toutes les 30 s ; exécuter 100 zappings puis 5 min au repos. Objectif initial par processus : pic stabilisé ≤ baseline + max(15 % de la baseline, 40 Mio), sans crash ; p95 D-pad reste ≤150 ms. Toute pente RSS >1 Mio/min sur les 5 dernières minutes est un no-go jusqu’à analyse. Conserver heap JS et RSS séparés dans le rapport.
- Session live longue, Home/retour, veille/réveil si scénario utilisé ; mesurer CPU/mémoire séparément en navigation et lecture.

Utiliser le **webOS TV Simulator 6.0** pour accélérer layout, états et navigation ; il est disponible pour webOS 6.0. L’ancien Emulator 6.0 est déprécié, mais reste installable si nécessaire. Ni l’un ni l’autre ne remplace la validation média/codec/DRM/performance sur TV : LG signale que le Simulator a des capacités audio/vidéo différentes et que certaines API/DRM sont absentes ; sur Emulator 6.0, les keycodes des touches média de la télécommande conventionnelle sont mal mappés. [S24]

### 11.3 Scénarios d’acceptation essentiels

1. **Premier lancement hors ligne** : accueil et quatre boutons visibles ; aucune erreur bloquante ; Profil accessible.
2. **Ajout Xtream** : saisie via clavier virtuel ; consentement explicite avant mémorisation ; afficher expiration/connexions/formats uniquement si l’API les expose ; entrer dans Live.
3. **Ajout M3U** : afficher groupes, taille/progression, état EPG et options de groupes ; indexer un grand catalogue sans le charger en DB8/RAM ; le service peut redémarrer puis renvoyer une page ; quota atteint ne détruit jamais l’ancien index.
4. **Live** : catégories/chaînes adjacentes, EPG courant/suivant pleine hauteur en V1, sans panneau vide ; zapping coalescé/flux précédent fermé ; Back revient à la dernière chaîne réellement lue et synchronise catégorie, scroll et focus.
5. **VOD** : catégorie, recherche globale, saut alphabétique, grille d’affiches modérée, fiche et lecture ; focus/scroll/query restaurés.
6. **Séries** : catégorie, recherche globale, saut alphabétique, série/saison/épisode et lecture ; champs manquants et M3U non structurée restent récupérables.
7. **EPG** : ID exact, suggestion unique par nom, ambiguïté présentée à l’utilisateur, correction manuelle persistante ; décalage manuel Xtream vérifiable sans changer XMLTV.
8. **Flux KO** : URL brisée, URL sans extension/MIME inconnu, redirect cross-host, flux expiré, serveur lent, HTTP clair, manifeste/codec non compatible, TLS service/media distinct et réseau coupé donnent des états séparés ; aucun retry infini.
9. **Limite fournisseur** : afficher connexions actives/max lorsque présentes ; erreur de limite spécifique ; 10 pressions Haut/Bas rapides ouvrent au plus le dernier flux demandé après arrêt confirmé de l’ancien.
10. **Télécommande seulement** : profil, recherche, lecture, catégories masquées, retour et suppression réalisables au D-pad ; vérifier CH+/CH− sans en dépendre.
11. **Confidentialité** : consentement de mémorisation, avertissement HTTP une seule fois par profil/host, aucun secret dans logs/diagnostics ; diagnostic uniquement local avec export manuel ; suppression efface DB8 et fichiers service.
12. **Accessibilité/localisation** : noms accessibles/ARIA, audio guidance activé et testé sur TV ; vérification RTL arabe, focus miroir et tailles de texte sans coupure.
13. **Fluidité** : objectifs p95, p50 de zapping et seuils mémoire du §11.2 mesurés sur la TV de référence ; un échec est un no-go à corriger avant extension de compatibilité.
14. **Consentement M3U** : refuser la persistance explique pourquoi aucun import ne démarre et propose Xtream ou l’abandon ; accepter écrit un index chiffré, relisible après redémarrage du service ; l’option « préfixe d’identifiants commun » redemande l’URL de playlist à la session suivante au lieu de stocker les URL complètes.
15. **Zapping hors catégorie** : lancer depuis Favoris et depuis une recherche globale, vérifier la file réellement suivie (liste affichée, pas de saut de catégorie) et le retour exact à l’écran et à l’état d’origine.
16. **Seek précis** : overlay visible, barre de progression focalisée, Gauche/Droite ajustent par pas bornés et OK reprend à la position choisie, y compris en RTL.
17. **Index remplacé** : lancer une recherche, déclencher un rafraîchissement pendant la pagination ; vérifier « catalogue mis à jour », le rejeu automatique au même ancrage et la conservation du focus.
18. **Hôte de flux privé** : une playlist contenant `http://192.168.1.1/...` ou une IP en notation décimale/octale est refusée à la lecture sans autorisation LAN du profil, puis jouable après autorisation explicite.

---

## 12. Séquencement recommandé

### Phase 0 — Prototype de risques avec critères go/no-go

- **Contrôle de distribution — go/no-go bloquant, avant tout développement lourd.** Relire la Self Checklist, le processus d’approbation (pretest, function test, content test) et la Privacy Guideline LG en vigueur, puis poser **par écrit** à LG Seller Lounge la question du positionnement : application qui lit des playlists/portails **fournis par l’utilisateur**, sans aucun contenu ni chaîne fourni, sans contournement de DRM, sans proxy ni SDK tiers utilisant les ressources de l’appareil. Récupérer : acceptabilité du modèle, mentions obligatoires (page confidentialité, « cette application ne fournit aucun contenu »), pays de distribution autorisés, exigences de revue et calendrier. Vérifier les applications comparables déjà publiées sur le Content Store. Un refus ou une condition inacceptable n’a **pas de repli silencieux** : il change le produit ou le mode de distribution, donc il se tranche ici. [S20][S29]
- Épingler le socle CI : **Chromium 79** pour l’interface et **Node 8.12.0** pour le service, avec le contrôle d’API interdites (§2.3).
- Installer le paquet Enact 3.4.9/Sandstone 1.4.6 sur une TV réelle webOS 6 moyenne/faible ; valider Spotlight (D-pad, pointeur, limites de focus, clavier virtuel, Back 461/history) et exécuter les mêmes scénarios dans Chromium 79.
- Utiliser au minimum **deux portails Xtream autorisés indépendants et deux playlists M3U autorisées**, anonymisables pour fixture, avec des **comptes de test dédiés** pour toutes les boucles d’essais (`max_connections` et formats documentés). Vérifier `player_api`, l’appel « tous les flux » et son repli par catégorie, compte/format, liste M3U, redémarrage service, import annulé, écriture/reprise de l’index privé, pagination, recherche et bascule d’index pendant une pagination.
- Matrice media obligatoire avec `<video>` natif : HLS et MPEG-TS progressif; URL sans extension avec MIME déclaré, absent et générique; redirect 302 vers même/autre hôte; HTTP clair et HTTPS; H.264, HEVC, MPEG-2, AAC, AC-3, E-AC-3 et 1080i. Ne conclure qu’à partir du modèle réel, pas d’un flux HLS public unique ; les essais sont menés **30 fois par cellule** selon le protocole §11.2 et sur **familles de modèles distinguées** (une seule TV webOS 6 ne représente pas toutes les limites HEVC).
- TLS par pile : capturer `process.versions.openssl` ; tester une API TLS 1.2, TLS 1.3-only et une chaîne à **racine récente** (postérieure au build du Node embarqué, par ex. ISRG Root X2) depuis Node 8.12 ; comparer magasin CA/validation du service au navigateur et au player, puis valider le bundle de racines publiques embarqué. Vérifier DNS/redirects et refus de loopback/IP privée (y compris notations décimales/octales), puis exception LAN explicitement autorisée.
- Importer un catalogue de 250 000 entrées/256 Mio, mesurer temps, RSS/heap, espace et annulation, **pic disque en comptant l’ancien index conservé** ; vérifier le chemin « choisir des groupes localement » quand le quota de sécurité est atteint.
- Modèle UX secrets : vérifier le consentement « Mémoriser ce profil » (Xtream) et le **prérequis affiché pour une source M3U**, le parcours Xtream sans mémorisation, l’écriture chiffrée de l’index et sa relecture après redémarrage du service, la suppression complète et l’alerte HTTP une fois par profil. Mesurer le coût du chiffrement par blocs sur la TV de référence ; Keymanager3 n’est pas dans le périmètre V1 (§8.2).

**GO V1 media :** sur chaque téléviseur de référence et pour chaque cellule (modèle × format × codec) que le produit annonce compatible, **30 tentatives indépendantes avec ≥ 29 premières images** — ou **20/20** en variante, mêmes règles §11.2 ; p95 première image ≤ 15 s **et p50 de zapping ≤ 4 s** sur réseau de labo stable ; le flux précédent est fermé avant le suivant ; aucune erreur TLS/CORS non documentée ni crash. Une cellule qui échoue perd son label « certifié » pour la famille de modèles concernée ; l’appartenance à une famille certifiée est publiée. **GO UI :** navigation complète D-pad, focus stable, p95 focus ≤150 ms et mémoire dans les seuils §11.2. **GO distribution :** réponse écrite de LG obtenue et conditions acceptables (§12, phase 0) — sinon le GO est suspendu, pas contourné.

**Repli explicite :** si le player natif échoue, ne pas changer de framework pour le résoudre ; borner les formats compatibles et évaluer séparément une autre intégration média, après preuve sur TV. Si seul Enact/DOM échoue les budgets UI sur les TVs visées, lancer un prototype **Lightning/Blits isolé** et le comparer aux mêmes métriques, sans mélanger les frameworks dans la V1. Ne pas contourner TLS/CORS ni attribuer à un nouveau framework un problème de pipeline média.

### Phase 1 — V1 fonctionnelle

- Squelette empaqueté, thème, quatre cartes, adaptateur remote, routeur/Retour.
- Profil Xtream et URL M3U ; consentement stockage secret, import/index paginé, catégories masquables/réordonnables et persistance stable.
- Live + lecteur + erreurs ; zapping coalescé, EPG courant/suivant, correspondance et décalage manuel.
- VOD/recherche globale/saut alphabétique/posters/fiche ; Séries/recherche/saison/épisodes.
- Favoris/reprises réassociables après refresh, cycle de vie, diagnostic local/export manuel, logs sans secret.

### Phase 2 — Durcissement et distribution

- Tests multi-modèles, performance/soak, sécurité et privacy ; corriger les écarts.
- Finaliser iconographie, langues, guide d’utilisation, politique de confidentialité et données de distribution.
- Compléter le self-checklist LG et préparer les captures/scénarios de validation demandés. La faisabilité du modèle ayant été tranchée en phase 0, cette phase ne décide plus de la distribution : elle la met en forme (checklist remplie et **téléversée** avec la soumission — son absence ou son imprécision entraîne un rejet sans QA —, métadonnées, politique de confidentialité, signalement des SDK tiers et section data safety). LG publie un processus de revue et un self-checklist ; vérifier les exigences à jour au moment de soumettre. [S20][S21][S29]

---

## 13. Sources de recherche

Sources consultées le **28 septembre 2026**. Les spécifications TV, versions d’Enact, exigences de distribution et APIs évoluent : vérifier ces pages à nouveau avant de figer le SDK ou soumettre l’application.

- **[S01] LG — Web API et versions du moteur Chromium** : [Web API and Web Engine](https://webostv.developer.lge.com/develop/specifications/web-api-and-web-engine)
- **[S02] LG — versions Enact/Sandstone prises en charge par génération** : [Enyo and Enact Guide](https://webostv.developer.lge.com/develop/guides/enyo-enact-guide)
- **[S03] Enact — migration 4.0 et abandon des TV 2021 et antérieures** : [Migrating to Enact 4.0](https://enactjs.com/docs/developer-guide/migration/enact/migrating-to-enact-4/)
- **[S04] Enact — TypeScript** : [TypeScript with Enact](https://enactjs.com/docs/tutorials/tutorial-typescript/typescript-overview/)
- **[S05] Enact — Spotlight, navigation 5-way et pointeur** : [Spotlight](https://enactjs.com/docs/developer-guide/spotlight/)
- **[S06] LG — Magic Remote, modes pointeur/5-way et keycodes** : [Magic Remote](https://webostv.developer.lge.com/develop/guides/magic-remote)
- **[S07] LG — Back, History API et keycode 461** : [Back Button](https://webostv.developer.lge.com/develop/guides/back-button)
- **[S08] Enact — virtualisation des listes et grilles** : [VirtualList, VirtualGridList and Scroller](https://enactjs.com/docs/developer-guide/virtual-list-scroller/) et [Performance Guide](https://enactjs.com/docs/developer-guide/performance/)
- **[S09] LG — distinction entre application empaquetée et application hébergée** : [Web App Types](https://webostv.developer.lge.com/develop/getting-started/web-app-types)
- **[S10] Enact — build de production, bundle et minification** : [Building Apps](https://enactjs.com/docs/developer-tools/cli/building-apps/)
- **[S11] LG — services JS et versions Node.js par webOS** : [JavaScript Service Basics](https://webostv.developer.lge.com/develop/guides/js-service-basics)
- **[S12] LG — politique CORS côté serveur** : [How to solve CORS](https://webostv.developer.lge.com/faq/2022-07-28-how-to-solve-the-problem-cors)
- **[S13] Forum officiel webOS — CORS d’une app empaquetée** : [CORS with a non-hosted app](https://forum.webostv.developer.lge.com/t/how-are-we-supposted-to-handle-cors-with-a-non-hosted-app/28241)
- **[S14] LG — protocoles de flux, HLS, DRM et limites média** : [Streaming Protocol and DRM](https://webostv.developer.lge.com/develop/specifications/streaming-protocol-drm)
- **[S15] LG — formats et contraintes AV webOS TV 6.0** : [Audio and Video Format on webOS TV 6.0](https://webostv.developer.lge.com/develop/specifications/video-audio-60)
- **[S16] LG — cycle de vie, relance et visibilité** : [App Lifecycle Management](https://webostv.developer.lge.com/develop/guides/app-lifecycle-management)
- **[S17] LG — DB8 et persistance** : [Database API](https://webostv.developer.lge.com/develop/references/database) et [DB8 Basics](https://webostv.developer.lge.com/develop/guides/db8-basic)
- **[S18] LG — Keymanager3, chiffrement TEE disponible depuis webOS 24** : [Keymanager3](https://webostv.developer.lge.com/develop/references/keymanager3)
- **[S19] LG — suppression possible du stockage local d’une app empaquetée** : [Web Storage data removal](https://webostv.developer.lge.com/faq/2015-11-24-web-storage-local-storage-data-removal-app-removal-or-uninstallation)
- **[S20] LG — confidentialité, chiffrement des données sensibles et SDK tiers** : [Privacy Guideline](https://webostv.developer.lge.com/develop/guides/privacy-guideline)
- **[S21] LG — outils, tests TV, ressources et workflow** : [Developer Workflow](https://webostv.developer.lge.com/develop/getting-started/developer-workflow)
- **[S22] LG — clavier virtuel, champs URL/mot de passe** : [Virtual Keyboard](https://webostv.developer.lge.com/develop/guides/virtual-keyboard)
- **[S23] LG — principes de conception TV** : [Design Principles](https://webostv.developer.lge.com/develop/guides/design-principles)
- **[S24] LG — Simulator/Emulator, disponibilité de webOS 6.0 et limites TV** : [Simulator Introduction](https://webostv.developer.lge.com/develop/tools/simulator-introduction), [Simulator Installation](https://webostv.developer.lge.com/develop/tools/simulator-installation) et [Emulator Installation](https://webostv.developer.lge.com/develop/tools/emulator-installation)
- **[S25] XMLTV — structure canal/programme et DTD** : [XMLTV DTD](https://github.com/XMLTV/xmltv/blob/master/xmltv.dtd) ; pour un résumé des champs de programme : [Wurl XMLTV EPG Format](https://support.wurl.com/hc/en-us/articles/52260113100051-XMLTV-EPG-Format)
- **[S26] Norigin — Spatial Navigation et compatibilité TV web** : [Norigin Spatial Navigation](https://devportal.noriginmedia.com/docs/Norigin-Spatial-Navigation/)
- **[S27] Lightning — framework TV Lightning 3 / Blits** : [LightningJS](https://lightningjs.io/)
- **[S28] LG — compatibilité TLS et certificats racines** : [TLS and Root Certificates](https://webostv.developer.lge.com/develop/specifications/tls)
- **[S29] LG — processus d’approbation et self-checklist** : [App Approval Process](https://webostv.developer.lge.com/distribute/app-approval-process)
- **[S30] OWASP — paramètres recommandés pour PBKDF2-HMAC-SHA-256** : [Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- **[S31] OWASP — pratiques de stockage cryptographique et gestion des clés** : [Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- **[S32] LG — cycle de vie, délai d’inactivité et activités des services JS** : [JavaScript Service FAQ](https://webostv.developer.lge.com/develop/guides/js-service-faq)
- **[S33] LG — stockage privé `/media/internal`, jail et durée de vie du service** : [JS Service Usage](https://webostv.developer.lge.com/develop/guides/js-service-usage)
- **[S34] LG — exposition publique des méthodes LS2 (`public:false` par défaut)** : [services.json](https://webostv.developer.lge.com/develop/references/services-json)
- **[S35] LG — MIME et `mediaOption` pour le lecteur natif** : [mediaOption Parameters](https://webostv.developer.lge.com/develop/guides/mediaoption-parameter)
- **[S36] Node.js — runtime Node 8, API disponibles, OpenSSL embarqué et version exposée par `process.versions`** : [Process API v8](https://nodejs.org/docs/latest-v8.x/api/process.html#process_process_versions), [fs v8](https://nodejs.org/docs/latest-v8.x/api/fs.html), [zlib v8](https://nodejs.org/docs/latest-v8.x/api/zlib.html), [Node.js 8.12.0](https://nodejs.org/en/blog/release/v8.12.0) et [en-tête OpenSSL du tag v8.12.0 (1.0.2p)](https://github.com/nodejs/node/blob/v8.12.0/deps/openssl/openssl/crypto/opensslv.h) — vérifie l’absence de `fs.promises` (Node 10.1), `for await`/`stream.pipeline` (Node 10.0), Brotli (Node 10.16) et `worker_threads` en 8.12.
- **[S37] LG — `supportsAudioGuidance` et ARIA dans `appinfo.json`** : [appinfo.json](https://webostv.developer.lge.com/develop/references/appinfo-json)
- **[S38] Xtream Codes — inventaire des actions `player_api.php`** (référence communautaire, non officielle) : `get_live_streams`, `get_vod_streams`, `get_series` sans filtre de catégorie, variantes `*_by_category`, `get_series_info` pour saisons/épisodes : [Xtream Codes API reference](https://xtreamiptv.codes/xtream-codes/api/)
- **[S39] Let’s Encrypt — chaînes d’émission et racines récentes** (ISRG Root X2, 2020 ; chaîne par défaut vers ISRG Root X1 depuis juin 2024) : [Changes to issuance chains](https://letsencrypt.org/2024/04/12/changes-to-issuance-chains.html) et [Policy on issuance chain changes](https://community.letsencrypt.org/t/policy-on-issuance-chain-changes/223073)

---

## 14. Décision finale en une phrase

Pour une V1 LG webOS 6+, partir sur **application empaquetée TypeScript + Enact 3.4.9/Sandstone 1.4.6 + Spotlight** et un **service JS webOS local** compilé pour **Node 8.12/ES2017**, index catalogue M3U/Xtream compact sur fichiers privés **chiffré au repos**, **DB8 séparé pour profils/favoris/reprises** avec mémorisation des secrets sur consentement explicite — **prérequis affiché pour une source M3U**, option pour Xtream — sans prétention de coffre matériel, puis un **unique `<video>` HTML natif** pour les formats validés au go/no-go ; recherche et catalogues paginés avec curseurs versionnés, EPG pleine hauteur en V1, filtrage best-effort des URL de flux privées, et compatibilité annoncée seulement après mesures sur TVs réelles, **sous réserve du go/no-go de distribution LG en phase 0**.

---

## Annexe A — Journal de révision 1.2

| Point de revue | Vérification | Correction apportée |
|---|---|---|
| 1. Consentement M3U inapplicable | Confirmé : dans une playlist M3U, l’URL de lecture est le secret ; LG exige des identifiants « stockés dans un espace sûr ou chiffrés ». | Persistance **obligatoire et affichée** comme prérequis pour M3U (§3.5, §8.2) ; variante « index sans segment d’identifiants » lorsque le préfixe est commun (§5.2) ; index **chiffré AES-256-GCM par blocs** (§8.2). |
| 2. Pas de cible de compilation du service | Confirmé : `fs.promises` (10.1), `for await` (10.0), Brotli (10.16), `worker_threads` absents en 8.12. | Cible **ES2017 / Node 8.12** explicite, liste d’API interdites, deux lint, **CI sur Node 8.12.0 épinglé** (§2.3, §11.1, §12) ; risque EOL documenté. |
| 3. Indexation Xtream par catégorie | Confirmé : les appels « tous les flux » sans filtre de catégorie existent ; le repli par catégorie reste possible mais coûteux (429, durée) et incompatible avec l’absence de travail en arrière-plan. | Appel global par défaut, par catégorie en repli ; indexation **attachée à la souscription au premier plan**, reprise par lots ; portée de recherche précisée (titres de séries, pas d’épisodes) (§5.1, §3.4). |
| 4. Collisions de clés M3U | Confirmé : une clé sur nom sans suffixes qualité fusionne des variantes distinctes. | Clé composite **stricte** (`contentType` + `tvg-id` + nom complet + groupe + rang d’occurrence) ; clé assouplie réservée à la **réassociation** (§5.3, §11.1). |
| 5. Sonde MIME vs connexions | Confirmé : une sonde consomme une connexion et peut déclencher `max_connections` ou un blocage. | Métadonnées d’abord ; sonde **interdite si `max_connections ≤ 1`**, unique par compte, comptée, fermée avant `play()` ; essais de phase 0 sur **comptes dédiés** (§6.1, §11.2, §12). |
| 6. Curseurs et bascule atomique | Confirmé : un offset nu devient faux après bascule ; la rétention de l’ancien index double le besoin disque. | `indexVersion` obligatoire, erreur `catalog/indexChanged` + rejeu au même ancrage ; rétention bornée et quota calculé sur **≈ 2 × index + temporaire** (§2.4, §7.2, §11.2). |
| 7. Keymanager3 sans menace couverte | Confirmé : API TEE **webOS 24+ uniquement**, non disponible avant, hors simulateur. | **Reporté hors V1**, à traiter avec l’option passphrase ; modèle de menace inchangé, chiffrement logiciel assumé et divulgué (§8.1, §8.2). |
| 8. URL de flux non filtrées | Confirmé : la politique réseau couvre les téléchargements, pas `<video>`. | **Marquage à l’indexation** des hôtes IP littéraux privés/loopback/link-local (notations décimale, octale, hexadécimale, IPv6) et contrôle avant `video.src`, sauf autorisation LAN du profil ; noms d’hôte = risque résiduel assumé (§5.2, §6.1, §7.2). |
| 9. Saut alphabétique et RTL | Confirmé : les tranches suivaient la locale de l’interface ; préfixes fournisseur et articles polluaient le tri. | Tranches issues des **scripts présents dans les données** ; **normalisation partagée** avec l’EPG ; règles de focus réécrites en **termes logiques** et décision explicite sur le seek physique du lecteur (§4.3, §4.6, §9.1, §9.3). |
| 10. Trous du lecteur | Confirmé : aucune file définie pour favoris/recherche ; aucun seek précis overlay visible. | **File de lecture explicite** par contexte (catégorie, favoris, résultats, saison, grille) et **barre de progression focalisable** (slider) pour le seek (§4.3, §4.6, §6.3, §11.3). |
| 11. Go/no-go statistiquement faible | Confirmé : 9/10 est compatible avec un taux réel proche de 60 %. | **30 tentatives par cellule avec ≥ 29/30, ou 20/20** (même ordre de confiance) ; **p50 de zapping ≤ 4 s** ajouté ; certification **par famille de modèles** et non par version d’OS seule (§0, §9.2, §11.2, §12). |
| 12. Risque de distribution tardif | Confirmé : le processus LG (pretest/function/content, self-checklist à téléverser) conditionne tout le produit. | **Go/no-go de phase 0** avec question écrite à LG Seller Lounge, mentions obligatoires et pays de distribution (§12 ; §0). |
| Dernier détail — magasin de certificats | Confirmé : bundle figé au build ; chaînes récentes (ISRG Root X2, 2020) peuvent échouer côté service alors que le média réussit. | Cas de test **racine récente** obligatoire en phase 0 et **bundle de racines publiques embarqué** pour le client HTTPS du service, sans contournement TLS (§2.5, §10, §11.2). |
