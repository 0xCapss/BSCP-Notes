---
tags: [bscp, server-side, file-upload]
niveau: practitioner
statut: à faire
---
# File upload vulnerabilities

## En bref
- Elles surviennent quand un serveur web permet de téléverser des fichiers dans son système de fichiers sans valider suffisamment leur nom, leur type et encore moins leur contenu ni leur taille.
- Sans restrictions, même une simple fonction de téléversement d'images peut servir à envoyer des fichiers arbitraire et potentiellement dangereux.
- Cela peut inclure des fichiers de script côté serveur, qui permettent l'exécution de code à distance.
## Impact des vulnérabilités
- L'impact dépend généralement de deux facteurs:
	- L'aspect du fichier que le site valide mal (taille, type, contenu, etc.).
	- Les restrictions imposées au fichier une fois téléversé avec succès.
- Pire cas
	- Le type de fichier n'est pas validé correctement.
	- La configuration du serveur autorise l'exécution de certains types de fichiers comme `.php` et `.jsp` en tant que code.
	- Un attaquant peut alors téléverser un fichier de code côté serveur faisant office de web shell, ce qui lui donne de fait le contrôle total du serveur.
- Nom de fichier mal validé
	- Un attaquant peut remplacer des fichiers critiques en téléversant un fichier portant le même nom.
	- Si le serveur est vulnérable au path transversal l'attaquant peut téléverser des fichiers vers des emplacements imprévus.
- Taille non contrôlée
	- Ne pas vérifier que la taille respecte les limites prévues peut permettre une attaque par déni de service (DoS).
## Traitement des requêtes de fichiers statiques
- Autrefois:
	- Les sites web étaient presque exclusivement constitués de fichiers statiques.
	- Le chemin d'accès de chaque requête correspondait de manière biunivoque à la hiérarchie des répertoires et fichiers sur le système de fichiers du serveur.
- Aujourd'hui:
	- Les sites sont de plus en plus dynamiques : le chemin d'accès d'une requête n'a souvent plus de rapport direct avec le système de fichiers.
	- Les serveurs web traitent néanmoins toujours des requêtes pour certains fichiers statiques, comme les feuilles de style et les images.
- Le processus reste globalement le même : le serveur analyse le chemin de la requête pour identifier l'extension du fichier.
- Il détermine ensuite le type du fichier, généralement en comparant l'extension à une liste préconfigurée de correspondances extension/type MIME.
- La suite dépend du type de fichier et de la configuration du serveur.
- Plusieurs cas possibles:
	- Fichier non exécutable (image, page HTML statique) : le serveur peut simplement renvoyer son contenu dans la réponse HTTP.
	- Fichier exécutable (par exemple PHP) et serveur configuré pour l'exécuter : le serveur assigne des variables à partir des en-têtes et paramètres de la requête, exécute le script, puis peut renvoyer la sortie dans la réponse HTTP.
	- Fichier exécutable mais serveur non configuré pour l'exécuter : il répond généralement par une erreur. Parfois, le contenu du fichier est quand même servi en texte brut, ce qui peut permettre de divulguer du code source et d'autres informations sensibles (exemple dans les supports sur la divulgation d'informations).
- Astuce;
	- L'en-tête de réponse `Content-Type` indique quel type de fichier le serveur pense avoir servi.
	- Si le code de l'application ne l'a pas défini explicitement, il contient normalement le résultat de la correspondance extension/type MIME.
## Types et variantes



## Comment détecter
- 

## Comment exploiter (principe)
### Origine des vulnérabilités de téléchargements de fichiers
- Les sites sans aucune restriction sont rares, vu les dangers évidents.
- Le plus souvent, les développeurs mettent en place une validation supposés robuste, mais souvent elle est soit défaillante, soit facilement contournable.
- Dans la réalité, il est peu probable de trouver un site sans aucune protection contre les attaques par téléversement. Cependant, cela ne signifie pas qu'elles sont robustes. Il est parfois encore possible d'exploiter des failles dans ces mécanismes pour obtenir un web shell et exécuter du code à distance.
### Liste noire de type de fichiers
- Les développeurs peuvent bloquer les types de fichiers dangereux, mais oublier les divergences d''analyse lors de la vérification des extensions.
- Comme toute liste liste, elle omet forcément des types de fichiers moins courants qui restent dangereux.
### Vérification des propriétés manipulables
- Le site peut vérifier le type de fichier à partir de propriétés et l'attaquant peut facilement modifier avec l'outil Repeater.
### Application incohérente
- Même une validation robuste peut être appliquée de manière incohérente à travers le réseau d'hébergeurs et de répertoires qui composent le site.
### Validation incorrecte du type de fichier
- Pour envoyer un formulaire HTML, le navigateur utilise généralement une requête POST avec le type de contenu `application/x-www-form-urlencoded`.
- Ce type convient pour du texte simple (nom, adresse).
- inadapté à l'envoi de grandes quantités de données binaires, comme une image ou un PDF.
- Pour ces cas, il vaut mieux utiliser `multipart/form-data`.
- Exemple:
	- Un formulaire avec trois champs (téléversement d'image, description, nom d'utilisateur).
	- La requête envoyée est une `POST /images` avec l'en-tête `Content-Type: multipart/form-data` et une valeur `boundary` qui sépare les parties.
	- Le corps est divisé en parties distinctes, une par champ du formulaire (`image`, `description`, `username`).
	- Chaque partie est délimitée par la chaîne `boundary`, et la dernière se termine par la même chaîne suivie de `--`.
	- `Content-Disposition` : donne les informations de base sur le champ concerné (`name`, et `filename` pour un fichier).
	- `Content-Type` (facultatif) : indique au serveur le type MIME des données soumises via ce champ (ici `image/jpeg` pour l'image).
	- Intérêt pour la sécurité: Le `Content-Type` de la partie fichier est fourni par le client : l'attaquant peut le modifier avec Burp Proxy ou Repeater. C'est pourquoi s'y fier pour valider le type de fichier est une faille (cela rejoint le point sur les "propriétés facilement manipulables").
- Validation par l'en-tête `Content-Type`:
	- Méthode: Vérifier que l'en-tête `Content-Type` propre au champ de fichier correspond au type MIME attendu.
		- Exemple: un serveur qui n'attend que des images peut n'autoriser que `image/jpeg` et `image/png`.
	- Problème:
		- Survient quand le serveur fait implicitement confiance à la valeur de cet en-tête.
		- Si aucune validation supplémentaire ne vérifie que le contenu du fichier correspond au type MIME annoncé, la protection est facilement contournable.
	- Contournement: 
		- Un outil comme Burp Repeater permet de modifier cet en-tête (par exemple, envoyer un script PHP avec `Content-Type: image/jpeg`).

## Exploiter les téléchargements sans restriction pour déployer un web shell
- Pire scénario: le site permet de téléverser des scripts côté serveur (PHP, Java, Python) et est configuré pour les exécuter comme du code.
- Il devient alors super facile de créer son propre web shell sur le serveur
- **Web shell**
	- Script malveillant qui permet d’exécuter des commandes arbitraires sur un serveur web distant, simplement en envoyant des requêtes HTTP vers le bon point de terminaison.
	- Cela donne de fait un contrôle total sur le serveur
	- Il permet de lire ou écrire des fichiers arbitraires.
	- On peut exfiltrer des données sensibles.
	- Utilise le serveur comme pivot pour attaquer l'infrastructure interne et d'autres serveurs extérieurs au réseau.
- Exemple en PHP:
	- Lecture de fichier : `<?php echo file_get_contents('/path/to/target/file'); ?>`. Une requête vers le fichier téléversé renvoie le contenu du fichier ciblé dans la réponse.
	- Web shell polyvalent : `<?php echo system($_GET['command']); ?>`. Il exécute une commande système transmise via un paramètre de requête.
	- Utilisation : `GET /example/exploit.php?command=id HTTP/1.1`.


## Pièges et points d'attention BSCP
- 

## Prévention
- 

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: Remote code execution via web shell upload
This lab contains a vulnerable image upload function. It doesn't perform any validation on the files users upload before storing them on the server's filesystem.
To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file `/home/carlos/secret`. Submit this secret using the button provided in the lab banner.
You can log in to your own account using the following credentials: `wiener:peter`
1. While proxying traffic through Burp, log in to your account and notice the option for uploading an avatar image.
2. Upload an arbitrary image, then return to your account page. Notice that a preview of your avatar is now displayed on the page.
3. In Burp, go to **Proxy > HTTP history**. Click the filter bar to open the **HTTP history filter** window. Under **Filter by MIME type**, enable the **Images** checkbox, then apply your changes.
4. In the proxy history, notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater.
5. On your system, create a file called `exploit.php`, containing a script for fetching the contents of Carlos's secret file. For example:
    `<?php echo file_get_contents('/home/carlos/secret'); ?>`
6. Use the avatar upload function to upload your malicious PHP file. The message in the response confirms that this was uploaded successfully.
7. In Burp Repeater, change the path of the request to point to your PHP file:
    `GET /files/avatars/exploit.php HTTP/1.1`
8. Send the request. Notice that the server has executed your script and returned its output (Carlos's secret) in the response.
9. Submit the secret to solve the lab.
### Lab: Web shell upload via Content-Type restriction bypass
This lab contains a vulnerable image upload function. It attempts to prevent users from uploading unexpected file types, but relies on checking user-controllable input to verify this.
To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file `/home/carlos/secret`. Submit this secret using the button provided in the lab banner.
You can log in to your own account using the following credentials: `wiener:peter`
1. Log in and upload an image as your avatar, then go back to your account page.
2. In Burp, go to **Proxy > HTTP history** and notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater.
3. On your system, create a file called `exploit.php`, containing a script for fetching the contents of Carlos's secret. For example:
    
    `<?php echo file_get_contents('/home/carlos/secret'); ?>`
4. Attempt to upload this script as your avatar. The response indicates that you are only allowed to upload files with the MIME type `image/jpeg` or `image/png`.
5. In Burp, go back to the proxy history and find the `POST /my-account/avatar` request that was used to submit the file upload. Send this to Burp Repeater.
6. In Burp Repeater, go to the tab containing the `POST /my-account/avatar` request. In the part of the message body related to your file, change the specified `Content-Type` to `image/jpeg`.
7. Send the request. Observe that the response indicates that your file was successfully uploaded.
8. Switch to the other Repeater tab containing the `GET /files/avatars/<YOUR-IMAGE>` request. In the path, replace the name of your image file with `exploit.php` and send the request. Observe that Carlos's secret was returned in the response.
9. Submit the secret to solve the lab.

## Mes notes
- 

## Liens
- [[Path-traversal]]
- [[Command-injection]]
- [[Access-control]]
