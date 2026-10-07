---
tags: [bscp, server-side, path-traversal]
niveau: apprentice
statut: à faire
---
# Path traversal - lecture de fichier arbitraire (cas simple)

## En bref
- L'application passe une entrée utilisateur à une API de fichiers sans validation : la séquence `../` permet de remonter dans l'arborescence.
- Note parente : [[Path-traversal]]

## Comment détecter
- Repérer un paramètre qui désigne un fichier (`filename`, `file`, `path`, `doc`, `template`, `page`, `image`).
- Injecter `../../../etc/passwd` (Unix) ou `..\..\..\windows\win.ini` (Windows) et chercher le contenu du fichier dans la réponse.

## Comment exploiter (principe)
- Imaginons que l'on souhaite charger une image avec ce code HTML :
  `<img src="/loadImage?filename=218.png">`
- L'URL `loadImage` prend en paramètre un `filename` et retourne le contenu exact de ce fichier. Les images sont stockées sur le disque dans `/var/www/images/`. L'application lit donc le fichier `/var/www/images/218.png`.
- Si l'application ne fait aucune validation sur ce paramètre, un attaquant peut récupérer `/etc/passwd` :
	- `https://insecure-website.com/loadImage?filename=../../../etc/passwd`
- La séquence `../` est valide dans un chemin d'accès car elle permet de remonter d'un niveau dans la hiérarchie des répertoires.
- Sur Windows, `../` et `..\` sont des séquences valides :
	- `https://insecure-website.com/loadImage?filename=..\..\..\windows\win.ini`
- Le nombre de `../` n'a pas besoin d'être exact : des séquences en trop restent à la racine.

## Pièges et points d'attention BSCP
- Un faux positif est possible : un message d'erreur générique ne prouve pas la lecture. Exiger le contenu du fichier ou une différence de comportement reproductible.
- Le `Content-Type` de la réponse peut rester celui d'une image alors que le corps est du texte : lire le corps de la réponse dans Burp, pas le rendu du navigateur.

## Labs PortSwigger
- [[Path-traversal#Lab 1 - Traversée de chemin, cas simple]] (Apprentice)

## Liens
- [[Path-traversal]]
- [[Path-traversal-chemin-absolu]]
- [[Path-traversal-sequences-et-encodage]]
