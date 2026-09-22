---
tags: [bscp, server-side, authentication]
niveau: apprentice
statut: à faire
---
# Authentication

## En bref
- Processus qui consiste à vérifier qu'un utilisateur est bien celui qu'il prétend être.
- Cela peut être un formulaire de connexion où l'on saisit son nom d'utilisateur et son mdp.
- Permet à des attaquants d'accéder à des données ou à des fonctionnalités sensibles.
![](Authentication.png)

## Types et variantes
- Dans ce chapitre, on va aborder les points suivants:
- Les mécanismes d'authentification les plus utilisés dans les sites web.
- Les vulnérabilités potentielles de ces mécanismes.
- Les vulnérabilités inhérentes aux différents mécanismes d'authentification.
- Les vulnérabilités typiques résultant d'une mise en oeuvre incorrecte.

## Comment détecter
- 

## Comment exploiter (principe)
- 

## Pièges et points d'attention BSCP
- 

## Prévention
- 

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
- 

## Mes notes
- Différence entre authentification et autorisation:
	- Authentification: Processus qui consiste à vérifier qu'un utilisateur est bien celui qui prétend être.
	- Autorisation: Consiste à vérifier si un utilisateur est autorisé à effectuer une action.
- Attaque par bruit de force:
	- Méthode d'essai et d'erreurs pour deviner les identifiants valides d'un utilisateur. 
	- Ces attaques sont automatisées à l'aide de listes de nom d'utilisateur et de mot de passe potentiels
	- S'appuie sur une logique élémentaire ou des informations récupérées lors de la phase passive. Ainsi les attaquants peuvent considérablement augmentées l'efficacité de leur attaque.
- Bruit de force des noms d'utilisateur
	- Particulièrement facile à deviner car ils suivent un schéma reconnaissable, comme une adresse mail. Les identifiants professionnels ont souvent le format prenom.nom@compagny.com.
	- il arrive même que des comptes à privilège élevées soient crées avec des noms prévisible comme "admin" ou "Administrator".
	- Lors d'un audit, toujours vérifier si le site web divulgue publiquement des noms d'utilisateur potentiels. 
	- Même si le contenu réel des profils est masqué, le nom utilisé dans le profil est parfois identique au nom d’utilisateur de connexion.
	- Vérifier les réponses HTTP pour voir si des adresses e-mail sont divulguées.
	- Les réponses peuvent contenir les adresses mails d'utilisateurs de privilèges élevées.

## Liens
- [[Access-control]]
- [[JWT-attacks]]
- [[OAuth]]
- [[Business-logic]]
