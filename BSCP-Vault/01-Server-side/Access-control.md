---
tags: [bscp, server-side, access-control]
niveau: apprentice
statut: en cours
---
# Access control

## En bref
- Le contrôle d'accès consiste à appliquer des contraintes afin de déterminer qui ou quoi est autorisé à effectuer des actions ou à accéder à des ressources. Dans le cas du web, le contrôle d'accès repose sur l'authentification et la gestion des sessions. La gestion des contrôles d'accès constitue un réel problème complexe qui impose de nombreuses contraintes. Ces décisions relatives au contrôle d'accès sont très souvent prises par des humains, ce qui augmente encore le risque d'erreur.

![](Access-control.png)

## Types et variantes
- Fonctionnalité non protégée : une ressource ou fonction sensible est accessible sans aucun contrôle dès qu'on en connaît (ou devine) l'URL, indépendamment du rôle de l'utilisateur.
- Contrôle d'accès basé sur un paramètre utilisateur : le rôle ou les droits sont stockés dans un élément modifiable côté client (paramètre de requête, cookie, champ caché) plutôt que dérivés côté serveur.
- Escalade verticale de privilèges (vertical privilege escalation) : un utilisateur avec des droits non administrateurs parvient à accéder à des fonctions réservées à un rôle supérieur.
- Escalade horizontale de privilèges (horizontal privilege escalation) : un utilisateur accède aux ressources d'un autre utilisateur de même niveau de privilège, très proche de l'IDOR.
- Escalade horizontale vers verticale : un vecteur horizontal peut parfois être détourné pour atteindre un compte à privilèges plus élevés (par exemple un compte administrateur), ce qui transforme l'escalade horizontale en escalade verticale.

## Comment détecter
- Cartographier toutes les fonctions sensibles ou administratives de l'application (menu, JavaScript, sitemap Burp) puis tenter d'y accéder directement en tant qu'utilisateur non privilégié ou non authentifié.
- Comparer le comportement entre plusieurs comptes de rôles différents (matrice de droits) pour repérer les incohérences.
- Rechercher les URLs sensibles référencées dans le code JavaScript, les commentaires HTML, `robots.txt`, `sitemap.xml`, ou déductibles par un pattern prévisible.
- Tester la modification de tout paramètre, cookie ou champ caché qui ressemble à un rôle ou un identifiant (`role`, `admin`, `id`, `uid`, `account`) en changeant sa valeur.
- Pour l'escalade horizontale, tester le changement d'identifiant de ressource (numérique incrémental ou deviné) avec un compte différent, sur toutes les fonctionnalités qui manipulent des ressources utilisateur (profil, factures, messages).
- Vérifier le contrôle d'accès sur toutes les méthodes HTTP exposées par un endpoint (GET, POST, PUT, DELETE), pas seulement celle utilisée par l'interface normale.

## Comment exploiter (principe)
### Fonctionnalité non protégée
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

### Contrôle d'accès basé sur un paramètre utilisateur
- Certaines applications déterminent les droits d'accès ou le rôle de l'utilisateur lors de la connexion, puis stockent ces informations dans un emplacement contrôlable par l'utilisateur. Cela peut prendre plusieurs formes :
	- Un champ caché
	- Un cookie
	- Un paramètre dans une requête

Plusieurs exemples d'URL :
`https://insecure-website.com/login/home.jsp?admin=true`
`https://insecure-website.com/login/home.jsp?role=1`

Cette approche est peu sécurisée car l'utilisateur peut modifier la valeur et ainsi accéder à des fonctionnalités non autorisées.

### Escalade horizontale de privilèges
- C'est le cas où un utilisateur peut accéder à des ressources d'un autre utilisateur. On peut utiliser les mêmes méthodes d'exploitation que pour l'escalade verticale. Par exemple, un utilisateur peut accéder à la page de son compte à l'aide de l'URL suivante :
`https://insecure-website.com/myaccount?id=123`

L'utilisateur peut ainsi modifier son identifiant pour afficher un autre utilisateur et ainsi gagner l'accès à un autre compte. Cette technique est très similaire à celle de l'IDOR. Dans certaines applications, le paramètre exploitable n'a pas de valeur prévisible : au lieu d'un numéro croissant, une application peut utiliser des identifiants uniques globaux (GUID) pour identifier les utilisateurs, ce qui peut empêcher un attaquant de deviner ou de prédire l'identifiant d'un autre utilisateur.

## Pièges et points d'attention BSCP
- Ne pas se contenter de tester un seul rôle : comparer systématiquement au moins trois cas (non authentifié, utilisateur standard, utilisateur admin) pour chaque fonction sensible.
- Une fonctionnalité peut être invisible dans l'interface mais rester accessible côté serveur : l'absence de lien n'est pas une preuve d'absence de vulnérabilité.
- Un identifiant non prévisible (GUID) ne signifie pas que le contrôle d'accès est correct : si l'ID est exposé ailleurs (export, notification, historique, requête précédente), l'escalade horizontale reste exploitable.
- Vérifier le contrôle d'accès sur toutes les méthodes HTTP d'un même endpoint (GET, POST, PUT, DELETE), pas seulement celle utilisée par l'UI normale.
- Une escalade horizontale peut devenir verticale si la ressource accédée appartient à un compte administrateur : ne pas s'arrêter à la première preuve d'IDOR, vérifier l'étendue réelle.
- Ne pas confondre absence de contrôle d'accès et défaut d'authentification : bien distinguer "qui es-tu" (authentification) de "as-tu le droit" (autorisation).

## Prévention
- Appliquer le contrôle d'accès côté serveur, sur chaque requête, jamais uniquement côté client (masquage dans l'interface ou en JavaScript).
- Centraliser la logique d'autorisation dans un seul mécanisme vérifié plutôt que de la dupliquer dans chaque contrôleur, pour éviter les oublis.
- Refuser par défaut (deny by default) et n'autoriser explicitement que ce qui est nécessaire, plutôt que de bloquer une liste de cas connus.
- Ne jamais stocker le rôle ou les droits dans un élément contrôlable par le client (paramètre, cookie, champ caché) ; les dériver côté serveur à partir de la session authentifiée.
- Ne pas se fier à des URLs non devinables comme seule protection (sécurité par l'obscurité) ; appliquer un contrôle d'accès explicite sur chaque endpoint sensible.
- Utiliser des identifiants non séquentiels (GUID) en défense en profondeur, sans que cela remplace une vérification explicite que la ressource demandée appartient bien à l'utilisateur courant.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-09-17
- [ ] Practitioner
- [ ] Expert

## Journal des labs
- 

## Mes notes
- 

## Liens
- [[Authentication]]
- [[Business-logic]]
- [[Information-disclosure]]
