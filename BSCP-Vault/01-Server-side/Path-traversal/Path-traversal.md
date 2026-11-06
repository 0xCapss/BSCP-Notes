---
tags: [bscp, server-side, path-traversal]
niveau: apprentice
statut: en cours
---
# Path traversal

## En bref
- Le path traversal (aussi appelé directory traversal) est une vulnérabilité web qui permet de lire des fichiers arbitraires sur le serveur qui héberge l'application, voire d'en écrire dans certains cas (voir [[File-upload]]).
- Ces fichiers peuvent contenir du code applicatif, des identifiants de back-end ou des fichiers système sensibles.
- Terme utilisé dans cette note : path traversal. Il diffère de l'inclusion de fichier locale (LFI) : le path traversal lit le contenu d'un fichier, alors que l'inclusion locale fait interpréter le fichier par le moteur de l'application (par exemple `include` en PHP). Les deux partent du même défaut : une entrée utilisateur utilisée comme chemin.

## Mécanisme
- L'application construit un chemin en concaténant un répertoire de base et une valeur fournie par l'utilisateur, par exemple `/var/www/images/` et `53.jpg`.
- Le système de fichiers résout ensuite les séquences `../` (remonter d'un dossier) : avec `../../../etc/passwd`, le chemin final sort du répertoire de base et vise `/etc/passwd`.
- Des séquences `../` en trop sont sans effet à la racine : on peut donc en mettre plus que nécessaire pour atteindre la racine (le lab 1 en a utilisé quatre, voir le journal).
- Exemple détaillé : [[Path-traversal-cas-simple]].

## Décodage en couches
- Une valeur traverse souvent plusieurs couches (navigateur, proxy inverse, serveur web, framework, application), et chacune peut décoder l'URL ou normaliser le chemin.
- Un filtre placé à un niveau voit la valeur dans l'état où elle est à ce niveau ; une couche située après lui peut décoder encore. C'est ce décalage que les encodages exploitent.
- Conséquence pour le choix du payload : un seul décodage après le filtre appelle `%2e%2e%2f`, deux décodages appellent `%252e%252e%252f`. Pour savoir lequel, comparer la réaction à la forme brute, à la forme simple et à la forme double.
- Une validation correcte se fait après le dernier décodage, jamais avant.

## Types et variantes
- Lecture de fichier arbitraire (le cas le plus courant) contre écriture de fichier arbitraire (via upload, export, sauvegarde de configuration ou extraction d'archive), cette dernière ayant un impact potentiellement plus critique (exécution de code).
- Path traversal non filtré (séquence simple suffisante) contre path traversal filtré, nécessitant une technique de bypass.
- Conventions de chemin : Unix (`../`) contre Windows (`..\` et en général `../` valides toutes les deux, selon le serveur).

| Type | Idée clé | Note | Niveau |
| --- | --- | --- | --- |
| Cas simple | `../../../etc/passwd` | [[Path-traversal-cas-simple]] | Apprentice |
| Chemin absolu | `filename=/etc/passwd` | [[Path-traversal-chemin-absolu]] | Practitioner |
| Séquences imbriquées et encodage | `....//`, `%252e%252e%252f` | [[Path-traversal-sequences-et-encodage]] | Practitioner |
| Préfixe et extension | préfixe imposé, `%00.png` | [[Path-traversal-prefixe-et-extension]] | Practitioner |

## Comment détecter
- Repérer les paramètres ou fonctionnalités qui manipulent un nom de fichier ou un chemin (paramètres nommés filename, file, path, doc, template, page, image, download, include, ou valeurs contenant une extension de fichier).
- Établir d'abord une référence (baseline) : relever la requête normale (valeur légitime), puis une valeur inexistante, puis la valeur de path traversal, et comparer code de statut, taille, type de contenu, message et temps de réponse.
- Exemple de comportement observé sur le lab 1 : pour un fichier existant, le serveur répond HTTP 200 ; pour un fichier inexistant, HTTP 400 avec le message "No such file". Le `Content-Type` peut rester celui d'une image alors que le corps est du texte : il faut lire le corps.
- Injecter une séquence simple visant un fichier connu et observable (`../../../etc/passwd` sur Linux, `..\..\..\windows\win.ini` sur Windows) et regarder si le contenu du fichier apparaît dans la réponse.
- Chercher la profondeur : ajouter des `../` jusqu'à obtenir le contenu ; les séquences en trop restent sans effet, donc partir large.
- Si le retour n'est pas directement visible, chercher des signaux indirects : différence de code de statut, de message d'erreur, de taille ou de temps de réponse entre un chemin existant et un chemin inexistant.
- Si la séquence simple est filtrée, tester méthodiquement chaque technique de bypass (chemin absolu, séquences imbriquées, encodage simple, double encodage, préfixe, octet nul) pour identifier précisément le filtre en place.
- Lieux d'injection à examiner, pas seulement le paramètre de requête : segment de chemin d'URL, corps de formulaire ou JSON, champ `filename` d'un envoi multipart, header, cookie.
- Outillage : un navigateur ou certains clients normalisent `../` avant l'envoi. Utiliser Burp Repeater, ou curl avec `--path-as-is`, et contrôler la requête brute réellement envoyée, car la saisie peut ré-encoder `%25`.
- Automatiser avec Burp Intruder et la liste de payloads "Fuzzing - path traversal" sur les paramètres suspects.
- Ne pas se limiter aux fonctionnalités de lecture : examiner aussi les fonctionnalités d'upload, d'export ou de sauvegarde de configuration (principe de l'écriture : [[File-upload]]). Pour confirmer sans risque, écrire un fichier inoffensif et reconnaissable dans un emplacement autorisé par le périmètre, puis le relire.

## Comment exploiter (principe)
1. Identifier le paramètre qui désigne un fichier.
2. Tester `../../../etc/passwd` (ou `..\..\..\windows\win.ini`) et lire le corps de la réponse.
3. Si la séquence est bloquée, essayer dans l'ordre : chemin absolu en premier (le plus rapide, voir [[Path-traversal-chemin-absolu]]), puis séquences imbriquées, encodage simple puis double, préfixe imposé, octet nul avant l'extension en dernier.
4. Confirmer la lecture avec un fichier connu, puis cibler des fichiers utiles (configuration, code source, clés).

Le détail de chaque technique est dans les notes de la table ci-dessus.

## Catalogue de payloads par variante
Les exemples supposent un paramètre `filename=<valeur>` et un répertoire de base de type `/var/www/images/`. Les listes courtes à copier sont dans [[Payloads-cheatsheet]].

| Variante | Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- | --- |
| Séquences simples | `../../../etc/passwd` | Aucune validation | Lignes `root:x:0:0:` dans le corps | Échoue si filtre, préfixe ou extension imposés. Quatre `../` ont suffi sur le lab 1 |
| Chemin absolu | `/etc/passwd` (Windows : `C:\windows\win.ini`) | `../` bloqué, valeur utilisée telle quelle | Contenu du fichier alors que `../` est refusé | Échoue si la base est toujours préfixée (`/var/www/images//etc/passwd`). |
| Séquences imbriquées | `....//....//....//etc/passwd`, `....\/....\/....\/etc/passwd`, `..././..././..././etc/passwd` | Un seul passage de suppression de `../` | Contenu du fichier, la forme simple échouant | Inefficace si la suppression est récursive. |
| Encodage simple | `..%2f..%2f..%2fetc/passwd`, `%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd` | Filtre avant un décodage | Forme brute refusée, forme encodée acceptée | L'outil d'envoi peut ré-encoder. |
| Double encodage | `..%252f..%252f..%252fetc/passwd`, `%252e%252e%252f...` | Filtre suivi de deux décodages | Idem, après le second décodage | Encoder seulement le `/` est souvent plus fiable. |
| Encodage non standard | `..%c0%af..%c0%af..%c0%afetc/passwd`, `..%ef%bc%8f..%ef%bc%8f..%ef%bc%8fetc/passwd` | Décodeur ancien ou normalisation Unicode | Contenu du fichier | Rare sur les piles récentes. |
| Préfixe imposé | `/var/www/images/../../../etc/passwd` | Valeur doit commencer par la base, sans canonisation | Contenu du fichier, `/etc/passwd` seul étant refusé | Reprendre la base exacte vue en requête normale. Échoue si canonisation avant comparaison. |
| Extension imposée | `../../../etc/passwd%00.png` (extension déjà vue en requête normale) | Contrôle d'extension puis API qui s'arrête à l'octet nul | Fichier lu malgré `.png` | Ne fonctionne plus sur les piles récentes ; à essayer en dernier. |
| Windows, antislash | `..\..\..\windows\win.ini`, `..%5c..%5c..%5cwindows%5cwin.ini`, `..%255c..%255c..%255cwindows%255cwin.ini` | Serveur acceptant `\` ; filtre ne traitant que `/` | Contenu de win.ini (sections `[fonts]`, `[extensions]`) | Sur Linux, `\` est un caractère ordinaire : un échec ne prouve rien. |
| Combinaisons | `/var/www/images/....//....//....//etc/passwd` | Préfixe imposé et suppression non récursive | Contenu du fichier | Cumule les limites du préfixe imposé et de la suppression non récursive |

### Fichiers cibles (selon les droits du processus)
- Linux, preuve de lecture : `/etc/passwd`, `/etc/hostname`, `/etc/os-release`, `/proc/version`.
- Linux, contexte du processus : `/proc/self/environ`, `/proc/self/cmdline`.
- Linux, configuration : `/etc/nginx/nginx.conf`, `/etc/apache2/apache2.conf`, fichiers de l'application (`.env`, `config.php`, `settings.py`, `web.xml`, `application.properties`) dont le chemin est à déduire.
- Linux, identifiants : `/home/<utilisateur>/.ssh/id_rsa` ; `/etc/shadow` est en général illisible sans privilèges élevés.
- Windows : `C:\windows\win.ini`, `C:\windows\system32\drivers\etc\hosts`, `C:\inetpub\wwwroot\web.config`.

## Limites
- L'octet nul ne fonctionne plus sur les piles récentes.
- Les droits du compte du processus bornent ce qui est lisible ; un conteneur ou un chroot réduit les fichiers présents.
- Le path traversal lit un fichier, pas un listing de répertoire ; un fichier binaire peut être illisible dans un affichage texte.
- Un échec d'un payload ne prouve pas l'absence de vulnérabilité : le fichier peut être absent ou le filtre différent.
- D'autres chemins de lecture de fichiers existent avec le schéma `file://` : voir [[SSRF]] et [[XXE-injection]].

## Pièges et points d'attention BSCP
- Distinguer un simple filtrage de la sous-chaîne `../` (contournable par séquences imbriquées) d'une validation par canonicalisation robuste (résolution du chemin absolu puis vérification qu'il reste dans le répertoire autorisé), beaucoup plus difficile à contourner.
- Ne pas s'arrêter au premier payload qui échoue : les filtres peuvent être empilés (décodage, suppression de séquences, vérification d'extension), chacun nécessitant sa technique.
- Sur un contrôle du début du chemin, tester un chemin absolu puis le préfixe attendu suivi de séquences ; sur un contrôle d'extension, l'octet nul avant l'extension attendue.
- Si l'environnement est inconnu, tester les deux conventions de chemin (Unix et Windows).
- Faux positifs : un message d'erreur générique ne prouve pas la lecture ; exiger le contenu du fichier ou une différence de comportement mesurable et reproductible.
- Dans une chaîne d'attaque, la lecture de configuration, d'identifiants ou de code source alimente les étapes suivantes, voir [[Information-disclosure]].

## Prévention
- Éviter de transmettre des entrées utilisateur directement à une API de système de fichiers ; privilégier un mapping indirect (identifiant ou clé associé côté serveur à un chemin fixe, sans jamais exposer le chemin réel à l'utilisateur).
- Si un nom de fichier doit néanmoins être accepté depuis l'utilisateur, le valider strictement par allowlist : extension autorisée, absence de séparateurs de chemin, correspondance à un motif ancré.
- À défaut d'allowlist stricte, n'accepter que des caractères autorisés (alphanumériques, point, tiret) ; c'est un minimum, pas un équivalent de l'allowlist.
- Décoder une seule fois, puis valider après le dernier décodage, jamais avant.
- Après validation, concaténer au répertoire de base, canoniser le chemin obtenu avec l'API de la plateforme, et vérifier que le résultat reste dans le répertoire de base avant tout accès au fichier. En cas d'échec du test, rejeter la requête, sans retomber sur une valeur par défaut.
- Ne jamais se fier à un simple retrait de la sous-chaîne `../` (vulnérable aux séquences imbriquées) ni implémenter sa propre logique de canonicalisation ; utiliser les fonctions du langage ou du framework.
- Appliquer le principe de moindre privilège sur le compte du processus serveur (accès restreint aux seuls répertoires nécessaires), en défense en profondeur.
- Journaliser et alerter sur les tentatives contenant des séquences de path traversal ou des caractères suspects dans les paramètres de fichier.
- Exemple Java corrigé. L'ancienne forme `getCanonicalPath().startsWith(BASE_DIRECTORY)` a deux défauts : la base n'est pas canonisée, et sans séparateur final un dossier voisin au préfixe identique (par exemple `/var/www/images-privees` pour la base `/var/www/images`) est accepté.
  ```java
  Path base = Paths.get(BASE_DIRECTORY).toRealPath();
  Path target = base.resolve(userInput).normalize();
  if (!target.startsWith(base)) {
      throw new SecurityException("Chemin refusé");
  }
  // traiter le fichier
  ```
  `Path.startsWith(Path)` compare des éléments de chemin entiers, pas des chaînes, ce qui évite le défaut du dossier voisin. Pour suivre aussi les liens symboliques, utiliser `target.toRealPath()` après `resolve`.
- Équivalents dans d'autres langages : `realpath` en PHP, `os.path.realpath` puis `os.path.commonpath` en Python, `path.resolve` en Node.js, chacun suivi d'une comparaison avec la base canonisée.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-09-14
- [x] Practitioner ✅ 2026-09-16

## Journal des labs

### Lab 1 - Traversée de chemin, cas simple
*File path traversal, simple case* - Apprentice - note : [[Path-traversal-cas-simple]]

Ce lab contient une vulnérabilité de traversée de chemin dans l'affichage des images de produits. Pour le résoudre, récupérez le contenu du fichier `/etc/passwd`.

1. Interceptez dans Burp la requête qui charge une image de produit.
2. Remplacez le paramètre `filename` par `../../../etc/passwd`.
3. Constatez que la réponse contient le contenu du fichier `/etc/passwd`.

**Mes notes de résolution**
- Cible de l'époque : `0a7400fb036c87db800efd7300b4005f.web-security-academy.net`.
- Point d'entrée vulnérable : `GET /image?filename=`, utilisé normalement pour servir les images produit (`filename=53.jpg`).
- Le payload `../../../../etc/passwd` a fonctionné directement, sans contournement : aucun filtre sur `../`, aucune canonicalisation, aucune restriction d'extension. La réponse a un `Content-Type: image/jpeg` alors que le corps est le texte de `/etc/passwd`.
- Oracle de comportement relevé avant l'exploitation : fichier existant, HTTP 200 ; fichier inexistant (`filename=doesnotexist.jpg`), HTTP 400 avec un corps JSON `"No such file"`. Utile pour confirmer une lecture réussie même quand le contenu n'est pas directement lisible.

### Lab 2 - Séquences de traversée bloquées, contournement par chemin absolu
*File path traversal, traversal sequences blocked with absolute path bypass* - Practitioner - note : [[Path-traversal-chemin-absolu]]

L'application bloque les séquences de traversée mais traite le nom de fichier fourni comme relatif au répertoire de travail par défaut. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit.
2. Remplacez `filename` par le chemin absolu `/etc/passwd`, sans aucune séquence de traversée.
3. Constatez que la réponse contient le contenu du fichier.

### Lab 3 - Séquences de traversée retirées de façon non récursive
*File path traversal, traversal sequences stripped non-recursively* - Practitioner - note : [[Path-traversal-sequences-et-encodage]]

L'application retire les séquences de traversée de l'entrée avant de l'utiliser. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit.
2. Remplacez `filename` par `....//....//....//etc/passwd`.
3. Constatez que la réponse contient le contenu du fichier : après le retrait d'une occurrence de `../`, il reste une séquence valide.

### Lab 4 - Séquences de traversée retirées avec un décodage d'URL superflu
*File path traversal, traversal sequences stripped with superfluous URL-decode* - Practitioner - note : [[Path-traversal-sequences-et-encodage]]

L'application bloque les entrées qui contiennent des séquences de traversée, puis décode l'entrée une seconde fois. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit.
2. Remplacez `filename` par `..%252f..%252f..%252fetc/passwd` (le `/` est double-encodé).
3. Constatez que la réponse contient le contenu du fichier.

### Lab 5 - Validation du début du chemin
*File path traversal, validation of start of path* - Practitioner - note : [[Path-traversal-prefixe-et-extension]]

L'application exige que le paramètre `filename` commence par le répertoire de base attendu. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit et notez la valeur de `filename` (par exemple `/var/www/images/12.jpg`).
2. Remplacez-la par `/var/www/images/../../../etc/passwd`.
3. Constatez que la réponse contient le contenu du fichier.

### Lab 6 - Validation de l'extension avec contournement par octet nul
*File path traversal, validation of file extension with null byte bypass* - Practitioner - note : [[Path-traversal-prefixe-et-extension]]

L'application exige que le paramètre `filename` se termine par une extension de fichier attendue. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit.
2. Remplacez `filename` par `../../../etc/passwd%00.png`.
3. Constatez que la réponse contient le contenu du fichier.

**Mes notes de résolution**
- Labs Practitioner résolus le 2026-09-16. Il a fallu tester plusieurs techniques de « Comment exploiter » (traversée classique, chemin absolu, préfixe, octet nul, encodage) selon le filtre rencontré. Détails d'endpoint et de requête à compléter si besoin de les rejouer.

## Mes notes
- 

## Liens
- [[Path-traversal-cas-simple]]
- [[Path-traversal-chemin-absolu]]
- [[Path-traversal-sequences-et-encodage]]
- [[Path-traversal-prefixe-et-extension]]
- [[File-upload]] (écriture arbitraire)
- [[Information-disclosure]] (exploitation des fichiers lus)
- [[SSRF]] et [[XXE-injection]] (autres chemins de lecture de fichiers via `file://`)
- [[Payloads-cheatsheet]]
