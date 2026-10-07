---
tags: [bscp, server-side, ssrf]
niveau: apprentice
statut: à faire
---
# SSRF contre d'autres systèmes back-end

## En bref
- Le serveur d'applications peut communiquer avec des systèmes back-end qui ne sont pas directement accessibles aux utilisateurs.
- Ces systèmes ont souvent des adresses IP privées non routables (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`).
- Note parente : [[SSRF]]

## Pourquoi c'est vulnérable
- Ils sont généralement protégés par la topologie du réseau, d'où un niveau de sécurité souvent plus faible.
- Beaucoup contiennent des fonctionnalités sensibles accessibles sans authentification à quiconque peut les atteindre.

## Comment détecter
- Même point d'entrée que pour [[SSRF-serveur-local]] : un paramètre qui contient une URL complète.
- Remplacer l'hôte par une adresse privée plausible et observer les différences de réponse (code de statut, taille, temps de réponse, message d'erreur).

## Comment exploiter (principe)
- **Exemple**
	- Une interface d'administration existe à l'URL back-end `http://192.168.0.68/admin`.
	- L'attaquant envoie `POST /product/stock` avec `stockApi=http://192.168.0.68/admin`.
	- Le serveur d'applications relaie la requête vers ce système interne et renvoie l'interface d'administration.
- **Quand l'adresse est inconnue : balayage avec Burp Intruder**
	- Placer une position de payload sur le dernier octet de l'adresse, avec un payload de type Numbers de 1 à 255.
	- Ajuster le port si l'énoncé le précise (par exemple `8080`).
	- Trier les résultats par code de statut : une réponse différente des autres (souvent `200`) révèle l'hôte actif.

## Pièges et points d'attention BSCP
- Une réponse lente ou une erreur de délai d'attente indique souvent un hôte filtré ; une réponse rapide avec une erreur indique un hôte atteint mais un chemin ou un port erroné.
- Penser à balayer aussi les ports courants (`80`, `8080`, `8000`, `8443`) si l'énoncé ne les donne pas.

## Labs PortSwigger
- [[SSRF#Lab 2 - SSRF basique contre un autre système back-end]] (Apprentice)

## Liens
- [[SSRF]]
- [[SSRF-serveur-local]]
- [[SSRF-aveugle]]
