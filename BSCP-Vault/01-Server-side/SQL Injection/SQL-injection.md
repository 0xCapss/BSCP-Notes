---
tags: [bscp, server-side, sql-injection]
niveau: apprentice
statut: en cours
---
# SQL injection

## En bref
- Faille de sécurité Web qui permet à un attaquant d'interférer avec les requêtes que l'application envoie à sa base de données.
- L'attaquant peut consulter des données auxquelles il n'a normalement pas accès.
- Il peut ainsi modifier, supprimer ces données, ce qui entraîne des changements persistants dans le contenu ou le comportement de l'application.
- Dans certaines situations, l'attaque peut être étendue pour compromettre le serveur sous-jacent ou d'autres infrastructures back-end.
- Elle peut également permettre de mener des attaques par déni de service.
## Récupération de données cachées
- Exemple : une boutique en ligne affiche des produits par catégorie. Cliquer sur la catégorie Gifts envoie la requête `https://insecure-website.com/products?category=Gifts`.
- L'application exécute alors la requête SQL : `SELECT * FROM products WHERE category = 'Gifts' AND released = 1`.
**Lecture de la requête**
- `*` : tous les détails.
- `products` : la table interrogée.
- `category = 'Gifts'` : uniquement la catégorie Gifts.
- `released = 1` : uniquement les produits publiés.
**Rôle de la restriction**
- `released = 1` sert à masquer les produits non publiés.
- On peut supposer que les produits non publiés ont `released = 0`.
## Exemple d'attaque
**Attaque 1 : supprimer la restriction avec un commentaire**
- L'application n'a aucune protection contre l'injection SQL.
- L'attaquant envoie `https://insecure-website.com/products?category=Gifts'--`.
- La requête devient : `SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1`.
- `--` est un indicateur de commentaire en SQL : le reste de la requête est ignoré, donc `AND released = 1` disparaît.
- Résultat : tous les produits de la catégorie sont affichés, y compris ceux qui ne sont pas encore sortis.

**Attaque 2 : afficher toutes les catégories**
- Une attaque similaire force l'application à afficher les produits de n'importe quelle catégorie, même inconnue : `category=Gifts'+OR+1=1--`.
- La requête devient : `SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1`.
- Elle renvoie tous les articles dont la catégorie est `Gifts` ou pour lesquels 1 est égal à 1.
- Comme `1=1` est toujours vrai, tous les articles sont renvoyés.

**Avertissement**
- Attention en injectant `OR 1=1` : même si cela semble inoffensif là où on l'injecte, les applications réutilisent souvent les données d'une requête dans plusieurs autres.
- Si la condition atteint un `UPDATE` ou un `DELETE`, elle peut provoquer une perte accidentelle de données.
## Détournement logique d'application
**Détourner la logique de l'application**
- Exemple : une application authentifie les utilisateurs par nom d'utilisateur et mot de passe. Avec `wiener` et `bluecheese`, elle exécute : `SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'`.
- Si la requête renvoie les détails d'un utilisateur, la connexion réussit. Sinon, elle est rejetée.
**Attaque**
- Un attaquant peut se connecter en tant que n'importe quel utilisateur, sans mot de passe.
- Il utilise la séquence de commentaire SQL `--` pour retirer la vérification du mot de passe de la clause `WHERE`.
- Exemple : le nom d'utilisateur `administrator'--` avec un mot de passe vide donne : `SELECT * FROM users WHERE username = 'administrator'--' AND password = ''`.
- Cette requête renvoie l'utilisateur `administrator` et connecte l'attaquant sous ce compte.

## Types et variantes
- Vus jusqu'ici :
	- Récupération de données cachées (suppression d'une restriction de la clause `WHERE`).
	- Détournement de la logique de l'application (contournement de connexion).
	- Attaque `UNION` : voir [[SQL-injection-Union-Attacks]].
- À venir (pas encore étudiés) : injection aveugle booléenne, basée sur les erreurs, basée sur le temps, hors bande (OAST), de second ordre.

## Comment détecter
- Il faut effectuer une série de tests systématiques sur chaque point d'entrée de l'application :
	- L'apostrophe simple `'`, en recherchant des erreurs ou d'autres anomalies.
	- Une syntaxe SQL qui renvoie la valeur de base du point d'entrée, puis une valeur différente, en recherchant des différences systématiques dans les réponses.
	- Des conditions booléennes comme `OR 1=1` et `OR 1=2`, en recherchant des différences dans les réponses.
	- Des charges utiles provoquant des retards d'exécution dans une requête SQL, en recherchant des différences de temps de réponse.
	- Des charges utiles OAST déclenchant une interaction réseau hors bande, en surveillant les interactions qui en résultent (par exemple avec Burp Collaborator).
- **Burp Scanner** permet de détecter rapidement et de manière fiable la majorité des injections SQL.
- Les points d'entrée ne se limitent pas aux paramètres d'URL : corps de la requête, cookies et en-têtes sont aussi à tester.

## Comment exploiter (principe)
- Comprendre la requête SQL probable côté serveur (ici une clause `WHERE` avec une chaîne entre guillemets) pour savoir où fermer le guillemet et quoi neutraliser.
- Détournement de logique :
	- Commenter la fin de la requête avec `--` pour supprimer une restriction (`AND released = 1`) ou une vérification (`AND password = ...`).
	- Ajouter une condition toujours vraie (`OR 1=1`) pour élargir les résultats.
- Extraction de données : `UNION` pour lire d'autres tables quand les résultats sont affichés, voir [[SQL-injection-Union-Attacks]].
- Payloads regroupés dans [[Payloads-cheatsheet]].

## Pièges et points d'attention BSCP
- MySQL : `--` doit être suivi d'un espace (`--+` dans une URL), ou utiliser `#` encodé en `%23`.
- Encodage URL : `+` est lu comme un espace, `#` non encodé n'est jamais envoyé au serveur.
- Le point d'injection n'est pas toujours dans l'URL (corps, cookie, en-tête).
- `OR 1=1` sur une fonctionnalité qui modifie des données (`UPDATE`, `DELETE`) peut causer une perte de données : le réserver à des lectures.

## Prévention
- Utiliser des requêtes paramétrées (requêtes préparées) pour toute donnée fournie par l'utilisateur, jamais de concaténation de chaînes dans la requête.
- Appliquer le principe du moindre privilège au compte de la base de données.
- Ne pas renvoyer les erreurs détaillées de la base à l'utilisateur.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-26
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data
Ce lab contient une injection SQL dans le filtre de catégorie des produits. Quand l'utilisateur choisit une catégorie, l'application exécute une requête SQL comme celle-ci :
`SELECT * FROM products WHERE category = 'Gifts' AND released = 1`
Pour réussir le lab, il faut réaliser une injection SQL qui fait afficher à l'application un ou plusieurs produits non publiés.
#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête qui définit le filtre de catégorie.
2. Modifier le paramètre `category` en lui donnant la valeur `'+OR+1=1--`.
3. Envoyer la requête et vérifier que la réponse contient maintenant un ou plusieurs produits non publiés.
### Lab: SQL injection vulnerability allowing login bypass
Ce lab contient une injection SQL dans la fonction de connexion.
Pour réussir le lab, il faut réaliser une injection SQL qui connecte à l'application en tant qu'utilisateur `administrator`.
#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête de connexion.
2. Modifier le paramètre `username` en lui donnant la valeur : `administrator'--`, avec un mot de passe quelconque (le champ est obligatoire dans le formulaire, mais la vérification est commentée).
## Mes notes
- 

## Liens
- [[Access-control]]
- [[Authentication]]
- [[Using-Burp-Suite]]
- [[SQL-injection-Union-Attacks]]
- [[Payloads-cheatsheet]]
