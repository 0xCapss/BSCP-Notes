# Revue documentaire de la note File-upload (2026-11-06)

Périmètre : /home/capss/web-bscp/BSCP-Vault/01-Server-side/File-upload/File-upload.md (156 lignes), plus lecture des trois fiches courtes (web-shell, validation-content-type, liste-noire) pour les doublons et les liens, et lecture de la section File upload de 04-Essential-skills/Payloads-cheatsheet.md (lignes 99 à 106).
Nature : revue documentaire uniquement. Aucune requête envoyée, aucun lab rejoué, aucun fichier du vault modifié. Toutes les techniques du catalogue sont donc "à vérifier" (rien n'a été rejoué par moi), sauf mention contraire.

## 0. Etat des labs d'après le contenu de la note

| Lab | Niveau | Etat d'après la note | Recommandation |
| --- | --- | --- | --- |
| Remote code execution via web shell upload | Apprentice | Journal présent (lignes 97 à 114) | Considéré réalisé, à confirmer (voir ci-dessous) |
| Web shell upload via Content-Type restriction bypass | Apprentice | Journal présent (lignes 116 à 133) | Considéré réalisé, à confirmer |
| Web shell upload via path traversal | Practitioner | Seulement nommé (ligne 136) | Non réalisé : laisser en l'état |
| Web shell upload via extension blacklist bypass | Practitioner | Seulement nommé (ligne 137) | Non réalisé : laisser en l'état |
| Web shell upload via obfuscated file extension | Practitioner | Seulement nommé (ligne 138) | Non réalisé : laisser en l'état |
| Remote code execution via polyglot web shell upload | Practitioner | Seulement nommé (ligne 139) | Non réalisé : laisser en l'état |
| Web shell upload via race condition | Expert | Seulement nommé (ligne 140) | Non réalisé : laisser en l'état |

Réserves sur les deux premiers labs :
- Les cases de la section "Labs PortSwigger" (lignes 88 à 90) sont toutes décochées, et l'encadré des lignes 94 et 95 dit que le texte est "votre texte d'origine traduit". Les étapes ressemblent à la solution officielle traduite plutôt qu'à un journal personnel (aucun constat propre : pas de nom de fichier réellement utilisé, pas de message réellement observé, pas de difficulté rencontrée). A confirmer par l'utilisateur : si les labs sont faits, cocher et ajouter un vrai retour d'expérience ; sinon, ne pas les compter comme réalisés.
- Pour les cinq labs non réalisés : la recommandation est de laisser la section 136 à 140 dans son état actuel (noms seuls). Cette revue ne propose ni solution ni déroulé pour eux. Le catalogue de la partie 3 décrit des familles de techniques génériques, demandé par le lead, sans lien avec un lab précis ni séquence d'étapes. L'utilisateur peut sauter les sous-parties 3.3 à 3.8 s'il veut éviter tout indice avant de faire les labs.

## 1. Ce qui manque ou est flou

### Concepts
- Aucune définition de web shell dans la note parente (elle est seulement dans la fiche web-shell) : la note parente renvoie sans résumer.
- Différence entre "stocké" et "exécuté" : suggérée lignes 31 à 34 mais sans lien explicite avec le facteur déterminant, qui est la configuration par répertoire (gestionnaire de scripts activé ou non dans le dossier d'upload).
- Notion de gestionnaire d'extension côté serveur (comment Apache, Nginx, IIS, Tomcat associent extension et interpréteur) absente. Elle est pourtant la base des contournements par extension et par fichier de configuration.
- Ordre des opérations de l'application (réception, stockage temporaire, validation, déplacement) absent : indispensable pour comprendre les conditions de concurrence (ligne 81 y fait allusion en prévention seulement).
- Types de validation côté serveur non listés : extension, Content-Type déclaré, signature (octets magiques), dimensions d'image, retraitement de l'image. La note ne donne que les deux premiers par allusion.
- Effets d'un upload autre que l'exécution de code côté serveur : script stocké servi au navigateur (HTML ou SVG avec script, donc XSS stocké), XML dans des formats Office ou SVG (XXE), requêtes sortantes déclenchées par un traitement d'image (SSRF), nom de fichier passé à un shell (injection de commande), écrasement de fichier. Les liens finaux (XSS, XXE, SSRF, Command-injection) existent mais ne sont jamais expliqués dans le texte.
- Méthode de transfert alternative : PUT direct sur le serveur (méthode HTTP autorisée). Absente.

### Méthodologie de détection
- Pas de procédure pour découvrir le mode de rejet : tester séparément extension, Content-Type, contenu (un seul élément modifié à la fois). Ligne 60 le suggère, mais sans tableau "ce que je change / ce que je conclus".
- Pas de méthode pour savoir où le fichier atterrit quand l'URL n'est pas renvoyée (répertoire non indexé, nom aléatoire, stockage sur un autre domaine ou un stockage objet).
- Pas de mention de l'énumération de méthodes (OPTIONS, PUT) ni de l'inspection des en-têtes de réponse (Server, X-Powered-By) pour deviner l'interpréteur et donc l'extension à viser.
- Pas de critère de succès clair par variante (voir catalogue : signal attendu).
- Ligne 61 : "nom conservé" est un indice, mais le test (téléverser deux fois le même nom, comparer) n'est pas décrit.

### Cas par variante
- Le tableau des variantes (lignes 38 à 43) ne contient que quatre lignes. Manquent : fichier de configuration serveur, traversée de chemin dans le nom, validation du contenu et polyglottes, conditions de concurrence, et variantes sans exécution de code (XSS, XXE). La ligne "Application incohérente" renvoie à "voir ci-dessous" mais la section ne donne aucun exemple concret.
- Le niveau de la note est `practitioner` mais seules les variantes Apprentice sont développées.

### Limites et contre-mesures
- Limites des contournements absentes : retraitement d'image (la bibliothèque réencode et détruit les charges utiles), renommage aléatoire, stockage hors racine web, exécution désactivée dans le dossier, pare-feu applicatif.
- Prévention (lignes 77 à 85) correcte mais générique. Manquent : désactiver l'exécution par répertoire avec un exemple de configuration, servir les fichiers d'un domaine séparé sans cookies, en-tête `X-Content-Type-Options: nosniff` et `Content-Disposition: attachment`, antivirus ou analyse du contenu, renvoi du Content-Type calculé par le serveur plutôt que celui du client, vérification après décodage et après normalisation Unicode et URL, désactivation du PUT.
- La remarque sur la taille (ligne 25) cite le déni de service : à garder en une ligne, rappeler qu'on ne le teste pas sur une cible réelle sans accord explicite (règle du dossier).

### Pièges BSCP
- Le contexte examen manque : lecture d'un fichier précis (`/home/carlos/secret`), limite de temps, ordre de priorité des tests.
- Piège de l'encodage des caractères spéciaux dans Repeater (octet nul, point-virgule, barre oblique encodée) : à documenter une fois les labs faits.
- Piège du cache et des noms déjà utilisés : tester avec un nom neuf à chaque essai.
- Piège du `Content-Length` quand on édite le corps multipart à la main (Repeater le recalcule, d'autres outils non).
- Piège de la session : certains labs invalident le téléversement ou la session après déconnexion.

## 2. Superflu, redondant, hors sujet, fautes

### Remarques de rédaction laissées dans le texte
- Lignes 94 et 95 : encadré "Traduction et étapes" (méta-commentaire sur la rédaction). A retirer de la version finale.
- Ligne 67 : "Les autres techniques ne sont pas encore étudiées." Remarque de progression, à remplacer par des liens ou à supprimer.
- Ligne 135 : titre "Labs à venir (techniques pas encore étudiées)". Acceptable comme suivi, mais à intégrer dans une checklist.
- Ligne 143 : section "Mes notes" avec une puce vide.
- Fiches courtes : "Cette section sera complétée à ce moment" (liste-noire) et "pas encore étudiées dans le cours" (validation-content-type, ligne de pièges) : mêmes remarques de progression.

### Redondances
- Lignes 47 et 48 disent deux fois la même chose (les sites sans protection sont rares). Fusionner.
- La section "Vérification des propriétés manipulables" (lignes 50 à 51) est une seule ligne qui renvoie à la fiche Content-Type : fusionner dans le tableau des variantes.
- Pièges dupliqués entre la note parente (lignes 71 à 73) et les fiches : requête GET dans Repeater (web-shell), partie fichier du Content-Type (validation-content-type), message de succès sans exécution (web-shell). Garder le détail dans les fiches, et en parent seulement une ligne de renvoi.
- "Comment détecter" parent (lignes 57 à 62) et "Comment détecter" de web-shell recoupent largement (repérer, téléverser un inoffensif, filtre Images, tester `.php`).
- Les deux labs rédigés dans la note parente reprennent des éléments déjà dits en fiches (exemples PHP, appel du fichier). Les étapes devraient vivre dans les fiches, la note parente ne gardant qu'un index.
- Le fichier cheat sheet contient déjà les cinq premières entrées de ce qui serait à reporter (voir partie 4) : ne pas les dupliquer.

### Liens hors sujet ou non justifiés (lignes 145 à 156)
- Command-injection, Access-control, XSS, XXE-injection, SSRF figurent dans les liens sans phrase qui explique la relation. Soit justifier en une ligne chacun (voir "effets d'un upload autre que l'exécution"), soit retirer. Access-control est le moins justifié (lien possible uniquement via "remplacer le fichier d'un autre utilisateur").
- Tous les fichiers cibles existent (vérifié par recherche de fichiers) : Path-traversal, Command-injection, Access-control, XSS, XXE-injection, SSRF, Race-conditions, Payloads-cheatsheet.
- Les liens de section `[[File-upload#Lab 1 - ...]]` dans les fiches dépendent des titres exacts : tout renommage de titre les casse.

### Fautes et formulations
- Ligne 54 : "réseau d'hébergeurs et de répertoires" imprécis ; "serveurs" ou "hôtes" conviendrait.
- Lignes 101 et 120 : "exfiltrer" employé pour une simple lecture de fichier dans le lab ; préférer "lire".
- Lignes 104 à 133 : adresse à la deuxième personne du pluriel ("Téléversez", "Vous pouvez"), alors que la note est une note personnelle ; incohérent avec le reste du texte à l'impératif ou à l'infinitif.
- Lignes 125 à 127 : ligne vide au milieu de la liste numérotée (l'étape 3 casse le rendu de la liste dans certains moteurs Markdown).
- Ligne 4 : `statut: en cours` alors que la note parente contient un journal sans cases cochées ; aligner le statut avec la réalité des labs.
- Ligne 18 : "Le type de fichier n'est pas validé correctement" suivi de la configuration : OK, mais le pire cas devrait citer aussi `.aspx` et `.jsp` (seulement `.php` et `.jsp` cités ligne 19).
- Section cheat sheet existante (lignes 100 à 105 de 04-Essential-skills/Payloads-cheatsheet.md) utilise des tirets cadratins comme séparateurs, contraire à la règle de style du dossier : à signaler à l'utilisateur, non modifié.

## 3. Catalogue de techniques et payloads par variante

Convention : tout est "à vérifier" (non rejoué). Les noms de répertoire, d'extension d'interpréteur et de comportement sont indicatifs ; le signal confirme ou infirme chaque hypothèse sur la cible réelle.

### 3.1 Web shell (variante sans restriction)
- Contexte : aucune validation, ou validation absente pour ce répertoire, et exécution activée dans le dossier de destination.
- Payloads (à vérifier, sauf les deux premiers déjà dans la note et signalés comme journal de lab 1) :
  - `<?php echo file_get_contents('/home/carlos/secret'); ?>` (lecture ciblée)
  - `<?php system($_GET['command']); ?>` (commande via paramètre)
  - Variante d'extension d'interpréteur : `.php`, `.phtml`, `.php5`, `.phar` pour PHP ; `.jsp`, `.jspx` pour Java ; `.asp`, `.aspx` pour IIS (à vérifier selon la pile).
- Signal attendu : la réponse à `GET` du fichier contient la sortie du script (contenu du secret, sortie de `id`), pas le code source.
- Limites : exécution désactivée dans le dossier (réponse en texte brut ou téléchargement), nom de fichier réécrit, contenu retraité.

### 3.2 Validation du Content-Type de la partie fichier
- Contexte : message de refus mentionnant des types MIME autorisés.
- Technique : rejouer la requête `POST` de téléversement avec `Content-Type: image/jpeg` (ou `image/png`) dans la partie fichier, en gardant l'extension du script et le contenu du script (journal du lab 2, mais l'effet reste "à vérifier" dans ce catalogue).
- Signal attendu : réponse de succès, puis exécution à l'appel.
- Limites : échoue si le serveur contrôle aussi l'extension ou la signature du contenu ; ne pas modifier le Content-Type de la requête principale (il porte le `boundary`).

### 3.3 Liste noire d'extensions (famille de techniques, à vérifier)
- Contexte : refus d'une extension précise (`.php`) alors que d'autres passent.
- Familles de contournement à tester :
  - Extensions alternatives exécutées par le même interpréteur (`.php5`, `.phtml`, `.phar`, variantes de casse `.pHp`).
  - Double extension (`exploit.php.jpg`) : ne marche que si le serveur choisit le gestionnaire sur une autre extension que celle qui est contrôlée.
  - Extension suivie de caractères de fin (point final, espace final, `;`, octet nul encodé `%00`) : dépend de la normalisation du serveur et du langage (à vérifier).
  - Extension doublée pour contourner un nettoyage non récursif (`exploit.p.phphp`) : suppose que le filtre retire la sous-chaîne une seule fois.
  - Encodage du point ou de lettres (URL encodé, doublement encodé, Unicode) : dépend de l'ordre décodage et validation.
- Signal attendu : le fichier est accepté ET la réponse à l'appel contient la sortie du script (pas le texte brut).
- Limites : liste blanche stricte, renommage serveur, extension forcée après validation.

### 3.4 Fichier de configuration serveur
- Contexte : l'application interdit les extensions de script mais accepte une extension inhabituelle, et le serveur honore des fichiers de configuration par répertoire.
- Idée (à vérifier) : téléverser un fichier de configuration qui associe une extension choisie à l'interpréteur.
  - Apache : fichier `.htaccess` avec une directive du type `AddType application/x-httpd-php .l33t` (exige `AllowOverride` suffisant pour le dossier).
  - IIS : `web.config` avec un mappage de gestionnaire pour une extension choisie.
  - Java (conteneur de servlets) : descripteur de déploiement, rarement exploitable par ce biais.
  - Nginx : pas de configuration par répertoire (non applicable).
- Signal attendu : après dépôt du fichier de configuration, un second fichier avec l'extension associée est exécuté.
- Limites : la directive est ignorée si la surcharge est désactivée ; le nom du fichier de configuration peut lui aussi être bloqué.

### 3.5 Traversée de chemin dans le nom de fichier
- Contexte : le nom de fichier du champ `filename` est utilisé pour construire le chemin de destination, et le dossier d'upload n'exécute pas les scripts alors que le dossier parent oui.
- Idée (à vérifier) : un `filename` contenant `../` (éventuellement encodé en URL, `..%2f`, ou doublement encodé) pour déposer le fichier un niveau plus haut. La sortie peut être filtrée avant ou après le décodage.
- Signal attendu : le fichier apparaît à l'emplacement parent (appel du chemin remonté), et il est exécuté.
- Limites : renommage ou nettoyage de la barre oblique ; le serveur ne décode pas le nom ; l'upload réussit mais rien n'apparaît (tester l'emplacement ailleurs).
- Voir [[Path-traversal]] pour les encodages déjà étudiés.

### 3.6 Validation du contenu (signature, dimensions, retraitement)
- Contexte : le serveur lit les premiers octets ou charge l'image avec une bibliothèque.
- Idée (à vérifier) : préfixer le contenu avec la signature d'un format d'image (par exemple `GIF89a`) suivie du code, ou placer le code dans les métadonnées (commentaire EXIF) d'une image valide.
- Signal attendu : acceptation par le contrôle de contenu et exécution lorsque l'extension est exécutable.
- Limites : une vérification de dimensions ou un réencodage détruit les métadonnées et la fin de fichier ; la signature seule ne suffit pas si l'extension est contrôlée.

### 3.7 Polyglottes
- Contexte : cas particulier de 3.6, fichier valide à la fois comme image et comme script.
- Idée (à vérifier) : image réelle valide dont les métadonnées contiennent un fragment de code (outil public d'édition de métadonnées, par exemple exiftool, qui est un outil standard).
- Signal attendu : la sortie du script apparaît dans la réponse, au milieu de données binaires de l'image.
- Limites : exige l'exécution du fichier avec l'extension de l'image ou une extension exécutable acceptée ; échoue si l'image est réencodée.

### 3.8 Conditions de concurrence
- Contexte : le fichier est écrit dans un dossier accessible puis validé ou supprimé après coup (fenêtre de temps).
- Idée (à vérifier) : envoyer en parallèle la requête de téléversement et des requêtes d'appel du fichier, avec l'outil de requêtes parallèles de Burp (groupes de requêtes de Repeater), pour appeler le fichier avant sa suppression. Variante : URL de téléchargement distante et nom temporaire prévisible.
- Signal attendu : au moins une réponse d'appel exécute le script alors que la réponse du téléversement est un refus.
- Limites : dépend de la latence ; plusieurs essais nécessaires ; instable ; ne jamais lancer de grand nombre de requêtes sur une cible non couverte par le périmètre. Voir [[Race-conditions]].

### 3.9 Hors exécution de code (à signaler)
- SVG ou HTML avec script : XSS stocké si servi en même origine avec un type affichable.
- Fichier XML (SVG, formats Office) : XXE selon l'analyseur.
- Traitement d'image côté serveur ou téléversement par URL : SSRF.
- Nom de fichier passé à un shell : injection de commande.
- Chaque point : signal = effet observable (alerte, requête sortante, sortie de commande). Tout "à vérifier".

## 4. A reporter dans la cheat sheet

Déjà présents (lignes 100 à 104 de 04-Essential-skills/Payloads-cheatsheet.md), ne pas dupliquer : les deux web shells PHP, `Content-Type: image/jpeg` dans la partie fichier, appel du fichier téléversé dans Repeater, filtre Images de l'historique.

Candidats à ajouter (tous "à vérifier" ; à ajouter seulement après que l'utilisateur les a rejoués, la ligne 105 de la cheat sheet dit déjà "à ajouter quand elles seront étudiées") :
- Extensions PHP alternatives : `.php5`, `.phtml`, `.phar`, casse mixte `.pHp`
- Double extension : `exploit.php.jpg`
- Octet nul encodé : `exploit.php%00.jpg`
- Extension doublée contre nettoyage non récursif : `exploit.p.phphp`
- Fichier de configuration Apache : `.htaccess` avec `AddType application/x-httpd-php .l33t`
- Traversée dans le nom : `filename="..%2fexploit.php"` (aussi `../exploit.php`, doublement encodé)
- Signature d'image : préfixe `GIF89a` avant le code
- Polyglotte : code dans le commentaire EXIF d'une image valide avec exiftool
- Concurrence : groupe de requêtes parallèles dans Repeater, téléversement plus appel du fichier

Pour respecter l'accord avec l'utilisateur sur les labs : ces entrées correspondent aux techniques des labs non réalisés. Les ajouter maintenant revient à lui donner des indices avant qu'il ait fait ces labs ; recommandation : les ajouter après chaque lab réalisé, une par une.

Style cheat sheet : remplacer les tirets cadratins existants (lignes 100 à 105) par des tirets normaux lors de la prochaine édition.

## 5. Plan de réécriture de la note

Principes : la note parente reste un index et un cours synthétique ; le détail des techniques va dans les fiches ; les labs non réalisés ne sont pas touchés.

1. Entête : aligner `statut` avec la réalité (en cours tant que les cases ne sont pas cochées), conserver le niveau practitioner.
2. En bref et impact : conserver, ajouter une phrase sur les effets hors exécution de code (XSS, XXE, SSRF, injection de commande) avec renvoi aux notes correspondantes.
3. Fichiers statiques et gestionnaires : conserver les lignes 27 à 35, ajouter un court paragraphe sur la configuration par répertoire et l'association extension et interpréteur.
4. Tableau des variantes : l'étendre à toutes les variantes (liste noire, fichier de configuration, traversée de chemin, contenu et polyglottes, concurrence, hors exécution). Les lignes non étudiées pointent vers les noms de labs existants, sans détail. Fusionner les sections "Origine" et "Vérification des propriétés manipulables" dans ce tableau.
5. Détection : un seul tableau "je change / j'observe / je conclus" (extension, Content-Type, contenu, taille, nom conservé, emplacement).
6. Exploitation (principe) : conserver les quatre étapes, retirer la remarque de progression de la ligne 67.
7. Pièges : ne garder que ce qui est transversal (message de succès sans exécution, texte brut = pas exécuté, `wiener:peter`, `/home/carlos/secret`, noms neufs à chaque essai) ; déplacer le reste dans les fiches.
8. Prévention : conserver, ajouter les éléments manquants (partie 1) sous forme de puces courtes.
9. Labs : cocher Apprentice seulement si l'utilisateur confirme avoir fait les deux labs ; remplacer l'encadré de traduction par un vrai journal personnel (difficultés, erreurs, ce qui a fonctionné). Ne pas modifier les cinq labs non réalisés : ils restent nommés sans étapes. A leur réalisation, l'utilisateur écrira lui-même son journal.
10. Nettoyage : supprimer l'encadré des lignes 94 et 95, la puce vide de "Mes notes", la ligne vide dans la liste du lab 2, les doublons des lignes 47 et 48, justifier ou retirer les liens Access-control, XSS, XXE-injection, SSRF, Command-injection.
11. Fiches courtes (statut "à faire") : après la réécriture, y déplacer les étapes détaillées de leurs labs et retirer les remarques de progression. Ne pas les étoffer avant que l'utilisateur ait étudié les techniques.
12. Cheat sheet : voir partie 4, ajout progressif après chaque lab réalisé.
