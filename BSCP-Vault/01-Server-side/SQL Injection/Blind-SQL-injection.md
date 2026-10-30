---
tags:
  - bscp
  - server-side
  - sql-injection
niveau: practitioner
statut: à faire
---
# Blind-SQL-injection

## En bref
- Se produit généralement quand l'application est vulnérable au SQLi mais que ses réponses HTTP ne contiennent ni les résultats de la requête SQL concernée, ni le détail d'éventuelles erreurs de base de données.
- **Limite des techniques classiques**:
	- De nombreuses techniques, comme les attaques UNION, sont inefficaces, car elles reposent sur la possibilité de voir les résultats de la requête injectée dans les réponses.
## Types et variantes
### Blind SQLi - Réponses conditionnelles
- Exemple: une application utilise un cookie de suivi pour collecter des données analytiques. Chaque requête contient `Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4`
- Pour savoir s’il s’agit d’un utilisateur connu, l’application exécute : `SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'`
#### Vulnérabilité
- La requête est vulnérable mais ses résultats ne sont pas envoyés à l'utilisateur.
- L'application se comporte différemment selon la requête renvoie ou non des données.
- Avoir un `TrackingId` reconnu, la requête renvoie des données et la réponse contient le message "Welcome Back"

## Comment détecter


## Comment exploiter (principe)
### Blind SQLi - Réponses conditionnelles
- Cette différence de comportement suffit pour exploiter une blind SQLi.
- On récupère des informations en déclenchant des réponses différentes de manières conditionnelles, selon une condition injectée.
- Concrètement, on injecte une condition vraie ou fausse dans le cookie et on observe la présence ou l’absence du message « Welcome back ».
- Deux requêtes sont envoyées tour à tour, avec ces valeurs pour le cookie `TrackingId` :
	- `…xyz' AND '1'='1`
	- `…xyz' AND '1'='2`
- **Résultat**:
	- Première valeur : la condition injectée `AND '1'='1` est vraie, la requête renvoie des résultats, donc le message « Welcome back » s’affiche.
	- Seconde valeur : la condition injectée est fausse, la requête ne renvoie aucun résultat, donc le message « Welcome back » n’apparaît pas.
- **Conséquence**:
	- On peut déterminer la réponse (vrai ou faux) à n’importe quelle condition injectée.
	- Les données s’extraient ainsi une information à la fois (par exemple caractère par caractère, avec des conditions du type « le premier caractère du mot de passe est-il `a` ? »
- **Extraire un mot de passe caractère par caractère**
- Hypothèse de l’exemple : une table `Users` avec les colonnes `Username` et `Password`, et un utilisateur `Administrator`.
- On teste le mot de passe caractère par caractère avec une série de requêtes.

**Étape 1 : première comparaison**
- Charge utile : `xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm`
- Elle renvoie « Welcome back » : la condition est vraie, donc le premier caractère est supérieur à `m`.
**Étape 2 : réduire l’intervalle**
- Charge utile : même requête avec `> 't`.
- « Welcome back » n’apparaît pas : la condition est fausse, donc le premier caractère n’est pas supérieur à `t`.
**Étape 3 : confirmer**
- Charge utile : même requête avec `= 's`.
- « Welcome back » s’affiche : le premier caractère du mot de passe est `s`.

On répète le processus pour chaque position, jusqu’à écrire le mot de passe complet.
**Point pratique**:
- Le principe des comparaisons `>` revient à une recherche par dichotomie : cela réduit le nombre de requêtes par caractère.
- Dans Burp, cette étape s’automatise avec Intruder (type d’attaque Cluster bomb ou Sniper, avec deux positions : l’index et le caractère), en filtrant les réponses sur la présence de « Welcome back ».
- Selon la base, la fonction s’appelle `SUBSTRING` ou `SUBSTR` (Oracle), et la concaténation de la requête varie : voir la cheat sheet de PortSwigger.

## Pièges et points d'attention BSCP


## Prévention


## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs

## Mes notes


## Liens
- [[Access-control]]
- [[Authentication]]
- [[Using-Burp-Suite]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
