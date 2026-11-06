# Revue documentaire de la note Path-traversal.md

- Date : 2026-11-06
- Auteur : web-analyst
- Objet : /home/capss/web-bscp/BSCP-Vault/01-Server-side/Path-traversal/Path-traversal.md (151 lignes)
- Nature : revue documentaire uniquement. Aucune requête envoyée, rien rejoué, aucune note modifiée.
- Source des constats : lecture de la note parente, des quatre fiches courtes (statut "à faire") et de 04-Essential-skills/Payloads-cheatsheet.md.
- Légende : "constat" = observé dans les fichiers lus. "à vérifier" = non rejoué par l'utilisateur, issu de la note, de la fiche courte ou de la connaissance générale.

## 0. Constats préalables

### 0.1 Format de l'en-tête (écart signalé)
- Note parente (lignes 1 à 8) : tags en liste YAML sur plusieurs lignes (`tags:` puis `- bscp`, `- server-side`, `- path-traversal`), statut `en cours`.
- Les quatre fiches courtes du dossier et les notes de Access-control lues (six fichiers contrôlés) : tags en ligne, `tags: [bscp, server-side, path-traversal]`, statut `à faire`.
- Conséquence : le format est incohérent dans le coffre. Une requête Dataview ou une recherche par tag peut traiter les deux formes différemment selon la configuration. Recommandation : aligner la note parente sur le format en ligne (non appliqué ici, la note n'est pas modifiée).
- Second écart : le statut `en cours` alors que les deux cases de la section Labs sont cochées (lignes 72 et 73). À trancher (voir 5).
- Non vérifié : les en-têtes de toutes les autres notes du coffre (seules celles de Path-traversal et six de Access-control ont été lues).

### 0.2 État des labs d'après le contenu de la note
| Lab | Intitulé | Réalisé d'après la note | Trace personnelle |
| --- | --- | --- | --- |
| 1 | Cas simple | Oui (case Apprentice cochée le 2026-09-14) | Oui : section "Mes notes de résolution" (lignes 86 à 90) avec endpoint, payload, oracle de comportement |
| 2 | Chemin absolu | Oui (case Practitioner cochée le 2026-09-16) | Non : seulement énoncé et étapes de type solution officielle (lignes 92 à 99) |
| 3 | Séquences retirées de façon non récursive | Oui (idem) | Non (lignes 101 à 108) |
| 4 | Décodage d'URL superflu | Oui (idem) | Non (lignes 110 à 117) |
| 5 | Validation du début du chemin | Oui (idem) | Non (lignes 119 à 126) |
| 6 | Validation de l'extension, octet nul | Oui (idem) | Non : une seule note collective ligne 138, qui dit que les détails sont "à compléter" |

- Aucun lab n'est marqué comme non réalisé dans la note. La règle "laisser la note en l'état pour les labs non réalisés" ne s'applique donc à aucun lab d'après les cases cochées.
- Les labs 2 à 6 sont déclarés faits mais sans journal propre : les textes des lignes 95 à 135 ressemblent aux énoncés et aux étapes de la solution officielle, pas à un journal de l'utilisateur. Cette revue ne propose pas de déroulé pour ces labs. Le seul travail suggéré est que l'utilisateur écrive lui-même ses observations (voir 5, étape 4). Si l'utilisateur considère en réalité qu'un de ces labs n'est pas fait, il faut décocher la case et laisser la section telle quelle.
- Point d'incohérence à trancher par l'utilisateur : la case "Practitioner" unique couvre cinq labs ; impossible de savoir d'après la note si les cinq sont faits ou seulement une partie.

### 0.3 Doublons et liens entre la note parente et les fiches courtes
- Les fiches courtes reprennent le contenu technique de la note parente : détection (cas-simple, sequences-et-encodage), payloads, pièges (faux positif, décodage simple ou double, préfixe, extension, filtres empilés). Les lignes 46 à 53 de la note parente répètent presque mot pour mot les pièges des fiches.
- Les fiches renvoient vers la note parente avec des liens de section de la forme `[[Path-traversal#Lab 1 - Traversée de chemin, cas simple]]`. Ces ancres dépendent du texte exact des titres de la note parente (lignes 77, 92, 101, 110, 119, 128) : toute réécriture des titres casse ces liens.
- Le lien de section de la fiche chemin absolu vers le lab 2 et celui de la fiche préfixe vers les labs 5 et 6 correspondent bien aux titres actuels (contrôlé).
- La fiche cas-simple contient la seule explication pédagogique du fonctionnement (exemple `loadImage`, code HTML, répertoire `/var/www/images/`). La note parente ne l'a pas : bon endroit pour la garder, mais la note parente devrait en résumer le principe.
- Le cheat sheet contient déjà une section Path traversal (lignes 7 à 20 de Payloads-cheatsheet.md) avec 10 entrées. La section 4 ci-dessous indique ce qui manque.

## 1. Ce qui manque ou est flou

### 1.1 Concepts
- Aucune explication du mécanisme : concaténation du répertoire de base et de l'entrée, puis résolution par le système de fichiers. L'exemple est dans la fiche cas-simple seulement.
- Pas de définition de la différence entre traversée de chemin et inclusion de fichier locale (LFI). Les deux termes apparaissent dans la littérature BSCP et dans les rapports.
- L'écriture arbitraire est citée trois fois (lignes 12, 16, 33, 51) mais jamais illustrée : aucun exemple concret (nom de fichier dans un multipart d'upload, sauvegarde de configuration, extraction d'archive de type "zip slip").
- Le rôle du décodage en couches (navigateur, proxy inverse, serveur web, framework, application) n'est pas expliqué ; c'est pourtant la clé des labs 3 et 4 et du choix entre `%2e%2e%2f` et `%252e%252e%252f`.
- Les lieux d'injection ne sont pas listés : paramètre de requête, segment de chemin d'URL, corps JSON ou formulaire, partie `filename` d'un multipart, en-tête, cookie. Seul le paramètre de requête est traité en pratique.
- Pas de mention des environnements applicatifs : serveur de fichiers statiques, PHP, Java, Node.js, ASP.NET, avec leurs particularités de normalisation.

### 1.2 Méthodologie de détection
- Pas de procédure de référence ("baseline") : relever d'abord la valeur normale, une valeur inexistante, puis les valeurs de traversée, et comparer code, taille, type de contenu, temps. La ligne 89 à 90 du journal du lab 1 contient exactement cet oracle (200 contre 400 avec "No such file"), mais il est enterré dans le journal et pas remonté dans "Comment détecter".
- Pas de méthode pour trouver la profondeur : le nombre de `../` n'est expliqué que dans la fiche cas-simple (séquences en trop sans effet). À remonter.
- Pas de test d'écriture ou de lecture à l'aveugle : comment confirmer sans retour de contenu (différence de comportement entre un fichier qui existe et un qui n'existe pas, ligne 30 mais sans exemple).
- Pas de mise en garde sur l'outillage : un navigateur ou certains clients normalisent `../` avant l'envoi. Dans Burp, utiliser Repeater ; avec curl, utiliser l'option `--path-as-is`. À vérifier avant d'écrire dans la note.
- Pas d'ordre de test qui tienne compte du coût : la ligne 39 donne un ordre, mais la fiche chemin-absolu dit de tester le chemin absolu en premier ; les deux ordres divergent.
- Pas de mention de la liste Burp Intruder "Fuzzing - path traversal" dans la note parente (seulement dans la fiche et la cheat sheet), alors que la ligne 32 parle de l'automatisation.

### 1.3 Cas par variante et par système d'exploitation
- Windows : seul `..\..\..\windows\win.ini` est donné. Manquent : chemin absolu avec lettre de lecteur, encodage de l'antislash (`%5c`, `%255c`), fichiers cibles Windows, comportement `/` contre `\` selon le serveur.
- Linux : seul `/etc/passwd` est donné. Manquent les fichiers utiles au-delà de la preuve (configuration, variables d'environnement du processus, clés), voir section 3.9.
- Variantes d'encodage non standard (`..%c0%af`, `..%ef%bc%8f`) présentes dans la fiche et la cheat sheet, absentes de la note parente.
- Combinaisons : préfixe imposé avec séquences imbriquées, extension imposée avec encodage. La ligne 47 les évoque sans exemple.
- Variantes de séquences imbriquées : seulement `....//` ; `....\/` et `..././` ne sont pas expliquées.

### 1.4 Limites
- Aucune limite d'applicabilité n'est donnée dans la note parente : l'octet nul ne fonctionne plus sur les piles récentes (la fiche préfixe-et-extension le dit, pas la note parente), droits du compte du processus, `/etc/shadow` illisible hors root, chroot ou conteneur, listes de fichiers absents.
- Pas de précision sur ce qu'on ne peut pas faire : lire un répertoire (la traversée lit un fichier, pas un listing), lire des fichiers binaires dans un affichage texte.

### 1.5 Contre-mesures
- La section Prévention est correcte dans ses principes mais :
  - L'exemple Java (lignes 62 à 68) est incomplet. `getCanonicalPath().startsWith(BASE_DIRECTORY)` sans séparateur final accepte un dossier voisin au préfixe identique (par exemple `/var/www/images-privees` pour une base `/var/www/images`). Il faut canoniser aussi la base et comparer avec un séparateur ou utiliser `Path.normalize()` puis `startsWith(Path)`. À vérifier avant de corriger la note.
  - L'exemple n'indique pas ce qu'il faut faire quand le test échoue (rejeter la requête, ne pas retomber sur une valeur par défaut).
  - Un seul langage est illustré ; les équivalents `realpath` (PHP), `os.path.realpath` puis `os.path.commonpath` (Python), `path.resolve` (Node.js) sont à ajouter, à vérifier avant écriture.
  - Rien sur le décodage unique avant validation (valider après le dernier décodage, jamais avant).
  - La ligne 57 mélange liste blanche et liste de caractères autorisés sans les distinguer.

### 1.6 Pièges BSCP
- Pas de lien avec le déroulé de l'examen : où la traversée de chemin peut apparaître dans une chaîne d'attaque (lecture de configuration, d'identifiants, de code source). À vérifier : formulation exacte à reprendre depuis la documentation officielle de l'examen, non consultée ici.
- Pas de mention de la durée de vie des labs (la cible de la ligne 88 n'existe plus), ni du fait que l'identifiant de produit et le répertoire de base varient d'un lab à l'autre.
- Pas de rappel d'un point pratique de Burp : la saisie dans Repeater peut ré-encoder ; vérifier la requête brute réellement envoyée avant de conclure à l'échec d'un encodage. À vérifier.

## 2. Ce qui est superflu, redondant ou hors sujet

### 2.1 Remarques de rédaction laissées dans le texte
- Ligne 138 : "Détails d'endpoint et de requête à compléter si besoin de les rejouer." C'est une consigne à soi-même, pas du contenu.
- Ligne 141 : section "Mes notes" avec une puce vide.
- Ligne 62 : "exemple de code ci-dessous" (ligne 58) puis l'exemple placé en dernière puce de Prévention ; référence au texte qui suit, à reformuler.
- Lignes 35, 43, 44 et 69 à 70 : lignes vides en double qui cassent le rythme des sections.
- Ligne 88 : l'identifiant de cible `0a7400fb...web-security-academy.net` est éphémère, sans utilité pour rejouer. À retirer.

### 2.2 Redondances internes
- Écriture arbitraire : lignes 12, 16, 33 et 51 disent la même chose quatre fois.
- Conventions Unix et Windows : ligne 17, ligne 29 et ligne 52.
- Filtre simple contre canonicalisation : ligne 18, ligne 46 et ligne 59.
- Décodage simple ou double : ligne 31, ligne 39 et ligne 48.
- Pièges extension et préfixe (lignes 49 et 50) : répètent la ligne 39 et les fiches courtes.
- Faux positif : ligne 53 et fiche cas-simple.
- Ordre de test : ligne 31 et ligne 39.
- Étapes des labs 1 à 6 : copiées de l'énoncé ou de la solution officielle, elles dupliquent aussi les payloads de la table (lignes 22 à 25) et des fiches. Seule la partie personnelle a de la valeur.

### 2.3 Liens hors sujet ou mal placés
- `[[Command-injection]]` (ligne 150) : lien sans rapport direct avec le contenu de la note ; la traversée de chemin n'y est pas traitée. Soit justifier le lien par une phrase, soit le retirer.
- `[[File-upload]]` : pertinent pour l'écriture arbitraire, à garder mais à justifier par une phrase.
- `[[Information-disclosure]]` : pertinent (fichiers de configuration lus), à garder.
- Lien manquant à envisager : [[SSRF]] ou [[XXE-injection]] pour d'autres chemins de lecture de fichiers (schéma `file://`). À vérifier : existence et titres exacts des notes (le dossier SSRF existe, XXE-injection.md existe).
- La table (lignes 20 à 25) porte "Séquences imbriquées et encodage" en troisième ligne alors que les labs suivent l'ordre cas simple, chemin absolu, séquences, préfixe et extension : cohérent. En revanche la colonne "Niveau" mélange majuscule et minuscule avec l'en-tête (`apprentice` en minuscule dans les en-têtes, `Apprentice` dans la table).

### 2.4 Fautes et style
- Anglicismes non expliqués : "doubled characters" (lignes 31, 47, 59, 46). Utiliser "séquences imbriquées" partout, comme dans la table.
- Mélange de "traversée de chemin", "path traversal", "directory traversal" et "traversal" nu. Fixer un terme principal (par exemple "traversée de chemin (path traversal)") et s'y tenir.
- Impératif copié de l'énoncé ("Constatez que..."), à remplacer par un style de note personnel.
- Ligne 89 : le payload relevé est `../../../../etc/passwd` (quatre `../`) alors que l'étape 2 ligne 83 en cite trois. Ce n'est pas faux (séquences en trop sans effet) mais cela mérite une phrase explicative.
- Ligne 57 : "à défaut d'une liste blanche stricte, vérifier au minimum" est confus ; reformuler en deux puces.
- Tirets : aucun tiret cadratin repéré dans la note parente (contrôle visuel à la lecture, non automatisé).

## 3. Catalogue de payloads par variante

Règle de marquage : aucun payload ci-dessous n'a été rejoué dans le cadre de cette revue. D'après la note, seul le payload du lab 1 est accompagné d'une trace personnelle (ligne 89). Tout le reste est marqué "à vérifier" : issu de l'énoncé officiel recopié dans la note, des fiches courtes, du cheat sheet ou de connaissance générale. Le terme "signal attendu" est un contrôle à faire, pas un résultat.

Hypothèse commune : le paramètre est de la forme `filename=<valeur>` sur un endpoint de lecture de fichier, avec répertoire de base de type `/var/www/images/`.

### 3.1 Séquences simples
- Payloads :
  - `../../../etc/passwd`
  - `../../../../etc/passwd` (une séquence en trop est sans effet)
  - `..\..\..\windows\win.ini` (Windows)
- Contexte : aucune validation de l'entrée, concaténation directe avec le répertoire de base.
- Signal attendu : le corps de la réponse contient des lignes de la forme `root:x:0:0:` (Linux) ou la section `[fonts]`/`[extensions]` de win.ini (Windows). Le `Content-Type` peut rester celui d'une image ; lire le corps.
- Limites : échoue dès qu'un filtre, un préfixe ou une extension est imposé.
- Statut : payload à quatre `../` rejoué par l'utilisateur sur le lab 1 (trace ligne 89). Variante Windows : à vérifier.

### 3.2 Chemin absolu
- Payloads :
  - `/etc/passwd`
  - `C:\windows\win.ini` (Windows, à vérifier)
  - `/windows/win.ini` (Windows, à vérifier)
- Contexte : l'application bloque ou neutralise `../` mais utilise la valeur telle quelle comme chemin, ou la résout relativement à un répertoire de travail sans forcer la base.
- Signal attendu : contenu du fichier alors que `../../../etc/passwd` est refusé.
- Limites : si l'application préfixe systématiquement la base, la valeur devient `/var/www/images//etc/passwd` et échoue ; passer alors à la section 3.3 ou 3.5.
- Statut : à vérifier.

### 3.3 Séquences retirées de façon non récursive
- Payloads :
  - `....//....//....//etc/passwd`
  - `....\/....\/....\/etc/passwd`
  - `..././..././..././etc/passwd`
- Contexte : un seul passage de suppression de la sous-chaîne `../` ; après suppression, il reste une séquence valide.
- Signal attendu : le contenu du fichier apparaît ; la forme simple `../` ne fonctionne pas.
- Limites : inefficace si la suppression est récursive, ou si elle supprime aussi `..\` et `./` dans un ordre qui recrée une séquence neutralisée. Utile surtout pour identifier le filtre.
- Statut : à vérifier (la forme `....//` provient de l'énoncé du lab 3 recopié dans la note, sans trace personnelle).

### 3.4 Encodage URL simple, double et non standard
- Encodage simple :
  - `%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd`
  - `..%2f..%2f..%2fetc/passwd`
- Double encodage :
  - `%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd`
  - `..%252f..%252f..%252fetc/passwd` (encodage du seul `/`)
- Encodage non standard :
  - `..%c0%af..%c0%af..%c0%afetc/passwd`
  - `..%ef%bc%8f..%ef%bc%8f..%ef%bc%8fetc/passwd`
- Contexte : le filtre inspecte la valeur à un niveau, puis un autre niveau décode à nouveau. Le choix simple ou double dépend du nombre de décodages après le filtre.
- Signal attendu : la forme brute `../` est refusée, la forme encodée aboutit au contenu du fichier. Une erreur différente (autre code de statut ou autre message) entre forme brute et forme encodée est déjà un indice sur l'emplacement du filtre.
- Limites : `%c0%af` (surcodage UTF-8 de `/`) ne passe que sur des décodeurs anciens ; `%ef%bc%8f` (barre oblique pleine chasse) dépend d'une normalisation Unicode côté serveur. Encoder seulement le `/` est souvent plus fiable, d'après la fiche sequences-et-encodage. Attention : l'outil d'envoi peut ré-encoder `%25` ; vérifier la requête brute.
- Statut : forme `..%252f` issue du lab 4 recopié dans la note sans trace personnelle ; toutes les formes sont à vérifier.

### 3.5 Préfixe de dossier imposé
- Payload : `/var/www/images/../../../etc/passwd`
- Variante : `/var/www/images/....//....//....//etc/passwd` (préfixe et suppression non récursive combinés, à vérifier).
- Contexte : l'application vérifie que la valeur commence par la base attendue, sans canoniser.
- Signal attendu : la valeur est acceptée et renvoie le contenu du fichier ; `/etc/passwd` seul est refusé.
- Limites : reprendre exactement le chemin de base vu dans une requête normale. Échoue si l'application canonise avant de comparer. Ne pas confondre avec 3.2 : ici la base est requise.
- Statut : à vérifier.

### 3.6 Extension imposée avec octet nul
- Payloads :
  - `../../../etc/passwd%00.png`
  - `../../../etc/passwd%00.jpg`
- Contexte : l'application vérifie que la valeur se termine par une extension attendue, puis passe la chaîne à une API qui s'arrête au premier octet nul.
- Signal attendu : le fichier est lu bien que l'extension de contrôle soit `.png`. Utiliser une extension déjà vue dans les requêtes normales.
- Limites : ne fonctionne plus sur les piles récentes (la fiche préfixe-et-extension le précise) ; à n'essayer sur une cible réelle qu'après les autres techniques. Versions exactes corrigées : à vérifier avant de les écrire.
- Statut : à vérifier.

### 3.7 Windows avec antislash
- Payloads :
  - `..\..\..\windows\win.ini`
  - `..%5c..%5c..%5cwindows%5cwin.ini`
  - `..%255c..%255c..%255cwindows%255cwin.ini`
  - `C:\windows\win.ini`
  - `....\\....\\....\\windows\\win.ini` (suppression non récursive de `..\`)
- Contexte : serveur Windows ou framework acceptant `\` comme séparateur ; un filtre qui ne traite que `/` laisse passer `\`.
- Signal attendu : contenu de win.ini ; différence de comportement entre `..\` et `../`.
- Limites : sur un serveur Linux, `\` est un caractère ordinaire du nom de fichier et le payload échoue ; il peut donc servir à deviner le système d'exploitation, mais un échec ne prouve rien (à vérifier).
- Statut : à vérifier (rien de Windows n'est rejoué d'après la note).

### 3.8 Écriture arbitraire (principe seulement)
- Contexte : un nom de fichier fourni dans un multipart d'upload, un export ou une sauvegarde de configuration. Le nom de fichier contient une séquence de traversée ou un chemin absolu.
- Vérification contrôlée : écrire un fichier inoffensif et nommé de façon reconnaissable dans un répertoire autorisé par le périmètre, puis le relire. Aucun fichier exécutable ni écrasement de fichier existant sans accord explicite du lead.
- Signal attendu : le fichier apparaît à l'emplacement visé.
- Statut : hypothèse, à vérifier. Détail à traiter dans [[File-upload]].

### 3.9 Fichiers cibles par système d'exploitation
Tous à vérifier (non rejoués). La lecture de chacun dépend des droits du processus.

Linux
- Preuve de lecture : `/etc/passwd`, `/etc/hosts`, `/etc/hostname`, `/etc/os-release`, `/proc/version`
- Contexte du processus : `/proc/self/environ`, `/proc/self/cmdline`, `/proc/self/cwd/` suivi d'un nom de fichier connu
- Configuration des services : `/etc/nginx/nginx.conf`, `/etc/apache2/apache2.conf`
- Application : fichiers `.env`, `config.php`, `settings.py`, `web.xml`, `application.properties` dans le répertoire de l'application (chemin à déduire)
- Identifiants : `/home/<utilisateur>/.ssh/id_rsa`, `/root/.ssh/id_rsa`, `/etc/shadow` (en général illisible sans privilèges élevés)
- Journaux : `/var/log/apache2/access.log`, `/var/log/nginx/access.log` (lecture seule dans cette revue ; toute exploitation relève de l'opérateur)

Windows
- Preuve de lecture : `C:\windows\win.ini`, `C:\windows\system32\drivers\etc\hosts`
- Configuration IIS : `C:\inetpub\wwwroot\web.config`, `C:\windows\system32\inetsrv\config\applicationHost.config`
- Anciens systèmes : `C:\boot.ini`
- Application : fichiers de configuration du site (chemin à déduire)

## 4. À reporter dans la cheat sheet (04-Essential-skills/Payloads-cheatsheet.md)

État actuel de la section (lignes 7 à 20) : traversée simple Unix et Windows, séquences imbriquées, encodage simple, double, non standard, double encodage du `/`, chemin absolu, préfixe, octet nul, fichiers cibles (4), liste Intruder, liens. Le fichier n'est pas modifié.

Déjà présents (ne pas dupliquer) : `../../../etc/passwd`, `..\..\..\windows\win.ini`, `....//` et `....\/`, `%2e%2e%2f`, `%252e%252e%252f`, `..%c0%af`, `..%ef%bc%8f`, `..%252f..%252f..%252fetc/passwd`, `/etc/passwd`, préfixe, octet nul.

Courts, prêts à copier, à ajouter (tous à vérifier, sauf mention contraire) :
- `..%2f..%2f..%2fetc/passwd` : encodage du seul `/`, simple
- `..././..././..././etc/passwd` : séquences imbriquées, variante
- `/var/www/images/....//....//etc/passwd` : préfixe combiné aux séquences imbriquées
- `../../../etc/passwd%00.jpg` : octet nul, extension jpg
- `..%5c..%5c..%5cwindows%5cwin.ini` : antislash encodé
- `..%255c..%255c..%255cwindows%255cwin.ini` : antislash en double encodage
- `C:\windows\win.ini` : chemin absolu Windows
- Fichiers cibles Linux à ajouter : `/etc/hostname`, `/etc/os-release`, `/proc/self/cmdline`, `/proc/version`, `/home/<utilisateur>/.ssh/id_rsa`
- Fichiers cibles Windows à ajouter : `C:\windows\system32\drivers\etc\hosts`, `C:\inetpub\wwwroot\web.config`
- Rappel d'outillage : curl avec `--path-as-is` ; ne pas coller dans la barre d'adresse du navigateur (normalisation de `../`) (à vérifier)
- Oracle de comportement (ligne 89 de la note) : fichier existant renvoie 200, fichier inexistant renvoie 400 avec "No such file" (propre au lab 1, rejoué par l'utilisateur)

Remarque : le cheat sheet utilise des flèches dans d'autres sections (par exemple `?id=123 → ?id=124`) ; hors périmètre de cette revue, mais à signaler au lead vu la consigne de style.

## 5. Plan de réécriture de la note

Principe : la note parente devient la vue d'ensemble ; le détail technique reste dans les quatre fiches courtes ; le journal reste personnel. Le contenu technique n'est pas dupliqué.

1. En-tête
   - Passer les tags au format en ligne `tags: [bscp, server-side, path-traversal]`.
   - Décider du statut : `en cours` tant que le journal des labs 2 à 6 n'est pas écrit par l'utilisateur ; les fiches courtes restent `à faire` tant qu'elles ne sont pas relues.
2. Sections de synthèse (ordre proposé)
   - En bref (garder, une phrase sur l'écriture arbitraire, une seule fois)
   - Mécanisme : trois lignes sur la concaténation et la résolution, avec renvoi vers la fiche cas-simple
   - Types et variantes : garder la table, supprimer les redondances des lignes 16 à 18
   - Comment détecter : ajouter la baseline (valeur normale, valeur inexistante, valeur de traversée), l'oracle de comportement remonté du lab 1, la liste des lieux d'injection, la recherche de profondeur
   - Comment exploiter : un seul ordre de test, cohérent avec la fiche chemin-absolu (chemin absolu d'abord, puis séquences imbriquées, encodage, préfixe, octet nul)
   - Catalogue de payloads : renvoyer au cheat sheet et aux fiches ; ne pas copier la section 3 en entier
   - Limites : octet nul, droits du processus, listing impossible
   - Pièges BSCP : fusionner les lignes 46 à 53 en cinq puces non redondantes
   - Prévention : conserver, corriger l'exemple Java (séparateur final, canonisation de la base), ajouter des équivalents d'autres langages après vérification, dire de valider après le dernier décodage
3. Nettoyage
   - Retirer la ligne 138, la puce vide ligne 141, l'identifiant de cible ligne 88, les lignes vides en double
   - Unifier la terminologie (traversée de chemin ou path traversal) et remplacer "doubled characters" par "séquences imbriquées"
   - Justifier ou retirer le lien Command-injection ; ajouter éventuellement SSRF et XXE-injection
4. Journal des labs
   - Lab 1 : conserver tel quel (journal personnel présent).
   - Labs 2 à 6 : déclarés faits. Les étapes recopiées de l'énoncé peuvent être remplacées, par l'utilisateur lui-même, par ses propres observations (endpoint, filtre identifié, signal obtenu). Aucune solution n'est fournie ici. Si un de ces labs n'est en réalité pas fait, ne pas toucher à sa section et décocher la case.
   - Séparer clairement "énoncé" et "mes notes de résolution" pour chaque lab (le lab 1 le fait déjà).
   - Conserver les titres des sections `### Lab N - ...` tels quels tant que les fiches courtes les citent en lien de section ; sinon mettre à jour les liens des fiches en même temps.
5. Vérification après réécriture
   - Contrôler les liens de section depuis les quatre fiches courtes.
   - Contrôler l'absence de tiret cadratin et de chaîne de flèches.
   - Rejouer, par l'utilisateur, les payloads marqués "à vérifier" avant de les retirer de cette liste.

## 6. Synthèse des points à décider par l'utilisateur
- Les labs 2 à 6 sont-ils tous réalisés (case unique "Practitioner") ?
- Statut final de la note parente (`en cours` ou autre) et alignement du format des tags.
- Faut-il garder le lien vers Command-injection ?
