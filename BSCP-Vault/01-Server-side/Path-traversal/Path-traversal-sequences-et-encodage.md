---
tags: [bscp, server-side, path-traversal]
niveau: practitioner
statut: à faire
---
# Path traversal - séquences imbriquées et encodage

## En bref
- Le serveur retire ou décode les séquences de traversal avant l'accès au fichier, mais de façon incomplète.
- Note parente : [[Path-traversal]]

## Comment détecter
- Si `../` simple échoue, tester les variantes ci-dessous une par une pour identifier le filtre exact.
- Automatiser avec Burp Intruder et la liste **Fuzzing - path traversal**.

## Comment exploiter (principe)
- **Séquences imbriquées** : `....//` ou `....\/`, utiles quand le serveur supprime une seule occurrence de `../` sans répéter l'opération. Après retrait, il reste `../`.
- Dans certains cas, que ce soit dans le chemin d'URL ou dans le paramètre `filename` d'une requête `multipart/form-data`, les serveurs web peuvent supprimer la séquence. On peut contourner ce filtrage de plusieurs façons :
	- Simple encodage : `../` devient `%2e%2e%2f`
	- Double encodage : `../` devient `%252e%252e%252f` (utile si le serveur décode une seconde fois après le filtre)
	- Encodage non standard : `../` devient `..%c0%af` ou `..%ef%bc%8f`
- Burp Intruder est capable de définir ces différents payloads avec la liste **Fuzzing - path traversal**.

## Pièges et points d'attention BSCP
- Vérifier si le serveur applique un décodage simple ou double avant de choisir entre `%2e%2e%2f` et `%252e%252e%252f`.
- Il est souvent plus fiable d'encoder seulement le `/` (`..%252f`) que tous les caractères.
- Les labs combinent parfois plusieurs filtres empilés : traiter chacun avec sa propre technique.

## Labs PortSwigger
- [[Path-traversal#Lab 3 - Séquences de traversée retirées de façon non récursive]] (Practitioner)
- [[Path-traversal#Lab 4 - Séquences de traversée retirées avec un décodage d'URL superflu]] (Practitioner)

## Liens
- [[Path-traversal]]
- [[Path-traversal-chemin-absolu]]
- [[Path-traversal-prefixe-et-extension]]
