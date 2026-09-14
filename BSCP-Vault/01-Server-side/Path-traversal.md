---
tags: [bscp, server-side, path-traversal]
niveau: apprentice
statut: en cours
---
# Path traversal

## En bref
- L'attaque path transversal est une vulnérabilité web qui permet à un attaquant de lire un fichier arbitraire sur le serveur web qui héberge l'application web.

## Types et variantes
- 

## Comment détecter
- Repérer les paramètres ou fonctionnalités qui manipulent un nom de fichier ou un chemin (paramètres nommés filename, file, path, doc, template, page, image, download, include, ou valeurs contenant une extension de fichier).
- Injecter une séquence de traversal simple visant un fichier connu et observable (`../../../etc/passwd` sur Linux, `..\..\..\windows\win.ini` sur Windows) et vérifier si le contenu du fichier apparaît dans la réponse.
- Si le retour n'est pas directement visible, chercher des signaux indirects : différence de code de statut, de message d'erreur, de taille de réponse ou de temps de réponse entre un chemin existant et un chemin inexistant.
- Si la séquence simple est filtrée, tester méthodiquement chaque technique de contournement (encodage simple, double encodage, doubled characters, chemin absolu, null byte) pour identifier précisément le filtre en place.
- Automatiser les tests avec Burp Intruder et une liste de payloads de traversal sur les paramètres suspects.
- Ne pas se limiter aux fonctionnalités de lecture : vérifier aussi les fonctionnalités d'upload, d'export ou de sauvegarde de configuration qui pourraient permettre une écriture de fichier arbitraire.

## Comment exploiter (principe)
### Lecture de fichier arbitraire


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
- Si un nom de fichier doit être accepté depuis l'utilisateur, le valider strictement par liste blanche (extension autorisée, absence de séparateurs de chemin, correspondance à un pattern ancré).
- Après toute normalisation ou décodage, vérifier que le chemin canonique résultant est bien un sous-chemin du répertoire autorisé avant tout accès au fichier.
- Ne jamais se fier à un simple retrait de la sous-chaîne `../` (vulnérable aux doubled characters) ; utiliser les fonctions de canonicalisation fournies par le langage ou le framework plutôt qu'une logique maison.
- Appliquer le principe de moindre privilège sur le compte du processus serveur (accès en lecture/écriture restreint aux seuls répertoires nécessaires) en défense en profondeur.
- Journaliser et alerter sur les tentatives contenant des séquences de traversal ou des caractères suspects dans les paramètres de fichier.

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Mes notes
- 

## Liens
- [[File-upload]]
- [[Information-disclosure]]
- [[Command-injection]]
