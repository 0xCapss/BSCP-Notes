---
tags: [bscp, server-side, access-control]
niveau: apprentice
statut: à faire
---
# Contrôle d'accès - fonctionnalité non protégée (escalade verticale)

## En bref
- Une fonction ou une ressource sensible est accessible sans aucun contrôle dès qu'on en connaît (ou devine) l'URL, indépendamment du rôle de l'utilisateur.
- C'est le cas typique d'escalade verticale : un utilisateur sans droits atteint une fonction réservée à un rôle supérieur.
- Note parente : [[Access-control]]

## Comment détecter
- Tester `/admin`, `/administrator`, `/administrator-panel` en tant qu'utilisateur non authentifié.
- Lire `robots.txt`, `sitemap.xml`, les commentaires HTML et le code JavaScript à la recherche d'URL sensibles.
- Utiliser Burp (Discover content, sitemap) ou Intruder pour deviner des chemins.

## Comment exploiter (principe)
- Elle provient très souvent du fait qu'une application web n'applique aucune protection sur des données sensibles. Par exemple, des fonctions administratives peuvent être disponibles sur la page d'accueil d'un administrateur. Cependant, un utilisateur pourrait accéder à ces fonctions administratives en se rendant directement à l'URL d'administration correspondante. Par exemple, un site web peut héberger des fonctionnalités sensibles à l'URL suivante :
`https://insecure-website.com/admin`
Cette URL peut être accessible à n'importe quel utilisateur, et pas seulement aux utilisateurs administratifs.
- Ces informations peuvent être divulguées à d'autres emplacements, comme le fichier `robots.txt`.
- Un attaquant peut effectuer une attaque par force brute pour trouver l'emplacement de la fonctionnalité.
	- Exemple d'URL : `https://0a08008b03d35c458287bb5f00a800ab.web-security-academy.net/robots.txt`
- Dans certains cas, on attribue à la fonctionnalité une URL moins prévisible, ce qu'on appelle la sécurité par l'obscurité. Le fait de masquer des fonctionnalités sensibles n'assure pas un contrôle d'accès efficace, car les utilisateurs peuvent découvrir l'URL dissimulée de différentes manières. L'application pourrait tout de même divulguer cette URL, par exemple dans le code JavaScript qui construit l'interface utilisateur en fonction du rôle de l'utilisateur. Imaginons une application avec la partie administrative à cette URL : `https://insecure-website.com/administrator-panel-yb556`. Le code JS peut alors ressembler à :
```js
<script>
var isAdmin = false;
if (isAdmin)
	{
		var adminPanelTag = document.createElement('a');
		adminPanelTag.setAttribute('href', 'https://insecure-website.com/administrator-panel-yb556');
		adminPanelTag.innerText = 'Admin panel';
	}
</script>
```

## Pièges et points d'attention BSCP
- Une URL imprévisible n'est pas une protection : elle est souvent divulguée dans le JavaScript ou ailleurs (sécurité par l'obscurité).
- Une fonctionnalité peut être invisible dans l'interface mais accessible côté serveur : l'absence de lien n'est pas une preuve d'absence de vulnérabilité.

## Labs PortSwigger
- [[Access-control#Lab 1 - Fonctionnalité d'administration non protégée]] (Apprentice)
- [[Access-control#Lab 2 - Fonctionnalité d'administration non protégée avec URL imprévisible]] (Apprentice)

## Liens
- [[Access-control]]
- [[Access-control-parametre-utilisateur]]
- [[Information-disclosure]]
