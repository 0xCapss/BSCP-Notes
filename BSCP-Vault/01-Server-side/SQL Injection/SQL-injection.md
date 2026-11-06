---
tags: [bscp, server-side, sql-injection]
niveau: apprentice
statut: en cours
---
# SQL injection

## En bref
- Faille de sécurité Web qui permet à un attaquant d'interférer avec les requêtes que l'application envoie à sa base de données.
- L'attaquant peut consulter des données auxquelles il n'a normalement pas accès.
- Il peut ainsi modifier, supprimer ces données, ce qui entraîne des changements persistants dans le contenu ou le comportement de l'application.
- Dans certaines situations, l'attaque peut être étendue pour compromettre le serveur sous-jacent ou d'autres infrastructures back-end.
- Elle peut également permettre de mener des attaques par déni de service.
- Impact concret, selon le mécanisme exploité :
	- authentication bypass (condition de connexion neutralisée) ;
	- lecture d'autres tables (UNION attack, Error-based SQLi, Blind SQLi) ;
	- écriture ou suppression de données (requête de modification, stacked queries) ;
	- lecture ou écriture de fichiers et exécution de commandes via des fonctions du moteur, sous réserve de privilèges élevés (voir plus bas).
## Récupération de données cachées
- Exemple : une boutique en ligne affiche des produits par catégorie. Cliquer sur la catégorie Gifts envoie la requête `https://insecure-website.com/products?category=Gifts`.
- L'application exécute alors la requête SQL : `SELECT * FROM products WHERE category = 'Gifts' AND released = 1`.
**Lecture de la requête**
- `*` : tous les détails.
- `products` : la table interrogée.
- `category = 'Gifts'` : uniquement la catégorie Gifts.
- `released = 1` : uniquement les produits publiés.
**Rôle de la restriction**
- `released = 1` sert à masquer les produits non publiés.
- On peut supposer que les produits non publiés ont `released = 0`.
## Exemple d'attaque
**Attaque 1 : supprimer la restriction avec un commentaire**
- L'application n'a aucune protection contre la SQL injection (SQLi).
- L'attaquant envoie `https://insecure-website.com/products?category=Gifts'--`.
- La requête devient : `SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1`.
- `--` est un indicateur de commentaire en SQL : le reste de la requête est ignoré, donc `AND released = 1` disparaît.
- Résultat : tous les produits de la catégorie sont affichés, y compris ceux qui ne sont pas encore sortis.
- L'URL est montrée sans encodage : un navigateur encode l'espace et `'` selon le contexte, d'où l'intérêt de saisir la valeur dans Burp Repeater.

**Attaque 2 : afficher toutes les catégories**
- Une attaque similaire force l'application à afficher les produits de n'importe quelle catégorie, même inconnue : `category=Gifts'+OR+1=1--`.
- La requête devient : `SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1`.
- Elle renvoie tous les articles dont la catégorie est `Gifts` ou pour lesquels 1 est égal à 1.
- Comme `1=1` est toujours vrai, tous les articles sont renvoyés.

**Avertissement**
- Attention en injectant `OR 1=1` : même si cela semble inoffensif là où on l'injecte, les applications réutilisent souvent les données d'une requête dans plusieurs autres.
- Si la condition atteint un `UPDATE` ou un `DELETE`, elle peut provoquer une perte accidentelle de données.
## Détournement logique d'application
**Détourner la logique de l'application**
- Exemple : une application authentifie les utilisateurs par nom d'utilisateur et mot de passe. Avec `wiener` et `bluecheese`, elle exécute : `SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'`.
- Si la requête renvoie les détails d'un utilisateur, la connexion réussit. Sinon, elle est rejetée.
**Attaque**
- Un attaquant peut se connecter en tant que n'importe quel utilisateur, sans mot de passe.
- Il utilise la séquence de commentaire SQL `--` pour retirer la vérification du mot de passe de la clause `WHERE`.
- Exemple : le nom d'utilisateur `administrator'--` avec un mot de passe vide donne : `SELECT * FROM users WHERE username = 'administrator'--' AND password = ''`.
- Cette requête renvoie l'utilisateur `administrator` et connecte l'attaquant sous ce compte.

## Contextes d'injection
Avant d'écrire un payload, identifier où la valeur se retrouve dans la requête. Tout l'exemple ci-dessus suppose une chaîne dans un `WHERE`.

| Contexte | Comment s'en sortir | Exemple de forme | Remarque |
|---|---|---|---|
| Chaîne entre guillemets (`WHERE col = '...'`) | Fermer avec `'`, puis commenter la fin | `Gifts'--` | Cas des labs de cette catégorie |
| Nombre (`WHERE id = 1`) | Aucun guillemet à fermer | `1 OR 1=1--` | Test : `id=2-1` doit donner la même page que `id=1` |
| Identifiant (nom de colonne ou de table) | Pas de guillemet ; la valeur est un mot-clé | `sort=price` | Non paramétrable : les requêtes préparées ne protègent pas, allowlist nécessaire |
| Clause `ORDER BY` | Indice ou nom de colonne | `ORDER BY 1` puis `ORDER BY 2` | Le tri change selon l'indice ; hors plage, la requête échoue |
| Clause `LIMIT` | Valeur numérique | `LIMIT 10` | Peut tronquer une ligne injectée par `UNION` |
| `INSERT` ou `UPDATE` | Fermer la chaîne, parfois aussi la parenthèse de `VALUES (...)` | `x')--` | Dangereux : modifie des données |

## Types et variantes
| Variante | Principe | Note |
|---|---|---|
| Récupération de données cachées | Suppression d'une restriction de la clause `WHERE` | Exemples plus haut |
| Détournement de la logique | Login bypass | Exemples plus haut |
| In-band (UNION attack) | Les résultats injectés sont affichés dans la réponse | [[SQL-injection-Union-Attacks]] |
| Error-based SQLi | Une erreur affiche une valeur, ou une conditional error sert de signal | [[Error-based SQL Injection]] |
| Blind SQLi (boolean-based, time-based) | Signal indirect : contenu, temps de réponse | [[Blind-SQL-injection]] |
| Out-of-band (OAST) | Interaction DNS ou HTTP vers un serveur contrôlé | [[Blind-SQL-injection]] |
| Second-order | Donnée stockée puis réutilisée dans une autre requête | Voir la section dédiée plus bas |

Autres langages de requête : la NoSQL injection, la LDAP injection, la XPath injection ou l'injection dans un ORM suit la même logique (entrée non fiable interprétée comme requête) mais avec une syntaxe propre, hors périmètre de ces notes.

## Comment détecter
On teste chaque point d'entrée avec une série de tests systématiques, dans cet ordre : erreur, boolean-based, time-based, out-of-band.
1. **Apostrophe** : envoyer `'` puis `''`. Erreur ou anomalie avec un seul guillemet, retour à la normale avec deux : le guillemet est interprété par la requête.
2. **Arithmétique sur un nombre** : `id=2-1` doit renvoyer la même page que `id=1` si la valeur est évaluée par la base.
3. **Concaténation neutre** : `x'||'` (Oracle, PostgreSQL), `x'+'` (SQL Server), `x' 'y` ou `CONCAT` (MySQL). Une réponse inchangée suggère une SQLi dans une chaîne.
4. **Boolean-based** : deux requêtes qui ne diffèrent que par la condition (`x' AND '1'='1` puis `x' AND '1'='2`, ou `OR 1=1` et `OR 1=2`). Comparer contenu, longueur de réponse (colonne Length d'Intruder), code HTTP.
5. **Time-based** : payload qui suspend la requête, comparé au temps de référence (détails et noms de fonctions dans [[Blind-SQL-injection]]).
6. **Out-of-band** : payload qui déclenche une interaction DNS ou HTTP, surveillée avec Burp Collaborator.
- **Critère de confirmation** : une différence reproductible qui dépend de la condition (vrai donne A, faux donne B, plusieurs fois de suite). Une différence isolée peut venir du bruit, d'une session expirée ou du cache.
- **Burp Scanner** permet de détecter rapidement et de manière fiable la majorité des SQLi.

### Identifier le moteur
Syntaxes issues de la cheat sheet PortSwigger. Le tableau maître par moteur (sous-chaîne, délai, stacked queries, etc.) est dans [[Blind-SQL-injection]] ; celui des concaténations et de l'énumération du schéma dans [[SQL-injection-Union-Attacks]].

| Moteur | Version | Commentaire | Concaténation |
|---|---|---|---|
| Oracle | `SELECT banner FROM v$version` | `--` | `'a'\|\|'b'` |
| MySQL | `@@version` | `-- ` (espace final) ou `#` | `CONCAT('a','b')` |
| PostgreSQL | `version()` | `--` | `'a'\|\|'b'` |
| SQL Server | `@@version` | `--` | `'a'+'b'` |
| SQLite | `sqlite_version()` | `--` | `'a'\|\|'b'` |

## Points d'entrée
Les points d'entrée ne se limitent pas aux paramètres d'URL. Dans Burp, modifier la valeur dans Repeater, ou poser les payload positions d'Intruder autour de la seule valeur testée.

| Emplacement | Particularité | Point d'injection |
|---|---|---|
| Formulaire `application/x-www-form-urlencoded` | Encodage URL : `+` lu comme espace, `#` non envoyé | Valeur du paramètre dans le corps |
| JSON | Échapper les guillemets (`\"`) ; garder le JSON valide | Valeur de la clé concernée |
| XML | Entités XML (`&#x53;`) pour un bypass de filtre | Contenu de l'élément |
| Cookie (`TrackingId`) | `;` sépare les cookies, l'encoder en `%3B` ; `+` n'est pas forcément décodé comme un espace ; ne pas casser le cookie `session` | Valeur du cookie, en remplaçant ou prolongeant l'originale |
| Headers (`X-Forwarded-For`, `Referer`, `User-Agent`, `Host`) | Dépend de ce que l'application journalise ou stocke | Valeur du header |

## Comment exploiter (principe)
- Comprendre la requête SQL probable côté serveur (ici une clause `WHERE` avec une chaîne entre guillemets) pour savoir où fermer le guillemet et quoi neutraliser.
- Détournement de logique :
	- Commenter la fin de la requête avec `--` pour supprimer une restriction (`AND released = 1`) ou une vérification (`AND password = ...`).
	- Ajouter une condition toujours vraie (`OR 1=1`) pour élargir les résultats.
- Extraction de données : UNION attack pour lire d'autres tables quand les résultats sont affichés, voir [[SQL-injection-Union-Attacks]].
- Données lisibles par les erreurs (Error-based SQLi) : voir [[Error-based SQL Injection]]. Sans aucun affichage (Blind SQLi) : voir [[Blind-SQL-injection]].
- Payloads regroupés dans [[Payloads-cheatsheet]].

### Stacked queries (principe)
- Le séparateur `;` enchaîne une seconde instruction : `'; <requête>--`.
- Dépend du moteur et de l'interface de programmation : souvent possible sur SQL Server et PostgreSQL, selon l'interface une seule instruction est acceptée (Oracle : non, MySQL : rare).
- Usage : délai conditionnel (time-based), interaction out-of-band, écriture. Signal : effet visible (délai, interaction reçue, ligne ajoutée).
- Danger : modification de données. Réserver à des fonctions inoffensives (délai, résolution DNS). Dans un cookie, encoder `;` en `%3B`.

### Lecture et écriture de fichiers (principe)
Ces fonctions sortent du cadre des labs standards et ne s'utilisent qu'en environnement autorisé et isolé.

| Moteur | Lecture | Écriture | Prérequis |
|---|---|---|---|
| MySQL | `LOAD_FILE('/etc/passwd')` | `SELECT ... INTO OUTFILE '/chemin'` | Privilège `FILE`, variable `secure_file_priv` |
| PostgreSQL | `pg_read_file('/etc/passwd')` | `COPY ... TO '/chemin'` | Superutilisateur ou rôle dédié |
| SQL Server | `OPENROWSET(BULK '...', SINGLE_CLOB)` | Via composants activés | Droits d'administration |
| Oracle | `UTL_FILE` | `UTL_FILE.PUT_LINE` | Objet `DIRECTORY` déclaré et droits |

### Second-order (principe)
- Étape 1 : enregistrer une valeur contenant `'` dans un champ stocké (nom, adresse, nom d'utilisateur). La première requête est sûre.
- Étape 2 : déclencher une fonction qui réutilise la valeur stockée (changement de mot de passe, rapport, profil).
- Signal : erreur ou comportement anormal dans la seconde fonction, pas dans la première. Peu de données reviennent en retour et l'effet peut survenir plus tard.

### Bypass de filtre (principe)
Utile quand un mot-clé ou un caractère est bloqué.
- Casse (`UnIoN`), commentaires en ligne (`UN/**/ION`), espaces remplacés (`/**/`, `%0a`, `%09`).
- Double encodage URL, entités XML dans un corps XML, valeurs hexadécimales pour éviter les guillemets.
- Limite : un pare-feu applicatif peut normaliser l'entrée avant analyse.

### Limites générales
- Privilèges du compte de base de données (lecture seule, tables restreintes, pas de fonctions réseau ou fichiers).
- Stacked queries non supportées par certaines interfaces de programmation.
- Longueur ou filtrage des entrées, pare-feu applicatif.

## Pièges et points d'attention BSCP
Les pièges communs à toutes les variantes sont regroupés ici ; les autres notes y renvoient.
- MySQL : `--` doit être suivi d'un espace (`--+` dans une URL), ou utiliser `#` encodé en `%23`. Cela concerne aussi `+OR+1=1--` des exemples ci-dessus.
- Encodage URL : `+` est lu comme un espace, `#` non encodé n'est jamais envoyé au serveur.
- Cookie : il n'est pas décodé comme une URL (le `+` n'y devient pas forcément un espace), `;` s'encode en `%3B`, et la valeur d'origine est parfois à remplacer plutôt qu'à prolonger. Laisser le cookie `session` intact.
- Corps de requête : respecter le `Content-Type` (JSON, XML, formulaire) et son encodage.
- `OR 1=1` sur une fonctionnalité qui modifie des données (`UPDATE`, `DELETE`) peut causer une perte de données : voir l'avertissement plus haut, le réserver à des lectures.

## Prévention
- Utiliser des requêtes paramétrées (requêtes préparées) pour toute donnée fournie par l'utilisateur, jamais de concaténation de chaînes dans la requête.
- Les requêtes paramétrées ne protègent pas les identifiants ni les `ORDER BY` dynamiques : utiliser une allowlist de valeurs autorisées.
- Une procédure stockée qui concatène des chaînes en interne reste vulnérable.
- La validation d'entrée est un complément, jamais un remède à elle seule.
- Appliquer le principe du moindre privilège au compte de la base de données, avec des comptes séparés pour la lecture et l'écriture.
- Ne pas renvoyer les erreurs détaillées de la base à l'utilisateur ; journaliser côté serveur.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-26
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data
Ce lab contient une injection SQL dans le filtre de catégorie des produits. Quand l'utilisateur choisit une catégorie, l'application exécute une requête SQL comme celle-ci :
`SELECT * FROM products WHERE category = 'Gifts' AND released = 1`
Pour réussir le lab, il faut réaliser une injection SQL qui fait afficher à l'application un ou plusieurs produits non publiés.
#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête qui définit le filtre de catégorie.
2. Modifier le paramètre `category` en lui donnant la valeur `'+OR+1=1--`.
3. Envoyer la requête et vérifier que la réponse contient maintenant un ou plusieurs produits non publiés.
### Lab: SQL injection vulnerability allowing login bypass
Ce lab contient une injection SQL dans la fonction de connexion.
Pour réussir le lab, il faut réaliser une injection SQL qui connecte à l'application en tant qu'utilisateur `administrator`.
#### Solution
1. Utiliser Burp Suite pour intercepter et modifier la requête de connexion.
2. Modifier le paramètre `username` en lui donnant la valeur : `administrator'--`, avec un mot de passe quelconque (le champ est obligatoire dans le formulaire, mais la vérification est commentée).


## Liens
- [[SQL-injection-Union-Attacks]]
- [[Blind-SQL-injection]]
- [[Error-based SQL Injection]]
- [[Payloads-cheatsheet]]
