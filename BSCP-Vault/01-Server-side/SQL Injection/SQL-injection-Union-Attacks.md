---
tags:
  - bscp
  - server-side
  - sql-injection
niveau: practitioner
statut: à faire
---
# SQL-injection-Union-Attacks

## En bref
- Elle s'applique quand l'application est vulnérable à l'injection SQL et que les résultats de la requête sont renvoyés dans ses réponses.
- Le mot clé `UNION` permet d'extraire des données d'autres tables de la base.
- Il permet d’exécuter une ou plusieurs requêtes `SELECT` supplémentaires et d’ajouter leurs résultats à ceux de la requête d’origine.
- Exemple: `SELECT a, b FROM table1 UNION SELECT c, d FROM table2`.
- Cette requête renvoie un seul jeu de résultats à deux colonnes, contenant les valeurs de `a` et `b` (table1) et celles de `c` et `d` (table2).

## Types et variantes
- 

## Comment détecter
- 

## Comment exploiter (principe)
- Pour qu'elle puisse fonctionner, deux conditions essentielles doivent être remplies:
	- Les requêtes individuelles doivent renvoyer le même nombre de colonnes.
	- Les types de données de chaque colonne doivent être compatibles entre les différentes requêtes.
- Cela implique également de déterminer:
	- Le nombre de colonnes renvoyées par la requête d'origine.
	- Quelles colonnes renvoyées par la requête d'origine ont un type de données adapté pour contenir les résultats de la requête injectée.
### Détermination du nombre de colonnes requises (Méthode order by)
- Lors d’une injection SQL UNION, il existe deux méthodes efficaces pour trouver le nombre de colonnes de la requête d’origine. Ce texte détaille la première.
- Principe : injecter une série de clauses `ORDER BY` en incrémentant l’index de colonne jusqu’à ce qu’une erreur se produise.
**Charges utiles**
- Si le point d'injection est une chaîne entre guillemets dans la clause `WHERE` on envoie donc:
	- `' ORDER BY 1--`
	- `' ORDER BY 2--`
	- ' `ORDER BY 3--`
	- et ainsi de suite
- Chaque charge utile trie les résultats selon une colonne différente du jeu de résultats.
- La colonne peut être désignée par son index : il n’est pas nécessaire de connaître les noms des colonnes.
- Quand l’index dépasse le nombre réel de colonnes, la base renvoie une erreur (exemple : la position n° 3 de la clause `ORDER BY` est hors de la plage de la liste de sélection.
### Lecture de la réponse
- L'application peut renvouer l'erreur de la base dans la réponse HTTP, une erreur générique, ou simplement aucun résultat.
- Tant qu’une différence dans la réponse est détectable, on peut en déduire le nombre de colonnes (la dernière valeur d’index qui n’a pas provoqué d’erreur).
### Détermination du nombre de colonnes requises (Méthode union select null)
- Seconde méthode: envoyer une série de charges utiles `UNION SELECT` avec un nombre croissant de valeur `NULL`
- Charges utiles:
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
- 

## Liens
- [[Access-control]]
- [[Authentication]]
- [[Using-Burp-Suite]]
