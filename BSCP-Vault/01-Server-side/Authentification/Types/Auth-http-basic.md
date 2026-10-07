---
tags: [bscp, server-side, authentication]
niveau: apprentice
statut: à faire
---
# Authentification - HTTP Basic

## En bref
- Le navigateur envoie à chaque requête un jeton `base64(username:password)` dans l'en-tête `Authorization`.
- Mécanisme simple mais fragile.
- Note parente : [[Authentication]]

## Comment détecter
- Chercher une réponse `401` avec l'en-tête `WWW-Authenticate: Basic`.
- Vérifier la présence du protocole HSTS et d'une protection contre le brute force.

## Comment exploiter (principe)
- Le client reçoit du serveur un token d'authentification qui est la concaténation `username:password` encodée en base64.
- Ce token est géré par le navigateur et ajouté à chaque requête dans l'en-tête `Authorization` :
`Authorization: Basic base64(username:password)`
- Cette méthode est peu sécurisée car :
	- Elle implique l'envoi répété des identifiants à chaque requête.
	- Sans HSTS, les identifiants risquent d'être interceptés lors d'une attaque man-in-the-middle.
	- Les implémentations ne prennent souvent pas en charge de protection contre le brute force.
	- Elle est particulièrement vulnérable aux exploits liés à la session, notamment le CSRF.

## Pièges et points d'attention BSCP
- Le base64 n'est qu'un encodage réversible : un jeton intercepté donne directement les identifiants.
- Rarement protégé contre le brute force : tester avec Intruder en encodant en base64 (règle de traitement de payload).

## Labs PortSwigger
- Aucun lab dédié dans la formation, mais le principe est utile pour les cookies encodés en base64 (voir [[Auth-maintien-connexion]]).

## Liens
- [[Authentication]]
- [[Auth-brute-force-identifiants]]
