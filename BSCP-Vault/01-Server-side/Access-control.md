---
tags: [bscp, server-side, access-control]
niveau: apprentice
statut: à faire
---
# Access control

## En bref
- Le contrôle d'accès consiste à appliquer des contraintes afin de déterminer qui ou quoi est autoriser à effectuer des actions ou à accéder à des ressources. Dans le cas du web, le contrôle d'accès repose sur l'authentification et la gestion des sessions. Cette gestion des contrôles d'accès constituent un réel problème complexe qui impose de nombreuses contraintes. Ces decisions relatives au contrôle d'accès sont très souvent prise par des humains, ce qui augmente encore plus le risque d'erreur.

![](Access-control.png)
## Mes notes
### Vertical privilege escalation
- C'est dans le cas ou un user avec des droits non administrateurs qui arrive à gagner l'accès à la page administrateur ou il peut supprimer des comptes.
## Fonctionnalité non protégée
- Elle provient très souvent lorsqu'une application web n'applique aucune protection aux données sensibles. Par exemple, des fonctions administratives peuvent être disponible sur la page d’accueil d'un administrateur. Cependant, un utilisateur pourrait accéder à ces fonctions administratives en se rendant directement à l’URL d’administration correspondante. Par exemple, un site web peut héberger des fonctionnalités sensibles à l'URL suivante :
https://insecure-website.com/admin
Cette URL peut être accessible à n'importe quel utilisateur, et pas seulement aux utilisateurs administratifs.
- Ces informations peut être divulguée à d'autres emplacement tel que le fichier `robots.txt`
- Un attaquant peut faire du bruit de force pour trouver l'emplacement de la fonctionnalité.
	- Exemple d'URL : `https://0a08008b03d35c458287bb5f00a800ab.web-security-academy.net/robots.txt`
- Dans certains cas, on leur attribue une URL moins prévisible, ce qu'on appelle la sécurité par l'obscurité. Le fait de masquer des fonctionnalités sensibles n’assure pas un contrôle d’accès efficace, car les utilisateurs peuvent découvrir l’URL dissimulée de différentes manières. Cependant, l'application pourrait tout de même divulguer cette URL aux utilisateurs. L'URL pourrait être révélée dans le code JavaScript qui construit l'interface utilisateur en fonction du rôle de l'utilisateur. imaginons une application avec la partie administrative à cette URL : `https://insecure-website.com/administrator-panel-yb556`. Alors le code JS devient:
```js
<script> var isAdmin = false; 
if (isAdmin) 
	{ ... 
		var adminPanelTag = document.createElement('a'); 
		adminPanelTag.setAttribute('href', 'https://insecure-website.com/administrator-panel-yb556');
		adminPanelTag.innerText = 'Admin panel';
	 ...
	}
</script>   
```
### Méthodes de contrôle d'accès basées sur des paramètres
- Certaines applications déterminent les droites accès ou le rôle de l'utilisateur lors de la connexion, puis stockent ces informations dans un emplacement contrôlable par l'utilisateur. Cela peut avoir plusieurs formes:
	- Un champ caché
	- Un cookie
	- Un paramètre dans une requête
Plusieurs exemples d'URL:
`https://insecure-website.com/login/home.jsp?admin=true` 
`https://insecure-website.com/login/home.jsp?role=1`
Cette approche est totalement insécurisé car l'utilisateur peut modifier la valeur et ainsi accéder à des fonctionnalités non autorisées.
### Horizontal Privilege Escalation
C'est le cas ou un utilisateur peut accéder à des ressources d'un autre utilisateur. On peut utiliser les mêmes méthodes d'exploitation que la vertical privilege escalation. Par exemple, un utilisateur peut accéder à la page de son compte à l'aide de l'URL suivante :
`https://insecure-website.com/myaccount?id=123`
L'utilisateur peut ainsi modifier son indentifiant pour afficher un autre utilisateur et ainsi gagner l'accès à un autre compte.Cette technique est très similaire à celle de l'IDOR. Dans certaines applications, le paramètre exploitable n'a pas de valeur prévisible. Par exemple, au lieu d'un numéro croissant, une application peut utiliser des identifiants uniques globaux (GUID) pour identifier les utilisateurs. Cela peut empêcher un attaquant de deviner ou de prédire l'identifiant d'un autre utilisateu
## Types et variantes
- 

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

## Liens
- [[Authentication]]
- [[Business-logic]]
- [[Information-disclosure]]
