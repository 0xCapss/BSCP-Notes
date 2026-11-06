---
tags: [bscp, server-side, sql-injection]
niveau: practitioner
statut: en cours
---
# SQL injection in different contexts
Une injecion SQL ne se limite pas à la query string. Toute entrée que l'application finit par traiter une requête SQL est un point d'injection: corps JSON ou XML, champs de formulaire, cookies, en-têtes comme `User-Agent` ou `Referer`. Le format change aussi la défense. Les filtres faibles cherchent des mots-clés SQL en clair dans la requête, et chaque format offre ses propres moyens d'échapper ces mots sans que l'application s'en aperçoive.

## Types et variantes
Le principe est toujours le mêe: le serveur décode l'entrée avant de le passer à l'interpréteur SQL, alors que le filtre inspecte la version encodée.
En XML, on encode un caractère du mot-clé avec une référence numérique. Le `S` de `SELECT` devient `&#x53;` :
```xml
<stockCheck>
    <productId>123</productId>
    <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId>
</stockCheck>
```
En JSON, la même idée passe par une séquence unicode : `\u0053ELECT`, que le parseur JSON décode avant que la valeur n’atteigne la requête.
D’autres contournements du même esprit existent quand le filtre est naïf : alterner majuscules et minuscules (`SeLeCt`), couper un mot-clé avec un commentaire (`SEL/**/ECT`), remplacer les espaces par des commentaires, doubler l’encodage URL. Certains endpoints acceptent aussi plusieurs formats : renvoyer en XML ou en JSON une requête prévue pour un formulaire peut suffire à passer à côté du filtre.
## Comment détecter
On teste chaque entrée de la requête, pas seulement les paramètres visibles : un `'` isolé, puis une comparaison des réponses (erreur, contenu différent, délai). Dans un corps XML ou JSON, il faut penser à l’échappement propre au format, sinon c’est le parseur qui plante avant la base et on conclut à tort que le champ est sain.

Un message du type « attack detected » ou un blocage immédiat signale un filtre sur les mots-clés. C’est le moment de passer à l’encodage plutôt que d’abandonner le point d’injection.

## Comment exploiter (principe)
1. Repérer une entrée qui atteint la base, quel que soit son format.
2. Constater qu’un payload classique est bloqué, ce qui prouve la présence d’un filtre.
3. Encoder seulement ce qui est nécessaire, en général une ou deux lettres de chaque mot-clé interdit (`UNION`, `SELECT`), pas la requête entière.
4. Terminer l’attaque comme d’habitude : `UNION`, extraction, concaténation si une seule colonne est affichée (`username || '~' || password` sur PostgreSQL).

## Pièges et points d'attention BSCP
- L’encodage doit correspondre au format : références `&#x..;` en XML, `\u....` en JSON. Un encodage URL dans un corps XML ne sera jamais décodé comme on l’espère.
- En XML, un `<` ou un `&` littéral dans le payload casse le document. Les échapper, ou les encoder en entités.
- N’encoder que les lettres qui déclenchent le filtre. Plus on encode, plus on multiplie les occasions de produire un payload invalide.
- Si rien ne passe, vérifier que l’entrée est bien vulnérable avec un test sans mot-clé (`'` seul, ou `1+1` à la place d’un nombre) avant de s’acharner sur le contournement.
- Un point d’injection peut se cacher dans un champ qu’on ne pense pas à tester : cookie, en-tête, valeur d’un champ JSON imbriqué.

## Prévention

## Journal des labs


## Liens
- [[SQL-injection]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
