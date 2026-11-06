---
tags: [bscp, server-side, sql-injection]
niveau: apprentice
statut: en cours
---
# Blind SQLi - Triggering time delay

## En bref


## Types et variantes



## Comment détecter


## Comment exploiter (principe)
- Déclencher un délai selon qu'une condition injectée est vraie ou fausse. Les requêtes SQL sont traitées de façon synchrone, donc un délai côté base retarde la réponse HTTP. Ce temps de réponse indique la valeur de vérité de la condition.
- Toutefois, la syntaxe du délai dépend du moteur de base de données.
```sql
'; IF (1=2) WAITFOR DELAY '0:0:10'--
'; IF (1=1) WAITFOR DELAY '0:0:10'--
```
### Extraction de données caractère par caractère
```sql
'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--
```
- Le délai se produit uniquement si la condition sur le caractère testé est vraie.
- On répète en fasant varier la position `SUBSTRING` et le caractère comparé pour reconstituer la valeur.

## Pièges et points d'attention BSCP


## Prévention


## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: Blind SQL injection with time delays and information retrieval
This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows or causes an error. However, since the query is executed synchronously, it is possible to trigger conditional time delays to infer information.

The database contains a different table called `users`, with columns called `username` and `password`. You need to exploit the blind SQL injection vulnerability to find out the password of the `administrator` user.

To solve the lab, log in as the `administrator` user.
#### Solution


## Liens
- [[SQL-injection-Union-Attacks]]
- [[Blind-SQL-injection]]
- [[Error-based SQL Injection]]
- [[Payloads-cheatsheet]]
