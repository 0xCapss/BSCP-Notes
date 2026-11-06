---
tags: [bscp, server-side, sql-injection]
niveau: practitioner
statut: en cours
---
# SQL injection using out-of-band techniques

## En bref
Quand la requête SQL est exécutée de manière asynchrone, la réponse HTTP ne dépend plus ni des données, ni d'une erreur, ni d'une exécution. Il reste encore une technique: faire émettre par le serveur de base de données une interaction réseau vers une machine contrôlée par l'attaquant.
## Pourquoi le DNS
- Les réseaux de production laissent presque toujours sortir le DNS car il est indispensable.
- La base n'a pas besoin de joindre l'attaquant en TCP: il suffit qu'elle résolve un nom dont l'attaquant contrôle le serveur DNS faisant autorité.
- Pour cela 2 usages:
	- **Détection**: provoquer une requête DNS vers son domaine. Si elle apparaît dans le journal, l'injection est confirmée.
	- **Exfiltration**: Placer le résultat d'une sous-requête dans le nom d'hote (`motdepasse.COLLAB`). La donnée arrive directement dans la requête DNS.
Une conditon peut aussi déclencher l'intéraction(`IF ... exec xp_dirtree`), ce qui donne un booléen sur un canal qui marche en asynchrone.

## Types et variantes


## Comment détecter


## Comment exploiter (principe)




## Pièges et points d'attention BSCP


## Prévention

## Journal des labs

## Liens
- [[SQL-injection]]
- [[Error-based SQL Injection]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
