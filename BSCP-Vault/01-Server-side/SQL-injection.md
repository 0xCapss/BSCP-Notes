---
tags: [bscp, server-side, sql-injection]
niveau: apprentice
statut: à faire
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
- 

## Types et variantes
- 

## Comment détecter
- 

## Comment exploiter (principe)
- Il faut effectuer une série de tests systématiques sur chaque point d'entrée de l'application.
- L'apostrophe simple `'`, en recherchant des erreurs ou d'autres anomalies.
- Une syntaxe SQL qui renvoie la valeur de base du point d'entrée, puis une valeur différente, en recherchant des différences systématiques dans les réponses.
- Des conditions booléennes comme `OR 1=1` et `OR 1=2`, en recherchant des différences dans les réponses.
- Des charges utiles provoquant des retards d'exécution dans une requête SQL, en recherchant des différences de temps de réponse.
- Des charges utiles OAST déclenchant une interaction réseau hors bande, en surveillant les interactions qui en résultent (par exemple avec Burp Collaborator).
- **Burp Scanner** permet de détecter rapidement et de manière fiable la majorité des injections SQL.

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
- 

## Liens
- [[Access-control]]
- [[Authentication]]
- [[Using-Burp-Suite]]
