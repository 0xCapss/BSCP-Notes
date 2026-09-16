---
tags: [bscp, server-side, path-traversal]
niveau: apprentice
statut: en cours
---
# Path traversal

## En bref
- L'attaque path transversal est une vulnérabilité web qui permet à un attaquant de lire un fichier arbitraire sur le serveur web qui héberge l'application web.

## Comment détecter
- Repérer les paramètres ou fonctionnalités qui manipulent un nom de fichier ou un chemin (paramètres nommés filename, file, path, doc, template, page, image, download, include, ou valeurs contenant une extension de fichier).
- Injecter une séquence de traversal simple visant un fichier connu et observable (`../../../etc/passwd` sur Linux, `..\..\..\windows\win.ini` sur Windows) et vérifier si le contenu du fichier apparaît dans la réponse.
- Si le retour n'est pas directement visible, chercher des signaux indirects : différence de code de statut, de message d'erreur, de taille de réponse ou de temps de réponse entre un chemin existant et un chemin inexistant.
- Si la séquence simple est filtrée, tester méthodiquement chaque technique de contournement (encodage simple, double encodage, doubled characters￼￼Types et variantes
- , chemin absolu, null byte) pour identifier précisément le filtre en place.
- Automatiser les tests avec Burp Intruder et une liste de payloads de traversal sur les paramètres suspects.
- Ne pas se limiter aux fonctionnalités de lecture : vérifier aussi les fonctionnalités d'upload, d'export ou de sauvegarde de configuration qui pourraient permettre une écriture de fichier arbitraire.

## Comment exploiter (principe)
### Lecture de fichier arbitraire
- Imaginons que l'on souhaite charger une image avec ce code HTML:
`<img src="/loadImage?filename=218.png">`
- L'URL `loadImage` prend en paramètre un `filename` et retourne le contenu exact de ce fichier. Les images sont stockées sur le disque à la localisation suivante : `/var/www/images/`. Ainsi l'application lit le fichier avec le chemin suivant : `/var/www/images/218.png`
- Si l'application ne fait aucune validation sur ce paramètre, un attaquant peut faire la requête suivante pour retrouver le fichier `/etc/passwd` dans les fichiers du serveur :
	- `https://insecure-website.com/loadImage?filename=../../../etc/passwd`
- La séquence `../` est valide dans un chemin d'accès car elle permet de remonter d'un niveau dans la hiérarchie des répertoires.
- Sur Windows `../` et `..\` sont des séquences valides. Un exemple sur Windows peut être :
	- `https://insecure-website.com/loadImage?filename=..\..\..\windows\win.ini`
- On peut également utiliser des séquences imbriquées tel que `...// ou ....\/` 
- Dans certains cas, dans le chemin d'URL ou dans le paramètre `filename` d'une requête`multipart/form-data` , les serveurs web peuvent supprimer cette séquence. On peut contourner ce filtrage de plusieurs façons:
	- Simple encodage: `../` devient `%2e%2e%2f`
	- Double encodage: `../` devient `%252e%252e%252f`
	- Encodage non-standard: `../` devient `%c0%af ou ..%ef%bc%8f,`
- Burp Intruder est capable de définir ces différents payloads avec la liste **Fuzzing - path traversal**
- Egalement, une application peut exiger que le nom de fichier soit fourni par l'utilisateur comme `/var/www/images.` avec ajout d'un répertoire de base comme par exemple `filename=/var/www/images/../../../etc/passwd`.
	- Enfin, une applicatin peut suggérer que l'on ajoute une extension de fichier. Dans ce cas, on injectte un octet nul pour forcer la fin du chemin d'accès au fichier avant l'extension. Par exemple : `filename=../../../etc/passwd%00.png`.

## Pièges et points d'attention BSCP
- Bien distinguer un simple filtrage de la sous-chaîne `../` (contournable par doubled characters) d'une validation par canonicalisation robuste (résolution du chemin absolu puis vérification qu'il reste dans le répertoire autorisé), beaucoup plus difficile à contourner.
- Ne pas s'arrêter au premier payload qui échoue : les labs combinent souvent plusieurs filtres empilés (décodage, puis suppression de séquences, puis vérification d'extension), chacun nécessitant sa propre technique de contournement.
- Vérifier si le serveur applique un décodage simple ou double avant de choisir entre `%2e%2e%2f` et `%252e%252e%252f`.
- Sur les labs de type "validation que le chemin commence par le dossier attendu", tester le remplacement complet du paramètre par un chemin absolu.
- Sur les labs de type "validation de l'extension de fichier", tester la technique du null byte avant l'extension attendue.
- Tester systématiquement la lecture et l'écriture/upload, cette dernière ayant un impact potentiellement plus critique (exécution de code).
- Si l'environnement cible n'est pas connu, tester les deux conventions de chemin (Unix et Windows).
- Attention aux faux positifs : un message d'erreur générique ne prouve pas la lecture effective du fichier ; exiger un signal explicite (contenu du fichier retourné, ou différence de comportement mesurable et reproductible).

## Prévention
- Éviter de transmettre des entrées utilisateur directement à une API de système de fichiers ; privilégier un mapping indirect (identifiant ou clé associé côté serveur à un chemin fixe, sans jamais exposer le chemin réel à l'utilisateur).
- Si un nom de fichier doit être accepté depuis l'utilisateur, le valider strictement par liste blanche (extension autorisée, absence de séparateurs de chemin, correspondance à un pattern ancré). Si cela n'est pas possible, vérifiez que les données ne contiennent que du contenu autorisé.
- Après avoir validé les données fournies, ajoutez-les au répertoire de base et utilisez une API du système de fichiers de la plateforme pour canoniser le chemin d’accès.
- Vérifiez que le chemin canonisé commence bien par le répertoire de base attendu.
- Après toute normalisation ou décodage, vérifier que le chemin canonique résultant est bien un sous-chemin du répertoire autorisé avant tout accès au fichier.
- Ne jamais se fier à un simple retrait de la sous-chaîne `../` (vulnérable aux doubled characters) ; utiliser les fonctions de canonicalisation fournies par le langage ou le framework plutôt qu'une logique maison.
- Appliquer le principe de moindre privilège sur le compte du processus serveur (accès en lecture/écriture restreint aux seuls répertoires nécessaires) en défense en profondeur.
- Journaliser et alerter sur les tentatives contenant des séquences de traversal ou des caractères suspects dans les paramètres de fichier.
- Voici un exemple de code Java simple permettant de valider le chemin d'un fichier en fonction des données saisies par l'utilisateur :
- ```java
- File file = new File(BASE_DIRECTORY, userInput); 
  if (file.getCanonicalPath().startsWith(BASE_DIRECTORY)) { 
  // process file 
  }`
  ```
## Labs PortSwigger
- [x] Apprentice ✅ 2026-09-14
- [x] Practitioner ✅ 2026-09-16


## Journal des labs
- Lab "File path traversal, simple case" resolu (cible : `0a7400fb036c87db800efd7300b4005f.web-security-academy.net`).
- Endpoint vulnerable : `GET /image?filename=`, utilise normalement pour servir les images produit (`filename=53.jpg`).
- Payload qui a fonctionne directement, sans aucun contournement : `../../../../etc/passwd`. Aucun filtre sur `../`, aucune canonicalisation, aucune restriction d'extension appliquee au contenu retourne (reponse en `Content-Type: image/jpeg` alors que le corps est le texte de `/etc/passwd`).
- Oracle de comportement utile releve avant l'exploitation : fichier existant -> HTTP 200 ; fichier inexistant (`filename=doesnotexist.jpg`) -> HTTP 400 avec corps JSON `"No such file"`. Utile pour confirmer une lecture reussie meme quand le contenu n'est pas directement lisible.
- Confirme en pratique le cas le plus simple de la theorie ci-dessus (aucune des techniques de contournement n'a ete necessaire ici).

## Liens
- [[File-upload]]
- [[Information-disclosure]]
- [[Command-injection]]
