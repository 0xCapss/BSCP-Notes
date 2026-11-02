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
	- **Message d'erreurs verbeux**: On peut parfois déclencher des messages d'erreurs qui affichent les données renvoyées par la requête. Cela transforme une injection aveu


## Types et variantes
.

## Comment détecter


## Comment exploiter (principe)


## Pièges et points d'attention BSCP


## Prévention


## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs


## Liens
- [[Access-control]]
- [[Authentication]]
- [[Using-Burp-Suite]]
- [[Payloads-cheatsheet]]
