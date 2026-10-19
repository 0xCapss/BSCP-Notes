---
tags: [bscp, server-side, file-upload]
niveau: apprentice
statut: à faire
---
# File upload - validation par propriétés manipulables et en-tête Content-Type

## En bref
- Le site vérifie le type d'un fichier à partir de propriétés que l'attaquant maîtrise, notamment l'en-tête `Content-Type` de la partie fichier d'une requête `multipart/form-data`.
- Une valeur fournie par le client ne peut pas servir de preuve : elle se modifie facilement avec Burp Repeater.
- Note parente : [[File-upload]]

## Comment détecter
- Téléverser un script (`exploit.php`) et lire le message de refus : s'il parle de types MIME autorisés (`image/jpeg`, `image/png`), la validation s'appuie probablement sur le `Content-Type`.
- Retrouver la requête `POST` de téléversement dans l'historique Burp et l'envoyer dans Repeater.

## Comment exploiter (principe)
- **Requête multipart**
	- Pour envoyer un formulaire HTML, le navigateur utilise généralement une requête `POST` avec le type de contenu `application/x-www-form-urlencoded`.
	- Ce type convient pour du texte simple (nom, adresse) mais il est inadapté à l'envoi de grandes quantités de données binaires, comme une image ou un PDF. Pour ces cas, on utilise `multipart/form-data`.
	- Exemple : un formulaire avec trois champs (image, description, nom d'utilisateur) produit une `POST /images` avec `Content-Type: multipart/form-data` et une valeur `boundary` qui sépare les parties.
	- Le corps est divisé en parties distinctes, une par champ. Chaque partie est délimitée par la chaîne `boundary`, et la dernière se termine par la même chaîne suivie de `--`.
	- `Content-Disposition` donne les informations de base sur le champ (`name`, et `filename` pour un fichier).
	- `Content-Type` (facultatif) indique au serveur le type MIME des données soumises via ce champ (ici `image/jpeg`).
- **Validation par l'en-tête Content-Type**
	- Méthode : vérifier que le `Content-Type` propre au champ de fichier correspond au type MIME attendu (par exemple n'autoriser que `image/jpeg` et `image/png`).
	- Problème : le serveur fait implicitement confiance à cette valeur. Si aucune vérification ne confirme que le contenu correspond au type annoncé, la protection est facilement contournable.
	- Contournement : dans Burp Repeater, envoyer le script PHP avec `Content-Type: image/jpeg` dans la partie du fichier.

## Pièges et points d'attention BSCP
- Modifier le `Content-Type` de la **partie fichier** (sous `Content-Disposition`), pas celui de l'en-tête principal de la requête qui porte le `boundary`.
- Ne pas confondre avec une validation du contenu ou de l'extension : ces techniques ne sont pas encore étudiées dans le cours.

## Labs PortSwigger
- [[File-upload#Lab 2 - Téléversement d'un web shell avec contournement de la restriction sur Content-Type]] (Apprentice)

## Liens
- [[File-upload]]
- [[File-upload-web-shell]]
- [[File-upload-liste-noire]]
