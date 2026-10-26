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
