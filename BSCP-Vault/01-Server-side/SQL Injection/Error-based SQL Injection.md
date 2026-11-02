---
tags: [bscp, server-side, sql-injection]
niveau: apprentice
statut: en cours
---
# Error-based SQL Injection

## En bref
- Cette technique d'injection désigne les cas où les messages d'erreurs permettent d'extraire ou de déduire des données sensibles de la base, même dans un contexte blind.
- Pour cela 2 approches:
	- **Erreurs conditionnelles:** On peut parfois induire en erreur une réponse d'erreur spécifique selon le résultat d'une exporession booléenne. L’exploitation est la même que pour les réponses conditionnelles vues précédemment.
	- **Message d'erreurs verbeux**: On peut parfois déclencher des messages d'erreurs qui affichent les données renvoyées par la requête. Cela transforme une blind SQLi en injection visible.


## Types et variantes
.

## Comment détecter


## Comment exploiter (principe)
- Il est souvent possible d’induire une réponse différente selon qu’une erreur SQL se produit ou non.
- On modifie la requête pour qu’elle provoque une erreur de base de données uniquement si la condition injectée est vraie.
- Une erreur non gérée provoque très souvent une différence dans la réponse de l’application, comme un message d’erreur.
- Ainsi cette différence permet de déduire si la condition injectée est vraie ou fausse.

### Erreurs conditionnelles avec `CASE`

- Deux requêtes sont envoyées tour à tour, avec ces valeurs pour le cookie `TrackingId` :
    - `xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a`
    - `xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a`
- Le mot-clé `CASE` teste une condition et renvoie une expression différente selon qu’elle est vraie ou fausse.

**Résultat de chaque requête**

- Première valeur : la condition `1=2` est fausse, `CASE` renvoie `'a'`, donc aucune erreur.
- Seconde valeur : la condition `1=1` est vraie, `CASE` renvoie `1/0`, ce qui provoque une erreur de division par zéro.
- Si l’erreur change la réponse HTTP de l’application, on peut en déduire si la condition injectée est vraie.

**Extraction de données**

- On récupère les données en testant un caractère à la fois : `xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a`
- Si l’erreur se produit, le premier caractère du mot de passe de `Administrator` est supérieur à `m`. Sinon, il ne l’est pas.
- On répète ensuite avec d’autres comparaisons et d’autres positions, comme pour les réponses conditionnelles.

**Point pratique** (connaissance générale, non issue de votre texte)

- Sur Oracle, `1/0` doit souvent être remplacé par `TO_CHAR(1/0)` et la requête nécessite `FROM DUAL`


## Pièges et points d'attention BSCP


## Prévention


## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Blind SQL injection avec erreurs conditionnelles
Cet exercice pratique comporte une vulnérabilité d’injection SQL aveugle. L’application utilise un cookie de suivi à des fins d’analyse et exécute une requête SQL contenant la valeur du cookie soumis.

Les résultats de la requête SQL ne sont pas renvoyés, et l’application ne réagit pas différemment selon que la requête renvoie ou non des lignes. Si la requête SQL provoque une erreur, l’application renvoie alors un message d’erreur personnalisé.

La base de données contient une table appelée « users », comportant des colonnes intitulées « username » et « password ». Vous devez exploiter la vulnérabilité d’injection SQL aveugle pour découvrir le mot de passe de l’utilisateur administrateur.

Pour résoudre cet exercice, connectez-vous en tant qu’utilisateur administrateur.
#### Solution


  

_Traduit avec [DeepL.com](https://www.deepl.com/?utm_campaign=product&utm_source=web_translator&utm_medium=web&utm_content=copy_free_translation) (version gratuite)_

## Liens
- [[Access-control]]
- [[Authentication]]
- [[Using-Burp-Suite]]
- [[Payloads-cheatsheet]]
