---
tags: [bscp, server-side, access-control]
niveau: apprentice
statut: à faire
---
# Contrôle d'accès - basé sur un paramètre contrôlé par l'utilisateur

## En bref
- Certaines applications déterminent les droits d'accès ou le rôle de l'utilisateur lors de la connexion, puis stockent ces informations dans un emplacement contrôlable par l'utilisateur : champ caché, cookie ou paramètre de requête.
- L'utilisateur peut modifier la valeur et accéder à des fonctionnalités non autorisées.
- Note parente : [[Access-control]]

## Comment détecter
- Chercher dans les cookies, champs cachés et corps de requêtes/réponses des valeurs qui ressemblent à un rôle ou à un niveau de droits (`Admin=false`, `role=1`, `roleid`, `isAdmin`).
- Modifier la valeur et comparer l'accès obtenu.

## Comment exploiter (principe)
- Certaines applications déterminent les droits d'accès ou le rôle de l'utilisateur lors de la connexion, puis stockent ces informations dans un emplacement contrôlable par l'utilisateur. Cela peut prendre plusieurs formes :
	- Un champ caché
	- Un cookie
	- Un paramètre dans une requête

Plusieurs exemples d'URL :
`https://insecure-website.com/login/home.jsp?admin=true`
`https://insecure-website.com/login/home.jsp?role=1`

Cette approche est peu sécurisée car l'utilisateur peut modifier la valeur et ainsi accéder à des fonctionnalités non autorisées.
- **Mass assignment** : dans une requête de mise à jour de profil (par exemple changement d'adresse e-mail en JSON), la réponse peut révéler des champs supplémentaires comme `roleid`. Les ajouter à la requête (`"roleid":2`) peut suffire à changer de rôle.

## Pièges et points d'attention BSCP
- Penser à vérifier la réponse d'une requête de mise à jour : elle expose souvent les champs internes modifiables.
- Le changement de rôle peut ne prendre effet qu'après avoir renvoyé la requête avec le bon paramètre, sans reconnexion.

## Labs PortSwigger
- [[Access-control#Lab 3 - Rôle de l'utilisateur contrôlé par un paramètre de requête]] (Apprentice)
- [[Access-control#Lab 4 - Rôle de l'utilisateur modifiable dans le profil]] (Apprentice)

## Liens
- [[Access-control]]
- [[Access-control-fonctionnalite-non-protegee]]
- [[Business-logic]]
