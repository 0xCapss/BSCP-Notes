---
tags: [bscp, server-side, file-upload]
niveau: practitioner
statut: en cours
---
# File upload vulnerabilities

## En bref
- Elles surviennent quand un serveur web permet de téléverser des fichiers dans son système de fichiers sans valider suffisamment leur nom, leur type et encore moins leur contenu ni leur taille.
- Sans restrictions, même une simple fonction de téléversement d'images peut servir à envoyer des fichiers arbitraires et potentiellement dangereux.
- Cela peut inclure des fichiers de script côté serveur, qui permettent l'exécution de code à distance.

## Impact des vulnérabilités
- L'impact dépend généralement de deux facteurs :
	- L'aspect du fichier que le site valide mal (taille, type, contenu, etc.).
	- Les restrictions imposées au fichier une fois téléversé avec succès.
- Pire cas
	- Le type de fichier n'est pas validé correctement.
	- La configuration du serveur autorise l'exécution de certains types de fichiers comme `.php` et `.jsp` en tant que code.
	- Un attaquant peut alors téléverser un fichier de code côté serveur faisant office de web shell, ce qui lui donne de fait le contrôle total du serveur.
- Nom de fichier mal validé
	- Un attaquant peut remplacer des fichiers critiques en téléversant un fichier portant le même nom.
	- Si le serveur est vulnérable au path traversal, l'attaquant peut téléverser des fichiers vers des emplacements imprévus (voir [[Path-traversal]]).
- Taille non contrôlée
	- Ne pas vérifier que la taille respecte les limites prévues peut permettre une attaque par déni de service (DoS).

## Traitement des requêtes de fichiers statiques
- Autrefois, les sites web étaient constitués presque exclusivement de fichiers statiques servis à la demande. Le chemin de chaque requête correspondait donc directement à la hiérarchie de répertoires et de fichiers du serveur.
- Aujourd'hui, les sites sont de plus en plus dynamiques, mais le principe de traitement d'une requête de fichier reste le même.
- Le serveur analyse le chemin de la requête pour identifier l'extension du fichier. Il s'en sert pour déterminer le type de fichier demandé, en le comparant à une liste préconfigurée de correspondances entre extensions et types MIME.
- La suite dépend du type de fichier et de la configuration du serveur :
	- Fichier non exécutable (image, page HTML statique) : le serveur renvoie en général le contenu du fichier dans la réponse HTTP.
	- Fichier exécutable (par exemple PHP) et serveur configuré pour l'exécuter : le serveur renseigne des variables à partir des en-têtes et paramètres de la requête, exécute le script, puis renvoie sa sortie au client.
	- Fichier exécutable mais serveur non configuré pour l'exécuter : le serveur répond en général par une erreur. Parfois, il renvoie quand même le contenu du fichier en texte brut, ce qui peut révéler du code source ou d'autres informations sensibles.
- L'en-tête de réponse `Content-Type` donne un indice sur le type de fichier que le serveur pense servir.

## Types et variantes
| Type | Idée clé | Note | Niveau |
| --- | --- | --- | --- |
| Téléversement sans restriction et web shell | `exploit.php` exécuté tel quel | [[File-upload-web-shell]] | Apprentice |
| Validation par `Content-Type` | en-tête de la partie fichier modifiable | [[File-upload-validation-content-type]] | Apprentice |
| Liste noire de types de fichiers | une liste omet forcément des cas | [[File-upload-liste-noire]] | Practitioner |
| Application incohérente | validation robuste ici, absente là | voir ci-dessous | Practitioner |

### Origine des vulnérabilités de téléversement de fichiers
- Les sites sans aucune restriction sont rares, vu les dangers évidents.
- Le plus souvent, les développeurs mettent en place une validation supposée robuste, mais elle est soit défaillante, soit facilement contournable.
- Dans la réalité, il est peu probable de trouver un site sans aucune protection contre les attaques par téléversement. Cependant, cela ne signifie pas qu'elles sont robustes. Il est parfois encore possible d'exploiter des failles dans ces mécanismes pour obtenir un web shell et exécuter du code à distance.

### Vérification des propriétés manipulables
- Le site peut vérifier le type de fichier à partir de propriétés que l'attaquant peut facilement modifier avec Burp Repeater (voir [[File-upload-validation-content-type]]).

### Application incohérente
- Même une validation robuste peut être appliquée de manière incohérente à travers le réseau d'hébergeurs et de répertoires qui composent le site.
- Piste de test : si un répertoire ou un serveur refuse un fichier, vérifier si un autre point d'entrée ou un autre emplacement l'accepte.

## Comment détecter
- Repérer les fonctions de téléversement : avatar, document, pièce jointe, import de fichier.
- Téléverser un fichier légitime et noter : le message de confirmation, l'URL à laquelle le fichier est servi (filtre **Images** de **Proxy > HTTP history**), le `Content-Type` de la réponse.
- Tester un fichier de script (`exploit.php`) et lire le message de refus pour savoir ce qui est contrôlé : extension, `Content-Type`, contenu.
- Vérifier si le nom de fichier d'origine est conservé, ce qui ouvre la voie à l'écrasement de fichiers.
- Vérifier si une limite de taille existe.

## Comment exploiter (principe)
1. Identifier la fonction de téléversement et l'URL de service des fichiers.
2. Téléverser un web shell minimal (voir [[File-upload-web-shell]]).
3. Si le téléversement est refusé, identifier ce qui est contrôlé et contourner : pour le `Content-Type`, voir [[File-upload-validation-content-type]]. Les autres techniques ne sont pas encore étudiées.
4. Appeler le fichier téléversé avec Repeater pour obtenir son exécution et lire le résultat.

## Pièges et points d'attention BSCP
- Garder dans Repeater une requête `GET` qui charge une image existante : on changera seulement le nom de fichier dans le chemin.
- Ne pas confondre le `Content-Type` de la requête principale (avec le `boundary`) et celui de la partie fichier : c'est ce dernier que le serveur teste.
- Un message « téléversé avec succès » ne prouve pas l'exécution : toujours rappeler le fichier.
- Une réponse en texte brut du fichier indique un type non exécuté : le contournement n'est pas encore bon.
- Les labs de ce thème utilisent souvent le compte `wiener:peter` et demandent de lire `/home/carlos/secret`.

## Prévention
- Valider l'extension par liste blanche plutôt que par liste noire : il est plus facile de lister ce qui est autorisé que ce qui est dangereux.
- Vérifier que le nom de fichier ne contient aucune sous-chaîne interprétable comme une traversée de répertoire (voir [[Path-traversal]]).
- Renommer les fichiers téléversés pour éviter que l'un ne remplace un fichier existant.
- Ne pas placer le fichier sur le système de fichiers permanent avant qu'il soit entièrement validé.
- Stocker les fichiers dans un répertoire où l'exécution de scripts est désactivée.
- Ne pas se fier à des valeurs fournies par le client (`Content-Type` de la partie fichier) pour décider du type.
- Utiliser un framework éprouvé pour le prétraitement et la validation plutôt que d'écrire sa propre logique.
- Limiter la taille des fichiers acceptés.

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs

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

## Mes notes
- 

## Liens
- [[File-upload-web-shell]]
- [[File-upload-validation-content-type]]
- [[File-upload-liste-noire]]
- [[Path-traversal]]
- [[Command-injection]]
- [[Access-control]]
- [[XSS]]
- [[XXE-injection]]
- [[SSRF]]
- [[Race-conditions]]
- [[Payloads-cheatsheet]]
