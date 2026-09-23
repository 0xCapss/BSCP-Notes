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
 on va aborder les points suivants:
- Les mécanismes d'authentification les plus utilisés dans les sites web.
- Les vulnérabilités potentielles de ces mécanismes.
- Les vulnérabilités inhérentes aux différents mécanismes d'authentification.
- Les vulnérabilités typiques résultant d'une mise en oeuvre incorrecte.
- Il existe 3 types d'authentification:
	- Quelque chose que l'on connait, comme un mot de passe, réponse à une question de sécurité. C'est le "facteur de connaissance"
	- Quelque chose que l'on possède, comme un objet physique, on parle de "facteur de possession"
	- Quelque chose qui nous est propre comme les données biométriques. On parle de "facteurs inhérents"
- Ces mécanismes d’authentification s’appuient sur toute une gamme de technologies pour vérifier un ou plusieurs de ces facteurs.

## Comment détecter
- 

## Comment exploiter (principe)
- La plupart des failles dans les mécanismes d'authentification proviennent des 2 manières suivantes:
	- Les mécanismes d'authentification sont faibles car ils ne protègent pas contre les attaques par bruit de force.
	- Des failles logiques ou un code mal écrit lors de la mise en oeuvre permettent à un attaquant de contourner pleinement les mécanismes d'authentification.
## Impact d'une authentification vulnérable
- Accès à toutes les données et fonctionnalités associés au compte compromis. D'autant plus s'il s'agit d'un compte avec des privilèges élevées comme celui d'un admin système, il pourrait prendre le contrôle de application.
- Même si s'agit d'un compte à bas privilège, un attaquant peut accéder à des données qui n'aurait pas normalement pas accès.
- L'attaquant peut ainsi accéder à d'autre pages, ce qui élargit la surface d'attaque.

## Vulnérabilités liées à la connexion par mot de passe
- Sur les sites web un processus de connexion par mot de passe, les utilisateurs créent eux-même un compte ou se voient attribuer un mot de passe attribué par un admin. Ce compte est associé à un identifiant unique et un mot de passe secret.
- Dans ce cas, le fait de connaitre le mot de passe est une preuve suffisante de l'identité du user. Cela peut se faire de plusieurs manières:
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
-  Bruit de force des mots de passe:
	- Beaucoup de sites adopte une stratégie de mot de passe qui obligent les users à créer des mots de passe à forte entropie.
	- Cela implique d'imposer des mots de passe respectant:
		- Un n ombre minimal de caractère
		- Une combinaison de minuscule/majuscule
		- Des caractères spéciaux
	- Bien que les mots de passe fort soient complexes à casser, le comportement humain introduit involontairement des failles dans ce système.  En effet, les utilisateur choisissent souvent un mot de passe qu'ils tentent de mémoriser et l'adapte pour qu'il respecte la politique de mot de passe.
	- Exemple: si « mypassword » n’est pas autorisé, les utilisateurs peuvent essayer quelque chose comme « Mypassword1! » ou « Myp4w0rd » à la place.
	- Lorsqu'il s'agit de changer de mot de passe, il est courant d'apporter des modification mineurs à leur mot de passe préférés. Par exemple, « Mypassword1! » devient « Mypassword1? » ou « Mypassword2! ».
	- Cette connaissance des identifiants probables et des schémas prévisibles signifie que les attaques par force brute peuvent souvent être bien plus sophistiquées.
- Enumeration des usernames:
	- Cette méthode consiste à un attaquant d'observer les changements de comportement d'un site web afin de savoir si un username est valide ou non.
	- L'énumération se produit sur une page de connexion, par exemple lorsque on saisit un nom d'utilisateur valide et un mot de passe incorrect. Cela réduit le temps nécessaire pour forcer une connexion par bruit de force.
	- Lorsque vous tentez une attaque par bruit de force sur une page de connexion, vous devez prêter une attention particulière aux différences concernant :
		- **Les codes d'état**: Si une tentative renvoie un code d'état différent, cela indique que le username était correct
		- **Les messages d'erreur**: Le message d'erreur peut différer selon que le username et le password sont incorrects
		- **Temps de réponse**:  Toute requête s’écartant de cette norme suggère qu’un événement inhabituel s’est produit en arrière-plan. C’est un autre indice laissant penser que le nom d’utilisateur deviné pourrait être correct.
## Méthode pour contourner un verrouillage de compte
- Etablir une liste de noms d'utilisateurs susceptible d'être valide.
- Définir une liste treès restreinte de mot de passe qu'au moins des user est susceptible d'utiliser.
- A l'aide de Burp Intruder, tester chacun des mots de passe sélectionnés avec chacun des noms d’utilisateur potentiels. Il suffit qu’un seul utilisateur utilise l’un des trois mots de passe pour compromettre un compte.
- Le verrouillage ne protègent pas non plus contre le "credential stuffing". Ces attaques consistent à utiliser un immense dictionnaire de paires nom d’utilisateur/mot de passe.
- Le « credential stuffing » tire parti du fait que de nombreuses personnes réutilisent le même nom d’utilisateur et le même mot de passe sur plusieurs sites web.
## Pièges et points d'attention BSCP
- 

## Prévention
- Toujours renvoyer le même code d'état quelque soit le résultat pour empêcher les attaques par bruit de force.
- Pour les messages d'erreur: utiliser des messages identiques et génériques dans les deux cas.
- Verrouiller le compte auquel l'utilisateur tente d’accéder s'il effectue un nombre élevé de tentatives de connexion sans succès.
- Bloquer l'adresse IP de l'utilisateur distant s'il effectue un nombre élevé de tentatives de connexion rapide.
- Dans ce cas, le simple fait d’inclure vos propres identifiants de connexion à intervalles réguliers dans la liste de mots suffit à rendre cette défense pratiquement inutile.
- On peut également tout simplement verrouiller le compte si l'on subit trop de connexion infructueuses.

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
### Bypass l'authentification 2 facteurs
- Il est possible que l'utilisateur soit invité à entrer un code de vérification sur une page distincte après avoir saisit sont mot de passe. Il se trouve en réalité dans un état "connecté" avant même d'avoir saisis son mot de passe.
- Arrive parfois qu'uns site web ne vérifie pas su on a bien effectué la 2ème étape avant d'afficher la page.


## Liens
- [[Access-control]]
- [[JWT-attacks]]
- [[OAuth]]
- [[Business-logic]]
