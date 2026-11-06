---
tags: [bscp, server-side, file-upload]
niveau: practitioner
statut: en cours
---
# File upload vulnerabilities

## En bref
- Elles apparaissent quand un serveur web accepte des fichiers téléversés sans contrôler correctement leur nom, leur type, leur contenu ou leur taille.
- Sans restrictions, même une simple fonction de téléversement d'images peut servir à envoyer des fichiers arbitraires et potentiellement dangereux.
- Cela peut inclure des fichiers de script côté serveur, qui permettent l'exécution de code à distance.

## Impact des vulnérabilités
- L'impact dépend généralement de deux facteurs :
	- L'aspect du fichier que le site valide mal (taille, type, contenu, etc.).
	- Les restrictions imposées au fichier une fois téléversé avec succès.
- Pire cas
	- Le type de fichier n'est pas validé correctement.
	- La configuration du serveur autorise l'exécution de certains types de fichiers comme `.php`, `.jsp` ou `.aspx` en tant que code.
	- Un attaquant peut alors téléverser un fichier de code côté serveur faisant office de web shell, ce qui lui donne de fait le contrôle total du serveur.
	- Définition : un web shell est un script malveillant téléversé sur un serveur, qui permet d'exécuter des commandes ou de lire des fichiers à distance par de simples requêtes HTTP vers ce script. Un fichier stocké n'est pas forcément exécuté : tout dépend de la configuration du répertoire de destination (voir la section sur la configuration par répertoire).
- Nom de fichier mal validé
	- Un attaquant peut remplacer des fichiers critiques en téléversant un fichier portant le même nom.
	- Si le serveur est vulnérable au path traversal, l'attaquant peut téléverser des fichiers vers des emplacements imprévus (voir [[Path-traversal]]).
- Taille non contrôlée
	- Ne pas vérifier que la taille respecte les limites prévues peut permettre une attaque par déni de service. Ce test ne se fait pas sur une cible réelle sans accord explicite.
- Effets autres que l'exécution de code côté serveur
	- Un fichier HTML ou SVG contenant un script, servi depuis la même origine avec un type affichable par le navigateur, donne une XSS stockée (voir [[XSS]]).
	- Un fichier XML ou un format qui en contient (SVG, documents Office) peut déclencher une injection d'entité externe selon l'analyseur (voir [[XXE-injection]]).
	- Un traitement d'image côté serveur, ou un téléversement par URL, peut provoquer des requêtes sortantes (voir [[SSRF]]).
	- Un nom de fichier transmis à un shell peut mener à une injection de commande (voir [[Command-injection]]).
	- Écraser le fichier d'un autre utilisateur est un défaut de contrôle d'accès (voir [[Access-control]]).

## Traitement des requêtes de fichiers statiques
- Autrefois, les sites web étaient constitués presque exclusivement de fichiers statiques servis à la demande. Le chemin de chaque requête correspondait donc directement à la hiérarchie de répertoires et de fichiers du serveur.
- Aujourd'hui, les sites sont de plus en plus dynamiques, mais le principe de traitement d'une requête de fichier reste le même.
- Le serveur analyse le chemin de la requête pour identifier l'extension du fichier. Il s'en sert pour déterminer le type de fichier demandé, en le comparant à une liste préconfigurée de correspondances entre extensions et types MIME.
- La suite dépend du type de fichier et de la configuration du serveur :
	- Fichier non exécutable (image, page HTML statique) : le serveur renvoie en général le contenu du fichier dans la réponse HTTP.
	- Fichier exécutable (par exemple PHP) et serveur configuré pour l'exécuter : le serveur renseigne des variables à partir des en-têtes et paramètres de la requête, exécute le script, puis renvoie sa sortie au client.
	- Fichier exécutable mais serveur non configuré pour l'exécuter : le serveur répond en général par une erreur. Parfois, il renvoie quand même le contenu du fichier en texte brut, ce qui peut révéler du code source ou d'autres informations sensibles.
- L'en-tête de réponse `Content-Type` donne un indice sur le type de fichier que le serveur pense servir.

### Configuration par répertoire et gestionnaires d'extension
- Le serveur associe une extension à un gestionnaire (interpréteur PHP, conteneur de servlets pour `.jsp`, moteur ASP.NET sous IIS). Cette association peut être définie globalement ou répertoire par répertoire.
- Un même fichier `.php` peut donc être exécuté dans un dossier et renvoyé en texte brut dans le dossier d'upload, si l'exécution y est désactivée. C'est le facteur qui sépare « stocké » de « exécuté ».
- Certains serveurs acceptent des fichiers de configuration déposés dans le répertoire (Apache avec `.htaccess`, IIS avec `web.config`). Nginx n'a pas de configuration par répertoire.
- Les en-têtes de réponse `Server` et `X-Powered-By` aident à deviner la pile, donc l'extension à viser. La méthode `OPTIONS` liste parfois les méthodes autorisées.

## Ordre des opérations de l'application
- Un traitement typique enchaîne : réception de la requête, écriture dans un emplacement temporaire, validation (extension, `Content-Type` déclaré, signature, dimensions, retraitement de l'image), puis déplacement vers le dossier définitif.
- Si le fichier est écrit à un emplacement accessible avant la fin de la validation, il existe une fenêtre de temps pendant laquelle il peut être appelé : c'est la base des conditions de concurrence (voir [[Race-conditions]]).
- Autre méthode de transfert : si le serveur autorise la méthode `PUT`, un fichier peut être déposé sans passer par le formulaire de téléversement (voir `OPTIONS` ci-dessus).

## Types et variantes
| Type | Idée clé | Note | Niveau |
| --- | --- | --- | --- |
| Téléversement sans restriction et web shell | `exploit.php` exécuté tel quel | [[File-upload-web-shell]] | Apprentice |
| Validation par `Content-Type` | en-tête de la partie fichier modifiable | [[File-upload-validation-content-type]] | Apprentice |
| Liste noire de types de fichiers | une liste omet forcément des cas | [[File-upload-liste-noire]] | Practitioner |
| Application incohérente | validation robuste ici, absente là | voir ci-dessous | Practitioner |
| Fichier de configuration serveur | associer une extension choisie à l'interpréteur | voir le catalogue ci-dessous | Practitioner |
| Traversée de chemin dans le nom de fichier | sortir d'un dossier où l'exécution est désactivée | [[Path-traversal]] | Practitioner |
| Validation du contenu et polyglottes | fichier valide à la fois comme image et comme script | voir le catalogue ci-dessous | Practitioner |
| Condition de concurrence | fichier appelé avant la fin de la validation | [[Race-conditions]] | Expert |
| Effets sans exécution de code | XSS stockée, XXE, SSRF, injection de commande | voir la section Impact | Variable |

### Origine des vulnérabilités de téléversement de fichiers
- Les sites sans aucune restriction sont rares, vu les dangers évidents. Le plus souvent, les développeurs mettent en place une validation supposée robuste, mais elle est défaillante ou facilement contournable : c'est elle qu'il faut tester.
- Le site peut vérifier le type de fichier à partir de propriétés que l'attaquant modifie facilement avec Burp Repeater (voir [[File-upload-validation-content-type]]).

### Application incohérente
- Même une validation robuste peut être appliquée de manière incohérente à travers les serveurs et les répertoires qui composent le site.
- Piste de test : si un répertoire ou un serveur refuse un fichier, vérifier si un autre point d'entrée ou un autre emplacement l'accepte.

### Catalogue de techniques par variante
Les noms d'extension, de répertoire et de directive sont donnés à titre d'exemple et dépendent de la cible. Le signal attendu indique comment reconnaître que la technique a fonctionné.

#### Web shell sans restriction
- Contexte : aucune validation et exécution activée dans le dossier de destination.
- Variantes d'extension d'interpréteur selon la pile : `.php`, `.phtml`, `.php5`, `.phar` (PHP) ; `.jsp`, `.jspx` (Java) ; `.asp`, `.aspx` (IIS).
- Signal attendu : la réponse à l'appel du fichier contient la sortie du script, pas son code source.
- Limites : exécution désactivée dans le dossier (texte brut ou téléchargement), nom réécrit, contenu retraité.

#### Validation du Content-Type de la partie fichier
- Contexte : message de refus qui cite des types MIME autorisés.
- Signal attendu : succès du téléversement puis exécution à l'appel.
- Limites : échoue si l'extension ou la signature est aussi contrôlée. Ne pas modifier le `Content-Type` de la requête principale, qui porte le `boundary`.

#### Liste noire d'extensions
- Contexte : une extension précise est refusée alors que d'autres passent.
- Pistes à tester :
	- Extensions alternatives gérées par le même interpréteur, et variantes de casse.
	- Double extension : utile seulement si le serveur choisit le gestionnaire sur une autre extension que celle contrôlée.
	- Caractères de fin (point, espace, point-virgule, octet nul encodé) : dépend de la normalisation du serveur et du langage.
	- Extension imbriquée pour contourner un nettoyage non récursif : suppose que le filtre retire la sous-chaîne une seule fois.
	- Encodage du point ou des lettres (URL, double encodage, Unicode) : dépend de l'ordre entre décodage et validation.
- Signal attendu : fichier accepté et sortie du script dans la réponse à l'appel.
- Limites : liste blanche stricte, renommage serveur, extension forcée après validation.

#### Fichier de configuration serveur
- Contexte : les extensions de script sont interdites mais une extension inhabituelle passe, et le serveur honore la configuration par répertoire.
- Idée : déposer un fichier de configuration qui associe une extension choisie à l'interpréteur. Apache : `.htaccess` (le dossier doit autoriser la surcharge de configuration). IIS : `web.config`. Nginx : non applicable.
- Signal attendu : un second fichier portant l'extension associée est exécuté.
- Limites : surcharge désactivée, ou nom du fichier de configuration lui aussi bloqué.

#### Traversée de chemin dans le nom de fichier
- Contexte : le nom du champ `filename` sert à construire le chemin de destination, et le dossier d'upload n'exécute pas les scripts alors que son parent le fait.
- Idée : utiliser un nom contenant `../`, éventuellement encodé en URL ou doublement encodé (voir [[Path-traversal]]).
- Signal attendu : le fichier apparaît à l'emplacement remonté et s'exécute.
- Limites : nettoyage de la barre oblique, nom non décodé, téléversement réussi mais fichier introuvable (chercher l'emplacement ailleurs).

#### Validation du contenu et polyglottes
- Contexte : le serveur lit la signature (premiers octets), les dimensions, ou recharge l'image avec une bibliothèque.
- Idée : préfixer le contenu avec la signature d'un format d'image, ou placer du code dans les métadonnées d'une image valide avec un outil public d'édition de métadonnées (exiftool).
- Signal attendu : acceptation par le contrôle de contenu, puis sortie du script au milieu des données binaires de l'image si l'extension est exécutée.
- Limites : le réencodage de l'image détruit les métadonnées ; la signature seule ne suffit pas si l'extension est contrôlée.

#### Conditions de concurrence
- Contexte : le fichier est écrit dans un emplacement accessible puis validé ou supprimé après coup.
- Idée : envoyer en parallèle le téléversement et des appels du fichier, avec les groupes de requêtes de Burp Repeater (voir [[Race-conditions]]). Variante : téléversement par URL avec nom temporaire prévisible.
- Signal attendu : au moins un appel exécute le script alors que la réponse du téléversement est un refus.
- Limites : dépend de la latence, instable, plusieurs essais nécessaires. Ne pas envoyer de volume important de requêtes hors périmètre.

#### Effets sans exécution de code
- Signal attendu : effet observable (alerte dans le navigateur, requête sortante, sortie de commande). Voir la section Impact pour les notes associées.

#### Limites générales des contournements
- Retraitement ou réencodage de l'image par une bibliothèque, qui détruit les charges utiles.
- Renommage aléatoire des fichiers, ou stockage hors de la racine web, ou sur un autre domaine.
- Exécution désactivée dans le dossier de destination.
- Pare-feu applicatif.

## Comment détecter
- Repérer les fonctions de téléversement : avatar, document, pièce jointe, import de fichier.
- Téléverser un fichier légitime et noter : le message de confirmation, l'URL à laquelle le fichier est servi (filtre **Images** de **Proxy > HTTP history**), le `Content-Type` de la réponse.
- Tester un fichier de script (`exploit.php`) et lire le message de refus pour savoir ce qui est contrôlé : extension, `Content-Type`, contenu.
- Isoler les contrôles : ne changer qu'un seul élément à la fois, avec un nom neuf à chaque essai.

| Je change | J'observe | Je conclus |
| --- | --- | --- |
| Seule l'extension (même contenu d'image) | refus ou succès | l'extension est contrôlée, ou non |
| Seul le `Content-Type` de la partie fichier | refus ou succès | le type déclaré est contrôlé, ou non |
| Seul le contenu (signature d'image présente ou absente) | refus ou succès | le contenu est inspecté, ou non |
| Un fichier plus gros | refus ou succès | une limite de taille existe, ou non |
| Même nom téléversé deux fois | nom conservé, suffixe ou nom aléatoire | écrasement possible, ou renommage |
| Appel de l'URL du fichier | sortie du script, texte brut ou téléchargement | exécution active, ou non, dans le dossier |

- Si l'URL n'est pas renvoyée, chercher l'emplacement dans l'historique Proxy, dans le code source de la page, ou dans les en-têtes de réponse.

## Comment exploiter (principe)
1. Identifier la fonction de téléversement et l'URL de service des fichiers.
2. Téléverser un web shell minimal (voir [[File-upload-web-shell]]).
3. Si le téléversement est refusé, identifier ce qui est contrôlé et contourner : pour le `Content-Type`, voir [[File-upload-validation-content-type]] ; pour une extension refusée, voir [[File-upload-liste-noire]] et le catalogue ci-dessus.
4. Appeler le fichier téléversé avec Repeater pour obtenir son exécution et lire le résultat.

## Pièges et points d'attention BSCP
- Un message « téléversé avec succès » ne prouve pas l'exécution : toujours rappeler le fichier. Une réponse en texte brut indique un fichier non exécuté. Détails dans les fiches [[File-upload-web-shell]] et [[File-upload-validation-content-type]].
- Les labs de ce thème utilisent souvent le compte `wiener:peter` et demandent de lire `/home/carlos/secret`. A l'examen, l'objectif est la lecture d'un fichier précis sous contrainte de temps : tester d'abord le cas sans restriction, puis isoler les contrôles un par un.
- Utiliser un nom de fichier neuf à chaque essai (cache, nom déjà pris, écrasement).
- Si le corps multipart est édité à la main, vérifier le `Content-Length` : Repeater le recalcule, d'autres outils non.
- Les caractères spéciaux (octet nul, point-virgule, barre oblique encodée) demandent de l'attention dans Repeater : contrôler l'encodage réellement envoyé.
- Certains labs invalident le fichier ou la session après une déconnexion : se reconnecter et retéléverser si l'appel échoue.

## Prévention
- Valider l'extension par liste blanche plutôt que par liste noire : il est plus facile de lister ce qui est autorisé que ce qui est dangereux.
- Vérifier que le nom de fichier ne contient aucune sous-chaîne interprétable comme une traversée de répertoire (voir [[Path-traversal]]).
- Renommer les fichiers téléversés pour éviter que l'un ne remplace un fichier existant.
- Ne pas placer le fichier sur le système de fichiers permanent avant qu'il soit entièrement validé.
- Stocker les fichiers dans un répertoire où l'exécution de scripts est désactivée (par exemple, pour Apache, désactiver le moteur PHP dans ce dossier) ou hors de la racine web.
- Ne pas se fier à des valeurs fournies par le client (`Content-Type` de la partie fichier) pour décider du type ; renvoyer un `Content-Type` calculé par le serveur.
- Utiliser un framework éprouvé pour le prétraitement et la validation plutôt que d'écrire sa propre logique.
- Limiter la taille des fichiers acceptés.
- Valider le nom après décodage et après normalisation (URL, Unicode).
- Servir les fichiers depuis un domaine séparé sans cookies, avec `X-Content-Type-Options: nosniff` et, si possible, `Content-Disposition: attachment`.
- Réencoder les images, analyser le contenu (antivirus) et désactiver la méthode `PUT` si elle n'est pas nécessaire.

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs

> [!warning] Traduction et étapes
> Les deux labs ci-dessous sont votre texte d'origine traduit en français. Les labs listés plus bas dépendent de techniques pas encore étudiées : ils sont seulement nommés, sans étapes.

### Lab 1 - Exécution de code à distance via le téléversement d'un web shell
*Remote code execution via web shell upload* - Apprentice - note : [[File-upload-web-shell]]

Ce lab contient une fonction de téléversement d'images vulnérable. Elle n'effectue aucune validation sur les fichiers que les utilisateurs téléversent avant de les stocker dans le système de fichiers du serveur.
Pour le résoudre, téléversez un web shell PHP basique et utilisez-le pour exfiltrer le contenu du fichier `/home/carlos/secret`. Soumettez ce secret avec le bouton de la bannière du lab.
Vous pouvez vous connecter à votre compte avec : `wiener:peter`

1. Avec le trafic passant par Burp, connectez-vous à votre compte et repérez l'option de téléversement d'un avatar.
2. Téléversez une image quelconque, puis retournez sur la page de votre compte. Un aperçu de votre avatar s'affiche.
3. Dans Burp, allez dans **Proxy > HTTP history**. Cliquez sur la barre de filtre pour ouvrir la fenêtre **HTTP history filter**. Sous **Filter by MIME type**, cochez **Images**, puis appliquez.
4. Dans l'historique, constatez que votre image a été chargée par une requête `GET` vers `/files/avatars/<VOTRE-IMAGE>`. Envoyez cette requête dans Burp Repeater.
5. Sur votre machine, créez un fichier `exploit.php` contenant un script qui lit le secret de Carlos. Par exemple :
   `<?php echo file_get_contents('/home/carlos/secret'); ?>`
6. Utilisez la fonction de téléversement d'avatar pour envoyer votre fichier PHP. Le message de la réponse confirme le téléversement.
7. Dans Burp Repeater, changez le chemin de la requête pour pointer vers votre fichier PHP :
   `GET /files/avatars/exploit.php HTTP/1.1`
8. Envoyez la requête. Le serveur a exécuté votre script et renvoyé sa sortie (le secret de Carlos) dans la réponse.
9. Soumettez le secret pour résoudre le lab.

### Lab 2 - Téléversement d'un web shell avec contournement de la restriction sur Content-Type
*Web shell upload via Content-Type restriction bypass* - Apprentice - note : [[File-upload-validation-content-type]]

Ce lab contient une fonction de téléversement d'images vulnérable. Elle tente d'empêcher les types de fichiers inattendus, mais s'appuie pour cela sur une entrée contrôlable par l'utilisateur.
Pour le résoudre, téléversez un web shell PHP basique et utilisez-le pour exfiltrer le contenu du fichier `/home/carlos/secret`. Soumettez ce secret avec le bouton de la bannière du lab.
Vous pouvez vous connecter à votre compte avec : `wiener:peter`

1. Connectez-vous, téléversez une image comme avatar, puis retournez sur la page de votre compte.
2. Dans Burp, allez dans **Proxy > HTTP history** et constatez que votre image a été chargée par une requête `GET` vers `/files/avatars/<VOTRE-IMAGE>`. Envoyez cette requête dans Burp Repeater.
3. Sur votre machine, créez un fichier `exploit.php` contenant un script qui lit le secret de Carlos. Par exemple :

   `<?php echo file_get_contents('/home/carlos/secret'); ?>`
4. Essayez de téléverser ce script comme avatar. La réponse indique que seuls les fichiers de type MIME `image/jpeg` ou `image/png` sont autorisés.
5. Dans Burp, retournez dans l'historique et trouvez la requête `POST /my-account/avatar` qui a servi au téléversement. Envoyez-la dans Burp Repeater.
6. Dans l'onglet Repeater de la requête `POST /my-account/avatar`, dans la partie du corps liée à votre fichier, changez le `Content-Type` en `image/jpeg`.
7. Envoyez la requête. La réponse indique que le fichier a bien été téléversé.
8. Passez à l'autre onglet Repeater contenant la requête `GET /files/avatars/<VOTRE-IMAGE>`. Dans le chemin, remplacez le nom de votre image par `exploit.php` et envoyez la requête. Le secret de Carlos est renvoyé dans la réponse.
9. Soumettez le secret pour résoudre le lab.

### Labs à venir (techniques pas encore étudiées)
- Web shell upload via path traversal - Practitioner
- Web shell upload via extension blacklist bypass - Practitioner
- Web shell upload via obfuscated file extension - Practitioner
- Remote code execution via polyglot web shell upload - Practitioner
- Web shell upload via race condition - Expert

## Liens
- [[File-upload-web-shell]] : fiche sur le web shell sans restriction
- [[File-upload-validation-content-type]] : fiche sur la validation par `Content-Type`
- [[File-upload-liste-noire]] : fiche sur les listes noires d'extensions
- [[Path-traversal]] : nom de fichier utilisé pour construire un chemin
- [[Race-conditions]] : fenêtre de temps entre écriture et validation
- [[Command-injection]] : nom de fichier transmis à un shell
- [[Access-control]] : écrasement du fichier d'un autre utilisateur
- [[XSS]] : script dans un fichier servi au navigateur
- [[XXE-injection]] : XML dans SVG ou formats Office
- [[SSRF]] : requêtes sortantes déclenchées par un traitement d'image ou un téléversement par URL
- [[Payloads-cheatsheet]] : web shells et en-têtes de base déjà listés pour le téléversement
