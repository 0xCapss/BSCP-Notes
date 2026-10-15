---
tags: [bscp, server-side, file-upload]
niveau: apprentice
statut: à faire
---
# File upload - téléversement sans restriction et web shell

## En bref
- Pire scénario : le site permet de téléverser des scripts côté serveur (PHP, Java, Python) et est configuré pour les exécuter comme du code.
- Il devient alors très simple de déposer son propre web shell sur le serveur.

## Comment détecter
- Repérer une fonction de téléversement (avatar, document, import).
- Téléverser un fichier inoffensif, puis retrouver l'URL à laquelle il est servi : dans **Proxy > HTTP history**, activer le filtre MIME **Images** et chercher le `GET` qui charge le fichier (par exemple `/files/avatars/<fichier>`).
- Tester le téléversement d'un fichier `.php` simple et observer la réponse : accepté, refusé, message d'erreur.

## Comment exploiter (principe)
- **Web shell**
	- Script malveillant qui permet d'exécuter des commandes arbitraires sur un serveur web distant, simplement en envoyant des requêtes HTTP vers le bon point de terminaison.
	- Cela donne de fait un contrôle total sur le serveur.
	- Il permet de lire ou d'écrire des fichiers arbitraires.
	- On peut exfiltrer des données sensibles.
	- Le serveur peut servir de pivot pour attaquer l'infrastructure interne et d'autres serveurs extérieurs au réseau.
- **Exemples en PHP**
	- Lecture de fichier : `<?php echo file_get_contents('/path/to/target/file'); ?>`. Une requête vers le fichier téléversé renvoie le contenu du fichier ciblé dans la réponse.
	- Web shell polyvalent : `<?php system($_GET['command']); ?>`. Il exécute une commande système transmise via un paramètre de requête (`echo` n'est pas nécessaire : `system()` affiche déjà la sortie).
	- Utilisation : `GET /example/exploit.php?command=id HTTP/1.1`.
- **Déroulé type**
	1. Téléverser le fichier `exploit.php`.
	2. Retrouver le chemin où les fichiers sont servis.
	3. Envoyer un `GET` vers `exploit.php` dans Burp Repeater : la réponse contient la sortie du script.

## Pièges et points d'attention BSCP
- Avant de téléverser, garder dans Repeater la requête `GET` qui charge une image existante : il suffira de changer le nom de fichier dans le chemin.
- La confirmation « fichier téléversé » ne prouve pas l'exécution : il faut ensuite appeler le fichier.
- Si le fichier est renvoyé en texte brut au lieu d'être exécuté, c'est que le répertoire ou le type n'est pas exécuté (voir la section sur les fichiers statiques de [[File-upload]]).

## Labs PortSwigger
- [[File-upload#Lab 1 - Exécution de code à distance via le téléversement d'un web shell]] (Apprentice)

## Liens
- [[File-upload]]
- [[File-upload-validation-content-type]]
- [[Command-injection]]
