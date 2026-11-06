---
tags: [bscp, server-side, sql-injection]
niveau: practitioner
statut: en cours
---
## Second-order SQL injection
Une injection de premier ordre exploite une entrée de la requête HTTP qui se retrouve aussitôt dans une requête SQL non sécurisée. Dans une injection de second ordre, l’application stocke d’abord l’entrée, en général dans la base, sans que rien ne se passe à cet endroit. La faille apparaît plus tard, dans une autre requête HTTP, quand l’application relit la donnée et la colle dans une requête SQL de façon dangereuse. On parle aussi d’injection SQL stockée.

Elle se rencontre surtout chez des développeurs qui connaissent le risque. Ils protègent correctement l’insertion, puis considèrent la donnée comme fiable puisqu’elle est déjà en base. Cette confiance est l’erreur.

## Types et variantes
Le schéma est toujours en deux temps : un point d’entrée où le payload est stocké, un point de déclenchement où il est relu et exécuté. Les deux sont souvent à des endroits très différents de l’application.

Cas classique : on crée un compte avec le nom d’utilisateur `admin'--`. L’inscription est paramétrée, le nom est stocké tel quel. Plus tard, la fonction de changement de mot de passe construit sa requête avec ce nom :

```sql
UPDATE users SET password='nouveau' WHERE username='admin'--'
```

Le commentaire neutralise la fin de la requête, et c’est le mot de passe de `admin` qui change, pas le nôtre.

Le déclenchement peut aussi dépendre d’une autre personne : un champ de profil lu par un administrateur dans son interface, un nom affiché dans un export, un rapport généré par une tâche planifiée. Dans ce cas on ne voit jamais le résultat soi-même.
## Comment détecter
La réponse à la requête qui stocke le payload n’indique rien : elle est normale, puisque l’insertion est sûre. Il faut donc repérer les champs stockés puis les fonctions qui les réutilisent.
1. Lister toutes les données qu’on peut enregistrer : nom d’utilisateur, adresse, nom de profil, commentaire, titre, préférences.
2. Y placer un payload de test (un `'`, ou une condition qui change le comportement) et noter ce qui a été stocké.
3. Parcourir ensuite les fonctionnalités qui relisent ces données : changement de mot de passe, profil, historique, recherche, export, panneau d’administration.
4. Comparer le comportement : erreur SQL, contenu différent, délai.

Quand le déclenchement est invisible pour nous (côté administrateur, tâche différée), la détection hors bande avec Collaborator devient le meilleur outil.
## Comment exploiter (principe)
1. Trouver un champ stocké qui est réutilisé dans une requête.
2. Construire le payload en fonction du contexte du second point, pas du premier. C’est la requête qui relit la donnée qui compte : le type de colonne, les guillemets autour de la valeur, le SGBD.
3. Stocker le payload via le point d’entrée.
4. Déclencher la fonctionnalité qui relit la donnée.
5. Exploiter comme d’habitude selon ce qu’on observe : `UNION` si le résultat est affiché, techniques aveugles ou OAST sinon.

## Pièges et points d'attention BSCP
- Le payload doit survivre au stockage. Une limite de longueur sur le champ, un filtre à l’inscription ou une normalisation peuvent le tronquer ou le modifier, et il ne marchera pas au moment de la relecture.
- On teste le mauvais endroit si on s’attend à une réaction immédiate. Rien ne se passe à l’insertion, c’est normal.
- Se placer dans le contexte de la requête qui lit la donnée. Un payload écrit pour une requête `SELECT` ne marche pas forcément dans un `UPDATE`, où il faut penser à ce qu’on modifie vraiment (l’exemple du mot de passe d’un autre compte).
- Le délai entre le stockage et l’exécution peut être long (tâche nocturne, revue par un administrateur). Patience, et ne pas conclure trop vite à l’absence de faille.
- Tester tous les champs enregistrables, y compris ceux qu’on ne pense pas à regarder (préférences, nom d’un fichier, métadonnées).
- Après un test, penser au nettoyage : un compte ou un enregistrement piégé reste en base et peut se déclencher plus tard, y compris en mission réelle.

## Prévention
Les requêtes paramétrées doivent s’appliquer partout, y compris quand la valeur vient de la base de données. Une donnée stockée n’est pas fiable pour autant : elle vient à l’origine d’un utilisateur. Il faut donc traiter chaque donnée relue comme une entrée non fiable, et ne jamais construire une requête par concaténation, quelle que soit la source de la valeur.

## Liens
- [[SQL-injection]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
