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
