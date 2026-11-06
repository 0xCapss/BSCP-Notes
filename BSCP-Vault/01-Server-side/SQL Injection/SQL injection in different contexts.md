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
Les requêtes paramétrées règlent le problème quel que soit le format d’entrée, car la donnée n’est jamais interprétée comme du SQL. Un filtre ou un WAF ne remplace pas cette mesure : il ne peut que compléter. S’il y en a un, il doit décoder et normaliser l’entrée de la même façon que l’application avant de l’inspecter, sinon l’encodage le contourne. Une liste blanche de valeurs attendues (un identifiant de produit numérique, par exemple) réduit encore la surface.

## Journal des labs
### SQL injection with filter bypass via XML encoding
This lab contains a SQL injection vulnerability in its stock check feature. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables.

The database contains a `users` table, which contains the usernames and passwords of registered users. To solve the lab, perform a SQL injection attack to retrieve the admin user's credentials, then log in to their account.
**Identify the vulnerability**

1. Observe that the stock check feature sends the `productId` and `storeId` to the application in XML format.
2. Send the `POST /product/stock` request to Burp Repeater.
3. In Burp Repeater, probe the `storeId` to see whether your input is evaluated. For example, try replacing the ID with mathematical expressions that evaluate to other potential IDs, for example:
    `<storeId>1+1</storeId>`
4. Observe that your input appears to be evaluated by the application, returning the stock for different stores
5. Try determining the number of columns returned by the original query by appending a `UNION SELECT` statement to the original store ID:
    `<storeId>1 UNION SELECT NULL</storeId>`
6. Observe that your request has been blocked due to being flagged as a potential attack.
    

**Bypass the WAF**
1. As you're injecting into XML, try obfuscating your payload using XML entities. One way to do this is using the [Hackvertor](https://portswigger.net/bappstore/65033cbd2c344fbabe57ac060b5dd100) extension. Just highlight your input, right-click, then select **Extensions > Hackvertor > Encode > dec_entities/hex_entities**.
2. Resend the request and notice that you now receive a normal response from the application. This suggests that you have successfully bypassed the WAF.

**Craft an exploit**

1. Pick up where you left off, and deduce that the query returns a single column. When you try to return more than one column, the application returns `0 units`, implying an error.
2. As you can only return one column, you need to concatenate the returned usernames and passwords, for example:
    `<storeId><@hex_entities>1 UNION SELECT username || '~' || password FROM users</@hex_entities></storeId>`
3. Send this query and observe that you've successfully fetched the usernames and passwords from the database, separated by a `~` character.
4. Use the administrator's credentials to log in and solve the lab.
## Liens
- [[SQL-injection]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
